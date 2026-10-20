# Развёртывание Zabbix 7.0 LTS на Ubuntu 24.04

Данное руководство описывает два способа развёртывания Zabbix 7.0 LTS на Ubuntu 24.04 LTS с PostgreSQL и Nginx: классическую ручную установку из пакетов и запуск в Docker. Zabbix 7.0 — это текущий релиз с долгосрочной поддержкой (LTS), выпущенный 4 июня 2024 года и получающий обновления безопасности до июня 2029 года.

## Архитектура и требования

Для развёртывания сервера мониторинга достаточно одной виртуальной машины. PostgreSQL и Nginx устанавливаются как зависимости при установке пакетов Zabbix и не требуют отдельной предварительной инсталляции.

| Параметр | Минимальное значение | Рекомендуемое значение |
| :--- | :--- | :--- |
| **CPU** | 2 ядра | 4 ядра |
| **RAM** | 2 ГБ | 4 ГБ (для 100+ хостов) |
| **Диск** | 20 ГБ | 50 ГБ (с учётом истории метрик) |
| **ОС** | Ubuntu 24.04 LTS | Ubuntu 24.04 LTS |

**Открываемые порты (UFW):**
- **80/tcp** (или **443/tcp**) — веб-интерфейс Zabbix Frontend
- **10051/tcp** — Zabbix Server (приём данных от агентов)
- **10050/tcp** — Zabbix Agent (пассивный режим опроса)

Перед началом работ обновите систему:
```bash
sudo apt update && sudo apt upgrade -y
```

---

## Вариант 1. Классическая ручная установка (пакеты)

Этот способ предполагает установку Server, Frontend, Agent 2, PostgreSQL и Nginx из официального репозитория Zabbix. Пакетный менеджер `apt` автоматически установит PostgreSQL 16, Nginx и PHP 8.3 FPM со всеми необходимыми расширениями.

### Шаг 1. Подключение репозитория Zabbix 7.0 LTS

```bash
wget https://repo.zabbix.com/zabbix/7.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo dpkg -i zabbix-release_latest_7.0+ubuntu24.04_all.deb
sudo apt update
```

### Шаг 2. Установка пакетов Zabbix

Одна команда устанавливает сервер с поддержкой PostgreSQL, веб-интерфейс, конфигурацию для Nginx, SQL-скрипты и Zabbix Agent 2 с плагинами расширенного мониторинга:

```bash
sudo apt install -y zabbix-server-pgsql zabbix-frontend-php php8.3-pgsql zabbix-nginx-conf zabbix-sql-scripts zabbix-agent2 zabbix-agent2-plugin-*
```

Проверьте, что сервер установился корректно:
```bash
zabbix_server --version
```

### Шаг 3. Создание базы данных и пользователя PostgreSQL

```bash
sudo -u postgres createuser --pwprompt zabbix
sudo -u postgres createdb -O zabbix zabbix
```

Команда `createuser --pwprompt` запросит пароль — задайте надёжный пароль и запомните его. Он понадобится на следующем шаге и при настройке `zabbix_server.conf`.

### Шаг 4. Импорт начальной схемы БД

Путь к SQL-скриптам может различаться в зависимости от версии пакета. Проверьте наличие файла одним из способов:

```bash
# Вариант А (наиболее распространённый в пакетах для Ubuntu 24.04)
ls /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz

# Вариант Б (альтернативный путь в некоторых сборках)
ls /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz
```

Выполните импорт, используя найденный путь. При выполнении система запросит пароль пользователя `zabbix`, заданный на предыдущем шаге:

```bash
# Если файл найден по пути А:
zcat /usr/share/zabbix/sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix

# Если файл найден по пути Б:
zcat /usr/share/zabbix-sql-scripts/postgresql/server.sql.gz | sudo -u zabbix psql zabbix
```

> **Примечание:** В официальной документации Zabbix для некоторых версий фигурирует путь `/usr/share/zabbix-sql-scripts/postgresql/`, в то время как в ряде сторонних руководств и сборок используется `/usr/share/zabbix/sql-scripts/postgresql/`. Проверка обоих вариантов исключает ошибку.

### Шаг 5. Настройка Zabbix Server

```bash
sudo nano /etc/zabbix/zabbix_server.conf
```

