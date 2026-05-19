# 12 · Observability & SLO

> Метрики, логи, дашборды, алерты, SLO-определения. Полный observability-стек: Prometheus + Grafana + Loki.
> Связано: `00_DECISIONS_LOG.md Q3, Q5`, `01_PRODUCT_SPEC.md NFR-1, NFR-5`.

---

## 1. Цели

- **Видеть, что система работает:** uptime коннекторов, coverage символов, freshness данных.
- **Видеть, когда что-то идёт не так:** Prometheus alerts → Telegram (через Alertmanager в тот же канал, что и market alerts, но с другой меткой).
- **Иметь доказательства SLO:** P95/P99 latency, error rates, coverage ratio.
- **Дебажить problems:** structured logs с trace IDs, audit trail в БД.

---

## 2. Стек

| Компонент | Назначение | Порт |
|---|---|---|
| **Prometheus** | Pull-based метрики, time-series storage метрик | 9090 |
| **Grafana** | Дашборды, визуализация, ad-hoc queries | 3000 |
| **Loki** | Лог-агрегация, structured search | 3100 |
| **Promtail** | Tail systemd journal → Loki | — |
| **Alertmanager** | Routing Prometheus alerts → Telegram | 9093 |

Все компоненты — отдельные systemd-юниты на том же сервере.

---

## 3. Метрики

### 3.1 Connector layer

| Метрика | Тип | Labels | Описание |
|---|---|---|---|
| `oi_collector_cycle_duration_seconds` | histogram | exchange | Длительность одного цикла (включая fetch_prices + fetch_oi) |
| `oi_collector_samples_collected_total` | counter | exchange | Сколько raw_events успешно получено |
| `oi_collector_request_total` | counter | exchange, endpoint, status | Все HTTP-запросы коннектора. `status` ∈ `{ok, http_4xx, http_5xx, timeout, dns_fail, connection_reset, rate_limited, parse_error, unknown}`. Error rate — `sum by (exchange) (rate(oi_collector_request_total{status!="ok"}[5m]))`. |
| `oi_collector_request_duration_seconds` | histogram | exchange, endpoint | Латентность одного HTTP-запроса. |
| `oi_collector_rate_limit_remaining` | gauge | exchange | Текущие токены в bucket |
| `oi_collector_circuit_breaker_state` | gauge | exchange | 0=closed, 1=half_open, 2=open |
| `oi_collector_up` | gauge | exchange | 1 если последний цикл успешен, 0 иначе |
| `oi_collector_symbol_coverage_ratio` | gauge | exchange | `received / active_instruments` за последний цикл |
| `oi_collector_stale_seconds` | gauge | exchange | Сколько секунд прошло с самого свежего `ts_exchange` последнего успешного цикла |

**Примечание (M5.A inventory).** Ранние черновики `12 §3.1` упоминали
`oi_collector_fetch_latency_seconds` и `oi_collector_errors_total` —
эти имена не были выбраны в имплементации. Канонические имена для
Grafana / PromQL — те, что эмитит код (`request_duration_seconds`,
`request_total{status}`).

### 3.2 Instruments registry

| Метрика | Тип | Labels |
|---|---|---|
| `oi_instruments_total` | gauge | exchange, status (active/delisted/blacklisted) |
| `oi_instruments_sync_duration_seconds` | histogram | exchange |
| `oi_instruments_new_total` | counter | exchange |
| `oi_instruments_delisted_total` | counter | exchange |
| `oi_instruments_changed_total` | counter | exchange, change_type |
| `oi_instruments_sync_errors_total` | counter | exchange |

### 3.3 Normalizer

| Метрика | Тип | Labels | Статус |
|---|---|---|---|
| `oi_normalize_duration_seconds` | histogram | exchange, stage | emitted |
| `oi_normalize_total` | counter | exchange, status (ok/error) | emitted |
| `oi_normalize_errors_total` | counter | exchange, error_type, failed_stage | emitted |
| `oi_valuation_status_total` | counter | exchange, status | emitted |
| `oi_normalization_version_active` | gauge | — | **emitted (M5.A)**, set on `pipeline.py` import |
| `oi_warnings_total` | counter | exchange, warning_type | **deferred (M5.A backlog)** — `pipeline.warnings` всегда пуст; активируем при появлении первого warning-call site |
| `oi_clock_skew_seconds` | gauge | exchange | **deferred (M5.A backlog)** — требует per-exchange `serverTime` poll |

