```docker-compose
version: '3.8'

x-django: &django
  build:
    context: .
    dockerfile: ./docker/django/Dockerfile
  image:  "camberauth_dev"
  depends_on:
    - postgres
  volumes:
    - ./src:/opt/project/src:cached
    - ./data/dev:/data:z
  env_file:
    - ./.env

services:
  django:
    <<: *django
    ports:
      - "8000:8000"
    command: dev
  postgres:
    build:
      context: .
      dockerfile: ./docker/postgres/Dockerfile
    image: "camberauth_dev_postgres"
    volumes:
      - dev_postgres:/var/lib/postgresql/data:Z
      - dev_backups:/backups:z
    env_file:
      - ./.env
    ports:
      - "5432:5432"
  mailhog:
    image: mailhog/mailhog
    logging:
      driver: 'none'  # disable saving logs
    ports:
      - "8025:8025" # web ui
  jupyterhub:
    build:
      context: .
      dockerfile: ./docker/jupyterhub/Dockerfile
      args:
        JUPYTERHUB_VERSION: 3.0.0
    image: "camberauth_dev_jupyterhub"
    volumes:
      - "./hub/jupyterhub_config.py:/srv/jupyterhub/jupyterhub_config.py:cached"
      - "./hub/templates:/srv/jupyterhub/templates:cached"
      - "/var/run/docker.sock:/var/run/docker.sock:rw"
      - "dev_jupyterhub:/data"
    depends_on:
      - django
    networks:
      - default
      - jupyterhub-network
    env_file:
      - ./.env
    extra_hosts:
      - "${SITE_DOMAIN}:172.17.0.1"
  caddy:
    build:
      context: .
      dockerfile: ./docker/caddy/Dockerfile
    image: "camberauth_dev_caddy"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - dev_caddy:/data
    env_file:
      - ./.env
    depends_on:
      - django

volumes:
  dev_postgres:
  dev_backups:
  dev_caddy:
  dev_jupyterhub:

networks:
  jupyterhub-network:
    name: jupyterhub-network
