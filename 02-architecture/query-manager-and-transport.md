# Архитектура: QueryManager + SynapsTransport

> **Статус:** дизайн v1 — **реализован и верифицирован** (флоу A/C/D + подписка) ·
> **Фаза:** архитектура (детальный дизайн) · **Дата:** 2026-06-06
> **Вход:** [концепция §4–§5, §7](../00-overview/architecture-concept.md) ·
> [PoC QtGrpc](poc-grpc-notes.md) · [ADR-0001](../adr/0001-grpc-stack.md)
> **Реализация:** см. [§14 Статус реализации](#14-статус-реализации-v1)

Детальный дизайн **доменного слоя работы с запросами** и **слоя транспорта**. Это
первый компонент фазы архитектуры: выносим логику из PoC-`SynapsClient` в правильные
слои по MVVM. Здесь — контракты, сигнатуры и машина состояний flow A–D, на которые
дальше опираются ViewModel'и и экраны.

Документ закрывает в рамках этого дизайна **Q4** (реактивность): чистые signals/slots
+ `Q_PROPERTY` + тонкий subject-роутер внутри QueryManager, **без отдельной
reactive-библиотеки**.

## Оглавление

1. [Место в слоях и зоны ответственности](#1-место-в-слоях-и-зоны-ответственности)
2. [Доменные типы запроса](#2-доменные-типы-запроса)
3. [SynapsTransport — контракт](#3-synapstransport--контракт)
4. [Query — наблюдаемая сущность](#4-query--наблюдаемая-сущность)
5. [QueryManager — контракт](#5-querymanager--контракт)
6. [Модель подписки (резолюция Q4)](#6-модель-подписки-резолюция-q4)
7. [Машина состояний flow A–D](#7-машина-состояний-flow-ad)
8. [Надёжность: retry, отмена, CANCELLED-гард](#8-надёжность-retry-отмена-cancelled-гард)
9. [QueryStore и восстановление после рестарта (F13)](#9-querystore-и-восстановление-после-рестарта-f13)
10. [Декомпозиция PoC SynapsClient → целевые компоненты](#10-декомпозиция-poc-synapsclient--целевые-компоненты)
11. [Диаграммы последовательностей](#11-диаграммы-последовательностей)
12. [Связь с ViewModel/QML](#12-связь-с-viewmodelqml)
13. [Открытые вопросы этого дизайна](#13-открытые-вопросы-этого-дизайна)
14. [Статус реализации (v1)](#14-статус-реализации-v1)

---

## 1. Место в слоях и зоны ответственности

```
Presentation (QML)
   │  bind: Query.* / QuerySubscription.*   |  call: ViewModel
ViewModel (C++)
   │  submit(QueryRequest) / subscribe(subject)
Application / Domain
   ├── QueryManager   — реестр, flow-движок A–D, подписки, retry, персистенция
   ├── Query          — наблюдаемое состояние одной операции (QObject)
   └── QueryStore     — персистенция/восстановление (file-store, per-user)
        │  async-примитивы по каждому QL-RPC (домен-DTO внутри)
Data / Transport
   └── SynapsTransport — единственный канал к DHGDBSharedAPI (QtGrpc),
                         user-token, proto↔домен; НИКАКОЙ flow-логики
```

**Правила зависимости** (жёстко): зависимости направлены только вниз. `SynapsTransport`
не знает про `Query`/`QueryManager`/flow. `QueryManager` не знает про QtGrpc/proto-типы
(их прячет транспорт). QML не видит ни транспорт, ни proto.

| Компонент | Делает | НЕ делает |
|---|---|---|
| **SynapsTransport** | один async-вызов на каждый QL-RPC; прикрепляет `user-token`; конвертит proto↔домен-DTO; отмена in-flight | flow A–D, polling, retry, реестр, persistence |
| **Query** | хранит наблюдаемое состояние одной операции | сетевые вызовы, оркестрация |
| **QueryManager** | принимает `QueryRequest`, гоняет flow, ведёт реестр, маршрутизирует подписки, retry/backoff, persistence/recovery | прямые gRPC-вызовы (только через транспорт), UI |
| **QueryStore** | сериализация записей запросов на диск (per-user) | бизнес-логика flow |

---

## 2. Доменные типы запроса

```cpp
// Какой это запрос — определяет реакцию UI (подписки). Расширяемый.
enum class QueryKind {
    AdhocSql,            // технический SQL Query экран — приложение НЕ реагирует
    CreateSourceIndex,
    CreateMetadata,
    LoadProject,
    SourcePreview,
    // … добавляется по мере появления экранов
};

// Как выполнять (4 flow из §4 концепции).
enum class QueryFlow { ExecuteDirect, Execute, FetchDirect, Fetch };
//  ExecuteDirect — A: статус один раз, без выборки
//  Execute       — B: polling статуса, без выборки
//  FetchDirect   — C: статус один раз + онтологии + fetch
//  Fetch         — D: polling статуса + онтологии + fetch

// Семантический ключ подписки. Value-тип, хешируемый (для QHash).
struct QuerySubject {
    QueryKind kind;
    QString   targetId;          // напр. имя источника "orders.csv"; пусто для AdhocSql
    bool operator==(const QuerySubject &) const = default;
};
size_t qHash(const QuerySubject &, size_t seed = 0) noexcept;

// Что подаётся в QueryManager::submit().
struct QueryRequest {
    QString    sql;
    QueryFlow  flow;
    QueryKind  kind     = QueryKind::AdhocSql;
    QString    targetId;                 // вместе с kind образует QuerySubject
    int        statusPollIntervalMs = 300;   // для flow B/D
    bool       reactive = true;          // false → не публиковать subject (как AdhocSql)
    QHash<QByteArray, QByteArray> extraMetadata;   // доп. заголовки к вызову
};
```

> `QueryKind` — точка расширения: добавление нового типа запроса не трогает ядро.
> `AdhocSql` (SQL Query экран) подаётся с `reactive=false` → ни один внешний
> подписчик не среагирует (§5.4 концепции).

Транспортный DTO для `QueryFact` (без proto-типов):

```cpp
struct QlKeyValue { quint64 index; QString key; QString value; };

struct QlFact {                          // = QueryFact
    QString description;                 // data_row / project_row / …
    quint64 factNo = 0;
    QList<QlKeyValue> data;

    // Регистронезависимый доступ к data[].
    std::optional<QString> value(QStringView key) const;
    QString executionStatus() const;     // value("QueryExecutionStatus"), нормализованный
};

struct QlFactCollection { QList<QlFact> rows; };   // = QueryFactCollection (онтологии)
```

---

## 3. SynapsTransport — контракт

Единственный владелец канала к `DHGDBSharedAPI`. Прячет QtGrpc/QtProtobuf целиком.

### 3.1 Статус и классификация ошибок

```cpp
struct QlStatus {
    int     code = 0;            // значение QtGrpc::StatusCode
    QString message;

    bool isOk()              const { return code == 0;  }   // Ok
    bool isEof()             const { return code == 11; }    // OutOfRange — конец выборки
    bool isUnauthenticated() const { return code == 16; }    // Unauthenticated
    bool isNotFound()        const { return code == 5;  }    // NotFound (нет queryId)
    bool isTransport()       const {                          // безопасно ретраить…
        return code == 1 || code == 4 || code == 14;         // Cancelled/DeadlineExceeded/Unavailable
    }                                                        // …с CANCELLED-гардом (§8)
};
```

> Коды — из `QtGrpc::StatusCode` (подтверждены в установленном Qt). `isTransport()`
> включает `Cancelled`, но повтор по нему разрешён только под гардом «клиент не
> инициировал отмену» — см. [§8](#8-надёжность-retry-отмена-cancelled-гард).

### 3.2 Async-обёртка вызова

Async-модель принята (Q2): event loop + сигналы `QGrpcCallReply::finished`, без
worker-потоков. Транспорт оборачивает это в свой объект-вызов `QlCall`, чтобы наружу
не торчали типы QtGrpc:

```cpp
// База с сигналом (moc-friendly: сигнал на не-шаблонном классе).
class QlCallBase : public QObject {
    Q_OBJECT
public:
    Q_INVOKABLE void abort();         // клиентская отмена in-flight вызова (QGrpcOperation::cancel)
    bool isFinished() const;
    QlStatus status() const;
signals:
    void finished(const QlStatus &status);   // ровно один раз
};

// Типизированный результат (значение валидно при status.isOk()).
template <class T>
class QlCall : public QlCallBase {
public:
    const T &value() const;           // T ∈ { QString(queryId), QlFact, QlFactCollection, void-маркер }
};
```

Использование в QueryManager:
```cpp
auto *call = transport->qlExecuteDirect(sql);
connect(call, &QlCallBase::finished, this, [this, call](const QlStatus &st){
    if (!st.isOk()) { /* … */ return; }
    const QString queryId = call->value();
    // …
    call->deleteLater();
});
```

> `QlCall<T>` владеет своим временем жизни до `finished` (как `QGrpcCallReply` в PoC,
> где `unique_ptr` захватывался в лямбду). Менеджер вызывает `deleteLater()` после
> обработки. Альтернатива — `std::shared_ptr`-владение; решается на реализации.

### 3.3 Интерфейс

```cpp
class SynapsTransport : public QObject {
    Q_OBJECT
public:
    // Конфигурация (пересоздаёт канал лениво при следующем вызове).
    void setEndpoint(const QString &url);          // "http://host:port" (h2c) или "https://…"
    void setUserToken(const QString &token);       // header user-token ко всем вызовам

    // По одному методу на каждый используемый QL-RPC.
    QlCall<QString>*          qlExecuteDirect(const QString &sql);    // → queryId
    QlCall<QlFact>*           qlQueryStatus(const QString &queryId);  // → статус-факт
    QlCall<QlFactCollection>* qlOntologies(const QString &queryId);   // → колонки
    QlCall<QlFact>*           qlFetch(const QString &queryId);        // → строка (или EOF)
    QlCall<void>*             qlCloseQuery(const QString &queryId);
    QlCall<void>*             qlCancelQuery(const QString &queryId);

signals:
    void unauthenticated();    // любой вызов вернул Unauthenticated → AuthService должен среагировать
};
```

`QLPrepare`/`QLExecute` **не входят** в интерфейс (решение Q6: старт всегда
`QLExecuteDirect`). `QLSetCursor` — **тоже не входит**: на сервере пока не
поддерживается (D2). Когда сервер добавит курсор, поддержка на клиенте делается
**отдельной фичей** (метод транспорта + оконная подгрузка `Query.rows`).

> Auth идёт через **отдельный** `AuthTransport` (сиблинг, тот же паттерн) — вне
> scope этого документа. Auth-мок запускается отдельно и пока опционален.

---

## 4. Query — наблюдаемая сущность

Одна операция = один `Query` (QObject), на свойства которого биндится UI.

```cpp
class Query : public QObject {
    Q_OBJECT
    Q_PROPERTY(QString   queryId         READ queryId         NOTIFY changed)
    Q_PROPERTY(QueryKind kind            READ kind            CONSTANT)
    Q_PROPERTY(QString   targetId        READ targetId        CONSTANT)
    Q_PROPERTY(Phase     phase           READ phase           NOTIFY changed)
    Q_PROPERTY(QString   executionStatus READ executionStatus NOTIFY changed)
    Q_PROPERTY(bool      running         READ running         NOTIFY changed)
    Q_PROPERTY(int       progress        READ progress        NOTIFY changed)   // -1 если неизвестно
    Q_PROPERTY(QString   error           READ error           NOTIFY changed)
    Q_PROPERTY(QStringList         columns READ columns       NOTIFY changed)   // flow C/D
    Q_PROPERTY(QAbstractItemModel* rows    READ rows          CONSTANT)         // flow C/D, модель строк
public:
    enum class Phase {
        Starting, PollingStatus, LoadingOntologies, Fetching,
        Closing, Completed, Failed, Cancelled
    };
    Q_ENUM(Phase)

    QuerySubject subject() const;        // {kind, targetId}
    bool isReactive() const;
signals:
    void changed();
    void finished(bool ok);
};
```

- `running == (phase ∈ {Starting, PollingStatus, LoadingOntologies, Fetching, Closing})`.
- `rows` — `QAbstractTableModel`, наполняется по мере `QLFetch`. Для AdhocSql и
  Source Preview UI биндит таблицу напрямую к `rows`.
- Владелец `Query` — `QueryManager`; UI держит **наблюдаемую** ссылку, не владеет.

---

## 5. QueryManager — контракт

Единая точка прохождения **всех** запросов.

```cpp
class QueryManager : public QObject {
    Q_OBJECT
public:
    // Подать запрос. Возвращает наблюдаемый Query (владеет менеджер).
    Q_INVOKABLE Query *submit(const QueryRequest &request);

    // Доступ к активным запросам.
    Q_INVOKABLE Query *byId(const QString &queryId) const;
    Q_INVOKABLE Query *bySubject(const QuerySubject &subject) const;   // активный для subject или nullptr

    // Подписка по семантическому ключу (см. §6).
    Q_INVOKABLE QuerySubscription *subscribe(const QuerySubject &subject);

    // Отмена (произвольная — выставляет cancel-флаг, см. §8).
    Q_INVOKABLE void cancel(const QString &queryId);

    // Модель активных/исторических запросов для нижнего таба Query Logs.
    QAbstractListModel *queriesModel();

signals:
    void queryStarted(Query *query);
    void queryFinished(Query *query, bool ok);
    void subjectChanged(const QuerySubject &subject);   // активный Query для subject появился/сменился/исчез
};
```

Внутри менеджера:
- **реестр**: `QHash<QString /*queryId*/, Query*>` + индекс `QHash<QuerySubject, QString>`
  для `bySubject`;
- **flow-движок** (§7) — по одному раннеру на активный `Query`;
- **роутер подписок** (§6);
- **retry/backoff + cancel-гард** (§8);
- **persistence/recovery** через `QueryStore` (§9).

---

## 6. Модель подписки (резолюция Q4)

**Паттерны:** Observer + Publish/Subscribe (topic = `QuerySubject`) + Mediator
(`QueryManager`). **Реализация:** чистые signals/slots + `Q_PROPERTY`, без внешней
reactive-библиотеки. Это и есть принятое решение **Q4**.

API подписки — отдельный объект `QuerySubscription`, удобный для биндинга в QML:

```cpp
class QuerySubscription : public QObject {
    Q_OBJECT
    Q_PROPERTY(Query *active READ active NOTIFY activeChanged)  // текущий Query для subject, или null
    Q_PROPERTY(bool   busy   READ busy   NOTIFY activeChanged)  // active && active->running
public:
    QuerySubject subject() const;
signals:
    void activeChanged();
};
```

Как это работает (сквозной пример «CREATE SOURCE INDEX» из §5.3 концепции):

1. ViewModel экрана источника: `auto *q = qm->submit({sql, Fetch, CreateSourceIndex, "orders.csv"})`.
2. **Три** разных места создают подписку на один subject и биндят `busy` к лоадеру:
   ```cpp
   auto *sub = qm->subscribe({CreateSourceIndex, "orders.csv"});  // экран источника
   auto *sub = qm->subscribe({CreateSourceIndex, "orders.csv"});  // строка файла на главной
   auto *sub = qm->subscribe({CreateSourceIndex, "orders.csv"});  // индикатор на табе
   ```
3. `QueryManager` при старте/смене/завершении активного `Query` для subject шлёт
   `subjectChanged(subject)`; каждый `QuerySubscription` обновляет `active`/`busy` →
   QML-binding автоматически гасит/зажигает лоадеры.

Никто из подписчиков не знает `queryId` и не управляет выполнением. Для `AdhocSql`
(`reactive=false`) `subjectChanged` не публикуется → внешних реакций нет.

> В QML: `property var sub: QueryManager.subscribe(subjectFor(...))` и
> `BusyIndicator { running: sub.busy }`.

---

## 7. Машина состояний flow A–D

Один раннер на `Query`. Шаги дёргают `SynapsTransport`; переходы — в обработчиках
`finished`. Это перенос логики PoC (`start → requestStatus → loadOntologies →
fetchNext → closeQuery`) в доменный слой, обобщённый по осям `poll` (B/D) и
`expectResults` (C/D).

```
submit(req)
  └─ qlExecuteDirect(sql) ──ok──> queryId  → Phase=PollingStatus
                          ──err─> finish(false)            // нет queryId → без close

PollingStatus: qlQueryStatus(queryId)
   exec==Executing & flow∈{B,D} & polls<max → wait(interval) → qlQueryStatus  (повтор)
   exec==Executing & предел                 → ok=false → close
   exec==Failed                             → ok=false → close
   exec==Completed:
        flow∈{A,B} (no results)             → close (ok=true)
        flow∈{C,D} (results)                → Phase=LoadingOntologies

LoadingOntologies: qlOntologies(queryId)
   ok  → columns ← rows[].value("name") → Phase=Fetching
   err → ok=false → close

Fetching: qlFetch(queryId)               // цикл
   ok       → Query.rows += factToRow(fact) → qlFetch (следующая)
   EOF(OutOfRange) → close (ok=true)
   err      → ok=false → close

Closing: qlCloseQuery(queryId)           // ВСЕГДА, в любом исходе
   → finish(ok, error)
```

Инварианты:
- `QLCloseQuery` вызывается при любом завершении (успех/ошибка/отмена), если `queryId`
  уже получен (§4.4 концепции).
- `QueryExecutionStatus` берётся из `QlFact.executionStatus()` (регистронезависимо).
- EOF — это `OutOfRange`, нормальный конец, не ошибка.

---

## 8. Надёжность: retry, отмена, CANCELLED-гард

Перенос политики из §4.5 концепции в контракт менеджера.

- **Retry** — только для идемпотентных шагов (`qlQueryStatus`, `qlCloseQuery`) и только
  на транспортные коды. Шаги `qlExecuteDirect`/`qlOntologies`/`qlFetch` — без retry.
- **Backoff** — экспоненциальный (1с → 2с → 5с), счётчик `transportErrorCount` хранится
  в `Query`/записи стора.
- **Коды**: `DeadlineExceeded`, `Unavailable` → повтор всегда; `Cancelled` → повтор
  **только если клиент сам не запрашивал отмену**.
- **Cancel-гард**: у `Query` есть флаг `cancelRequested`. `QueryManager::cancel()`
  выставляет его, шлёт `qlCancelQuery` + `qlCloseQuery`, и раннер **перед каждой
  попыткой** проверяет флаг → произвольная отмена терминальна (Phase=Cancelled),
  никогда не ретраится. Непроизвольный `Cancelled` (флаг не стоял) трактуется как
  обрыв транспорта и повторяется. Семантические (`NotFound`, `Unauthenticated`) — не
  ретраятся; `Unauthenticated` → сигнал `unauthenticated()` наверх.

```cpp
bool shouldRetry(const QlStatus &st, bool idempotentStep, const Query &q) {
    if (!idempotentStep || !st.isTransport()) return false;
    if (st.code == /*Cancelled*/1 && q.cancelRequested()) return false;  // гард
    return true;
}
```

---

## 9. QueryStore и восстановление после рестарта (F13)

**Решение Q3/Q5:** файловый стор, **отдельный для каждого пользователя**;
восстановление незавершённых запросов закладываем сразу.

```cpp
struct QueryRecord {
    QString    queryId;
    QString    sql;
    QueryKind  kind;
    QString    targetId;
    QueryFlow  flow;
    QString    phase;              // сериализованная Query::Phase
    QString    executionStatus;
    int        transportErrorCount = 0;
    QString    createdAt;          // ISO-8601 (время передаётся снаружи, не из Date.now)
};

class QueryStore {                 // файловая реализация; SQLite — позже за тем же интерфейсом
public:
    void put(const QueryRecord &);
    void remove(const QString &queryId);
    QList<QueryRecord> loadAll() const;
};
```

- **Локация:** `QStandardPaths::AppDataLocation/<userId>/queries/…`; выбирается при
  логине, переключается при logout (изоляция данных пользователей на одной машине).
- **Что персистится:** запись создаётся при `submit`, обновляется при смене фазы,
  удаляется при терминальном завершении (или переносится в «историю» для Query Logs).

**Серверная семантика (подтверждена, см. [§13.1](#131-вопросы-серверной-команде-по-reattach-d1)):**
`queryId` глобален; сервер **продолжает выполнение** после разрыва соединения; контекст
живёт **до перезапуска сервера** (отдельного TTL нет); к токену не привязан. Значит
**reattach при рестарте КЛИЕНТА реально возможен**, пока жив сервер. **Перезапуск
СЕРВЕРА теряет все `queryId`** — это естественный предел F13.

**Recovery при старте клиента.** `QueryManager` грузит незавершённые записи и по каждой
делает `qlQueryStatus(queryId)`:
- `Ok` + `Executing` → запрос жив и идёт → **возобновить polling** до `Completed`/`Failed`
  → `QLCloseQuery`. Это главный кейс reattach (долгие no-result операции вроде
  `CREATE SOURCE INDEX`).
- `Ok` + `Completed`/`Failed` → дождались исхода → `QLCloseQuery`, финализировать.
- `NotFound` *(TODO подтвердить код — D1.6)* → сервер уже не знает запрос (вероятно,
  перезапускался) → финализировать запись как незавершённую/orphaned.

**Каверзный кейс — результирующие flow (C/D) на момент рестарта клиента.** Результат
(`Query.rows`) хранится только в памяти и при рестарте теряется; `QLSetCursor` нет (D2),
поэтому переставить серверный курсор в начало нельзя, а возобновление с середины дало бы
неполную таблицу. **Решение v1:** reattach для result-flow доводим только до статуса
(`Completed`/`Failed`) и финализируем; **саму выборку не «дочитываем»** — если результат
нужен, пользователь перезапускает запрос. Основная ценность reattach — именно no-result
длительные операции.

---

## 10. Декомпозиция PoC SynapsClient → целевые компоненты

PoC-`SynapsClient` (один QObject, делал всё) разносится по слоям:

| PoC (`SynapsClient`) | Целевой компонент | Примечание |
|---|---|---|
| `QGrpcHttp2Channel` + `DHGDBSharedAPI::Client`, `ensureClient()`, `user-token` | **SynapsTransport** | + конвертация proto↔`QlFact`/DTO |
| `executeDirect/fetchDirect/fetch` (выбор `poll`/`expectResults`) | **QueryManager::submit** + flow-движок | оси сохранены |
| `requestStatus/loadOntologies/fetchNext/closeQuery` | flow-движок в **QueryManager** | дёргает транспорт |
| `queryId/executionStatus/columns/rowsFetched/running/error` | свойства **Query** | наблюдаемые |
| `log` (append-only) | **Query Logs** поверх `queriesModel()` | + структурное логирование |
| `flowFinished(bool)` | `Query::finished` / `QueryManager::queryFinished` | |
| `--grpc-selftest` (headless) | сохранить как dev/CI-инструмент | гоняет через новый стек |

PoC остаётся как референс-реализация и smoke-тест; продакшен-код строится по этому
дизайну.

---

## 11. Диаграммы последовательностей

**Flow A (executeDirect, без выборки):**
```
ViewModel   QueryManager     Query        SynapsTransport      Server
   │ submit(A) │                │                │                │
   │──────────>│ new Query ─────>│                │                │
   │           │ qlExecuteDirect(sql) ───────────>│ QLExecuteDirect>│
   │           │<───────────── finished(queryId) ─┤<───────────────┤
   │           │ set queryId, Phase=PollingStatus>│                │
   │           │ qlQueryStatus(queryId) ─────────>│ QLQueryStatus ─>│
   │           │<───────── finished(QlFact:Compl) ┤<───────────────┤
   │           │ qlCloseQuery(queryId) ──────────>│ QLCloseQuery ──>│
   │           │<───────────────── finished(ok) ──┤<───────────────┤
   │           │ Query.finished(true) ───────────>│                │
```

**Flow C/D (с выборкой):** после `Completed` —
```
   │ qlOntologies ─> columns ; loop[ qlFetch ─> Query.rows += row ] until OutOfRange ; qlCloseQuery
```
(flow D отличается только повтором `qlQueryStatus` через интервал, пока `Executing`).

---

## 12. Связь с ViewModel/QML

- **Подача запроса:** ViewModel экрана вызывает `QueryManager::submit(req)` и держит
  возвращённый `Query*` (биндит его свойства: лоадеры, статус, таблицу `rows`).
- **Реакция «со стороны»:** любой компонент создаёт `QuerySubscription` на нужный
  subject и биндит `busy`/`active` — не зная `queryId` (§6).
- **Query Logs:** нижний таб биндится к `QueryManager::queriesModel()`.
- **Регистрация в QML:** `QueryManager` — синглтон (`QML_SINGLETON`), как текущий
  `AuthService`; `Query`/`QuerySubscription` — uncreatable-типы (возвращаются из C++,
  биндятся в QML). `QueryKind`/`QueryFlow`/`Phase` — `Q_ENUM`.
- **MVVM-граница:** QML видит только `QueryManager` + `Query`/`QuerySubscription`;
  транспорт и proto скрыты.

---

## 13. Открытые вопросы этого дизайна

| # | Вопрос | Предложение / решение |
|---|---|---|
| D1 | **Reattach после рестарта**: переживает ли `queryId` рестарт клиента на сервере? | ✅ В основном отвечено ([§13.1](#131-вопросы-серверной-команде-по-reattach-d1)): reattach при рестарте клиента возможен (queryId глобален, сервер продолжает выполнение, без TTL до рестарта сервера, к токену не привязан). Перезапуск сервера теряет queryId. **TODO:** подтвердить код для забытого `queryId` (ожидаем `NOT_FOUND`). |
| D2 | **Большие результаты `QLFetch`**: всё в память vs оконная подгрузка. | ✅ **Решено: `QLSetCursor` пока не используем** (на сервере не поддерживается). v1 — накопление в память (`Query.rows`) последовательными `QLFetch` до EOF, с мягким лимитом + `log` об усечении. Курсорная/оконная выборка — **отдельная будущая фича** на клиенте, когда сервер добавит `QLSetCursor`. |
| D3 | **Лимит одновременных запросов** в QueryManager. | Ввести конфигурируемый предел + очередь; уточнить на нагрузочном профиле. |
| D4 | **Владение временем жизни `QlCall`**: `deleteLater` после `finished` vs `shared_ptr`. | Решить на реализации; `deleteLater` проще и достаточно. |
| D5 | **История vs активные** в `queriesModel()`: хранить ли завершённые и сколько. | Кольцевой буфер последних N + опционально из стора; уточнить с UX. |
| D6 | **Гранулярность subject** для проектов/веток (составной `targetId`). | Зафиксировать схему `targetId` per `QueryKind` отдельной таблицей. |

### 13.1 Вопросы серверной команде по reattach (D1)

Контекст: для F13 (восстановление после рестарта) клиент **персистит `queryId` и
состояние**, перезапускается (новый процесс → новое HTTP/2-соединение) и хочет либо
**переподключиться** к ещё живому запросу, либо корректно его финализировать. Нужно
понять серверную семантику жизненного цикла `queryId`. Конкретные вопросы:

1. **Область видимости `queryId`.** `queryId` привязан к gRPC-соединению/сессии или
   он глобален на сервере? Может ли **новое соединение** (после рестарта клиента)
   вызвать `QLQueryStatus`/`QLFetch`/`QLCloseQuery` со старым `queryId` и быть
   распознанным?

2. **Выполнение при разрыве соединения.** Если клиент отвалился во время выполнения
   (напр. `CREATE SOURCE INDEX`, статус `Executing`) — сервер **продолжает** операцию
   до конца независимо от клиента, или **прерывает/отменяет** её при разрыве?

3. **TTL контекста запроса.** Сколько сервер хранит контекст запроса (и результат)
   после последнего обращения клиента? Есть ли idle-timeout, который удалит `queryId`
   раньше, чем клиент перезапустится и переподключится? Что происходит с запросом,
   по которому так и не вызвали `QLCloseQuery` (утечка/сборка мусора)?

4. **Reattach по фазам.**
   - Был `Executing` на момент рестарта → может ли новое соединение опросить
     `QLQueryStatus` и дождаться `Completed`?
   - Был `Completed`, и клиент уже начал `QLFetch` (курсор сдвинут) → при
     переподключении `QLFetch` **продолжает с серверной позиции курсора** или курсор
     сбрасывается? (Без `QLSetCursor` — D2 — клиент не может переставить позицию, так
     что важно, переживает ли позиция курсора разрыв.)
   - Доступен ли ещё материализованный результат после паузы в обращениях клиента?

5. **Привязка к пользователю/токену.** `queryId` привязан к user-token/сессии?
   Допустим ли reattach с тем же `user-token` после рестарта? Станет ли `queryId`
   недействителен при смене токена/сессии?

6. **Код для неизвестного/протухшего `queryId`.** Какой gRPC-статус возвращает сервер
   для `queryId`, который он больше не знает? (Клиент рассчитывает на `NOT_FOUND` (5),
   чтобы отличить «финализировать» от «переподключиться».)

**Что меняет ответ в дизайне:** если «выполнение продолжается + `queryId` доступен с
нового соединения + результат/курсор переживают разрыв в пределах TTL» → F13 = живой
reattach (возобновляем polling/fetch). Иначе → F13 = best-effort: показать, что запрос
был, и финализировать запись как незавершённую (без живого возобновления).

#### Ответы серверной команды (2026-06-06, по текущему состоянию сервера)

> Для сервера gRPC — тоже лишь транспорт; все запросы живут **только в рамках
> запущенного процесса сервера**. Перезапуск сервера → все `queryId` теряются.

1. **Область видимости** — `queryId` **глобален** (пока что).
2. **Выполнение при разрыве** — сервер **продолжает** операцию.
3. **TTL** — отдельного TTL нет; контекст живёт **до перезапуска сервера**.
4. **Reattach по фазам** — `QLSetCursor` нет; с нового соединения **можно дождаться
   `Completed`**. Поведение курсора при reattach в середине `QLFetch` не специфицировано
   → в v1 не полагаемся на него (см. «каверзный кейс» в [§9](#9-querystore-и-восстановление-после-рестарта-f13)).
5. **Привязка к токену** — **не привязан**.
6. **Код для забытого `queryId`** — предположительно **`NOT_FOUND`**. **TODO:**
   подтвердить отдельно (от этого зависит ветка «финализировать vs переподключиться»).

**Вывод для дизайна:** F13 = **живой reattach при рестарте клиента** (пока жив сервер),
наиболее ценный для долгих **no-result** операций; перезапуск сервера и result-flow
в середине выборки → best-effort финализация. Зафиксировано в [§9](#9-querystore-и-восстановление-после-рестарта-f13).

---

## 14. Статус реализации (v1)

Доменный слой реализован в `synaps_client/src/ql/` и проверен через headless
self-test против `synaps_mock_server`.

**Файлы:**

| Файл | Содержит |
|---|---|
| `src/ql/QlTypes.h` | `QlKeyValue`/`QlFact`/`QlFactCollection`, `QlStatus` (классификация), `QlCallBase`+`QlCall<T>` |
| `src/ql/QueryTypes.h` | `QueryKind`/`QueryFlow`/`QuerySubject`(+`qHash`)/`QueryRequest` |
| `src/ql/SynapsTransport.{h,cpp}` | канал QtGrpc + клиент, `user-token`, async-методы QL, proto→DTO, `unauthenticated()` |
| `src/ql/Query.{h,cpp}` | `Query` (наблюдаемый) + `QueryResultModel` (таблица строк) |
| `src/ql/QueryManager.{h,cpp}` | `QueryManager` (QML-синглтон), `QuerySubscription`, внутренние `QueryRunner` (flow A–D) и `QueryListModel` |
| `src/qml/SqlQueryPage.qml` | **Экран SQL Query** — первый UI на доменном стеке |
| `src/main.cpp` | `--grpc-selftest [--flow A\|B\|C\|D] [--reactive]` через `QueryManager` |

**Проверено (exit 0):**
- **Flow A** (`executeDirect`, no-result), **C** (`fetchDirect`), **D** (`fetch`,
  polling) — полный путь через `QueryManager`+`SynapsTransport`+`Query`;
  онтологии → строки в `Query.rows` → EOF (`OutOfRange`) → `QLCloseQuery`.
- **Подписка**: `QuerySubscription` на реактивный subject видит переходы
  `busy: true → false` и `active: yes → no`, **не зная `queryId`** — Observer/PubSub
  работает (резолюция Q4 подтверждена на практике).
- `user-token` в метаданных; async без worker-потоков (event loop + сигналы).

**Решения реализации:**
- `QlCall<T>` — типизированный результат на не-шаблонной `QlCallBase` (moc-friendly:
  сигнал на базе, `value()` на шаблоне). Транспорт владеет временем жизни
  `QGrpcCallReply` (parented + `deleteLater` после `finished`) — без цикла захвата.
- `QueryRunner`/`QueryListModel` — внутренние, без `Q_OBJECT` (только контекст-получатели).

**UI (первый экран — SQL Query):**
- `QueryManager` зарегистрирован как **QML-синглтон** (`QML_SINGLETON`), `Query` —
  uncreatable QML-тип; QML-методы `runAdhoc(sql, flow)` / `cancel(query)`.
- `SqlQueryPage.qml`: поле SQL, выбор flow (A/B/C/D), Run/Cancel, статус
  (phase/executionStatus/queryId), таблица результатов (`TableView` ↔ `Query.rows`),
  индикатор загрузки (`BusyIndicator ↔ Query.running`). Встроен в центральную область
  `HomePage`. Запрос — `AdhocSql` (`reactive=false`).
- Проверено: чистая сборка (qmlcachegen) + загрузка без QML-ошибок (offscreen).
  Визуальная проверка — на машине с дисплеем (login `ma`/`x` → экран).

**Отложено (следующие инкременты):**
- **QueryStore + восстановление (F13, §9)** — персистенция и reattach не реализованы.
- **`queriesModel()`** — минимальный (display-строка); роли/`roleNames` для Query Logs — позже.
- **Отмена**: `QueryManager::cancel()` + cancel-гард реализованы, но ещё не покрыты
  тестом (mock завершает запросы мгновенно — не успеть отменить).
- PoC-`SynapsClient` теперь **вытеснен** этим стеком; оставлен в дереве как референс.

---

## Следующие шаги

1. Согласовать этот контракт; закрыть D1 (серверная семантика reattach) и D2.
2. Реализовать **SynapsTransport** (вынести из PoC `SynapsClient`), затем
   **QueryManager** (flow-движок + подписки), затем **QueryStore** (file, per-user).
3. Подключить первый экран — **SQL Query** (`AdhocSql`, `reactive=false`) поверх
   `submit` + `Query.rows`, как вертикальный срез.
4. Дальше — `TabManager` и остальные экраны (отдельные документы фазы архитектуры).
