# 00 · Decisions Log

> Источник истины для всех архитектурных решений по проекту `oi-tracker`.
> При конфликте с любым из исходных файлов (`OI_TRACKER_SPEC.md`, `SPEC.md`, `NORMALIZER.md`, `TIME_SERIES_STORAGE.md`, `ALERT_ENGINE.md`) — приоритет у этого документа.

**Статус:** все решения согласованы 2026-04-30. Записи F12–F17 добавлены 2026-04-30 после инвентаризации хост-машины. Запись F18 добавлена 2026-05-01 при подготовке к старту разработки.
**Версия:** 1.2
**Владелец:** заказчик проекта (single user).

---

## Как читать этот документ

Каждое решение имеет ID:
- **C-номер** — спорные точки, обнаруженные при cross-document анализе исходных спеков.
- **Q-номер** — продуктовые/операционные вопросы, не затронутые в исходных спеках.
- **F-номер** — финальные доуточнения после первичного раунда решений.

Колонки таблиц:
- **Решение** — что зафиксировано.
- **Обоснование** — почему именно так.
- **Отход от исходного ТЗ** — если решение меняет первичные формулировки, это явно отмечено.

---

## 1. Семантика данных

### C1 · Канонический символ

**Решение:** `canonical_symbol` хранит только базовый актив (например `"BTC"`, не `"BTCUSDT"`). Раздельные колонки описывают рынок:
- `quote_asset` — `"USDT"` (всегда для текущего scope).
- `settle_asset` — `"USDT"` (всегда для текущего scope).
- `market_type` — `"usdt_perp"` (всегда для текущего scope).
- `contract_type` — `"perpetual"` (всегда для текущего scope).

**Обоснование:** для чисто USDT-M perp scope `BASE` достаточен и нагляден. Раздельные поля делают схему расширяемой на USDC-M / COIN-M / опционы без миграций.

**Отход от исходного ТЗ:** в `OI_TRACKER_SPEC §2` сказано `→ BTC`, а в §3.1.2 пример показывает `BTCUSDT`. Это противоречие закрыто в пользу `BASE`.

### C5 · База расчёта дельты

**Решение:** Δ считается по `oi_coins` (количество монет в открытых позициях). `oi_notional_usdt = oi_coins × price_used` — производное display-поле.

**Формула:**
```
delta_pct = (oi_coins_now − oi_coins_then) / oi_coins_then × 100
```

**Обоснование:** Δ по монетам изолирует «реальное изменение OI» от «движения цены». Δ по notional USDT смешивает два разных сигнала, что искажает алерты при волатильности цены.

**Отход от исходного ТЗ:** `OI_TRACKER_SPEC §3.4` содержал TODO-маркер `«НУЖНО: Переформулировать расчёт дельты: Δ по OI в монетах»` — закрыто.

### C4 + F5 · Тип порогов

**Решение:**
- Триггер алерта — только относительный Δ в %.
- `min_oi_notional_usdt` — floor-фильтр (не порог Δ, а порог абсолютного OI), отсекающий шум на токенах с малым OI. Default: `5_000_000` USDT.
- Абсолютные Δ-пороги (например «прирост на $X миллионов») в v1 не используются.

**Обоснование:** % сравним между BTC и DOGE; абсолютный USDT-фильтр отсекает «5% на токене с OI $50k», что и было исходной заботой ТЗ. Введение абсолютных Δ-порогов как отдельной сущности усложняет правила без явного use case.

**Совместимость с ТЗ:** `OI_TRACKER_SPEC §3.4` — «только относительное изменение в процентах (без абсолютов)» — соблюдено для **триггера**. Floor-фильтр — это качество данных, не дополнительный порог Δ.

---

## 2. Сбор данных

### C2 · Polling cadence

**Решение:** flat 60s polling, выровнено по минуте, для всех символов всех бирж.

**Обоснование:** окна 5/15/30 — это минутная решётка. Tier-based polling (15s топ / 60s хвост) ломает регулярность ряда: top-символы получали бы 4 точки в минуту, хвост — одну, что делает Δ семантически разными для разных символов. Для рассматриваемого объёма (~300 символов × 12 бирж) 60s flat укладывается в rate limits всех бирж.

**Эскалация:** если для отдельной биржи rate limit будет нарушаться (например Binance Futures с `weight=1` per request), tiering вводится **только для этой биржи**, не системно.

### F2 · Sync списка инструментов

**Решение:** список инструментов (instruments registry) обновляется каждые **5 минут**, не каждую минуту.

**Обоснование:** новые листинги не появляются в пределах секунд; типичные `exchangeInfo` endpoint'ы тяжёлые (Binance Futures `exchangeInfo` ≈ 250 KB). Опрос каждую минуту = 12 лишних тяжёлых запросов в минуту без выгоды.

**Отход от исходного ТЗ:** в `OI_TRACKER_SPEC §3.1` сказано «при каждом цикле обновляется список символов» — заменено на каждые 5 минут. Делистнутые символы перестают опрашиваться (история сохраняется).

### C12 · Источник истины для окон 5/15/30

**Решение:**

| Биржи | Primary source | Fallback |
|---|---|---|
| Bybit, KuCoin | `native_interval` (готовые 5/15/30-мин бары через API) | `degraded_fallback` на snapshot при > 2 cycles деградации |
| Binance, OKX, Bitget, Gate.io, MEXC, HTX, Hyperliquid, Aster, Bitmart | `snapshot` (текущее OI каждую минуту, окна считаются у нас) | — |
| XT | `snapshot` (через bulk-endpoint, возвращающий все контракты сразу) | — |

В пределах одной пары `(exchange, canonical_symbol)` источник один и тот же; миксования в одном ряде нет. Между биржами — да, BTC на Binance может быть `snapshot`, BTC на Bybit — `native_interval`.

**Обоснование:** native interval даёт точные exchange-side бары (выровненные по серверным часам биржи); snapshot-derived — наш собственный ряд, точный по нашим часам. Смешивать их в одном ряде = ложные дельты на стыках.

### F3 · Изоляция ошибок биржи

**Решение:**
- Per-exchange отдельный async task (один коннектор не блокирует остальные).
- При недоступности биржи: тик пропускается, событие пишется в `connector_errors`, retry в следующую минуту.
- Circuit breaker per-exchange: после `N=5` подряд неудачных циклов — пауза `30s` перед следующей попыткой.
- Падение одного коннектора не ломает остальные.

**Совместимость с ТЗ:** соблюдено как в `OI_TRACKER_SPEC §3.1`.

---

## 3. Хранение

### C3 · Retention

**Решение:** **90 дней** — для всего: и для сырых нормализованных точек, и для всех continuous aggregates (5m/15m/30m). Без downsampling.

**Обоснование:** TimescaleDB compression снижает объём в 10–20× → даже 90 дней с 1-минутным разрешением остаётся управляемо. Хранить агрегаты дольше, чем raw, бессмысленно — теряется консистентность.

**Отход от исходного ТЗ:** `TIME_SERIES_STORAGE §6.1` предлагал «агрегаты — 1 год». Заменено на единые 90 дней.

### C9 · Структура schema

**Решение:** wide TEXT schema с TimescaleDB compression, без surrogate FK для exchange/symbol.
- Колонки `exchange TEXT`, `canonical_symbol TEXT` хранятся прямо в `oi_samples`.
- Compression `segmentby = (exchange, canonical_symbol)` сжимает повторяющиеся TEXT'ы в 30–50×.
- Запросы UI/alert engine читаются без 3 JOIN'ов.

**Отход от исходного ТЗ:** `OI_TRACKER_SPEC §7` предлагал нормализованную схему `oi_ticks(exchange_id, symbol_id)` с FK. Заменено wide TEXT'ом.

### C11 · Hypertable time-axis

**Решение:**
- Hypertable партиционируется по `ts_exchange` (тип `TIMESTAMPTZ`).
- `ts_ingested` (`TIMESTAMPTZ`) — операционная колонка для SLA / диагностики lag'а.
- `ts_processed` (`TIMESTAMPTZ`) — момент прохождения normalizer + alert engine, для трассировки.

**Обоснование:** все бизнес-запросы (окна, Δ, алерты) — по market-time. Если партиционировать по `ts_ingested`, поздно прибывшие точки попадают в «не свой» chunk → continuous aggregates становятся неконсистентны.

**Отход от исходного ТЗ:** `TIME_SERIES_STORAGE §2.1` предлагал `create_hypertable(..., 'ts_ingested')` — заменено на `ts_exchange`.

### F4 · Numeric precision

**Решение:**
- `oi_coins`, `price_used` — `NUMERIC(38, 18)`.
- `oi_notional_usdt` — `NUMERIC(38, 8)`.

**Обоснование:** низкокаповые токены имеют цены `0.000000123` — 18 знаков после запятой минимум. USDT-notional с 8 знаками — стандарт для финансовых полей.

### F5 · Compression и chunk

**Решение:**
- `chunk_time_interval = INTERVAL '1 day'`.
- Compression policy: после 7 дней.
- Retention policy: 90 дней (см. C3).

---

## 4. Алерты

### C6 · State machine

**Решение:** 4-state machine на ключе `(rule_id, exchange, canonical_symbol, window)`:
```
idle → armed → fired → cooldown → armed
                          ↓ (cooldown TTL истёк И value < threshold)
                        armed
```

- `idle` — нет истории по ключу (новый символ).
- `armed` — следим, value < threshold.
- `fired` — пересекли threshold, алерт отправлен.
- `cooldown` — N минут блокировка от повторов (smart cooldown — см. F7).

`confirm_points = 1` по умолчанию (rising edge без подтверждения).

**Отход от исходного ТЗ:** `ALERT_ENGINE §4` предлагал 5-state с `pending_confirm` и `resolved`. Упрощено: `pending_confirm` добавляет 1 минуту латентности (20% времени 5-мин окна), `resolved` для pump/dump неактуален. State machine оставлена как абстракция — confirm_points можно увеличить через config без рефакторинга.

### C7 · Freshness формулы

**Решение:** все freshness budgets — формулы, не magic numbers.

| Контекст | Формула | Значение при `poll = 60s` |
|---|---|---|
| «Свежесть последней точки для алерта» | `2 × poll_interval` | 120s |
| «Допуск на поиск точки t−N для Δ» | `1.5 × poll_interval` | 90s |
| «Свежесть native interval» | `bucket_size + 60s grace` | 5m+60s = 360s |

Хранятся в `settings(key, value_json)`, не хардкодятся.

**Совместимость с ТЗ:** `OI_TRACKER_SPEC §3.4` — «±90 сек» — теперь как `tolerance_for_t_minus_n = 1.5 × poll_interval`.

### F8 · Минимум истории

**Решение:** `min_history_minutes = max(30, 2 × window_size_minutes)`.

| Window | Min history |
|---|---|
| 5 min | 30 min |
| 15 min | 30 min |
| 30 min | 60 min |

**Обоснование:** для 5/15-мин окон 30 минут истории достаточно (как в ТЗ). Для 30-мин окна 30 минут — это **минимум для первого расчёта Δ** без буфера; нужно ≥ 60 минут, чтобы алерт не сработал на первой же точке.

**Отход от исходного ТЗ:** `OI_TRACKER_SPEC §3.5` фиксировал плоско `≥ 30 мин`. Заменено формулой.

### Q1 + F7 · Smart cooldown

**Решение:** smart cooldown.
- Default `cooldown_seconds = 900` (15 минут).
- Внутри cooldown допускается re-fire, **только если** новое значение Δ ≥ `1.5 × previous_fired_delta`.
- Пример: первый алерт +5.2% → второй разрешён только при ≥ +7.8%.

**Отход от исходного ТЗ:** `OI_TRACKER_SPEC §3.5` описывал hard cooldown («после срабатывания алерт по этой комбинации блокируется на N минут»). Заменено на smart.

### Q2 · Типы сигналов в v1

**Решение:** все 5 типов с самого начала.

1. **Threshold** — `delta_window >= +X` или `<= -X`.
2. **Confirmed** — threshold с `confirm_points >= 2` (требует N последовательных evaluation cycles).
3. **Consensus** — тот же canonical_symbol триггерит на ≥ M бирж одновременно (default M=3).
4. **Divergence** — рост OI + падение цены, или наоборот.
5. **Exchange-health** — операционный сигнал (биржа stale, coverage упал, missing samples).

Подробности — в `09_ALERT_ENGINE.md`.

### F6 · Хранение правил алертов

**Решение:** структурированная таблица `alert_rules(id, scope, rule_type, exchange_filter, symbol_filter, window, metric, operator, threshold, min_oi_notional, cooldown_sec, confirm_points, valuation_status_min, source_kind_filter, enabled)`.

- 6 базовых порогов из ТЗ (`up_5m, down_5m, up_15m, down_15m, up_30m, down_30m`) реализуются как **6 default-строк** scope=`global`. UI показывает 6 знакомых ползунков.
- Правила Q2 (divergence, exchange-health) — отдельные строки, тоже scope=`global` или scope=`symbol`. _(Consensus был в исходном Q2, sunset per F40.)_

**Отход от исходного ТЗ:** `OI_TRACKER_SPEC §7` предлагал `settings(key, value_json)` для всех настроек. Заменено: глобальные тунабли остаются в `settings`, правила алертов — в структурной таблице.

---

## 5. Доставка и интерфейс

### C8 · Single user, групповой чат

**Решение:**
- `user_id` отсутствует в schema полностью.
- Алерты отправляются в один Telegram групповой чат.
- `chat_id` хранится в `settings`.

### Q8 · Telegram credentials

**Решение:**
- TG bot token — `env-only` (читается на старте процесса). В БД не хранится.
- `chat_id` — в `settings`, редактируется из UI.

### C14 · UI обновление

**Решение:** SSE-push, без 5-секундного polling.
- Backend пушит в SSE при появлении новой точки или нового алерта.
- React UI рендерит на receive event.
- На первом коннекте — REST snapshot текущего состояния, дальше — только delta.
- Reconnect с экспоненциальным backoff (1s → 2s → 4s → ... → 30s cap).

**Отход от исходного ТЗ:** `OI_TRACKER_SPEC §3.7` — «обновление каждые ~5 сек» — заменено на event-driven SSE.

### Q4 · Пользовательская модель

**Решение:** 1 пользователь, без авторизации, без публичного API. UI и REST-endpoint'ы открыты в локальной/VPN-сети, без аутентификации.

---

## 6. Operations

### Q3 · Latency budget

**Решение:** end-to-end (тик биржи → отправка в TG):
- **P95 ≤ 90 секунд.**
- **P99 ≤ 180 секунд.**

Это включает: fetch (биржа → коннектор), normalize, persist, CA refresh, alert evaluation, TG send.

### Q5 · Observability

**Решение:** полный стек:
- **Prometheus** — метрики.
- **Grafana** — дашборды.
- **Loki** — логи.

Подробности — в `12_OBSERVABILITY_SLO.md`.

### Q6 · Deployment

**Решение:** bare-metal, systemd-юниты, **без Docker**.

Юниты:
- `oi-tracker-api.service` — FastAPI + SSE.
- `oi-tracker-scheduler.service` — async scheduler + 12 коннекторов.
- `oi-tracker-tg-sender.service` — TG sender worker.
- `oi-tracker-cleanup.service` (timer) — daily retention job.

### Q7 · Backups

**Решение:** в v1 не настраиваем. Future work: pg_dump nightly + WAL-G в S3-совместимый storage.

---

## 7. Reprocessing и качество данных

### F9 · No retro-fire

