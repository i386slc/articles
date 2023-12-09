# Celery с docker-compose

{% hint style="info" %}
**Оригинальное название**: [Celery with docker compose](https://stackoverflow.com/questions/75024646/celery-with-docker-compose)

**Автор**: [winter](https://stackoverflow.com/users/19130803/winter)

**Дата**: 5 января 2023
{% endhint %}

## Вопрос

У меня есть докер для моего приложения. Celery является одной из служб.

* Конманда `celery worker` работает, но
* Команда `celery multi` не работает.

```yaml
  celery:
    container_name: celery_application
    build: 
      context: .
      dockerfile: deploy/Dockerfile
    # restart: always
    networks:
      - internal_network
    env_file:
      - deploy/.common.env
    # command: ["celery", "--app=tasks", "worker", "--queues=upload_queue", "--pool=prefork", "--hostname=celery_worker_upload_queue", "--concurrency=1", "--loglevel=INFO", "--statedb=/external/celery/worker.state"]  # This is working
     command: ["celery", "-A", "tasks", "multi", "start", "default", "model", "upload", "--pidfile=/external/celery/%n.pid", "--logfile=/external/celery/%n%I.log", "--loglevel=INFO", "--concurrency=1", "-Q:default", "default_queue", "-Q:model", "model_queue", "-Q:upload", "upload_queue"]  # This is not working
    # tty: true
    # stdin_open: true
    depends_on:
      - redis
      - db
      - pgadmin
      - web
    volumes:      
      - my_volume:/external

```

Получает этот результат.

```bash
celery | celery multi v5.2.7 (dawn-chorus)
celery | > Starting nodes...
celery |     > default@be788ec5974d: 
celery | OK
celery |     > model@be788ec5974d:
celery | OK
celery |     > upload@be788ec5974d:
celery exited with code 0
```

Все службы встают, кроме Celery, который вышел с кодом 0. Что мне не хватает при использовании `celery multi`? Пожалуйста, предложите.

## Ответ 1

Команда `celery multi` не ждет завершения рабочего процесса celery, а **запускает несколько рабочих процессов celery в фоновом режиме, а затем завершает работу**. К сожалению, завершение процесса переднего плана приводит к тому, что дочерние процессы также завершаются в среде контейнера докеров.

Не рекомендуется использовать `celery multi` с таким докером, потому что любая проблема с одним воркером может не отражаться на консоли контейнера, и ваш воркер может выйти из строя, умереть или войти в вечный цикл внутри контейнера без какого-либо сигнала для управления или мониторинга. С помощью одной рабочей команды код выхода будет возвращен в контейнер докеров, и он перезапустит службу в случае завершения работы воркера.

Если вам все еще действительно нужно использовать celery multi, как это. Вы можете попробовать использовать **bash** для добавления еще одной команды цикла навсегда, чтобы предотвратить выход из контейнера:

```yaml
command: [
    "bash", "-c",
    "celery -A tasks multi start default model upload --pidfile=/external/celery/%n.pid --logfile=/external/celery/%n%I.log --loglevel=INFO --concurrency=1 -Q:default default_queue -Q:model model_queue -Q:upload upload_queue; tail -f /dev/null"
]
```

`tail -f /dev/null` сохранит ваш контейнер там навсегда, независимо от того, запущен celery worker или нет. Конечно, в вашем контейнере должен быть установлен bash.

Я предполагаю, что вы хотели бы поместить всех Celery workers в один контейнер для простоты использования. Это так? Вы можете попробовать [https://github.com/just-containers/s6-overlay](https://github.com/just-containers/s6-overlay) вместо `celery multi`. Оверлей S6 может отслеживать вашего celery worker, перезапускать его при выходе, а также предоставлять некоторые утилиты супервизора процессов, такие как `celery multi`, но он предназначен для этой цели.

## Ответ 2

Я думаю, что вам не хватает того, что контейнеры докеров (в отличие от виртуальных машин) предназначены для запуска процесса и выхода. Если вы запускаете celery, используя **multi**, вы фактически запускаете celery как [процесс демона](https://docs.celeryq.dev/en/stable/userguide/daemonizing.html), а **не** фактический процесс для запуска контейнера.

Одно из решений может быть предложено [@truong-hua](https://stackoverflow.com/users/1530486/truong-hua) - оно запустит новую оболочку (bash) в новом процессе, а затем вызовет команду `celery multi`. Celery по-прежнему будет выходить после запуска демона celery, но процесс оболочки будет преобладать. Хотя мне это кажется чрезмерным усложнением - какой тогда смысл запускать его в фоновом режиме?

Более простым решением было бы запустить прикрепленный celery (в качестве основного процесса), поэтому просто запустите, например. `celery -A proj worker --concurrency= ...` (я бы посоветовал не устанавливать параметр `--concurrency` выше **1** в докеризованной среде). Тогда celery — ваш основной процесс, который вы можете отслеживать.

Если это все еще не то, что вам нужно, вот ветка о том, [как прикрепить отсоединенный процесс](https://unix.stackexchange.com/questions/31824/how-do-i-attach-a-terminal-to-a-detached-process).
