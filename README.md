# Развёртывание Zabbix 7.0 LTS на Ubuntu 24.04

Данный раздел описывает развёртывание Zabbix 7.0 LTS в изолированной лабораторной среде на базе VirtualBox. В качестве агентов на целевых хостах используется классический **Zabbix Agent 1** (пассивные проверки + PSK-шифрование).

## 🗺 Архитектура стенда

Все хосты находятся в единой сети `192.168.200.0/24`. Шлюзом и DNS-сервером выступает `DC01` (192.168.200.10).

| Хостнейм | IP-адрес | Роль | Устанавливаемое ПО |
| :--- | :--- | :--- | :--- |
| **LINUX-ZABBIX-01** | 192.168.200.30 | Сервер мониторинга | Zabbix Server, Frontend, Agent 1, PostgreSQL, Nginx, PHP-FPM |
| **LINUX-NODE-1** | 192.168.200.34 | Целевой хост 1 | Zabbix Agent 1 |
| **LINUX-NODE-2** | 192.168.200.35 | Целевой хост 2 | Zabbix Agent 1 |

**Требования к серверу Zabbix:** минимум 2 ядра CPU, 2 ГБ RAM, 20 ГБ диска.
**Открываемые порты (UFW):**
- На сервере Zabbix: **80/tcp** (веб-интерфейс), **10051/tcp** (приём данных).
- На целевых хостах: **10050/tcp** (для пассивных проверок сервером, доступ только с IP `.30`).

> ⚠️ Перед началом работ на всех хостах обновите систему: `sudo apt update && sudo apt upgrade -y`.

---

## Часть 1. Установка Zabbix Server (192.168.200.30)

> 💡 **Автоматизация:** Вы можете выполнить установку вручную по шагам ниже или запустить готовый скрипт: `sudo ./install_zabbix_server.sh`

### Шаг 1. Подключение репозитория Zabbix 7.0 LTS
```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update
```

### Шаг 2. Установка пакетов
Устанавливаем Zabbix, а также явно указываем **PostgreSQL**, **Nginx** и **PHP-FPM**, так как зависимости пакетов Zabbix не всегда подтягивают сами сервисы.
```bash
sudo apt install -y postgresql nginx php8.3-fpm \
                    zabbix-server-pgsql zabbix-frontend-php php8.3-pgsql \
                    zabbix-nginx-conf zabbix-sql-scripts zabbix-agent
```

### Шаг 3. Создание базы данных и пользователя
```bash
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb -O zabbix zabbix
```
*Задайте и запомните пароль для пользователя `zabbix`.*

### Шаг 4. Импорт начальной схемы БД
```bash
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix
```
*При запросе введите пароль пользователя `zabbix` из Шага 3.*

### Шаг 5. Настройка Zabbix Server
```bash
sudo nano /etc/zabbix/zabbix_server.conf
```
Найдите строку `# DBPassword=` и укажите пароль от БД:
```ini
DBPassword=ваш_пароль_от_zabbix
```

### Шаг 6. Настройка Nginx и PHP
Пакет `zabbix-nginx-conf` уже размещает конфигурацию в `/etc/nginx/conf.d/zabbix.conf`.
Опционально задайте таймзону PHP, чтобы убрать предупреждение в веб-интерфейсе:
```bash
sudo sed -i 's|^;date.timezone =.*|date.timezone = Europe/Moscow|' /etc/php/8.3/fpm/php.ini
```

### Шаг 7. Запуск служб и настройка UFW
```bash
sudo systemctl restart zabbix-server zabbix-agent nginx php8.3-fpm
sudo systemctl enable zabbix-server zabbix-agent nginx php8.3-fpm

sudo ufw allow 80/tcp
sudo ufw allow 10051/tcp
sudo ufw --force enable
sudo ufw reload
```

### Шаг 8. Завершение установки через веб-интерфейс
Перейдите по адресу `http://192.168.200.30`. Пройдите мастер, указав:
- **Database host:** `localhost`
- **Database name:** `zabbix`
- **User:** `zabbix`
- **Password:** пароль с Шага 3.

Войдите в систему (логин: **Admin**, пароль: **zabbix**) и сразу смените пароль.

---

## Часть 2. Установка Zabbix Agent 1 (192.168.200.34 и .35)

> 💡 **Автоматизация:** На каждом хосте можно запустить: `sudo ./install_zabbix_agent.sh 192.168.200.30 <HOSTNAME>` (например, `LINUX-NODE-1`)

Выполняется **на каждом** целевом хосте.

### Шаг 1. Установка агента
```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update
sudo apt install -y zabbix-agent
```

### Шаг 2. Настройка конфигурации
Откройте `/etc/zabbix/zabbix_agentd.conf`:
```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```
Укажите IP сервера и **точное** имя хоста (совпадающее с тем, что вы создадите в веб-интерфейсе):
```ini
Server=192.168.200.30
Hostname=LINUX-NODE-1
```
*(Для второго хоста: `Hostname=LINUX-NODE-2`)*.

