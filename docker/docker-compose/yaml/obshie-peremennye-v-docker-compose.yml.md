# Общие переменные в docker-compose.yml

{% hint style="info" %}
Ссылка на оригинальную статью: [Shared variables in docker-compose.yml](https://rosengren.me/blog/shared-variables-in-docker-compose/)

Опубликовано: 10 сентября 2020

Автор:
{% endhint %}

Этот пост начался с того, что я пытался решить довольно простую проблему (изложенную ниже), но вместо этого попал в кроличью нору структуры и концепций YAML.

## Проблема

У меня есть настройка docker-compose для локального запуска приложения, она состоит из одной службы базы данных и одной службы API.

В основном это выглядит так:

```yaml
version: '3'

services:
  database:
    ...
    container_name: {static-name-of-our-database-container}
    environment:
      - ...
      - SA_PASSWORD={secret-password}
  api:
    ...
    depends_on:
      - database
    environment:
      - ...
      - DB_USER_PASSWORD={secret-password}
      - DB_HOST={static-name-of-our-database-container}
```

Вы, наверное, уже догадались, в чем проблема: я использую один и тот же буквальный пароль в двух местах и ​​использую имя контейнера базы данных в качестве хоста базы данных для API (также литералов). Для двух сервисов в надуманном примере это может быть и нормально, но для меня ввод одного и того же значения более одного раза означает проблемы в будущем, лучше исправить это сейчас!

## Решение

Я хочу иметь возможность объявить две переменные один раз и использовать их дважды.

Оказывается, мы можем сделать это разными способами, используя концепции YAML и docker-compose.

* YAML-якоря и псевдонимы: [https://yaml.org/spec/1.2/spec.html#id2765878](https://yaml.org/spec/1.2/spec.html#id2765878).
* Поля расширения docker-compose (требуется docker-compose.yml версии 3.4+): [https://docs.docker.com/compose/compose-file/#extension-fields](https://docs.docker.com/compose/compose-file/#extension-fields)

Давайте попробуем несколько сценариев, чтобы увидеть, как мы можем объединить эти концепции для решения различных проблем:

### Повторно используйте скалярное значение, но переименуйте сопоставление (проблема, указанная в начале).

В этом случае я бы сказал, что служба database «владеет» значениями как имени контейнера, так и пароля. Это означает, что мы можем объявить значения как привязки (обозначаемые символом `&`) в той же позиции, в которой они находятся сейчас, а затем вывести их как псевдоним (обозначаемый символом `*`).

Тогда наш файл `docker-compose.yml` будет выглядеть так:

```yaml
version: '3'

services:
  database:
    ...
    container_name: &database-container-name mycontainername
    environment:
      ...
      SA_PASSWORD: &sa-password reallysecretpassword
  api:
    ...
    depends_on:
      - database
    environment:
      ...
      DB_USER_PASSWORD: *sa-password
      DB_HOST: *database-container-name
```

В результате в службе api появятся следующие переменные среды:

```yaml
environment:
  DB_USER_PASSWORD: reallysecretpassword
  DB_HOST: mycontainername
```

Вы могли заметить, что я изменил синтаксис в обоих блоках сопоставлений environmant с использования синтаксиса `- {key}={value}` на синтаксис `{key}: {value}`. Это потому, что первый из них не имеет правильного синтаксиса, но работает, по крайней мере, до тех пор, пока мы не начали добавлять привязки и псевдонимы, после чего он перестает работать.

### Повторное использование всего блока/последовательности

Допустим, у нас есть приложение, в котором два клиента взаимодействуют с одним и тем же бэкэндом, тогда нам может потребоваться иметь одинаковые переменные среды в обоих из них, чтобы взаимодействовать с API.

В этом случае мы могли бы объявить якорь для всего блока:

```yaml
services:
  api:
    ...
    environment:
      &api-configuration
      hostname: {host}
      apikey: {apikey}
  client1:
    environment: *api-configuration
  client2:
    environment: *api-configuration
```

Здесь переменные среды для client1 и client2 будут именно теми, которые указаны в якоре `&api-configuration`.

В некоторых случаях это может быть полезно, но, вероятно, не в этом, поэтому давайте продолжим.

### Объединение спецификаций блоков/последовательностей

В приведенном выше примере мы ограничены тем, что переменные среды client1 и client2 должны быть точно такими же, как у api. Но что, если у client1 и client2 есть другие переменные, которых нет в api? А что, если эти переменные не являются общими для client1 и client2? Вместо этого давайте объединим переменные api environment с каждым соответствующим клиентом.

```yaml
services:
  api:
    ...
    environment:
      &api-configuration
      hostname: {host}
      apikey: {apikey}
  client1:
    environment: 
      <<: *api-configuration
      uniquekey: uniquevalue
  client2:
    environment:
      <<: *api-configuration
      uniquekey2: uniquevalue2
```

Здесь мы используем тип слияния `<<` для объединения `&api-configuration` с переменными environmant каждого клиента. Если вы хотите узнать точные правила слияния, ознакомьтесь со [Спецификацией](https://yaml.org/type/merge.html).

Результатом этого будет:

```yaml
services:
  api:
    ...
    environment:
      hostname: {host}
      apikey: {apikey}
  client1:
    environment: 
      hostname: {host}
      apikey: {apikey}
      uniquekey: uniquevalue
  client2:
    environment:
      hostname: {host}
      apikey: {apikey}
      uniquekey2: uniquevalue2
```

### Расширенные поля spec

Опираясь на приведенный выше пример, предположим, что наш api на самом деле не знает своего собственного имени хоста (по крайней мере, не в этой конфигурации). Как нам избежать попадания этой переменной в переменные environment?

В этом случае я бы разбил общие блоки на поля расширения (специфично для docker-compose, требуется версия docker-compose.yml 3.4+):

```yaml
x-common-extensions
  &common
  apikey: {apikey}

x-client-common-extensions
  &client-common
  <<: *common
  hostname: {host}

services:
  api:
    ...
    environment:
      <<: *common
  client1:
    environment: 
      <<: *client-common
      uniquekey: uniquevalue
  client2:
    environment:
      <<: *client-common
      uniquekey2: uniquevalue2
```

Теперь мы определили расширения `x-common-extensions` (с именем привязки `&common`) и расширения `x-client-common-extensions` (с именем привязки `&client-common`). Название самих расширений не имеет значения (нами не используется), за исключением того, что они должны начинаться с `x-`. Используя это, вы можете создать иерархию общих блоков. Если вы предпочитаете одноуровневую композицию иерархии, то вы можете достичь того же конечного результата, используя следующее:

```yaml
x-common-extensions
  &common
  apikey: {apikey}

x-client-common-extensions
  &client-common
  hostname: {host}

services:
  api:
    ...
    environment:
      <<: *common
  client1:
    environment: 
      <<: *common
      <<: *client-common
      uniquekey: uniquevalue
  client2:
    environment:
      <<: *common
      <<: *client-common
      uniquekey2: uniquevalue2
```

Разница здесь в том, что мы переместили слияние из `&common` в `&client-common` и вместо этого поместили несколько слияний в блоки среды клиентов.

Конечный результат будет тот же, а именно:

```yaml
services:
  api:
    ...
    environment:
      apikey: {apikey}
  client1:
    environment: 
      hostname: {host}
      apikey: {apikey}
      uniquekey: uniquevalue
  client2:
    environment:
      hostname: {host}
      apikey: {apikey}
      uniquekey2: uniquevalue2
```

### Скалярное значение как поле расширения

Допустим, мы все равно хотим, чтобы hostname было в переменных api environmant, но мы хотим, чтобы у него был другой ключ, как мы можем это сделать?

Один из способов — определить это значение как отдельное поле расширения, а не как часть блока, примерно так:

```yaml
x-hostname &hostname {host}

x-common-extensions
  &common
  apikey: {apikey}

x-client-common-extensions
  &client-common
  <<: *common
  hostname: *hostname

services:
  api:
    ...
    environment:
      <<: *common
      customhostnamekey: *hostname
  ...
```

С конечным результатом:

```yaml
services:
  api:
    ...
    environment:
      ...
      customhostnamekey: {host}
  client1:
    environment: 
      hostname: {host}
      ...
  client2:
    environment:
      hostname: {host}
      ...
```

## Заключение

Выяснить, как это сделать, было не так уж просто, потребовалось немного покопаться, особенно найти концепцию якорей/псевдонимов, надеюсь, это может помочь кому-то, пытающемуся достичь чего-то подобного.

Этот пост посвящен блоку окружающей среды. Но привязки/псевдонимы можно использовать в любом месте любого определения yaml (по крайней мере, те, которые анализируются парсером версии 1.2+), а поля расширения docker-compose можно использовать в любом месте определений вашего сервиса.

## Ресурсы, использованные при поиске этого решения

* [The first stackoverflow post that exposed me to the concepts](https://stackoverflow.com/questions/36283908/re-using-environmental-variables-in-docker-compose-yml)
* [Stack overflow that explains scalar anchors](https://stackoverflow.com/questions/39282485/can-i-anchor-individual-items-in-a-yaml-collection)
* [Very good post that explained most of this to me](https://medium.com/@kinghuang/docker-compose-anchors-aliases-extensions-a1e4105d70bd)
* [Extension fields specification](https://docs.docker.com/compose/compose-file/#extension-fields)
* [Good overview of yaml structure](https://www.tutorialspoint.com/yaml/yaml\_collections\_and\_structures.htm)
