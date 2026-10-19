# 🌐 Nginx + Certbot в Docker

Цепочка заметок: reverse-proxy на **nginx**, SSL через **Let's Encrypt (certbot)**, логи по доменам в `/var/log/nginx/<domain>/`.

## 🗂 Порядок чтения

| # | Файл | О чём |
|---|------|--------|
| 1 | [nginx-docker.md](./nginx-docker.md) | Структура каталогов, `docker-compose.yml`, сети, тома логов |
| 2 | [nginx-init.md](./nginx-init.md) | Временный `nginx-init.conf` для выпуска сертификата (HTTP + ACME) |
| 3 | [get-ssl.md](./get-ssl.md) | Команда `certbot certonly`, продление, проверка |
| 4 | *(ниже в docker-заметке)* | Полноценный `nginx.conf` с HTTPS, proxy и access/error логами |

## 🧭 Короткий сценарий

```text
1. Поднять nginx с init-конфигом (только :80 + /.well-known/acme-challenge/)
2. Выпустить сертификат через certbot (webroot)
3. Подменить конфиг на боевой (443 + SSL + proxy_pass)
4. Перезагрузить nginx; certbot renew крутится в фоне
```

## ✅ Перед стартом

- [ ] Установлен Docker — см. [../install-docker.md](../install-docker.md)
- [ ] Домен(ы) указывают A/AAAA на IP VPS
- [ ] В UFW открыты `80/tcp` и `443/tcp`
- [ ] Есть внешние docker-сети приложений (если proxy на контейнеры)
