# Table of contents

## Очереди задач, оркестраторы

* [Airflow](README.md)
  * [Краткое руководство: как запустить Apache Airflow с помощью docker-compose](ocheredi-zadach-orkestratory/airflow/kratkoe-rukovodstvo-kak-zapustit-apache-airflow-s-pomoshyu-docker-compose.md)
  * [Мониторинг показателей Airflow с помощью Grafana](ocheredi-zadach-orkestratory/airflow/monitoring-pokazatelei-airflow-s-pomoshyu-grafana.md)
* [Celery](ocheredi-zadach-orkestratory/celery/README.md)
  * [Создание удаленного воркера Celery для Flask с отдельной базой кода](ocheredi-zadach-orkestratory/celery/sozdanie-udalennogo-vorkera-celery-dlya-flask-s-otdelnoi-bazoi-koda.md)
  * [Как вызвать задачу Celery из другого приложения](ocheredi-zadach-orkestratory/celery/kak-vyzvat-zadachu-celery-iz-drugogo-prilozheniya.md)
  * [Как отправлять сообщения Celery удаленному воркеру](ocheredi-zadach-orkestratory/celery/kak-otpravlyat-soobsheniya-celery-udalennomu-vorkeru.md)
  * [Масштабирование Celery — отправка задач на удаленные машины](ocheredi-zadach-orkestratory/celery/masshtabirovanie-celery-otpravka-zadach-na-udalennye-mashiny.md)
  * [Запуск Celery в Windows 10](ocheredi-zadach-orkestratory/celery/zapusk-celery-v-windows-10.md)
  * [Celery shared\_task](ocheredi-zadach-orkestratory/celery/celery-shared_task.md)
* [Celery Docker](ocheredi-zadach-orkestratory/celery-docker/README.md)
  * [Докеризация Celery и Django](ocheredi-zadach-orkestratory/celery-docker/dokerizaciya-celery-i-django.md)
  * [Несколько контейнеров Docker и Celery](ocheredi-zadach-orkestratory/celery-docker/neskolko-konteinerov-docker-i-celery.md)
  * [Мульти-Celery внутри докер-контейнера](ocheredi-zadach-orkestratory/celery-docker/multi-celery-vnutri-doker-konteinera.md)
  * [Celery с docker-compose](ocheredi-zadach-orkestratory/celery-docker/celery-s-docker-compose.md)
  * [Github gist к статье Django+Celery](ocheredi-zadach-orkestratory/celery-docker/github-gist-k-state-django+celery.md)
* [Celery workflow (chains, groups, etc.)](ocheredi-zadach-orkestratory/celery-workflow-chains-groups-etc./README.md)
  * [Цепи, группы и хорды в Celery](ocheredi-zadach-orkestratory/celery-workflow-chains-groups-etc./cepi-gruppy-i-khordy-v-celery.md)
  * [Интересные случаи рабочих процессов с Celery](ocheredi-zadach-orkestratory/celery-workflow-chains-groups-etc./interesnye-sluchai-rabochikh-processov-s-celery.md)

## win32api

* [Печать win32](win32api/pechat-win32/README.md)
  * [Win32 как распечатать?](win32api/pechat-win32/win32-kak-raspechatat.md)

## Django

* [Django модели](django/django-modeli/README.md)
  * [Добавление к классам](django/django-modeli/dobavlenie-k-klassam.md)
  * [Введение в поля выбора Enum для Django](django/django-modeli/vvedenie-v-polya-vybora-enum-dlya-django.md)
  * [Использование пользовательских классов для полей модели](django/django-modeli/ispolzovanie-polzovatelskikh-klassov-dlya-polei-modeli.md)
  * [Поле модели — работа Django ORM — часть 2](django/django-modeli/pole-modeli-rabota-django-orm-chast-2.md)
  * [Методы переопределения для определения пользовательского поля модели в django](django/django-modeli/metody-pereopredeleniya-dlya-opredeleniya-polzovatelskogo-polya-modeli-v-django.md)
