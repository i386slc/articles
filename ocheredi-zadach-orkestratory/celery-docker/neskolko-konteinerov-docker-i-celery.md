# Несколько контейнеров Docker и Celery

{% hint style="info" %}
**Оригинальное название**: Multiple Docker containers and Celery

**Ссылка**: [https://stackoverflow.com/questions/45863053/multiple-docker-containers-and-celery](https://stackoverflow.com/questions/45863053/multiple-docker-containers-and-celery)

**Автор**: [Tzoiker](https://stackoverflow.com/users/1555653/tzoiker)

**Дата**: 24 августа 2017
{% endhint %}

## Вопрос

Сейчас у нас следующая структура проекта:

1. Веб-сервер, обрабатывающий входящие запросы от клиентов.
2. Модуль аналитики, который предоставляет некоторые рекомендации пользователям.

Мы решили оставить эти модули полностью независимыми и переместить их в разные докер-контейнеры. Когда запрос от пользователя поступает на веб-сервер, он отправляет еще один запрос в модуль аналитики для получения рекомендаций.

Чтобы рекомендации были последовательными, нам необходимо периодически выполнять некоторые фоновые вычисления, например, когда в нашей системе регистрируются новые пользователи. Также некоторые фоновые задачи связаны исключительно с логикой веб-сервера. Для этого мы решили использовать распределенную очередь задач, например, **Celery**.

Возможны следующие сценарии создания и выполнения задачи:

1. Задача поставлена в очередь на веб-сервере, выполняется на веб-сервере (например, обработка загруженного изображения)
2. Задача поставлена в очередь на веб-сервере, выполняется в модуле аналитики (например, вычислить рекомендации для нового пользователя)
3. Задача ставится в очередь в модуле аналитики и выполняется там (например, периодическое обновление)

Пока я вижу 3 довольно странные возможности использования **Celery** здесь:

### I. Celery в отдельном контейнере и делает все

1. Переместить **Celery** в отдельный док-контейнер.
2. Предоставить все необходимые пакеты как веб-сервера, так и аналитики для выполнения задач.
3. Делиться кодом задач с другими контейнерами (или объявлять фиктивные задачи на веб-сервере и в аналитике)

### II. Celery в отдельном контейнере и гораздо меньше

То же, что и [I](neskolko-konteinerov-docker-i-celery.md#i.-celery-v-otdelnom-konteinere-i-delaet-vse), но теперь задачи — это просто запросы к веб-серверу и модулю аналитики, которые там обрабатываются асинхронно, а результат опрашивается внутри задачи до тех пор, пока он не будет готов.

Таким образом, мы получаем преимущества от наличия брокера, но все тяжелые вычисления переносятся с воркеров **Celery**.

## III. Отдельный Celery в каждом контейнере

1. Запустить **Celery** как в веб-сервере, так и в модуле аналитики.
2. Добавить объявления фиктивных задач (задач аналитики) на веб-сервер.
3. Добавить 2 очереди задач, одну для веб-сервера, одну для аналитики.

Таким образом, задачи, запланированные на веб-сервере, могут выполняться в модуле аналитики. Тем не менее, по-прежнему приходится разделять код задач между контейнерами или использовать фиктивные задачи, а также запускать `celery workers` в каждом контейнере.

Как лучше это сделать, или логику нужно полностью менять, например, переместить все в один контейнер?

## Ответ

Во-первых, давайте проясним разницу между библиотекой **celery** (которую вы получаете при `pip install` или в вашем `setup.py`) и **celery worker** — это фактический процесс, который удаляет задачи из очереди у брокера и обрабатывает их. Конечно, вы можете захотеть иметь несколько **workers/processes** (например, для разделения разных задач на другого **worker**).

Допустим, у вас есть две задачи: **calculate\_recommendations\_task** и **periodic\_update\_task**, и вы хотите запустить их на отдельном **worker**, например, **recommendation\_worker** и **periodic\_worker**. Другим процессом будет `celery beat`, который просто ставит в очередь **periodic\_update\_task** брокеру каждые x часов.

Кроме того, допустим, у вас есть простой веб-сервер, реализованный с помощью [bottle](https://bottlepy.org/docs/dev/).

Я предполагаю, что вы хотите использовать брокера Celery и backend с докером, и я выберу рекомендуемое использование Celery — [RabbitMQ](https://www.rabbitmq.com/) в качестве брокера и [Redis](https://redis.io/) в качестве backend.

Итак, теперь у нас есть 6 контейнеров, я напишу их в файле `docker-compose.yml`:

```yaml
version: '2'
services:
  rabbit:
    image: rabbitmq:3-management
    ports:
      - "15672:15672"
      - "5672:5672"
    environment:
      - RABBITMQ_DEFAULT_VHOST=vhost
      - RABBITMQ_DEFAULT_USER=guest
      - RABBITMQ_DEFAULT_PASS=guest
  redis:
    image: library/redis
    command: redis-server /usr/local/etc/redis/redis.conf
    expose:
      - "6379"
    ports:
      - "6379:6379"
  recommendation_worker:
    image: recommendation_image
    command: celery worker -A recommendation.celeryapp:app -l info -Q recommendation_worker -c 1 -n recommendation_worker@%h -Ofair
  periodic_worker:
    image: recommendation_image
    command: celery worker -A recommendation.celeryapp:app -l info -Q periodic_worker -c 1 -n periodic_worker@%h -Ofair
  beat:
    image: recommendation_image
    command: <not sure>
  web:
    image: web_image
    command: python web_server.py
```

Оба файла докеров, которые создают **recommendation\_image** и **web\_image**, должны установить библиотеку **Celery**. Код задач должен быть только в **recommendation\_image**, потому что **workers** будут обрабатывать эти задачи:

**RecommendationDockerfile:**

```docker
FROM python:2.7-wheezy
RUN pip install celery
COPY tasks_src_code..
```

**WebDockerfile:**

```docker
FROM python:2.7-wheezy
RUN pip install celery
RUN pip install bottle 
COPY web_src_code..
```

Другие образы (`rabbitmq:3-management` и `library/redis` доступны в Docker Hub, и они будут загружены автоматически, когда вы запустите `docker-compose up`).

Теперь вот что: на вашем веб-сервере вы можете запускать задачи **Celery** по их строковому имени и получать результат по идентификаторам задач (без совместного использования кода) `web_server.py`:

```python
import bottle
from celery import Celery
rabbit_path = 'amqp://guest:guest@rabbit:5672/vhost'
celeryapp = Celery('recommendation', broker=rabbit_path)
celeryapp.config_from_object('config.celeryconfig')

@app.route('/trigger_task', method='POST')
def trigger_task():
    r = celeryapp.send_task('calculate_recommendations_task', args=(1, 2, 3))
    return r.id

@app.route('/trigger_task_res', method='GET')
def trigger_task_res():
    task_id = request.query['task_id']
    result = celery.result.AsyncResult(task_id, app=celeryapp)
    if result.ready():
        return result.get()
    return result.state
```

последний файл `config.celeryconfig.py`:

```python
CELERY_ROUTES = {
    'calculate_recommendations_task': {
        'exchange': 'recommendation_worker',
        'exchange_type': 'direct',
        'routing_key': 'recommendation_worker'
    }
}
CELERY_ACCEPT_CONTENT = ['pickle', 'json', 'msgpack', 'yaml']
```
