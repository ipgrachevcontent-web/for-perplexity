# 13 · Operations

> Bare-metal deployment, systemd units, runbooks, maintenance процедуры.
> Инфраструктурная политика (домен, TLS, изоляция от соседей, noindex, выбор IPC) — в `15_DEPLOYMENT_INFRA.md`. Этот документ — **исполнение** этой политики (concrete commands, runbooks).
> Связано: `00_DECISIONS_LOG.md Q5, Q6, Q7, F12–F17`, `02_ARCHITECTURE.md §5`, `12_OBSERVABILITY_SLO.md`, `15_DEPLOYMENT_INFRA.md`.

---

## 1. Hardware и OS

### 1.1 Минимальные требования

| Ресурс | Минимум | Рекомендуется |
|---|---|---|
| CPU | 4 cores | 8 cores |
| RAM | 8 GB | 16 GB |
| Disk | 200 GB SSD | 500 GB NVMe SSD |
| Network | 100 Mbps | 1 Gbps |

**Обоснование:**
- 12 коннекторов асинхронно + PostgreSQL+TimescaleDB + Prometheus+Grafana+Loki + nginx — всё на одной машине.
- БД ~50–80 GB (см. `08_TIME_SERIES_STORAGE.md §8.4`), плюс OS, логи, метрики Prometheus.
- SSD обязателен (TimescaleDB compression — write-heavy при rebalance).

### 1.2 OS

**Ubuntu Server 22.04 LTS** или **24.04 LTS**.

Альтернативы:
- Debian 12 — приемлемо.
- RHEL/Rocky 9 — приемлемо, но systemd-юниты могут потребовать корректировок.

---

## 2. Установка

### 2.1 Пошагово

#### 2.1.1 System packages

```bash
# Обновление
sudo apt update && sudo apt upgrade -y

# Базовые
sudo apt install -y \
    build-essential git curl wget \
    nginx \
    python3.12 python3.12-venv python3.12-dev \
    postgresql-16 postgresql-16-server-dev-all \
    postgresql-contrib-16 \
    nodejs npm
```

#### 2.1.2 TimescaleDB

> **Внимание:** установка требует restart кластера PG, который затрагивает соседние проекты (`detector`, `grach-ege`). **Полная процедура с maintenance window и backup'ом соседей** — `15_DEPLOYMENT_INFRA.md §6.1`. Здесь — только короткая версия для повторных установок.

```bash
# Add TimescaleDB repo (signed-by/keyrings — apt-key deprecated on Ubuntu 22.04+)
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://packagecloud.io/timescale/timescaledb/gpgkey \
    | sudo gpg --dearmor -o /etc/apt/keyrings/timescaledb.gpg
echo "deb [signed-by=/etc/apt/keyrings/timescaledb.gpg] https://packagecloud.io/timescale/timescaledb/ubuntu/ $(lsb_release -cs) main" \
    | sudo tee /etc/apt/sources.list.d/timescaledb.list
sudo apt update
sudo apt install -y timescaledb-2-postgresql-16

# Backup postgresql.conf перед tune
sudo cp /etc/postgresql/16/main/postgresql.conf \
        /etc/postgresql/16/main/postgresql.conf.bak-$(date +%Y%m%d)

# Tune PostgreSQL (изменяет shared_preload_libraries — затрагивает соседей)
sudo timescaledb-tune --quiet --yes --pg-config=/usr/lib/postgresql/16/bin/pg_config

# Restart кластера (downtime ~30-60s для соседей detector + grach-ege)
sudo systemctl restart postgresql@16-main

# Verify
sudo -u postgres psql -tAc \
    "SELECT name, default_version FROM pg_available_extensions WHERE name='timescaledb';"
```

#### 2.1.3 PostgreSQL setup

См. `15_DEPLOYMENT_INFRA.md §4.2` — там полный SQL с правильным OWNER и REVOKE на соседние БД. Краткий вариант:

```bash
sudo -u postgres psql <<EOF
CREATE DATABASE oi_tracker WITH ENCODING = 'UTF8' TEMPLATE = template0;
CREATE ROLE oi_tracker LOGIN PASSWORD '<RANDOM_STRONG_PASS>';
GRANT CONNECT ON DATABASE oi_tracker TO oi_tracker;

\c oi_tracker
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
ALTER SCHEMA public OWNER TO oi_tracker;
GRANT ALL ON SCHEMA public TO oi_tracker;

-- Защита от случайных grant'ов на соседние БД (наша роль).
REVOKE ALL ON DATABASE detector FROM oi_tracker;
REVOKE ALL ON DATABASE grachege FROM oi_tracker;
REVOKE ALL ON DATABASE grachege_shadow FROM oi_tracker;

-- Закрываем CONNECT через PUBLIC (REVOKE FROM <role> сам по себе не помогает,
-- т.к. PUBLIC по умолчанию имеет CONNECT на каждой БД; oi_tracker наследует
-- от PUBLIC). Это требование §10.0 security audit: oi_tracker не должен
-- мочь подключиться к соседним БД даже на уровне CONNECT.
-- Owner-роли соседей (`detector`, `grachege`) сохраняют explicit CTc grant —
-- их сервисы не пострадают.
REVOKE CONNECT ON DATABASE detector FROM PUBLIC;
REVOKE CONNECT ON DATABASE grachege FROM PUBLIC;
REVOKE CONNECT ON DATABASE grachege_shadow FROM PUBLIC;
EOF
```

`pg_hba.conf` — **не трогаем** (в нём уже `host ... 127.0.0.1/32 scram-sha-256` для соседей). Никакого `trust`, никакого открытого 0.0.0.0.

#### 2.1.4 Application user

```bash
# nologin shell — никаких интерактивных входов под этим юзером
sudo useradd -r -s /usr/sbin/nologin -m -d /opt/oi-tracker oi-tracker

sudo mkdir -p /opt/oi-tracker /etc/oi-tracker /var/log/oi-tracker /var/www/oi-tracker/frontend-dist
sudo chown oi-tracker:oi-tracker /opt/oi-tracker /var/log/oi-tracker
sudo chown oi-tracker:oi-tracker /etc/oi-tracker
sudo chown oi-tracker:oi-tracker /var/www/oi-tracker/frontend-dist
sudo chmod 0750 /etc/oi-tracker
sudo chmod 0755 /var/log/oi-tracker
```

