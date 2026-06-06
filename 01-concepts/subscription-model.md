# Концепция: модель подписки на запрос

> **Статус:** стаб. Углубляет [architecture-concept.md §5.3](../00-overview/architecture-concept.md#53-модель-подписки).

## Scope этого документа (заполнить в фазе концепций/архитектуры)

- Паттерны: **Observer + Publish/Subscribe (topic-based)** с **Mediator**
  (QueryManager). Топик = семантический ключ `subject = (QueryKind, targetId)`.
- Реализация на Qt: `Q_PROPERTY` + `NOTIFY` для состояния запроса; broadcast по
  subject через сигналы QueryManager / event-bus; `QAbstractListModel` для списков.
- Сквозной сценарий **CREATE SOURCE INDEX**: один запрос → лоадеры в 3 местах
  (экран источника, строка файла на главной, индикатор на табе) — подписка по
  subject без знания `queryId`.
- Правила жизненного цикла подписок (attach/detach, отсутствие активного запроса,
  гонки при завершении).
- Особый случай: `AdhocSql` (SQL Query экран) — без внешних подписчиков.

Прообраз в POC: `ReactiveEventBus` (`ddbs_client/lib/src/bloc/reactive/`).
