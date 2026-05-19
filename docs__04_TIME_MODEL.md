# 04 · Time Model

> Временные оси системы. Как работает каждая, что считается источником истины, как обрабатываются клок-дрейф и поздние данные.
> Связано: `00_DECISIONS_LOG.md C7, C11, F5`, `03_GLOSSARY.md §B`, `08_TIME_SERIES_STORAGE.md`.

---

## 1. Три временные оси

В системе есть **три семантически разные** временные метки. Они **никогда** не взаимозаменяемы.

| Ось | Тип | Источник | Назначение |
|---|---|---|---|
| `ts_exchange` | `TIMESTAMPTZ` | биржевые часы | Бизнес-логика: окна 5/15/30, Δ, алерты |
| `ts_ingested` | `TIMESTAMPTZ` | наш сервер при INSERT в БД | Операционная диагностика: lag сети + parsing |
| `ts_processed` | `TIMESTAMPTZ` | наш сервер при завершении alert evaluation | End-to-end SLO трассировка |

### 1.1 Инварианты

```
ts_exchange ≤ ts_ingested ≤ ts_processed
```

Это инвариант **в норме**. Нарушения возможны при clock skew (см. §5) и должны вызывать алерт.

### 1.2 Где и как заполняется каждое

| Ось | Кто заполняет | Когда |
|---|---|---|
| `ts_exchange` | биржевой коннектор из payload биржи | при парсинге response |
| `ts_ingested` | PostgreSQL `DEFAULT now()` | при INSERT в `raw_exchange_events` |
| `ts_processed` | alert engine | после rule evaluation для конкретной точки |

---

## 2. ts_exchange — источник истины

### 2.1 Почему именно ts_exchange

Все бизнес-вопросы — про рыночное время:
- «Δ OI за последние 5 минут» — это 5 минут market time, не серверного времени.
- «Алерт сработал в 12:15 UTC» — это `ts_exchange`, а не «когда наш сервер это обработал».
- Continuous aggregates по 5-минутным bucket'ам — выровнены по `ts_exchange`, чтобы биржевой бар совпадал с нашим bucket'ом.

### 2.2 Что хранит ts_exchange

| Source kind | Что хранит ts_exchange |
|---|---|
| `snapshot` | `time` поле из payload биржи (если есть); иначе `now()` коннектора в момент успешного fetch |
| `native_interval` | `bucket_end_ts` или `bucket_start_ts` из биржевого ответа (зависит от биржи; конвенция в `11_EXCHANGE_ADAPTERS/<exchange>.md`) |

### 2.3 Convention для native_interval

Для бирж с native interval API всегда сохраняем `bucket_end_ts` как `ts_exchange`. Это означает, что точка с `ts_exchange = 12:05:00` представляет 5-минутный бар `[12:00:00, 12:05:00)` биржевого времени.

При расчёте Δ это даёт корректное «t-N» для t=12:05:00 → t-5=12:00:00, что = bucket_end предыдущего 5-min бара.

### 2.4 Convention для `volume_samples_5m` (F51, Aster-only)

`volume_samples_5m.bucket` хранит `kline.openTime` — **bucket-start**, не bucket-end. Это расходится с native_interval convention (§2.3), но это **другая ось**, не `ts_exchange`:

- Hypertable `oi_samples` партиционируется по `ts_exchange`.
- Hypertable `volume_samples_5m` партиционируется по `bucket`.

Соседние бакеты идут шагом ровно 5 мин (`bucket_now - bucket_prev = 5min`), потому что биржа сама агрегирует kline'ы — никакого drift'а от polling latency.

**Freshness semantics для volume-сигнала иной.** `VolumeThresholdSignal` (`09_ALERT_ENGINE.md §5.6`) вычисляет `sample_age_seconds = now − current.ts_ingested`, НЕ `now − bucket`. Причина: закрытый 5m-бакет по определению ≥ 5 мин старше `now` (`bucket + 5min ≤ now`), стандартный OI-style budget 120s отсёк бы каждый candidate. На volume-pipeline «свежесть» это «когда мы прочитали kline в последнем poll'е», а не «время bucket'а».

---

## 3. ts_ingested — операционная ось

### 3.1 Назначение

