# AI Agent Setup (Project Rules for Claude / Cursor)

How we give AI assistants the same starting context a new engineer would
get on day one. Two files do the work. `CLAUDE.md` at the repo root tells
the agent what the project is: stack, apps, env vars, key files, and the
must-follow rules. Next to it, a `.rules/` folder holds the coding
standards as one short file per topic (code style, clean code, Django
patterns, docstrings, logging, security, testing, app structure). The
agent reads `CLAUDE.md` every session; the topic files get pulled in when
the current task touches them. Same files work for Claude Code, Cursor,
Copilot, and JetBrains AI.

The example below is the real setup we use on one of our Django SaaS
projects. Stack: Django 5, PostgreSQL, Celery + Redis, an external
identity provider for user auth, and an external payment processor for
billing.

------------------------------------------------------------------------

## Repo Layout

```
example-project/
  CLAUDE.md                # project context + must-follow rules (loaded by Claude)
  .rules/
    README.md              # index of topic files
    app-structure.md       # standard layout for new Django apps
    clean-code.md          # SOLID, DRY, KISS, naming, comments
    code-style.md          # PEP 8, type hints, imports, formatting
    django.md              # views, models, services, URLs, migrations
    docstrings.md          # Google-style docstrings with examples
    logging.md             # levels, what to log, what to avoid
    security.md            # secrets, encryption, auth, CSRF, throttling
    testing.md             # philosophy, structure, naming, coverage
```

`CLAUDE.md` is the entry point. `.rules/` is the standards library it
points to. One file per topic keeps each rule file short enough that a
human will actually read it end to end, and lets the agent pull only what
the current task needs.

------------------------------------------------------------------------

## CLAUDE.md (project entry point)

Single file, around 120 lines. It answers the questions a new contributor
asks before touching anything: what is this product, what is the stack,
which apps own what, which files matter, which env vars are required, how
do I run it locally, what must I never break.

_CLAUDE.md_

````markdown
# Example App — Project Context & Agent Rules

## What this project is
A Django-based SaaS application with a marketplace, subscription billing, and
several third-party integrations. The platform handles discovery, subscriptions,
billing, outbound webhook fan-out, and user identity.

## Architecture
- **Framework**: Django 5.x monolith (single repo, multiple apps)
- **Database**: managed PostgreSQL in prod; SQLite for local dev only
- **Auth**: external identity provider (JWT + session) for regular users; Django session auth for admin staff
- **Payments**: external payment processor (subscriptions + one-time)
- **Async**: Redis + Celery (task queue, scheduled jobs)
- **Delivery**: outbound webhook fan-out to partner endpoints
- **Data pipeline**: Nightly ETL pulling source data into the primary database

## Apps
| App | Responsibility |
|---|---|
| `accounts` | User profile, identity sync, onboarding, support tools |
| `catalog` | Listings, discovery, checkout |
| `billing` | Subscription lifecycle, payment webhooks |
| `tiers` | Membership tiers |
| `integrations` | Partner OAuth, connection management, MFA gate |
| `dashboard` | Item management for creators |
| `rankings` | Performance rankings |
| `community` | Discussion forum |
| `notifications` | In-app + email notifications |
| `activity` | Activity feed |
| `metrics` | Performance metrics ingestion |
| `deployments` | Cloud deployment of user-created assets |
| `admin_2fa` | TOTP 2FA for Django admin staff (in-house) |

## Admin 2FA (in-house feature)
TOTP-based 2FA for Django admin, separate from the external identity provider.
- Model: `admin_2fa.AdminTOTP` — Fernet-encrypted secret, per staff user
- Middleware: `admin_2fa.Admin2FAMiddleware` — gates all `/admin/` routes
- Throttle: 5 failed attempts → 5-minute lockout (per user, via Django cache)
- Required env var: `ADMIN_TOTP_ENCRYPTION_KEY` (Fernet key, generate with `Fernet.generate_key()`)

## Key files
- `project/settings.py` — all settings, env-gated
- `project/urls.py` — root URL config
- `accounts/utils_http.py` — shared HTTP client, circuit breaker, safe_request
- `accounts/mfa.py` — external-provider MFA enforcement (use this, not legacy modules)
- `catalog/api/listings.py` — listing creation API (owner bound server-side)
- `admin_2fa/consts.py` — session keys, messages (no magic strings in views)

