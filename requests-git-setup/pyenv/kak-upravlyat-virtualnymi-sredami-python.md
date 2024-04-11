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
