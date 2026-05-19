# M5.A.1 — Metric inventory (Pre-M5 baseline)

> Сгенерировано 2026-05-02 как первый шаг M5.A. Сравнение spec'а
> (`docs/12_OBSERVABILITY_SLO.md §3.1–§3.7`) с реальными emit-точками в
> `backend/app/observability/metrics.py` + кодовая база.

---

## Сводка

| Категория | В спеке | В коде | Emit'ятся | Cheap-to-add | Defer |
|---|---|---|---|---|---|
| Connector | 8 | 8 | 8 | 0 | 1 (`oi_clock_skew_seconds`) |
| Instruments | 6 | 6 | 6 | 0 | 0 |
| Normalizer | 7 | 4 | 4 | 2 (`warnings_total`, `normalization_version_active`) | 1 (`clock_skew`) |
| Storage | 7 | 3 | 3 | 0 | 4 (TS exporter / pg exporter) |
| Alert engine | 9 | 2 | 2 | 6 (см. ниже) | 1 (`alert_state_size`) |
| SSE / API | 5 | 5 | 5 | 0 | 0 |
| Delivery / TG | 9 | 9 | 9 | 0 | 0 |
| **Итого** | **51** | **37** | **37** | **8** | **7** |

Все 37 объявленных метрик имеют рабочие emit-точки (greppable в коде).

---

## Naming aliases (spec ↔ code)

Документация использовала имена-черновики; код устаканил иные. Грат
переименовывать 22 callsite — overkill ради cosmetic; вместо этого
spec `12 §3.1` обновится в **A.4.3** под реальные имена.

| Spec (`12 §3.1`) | Code (`metrics.py`) | Решение |
|---|---|---|
| `oi_collector_fetch_latency_seconds{exchange,endpoint}` | `oi_collector_request_duration_seconds{exchange,endpoint}` | docs → code |
| `oi_collector_errors_total{exchange,error_type}` | `oi_collector_request_total{exchange,endpoint,status}` | docs → code; `status!="ok"` filter заменяет `errors_total` |

---

## Cheap-to-emit (Stage 1)

Добавляем сейчас. Все ≤30 LOC suteach, без новых таблиц / тяжёлых
запросов.

### N1. `oi_warnings_total{exchange,warning_type}` (Counter) — **deferred**

- **Зачем deferred:** проверка показала что `pipeline.py:255` всегда
  emit'ит `warnings=()`. Реального warning-signal в нормализаторе нет
  (только errors через `NormalizationFailure`). Метрика без сигнала —
  пустая. Активируем как только появится первый warning-call site.
- **Spec:** `12 §3.3`.

### N2. `oi_normalization_version_active` (Gauge, без labels)

- **Где emit:** при импорте `app/normalizer/pipeline.py` set'ится в
  `NORMALIZATION_VERSION`.
- **Cardinality:** 1.
- **Spec:** `12 §3.3`.

### A1. `oi_alert_engine_cycle_duration_seconds` (Histogram, без labels)

- **Где emit:** `app/alert_engine/evaluation_cycle.py::run_one_cycle`,
  оборачиваем cycle через `TimedHistogram`.
- **Cardinality:** 1.
- **Spec:** `12 §3.5`.

### A2. `oi_alert_engine_decisions_total{rule_id, decision}` (Counter)

- **Где emit:** `app/alert_engine/evaluation_cycle.py::_evaluate_rule` —
  inc per `TransitionResult.decision` (FIRE/SUPPRESS/QUALITY_GATE_FAIL/...).
- **Cardinality:** 6 rules × 6 decisions = 36. OK.
- **Spec:** `12 §3.5`.

### A3. `oi_alert_engine_state_transitions_total{from_state, to_state}` (Counter)

- **Где emit:** там же, inc по `(existing.state, result.new_state.state)`.
- **Cardinality:** 4 states × 4 states = 16. OK.
- **Spec:** `12 §3.5`.

### A4. `oi_alert_engine_smart_re_fires_total{rule_id}` (Counter)

- **Где emit:** там же, inc когда `result.smart_re_fire is True`.
- **Cardinality:** 6 rules.
- **Spec:** `12 §3.5`. Зависит от поля `smart_re_fire` в `TransitionResult`
  (проверим — есть ли).

### A5. `oi_alert_engine_fires_total{rule_id, exchange, signal_type}` (Counter)

