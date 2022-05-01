# Масштабирование Celery — отправка задач на удаленные машины

{% hint style="info" %}
**Оригинальное название**: Scaling Celery - Sending Tasks To Remote Machines

**Ссылка**: [https://avilpage.com/2014/11/scaling-celery-sending-tasks-to-remote.html](https://avilpage.com/2014/11/scaling-celery-sending-tasks-to-remote.html)

**Автор**: **Chillar Anand**

**Дата**: 9 ноября 2014
{% endhint %}

## Celery:

Celery — это пакет Python, который реализует механизм очереди задач с упором на обработку в реальном времени, а также поддерживает планирование задач. Он имеет 3 основных компонента.

* **Приложение Celery Application (или клиент Client)**: оно отвечает за добавление задач в очередь.
* **Celery Worker (или сервер Server)**: он отвечает за выполнение поставленных перед ним задач.
* **Брокер (Broker)**: отвечает за передачу сообщений между клиентом и сервером.

## Что вы должны знать:

Вы должны знать основы Celery, и вы должны быть знакомы с созданием задач Celery

```python
from celery import Celery

app = Celery('tasks', backend='amqp',
             broker='amqp://<user>:<password>@<ip>/<vhost>')

@app.task
def add(x, y):
    return x + y
```

добавлением задач в очередь,

```python
add.apply_async(args=[1,2])
```

и потреблением задач воркером

```bash
celery worker -A my_app -l info
```

Если для ваших задач не требуется много системных ресурсов, вы можете настроить их все на одном компьютере. Но если у вас много заданий, которые потребляют ресурсы, то вам нужно распределить их по нескольким машинам.

В этом уроке давайте переместим наших Celery Workers на удаленную машину, оставив клиента и брокера на одной машине.

## Отправка задач на другую машину:

### На машине А:

* Установите Celery и RabbitMQ.
* Настройте RabbitMQ, чтобы машина B могла к нему подключиться.

```python
# добавить нового пользователя
sudo rabbitmqctl add_user <user> <password>

# добавить новый виртуальный хост
sudo rabbitmqctl add_vhost <vhost_name>

# установить права для пользователя на vhost
sudo rabbitmqctl set_permissions -p <vhost_name> <user> ".*" ".*" ".*"

# перезапуск rabbit
sudo rabbitmqctl restart
```

* Создайте новый файл `remote.py` с простой задачей. Здесь у нас есть брокер, установленный на машине A. Поэтому укажите IP-адрес машины 1 в опции URL-адреса брокера.

```python
from celery import Celery

app = Celery('tasks', backend='amqp',
             broker='amqp://<user>:<password>@<ip>/<vhost>')

def add(x, y):
    return x + y
```

* Теперь у нас все настроено на машине А. Теперь мы можем поставить некоторые задачи в очередь.

```python
In [1]: from remote import add

In [2]: add.delay(1, 2)
Out[2]: <AsyncResult: 3eb96a11-aa61-46d3-9b9d-e0e1703438d0>

In [3]: b.delay(2, 3)
Out[3]: <AsyncResult: ec40db1a-a43c-4486-9530-0a3153fe1380>

In [4]: b.delay(3, 4)
Out[4]: <AsyncResult: ca53a4c7-061b-408e-82ee-86c2d43d21a0>
```

Все настроено на машине А. Теперь давайте перейдем к машине B.

### На машине В:

* Установите Celery.
* Скопируйте файл `remote.py` с компьютера A на этот компьютер.
* Запустите воркер, чтобы потреблять задачи

```bash
celery worker -l info -A remote
```

Как только вы запустите воркера, вы получите задачи, поставленные в очередь, и они сразу же выполнятся.

```bash
[2014-11-09 00:01:19,168: INFO/MainProcess] Received task: remote.add[c2d2bb27-ff5f-47da-b2b9-6fb11669ee1a]
[2014-11-09 00:01:19,170: INFO/MainProcess] Received task: remote.add[8daa1a5c-17d0-46dc-9c93-faf7fbeccdd9]
[2014-11-09 00:01:19,172: INFO/MainProcess] Received task: remote.add[79603d15-24f1-43f8-b8b7-525b7cd4b9a2]
[2014-11-09 00:01:19,401: INFO/MainProcess] Task remote.add[8daa1a5c-17d0-46dc-9c93-faf7fbeccdd9] succeeded in 0.226168102003s: 3
[2014-11-09 00:01:19,462: INFO/MainProcess] Task remote.add[c2d2bb27-ff5f-47da-b2b9-6fb11669ee1a] succeeded in 0.286503815001s: 5
[2014-11-09 00:01:19,464: INFO/MainProcess] Task remote.add[79603d15-24f1-43f8-b8b7-525b7cd4b9a2] succeeded in 0.288741396998s: 7
```

Это простое руководство о том, как отправлять задачи на удаленные машины.

В зависимости от ваших потребностей вам, возможно, придется настроить кластер серверов и маршрутизировать задачи в соответствии с масштабом.
