# 10 · Delivery Layer

> Как алерты доставляются в Telegram, как UI получает данные. FastAPI REST + SSE, aiogram TG sender.
> Связано: `00_DECISIONS_LOG.md C8, C14, Q4, Q8`, `05_DATA_CONTRACTS.md §10`.

---

## 1. Обзор

Delivery Layer — это три параллельных канала доставки данных пользователю:

1. **Telegram sender** — push алертов в групповой чат.
2. **REST API** — pull-доступ к истории и текущему состоянию из UI.
3. **SSE** — real-time push в UI dashboard.

Все три читают из БД (PostgreSQL), не из коннекторов или alert engine напрямую.

---

## 2. Telegram sender

### 2.1 Назначение

Воркер `oi-tracker-tg-sender.service`, читающий `delivery_queue` и отправляющий сообщения через Telegram Bot API (через aiogram 3.x).

**Sender-only:** бот не обрабатывает входящие сообщения, не имеет команд, нет long polling. Он только шлёт.

### 2.2 Архитектура

```
                                    ┌──────────────────┐
                                    │  Telegram API    │
                                    └────────┬─────────┘
                                             ▲
                                             │ HTTPS
                                             │ aiogram.Bot.send_message()
                                             │
┌──────────────┐     ┌──────────────────────┴──────────────┐
│ delivery_    │     │      tg-sender process              │
│ queue (DB)   │◀────│  poll every 2 sec                   │
│              │     │  for status='pending'               │
│              │─────│  ↓                                  │
│ ON CONFLICT  │     │  for each: render template,         │
│ idx_delivery │     │  send to chat_id, mark sent         │
│ _dedupe      │     │  retries with exponential backoff   │
└──────────────┘     └─────────────────────────────────────┘
```

### 2.3 Polling loop

```python
async def tg_sender_loop():
    bot = aiogram.Bot(token=settings.TELEGRAM_BOT_TOKEN)
    chat_id = await settings_repo.get("tg_chat_id")
    dry_run = await settings_repo.get("tg_dry_run", default=False)

    while True:
        pending = await delivery_repo.fetch_pending(limit=10)
        if not pending:
            await asyncio.sleep(2.0)
            continue

        for delivery in pending:
            await _process_delivery(bot, chat_id, delivery, dry_run)

        # Не sleep если pending много (drain mode)
```

### 2.4 Per-delivery обработка

```python
async def _process_delivery(bot, chat_id, delivery, dry_run):
    try:
        # Mark as sending (pessimistic lock)
        await delivery_repo.mark_sending(delivery.id)

        # Render
        text = render_template(delivery.template, delivery.payload)

        if dry_run:
            logger.info("tg_dry_run", text=text, dedupe_key=delivery.dedupe_key)
            await delivery_repo.mark_sent(delivery.id)
            return

        # Send
        await bot.send_message(
            chat_id=chat_id,
            text=text,
            parse_mode="HTML",
            disable_web_page_preview=True,
        )

        await delivery_repo.mark_sent(delivery.id)
        metrics.tg_sent_total.labels(template=delivery.template).inc()

    except aiogram.exceptions.TelegramRetryAfter as e:
        # Telegram rate limit
        await delivery_repo.schedule_retry(delivery.id,
            delay_sec=e.retry_after + 1,
            error=f"retry_after_{e.retry_after}")
        metrics.tg_retry_after_total.inc()

    except aiogram.exceptions.TelegramAPIError as e:
        # Other API error
        if delivery.retry_count >= 5:
            await delivery_repo.mark_failed(delivery.id, error=str(e))
            metrics.tg_failed_total.inc()
        else:
            backoff = min(60 * (2 ** delivery.retry_count), 600)
            await delivery_repo.schedule_retry(delivery.id, delay_sec=backoff, error=str(e))
            metrics.tg_retry_total.inc()

    except Exception as e:
        logger.exception("tg_unexpected_error", delivery_id=delivery.id)
        await delivery_repo.schedule_retry(delivery.id, delay_sec=30, error=str(e))
```

### 2.5 Retry policy + outbound throttle

#### 2.5.1 Retry policy (reactive, on failure)

| retry_count | Delay before next try |
|---|---|
| 0 → 1 | 60s |
| 1 → 2 | 120s |
| 2 → 3 | 240s |
| 3 → 4 | 480s |
| 4 → 5 | 600s (cap) |
| ≥ 5 | mark_failed, не пытаемся больше |

Failed deliveries видны в UI Alerts log с пометкой "delivery failed".

#### 2.5.2 Outbound throttle (proactive, before send)

> ADR-status: **decision locked, implementation lands в M3.B**. См. `00_DECISIONS_LOG.md F22`.

**Проблема.** Telegram Bot API лимитирует:

- **Per-chat:** ~1 message / second к одному chat_id (документировано как «не более 1 msg/sec в группу»).
- **Global per bot:** ~30 messages / second совокупно (документация).
- При превышении → HTTP 429 с `parameters.retry_after` (секунды). Дальнейшие send'ы в окне retry — отклоняются.

В нашей системе одновременные threshold-fire'ы на нескольких биржах (12 бирж × 3 окна = до 36 алертов в худшем сценарии market-wide движения) могут породить burst, плюс retro-fire запрещён (`F9`) — всё надо доставить в реальном времени. Без throttle между «schedule_delivery → bot.send_message» получаем 429-каскад: первые ~30 проходят, остальные плодят retry-цепочки, которые забивают delivery_queue и create запаздывание per `Q3` SLO (≤ 90s P95 / ≤ 180s P99).

**Решение.** Leaky-bucket throttle на стороне `tg_sender_loop` ПЕРЕД каждым `bot.send_message`. Два независимых bucket'а:

| Bucket | Capacity | Refill rate | Что защищает |
|---|---|---|---|
| `tg_throttle_global` | 25 tokens | 25 tokens/sec | Bot-wide лимит TG (margin к 30 — оставляем headroom для transient bursts) |
| `tg_throttle_per_chat` | 1 token | 1 token/sec | Per-chat лимит для каждого `chat_id` |