`/run/oi-tracker/` создаётся systemd автоматически через `RuntimeDirectory=oi-tracker` в unit-файлах (см. §3).

#### 2.1.5 Backend

```bash
sudo -iu oi-tracker
cd /opt/oi-tracker
git clone <repo_url> code
cd code/backend
python3.12 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install -e .
exit
```

#### 2.1.6 Frontend

```bash
sudo -iu oi-tracker
cd /opt/oi-tracker/code/frontend
pnpm install --frozen-lockfile
pnpm build
# Билд в /opt/oi-tracker/code/frontend/dist
exit

# Копируем в nginx target (root path из site-config)
sudo cp -r /opt/oi-tracker/code/frontend/dist/* /var/www/oi-tracker/frontend-dist/
sudo chown -R oi-tracker:oi-tracker /var/www/oi-tracker/frontend-dist

# robots.txt — статически отдаётся nginx'ом, см. 15_DEPLOYMENT_INFRA.md §3.2
sudo tee /var/www/oi-tracker/frontend-dist/robots.txt <<'EOF'
User-agent: *
Disallow: /
EOF
```

#### 2.1.7 Environment file

```bash
sudo nano /etc/oi-tracker/oi-tracker.env
```

Содержимое:
```
# Database
DATABASE_URL=postgresql+asyncpg://oi_tracker:<PASS>@localhost:5432/oi_tracker

# Telegram
TELEGRAM_BOT_TOKEN=<TG_BOT_TOKEN>

# Logging
LOG_LEVEL=INFO

# Misc
PYTHONUNBUFFERED=1
APP_ENV=production
```

Права:
```bash
sudo chmod 0600 /etc/oi-tracker/oi-tracker.env
sudo chown oi-tracker:oi-tracker /etc/oi-tracker/oi-tracker.env
```

#### 2.1.8 Migrations

```bash
sudo -iu oi-tracker
cd /opt/oi-tracker/code/backend
source .venv/bin/activate
export $(grep -v '^#' /etc/oi-tracker/oi-tracker.env | xargs)
alembic upgrade head
exit
```

#### 2.1.9 Seed данных

```bash
sudo -iu oi-tracker
cd /opt/oi-tracker/code/backend
source .venv/bin/activate
export $(grep -v '^#' /etc/oi-tracker/oi-tracker.env | xargs)

# Seed default 6 alert rules
python -m app.tools.seed_default_rules

# Seed initial settings
python -m app.tools.seed_default_settings
exit
```

#### 2.1.10 systemd units

Скопировать из `deploy/systemd/`:
```bash
sudo cp deploy/systemd/*.service /etc/systemd/system/
sudo cp deploy/systemd/*.timer /etc/systemd/system/
sudo systemctl daemon-reload

# Порядок: сначала scheduler (создаёт runtime dir), потом api, потом tg-sender
sudo systemctl enable --now oi-tracker-scheduler.service
sudo systemctl enable --now oi-tracker-api.service
sudo systemctl enable --now oi-tracker-tg-sender.service
sudo systemctl enable --now oi-tracker-cleanup.timer
```

##### Logrotate

```bash
sudo tee /etc/logrotate.d/oi-tracker <<'EOF'
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
EOF

# Проверить
sudo logrotate -d /etc/logrotate.d/oi-tracker
```

Loki/promtail тащит из `/var/log/oi-tracker/*.log` (см. `12_OBSERVABILITY_SLO.md §5.2`). journald дублирует stdout/stderr для runbooks.

#### 2.1.11 nginx + TLS

Полный nginx-config — `15_DEPLOYMENT_INFRA.md §5`. Шаги установки:

```bash
# 1. Скопировать site-config
sudo cp deploy/nginx/oi-tracker.conf /etc/nginx/sites-available/oi-tracker
sudo ln -s /etc/nginx/sites-available/oi-tracker /etc/nginx/sites-enabled/

# 2. Проверка синтаксиса (важно — соседи robot-detector + grach-ege уже в sites-enabled)
sudo nginx -t

# 3. Reload
sudo systemctl reload nginx

# 4. Smoke-test что соседи живы
curl -fsS https://robot-detector.ru/ -o /dev/null && echo "robot-detector OK"
# и т.п.

# 5. Выпустить TLS-cert для subdomain (см. 15 §2.2)
sudo certbot --nginx -d oi-tracker.robot-detector.ru

# 6. Проверка noindex headers
curl -I https://oi-tracker.robot-detector.ru/ | grep -i 'x-robots-tag'
# Ожидаем: X-Robots-Tag: noindex, nofollow, noarchive, nosnippet
curl -fsS https://oi-tracker.robot-detector.ru/robots.txt
# Ожидаем: User-agent: * / Disallow: /
```

#### 2.1.12 Observability stack

Все артефакты лежат в репо: `deploy/observability/{prometheus,alertmanager,promtail,grafana}`.
Полный пошаговый install + smoke checks — `deploy/observability/README.md`.
Кратко:

```bash
# Prometheus + alert rules
sudo apt install -y prometheus
sudo cp deploy/observability/prometheus/prometheus.yml /etc/prometheus/prometheus.yml
sudo mkdir -p /etc/prometheus/rules
sudo cp deploy/observability/prometheus/rules/oi-tracker.yml /etc/prometheus/rules/oi-tracker.yml
sudo systemctl restart prometheus

# Alertmanager (требует bot_token_file и chat_id — см. README §2 step "Alertmanager")
sudo apt install -y prometheus-alertmanager
sudo install -o alertmanager -g alertmanager -m 0600 /dev/null /etc/oi-tracker/.env-tg-prom-token
# echo "<TG_INFRA_BOT_TOKEN>" | sudo tee /etc/oi-tracker/.env-tg-prom-token
# sudo sed -i 's/chat_id: 0/chat_id: -100XXXXXXXXXX/' deploy/observability/alertmanager/config.yml
sudo cp deploy/observability/alertmanager/config.yml /etc/alertmanager/config.yml
sudo systemctl restart alertmanager

# Loki + Promtail (journald scrape per `12 §5.3`)
sudo apt install -y loki promtail
sudo cp deploy/observability/promtail/config.yml /etc/promtail/config.yml
sudo usermod -aG systemd-journal promtail   # access /var/log/journal
sudo systemctl enable --now loki promtail

# Grafana + provisioning (datasources + dashboards) per `F30` ports
sudo apt install -y grafana
sudo mkdir -p /etc/grafana/provisioning/datasources /etc/grafana/provisioning/dashboards
sudo mkdir -p /etc/grafana/dashboards/oi-tracker
sudo cp deploy/observability/grafana/provisioning/datasources.yml /etc/grafana/provisioning/datasources/oi-tracker.yml
sudo cp deploy/observability/grafana/provisioning/dashboards.yml /etc/grafana/provisioning/dashboards/oi-tracker.yml
sudo cp deploy/observability/grafana/dashboards/*.json /etc/grafana/dashboards/oi-tracker/
sudo systemctl enable --now grafana-server
# Grafana provisioning читается только при старте; после правки provisioning — restart, не reload.
```