## Required environment variables (local)
See `.env.dev`. Key ones:
- `SECRET_KEY` — Django secret key
- `AUTH_PROVIDER_SECRET_KEY`, `AUTH_PROVIDER_PUBLIC_KEY`, `AUTH_PROVIDER_API_URL`
- `ADMIN_TOTP_ENCRYPTION_KEY` — Fernet key for admin 2FA secrets
- `INTEGRATION_ENCRYPTION_KEY` — Fernet key for partner credentials
- `DATABASE_URL` — set to `sqlite:///db.sqlite3` locally

## Local setup
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.dev.example .env.dev  # fill in your values
python manage.py migrate
python manage.py runserver
```
````

The "Must-follow rules" block at the bottom of `CLAUDE.md` lists the
things that have bitten us before, in compact form. The agent treats them
as hard constraints.

_CLAUDE.md (continued)_

````markdown
## Must-follow rules

### Never commit secrets
- `.env.prod` and `.env.dev` are gitignored. Never write credentials into source files.
- Use `os.getenv('KEY')` in `settings.py`. Never hardcode fallback secrets.

### Services pattern
Business logic goes in a `services/` module or `services_*.py` file, not inside views.
- `integrations/services/` — partner connection/OAuth
- `catalog/services_*.py` — checkout, deployment, data sync
- `accounts/services/` — identity provider sync, third-party API calls
- Views are thin: validate input → call service → return response

### No magic strings
- Extract repeated strings, session keys, and URL names to a `consts.py` in the app.
- Use named URLs (`reverse('app:name')`) instead of hardcoded paths.

### External HTTP calls
- Always set `timeout=(5, 10)` (connect, read) on every `requests.*` call.
- Use `@circuit_breaker("service_name")` from `accounts/utils_http.py` on any function
  that makes outbound calls to a third-party API.

### No HTML in Python
- Error pages and responses belong in templates, not in `mark_safe(...)` strings.
- CSS belongs in `static/<app>/` files, linked with `{% load static %}` — never inline `<style>`.
- Inline `<script>` blocks belong in `static/js/` files, referenced with `{% static %}`.

### Django conventions
- New models need a migration: always run `makemigrations` before committing.
- Use `select_related` for FK traversal, `prefetch_related` for M2M / reverse-FK.
- Replace `.count() > 0` with `.exists()`. Use `bulk_create` for multi-row inserts.
- Never put side-effects (image resize, email send, API calls) inside `Model.save()`.

### Security
- Never use `@csrf_exempt` on views that mutate state unless behind API key auth.
- `@staff_member_required` on all diagnostic/debug endpoints.
- Never re-introduce `ALLOWED_HOSTS = ['*']`.
- `LISTING_API_OWNER_USERNAME` must be set server-side; never resolve owner from request body.
- Admin access requires both `is_staff` or `is_superuser` — check both, not just one.

### Branch strategy
- Feature branches off `dev`. PR into `dev`. `dev` → `main` after review.
- Never push directly to `main`.
- Migrations must pass `python manage.py makemigrations --check --dry-run` before merge.

## Run before pushing
```bash
python manage.py check
python manage.py makemigrations --check --dry-run
```

## Coding standards
See `.rules` for code style, docstrings, testing, and clean code principles.
````

------------------------------------------------------------------------

## .rules/ directory (coding standards index)

A short README that lists every topic file. The agent uses it as a lookup
table.

_.rules/README.md_

```markdown
# Coding Rules Index

Standards that apply to all new code in this project.
Each file covers one topic. Read before starting a feature.

