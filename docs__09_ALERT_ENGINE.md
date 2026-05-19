# 09 · Alert Engine

> Stateful движок принятия решений: когда отправлять алерт, когда молчать, когда фиксировать «вернулось».
> Связано: `00_DECISIONS_LOG.md C4, C6, C7, F6, F7, F8, F9`, `05_DATA_CONTRACTS.md §6, §7, §8, §9`.

---

## 1. Назначение

Alert engine — это слой, превращающий поток `OIBar` (5/15/30 min) в решения о доставке сигналов. Это **не** условный `if delta > threshold: send()` — это:
- Stateful machine с дедупликацией.
- Quality gates (свежесть, valuation_status, history).
- Smart cooldown (re-fire при усилении).
- Поддержка 5 типов сигналов.
- No-retro-fire policy.

**Принцип:** *normalizer отвечает за истинность точки, time-series storage — за истинность ряда, alert engine — за истинность события.*

---

## 2. Pipeline

```
oi_5m / oi_15m / oi_30m
        │ (read)
        ▼
┌──────────────────┐
│ 1. Candidate     │  Достать (curr, prev) для каждой
│    Selector      │  применимой (rule, exchange, symbol, window)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 2. Quality Gate  │  Свежесть, valuation, history,
│                  │  source_kind фильтр, min_oi_notional
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 3. Rule          │  Для каждого правила: оператор + threshold
│    Evaluator     │  → result: above | below | crossing | not_crossed
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 4. State Machine │  Применить переход состояний
│                  │  (idle → armed → fired → cooldown → armed)
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 5. Smart Cooldown│  В cooldown? Δ ≥ 1.5×?
│    + Dedupe      │  dedupe_key уже отправлен?
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ 6. Decision      │  FIRE / SUPPRESS / RESOLVE / late_data_skip / ...
│    + Audit log   │  Запись в alert_events
└────────┬─────────┘
         │ (only if FIRE)
         ▼
┌──────────────────┐
│ 7. Delivery      │  INSERT в delivery_queue
│    Producer      │
└──────────────────┘
```

---

## 3. State machine

### 3.1 Состояния

```
                    ┌─────────────────────────────┐
                    │                             │
                    │ Симптом: новый символ       │
                    │ (insufficient_history)      │
                    ▼                             │
                ┌───────┐                         │
        ┌──────▶│ idle  │                         │
        │       └───┬───┘                         │
        │           │ history >= min_history      │
        │           ▼                             │
        │       ┌───────┐                         │
        │       │ armed │◀────────┐               │
        │       └───┬───┘         │               │
        │           │             │               │
        │           │ rising      │ value         │
        │           │ edge cross  │ goes back     │
        │           │ (above thr) │ below thr     │
        │           ▼             │               │
        │       ┌───────┐    ┌────┴───────┐       │
        │       │ fired │───▶│ cooldown   │       │
        │       └───┬───┘    └────┬───────┘       │
        │           │             │               │
        │           │             │ cooldown      │
        │           │             │ TTL expired   │
        │           │             │ AND value     │
        │           │             │ below thr     │
        │           │             ▼               │
        │           │         (back to armed)─────┘
        │           │
        │           │ smart cooldown: new Δ ≥ 1.5× last
        │           │ → re-fire (stay in fired)
        │           └────loop───────────────┐
        │                                   │
        └───────────────────────────────────┘
        (rare: rule disabled / instrument delisted)
```

### 3.2 Переходы

| Из | В | Условие |
|---|---|---|
| `idle` | `armed` | `history_minutes >= min_history` |
| `armed` | `fired` | rising edge crossing: `prev < threshold AND curr >= threshold` (для up); confirm_points satisfied; quality gates passed |
| `armed` | `armed` | value < threshold; либо value >= threshold но quality gate fails |
| `fired` | `cooldown` | сразу после успешной записи в delivery_queue |
| `cooldown` | `fired` | smart cooldown: `current_delta >= 1.5 × last_fired_delta` AND `value >= threshold` (повторный fire без выхода из cooldown) |
| `cooldown` | `armed` | `cooldown_until <= now()` AND `value < threshold` |
| `cooldown` | `cooldown` | cooldown активен и нет smart-fire триггера |
| any | `idle` | `instrument_delisted` ИЛИ `rule_disabled` |