Найдите параметр `DBPassword` и укажите пароль, заданный для пользователя `zabbix` в PostgreSQL:
```ini
DBPassword=ваш_пароль_от_zabbix
```

### Шаг 6. Настройка Nginx для Zabbix Frontend

Пакет `zabbix-nginx-conf` уже устанавливает готовый конфигурационный файл в `/etc/nginx/conf.d/zabbix.conf`. Создавать символические ссылки или копировать файлы вручную не требуется.

Откройте файл для проверки и при необходимости измените `listen` (порт) и `server_name` (IP-адрес или DNS-имя вашего сервера):

```bash
sudo nano /etc/nginx/conf.d/zabbix.conf
```

Убедитесь, что директива `listen` раскомментирована и содержит нужный порт (по умолчанию 80), а `server_name` соответствует вашему окружению.

### Шаг 7. Запуск служб и добавление в автозагрузку

```bash
sudo systemctl restart zabbix-server zabbix-agent2 nginx php8.3-fpm
sudo systemctl enable zabbix-server zabbix-agent2 nginx php8.3-fpm
```

Проверьте статус ключевых служб:
```bash
systemctl status zabbix-server --no-pager
systemctl status zabbix-agent2 --no-pager
```

### Шаг 8. Настройка файрвола UFW

```bash
sudo ufw allow 80/tcp
sudo ufw allow 10051/tcp
sudo ufw allow 10050/tcp
sudo ufw reload
```

### Шаг 9. Завершение установки через веб-интерфейс

Откройте браузер и перейдите по адресу `http://<IP-адрес_сервера>`. Запустится веб-мастер установки Zabbix. На шаге настройки базы данных укажите:

- **Database host:** `localhost`
- **Database name:** `zabbix`
- **User:** `zabbix`
- **Password:** пароль, заданный на Шаге 3.

Завершите мастер и войдите в систему. Учётные данные по умолчанию: **Admin** / **zabbix**. Сразу после входа смените пароль.

---

## Вариант 2. Развёртывание в Docker (Docker Compose)

Для быстрого старта можно использовать официальные образы Zabbix в Docker. Важно: файл `docker-compose_v3_ubuntu_pgsql_latest.yaml` из официального репозитория `zabbix-docker` может указывать на тег `latest`, который не гарантирует фиксацию именно версии 7.0 LTS. Для строгого соответствия версии 7.0 рекомендуется либо использовать файл с явным указанием версии (если он присутствует в вашей копии репозитория), либо воспользоваться проверенным сторонним compose-файлом, который развёртывает именно Zabbix 7.0 LTS.

### Шаг 1. Установка Docker Engine и Compose (если не установлены)

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Проверьте установку и добавьте пользователя в группу `docker`:
```bash
docker --version
docker compose version
sudo usermod -aG docker $USER
```
После добавления в группу потребуется перелогиниться.

### Шаг 2. Получение compose-файла для Zabbix 7.0 LTS

Вместо клонирования всего официального репозитория (где compose-файлы могут быть ориентированы на `latest`), можно использовать готовый compose-файл, который явно развёртывает Zabbix 7.0 LTS. Один из поддерживаемых вариантов — репозиторий `heyvaldemar/zabbix-docker-compose`:

```bash
cd /opt
sudo git clone https://github.com/heyvaldemar/zabbix-docker-compose.git
cd zabbix-docker-compose
```

Этот compose-файл развёртывает полный стек Zabbix 7.0 LTS: сервер, веб-интерфейс на Nginx, Agent 2, PostgreSQL и контейнер резервного копирования.

### Шаг 3. Настройка переменных окружения

```bash
cp .env.example .env
sudo nano .env
```

В файле `.env` обязательно задайте пароль для базы данных (переменная `ZABBIX_DB_PASSWORD`). Остальные параметры имеют значения по умолчанию. При необходимости измените часовой пояс или порт веб-интерфейса.

### Шаг 4. Запуск стека контейнеров

```bash
sudo docker compose -f zabbix-docker-compose.yml -p zabbix up -d
```

Первый запуск может занять 1–3 минуты, пока PostgreSQL инициализируется и создаётся схема базы данных.

### Шаг 5. Проверка состояния и доступ к веб-интерфейсу

```bash
sudo docker compose -f zabbix-docker-compose.yml -p zabbix ps
```

