# Python: управление версиями с помощью Pyenv и Pyenv-Virtualenv (Linux)

{% hint style="info" %}
Ссылка на оригинальную статью: [Python: Version Management with Pyenv and Pyenv-Virtualenv (Linux)](https://medium.com/codex/python-version-management-with-pyenv-and-pyenv-virtualenv-linux-ecd6578b7bbf)

Опубликовано: 6 января 2022

Авторы: [Yuvrender Gill](https://medium.com/@yuvrendergill21?source=post\_page-----ecd6578b7bbf--------------------------------) (Я помогаю стартапам создавать передовые системы машинного обучения и обработки данных. Я верю во влияние через образование и технологии.)
{% endhint %}

[Pyenv](https://github.com/pyenv/pyenv) — один из самых крутых инструментов для управления несколькими версиями Python в вашей среде разработки. Будь то ваше новое предприятие по машинному обучению или полноценная инициатива по разработке продукта, вы найдете pyenv очень удобным для быстрой и эффективной настройки среды Python. Итак, давайте перейдем к делу.

## Настройка Pyenv и установка Python

Здесь я буду настраивать pyenv в подсистеме Linux под управлением Ubuntu 18.04 и устанавливать последнюю версию Python, то есть 3.10.1 (версия на дату создания).

### 1. Обновление системы

Если вы используете Ubuntu, это первый естественный шаг, который вам нужно сделать перед установкой чего-то нового. Скопируйте следующую команду в свою оболочку.

```bash
sudo apt update -y
```

sudo не является обязательным в зависимости от учетной записи и системных требований для доступа администратора.

### 2. Установка зависимостей

Pyenv требует следующих зависимостей, поэтому установите их, скопировав следующую команду в свою оболочку.

```bash
apt install -y make build-essential libssl-dev zlib1g-dev \
    libbz2-dev libreadline-dev libsqlite3-dev wget curl \
    llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev \
    libffi-dev liblzma-dev python-openssl git
```

### 3. Получение репозитория Git

Клонируйте следующий репозиторий. Вы получите последнюю версию pyenv с Github.

```bash
git clone https://github.com/pyenv/pyenv.git ~/.pyenv
```

### 4. Настройка переменных среды и среды

Используйте следующие команды для настройки переменных PYENV\_ROOT и PATH в файле .bashrc вашего Ubuntu. Во-вторых, настройте инициализацию pyenv для его инициализации. Запустите три команды в своей оболочке.

```bash
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc 
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc 
echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n eval "$(pyenv init -)"\nfi' >> ~/.bashrc
```

### 5. Перезапустить оболочку

Теперь вам нужно перезапустить оболочку с помощью следующей команды.

```bash
exec bash
```

или

```bash
exec "$SHELL"
```

Если вы используете zsh, используйте следующую команду.

```bash
exec zsh
```

### 6. Убедитесь, что Pyenv установлен

Проверьте, правильно ли установлен ваш pyenv, с помощью следующей команды `pyenv versions`, и соответствующий вывод будет выглядеть так, как показано ниже.

```bash
user@ubuntu:/mnt/c/Users/user$ pyenv versions
* system
```

Это показывает, что текущая версия Python выбрана по умолчанию.

### 7. Установка Python

Чтобы просмотреть все доступные версии Python для установки, используйте следующую команду.

```bash
pyenv install --list
```

Теперь выберите нужную версию и установите. Здесь я устанавливаю последнюю доступную версию. Выполнение этой команды обычно занимает много времени, поскольку выбранная версия Python собирается из исходного кода.

```bash
pyenv install 3.10.1
```

После завершения установки проверьте, выполнив следующий код.

```bash
user@ubuntu:/mnt/c/Users/user$ pyenv versions
* system (set by /mnt/c/Users/user/.python-version)
  3.10.1
```

Теперь ваш Python 3.10.1 готов к работе. Давайте теперь настроим pyenv-virtualenv.

## Установка Pyenv-virtualenv

Теперь мы будем устанавливать pyenv-virtualenv, который поможет нам настроить локальную среду разработки, не усложняя глобальную установку Python.

### 2. Клонировать репозиторий Git

Давайте сначала клонируем репозиторий virtualenv из Github.

```bash
git clone https://github.com/pyenv/pyenv-virtualenv.git $(pyenv root)/plugins/pyenv-virtualenv
```

### 2. Конфигурация среды

Выполните следующую команду, чтобы автоматически включать среды при запуске оболочки.

```bash
echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bashrc
```

### 3. Перезапустить оболочку

Теперь вам нужно перезапустить оболочку с помощью следующей команды.

```bash
exec bash
```

или

```bash
exec "$SHELL"
```

Если вы используете zsh, используйте следующую команду.

```bash
exec zsh
```

### 4. Проверка установки

Чтобы проверить, правильно ли установлен virtualenv, выполните следующую команду.

```bash
user@ubuntu:/mnt/c/Users/user$ pyenv virtualenv --version
pyenv-virtualenv 1.1.5 (python3.10 -m venv)
```

Теперь вы успешно установили pyenv-virualenv.

## Создание и использование Pyenv-Virtualenv

Теперь давайте посмотрим, как создать и использовать pyenv-virtualenv.

### 1. Создание виртуального окружения

Запустите следующую команду, чтобы создать новый virtualenv. Для создания среды нам необходимо предоставить версию Python из списка _**установленных**_ в системе Python. Список установленных версий Python доступен в `pyenv versions`. Второй аргумент — это имя виртуальной среды.

```bash
pyenv virtualenv 3.10.1 venv-name-3.10.1
```

Теперь, когда virtualenv создан, проверьте, правильно ли он создан, выполнив следующую команду.

```bash
user@ubuntu:/mnt/c/Users/user$ pyenv versions
* system (set by /mnt/c/Users/user/.python-version)
  3.10.1
  3.10.1/envs/venv-name  
  venv-name
```

### 2. Активация и деактивация Virtualenv

Чтобы активировать virtualenv, выполните следующую команду.

```bash
pyenv activate venv-name
```

Чтобы деактивировать, выполните следующую команду.

```bash
pyenv deactivate
```

Чтобы автоматически активировать локальную среду при входе в локальную папку разработки вашего проекта, выполните следующую команду внутри вашей папки.

```bash
pyenv local venv-name
```

Теперь это должно привести к следующему:

```bash
(venv-name) user@ubuntu:/mnt/c/Users/user/project-folder$
```

и теперь, если вы выйдете из текущей папки, вы получите следующее

```bash
(venv-name) user@ubuntu:/mnt/c/Users/user/project-folder$ cd..
user@ubuntu:/mnt/c/Users/user$
```

## Заключение

В этом уроке вы узнаете, как установить pyenv, различные версии Python и pyenv-virtualevn. Наслаждайтесь своим следующим проектом Python без каких-либо хлопот.

Я инженер данных с опытом работы со стеком GCP, SQL и Python. Если вам нужна помощь в настройке инфраструктуры данных для вашего стартапа или следующего бизнес-проекта, не стесняйтесь обращаться ко мне в мой [Linkedin](https://www.linkedin.com/in/yuvrender-gill/).
