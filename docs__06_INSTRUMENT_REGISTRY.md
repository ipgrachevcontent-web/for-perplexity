# 06 · Instrument Registry

> Master data об инструментах. Lifecycle, sync, версионирование, инвалидация.
> Связано: `00_DECISIONS_LOG.md F2`, `05_DATA_CONTRACTS.md §11`, `07_NORMALIZER.md §3`.

---

## 1. Назначение

`Instrument Registry` — это локальный справочник USDT-M perpetual инструментов, поддерживаемый по всем 12 биржам. Без него normalizer не может:
- Разрешить `native_symbol → canonical_symbol`.
- Узнать `contract_size` для конверсии `oi_contracts → oi_coins`.
- Определить, является ли символ актуальным (active vs delisted).

Это **критическая зависимость** для корректности `oi_notional_usdt`. Без актуального registry будет ошибочная оценка OI.

---

## 2. Хранение

### 2.1 Таблица `instruments`

См. `05_DATA_CONTRACTS.md §11.3`. Ключевые свойства:
- `UNIQUE (exchange, native_symbol)` — гарантия дедупликации.
- `instrument_version` — счётчик изменений важных полей (`contract_size`, `multiplier`).
- Soft delete: `delisted_at` ставится в момент, когда инструмент перестал появляться в `exchangeInfo`. Записи не удаляются физически.
- `blacklisted` — ручная установка (по решению пользователя), даже если биржа всё ещё его отдаёт.

### 2.2 Индексы

```sql
-- быстрый lookup в normalizer'е
CREATE INDEX idx_instruments_lookup ON instruments (exchange, native_symbol)
    WHERE delisted_at IS NULL AND NOT blacklisted;

-- canonical_symbol → set of (exchange, native_symbol) для cross-exchange lookup
-- (Symbol page графики, ad-hoc analytics; consensus-сигнал sunset per `00 F40`).
CREATE INDEX idx_instruments_canonical ON instruments (canonical_symbol);
```

### 2.3 In-memory cache

Normalizer хранит активные инструменты в `dict[(exchange, native_symbol), Instrument]` для O(1) lookup'а без БД:
- TTL: 5 минут.
- Refresh: при каждом sync_job'е заново строится из БД.
- Cache miss → fallback на БД-запрос + warning `instrument_cache_miss`.

---

## 3. Sync lifecycle

### 3.1 Sync interval

**Default: каждые 5 минут.** (см. `00_DECISIONS_LOG.md F2`).

Не каждую минуту, потому что:
- Endpoint'ы `exchangeInfo` тяжёлые (`Binance Futures` ≈ 250 KB).
- Новые листинги не появляются в пределах секунд.
- 12 тяжёлых вызовов в минуту — пустая трата rate limit.

### 3.2 Кто запускает

`oi-tracker-scheduler.service` запускает `instruments_sync_job` каждые `instruments_sync_interval_sec` (default 300).

### 3.3 Что делает sync_job

```python
async def sync_instruments_for_exchange(exchange: str) -> SyncResult:
    """
    Запрашивает биржевой exchangeInfo / instruments endpoint,
    сравнивает с текущим состоянием БД, обновляет.
    """
    # 1. Получить актуальный список с биржи
    connector = get_connector(exchange)
    fetched_instruments = await connector.sync_instruments()

    # 2. Прочитать текущее состояние из БД
    existing = await repo.get_all_instruments(exchange)

    # 3. Сравнить и обновить
    new_count = 0
    updated_count = 0
    delisted_count = 0

    for fetched in fetched_instruments:
        existing_record = existing.get(fetched.native_symbol)

        if existing_record is None:
            # Новый листинг
            await repo.insert_instrument(fetched)
            new_count += 1
            logger.info("new_instrument", exchange=exchange, symbol=fetched.native_symbol)

        elif _has_material_change(existing_record, fetched):
            # Изменился contract_size / multiplier
            new_version = existing_record.instrument_version + 1
            await repo.update_instrument(fetched, new_version)
            updated_count += 1
            logger.warn("instrument_changed",
                exchange=exchange,
                symbol=fetched.native_symbol,
                old_contract_size=existing_record.contract_size,
                new_contract_size=fetched.contract_size,
                new_version=new_version)

        else:
            # Просто bump last_seen_at
            await repo.touch_instrument(exchange, fetched.native_symbol)

    # 4. Mark as delisted what's missing in fetched
    fetched_symbols = {f.native_symbol for f in fetched_instruments}
    for existing_symbol, existing_record in existing.items():
        if existing_record.delisted_at is None and existing_symbol not in fetched_symbols:
            await repo.mark_delisted(exchange, existing_symbol)
            delisted_count += 1
            logger.info("delisted", exchange=exchange, symbol=existing_symbol)

    return SyncResult(
        exchange=exchange,
        new=new_count,
        updated=updated_count,
        delisted=delisted_count,
    )
```

