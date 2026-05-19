# 11 · Exchange Adapters — Base Connector

> Единый интерфейс для всех 12 биржевых коннекторов. Реализация — `app/exchanges/base.py`.
> Связано: `05_DATA_CONTRACTS.md §3, §11, §13`, `06_INSTRUMENT_REGISTRY.md`, `07_NORMALIZER.md`.

---

## 1. Цель

`BaseConnector` — это абстрактный класс, который реализует **общую логику**, а наследники — **per-exchange специфику**. Цели:

- Единый контракт для scheduler'а.
- Изоляция сетевого слоя от бизнес-логики.
- Стандартный watermark, retry, circuit breaker, rate limiting.
- Гарантированная запись `RawExchangeEvent` и `ConnectorErrorEvent`.

Per-exchange файлы (`binance.md`, `bybit.md`, ...) описывают **только специфику**: endpoint URLs, поля payload, особенности.

---

## 2. Базовый интерфейс

```python
from abc import ABC, abstractmethod
from datetime import datetime
from decimal import Decimal
from typing import Iterable, Optional

class BaseConnector(ABC):
    """
    Single-exchange data fetcher. One instance per exchange per process.
    """

    exchange: str                          # Класс-атрибут: "binance", "bybit", ...
    primary_source_kind: SourceKind        # SNAPSHOT | NATIVE_INTERVAL | ...
    connector_version: str                 # bump при изменении логики коннектора

    def __init__(
        self,
        http_client: httpx.AsyncClient,
        rate_limiter: RateLimiter,
        instruments_repo: InstrumentRepository,
        raw_events_repo: RawEventsRepository,
        connector_errors_repo: ConnectorErrorsRepository,
        circuit_breaker: CircuitBreaker,
        price_cache: PriceCache,
    ): ...

    # -----------------------------
    # ОБЯЗАТЕЛЬНЫЕ методы (per-exchange)
    # -----------------------------

    @abstractmethod
    async def sync_instruments(self) -> list[Instrument]:
        """
        Получить список активных USDT-M perpetual инструментов.
        Каждые 5 минут вызывается scheduler'ом.
        """

    @abstractmethod
    async def fetch_snapshot(self, instruments: list[Instrument]) -> list[RawExchangeEvent]:
        """
        Получить текущее OI для перечисленных инструментов.
        Может выполнять много HTTP-запросов параллельно (с учётом rate limit).
        Возвращает список raw events (один на инструмент или batch).
        """

    async def fetch_range(
        self,
        instrument: Instrument,
        interval: str,             # "5m" | "15m" | "30m"
        start: datetime,
        end: datetime,
    ) -> list[RawExchangeEvent]:
        """
        Опциональный метод: получить native interval бары для одного инструмента.
        Реализуют только Bybit и KuCoin (и Binance для historical statistics).
        Default raises NotImplementedError.
        """
        raise NotImplementedError(f"{self.exchange} does not support fetch_range")

    @abstractmethod
    async def fetch_prices(self, instruments: list[Instrument]) -> dict[str, PriceSnapshot]:
        """
        Получить mark / index / last цены для перечисленных инструментов.
        Используется normalizer'ом для конверсии в notional.
        Возвращает dict[native_symbol, PriceSnapshot].
        """

    # -----------------------------
    # ОБЩИЕ методы (реализованы в Base)
    # -----------------------------

    async def healthcheck(self) -> HealthCheckResult:
        """
        Light HTTP-запрос (например, `GET /ping`). Должен возвращаться < 5s.
        Используется в /api/v1/health.
        """

    async def execute_cycle(self) -> CycleResult:
        """
        Один полный цикл: fetch instruments cache → fetch prices → fetch snapshots/range.
        Вызывается scheduler'ом раз в минуту.
        """
        ...

    def get_rate_limit_policy(self) -> RateLimitPolicy: ...
    def get_circuit_breaker_state(self) -> CircuitBreakerState: ...
```

