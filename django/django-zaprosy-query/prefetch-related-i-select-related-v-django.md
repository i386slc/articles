# Prefetch Related и Select Related в Django

{% hint style="info" %}
Ссылка на оригинальную статью: [Prefetch Related and Select Related in Django](https://medium.com/codeptivesolutions/prefetch-related-and-select-related-in-django-90f07a2379c0)

Опубликовано: 31 октября 2019

Автор: [Nensi Trambadiya](https://medium.com/@nensi26?source=post\_page-----90f07a2379c0--------------------------------)
{% endhint %}

<figure><img src="../../.gitbook/assets/1 z9EoJ-b8paWE2qLwxbb9iA.webp" alt=""><figcaption></figcaption></figure>

Когда Django извлекает объект, он не извлекает связанные объекты этого объекта. Он будет делать отдельные запросы для всех связанных объектов во время доступа. Такое поведение хорошо не во всех случаях.

Давайте разберемся в этом на практическом примере.

Во-первых, мы добавляем ведение журнала в файл **settings.py** ниже, чтобы увидеть все вызовы SQL-запросов за Django ORM.

```python
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'console': {
            'level': 'DEBUG',
            'class': 'logging.StreamHandler',
        }
    },
    'loggers': {
        'django.db.backends': {
            'handlers': ['console'],
            'level': 'DEBUG',
        },
    }
}
```

Мы рассмотрим эти модели для всех выбранных связанных примеров ниже, поэтому мы всегда можем вернуться сюда и прочитать его снова.

```python
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    name= models.CharField(max_length=50)
    author = models.ForeignKey(Author,on_delete=models.CASCADE)
```

Это довольно просто, верно? Книга **Book** имеет связь внешнего ключа с таблицей **Author**.

```python
>>> book = Book.objects.get(id=1)
(0.000) SELECT "django_orms_book"."id", "django_orms_book"."name", "django_orms_book"."author_id" FROM "django_orms_book" WHERE "django_orms_book"."id" = 1; args=(1,)

>>> book.author.name
(0.000) SELECT "django_orms_author"."id", "django_orms_author"."name" FROM "django_orms_author" WHERE "django_orms_author"."id" = 1; args=(1,)
'Author1'

>>> book.author.name
'Author1'
```

Как и выше, отдельный запрос для автора выполняется, когда мы обращаемся к имени автора. Эти отдельные запросы для связанных объектов снижают производительность приложения. Предположим, у нас есть 1000 книг, и мы должны создать список книг с именем автора. Каждый раз, когда мы обращаемся к одному внешнему ключу **foreign key**, которого нет в кеше, будет выполняться другой запрос для извлечения значения. Таким образом, в итоге будет выполнен 1001 запрос для получения списка книг.

Мы можем сократить **1001 запрос до 1 запроса**, используя **select\_related**.

## Select Related

Давайте создадим запрос, который извлекает все книги с именем автора за 1 запрос.

```python
from django_orms.models import *
from django.db.models import F

def get_all_books():
    books = Book.objects.select_related('author').annotate(
        author_name=F('author__name')
    ).values('id', 'name', 'author_name')
    print(books)
```

Я определила функцию `get_all_books()` для получения всех книг с именем автора.

Теперь я вызываю эту функцию из оболочки, чтобы увидеть количество запросов.

```python
>>> get_all_books()
(0.001) SELECT "django_orms_book"."id", "django_orms_book"."name",
               "django_orms_author"."name" AS "author_name"
               FROM "django_orms_book"
               INNER JOIN "django_orms_author"
               ON ("django_orms_book"."author_id" = "django_orms_author"."id")
               LIMIT 21; args=()
<QuerySet [
    {'id': 1, 'name': 'Book1', 'author_name': 'Author1'},
    {'id': 2, 'name': 'Book2', 'author_name': 'Author2'},
    {'id': 3, 'name': 'Book3', 'author_name': 'Author3'},
    '...(remaining elements truncated)...'
]>
```

Как видно из выходных данных, для получения всех книг был вызван только один запрос соединения. Это большое улучшение для приложения.

Вы не помните, что мы не можем использовать **select\_related** для отношений многие-ко-многим? Чтобы улучшить наши запросы, нам нужно использовать новый метод **prefetch\_related**.

Теперь давайте обновим модели, добавим новые поля издателей в модель **Book** и изменим имя на модель **Person**.

```python
from django.db import models

class Person(models.Model):
    name = models.CharField(max_length=50)

class Book(models.Model):
    name= models.CharField(max_length=50)
    author = models.ForeignKey(Person,on_delete=models.CASCADE)
    publishers = models.ManyToManyField(Person, related_name='publishers')
```

## Prefetch Related

Мы можем использовать метод **prefetch\_related** с отношениями «многие ко многим», чтобы повысить производительность за счет уменьшения количества запросов.

```python
def get_all_books():
    books = Book.objects.prefetch_related('publishers')
    for book in books:
        print(book.publishers.all())
```

Давайте вызовем эту функцию и посмотрим, что произошло.

```python
>>> get_all_books()
(0.001) SELECT "django_orms_book"."id", "django_orms_book"."name",
               "django_orms_book"."author_id" FROM "django_orms_book"; args=()
(0.000) SELECT ("django_orms_book_publishers"."book_id")
               AS "_prefetch_related_val_book_id", "django_orms_person"."id",
               "django_orms_person"."name"
               FROM "django_orms_person"
               INNER JOIN "django_orms_book_publishers"
               ON
               ("django_orms_person"."id" = "django_orms_book_publishers"."person_id")
               WHERE "django_orms_book_publishers"."book_id" IN (1, 2); args=(1, 2)

<QuerySet [<Person: Person object (2)>, <Person: Person object (3)>]>
<QuerySet [<Person: Person object (1)>, <Person: Person object (3)>]>
```

Как видите, 2 запроса выполняются с использованием **prefetch\_related**.

Давайте немного изменим функцию и добавим `values()` вместо `all()`.

```python
def get_all_books():
    books = Book.objects.prefetch_related('publishers')
    for book in books:
        print(book.publishers.values('id', 'name'))
```

Посмотрим на результат.

```python
>>> get_all_books()
(0.000) SELECT "django_orms_book"."id", "django_orms_book"."name",
               "django_orms_book"."author_id" FROM "django_orms_book"; args=()
(0.000) SELECT ("django_orms_book_publishers"."book_id")
               AS "_prefetch_related_val_book_id", "django_orms_person"."id",
               "django_orms_person"."name" FROM "django_orms_person"
               INNER JOIN "django_orms_book_publishers"
               ON
               ("django_orms_person"."id" = "django_orms_book_publishers"."person_id")
               WHERE "django_orms_book_publishers"."book_id" IN (1, 2); args=(1, 2)
(0.000) SELECT "django_orms_person"."id", "django_orms_person"."name"
               FROM "django_orms_person"
               INNER JOIN "django_orms_book_publishers"
               ON
               ("django_orms_person"."id" = "django_orms_book_publishers"."person_id")
               WHERE "django_orms_book_publishers"."book_id" = 1
               LIMIT 21; args=(1,)
<QuerySet [{'id': 2, 'name': 'Person2'}, {'id': 3, 'name': 'Person3'}]>

(0.000) SELECT "django_orms_person"."id", "django_orms_person"."name"
               FROM "django_orms_person"
               INNER JOIN "django_orms_book_publishers"
               ON
               ("django_orms_person"."id" = "django_orms_book_publishers"."person_id")
               WHERE "django_orms_book_publishers"."book_id" = 2  LIMIT 21; args=(2,)
<QuerySet [{'id': 1, 'name': 'Person1'}, {'id': 3, 'name': 'Person3'}]>
```

Количество запросов увеличилось до 4. Это связано с использованием метода `values()` вместо `all()`. На каждой итерации выполняется 1 запрос для получения издателей.

По умолчанию предварительная выборка объединяет все результаты, но здесь мы использовали `values('id', 'name')`, и из-за этого Django не объединяет нужные нам результаты.

```python
from django.db.models import Prefetch

def get_all_books():
    books = Book.objects.prefetch_related(
        Prefetch(
            'publishers',
            queryset=Person.objects.only('id', 'name'),
            to_attr='all_publishers'
        ))
    for book in books:
        print(book.all_publishers)
```

Я добавила **Prefetch**, чтобы установить для нас новые атрибуты. Мы извлекаем значения из поля издателей, используя `Person.objects.only('id','name')` в качестве базового набора запросов и сообщаем Django, что нам нужны все предварительно выбранные значения в атрибуте **all\_publishers**.

Давайте проверим вывод.

```python
>>> get_all_books()
(0.000) SELECT "django_orms_book"."id", "django_orms_book"."name",
               "django_orms_book"."author_id" FROM "django_orms_book"; args=()
(0.001) SELECT ("django_orms_book_publishers"."book_id")
               AS "_prefetch_related_val_book_id", "django_orms_person"."id",
               "django_orms_person"."name"
               FROM "django_orms_person"
               INNER JOIN "django_orms_book_publishers"
               ON
               ("django_orms_person"."id" = "django_orms_book_publishers"."person_id")
               WHERE "django_orms_book_publishers"."book_id" IN (1, 2); args=(1, 2)
[<Person: Person object (2)>, <Person: Person object (3)>]
[<Person: Person object (1)>, <Person: Person object (3)>]
```

YESS 😃. Проблема решается с помощью **Prefetch**.

Мы также можем сделать то же самое без атрибута **to\_attr** **Prefetch**. Ниже приведена функция без **to\_attr**.

```python
from django.db.models import Prefetch

def get_all_books():
    books = Book.objects.prefetch_related(
        Prefetch(
            'publishers',
            queryset=Person.objects.only('id', 'name'),
        ))
    for book in books:
        print(book.publishers.all())
```

Это дает тот же результат, что и выше.

Я надеюсь, вы узнали немного больше о предварительной выборке и связанном выборе.

Спасибо, что прочитали эту статью. Если вам это нравится, нажмите на 👏, чтобы оценить его из 50, а также поделиться им с друзьями. Это многое значит для меня.
