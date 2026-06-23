# synaps_client — Фаза 0: швы тестируемости и тестовый фундамент — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ввести швы тестируемости (`ISynapsTransport`, инъекция транспорта, выделенный `QueryRunner`, таблица переходов фаз) и поднять рабочий тестовый фундамент (ctest + qmltest), не меняя поведения приложения.

**Architecture:** Транспорт прячется за чистым интерфейсом `ISynapsTransport`; `SynapsTransport` (QtGrpc) и тестовый `FakeTransport` его реализуют. `QueryManager` получает конструктор с инъекцией транспорта, его внутренний флоу-движок `QueryRunner` выносится в отдельную единицу трансляции и зависит только от интерфейса. Тесты собираются в отдельную статическую библиотеку `synaps_testable`, к которой линкуются C++ юнит-тесты (QtTest) и qmltest-раннер.

**Tech Stack:** Qt 6.10+ (Core/Qml/Grpc/Protobuf/Test/QuickTest), C++20, CMake/Ninja, QtTest, Qt Quick Test.

> Спека: `docs/superpowers/specs/2026-06-23-synaps_client-redesign-plan.md` (Фаза 0).
> **Отклонение от §5 спеки:** абстракция времени (clock seam) перенесена в начало Фазы 1, где она нужна для детерминированных тестов backoff/поллинга. Фаза 0 покрывает transport-seam, выделение `QueryRunner`, таблицу переходов и харнесс.

## Global Constraints

- Qt **6.10+**, C++ **20** (`CMAKE_CXX_STANDARD 20`, extensions OFF).
- Сборка: CMake + Ninja. Все команды — из каталога-умбреллы `/Users/ma/wrk/bt`.
- Одна версия Qt для codegen и runtime (ABI; ADR-0001).
- Все строки, видимые пользователю, — **на английском**. Тестовые/лог-строки — свободно.
- **Никогда** не редактировать сгенерированный protobuf/gRPC-код вручную.
- `synaps_client` — отдельный git-репозиторий: коммиты делаются в нём (`git -C synaps_client …`).
- Никаких изменений поведения в этой фазе: существующий smoke `--grpc-selftest` должен продолжать собираться.

**Команды сборки и тестов (используются во всех задачах):**

```bash
# Конфигурация (один раз; build/ уже существует, повторный вызов безопасен)
cmake -S synaps_client -B synaps_client/build -G Ninja

# Сборка всего
cmake --build synaps_client/build

# Прогон тестов
ctest --test-dir synaps_client/build --output-on-failure
```

---

### Task 1: Интерфейс `ISynapsTransport` + реализация в `SynapsTransport`

**Files:**
- Create: `synaps_client/src/ql/ISynapsTransport.h`
- Modify: `synaps_client/src/ql/SynapsTransport.h`
- Modify: `synaps_client/CMakeLists.txt` (добавить `ISynapsTransport.h` в список исходников цели)

**Interfaces:**
- Produces: абстрактный QObject-класс `ISynapsTransport` с чисто-виртуальными методами конфигурации и QL-RPC и сигналом `unauthenticated()`. Сигнатуры методов идентичны текущим в `SynapsTransport`.

- [ ] **Step 1: Создать интерфейс**

Создать `synaps_client/src/ql/ISynapsTransport.h`:

```cpp
#pragma once

#include "QlTypes.h"

#include <QObject>
#include <QString>

// Pure transport contract over DHGDBSharedAPI. SynapsTransport (QtGrpc) is the
// production implementation; FakeTransport (tests) is the in-memory one. Nothing
// above this interface sees the wire format. One method per used QL-RPC; callers
// own the returned QlCall and deleteLater() it after `finished`.
class ISynapsTransport : public QObject
{
    Q_OBJECT
public:
    explicit ISynapsTransport(QObject *parent = nullptr) : QObject(parent) {}
    ~ISynapsTransport() override = default;

    virtual void setEndpoint(const QString &url) = 0;
    virtual void setUserToken(const QString &token) = 0;
    virtual void setDeadlineMs(int ms) = 0;
    virtual QString endpoint() const = 0;

    virtual QlCall<QString> *qlExecuteDirect(const QString &sql) = 0;
    virtual QlCall<QlFact> *qlQueryStatus(const QString &queryId) = 0;
    virtual QlCall<QlFactCollection> *qlOntologies(const QString &queryId) = 0;
    virtual QlCall<QlFact> *qlFetch(const QString &queryId) = 0;
    virtual QlCallBase *qlCloseQuery(const QString &queryId) = 0;
    virtual QlCallBase *qlCancelQuery(const QString &queryId) = 0;

signals:
    void unauthenticated(); // any call returned Unauthenticated
};
```

- [ ] **Step 2: Сделать `SynapsTransport` реализацией интерфейса**

В `synaps_client/src/ql/SynapsTransport.h` заменить базовый класс и убрать собственный сигнал (он наследуется). Заменить блок объявления класса:

Было (строки ~3 и ~17–41):
```cpp
#include "QlTypes.h"
```
```cpp
class SynapsTransport : public QObject
{
    Q_OBJECT
public:
    explicit SynapsTransport(QObject *parent = nullptr);
    ~SynapsTransport() override;

    void setEndpoint(const QString &url);    // "http://host:port" (h2c) or "https://…"
    void setUserToken(const QString &token); // attached as `user-token` to every call
    void setDeadlineMs(int ms);              // per-call deadline; <=0 keeps the default

    QString endpoint() const;
```
Стало:
```cpp
#include "ISynapsTransport.h"
#include "QlTypes.h"
```
```cpp
class SynapsTransport : public ISynapsTransport
{
    Q_OBJECT
public:
    explicit SynapsTransport(QObject *parent = nullptr);
    ~SynapsTransport() override;

    void setEndpoint(const QString &url) override;    // "http://host:port" (h2c) or "https://…"
    void setUserToken(const QString &token) override; // attached as `user-token` header
    void setDeadlineMs(int ms) override;              // per-call deadline; <=0 keeps the default

    QString endpoint() const override;
```

В этом же файле пометить QL-методы `override` и удалить блок `signals: void unauthenticated();` (наследуется от интерфейса). Заменить:
```cpp
    QlCall<QString> *qlExecuteDirect(const QString &sql);          // -> queryId
    QlCall<QlFact> *qlQueryStatus(const QString &queryId);         // -> status fact
    QlCall<QlFactCollection> *qlOntologies(const QString &queryId);// -> columns
    QlCall<QlFact> *qlFetch(const QString &queryId);               // -> row fact (or EOF)
    QlCallBase *qlCloseQuery(const QString &queryId);
    QlCallBase *qlCancelQuery(const QString &queryId);

signals:
    void unauthenticated(); // any call returned Unauthenticated

private:
```
на:
```cpp
    QlCall<QString> *qlExecuteDirect(const QString &sql) override;          // -> queryId
    QlCall<QlFact> *qlQueryStatus(const QString &queryId) override;         // -> status fact
    QlCall<QlFactCollection> *qlOntologies(const QString &queryId) override;// -> columns
    QlCall<QlFact> *qlFetch(const QString &queryId) override;               // -> row fact (or EOF)
    QlCallBase *qlCloseQuery(const QString &queryId) override;
    QlCallBase *qlCancelQuery(const QString &queryId) override;

private:
```