`ts_ingested` существует для:
- Диагностики latency: `ts_ingested - ts_exchange` = network + connector parse time.
- Метрики `oi_collector_fetch_latency_seconds`.
- Аудита: «когда мы реально получили эту точку».
- Ничего больше.

### 3.2 Когда не использовать

❌ **Не использовать `ts_ingested` для:**
- Расчёта Δ.
- Поиска точки t-N.
- Bucket alignment.
- Continuous aggregates time_bucket.
- Ordering в alerts.

Если в коде попадёт `ts_ingested` там, где должен быть `ts_exchange`, это баг — детектируется linter правилом + ревью.

---

## 4. ts_processed — end-to-end трассировка

### 4.1 Назначение

`ts_processed` отмечает момент, когда:
1. Точка прошла normalizer.
2. CA refresh захватил её.
3. Alert engine проверил все применимые правила.
4. Решение (FIRE / SUPPRESS / no-op) записано в `alert_events`.

После `ts_processed` точка считается «полностью обработанной».

### 4.2 SLO трассировка

```
e2e_latency_seconds = ts_processed - ts_exchange
```

Это и есть наша основная end-to-end метрика для SLO `Q3` (`90s P95`, `180s P99`).

### 4.3 Где хранится

В `alert_events` для каждого decision:
```sql
CREATE TABLE alert_events (
    ...
    ts_exchange   TIMESTAMPTZ NOT NULL,
    ts_processed  TIMESTAMPTZ NOT NULL DEFAULT now(),
    ...
);
```

Гистограмма `e2e_latency_seconds` экспортируется в Prometheus.

---

## 5. Clock skew (расхождение часов)

### 5.1 Проблема

Биржевые серверы могут иметь свои часы. На stable биржах skew обычно `< 100ms`, но бывают случаи до `±10 секунд` при NTP-проблемах биржи.

Симптом: `ts_ingested < ts_exchange` (биржа «из будущего»).

### 5.2 Как обрабатываем

**На уровне normalizer:**
- Если `ts_exchange > ts_ingested + tolerance` (default `tolerance = 30s`) — это аномалия:
  - Запись всё равно сохраняется (`ts_exchange` остаётся биржевым).
  - В `quality_meta.warnings` добавляется `clock_skew_forward`.
  - Считается ok, без блокировки.
- Если `ts_exchange < ts_ingested - tolerance_backward` (default `tolerance_backward = 300s`) — биржа «из прошлого»:
  - Это часто означает stale data или повторный отдач кэша.
  - Помечается `clock_skew_backward`, `valuation_status` понижается.

**На уровне monitoring:**
- Метрика `oi_clock_skew_seconds` per (exchange).
- Алерт `exchange-health` при `|skew| > 30s` стабильно держится 5 циклов подряд.

### 5.3 Bucket alignment под skew

`time_bucket('5 minutes', ts_exchange)` использует биржевое время. Если биржа сдвинута на 7 секунд вперёд, bucket boundary тоже сдвигается на 7 секунд для этой биржи относительно других. Это **намеренно** — внутри биржи всё консистентно, что важнее cross-exchange синхронизации.

_(Cross-exchange сравнение через consensus-сигнал sunset per `00 F40`. См. `09_ALERT_ENGINE.md §5.3`.)_

---

## 6. Freshness — формулы

Все freshness budgets — функции от `poll_interval` (по умолчанию 60s). Захардкоженных чисел в коде быть не должно.

### 6.1 Таблица budget'ов

| Контекст | Формула | Default | Хранится в |
|---|---|---|---|
| Свежесть для алерта (now) | `2 × poll_interval` | 120s | `settings.freshness_budget_now_sec` |
| Допуск для t-N точки | `1.5 × poll_interval` | 90s | `settings.freshness_budget_lookback_sec` |
| Native interval grace | `bucket_size + 60s` | varies | per-rule в `alert_rules` |
| Symbol coverage timeout | `3 × poll_interval` | 180s | `settings.coverage_timeout_sec` |
| Clock skew warning threshold | const `30s` | 30s | hardcoded |
| Clock skew critical threshold | const `300s` | 300s | hardcoded |

### 6.2 Применение

