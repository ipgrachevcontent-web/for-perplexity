# 08 · Time-Series Storage

> TimescaleDB hypertable + continuous aggregates для OI. Полный DDL, политики, процедуры.
> Связано: `00_DECISIONS_LOG.md C3, C9, C11, F4, F5`, `04_TIME_MODEL.md`, `05_DATA_CONTRACTS.md §4`.

---

## 1. Обзор

Storage — это слой, превращающий поток нормализованных OI-точек в:
- **Сырой временной ряд** (`oi_samples`, hypertable).
- **Готовые окна** (`oi_5m`, `oi_15m`, `oi_30m`, continuous aggregates).
- **Запросы UI** (история символа, dashboard live-таблица).
- **Запросы alert engine** (последний bar + предыдущий для Δ).

Особенности:
- Append-only.
- Партиционирование по `ts_exchange` (рыночное время).
- Compression после 7 дней.
- Retention 90 дней.
- Late-arriving data поддерживается (через decompress + reinsert).

---

## 2. DDL

### 2.1 Главная таблица `oi_samples`

```sql
CREATE TABLE oi_samples (
    -- временные оси
    ts_exchange      TIMESTAMPTZ      NOT NULL,
    ts_ingested      TIMESTAMPTZ      NOT NULL DEFAULT now(),

    -- идентификация
    exchange         TEXT             NOT NULL,
    canonical_symbol TEXT             NOT NULL,             -- "BTC"
    native_symbol    TEXT             NOT NULL,             -- "BTCUSDT", "BTC-USDT-SWAP"

    -- market info (избыточно, но удобно для запросов)
    quote_asset      TEXT             NOT NULL DEFAULT 'USDT',
    settle_asset     TEXT             NOT NULL DEFAULT 'USDT',
    market_type      TEXT             NOT NULL DEFAULT 'usdt_perp',

    -- OI и цена
    oi_coins         NUMERIC(38, 18)  NOT NULL,
    oi_native_unit   TEXT             NOT NULL,             -- "base_asset" | "contracts" | ...
    oi_notional_usdt NUMERIC(38, 8)   NOT NULL,
    price_used       NUMERIC(38, 18)  NOT NULL,
    price_source     TEXT             NOT NULL,             -- "mark" | "index" | "last" | ...

    -- качество
    source_kind          TEXT         NOT NULL,             -- "snapshot" | "native_interval" | ...
    valuation_status     TEXT         NOT NULL,             -- "authoritative" | ...
    normalization_version SMALLINT    NOT NULL,
    instrument_version    INT         NOT NULL,
    warnings              TEXT[]      NOT NULL DEFAULT '{}',

    -- трассировка
    raw_event_id     BIGINT,
    raw_hash         TEXT             NOT NULL,

    PRIMARY KEY (exchange, canonical_symbol, ts_exchange)
);

-- Hypertable
SELECT create_hypertable(
    'oi_samples',
    'ts_exchange',
    chunk_time_interval => INTERVAL '1 day'
);

-- Compression
ALTER TABLE oi_samples SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'exchange, canonical_symbol',
    timescaledb.compress_orderby   = 'ts_exchange DESC'
);
SELECT add_compression_policy('oi_samples', INTERVAL '7 days');

-- Retention (90 days, см. C3)
SELECT add_retention_policy('oi_samples', INTERVAL '90 days');

-- Дополнительные индексы (для запросов UI и alert engine)
CREATE INDEX idx_oi_samples_canonical_time
    ON oi_samples (canonical_symbol, ts_exchange DESC);

CREATE INDEX idx_oi_samples_exchange_time
    ON oi_samples (exchange, ts_exchange DESC);

CREATE INDEX idx_oi_samples_recent_native
    ON oi_samples (exchange, native_symbol, ts_exchange DESC);
```

### 2.2 Почему такие индексы

- **PK `(exchange, canonical_symbol, ts_exchange)`** — обеспечивает уникальность точки + быстрый range scan по `ts_exchange` внутри одного `(exchange, canonical_symbol)`. Это формат запроса для расчёта Δ.
- **`idx_oi_samples_canonical_time`** — для cross-exchange запросов по одному символу (Symbol page, ad-hoc analytics).
- **`idx_oi_samples_exchange_time`** — для exchange-health (последняя точка per exchange).
- **`idx_oi_samples_recent_native`** — для трассировки конкретного native_symbol (например при дебаге).

### 2.3 Замечания по DDL

#### Почему PK включает `ts_exchange`

TimescaleDB **требует**, чтобы любой `UNIQUE` индекс/`PRIMARY KEY` на hypertable содержал партиционирующую колонку. PK без `ts_exchange` упадёт:
```
ERROR: cannot create a unique index without the column "ts_exchange"
```

#### Почему нет `id BIGSERIAL`

В time-series append-only с natural key (`exchange + canonical_symbol + ts_exchange`) surrogate `id` ничего не даёт, занимает 8 байт на строку. ~5М строк/день × 8 байт = ~40 МБ/день экономии.

#### Почему `NUMERIC(38, 18)` для coins

Низкокаповые токены: `SHIB` цена ~`0.000028`, `oi_coins` может быть `1_500_000_000_000.345_678_910_123_456_789`. Запас precision важен.

#### Почему `NUMERIC(38, 8)` для USDT

USDT не нужны 18 знаков после запятой (USDT минимально дробится до `0.000001`). 8 знаков достаточно, экономит память.

---

## 3. Continuous Aggregates

### 3.1 Концепция

Continuous aggregates — это materialized views, автоматически обновляемые TimescaleDB по расписанию. У нас три:

- `oi_5m` — построен поверх `oi_samples`.
- `oi_15m` — построен поверх `oi_5m` (hierarchical CA).
- `oi_30m` — построен поверх `oi_5m` (hierarchical CA).

Зачем hierarchical: 15m строится из уже агрегированных 5m bucket'ов, не из raw points → пересчёт быстрее в 5×.

