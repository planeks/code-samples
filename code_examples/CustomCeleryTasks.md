# Custom celery taks

## tasks.py

```python
from abc import abstractmethod
from typing import Optional

from celery.app.task import Task
from django.contrib.auth import get_user_model
from django.db.models import Sum

import project_name.logs as project_name_logs

from .api_client.client import Client
from .models import AppsFlyerDevice, AppsFlyerEvent

UserModel = get_user_model()


class AppsFlyerTask(Task):
    event_name: Optional[str] = None
    autoregister = True

    def __init__(self, need_to_check: bool = False):
        self.need_to_check = need_to_check

    @property
    @abstractmethod
    def name(self):
        """Task name."""

    def check_event(self, user=None, **kwargs):
        """Check if we need to send an event"""
        if not self.need_to_check:
            return True

    def _check_existing_events(self, user):
        """Check if event is exist with such name"""
        return AppsFlyerEvent.objects.filter(
            event=self.event_name,
            device__user=user
        ).exists()

    @staticmethod
    def _get_sum_of_transactions(transactions):
        """Check if event is exist with such name"""
        sum_transactions = transactions.aggregate(summary=Sum("amount"))
        return sum_transactions["summary"] if sum_transactions["summary"] else 0

    def _send_appsflyer_event(self, user, **kwargs):
        """Api call to appsflyer"""
        client = Client(user=user)
        user_device = AppsFlyerDevice.get_last_device_by_user(user)
        if not user_device:
            return
        client.send_appsflyer_event(
            appsflyer_id=user_device.appsflyer_id,
            event_name=self.event_name,
            customer_user_id=str(user.id),
            advertiser_id=user_device.advertiser_id,
            **kwargs
        )
        project_name_logs.create_log(
            msg=f"{self.event_name} was send to the Appsflyer",
            user=user,
            user_device=user_device,
            logger_name=__name__,
            category=project_name_logs.CAT_BILL,
        )

    def run(self, user_id, *args, **kwargs):
        """Check task and send request to a appsflyer"""
        try:
            user = UserModel.objects.get(id=user_id)
        except UserModel.DoesNotExist:
            return
        send = self.check_event(user=user, **kwargs)
        if not send:
            return
        self._send_appsflyer_event(user, **kwargs)


class ActiveAppsFlyerTask(AppsFlyerTask):
    name = "third_party.appsflyer.tasks.ActiveCardAppsFlyerTask"
    event_name = "ob_card_activation"


class ConnectFirstAccountAppsFlyerTask(AppsFlyerTask):
    name = "third_party.appsflyer.tasks.ConnectFirstAccountAppsFlyerTask"
    event_name = "ob_conn_ext_bank_account"

    def check_event(self, user=None, **kwargs):
        """Check if we need to send an event"""
        events = self._check_existing_events(user)
        if not events:
            return True
        return False
```