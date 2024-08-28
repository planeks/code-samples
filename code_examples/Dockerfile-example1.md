```dockerfile
FROM python:3.9-buster

ENV PYTHONUNBUFFERED 1

RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add -
RUN curl https://packages.microsoft.com/config/debian/10/prod.list > /etc/apt/sources.list.d/mssql-release.list

RUN apt-get update \
  # dependencies for building Python packages
  && apt-get install -y build-essential libz-dev libjpeg-dev libfreetype6-dev python-dev \
  # psycopg2 dependencies
  && apt-get install -y libpq-dev \
  # Translations dependencies
  && apt-get install -y gettext \
  # Some extra packages
  && apt-get install -y curl mc nano graphviz \
  && ACCEPT_EULA=Y apt-get install -y msodbcsql17 \
  && ACCEPT_EULA=Y apt-get install -y mssql-tools \
  && apt-get install -y unixodbc-dev \
  # cleaning up unused files
  && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
  && rm -rf /var/lib/apt/lists/*

RUN addgroup --gid 1024 appuser \
    && adduser --disabled-password --gecos "" --force-badname --ingroup appuser appuser

COPY ./compose/local/openssl/openssl.cnf /etc/ssl/openssl.cnf

# Requirements are installed here to ensure they will be cached.
COPY ./src/requirements.txt /requirements.txt
RUN pip install -U pip==24.0 pip-tools && pip install --no-cache-dir -r /requirements.txt && rm -rf /requirements.txt

COPY --chown=appuser:appuser ./compose/local/django/entrypoint /entrypoint
RUN sed -i 's/\r$//g' /entrypoint
RUN chmod +x /entrypoint

COPY --chown=appuser:appuser ./compose/local/django/start /start
RUN sed -i 's/\r$//g' /start
RUN chmod +x /start

COPY --chown=appuser:appuser ./compose/local/django/celery/worker/start /start-celeryworker
RUN sed -i 's/\r$//g' /start-celeryworker
RUN chmod +x /start-celeryworker

COPY --chown=appuser:appuser ./compose/local/django/celery/beat/start /start-celerybeat
RUN sed -i 's/\r$//g' /start-celerybeat
RUN chmod +x /start-celerybeat

COPY --chown=appuser:appuser ./src /app

RUN mkdir /data
RUN chown -R appuser:appuser /data
RUN chmod 775 /data
RUN chmod g+s /data

USER appuser

WORKDIR /app

ENTRYPOINT ["/entrypoint"]

