## 1. Что хранит time‑series storage
Он принимает от normalizer объект вида:
```json
{
  "exchange": "bybit",
  "native_symbol": "BTCUSDT",
  "canonical_symbol": "BTCUSDT",
  "ts_exchange": 1714478700000,
  "oi_native": 1234.56,
  "oi_native_unit": "BTC",
  "oi_notional_usdt": 93264129.96,
  "price_used": 75541.00,
  "normalization_version": 3
}
```
И хранит как raw time‑series по time_bucket и exchange, symbol одновременно, а не просто как “ещё одну таблицу в PostgreSQL”.

---

## 2. Схема таблиц
Рекомендуется две основные таблицы:
### 2.1 normalized_oi_samples (гипертаблица)
```sql
CREATE TABLE normalized_oi_samples (
  id SERIAL PRIMARY KEY,
  exchange TEXT NOT NULL,       -- 7 бирж + остальные
  native_symbol TEXT NOT NULL,  -- BTCUSDT, etc.
  canonical_symbol TEXT NOT NULL, -- BTCUSDT, etc.
  market_type TEXT,             -- usdt_perp
  ts_exchange BIGINT NOT NULL,  -- ms
  ts_ingested TIMESTAMPTZ NOT NULL, -- время записи в БД
  oi_native NUMERIC NOT NULL,
  oi_native_unit TEXT,          -- "BTC", "contracts", "BTCUSDT notional"
  oi_notional_usdt NUMERIC NOT NULL,
  price_used NUMERIC,           -- для проверки
  raw_hash TEXT,                -- dup‑key
  normalization_version INT
);

-- time‑series hypertable
SELECT create_hypertable('normalized_oi_samples', 'ts_ingested');
```

Это сырой временной ряд, по которому:
- поиски по time идут быстро;
- можно делать time_bucket и агрегации;
- реализуется retention_policy по ts_ingested или ts_exchange.

### 2.2 oi_metrics_5m_15m_30m (continuous aggregates)
```sql
-- 5 минут
CREATE MATERIALIZED VIEW oi_5m
WITH (timescaledb.continuous) AS
SELECT
  time_bucket('5 minutes', ts_exchange) AS bucket,
  exchange,
  canonical_symbol,
  FIRST(oi_notional_usdt, ts_exchange) AS open_5m,
  MAX(oi_notional_usdt) AS high_5m,
  MIN(oi_notional_usdt) AS low_5m,
  LAST(oi_notional_usdt, ts_exchange) AS close_5m,
  COUNT(*) AS sample_count
FROM normalized_oi_samples
GROUP BY 1, 2, 3;

-- 15 минут (строим поверх 5‑минутного time‑bucket, если есть)
-- 30 минут — аналогично
```

*Такие continuous aggregates автоматически пересчитываются по расписанию, и запросы по 5/15/30‑минутным окнам выполняются за миллисекунды, а не через ручной GROUP BY ts_exchange по всей таблице.*

---

## 3. Принцип работы при записи
### 3.1. Вставка новых точек
Каждый normalized_oi_sample попадает в normalized_oi_samples как INSERT ... ON CONFLICT (exchange, native_symbol, ts_exchange) DO NOTHING (anti‑duplicating по бирже‑символу‑времени).

Типичный паттерн:
```python
stmt = """
INSERT INTO normalized_oi_samples (
  exchange, native_symbol, canonical_symbol,
  market_type, ts_exchange, ts_ingested,
  oi_native, oi_native_unit, oi_notional_usdt,
  price_used, raw_hash, normalization_version
)
VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
ON CONFLICT (exchange, native_symbol, ts_exchange) DO NOTHING
"""
```
*Такой append‑only и upsert‑very‑rare режим даёт максимально высокую throughput.*

### 3.2. Partitioning и индексы
Hypertable в TimescaleDB автоматически разбивает normalized_oi_samples по ts_ingested (например, по часам/дням). Это означает:
- queries по ts_ingested BETWEEN ... пропускают ненужные chucks;
- time_bucket‑функции работают эффективно;
- retention_policy удаляет старые чанки, а не сотни миллионов строк по одной.

Индексом по умолчанию достаточно:
- ts_ingested
- exchange
- canonical_symbol

*При крупном масштабе можно добавить ts_exchange и market_type.*

---

