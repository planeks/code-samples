```docker-compose
networks:
  no-internet:
    external: true
    name: no-internet
  internet:
    external: true
    name: internet
  private-db:
    external: true
    name: private-db

services:
  web:
    build:
      context: ./
      dockerfile: Dockerfile.web.dev
    container_name: web
    command: >
      sh -c "python manage.py wait_for_db &&
             python manage.py migrate &&
             python manage.py runserver 0.0.0.0:8000"
    healthcheck:
      test: "curl -Is http://localhost:8000/metrics || exit 1"
      interval: 30s
      timeout: 5s
      retries: 5
    volumes:
      - ./:/app/
    ports:
      - 8000:8000
    env_file:
      - environments/.env.container
    networks:
      - internet
      - no-internet
      - private-db
  micro_serice1:
    build:
      context: ./
      dockerfile: micro_serice1/Dockerfile.dev
    volumes:
      - ./:/app/
    networks:
      - no-internet
      - internet
      - private-db
    restart: always
    env_file:
      - micro_serice1/environments/.env.container
    deploy:
      mode: replicated
      replicas: 2
  micro_serice2:
    build:
      context: ./
      dockerfile: micro_serice2/Dockerfile.dev
    volumes:
      - ./:/app/
    networks:
      - no-internet
      - internet
      - private-db
    restart: always
    env_file:
      - micro_serice2/environments/.env.container
    deploy:
      mode: replicated
      replicas: 2
  micro_serice3:
    build:
      context: ./
      dockerfile: micro_serice3/Dockerfile.dev
    container_name: micro_serice3
    volumes:
      - ./:/app/
    expose:
      - 8020:8020
    networks:
      - no-internet
      - internet
    restart: always
    env_file:
      - micro_serice3/environments/.env.container

  celery-beat:
    build:
      context: ./
      dockerfile: Dockerfile.celery
    container_name: quantcloud_backend_celery-beat
    command: >
      sh -c "celery -A qc_client beat -l info"
    volumes:
      - ./:/app/
    env_file:
      - environments/.env.container.celery
    networks:
      - internet
      - no-internet
  celery:
    build:
      context: ./
      dockerfile: Dockerfile.celery
    container_name: quantcloud_backend_celery
    command: >
      sh -c "celery -A qc_client worker -l info"
    volumes:
      - ./:/app/
    env_file:
      - environments/.env.container.celery
    networks:
      - internet
      - no-internet

