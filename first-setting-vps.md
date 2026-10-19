# 🔒 Первоначальная настройка VPS (SSH + fail2ban + UFW)

Базовая настройка свежего VPS на **Debian / Ubuntu**: обновление системы, sudo-пользователь, вход по ключу, смена порта SSH, fail2ban, фаервол, время и swap.

> ⚠️ **Главное правило:** не закрывай текущую SSH-сессию, пока не проверишь новое подключение в **отдельном** окне терминала.  
> Если доступ потерян — используй консоль провайдера (VNC / KVM / rescue) или снапшот.

В примерах ниже:

| Плейсхолдер | Значение |
|-------------|----------|
| `ТВОЙ_IP` | IP-адрес VPS |
| `ТВОЙ_ДОМАШНИЙ_IP` | Твой постоянный IP (дома / офис / VPN) |
| `2222` | Новый SSH-порт (`1024`–`65535`, не `22` / `80` / `443`) |
| `deploy` | Имя обычного пользователя с sudo |
| `id_ed25519_vps` | Имя пары SSH-ключей на ПК |
| `vps-hostname` | Имя хоста сервера |

> 📌 На **RHEL / Alma / Rocky** сервис часто называется `sshd`, фаервол — `firewalld` (не UFW). Команды ниже — для Debian/Ubuntu.

---

## 📑 Содержание

