# 🚀 Nginx init-конфиг (до получения SSL)

Пока сертификатов Let's Encrypt ещё нет, nginx **нельзя** стартовать с путями на `fullchain.pem` — контейнер упадёт.  
Нужен временный **`nginx-init.conf`**: только порт 80, раздача ACME-challenge и заглушка.

> После выпуска сертификата → переключаешься на боевой конфиг из [nginx-docker.md](./nginx-docker.md).

---

## 📑 Содержание

1. [Зачем init](#1-зачем-init)
2. [nginx-init.conf](#2-nginx-initconf)
3. [Как подключить в Compose](#3-как-подключить-в-compose)
4. [Проверка ACME path](#4-проверка-acme-path)
5. [Переход на боевой конфиг](#5-переход-на-боевой-конфиг)

---

## 1. Зачем init

| Этап | Конфиг | Порты | SSL |
|------|--------|-------|-----|
| 1. Старт | `nginx-init.conf` | только `80` | нет |
| 2. Certbot | `certonly --webroot` | — | выпуск в `certbot/conf` |
| 3. Боевой | `nginx.conf` | `80` + `443` | да |

Certbot (HTTP-01) кладёт файл в:

```text
/var/www/certbot/.well-known/acme-challenge/<token>
```

Let's Encrypt ходит на:

```text
http://твой-домен/.well-known/acme-challenge/<token>
```

Поэтому в init-конфиге обязательно:

```nginx
location /.well-known/acme-challenge/ {
    root /var/www/certbot;
}
```

---

## 2. `nginx-init.conf`

Создай файл рядом с compose:

```bash
cd ~/nginx
nano nginx-init.conf
```

### Один домен

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
    sendfile      on;

    server {
        listen 80;
        listen [::]:80;
        server_name example.com www.example.com;

        # ACME HTTP-01
        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        # Всё остальное — заглушка (или временный proxy без SSL)
        location / {
            return 200 'nginx-init: waiting for SSL certificates\n';
            add_header Content-Type text/plain;
        }
    }
}
```

### Несколько доменов

Либо несколько `server { ... }`, либо один с перечислением имён:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name
        example.com
        www.example.com
        admin.example.com
        stage.example.com
        ;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 200 'nginx-init OK\n';
        add_header Content-Type text/plain;
    }
}
```

> 💡 Для init логи по доменам не обязательны — достаточно общего `error_log`. Каталоги `/var/log/nginx/<domain>/` всё равно лучше создать заранее под будущий боевой конфиг.

---

## 3. Как подключить в Compose

В `docker-compose.yml` временно смонтируй init:

```yaml
volumes:
  - ./nginx-init.conf:/etc/nginx/nginx.conf:ro
  - ./certbot/conf:/etc/letsencrypt:ro
  - ./certbot/www:/var/www/certbot:ro
```

Порты `443:443` можно оставить — просто не будет SSL-серверов, пока не переключишь конфиг.

Запуск:

```bash
cd ~/nginx
docker compose up -d nginx
docker exec nginx_proxy nginx -t
curl -sI http://example.com/
# ожидаем 200 и текст заглушки
```

DNS домена должен уже смотреть на IP этого VPS.

---

## 4. Проверка ACME path

Положи тестовый файл и открой снаружи:

```bash
mkdir -p ~/nginx/certbot/www/.well-known/acme-challenge
echo ok > ~/nginx/certbot/www/.well-known/acme-challenge/ping

curl -s http://example.com/.well-known/acme-challenge/ping
# ожидаем: ok
```

Если `ok` не видно:

- DNS не на этот сервер  
- UFW/облачный фаервол режет 80  
- неверный `root` в `location` (должен быть `/var/www/certbot`, **без** `.well-known` в root)  
- смонтирован не тот том `certbot/www`

---

## 5. Переход на боевой конфиг

1. Выпусти сертификат — [get-ssl.md](./get-ssl.md)  
2. Напиши боевой `nginx.conf` — [nginx-docker.md §5](./nginx-docker.md#5-боевой-nginxconf)  
3. В compose верни том:

```yaml
- ./nginx.conf:/etc/nginx/nginx.conf:ro
```

4. Перезапусти / reload:

```bash
docker compose up -d nginx
docker exec nginx_proxy nginx -t
docker exec nginx_proxy nginx -s reload
```

Init-файл можно оставить в репозитории/на сервере на будущее (новый домен = снова короткий HTTP-only server + certbot).

---

## ✅ Чек-лист init

- [ ] `nginx-init.conf` слушает только 80  
- [ ] Есть `location /.well-known/acme-challenge/` с `root /var/www/certbot`  
- [ ] В compose смонтирован именно init-конфиг  
- [ ] `curl` к домену и к `/acme-challenge/ping` работает  
- [ ] Можно переходить к [get-ssl.md](./get-ssl.md)  
