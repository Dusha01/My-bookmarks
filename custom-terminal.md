# 🐧 Первоначальная настройка терминала Linux

Краткий чеклист комфортной работы в терминале: редактор **nano**, приглашение **bash**, алиасы, история команд и **gitconfig**.

---

## 📑 Содержание

1. [✏️ Настройка nano](#1--настройка-nano)
2. [🌿 Настройка bashrc (Git prompt)](#2--настройка-bashrc-git-prompt)
3. [⚡ Алиасы + история bash](#3--алиасы--история-bash)
4. [🔧 Настройка gitconfig](#4--настройка-gitconfig)

---

## 1. ✏️ Настройка nano

Файл конфигурации:

```bash
nano ~/.nanorc
```

### 📄 Конфиг `~/.nanorc`

```nanorc
set linenumbers
set tabsize 4
set autoindent
set tabstospaces
#set smooth
#set mouse
set positionlog
#set rebinddelete
#set rebindkeypad
set softwrap
set atblanks
set indicator
set titlecolor white,black
set statuscolor brightcyan,black
set errorcolor brightred,white
set selectedcolor black,cyan
set numbercolor green,black
```

### 📋 Что делает каждая опция

| Опция | Статус | Описание |
|-------|--------|----------|
| `linenumbers` | ✅ | Номера строк слева |
| `tabsize 4` | ✅ | Ширина таба — 4 пробела |
| `autoindent` | ✅ | Автоотступ как у предыдущей строки |
| `tabstospaces` | ✅ | Tab вставляется пробелами |
| `smooth` | ⏸️ | Плавный скролл (закомментировано) |
| `mouse` | ⏸️ | Поддержка мыши (закомментировано) |
| `positionlog` | ✅ | Запоминает позицию курсора при повторном открытии |
| `rebinddelete` / `rebindkeypad` | ⏸️ | Альтернативная привязка Delete / keypad |
| `softwrap` | ✅ | Мягкий перенос длинных строк |
| `atblanks` | ✅ | Перенос по пробелам (удобнее читать) |
| `indicator` | ✅ | Индикатор положения в файле |
| `titlecolor` | 🎨 | Заголовок: белый на чёрном |
| `statuscolor` | 🎨 | Статус-бар: яркий cyan на чёрном |
| `errorcolor` | 🎨 | Ошибки: яркий красный на белом |
| `selectedcolor` | 🎨 | Выделение: чёрный на cyan |
| `numbercolor` | 🎨 | Номера строк: зелёный на чёрном |

### ✅ Применить

Сохраните файл (`Ctrl+O`, `Enter`, `Ctrl+X`) — настройки подхватятся при следующем запуске `nano`.

Проверка:

```bash
cat ~/.nanorc
```

---

## 2. 🌿 Настройка bashrc (Git prompt)

Файл конфигурации:

```bash
nano ~/.bashrc
```

Добавьте блок **в конец** файла (или замените существующий `PS1`, если он конфликтует).

### 🧩 Блок: ветка Git в приглашении

```bash
# Git branch in prompt
if [ -f /usr/lib/git-core/git-sh-prompt ]; then
    source /usr/lib/git-core/git-sh-prompt
    export GIT_PS1_SHOWDIRTYSTATE=1
    export GIT_PS1_SHOWSTASHSTATE=1
    export GIT_PS1_SHOWUNTRACKEDFILES=1
    export GIT_PS1_SHOWUPSTREAM="git"

    # Если хотите простую версию без символов (=, >, <)
    # export GIT_PS1_SHOWUPSTREAM=""

    PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[01;31m\]$(__git_ps1 " (%s)")\[\033[00m\]\$ '
fi
```

### 🚩 Что показывают флаги Git

| Переменная | Эффект |
|------------|--------|
| `GIT_PS1_SHOWDIRTYSTATE` | `*` — незакоммиченные изменения, `+` — staged |
| `GIT_PS1_SHOWSTASHSTATE` | `$` — есть stash |
| `GIT_PS1_SHOWUNTRACKEDFILES` | `%` — неотслеживаемые файлы |
| `GIT_PS1_SHOWUPSTREAM` | `=` / `>` / `<` — синхрон с remote / впереди / позади |

### 👀 Как выглядит prompt

```text
user@host:~/path (main *)$
         │      │    │  └─ состояние репозитория
         │      │    └──── имя ветки (красный)
         │      └───────── текущий каталог (синий)
         └──────────────── user@host (зелёный)
```

### ✅ Применить

```bash
source ~/.bashrc
```

или откройте новый терминал.

Проверка внутри git-репозитория:

```bash
cd ~/Projects/My-bookmarks
# в prompt должна появиться ветка, например: (main)
```

> 💡 **Замечание.** Путь `/usr/lib/git-core/git-sh-prompt` типичен для Debian/Ubuntu.  
> На других дистрибутивах файл может лежать в `/usr/share/git-core/contrib/completion/git-prompt.sh` — подставьте свой путь.

---

## 3. ⚡ Алиасы + история bash

Добавьте в **конец** `~/.bashrc` (после блока Git prompt или рядом с ним).

### 📜 История команд

```bash
# --- История bash ---
export HISTSIZE=10000
export HISTFILESIZE=20000
export HISTCONTROL=ignoreboth:erasedups   # не писать дубли и команды с пробелом в начале
export HISTTIMEFORMAT="%F %T  "           # дата и время в истории
shopt -s histappend                       # дописывать историю, а не перезаписывать
shopt -s cmdhist                          # многострочные команды — одной записью
```

| Переменная / опция | 💬 Смысл |
|--------------------|----------|
| `HISTSIZE` | Сколько команд держать в памяти сессии |
| `HISTFILESIZE` | Сколько строк хранить в `~/.bash_history` |
| `HISTCONTROL=ignoreboth:erasedups` | Игнор пробела в начале + дублей; удалять повторы |
| `HISTTIMEFORMAT` | Показывать время рядом с командой (`history`) |
| `histappend` | Не затирать историю при закрытии терминала |
| `cmdhist` | Многострочные команды сохраняются целиком |

### 🚀 Полезные алиасы

```bash
# --- Алиасы ---
# Навигация
alias ..='cd ..'
alias ...='cd ../..'
alias ~='cd ~'
alias c='clear'

# ls
alias ll='ls -alF --color=auto'
alias la='ls -A --color=auto'
alias l='ls -CF --color=auto'
alias ls='ls --color=auto'

# Безопасность / удобство
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
alias grep='grep --color=auto'

# Git (короткие)
alias gs='git status -sb'
alias ga='git add'
alias gc='git commit'
alias gp='git push'
alias gl='git log --oneline --graph --decorate -20'
alias gd='git diff'
alias gco='git checkout'
alias gb='git branch'

# Система
alias dfh='df -h'
alias duh='du -h --max-depth=1'
alias ports='ss -tulpn'
alias reload='source ~/.bashrc'
```

### 📖 Шпаргалка по алиасам

| Алиас | Команда | Зачем |
|-------|---------|-------|
| `ll` | `ls -alF` | Подробный список файлов |
| `..` / `...` | `cd ..` | Быстрый выход вверх |
| `gs` | `git status -sb` | Короткий статус репо |
| `gl` | `git log --oneline...` | Красивый лог |
| `reload` | `source ~/.bashrc` | Перечитать конфиг без перезапуска |
| `ports` | `ss -tulpn` | Какие порты слушаются |

### ✅ Применить

```bash
source ~/.bashrc
# или
reload
```

Проверка:

```bash
alias ll
history | tail -5
```

---

## 4. 🔧 Настройка gitconfig

Глобальный конфиг Git:

```bash
nano ~/.gitconfig
```

или через команды:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### 📄 Пример `~/.gitconfig`

```gitconfig
[user]
    name = Your Name
    email = you@example.com

[core]
    editor = nano
    autocrlf = input
    pager = less -FRX

[init]
    defaultBranch = main

[pull]
    rebase = false

[push]
    default = simple
    autoSetupRemote = true

[color]
    ui = auto

[alias]
    st = status -sb
    co = checkout
    br = branch
    ci = commit
    lg = log --oneline --graph --decorate -20
    last = log -1 HEAD
    unstage = reset HEAD --
    discard = checkout --
```

### 📋 Что важно в конфиге

| Секция | Зачем |
|--------|-------|
| `user.name` / `user.email` | 🪪 Автор коммитов (обязательно) |
| `core.editor` | ✏️ Редактор для commit / rebase |
| `init.defaultBranch` | 🌿 Имя ветки по умолчанию (`main`) |
| `pull.rebase` | 🔀 `false` = merge при pull; `true` = rebase |
| `push.autoSetupRemote` | ☁️ Не нужно `-u` при первом push новой ветки |
| `color.ui` | 🎨 Цветной вывод git |
| `alias.*` | ⚡ Короткие команды (`git st`, `git lg`) |

> ⚠️ Подставьте **свои** `name` и `email`. Для GitHub удобно email из настроек аккаунта (можно noreply).

### ✅ Применить и проверить

Сохраните файл — git читает его сразу.

```bash
git config --global --list
git config --global user.name
git config --global user.email
```

---

## ✅ Быстрый чеклист

- [ ] ✏️ Создан `~/.nanorc` с нужными опциями
- [ ] 👁️ `nano` показывает номера строк и корректные цвета
- [ ] 🌿 В `~/.bashrc` добавлен блок Git prompt
- [ ] ⚡ Добавлены алиасы и настройки истории
- [ ] 🔄 Выполнен `source ~/.bashrc` / `reload`
- [ ] 🔧 Настроен `~/.gitconfig` (имя, email, алиасы)
- [ ] 🧪 В git-репозитории в prompt видна ветка