---

## 3. Лайфсайкл `execute_cycle()`

```python
async def execute_cycle(self) -> CycleResult:
    cycle_id = uuid4()
    cycle_start = monotonic()

    # 0. Circuit breaker
    if self._circuit_breaker.is_open():
        logger.warn("circuit_breaker_open", exchange=self.exchange)
        return CycleResult.skipped(reason="circuit_breaker_open")

    try:
        # 1. Получить активные инструменты (из cache, не запрос к бирже)
        instruments = await self._instruments_repo.get_active_for_exchange(self.exchange)

        if not instruments:
            return CycleResult.skipped(reason="no_active_instruments")

        # 2. Применить blacklist / whitelist
        instruments = self._apply_filters(instruments)

        # 3. Fetch prices (параллельно с OI или раньше, в зависимости от биржи)
        prices = await self.fetch_prices(instruments)
        await self._price_cache.update(self.exchange, prices)

        # 4. Fetch OI
        if self.primary_source_kind == SourceKind.NATIVE_INTERVAL:
            raw_events = await self._fetch_native_interval(instruments)
        else:
            raw_events = await self.fetch_snapshot(instruments)

        # 5. Persist raw events
        await self._raw_events_repo.bulk_insert(raw_events)

        # 6. Reset circuit breaker counter on success
        self._circuit_breaker.record_success()

        # 7. Metrics
        cycle_duration = monotonic() - cycle_start
        metrics.connector_cycle_duration_seconds.labels(exchange=self.exchange).observe(cycle_duration)
        metrics.connector_samples_collected_total.labels(exchange=self.exchange).inc(len(raw_events))

        return CycleResult.ok(samples_count=len(raw_events), duration=cycle_duration)

    except RateLimitedError as e:
        # Soft fail
        await self._handle_rate_limit(e)
        return CycleResult.failed(reason="rate_limited", retry_after=e.retry_after)

    except (httpx.TimeoutException, httpx.ConnectError, httpx.HTTPStatusError) as e:
        # Network or HTTP error
        await self._record_connector_error(e)
        self._circuit_breaker.record_failure()
        return CycleResult.failed(reason=type(e).__name__, error=str(e))

    except Exception as e:
        logger.exception("connector_unexpected_error", exchange=self.exchange)
        await self._record_connector_error(e)
        self._circuit_breaker.record_failure()
        return CycleResult.failed(reason="unexpected", error=str(e))
```

---

## 4. Rate limiting

### 4.1 Token bucket

Каждый коннектор имеет `RateLimiter` (token bucket):

```python
class RateLimiter:
    """
    Token bucket rate limiter. Per-exchange config.
    """

    def __init__(self, capacity: int, refill_rate_per_sec: float):
        self.capacity = capacity
        self.refill_rate = refill_rate_per_sec
        self.tokens = capacity
        self.last_refill = monotonic()
        self._lock = asyncio.Lock()

    async def acquire(self, weight: int = 1) -> None:
        async with self._lock:
            now = monotonic()
            elapsed = now - self.last_refill
            self.tokens = min(self.capacity, self.tokens + elapsed * self.refill_rate)
            self.last_refill = now

            if self.tokens >= weight:
                self.tokens -= weight
                return

            wait_sec = (weight - self.tokens) / self.refill_rate

        await asyncio.sleep(wait_sec)
        await self.acquire(weight)
```

### 4.2 Per-exchange конфигурация

Каждая биржа документирует свои лимиты в `11_EXCHANGE_ADAPTERS/<exchange>.md` (раздел "Rate limits"). Кратко:

| Биржа | Лимит (примерно) | Notes |
|---|---|---|
| Binance | 2400 weight/min для futures, openInterest weight=1 | окно 1 мин |
| Bybit | 600 req/5sec для market endpoints | |
| OKX | 20 req/2sec для public endpoints | per IP |
| Bitget | 20 req/sec | |
| Gate.io | 200 req/10sec | |
| MEXC | 20 req/2sec | |
| KuCoin | 30 req/3sec для futures public | |
| HTX | 800 req/10sec | |
| Hyperliquid | mid-tier, точно не зафиксировано в публичных доках | best effort |
| Aster | копия Binance Futures | |
| Bitmart | ~12 req/sec/IP для contract public | bulk endpoint = 1 запрос/мин — окей |
| XT | 50 req/10sec | bulk endpoint = 1 запрос/мин — окей |

Конфиг хранится в `app/exchanges/<exchange>.py` как class attribute:
```python
class BinanceConnector(BaseConnector):
    RATE_LIMIT_CAPACITY = 2400
    RATE_LIMIT_REFILL_PER_SEC = 40   # 2400/60
```

### 4.3 Request weight

Если у биржи есть концепция "weight" (Binance), коннектор передаёт `weight=N` в `acquire()`:
```python
await self._rate_limiter.acquire(weight=1)   # openInterest weight=1
data = await self._http.get(url)
```

---

## 5. Circuit breaker

### 5.1 Конфигурация

```python
class CircuitBreaker:
    failure_threshold: int = 5         # сколько подряд ошибок для open
    pause_duration_sec: int = 30       # сколько ждём перед re-attempt
    half_open_attempts: int = 1        # сколько проб в half_open
```

### 5.2 Состояния

```
closed (normal) ──5 failures──▶ open ──30s──▶ half_open ──success──▶ closed
                                     ──failure──▶ open
```

### 5.3 Поведение

- В `open` — `execute_cycle` пропускается, возвращает `CycleResult.skipped(reason="circuit_breaker_open")`.
- В `half_open` — пропускается одна попытка; если success → close; если fail → open снова.
- В `closed` — нормальная работа.

### 5.4 Сброс счётчика

При успешном цикле `record_success()` сбрасывает `consecutive_failures = 0`. Это означает, что после **5 подряд** ошибок мы пауза, но если есть **вкрапления успехов** — мы не пауза.

---

## 6. Watermark и инкрементальный fetch

### 6.1 Для snapshot бирж

Watermark не нужен — каждый цикл fetch'им «текущий снимок».

### 6.2 Для native_interval бирж (Bybit, KuCoin)

```python
class WatermarkRepo:
    async def get_watermark(self, exchange: str, native_symbol: str, interval: str) -> Optional[datetime]: ...
    async def set_watermark(self, exchange: str, native_symbol: str, interval: str, ts: datetime) -> None: ...
```

```python
async def _fetch_native_interval(self, instruments: list[Instrument]) -> list[RawExchangeEvent]:
    raw_events = []
    for inst in instruments:
        for interval in ["5m", "15m", "30m"]:
            last_ts = await self._watermark_repo.get_watermark(self.exchange, inst.native_symbol, interval)
            start = last_ts or (utcnow() - timedelta(hours=1))
            end = utcnow()

            bars = await self.fetch_range(inst, interval, start, end)
            raw_events.extend(bars)

            if bars:
                latest_ts = max(b.ts_exchange for b in bars)
                await self._watermark_repo.set_watermark(self.exchange, inst.native_symbol, interval, latest_ts)
    return raw_events
```

### 6.3 Watermark schema

```sql
CREATE TABLE connector_watermarks (
    exchange      TEXT NOT NULL,
    native_symbol TEXT NOT NULL,
    interval      TEXT NOT NULL,
    last_ts       TIMESTAMPTZ NOT NULL,
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (exchange, native_symbol, interval)
);
```

---

## 7. RawExchangeEvent generation

### 7.1 Все коннекторы создают одинаковый формат