> `SynapsTransport.cpp` не меняется: `emit unauthenticated()` остаётся валидным (унаследованный сигнал), сигнатуры методов совпадают.

- [ ] **Step 3: Зарегистрировать заголовок в сборке**

В `synaps_client/CMakeLists.txt`, в списке исходников `qt_add_executable(synaps_client …)`, после строки `src/ql/QueryTypes.h` добавить:
```cmake
        src/ql/ISynapsTransport.h
```

- [ ] **Step 4: Собрать и убедиться, что приложение компилируется**

Run: `cmake -S synaps_client -B synaps_client/build -G Ninja && cmake --build synaps_client/build`
Expected: успешная сборка цели `synaps_client` без ошибок.

- [ ] **Step 5: Commit**

```bash
git -C synaps_client add src/ql/ISynapsTransport.h src/ql/SynapsTransport.h CMakeLists.txt
git -C synaps_client commit -m "refactor(ql): extract ISynapsTransport interface

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 2: Тестовый каркас (ctest) + первый sanity-тест

**Files:**
- Create: `synaps_client/tests/CMakeLists.txt`
- Create: `synaps_client/tests/tst_sanity.cpp`
- Modify: `synaps_client/CMakeLists.txt` (включить тесты)

**Interfaces:**
- Produces: статическая библиотека `synaps_testable` (доменные исходники для линковки тестами) и зарегистрированный в ctest тест `tst_sanity`.

- [ ] **Step 1: Написать падающий тест**

Создать `synaps_client/tests/tst_sanity.cpp`:

```cpp
#include <QtTest>

// Smoke test: proves the ctest harness builds, links and runs.
class TestSanity : public QObject
{
    Q_OBJECT
private slots:
    void arithmetic() { QCOMPARE(1 + 1, 2); }
};

QTEST_GUILESS_MAIN(TestSanity)
#include "tst_sanity.moc"
```

- [ ] **Step 2: Описать тестовую сборку**

Создать `synaps_client/tests/CMakeLists.txt`:

```cmake
find_package(Qt6 6.10 REQUIRED COMPONENTS Test)

# Domain/transport sources compiled once for all C++ tests. (The app target
# compiles the same sources independently; extracting a shared core library is
# a later improvement.)
add_library(synaps_testable STATIC
    ${CMAKE_SOURCE_DIR}/src/ql/Query.cpp
    ${CMAKE_SOURCE_DIR}/src/ql/QueryManager.cpp
    ${CMAKE_SOURCE_DIR}/src/ql/SynapsTransport.cpp
)
target_include_directories(synaps_testable PUBLIC
    ${CMAKE_SOURCE_DIR}/src
    ${CMAKE_SOURCE_DIR}/src/ql
)
target_link_libraries(synaps_testable PUBLIC
    Qt6::Core
    Qt6::Qml
    Qt6::Grpc
    Qt6::Protobuf
    synaps_proto
    synaps_grpc
)

qt_add_executable(tst_sanity tst_sanity.cpp)
target_link_libraries(tst_sanity PRIVATE Qt6::Test)
add_test(NAME tst_sanity COMMAND tst_sanity)
```

- [ ] **Step 3: Включить тесты в верхнем CMake**

В `synaps_client/CMakeLists.txt`, сразу после строки `qt_standard_project_setup(REQUIRES 6.10)`, добавить:
```cmake

# Tests (ctest). BUILD_TESTING is defined by CTest and defaults to ON.
include(CTest)
```
И в самом конце файла (после блока `install(...)`) добавить:
```cmake

if(BUILD_TESTING)
    add_subdirectory(tests)
endif()
```

- [ ] **Step 4: Сконфигурировать, собрать, прогнать тест**

Run: `cmake -S synaps_client -B synaps_client/build -G Ninja && cmake --build synaps_client/build && ctest --test-dir synaps_client/build --output-on-failure`
Expected: `tst_sanity` присутствует и проходит (`100% tests passed, 0 tests failed out of 1`).

- [ ] **Step 5: Commit**

```bash
git -C synaps_client add CMakeLists.txt tests/CMakeLists.txt tests/tst_sanity.cpp
git -C synaps_client commit -m "test: add ctest harness with synaps_testable library

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 3: `FakeTransport` (in-memory реализация интерфейса) + тест на него

**Files:**
- Create: `synaps_client/tests/fakes/FakeTransport.h`
- Create: `synaps_client/tests/fakes/FakeTransport.cpp`
- Create: `synaps_client/tests/tst_fake_transport.cpp`
- Modify: `synaps_client/tests/CMakeLists.txt`

**Interfaces:**
- Consumes: `ISynapsTransport` (Task 1), `QlTypes.h`.
- Produces: класс `FakeTransport : public ISynapsTransport` с публичными «ручками» (`queryId`, `executionStatus`, `columns`, `rowsToFetch`) и счётчиками вызовов (`executeDirectCount`, `statusCount`, `ontologiesCount`, `fetchCount`, `closeCount`, `cancelCount`). Все методы завершают `QlCall` асинхронно (queued), поэтому `finished`-подписка, сделанная сразу после вызова, его поймает. Фаза 0: только happy-path.

- [ ] **Step 1: Заголовок `FakeTransport`**

Создать `synaps_client/tests/fakes/FakeTransport.h`:

```cpp
#pragma once

#include "ISynapsTransport.h"
#include "QlTypes.h"

#include <QMetaObject>
#include <QString>
#include <QStringList>

// In-memory ISynapsTransport for unit tests. Every ql* method returns a QlCall
// that completes asynchronously (queued), so the runner's finished-connection,
// made right after the call returns, still catches it. Phase 0: happy path.
class FakeTransport : public ISynapsTransport
{
    Q_OBJECT
public:
    explicit FakeTransport(QObject *parent = nullptr) : ISynapsTransport(parent) {}

    // Configuration is a no-op for the fake.
    void setEndpoint(const QString &) override {}
    void setUserToken(const QString &) override {}
    void setDeadlineMs(int) override {}
    QString endpoint() const override { return QStringLiteral("fake://"); }

    QlCall<QString> *qlExecuteDirect(const QString &sql) override;
    QlCall<QlFact> *qlQueryStatus(const QString &queryId) override;
    QlCall<QlFactCollection> *qlOntologies(const QString &queryId) override;
    QlCall<QlFact> *qlFetch(const QString &queryId) override;
    QlCallBase *qlCloseQuery(const QString &queryId) override;
    QlCallBase *qlCancelQuery(const QString &queryId) override;

    // --- scripted happy-path response (test knobs) ---
    QString queryId = QStringLiteral("q-fake-1");
    QString executionStatus = QStringLiteral("Completed");
    QStringList columns{QStringLiteral("col")};
    int rowsToFetch = 0; // rows returned before EOF (flows C/D)

    // --- call counters for assertions ---
    int executeDirectCount = 0;
    int statusCount = 0;
    int ontologiesCount = 0;
    int fetchCount = 0;
    int closeCount = 0;
    int cancelCount = 0;

private:
    // Build a typed call and schedule its completion on the next event-loop turn.
    template <class T>
    QlCall<T> *makeCall(T value, QlStatus status = {})
    {
        auto *call = new QlCall<T>(); // owned by caller (runner deleteLater()s it)
        call->setValue(std::move(value));
        QMetaObject::invokeMethod(
            call, [call, status]() { call->complete(status); }, Qt::QueuedConnection);
        return call;
    }

    QlCallBase *makeBareCall(QlStatus status = {});

    int m_fetched = 0;
};
```

- [ ] **Step 2: Реализация `FakeTransport`**

Создать `synaps_client/tests/fakes/FakeTransport.cpp`:

```cpp
#include "FakeTransport.h"

QlCallBase *FakeTransport::makeBareCall(QlStatus status)
{
    auto *call = new QlCallBase();
    QMetaObject::invokeMethod(
        call, [call, status]() { call->complete(status); }, Qt::QueuedConnection);
    return call;
}

QlCall<QString> *FakeTransport::qlExecuteDirect(const QString &)
{
    ++executeDirectCount;
    return makeCall<QString>(queryId);
}

QlCall<QlFact> *FakeTransport::qlQueryStatus(const QString &)
{
    ++statusCount;
    QlFact fact;
    fact.description = QStringLiteral("status");
    fact.data.push_back({0, QStringLiteral("QueryExecutionStatus"), executionStatus});
    return makeCall<QlFact>(fact);
}

QlCall<QlFactCollection> *FakeTransport::qlOntologies(const QString &)
{
    ++ontologiesCount;
    QlFactCollection coll;
    for (const QString &name : columns) {
        QlFact f;
        f.data.push_back({0, QStringLiteral("name"), name});
        coll.rows.push_back(f);
    }
    return makeCall<QlFactCollection>(coll);
}

QlCall<QlFact> *FakeTransport::qlFetch(const QString &)
{
    ++fetchCount;
    if (m_fetched < rowsToFetch) {
        ++m_fetched;
        QlFact row;
        row.description = QStringLiteral("data_row");
        row.factNo = static_cast<quint64>(m_fetched);
        for (int c = 0; c < columns.size(); ++c)
            row.data.push_back(
                {static_cast<quint64>(c), columns.at(c), QStringLiteral("v%1").arg(m_fetched)});
        return makeCall<QlFact>(row);
    }
    return makeCall<QlFact>(QlFact{}, QlStatus{11, QStringLiteral("EOF")}); // OutOfRange
}

QlCallBase *FakeTransport::qlCloseQuery(const QString &)
{
    ++closeCount;
    return makeBareCall();
}

QlCallBase *FakeTransport::qlCancelQuery(const QString &)
{
    ++cancelCount;
    return makeBareCall();
}
```

- [ ] **Step 3: Написать падающий тест на `FakeTransport`**

Создать `synaps_client/tests/tst_fake_transport.cpp`:

```cpp
#include <QtTest>
#include <QSignalSpy>

#include "FakeTransport.h"

class TestFakeTransport : public QObject
{
    Q_OBJECT
private slots:
    void executeDirect_completes_with_queryId();
};

void TestFakeTransport::executeDirect_completes_with_queryId()
{
    FakeTransport fake;
    fake.queryId = QStringLiteral("q-7");

    QlCall<QString> *call = fake.qlExecuteDirect(QStringLiteral("SELECT 1"));
    QSignalSpy spy(call, &QlCallBase::finished);
    QVERIFY(spy.wait(1000));
    QVERIFY(call->status().isOk());
    QCOMPARE(call->value(), QStringLiteral("q-7"));
    QCOMPARE(fake.executeDirectCount, 1);
    call->deleteLater();
}

QTEST_GUILESS_MAIN(TestFakeTransport)
#include "tst_fake_transport.moc"
```

- [ ] **Step 4: Подключить Fake и тест к сборке**

В `synaps_client/tests/CMakeLists.txt` добавить `FakeTransport.cpp` в `synaps_testable` и расширить его include-пути. Заменить блок:
```cmake
add_library(synaps_testable STATIC
    ${CMAKE_SOURCE_DIR}/src/ql/Query.cpp
    ${CMAKE_SOURCE_DIR}/src/ql/QueryManager.cpp
    ${CMAKE_SOURCE_DIR}/src/ql/SynapsTransport.cpp
)
target_include_directories(synaps_testable PUBLIC
    ${CMAKE_SOURCE_DIR}/src
    ${CMAKE_SOURCE_DIR}/src/ql
)
```
на:
```cmake
add_library(synaps_testable STATIC
    ${CMAKE_SOURCE_DIR}/src/ql/Query.cpp
    ${CMAKE_SOURCE_DIR}/src/ql/QueryManager.cpp
    ${CMAKE_SOURCE_DIR}/src/ql/SynapsTransport.cpp
    fakes/FakeTransport.cpp
)
target_include_directories(synaps_testable PUBLIC
    ${CMAKE_SOURCE_DIR}/src
    ${CMAKE_SOURCE_DIR}/src/ql
    ${CMAKE_CURRENT_SOURCE_DIR}/fakes
)
```
И в конец файла добавить тест:
```cmake

qt_add_executable(tst_fake_transport tst_fake_transport.cpp)
target_link_libraries(tst_fake_transport PRIVATE synaps_testable Qt6::Test)
add_test(NAME tst_fake_transport COMMAND tst_fake_transport)
```

- [ ] **Step 5: Прогнать тест — должен пройти**

Run: `cmake --build synaps_client/build && ctest --test-dir synaps_client/build --output-on-failure -R tst_fake_transport`
Expected: `tst_fake_transport` проходит.

- [ ] **Step 6: Commit**

