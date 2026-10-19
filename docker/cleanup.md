# 🧹 Docker: обслуживание и очистка диска

На VPS место часто съедают **старые образы**, **остановленные контейнеры**, **билд-кэш** и **логи**.

---

## 📑 Содержание

1. [Сколько места занято](#1-сколько-места-занято)
2. [Безопасная уборка](#2-безопасная-уборка)
3. [Агрессивная уборка](#3-агрессивная-уборка)
4. [Логи контейнеров](#4-логи-контейнеров)
5. [Ротация логов (json-file)](#5-ротация-логов-json-file)
6. [Чек-лист раз в месяц](#6-чек-лист-раз-в-месяц)

---

## 1. Сколько места занято

```bash
df -h /
docker system df
docker system df -v
```

Смотри блоки: Images / Containers / Local Volumes / Build Cache.

```bash
# самые жирные образы
docker images --format '{{.Size}}\t{{.Repository}}:{{.Tag}}' | sort -h

# тома
docker volume ls
docker volume inspect VOLUME_NAME
```

---

## 2. Безопасная уборка

Не трогает **используемые** образы, тома живых контейнеров и named volumes в работе:

```bash
# остановленные контейнеры
docker container prune -f

# сети без контейнеров
docker network prune -f

# «висячие» образы (<none>)
docker image prune -f

# build cache
docker builder prune -f

# всё перечисленное разом (без томов)
docker system prune -f
```

После обновления стека:

```bash
docker compose pull
docker compose up -d
docker image prune -f
```

---

## 3. Агрессивная уборка

> ⚠️ Может удалить данные. Не делай на проде без понимания.

```bash
# все образы, на которые никто не ссылается
docker image prune -a -f

# + неиспользуемые тома (БД без контейнера — исчезнут!)
docker system prune -a --volumes -f
```

Перед `prune --volumes`:

```bash
docker volume ls
docker ps -a
# убедись, что нужные БД-контейнеры запущены или тома тебе не жалко
```

---

## 4. Логи контейнеров

По умолчанию драйвер `json-file` — логи растут в:

```text
/var/lib/docker/containers/<id>/<id>-json.log
```

```bash
# размер логов по контейнерам (грубо)
sudo du -sh /var/lib/docker/containers/*/*-json.log 2>/dev/null | sort -h
```

Обрезать «на живую» (контейнер можно не останавливать, но лучше понимать риск):

```bash
# пример: обнулить лог конкретного файла
sudo truncate -s 0 /var/lib/docker/containers/CONTAINER_ID/CONTAINER_ID-json.log
```

Надёжнее — настроить ротацию заранее (следующий раздел).

Логи приложения через bind (как у nginx в `/var/log/nginx/...`) ротируй обычным `logrotate` на хосте — это отдельно от docker json-логов.

---

## 5. Ротация логов (json-file)

### Глобально — `/etc/docker/daemon.json`

```bash
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
sudo systemctl restart docker
```

> После рестарта демона контейнеры с `restart: unless-stopped` поднимутся сами.

### На один сервис в Compose

```yaml
services:
  api:
    image: myapi:1.0
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

---

## 6. Чек-лист раз в месяц

- [ ] `df -h` и `docker system df` — нет сюрпризов  
- [ ] `docker container prune` / `docker image prune`  
- [ ] Старые теги образов удалены после удачного деплоя  
- [ ] Включена ротация json-логов  
- [ ] Бэкапы named volumes / БД (отдельно от prune!)  
- [ ] Снапшот VPS перед агрессивным `prune -a --volumes`  

---

## ➡️ Связанное

- [cheatsheet.md](./cheatsheet.md) — команды  
- [compose.md](./compose.md) — обновление стека  
- [../nginx/](../nginx/) — логи приложений на хосте по доменам  
