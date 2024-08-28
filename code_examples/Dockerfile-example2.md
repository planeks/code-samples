```dockerfile
FROM ubuntu:20.04

ENV PATH /usr/local/bin:$PATH
ENV PYTHONUNBUFFERED 1
ENV LANG C.UTF-8
ENV DEBIAN_FRONTEND noninteractive

RUN apt-get update && apt-get -y install software-properties-common build-essential \
    apt-utils unzip libaio-dev libcairo2-dev locales libgs-dev ssh imagemagick \
    && echo ttf-mscorefonts-installer msttcorefonts/accepted-mscorefonts-eula select true | debconf-set-selections \
    && apt-get install -y --no-install-recommends apt-utils ttf-mscorefonts-installer fontconfig \
    && apt-get install -y icc-profiles-free liblept5 libxml2 pngquant \
    && apt-get install -y ffmpeg libsm6 libxext6 \
    && apt-get install -y tesseract-ocr tesseract-ocr-all zlib1g poppler-utils unpaper \
    && add-apt-repository -y ppa:deadsnakes/ppa && apt-get update \
    && apt-get -y install python3.9 python3.9-dev python3.9-distutils python3-pip \
    && apt-get -y install curl mc nano \
    && apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false \
    && rm -rf /var/lib/apt/lists/*

COPY ./compose/ghostscript-10.01.0.tar.gz /ghostscript-10.01.0.tar.gz
RUN tar -xvf /ghostscript-10.01.0.tar.gz && cd /ghostscript-10.01.0 && ./configure && make && make install && cd / && rm -rf /ghostscript-10.01.0.tar.gz /ghostscript-10.01.0

RUN ln -s /usr/bin/python3.9 /usr/local/bin/python \
    && ln -s /usr/bin/pip3 /usr/local/bin/pip

RUN addgroup --gid 1014 appuser \
    && adduser --disabled-password --gecos "" --force-badname --ingroup appuser appuser

COPY ./src/requirements.txt /requirements.txt

RUN python3.9 -m pip install -U pip pip-tools && python3.9 -m pip install --no-cache-dir -r /requirements.txt && rm -rf /requirements.txt

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

COPY --chown=appuser:appuser ./compose/local/django/celery/flower/start /start-flower
RUN sed -i 's/\r$//g' /start-flower
RUN chmod +x /start-flower

COPY --chown=appuser:appuser ./src /app
COPY --chown=appuser:appuser ./README.md /README.md

RUN mkdir /data
RUN chown -R appuser:appuser /data
RUN chmod 775 /data
RUN chmod g+s /data
RUN mkdir /staticfiles
RUN chown -R appuser:appuser /staticfiles
RUN chmod 775 /staticfiles
RUN chmod g+s /staticfiles
RUN mkdir /extras
RUN chown -R appuser:appuser /extras
RUN chmod 775 /extras
RUN chmod g+s /extras
RUN mkdir /logs
RUN chown -R appuser:appuser /logs
RUN chmod 775 /logs
RUN chmod g+s /logs

WORKDIR /app
USER appuser
ENTRYPOINT ["/entrypoint"]
