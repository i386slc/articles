# Запрос JSONField в Django

{% hint style="info" %}
Ссылка на оригинальную статью: [Querying JSONField in Django](https://ishan1608.wordpress.com/2018/01/05/querying-jsonfield-in-django/)

Опубликовано: 5 января 2018

Автор: [Enriching Myself](https://ishan1608.wordpress.com/)
{% endhint %}

Если вы являетесь разработчиком Django и вам приходится иметь дело с типами данных, для которых вы не знаете тип данных или количество хранимых значений, лучше всего использовать **JSONField**.

Вы получаете **JSONField**, если используете Postgres.

Когда вы не знаете схему данных, которые вам нужно хранить, первое, что приходит на ум, это **NOSQL**. Однако это означает, что многим придется пожертвовать удобствами SQL и **ForeignKey** (лично я доволен обоими).

Использование **JSONField** немного затрудняет фильтрацию данных в **JSONField**.

В моей работе мне приходится иметь дело с данными, где мы заранее не знаем количество полей или тип данных. Мы решили использовать **JSONField**. Мне нужно было выполнить некоторую фильтрацию значений, и для фильтрации целых и десятичных значений не так много поддержки или документации. В итоге я сделал **KeyTransform** для значений, прежде чем выполнять их фильтрацию. Вот что у меня получилось.

Рассмотрим эту модель:

```python
class Data(models.Model):
    data = JSONField(default=dict)
```

и здесь мы заполняем некоторые данные:

```python
for i in xrange(10):
    data = {
        'string': 'Data: {}'.format(i),
        'integer': i,
        'float': i * 1.1
    }
    Data.objects.create(data=data)

for i in xrange(10):
    data = {
        'string2': 'Data: {}'.format(i),
        'integer2': i,
        'float2': i * 1.1
    }
    Data.objects.create(data=data)
```

Аннотация:

Чтобы получить доступ к значению внутри **JSONField**, а затем выполнить его фильтрацию, вам необходимо аннотировать его.

Пример аннотации:

```python
@classmethod
def fetch_values_constant_key(cls):
    """
    Выбирает все значения ключа "string"
    :return: значения
    """
    return Data.objects.annotate(
        **{
            'val': KeyTransform('string', 'data')
        }
    ).values_list('val', flat=True)
```

Совет:

Не используйте `«val»` в качестве аннотированного имени. Это переопределит любую аннотацию, которую вы сделали в прошлом, если вы делаете более одной аннотации. Используйте ключ значения непосредственно в качестве аннотированного имени. В общем, вы будете в безопасности без каких-либо конфликтов, но для безопасности вы также можете добавить некоторую константу (Совет: используйте имя **JSONField**).

```python
@classmethod
def fetch_values_dynamic_key(cls, key):
    """
    Извлекает все значения с помощью динамического ключа
    :param key: ключ для поиска внутри JSONField
    :return: значения
    """
    return Data.objects.annotate(
        **{
            key: KeyTransform(key, 'data')
        }
    ).values_list(key, flat=True)
```

Вам нужно использовать **KeyTextTransform** для текста. Вот пример применения фильтрации `«endswith»`.

```python
@classmethod
def filter_string_dynamic_key(cls, key, value):
    """
    Фильтрует ENDSWITH и возвращает текст из JSONField
    Ключ внутри JSONField является динамическим
    :param key: ключ для поиска внутри JSONField
    :param value: значение для фильтрации
    :return: значения
    """
    transforms = {
        key: KeyTextTransform(key, 'data')
    }
    annotated = Data.objects.annotate(**transforms)
    return annotated.filter(
        **{'{}__endswith'.format(key): value}
    ).values_list(key, flat=True)
```

## Фильтрация чисел

К сожалению, Django не предоставляет преобразования для правильной фильтрации числовых значений и возврата данных. По понятным причинам вы не можете использовать **KeyTextTransform**, как в случае с текстом.

(Не выполняется) Пример кода здесь с **KeyTransform**

В итоге я создал собственное преобразование для целых и десятичных значений.

### KeyIntegerTransform

```python
class KeyIntegerTransform(KeyTextTransform):
 
    def as_sql(self, compiler, connection):
        key_transforms = [self.key_name]
        previous = self.lhs
        while isinstance(previous, KeyTransform):
            key_transforms.insert(0, previous.key_name)
            previous = previous.lhs
        lhs, params = compiler.compile(previous)
        if len(key_transforms) > 1:
            return u"((%s %s %%s)::integer)" % (lhs, self.nested_operator), [key_transforms] + params
        try:
            int(self.key_name)
        except ValueError:
            lookup = u"'%s'" % self.key_name
        else:
            lookup = u"%s" % self.key_name
        return u"((%s %s %s)::integer)" % (lhs, self.operator, lookup), params
```

Теперь, используя **KeyIntegerTransform**, вы получаете взамен целочисленные значения (отлично!).

```python
@classmethod
def filter_number_key_integer_transform(cls, key, value):
    """
    Фильтрует GTE (целые числа) и возвращает число из JSONField,
    вызывает DataError, если данные не в целочисленном формате.
    :param key: ключ для поиска внутри JSONField
    :param value: значение для фильтрации
    :return: значения (числа как int) Н-р: 8
    """
    transforms = {
        key: KeyIntegerTransform(key, 'data')
    }
    annotated = Data.objects.annotate(**transforms)
    return annotated.filter(
        **{'{}__gte'.format(key): value}
    ).values_list(key, flat=True)
```

### KeyDecimalTransform

В итоге я создал **KeyDecimalTransform** для обработки значений с плавающей запятой.

```python
class KeyDecimalTransform(KeyTextTransform):
 
    def as_sql(self, compiler, connection):
        key_transforms = [self.key_name]
        previous = self.lhs
        while isinstance(previous, KeyTransform):
            key_transforms.insert(0, previous.key_name)
            previous = previous.lhs
        lhs, params = compiler.compile(previous)
        if len(key_transforms) > 1:
            return u"((%s %s %%s)::decimal)" % (lhs, self.nested_operator), [key_transforms] + params
        try:
            int(self.key_name)
        except ValueError:
            lookup = u"'%s'" % self.key_name
        else:
            lookup = u"%s" % self.key_name
        return u"((%s %s %s)::decimal)" % (lhs, self.operator, lookup), params
```

Однако он возвращает мне десятичный объект вместо числа с плавающей запятой.

```python
@classmethod
def filter_number_key_decimal_transform(cls, key, value):
    """
    Фильтрует GTE и возвращает число из JSONField, вызывает DataError,
    если данные не в целочисленном формате.
    :param key: ключ для поиска внутри JSONField
    :param value: значение для фильтрации
    :return: значения (числа как Decimal) Н-р: Decimal('8')
    """
    transforms = {
        key: KeyDecimalTransform(key, 'data')
    }
    annotated = Data.objects.annotate(**transforms)
    return annotated.filter(
        **{'{}__gte'.format(key): value}
    ).values_list(key, flat=True)
```

Если вас не волнует, что полученные значения являются **int**. Вы можете использовать более общее преобразование:

```python
class KeyNumericTransform(KeyTextTransform):
 
    def as_sql(self, compiler, connection):
        key_transforms = [self.key_name]
        previous = self.lhs
        while isinstance(previous, KeyTransform):
            key_transforms.insert(0, previous.key_name)
            previous = previous.lhs
        lhs, params = compiler.compile(previous)
        if len(key_transforms) > 1:
            return u"((%s %s %%s)::numeric)" % (lhs, self.nested_operator), [key_transforms] + params
        try:
            int(self.key_name)
        except ValueError:
            lookup = u"'%s'" % self.key_name
        else:
            lookup = u"%s" % self.key_name
        return u"((%s %s %s)::numeric)" % (lhs, self.operator, lookup), params
```

Это вернет объекты **Decimal**.

```python
@classmethod
def filter_number_key_numeric_transform(cls, key, value):
    """
    Фильтрует GTE и возвращает число из JSONField
    :param key: ключ для поиска внутри JSONField
    :param value: значение для фильтрации
    :return: значения (числа как Decimal Objects) Н-р: Decimal('8')
    """
    transforms = {
        key: KeyNumericTransform(key, 'data')
    }
    annotated = Data.objects.annotate(**transforms)
    return annotated.filter(
        **{'{}__gte'.format(key): value}
    ).values_list(key, flat=True)
```

Весь этот пример кода можно найти в моей учетной записи bitbucket [здесь](https://bitbucket.org/ishanatmuz/django\_snippets/src/06ac2c3d0165a631e859c0de2145267ed69b28f0?at=PostgreSQL-JSON-NumericTransform). Примеры [моделей и аннотаций](https://bitbucket.org/ishanatmuz/django\_snippets/src/06ac2c3d0165a631e859c0de2145267ed69b28f0/website/models.py?at=PostgreSQL-JSON-NumericTransform\&fileviewer=file-view-default), [KeyTransform(s)](https://bitbucket.org/ishanatmuz/django\_snippets/src/06ac2c3d0165a631e859c0de2145267ed69b28f0/website/utils.py?at=PostgreSQL-JSON-NumericTransform\&fileviewer=file-view-default). Попробуйте запустить [run\_examples](https://bitbucket.org/ishanatmuz/django\_snippets/src/06ac2c3d0165a631e859c0de2145267ed69b28f0/website/models.py?at=PostgreSQL-JSON-NumericTransform\&fileviewer=file-view-default#models.py-120), чтобы увидеть весь код в действии.

Я искренне надеюсь, что вы найдете это полезным.

Это все люди.
