# 07 · Normalizer

> 5-стадийный pipeline превращения сырых биржевых ответов в каноничные `NormalizedOISample`. Детерминированный, идемпотентный, версионируемый.
> Связано: `00_DECISIONS_LOG.md C5, F10, F11`, `05_DATA_CONTRACTS.md §3, §4, §14`, `06_INSTRUMENT_REGISTRY.md`.

---

## 1. Назначение

Normalizer — компилятор биржевых форматов в наш внутренний OI-IR (Internal Representation).

Биржи говорят на разных «языках»:
- Binance даёт snapshot с `time + symbol + openInterest`.
- Bybit даёт interval records, с `unit = base_asset` (BTC, не USDT).
- XT даёт `open_interest` в контрактах + `contractSize` отдельно.
- OKX даёт OI в контрактах + `ctVal` + `settleCcy`.

Normalizer выдаёт всегда одно и то же:
```python
NormalizedOISample(
    canonical_symbol="BTC",
    oi_coins=Decimal("123.456"),
    oi_notional_usdt=Decimal("9326412.99"),
    valuation_status="good_estimate",
    source_kind="snapshot",
    ...
)
```

---

## 2. Pipeline

```
RawExchangeEvent (from connector)
        │
        ▼
┌──────────────────┐
│ 1. Exchange     │  Распаковать payload по правилам конкретной биржи
│    Parser        │  → ParsedOI (промежуточный объект)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 2. Instrument    │  ParsedOI.native_symbol → Instrument
│    Resolver      │  Доступ к registry, проверка active/delisted
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 3. Unit          │  Определить, в чём измерен OI
│    Resolver      │  (coins | contracts | quote | already-USDT-notional)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 4. Price         │  Если нужна конверсия — взять mark/index/last
│    Attachment    │  и зафиксировать price_source
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 5. OI Converter  │  Применить формулу, получить
│    + Quality     │  oi_coins + oi_notional_usdt + valuation_status
│    Assessor      │
└────────┬─────────┘
         │
         ▼
NormalizedOISample → write to oi_samples
       OR
NormalizationError → write to normalization_errors
                    (raw_event сохраняется в любом случае)
```

---

## 3. Stage 1: Exchange Parser

### 3.1 Назначение

Распаковать `raw_payload` в `ParsedOI` — внутренний промежуточный объект. Парсер знает форму ответа конкретной биржи и какие поля где лежат.

### 3.2 ParsedOI schema

```python
@dataclass(frozen=True)
class ParsedOI:
    native_symbol: str
    ts_exchange: datetime                  # из payload или fetched_at fallback
    oi_raw: Decimal                        # значение, как пришло
    oi_unit_hint: str                      # "base_asset" | "contracts" | "quote_asset" | "usdt_notional"
    oi_notional_provided: Optional[Decimal]  # если биржа сама дала USDT-notional (Binance statistics)
    price_provided: Optional[Decimal]      # если биржа вернула цену в payload OI
    raw_extras: dict[str, Any]             # дополнительные поля (для аудита)
```

### 3.3 Per-exchange парсеры

Подробности — в `11_EXCHANGE_ADAPTERS/<exchange>.md`. Кратко:

