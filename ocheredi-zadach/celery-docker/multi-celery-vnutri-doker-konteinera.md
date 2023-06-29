# Мульти-Celery внутри докер-контейнера

{% hint style="info" %}
**Оригинальное название**: Celery multi inside docker container

**Ссылка**: [https://stackoverflow.com/questions/48646745/celery-multi-inside-docker-container](https://stackoverflow.com/questions/48646745/celery-multi-inside-docker-container)

**Автор**: [dluhhbiu](https://stackoverflow.com/users/2223918/dluhhbiu)

**Дата**: 6 февраля 2018
{% endhint %}

## Вопрос

У меня есть приложение Python с Celery в контейнерах Docker. Я хочу иметь несколько workers с другой очередью. Например:

```bash
celery worker -c 3 -Q queue1
celery worker -c 7 -Q queue2,queue3
```

Но я не делаю этого в Docker. Я узнал о мульти Celery. Я пытался использовать его.

```yaml
version: '3.2'
services:
  app:
    image: "app"
    build:
      context: .
    networks:
      - net
    ports:
      - 5004:5000
    stdin_open: true
    tty: true
    environment:
      FLASK_APP: app/app.py
      FLASK_DEBUG: 1
    volumes:
      - .:/home/app
  app__celery:
    image: "app"
    build:
      context: .
    command: sh -c 'celery multi start 2 -l INFO -c:1 3 -c:2 7 -Q:1 queue1 -Q:2 queue2,queue3'

```

Но я понял...

```bash
app__celery_1  |    > celery1@1ab37081acb9: OK
app__celery_1  |    > celery2@1ab37081acb9: OK
app__celery_1 exited with code 0
```

И мой контейнер с Celery закрывается. Как не дать ему закрыться и получить от него свои логи?

UPD: `Celery multi` создал фоновые процессы. Как запустить `celery multi` на переднем плане?

## Ответ 1

Я сделал это задание так. Я использовал **supervisord** вместо `celery multi`. Супервизор запускается на переднем плане, а мой контейнер не закрыт.

```bash
command: supervisord -c supervisord.conf
```

И я добавил все очереди в `supervisord.con`

```ini
[program:celery]
command = celery worker -A app.celery.celery -l INFO -c 3 -Q q1
directory = %(here)s
startsecs = 5
autostart = true
autorestart = true
stopwaitsecs = 300
stderr_logfile = /dev/stderr
stderr_logfile_maxbytes = 0
stdout_logfile = /dev/stdout
stdout_logfile_maxbytes = 0

[program:beat]
command = celery -A app.celery.celery beat -l INFO --pidfile=/tmp/beat.pid
directory = %(here)s
startsecs = 5
autostart = true
autorestart = true
stopwaitsecs = 300
stderr_logfile = /dev/stderr
stderr_logfile_maxbytes = 0
stdout_logfile = /dev/stdout
stdout_logfile_maxbytes = 0

[supervisord]
loglevel = info
nodaemon = true
pidfile = /tmp/supervisord.pid
logfile = /dev/null
logfile_maxbytes = 0

```

## Ответ 2