После каждого `cp` — `systemctl reload prometheus` / `systemctl restart alertmanager`
для подхвата изменений (Promtail не поддерживает SIGHUP — `restart`).
JSON-дашборды Grafana подхватываются файл-watcher'ом (`updateIntervalSeconds: 30`),
restart нужен только при изменении провайдера / datasources.

Smoke (полный — в `deploy/observability/README.md`):

```bash
curl -sf http://localhost:9090/-/ready && echo "prometheus ready"
curl -sf http://localhost:9093/-/ready && echo "alertmanager ready"
curl -s  http://localhost:9090/api/v1/targets | jq '.data.activeTargets[].labels.job'
```

---

## 3. systemd units

### 3.1 oi-tracker-scheduler.service

```ini
[Unit]
Description=OI Tracker Scheduler (collectors + normalizer + alert engine)
After=network-online.target postgresql.service
Wants=network-online.target postgresql.service

[Service]
Type=notify
User=oi-tracker
Group=oi-tracker
WorkingDirectory=/opt/oi-tracker/code/backend
EnvironmentFile=/etc/oi-tracker/oi-tracker.env
ExecStart=/opt/oi-tracker/code/backend/.venv/bin/python -m oi_tracker.scheduler.main
Restart=always
RestartSec=10
WatchdogSec=120
StartLimitInterval=10min
StartLimitBurst=5

# Resources
MemoryMax=4G
CPUWeight=150
TasksMax=512

# Security
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/log/oi-tracker
PrivateTmp=true

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=oi-tracker-scheduler

[Install]
WantedBy=multi-user.target
```

### 3.2 oi-tracker-api.service

```ini
[Unit]
Description=OI Tracker API (FastAPI + SSE)
After=network-online.target postgresql.service
Wants=network-online.target postgresql.service

[Service]
Type=simple
User=oi-tracker
Group=oi-tracker
WorkingDirectory=/opt/oi-tracker/code/backend
EnvironmentFile=/etc/oi-tracker/oi-tracker.env

# RuntimeDirectory создаёт /run/oi-tracker/ с правами 0750 oi-tracker:oi-tracker
RuntimeDirectory=oi-tracker
RuntimeDirectoryMode=0750

# UDS вместо TCP-порта (порт 8000 занят соседом detector_api, см. 15 §4.1)
ExecStart=/opt/oi-tracker/code/backend/.venv/bin/uvicorn oi_tracker.api.main:app \
    --uds /run/oi-tracker/api.sock \
    --workers 1 \
    --log-config /opt/oi-tracker/code/backend/log_config.json

# Сделать socket доступным для www-data (nginx user) на чтение/запись
ExecStartPost=/bin/bash -c 'chgrp www-data /run/oi-tracker/api.sock && chmod 0660 /run/oi-tracker/api.sock'

Restart=always
RestartSec=5

MemoryMax=2G
CPUWeight=100
TasksMax=512

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/log/oi-tracker /run/oi-tracker

StandardOutput=journal
StandardError=journal
SyslogIdentifier=oi-tracker-api

[Install]
WantedBy=multi-user.target
```

> Если по какой-то причине UDS не работает (отладка / тестовая среда) — fallback TCP на `127.0.0.1:8010` (порт 8000 занят, см. `15 §4.1`). Тогда заменить `--uds /run/...` на `--host 127.0.0.1 --port 8010`, и upstream nginx — на `server 127.0.0.1:8010;`.

### 3.3 oi-tracker-tg-sender.service

```ini
[Unit]
Description=OI Tracker Telegram Sender
After=network-online.target postgresql.service
Wants=network-online.target postgresql.service

[Service]
Type=simple
User=oi-tracker
Group=oi-tracker
WorkingDirectory=/opt/oi-tracker/code/backend
EnvironmentFile=/etc/oi-tracker/oi-tracker.env
ExecStart=/opt/oi-tracker/code/backend/.venv/bin/python -m oi_tracker.tg_sender.main
Restart=always
RestartSec=5

MemoryMax=512M
CPUWeight=50
TasksMax=128

NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/log/oi-tracker

StandardOutput=journal
StandardError=journal
SyslogIdentifier=oi-tracker-tg-sender

[Install]
WantedBy=multi-user.target
```

### 3.4 oi-tracker-cleanup.timer + .service

```ini
# /etc/systemd/system/oi-tracker-cleanup.timer
[Unit]
Description=OI Tracker daily cleanup

[Timer]
OnCalendar=*-*-* 03:15:00 UTC
Persistent=true
Unit=oi-tracker-cleanup.service

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/oi-tracker-cleanup.service
[Unit]
Description=OI Tracker cleanup job
After=postgresql.service

[Service]
Type=oneshot
User=oi-tracker
WorkingDirectory=/opt/oi-tracker/code/backend
EnvironmentFile=/etc/oi-tracker/oi-tracker.env
ExecStart=/opt/oi-tracker/code/backend/.venv/bin/python -m app.cleanup_main
StandardOutput=journal
StandardError=journal
```

---

## 4. nginx config

> **Полный production-config** — `15_DEPLOYMENT_INFRA.md §5`. Включает: TLS (отдельный cert для subdomain), `X-Robots-Tag: noindex, ...`, security headers, UDS upstream, SSE с `proxy_buffering off`. Не дублируем здесь, чтобы не было двух источников истины.

