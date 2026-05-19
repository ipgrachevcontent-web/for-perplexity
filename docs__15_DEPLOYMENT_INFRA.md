# 15 · Deployment Infrastructure

> Политика инфраструктурного развёртывания `oi-tracker` на shared-сервере. Домен, TLS, изоляция от соседних проектов, запрет индексации, выбор IPC. Конкретные команды установки — `13_OPERATIONS.md`.
> Связано: `00_DECISIONS_LOG.md F12–F17`, `13_OPERATIONS.md`, `10_DELIVERY_LAYER.md §7`.

---

## 1. Контекст хост-машины

`oi-tracker` развёртывается на той же физической/виртуальной машине, что и другие проекты владельца. **Ничего из соседей нельзя задеть**: ни порты, ни PG-кластер, ни nginx-конфиги, ни logrotate, ни ресурсы.

### 1.1 Inventory соседних проектов (snapshot 2026-04-30)

| Сосед | Путь | Тип | Порты | nginx site | systemd | PG database |
|---|---|---|---|---|---|---|
| `detector` (robot-detector.ru) | `/var/www/detector` | Go API + Next.js | TCP `127.0.0.1:8000` (API), TCP `0.0.0.0:3001` (Next), `0.0.0.0:3000` (PM2/Go), `0.0.0.0:4000` (PM2/Go) | `robot-detector` | `pm2-root.service`, `detector-watcher.service` | `detector` |
| `grach-ege` | `/var/www/grach-ege` | Next.js + dedicated Redis | TCP `127.0.0.1:6380` (redis), Next через PM2 | `grach-ege` | через `pm2-root.service` | `grachege`, `grachege_shadow` |
| `app` | `/var/www/app` | (неизвестно) | — | — | — | — |
| `html` | `/var/www/html` | nginx default | — | — | — | — |
| общая инфра | — | — | TCP `127.0.0.1:5432` (PG), `127.0.0.1:6379` (redis main), `8123/9000/...` (ClickHouse), `10050` (zabbix) | — | `postgresql@16-main`, `redis-server`, `nginx`, `clickhouse-server`, `zabbix-agent`, `docker`/`containerd` (соседями) | — |

### 1.2 Правила сосуществования

- **NEVER** трогать nginx-конфиги соседей. Свой site-config — только в `/etc/nginx/sites-available/oi-tracker` + symlink в `sites-enabled/`.
- **NEVER** менять глобальные параметры PG (`postgresql.conf`, `pg_hba.conf`) без явного maintenance-window и уведомления (исключение — установка TimescaleDB extension, см. `00_DECISIONS_LOG.md F17`).
- **NEVER** занимать порты, перечисленные в §1.1.
- **NEVER** использовать `root` для процессов `oi-tracker`. Только dedicated user.
- **NEVER** писать в общие пути `/var/log/` без своей поддиректории.

---

## 2. Домен и TLS

### 2.1 Domain (F12)

Production домен — `oi-tracker.robot-detector.ru`. Это subdomain существующего `robot-detector.ru` (соседний проект `detector`).

DNS A-запись на subdomain должна указывать на эту же машину (внешняя задача владельца DNS-зоны).

### 2.2 TLS (F16)

**Отдельный** Let's Encrypt-сертификат через certbot:

```bash
sudo certbot --nginx -d oi-tracker.robot-detector.ru
```

**Почему отдельный, а не multi-domain (`-d robot-detector.ru -d oi-tracker...`):**
- Существующий cert `robot-detector.ru` (выпущен до `2026-06-30`) — **single-domain** (SAN: только `robot-detector.ru`, проверено `openssl x509 -ext subjectAltName`). Не wildcard.
- Multi-domain переиздание свяжет renewal двух проектов; любая операция certbot будет затрагивать соседний site → выше риск.
- Отдельный cert: independent renewal cycle через тот же systemd-таймер `certbot.timer`, нулевая связность с соседом.

HTTP→HTTPS redirect — стандартный, добавляется certbot'ом автоматически.

### 2.3 TLS отказы

