# Celery shared\_task

{% hint style="info" %}
**Оригинальное название**: Celery shared\_task

**Ссылка**: [https://appliku.com/post/celery-shared\_task](https://appliku.com/post/celery-shared\_task)

**Автор**: [**Appliku Crew**](https://appliku.com/post/celery-shared\_task)

**Дата**:
{% endhint %}

Декоратор `"shared_task"` позволяет создавать задачи Celery для повторно используемых приложений, поскольку для него не требуется экземпляр приложения Celery.

Это также более простой способ определить задачу, поскольку вам не нужно импортировать экземпляр приложения Celery.

* shared\_task
* Простейшая shared\_task
* shared\_task с политикой повтора
* shared\_task с этим должен запускаться максимум сразу

Если вы ищете руководство по развертыванию Django и Celery, обязательно ознакомьтесь с одним из них:

* [Развертывание Django в облаке Hetzner](https://appliku.com/post/deploy-django-hetzner-cloud)
* [Развертывание Django в Digital Ocean Droplet](https://appliku.com/post/deploy-django-digital-ocean-droplet)
* [Развертывание Django в AWS Lightsail](https://appliku.com/post/deploy-django-aws-lightsail)
* [Развертывание Django в AWS EC2](https://appliku.com/post/deploy-django-to-aws-ec2)
* [Развертывание Django на Linode](https://appliku.com/post/deploy-django-linode)
* [Развертывание Django на Google Cloud Platform](https://appliku.com/post/deploy-django-google-cloud-platform-gcp)

## shared\_task

Распределенная (shared) задача имеет множество параметров, и вам нужно решить, какой из них использовать в зависимости от типа задачи.

Некоторые вещи, которые следует учитывать:

* нормально ли, если задача выполняется несколько раз в определенных условиях или она должна запускаться только один раз и рисковать невыполнением?
* должна ли задача повторяться, в каких случаях и сколько раз?
* вам нужно сохранить результат задачи?
* вам нужен доступ к статусу задачи вне задачи?
* вам нужно знать, началось ли задание на самом деле?

Ответы на эти вопросы помогут вам определить правильный набор параметров, которые необходимо передать декоратору **@shared\_task**.

Один из параметров, который я рекомендую всегда устанавливать, — это явное имя задачи **name**.

Совет по именованию — использовать некоторый префикс для модуля или цели задачи, чтобы очень распространенное имя не конфликтовало между разными частями приложения или с другими библиотеками.

```python
# our_api/tasks.py
@shared_task(name="api_check_availability")
def check_availability():
    pass
```

Эту задачу можно вызвать через **import** следующим образом:

```python
# our_api.tasks.py
from our_api.tasks import check_availability

check_availability.delay()
check_availability.apply_async()
```

Если вы получаете доступ к нему из `models.py`, а `tasks.py` импортирует что-либо из моделей — вы получите циклическую ошибку импорта, и задачи не будут инициализированы. Кроме того, если вы отправляете задачу из совершенно другого приложения, работающего с тем же экземпляром Celery, вы не можете использовать импорт и, следовательно, не можете использовать `.delay` или `.apply_async`.

В этом случае вы можете вызвать задачу по имени.

Давайте посмотрим пример вызывающей задачи из файла `models.py`.

```python
# our_api/models.py
from celery import current_app
from django.db import transaction

class Post(models.Model):
    title = models.CharField(max_length=255)

    def generate_og_images(self):
        pass

    def save(self, *args, **kwargs):
        super().save(*args, **kwargs)
        transaction.on_commit(
            lambda: current_app.send_task(
                "content_generate_og_images",
                kwargs={"post_pk": self.pk}
            )
        )
```

```python
# our_api/tasks.py

from our_api.models import Post


@shared_task(name="content_generate_og_images")
def content_generate_og_images(post_pk):
    try:
            post = Post.objects.get(pk=post_pk)
    except Post.DoesnNotExist():
            return False
```

В данном случае мы импортируем модель **Post** из задач. Мы не можем импортировать задачу из модели, чтобы использовать `.delay()` или `.apply_async()`, потому что это приведет к циклическому импорту, поэтому мы просто отправляем задачу по имени.

Мы рассмотрим, какие аргументы следует устанавливать для каждого типичного случая.

## Простейшая shared\_task

Если у вас есть задача, результат которой вас не особо волнует, и вы хотели бы снизить влияние ее выполнения на производительность, вы можете определить ее следующим образом:

```python
@shared_task(name="simple_task", ignore_result=True)
def simple_task():
    pass
```

Для такой задачи у вас не может быть стратегии повторной попытки, поскольку она не имеет `bind=True`, а благодаря `ignore_result=True` она даже не будет подключаться к серверной части результатов, поскольку в ней нет необходимости.

## shared\_task с политикой повтора

Этот пример подходит для вызова внешнего API. Поскольку API может быть неработоспособен, рекомендуется повторить попытку.

Кроме того, это должна быть идемпотентная задача, например операция чтения. Это означает, что нас устраивают потенциально множественные вызовы этого API в случае сбоя с обеих сторон.

```python
@shared_task(
    name="read_from_external_api",
    bind=True,
    acks_late=True,
    autoretry_for=(Exception,),
    max_retries=5,
    retry_backoff=True,
    retry_backoff_max=500,
    retry_jitter=True)
def read_from_external_api(url):
    result = requests.get(url)
        return result.json()
```

Здесь у нас есть `acks_late=True` на случай, если задача завершится сбоем из-за сбоя воркера, повторите попытку для любого исключения **Exception** и повторите попытку до 5 раз.

## shared\_task с этим должен запускаться максимум сразу

В этом примере мы отправляем запрос, который требует больших затрат или по какой-то другой причине не должен повторяться более одного раза.

```python
@shared_task(
    name="expensive_api_call",
    bind=True,
    acks_late=False,)
def read_from_external_api(url):
    result = requests.post(url, {})
        return result.json()
```

Воркер подтвердит задачу перед ее выполнением, поэтому, если задача не удастся выполнить из-за сбоя воркера, запрос не повторится. Если вызов API (цель задачи) все еще должен произойти, его следует выполнить другими способами, возможно, вручную пользователем.

## Похожие сообщения

* [Django Celery Multiple Queues, when and how to use them](https://appliku.com/post/django-celery-multiple-queues-when-and-how-use-the)
* [Flower Celery Monitoring](https://appliku.com/post/celery-flower)
* [Celery Task Priority](https://appliku.com/post/celery-task-priority)
* [Django Celery Tutorial to Background tasks](https://appliku.com/post/django-celery-tutorial-to-background-tasks)
* [Celery Groups and Chords](https://appliku.com/post/celery-groups-and-chords)
