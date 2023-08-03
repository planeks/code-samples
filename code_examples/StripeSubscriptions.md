# Stripe subscriptions

An example of the relatively complex monthly subscription logic with Stripe for the application
where users operate with data and have a limited amount of queries per month. These are the
queries to the internal database the users try to search in.

_models.py_

```python
from django.db import models
from django.conf import settings


class Subscription(models.Model):
    class SubscriptionDurationPeriods:
        MONTH = 30
        YEAR  = 365
        PERIODS = (
            (MONTH, 'Month'),
            (YEAR, 'Year'),
        )

    api_id = models.CharField(max_length=1000)
    price = models.FloatField()
    name = models.CharField(max_length=100)
    description = models.CharField(max_length=2000, blank=True, null=True)
    order = models.PositiveSmallIntegerField(default=0)
    duration = models.IntegerField(blank=False, null=False, choices=SubscriptionDurationPeriods.PERIODS, default=SubscriptionDurationPeriods.MONTH)
    config = models.JSONField(blank=True, null=False, default=dict)
    is_visible = models.BooleanField(default=True) # defines free tier subscription
    queries_amount = models.PositiveIntegerField(default=0)

    class Meta:
        verbose_name = 'Subscription'
        verbose_name_plural = 'Subscriptions'
        ordering = ['order', 'id']

    def __str__(self):
        return self.name


class UserSubscriptionLog(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, related_name='subscription_logs', on_delete=models.CASCADE)
    subscription = models.ForeignKey(Subscription, related_name='subscription_logs', on_delete=models.CASCADE)
    to_date = models.DateField() # a start date of this subscription
    data = models.JSONField('Response data', blank=True, null=False, default=dict)
    created = models.DateTimeField(auto_now_add=True)
    duration = models.PositiveIntegerField(default=30) # duration since to_date
    is_enabled = models.BooleanField(default=True) # defines whether this subscription is enabled 

    class Meta:
        verbose_name = 'User Subscription Log'
        verbose_name_plural = 'User Subscription Logs'
        ordering = ['-created', '-id']
    
    def __str__(self):
        return f"{self.user.name} - {self.subscription.name} - {self.to_date}"


class UserMonthQueries(models.Model):
    """Keeps track of user's used queries for a month
    """
    user = models.ForeignKey(settings.AUTH_USER_MODEL, related_name='month_queries', on_delete=models.CASCADE)
    start_date = models.DateField()
    count = models.PositiveIntegerField(default=0)

    class Meta:
        verbose_name = 'User Month Queries'
        verbose_name_plural = 'User Month Queries'
        ordering = ['-start_date', '-id']

    def __str__(self):
        return f"{self.user.name} - {self.start_date}"
```

_service.py_

```python
import stripe
from accounts.models import User


def find_user_subscription_id(user: User):
    subscription_log = user.subscription_logs.first()
    subscription_id = None
    if subscription_log:
        subscription_id = subscription_log.data.get("data", {}).get("object", {}).get("subscription", None)
    return subscription_id


def cancel_user_subscription(subscription_id, user: User):
    if not subscription_id:
        return
    try:
        stripe.Subscription.cancel(subscription_id)
        subscription_logs = user.subscription_logs.filter(subscription_id=user.subscription.id).first()
        subscription_logs.is_enabled = False
        subscription_logs.save()
    except ValueError as e: # if subscription was deleted from dashboard
        subscription_logs = user.subscription_logs.filter(subscription_id=user.subscription.id).first()
        subscription_logs.is_enabled = False
        subscription_logs.save()
    except Exception as e:
        print(f"Exception: {e}")
```

_views.py_

