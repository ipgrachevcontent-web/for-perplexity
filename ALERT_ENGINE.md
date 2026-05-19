# Что делает alert engine
Его вход — не raw API и не normalizer, а уже готовые метрики из time-series storage:

- oi_now
- delta_5m_abs, delta_5m_pct
- delta_15m_abs, delta_15m_pct
- delta_30m_abs, delta_30m_pct
- sample_age_ms
- sample_count
- exchange
- canonical_symbol

На выходе он создаёт одно из трёх решений:
- FIRE — отправить алерт;
- SUPPRESS — условие формально true, но нельзя слать из-за cooldown/дубля/плохого качества;
- RESOLVE — условие больше не выполняется, считаем сигнал завершённым.

---

## Основной принцип
Правильная логика alert engine — stateful, а не stateless. Он должен помнить, было ли правило уже активировано по этому exchange + symbol + window + rule_id, потому что иначе при каждом цикле, пока OI выше порога, ты будешь спамить в Telegram одинаковыми сообщениями.

*То есть у движка должно быть состояние наподобие:*
```json
{
  "rule_id": 17,
  "exchange": "binance",
  "canonical_symbol": "BTCUSDT",
  "window": "15m",
  "state": "fired",
  "last_fired_at": "2026-04-30T12:15:00Z",
  "cooldown_until": "2026-04-30T12:30:00Z",
  "last_value": 8.42
}
```
***Это и есть база антиспама и дедупликации.***

---

## Какие правила он проверяет
Базовые типы правил для OI-мониторинга:
- Рост OI больше X% за 5/15/30 минут.
- Падение OI меньше -X% за 5/15/30 минут.
- Абсолютный прирост OI в USDT больше X.
- Рост OI на нескольких биржах одновременно по одному символу.
- Рост OI при росте/падении цены, то есть divergence/confirmation rule.
- Только top-liquidity symbols или исключения по биржам.

*То есть правило — это не просто порог, а структура:*
```json
{
  "rule_id": 17,
  "scope": "symbol",
  "exchange_filter": ["binance", "bybit"],
  "symbol_filter": ["BTCUSDT", "ETHUSDT"],
  "window": "15m",
  "metric": "delta_oi_pct",
  "operator": ">=",
  "threshold": 5.0,
  "min_oi_notional": 10000000,
  "cooldown_sec": 900,
  "require_freshness_sec": 120,
  "confirm_points": 2
}
```

---

## Pipeline alert engine
Работа движка в 7 шагов.
### 1. Выбор кандидатов
Движок не должен проходить по всем символам и всем правилам “вслепую”. Он сначала получает candidate set: последние 5m/15m/30m метрики из continuous aggregates или materialized views, где уже рассчитаны delta_abs, delta_pct, sample_age и sample_count.

На этом шаге можно сразу отсечь:
- stale symbols;
- слишком маленький OI;
- символы вне watchlist;
- биржи, которые временно деградировали.

### 2. Quality gate
Перед сравнением с порогом движок проверяет качество данных:
- sample_age_ms не слишком старый;
- в окне достаточно точек;
- нет большого гэпа в истории;
- источник не находится в degraded state.

***Это особенно важно из-за возможных задержек API. Если Bybit сам предупреждает о temporary delays в data delivery при сильной волатильности, alert engine не должен слепо считать поздний пакет “свежим market event”.***

### 3. Rule evaluation
На этом шаге движок применяет правило:
- берёт конкретную метрику, например delta_15m_pct;
- сравнивает с порогом;
- при необходимости применяет вторичные фильтры, например oi_notional_usdt > X и price_change_15m > Y.

Пример:
- delta_15m_pct >= 6
- oi_now >= 20M USDT
- sample_age <= 90 sec
- sample_count >= 2

***Если всё true, кандидат переходит в стадию state machine.***

### 4. State machine
Здесь alert engine решает, это новое событие или повтор уже известного.
Состояния я бы сделал такие:
- idle
- pending_confirm
- fired
- cooldown
- resolved

Логика:
- idle -> pending_confirm, если условие впервые выполнено.
- pending_confirm -> fired, если условие подтверждено ещё N раз подряд.
- fired -> cooldown, сразу после отправки.
- cooldown -> fired только если произошло новое пересечение, а не просто условие остаётся true.
- fired/cooldown -> resolved, если метрика вернулась ниже recovery-level.

***Это и есть ядро anti-noise логики.***

### 5. Crossing logic
Ключевой принцип: алерт шлётся на пересечении, а не на постоянном нахождении выше порога. Для этого нужно хранить предыдущее значение метрики.
Например:
- было 4.8%, стало 5.2% при пороге 5% → cross_up, отправляем.
- было 7.1%, стало 6.8% → порог всё ещё превышен, но нового пересечения нет, не отправляем.
- было 6.2%, стало 4.7% → cross_down, можно пометить resolved.