```bash
git -C synaps_client add tests/fakes/FakeTransport.h tests/fakes/FakeTransport.cpp tests/tst_fake_transport.cpp tests/CMakeLists.txt
git -C synaps_client commit -m "test: add in-memory FakeTransport and its unit test

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 4: Инъекция транспорта в `QueryManager` + первый сквозной тест flow A

**Files:**
- Modify: `synaps_client/src/ql/QueryManager.h`
- Modify: `synaps_client/src/ql/QueryManager.cpp`
- Create: `synaps_client/tests/tst_query_flows.cpp`
- Modify: `synaps_client/tests/CMakeLists.txt`

**Interfaces:**
- Consumes: `ISynapsTransport` (Task 1), `FakeTransport` (Task 3).
- Produces: новый конструктор `QueryManager(ISynapsTransport *transport, QObject *parent = nullptr)` (транспорт НЕ во владении менеджера); внутренний `m_transport` имеет тип `ISynapsTransport *`; `QueryRunner` принимает `ISynapsTransport *`.

- [ ] **Step 1: Написать падающий тест flow A**

Создать `synaps_client/tests/tst_query_flows.cpp`:

```cpp
#include <QtTest>
#include <QSignalSpy>

#include "FakeTransport.h"
#include "Query.h"
#include "QueryManager.h"
#include "QueryTypes.h"

class TestQueryFlows : public QObject
{
    Q_OBJECT
private slots:
    void flowA_executeDirect_completes();
};

void TestQueryFlows::flowA_executeDirect_completes()
{
    FakeTransport fake;
    fake.executionStatus = QStringLiteral("Completed");
    QueryManager mgr(&fake); // injected transport (not owned by the manager)

    QueryRequest req;
    req.sql = QStringLiteral("SELECT 1");
    req.flow = QueryFlow::ExecuteDirect;
    req.kind = QueryKind::AdhocSql;
    req.reactive = false;

    Query *q = mgr.submit(req);
    QVERIFY(q);
    QSignalSpy spy(q, &Query::finished);
    QVERIFY(spy.wait(2000));

    QCOMPARE(q->phase(), Query::Phase::Completed);
    QCOMPARE(spy.takeFirst().at(0).toBool(), true);
    QCOMPARE(fake.executeDirectCount, 1);
    QCOMPARE(fake.statusCount, 1);
    QCOMPARE(fake.closeCount, 1);
    QCOMPARE(fake.ontologiesCount, 0);
    QCOMPARE(fake.fetchCount, 0);
}

QTEST_GUILESS_MAIN(TestQueryFlows)
#include "tst_query_flows.moc"
```

- [ ] **Step 2: Зарегистрировать тест (должен не собираться — нет нужного конструктора)**

В `synaps_client/tests/CMakeLists.txt` в конец добавить:
```cmake

qt_add_executable(tst_query_flows tst_query_flows.cpp)
target_link_libraries(tst_query_flows PRIVATE synaps_testable Qt6::Test)
add_test(NAME tst_query_flows COMMAND tst_query_flows)
```

Run: `cmake -S synaps_client -B synaps_client/build -G Ninja && cmake --build synaps_client/build --target tst_query_flows`
Expected: ошибка компиляции — нет конструктора `QueryManager(ISynapsTransport*)`.

- [ ] **Step 3: Добавить конструктор с инъекцией и сменить тип транспорта**

В `synaps_client/src/ql/QueryManager.h`:

Заменить forward-declare:
```cpp
class SynapsTransport;
```
на:
```cpp
class ISynapsTransport;
```

Заменить объявление конструктора:
```cpp
    explicit QueryManager(QObject *parent = nullptr);
    ~QueryManager() override;
```
на:
```cpp
    explicit QueryManager(QObject *parent = nullptr);
    // Test seam: inject a transport (NOT owned by the manager).
    explicit QueryManager(ISynapsTransport *transport, QObject *parent = nullptr);
    ~QueryManager() override;
```

Заменить поле:
```cpp
    SynapsTransport *m_transport;
```
на:
```cpp
    ISynapsTransport *m_transport;
```

В `synaps_client/src/ql/QueryManager.cpp`:

Добавить include интерфейса рядом с существующим include транспорта. Заменить:
```cpp
#include "QlTypes.h"
#include "SynapsTransport.h"
```
на:
```cpp
#include "ISynapsTransport.h"
#include "QlTypes.h"
#include "SynapsTransport.h"
```

Заменить сигнатуру конструктора `QueryRunner`:
```cpp
    QueryRunner(QueryManager *mgr, SynapsTransport *transport, Query *query)
        : QObject(mgr), m_mgr(mgr), m_transport(transport), m_query(query)
    {
    }
```
на:
```cpp
    QueryRunner(QueryManager *mgr, ISynapsTransport *transport, Query *query)
        : QObject(mgr), m_mgr(mgr), m_transport(transport), m_query(query)
    {
    }
```

Заменить поле раннера:
```cpp
    SynapsTransport *m_transport;
```
на:
```cpp
    ISynapsTransport *m_transport;
```

Добавить второй конструктор менеджера сразу после существующего. Заменить:
```cpp
QueryManager::QueryManager(QObject *parent)
    : QObject(parent),
      m_transport(new SynapsTransport(this)),
      m_model(new QueryListModel)
{
    m_model->setParent(this);
    // Connection config (endpoint/deadline) is injected from AppConfig by main();
    // the token is wired in from AuthService.
}
```
на:
```cpp
QueryManager::QueryManager(QObject *parent)
    : QObject(parent),
      m_transport(new SynapsTransport(this)),
      m_model(new QueryListModel)
{
    m_model->setParent(this);
    // Connection config (endpoint/deadline) is injected from AppConfig by main();
    // the token is wired in from AuthService.
}

QueryManager::QueryManager(ISynapsTransport *transport, QObject *parent)
    : QObject(parent),
      m_transport(transport), // not owned: caller (test) keeps ownership
      m_model(new QueryListModel)
{
    m_model->setParent(this);
}
```

- [ ] **Step 4: Собрать и прогнать тест flow A — должен пройти**

Run: `cmake --build synaps_client/build && ctest --test-dir synaps_client/build --output-on-failure -R tst_query_flows`
Expected: `tst_query_flows` проходит (flow A: executeDirect → status(Completed) → close → Completed).

- [ ] **Step 5: Убедиться, что приложение по-прежнему собирается (без регресса)**

Run: `cmake --build synaps_client/build --target synaps_client`
Expected: успешная сборка.

- [ ] **Step 6: Commit**

```bash
git -C synaps_client add src/ql/QueryManager.h src/ql/QueryManager.cpp tests/tst_query_flows.cpp tests/CMakeLists.txt
git -C synaps_client commit -m "feat(ql): inject ISynapsTransport into QueryManager; add flow A test

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 5: Вынести `QueryRunner` в отдельную единицу трансляции

**Files:**
- Create: `synaps_client/src/ql/QueryRunner.h`
- Create: `synaps_client/src/ql/QueryRunner.cpp`
- Modify: `synaps_client/src/ql/QueryManager.cpp` (удалить встроенный `QueryRunner`, добавить include)
- Modify: `synaps_client/CMakeLists.txt` (исходники приложения)
- Modify: `synaps_client/tests/CMakeLists.txt` (исходники `synaps_testable`)

