# Архитектура (фаза 2)

> Наполняется в фазе архитектуры на основе
> [00-overview/architecture-concept.md](../00-overview/architecture-concept.md).

## Документы

| Документ | Статус |
|---|---|
| [poc-grpc-notes.md](poc-grpc-notes.md) — PoC QtGrpc (flows A/C/D) | ✅ выполнено |
| [query-manager-and-transport.md](query-manager-and-transport.md) — дизайн QueryManager + SynapsTransport + Query + QueryStore | дизайн v1 |

## Что ещё появится

- **TabManager** — детальный дизайн (отдельный документ).
- **AuthTransport / AuthService** под QtGrpc.
- Поэкранные ViewModel'и (начиная с SQL Query как вертикального среза).
- Структура каталогов C++/QML, правила композиции/DI.
- Диаграммы компонентов; решения по открытым вопросам (см. [adr/](../adr/)).

После этой фазы — план работ и выполнение.