### 3.2 DDL `oi_5m`

```sql
CREATE MATERIALIZED VIEW oi_5m
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('5 minutes', ts_exchange) AS bucket,
    exchange,
    canonical_symbol,
    native_symbol,                                        -- для UI
    -- OI агрегаты (по oi_coins, см. C5)
    FIRST(oi_coins, ts_exchange) AS open_oi_coins,
    MAX(oi_coins)                AS high_oi_coins,
    MIN(oi_coins)                AS low_oi_coins,
    LAST(oi_coins, ts_exchange)  AS close_oi_coins,
    -- USDT notional (для display)
    FIRST(oi_notional_usdt, ts_exchange) AS open_oi_notional_usdt,
    LAST(oi_notional_usdt, ts_exchange)  AS close_oi_notional_usdt,
    LAST(price_used, ts_exchange)        AS close_price,
    -- качество
    COUNT(*)                            AS sample_count,
    LAST(ts_exchange, ts_exchange)      AS last_ts_exchange,
    LAST(valuation_status, ts_exchange) AS last_valuation_status,
    LAST(source_kind, ts_exchange)      AS last_source_kind,
    BOOL_OR(source_kind = 'degraded_fallback') AS has_degraded_source
FROM oi_samples
GROUP BY bucket, exchange, canonical_symbol, native_symbol
WITH NO DATA;

-- Refresh policy: каждые 30 секунд обновляем последний час
SELECT add_continuous_aggregate_policy(
    'oi_5m',
    start_offset      => INTERVAL '1 hour',
    end_offset        => INTERVAL '30 seconds',
    schedule_interval => INTERVAL '30 seconds'
);

-- Retention (см. C3)
SELECT add_retention_policy('oi_5m', INTERVAL '90 days');

-- Индекс
CREATE INDEX idx_oi_5m_lookup
    ON oi_5m (exchange, canonical_symbol, bucket DESC);
```

### 3.3 DDL `oi_15m`

```sql
CREATE MATERIALIZED VIEW oi_15m
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('15 minutes', bucket) AS bucket,
    exchange,
    canonical_symbol,
    FIRST(open_oi_coins, bucket)   AS open_oi_coins,
    MAX(high_oi_coins)             AS high_oi_coins,
    MIN(low_oi_coins)              AS low_oi_coins,
    LAST(close_oi_coins, bucket)   AS close_oi_coins,
    FIRST(open_oi_notional_usdt, bucket) AS open_oi_notional_usdt,
    LAST(close_oi_notional_usdt, bucket) AS close_oi_notional_usdt,
    LAST(close_price, bucket)            AS close_price,
    SUM(sample_count)                    AS sample_count,
    LAST(last_ts_exchange, bucket)       AS last_ts_exchange,
    LAST(last_valuation_status, bucket)  AS last_valuation_status,
    LAST(last_source_kind, bucket)       AS last_source_kind,
    BOOL_OR(has_degraded_source)         AS has_degraded_source
FROM oi_5m
GROUP BY time_bucket('15 minutes', bucket), exchange, canonical_symbol
WITH NO DATA;

SELECT add_continuous_aggregate_policy(
    'oi_15m',
    start_offset      => INTERVAL '1 hour',
    end_offset        => INTERVAL '30 seconds',
    schedule_interval => INTERVAL '60 seconds'
);

SELECT add_retention_policy('oi_15m', INTERVAL '90 days');

CREATE INDEX idx_oi_15m_lookup
    ON oi_15m (exchange, canonical_symbol, bucket DESC);
```

### 3.4 DDL `oi_30m`

Аналогично `oi_15m`, но `time_bucket('30 minutes', ...)`. Refresh каждые 60 секунд.

### 3.5 `volume_samples_5m` (F51) — отдельная hypertable, не CA

Aster-only 5m trade-volume bucket'ы. **Не** continuous aggregate — данные приходят уже агрегированными со стороны биржи (kline `quoteAssetVolume`), агрегировать у нас нечего.

DDL (см. миграцию `0016_volume_samples_5m.py`):

- `bucket TIMESTAMPTZ` — kline `openTime`, 5m-aligned. Partition axis hypertable'а (chunk_time_interval = 1 day, аналогично `oi_samples`).
- `exchange TEXT`, `canonical_symbol TEXT`, `native_symbol TEXT`.
- `base_volume NUMERIC(38,18)` — kline `volume` (base asset).
- `quote_volume_usdt NUMERIC(38,8)` — kline `quoteAssetVolume` (USDT notional).
- `trade_count BIGINT NULL` — kline `numberOfTrades`.
- `ts_ingested TIMESTAMPTZ DEFAULT now()` — wall-clock записи.
- PK = `(exchange, canonical_symbol, bucket)`. UPSERT'им running bucket каждую минуту (биржа добавляет в него трейды до закрытия 5m окна).
- Compression: `segmentby='exchange, canonical_symbol'`, `orderby='bucket DESC'`, `compress_after='7 days'` (симметрично `oi_samples`, F5).
- Retention: 90 дней (C3).
- Дополнительный индекс `(exchange, bucket DESC)` для hot-path scan'а alert engine'а (PR2).

Read API репо: `app.storage.repositories.volume_samples.get_latest_two_closed_buckets(pool, exchange, before)` возвращает только **закрытые** бакеты (`bucket + 5min ≤ before`), чтобы не сравнивать с растущим in-progress bucket'ом, где `quote_volume_usdt` ещё увеличивается.

---

## 4. Запись (insert path)

### 4.1 Идемпотентный insert

