# Советы по работе с базами данных

{% hint style="info" %}
Ссылка на оригинальную статью: [Django Tips for Working with Databases](https://www.designmycodes.com/python/django-tips-for-working-with-databases.html)

Опубликовано:&#x20;

Автор:&#x20;
{% endhint %}

ORM предлагают большую полезность для разработчиков, но абстрагирование доступа к базе данных имеет свои издержки. Разработчики, готовые покопаться в базе данных и изменить некоторые значения по умолчанию, часто обнаруживают, что можно добиться значительных улучшений.

В этой статье я поделюсь 9 советами по работе с базами данных в Django.

## Агрегация с фильтром

До Django 2.0, если мы хотели получить что-то вроде общего количества пользователей и общего количества активных пользователей, нам приходилось прибегать к условным выражениям:

```python
from django.contrib.auth.models import User
from django.db.models import (
    Count,
    Sum,
    Case,
    When,
    Value,
    IntegerField,
)

User.objects.aggregate(
    total_users=Count('id'),
    total_active_users=Sum(Case(
        When(is_active=True, then=Value(1)),
        default=Value(0),
        output_field=IntegerField(),
    )),
)
```

В Django 2.0 был добавлен аргумент **filter** для агрегирующих функций, чтобы сделать это намного проще:

```python
from django.contrib.auth.models import User
from django.db.models import Count, F

User.objects.aggregate(
    total_users=Count('id'),
    total_active_users=Count('id', filter=F('is_active')),
)
```

Красиво, коротко и мило.

Если вы используете PostgreSQL, два запроса будут выглядеть так:

```sql
SELECT
    COUNT(id) AS total_users,
    SUM(CASE WHEN is_active THEN 1 ELSE 0 END) AS total_active_users
FROM
    auth_users;
SELECT
    COUNT(id) AS total_users,
    COUNT(id) FILTER (WHERE is_active) AS total_active_users
FROM
    auth_users;
```

Второй запрос использует предложение `FILTER (WHERE…)`.

## Результаты QuerySet в виде именованных кортежей

В Django 2.0 в **values\_list** был добавлен новый атрибут с именем **named**. Установка **named** в **true** вернет набор запросов в виде списка **namedtuples**:

```python
> user.objects.values_list(
    'first_name',
    'last_name',
)[0]
(‘Haki’, ‘Benita’)

> user_names = User.objects.values_list(
    'first_name',
    'last_name',
    named=True,
)

> user_names[0]
Row(first_name='Haki', last_name='Benita')

> user_names[0].first_name
'Haki'

> user_names[0].last_name
'Benita'
```

## Пользовательские функции

Django ORM очень мощный и многофункциональный, но он не может идти в ногу со всеми поставщиками баз данных. К счастью, ORM позволяет нам расширить его с помощью пользовательских функций.

Скажем, у нас есть модель отчета с полем продолжительности. Мы хотим найти среднюю продолжительность всех отчетов:

```python
from django.db.models import Avg

Report.objects.aggregate(avg_duration=Avg(‘duration’))
> {'avg_duration': datetime.timedelta(0, 0, 55432)}
```

Это здорово, но само по себе среднее значение говорит нам очень мало. Попробуем также получить стандартное отклонение:

```python
from django.db.models import Avg, StdDev

Report.objects.aggregate(
    avg_duration=Avg('duration'),
    std_duration=StdDev('duration'),
)

ProgrammingError: function stddev_pop(interval) does not exist
LINE 1: SELECT STDDEV_POP("report"."duration") AS "std_dura...
               ^
HINT:  No function matches the given name and argument types. You might need to add explicit type casts.
```

Упс… PostgreSQL не поддерживает **stddev** в поле интервала **duration**  — нам нужно преобразовать интервал в число, прежде чем мы сможем применить к нему **STDDEV\_POP**.

Один из вариантов - извлечь эпоху из продолжительности:

```sql
SELECT
    AVG(duration),
    STDDEV_POP(EXTRACT(EPOCH FROM duration))
FROM 
    report;
      avg       |    stddev_pop    
----------------+------------------
 00:00:00.55432 | 1.06310113695549
(1 row)
```

Итак, как мы можем реализовать это в Django? Как вы уже догадались  — пользовательская функция:

```python
# common/db.py
from django.db.models import Func

class Epoch(Func):
   function = 'EXTRACT'
   template = "%(function)s('epoch' from %(expressions)s)"
```

И используйте нашу новую функцию следующим образом:

```python
from django.db.models import Avg, StdDev, F
from common.db import Epoch

Report.objects.aggregate(
    avg_duration=Avg('duration'), 
    std_duration=StdDev(Epoch(F('duration'))),
)

{'avg_duration': datetime.timedelta(0, 0, 55432), 'std_duration': 1.06310113695549}
```

Обратите внимание на использование выражения **F** в вызове **Epoch**.

## Тайм-аут заявления

Это, наверное, самый простой и самый важный совет, который я могу дать. Все мы люди и делаем ошибки. Мы не можем справиться с каждым пограничным случаем, поэтому мы должны установить границы.

В отличие от других неблокирующих серверов приложений, таких как Tornado, asyncio или даже Node, Django обычно использует синхронные рабочие процессы. Это означает, что когда пользователь выполняет длительную операцию, рабочий процесс блокируется, и никто другой не может его использовать, пока он не будет выполнен.

Я уверен, что на самом деле никто не запускает Django в продакшене только с одним рабочим процессом, но мы все же хотим убедиться, что один запрос не занимает слишком много ресурсов слишком долго.

В большинстве приложений Django большую часть времени тратится на ожидание запросов к базе данных. Итак, установка тайм-аута для SQL-запросов — хорошее место для начала.

Нам нравится устанавливать глобальное время ожидания в нашем файле `wsgi.py` следующим образом:

```python
# wsgi.py
from django.db.backends.signals import connection_created
from django.dispatch import receiver

@receiver(connection_created)
def setup_postgres(connection, **kwargs):
    if connection.vendor != 'postgresql':
        return
    
    # Операторы тайм-аута через 30 секунд.
    with connection.cursor() as cursor:
        cursor.execute("""
            SET statement_timeout TO 30000;
        """)
```

Почему `wsgi.py`? Таким образом, он влияет только на рабочие процессы, а не на внеполосные аналитические запросы, задачи cron и т. д.

Надеюсь, вы используете постоянные соединения с базой данных, поэтому такая настройка для каждого соединения не должна увеличивать нагрузку на каждый запрос.

Тайм-аут также можно установить на уровне пользователя:

```bash
postgresql=#> alter user app_user set statement_timeout TO 30000;
ALTER ROLE
```

ПРИМЕЧАНИЕ: Другое общее место, на которое мы потратили много времени, — это общение в сети. Поэтому убедитесь, что при вызове удаленной службы всегда устанавливается тайм-аут:

```python
import requests

response = requests.get(
    'https://api.slow-as-hell.com',
    timeout=3000,
)
```

## LIMIT

Это в некоторой степени связано с последним пунктом об установлении границ. Иногда мы хотим, чтобы пользователи создавали отчеты и, возможно, экспортировали их в электронную таблицу. Эти типы представлений обычно являются непосредственными подозреваемыми в любом странном поведении в рабочей среде.

Нередко можно встретить пользователя, который считает разумным экспортировать все продажи с незапамятных времен в середине рабочего дня. Также нередко этот же пользователь открывает другую вкладку и пытается снова, когда первая попытка «зависла».

Вот тут-то и появляется **LIMIT**.

Давайте ограничим определенный запрос не более чем 100 строками:

```python
# плохой пример
data = list(Sale.objects.all())[:100]
```

Это худшее, что вы можете сделать. Вы только что извлекли в память все миллионы строк, чтобы вернуть первые 100.

Давай еще раз попробуем:

```python
data = Sale.objects.all()[:100]
```

Это лучше. Django будет использовать предложение **limit** в SQL, чтобы получить только 100 строк.

Теперь допустим мы добавили лимит, пользователи под контролем и все хорошо. У нас осталась одна проблема  — пользователь запросил все продажи, а мы дали ему 100. Теперь пользователь думает, что продаж всего 100  — это неправильно.

Вместо того, чтобы слепо возвращать первые 100 строк, давайте удостоверимся, что если строк больше 100 (обычно после фильтрации), мы выдаем исключение:

```python
LIMIT = 100
if Sales.objects.count() > LIMIT:
    raise ExceededLimit(LIMIT)
return Sale.objects.all()[:LIMIT]
```

Это будет работать, но мы только что добавили еще один запрос.

Можем ли мы сделать лучше? Я думаю, мы можем:

```python
LIMIT = 100
data = Sale.objects.all()[:(LIMIT + 1)]
if len(data) > LIMIT:
    raise ExceededLimit(LIMIT)
return data
```

Вместо выборки 100 строк мы получаем 100 + 1 = 101 строку. Если 101 строка существует, нам достаточно знать, что существует более 100 строк. Или, другими словами, выборка LIMIT + 1 строк — это меньшее, что нам нужно, чтобы убедиться, что в результате запроса не более LIMIT строк.

Помните о трюке LIMIT + 1, иногда он может пригодиться.

## SELECT for UPDATE … of

Этому мы научились на собственном горьком опыте. Среди ночи мы начали получать ошибки о превышении времени ожидания транзакций из-за блокировок в базе данных.

Общий шаблон для манипулирования транзакцией в нашем коде будет выглядеть так:

```python
from django.db import transaction as db_transaction
...
with db_transaction.atomic():
  transaction = (
        Transaction.objects
        .select_related(
            'user',
            'product',
            'product__category',
        )
        .select_for_update()
        .get(uid=uid)
  )
    ...
```

Управление транзакцией обычно включает в себя некоторые свойства пользователя и продукта, поэтому мы часто используем **select\_related** для принудительного объединения и сохранения некоторых запросов.

Обновление транзакции также включает в себя получение блокировки, чтобы убедиться, что ею никто не манипулирует.

Теперь вы видите проблему? НЕТ? Мы тоже.

У нас были некоторые процессы ETL, работающие по ночам, выполняющие обслуживание таблиц продуктов и пользователей. Эти ETL выполняли обновления и вставки в таблицы, поэтому они также получали блокировки для таблиц.

Так в чем была проблема? Когда **select\_for\_update** используется вместе с **select\_related**, Django попытается заблокировать все таблицы в запросе.

Код, который мы использовали для получения транзакции, пытался заблокировать как таблицу транзакций, так и таблицы пользователей, продуктов и категорий. Как только ETL заблокировал последние три таблицы посреди ночи, транзакции начали терпеть неудачу.

Как только мы лучше поняли проблему, мы начали искать способы блокировки только необходимой таблицы  — таблицы транзакций. К счастью, новая опция **select\_for\_update** только что стала доступна в Django 2.0:

```python
from django.db import transaction as db_transaction
...
with db_transaction.atomic():
  transaction = (
        Transaction.objects
        .select_related(
            'user',
            'product',
            'product__category',
        )
        .select_for_update(
            of=('self',)
        )
        .get(uid=uid)
  )
  ...
```

Параметр **of** был добавлен в **select\_for\_update**. С помощью **of** мы можем явно указать, какие таблицы мы хотим заблокировать. **self** — это специальное ключевое слово, указывающее, что мы хотим заблокировать модель, над которой работаем, в данном случае — **Transaction**.

В настоящее время эта функция доступна только для серверных частей PostgreSQL и Oracle.