### 3.4 Storage

| Метрика | Тип | Labels | Статус |
|---|---|---|---|
| `oi_storage_insert_duration_seconds` | histogram | table | emitted (включая `table="volume_samples_5m"` начиная с F51) |
| `oi_storage_insert_rows_total` | counter | table | emitted (включая `table="volume_samples_5m"` начиная с F51) |
| `oi_db_pool_connections` | gauge | service, state (acquired/idle) | emitted; per-service asyncpg pool — capacity planning per `02 §5.6` |
| `oi_storage_chunk_count` | gauge | table | **deferred** — поступит из `timescaledb_exporter` |
| `oi_storage_compressed_chunk_count` | gauge | table | **deferred** — `timescaledb_exporter` |
| `oi_storage_size_bytes` | gauge | table, compressed (true/false) | **deferred** — `timescaledb_exporter` |
| `oi_storage_ca_refresh_duration_seconds` | histogram | view_name | **deferred** — `pg_stat_progress_*` через `postgres_exporter` |

### 3.5 Alert engine

| Метрика | Тип | Labels | Статус |
|---|---|---|---|
| `oi_alert_engine_cycle_duration_seconds` | histogram | — | emitted (M5.A) |
| `oi_alert_engine_decisions_total` | counter | rule_id, decision | emitted (M5.A) |
| `oi_alert_engine_fires_total` | counter | rule_id, exchange, signal_type | emitted (M5.A) |
| `oi_alert_engine_state_transitions_total` | counter | from_state, to_state | emitted (M5.A) |
| `oi_alert_engine_smart_re_fires_total` | counter | rule_id | emitted (M5.A; detected via `event.reason == "smart_cooldown_re_fire"`) |
| `oi_alert_engine_signal_candidates_total` | counter | rule_type | emitted (M3.E) |
| `oi_alert_e2e_latency_seconds` | histogram | rule_type | emitted (M3.E) |
| `oi_alert_quality_gate_fails_total` | counter | reason | emitted (M5.A; `reason` = `AlertEvent.reason`) |
| `oi_alert_state_size` | gauge | — | emitted (M5.A; `SELECT count(*)` per cycle) |

**`oi_alert_e2e_latency_seconds`** — критическая метрика для SLO. Считается как `ts_processed - ts_exchange` для всех FIRE-решений. Эмиттер: `evaluation_cycle._evaluate_rule` после `delivery_producer.enqueue()`. Для exchange_health bypass-ветки (M3.E, F25) latency = `lag_seconds` (как давно биржа молчит) с тем же rule_type-лейблом.

**`oi_alert_engine_signal_candidates_total`** (M3.E) — счётчик candidates каждого signal handler за цикл. Помогает диагностировать «почему правило не сработало» без обращения к alert_events. Лейбл `rule_type` ∈ {threshold, confirmed, divergence, exchange_health}. _Лейбл `consensus` оставлен в схеме (всегда 0 после `00 F40` sunset) до ревизии Grafana панелей._

### 3.6 Delivery

| Метрика | Тип | Labels |
|---|---|---|
| `oi_tg_sent_total` | counter | template |
| `oi_tg_failed_total` | counter | template, reason |
| `oi_tg_retry_total` | counter | template |
| `oi_tg_pending_size` | gauge | — |
| `oi_tg_oldest_pending_age_seconds` | gauge | — |
| `oi_tg_chat_validation_total` | counter | result |
| `oi_tg_chat_resolution_failed_total` | counter | reason |
| `oi_tg_chat_blocked_total` | counter | chat_id_native |
| `oi_tg_chat_health` | gauge | chat_id, title, chat_id_native |
| `oi_tg_chats_total` | gauge | is_active |
| `oi_api_requests_total` | counter | endpoint, method, status_code |
| `oi_api_request_duration_seconds` | histogram | endpoint |
| `oi_sse_connections` | gauge | channel |
| `oi_sse_events_published_total` | counter | channel, event_type |
| `oi_sse_queue_full_total` | counter | channel |