`verify=False` в любом исходящем HTTPS — **запрещён** (см. `00_DECISIONS_LOG.md F14`). Применимо к httpx-клиентам коннекторов и aiogram. Все вызовы — с дефолтной cert verification.

---

## 3. Запрет индексации (F13)

`oi-tracker` — single-user приватная утилита. **Не должна индексироваться поисковыми системами**. Защита по трём уровням (defense in depth):

### 3.1 HTTP-заголовок (приоритет)

В nginx site-config для `oi-tracker.robot-detector.ru`:

```nginx
add_header X-Robots-Tag "noindex, nofollow, noarchive, nosnippet" always;
```

`always` — заголовок добавляется в т.ч. при error responses (4xx/5xx), которые иначе отдаются без add_header.

### 3.2 robots.txt

Отдаётся nginx'ом из static директории `/var/www/oi-tracker/frontend-dist/` (см. `F14`):

```
# /var/www/oi-tracker/frontend-dist/robots.txt
User-agent: *
Disallow: /
```

### 3.3 HTML meta

Базовый `index.html` фронтенда содержит:

```html
<meta name="robots" content="noindex, nofollow, noarchive, nosnippet">
```

Vite production build сохраняет meta тег в `dist/index.html`.

### 3.4 Что НЕ делаем

- HTTP Basic Auth не вешаем — единственный пользователь работает напрямую, доп. барьер не оправдан (см. `Q4` в `00_DECISIONS_LOG.md`).
- IP-ограничение через `allow/deny` в nginx — опционально (если владелец захочет — добавить allow-list своих IP).

---

## 4. Изоляция от соседних проектов (F14)

### 4.1 Сетевой уровень: Unix socket вместо TCP-порта

FastAPI слушает на **Unix domain socket** `/run/oi-tracker/api.sock`, не на TCP-порту.

**Почему UDS:**
- Полностью исключает port collision (TCP `8000` уже занят соседом `detector_api`).
- Чуть быстрее TCP loopback (no IP stack overhead).
- Permissions через filesystem (только `oi-tracker` user пишет, `www-data`/nginx читает).
- В systemd через `RuntimeDirectory=oi-tracker` socket автоматически создаётся в `/run/oi-tracker/` с нужными правами.

nginx upstream:
```nginx
upstream oi_tracker_api {
    server unix:/run/oi-tracker/api.sock;
    keepalive 32;
}
```

**Зарезервированный TCP-порт на случай отката:** `127.0.0.1:8010` (свободен, не занят никем). Используется только если UDS вызывает проблемы.

### 4.2 PostgreSQL: общий кластер, изолированные database + role

Используем существующий PG16 кластер (`postgresql@16-main.service`).

```sql
-- Run as superuser postgres
CREATE DATABASE oi_tracker
    WITH OWNER = postgres
         ENCODING = 'UTF8'
         LC_COLLATE = 'en_US.UTF-8'
         LC_CTYPE = 'en_US.UTF-8'
         TEMPLATE = template0;

CREATE ROLE oi_tracker LOGIN PASSWORD '<RANDOM_STRONG>';

-- Минимально необходимые grants — только наша db
GRANT CONNECT ON DATABASE oi_tracker TO oi_tracker;

\c oi_tracker
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Schema-level grants — оставляем oi_tracker владельцем
ALTER SCHEMA public OWNER TO oi_tracker;
GRANT ALL ON SCHEMA public TO oi_tracker;

-- Никаких прав на соседние БД (наша роль).
REVOKE ALL ON DATABASE detector FROM oi_tracker;
REVOKE ALL ON DATABASE grachege FROM oi_tracker;
REVOKE ALL ON DATABASE grachege_shadow FROM oi_tracker;

-- Закрываем CONNECT через PUBLIC. Без этого `oi_tracker` всё ещё может
-- подключиться к соседним БД, потому что PUBLIC по умолчанию имеет CONNECT,
-- а наша роль наследует от PUBLIC. Owner-роли (`detector`, `grachege`)
-- сохраняют explicit CTc-grant, их сервисы не пострадают; superuser
-- `postgres` обходит все ACL.
-- Это требование §10.0 security audit checklist.
REVOKE CONNECT ON DATABASE detector FROM PUBLIC;
REVOKE CONNECT ON DATABASE grachege FROM PUBLIC;
REVOKE CONNECT ON DATABASE grachege_shadow FROM PUBLIC;
```

