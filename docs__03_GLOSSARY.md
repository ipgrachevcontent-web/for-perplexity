# 03 · Glossary

> Словарь терминов проекта `oi-tracker`. При сомнениях в трактовке — этот документ авторитетен.

---

## A. Биржевая семантика

### Open Interest (OI)
Количество открытых (нелiquidированных) позиций по конкретному фьючерсному контракту в данный момент. Хранится в количестве монет (`oi_coins`) и в эквиваленте USDT (`oi_notional_usdt`).

### USDT-M perpetual / USDT-M perp
Перпетуальный фьючерс с маржой и расчётом в USDT. Линейный (linear) контракт без даты экспирации. Единственный поддерживаемый тип контрактов в этом проекте.

### Linear vs Inverse
- **Linear** = USDT-M, маржа в USDT, цена в USDT, расчёт в USDT. Это наш scope.
- **Inverse** = COIN-M, маржа в base asset (BTC), расчёт в base asset. **Не поддерживается.**

### Base / Quote / Settle
Для контракта `BTC/USDT`:
- `base_asset = "BTC"` — что покупаем/продаём.
- `quote_asset = "USDT"` — в чём измеряется цена.
- `settle_asset = "USDT"` — в чём проходит расчёт PnL.

В нашем scope `quote_asset == settle_asset == "USDT"` всегда.

### Native symbol
Символ в той форме, в которой его отдаёт конкретная биржа.
Примеры:
- Binance: `BTCUSDT`
- OKX: `BTC-USDT-SWAP`
- Gate.io: `BTC_USDT` или `BTC_USDT_PERP`
- Hyperliquid: `BTC-USDT`
- XT: `btc_usdt` (lowercase)

### Canonical symbol
Унифицированное обозначение базового актива в нашей системе. Только базовый актив, без quote/suffix.
Примеры:
- `"BTC"`, `"ETH"`, `"SOL"`.

NB: рынок всегда USDT-M perp, поэтому `BASE` достаточно для идентификации в текущем scope.

### Mark price / Index price / Last price
- **Mark price** — расчётная справедливая цена контракта, обычно усреднённая между биржами + funding adjustment. Используется для liquidation calc.
- **Index price** — индекс спотовых цен на нескольких площадках.
- **Last price** — последняя цена сделки на этой бирже.

В нашем нормализаторе для конверсии `oi_coins → oi_notional_usdt` приоритет: `mark > index > last` (см. `07_NORMALIZER.md §4`).

### Contract size / Multiplier
Для бирж, выражающих OI в **контрактах** (XT, частично OKX), нужен `contract_size`:
```
oi_coins = oi_contracts × contract_size
```
Для большинства линейных контрактов `contract_size = 1`, но не везде.

### Snapshot vs Native interval
**Snapshot** — биржа отдаёт «текущее значение OI». Мы строим временной ряд, опрашивая каждую минуту.

**Native interval** — биржа сама агрегирует OI в бары (5m / 15m / 30m). Мы тянем готовые бары, строим временной ряд из них.

Примеры:
- Bybit `/v5/market/open-interest` — native_interval (`intervalTime` query parameter).
- KuCoin `/api/.../open-interest` — native_interval.
- Binance `/fapi/v1/openInterest` — snapshot (текущее значение, без bar-aggregation).

---

## B. Временные термины

### ts_exchange
**Тип:** `TIMESTAMPTZ`.
**Семантика:** время измерения OI на стороне биржи (биржевые часы).

Это **источник истины** для всех бизнес-вычислений: окон 5/15/30, Δ, алертов.

Hypertable `oi_samples` партиционируется именно по `ts_exchange` — см. `08_TIME_SERIES_STORAGE.md`.

### ts_ingested
**Тип:** `TIMESTAMPTZ`.
**Семантика:** время, когда наш сервер получил и записал точку в БД.

Используется для диагностики: `ts_ingested - ts_exchange` = сетевой + processing lag.

Не используется для бизнес-логики.

### ts_processed
**Тип:** `TIMESTAMPTZ`.
**Семантика:** время, когда normalizer + alert engine завершили обработку точки.

Используется для измерения end-to-end latency: `ts_processed - ts_exchange` = полный pipeline lag.

### Bucket / Time bucket
Дискретный интервал, к которому приписывается агрегат. Примеры:
- 5-min bucket для `BTC` на Binance, начинающийся в `12:00:00 UTC`, агрегирует все точки с `ts_exchange ∈ [12:00:00, 12:05:00)`.
- TimescaleDB `time_bucket('5 minutes', ts_exchange)` функция.

### Freshness
Свойство точки: «насколько она свежая относительно now()».
**Формула:** `freshness_age = now() - ts_exchange`.
**Лимиты** (см. `00_DECISIONS_LOG.md C7`):
- «Свежесть последней точки для алерта»: `2 × poll_interval` = 120s.
- «Допуск на поиск точки t-N»: `1.5 × poll_interval` = 90s.

