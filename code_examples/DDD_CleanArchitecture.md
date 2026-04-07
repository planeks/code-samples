# DDD & Clean Architecture (Django, Error Logging Application)

A Django app split into three layers following DDD and Clean Architecture (Ports & Adapters), taken from an Error Logging Application (see the [System Architecture](SystemArchitecture.md) sample for the full C4 diagram). Domain has the models, repositories, factories, value objects (Pydantic), and abstract interfaces that act as ports. Application has use cases and policies. Infrastructure wires it all together with concrete adapters (API gateways, SQL executor, AI client), a DI container, Celery tasks, and Django-specific code. The domain never imports from infrastructure.

---

## Structure

```
app/
|--application/                 # Use cases and business flow orchestration
|  |--policies/                 ## Application-level business rules
|  |--use_cases/                ## Core business operations
|  |--settings.py               ## Application-level config
|
|--domain/                      # Core business logic and entities
|  |--services/                 ## Domain services
|  |  |--sql/                   ### Complex queries in plain SQL
|  |--choices.py                ## Domain enums and constants
|  |--factories.py              ## Factory pattern implementations
|  |--interfaces.py             ## Abstract ports (hexagonal boundaries)
|  |--managers.py               ## Repository pattern (custom Django managers)
|  |--models.py                 ## Domain entities (Django models)
|  |--settings.py               ## Domain-level config
|  |--types.py                  ## Value objects and DTOs (Pydantic)
|
|--infrastructure/              # External integrations and Django-specific code
|  |--management/               ## Django management commands
|  |--migrations/               ## Database schema migrations
|  |--templates/                ## UI presentation layer (HTMX)
|  |--admin.py                  ## Django admin interface
|  |--apps.py                   ## AppConfig with models_module override
|  |--di.py                     ## Dependency injection container
|  |--gateways.py               ## Concrete adapter implementations
|  |--tasks.py                  ## Celery tasks and pipeline orchestration
|  |--urls.py                   ## URL routing
|  |--views.py                  ## HTTP handlers (thin, delegate to use cases)
```

No downward dependencies between layers:

```
        -------------------
        |                 |
        | INFRASTRUCTURE  |
        |                 |
        | --------------- |
        | |             | |
        | | APPLICATION | |
        | |             | |
        | | ----------  | |
        | | |        |  | |
        | | | DOMAIN |  | |
        | | |        |  | |
        | | ----------  | |
        | |             | |
        | | APPLICATION | |
        | |             | |
        | --------------- |
        |                 |
        | INFRASTRUCTURE  |
        |                 |
        -------------------
```

---

## Domain Interfaces (Ports)

ABCs that define what infrastructure must implement. Use cases depend on these, never on concrete gateways. A generic type parameter on the AI interface lets the adapter declare its SDK client type without the domain knowing about it.

_domain/interfaces.py_

```python
from abc import ABC, abstractmethod
from typing import Any, Generic, TypeVar

from django.db.backends.utils import CursorWrapper


class ExternalAPIInterface(ABC):
    """Port for any external HTTP API."""

    @abstractmethod
    def get(self, params: dict, timeout: int = 20) -> Any: ...

    @abstractmethod
    async def async_get(self, params: dict) -> Any: ...

    @abstractmethod
    async def close(self) -> None: ...


class SqlExecutorInterface(ABC):
    """Port for running raw SQL files."""

    @abstractmethod
    def execute(self, filename: str, **kwargs: Any) -> CursorWrapper: ...

    @abstractmethod
    def fetchall(self, filename: str, **kwargs: Any) -> list[tuple]: ...


ClientType = TypeVar("ClientType")


class AIProviderInterface(ABC, Generic[ClientType]):
    """Port for AI/LLM providers. Generic over the SDK client type."""
    _client: ClientType

    @abstractmethod
    def run(self, instructions: str, prompt: str) -> str: ...
```

---

## Value Objects & DTOs

Pydantic `BaseModel` classes used as typed value objects and DTOs. The rest of the domain works with these instead of raw dicts.

_domain/types.py_

```python
from pydantic import BaseModel


class AIInitParams(BaseModel):
    api_key: str
    ai_model: str


class AIRequestParams(BaseModel):
    instructions: str
    prompt: str
```

---

## Domain Entities (dual-manager pattern)

Django models are the domain entities. Every model has two managers: `objects` (the default, needed for admin and migrations) and `repository` (a custom manager with domain-specific queries). You still have the full ORM when you need it, but day-to-day code goes through the repository.

_domain/models.py_

```python
from django.db import models
from myapp.domain.managers import OrderRepository


class Order(models.Model):
    # ... fields

    objects = models.Manager()          # default (admin, migrations)
    repository = OrderRepository()      # domain queries
```

---

## Repository Pattern

Custom Django managers and querysets that work as repositories. The `Manager` delegates to a custom `QuerySet` with chainable domain filters. The rest of the codebase calls `Order.repository.active()` instead of scattering ORM filters everywhere.

_domain/managers.py_

```python
from django.db import models


class OrderQuerySet(models.QuerySet):
    def active(self):
        return self.filter(is_cancelled=False, is_archived=False)

    def for_customer(self, customer_id: int):
        return self.filter(customer_id=customer_id)


class OrderRepository(models.Manager):
    def get_queryset(self) -> OrderQuerySet:
        return OrderQuerySet(self.model, using=self._db)

    def active(self) -> OrderQuerySet:
        return self.get_queryset().active()

    def bulk_upsert(self, orders: list) -> None:
        orders = list({o.id: o for o in orders}.values())
        self.bulk_create(
            orders,
            update_conflicts=True,
            update_fields=["status", "total", "updated_at"],
            unique_fields=["id"],
        )
```

