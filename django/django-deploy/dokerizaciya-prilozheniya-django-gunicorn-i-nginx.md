# Докеризация приложения Django — Gunicorn и Nginx

{% hint style="info" %}
Ссылка на оригинальную статью: [Dockerizing Django Application — Gunicorn and Nginx](https://blog.devgenius.io/dockerizing-django-application-gunicorn-and-nginx-5a74b250198f)

Опубликовано: 17 февраля 202

Автор: [Adesh Nalpet Adimurthy](https://pyblog.medium.com/?source=---two\_column\_layout\_sidebar----------------------------------)
{% endhint %}

<figure><img src="../../.gitbook/assets/1 Gr_0hyRyqwJiYKAKFQSHdw.webp" alt=""><figcaption></figcaption></figure>

Пошаговое руководство по докеризации приложения Django с базой данных **MySQL** с использованием **Guinicorn** и **Nginx**.

В этом посте предполагается, что вы немного знакомы с Django и, вероятно, имеете локальный проект разработки Django и изучаете, как развернуть проект в рабочей среде.

В следующем разделе поста мы увидим, как развернуть проект на [AWS EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html).

## Контрольный список

1. Проверьте, запущено ли приложение на вашем локальном компьютере: `python manage.py runserver`, доступный по адресу `http://localhost:8000`.
2. Не забудьте запустить: `python manage.py makemigrations`, `python manage.py migrate` и `python manage.py collectstatic`.

## План

1. Мы не можем использовать сервер разработки Django в производстве; он предназначен для локальной разработки и не может обрабатывать параллельные запросы — поэтому мы будем использовать комбинацию **Guinicorn** и **Nginx**.
2. Создайте файлы докера для приложения Django, работающего на порту `8000`, и используйте **Nginx** для проксирования входящего запроса с порта `80` на порт `8000`.
3. Файл **docker-compose**, чтобы собрать все вместе, определить сеть, ссылки и тома для статических и мультимедийных файлов сервера.

## Dockerfile для приложения Django

1. Убедитесь, что у вас есть файл **requirements.txt** со всеми зависимостями. Вы можете сгенерировать его с помощью: `pip freeze > requirements.txt`
2. Пример **Dockerfile** ниже предназначен для приложения Django с базой данных MySQL и может потребовать незначительных изменений для других баз данных.

```docker
FROM ubuntu:20.04
ADD . /app
WORKDIR /app

RUN apt-get update -y
RUN apt-get install software-properties-common -y
RUN add-apt-repository ppa:deadsnakes/ppa
RUN apt-get install python3.9 -y
RUN apt-get install python3-pip -y
RUN python3.9 -m pip install --upgrade setuptools
RUN apt-get install sudo ufw build-essential libpq-dev libmysqlclient-dev python3.9-dev default-libmysqlclient-dev libpython3.9-dev -y
RUN python3.9 -m pip install -r requirements.txt
RUN python3.9 -m pip install psycopg2-binary
RUN sudo ufw allow 8000

EXPOSE 8000
```

На момент написания этого поста [Ubuntu: 20.04](https://hub.docker.com/\_/ubuntu) и [python: 3.9](https://www.python.org/downloads/release/python-390/) были последними стабильными версиями.

Поместите файл докера в корень проекта, в тот же каталог, что и **manage.py**.

## Dockerfile для Nginx
