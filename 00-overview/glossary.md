# Глоссарий

Единая терминология проекта. Краткая версия встроена в
[architecture-concept.md §2](architecture-concept.md#2-глоссарий-кратко).

## Запросы и протокол QL

- **Query (Запрос)** — логическая операция к БД, выраженная SQL-подобным текстом.
  Центральная сущность приложения; моделируется как наблюдаемый объект с состоянием.
- **QueryKind** — внутренний тип запроса в приложении (enum). Определяет реакцию UI
  (подписки). Примеры: `CreateSourceIndex`, `CreateMetadata`, `LoadProject`,
  `AdhocSql`.
- **Subject (семантический ключ)** — `(QueryKind, targetId)`. Ключ подписки, по
  которому UI наблюдает запрос **не зная `queryId`**. Пример: `(CreateSourceIndex, "orders.csv")`.
- **queryId** — идентификатор запроса на сервере; возвращается при старте, передаётся
  в `QueryRequest.text` во всех последующих вызовах цепочки.
- **Flow** — сценарий выполнения: `executeDirect` (A), `execute` (B), `fetchDirect`
  (C), `fetch` (D). Оси: с выборкой / без; direct / non-direct.
- **Direct / Non-direct** — оба стартуют через `QLExecuteDirect` (`QLPrepare` не
  используем — решение принято, как в POC). Direct: `QLQueryStatus` читается один
  раз (статус готов почти сразу). Non-direct: `QLQueryStatus` в режиме polling до
  завершения.
- **QueryExecutionStatus** — статус выполнения `Executing | Failed | Completed`.
  Приходит **внутри `QueryFact.data[]`** парой `key=QueryExecutionStatus` (читать
  регистронезависимо).
- **QueryFact** — ответ сервера: `description` (`data_row`/`project_row`/
  `project_metrix_row`), `factNo`, массив `data[]` из `QueryKeyValuePair`. Служит и
  статусом, и строкой данных.
- **QueryKeyValuePair** — ячейка: `index` (номер колонки), `key` (имя), `value`.
- **QueryFactCollection** — набор `QueryFact`; результат `QLOntologies` (описание
  колонок).
- **Ontology (Онтология)** — в контексте QL: описание колонки результирующей таблицы.
- **EOF на Fetch** — признак «строк больше нет»; приходит как **ошибка** ответа
  `QLFetch` (в POC — gRPC `OUT_OF_RANGE`), не как пустой результат.

## Сервер и сущности

- **DHGDBSharedAPI / SynapsDbClient** — основной gRPC-сервис доступа к данным через
  QL (`dhgdb.shared.api.proto`).
- **AuthService** — gRPC-сервис авторизации (`auth.api.proto`).
- **DdbsApiClient / DynamicHyperGraphDB** — старый специализированный сервис
  (`dhgdb.api.proto`), выводится из эксплуатации.
- **user-token** — токен пользователя, передаётся в header каждого вызова
  `DHGDBSharedAPI`.
- **DynaObject** — динамический объект сервера (проект/калькулятор) с метаданными
  (статус `Ready/Busy/Unprepared`, размеры, прогресс).
- **Source (Источник)** — источник данных (CSV / БД-таблица); просматривается и
  индексируется (CREATE METADATA / CREATE SOURCE INDEX).

## Приложение и UI

- **QueryManager** — единая точка прохождения всех запросов; реестр, flow-движок,
  маршрутизация событий по subjects.
- **TabManager** — реестр и владелец состояния табов.
- **Tab (Таб)** — рабочая вкладка. **TabKind** — тип таба (home, SQL Query, source
  preview, project, branch, rdb-project, ...).
- **Single-instance таб** — тип таба, существующий только в одном экземпляре.
- **Домашний таб** — главный экран; всегда один, закрыть нельзя.
- **Sidebar** — скрываемая контекстная панель (левая/правая).
- **Bottom tab** — таб нижнего сайдбара; **контекстный** (напр. Metrix) или
  **глобальный** (напр. Query Logs).
- **ViewModel** — C++ `QObject`-граница между QML и доменом (паттерн MVVM).
