# 05 · Data Contracts

> Все события и сущности системы, передаваемые между слоями. Формат: pydantic 2.x классы с автогенерацией JSON Schema. Это единственный авторизованный реестр контрактов; любое расхождение в коде = баг.
> Связано: `00_DECISIONS_LOG.md`, `04_TIME_MODEL.md`, `07_NORMALIZER.md`.

---

## 1. Принципы

### 1.1 Versioning

Каждое сообщение, пересекающее границу слоя, имеет:
- `schema_version: int` — версия структуры контракта.
- `schema_name: str` — имя контракта (`"raw_exchange_event"`, `"normalized_oi_sample"`, ...).

Bump `schema_version` при breaking change в полях. Минорные изменения (новое optional поле) — без bump.

### 1.2 Decimal serialization

Все денежные/количественные поля — `Decimal` в Python, **сериализуются как строки** в JSON для сохранения precision. Pydantic-конфиг включает:

```python
class StrictDecimalConfig:
    json_encoders = {Decimal: lambda v: format(v, "f")}
    arbitrary_types_allowed = True
```

При десериализации — back to `Decimal` через `Decimal(str_value)`.

### 1.3 Timestamp serialization

Все `datetime` поля — `TIMESTAMPTZ` в БД, ISO-8601 в JSON, всегда UTC.

### 1.4 Enums

Все enum'ы — `str, Enum` для удобной сериализации:

```python
class SourceKind(str, Enum):
    SNAPSHOT = "snapshot"
    NATIVE_INTERVAL = "native_interval"
    BACKFILL = "backfill"
    REPLAY = "replay"
    DEGRADED_FALLBACK = "degraded_fallback"
```

### 1.5 Immutability

Все контракты — `frozen=True` (pydantic Config). Изменения = новый объект через `.model_copy(update=...)`.

---

## 2. Pipeline overview

```
┌──────────────────┐
│ exchange API     │  raw HTTP response (JSON / msgpack / etc)
└────────┬─────────┘
         │ Connector
         ▼
┌──────────────────┐
│ RawExchangeEvent │  ← appended to raw_exchange_events
└────────┬─────────┘
         │ Normalizer
         ▼
┌──────────────────┐
│ NormalizedOI     │  ← inserted to oi_samples
│ Sample           │
└────────┬─────────┘
         │ TimescaleDB CA refresh
         ▼
┌──────────────────┐
│ OIBar5m / 15m /  │  ← read-only views (materialized)
│ 30m              │
└────────┬─────────┘
         │ Alert engine
         ▼
┌──────────────────┐
│ AlertCandidate   │  ← in-memory, не персистится
└────────┬─────────┘
         │ State machine
         ▼
┌──────────────────┐
│ AlertEvent       │  ← persisted to alert_events
│ (FIRE/SUPPRESS/  │
│  RESOLVE/        │
│  late_data_skip) │
└────────┬─────────┘
         │ delivery_producer (only if FIRE)
         ▼
┌──────────────────┐
│ DeliveryRequest  │  ← inserted to delivery_queue
└────────┬─────────┘
         │ TG sender
         ▼
┌──────────────────┐
│ DeliveryResult   │  ← updates delivery_queue.status
└──────────────────┘
```

---

## 3. RawExchangeEvent

### 3.1 Назначение

Сырая запись ответа биржи. Хранится append-only. Источник для replay при изменении нормализатора.

### 3.2 Schema

```python
from datetime import datetime
from decimal import Decimal
from typing import Optional, Any
from pydantic import BaseModel, Field

class RawExchangeEvent(BaseModel):
    schema_name: str = "raw_exchange_event"
    schema_version: int = 1

    # Идентификация источника
    exchange: str                          # "binance" | "bybit" | ...
    endpoint: str                          # "/fapi/v1/openInterest"
    request_id: str                        # uuid4 нашей системы (трассировка)

    # Запрос
    request_params: dict[str, Any]         # query params, использованные при fetch
    fetched_at: datetime                   # ts_ingested, server clock
    request_duration_ms: int               # время выполнения HTTP-запроса

    # Ответ
    http_status: int
    response_payload: dict[str, Any] | list[Any]  # raw JSON, как пришло
    raw_hash: str                          # sha256 от canonical JSON repr (для dedupe)

    # Контекст инструмента (для нормализации)
    native_symbol: Optional[str] = None    # если коннектор знает, по какому символу запрос
    instrument_version: Optional[int] = None  # версия instruments registry на момент запроса

    # Источник
    source_kind_intended: str              # "snapshot" | "native_interval" | ...
    connector_version: str                 # для трассировки

    class Config:
        frozen = True
```

### 3.3 Хранение

```sql
CREATE TABLE raw_exchange_events (
    id              BIGSERIAL,
    fetched_at      TIMESTAMPTZ NOT NULL,
    exchange        TEXT        NOT NULL,
    endpoint        TEXT        NOT NULL,
    native_symbol   TEXT,
    request_id      UUID        NOT NULL,
    http_status     SMALLINT    NOT NULL,
    payload_json    JSONB       NOT NULL,
    raw_hash        TEXT        NOT NULL,
    instrument_version INT,
    source_kind_intended TEXT   NOT NULL,
    connector_version TEXT      NOT NULL,
    request_duration_ms INT     NOT NULL,
    PRIMARY KEY (id, fetched_at)
);
SELECT create_hypertable('raw_exchange_events', 'fetched_at',
    chunk_time_interval => INTERVAL '1 day');
SELECT add_compression_policy('raw_exchange_events', INTERVAL '1 day');
SELECT add_retention_policy('raw_exchange_events', INTERVAL '14 days');

CREATE INDEX idx_raw_events_dedupe ON raw_exchange_events (exchange, raw_hash, fetched_at DESC);
CREATE INDEX idx_raw_events_replay ON raw_exchange_events (exchange, native_symbol, fetched_at);
```

