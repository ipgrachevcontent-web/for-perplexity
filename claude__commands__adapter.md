---
description: Открыть спецификацию exchange-адаптера и подготовить контекст для билда коннектора. Опционально вызвать exchange-adapter-builder.
argument-hint: <exchange> [--build]
---

Биржа: $ARGUMENTS

## Инструкции

1. Извлеки имя биржи из `$ARGUMENTS`. Допустимые значения:
   `binance`, `bybit`, `okx`, `bitget`, `gateio`, `mexc`, `kucoin`, `htx`, `hyperliquid`, `aster`, `bitmart`, `xt`.

2. Если имя не указано или невалидно — вывести список доступных бирж и попросить уточнить.

3. Прочитать **в основной контекст** (не через субагента):
   - `docs/11_EXCHANGE_ADAPTERS/_BASE_CONNECTOR.md`
   - `docs/11_EXCHANGE_ADAPTERS/<exchange>.md`
   - Релевантные пункты `00_DECISIONS_LOG.md` (особенно C1, C5, C12, и `source_kind` правила).

4. Кратко (5–8 пунктов) суммировать спеку именно этой биржи:
   - Endpoints (instruments / OI snapshot или native_interval / bulk).
   - Маппинг символов → `canonical_symbol = BASE`.
   - Quirks биржи (rate limits, payload отличия, settle currency).
   - Какой `source_kind` ожидается по умолчанию.
   - Особые поля valuation_status (если применимо).

5. Если в `$ARGUMENTS` есть флаг `--build`:
   - Подтверди у пользователя, что готов запустить `exchange-adapter-builder`.
   - Затем вызови subagent `exchange-adapter-builder` с темой «build/refresh `<exchange>` connector per spec».

6. Если флага `--build` нет — просто отчитаться о загруженной спеке и предложить дальнейшие шаги:
   - «Запустить `/adapter <exchange> --build` чтобы сгенерировать коннектор», или
   - «Запустить `decisions-guardian`, чтобы проверить существующий коннектор».
