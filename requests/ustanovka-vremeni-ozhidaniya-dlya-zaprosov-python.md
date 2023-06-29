# Установка времени ожидания для запросов Python

{% hint style="info" %}
Ссылка на оригинальную статью: [Setting Timeout for Python Requests](https://reqbin.com/code/python/3zdpeao1/python-requests-timeout-example)

Опубликовано: 27 марта 2023

Авторы: [ReqBin](https://reqbin.com/about)
{% endhint %}

Чтобы установить время ожидания для библиотеки запросов Python, вы можете передать параметр `"timeout"` для методов **GET**, **POST**, **PUT**, **HEAD** и **DELETE**. Параметр **timeout** позволяет выбрать максимальное время (в секундах) для выполнения запроса. По умолчанию запросы не имеют тайм-аута, если вы не укажете его явно. Рекомендуется устанавливать тайм-аут для запросов, чтобы избежать бесконечного ожидания зависших запросов. Если вы хотите дождаться завершения запроса независимо от того, сколько времени это займет, вы можете указать `None` для `"timeout"`. В этом примере тайм-аута запросов Python мы передаем тайм-аут для запроса, чтобы указать, как долго ждать ответа сервера. Нажмите «Execute», чтобы запустить пример тайм-аута запросов Python онлайн и увидеть результат (не используется в этом переводе).

```python
import requests

r = requests.get('https://reqbin.com/echo', timeout=5)

print(f"Status Code: {r.status_code}")
```

## Что такое Python?

[Python](https://reqbin.com/code/python) — это интерпретируемый язык программирования высокого уровня, известный своим чистым, удобочитаемым синтаксисом, гибкостью и простотой использования. Python — популярный язык для многих приложений, включая веб-разработку, научные вычисления, анализ данных, искусственный интеллект и т. д. Python поддерживает несколько парадигм программирования, таких как объектно-ориентированное, императивное и функциональное программирование. Python может работать на различных платформах, включая Windows, Linux и macOS.

## Что такое библиотека requests Python?

[Библиотека requests](https://requests.readthedocs.io/) — это распространенная библиотека, которая упрощает отправку [HTTP-запросов](https://reqbin.com/Article/HttpProtocol) с использованием методов [POST](https://reqbin.com/code/python/ighnykth/python-requests-post-example), [GET](https://reqbin.com/code/python/poyzx88o/python-requests-get-example) и [DELETE](https://reqbin.com/Article/HttpDelete). Библиотека requests основана на библиотеке urllib3 и скрывает сложность создания HTTP-запросов за простым [API](https://reqbin.com/req/zvxdp4hd/test-api-endpoint). Библиотека requests автоматически проверяет SSL-сертификаты сервера и поддерживает международные доменные имена и файлы cookie [сеанса](https://reqbin.com/code/python/9ooszjzg/python-requests-session-example). Библиотека requests не включена в дистрибутив Python, но все используют ее, потому что код Python для HTTP становится коротким, простым и понятным.

## Что такое тайм-аут в Python?

Тайм-аут в Python — это механизм, который ограничивает время, необходимое для выполнения операций в программе. В библиотеке requests Python тайм-ауты используются для определения времени ожидания ответа от сервера, прежде чем запрос будет считаться неудачным. Установка таймаутов важна при работе с сетевыми запросами, так как сервер может долго не отвечать на запрос, что может привести к зависанию приложения. В запросах Python вы можете установить тайм-ауты для каждого запроса и глобально для всех запросов в рамках сеанса. Установка тайм-аута позволяет программе продолжить выполнение, если операция не завершится в течение указанного времени. Тайм-ауты широко используются в различных библиотеках и модулях Python, таких как «[socket](https://socket.io/)», «[urllib](https://urllib3.readthedocs.io/)», «requests» и других. Они позволяют контролировать тайм-ауты на уровне кода, что повышает производительность и предсказуемость программы.

## Какие тайм-ауты можно установить для библиотеки requests Python?

Ниже приведены различные тайм-ауты, которые можно установить с помощью библиотеки запросов Python:

* **timeout**: общий тайм-аут для запроса, то есть время, в течение которого весь запрос должен быть завершен, включая установление соединения, передачу запроса и получение ответа. Значение этого тайм-аута можно установить, передав параметр `"timeout"` в функцию `requests.request()`:

```python
import requests

r = requests.get('https://reqbin.com/echo', timeout=5)

print(f"Status Code: {r.status_code}")
```

* **connect timeout**: тайм-аут для установления соединения с сервером. Это значение тайм-аута можно установить, передав параметр `"timeout"` в функцию `requests.request()` и установив ключ `"connect"` на желаемое значение тайм-аута:

```python
import requests

r = requests.get('https://reqbin.com/echo', timeout=(5, None))

print(f"Status Code: {r.status_code}")
```

* **read timeout**: тайм-аут ожидания ответа от сервера после установления соединения. Это значение тайм-аута можно установить, передав параметр `"timeout"` в функцию `requests.request()` и установив ключ `"read"` на желаемое значение тайм-аута:

```python
import requests

r = requests.get('https://reqbin.com/echo', timeout=(None, 5))

print(f"Status Code: {r.status_code}")
```

## Как использовать тайм-аут для запросов Python?

Вы можете использовать опцию `"timeout"`, чтобы установить тайм-аут для запросов Python. Параметр **timeout** указывает максимальное время ожидания ответа от сервера. Если в течение этого времени ответ не будет получен, программа выдаст исключение (**ConnectionError** и **ReadTimeout**). По умолчанию запросы не имеют времени ожидания, если вы не укажете его явно.

```python
import requests

try:
    response = requests.get('https://reqbin.com/echo', timeout=5)
    print(response.status_code)
except requests.exceptions.Timeout:
    print('The request timed out')
```

## Как установить таймауты подключения и чтения?

Чтобы установить тайм-аут соединения и чтения для HTTP-запроса в Python, вы можете передать кортеж, содержащий два значения, в параметр `"timeout"`, где первое значение — это тайм-аут соединения (в секундах), а второе значение — это тайм-аут чтения. (в секундах).

```python
import requests

try:
    response = requests.get('https://reqbin.com/echo', timeout = (3, 5))
    print(response.status_code)
except requests.exceptions.Timeout:
    print("The request timed out")
except requests.exceptions.RequestException as e:
    print("An error occurred:", e)
```

## Почему важно установить разумное значение времени ожидания?

Установка разумного значения времени ожидания в запросах Python зависит от конкретной задачи, которую выполняет ваша программа, а также от ваших требований к производительности и безопасности. Обычно рекомендуется устанавливать тайм-аут от 5 до 30 секунд для обычных HTTP-запросов. Если ваша программа обрабатывает большие объемы данных или выполняет сложные операции, вам может потребоваться установить более длительное время ожидания. Таким образом, использование тайм-аутов в запросах Python — отличная практика для обеспечения стабильности, безопасности и эффективности вашей программы.

## Смотрите также

* [How do I use session objects in Python Requests?](https://reqbin.com/code/python/9ooszjzg/python-requests-session-example)
* [How do I post JSON using the Python Requestslibrary?](https://reqbin.com/code/python/m2g4va4a/python-requests-post-json-example)
* [How do I send a POST request using Python RequestsLibrary?](https://reqbin.com/code/python/ighnykth/python-requests-post-example)
* [How do I send a GET request using Python RequestsLibrary?](https://reqbin.com/code/python/poyzx88o/python-requests-get-example)
* [How to send custom HTTP headers using the PythonRequests Library?](https://reqbin.com/code/python/jyhvwlgz/python-requests-headers-example)