Краткое:
- Site name: `/etc/nginx/sites-available/oi-tracker`.
- server_name: `oi-tracker.robot-detector.ru` (см. `00_DECISIONS_LOG.md F12`).
- Upstream: `unix:/run/oi-tracker/api.sock` (см. `F14`).
- `X-Robots-Tag: noindex, nofollow, noarchive, nosnippet always` (см. `F13`).
- robots.txt отдаётся статически из `/var/www/oi-tracker/frontend-dist/robots.txt`.

> **Важно при rolling update nginx:** перед `reload nginx` обязателен `nginx -t` (соседи `robot-detector` + `grach-ege` уже в `sites-enabled/`; синтаксическая ошибка нашего файла может задержать применение их обновлений).

### 4.1 HTTPS / certbot

Стратегия — **отдельный** Let's Encrypt cert для subdomain (см. `15_DEPLOYMENT_INFRA.md §2.2`, `00_DECISIONS_LOG.md F16`). Существующий cert `robot-detector.ru` — single-domain (не wildcard, проверено `openssl x509 -ext subjectAltName`).

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d oi-tracker.robot-detector.ru
```

Auto-renewal — через системный таймер `certbot.timer` (уже активен для соседнего проекта).

---

## 5. Runbooks

### 5.1 Биржа упала

**Симптом:** Prometheus alert `ConnectorDown` для конкретной биржи.

**Action:**
1. Проверить `oi_collector_errors_total` — какой `error_type`?
2. Проверить логи:
   ```bash
   journalctl -u oi-tracker-scheduler.service -f --grep="exchange=binance"
   ```
3. Проверить, доступна ли биржа извне:
   ```bash
   curl -I https://fapi.binance.com/fapi/v1/ping
   ```
4. Если биржа off — ждём, circuit breaker сам отключит.
5. Если наша проблема (DNS / firewall) — fix и restart scheduler:
   ```bash
   sudo systemctl restart oi-tracker-scheduler.service
   ```
6. Если биржа изменила API → обновить connector + bump `connector_version` + replay при необходимости.

### 5.2 БД упала

**Симптом:** scheduler/api/tg-sender падают; в логах `connection refused`.

**Action:**
1. Проверить:
   ```bash
   sudo systemctl status postgresql
   ```
2. Если down — start:
   ```bash
   sudo systemctl start postgresql
   ```
3. Если start падает — проверить логи:
   ```bash
   sudo journalctl -u postgresql -n 100
   ```
4. Disk full? OOM? Corrupted WAL? — действия по PostgreSQL playbook'у.
5. После восстановления БД — перезапустить наши сервисы:
   ```bash
   sudo systemctl restart oi-tracker-scheduler oi-tracker-api oi-tracker-tg-sender
   ```

### 5.3 TG sender stuck

**Симптом:** Prometheus alert `TGDeliveryBacklog`, `delivery_queue` растёт.

**Action:**
1. Логи TG sender:
   ```bash
   journalctl -u oi-tracker-tg-sender.service -f
   ```
2. Проверить TG bot token (не отозван?):
   ```bash
   curl https://api.telegram.org/bot$TOKEN/getMe
   ```
3. Проверить chat_id (бот всё ещё в чате?):
   ```bash
   curl https://api.telegram.org/bot$TOKEN/sendMessage -d "chat_id=$CHAT&text=test"
   ```
4. Если token revoked — обновить env, restart sender.
5. Если chat_id wrong — обновить через UI Settings.
6. Если spam (TG ratelimit) — ничего не делать, retries отработают.

### 5.4 Disk full

**Симптом:** Prometheus alert `DBDiskUsageHigh`.

**Action:**
1. Проверить:
   ```bash
   df -h /var/lib/postgresql
   ```
2. Размер БД:
   ```bash
   sudo -u postgres psql oi_tracker -c "
     SELECT pg_size_pretty(pg_database_size('oi_tracker'));
   "
   ```
3. Размер chunks:
   ```bash
   sudo -u postgres psql oi_tracker -c "
     SELECT chunk_name,
            pg_size_pretty(pg_total_relation_size(format('%I.%I', chunk_schema, chunk_name)::regclass)) AS sz
     FROM timescaledb_information.chunks
     WHERE hypertable_name='oi_samples'
     ORDER BY range_start DESC LIMIT 20;
   "
   ```
4. Если retention policy не сработал:
   ```bash
   sudo -u postgres psql oi_tracker -c "
     SELECT job_id, schedule_interval, last_run_status
     FROM timescaledb_information.jobs;
   "
   ```
5. Force compress old chunks:
   ```sql
   SELECT compress_chunk(c)
   FROM show_chunks('oi_samples', older_than => INTERVAL '7 days') c
   WHERE c::regclass::text NOT IN (
     SELECT format('%I.%I', chunk_schema, chunk_name)
     FROM timescaledb_information.chunks
     WHERE is_compressed = true AND hypertable_name='oi_samples'
   );
   ```
6. Если предельная ситуация — увеличить disk или временно ужесточить retention до 60 дней:
   ```sql
   SELECT remove_retention_policy('oi_samples');
   SELECT add_retention_policy('oi_samples', INTERVAL '60 days');
   ```

### 5.5 SSE отвалился у пользователя

**Симптом:** UI показывает "disconnected", экран не обновляется.

**Action:**
1. Проверить, работает ли SSE напрямую через UDS (минуя nginx):
   ```bash
   sudo -u oi-tracker curl -N --unix-socket /run/oi-tracker/api.sock \
       http://localhost/sse/v1/live
   # Если включён fallback TCP (см. F14, отладочный режим):
   #   curl -N http://127.0.0.1:8010/sse/v1/live
   ```
   Должен лить events.
2. Проверить nginx (`proxy_buffering off` стоит?):
   ```bash
   sudo nginx -T | grep -A 5 'location /sse'
   ```
3. UI должен auto-reconnect; если не reconnect — refresh браузера.
4. Если массово — bug, проверить API logs.

### 5.6 Фейл нормализации для одной биржи

**Симптом:** Prometheus alert `oi_normalize_errors_total` ↑ для одной биржи.

**Action:**
1. Какой error_type?
   ```bash
   curl 'http://localhost:9090/api/v1/query?query=sum%20by%20(error_type,exchange)%20(rate(oi_normalize_errors_total%5B5m%5D))'
   ```
2. Выборка из БД:
   ```sql
   SELECT error_type, failed_stage, error_message, COUNT(*)
   FROM normalization_errors
   WHERE exchange = 'binance' AND occurred_at >= now() - INTERVAL '15 minutes'
   GROUP BY 1,2,3
   ORDER BY 4 DESC;
   ```
3. Скорее всего:
   - `parse_error` / `schema_drift` → биржа изменила API. Update parser, bump `connector_version` + `normalization_version`.
   - `instrument_unresolved` → новый листинг, sync_job догонит. Если массово — sync_job сломался, restart scheduler.
4. После fix — запустить replay для затронутого периода:
   ```bash
   sudo -iu oi-tracker
   cd /opt/oi-tracker/code/backend
   .venv/bin/python -m app.tools.replay --exchange=binance --since="..." --until="..." --version=NEW_VERSION
   ```

### 5.7 SLO breach (latency P99 > 180s)

**Симптом:** Prometheus alert `E2ELatencyHigh`.

**Action:**
1. Где bottleneck? Гистограмма duration:
   - `oi_collector_cycle_duration_seconds` — медленный fetch?
   - `oi_normalize_duration_seconds` — медленный normalize?
   - `oi_storage_insert_duration_seconds` — медленный insert?
   - `oi_storage_ca_refresh_duration_seconds` — медленный CA refresh?
   - `oi_alert_engine_cycle_duration_seconds` — медленный evaluation?
2. Если `cycle_overlap` events в логах — следующий цикл стартует, не дождавшись предыдущего → перегрузка.
3. Mitigation:
   - Tier-based polling для самой медленной биржи (top symbols every 30s).
   - Увеличить evaluation_cycle_sec с 30 до 60.
   - Отключить divergence rules временно (они ресурсоёмкие).

---

### 5.8 TG chat unhealthy / bot blocked (M7, F37)

**Симптом:** Prometheus alert `OiTgChatUnhealthy` для конкретного `chat_id_native`. Hourly background revalidation две подряд итерации получает не-`ok` от `bot.get_chat()`.

**Diagnose:**
1. Открыть Settings → Telegram. У строки чата виден badge с error kind: `forbidden` / `not_found` / `timeout` / `transient`.
2. Если хочется свежее значение — нажать «Refresh» (читает из БД), либо `curl -s --unix-socket /run/oi-tracker/api.sock http://api/api/v1/chats | jq '.data[] | select(.chat_id_native == "<id>")'`.
3. Кросс-чек на dispatcher: `journalctl -u oi-tracker-tg-sender -n 200 | grep "{chat_id_native}"` — есть ли `tg_chat_blocked_total` инкременты + `TelegramForbiddenError`.

