# Учебник по Django Formsets — создание динамических форм с помощью Htmx

{% hint style="info" %}
Ссылка на оригинальную статью: [Django Formsets Tutorial - Build dynamic forms with Htmx](https://justdjango.com/blog/dynamic-forms-in-django-htmx)

Опубликовано: 24 августа 2021

Автор: [**Matthew Freire**](https://justdjango.com/blog)****
{% endhint %}

Узнайте, как создавать динамические формы с помощью Django и Htmx.

<figure><img src="../../.gitbook/assets/django-htmx-dynamic-forms-1.webp" alt=""><figcaption></figcaption></figure>

В этом руководстве рассказывается, как создавать динамические формы в Django с использованием **Htmx**. Также будут рассмотрены основные концепции наборов форм Django. Но больше всего мы сосредоточимся на том, как заставить динамические формы выглядеть и чувствовать себя хорошо.

Вы можете найти код из этого руководства в [этом репозитории GitHub](https://github.com/justdjango/django\_htmx\_dynamic\_forms).

Если вы хотите посмотреть видео, а не читать: (_ссылка отсутствует_)

## Настройка проекта

Начните с создания проекта Django:

```bash
virtualenv env
source env/bin/activate
pip install django
django-admin startproject djforms
```

> Последняя версия Django на момент написания этого руководства — 3.2.6.

Запустите миграции:

```bash
python manage.py migrate
```

## Модели

Для этого проекта мы будем работать с тем же набором моделей. Создайте приложение Django и зарегистрируйте его в настройках:

```bash
python manage.py startapp books
```

```python
# settings.py
INSTALLED_APPS = [
    # ...
    'django.contrib.staticfiles',
    'books'
]
```

Добавьте его в **INSTALLED\_APPS** в **settings.py.** Внутри `books/models.py` добавьте следующие модели:

```python
# books/models.py
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=50)

    def __str__(self):
        return self.name

class Book(models.Model):
    author = models.ForeignKey(Author, on_delete=models.CASCADE)
    title = models.CharField(max_length=100)
    number_of_pages = models.PositiveIntegerField(default=1)

    def __str__(self):
        return self.title
```

И добавьте следующее в `books/admin.py`:

```python
# books/admin.py
from django.contrib import admin
from .models import Author, Book

class BookInLineAdmin(admin.TabularInline):
    model = Book

class AuthorAdmin(admin.ModelAdmin):
    inlines = [BookInLineAdmin]

admin.site.register(Author, AuthorAdmin)
```

Используя эти модели, мы можем создать автора и добавить этому автору столько книг, сколько захотим.

Запустите миграции:

```bash
python manage.py makemigrations
python manage.py migrate
```

## Как использовать Django Formsets
