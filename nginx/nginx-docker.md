# 🐳 Nginx + Certbot: Docker Compose и боевой конфиг

Reverse-proxy в Docker: **nginx** слушает 80/443, **certbot** хранит сертификаты и периодически делает `renew`. Логи с хоста — в `/var/log/nginx/<домен>/`.

> 🔗 Шаги выпуска SSL: [nginx-init.md](./nginx-init.md) → [get-ssl.md](./get-ssl.md) → вернуться сюда за боевым `nginx.conf`.

---

## 📑 Содержание

1. [Структура каталогов](#1-структура-каталогов)
2. [Логи на хосте](#2-логи-на-хосте)
3. [docker-compose.yml](#3-docker-composeyml)
4. [Сети](#4-сети)
5. [Боевой nginx.conf](#5-боевой-nginxconf)
6. [Запуск и перезагрузка](#6-запуск-и-перезагрузка)
7. [Чек-лист](#7-чек-лист)

---

## 1. Структура каталогов

На сервере (пример — домашний каталог пользователя):

```text
~/nginx/
├── docker-compose.yml
├── nginx.conf              # боевой конфиг (после SSL)
├── nginx-init.conf         # временный конфиг (до SSL) — см. nginx-init.md
└── certbot/
    ├── conf/               # → /etc/letsencrypt в контейнерах
    └── www/                # webroot для ACME → /var/www/certbot
```

```bash
mkdir -p ~/nginx/certbot/{conf,www}
cd ~/nginx
```

---

## 2. Логи на хосте

Чтобы логи не жили только внутри контейнера, монтируем каталоги с хоста **по доменам**:

```bash
sudo mkdir -p /var/log/nginx/{example.com,admin.example.com,stage.example.com}
sudo chown -R "$USER":"$USER" /var/log/nginx
```

Внутри контейнера nginx пишет, например:

```text
/var/log/nginx/example.com/access.log
/var/log/nginx/example.com/error.log
```

На хосте те же файлы видны в `/var/log/nginx/example.com/`.

> 💡 Каталог на хосте должен существовать **до** `docker compose up`, иначе Docker создаст его как root-директорию с неудобными правами (или как файл — зависит от ситуации). Лучше всегда `mkdir -p` заранее.

---

## 3. `docker-compose.yml`

Пример по мотивам рабочего сетапа (подставь свои домены и сети):

```yaml
services:
  nginx:
    image: nginx:1.27
    container_name: nginx_proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certbot/conf:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot:ro
      # логи по доменам (хост → контейнер)
      - /var/log/nginx/example.com:/var/log/nginx/example.com
      - /var/log/nginx/admin.example.com:/var/log/nginx/admin.example.com
      - /var/log/nginx/stage.example.com:/var/log/nginx/stage.example.com
    restart: unless-stopped
    networks:
      - nginx_internal
      - app-net          # сеть бэкенда (external)

  certbot:
    image: certbot/certbot:v2.11.0
    container_name: certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew --webroot -w /var/www/certbot; sleep 12h & wait $${!}; done;'"
    restart: unless-stopped
    networks:
      - nginx_internal

networks:
  nginx_internal:
    driver: bridge
  app-net:
    external: true
    name: app-net
```

| Том | Зачем |
|-----|--------|
| `./nginx.conf` → `/etc/nginx/nginx.conf` | Единый конфиг proxy |
| `./certbot/conf` → `/etc/letsencrypt` | Сертификаты (у nginx — `:ro`) |
| `./certbot/www` → `/var/www/certbot` | Webroot для HTTP-01 challenge |
| `/var/log/nginx/<domain>` | Access/error логи на хосте |

> ⚠️ Пока сертификатов нет, монтируй **init-конфиг**:  
> `./nginx-init.conf:/etc/nginx/nginx.conf:ro`  
> После выпуска SSL — верни `./nginx.conf`.

---

## 4. Сети

Nginx должен быть в тех же Docker-сетях, что и приложения, иначе `proxy_pass http://service:port` не резолвится.

```bash
# посмотреть сети
docker network ls

# создать, если ещё нет
docker network create app-net
```

В `docker-compose` приложений укажи ту же `external` сеть, например `app-net`.

Несколько проектов:

```yaml
networks:
  - nginx_internal
  - dusha-web_dusha-net   # external
  - landing-net           # external
```

---

## 5. Боевой `nginx.conf`

После получения сертификатов (см. [get-ssl.md](./get-ssl.md)) — полный конфиг.

### Каркас

```nginx
worker_processes auto;

error_log /var/log/nginx/error.log warn;
pid       /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout 65;
    client_max_body_size 32m;

    # Реальные IP за прокси (если нужно)
    # set_real_ip_from 172.16.0.0/12;
    # real_ip_header X-Forwarded-For;

    # ---- HTTP: ACME + редирект на HTTPS ----
    server {
        listen 80;
        listen [::]:80;
        server_name example.com www.example.com;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    # ---- HTTPS: основной сайт ----
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name example.com www.example.com;

        ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

        access_log /var/log/nginx/example.com/access.log;
        error_log  /var/log/nginx/example.com/error.log;

        location / {
            proxy_pass http://frontend:80;
            proxy_http_version 1.1;
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    # ---- HTTPS: админка ----
    server {
        listen 443 ssl http2;
        listen [::]:443 ssl http2;
        server_name admin.example.com;

        ssl_certificate     /etc/letsencrypt/live/admin.example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/admin.example.com/privkey.pem;
        include /etc/letsencrypt/options-ssl-nginx.conf;
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

        access_log /var/log/nginx/admin.example.com/access.log;
        error_log  /var/log/nginx/admin.example.com/error.log;

        location / {
            proxy_pass http://backend:8000;
            proxy_http_version 1.1;
            proxy_set_header Host              $host;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### Что важно в SSL-блоке

| Директива / файл | Смысл |
|------------------|--------|
| `ssl_certificate` | `fullchain.pem` из `live/<domain>/` |
| `ssl_certificate_key` | `privkey.pem` |
| `options-ssl-nginx.conf` | Готовые TLS-настройки от certbot (появляется после первого выпуска) |
| `ssl_dhparams.pem` | DH-параметры (certbot создаёт сам) |
| `access_log` / `error_log` | Отдельная папка на домен |

> 📌 Если `include ... options-ssl-nginx.conf` падает с «file not found» — сначала выпусти сертификат; certbot положит эти файлы в `certbot/conf/`.  
> Временно можно закомментировать `include` и `ssl_dhparam` или скопировать дефолты из образа certbot.

### Несколько доменов на один сертификат

Если сертификат выпускался с `-d example.com -d www.example.com`, путь всё равно:

```text
/etc/letsencrypt/live/example.com/
```

(имя каталога = **первый** `-d` в команде certbot).

---

## 6. Запуск и перезагрузка

```bash
cd ~/nginx

# поднять
docker compose up -d

# проверить
docker compose ps
docker compose logs -f nginx

# проверка конфига внутри контейнера
docker exec nginx_proxy nginx -t

# мягкая перезагрузка после правок nginx.conf
docker exec nginx_proxy nginx -s reload

# логи на хосте
tail -f /var/log/nginx/example.com/access.log
```

UFW (если ещё не открыто):

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw status
```

---

## 7. Чек-лист

- [ ] Есть `~/nginx/certbot/{conf,www}`  
- [ ] Созданы `/var/log/nginx/<domain>/` на хосте  
- [ ] Внешние сети приложений созданы и прописаны в compose  
- [ ] Сначала работал [init-конфиг](./nginx-init.md), потом получен [SSL](./get-ssl.md)  
- [ ] В боевом `nginx.conf` — редирект 80→443, ACME location, proxy_pass, логи по доменам  
- [ ] `nginx -t` OK, сайт открывается по HTTPS  

---

## ➡️ Дальше

1. [nginx-init.md](./nginx-init.md) — временный конфиг до сертификата  
2. [get-ssl.md](./get-ssl.md) — выпуск и renew  
