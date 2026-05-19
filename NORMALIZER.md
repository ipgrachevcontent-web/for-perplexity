# Что именно должен делать normalizer по биржам

## Binance
Для openInterest normalizer берёт symbol, openInterest, time и через instrument registry определяет тип инструмента. Если это snapshot endpoint, то oi_native хранится как есть, а oi_notional_usdt считается через текущую цену; если это Open Interest Statistics, то можно взять sumOpenInterestValue как готовую notional-метрику и сохранить её как primary normalized value.

## Bybit
Normalizer знает правило: для linear OI unit может быть base asset, например BTC для BTCUSDT. Поэтому он не должен считать число “уже USDT”, а обязан прикрепить цену и перевести в notional USDT.

## XT
Normalizer читает open_interest, но трактует его как contracts, потому что docs так и описывают это поле, и использует contractSize для перевода. Дополнительно он должен учитывать product_type=PERPETUAL и underlyingType, чтобы не пустить в канонический слой не тот тип рынка.

# Структура внутри кода

Нормальный дизайн normalizer такой:
```python
class BaseNormalizer:
    def normalize(event: RawExchangeEvent) -> NormalizedOISampleResult: ...

class BinanceNormalizer(BaseNormalizer): ...
class BybitNormalizer(BaseNormalizer): ...
class XTNormalizer(BaseNormalizer): ...

class InstrumentResolver: ...
class UnitResolver: ...
class PriceResolver: ...
class OIConverter: ...
class QualityAssessor: ...
```

### Поток данных:
1. RawExchangeEvent
2. Exchange parser
3. InstrumentResolver
4. UnitResolver
5. PriceResolver
6. OIConverter
7. QualityAssessor
8. NormalizedOISample

*То есть “normalizer” — это не одна функция, а пайплайн из нескольких детерминированных стадий.*

### Что он пишет в БД
Я бы разделил на три таблицы:

1. raw_exchange_events
*Храним как есть:*

- exchange
- endpoint
- payload_json
- fetched_at
- http_status
- request_id
- raw_hash

2. normalized_oi_samples
*Храним канонику:*

- exchange
- native_symbol
- canonical_symbol
- market_type
- ts_exchange
- ts_ingested
- oi_native
- oi_native_unit
- oi_notional_usdt
- price_used
- source_kind
- normalization_version
- status

3. normalization_errors
*Храним неудачи:*

- exchange
- native_symbol
- error_type
- error_message
- raw_hash
- created_at

***Это важно, чтобы новые или сломанные символы не исчезали бесследно.***

# Важные правила работы

## Rule 1: Normalizer должен быть идемпотентным
Один и тот же raw event при повторной обработке должен давать тот же normalized result той же версии правил. Для этого нужен raw_hash и normalization_version.

---

## Rule 2: Никогда не теряй сырой payload
Даже если символ не распознан или неясна единица измерения, raw event всё равно сохраняется. Иначе ты не сможешь потом переиграть нормализацию, когда обновишь правила.

---

## Rule 3: Нормализация должна быть versioned
Формулы пересчёта могут меняться. Например, сначала ты считал XT по одной формуле contracts → notional, потом уточнил спецификацию контракта и изменил расчёт. Значит, нужно хранить normalization_version, чтобы уметь делать backfill/replay.

---

## Rule 4: Статус важнее “молчаливого успеха”
Каждый normalized record должен иметь status:
- ok
- ok_with_warnings
- partial
- failed

*Например, если price resolver взял last_price вместо mark_price, это уже warning.*

---

# Как normalizer связан с окнами 5/15/30
Normalizer не должен сам считать rolling windows. Его работа — дать корректную каноническую точку во времени. Уже storage/metrics layer считает:
- последнюю валидную точку;
- точку 5 минут назад;
- точку 15 минут назад;
- точку 30 минут назад;
- проценты и абсолюты.

**Исключение** — если биржа сама отдаёт интервальные OI-бары, как Bybit и KuCoin. Тогда normalizer лишь помечает source_kind=exchange_interval, а metrics layer решает, использовать ли их как первичный источник для окон или как ещё один тип time-series.

# Самая правильная ментальная модель
Думай о normalizer как о компиляторе биржевых форматов в твой внутренний OI-IR — internal representation. Биржи говорят на разных “языках”: Binance даёт snapshot с time, Bybit даёт interval records и unit в base asset, XT даёт contracts + contractSize; normalizer компилирует всё это в один внутренний формат, пригодный для хранения, расчётов и алертов.