### 3.3 Persistence

Состояние хранится в `alert_state` (см. `05_DATA_CONTRACTS.md §9.3`). PK: `(rule_id, exchange, canonical_symbol, window)`.

State machine **переживает рестарт** процесса: при старте scheduler читает `alert_state`, не теряет cooldown'ы.

### 3.4 Pseudocode

```python
def transition(state: AlertState, candidate: AlertCandidate, rule: AlertRule) -> NewState:
    now = candidate.candidate_ts_processed

    # 1. Insufficient history
    if candidate.history_minutes < rule.effective_min_history():
        return state.with_(state="idle", reason="insufficient_history")

    # 2. Quality gate
    if not _quality_gate_pass(candidate, rule):
        return state.with_(reason="quality_gate_fail")

    # 3. Late data
    if candidate.sample_age_seconds > settings.freshness_budget_now_sec:
        return state.with_(reason="late_data_skip")

    # 4. Crossing logic
    crosses_up   = (state.last_eval_value or 0) < rule.threshold and candidate.delta_pct >= rule.threshold
    crosses_down = (state.last_eval_value or 0) > rule.threshold and candidate.delta_pct <= rule.threshold
    above_thr    = candidate.delta_pct >= rule.threshold if rule.operator == ">=" else candidate.delta_pct <= rule.threshold

    # 5. Apply state machine
    if state.state == "idle":
        return state.with_(state="armed")

    if state.state == "armed":
        if above_thr and crosses_up:
            # Confirm points logic
            new_count = state.consecutive_above_count + 1
            if new_count >= rule.confirm_points:
                return _fire(state, candidate, rule, now)
            return state.with_(consecutive_above_count=new_count)
        else:
            return state.with_(consecutive_above_count=0)

    if state.state == "fired":
        # Right after firing, transition to cooldown
        return state.with_(state="cooldown",
                           cooldown_until=now + timedelta(seconds=rule.cooldown_sec))

    if state.state == "cooldown":
        if state.cooldown_until and now >= state.cooldown_until:
            if not above_thr:
                return state.with_(state="armed", consecutive_above_count=0)
            # Если cooldown истёк, но всё ещё выше — остаёмся в cooldown,
            # ждём пока выйдет из threshold (no perpetual re-fire)
            return state

        # Smart cooldown: re-fire?
        smart_factor = rule.smart_cooldown_factor   # default 1.5
        if (
            state.last_fired_value is not None
            and abs(candidate.delta_pct) >= abs(state.last_fired_value) * smart_factor
            and above_thr
        ):
            return _fire(state, candidate, rule, now,
                         extend_cooldown=True,
                         reason="smart_cooldown_re_fire")
        return state

    raise InvalidStateError(state.state)


def _fire(state, candidate, rule, now, extend_cooldown=False, reason="threshold_cross"):
    return state.with_(
        state="fired",
        last_fired_at=now,
        last_fired_value=candidate.delta_pct,
        last_eval_value=candidate.delta_pct,
        last_eval_at=now,
        consecutive_above_count=0,
        cooldown_until=now + timedelta(seconds=rule.cooldown_sec)
            if extend_cooldown else state.cooldown_until,
        _decision="fire",
        _reason=reason,
    )
```

---

## 4. Quality gates

### 4.1 Что проверяется

Для каждого `(rule, candidate)`:

1. **Свежесть последней точки:**
   `now() - candidate.last_ts_exchange <= settings.freshness_budget_now_sec` (default 120s).
2. **Valuation status:** `candidate.worst_valuation_status >= rule.valuation_status_min`.
3. **Source kind:** `candidate.last_source_kind in rule.source_kind_filter` (default включает `snapshot` и `native_interval`, исключает `degraded_fallback`).
4. **Минимальный OI:** `candidate.oi_now_notional_usdt >= rule.min_oi_notional_usdt`.
5. **Минимум истории:** `candidate.history_minutes >= rule.effective_min_history()` (формула `max(30, 2 × window)`).
6. **Δ доступна:** `candidate.delta_pct is not None` (есть prev_bar в tolerance window).

