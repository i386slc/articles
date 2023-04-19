# Планирование задач с помощью APScheduler

{% hint style="info" %}
Ссылка на оригинальную статью: [Scheduling tasks using APScheduler in Django](https://dev.to/brightside/scheduling-tasks-using-apscheduler-in-django-2dbl)

Опубликовано: 29 января 2021

Автор: ["edisthgirb"\[::-1\]](https://dev.to/brightside)
{% endhint %}

В этом руководстве показано, как планировать задачи с помощью **APScheduler** в Django, а не с реальными основами Python или Django.

Хорошо, давайте начнем

## Установка APScheduler

Выполните следующую команду в терминале:

```bash
pip install apscheduler
```

## Настройка APScheduler

Давайте рассмотрим приложение с именем room.

### Добавляем something\_update.py в каталог нашего приложения:

Вот как должен выглядеть ваш `room/something_update.py`:

```python
def update_something():
    print("this function runs every 10 seconds")
```

### Добавляем updater.py в каталог нашего приложения:

Вот как должна выглядеть ваша `room/updater.py`:

```python
from apscheduler.schedulers.background import BackgroundScheduler
from .something_update import update_something

def start():
    scheduler = BackgroundScheduler()
    scheduler.add_job(update_something, 'interval', seconds=10)
    scheduler.start()
```

### Запускаем программу Updater:

Вот как должен выглядеть ваш `room/apps.py`:

```python
from django.apps import AppConfig

class RoomConfig(AppConfig):
    name = 'room'

    def ready(self):
        from . import updater
        updater.start()
```

Спасибо, это все для этого урока.