1. [План действий](#1-план-действий)
2. [Создание SSH-ключей на ПК](#2-создание-ssh-ключей-на-пк)
3. [Копирование ключа на VPS](#3-копирование-ключа-на-vps)
4. [Первый вход: система, hostname, пакеты](#4-первый-вход-система-hostname-пакеты)
5. [Время (NTP) и часовой пояс](#5-время-ntp-и-часовой-пояс)
6. [Swap (для маленьких VPS)](#6-swap-для-маленьких-vps)
7. [Пользователь с sudo](#7-пользователь-с-sudo)
8. [Настройка SSH на сервере](#8-настройка-ssh-на-сервере)
9. [Ubuntu: ssh.socket vs ssh.service](#9-ubuntu-sshsocket-vs-sshservice)
10. [Проверка нового подключения](#10-проверка-нового-подключения)
11. [fail2ban](#11-fail2ban)
12. [UFW](#12-ufw)
13. [Удобный ~/.ssh/config](#13-удобный-sshconfig)
14. [Бэкап и восстановление доступа](#14-бэкап-и-восстановление-доступа)
15. [Финальная проверка](#15-финальная-проверка)
16. [Если что-то пошло не так](#16-если-что-то-пошло-не-так)

---

## 1. План действий

1. Создать SSH-ключи на ПК и сделать бэкап приватного ключа  
2. Скопировать публичный ключ на VPS (сначала можно на `root`)  
3. Обновить систему, задать hostname, поставить базовые пакеты  
4. Настроить время (NTP) и часовой пояс  
5. При необходимости добавить swap  
6. Создать пользователя с sudo и перенести ключ на него  
7. Ужесточить `sshd_config` (порт, ключи, `AllowUsers`, без root-логина)  
8. На Ubuntu отключить `ssh.socket`  
9. Проверить вход **новым пользователем** во втором окне  
10. Поставить fail2ban и включить UFW (лучше — только с твоего IP)  

---

## 2. Создание SSH-ключей на ПК

```bash
ssh-keygen -t ed25519 -C "vps-key" -f ~/.ssh/id_ed25519_vps
```

| Флаг | Смысл |
|------|--------|
| `-t ed25519` | Современный алгоритм (предпочтительнее RSA) |
| `-C "vps-key"` | Комментарий к ключу |
| `-f ~/.ssh/id_ed25519_vps` | Путь к файлу ключа |

```text
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```

💡 Для безопасности задай passphrase.

| Файл | Тип | Важно |
|------|-----|--------|
| `~/.ssh/id_ed25519_vps` | приватный | 🚨 никому не показывать, сделать бэкап |
| `~/.ssh/id_ed25519_vps.pub` | публичный | копируем на сервер |

```bash
cp ~/.ssh/id_ed25519_vps ~/Documents/backup_id_ed25519_vps
chmod 600 ~/Documents/backup_id_ed25519_vps
```

---

## 3. Копирование ключа на VPS

### Способ 1 (рекомендуемый) — `ssh-copy-id`

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_vps.pub -p 22 root@ТВОЙ_IP
```

### Способ 2 (ручной)

```bash
cat ~/.ssh/id_ed25519_vps.pub
```

На сервере:

```bash
ssh root@ТВОЙ_IP -p 22
mkdir -p ~/.ssh
echo "ВСТАВЬ_СЮДА_ПУБЛИЧНЫЙ_КЛЮЧ" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
exit
```

---

## 4. Первый вход: система, hostname, пакеты

```bash
ssh -i ~/.ssh/id_ed25519_vps root@ТВОЙ_IP -p 22
```

### 4.1. Обновление системы

```bash
apt update
apt upgrade -y
apt autoremove -y
```

Автообновления безопасности (рекомендуется):

```bash
apt install unattended-upgrades -y
dpkg-reconfigure -plow unattended-upgrades
```

### 4.2. Hostname

```bash
hostnamectl set-hostname vps-hostname
```

Проверь `/etc/hosts` — должна быть строка вида:

```text
127.0.1.1   vps-hostname
```

```bash
nano /etc/hosts
hostnamectl
```

### 4.3. Базовые пакеты и локаль

```bash
apt install -y \
  sudo curl wget git htop nano \
  ca-certificates gnupg lsb-release \
  ufw fail2ban
```

Локаль (пример):

```bash
apt install -y locales
locale-gen en_US.UTF-8
update-locale LANG=en_US.UTF-8
```

---

## 5. Время (NTP) и часовой пояс

```bash
timedatectl set-ntp true
timedatectl
```

Часовой пояс (пример — Владивосток):

```bash
timedatectl set-timezone Asia/Vladivostok
# или интерактивно:
dpkg-reconfigure tzdata
```

Проверка:

```bash
date
timedatectl status
```

---

## 6. Swap (для маленьких VPS)

Если RAM ≤ 1–2 ГБ, проверь текущий swap:

```bash
swapon --show
free -h
```

### 6.1. Swap ещё нет — создать

Пример: файл на 2 ГБ:

```bash
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

# в fstab — только если записи ещё нет
grep -q '/swapfile' /etc/fstab || echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

### 6.2. Swap уже есть — увеличить / пересоздать

Если `swapon --show` уже показывает `/swapfile`, а ты хочешь другой размер (например, было `512M`, нужно `2G`):

```text
fallocate: fallocate failed: Text file busy
```

Файл занят системой — сначала отключи swap, потом пересоздай:

```bash
swapoff /swapfile
rm /swapfile

fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile

swapon --show
# NAME      TYPE SIZE USED PRIO
# /swapfile file   2G   0B   -2
```

Проверь `fstab` (запись должна быть одна):

```bash
grep swap /etc/fstab
# /swapfile none swap sw 0 0
```

Если строки нет — добавь:

```bash
echo '/swapfile none swap sw 0 0' >> /etc/fstab
```

> 💡 На очень маленьких дисках `fallocate` иногда недоступен — запасной вариант:  
> `dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress`

### 6.3. Swappiness (опционально)

```bash
echo 'vm.swappiness=10' > /etc/sysctl.d/99-swappiness.conf
sysctl -p /etc/sysctl.d/99-swappiness.conf
```

---

## 7. Пользователь с sudo

Не работай постоянно под `root`.

```bash
adduser deploy
usermod -aG sudo deploy
```

Ключ для нового пользователя:

```bash
mkdir -p /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys
```

Проверка sudo (в текущей сессии можно так):

```bash
su - deploy
sudo whoami    # должно вывести: root
exit
```

🚨 **Пока не отключай root-логин**, пока не проверишь вход под `deploy` в новом окне (см. §10).

---

## 8. Настройка SSH на сервере

### 8.1. Открыть конфиг

```bash
nano /etc/ssh/sshd_config
```

### 8.2. Рекомендуемые параметры

```sshd
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
AllowUsers deploy
MaxAuthTries 3
MaxSessions 2
LoginGraceTime 30
```

| Параметр | Значение | Зачем |
|----------|----------|--------|
| `Port` | `2222` | Меньше шума от сканеров |
| `PermitRootLogin` | `no` | Root по SSH запрещён |
| `PasswordAuthentication` | `no` | Только ключи |
| `PermitEmptyPasswords` | `no` | Запрет пустых паролей |
| `AllowUsers` | `deploy` | Явный whitelist пользователей |
| `PubkeyAuthentication` | `yes` | Вход по ключу |
| `MaxAuthTries` / `MaxSessions` | `3` / `2` | Лимиты brute-force / сессий |

> 💡 Пока тестируешь, временно можно оставить `PermitRootLogin prohibit-password`, а после успешного входа под `deploy` сменить на `no` и перезапустить SSH.

### 8.3. Права на ключи

```bash
chmod 600 /home/deploy/.ssh/authorized_keys
chmod 700 /home/deploy/.ssh
```

### 8.4. Проверка синтаксиса

```bash
sshd -t
```

Пустой вывод = ошибок нет.

---

## 9. Ubuntu: `ssh.socket` vs `ssh.service`

На многих Ubuntu SSH идёт через **`ssh.socket`** и может **игнорировать** `Port` из `sshd_config`.

```bash
systemctl stop ssh.socket
systemctl disable ssh.socket
systemctl mask ssh.socket

systemctl enable ssh
systemctl restart ssh
```

Одной командой (после `sshd -t`):

```bash
systemctl stop ssh.socket && \
systemctl disable ssh.socket && \
systemctl mask ssh.socket && \
systemctl enable ssh && \
systemctl restart ssh && \
systemctl status ssh && \
ss -tulpn | grep ssh
```

Проверка:

```bash
systemctl status ssh
systemctl status ssh.socket   # masked / inactive
ss -tulpn | grep -E '2222|:22'
```

Должен слушаться **`2222`**.

### Откат socket

```bash
systemctl unmask ssh.socket
systemctl enable --now ssh.socket
systemctl stop ssh
systemctl disable ssh
```

---

## 10. Проверка нового подключения

🚨 **Не закрывай** старую сессию.

В **новом** окне:

```bash
ssh -i ~/.ssh/id_ed25519_vps deploy@ТВОЙ_IP -p 2222
sudo whoami
```

Если вошёл и `sudo` работает — можно закрывать старую сессию и окончательно выставить `PermitRootLogin no` (если ещё не выставил).

---

## 11. fail2ban

```bash
apt update
apt install fail2ban -y
nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime = 3600
findtime = 600
maxretry = 3
ignoreip = 127.0.0.1/8 ::1 ТВОЙ_ДОМАШНИЙ_IP

[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600
```

Опциональный фильтр `/etc/fail2ban/filter.d/sshd.local`:

```ini
[Definition]
failregex = ^.*(?:Failed password|authentication failure|Connection closed by authenticating user).*$
ignoreregex =
```

```bash
systemctl enable fail2ban
systemctl restart fail2ban
systemctl status fail2ban
fail2ban-client status sshd
```

---

## 12. UFW

> ⚠️ Сначала открой SSH-порт, потом включай UFW.

### Рекомендуемый вариант — только твой IP

```bash
ufw default deny incoming
ufw default allow outgoing

ufw allow from ТВОЙ_ДОМАШНИЙ_IP to any port 2222 proto tcp

# Если IP динамический и так нельзя — открыть порт всем (хуже):
# ufw allow 2222/tcp

ufw delete allow 22/tcp 2>/dev/null || true
ufw logging on
ufw --force enable
ufw status verbose
```

Проверка:

```bash
ss -tulpn | grep -E '2222|:22'
ufw status numbered
```

### Альтернатива — iptables

```bash
iptables -A INPUT -p tcp -s ТВОЙ_ДОМАШНИЙ_IP --dport 2222 -j ACCEPT
apt install iptables-persistent -y
netfilter-persistent save
```

---

## 13. Удобный `~/.ssh/config`

На **ПК**:

```bash
nano ~/.ssh/config
```

```sshconfig
Host myserver
    HostName ТВОЙ_IP
    Port 2222
    User deploy
    IdentityFile ~/.ssh/id_ed25519_vps
```

```bash
chmod 600 ~/.ssh/config
ssh myserver
```

---

## 14. Бэкап и восстановление доступа

| Что | Зачем |
|-----|--------|
| Приватный ключ на ПК + копия офлайн | Без ключа на сервер не зайдёшь |
| Снапшот у провайдера **до** смены порта / UFW | Быстрый откат |
| Консоль VNC/KVM | Если SSH отрезан |
| Запись порта, пользователя, IP whitelist | Не забыть свои же правила |

Если потерял доступ:

1. Открой консоль провайдера или загрузись в rescue  
2. Смонтируй диск / зайди в систему  
3. Временно ослабь правила: `ufw disable` или поправь `sshd_config`  
4. `systemctl restart ssh` и проверь вход  

---

## 15. Финальная проверка

```bash
# Кто слушает SSH
ss -tulpn | grep 2222

# UFW
ufw status verbose

# Параметры sshd
grep -E '^(Port|PasswordAuthentication|PubkeyAuthentication|PermitRootLogin|AllowUsers)' /etc/ssh/sshd_config

# Время и hostname
timedatectl
hostnamectl

# Место на диске / память
df -h
free -h

# fail2ban
fail2ban-client status sshd

# Неудачные попытки входа
grep 'Failed password' /var/log/auth.log | tail -20

# Логи SSH
journalctl -u ssh -n 50 --no-pager
```

### Чек-лист

- [ ] Ключи созданы, приватный ключ в бэкапе  
- [ ] Система обновлена, `unattended-upgrades` включены  
- [ ] Заданы hostname и timezone, NTP работает  
- [ ] (При необходимости) настроен swap  
- [ ] Создан пользователь `deploy` с sudo и ключом  
- [ ] SSH-порт изменён, `ssh.socket` отключён (Ubuntu)  
- [ ] `PasswordAuthentication no`, `PermitRootLogin no`, `AllowUsers` задан  
- [ ] Вход под `deploy` проверен во втором окне  
- [ ] fail2ban работает на новом порту  
- [ ] UFW включён (лучше — только с твоего IP), порт `22` закрыт  
- [ ] Есть снапшот / план восстановления  
- [ ] Настроен `~/.ssh/config` на ПК  

---

## 16. Если что-то пошло не так

| Проблема | Что делать |
|----------|------------|
| Не коннектится на новый порт | Старая сессия + `sshd -t`, `ss -tulpn`, UFW |
| Старая сессия закрыта | VNC / KVM / rescue / снапшот |
| `Permission denied` | Права `700`/`600` на `.ssh`, верный `-i`, пользователь из `AllowUsers` |
| Порт всё ещё `22` | См. [§9 ssh.socket](#9-ubuntu-sshsocket-vs-sshservice) |
| `sudo` просит пароль и мешает | Задай пароль пользователю (`passwd deploy`) или настрой NOPASSWD осознанно |
| UFW отрезал доступ | Консоль провайдера → `ufw allow ...` / `ufw disable` |
| После рестарта SSH молчит | `journalctl -u ssh -n 50`, `sshd -t` |

---

## ➡️ Что дальше

После базовой защиты сервера можно поставить Docker:

→ [docker/](./docker/) — Docker Engine, Compose, команды и обслуживание