**Alert engine quality gate:**
```python
def is_fresh_for_alert(point: NormalizedOISample) -> bool:
    age = now() - point.ts_exchange
    return age <= settings.freshness_budget_now_sec
```

**Lookback search для Δ:**
```python
def find_t_minus_n(symbol, exchange, target_ts) -> Optional[OISample]:
    # ищем ближайшую точку в окне [target_ts - tolerance, target_ts + tolerance]
    tolerance = timedelta(seconds=settings.freshness_budget_lookback_sec)
    return repo.find_nearest_sample(symbol, exchange, target_ts, tolerance)
```

Если ближе чем tolerance точки нет → Δ = null → алерт не считается.

---

## 7. Late-arriving data

### 7.1 Что это

Точка с `ts_exchange = 12:05:00`, прибывшая в `12:08:30` (слишком поздно для текущего цикла). Причины:
- Сетевые задержки.
- Backfill после рестарта.
- Replay при изменении `normalization_version`.

### 7.2 Поведение системы

**Storage (TimescaleDB):**
- Точка вставляется в `oi_samples` нормально, попадает в правильный chunk по `ts_exchange`.
- Если chunk уже compressed — TimescaleDB автоматически decompress + insert + recompress (медленно, но работает).
- При массовом backfill (replay) — отдельная процедура с `decompress_chunk()` явно (см. `08_TIME_SERIES_STORAGE.md §6`).

**Continuous aggregates:**
- Refresh policy запускает `refresh_continuous_aggregate()` на интервале `[now() - 1 hour, now()]` каждые 30 секунд.
- Если поздняя точка попадает в этот диапазон — CA автоматически пересчитывает соответствующий bucket.
- Если точка старше — нужно вручную вызвать `refresh_continuous_aggregate()` на нужный интервал (часть процедуры backfill/replay).

**Alert engine:**
- Поздние точки не триггерят retro-fire алертов.
- Если `now() - ts_exchange > freshness_budget_now_sec` → точка считается stale, alert engine её игнорирует для FIRE-решений.
- Per `00_DECISIONS_LOG.md F43` decision `late_data_skip` НЕ пишется в `alert_events` (counter-only через `oi_alert_engine_decisions_total{decision="late_data_skip"}`). До F43 запись была в audit-таблицу — теперь только в counter.

### 7.3 Reprocessing — отдельная процедура

См. `08_TIME_SERIES_STORAGE.md §6` и `00_DECISIONS_LOG.md F9`. Ключевое:
- Backfill / replay = операционная процедура, запускается вручную или по расписанию.
- Никогда не создаёт retro-fire.
- Может пометить уже отправленные алерты как `historically_inaccurate`.

---

## 8. Polling alignment

### 8.1 Минутная решётка

Scheduler выровнен по UTC-минутам:
- Цикл начинается в `00:00:00.000`, `00:01:00.000`, `00:02:00.000` UTC.
- Tolerance запуска: `±100ms` (asyncio loop overhead).

### 8.2 Что внутри одного цикла

```
T:00.000  Scheduler tick
T:00.005  Все 12 connectors стартуют параллельно
T:00.500  Connector(Binance) начинает fetch (300 запросов с rate limit ~5/sec → 60 sec)
T:00.500  Connector(Bybit) начинает fetch (1 batch query → 200ms)
...
T:30.000  Большинство connectors закончили
T:55.000  Slowest connector закончил
T:55.500  Normalizer обработал все события
T:56.000  Storage записал
T:56.500  CA refresh подхватил
T:57.000  Alert engine evaluated
T:58.000  TG sender отправил все pending алерты
T:60.000  Следующий цикл стартует
```

### 8.3 Что если цикл не уложился в 60 сек

- **Норма:** уложились в 60s — следующий цикл стартует ровно по минуте.
- **Перебор:** scheduler видит, что предыдущий цикл ещё работает → пропускает текущий tick (логирует `cycle_overlap`), ждёт следующего.
- При двух подряд `cycle_overlap` — алерт `overload`.

---

## 9. Bucket boundaries для native interval

### 9.1 Биржевые bucket'ы vs наши

Bybit отдаёт 5-min OI бары по своей минутной решётке (UTC). KuCoin тоже.

Наши TimescaleDB CA `oi_5m` тоже по UTC 5-минутной решётке.

