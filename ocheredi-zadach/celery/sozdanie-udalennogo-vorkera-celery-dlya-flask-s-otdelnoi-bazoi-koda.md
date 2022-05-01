# Создание удаленного воркера Celery для Flask с отдельной базой кода

{% hint style="info" %}
**Оригинальное название**: Creating remote Celery worker for Flask with separate code base

**Ссылка**: [https://moinulhossain.me/remote-celery-worker-for-flask-with-separate-code-base/](https://moinulhossain.me/remote-celery-worker-for-flask-with-separate-code-base/)

**Автор**: [**Moinul Hossain**](https://moinulhossain.me/author/moinul/)

**Дата**: 1 марта 2016
{% endhint %}

Этот [flask сниппет](http://flask.pocoo.org/docs/latest/patterns/celery/) показывает, как интегрировать Celery в flask, чтобы иметь доступ к контексту приложения flask. Что, если мы не хотим, чтобы задачи celery были в кодовой базе приложений Flask? Мы можем вызывать задачи Celery, не имея доступа к функциям задачи в Flask, используя имя задачи и отправляя задачу с помощью метода `celery.send_task`.

Вот пример приложения Flask, использующего задачу Celery, которая недоступна в кодовой базе Flask:

```bash
- flask-app
    - app.py
```

Вот как выглядит пример `app.py`

```python
import os

from flask import Flask  
from flask import url_for

from celery import Celery  
from celery.result import AsyncResult  
import celery.states as states


env=os.environ  
CELERY_BROKER_URL=env.get('CELERY_BROKER_URL','redis://localhost:6379'),  
CELERY_RESULT_BACKEND=env.get('CELERY_RESULT_BACKEND','redis://localhost:6379')

celery= Celery('tasks',  
                broker=CELERY_BROKER_URL,
                backend=CELERY_RESULT_BACKEND)

env=os.environ  
app = Flask(__name__)

# Отправьте два числа для сложения
@app.route('/add/<int:param1>/<int:param2>')
def add(param1,param2):  
    task = celery.send_task('mytasks.add', args=[param1, param2])
    return task.id

# Проверьте статус задачи с идентификатором, найденным в функции добавления
@app.route('/check/<string:id>')
def check_task(id):  
    res = celery.AsyncResult(id)
    return res.state if res.state==states.PENDING else str(res.result)

if __name__ == '__main__':  
    app.run(debug=env.get('DEBUG',True),
            port=int(env.get('PORT',5000)),
            host=env.get('HOST','0.0.0.0'))
```

Теперь задачи Celery могут выполняться на отдельной машине (или нескольких машинах) с собственной кодовой базой.

```bash
- flask-celery
    - tasks.py
```

```python
import os  
import time  
from celery import Celery

env=os.environ  
CELERY_BROKER_URL=env.get('CELERY_BROKER_URL','redis://localhost:6379'),  
CELERY_RESULT_BACKEND=env.get('CELERY_RESULT_BACKEND','redis://localhost:6379')


celery= Celery('tasks',  
                broker=CELERY_BROKER_URL,
                backend=CELERY_RESULT_BACKEND)

# Параметр name является ключевым здесь
@celery.task(name='mytasks.add')
def add(x, y):  
    # давайте немного поспим, прежде чем выполнять гигантскую задачу по сложению!
    time.sleep(5)
    return x + y
```

И запустить воркер как всегда:

```bash
celery -A tasks worker --loglevel=info
```

Теперь мы можем запустить столько воркеров, сколько захотим, клонировав модуль **flask-celery** на столько серверов, сколько захотим.

### Пример докеризации для масштабирования воркеров

Докеризованный пример доступен [здесь](https://github.com/itsrifat/flask-celery-docker-scale).

Чтобы запустить пример докера:

```bash
docker-compose build  
docker-compose up -d # запустить в автономном режиме  
```

Теперь загрузите `http://your-dockermachine-ip:5000/add/2/3` в браузере. Он должен создать задачу и вернуть идентификатор задачи.

Чтобы проверить статус задания, нажмите `http://your-dockermachine-ip:5000/check/taskid`. Он должен либо показывать **PENDING**, либо результат **5**.

Чтобы следить за тем, чтобы воркер с [flower](http://flower.readthedocs.org) зашел на `http://your-dockermachine-ip:5555`. Он должен показать один воркер, готовый работать.

Чтобы масштабировать воркеры, запустите `docker-compose scale worker=5`. Это создаст еще 4 контейнера, в каждом из которых будет работать воркер. `http://your-dockermachine-ip:5555` теперь должно показывать 5 воркеров, ожидающих некоторых заданий!
