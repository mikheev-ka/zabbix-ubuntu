# Развёртывание Zabbix 7.0 LTS на Ubuntu 24.04 (с Zabbix Agent 1)

Данное руководство описывает развёртывание Zabbix 7.0 LTS на Ubuntu 24.04 LTS в изолированной лабораторной среде на базе VirtualBox. В качестве агентов на целевых хостах используется классический **Zabbix Agent 1**, обеспечивающий стабильную работу и простоту настройки.

## Архитектура лабораторного стенда

Схема сети соответствует вашей топологии: доступ в интернет через NAT (DC01), внутренняя сеть `Host-Only` для виртуальных машин.

| Хостнейм | IP-адрес | Роль | Устанавливаемое ПО |
| :--- | :--- | :--- | :--- |
| **LINUX-ZABBIX-01** | 192.168.200.30 | Сервер мониторинга | Zabbix Server, Frontend, Agent 1, PostgreSQL, Nginx |
| **LINUX-NODE-1** | 192.168.200.34 | Целевой хост 1 | Zabbix Agent 1 |
| **LINUX-NODE-2** | 192.168.200.35 | Целевой хост 2 | Zabbix Agent 1 |

**Особенности сети:** Все хосты находятся в сети `192.168.200.0/24`. Шлюзом и DNS-сервером выступает `DC01` (192.168.200.10). Прямая сетевая связность между сервером и агентами гарантирована.

**Требования к серверу:** минимум 2 ядра CPU, 2 ГБ RAM, 20 ГБ диска.

**Открываемые порты (UFW):**
- На сервере Zabbix: **80/tcp** (веб-интерфейс), **10051/tcp** (приём данных от агентов, если используются активные проверки).
- На целевых хостах: **10050/tcp** (для пассивных проверок сервером).

Перед началом работ на всех хостах обновите систему:
```bash
sudo apt update && sudo apt upgrade -y
```

---

## Часть 1. Установка Zabbix Server на хост 192.168.200.30

### Шаг 1. Подключение официального репозитория Zabbix 7.0 LTS

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update
```

### Шаг 2. Установка пакетов Zabbix

Устанавливаем сервер, веб-интерфейс, PostgreSQL и классический Agent 1 (пакет `zabbix-agent`).

```bash
sudo apt install -y zabbix-server-pgsql zabbix-frontend-php php8.3-pgsql zabbix-nginx-conf zabbix-sql-scripts zabbix-agent
```

### Шаг 3. Создание базы данных и пользователя PostgreSQL

```bash
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb -O zabbix zabbix
```
Задайте и запомните пароль для пользователя `zabbix`.

### Шаг 4. Импорт начальной схемы БД

Проверьте оба возможных пути к SQL-скриптам:
```bash
ls /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz
ls /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz
```
Выполните импорт, используя найденный путь:
```bash
# Если файл найден по первому пути:
zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix

# Если файл найден по второму пути:
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix
```
При запросе введите пароль пользователя `zabbix`, заданный на Шаге 3.

### Шаг 5. Настройка Zabbix Server

```bash
sudo nano /etc/zabbix/zabbix_server.conf
```
Укажите пароль от базы данных:
```ini
DBPassword=ваш_пароль_от_zabbix
```

### Шаг 6. Настройка Nginx

Пакет `zabbix-nginx-conf` уже размещает конфигурацию в `/etc/nginx/conf.d/zabbix.conf`. Отредактируйте `server_name`, если требуется:
```bash
sudo nano /etc/nginx/conf.d/zabbix.conf
```

### Шаг 7. Запуск служб

```bash
sudo systemctl restart zabbix-server zabbix-agent nginx php8.3-fpm
sudo systemctl enable zabbix-server zabbix-agent nginx php8.3-fpm
```

### Шаг 8. Настройка файрвола UFW

```bash
sudo ufw allow 80/tcp
sudo ufw allow 10051/tcp
sudo ufw reload
```

### Шаг 9. Завершение установки через веб-интерфейс

Откройте браузер и перейдите по адресу `http://192.168.200.30`. Пройдите мастер установки, указав параметры БД:
- **Database host:** `localhost`
- **Database name:** `zabbix`
- **User:** `zabbix`
- **Password:** пароль с Шага 3.

Войдите в систему (логин: **Admin**, пароль: **zabbix**) и сразу смените пароль.

---

## Часть 2. Установка и настройка Zabbix Agent 1 на целевых хостах

Все шаги выполняются **на каждом целевом хосте** (`192.168.200.34` и `192.168.200.35`) независимо.