- **Где emit:** при каждом FIRE — inc.
- **Cardinality:** 6 rules × 12 exchanges × 5 signal_types = 360. OK
  (большинство ячеек никогда не fire).
- **Spec:** `12 §3.5`. Дублирует `decisions_total` с decision=fire, но spec
  явно просит и обещает в Grafana panel.

### A6. `oi_alert_quality_gate_fails_total{reason}` (Counter)

- **Где emit:** там же, inc когда decision == QUALITY_GATE_FAIL; reason
  из `TransitionResult.event.reason` или подобного.
- **Cardinality:** ~10 reason strings. OK.
- **Spec:** `12 §3.5`.

---

## Deferred (Stage 1 не делает)

Перенесено в M5.A backlog (после Stage 1) или — если требует внешних
exporter'ов — в M5.B / out-of-scope.

### D1. `oi_clock_skew_seconds{exchange}` (Gauge)

- **Зачем deferred:** требует получать `serverTime`-like endpoint от
  каждой биржи и вычислять skew vs наш wallclock. У 12 коннекторов
  разные API; реализация ≥6h работы.
- **Track:** `M5.A backlog` issue, или отдельный M5.B/M6 task.

### D2. `oi_storage_chunk_count{table}` / `_compressed_chunk_count{table}` /
### `_size_bytes{table,compressed}` (Gauge)

- **Зачем deferred:** TimescaleDB-specific. Лучшее решение —
  `timescaledb-prometheus-exporter` (отдельный systemd-юнит). Эмитить
  из API через периодические `SELECT chunk_*` query — расточительно.
- **Track:** alternative path = deploy timescaledb_exporter в M5.A.2.1
  (prometheus.yml scrape job). M5.A docs обозначат как optional.

### D3. `oi_storage_ca_refresh_duration_seconds{view_name}` (Histogram)

- **Зачем deferred:** continuous aggregates refresh'ятся background-job'ом
  TimescaleDB; наш код в неё не ходит. Метрика — через `pg_stat_progress_*`
  и postgres_exporter.
- **Track:** см. D2 (postgres_exporter scrape job).

### D4. `oi_alert_state_size` (Gauge, без labels)

- **Зачем deferred:** `SELECT count(*) FROM alert_state` каждые N секунд.
  Нужна отдельная background-task в alert_engine. Cheap но требует кода
  не из критпути; добавим в Stage1 если успеваю, иначе — M5.A backlog.
- **Решение:** в Stage 1 **добавить как cheap** (single periodic SELECT
  раз в минуту — overhead negligible).

→ Перенос D4 в cheap-to-emit:

### A7. `oi_alert_state_size` (Gauge)

- **Где emit:** `app/alert_engine/evaluation_cycle.py::run_one_cycle`,
  в начале/конце цикла — `SELECT count(*) FROM alert_state` cheap (table
  обычно <10K rows). set один раз per cycle.
- **Cardinality:** 1.
- **Spec:** `12 §3.5`.

---

## Final Stage 1 todo

| # | Метрика | Файл | LOC est. |
|---|---|---|---|
| ~~N1~~ | ~~`oi_warnings_total`~~ | (deferred — no signal) | — |
| N2 | `oi_normalization_version_active` | `metrics.py`, `normalizer/pipeline.py` | ~5 |
| A1 | `oi_alert_engine_cycle_duration_seconds` | `metrics.py`, `evaluation_cycle.py` | ~5 |
| A2 | `oi_alert_engine_decisions_total{rule_id,decision}` | `metrics.py`, `evaluation_cycle.py` | ~5 |
| A3 | `oi_alert_engine_state_transitions_total{from,to}` | `metrics.py`, `evaluation_cycle.py` | ~5 |
| A4 | `oi_alert_engine_smart_re_fires_total{rule_id}` | `metrics.py`, `evaluation_cycle.py` | ~5 |
| A5 | `oi_alert_engine_fires_total{rule_id,exchange,signal_type}` | `metrics.py`, `evaluation_cycle.py` | ~5 |
| A6 | `oi_alert_quality_gate_fails_total{reason}` | `metrics.py`, `evaluation_cycle.py` | ~5 |
| A7 | `oi_alert_state_size` | `metrics.py`, `evaluation_cycle.py` | ~10 |

**Итого: 8 новых метрик, ~45 LOC изменений, 0 миграций.**

Naming alias renaming: docs only, ~10 LOC в `12 §3.1` (в Stage 4).

Out-of-scope (D1–D3): tracked in todo.md M5.A Review section.
