---
title: Подготовка рабочего окружения Python
excerpt: 'Подготовка рабочего окружения Python.'
categories:
  - notes
tags:
  - code
  - python
toc: true
toc_label: "Getting Started"
---
Актуально для Ubuntu 22.04 с system python 3.10 
{: .notice--info}

Чтобы изолировать рабочие окружения я использую [pyenv](https://github.com/pyenv/pyenv){:target="_blank"} + [virtualenv](https://github.com/pypa/virtualenv){:target="_blank"} + [virtualenvwrapper](https://github.com/python-virtualenvwrapper/virtualenvwrapper){:target="_blank"}. Для работы с проектами добавляю [poetry](https://python-poetry.org/){:target="_blank"}. 

## Подготовка Ubuntu

Сперва ставлю недостающие пакеты и библиотеки:
```bash
sudo apt update
sudo apt install \
    python3-pip \
    build-essential \
    curl \
    libbz2-dev \
    libffi-dev \
    liblzma-dev \
    libncursesw5-dev \
    libreadline-dev \
    libsqlite3-dev \
    libssl-dev \
    libxml2-dev \
    libxmlsec1-dev \
    llvm \
    make \
    tk-dev \
    wget \
    xz-utils \
    zlib1g-dev
```

Это нужно для установки и безошибочной работы всех указанных выше инструментов.

## pyenv

Все дальнейшие действия выполняю из под пользователя, находясь в домашнем каталоге, root не использую.  
Устанавливаю pyenv:
```bash
curl https://pyenv.run | bash
```
Чтобы посмотреть доступные версии Python:
```bash
pyenv install --list
```
Для установки выбранной из списка версии (3.9.5 как пример):
```bash
pyenv install 3.9.5
```
После установки Python 3.9.5 будет доступен в директории `~/.pyenv/versions/3.9.5/bin`.  
Чтобы посмотреть установленные версии:
```bash
pyenv versions
```
Чтобы удалить установленный Python 3.9.5 (не удаляю, т.к. буду использовать):
```bash
pyenv uninstall 3.9.5
```
pyenv я использую только для установки дополнительных версий Python. Для работы с виртуальными окружениями мне нравятся virtualenv и virtualenvwrapper.

## virtualenv + virtualenvwrapper

Устанавливаю virtualenv:
```bash
pip install virtualenv --verbose
```
Поскольку virtualenv будет установлен в `~/.local/bin`, нужно убедиться, что путь добавлен в переменную PATH:
```bash
echo $PATH
```
Если нет, нужно добавить в `~/.profile`:
```bash
# set PATH so it includes user's private bin if it exists
if [ -d "$HOME/.local/bin" ] ; then
    PATH="$HOME/.local/bin:$PATH"
fi
```
Теперь перезапускаю оболочку:
```bash
exec bash
# или
source ~/.profile
```
В моем случае, после установки virtualenv, `~/.profile` уже был обновлен, осталось только загрузить новую конфигурацию.  
Для удобства управления виртуальными окружениями устанавливаю virtualenvwrapper:
```bash
pip install virtualenvwrapper
```
Во время установки virtualenvwrapper добавит в `~/.bashrc` следующую конфигурацию:
```bash
# virtualenvwrapper
export WORKON_HOME=$HOME/.virtualenvs
export PROJECT_HOME=$HOME/Devel
source /usr/local/bin/virtualenvwrapper.sh 
```
Поскольку я установил virtualenvwrapper из под обычного пользователя, нужно изменить расположение `virtualenvwrapper.sh` скрипта.
Чтобы найти скрипт:
```bash
which virtualenvwrapper.sh
```
Обновляю `~/.bashrc`:
```bash
# virtualenvwrapper
export WORKON_HOME=$HOME/.virtualenvs
export PROJECT_HOME=$HOME/Devel
# user - имя пользователя
source /home/user/.local/bin/virtualenvwrapper.sh
```
Перезапускаю оболочку:
```bash
exec bash
```
Если Python 3.9.5 был удален, предварительно его нужно установить:
```bash
pyenv install 3.9.5
```
Теперь, чтобы создать виртуальное окружение для установленного ранее Python 3.9.5:
```bash
# user - имя пользователя
mkvirtualenv -p /home/user/.pyenv/versions/3.9.5/bin/python3.9 3.9.5
```
Окружение будет установлено в `~/.virtualenvs` (согласно конфигурации `~/.bashrc`) и активировано:
- `3.9.5/bin/` – содержит скрипты для активации/деактивации окружения, интерпретатор Python, используемый в рамках данного окружения, менеджер pip и ещё несколько инструментов, обеспечивающих работу с пакетами Python.
- `3.9.5/lib/` – директория, содержащая библиотеки. Новые пакеты будут установлены в каталог `3.9.5/lib/python3.9/site-packages/`

Чтобы проверить доступные окружения:
```bash
workon
```
Чтобы переключаться между окружениями:
```bash
workon 3.9.5
```
Чтобы проверить работу виртуального окружения:
```bash
which pip
which python
python -V
```
Чтобы деактивировать действующее активное виртуальное окружение:
```bash
deactivate
```
Чтобы удалить виртуальное окружение:
```bash
rmvirtualenv 3.9.5
```

## poetry

Для работы с packages использую poetry. Устанавливаю глобально, в систему:
```bash
curl -sSL https://install.python-poetry.org | python3 -
# проверяю доступность poetry
poetry --version
```
Добавляю bash autocompletion для poetry:
```bash
poetry completions bash >> ~/.bash_completion
exec bash
```
Для примера создам новый проект в директории `~/test_poetry`:
```bash
poetry new test_poetry && cd test_poetry
```
Директория test_poetry будет выглядеть так:
```bash
tree
.
├── pyproject.toml
├── README.md
├── test_poetry
│   └── __init__.py
└── tests
    └── __init__.py
```
При создании проекта poetry использует Python-интерпретатор, который был использован при установке poetry. Это не зависит от активного в данный момент виртуального окружения. В конфигурационном файле `pyproject.toml` будет указан Python 3.10:
```bash
[tool.poetry.dependencies]
python = "^3.10"
```
Чтобы создать новый poetry-проект, зависящий от версии Python внутри виртуального окружения, нужно установить poetry в это окружение и создать проект:
```bash
# переключиться в созданное раньше виртуальное окружение с Python 3.9.5 (использую virtualenvwrapper)
workon 3.9.5

# установить poetry
pip install poetry

# создать новый poetry-проект
python -m poetry new test_poetry && cd test_poetry
```
Теперь в конфигурационном файле `pyproject.toml` будет Python 3.9:
```bash
[tool.poetry.dependencies]
python = "^3.9"
```
Чтобы деактивировать виртуальное окружение (использую virtualenv):
```bash
deactivate
```
Одна из задач poetry - изолировать Python-проекты. Для этого [poetry умеет управлять виртуальными окружениями](https://python-poetry.org/docs/managing-environments/){:target="_blank"}. Чтобы poetry создал виртуальное окружение для только что созданного проекта с `python = "^3.9"`:
```bash
# указать poetry интерпретатор Python и создать виртуальное окружение
poetry env use ~/.pyenv/versions/3.9.5/bin/python3.9
```
Созданное окружение разместится здесь - `~/.cache/pypoetry/virtualenvs/test-NP7pY1Pm-py3.9`. При создании виртуального окружения poetry опирается на конфигурационный файл `pyproject.toml`. Указанная при создании версия Python-интерпретатора должна соответствовать указанной в `pyproject.toml`, иначе poetry сообщит об ошибке. 

Poetry и virtualenvwrapper создают окружения в разных директориях. Для меня это ок, я использую связку pyenv + poetry для Python packages, а pyenv + virtualenv + virtualenvwrapper - для всего остального.

Чтобы установить пакет в новое виртуальное окружение и добавить в `pyproject.toml`, используется команда `poetry add`. Например, добавлю `anac = "^0.1.2"` (текущая версия в PyPI):
```bash
poetry add anac
```
Дополнительно `poetry add`, запущенная впервые, создаст файл `poetry.lock`, где зафиксирует версии packages и укажет зависимости. Например, кусок `poetry.lock` для `anac`:
```bash
[[package]]
name = "anac"
version = "0.1.2"
description = "Async NetBox API Client"
optional = false
python-versions = ">=3.8,<4.0"
...

[package.dependencies]
httpx = ">=0.22.0,<0.23.0"
pydantic = ">=1.9.0,<2.0.0"
```
Чтобы удалить пакет, используется команда `poetry remove`:
```bash
poetry remove anac
```
`poetry remove` удалит пакет из виртуального окружения, `pyproject.toml` и `poetry.lock`.

Чтобы посмотреть созданные poetry виртуальные окружения (команда покажет текущее активное виртуальное окружение):
```bash
poetry env list
```
Чтобы посмотреть информацию о текущем активном виртуальном окружении:
```bash
poetry env info
```
Чтобы удалить виртуальное окружение:
```bash
poetry env remove test-NP7pY1Pm-py3.9
```
Чтобы удалить все виртуальные окружения:
```bash
poetry env remove --all
```

Существует альтернативный способ изолировать Python-проект в виртуальном окружении. Он заключается в использовании poetry + pyenv:
```bash
# сконфигурировать poetry
poetry config virtualenvs.prefer-active-python true

# установить Python 3.9.5, если не был установлен раньше
pyenv install 3.9.5
# создать файл .python-version в директории с проектом
pyenv local 3.9.5
# создать виртуальное окружение и установить проект
poetry install
```
Этот отличный способ, чтобы быстро развернуть изолированное окружение для работы над уже существующим poetry-проектом со своими `pyproject.toml` и `poetry.lock` конфигурационными файлами.

Для управления poetry-проектом нужно использовать командный интерфейс poetry. Это необходимо, поскольку poetry не только вносит изменения в `pyproject.toml`, но также обновляет `poetry.lock` и виртуальное окружение. Хотя, не рекомендуется вносить изменения вручную в `pyproject.toml`, иногда я меняю версию Python сразу после создания нового проекта и до создания poetry виртуального окружения.

Poetry создает виртуальное окружение после указания Python-интерпретатора командой `poetry env use` или во время установки проекта, выполненной командой `poetry install`. При создании виртуального окружения poetry опирается на `pyproject.toml` и версию Python, указанную явно, с помощью `pyenv local` или `poetry env use`, иначе используется версия, которая использовалась при установке poetry.

### pyenv-virtualenv

Вместо связки pyenv + virtualenv + virtualenvwrapper, можно использовать pyenv + [pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv){:target="_blank"}. pyenv-virtualenv - это плагин, который поставляется в коробке с pyenv. Плагин хорош тем, что он не просто расширяет интерфейс pyenv возможностями virtualenv, а комбинируется с pyenv для полноценной работы с Python и виртуальными окружениями. Хотя, мне больше нравится идея использования исходных проектов, этот плагин нельзя обойти стороной.

pyenv-virtualenv устанавливается вместе с pyenv. Проверяю:
```bash
ls $(pyenv root)/plugins
```
Чтобы активировать плагин:
```bash
echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bashrc
exec "$SHELL"
```
Для примера установлю Python 3.9.8 и создам виртуальное окружение:
```bash
pyenv install 3.9.8

pyenv virtualenv pyvenv398
```
Чтобы посмотреть виртуальные окружения:
```bash
pyenv virtualenvs
```
Чтобы активировать виртуальное окружение:
```bash
pyenv activate pyvenv398
```
Чтобы деактивировать виртуальное окружение:
```bash
pyenv deactivate
```
Чтобы удалить виртуальное окружение:
```bash
pyenv uninstall pyvenv398
```

[Подробнее о pyenv-virtualenv](https://github.com/pyenv/pyenv-virtualenv/blob/master/README.md){:target="_blank"}.

## vim

Продолжая тему подготовки рабочего окружения для Python, оставлю линк на [инструкцию по настройке vim](https://timeforplanb123.github.io/code/vim-python/){:target="_blank"}.

## Полезные ссылки

- [раз](https://gist.github.com/wronk/a902185f5f8ed018263d828e1027009b){:target="_blank"}
- [два](https://pyneng.readthedocs.io/ru/latest/book/01_intro/virtualenv.html){:target="_blank"}
- [три](https://stackoverflow.com/questions/41573587/what-is-the-difference-between-venv-pyvenv-pyenv-virtualenv-virtualenvwrappe){:target="_blank"}
- [четыре](https://habr.com/ru/articles/740376/){:target="_blank"}
