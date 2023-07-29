# Когда декоратор встречает дескриптор

{% hint style="info" %}
Ссылка на оригинальную статью: [When a decorator meets a descriptor](https://blog.rebased.pl/2021/03/30/when-a-decorator-meets-a-descriptor.html)

Опубликовано: 30 марта 2021

Автор: Mariusz Michalak
{% endhint %}

Лучшее в этой работе – это такие моменты. Когда стоящая перед вами проблема позволяет вам еще глубже погрузиться в тайны языка, которым вы сейчас пользуетесь.

## Декоратор

Согласно официальной документации, определение декоратора:

> Функция, возвращающая другую функцию, обычно применяется как преобразование функции с использованием синтаксиса **@wrapper**. Типичными примерами декораторов являются **@classmethod()** и **@staticmethod()**.

Который часто предшествует примерам кода, подобным этому (для повышения удобочитаемости я добавил наивную реализацию статического метода ниже):

```python
def staticmethod(original_func):
  def wrapper(*args, **kwargs):
    # pre func code
    result = original_func()
    # post func code
    return result

  return wrapper

def f(...):
    ...

f = staticmethod(f)

@staticmethod
def f(...):
    ...
```

До этого момента здесь нет ничего необычного или сложного для понимания. Декораторы — это просто функции высокого уровня.

Но что, если вы хотите использовать декоратор для метода экземпляра? А что, если вы решите использовать класс в качестве декоратора? Подумайте: у вас есть определенный набор классов, используемых для фильтрации доступа, которые вы хотите использовать исключительно для определенных методов.

Наш класс в исходном виде мог бы выглядеть примерно так:

```python
class AccountActive():
  def apply(self):
    # Для простоты я опускаю тело этого метода.
    # Рассматривайте этот метод как своего рода фильтр before_filter
    # для проверки активности учетной записи.
    pass

```

Итак, что происходит, когда вы используете наш класс в качестве декоратора? Давайте начнем с определения нашего примера метода. Для целей этого поста я буду использовать простой случай, который является конечной точкой ресурса в **Flask**:

```python
from flask.views import MethodView

class Vehicles(MethodView):
  def get(self):
    # Найти конкретное транспортное средство и вернуть его
    pass
```

С примененным **декоратором** это не будет сильно отличаться:

```python
class Vehicles(API):
  @AccountActive
  def get(self):
    # Найти конкретное транспортное средство и вернуть его
    pass
```

Помня о том, что мы прочитали в представленном выше определении **декоратора**, класс **AccountActive** будет использоваться следующим образом:

```python
get = AccountActive(get)
```

Итак, мы видим, что наша исходная функция **get** будет использоваться в качестве аргумента конструктора. Затем мы должны обновить наш класс:

```python
class AccountActive():
  def __init__(self, func):
    self.func = func

  def apply(self):
    pass
```

Что дальше? Очевидно, что наш новый метод **get** является экземпляром класса **AccountActive** и должен вызываться как таковой. Впрочем, здесь по-прежнему нет ничего, что могло бы нас удивить:

```python
class AccountActive():
  def __init__(self, func):
    self.func = func

  def __call__(self, *args, **kwargs):
    # Это место, где мы применяем нашу логику before_filter
    self.apply()
    return self.func(*args, **kwargs)

  def apply(self):
    pass
```

Это должно сработать, верно? Работа сделана? Ну, не совсем так. Давайте посмотрим, что сообщает нам REPL, когда мы используем этот код:

```python
>>> resource = Vehicles()
>>> resource.get()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 6, in __call__
TypeError: get() missing 1 required positional argument: 'self'
```

Почему так, спросите вы. Разве мы не передали все аргументы внутри нашей обертки? Да, это так, но давайте вернемся назад и проверим определение декоратора. В нем говорится, что декоратор применяется к объекту функции, а не к связанному методу (подробнее о различиях между этими двумя концепциями можно прочитать в блоге моего коллеги «[Действительная сигнатура метода?](https://blog.rebased.pl/2021/03/09/a-valid-signature.html)»). Это означает, что мы потеряли наш аргумент экземпляра здесь и должны восстановить его, чтобы иметь возможность использовать наши фильтры.

Есть несколько решений, но мы сосредоточимся на том, которое оправдывает наличие дескриптора в заголовке поста.

## Дескриптор

Перво-наперво — давайте посмотрим, что такое **дескриптор**:

> Любой объект, который определяет методы **get()**, **set()** или **delete()**. Когда атрибут класса является дескриптором, его особое поведение связывания запускается при поиске атрибута. Обычно использование **a.b** для получения, установки или удаления атрибута ищет объект с именем **b** в словаре класса для **a**, но если **b** является дескриптором, вызывается соответствующий метод дескриптора. Понимание дескрипторов является ключом к глубокому пониманию Python, поскольку они являются основой для многих функций, включая функции, методы, свойства, методы класса, статические методы и ссылки на суперклассы.

Метод **get()** выглядит многообещающе, давайте углубимся в документацию:

> Вызывается для получения атрибута класса-владельца (доступ к атрибуту класса) или экземпляра этого класса (доступ к атрибуту экземпляра). Необязательный аргумент **owner** — это класс владельца, а **instance** — экземпляр, через который был получен доступ к атрибуту, или `None`, если доступ к атрибуту осуществляется через владельца.
>
> Этот метод должен возвращать вычисленное значение атрибута или вызывать исключение **AttributeError**.
>
> [PEP 252](https://www.python.org/dev/peps/pep-0252/) указывает, что **get()** можно вызывать с одним или двумя аргументами. Собственные встроенные дескрипторы Python поддерживают эту спецификацию; однако вполне вероятно, что некоторые сторонние инструменты имеют дескрипторы, требующие оба аргумента. Собственная реализация Python **getattribute()** всегда передает оба аргумента независимо от того, требуются они или нет.

Итак, после применения декоратора у нас уже не связанный метод, а простая функция. Теперь попробуем восстановить его, используя концепцию **дескриптора**. Учитывая вышеизложенное, мы можем ожидать, что после преобразования нашего фильтра **AccountActive** в дескриптор во время поиска атрибута на уровне объекта **Vehicles** мы столкнемся с методом **get** с некоторыми удобными аргументами.

Во-первых, давайте сделаем наш **AccountActive** дескриптором (точнее, **дескриптором, не связанным с данными**, потому что мы будем реализовывать только метод **get**):

```python
class AccountActive():
  def __init__(self, func):
    self.func = func

  def __call__(self, *args, **kwargs):
    self.apply()
    return self.func(*args, **kwargs)

  def __get__(self, instance, owner=None):
    # self — это экземпляр нашего класса AccountActive,
    # instance — это экземпляр нашего класса Vehicles
    # (место, с которого начинается поиск)

    # Теперь мы возвращаем новую функцию с восстановленным аргументом экземпляра
    # в первой позиции, используя частичный хелпер.
    return partial(self.__call__, instance)

  def apply(self):
    pass
```

На этот раз REPL молчит, а наши тесты все зеленые.

## Заключение

Мы использовали концепцию **дескриптора** вместе с хелпером **partial** для создания связанного метода. Таким образом, мы возвращаем недостающий позиционный аргумент. Тем не менее, дескриптор может предложить гораздо больше, чем вы могли подумать. Я призываю вас изучить его возможности в [Руководстве по дескриптору](https://docs.python.org/3/howto/descriptor.html#descriptorhowto).