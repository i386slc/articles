# Как вызвать задачу Celery из другого приложения

{% hint style="info" %}
**Оригинальное название**: How to call a Celery task from another app

**Ссылка**: [https://www.distributedpython.com/2018/06/19/call-celery-task-outside-codebase/](https://www.distributedpython.com/2018/06/19/call-celery-task-outside-codebase/)

**Автор**: Bjorn Stiel

**Дата**: 19 июня 2018
{% endhint %}

Стандартные документы и руководства по Celery обычно предполагают, что исходный код вашего Celery и вашего API находится в одной и той же кодовой базе. В этом сценарии вы просто импортируете задачу в свой модуль API и асинхронно вызываете задачу Celery.

```python
from flask import Flask, jsonify
from tasks import fetch_data

app = Flask(__name__)

@app.route('/', methods=['POST'])
def index():
    fetch_data.s(url=request.json['url']).delay()
    return jsonify({'url': request.json['url']}), 201

```

Это работает только до тех пор, пока задача зарегистрирована в вашем текущем процессе. Вам нужна альтернативная стратегия, когда ваш исходный код Celery и ваш API не являются частью одной и той же кодовой базы. Или когда у вас есть несколько микросервисов Celery и вам нужно вызвать задачу Celery из другого микросервиса Celery.

## Вариант 1: app.send\_task

Экземпляр приложения Celery имеет метод **send\_task**, который можно использовать для вызова задачи по ее имени.

```python
from flask import Flask, jsonify
from worker import app as celery_app

app = Flask(__name__)

@app.route('/', methods=['POST'])
def index():
    celery_app.send_task('fetch_data', kwargs={'url': request.json['url']})
    return jsonify({'url': request.json['url']}), 201

```

## Вариант 2: app.signature

В нашем первоначальном примере метод `.s(...)` задачи Celery создает вызываемый объект Celery **Signature**. Подпись Celery, по сути, оборачивает аргументы, аргументы ключевых слов и параметры выполнения одного вызова задачи Celery, чтобы их можно было передать функциям или сериализовать и отправить по сети.

Поэтому альтернативная стратегия заключается в том, чтобы явно создать объект **Signature** с помощью метода **signature** приложения Celery, передав имя задачи и ее **kwargs**. Полученный объект **Signature** затем может быть асинхронно выполнен методом **delay()** так же, как мы делали это в стандартном примере.

```python
from flask import Flask, jsonify
from worker import app as celery_app

app = Flask(__name__)

@app.route('/', methods=['POST'])
def index():
    celery_app.signature('fetch_data', kwargs={'url': request.json['url']).delay()
    return jsonify({'url': request.json['url']}), 201

```

## Резюме

Celery не требует доступа к базе кода задачи для ее вызова. Хитрость заключается в том, чтобы вызвать задачу по ее имени либо напрямую через `celery_app.send_task(...)`, либо путем создания объекта **Signature** `celery_app.signature(...)`, который эквивалентен вызову `task.s(...)` когда у вас есть доступ к базе кода задачи.

Какой подход вы предпочитаете, в конечном счете, вопрос вкуса, есть незначительные последствия того, как вы тестируете свои вызовы задач, которые мы рассмотрим в другом сообщении в блоге.