```python
INSERT_SAMPLE_SQL = """
INSERT INTO oi_samples (
    ts_exchange, ts_ingested, exchange, canonical_symbol, native_symbol,
    quote_asset, settle_asset, market_type,
    oi_coins, oi_native_unit, oi_notional_usdt, price_used, price_source,
    source_kind, valuation_status, normalization_version, instrument_version,
    warnings, raw_event_id, raw_hash
) VALUES (
    $1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13,
    $14, $15, $16, $17, $18, $19, $20
)
ON CONFLICT (exchange, canonical_symbol, ts_exchange)
DO UPDATE SET
    ts_ingested = EXCLUDED.ts_ingested,
    oi_coins = EXCLUDED.oi_coins,
    oi_notional_usdt = EXCLUDED.oi_notional_usdt,
    price_used = EXCLUDED.price_used,
    price_source = EXCLUDED.price_source,
    source_kind = EXCLUDED.source_kind,
    valuation_status = EXCLUDED.valuation_status,
    normalization_version = EXCLUDED.normalization_version,
    instrument_version = EXCLUDED.instrument_version,
    warnings = EXCLUDED.warnings,
    raw_event_id = EXCLUDED.raw_event_id,
    raw_hash = EXCLUDED.raw_hash
WHERE oi_samples.normalization_version < EXCLUDED.normalization_version
   OR oi_samples.raw_hash != EXCLUDED.raw_hash;
"""
```

**Логика:**
- Если запись с тем же `(exchange, canonical_symbol, ts_exchange)` уже есть, INSERT попадает в `ON CONFLICT`.
- UPDATE срабатывает, только если:
  - Новая `normalization_version` строго выше (replay с новыми правилами).
  - Или `raw_hash` другой (та же временная точка, но новый payload, что бывает при backfill после corrupted run).
- Иначе — DO NOTHING.

### 4.2 Bulk insert

При normalize пакета (1 цикл = 12 бирж × ~300 символов = ~3600 точек):

```python
async def bulk_insert_samples(samples: list[NormalizedOISample]) -> int:
    async with pool.acquire() as conn:
        # Используем executemany; для 3600 строк это ~50–100 ms
        result = await conn.executemany(INSERT_SAMPLE_SQL, [
            sample.to_db_tuple() for sample in samples
        ])
        return result.rowcount
```

При desire высокой производительности — `COPY ... FROM STDIN` (но requires aiosqlite-стиля, что усложняет код).

### 4.3 Late-arriving data в compressed chunks

TimescaleDB поддерживает insert в compressed chunks начиная с версии ≥ 2.11. Поведение:
- При insert в compressed chunk — TimescaleDB автоматически декомпрессит chunk, вставляет, затем фоновый job recompresses.
- Это медленнее, чем insert в hot (uncompressed) chunk.
- Для обычного flow это не проблема (последний chunk всегда uncompressed первые 7 дней).
- Для replay/backfill в старые chunks — медленно, но работает.

---

## 5. Чтение (query path)

### 5.1 Запрос для UI dashboard (live-таблица)

```sql
-- Для каждой пары (exchange, canonical_symbol) последняя точка + Δ_5m, Δ_15m, Δ_30m
WITH latest AS (
    SELECT
        exchange,
        canonical_symbol,
        native_symbol,
        ts_exchange,
        oi_coins,
        oi_notional_usdt,
        price_used,
        valuation_status,
        source_kind
    FROM oi_samples
    WHERE ts_exchange >= now() - INTERVAL '5 minutes'
    ORDER BY exchange, canonical_symbol, ts_exchange DESC
    LIMIT 1
), deltas_5m AS (
    SELECT
        exchange,
        canonical_symbol,
        close_oi_coins,
        LAG(close_oi_coins, 1) OVER (
            PARTITION BY exchange, canonical_symbol ORDER BY bucket
        ) AS prev_close
    FROM oi_5m
    WHERE bucket >= now() - INTERVAL '15 minutes'
)
SELECT ...;
```

В реальном коде это будет несколько запросов к CA + JOIN с latest sample. Чтобы не перегружать SQL — материализуем view `live_dashboard_view` (см. §5.4).

### 5.2 Запрос для Symbol page (history график)

Для одного `canonical_symbol`, 12 бирж, период (например 24h):

```sql
SELECT
    bucket,
    exchange,
    close_oi_notional_usdt,
    close_price,
    close_oi_coins
FROM oi_5m
WHERE canonical_symbol = $1
  AND bucket >= $2
  AND bucket <= $3
ORDER BY exchange, bucket;
```

Для длинных периодов (90 дней) на минутном разрешении — слишком много точек для UI. Используем decimation (см. §5.5).

### 5.3 Запрос для alert engine (последний bar + предыдущий)

```sql
WITH ranked AS (
    SELECT
        exchange,
        canonical_symbol,
        bucket,
        close_oi_coins,
        last_valuation_status,
        last_source_kind,
        has_degraded_source,
        ROW_NUMBER() OVER (
            PARTITION BY exchange, canonical_symbol
            ORDER BY bucket DESC
        ) AS rn
    FROM oi_5m
    WHERE bucket >= now() - INTERVAL '15 minutes'
)
SELECT
    exchange,
    canonical_symbol,
    MAX(CASE WHEN rn = 1 THEN close_oi_coins END)        AS curr_oi,
    MAX(CASE WHEN rn = 1 THEN bucket END)                AS curr_bucket,
    MAX(CASE WHEN rn = 1 THEN last_valuation_status END) AS curr_valuation,
    MAX(CASE WHEN rn = 1 THEN last_source_kind END)      AS curr_source,
    MAX(CASE WHEN rn = 2 THEN close_oi_coins END)        AS prev_oi,
    MAX(CASE WHEN rn = 2 THEN bucket END)                AS prev_bucket
FROM ranked
WHERE rn IN (1, 2)
GROUP BY exchange, canonical_symbol;
```

Для 15m/30m — аналогично с `oi_15m`/`oi_30m`.

### 5.4 Materialized view для UI dashboard

Чтобы не считать live-таблицу 12×300 = 3600 строк on the fly каждый SSE push:

