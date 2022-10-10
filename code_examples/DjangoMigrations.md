# Django migrations

The custom migration that do some normalization of available data in the database.

```python
from django.db import migrations, IntegrityError, transaction
import users.models


def replace_email_to_lowercase(apps, schema_editor):
    CustomUser = apps.get_model("users", "CustomUser")
    UnregisteredUser = apps.get_model("users", "UnregisteredUser")

    for custom_user in CustomUser.objects.all():
        try:
            with transaction.atomic():
                custom_user.email = custom_user.email.lower()
                custom_user.save()
        except IntegrityError:
            custom_user.delete()

    for unregistered_user in UnregisteredUser.objects.all():
        try:
            with transaction.atomic():
                unregistered_user.email = unregistered_user.email.lower()
                unregistered_user.save()
        except IntegrityError:
            unregistered_user.delete()


class Migration(migrations.Migration):

    dependencies = [
        ('users', '0002_customuser_unread_notifications_count'),
    ]

    operations = [
        migrations.AlterField(
            model_name='customuser',
            name='email',
            field=users.models.LowerEmailField(max_length=254, unique=True, verbose_name='email address'),
        ),
        migrations.AlterField(
            model_name='unregistereduser',
            name='email',
            field=users.models.LowerEmailField(max_length=254, unique=True),
        ),
        migrations.RunPython(replace_email_to_lowercase)
    ]
```

