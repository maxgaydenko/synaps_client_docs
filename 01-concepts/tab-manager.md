# Концепция: TabManager

> **Статус:** стаб. Углубляет [architecture-concept.md §5.5](../00-overview/architecture-concept.md#55-tabmanager).

## Scope этого документа (заполнить в фазе концепций/архитектуры)

- Реестр табов и владение состоянием; `TabKind` (home, SQL Query, source preview,
  project, branch, rdb-project, ...).
- Правила: **single-instance** табы; **неудаляемый** домашний таб; активный таб;
  переключение/переименование/дублирование/закрытие.
- Индикатор активности запроса на табе (через подписку на subject).
- Состояние нижнего сайдбара на уровне таба (выбранный bottom-tab).
- **Персистенция** табов между запусками (формат — см. ADR).
- Контракты и модель данных — на фазе архитектуры.

Прообраз в POC: `TabsRepository` / `TabsBloc` / `TabManagerEntity`
(`ddbs_client/lib/src/repositories/tabs_repository.dart`,
`.../bloc/tabs/`, `.../entities/tab_entity.dart`),
UI — `widgets/layout/layout_tabs_widget.dart`.