### 4.2 Если quality gate не прошёл

- Никакого `transition` не происходит.
- В `alert_events` пишется решение с конкретной `decision` enum:
  - `quality_gate_fail` — общий случай.
  - `late_data_skip` — конкретно freshness.
  - `insufficient_history` — конкретно history.
  - `below_min_oi` — конкретно min_oi_notional.

### 4.3 Smart valuation

Если `valuation_status == "low_confidence"` для **верхнего** порога (например +5%) — пропускаем.
Но если low_confidence относится к **очень крупному** OI (10× от mean), это может быть pump на тонкой ликвидности — **тоже** пропускаем (это не "high signal").

В v1 — простая логика: `valuation_status >= rule.valuation_status_min`. Усложнения — future work.

---

## 5. Типы сигналов

### 5.1 Threshold (базовый)

```python
@register_signal_type("threshold")
class ThresholdSignal:
    """delta_window_pct >= threshold (или <= для падения)."""

    async def evaluate(rule: AlertRule, ctx: EvaluationContext) -> list[AlertCandidate]:
        bars_now  = await ctx.storage.get_latest_bars(rule.window, exchanges=rule.exchange_filter)
        bars_prev = await ctx.storage.get_previous_bars(rule.window, exchanges=rule.exchange_filter)
        candidates = []
        for (ex, sym), curr in bars_now.items():
            prev = bars_prev.get((ex, sym))
            delta = compute_delta_pct(curr, prev)
            cand = AlertCandidate(
                rule_id=rule.id,
                exchange=ex,
                canonical_symbol=sym,
                window=rule.window,
                current_bar=curr,
                prev_bar=prev,
                delta_pct=delta,
                ...
            )
            candidates.append(cand)
        return candidates
```

Это базовый и **самый частый** сигнал. 6 default-правил (`up_5m, down_5m, ..., down_30m`) — все типа `threshold`.

### 5.2 Confirmed

То же самое, что threshold, но с `confirm_points >= 2`.
- Условие должно держаться 2 последовательных evaluation cycles.
- Это даёт +1 evaluation cycle латентности (по умолчанию ~30 сек), но снижает шум.
- Реализуется через `consecutive_above_count` в `alert_state`.

### 5.3 ~~Consensus~~ (sunset, F40)

Sunset per `00_DECISIONS_LOG F40` (M8 Phase A). Один алерт = одна биржа = одно сообщение. Группировка по символу больше не делается — `RuleType.CONSENSUS`, `signals/consensus.py`, sentinel `<consensus>`, `consensus_count` параметр и `consensus_min_exchanges` setting удалены.

### 5.4 Divergence

Триггер: расхождение между OI и ценой.

**Тип A:** OI растёт, цена падает (или наоборот).
**Тип B:** OI ускоряется, цена флэтит.

В v1 реализуем тип A:

```python
@register_signal_type("divergence")
class DivergenceSignal:
    async def evaluate(rule: AlertRule, ctx: EvaluationContext) -> list[AlertCandidate]:
        candidates = []
        for (ex, sym), curr in ctx.current_bars[rule.window].items():
            prev = ctx.previous_bars[rule.window].get((ex, sym))
            delta_oi = compute_delta_pct(curr.close_oi_coins, prev.close_oi_coins)
            delta_price = compute_delta_pct(curr.close_price, prev.close_price)

            divergence_type = None
            if delta_oi >= rule.threshold and delta_price <= -rule.divergence_price_threshold:
                divergence_type = "oi_up_price_down"
            elif delta_oi <= -rule.threshold and delta_price >= rule.divergence_price_threshold:
                divergence_type = "oi_down_price_up"

            if divergence_type:
                cand = AlertCandidate(
                    rule_id=rule.id,
                    exchange=ex,
                    canonical_symbol=sym,
                    window=rule.window,
                    delta_pct=delta_oi,
                    extra={"divergence_type": divergence_type, "delta_price_pct": delta_price},
                    ...
                )
                candidates.append(cand)
        return candidates
```

