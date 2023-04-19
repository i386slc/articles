# Интеграция APScheduler и django\_apscheduler в проект Django

{% hint style="info" %}
Ссылка на оригинальную статью: [Integrating APScheduler and django\_apscheduler into a real life Django project](https://medium.com/@mrgrantanderson/replacing-cron-and-running-background-tasks-in-django-using-apscheduler-and-django-apscheduler-d562646c062e)

Опубликовано: 19 декабря 2018

Автор: [Grant Anderson](https://medium.com/@mrgrantanderson?source=---two\_column\_layout\_sidebar----------------------------------)
{% endhint %}

<figure><img src="../../.gitbook/assets/django.webp" alt=""><figcaption></figcaption></figure>

[Расширенный планировщик Python (APScheduler)](https://apscheduler.readthedocs.io/) — это мощная и универсальная библиотека, которую я использовал в прошлом в качестве замены **cron**, а также для запуска фонового выполнения кода.

Реализация **APScheduler** в простом скрипте Python, как показано в его документации, проста. Заставить его работать в производственном проекте Django с автоматическими тестами, атомарными транзакциями базы данных и т. д. Однако я обнаружил, что это немного сложнее.

Интеграция **APScheduler** в проект Django немного упрощается с помощью [django\_apscheduler](https://github.com/jarekwg/django-apscheduler), который обеспечивает поддержку постоянного хранения заданий в базе данных через Django ORM и управление заданиями через административную консоль Django.

Недавно я использовал **APScheduler** с **django\_apscheduler** в проекте, чтобы заменить задания **cron** и разрешить выполнение длительных операций в фоновом режиме.

Этот подход предназначен в качестве альтернативы использованию чего-то вроде **Celery**, который требует немного большей настройки и должен запускаться независимо от кода проекта.

Другой альтернативой этому подходу и Celery может быть использование [каналов Django](https://channels.readthedocs.io/en/latest/), но это то, что мне еще предстоит попробовать.

В этой статье будут рассмотрены следующие проблемы, с которыми я столкнулся при интеграции **APScheduler** с полным проектом:

* Запуск планировщика только после того, как Django будет готов
* Отключение планировщика при запуске pytest
* Добавление заданий во время атомарной транзакции базы данных

{% hint style="info" %}
Обратите внимание, что этот подход еще не был полностью протестирован и развернут в рабочей среде и может страдать от проблем, включая безопасность потоков, в зависимости от исполнителя, который вы решите использовать.
{% endhint %}

В этих примерах предполагается, что у вас уже есть настройка проекта Django.

## Настройка

Используйте pip для установки этих пакетов:

* apscheduler
* django-apscheduler

Добавьте **django\_apscheduler** в настройки вашего проекта Django, а также в конфигурацию по умолчанию, обратите внимание, что **DjangoJobStore** не является зарегистрированным плагином, поэтому мы должны добавить класс:

```python
# settings.py

INSTALLED_APPS = [
	...
	"django_apscheduler",
]

# Эта конфигурация планировщика будет:
# - Хранить задания в базе данных проекта
# - Выполнять задания в потоках внутри процесса приложения
SCHEDULER_CONFIG = {
    "apscheduler.jobstores.default": {
        "class": "django_apscheduler.jobstores:DjangoJobStore"
    },
    'apscheduler.executors.processpool': {
        "type": "threadpool"
    },
}
SCHEDULER_AUTOSTART = True
```

Теперь мы можем настроить наш планировщик:

```python
# core/scheduler.py

import logging

from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.executors.pool import ProcessPoolExecutor, ThreadPoolExecutor
from django_apscheduler.jobstores import register_events, register_job

from django.conf import settings

# Создайте планировщик для запуска в потоке внутри процесса приложения.
scheduler = BackgroundScheduler(settings.SCHEDULER_CONFIG)

def start():
    if settings.DEBUG:
      	# Хук для подключения к регистратору apscheduler
        logging.basicConfig()
        logging.getLogger('apscheduler').setLevel(logging.DEBUG)

    # Добавляем эту работу сюда, а не в cron.
    # Это сделает следующее:
    # - Добавляет запланированное задание в хранилище заданий при инициализации приложения
    # - Задание будет выполнять метод класса модели в полночь каждый день.
    # - replace_existing в сочетании с уникальным идентификатором предотвращает дублирование заданий.
    scheduler.add_job(
        "core.models.MyModel.my_class_method",
        "cron", id="my_class_method", hour=0,
        replace_existing=True
    )

    # Добавляет запланированные задания в интерфейс администрирования Django.
    register_events(scheduler)

    scheduler.start()
```

### Запуск планировщика только после того, как Django будет готов

Примеры в документах **APScheduler** и **django\_apscheduler** показывают, что планировщик запускается, как только в него добавляются задания.

Это вызовет ошибки, так как Django не будет должным образом настроен и готов к этому моменту.

Вот почему мы использовали функцию-оболочку выше, а затем вызвали ее, как только основное приложение будет готово:

```python
# core/apps.py

from django.conf import settings

class CoreConfig(AppConfig):
    name = "core"

     def ready(self):
        from . import scheduler
        if settings.SCHEDULER_AUTOSTART:
        	scheduler.start()
```

### Отключение планировщика при запуске pytest

**Pytest** и **APScheduler**, похоже, вообще не очень хорошо работают вместе.

Запуск pytest вернет такие ошибки:

```
Exception Value: Database access not allowed, use the “django_db” mark,
or the “db” or “transactional_db” fixtures to enable it.
```

Даже с изменением хранилища заданий с `django_apscheduler.DjangoJobStore` на `apscheduler.MemoryJobStore` и без добавления заданий мне не удалось запустить pytest без ошибок подключения к базе данных, таких как это:

```
django.db.utils.OperationalError: database "my_database” is being
accessed by other users
DETAIL: There is 1 other session using the database.
```

Я считаю, что это связано с тем, что **APScheduler** использует многопоточность, и пока не нашел решения.

Обходной путь — запретить запуск планировщика, когда проект запускается **Pytest**.

Мы можем создать отдельный файл настроек для использования pytest и изменить значение параметра **SCHEDULER\_AUTOSTART**:

```python
# pytest_settings.py

SCHEDULER_AUTOSTART = False
```

Теперь нам просто нужно сказать pytest использовать новые настройки, это [можно сделать любым из следующих способов (в порядке приоритета)](https://pytest-django.readthedocs.io/en/latest/configuring\_django.html):

Командная строка:

```bash
$ pytest --ds=pytest_settings
```

Переменная среды:

```bash
$ DJANGO_SETTINGS_MODULE=pytest_settings pytest
```

Файл конфигурации `pytest.ini`:

```ini
[pytest]
DJANGO_SETTINGS_MODULE = pytest_settings
```

### Добавление заданий во время атомарной транзакции базы данных

Добавление задания в планировщик приведет к его проверке на наличие каких-либо заданий, которые нужно запустить, поскольку добавленное задание может быть предназначено для немедленного запуска.

Когда мы храним задания в БД с помощью **DjangoJobStore**, это означает, что задания должны находиться в БД, чтобы планировщик мог их найти. Если код для добавления задания выполняется в атомарной транзакции, планировщик, скорее всего, проверит базу данных на наличие заданий до того, как транзакция будет зафиксирована в базе данных.

К счастью, решение очень простое, используя транзакции Django `transaction.on_commit`, чтобы добавить задание только после того, как данные будут сохранены в базе данных.

Моя проблема в реальной жизни возникла из-за того, что импортер данных использует `transaction.atomic` при сохранении в базу данных, а фоновая задача вызывается из сигнала **post\_save** в моей модели. Этот пример показался немного тяжелым для этой демонстрации, поэтому ниже я использовал более простой пример.

Рассмотрим этот (надуманный) пример:

```python
# atomic_function.py

from django.db import transaction

from core.models import MyDataModel
from core.scheduler import scheduler


@transaction.atomic
def create_data_set(set_id, data_values_list):
    """
    Атомарная функция для создания уникального набора данных в базе данных.
    
    Либо все элементы в наборе данных будут сохранены в БД, либо ничего не будет.
    """
    if MyDataModel.objects.filter(set_id=set_id).exists():
        raise ValueError(f"Data set with set_id {set_id} already exists.")
    
    for data_value in data_values_list:
        MyDataModel.objects.create(set_id=st_id, data=data_value)
    
    def add_processing_job():
    	scheduler.add_job(
            MyDataModel.my_data_processing_method,
            id="my_data_processing_job",
            replace_existing=True
        )
    transaction.on_commit(add_processing_job)
    
    return MyDataModel.objects.filter(set_id=set_id)
```
