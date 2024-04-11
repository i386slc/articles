# Как управлять виртуальными средами Python

{% hint style="info" %}
Ссылка на оригинальную статью: [How To Manage Your Python Virtual Environments](https://medium.com/@adocquin/mastering-python-virtual-environments-with-pyenv-and-pyenv-virtualenv-c4e017c0b173)

Опубликовано: 27 сентября 2023

Авторы: [Avel Docquin](https://medium.com/@adocquin?source=post\_page-----c4e017c0b173--------------------------------)
{% endhint %}

Работая над проектами и блокнотами Python, вы, возможно, пытались запустить проект, требующий Python 3.6, но в вашей системе установлен только Python 3.8. Или запустить скрипт, требующий определенную версию пакета, которая конфликтует с другой ранее установленной версией.

Чтобы избежать этих проблем, вы можете использовать [pyenv](https://github.com/pyenv/pyenv) и [pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv). Pyenv позволит вам легко переключаться между несколькими версиями Python и pyenv-virtualenv для создания изолированных виртуальных сред.

Эти инструменты помогут вам избежать падения вашей системы, разрешить конфликты зависимостей и воспроизвести результаты анализа ваших коллег, создав выделенные среды со своими собственными пакетами для каждого проекта.

В этом руководстве вы сначала узнаете, как установить pyenv и pyenv-virtualenv на Mac и Ubuntu. Затем просмотрите команды, которые вы будете использовать чаще всего. Наконец, вы примените их в простом примере использования с помощью VS Code. Давайте начнем!

## 1. Установка Pyenv и pyenv-virtualenv

Установка pyenv и pyenv-virtualenv проста. Во-первых, в вашей системе должны быть установлены [git](https://git-scm.com/) и [curl](https://curl.se/).

Вы можете установить их в свой терминал, используя на Mac:

```bash
brew install git
brew install curl
```

Или на Ubuntu:

```bash
sudo apt install git
sudo apt install curl
```

Как только они у вас есть, все, что вам нужно сделать, это запустить:

```bash
curl https://pyenv.run | bash
```

И добавьте это в конец вашего файла `$HOME/.bashrc` или `$HOME/.zshrc`:

```bash
export PYENV_ROOT="$HOME/.pyenv"
command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

Перезагрузите терминал, и вы сможете запустить команду `pyenv`, которая выведет список возможных параметров.

<figure><img src="../../.gitbook/assets/pyenv-1.webp" alt=""><figcaption></figcaption></figure>

Мы также проверим, работает ли команда `pyenv virtualenv`. В этом случае он должен возвращать `no virtualenv name given`, поскольку для его создания необходимо указать имя среды.

<figure><img src="../../.gitbook/assets/pyenv-2.webp" alt=""><figcaption></figcaption></figure>

Установив pyenv и pyenv-virtualenv, мы можем перейти к следующей части!

## 2. Обзор команд

В этой части вы узнаете команды pyenv, которые будете использовать чаще всего:

* `pyenv install`: установка новой версии Python.
* `pyenv update`: обновление pyenv
* `pyenv virtualenv`: для создания новой виртуальной среды Python.
* `pyenv active`: для активации ранее созданной виртуальной среды.
* `source deactivate`: деактивировать виртуальную среду, используемую в данный момент.
* `pyenv uninstall`: Чтобы удалить виртуальную среду или версию Python.

### 2.1 Установите новую версию Python

Используя pyenv, вы можете установить новую версию Python, скажем, 3.11.3. Сначала вы можете ввести эту команду в своем терминале:

```bash
pyenv install -l
```

Он покажет все доступные версии Python.

<figure><img src="../../.gitbook/assets/pyenv-3.webp" alt=""><figcaption></figcaption></figure>

В списке вы увидите версию 3.11.3, которую можно установить с помощью:

```bash
pyenv install 3.11.3
```

<figure><img src="../../.gitbook/assets/pyenv-4.webp" alt=""><figcaption></figcaption></figure>

### 2.2 Обновление pyenv

Новые версии Python выпускаются регулярно. Если вы хотите установить последнюю версию, все, что вам нужно сделать, это запустить команду:

```bash
pyenv update
```

<figure><img src="../../.gitbook/assets/pyenv-5.webp" alt=""><figcaption></figcaption></figure>

Он обновит pyenv и список версий Python, доступных для установки.

### 2.3 Создайте виртуальную среду

После установки версии Python все, что вам нужно сделать для создания среды, — это ввести:

```bash
pyenv virtualenv <python_version> <environment_name>
```

Что в нашем случае может быть:

```bash
pyenv virtualenv 3.11.3 test_environment
```

<figure><img src="../../.gitbook/assets/pyenv-6.webp" alt=""><figcaption></figcaption></figure>

### 2.4 Активируйте виртуальную среду

Чтобы использовать существующую виртуальную среду, вы можете активировать ее, набрав:

```bash
pyenv activate <environment_name>
```

После активации каждый установленный пакет и запущенный скрипт Python будут находиться в этой среде.

<figure><img src="../../.gitbook/assets/pyenv-7.webp" alt=""><figcaption></figcaption></figure>

### 2.5 Деактивация виртуальной среды

Если вы хотите выйти из текущей виртуальной среды, вы можете деактивировать ее, введя команду:

```bash
source deactivate
```

### 2.6 Удаление среды или версии Python

Если вам больше не нужна среда или версия Python, вы можете удалить ее, используя:

```bash
pyenv uninstall <environment_name or python version>
```

<figure><img src="../../.gitbook/assets/pyenv-8.webp" alt=""><figcaption></figcaption></figure>

## 3. Простой вариант использования в VS Code