## 4. Как время‑серии дают окна 5/15/30 минут
### 4.1. Сырые точки → окна
Для расчёта дельт за 5/15/30 минут нужно найти последнюю точку и точку, отстоящую назад на заданный интервал. В raw‑таблице это делается через LATERAL JOIN или window functions, но при десятках тысяч символов это медленно, поэтому и нужны continuous aggregates.

### 4.2. Continuous aggregates как “готовые окна”
- oi_5m — уже готовый 5‑минутный bar OI по exchange + symbol.
- oi_15m и oi_30m строятся поверх более мелких бакетов, а не из normalized_oi_samples с нуля.

Такой подход даёт:
- быстрые графики и heatmaps на веб‑сайте;
- почти мгновенные списки “top movers” за 5/15/30 минут;
- preview‑данные для алерт‑движка без тяжёлых GROUP BY на лету.

---

## 5. Принцип работы при чтении
### 5.1. История по бирже
Ты читаешь normalized_oi_samples с фильтрацией:
```sql
SELECT *
FROM normalized_oi_samples
WHERE exchange = 'bybit'
  AND ts_exchange BETWEEN 1714468200000 AND 1714478700000
ORDER BY ts_exchange
```

*Это даст тебе сырую историю по одной бирже именно так, как ты задумал.*

### 5.2. Метрики для алертов
Для алерт‑движка удобно читать continuous_aggregates и добавлять LAG‑подобные аналитические функции:
```sql
SELECT
  bucket,
  exchange,
  canonical_symbol,
  close_5m,
  LAG(close_5m, 1) OVER (PARTITION BY exchange, canonical_symbol ORDER BY bucket) AS prev_5m
FROM oi_5m
WHERE bucket >= now() - INTERVAL '2 hours'
```
*Разница close_5m - prev_5m уже даёт тебе delta_abs, а отношение — delta_pct, которые можно отправлять в алерт‑движок.*

### 5.3. UI‑сервис
Веб‑приложение и Telegram‑бот читают либо:
- normalized_oi_samples для детального графика;
- oi_5m/15m/30m для summary‑таблиц и heatmaps, где абсолютная синхронность с секундными точками не критична.

---

## 6. Politica жизненного цикла и качества
### 6.1. Retention policies
Timescale умеет add_retention_policy по ts_ingested или ts_exchange:
```sql
SELECT add_retention_policy(
  'normalized_oi_samples',
  INTERVAL '90 days'
);
```

Ты можешь:
- хранить сырой ряд 90 дней;
- агрегаты 5/15/30 минут — 1 год (чанки можно удалять дольше, так как они уже сжаты).

### 6.2. Compression
Timescale‑compression применяется к normalized_oi_samples после N дней:
```sql
ALTER TABLE normalized_oi_samples
SET (
  timescaledb.compress,
  timescaledb.compress_orderby = 'ts_exchange DESC'
);
```
Это даёт до 10–20× сжатие без потери доступности для запросов.

### 6.3. Quality‑контур внутри storage
Внутри time‑series storage стоит держать метрики качества:
- missing_samples_count по каждому exchange + canonical_symbol (разница между ts_exchange соседних точек);
- stale_ts_exchange — если MAX(ts_exchange) по инструменту сильно отстаёт от now();
- hourly_sample_count — для контроля, что polling не сломался.

Такие метрики можно хранить в отдельной quality_metrics‑таблице, заполняемой агрегационными job’ами, и использовать как триггер для алерта “внешний контур потерял данные по бирже X”.

---

## 7. Как это всё связано с normalizer
- normalizer выпускает канонические NormalizedOISample и пишет их в normalized_oi_samples.
- time‑series storage заботится о time_bucket, continuous_aggregates, retention и compression.
- alert engine и delivery layer лишь читают уже нормализованные и агрегированные ряды, они не знают, какой именно коннектор их собирал.

---

## 8. Краткий summary принципа
Time‑series storage работает так:
- принимает от normalizer канонические OI‑точки;
- хранит их как time‑series hypertable с time_bucket и exchange + symbol фильтрами;
- выстраивает continuous aggregates под 5/15/30‑минутные окна;
- держит retention и compression для долгосрочного хранения;
- предоставляет алерт‑движку и UI‑слою уже готовые временные ряды, а не “сырую‑сырую‑сырую” таблицу.
