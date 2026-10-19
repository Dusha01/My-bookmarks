# ⚡ Docker: быстрые команды

Шпаргалка без теории. Понятия — в [basics.md](./basics.md).

---

## 📑 Содержание

1. [Контейнеры](#1-контейнеры)
2. [Образы](#2-образы)
3. [Логи и exec](#3-логи-и-exec)
4. [Сети и тома](#4-сети-и-тома)
5. [Ресурсы и статистика](#5-ресурсы-и-статистика)
6. [Compose (частое)](#6-compose-частое)
7. [Очистка (коротко)](#7-очистка-коротко)

---

## 1. Контейнеры

```bash
# список
docker ps                  # запущенные
docker ps -a               # все
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'

# запуск / остановка
docker run -d --name web -p 8080:80 nginx:1.27
docker start web
docker stop web
docker restart web
docker rm web              # удалить остановленный
docker rm -f web           # остановить и удалить

# инфо
docker inspect web
docker port web
docker top web
```

Полезные флаги `run`:

```bash
docker run -d \
  --name app \
  --restart unless-stopped \
  -p 8080:80 \
  -e NODE_ENV=production \
  -v "$(pwd)/data:/data" \
  --network app-net \
  myapp:1.0
```

| Флаг | Смысл |
|------|--------|
| `-d` | В фоне |
| `--name` | Имя контейнера |
| `-p host:container` | Порты |
| `-e KEY=val` | Env |
| `-v src:dst[:ro]` | Том / bind |
| `--network` | Сеть |
| `--rm` | Удалить после выхода (разовые задачи) |
| `-it` | Интерактив + TTY (`bash` / `sh`) |

---

## 2. Образы

```bash
docker images
docker images -a

docker pull nginx:1.27
docker pull postgres:16

docker rmi nginx:1.27          # удалить образ
docker image prune             # висячие (dangling)
docker image prune -a          # все неиспользуемые (осторожно)

docker tag nginx:1.27 my-nginx:prod
docker history nginx:1.27
```

---

## 3. Логи и exec

```bash
docker logs web
docker logs -f web             # follow
docker logs --tail 100 web
docker logs --since 1h web

docker exec -it web bash       # или sh, если нет bash
docker exec web nginx -t
docker exec web nginx -s reload

# скопировать файлы
docker cp web:/etc/nginx/nginx.conf ./nginx.conf.bak
docker cp ./file.txt web:/tmp/
```

---

## 4. Сети и тома

```bash
# сети
docker network ls
docker network create app-net
docker network inspect app-net
docker network connect app-net web
docker network disconnect app-net web
docker network rm app-net

# тома
docker volume ls
docker volume create pgdata
docker volume inspect pgdata
docker volume rm pgdata
```

---

## 5. Ресурсы и статистика

```bash
docker stats                   # CPU / RAM в реальном времени
docker stats --no-stream
docker system df               # место: images, containers, volumes
docker system df -v
```

---

## 6. Compose (частое)

```bash
cd ~/project

docker compose up -d
docker compose up -d --build
docker compose ps
docker compose logs -f
docker compose logs -f api

docker compose stop
docker compose start
docker compose restart
docker compose down            # остановить и убрать контейнеры сети проекта
docker compose down -v         # + тома проекта (данные!)

docker compose exec api sh
docker compose pull
docker compose config          # показать итоговый YAML
```

Подробнее: [compose.md](./compose.md).

---

## 7. Очистка (коротко)

```bash
# остановленные контейнеры
docker container prune

# неиспользуемые сети / образы / всё сразу
docker network prune
docker image prune -a
docker system prune            # контейнеры + сети + dangling images
docker system prune -a --volumes   # агрессивно: +неиспользуемые образы и тома
```

> ⚠️ `prune --volumes` может снести данные БД, если том не примонтирован к живому контейнеру. Подробности: [cleanup.md](./cleanup.md).

---

## 🆘 Если «permission denied»

```bash
# либо через sudo
sudo docker ps

# либо группа docker + перелогин
sudo usermod -aG docker "$USER"
# выйти из SSH и зайти снова
```