**`pg_hba.conf`:** не трогаем; новая роль использует существующие правила (`host scram-sha-256` для `127.0.0.1/32`).

### 4.3 OS-уровень: dedicated user + paths

| Ресурс | Путь / имя |
|---|---|
| User / group | `oi-tracker` / `oi-tracker` (system, no-login shell) |
| Home | `/opt/oi-tracker` |
| Code | `/opt/oi-tracker/code` |
| venv | `/opt/oi-tracker/code/backend/.venv` |
| Frontend build target | `/var/www/oi-tracker/frontend-dist` (внутри проекта, не в общем `/var/www/html`) |
| Env-директория | `/etc/oi-tracker/` (`chmod 0750`, **owner `oi-tracker:oi-tracker`** — иначе user не может войти и прочитать env-файл) |
| Env file | `/etc/oi-tracker/oi-tracker.env` (`chmod 0600`, owner `oi-tracker`) |
| Logs | `/var/log/oi-tracker/` (отдельная директория, не корень `/var/log/`) |
| Runtime sockets | `/run/oi-tracker/` (создаётся systemd через `RuntimeDirectory=`) |
| systemd unit prefix | `oi-tracker-*` (api, scheduler, tg-sender, cleanup) |
| nginx site name | `oi-tracker` (в `sites-available/`) |

**`useradd`:**
```bash
sudo useradd -r -s /usr/sbin/nologin -m -d /opt/oi-tracker oi-tracker
```

`-s /usr/sbin/nologin` — никаких интерактивных логинов под этим user.

### 4.4 Resource limits в systemd

Чтобы наш scheduler не задушил соседей при нагрузке:

```ini
# В каждом oi-tracker-*.service
MemoryMax=4G        # scheduler
MemoryMax=2G        # api
MemoryMax=512M      # tg-sender
CPUWeight=100       # default; увеличиваем только scheduler до 150
TasksMax=512
```

Конкретные значения — `13_OPERATIONS.md §3`.

### 4.5 Логи: rotation и размер

**journald per-unit конфиг** (`/etc/systemd/journald.conf.d/oi-tracker.conf`):

```ini
[Journal]
SystemMaxUse=2G
SystemKeepFree=1G
SystemMaxFileSize=200M
MaxRetentionSec=7day
```

Это **глобальный** файл — затрагивает соседей. **Лучшая опция**: не менять журнальную политику глобально, а делать direct logging в `/var/log/oi-tracker/` через `structlog` + logrotate:

```
# /etc/logrotate.d/oi-tracker
/var/log/oi-tracker/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0640 oi-tracker oi-tracker
    postrotate
        systemctl kill --signal=SIGHUP oi-tracker-api.service oi-tracker-scheduler.service oi-tracker-tg-sender.service 2>/dev/null || true
    endscript
}
```

Loki/promtail тащит из `/var/log/oi-tracker/*.log` (label `service=oi-tracker`, см. `12_OBSERVABILITY_SLO.md §5.2`).

journald дублирует stdout/stderr (для `journalctl -u oi-tracker-*` runbooks), но это ограниченное количество — основной поток уходит в файлы.

---

## 5. nginx site-config

