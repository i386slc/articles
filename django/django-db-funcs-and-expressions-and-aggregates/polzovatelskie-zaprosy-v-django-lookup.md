# Пользовательские запросы в Django (lookup)

{% hint style="info" %}
Ссылка на оригинальную статью: [Custom Lookups in Django](https://medium.com/nerd-for-tech/custom-lookups-in-django-69fd13e35bdb)

Опубликовано: 14 апреля 2021

Автор: [Nishanth G](https://medium.com/@nishanth-g?source=post\_page-----69fd13e35bdb--------------------------------)
{% endhint %}

Прежде чем мы проверим пользовательские запросы, давайте посмотрим на основы и поймем, что такое поиск.

## Итак, что такое поиск…

Согласно документации Django

> Поиск по полям — это то, как вы определяете суть предложения SQL **WHERE**. Они указываются в качестве аргументов ключевых слов для методов [filter()](https://docs.djangoproject.com/en/3.1/ref/models/querysets/#django.db.models.query.QuerySet.filter), [exclude()](https://docs.djangoproject.com/en/3.1/ref/models/querysets/#django.db.models.query.QuerySet.exclude) и [get()](https://docs.djangoproject.com/en/3.1/ref/models/querysets/#django.db.models.query.QuerySet.get) **QuerySet**.

не очень понятно, но пример сделал бы это намного проще

```python
Entry.objects.filter(id__in=[1, 3, 4])
```

В приведенном выше примере `"in"` — это то, что называется поиском **Lookup**, и приведенный выше запрос фильтрует все данные **Entry**, идентификаторы **id** которых принадлежат любому из значений в списке `[1, 3, 4]`

Теперь я надеюсь, что вы имеете некоторое представление о том, как и где используется поиск **Lookup**.

В Django имеется множество встроенных поисковых запросов, и если ни один из них не указан как таковой, то по умолчанию используется `'exact'`, например:

```python
Entry.objects.get(id=14)
```

то же самое, что

```python
Entry.objects.get(id__exact=14)
```

Несколько примеров встроенных поисков: \[`in`, `iexact`, `contains`, `gt`, `gte`, `lt`, `startswith`….]

Вы можете проверить ссылку на остальные: [https://docs.djangoproject.com/en/3.1/ref/models/querysets/#field-lookups](https://docs.djangoproject.com/en/3.1/ref/models/querysets/#field-lookups)

## Давайте прыгнем и обратимся к 🐘 в 🏠 — — — — Custom 👀👆🏻

Хаха, извини, я увлекся смайликами,

## Так зачем нам нужны пользовательские запросы?

Почему, ответ прост, потому что может не быть никакого поиска, который мог бы получить данные в соответствии с вашими сложными требованиями.

Давайте проверим очень простой пример из Djangodocs, который даст вам достаточно информации, чтобы начать работу над пользовательскими поисками.

Мы напишем собственный поиск, который будет работать противоположно **exact**. `Author.objects.filter(name__ne='Jack')` переведет в SQL:

```sql
"author"."name" <> 'Jack'
```

Этот SQL не зависит от серверной части, поэтому нам не нужно беспокоиться о разных базах данных.

Есть два шага, чтобы сделать эту работу. Сначала нам нужно реализовать поиск, а затем сообщить об этом Django:

```python
from django.db.models import Lookup

class NotEqual(Lookup):
    lookup_name = 'ne'
    
    def as_sql(self, compiler, connection):
        lhs, lhs_params = self.process_lhs(compiler, connection)
        rhs, rhs_params = self.process_rhs(compiler, connection)
        params = lhs_params + rhs_params
        return '%s <> %s' % (lhs, rhs), params
```

Чтобы зарегистрировать поиск **NotEqual**, нам нужно будет вызвать **register\_lookup** для класса поля, для которого мы хотим, чтобы поиск был доступен. В этом случае поиск имеет смысл во всех подклассах **Field**, поэтому мы регистрируем его напрямую в **Field**:

```python
from django.db.models import Field

Field.register_lookup(NotEqual)
```

Регистрация поиска также может быть выполнена с использованием шаблона декоратора:

```python
from django.db.models import Field

@Field.register_lookup
class NotEqualLookup(Lookup):
    ...
```

для получения дополнительной информации проверьте: [https://docs.djangoproject.com/en/3.1/howto/custom-lookups/](https://docs.djangoproject.com/en/3.1/howto/custom-lookups/)

На этом все, ребята, и как всегда, если вы дочитали до сих пор, то…..