**Дополнительные параметры divergence rule:**
- `divergence_price_threshold` — какой Δ цены считаем «значимым» (default 1%).
- divergence игнорирует `degraded_fallback` source (вычитка из low-confidence price ненадёжна).

### 5.5 Exchange-health

Не market signal, а **операционный**. Триггеры:
- `stale_data` — для биржи max(ts_exchange) старше now() − 5 минут.
- `low_coverage` — `symbol_coverage_ratio < 95%` для биржи в течение 3 циклов подряд.
- `high_error_rate` — `connector_errors_per_minute > 5` для биржи.
- `instruments_sync_failing` — sync_job провалился 3 раза подряд.

```python
@register_signal_type("exchange_health")
class ExchangeHealthSignal:
    async def evaluate(rule: AlertRule, ctx: EvaluationContext) -> list[AlertCandidate]:
        candidates = []
        # 1. stale_data
        stale = await ctx.health_repo.find_stale_exchanges(threshold_minutes=5)
        for ex in stale:
            candidates.append(_make_health_candidate(rule, ex, "stale_data"))

        # 2. low_coverage (тоже 3 cycles)
        low_cov = await ctx.quality_repo.find_low_coverage(threshold=0.95, cycles=3)
        for ex, ratio in low_cov:
            candidates.append(_make_health_candidate(rule, ex, "low_coverage", details={"ratio": ratio}))

        # 3. high_error_rate
        # 4. instruments_sync_failing
        ...
        return candidates
```

Cooldown для exchange-health длиннее (1 час по умолчанию), чтобы не спамить во время длительной деградации биржи.

#### v1 implementation scope (M3.E, F25)

В v1 реализован только sub-trigger `stale_data`. Параметры через `rule.extra`:

```jsonb
{ "sub_trigger": "stale_data", "stale_threshold_min": 5 }
```

Остальные 3 sub-triggers (`low_coverage`, `high_error_rate`,
`instruments_sync_failing`) **deferred** до появления
`connector_health_history` table или Prometheus scrape adapter — без них
у нас нет persisted N-cycle history для расчёта.

#### Bypass-дизайн (F25)

ExchangeHealthSignal возвращает `HealthCandidate` (не `AlertCandidate`).
`evaluation_cycle._evaluate_exchange_health_rule` обходит state_machine и
пишет напрямую в `delivery_queue` через `delivery_producer.enqueue_health()`.

Cooldown enforced через dedupe_key:
```
health:rule:<id>|ex:<exchange>|trig:<sub_trigger>|bucket:<floor(now/cooldown_sec)>
```
+ `ON CONFLICT (dedupe_key) DO NOTHING` — внутри одного cooldown-bucket
только одна доставка.

`alert_state` для health не пишется (audit log отложен до v1.1).

### 5.6 Volume-threshold (F51, Aster-only)

Aster обновляет `/fapi/v1/openInterest` нерегулярно раз в 5–25 мин, поэтому 5m OI-Δ% на Aster нерепрезентативен (см. `00_DECISIONS_LOG F51`). Вместо OI для Aster@5m считаем Δ% **торгового объёма** через `quoteAssetVolume` из 5m kline.

Источник данных — hypertable `volume_samples_5m` (заполняется `_aster_volume_lifecycle` в scheduler). Сигнал читает 2 последних **закрытых** бакета через `volume_samples.get_latest_two_closed_buckets`, чтобы не сравнивать с растущим in-progress бакетом.

Formula: `delta_pct = (current.quote_volume_usdt − prev.quote_volume_usdt) / prev.quote_volume_usdt × 100`.

Filter (применяется ДО Δ%): `current.quote_volume_usdt ≥ settings_kv.min_volume_usdt_default` (default `"50000"`, JSON-stringified Decimal в KV). Защищает от шума микропар, где краткий всплеск с $10 → $50 даёт технически +400% но операционно не релевантен.