**Решение:** backfill / replay обновляют `oi_samples` и continuous aggregates, **но не создают новые FIRE-события задним числом**. Alert engine работает только на «срезе вперёд» (текущий + будущие bucket'ы по `ts_exchange`).

При backfill можно опционально помечать уже отправленные алерты как `historically_inaccurate` в `alert_events` — но в TG ничего не отправляется.

### F10 · Valuation status

**Решение:** каждая нормализованная точка имеет `valuation_status ∈ {authoritative, good_estimate, low_confidence}`.
- `authoritative` — биржа сама вернула notional (Binance `sumOpenInterestValue`).
- `good_estimate` — `oi_coins × mark_price` или `contracts × contract_size × price`.
- `low_confidence` — fallback на `last_price` или приблизительная конверсия.

Алерты по умолчанию разрешены при `>= good_estimate`. UI/heatmap может показывать `low_confidence` точки с пометкой.

### F11 · Source kind

**Решение:** каждая точка имеет `source_kind ∈ {snapshot, native_interval, backfill, replay, degraded_fallback}`.

`degraded_fallback` понижает severity алертов или отключает часть стратегий (например, divergence игнорируется).

---

## 8. Прочее

### C13 · Mermaid поток данных

**Решение:** перерисованная диаграмма без отдельной "Compute 5m/15m/30m" ноды. Деривация окон — это автомат TimescaleDB CA, не отдельный сервис. Финальная диаграмма — в `02_ARCHITECTURE.md §3`.

### C15 · §2 OI_TRACKER_SPEC корректуры

**Решение:** в `02_ARCHITECTURE.md` и `11_EXCHANGE_ADAPTERS/<exchange>.md` для OKX, Hyperliquid, Aster используются формулировки, присланные пользователем — без копипаста между «Основной тип» и «Тип контракта в API».

### C16 · Имя проекта

**Решение:** `oi-tracker`. Опечатка `traadefi/` в `OI_TRACKER_SPEC §6` исправлена.

### C17 · Количество бирж

**Решение:** все упоминания «7 бирж» в DDL-комментариях → «12 бирж».

---

## 9. Infrastructure & Deployment (F12–F17)

Эти решения зафиксированы 2026-04-30 после инвентаризации хост-машины и формулировки требований единства домена/изоляции от соседних проектов. Подробности — `15_DEPLOYMENT_INFRA.md`.

### F12 · Домен

**Решение:** production-домен — `oi-tracker.robot-detector.ru` (subdomain существующего `robot-detector.ru`).

**Обоснование:** заказчик уже владеет зоной `robot-detector.ru`; subdomain не требует регистрации нового domain'а; nginx site-config полностью изолирован от существующего `robot-detector` server-block (server_name отличается).

**Применение:** server_name в nginx, OpenAPI servers field, документация, фиксация в frontend `vite.config.ts` для production build (если CORS-эндпоинты будут).

### F13 · Запрет индексации

**Решение:** все ответы oi-tracker имеют `noindex, nofollow, noarchive, nosnippet`. Защита трёхуровневая (defense in depth):

1. **HTTP-заголовок** в nginx: `add_header X-Robots-Tag "noindex, nofollow, noarchive, nosnippet" always;`.
2. **`robots.txt`** в `/var/www/oi-tracker/frontend-dist/robots.txt`: `User-agent: * / Disallow: /`.
3. **HTML meta** в `index.html`: `<meta name="robots" content="noindex, nofollow, noarchive, nosnippet">`.

**Обоснование:** single-user приватная утилита; индексация поисковиками = утечка структуры приложения, рисков не несёт, но и пользы нет. Three-layer защита потому, что `X-Robots-Tag` достаточно для compliant роботов, а `robots.txt` + meta — для тех, что игнорируют HTTP-заголовки.

**Что НЕ делаем:** HTTP Basic Auth, IP allowlist (single-user через UI без барьеров — `Q4`).

### F14 · Изоляция от соседних проектов и production hardening

**Решение:** `oi-tracker` развёртывается на shared-машине (`/var/www/detector` для robot-detector.ru, `/var/www/grach-ege` для grach-ege, общий PG16) с полной изоляцией:

| Аспект | Решение |
|---|---|
| IPC FastAPI ↔ nginx | **Unix domain socket** `/run/oi-tracker/api.sock`. Не TCP. Порт 8000 уже занят соседом `detector_api`. |
| Fallback порт | `127.0.0.1:8010` (не открывать наружу). Использовать только для отладки. |
| PostgreSQL | Общий кластер, **отдельная database** `oi_tracker`, **отдельная role** `oi_tracker`. `REVOKE ALL` на соседние БД (`detector`, `grachege`, `grachege_shadow`). |
| OS user | `oi-tracker:oi-tracker` (system user, `/usr/sbin/nologin`). Никаких процессов под `root`. |
| Code path | `/opt/oi-tracker/code` |
| Env file | `/etc/oi-tracker/oi-tracker.env`, `chmod 0600`, owner `oi-tracker`. |
| Logs | `/var/log/oi-tracker/` + logrotate (rotate 14 daily). Не трогаем глобальный journald.conf. |
| Frontend root | `/var/www/oi-tracker/frontend-dist` (отдельная директория, не общий `/var/www/html`). |
| systemd unit prefix | `oi-tracker-*` (`scheduler`, `api`, `tg-sender`, `cleanup`). |
| nginx site name | `oi-tracker` в `sites-available/`. Никаких правок в чужие site-configs. |
| Resource limits | `MemoryMax=4G/2G/512M` (scheduler/api/tg-sender), `TasksMax=512/512/128`. |

**Hardening (общий для всего production-кода):**
- `verify=False` в любом исходящем HTTPS — **запрещён**. Отлов: `grep -RE 'verify\s*=\s*False' src/` → 0 строк.
- f-string в SQL — **запрещён**. Только параметризация. Отлов: `ruff` rule `S608`.
- Логи **не содержат secrets**. Audit перед merge.
- `pg_hba.conf` для `oi_tracker` — без `trust`.

**Обоснование:** машина shared между несколькими production-проектами; любой инцидент или конфликт ресурсов не должен задевать `robot-detector` или `grach-ege`. Unix socket и dedicated user — стандартный multi-tenant pattern для bare-metal хостинга.

### F15 · UI Settings scope (максимум безопасных knob'ов)

**Решение:** все настройки, которые можно безопасно менять без code/migration changes, доступны через UI Settings page. Категоризация — `10_DELIVERY_LAYER.md §4.8`:

- **Категория A (всегда видимы):** `tg_chat_id`, `tg_dry_run`, `exchange_whitelist`, `symbol_blacklist`, `min_oi_notional_default_usdt`.
- **Категория B (Advanced tab, доступны):** `cooldown_default_sec`, `smart_cooldown_factor`, `freshness_*` factors, `min_history_minutes_floor`. Все с валидацией диапазонов на бэкенде через pydantic. _(`consensus_min_exchanges` был в Категории B, sunset per F40.)_
- **Категория C (НЕ в UI):** `polling_interval_sec`, `instruments_sync_interval_sec`, `evaluation_cycle_sec`, TimescaleDB policies, `*_version`. Меняются через миграции / редеплой.

**Rate-limit:** PUT `/api/v1/settings/{key}` — 5/min per IP. Audit — таблица `settings_audit_log`.

**Обоснование:** заказчик прямо просил «выносить максимум того, что безопасно настраивать». Категоризация защищает от случайного изменения параметров, ломающих инварианты (например, `polling_interval_sec=10` нарушит `C2` — flat 60s).

### F16 · TLS-стратегия

**Решение:** **отдельный** Let's Encrypt-сертификат для `oi-tracker.robot-detector.ru`, выпускаемый через `certbot --nginx -d oi-tracker.robot-detector.ru`. Не multi-domain reissue текущего cert'а.

**Обоснование:**
- Существующий cert `robot-detector.ru` — single-domain (SAN: только `DNS:robot-detector.ru`, проверено `openssl x509 -ext subjectAltName`). Не wildcard.
- Multi-domain reissue связал бы renewal двух проектов; любая операция certbot затрагивала бы соседский site → выше риск.
- Отдельный cert: independent renewal через тот же `certbot.timer` (уже активен), нулевая связность с соседом.

**Что НЕ делаем:** wildcard cert на `*.robot-detector.ru` через DNS-01 challenge — overkill, нет других subdomain'ов.

### F17 · Установка TimescaleDB extension

**Решение:** TimescaleDB extension **не установлена** в текущем PG16 (проверено `SELECT * FROM pg_available_extensions WHERE name='timescaledb'` — пусто). Установка через `apt install timescaledb-2-postgresql-16` + `timescaledb-tune`. **Требует restart кластера**, что задевает соседей `detector` и `grach-ege` (~30–60s downtime).

**Процедура** — `15_DEPLOYMENT_INFRA.md §6.1`. Краткая суть:
1. Уведомить владельцев соседних проектов.
2. `pg_dump` соседних БД (страховка).
3. Backup `postgresql.conf`.
4. `apt install timescaledb-2-postgresql-16`.
5. `timescaledb-tune --quiet --yes`.
6. `systemctl restart postgresql@16-main`.
7. Smoke-test соседей.
8. `CREATE EXTENSION timescaledb` в БД `oi_tracker` (не в соседних).

**Откат:** restore `postgresql.conf` из бэкапа + restart. Пакет можно оставить (inert без `shared_preload_libraries`).

**Обоснование:** TimescaleDB — обязательная зависимость для `C9, C11, F4, F5, F2 (CA)`. Установка extension в shared-кластер — стандартный one-time maintenance event, после которого все три проекта (`oi_tracker` + `detector` + `grachege`) работают как раньше; `oi_tracker` дополнительно может создавать hypertables.

### F18 · Default thresholds для 6 base alert rules

**Решение:** при первом seed `alert_rules` (см. `09_ALERT_ENGINE §7`, `F6`) создаются 6 строк со следующими `delta_pct`:

| window | up | down | smart_cooldown_factor | min_history_minutes |
|---|---|---|---|---|
| 5m   | +5%  | −5%  | 1.5 | 30 |
| 15m  | +8%  | −8%  | 1.5 | 30 |
| 30m  | +12% | −12% | 1.5 | 60 |

Все 6 строк — `confirm_points = 1`, `min_oi_notional_usdt = 5_000_000`, `cooldown_default_sec = 900`, статус `active`.

**Обоснование:**
- 5/8/12 — эмпирический baseline для USDT-M perp OI: 5% за 5 минут — заметное движение, отсекает шум; 8% за 15 минут и 12% за 30 минут — последовательная эскалация чувствительности по окнам.
- Симметричные up/down — чтобы пользователь не пропустил squeeze-вниз так же, как memo-вверх.
- Все остальные параметры наследуются от глобальных defaults (`F7, F8, F5, C4`) — рулса не вводит новых констант.

**Где меняется:** через UI Settings (категория B, см. `F15`) или через `PUT /api/v1/alert-rules/{id}` (см. `10_DELIVERY_LAYER §4`).

**Совместимость с ТЗ:** `F6` фиксировал «6 default rows» без явных значений. F18 закрывает implementation-default, чтобы значения не «всплывали» из кода как магические числа.

---

### F19 · Bybit/KuCoin shipped on snapshot path (deferred native_interval to M3)

**Решение:** Bybit и KuCoin в M2.B shipped с **snapshot** primary path, не `native_interval` как фиксировал `C12`. `native_interval` отложен на M3.

**Обоснование:**
- Bybit `/v5/market/tickers?category=linear` отдаёт `openInterestValue` (USDT notional) — pipeline emits `valuation_status=authoritative`. Тот же tier, на который вышел бы native_interval путь, без потери качества данных.
- KuCoin `/api/v1/contracts/active` — единственный bulk-endpoint, возвращающий roster + OI + prices одновременно. Цикл ~1.4s vs N per-symbol calls для native_interval.
- `native_interval` требует `fetch_range` + watermark machinery (resume-from-checkpoint, dedupe by bucket boundary) — это новый primary path в scheduler/normalizer/pipeline, архитектурно относится к M3.
- `oi_samples.source_kind = SNAPSHOT` корректен для текущего поведения; downgrade `C12` не требуется — он остаётся целью, F19 фиксирует временную траекторию.

**Что это меняет:**
- В M3 (replay/backfill) — добавить native_interval primary path для Bybit и KuCoin; миграция samples не нужна (они уже в правильном valuation tier).
- `11_EXCHANGE_ADAPTERS/bybit.md` и `kucoin.md` обновить: snapshot как «shipped path в v1, native_interval в M3».

**Совместимость с ТЗ:** валидаторская семантика `C12` (источник единый для пары `(exchange, canonical_symbol)`, не миксуется в одном ряде) сохранена — пишем только snapshot, не миксуем.

---

### F20 · Bitunix deferred — public API без OI

> **Superseded by F42** (2026-05-04): 12-я биржа = Bitmart, не Bitunix. Bitunix снят с roadmap.

**Решение:** Bitunix коннектор отложен из M2 scope. Production работает на 11 биржах (binance + 10), не 12 как фиксировал `C17`.

**Обоснование:** при попытке реализации (2026-05-01) обнаружено, что public API Bitunix **не отдаёт OI**:
- `/api/v1/futures/market/tickers` — только `markPrice/lastPrice/last/open/high/low/baseVol/quoteVol`, поля `openInterest` нет вопреки спеке `11_EXCHANGE_ADAPTERS/bitunix.md §3`.
- `/api/v1/futures/market/open_interest?symbol=...` со всеми вариантами параметров → `code=2 "Parameter error"`.
- OI закрыто за authenticated API (per spec §10.2 review-based access).

**Что это меняет:**
- `C17` «12 бирж» → «11 бирж в v1, Bitunix deferred». DDL-комментарии оставить «12 поддерживаемых архитектурно» — connector factory готова принять Bitunix как только появится auth-flow.
- ~~`consensus_min_exchanges = 3` (per `Q2`) пересчёта не требует — 11 бирж > 3.~~ _Не актуально — consensus sunset per F40._
- `11_EXCHANGE_ADAPTERS/bitunix.md` переписать: §3 + §10 под реальную ситуацию (bulk /tickers без OI, public только funding_rate/depth/kline).

**Reactivation criteria** (любое из):
1. Bitunix откроет OI в public API.
2. Получаем review-approved API key и адаптируем connector под authenticated endpoints.

**Lesson:** `connector-spec-vs-reality` (`tasks/lessons.md`) — capture API behaviour first, write spec second.

---

### F21 · Pipeline notional fix для `base_asset` hint с multiplier > 1

**Решение:** в `pipeline.normalize()` Stage 4 при `oi_unit_hint=base_asset && instrument.multiplier > 1` mark price делится на multiplier перед расчётом `oi_notional_usdt`. Хранимое `price_used` — per-base-unit (не per-prefixed-unit). Bumped `NORMALIZATION_VERSION` 1 → 2.

**Формула:**
```
if oi_unit_hint == "base_asset" and instrument.multiplier > 1:
    per_base_price = mark_price / multiplier
else:
    per_base_price = mark_price
oi_notional_usdt = oi_coins * per_base_price
price_used = per_base_price
```

**Обоснование:** Stage 3 умножает `oi_raw × multiplier` чтобы получить pure-base `oi_coins` (1000PEPE: 88M контрактов × 1000 = 88B PEPE). Mark price из биржи — per-prefixed-unit (per 1000-PEPE unit). Без поправки `oi_coins × mark_price` даёт `88B × $0.004 = $88B`, реальный notional ~$88M. Деление mark_price на multiplier восстанавливает консистентность.

**Affected connectors** (verified DB query 2026-05-01):
- Binance: ~50 1000-prefix tokens (1000PEPE, 1000SHIB, 1000BONK, etc.).
- Aster: same set (Binance-mirror).
- Hyperliquid: 7 k-prefix tokens (kPEPE, kSHIB, kBONK, kLUNC, kFLOKI, kDOGS, kNEIRO).
- НЕ затронуты: OKX/HTX/Bybit (authoritative path через provided notional минует mark-fallback ветку), Bitget/Gate.io/MEXC/KuCoin/XT (используют `contracts` unit hint, multiplier=1 везде).

**DB before/after:**
- Binance 1000PEPE: $88B → $87.9M.
- Aster 1000PEPE: $345M → $345K.
- Hyperliquid kPEPE: $21.7B → $21.7M.

**Инвариант сохранён:** `oi_notional_usdt == oi_coins × price_used` (numeric quantize aside). Downstream Δ-расчёт по `oi_coins` (per `C5`) не затронут — `oi_coins` корректен в обоих случаях, сломан был только display notional.

**`NORMALIZATION_VERSION` bump:** 1 → 2 одновременно с merge fix. M3 replay по желанию пересчитает historical notional из `oi_coins × per_base_price`; селектор по `oi_samples.normalization_version` per `08 §6.2`.

**Implementation:** commit `1bad7cb` (`app/normalizer/pipeline.py` Stage 4 base_asset+mark branch). Lock-in test в `tests/unit/normalizer/test_pipeline.py::test_thousand_prefix_applies_multiplier`.

**Совместимость с `F4, F10`:** оба сохранены — precision и valuation tier не изменились.

---

### F22 · aiogram throttle (Pre-M3 ADR)

**Решение:** на стороне `app/tg_sender/throttle.py` ставим **leaky-bucket throttle** ПЕРЕД каждым `bot.send_message`. Два bucket'а:

| Bucket | Capacity | Refill | Защищает от |
|---|---|---|---|
| global | 25 tokens | 25 tok/sec | bot-wide TG limit (~30/sec) |
| per-chat | 1 token  | 1 tok/sec  | per-chat TG limit (~1/sec) |

Acquire requires 1 token из обоих bucket'ов; пустой → asyncio sleep до refill.

**Обоснование.** Worst-case burst: одновременные threshold-fire'ы на 12 биржах × 3 окна = до 36 алертов. _(Изначально обоснование апеллировало к consensus-event'у, sunset per F40 — но throttle одинаково защищает от любого burst-источника.)_ TG отвечает HTTP 429 с `retry_after`. Без throttle: 429-каскад → retry storm → нарушение `Q3` latency SLO (P95 ≤ 90s). С throttle: 25 msg/sec × 90s = 2250 deliveries в окне — на порядок больше любого реального burst.

**Реализация.**
- `app/tg_sender/throttle.py` — `TgThrottle` class. `LeakyBucket` переиспользует `app/exchanges/_rate_limiter.py:RateLimiter` (тот же алгоритм).
- 429 fallback: при получении 429 от TG используем `retry_after` напрямую как next-delay (не ladder из `10 §2.5.1`). На `retry_after` секунд `throttle.freeze_global()` блокирует acquire — не размножаем 429 в параллельных worker'ах. `retry_count` НЕ инкрементируется (это не delivery failure, а throttle event).
- Метрики: `oi_tg_throttle_wait_seconds` (histogram, label `bucket=global|per_chat`), `oi_tg_throttle_acquire_total` (counter).

**Где задокументировано:** `10_DELIVERY_LAYER.md §2.5.2`.

**Когда land'ит:** decision locked сейчас; код — в M3.B (Telegram dispatcher). До M3.B `tg_dry_run=true`, throttle нужен, потому что dry-run всё равно проходит через тот же worker для тестирования pipeline'а.

**Что это меняет:** ничего в данных или DDL; чистая sender-loop логика. Performance impact на golden path: throttle.acquire() при наличии токенов = O(1), no lock; при пустом bucket — asyncio sleep, что не блокирует event loop.

**Совместимость с ТЗ:** `Q3` SLO держим (см. расчёт выше); `F9` (no retro-fire) сохранён — burst остаётся real-time, только распределяется на 1-2 секунды; `Q8` chat_id из БД, throttle никак не зависит от его значения.

---

### F24 · alert_rules.extra JSONB schema (M3.E)

**Решение.** В `alert_rules` добавлена колонка `extra JSONB NOT NULL DEFAULT '{}'`
(миграция `0009_alert_rules_extra`). Колонка хранит signal-type-specific
параметры, схема flexible — валидация на runtime в каждом сигнале:

- ~~`consensus`: `{"consensus_count": int}`~~ — sunset per F40.
- `divergence`: `{"divergence_price_threshold": Decimal-string}` (default `"1.0"`).
- `exchange_health`: `{"sub_trigger": str, "stale_threshold_min": int}` (default
  `stale_data` / 5min).
- `threshold`/`confirmed`: empty `{}`.

**Зачем JSONB, а не отдельные колонки.** Каждый signal-type имеет свой набор
параметров; column-per-param приводит к разрастанию таблицы и большому числу
NULL'ов. JSONB читается репозиторием в `dict[str, Any]`, проверяется в
конкретном сигнале — нарушение схемы → log warning + return [].

**Pydantic side.** `AlertRule.extra: dict[str, Any] = Field(default_factory=dict)`.
`frozen=True` сохраняется на уровне model_copy (вложенный dict mutable, но мы
никогда не мутируем in-place — только model_copy(update=...)).

**Migration safety.** Additive (`server_default '{}'`); существующие 6 threshold
rules получают пустой extra и продолжают работать. Применено на проде
2026-05-01 14:38 UTC, 0 errors на первом цикле.

---

### F25 · ExchangeHealth v1 ships only `stale_data` sub-trigger

**Решение.** В M3.E реализован только sub-trigger `stale_data` из `09 §5.5` (4
sub-triggers по спецификации). Остальные три отложены:

- `low_coverage` (symbol_coverage_ratio < 95% за 3 цикла) — нужна persistence
  истории coverage на N циклов, которой нет в v1.
- `high_error_rate` (>5 errors/min для биржи) — нужна error log persistence
  или Prometheus scrape (нарушает single-source-of-truth design).
