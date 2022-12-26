# Разница между STATIC\_URL и STATIC\_ROOT

{% hint style="info" %}
Ссылка на оригинальную статью (**stackoverflow**): [Difference between static STATIC\_URL and STATIC\_ROOT on Django](https://stackoverflow.com/questions/8687927/difference-between-static-static-url-and-static-root-on-django)

Опубликовано: 31 декабря 2011

Автор: [user993563](https://stackoverflow.com/users/993563/user993563)
{% endhint %}

Меня смущает `static root`, и я хочу прояснить ситуацию.

Для обслуживания статических файлов в Django в `settings.py` и `urls.py` должно быть следующее:

```python
import os
PROJECT_DIR = os.path.dirname(__file__)
```

### 1. Абсолютный путь к каталогу, в котором должны быть собраны статические файлы

```python
STATIC_ROOT = os.path.join(PROJECT_DIR,'static_media/')
```

### 2. Префикс URL для статических файлов

```
STATIC_URL = '/static/'
```

### 3. Дополнительные места для статических файлов

```python
STATICFILES_DIRS = ( os.path.join(PROJECT_DIR,'static/'),)
```

... и в `urls.py` следующие строки:

```python
from django.contrib.staticfiles.urls import staticfiles_urlpatterns

urlpatterns += patterns('', (
    r'^static/(?P<path>.*)$',
    'django.views.static.serve',
    {'document_root': settings.STATIC_ROOT}
))
```

### 4. Мы также используем python manage.py collectstatic

Вопросы:

1. Может ли кто-нибудь объяснить мне рабочий процесс: как все должно быть сделано в идеале. На данный момент я копирую/вставляю приведенные выше фрагменты кода в назначенные места и продолжаю создавать новые файлы в статическом каталоге, и это работает. Однако в моих `settings.STATIC_ROOT` я указал на другой каталог.
2. Было бы здорово, если бы кто-нибудь мог объяснить рабочий процесс каждого параметра: как файлы собираются и управляются, и что было бы хорошей практикой для подражания.

## Ответ 1

### STATIC\_ROOT

> Абсолютный путь к каталогу, в котором `./manage.py collectstatic` будет собирать статические файлы для развертывания. **Пример**: `STATIC_ROOT="/var/www/example.com/static/"`

теперь команда `./manage.py collectstatic` **скопирует** все статические файлы (т. е. в папке **static** в ваших приложениях, статические файлы по всем путям) в каталог `/var/www/example.com/static/`. Теперь вам нужно только обслуживать этот каталог на **apache** или **nginx** или др.

### STATIC\_URL

> **URL**-адрес, по которому обслуживаются статические файлы в каталоге **STATIC\_ROOT** (Apache или nginx..etc). **Пример**: `/static/` или `http://static.example.com/`

Если вы установите `STATIC_URL = 'http://static.example.com/'`, вы должны обслуживать папку **STATIC\_ROOT** (т.е. `"/var/www/example.com/static/"`) с помощью apache или nginx по адресу `'http: //static.example.com/'`(чтобы вы могли ссылаться на статический файл `'/var/www/example.com/static/jquery.js'` с помощью `'http://static.example.com/jquery.js'` )

Теперь в ваших шаблонах django вы можете ссылаться на него:

```django
{% raw %}
{% load static %}
<script src="{% static "jquery.js" %}
{% endraw %}"></script>
```

который будет отображаться в:

```html
<script src="http://static.example.com/jquery.js"></script>
```

## Ответ 2

**STATICFILES\_DIRS**: здесь вы можете хранить статические файлы для вашего проекта, например те, которые используются вашими шаблонами.

**STATIC\_ROOT**: оставьте это поле пустым, когда вы сделаете `manage.py collectstatic`, он будет искать все статические файлы в вашей системе и перемещать их сюда. Ваш статический файловый сервер должен быть привязан к этой папке, где бы он ни находился. Проверьте это после запуска **collectstatic**, и вы обнаружите структуру каталогов, созданную django.

Как указал @DarkCygnus, **STATIC\_ROOT** должен указывать на каталог в вашей файловой системе, папка должна быть пустой, так как она будет заполнена Django.

```python
STATIC_ROOT = os.path.join(BASE_DIR, 'staticfiles')
```

или

```python
STATIC_ROOT = '/opt/web/project/static_files'
```

**STATIC\_URL**: `'/static/'` обычно подходит, это просто префикс для статических файлов.

## Ответ 3 (подробный)

#### Обновление за май 2022 г.:

Прежде всего, я объясняю **STATIC\_ROOT**, затем **STATIC\_URL**.

### \<STATIC\_ROOT>