**Multi-chat routing метрики (M7, F37):**

- `oi_tg_chat_validation_total{result}` — `result ∈ {ok, not_found, forbidden, timeout, transient}`. Инкрементируется на каждый `bot.get_chat()` вызов (sync на create + hourly background revalidation). Анализируется per-result: высокая доля `transient` — Telegram API лагает, `forbidden` — бот удалён из чата, `not_found` — мисс-конфиг chat_id_native.
- `oi_tg_chat_resolution_failed_total{reason}` — `reason ∈ {no_default, inactive, missing}`. Каждый инкремент означает, что producer'у пришлось fall back'нуться или вообще отказать в enqueue. `no_default` — критично (alert не доставлен), `inactive` / `missing` — деградация на default chat.
- `oi_tg_chat_blocked_total{chat_id_native}` — TelegramForbiddenError в dispatcher.send_message. Per-chat счётчик; sustained рост = бот заблокирован в этом чате, нужна ротация default или re-add бота.
- `oi_tg_chat_health{id, title, chat_id_native}` — gauge 1/0 от последней background revalidation. 0 более 10 минут → Prom alert (см. §7.2).
- `oi_tg_chats_total{is_active}` — общее количество чатов в реестре, разрезанное по active flag. Базовый health-сигнал для дашборда.

**`oi_sse_queue_full_total`** (Pre-M4 `F27`) инкрементируется в `SSEHub.publish` при `asyncio.QueueFull` — slow consumer / клиент висит в reconnect loop. Grafana алерт: «>0 за 5 минут на любом channel» — операционный сигнал, что у subscriber'а давление превышает дренаж и часть событий потеряна. Соответствует drop-on-full контракту из `10 §5.4`; пользователь видит это через UI badge `offline` (`18 §7.6`).

### 3.7 System (через node_exporter)

- `node_cpu_seconds_total`, `node_memory_*`, `node_filesystem_*`, `node_network_*` — стандартные.
- `pg_stat_database_*`, `pg_stat_bgwriter_*` (через postgres_exporter).
- `timescaledb_*` (через prometheus exporter for TimescaleDB).

---

## 4. Дашборды Grafana

### 4.1 System overview

**Цель:** «всё ли в порядке прямо сейчас».

**Панели:**
- Status grid: 12 ячеек (по биржам), цветные (green/yellow/red).
- E2E latency P50/P95/P99 (last 1h, real-time).
- Alerts fired per minute (timeseries).
- Coverage ratio (12 lines).
- Disk usage / DB size.
- Memory / CPU per service (api, scheduler, tg-sender).

### 4.2 Exchanges

**Цель:** «как живёт каждая биржа».

**Панели:**
- Per-exchange uptime (`oi_collector_up` aggregate).
- Per-exchange fetch latency.
- Per-exchange error rate.
- Per-exchange symbol coverage.
- Per-exchange clock skew.
- Per-exchange rate limit budget remaining.
- Top errors per exchange.

### 4.3 Alerts

**Цель:** «как работает alert engine».

**Панели:**
- Decisions distribution (FIRE/SUPPRESS/...) per rule (last 24h, stacked).
- Top firing symbols.
- Fires per signal type.
- E2E latency histogram (для SLO).
- Smart re-fire rate.
- TG delivery success rate.

### 4.4 Storage

**Цель:** «как живёт БД».

**Панели:**
- Hypertable size growth (timeseries).
- Compressed vs uncompressed chunks.
- CA refresh duration.
- Slow queries (top 10 by mean time).
- Insert rate per table.
- Connection pool utilization.

### 4.5 Business

**Цель:** «что показывает рынок».

**Панели:**
- Top movers (hourly): top-10 symbols по |Δ_oi_pct|.
- Total OI per exchange (timeseries, для тренда).
- Number of active symbols per exchange.
- Heatmap Δ × symbols × exchanges.

---

