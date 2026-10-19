# 🔐 Получение SSL-сертификатов (Certbot + webroot)

Выпуск Let's Encrypt через **HTTP-01** в webroot, который раздаёт nginx из init-конфига.

> Предварительно: nginx уже крутится с [nginx-init.conf](./nginx-init.md), домен смотрит на VPS, порт 80 открыт.

---

## 📑 Содержание

1. [Одноразовый выпуск](#1-одноразовый-выпуск)
2. [Несколько доменов](#2-несколько-доменов)
3. [Куда кладутся файлы](#3-куда-кладутся-файлы)
4. [Проверка](#4-проверка)
5. [Автопродление](#5-автопродление)
6. [Типичные ошибки](#6-типичные-ошибки)

---

## 1. Одноразовый выпуск

Из каталога `~/nginx` (рядом с `certbot/`):

```bash
cd ~/nginx

docker run --rm \
  -v "$(pwd)/certbot/www:/var/www/certbot" \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  certbot/certbot certonly --webroot \
  -w /var/www/certbot \
  -d example.com \
  --email you@example.com \
  --agree-tos \
  --no-eff-email
```

| Флаг | Смысл |
|------|--------|
| `certonly` | Только выпустить сертификат, не трогать системный nginx |
| `--webroot` | HTTP-01 через файлы в webroot |
| `-w /var/www/certbot` | Корень webroot (совпадает с томом и `root` в nginx) |
| `-d example.com` | Домен (можно несколько `-d`) |
| `--email` | Почта для уведомлений Let's Encrypt |
| `--agree-tos` | Согласие с ToS |
| `--no-eff-email` | Не подписываться на рассылку EFF |

Подставь свои `example.com` и `you@example.com`.

### С www

```bash
docker run --rm \
  -v "$(pwd)/certbot/www:/var/www/certbot" \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  certbot/certbot certonly --webroot \
  -w /var/www/certbot \
  -d example.com \
  -d www.example.com \
  --email you@example.com \
  --agree-tos \
  --no-eff-email
```

Каталог сертификата будет назван по **первому** `-d` → `live/example.com/`.

---

## 2. Несколько доменов

Отдельный сертификат на каждый vhost (удобно для логов и независимых сроков):

```bash
# основной
docker run --rm \
  -v "$(pwd)/certbot/www:/var/www/certbot" \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  certbot/certbot certonly --webroot \
  -w /var/www/certbot \
  -d example.com -d www.example.com \
  --email you@example.com --agree-tos --no-eff-email

# админка
docker run --rm \
  -v "$(pwd)/certbot/www:/var/www/certbot" \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  certbot/certbot certonly --webroot \
  -w /var/www/certbot \
  -d admin.example.com \
  --email you@example.com --agree-tos --no-eff-email

# stage
docker run --rm \
  -v "$(pwd)/certbot/www:/var/www/certbot" \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  certbot/certbot certonly --webroot \
  -w /var/www/certbot \
  -d stage.example.com \
  --email you@example.com --agree-tos --no-eff-email
```

Для каждого домена в init-конфиге должен быть `server_name` (или общий server), который отдаёт `/.well-known/acme-challenge/`.

---

## 3. Куда кладутся файлы

На хосте:

```text
~/nginx/certbot/conf/
├── live/
│   └── example.com/
│       ├── fullchain.pem   ← ssl_certificate
│       ├── privkey.pem     ← ssl_certificate_key
│       ├── cert.pem
│       └── chain.pem
├── archive/
├── renewal/
├── options-ssl-nginx.conf  ← include в nginx.conf
└── ssl-dhparams.pem        ← ssl_dhparam
```

В контейнере nginx те же пути как `/etc/letsencrypt/...` (том `certbot/conf`).

Права: `live/` и `archive/` обычно только для root внутри тома — nginx читает их ок, если контейнер стартует от root (официальный образ nginx).

---

## 4. Проверка

```bash
# список сертификатов
docker run --rm \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  certbot/certbot certificates

# файлы на диске
ls -la certbot/conf/live/example.com/

# с хоста (после переключения на боевой nginx.conf)
curl -sI https://example.com/
```

Дальше:

1. Пропиши пути к `fullchain.pem` / `privkey.pem` в [боевом nginx.conf](./nginx-docker.md#5-боевой-nginxconf)  
2. В compose смонтируй `./nginx.conf` вместо init  
3. `docker compose up -d nginx && docker exec nginx_proxy nginx -t && docker exec nginx_proxy nginx -s reload`

---

## 5. Автопродление

В [docker-compose](./nginx-docker.md) сервис `certbot` уже крутит цикл:

```yaml
entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew --webroot -w /var/www/certbot; sleep 12h & wait $${!}; done;'"
```

Раз в ~12 часов вызывается `certbot renew`.  
После удачного renew nginx нужно подхватить новые файлы:

```bash
# разовый reload вручную
docker exec nginx_proxy nginx -s reload
```

Опционально — deploy-hook при renew (пример идеи):

```bash
docker run --rm \
  -v "$(pwd)/certbot/www:/var/www/certbot" \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  certbot/certbot renew --webroot -w /var/www/certbot \
  --deploy-hook "docker exec nginx_proxy nginx -s reload"
```

(для постоянного hook удобнее прописать его в `renewal/*.conf` или отдельный cron на хосте.)

Проверка renew «всухую»:

```bash
docker run --rm \
  -v "$(pwd)/certbot/www:/var/www/certbot" \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  certbot/certbot renew --dry-run
```

---

## 6. Типичные ошибки

| Ошибка / симптом | Что проверить |
|------------------|---------------|
| `Connection refused` / timeout от LE | DNS, UFW `80/tcp`, nginx слушает 80 |
| `404` на challenge | `root /var/www/certbot` + том `./certbot/www` |
| `unauthorized` | Домен указывает не на этот IP / CDN режет challenge |
| Rate limit Let's Encrypt | Не спамить выпуском; для тестов — `--staging` |
| nginx не стартует после SSL | Нет файлов в `live/...` или забыли init→боевой |
| `options-ssl-nginx.conf` not found | Сначала успешный `certonly`, потом include |

Тестовый (staging) выпуск — чтобы не упереться в лимиты:

```bash
docker run --rm \
  -v "$(pwd)/certbot/www:/var/www/certbot" \
  -v "$(pwd)/certbot/conf:/etc/letsencrypt" \
  certbot/certbot certonly --webroot \
  -w /var/www/certbot \
  -d example.com \
  --email you@example.com --agree-tos --no-eff-email \
  --staging
```

Staging-сертификат браузер не доверит — для отладки challenge. Потом выпусти без `--staging` (иногда нужно `--force-renewal` / другой путь — обычно просто повторный `certonly` на prod).

---

## ✅ Чек-лист

- [ ] Init-nginx отдаёт `/.well-known/acme-challenge/`  
- [ ] `certonly --webroot` завершился без ошибок  
- [ ] Есть `certbot/conf/live/<domain>/fullchain.pem`  
- [ ] Боевой `nginx.conf` указывает на эти файлы  
- [ ] HTTPS открывается, HTTP редиректит на HTTPS  
- [ ] `certbot renew --dry-run` проходит  

---

## ➡️ Дальше

Вернись к [nginx-docker.md](./nginx-docker.md) — подключи боевой конфиг, логи по доменам и `nginx -s reload`.
