# Использование пользовательских классов для полей модели

{% hint style="info" %}
Ссылка на оригинальную статью: [Django: using custom classes for model fields](https://medium.com/@luccascorrea/django-using-custom-classes-for-model-fields-38e58914ba5c)

Опубликовано: 27 июня 2022

Автор: [Luccas Correa](https://medium.com/@luccascorrea?source=post\_page-----38e58914ba5c--------------------------------)
{% endhint %}

Django поставляется с множеством различных типов полей модели (**CharField**, **UUIDField**, **FileField** и т. д.), но бывают случаи, когда вам нужно что-то другое.

Эта потребность возникала у меня несколько раз, когда я сохранял данные с помощью **JSONField**. Эти типы полей отлично подходят для хранения метаданных и тому подобного. По умолчанию данные, хранящиеся в **JSONFields**, декодируются в python либо в виде списка, либо в виде словаря. Это хорошо, но иногда было бы безопаснее, если бы вы могли использовать экземпляры своего собственного класса, так как же мы это делаем?

В качестве примера предположим, что у вас есть **JSONField** в вашей модели, где вы хотите сохранить адрес.

```python
from django.db import models

class MyModel(models.Model):
    ...
    address = models.JSONField()
```

Этого было бы достаточно для большинства нужд, но основная проблема, с которой вы можете столкнуться при использовании такого **JSONField**, заключается в том, что вы теряете вводимые данные. Поскольку данные будут обрабатываться как простой словарь Python, вы можете, например, ошибиться в написании атрибута вашего адреса, и все будет работать без каких-либо исключений, и вы поймете проблему только позже, когда вам понадобится доступ к этой информации.

Лучшим способом справиться с этим было бы, если бы наше поле могло хранить данные JSON как экземпляры нашего собственного пользовательского класса. Класс, подобный приведенному ниже, подойдет:

```python
class Address:
    def __init__(self, street, number, neighborhood, city, state, zip_code):
        self.street = street
        self.number = number
        self.neighborhood = neighborhood
        self.city = city
        self.state = state
        self.zip_code = zip_code
    
    def to_dict(self):
        return {
          "street": self.street, 
          "number": self.number, 
          "neighborhood": self.neighborhood,
          "city": self.city,
          "state": self.state,
          "zip_code": self.zip_code,
        }
```

Чтобы сделать это возможным, первое, что нужно сделать, это создать подкласс **JSONField**. В этом классе мы проинструктируем Django о том, как преобразовать данные JSON в экземпляры нашего класса и наоборот:

```python
from django.db.models import JSONField
from address import Address

class AddressField(JSONField):

    def from_db_value(self, value, expression, connection):
        db_val = super().from_db_value(value, expression, connection)

        if db_val is None:
            return db_val
        
        return Address(**db_val)
    
    def get_prep_value(self, value):
        dict_value = value.to_dict()
        prep_value = super().get_prep_value(dict_value)
        return prep_value
```

Приведенный выше метод **from\_db\_value** сообщает Django, как декодировать значение базы данных в экземпляр нашего класса. Сначала мы вызываем суперреализацию **JSONField**, которая просто декодирует строку JSON в словарь Python. Используя этот словарь, мы можем просто создать экземпляр нашего пользовательского класса.

Метод **get\_prep\_value** делает обратное. Сначала он преобразует наш класс в словарь Python, а затем вызывает суперреализацию **JSONField**, которая просто закодирует словарь в виде строки.

Хорошо, мы почти у цели. Одна вещь, которую вы можете заметить, заключается в том, что приведенный выше код не мешает кому-то просто назначить словарь нашему пользовательскому полю. Это может привести к проблемам в будущем, поскольку вы ожидаете получить доступ к свойствам адреса как к атрибутам, а не как к ключам в словаре:

```python
my_instance.address = {
  "street": "Wall St",
  "number": 1,
  "city": "New York",
  "state":"NY",
  "zip_code":"10000",
}

# Позже приведенный ниже вызов завершится ошибкой,
# поскольку my_instance.address является словарем.
print(my_instance.address.street)
```

Давайте исправим это, изменив наш настраиваемый класс поля:

```python
from django.db.models import JSONField
from address import Address

class AddressCreator:
  
    def __init__(self, field):
        self.field = field
  
    def __get__(self, obj):
        if obj is None:
            return self
      
        return obj.__dict__(self.field.name)
  
    def __set__(self, obj, value):
        obj.__dict__[self.field.name] = self.convert_input(value)

    def convert_input(self, value):
        if value is None:
            return None
        
        if isinstance(value, Address):
            return value
        else:
            return Address(**value)

class AddressField(JSONField):

    def from_db_value(self, value, expression, connection):
        db_val = super().from_db_value(value, expression, connection)

        if db_val is None:
            return db_val
        
        return Address(**db_val)
    
    def get_prep_value(self, value):
        dict_value = value.to_dict()
        prep_value = super().get_prep_value(dict_value)
        return prep_value
    
    def contribute_to_class(self, cls, name, private_only=False):
        super().contribute_to_class(cls, name, private_only=private_only)
        setattr(cls, self.name, AddressCreator(self))
```

Итак, давайте проверим внесенные изменения. На самом деле мы начнем с нижней части файла. Вы увидите, что мы реализовали метод **contribute\_to\_class** в нашем классе **AddressField**. Этот метод позволит нам использовать другой класс ( **AddressCreator** ) для перехвата присвоений атрибуту адреса экземпляра нашей модели.

Затем класс **AddressCreator** будет реализовывать два метода дескриптора `__get__` и `__set__`, которые вызываются всякий раз, когда происходит доступ к атрибуту нашего экземпляра и его назначение соответственно. Наш метод `__get__` мало что делает и просто возвращает любое значение, которое было присвоено полю. Наш метод `__set__`, с другой стороны, проверит, является ли присваиваемое значение уже экземпляром нашего класса **Address** (в этом случае оно будет просто присвоено), и если нет, он предположит, что значение является словарем, и создаст экземпляр нашего класса **Address** вместо этого. Таким образом, даже если полю адреса нашей модели назначен словарь, мы можем быть уверены, что вместо него будет использоваться экземпляр нашего класса **Address**:

```python
my_instance.address = {
  "street": "Wall St", 
  "number": 1, 
  "city": "New York", 
  "state":"NY", 
  "zip_code":"10000",
}

# Приведенный ниже вызов на этот раз не завершится ошибкой,
# так как my_instance.address является экземпляром нашего класса.
print(my_instance.address.street)
```

Хорошо, мы все сделали. Последнее, что нужно сделать, это просто использовать наше недавно созданное пользовательское поле модели:

```python
from django.db import models
from fields import AddressField

class MyModel(models.Model):
    ...
    address = AddressField()
```

## Заключение

Использование пользовательских классов для полей модели (особенно **JSONFields**) — хороший способ гарантировать, что данные соответствуют определенному формату. Поля **JSONField** хороши своей гибкостью, но та же самая гибкость может затруднить понимание того, какие данные они хранят, просто взглянув на определение модели.

Это немного более сложная тема, которая возникла в проекте, над которым я недавно работал, поэтому я подумал, что напишу немного об этом на случай, если это может быть полезно кому-то еще. Я надеюсь, что это было полезно для вас!