- `instruments_sync_failing` (3 fail подряд) — нужен `instruments_sync_history`
  table (отсутствует).

**Reactivation criteria.** Когда появится `connector_health_history` table
(или Prometheus scrape adapter в alert engine). Реализация остальных трёх
sub-triggers — feat-PR на ~3-4h, сигнал-абстракция к этому готова.

**Bypass дизайн.** ExchangeHealthSignal возвращает `HealthCandidate`
(не `AlertCandidate`). evaluation_cycle ветвится на `RuleType.EXCHANGE_HEALTH`
→ `_evaluate_exchange_health_rule`, обходит state_machine, пишет напрямую в
`delivery_queue`. Cooldown enforced via dedupe_key = `health:rule:<id>|ex:<exchange>|trig:<sub_trigger>|bucket:<floor(now/cooldown_sec)>`
+ `ON CONFLICT (dedupe_key) DO NOTHING`. `alert_state` для health не пишется
(audit log отложен до v1.1 — сейчас trail только в TG + delivery_queue).

---

### F26 · Consensus signal exchange sentinel `<consensus>` _(SUPERSEDED by F40)_

> **Status:** Superseded by `F40` (M8 Phase A, sunset consensus). Sentinel `<consensus>` остаётся только в исторических `alert_events` строках до retention 90 дней. Описание ниже сохранено для археологии.

**Решение.** ConsensusSignal-агрегаты по `canonical_symbol` пересекают
exchanges; AlertCandidate.exchange ставится в строковый sentinel `"<consensus>"`.

**Зачем sentinel.** Сохраняем shape ключа `(rule_id, exchange,
canonical_symbol, window)` для `alert_state` и `dedupe_key` без специальной
ветви в state machine. Один state-row на (rule, symbol, window) для consensus
вместо одного на (rule, ex, symbol, window) для threshold.

**Aggregated bar.** Берём `current_bar` от leg'а с наибольшим
`close_oi_notional_usdt` (самый ликвидный). Это даёт стабильный
`bucket`/`last_ts_exchange` для hash + observability. Aggregated `delta_pct`
= mean Δ всех triggering legs; `oi_now_notional_usdt` = sum по triggering
legs (для quality gate min OI). Worst valuation propagates.

**Cross-exchange time alignment.** `09 §5.3.1` — все CA выровнены на UTC
5/15/30-min boundaries; разные биржи попадают в один bucket. Skew >5min
редок и не bridge'ится в v1.

---

### F27 · SSE — no event replay on reconnect (Pre-M4 ADR)

**Решение.** При reconnect SSE-клиент **не получает** события, пропущенные
во время разрыва. Контракт восстановления свежести:

1. Клиент рестартует EventSource с экспоненциальным backoff
   (`10 §5.6`: 1s → 2s → 4s → ... → 30s cap).
2. На успешном reconnect сервер отправляет `event: connection_init` с
   полным snapshot'ом (`/api/v1/live` shape) — клиент пересобирает
   состояние with one consistent read, не пытается merge'ить gaps.
3. Промежуточные события (`sample`, `alert_fired`) во время offline
   потеряны — это допустимо, потому что:
   - **Алерты** — каноническая копия в Telegram (`F22`-throttled, retry до
     5 попыток), пользователь не пропускает FIRE.
   - **Live OI** — следующий poll (≤60s) перепишет все позиции.
4. На сервере `SSEHub.publish` дропает события для slow consumer'ов через
   `asyncio.QueueFull` (`10 §5.4`) и логирует `sse_queue_full` —
   это операционный сигнал, не функциональный баг.

**UI обязательства (`18 §7.6 ConnectionStatusBadge`):**
- `live` — SSE connected, последнее событие < 5s. Pulse animation.
- `reconnecting` — backoff активен. text-warn, без pulse.
- `offline` — нет SSE > 30s. text-bear, иконка ⚠.

Badge виден всегда (sidebar bottom + dashboard header). Без него
пользователь не отличит «нет движений на рынке» от «соединение умерло
2 минуты назад» — это нарушает observability контракт.

**Зачем явно зафиксировано.** EventSource по умолчанию уже re-connect'ится
с `Last-Event-ID` header — но мы НЕ устанавливаем `id:` в SSE
сообщениях, и наш сервер игнорирует Last-Event-ID на приёме. Без явного
ADR следующий разработчик может попытаться добавить replay-семантику и
сломать контракт. Per-subscriber queue с `maxsize=1000` + drop-oldest —
осознанное решение, не баг.

**Совместимость с ТЗ:** `Q3` latency SLO 90s P95 — single-connection
reconnect укладывается в это. `Q5` observability — `oi_sse_queue_full_total`
counter (M4.B.12) тригерит Grafana alert, если >0 в течение 5 минут.

---

### F28 · PG NOTIFY payload — keys-only policy (Pre-M4 ADR)

**Решение.** Триггеры `notify_new_oi_sample` и `notify_new_alert` пишут в
`pg_notify(channel, payload)` **только идентификаторы**, не полные строки:

```sql
-- oi_samples триггер
PERFORM pg_notify('oi_new_sample',
    json_build_object(
        'exchange', NEW.exchange,
        'canonical_symbol', NEW.canonical_symbol,
        'ts_exchange', NEW.ts_exchange
    )::text
);

-- alert_events триггер (когда decision='fire')
PERFORM pg_notify('oi_new_alert',
    json_build_object(
        'rule_id', NEW.rule_id,
        'exchange', NEW.exchange,
        'canonical_symbol', NEW.canonical_symbol,
        'ts_processed', NEW.ts_processed
    )::text
);
```

Полные строки fetched отдельным `SELECT` в API-процессе после получения
NOTIFY — keys-only payload идёт по NOTIFY, дальше API делает batch read
актуальных данных и публикует в SSE Hub.

**Зачем.** PostgreSQL ограничивает payload `pg_notify` 8000 байт; превышение
поднимает `ERROR: payload string too long`. Если триггер начнёт включать
notional, valuation_status, payload и пр. — мы ушли в truncation/error
risk без видимой деградации до production scale (большие строки начинаются
при ~30+ символьных канонических символах + полный JSON `extra`).

**Расширение payload запрещено** без явной F-записи. Кандидаты на
расширение должны:
1. Доказать, что 99-й перцентиль payload size остаётся <4 KB.
2. Бенчмарком показать, что добавление полей экономит >1 SELECT в hot
   path (если экономия меньше — добавление не оправдано).
3. Получить F-запись (`F<n>`) с обновлением этого ADR'а.

**Реализация.** Триггеры создаются в M4.B.1 (миграция `0010_pg_notify_triggers.py`);
до этого момента LISTEN/NOTIFY не используется. Текущий M3 alert engine
работает через polling `delivery_queue`.

**Совместимость с ТЗ:** `02 §5.3` уже содержит этот пункт со ссылкой
«финальная фиксация — `10 §5.5`» — F28 закрывает forward-reference.
`10 §5.5` обновляется одновременно с этой ADR.

---

### F23 · Settings hot-reload (Pre-M3 ADR)

**Решение:** в v1 (M3-M5) settings **не реализуем hot-reload**. Все процессы (`scheduler`, `api`, `tg_sender`, `cleanup`) читают `settings` table при старте; изменения через UI Settings (`05 §12`, M5) применяются только после `systemctl restart oi-tracker-<service>`.

**Обоснование.**

1. **Cost.** Hot-reload требует одного из:
   - SIGHUP handler в каждом сервисе + thread-safe настроечные структуры.
   - DB-poll watcher (раз в N секунд `SELECT updated_at FROM settings WHERE updated_at > $last`).
   - LISTEN/NOTIFY на `settings` table.
   - Все три добавляют тесты, observability, потенциальные race conditions при чтении settings во время цикла normalizer'а.

2. **Benefit в v1 минимален.** `Q4` фиксирует single-user. Restart всех 4-х сервисов = ~5 секунд downtime (`systemctl restart oi-tracker-{scheduler,api,tg_sender,cleanup}`). Пользователь редактирует settings из UI Settings (M5) от случая к случаю — `tg_chat_id` один раз, `tg_dry_run` при отладке, `exchange_whitelist` при инцидентах. Стоимость 5-секундного restart << стоимости hot-reload machinery + тестов.

3. **Что меняется per-key reload-cost-vs-restart:**
   | Key | Affects | Hot-reload bonus |
   |---|---|---|
   | `tg_chat_id` | tg_sender | none — restart 5s |
   | `tg_dry_run` | tg_sender, alert engine dispatcher | none |
   | `exchange_whitelist` | scheduler | none — instruments syncs every 5min anyway |
   | `min_oi_notional_default_usdt` | alert engine | none — re-evaluation every 30s |
   | `cooldown_default_sec` | alert engine | none |
   | `polling_interval_sec` | scheduler timer wheel | restart-required даже при hot-reload (timer wheel rebuild) |
   | `freshness_budget_now_sec` | alert engine | none |

4. **Risk откладывания:** zero. Если в M5+ появится потребитель, у которого 5s downtime — проблема (например, multi-user shared bot setup), реализуется отдельным PR. До тех пор YAGNI.

**Реализация (placeholder, чтобы не было сюрпризов в коде):**

- В `app/config.py` (Pydantic Settings) и `app/storage/repositories/settings_kv.py` явный комментарий: «settings frozen at process startup; restart required after UI write».
- UI Settings PUT endpoint (`10 §4.8`, M5) возвращает 200 OK + body `{"applied": false, "restart_required": ["scheduler", "tg_sender"]}` — пользователь видит, какие сервисы рестартить.
- Audit log `settings_audit` (`F15`) фиксирует `applied_at = NULL` пока процесс не перезапустится; перезапуск пишет `applied_at = NOW()` через startup hook.

**Reactivation criteria** (когда стоит вернуться):

1. Multi-user или shared-bot setup появится (downtime неприемлем).
2. Появится setting, меняющийся часто (например, dynamic blacklist на основе ML).
3. systemd restart времени станет > 30 секунд (вряд ли при single-process design).

**Совместимость с ТЗ:** `Q4` single-user — restart-required приемлем; `F15` UI Settings categories A/B/C сохраняются — write идёт в БД даже если процесс не подхватывает; `Q8` chat_id remains в БД (см. `F22` — throttle не зависит от reload механизма).

---

### F29 · trace_id propagation — log-correlation, no OTEL in v1 (Pre-M5 ADR)

**Решение.** Сквозной `trace_id` через 4 сервиса (scheduler / alert_engine /
tg_sender / api) реализуется как **log correlation field** через
`structlog.contextvars`, без OpenTelemetry / Tempo.

**Naming convention** (полная спека — `12 §5.4`):

| prefix | где генерируется | формат |
|---|---|---|
| `cycle-` | `scheduler/loop.py::_run_one_cycle` | `cycle-{exchange}-{uuid8}` |
| `replay-` | `app/tools/replay.py` | `replay-{uuid8}` |
| `api-` | FastAPI middleware per request | `api-{uuid8}` |
| `tg-` | `tg_sender` fallback (если payload._trace_id отсутствует) | `tg-{uuid8}` |

**Cross-process persistence.** structlog.contextvars живёт внутри одного
процесса. Между сервисами trace_id передаётся через `delivery_queue.payload._trace_id`
(JSONB-поле). tg_sender читает оттуда и rebinds на свой dispatch.

**Обоснование.**

1. **Cost / Benefit baseline.** OTEL Python SDK + Tempo backend = +2 deps,
   +отдельный systemd сервис, +порт, +конфиг. Single-user / single-host
   deployment не нуждается в distributed tracing tree. Достаточно
   group-by-trace_id в Loki.

2. **Forward compat.** `make_trace_id()` — единственная точка генерации.
   Когда понадобится OTEL (multi-host / multi-user), `trace_id` field
   reused как root span ID; `make_trace_id` заменится на
   `tracer.start_as_current_span().get_span_context().trace_id` без
   изменений в кодовых вызовах.

3. **Что НЕ покрывается v1:** parent/child span relationships, sampling
   policies, exemplars в Prometheus histograms. Если debugger spec потребует —
   отдельный F-record.

**Reactivation criteria** (когда поднимать OTEL):
- Multi-host deployment.
- Distributed tracing UI требуется (Tempo / Jaeger).
- Cross-service latency breakdown с авто-инструментацией http-клиентов.

**Реализация:** Pre-M5.2 (хелпер `make_trace_id`, bind/clear в 4-х
точках, `_trace_id` field в DeliveryRequest.payload).

**Совместимость с ТЗ:** `Q5` observability — log correlation через Loki
обеспечивает MTTR-debug требования; `12 §5.4` определяет full chain;
`02 §5.4` уже описывал bind/clear как forward-compat — F29 формализует.

---

### F30 · TCP loopback `/metrics` для Prometheus scrape (M5.A ADR)

**Решение.** Каждый long-running service oi-tracker ставит
`prometheus_client.start_http_server(port, addr="127.0.0.1")` на
loopback-интерфейс для Prometheus scrape. Распределение портов:

| Service | Port | Состояние |
|---|---|---|
| `oi-tracker-api` | `127.0.0.1:9100` | новый (Stage 2) |
| `oi-tracker-scheduler` | `127.0.0.1:9101` | уже wired (M2.A) |
| `oi-tracker-alert-engine` | `127.0.0.1:9102` | уже wired (M3.E) |
| `oi-tracker-tg-sender` | `127.0.0.1:9103` | уже wired (M3.C) |
| `oi-tracker-cleanup` | n/a | oneshot — push gateway / no-op |

API process отдельно сохраняет существующий `GET /metrics` через UDS
(внутренний дебаг через nginx upstream); TCP loopback — параллельный
endpoint **только для Prometheus**. Same global `prometheus_client.REGISTRY`
→ identical exposition.

**Обоснование.**

1. **UDS не поддерживается Prometheus scrape.** Prometheus конфигурится
   через `static_configs.targets: ["host:port"]` с TCP. UDS scrape
   потребовал бы либо exec scrape (custom shell), либо nginx-stream
   proxy UDS→TCP. Оба варианта добавляют операционную сложность ради
   косметики.

2. **Loopback-only — не нарушает `F14`.** F14 запрещает публичные TCP
   listeners; `127.0.0.1`-bind виден только локальному Prometheus,
   не доступен через nginx (нет server-block для этих портов) и не
   доступен из сети (network namespace не пропускает loopback наружу).

3. **Стандартный pattern prometheus_client.** `start_http_server` —
   стандартная утилита; не вводит новых deps, не блокирует event loop
   (запускается в отдельном thread), thread-safe.

4. **Один `REGISTRY` per process.** prometheus_client использует
   глобальный `REGISTRY` — все наши Counter/Gauge/Histogram объявлены
   модульно в `metrics.py` и автоматически попадают в exposition.
   `/metrics` через uvicorn (UDS) и `:9100/metrics` через
   start_http_server возвращают идентичный output.

**Альтернативы (отклонены).**

- **Pushgateway.** Подходит для batch/oneshot; не оптимально для
  long-running. Введёт ещё один service в стек.
- **Текстовый file exporter.** Append metrics в файл, node_exporter
  textfile collector подхватит. Лишний disk I/O + лаг до min-resolution
  Prometheus scrape (15s) теряется.
- **nginx UDS-stream proxy.** `stream` block с `proxy_pass unix:...`.
  Работает, но добавляет nginx config для каждого сервиса; debugging
  сложнее (network log + nginx log + uvicorn log).

**Реализация:** Stage 2 (`expose_metrics_http(port)` helper в
`app/observability/metrics.py`; ports переезжают в `Settings`;
api.lifespan получает hook).

**Совместимость с ТЗ:** `F14` (UDS-primary, no public TCP) — выполнен:
loopback-only ≠ public; `Q5` observability — Prometheus scrape
работает; `13 §3.x` systemd units не меняются (порты внутренние).

---

### F31 · M5 coverage closure через Path C (M5.C ADR)

**Решение.** M5 milestone закрывается при следующем coverage state:

| Слой | `14 §2.1` target | M5.C state | Verdict |
|---|---|---|---|
| Normalizer parsers | ≥ 90% | ~99% | PASS |
| Alert engine state_machine | ≥ 95% | 100% | PASS |
| **Alert engine evaluation_cycle** (decision orchestration) | n/a explicit | **99%** | new in M5.C |
| **Delivery producer** (dedupe + payload) | n/a explicit | **100%** | new in M5.C |
| **Observability logging_config** | n/a explicit | **98%** | new in M5.C |
| **TG dispatcher** | n/a explicit | **98%** | new in M5.C |
| Connectors per file | ≥ 80% | 84-92% | PASS |
| Storage repos | ≥ 85% | 17-33% | **DEFERRED to M6** |
| API endpoints | ≥ 80% | 0% (routes) | **DEFERRED to M6** |
| Frontend | ≥ 70% | 96.61% | PASS (M4.C.X.2) |
| Total backend | ≥ 80% | **66%** unit / **64%** with integration | **DEFERRED to M6** |

`pytest --cov-fail-under=65` фиксирует unit-baseline; любой regression
на покрытых путях ломает CI.

**Обоснование.**

1. **Decision-heavy code покрыт ≥98%.** `evaluation_cycle`, `producer`,
   `dispatcher`, `logging_config` содержат всю branching-логику
   аlert-pipeline. Bugs здесь = silent ошибки доставки. State
   machine 100%, normalizer 99%, connectors 84-92% — это business
   logic, и она тестирована.

2. **Repository CRUD wrappers тестируют asyncpg, не нашу логику.**
   `oi_samples_repository.insert` = `INSERT ... ON CONFLICT DO NOTHING`
   с `$N` binding — тестировать нечего сверх того, что Postgres
   гарантирует. Покрытие через unit-mocks даёт false sense of
   completeness.

3. **API routes требуют integration harness.** FastAPI TestClient +
   real DB fixture + auth context — это >20h работы. Корректнее
   spin off как M6 milestone с явной целью (≥80% routes), чем
   backfill наспех на M5 closure gate.

4. **Path A (literal 80%) blow timeline 5-7×.** Тесты под цифру
   (smokes на тривиальных CRUD) — низкая ROI. Path B (zero new
   tests + F-record) оставлял quick-wins на столе. Path C — гибрид:
   business logic покрыт, integration debt acknowledged + planned.

**Альтернативы рассмотрены и отвергнуты.**

- **Path A — literal 80% backfill.** Реалистично 20-30h: integration
  tests для всех 8 repos + 12 API routes + scheduler/loop. Smokes
  под цифру не находят bugs, добавляют maintenance burden.
- **Path B — F-record only, no new tests.** Быстро, но `evaluation_cycle`
  и `dispatcher` — критичные пути, реальные баги возможны при
  refactoring. Quick-win backfill (5-8h) меняет character тестов
  с «hopeful» на «verified».

