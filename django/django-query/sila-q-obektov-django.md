# Сила Q-объектов Django

{% hint style="info" %}
Ссылка на оригинальную статью: [The power of django's Q objects](https://www.michelepasin.org/blog/2010/07/20/the-power-of-djangos-q-objects/)

Опубликовано: 20 июля 2010

Автор: [Michele Pasin](https://www.michelepasin.org/contact.html)
{% endhint %}

Сегодня я провел некоторое время, изучая, как лучше всего использовать [объекты Q django](https://docs.djangoproject.com/en/3.2/topics/db/queries/#complex-lookups-with-q-objects). Затем я провел небольшое тестирование и решил собрать все это воедино на будущее, понимаете, на тот момент, когда я потеряюсь и мне понадобится какой-нибудь рецепт кодирования.

Перво-наперво: **официальная документация**.

* Документация Django: [Комплексный поиск с объектами Q](http://docs.djangoproject.com/en/dev/topics/db/queries/#complex-lookups-with-q-objects). Основное введение.
* Документация Django: [Поиск по OR](http://www.djangoproject.com/documentation/models/or\_lookups/). Более подробные примеры.

Теперь... давайте начнем с воскрешения простой модели, использованной в [учебнике по django](http://docs.djangoproject.com/en/dev/intro/tutorial01/), **Polls**, и добавления нескольких **экземпляров** для пробы:

```python
# ==> в models.py
from django.db import models
import datetime

class Poll(models.Model):
        question = models.CharField(max_length=200)
        pub_date = models.DateTimeField('date published')

# ==> добавить несколько экземпляров из оболочки
>>> p = Poll(
        question='what shall I make for dinner', pub_date=datetime.date.today()
    )
>>> p.save()
>>> p = Poll(
        question='what is your favourite meal?',
        pub_date=datetime.datetime(2009, 7, 19, 0, 0)
    ) # год назад
>>> p.save()
>>> p = Poll(question='how do you make omelettes?', pub_date=datetime.date.today())
>>> p.save()
```

## Для чего нужны удобные объекты Q?

Короче говоря, основная причина, по которой вы можете захотеть их использовать, заключается в том, что вам нужно выполнять **сложные запросы**, например, с использованием операторов **OR** и **AND** одновременно.

Представьте, что вы хотите получить все опросы, вопрос которых **содержит слово `'dinner' OR 'meal'`**. Обычная опция **filter** позволяет вам сделать следующее:

```python
Poll.objects.filter(question__contains='dinner').filter(question__contains='meal')
```

Но это **не совсем то, чего мы хотим**, не так ли? Мы только что получили конкатенацию операторов **AND** (= все ограничения должны быть выполнены), в то время как мы хотели **OR** (= хотя бы одно из ограничений должно быть выполнено). Вот где объекты Q становятся полезными:

```python
>>> from django.db.models import Q

# простой запрос
>>> Poll.objects.filter(Q(question__contains='dinner'))
[<Poll: what shall I make **for** dinner>]

# два ограничения, связанные AND
>>> Poll.objects.filter(Q(question__contains='dinner') & Q(question__contains='meal'))
[]

# два ограничения, связанные OR - вот и все!
>>> Poll.objects.filter(Q(question__contains='dinner') | Q(question__contains='meal'))
[<Poll: what shall I make **for** dinner>, <Poll: what **is** your favourite meal?>]
```

Обратите внимание, что если вы не укажете никакого логического соединителя, последовательность Q неявно интерпретируется как **AND**:

```python
# нет логического соединителя - интерпретируется как AND
>>> Poll.objects.filter(Q(question__contains='dinner'), Q(question__contains='meal'))
[]
```

Все становится интереснее при создании сложных запросов:

```python
# например, (A OR B) AND C:
>>> Poll.objects.filter(
    (Q(question__contains='dinner') | Q(question__contains='meal')) &\
    Q(pub_date=datetime.date.today())
)
[<Poll: what shall I make **for** dinner>]
```

## Динамическое построение запросов

Теперь вполне вероятно, что вы хотите динамически создавать запросы, подобные приведенным выше. Это еще одно место, где объекты Q могут сэкономить много времени... например, при создании **поисковых систем** или **фасетных браузеров**, где интерфейс позволяет пользователю постепенно **накапливать поисковые фильтры**.

Один из способов справиться с этой ситуацией — создать **списки объектов Q**, а затем **объединить их вместе** с помощью python методов [operator](http://docs.python.org/library/operator.html#operator.and\_) и [reduce](http://docs.python.org/library/functions.html#reduce):

```python
>>> import operator

# создать список объектов Q
>>> mylist = [Q(question__contains='dinner'), Q(question__contains='meal')]

# OR
>>> Poll.objects.filter(reduce(operator.or_, mylist))
[<Poll: what shall I make **for** dinner>, <Poll: what **is** your favourite meal?>]

# AND
>>> Poll.objects.filter(reduce(operator.and_, mylist))
[]
```

Теперь, если вы создаете запрос динамически, вы, вероятно, не будете знать заранее, какие фильтры вам нужно использовать. Скорее всего, вместо этого вам придется **программно генерировать список объектов Q** из списка строк, представляющих запросы к вашим моделям:

```python
# строковое представление наших запросов
>>> predicates = [('question__contains', 'dinner'), ('question__contains', 'meal')]

# создайте список объектов Q и выполните запросы, как указано выше...
>>> q_list = [Q(x) **for** x **in** predicates]
>>> Poll.objects.filter(reduce(operator.or_, q_list))
[<Poll: what shall I make **for** dinner>, <Poll: what **is** your favourite meal?>]
>>> Poll.objects.filter(reduce(operator.and_, q_list))
[]

# теперь давайте добавим еще один фильтр к строкам запроса..
>>> predicates.append(('pub_date', datetime.date.today()))
>>> predicates
[('question__contains', 'dinner'), ('question__contains', 'meal'), ('pub_date', datetime.date(2010, 7, 19))]

# .. и результаты тоже изменятся
>>> q_list = [Q(x) **for** x **in** predicates]
>>> Poll.objects.filter(reduce(operator.or_, q_list))
[<Poll: what shall I make **for** dinner>, <Poll: what **is** your favourite meal?>, <Poll: how do you make omelettes?>]
>>> Poll.objects.filter(reduce(operator.and_, q_list))
[]
```

## Расширение запроса и объекты Q

Используя синтаксис расширения запроса, вы обычно можете делать такие вещи, **как это**:

```python
>>> mydict = {'question__contains': 'omelette', 'pub_date' : datetime.date.today()}
>>> Poll.objects.filter(**mydict)
[<Poll: how do you make omelettes?>]
```

Здесь вы в основном **упорядочиваете операторы AND, которые создаются динамически** из некоторых строковых значений.

Действительно крутая особенность django ORM заключается в том, что вы можете **продолжать делать** это даже при **использовании объектов Q**. Другими словами, вы можете делегировать все «обычные» вещи механизму расширения запроса, вместо этого прибегая к объектам Q всякий раз, когда вам нужны более сложные запросы, такие как оператор **OR**.

Например (не забывайте ставить словарь всегда на вторую позицию):\*\*\*\*

```python
# OR плюс расширение запроса
>>> Poll.objects.filter(reduce(operator.or_, q_list), **mydict)
[<Poll: how do you make omelettes?>]
# AND плюс расширение запроса
>>> Poll.objects.filter(reduce(operator.and_, q_list), **mydict)
[]
```

В **первом** случае у нас есть один результат только потому, что, хотя ограничение **OR** для **q\_list** само по себе соответствовало бы большему количеству вещей, _**mydict**_** разбивается на ряд ограничений, связанных с AND, что делает \[**_**Poll: how do you make omelettes?**_**] единственным подходящим объектом**. Во **втором**\*\* случае нет вообще никаких результатов просто потому, что мы запрашиваем экземпляр опроса, который одновременно удовлетворяет всем ограничениям (которых не существует)!

## Другие ресурсы

Это все на данный момент! Как я уже сказал, я нашел **довольно много очень хороших постов** на эту тему в Интернете, и я настоятельно рекомендую вам **ознакомиться с ними**, чтобы получить более четкое представление о том, как работают Q-объекты.

* [Добавление объектов Q в Django](http://bradmontgomery.blogspot.com/2009/06/adding-q-objects-in-django.html). Показывает другой метод объединения объектов Q с помощью `'add'`.
* [Совет по сложным запросам Django](http://jehiah.cz/archive/django-q-objects). Обсуждается, как должны быть написаны запросы, чтобы создать ожидаемый код MySQL.
* [Сила Q](http://www.djangozen.com/blog/the-power-of-q). Действительно хорошая статья, в которой рассматриваются все основные особенности объектов Q (и, или, отрицание, выбор)
* [Динамические запросы Django (или почему kwargs — ваш друг)](http://www.nomadjourney.com/2009/04/dynamic-django-queries-with-kwargs/). Обзор динамических запросов с помощью django
