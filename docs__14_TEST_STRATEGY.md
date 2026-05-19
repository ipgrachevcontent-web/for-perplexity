# 14 · Test Strategy

> Тестовая пирамида, контрактные тесты, фикстуры, coverage targets.
> Связано: `01_PRODUCT_SPEC.md NFR-8`, `02_ARCHITECTURE.md §8.2`.

---

## 1. Цели тестирования

1. **Корректность нормализации** для каждой биржи: real-world payloads → правильный `NormalizedOISample`.
2. **Корректность state machine** alert engine: правильные переходы при всех сценариях.
3. **Корректность storage**: insert / CA refresh / retention / compression.
4. **Schema drift detection**: тест ловит, когда биржа изменила формат.
5. **End-to-end flow**: synthetic данные → realistic alert.
6. **Idempotency**: replay с той же версией → тот же результат.

---

## 2. Test pyramid

```
                    ▲
                   ╱ ╲
                  ╱E2E╲              ~5% — full flow synthetic
                 ╱─────╲
                ╱       ╲
               ╱Integ-   ╲           ~15% — DB + CA + connectors mocked
              ╱ ration    ╲
             ╱─────────────╲
            ╱               ╲
           ╱ Contract        ╲       ~20% — per-exchange replay tests
          ╱   (per exchange)  ╲
         ╱─────────────────────╲
        ╱                       ╲
       ╱        Unit             ╲   ~60% — pure logic tests
      ╱___________________________╲
```

### 2.1 Coverage targets

| Layer | Target |
|---|---|
| Normalizer (per parser) | ≥ 90% |
| Alert engine (state machine) | ≥ 95% |
| Storage repos | ≥ 85% |
| Connectors (per file) | ≥ 80% |
| Time model utilities | ≥ 95% |
| API endpoints | ≥ 80% |
| Frontend (custom hooks) | ≥ 70% |
| Total backend | ≥ 80% |

Coverage снимается через `pytest-cov`, отчёт в `htmlcov/`.

---

## 3. Unit tests

### 3.1 Что покрываем unit-тестами

- **Pure functions:** `compute_delta_pct`, `humanize_usd`.
- **State machine transitions:** все 16+ переходов из/в каждое состояние.
- **Quality assessor:** matrix valuation_status × warning combinations.
- **Cooldown logic:** smart vs hard cooldown edge cases.
- **Time formulas:** freshness, lookback tolerance, min_history.
- **Symbol mapping:** native_symbol → canonical_symbol для каждой биржи.

### 3.2 Структура

```
backend/tests/unit/
├── normalizer/
│   ├── test_unit_resolver.py
│   ├── test_price_resolver.py
│   ├── test_oi_converter.py
│   ├── test_quality_assessor.py
│   └── test_pipeline.py
├── alerts/
│   ├── test_state_machine.py
│   ├── test_smart_cooldown.py
│   ├── test_quality_gate.py
│   ├── test_signal_threshold.py
│   ├── test_signal_divergence.py
│   └── test_signal_exchange_health.py
├── time_model/
│   ├── test_freshness.py
│   ├── test_bucket_alignment.py
│   └── test_clock_skew.py
├── instruments/
│   ├── test_resolver.py
│   ├── test_versioning.py
│   └── test_lifecycle.py
└── delivery/
    ├── test_tg_templates.py
    ├── test_humanizer.py
    └── test_dedupe_key.py
```

### 3.3 Пример: state machine

