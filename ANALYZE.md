## Что уже хорошо

По `OI_TRACKER_SPEC` система декомпозирована на понятные подсистемы: collectors, normalizer, time-series storage, alert engine, delivery layer, API/UI и настройки, что является сильной архитектурной базой для дальнейшей детализации.
Также хорошо, что отдельно выделены exchange-specific нюансы, нормализация единиц OI и хранение `oinotionalusdt` как общего канонического значения — это правильное решение для кросс-биржевого сравнения.

Отдельным плюсом является то, что в спецификациях уже присутствуют quality-сигналы: `sampleage`, `missing samples`, `stale symbols`, `coverage ratio`, dead-letter сценарии, cooldown и dedupe. Это означает, что система мыслится не как “парсер API”, а как observability-driven продукт, что для биржевой аналитики критично.

## Ключевые несогласованности

Самая заметная коллизия — разное понимание базового временного поля. В `TIME_SERIES_STORAGE` hypertable строится по `tsingested`, но агрегаты и бизнес-смысл системы завязаны на `tsexchange`; при этом в `OI_TRACKER_SPEC` окна 5/15/30 минут трактуются как market-time, а не ingest-time. Это создаёт риск ложных дельт, если доставка с биржи запаздывает или приходит пачками.
Для OI-аналитики источником истины должен быть `tsexchange`, а `tsingested` — только техническим полем для SLA, лагов и диагностики.

Вторая проблема — разный смысл “истории” по биржам. В `SPEC-5` для одних бирж описан snapshot-only polling, для других — native interval history, а в `OI_TRACKER_SPEC` итоговые окна 5/15/30 выглядят как единый сопоставимый слой. Без явной фиксации, какие окна derived, а какие exchange-native, метрики будут смешивать несопоставимые ряды.
Особенно опасно это для Bybit и KuCoin, где исторические интервалы нативны, против Binance/OKX/Bitget, где многое восстанавливается из snapshot sequence.

Третье расхождение — конфликт между единицей хранения и единицей вычисления. В `NORMALIZER` подчёркнуто, что rolling windows не задача normalizer, а задача metrics/storage layer, но в `OI_TRACKER_SPEC` и частично в `ALERT_ENGINE` местами создаётся впечатление, что normalized sample уже почти готов к вычислению 5/15/30-метрик. Это не критическая ошибка, но контракт между `normalized sample`, `bar`, `metric snapshot` и `alert candidate` явно недоопределён.

## Архитектурные риски

Сейчас в документах не зафиксирован единый event contract между collector → normalizer → storage → alert engine. В разных файлах фигурируют похожие, но не идентичные сущности: `RawExchangeEvent`, `NormalizedOISample`, `oi5m/15m/30m`, `metric snapshot`, `alert event`, `delivery request`; без формального schema registry это приведёт к дрейфу JSON-полей и тихим несовместимостям версий.
Для production-системы нужен один canonical contract document с обязательными полями, nullable-полями, enum-значениями и versioning policy.

Есть риск ложной точности в формуле OI notional. `NORMALIZER` корректно описывает, что у разных бирж OI может быть в base asset, contracts или уже в notional, а цена берётся через mark/index/last fallback. Но нигде не определено, в каких случаях `oinotionalusdt` считается authoritative, а в каких estimated, и как это влияет на downstream алерты.
Без `confidence tier` или `valuation_quality` система будет одинаково доверять Binance historical notional и XT-derived conversion через `contractSize * price`, хотя качество этих оценок разное.

Сильно недоописан слой instrument master data. В нескольких файлах упоминаются `instrument registry`, `contractSize`, `ctVal`, `settleAsset`, `quoteAsset`, `isUsdtM`, canonical symbol mapping, но не определён lifecycle этих данных: когда они синкаются, как версионируются, как инвалидируются при листинге/делистинге, что делать при изменении multiplier. Для деривативных бирж это не второстепенная деталь, а опорная часть корректной нормализации.

## Плохие практики

В документах слишком часто смешаны продуктовые требования, технический дизайн и почти кодовый уровень деталей в одном слое. Например, в `OI_TRACKER_SPEC` рядом находятся бизнес-правила, SQL-таблицы, список файлов проекта, стек FastAPI/React и поведение Telegram-алертов; из-за этого сложно понять, что является обязательным контрактом, а что — лишь одной из реализаций.
Лучше разделить артефакты на: Product Spec, Architecture Spec, Data Contracts, Exchange Adapter Specs, Alert Rule Spec, Operational SLO/SLA.

Есть smell в хранении и вычислении через “append-only + ON CONFLICT DO NOTHING” без чёткой стратегии late-arriving data и replay. Это допустимо для ingestion, но недостаточно для системы, где часть источников history-native, а часть snapshot-derived; при позднем backfill агрегаты и алерты могут оказаться неконсистентными.
Нужно отдельно определить reprocessing model: immutable raw, versioned normalized, recomputable aggregates, alert invalidation policy.

Ещё одна слабая практика — отсутствие формального определения “freshness” и “missingness”. В `ALERT_ENGINE` упоминаются `sampleage`, `requirefreshnesssec`, stale/degraded state, а в `TIME_SERIES_STORAGE` есть `missingsamplescount`; однако нет таблицы с единым правилом, например: для snapshot exchanges свежесть 2 * poll_interval + jitter, для native interval exchanges свежесть = конец окна + grace period.
Пока это выглядит как набор хороших идей, но не как проверяемый operational contract.