| Биржа | Endpoint | Поля payload → ParsedOI |
|---|---|---|
| Binance | `/fapi/v1/openInterest` | `symbol → native_symbol`, `openInterest → oi_raw`, `time → ts_exchange`, `oi_unit_hint = "base_asset"` |
| Binance (statistics) | `/futures/data/openInterestHist` | `sumOpenInterest → oi_raw`, `sumOpenInterestValue → oi_notional_provided`, `oi_unit_hint = "base_asset"` |
| Bybit | `/v5/market/open-interest` | `symbol → native_symbol`, `openInterest → oi_raw`, `timestamp → ts_exchange`, `oi_unit_hint = "base_asset"` |
| OKX | `/api/v5/public/open-interest` | `instId → native_symbol`, `oi → oi_raw` (в контрактах), `oiCcy → используется для валидации`, `ts → ts_exchange`, `oi_unit_hint = "contracts"` |
| Bitget | `/api/mix/v1/market/open-interest` | `symbol → native_symbol`, `amount → oi_raw`, `oi_unit_hint = "base_asset"` (для USDT-perp = base) |
| Gate.io | `/futures/usdt/contracts` (per contract) | `name → native_symbol`, `total_size → oi_raw` (в контрактах), `quanto_multiplier → нужен для конверсии`, `oi_unit_hint = "contracts"` |
| MEXC | `/api/v1/contract/open_interest` | `symbol → native_symbol`, `holdVol → oi_raw`, `oi_unit_hint = "contracts"` |
| KuCoin | `/api/ua/v1/market/open-interest` | `symbol → native_symbol`, `value → oi_raw`, `oi_unit_hint = "base_asset"` |
| HTX | `/linear-swap-api/v1/swap_open_interest` | `contract_code → native_symbol`, `volume → oi_raw` (контракты), `amount → oi_raw_alt` (base), `oi_unit_hint = "base_asset"` если используем `amount`, иначе `"contracts"` |
| Hyperliquid | `info/perpetuals` | `name → native_symbol`, `openInterest → oi_raw`, `oi_unit_hint = "base_asset"` |
| Aster | `/fapi/v1/openInterest` | (binance-like): `symbol`, `openInterest`, `time`, `oi_unit_hint = "base_asset"` |
| Bitmart | `/contract/public/details` (bulk) | `symbol → native_symbol`, `open_interest → oi_raw` (контракты), `contract_size → instrument.contract_size`, `open_interest_value → oi_notional_provided`, `oi_unit_hint = "contracts"` |
| XT | `/future/market/v1/public/cg/contracts` | `symbol → native_symbol`, `open_interest → oi_raw` (в контрактах), `contractSize → нужен для конверсии`, `oi_unit_hint = "contracts"` |

### 3.4 Errors на этой стадии

- `parse_error` если payload не парсится (поля отсутствуют, тип не тот).
- `schema_drift` если ключевые поля имеют неожиданное значение (например, `quoteAsset` вдруг не USDT).

В обоих случаях `NormalizationError` пишется в `normalization_errors`, исходный `raw_event` сохранён.

---

## 4. Stage 2: Instrument Resolver

### 4.1 Что делает

```python
async def resolve_instrument(
    parsed: ParsedOI,
    exchange: str,
    at_time: datetime,                     # обычно raw_event.fetched_at
) -> Instrument:
    inst = await registry.get_at_time(exchange, parsed.native_symbol, at_time)
    if inst is None:
        raise InstrumentUnresolvedError(exchange, parsed.native_symbol)
    if inst.delisted_at is not None and inst.delisted_at <= at_time:
        raise InstrumentDelistedError(exchange, parsed.native_symbol, inst.delisted_at)
    if inst.blacklisted:
        raise InstrumentBlacklistedError(exchange, parsed.native_symbol)
    if not inst.is_usdt_m:
        raise InstrumentWrongMarketError(exchange, parsed.native_symbol)
    return inst
```

### 4.2 Что получаем

- `canonical_symbol` (`"BTC"`).
- `base_asset`, `quote_asset`, `settle_asset`, `market_type`.
- `contract_size`, `multiplier`, `is_usdt_m`.
- `instrument_version` (для `oi_samples.instrument_version`).

### 4.3 Errors

- `instrument_unresolved` — биржа вернула символ, которого нет в registry. Чаще всего: новый листинг ещё не подхвачен sync_job'ом. Нормализация откладывается на следующий цикл (точка пишется в `normalization_errors`, **не** в `oi_samples`).
- `instrument_delisted` — символ был делистнут до момента fetch. Пишется в errors, не в samples.
- `instrument_blacklisted` — пользователь явно исключил. Тихо игнорируется (не `error`, а `skip`).
- `instrument_wrong_market` — `is_usdt_m == false`. Это означает, что фильтр sync_job ошибся; в errors с `failed_stage = "instrument_resolve"`.

---

## 5. Stage 3: Unit Resolver

### 5.1 Назначение

Понять, в чём именно выражен `oi_raw`, и привести к каноническому unit'у — `oi_coins` (количество монет базового актива).

### 5.2 Дерево решений

```python
def resolve_unit(parsed: ParsedOI, inst: Instrument) -> ResolvedUnit:
    hint = parsed.oi_unit_hint

    if hint == "base_asset":
        # уже в монетах
        return ResolvedUnit(
            oi_coins=parsed.oi_raw,
            method="exchange_native_base",
        )

    if hint == "contracts":
        # OI = oi_raw × contract_size (linear) [× multiplier]
        oi_coins = parsed.oi_raw * inst.contract_size * inst.multiplier
        return ResolvedUnit(
            oi_coins=oi_coins,
            method="contracts_to_base",
            details={
                "contract_size": inst.contract_size,
                "multiplier": inst.multiplier,
            },
        )

    if hint == "quote_asset":
        # OI = oi_raw / price = монеты
        # цена прикрепляется на следующей стадии; здесь возвращаем
        # placeholder с нужным методом
        return ResolvedUnit(
            oi_coins=None,                # будет вычислено после Price Attachment
            method="quote_to_base_via_price",
        )

    if hint == "usdt_notional":
        # OI уже в USDT, монеты считаем обратно через цену
        return ResolvedUnit(
            oi_coins=None,
            method="usdt_notional_to_base_via_price",
            oi_notional_usdt_directly=parsed.oi_raw,
        )

    raise UnitUnresolvedError(f"Unknown oi_unit_hint: {hint}")
```