NB: `raw_exchange_events` имеет **более короткий retention (14 дней)** — это окно для replay при найденной ошибке нормализации. Дольше держать сырьё дорого по диску, потому что это самые «жирные» строки.

---

## 4. NormalizedOISample

### 4.1 Назначение

Каноническая запись OI. Это то, на чём строятся continuous aggregates, окна, алерты, графики UI.

### 4.2 Schema

```python
class NormalizedOISample(BaseModel):
    schema_name: str = "normalized_oi_sample"
    schema_version: int = 1

    # Идентификация
    exchange: str
    canonical_symbol: str                  # "BTC", не "BTCUSDT"
    native_symbol: str                     # "BTCUSDT" (Binance) | "BTC-USDT-SWAP" (OKX) | ...

    # Market info (для расширения за пределы USDT-perp)
    quote_asset: str = "USDT"
    settle_asset: str = "USDT"
    market_type: str = "usdt_perp"
    contract_type: str = "perpetual"

    # Времена
    ts_exchange: datetime                  # биржевые часы; источник истины
    ts_ingested: datetime                  # когда положили в БД

    # OI и цена
    oi_coins: Decimal                      # NUMERIC(38,18); для Δ
    oi_native_unit: str                    # "BTC" | "contracts" | "USDT" | ...
    oi_notional_usdt: Decimal              # NUMERIC(38,8); display
    price_used: Decimal                    # NUMERIC(38,18); цена для конверсии
    price_source: str                      # "mark" | "index" | "last" | "exchange_provided"

    # Качество и происхождение
    source_kind: str                       # SourceKind enum
    valuation_status: str                  # ValuationStatus enum
    normalization_version: int

    # Трассировка
    raw_event_id: int                      # FK на raw_exchange_events.id
    raw_hash: str                          # копия для удобных JOIN'ов
    instrument_version: int                # версия инструмента, использованная при нормализации

    # Опциональные warning'и. Tuple а не list — `frozen=True` требует
    # hashable типов; код использует `tuple[str, ...]` через
    # `Field(default_factory=tuple)`.
    warnings: tuple[str, ...] = Field(default_factory=tuple)
    # пример warnings: ("clock_skew_forward", "fallback_to_last_price")

    class Config:
        frozen = True
```

### 4.3 Enums

```python
class SourceKind(str, Enum):
    SNAPSHOT = "snapshot"
    NATIVE_INTERVAL = "native_interval"
    BACKFILL = "backfill"
    REPLAY = "replay"
    DEGRADED_FALLBACK = "degraded_fallback"

class ValuationStatus(str, Enum):
    AUTHORITATIVE = "authoritative"
    GOOD_ESTIMATE = "good_estimate"
    LOW_CONFIDENCE = "low_confidence"

class PriceSource(str, Enum):
    MARK = "mark"
    INDEX = "index"
    LAST = "last"
    EXCHANGE_PROVIDED = "exchange_provided"   # биржа вернула цену в payload OI
    NONE = "none"                              # для authoritative notional, цена не нужна
```

### 4.4 Хранение

DDL — в `08_TIME_SERIES_STORAGE.md §2.1`.

---

## 4b. VolumeSample5m (F51, добавлено 2026-05-16)

### 4b.1 Назначение

5-минутная свеча торгового объёма от биржи. На данный момент эмитится **только** `AsterConnector.fetch_5m_volumes` — Aster в окне 5m альертит по объёму, а не OI (см. `00_DECISIONS_LOG F51`). 15m/30m на Aster и все 11 остальных бирж не используют этот контракт.

### 4b.2 Schema

Pydantic v2 (`app.domain.events.VolumeSample5m`): `frozen=True`, `strict=True`, `extra="forbid"`.

| Поле | Тип | Описание |
|---|---|---|
| `schema_name` | `Literal["volume_sample_5m"]` | Идентификатор контракта. |
| `schema_version` | `Literal[1]` | Major version. |
| `exchange` | `Exchange` | `"aster"` (currently the only emitter). |
| `canonical_symbol` | `CanonicalSymbol` | `"LINK"` (BASE per C1). |
| `native_symbol` | `NativeSymbol` | `"LINKUSDT"`. |
| `bucket` | `datetime` (UTC tz-aware) | Kline `openTime`, 5m-aligned. Partition axis hypertable'а. |
| `ts_ingested` | `datetime` (UTC tz-aware) | Wall-clock записи. |
| `base_volume` | `Decimal` ≥ 0 (NUMERIC(38,18)) | Kline `volume` (base asset). |
| `quote_volume_usdt` | `Decimal` ≥ 0 (NUMERIC(38,8)) | Kline `quoteAssetVolume` (USDT notional). |
| `trade_count` | `int \| None` ≥ 0 | Kline `numberOfTrades`. |

UTC-валидатор отвергает naive datetime. `_non_negative` валидатор: отрицательные volume значения → `ValueError`.

### 4b.3 Хранение

DDL — `08_TIME_SERIES_STORAGE.md §3.5`. PK `(exchange, canonical_symbol, bucket)`. UPSERT при повторных опросах running bucket'а каждую минуту (см. `app.storage.repositories.volume_samples.bulk_upsert`).

### 4b.4 Дальнейшее

PR2 добавит `VolumeThresholdSignal`, который читает закрытые бакеты через `get_latest_two_closed_buckets`. PR3 добавит UI-отображение.

---

## 5. OIBar5m / OIBar15m / OIBar30m

### 5.1 Назначение

Materialized continuous aggregate'ы. Не пишутся вручную, а получаются из `oi_samples` через TimescaleDB.

### 5.2 Schema (read-only view)

Соответствуют SQL CA, типизованные на стороне Python:

