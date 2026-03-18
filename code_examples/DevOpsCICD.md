# DevOps/CI/CD Setup (Docker + GitHub Actions + Ansible)

A production-ready DevOps setup for a Django/React application. Docker Compose orchestrates the services -- Django (Gunicorn), Celery worker and beat, PostgreSQL, Redis, and Caddy as a reverse proxy with automatic HTTPS. When code is pushed, GitHub Actions runs a CI pipeline (Ruff linting, pip-audit, ESLint, tests with migration conflict detection), then deploys via SSH using a reusable workflow. The deploy script on the server pulls the latest code, builds Docker images, waits for PostgreSQL, runs `manage.py check --deploy` as a health check, and automatically rolls back to the previous commit and image on failure. Fresh servers are provisioned with Ansible, which hardens SSH, configures UFW and fail2ban, installs Docker, sets up monitoring agents, clones the repo, injects secrets into `.env`, and triggers the initial deployment. A separate development compose file (`compose.dev.yml`) provides hot-reload for Django, Celery (via watchfiles), and Vite, along with MailHog for email testing and a live MkDocs documentation server.

For Dockerfile and Docker Compose examples, see the existing [Dockerfile](Dockerfile-example1.md) and [Docker Compose](Docker-compose-example1.md) code samples. The production compose for this setup uses YAML anchors for DRY configuration across Django, Celery worker, and Celery beat, journald logging on all containers, a `build` profile for frontend asset compilation (Vite), and Caddy as the reverse proxy. The Django Dockerfile uses Poetry with layer caching (dependency files copied before application code), a non-root user with gosu for runtime privilege dropping, and a conditional `ENVIRONMENT` build arg to omit dev dependencies in production.

## Entrypoint Script

A single entrypoint supporting multiple commands -- `prod` starts Gunicorn with collectstatic and migrations, `dev` starts the development server, `celery` and `celery-dev` (hot-reload via watchfiles) run Celery, and `test` runs pytest. All commands wait for PostgreSQL readiness before starting, and all processes run as `nonroot` via gosu.

_docker/django/entrypoint_

```bash
#!/bin/bash
set -o errexit
set -o pipefail
set -o nounset

PORT="${APP_PORT:-8000}"

# Fix permissions on mounted volumes (running as root)
if [ "$(id -u)" = "0" ]; then
    mkdir -p /data/staticfiles /data/media
    chown -R nonroot:nonroot /data/staticfiles
    chown nonroot:nonroot /data /data/media
fi

postgres_ready() {
gosu nonroot poetry run python << END
import sys, psycopg2
try:
    psycopg2.connect(
        dbname="${POSTGRES_DB}", user="${POSTGRES_USER}",
        password="${POSTGRES_PASSWORD}", host="${POSTGRES_HOST}",
        port="${POSTGRES_PORT}",
    )
except psycopg2.OperationalError:
    sys.exit(-1)
sys.exit(0)
END
}

wait_for_postgres() {
    count=0
    until postgres_ready; do
        count=$((count+1))
        if [ $count -eq 10 ]; then
            >&2 echo 'PostgreSQL wait timeout. Exiting...'
            exit 1
        fi
        sleep 2
    done
    >&2 echo 'PostgreSQL is available'
}

case "$1" in
    dev)
        wait_for_postgres
        gosu nonroot poetry run python manage.py migrate
        exec gosu nonroot poetry run python manage.py runserver 0.0.0.0:"${PORT}"
    ;;
    prod)
        wait_for_postgres
        gosu nonroot poetry run python manage.py collectstatic --noinput
        gosu nonroot poetry run python manage.py migrate
        exec gosu nonroot poetry run gunicorn config.wsgi \
            --bind 0.0.0.0:"${PORT}" --chdir=/opt/project/src
    ;;
    celery)
        wait_for_postgres
        exec gosu nonroot poetry run celery -A config "${@:2}"
    ;;
    celery-dev)
        wait_for_postgres
        exec gosu nonroot poetry run watchfiles \
            celery.__main__.main --args "-A config ${*:2}"
    ;;
    test)
        export DJANGO_SETTINGS_MODULE=config.settings.test
        exec gosu nonroot poetry run pytest
    ;;
    # ... bash, manage, shell, add commands omitted
esac
```

## CI Workflow