**Реализация (закрыто 2026-05-02):**

- Новые unit-тесты:
  - `tests/unit/alert_engine/test_evaluation_cycle.py` (15 tests)
  - `tests/unit/delivery/test_producer.py` (расширен 14 tests)
  - `tests/unit/observability/test_logging_config.py` (8 tests)
  - `tests/unit/tg_sender/test_dispatcher_loop.py` (10 tests)
- `pyproject.toml`: `[tool.coverage.report] fail_under = 30 → 65`.
- M6 backlog (отдельный milestone, не входит в M5):
  - `tests/integration/test_repositories.py` — асинхронные fixtures
    с testcontainers PostgreSQL для всех repos (`alert_*`, `delivery_*`,
    `instruments`, `live_dashboard`, `oi_*`, `settings_kv`,
    `symbol_history`).
  - `tests/integration/test_api_routes.py` — FastAPI TestClient +
    pytest-asyncio + DB fixture + reverse-proxy mock для UDS.
  - `tests/integration/test_scheduler_loop.py` — supervisor +
    TaskGroup + connector pool через testcontainers.
  - Цель M6: total backend ≥ 80%, repos ≥ 85%, API routes ≥ 80%.

**Совместимость с ТЗ:** `14 §2.1` остаётся target'ом для M6;
`16_ROADMAP.md M5 DoD` обновлён с явной ссылкой на F31; `pytest
--cov-fail-under=65` защищает уже покрытые пути от регрессии.

---

### F32 · Settings PUT response — single-row, no restart-required envelope (M5.C ADR)

**Решение.** `PUT /api/v1/settings/{key}` возвращает upserted row из
`settings_kv` напрямую (raw shape: `key, value, updated_at`). НЕ
возвращаем envelope `{applied: false, restart_required: ["scheduler",
"tg_sender"]}` из placeholder в `10 §4.8` / `00 F23 §9.6`.

**Обоснование.**

1. **Single-user pragma `Q4`.** Оператор системы — один человек,
   который знает архитектуру и читает `13 §2`. Информация
   «изменение требует restart какой-сервис» доступна в:
   - F23 в `00_DECISIONS_LOG.md` (явный mapping setting → service);
   - `13 §3` systemd units list;
   - tooltip в UI (`/settings/{tab}` страница).

2. **Envelope не несёт safety-критичной нагрузки.** Если оператор
   забыл restart — сервис продолжает работать со старым значением,
   user видит расхождение в UI (которое читает свежее значение
   через `GET /settings`). Нет «silently broken» state.

3. **Audit-log пишется по-прежнему.** `settings_audit` содержит
   `(key, old_value, new_value, changed_at)` — оператор может
   reverse-engineer когда что менялось и был ли restart.

**Альтернатива (отвергнута).** Реализовать envelope ~30 LOC в
`api/routes/settings.py` + frontend update + tests. Single-user не
оправдывает инвестицию.

**Реализация:** уже shipped (M3.E). `app/api/routes/settings.py:188-205`
возвращает `{key, value, updated_at}` shape.

**Совместимость с ТЗ:** `Q4` (single-user) — выполнен; `F23`
(restart-required) — поведение сохранено; `10 §4.8` placeholder
переписан в Review этого F.

---

### F33 · ConnectorErrorEvent superseded by Prometheus emission (M5.C ADR)

**Решение.** `ConnectorErrorEvent` Pydantic-контракт (`05 §13`,
`app/domain/events.py:313`) остаётся как domain type для
forward-compat и symmetry, но **НЕ конструируется в production
path**. Forensic / monitoring замещается:

1. `oi_collector_request_total{exchange, status, error_type}` —
   counter (`12 §3.1`, M1.B.6), включает `error_type ∈ {timeout,
   http_error, parse_error, schema_drift, ...}`.
2. Structured logs через `structlog` с `error_type` + `exchange` +
   `trace_id` fields — searchable в Loki через `12 §5.4`.
3. M5.A.6 «exchanges» dashboard содержит панели `Top error types per
   exchange`, `Error rate trend`, `5xx vs network errors` —
   полноценная forensic картина.

**Никакого `connector_errors` hypertable** не создаётся.

**Обоснование.**

1. **Prometheus + Loki стек уже покрывает use-case.** Изначальный
   мотив `connector_errors` table — forensic queries по `error_type
   × exchange × time range`. Это PromQL запрос, не SQL, и Grafana
   уже подключена.

2. **Hypertable добавляет maintenance debt без value.** Compression
   policy, retention, partition rotation — всё это не нужно при
   Prometheus retention 90 дней (`12 §6.1`).

3. **Domain type сохраняется.** Если в M6+ возникнет need в SQL-side
   forensic (e.g. correlation с `oi_samples` через JOIN) — контракт
   готов, добавить writer-path тривиально.

**Альтернатива (отвергнута).** Wire `ConnectorErrorEvent` insertion в
all 12 connector adapters (~50 LOC + tests) + `connector_errors`
migration + retention policy. Total ~10h работы — without observable
user benefit при существующем dashboard стеке.

**Реализация:** Already shipped — production использует Prometheus
emission с M1.B.6.

**Совместимость с ТЗ:** `12 §3.1` (collector metrics) — выполнен;
`12 §5.x` (logs) — выполнен; `Q5` (observability) — выполнен через
metrics+logs стек, без БД.

---

### F34 · M6 integration harness — session container + per-test TRUNCATE+reseed (M6.A ADR)

**Решение.** Интеграционный test harness использует
`testcontainers[postgres]>=4.8` с образом
`timescale/timescaledb:latest-pg16`:

1. **Один `PostgresContainer` per pytest session.** Container start +
   `alembic upgrade head` амортизируются на всю session
   (~10-15s одноразово вместо per-test).
2. **asyncpg pool per test (function-scoped).** Изначально планировался
   shared session pool, но pytest-asyncio default loop scope =
   `function`, и asyncpg отвергает cross-loop usage с
   `InterfaceError("another operation is in progress")`. Per-test pool
   creation против уже поднятого контейнера — ~30ms, в бюджете.
3. **Per-test isolation через TRUNCATE+RESTART IDENTITY+CASCADE на всех
   schema-managed BASE TABLE** + reseed defaults (4 instruments + 6
   default alert rules per F18). `oi_5m / oi_15m / oi_30m /
   oi_quality_5m` — continuous aggregates, ре-материализуются от
   source hypertable, явный TRUNCATE не нужен.

**Обоснование.**

1. **Savepoint pattern не подходит.** Repo API подписан на
   `asyncpg.Pool` и каждый вызов берёт свежее соединение из пула;
   savepoint, открытый на одном соединении, не покрывает repo
   writes на других.
2. **Per-test container — слишком медленно.** ~10-15s × N тестов; для
   ~30 интеграционных тестов это +5 минут к suite, неприемлемо для
   локальной TDD-итерации.
3. **TRUNCATE на TimescaleDB ≥ 2.x работает на hypertable directly.**
   Не нужен detach/recreate цикл.

**Альтернативы (отвергнуты).**

- **Persistent dev DB.** Не reproducible, конфликт с локальной
  разработкой, нет CI-clean state.
- **pytest-xdist parallel workers с per-worker container.** Отложено
  до момента, когда test suite > 5 min — см. F-M6-D-deferral note в
  `tasks/development_plan.md` M6 section.

**Реализация:** `backend/tests/integration/conftest.py` (M6.A). Smoke
tests: `tests/integration/test_harness_smoke.py` 4/4 PASS, ~10s
включая container start.

**Совместимость с ТЗ:** `14 §2.1` (integration test scope), `Q6`
(Docker запрещён только в **production**, тесты — допустимо).

---

### F35 · psycopg2-binary в dev-deps для alembic в integration tests (M6.A ADR)

**Решение.** Добавлен `psycopg2-binary>=2.9,<3.0` в
`pyproject.toml [project.optional-dependencies] dev`. Production app
по-прежнему использует только `asyncpg` (`02 §2.1.1` ADR), `psycopg2`
никогда не попадает в production extras.

**Обоснование.**

1. **Alembic в test fixture требует sync engine.** `testcontainers`
   возвращает SQLAlchemy URL `postgresql+psycopg2://...`; alembic
   `upgrade head` через `subprocess.run(["uv", "run", "alembic", ...])`
   без `psycopg2-binary` падает с `ModuleNotFoundError`.
2. **Production-extras не затрагивается.** `pyproject` `[project]
   dependencies` не включает psycopg2; sync driver виден только при
   `uv sync --extra dev`.
3. **Binary wheel.** `psycopg2-binary` (не source `psycopg2`) — wheels
   precompiled, не требует `libpq-dev` на dev-машинах.

**Альтернатива (отвергнута).** Программно вызывать alembic API через
async-обёртку с asyncpg-DSN. Возможно, но требует patching alembic
internals; sync subprocess проще и матчится с CI / production deploy
сценарием.

**Реализация:** одна строка в `pyproject.toml` + `uv sync --extra dev`.
Verified by `tests/integration/test_migrations.py` PASS in 17s.

**Совместимость с ТЗ:** `02 §2.1.1` (production app — pure asyncpg),
`14 §5.2` (alembic round-trip test).

---

### F36 · Continuous aggregates пропускаются при per-test reset (M6.A ADR)

**Решение.** `_RESET_TABLES` в integration conftest содержит только
BASE TABLEs: `alert_events, alert_state, alert_rules, delivery_queue,
instruments, settings_kv, oi_samples`. Continuous aggregates (`oi_5m,
oi_15m, oi_30m, oi_quality_5m`) не TRUNCATE'аются.

**Обоснование.**

1. **CA — read-only matview**, реализуется политикой
   `add_continuous_aggregate_policy` от source hypertable.
2. **TRUNCATE source hypertable освобождает CA-материализацию.**
   Для тестов CA — read-only assertion target; их seed не нужен,
   а явный TRUNCATE на CA даже не разрешён TimescaleDB.
3. **Тесты, которым нужны CA-данные**, явно вызывают
   `CALL refresh_continuous_aggregate(...)` — pattern уже есть в
   alert_engine integration tests.

**Реализация:** F-M6-C realised in `tests/integration/conftest.py`.

**Совместимость с ТЗ:** `08 §6` (continuous aggregates spec).

---

### F37 · Multi-chat routing — per-rule chat with default fallback (M7 ADR)

**Решение.** Заменяем single-chat доставку (`settings.tg_chat_id`) на
per-rule routing с обязательным дефолтным чатом-fallback'ом:

1. **Новая таблица `tg_chats`** (`05 §8a` — добавляется в M7.A.3):
   `id SERIAL PK`, `chat_id_native TEXT UNIQUE NOT NULL`,
   `title TEXT NOT NULL`, `is_default BOOL DEFAULT false`,
   `is_active BOOL DEFAULT true`, `last_validated_at TIMESTAMPTZ`,
   `last_validation_error TEXT`, `created_at`, `updated_at`.
2. **Partial unique index** `WHERE is_default = true` гарантирует ровно
   один дефолтный чат на уровне БД (не приложения).
3. **`alert_rules.chat_id INT NULL REFERENCES tg_chats(id) ON DELETE
   RESTRICT`** — NULL означает «использовать default».
4. **`delivery_queue.chat_id_native TEXT NULL`** — снапшот destination
   на момент enqueue (производитель резолвит routing в FIRE-time, а не
   в send-time, чтобы изменения чатов между FIRE и send не ломали
   доставку).
5. **`settings.tg_chat_id` остаётся deprecated alias** на ≥1 milestone
   (плановое удаление M8+). API GET читает `chat_id_native` дефолтного
   чата; PUT — upsert'ит дефолтный.
6. **Sync getChat-валидация при создании чата** через `bot.get_chat()`
   с 5s timeout; маппинг ошибок: `not_found / forbidden / timeout /
   transient`. Background revalidation hourly с jitter ±300s и 10s
   inter-chat jitter (anti rate-limit).
7. **Per-chat throttle (F22) не трогаем** — leaky-bucket 1 msg/sec
   per chat_id уже корректен и переиспользуется автоматически.

**Обоснование.**

1. **Single-user, но не single-chat.** Sentinel use case: дашборд-чат
   для команды + личка владельца + 1–2 specialty-чата для отдельных
   групп инструментов / порогов. Cardinality единиц чатов, поэтому
   таблица оправдана; JSON-список в `settings_kv` создал бы атомарные
   проблемы при concurrent edit и не дал бы нормального FK на
   `alert_rules`.
2. **Per-rule routing, не per-symbol.** Per-symbol/exchange routing
   усложнил бы Alert rule editor без ясной ценности — symbol_filter
   уже даёт нужную точность; per-rule достаточно.
3. **Snapshot `chat_id_native` в `delivery_queue`** обеспечивает
   determinism при retry: между FIRE и send rule может потерять
   chat_id (admin переназначил), но сообщение всё равно уйдёт по
   первоначально запланированному адресу. Альтернатива
   (lazy-resolve в dispatcher) добавила бы JOIN на каждый send и
   open вопрос «кому слать, если rule сейчас указывает на другой
   inactive chat».
4. **Default fallback обязателен.** Single-user не должен переживать
   misconfiguration: если rule.chat_id отсутствует или inactive, есть
   гарантированный destination. Producer falls back **молча**, но
   эмитит метрику `oi_tg_chat_resolution_failed_total{reason}` для
   observability. Если default тоже отсутствует — producer **не**
   enqueue'ит (alert_event сохраняется как audit, доставка теряется
   с лог-error). Это допустимый failure mode в обмен на
   архитектурную простоту: альтернатива (создать «system» fallback
   по-умолчанию из env) усложняет lifecycle.
5. **Sync validation на create.** Telegram getChat быстрый (~200ms);
   фейл-фаст лучше, чем тихо принять mistyped chat_id и обнаружить
   через 15 минут на первом FIRE. Background revalidation — для
   обнаружения post-hoc проблем (admin удалил бота из чата).

**Альтернативы (отвергнуты).**

- **JSON-массив `tg_chats` в `settings_kv`.** Нет FK ⇒ нет
  invariant'а «rule ссылается на существующий chat». Concurrent
  edit без атомарности через JSONB UPDATE — сложно. Нет partial
  unique index для default. Отвергнуто.
- **`alert_rules.chat_id_native TEXT` напрямую (без `tg_chats`).**
  Дублирование (title/health/last_validated_at — куда? в settings?
  в каждом rule?). Нет SoT для "active chats" в UI. Отвергнуто.
- **Webhook / inline button-управление чатами через бот.** Нарушает
  Q4 (no public API surface), требует auth для предотвращения
  hijacking. UI через REST + audit log — стандартный путь.
- **Per-symbol / per-exchange routing** — over-engineering для
  single-user; symbol_filter в alert_rules покрывает 90% сценариев
  routing'а через сужение rule'ов вместо размножения чатов.

**Backward compatibility.**

Migration `0012` создаёт дефолтный chat из текущего
`settings.tg_chat_id` (если он установлен) и оставляет существующие
rules с `chat_id = NULL` — они продолжают слать в дефолтный
автоматически. Existing alert flow продолжает работать без ручных
действий после deploy.

**Downgrade — one-way data loss for non-default routings.**
`downgrade()` записывает chat_id_native дефолтного чата обратно в
`settings.tg_chat_id`, дропает FK + tg_chats. Любые rules, которые
были назначены на не-default chat, теряют routing — после
downgrade они снова пойдут в default. Документировано в migration
docstring per `CLAUDE.md §3.4` (one-way с обоснованием).

**Совместимость с ТЗ.** Override `10_DELIVERY_LAYER.md §2.6` («v1:
один чат, multi-chat — future work»); раздел переписывается в M7.C
с пометкой ADR-link на F37. F22 (throttle), Q8 (token in env, chat
in DB), C8 (single-user) — без изменений.

**Реализация:** Phase M7 (см. `tasks/todo.md` M7-блок).

---

### F38 · settings.tg_chat_id alias removed (M7.E follow-up, sunset accelerated)

**Решение.** Legacy alias `settings.tg_chat_id` (введён в F37 как deprecated с `Sunset: M8` RFC 8594 header) удалён сразу после M7-close, не дожидаясь календарного M8. Source of truth для destination Telegram chat — `tg_chats` table per §8a.

**Обоснование (sunset accelerated).**

1. **Single-user продукт без публичного API (`Q4`).** Sunset header — это контракт с внешними потребителями; на single-user продукте реальных потребителей за пределами оператора нет. Pre-flight grep по production code показал ноль runtime-консьюмеров `settings.tg_chat_id` за пределами alias-route — соблюдение 2-недельного calendar-окна было бы ceremonial.
2. **Нагрузка кода.** Alias держал ~70 строк (две branch-ветки в `get_one`/`put_one` + `TgChatId` Pydantic class + `chats_repo` import + headers + audit-log inline path). Pure dead code в hot path.
3. **Чистая ADR-граница.** F37 ввёл alias; F38 снимает его. Коммит-граница (`docs(m7)` close → `chore(m7.e)` alias removal) сохраняется чище, чем разнести cleanup на 2 недели вперёд в искусственный "M8".
4. **Recovery procedure для default chat остаётся.** Runbook `docs/13 §5.9` обновлён: вместо `PUT /api/v1/settings/tg_chat_id` рекомендован `POST /api/v1/chats` с `is_default=true` (либо direct DB INSERT).

**Что удалено.**

- `app/api/routes/settings.py`: branch-ветки `if key == "tg_chat_id":` в `get_one`/`put_one`, `TgChatId` Pydantic class, ключ `"tg_chat_id"` из `_KEY_VALIDATORS`, `chats_repo` import, `Response` параметр (был использован только для Deprecation/Sunset/Link headers).
- `docs/05 §12.3`: строка `tg_chat_id` из таблицы known-keys.
- `docs/13 §5.9`: легаси `PUT /settings/tg_chat_id` curl-пример заменён на `POST /api/v1/chats`.
- `docs/16_ROADMAP.md`: пункт «Удаление legacy `settings.tg_chat_id` alias (M8+)» из post-M7 future-work.

**Совместимость с ТЗ.**

- F37 sunset commitment honored, but timeline accelerated — "M8" milestone в roadmap не существует как отдельный milestone-record, был placeholder для следующего release. Логика "next major release window" соблюдена.
- TG-sender legacy fallback (env `TELEGRAM_CHAT_ID` для pre-M7 queue rows без `chat_id_native`) — не затронут. Это отдельный fallback-path в `tg_sender_main._resolve_startup_fallback_chat`.

**Реализация:** M7.E follow-up (см. `tasks/todo.md` M7 Review section).

---