```python
class OIBar(BaseModel):
    schema_name: str = "oi_bar"
    schema_version: int = 1

    bucket: datetime                       # bucket_start (UTC)
    window: str                            # "5m" | "15m" | "30m"
    exchange: str
    canonical_symbol: str

    # OI агрегаты
    open_oi_coins: Decimal                 # FIRST(oi_coins ORDER BY ts_exchange)
    high_oi_coins: Decimal
    low_oi_coins: Decimal
    close_oi_coins: Decimal

    open_oi_notional_usdt: Decimal
    close_oi_notional_usdt: Decimal

    # Качество
    sample_count: int
    last_ts_exchange: datetime
    worst_valuation_status: str            # min(valuation_status) в bucket'e
    has_degraded_source: bool              # any source_kind == "degraded_fallback"

    class Config:
        frozen = True
```

### 5.3 Расчёт Δ из OIBar

Δ_pct для конкретного `(exchange, canonical_symbol, window)`:

```python
def compute_delta_pct(current: OIBar, prev: OIBar) -> Optional[Decimal]:
    if prev.close_oi_coins == 0:
        return None
    return (current.close_oi_coins - prev.close_oi_coins) / prev.close_oi_coins * Decimal("100")
```

NB: считается по `oi_coins`, не по `oi_notional_usdt` (см. `00_DECISIONS_LOG.md C5`).

---

## 6. AlertCandidate

### 6.1 Назначение

Внутренний контракт alert engine. **Не персистится в БД**, существует только в памяти процесса в момент evaluation cycle.

### 6.2 Schema

```python
class AlertCandidate(BaseModel):
    schema_name: str = "alert_candidate"
    schema_version: int = 1

    # Идентификация
    rule_id: int
    exchange: str
    canonical_symbol: str
    window: str                            # "5m" | "15m" | "30m"

    # Метрики
    current_bar: OIBar
    prev_bar: Optional[OIBar]              # для Δ вычисления
    delta_pct: Optional[Decimal]           # null если prev_bar нет или не свежий
    delta_abs_coins: Optional[Decimal]
    oi_now_notional_usdt: Decimal

    # Контекст для правил
    price_now: Decimal
    price_prev: Optional[Decimal]
    price_delta_pct: Optional[Decimal]

    # Качество
    sample_age_seconds: float
    valuation_status: str
    source_kind: str
    history_minutes: int                   # сколько собственной истории по этому символу

    # Решение state machine на момент кандидата (заполняется позже)
    candidate_ts_processed: datetime

    class Config:
        frozen = True
```

---

## 7. AlertEvent

### 7.1 Назначение

Аудит-лог решений alert engine, требующих ручного разбора. Per `00_DECISIONS_LOG.md F43` пишутся **только** decisions из persistance-whitelist: `FIRE`, `SUPPRESS`, `RESOLVE`, `QUALITY_GATE_FAIL`. Частотные noise-decisions (`late_data_skip`, `insufficient_history`, `below_min_oi`, `not_crossed`, `no_transition`) — counter-only через `oi_alert_engine_decisions_total{rule_id, decision}`. См. `09_ALERT_ENGINE.md §10.1`.

### 7.2 Schema

```python
class AlertEvent(BaseModel):
    schema_name: str = "alert_event"
    schema_version: int = 1

    # Идентификация
    event_id: Optional[int] = None         # PK в БД, None до INSERT (BIGSERIAL)
    rule_id: int
    exchange: str
    canonical_symbol: str
    window: str

    # Времена
    ts_exchange: datetime                  # market time события
    ts_processed: datetime                 # когда engine принял решение

    # Решение
    decision: str                          # AlertDecision enum
    reason: str                            # машино-читаемая причина
    reason_human: str                      # человеко-читаемая причина (для UI)

    # Снимок метрик на момент решения. Все опциональны кроме `threshold`
    # — у NO_TRANSITION / cooldown-tick события may иметь незаполненный
    # snapshot, см. F31.
    delta_pct: Optional[Decimal] = None
    oi_now_notional_usdt: Optional[Decimal] = None
    threshold: Decimal
    sample_age_seconds: Optional[float] = None
    valuation_status: Optional[str] = None  # ValuationStatus enum
    source_kind: Optional[str] = None       # SourceKind enum
    state_before: str                       # AlertStateName
    state_after: str                        # AlertStateName

    # Trace
    candidate_hash: str                    # sha256 от AlertCandidate (для дедупа)

    class Config:
        frozen = True


class AlertDecision(str, Enum):
    # Persisted в alert_events (audit-whitelist, F43):
    FIRE = "fire"                          # отправляем алерт
    SUPPRESS = "suppress"                  # условие true, но не шлём (cooldown / dup)
    RESOLVE = "resolve"                    # сигнал «вернулось в норму» (опционально)
    QUALITY_GATE_FAIL = "quality_gate_fail"  # stale, low_confidence

    # Counter-only (НЕ пишутся в alert_events, F43):
    LATE_DATA_SKIP = "late_data_skip"      # точка пришла слишком поздно
    INSUFFICIENT_HISTORY = "insufficient_history"
    BELOW_MIN_OI = "below_min_oi"
    NOT_CROSSED = "not_crossed"            # value > threshold, но не crossing
    NO_TRANSITION = "no_transition"        # internal: cooldown ticking, no state change


# Persistance whitelist для alert_events (F43, см. 09_ALERT_ENGINE §10.1).
# State machine продолжает выдавать все 9 decisions; counter
# oi_alert_engine_decisions_total{decision} инкрементируется по всем.
AUDIT_PERSISTED_DECISIONS = frozenset({
    AlertDecision.FIRE,
    AlertDecision.SUPPRESS,
    AlertDecision.RESOLVE,
    AlertDecision.QUALITY_GATE_FAIL,
})
```