**При совпадении границ:** биржевой бар `[12:00, 12:05)` ↔ наш bucket `[12:00, 12:05)` 1:1.

**При несовпадении** (если биржа использует свою timezone): mapping per-exchange конкретно описан в `11_EXCHANGE_ADAPTERS/<exchange>.md`. По текущим спецификациям все биржи в нашем scope используют UTC, поэтому совпадение.

### 9.2 Как сохраняем native interval точки

```python
# Для Bybit native_interval, бар [12:00:00, 12:05:00)
sample = NormalizedOISample(
    ts_exchange = datetime(2026, 4, 30, 12, 5, 0, tzinfo=UTC),  # bucket_end_ts
    source_kind = SourceKind.NATIVE_INTERVAL,
    ...
)
```

При запросе `time_bucket('5 minutes', ts_exchange)` для этой точки получим `12:00:00` (TS bucket'ы используют bucket_start). Для вычисления Δ это семантически верно: точка с `ts_exchange=12:05:00` представляет состояние OI **в конце** интервала `[12:00, 12:05)`.

---

## 10. Diagnostics queries

### 10.1 Latency distribution per exchange

```sql
SELECT
  exchange,
  PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (ts_ingested - ts_exchange))) AS p50_seconds,
  PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (ts_ingested - ts_exchange))) AS p95_seconds,
  PERCENTILE_DISC(0.99) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (ts_ingested - ts_exchange))) AS p99_seconds
FROM oi_samples
WHERE ts_exchange >= now() - INTERVAL '1 hour'
GROUP BY exchange
ORDER BY p95_seconds DESC;
```

### 10.2 Clock skew detection

```sql
SELECT
  exchange,
  AVG(EXTRACT(EPOCH FROM (ts_ingested - ts_exchange))) AS avg_skew_sec,
  STDDEV(EXTRACT(EPOCH FROM (ts_ingested - ts_exchange))) AS stddev_skew_sec
FROM oi_samples
WHERE ts_exchange >= now() - INTERVAL '15 minutes'
GROUP BY exchange
ORDER BY avg_skew_sec;
```

Аномалия: `avg_skew_sec < 0` (биржа «из будущего») или `> 60` (биржа сильно отстала).

### 10.3 End-to-end latency

```sql
SELECT
  PERCENTILE_DISC(0.5) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (ts_processed - ts_exchange))) AS p50_e2e,
  PERCENTILE_DISC(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (ts_processed - ts_exchange))) AS p95_e2e,
  PERCENTILE_DISC(0.99) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (ts_processed - ts_exchange))) AS p99_e2e
FROM alert_events
WHERE ts_processed >= now() - INTERVAL '1 hour';
```

SLO check: `p95_e2e ≤ 90`, `p99_e2e ≤ 180`.

---

## 11. Pitfalls — типичные ошибки

| Что лучше не делать | Почему |
|---|---|
| `time_bucket(..., ts_ingested)` | смешивает rendering bug'и в data; CA становятся неконсистентны при late data. |
| Использовать `now()` в alert engine для определения «свежести» точки | clock drift делает это ненадёжным; используем `ts_exchange + freshness_budget`. |
| Использовать `ts_exchange BETWEEN now() - INTERVAL '5 minutes' AND now()` | если биржа в будущем (clock skew forward), запрос пропускает данные. Лучше `ts_exchange >= now() - INTERVAL '5 minutes'`. |
| Хранить `ts_exchange` как BIGINT (epoch ms) | TS time_bucket для целочисленных колонок имеет другую сигнатуру; неудобство везде. Используем `TIMESTAMPTZ`. |
| Округлять `ts_exchange` до минут на стороне коннектора | теряется реальная точка времени, ломается lag-диагностика. Сохраняем как пришло. |

---

## 12. Cross-references

- DDL hypertable по `ts_exchange` → `08_TIME_SERIES_STORAGE.md §2`.
- Freshness в alert engine → `09_ALERT_ENGINE.md §4`.
- Latency метрики → `12_OBSERVABILITY_SLO.md §3`.
- E2E SLO → `01_PRODUCT_SPEC.md §6 NFR-1`.
- Reprocessing процедура → `08_TIME_SERIES_STORAGE.md §6`.