## 5. Логи (Loki)

### 5.1 Структурированный формат

```python
import structlog

logger = structlog.get_logger()
logger.info("connector_cycle_complete",
    exchange="binance",
    duration_sec=2.34,
    samples_collected=312,
    cycle_id=str(uuid4()),
)
```

Loki получает JSON-event:
```json
{
  "level": "info",
  "ts": "2026-04-30T12:15:23Z",
  "service": "scheduler",
  "event": "connector_cycle_complete",
  "exchange": "binance",
  "duration_sec": 2.34,
  "samples_collected": 312,
  "cycle_id": "abc123..."
}
```

### 5.2 Labels (low cardinality)

В Loki labels — для индексирования. Кардинальность важна:
- `service`: `api`, `scheduler`, `tg-sender`, `cleanup`.
- `level`: `debug`, `info`, `warn`, `error`.
- `exchange`: 12 значений (приемлемо).

**НЕ labels** (high cardinality):
- request_id — внутри log content, для grep.
- canonical_symbol — внутри content.
- dedupe_key — внутри content.

### 5.3 Promtail config

```yaml
# /etc/promtail/config.yml
server:
  http_listen_port: 9080
positions:
  filename: /var/lib/promtail/positions.yaml
clients:
  - url: http://localhost:3100/loki/api/v1/push
scrape_configs:
  - job_name: oi-tracker-systemd
    journal:
      max_age: 12h
      labels:
        job: oi-tracker
      path: /var/log/journal
    relabel_configs:
      - source_labels: ['__journal__systemd_unit']
        target_label: 'service'
        regex: '(oi-tracker-[a-z-]+)\.service'
        replacement: '$1'
      - source_labels: ['__journal_priority']
        target_label: 'level'
        regex: '(\d+)'
        replacement: '$1'
```

### 5.4 trace_id propagation chain (Pre-M5)

> Закрывает дебажный пробел: «почему этот алерт пришёл с задержкой 87s».
> Один `trace_id` в логе → полная история события за один Loki-запрос.

**Naming convention** (префикс по точке инициации):

| prefix | где генерируется | формат | пример |
|---|---|---|---|
| `cycle-` | `scheduler/loop.py::_run_one_cycle` (один per-exchange tick) | `cycle-{exchange}-{uuid8}` | `cycle-binance-a3f0ee6b` |
| `replay-` | `app/tools/replay.py` (manual replay backfill) | `replay-{uuid8}` | `replay-7f11ab44` |
| `api-` | FastAPI middleware per HTTP request | `api-{uuid8}` | `api-d92c0fe1` |
| `tg-` | `tg_sender/dispatcher.py` fallback when payload._trace_id отсутствует | `tg-{uuid8}` | `tg-5b3e1d77` |

`uuid8` = первые 8 символов `uuid4().hex`. Достаточно для 1-min uniqueness в одном сервисе при текущем cardinality (~12 cycles/min).

**Pipeline propagation:**

```
exchange_tick
    └── scheduler/loop.py::_run_one_cycle
        ├── trace_id = make_trace_id("cycle", exchange)
        ├── bind_trace(cycle_id=trace_id, trace_id=trace_id, exchange=...)
        ├── connector.fetch_*       ◀── inherits via structlog.contextvars
        ├── normalizer.pipeline     ◀── inherits
        ├── oi_samples_repo.bulk_insert   ◀── inherits
        └── (after cycle returns; clear_trace())

alert_engine/evaluation_cycle.py::run_one_cycle
    └── per-rule iteration
        ├── trace_id = make_trace_id("cycle", "alert-engine") at top of cycle
        ├── bind_trace(trace_id=...)
        ├── signal_handler.evaluate     ◀── inherits
        ├── state_machine.transition    ◀── pure, no logging
        ├── delivery_producer.enqueue
        │   └── DeliveryRequest.payload._trace_id = trace_id   ◀── PERSISTED
        └── (after cycle returns; clear_trace())

tg_sender/dispatcher.py::_process_delivery
    ├── trace_id = delivery.payload.get("_trace_id") or make_trace_id("tg")
    ├── bind_trace(trace_id=trace_id, delivery_id=...)
    ├── bot.send_message
    └── mark_sent / mark_failed   ◀── all logs include trace_id

api/main.py middleware
    ├── trace_id = make_trace_id("api")
    ├── bind_trace(trace_id=trace_id, endpoint=..., method=...)
    └── route handler   ◀── inherits
```