* [Django запросы (query)](django/django-zaprosy-query/README.md)
  * [Сила Q-объектов Django](django/django-zaprosy-query/sila-q-obektov-django.md)
  * ["Добавление" объектов Q в Django](django/django-zaprosy-query/dobavlenie-obektov-q-v-django.md)
  * [Как динамически фильтровать queryset](django/django-zaprosy-query/kak-dinamicheski-filtrovat-queryset.md)
  * [Prefetch Related и Select Related в Django](django/django-zaprosy-query/prefetch-related-i-select-related-v-django.md)
  * [Запрос JSONField в Django](django/django-zaprosy-query/zapros-jsonfield-v-django.md)
* [Django формы](django/django-formy/README.md)
  * [Как использовать Django Widget Tweaks](django/django-formy/kak-ispolzovat-django-widget-tweaks.md)
  * [Учебник по Django Formsets — создание динамических форм с помощью Htmx](django/django-formy/uchebnik-po-django-formsets-sozdanie-dinamicheskikh-form-s-pomoshyu-htmx.md)
  * [Детальное понимание модельных Django formets и их расширенное использование](django/django-formy/detalnoe-ponimanie-modelnykh-django-formets-i-ikh-rasshirennoe-ispolzovanie.md)
  * [Formsets и Inlines](django/django-formy/formsets-i-inlines.md)
  * [Представления на основе классов Django с несколькими встроенными наборами форм](django/django-formy/predstavleniya-na-osnove-klassov-django-s-neskolkimi-vstroennymi-naborami-form.md)
  * [Объяснить фабрику встроенных форм Django на примере?](django/django-formy/obyasnit-fabriku-vstroennykh-form-django-na-primere.md)
  * [Как использовать вложенные наборы форм в django](django/django-formy/kak-ispolzovat-vlozhennye-nabory-form-v-django.md)
  * [Работа с вложенными формами в Django](django/django-formy/rabota-s-vlozhennymi-formami-v-django.md)
  * [Django inline formset factory с примерами](django/django-formy/django-inline-formset-factory-s-primerami.md)
  * [Как использовать выбор даты с Django](django/django-formy/kak-ispolzovat-vybor-daty-s-django.md)
  * [Обработка нескольких входных значений для одного поля формы Django](django/django-formy/obrabotka-neskolkikh-vkhodnykh-znachenii-dlya-odnogo-polya-formy-django.md)