```sql
CREATE MATERIALIZED VIEW live_dashboard_mv AS
SELECT
    s.exchange,
    s.canonical_symbol,
    s.native_symbol,
    s.ts_exchange,
    s.oi_coins,
    s.oi_notional_usdt,
    s.price_used,
    s.valuation_status,
    s.source_kind,
    d5.delta_pct  AS delta_5m_pct,
    d15.delta_pct AS delta_15m_pct,
    d30.delta_pct AS delta_30m_pct
FROM (
    SELECT DISTINCT ON (exchange, canonical_symbol)
        exchange, canonical_symbol, native_symbol,
        ts_exchange, oi_coins, oi_notional_usdt, price_used,
        valuation_status, source_kind
    FROM oi_samples
    WHERE ts_exchange >= now() - INTERVAL '15 minutes'
    ORDER BY exchange, canonical_symbol, ts_exchange DESC
) s
LEFT JOIN LATERAL (
    SELECT
        CASE WHEN prev_close IS NULL OR prev_close = 0 THEN NULL
             ELSE (close_oi_coins - prev_close) / prev_close * 100
        END AS delta_pct
    FROM (
        SELECT
            close_oi_coins,
            LAG(close_oi_coins) OVER (ORDER BY bucket) AS prev_close
        FROM oi_5m
        WHERE exchange = s.exchange
          AND canonical_symbol = s.canonical_symbol
          AND bucket >= now() - INTERVAL '15 minutes'
        ORDER BY bucket DESC
        LIMIT 2
    ) sub
    ORDER BY 1 DESC LIMIT 1
) d5 ON true
LEFT JOIN LATERAL (...) d15 ON true
LEFT JOIN LATERAL (...) d30 ON true;
```

NB: refresh `REFRESH MATERIALIZED VIEW CONCURRENTLY live_dashboard_mv;` каждые 30 секунд (cron job или scheduler task).

Альтернатива: считать live-таблицу динамически при каждом SSE push. Для 1 пользователя (1–2 SSE подключения) это приемлемо без MV.

### 5.5 Decimation для длинных периодов

Для 12-line × 90d графика нужна децимация. Cтандартный приём — `time_bucket` до плотности экрана:

```python
def pick_bucket_size(period_hours: int, target_points: int = 1000) -> str:
    seconds_total = period_hours * 3600
    seconds_per_point = seconds_total / target_points
    if seconds_per_point < 60:    return "1 minute"
    if seconds_per_point < 300:   return "5 minutes"
    if seconds_per_point < 900:   return "15 minutes"
    if seconds_per_point < 1800:  return "30 minutes"
    if seconds_per_point < 3600:  return "1 hour"
    if seconds_per_point < 14400: return "4 hours"
    return "1 day"
```

Запрос:
```sql
SELECT
    time_bucket($1::interval, ts_exchange) AS bucket,
    exchange,
    LAST(oi_notional_usdt, ts_exchange) AS oi_notional_usdt
FROM oi_samples
WHERE canonical_symbol = $2
  AND ts_exchange >= $3 AND ts_exchange <= $4
GROUP BY bucket, exchange
ORDER BY bucket, exchange;
```

Для bucket'ов 5m / 15m / 30m / 1h / 4h можем читать из CA, а не raw `oi_samples`. Для 1m — из raw. Для 1d — из CA `oi_5m` с group by 288.

#### Source routing в `symbol_history.pick_source()`

Текущая реализация выбирает источник так:

| Timeframe (UI) | Source                | Notes                                     |
|----------------|------------------------|-------------------------------------------|
| `auto`         | raw + `time_bucket`    | bucket подбирается ladder'ом выше         |
| `1m`           | raw + `time_bucket`    | CA для `1m` нет                           |
| `5m`           | CA `oi_5m`             | колонки CA aliasятся в wire-схему         |
| `15m`          | CA `oi_15m`            | то же                                     |
| `30m`          | CA `oi_30m`            | то же                                     |
| `1h` / `4h`    | raw + `time_bucket`    | отдельных CA на эти грейны нет            |
| `1d`           | raw + `time_bucket`    | (вариант через `oi_5m` group by 288 — future work) |

CA-путь использует SELECT с алиасами: `close_oi_coins → oi_coins`, `close_oi_notional_usdt → oi_notional_usdt`, `close_price → price_used`, `last_valuation_status → valuation_status`. Wire-формат идентичен raw-пути — фронтенд (`pivotForChart`) не различает источники.

---

## 6. Reprocessing procedures

### 6.1 Backfill (заполнение пропусков)

Сценарий: коннектор был down 30 минут, в `oi_samples` дырка.

**Шаги:**
1. Запустить collector в режиме `--from=...` `--until=...` для данной биржи + символов.
2. Если биржа поддерживает `fetch_range` (Bybit, KuCoin, Binance statistics) — пакетный fetch.
3. Иначе — ничего не сделать, dropping data accepted (логируется).
4. Полученные точки идут через normalizer с `source_kind = "backfill"`.
5. Insert в `oi_samples` (ON CONFLICT DO UPDATE при необходимости).
6. Refresh CA для затронутого диапазона:
   ```sql
   CALL refresh_continuous_aggregate('oi_5m', $start, $end);
   CALL refresh_continuous_aggregate('oi_15m', $start, $end);
   CALL refresh_continuous_aggregate('oi_30m', $start, $end);
   ```
7. Alert engine **не пересматривает** прошлые алерты (см. F9).

CLI:
```bash
python -m app.tools.backfill --exchange=bybit --since="2026-04-30 12:00" --until="2026-04-30 12:30"
```

### 6.2 Replay (новая `normalization_version`)

Сценарий: нашли баг в формуле для XT contract_size → bump `normalization_version`.

**Шаги:**
1. Bump версии в `app/normalizer/version.py`: `NORMALIZATION_VERSION = 4`.
2. Развернуть новый код (систематически, через рестарт сервиса).
3. Запустить replay для XT за нужный период:
   ```bash
   python -m app.tools.replay --exchange=xt --since="2026-04-25" --until="2026-04-30" --version=4
   ```
