# Github gist к статье Django+Celery

{% hint style="info" %}
Ссылка на саму статью:

[https://realpython.com/blog/python/asynchronous-tasks-with-django-and-celery/](https://realpython.com/blog/python/asynchronous-tasks-with-django-and-celery/)
{% endhint %}

## Установка Celery

```bash
$ pip install celery
```

## Установка брокера Celery

```bash
$ sudo apt-get install rabbitmq-server
```

## Настройки Django

```python
BROKER_URL = 'amqp://localhost'
CELERY_RESULT_BACKEND = 'rpc://'
CELERY_RESULT_PERSISTENT = False
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'America/Sao_Paulo'
```

Внутри вашего приложения создайте файл с именем `celery.py` и добавьте следующее:

```python
from __future__ import absolute_import

import os

from celery import Celery

# установите модуль настроек Django по умолчанию для программы 'celery'.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'sendmail.settings')

from django.conf import settings  # noqa

app = Celery('sendmail')

# Использование здесь строки означает, что воркеру не придется обрабатывать
# объект при использовании Windows.
app.config_from_object('django.conf:settings')
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)


@app.task(bind=True)
def debug_task(self):
    print('Request: {0!r}'.format(self.request))
```

Теперь добавьте в приложение еще один файл с именем `tasks.py` со следующим содержимым:

```python
import os
from celery.schedules import crontab
from celery.task import periodic_task
from celery.utils.log import get_task_logger

from core.utils import send_mail
import configparser
from django.conf import settings


try:
    config = configparser.ConfigParser()
    file = os.path.join(settings.BASE_DIR, 'schedule.ini')
    config.read(file)
    hour = config['DEFAULT']['hour']
    minute = config['DEFAULT']['minute']
    day = config['DEFAULT']['day']
except Exception as e:
    hour = 8
    minute = 30
    day = 'mon'

logger = get_task_logger(__name__)
@periodic_task(
    run_every=(crontab(hour=hour, minute=minute, day_of_week=day)),
    name="task_send_mail",
    ignore_result=True
)
def task_send_mail():
    """
    Saves latest image from Flickr
    """
    send_mail()
    logger.info("Sent e-mails ;)")
```

## Supervisor

```bash
$ sudo apt-get install supervisor
```

Теперь нам нужно добавить файлы конфигурации супервизора в `/etc/supervisor/conf.d`:

```bash
$ sudo nano /etc/supervisor/conf.d/mail_celery.conf
```

Вставьте в файл следующее содержимое (отредактируйте пути, чтобы они соответствовали вашему серверу):

```ini
; ==================================
;  пример celery worker supervisor
; ==================================

; название вашей программы supervisord
[program:mailcelery]

; Установите полный путь к программе celery, если используете virtualenv
command=/home/maumau/.virtualenvs/sendmail/bin/celery worker -A sendmail --loglevel=INFO

; Каталог вашего проекта Django
directory=/home/maumau/Workspace/projects/sendmail

; Если supervisord запущен от имени пользователя root, перед выполнением
; какой-либо обработки переключите пользователей на эту учетную запись
; пользователя UNIX.
user=maumau

; Supervisor запустит столько экземпляров этой программы, сколько указано numprocs
numprocs=1

; Поместите вывод процесса stdout в этот файл
stdout_logfile=/var/log/celery/mail_worker.log

; Поместите вывод процесса stderr в этот файл
stderr_logfile=/var/log/celery/mail_worker.log

; Если true, эта программа запустится автоматически при запуске supervisord.
autostart=true

; Может быть одним из false, unexpected или true.
; Если false, процесс никогда не будет автоматически перезапущен.
; В случае unexpected процесс будет перезапущен при завершении программы
; с кодом выхода, который не является одним из кодов выхода,
; связанных с конфигурацией этого процесса (см. коды выхода).
; Если true, процесс будет безоговорочно перезапущен при выходе,
; независимо от его кода выхода.
autorestart=true

; Общее количество секунд, которое программа должна продолжать работать после запуска,
; чтобы считать запуск успешным.
startsecs=10

; Необходимо дождаться завершения текущих выполняемых задач при завершении работы.
; Увеличьте это значение, если у вас очень длительные задачи.
stopwaitsecs = 600

; Когда вы прибегаете к отправке SIGKILL программе для завершения,
; она вместо этого отправляет SIGKILL всей своей группе процессов,
; заботясь также и о своих потомках.
killasgroup=true

; если ваш брокер находится под наблюдением, установите для него
; более высокий приоритет, чтобы он стартовал первым
priority=998
```

А потом...

```bash
$ sudo nano /etc/supervisor/conf.d/mail_celerybeat.conf
```

Со следующим содержанием:

```ini
; ================================
;  пример celery beat supervisor
; ================================

; название вашей программы supervisord
[program:mailcelerybeat]

; Установите полный путь к программе celery, если используете virtualenv
command=/home/maumau/.virtualenvs/sendmail/bin/celerybeat -A sendmail --loglevel=INFO

;  Каталог вашего проекта Django
directory=/home/maumau/Workspace/projects/sendmail

; Если supervisord запущен от имени пользователя root, перед выполнением
; какой-либо обработки переключите пользователей на эту учетную запись
; пользователя UNIX.
user=maumau

; Supervisor запустит столько экземпляров этой программы, сколько указано numprocs
numprocs=1

; Поместите вывод процесса stdout в этот файл
stdout_logfile=/var/log/celery/mail_beat.log

; Поместите вывод процесса stderr в этот файл
stderr_logfile=/var/log/celery/mail_beat.log

; Если true, эта программа запустится автоматически при запуске supervisord.
autostart=true

; Может быть одним из false, unexpected или true.
; Если false, процесс никогда не будет автоматически перезапущен.
; В случае unexpected процесс будет перезапущен при завершении программы
; с кодом выхода, который не является одним из кодов выхода,
; связанных с конфигурацией этого процесса (см. коды выхода).
; Если true, процесс будет безоговорочно перезапущен при выходе,
; независимо от его кода выхода.
autorestart=true

; Общее количество секунд, которое программа должна продолжать работать после запуска,
; чтобы считать запуск успешным.
startsecs=10

; если ваш брокер находится под наблюдением, установите для него
; более высокий приоритет, чтобы он стартовал первым
priority=999
```

Теперь, чтобы запустить супервизора:

```bash
$ sudo supervisord
```

Он загрузит файлы `.conf`, которые мы только что создали.

Для запуска/остановки и состояния используйте следующие команды:

```bash
$ sudo supervisorctl stop mailcelery
$ sudo supervisorctl start mailcelery
$ sudo supervisorctl status mailcelery
```

## Обсуждение

Я считаю, что между **celery** и **beat** в аргументе команды для конфигурации `celery beat` должен быть пробел, нет?

Сохранение файла в `/etc/supervisor/conf.d` у меня не работает: SupervisorD не ищет там файлы. К сожалению :(

Посмотрите файл `/etc/supervisord.conf` в разделе `[include]` и замените строку `"files = supervisord.d/*.ini"` на `"files = supervisord.d/*.conf"`.

Спасибо исправил мою проблему