| File | Topic |
|---|---|
| [code-style.md](code-style.md) | PEP 8, type hints, imports, formatting |
| [clean-code.md](clean-code.md) | SOLID, DRY, KISS, naming, comments |
| [docstrings.md](docstrings.md) | Google-style docstrings with examples |
| [django.md](django.md) | Views, models, services, URLs, migrations |
| [security.md](security.md) | Secrets, encryption, auth, CSRF, throttling |
| [testing.md](testing.md) | Test philosophy, structure, naming, coverage |
| [git.md](git.md) | Branch strategy, commits, PR checklist |
| [app-structure.md](app-structure.md) | Standard layout for new Django apps |
| [logging.md](logging.md) | Logging levels, what to log, what to avoid |
```

------------------------------------------------------------------------

### .rules/app-structure.md — standard Django app layout

Every new app uses the same skeleton, so when the agent adds a new model
or service it already knows which file to open.

_.rules/app-structure.md_

````markdown
# App Structure

Standard layout for every new isolated Django app.

```
<app>/
  __init__.py
  apps.py
  consts.py          # session keys, messages — no magic strings in views/middleware
  models.py          # data layer only, no business logic
  utils.py           # pure helpers: encryption, formatting, etc.
  services.py        # all business logic (or services/ package for larger apps)
  mixins.py          # reusable view mixins, one responsibility each
  views.py           # thin CBVs: parse input → call service → return response
  urls.py            # named routes, app_name always set
  middleware.py      # if the app needs request interception
  throttle.py        # if the app needs rate limiting
  admin.py           # ModelAdmin registrations
  migrations/
  templates/
    <app>/           # namespaced to avoid collisions
      *.html
  tests/
    __init__.py
    test_views.py
    test_services.py
    test_models.py
```

## Rules
- Each file has one clear responsibility — see descriptions above.
- Never put business logic in `views.py` or `models.py`.
- New apps must be added to `INSTALLED_APPS` in `settings.py`.
- Migrations live inside the app. Never share a migrations folder across apps.
- Templates and static files are namespaced under the app name to avoid collisions.
````

------------------------------------------------------------------------

### .rules/clean-code.md — SOLID, DRY, KISS

The principles we want the agent to apply when writing new code. The KISS
section is here because LLMs love to over-engineer for hypothetical future
requirements; this file tells them to stop.

_.rules/clean-code.md_

```markdown
# Clean Code

## KISS — Keep It Simple
- Solve the problem at hand. Do not design for hypothetical future requirements.
- Three similar lines is better than a premature abstraction.
- No half-finished implementations or feature flags for unbuilt features.
- If you need to explain what code does in a comment, simplify the code instead.

## DRY — Don't Repeat Yourself
- Extract repeated logic to a service, utility, or mixin.
- Extract repeated strings to `consts.py`.
- Wait until you have **three real repetitions** before extracting — premature DRY is its own mess.

## SOLID

### Single Responsibility
- One class or function does one thing.
- Views validate input and delegate — nothing else.
- Services contain business logic — no HTTP context, no `request` object.
- Models are data containers — no business logic, no side-effects in `save()`.

### Open / Closed
- Extend behaviour via new classes or services, not by editing core logic.
- New authentication provider? New class. Not a new `if` branch in existing auth code.

### Liskov Substitution
- Subclasses must honour the contract of their parent.
- Mixins must not break the CBV dispatch flow.

### Interface Segregation
- Keep interfaces narrow. Don't force callers to depend on methods they don't use.
- A service that does three unrelated things should be three services.

### Dependency Inversion
- Depend on abstractions (service layer), not on concrete implementations in views.
- Views call services. Services call models or external clients. Not the other way around.

## Naming
- Names should be self-documenting. If you need a comment to explain a name, rename it.
- Booleans: `is_`, `has_`, `can_` prefix. (`is_enabled`, `has_backup_codes`).
- Functions: verb + noun. (`get_cipher`, `record_failure`, `verify_code`).
- Classes: noun, PascalCase. (`AdminTOTP`, `SetupView`, `TOTPService`).
- No abbreviations unless universally understood (`url`, `id`, `pk` are fine; `usrnm` is not).

## Comments
- Default: no comments.
- Only add one when the **WHY** is non-obvious: a hidden constraint, a workaround, a subtle invariant.
- Never explain WHAT the code does — names do that.
- Never reference the current task, fix, or issue number in source code (put it in the commit/PR).
- Delete dead code. Don't comment it out.

## Functions & methods
- Prefer early returns over deeply nested conditionals.
- A function should fit on one screen (~30 lines max).
- Max 3 parameters. Use a dataclass or dict for more.
- No side-effects in functions whose names suggest they are queries (`get_*`, `is_*`, `has_*`).
```

------------------------------------------------------------------------

### .rules/code-style.md — formatting, imports, type hints

_.rules/code-style.md_

````markdown
# Code Style

## General
- Python 3.12. Django 5.x.
- Max line length: **100 characters**.
- 4 spaces for indentation. Never tabs.
- One blank line between methods; two between top-level definitions.

## Imports
Order: stdlib → third-party → Django → local app. One blank line between groups.
```python
import os
from typing import Optional

