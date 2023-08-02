# Написание пользовательских функций БД

{% hint style="info" %}
Ссылка на оригинальную статью: [Writing Custom Django Database Functions](https://dev.to/idrisrampurawala/writing-custom-django-database-functions-4dmb)

Опубликовано: 19 мая 2022

Автор: [Idris Rampurawala](https://dev.to/idrisrampurawala)
{% endhint %}

Функции базы данных Django представляют собой функции, которые будут _**выполняться в базе данных**_. Он предоставляет пользователям возможность использовать функции, предоставляемые базовой базой данных, в качестве аннотаций, агрегаций или фильтров. Функции также являются выражениями ([expressions](https://docs.djangoproject.com/en/4.0/ref/models/expressions/)), поэтому их можно использовать и комбинировать с другими выражениями, такими как агрегатные функции ([aggregate functions](https://docs.djangoproject.com/en/4.0/ref/models/querysets/#aggregation-functions)).

Одна из богатых функций Django включает настройку различных функций. Да, вы правильно поняли! Мы можем настроить функции базы данных Django в соответствии с нашими потребностями.

В этом посте мы рассмотрим пару примеров ниже, чтобы понять суть написания пользовательских функций, основанных на наших бизнес-потребностях.

Давайте сначала разберемся с классом Django `Func()`, который служит основой для нашего продвижения вперед.

## Класс Django `Func(*expressions, **extra)`

* Класс [Func()](https://docs.djangoproject.com/en/4.0/ref/models/expressions/#django.db.models.Func) является наиболее общей частью выражений запросов Django.
* Он позволяет каким-либо образом реализовать практически любую функцию или оператор в **Django ORM**.
* Выражение [Func() expression](https://docs.djangoproject.com/en/4.0/ref/models/expressions/#django.db.models.Func) является базовым типом всех выражений, которые включают функции базы данных, такие как **COALESCE** и **LOWER**, или агрегаты, такие как **SUM**.
* Я рекомендую прочитать [Избегание SQL-инъекций](https://docs.djangoproject.com/en/4.0/ref/models/expressions/#avoiding-sql-injection) перед использованием `Func()`

Ниже приведены некоторые способы написания наших пользовательских функций базы данных:

### Пользовательские функции базы данных

Мы можем создавать собственные функции базы данных, используя [класс Func](https://docs.djangoproject.com/en/4.0/ref/models/expressions/#django.db.models.Func) Django. В одном из моих проектов я хотел преобразовать временную метку **UTC** в **IST** в фильтре Django с определенным форматом даты. Написание двух простых функций базы данных Django помогло мне повторно использовать ее в нескольких случаях следующим образом:

```python
from django.db.models import Func

class TimestampToIST(Func):
    """Преобразует значение временной метки db (UTC) в эквивалентную
    временную метку IST."""

    function = 'timezone'
    template = "%(function)s('Asia/Calcutta', %(expressions)s)"


class TimestampToStr(Func):
    """Преобразует метку времени в строку, используя заданный формат"""

    function = 'to_char'
    # 21/06/2021 16:08:34
    template = "%(function)s(%(expressions)s, 'DD/MM/YYYY HH24:MI:SS')"
```

```python
# Использование
Author.objects.annotate(
    last_updated=TimestampToStr(TimestampToIST(F('updated_at')))
)
```

### Частичная реализация функций базы данных

Еще один отличный пример настройки — создать новую версию функции с одним или двумя уже заполненными аргументами. Например, давайте создадим специализированную **SubStr**, которая извлекает первый символ из строки:

```python
from functools import partial
from django.db.models.functions import Substr

ExtractFirstChar = partial(Substr, pos=1, length=1)
```

```python
# Использование
User.objects.annotate(name_initial=ExtractFirstChar('first_name'))
```

### Выполнение GROUP BY без функции агрегации

Представьте ситуацию, когда мы хотим использовать **GROUP BY** без использования какой-либо агрегатной функции. Django ORM не позволяет нам использовать **GROUP BY** без агрегатной функции. Следовательно, для этого мы можем создать функцию Django, которая обрабатывается Django как агрегатная функция, но оценивается как **NULL** в SQL-запросе, полученном из [StackOverflow](https://stackoverflow.com/a/65066965/9334209).

```python
from django.db.models import CharField, Func

class NullAgg(Func):
    """Аннотация, вызывающая GROUP BY без агрегирования.

    Фальшивый агрегатный класс Func, который можно использовать в аннотации,
    чтобы заставить запрос выполнять GROUP BY без выполнения агрегатной операции,
    требующей от сервера перечисления всех строк в каждой группе.

    Не принимает аргументов конструктора и возвращает значение NULL.

    Пример:
        ContentType.objects.values('app_label').annotate(na=NullAgg())
    """
    template = 'NULL'
    contains_aggregate = True
    window_compatible = False
    arity = 0
    output_field = CharField()
```

## Ресурсы

* [Django database functions](https://docs.djangoproject.com/en/4.0/ref/models/database-functions/)
* [Django Func() class](https://docs.djangoproject.com/en/4.0/ref/models/expressions/#django.db.models.Func)
* [StackOverflow answer GROUP BY without aggregate](https://stackoverflow.com/a/65066965/9334209)