```python
# tests/unit/alerts/test_state_machine.py
import pytest
from datetime import datetime, timedelta
from app.alerts.state_machine import transition
from app.domain.events import AlertState, AlertCandidate, AlertRule

@pytest.fixture
def base_rule():
    return AlertRule(
        id=1, name="test",
        rule_type="threshold", scope="global",
        window="5m", metric="delta_oi_pct", operator=">=",
        threshold=Decimal("5.0"),
        cooldown_sec=900, confirm_points=1,
        smart_cooldown_factor=Decimal("1.5"),
        valuation_status_min="good_estimate",
        source_kind_filter=["snapshot", "native_interval"],
        ...
    )

@pytest.mark.parametrize("scenario", [
    {
        "name": "armed_to_fired_on_first_cross",
        "initial_state": "armed",
        "last_eval_value": Decimal("3.0"),
        "candidate_delta": Decimal("5.5"),
        "expected_decision": "fire",
        "expected_state": "fired",
    },
    {
        "name": "armed_stays_below",
        "initial_state": "armed",
        "last_eval_value": Decimal("3.0"),
        "candidate_delta": Decimal("4.5"),
        "expected_decision": None,
        "expected_state": "armed",
    },
    {
        "name": "armed_already_above_no_cross",
        "initial_state": "armed",
        "last_eval_value": Decimal("5.5"),     # уже выше
        "candidate_delta": Decimal("6.0"),
        "expected_decision": None,             # не crossing
        "expected_state": "armed",
    },
    {
        "name": "cooldown_smart_re_fire",
        "initial_state": "cooldown",
        "last_fired_value": Decimal("5.2"),
        "candidate_delta": Decimal("8.0"),     # 8.0 >= 5.2 * 1.5 = 7.8
        "expected_decision": "fire",
        "expected_state": "fired",
    },
    {
        "name": "cooldown_no_smart_re_fire",
        "initial_state": "cooldown",
        "last_fired_value": Decimal("5.2"),
        "candidate_delta": Decimal("7.0"),     # 7.0 < 7.8
        "expected_decision": None,
        "expected_state": "cooldown",
    },
    {
        "name": "cooldown_to_armed_after_ttl",
        "initial_state": "cooldown",
        "cooldown_until": datetime(2026, 4, 30, 11, 0, tzinfo=UTC),  # past
        "candidate_delta": Decimal("3.0"),     # below threshold
        "now": datetime(2026, 4, 30, 12, 0, tzinfo=UTC),
        "expected_state": "armed",
    },
    {
        "name": "insufficient_history",
        "initial_state": "idle",
        "history_minutes": 5,                   # < min_history
        "expected_decision": "insufficient_history",
        "expected_state": "idle",
    },
    {
        "name": "stale_data_skip",
        "initial_state": "armed",
        "sample_age_seconds": 200,             # > 120s budget
        "expected_decision": "late_data_skip",
    },
])
def test_state_transition(scenario, base_rule):
    state = AlertState(
        rule_id=1, exchange="binance", canonical_symbol="BTC", window="5m",
        state=scenario["initial_state"],
        last_eval_value=scenario.get("last_eval_value"),
        last_fired_value=scenario.get("last_fired_value"),
        cooldown_until=scenario.get("cooldown_until"),
        consecutive_above_count=0,
        last_eval_at=None, last_fired_at=None,
    )
    candidate = AlertCandidate(
        rule_id=1, exchange="binance", canonical_symbol="BTC", window="5m",
        delta_pct=scenario["candidate_delta"],
        sample_age_seconds=scenario.get("sample_age_seconds", 30),
        history_minutes=scenario.get("history_minutes", 60),
        candidate_ts_processed=scenario.get("now", utcnow()),
        ...
    )
    new_state = transition(state, candidate, base_rule)

    if "expected_state" in scenario:
        assert new_state.state == scenario["expected_state"]
    if "expected_decision" in scenario:
        assert new_state._decision == scenario["expected_decision"]
```

---

## 4. Contract tests (per exchange)

### 4.1 Что это

Контрактные тесты — это unit-tests с **реальными payload'ами** от бирж. Они гарантируют, что наш parser корректно работает с **актуальной** версией API.

### 4.2 Структура

```
tests/contract/
├── conftest.py                    # фикстуры loading helpers
├── binance/
│   ├── fixtures/
│   │   ├── exchange_info.json
│   │   ├── open_interest_btcusdt.json
│   │   ├── open_interest_hist_btcusdt.json
│   │   ├── premium_index_all.json
│   │   └── README.md              # дата захвата, версия API
│   ├── test_binance_filter.py     # is_usdt_m_perpetual для разных типов
│   ├── test_binance_parser.py     # parse_open_interest для всех endpoint'ов
│   ├── test_binance_normalize.py  # full normalize pipeline для одной точки
│   └── test_binance_e2e.py        # mock httpx → connector cycle → samples in DB
├── bybit/
│   └── ... (аналогично)
└── ... (12 директорий)
```

### 4.3 Фикстуры

Захватываются из реальных API responses:

```python
# tests/contract/conftest.py
import json
from pathlib import Path

FIXTURES_DIR = Path(__file__).parent

def load_fixture(exchange: str, name: str) -> dict | list:
    path = FIXTURES_DIR / exchange / "fixtures" / name
    with path.open() as f:
        return json.load(f)
```