4. Replay читает `raw_exchange_events`, прогоняет через новый normalizer.
5. INSERT с `ON CONFLICT DO UPDATE WHERE oi_samples.normalization_version < EXCLUDED.normalization_version`.
6. Refresh CA.
7. Опционально пометить старые `alerts_sent` как `historically_inaccurate`.
8. Alert engine **не отправляет** retro-fire.

### 6.3 Manual decompression (если требуется)

Для массового backfill в compressed chunk эффективнее декомпрессить вручную:

```sql
SELECT decompress_chunk(c)
FROM show_chunks('oi_samples', older_than => INTERVAL '7 days', newer_than => INTERVAL '14 days') c;

-- Здесь делаем массовый INSERT/UPDATE
-- ...

-- Затем компрессируем обратно (фоновый policy сам это сделает,
-- или вызвать вручную)
SELECT compress_chunk(c)
FROM show_chunks('oi_samples', older_than => INTERVAL '7 days') c;
```

---

## 7. Quality metrics view

Отдельная view для observability качества данных:

```sql
CREATE MATERIALIZED VIEW oi_quality_5m AS
SELECT
    time_bucket('5 minutes', ts_exchange) AS bucket,
    exchange,
    -- Coverage
    COUNT(DISTINCT canonical_symbol) AS symbols_with_data,
    -- Quality status mix
    SUM(CASE WHEN valuation_status = 'authoritative' THEN 1 ELSE 0 END) AS authoritative_count,
    SUM(CASE WHEN valuation_status = 'good_estimate'  THEN 1 ELSE 0 END) AS good_estimate_count,
    SUM(CASE WHEN valuation_status = 'low_confidence' THEN 1 ELSE 0 END) AS low_confidence_count,
    -- Source kind mix
    SUM(CASE WHEN source_kind = 'native_interval' THEN 1 ELSE 0 END) AS native_interval_count,
    SUM(CASE WHEN source_kind = 'snapshot'        THEN 1 ELSE 0 END) AS snapshot_count,
    SUM(CASE WHEN source_kind = 'degraded_fallback' THEN 1 ELSE 0 END) AS degraded_count,
    -- Latency
    AVG(EXTRACT(EPOCH FROM (ts_ingested - ts_exchange))) AS avg_lag_sec,
    MAX(EXTRACT(EPOCH FROM (ts_ingested - ts_exchange))) AS max_lag_sec
FROM oi_samples
GROUP BY bucket, exchange
WITH NO DATA;

SELECT add_continuous_aggregate_policy(
    'oi_quality_5m',
    start_offset      => INTERVAL '1 hour',
    end_offset        => INTERVAL '30 seconds',
    schedule_interval => INTERVAL '60 seconds'
);
SELECT add_retention_policy('oi_quality_5m', INTERVAL '90 days');
```

Используется в Grafana дашборде "Exchange Health" и при оценке `symbol_coverage_ratio`.

---

## 8. Sizing & capacity

### 8.1 Объёмы

При `12 бирж × ~300 символов × 1440 циклов/сутки`:
- **Сырых точек:** ~5М/сутки = ~150М/месяц.
- **Размер строки** (несжатый): ~250 байт.
- **Сырой объём:** ~37 GB/месяц.
- **После compression** (10–20×): ~2–3 GB/месяц.
- **На 90 дней:** ~6–10 GB compressed + последние 7 дней uncompressed (~9 GB) = **~15–20 GB**.

### 8.2 Continuous aggregates

| CA | Bucket'ов в день | Строк/день | На 90 дней |
|---|---|---|---|
| `oi_5m` | 288 × 12 × 300 | ~1.04М | ~93М (compressed: ~5–10 GB) |
| `oi_15m` | 96 × 12 × 300 | ~346k | ~31М |
| `oi_30m` | 48 × 12 × 300 | ~173k | ~15М |

CA имеют свою compression (если включена).

### 8.3 Raw events (`raw_exchange_events`)

Это **самые жирные** строки (полный JSON payload). Поэтому retention 14 дней (vs 90 для нормализованных).
- `~5М raw events × 5–20 KB JSON = 25–100 GB` без compression.
- Compression spec'ом jsonb достигает 4–8×.
- На 14 дней: `~5–15 GB compressed`.

### 8.4 Total disk plan

```
oi_samples              ~15–20 GB
oi_5m + oi_15m + oi_30m ~10–15 GB
raw_exchange_events     ~5–15 GB
alert_events            < 100 MB (post F43 + F44)
oi_quality_5m           ~1 GB
PostgreSQL system       ~5 GB
WAL archive             ~5 GB
Logs (Loki)             ~5 GB / 30 days
─────────────────────────────────
Total                   ~45–70 GB
```

Рекомендуется отдельный SSD ≥ 200 GB под `/var/lib/postgresql/`.

> `alert_events` — после F43 (persistance whitelist) пишется ~2K строк/день, после F44 (columnstore с `compress_after='7 days'`) hot chunk uncompressed первые 7 дней, далее columnstore. На 90 дней — десятки MB. Pattern compression симметричен `oi_samples`: `segmentby='rule_id, exchange'`, `orderby='ts_processed DESC'`. См. `00_DECISIONS_LOG.md F43, F44`.

---

## 9. Maintenance операции

### 9.1 Daily cleanup (timer)

`oi-tracker-cleanup.timer` запускает `oi-tracker-cleanup.service` ежедневно 03:15 UTC:

```python
async def daily_cleanup() -> None:
    # 1. retention в TimescaleDB запускается автоматически по policy,
    #    но сделаем explicit refresh CA для последних суток (на случай late data)
    await db.execute("CALL refresh_continuous_aggregate('oi_5m',  now() - INTERVAL '1 day', now())")
    await db.execute("CALL refresh_continuous_aggregate('oi_15m', now() - INTERVAL '1 day', now())")
    await db.execute("CALL refresh_continuous_aggregate('oi_30m', now() - INTERVAL '1 day', now())")
    await db.execute("CALL refresh_continuous_aggregate('oi_quality_5m', now() - INTERVAL '1 day', now())")

    # 2. ANALYZE for query planner
    await db.execute("ANALYZE oi_samples;")
    await db.execute("ANALYZE oi_5m;")

    # 3. Cleanup старых SSE-сессий, expired settings и т.п.
    await db.execute("DELETE FROM sse_sessions WHERE last_seen_at < now() - INTERVAL '1 hour'")
```