import pyotp
from cryptography.fernet import Fernet

from django.db import models
from django.conf import settings

from .utils import get_cipher
from . import consts
```
- No wildcard imports (`from module import *`).
- Import the name, not the module, unless there is a collision or clarity reason.

## Type hints
All function signatures must have type hints. Return types are mandatory.
```python
def get_secret(self) -> str: ...
def record_failure(user: User) -> int: ...
def verify(secret: str, code: str) -> bool: ...
```
- Avoid `Any`. Be specific.
- Use `Optional[X]` or `X | None` (prefer `X | None` in Python 3.10+).
- Use `list[str]` not `List[str]` (built-in generics since Python 3.9).

## Strings
- Prefer f-strings over `.format()` or `%`.
- No magic strings in views, middleware, or services — use `consts.py`.
- Use named URLs (`reverse('app:name')`) instead of hardcoded paths.

## Formatting
- Use Black-compatible formatting (enforced in CI).
- Trailing commas in multi-line collections.
- No semicolons.
````

------------------------------------------------------------------------

### .rules/django.md — Django-specific patterns

Views, models, querysets, services, URLs, templates, migrations, settings,
external HTTP calls. The rule that matters most: business logic goes in
`services.py`, never in a view.

_.rules/django.md_

```markdown
# Django Patterns

## Views
- Use **class-based views (CBVs)** for all new views.
- Mixins go in `mixins.py`. One responsibility per mixin.
- Views are thin: parse input → call service → return response. No business logic.
- Use `reverse()` or named URLs in `redirect()`. Never hardcode URL strings.
- Always validate `next` redirect URLs with `url_has_allowed_host_and_scheme`.

## Models
- No business logic or side-effects in `Model.save()`.
- Encryption and other helpers go in `utils.py`, not inline in the model.
- Always set `verbose_name` (and `verbose_name_plural`) in `Meta`.
- Use `auto_now_add=True` for `created_at`, `auto_now=True` for `updated_at`.
- Always provide `related_name` on `ForeignKey` and `OneToOneField`.
- New models **must** have a migration before merging.

## Querysets
- Use `.exists()` instead of `.count() > 0`.
- Use `select_related` for FK/OneToOne traversal.
- Use `prefetch_related` for M2M and reverse FK.
- Use `bulk_create` and `bulk_update` for multi-row operations.
- Never query inside a loop — build the queryset first.

## Services
- All business logic lives in `services.py` or a `services/` package.
- Services are plain Python classes or module-level functions.
- No `request` object, no HTTP context inside services.
- Service methods must be independently testable without a running server.

## URLs
- Always set `app_name` in `urls.py` to enable namespaced URL reversal.
- Keep `urlpatterns` ordered: specific paths before catch-alls.
- Use `include()` for app-level URL configs.

## Templates & static files
- CSS in `static/<app>/`, linked with `{% load static %}` and `<link>` tag.
- No inline `<style>` blocks. No `<script>` blocks in templates.
- JS in `static/<app>/js/`. Use `{% static %}` for all asset paths.
- No HTML in Python (`mark_safe` is banned except in admin form widgets).

## Migrations
- Run `python manage.py makemigrations --check --dry-run` before every commit.
- Never edit a migration that has already been merged to `dev` or `main`.
- Squash migrations only on `dev` before a release, never on feature branches.

## Settings
- All settings read from environment via `os.getenv()`. No hardcoded secrets or fallback credentials.
- Feature flags and per-environment config must be env-gated.
- New settings go at the end of their logical group (auth, external services, etc.).

## External HTTP calls
- Always set `timeout=(5, 10)` (connect, read) on every `requests.*` call.
- Use `@circuit_breaker("service_name")` from `accounts/utils_http.py` for third-party APIs.
- Use `external_session()` context manager for retry + timeout on non-critical calls.
```

------------------------------------------------------------------------

### .rules/docstrings.md — Google-style with examples

_.rules/docstrings.md_

````markdown
# Docstrings

Use **Google style** for all docstrings.

## When to write one
- All public classes — always.
- All public methods and functions — always.
- Private methods (`_name`) — only when the logic is non-obvious.
- Simple getters, setters, and one-liner properties — skip it.

## Class docstring
One-line summary. If the class needs more explanation, add a blank line then a paragraph.
```python
class SetupView(StaffRequiredMixin, View):
    """Handles TOTP 2FA enrollment for admin staff.

    GET renders a QR code and manual secret for the authenticator app.
    POST confirms the setup by verifying a submitted TOTP code.
    """