**Persistence rule:** между сервисами structlog.contextvars **не передаётся** (разные процессы). Persisted-канал — `delivery_queue.payload._trace_id` (JSONB-поле). tg_sender читает оттуда и rebinds.

**Implementation primitives:**

```python
# app/observability/logging_config.py
def make_trace_id(prefix: Literal["cycle", "replay", "api", "tg"],
                  suffix: str | None = None) -> str:
    short = uuid4().hex[:8]
    if suffix:
        return f"{prefix}-{suffix}-{short}"
    return f"{prefix}-{short}"


def bind_trace(**values: object) -> None:
    """Bind context vars for the current asyncio task chain."""
    structlog.contextvars.bind_contextvars(**values)


def clear_trace() -> None:
    """Clear contextvars at the end of a task to avoid cross-task leakage."""
    structlog.contextvars.clear_contextvars()
```

**Loki: querying by trace_id:**

```
{job="oi-tracker"} | json | trace_id="cycle-binance-a3f0ee6b"
```

Возвращает все события одного cycle через все 4 сервиса (scheduler → alert_engine → tg-sender; api отдельно).

**Future work:** OpenTelemetry / Tempo интеграция отложена. Когда понадобится distributed tracing с спан-деревом (а не log correlation), `trace_id` field reuse'ается как root span ID; `make_trace_id` заменится на `tracer.start_as_current_span().get_span_context().trace_id`. Сейчас — log correlation достаточно для single-host deployment.

### 5.5 Useful Loki queries

**Все ошибки за последний час по бирже:**
```
{service="oi-tracker-scheduler"} |= "error" | json | exchange="binance"
```

**TG retries:**
```
{service="oi-tracker-tg-sender"} | json | event="tg_retry"
```

**Late data events:** per `00_DECISIONS_LOG.md F43` `late_data_skip` — counter-only (в audit-таблицу не пишется). Используй Prometheus вместо Loki:
```promql
sum by (rule_id) (rate(oi_alert_engine_decisions_total{decision="late_data_skip"}[5m]))
```

**Trace по dedupe_key:**
```
{job="oi-tracker"} |= "rule:1|ex:binance|sym:BTC|win:5m"
```

**Trace по trace_id (см. §5.4):**
```
{job="oi-tracker"} | json | trace_id="cycle-binance-a3f0ee6b"
```

---

## 6. SLOs

### 6.1 SLI (Service Level Indicators)

| SLI | Definition | Источник |
|---|---|---|
| **Latency P95** | `histogram_quantile(0.95, oi_alert_e2e_latency_seconds)` | Prometheus |
| **Latency P99** | `histogram_quantile(0.99, oi_alert_e2e_latency_seconds)` | Prometheus |
| **Connector uptime** | `avg_over_time(oi_collector_up[5m])` per exchange | Prometheus |
| **Symbol coverage** | `avg(oi_collector_symbol_coverage_ratio)` per exchange | Prometheus |
| **TG delivery success** | `oi_tg_sent_total / (oi_tg_sent_total + oi_tg_failed_total)` | Prometheus |
| **Normalize success rate** | `1 - oi_normalize_errors_total / oi_normalize_total` | Prometheus |

### 6.2 SLOs

| SLI | SLO target | Window | Action on breach |
|---|---|---|---|
| Latency P95 | ≤ 90s | 1 hour rolling | Investigate если стабильно |
| Latency P99 | ≤ 180s | 1 hour rolling | Alert (Prometheus → TG) |
| Connector uptime | ≥ 99% | 24h rolling | Alert при < 99% |
| Symbol coverage | ≥ 95% | 1h rolling per exchange | Exchange-health alert |
| TG delivery | ≥ 99% | 24h rolling | Alert при < 99% |
| Normalize success | ≥ 99.5% | 1h rolling | Alert при > 0.5% errors |

