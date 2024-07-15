# Приоритет переменных среды в Docker Compose

{% hint style="info" %}
Ссылка на оригинальную статью: [Environment variables precedence in Docker Compose](https://docs.docker.com/compose/environment-variables/envvars-precedence/)

Опубликовано: 2024

Автор: docker.docs
{% endhint %}

Если одна и та же переменная среды установлена ​​в нескольких источниках, Docker Compose следует правилу приоритета, чтобы определить значение этой переменной в среде вашего контейнера.

Эта страница содержит информацию об уровне приоритета каждого метода установки переменных среды.

Порядок приоритета (от высшего к низшему) следующий:

1. Установка с помощью [`docker compose run -e` в CLI](https://docs.docker.com/compose/environment-variables/set-environment-variables/#set-environment-variables-with-docker-compose-run---env).
2. Установка с помощью атрибута `environment` или `env_file`, но со значением, интерполированным из вашей оболочки [shell](https://docs.docker.com/compose/environment-variables/variable-interpolation/#substitute-from-the-shell) или файла среды. (либо [файл `.env`](https://docs.docker.com/compose/environment-variables/variable-interpolation/#env-file) по умолчанию, либо [аргумент `--env-file`](https://docs.docker.com/compose/environment-variables/variable-interpolation/#substitute-with---env-file) в CLI).
3. Установка, используя только [атрибут `environment`](https://docs.docker.com/compose/environment-variables/set-environment-variables/#use-the-environment-attribute) в файле Compose.
4. Использование [атрибута `env_file`](https://docs.docker.com/compose/environment-variables/set-environment-variables/#use-the-env\_file-attribute) в файле Compose.
5. Установка в образе контейнера в [директиве ENV](https://docs.docker.com/reference/dockerfile/#env). Наличие каких-либо настроек `ARG` или `ENV` в `Dockerfile` оценивается только в том случае, если нет записи Docker Compose для `environment`, `env_file` или `run --env`.

## Простой пример

В следующем примере другое значение для одной и той же переменной среды в файле `.env` и с атрибутом `environment` в файле Compose:

```yaml
cat ./webapp.env
NODE_ENV=test

cat compose.yml
services:
  webapp:
    image: 'webapp'
    env_file:
     - ./webapp.env
    environment:
     - NODE_ENV=production
```

Переменная среды, определенная с помощью атрибута `environment`, имеет приоритет.

```bash
docker compose run webapp env | grep NODE_ENV
NODE_ENV=production
```

## Расширенный пример

В следующей таблице в качестве примера используется `VALUE` — переменная среды, определяющая версию изображения.

### Как работает таблица

Каждый столбец представляет собой контекст, в котором вы можете установить значение или заменить значение `VALUE`.

Столбцы `«Среда ОС хоста»` и `".env"` файл указаны только в иллюстративных целях. На самом деле они приводят к появлению переменной в контейнере не сами по себе, а в сочетании с атрибутом `environment` или атрибутом `env_file`.

Каждая строка представляет собой комбинацию контекстов, где `VALUE` установлено, заменено или и то, и другое. В столбце `«Результат»` указано окончательное значение `VALUE` в каждом сценарии.

| docker compose run | environment атрибут | env\_file атрибут | ENV образа | Среда ОС хоста | .env файл  | Результат  |
| ------------------ | ------------------- | ----------------- | ---------- | -------------- | ---------- | ---------- |
|                    |                     |                   |            | VALUE==1.4     | VALUE==1.3 |            |
|                    |                     | VALUE==1.6        | VALUE==1.5 | VALUE==1.4     |            | VALUE==1.6 |
|                    | VALUE==1.7          |                   | VALUE==1.5 | VALUE==1.4     |            | VALUE==1.7 |
|                    |                     |                   | VALUE==1.5 | VALUE==1.4     | VALUE==1.3 | VALUE==1.5 |
| `--env VALUE=1.8`  |                     |                   | VALUE==1.5 | VALUE==1.4     | VALUE==1.3 | VALUE==1.8 |
| `--env VALUE`      |                     |                   | VALUE==1.5 | VALUE==1.4     | VALUE==1.3 | VALUE==1.4 |
| `--env VALUE`      |                     |                   | VALUE==1.5 |                | VALUE==1.3 | VALUE==1.3 |
|                    |                     | VALUE             | VALUE==1.5 | VALUE==1.4     | VALUE==1.3 | VALUE==1.4 |
|                    |                     | VALUE             | VALUE==1.5 |                | VALUE==1.3 | VALUE==1.3 |
|                    | VALUE               |                   | VALUE==1.5 | VALUE==1.4     | VALUE==1.3 | VALUE==1.4 |
|                    | VALUE               |                   | VALUE==1.5 |                | VALUE==1.3 | VALUE==1.3 |
| `--env VALUE`      | VALUE==1.7          |                   | VALUE==1.5 | VALUE==1.4     | VALUE==1.3 | VALUE==1.4 |
| `--env VALUE=1.8`  | VALUE==1.7          |                   | VALUE==1.5 | VALUE==1.4     | VALUE==1.3 | VALUE==1.8 |
| `--env VALUE=1.8`  |                     | VALUE==1.6        | VALUE==1.5 | VALUE==1.4     | VALUE==1.3 | VALUE==1.8 |
| `--env VALUE=1.8`  | VALUE==1.7          | VALUE==1.6        | VALUE==1.5 | VALUE==1.4     | VALUE==1.3 | VALUE==1.8 |

### Объяснение результата

Результат 1: Локальная среда имеет приоритет, но файл Compose не настроен на репликацию этого внутри контейнера, поэтому такая переменная не задана.

Результат 2: Атрибут `env_file` в файле Compose определяет явное значение для `VALUE`, поэтому среда контейнера устанавливается соответствующим образом.

Результат 3. Атрибут `environment` в файле Compose определяет явное значение для `VALUE`, поэтому среда контейнера устанавливается соответствующим образом.

Результат 4. Директива `ENV` образа объявляет переменную `VALUE`, и поскольку файл Compose не настроен на переопределение этого значения, эта переменная определяется образом.

Результат 5: команда `docker compose run` имеет установленный флаг `--env`, который имеет явное значение и переопределяет значение, установленное образом.

Результат 6. В команде `docker compose run` установлен флаг `--env` для репликации значения из среды. Значение ОС хоста имеет приоритет и реплицируется в среду контейнера.

Результат 7. Для команды запуска docker Compose установлен флаг --env для репликации значения из среды. Значение из файла `.env` выбирается для определения среды контейнера.

Результат 8: Атрибут `env_file` в файле Compose настроен на репликацию `VALUE` из локальной среды. Значение ОС хоста имеет приоритет и реплицируется в среду контейнера.

Результат 9: Атрибут `env_file` в файле Compose настроен на репликацию `VALUE` из локальной среды. Значение из файла `.env` выбирается для определения среды контейнера.

Результат 10: Атрибут `environment` в файле Compose настроен на репликацию `VALUE` из локальной среды. Значение ОС хоста имеет приоритет и реплицируется в среду контейнера.

Результат 11: Атрибут `environment` в файле Compose настроен на репликацию `VALUE` из локальной среды. Значение из файла `.env` выбирается для определения среды контейнера.

Результат 12: Флаг `--env` имеет более высокий приоритет, чем атрибуты `environment` и `env_file`, и должен быть установлен для репликации `VALUE` из локальной среды. Значение ОС хоста имеет приоритет и реплицируется в среду контейнера.

Результаты с 13 по 15: Флаг `--env` имеет более высокий приоритет, чем атрибуты `environment` и `env_file`, и поэтому устанавливает значение.
