# Асинхронные запросы в Python

{% hint style="info" %}
Ссылка на оригинальную статью: [Asynchronous Requests in Python](https://superfastpython.com/python-async-requests/)

Опубликовано: 12 декабря 2023

Авторы:  [Jason Brownlee](https://superfastpython.com/about)
{% endhint %}

Мы можем выполнять асинхронные запросы в Python.

Библиотека Requests Python не поддерживает asyncio напрямую.

Если мы сделаем HTTP-запросы с использованием библиотеки Requests, это заблокирует цикл событий asyncio и предотвратит выполнение всех других сопрограмм в программе.

Вместо этого мы можем выполнять асинхронные запросы, используя метод asyncio.to\_thread(), предоставленный в модуле asyncio стандартной библиотеки Python. Это запустит блокирующий сетевой вызов ввода-вывода в отдельном рабочем потоке, имитируя асинхронный или неблокирующий вызов функции ввода-вывода.

В этом руководстве вы узнаете, как безопасно использовать библиотеку Requests для HTTP-запросов в асинхронных программах.

Давайте начнем.

## Содержание

1. [Requests блокируют цикл событий Asyncio (это плохо)](asinkhronnye-zaprosy-v-python.md#requests-blokiruet-cikl-sobytii-asyncio-eto-plokho)
2. [Как использовать requests в Asyncio](asinkhronnye-zaprosy-v-python.md#kak-ispolzovat-requests-v-asyncio)
3. [Пример API requests (обычный Python)](asinkhronnye-zaprosy-v-python.md#primer-api-zaprosov-requests-obychnyi-python)
4. [Пример API requests в Asyncio (блокировка цикла событий)](asinkhronnye-zaprosy-v-python.md#primer-api-requests-v-asyncio-blokirovka-cikla-sobytii)
5. [Пример подтверждения запросов API, блокирующих цикл событий Asyncio](asinkhronnye-zaprosy-v-python.md#primer-podtverzhdeniya-zaprosov-api-requests-blokiruyushikh-cikl-sobytii-asyncio)
6. [Пример выполнения вызовов API запросов в отдельном потоке](asinkhronnye-zaprosy-v-python.md#primer-vypolneniya-vyzovov-api-requests-v-otdelnom-potoke)
7. [Дальнейшее чтение](asinkhronnye-zaprosy-v-python.md#dalneishee-chtenie)
8. [Заключение](asinkhronnye-zaprosy-v-python.md#zaklyuchenie)

## Requests блокирует цикл событий Asyncio (это плохо)

HTTP-клиенты составляют большую часть разработки Python.

Большинство API, которые мы используем для доступа к удаленным ресурсам, веб-сайтам и SaaS, будут скрытно использовать HTTP-запросы, обычно через интерфейс RESTful.

HTTP-запросы — это тип сетевого ввода-вывода, и мы можем захотеть разработать асинхронную программу, которая использует API, чтобы у нас было много сотен или тысяч одновременных клиентских HTTP-соединений.

Самая популярная библиотека Python для выполнения HTTP-запросов — это библиотека Requests.

[Requests](https://github.com/psf/requests): простая, но элегантная HTTP-библиотека.

> Requests позволяет очень легко отправлять запросы HTTP/1.1. Нет необходимости вручную добавлять строки запроса к вашим URL-адресам или кодировать данные PUT и POST с помощью формы — но в настоящее время просто используйте метод json!
>
> _—_ [_Requests GitHub Project_](https://github.com/psf/requests)_._

Проблема в том, что библиотеку Requests нельзя использовать напрямую в [асинхронных программах](https://superfastpython.com/python-asyncio/).

Причина в том, что библиотека Requests блокирует цикл событий asyncio при открытии, записи и чтении из соединения HTTP-сокета.

Это означает, что вся программа asyncio будет приостановлена до тех пор, пока HTTP-запрос не будет завершен.

Это плохо, потому что вся идея внедрения asyncio заключается в использовании асинхронного программирования для одновременного выполнения множества задач и выполнения неблокирующего сетевого ввода-вывода.

_**Как мы можем безопасно использовать запросы в асинхронных программах?**_

Выполняйте циклы, используя все процессоры. [Загрузите БЕСПЛАТНУЮ книгу](https://superfastpython.com/plip-incontent), чтобы узнать, как это сделать.

## Как использовать requests в Asyncio

Мы можем использовать библиотеку Requests для безопасного выполнения HTTP-запросов в асинхронных программах, запустив блокирующий вызов в новом потоке.

Этого можно добиться с помощью функции модуля [asyncio.to\_thread()](https://superfastpython.com/asyncio-to\_thread/).

asyncio.to\_thread() будет использовать ThreadPoolExecutor за кулисами для выполнения вызова в новом потоке одновременно с программой asyncio.

Нам нужно запустить блокирующий сетевой запрос в новом потоке, поскольку asyncio запускает все сопрограммы в одном потоке. Если одна сопрограмма в программе asyncio блокирует текущий поток, она блокирует все сопрограммы в потоке.

Выполняя блокирующий вызов в отдельном потоке, он позволяет циклу событий продолжать работу и выполнять все другие сопрограммы, работающие в программе.

Функцию модуля asyncio.to\_thread() можно рассматривать как неблокирующий вызов, что позволяет нам моделировать асинхронные запросы или неблокирующий сетевой ввод-вывод с помощью библиотеки requests.

Типичный HTTP-запрос GET, выполняемый с помощью библиотеки requests, выполняется с помощью функции `requests.get()`.

Например:

```python
...
# perform http get
result = requests.get('https://python.org/')
```

Это заблокирует цикл событий.

Мы можем сделать этот вызов в новом потоке, ожидая вызова asyncio.to\_thread().

Например:

```python
...
# perform http get
result = await asyncio.to_thread(requests.get, 'https://python.org/')
```

Обратите внимание, что asyncio.to\_thread() принимает имя метода или функции для вызова и каждый аргумент, который необходимо предоставить целевой функции.

Вы можете узнать больше о функции asyncio.to\_thread() в руководстве:

* [Как использовать Asyncio to\_thread()](https://superfastpython.com/asyncio-to\_thread/)

Вы можете узнать больше об обработке задач блокировки в асинхронных программах в руководстве:

* [Как запускать задачи блокировки в Asyncio](https://superfastpython.com/asyncio-blocking-tasks/)

Другим решением было бы использовать альтернативу библиотеке Requests, которая напрямую поддерживает asyncio, например, клиентская библиотека HTTP с асинхронной поддержкой. Популярные примеры включают [aiohttp](https://github.com/aio-libs/aiohttp) и [httpx](https://github.com/encode/httpx/).

Теперь, когда мы знаем, как безопасно использовать библиотеку Requests в asyncio, давайте посмотрим на некоторые рабочие примеры.

[Загрузите сейчас: бесплатную шпаргалку по Asyncio в формате PDF](https://marvelous-writer-6152.ck.page/d29b7d8dfb)

## Пример API запросов requests (обычный Python)

Во-первых, давайте посмотрим, как создать простой HTTP-запрос GET с использованием библиотеки Requests.

В приведенном ниже примере выполняется запрос и сообщается статус HTTP и длина контента.

```python
# SuperFastPython.com
# пример http-клиента с запросами
import requests
# определите URL-адрес, который мы хотим получить
url = 'https://python.org/'
# выполнить HTTP GET
result = requests.get(url)
# сообщить о результатах
print(f'Status Code: {result.status_code}')
print(f'Content Length: {len(result.text)}')
```

Запуск программы выполняет запрос и сообщает код состояния и длину всего загруженного текста.

Мы видим, что это очень чистый и простой API.

HTTP GET — это блокирующий вызов, который открывает сетевое соединение, отправляет HTTP-запрос и загружает HTTP-ответ, а затем анализирует ответ на структуры, к которым мы можем получить доступ.

```bash
Status Code: 200
Content Length: 51225
```

## Пример API requests в Asyncio (блокировка цикла событий)

Мы можем вызвать `request.get()` в асинхронных программах, но это заблокирует цикл событий, как мы уже обсуждали.

В приведенном ниже примере обновляется пример для запуска в цикле событий asyncio.

```python
# SuperFastPython.com
# пример http-клиента с запросами в asyncio
import requests
import asyncio
 
# основная корутина
async def main():
    # определите URL-адрес, который мы хотим получить
    url = 'https://python.org/'
    # выполнить блокирующий http-запрос GET
    result = requests.get(url)
    # сообщить о результатах
    print(f'Status Code: {result.status_code}')
    print(f'Content Length: {len(result.text)}')
 
# запустить цикл событий
asyncio.run(main())
```

Пример работает, в чем проблема?

Проблема в том, что когда мы вызываем `request.get()`, поток блокируется.

Это означает, что любые другие сопрограммы, которые мы можем запустить в нашем цикле событий asyncio, **не будут выполняться**.

Блокировка цикла событий является антишаблоном в асинхронных программах. Плохая практика. Противоречиво. Мы используем asyncio, чтобы наши задачи могли взаимодействовать и выполняться одновременно.

```bash
Status Code: 200
Content Length: 51225
```

Перегружены API-интерфейсами параллелизма Python?

Найдите облегчение, загрузите мои БЕСПЛАТНЫЕ [ментальные карты Python Concurrency](https://marvelous-writer-6152.ck.page/8f23adb076).

## Пример подтверждения запросов API requests, блокирующих цикл событий Asyncio

Давайте сделаем это очевидным, запустив еще одну сопрограмму в фоновом режиме.

Мы можем определить сопрограмму, которая печатает сообщение и переходит в режим ожидания каждые 10 миллисекунд.

```python
# какая-то фоновая задача
async def background():
    while True:
        print('Running in the background...')
        await asyncio.sleep(0.01)
```

Мы можем сначала запустить эту задачу и позволить ей начать работу немедленно.

```python
...
# создать фоновую задачу
task = asyncio.create_task(background())
# разрешить фоновой задаче начать выполнение
await asyncio.sleep(0)
```

Затем мы можем выполнить наш запрос и вывести сообщение прямо перед тем, как заблокировать поток с нашим запросом.

```python
...
# выполнить блокирующий http-запрос на получение в отдельном потоке
print('Starting request')
result = requests.get(url)
```

Полный пример приведен ниже.

```python
# SuperFastPython.com
# пример http-клиента с запросами в asyncio
import requests
import asyncio
 
# какая-то фоновая задача
async def background():
    while True:
        print('Running in the background...')
        await asyncio.sleep(0.01)
 
# основная корутина
async def main():
    # создать фоновую задачу
    task = asyncio.create_task(background())
    # разрешить фоновой задаче начать выполнение
    await asyncio.sleep(0)
    # определите URL-адрес, который мы хотим получить
    url = 'https://python.org/'
    # выполнить блокирующий http-запрос GET в отдельном потоке
    print('Starting request')
    result = requests.get(url)
    # сообщить о результатах
    print(f'Status Code: {result.status_code}')
    print(f'Content Length: {len(result.text)}')
 
# запустить цикл событий
asyncio.run(main())
```

Запуск примера сначала запускает нашу фоновую задачу и позволяет ей запуститься.

Фоновая задача получает одну возможность запуститься перед сном.

Сопрограмма main() возобновляет работу, печатает сообщение и выполняет запрос HTTP GET.

Это занимает некоторое время и блокирует цикл событий asyncio на это время. Наша фоновая задача не может выполняться и печатать сообщения.

GET завершается, сопрограмма main() сообщает о состоянии, завершает работу и отменяет фоновую задачу, поскольку цикл обработки событий завершается.

Мы должны выполнить блокирующий вызов таким образом, чтобы не блокировать цикл событий.

```bash
Running in the background...
Starting request
Status Code: 200
Content Length: 51225
```

## Пример выполнения вызовов API requests в отдельном потоке

Мы можем выполнять блокировку вызовов Requests API в отдельном потоке наших асинхронных программ.

Этого можно достичь с помощью функции `asyncio.to_thread()`, которая будет скрыто использовать пул рабочих потоков для выполнения задач блокировки.

Затем мы можем дождаться вызова, как и любую другую сопрограмму.

Например:

```python
...
# выполнить блокирующий http-запрос на получение в отдельном потоке
result = await asyncio.to_thread(requests.get, url)
```

Обновленная версия нашей программы с этим изменением представлена ниже.

```python
# SuperFastPython.com
# пример http-клиента с запросами в asyncio
import requests
import asyncio
 
# какая-то фоновая задача
async def background():
    while True:
        print('Running in the background...')
        await asyncio.sleep(0.01)
 
# основная корутина
async def main():
    # создать фоновую задачу
    task = asyncio.create_task(background())
    # allow the background task to start executing
    await asyncio.sleep(0)
    # определите URL-адрес, который мы хотим получить
    url = 'https://python.org/'
    # выполнить блокирующий http-запрос GET в отдельном потоке
    print('Starting request')
    result = await asyncio.to_thread(requests.get, url)
    # сообщить о результатах
    print(f'Status Code: {result.status_code}')
    print(f'Content Length: {len(result.text)}')
 
# запустить цикл событий
asyncio.run(main())
```

Программа работает так же, как и раньше, с одним важным отличием.

Это не блокирует текущий поток.

Фоновая задача создается и запускается, запускается один раз перед возобновлением работы сопрограммы `main()`.

Сопрограмма `main()` затем выводит сообщение “Starting request”, а затем приостанавливается, поскольку она выполняет запрос HTTP GET в отдельном потоке.

Это позволяет всем остальным сопрограммам в цикле событий работать. У нас работает только еще одна сопрограмма — наша фоновая задача, которая запускается несколько раз в своем цикле и радостно выводит сообщения.

Запрос GET в конечном итоге завершается, а сопрограмма `main()` возобновляет работу и сообщает подробности перед завершением цикла обработки событий.

Во-первых, это показывает, как мы можем запускать блокировку HTTP-вызовов в библиотеке Requests в отдельном потоке наших асинхронных программ. Во-вторых, это показывает, как это позволяет циклу событий выполнять работу других сопрограмм, пока выполняется блокирующий вызов.

```bash
Running in the background...
Starting request
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Running in the background...
Status Code: 200
Content Length: 51435
```

Это работает, однако есть и другие способы.

Мы можем использовать асинхронную библиотеку для выполнения наших HTTP-запросов.

Это может быть предпочтительнее, если мы не хотим разворачивать пул рабочих потоков в качестве обходного пути для работы с синхронными сетевыми вызовами ввода-вывода.

## Дальнейшее чтение

В этом разделе представлены дополнительные ресурсы, которые могут оказаться вам полезными.

Книги по Python Asyncio

* [Python Asyncio Mastery](https://superfastpython.com/pam-further-reading), Jason Brownlee (_**my book!**_)
* [Python Asyncio Jump-Start](https://superfastpython.com/paj-further-reading), Jason Brownlee.
* [Python Asyncio Interview Questions](https://superfastpython.com/python-asyncio-interview-questions/), Jason Brownlee.
* [Asyncio Module API Cheat Sheet](https://marvelous-writer-6152.ck.page/d29b7d8dfb)

Также рекомендую следующие книги:

* [Python Concurrency with asyncio](https://amzn.to/3LZvxNn), Matthew Fowler, 2022.
* [Using Asyncio in Python](https://amzn.to/3lNp2ml), Caleb Hattingh, 2020.
* [asyncio Recipes](https://amzn.to/47oN8dk), Mohamed Mustapha Tahrioui, 2019.

Путеводители

* [Python Asyncio: The Complete Guide](https://superfastpython.com/python-asyncio/)
* [Python Asynchronous Programming](https://superfastpython.com/python-asynchronous-programming/)

API

* [asyncio — Asynchronous I/O](https://docs.python.org/3/library/asyncio.html)
* [Asyncio Coroutines and Tasks](https://docs.python.org/3/library/asyncio-task.html)
* [Asyncio Streams](https://docs.python.org/3/library/asyncio-stream.html)
* [Asyncio Subprocesses](https://docs.python.org/3/library/asyncio-subprocess.html)
* [Asyncio Queues](https://docs.python.org/3/library/asyncio-queue.html)
* [Asyncio Synchronization Primitives](https://docs.python.org/3/library/asyncio-sync.html)

Ссылки

* [Asynchronous I/O, Wikipedia](https://en.wikipedia.org/wiki/Asynchronous\_I/O).
* [Coroutine, Wikipedia](https://en.wikipedia.org/wiki/Coroutine).

## Заключение

Теперь вы знаете, как использовать библиотеку Requests в asyncio.

Я допустил ошибку? Видите опечатку? Я простой скромный человек. Поправьте меня, пожалуйста!

Есть ли у вас дополнительные советы? Я хотел бы услышать о них!

Есть вопросы? Задавайте свои вопросы в комментариях ниже, и я постараюсь ответить.
