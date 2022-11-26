# Добавление к классам

{% hint style="info" %}
Ссылка на оригинальную статью: [Contributing to classiness (in Django)](https://www.b-list.org/weblog/2019/mar/04/class/)

Опубликовано: 4 марта 2019

Автор: [James Bennett](https://www.b-list.org/)
{% endhint %}

Пару недель назад я провел [опрос в Твиттере](https://twitter.com/ubernostrum/status/1096609885213491201), в котором спрашивал людей, использовали ли они когда-либо или рассматривали возможность использования метода `contribute_to_class()`, чтобы написать что-то, что прикрепляется к классу модели Django ORM или подключается к нему, и если да, то что они думают об этом. Был также вариант «не знаю, что это такое», который выиграл с большим отрывом, и я пообещал дать объяснение.

К сожалению, это было примерно в то время, когда я попал в аварию на кухне, из-за которой я на некоторое время полностью не мог пользоваться левой рукой. Полное заживление займет немного больше времени, но это тот момент, когда я снова могу нормально печатать, поэтому пришло время дать объяснение.

Однако сначала мне нужно дать некоторое объяснение объяснению. Оставайтесь со мной, и, надеюсь, это будет иметь смысл в конце.

## Сохранение стиля

Если вы использовали Django или хотя бы прошли несколько вводных руководств по Python, вы, вероятно, написали или видели несколько классов Python. Вот простой пример:

```python
import math

class Circle:
    def __init__(self, center, radius):
        self.center = center
        self.radius = radius

    def area(self):
        return math.pi * (self.radius ** 2)

    def circumference(self):
        return 2 * math.pi * self.radius
```

В нем есть большинство вещей, к которым мы привыкли из классов: у него есть некоторые методы, у него есть некоторые атрибуты, у него есть конструктор, который вы можете вызывать для создания нового экземпляра и передавать аргументы, которые будут влиять на новый экземпляр. Но учебники обычно не углубляются в это и не рассказывают, как Python на самом деле обрабатывает классы. Итак, давайте поработаем над этим.

В приведенном выше классе **Circle** строка `class Circle:` начинает блок, который содержит еще девять строк кода (за которыми следует, предположительно, либо удаление отступа, либо конец файла). Все внутри этого блока выполняется, и результатом является словарь dict, _**представляющий локальное пространство имен этого блока**_. Ключи — это имена вещей, определенных внутри блока, а значения — это… их значения. Например, блок кода для класса **Circle** создаст словарь, содержащий три ключа: `'__init__'`, `'area'` и `'circumference'`.

Затем Python собирает три вещи — имя, данное в начальном операторе **class**, родительские классы (если они есть), включенные в начальный оператор **class**, и словарь **dict**, представляющий пространство имен тела класса, — и передает их в указанном порядке в `type()`. Возвращаемое значение из `type()` привязано к имени класса. Важно отметить, что `type()` не является функцией. Это класс, а именно класс классов: точно так же, как вы вызываете `Circle()` для создания нового экземпляра класса **Circle**, вы вызываете `type()` для создания нового экземпляра класса… класса.

Таким образом, процесс определения класса **Circle** состоит из следующих этапов:

1. Выполняется тело класса и собирается всё из полученного пространства имен классов в словарь **dict**. В данном случае у него будут три упомянутых выше ключа, соответствующих названиям трех определенных методов.
2. Вызывается `type('Circle', (), namespace_dict)`.
3. Привязывается результат к имени **Circle**.

Если вы хотите, вы можете сделать это вручную, создав словарь **dict** вещей, которые нужно поместить в класс, и вызвать `type()`:

```python
>>> import math
>>> def __init__(self, center, radius):
...     self.center = center
...     self.radius = radius
...
>>> def area(self):
...     return math.pi * (self.radius ** 2)
...
>>> def circumference(self):
...     return 2 * math.pi * self.radius
...
>>> Circle = type('Circle', (), {'__init__': __init__, 'area': area, 'circumference': circumference})
>>> Circle
<class '__main__.Circle'>
>>> c = Circle(center=(0, 0), radius=1)
>>> c.area()
3.141592653589793
>>> c.circumference()
6.283185307179586
```

Итак, теперь мы понимаем, как Python обрабатывает определение класса. Но есть еще одна вещь, о которой нам нужно знать, прежде чем мы вернемся к Django.

## Вот так meta

Одна продвинутая — и неправильно понятая, и часто неправильно используемая — функция Python — это то, что называется **«metaclass»**. Идею довольно легко объяснить: _**это способ подключиться к описанному выше процессу и изменить способ создания объектов класса**_.

Ранее в этом посте я описал `__init__()` как конструктор класса, и часто именно так с ним знакомятся новые программисты Python. Хотя различие почти никогда не имеет значения, _**это не совсем правильно**_: создание экземпляра класса в Python на самом деле представляет собой двухэтапный процесс, где сначала будет вызываться метод класса `__new__()`, а затем его метод `__init__()`.

Другими словами, если мы возьмем описанный выше класс **Circle** и сделаем `c = Circle(center=(0, 0), radius=1)`, Python на самом деле сделает примерно следующее:

```python
c = Circle.__new__(Circle, center=(0, 0), radius=1)
c.__init__(center=(0, 0), radius=1)
```

Если эта первая строка выглядит немного странно: `__new__()` на самом деле является _**статическим**_ методом, хотя Python использует его в особых случаях, поэтому вам не нужно использовать декоратор **staticmethod** при его определении. Это не может быть метод экземпляра, поскольку именно он создает экземпляры. А поскольку неявный первый аргумент не передается автоматически (например, неявный **self** метода экземпляра), класс передается явно в качестве первого аргумента.

Людям нравится быть педантичными и утверждать, что `__new__()` — это настоящий «конструктор», а `__init__()` — «инициализатор», но это редко бывает полезным аргументом; когда вы создаете новый экземпляр класса, Python _**вызывает оба метода**_, прежде чем вернуть вам экземпляр.

Но: поскольку определение нового класса в конечном итоге включает создание нового экземпляра **type**, это означает, что он будет включать вызов `type.__new__()`. Таким образом, механизм Python для настройки создания классов состоит из подкласса **type** и переопределения `__new__()`, а затем указывает Python использовать ваш подкласс **type**. Класс, который делает это, является **метаклассом**.

Вот очень небольшой пример:

```python
>>> class SimpleMetaclass(type):
...     def __new__(cls, name, bases, attrs):
...         attrs['special_attribute'] = 'Special!'
...         return super().__new__(cls, name, bases, attrs)
...
>>> class SpecialClass(metaclass=SimpleMetaclass):
...     pass
...
>>> SpecialClass.special_attribute
'Special!'
```

Набор аргументов для `__new__()` — это класс (это всегда первый аргумент для `__new__()`, как мы видели выше), плюс конкретные аргументы, используемые для создания экземпляра `type()`, поскольку это то, что мы создаем в подклассе.

И хотя **SpecialClass** никогда не определял атрибут с именем **special\_attribute**, он по-прежнему имеет этот атрибут, потому что мы сказали Python использовать **SimpleMetaclass** (вместо **type**) при построении класса **SpecialClass**, а **SimpleMetaclass** вставляет атрибут **special\_attribute** в любой объект класса, который он создает.

Это может быть _**очень**_ полезной функцией для настройки автоматических и, казалось бы, «магических» атрибутов, методов и поведения класса; вы можете написать метакласс, который вносит нужные вам изменения, и либо использовать его напрямую, либо предоставить базовый класс, который его использует, и наследовать от этого базового класса (_**дочерние классы наследуют метакласс своих родителей**_).

А теперь пришло время для краткого отступления. В этом посте есть два места, где я обязан сделать предупреждение, и это одно из них. Метаклассы кажутся классной вещью, когда вы впервые о них узнаете, и вы, вероятно, сможете придумать всевозможные случаи, когда они будут полезны. Но случаи, когда они действительно полезны, более редки; _**обычно то, для чего вы написали бы метакласс, это то, что вы могли бы сделать так же легко (и гораздо яснее), поместив поведение в свой класс в первую очередь**_. Большинство недостатков метаклассов можно разделить на две категории:

1. Они могут затруднить понимание и анализ кода, поскольку глубоко внутри иерархии классов может быть неочевидно, что один из родительских классов имеет метакласс, и в результате может показаться, что что-то просто появляется или меняется внутри класса без причины.
2. Когда у класса есть несколько родителей, использующих _**разные**_ метаклассы, Python выдает **TypeError** и сообщает вам: «метакласс производного класса должен быть (нестрогим) подклассом метаклассов всех его баз» (_the metaclass of a derived class must be a (non-strict) subclass of the metaclasses of all its bases_). Поскольку это не очень полезно, вы затем вставляете это сообщение об ошибке в Google, чтобы узнать, что вы должны делать. Ответ заключается в том, что вы пишете еще один метакласс, который подклассирует все метаклассы родительских классов, разрешая любые конфликтующие или противоречивые вещи, которые они хотят сделать, и используете этот метакласс в своем дочернем классе. Это может быть сложно или невозможно, в зависимости от того, что делают другие метаклассы.

## А теперь немного Django

Если вы писали классы моделей с помощью Django ORM, вы, возможно, заметили некоторое «магическое» поведение — некоторые вещи, такие как объявление **Meta**, перемещаются после того, как вы их определили, а иногда в классе модели появляются вещи, которые вы никогда не определяли. Например, если вы добавите в модель ненулевое поле **DateField** с именем **pub\_date**, эта модель автоматически создаст методы с именами `get_next_by_pub_date()` и `get_previous_by_pub_date()`.

Если вы дочитали до этого поста, то теперь знаете (или можете понять), как это происходит: **django.db.models.Model**, базовый класс для всех моделей Django, использует метакласс (**django.db .models.base.ModelBase**). Метакласс модели обрабатывает множество вещей, в том числе:

1. Регистрация нового класса модели с правильной конфигурацией приложения.
2. Создание и присоединение подклассов исключений, чтобы класс модели мог иметь свои собственные **DoesNotExist** и другие исключения.
3. При необходимости создание менеджера по умолчанию и прикрепление его в качестве атрибута **objects**.

Если вы хотите, вы можете прочитать [весь метакласс модели](https://github.com/django/django/blob/2.1.7/django/db/models/base.py#L61) (эта ссылка ведет на реализацию Django 2.1) на GitHub, чтобы увидеть все, что он делает. Но для этого поста важно то, как обрабатываются атрибуты класса модели.

Когда **ModelBase** просматривает словарь **dict** атрибутов, которые составят класс модели, который он создает, он проверяет каждый атрибут, который в конечном итоге окажется в модели, чтобы увидеть, _**имеет ли значение, присвоенное атрибуту, метод с именем**_ `contribute_to_class()`. Если это так, он вызовет метод `contribute_to_class()`, передав определяемый класс модели и имя назначаемого атрибута. Это позволяет перенести большую часть работы из **ModelBase** в различные типы вещей, которые в конечном итоге становятся атрибутами в классах моделей.

Довольно много встроенных полей модели Django определяют метод `contribute_to_class()` и используют его для «магии». Например, метод **dateField** `contribute_to_class()` — это то, что устанавливает эти методы поиска следующего/предыдущего значения. Поля отношения используют `contribute_to_class()` для настройки своей инфраструктуры, включая вставку «reverse» конца отношения в другой класс модели. Менеджеры используют `contribute_to_class()`, чтобы узнать, к какой модели они подключены, а некоторые другие внутренние компоненты ORM используют его для учета и обеспечения правильной обработки конфигурации модели.

А теперь настало время для второго предупреждения в этом посте: `contribute_to_class()` — это _**недокументированный частный внутренний API**_. Django не предоставляет гарантий обратной совместимости для него, и если вы используете его, вы принимаете на себя риск того, что он может измениться или сломаться в любое время.

Но иногда стороннему коду нужен способ подключиться к ORM и повлиять на настройку класса модели. Например, некоторые типы сложных настраиваемых полей модели будут использовать это, хотя это не единственный вариант использования. И когда вам нужно это сделать, вы должны это сделать; `contribute_to_class()` есть, и он сделает это за вас.

## Мой (потенциальный) вариант использования

Я провел этот опрос в Твиттере из-за кое-чего, что я делал на работе. Не вдаваясь в подробности: я работал над доказательством концепции замены нескольких экземпляров специального кода конечного автомата реальными конечными автоматами (я очень поддерживаю людей, использующих больше конечных автоматов, особенно в комбинации с Django ORM, но это история для другого дня).

Это включало подключение библиотеки конечного автомата — той, с которой я экспериментировал, была [Automat](https://github.com/glyph/automat) — к некоторым моделям Django. Я хотел что-то, что я мог бы установить в качестве атрибута в классе модели, указав используемый класс конечного автомата и поля модели, которые он должен использовать для хранения и считывания состояния, и я хотел убедиться, что:

1. Рассматриваемые поля больше нельзя будет редактировать с помощью админки Django или других вещей, использующих библиотеку форм Django, и
2. Всякий раз, когда создается экземпляр модели, также будет создан экземпляр конечного автомата, подключенный к правильному атрибуту и правильно инициализированный из полей модели (или переведенный в исходное состояние по умолчанию для новых экземпляров модели).

Потратив немного времени на размышления об этом, я, в конце концов, решил написать класс-оболочку для конечного автомата, и пусть оболочка будет тем, что установлено в качестве атрибута класса модели. Затем оболочка может использовать хук `contribute_to_class()`, чтобы обеспечить настройку требуемого поведения.

Но насколько я помню, это был первый раз, когда я когда-либо писал что-то, что использовало `contribute_to_class()` и не было ни частью Django, ни пользовательским полем модели. Поэтому мне стало любопытно, есть ли другие люди, использующие эту технику. GitHub находит 363 000 результатов для `contribute_to_class` в общедоступных репозиториях, но многие из них, по-видимому, являются репозиториями, в которых собраны полные копии Django. И тогда я обратился к Твиттеру.

Я до сих пор не уверен, что это был правильный подход к тому, что я пытался написать, или даже в том, что я писал правильные вещи. И никто из людей, выбравших «Да, и я бы хотел, чтобы я этого не делал» или «Нет, я решил против этого» в опросе в Твиттере, не дал никаких дополнительных объяснений в ответах, так что у меня также нет большого контекста относительно отрицательный опыт людей. Я ожидаю, что это то, о чем я потрачу еще некоторое время на размышления .

## Последнее замечание

Если вы были на DjangoCon в США в конце прошлого года и посещали мой туториал «[Освоение Django ORM](https://2018.djangocon.us/tutorial/mastering-the-django-orm/)», надеюсь, вам не нужно было отвечать «Не знаю, что это такое» в опросе в Твиттере, потому что один раздел этого туториала был экскурсия по внутренностям ORM, включая метакласс модели и хук `contribute_to_class()`. Другие разделы касались общедоступного API, лучших практик и многих других материалов по ORM.

К сожалению, в 2018 году DjangoCon не записывала туториалы, поэтому видео «Освоение Django ORM» нет. PyCon US записывает учебные пособия, но PyCon отклоняет это руководство уже два года подряд (и на данный момент я не в восторге от комитета по учебным пособиям PyCon, хотя и по другим причинам, чем просто «мое руководство было отклонено»).

Поэтому в редкие свободные минуты я думал о том, как сделать этот материал более доступным. Наиболее вероятным вариантом на данный момент является книга (и нет, я не ищу издателя прямо сейчас; если я это сделаю, я сначала напишу книгу, а потом решу, как ее опубликовать), но это довольно серьезное обязательство, и я не думаю, что сделал бы это, если бы не знал, что на это есть большой спрос. Поэтому, если вы считаете, что углубленная книга по Django ORM будет вам полезна — особенно если вы считаете, что она будет достаточно полезна, чтобы вы купили копию — дайте мне знать.