**Remedy:**

| Error kind | Что значит | Что делать |
|---|---|---|
| `forbidden` | Бот удалён из чата / заблокирован админом | Re-add бота в чат → нажать ⭐/Test в UI чтобы запустить sync revalidation. |
| `not_found` | chat_id_native неверный или чат удалён | Если чат удалён — soft-delete в UI; иначе исправить chat_id (пересоздать запись с правильным id). |
| `timeout` / `transient` | Telegram API недоступен | Подождать 1 hourly cycle. Если устойчиво — проверить outbound HTTPS до api.telegram.org с хоста. |

**Если бот реально заблокирован и чат был default'ом:**

1. Создать или промоутить другой чат как default (UI «Set default» или `POST /api/v1/chats/{id}/set-default`).
2. Деактивировать сломанный (`PATCH {is_active: false}` через UI «Delete»).
3. Rules, привязанные к сломанному чату, продолжат работать через fallback на новый default; явно перепривязать через rule editor если нужна другая адресация.

**Не делаем:**
- Auto-deactivate failing chats (по дизайну — F37 §5). Решение про дезактивацию принимает оператор; auto-disable тихо ломает routing у привязанных rules.

---

### 5.9 Default chat missing (M7, F37 + F38)

**Симптом:** Prometheus alert `OiTgChatResolutionFailed` с `reason="no_default"` (rate > 0 за 5m, severity critical).

**Эффект:** Producer'ит каждый FIRE-decision получает `NoDefaultChatError`, **enqueue не происходит**, alert_event сохраняется в audit (`alert_events` hypertable), но Telegram-уведомления теряются. Это критический пробой routing'а.

**Action:**

1. Быстрая проверка — есть ли вообще active default:
   ```bash
   curl -s --unix-socket /run/oi-tracker/api.sock http://api/api/v1/chats \
     | jq '.data[] | select(.is_default and .is_active)'
   ```
   Должна вернуться ровно одна запись.
2. Если default отсутствует:
   - **(штатно) Settings UI** → Telegram → Add chat → Set default. Sync `bot.get_chat()` валидация выполнится автоматически.
   - **(CLI без UI) `POST /api/v1/chats`** + `set-default`:
     ```bash
     # 1. Create the chat (sync-validates via bot.get_chat unless dry-run)
     curl -s --unix-socket /run/oi-tracker/api.sock \
       -X POST http://api/api/v1/chats \
       -H 'Content-Type: application/json' \
       -d '{"chat_id_native": "-1001234567890", "title": "Default"}'
     # response includes "id"; capture it
     # 2. Promote to default
     curl -s --unix-socket /run/oi-tracker/api.sock \
       -X POST http://api/api/v1/chats/<id>/set-default
     ```
   - **(emergency without bot token) Direct DB INSERT** — обходит sync-validation; используется только когда API недоступен и в env стоит `TELEGRAM_DRY_RUN=true`:
     ```sql
     INSERT INTO tg_chats (chat_id_native, title, is_default, is_active)
     VALUES ('-1001234567890', 'Default (manual)', true, true);
     ```
     Restart `oi-tracker-tg-sender` чтобы fallback_chat_id_native перечитался из БД.
3. Если default есть, но `is_active=false` (например, deactivated по ошибке):
   ```bash
   curl -s --unix-socket /run/oi-tracker/api.sock \
     -X PATCH http://api/api/v1/chats/<id> \
     -H 'Content-Type: application/json' \
     -d '{"is_active": true}'
   ```
   Дальше hourly revalidation подтвердит health, метрика вернётся к 1.
4. После восстановления — alert_events за период провала есть в БД, но TG-доставка для них не повторится по дизайну (no retro-fire, F9). Если уведомления критичные — выгрузить из БД и отправить вручную:
   ```sql
   SELECT event_id, ts_processed, exchange, canonical_symbol, reason_human
   FROM alert_events
   WHERE ts_processed > now() - interval '1 hour'
     AND decision = 'fire'
   ORDER BY ts_processed;
   ```