* [Django менеджеры](django/django-menedzhery/README.md)
  * [Совет #11. Пользовательский менеджер с цепочками запросов](django/django-menedzhery/sovet-11.-polzovatelskii-menedzher-s-cepochkami-zaprosov.md)
* [Django DB funcs & expressions & aggregates](django/django-db-funcs-and-expressions-and-aggregates/README.md)
  * [Написание пользовательских функций БД](django/django-db-funcs-and-expressions-and-aggregates/napisanie-polzovatelskikh-funkcii-bd.md)
  * [Советы по работе с базами данных](django/django-db-funcs-and-expressions-and-aggregates/sovety-po-rabote-s-bazami-dannykh.md)
  * [Агрегация с django-filter через прокси-модели](django/django-db-funcs-and-expressions-and-aggregates/agregaciya-s-django-filter-cherez-proksi-modeli.md)
* [Django settings](django/django-settings/README.md)
  * [Разница между STATIC\_URL и STATIC\_ROOT](django/django-settings/raznica-mezhdu-static_url-i-static_root.md)
* [Django deploy](django/django-deploy/README.md)
  * [Докеризация приложения Django — Gunicorn и Nginx](django/django-deploy/dokerizaciya-prilozheniya-django-gunicorn-i-nginx.md)
* [Django gis](django/django-gis/README.md)
  * [Создание веб-приложения на основе местоположения с помощью Django и GeoDjango](django/django-gis/sozdanie-veb-prilozheniya-na-osnove-mestopolozheniya-s-pomoshyu-django-i-geodjango.md)
  * [Настройка Geodjango-GDAL для Windows 10](django/django-gis/nastroika-geodjango-gdal-dlya-windows-10.md)
* [Django async](django/django-async/README.md)
  * [Планирование задач с помощью APScheduler](django/django-async/planirovanie-zadach-s-pomoshyu-apscheduler.md)
  * [Интеграция APScheduler и django\_apscheduler в проект Django](django/django-async/integraciya-apscheduler-i-django_apscheduler-v-proekt-django.md)
* [Django lookup](django/django-lookup/README.md)
  * [Пользовательские запросы в Django (lookup)](django/django-lookup/polzovatelskie-zaprosy-v-django-lookup.md)
  * [Пользовательский поиск \_\_date для Django](django/django-lookup/polzovatelskii-poisk-__date-dlya-django.md)

## docker

* [ENV, ARG](docker/env-arg/README.md)
  * [Docker ARG, ENV и .env — полное руководство](docker/env-arg/docker-arg-env-i-.env-polnoe-rukovodstvo.md)
* [Docker-compose](docker/docker-compose/README.md)
  * [ENV](docker/docker-compose/env/README.md)
    * [Приоритет переменных среды в Docker Compose](docker/docker-compose/env/prioritet-peremennykh-sredy-v-docker-compose.md)
  * [YAML](docker/docker-compose/yaml/README.md)
    * [Общие переменные в docker-compose.yml](docker/docker-compose/yaml/obshie-peremennye-v-docker-compose.yml.md)
* [Graylog](docker/graylog/README.md)
  * [Как запустить сервер Graylog в контейнерах Docker](docker/graylog/kak-zapustit-server-graylog-v-konteinerakh-docker.md)
* [NFS](docker/nfs/README.md)
  * [Что такое сетевая файловая система NFS?](docker/nfs/chto-takoe-setevaya-failovaya-sistema-nfs.md)
  * [Общий доступ к файлам: NFS, SMB и CIFS](docker/nfs/obshii-dostup-k-failam-nfs-smb-i-cifs.md)
  * [Установка NFS-сервера в Ubuntu](docker/nfs/ustanovka-nfs-servera-v-ubuntu.md)
  * [NFS Docker Volumes: как создавать и использовать](<README (1).md>)
  * [Монтирование общих ресурсов NFS внутри контейнера Docker](docker/nfs/montirovanie-obshikh-resursov-nfs-vnutri-konteinera-docker.md)

## DRF

* [DRF exceptions](drf/drf-exceptions/README.md)
  * [Проверка запросов и обработка пользовательских исключений в DRF](drf/drf-exceptions/proverka-zaprosov-i-obrabotka-polzovatelskikh-isklyuchenii-v-drf.md)
* [DRF routers](drf/drf-routers/README.md)
  * [Написание пользовательских маршрутов в DRF Viewsets](drf/drf-routers/napisanie-polzovatelskikh-marshrutov-v-drf-viewsets.md)
  * [Общие и вложенные маршрутизаторы](drf/drf-routers/obshie-i-vlozhennye-marshrutizatory.md)

## Python

* [JSON](python/json/README.md)
  * [Преобразование данных JSON в пользовательский объект Python](python/json/preobrazovanie-dannykh-json-v-polzovatelskii-obekt-python.md)
* [Декораторы, дескрипторы](python/dekoratory-deskriptory/README.md)
  * [Когда декоратор встречает дескриптор](python/dekoratory-deskriptory/kogda-dekorator-vstrechaet-deskriptor.md)
  * [Класс декоратора Python для оформления автономных функций и методов объектов](python/dekoratory-deskriptory/klass-dekoratora-python-dlya-oformleniya-avtonomnykh-funkcii-i-metodov-obektov.md)
* [Работа с файлами](python/rabota-s-failami/README.md)
  * [Как создать Watchdog в Python](python/rabota-s-failami/kak-sozdat-watchdog-v-python.md)
* [Асинхронная работа](python/asinkhronnaya-rabota/README.md)
  * [Асинхронные запросы в Python](python/asinkhronnaya-rabota/asinkhronnye-zaprosy-v-python.md)
  * [Как использовать asyncio to\_thread()](python/asinkhronnaya-rabota/kak-ispolzovat-asyncio-to_thread.md)
  * [Как использовать asyncio.gather()](python/asinkhronnaya-rabota/kak-ispolzovat-asyncio.gather.md)
* [Многопоточная работа](python/mnogopotochnaya-rabota/README.md)
  * [Как остановить поток в Python](python/mnogopotochnaya-rabota/kak-ostanovit-potok-v-python.md)

## Requests, git, setup

* [dynaconf](requests-git-setup/dynaconf/README.md)
  * [from dynaconf import settings](requests-git-setup/dynaconf/from-dynaconf-import-settings.md)
* [requests](requests-git-setup/requests/README.md)
  * [Тайм-ауты в Python requests](requests-git-setup/requests/taim-auty-v-python-requests.md)
  * [Установка времени ожидания для запросов Python](requests-git-setup/requests/ustanovka-vremeni-ozhidaniya-dlya-zaprosov-python.md)
* [setuptools](requests-git-setup/setuptools/README.md)
  * [Использование веток Git с Setuptools](requests-git-setup/setuptools/ispolzovanie-vetok-git-s-setuptools.md)
* [pyenv](requests-git-setup/pyenv/README.md)
  * [Python: управление версиями с помощью Pyenv и Pyenv-Virtualenv (Linux)](requests-git-setup/pyenv/python-upravlenie-versiyami-s-pomoshyu-pyenv-i-pyenv-virtualenv-linux.md)
  * [Как управлять виртуальными средами Python](requests-git-setup/pyenv/kak-upravlyat-virtualnymi-sredami-python.md)
* [gunicorn](requests-git-setup/gunicorn/README.md)
  * [Повышение производительности за счет оптимизации конфигурации Gunicorn](requests-git-setup/gunicorn/povyshenie-proizvoditelnosti-za-schet-optimizacii-konfiguracii-gunicorn.md)
  * [Как правильно выбрать тип воркера Gunicorn для прода](requests-git-setup/gunicorn/kak-pravilno-vybrat-tip-vorkera-gunicorn-dlya-proda.md)

## FastAPI

* [dynaconf+fastapi](fastapi/dynaconf+fastapi/README.md)
  * [Динамическая настройка приложений Python — FastAPI Pro (часть 1)](fastapi/dynaconf+fastapi/dinamicheskaya-nastroika-prilozhenii-python-fastapi-pro-chast-1.md)

## WEB-серверы

* [NGINX](web-servery/nginx/README.md)
  * [Nginx PHP не работает при загрузке больших файлов (более 6 ГБ)](web-servery/nginx/nginx-php-ne-rabotaet-pri-zagruzke-bolshikh-failov-bolee-6-gb.md)

## Monitoring, Logging

* [Prometheus](monitoring-logging/prometheus/README.md)
  * [Безопасность Node Exporter Prometheus](monitoring-logging/prometheus/bezopasnost-node-exporter-prometheus.md)
  * [Безопасность Prometheus](monitoring-logging/prometheus/bezopasnost-prometheus.md)
  * [Масштабирование мониторинга Kubernetes с помощью Prometheus Federation](monitoring-logging/prometheus/masshtabirovanie-monitoringa-kubernetes-s-pomoshyu-prometheus-federation.md)
  * [Prometheus Federation ⏤ Руководство по масштабированию Prometheus](monitoring-logging/prometheus/prometheus-federation-rukovodstvo-po-masshtabirovaniyu-prometheus.md)
  * [Высокая доступность и иерархическая федерация с Prometheus](monitoring-logging/prometheus/vysokaya-dostupnost-i-ierarkhicheskaya-federaciya-s-prometheus.md)
  * [Настройка стека мониторинга — Часть 4: Blackbox exporter](monitoring-logging/prometheus/nastroika-steka-monitoringa-chast-4-blackbox-exporter.md)