Все контейнеры должны перейти в состояние `Up`. Веб-интерфейс будет доступен по адресу `http://<IP-адрес_сервера>/`. Учётные данные по умолчанию: **Admin** / **zabbix**.

### Шаг 6. Управление стеком

- **Остановка:** `sudo docker compose -f zabbix-docker-compose.yml -p zabbix down`
- **Запуск:** `sudo docker compose -f zabbix-docker-compose.yml -p zabbix up -d`
- **Обновление:** используйте скрипт `./update.sh` из репозитория (он обновляет compose-файл до последнего релиза и перезапускает стек).

> **Альтернатива:** Если вы предпочитаете официальный репозиторий `zabbix/zabbix-docker`, проверьте наличие файла с тегом версии в имени (например, содержащего `7.0`) или явно задайте тег образа в файле `.env` (переменная `ZABBIX_IMAGE_TAG`), чтобы избежать неконтролируемого обновления до более новой версии.

---

## Возможные проблемы и способы их решения

| Проблема | Вероятная причина | Решение |
| :--- | :--- | :--- |
| **Zabbix Server не запускается** (`status=1`) | Неверный пароль БД или PostgreSQL недоступен | Проверьте `DBPassword` в `/etc/zabbix/zabbix_server.conf`. Убедитесь, что PostgreSQL запущен: `systemctl status postgresql`. Просмотрите логи: `journalctl -u zabbix-server -n 50`. |
| **Ошибка 502 Bad Gateway** | PHP-FPM не запущен или Nginx не может подключиться к сокету | Проверьте статус `php8.3-fpm`. Убедитесь, что в `/etc/nginx/conf.d/zabbix.conf` сокет PHP-FPM указан верно (обычно `unix:/run/php/php8.3-fpm.sock`). |
| **Файл `server.sql.gz` не найден** | Путь к SQL-скриптам отличается в вашей сборке | Проверьте оба пути: `ls /usr/share/zabbix/sql-scripts/postgresql/` и `ls /usr/share/zabbix-sql-scripts/postgresql/`. Используйте тот, где файл присутствует. |
| **Zabbix Agent недоступен (статус “Not available”)** | Порт 10050 закрыт или агент не запущен | На целевом хосте: `systemctl status zabbix-agent2` и `sudo ufw allow 10050/tcp`. С сервера Zabbix проверьте доступность: `telnet <IP_агента> 10050`. |
| **Контейнер `zabbix-server` в Docker завершается** | База данных ещё не готова или неверный пароль | Подождите 1–2 минуты (PostgreSQL инициализируется не мгновенно). Проверьте логи: `docker logs <имя_контейнера>`. Убедитесь, что `ZABBIX_DB_PASSWORD` в `.env` совпадает с используемым сервером. |
| **Веб-интерфейс Docker недоступен на порту 80** | Порт 80 занят другим процессом | Измените порт в `.env` (например, `ZABBIX_WEB_NGINX_HTTP_PORT=8080`), затем перезапустите стек: `docker compose down && docker compose up -d`. |

---

## Сравнение вариантов развёртывания

| Критерий | Ручная установка (пакеты) | Docker Compose |
| :--- | :--- | :--- |
| **Сложность** | Средняя (требует понимания компонентов) | Низкая (одна команда для запуска) |
| **Гибкость настройки** | Высокая (прямой доступ ко всем конфигам) | Средняя (через переменные окружения) |
| **Обновление** | `apt upgrade` | `./update.sh` или `docker compose pull && up -d` |
| **Изоляция** | Компоненты используют общие системные библиотеки | Полная изоляция в контейнерах |
| **Фиксация версии** | Явно задаётся версией пакета (7.0 LTS) | Требует явного указания тега или использования проверенного compose-файла |
| **Рекомендуемое применение** | Продакшен с тонкой настройкой | Быстрое развёртывание, тестирование, небольшие инфраструктуры |

Оба варианта полностью совместимы с Zabbix 7.0 LTS на Ubuntu 24.04. Для подключения целевых хостов (аналогично вашим `LINUX-NODE-1` и `LINUX-NODE-2`) установите на них пакет `zabbix-agent2`, укажите в `/etc/zabbix/zabbix_agent2.conf` адрес Zabbix Server (`ServerActive=<IP_сервера>`) и запустите службу.