### 5.3 Per-exchange правила (примеры)

| Биржа | hint | Resolution |
|---|---|---|
| Binance current OI | `base_asset` | `oi_coins = oi_raw` |
| Binance statistics | `base_asset` | `oi_coins = oi_raw`; `sumOpenInterestValue` — отдельно как authoritative notional |
| Bybit linear | `base_asset` | `oi_coins = oi_raw` |
| OKX SWAP | `contracts` | `oi_coins = oi_raw × ctVal` (где `ctVal = contract_size`) |
| Gate.io | `contracts` | `oi_coins = oi_raw × quanto_multiplier` |
| HTX | `base_asset` (поле `amount`) или `contracts` (поле `volume`) — выбираем `amount` | `oi_coins = amount` |
| XT | `contracts` | `oi_coins = open_interest × contractSize` |

### 5.4 Errors

- `unit_unresolved` — `oi_unit_hint` не из перечисленных. Это указывает на schema drift.

---

## 6. Stage 4: Price Attachment

### 6.1 Когда нужна цена

| Случай | Нужна цена? |
|---|---|
| Биржа сама вернула notional (Binance `sumOpenInterestValue`) | ❌ нет |
| OI в base_asset, нужен notional USDT | ✅ да, mark > index > last |
| OI в contracts с linear contract_size, оставшийся unit = monetary | ✅ да, для notional |
| OI в quote_asset (USDT) — нужен price для обратной конверсии в монеты | ✅ да |

### 6.2 Источники цены

Каждый коннектор поддерживает price cache: TTL 30 секунд, обновляется по запросу или вместе с OI fetch.

```python
class PriceCache:
    """Per-exchange in-memory cache mark/index/last prices."""

    async def get_price(
        self,
        exchange: str,
        native_symbol: str,
        at_time: datetime,
    ) -> PriceSnapshot:
        cached = self._cache.get((exchange, native_symbol))
        if cached and (at_time - cached.fetched_at).total_seconds() <= 30:
            return cached
        # fetch fresh
        snapshot = await self._fetch(exchange, native_symbol)
        self._cache[(exchange, native_symbol)] = snapshot
        return snapshot
```

### 6.3 Приоритет источников

```python
def select_price(snap: PriceSnapshot) -> tuple[Decimal, PriceSource]:
    if snap.mark_price is not None:
        return (snap.mark_price, PriceSource.MARK)
    if snap.index_price is not None:
        return (snap.index_price, PriceSource.INDEX)
    if snap.last_price is not None:
        return (snap.last_price, PriceSource.LAST)
    raise PriceUnavailableError(...)
```

### 6.4 Per-exchange price endpoints

| Биржа | Endpoint | mark | index | last |
|---|---|---|---|---|
| Binance | `/fapi/v1/premiumIndex` | ✅ markPrice | ✅ indexPrice | (другой endpoint) |
| Bybit | `/v5/market/tickers` | ✅ markPrice | ✅ indexPrice | ✅ lastPrice |
| OKX | `/api/v5/market/index-tickers` | (markPrice через mark-price endpoint) | ✅ idxPx | (отдельно) |
| Bitget | `/api/mix/v1/market/mark-price` | ✅ | ✅ | ✅ |
| Gate.io | `/futures/usdt/tickers` | ✅ mark_price | — | ✅ last |
| MEXC | `/api/v1/contract/index_price` | ✅ | ✅ | ✅ |
| KuCoin | `/api/v1/mark-price/{symbol}/current` | ✅ value | ✅ indexPrice | (другой endpoint) |
| HTX | `/linear-swap-api/v1/swap_mark_price` | ✅ | (отдельно) | (ticker endpoint) |
| Hyperliquid | `info/perpetuals` (включает mark price) | ✅ | (oraclePx) | — |
| Aster | как Binance | ✅ | ✅ | — |
| Bitmart | `/contract/public/details` (bulk) | — | ✅ index_price | ✅ last_price |
| XT | `/future/market/v1/public/q/symbol-mark-price` | ✅ | (отдельно) | ✅ |