### 9.2 Manual maintenance

| Задача | Команда |
|---|---|
| Принудительный compress | `SELECT compress_chunk(c) FROM show_chunks('oi_samples', older_than => INTERVAL '7 days') c;` |
| Принудительный decompress (для backfill) | `SELECT decompress_chunk(c) FROM show_chunks('oi_samples', older_than => '<from>', newer_than => '<to>') c;` |
| Reindex | `REINDEX TABLE oi_samples;` (редко нужно) |
| Vacuum analyze | `VACUUM ANALYZE oi_samples;` |
| Просмотр chunks | `SELECT * FROM timescaledb_information.chunks WHERE hypertable_name = 'oi_samples' ORDER BY range_start DESC;` |

---

## 10. Edge cases

### 10.1 Time skew → late insert
Биржа в будущем (clock_skew_forward) → `ts_exchange > now()`. Insert OK, попадает в правильный chunk; CA refresh policy с `end_offset = '30 seconds'` подхватит при следующем запуске.

### 10.2 Несколько points с одним `ts_exchange` для одной пары
Не должно случиться (unique constraint). Если случилось — последний выигрывает (UPDATE). В логи warning `duplicate_ts_exchange`.

### 10.3 Нулевой OI
Запись OK (NUMERIC может быть 0). Алерт-движок при `oi_now < min_oi_notional` отбрасывает.

### 10.4 Очень большой OI (overflow)
`NUMERIC(38, 18)` вмещает значения до `10^20` — заведомо больше любого реального OI. Не overflow.

### 10.5 NULL price (если price endpoint failed)
Normalizer raises `PriceUnavailableError` → точка не попадает в `oi_samples`, идёт в `normalization_errors`. Это корректное поведение — лучше пропустить точку, чем записать с фейковой ценой.

---

## 11. Testing storage layer

### 11.1 Integration тесты

Поднимаем PostgreSQL+TimescaleDB в test-режиме (testcontainers или dedicated test DB):
- Insert 10k synthetic samples.
- Проверяем CA refresh выдаёт правильные агрегаты.
- Проверяем retention policy удаляет старые chunks.
- Проверяем UPDATE по `normalization_version`.

### 11.2 Performance тесты

```python
@pytest.mark.performance
async def test_bulk_insert_throughput():
    samples = generate_samples(count=3600)  # 1 cycle × 12 ex × 300 sym
    start = perf_counter()
    await repo.bulk_insert(samples)
    elapsed = perf_counter() - start
    assert elapsed < 1.0  # 1 cycle should fit in << 1 sec
```

### 11.3 Late-insert test

```python
async def test_late_insert_in_compressed_chunk():
    # 1. Insert старые samples
    # 2. Дождаться compression
    # 3. Insert ещё одну точку с тем же ts_exchange
    # 4. Проверить, что upsert сработал, чанк опять compressed
```

---

## 12. Cross-references

- Schema `NormalizedOISample` → `05_DATA_CONTRACTS.md §4`.
- ts_exchange как axis → `04_TIME_MODEL.md §2`.
- Decisions log (retention, compression) → `00_DECISIONS_LOG.md C3, F5`.
- Read patterns для alert engine → `09_ALERT_ENGINE.md §4`.
- UI запросы → `10_DELIVERY_LAYER.md §5`.
- Reprocessing procedures → `13_OPERATIONS.md §6`.
- Storage capacity мониторинг → `12_OBSERVABILITY_SLO.md §4`.

---

## 13. Migrations Template (Alembic + TimescaleDB)

> Alembic не понимает hypertable / continuous aggregate / compression / retention — всё это администрируется TimescaleDB-функциями. В миграциях используем `op.execute(...)` с raw SQL.
> Цель раздела — единый шаблон, чтобы первая миграция M1 (`oi_samples` hypertable) и все последующие следовали одному стилю и были обратимыми, идемпотентными и не падали при повторном применении на staging/prod.

### 13.1 Принципы

1. **Все TimescaleDB-операции — через `op.execute(text(...))`.** SQLAlchemy ORM-операции (`op.create_table`, `op.add_column`) применимы для базовой части DDL до `create_hypertable`.
2. **Идемпотентность на upgrade.** TimescaleDB-функции имеют параметр `if_not_exists => TRUE` (для `create_hypertable`, `add_compression_policy`, `add_retention_policy`, `add_continuous_aggregate_policy`). **Использовать всегда** — иначе повторное применение на проде/тесте упадёт.
3. **Отсутствие `if_exists` — обходим через `DROP ... IF EXISTS`** (для `MATERIALIZED VIEW`) или через ручную проверку `_timescaledb_catalog`. Конкретные шаблоны — §13.4.
4. **Порядок drop в downgrade — обратный созданию:**
   - сначала retention policies → compression policies → CA refresh policies → CAs → hypertable indexes → hypertable → таблица.
   - Иначе TimescaleDB вернёт ошибку «cannot drop table while dependent continuous aggregate exists».
5. **Транзакционность.** `create_hypertable` и большинство `add_*_policy` совместимы с DDL transaction. `decompress_chunk`, `compress_chunk`, `refresh_continuous_aggregate` — **нельзя** в transaction (только в migration с `op.get_bind().execute("COMMIT")` — избегаем; делаем data-fix отдельным maintenance скриптом, не в Alembic).
6. **Один файл миграции = одна логическая ступень.** `oi_samples + hypertable + compression + retention` живут вместе. CA `oi_5m/15m/30m` — отдельная миграция (хронологически после insert path работает). `oi_quality_5m` — третья миграция.

### 13.2 Header каждой миграции