Использование:
```python
def test_binance_parser_open_interest():
    payload = load_fixture("binance", "open_interest_btcusdt.json")
    parser = BinanceParser()
    parsed = parser.parse_open_interest(payload)

    assert parsed.native_symbol == "BTCUSDT"
    assert parsed.oi_unit_hint == "base_asset"
    assert parsed.oi_raw == Decimal("10659.509")
    assert parsed.ts_exchange == datetime(2020, 5, 14, 6, 25, 30, 11000, tzinfo=UTC)
```

### 4.4 Fixture refresh policy

Раз в месяц или при подозрении на schema drift:
1. Запускаем capture-script:
   ```bash
   python -m app.tools.capture_fixtures --exchange=binance --output=tests/contract/binance/fixtures/
   ```
2. Запускаем тесты:
   ```bash
   pytest tests/contract/binance/
   ```
3. Если упали — investigation. Может быть schema drift, может быть наш парсер слабый.

### 4.5 Schema drift тест

Для каждого parser'а — отдельный тест на устойчивость к drift:

```python
def test_binance_parser_survives_extra_field():
    payload = load_fixture("binance", "open_interest_btcusdt.json")
    payload["extraNewField"] = "value"          # биржа добавила поле

    parser = BinanceParser()
    parsed = parser.parse_open_interest(payload)
    assert parsed.native_symbol == "BTCUSDT"     # парсер игнорирует unknown


def test_binance_parser_raises_on_missing_required():
    payload = load_fixture("binance", "open_interest_btcusdt.json")
    del payload["openInterest"]                   # биржа убрала поле

    parser = BinanceParser()
    with pytest.raises(ParseError, match="openInterest"):
        parser.parse_open_interest(payload)


def test_binance_parser_handles_changed_type():
    payload = load_fixture("binance", "open_interest_btcusdt.json")
    payload["openInterest"] = 10659.509          # из строки в число

    parser = BinanceParser()
    parsed = parser.parse_open_interest(payload)
    assert parsed.oi_raw == Decimal("10659.509") # парсер смотрит на str() обертки
```

---

## 5. Integration tests

### 5.1 Что покрываем

- Full normalizer pipeline → INSERT в `oi_samples`.
- CA refresh → правильные агрегаты.
- Retention → удаление старых chunks.
- Compression → данные сжаты, читаемы.
- Late insert в compressed chunk.
- Replay process.
- alert_engine evaluation cycle с реальной БД.

### 5.2 Test DB setup

Используем `pytest-postgres` или `testcontainers-python`:

```python
# tests/integration/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer

@pytest.fixture(scope="session")
def test_db():
    with PostgresContainer("timescale/timescaledb:latest-pg16") as pg:
        # Run migrations
        os.environ["DATABASE_URL"] = pg.get_connection_url()
        subprocess.run(["alembic", "upgrade", "head"], check=True)
        yield pg
```

Альтернатива (быстрее, но менее изолированно): отдельная `oi_tracker_test` БД на том же PG, очищается перед каждым test'ом.

### 5.3 Пример

```python
async def test_oi_samples_insert_and_ca_refresh(test_db):
    # 1. Insert
    samples = generate_samples(count=300, exchange="binance", canonical_symbol="BTC")
    await repo.bulk_insert(samples)

    # 2. Refresh CA
    await db.execute("CALL refresh_continuous_aggregate('oi_5m', null, null)")

    # 3. Query CA
    bars = await db.fetch("""
        SELECT * FROM oi_5m
        WHERE exchange='binance' AND canonical_symbol='BTC'
        ORDER BY bucket
    """)
    assert len(bars) > 0
    assert bars[0]["sample_count"] > 0


async def test_late_insert_in_compressed_chunk(test_db):
    # 1. Insert старые данные
    old_samples = generate_samples(at_time=utcnow() - timedelta(days=10), count=10)
    await repo.bulk_insert(old_samples)

    # 2. Compress
    await db.execute("""
        SELECT compress_chunk(c)
        FROM show_chunks('oi_samples', older_than => INTERVAL '7 days') c
    """)

    # 3. Insert новой точки в тот же chunk (late data)
    late_sample = old_samples[0].model_copy(update={
        "ts_exchange": old_samples[0].ts_exchange + timedelta(seconds=30),
        "oi_coins": old_samples[0].oi_coins + Decimal("100"),
    })
    await repo.bulk_insert([late_sample])

    # 4. Verify
    result = await db.fetch("SELECT count(*) FROM oi_samples WHERE ...")
    assert result[0]["count"] == 11
```