Алгоритм: для каждой `delivery` дёргаем `await throttle.acquire(chat_id=delivery.chat_id, weight=1)` который требует 1 токен из глобального bucket И 1 токен из per-chat bucket; если хотя бы один пуст — sleep'ит до refill.

Реализация — `app/tg_sender/throttle.py`. Простая структура:

```python
class TgThrottle:
    def __init__(self, *, global_capacity=25, global_rate=25.0,
                 per_chat_capacity=1, per_chat_rate=1.0):
        self._global = LeakyBucket(global_capacity, global_rate)
        self._per_chat: dict[str, LeakyBucket] = {}
        self._per_chat_capacity = per_chat_capacity
        self._per_chat_rate = per_chat_rate

    async def acquire(self, *, chat_id: str, weight: int = 1) -> None:
        bucket = self._per_chat.setdefault(
            chat_id, LeakyBucket(self._per_chat_capacity, self._per_chat_rate))
        # Acquire global first; if blocked there, no point consuming per-chat.
        await self._global.acquire(weight)
        await bucket.acquire(weight)
```

`LeakyBucket` — re-use of existing `app/exchanges/_rate_limiter.py` (`RateLimiter`) — тот же алгоритм, переиспользуем не копируя.

**Метрики** (per `12_OBSERVABILITY_SLO`):

- `oi_tg_throttle_wait_seconds` (histogram, label `bucket=global|per_chat`) — сколько ждали токена. P95 > 0 означает, что мы регулярно упираемся в лимит.
- `oi_tg_throttle_acquire_total` (counter, label `bucket`) — сколько acquire прошло.