### Шаг 3. Шифрование PSK
Сгенерируйте ключ на **каждом** агенте (флаг `tr -d '\n'` убирает перевод строки, предотвращая ошибку `SSL alert number 40`):
```bash
openssl rand -hex 32 | tr -d '\n' | sudo tee /etc/zabbix/zabbix_agentd.psk
sudo chown zabbix:zabbix /etc/zabbix/zabbix_agentd.psk
sudo chmod 600 /etc/zabbix/zabbix_agentd.psk
```

Добавьте в конец файла `zabbix_agentd.conf`:
```ini
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=PSK LINUX-NODE-1
TLSPSKFile=/etc/zabbix/zabbix_agentd.psk
```

### Шаг 4. UFW и запуск
```bash
sudo ufw allow from 192.168.200.30 to any port 10050 proto tcp
sudo ufw --force enable
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent
```

Проверка метрик локально:
```bash
zabbix_agentd -t system.cpu.load[all,avg1]
```

---

## Часть 3. Добавление хостов в Zabbix Frontend

1. Перейдите в **Data collection → Hosts** → **Create host**.
2. **Host name**: `LINUX-NODE-1` (точно как в конфиге).
3. **Interfaces**: добавьте Agent, IP `192.168.200.34`, порт `10050`.
4. **Templates**: привяжите `Linux by Zabbix agent`.
5. **Encryption**:
   - Connections to host: `PSK`
   - Connections from host: `PSK`
   - PSK identity: `PSK LINUX-NODE-1`
   - PSK: вставьте содержимое файла `/etc/zabbix/zabbix_agentd.psk` с агента.
6. Нажмите **Add**. Через 1–2 минуты данные появятся в **Monitoring → Latest data**.

---

## 🛠 Возможные проблемы и решения

| Проблема | Вероятная причина | Решение |
| :--- | :--- | :--- |
| **Zabbix Server не запускается** | Неверный пароль БД или PostgreSQL недоступен | Проверьте `DBPassword`. Логи: `journalctl -u zabbix-server -n 50`. Статус БД: `systemctl status postgresql`. |
| **502 Bad Gateway** | PHP-FPM не запущен | Проверьте `systemctl status php8.3-fpm`. В `/etc/nginx/conf.d/zabbix.conf` сокет должен быть `unix:/run/php/php8.3-fpm.sock`. |
| **`SSL alert number 40`** | Ключ или identity PSK не совпадают | Сверьте PSK identity и сам ключ в веб-интерфейсе и конфиге. Проверьте длину файла PSK: `wc -c /etc/zabbix/zabbix_agentd.psk` (должно быть `64`). |
| **`user postgres does not exist`** | Не установлен PostgreSQL | Выполните `sudo apt install -y postgresql`. |
| **Хост "Not available"** | Агент не отвечает или закрыт порт | С сервера: `telnet 192.168.200.34 10050`. На агенте: `systemctl status zabbix-agent`, логи `/var/log/zabbix/zabbix_agentd.log`. |
```

---

### Файл `zabbix/install_zabbix_server.sh`

```bash
#!/bin/bash
# Установка Zabbix Server 7.0 LTS на Ubuntu 24.04 (IP: 192.168.200.30)
# Требует прав root (sudo)
# Использование: sudo ./install_zabbix_server.sh [DB_PASSWORD] [TIMEZONE]

set -euo pipefail

if [ "$EUID" -ne 0 ]; then
  echo "Пожалуйста, запустите с sudo: sudo ./install_zabbix_server.sh"
  exit 1
fi

DB_PASSWORD="${1:-$(openssl rand -hex 16)}"
TIMEZONE="${2:-Europe/Moscow}"

echo "=== Обновление списка пакетов ==="
apt update

echo "=== Подключение официального репозитория Zabbix 7.0 LTS ==="
cd /tmp
wget -q --show-progress https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
apt update

echo "=== Установка пакетов ==="
apt install -y postgresql nginx php8.3-fpm \
               zabbix-server-pgsql zabbix-frontend-php php8.3-pgsql \
               zabbix-nginx-conf zabbix-sql-scripts zabbix-agent

echo "=== Настройка таймзоны PHP ==="
PHP_INI=/etc/php/8.3/fpm/php.ini
if [ -f "${PHP_INI}" ]; then
  sed -i "s|^;date.timezone =.*|date.timezone = ${TIMEZONE}|; s|^date.timezone =.*|date.timezone = ${TIMEZONE}|" "${PHP_INI}"
fi

echo "=== Создание БД и пользователя ==="
if sudo -u postgres psql -tAc "SELECT 1 FROM pg_roles WHERE rolname='zabbix'" | grep -q 1; then
  sudo -u postgres psql -c "ALTER USER zabbix WITH PASSWORD '${DB_PASSWORD}';"
