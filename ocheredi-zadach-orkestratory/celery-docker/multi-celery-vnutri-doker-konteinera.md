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

В зависимости от потребностей и дизайна вашего приложения вы можете захотеть разделить воркеры в разных контейнерах для разных задач.

Однако, если использование ресурсов низкое и имеет смысл объединить несколько рабочих процессов в одном контейнере, вы можете сделать это с помощью сценария точки входа.

**Редакция 2019-12-05**: после запуска этого на некоторое время. Это не очень хорошая идея для производственного использования. 2 предостережения:

1. Существует риск того, что фоновый worker тихо выйдет, но не будет захвачен на переднем плане. `tail-f` будет продолжать работать, но докер не будет знать, что фоновый рабочий процесс остановлен. В зависимости от настроек уровня отладки Celery в журналах могут отображаться некоторые указания, но докеру неизвестно, когда вы выполняете `docker ps`. Чтобы быть надежными, воркеры должны перезапускаться при сбое, что приводит нас к предложениям по использованию **supervisord**.
2. Когда контейнер запускается и останавливается (но не удаляется), состояние докер-контейнера сохраняется. Это означает, что если ваши рабочие процессы celery зависят от pid-файла для идентификации, и все же происходит неизящное завершение работы, есть вероятность, что pid-файл будет сохранен, и рабочий процесс **не будет перезапущен корректно** даже при `docker stop; docker start`. Это связано с тем, что запуск Celery обнаруживает наличие оставшегося файла PID от предыдущего нечистого завершения работы. Чтобы предотвратить несколько экземпляров, перезапущенный рабочий процесс останавливается с сообщением «PIDfile found, celery is already running?». Весь контейнер должен быть удален с помощью `docker rm` или `docker-compose down; docker-compose up`. Несколько способов справиться с этим:
   * контейнер должен быть запущен `run` с флагом `--rm`, чтобы удалить контейнер после его остановки.
   * возможно, лучше не включать параметр `--pidfile` в команду `celery multi` или `celery worker`.

_Сводная рекомендация: возможно, лучше использовать **supervisord**_.

Теперь о деталях:

Контейнеры Docker нуждаются в запущенной задаче переднего плана, иначе контейнер завершится. Это будет рассмотрено ниже.

Кроме того, celery workers могут запускать длительные задачи и должны реагировать на сигнал выключения докера (**SIGTERM**) для [корректного завершения работы](https://docs.celeryproject.org/en/latest/reference/celery.worker.html#celery.worker.WorkController.stop), т. е. завершения длительных задач перед выключением или перезапуском.

Чтобы добиться распространения и обработки сигнала Docker, лучше всего объявить точку входа **entrypoint** в файле docker в [exec form](https://docs.docker.com/engine/reference/builder/#entrypoint), вы также можете сделать это в [файле docker-compose](https://docs.docker.com/compose/compose-file/#entrypoint).

Кроме того, поскольку `celery multi` работает в фоновом режиме, докер не видит никаких логов. Вам нужно будет отображать журналы на переднем плане, чтобы `docker logs` могли видеть, что происходит. Мы сделаем это, настроив файл журнала для мультиворкеров celery и отобразив его на переднем плане консоли с помощью `tail -f <logfile_pattern>` для бесконечной работы.

Нам нужно достичь трех целей:

1. Запускать контейнер Docker с задачей переднего плана
2. Получать, перехватывать **trap** и обрабатывать сигналы отключения докеров
3. Останавливать воркеры изящно

Для #1 мы запустим `tail -f &`, а затем подождем [wait](https://linuxhint.com/wait\_command\_linux/) его как задачу переднего плана.

Для #2 это достигается установкой функции **trap** и захватом сигнала. Чтобы получать и обрабатывать сигналы с помощью функции trap, **wait** должно быть запущенной задачей переднего плана, достигнутой в #1.

Для #3 мы запустим `celery multi stop <number_of_workers_in_start_command>` и другие параметры аргументов во время запуска в `celery multi start`.

Вот [gist](https://gist.github.com/VKen/4a86bda8f65d76d8106f68587cd64327), которую я написал, скопировал сюда:

```bash
#!/bin/sh

# предохранительный выключатель, выход из скрипта, если есть ошибка.
# Полная команда ярлыка `set -e`
set -o errexit
# предохранительный выключатель, неинициализированные переменные остановят скрипт.
# Полная команда ярлыка `set -u`
set -o nounset

# функция демонтажа
teardown()
{
    echo " Signal caught..."
    echo "Stopping celery multi gracefully..."

    # отправить сигнал отключения воркеру Celery через `celery multi`
    # команда должна отражать некоторые из аргументов `celery multi start`
    celery -A config.celery_app multi stop 3 --pidfile=./celery-%n.pid --logfile=./celery-%n%I.log

    echo "Stopped celery multi..."
    echo "Stopping last waited process"
    kill -s TERM "$child" 2> /dev/null
    echo "Stopped last waited process. Exiting..."
    exit 1
}

# Старт 3 Celery воркеров `celery multi` с объявленным лог-файлом для `tail -f`
celery -A config.celery_app multi start 3 -l INFO -Q:1 queue1 -Q:2 queue1 -Q:3 queue3,celery -c:1-2 1 \
    --pidfile=./celery-%n.pid \
    --logfile=./celery-%n%I.log

# начать захват сигналов (docker отправит `SIGTERM` для отключения)
trap teardown SIGINT SIGTERM

# tail все журналы непрерывно на консоль для `docker logs`, чтобы увидеть
tail -f ./celery*.log &

# захватить идентификатор id процесса `tail` для демонтажа
child=$!

# Ожидание `tail -f` на неопределенный срок и позволяет внешним сигналам,
# включая сигналы остановки докера, которые должны быть захвачены `trap`
wait "$child"

```

Используйте приведенный выше код в качестве содержимого файла сценария точки входа и измените его в соответствии с вашими потребностями.

Объявите его в файле **dockerfile** или **docker-compose** в форме **exec**:

```docker
ENTRYPOINT ["entrypoint_file"]
```

Рабочие celery затем могут запускаться в контейнере докеров, а также могут быть изящно остановлены.

## Ответ 3

Во-первых, я не понимаю преимущества использования **multi & docker**. Насколько я понимаю, вы хотите, чтобы каждый воркер находился в отдельном контейнере. Таким образом, у вас есть гибкость и среда микросервисов.

Если вы все еще хотите иметь несколько рабочих процессов в одном контейнере, я могу предложить обходной путь, чтобы ваш контейнер оставался открытым, добавив `while true; do sleep 2; done` до конца вашей команды: `celery multi start 2 -l INFO -c:1 3 -c:2 7 -Q:1 queue1 -Q:2 queue2,queue3 && while true; sleep 2; done`.

В качестве альтернативы, оберните его в короткий скрипт:

```bash
#!/bin/bash
celery multi start 2 -l INFO -c:1 3 -c:2 7 -Q:1 queue1 -Q:2 queue2,queue3
while true; do sleep 2; done
```