**Взаимодействие с retry policy (§2.5.1).** Если throttle всё-таки пропустит burst (e.g. из-за clock skew или TG rate window'а сдвинутого относительно нашего) и bot.send_message вернёт 429 с `retry_after=N` секунд:

1. Используем `retry_after` напрямую как next-delay (не из ladder §2.5.1) — TG явно говорит сколько ждать.
2. На время `retry_after` глобальный bucket «замораживается» — `throttle.freeze_global(seconds=retry_after)` блокирует acquire до окончания окна. Это предотвращает повторный 429 у параллельных delivery worker'ов.
3. retry_count не инкрементируется (это не failure delivery, а throttle event).

**Совместимость с ТЗ:**

- `Q3` Latency SLO: 25 msg/sec global × 90s window = 2250 deliveries в окне P95 — намного больше, чем worst-case burst на 36 алертов (12 бирж × 3 окна). Throttle не нарушает SLO даже при максимальном burst.
- `F11` source_kind: throttle не различает source — все алерты идут через один pipeline.
- `Q2` Signal types: 4 типа алертов (после `F40`) — все через один `delivery_queue` worker, throttle общий.

### 2.6 Routing model

> See `00_DECISIONS_LOG.md F37` (per-rule routing) + `F41` (per-exchange routing groups).

- **Token:** `TELEGRAM_BOT_TOKEN` env-переменная. В БД не хранится. Считывается на старте процесса. При смене — restart сервиса.
- **Chats registry:** `tg_chats` table (M7 / F37). CRUD через `/api/v1/chats`. Один помечен `is_default = true` — fallback для алертов без явного назначения.
- **Per-rule routing (F37):** `alert_rules.chat_id` (FK → `tg_chats`). NULL = use default. Explicit override на уровне правила.
- **Per-exchange routing (F41):** `tg_routing_groups` (имя, чат, optional `message_thread_id`, набор бирж через join `tg_routing_group_exchanges` с `UNIQUE(exchange)`). Каждая биржа — максимум в одной группе. CRUD через `/api/v1/routing-groups`.

**Resolver order** (`app/delivery/chat_resolver.py:resolve_route`):

```
rule.chat_id (explicit override, F37)
  → group covering candidate.exchange (F41)
  → default chat (no thread)
```

При degradation на любом шаге — fallback вниз по цепочке + метрика `oi_tg_chat_resolution_failed_total{reason}` (`reason ∈ {missing, inactive, group_inactive, group_chat_inactive, no_default}`). Если default тоже неюзабелен — `NoDefaultChatError`, producer skip'ает enqueue, `alert_events` upstream остаётся как audit trail.

**Snapshot policy.** Producer пишет `(chat_id_native, message_thread_id)` в `delivery_queue` на FIRE-time. Dispatcher шлёт ровно туда — даже если группа/чат переназначены между enqueue и send, retry попадает в исходный destination (детерминированно).

**Throttle.** `TgThrottle` keyed по `chat_id_native` (per-chat ведро 1 msg/sec) + global ведро (25 msg/sec). Threads НЕ имеют отдельного лимита у Telegram — несколько групп с одним чатом и разными threads делят одно per-chat ведро.

**Async-валидация thread_id (F41).** Telegram не предоставляет sync `getForumTopic` без админ-прав. Принимаем thread_id «как есть» при create/patch. На permanent-failure отправки (`TelegramBadRequest`/`TelegramForbiddenError`) dispatcher вызывает `routing_groups_repo.find_by_chat_native_thread(...)` и пишет ошибку в `last_send_error` группы. UI badge `send error` сигнализирует оператору о сломанном маршруте.

### 2.7 Health checks

- `oi_tg_sender_pending_size` (gauge) — сколько записей в `delivery_queue` со статусом `pending`.
- `oi_tg_sender_oldest_pending_age_seconds` — самая старая pending запись.
- Alertmanager rule: `oldest_pending > 300s` → Prometheus алерт «TG sender stuck».

---

## 3. Telegram templates

### 3.1 oi_threshold_cross

**Payload (см. `05_DATA_CONTRACTS.md §10.3`):**
```json
{
  "exchange": "binance",
  "canonical_symbol": "BTC",
  "native_symbol": "BTCUSDT",
  "window": "5m",
  "delta_pct": "6.2",
  "oi_notional_usdt": "1240000000",
  "oi_delta_abs_usdt": "76000000",
  "price": "67420.50",
  "price_change_pct": "0.4",
  "valuation_status": "authoritative"
}
```

`oi_delta_abs_usdt` — `current_bar.close_oi_notional_usdt - prev_bar.close_oi_notional_usdt`
в USDT, со знаком. Считается от тех же двух баров, что дают `delta_pct`,
поэтому знак % и абсолюта консистентен. Поле **опциональное**: producer
кладёт его только когда `AlertCandidate.prev_bar` известен (есть история
≥ окна). Renderer вставляет абсолют **inline** во 2-ю строку
(`±X.XX% (±$Y) за W`) только при наличии `oi_delta_abs_usdt`; иначе —
fallback на голую форму `±X.XX% за W`. Это сохраняет различие между
«нет данных» (cold-start) и «реально 0» и не подсовывает пользователю
фальшивый `+0.00% ($0)` (`F50`, replaces `F49`).

**Render:**
```
<b>{symbol}</b> @ <i>{exchange}</i>
{sign}{delta_pct}% ({signed_abs}) за {window}   ← скобки только при наличии oi_delta_abs_usdt
OI: ${oi_humanized}   Цена: ${price_humanized}
```

**Output:**
```
BTC @ Binance
+6.20% (+$76.00M) за 5мин
OI: $1.24B   Цена: $67,420
```

**Output (cold-start, нет `oi_delta_abs_usdt`):**
```
BTC @ Binance
+6.20% за 5мин
OI: $1.24B   Цена: $67,420
```

### 3.2 oi_threshold_cross_smart_refire

**Render:**
```
<b>BTC</b> @ <i>Binance</i> ⚡
Ускорение: {prev_delta}% → {delta_pct}% ({window})
OI: ${oi_humanized}   Цена: ${price_humanized}
```

### 3.2.5 volume_threshold_cross (F51, Aster-only)

Aster в окне 5m альертит по торговому объёму, не OI (см. `00_DECISIONS_LOG F51` и `09_ALERT_ENGINE §5.6`). Шаблон — twin `oi_threshold_cross` с двумя отличиями: 2-я строка получает trailer `· vol`, 3-я строка — `Vol W:` вместо `OI:`.

**Required payload keys:** `exchange`, `canonical_symbol`, `window`, `delta_pct`, `quote_volume_usdt`, `price`.
**Optional keys:** `volume_delta_abs_usdt` (signed USDT абсолют volume Δ за окно; producer омитит при `prev_bar is None`, renderer fallback'ит на bare form).
**Informational:** `valuation_status`, `threshold`, `rule_name`.

**Render (с known prev bucket):**
```
<b>LINK</b> @ <i>Aster</i>
+12.50% (+$1.20M) за 5мин · vol
Vol 5m: $2.10M   Цена: $9.6530
```

**Render (cold start, prev_bar=None):**
```
<b>LINK</b> @ <i>Aster</i>
+12.50% за 5мин · vol
Vol 5m: $2.10M   Цена: $9.6530
```

**Known limitation:** `price` сейчас = `"0"` (volume cycle не запрашивает mark price; рендер показывает `$0`). PR3 enrichment'нёт payload последней `oi_samples.price_used` для символа.

### 3.3 ~~consensus_alert~~ (sunset, F40)

Sunset per `00_DECISIONS_LOG F40` (M8 Phase A). Template никогда не был залит в `tg_sender/templates.py`, signal handler удалён. Один алерт = одна биржа = одно сообщение.

### 3.4 divergence_alert

**Render:**
```
↗️↘️ <b>{symbol}</b> @ <i>{exchange}</i>
{type_emoji} OI {oi_delta_pct}% / Цена {price_delta_pct}% за {window}
```

**Output:**
```
↗️↘️ BTC @ Binance
🚀📉 OI +8.1% / Цена -2.3% за 15мин
```

### 3.5 exchange_health_alert

**Render:**
```
⚠️ <b>{exchange}</b>: {issue_human}
{details}
```

**Output:**
```
⚠️ KuCoin: Stale data
Нет новых точек 7 минут (затронуто 312 символов)
```

### 3.6 Humanizer для USD

```python
def humanize_usd(value: Decimal) -> str:
    """1240000000 → '1.24B'; 5300000 → '5.30M'; 12500 → '12.5K'"""
    abs_val = abs(value)
    if abs_val >= 1_000_000_000:
        return f"{value / 1_000_000_000:.2f}B"
    if abs_val >= 1_000_000:
        return f"{value / 1_000_000:.2f}M"
    if abs_val >= 1_000:
        return f"{value / 1_000:.1f}K"
    return f"{value:.2f}"
```

### 3.7 Языковая локализация

В v1 — только русский для UI-friendly строк ("за 5мин"), английский для имён ("Binance"). Локализация через простые dict'ы в `tg_templates.py`. Future work — i18n.

---

## 4. FastAPI REST API

### 4.1 Назначение

Внутренний API для UI. **Не публичный**, без аутентификации (защита на сетевом уровне через VPN/firewall, см. `02_ARCHITECTURE.md §6`).

### 4.2 Endpoint group

```
/api/v1/
├── /live                          # текущий снимок dashboard
├── /symbols/<canonical>           # история одного символа
├── /alerts                        # журнал alerts_sent
├── /alert-events                  # журнал alert_events (дебаг)
├── /alert-rules                   # CRUD правил
├── /settings                      # KV settings
├── /instruments                   # список инструментов
├── /chats                         # CRUD Telegram чатов (F37)
├── /routing-groups                # CRUD per-exchange routing групп (F41)
├── /health                        # liveness + readiness
└── /metrics                       # prometheus_client export (отдельный port)

/sse/v1/
├── /live                          # SSE для dashboard
└── /alerts                        # SSE для alerts log
```

### 4.3 GET /api/v1/live

**Query params:**
- `exchange?: str` — фильтр.
- `min_oi_usdt?: number` — фильтр.
- `sort?: "delta_5m"|"delta_15m"|"delta_30m"|"oi"|"symbol"` — сортировка.
- `limit?: int = 1000`.

**Response:**
```json
{
  "data": [
    {
      "exchange": "binance",
      "canonical_symbol": "BTC",
      "native_symbol": "BTCUSDT",
      "ts_exchange": "2026-04-30T12:15:00Z",
      "oi_coins": "12345.678",
      "oi_notional_usdt": "832500000.00",
      "price": "67420.50",
      "delta_5m_pct": "0.42",
      "delta_15m_pct": "1.21",
      "delta_30m_pct": "2.05",
      "valuation_status": "authoritative",
      "source_kind": "snapshot"
    },
    ...
  ],
  "meta": {
    "total": 3590,
    "fetched_at": "2026-04-30T12:15:23Z"
  }
}
```

### 4.4 GET /api/v1/symbols/{canonical_symbol}

**Path:** `canonical_symbol` (e.g. `"BTC"`).

**Query params:**
- `period: "1h"|"6h"|"24h"|"7d"|"30d"|"90d"` (default `"24h"`).
- `timeframe: "auto"|"1m"|"5m"|"15m"|"30m"|"1h"|"4h"|"1d"` (default `"auto"`).
  При `"auto"` bucket подбирается ladder-декимацией под `target_points` (см. ниже). При явном таймфрейме `target_points` игнорируется; `5m/15m/30m` читаются напрямую из continuous aggregates `oi_5m/oi_15m/oi_30m`, остальные — из raw `oi_samples` через `time_bucket()`. См. `08 §5.5`.
- `exchanges?: str` — comma-separated список бирж, фильтр (пусто = все).
- `target_points: int` (100..5000, default 1000) — целевое число точек, актуально только при `timeframe="auto"`.

**Response:**
```json
{
  "canonical_symbol": "BTC",
  "period_hours": 24,
  "timeframe": "auto",
  "bucket_interval": "5 minutes",
  "since": "2026-04-29T12:00:00+00:00",
  "until": "2026-04-30T12:00:00+00:00",
  "data": [
    {
      "bucket": "2026-04-29T12:00:00+00:00",
      "exchange": "binance",
      "oi_coins": "12345.678901234567890123",
      "oi_notional_usdt": "832000000.00000000",
      "price_used": "67380.000000000000000000",
      "valuation_status": "authoritative",
      "has_degraded_source": false
    },
    {
      "bucket": "2026-04-29T12:00:00+00:00",
      "exchange": "bybit",
      "oi_coins": "9876.543210987654321098",
      "oi_notional_usdt": "665000000.00000000",
      "price_used": "67375.000000000000000000",
      "valuation_status": "authoritative",
      "has_degraded_source": false
    }
  ]
}
```

Поля row:
- `bucket: str` — ISO-8601 timestamptz, начало интервала.
- `exchange: str` — каноническое имя биржи (lowercase, см. `03_GLOSSARY`).
- `oi_coins: str | null` — Decimal как string (`NUMERIC(38,18)`, C9), `null` если в bucket нет данных.
- `oi_notional_usdt: str | null` — Decimal как string (`NUMERIC(38,8)`).
- `price_used: str | null` — Decimal как string.
- `valuation_status: "authoritative" | "good_estimate" | "low_confidence" | null`.
- `has_degraded_source: bool` — `true`, если в bucket попала точка `source_kind = 'degraded_fallback'`.

Поля envelope:
- `canonical_symbol: str` — echo path-параметра.
- `period_hours: int` — окно в часах (1 / 6 / 24 / 168 / 720 / 2160).
- `timeframe: str` — echo query-параметра (`auto` или явный grain).
- `bucket_interval: str` — фактический шаг bucket'а (`"1 minute"`, `"5 minutes"`, ..., `"1 day"`).
- `since`, `until: str` — ISO-8601 timestamptz границ окна.

**Decimation:** при `timeframe="auto"` (default) и `period > 1h` сервер автоматически выбирает bucket size так, чтобы вернуть ~`target_points` точек на ряд (см. `08_TIME_SERIES_STORAGE.md §5.5`). При явном таймфрейме декимации нет — используется фиксированный шаг (CA для `5m/15m/30m`, raw + `time_bucket` для остальных).

### 4.5 GET /api/v1/alerts

**Query params:**
- `since?: ISO8601` (default: now() − 24h).
- `until?: ISO8601`.
- `exchange?: str`.
- `symbol?: str`.
- `window?: "5m"|"15m"|"30m"`.
- `decision?: "fire"|"resolve"|...`.
- `limit?: int = 100`.
- `offset?: int = 0`.

**Response:**
```json
{
  "data": [
    {
      "alert_event_id": 12345,
      "rule_id": 1,
      "rule_name": "Default up 5m",
      "exchange": "binance",
      "canonical_symbol": "BTC",
      "window": "5m",
      "ts_exchange": "2026-04-30T12:15:00Z",
      "ts_processed": "2026-04-30T12:15:23Z",
      "decision": "fire",
      "delta_pct": "5.23",
      "oi_now_notional_usdt": "832500000.00",
      "valuation_status": "authoritative",
      "tg_delivery_status": "sent",
      "tg_sent_at": "2026-04-30T12:15:25Z"
    },
    ...
  ],
  "meta": {
    "total": 47,
    "limit": 100,
    "offset": 0
  }
}
```

### 4.6 GET /api/v1/alert-rules

Полный список правил с возможностью фильтрации по `enabled`, `rule_type`.

### 4.7 PUT /api/v1/alert-rules/{id}

Обновление правила. Body — поля `AlertRule`. Особое внимание:
- При смене `threshold` для default правил — изменение сразу вступает в силу для следующего evaluation cycle.
- При смене `enabled = false` — все state transitions для этого правила прекращаются; `alert_state` строки не удаляются (для возможного восстановления).

### 4.8 GET /api/v1/settings + PUT /api/v1/settings/{key}

Чтение/обновление KV-настроек.

**Доступные ключи (полный список — см. `05_DATA_CONTRACTS.md §12.3`).** Категоризация по уровню «безопасности» правки из UI:

#### Категория A — UI первого уровня (Settings page, всегда видимы)
- `tg_chat_id` (string) — id Telegram группового чата.
- `tg_dry_run` (bool) — режим без отправки в TG.
- `exchange_whitelist` (list[str]) — какие биржи активны.
- `symbol_blacklist` (list[str]) — какие символы игнорируются.
- `min_oi_notional_default_usdt` (number ≥ 0) — глобальный floor-фильтр (см. `00_DECISIONS_LOG.md C4 + F5`).

#### Категория B — UI «Advanced settings» (свернуто по умолчанию, но доступно)
- `cooldown_default_sec` (int, range `[60, 86400]`) — default cooldown для новых rules.
- `smart_cooldown_factor` (number, range `[1.0, 5.0]`) — коэффициент re-fire (см. `Q1 + F7`).
- `freshness_last_point_factor` (number, range `[1.0, 4.0]`) — множитель `poll_interval` для «свежести последней точки» (см. `C7`). Default 2.0.
- `freshness_t_minus_n_factor` (number, range `[1.0, 3.0]`) — множитель для tolerance при поиске t-N (см. `C7`). Default 1.5.
- `freshness_native_grace_sec` (int, range `[10, 300]`) — grace-period для native interval. Default 60.
- `min_history_minutes_floor` (int, range `[10, 240]`) — floor для `min_history_minutes` (см. `F8`, default 30).

#### Категория C — НЕ вынесены в UI (требуют code/migration changes)
- `polling_interval_sec` — менять без полного передеплоя scheduler нельзя (нарушит `C2`).
- `instruments_sync_interval_sec` — то же.
- `evaluation_cycle_sec` — затрагивает alert engine timing.
- `chunk_time_interval`, `compression_after_days`, `retention_days` — TimescaleDB policies, меняются миграциями.
- `connector_version`, `normalization_version` — code-level версии.

#### Валидация
Все правки идут через pydantic-модель `SettingUpdate` (`05_DATA_CONTRACTS.md §12`), которая enforces типы и диапазоны. Невалидное значение → `400 Bad Request` с указанием поля.

#### Rate-limit
PUT /settings/{key} — `5 запросов в минуту` per IP. Реализуется через `slowapi` или эквивалент. Single-user сценарий, но это boundary input + защита от случайной кнопки в UI, отправляющей storm.

429 ответ:
```json
{ "detail": "Too many requests", "retry_after_sec": 12 }
```

#### Audit
Каждый PUT /settings/{key} логируется в `settings_audit_log` (DDL — `05_DATA_CONTRACTS.md §12.4`):
```sql
CREATE TABLE settings_audit_log (
    id          BIGSERIAL PRIMARY KEY,
    key         TEXT NOT NULL,
    old_value   JSONB,
    new_value   JSONB,
    changed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    changed_by  TEXT NOT NULL DEFAULT 'ui'   -- 'ui' / 'cli' / 'migration'
);
```

UI отображает последние 50 записей в Settings page → tab "History".

### 4.9 GET /api/v1/instruments

Поиск инструментов. Query params:
- `exchange?`, `canonical_symbol?`, `delisted?`, `blacklisted?`.

### 4.10 POST /api/v1/instruments/{id}/blacklist

Body: `{"value": true}`. Помечает инструмент `blacklisted = value`.

### 4.11 Routing groups CRUD (F41)

`/api/v1/routing-groups` — управление per-exchange группами. Rate-limit `5/min/IP` (F15). Audit-log пишется на каждое мутирующее действие (`settings_audit_log` ключ `routing_group:{action}:{id}`).

#### GET /api/v1/routing-groups

**Response:**
```json
{
  "data": [
    {
      "id": 1,
      "name": "tier1",
      "chat_id": 5,
      "message_thread_id": 142,
      "is_active": true,
      "exchanges": ["binance", "bybit", "okx"],
      "last_send_error": null,
      "last_send_error_at": null,
      "created_at": "2026-05-04T12:00:00+00:00",
      "updated_at": "2026-05-04T12:00:00+00:00"
    }
  ]
}
```

#### POST /api/v1/routing-groups

**Body:**
```json
{
  "name": "tier1",
  "chat_id": 5,
  "message_thread_id": 142,
  "exchanges": ["binance", "bybit"],
  "is_active": true
}
```

**Errors:**
- `409` — `name` UNIQUE conflict.
- `422 {"error": "exchange_in_other_group", "exchanges": [...]}` — одна или несколько бирж уже привязаны к другой группе.
- `422` — `chat_id` не существует в `tg_chats`.

#### PATCH /api/v1/routing-groups/{id}

Partial update. Поля `name`, `chat_id`, `message_thread_id`, `is_active`, `exchanges` — все опциональные.

`message_thread_id` — tri-state:
- omitted → не меняется.
- `int` value → set.
- explicit `null` → clear (no thread).

`exchanges` — replace-семантика: передан = полная замена набора (атомарно через transaction); опущен = не трогается.

Те же ошибки, что в POST.

#### DELETE /api/v1/routing-groups/{id}

Hard-delete + cascade на связанные `tg_routing_group_exchanges`. `204 No Content` на успех; `404` если не найдена.

### 4.12 GET /api/v1/health

```json
{
  "status": "ok",
  "components": {
    "db": "ok",
    "scheduler": "ok",
    "tg_sender": "ok",
    "exchanges": {
      "binance": {"status": "ok", "last_sample_age_sec": 23},
      "bybit":   {"status": "ok", "last_sample_age_sec": 18},
      "kucoin":  {"status": "degraded", "last_sample_age_sec": 312, "reason": "stale_data"},
      ...
    }
  },
  "version": "1.2.0",
  "ts": "2026-04-30T12:15:00Z"
}
```

### 4.13 OpenAPI spec

FastAPI авто-генерирует `/api/v1/openapi.json` и Swagger UI на `/api/v1/docs`. Используется для:
- Codegen TypeScript клиента в `frontend/src/api/`.
- Документации.

---

## 5. SSE (Server-Sent Events)

### 5.1 Назначение

Real-time push в UI без polling. Поддерживается всеми современными браузерами через `EventSource` API.

### 5.2 Endpoints

#### GET /sse/v1/live

Push дельт `live` таблицы при появлении новых точек.

**Event types:**
- `sample` — новая нормализованная точка.
- `delta_update` — пересчитанная Δ для пары.
- `connection_init` — отправляется при подключении (fresh snapshot).
- `heartbeat` — каждые 30 сек, чтобы предотвратить idle disconnect.

**Event format (SSE):**

`sample` payload === one row of `GET /api/v1/live` (см. §4.3) — listener
обогащает keys-only `pg_notify` payload (`F28`) полной формой через
`live_dashboard.query_one_with_deltas()` перед публикацией в Hub.
Frontend потребляет одну схему строки на оба входа (REST snapshot и
SSE delta).

```
event: sample
data: {
  "exchange":"binance","canonical_symbol":"BTC","native_symbol":"BTCUSDT",
  "ts_exchange":"2026-04-30T12:15:00.000Z",
  "oi_coins":"103823.706000000000000000",
  "oi_notional_usdt":"8123675167.08111229",
  "last_price":"78244.896855070000000000",
  "valuation_status":"good_estimate",
  "source_kind":"snapshot",
  "sample_age_seconds":41.17,
  "delta_5m_pct":"0.42","delta_15m_pct":"1.21","delta_30m_pct":"2.05"
}

event: heartbeat
data: {"ts":"2026-04-30T12:15:00Z"}
```

#### GET /sse/v1/alerts

Push новых alert_events с decision=fire.

**Event types:**
- `alert_fired` — новый алерт.
- `alert_resolved` — фиксация resolve (опционально).
- `heartbeat`.

### 5.3 Backend реализация

```python
@router.get("/sse/v1/live")
async def sse_live(request: Request):
    queue = asyncio.Queue(maxsize=1000)
    sse_hub.subscribe(queue, channel="live")

    async def event_stream():
        # Initial snapshot
        snapshot = await live_repo.get_current_dashboard()
        yield f"event: connection_init\ndata: {json.dumps(snapshot)}\n\n"

        try:
            while True:
                if await request.is_disconnected():
                    break
                try:
                    event = await asyncio.wait_for(queue.get(), timeout=30)
                    yield f"event: {event.type}\ndata: {json.dumps(event.data)}\n\n"
                except asyncio.TimeoutError:
                    yield f"event: heartbeat\ndata: {{\"ts\":\"{utcnow().isoformat()}\"}}\n\n"
        finally:
            sse_hub.unsubscribe(queue, channel="live")

    return StreamingResponse(event_stream(), media_type="text/event-stream")
```

### 5.4 SSE Hub

In-process pub/sub для `live` и `alerts` событий.

```python
class SSEHub:
    def __init__(self):
        self._subscribers: dict[str, list[asyncio.Queue]] = defaultdict(list)

    def subscribe(self, queue, channel: str):
        self._subscribers[channel].append(queue)

    def unsubscribe(self, queue, channel: str):
        self._subscribers[channel].remove(queue)

    async def publish(self, channel: str, event):
        for queue in list(self._subscribers[channel]):
            try:
                queue.put_nowait(event)
            except asyncio.QueueFull:
                # Drop event for slow consumer
                logger.warn("sse_queue_full", channel=channel)
```

### 5.5 Источник событий: PostgreSQL NOTIFY

Чтобы SSE Hub узнавал о новых точках, используем PG NOTIFY/LISTEN:

```sql
-- Триггер на oi_samples insert
CREATE FUNCTION notify_new_oi_sample() RETURNS trigger AS $$
BEGIN
    PERFORM pg_notify('oi_new_sample',
        json_build_object(
            'exchange', NEW.exchange,
            'canonical_symbol', NEW.canonical_symbol,
            'ts_exchange', NEW.ts_exchange
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_notify_new_oi_sample
AFTER INSERT ON oi_samples
FOR EACH ROW EXECUTE FUNCTION notify_new_oi_sample();

-- Аналогично для alert_events (decision='fire' через WHEN-clause)
CREATE FUNCTION notify_new_alert() RETURNS trigger AS $$
BEGIN
    PERFORM pg_notify('oi_new_alert',
        json_build_object(
            'rule_id', NEW.rule_id,
            'exchange', NEW.exchange,
            'canonical_symbol', NEW.canonical_symbol,
            'ts_processed', NEW.ts_processed
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_notify_new_alert
AFTER INSERT ON alert_events
FOR EACH ROW WHEN (NEW.decision = 'fire')
EXECUTE FUNCTION notify_new_alert();
```

API процесс держит постоянное `LISTEN oi_new_sample / oi_new_alert`
соединение и форвардит в SSE Hub. Это:
- Без необходимости Redis/RabbitMQ.
- Низкая латентность (~1ms).
- Нет polling БД из API.

#### 5.5.1 Keys-only payload policy (`F28`)

Payload `pg_notify` лимитирован 8000 байтами; PostgreSQL поднимает
`ERROR: payload string too long` при превышении (silent truncation
исключена, но ошибка прерывает INSERT, что недопустимо в hot path
oi_samples).

**Правило:** payload содержит **только идентификаторы**:
- `oi_new_sample`: `exchange`, `canonical_symbol`, `ts_exchange`.
- `oi_new_alert`: `rule_id`, `exchange`, `canonical_symbol`, `ts_processed`.

Полная строка (`oi_notional_usdt`, `valuation_status`, `payload` JSON и пр.)
fetched API-процессом отдельным `SELECT` после приёма NOTIFY. На малом
fan-out'е API процессов (1) batching не нужен — round-trip <1ms.

**Расширение запрещено** без новой F-записи. Кандидаты обязаны:
1. Показать, что 99-й перцентиль payload <4 KB.
2. Бенчмарком показать, что добавление полей экономит >1 SELECT в hot
   path (иначе экономия не оправдана).
3. Зафиксировать обновление этого ADR'а как `F<n>`.

См. также `02 §5.3` (architecture-level note) и `00_DECISIONS_LOG F28`.

### 5.5.2 No event replay on reconnect (`F27`)

При reconnect клиент **НЕ получает** события, пропущенные во время разрыва.
Контракт восстановления свежести:

1. Клиент рестартует EventSource с экспоненциальным backoff (1s → 2s →
   4s → ... → 30s cap, см. §5.6).
2. На успешном reconnect сервер отправляет `event: connection_init` со
   полным snapshot'ом — клиент пересобирает состояние одним consistent
   read'ом, не пытается merge'ить пропуски.
3. Промежуточные события (`sample`, `alert_fired`) во время offline
   потеряны. Это допустимо:
   - **Алерты** имеют каноническую копию в Telegram (с retry до 5
     попыток, throttled `F22`) — пользователь не пропускает FIRE.
   - **Live OI** перепишется следующим polling cycle (≤60s).
4. Сервер дропает события для slow consumer'ов через `asyncio.QueueFull`
   и логирует `sse_queue_full` (см. §5.4) — операционный сигнал, не
   функциональный баг. Метрика `oi_sse_queue_full_total` (M4.B.12)
   тригерит alert при >0 за 5 минут.

**Что НЕ делаем:**
- Не устанавливаем `id:` в SSE payload — клиент не сможет резюмировать
  через `Last-Event-ID`.
- На сервере игнорируем входящий `Last-Event-ID` header — нет журнала
  событий для replay.
- Не пытаемся merge'ить gap'ы на frontend — `connection_init` snapshot
  достаточен.

**UI обязательства** (`18_DESIGN_SYSTEM §7.6 ConnectionStatusBadge`):
- `live`: SSE connected, последнее событие <5s. Pulse animation.
- `reconnecting`: backoff активен. text-warn, без pulse.
- `offline`: нет SSE >30s. text-bear, иконка ⚠.

Без видимого badge пользователь не отличит «нет рыночных движений» от
«соединение умерло 2 минуты назад» — нарушение observability контракта.

См. `00_DECISIONS_LOG F27`.

### 5.6 Frontend реализация

```typescript
// hooks/useSSE.ts
export function useSSE(url: string, onEvent: (e: SSEEvent) => void) {
    const [connected, setConnected] = useState(false);

    useEffect(() => {
        let es: EventSource | null = null;
        let reconnectDelay = 1000;
        const maxDelay = 30000;

        const connect = () => {
            es = new EventSource(url);
            es.onopen = () => {
                setConnected(true);
                reconnectDelay = 1000;
            };
            es.addEventListener("connection_init", (e) => onEvent({ type: "init", data: JSON.parse(e.data) }));
            es.addEventListener("sample", (e) => onEvent({ type: "sample", data: JSON.parse(e.data) }));
            es.addEventListener("alert_fired", (e) => onEvent({ type: "alert", data: JSON.parse(e.data) }));
            es.addEventListener("heartbeat", (e) => { /* tick */ });

            es.onerror = () => {
                setConnected(false);
                es?.close();
                setTimeout(connect, reconnectDelay);
                reconnectDelay = Math.min(reconnectDelay * 2, maxDelay);
            };
        };

        connect();
        return () => { es?.close(); };
    }, [url, onEvent]);

    return { connected };
}
```

### 5.7 SSE против WebSocket

Выбран SSE из-за:
- Однонаправленность (server → client) — нам не нужен upstream.
- Native HTTP, проще через nginx.
- Auto-reconnect в браузере (с нашим backoff).
- Не требует socket upgrade.

WebSocket — overkill для single-user приложения.

---

## 6. Frontend (React + TS)

> **Визуальный язык, токены, типографика, motion, per-page UX-требования** — см. `18_DESIGN_SYSTEM.md`. Этот раздел описывает структуру страниц и data flow, не визуал.

### 6.1 Pages

#### Dashboard (`/`)

**Содержимое:**
- Live-таблица: `exchange | canonical_symbol | OI ($) | Δ5m | Δ15m | Δ30m | price`.
- Сортировка по любой колонке (TanStack Table).
- Фильтры: exchange (multi-select), min_oi_usdt (slider), search by symbol.
- Цветовая подсветка Δ: green (positive), red (negative), grey (null).
- Status indicator: connected (SSE) / disconnected.
- "Last update: N seconds ago" badge.

**Data sources:**
- Initial: `GET /api/v1/live` (snapshot).
- Live: `EventSource /sse/v1/live` (delta updates).

#### Symbol page (`/symbol/:canonical`)

**Содержимое:**
- График: 12 линий (по биржам), Y = OI (USDT), X = время.
- Period selector: 1h / 6h / 24h / 7d / 30d / 90d.
- Toggle линий (legend).
- Сводная таблица: для каждой биржи — текущее OI, Δ5m, Δ15m, Δ30m.
- Метаданные инструмента (contract_size, instrument_version, list of exchanges).

**Data sources:**
- `GET /api/v1/symbols/:canonical?period=24h`.

**Decimation:** сервер сам решает bucket_size (см. §4.4); UI просто рендерит.

#### Alerts log (`/alerts`)

**Содержимое:**
- Таблица: `time | exchange | symbol | window | rule_name | Δ | decision | delivery_status`.
- Фильтры: time range, exchange, symbol, window, decision.
- Pagination.
- "Live mode" toggle: SSE подписка на новые алерты.

**Data sources:**
- `GET /api/v1/alerts?...`.
- Live: `EventSource /sse/v1/alerts`.

#### Settings (`/settings`)

Таб-структура страницы:

##### Tab 1 — Alerts (default open)
- 6 default-порогов в виде form (up_5m, down_5m, ..., down_30m) с slider'ами (0.1% step).
  Сохранение → PUT /api/v1/alert-rules/:id.
- `min_oi_notional_default_usdt` — slider (`100k → 100M USDT`, log-scale).
- `cooldown_default_sec` — slider (`60s → 24h`).
- `smart_cooldown_factor` — slider (`1.0 → 5.0`, step 0.1).

##### Tab 2 — Filters
- Exchange whitelist — toggle chips (12 чипов, по одному на биржу).
- Symbol blacklist — text input (comma-separated, autocomplete по `instruments`).
- "Force re-sync instruments" button per exchange (POST /api/v1/instruments/sync с `exchange` body).

##### Tab 3 — Telegram
- **Чаты** (M7 / F37): таблица `tg_chats` с действиями: «Добавить чат» (+sync getChat валидация), «Назначить по умолчанию», «Тест отправки», «Деактивировать». Бот должен быть добавлен в чат заранее.
- **Маршрутизация по биржам** (M8 / F41): список `tg_routing_groups`. Карточка группы: name input, chat picker (select из активных tg_chats), thread input (numeric, optional), exchange chip-multiselect (12 чипов; занятые другими группами — disabled с tooltip). Кнопки Save/Активна/Удалить. Badge `send error` если group ловила permanent failure отправки. Выводится подсказка «Биржи без группы → default chat».
- **Режим:** `tg_dry_run` toggle.
- Bot token хранится только в env (не в UI).

##### Tab 4 — Advanced (свёрнуто по умолчанию, warning «менять только если знаете зачем»)
- `freshness_last_point_factor` — slider (`1.0 → 4.0`).
- `freshness_t_minus_n_factor` — slider (`1.0 → 3.0`).
- `freshness_native_grace_sec` — slider (`10s → 300s`).
- `min_history_minutes_floor` — slider (`10 → 240` мин).

##### Tab 5 — History
- Последние 50 записей `settings_audit_log` (см. §4.8 Audit).
- Колонки: `changed_at`, `key`, `old_value`, `new_value`, `changed_by`.

##### Validation UX
- Все ползунки и input'ы валидируют клиентски (диапазон из §4.8 категория B).
- При server `400` (валидация на бэке) — toast с указанием поля.
- При `429` (rate-limit) — toast «слишком много правок, попробуйте через `retry_after_sec`».

### 6.2 Component structure

```
src/
├── api/
│   ├── client.ts           # axios/fetch wrapper, error handling
│   ├── types.ts            # generated from OpenAPI
│   └── sse.ts              # EventSource wrapper
├── pages/
│   ├── Dashboard.tsx
│   ├── Symbol.tsx
│   ├── Alerts.tsx
│   └── Settings.tsx
├── components/
│   ├── LiveTable.tsx       # TanStack Table
│   ├── OIChart.tsx         # Recharts wrapper
│   ├── DeltaCell.tsx       # цветной Δ
│   ├── ExchangeBadge.tsx
│   ├── FreshnessIndicator.tsx
│   └── ConnectionStatus.tsx
├── hooks/
│   ├── useSSE.ts
│   ├── useApi.ts           # типизированный fetch
│   └── useDebounce.ts
└── styles/
    └── tailwind.config.js
```

### 6.3 Routing

```typescript
// App.tsx
<BrowserRouter>
  <Routes>
    <Route path="/" element={<Dashboard />} />
    <Route path="/symbol/:canonical" element={<Symbol />} />
    <Route path="/alerts" element={<Alerts />} />
    <Route path="/settings" element={<Settings />} />
  </Routes>
</BrowserRouter>
```

### 6.4 State management

В v1 — `useState` + `useReducer` локально, без global store. Для 4 страниц single-user сложный state-management не нужен.

При необходимости (например, общий filters state) — `Context` API.

### 6.5 Performance

- Live-таблица @ 3000+ строк → virtualization через TanStack Table `useVirtualizer`.
- График @ 1000 точек × 12 линий → Recharts с `dot={false}` и debounced re-render.
- SSE event throttling: max 1 redraw per 200ms (не дёргать React при каждом event).

---

## 7. nginx config

> **Источник истины — `15_DEPLOYMENT_INFRA.md §5`.** Там полный production-конфиг с TLS, `X-Robots-Tag`, security headers, UDS upstream, SSE-friendly buffering off.

Ключевые фрагменты, важные именно для delivery:

- **Upstream:** `unix:/run/oi-tracker/api.sock` (Unix socket, не TCP — порт 8000 занят соседом, см. `00_DECISIONS_LOG.md F14`).
- **`/sse/` location:** `proxy_buffering off; proxy_cache off; proxy_read_timeout 24h; chunked_transfer_encoding on;` — SSE требует disable buffering на всех уровнях, иначе события доходят пачками после задержки.
- **`/api/` location:** standard reverse-proxy с `proxy_set_header Host`, `X-Forwarded-Proto`.
- **server_name:** `oi-tracker.robot-detector.ru` (`F12`).
- **Headers:** `X-Robots-Tag: noindex, nofollow, noarchive, nosnippet always` (`F13`).

---

## 8. Observability

### 8.1 Метрики Delivery Layer

| Метрика | Тип | Labels |
|---|---|---|
| `oi_tg_sent_total` | counter | template |
| `oi_tg_failed_total` | counter | template, reason |
| `oi_tg_retry_total` | counter | template |
| `oi_tg_pending_size` | gauge | — |
| `oi_tg_oldest_pending_age_seconds` | gauge | — |
| `oi_api_requests_total` | counter | endpoint, method, status_code |
| `oi_api_request_duration_seconds` | histogram | endpoint |
| `oi_sse_connections` | gauge | channel |
| `oi_sse_events_published_total` | counter | channel, event_type |

### 8.2 Логирование

Все три процесса (TG sender, API) логируют в structured JSON через `structlog`, агрегируется Loki/promtail.

Полезные labels:
- `service` ∈ {`tg-sender`, `api`}.
- `request_id` (для tracing).
- `dedupe_key` (для TG).
- `endpoint` (для API).

---

## 9. Cross-references

- TG `delivery_queue` schema → `05_DATA_CONTRACTS.md §10`.
- SSE event structure → `05_DATA_CONTRACTS.md §10.3`.
- AlertEvent для alerts log → `05_DATA_CONTRACTS.md §7`.
- Settings storage → `05_DATA_CONTRACTS.md §12`.
- Reverse-proxy / TLS → `13_OPERATIONS.md §4`.
- Security boundaries → `02_ARCHITECTURE.md §6`.