### 6.5 Errors

- `price_unavailable` — ни mark, ни index, ни last недоступны. Точка не нормализуется в `oi_samples`, идёт в `normalization_errors`. Это редкий случай; обычно last-price есть.

### 6.6 Влияние на `valuation_status`

| Использован price | valuation_status |
|---|---|
| Биржа сама дала notional | `authoritative` |
| `mark_price` | `good_estimate` |
| `index_price` | `good_estimate` |
| `last_price` | `low_confidence` |
| `last_price` И bid-ask spread > 1% | `low_confidence` + warning `wide_spread` |

---

## 7. Stage 5: OI Converter + Quality Assessor

### 7.1 Финальная конверсия

```python
def convert(
    parsed: ParsedOI,
    inst: Instrument,
    unit_resolution: ResolvedUnit,
    price_attachment: PriceAttachment,
) -> tuple[Decimal, Decimal]:
    """
    Returns (oi_coins, oi_notional_usdt).
    """
    if unit_resolution.method == "exchange_native_base":
        oi_coins = unit_resolution.oi_coins
        oi_notional_usdt = oi_coins * price_attachment.price

    elif unit_resolution.method == "contracts_to_base":
        oi_coins = unit_resolution.oi_coins
        oi_notional_usdt = oi_coins * price_attachment.price

    elif unit_resolution.method == "quote_to_base_via_price":
        oi_notional_usdt = parsed.oi_raw     # уже в USDT
        oi_coins = oi_notional_usdt / price_attachment.price

    elif unit_resolution.method == "usdt_notional_to_base_via_price":
        oi_notional_usdt = unit_resolution.oi_notional_usdt_directly
        oi_coins = oi_notional_usdt / price_attachment.price

    elif parsed.oi_notional_provided is not None:
        # биржа сама дала notional → authoritative
        oi_notional_usdt = parsed.oi_notional_provided
        oi_coins = unit_resolution.oi_coins or (oi_notional_usdt / price_attachment.price)

    else:
        raise ConversionFailedError(...)

    return oi_coins, oi_notional_usdt
```

### 7.2 Quality Assessor

После конверсии проверяем:

```python
def assess_quality(
    parsed: ParsedOI,
    inst: Instrument,
    unit_resolution: ResolvedUnit,
    price_attachment: PriceAttachment,
    oi_coins: Decimal,
    oi_notional_usdt: Decimal,
    raw_event: RawExchangeEvent,
) -> QualityMeta:
    warnings = []

    # 1. Valuation status
    if parsed.oi_notional_provided is not None:
        valuation_status = ValuationStatus.AUTHORITATIVE
    elif price_attachment.price_source in (PriceSource.MARK, PriceSource.INDEX):
        valuation_status = ValuationStatus.GOOD_ESTIMATE
    else:
        valuation_status = ValuationStatus.LOW_CONFIDENCE

    # 2. Clock skew
    skew_seconds = (raw_event.fetched_at - parsed.ts_exchange).total_seconds()
    if skew_seconds > 30:
        warnings.append("clock_skew_backward")
        valuation_status = min(valuation_status, ValuationStatus.GOOD_ESTIMATE)
    if skew_seconds < -30:
        warnings.append("clock_skew_forward")

    # 3. Sanity check
    if oi_coins <= 0:
        warnings.append("non_positive_oi")
        valuation_status = ValuationStatus.LOW_CONFIDENCE

    if oi_notional_usdt < 1000:
        warnings.append("very_low_notional")

    # 4. Contract size assumed
    if inst.raw_metadata.get("contract_size_assumed"):
        warnings.append("contract_size_assumed")
        valuation_status = min(valuation_status, ValuationStatus.GOOD_ESTIMATE)

    return QualityMeta(
        valuation_status=valuation_status,
        warnings=warnings,
        unit_resolution_method=unit_resolution.method,
        price_resolution_method=price_attachment.price_source,
    )
```

`min(ValuationStatus.X, Y)` — приоритет: `LOW_CONFIDENCE < GOOD_ESTIMATE < AUTHORITATIVE`.

---

## 8. Полный pipeline в коде

