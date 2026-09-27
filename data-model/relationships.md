# Relationships

Документ описывает связи между основными сущностями актуальной
модели данных MVP DiceBound.

## User и GameSystem

Один `User` может владеть несколькими `GameSystem`.

Связь имеет cardinality `1:N`.

Внешний ключ:

`game_systems.owner_id -> users.id`.

## User и Character

Один `User` может владеть несколькими `Character`.

Связь имеет cardinality `1:N`.

Внешний ключ:

`characters.owner_id -> users.id`.

## User и CharacterShare

Один `User` может иметь несколько записей `CharacterShare`.

Связь имеет cardinality `1:N`.

Внешний ключ:

`character_shares.user_id -> users.id`.

## GameSystem и SheetSchema

Один `GameSystem` может иметь несколько `SheetSchema`.

Каждый `SheetSchema` относится только к одному `GameSystem`.

Связь имеет cardinality `1:N`.

Внешний ключ:

`sheet_schemas.game_system_id -> game_systems.id`.

## GameSystem и Character

Один `GameSystem` может использоваться несколькими `Character`.

Каждый `Character` относится к одному `GameSystem`.

Связь имеет cardinality `1:N`.

Внешний ключ:

`characters.game_system_id -> game_systems.id`.

## SheetSchema и Character

Один `SheetSchema` может использоваться несколькими `Character`.

Каждый `Character` ссылается на один конкретный `SheetSchema`.

Связь имеет cardinality `1:N`.

Внешний ключ:

`characters.sheet_schema_id -> sheet_schemas.id`.

Таким образом, актуальная MVP-цепочка имеет вид:

```text
GameSystem
    |
    +---- SheetSchema
              |
              +---- Character
```

`Character` также напрямую связан с `GameSystem`.

## Character и CharacterVersion

Один `Character` может иметь несколько `CharacterVersion`.

Каждый `CharacterVersion` относится только к одному `Character`.

Связь имеет cardinality `1:N`.

Внешний ключ:

`character_versions.character_id -> characters.id`.

`CharacterVersion` хранит полный снимок данных персонажа.

## Character и CharacterShare

Один `Character` может иметь несколько `CharacterShare`.

Каждый `CharacterShare` относится только к одному `Character`.

Связь имеет cardinality `1:N`.

Внешний ключ:

`character_shares.character_id -> characters.id`.

`CharacterShare` также связан с `User` через `user_id`.

## Character и User через CharacterShare

`CharacterShare` реализует связь между `Character` и `User`.

Один `User` может иметь доступ к нескольким `Character`.
Один `Character` может быть доступен нескольким `User`.

Логически это связь `N:M`, реализованная через таблицу
`CharacterShare`.

Для пары `character_id` и `user_id` действует ограничение:

`UNIQUE(character_id, user_id)`.

## Сводка связей

| Первая сущность | Cardinality | Вторая сущность |
| --- | --- | --- |
| `User` | `1:N` | `GameSystem` |
| `User` | `1:N` | `Character` |
| `User` | `1:N` | `CharacterShare` |
| `GameSystem` | `1:N` | `SheetSchema` |
| `GameSystem` | `1:N` | `Character` |
| `SheetSchema` | `1:N` | `Character` |
| `Character` | `1:N` | `CharacterVersion` |
| `Character` | `1:N` | `CharacterShare` |
| `User` | `N:M` | `Character` через `CharacterShare` |

## Ограничения MVP

`SystemVersion` не входит в актуальную модель MVP.

Связь `GameSystem -> SystemVersion` отсутствует.

`CharacterVersion` не содержит `system_version_id`.

`CharacterVersion` не содержит `created_by_user_id`.

Миграция существующих персонажей между структурами
`SheetSchema` не входит в MVP.

## Связанные документы

- [Сущности модели данных](entities.md)
- [Стратегия хранения данных](storage-strategy.md)
- [Версионирование персонажей](character-versioning.md)
