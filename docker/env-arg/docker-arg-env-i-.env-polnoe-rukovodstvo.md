# Docker ARG, ENV и .env — полное руководство

{% hint style="info" %}
Ссылка на оригинальную статью: [Docker ARG, ENV and .env - a Complete Guide](https://vsupalov.com/docker-arg-env-variable-guide/)

Опубликовано:

Автор:
{% endhint %}

<figure><img src="../../.gitbook/assets/1.png" alt=""><figcaption></figcaption></figure>

Создание образов Docker и настройка докеризированных приложений не обязательно должны быть феерией Google. Эта статья поможет вам уверенно работать с файлами Docker **ARG**, **ENV**, **env\_file** и `.env`.

> Единственное условие: убедитесь, что вы знакомы с [основами Docker](https://vsupalov.com/6-docker-basics/).

Читайте дальше, и вы поймете, как с легкостью настраивать образы Docker и докеризованные приложения — с помощью переменных во время сборки Docker, переменных среды и шаблонов для docker-compose.

## Частые заблуждения

Это долгое, глубокое чтение. Давайте начнем с того, что вы можете использовать прямо сейчас, без необходимости читать все!

Вот список **простых выводов**:

1. **Файл .env** используется только на этапе предварительной обработки при работе с файлами `docker-compose.yml`. Переменные в $-нотации, такие как `$HI`, заменяются значениями, содержащимися в именованном файле «`.env`» в том же каталоге.
2. **ARG** доступен **только во время сборки образа** Docker (**RUN** и т. д.), а не после создания образа и запуска из него контейнеров (**ENTRYPOINT**, **CMD**). Вы можете использовать значения **ARG**, чтобы установить значения **ENV**, чтобы [обойти это](https://vsupalov.com/docker-build-time-env-values/).
3. Значения **ENV** доступны для контейнеров, а также для команд в стиле **RUN** во время сборки Docker, начиная со строки, в которой они представлены.
4. Если вы установите переменную среды в **промежуточном контейнере** с помощью **bash** (`RUN export VARI=5 && …`), она **не сохранится** в следующей команде. Есть способ [обойти это](https://vsupalov.com/set-dynamic-environment-variable-during-docker-image-build/).
5. **env\_file** — это удобный способ передать **множество** переменных среды одной команде в одном пакете. Его не следует путать с файлом `.env`.
6. Установка значений **ARG** и **ENV** оставляет **следы** в образе Docker. Не используйте их для секретов, которые не предназначены для хранения (ну, вы можете использовать [многоэтапные сборки](https://vsupalov.com/build-docker-image-clone-private-repo-ssh-key/)).

## Обзор

Хорошо, давайте начнем с деталей. Руководство разбито на следующие темы:

* [Файл Dot-Env (.env)](docker-arg-env-i-.env-polnoe-rukovodstvo.md#fail-dot-env-.env)
* [Доступность ARG и ENV](docker-arg-env-i-.env-polnoe-rukovodstvo.md#dostupnost-arg-i-env)
* [Установка значений ARG](docker-arg-env-i-.env-polnoe-rukovodstvo.md#ustanovka-znachenii-arg)
* [Установка значений ENV](docker-arg-env-i-.env-polnoe-rukovodstvo.md#ustanovka-znachenii-env)
* [Переопределение значений ENV](docker-arg-env-i-.env-polnoe-rukovodstvo.md#pereopredelenie-znachenii-env)

Не стесняйтесь сразу перейти к тому, что вам нужно прямо сейчас. Тем не менее, вы получите наилучший результат, если внимательно прочитаете их все.

## Файл Dot-Env (.env)

Это довольно просто и сбивает с толку только из-за плохих примеров и похожих концепций, использующих тот же формат, что очень похоже на это. Важна точка перед **env** — `.env`, а не «**env\_file**».

Если в вашем проекте есть файл с именем [.env](https://docs.docker.com/compose/environment-variables/#the-env-file), он используется только для помещения значений в файл **docker-compose.yml**, который находится в той же папке. Они используются с **Docker Compose** и **Docker Stack**. Это не имеет никакого отношения к **ENV**, **ARG** или чему-либо специфичному для Docker, описанному выше. Это касается исключительно **docker-compose.yml**.

Значения в файле `.env` записываются в следующих обозначениях:

```ini
VARIABLE_NAME=some value
OTHER_VARIABLE_NAME=some other value, like 5
```

Эти пары ключ-значение используются для замены переменных долларового обозначения в файле **docker-compose.yml**. Это своего рода этап предварительной обработки, и используется полученный временный файл. Это хороший способ избежать жесткого кодирования значений. Вы также можете использовать это, чтобы установить значения для переменных среды, заменив строку, но это не происходит автоматически.

Вот пример файла `docker-compose.yml`, основанный на значениях, предоставленных из файла `.env`:

```yaml
version: '3'

services:
  plex:
    image: linuxserver/plex
      environment:
        - env_var_name=${VARIABLE_NAME} # вот оно
```

{% hint style="info" %}
При работе с файлом **.env** вы можете легко отлаживать файлы **docker-compose.yml**. Просто введите `docker-compose config`. Таким образом, вы увидите, как выглядит содержимое файла **docker-compose.yml** после выполнения шага подстановки без запуска чего-либо еще.
{% endhint %}

> Вот подсказка, которую вы должны знать: переменные среды на вашем хосте могут переопределять значения в вашем файле **.env**. Подробнее [здесь](https://vsupalov.com/override-docker-compose-dot-env/).

## Доступность ARG и ENV

При использовании Docker мы различаем два разных типа переменных — **ARG** и **ENV**.

**ARG** также известны как [переменные времени сборки](https://docs.docker.com/engine/reference/builder/#arg). Они доступны только с момента их «объявления» в **Dockerfile** с инструкцией **ARG** и до момента сборки образа. Запущенные контейнеры не могут получить доступ к значениям переменных **ARG**. Это также относится к инструкциям **CMD** и **ENTRYPOINT**, которые просто сообщают, **что** контейнер должен запускать по умолчанию. Если вы укажете **Dockerfile** ожидать различные переменные **ARG** (без значения по умолчанию), но при выполнении команды сборки **build** они не предоставляются, появится сообщение об ошибке.

Однако значения **ARG** можно легко проверить после создания образа, просмотрев историю Docker для образа `docker history`. Таким образом, они являются _**плохим выбором для конфиденциальных данных**_.

Переменные **ENV** также доступны во время сборки, как только вы вводите их с помощью инструкции **ENV**. Однако, в отличие от **ARG**, они также доступны контейнерам, запущенным из финального образа. Значения **ENV** можно переопределить при запуске контейнера, подробнее об этом ниже.

Вот упрощенный обзор возможностей **ARG** и **ENV** в процессе создания образа Docker из **Dockerfile** и запуска контейнера. Они перекрываются, но **ARG** нельзя использовать внутри контейнеров.

<figure><img src="../../.gitbook/assets/docker_environment_build_args.png" alt=""><figcaption><p>Обзор доступности ARG и ENV.</p></figcaption></figure>

## Установка значений ARG

Итак, у вас есть файл **Dockerfile**, определяющий значения **ARG** и **ENV**. Как их установить и где? Вы можете оставить их пустыми в **Dockerfile** или установить значения по умолчанию. Если вы не укажете значение для ожидаемых переменных **ARG**, у которых нет значения по умолчанию, вы получите сообщение об ошибке.

Вот пример **Dockerfile** как со значениями по умолчанию, так и без них:

```docker
ARG some_variable_name
# или с жестко заданным значением по умолчанию:
#ARG some_variable_name=default_value

RUN echo "Oh dang look at that $some_variable_name"
# вы также можете использовать фигурные скобки - ${some_variable_name}
```

[Соответствующие документы](https://docs.docker.com/engine/reference/builder/#arg).

При сборке образа Docker из командной строки вы можете установить значения **ARG**, используя `--build-arg`:

```bash
$ docker build --build-arg some_variable_name=a_value
```

Выполнение этой команды с указанным выше **Dockerfile** приведет к печати следующей строки (среди прочего):

```
Oh dang look at that a_value
```

Итак, как это перевести на использование файлов **docker-compose.yml**? При использовании **docker-compose** вы можете указать значения для передачи для **ARG** в блоке **args**:

(файл docker-compose.yml)

```yaml
version: '3'

services:
  somename:
    build:
      context: ./app
      dockerfile: Dockerfile
      args:
        some_variable_name: a_value
```

[Соответствующие документы](https://docs.docker.com/compose/compose-file/#args).

Когда вы пытаетесь установить переменную **ARG**, которая не упоминается в **Dockerfile**, Docker будет жаловаться.

## Установка значений ENV

Итак, как установить значения **ENV**? Вы можете сделать это при запуске ваших контейнеров (и мы рассмотрим это немного ниже), но вы также можете указать значения **ENV** по умолчанию непосредственно в вашем **Dockerfile**, жестко запрограммировав их. Кроме того, вы можете установить динамические значения по умолчанию для переменных среды!

При создании образа единственное, что вы можете предоставить, — это значения **ARG**, как описано выше. Вы не можете указать значения для переменных **ENV** напрямую. Однако и **ARG**, и **ENV** могут работать вместе. Вы можете использовать **ARG** для установки значений переменных **ENV** по умолчанию. Вот базовый файл **Dockerfile**, использующий жестко заданные значения по умолчанию:

```docker
# нет значения по умолчанию
ENV hey
# значение по умолчанию
ENV foo /bar
# или ENV foo=/bar

# значения ENV могут использоваться во время сборки
ADD . $foo
# или ADD . ${foo}
# переводится как: ADD . /bar
```

[Соответствующие документы](https://docs.docker.com/engine/reference/builder/#env).

А вот фрагмент файла **Dockerfile** с использованием динамических значений **env** при сборке:

```docker
# ожидает переменную времени сборки
ARG A_VARIABLE
# использует значение, чтобы установить переменную ENV по умолчанию
ENV an_env_var=$A_VARIABLE
# если не переопределить, это значение an_env_var будет доступно для ваших контейнеров!
```

**После создания образа** вы можете запускать контейнеры и указывать значения для переменных ENV **тремя различными способами**: либо из командной строки, либо с помощью файла docker-compose.yml. Все они переопределяют любые значения **ENV** по умолчанию в **Dockerfile**. В отличие от **ARG**, вы можете передавать в контейнер всевозможные переменные среды. Даже те, которые явно не определены в файле **Dockerfile**. Однако это зависит от вашего приложения.

### 1. Указать значения по одному

В командной строке используйте флаг `-e`:

```bash
$ docker run -e "env_var_name=another_value" alpine env
```

[Соответствующие документы](https://docs.docker.com/engine/reference/builder/#environment-replacement).

Из файла docker-compose.yml:

```yaml
version: '3'

services:
  plex:
    image: linuxserver/plex
      environment:
        - env_var_name=another_value
```

[Соответствующие документы](https://docs.docker.com/compose/compose-file/#environment).

### 2. Передайте значения переменных среды с вашего хоста

Это то же самое, что и вышеописанный метод. Единственная разница в том, что вы не предоставляете значение, а просто называете переменную. Это заставит Docker получить доступ к текущему значению в среде хоста и передать его контейнеру.

```bash
$ docker run -e env_var_name alpine env
```

Для файла **docker-compose.yml** пропустите знак уравнения и все, что после него, для того же эффекта.

```yaml
version: '3'

services:
  plex:
    image: linuxserver/plex
      environment:
        - env_var_name
```

### 3. Взять значения из файла (env\_file)

Вместо того, чтобы записывать переменные или жестко кодировать их (что не очень хорошо, по мнению [сторонников 12-фактора](https://12factor.net/)), мы можем указать файл, из которого будут считываться значения. Содержимое такого файла выглядит примерно так:

```ini
env_var_name=another_value
```

Файл выше называется **env\_file\_name** (имя произвольное) и находится в текущем каталоге. Вы можете сослаться на имя файла, которое анализируется для извлечения переменных среды для установки:

```bash
$ docker run --env-file=env_file_name alpine env
```

[Соответствующие документы](https://docs.docker.com/compose/compose-file/#env\_file).

В файлах **docker-compose.yml** мы просто ссылаемся на **env\_file**, и Docker анализирует его для установки переменных.

```yaml
version: '3'

services:
  plex:
    image: linuxserver/plex
    env_file: env_file_name
```

[Соответствующие документы](https://docs.docker.com/compose/compose-file/#env\_file).

Вот небольшая шпаргалка, объединяющая обзор доступности **ARG** и **ENV** с распространенными способами их установки из командной строки.

<figure><img src="../../.gitbook/assets/docker_environment_build_args_overview.png" alt=""><figcaption><p>Обзор доступности ARG и ENV.</p></figcaption></figure>

Прежде чем мы двинемся дальше: частая ошибка, если вы новичок в Docker и не привыкли думать об образах и контейнерах: если вы попытаетесь установить значение переменной среды **внутри оператора RUN**, такого как `RUN export VARI=5 && ...`, у вас **не будет доступа к нему** ни в одном из следующих операторов **RUN**. Причина этого в том, что для каждого оператора **RUN** новый контейнер запускается из промежуточного образа. Образ сохраняется в конце команды, но переменные среды **не сохраняются** таким образом.

Если вам интересно узнать об образе и вы хотите узнать, предоставляет ли оно значения переменных **ENV** по умолчанию до запуска контейнера, вы можете проверить образ и посмотреть, какие записи **ENV** установлены по умолчанию:

```bash
# сначала получите образы в вашей системе и их идентификаторы
$ docker images
# используйте один из этих идентификаторов, чтобы посмотреть подробности
$ docker inspect image-id

# обратите внимание на записи "Env"
```

Фу, это было совсем немного. Остается только одно: если у вас так много разных способов установки значений переменных **ENV**, какие из них переопределяют другие?

## Переопределение значений ENV

Предположим, у вас есть образ, созданный из **Dockerfile**, который предоставляет значения **ENV** по умолчанию. Контейнеры, запущенные из него, имеют доступ к переменным **ENV**, определенным в **Dockerfile**. Однако эти значения можно **переопределить**, предоставив отдельные переменные среды или **env\_files**, из которых переменные среды анализируются и передаются в контейнер.

Как только процесс запускается внутри контейнера или когда команда оценивается, они могут изменить значения среды для себя. Такие вещи, как:

```bash
$docker run myimage SOME_VAR=hi python app.py
```

полностью переопределяют любую **SOME\_VAR**, которую вы могли бы установить в противном случае для сценария `app.py`, даже если перед последней командой было какое-то значение с флагом `-e`.

Приоритет от **сильного** к менее сильному: заполнение контейнерных наборов приложений, значений из отдельных записей среды, значений из **env\_file** и, наконец, значений по умолчанию **Dockerfile**.

## В заключении

Это был действительно тщательный обзор всех способов установки переменных **ARG** и **ENV** при создании образов Docker и запуске контейнеров. К настоящему времени у вас должен быть действительно хороший обзор аргументов времени сборки, переменных среды, **env\_files** и шаблонов **docker-compose** с файлами **.env**. Я надеюсь, что вы извлекли из этого большую пользу и сможете использовать эти знания, чтобы избавить себя от множества ошибок в будущем.

Чтобы по-настоящему овладеть этими понятиями, недостаточно просто прочитать о них. Вы должны увидеть их в действии и применить в своей работе, чтобы они действительно стали частью вашего набора инструментов. Лучший способ убедиться, что вы сможете использовать эту информацию, — это учиться на практике — попробуйте некоторые из этих методов в своих текущих проектах!