***Без crossing logic Telegram превратится в flood-channel.***

### 6. Cooldown и dedupe
*После отправки alert engine записывает dedupe_key, например:*
`rule_id:17|exchange:binance|symbol:BTCUSDT|window:15m|bucket:2026-04-30T12:15:00Z`

Если такой ключ уже был отправлен, повтор запрещён. Дополнительно ставится cooldown_until, чтобы даже при незначительном повторном пересечении внутри короткого интервала не спамить пользователя.

Cooldown бывает двух типов:
- hard cooldown — не слать вообще ничего N минут;
- smart cooldown — слать только если новое значение существенно сильнее предыдущего, например было +5.2%, стало +11.8%.

*Для OI-алертов smart cooldown полезнее.*

### 7. Delivery request
Когда движок решил FIRE, он не отправляет Telegram напрямую, а создаёт delivery job:
- channel=telegram
- template=oi_threshold_cross
- payload={exchange, symbol, window, current_oi, delta_pct, delta_abs, price_change, url}

***Это важно, чтобы alert engine оставался pure decision engine, а форматирование и доставка были отделены.***

**Типы сигналов**
1. Threshold signal - Самый простой:
- delta_5m_pct > X
- delta_15m_pct > X
- delta_30m_pct < -X

2. Confirmed signal - Сигнал с подтверждением:
- условие выполнено на двух последовательных evaluation cycles;
- или подтверждено двумя окнами, например 5m и 15m одновременно.

*это снижает шум*

3. Consensus signal - Сигнал, когда тот же canonical symbol активен на нескольких биржах:
- BTC на Binance + Bybit + OKX одновременно показывает рост OI.
*Это сильнее, чем локальный всплеск на одной площадке.*

4. Divergence signal - Пример:
- OI сильно растёт, а цена падает;
- OI падает, а цена растёт.
*Такой тип сигнала часто полезнее чистого OI%.*

5. Exchange health signal - Alert engine может слать не только market alerts, но и infra alerts:
- “Bybit OI stale > 5 min”
- “KuCoin 5m series missing”
- “Binance symbol coverage dropped to 72%”

*Это крайне полезно для поддержки системы.*

---

## Что хранить в БД alert engine

*Минимум 4 таблицы.*

1. alert_rules - Пользовательские правила:
- rule_id
- user_id
- exchange_filter
- symbol_filter
- window
- metric
- threshold
- cooldown_sec
- confirm_points
- enabled

2. alert_state - Текущее состояние по правилу и объекту:
- rule_id
- exchange
- canonical_symbol
- window
- state
- last_metric_value
- last_eval_at
- last_fired_at
- cooldown_until

3. alerts_sent - История реально отправленных алертов:
- rule_id
- dedupe_key
- sent_at
- payload_json
- delivery_status

4. alert_events - Журнал решений движка:
- decision=fire/suppress/resolve
- reason=threshold_cross/cooldown/stale_data/...
- metric_snapshot

*Эта таблица очень помогает отлаживать, почему сигнал не пришёл.*

---

## Как часто должен работать движок
Оптимально запускать evaluation cycle чаще, чем самое маленькое окно, но не слишком часто. Для твоей задачи обычно достаточно:
- каждые 15–30 секунд для snapshot-based источников;
- каждые 1–2 минуты для interval-based окон 5/15/30 минут.

*Если используешь Binance historical 5m/15m/30m и KuCoin/Bybit interval APIs как primary source, нет смысла “тревожить” движок 10 раз в секунду. А вот если часть бирж идёт через собственные снапшоты, полезно держать evaluation cycle чаще и использовать freshness gates.*

---

## Практическая формула хорошего сигнала
- window = 15m
- delta_oi_pct >= 5
- oi_now >= 5_000_000 USDT
- sample_age <= 120 sec
- confirm_points = 1
- cooldown = 15 min
- dedupe on bucket

Для high-signal режима:
- delta_oi_pct >= 8
- delta_price_pct >= 1 или divergence-rule
- consensus_count >= 2 exchanges
- cooldown = 30 min

---

## Архитектурно
Правильная структура такая:
- metrics_reader — читает 5m/15m/30m из storage.
- rule_matcher — находит, какие правила применимы.
- quality_gate — проверяет freshness и полноту данных.
- state_machine — определяет cross/cooldown/resolve.
- event_writer — пишет решение в БД.
- delivery_producer — создаёт задачу в Telegram/web-notification очередь.

*То есть alert engine — это правиловый автомат поверх временных рядов, а не просто SQL с WHERE delta_15m_pct > 5.*

---

## Самая полезная инженерная мысль
***Думай о нём так: normalizer отвечает за истинность точки, time-series storage — за истинность ряда, а alert engine — за истинность события. Событие возникает только когда данные свежие, ряд достаточно полный, порог пересечён, сигнал не был уже отправлен и пользователь действительно должен об этом узнать.***