**Interfaces:**
- Consumes: `QueryManager` (friend), `Query`, `ISynapsTransport`, `QlTypes.h`, `QueryTypes.h`.
- Produces: класс `QueryRunner` в `QueryRunner.h/.cpp` с публичными `run()` и `requestCancel()` и конструктором `QueryRunner(QueryManager *mgr, ISynapsTransport *transport, Query *query)`. Поведение идентично прежнему встроенному классу (чистый рефактор под защитой теста flow A).

- [ ] **Step 1: Создать заголовок `QueryRunner.h`**

Создать `synaps_client/src/ql/QueryRunner.h`:

```cpp
#pragma once

#include "Query.h"
#include "QueryTypes.h"

#include <QObject>
#include <QPointer>
#include <QString>

#include <functional>

class QueryManager;
class ISynapsTransport;
class QlCallBase;

// Internal flow engine for one Query (flows A-D). Plain QObject (no Q_OBJECT):
// it owns no signals/slots, only acts as a context receiver for transport
// callbacks and timers. Created and owned by QueryManager.
class QueryRunner : public QObject
{
public:
    QueryRunner(QueryManager *mgr, ISynapsTransport *transport, Query *query);

    void run();
    void requestCancel();

private:
    const QueryRequest &req() const { return m_query->request(); }
    bool expectResults() const { return flowExpectsResults(req().flow); }
    bool polls() const { return flowPolls(req().flow); }
    bool cancelledNow() const { return m_query->cancelRequested() && !m_cancelling; }
    QString flowName() const;

    void requestStatus();
    void loadOntologies();
    void fetchNext();
    void closeQuery();

    void cancelPath();
    void fail(const QString &message, bool canClose);
    void failClose(const QString &message) { fail(message, /*canClose=*/true); }
    void finalize();

    bool retryIdempotent(std::function<void()> retry, const QlStatus &st);

    void track(QlCallBase *call) { m_current = call; }
    void cleanup(QlCallBase *call);
    void log(const QString &line);

    QueryManager *m_mgr;
    ISynapsTransport *m_transport;
    Query *m_query;
    QPointer<QlCallBase> m_current;
    int m_pollCount = 0;
    int m_rowCount = 0;
    int m_transportErrors = 0;
    bool m_ok = true;
    bool m_cancelling = false;
    bool m_waiting = false;
    QString m_pendingError;
};
```

- [ ] **Step 2: Создать `QueryRunner.cpp` (перенос тела как есть)**

Создать `synaps_client/src/ql/QueryRunner.cpp` — перенести реализацию из `QueryManager.cpp` (текущие строки 13–24 namespace-хелперов и 95–362 тело `QueryRunner`), адаптировав к out-of-class определениям:

```cpp
#include "QueryRunner.h"

#include "ISynapsTransport.h"
#include "QlTypes.h"
#include "QueryManager.h"

#include <QDebug>
#include <QStringList>
#include <QTimer>
#include <QtGlobal>

namespace {
constexpr int kMaxPolls = 200;
constexpr int kRowLogLimit = 5;

int backoffMs(int transportErrorCount)
{
    if (transportErrorCount <= 1)
        return 1000;
    if (transportErrorCount == 2)
        return 2000;
    return 5000;
}
} // namespace

QueryRunner::QueryRunner(QueryManager *mgr, ISynapsTransport *transport, Query *query)
    : QObject(mgr), m_mgr(mgr), m_transport(transport), m_query(query)
{
}

void QueryRunner::run()
{
    log(QStringLiteral("=== flow %1 ===").arg(flowName()));
    m_query->setPhase(Query::Phase::Starting);
    log(QStringLiteral("-> QLExecuteDirect: %1").arg(req().sql));
    auto *call = m_transport->qlExecuteDirect(req().sql);
    track(call);
    QObject::connect(call, &QlCallBase::finished, this, [this, call](const QlStatus &st) {
        cleanup(call);
        if (!st.isOk()) { // no queryId -> cannot close
            fail(QStringLiteral("QLExecuteDirect failed [%1]: %2").arg(st.code).arg(st.message),
                 /*canClose=*/false);
            return;
        }
        m_query->setQueryId(call->value());
        m_mgr->onQueryIdAssigned(m_query);
        log(QStringLiteral("   queryId = %1").arg(call->value()));
        if (cancelledNow())
            return cancelPath();
        requestStatus();
    });
}

void QueryRunner::requestCancel()
{
    if (m_cancelling)
        return;
    m_query->requestCancel();
    if (m_current)
        m_current->abort(); // its finished handler will route to cancelPath
    else if (m_waiting) {   // interrupted between polls
        m_waiting = false;
        cancelPath();
    }
}

QString QueryRunner::flowName() const
{
    switch (req().flow) {
    case QueryFlow::ExecuteDirect: return QStringLiteral("A (executeDirect)");
    case QueryFlow::Execute:       return QStringLiteral("B (execute)");
    case QueryFlow::FetchDirect:   return QStringLiteral("C (fetchDirect)");
    case QueryFlow::Fetch:         return QStringLiteral("D (fetch)");
    }
    return {};
}

void QueryRunner::requestStatus()
{
    m_query->setPhase(Query::Phase::PollingStatus);
    log(polls() ? QStringLiteral("-> QLQueryStatus (poll %1)").arg(m_pollCount + 1)
                : QStringLiteral("-> QLQueryStatus"));
    auto *call = m_transport->qlQueryStatus(m_query->queryId());
    track(call);
    QObject::connect(call, &QlCallBase::finished, this, [this, call](const QlStatus &st) {
        cleanup(call);
        if (cancelledNow())
            return cancelPath();
        if (!st.isOk()) {
            if (retryIdempotent([this] { requestStatus(); }, st))
                return;
            return failClose(
                QStringLiteral("QLQueryStatus failed [%1]: %2").arg(st.code).arg(st.message));
        }
        const QlFact &fact = call->value();
        const QString exec = fact.executionStatus();
        m_query->setExecutionStatus(exec.isEmpty() ? QStringLiteral("(not reported)") : exec);
        log(QStringLiteral("   QueryExecutionStatus = %1").arg(m_query->executionStatus()));

        const bool executing = exec.compare(QLatin1String("Executing"), Qt::CaseInsensitive) == 0;
        const bool failed = exec.compare(QLatin1String("Failed"), Qt::CaseInsensitive) == 0;

        if (executing) {
            if (polls() && m_pollCount < kMaxPolls) {
                m_pollCount += 1;
                m_waiting = true;
                QTimer::singleShot(req().statusPollIntervalMs, this, [this] {
                    m_waiting = false;
                    if (cancelledNow()) return cancelPath();
                    requestStatus();
                });
                return;
            }
            return failClose(QStringLiteral("still Executing after %1 polls").arg(m_pollCount));
        }
        if (failed)
            return failClose(QStringLiteral("server reported QueryExecutionStatus=Failed"));

        // Completed (or not reported)
        if (expectResults())
            loadOntologies();
        else
            closeQuery();
    });
}

void QueryRunner::loadOntologies()
{
    m_query->setPhase(Query::Phase::LoadingOntologies);
    log(QStringLiteral("-> QLOntologies"));
    auto *call = m_transport->qlOntologies(m_query->queryId());
    track(call);
    QObject::connect(call, &QlCallBase::finished, this, [this, call](const QlStatus &st) {
        cleanup(call);
        if (cancelledNow())
            return cancelPath();
        if (!st.isOk())
            return failClose(
                QStringLiteral("QLOntologies failed [%1]: %2").arg(st.code).arg(st.message));
        QStringList columns;
        for (const QlFact &row : call->value().rows) {
            if (const auto name = row.value(QStringLiteral("name")))
                columns << *name;
        }
        m_query->resetResult(columns);
        log(QStringLiteral("   %1 column(s): %2").arg(columns.size()).arg(columns.join(", ")));
        fetchNext();
    });
}

void QueryRunner::fetchNext()
{
    m_query->setPhase(Query::Phase::Fetching);
    auto *call = m_transport->qlFetch(m_query->queryId());
    track(call);
    QObject::connect(call, &QlCallBase::finished, this, [this, call](const QlStatus &st) {
        cleanup(call);
        if (cancelledNow())
            return cancelPath();
        if (st.isEof()) {
            log(QStringLiteral("   EOF after %1 row(s)").arg(m_rowCount));
            return closeQuery();
        }
        if (!st.isOk())
            return failClose(QStringLiteral("QLFetch failed [%1]: %2").arg(st.code).arg(st.message));
        QStringList cells;
        for (const QlKeyValue &kv : call->value().data)
            cells << kv.value;
        m_query->appendResultRow(cells);
        m_rowCount += 1;
        if (m_rowCount <= kRowLogLimit)
            log(QStringLiteral("   row %1: %2").arg(m_rowCount).arg(cells.join(" | ")));
        fetchNext();
    });
}

void QueryRunner::closeQuery()
{
    m_query->setPhase(Query::Phase::Closing);
    log(QStringLiteral("-> QLCloseQuery"));
    auto *call = m_transport->qlCloseQuery(m_query->queryId());
    track(call);
    QObject::connect(call, &QlCallBase::finished, this, [this, call](const QlStatus &st) {
        cleanup(call);
        if (!st.isOk() && retryIdempotent([this] { closeQuery(); }, st))
            return;
        if (!st.isOk())
            log(QStringLiteral("   close failed [%1]: %2 (finalizing anyway)")
                    .arg(st.code).arg(st.message));
        else
            log(QStringLiteral("   closed"));
        finalize();
    });
}

void QueryRunner::cancelPath()
{
    if (m_cancelling)
        return;
    m_cancelling = true;
    m_ok = false;
    m_pendingError = QStringLiteral("cancelled by user");
    log(QStringLiteral("!! %1").arg(m_pendingError));
    if (!m_query->queryId().isEmpty()) {
        auto *cancelCall = m_transport->qlCancelQuery(m_query->queryId()); // best-effort
        QObject::connect(cancelCall, &QlCallBase::finished, this,
                         [cancelCall](const QlStatus &) { cancelCall->deleteLater(); });
        closeQuery();
    } else {
        finalize();
    }
}

void QueryRunner::fail(const QString &message, bool canClose)
{
    m_ok = false;
    m_pendingError = message;
    log(QStringLiteral("!! %1").arg(message));
    if (canClose && !m_query->queryId().isEmpty())
        closeQuery();
    else
        finalize();
}

void QueryRunner::finalize()
{
    if (!m_pendingError.isEmpty())
        m_query->setError(m_pendingError);
    const Query::Phase term = m_cancelling ? Query::Phase::Cancelled
                              : m_ok        ? Query::Phase::Completed
                                            : Query::Phase::Failed;
    log(m_ok && !m_cancelling ? QStringLiteral("=== flow OK (rows: %1) ===").arg(m_rowCount)
                              : QStringLiteral("=== flow ENDED (%1) ===")
                                    .arg(m_cancelling ? "cancelled" : "failed"));
    m_query->finishWith(term);
    m_mgr->onQueryFinished(m_query, m_ok && !m_cancelling);
    deleteLater();
}

bool QueryRunner::retryIdempotent(std::function<void()> retry, const QlStatus &st)
{
    if (!st.isTransport())
        return false;
    if (st.code == 1 /*Cancelled*/ && m_query->cancelRequested())
        return false; // voluntary cancel — never retry
    m_transportErrors += 1;
    const int delay = backoffMs(m_transportErrors);
    log(QStringLiteral("   transient [%1]; retry in %2ms").arg(st.code).arg(delay));
    m_waiting = true;
    QTimer::singleShot(delay, this, [this, retry] {
        m_waiting = false;
        if (cancelledNow()) return cancelPath();
        retry();
    });
    return true;
}

void QueryRunner::cleanup(QlCallBase *call)
{
    if (m_current == call)
        m_current = nullptr;
    call->deleteLater();
}

void QueryRunner::log(const QString &line)
{
    qInfo().noquote() << "[QueryManager]" << line;
}
```

- [ ] **Step 3: Удалить встроенный `QueryRunner` из `QueryManager.cpp` и подключить заголовок**

В `synaps_client/src/ql/QueryManager.cpp`:

1. В блоке includes заменить:
```cpp
#include "ISynapsTransport.h"
#include "QlTypes.h"
#include "SynapsTransport.h"

#include <QAbstractListModel>
#include <QDebug>
#include <QPointer>
#include <QStringList>
#include <QTimer>
#include <QtGlobal>
```
на:
```cpp
#include "ISynapsTransport.h"
#include "QlTypes.h"
#include "QueryRunner.h"
#include "SynapsTransport.h"

#include <QAbstractListModel>
#include <QPointer>
```

2. Удалить весь анонимный namespace-блок с `kMaxPolls`/`kRowLogLimit`/`backoffMs` (текущие строки 13–25) — он переехал в `QueryRunner.cpp`.

3. Удалить весь блок `// ---- QueryRunner (internal flow engine A-D) ---` вместе с определением класса `QueryRunner` (текущие строки 91–362). Оставить `QueryListModel`, `QuerySubscription` и сам `QueryManager` без изменений.

> `forward declaration` `class QueryRunner;` в `QueryManager.h` остаётся — теперь он соответствует реальному типу из `QueryRunner.h`. Дружба `friend class QueryRunner;` продолжает работать.

