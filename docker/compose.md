# 🧩 Docker Compose

Несколько сервисов одним YAML: приложение + БД + proxy, общие сети и тома.

> Команды-шпаргалка: [cheatsheet.md](./cheatsheet.md) · Пример proxy: [../nginx/nginx-docker.md](../nginx/nginx-docker.md)

---

## 📑 Содержание

1. [Минимальный файл](#1-минимальный-файл)
2. [Порты, env, тома](#2-порты-env-тома)
3. [Сети между проектами](#3-сети-между-проектами)
4. [depends_on и порядок старта](#4-depends_on-и-порядок-старта)
5. [Типичные команды](#5-типичные-команды)
6. [Обновление стека](#6-обновление-стека)

---

## 1. Минимальный файл

`docker-compose.yml` (или `compose.yaml`):

```yaml
services:
  web:
    image: nginx:1.27
    container_name: demo_web
    ports:
      - "8080:80"
    restart: unless-stopped

  api:
    image: myapi:1.0
    container_name: demo_api
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/app
    restart: unless-stopped
    depends_on:
      - db

  db:
    image: postgres:16
    container_name: demo_db
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  pgdata:
```

Внутри проекта `api` ходит к БД по имени сервиса: `db:5432` (не `localhost`).

---

## 2. Порты, env, тома

```yaml
services:
  api:
    image: myapi:1.0
    ports:
      - "127.0.0.1:8000:8000"   # только localhost хоста
    env_file:
      - .env                    # не коммить секреты
    volumes:
      - ./app:/app:ro           # bind
      - uploads:/data/uploads   # named volume
    networks:
      - frontend
      - backend

networks:
  frontend:
  backend:

volumes:
  uploads:
```

`.env` рядом с compose (пример):

```env
POSTGRES_PASSWORD=change_me
```

```bash
chmod 600 .env
```

---

## 3. Сети между проектами

Один nginx-прокси + несколько приложений → общая **external** сеть:

```bash
docker network create edge
```

В nginx-проекте:

```yaml
networks:
  edge:
    external: true
    name: edge
```

В приложении:

```yaml
services:
  api:
    networks:
      - edge
      - default

networks:
  edge:
    external: true
    name: edge
```

Тогда в nginx: `proxy_pass http://api:8000;` (имя = **имя сервиса** или `container_name`, если задан и резолвится в сети).

---

## 4. `depends_on` и порядок старта

```yaml
depends_on:
  - db
```

Это **порядок запуска**, не «БД уже готова принимать коннекты».  
Для Postgres часто нужны healthcheck + `condition: service_healthy` (Compose v2):

```yaml
services:
  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 10

  api:
    depends_on:
      db:
        condition: service_healthy
```

---

## 5. Типичные команды

```bash
docker compose up -d              # поднять
docker compose ps
docker compose logs -f api
docker compose exec api sh
docker compose restart api
docker compose down               # контейнеры + сеть проекта
docker compose down -v            # ещё и тома — данные пропадут
docker compose config             # валидный «итоговый» YAML
```

Несколько файлов:

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 6. Обновление стека

```bash
cd ~/project
docker compose pull
docker compose up -d
docker image prune -f    # опционально убрать старые слои
```

Пересобрать свой образ:

```bash
docker compose up -d --build
```

Откат: вернуть старый тег в YAML и снова `up -d` (если образ ещё локально есть).

---

## ✅ Чек-лист проекта

- [ ] Зафиксированы теги образов (не только `latest`)  
- [ ] Секреты в `.env` / secrets, не в git  
- [ ] Данные БД в named volume  
- [ ] `restart: unless-stopped`  
- [ ] Наружу торчат только нужные порты (или только через nginx)  
- [ ] Для multi-project — общая external-сеть  