### Шаг 1. Подключение репозитория и установка Agent 1

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update
sudo apt install -y zabbix-agent
```

### Шаг 2. Настройка конфигурации агента

Откройте файл конфигурации (обратите внимание, у Agent 1 он называется `zabbix_agentd.conf`):
```bash
sudo nano /etc/zabbix/zabbix_agentd.conf
```

#### 2.1. Режим проверок и адрес сервера

Для лабораторного стенда используем **пассивные проверки** — сервер Zabbix сам опрашивает агентов. Это самый простой и надежный путь.

Укажите IP-адрес Zabbix Server в параметре `Server`:
```ini
### Option: Server
Server=192.168.200.30
```

#### 2.2. Критически важный параметр Hostname

Параметр `Hostname` должен **точно совпадать** с именем хоста, которое вы создадите в веб-интерфейсе Zabbix. Регистр имеет значение.

```ini
### Option: Hostname
Hostname=LINUX-NODE-1
```
(Для второго хоста соответственно: `Hostname=LINUX-NODE-2`.)

### Шаг 3. Шифрование трафика с помощью PSK

Сгенерируйте ключ на **каждом** агенте отдельно:
```bash
openssl rand -hex 32 | sudo tee /etc/zabbix/zabbix_agentd.psk
sudo chown zabbix:zabbix /etc/zabbix/zabbix_agentd.psk
sudo chmod 600 /etc/zabbix/zabbix_agentd.psk
```

Добавьте в конец файла `/etc/zabbix/zabbix_agentd.conf`:
```ini
TLSConnect=psk
TLSAccept=psk
TLSPSKIdentity=PSK LINUX-NODE-1
TLSPSKFile=/etc/zabbix/zabbix_agentd.psk
```
`TLSPSKIdentity` должен быть уникальным для каждого хоста (для Node-2 используйте `PSK LINUX-NODE-2`).

### Шаг 4. Файрвол UFW

Поскольку используются пассивные проверки, откройте порт 10050, ограничив доступ только IP-адресом сервера:
```bash
sudo ufw allow from 192.168.200.30 to any port 10050 proto tcp
```

### Шаг 5. Запуск и проверка агента

```bash
sudo systemctl restart zabbix-agent
sudo systemctl enable zabbix-agent
systemctl status zabbix-agent --no-pager
```

Проверьте сбор метрик локально:
```bash
zabbix_agentd -t system.cpu.load[all,avg1]
```
Успешный ответ (например, `[d|0.000000]`) подтвердит, что агент работает.

---

## Часть 3. Добавление хостов в Zabbix Frontend

1.  В веб-интерфейсе перейдите в **Data collection → Hosts** и нажмите **Create host**.
2.  Укажите **Host name**, которое **точно совпадает** с параметром `Hostname` в конфигурации агента (например, `LINUX-NODE-1`).
3.  В поле **Interfaces** добавьте агентский интерфейс с IP-адресом целевого хоста (например, `192.168.200.34`) и портом `10050`.
4.  Привяжите шаблон **Linux by Zabbix agent**.
5.  Перейдите на вкладку **Encryption** и укажите:
    - **Connections to host:** `PSK`
    - **Connections from host:** `PSK`
    - **PSK identity:** тот же идентификатор, что в `TLSPSKIdentity` агента (например, `PSK LINUX-NODE-1`).
    - **PSK:** вставьте содержимое файла `/etc/zabbix/zabbix_agentd.psk` с агента.
6.  Нажмите **Add**. Через 1–2 минуты на вкладке **Monitoring → Latest data** появятся первые метрики.

Повторите Часть 2 и Часть 3 для второго хоста (`192.168.200.35`), заменив `Hostname`, `TLSPSKIdentity` и IP-адрес.

---

## Возможные проблемы и способы их решения

| Проблема | Вероятная причина | Решение |
| :--- | :--- | :--- |
| **Zabbix Server не запускается** (`status=1`) | Неверный пароль БД или PostgreSQL недоступен | Проверьте `DBPassword` в `/etc/zabbix/zabbix_server.conf`. Логи: `journalctl -u zabbix-server -n 50`. |
| **Ошибка 502 Bad Gateway** | PHP-FPM не запущен | Проверьте статус `php8.3-fpm`. Убедитесь, что в `/etc/nginx/conf.d/zabbix.conf` сокет указан верно (`unix:/run/php/php8.3-fpm.sock`). |
| **`SSL alert number 40`** | Ключ или идентификатор PSK не совпадают | Убедитесь, что на вкладке **Encryption** хоста указаны те же `PSK identity` и значение `PSK`, что и в конфигурации агента. Проверьте, нет ли лишнего перевода строки в PSK-файле (можно использовать `truncate -s 64 /etc/zabbix/zabbix_agentd.psk`). |
| **`connection refused` на агенте** | Служба остановлена или порт 10050 закрыт | Проверьте `systemctl status zabbix-agent`. Убедитесь, что UFW разрешает доступ с IP сервера: `sudo ufw allow from 192.168.200.30 to any port 10050 proto tcp`. |
| **Хост в статусе “Not available”** | Агент не отвечает | С сервера проверьте доступность: `telnet 192.168.200.34 10050`. На агенте проверьте конфигурацию и логи (`/var/log/zabbix/zabbix_agentd.log`). |
