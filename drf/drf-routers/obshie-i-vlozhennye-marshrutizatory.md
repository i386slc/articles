# Общие и вложенные маршрутизаторы

{% hint style="info" %}
Ссылка на оригинальную статью: [General and Nested Routers](https://www.scaler.com/topics/django/general-and-nested-routers-in-django/)

Опубликовано: 6 апреля 2023

Автор: Nikhil Raj
{% endhint %}

## Обзор

Маршруты в Django определяются как путь, который сопоставляет URL-адрес с функцией просмотра для обработки входящего запроса. Во вложенных маршрутизаторах маршрут ресурса объявляет маршруты вашего индекса в одной строке кода, а не отдельно.

## Введение

Во время работы над проектом мы сопоставляем разные URL-адреса для разных функций просмотра.

Например, если мы строим API для системы управления школой, то у нас будут разные маршруты.

* `/api/students/` — получить список всех учеников
* `/api/students/{class}/` — получить информацию обо всех учениках определенного класса
* `/api/students/{class}/sections` — получить список всех разделов определенного класса
* `/api/students/{class}/sections/{section_id}` — получить сведения обо всех учениках из определенного раздела класса.

До этой части и даже дальше возможно сопоставление URL-адреса. Но если это масштабируется и нам нужно создать много вложенных маршрутизаторов, мы можем столкнуться с проблемами, а также ухудшится читаемость кода. В данном случае мы используем **Nested** router, который мы рассмотрим далее в статье.

## Применение

Существует несколько способов сопоставления или включения URL-адресов. Сначала мы можем использовать `SimpleRouter()` и добавить URL-адреса в существующий список. Например, для нашей системы управления школой мы хотим добавить URL-адреса для учащихся и классов, мы можем реализовать это, как показано ниже.

```python
from rest_framework import routers
from . import views

router = routers.SimpleRouter()
router.register(r'students', views.StudentsViews)
router.register(r'class', views.ClassViews)

urlpatterns = [
    path('add_school/', AddSchoolFormView.as_view()),
]

urlpatterns += router.urls
```

Другой метод заключается в включении URL-адресов, которые мы увидим в следующем разделе.

## Использование Include с маршрутизаторами

Мы можем просто включить `routers.urls` в существующий список шаблонов URL. Для этого нам сначала нужно импортировать включенный метод, написав следующий скрипт.

```python
from django.urls import include
```

Теперь мы можем просто использовать этот метод для включения URL-адресов, как показано в коде ниже.

```python
from rest_framework import routers
from . import views

router = routers.SimpleRouter()
router.register(r'students', views.StudentsViews)
router.register(r'class', views.ClassViews)

urlpatterns = [
    path('add_school/', AddSchoolFormView.as_view()),
    path('', include(router.urls)),
]
```

Мы также можем добавить пространство имен приложения и экземпляра, включая маршрутизаторы.

```python
urlpatterns = [
    path('add_school/', AddSchoolFormView.as_view()),
    path('api/', include((router.urls, 'app_name'), namespace='instance_name')),
]
```

## Маршрутизация дополнительных действий

Используя декоратор **@action** для указания метода, **viewset** может помечать дополнительные действия для маршрутизации. Созданные маршруты будут включать эти дополнительные шаги.

Чтобы реализовать это, сначала нам нужно импортировать декоратор **action**.

```python
from rest_framework.decorators import action
```

Теперь мы можем использовать этот декоратор для использования функции внутри класса в файле views.py. Например,

```python
class SchoolView(ModelViewSet):
    ...

    @action(methods=['post'], detail=True)
    # Функция, для которой вы хотите реализовать этот декоратор.
    ...
```

## Руководство по API

### SimpleRouter

Типичный набор действий - list, create, retrieve, update, partial-update и delete направляется через этот простой маршрутизатор.

По умолчанию **SimpleRouter** выглядит так:

```python
from rest_framework import routers

router = routers.SimpleRouter()
```

После каждого **SimpleRouter** добавляется косая черта, которую можно изменить, изменив значение на **false**.

```python
router = SimpleRouter(trailing_slash=False)
```

### DefaultRouter

В дополнение к корневому представлению API по умолчанию, которое создает ответ со ссылками на все представления списка, маршрутизатор **DefaultRouter** идентичен **SimpleRouter**, как указано выше.

В общем, **DefaultRouter** можно использовать как

```python
from rest_framework import routers

router = routers.DefaultRouter()
```

В отличие от **DefaultRouter**, при создании маршрутизатора параметр косой черты в конце может быть установлен в значение `False`, чтобы исключить косые черты в конце URL-маршрутов.

```python
router = DefaultRouter(trailing_slash=False)
```

## Пользовательские маршрутизаторы

В случае, если нам нужно выполнить определенные требования к структуре URL-адресов для API, пользовательские маршрутизаторы могут быть очень полезными в этом сценарии.

Атрибут `.routes` используется для паттернов шаблонов URL, а его атрибут представляет собой список именованных кортежей **Route**, аргументы которых

* **url** - строка, содержащая следующие строки формата, представляющие URL-адрес для пересылки.
  * **prefix** - префикс URL для этих маршрутов
  * **lookup** - поле поиска, которое использовалось для сравнения с одним конкретным экземпляром.
  * **trailing\_slash** - в зависимости от опции **trailing\_slash** будет возвращена либо `'/'`, либо пустая строка.
* **mapping** - словарь имен методов HTTP, соответствующих методам представления.
* **name** - определенное имя используется в обратном вызове, оно обычно включает в себя **basename** в качестве имени.
* **initkwargs** - словарь, содержащий любые дополнительные аргументы, которые необходимо указать при создании экземпляра представления.

### Настройка динамических маршрутов

Когда дело доходит до настройки динамических маршрутов, можно изменить даже маршрут декоратора **@action**. Именованный кортеж **DynamicRoute** может быть добавлен в список `.routes`, с аргументом **detail**, установленным в правильное значение как для маршрутов на основе **list**, так и для маршрутов на основе **detail**, чтобы реализовать это.

**DynamicRoute** принимает следующие аргументы

* **url** - работает так же, как и в **Route**, как показано выше, кроме того, он может включать **url\_path**.
* **name** - используется в обратных вызовах и включает **basename** и **url\_name**.
* **initkwargs** - словарь любых дополнительных аргументов, которые должны быть указаны при создании представления.

_Пример_. Ниже приведен пример реализации пользовательских и динамических маршрутов.

Для _пользовательских_ маршрутов мы видели, что у нас есть аргументы, такие как **url**, **mapping**, **name** и **details**, как показано в коде ниже.

```python
from rest_framework.routers import Route, SimpleRouter

class CustomRouter(SimpleRouter):

    routes = [
        Route(
            url=r'^{prefix}$',
            mapping={'get': 'list'},
            name='{basename}-list',
            detail=False,
            initkwargs={'suffix': 'List'}
        )
    ]
```

А для _динамических_ маршрутов мы можем сделать то же самое, как показано в коде ниже.

```python
from rest_framework.routers import Route, DynamicRoute

class CustomReadOnlyRouter(SimpleRouter):

    routes = [
        DynamicRoute(
            url=r'^{prefix}/{lookup}/{url_path}$',
            name='{basename}-{url_name}',
            detail=True,
            initkwargs={}
        )
    ]
```

### Расширенные пользовательские маршрутизаторы

Расширенные пользовательские маршрутизаторы позволяют нам переопределить методы, которые мы определили в предыдущем разделе.

## Сторонние пакеты

Некоторые сторонние пакеты могут использоваться для достижения цели вложенных маршрутизаторов DRF.

### Вложенные маршрутизаторы DRF

Сторонний пакет, целью которого является предоставление маршрутизаторов и полей для создания вложенных ресурсов в Django Rest Framework.

Сначала нам нужно установить из пакета **pip**

```bash
pip install drf-nested-routers
```

Чтобы использовать и реализовать это в нашем коде, нам нужно импортировать **routers** из **rest\_framework\_nested**, который является модулем **drf-nested-routers**.

Теперь этот маршрутизатор можно использовать для создания экземпляра **SimpleRouter** и регистрации.

```python
# urls.py
from rest_framework_nested import routers

router = routers.SimpleRouter()
router.register(...)
```

### ModelRouter (wq.db.rest) <a href="#modelrouter-wq-db-rest" id="modelrouter-wq-db-rest"></a>

Представляем **wq.db**, это набор модулей Python для создания надежных, адаптируемых схем и REST API, которые можно использовать для создания приложений для сбора данных полей с постепенными улучшениями.

Чтобы реализовать это, нам сначала нужно установить его из пакета **pip** как

```bash
python3 -m pip install wq
```

Затем мы можем использовать его, как показано в коде ниже.

```python
from wq.db import rest
from myapp.models import Model_Name

rest.router.register_model(Model_Name)
```

**Wq.db.rest** упрощает создание полнофункциональных API REST, которые могут служить целыми веб-страницами. Несмотря на наличие интерфейса REST только в названии, **wq.db.rest** предназначен для использования в качестве основного контроллера в веб-приложении, а не в качестве дополнительного API. Вам не нужно включать весь модуль для интеграции **wq.db.rest**, а только тот, который вы считаете полезным.

И затем мы можем использовать его для регистрации модели, соответствующей маршрутизатору, который мы хотели интегрировать.

### DRF-Extension

**DRF-Extension** — это еще один набор пользовательских расширений для Django REST Framework (DRF), которые можно использовать для вложенной маршрутизации (nested routers), а также для **DetailSerializerMixin**, **Caching**, **Conditional requests**, массовых операций (**Bulk operations**) и т. д.

Нам нужно сначала установить его

```bash
pip3 install drf-extensions
```

Пакет **DRF-extension** содержит некоторые функции, полезные для вложенных маршрутов, две наиболее часто используемые из них — это **ExtendedSimpleRouter**, который импортируется из `rest_framework_extensions.routers`, и **NestedViewSetMixin**, импортируемый из `rest_framework_extensions.mixins`.

Мы можем реализовать это в нашем коде, как показано ниже.

```python
from rest_framework_extensions.routers import ExtendedSimpleRouter

router = ExtendedSimpleRouter()
(...)
urlpatterns = router.urls
```

А для **NestedViewSetMixin** мы можем

```python
from rest_framework_extensions.mixins import NestedViewSetMixin

class ViewSet1(NestedViewSetMixin, ViewSet):
    model = model_name
```

## Дополнительно

### Гиперссылки для вложенных ресурсов

Вам нужен собственный сериализатор, если вам нужны гиперссылки для вложенных отношений, и если вы хотите немного больше контролировать поля, отображаемые для вложенных отношений при просмотре родителя, вам нужен собственный сериализатор, использующий **NestedHyperlinkedModelSerializer**, который импортируется из `rest_framework_nested.relations`.

Это может быть реализовано в коде, как показано ниже,

```python
from rest_framework_nested.relations import NestedHyperlinkedRelatedField

class DomainSerializer(HyperlinkedModelSerializer):
    class Meta:
        model = Domain

     nameservers = NestedHyperlinkedRelatedField(
        many=True,
        read_only=True,
        view_name='',
        parent_lookup_kwargs={}
```

### Вложенность бесконечной глубины

Для вложения маршрутизатора на нескольких уровнях мы можем использовать концепцию вложения бесконечной глубины, чтобы вкладывать маршрутизаторы настолько глубоко, насколько вам нужно.

Например, если мы хотим создать двухуровневые вложенные маршрутизаторы, как показано ниже

```
/path/
/path/{pk}/
/path/{client_pk}/other_path/
/path/{client_pk}/other_path/{pk}/
```

```python
router = DefaultRouter()
router.register(r'path', PathViewSet, basename='path')

client_router = routers.NestedSimpleRouter(router, r'path', lookup='path')
client_router.register(r'other_path', OtherPathViewSet, basename='other_path')
```

## Заключение

Завершим статью, резюмируя ее по некоторым пунктам.

* Используя концепцию вложенного маршрутизатора, мы можем легко создавать маршруты, такие как `/api/students/{class}/sections` и `/api/students/{class}/sections/{section_id}`, как описано в статье.
* Мы можем сопоставить URL-адрес с помощью `SimpleRouter()`, а в качестве альтернативы мы можем использовать **include** для включения URL-адреса.
* Кроме того, у нас есть корневое представление API по умолчанию, которое, как и **SimpleRouter**, возвращает ответ, содержащий ссылки на все представления списка.
* Для достижения конкретных требований мы можем использовать пользовательские маршрутизаторы, которые принимают **url**, **prefix**, **lookup** и **trailing\_slash**, что делает его более выполнимым, это также может быть изменено.
* В Django Rest Framework мы можем использовать сторонний пакет под названием **drf-nested-routers**, который предлагает маршрутизаторы и поля для создания вложенных ресурсов.
* Кроме того, у нас есть **ModelRouter** `wq.db.rest`, набор модулей Python, которые можно использовать для создания надежных, адаптируемых схем, и REST API, которые можно использовать для создания приложений для сбора данных полей, способных вносить небольшие корректировки.
* Другой набор уникальных расширений для Django REST Framework находится в пакете **DRF-extension** с именем `rest_framework_extensions.routers`, который можно использовать для вложенной маршрутизации, **DetailSerializerMixin**, кэширования, условных запросов, массовых операций и т. д.
* Мы используем **NestedHyperlinkedModelSerializer**, который импортируется из `rest_framework_nested.relations` и используется для создания гиперссылок для вложенных отношений.
* Мы можем использовать идею вложенности бесконечной глубины, чтобы вкладывать маршрутизаторы настолько глубоко, насколько это необходимо для многоуровневой вложенности.