- [ ] **Step 4: Добавить `QueryRunner` в обе сборки**

В `synaps_client/CMakeLists.txt`, в списке исходников `qt_add_executable(synaps_client …)`, после строки `src/ql/QueryManager.h` добавить:
```cmake
        src/ql/QueryRunner.cpp
        src/ql/QueryRunner.h
```

В `synaps_client/tests/CMakeLists.txt`, в список исходников `add_library(synaps_testable STATIC …)` после `${CMAKE_SOURCE_DIR}/src/ql/QueryManager.cpp` добавить:
```cmake
    ${CMAKE_SOURCE_DIR}/src/ql/QueryRunner.cpp
```

- [ ] **Step 5: Собрать всё и прогнать тесты — регресс не допускается**

Run: `cmake -S synaps_client -B synaps_client/build -G Ninja && cmake --build synaps_client/build && ctest --test-dir synaps_client/build --output-on-failure`
Expected: сборка `synaps_client` успешна; `tst_sanity`, `tst_fake_transport`, `tst_query_flows` проходят (flow A после рефактора по-прежнему зелёный).

- [ ] **Step 6: Commit**

```bash
git -C synaps_client add src/ql/QueryRunner.h src/ql/QueryRunner.cpp src/ql/QueryManager.cpp CMakeLists.txt tests/CMakeLists.txt
git -C synaps_client commit -m "refactor(ql): extract QueryRunner into its own translation unit

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 6: Таблица легальных переходов фаз `Query` + проверка в `setPhase`

**Files:**
- Create: `synaps_client/src/ql/QueryPhaseTransitions.h`
- Create: `synaps_client/src/ql/QueryPhaseTransitions.cpp`
- Modify: `synaps_client/src/ql/Query.cpp` (предупреждение при нелегальном переходе)
- Create: `synaps_client/tests/tst_phase_transitions.cpp`
- Modify: `synaps_client/CMakeLists.txt`, `synaps_client/tests/CMakeLists.txt`

**Interfaces:**
- Consumes: `Query::Phase` (из `Query.h`).
- Produces: свободная функция `bool isLegalQueryPhaseTransition(Query::Phase from, Query::Phase to)`. Переход в то же состояние всегда легален; терминальные фазы (Completed/Failed/Cancelled) не имеют исходящих переходов.

- [ ] **Step 1: Написать падающий тест таблицы переходов**

Создать `synaps_client/tests/tst_phase_transitions.cpp`:

```cpp
#include <QtTest>

#include "Query.h"
#include "QueryPhaseTransitions.h"

class TestPhaseTransitions : public QObject
{
    Q_OBJECT
private slots:
    void legalTransitions();
    void illegalTransitions();
    void sameStateIsLegal();
};

void TestPhaseTransitions::legalTransitions()
{
    using P = Query::Phase;
    QVERIFY(isLegalQueryPhaseTransition(P::Starting, P::PollingStatus));
    QVERIFY(isLegalQueryPhaseTransition(P::PollingStatus, P::LoadingOntologies));
    QVERIFY(isLegalQueryPhaseTransition(P::PollingStatus, P::Closing));
    QVERIFY(isLegalQueryPhaseTransition(P::LoadingOntologies, P::Fetching));
    QVERIFY(isLegalQueryPhaseTransition(P::Fetching, P::Closing));
    QVERIFY(isLegalQueryPhaseTransition(P::Closing, P::Completed));
    QVERIFY(isLegalQueryPhaseTransition(P::Starting, P::Cancelled));
}

void TestPhaseTransitions::illegalTransitions()
{
    using P = Query::Phase;
    QVERIFY(!isLegalQueryPhaseTransition(P::Completed, P::PollingStatus));
    QVERIFY(!isLegalQueryPhaseTransition(P::Starting, P::Fetching));
    QVERIFY(!isLegalQueryPhaseTransition(P::Fetching, P::PollingStatus));
    QVERIFY(!isLegalQueryPhaseTransition(P::Failed, P::Completed));
}

void TestPhaseTransitions::sameStateIsLegal()
{
    using P = Query::Phase;
    QVERIFY(isLegalQueryPhaseTransition(P::PollingStatus, P::PollingStatus));
    QVERIFY(isLegalQueryPhaseTransition(P::Fetching, P::Fetching));
}

QTEST_GUILESS_MAIN(TestPhaseTransitions)
#include "tst_phase_transitions.moc"
```

- [ ] **Step 2: Зарегистрировать тест (должен не собираться — нет заголовка)**

В `synaps_client/tests/CMakeLists.txt` в конец добавить:
```cmake

qt_add_executable(tst_phase_transitions tst_phase_transitions.cpp)
target_link_libraries(tst_phase_transitions PRIVATE synaps_testable Qt6::Test)
add_test(NAME tst_phase_transitions COMMAND tst_phase_transitions)
```

Run: `cmake -S synaps_client -B synaps_client/build -G Ninja && cmake --build synaps_client/build --target tst_phase_transitions`
Expected: ошибка компиляции — нет `QueryPhaseTransitions.h`.

- [ ] **Step 3: Реализовать таблицу переходов**

Создать `synaps_client/src/ql/QueryPhaseTransitions.h`:

```cpp
#pragma once

#include "Query.h"

// Legal Query::Phase transitions, matching the QueryRunner flow engine. Same-phase
// transitions are always legal (re-poll, re-fetch). Terminal phases (Completed /
// Failed / Cancelled) have no outgoing transitions. Used for observability:
// Query::setPhase warns (does not abort) on an illegal transition.
bool isLegalQueryPhaseTransition(Query::Phase from, Query::Phase to);
```

Создать `synaps_client/src/ql/QueryPhaseTransitions.cpp`:

```cpp
#include "QueryPhaseTransitions.h"

bool isLegalQueryPhaseTransition(Query::Phase from, Query::Phase to)
{
    using P = Query::Phase;
    if (from == to)
        return true; // re-poll / re-fetch

    switch (from) {
    case P::Starting:
        // executeDirect ok -> poll; cancel before/after queryId -> close or terminal;
        // executeDirect failure -> Failed (no close).
        return to == P::PollingStatus || to == P::Closing
            || to == P::Failed || to == P::Cancelled;
    case P::PollingStatus:
        return to == P::LoadingOntologies || to == P::Closing
            || to == P::Failed || to == P::Cancelled;
    case P::LoadingOntologies:
        return to == P::Fetching || to == P::Closing
            || to == P::Failed || to == P::Cancelled;
    case P::Fetching:
        return to == P::Closing || to == P::Failed || to == P::Cancelled;
    case P::Closing:
        return to == P::Completed || to == P::Failed || to == P::Cancelled;
    case P::Completed:
    case P::Failed:
    case P::Cancelled:
        return false; // terminal
    }
    return false;
}
```

- [ ] **Step 4: Подключить заголовок и предупреждать в `setPhase`**

В `synaps_client/src/ql/Query.cpp` заменить include:
```cpp
#include "Query.h"
```
на:
```cpp
#include "Query.h"

