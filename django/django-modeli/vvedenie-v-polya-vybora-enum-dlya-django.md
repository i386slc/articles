# Введение в поля выбора Enum для Django

{% hint style="info" %}
Ссылка на оригинальную статью: [Introducing an Enum choice field for Django](https://www.hacksoft.io/blog/introducing-django-enum-choices-2)

Опубликовано: 30 июля 2019

Автор: Radoslav Georgiev
{% endhint %}

{% hint style="warning" %}
Отказ от ответственности

Начиная с версии 3.0, Django начал поддерживать [Enumerations for model field choices](https://docs.djangoproject.com/en/dev/releases/3.0/#enumerations-for-model-field-choices), и мы рекомендуем использовать это как собственную функцию Django вместо нашей библиотеки [django-enum-choices](https://github.com/HackSoftware/django-enum-choices).
{% endhint %}

## Мотивация

Во многих проектах Django мы используем поля выбора.

Типичная модель может выглядеть так:

```python
class SomeModelWithChoices(models.Model):
    OK = 'ok'
    PENDING = 'pending'
    FAILED = 'failed'

    CHOICES = (
        (OK, 'Ok'),
        (PENDING, 'Pending'),
        (FAILED, 'Failed'),
    )

    status = models.CharField(max_length=255, choices=CHOICES, default=PENDING)
```

Обычно удобочитаемая часть создается на фронтенде, поэтому мы просто избавляемся от нее:

```python
class SomeModelWithChoices(models.Model):
    OK = 'ok'
    PENDING = 'pending'
    FAILED = 'failed'

    CHOICES = (
        (OK, OK),
        (PENDING, PENDING),
        (FAILED, FAILED),
    )

    status = models.CharField(max_length=255, choices=CHOICES, default=PENDING)
```

Это нормально.

**Ситуация начинает запутываться, если у нас есть более 1 поля выбора в модели.**

Например:

```python
class SomeModelWithChoices(models.Model):
    OK = 'ok'
    PENDING = 'pending'
    FAILED = 'failed'

    CHOICES_A = (
        (OK, OK),
        (PENDING, PENDING),
        (FAILED, FAILED),
    )

    WAITING = 'waiting'
    CANCELLED = 'cancelled'
    READY = 'ready'

    CHOICES_B = (
        (WAITING, WAITING),
        (CANCELLED, CANCELLED),
        (READY, READY),
    )

    status_A = models.CharField(max_length=255, choices=CHOICES_A, default=PENDING)
    status_B = models.CharField(max_length=255, choices=CHOICES_B, default=WAITING)
```

Даже с двумя полями выбора это становится нечитаемым.

Таким образом, естественным прогрессом является извлечение констант и добавление небольшого уровня абстракции:

```python
def get_choices(constants_class: Any) -> List[Tuple[str, str]]:
    return [
        (value, value)
        for key, value in vars(constants_class).items()
        if not key.startswith('__')
    ]

class StatusAConstants:
    OK = 'ok'
    PENDING = 'pending'
    FAILED = 'failed'

class StatusBConstants:
    WAITING = 'waiting'
    CANCELLED = 'cancelled'
    READY = 'ready'

class SomeModelWithChoices(models.Model):
    status_A = models.CharField(
        max_length=255,
        choices=get_choices(StatusAConstants),
        default=StatusAConstants.PENDING
    )
    status_B = models.CharField(
        max_length=255,
        choices=get_choices(StatusBConstants),
        default=StatusBConstants.WAITING
    )
```

Несколько вещей, которые мы заметили при таком подходе:

* Мы всегда указываем **max\_length=255** и фактически не учитываем правильную максимальную длину.
* Если мы хотим перебрать все возможные константы, нам нужно снова использовать **get\_choices**.
* Мы копируем Enums, а [в Python есть enum.Enum](https://docs.python.org/3/library/enum.html).

Так почему бы не создать что-то, что использует Enums? [Что ж, мы так и сделали](https://github.com/HackSoftware/django-enum-choices).

Мы создали небольшой слой поверх **models.CharField** с вариантами выбора, который использует **enum.Enum** Python в качестве источника.

Вот приведенный выше пример с использованием **EnumChoiceField**:

```python
class StatusAEnum(Enum):
    OK = 'ok'
    PENDING = 'pending'
    FAILED = 'failed'

class StatusBEnum(Enum):
    WAITING = 'waiting'
    CANCELLED = 'cancelled'
    READY = 'ready'

class SomeModelWithChoices(models.Model):
    status_A = EnumChoiceField(
        StatusAEnum,
        default=StatusAEnum.PENDING
    )
    status_B = EnumChoiceField(
        StatusBEnum,
        default=StatusBEnum.WAITING
    )
```

2 быстрые победы:

* **max\_length** вычисляется автоматически, принимая самое длинное значение.
* Нам не нужна утилита **get\_choices** каждый раз, когда нам нужно перебирать все значения перечисления.

## Собачья еда

Одна очень важная вещь, которую мы решили сделать для наших проектов с открытым исходным кодом, — это кормить их собачьей пищей.

Или другими словами – **используйте их как можно больше**. Это заставит нас исправлять ошибки и обеспечивать лучшую поддержку всего, что мы выпускаем.

В настоящее время мы прорабатываем наши проекты и интегрируем [django-enum-choices](https://github.com/HackSoftware/django-enum-choices), что на самом деле привело нас к решению нескольких очень интересных случаев.

## Технические детали

Как уже упоминалось, при разработке библиотеки мы столкнулись с интересными проблемами.

Василь Славов, проделавший большую часть работы над библиотекой, опубликует статью в блоге, в которой все будет объяснено более подробно.

А пока посмотрите [примеры в репозитории GitHub](https://github.com/HackSoftware/django-enum-choices), а также рассмотрите возможность использования этого в своих проектах.

Как всегда, отзывы приветствуются!

## Открытый источник

Как упоминалось в нашей предыдущей статье с открытым исходным кодом, мы хотим быть активными на всех трех фронтах:

* Поддержка библиотек с открытым исходным кодом, которые мы используем.
* Участие в библиотеках с открытым исходным кодом, которые мы используем.
* Создание открытого исходного кода из нашей повседневной работы.

Это шаг в сторону одного из фронтов. Продолжение следует.