```python
import stripe
from datetime import date, timedelta
from django.conf import settings
from django.contrib.auth.decorators import login_required
from django.views.decorators.csrf import csrf_exempt
from django.shortcuts import redirect, render
from django.urls import reverse
from django.http import HttpResponse, HttpResponseForbidden
from accounts.models import User, Subscription, UserSubscriptionLog, UserMonthQueries
from accounts.service import find_user_subscription_id, cancel_user_subscription
from dashboards.models import ApplicationSettings, Dashboard, Widget


@login_required(login_url='/login')
def manage_subscriptions(request):
    subscriptions = Subscription.objects.filter(is_visible=True)
    user_subscription = request.user.subscription
    payment_info = None
    future_subscription = None
    cancelled_subscription = False
    if not request.user.is_free_tier:
        subscription_id = find_user_subscription_id(request.user) # always the last subscription to show actual payment info
        subscription_data = stripe.Subscription.retrieve(subscription_id)
        payment_method = subscription_data.get("default_payment_method")
        payment_info = stripe.PaymentMethod.retrieve(payment_method)
        future_subscription = request.user.subscription_logs.filter(to_date__gt=date.today()).first()
        last_subscription_log = request.user.subscription_logs.filter(subscription_id=user_subscription.id).first()
        if last_subscription_log:
            cancelled_subscription = not last_subscription_log.is_enabled

    context = {
        "subscriptions": subscriptions,
        "user_subscription": user_subscription,
        "title": "Manage your subscriptions",
        "payment_info": payment_info,
        "future_subscription": future_subscription,
        "cancelled_subscription": cancelled_subscription
    }
    return render(request, 'accounts/user_settings/subscriptions/manage_subscriptions.html', context)


def create_checkout_session(request):
    if not request.POST:
        return redirect(reverse(viewname='manage_subscriptions'))

    user = request.user
    subscription_id = request.POST.get("item_id", None)
    new_subscription = Subscription.objects.get(id=subscription_id)

    discounts = [{}]
    subscription_data={
        "metadata": {
            "client_reference_id": user.id,
            "subscription_id": subscription_id,
            "mode": settings.CONFIGURATION
        }
    }
    user_subscription = user.subscription
    if not user.is_free_tier:
        price_diff = user_subscription.price - new_subscription.price
        if user_subscription.id == new_subscription.id:
            subscription_days_left = user.subscription_days_left + 1
            
            if subscription_days_left:
                subscription_data.update({
                    "trial_period_days": subscription_days_left,
                })
        
        # if downgrading 
        elif price_diff > 0:
            # add new one with delay in days left in this one
            # after confirm - cancel current one
            subscription_days_left = user.subscription_days_left + 1
            if subscription_days_left:
                subscription_data.update({
                    "trial_period_days": subscription_days_left
                })

        # if upgrading
        else: 
            # add new subscription with discount
            # after confirm - cancel current one
            discount = int(user_subscription.price * 100)
            coupon = stripe.Coupon.create(amount_off=discount, duration="once", currency="usd", name=f"${user_subscription.price:.2f} discount for plan change")
            coupon_id = coupon.get("id")
            discounts = [{
                "coupon": coupon_id
            }]

    price_id = new_subscription.api_id
    session = stripe.checkout.Session.create(
        success_url=settings.SITE_URL + reverse(viewname='manage_subscriptions') + '?status=success',
        cancel_url=settings.SITE_URL + reverse(viewname='manage_subscriptions'),
        mode='subscription',
        line_items=[{
            'price': price_id,
            # For metered billing, do not pass quantity
            'quantity': 1
        }],
        discounts=discounts,
        subscription_data=subscription_data,
    )
    return redirect(session.url, code=303)


@csrf_exempt
def stripe_webhook(request):
    if settings.CONFIGURATION == 'prod':
        endpoint_secret = ApplicationSettings.load().stripe_endpoint_secret
    else:
        endpoint_secret = settings.STRIPE_ENDPOINT_SECRET
    payload = request.body
    sig_header = request.META['HTTP_STRIPE_SIGNATURE']
    event = None
    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, endpoint_secret
        )
    except ValueError as e:
        # Invalid payload
        print(f"ValueError: {e}")
        return HttpResponse(status=400)
    except stripe.error.SignatureVerificationError as e:
        # Invalid signature
        print(f"SignatureVerificationError: {e}")
        return HttpResponse(status=400)

    if event['type'] == 'invoice.upcoming':
        data = event['data']['object']
        # notify a user that he will be charged in few days, 
        # unless he already has subscription for the upcoming month via tier downgrading

    elif event['type'] == 'checkout.session.completed':
        data = event['data']['object']

    elif event['type'] == 'invoice.paid':
        # successful payment -> change user subscription        
        stripe_subscription_id = event.get("data", {}).get("object", {}).get("subscription")
        stripe_subscription = stripe.Subscription.retrieve(stripe_subscription_id)
        client_reference_id = stripe_subscription.get("metadata", {}).get("client_reference_id")
        subscription_id = stripe_subscription.get("metadata", {}).get("subscription_id")
        mode = stripe_subscription.get("metadata", {}).get("mode", "dev")
        # check for mode match
        # to exclude errors with users when prod and dev work with the same stripe products
        if mode != settings.CONFIGURATION:
            return HttpResponse(status=200)
        
        user = User.objects.get(id=client_reference_id)
        subscription = Subscription.objects.get(id=subscription_id)
        days_left = user.subscription_days_left
        duration = days_left if days_left else subscription.duration
        today = date.today()
        if user.is_free_tier:
            user.subscription = subscription
            user.save()
            UserSubscriptionLog.objects.create(
                user=user,
                subscription=subscription,
                to_date=today,
                data=event,
                duration=duration
            )
        else:
            cancel_user_subscription(find_user_subscription_id(user), user)

            trial_end = stripe_subscription.get("trial_end", None)
            if trial_end is not None:
                # downgrading
                # means that this payment was done manually and sets subscription for the next billing period
                # do not update current user subscription, set to_date to future
                # no way to set up scheduled subscription if current one is free tier
                UserSubscriptionLog.objects.create(
                    user=user,
                    subscription=subscription,
                    to_date=today + timedelta(days=60),
                    data=event,
                    duration=duration
                )

            elif days_left:
                # same subscription (updating current payment method) or upgrading
                # new subscription comes in place of the current one 
                last_user_subscription = user.subscription_logs.filter(subscription_id=user.subscription.id).first()
                last_to_date = last_user_subscription.to_date
                user.subscription = subscription
                user.save()
                scheduled_subscriptions = user.subscription_logs.filter(to_date__gt=last_to_date)
                scheduled_subscriptions.delete()
                UserSubscriptionLog.objects.create(
                    user=user,
                    subscription=subscription,
                    to_date=last_to_date,
                    data=event,
                    duration=subscription.duration
                )
            else:
                # current free tier is changed with some other
                user.subscription = subscription
                user.save()
                UserSubscriptionLog.objects.create(
                    user=user,
                    subscription=subscription,
                    to_date=today,
                    data=event,
                    duration=duration
                )
            
    elif event['type'] == 'invoice.payment_failed':
        # payment failed (not enough money) -> set free tier and notify user
        pass

    return HttpResponse(status=200)


@login_required(login_url='/login')
def cancel_subscription(request):
    if request.POST:
        user = request.user
        cancel_user_subscription(find_user_subscription_id(user), user)
    return redirect(reverse(viewname='manage_subscriptions'))
```
