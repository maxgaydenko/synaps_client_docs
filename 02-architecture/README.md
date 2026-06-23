# Архитектура (фаза 2)

> Наполняется в фазе архитектуры на основе
> [00-overview/architecture-concept.md](../00-overview/architecture-concept.md).

## Документы

| Документ | Статус |
|---|---|
| [poc-grpc-notes.md](poc-grpc-notes.md) — PoC QtGrpc (flows A/C/D) | ✅ выполнено |
| [query-manager-and-transport.md](query-manager-and-transport.md) — дизайн QueryManager + SynapsTransport + Query + QueryStore | дизайн v1 ✅ реализован (A/C/D + подписка) |
| **AuthTransport + AuthService** под QtGrpc (`Auth.Login`/`Auth.User`) | ✅ реализован (без отдельного дизайн-документа) |
| **AppConfig** — JSON-конфиг connection-параметров (`endpoint`/`deadlineMs` на сервис) | ✅ реализован; приоритет `defaults < файл < env`, путь `--config`/`./synaps.config.json`/`AppConfigLocation` |

## Что ещё появится

- **TabManager** — детальный дизайн (отдельный документ).
- Поэкранные ViewModel'и (начиная с SQL Query как вертикального среза).
- Структура каталогов C++/QML, правила композиции/DI.
- Диаграммы компонентов; решения по открытым вопросам (см. [adr/](../adr/)).

После этой фазы — план работ и выполнение.