### 6.3 Error budget

С учётом single-user системы, error budget рассчитывается мягко:
- 99% uptime = ~14 минут downtime в день (приемлемо, single-user может пережить).
- 95% coverage = до 5% символов могут быть потеряны (приемлемо для второстепенных альтов).

Отдельная SLA с заказчиком не нужна — это самообслуживание.

---

## 7. Prometheus Alerts (Alertmanager)

### 7.1 Routing

```yaml
# /etc/alertmanager/config.yml
route:
  receiver: telegram-default
  group_by: [alertname, exchange]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

receivers:
  - name: telegram-default
    telegram_configs:
      - bot_token_file: /etc/oi-tracker/.env-tg-prom-token
        chat_id: <SAME_CHAT_ID_AS_MARKET_ALERTS_OR_DEDICATED>
        message: |
          🚨 *{{ .GroupLabels.alertname }}*
          {{ range .Alerts }}
          • {{ .Labels.exchange }}: {{ .Annotations.summary }}
          {{ end }}
        parse_mode: Markdown
```

NB: Используется отдельный TG bot token (или тот же, но чат-разделение тегом). Чтобы не путать market alerts с infra alerts.

### 7.2 Rules

```yaml
# /etc/prometheus/rules/oi-tracker.yml
groups:
  - name: oi-tracker-collectors
    interval: 30s
    rules:
      - alert: ConnectorDown
        expr: oi_collector_up == 0
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Коннектор {{ $labels.exchange }} не работает 5 минут"

      - alert: ConnectorHighErrorRate
        expr: |
          rate(oi_collector_errors_total[5m]) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.exchange }}: > 0.5 ошибок/сек за последние 10 минут"

      - alert: SymbolCoverageLow
        expr: oi_collector_symbol_coverage_ratio < 0.95
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.exchange }}: coverage {{ $value | humanizePercentage }}"

      - alert: ClockSkewHigh
        expr: abs(oi_clock_skew_seconds) > 30
        for: 5m
        labels:
          severity: info
        annotations:
          summary: "{{ $labels.exchange }}: clock skew {{ $value }}s"

  - name: oi-tracker-storage
    rules:
      - alert: DBDiskUsageHigh
        expr: |
          (node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"} -
           node_filesystem_avail_bytes{mountpoint="/var/lib/postgresql"}) /
           node_filesystem_size_bytes{mountpoint="/var/lib/postgresql"} > 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "DB disk usage {{ $value | humanizePercentage }}"

      - alert: CARefreshFailing
        expr: |
          time() - max(timestamp(oi_storage_ca_refresh_duration_seconds_count{view_name=~"oi_5m|oi_15m|oi_30m"})) > 300
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Continuous aggregate {{ $labels.view_name }} не обновлялся 5+ минут"

  - name: oi-tracker-alerts
    rules:
      - alert: AlertEngineSlow
        expr: histogram_quantile(0.95, oi_alert_engine_cycle_duration_seconds) > 30
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Alert engine cycle P95 > 30s (overload risk)"

      - alert: E2ELatencyHigh
        expr: histogram_quantile(0.99, oi_alert_e2e_latency_seconds) > 180
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "End-to-end latency P99 > 180s (SLO breach)"

      - alert: TGDeliveryBacklog
        expr: oi_tg_oldest_pending_age_seconds > 300
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "TG sender stuck — самая старая pending запись {{ $value }}s"

      - alert: TGDeliveryFailing
        expr: |
          rate(oi_tg_failed_total[5m]) /
          (rate(oi_tg_sent_total[5m]) + rate(oi_tg_failed_total[5m])) > 0.05
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "TG delivery failure rate > 5%"
```

---

## 8. Health endpoint

`GET /api/v1/health` (см. `10_DELIVERY_LAYER.md §4.11`) служит как:
- Liveness check (Kubernetes-style; даже на bare-metal используется в systemd).
- Readiness — для проверки готовности после старта.
- Manual debug — пользователь может посмотреть в браузере.

### 8.1 Liveness в systemd