```python
class Normalizer:
    def __init__(
        self,
        registry: InstrumentRepository,
        price_cache: PriceCache,
        normalization_version: int,
    ):
        self._registry = registry
        self._price_cache = price_cache
        self._version = normalization_version
        self._parsers = {
            "binance": BinanceParser(),
            "bybit": BybitParser(),
            ...
        }

    async def normalize(self, raw_event: RawExchangeEvent) -> NormalizationResult:
        try:
            # Stage 1
            parser = self._parsers[raw_event.exchange]
            parsed = parser.parse(raw_event.response_payload, raw_event.endpoint)

            # Stage 2
            inst = await self._registry.get_at_time(
                raw_event.exchange, parsed.native_symbol, raw_event.fetched_at
            )
            if inst is None or inst.delisted_at:
                return NormalizationResult.error(
                    raw_event, NormalizationErrorType.INSTRUMENT_UNRESOLVED, "instrument_resolve"
                )

            # Stage 3
            unit_res = self._resolve_unit(parsed, inst)

            # Stage 4
            price_att = await self._attach_price(parsed, inst, raw_event)

            # Stage 5
            oi_coins, oi_notional_usdt = self._convert(parsed, inst, unit_res, price_att)
            quality = self._assess_quality(parsed, inst, unit_res, price_att, oi_coins, oi_notional_usdt, raw_event)

            sample = NormalizedOISample(
                exchange=raw_event.exchange,
                canonical_symbol=inst.canonical_symbol,
                native_symbol=parsed.native_symbol,
                quote_asset=inst.quote_asset,
                settle_asset=inst.settle_asset,
                market_type=inst.market_type,
                contract_type=inst.contract_type,
                ts_exchange=parsed.ts_exchange,
                ts_ingested=raw_event.fetched_at,
                oi_coins=oi_coins,
                oi_native_unit=parsed.oi_unit_hint,
                oi_notional_usdt=oi_notional_usdt,
                price_used=price_att.price,
                price_source=price_att.price_source,
                source_kind=raw_event.source_kind_intended,
                valuation_status=quality.valuation_status,
                normalization_version=self._version,
                raw_event_id=raw_event.id,
                raw_hash=raw_event.raw_hash,
                instrument_version=inst.instrument_version,
                warnings=quality.warnings,
            )
            return NormalizationResult.ok(sample)

        except ParseError as e:
            return NormalizationResult.error(raw_event, NormalizationErrorType.PARSE_ERROR, "parse", str(e))
        except UnitUnresolvedError as e:
            return NormalizationResult.error(raw_event, NormalizationErrorType.UNIT_UNRESOLVED, "unit_resolve", str(e))
        except PriceUnavailableError as e:
            return NormalizationResult.error(raw_event, NormalizationErrorType.PRICE_UNAVAILABLE, "price_attach", str(e))
        except ConversionFailedError as e:
            return NormalizationResult.error(raw_event, NormalizationErrorType.CONVERSION_FAILED, "convert", str(e))
```

---

## 9. Идемпотентность и replay

### 9.1 Идемпотентность

```python
def is_idempotent(raw_event: RawExchangeEvent, version: int) -> bool:
    """
    Один и тот же raw_event при той же normalization_version
    должен дать тот же NormalizedOISample.
    """
```

Гарантии:
- Все стадии — pure функции от `(raw_event, instrument_at_time, price_at_time, version)`.
- Нет глобального состояния, влияющего на результат.
- Нет Random / time.now() внутри pipeline (`now()` только в `ts_ingested`, `ts_processed` — не во входе).

### 9.2 Replay

При bump `normalization_version`:
1. CLI команда `replay --since=2026-04-25 --until=2026-04-30 --version=4`.
2. Читает `raw_exchange_events` в указанном диапазоне.
3. Прогоняет через `Normalizer` с новой версией.
4. INSERT в `oi_samples` с `normalization_version = 4`.
5. UNIQUE constraint `(exchange, canonical_symbol, ts_exchange)` будет конфликтовать со старыми строками.

Стратегия:
- **Default:** UPDATE (заменяем старые строки новой версией). Это значит, что в БД хранится только последняя версия.
- **Опционально:** хранить N версий параллельно (через `(exchange, canonical_symbol, ts_exchange, normalization_version)` как PK). Это нужно, если хотим A/B сравнение алгоритмов. В v1 не делаем.

### 9.3 Continuous aggregates после replay

После UPDATE строк в `oi_samples`:
```sql
CALL refresh_continuous_aggregate('oi_5m', '2026-04-25 00:00+00', '2026-04-30 23:59+00');
CALL refresh_continuous_aggregate('oi_15m', ...);
CALL refresh_continuous_aggregate('oi_30m', ...);
```