Полный конфиг для `oi-tracker.robot-detector.ru` (подставляется certbot'ом после первого выпуска cert):

```nginx
# /etc/nginx/sites-available/oi-tracker
upstream oi_tracker_api {
    server unix:/run/oi-tracker/api.sock;
    keepalive 32;
}

# HTTP → HTTPS redirect (вставляется certbot'ом, показано здесь для полноты)
server {
    listen 80;
    listen [::]:80;
    server_name oi-tracker.robot-detector.ru;

    if ($host = oi-tracker.robot-detector.ru) {
        return 301 https://$host$request_uri;
    }
    return 404;
}

server {
    # Combined `listen ... ssl http2` syntax — works on nginx >=1.9.5.
    # The newer `http2 on;` directive (nginx >=1.25.1) is equivalent and
    # preferred when we move to a host with newer nginx; current Ubuntu 24.04
    # ships nginx 1.24, where `http2 on;` raises `unknown directive`.
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name oi-tracker.robot-detector.ru;

    # TLS — managed by certbot (отдельный cert для subdomain)
    ssl_certificate     /etc/letsencrypt/live/oi-tracker.robot-detector.ru/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/oi-tracker.robot-detector.ru/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam         /etc/letsencrypt/ssl-dhparams.pem;

    # No indexing
    add_header X-Robots-Tag "noindex, nofollow, noarchive, nosnippet" always;

    # Security headers
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "no-referrer" always;
    add_header Strict-Transport-Security "max-age=31536000" always;

    # Static frontend
    root /var/www/oi-tracker/frontend-dist;
    index index.html;

    # Static robots.txt (defense in depth)
    location = /robots.txt {
        try_files /robots.txt =404;
        access_log off;
    }

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://oi_tracker_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 60s;
    }

    location /sse/ {
        proxy_pass http://oi_tracker_api;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;          # SSE: no buffering
        proxy_cache off;
        proxy_read_timeout 24h;       # long-lived stream
        chunked_transfer_encoding on;
    }

    # Метрики и Grafana — НЕ выставляем наружу.
    # Доступ к Prometheus/Grafana — только локально или через SSH-tunnel (см. 12_OBSERVABILITY_SLO.md).

    client_max_body_size 1m;          # API не принимает большие body
}
```

**Важно:** конфиг затрагивает только `server_name oi-tracker.robot-detector.ru`. Соседский `robot-detector` (server_name `robot-detector.ru`) — отдельный server-block, не пересекается.

### 5.1 Frontend deploy (build → frontend-dist)

`vite build` пишет в `frontend/dist/`, а nginx раздаёт `/var/www/oi-tracker/frontend-dist/`. Это **разные директории**, ручная синхронизация — обязательный шаг, иначе браузер видит старый бандл.

**Канонический деплой:**

```bash
cd /var/www/oi-tracker/frontend
npm run deploy
```

Скрипт `npm run deploy` (см. `frontend/package.json`) выполняет:

1. `npm run build` — `tsc --noEmit && vite build` → `frontend/dist/`.
2. `rsync -a --delete --exclude=/.well-known/ --exclude=/robots.txt --chown=oi-tracker:oi-tracker dist/ /var/www/oi-tracker/frontend-dist/`.

**Что сохраняется в `frontend-dist/`:**
- `.well-known/acme-challenge/` — нужно certbot для renew TLS (§2).
- `robots.txt` — defense-in-depth для F13 (§3.2).

**Что не делаем:**
- Не коммитим `frontend-dist/` в git.
- Не правим файлы в `frontend-dist/` руками — следующий `npm run deploy` их перезатрёт.

---

## 6. TimescaleDB установка (F17)

TimescaleDB extension **не установлена** в текущем PG16 (проверено `SELECT * FROM pg_available_extensions WHERE name='timescaledb'` → пусто). Установка требует **restart PG-кластера**, что затрагивает соседей `detector` и `grach-ege`.

### 6.1 Maintenance window protocol

1. **Уведомить владельцев** соседних проектов о предстоящем restart (~30–60s downtime).
2. Снять `pg_dump` для соседей (страховка):
   ```bash
   sudo -u postgres pg_dump detector  | gzip > /root/backups/detector-$(date +%Y%m%d).sql.gz
   sudo -u postgres pg_dump grachege  | gzip > /root/backups/grachege-$(date +%Y%m%d).sql.gz
   sudo -u postgres pg_dump grachege_shadow | gzip > /root/backups/grachege_shadow-$(date +%Y%m%d).sql.gz
   ```
3. Установить пакет (signed-by/keyrings — `apt-key` deprecated на Ubuntu 22.04+):
   ```bash
   sudo mkdir -p /etc/apt/keyrings
   curl -fsSL https://packagecloud.io/timescale/timescaledb/gpgkey \
       | sudo gpg --dearmor -o /etc/apt/keyrings/timescaledb.gpg
   echo "deb [signed-by=/etc/apt/keyrings/timescaledb.gpg] https://packagecloud.io/timescale/timescaledb/ubuntu/ $(lsb_release -cs) main" \
       | sudo tee /etc/apt/sources.list.d/timescaledb.list
   sudo apt update
   sudo apt install -y timescaledb-2-postgresql-16
   ```
4. Применить tune (изменяет `postgresql.conf` — добавляет `shared_preload_libraries = 'timescaledb'`, выставляет memory params):
   ```bash
   sudo cp /etc/postgresql/16/main/postgresql.conf \
           /etc/postgresql/16/main/postgresql.conf.bak-$(date +%Y%m%d)
   sudo timescaledb-tune --quiet --yes --pg-config=/usr/lib/postgresql/16/bin/pg_config
   ```
5. Restart кластера (downtime начинается):
   ```bash
   sudo systemctl restart postgresql@16-main
   ```
6. Verify:
   ```bash
   sudo -u postgres psql -c "SELECT name, default_version, installed_version FROM pg_available_extensions WHERE name='timescaledb';"
   ```
   Должна быть строка с `default_version` ≠ NULL.
7. Post-restart smoke test соседей:
   ```bash
   curl -fsS https://robot-detector.ru/api/health
   # И аналогично grach-ege
   ```
8. Уведомить о завершении.

### 6.2 Откат

Если что-то пошло не так после step 4, до step 5:
```bash
sudo cp /etc/postgresql/16/main/postgresql.conf.bak-<DATE> /etc/postgresql/16/main/postgresql.conf
sudo systemctl restart postgresql@16-main
```
Это вернёт config без `shared_preload_libraries`. Пакет можно оставить установленным — он inert без preload.

### 6.3 Создание extension в нашей БД

После успешного restart:
```sql
\c oi_tracker
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

В соседних БД extension **не** создаётся — нагрузки на их работу нет.

---

## 7. Deployment checklist (один раз перед стартом)

Перед началом разработки кода (см. `16_ROADMAP.md`) должны быть выполнены инфраструктурные шаги:

- [ ] DNS A-запись `oi-tracker.robot-detector.ru` указывает на машину.
- [ ] TimescaleDB установлен в PG16 (см. §6).
- [ ] PG database `oi_tracker` + role `oi_tracker` созданы (см. §4.2).
- [ ] OS user `oi-tracker` создан, директории `/opt/oi-tracker`, `/var/log/oi-tracker`, `/etc/oi-tracker` существуют (см. §4.3).
- [ ] Frontend build target `/var/www/oi-tracker/frontend-dist` существует (placeholder index.html).
- [ ] nginx site `/etc/nginx/sites-available/oi-tracker` создан и symlinked (см. §5).
- [ ] Let's Encrypt cert для subdomain выпущен (см. §2.2).
- [ ] `nginx -t` проходит, `systemctl reload nginx` отрабатывает без ошибок.
- [ ] Соседи `robot-detector.ru` и `grach-ege` всё ещё отвечают `200` после restart PG.
- [ ] `curl -I https://oi-tracker.robot-detector.ru/` возвращает `200` или `404` с `X-Robots-Tag: noindex, nofollow, noarchive, nosnippet`.
- [ ] `curl https://oi-tracker.robot-detector.ru/robots.txt` возвращает `User-agent: * / Disallow: /`.

Только после всех галочек — старт разработки кода.

---

## 8. Cross-references

- Concrete install commands → `13_OPERATIONS.md §2`.
- systemd unit definitions → `13_OPERATIONS.md §3`.
- Decisions: F12–F17 → `00_DECISIONS_LOG.md`.
- UI & SSE → `10_DELIVERY_LAYER.md`.
- Observability paths → `12_OBSERVABILITY_SLO.md`.
- Roadmap (когда какие шаги) → `16_ROADMAP.md`.