### 3.4 Что считать "material change"

```python
def _has_material_change(existing: Instrument, fetched: Instrument) -> bool:
    return (
        existing.contract_size != fetched.contract_size
        or existing.multiplier != fetched.multiplier
        or existing.base_asset != fetched.base_asset
        or existing.is_usdt_m != fetched.is_usdt_m
    )
```

Изменения `last_seen_at`, `raw_metadata` без изменения вышеперечисленного — не считаются material и `instrument_version` не bump'ается.

### 3.5 Параллельный sync

Все 12 sync_job'ов для разных бирж запускаются параллельно через `asyncio.gather`. Если одна биржа недоступна — остальные не блокируются.

---

## 4. Versioning (`instrument_version`)

### 4.1 Зачем

Если у XT для контракта `BTC_USDT` `contract_size` сменился с `0.001` на `0.01` (биржа изменила спецификацию), все наши прошлые точки имеют **неверный** `oi_coins` (был посчитан как `oi_contracts × 0.001`). При replay с новой версией нужно понимать, что `0.01` нужно применять только к точкам, собранным **после** изменения.

### 4.2 Как используется

В `oi_samples` хранится `instrument_version`. При normalizer переборе сырья (replay), он:
1. Читает `raw_event` из `raw_exchange_events`.
2. Берёт `Instrument` из registry по версии, которая была активна в момент `fetched_at`.
3. Использует тот `contract_size`, который был тогда.

Для этого нужна **history таблица**:

```sql
CREATE TABLE instrument_versions (
    instrument_id   INT NOT NULL REFERENCES instruments(id),
    version         INT NOT NULL,
    valid_from      TIMESTAMPTZ NOT NULL,
    valid_to        TIMESTAMPTZ,             -- NULL = текущая
    contract_size   NUMERIC(38, 18) NOT NULL,
    multiplier      NUMERIC(38, 18) NOT NULL,
    raw_metadata    JSONB NOT NULL,
    PRIMARY KEY (instrument_id, version)
);

CREATE INDEX idx_instrument_versions_lookup
    ON instrument_versions (instrument_id, valid_from DESC);
```

При material change в sync_job:
1. Текущая запись в `instrument_versions` обновляется (`valid_to = now()`).
2. Создаётся новая запись с `version = old + 1`, `valid_from = now()`, `valid_to = NULL`.
3. В `instruments` обновляются «текущие» поля, `instrument_version` bump'ается.

### 4.3 Lookup для replay

```sql
SELECT *
FROM instrument_versions iv
WHERE iv.instrument_id = $1
  AND iv.valid_from <= $2          -- raw_event.fetched_at
  AND (iv.valid_to IS NULL OR iv.valid_to > $2)
ORDER BY iv.valid_from DESC
LIMIT 1;
```

---

## 5. Filtering: что считается USDT-M perp

### 5.1 Per-exchange правила

Каждый коннектор реализует фильтр в `sync_instruments()`. Подробности — в `11_EXCHANGE_ADAPTERS/<exchange>.md`. Общие принципы:

| Биржа | USDT-M perp determined by |
|---|---|
| Binance | `contractType == "PERPETUAL"` AND `quoteAsset == "USDT"` |
| Bybit | `category == "linear"` AND `settleCoin == "USDT"` AND `contractType == "LinearPerpetual"` |
| OKX | `instType == "SWAP"` AND `settleCcy == "USDT"` |
| Bitget | `productType == "umcbl"` (Mix USDT) AND `symbolType == "perpetual"` |
| Gate.io | endpoint `/futures/usdt/contracts`, фильтр `type == "direct"` (linear) |
| MEXC | `quoteCoin == "USDT"` AND `state == "ENABLED"` AND `contractType == 1` (perpetual) |
| KuCoin | endpoint `/api/v1/contracts/active`, фильтр `quoteCurrency == "USDT"` AND `type == "FFWCSX"` (perpetual) |
| HTX | `business_type == "linear-swap"` AND `contract_type == "swap"` AND `pair == "*-USDT"` |
| Hyperliquid | по universe, фильтр `marginToken == "USDC"` (Hyperliquid использует USDC под капотом, но котировка в USDT — see exchange-specific notes) |
| Aster | mirror Binance: `contractType == "PERPETUAL"` AND `quoteAsset == "USDT"` |
| Bitmart | `product_type == 1` AND `quote_currency == "USDT"` AND `expire_timestamp == 0` AND `status == "Trading"` |
| XT | `productType == "PERPETUAL"` AND `underlyingType == 1` (USDT-M) |

### 5.2 Если фильтр пропустил не USDT-M

→ нормализатор должен это поймать на стадии `instrument_resolve`:
- `is_usdt_m == false` → запись идёт в `normalization_errors` со статусом `instrument_unresolved`.
- Точка не попадает в `oi_samples`.

Это страховка: даже если фильтр sync_job'а ошибся, normalizer не пропустит грязные данные дальше.

---

## 6. Whitelist / Blacklist

### 6.1 Exchange whitelist

`settings.exchange_whitelist` (list[str]) — какие биржи опрашиваем. Пустой = все 12.

При **отключении** биржи (удаление из whitelist):
- sync_job для этой биржи перестаёт запускаться.
- Существующие `instruments` остаются в БД, но не обновляются.
- Через 24 часа они автоматически помечаются `delisted_at = now()` (sync_job `mark_stale_as_delisted`).
- История `oi_samples` сохраняется для UI, но новые точки не приходят.

При **возврате** биржи в whitelist:
- sync_job возобновляется.
- Если инструменты ещё актуальны — `delisted_at` сбрасывается.

### 6.2 Symbol blacklist

`settings.symbol_blacklist` (list[str]) — символы, исключаемые **глобально** (для всех бирж).

Применяется на двух уровнях:
1. На уровне sync: `blacklisted = true` ставится в `instruments`.
2. На уровне scheduler: список blacklisted symbols загружается в коннектор, fetch_snapshot пропускает их.

Используется для:
- Стейблкоинов (если кто-то их торгует как perp): `["USDC", "DAI"]`.
- Известных проблемных токенов с искажёнными OI.
- Символов, которые пользователь не хочет видеть в UI / алертах.

### 6.3 Symbol whitelist (опциональный)

`settings.symbol_whitelist` (list[str]) — если задан, опрашиваем **только** эти символы.

Use case: «следить только за топ-50 по объёму». При непустом whitelist остальные символы:
- В `instruments` существуют, но не опрашиваются.
- В UI показываются как «inactive».

---

## 7. Public API (для других слоёв)

### 7.1 InstrumentRepository

```python
class InstrumentRepository(Protocol):
    async def get(self, exchange: str, native_symbol: str) -> Instrument | None:
        """Latest active instrument."""

    async def get_at_version(
        self, exchange: str, native_symbol: str, version: int
    ) -> Instrument | None:
        """Specific historical version (for replay)."""

    async def get_at_time(
        self, exchange: str, native_symbol: str, at: datetime
    ) -> Instrument | None:
        """Version active at given moment (for replay by raw_event.fetched_at)."""

    async def get_active_for_exchange(self, exchange: str) -> list[Instrument]:
        """All active instruments of one exchange (for sync diff and scheduler)."""

    async def get_by_canonical(self, canonical_symbol: str) -> list[Instrument]:
        """Cross-exchange lookup (Symbol page; consensus signal sunset per `00 F40`)."""

    async def upsert(self, inst: Instrument) -> Instrument: ...
    async def mark_delisted(self, exchange: str, native_symbol: str) -> None: ...
    async def set_blacklisted(self, exchange: str, native_symbol: str, value: bool) -> None: ...
```

### 7.2 InstrumentResolver

Используется normalizer'ом:

```python
class InstrumentResolver:
    """
    Resolve native_symbol → canonical_symbol + market metadata.
    Uses local cache; falls back to repo on miss.
    """

    async def resolve(
        self, exchange: str, native_symbol: str, at_time: datetime | None = None
    ) -> Instrument:
        """
        Returns Instrument or raises InstrumentNotFoundError.
        If at_time is None, returns current version.
        Otherwise, returns version active at at_time (for replay).
        """
        ...
```

---

## 8. Edge cases

### 8.1 Биржа возвращает дубликаты

Некоторые биржи (XT, Bitget) могут возвращать один и тот же символ с разными `productType`. Sync_job фильтрует только `PERPETUAL` USDT-M. Если после фильтра дубликаты остаются — последний выигрывает (бывает редко, фиксируется в WARN-логе).

### 8.2 Биржа меняет capitalization

`btc_usdt` → `BTC_USDT`. Считается **тем же** инструментом если `lower(native_symbol)` совпадает; sync_job обновляет `native_symbol` без bump versionа.

### 8.3 Делистинг и повторный листинг

Если символ был `delisted_at = 2026-01-15`, а потом снова появился `2026-04-01`:
- `delisted_at = NULL`, `last_seen_at = now()`.
- `instrument_version` bump НЕ происходит (если parameters не изменились).
- В UI отрисовывается gap в графике (90 дней истории, нет точек в делистинговый период).

### 8.4 Биржа изменила native_symbol для того же ассета

Например, биржа сделала ребрендинг `LUNA → LUNAC`. Это **новый** инструмент с точки зрения registry. Старый `LUNA` помечается `delisted_at`, новый `LUNAC` создаётся.

В UI это два разных ряда. Если пользователь хочет видеть единый поток — нужен ручной mapping (через `settings.symbol_aliases` JSON, future work).

### 8.5 Биржа не отдаёт contract_size

Если в payload биржи нет contract_size, normalizer:
- Использует default `1` (для linear contracts это норма).
- Помечает `Instrument.raw_metadata.contract_size_assumed = true`.
- При первой ошибке корректности (например, OI явно неправдоподобен) — пользователь должен ручно поправить.

### 8.6 Sync_job частично сломал данные

Если `sync_job` вернул `error` посреди обработки (биржа half-respond, network split):
- Транзакция rollback'ается.
- Логируется ошибка.
- Следующий sync через 5 минут попробует снова.
- Существующие `Instrument` не теряются.

---

## 9. Observability

### 9.1 Метрики

| Метрика | Тип | Labels | Описание |
|---|---|---|---|
| `oi_instruments_total` | gauge | exchange, status | Количество инструментов: status ∈ {active, delisted, blacklisted} |
| `oi_instruments_sync_duration_seconds` | histogram | exchange | Время выполнения sync_job |
| `oi_instruments_new_total` | counter | exchange | Сколько новых листингов |
| `oi_instruments_delisted_total` | counter | exchange | Сколько делистингов |
| `oi_instruments_changed_total` | counter | exchange, change_type | Сколько material changes (change_type ∈ {contract_size, multiplier, base_asset}) |
| `oi_instruments_sync_errors_total` | counter | exchange, error_type | Ошибки sync |

### 9.2 Алерты

- `instruments_sync_failing`: 3 sync'а подряд провалились → `exchange-health` алерт.
- `instrument_changed`: при material change → INFO лог + Telegram уведомление (если включено).
- `instrument_resolution_high_miss_rate`: если cache miss > 1% → WARNING.

---

## 10. UI

В `Settings` UI:
- Список бирж в whitelist (toggle).
- Поле `symbol_blacklist` (textarea, comma-separated).
- Поле `symbol_whitelist` (textarea; если пустой — все).
- Список «делистнутых за последние 7 дней» (read-only, для информирования).
- Список «новых листингов за последние 7 дней» (read-only).
- Кнопка `Force re-sync` (запустить sync_job для одной биржи немедленно).

В `Symbol` page:
- Метаданные инструмента: `contract_size`, `instrument_version`, `first_seen_at`, статусы по биржам.

---

## 11. Cross-references

- Schema → `05_DATA_CONTRACTS.md §11`.
- Per-exchange filtering rules → `11_EXCHANGE_ADAPTERS/<exchange>.md` (раздел "Filter").
- Replay использует `instrument_versions` → `08_TIME_SERIES_STORAGE.md §6`.
- Instrument resolution в normalizer'е → `07_NORMALIZER.md §3`.