> `NO_TRANSITION` — внутренний vocabulary state machine; cooldown
> tick не меняет state. Counter-only per F43; в `alert_events` не
> пишется.

### 7.3 Хранение

```sql
CREATE TABLE alert_events (
    event_id     BIGSERIAL,
    ts_processed TIMESTAMPTZ NOT NULL DEFAULT now(),
    ts_exchange  TIMESTAMPTZ NOT NULL,
    rule_id      INT         NOT NULL REFERENCES alert_rules(id),
    exchange     TEXT        NOT NULL,
    canonical_symbol TEXT    NOT NULL,
    window       TEXT        NOT NULL,
    decision     TEXT        NOT NULL,
    reason       TEXT        NOT NULL,
    reason_human TEXT        NOT NULL,
    delta_pct          NUMERIC(18, 6),
    oi_now_notional_usdt NUMERIC(38, 8),
    threshold    NUMERIC(18, 6),
    sample_age_seconds NUMERIC(10, 3),
    valuation_status   TEXT,
    source_kind        TEXT,
    state_before TEXT,
    state_after  TEXT,
    candidate_hash TEXT,
    PRIMARY KEY (event_id, ts_processed)
);
SELECT create_hypertable('alert_events', 'ts_processed',
    chunk_time_interval => INTERVAL '1 day');
SELECT add_retention_policy('alert_events', INTERVAL '90 days');

CREATE INDEX idx_alert_events_lookup ON alert_events (rule_id, exchange, canonical_symbol, window, ts_processed DESC);
CREATE INDEX idx_alert_events_decision ON alert_events (decision, ts_processed DESC);
```

### 7.4 TransitionResult — in-memory only

```python
class TransitionResult(BaseModel):
    """Output one transition в state_machine. NOT persisted (transport-only).

    Bundles new state (для upsert) + audit event (для bulk_append) +
    convenience flag `fire`. evaluation_cycle.py делает I/O; state_machine
    остаётся pure function.
    """
    schema_name: str = "transition_result"
    schema_version: int = 1

    new_state: AlertState                  # см. §9
    event: AlertEvent                      # см. §7
    fire: bool                             # True iff event.decision == FIRE

    class Config:
        frozen = True
```

---

## 8. AlertRule

### 8.1 Назначение

Конфигурация одного правила алертов. Структурированная таблица, не KV.

### 8.2 Schema

```python
class AlertRule(BaseModel):
    schema_name: str = "alert_rule"
    schema_version: int = 1

    id: int
    name: str                              # human-readable, "BTC pump 5m default"
    rule_type: str                         # RuleType enum

    # Scope
    scope: str                             # "global" | "exchange" | "symbol"
    exchange_filter: list[str]             # пустой = все
    symbol_filter: list[str]               # пустой = все

    # Метрика
    window: str                            # "5m" | "15m" | "30m"
    metric: str                            # "delta_oi_pct" | "delta_price_pct" | ...
    operator: str                          # ">=" | "<=" | "=="
    threshold: Decimal                     # знаковый: +5 для роста, -5 для падения

    # Фильтры качества
    min_oi_notional_usdt: Decimal          # default 5_000_000
    min_history_minutes: Optional[int]     # null → формула max(30, 2*window)
    valuation_status_min: str              # default "good_estimate"
    source_kind_filter: list[str]          # default ["snapshot", "native_interval"], исключаем degraded_fallback из боевых

    # State machine параметры
    confirm_points: int                    # default 1
    cooldown_sec: int                      # default 900 (15 min)
    smart_cooldown_factor: Decimal         # default 1.5

    # Доставка
    enabled: bool                          # default true
    chat_id: Optional[int]                 # FK → tg_chats.id; NULL = use default chat (F37)

    # Signal-type specific параметры (per F24): JSONB, runtime-validated
    # сигналом. Examples: {"divergence_price_threshold": "1.0"} для
    # DIVERGENCE, {"sub_trigger": "stale_data"} для EXCHANGE_HEALTH.
    # (CONSENSUS / consensus_count sunset per F40.)
    extra: dict[str, Any]                  # default {}

    # Аудит
    created_at: datetime
    updated_at: datetime


class RuleType(str, Enum):
    THRESHOLD = "threshold"
    CONFIRMED = "confirmed"
    DIVERGENCE = "divergence"
    EXCHANGE_HEALTH = "exchange_health"
    # CONSENSUS sunset per F40 (M8 Phase A).
```

### 8.3 Хранение

```sql
CREATE TABLE alert_rules (
    id              SERIAL PRIMARY KEY,
    name            TEXT NOT NULL,
    rule_type       TEXT NOT NULL,
    scope           TEXT NOT NULL,
    exchange_filter TEXT[]   DEFAULT '{}',
    symbol_filter   TEXT[]   DEFAULT '{}',
    window          TEXT     NOT NULL,
    metric          TEXT     NOT NULL,
    operator        TEXT     NOT NULL,
    threshold       NUMERIC(18, 6) NOT NULL,
    min_oi_notional_usdt NUMERIC(38, 8) DEFAULT 5000000,
    min_history_minutes  INT,
    valuation_status_min TEXT DEFAULT 'good_estimate',
    source_kind_filter   TEXT[] DEFAULT ARRAY['snapshot','native_interval'],
    confirm_points       SMALLINT DEFAULT 1,
    cooldown_sec         INT DEFAULT 900,
    smart_cooldown_factor NUMERIC(4, 2) DEFAULT 1.5,
    enabled         BOOLEAN  DEFAULT true,
    chat_id         INT NULL REFERENCES tg_chats(id) ON DELETE RESTRICT, -- F37; NULL = default
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_alert_rules_enabled ON alert_rules (enabled, rule_type);
```

### 8.4 Базовые seed-rows (6 порогов из ТЗ)