---

## 6. End-to-end tests

### 6.1 Что покрываем

- Synthetic exchange responses → connector → normalizer → storage → CA → alert engine → delivery_queue.
- Проверка дедупликации через несколько eval cycles.
- Проверка smart cooldown поведения.
- Failure injection: коннектор падает → правильный recovery.

### 6.2 Структура

```
tests/e2e/
├── conftest.py                    # spin up PG + mock HTTP servers
├── test_full_pipeline_threshold.py
├── test_full_pipeline_divergence.py
├── test_full_pipeline_exchange_health.py
├── test_smart_cooldown.py
├── test_no_retro_fire_after_replay.py
├── test_failure_recovery.py
└── mock_exchange_server.py        # FastAPI server that pretends to be Binance/Bybit/...
```

### 6.3 Mock exchange server

```python
# tests/e2e/mock_exchange_server.py
from fastapi import FastAPI

app = FastAPI()

# State managed by tests
app.state.scenarios = {}      # symbol → list of OI values to return

@app.get("/fapi/v1/openInterest")
def get_oi(symbol: str):
    scenario = app.state.scenarios.get(symbol, [])
    if not scenario:
        return {"openInterest": "100", "symbol": symbol, "time": int(time.time() * 1000)}
    next_value = scenario.pop(0)
    return {"openInterest": str(next_value), "symbol": symbol, "time": int(time.time() * 1000)}
```

### 6.4 Пример

```python
async def test_e2e_threshold_alert_fires(test_db, mock_binance_server):
    # 1. Setup: BTC OI 100 → 110 (10% pump)
    mock_binance_server.scenarios["BTCUSDT"] = [
        Decimal("100"), Decimal("100"), Decimal("100"), Decimal("100"), Decimal("100"),
        Decimal("110"),  # +10% относительно 5 min ago
    ]
    # ... + setup mark_price = 67000

    # 2. Run 6 connector cycles + alert engine eval
    for _ in range(6):
        await scheduler.run_one_cycle()
        await alert_engine.run_one_cycle()
        await asyncio.sleep(0.01)

    # 3. Check delivery_queue
    pending = await db.fetch("SELECT * FROM delivery_queue WHERE template='oi_threshold_cross'")
    assert len(pending) >= 1
    assert pending[0]["payload"]["canonical_symbol"] == "BTC"
    assert Decimal(pending[0]["payload"]["delta_pct"]) >= Decimal("9")    # ~10% pump
```

---

## 7. Performance tests

### 7.1 Что меряем

- Bulk insert throughput: 3600 samples/cycle должны вставляться в <500ms.
- CA refresh duration: для последнего часа должно быть <2s.
- Alert engine cycle: 6 default rules × 12 ex × 300 sym должны эвалюироваться в <5s.
- API endpoint latency: GET /api/v1/live в <100ms.

### 7.2 Запуск

```bash
pytest tests/performance/ -m performance --benchmark-only
```

```python
@pytest.mark.performance
def test_bulk_insert_throughput(test_db, benchmark):
    samples = generate_samples(count=3600)
    result = benchmark(repo.bulk_insert_sync, samples)
    assert benchmark.stats["mean"] < 0.5    # 500ms
```

---

## 8. Property-based tests (опционально)

Через `hypothesis`:

```python
from hypothesis import given, strategies as st

@given(
    oi_now=st.decimals(min_value=Decimal("0.001"), max_value=Decimal("10000000")),
    oi_prev=st.decimals(min_value=Decimal("0.001"), max_value=Decimal("10000000")),
)
def test_compute_delta_pct_bounded(oi_now, oi_prev):
    delta = compute_delta_pct(oi_now, oi_prev)
    # Не бесконечность, не NaN
    assert delta.is_finite()
```

---

## 9. Frontend tests

### 9.1 Стек

- Vitest (Vite-native test runner, jest-compatible API).
- React Testing Library (component tests).
- MSW (Mock Service Worker для mocking API calls).

### 9.2 Что покрываем

- Custom hooks: `useSSE`, `useApi` (моки EventSource / fetch).
- Logic в `Dashboard.tsx`, `Symbol.tsx`: фильтры, сортировка, decimation.
- Form validation в `Settings.tsx`.

### 9.3 Не покрываем (в v1)