### 9.4 Alert engine после replay

См. `00_DECISIONS_LOG.md F9`: **no retro-fire**. После replay alert engine не пересматривает прошлые алерты для отправки в TG; опционально может пометить `alerts_sent` записи как `historically_inaccurate`.

---

## 10. Edge cases per source kind

### 10.1 source_kind = "snapshot"

- `ts_exchange` берём из payload биржи (Binance `time`, OKX `ts`, etc.). Если поле отсутствует — fallback на `raw_event.fetched_at`, добавляем warning `ts_exchange_inferred`.
- Каждый snapshot = одна строка в `oi_samples` с `source_kind = "snapshot"`.

### 10.2 source_kind = "native_interval"

- `ts_exchange` = `bucket_end` (см. `04_TIME_MODEL.md §9`).
- За один HTTP-запрос биржа может вернуть 1–N баров (Bybit пагинирует, KuCoin — тоже).
- Каждый бар становится одной строкой в `oi_samples`.
- ON CONFLICT нужен (если перезапрос дал тот же бар — upsert безопасен).

### 10.3 source_kind = "backfill"

- Как `snapshot` или `native_interval`, но `source_kind = "backfill"`.
- Используется когда заполняем gap после downtime коннектора.
- Точки идут в `oi_samples` нормально, но alert engine их игнорирует для FIRE.

### 10.4 source_kind = "replay"

- Перезапись существующих точек с новой `normalization_version`.
- В `oi_samples` поле `source_kind` остаётся изначальным (snapshot / native_interval), `normalization_version` обновляется.
- Это NOT равно «backfill» — backfill заполняет пропуски, replay переинтерпретирует существующие данные.

### 10.5 source_kind = "degraded_fallback"

- Нативный native_interval недоступен для Bybit/KuCoin → переключение на snapshot polling.
- Помечается `source_kind = "degraded_fallback"`.
- Alert engine применяет более строгие фильтры (например, divergence сигнал отключается, threshold повышается).

---

## 11. Testing

### 11.1 Unit-тесты

Каждый parser имеет суит unit-тестов с реальными payload-фикстурами:

```
tests/contract/binance/
├── fixtures/
│   ├── open_interest_btcusdt.json
│   ├── open_interest_btcusdt_history.json
│   └── premium_index.json
├── test_binance_parser.py
└── test_binance_normalizer_e2e.py
```

### 11.2 Property-based тесты (опционально)

Через `hypothesis`:
- Произвольный валидный payload → нормализация не падает.
- `oi_coins >= 0` всегда.
- `valuation_status` корректно деградирует при отсутствии mark/index.

### 11.3 Schema drift тесты

Для каждого parser'а — тест «что будет, если биржа уберёт поле» / «что будет, если биржа изменит тип значения». Цель: убедиться, что schema drift попадает в `normalization_errors`, а не падает с unhandled exception.

### 11.4 Idempotence test

```python
async def test_idempotence(raw_event):
    sample1 = await normalizer.normalize(raw_event)
    sample2 = await normalizer.normalize(raw_event)
    assert sample1 == sample2
```

---

## 12. Observability

### 12.1 Метрики

| Метрика | Тип | Labels |
|---|---|---|
| `oi_normalize_duration_seconds` | histogram | exchange, stage |
| `oi_normalize_total` | counter | exchange, status (ok/error) |
| `oi_normalize_errors_total` | counter | exchange, error_type, failed_stage |
| `oi_valuation_status_total` | counter | exchange, status |
| `oi_warnings_total` | counter | exchange, warning_type |
| `oi_normalization_version_active` | gauge | (без labels) |

### 12.2 Алерты

- `normalization_error_rate > 1%` за 5 минут per exchange → `exchange-health` алерт.
- `low_confidence_ratio > 10%` per exchange → WARNING.
- `instrument_unresolved_count > 0` for new symbol → информативный лог, sync_job догонит через 5 минут.

---

## 13. Cross-references

- Per-exchange parsers → `11_EXCHANGE_ADAPTERS/<exchange>.md` (раздел "Parser").
- Storage `oi_samples` → `08_TIME_SERIES_STORAGE.md`.
- Replay procedure → `08_TIME_SERIES_STORAGE.md §6`.
- Alert engine использует `valuation_status` и `source_kind` → `09_ALERT_ENGINE.md §4`.
- Instrument resolution → `06_INSTRUMENT_REGISTRY.md §7`.
