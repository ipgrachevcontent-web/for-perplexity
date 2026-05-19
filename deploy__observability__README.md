# oi-tracker observability stack

Drop-in configuration for Prometheus + Alertmanager + Loki + Promtail
serving the oi-tracker services. Install order, copy commands, smoke
checks below.

> Spec: `docs/12_OBSERVABILITY_SLO.md`, `docs/13_OPERATIONS.md §2.1.10–§2.1.12`.
> Decision: `docs/00_DECISIONS_LOG.md F30` (TCP-loopback `/metrics`).

## Layout

```
deploy/observability/
├── prometheus/
│   ├── prometheus.yml          # scrape config (4 services + optional exporters)
│   └── rules/
│       └── oi-tracker.yml      # 10 alert rules per `12 §7.2`
├── alertmanager/
│   └── config.yml              # routing → Telegram (infra bot, [INFRA] prefix)
├── promtail/
│   └── config.yml              # journald → Loki, labels per `12 §5.2-5.3`
├── grafana/                    # populated in M5.A Stage 3
└── README.md                   # this file
```

## Port allocation (per F30)

| Service | TCP loopback `/metrics` |
|---|---|
| `oi-tracker-api` | `127.0.0.1:9100` |
| `oi-tracker-scheduler` | `127.0.0.1:9101` |
| `oi-tracker-alert-engine` | `127.0.0.1:9102` |
| `oi-tracker-tg-sender` | `127.0.0.1:9103` |
| `oi-tracker-cleanup` | n/a (oneshot) |

> If you install `node_exporter` later, relocate it to `:9110` —
> default is `:9100`, which collides with `oi-tracker-api`. Set
> `OPTIONS='--web.listen-address=:9110'` in `/etc/default/node_exporter`.

## Install

These commands assume Ubuntu 22.04+ as on the production host
(`docs/13 §2.1.12`). All write to `/etc/<service>/...` and require root.

### 1. Prometheus

```bash
apt install -y prometheus
sudo cp /var/www/oi-tracker/deploy/observability/prometheus/prometheus.yml \
        /etc/prometheus/prometheus.yml
sudo mkdir -p /etc/prometheus/rules
sudo cp /var/www/oi-tracker/deploy/observability/prometheus/rules/oi-tracker.yml \
        /etc/prometheus/rules/oi-tracker.yml
sudo systemctl restart prometheus
# smoke
curl -sf http://localhost:9090/-/ready
curl -s  http://localhost:9090/api/v1/targets | jq '.data.activeTargets[].labels.job'
```

### 2. Alertmanager

```bash
apt install -y prometheus-alertmanager

# Install the bot token (NOT in repo — chmod 0600, owner alertmanager).
sudo install -o alertmanager -g alertmanager -m 0600 /dev/null \
     /etc/oi-tracker/.env-tg-prom-token
sudo bash -c 'echo "<TG_INFRA_BOT_TOKEN>" > /etc/oi-tracker/.env-tg-prom-token'

# Patch chat_id in config.yml (placeholder is `chat_id: 0`).
sudo sed -i 's/chat_id: 0/chat_id: -100XXXXXXXXXX/' \
     /etc/alertmanager/config.yml

sudo cp /var/www/oi-tracker/deploy/observability/alertmanager/config.yml \
        /etc/alertmanager/config.yml
sudo systemctl restart alertmanager
# smoke
curl -sf http://localhost:9093/-/ready
amtool check-config /etc/alertmanager/config.yml
```

### 3. Loki + Promtail

```bash
apt install -y loki promtail
sudo cp /var/www/oi-tracker/deploy/observability/promtail/config.yml \
        /etc/promtail/config.yml
# Promtail needs to read /var/log/journal — add to systemd-journal group:
sudo usermod -aG systemd-journal promtail
sudo systemctl restart promtail
# smoke
curl -s http://localhost:9080/metrics | grep -E 'promtail_(targets|read_lines)'
curl -sG http://localhost:3100/loki/api/v1/labels | jq .
```

### 4. Optional: node_exporter / postgres_exporter / timescaledb_exporter

Uncomment scrape jobs in `prometheus.yml` after install. Postgres exporter
needs a read-only role:

```sql
-- run as `postgres` superuser on the oi_tracker DB
CREATE USER prometheus_ro WITH PASSWORD '<RANDOM>' INHERIT;
GRANT pg_monitor TO prometheus_ro;
```

Then set `DATA_SOURCE_NAME=postgresql://prometheus_ro:<RANDOM>@localhost:5432/oi_tracker?sslmode=disable`
in `/etc/default/postgres_exporter`.

## Smoke checks

After install:

```bash
# 1. Prometheus targets all UP
curl -s http://localhost:9090/api/v1/targets \
  | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'

# 2. Sample query — alert engine cycle latency exists
curl -sG http://localhost:9090/api/v1/query \
     --data-urlencode 'query=oi_alert_engine_cycle_duration_seconds_count' \
  | jq '.data.result[0].metric'

# 3. Loki picks up trace_id (Pre-M5 F29)
curl -sG 'http://localhost:3100/loki/api/v1/query' \
     --data-urlencode 'query={service="scheduler"} | json | trace_id != ""' \
  | jq '.data.result | length'

# 4. Alertmanager firing rules
curl -s http://localhost:9093/api/v2/alerts | jq 'length'
```

## Reload after config edit

```bash
sudo systemctl reload prometheus     # rule file changes
sudo systemctl reload alertmanager   # routing changes
sudo systemctl restart promtail      # promtail doesn't support SIGHUP reload
```

## Backup advice

- `/etc/prometheus/`, `/etc/alertmanager/`, `/etc/promtail/` — back up
  with the rest of the host config.
- Prometheus TSDB at `/var/lib/prometheus/` — *not* backed up (re-derivable
  from scraped sources; 90-day retention is enough).
- Loki chunks at `/var/lib/loki/` — same as above.
- Grafana dashboards — provisioned via files in this repo (not stored
  in Grafana's own SQLite), so source-controlled.
