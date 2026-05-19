# Binance
## Принцип работы
Binance-коннектор для USDT-M надо строить как snapshot-poller + optional historical reconciler. Публичный endpoint GET /fapi/v1/openInterest возвращает текущее OI по конкретному символу и имеет маленький вес запроса, но он не отдаёт пачку всех символов одним вызовом, поэтому коннектор должен опрашивать каждый активный symbol отдельно по расписанию.

Для окон 5/15/30 минут у Binance у тебя два режима:
- основной: часто опрашивать текущее OI и самому строить временной ряд;
- дополнительный: использовать отдельный historical/statistics endpoint для сверки и backfill, если тебе нужна корректировка пропусков. Binance документирует и текущее OI, и отдельную статистику Open Interest, так что этот коннектор лучше делать двухконтурным.

## Как должен работать адаптер
- sync_instruments() получает список только активных USDT perpetual.
- Планировщик разбивает символы на батчи по приоритету ликвидности.
- fetch_snapshot(symbol) ходит в /fapi/v1/openInterest.
- Каждая точка пишется как raw sample с symbol, openInterest, exchange_time.
- Окна 5/15/30 минут считаются уже в БД или в агрегаторе.

## Что учитывать
Главная особенность Binance — ты почти наверняка будешь собирать OI своими снапшотами, а не готовыми свечами OI. Поэтому для “без потерь” нужен частый polling, например 15–30 секунд для топовых символов и 60 секунд для хвоста, плюс gap detector и backfill-процедура на случай лагов.

---

# Bybit
## Принцип работы
Bybit-коннектор — один из самых удобных для твоего use case, потому что GET /v5/market/open-interest уже умеет выдавать OI по linear контрактам с интервалами 5min, 15min, 30min, 1h, 4h, 1d, а также пагинацию курсором. Для USDT-M тебе нужен category=linear, а не inverse. Это значит, что на Bybit коннектор не обязан строить 5/15/30-минутные окна из сырых снапшотов с нуля. Он может регулярно тянуть готовые интервальные точки и хранить их как “exchange-native bars”, а потом поверх этого уже считать свои алерты и derived metrics.

## Как должен работать адаптер
- sync_instruments() берёт список linear perpetual.
- fetch_range(symbol, interval, start, end) вызывает /v5/market/open-interest.
- Коннектор идёт инкрементально: для каждого symbol хранит last_ts_fetched.
- Если ответ длинный, догружает страницы через nextPageCursor.
- В raw storage сохраняет и родной ответ, и нормализованные точки.

## Что учитывать
Bybit прямо пишет, что при экстремальной волатильности возможны задержки доставки данных. Ещё важнее, что unit open interest зависит от типа контракта: для BTCUSDT linear значение в BTC, а не в USDT, поэтому коннектор обязан домножать на цену и хранить oi_notional_usdt, иначе алерты по разным биржам будут несопоставимы.

---

# OKX
## Принцип работы
OKX-коннектор надо делать как instrument-driven poller для SWAP-инструментов с отдельной нормализацией naming scheme. В найденных источниках есть подтверждение публичного open-interest для OKX через клиентские обёртки и документацию по endpoint list, так что логика здесь похожа на Binance: получаешь список swap-инструментов, потом регулярно забираешь OI по каждому инструменту или по допустимому батчу API.

## Как должен работать адаптер
- sync_instruments() фильтрует только USDT-margin SWAP.
- native_symbol у OKX обычно не совпадает с Binance-style symbol, поэтому нужен слой canonical_symbol.
- fetch_snapshot(instId) забирает OI и timestamp.
- Коннектор сразу приводит всё к BTCUSDT-виду для общей системы.
- Историю 5/15/30 строит либо из снапшотов, либо из exchange-native history, если endpoint доступен в финальной реализации.

## Что учитывать
У OKX чаще всего больше мороки не в самом OI, а в нормализации инструмента и единиц контракта. Поэтому коннектор должен хранить метаданные contractSize, ctVal, settleCcy, чтобы корректно перевести open interest из контрактов в notional USDT.

---

# Bitget
## Принцип работы
Bitget-коннектор надо строить через их mix/contract API, потому что деривативы у Bitget исторически идут не через spot API. Документация Bitget mix API подтверждает отдельный раздел для деривативов, поэтому коннектор должен жить как отдельный exchange-adapter с собственным rate limit policy и symbol schema.

## Как должен работать адаптер
- sync_instruments() получает mix perpetual contracts.
- Определяет только USDT-settled perpetual.
- Если API даёт текущее OI — собирает snapshot stream.
- Если есть candles/history OI — использует incremental backfill по окнам.
- Для каждого symbol хранит last successful fetch и last normalized value.

## Что учитывать
У Bitget важно не смешать разные product types и margin coins. Коннектор должен явно фильтровать по типу продукта и quote/margin currency, иначе в агрегатор уйдут coin-margined контракты и всё сломают.