### Staleness
Состояние, противоположное freshness: точка «устарела».
- `stale = freshness_age > freshness_budget`.
- При stale = true алерт по этой точке не считается, состояние машины не двигается.

### Gap
Промежуток между двумя соседними `ts_exchange` для одной пары `(exchange, canonical_symbol)`. Большой gap = пропуск.
**Norma:** `~60s` (= polling interval).
**Подозрительно:** `> 120s` (двойной интервал).
**Алерт:** `gap > 5 × poll_interval` для активного символа = `exchange-health` сигнал.

### Jitter
Расброс ts_exchange внутри одного логического минутного цикла. Биржи возвращают точки чуть в разные секунды; jitter обычно ±5 сек. Учитывается в freshness формулах.

### Watermark
Маркер прогресса инкрементального fetch. Для каждой пары `(exchange, canonical_symbol)` хранится `last_ts_fetched`, чтобы при следующем цикле тянуть только новое.

### Backfill / Replay
- **Backfill** — повторное получение исторических точек, пропущенных из-за сбоя (полная заполняемость диапазона).
- **Replay** — повторная нормализация уже сохранённых сырых событий с обновлёнными правилами (`normalization_version` bump).

Оба не создают retro-fire алертов (см. `00_DECISIONS_LOG.md F9`).

---

## C. Качество данных

### normalization_version
Целое число, инкрементируется при изменении правил нормализатора (формул конверсии единиц, instrument mapping). Хранится в каждой записи `oi_samples`. Позволяет replay при изменении правил.

### source_kind
Enum: происхождение точки.
- `snapshot` — получена через snapshot polling (наша derived time series).
- `native_interval` — получена из native interval API биржи.
- `backfill` — получена через backfill процедуру (заполнение пропусков).
- `replay` — повторно нормализована с новой `normalization_version`.
- `degraded_fallback` — primary источник недоступен, перешли на резервный (например native_interval → snapshot для Bybit при деградации).

### valuation_status
Enum: качество расчёта `oi_notional_usdt`.
- `authoritative` — биржа сама вернула notional (Binance `sumOpenInterestValue`, etc.).
- `good_estimate` — `oi_coins × mark_price` (mark price стабилен и доступен).
- `low_confidence` — fallback на `last_price` или приблизительная конверсия.

Алерты по умолчанию разрешены при `>= good_estimate`.

### Symbol coverage ratio
Доля активных символов биржи, по которым в текущем минутном цикле получена точка.
**Формула:** `coverage = received_points / active_symbols_count`.
**Норма:** `≥ 95%`.
**Алерт `exchange-health`:** `< 95%`.

---

## D. Алерты

### Window
Окно расчёта Δ. В системе три фиксированных окна: `5m`, `15m`, `30m`.

### Threshold
Пороговое значение Δ для срабатывания. Задаётся per (window, direction):
- `up_5m` — порог роста на 5-мин окне.
- `down_5m` — порог падения (отрицательное число).
- `up_15m`, `down_15m`, `up_30m`, `down_30m`.

### Crossing / Rising edge
Момент, когда Δ перешёл через threshold снизу-вверх (или сверху-вниз для падения).
- Было `Δ = 4.8%`, стало `Δ = 5.2%` при `threshold = 5%` → rising edge crossing.
- Было `Δ = 7.1%`, стало `Δ = 6.8%` (всё ещё выше) → не crossing, а continuation.

Алерт срабатывает только на crossing.

### State machine states
- `idle` — для пары `(rule, exchange, canonical_symbol, window)` нет истории / новый символ.
- `armed` — следим, текущий Δ ниже threshold.
- `fired` — пересекли threshold, алерт отправлен.
- `cooldown` — N минут блокировки от повторов (smart cooldown — см. ниже).

Переходы — в `09_ALERT_ENGINE.md §3`.

### Cooldown
Блокировка повторных алертов после fire.
**Default:** 15 минут.

### Hard cooldown vs Smart cooldown
- **Hard:** N минут полная тишина по ключу `(rule, exchange, symbol, window)`.
- **Smart:** разрешено повторное срабатывание внутри cooldown, **если** новое Δ ≥ `1.5 × previous_fired_delta`.

В нашем проекте — smart (см. `00_DECISIONS_LOG.md Q1 + F7`).

### Dedupe key
Ключ дедупликации алертов:
```
rule_id:<id>|exchange:<ex>|symbol:<sym>|window:<w>|bucket:<ts>
```
Если такой ключ уже отправлен — повтор запрещён.

### Confirm points
Сколько последовательных evaluation cycles должны подтвердить условие до fire.
**Default:** 1 (rising edge fires immediately).
**Можно увеличить:** 2 → ждём 2 cycles, что снижает шум, но добавляет 1 минуту латентности.