#include "QueryPhaseTransitions.h"

#include <QLoggingCategory>
```
И заменить тело `setPhase`:
```cpp
void Query::setPhase(Phase phase)
{
    if (m_phase == phase)
        return;
    m_phase = phase;
    emit changed();
}
```
на:
```cpp
void Query::setPhase(Phase phase)
{
    if (m_phase == phase)
        return;
    if (!isLegalQueryPhaseTransition(m_phase, phase))
        qWarning("Query: illegal phase transition %d -> %d",
                 static_cast<int>(m_phase), static_cast<int>(phase));
    m_phase = phase;
    emit changed();
}
```

- [ ] **Step 5: Добавить исходник в обе сборки**

В `synaps_client/CMakeLists.txt`, в списке исходников приложения после `src/ql/QueryTypes.h` (рядом с `ISynapsTransport.h` из Task 1) добавить:
```cmake
        src/ql/QueryPhaseTransitions.cpp
        src/ql/QueryPhaseTransitions.h
```

В `synaps_client/tests/CMakeLists.txt`, в список исходников `synaps_testable` после `${CMAKE_SOURCE_DIR}/src/ql/QueryRunner.cpp` добавить:
```cmake
    ${CMAKE_SOURCE_DIR}/src/ql/QueryPhaseTransitions.cpp
```

- [ ] **Step 6: Собрать и прогнать тесты — должны пройти**

Run: `cmake -S synaps_client -B synaps_client/build -G Ninja && cmake --build synaps_client/build && ctest --test-dir synaps_client/build --output-on-failure`
Expected: `tst_phase_transitions` проходит; остальные тесты остаются зелёными; `tst_query_flows` не печатает предупреждений о нелегальных переходах (flow A идёт по легальному пути).

- [ ] **Step 7: Commit**

```bash
git -C synaps_client add src/ql/QueryPhaseTransitions.h src/ql/QueryPhaseTransitions.cpp src/ql/Query.cpp tests/tst_phase_transitions.cpp CMakeLists.txt tests/CMakeLists.txt
git -C synaps_client commit -m "feat(ql): add Query phase transition table with setPhase guard

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

### Task 7: Каркас qmltest + дымовой QML-тест

**Files:**
- Create: `synaps_client/tests/qmltest_main.cpp`
- Create: `synaps_client/tests/qml/tst_smoke.qml`
- Modify: `synaps_client/tests/CMakeLists.txt`

**Interfaces:**
- Produces: исполняемый `synaps_qmltest`, прогоняющий `.qml`-тесты из `tests/qml` (вход через `-input`). Доказывает, что Qt Quick Test поднимается; реальные QML-тесты экранов появятся в Фазе 3.

- [ ] **Step 1: Написать дымовой QML-тест**

Создать `synaps_client/tests/qml/tst_smoke.qml`:

```qml
import QtQuick
import QtTest

TestCase {
    name: "Smoke"

    function test_arithmetic() {
        compare(1 + 1, 2);
    }
}
```

- [ ] **Step 2: Создать main для раннера**

Создать `synaps_client/tests/qmltest_main.cpp`:

```cpp
#include <QtQuickTest/quicktest.h>

QUICK_TEST_MAIN(synaps_qmltest)
```

- [ ] **Step 3: Описать сборку qmltest**

В `synaps_client/tests/CMakeLists.txt`:

1. Расширить `find_package`, заменив:
```cmake
find_package(Qt6 6.10 REQUIRED COMPONENTS Test)
```
на:
```cmake
find_package(Qt6 6.10 REQUIRED COMPONENTS Test QuickTest Qml Quick)
```

2. В конец файла добавить:
```cmake

qt_add_executable(synaps_qmltest qmltest_main.cpp)
target_link_libraries(synaps_qmltest PRIVATE Qt6::QuickTest Qt6::Qml Qt6::Quick)
add_test(NAME synaps_qmltest
    COMMAND synaps_qmltest -input ${CMAKE_CURRENT_SOURCE_DIR}/qml)
```

- [ ] **Step 4: Собрать и прогнать qmltest**

Run: `cmake -S synaps_client -B synaps_client/build -G Ninja && cmake --build synaps_client/build && ctest --test-dir synaps_client/build --output-on-failure -R synaps_qmltest`
Expected: `synaps_qmltest` проходит (1 TestCase "Smoke").

- [ ] **Step 5: Полный прогон всех тестов**

Run: `ctest --test-dir synaps_client/build --output-on-failure`
Expected: все тесты зелёные — `tst_sanity`, `tst_fake_transport`, `tst_query_flows`, `tst_phase_transitions`, `synaps_qmltest`.

- [ ] **Step 6: Commit**

```bash
git -C synaps_client add tests/qmltest_main.cpp tests/qml/tst_smoke.qml tests/CMakeLists.txt
git -C synaps_client commit -m "test: add Qt Quick Test (qmltest) harness with smoke test

Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>"
```

---

## Definition of Done (Фаза 0)

- `ISynapsTransport` введён; `SynapsTransport` и `FakeTransport` его реализуют.
- `QueryManager` принимает транспорт через конструктор (тестовый шов); `QueryRunner` зависит только от интерфейса и вынесен в отдельный `.h/.cpp`.
- Таблица легальных переходов фаз есть, протестирована и подключена к `setPhase` (предупреждение).
- `ctest` поднят: проходят `tst_sanity`, `tst_fake_transport`, `tst_query_flows` (flow A сквозь Fake), `tst_phase_transitions`.
- `qmltest` поднят: проходит `synaps_qmltest`.
- Приложение `synaps_client` собирается без регресса; поведение не изменено.
- Следующий шаг (вне этой фазы): `qt-cpp-review` по затронутому коду и фиксация доказательств прогона по `verification-before-completion`.

## Self-Review (выполнено при написании)

- **Покрытие спеки (Фаза 0):** transport-seam (Tasks 1,3,4) ✓; выделение `QueryRunner` (Task 5) ✓; таблица переходов (Task 6) ✓; ctest+qmltest (Tasks 2,7) ✓. Clock-seam явно перенесён в Фазу 1 (отмечено в шапке).
- **Плейсхолдеры:** отсутствуют — весь код приведён целиком.
- **Согласованность типов:** `ISynapsTransport` (методы совпадают с `SynapsTransport`), конструктор `QueryManager(ISynapsTransport*, QObject*)`, `QueryRunner(QueryManager*, ISynapsTransport*, Query*)`, `isLegalQueryPhaseTransition(Query::Phase, Query::Phase)`, цели CMake `synaps_testable`/`tst_*`/`synaps_qmltest` — согласованы между задачами.