---

# Gate.io
## Принцип работы
Gate.io-коннектор строится как futures-only адаптер поверх v4 futures API. Найденные источники подтверждают наличие futures API у Gate.io, но для production тебе нужно отдельно зафиксировать точный endpoint OI и его семантику в коде интеграции перед запуском.

## Как должен работать адаптер
- sync_instruments() берёт futures contracts.
- Фильтрует только perpetual и только USDT-settled.
- Дальше два режима: snapshot poll либо history fetch, в зависимости от конкретного OI endpoint.
- Все сырые ответы обязательно сохраняются, потому что документация и SDK вокруг Gate.io часто разъезжаются по naming.

## Что учитывать
Для Gate.io я бы сразу делал strict parser versioning: если структура ответа меняется, raw payload всё равно останется, и ты не потеряешь данные. Это особенно важно для бирж, где документация и реальные ответы иногда не полностью синхронизированы.

---

# MEXC
## Принцип работы
MEXC-коннектор надо строить через futures/contract market endpoints. Публичная документация MEXC подтверждает раздел market endpoints для futures и отдельную contract API документацию, так что схема здесь стандартная: discover contract list, fetch OI/contract state, normalize, persist.

## Как должен работать адаптер
- sync_instruments() получает список contract symbols.
- Фильтрует USDT perpetual.
- fetch_snapshot(symbol) забирает текущее OI, если endpoint snapshot-only.
- Если MEXC даёт исторические точки — используешь fetch_range.
- Коннектор должен отдельно кешировать spec каждого контракта: multiplier, quote coin, settle coin.

## Что учитывать
У MEXC важно не полагаться только на “название похоже на USDT-пару”. Нужен строгий фильтр по типу контракта и settle currency из спецификации инструмента, иначе можно поймать не тот рынок.

---

# KuCoin
## Принцип работы
KuCoin-коннектор под твою задачу почти идеален, потому что endpoint GET /api/ua/v1/market/open-interest уже даёт OI по интервалам 5min, 15min, 30min, 1hour, 4hour, 1day и описывает retention. Для 5/15/30 минут это означает, что коннектор можно строить как incremental history puller, а не как частый snapshot poller.

## Как должен работать адаптер
- sync_instruments() получает futures symbols.
- fetch_range(symbol, interval, startAt, endAt, pageSize) тянет нужный диапазон.
- Для каждого окна ведётся свой watermark, чтобы не перекачивать всё заново.
- Если symbol не задан, API может вернуть all symbols, но в production лучше идти контролируемо по списку символов, чтобы не зависеть от размера ответа.

## Что учитывать
KuCoin прямо указывает retention: для 5m/15m/30m/1h/4h доступно 7 дней, для 1d — 70 дней. Это значит, что коннектор обязан регулярно снимать данные и не рассчитывать на бесконечный бэкофилл задним числом.

---

# HTX
## Принцип работы
HTX-коннектор стоит проектировать как derivatives-specific адаптер с приоритетом на отдельный public open-interest endpoint, потому что HTX давно держит отдельную линейку futures API и публикует историю изменений API для contract open interest. Найденные источники подтверждают именно наличие отдельной contract open interest части в HTX API-версии.

## Как должен работать адаптер
- sync_instruments() получает swap contracts.
- Дальше fetch_snapshot или fetch_range зависит от фактического endpoint, который ты зафиксируешь в коде.
- Обязательно маппишь HTX symbol naming в canonical symbol.
- Хранишь raw и normalized payload отдельно, потому что у HTX naming и поля исторически менялись между версиями API.

## Что учитывать
Для HTX я бы сразу сделал compatibility layer по версиям API. Даже если сейчас всё работает на одной версии, лучше изолировать parser, чтобы смена формата не затронула весь пайплайн.

---

# Hyperliquid
## Принцип работы
Hyperliquid-коннектор нужно строить не как “обычную CEX REST-интеграцию”, а как адаптер под их собственную developer API модель по perpetuals. Документация Hyperliquid для perpetuals подтверждена через их developer docs, поэтому тут правильнее сразу проектировать connector с акцентом на low-latency polling и аккуратную интерпретацию полей рынка.

## Как должен работать адаптер
- sync_instruments() получает список perp markets.
- fetch_snapshot(asset) тянет рыночное состояние и OI-поле, если оно есть в info/perpetuals response.
- Символы приводятся к общей canonical-схеме.
- Так как это не классическая биржа с binance-like символикой, нормализатор тут особенно важен.

## Что учитывать
У Hyperliquid основная сложность — не rate limit, а правильная адаптация их структуры рынка к общей схеме “биржа → USDT perp → symbol”. Поэтому коннектор должен иметь свой отдельный parser и unit conversion policy, а не копипаститься с Binance/Bybit.

