# ADR 0001 — Стек gRPC для Qt-клиента

**Статус:** ✅ Accepted (2026-06-06) — подтверждено PoC.
См. [02-architecture/poc-grpc-notes.md](../02-architecture/poc-grpc-notes.md).

## Контекст

Qt-клиент общается с сервером по gRPC (сервисы `AuthService` и `DHGDBSharedAPI`).
Нужно выбрать способ интеграции gRPC/Protobuf. Требования: неблокирующий UI
(асинхронность), удобная связка с QML/QObject, передача header `user-token`,
кроссплатформенность (Windows/macOS/Linux). В текущем каркасе gRPC ещё не подключён
(`AuthService` — mock).

## Варианты

1. **QtGrpc + QtProtobuf** (нативный модуль Qt 6).
   - **+** Нативно ложится на QObject/сигналы; проще async и связка с QML;
     единый инструментарий Qt/CMake.
   - **−** Относительно молодой модуль; нужно проверить зрелость/покрытие фич на
     целевых платформах и версии Qt.
2. **Raw `grpc++`** (обёртка в worker-потоки/исполнитель).
   - **+** Максимальная гибкость и контроль; зрелая библиотека.
   - **−** Больше ручной работы по связке с UI-потоком, маршалингу в QObject,
     управлению потоками; отдельная сборочная интеграция.

## Решение

**Принят QtGrpc + QtProtobuf.** Подтверждено PoC (flows **A, C, D** end-to-end против
`synaps_mock_server`): кодогенерация из `ddbs_proto`, cleartext HTTP/2 (h2c),
async без блокировки UI (event loop + сигналы `QGrpcCallReply::finished`, без
worker-потоков), `user-token` в метаданных, корректный разбор
`QueryResponse`/`QueryFact`/`QueryFactCollection` и `QueryExecutionStatus` из `data[]`,
выборка данных (`QLOntologies` + цикл `QLFetch`) с **EOF = `OUT_OF_RANGE`**. Модули и
кодогенераторы (`qtprotobufgen`/`qtgrpcgen`) доступны «из коробки» в Qt 6.10/6.11.

**Async-модель (закрывает Q2):** унарные вызовы → `std::unique_ptr<QGrpcCallReply>`,
результат сигналом `finished(QGrpcStatus)` в потоке UI; цепочка flow собирается
последовательными обработчиками. Потоковый Fetch и polling ложатся на ту же модель.

Не покрыто PoC (следующие инкременты): polling с реальным `Executing` (flow D сошёлся
за одну итерацию — старт всегда `QLExecuteDirect`), отмена (`QLCancelQuery`), живой
`Auth.Login` (отдельный auth-мок, опционален), сборка на Windows/Linux.

## Последствия

- Слой **Data/Transport** (`SynapsTransport`) строится на QtGrpc; async-модель —
  event loop + сигналы (Q2 закрыт).
- CMake: `qt_add_protobuf` + `qt_add_grpc(CLIENT)` из `ddbs_proto`; зависимости
  `Qt6::Protobuf`/`Qt6::Grpc`.
- **Зафиксировать одну версию Qt** в build/CI — нельзя смешивать codegen из одного
  Qt с runtime из другого (детали и обход package-registry — в PoC-заметках).

## Связи

- [architecture-concept.md §7.2, §9](../00-overview/architecture-concept.md#72-технологические-рекомендации)