```python
def _make_raw_event(
    self,
    payload: dict | list,
    endpoint: str,
    request_params: dict,
    request_duration_ms: int,
    http_status: int,
    native_symbol: str | None = None,
    source_kind: SourceKind | None = None,
) -> RawExchangeEvent:
    raw_str = json.dumps(payload, sort_keys=True)
    raw_hash = hashlib.sha256(raw_str.encode()).hexdigest()

    return RawExchangeEvent(
        exchange=self.exchange,
        endpoint=endpoint,
        request_id=str(uuid4()),
        request_params=request_params,
        fetched_at=utcnow(),
        request_duration_ms=request_duration_ms,
        http_status=http_status,
        response_payload=payload,
        raw_hash=raw_hash,
        native_symbol=native_symbol,
        instrument_version=self._get_current_instrument_version(native_symbol),
        source_kind_intended=source_kind or self.primary_source_kind,
        connector_version=self.connector_version,
    )
```

### 7.2 Идемпотентность по `raw_hash`

Если `raw_hash` уже встречался (биржа вернула буквально тот же payload через минуту), Insert в `raw_exchange_events` всё равно происходит — это легитимные данные (ts_exchange может отличаться, и normalizer создаст новую точку только если ts_exchange новый).

`raw_hash` используется для:
- Дебаг: «биржа залипла, отдаёт один и тот же payload».
- Дедупликации в normalizer (если `raw_hash + ts_exchange` уже был — не нормализуем второй раз).

---

## 8. ConnectorErrorEvent

### 8.1 Когда создаётся

При любом исключении в `execute_cycle()`:
- `httpx.TimeoutException` → `error_type = "timeout"`.
- `httpx.ConnectError` → `error_type = "connection_reset"`.
- `httpx.HTTPStatusError`:
  - 4xx → `error_type = "http_4xx"`.
  - 429 → `error_type = "rate_limited"`.
  - 5xx → `error_type = "http_5xx"`.
- `json.JSONDecodeError` → `error_type = "parse_error"`.
- Прочее → `error_type = "unknown"`.

### 8.2 Запись

```python
async def _record_connector_error(self, exc: Exception) -> None:
    err = ConnectorErrorEvent(
        exchange=self.exchange,
        endpoint=getattr(exc, "request", {}).get("url", ""),
        error_type=self._classify_error(exc),
        error_message=str(exc),
        http_status=getattr(exc, "response", {}).get("status_code"),
        ...
    )
    await self._connector_errors_repo.insert(err)
    metrics.connector_errors_total.labels(
        exchange=self.exchange,
        error_type=err.error_type,
    ).inc()
```

---

## 9. HTTP client конфигурация

### 9.1 Один `httpx.AsyncClient` на коннектор

```python
def create_http_client(exchange: str) -> httpx.AsyncClient:
    return httpx.AsyncClient(
        base_url=EXCHANGE_BASE_URLS[exchange],
        timeout=httpx.Timeout(connect=5.0, read=15.0, write=5.0, pool=5.0),
        limits=httpx.Limits(
            max_connections=20,
            max_keepalive_connections=10,
            keepalive_expiry=30.0,
        ),
        http2=True,
        headers={"User-Agent": f"oi-tracker/{APP_VERSION}"},
        follow_redirects=False,
    )
```

### 9.2 Retry policy на уровне HTTP

Не используем embedded retry в httpx — retries обрабатываются:
- На уровне circuit breaker (между циклами).
- На уровне scheduler (next minute).

Внутри одного цикла попыток нет (one-shot).

Исключение: idempotent GET может ретраиться 1 раз при `httpx.ConnectError` (network glitch):
```python
async def _fetch_with_retry(self, url, params, weight=1, max_retries=1):
    for attempt in range(max_retries + 1):
        try:
            await self._rate_limiter.acquire(weight=weight)
            response = await self._http.get(url, params=params)
            response.raise_for_status()
            return response.json()
        except (httpx.ConnectError, httpx.TimeoutException) as e:
            if attempt == max_retries:
                raise
            await asyncio.sleep(0.5)
```

---

## 10. Per-symbol параллелизм (snapshot бирж)

