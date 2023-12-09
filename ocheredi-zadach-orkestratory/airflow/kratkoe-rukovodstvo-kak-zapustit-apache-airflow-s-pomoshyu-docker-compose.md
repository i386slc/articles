# Краткое руководство: как запустить Apache Airflow с помощью docker-compose

{% hint style="info" %}
**Оригинальное название**: [Quick guide: How to run Apache Airflow with docker-compose](https://xnuinside.medium.com/quick-guide-how-to-run-apache-airflow-cluster-in-docker-compose-615eb8abd67a)

**Автор**: [Iuliia Volkova](https://xnuinside.medium.com/?source=post\_page-----615eb8abd67a--------------------------------)

**Дата**: 30 июля 2019
{% endhint %}

**UPD от 20 июля 2020**: эта статья довольно старая, и когда я ее писала, официального образа Docker еще не существовало, поэтому в статье я использовала `puckel/docker-airflow`. Но теперь Apache Airflow уже создал и поддерживает готовый к использованию официальный образ докера. Вы можете прочитать больше об этом здесь: [https://github.com/apache/airflow/blob/master/IMAGES.rst#airflow-docker-images](https://github.com/apache/airflow/blob/master/IMAGES.rst#airflow-docker-images)

Я подготовила docker-compose на официальном образе Docker, его можно найти здесь: [https://github.com/xnuinside/airflow\_in\_docker\_compose/blob/master/docker-compose-with-celery-executor.yml](https://github.com/xnuinside/airflow\_in\_docker\_compose/blob/master/docker-compose-with-celery-executor.yml) (с Apache Airflow версии 1.10.11) и файлом `.env` для него [https://github.com/xnuinside/airflow\_in\_docker\_compose/blob/master/.env](https://github.com/xnuinside/airflow\_in\_docker\_compose/blob/master/.env). Надеюсь, скоро у меня будет время создать статью с пояснениями.

**UPD от 29.11.2019**: обновление версий до 1.10.6

**ОБНОВЛЕНИЕ UPPER 1.10.5 ВНИМАНИЕ**: Вам необходимо определить `'default_pool'` для экземпляров задач и установить для него числовые слоты, например, 100. Раньше в этом не было необходимости, поскольку **default\_poll** существовал по умолчанию. **Но теперь вам нужно создать его вручную**. Так что просто зайдите в пользовательский интерфейс, Admin-> Pools (`http://localhost:8080/admin/pool/`) и нажать **Create**. Создайте пул с именем `'default_pool'` и слотами, например 100.

В этой статье предполагается, что вы уже знаете, что такое [Apache Airflow](https://airflow.readthedocs.io/en/stable/) и зачем он нужен. И вы здесь, потому что вам нужен простой способ использования и разработки рабочего процесса в локальной среде. Что можно спроектировать близко к реальному производству, или вы просто не хотите запускать глобальный Apache Airflow.

<figure><img src="../../.gitbook/assets/111.webp" alt=""><figcaption></figcaption></figure>

Итак, это очень простое и очень быстрое руководство о том, как активировать **Apache Airflow** с помощью [docker-compose](https://docs.docker.com/compose/) и работать с ним. Я работаю с macOS, так что все работает на ней на 100%, я думаю, то же самое должно быть близко к истине и для пользователей Linux.

<figure><img src="../../.gitbook/assets/222.webp" alt=""><figcaption></figcaption></figure>

## Что мы будем делать? Сначала запустим Airflow с LocalExecutor и PostgreSQL

Мы создадим файл docker-compose с:

* веб-сервер airflow
* airflow scheduler (с LocalExecutor)
* БД PostgreSQL (мы будем использовать его в качестве базы данных для Airflow)

Для **PostgreSQL** мы будем использовать официальный образ докера: [https://hub.docker.com/\_/postgres/](https://hub.docker.com/\_/postgres/) версии `9.6.14`, для **Apache Airflow**: [https://hub.docker.com/r/puckel/docker- airflow/](https://hub.docker.com/r/puckel/docker-airflow/) версия 1.10.6 (примечание: мы перейдём на использование официального образа докера, как только он будет выпущен, теперь мы можем просто проверить AIP и подождать или поучаствовать, если можете и хотите, в процессе создания).

Вначале для быстрой разработки мы будем использовать одну папку DAG, которую мы будем использовать планировщиком Airflow и веб-сервером Airflow, поэтому вам не нужно будет пересобирать и перезапускать сервер каждый раз, когда вы захотите добавить даги. Для этого мы будем использовать **volumes**.

### Предварительные условия

На первом этапе вам понадобится Docker (с поддержкой docker-compose версии 3), если по каким-то причинам в 2019 году вы его так и не установили — пожалуйста, погуглите, как его установить на вашу ОС. Или воспользуйтесь официальными руководствами: [https://docs.docker.com/install/linux/docker-ce/ubuntu/](https://docs.docker.com/install/linux/docker-ce/ubuntu/) (Ubuntu) и [https://docs.docker.com/docker-for-mac/install/](https://docs.docker.com/docker-for-mac/install/) (Mac).

### Шаг 1. Подготовьте airflow.cfg и Dockerfile.

Для работы с БД PostgreSQL нам необходимо изменить `airflow.cfg` (если у вас нет `airflow.cfg`, вы можете взять его здесь: [https://github.com/xnuinside/airflow\_in\_docker\_compose/blob/master/airflow.cfg](https://github.com/xnuinside/airflow\_in\_docker\_compose/blob/master/airflow.cfg)).

Нам нужно изменить переменную `sql_alchemy_conn =` с использования **sqlite** на **PostgreSQL**.

<figure><img src="../../.gitbook/assets/333.webp" alt=""><figcaption></figcaption></figure>

Итак, в `airflow.cfg` у вас будет:

```ini
sql_alchemy_conn = postgresql+psycopg2://$your_db_user:$your_db_password@$your_postgres_db_host:$postgres_port/$db_name

# в моем примере это будет:
sql_alchemy_conn = postgresql+psycopg2://airflow:airflow@postgres:5432/airflow
```

Честно говоря, нам не нужно изменять отдельный `airflow.cfg` и помещать внутрь файл образа докера с помощью **COPY**. Поскольку с помощью образа **puckle**, который мы используем, вы можете просто установить параметры БД с переменными среды в `docker-compose.yml` (см. источники, как это сделать — [https://github.com/puckel/docker-airflow/blob/master/script/entrypoint.sh](https://github.com/puckel/docker-airflow/blob/master/script/entrypoint.sh)). Но я добавляю этот способ «с отдельным файлом cfg», чтобы иметь больше способов настройки. Так что просто добавьте/перезапишите все, что вам нужно, в Dockerfile — и будьте счастливы.

Кроме того, поскольку в образе **puckle** есть скрипт `entrypoint.sh` с переменными синтаксического анализа и изменением cfg на основе этих переменных — **мы перезапишем точку входа в файле docker-compose**, чтобы быть уверенным, что мы запускаем чистый airflow без каких-либо переменных и изменений перед запуском.

1.1 Подготовьте основную папку (в исходниках **git** она называется **airflow\_in\_docker\_compose**), в нее будут помещены все файлы.

1.2 В папку **main\_folder** поместите файл `airflow.cfg` с измененной переменной `sql_alchemy_conn =` (описано выше).

1.3 Далее создайте файл с именем **Dockerfile** (это имя по умолчанию для файлов Docker для создания образов, давайте его использовать)

В **Dockerfile**:

```docker
FROM puckel/docker-airflow:1.10.6
RUN pip install --user psycopg2-binary
ENV AIRFLOW_HOME=/usr/local/airflow
COPY ./airflow.cfg /usr/local/airflow/airflow.cfg
```

Краткое описание: **FROM** — определяем базовый образ, который мы используем для создания нашего образа, **ENV** — определяем переменные среды в образе, **COPY** — копируем файл по локальному пути в образ, **RUN** — запускаем команду.

1.4 Также создайте файл с именем `docker-compose.yml` и оставьте его пустым.

После этого вы получили структуру:

```
…/main_folder 
   — airflow.cfg
   — Dockerfile
   - docker-compose.yml
```

### Шаг 2. Подготовьте файл docker-compose.

2.1 Теперь пришло время заполнить `docker-compose.yml`:

```yaml
version: "3"
services:

  postgres:
    image: "postgres:9.6"
    container_name: "postgres"
    environment:
      - POSTGRES_USER=airflow
      - POSTGRES_PASSWORD=airflow
      - POSTGRES_DB=airflow
    ports:
      - "5432:5432"
    volumes:
      - ./data/postgres:/var/lib/postgresql/data

  # закомментируйте initdb после того, как будете использовать его при первом запуске
  initdb:
    build: .
    entrypoint: airflow initdb
    depends_on:
      - postgres

  webserver:
    build: .
    restart: always
    depends_on:
      - postgres
    volumes:
      - ./airflow_files/dags:/usr/local/airflow/dags
    ports:
      - "8080:8080"
    entrypoint: airflow webserver
    healthcheck:
      test: ["CMD-SHELL", "[ -f /usr/local/airflow/airflow-webserver.pid ]"]
      interval: 30s
      timeout: 30s
      retries: 3

  scheduler:
    build: .
    restart: always
    depends_on:
      - postgres
    volumes:
      - ./airflow_files/dags:/usr/local/airflow/dags
    entrypoint: airflow scheduler
    healthcheck:
      test: ["CMD-SHELL", "[ -f /usr/local/airflow/airflow-scheduler.pid ]"]
      interval: 30s
      timeout: 30s
      retries: 3
```

Что такое проверка работоспособности, вы можете прочитать здесь: [https://docs.docker.com/engine/reference/builder/#healthcheck](https://docs.docker.com/engine/reference/builder/#healthcheck). Но, вы можете удалить этот раздел, тоже все будет работать корректно.

Обратите внимание на тома:

```yaml
volumes:
      - ./data/postgres:/var/lib/postgresql/data
```

и

```yaml
volumes:
      - ./airflow_files/dags:/usr/local/airflow/dags
```

Путь в левой части томов перед `":"` (`./airflow_files/dags` и `./data/postgres`) — это папки на вашем локальном компьютере (не в образе докера!!!), которые будут доступны для образов докеров как путь с правой стороны.

Итак, если вы поместите файл в `main_folder/airflow/dags` (`'.'` — означает текущий рабочий каталог, в котором находится `docker-compose.yaml`, то есть это **main\_folder**), он будет отображаться в образах Airflow в `/usr/local/ airflow/dags`, поэтому веб-сервер Airflow и планировщик Airflow увидят его.

Вы также можете добавить тома для **папки плагинов airflow**, если вам это нужно.

2.2 Сохраните `docker-compose.yml`.

### Шаг 3. Поднимите кластер docker-compose.

В первый раз и каждый раз, когда вы будете очищать папку данных Postgres (в нашем примере — каталог `./data/postgres`), вам нужно запускать службу postgres в отдельном окне терминала и ждать, пока она создаст базу данных и инициализировать папку данных. Для этого урока этого будет достаточно, но…

Если вы не хотите делать это с помощью способа, вы можете потратить немного времени и создать .sh-скрипт для добавления служб airflow в docker-compose, подождите, пока БД не будет создана, то есть это вы можете увидеть, например, здесь : `https://github.com/vishnubob/wait-for-it`, но, конечно, вам нужно изменить sh, чтобы проверить, доступна ли БД для использования или нет.

3.1 Запустим:

```bash
# в нашем случае вы должны запустить команду docker-compose
# из папки с файлом docker-compose (main_folder),
# это не обязательно, просто для упрощения руководства.
docker-compose up postgres
```

3.1.1 после этого в другом окне терминала запускаем **initdb** (он нам понадобится только один раз, если мы будем очищать БД и начинать все с нуля)

```bash
docker-compose up initdb
```

3.2 После этого запустите в **другом окне терминала**:

```bash
docker-compose up scheduler webserver
```

Как вы видите, postgres, scheduler и webserver — имена наших сервисов, которые мы определили в `docker-compose.yml`.

При следующих запусках вы можете просто использовать (не забудьте прокомментировать службу **initdb** в `docker-compose.yml`):

```bash
docker-compose up
```

3.3 Теперь войдите в пользовательский интерфейс и проверьте, что все работает успешно по адресу `http://localhost:8080`. Оно должно быть пустым:

<figure><img src="../../.gitbook/assets/airflow-1.webp" alt=""><figcaption></figcaption></figure>

### Шаг 4. Добавьте DAG.

Для тестирования нашей инфраструктуры мы будем использовать простой **BashOperator** DAG из примеров Apache Airflow.

4.1 Берём здесь  (нужен весь файл как DAG):

[https://github.com/apache/airflow/blob/master/airflow/example\_dags/example\_bash\_operator.py](https://github.com/apache/airflow/blob/master/airflow/example\_dags/example\_bash\_operator.py)

И скопируйте его в папку **dags**, которая была определена в **разделе томов файла docker-compose**.

Итак, вы получили структуру основной папки:

```
…/main_folder 
   — airflow.cfg
   — Dockerfile
   - docker-compose.yml
   airflow_files/
      dags/
        - example_bash_operator.py
```

4.2 Подождите 10–15 секунд и проверьте пользовательский интерфейс, обновите его и подождите еще, если он все еще пуст.

Вы должны увидеть свою новую группу обеспечения доступности баз данных в пользовательском интерфейсе.

<figure><img src="../../.gitbook/assets/airflow-2.webp" alt=""><figcaption></figcaption></figure>

### Шаг 5. Запустите DAG и протестируйте его.

Теперь переключите Dag из состояния паузы (измените `"Off"` на `"On"`) и подождите несколько секунд. Поскольку у него есть расписание и дата начала в прошлом, он будет автоматически подхвачен планировщиком airflow и запущен.

В пользовательском интерфейсе у вас будут зеленые запуски DAG:

<figure><img src="../../.gitbook/assets/airflow-3.webp" alt=""><figcaption></figcaption></figure>

А еще можно проверить консоль и увидеть на ней что-то подобное с информацией о запуске

<figure><img src="../../.gitbook/assets/airflow-4.webp" alt=""><figcaption></figcaption></figure>

## Что дальше?

[Как запустить Airflow в docker-compose с несколькими работниками Celery](https://medium.com/@xnuinside/quick-tutorial-apache-airflow-with-3-celery-workers-in-docker-composer-9f2f3b445e4).

[Установите зависимости Python для создания кластера Docker без повторной сборки образов](https://medium.com/@xnuinside/install-python-dependencies-to-docker-compose-cluster-without-re-build-images-8c63a431e11c).

Все исходники вы можете найти здесь: [https://github.com/xnuinside/airflow\_in\_docker\_compose](https://github.com/xnuinside/airflow\_in\_docker\_compose).
