# Класс декоратора Python для оформления автономных функций и методов объектов

{% hint style="info" %}
Ссылка на оригинальную статью: [Python decorator class to decorate standalone functions and object methods](https://sorokin.engineer/posts/en/python\_class\_decorator.html)

Опубликовано:

Автор: [andGINEER](https://sorokin.engineer/en/index.html), Andrey Sorokin
{% endhint %}

Продвинутая техника, которую я описываю ниже, предполагает, что вы понимаете базовую механику декораторов Python и дескрипторов Python.

Предположим, мы хотим декорировать метод **func\_to\_wrap** класса **ClassToWrap**.

Если мы создадим класс декоратора Python, как показано ниже:

```python
class Decorator:
    def __init__(self, orig_func):
        self.orig_func = orig_func

    def __call__(self, *args):
        return self.orig_func(*args)
```

Мы не сможем его использовать:

```python
class ClassToWrap:
    @Decorator
    def func_to_wrap(self):
        self.name += '!'
        print(self.name)
        return self.name
        
c = ClassToWrap()
c.func_to_wrap()
```

```bash
func_to_wrap() missing 1 required positional argument: 'self'
```

Как видно из ошибки, `Decorator.__call__` вызывает **func\_to\_wrap** без **self**. Это связано с тем, что **orig\_func** не привязан к экземпляру объекта, и мы должны передать экземпляр в качестве первого аргумента. Но мы не можем!

**Self** внутри `Decorator.__call__` является экземпляром **Decorator**.

У нас нет экземпляра **ClassToWrap** внутри `Decorator.__call__`.

Конечно, есть простое решение - написать декоратор как функцию.

Но я собираюсь показать вам, как написать декоратор как класс, используя дескрипторы Python.

Более того, декоратор будет универсальным и может применяться как к отдельным функциям, так и к методам объекта.

Протокол дескриптора Python очень прост — если атрибут объекта имеет метод `__get__`, тогда Python вызовет его и вернет в качестве значения атрибута результат `__get__`.

Ниже вы можете увидеть рабочий пример.

`UniversalDecorator.__call__` будет работать, если мы декорируем автономные функции.

Для метода класса вместо `__call__` Python вызовет `__get__` и после этого вызовет его результат как функцию.

`UniversalDecorator.__get__` возвращает **WrapperHelper** со ссылками на экземпляры **ClassToWrap** и **Decorator**.

Когда Python вызывает результат `__get__` (экземпляр **WrapperHelper**), он фактически вызывает `WrapperHelper.__call__`.

`WrapperHelper.__call__` добавляет экземпляр **ClassToWrap** в качестве первого элемента в `*args` (строка 6). Таким образом, **func\_to\_wrap** получит экземпляр **ClassToWrap** в качестве первого аргумента.

```python
class UniversalDecorator:
    """ Хорошо подходит для автономных функций, а также для методов объекта """
    def __init__(self, orig_func):
        self.orig_func = orig_func

    def __call__(self, *args):
        """ Для самостоятельной функциональной отделки """
        return self.orig_func(*args)

    def __get__(self, wrapped_instance, owner):
        """ Для оформления метода объекта.
            Он обнаружит __get__ как протокол дескриптора.
            Итак, __get__ должен возвращать фактический метод.
        """
        print(wrapped_instance.name)
        # передает экземпляр декоратора и экземпляр декоративного объекта
        return WrapperHelper(self, wrapped_instance)


class WrapperHelper:
    """ Вызываемый объект, хранящий обернутый экземпляр класса и экземпляр декоратора """
    def __init__(self, decorator_instance, wrapped_instance):
        self.decorator_instance = decorator_instance
        self.wrapped_instance = wrapped_instance

    def __call__(self, *args, **kwargs):
        """ Вызовите func из декоратора как метод объекта —
        добавьте сохраненный экземпляр декорированного объекта """
        return self.decorator_instance(self.wrapped_instance, *args, **kwargs)


class ClassToWrap:
    name="123"

    def __init__(self):
        pass

    @UniversalDecorator
    def func_to_wrap(self):
        self.name += '!'
        print(self.name)
        return self.name


u = ClassToWrap()
u.func_to_wrap()
```