Runs on pull requests to `develop`, `staging`, and `main`, and can be called by deploy workflows via `workflow_call`. Concurrency control cancels in-progress runs on the same PR to save Actions minutes. The pipeline also includes a migration conflict detection job (omitted below) that catches colliding migrations between PR and base branch. Tests are skipped on PRs to `develop` (lint is sufficient) but always run for staging and production. A similar `lint-frontend` job runs ESLint, Prettier, and `npm audit`.

_.github/workflows/ci.yml_

```yaml
name: CI

on:
  workflow_call: {}
  pull_request:
    branches: [develop, staging, main]
    paths-ignore:
      - 'docs/**'
      - '*.md'
      - 'ansible/**'

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  # ... check_migration_conflicts job omitted
  #     (detects if PR adds migrations that collide with base branch)

  lint-backend:
    name: Lint Python
    runs-on: ubuntu-latest
    timeout-minutes: 5
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: pip
      - run: pip install ruff pip-audit poetry
      - name: Ruff check
        working-directory: ./src
        run: ruff check .
      - name: Ruff format check
        working-directory: ./src
        run: ruff format --check .
      - name: Audit Python dependencies
        continue-on-error: true
        working-directory: ./src
        run: >-
          poetry export -f requirements.txt --without-hashes -o /tmp/requirements.txt
          && pip-audit -r /tmp/requirements.txt

  # ... lint-frontend job omitted (ESLint, Prettier, npm audit)

  test-backend:
    name: Test Django Backend
    # Skip on develop PRs to save Actions minutes;
    # tests still run for staging/production via workflow_call
    if: ${{ github.base_ref != 'develop' && !failure() && !cancelled() }}
    needs: [lint-backend, check_migration_conflicts]
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v5
      - run: cp dev.env .env
      - run: docker compose -f compose.dev.yml build
      - run: docker compose -f compose.dev.yml run --rm django test
      - name: Cleanup
        if: always()
        run: docker compose -f compose.dev.yml down -v

  # ... test-frontend job omitted (npm ci, npm test, npm run build)
```

## Deploy Script

Runs on the server via SSH from GitHub Actions. Validates that critical environment variables (`SECRET_KEY`, `POSTGRES_PASSWORD`) are not placeholders, pulls the latest code with force checkout, builds Docker images (with the `build` profile for frontend assets in production), waits for PostgreSQL readiness, and runs Django's `manage.py check --deploy`. On health check failure, it rolls back to the previous git commit, then restarts the containers. Environment-specific deploy workflows (`dev_deploy.yml`, `staging_deploy.yml`, `production_deploy.yml`) call a shared reusable workflow (`deploy-reusable.yml`) that SSHes into the target server and runs this script.

_scripts/deploy.sh_

```bash
#!/bin/bash
set -euo pipefail

COMPOSE_FILE="${1:-compose.prod.yml}"
BRANCH="${2:-main}"
PROJECT_PATH="${3:-$(pwd)}"

cd "$PROJECT_PATH" || exit 1

# ---- Pre-deployment checks ----

if ! docker info > /dev/null 2>&1; then echo "Docker is not running!"; exit 1; fi
if [ ! -f ".env" ]; then echo ".env file not found!"; exit 1; fi

is_placeholder() {
    local value="$1"
    [[ -z "$value" ]] || [[ "$value" =~ ^\ *\<.*\>\ *$ ]] || [[ "$value" == "secret_key" ]]
}

set -a && source .env && set +a

if is_placeholder "${SECRET_KEY:-}" || is_placeholder "${POSTGRES_PASSWORD:-}"; then
    echo "Critical secrets not configured in .env. Aborting."
    exit 1
fi

# ---- Pull latest code ----

CURRENT_COMMIT=$(git rev-parse HEAD)

git fetch --all --prune
git checkout -B "$BRANCH" "origin/$BRANCH"
git reset --hard "origin/$BRANCH"

NEW_COMMIT=$(git rev-parse HEAD)

# ---- Build and deploy ----

if [ "$COMPOSE_FILE" = "compose.prod.yml" ]; then
    docker compose -f "$COMPOSE_FILE" --profile build build
else
    docker compose -f "$COMPOSE_FILE" build
fi

docker compose -f "$COMPOSE_FILE" up -d

# Wait for PostgreSQL (up to 60 seconds)
for i in {1..30}; do
    if docker compose -f "$COMPOSE_FILE" exec -T postgres \
        bash -c 'pg_isready -U "$POSTGRES_USER" -d "$POSTGRES_DB"' > /dev/null 2>&1; then
        break
    fi
    sleep 2
done

# Wait for Django startup (migrations, collectstatic)
sleep 15

# ---- Health check with rollback ----

# ... color log helpers and detailed error reporting omitted

HEALTH_CHECK_OUTPUT=$(docker compose -f "$COMPOSE_FILE" exec -T django \
    bash -c "cd /opt/project/src && poetry run python manage.py check --deploy" 2>&1) || {
    echo "Health check failed! Rolling back to $CURRENT_COMMIT..."
    git checkout "$CURRENT_COMMIT"
    docker compose -f "$COMPOSE_FILE" up -d
    docker image prune -f
    exit 1
}

docker image prune -f
echo "Deployment completed: $NEW_COMMIT"
```