См. `09_ALERT_ENGINE.md §7` для конкретных INSERT'ов.

---

## 8a. TgChat (multi-chat routing)

### 8a.1 Назначение

Реестр Telegram-чатов, в которые доставляются алерты. Введён в M7 per `00_DECISIONS_LOG.md F37`. Заменяет single-value `settings.tg_chat_id` (последний остаётся deprecated alias на ≥M8 — см. §12).

`alert_rules.chat_id` ссылается на `tg_chats.id`; `NULL` означает «использовать default». На момент enqueue producer резолвит routing и сохраняет destination как снапшот в `delivery_queue.chat_id_native` (см. §10.4).

### 8a.2 Schema

```python
class TgChat(BaseModel):
    schema_name: str = "tg_chat"
    schema_version: int = 1

    id: int
    chat_id_native: str                    # Telegram chat id, e.g. "-1001234567890"
    title: str                             # human label, 1..200 chars
    is_default: bool                       # ровно одна запись с true (partial unique index)
    is_active: bool                        # soft-delete; default fallback пропускает inactive

    last_validated_at: Optional[datetime]  # ts последнего успешного getChat
    last_validation_error: Optional[str]   # 'not_found' | 'forbidden' | 'timeout' | 'transient' | None

    created_at: datetime
    updated_at: datetime

    class Config:
        frozen = True
```

### 8a.3 Хранение

```sql
CREATE TABLE tg_chats (
    id                    SERIAL PRIMARY KEY,
    chat_id_native        TEXT NOT NULL UNIQUE,
    title                 TEXT NOT NULL,
    is_default            BOOLEAN NOT NULL DEFAULT false,
    is_active             BOOLEAN NOT NULL DEFAULT true,
    last_validated_at     TIMESTAMPTZ,
    last_validation_error TEXT,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Гарантирует ровно один is_default=true на уровне БД (не приложения).
CREATE UNIQUE INDEX uq_tg_chats_one_default ON tg_chats (is_default)
    WHERE is_default = true;

CREATE INDEX idx_tg_chats_active ON tg_chats (is_active)
    WHERE is_active = true;
```

### 8a.4 Backfill

Migration `0012` создаёт ровно один `is_default = true, is_active = true` row из `settings_kv.tg_chat_id` (если установлен на момент upgrade). Существующие `alert_rules` получают `chat_id = NULL` и автоматически идут в default — поведение single-chat сценария не меняется.

### 8a.5 Жизненный цикл

| Состояние | Триггер | Эффект |
|---|---|---|
| `is_active=true, is_default=true` | один на БД, обязателен | используется как fallback, если `rule.chat_id` NULL/inactive |
| `is_active=true, is_default=false` | назначается через UI | rule может ссылаться через `alert_rules.chat_id` |
| `is_active=false` | soft-delete через UI | пропускается в resolve, FK на rule препятствует hard-delete (`ON DELETE RESTRICT`) |

`set_default(id)` атомарно переключает дефолт: в одной транзакции `UPDATE ... SET is_default=false WHERE is_default = true` + `UPDATE ... SET is_default=true WHERE id=$1`. Partial unique index гарантирует corrupt-state импоссибл.

### 8a.6 Validation

Sync-валидация на create через `bot.get_chat(chat_id_native)` с 5s timeout. Маппинг ошибок: `not_found` (chat не существует / бот вне) → 400; `forbidden` (бот заблокирован) → 400; `timeout` / `transient` → 400 с подсказкой повторить.

Background revalidation hourly (`tg_chat_revalidation_interval_sec`, default 3600) с jitter ±300s обходит все active-чаты, обновляет `last_validated_at` / `last_validation_error`. Метрика `oi_tg_chat_health{id, title, chat_id_native}` (gauge 1/0) рендерится в Grafana panel.

---

## 8b. TgRoutingGroup (per-exchange routing, F41)

### 8b.1 Назначение

Группа = (имя, чат, optional `message_thread_id`, набор бирж). Resolver сопоставляет `candidate.exchange` с группой; биржа максимум в одной группе (`UNIQUE` на join). Поверх F37 (per-rule routing) и не отменяет его — `rule.chat_id` остаётся explicit override.

### 8b.2 Schema

```python
class TgRoutingGroup(BaseModel):
    schema_name: str = "tg_routing_group"
    schema_version: int = 1

    id: int
    name: str                              # 1..128, UNIQUE
    chat_id: int                           # FK → tg_chats.id
    message_thread_id: Optional[int]       # BIGINT NULL (Telegram forum topic id)
    is_active: bool

    exchanges: tuple[str, ...]             # normalised: stripped, lowercased,
                                           # deduplicated, sorted

    last_send_error: Optional[str]         # snapshot последней permanent
    last_send_error_at: Optional[datetime] # ошибки отправки (F41 async-валидация)

    created_at: datetime
    updated_at: datetime


class RouteResolution(NamedTuple):
    """Producer's snapshot of one routing decision (snapshotted into
    delivery_queue.chat_id_native + .message_thread_id at FIRE-time)."""
    chat_id_native: str
    message_thread_id: Optional[int]
```

### 8b.3 Хранение

```sql
CREATE TABLE tg_routing_groups (
    id                  INT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
    name                TEXT NOT NULL UNIQUE,
    chat_id             INT NOT NULL REFERENCES tg_chats(id) ON DELETE RESTRICT,
    message_thread_id   BIGINT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT true,
    last_send_error     TEXT NULL,
    last_send_error_at  TIMESTAMPTZ NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_tg_routing_groups_chat_id ON tg_routing_groups (chat_id);
CREATE INDEX idx_tg_routing_groups_active  ON tg_routing_groups (is_active) WHERE is_active = true;

CREATE TABLE tg_routing_group_exchanges (
    group_id  INT NOT NULL REFERENCES tg_routing_groups(id) ON DELETE CASCADE,
    exchange  TEXT NOT NULL,
    PRIMARY KEY (group_id, exchange),
    UNIQUE (exchange)                     -- биржа максимум в одной группе
);
CREATE INDEX idx_tg_routing_group_exchanges_exchange ON tg_routing_group_exchanges (exchange);
```

