# Концепция: Query и QueryManager

> **Статус:** стаб. Углубляет [architecture-concept.md §5.1–5.2](../00-overview/architecture-concept.md#5-ключевые-архитектурные-концепции).

## Scope этого документа (заполнить в фазе концепций/архитектуры)

- **Query как наблюдаемый объект**: модель состояния (фаза, `QueryExecutionStatus`,
  прогресс, ошибка, онтологии, поток строк), его жизненный цикл.
- **QueryManager**: реестр активных/исторических запросов; flow-движок (A–D);
  присвоение `QueryKind` и семантического ключа `subject = (QueryKind, targetId)`;
  владение polling / fetch-loop / retry / cancel / close.
- Связь с персистенцией (**QueryStore**) и нижним табом **Query Logs**.
- Контракты и сигнатуры — на фазе архитектуры.

Прообраз в POC: `SynapsDbClient` + `SynapsQueryStateManager`
(`ddbs_client/lib/src/services/synaps_client.dart`,
`.../manager/synaps_query_state_manager.dart`).