## Caddy Configuration

Caddy provides automatic HTTPS via Let's Encrypt, eliminating manual certificate management. The configuration routes API and admin requests to Django, serves Vite-built frontend assets and Django static/media files directly, protects API documentation (Swagger/ReDoc) with basic auth, and proxies Celery Flower for task monitoring. JSON access logs feed into fail2ban for automated intrusion detection.

_docker/caddy/Caddyfile_

```caddyfile
{
    # JSON access logs for fail2ban integration
    log {
        output file /var/log/caddy/access.log
        format json
    }
}

{$SITE_DOMAIN} {
    # API docs protected with basic auth
    @docs path /api/schema/* /api/docs/* /api/redoc/*
    handle @docs {
        basic_auth {
            {$CADDY_USER:admin} {$CADDY_PASSWORD}
        }
        reverse_proxy django:8000
    }

    # API and admin routes
    @api path /api/* /admin/*
    handle @api {
        reverse_proxy django:8000
    }

    # Vite-built frontend assets
    handle /static/assets/* {
        uri strip_prefix /static
        root * /vite-dist
        file_server
    }

    # Django static files
    handle /static/* {
        root * /static
        file_server
    }

    # Media files
    handle /media/* {
        root * /media
        file_server
    }

    # Celery Flower monitoring
    reverse_proxy /flower/* flower:5555

    # All other requests to Django
    reverse_proxy /* django:8000
}
```

## Ansible Provisioning

Ansible provisions fresh servers in two plays -- the first installs system packages, hardens security, and installs Docker; the second sets up monitoring (Node Exporter and Promtail for Prometheus/Loki), configures GitHub Actions SSH access, and optionally runs the initial deployment. The split ensures Docker group membership is active before the application setup begins. A backup role sets up weekly cron jobs on production servers.

_ansible/playbooks/provision.yml_

```yaml
---
- name: Provision Django servers - system setup
  hosts: all
  become: yes

  pre_tasks:
    - name: Update and upgrade packages
      apt:
        update_cache: yes
        upgrade: dist

  roles:
    - role: common    # UFW firewall, fail2ban, SSH hardening, app group (GID 1024)
    - role: docker    # Docker + compose plugin installation

- name: Provision Django servers - application setup
  hosts: all
  become: yes

  roles:
    - role: monitoring      # Node Exporter + Promtail (Prometheus/Loki)
      when: enable_monitoring | default(true)
    - role: github_actions  # SSH key, git clone, .env injection, initial deploy
      when: setup_github_actions | default(true)
    - role: backup          # Weekly cron job for DB + media backups
      when: deploy_env == 'production'

  post_tasks:
    - name: Verify Docker is running
      systemd:
        name: docker
        state: started

    # ... reboot check, provisioning summary omitted
```

The `common` role hardens security: disables SSH password authentication, configures UFW (allow 22, 80, 443; deny all else), sets up fail2ban with Caddy-specific filters (HTTP auth, bad bots, DoS), installs unattended-upgrades, and creates the `app` group with GID 1024 to match the container's `nonroot` for volume permissions. The `github_actions` role adds the deployment SSH public key, clones the repository, copies the environment template with injected secrets (`SECRET_KEY`, `POSTGRES_PASSWORD`, domain), and runs the initial `deploy.sh`.
