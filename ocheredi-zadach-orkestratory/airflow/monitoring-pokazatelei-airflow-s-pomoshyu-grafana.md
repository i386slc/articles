# Мониторинг показателей Airflow с помощью Grafana

{% hint style="info" %}
**Оригинальное название**: [Monitoring Airflow Metrics with Grafana](https://medium.com/@perkasaid.rio/monitoring-airflow-metrics-with-grafana-29ebb43100a3)

**Автор**: [Rio Dwi Putra Perkasa](https://medium.com/@perkasaid.rio?source=post\_page-----29ebb43100a3--------------------------------)

**Дата**: 19 сентября 2021
{% endhint %}

<figure><img src="../../.gitbook/assets/airflow-grafana-1.webp" alt=""><figcaption><p>Airflow x Grafana</p></figcaption></figure>

Airflow — важный инструмент в мире обработки данных. Airflow — это инструмент планирования, который позволяет обеспечить своевременную доставку данных, выполнить этап преобразования данных и проверить зависимости для каждого процесса. Иногда вы сталкиваетесь с проблемами с Airflow и хотите знать, что произошло. Вот почему вам нужно следить за Airflow.

По сути, в вашей установке airflow предварительно установлен демон под названием statsd. Statsd отправит метрики на определенный порт. Эти метрики будут использоваться для нашего мониторинга. Процесс будет похож на схему ниже.

<figure><img src="../../.gitbook/assets/airflow-grafana-2.webp" alt=""><figcaption><p>Мониторинг Airflow с помощью Grafana</p></figcaption></figure>

В этом уроке мы рассмотрим среду внутри Docker-контейнера на вашем локальном компьютере. Продолжим предварительное условие. Установите это необходимое условие, прежде чем переходить к следующему шагу.

## Предварительное условие

* [Docker](https://docs.docker.com/get-docker/)

## Установка Airflow

Прежде всего, вам нужно установить то, что вы хотите отслеживать, да, Airflow. Я не буду подробно рассказывать об установке Airflow в Docker, поскольку у них уже есть официальная документация о том, [как запустить Airflow в Docker](https://airflow.apache.org/docs/apache-airflow/stable/start/docker.html). Пожалуйста, следуйте инструкциям выше, чтобы установить Airflow в Docker с помощью Docker Compose.

### Настройка установки Airflow:

* Включите пример загрузки ядра для проверки Airflow, установив `AIRFLOW__CORE__LOAD_EXAMPLES=True` в среде внутри файла `docker-compose.yml`. Эта переменная предназначена для создания примеров DAG в вашем Airflow в целях тестирования.
* Добавьте приведенную ниже конфигурацию в свой `docker-compose.yaml` в среде. Эта конфигурация позволит вашей статистике Airflow отправлять метрики.

```yaml
AIRFLOW__SCHEDULER__STATSD_ON: 'true'
AIRFLOW__SCHEDULER__STATSD_HOST: statsd-exporter
AIRFLOW__SCHEDULER__STATSD_PORT: 8125
AIRFLOW__SCHEDULER__STATSD_PREFIX: airflow
```

После этого войдите на свой веб-сервер airflow и используйте «airflow» в качестве имени пользователя и пароля. Ваш Airflow будет выглядеть так.

<figure><img src="../../.gitbook/assets/airflow-grafana-3.webp" alt=""><figcaption><p>пример данных из примера нагрузки на ядро Airflow</p></figcaption></figure>

## Установка Statsd-экспортера

После завершения установки airflow следующим шагом будет установка экспортера statsd. В statsd-exporter есть функция переназначения метрик, полученных от Airflow, и экспорта их как метрик Prometheus. Таким образом, statsd-exporter играет роль моста между airflow и prometheus.

Чтобы установить statsd-exporter, вам необходимо добавить службу в ваш docker-compose.yml для Airflow с помощью этой строки кода.

```yaml
statsd-exporter:
        image: prom/statsd-exporter
        container_name: airflow-statsd-exporter
        command: "--statsd.listen-udp=:8125 --web.listen-address=:9102"
        ports:
            - 9102:9102
            - 8125:8125/udp
```

**image**: определите место, куда вы хотите переместить докер-образ. В моем случае используется общедоступный образ из Docker Hub.

**volumes**: определите дополнительную конфигурацию сопоставления statsd. Если вы не используете дополнительную конфигурацию сопоставления, ваши метрики будут экспортированы так же, как в Prometheus. Эта конфигурация зависит от ваших потребностей. Вы можете перейти по этой ссылке [Statsd-exporter](https://hub.docker.com/r/prom/statsd-exporter), чтобы узнать, какую конфигурацию вы можете использовать в statsd-exporter. Сохраните дополнительную конфигурацию в файле yml и подключитесь к службе, используя тома, как указано выше.

**command**: вы можете перейти к документации statsd-exporter, чтобы узнать, какие команды вы можете здесь использовать. В этом случае мы используем три команды. `-- statsd.listen-udp` для определения порта, через который Airflow может передавать метрики. `-- web.listen-address`, чтобы определить, откуда Prometheus может получать метрики.

(Опционально) Вы можете добавить еще одну команду на основе документации из statsd-exporter [здесь](https://github.com/prometheus/statsd\_exporter). Например, если вы хотите переназначить показатели Airflow, чтобы упростить запрос в prometheus/grafana, вы можете добавить дополнительное сопоставление. Чтобы добавить дополнительное сопоставление, вы можете использовать statsd.mapping-config.

```yaml
statsd-exporter:
        image: prom/statsd-exporter
        container_name: airflow-statsd-exporter
        command: "--statsd.listen-udp=:8125 --web.listen-address=:9102 --statsd.mapping-config=/tmp/statsd_mapping.yml"
        ports:
            - 9102:9102
            - 8125:8125/udp
```

Вы заметите, что `statsd.mapping-config` использует файл yaml, поэтому вам необходимо определить файл yaml, содержащий дополнительную конфигурацию сопоставления. Создайте новый файл с именем `statsd_mapping.yaml` в корневом проекте и добавьте эту конфигурацию.

```yaml
mappings:
  - match: "(.+)\\.operator_successes_(.+)$"
    match_metric_type: counter
    name: "af_agg_operator_successes"
    match_type: regex
    labels:
      airflow_id: "$1"
      operator_name: "$2"
  - match: "*.ti_failures"
    match_metric_type: counter
    name: "af_agg_ti_failures"
    labels:
      airflow_id: "$1"
  - match: "*.ti_successes"
    match_metric_type: counter
    name: "af_agg_ti_successes"
    labels:
      airflow_id: "$1"
```

Это пример дополнительной конфигурации сопоставления показателей Airflow. Вы можете посетить [документацию statsd](https://github.com/prometheus/statsd\_exporter) для получения подробной информации о сопоставлении метрик и некоторых примеров переназначения метрик Airflow [здесь](https://github.com/databand-ai/airflow-dashboards/tree/main/statsd).

После определения файла конфигурации вам необходимо смонтировать его к службе statsd-exporter, используя том.

```yaml
statsd-exporter:
        image: prom/statsd-exporter
        container_name: airflow-statsd-exporter
        volumes:
            - $PWD/statsd_mapping.yml:/tmp/statsd_mapping.yml 
        command: "--statsd.listen-udp=:8125 --web.listen-address=:9102 --statsd.mapping-config=/tmp/statsd_mapping.yml"
        ports:
            - 9102:9102
            - 8125:8125/udp
```

После того, как вы добавите statsd-exporter в службу Docker Compose, вам нужно создать новый сервис.

```bash
docker-compose up -d
```

Эта команда создаст новый контейнер для statsd-exporter, который вы добавили ранее. Вы также можете проверить свой докер и найти там имя контейнера. После запуска докера вы можете перейти по адресу `http://127.0.0.1:9102`, чтобы проверить метрики, которые отправляются в statsd-exporter.

<figure><img src="../../.gitbook/assets/airflow-grafana-4.webp" alt=""><figcaption></figcaption></figure>

## Установка Prometheus

Prometheus — это набор инструментов для мониторинга и оповещения системы с открытым исходным кодом. Prometheus сохраняет показатели в виде данных временных рядов. Добавьте Prometheus в свои службы Docker-Compose, добавив следующую строку кода:

```yaml
prometheus:
    image: prom/prometheus
    container_name: airflow-prometheus
    ports:
        - 9090:9090
    volumes:
        - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
```

Вам необходимо создать конфигурацию Prometheus, чтобы установить целевой парсинг. В этом случае вам нужно очистить statsd-exporter, который мы уже создали ранее. Создайте новый файл с именем `prometheus.yml` внутри папки prometheus вашего корневого проекта.

```yaml
scrape_configs:
  - job_name: 'statsd-exporter'
    static_configs:
      - targets: ['airflow-statsd-exporter:9102']
```

Эта служба создаст контейнер с именем airflow-prometheus на порту 9090. После создания docker-compose вы можете получить доступ к Prometheus в своем веб-браузере, посетив `http://127.0.0.1:9090`. Вы увидите такой пользовательский интерфейс Prometheus:

<figure><img src="../../.gitbook/assets/airflow-grafana-5.webp" alt=""><figcaption><p>Prometheus UI</p></figcaption></figure>

В пользовательском интерфейсе Prometheus вы можете запрашивать свои показатели в поле выражения.

## Установка Grafana

После того, как вы настроили свой statsd-exporter и Prometheus, пришло время перейти к панели мониторинга. Grafana поможет вам анализировать ваши показатели с помощью настраиваемых графиков и интерактивной визуализации.

Чтобы установить Grafana, добавьте еще один сервис в свой docker-compose.

```yaml
grafana:
    image: grafana/grafana:latest
    container_name: airflow-grafana
    environment:
        GF_SECURITY_ADMIN_USER: grafana
        GF_SECURITY_ADMIN_PASSWORD: grafana
    ports:
      - 3000:3000
```

В этом параметре среды мы устанавливаем учетные данные по умолчанию для нашей Grafana.

После создания вы можете получить доступ к Grafana через порт 3000 в локальном браузере. Войдите в систему, используя пользователя admin по умолчанию и пароль admin. Вы увидите панель управления Grafana вот так:

<figure><img src="../../.gitbook/assets/airflow-grafana-6.webp" alt=""><figcaption><p>Главная страница Grafama</p></figcaption></figure>

Перейдите к левой панели навигации и выберите "Explore".

<figure><img src="../../.gitbook/assets/airflow-grafana-7.webp" alt=""><figcaption></figcaption></figure>

На этой странице вы можете увидеть поле типа запроса, в этом поле вы можете запросить свои метрики. Но сейчас вы все еще не можете выбрать источник данных для своей Grafana. Следующим шагом будет добавление Prometheus в качестве источника данных Grafana.

Создайте папку `grafana/provisioning/datasources/datasources-provision.yaml` и определите источник данных.

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://airflow-prometheus:9090
```

Приведенный выше код скажет Grafana сделать Prometheus источником данных. Кроме того, добавьте среду, чтобы установить путь подготовки.

```yaml
GF_PATHS_PROVISIONING: /grafana/provisioning
```

и тома для загрузки файла подготовки в образ докера.

```yaml
volumes:
      - ./grafana/provisioning:/grafana/provisioning
```

После этого ваша полная конфигурация сервиса для Grafana будет выглядеть так:

```yaml
grafana:
    image: grafana/grafana:latest
    container_name: airflow-grafana
    environment:
        GF_SECURITY_ADMIN_USER: grafana
        GF_SECURITY_ADMIN_PASSWORD: grafana
        GF_PATHS_PROVISIONING: /grafana/provisioning
    ports:
      - 3000:3000
    volumes:
      - ./grafana/provisioning:/grafana/provisioning
```

Теперь с помощью этой конфигурации вы можете запрашивать метрики из Grafana Explore и готовы создать панель мониторинга.

### Создание панели мониторинга

Теперь вы можете перейти на страницу панели мониторинга и попробовать создать панель мониторинга Airflow.

<figure><img src="../../.gitbook/assets/airflow-grafana-8.webp" alt=""><figcaption></figcaption></figure>

Добавьте новую пустую панель, и вы будете перенаправлены на настройку панели.

<figure><img src="../../.gitbook/assets/airflow-grafana-9.webp" alt=""><figcaption></figcaption></figure>

В левом нижнем углу вы увидите раздел запросов query, измените источник данных на Prometheus и добавьте запрос. Например, я добавлю визуализацию размера Airflow DagBag.

* Измените заголовок на "Dag Bag Size".
* Добавьте запрос «airflow\_dagbag\_size» в поле запроса.
* Измените тип визуализации в правом верхнем углу на "stats".
* Применить изменение

Ваша панель будет выглядеть так:

<figure><img src="../../.gitbook/assets/airflow-grafana-10.webp" alt=""><figcaption></figcaption></figure>

Итак, мы знаем, что в настоящее время размер нашего пакета сAirflow составляет 34. Вот как вы можете добавить новую панель на дашборд. Вы можете попробовать добавить новую панель и использовать другие визуализации. Используя все показатели, отправляемые из Airflow, вы можете создать панель мониторинга для мониторинга вашего Airflow.

## Исходный код

Полный исходный код для Docker Compose, Grafana и Prometheus можно найти здесь [https://github.com/riodpp/airflow-metrics-monitoring](https://github.com/riodpp/airflow-metrics-monitoring).

## Краткое содержание

Мониторинг Airflow с помощью Grafana может сэкономить наше время, чтобы понять, где произошел сбой, и быстро узнать, что не так с Airflow. Grafana предоставляет множество типов визуализации, позволяющих создать потрясающую и простую для понимания панель мониторинга. Кроме того, в Grafana есть система уведомлений, которая позволяет вам узнать о проблеме как можно скорее.

## Рекомендации

* [https://towardsdatascience.com/airflow-in-docker-metrics-reporting-83ad017a24eb](https://towardsdatascience.com/airflow-in-docker-metrics-reporting-83ad017a24eb)
* [https://databand.ai/blog/everyday-data-engineering-monitoring-airflow-with-prometheus-statsd-and-grafana/](https://databand.ai/blog/everyday-data-engineering-monitoring-airflow-with-prometheus-statsd-and-grafana/)
* [https://dev.to/kirklewis/metrics-with-prometheus-statsd-exporter-and-grafana-5145](https://dev.to/kirklewis/metrics-with-prometheus-statsd-exporter-and-grafana-5145)