**Prevention:** Settings UI не позволяет удалить default chat без явного re-assign — invariant защищён в `chats_repo.update()` через `DefaultChatProtectedError`. Это правило обходится только raw SQL'ом.

**Историческая заметка (F38):** Раньше существовал legacy alias `PUT /api/v1/settings/tg_chat_id`, который автоматически создавал/обновлял default chat в один curl. Удалён в M7.E (F38) — новые скрипты должны использовать `POST /api/v1/chats` + `set-default`. См. `docs/00_DECISIONS_LOG.md F38`.

### 5.10 Audit-log historical cleanup (one-shot, F45)

**Сценарий:** `alert_events` распух из-за pre-F43 noise (см. `00_DECISIONS_LOG.md F43, F44, F45`). Procedure ниже — **historical record**, выполнен 2026-05-06; не recurring. Документирован для повторных случаев (внешние причины распухания audit-таблицы).

**Pre-flight:**
- F43 (whitelist) ДОЛЖЕН быть задеплоен: `systemctl restart oi-tracker-alert-engine.service` после commit'а `app/domain/alerts.py` + `app/alert_engine/evaluation_cycle.py`. Иначе noise приходит обратно после cleanup'а.
- Verify: `SELECT decision, count(*) FROM alert_events WHERE ts_processed > '<RESTART_TS>' GROUP BY decision;` должен показать только whitelist (fire/suppress/resolve/quality_gate_fail).
- Disk free check: `df -h /var/lib/postgresql`. Если < 30 GB free — обязательно идти через drop_chunks (path B), не через DELETE+compress (path A).
- F44 (columnstore policy) применён: `alembic upgrade head`.

**Procedure (path B — disk-constrained, как был case 2026-05-06):**

```sql
-- 1. Preserve audit (~few thousand rows, ~few MB)
CREATE TABLE alert_events_preserved_audit AS
  SELECT * FROM alert_events
  WHERE decision IN ('fire','suppress','resolve','quality_gate_fail');
SELECT count(*) FROM alert_events_preserved_audit;  -- sanity

-- 2. Drop closed chunks (instant, ~0 WAL)
SELECT drop_chunks('alert_events', older_than => '<HOT_CHUNK_START>'::timestamptz);

-- 3. Restore preserved audit для удалённых дней (TS пересоздаст тонкие чанки)
INSERT INTO alert_events
  SELECT * FROM alert_events_preserved_audit
  WHERE ts_processed < '<HOT_CHUNK_START>'::timestamptz;

-- 4. DELETE noise на hot chunk (большой WAL, проверь disk)
BEGIN;
DELETE FROM alert_events
  WHERE ts_processed >= '<HOT_CHUNK_START>'::timestamptz
    AND ts_processed <  '<HOT_CHUNK_END>'::timestamptz
    AND decision NOT IN ('fire','suppress','resolve','quality_gate_fail');
COMMIT;

-- 5. VACUUM (без FULL — диск возвращается через policy compress на (HOT_CHUNK_END + 7 days))
VACUUM (ANALYZE) alert_events;
```