API слушает Unix socket `/run/oi-tracker/api.sock` (см. `00_DECISIONS_LOG F14`), TCP на `:8000` не используется (порт занят соседом). Поэтому healthcheck из `ExecStartPre` (TCP curl) бессмысленен — сокета ещё нет на момент запуска. Liveness строится из двух механизмов:

**1. Watchdog (основной):**
```
[Service]
Type=notify
NotifyAccess=main
WatchdogSec=60s
```
FastAPI приложение периодически (раз в ~20s) шлёт `sd_notify("WATCHDOG=1")` из background-таски. Если не пришло за `WatchdogSec` — systemd рестартует сервис. Готовность к приёму трафика на старте — `sd_notify("READY=1")` сразу после `uvicorn` опубликовал сокет.

**2. Post-start probe (опционально, для отладки):**
```
[Service]
ExecStartPost=/bin/sh -c 'sleep 2 && curl -f --unix-socket /run/oi-tracker/api.sock http://localhost/api/v1/health/liveness'
```
Проверка через сам UDS (а не TCP). `sleep 2` — даём uvicorn время открыть сокет. Не блокирует service start (`Type=notify` уже подтвердил готовность через `READY=1`); фейл `ExecStartPost` логируется, но не валит сервис.

> Не использовать `ExecStartPre` для health probe — она запускается до `ExecStart`, когда сокета ещё нет. Любая попытка curl в `ExecStartPre` всегда падает.

---

## 9. Audit trail

### 9.1 Где найти "что произошло"

| Вопрос | Где смотреть |
|---|---|
| Когда последний раз сработал алерт BTC 5m? | `alert_events` table, decision='fire', symbol='BTC', window='5m' |
| Почему не сработал ожидаемый алерт? | `alert_events` с decision in ('suppress','quality_gate_fail',...), reason field |
| Что биржа вернула в payload? | `raw_exchange_events` table, фильтр по exchange + ts |
| Почему упал коннектор? | `connector_errors` table |
| Что нормализатор не смог обработать? | `normalization_errors` table |
| Что было в state machine? | `alert_state` table (текущее) + `alert_events` (история) |

### 9.2 Long-term retention

| Таблица | Retention |
|---|---|
| `oi_samples` | 90 дней |
| `oi_5m`, `oi_15m`, `oi_30m`, `oi_quality_5m` | 90 дней |
| `raw_exchange_events` | 14 дней (для replay только) |
| `connector_errors` | 30 дней |
| `normalization_errors` | 30 дней |
| `alert_events` | 90 дней |
| `delivery_queue` | 90 дней (для дебага TG) |

---

## 10. Useful commands

```bash
# Текущий рейт raw events
journalctl -u oi-tracker-scheduler.service -f | grep raw_event_persisted

# E2E latency распределение прямо сейчас
curl -s http://localhost:9090/api/v1/query?query='histogram_quantile(0.95,sum(rate(oi_alert_e2e_latency_seconds_bucket[5m]))by(le))'

# Какие коннекторы down
curl -s http://localhost:9090/api/v1/query?query='oi_collector_up==0'

# DB size
sudo -u postgres psql -c "SELECT pg_size_pretty(pg_database_size('oi_tracker'));"

# Смотреть chunks
sudo -u postgres psql oi_tracker -c "
  SELECT chunk_name, range_start, range_end, is_compressed,
         pg_size_pretty(pg_total_relation_size(format('%I.%I', chunk_schema, chunk_name)::regclass)) AS size
  FROM timescaledb_information.chunks
  WHERE hypertable_name='oi_samples'
  ORDER BY range_start DESC LIMIT 10;
"
```

---

## 11. Cross-references

- SLO definitions → `01_PRODUCT_SPEC.md NFR-1, NFR-2, NFR-3`.
- E2E latency формула → `04_TIME_MODEL.md §4.2`.
- Normalize errors schema → `05_DATA_CONTRACTS.md §14`.
- Connector errors schema → `05_DATA_CONTRACTS.md §13`.
- Alert events schema → `05_DATA_CONTRACTS.md §7`.
- Operations runbooks → `13_OPERATIONS.md §5`.