- Visual regression / Playwright e2e — overkill для single-user приложения. Можно добавить позже.

---

## 10. CI / CD

### 10.1 Pipeline

В v1 без полноценного CI (single-user, single-server). Локальные правила:

1. Перед commit:
   ```bash
   make lint        # ruff, mypy
   make test-unit   # быстрые тесты
   ```
2. Перед deploy:
   ```bash
   make test-all    # unit + integration + contract
   ```
3. После deploy:
   - Manual smoke test в браузере.
   - Проверить логи 10 минут.

### 10.2 Future CI

Когда придёт время:
- GitHub Actions / Gitea Actions.
- Каждый push → unit + contract tests.
- Каждый PR → integration tests с testcontainers.
- Tag `v*` → deploy script.

---

## 11. Pytest config

```toml
# backend/pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
markers = [
    "unit: pure logic tests, no IO",
    "contract: per-exchange replay tests",
    "integration: tests with real DB",
    "e2e: full pipeline tests",
    "performance: benchmark tests",
    "slow: skip in fast CI",
]
addopts = "-ra --strict-markers --tb=short"
```

```bash
# Run by category
pytest -m unit              # быстрый прогон
pytest -m "not slow"        # без долгих
pytest -m contract          # only contract
pytest                      # all
```

---

## 12. Linting & type checking

### 12.1 Backend

```bash
# ruff: fast linter + formatter
ruff check backend/app/
ruff format backend/app/

# mypy: strict mode
mypy --strict backend/app/
```

`pyproject.toml`:
```toml
[tool.ruff]
target-version = "py312"
line-length = 100

[tool.ruff.lint]
select = ["E", "W", "F", "I", "N", "UP", "B", "S", "A", "C4", "DTZ", "T20", "RET", "SIM"]
ignore = ["S101"]    # allow assert in tests

[tool.mypy]
python_version = "3.12"
strict = true
warn_unreachable = true
disallow_untyped_defs = true
```

### 12.2 Frontend

```bash
# ESLint + Prettier
npm run lint
npm run format

# TypeScript strict
npm run type-check
```

---

## 13. Test data generation

### 13.1 Synthetic samples

```python
# tests/fixtures/factories.py
def generate_samples(
    count: int = 100,
    exchange: str = "binance",
    canonical_symbol: str = "BTC",
    start_time: datetime | None = None,
    interval_seconds: int = 60,
    base_oi: Decimal = Decimal("10000"),
    drift_per_step: Decimal = Decimal("10"),
    price: Decimal = Decimal("67000"),
) -> list[NormalizedOISample]:
    start_time = start_time or (utcnow() - timedelta(seconds=count * interval_seconds))
    samples = []
    for i in range(count):
        ts = start_time + timedelta(seconds=i * interval_seconds)
        oi_coins = base_oi + drift_per_step * i
        samples.append(NormalizedOISample(
            ts_exchange=ts,
            ts_ingested=ts + timedelta(seconds=2),
            exchange=exchange,
            canonical_symbol=canonical_symbol,
            native_symbol=f"{canonical_symbol}USDT",
            oi_coins=oi_coins,
            oi_native_unit="base_asset",
            oi_notional_usdt=oi_coins * price,
            price_used=price,
            price_source="mark",
            source_kind="snapshot",
            valuation_status="good_estimate",
            normalization_version=1,
            instrument_version=1,
            warnings=[],
            raw_event_id=i,
            raw_hash=f"hash_{i}",
            ...
        ))
    return samples
```

### 13.2 Pump scenario

```python
def generate_pump_scenario(
    base_oi: Decimal = Decimal("10000"),
    pump_pct: Decimal = Decimal("10"),
    pump_at_minute: int = 5,
    total_minutes: int = 10,
) -> list[NormalizedOISample]:
    samples = []
    for i in range(total_minutes):
        if i < pump_at_minute:
            oi = base_oi
        else:
            oi = base_oi * (Decimal("1") + pump_pct / Decimal("100"))
        ...
    return samples
```

---

## 14. Cross-references

- Coverage targets → `01_PRODUCT_SPEC.md NFR-8`.
- Per-exchange parsers → `11_EXCHANGE_ADAPTERS/<exchange>.md`.
- State machine logic → `09_ALERT_ENGINE.md §3`.
- Storage operations → `08_TIME_SERIES_STORAGE.md`.
- Schema contracts → `05_DATA_CONTRACTS.md`.