## Что улучшить в первую очередь

Нужно зафиксировать единый семантический pipeline:

- Raw exchange event.
- Normalized point-in-time sample.
- Windowed OI bar/metric snapshot for 5m/15m/30m.
- Alert evaluation snapshot.
- Delivery event.

Нужно отделить три времени везде одинаково:

- `tsexchange` — рыночное время наблюдения.
- `tsingested` — когда событие получено системой.
- `tsprocessed` или `tsevaluated` — когда оно прошло normalizer/alert engine. Сейчас этого поля явно не хватает для трассировки end-to-end lag.

Нужно ввести data quality model минимум с такими полями:

- `normalization_status`, `valuation_status`, `confidence_score/tier`, `price_source`, `instrument_version`, `source_kind` (`snapshot`, `native_interval`, `backfill`, `replay`).
Это резко повысит надёжность алертов, потому что правила смогут фильтровать не только по величине дельты, но и по качеству исходных данных.

Нужно явно решить, что именно считается “истиной” для окон 5/15/30:

- exchange-native bars, если доступны;
- derived windows from snapshots, если native bars нет;
- единый приоритет и fallback policy, если доступны оба источника. Сейчас это implied, но не formalized.


## Вывод о полноте ТЗ

На текущем этапе ТЗ можно считать хорошим концептуальным черновиком и неплохим architecture draft, но не завершённой технической спецификацией для уверенной реализации без дополнительных решений. Основные пробелы — data contracts, временная модель, quality semantics, reprocessing strategy, instrument master-data lifecycle и формализация сопоставимости данных между биржами.
Если стартовать разработку прямо сейчас, команда почти наверняка быстро упрётся в неоднозначности и начнёт “додумывать” систему в коде, а это самый дорогой способ закрывать требования.

Ниже — вопросы, без которых не стоит финализировать новую версию md-спеков:

- Что является primary time axis для всех окон и алертов: `tsexchange` без исключений?
- - Нужно однозначно считать primary осью времени tsexchange для всех 5/15/30-окон и алертов.
- Разрешены ли mixed-source windows, когда часть истории native, а часть достроена snapshot-ами?
- - Допускаем mixed-source окна, но с явным приоритетом и аннотацией качества.
Предлагаемая политика:
1. Если биржа даёт native interval (Bybit, KuCoin), то для 5/15/30 берём их как source_kind = native_interval и считаем их authoritative для данной биржи, если не доказано обратное (например, проблемы ретенции).
2. Для snapshot-only бирж (Binance openInterest, OKX/Bitget/MEXC/… в режиме snapshot) считаем окна на своей стороне, source_kind = derived_snapshot.
3. Если у конкретной биржи доступны и native bars, и возможность derived, используем только один канал в проде (по умолчанию native) и фиксируем fallback: при деградации native history временно переключаемся на snapshot-derived, но ставим source_kind = degraded_fallback и опускаем важность алертов/дополнительно фильтруем.

Таким образом mixed-source возможен на уровне “по одной бирже native, по другой derived”, но внутри конкретного exchange+symbol в нормальном режиме используем один доминирующий источник.

- Должны ли алерты пересчитываться/откатываться после late backfill или replay?
- - Да, система должна уметь пересчитывать метрики, но не делать ретро-алерты пользователю.
Политика:
- backfill и replay обновляют normalizedoisamples и агрегаты, выравнивая исторические ряды, чтобы аналитика и дашборды были корректными;
- алерт-движок работает только на “актуальном срезе” вперёд: он не создаёт новые FIRE-события задним числом и не шлёт в Telegram алерты за прошлые окна, даже если после backfill’а стало ясно, что тогда было пересечение порога;
- при backfill’е алерт-движок может:
- - либо игнорировать изменения в исторических bucket’ах;
- - либо помечать уже отправленные алерты как “historically_inaccurate” в внутреннем alertevents, но без уведомления пользователя;
- для realtime окон (последние N bucket’ов) можно разрешить лёгкую “подтяжку” метрик, но без изменения уже отправленных сообщений; новый FIRE допускается только, если событие происходит в текущем/будущем bucket’е по tsexchange.

Иначе система превращается в ретро-симулятор, а не в живой алертинг.

- Какой уровень достоверности нужен для firing: `authoritative notional only` или допускаются estimated conversions?
- - Для боевых сигналов ставим правило:
- по умолчанию алерты разрешены только по данным, где oinotionalusdt имеет статус не ниже “good_estimate”;
- authoritative notional (как Binance sumOpenInterestValue или биржи с native notional) считается максимальным уровнем качества, good_estimate (contracts × mark price × contractSize) — допустимым, low_confidence — только для UI/heatmap, но не для алертов.

Это можно формализовать как:
- valuation_status ∈ {authoritative, good_estimate, low_confidence};
- алерт-правило имеет параметр min_valuation_status, по умолчанию good_estimate, а для “тяжёлых” сигналов можно требовать authoritative.

Плюс:
- если source_kind = degraded_fallback (когда перешли с native interval на snapshot-derived), можно автоматически повышать минимальный порог дельты или отключать часть стратегий (например, divergence).