Каждая Alembic-миграция, трогающая TimescaleDB, начинается с проверки extension и версии:

```python
"""create oi_samples hypertable

Revision ID: 0002_oi_samples_hypertable
Revises: 0001_init
Create Date: 2026-05-XX
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.sql import text

revision = "0002_oi_samples_hypertable"
down_revision = "0001_init"
branch_labels = None
depends_on = None


def _ensure_timescaledb() -> None:
    """Fail-fast: extension должна быть установлена до миграций (см. 13_OPERATIONS §2.1.2)."""
    op.execute(text("CREATE EXTENSION IF NOT EXISTS timescaledb"))
    # Минимальная версия для insert into compressed chunks (см. §4.3) — 2.11
    op.get_bind().execute(text("""
        DO $$
        DECLARE v TEXT;
        BEGIN
            SELECT extversion INTO v FROM pg_extension WHERE extname='timescaledb';
            IF string_to_array(v, '.')::int[] < ARRAY[2,11] THEN
                RAISE EXCEPTION 'TimescaleDB %, требуется ≥ 2.11', v;
            END IF;
        END $$;
    """))
```

> Вызывать `_ensure_timescaledb()` в **первой** миграции, создающей hypertable. Последующие могут полагаться на её эффект — extension не пропадёт.

### 13.3 Шаблон: hypertable + compression + retention (`oi_samples`)

```python
def upgrade() -> None:
    _ensure_timescaledb()

    # 1. Базовая таблица (DDL — стандартный SQLAlchemy)
    op.create_table(
        "oi_samples",
        sa.Column("ts_exchange",       sa.TIMESTAMP(timezone=True), nullable=False),
        sa.Column("ts_ingested",       sa.TIMESTAMP(timezone=True), nullable=False, server_default=sa.func.now()),
        sa.Column("exchange",          sa.Text, nullable=False),
        sa.Column("canonical_symbol",  sa.Text, nullable=False),
        sa.Column("native_symbol",     sa.Text, nullable=False),
        sa.Column("quote_asset",       sa.Text, nullable=False, server_default="USDT"),
        sa.Column("settle_asset",      sa.Text, nullable=False, server_default="USDT"),
        sa.Column("market_type",       sa.Text, nullable=False, server_default="usdt_perp"),
        sa.Column("oi_coins",          sa.Numeric(38, 18), nullable=False),
        sa.Column("oi_native_unit",    sa.Text, nullable=False),
        sa.Column("oi_notional_usdt",  sa.Numeric(38, 8),  nullable=False),
        sa.Column("price_used",        sa.Numeric(38, 18), nullable=False),
        sa.Column("price_source",      sa.Text, nullable=False),
        sa.Column("source_kind",       sa.Text, nullable=False),
        sa.Column("valuation_status",  sa.Text, nullable=False),
        sa.Column("normalization_version", sa.SmallInteger, nullable=False),
        sa.Column("instrument_version",    sa.Integer, nullable=False),
        sa.Column("warnings",          sa.ARRAY(sa.Text), nullable=False, server_default="{}"),
        sa.Column("raw_event_id",      sa.BigInteger, nullable=True),
        sa.Column("raw_hash",          sa.Text, nullable=False),
        sa.PrimaryKeyConstraint("exchange", "canonical_symbol", "ts_exchange",
                                name="pk_oi_samples"),
    )

    # 2. Hypertable (idempotent через if_not_exists => TRUE)
    op.execute(text("""
        SELECT create_hypertable(
            'oi_samples',
            'ts_exchange',
            chunk_time_interval => INTERVAL '1 day',
            if_not_exists       => TRUE
        )
    """))

    # 3. Compression
    op.execute(text("""
        ALTER TABLE oi_samples SET (
            timescaledb.compress,
            timescaledb.compress_segmentby = 'exchange, canonical_symbol',
            timescaledb.compress_orderby   = 'ts_exchange DESC'
        )
    """))
    op.execute(text("""
        SELECT add_compression_policy(
            'oi_samples',
            INTERVAL '7 days',
            if_not_exists => TRUE
        )
    """))

    # 4. Retention
    op.execute(text("""
        SELECT add_retention_policy(
            'oi_samples',
            INTERVAL '90 days',
            if_not_exists => TRUE
        )
    """))

    # 5. Дополнительные индексы
    op.create_index(
        "idx_oi_samples_canonical_time", "oi_samples",
        ["canonical_symbol", sa.text("ts_exchange DESC")],
    )
    op.create_index(
        "idx_oi_samples_exchange_time", "oi_samples",
        ["exchange", sa.text("ts_exchange DESC")],
    )
    op.create_index(
        "idx_oi_samples_recent_native", "oi_samples",
        ["exchange", "native_symbol", sa.text("ts_exchange DESC")],
    )


def downgrade() -> None:
    # Порядок: policies → compression off → indexes → table.
    # remove_*_policy в TimescaleDB не имеет if_exists; оборачиваем в DO/EXCEPTION.

    op.execute(text("""
        DO $$ BEGIN
            PERFORM remove_retention_policy('oi_samples', if_exists => TRUE);
        EXCEPTION WHEN undefined_function THEN
            PERFORM remove_retention_policy('oi_samples');
        END $$;
    """))
    op.execute(text("""
        DO $$ BEGIN
            PERFORM remove_compression_policy('oi_samples', if_exists => TRUE);
        EXCEPTION WHEN undefined_function THEN
            PERFORM remove_compression_policy('oi_samples');
        END $$;
    """))

    # decompress всех chunks перед drop (страховка; обычно retention уже почистил старые)
    op.execute(text("""
        SELECT decompress_chunk(c, if_compressed => TRUE)
        FROM show_chunks('oi_samples') c
    """))
    op.execute(text("ALTER TABLE oi_samples SET (timescaledb.compress = false)"))

    op.drop_index("idx_oi_samples_recent_native", table_name="oi_samples")
    op.drop_index("idx_oi_samples_exchange_time", table_name="oi_samples")
    op.drop_index("idx_oi_samples_canonical_time", table_name="oi_samples")

    # drop hypertable дропает все chunks; нет отдельного `drop_hypertable`
    op.drop_table("oi_samples")
```

