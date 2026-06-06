# Протокол QL — детальная спецификация

> **Статус:** стаб. Краткая формализация уже есть в
> [00-overview/architecture-concept.md §4](../00-overview/architecture-concept.md#4-протокол-ql);
> здесь — полная спецификация.

## Что здесь появится (заполнить в фазе архитектуры)

- Полное описание каждого QL-эндпоинта: поля request/response, предусловия,
  возможные ошибки и их семантика.
- Детальные диаграммы 4 flow (A `executeDirect`, B `execute`, C `fetchDirect`,
  D `fetch`) с обработкой ветвлений `Executing/Failed/Completed`, EOF на `QLFetch`,
  обязательным `QLCloseQuery`.
- Polling: интервалы, таймауты, верхние границы.
- Retry/backoff: классификация ошибок (транспортные vs семантические), политика
  повторов, безопасные/небезопасные для retry шаги.
- Отмена (`QLCancelQuery`) и корректное закрытие.
- Курсор (`QLSetCursor`): направления и абсолютное позиционирование.
- Уточнение non-direct старта: `QLPrepare`/`QLExecute` vs `QLExecuteDirect`
  (см. [adr/](../adr/), открытый вопрос Q6).
- Парсинг `QueryFact`/`QueryFactCollection` → доменные модели (онтологии, строки,
  статус).

Источники истины: `ddbs_proto/dhgdb.shared.api.proto`,
`ddbs_client/lib/src/services/synaps_client.dart`, `synaps_mock_server`.
