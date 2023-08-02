# Агрегация с django-filter через прокси-модели

{% hint style="info" %}
Ссылка на оригинальную статью: [Aggregation with django-filter via Proxy Models](https://www.aaronoellis.com/articles/aggregation-with-django-filter-via-proxy-models)

Опубликовано: 26 мая 2022

Автор: **Aaron O. Ellis**
{% endhint %}

Пакет [django-filter](https://django-filter.readthedocs.io/en/main/index.html) добавляет быстрый способ добавления фильтрации на основе параметров URL к существующим моделям баз данных Django. Например, возьмем следующую модель:

```python
from django.db import models

class Purchase(models.Model):
    item = models.TextField()
    amount = models.FloatField()
    at = models.DateTimeField()

    class Meta:
        db_table = "purchases"
```

Затем мы можем использовать **django-filter** для быстрого создания класса **FilterSet**, который обертывает приведенную выше модель базы данных:

```python
import django_filters
from .models import Purchase

class PurchaseFilter(django_filters.FilterSet):
    item = django_filters.CharFilter(lookup_expr="iexact")
    sort = django_filters.OrderingFilter(
        fields=(
            ("item", "item"),
            ("amount", "amount"),
            ("at", "at"),
        ),
    )

    class Meta:
        model = Purchase
        fields = ("amount", "at")
```

Вышеупомянутое затем может быть интегрировано в представления, такие как **ListView**:

```python
from django.views.generic.list import ListView
from .filters import PurchaseFilter

class Purchases(ListView):
    def get_queryset(self, **kwargs):
        return PurchaseFilter(self.request.GET).qs
```

Эта конфигурация позволяет параметрам **GET** управлять фильтрацией и порядком запросов к базе данных в представлениях с минимальным объемом связующего кода. Например, если приведенное выше представление **Purchases** находится по URL-адресу `/purchases`, запрос `/purchases?item=lunch&sort=amount` сгенерирует следующий SQL:

```sql
SELECT "purchases"."id", "purchases"."item", "purchases"."amount", "purchases"."at"
FROM "purchases"
WHERE "purchases"."item" LIKE 'lunch'
ORDER BY "purchases"."amount";
```

Набор фильтров **FilterSet** можно расширить, переопределив его свойство **qs**. Например, чтобы агрегировать результаты по элементам и суммировать их суммы:

```python
class AggregatedPurchaseFilter(django_filters.FilterSet):
    item = django_filters.CharFilter(lookup_expr="iexact")
    sort = django_filters.OrderingFilter(
        fields=(
            ("item", "item"),
            ("total", "total"),
        ),
    )

    @property
    def qs(self):
        return super().qs.values("item").annotate(total=models.Sum("amount"))

    class Meta:
        model = Purchase
        fields = ("item",)
```

Вышеупомянутое создает знакомый оператор **GROUP BY** в SQL:

```sql
SELECT "purchases"."item", SUM("purchases"."amount") AS "total"
FROM "purchases"
GROUP BY "purchases"."item";
```

Однако при попытке отсортировать фильтр по сумме с помощью параметра сортировки URL возвращается следующая ошибка:

```
FieldError
Cannot resolve keyword 'total' into field. Choices are: amount, at, id, item
```

## Порядок имеет значение

Эта ошибка возникает из-за планировщика запросов Django ORM, когда родительский класс **FilterSet** пытается [упорядочить](https://github.com/django/django/blob/73b4f3f9b3c2c64482c8a1174b3c5ab208d11279/django/db/models/sql/query.py#L2145) набор запросов. Короче говоря, планировщик запросов получает указание упорядочить поле, которое еще не создано! Мы можем воспроизвести ту же ошибку без **django-filter**, используя следующий запрос:

```python
Purchase.objects.order_by("total").values("item").annotate(total=models.Sum("amount"))
```

Разработчики, знакомые с SQL, могут счесть это очевидной ошибкой. _**Конечно порядок имеет значение!**_ SQL довольно строго относится к своему синтаксису, и размещение предложения **ORDER BY** перед **GROUP BY** приведет к синтаксической ошибке. Однако Django ORM абстрагирует построение запроса, и в наших примерах выше это построение распределено между несколькими классами и их подклассами.

Чтобы исправить нашу проблему с **FieldError**, нам нужно агрегировать набор запросов, прежде чем он будет передан в **FilterSet**. К счастью, **FilterSet** включает необязательный параметр **queryset**, который делает именно это:

```python
queryset = Purchase.objects.values("item").annotate(total=models.Sum("amount"))
PurchaseFilter(request.GET, queryset=queryset)
```

Который производит ожидаемый SQL без проблем:

```sql
SELECT "purchases"."item", SUM("purchases"."amount") AS "total"
FROM "purchases"
GROUP BY "purchases"."item"
ORDER BY "total";
```

## Прокси-модели

К сожалению, необходимость фильтровать набор запросов перед инициализацией **FilterSet** добавляет к нашему приложению значительный объем связующего кода и не позволяет нам писать представления с модульными фильтрами, которые могут быть указаны классом.

К счастью, мы также можем агрегировать наш набор запросов перед **FilterSet** с помощью [прокси-моделей](https://docs.djangoproject.com/en/4.0/topics/db/models/#proxy-models) и [пользовательских менеджеров](https://docs.djangoproject.com/en/4.0/topics/db/managers/#custom-managers). Мы можем добавить следующее вместе с нашим классом **Purchase**:

```python
class AggregatedPurchaseManager(models.Manager):
    def get_queryset(self):
        return super().get_queryset().values("item").annotate(total=models.Sum("amount"))


class AggregatedPurchase(Purchase):
    objects = AggregatedPurchaseManager()

    class Meta:
        proxy = True
```

Приведенная выше модель прокси позволяет нам изменить набор запросов по умолчанию без дополнительных таблиц базы данных. Нам также больше не нужно расширять набор запросов **FilterSet** или передавать ему отфильтрованный набор запросов во время инициализации. Нам нужно только изменить его свойство модели на наш прокси:

```python
class AggregatedPurchaseFilter(django_filters.FilterSet):
    item = django_filters.CharFilter(lookup_expr="iexact")
    sort = django_filters.OrderingFilter(
        fields=(
            ("item", "item"),
            ("total", "total"),
        ),
    )

    class Meta:
        model = AggregatedPurchase
        fields = ("item",)
```

Используя приведенный выше **FilterSet** в нашем представлении, мы теперь можем упорядочивать по агрегированному полю с помощью параметров URL, таких как `/purchases?sort=total`.
