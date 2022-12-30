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

1. Создайте каталог **nginx** с двумя файлами: **Dockerfile** и **default.conf** (название каталога может быть любым).
2. Содержимое этих двух файлов следующее:

```docker
FROM nginx:stable-alpine

COPY default.conf /etc/nginx
COPY default.conf /etc/nginx/conf.d

EXPOSE 80
```

```nginx
server {
    listen 80 default_server;
    server_name _;
    
    location / {
        proxy_pass http://web:8000;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;
    }
    location /static/ {
        alias /app/static/;
    }
    location /media/ {
        alias /app/static/;
    }
}
```

Почему `proxy_pass http://web:8000`? Мы подойдем к этому через секунду.

## Файл docker-compose

Даже перед отправкой образа в хаб **Docker** рекомендуется несколько раз протестировать приложение, создав образ **Docker** локально. Следовательно, два файла для [docker compose](https://docs.docker.com/compose/), один для локальной разработки, а другой для производственного использования.

```yaml
version: '3.8'

services:
  web:
    build:
      context: .
    command: gunicorn --bind 0.0.0.0:8000 <project-name>.wsgi --workers=4
    volumes:
      - static_volume:/app/static
      - media_volume:/app/media
    expose:
      - "8000"
    networks:
      - django-network

  nginx:
    build: nginx
    restart: always
    volumes:
      - static_volume:/app/static
      - media_volume:/app/media
    ports:
      - "80:80"
    depends_on:
      - web
    networks:
      - django-network

networks:
  django-network:
    name: django-network

volumes:
  media_volume:
  static_volume:
```

Команда для запуска контейнеров докеров: `docker-compose up -f docker-compose.dev.yml`

### Примечание:

1. **web** относится к веб-приложению Django, имя службы может быть любым, но обязательно укажите это же имя в файле конфигурации **nginx**.
2. В идеале мы хотели бы обслуживать статический контент (HTML и CSS) из CDN для более быстрой доставки контента, а не из инстанса **EC2** (рассмотрено это в последующем посте).
3. Вместо использования сети по умолчанию лучше определить явную сеть, такую как **django-network**.
4. Обратите внимание, что служба **web** имеет **expose**, а не **ports**; **expose** гарантирует, что порт `8000` доступен только в локальной сети, а не с хост-компьютера (внешнего мира).

Точно так же для производственного использования создайте и отправьте образы **Django** и **Nginx** в Docker хаб (убедитесь, что не включаете конфиденциальную информацию, используя **.dockerignore**).

```bash
*.pyc
migrations/
__pycache__
db.sqlite3
.idea
*.DS_Store
.env
static
```

Запустите в корне проекта (тот же уровень **manage.py**): `docker build -t <docker-username>/<project-name>:<tag|latest> .` и `docker push <docker-username>/<project-name>:<tag|latest>`

Запустите внутри созданного ранее каталога **nginx**: `docker build -t <docker-username>/<nginx-for-project-name>:<tag|latest> .` и `docker push <docker-username>/<nginx-for-project-name>:<tag|latest>`

Файл **docker-compose** для производственного использования будет выглядеть так:

```yaml
version: '3.8'

services:
  web:
    image: <docker-username>/<project-name>:<tag|latest>
    command: gunicorn --bind 0.0.0.0:8000 licensing_platform.wsgi --workers=4
    volumes:
      - static_volume:/app/static
      - media_volume:/app/media
    expose:
      - "8000"
    networks:
      - django-network

  nginx:
    image: <docker-username>/<nginx-for-project-name>:<tag|latest>
    restart: always
    volumes:
      - static_volume:/app/static
      - media_volume:/app/media
    ports:
      - "80:80"
    depends_on:
      - web
    networks:
      - django-network

networks:
  django-network:
    name: django-network

volumes:
  media_volume:
  static_volume:
```

Единственное изменение здесь — использовать образы из **Docker Hub**, а не создавать их локально.

Команда для запуска контейнеров докеров: `docker-compose up` Используйте флаг `-d` для запуска в фоновом режиме: `docker-compose -d up`

Вы можете идти; для использования переменных **env** внутри контейнера я предпочитаю явно указывать переменные **env**, которые будут использоваться в контейнерах докеров, которые находятся на хост-компьютере, поэтому файл **docker-compose**:

```yaml
version: '3.8'

services:
  web:
    image: <docker-username>/<project-name>:<tag|latest>
    command: gunicorn --bind 0.0.0.0:8000 licensing_platform.wsgi --workers=4
    environment:
      - DEBUG
      - DATABASE_NAME
      - DATABASE_USER
      - DATABASE_PASSWORD
      - HOST_ENDPOINT
      - REDIS_LOCATION
    volumes:
      - static_volume:/app/static
      - media_volume:/app/media
    expose:
      - "8000"
    networks:
      - django-network
  nginx:
    image: <docker-username>/<nginx-for-project-name>:<tag|latest>
    restart: always
    volumes:
      - static_volume:/app/static
      - media_volume:/app/media
    ports:
      - "80:80"
    depends_on:
      - web
    networks:
      - django-network

networks:
  django-network:
    name: django-network

volumes:
  media_volume:
  static_volume:
```

Чтобы получить доступ к локальному серверу **MySQL** (для тестирования на локальном компьютере), имя хоста: **host.docker.internal**

В рабочей среде используйте **AWS RDS** — **MySQL** с правильными правилами входящего трафика в группе безопасности инстансов **EC2**.

```bash
DEBUG=True
DATABASE_NAME = ''
DATABASE_USER = ''
DATABASE_PASSWORD = ''
HOST_ENDPOINT = 'host.docker.internal'
REDIS_LOCATION = 'redis://127.0.0.1:6379/'
```

Wohoo Запуск готов!

Все намного проще, если есть проект для справки; вот он: [https://github.com/addu390/licensing-as-a-platform](https://github.com/addu390/licensing-as-a-platform)

## Развертывание — EC2

1. Хотя использовать [ECS](https://aws.amazon.com/ecs/) гораздо проще, я предпочитаю не ограничиваться использованием управляемых сервисов облачного провайдера.
2. Это просто: запустите инстанс **EC2**, установите [Docker Engine](https://docs.docker.com/engine/install/) и [Docker Compose](https://docs.docker.com/compose/install/) и создайте **AMI**; это будет ваш **Docker AMI**.
3. Завершите работу и запустите новый экземпляр с **Docker AMI**, ранее созданным в качестве базового **AMI** для **EC2**, настройте в соответствии с вашими потребностями, включите необходимые пользовательские данные, импортируйте `docker-compose.yml` (скорее всего, с помощью **git**) и запустите `docker-compose up`.
4. Вам, конечно, придется написать сценарий оболочки для автоматизации запуска экземпляров **EC2**.

Как правило, создайте **ALB** (балансировщик нагрузки приложений) с [целевой группой](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html) и [группой автоматического масштабирования](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html).

Вскоре я напишу дополнительный пост, посвященный [AMI](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html), [EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html), [RDS](https://aws.amazon.com/rds/), [ALB](https://aws.amazon.com/elasticloadbalancing/), [VPC](https://aws.amazon.com/vpc/), [группам безопасности](https://docs.aws.amazon.com/vpc/latest/userguide/VPC\_SecurityGroups.html#VPCSecurityGroups) и всему остальному, что необходимо для масштабируемого веб-приложения.