`UNIQUE(exchange)` — ключевой invariant: resolver делает одну индексную lookup по `exchange` без агрегации, и API на конфликт returns `422` с echo offending списка.

### 8b.4 Resolver order

См. `00_DECISIONS_LOG F41` + `10 §2.6`. На FIRE-time:

1. `rule.chat_id` (explicit override, F37) — если задан и чат активен.
2. `tg_routing_groups.find_by_exchange(exchange)` — если group активна и её chat активен.
3. Default chat (`tg_chats WHERE is_default=true`) — fallback.
4. Иначе `NoDefaultChatError` → producer skip.

Каждый fallback инкрементит `oi_tg_chat_resolution_failed_total{reason ∈ {missing, inactive, group_inactive, group_chat_inactive, no_default}}`.

### 8b.5 Async-валидация thread_id

Telegram не даёт sync `getForumTopic` без админ-прав. Поэтому thread_id принимается как-есть. На permanent-failure отправки (`TelegramBadRequest`/`TelegramForbiddenError`) dispatcher вызывает `routing_groups_repo.find_by_chat_native_thread(chat, thread)` и пишет `last_send_error` в группу. UI badge `send error` сигнализирует оператору.

---

## 9. AlertState

### 9.1 Назначение

Постоянное состояние state machine на ключе `(rule_id, exchange, canonical_symbol, window)`. Переживает рестарты процесса.

### 9.2 Schema

```python
class AlertState(BaseModel):
    schema_name: str = "alert_state"
    schema_version: int = 1

    rule_id: int
    exchange: str
    canonical_symbol: str
    window: str

    # State machine
    state: str                             # AlertStateName enum
    last_eval_at: datetime
    last_eval_value: Optional[Decimal]     # последний Δ_pct
    last_fired_at: Optional[datetime]
    last_fired_value: Optional[Decimal]    # для smart cooldown
    cooldown_until: Optional[datetime]
    consecutive_above_count: int           # для confirm_points


class AlertStateName(str, Enum):
    IDLE = "idle"
    ARMED = "armed"
    FIRED = "fired"
    COOLDOWN = "cooldown"
```

### 9.3 Хранение

```sql
CREATE TABLE alert_state (
    rule_id          INT NOT NULL REFERENCES alert_rules(id) ON DELETE CASCADE,
    exchange         TEXT NOT NULL,
    canonical_symbol TEXT NOT NULL,
    window           TEXT NOT NULL,
    state            TEXT NOT NULL,
    last_eval_at     TIMESTAMPTZ,
    last_eval_value  NUMERIC(18, 6),
    last_fired_at    TIMESTAMPTZ,
    last_fired_value NUMERIC(18, 6),
    cooldown_until   TIMESTAMPTZ,
    consecutive_above_count SMALLINT NOT NULL DEFAULT 0,
    PRIMARY KEY (rule_id, exchange, canonical_symbol, window)
);
```

---

## 10. DeliveryRequest

### 10.1 Назначение

Запрос на доставку сообщения. Создаётся alert engine при FIRE, потребляется TG sender'ом.

### 10.2 Schema

```python
class DeliveryRequest(BaseModel):
    schema_name: str = "delivery_request"
    schema_version: int = 1

    id: int
    channel: str                           # "telegram"
    template: str                          # "oi_threshold_cross" | "divergence_alert" | "exchange_health_alert"
    payload: dict[str, Any]                # template-specific params
    dedupe_key: str                        # уникальный, для idempotency
    rule_id: int
    alert_event_id: int                    # FK на alert_events.event_id
    chat_id_native: Optional[str]          # F37: chat snapshot, NULL = legacy/default fallback
    message_thread_id: Optional[int]       # F41: thread snapshot, NULL = main chat (no thread)

    status: str                            # DeliveryStatus enum
    retry_count: int
    next_retry_at: Optional[datetime]
    last_error: Optional[str]

    created_at: datetime
    sent_at: Optional[datetime]


class DeliveryStatus(str, Enum):
    PENDING = "pending"
    SENDING = "sending"
    SENT = "sent"
    FAILED = "failed"                      # exhausted retries
    SKIPPED = "skipped"                    # duplicate dedupe_key
```

### 10.3 Payload схемы

#### Template `oi_threshold_cross`

```json
{
  "exchange": "binance",
  "canonical_symbol": "BTC",
  "native_symbol": "BTCUSDT",
  "window": "5m",
  "delta_pct": "6.2",
  "oi_notional_usdt": "1240000000",
  "price": "67420.50",
  "price_change_pct": "0.4",
  "valuation_status": "authoritative"
}
```

Render:
```
BTC @ Binance
+6.2% за 5мин
OI: $1.24B   Цена: $67,420
```

#### Template `consensus_alert`

```json
{
  "canonical_symbol": "BTC",
  "window": "15m",
  "exchange_count": 4,
  "exchanges": [
    {"exchange": "binance", "delta_pct": "5.3"},
    {"exchange": "bybit",   "delta_pct": "5.8"},
    {"exchange": "okx",     "delta_pct": "5.1"},
    {"exchange": "bitget",  "delta_pct": "6.2"}
  ],
  "avg_delta_pct": "5.6"
}
```

#### Template `divergence_alert`

```json
{
  "exchange": "binance",
  "canonical_symbol": "BTC",
  "window": "15m",
  "oi_delta_pct": "+8.1",
  "price_delta_pct": "-2.3",
  "divergence_type": "oi_up_price_down"
}
```