### F39 · API route prefixes выровнены под `/api/v1/*` per spec (hotfix)

**Контекст.** Settings-страница в UI не открывалась: `TabAlerts` рендерится первым (default tab), он зовёт `useAlertRules` → `GET /alert-rules`, ответ — HTML 200 (SPA index.html), `apiFetch` парсит как text, `setRules(undefined)`, `[...rules].sort()` бросает → React render fail.

**Корень.** `backend/app/api/routes/alert_rules.py` использовал `prefix="/alert-rules"`, `instruments.py` — `prefix="/instruments"`. Спека `docs/10_DELIVERY_LAYER.md §4.6/§4.7/§4.9/§4.10` требует `/api/v1/alert-rules` и `/api/v1/instruments`. nginx (`/etc/nginx/sites-available/oi-tracker`) проксирует на UDS только `/api/` — остальное падает в SPA-fallback `try_files $uri $uri/ /index.html`. Routes были собраны на путях, не достигаемых через nginx.

**Решение.** Привести код к спеке (`CLAUDE.md §0`: корректность > краткость; `§1`: docs приоритетнее кода).

**Изменения.**

- `backend/app/api/routes/alert_rules.py`: `prefix="/alert-rules"` → `prefix="/api/v1/alert-rules"` + docstring.
- `backend/app/api/routes/instruments.py`: `prefix="/instruments"` → `prefix="/api/v1/instruments"` + docstring.
- `frontend/src/hooks/useAlertRules.ts`: `apiFetch("/alert-rules")` → `apiFetch("/api/v1/alert-rules")`; `apiFetch(`/alert-rules/${id}`)` → `apiFetch(`/api/v1/alert-rules/${id}`)`.

**Верификация.**

```
GET https://.../api/v1/alert-rules        → 200 application/json (6 rules)
GET https://.../api/v1/instruments?limit=1 → 200 application/json
GET https://.../alert-rules               → 200 text/html (SPA fallback, ожидаемо)
```

**Тесты.** Существующие unit/integration tests не били по HTTP-пути напрямую (`backend/tests/*` использует repository layer и `TestClient` без захардкоженного prefix); `grep` подтвердил отсутствие задетых путей. Frontend тестов на `useAlertRules` нет.

**Технический долг для отдельного PR.**

1. Defensive guard в `frontend/src/api/client.ts`: при `Content-Type != application/json` считать ответ ошибкой, не парсить как text. Защищает от любых будущих nginx/proxy misroute (а не только от текущего класса).
2. `frontend/src/api/types.gen.ts` всё ещё содержит сгенерированные пути `/instruments*` без `/api/v1` — пересобрать через `pnpm gen:api`, когда будет открыт PR на этот hook.
3. Audit остальных backend route-prefix'ов: `/api/v1` consistent; нет ли других «голых» prefix'ов, которые ещё не зацепили UI.

**SLO impact.** Нулевой — restart oi-tracker-api ~2s, soft, всё в одной транзакции nginx-keepalive pool.

---

### F40 · Sunset consensus signal (M8 Phase A)

**Контекст.** ConsensusSignal был специфицирован в `Q2` и реализован в M3.E (`signals/consensus.py`, sentinel `<consensus>` в `F26`, JSONB-параметр `consensus_count` через `F24`, UI-slider `consensus_min_exchanges`). Однако TG-template `consensus_alert` (`docs/10 §3.3`) **никогда не был залит** в `backend/app/tg_sender/templates.py` (`templates.py:4` явно отметка-placeholder). Каждое FIRE consensus-правила доходило до `delivery_queue.enqueue`, рендерилось dispatcher'ом, падало в `unknown_template` и `mark_failed`. Пользователь ни разу не видел consensus-сообщения в Telegram (см. сессию M8 уточнений).

**Решение.** Полностью выпилить consensus-фичу из активного пайплайна. Один алерт = одна биржа = одно сообщение в Telegram. Группировка по символу — отдельная фича для другого продукта/итерации.

