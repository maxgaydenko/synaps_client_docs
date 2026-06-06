# PoC: QtGrpc transport (flow A end-to-end)

> **Статус:** выполнено ✅ · **Дата:** 2026-06-06 · Закрывает открытые вопросы
> **Q1** (gRPC-стек) и **Q2** (async-модель) из
> [architecture-concept.md §9](../00-overview/architecture-concept.md#9-открытые-вопросы--кандидаты-в-adr).
> Решение зафиксировано в [adr/0001-grpc-stack.md](../adr/0001-grpc-stack.md).

## Цель

Проверить на реальном коде, что **QtGrpc + QtProtobuf** пригодны как слой
Data/Transport: кодогенерация из `ddbs_proto`, асинхронный неблокирующий вызов,
заголовок `user-token`, и сквозной **flow A** (`QLExecuteDirect → QLQueryStatus →
QLCloseQuery`) против `synaps_mock_server`.

## Результат: успех

```
-> QLExecuteDirect: SELECT preview_struct FROM datasource WHERE filename='test_t-30000.csv' AND codepage='UTF-8' AND delimiter=',' AND null_treatment='as_is'
   channel: http://localhost:5129  (user-token: set)
   queryId = 774e9089-9fd6-4558-b870-2fe8a9758d31
-> QLQueryStatus
   On = 1 | Ft = 1 | FetchCursorPosition = 0 | QueryExecutionStatus = Completed
-> QLCloseQuery
   closed
=== flow A OK ===   (exit 0)
```

Подтверждено end-to-end:
- кодогенерация QtProtobuf/QtGrpc из `dhgdb.shared.api.proto` **и** `auth.api.proto`;
- **cleartext HTTP/2 (h2c)** через `QGrpcHttp2Channel("http://…")` против grpc-js
  insecure-сервера — **TLS для локали не нужен**;
- декодирование `QueryResponse.queryId` и `QueryFact.data[]` (repeated);
- **`QueryExecutionStatus` извлекается из `data[]`** регистронезависимо = `Completed`
  — ровно как в контракте (§4.2 концепции);
- `user-token` уезжает в метаданных запроса;
- **асинхронность без ручных потоков** (см. Q2 ниже).

## Что построено (артефакты PoC)

- `synaps_client/CMakeLists.txt` — компоненты `Protobuf`/`Grpc`,
  `qt_add_protobuf(synaps_proto …)` + `qt_add_grpc(synaps_grpc CLIENT …)` из
  `ddbs_proto` (путь через кэш-переменную `DDBS_PROTO_DIR`).
- `synaps_client/src/SynapsClient.{h,cpp}` — `QObject` (PIMPL, генерённые grpc-типы
  спрятаны из заголовка), реализует flow A асинхронно; свойства/сигналы для QML.
- `synaps_client/src/main.cpp` — режим `--grpc-selftest ["<sql>"]` для headless-прогона.

Генерённые имена (Qt 6.x): `dhgdb.shared.api.qpb.h`,
`dhgdb.shared.api_client.grpc.qpb.h`; namespace `dhgdb::shared::api`, клиент
`DHGDBSharedAPI::Client`, методы `QLExecuteDirect(...)` → `std::unique_ptr<QGrpcCallReply>`.

## Ответ на Q2 (async-модель)

Унарные вызовы возвращают `std::unique_ptr<QGrpcCallReply>`; результат приходит
сигналом `finished(const QGrpcStatus&)` **в потоке UI через event loop Qt**. Цепочка
flow собирается соединением последовательных `finished`-обработчиков — **никаких
`QThread`/worker-потоков не требуется**, UI не блокируется.

→ **Рекомендуемая async-модель:** event loop + сигналы `QGrpcCallReply`; цепочкой
владеет доменный слой (QueryManager), не ViewModel/QML. Polling и потоковый Fetch
(flow C/D) ложатся на ту же модель (повторный вызов по таймеру / по приходу `finished`).

## Как воспроизвести

```bash
# 1. Сборка (из одного Qt — см. оговорку ниже)
cd synaps_client && cmake -S . -B build -G Ninja && cmake --build build

# 2. Mock-сервер (отдельный терминал)
cd synaps_mock_server && SOURCES_DIR=/Users/ma/wrk/bt/sources PORT=5129 npm start

# 3. Прогон flow A
SYNAPS_USER_TOKEN=poc-token \
  synaps_client/build/synaps_client.app/Contents/MacOS/synaps_client \
  --grpc-selftest "SELECT preview_struct FROM datasource WHERE filename='test_t-30000.csv' AND codepage='UTF-8' AND delimiter=',' AND null_treatment='as_is'"
# endpoint можно переопределить: SYNAPS_ENDPOINT=http://host:port
```

## Оговорки и находки окружения

1. **Не смешивать установки Qt.** На машине есть официальный Qt 6.10.1/6.10.2 (с
   gen-тулзами `qtprotobufgen`/`qtgrpcgen` в `libexec/`) и Homebrew Qt 6.11
   (полный, gen-тулзы в `share/qt/libexec/`). Опасность: codegen из одного Qt +
   runtime из другого = хрупкая сборка с потенциально несовместимым ABI. **Нужно
   собирать строго одним Qt.** Текущий PoC собран однородно на Homebrew 6.11.
2. **CMake находит Homebrew Qt по умолчанию** (он в дефолтном prefix). Чтобы
   принудительно взять официальный Qt: `-DCMAKE_PREFIX_PATH=<Qt>/macos`
   **и** отключить package-registry
   (`-DCMAKE_FIND_USE_PACKAGE_REGISTRY=OFF -DCMAKE_FIND_USE_SYSTEM_PACKAGE_REGISTRY=OFF`).
   Для CI/воспроизводимости — **зафиксировать одну версию Qt** (проект задокументирован
   под 6.10; решить отдельно, см. ниже).
3. **Mock-сервер не реализует Auth-сервис** — только `DHGDBSharedAPI`. Живой
   `Auth.Login` локально проверить не на чем; codegen `auth.api.proto` при этом
   компилируется без проблем (Auth-биндинги генерируются). Живой Login — когда будет
   доступен реальный auth-эндпоинт.
4. **Часть `… FROM system`-запросов в mock ходит в Postgres** (`DATABASE_URL`/
   `PGPASSWORD`); без БД они падают с `SASL … client password must be a string`
   (gRPC `INTERNAL`). Файловые `preview`/`datasource`-запросы работают in-memory из
   `SOURCES_DIR` — их и использовали для зелёного flow A.
5. `user-token` ставится через `QGrpcChannelOptions::setMetadata(QMultiHash<…>)`
   (канальные метаданные) либо `QGrpcCallOptions::addMetadata(...)` (на вызов).

## Что НЕ проверялось (следующие инкременты)

- Flow C/D: `QLOntologies` + цикл `QLFetch` + обработка **EOF как `OUT_OF_RANGE`**.
- Polling статуса (non-direct) с интервалом/таймаутом.
- Отмена (`QLCancelQuery`) и гард на произвольную отмену (см. §4.5 концепции).
- Живой `Auth.Login` против реального auth-сервиса.
- Сборка/прогон на Windows и Linux (PoC выполнен на macOS).

## Рекомендации

- **Q1 → принять QtGrpc** (см. ADR-0001).
- **Q2 → event loop + сигналы `QGrpcCallReply`**, без worker-потоков; цепочкой flow
  владеет QueryManager.
- Зафиксировать **одну версию Qt** в build-инструкциях и CI (включая отключение
  package-registry, если берём официальный Qt).
- Следующий инкремент PoC — flow C/D (Ontologies + Fetch + EOF) как мост к
  реальному QueryManager.