**Verify after:**
- `SELECT pg_size_pretty(pg_database_size('oi_tracker'));` — упало с pre-cleanup до ожидаемого.
- `SELECT chunk_name, pg_size_pretty(pg_total_relation_size(format('%I.%I', chunk_schema, chunk_name)::regclass)), is_compressed FROM timescaledb_information.chunks WHERE hypertable_name='alert_events' ORDER BY range_start;` — closed chunks тонкие (~kB), hot chunk остался физически большим (dead tuples, reclaim'нутся при следующем policy compress).
- Audit row count match: `SELECT count(*) FROM alert_events;` ≈ count в `alert_events_preserved_audit`.
- alert-engine continues to write only whitelist decisions: `SELECT decision, count(*) FROM alert_events WHERE ts_processed > now() - interval '5 minutes' GROUP BY decision;`.

**Rollback (если что-то пошло не так до COMMIT в шаге 4):**
- Шаги 1–3 reversible через TRUNCATE alert_events + INSERT всех строк из `alert_events_preserved_audit` обратно. Noise rows из drop'нутых чанков утеряны навсегда.
- После COMMIT шаг 4 — noise rows утеряны навсегда. Audit сохранён в sidecar.

**Cleanup:** `DROP TABLE alert_events_preserved_audit;` через ~1 неделю после успешной верификации (safety net).

---

## 6. Reprocessing процедуры

### 6.1 Backfill

**Сценарий:** scheduler был down 30 минут.

```bash
sudo -iu oi-tracker
cd /opt/oi-tracker/code/backend
source .venv/bin/activate
python -m app.tools.backfill \
    --exchanges=binance,bybit,okx \
    --since="2026-04-30 12:00 UTC" \
    --until="2026-04-30 12:30 UTC"
```

NB: некоторые биржи (snapshot-based) не поддерживают backfill — они отдают только current state. Это нормально, gap остаётся.

### 6.2 Replay (новая нормализация)

**Сценарий:** обновлены формулы → bump `normalization_version` 3 → 4.

```bash
# 1. Deploy нового кода (рестарт scheduler)
sudo systemctl restart oi-tracker-scheduler

# 2. Replay
python -m app.tools.replay \
    --exchange=xt \
    --since="2026-04-25" \
    --until="2026-04-30" \
    --version=4
```

NB: replay не создаёт retro-fire алертов (см. `00_DECISIONS_LOG.md F9`).

---

## 7. Deployment workflow

### 7.1 Обычный deploy (новая фича)

```bash
sudo -iu oi-tracker
cd /opt/oi-tracker/code

# 1. Pull new code
git fetch && git pull

# 2. Update backend deps
cd backend
source .venv/bin/activate
pip install -e .

# 3. Migrations
alembic upgrade head

# 4. Build frontend (если есть изменения)
cd ../frontend
pnpm install --frozen-lockfile
pnpm build
# Билд → /var/www/oi-tracker/frontend-dist/ (см. F14, согласовано с §2.1.6)
sudo cp -r dist/* /var/www/oi-tracker/frontend-dist/
sudo chown -R oi-tracker:oi-tracker /var/www/oi-tracker/frontend-dist

exit

# 5. Restart services
sudo systemctl restart oi-tracker-api
sudo systemctl restart oi-tracker-scheduler
sudo systemctl restart oi-tracker-tg-sender
```

### 7.2 Hot-fix только для коннектора

```bash
sudo -iu oi-tracker
cd /opt/oi-tracker/code/backend
git pull
exit

# Только scheduler
sudo systemctl restart oi-tracker-scheduler
```

### 7.3 Rollback

```bash
sudo -iu oi-tracker
cd /opt/oi-tracker/code
git log --oneline -10           # найти коммит до бага
git checkout <good_commit>
exit

# Перезапустить всё
sudo systemctl restart oi-tracker-{api,scheduler,tg-sender}
```

DB migration rollback — отдельная история. Если миграция дропает колонку — rollback через Alembic `downgrade`. Если только added — оставить как есть, новый код просто не использует.

---

## 8. Monitoring health post-deploy

После каждого deploy:
1. Подождать 2 минуты.
2. Проверить health через UDS (см. F14) или через nginx:
   ```bash
   # Прямо в сокет (минуя nginx):
   sudo -u oi-tracker curl -fsS --unix-socket /run/oi-tracker/api.sock \
       http://localhost/api/v1/health
   # Через nginx (так же видит пользователь):
   curl -fsS https://oi-tracker.robot-detector.ru/api/v1/health
   ```
3. Проверить Prometheus targets:
   ```bash
   curl http://localhost:9090/api/v1/targets
   ```
4. Проверить, что новые цикл собрал данные:
   ```sql
   SELECT exchange, MAX(ts_exchange) FROM oi_samples GROUP BY exchange ORDER BY 2 DESC;
   ```
5. Если что-то сломалось — rollback (см. 7.3).

---

## 9. Backups (Future Work — план активации)

В v1 бэкапы **не активированы** (см. `00_DECISIONS_LOG.md Q7`, `01_PRODUCT_SPEC.md C-3`). Этот раздел описывает **готовый план активации** — чтобы при принятии решения «включаем» не было импровизации.

### 9.1 Targets

| Параметр | Target |
|---|---|
| **RPO** (Recovery Point Objective) | ≤ 24 часа (баланс между ценностью accumulated истории и сложностью WAL-G в bare-metal) |
| **RTO** (Recovery Time Objective) | ≤ 2 часа (полное восстановление БД + промежуточных шагов) |
| Что приемлемо потерять | До 24 ч raw `oi_samples` (легко восстанавливается через свежий polling, alert-история теряется) |
| Что НЕ приемлемо потерять | `alert_rules` (пользовательские настройки), `settings`, `instruments` blacklist'ы |

При активации WAL-G — RPO опускается до ≤ 1 часа, RTO остаётся ≤ 2 часа.

### 9.2 pg_dump nightly (минимум)

```bash
# /etc/cron.d/oi-tracker-backup
0 4 * * * postgres pg_dump --format=custom --compress=9 oi_tracker > /var/backups/oi-tracker/oi_tracker-$(date +\%Y\%m\%d).dump
0 5 * * * postgres find /var/backups/oi-tracker/ -name 'oi_tracker-*.dump' -mtime +30 -delete
```

Директория `/var/backups/oi-tracker/` создаётся owner=postgres, mode=0700.

Копировать в **off-host** storage (Backblaze B2, Wasabi, MinIO) — обязательно. Локальный backup при потере диска бесполезен. Команда:

```bash
# В rclone-конфиг добавить remote, далее:
rclone copy /var/backups/oi-tracker/ remote:oi-tracker-backups/ --include 'oi_tracker-*.dump'
```

### 9.3 WAL-G (опционально, для RPO ≤ 1ч)

Continuous WAL archiving для point-in-time recovery:
```bash
# /etc/postgresql/16/main/postgresql.conf
archive_mode = on
archive_command = 'wal-g wal-push %p'
```

**Внимание:** включение `archive_mode` требует restart PG-кластера → maintenance window с уведомлением соседей (см. `15_DEPLOYMENT_INFRA.md §6.1` для аналогичной процедуры).

### 9.4 Restore drill

При активации (любой из вариантов) — **обязательно** провести drill раз в квартал:
1. Создать новую БД `oi_tracker_restore_test`.
2. Восстановить из последнего dump'а: `pg_restore -d oi_tracker_restore_test /var/backups/.../oi_tracker-YYYYMMDD.dump`.
3. Замерить время.
4. Проверить, что `alert_rules`, `settings` восстановились.
5. Удалить test-БД.

Если drill > RTO target — менять стратегию.

### 9.5 Активация в M5 или раньше

См. `16_ROADMAP.md M5` — формальная активация бэкапов по этому плану. Раньше — по решению владельца.

### 9.6 Reference implementation

Рабочие скрипты + cron-stub живут в репо:

```
deploy/backups/
├── pg-dump.sh                          # nightly dump wrapper (flock + verify)
├── restore-drill.sh                    # quarterly drill + RTO budget gate
├── oi-tracker-backup.cron.disabled     # crontab entries (НЕ активны)
└── README.md                           # install order + activation/deactivation
```

`.disabled` суффикс на cron-файле — намеренный сигнал «не активирован»
(`Q7`). Активация = drop суффикс + `install` в `/etc/cron.d/` (см. README §4).

Все скрипты:
- Read-only / scoped — `restore-drill.sh` создаёт throwaway DB
  `oi_tracker_restore_test`, не трогает prod `oi_tracker`.
- Lock-protected — `pg-dump.sh` использует `flock` чтобы overlapping run'ы
  не коллидировали.
- RTO-aware — `restore-drill.sh` enforce'ит budget (default 7200s = 2h
  per §9.1) и возвращает exit 1 при breach.

---

## 10. Security operations

### 10.0 Запреты (production hardening)

Эти правила не имеют исключений в production-коде:

- **`verify=False` в любом исходящем HTTPS запрещён** (см. `00_DECISIONS_LOG.md F14`, `CLAUDE.md §3.5`). Применимо к httpx, aiogram, любым клиентам коннекторов. Если cert у биржи якобы «битый» — это сигнал MITM/подмены, не повод отключать verify. Отлов в CI: `grep -RE 'verify\s*=\s*False' src/` должен возвращать 0 строк.
- **f-string в SQL запрещён.** Только параметризация через SQLAlchemy/asyncpg. Отлов: `ruff` правила `S608`.
- **Логи без secrets.** Перед merge — sample audit: `journalctl -u oi-tracker-* | grep -iE '(token|password|secret|api[_-]?key)'` должен быть пустым.
- **chmod 0600 для env-файлов.** Проверка: `stat -c '%a' /etc/oi-tracker/oi-tracker.env` → `600`.
- **`pg_hba.conf` не содержит `trust`** для нашей роли. Проверка: `grep -E '^[^#]*\boi_tracker\b.*\btrust\b' /etc/postgresql/16/main/pg_hba.conf` → пусто.

