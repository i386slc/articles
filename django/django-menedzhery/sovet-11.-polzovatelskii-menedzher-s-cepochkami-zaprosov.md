# Совет #11. Пользовательский менеджер с цепочками запросов

{% hint style="info" %}
Ссылка на оригинальную статью: [Django Tips #11 Custom Manager With Chainable QuerySets](https://simpleisbetterthancomplex.com/tips/2016/08/16/django-tip-11-custom-manager-with-chainable-querysets.html)

Опубликовано: 16 августа 2016

Автор: [Vitor Freitas](https://simpleisbetterthancomplex.com/about/)
{% endhint %}

<figure><img src="../../.gitbook/assets/featured.jpg" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
Обновлено 17 августа 2016 г.: я обновил статью, в которой упоминается параметр **DocumentQuerySet.as\_manager()**. Спасибо Дэррилу и Марку за предложения в комментариях ниже!
{% endhint %}

В модели Django **Manager** — это интерфейс, взаимодействующий с базой данных. По умолчанию менеджер доступен через свойство **Model.objects**. Менеджером по умолчанию для каждой модели Django является `django.db.models.Manager`. Его очень просто расширить и изменить менеджер по умолчанию.

```python
from django.db import models

class DocumentManager(models.Manager):
    def pdfs(self):
        return self.filter(file_type='pdf')

    def smaller_than(self, size):
        return self.filter(size__lt=size)

class Document(models.Model):
    name = models.CharField(max_length=30)
    size = models.PositiveIntegerField(default=0)
    file_type = models.CharField(max_length=10, blank=True)

    objects = DocumentManager()
```

При этом вы сможете получить все файлы PDF следующим образом:

```python
Document.objects.pdfs()
```

Дело в том, что вы не можете использовать этот метод по цепочке. Я имею в виду, что вы все еще можете использовать **order\_by** или **filter** в результате:

```python
Document.objects.pdfs().order_by('name')
```

Но если вы попытаетесь связать методы, они сломаются:

```python
Document.objects.pdfs().smaller_than(1000)
```

```bash
AttributeError: 'QuerySet' object has no attribute 'smaller_than'
```

Чтобы заставить его работать, вы должны создать собственные методы **QuerySet**:

```python
class DocumentQuerySet(models.QuerySet):
    def pdfs(self):
        return self.filter(file_type='pdf')

    def smaller_than(self, size):
        return self.filter(size__lt=size)

class DocumentManager(models.Manager):
    def get_queryset(self):
        return DocumentQuerySet(self.model, using=self._db)  # Важно!

    def pdfs(self):
        return self.get_queryset().pdfs()

    def smaller_than(self, size):
        return self.get_queryset().smaller_than(size)

class Document(models.Model):
    name = models.CharField(max_length=30)
    size = models.PositiveIntegerField(default=0)
    file_type = models.CharField(max_length=10, blank=True)

    objects = DocumentManager()
```

Теперь вы можете использовать его так же, как и любой другой метод **QuerySet**:

```python
Document.objects.pdfs().smaller_than(1000).exclude(name='Article').order_by('name')
```

Если вы определяете только пользовательские наборы запросов в диспетчере, вы можете просто расширить **models.QuerySet** и в модели установить менеджер как `objects = DocumentQuerySet.as_manager()`:

```python
class DocumentQuerySet(models.QuerySet):
    def pdfs(self):
        return self.filter(file_type='pdf')

    def smaller_than(self, size):
        return self.filter(size__lt=size)

class Document(models.Model):
    name = models.CharField(max_length=30)
    size = models.PositiveIntegerField(default=0)
    file_type = models.CharField(max_length=10, blank=True)

    objects = DocumentQuerySet.as_manager()
```

Вы можете сохранить код внутри **models.py**. Но по мере роста базы кода я предпочитаю хранить менеджеры и наборы запросов **QuerySet** в другом модуле с именем **manager.py**.
