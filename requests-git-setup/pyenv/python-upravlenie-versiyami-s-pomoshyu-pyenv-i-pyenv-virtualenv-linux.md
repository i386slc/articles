# Python: управление версиями с помощью Pyenv и Pyenv-Virtualenv (Linux)

{% hint style="info" %}
Ссылка на оригинальную статью: [Python: Version Management with Pyenv and Pyenv-Virtualenv (Linux)](https://medium.com/codex/python-version-management-with-pyenv-and-pyenv-virtualenv-linux-ecd6578b7bbf)

Опубликовано: 6 января 2022

Авторы: [Yuvrender Gill](https://medium.com/@yuvrendergill21?source=post\_page-----ecd6578b7bbf--------------------------------)
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
