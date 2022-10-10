# Celery configuration

## Multiple queues

Tasks routing:


```python
    CELERY_TASK_ROUTES = {
        'project.tasks.clinical_content.*': {'queue': 'general_queue'},
        'project.tasks.general.*': {'queue': 'general_queue'},
        'project.tasks.messaging.*': {'queue': 'general_queue'},
        'project.tasks.scheduling.*': {'queue': 'general_queue'},
        'project.tasks.synchronization.*': {'queue': 'general_queue'},
        'project.tasks.adobe_extract_api.*': {'queue': 'general_queue'},
        'project.tasks.lab_profiles.*': {'queue': 'general_queue'},
        'project.tasks.autorelease.*': {'queue': 'general_queue'},
        'project.tasks.healthjump.hj_message_processing': {'queue': 'general_queue'},
        'project.tasks.healthjump.import_health_jump_messages': {'queue': 'general_queue'},
        'project.tasks.healthjump.import_diagnoses_from_hj': {'queue': 'general_queue'},
        'project.tasks.healthjump.hj_incomplete_message_processing': {'queue': 'hj_queue'},
        'project.tasks.healthjump.process_new_hj_message': {'queue': 'hj_queue'},
        'project.tasks.healthjump.import_historical_data': {'queue': 'hj_queue'},
        'project.tasks.healthjump.import_historical_lab_orders_from_csv': {'queue': 'historical_queue'},
    }
```

`Procfile` for Heroku with multiple Celery workers:

```
release: python manage.py migrate --noinput
web: gunicorn project.wsgi --log-file - --preload
celery_1: celery -A project worker -Q general_queue -l INFO -c 4
celery_2: celery -A project worker -Q hj_queue -l INFO -c 4
celery_3: celery -A project worker -Q historical_queue -l INFO -c 2
beat: celery -A project beat -S redbeat.RedBeatScheduler -l INFO
```