#### Template `exchange_health_alert`

```json
{
  "exchange": "kucoin",
  "issue": "stale_data",
  "details": {
    "minutes_since_last_sample": 7,
    "affected_symbols_count": 312
  }
}
```

### 10.4 Хранение

```sql
CREATE TABLE delivery_queue (
    id              BIGSERIAL PRIMARY KEY,
    channel         TEXT NOT NULL,
    template        TEXT NOT NULL,
    payload         JSONB NOT NULL,
    dedupe_key      TEXT NOT NULL UNIQUE,
    rule_id         INT REFERENCES alert_rules(id),
    alert_event_id  BIGINT,
    chat_id_native  TEXT,                 -- F37: chat snapshot at FIRE-time, NULL = legacy/default fallback
    message_thread_id BIGINT,              -- F41: thread snapshot at FIRE-time, NULL = main chat (no thread)
    status          TEXT NOT NULL DEFAULT 'pending',
    retry_count     SMALLINT NOT NULL DEFAULT 0,
    next_retry_at   TIMESTAMPTZ,
    last_error      TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    sent_at         TIMESTAMPTZ
);

CREATE INDEX idx_delivery_pending ON delivery_queue (status, next_retry_at)
    WHERE status IN ('pending', 'failed') AND retry_count < 5;
```

---

## 11. Instrument

### 11.1 Назначение

Master data об инструменте. Используется normalizer'ом для resolve canonical_symbol и unit conversion.

### 11.2 Schema

```python
class Instrument(BaseModel):
    schema_name: str = "instrument"
    schema_version: int = 1

    id: int
    exchange: str
    native_symbol: str

    canonical_symbol: str                  # "BTC"
    base_asset: str
    quote_asset: str = "USDT"
    settle_asset: str = "USDT"
    market_type: str = "usdt_perp"
    contract_type: str = "perpetual"

    # Параметры контракта
    contract_size: Decimal                 # default 1
    multiplier: Decimal = Decimal("1")
    is_usdt_m: bool = True

    # Lifecycle
    first_seen_at: datetime
    last_seen_at: datetime
    delisted_at: Optional[datetime] = None
    blacklisted: bool = False

    # Версионирование
    instrument_version: int                # инкрементируется при изменении contract_size / multiplier

    # Сырые метаданные с биржи (для дебага)
    raw_metadata: dict[str, Any]

    class Config:
        frozen = True
```

### 11.3 Хранение

```sql
CREATE TABLE instruments (
    id                  SERIAL PRIMARY KEY,
    exchange            TEXT NOT NULL,
    native_symbol       TEXT NOT NULL,
    canonical_symbol    TEXT NOT NULL,
    base_asset          TEXT NOT NULL,
    quote_asset         TEXT NOT NULL DEFAULT 'USDT',
    settle_asset        TEXT NOT NULL DEFAULT 'USDT',
    market_type         TEXT NOT NULL DEFAULT 'usdt_perp',
    contract_type       TEXT NOT NULL DEFAULT 'perpetual',
    contract_size       NUMERIC(38, 18) NOT NULL DEFAULT 1,
    multiplier          NUMERIC(38, 18) NOT NULL DEFAULT 1,
    is_usdt_m           BOOLEAN NOT NULL DEFAULT true,
    first_seen_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    delisted_at         TIMESTAMPTZ,
    blacklisted         BOOLEAN NOT NULL DEFAULT false,
    instrument_version  INT NOT NULL DEFAULT 1,
    raw_metadata        JSONB NOT NULL DEFAULT '{}'::jsonb,
    UNIQUE (exchange, native_symbol)
);

CREATE INDEX idx_instruments_lookup ON instruments (exchange, native_symbol)
    WHERE delisted_at IS NULL AND NOT blacklisted;

CREATE INDEX idx_instruments_canonical ON instruments (canonical_symbol);
```

---

## 12. Settings (KV)

### 12.1 Назначение

Глобальные тунабли, не помещающиеся в `alert_rules`. Whitelist бирж, blacklist символов, пороговые формулы, TG chat_id.

### 12.2 Schema

```python
class SettingsKV(BaseModel):
    schema_name: str = "settings_kv"
    schema_version: int = 1

    key: str
    value: Any                             # JSON-сериализуемое
    updated_at: datetime
    description: Optional[str] = None
```

### 12.3 Известные ключи

| Key | Type | Default | Описание |
|---|---|---|---|
| `polling_interval_sec` | int | 60 | Период опроса |
| `freshness_budget_now_sec` | int | 120 | `2 × polling` |
| `freshness_budget_lookback_sec` | int | 90 | `1.5 × polling` |
| `coverage_timeout_sec` | int | 180 | `3 × polling` |
| `instruments_sync_interval_sec` | int | 300 | каждые 5 мин |
| `min_oi_notional_default_usdt` | str (Decimal) | "5000000" | floor-фильтр |
| `cooldown_default_sec` | int | 900 | default cooldown |
| `smart_cooldown_factor` | str (Decimal) | "1.5" | smart re-fire порог |
| `exchange_whitelist` | list[str] | (все 12) | какие биржи опрашиваем |
| `symbol_blacklist` | list[str] | [] | глобально исключённые символы |
| `symbol_whitelist` | list[str] | [] | если задан — только эти; если пустой — все |
| `tg_dry_run` | bool | false | если true — не шлём в TG, пишем в лог |
| `evaluation_cycle_sec` | int | 30 | как часто запускается alert engine |

### 12.4 Хранение