**Synthetic OIBar trick.** AlertCandidate в системе спроектирован вокруг `current_bar: OIBar`. Чтобы не делать AlertCandidate полиморфным, сигнал конструирует `OIBar` из `VolumeSample5m` (close_oi_notional_usdt = quote_volume_usdt, close_oi_coins = base_volume). State machine работает с ним идентично OI-кандидату; маскировка типа делается на producer-уровне через routing по `rule.rule_type`.

**Свежесть.** `sample_age_seconds = now − current.ts_ingested` (НЕ `now − bucket`). Closed bucket по определению ≥ 5 мин старше now, поэтому стандартный OI-style budget (120s) всё бы отсёк. Свежесть здесь — это «когда мы прочитали kline в последнем poll'е», не «время bucket'а».

**Guard в OI-`ThresholdSignal`** (`signals/threshold.py:54-62`): пропускает пары `(exchange='aster', window='5m')`. Двойной fire (OI + volume) исключён.

Default rules (миграция `0017_aster_volume_defaults.py`):
- `Aster volume up 5m`: `threshold = +5.0`, `operator = '>='`, `exchange_filter = ['aster']`, `min_oi_notional_usdt = 0`.
- `Aster volume down 5m`: `threshold = -5.0`, `operator = '<='`, остальное идентично.

KV `settings_kv.min_volume_usdt_default = "50000"` (USDT, JSON-stringified Decimal, editable через Settings UI — PR3).

Продукт-логика: 15m/30m на Aster и все 11 остальных бирж на всех окнах — без изменений, продолжают работать на OI-сигнале.

---

## 6. Smart cooldown

### 6.1 Алгоритм

```python
def is_smart_re_fire(state: AlertState, candidate: AlertCandidate, rule: AlertRule) -> bool:
    if state.state != "cooldown":
        return False
    if state.last_fired_value is None:
        return False
    factor = rule.smart_cooldown_factor    # default 1.5

    # Для роста: новое Δ должно быть в 1.5× больше старого (например 5% → 7.5%)
    if state.last_fired_value > 0:
        return candidate.delta_pct >= state.last_fired_value * factor

    # Для падения: новое Δ должно быть в 1.5× больше по абсолюту (-5% → -7.5%)
    if state.last_fired_value < 0:
        return candidate.delta_pct <= state.last_fired_value * factor

    return False
```

### 6.2 Поведение

```
Time   | Δ_5m  | State        | Action
-------+-------+--------------+--------------------------------------------
12:00  | +3.0% | armed        | (under threshold)
12:05  | +5.2% | armed→fired  | crossed +5%, FIRE: "BTC +5.2%"
12:05  | +5.2% | fired→cooldwn| persisted; cooldown 12:20
12:10  | +6.0% | cooldown     | 6.0 < 5.2*1.5=7.8, suppress
12:15  | +9.5% | cooldown     | 9.5 >= 7.8, smart re-fire: "BTC ускорился +9.5%",
                                cooldown extended to 12:30
12:20  | +9.0% | cooldown     | 9.0 < 9.5*1.5=14.25, suppress
12:31  | +6.0% | cooldown→armed| cooldown expired AND below threshold? нет, выше.
                                остаёмся в cooldown пока не упадёт
12:35  | +4.5% | cooldown→armed| ниже threshold → armed
12:42  | +5.3% | armed→fired  | rising edge снова, FIRE
```

### 6.3 Extension cooldown при smart re-fire

После smart re-fire `cooldown_until` сдвигается на `now() + cooldown_sec`. То есть каждое усиление перезапускает таймер. Это страховка от слишком частых re-fire.

---

## 7. Default правила

При первой инициализации БД seed'им 6 базовых правил типа `threshold`. Конкретные значения (5%/8%/12% × up/down) зафиксированы как `00_DECISIONS_LOG F18`:

```sql
INSERT INTO alert_rules (
    name, rule_type, scope,
    window, metric, operator, threshold,
    cooldown_sec, confirm_points, smart_cooldown_factor,
    min_oi_notional_usdt, valuation_status_min, source_kind_filter,
    enabled
) VALUES
    ('Default up 5m',   'threshold', 'global',
     '5m',  'delta_oi_pct', '>=',  5.0,
     900, 1, 1.5, 5000000, 'good_estimate', ARRAY['snapshot','native_interval'], true),

    ('Default down 5m', 'threshold', 'global',
     '5m',  'delta_oi_pct', '<=', -5.0,
     900, 1, 1.5, 5000000, 'good_estimate', ARRAY['snapshot','native_interval'], true),

    ('Default up 15m',  'threshold', 'global',
     '15m', 'delta_oi_pct', '>=',  8.0,
     900, 1, 1.5, 5000000, 'good_estimate', ARRAY['snapshot','native_interval'], true),

    ('Default down 15m','threshold', 'global',
     '15m', 'delta_oi_pct', '<=', -8.0,
     900, 1, 1.5, 5000000, 'good_estimate', ARRAY['snapshot','native_interval'], true),

    ('Default up 30m',  'threshold', 'global',
     '30m', 'delta_oi_pct', '>=', 12.0,
     900, 1, 1.5, 5000000, 'good_estimate', ARRAY['snapshot','native_interval'], true),

    ('Default down 30m','threshold', 'global',
     '30m', 'delta_oi_pct', '<=', -12.0,
     900, 1, 1.5, 5000000, 'good_estimate', ARRAY['snapshot','native_interval'], true);
```

Эти 6 строк отражены в UI Settings как 6 числовых полей (up_5m, down_5m, ...). UI редактирует поле `threshold` соответствующей строки.

Дополнительные правила типов divergence / exchange-health пользователь добавляет вручную через DB или через расширенный UI (future work — в v1 минимум через CLI/API).

---

## 8. Evaluation cycle

### 8.1 Когда запускается

`evaluation_cycle_sec` (default 30) — alert engine evaluation запускается каждые 30 секунд.

Это **не привязано** к minute boundary. Цикл независим, чтобы быстро реагировать на свежие native_interval бары (которые могут прийти не ровно по минуте).

### 8.2 Что делает один цикл

```python
async def evaluation_cycle() -> None:
    cycle_start = monotonic()
    rules = await rules_repo.get_enabled()

    for rule in rules:
        signal_handler = registry[rule.rule_type]
        candidates = await signal_handler.evaluate(rule, ctx)

        for cand in candidates:
            state = await state_repo.get_or_create(rule.id, cand.exchange,
                                                    cand.canonical_symbol, cand.window)
            new_state = transition(state, cand, rule)
            decision = new_state._decision
            reason = new_state._reason

            await state_repo.upsert(new_state)
            await events_repo.append(AlertEvent(
                rule_id=rule.id,
                exchange=cand.exchange,
                canonical_symbol=cand.canonical_symbol,
                window=cand.window,
                ts_exchange=cand.current_bar.last_ts_exchange,
                ts_processed=now(),
                decision=decision,
                reason=reason,
                ...
            ))

            if decision == AlertDecision.FIRE:
                await delivery_producer.enqueue(rule, cand, new_state)

    cycle_duration = monotonic() - cycle_start
    metrics.alert_engine_cycle_duration.observe(cycle_duration)
```

### 8.3 Если цикл не уложился в 30 сек

- Скип следующего цикла (`cycle_overlap` лог).
- При двух подряд → WARNING + Prometheus alert `alert_engine_overload`.

---

## 9. Dedupe

### 9.1 dedupe_key формат

#### Standard (threshold/confirmed/divergence)

```
rule:<rule_id>|ex:<exchange>|sym:<canonical>|win:<window>|bucket:<ts_exchange_iso>
```

Пример:
```
rule:1|ex:binance|sym:BTC|win:5m|bucket:2026-04-30T12:15:00+00:00
```

#### Exchange-health (M3.E)