### 10.1 Rotate TG bot token

```bash
# 1. Get new token from BotFather
# 2. Update env
sudo nano /etc/oi-tracker/oi-tracker.env
# 3. Restart
sudo systemctl restart oi-tracker-tg-sender
```

### 10.2 Rotate DB password

```bash
sudo -u postgres psql -c "ALTER USER oi_tracker WITH PASSWORD '<NEW_PASS>';"
sudo nano /etc/oi-tracker/oi-tracker.env
sudo systemctl restart oi-tracker-{api,scheduler,tg-sender}
```

### 10.3 OS updates

```bash
sudo apt update && sudo apt upgrade
# Если ядро обновлено — reboot
sudo reboot
```

### 10.4 PostgreSQL major upgrade

Через `pg_upgrade` (отдельный playbook, не входит в обычные операции). Текущая версия 16 — стабильная LTS до 2028.

### 10.5 Pre-deploy security audit playbook

Запускается перед merge в `main` или перед прод-релизом. Делится на
**code-side** (выполняется в репо локально / в CI; ленится через ruff)
и **host-side** (выполняется на проде в maintenance window — пользователь
руками или через `deploy/security/audit.sh`).

#### 10.5.1 Code-side checks

Выполняются в `backend/`. Все должны вернуть указанный результат:

```bash
# 1. SQL injection (S608) — должен быть clean.
#    Существующие `# noqa: S608` exemptions: 2 шт. в repositories,
#    оба whitelist-based WHERE-builders с asyncpg $N binding.
uv run ruff check --select S608 app
# expected: "All checks passed!"

# 2. verify=False в любом исходящем HTTPS — 0 строк.
grep -RnE 'verify\s*=\s*False' app/ tests/
# expected: пусто

# 3. Sanity: f-string SQL, .format() / + concat в DML — 0 строк.
grep -RnE 'f["'\''].*(SELECT |INSERT |UPDATE |DELETE )' app/
grep -RnE '("|'\'')(SELECT|INSERT|UPDATE|DELETE)\b.*("|'\'')\s*\.format\(|("|'\'')(SELECT|INSERT|UPDATE|DELETE)\b.*("|'\'')\s*\+' app/ \
  | grep -v '# noqa: S608'
# expected: пусто

# 4. mypy --strict + ruff check полные.
uv run mypy app
uv run ruff check app tests
# expected: 0 errors
```

**M5.A baseline (зафиксировано 2026-05-02):** все 4 проверки PASS.

#### 10.5.2 Host-side checks

Запускаются как root на проде. Wrapper: `deploy/security/audit.sh` (read-only).

```bash
# 5. Env file ownership + chmod (`F14`).
stat -c '%U:%G %a %n' /etc/oi-tracker/oi-tracker.env
# expected: oi-tracker:oi-tracker 600 /etc/oi-tracker/oi-tracker.env

# 6. Логи без secrets за последние 24h (`F14`).
journalctl --since '24 hours ago' -u 'oi-tracker-*' \
  | grep -iE '(token|password|secret|api[_-]?key)' \
  | grep -vE '(token_file|secret_file)' \
  || echo OK
# expected: OK (с .env-tg-prom-token / bot_token_file false-positives отфильтрованы)

# 7. pg_hba.conf не содержит trust для нашей роли.
grep -E '^[^#]*\boi_tracker\b.*\btrust\b' /etc/postgresql/16/main/pg_hba.conf || echo OK
# expected: OK

# 8. REVOKE на соседние БД (`F14` — изоляция от detector / grach-ege).
sudo -u postgres psql -d detector  -c '\du+ oi_tracker' | grep -i '^oi_tracker' && echo "FAIL: oi_tracker имеет права на detector" || echo OK
sudo -u postgres psql -d grach_ege -c '\du+ oi_tracker' | grep -i '^oi_tracker' && echo "FAIL: oi_tracker имеет права на grach_ege" || echo OK
# expected: OK + OK

# 9. UDS только для api (F14): `oi-tracker-api` не должен слушать публичный TCP.
sudo ss -tlnp | grep -E ':[0-9]+\s.*oi-tracker-api' \
  | grep -vE '127\.0\.0\.1' \
  || echo OK
# expected: OK (метрики на 127.0.0.1:9100/9101/9102/9103 per F30 — допустимо)

# 10. Loopback-only metrics endpoints (F30). Все 4 порта bound to 127.0.0.1.
for port in 9100 9101 9102 9103; do
  sudo ss -tlnp "src 127.0.0.1:$port" | grep -q LISTEN \
    && echo "OK: 9100..9103 loopback ($port up)" \
    || echo "WARN: $port not listening"
done
# expected: 4 × OK
```

**Полный скрипт:** `deploy/security/audit.sh` объединяет все 6 host-checks
с цветным выводом и exit-code-driven результатом (0 = clean, 1 = fail).
Запуск: `sudo bash /var/www/oi-tracker/deploy/security/audit.sh`.

#### 10.5.3 Cadence

| Когда | Проверки |
|---|---|
| Каждый PR | code-side (10.5.1) — через CI |
| Перед merge в main большого изменения (≥50 LOC, новый коннектор, schema change) | code-side + ручной просмотр host-side из maintenance window |
| Раз в квартал | полный code+host audit + ротация `oi_tracker` DB password (`§10.2`) |
| После любого incident'а с подозрением на breach | весь playbook + ротация TG token (`§10.1`) |

---

## 11. Cross-references

- Architecture deployment topology → `02_ARCHITECTURE.md §5`.
- Storage capacity planning → `08_TIME_SERIES_STORAGE.md §8`.
- Reprocessing details → `08_TIME_SERIES_STORAGE.md §6`.
- Observability metrics & alerts → `12_OBSERVABILITY_SLO.md`.
- Test execution → `14_TEST_STRATEGY.md`.
