# Концепция: разделение слоёв и MVVM

> **Статус:** стаб. Углубляет [architecture-concept.md §5.6](../00-overview/architecture-concept.md#56-разделение-слоёв-mvvm).

## Scope этого документа (заполнить в фазе концепций/архитектуры)

- Слои: **Presentation (QML) → ViewModel (C++ QObject) → Application/Domain →
  Data/Transport**. Ответственность и границы каждого.
- Жёсткие правила: нет бизнес-логики в QML; нет представления в слое данных.
- ViewModel как граница: маппинг доменных событий в `Q_PROPERTY`/сигналы; ввод —
  в доменные команды.
- Регистрация C++-типов в QML (`QML_ELEMENT`/`QML_SINGLETON`/`Q_PROPERTY`/
  `Q_INVOKABLE`/`Q_ENUM`) — по образцу текущего `AuthService`.
- Стратегия DI/композиции, владение временем жизни объектов.
- Антипаттерны из POC, которых избегаем (смешение слоёв).

Эталон в каркасе: `synaps_client/src/AuthService.{h,cpp}`,
`synaps_client/src/qml/`.