```
health:rule:<rule_id>|ex:<exchange>|trig:<sub_trigger>|bucket:<cooldown_bucket_iso>
```

Где `cooldown_bucket = floor(now / cooldown_sec) * cooldown_sec` — не CA
bucket, а cooldown window. Это даёт детерминированный ключ внутри одного
cooldown-окна, обеспечивая cooldown-as-dedupe вместо отдельной
`alert_state`-записи.

Пример (cooldown_sec=3600):
```
health:rule:8|ex:binance|trig:stale_data|bucket:2026-04-30T12:00:00+00:00
```

### 9.2 UNIQUE constraint в `delivery_queue`

```sql
CREATE UNIQUE INDEX idx_delivery_dedupe ON delivery_queue (dedupe_key);
```

При INSERT с дублем — `ON CONFLICT DO NOTHING`. Это защищает от:
- Двойного fire в один и тот же bucket из-за гонок.
- Двойной отправки при рестарте процесса между `transition` и `enqueue`.

### 9.3 Smart re-fire dedupe_key

При smart re-fire bucket уже другой (новый), поэтому `dedupe_key` отличается:
```
rule:1|...|bucket:2026-04-30T12:15:00 → first fire
rule:1|...|bucket:2026-04-30T12:20:00 → smart re-fire (new bucket)
```

Конфликта нет.

---

## 10. Audit log (`alert_events`)

### 10.1 Что пишется

В `alert_events` пишутся **только** decisions из persistance-whitelist (`00_DECISIONS_LOG.md F43`):

```python
AUDIT_PERSISTED_DECISIONS = frozenset({
    AlertDecision.FIRE,
    AlertDecision.SUPPRESS,
    AlertDecision.RESOLVE,
    AlertDecision.QUALITY_GATE_FAIL,
})
```

Остальные пять decisions (`late_data_skip`, `insufficient_history`, `below_min_oi`, `not_crossed`, `no_transition`) — **counter-only**: инкрементируется метрика `oi_alert_engine_decisions_total{rule_id, decision}` (см. `§14.1`), но строка в `alert_events` не создаётся.

**Принцип:** audit-таблица хранит факты, требующие ручного разбора. Counter — для частотных decisions, по которым достаточно агрегатной картины. State machine продолжает выдавать все 9 decisions — фильтрация только на стороне persistance в `evaluation_cycle._evaluate_rule`.

### 10.2 Зачем audit

- Дебаг: «почему не сработало?» (для FIRE / SUPPRESS / RESOLVE / QUALITY_GATE_FAIL).
- Аудит: «когда последний раз была сигнальная ситуация?»
- Анализ false-positive / false-negative.

> SLO `e2e_latency` снимается через histogram `oi_alert_e2e_latency_seconds` (см. `§14.1`), а не через `ts_processed - ts_exchange` в audit-таблице. Histogram сохраняет картину для всех FIRE-событий и для health-сигналов независимо от audit-persistance.

### 10.3 Объём

После F43 (whitelist) полный день: ~`fire + suppress + resolve + quality_gate_fail` ≈ 2 K строк/день (vs ~60 M строк/день до F43). После F44 (columnstore, `compress_after='7 days'`, симметрично F5/`oi_samples`): hot chunk uncompressed первые 7 дней, далее columnstore с `segmentby='rule_id, exchange'`. На 90 дней retention'а — десятки MB вместо ~2.7 TB при «всё в audit». См. `00_DECISIONS_LOG.md F43, F44`.

Counter `oi_alert_engine_decisions_total{rule_id, decision}` инкрементируется по **всем** decisions — observability сохранена в полном объёме без storage-цены.

---

## 11. Resolve логика (опциональная)

В нашей минимальной версии:
- **RESOLVE решение НЕ отправляется в TG** (чтобы не спамить).
- В `alert_events` пишется RESOLVE для аудита.
- В `alerts_sent` обновляется `resolved_at = now()` для последнего fire.

Это можно использовать в UI: «алерт BTC +5.2% (5m) → resolved через 23 минуты».

---

## 12. Failure modes

### 12.1 Alert engine падает

