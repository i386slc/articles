# Поле модели — работа Django ORM — часть 2

{% hint style="info" %}
Ссылка на оригинальную статью: [Model Field - Django ORM Working - Part 2](https://kracekumar.com/post/django\_model\_fields\_orm\_part2/)

Опубликовано:

Автор: Kracekumar
{% endhint %}

Последний пост был посвящен [структуре Django Model](https://kracekumar.com/post/structure\_django\_orm\_working\_part1/). В этом посте рассказывается, как работает поле модели, каковы некоторые важные методы, функциональные возможности и свойства поля.

[Object-Relational Mapper](https://en.wikipedia.org/wiki/Object%E2%80%93relational\_mapping) — это метод объявления и запроса таблиц базы данных с использованием отношений объектов на языке программирования. Вот пример объявления модели в Django.

```python
class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField('date published')
```

Каждый класс наследуется от `models.Model` и становится таблицей внутри базы данных SQL, если она явно не помечена как абстрактная. Модель **Question** становится таблицей **\<app\_name>\_question** в базе данных. **question\_text** и **pub\_date** становятся столбцами в таблице. Свойства каждого поля объявляются путем создания экземпляра соответствующего класса. Ниже приведен порядок разрешения методов для **CharField**.

```python
In [5]: models.CharField.mro()
Out[5]:
[django.db.models.fields.CharField,
 django.db.models.fields.Field,
 django.db.models.query_utils.RegisterLookupMixin,
 object]
```

**CharField** наследует **Field**, а **Field** наследует **RegisterLookUpMixin**.

## Высокоуровневая роль класса Field

1. Роль класса поля заключается в сопоставлении типа поля с типом базы данных SQL.
2. Сериализация — для преобразования объекта Python в соответствующее значение базы данных SQL.
3. Десериализация — для преобразования значения базы данных SQL в объект Python.
4. Проверяет объявленные проверки на уровне поля и встроенные проверки перед сериализацией данных. Например, в **PositiveIntegerField** значение должно быть больше нуля — встроенное ограничение.

## Структура класса поля

```python
# Узнать все классы, наследующие Field

In [7]: models.Field.__subclasses__()
Out[7]:
[django.db.models.fields.BooleanField,
 django.db.models.fields.CharField,
 django.db.models.fields.DateField,
 django.db.models.fields.DecimalField,
 django.db.models.fields.DurationField,
 django.db.models.fields.FilePathField,
 django.db.models.fields.FloatField,
 django.db.models.fields.IntegerField,
 django.db.models.fields.IPAddressField,
 django.db.models.fields.GenericIPAddressField,
 django.db.models.fields.TextField,
 django.db.models.fields.TimeField,
 django.db.models.fields.BinaryField,
 django.db.models.fields.UUIDField,
 django.db.models.fields.json.JSONField,
 django.db.models.fields.files.FileField,
 django.db.models.fields.related.RelatedField,
 django.contrib.postgres.search.SearchVectorField,
 django.contrib.postgres.search.SearchQueryField,
 fernet_fields.fields.EncryptedField,
 enumchoicefield.fields.EnumChoiceField,
 django.contrib.postgres.fields.array.ArrayField,
 django.contrib.postgres.fields.hstore.HStoreField,
 django.contrib.postgres.fields.ranges.RangeField]
```

Здесь **fernet\_fields** — это сторонняя библиотека, которая реализует **EncryptedField**, наследуя класс **Field**. Также это поля высокого уровня. Например, Django реализует другие высокоуровневые поля, которые наследуют вышеуказанные поля.

Например, **EmailField** наследует **CharField**.

```python
In [10]: models.CharField.__subclasses__()
Out[10]:
[django.db.models.fields.CommaSeparatedIntegerField,
 django.db.models.fields.EmailField,
 django.db.models.fields.SlugField,
 django.db.models.fields.URLField,
 django.contrib.postgres.fields.citext.CICharField,
 django_extensions.db.fields.RandomCharField,
 django_extensions.db.fields.ShortUUIDField]
```

Вот сгнатура инициализатора класса **Field**

```python
In [11]: models.Field?
Init signature:
models.Field(
    verbose_name=None,
    name=None,
    primary_key=False,
    max_length=None,
    unique=False,
    blank=False,
    null=False,
    db_index=False,
    rel=None,
    default=<class 'django.db.models.fields.NOT_PROVIDED'>,
    editable=True,
    serialize=True,
    unique_for_date=None,
    unique_for_month=None,
    unique_for_year=None,
    choices=None,
    help_text='',
    db_column=None,
    db_tablespace=None,
    auto_created=False,
    validators=(),
    error_messages=None,
)
```

Инициализатор **Field** содержит 22 аргумента. Большинство аргументов относятся к свойствам столбца базы данных SQL, а остальные аргументы относятся к формам админки и модели Django.

Например, Django предоставляет интерфейс администратора для просмотра записей базы данных и позволяет редактировать их. Параметр **blank** определяет, является ли поле обязательным при заполнении данных в интерфейсе администратора и пользовательской форме. Поле **help\_text** используется при отображении формы.

Наиболее часто используемые поля: **max\_length**, **unique**, **empty**, **null**, **db\_index**, **validators**, **default**, **auto\_created**. Атрибут **null** является логическим типом, если для него установлено значение `True`, разрешает нулевое значение при сохранении в базе данных. `db_index=True` создает индекс `B-Tree` для столбца. Атрибут **default** хранит значение по умолчанию, переданное в базу данных, когда значение для поля отсутствует.

Атрибут **validators** содержит список валидаторов, переданных пользователем, и внутренних валидаторов Django. Функция валидатора заключается в том, чтобы определить, является ли значение действительным или нет. Например, в нашем объявлении поля **question\_text** для параметра **max\_length** установлено значение 200. Когда значение поля больше 200, Django вызывает **ValidationError**. Атрибут **max\_length** полезен только для текстового поля, а **MaxLengthValidator** будет отсутствовать в нетекстовых полях.

```python
In [29]: from django.core.exceptions import ValidationError

In [30]: def allow_odd_validator(value):
    ...:     if value % 2 == 0:
    ...:         raise ValidationError(f'{value} is not odd number')
    ...:

In [31]: int_field = models.IntegerField(validators=[allow_odd_validator])

In [32]: int_field.validators
Out[32]:
[<function __main__.allow_odd_validator(value)>,
 <django.core.validators.MinValueValidator at 0x1305fdac0>,
 <django.core.validators.MaxValueValidator at 0x1305fda30>]

In [33]: # давайте посмотрим на валидаторы поля question_text

In [38]: question_text.validators
Out[38]: [<django.core.validators.MaxLengthValidator at 0x12e767fa0>]
```

Пока функция валидации или пользовательский класс не вызывает исключение, значение считается действительным.

Подробную информацию о каждом поле можно найти в [справочнике по полям модели Django](https://docs.djangoproject.com/en/3.2/ref/models/fields/).

## Методы Field

```python
In [41]: import inspect

In [44]: len(inspect.getmembers(models.Field, predicate=inspect.isfunction))
Out[44]: 59

In [45]: len(inspect.getmembers(models.Field, predicate=inspect.ismethod))
Out[45]: 6
```

Класс **Field** состоит из (вместе с унаследованными) 65 методов. Давайте посмотрим на некоторые из наиболее важных.

### to\_python

Метод **to\_python** отвечает за преобразование значения, переданного модели во время инициализации. Например, **to\_python** для **IntegerField** преобразует значение в целое число Python. Исходное значение может быть **string** или **float**. Каждое поле переопределяет метод **to\_python**. Вот пример вызова метода **to\_python** для **IntegerField**.

```python
In [46]: int_field.to_python
Out[46]: <bound method IntegerField.to_python of <django.db.models.fields.IntegerField>>

In [47]: int_field.to_python('23')
Out[47]: 23

In [48]: int_field.to_python(23)
Out[48]: 23

In [49]: int_field.to_python(23.56)
Out[49]: 23
```

### get\_db\_prep\_value

Метод **get\_db\_prep\_value** отвечает за преобразование значения Python в конкретное значение базы данных SQL. Каждое поле может иметь различную реализацию в зависимости от типа поля. Например, **Postgres** имеет собственный тип **UUID**, тогда как в **SQLite** и **MySQL** Django использует **varchar(32)**. Вот реализация **get\_db\_prep\_value** из **UUIDField**.

```python
def get_db_prep_value(self, value, connection, prepared=False):
	if value is None:
  	return None
  if not isinstance(value, uuid.UUID):
    value = self.to_python(value)

  if connection.features.has_native_uuid_field:
  	return value
  return value.hex
```

**connection** — это соединение с базой данных Database Connection или объект-оболочка Wrapper основной базы данных. Ниже приведен пример вывода из **Postgres Connection** и **SQLite Connection** для проверки поля **uuid**.

```python
In [50]: from django.db import connection
    ...:

In [51]: connection
Out[51]: <django.utils.connection.ConnectionProxy at 0x10e3c8970>

In [52]: connection.features
Out[52]: <django.db.backends.postgresql.features.DatabaseFeatures at 0x1236a6a00>

In [53]: connection.features.has_native_uuid_field
Out[53]: True
```

```python
In [1]: from django.db import connection

In [2]: connection
Out[2]: <django.utils.connection.ConnectionProxy at 0x10fe3b4f0>

In [3]: connection.features
Out[3]: <django.db.backends.sqlite3.features.DatabaseFeatures at 0x110ba5d90>

In [4]: connection.features.has_native_uuid_field
Out[4]: False
```

Следует отметить, что Django использует драйвер **psycopg2** для Postgres, и он позаботится об обработке UUID, характерного для Postgres, поскольку объект UUID Python необходимо преобразовать в строку или байты перед отправкой на сервер Postgres.

Подобно _get\_db\_prep\_value_, **get\_prep\_value** преобразует значение Python в значение запроса (_query value_).

### formfield

Django поддерживает **ModelForm**, который является однозначным сопоставлением HTML-формы с моделью Django. Админка Django использует **ModelForm**. Форма состоит из нескольких полей. Каждое поле в форме сопоставляется с полем в модели. Таким образом, Django может автоматически создать форму со списком валидаторов из поля модели.

Вот реализация **UUIDField**.

```python
def formfield(self, **kwargs):
	return super().formfield(**{
  	'form_class': forms.UUIDField,
    **kwargs,
   })
```

Когда вы создаете пользовательское поле базы данных, вам необходимо создать пользовательское поле формы для работы с админкой Django и передать его в качестве аргумента методу `super()`.

### deconstruct

Метод **deconstruct** возвращает значение для создания точной копии поля. Метод возвращает кортеж с 4 значениями.

* Первое значение — это имя поля **name**, переданного во время инициализации. Значение по умолчанию — `None`.
* Путь импорта поля.
* Список позиционных аргументов, переданных при создании поля.
* Словарь аргументов ключевых слов, передаваемых при создании поля.

```python
In [62]: # Давайте посмотрим, как метод деконструкции question_text
# возвращает значение.
In [63]: question_text.deconstruct()
Out[63]: (None, 'django.db.models.CharField', [], {'max_length': 200})

In [65]: # давайте создадим новое целочисленное поле с именем

In [66]: int_field = models.IntegerField(name='int_field', validators=[allow_odd_validator])

In [67]: int_field.deconstruct()
Out[67]:
('int_field',
 'django.db.models.IntegerField',
 [],
 {'validators': [<function __main__.allow_odd_validator(value)>]})

In [68]: models.IntegerField(**int_field.deconstruct()[-1])
Out[68]: <django.db.models.fields.IntegerField>

In [69]: int_2_field = models.IntegerField(default=2)

In [70]: int_2_field.deconstruct()
Out[70]: (None, 'django.db.models.IntegerField', [], {'default': 2})
```

Кроме того, когда вы реализуете пользовательское поле, вы можете переопределить метод **deconstruct**. Вот реализация **deconstruct** для **UUIDField**.

```python
def deconstruct(self):
    name, path, args, kwargs = super().deconstruct()
    del kwargs['max_length']
    return name, path, args, kwargs
```

### \_\_init\_\_

Метод `__init__` — это хорошее место для переопределения некоторых значений по умолчанию. Например, UUIDField **max\_length** всегда должно быть равно 32 независимо от передаваемого значения. В десятичном поле **max\_digits** можно изменить во время инициализации.

Вот реализация метода инициализатора **UUIDField**.

```python
def __init__(self, verbose_name=None, **kwargs):
    kwargs['max_length'] = 32
    super().__init__(verbose_name, **kwargs)
```

### db\_type <a href="#db_type" id="db_type"></a>

Метод **db\_type** принимает соединение Django в качестве аргумента и возвращает конкретный тип реализации базы данных для этого поля. Метод принимает **connection** в качестве аргумента. Вот вывод **db\_type** для Postgres и SQLite.

```python
In [72]: # Postgres

In [73]: uuid_field = models.UUIDField()

In [74]: uuid_field.db_type(connection)
Out[74]: 'uuid'
```

```python
In [8]: # Sqlite

In [9]: uuid_field = models.UUIDField()

In [10]: uuid_field.rel_db_type(connection)
Out[10]: 'char(32)'
```

Метод **get\_internal\_type** возвращает внутренний **internal** тип Python, который дополняет метод **db\_type**. На практике тип полей Django и сопоставление полей базы данных поддерживаются как переменная класса в **DatabaseWrapper**. Вы можете найти [поля Django и сопоставление полей Postgres в бэкэнд-модуле](https://github.com/django/django/blob/main/django/db/backends/postgresql/base.py). Ниже приведено сопоставление, взятое из исходного кода.

```python
class DatabaseWrapper(BaseDatabaseWrapper):
    vendor = 'postgresql'
    display_name = 'PostgreSQL'
    # Этот словарь сопоставляет объекты Field со связанными с ними типами
    # столбцов PostgreSQL в виде строк. Строки типа столбца могут содержать
    # строки формата; они будут интерполированы по значениям Field.__dict__
    # перед выводом. Если для типа столбца задано значение None, он не будет
    # включен в выходные данные.
    data_types = {
        'AutoField': 'serial',
        'BigAutoField': 'bigserial',
        'BinaryField': 'bytea',
        'BooleanField': 'boolean',
        'CharField': 'varchar(%(max_length)s)',
        'DateField': 'date',
        'DateTimeField': 'timestamp with time zone',
        'DecimalField': 'numeric(%(max_digits)s, %(decimal_places)s)',
        'DurationField': 'interval',
        'FileField': 'varchar(%(max_length)s)',
        'FilePathField': 'varchar(%(max_length)s)',
        'FloatField': 'double precision',
        'IntegerField': 'integer',
        'BigIntegerField': 'bigint',
        'IPAddressField': 'inet',
        'GenericIPAddressField': 'inet',
        'JSONField': 'jsonb',
        'OneToOneField': 'integer',
        'PositiveBigIntegerField': 'bigint',
        'PositiveIntegerField': 'integer',
        'PositiveSmallIntegerField': 'smallint',
        'SlugField': 'varchar(%(max_length)s)',
        'SmallAutoField': 'smallserial',
        'SmallIntegerField': 'smallint',
        'TextField': 'text',
        'TimeField': 'time',
        'UUIDField': 'uuid',
    }
```

Значения **get\_internal\_type** — это ключи, а значения — это имена полей Postgres.

Я пропустил реализацию остальных методов, таких как **\_\_reduce\_\_** и **check**. Вы можете просмотреть исходный код полей Django в [GitHub](https://github.com/django/django/blob/main/django/db/models/fields/\_\_init\_\_.py), а также найти переменные класса и использование частных методов.

В документации Django есть отличная страница о том, как [написать пользовательское поле модели](https://docs.djangoproject.com/en/3.2/howto/custom-model-fields/).

## Краткое изложение

1. `models.Field` является корнем всех полей модели.
2. Инициализатор поля принимает сведения о конфигурации, такие как название **name**, значение по умолчанию **default**, **db\_index**, **null** для столбцов базы данных и **blank**, **help\_text** для функций, не относящихся к столбцам, таких как форма модели Django и админка Django.
3. Метод `__init__` в дочернем классе может переопределить значение, переданное пользователем, и (или) установить пользовательское значение по умолчанию.
4. Атрибут **validators** в поле содержит определяемые пользователем валидаторы и валидаторы по умолчанию, характерные для поля.
5. Каждое поле должно реализовать несколько методов для работы с конкретными базами данных. Некоторые из полезных методов: **to\_python**, **get\_db\_prep\_value**, **get\_prep\_value**, **deconstruct**, **formfield**, **db\_type**.
6. Объект или оболочка Django **Connection** содержит детали и функции основной базы данных.