### Min history
Минимум собственной истории символа на бирже, до которого алерты не считаются.
**Формула:** `max(30, 2 × window_size_minutes)` минут.

### Min OI notional
Floor-фильтр на абсолютное значение OI. Алерт не считается, если `oi_notional_usdt < min_oi_notional`.
**Default:** `5_000_000` USDT.

### Signal types
Пять типов сигналов:
1. **Threshold** — `Δ_window crosses threshold`.
2. **Confirmed** — threshold с `confirm_points >= 2`.
3. **Consensus** — тот же canonical_symbol triggers на ≥ M бирж одновременно.
4. **Divergence** — рост OI + падение цены, или наоборот.
5. **Exchange-health** — операционный (биржа stale, coverage упал).

---

## E. Хранение

### Hypertable
Тип таблицы TimescaleDB, автоматически партиционируемой по времени. Под капотом — обычные PostgreSQL chunks (партиции).

### Chunk
Партиция hypertable, охватывающая определённый временной диапазон. У нас `chunk_time_interval = INTERVAL '1 day'`.

### Continuous aggregate (CA)
Materialized view, автоматически обновляющийся в TimescaleDB. У нас три:
- `oi_5m` — поверх `oi_samples`.
- `oi_15m` — поверх `oi_5m` (hierarchical).
- `oi_30m` — поверх `oi_5m` (hierarchical).

### Retention policy
Автоматическое удаление chunks старше N дней. У нас N = 90.

### Compression policy
Автоматическое сжатие chunks старше M дней. У нас M = 7. Сжатие 10–20×.

### segmentby / orderby (TS compression)
Параметры compression:
- `segmentby = exchange, canonical_symbol` — компрессия группирует по этим колонкам, что эффективно для time-series.
- `orderby = ts_exchange DESC` — порядок внутри сегмента, влияет на скорость range-запросов.

---

## F. Доставка

### SSE (Server-Sent Events)
HTTP-протокол для push-уведомлений с сервера в браузер. Однонаправленный (server → client), переподключаемый, легче WebSocket. Используется для real-time обновлений UI.

### REST API
Внутренний API для UI. Префикс `/api/v1/`. Без публичных потребителей; авторизация — на сетевом уровне.

### Delivery queue
Таблица `delivery_queue` в БД, выполняющая роль очереди для TG sender. Alert engine пишет, sender читает.

### Delivery job
Запись в `delivery_queue`:
- `channel = "telegram"`,
- `payload = JSON message`,
- `status ∈ {pending, sending, sent, failed}`,
- `retry_count`.

### TG sender
Воркер `oi-tracker-tg-sender.service`, читающий `delivery_queue` каждые 2 секунды, отправляющий через aiogram.

---

## G. Сокращения

| Abbr | Расшифровка |
|---|---|
| OI | Open Interest |
| TS | TimescaleDB / time-series |
| CA | Continuous aggregate |
| SSE | Server-Sent Events |
| TG | Telegram |
| FK | Foreign key |
| PK | Primary key |
| DDL | Data Definition Language (CREATE TABLE и т.п.) |
| DML | Data Manipulation Language (INSERT/UPDATE/DELETE) |
| RPO | Recovery Point Objective |
| RTO | Recovery Time Objective |
| SLO | Service Level Objective |
| SLA | Service Level Agreement |
| FR | Functional Requirement |
| NFR | Non-Functional Requirement |
| LTTB | Largest Triangle Three Buckets (decimation algorithm) |
| KPI | Key Performance Indicator |
| HA | High Availability |

---

## H. Cross-references

| Термин | Подробнее |
|---|---|
| OI Tracker scope | `01_PRODUCT_SPEC.md §3, §4` |
| Architecture | `02_ARCHITECTURE.md` |
| ts_exchange / ts_ingested / ts_processed | `04_TIME_MODEL.md` |
| RawExchangeEvent / NormalizedOISample / etc. | `05_DATA_CONTRACTS.md` |
| Instruments lifecycle | `06_INSTRUMENT_REGISTRY.md` |
| Normalization pipeline | `07_NORMALIZER.md` |
| oi_samples / CA / retention DDL | `08_TIME_SERIES_STORAGE.md` |
| State machine, signal types, cooldown | `09_ALERT_ENGINE.md` |
| TG, REST, SSE | `10_DELIVERY_LAYER.md` |
| Per-exchange specifics | `11_EXCHANGE_ADAPTERS/<exchange>.md` |
| Metrics, dashboards, SLO | `12_OBSERVABILITY_SLO.md` |
| systemd, deployment, runbooks | `13_OPERATIONS.md` |
| Test pyramid, fixtures | `14_TEST_STRATEGY.md` |
