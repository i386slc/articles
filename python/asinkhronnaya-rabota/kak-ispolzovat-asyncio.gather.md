# Как использовать asyncio.gather()

{% hint style="info" %}
Ссылка на оригинальную статью: [How to Use asyncio.gather() in Python](https://superfastpython.com/asyncio-gather/)

Опубликовано: 20 ноября 2022

Последнее обновление: 23 ноября 2022

Авторы:  [Jason Brownlee](https://superfastpython.com/about)
{% endhint %}

Нам нужны способы работы с наборами задач таким образом, чтобы их можно было рассматривать как группу.

Функция модуля asyncio.gather() предоставляет эту возможность и возвращает итерацию возвращаемых значений из ожидаемых запросов.

В этом руководстве вы узнаете, как ожидать выполнения задач asyncio одновременно с asyncio.gather().

После завершения этого урока вы будете знать:

* Что функция asyncio.gather() будет ждать завершения набора задач и получить все возвращаемые значения.
* Как использовать asyncio.gather() с коллекциями сопрограмм и коллекциями задач.
* Как использовать asyncio.gather() для создания вложенных групп задач, которые можно ожидать и отменять.

Давайте начнем.

## Оглавление

1. [Что такое Asyncio gather()](kak-ispolzovat-asyncio.gather.md#id-1.-chto-takoe-asyncio-gather)
2. [Как использовать Asyncio gather()](kak-ispolzovat-asyncio.gather.md#id-2.-kak-ispolzovat-asyncio-gather)
   * [gather() принимает задачи и сопрограммы](kak-ispolzovat-asyncio.gather.md#id-2.1.-gather-prinimaet-zadachi-i-soprogrammy)
   * [gather() возвращает будущее](kak-ispolzovat-asyncio.gather.md#id-2.2.-gather-vozvrashaet-budushee)
   * [В ожидании объекта Future метода gather()](kak-ispolzovat-asyncio.gather.md#id-2.3.-v-ozhidanii-obekta-future-metoda-gather)
   * [gather() может вкладывать группы ожидаемых объектов](kak-ispolzovat-asyncio.gather.md#id-2.4.-gather-mozhet-vkladyvat-gruppy-ozhidaemykh-obektov)
   * [gather() и исключения](kak-ispolzovat-asyncio.gather.md#id-2.5.-gather-i-isklyucheniya)
   * [объект Future метода gather() может быть отменен](kak-ispolzovat-asyncio.gather.md#id-2.6.-obekt-future-metoda-gather-mozhet-byt-otmenen)
3. [Примеры gather() с сопрограммами](kak-ispolzovat-asyncio.gather.md#id-3.-primery-gather-s-soprogrammami)
   * [Пример gather() для одной сопрограммы](kak-ispolzovat-asyncio.gather.md#id-3.1.-primer-gather-dlya-odnoi-soprogrammy)
   * [Пример gather() для множества сопрограмм](kak-ispolzovat-asyncio.gather.md#id-3.2.-primer-gather-dlya-mnozhestva-soprogramm)
   * [Пример gather() для множества сопрограмм в списке](kak-ispolzovat-asyncio.gather.md#id-3.3.-primer-gather-dlya-mnozhestva-soprogramm-v-spiske)
   * [Пример метода gather() с возвращаемыми значениями](kak-ispolzovat-asyncio.gather.md#id-3.4.-primer-metoda-gather-s-vozvrashaemymi-znacheniyami)
   * [Пример gather() с вложенными группами](kak-ispolzovat-asyncio.gather.md#id-3.5.-primer-gather-s-vlozhennymi-gruppami)
   * [Пример сочетания задач и сопрограмм с помощью метода gather()](kak-ispolzovat-asyncio.gather.md#id-3.6.-primer-sochetaniya-zadach-i-soprogramm-s-pomoshyu-metoda-gather)
4. [Примеры gather() с ошибками](kak-ispolzovat-asyncio.gather.md#id-4.-primery-gather-s-oshibkami)
   * [Пример метода gather(), где один ожидаемый метод не работает](kak-ispolzovat-asyncio.gather.md#id-4.1.-primer-metoda-gather-gde-odin-ozhidaemyi-metod-ne-rabotaet)
   * [Пример метода gather() с возвращением исключения](kak-ispolzovat-asyncio.gather.md#id-4.2.-primer-metoda-gather-s-vozvrasheniem-isklyucheniya)
   * [Пример метода gather(), в котором одна задача отменена](kak-ispolzovat-asyncio.gather.md#id-4.3.-primer-metoda-gather-v-kotorom-odna-zadacha-otmenena)
   * [Пример метода gather(), где одна задача отменяется с возвратом исключения](kak-ispolzovat-asyncio.gather.md#id-4.4.-primer-metoda-gather-gde-odna-zadacha-otmenyaetsya-s-vozvratom-isklyucheniya)
   * [Пример отмены всех задач в gather()](kak-ispolzovat-asyncio.gather.md#id-4.5.-primer-otmeny-vsekh-zadach-v-gather)
5. [Распространенные ошибки с asyncio.gather()](kak-ispolzovat-asyncio.gather.md#id-5.-rasprostranennye-oshibki-s-asyncio.gather)
   * [Пример ошибки gather() со списком](kak-ispolzovat-asyncio.gather.md#id-5.1.-primer-oshibki-gather-so-spiskom)
   * [Пример ошибки gather() без ожидания await](kak-ispolzovat-asyncio.gather.md#id-5.2.-primer-oshibki-gather-bez-ozhidaniya-await)
6. [Дальнейшее чтение](kak-ispolzovat-asyncio.gather.md#id-6.-dalneishee-chtenie)
7. [Заключение](kak-ispolzovat-asyncio.gather.md#id-7.-zaklyuchenie)



## 1. Что такое Asyncio gather()

[Функция модуля asyncio.gather()](https://docs.python.org/3/library/asyncio-task.html#asyncio.gather) позволяет вызывающей стороне группировать несколько ожидаемых объектов вместе.

После группировки ожидаемые объекты могут выполняться одновременно, ожидаться и отменяться.

> Запускайте ожидаемые объекты в последовательности aws одновременно.
>
> _—_ [_Coroutines and Tasks_](https://docs.python.org/3/library/asyncio-task.html)

Это полезная служебная функция как для группировки, так и для выполнения нескольких сопрограмм или нескольких задач.

Например:

```python
...
# запустить коллекцию ожидаемых объектов
results = await asyncio.gather(coro1(), asyncio.create_task(coro2()))
```

Мы можем использовать функцию asyncio.gather() в ситуациях, когда мы можем заранее создать множество задач или сопрограмм, а затем захотеть выполнить их все сразу и дождаться их завершения, прежде чем продолжить.

Это вероятная ситуация, когда результат требуется от многих подобных задач, например. одна и та же задача или сопрограмма с разными данными.

Ожидаемые объекты могут выполняться одновременно, возвращать результаты, а основная программа может возобновить работу, используя результаты, от которых она зависит.

Функция gather() является более мощной, чем просто ожидание завершения задач.

Это позволяет рассматривать группу ожидаемых объектов как один ожидаемый объект.

Это позволяет:

* Выполнение и ожидание выполнения всех ожидаемых объектов в группе с помощью выражения await.
* Получение результатов из всех сгруппированных ожидаемых объектов для последующего получения с помощью метода result().
* Группа ожидаемых объектов, которые будут отменены с помощью метода cancel().
* Проверка того, все ли ожидаемые объекты в группе выполнены с помощью метода done().
* Выполнение функций обратного вызова только тогда, когда все задачи в группе выполнены.

И более.

Функция asyncio.gather() аналогична asyncio.wait(). Она будет приостанавливаться до тех пор, пока все предоставленные задачи не будут выполнены, за исключением того, что он будет возвращать итерацию возвращаемых значений из всех задач, тогда как wait() по умолчанию не будет получать результаты задач.

Вы можете узнать больше о том, чем asyncio.gather() отличается от asyncio.wait(), в этом руководстве:

* [Asyncio gather() vs wait() in Python](https://superfastpython.com/asyncio-gather-vs-wait/)

Теперь, когда мы знаем, что такое функция asyncio.gather(), давайте посмотрим, как мы можем ее использовать.

Запускайте циклы, используя все процессоры. [Загрузите БЕСПЛАТНУЮ книгу](https://superfastpython.com/plip-incontent), чтобы узнать, как это сделать.

## 2. Как использовать Asyncio gather()

В этом разделе мы подробнее рассмотрим, как можно использовать функцию asyncio.gather().

### 2.1. gather() принимает задачи и сопрограммы

### 2.2. gather() возвращает будущее

### 2.3. В ожидании объекта Future метода gather()

### 2.4. gather() может вкладывать группы ожидаемых объектов

### 2.5. gather() и исключения

### 2.6. объект Future метода gather() может быть отменен

## 3. Примеры gather() с сопрограммами

### 3.1. Пример gather() для одной сопрограммы

### 3.2. Пример gather() для множества сопрограмм

### 3.3. Пример gather() для множества сопрограмм в списке

### 3.4. Пример метода gather() с возвращаемыми значениями

### 3.5. Пример gather() с вложенными группами

### 3.6. Пример сочетания задач и сопрограмм с помощью метода gather()

## 4. Примеры gather() с ошибками

### 4.1. Пример метода gather(), где один ожидаемый метод не работает

### 4.2. Пример метода gather() с возвращением исключения

### 4.3. Пример метода gather(), в котором одна задача отменена

### 4.4. Пример метода gather(), где одна задача отменяется с возвратом исключения

### 4.5. Пример отмены всех задач в gather()

## 5. Распространенные ошибки с asyncio.gather()

### 5.1. Пример ошибки gather() со списком

### 5.2. Пример ошибки gather() без ожидания await

## 6. Дальнейшее чтение

## 7. Заключение
