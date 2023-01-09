# Настройка Geodjango-GDAL для Windows 10

{% hint style="info" %}
Ссылка на оригинальную статью: [Geodjango-GDAL Setup Windows 10](https://www.pointsnorthgis.ca/blog/geodjango-gdal-setup-windows-10/)

Опубликовано: ???

Автор: [Points North GIS](https://www.pointsnorthgis.ca/)
{% endhint %}

## Вступление

[GeoDjango](https://docs.djangoproject.com/en/3.0/ref/contrib/gis/) — это геопространственное расширение [Django](https://www.djangoproject.com/). **GeoDjango** — это дополнение к мощной среде веб-разработки, которая позволяет интегрировать пространственные данные в веб-сайты и REST API. После того, как среда разработки настроена, этот мощный инструмент прост в освоении и использовании; однако заставить все работать в среде Windows 10 иногда может быть сложно. Если вы профессионал/энтузиаст ГИС, как и я, у вас, вероятно, есть несколько программ, использующих геопространственные библиотеки, такие как [GDAL](https://www.lfd.uci.edu/\~gohlke/pythonlibs/#gdal), которые периодически обновляются, чтобы не отставать от таких программ, как **QGIS**. Это может привести к проблемам при разработке с Django на том же компьютере, поскольку виртуальная среда Python может быть чувствительна к изменениям во внешних библиотеках ГИС. Следующее руководство расскажет нам, как настроить нашу среду разработки, чтобы мы могли начать работать с пространственными данными в GeoDjango/Django с включенными библиотеками Python **GDAL** и **OSGEO**. Надеемся, что этот процесс поможет новым пользователям быстро начать работу с GeoDjango и поможет существующим разработчикам GeoDjango выявить проблемные области, если ошибки библиотеки зависимостей начнут возникать после обновлений.

## Обзор

* Установка Python
* Установка OSGeo4W
* Настройка виртуальной среды Python
* Установка Джанго
* Установка GDAL Python
* Создание проекта Джанго
* BAT-файл GeoDjango
* Изменение Django settings.py
* Обновление файла Django GDAL (libgdal.py)
* Тестирование

## Шаг 1. Установка Python

**Django/GeoDjango** — это фреймворки, основанные на языке Python, а именно на [Python 3](https://www.python.org/download/releases/3.0/). Если у вас уже установлена версия Python 3, вы можете пропустить этот шаг, в противном случае установите последнюю стабильную версию Python 3 для Windows [здесь](https://www.python.org/downloads/windows/).

При установке Python я рекомендую устанавливать его с выбранной опцией установки для всех пользователей. Это упростит дальнейшие шаги по настройке переменных PATH в Windows 10. Если ваша учетная запись Windows не имеет прав администратора, вы сможете установить ее только для той учетной записи, с которой вы вошли в Windows. Вы по-прежнему можете установить и использовать Python, просто не забудьте указать место установки (например, `C:\Users\username\AppData\Local\Programs\Python\Python3X`).

Чтобы установить для всех пользователей, вам нужно будет выбрать опцию `"Customized installation"`. В следующем меню обязательно выберите **"pip", "tcl/tl и IDLE", "py launcher" и "for all users (requires elevation)"**. Нажмите "Next" и в следующем меню убедитесь, что выбраны **"Install for all users", "Add Python to environment variables" и "Precompile standard library"**.

<figure><img src="../../.gitbook/assets/install_python_1SQfSvo.gif" alt=""><figcaption></figcaption></figure>

## Шаг 2. Установка OSGeo4W

[OSGeo4W](https://www.osgeo.org/) или **Open Source Geospatial For Windows** — это менеджер пакетов ГИС, который загружает программы и инструменты ГИС с открытым исходным кодом и обеспечивает совместную работу всех программ по мере необходимости. Это самый простой способ загрузить библиотеки, необходимые для **GeoDjango**.

Загрузите установщик с [https://trac.osgeo.org/osgeo4w/](https://trac.osgeo.org/osgeo4w/) и запустите **программу установки OSGeo4W**. Выберите **Express Web-GIS Install**, нажмите "Next", выберите сайт для загрузки программ (данные размещаются на нескольких серверах, попробуйте выбрать географически ближайший к вам), выберите все опции в последующих меню и согласитесь с условия лицензий. Процесс установки может занять несколько минут, так как необходимо загрузить и установить несколько программ. Нажмите "Finish" после завершения процесса загрузки/установки.

<figure><img src="../../.gitbook/assets/install_osgeo4w.gif" alt=""><figcaption></figcaption></figure>

## Шаг 3. Настройка виртуальной среды Python

Виртуальная среда Python позволяет вам использовать только те библиотеки, которые вам нужны для конкретного приложения, что удобно для разработчиков и программистов, поскольку позволяет каждому проекту использовать разные версии одной и той же библиотеки и снижает количество проблем с обновлением и конфликтами.

Для простоты мы будем хранить нашу виртуальную среду в папке/каталоге с именем "dev", которая находится на верхнем уровне диска компьютера (т. е. `C:\dev`). Эта настройка требует, чтобы большинство операций выполнялось через командную строку Windows, поэтому простые пути позволяют уменьшить количество ошибок при вводе и запомнить меньше местоположений. В дальнейшем мы будем использовать командную строку для создания папок, установки зависимостей и программ.

### Шаг 3А. Создать каталог папки dev

Начните с открытия командной строки Windows, выполнив поиск "Command Prompt" или с помощью клавиши **Windows + R**, чтобы открыть приложение "Run", введите "cmd" в строку "Open:" и нажмите Enter.

В командной строке введите **cd \\**, чтобы перейти в папку верхнего уровня. Используйте команду **mkdir dev**, чтобы создать папку с именем "dev". Это место, где мы создадим виртуальную среду Python и запустим приложение GeoDjango. Теперь войдите в эту папку, набрав **cd dev**.

```powershell
# Windows Command Prompt
C:\Users\username>cd \
C:>mkdir dev
C:>cd dev
C:\dev>
```

### Шаг 3Б. Инициировать виртуальную среду

Теперь мы создадим **виртуальную среду Python** с именем "djangovenv", которая по сути является клоном базовой установки Python, в которой будут храниться все зависимости GeoDjango.

В командной строке, начиная с каталога "dev", мы проверим установку Python, введя `python --version`. Это скажет нам две вещи. Во-первых, появится сообщение об ошибке, если Python не был правильно установлен. Во-вторых, если установка работает, она сообщит вам установленную версию Python. Если у вас есть другие версии Python (например, Python 2.X), вам может потребоваться использовать другую команду для вызова python (например, `python3 --version`). Если это так, используйте эту команду для работы с python, пока мы не войдем в виртуальную среду.

Теперь запустите виртуальную среду, введя `python -m venv djangoenv`. Настройка среды может занять минуту. После завершения выполнения команды введите `djangovenv\Scripts\activate.bat`. "(djangovenv)" теперь должно отображаться в квадратных скобках в начале текущей командной строки, указывая на то, что ваша виртуальная среда теперь активна.

```powershell
# Windows Command Prompt
C:\dev>python --version
Python 3.8.4rc1
C:\dev>python -m venv djangoenv
C:\dev>djangovenv\Scripts\activate.bat
(djangovenv) C:\dev>
```

<figure><img src="../../.gitbook/assets/install_venv.gif" alt=""><figcaption></figcaption></figure>

## Шаг 4. Установка Джанго

Теперь, когда Python проверен на работоспособность, и у нас есть новая виртуальная среда, инициированная и активированная для GeoDjango, мы можем начать установку всех пакетов Python, необходимых для запуска GeoDjango, с помощью команды [pip](https://pypi.org/project/pip/), которая позволяет нам легко устанавливать пакеты из репозитория [индекса пакетов Python](https://pypi.org/).

На этом этапе у вас все еще должна быть активна виртуальная среда Python "djangovenv", если нет, обратитесь к предыдущему шагу для команды активации. Теперь мы собираемся установить пакет Django, используя следующую команду **pip**: `pip install django`. Теперь установщик pip загрузит и установит самую последнюю версию Django в виде пакета Python, который можно запустить из виртуальной среды "djangovenv". Вы можете запустить команду **pip list**, в которой пакеты установлены в виртуальной среде, и запустить **django-admin**, чтобы убедиться, что Django установлен.

```powershell
# Windows Command Prompt
(djangovenv) C:\dev>pip install django
Collecting django
  Downloading Django-3.0.8-py3-none-any.whl (7.5 MB)
     |████████████████████████████████| 7.5 MB 6.8 MB/s
Collecting asgiref~=3.2
  Downloading asgiref-3.2.10-py3-none-any.whl (19 kB)
Collecting sqlparse>=0.2.2
  Downloading sqlparse-0.3.1-py2.py3-none-any.whl (40 kB)
     |████████████████████████████████| 40 kB 2.5 MB/s
Collecting pytz
  Downloading pytz-2020.1-py2.py3-none-any.whl (510 kB)
     |████████████████████████████████| 510 kB 3.3 MB/s
Installing collected packages: asgiref, sqlparse, pytz, django
Successfully installed asgiref-3.2.10 django-3.0.8 pytz-2020.1 sqlparse-0.3.1
(djangovenv) C:\dev>pip list
Package    Version
---------- -------
asgiref    3.2.10
Django     3.0.8
pip        20.1.1
pytz       2020.1
setuptools 47.1.0
sqlparse   0.3.1
(djangovenv) C:\dev>django-admin
```

<figure><img src="../../.gitbook/assets/install_django.gif" alt=""><figcaption></figcaption></figure>

## Шаг 5. Установка GDAL Python

Теперь, когда виртуальная среда и Django готовы, мы можем добавить библиотеку абстракции геопространственных данных (**GDAL**) для Python. Самый последовательный способ сделать это — загрузить файл PIP Wheel (`.whl`) для вашей версии Python с сайта Кристофа Гольке [_Unofficial Windows Binaries for Python Extension Packages_](https://www.lfd.uci.edu/\~gohlke/pythonlibs/#gdal) (у вас может возникнуть соблазн использовать **pip install GDAL**; однако из моего прошлого это вряд ли сработает. Обязательно загрузите файл **.whl**, соответствующий вашей версии Python (например, **Python 3.7, Windows 64-Bit** => `GDAL‑3.1.2‑cp37‑cp37m‑win_amd64.whl`, `Python 3.8, Windows 64-Bit` => `GDAL‑3.1.2‑cp38‑cp38‑win_amd64.whl`). Для простоты переместите загруженный файл **.whl** в папку dev для упрощения доступа к командной строке и для дальнейшего использования. Используя командную строку с активированной виртуальной средой, используйте **pip** для установки **.whl:** `pip install GDAL-3.X.X-cp3X-cp3X-win_amd64.whl` или `pip install C:/path/to/GDAL‑3.X.X‑cp3X‑cp3X‑win_amd64.whl`

Теперь библиотеки GDAL должны быть доступны через командную строку виртуальной среды Python. Вы можете протестировать их, активировав терминал Python с помощью командной строки: **python**. Командная строка python теперь активна, проверьте, доступны ли библиотеки **gdal**, введя `import gdal; import ogr; exit()`. Если ошибок импорта нет, вы можете начать программировать с помощью GDAL python!

```powershell
# Используйте следующие команды для установки GDAL
# из загруженного файла .whl.
(djangovenv) C:\dev>dir
05/23/2020  10:42 AM    <DIR>          .
05/23/2020  10:42 AM    <DIR>          ..
05/23/2020  10:42 AM    <DIR>          djangovenv
05/23/2020  10:42 AM    <DIR>          GDAL‑3.1.2‑cp38‑cp38‑win_amd64.whl
(djangovenv) C:\dev>pip install GDAL‑3.1.2‑cp38‑cp38‑win_amd64.whl
Processing c:\dev\gdal-3.1.2-cp38-cp38-win_amd64.whl
Installing collected packages: GDAL
Successfully installed GDAL-3.1.2
(djangovenv) C:\dev>python
Python 3.8.4 (tags/v3.8.4:d7c567b08f, Mar 10 2020, 10:41:24) [MSC v.1900 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information
>>>import gdal
>>>import ogr
>>>exit()
(djangovenv) C:\dev>
```

## Шаг 6. Создание проекта Джанго

Теперь мы собираемся создать новый проект Django, который создаст файл **settings.py**, который нам нужно будет обновить, чтобы убедиться, что GDAL правильно работает с Django, и даст нам платформу для тестирования GDAL и Django в среде Django.

Запустить проект Django очень просто. При активной виртуальной среде Python "djangovenv" в командной строке введите `django-admin startproject geodjango`. Вы можете использовать любое название проекта вместо **geodjango**. Это создаст базовую папку проекта Django и структуру файлов, которые позволят вам начать создание приложения Django. Важно отметить, что на первом уровне проекта есть файл Python с именем **manage.py**. Активировав этот файл в виртуальной среде, мы можем давать команды приложению Django для разработки и тестирования. Теперь мы будем использовать командную строку **cd geodjango**, чтобы войти на верхний уровень этого каталога проекта.

```powershell
# Windows Command Prompt
# Запустить проект Джанго
(djangovenv) C:\dev>django-admin startproject geodjango
# Переместиться в директорию проекта
(djangovenv) C:\dev>cd geodjango
(djangovenv) C:\dev\geodjango>
```

## Шаг 7. BAT-файл GeoDjango

Прежде чем мы продолжим изменять код GeoDjango, нам нужно сообщить системе Windows, где получить доступ к программе **GDAL**, установив переменные **PATH**. В противном случае Django/Python не сможет найти правильное место для выполнения программ/двоичных файлов GDAL.

Мы собираемся создать файл **.bat**, который по сути представляет собой список командных подсказок, которые будут запускаться последовательно, вместо того, чтобы вводить их построчно. Файлы **BAT** хороши тем, что позволяют свести к минимуму ошибки ввода.

В нашем проекте/папке/каталоге **geodjango** (или имени созданного вами проекта **geodjango**) мы будем использовать файловый браузер Windows Explorer для перехода к папке `C:\dev\geodjango`, щелкните правой кнопкой мыши и создайте новый текстовый файл, который называется **geodjango\_setup**. Щелкните правой кнопкой мыши файл **geodjango.txt** и измените расширение файла на **.bat**. Следуйте [этим инструкциям](https://www.howtogeek.com/205086/beginner-how-to-make-windows-show-file-extensions/), если вы не знаете, как изменить или просмотреть расширения файлов ваших документов.

Откройте файл **geodjango\_setup.bat** в своем любимом текстовом редакторе, таком как встроенный в Windows Notepad или мой любимый [Notepad++](https://notepad-plus-plus.org/), и скопируйте фрагмент кода из [официальной документации GeoDjango](https://docs.djangoproject.com/en/3.0/ref/contrib/gis/install/#modify-windows-environment) или из приведенного ниже примера. Возможно, вам придется изменить строку 1 с ~~`set OSGEO4W_ROOT=C:\OSGeo4W`~~ на `set OSGEO4W_ROOT=C:\OSGeo4W64`, если у вас 64-разрядная система Windows. Вам также придется изменить версию Python в строке 2 с ~~`set PYTHON_ROOT=C:\Python3X`~~ на любую версию Python 3, которую вы установили, например, `set PYTHON_ROOT=C:\Python38` (см. анимацию ниже, где я нахожу версии **OSGEO** установлен и установлен Python, и соответствующим образом настройте параметры).

```powershell
set OSGEO4W_ROOT=C:\OSGeo4W
set PYTHON_ROOT=C:\Python3X
set GDAL_DATA=%OSGEO4W_ROOT%\share\gdal
set PROJ_LIB=%OSGEO4W_ROOT%\share\proj
set PATH=%PATH%;%PYTHON_ROOT%;%OSGEO4W_ROOT%\bin
reg ADD "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment" /v Path /t REG_EXPAND_SZ /f /d "%PATH%"
reg ADD "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment" /v GDAL_DATA /t REG_EXPAND_SZ /f /d "%GDAL_DATA%"
reg ADD "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment" /v PROJ_LIB /t REG_EXPAND_SZ /f /d "%PROJ_LIB%"
```

В моем примере вы можете видеть, что файл **.bat** отражает, что мой **OSGEO\_ROOT** — это место установки `C:\OSGeo4W64`, потому что он установлен как 64-разрядная версия, а мой **PYTHON\_ROOT** — это `C:\Program Files\Python38`, потому что установлен **Python 3.8.4**.

Теперь мы запустим файл **geodjango\_setup.bat**, дважды щелкнув его или щелкнув правой кнопкой мыши, а затем выбрав "run". Это займет всего секунду, таким образом обновив настройки **Windows PATH**, чтобы GeoDjango знал, где искать, чтобы заставить GDAL и Python работать вместе.

## Шаг 8. Изменение Django settings.py

В файле Django **settings.py** находятся все ключевые параметры приложения и среды для приложения Django. В этом примере файл будет расположен по адресу `C:\dev\geodjango\geodjango\settings.py`. Нам нужно будет добавить некоторый код в начало этого файла, чтобы указать Django, где искать наши файлы Python GDAL при запуске программы.

Откройте файл **settings.py** в текстовом редакторе (например, Notepad++) и добавьте следующий код после `BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))`

```python
# settings.py file
import os

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

# используйте это при настройке в Windows 10 с GDAL, установленным из OSGeo4W,
# с использованием значений по умолчанию
if os.name == 'nt':
    VIRTUAL_ENV_BASE = os.environ['VIRTUAL_ENV']
    os.environ['PATH'] = os.path.join(
        VIRTUAL_ENV_BASE, r'.\Lib\site-packages\osgeo'
    ) + ';' + os.environ['PATH']
    os.environ['PROJ_LIB'] = os.path.join(
        VIRTUAL_ENV_BASE, r'.\Lib\site-packages\osgeo\data\proj'
    ) + ';' + os.environ['PATH']

#[...] код settings.py продолжается
```

## Шаг 9. Обновление файла Django GDAL (libgdal.py)

Последний файл, который нам нужно изменить, чтобы привязки GDAL работали надежно, — это файл **libgdal.py**. Это один из основных файлов Django, поэтому мы должны быть осторожны!

По сути, здесь мы говорим Django искать правильный файл привязки библиотеки GDAL (**.dll**), который был упакован в файлы из GDAL **pip wheel**. Это позволяет Django использовать функции **GDAL**.

Используя текстовый редактор, мы собираемся изменить строку **23** файла Django **libgdal.py**, который находится в файлах виртуальной среды. В этом примере файл находится здесь: `C:\dev\djangovenv\Lib\site-packages\django\contrib\gis\gdal\libgdal.py`. Для версии Python, которую мы установили, мы добавим **"gdal300"** в начало массива **lib\_names** как таковой:

```python
# C:\dev\djangovenv\Lib\site-packages\django\contrib\gis\gdal\libgdal.py
# [...]
# Line 21
elif os.name == 'nt':
    # Общие библиотеки Windows NT
    lib_names = ['gdal300', 'gdal204', 'gdal203', 'gdal202', 'gdal201', 'gdal20']
```

Значение, которое мы добавили в этот массив, ссылается на файл **.dll**, это может измениться в разных версиях python/GDAL. Имя этого файла можно проверить, просмотрев файлы **.dll** в виртуальной среде здесь: `C:\dev\djangovenv\Lib\site-packages\osgeo`. В этом примере есть файл **.dll** с именем **gdal300.dll**. Django будет искать этот файл на основе приведенного выше массива и выдаст ошибку, если не сможет найти файл **.dll** из этого списка.

## Шаг 10. Тестирование

Наконец, последний шаг — убедиться, что Django может работать с GDAL. Убедитесь, что ваша виртуальная среда активирована и вы находитесь на корневом уровне вашего проекта Django (т. е. `C:\dev\geodjango`), а затем активируйте оболочку Django с помощью команды `python manage.py shell`. Затем введите команды в соответствии с приведенными ниже фрагментами.

```powershell
# windows command prompt
(djangovenv) C:\dev\geodjango>python manage.py shell
Python 3.8.4 (tags/v3.8.4:d7c567b08f, Mar 10 2020, 10:41:24) [MSC v.1900 64 bit (AMD64)] on win32
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>>import gdal
>>>import ogr
>>>exit()
(djangovenv) C:\dev\geodjango>
```

Если все вышеперечисленные импорты работают, вы готовы начать работу с Django и GDAL!

<figure><img src="../../.gitbook/assets/update_settings.gif" alt=""><figcaption></figcaption></figure>