```sql
CREATE TABLE settings (
    key         TEXT PRIMARY KEY,
    value       JSONB NOT NULL,
    description TEXT,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 13. ConnectorErrorEvent

### 13.1 Назначение

Лог ошибок коннекторов (timeouts, 5xx, parse errors). Используется для exchange-health алертов и диагностики.

### 13.2 Schema

```python
class ConnectorErrorEvent(BaseModel):
    schema_name: str = "connector_error_event"
    schema_version: int = 1

    id: int
    occurred_at: datetime
    exchange: str
    endpoint: str
    error_type: str                        # ConnectorErrorType enum
    error_message: str
    http_status: Optional[int]
    request_duration_ms: Optional[int]
    retry_count: int


class ConnectorErrorType(str, Enum):
    TIMEOUT = "timeout"
    DNS_FAIL = "dns_fail"
    CONNECTION_RESET = "connection_reset"
    HTTP_4XX = "http_4xx"
    HTTP_5XX = "http_5xx"
    RATE_LIMITED = "rate_limited"
    PARSE_ERROR = "parse_error"
    UNKNOWN = "unknown"
```

### 13.3 Хранение

```sql
CREATE TABLE connector_errors (
    id              BIGSERIAL,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    exchange        TEXT NOT NULL,
    endpoint        TEXT NOT NULL,
    error_type      TEXT NOT NULL,
    error_message   TEXT NOT NULL,
    http_status     SMALLINT,
    request_duration_ms INT,
    retry_count     SMALLINT NOT NULL DEFAULT 0,
    PRIMARY KEY (id, occurred_at)
);
SELECT create_hypertable('connector_errors', 'occurred_at',
    chunk_time_interval => INTERVAL '7 days');
SELECT add_retention_policy('connector_errors', INTERVAL '30 days');

CREATE INDEX idx_connector_errors_recent ON connector_errors (exchange, occurred_at DESC);
```

---

## 14. NormalizationError

### 14.1 Назначение

Когда normalizer не смог разрешить событие (unknown symbol, unparseable payload). Раздельно от raw_exchange_events для удобной фильтрации «что не нормализовалось».

### 14.2 Schema

```python
class NormalizationError(BaseModel):
    schema_name: str = "normalization_error"
    schema_version: int = 1

    id: int
    occurred_at: datetime
    exchange: str
    native_symbol: Optional[str]
    raw_event_id: int
    raw_hash: str

    error_type: str                        # NormalizationErrorType enum
    error_message: str
    failed_stage: str                      # "parse" | "instrument_resolve" | "unit_resolve" | "price_attach" | "convert"
    normalization_version: int


class NormalizationErrorType(str, Enum):
    PARSE_ERROR = "parse_error"
    INSTRUMENT_UNRESOLVED = "instrument_unresolved"
    UNIT_UNRESOLVED = "unit_unresolved"
    PRICE_UNAVAILABLE = "price_unavailable"
    CONVERSION_FAILED = "conversion_failed"
    SCHEMA_DRIFT = "schema_drift"
```

### 14.3 Хранение

```sql
CREATE TABLE normalization_errors (
    id              BIGSERIAL,
    occurred_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    exchange        TEXT NOT NULL,
    native_symbol   TEXT,
    raw_event_id    BIGINT,
    raw_hash        TEXT NOT NULL,
    error_type      TEXT NOT NULL,
    error_message   TEXT NOT NULL,
    failed_stage    TEXT NOT NULL,
    normalization_version INT NOT NULL,
    PRIMARY KEY (id, occurred_at)
);
SELECT create_hypertable('normalization_errors', 'occurred_at',
    chunk_time_interval => INTERVAL '7 days');
SELECT add_retention_policy('normalization_errors', INTERVAL '30 days');
```

---

## 15. Schema lifecycle

### 15.1 Эволюция

При добавлении/изменении полей:
1. Бамп `schema_version`.
2. Alembic миграция БД (если применимо).
3. Обновление этого документа (§ соответствующий контракт).
4. Замечание в `00_DECISIONS_LOG.md` — какое решение это вызвало.

### 15.2 Backward compatibility

- Добавление optional поля → не bump major.
- Удаление поля → bump major, поддержка старых записей в БД через миграцию (default value или удаление).
- Переименование → bump major, alias на старое имя на 1 deploy.

### 15.3 Forward compatibility (при reading)

Pydantic model имеет `extra = "ignore"` по умолчанию, чтобы новые поля от продвинутого продьюсера не ломали старого консьюмера. Логируем `unknown_field` warning.

---

## 16. JSON Schema export

Все pydantic модели экспортируются в JSON Schema для:
- Frontend — типизация TypeScript через codegen (`json-schema-to-typescript`).
- Документации API (FastAPI авто-генерирует OpenAPI из pydantic).
- Тестов — валидация фикстур.

CI команда (например):
```bash
python -m app.tools.export_schemas docs/schemas/
```
Генерирует:
```
docs/schemas/
├── raw_exchange_event.json
├── normalized_oi_sample.json
├── alert_event.json
├── ...
```

---

## 17. Cross-references

| Контракт | Используется в |
|---|---|
| `RawExchangeEvent` | `11_EXCHANGE_ADAPTERS/_BASE_CONNECTOR.md` |
| `NormalizedOISample` | `07_NORMALIZER.md`, `08_TIME_SERIES_STORAGE.md` |
| `OIBar*` | `08_TIME_SERIES_STORAGE.md`, `09_ALERT_ENGINE.md` |
| `AlertCandidate` / `AlertEvent` / `AlertRule` / `AlertState` | `09_ALERT_ENGINE.md` |
| `DeliveryRequest` | `10_DELIVERY_LAYER.md` |
| `Instrument` | `06_INSTRUMENT_REGISTRY.md` |
| `Settings` | `09_ALERT_ENGINE.md`, `10_DELIVERY_LAYER.md` |
| `ConnectorErrorEvent` | `11_EXCHANGE_ADAPTERS/_BASE_CONNECTOR.md`, `12_OBSERVABILITY_SLO.md` |
| `NormalizationError` | `07_NORMALIZER.md` |