---

## Factory Pattern

Factories turn raw external data (API JSON, CSV rows, etc.) into domain entities. Each is a static class with `from_*_data()` methods so entity construction lives in one place, not scattered across use cases.

_domain/factories.py_

```python
from typing import Any
from myapp.domain.models import Order


class OrderFactory:
    @staticmethod
    def from_api_data(data: dict[str, Any]) -> Order:
        return Order(
            external_id=data["id"],
            customer_id=data["customerId"],
            total=data["total"],
            status=data.get("status", "pending"),
        )
```

---

## Use Cases

Use cases sit in the application layer and take domain interfaces as constructor arguments. Infrastructure injects the concrete gateways through the DI container. The use case depends on `ExternalAPIInterface`, never on `SomeConcreteGateway` directly.

_application/use_cases/sync_orders.py_

```python
from myapp.domain.interfaces import ExternalAPIInterface, SqlExecutorInterface
from myapp.domain.factories import OrderFactory
from myapp.domain.models import Order


class SyncOrders:
    def __init__(
        self,
        api: ExternalAPIInterface,
        sql: SqlExecutorInterface,
    ):
        self.api = api
        self.sql = sql

    def run(self, since_id: int = 0) -> int:
        data = self.api.get(params={"since": since_id})
        orders = [OrderFactory.from_api_data(d) for d in data["results"]]
        Order.repository.bulk_upsert(orders)
        return len(orders)
```

---

## Application Policies

Application-level rules that don't belong in the domain but affect how operations run. This one figures out worker counts and chunk sizes based on task type and CPU cores. Celery tasks and use cases both reference it.

_application/policies/worker_allocation.py_

```python
from enum import Enum
from multiprocessing import cpu_count


class TaskType(Enum):
    CPU_BOUND = "cpu_bound"
    IO_BOUND = "io_bound"


class WorkerAllocationPolicy:
    @staticmethod
    def optimal_workers(task_type: TaskType, data_size: int | None = None) -> int:
        cores = cpu_count()
        if task_type == TaskType.CPU_BOUND:
            optimal = max(cores - 1, 2)
        else:
            optimal = max(cores * 2, 10)

        if data_size is not None:
            optimal = min(optimal, max(data_size // 100, 1))
        return optimal
```

---

## Infrastructure Gateways (Adapters)

Concrete implementations of the domain ports. Each gateway inherits from its interface and does the actual HTTP calls, SQL execution, or SDK work. Domain and application layers never import these directly -- they get wired in through the DI container.

_infrastructure/gateways.py_

```python
from django.db import connection
from openai import OpenAI

from myapp.domain.interfaces import (
    ExternalAPIInterface,
    SqlExecutorInterface,
    AIProviderInterface,
)


class SomeAPIGateway(ExternalAPIInterface):
    def __init__(self, api_endpoint: str):
        self.endpoint = api_endpoint
        # ... session setup, retry policy, proxy config

    def get(self, params: dict, timeout: int = 20):
        # ... sync HTTP GET via requests
        ...

    async def async_get(self, params: dict):
        # ... async HTTP GET via aiohttp
        ...

    async def close(self): ...


class SqlExecutorGateway(SqlExecutorInterface):
    """Discovers, caches, and runs .sql files via Django's connection."""

    def execute(self, filename: str, **kwargs):
        sql = self._load_and_format(filename, **kwargs)
        with connection.cursor() as cursor:
            cursor.execute(sql)
            return cursor

    def fetchall(self, filename: str, **kwargs):
        sql = self._load_and_format(filename, **kwargs)
        with connection.cursor() as cursor:
            cursor.execute(sql)
            return list(cursor.fetchall())


class OpenAIGateway(AIProviderInterface[OpenAI]):
    def __init__(self, api_key: str, model: str):
        self._client = OpenAI(api_key=api_key)
        self._model = model

    def run(self, instructions: str, prompt: str) -> str:
        resp = self._client.responses.parse(
            model=self._model, instructions=instructions,
            input=prompt, max_output_tokens=30000,
        )
        return resp.output_text
```

---

## DI Container

The composition root. Wires concrete gateways into use case constructors. Celery tasks and views call factory methods here instead of building use cases by hand, so infrastructure imports stay in one file.

_infrastructure/di.py_

```python
from myapp.application.use_cases.sync_orders import SyncOrders
from myapp.infrastructure.gateways import SomeAPIGateway, SqlExecutorGateway


class Container:
    @staticmethod
    def get_order_syncer() -> SyncOrders:
        return SyncOrders(
            api=SomeAPIGateway(api_endpoint="orders"),
            sql=SqlExecutorGateway(),
        )

    # one factory method per use case
```

---

## App Configuration

The one line that makes this whole layout work with Django: `models_module` tells Django to look for models in `myapp.domain.models` instead of the default `myapp.models`. Migrations, admin, ORM -- everything works as usual.

_infrastructure/apps.py_

```python
from django.apps import AppConfig


class MyAppConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    models_module = "myapp.domain.models"
    name = "myapp"
```