Большинство snapshot-бирж требуют запрос **per symbol**. ~300 символов × 12 бирж = 3600 запросов в минуту.

### 10.1 asyncio.gather с ограничением

```python
async def fetch_snapshot(self, instruments: list[Instrument]) -> list[RawExchangeEvent]:
    semaphore = asyncio.Semaphore(self.MAX_CONCURRENT_REQUESTS)

    async def fetch_one(inst: Instrument) -> RawExchangeEvent | None:
        async with semaphore:
            try:
                return await self._fetch_oi_for_symbol(inst)
            except Exception as e:
                await self._record_connector_error(e)
                return None

    results = await asyncio.gather(*[fetch_one(inst) for inst in instruments])
    return [r for r in results if r is not None]
```

`MAX_CONCURRENT_REQUESTS` — per-exchange, обычно 10–20.

---

## 11. Структура файлов коннектора

```
app/exchanges/
├── base.py                  # BaseConnector
├── binance.py               # class BinanceConnector(BaseConnector)
├── bybit.py
├── okx.py
├── bitget.py
├── gateio.py
├── mexc.py
├── kucoin.py
├── htx.py
├── hyperliquid.py
├── aster.py
├── bitmart.py
├── xt.py
├── factory.py               # маппинг "binance" → BinanceConnector class
├── http_client.py           # httpx factory
├── rate_limiter.py
├── circuit_breaker.py
└── price_cache.py
```

---

## 12. Контрактные тесты

Каждый коннектор имеет директорию с фикстурами:
```
tests/contract/<exchange>/
├── fixtures/
│   ├── exchange_info.json
│   ├── open_interest_btcusdt.json
│   ├── premium_index.json
│   └── ...
├── test_<exchange>_parser.py
├── test_<exchange>_filter.py
└── test_<exchange>_e2e.py
```

Подробнее — в `14_TEST_STRATEGY.md §3`.

---

## 13. Скелет для нового коннектора

```python
# app/exchanges/example.py

from app.exchanges.base import BaseConnector
from app.domain.enums import SourceKind

class ExampleConnector(BaseConnector):
    exchange = "example"
    primary_source_kind = SourceKind.SNAPSHOT
    connector_version = "1.0.0"

    BASE_URL = "https://api.example.com"
    RATE_LIMIT_CAPACITY = 100
    RATE_LIMIT_REFILL_PER_SEC = 5
    MAX_CONCURRENT_REQUESTS = 10

    async def sync_instruments(self) -> list[Instrument]:
        # 1. GET /v1/exchangeInfo or equivalent
        # 2. Parse, filter only USDT-M perp
        # 3. Convert to Instrument list
        ...

    async def fetch_snapshot(self, instruments) -> list[RawExchangeEvent]:
        # 1. For each instrument, GET /v1/openInterest?symbol=...
        # 2. Make RawExchangeEvent
        ...

    async def fetch_prices(self, instruments) -> dict[str, PriceSnapshot]:
        # 1. GET /v1/ticker (mark/index/last)
        # 2. Convert
        ...
```

Затем регистрация в `factory.py`:
```python
EXCHANGE_REGISTRY = {
    "binance": BinanceConnector,
    "bybit": BybitConnector,
    ...
    "example": ExampleConnector,
}
```

---

## 14. Cross-references

- Per-exchange specifics: `binance.md`, `bybit.md`, `okx.md`, `bitget.md`, `gateio.md`, `mexc.md`, `kucoin.md`, `htx.md`, `hyperliquid.md`, `aster.md`, `bitmart.md`, `xt.md`.
- Schemas → `05_DATA_CONTRACTS.md §3, §11, §13`.
- Instrument resolution → `06_INSTRUMENT_REGISTRY.md`.
- Normalizer reads RawExchangeEvent → `07_NORMALIZER.md §3`.
- Storage → `08_TIME_SERIES_STORAGE.md`.
- Test strategy → `14_TEST_STRATEGY.md §3`.