```

## Method / function docstring
```python
def verify(secret: str, code: str) -> bool:
    """Verifies a TOTP code against the given secret.

    Args:
        secret: Base32-encoded TOTP secret.
        code: 6-digit code submitted by the user.

    Returns:
        True if the code is valid within the allowed time window.

    Raises:
        ValueError: If the secret is empty or malformed.
    """
```

## Sections
| Section | Use when |
|---|---|
| `Args:` | Function has parameters worth explaining |
| `Returns:` | Return value is non-obvious from the type hint |
| `Raises:` | Function raises exceptions the caller must handle |
| `Example:` | Usage is non-obvious and a snippet helps |

## Rules
- First line is a short imperative summary. No period.
- Blank line before sections.
- `Raises:` only for exceptions the **caller** is expected to handle.
- Don't restate the type hint in the arg description — add meaning instead.
  - Bad: `code: str — the code string`
  - Good: `code: 6-digit TOTP code submitted by the user`
- Never write multi-paragraph docstrings for trivial methods.
````

------------------------------------------------------------------------

### .rules/logging.md — levels and what to log

_.rules/logging.md_

````markdown
# Logging

## Levels
| Level | When to use |
|---|---|
| `DEBUG` | Verbose internal state — local dev only, never in prod paths |
| `INFO` | Normal significant events: user signed up, payment processed, job completed |
| `WARNING` | Unexpected but recoverable: missing optional config, deprecated usage |
| `ERROR` | Something failed and needs attention: uncaught exception, third-party failure |
| `CRITICAL` | System is broken: database unreachable, encryption key missing |

## Setup
```python
import logging
logger = logging.getLogger(__name__)  # always use __name__, never a hardcoded string
```

## What to log
- Authentication events: login, logout, 2FA success/failure, lockout triggered.
- Payment events: charge created, webhook received, subscription changed.
- External API failures: identity provider, payment processor, partner connections.
- Background job start/finish/failure.
- Security events: throttle triggered, invalid token, permission denied.

## What NOT to log
- Passwords, tokens, secrets, or any credential — ever.
- PII unless required by compliance and explicitly approved (mask or hash instead).
- High-frequency routine events (e.g. every DB query, every cache hit).
- Stack traces at `INFO` — use `logger.exception(...)` which logs at ERROR with traceback.

## Format
```python
# Bad — no context
logger.error("Login failed")

# Good — who, what, why
logger.warning("Admin 2FA throttle triggered", extra={"user_id": user.pk, "attempts": count})

# For exceptions — use exception() to capture traceback automatically
try:
    ...
except Exception:
    logger.exception("Failed to decrypt TOTP secret for user %s", user.pk)
```

## Rules
- Never use `print()` for logging in production code — use the `logging` module.
- Never log inside tight loops — aggregate and log the result.
- Use `extra={}` for structured fields rather than embedding data in the message string.
- `logger.exception()` automatically captures the current traceback — no need to pass `exc_info=True`.
````

------------------------------------------------------------------------

### .rules/security.md — secrets, encryption, auth

_.rules/security.md_

```markdown
# Security

## Secrets & environment
- Never hardcode secrets, API keys, or credentials in source code.
- Never use `SECRET_KEY` as an encryption key — it can rotate; derive nothing from it.
- Use a **dedicated Fernet key per domain** (e.g. `ADMIN_TOTP_ENCRYPTION_KEY`, `INTEGRATION_ENCRYPTION_KEY`).
  Generate with: `from cryptography.fernet import Fernet; Fernet.generate_key()`
- `.env.dev` and `.env.prod` are gitignored. Never commit them.

## Authentication & access control
- Django admin access requires `is_staff or is_superuser` — check both, they are independent flags.
- Use `@staff_member_required` or `StaffRequiredMixin` on all admin-facing views.
- Use `@login_required` or `LoginRequiredMixin` on all user-facing views.
- Never use `@csrf_exempt` on views that mutate state unless the endpoint is behind API key auth.
- Never re-introduce `ALLOWED_HOSTS = ['*']`.