**Изменения (Phase A, PR #1).**

- **DDL — миграция `0013_drop_consensus.py`:**
  - `DELETE FROM alert_state WHERE rule_id IN (SELECT id FROM alert_rules WHERE rule_type='consensus')` — порядок важен, FK `alert_state.rule_id → alert_rules.id`.
  - `DELETE FROM alert_rules WHERE rule_type = 'consensus'`.
  - `DELETE FROM settings_kv WHERE key = 'consensus_min_exchanges'`.
  - Downgrade — no-op (документировано как irreversible data loss, CLAUDE.md §3.4).
- **Backend code:**
  - `domain/alerts.py`: убрана `RuleType.CONSENSUS`; sunset-note в docstring `RuleType`.
  - `alert_engine/signals/__init__.py`: убрана `ConsensusSignal` из `_REGISTRY`.
  - `alert_engine/signals/consensus.py`: **удалён**.
  - `delivery/producer.py`: удалён `_format_payload_consensus`; убран `RuleType.CONSENSUS: "consensus_alert"` из `_TEMPLATE_BY_RULE_TYPE`; убрана ветка `if rule.rule_type is RuleType.CONSENSUS` в `enqueue`.
  - `api/routes/settings.py`: удалён `ConsensusMinExchanges` validator + ключ `consensus_min_exchanges` из `_KEY_VALIDATORS`.
  - `storage/repositories/oi_bars.py`: удалён `get_bars_grouped_by_symbol` (был только для ConsensusSignal).
- **Frontend:**
  - `components/settings/TabAlerts.tsx`: удалён slider «Консенсус: минимум бирж» + `useSetting<number>("consensus_min_exchanges", 3)`; обновлён header-комментарий.
- **Tests:**
  - Удалён `tests/unit/alert_engine/test_consensus_signal.py`.
  - Удалены 3 теста из `tests/unit/delivery/test_producer.py` (consensus payload + dispatch).
  - Удалён `test_consensus_fires_to_delivery` из `tests/integration/alert_engine/test_full_pipeline.py`; заголовочный docstring обновлён («4 active signal types»).
- **Docs:**
  - `01_PRODUCT_SPEC.md`, `02_ARCHITECTURE.md`, `04_TIME_MODEL.md`, `08_TIME_SERIES_STORAGE.md`, `09_ALERT_ENGINE.md`, `10_DELIVERY_LAYER.md`, `13_OPERATIONS.md`, `14_TEST_STRATEGY.md`, `16_ROADMAP.md`, `18_DESIGN_SYSTEM.md`, `README.md` — удалены / обновлены упоминания consensus.

**Что сознательно оставлено.**

- Исторические `alert_events.exchange='<consensus>'` строки — аудит-журнал прошлых FIRE'ов, retention 90 дней (`08`) автоматически их вычистит. Удаление руками — потеря истории своих же действий.
- Prometheus-лейбл `rule_type="consensus"` в `oi_alert_engine_signal_candidates_total` — будет всегда 0 (новых candidates нет). Не удаляю, чтобы не сломать существующие Grafana-панели; cosmetic cleanup отдельной задачей при ревизии M5 дашбордов.

**Supersedes.**

- **`Q2` (signal types):** «5 типов» → 4 типа (threshold, confirmed, divergence, exchange-health).
- **`F26` (Consensus exchange sentinel `<consensus>`):** не используется новым кодом; sentinel значение в исторических строках сохраняется как археология.
- **`F24` (alert_rules.extra)** — частично: `consensus_count` ключ больше не валиден; divergence + exchange_health параметры остаются.

**SLO impact.** Положительный — устраняется бессмысленный поток `tg_failed_total{template=consensus_alert,reason=unknown_template}`, чище dashboards.

**Migration impact.** Если в проде есть пользовательские consensus-правила (не дефолтные — F18 их не создаёт), они будут удалены без возможности восстановления. На момент решения у пользователя такие не настроены (подтверждено в сессии M8).

---

### F41 · Per-exchange routing groups (M8 Phase B)

**Контекст.** F37 дал per-rule routing — оператор мог прописать в каждом alert-rule свой `chat_id`. На практике пользователь хочет **per-exchange routing**: «алёрты с {Aster, Hyperliquid} → чат A; с {Binance, OKX, Bybit, Bitget, Gate} → чат B + thread N в форумной группе». Делать это через rule.chat_id потребовало бы дублирования каждого правила N раз — не масштабируется. Threads (форумные топики Telegram) ещё и не поддерживались никак — `delivery_queue` не имел колонки `message_thread_id`, dispatcher не пробрасывал параметр в `bot.send_message`.

**Решение.** Добавить поверх F37 новую сущность — **routing group** = (имя, чат, опциональный thread, набор бирж). Resolver выбирает destination в порядке: rule.chat_id (explicit override) → group по бирже алерта → default chat. Поведение F37 сохраняется (rule.chat_id побеждает группу) — миграционно безопасно.

**DDL — миграция `0014_tg_routing_groups.py`:**

- `tg_routing_groups (id PK, name UNIQUE, chat_id FK→tg_chats(id) ON DELETE RESTRICT, message_thread_id BIGINT NULL, is_active, last_send_error TEXT NULL, last_send_error_at TIMESTAMPTZ NULL, created_at, updated_at)`. Partial index `WHERE is_active=true` для быстрого resolver-пути.
- `tg_routing_group_exchanges (group_id FK→tg_routing_groups CASCADE, exchange TEXT, PK(group_id, exchange), UNIQUE(exchange))`. **UNIQUE на exchange** — биржа максимум в одной группе. Одна индексная lookup в hot-path resolver'а, без агрегации.
- `delivery_queue ADD COLUMN message_thread_id BIGINT NULL` — снапшот thread, как `chat_id_native` для chat (F37). Детерминированные ретраи: даже если группа меняется между enqueue и send, dispatcher шлёт на исходный (chat, thread).

**Resolver order (`app/delivery/chat_resolver.py:resolve_route`):**

1. `rule.chat_id` задан + чат активен → `(chat.chat_id_native, None)`. Rule-level routing никогда не несёт thread.
2. `tg_routing_groups.find_by_exchange(candidate.exchange)`:
   - группа активна + её чат активен → `(chat.chat_id_native, group.message_thread_id)`.
   - группа inactive → fallback в default, метрика `oi_tg_chat_resolution_failed_total{reason=group_inactive}`.
   - группа активна, но её чат inactive → fallback, метрика `{reason=group_chat_inactive}`.
3. Default chat (`tg_chats WHERE is_default=true AND is_active=true`) → `(chat.chat_id_native, None)`.
4. Default отсутствует → `NoDefaultChatError` (как было в F37).

**Dispatcher.** `bot.send_message(chat_id=..., text=..., message_thread_id=delivery.message_thread_id)`. aiogram пропускает `None` корректно. Throttle keyed по `chat_id` — Telegram per-chat лимит per-chat, не per-thread; одна группа = один chat = одно ведро.

**Async-валидация thread_id.** Telegram не даёт sync `getForumTopic` без админ-прав. Поэтому: thread_id принимается как-есть на create/patch. На permanent-failure отправки (`TelegramBadRequest`/`TelegramForbiddenError`) dispatcher вызывает `tg_routing_groups.find_by_chat_native_thread(chat, thread)` и пишет ошибку в `last_send_error` группы. UI показывает badge `send error` в карточке группы, оператор чинит вручную.

**REST API** (`/api/v1/routing-groups`, rate-limited 5/min/IP per F15):
- `GET /` — list всех групп (включая inactive).
- `POST /` — создать. 422 на конфликт `exchange already in another group` с echo offending списком.
- `PATCH /{id}` — частичное обновление; `exchanges` (если передан) replace-семантика; `message_thread_id` tri-state (omit/value/null).
- `DELETE /{id}` — hard-delete + cascade на exchange-assignments. (Не soft-delete как у chats — группы дешевле пересоздать.)

Audit-log — каждое мутирующее действие пишется в `settings_audit_log` с ключом `routing_group:{action}:{id}`, как и chats.

**UI** (`Settings → Telegram`). Новая секция «Маршрутизация по биржам» под существующей секцией Telegram-чатов. Карточки групп: name input, chat picker (select из `useTgChats`), thread input (numeric, optional), exchange chip-multiselect (12 чипов; занятые другими группами — disabled с tooltip). Кнопки Save/Disable/Delete. Badge `send error` если group ловила ошибку отправки.

**Что сознательно НЕ делаем:**
- Не enforce'им (chat_id, message_thread_id) UNIQUE — оператор может создать две группы с одним и тем же thread (config-bug, не схемный invariant). При reverse-lookup в dispatcher берём первую match'нувшуюся.
- Не делаем dry-run для группового маршрута. «Send test message» в Telegram tab остаётся per-chat.
- Не реализуем per-symbol группы (оператор не просил).

**Extends, не отменяет F37.** rule.chat_id остаётся explicit override. Существующие rule.chat_id-настройки продолжают работать. Migration purely additive.

**SLO impact.** Resolver: +1 lookup `find_by_exchange` (один индекс UNIQUE) per FIRE на группированных биржах. Производственный профиль (~1–10 FIRE/min) не чувствует.

---

### F42 · Bitmart replaces Bitunix — 12-я биржа восстановлена

**Контекст.** F20 (2026-05-01) задеферил Bitunix из M2 scope: public API не отдаёт OI (`/futures/market/tickers` без `openInterest`, `/futures/market/open_interest` → `Parameter error`, OI закрыт за authenticated review-based access). Production работал на 11 биржах вместо 12 как фиксировал `C17`. Reactivation criteria (Bitunix откроет public OI) не выполнены и в обозримой перспективе не предвидятся.

**Решение** (2026-05-04). 12-я биржа = **Bitmart**. Bitunix permanently removed from roadmap.

**Обоснование.** Live capture `https://api-cloud-v2.bitmart.com/contract/public/details` подтвердил: один bulk endpoint возвращает ВСЕ futures с полями `symbol`, `product_type`, `base_currency`, `quote_currency`, `contract_size`, `open_interest`, `open_interest_value`, `index_price`, `last_price`, `funding_rate`, `status`, `expire_timestamp` (0 = perpetual). Public, без auth. Pattern идентичен Binance/Bitget — bulk-tickers с OI + valuation + price + filter metadata в одном запросе.

**Что меняется:**
- `C17` «12 бирж» восстановлен по факту: Binance, Bybit, OKX, Bitget, Gate.io, MEXC, KuCoin, HTX, Hyperliquid, Aster, **Bitmart**, XT.
- F20 помечен superseded; spec `11_EXCHANGE_ADAPTERS/bitunix.md` удалён; новый spec `11_EXCHANGE_ADAPTERS/bitmart.md`.
- `connector factory` registry: запись `bitunix` (которой никогда не было — был только TODO-комментарий) → `bitmart` с реальным `BitmartConnector`.
- Bitunix не остаётся как dormant-13-я — никаких "архитектурно поддерживаемых 13 бирж", чтобы не плодить dead docs. Если Bitunix когда-то откроет OI — открывается новый `Fxx` add-as-13th.

**Bitmart specifics:**
- `oi_unit_hint = "contracts"` (не `base_asset` как было в неправильном bitunix.md spec'е; `"contracts"` уже поддерживается pipeline для XT/OKX/Gate.io/MEXC). `instrument.contract_size = bitmart_contract_size` (per-symbol); `oi_coins = open_interest × contract_size × multiplier(=1)` per pipeline Stage 3.
- `open_interest_value` provided биржей → `valuation_status = "authoritative"`. Без fallback на estimate (поле всегда присутствует в наблюдённых ответах).
- Filter perpetual: `status == "Trading" AND quote_currency == "USDT" AND expire_timestamp == 0 AND product_type == 1`.
- Symbol naming: `BASEUSDT` без разделителя (как Binance) — `canonical_symbol` derive стандартный.
- Rate limit: 10 req/sec/IP (консервативно, peer-aligned), bulk pattern = 1 request / poll cycle.

**Lesson reapplied** (`tasks/lessons.md:connector-spec-vs-reality`). Spec `bitmart.md` пишется ПОСЛЕ live capture endpoints, не до. Применили к этой замене: `curl /contract/public/details` сделан в plan-фазе, поля проверены, **только потом** spec написан.

**Affected files** (одна PR-атомарная замена):
- docs: `01_PRODUCT_SPEC.md`, `02_ARCHITECTURE.md`, `06_INSTRUMENT_REGISTRY.md`, `07_NORMALIZER.md`, `11_EXCHANGE_ADAPTERS/{bitunix.md→bitmart.md}`, `11_EXCHANGE_ADAPTERS/_BASE_CONNECTOR.md`, `18_DESIGN_SYSTEM.md`, `README.md`.
- code: `backend/app/exchanges/bitmart.py` (new), `factory.py`, `tg_sender/templates.py`.
- tests: `backend/tests/contract/bitmart/`, `backend/tests/unit/exchanges/test_bitmart.py`, `test_producer.py`.
- frontend: `styles/tokens/exchange-hues.ts` (rename slot bitunix=200 → bitmart=200; teal-blue остаётся, перцептуально-разнесённый от kucoin=180/mexc=220, ближе к Bitmart-брендовому teal чем альтернативы).
- tooling: `.claude/agents/exchange-adapter-builder.md`, `.claude/commands/adapter.md`, `.claude/settings.json`.
- tasks: `tasks/{todo,development_plan,lessons}.md`.

**Migration / data.** DDL без изменений (oi_samples exchange-agnostic). Production rows с `exchange='bitunix'` отсутствуют (connector никогда не запускался). Никакой Alembic миграции.

---

### F43 · Persistance whitelist для `alert_events` (audit-log noise sunset)

**Контекст** (2026-05-06). Production-замер `oi_tracker` базы: 81 GB суммарно, из них `alert_events` = 71 GB / 88%; 180 M строк за 3 дня (~25–30 GB/день полным днём). Распределение по `decision`:

| decision | rows / 3 дня | доля |
|---|---:|---:|
| `late_data_skip` | 168.1 M | 93.4% |
| `insufficient_history` | 6.1 M | 3.4% |
| `below_min_oi` | 4.8 M | 2.7% |
| `not_crossed` | 1.1 M | 0.6% |
| `suppress` | 4.0 K | <0.01% |
| `no_transition` | 4.7 K | <0.01% |
| `fire` | 2.2 K | <0.01% |
| `quality_gate_fail` | 0 | — |
| `resolve` | 0 | — |

99.5%+ объёма — частотный noise freshness-фильтра (`late_data_skip`) и bootstrap'ов (`insufficient_history`, `below_min_oi`), по которым достаточно агрегатной метрики. Спека `09 §10.1` требовала «**Каждое** решение пишется в `alert_events`» — это решение пересматривается.

**Решение.** Persistance whitelist:

```python
AUDIT_PERSISTED_DECISIONS = frozenset({
    AlertDecision.FIRE,
    AlertDecision.SUPPRESS,
    AlertDecision.RESOLVE,
    AlertDecision.QUALITY_GATE_FAIL,
})
```

В `alert_events` пишутся **только** decisions из этого набора. Остальные пять (`late_data_skip`, `insufficient_history`, `below_min_oi`, `not_crossed`, `no_transition`) → counter-only через `oi_alert_engine_decisions_total{rule_id, decision}`, который **остаётся неизменным** и инкрементируется по всем decisions без исключения.

**Принцип.** Audit-таблица хранит факты, **требующие ручного разбора**: алерт ушёл (FIRE), алерт подавили (SUPPRESS — smart cooldown), алерт зарезолвили (RESOLVE), сработал quality gate (QUALITY_GATE_FAIL — редко, диагностически ценно). Counter — для частотных decisions, по которым нужна только агрегатная картина «сколько чего было». SLO `e2e_latency` уже снимается через histogram `oi_alert_e2e_latency_seconds`, не требует event-row.

**Что НЕ меняется.**
- State machine logic — все decisions по-прежнему вычисляются, `alert_state` (cooldown, last_eval) пишется как раньше.
- `oi_alert_engine_decisions_total{rule_id, decision}` инкрементируется по **всем** 9 decisions — observability сохранена в полном объёме.
- `delivery_queue` — только `FIRE` идёт в TG, как и раньше.
- API `/api/v1/alerts?decision=…` — литералы всех 9 decisions остаются в enum для backward-compat (graceful degradation: фильтр на noise-decision вернёт ноль строк после релиза).

**Прогноз объёма после F43.** Полный день при whitelist = `fire (2.2K) + suppress (4.0K) + resolve (0) + quality_gate_fail (0)` ≈ 6 K строк/день вместо 60 M. То есть ~99.99% drop в audit-таблице. 90-дневный retention: ~12 GB вместо ~2.7 TB.

**Что НЕ покрыто этой записью.**
- Очистка существующих 168 M строк `late_data_skip` (отдельный шаг — `DROP CHUNK` или фильтрующий `DELETE` + `VACUUM`).
- Columnstore policy на `alert_events` (отдельный шаг — `ADD COLUMNSTORE POLICY ... compress_after '7 days'`, симметрично `oi_samples`). Полезен и после whitelist'а — для retention'а fire/suppress/resolve.

**Affected files.**
- docs: `09_ALERT_ENGINE.md §10.1, §10.3`, `05_DATA_CONTRACTS.md §AlertDecision`, `04_TIME_MODEL.md §7.2`.
- code: `backend/app/domain/alerts.py` (новая константа `AUDIT_PERSISTED_DECISIONS`), `backend/app/alert_engine/evaluation_cycle.py::_evaluate_rule` (фильтр перед `bulk_append`).
- tests: `tests/integration/alert_engine/test_full_pipeline.py`, `tests/unit/alert_engine/test_*` — переключить assert'ы на counter, добавить table-driven тест на whitelist.

**Migration / data.** DDL без изменений. Существующие строки нетронуты этим решением (см. отдельный шаг очистки выше).

---

### F44 · Columnstore policy на `alert_events` (audit-log compression)

**Контекст** (2026-05-06). Шаг 2/3 audit-log cleanup'а после F43. F43 убрал ~99.99% будущей записи в `alert_events`, но (а) на новых тонких чанках всё равно полезен columnstore для долгого retention'а fire/suppress; (б) физический rewrite чанков через `compress_chunk()` — единственный способ вернуть диск **без** `VACUUM FULL` (который требует эксклюзивный lock на часы). Шаг 3 (cleanup существующих 168M noise-rows) опирается на F44: после `DELETE noise FROM <chunk>; SELECT compress_chunk(<chunk>);` — диск возвращается, `VACUUM FULL` не нужен.

**Решение.** Симметрично F5 (`oi_samples`): включаем native columnstore на `alert_events`.

```sql
ALTER TABLE alert_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'rule_id, exchange',
    timescaledb.compress_orderby   = 'ts_processed DESC'
);
SELECT add_compression_policy('alert_events', INTERVAL '7 days');
```

**Обоснование параметров.**
- `segmentby = 'rule_id, exchange'` — соответствует префиксу существующего индекса `idx_alert_events_lookup (rule_id, exchange, canonical_symbol, window, ts_processed DESC)` и кардинальности audit-чтений (UI Alerts page фильтрует по rule_id/exchange чаще, чем по symbol). Не добавляем `canonical_symbol` в segmentby — кардинальность ~3000 символов × 12 ex × 6 правил = 216K сегментов / чанк → compression ratio падает.
- `orderby = 'ts_processed DESC'` — recency-biased reads (UI «последние алерты»), симметрично `oi_samples`.
- `compress_after = INTERVAL '7 days'` — F5 precedent. Hot chunk остаётся uncompressed первые 7 дней (живые insert'ы быстрые), потом columnstore.

**Что НЕ меняется.**
- DDL базовой таблицы `alert_events` (схема, индексы, retention 90 дней).
- Read-paths `query_alerts`/`count_alerts` — TS прозрачно читает compressed chunks.
- F43 whitelist — продолжает работать как раньше.

**Эффект на диск.**
- Сразу после деплоя: **0 GB освобождено** (policy job — schedule 12h, впервые сожмёт May 4 чанк около May 11 при `compress_after '7d'`).
- На новых чанках после F43 (~2K rows/день): compression тривиальная, belt-and-suspenders для долгого retention'а.
- На существующих 71 GB: реальное возвращение диска происходит при ручном `compress_chunk()` в шаге 3 (физический rewrite чанка).

**Sequencing с шагом 3.**
- F44 → шаг 3 в течение пары дней (отдельным PR).
- Шаг 3 необходимо запустить **до ~May 11** — иначе policy job сожмёт 168M noise rows впустую (CPU), и DELETE в шаге 3 будет идти через медленный auto-decompress→delete→recompress цикл.
- Path forward в шаге 3: для каждого существующего чанка — `DELETE noise → compress_chunk()`. Возвращает диск без `VACUUM FULL`.

**Affected files.**
- docs: `08_TIME_SERIES_STORAGE.md` (compression-сводка), `09_ALERT_ENGINE.md §10.3` (упоминание compression effect).
- migration: `backend/alembic/versions/0015_alert_events_columnstore.py` (новая, reversible).
- code: нет (read-paths прозрачны).
- tests: проверка `alembic upgrade head` / `downgrade -1` через testcontainers harness.

**Migration / data.** Reversible. Downgrade: `remove_compression_policy(if_exists)` → `decompress_chunk` для всех чанков → `ALTER TABLE ... compress=false`. Pattern из `0002_oi_samples_hypertable.py:downgrade()`.

---

### F45 · One-shot cleanup of historical noise rows в `alert_events`

**Контекст** (2026-05-06). Шаг 3/3 audit-log cleanup'а. После F43 (whitelist) + F44 (columnstore) на production оставалось 71 GB исторических pre-F43 noise rows (168M строк за 3 дня). Compression policy job впервые сработал бы около 2026-05-13 — без cleanup'а потратил бы CPU на сжатие будущих DELETE-кандидатов.

**Disk constraint at execution time.** На момент запуска: 7.8 GB свободно из 154 GB (95% used). Path A (DELETE+compress per chunk) требовал ~25 GB WAL — невозможно без переполнения. Path реализован: hybrid «preserve audit → drop closed chunks → restore audit → DELETE noise on hot chunk → VACUUM».

**Procedure (one-shot, не recurring).**

```sql
-- 1. Preserve audit-valuable rows (~6.4K строк)
CREATE TABLE alert_events_preserved_audit AS
  SELECT * FROM alert_events
  WHERE decision IN ('fire','suppress','resolve','quality_gate_fail');

-- 2. Drop closed chunks (May 4 + May 5) — instant, ~0 WAL, −53 GB
SELECT drop_chunks('alert_events', older_than => '<HOT_CHUNK_START>'::timestamptz);

-- 3. Restore preserved audit для удалённых дней
INSERT INTO alert_events
  SELECT * FROM alert_events_preserved_audit
  WHERE ts_processed < '<HOT_CHUNK_START>'::timestamptz;

-- 4. DELETE noise из hot chunk (large WAL, но free space уже есть)
BEGIN;
DELETE FROM alert_events
  WHERE ts_processed >= '<HOT_CHUNK_START>'::timestamptz
    AND ts_processed <  '<HOT_CHUNK_END>'::timestamptz
    AND decision NOT IN ('fire','suppress','resolve','quality_gate_fail');
COMMIT;

-- 5. VACUUM (без FULL — диск возвращается через compress_chunk policy на May 14)
VACUUM (ANALYZE) alert_events;
```

**Effect (production run).**
- DB size: 81 GB → 30 GB (−51 GB instant).
- `alert_events`: 71 GB → 19 GB (hot chunk dead tuples remain reusable; finalized to ~few MB при policy compress на 2026-05-14).
- Audit: 100% preserved (6.4K строк fire/suppress/resolve/quality_gate_fail сохранены в восстановленных May 4/5 чанках + остались в May 6 чанке).
- F43 деплой подтверждён: noise decisions counter инкрементируется, в `alert_events` пишутся только whitelist (fire/suppress).

**Что НЕ покрыто.**
- Hot chunk (May 6) физически 19 GB до policy compress на 2026-05-14. Acceptable — PG переиспользует pages для будущих writes; на 2026-05-14 columnstore physical rewrite reclaim'ит всё.
- Sidecar `alert_events_preserved_audit` оставлен как safety net на ~неделю; drop по необходимости (`DROP TABLE alert_events_preserved_audit;`).

**Affected files.**
- docs: `13_OPERATIONS.md` (новый раздел «Audit-log historical cleanup» — runbook).
- code: нет (one-shot SQL ops, не Python tool — простота per CLAUDE.md §3.1).
- migration: нет (data fix, не schema change).

**Migration / data.** Не reversible (noise rows утеряны навсегда — это и есть цель). Audit-rows preserved через sidecar. Side table `alert_events_preserved_audit` сохраняется до явного drop'а.

---

### F46 · Hyperliquid WS streaming — observer-only (hot lane PR2a)

**Контекст** (2026-05-11). Расследование инцидента 10.05 (HL/S, скачок >5% в окне 14:30-16:30 UTC без TG-уведомления) обнажило две проблемы:

1. **Транспортная задержка.** Hyperliquid опрашивается через REST POST `/info type=metaAndAssetCtxs` раз в 60 с. Latency «тик биржи → `oi_samples`» = до 60 с (avg 30 с). Для остальных бирж это приемлемо, но Hyperliquid не отдаёт per-asset `ts_exchange` в payload (§10.3 спеки), поэтому мы используем `fetched_at` (clock сервера) — любое транспортное ускорение бьёт 1-в-1 в качество `ts_exchange`.
2. **Семантика окон.** Tumbling 5m bucket'ы (CA `oi_5m`, `08 §3.2`) прячут скачок, целиком уместившийся внутри одного bucket'а: prev.close и current.close могут оба оказаться post-spike, Δ ≈ 0%, alert не срабатывает. Это системная проблема, но **для Hyperliquid решаем точечно** через hot lane со sliding-window Δ.

**Решение PR2a.** Параллельный WS-таск без изменения существующего data flow:

- WebSocket connection на `wss://api.hyperliquid.xyz/ws`, подписка `activeAssetCtx` на все активные перпы.
- Per-symbol in-memory ring buffer (`app/exchanges/_ring_buffer.py`, retention=32 мин) — каждый WS-фрейм с валидным `markPx` пишется в ring как canonical `oi_coins` (multiplied для k-prefix токенов: kPEPE x 1000 и т.п.).
- Resilience: exp backoff reconnect (1s→30s cap с jitter), heartbeat ping каждые 30s, stall watchdog (no message >60s → форс reconnect через TaskGroup unwind), per-symbol 1s debounce (HL пушит до ~2 Hz на активных рынках).
- TaskGroup-supervised под существующим pattern'ом `app/scheduler/supervisor.py`. Падение WS-таска изолировано — не валит ни остальные коннекторы, ни даже REST-путь Hyperliquid.
- **Поведение `fetch_snapshot`/`fetch_prices` НЕ меняется** — никто не читает из ring в PR2a. Цель: собрать боевые метрики (message rate, reconnect частоту, payload shape) перед PR2b.

**Feature flag.** `OI_TRACKER_HYPERLIQUID_WS_ENABLED` (default `false`). Включается на проде через env var в systemd-юните + рестарт сервиса. Off-by-default — нулевая риск регрессии при merge.

**Что НЕ меняется в PR2a.**

- `C2` (60 с polling) сохраняется на уровне storage.
- `C12` (snapshot источник для Hyperliquid) формально сохраняется — `oi_samples` пишется ровно как раньше, REST → parser → normalize.
- `oi_5m`/`oi_15m`/`oi_30m` CAs работают как раньше.
- Alert engine для Hyperliquid идёт через cold path (tumbling CA, как у остальных 11 бирж).

**Метрики (новые в `12_OBSERVABILITY_SLO.md`).** `hyperliquid_ws_{connected, subscriptions_active, messages_total{type}, reconnects_total{reason}, last_message_age_seconds}` + `hyperliquid_ring_{samples,symbols}_total`.

**Affected files.**

- docs: `11_EXCHANGE_ADAPTERS/hyperliquid.md §1, §3, §4, §10` (новые секции для WS path), `12_OBSERVABILITY_SLO.md §3` (новые метрики).
- code (new): `backend/app/exchanges/_ring_buffer.py`, `backend/app/exchanges/hyperliquid_ws.py`.
- code (edit): `backend/app/exchanges/hyperliquid.py` (+RingBuffer instance), `backend/app/scheduler/supervisor.py` (+WS lifecycle), `backend/app/observability/metrics.py` (+7 метрик), `backend/pyproject.toml` (sortedcontainers production dep).
- tests (new): `test_ring_buffer.py` (21), `test_hyperliquid_ws.py` (16), fixture `ws_active_asset_ctx.json`.

**Migration / data.** Никакой DB-миграции. Ring buffer — in-memory; пустеет при рестарте сервиса.

**Связано с.** `F47` (PR2b — интеграция WS как primary). `C2` (polling cadence в storage сохраняется). `C12` (источник истины для окон).

---

### F47 · Hyperliquid WS как primary data source + REST fallback (hot lane PR2b)

**Контекст** (2026-05-11). После недели observer-only-наблюдений PR2a метрики покажут реальный профиль WS-стрима (message rate, reconnect частоту, payload sanity). PR2b переводит Hyperliquid на WS как primary, REST — как fallback при stall.

**Решение PR2b.**

- `HyperliquidConnector.fetch_snapshot`: если `OI_TRACKER_HYPERLIQUID_WS_PRIMARY=true` и ring имеет fresh-сэмпл (<10s old) для инструмента — строим `RawExchangeEvent` из ring; для инструментов без fresh-сэмпла — fallback на существующий REST путь `metaAndAssetCtxs`. Decision per-instrument, mixing allowed (одна большая REST-команда покрывает всё, ring дополняет более свежим там, где есть).
- **Multiplier reverse trick.** Ring хранит canonical `oi_coins` (post-multiplier, для совместимости с `oi_samples` и hot-lane evaluator'ом PR3). Parser ожидает native units в payload `{name, openInterest}` (применяет multiplier downstream). При построении raw_event из ring делим обратно: `native_oi = sample.oi_coins / instrument.multiplier`. Точная Decimal-арифметика для multiplier ∈ {1, 1000} даёт нулевую потерю precision.
- **Cold start backfill.** При старте WS-lifecycle, до connect, SELECT из `oi_samples` за последние 32 мин (где `exchange='hyperliquid'`), `bulk_insert` в ring per symbol. Это нужно для PR3 (hot-lane evaluator должен иметь anchor 5/15/30 мин назад с первого тика), но и в PR2b даёт ring «прогретое» состояние при рестартe сервиса.

**Feature flag.** `OI_TRACKER_HYPERLIQUID_WS_PRIMARY` (default `false`, отдельно от `_WS_ENABLED`). Двухключевая схема:
- `_WS_ENABLED=true, _WS_PRIMARY=false` — observer-only (PR2a). Безопасно, диагностически.
- `_WS_ENABLED=true, _WS_PRIMARY=true` — WS primary, REST fallback (PR2b цель).
- `_WS_ENABLED=false` — ring пуст, всё через REST (rollback path).

**Что меняется в data path.**

- `oi_samples` для Hyperliquid: `ts_exchange` всё ещё = `fetched_at` (сервер clock), но теперь это «момент прихода WS-фрейма» (~1-2 с от реального тика биржи), а не «момент REST-вызова» (до 60 с после). Реальная задержка падает с avg ~30 с до ~1-2 с — это и есть цель PR2b.
- `C2` (60 с cadence на запись в storage) **сохраняется**: scheduler по-прежнему вызывает `fetch_snapshot` раз в минуту. Между вызовами WS наполняет ring, но в `oi_samples` пишется одна точка раз в минуту (last-known из ring на момент cycle tick).
- `C12` (snapshot semantics для Hyperliquid) **сохраняется** — `primary_source_kind = SNAPSHOT`, минутная решётка. WS = transport optimization, не смена модели.

**Edge cases.**

- WS stall > 10 с для конкретного инструмента → этот инструмент идёт через REST в текущем цикле; остальные продолжают через ring.
- WS stall > 30 с глобально → все идут через REST (per-instrument decision естественно собьётся, ring пуст для всех).
- Cold start (рестарт сервиса): backfill из `oi_samples` даёт ring с историей; первые WS-фреймы (~1-2 с после connect) обновят latest.
- Late-arriving WS-фрейм (после reconnect): `RingBuffer.insert` использует `SortedList`, кладёт в правильную позицию по `ts`.
- Multiplier=0 для нового k-prefix токена (теоретически невозможно): защита `if instrument.multiplier > 0 else sample.oi_coins`.

**Affected files (PR2b).**

- docs: `00_DECISIONS_LOG.md` (эта запись), `11_EXCHANGE_ADAPTERS/hyperliquid.md` (отметить WS как primary path).
- code (new): `backend/app/exchanges/hyperliquid_backfill.py` (cold-start backfill из `oi_samples`).
- code (edit): `backend/app/exchanges/hyperliquid.py::fetch_snapshot` (per-instrument ring-vs-REST decision), `backend/app/scheduler/supervisor.py` (вызов backfill перед `client.run()`).
- tests: integration-тест mixed ring+REST path.

**Migration / data.** Никакой DB-миграции. `oi_samples` структура без изменений.

**SLO impact.** Для Hyperliquid: `connector_stale_seconds{exchange="hyperliquid"}` падает с типичных 30-60 с до 1-2 с. Это **prerequisite для PR3** (hot-lane evaluator со sliding-window Δ), где latency E2E «тик → TG» становится ~3-5 с p95 для Hyperliquid алертов.

**Связано с.** `F46` (PR2a observer-only — фундамент). `C2` сохраняется на уровне storage. `C12` сохраняется на уровне semantics. PR3 = hot-lane evaluator (отдельная запись `Fxx` будет).

---

### F48 · Hyperliquid hot-lane evaluator со sliding-window Δ (PR3)

**Контекст** (2026-05-11). Завершение трека 1. После F46 (WS observer) + F47 (WS как primary) есть свежие данные в ring buffer'е, но alert engine всё ещё работает на tumbling-CA bucket'ах через cold path для **всех** бирж включая Hyperliquid. Это значит:

1. **30-секундный alert cycle лаг** добавляется к ~1-2с WS latency.
2. **Tumbling bucket прячет скачки** в середине 5-минутного окна (см. F46 контекст про инцидент S 10.05).

PR3 закрывает оба недостатка для Hyperliquid через **hot-lane evaluator** — отдельный таск, тикающий каждые 2с, считающий sliding-window Δ напрямую из ring.

**Архитектурный момент** (обнаружено в процессе). Scheduler и alert_engine — два **разных systemd-юнита** (`oi-tracker-scheduler.service` и `oi-tracker-alert-engine.service`), каждый — отдельный Python-процесс. In-memory ring buffer scheduler'а невидим для alert_engine. Hot lane evaluator поэтому **живёт в scheduler-процессе**, а не в alert_engine.

**Решение PR3.**

- Новый модуль `app/alert_engine/hot_lane.py`, который запускается в supervisor scheduler-процесса:
  - Тик каждые 2с (default `_HOT_LANE_CYCLE_SEC`).
  - Загрузка enabled rules через `rules_repo.get_enabled(pool)`, фильтрация на `RuleType.THRESHOLD | CONFIRMED` (DIVERGENCE и EXCHANGE_HEALTH остаются на cold path — у них другая семантика, не sliding-window).
  - Для каждого правила: walk через `ring.symbols()`, для каждого symbol'а — `latest = ring.latest()`, `anchor = ring.anchor_at(latest.ts - window, tolerance=15s)`. Δ% = `(latest.oi - anchor.oi) / anchor.oi * 100`.
  - Quality gates: latest age ≤ 10s (WS свеж), anchor найден в tolerance (есть история). State machine применяет остальные gates (min_oi_notional_usdt, threshold crossing, cooldown).
  - Synthetic `OIBar` для совместимости с существующим `AlertCandidate` schema'ой (`F48 A1`): degenerate 1-point bucket, open=high=low=close. State machine не знает об отличиях — переиспользуется `transition()` без модификаций.
- **Cold-path exclusion** в `evaluation_cycle._evaluate_rule`: при `OI_TRACKER_HYPERLIQUID_HOT_LANE=true` candidates с `exchange="hyperliquid"` отфильтровываются ДО state machine writes. Защита от double-fires (оба процесса пишут в один `alert_state`).
- Cooldown coordination через PG row-level locks: hot lane (scheduler) и cold path (alert_engine) пишут в `alert_state` через `bulk_upsert`. PG обеспечивает консистентность; smart re-fire / cooldown работают через тот же state.
- Метрики: `hot_lane_cycle_duration_seconds` (histogram), `hot_lane_candidates_total{rule_id, result}` (built/skip_ws_stale/skip_no_anchor), `hot_lane_fires_total{rule_id}`, `hot_lane_e2e_latency_seconds` (histogram от `latest.ts` до `event.ts_processed`).

**Feature flag.** `OI_TRACKER_HYPERLIQUID_HOT_LANE` (default `false`). **Требует** `OI_TRACKER_HYPERLIQUID_WS_ENABLED=true` иначе ring пуст и hot lane бесполезна — supervisor проверяет это на старте и логирует warning + disable hot lane при misconfig.

**Trinity flags для трека 1:**

| `_WS_ENABLED` | `_WS_PRIMARY` | `_HOT_LANE` | Поведение |
|---|---|---|---|
| false | * | * | Полностью legacy: REST для oi_samples, cold-path алерты с tumbling-CA |
| true | false | false | F46 observer-only: WS наблюдает, метрики; oi_samples ещё через REST; cold-path алерты |
| true | true | false | F47: WS primary для oi_samples (~1-2с latency); cold-path алерты с tumbling-CA |
| true | true | true | F48 цель: WS primary + hot-lane алерты со sliding-window Δ |
| false | * | true | Misconfig — supervisor пишет warning, hot lane disabled |
| true | false | true | Технически работает, но oi_samples всё ещё через REST (60s). Hot lane даёт sliding Δ из ring, который наполняется WS. Smysl есть. |

**Edge cases.**

- WS stall во время hot-lane cycle: latest ring sample становится старше 10s → quality gate `skip_ws_stale` отсекает symbol. Алерты по нему не уйдут пока WS не восстановится. Cold-path тоже не подхватит (filter активен). **Это сознательный выбор**: при WS stall лучше пропустить тик чем алертить на старых данных. WS-monitoring пометит проблему через `hyperliquid_ws_last_message_age_seconds`.
- Hot lane crash: supervisor TaskGroup ловит exception в `_hyperliquid_hot_lane_lifecycle`, логирует, поднимает заново при следующем рестарте. Alert engine cold path продолжает работать, но НЕ обрабатывает Hyperliquid (flag всё ещё on в env). Workaround: ручной flag off через systemd + restart.
- Cooldown гонка: hot lane (scheduler) и cold path (alert_engine) теоретически могут попытаться обновить тот же `alert_state` rows одновременно. PG row-level locks на `INSERT ON CONFLICT DO UPDATE` обеспечивают линеаризуемость. Cold path фильтрует Hyperliquid → реально только hot lane пишет в эти rows когда флаг on.

**SLO impact.** Для Hyperliquid: E2E latency «тик биржи → TG алерт» падает с avg ~47с (P95 ~90с) до:
- ~1-2с WS receive
- ~1-2с hot-lane cycle (avg 1с с jitter)
- <100ms state machine + DB writes
- ~1-2с TG sender poll
- **Total: ~3-5с p95**

Также **исчезает кейс «скачок съеден одним tumbling bucket'ом»** — sliding-window Δ детектирует скачок в любом месте окна, в момент его появления.

**Affected files.**

- code (new): `backend/app/alert_engine/hot_lane.py` (~350 строк).
- code (edit): `backend/app/alert_engine/evaluation_cycle.py` (cold-path exclusion ~10 строк), `backend/app/scheduler/supervisor.py` (lifecycle task + flag check ~30 строк), `backend/app/observability/metrics.py` (+4 метрики).
- tests (new): `tests/unit/alert_engine/test_hot_lane.py` (15 тестов).
- tests (edit): `tests/unit/alert_engine/test_evaluation_cycle.py` (+3 теста на exclusion).
- docs: эта запись.

**Migration / data.** Никакой DB-миграции. `alert_state`, `alert_events`, `delivery_queue` структуры без изменений — hot lane пишет ровно то же, что cold path.

**Связано с.** `F46`, `F47` (WS prerequisites). Также overrides `C2` для Hyperliquid в части **alert cadence** (60с → 2с) при сохранении storage cadence 60с. `C12` сохраняется на уровне `oi_samples`.

---

### F49 · TG `oi_threshold_cross` — строка «Δ цены» (% + абсолют)

**Контекст** (2026-05-11). Пользователь в TG-сообщении хочет видеть не только %-дельту OI и текущий уровень цены, но и **движение цены за окно** — насколько и в каком направлении ушла цена параллельно скачку OI. Без этого «+6.2% OI» не отвечает на естественный вопрос «а что цена сделала?»: она в payload присутствует только в виде уровня (`Цена: $67,420`), не в виде дельты. Поле `price_change_pct` ранее уже клалось в payload, но было помечено как `informational, not in default render` и не рисовалось.

**Решение.**

- В `_format_payload_threshold` (`backend/app/delivery/producer.py`) добавлен **опциональный** ключ `price_delta_abs` = `str(price_now − price_prev)` в USDT, со знаком. Кладётся **только** когда `AlertCandidate.price_prev is not None` (есть история ≥ окна). Прочие поля payload не меняются; новых столбцов в БД / новых полей в `AlertCandidate` нет.
- В `render_oi_threshold_cross` (`backend/app/tg_sender/templates.py`) добавлена четвёртая строка:
  ```
  Δ цены: {±price_change_pct%} ({±$price_delta_abs}) за {window}
  ```
  Строка рисуется **только при одновременном** наличии `price_change_pct` и `price_delta_abs` в payload. Если хотя бы одного нет — строка пропускается, и сообщение выглядит как до F49.
- Новый helper `humanize_signed_usd` в `templates.py` — `humanize_usd` со знаком и `$` (e.g. `+$820`, `-$1.24M`, `+$0.50`). Формат магнитуды совпадает с `humanize_usd` (K/M/B + 2 знака после запятой, для целых $ — без дробей).

**Почему «опционально, а не подменять на 0».** В producer'е для `price_change_pct` уже стоит подмена `None → "0"`. Это смешивает «истории нет» и «цена реально не двинулась». Для абсолюта не повторяем эту ошибку: при `price_prev=None` ключа просто нет, и renderer пропускает строку. Это даёт пользователю однозначное прочтение: если строка есть — данные достоверные; если нет — у бара короткая история. Подмена для `price_change_pct` сохранена как есть (out-of-scope этого решения), но рендер защищён от мисcoupling: пока абсолюта нет — строка не рисуется, и фальшивый «+0.00% ($0)» в TG не уйдёт.

**Output (пример).**
```
BTC @ Binance
+6.2% за 5мин
OI: $1.24B   Цена: $67,420
Δ цены: +0.40% (+$270) за 5мин
```

**Affected files.**

- code (edit): `backend/app/delivery/producer.py` (+5 строк в `_format_payload_threshold`).
- code (edit): `backend/app/tg_sender/templates.py` (новый `humanize_signed_usd` ~25 строк, расширение `render_oi_threshold_cross` ~10 строк).
- tests (edit): `backend/tests/unit/tg_sender/test_templates.py` (новый блок `TestHumanizeSignedUsd`, +3 теста для `render_oi_threshold_cross`).
- tests (edit): `backend/tests/unit/delivery/test_producer.py` (+3 теста в `TestFormatPayloadThreshold`).
- docs (edit): `docs/10_DELIVERY_LAYER.md §3.1` (payload + render + output обновлены).
- docs (edit): эта запись + строка в summary-table.

**Migration / data.** Нет. `delivery_queue.payload` jsonb расширяется новым optional ключом — обратная совместимость сохранена: старые рендеры без `price_delta_abs` рисуются как было, новые — с дополнительной строкой.

**Связано с.** `10 §3.1`. Не отменяет ни одно из C1–C17, Q1–Q8, F-записей. Чистое расширение TG copy.

---

### F50 · TG `oi_threshold_cross` — abs(OI) inline во 2-й строке (replaces F49)

**Контекст** (2026-05-11). После F49 4-я строка `Δ цены: ±X.XX% (±$Y) за W` показывала **изменение цены** за окно. Пользователь поправил продуктовое требование: интереснее видеть **абсолют изменения OI** (не цены) и не отдельной строкой, а inline во 2-й строке рядом с уже отображаемым % дельты OI. Это сжимает сообщение с 4 строк до 3 и убирает визуальный дубль (% OI оставался на 2-й строке, а строка ниже была про цену).

**Решение.**

- В `_format_payload_threshold` (`backend/app/delivery/producer.py`) **убран** ключ `price_delta_abs`; добавлен `oi_delta_abs_usdt` = `str(current_bar.close_oi_notional_usdt - prev_bar.close_oi_notional_usdt)` (USDT, со знаком). Кладётся только при `cand.prev_bar is not None`. Дельта берётся от тех же двух closes баров, что считают `delta_pct` — знак и направление гарантированно консистентны с %.
- В `render_oi_threshold_cross` (`backend/app/tg_sender/templates.py`):
  - 2-я строка теперь `{±X.XX%} ({±$Y}) за {window}` при наличии `oi_delta_abs_usdt`, иначе fallback на голую `{±X.XX%} за {window}`.
  - 4-я строка «Δ цены: …» **полностью удалена**.
  - `humanize_signed_usd` остаётся (теперь используется для OI абсолюта вместо цены).
- `price_change_pct` в payload оставлен как информационное поле (как было до F49 — рендером не используется); не ломаем backward-compat consumers, которые на него полагались.

**Почему replaces, не extends.** F49 был тактическим решением (отдельная строка про цену), F50 пересматривает продуктовое требование на основе обратной связи. F49 деплоился в проде ~1 час перед заменой; никаких миграций или persisted данных не требует — payload в `delivery_queue` jsonb просто получает другой набор optional ключей с момента F50 deploy.

**Output (пример).**
```
BTC @ Binance
+6.20% (+$76.00M) за 5мин
OI: $1.24B   Цена: $67,420
```

**Output (cold-start).**
```
BTC @ Binance
+6.20% за 5мин
OI: $1.24B   Цена: $67,420
```

**Affected files.**

- code (edit): `backend/app/delivery/producer.py` (5 строк: `price_delta_abs` → `oi_delta_abs_usdt`, источник — `current_bar`/`prev_bar`).
- code (edit): `backend/app/tg_sender/templates.py` (~15 строк: 4-я строка удалена, 2-я строка стала conditional).
- tests (edit): `backend/tests/unit/tg_sender/test_templates.py` (3 теста переделаны под inline-формат).
- tests (edit): `backend/tests/unit/delivery/test_producer.py` (3 теста: `oi_delta_abs_usdt` present/negative/absent).
- docs (edit): `docs/10_DELIVERY_LAYER.md §3.1` (payload + render + output обновлены).
- docs (edit): эта запись + строка в summary-table.

**Migration / data.** Нет. `delivery_queue.payload` jsonb меняет набор optional ключей (key turnover): `price_delta_abs` больше не пишется новыми deliveries; старые pending rows с этим ключом отрисуются по новому шаблону без 4-й строки (renderer его просто игнорирует). Pending'ов на момент deploy нет.

**Связано с.** `F49` (replaced), `10 §3.1`.

---

### F51 · Aster 5m alerts switch from OI to trade volume

**Контекст** (2026-05-16). Aster — единственная биржа из 12, которая публикует OI с очень медленным cadence: измеренный лаг `ts_ingested − ts_exchange` имеет P50=26 мин, P95=2.5 ч (vs ≤8с у всех остальных). Aster обновляет `openInterest` нерегулярно раз в 5–25 мин — наш минутный poll получает один и тот же `time`-stamp по нескольку раз, и в `oi_5m` CA соседние «buckets» по факту разнесены на 15–25 wall-clock минут. Значит правило «−8% за 5мин» на Aster реально означает «−8% между двумя точками OI с разрывом 15+ минут» — продуктово нерепрезентативно.

Альтернативы:
- (а) исключить Aster из 5m-правил → теряем сигналы там, где они есть;
- (б) глобальный bucketing по `ts_ingested` → ломает корректность остальных бирж;
- (в) **сменить метрику для Aster@5m** на торговый объём, который Aster, как и Binance, отдаёт через `/fapi/v1/klines` с минутным cadence и точной 5m-аггрегацией на стороне биржи.

Выбран вариант (в): для Aster в окне 5m альертим по **Δ% torgового объёма** (`quoteAssetVolume` 5m kline) с фильтром по абсолютному объёму. Окна 15m/30m на Aster продолжают использовать OI — на этой шкале медленный cadence Aster укладывается.

**Решение (PR1 — pipeline, scope этого решения):**

- Новая hypertable `volume_samples_5m` (миграция `0016_volume_samples_5m.py`):
  - Partition axis `bucket TIMESTAMPTZ` = kline `openTime`, 5m-aligned.
  - PK `(exchange, canonical_symbol, bucket)`, UPSERT при повторных опросах running bucket'а.
  - Numeric precision как у `oi_samples`: `base_volume NUMERIC(38,18)`, `quote_volume_usdt NUMERIC(38,8)`.
  - Compression policy: 7d compress_after, segmentby `(exchange, canonical_symbol)`, orderby `bucket DESC`.
  - Retention: 90d (симметрично `oi_samples`, C3).
  - Index `(exchange, bucket DESC)` для быстрого lookback.
- Domain: новый `VolumeSample5m` (Pydantic, frozen+strict, UTC-валидация, `≥0` гарды на volumes).
- Parser: `backend/app/normalizer/parsers/aster_klines.py` — pure pin поля `[openTime, ..., volume, ..., quoteAssetVolume, numberOfTrades]` Binance-compatible kline'а.
- Connector: `AsterConnector.fetch_5m_volumes(instruments)` → `/fapi/v1/klines?symbol=X&interval=5m&limit=2` per symbol, concurrency-limited, soft-skip per-symbol HTTP error. Эмиттим **обе** клины (closed + running) — repo UPSERT'ит running bucket каждый цикл до его закрытия.
- Repo: `app.storage.repositories.volume_samples`:
  - `bulk_upsert(pool, samples)` — асинхронный executemany с `ON CONFLICT DO UPDATE`.
  - `get_latest_two_closed_buckets(pool, exchange, before)` — для PR2 (volume signal). «Closed» = `bucket + 5min ≤ before`, чтобы не сравнивать с растущим бакетом.
- Scheduler: `_aster_volume_lifecycle` task в `supervisor.run_supervisor` TaskGroup. 60s wall-clock cadence, изоляция per-exchange — упавший volume tick не валит OI pipeline или другие сервисы. Создаётся ТОЛЬКО если в конфигурации присутствует `AsterConnector` (без флагов — встроено).
- Metrics: переиспользуем `connector_request_*` (label endpoint=`/fapi/v1/klines`), `connector_cycle_duration_seconds{exchange="aster"}`, `storage_insert_*{table="volume_samples_5m"}`.

**Решение (PR2 — signal + delivery, deployed 2026-05-16):**

- `RuleType.VOLUME_THRESHOLD = "volume_threshold"` — новое значение enum'а.
- `VolumeThresholdSignal` (`app/alert_engine/signals/volume_threshold.py`):
  - Читает `volume_samples_5m` через `get_latest_two_closed_buckets`.
  - Floor: `settings_kv.min_volume_usdt_default` (JSON-string Decimal, default 50000). Floor применяется ДО Δ% — отсекает микропары.
  - Δ% = `(current.quote_volume − prev.quote_volume) / prev.quote_volume × 100`.
  - **Synthetic OIBar trick:** конструируем `OIBar` из `VolumeSample5m` (close_oi_notional_usdt = quote_volume_usdt). Это позволяет переиспользовать state machine без модификации `AlertCandidate`. Producer routing по `rule.rule_type` маскирует семантический overload на wire.
  - `sample_age_seconds = now − current.ts_ingested` (НЕ `now − bucket`), потому что closed bucket по определению ≥ 5 мин старше now, и OI-style `LATE_DATA_SKIP` budget 120s всё бы убил. Свежесть здесь — это «когда мы прочитали kline», не «время bucket'а».
- Guard в `ThresholdSignal.evaluate` (`app/alert_engine/signals/threshold.py:54-62`): пропускает пары `(exchange='aster', window='5m')`. In-engine check, без модификации `exchange_filter` существующих 5m OI-правил (id=1, id=2 продолжают работать на остальных 11 биржах).
- Seed (миграция `0017_aster_volume_defaults.py`): 2 default-rule (`Aster volume up 5m` ≥5%, `Aster volume down 5m` ≤-5%, `exchange_filter=['aster']`, `min_oi_notional_usdt=0`) + KV `min_volume_usdt_default="50000"`.
- Producer (`app/delivery/producer.py`): `_format_payload_volume(rule, cand)` — формирует payload с ключами `quote_volume_usdt`, `volume_delta_abs_usdt` (optional, omitted when `prev_bar is None`). Маршрутизация в `enqueue` через `rule.rule_type is RuleType.VOLUME_THRESHOLD`.
- TG template (`app/tg_sender/templates.py`): `render_volume_threshold_cross` — 3-строчный формат, 2-я строка `{±X.XX%} ({±$Y}) за W · vol`, 3-я строка `Vol W: $Y   Цена: $Z`.

**Verified end-to-end (2026-05-16 после deploy):** Aster BTC -53% и ETH -30% volume Δ за 5m → `delivery_queue` row → TG `status='sent'` за ~1 секунду.

**Known limitation (deferred to PR3):** `price` в volume payload = `"0"`, потому что volume cycle не запрашивает mark price. PR3 (UI/enrichment) добавит чтение последней `oi_samples.price_used` для символа на момент рендера.

**PR3 (вне scope этого F-record):** UI — Settings/Live/Symbol page переключатели для Aster@5m, Alerts log renderer.

**Rate-limit.** Aster ~190 USDT-perp символов × 1 req/мин × weight 1 = ~190 weight/мин против бюджета 1200 weight/мин (`aster.py:58`). Запас ×6, в пределах SLO без рисков.

**Output (после PR2/PR3 пример).**
```
LINK @ Aster
+12.50% (+$1.20M) за 5мин · vol
Vol 5m: $2.10M   Цена: $9.6530
```

**Affected files (PR1).**

- code (new): `backend/alembic/versions/0016_volume_samples_5m.py` (migration).
- code (new): `backend/app/normalizer/parsers/aster_klines.py` (parser).
- code (new): `backend/app/storage/repositories/volume_samples.py` (repo).
- code (edit): `backend/app/domain/events.py` (`VolumeSample5m`).
- code (edit): `backend/app/exchanges/aster.py` (`fetch_5m_volumes`).
- code (edit): `backend/app/scheduler/supervisor.py` (`_aster_volume_lifecycle` + wiring).
- tests (new): `backend/tests/unit/exchanges/test_aster_klines_parser.py`, `test_aster_klines_connector.py`.
- docs (edit): `docs/05_DATA_CONTRACTS.md`, `docs/08_TIME_SERIES_STORAGE.md`, `docs/11_EXCHANGE_ADAPTERS/aster.md`, `docs/12_OBSERVABILITY_SLO.md`, эта запись + строка в summary-table.

**Связано с.** План в `tasks/todo.md` от 2026-05-16 («Aster 5m: volume-based сигнал вместо OI»).

---

## 10. Summary table

| ID | Тема | Краткое решение |
|---|---|---|
| C1 | canonical_symbol | `BASE` (`"BTC"`), market-info в раздельных колонках |
| C2 | polling | flat 60s aligned to minute |
| C3 | retention | 90 дней для всего |
| C4 | Δ-порог | только % |
| C5 | Δ-база | `oi_coins` |
| C6 | state machine | 4-state, confirm_points=1 default |
| C7 | freshness | формулы от poll_interval |
| C8 | user model | single user, group chat, no user_id |
| C9 | DB schema | wide TEXT + TS compression |
| C10 | settings | гибрид: KV для глобалок, table для rules, env для secrets |
| C11 | hypertable axis | `ts_exchange` (TIMESTAMPTZ) |
| C12 | window source | native primary для Bybit/KuCoin, snapshot для остальных |
| C13 | mermaid | перерисован без Compute ноды |
| C14 | UI updates | SSE-push, no 5s polling |
| C15 | §2 corrections | OKX/Hyperliquid/Aster без копипаста |
| C16 | repo name | `oi-tracker` |
| C17 | exchange count | 12 в DDL-комментариях |
| Q1 | cooldown | smart, ≥1.5× threshold |
| Q2 | signal types | 4 типа (threshold, confirmed, divergence, exchange-health). _Изначально 5; consensus sunset per F40._ |
| Q3 | latency SLO | 90s P95 / 180s P99 |
| Q4 | scale | 1 user, no public API |
| Q5 | observability | Prometheus + Grafana + Loki |
| Q6 | deployment | bare-metal, systemd, no Docker |
| Q7 | backups | future work |
| Q8 | TG creds | token in env, chat_id in DB |
| F2 | instruments sync | каждые 5 минут |
| F3 | error isolation | per-exchange task, circuit breaker |
| F4 | numeric precision | (38,18) для coins/price, (38,8) для USDT |
| F5 | compression | chunk 1d, compress after 7d |
| F6 | alert rules | structured table, 6 default rows |
| F7 | smart cooldown | re-fire if new Δ ≥ 1.5× previous |
| F8 | min history | `max(30, 2 × window)` |
| F9 | reprocessing | no retro-fire |
| F10 | valuation status | authoritative / good_estimate / low_confidence |
| F11 | source kind | snapshot / native_interval / backfill / replay / degraded_fallback |
| F12 | домен | `oi-tracker.robot-detector.ru` (subdomain) |
| F13 | noindex | three-layer: `X-Robots-Tag` + robots.txt + meta |
| F14 | изоляция + hardening | UDS, dedicated PG db+role, OS user, paths, no `verify=False`, no f-string SQL |
| F15 | UI settings scope | категории A/B/C, rate-limit 5/min, audit log |
| F16 | TLS | отдельный certbot per subdomain (cert robot-detector — single-domain) |
| F17 | TimescaleDB install | apt + tune + restart кластера в maintenance window |
| F18 | default thresholds | 6 base rules: 5%/8%/12% × up/down (5m/15m/30m), `confirm_points=1` |
| F19 | Bybit/KuCoin path | shipped snapshot path в v1; native_interval отложен на M3 |
| F20 | Bitunix deferred | public API без OI; 11 бирж в v1 (не 12 как `C17`). _Superseded by F42._ |
| F21 | notional fix | для `base_asset+multiplier>1` mark price делится на multiplier; `NORMALIZATION_VERSION` 1→2 |
| F22 | aiogram throttle | leaky-bucket 25 msg/sec global + 1 msg/sec per chat; код в M3.B |
| F23 | settings hot-reload | НЕ реализуем в v1 — restart-required (5s downtime приемлем для single-user) |
| F24 | alert_rules.extra | JSONB колонка для signal-type-specific параметров (divergence_price_threshold, health sub_trigger). Schema flexible, runtime validation в сигналах. _Раньше включал `consensus_count` — удалён per F40._ |
| F25 | ExchangeHealth v1 scope | ship'аем только sub-trigger `stale_data`; остальные 3 (low_coverage / high_error_rate / instruments_sync_failing) deferred до появления `connector_health_history` table. Bypass state_machine, dedupe via cooldown bucket. |
| F26 | Consensus exchange sentinel | _Superseded by F40 (sunset consensus). Sentinel остаётся только в исторических `alert_events` строках до retention 90д._ |
| F27 | SSE no event replay on reconnect | Client re-fetches snapshot via `connection_init`; gaps acceptable (TG canonical for alerts; next poll repaints live OI). UI badge mandatory. |
| F28 | PG NOTIFY keys-only payload | Triggers emit only ids (`exchange, canonical, ts_*`); 8KB limit guard; полный SELECT на API стороне. Расширение payload — отдельным F. |
| F29 | trace_id propagation | structlog.contextvars + persisted `payload._trace_id`; prefix-based naming (`cycle/replay/api/tg-{uuid8}`); no OTEL in v1. Spec: `12 §5.4`. |
| F30 | Prometheus scrape | TCP loopback `127.0.0.1:9100/9101/9102/9103` для api/scheduler/alert-engine/tg-sender. UDS не поддерживается Prometheus; F14 не нарушается (loopback ≠ public). |
| F31 | M5 coverage closure (Path C) | M5 закрывается при ≥98% coverage на decision-heavy modules (alert_engine.evaluation_cycle, delivery.producer, observability.logging_config, tg_sender.dispatcher) при overall unit baseline ~66%. Backfill для `storage/repositories/*` и `api/routes/*` (требуют integration harness) deferred в M6 с явным планом. `pytest --cov-fail-under=65` отражает реальный baseline. |
| F32 | Settings PUT response shape | Возвращаем upsert-row как есть — без envelope `{applied, restart_required: [...]}` из §9.6 placeholder. Single-user pragma: оператор знает, что F23 = restart-required, и сам restartит затронутый сервис; envelope не несёт safety-критичной нагрузки. Audit-log пишется по-прежнему в `settings_audit`. |
| F33 | ConnectorErrorEvent superseded | Pydantic-контракт остаётся как domain type, но НЕ конструируется в production path — заменён на `oi_collector_request_total{exchange, status, error_type}` Prometheus метрику (M1.B.6) + structured logs с `error_type` field. Никакого `connector_errors` hypertable. Дашборд M5.A.6 покрывает forensic анализ по error_type / exchange без БД. |
| F34 | M6 integration harness | `testcontainers[postgres]` с timescaledb:latest-pg16, session-scope container + alembic upgrade один раз; **function-scoped asyncpg pool** (per-test, ~30ms — обходит cross-loop asyncpg ограничения); per-test TRUNCATE+RESTART IDENTITY+CASCADE на BASE TABLEs + reseed (4 instruments + 6 rules). Savepoint pattern не подходит, т.к. repo API берёт connection из pool. |
| F35 | psycopg2-binary in dev-deps | Sync alembic в test fixture требует psycopg2; добавлено в `[project.optional-dependencies] dev`. Production app по-прежнему pure asyncpg (`02 §2.1.1`); psycopg2 не попадает в production extras. |
| F36 | Continuous aggregates skip in test reset | CA (`oi_5m/15m/30m`, `oi_quality_5m`) — read-only matviews, материализуются от source hypertable; TRUNCATE на CA не разрешён TimescaleDB. `_RESET_TABLES` включает только BASE TABLEs; тесты, которым нужны CA-данные, явно вызывают `refresh_continuous_aggregate(...)`. |
| F37 | Multi-chat routing | Per-rule `alert_rules.chat_id` FK на `tg_chats` с partial-unique-default; `delivery_queue.chat_id_native` снапшот на FIRE-time; sync getChat-валидация на create + hourly background revalidation; `settings.tg_chat_id` deprecated alias (удалён в F38). Override `10 §2.6`. _Extended by F41 (per-exchange routing groups) — rule.chat_id остаётся explicit override._ |
| F38 | settings.tg_chat_id alias removed | Sunset accelerated: single-user продукт без публичных потребителей, pre-flight grep чист, ~70 строк dead code удалены. Recovery default chat — через `POST /api/v1/chats` (runbook `13 §5.9`). |
| F39 | API route prefixes hotfix | `alert_rules.py` (`/alert-rules` → `/api/v1/alert-rules`), `instruments.py` (`/instruments` → `/api/v1/instruments`). Без `/api/v1` nginx не проксировал на UDS, ответ — SPA HTML, `useAlertRules` крашил Settings-страницу. Frontend `useAlertRules.ts` обновлён симметрично. |
| F40 | Sunset consensus signal | `RuleType.CONSENSUS`, `signals/consensus.py`, `consensus_alert` template, `consensus_min_exchanges` setting, slider, `oi_bars.get_bars_grouped_by_symbol` helper — удалены. Миграция `0013_drop_consensus.py` удаляет строки `alert_state`/`alert_rules` с `rule_type='consensus'` + `settings_kv` ключ. Один алерт = одна биржа = одно сообщение. Supersedes Q2 (5→4 типа), F26 (sentinel exchange), F24 (часть про `consensus_count`). |
| F41 | Per-exchange routing groups | Новые таблицы `tg_routing_groups` + `tg_routing_group_exchanges (UNIQUE exchange)` + колонка `delivery_queue.message_thread_id`. Resolver order: rule.chat_id (override) → group по бирже → default chat. Dispatcher шлёт с `message_thread_id`. Throttle keyed по chat_id. Async-валидация thread_id через `last_send_error` badge. Extends F37 (per-rule routing остаётся explicit override). |
| F42 | Bitmart replaces Bitunix | 12-я биржа = Bitmart (public bulk `/contract/public/details` отдаёт OI + valuation + price); Bitunix permanently removed. Supersedes F20. C17 «12 бирж» восстановлен. `oi_unit_hint=contracts`, `oi_coins=open_interest×contract_size×multiplier(=1)`, `open_interest_value` provided → `valuation_status=authoritative`. |
| F43 | alert_events persistance whitelist | В `alert_events` пишутся только `{FIRE, SUPPRESS, RESOLVE, QUALITY_GATE_FAIL}`. Остальные 5 decisions — counter-only через `oi_alert_engine_decisions_total`. Сокращает запись в audit-таблицу с 60 M/день до ~2 K/день (~99.997% drop). State machine, `alert_state`, `delivery_queue`, API filter literal — без изменений. Overrides `09 §10.1`. |
| F44 | alert_events columnstore | Native compression на `alert_events` (`segmentby='rule_id, exchange'`, `orderby='ts_processed DESC'`, `compress_after='7 days'`) — симметрично F5/`oi_samples`. Базис для шага 3: cleanup существующих 168M noise rows через `DELETE → compress_chunk()`, возврат диска без `VACUUM FULL`. Read-paths прозрачны. |
| F45 | alert_events historical cleanup | One-shot: preserve audit (~6.4K rows) → drop closed chunks (May 4/5, instant −53 GB) → restore audit → DELETE noise на hot chunk → VACUUM. DB 81 GB → 30 GB. Hot chunk финализируется через policy compress на 2026-05-14. Audit 100% сохранён. |
| F46 | Hyperliquid WS observer-only (hot lane PR2a) | WS-таск на `wss://api.hyperliquid.xyz/ws` параллельно REST. Подписка `activeAssetCtx`, in-memory ring buffer (32 мин retention, canonical `oi_coins`). Resilience: exp backoff, heartbeat, stall watchdog, debounce 1s. `fetch_snapshot`/`fetch_prices` НЕ меняются. Feature flag `OI_TRACKER_HYPERLIQUID_WS_ENABLED` (default false). Цель: собрать боевые метрики до PR2b. |
| F47 | Hyperliquid WS primary + REST fallback (hot lane PR2b) | `fetch_snapshot` per-instrument: fresh ring (<10s) → ring; иначе → REST (mixing разрешён). Multiplier reverse trick для совместимости с parser. Cold-start backfill из `oi_samples` за 32 мин при старте WS-lifecycle. Feature flag `OI_TRACKER_HYPERLIQUID_WS_PRIMARY` (default false). `C2`/`C12` сохраняются на уровне storage (минутная решётка). `connector_stale_seconds` падает 30-60s → 1-2s. Prerequisite для PR3 hot-lane evaluator. |
| F48 | Hyperliquid hot-lane evaluator (PR3) | Sliding-window Δ напрямую из ring каждые 2с, в scheduler-процессе (alert_engine — отдельный процесс, ring невидим). Synthetic OIBar для совместимости с `AlertCandidate`. Cold-path filter в `evaluation_cycle` пропускает Hyperliquid при флаге → защита от double-fires. Cooldown через тот же `alert_state` + PG row-level locks. Feature flag `OI_TRACKER_HYPERLIQUID_HOT_LANE` (default false, требует `_WS_ENABLED=true`). E2E latency Hyperliquid алертов 47с avg → 3-5с p95. Убирает «скачок в bucket» edge case. |
| F49 | TG `oi_threshold_cross` строка «Δ цены» | ~~Producer кладёт optional `price_delta_abs` = `price_now - price_prev` (USDT, signed). Renderer рисует 4-ю строку `Δ цены: ±X.XX% (±$Y) за {window}` при наличии обоих полей.~~ **Replaced by F50** (изменилось продуктовое требование: показывать абсолют OI inline во 2-й строке, не цены отдельной строкой). |
| F50 | TG `oi_threshold_cross` — abs(OI) inline | Replaces F49. Producer кладёт optional `oi_delta_abs_usdt` = `current_bar.close - prev_bar.close` (USDT, signed, от тех же баров что считают `delta_pct`) только при `prev_bar is not None`. Renderer вставляет абсолют inline во 2-ю строку `{±X.XX%} (±$Y) за W` при наличии ключа; иначе fallback на голую `{±X.XX%} за W`. 4-я строка «Δ цены» удалена. `price_delta_abs` больше не пишется. `humanize_signed_usd` сохранён. |
| F51 | Aster 5m: trade volume вместо OI | PR1 (pipeline): новая hypertable `volume_samples_5m` (мигр. `0016`), `VolumeSample5m` domain, parser `aster_klines.py`, `AsterConnector.fetch_5m_volumes` (`/fapi/v1/klines?interval=5m&limit=2` per symbol), repo `volume_samples` (UPSERT + read `get_latest_two_closed_buckets`), supervisor task `_aster_volume_lifecycle` (60s cadence, изолирован от OI). PR2 (signal + delivery): `RuleType.VOLUME_THRESHOLD`, `VolumeThresholdSignal` (synthetic-OIBar трюк), guard в OI-`ThresholdSignal` пропускает `(aster, 5m)`, seed 2 default-rules (`Aster volume up/down 5m`, ±5%), KV `min_volume_usdt_default=50000` (мигр. `0017`), producer `_format_payload_volume` + template `volume_threshold_cross`. PR3 (UI) — следующее. Окна 15m/30m на Aster остаются OI. |