- Restart через systemd (auto).
- При старте: читаем `alert_state` (cooldown'ы и последние значения восстанавливаются).
- Возможен gap: между падением и рестартом мы не сделали fire по событию из этого периода.
- Это **приемлемо** — пользователь видит, что система была off (через exchange-health сигналы).

### 12.2 Storage недоступен

- evaluation_cycle падает с error.
- Retry в следующем cycle.
- Если > 3 циклов подряд — system-level Prometheus alert (через Alertmanager).

### 12.3 Delivery_queue растёт

Если TG sender тормозит:
- `delivery_queue` накапливается.
- Метрика `delivery_queue_pending_size` экспортируется.
- Алерт `delivery_queue_backlog > 100` → Prometheus alert.

---

## 13. Testing

### 13.1 Unit-тесты state machine

```python
@pytest.mark.parametrize("scenario", [
    {"name": "first_cross", "history": [3, 5.5], "expected": "fire"},
    {"name": "no_cross", "history": [5, 5.5], "expected": "no_fire"},  # already above, not crossing
    {"name": "back_to_armed", "history": [5.5, 4], "expected": "armed"},
    {"name": "smart_re_fire", "history": [5.5, 4, 5.5, 9], "expected": "fire (smart)"},
    ...
])
def test_state_machine(scenario):
    ...
```

### 13.2 Integration: full pipeline

```python
async def test_e2e_alert_flow():
    # 1. INSERT synthetic oi_samples с пересечением порога
    # 2. Refresh CA вручную
    # 3. Запустить evaluation_cycle
    # 4. Проверить, что alerts_sent содержит запись
    # 5. Проверить state machine в alert_state
```

### 13.3 Smart cooldown тесты

```python
async def test_smart_cooldown_re_fire():
    # 1. Fire +5.2%
    # 2. Через 5 минут (внутри cooldown 15 min) — Δ +6%, проверить SUPPRESS
    # 3. Ещё через 5 минут — Δ +9%, проверить FIRE
    # 4. Ещё через 5 минут — Δ +10%, проверить SUPPRESS (10 < 9*1.5=13.5)
```

### 13.4 No retro-fire

```python
async def test_no_retro_fire_after_replay():
    # 1. Стейт фиксирует cooldown_until = past_time
    # 2. Replay создаёт «исторический» FIRE-кандидат с ts_exchange < state.cooldown_until
    # 3. Проверить, что в delivery_queue ничего не появилось
```

---

## 14. Observability

### 14.1 Метрики

| Метрика | Тип | Labels |
|---|---|---|
| `oi_alert_engine_cycle_duration_seconds` | histogram | (без labels) |
| `oi_alert_engine_decisions_total` | counter | rule_id, decision |
| `oi_alert_engine_fires_total` | counter | rule_id, exchange, signal_type |
| `oi_alert_engine_state_transitions_total` | counter | from_state, to_state |
| `oi_alert_engine_smart_re_fires_total` | counter | rule_id |
| `oi_alert_e2e_latency_seconds` | histogram | rule_type |
| `oi_alert_quality_gate_fails_total` | counter | reason |
| `oi_alert_state_size` | gauge | (общее количество строк в alert_state) |

### 14.2 Grafana панели

- Decisions distribution (FIRE/SUPPRESS/...) per rule.
- E2E latency histogram (для SLO P95/P99).
- Top firing symbols / exchanges.
- Smart re-fire rate.

---

## 15. Cross-references

- Schemas → `05_DATA_CONTRACTS.md §6, §7, §8, §9`.
- Threshold formulas → `00_DECISIONS_LOG.md §4`.
- Quality gates definitions → `04_TIME_MODEL.md §6`.
- Storage queries → `08_TIME_SERIES_STORAGE.md §5`.
- TG templates → `10_DELIVERY_LAYER.md §3`.
- E2E SLO → `01_PRODUCT_SPEC.md NFR-1`, `12_OBSERVABILITY_SLO.md`.
- Test strategy → `14_TEST_STRATEGY.md §4`.