> NB: `if_compressed => TRUE` для `decompress_chunk` — TimescaleDB ≥ 2.13. Если в проде версия меньше, использовать без аргумента и ловить exception в `DO $$ ... $$`.

### 13.4 Шаблон: continuous aggregate

```python
def upgrade() -> None:
    # CA нельзя создать через op.create_table — только raw DDL.
    op.execute(text("""
        CREATE MATERIALIZED VIEW IF NOT EXISTS oi_5m
        WITH (timescaledb.continuous) AS
        SELECT
            time_bucket('5 minutes', ts_exchange) AS bucket,
            exchange,
            canonical_symbol,
            native_symbol,
            FIRST(oi_coins, ts_exchange)         AS open_oi_coins,
            MAX(oi_coins)                        AS high_oi_coins,
            MIN(oi_coins)                        AS low_oi_coins,
            LAST(oi_coins, ts_exchange)          AS close_oi_coins,
            FIRST(oi_notional_usdt, ts_exchange) AS open_oi_notional_usdt,
            LAST(oi_notional_usdt, ts_exchange)  AS close_oi_notional_usdt,
            LAST(price_used, ts_exchange)        AS close_price,
            COUNT(*)                             AS sample_count,
            LAST(ts_exchange, ts_exchange)       AS last_ts_exchange,
            LAST(valuation_status, ts_exchange)  AS last_valuation_status,
            LAST(source_kind, ts_exchange)       AS last_source_kind,
            BOOL_OR(source_kind = 'degraded_fallback') AS has_degraded_source
        FROM oi_samples
        GROUP BY bucket, exchange, canonical_symbol, native_symbol
        WITH NO DATA
    """))

    op.execute(text("""
        SELECT add_continuous_aggregate_policy(
            'oi_5m',
            start_offset      => INTERVAL '1 hour',
            end_offset        => INTERVAL '30 seconds',
            schedule_interval => INTERVAL '30 seconds',
            if_not_exists     => TRUE
        )
    """))

    op.execute(text("""
        SELECT add_retention_policy(
            'oi_5m',
            INTERVAL '90 days',
            if_not_exists => TRUE
        )
    """))

    op.execute(text("""
        CREATE INDEX IF NOT EXISTS idx_oi_5m_lookup
        ON oi_5m (exchange, canonical_symbol, bucket DESC)
    """))


def downgrade() -> None:
    # Порядок: retention → CA refresh policy → index → CA.
    op.execute(text("""
        DO $$ BEGIN
            PERFORM remove_retention_policy('oi_5m', if_exists => TRUE);
        EXCEPTION WHEN undefined_function THEN
            PERFORM remove_retention_policy('oi_5m');
        END $$;
    """))
    op.execute(text("""
        DO $$ BEGIN
            PERFORM remove_continuous_aggregate_policy('oi_5m', if_exists => TRUE);
        EXCEPTION WHEN undefined_function THEN
            PERFORM remove_continuous_aggregate_policy('oi_5m');
        END $$;
    """))
    op.execute(text("DROP INDEX IF EXISTS idx_oi_5m_lookup"))
    op.execute(text("DROP MATERIALIZED VIEW IF EXISTS oi_5m"))
```

> **Hierarchical CA (`oi_15m`, `oi_30m` поверх `oi_5m`).** Та же схема, но `FROM oi_5m`. Drop-порядок строго от детей к родителю: сначала `oi_30m` и `oi_15m`, потом `oi_5m`.

### 13.5 Чек-лист перед merge миграции

- [ ] `upgrade()` идёт в порядке: table → hypertable → compression → retention → indexes → CA → CA policy.
- [ ] `downgrade()` — обратный порядок (см. §13.1 п.4).
- [ ] Все TS-функции, поддерживающие `if_not_exists`, вызваны с `if_not_exists => TRUE`.
- [ ] Все `remove_*_policy` обёрнуты в `DO $$ EXCEPTION ...` для совместимости с TS < 2.13.
- [ ] Миграция запущена дважды подряд на test-БД — второй прогон не падает (idempotency проверена).
- [ ] `alembic downgrade -1` → `alembic upgrade +1` работает на test-БД.
- [ ] Если миграция трогает hypertable, в которой уже есть данные на staging — оценен размер (`SELECT pg_size_pretty(pg_total_relation_size('oi_samples'))`), и время `decompress_chunk` для downgrade.
- [ ] Тест: `pytest tests/integration/test_migrations.py::test_upgrade_downgrade_cycle`.

### 13.6 Что НЕ делаем в Alembic-миграциях

- **Не вызываем `refresh_continuous_aggregate`, `compress_chunk`, `decompress_chunk` в `upgrade()` для существующих данных.** Эти операции долгие и не транзакционны; запускаем отдельным maintenance-скриптом из `app/tools/` (см. `13_OPERATIONS §6`).
- **Не делаем backfill через миграцию.** Любая data-операция (например, replay при bump `normalization_version`) — отдельный CLI tool, не Alembic.
- **Не трогаем `pg_hba.conf`, `postgresql.conf`, extension shared-state.** Они принадлежат host-машине (см. `15_DEPLOYMENT_INFRA §6`).

### 13.7 Cross-references

- Базовый DDL → `08 §2.1`.
- CA refresh policy и hierarchical setup → `08 §3`.
- Reprocessing tools (replay, backfill) — отдельно от Alembic → `08 §6`, `13_OPERATIONS §6`.
- TimescaleDB extension installation на host (one-time, до первой Alembic-миграции) → `13_OPERATIONS §2.1.2`, `15_DEPLOYMENT_INFRA §6.1`, `00_DECISIONS_LOG F17`.
- Roadmap M1 DoD (первая миграция) → `16_ROADMAP §M1`.
