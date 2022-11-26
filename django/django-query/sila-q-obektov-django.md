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