---

# Aster
## Принцип работы
Aster-коннектор логично строить как binance-style derivatives adapter, потому что их docs используют /fapi/v1/...-паттерн и содержат exchangeInfo-подобную структуру rate limits. В найденной документации Aster есть futures API со схемой, очень похожей на Binance Futures, поэтому коннектор можно проектировать по тому же шаблону: discover instruments → per-symbol market endpoints → normalize.

## Как должен работать адаптер
- sync_instruments() читает futures exchangeInfo и фильтрует USDT perpetual.
- fetch_snapshot(symbol) идёт в рыночный endpoint OI, если он подтверждён в полном docs mapping.
- Naming и ошибки обрабатываются так же, как в binance-like биржах.
- В коде удобно выделить общий базовый класс BinanceLikeConnector, а Aster сделать наследником.

## Что учитывать
Хотя Aster выглядит binance-like, я бы не полагался на полную 1:1 совместимость. Для production нужно отдельно проверить, совпадают ли коды ошибок, лимиты и типы символов с ожиданием базового класса.

---

# Bitunix
## Принцип работы
Bitunix-коннектор можно строить как market-wide contract poller. Документация Bitunix подтверждает endpoint GET /api/v1/futures/market/trading_pairs для списка futures-пар, а rate limit указан как 10 req/sec/ip; это удобно, потому что сначала можно синхронизировать весь universe инструментов, а затем уже по каждому symbol получать OI или связанные market stats из futures market API.

## Как должен работать адаптер
- sync_instruments() ходит в /api/v1/futures/market/trading_pairs.
- Определяет только perpetual USDT pairs.
- Отдельный метод fetch_snapshot(symbol) тянет market details/OI из futures market endpoints.
- Все ответы нужно логировать с nonce/request_id, если документация указывает нестабильные схемы или review-based API access.

## Что учитывать
Bitunix ещё важен тем, что у них есть отдельные организационные ограничения на futures API onboarding, описанные в их guide. Даже если market endpoints публичны, я бы сразу разделил публичный market connector и приватный trading connector, чтобы потом не мешать контуры доступа.

---

# XT
## Принцип работы
XT-коннектор — один из самых удобных для snapshot-модели, потому что их публичный endpoint GET /future/market/v1/public/cg/contracts уже возвращает массив контрактов и включает поле open_interest, а base URL для USDT-M futures — https://fapi.xt.com. Это позволяет за один вызов забирать market info сразу по множеству контрактов, вместо отдельного запроса на каждый symbol.

## Как должен работать адаптер
- sync_instruments() и fetch_snapshots() можно объединить: один запрос получает сразу список контрактов и текущее open_interest.
- Коннектор фильтрует product_type=PERPETUAL и underlyingType, чтобы оставить только USDT-M.
- Потом каждая запись сохраняется как отдельный raw sample по symbol.
- Окна 5/15/30 минут XT-коннектор строит уже на своей стороне из периодических снапшотов.

## Что учитывать
Для XT надо внимательно трактовать, что именно означает open_interest в их ответе: в docs это обозначено как contracts, а не notional USD/USDT. Поэтому conversion в oi_notional_usdt обязателен через contractSize и цену контракта.

---

# Как это оформить в коде
| Тип коннектора                  | Биржи                                                                 | Принцип                                                                                       |
| ------------------------------- | --------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Snapshot per symbol             | Binance, OKX, Bitget, Gate.io, MEXC, HTX, Hyperliquid, Aster, Bitunix | Частый опрос текущего OI по каждому symbol, свои окна считаешь сам. developers.binance+7      |
| Native interval history         | Bybit, KuCoin                                                         | Биржа уже отдаёт 5m/15m/30m OI, коннектор тянет инкрементальную историю. developers.binance+1 |
| Bulk snapshot list              | XT                                                                    | Один endpoint может вернуть много контрактов с open_interest. github                          |
| Binance-like derivative adapter | Aster                                                                 | Можно унаследовать общий паттерн binance-style futures API. asterdex                          |

## Что обязательно сделать у всех
У каждого коннектора должны быть одинаковые защитные механизмы:
- watermark по последнему успешному времени загрузки.
- idempotent insert по ключу (exchange, symbol, ts_exchange).
- dead-letter log для нераспарсенных ответов.
- rate-limit aware scheduler.
- staleness detector по каждому symbol.
- unit normalization в oi_native и oi_notional_usdt.

*Самая важная инженерная мысль здесь такая: коннектор — это не просто HTTP-запрос, а мини-репликационный агент конкретной биржи. Там, где биржа даёт готовые интервалы, ты их инкрементально реплицируешь; там, где она даёт только текущий OI, ты сам строишь свой временной ряд из снапшотов, но делаешь это с watermark, дедупликацией, quality metrics и нормализацией единиц.*