else
  sudo -u postgres psql -c "CREATE USER zabbix WITH PASSWORD '${DB_PASSWORD}';"
fi

if sudo -u postgres psql -tAc "SELECT 1 FROM pg_database WHERE datname='zabbix'" | grep -q 1; then
  echo "База zabbix уже существует — пропускаем."
else
  sudo -u postgres createdb -O zabbix zabbix
fi

echo "=== Импорт схемы БД (если не импортирована) ==="
if PGPASSWORD="${DB_PASSWORD}" psql -h localhost -U zabbix -d zabbix \
     -tAc "SELECT 1 FROM information_schema.tables WHERE table_name='users'" | grep -q 1; then
  echo "Схема уже импортирована."
else
  zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | \
    PGPASSWORD="${DB_PASSWORD}" psql -h localhost -U zabbix -d zabbix
fi

echo "=== Настройка zabbix_server.conf ==="
ZBX_CONF=/etc/zabbix/zabbix_server.conf
if grep -q "^DBPassword=" "${ZBX_CONF}"; then
  sed -i "s|^DBPassword=.*|DBPassword=${DB_PASSWORD}|" "${ZBX_CONF}"
elif grep -q "^# DBPassword=" "${ZBX_CONF}"; then
  sed -i "s|^# DBPassword=.*|DBPassword=${DB_PASSWORD}|" "${ZBX_CONF}"
else
  echo "DBPassword=${DB_PASSWORD}" >> "${ZBX_CONF}"
fi

echo "=== Запуск служб и настройка UFW ==="
systemctl restart zabbix-server zabbix-agent nginx php8.3-fpm
systemctl enable zabbix-server zabbix-agent nginx php8.3-fpm

ufw allow 80/tcp
ufw allow 10051/tcp
ufw --force enable
ufw reload

echo ""
echo "=== Готово! ==="
echo "Веб-интерфейс Zabbix: http://192.168.200.30"
echo "Данные для мастера: DB host: localhost | DB name: zabbix | User: zabbix | Password: ${DB_PASSWORD}"
echo "Логин по умолчанию: Admin / zabbix"
```

---

### Файл `zabbix/install_zabbix_agent.sh`

```bash
#!/bin/bash
# Установка Zabbix Agent 1 на Ubuntu 24.04 (PSK, пассивные проверки)
# Требует прав root (sudo)
# Использование: sudo ./install_zabbix_agent.sh [SERVER_IP] [HOSTNAME] [PSK_IDENTITY]

set -euo pipefail

if [ "$EUID" -ne 0 ]; then
  echo "Пожалуйста, запустите с sudo."
  exit 1
fi

ZABBIX_SERVER="${1:-192.168.200.30}"
AGENT_HOSTNAME="${2:-$(hostname)}"
PSK_IDENTITY="${3:-PSK ${AGENT_HOSTNAME}}"
AGENT_CONF=/etc/zabbix/zabbix_agentd.conf
PSK_FILE=/etc/zabbix/zabbix_agentd.psk

echo "=== Параметры ==="
echo "Server: ${ZABBIX_SERVER} | Hostname: ${AGENT_HOSTNAME} | PSK ID: ${PSK_IDENTITY}"

echo "=== Установка репозитория и агента ==="
cd /tmp
wget -q --show-progress https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
apt update
apt install -y zabbix-agent

echo "=== Настройка конфигурации ==="
sed -i "s|^Server=.*|Server=${ZABBIX_SERVER}|; s|^# Server=.*|Server=${ZABBIX_SERVER}|" "${AGENT_CONF}"
sed -i "s|^Hostname=.*|Hostname=${AGENT_HOSTNAME}|; s|^# Hostname=.*|Hostname=${AGENT_HOSTNAME}|" "${AGENT_CONF}"

sed -i '/^TLSConnect=/d;/^TLSAccept=/d;/^TLSPSKIdentity=/d;/^TLSPSKFile=/d' "${AGENT_CONF}"

echo "=== Генерация PSK ==="
openssl rand -hex 32 | tr -d '\n' > "${PSK_FILE}"
chown zabbix:zabbix "${PSK_FILE}"
chmod 600 "${PSK_FILE}"

cat >> "${AGENT_CONF}" <<EOF

### TLS PSK
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=${PSK_IDENTITY}
TLSPSKFile=${PSK_FILE}
EOF

echo "=== UFW и запуск ==="
ufw allow from ${ZABBIX_SERVER} to any port 10050 proto tcp
ufw --force enable
systemctl restart zabbix-agent
systemctl enable zabbix-agent

echo ""
echo "=== Готово! ==="
echo "Параметры для добавления в веб-интерфейс Zabbix:"
echo "  Host name:  ${AGENT_HOSTNAME}"
echo "  IP:         $(hostname -I | awk '{print $1}')"
echo "  PSK ID:     ${PSK_IDENTITY}"
echo "  PSK Key:    $(cat ${PSK_FILE})"
```
