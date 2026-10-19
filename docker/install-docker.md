# 🐳 Установка Docker и Docker Compose на VPS

Установка **Docker Engine** и плагина **Compose** (`docker compose`) на **Debian / Ubuntu**.

> 🔗 Логично ставить после [первой настройки VPS](./first-setting-vps.md) (SSH, пользователь с sudo, фаервол).

---

## 📑 Содержание

1. [Подготовка системы](#1-подготовка-системы)
2. [Установка Docker Engine](#2-установка-docker-engine)
3. [Установка Docker Compose](#3-установка-docker-compose)
4. [После установки](#4-после-установки)
5. [Проверка](#5-проверка)

---

## 1. Подготовка системы

Обнови списки пакетов и поставь зависимости:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release
```

---

## 2. Установка Docker Engine

### 2.1. GPG-ключ репозитория

Ключ нужен `apt` для проверки подлинности пакетов Docker.

**Ubuntu:**

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

**Debian:**

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
```

### 2.2. APT-репозиторий

**Ubuntu:**

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**Debian:**

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

### 2.3. Установка пакетов

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

| Пакет | Зачем |
|-------|--------|
| `docker-ce` | Docker Engine |
| `docker-ce-cli` | Клиент `docker` |
| `containerd.io` | Runtime контейнеров |
| `docker-buildx-plugin` | Расширенная сборка образов |
| `docker-compose-plugin` | Команда `docker compose` из репозитория Docker |

> 💡 Если ставишь `docker-compose-plugin` через apt — ручная загрузка бинарника из §3 не нужна. Раздел 3 — запасной вариант.

### 2.4. Запуск сервиса

```bash
sudo systemctl enable --now docker
sudo systemctl status docker
```

---

## 3. Установка Docker Compose

### Вариант A (рекомендуемый) — плагин из apt

Уже входит в команду установки выше (`docker-compose-plugin`).

Проверка:

```bash
docker compose version
```

### Вариант B — бинарник с GitHub вручную

Docker CLI ищет плагины в таком порядке:

**Локальные пути пользователя:**

| Путь | Описание |
|------|----------|
| `$DOCKER_CLI_PLUGIN_PATH` | Если задана переменная — проверяется первой |
| `~/.docker/cli-plugins/` | Плагины текущего пользователя |

**Системные пути:**

| Путь | Описание |
|------|----------|
| `/usr/local/lib/docker/cli-plugins/` | Ручная установка (удобно для всех пользователей) |
| `/usr/lib/docker/cli-plugins/` | Путь пакетного менеджера |

```bash
sudo mkdir -p /usr/local/lib/docker/cli-plugins/

sudo curl -SL \
  "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/lib/docker/cli-plugins/docker-compose

sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
docker compose version
```

---

## 4. После установки

### 4.1. Пользователь без `sudo` для Docker

```bash
sudo usermod -aG docker "$USER"
```

Выйди из SSH и зайди снова (или `newgrp docker`), чтобы группа применилась.

> ⚠️ Членство в группе `docker` по сути даёт права root на хосте. Добавляй только доверенных пользователей.

### 4.2. UFW (если контейнеры должны быть доступны снаружи)

Пример — открыть HTTP/HTTPS:

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status
```

Порты пробрасывай осознанно — только то, что реально нужно.

---

## 5. Проверка

```bash
docker --version
docker compose version
sudo docker run --rm hello-world
```

Ожидаемо: контейнер скачается, выведет приветствие и завершится.

### Чек-лист

- [ ] Добавлен GPG-ключ и apt-репозиторий Docker  
- [ ] Установлены `docker-ce` и плагины  
- [ ] `systemctl` показывает `docker` как `active`  
- [ ] `docker compose version` работает  
- [ ] Пользователь добавлен в группу `docker` (если нужно)  
- [ ] `hello-world` успешно запускается  

---

## 🆘 Частые проблемы

| Проблема | Что делать |
|----------|------------|
| `permission denied` к docker.sock | Пользователь не в группе `docker` или не перелогинился |
| `docker compose`: command not found | Поставь `docker-compose-plugin` или бинарник из §3 |
| `apt update` ругается на ключ/репо | Проверь Ubuntu vs Debian URL и путь к `docker.gpg` |
| Сервис не стартует | `journalctl -u docker -n 50 --no-pager` |

---

## ➡️ Что дальше

1. [basics.md](./basics.md) — образы, порты, тома, сети  
2. [cheatsheet.md](./cheatsheet.md) — быстрые команды  
3. [compose.md](./compose.md) — несколько сервисов  
4. [../nginx/](../nginx/) — reverse-proxy Nginx + Certbot