## Throttling
- Any endpoint that accepts user-submitted codes, passwords, or credentials must be throttled.
- Use per-user keys for throttling auth endpoints (not per-IP — IPs are easy to rotate).
- Standard: 5 attempts → 5-minute lockout. Adjust per sensitivity.
- Clear the throttle counter on successful auth.

## Input validation
- Validate at system boundaries: user input, webhook payloads, external API responses.
- Never trust `request.POST` or `request.GET` values without validation.
- Use Django forms or DRF serializers for structured input validation.
- Always validate `next` redirect parameters with `url_has_allowed_host_and_scheme`.

## Redirects
- Never redirect to a URL taken directly from user input without validation.
- Use `url_has_allowed_host_and_scheme(url, allowed_hosts=...)` before redirecting.

## API security
- `LISTING_API_OWNER_USERNAME` must be resolved server-side. Never from request body.
- Webhook endpoints must verify signatures (payment provider, identity provider, etc.) before processing payload.
- API keys must be transmitted in headers, never in query strings.

## Encryption
- Use Fernet (symmetric) for secrets stored at rest (TOTP secrets, partner credentials).
- Never roll your own crypto. Use `cryptography` library primitives.
- Rotate encryption keys by re-encrypting data — not by changing the key in place.
```

------------------------------------------------------------------------

### .rules/testing.md — philosophy, structure, naming

_.rules/testing.md_

````markdown
# Testing

## Philosophy
- Test **behaviour**, not implementation. Tests must survive internal refactoring.
- Prefer integration tests that hit a real test database over mocking the ORM.
- Unit test pure logic (services, utils) in isolation.
- Never mock what you own — mock only external APIs, third-party services, and system calls.
- All new features ship with tests. No merging untested code.

## Structure
```
tests/
  test_<feature>.py         # project-wide integration tests

<app>/
  tests/
    test_<module>.py        # app-level tests (test_views.py, test_services.py, etc.)
```

## Writing tests

### Naming
- Test file: `test_<module>.py` (mirrors the module under test).
- Test class: `<Feature>Tests` or `<ClassName>Tests`.
- Test method: `test_<behaviour>_<condition>` — describe what is being tested and under what condition.
  - Good: `test_verify_rejects_expired_code`, `test_throttle_locks_after_five_failures`
  - Bad: `test_verify_2`, `test_case_3`

### Structure (Arrange / Act / Assert)
```python
def test_throttle_locks_after_five_failures(self):
    # Arrange
    for _ in range(5):
        throttle.record_failure(self.user)

    # Act
    locked = throttle.is_locked(self.user)

    # Assert
    self.assertTrue(locked)
```

### Rules
- One logical assertion per test. Multiple `assert` calls are fine if they test the same behaviour.
- Use `setUp` for per-test fixtures; use `setUpTestData` for expensive shared fixtures.
- Test the happy path, edge cases, and failure paths for every feature.
- Do not test Django internals (form rendering, ORM basics) — test your code.
- Use `self.assertRedirects`, `self.assertContains`, `self.assertEqual` over raw `assert`.

## Mocking
- Mock at the boundary: mock the HTTP call, not the service that wraps it.
- Use `unittest.mock.patch` as a context manager or decorator — not as a class attribute.
- Assert mock calls when the call itself is the behaviour under test.
- Reset mocks between tests — prefer `patch` as a decorator so teardown is automatic.

## Coverage
- Aim for coverage on all service and utility functions.
- Views: test at least the happy path, auth-rejected path, and validation-failure path.
- Do not chase 100% coverage at the expense of test quality.

## Running tests
```bash
python manage.py test                        # all tests
python manage.py test admin_2fa             # single app
python manage.py test admin_2fa.tests.test_views  # single file
```
````

------------------------------------------------------------------------

## Notes

Every AI client picks up
settings automatically because it sits at the repo root. No per-tool
configuration. The project context lives in one file; the coding
standards live in `.rules/` and are mostly reusable across our other
Django projects with small edits.

Each topic file stays under 100 lines, which keeps the agent's context
cost low when only one or two of them apply to a task. The rules we care
about most are written down rather than left for the agent to infer from
existing code: security boundaries, the services pattern, the branch
model, throttling thresholds.

Same files double as onboarding docs for new engineers. Anything we tell
the agent, humans on the team are expected to follow too.
