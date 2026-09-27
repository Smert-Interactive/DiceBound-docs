# Связи между сущностями DiceBound

Этот документ описывает связи между сущностями модели данных DiceBound.
Он фиксирует кардинальность связей и используемые внешние ключи.
Состав и поля сущностей определены в [`entities.md`](entities.md).

## Основные связи

| Родительская сущность | Связь | Дочерняя сущность | Внешний ключ |
| --- | --- | --- | --- |
| `User` | 1:N | `GameSystem` | `game_systems.owner_id` |
| `User` | 1:N | `Character` | `characters.owner_id` |
| `GameSystem` | 1:N | `Sheet Schema` | `sheet_schemas.game_system_id` |
| `GameSystem` | 1:N | `Character` | `characters.game_system_id` |
| `Sheet Schema` | 1:N | `Character` | `characters.sheet_schema_id` |
| `Character` | 1:N | `CharacterVersion` | `character_versions.character_id` |
| `Character` | 1:N | `CharacterShare` | `character_shares.character_id` |
| `User` | 1:N | `CharacterShare` | `character_shares.user_id` |

## User и GameSystem

Один `User` может владеть несколькими `GameSystem`.
Каждая `GameSystem` имеет одного владельца.
Связь реализуется через `game_systems.owner_id`.

## User и Character

Один `User` может владеть несколькими `Character`.
Каждый `Character` имеет одного владельца.
Связь реализуется через `characters.owner_id`.

## GameSystem и Sheet Schema

Одна `GameSystem` может иметь несколько `Sheet Schema`.
Каждая `Sheet Schema` относится только к одной `GameSystem`.
Связь реализуется через `sheet_schemas.game_system_id`.

`Sheet Schema` определяет структуру листа персонажа для связанной `GameSystem`.

## GameSystem и Character

Одна `GameSystem` может использоваться несколькими `Character`.
Каждый `Character` относится к одной `GameSystem`.
Связь реализуется через `characters.game_system_id`.

## Sheet Schema и Character

Одна `Sheet Schema` может использоваться несколькими `Character`.
Каждый `Character` использует одну `Sheet Schema`.
Связь реализуется через `characters.sheet_schema_id`.

Таким образом, структура персонажа определяется цепочкой `GameSystem` → `Sheet Schema` → `Character`.

## Character и CharacterVersion

Один `Character` может иметь несколько `CharacterVersion`.
Каждая `CharacterVersion` относится к одному `Character`.
Связь реализуется через `character_versions.character_id`.

`CharacterVersion` хранит снимок состояния персонажа на момент логического сохранения.
Правила создания и восстановления версий описаны в [`character-versioning.md`](character-versioning.md).

## Character и CharacterShare

Один `Character` может иметь несколько записей `CharacterShare`.
Каждая запись `CharacterShare` относится к одному исходному `Character`.
Связь реализуется через `character_shares.character_id`.

Для одной пары `character_id` и `user_id` допускается только одна запись.
Это обеспечивается ограничением `UNIQUE(character_id, user_id)`.

## User и CharacterShare

Один `User` может иметь доступ к нескольким `Character` через `CharacterShare`.
Каждая запись `CharacterShare` относится к одному пользователю.
Связь реализуется через `character_shares.user_id`.

Режим `VIEW` предоставляет доступ только для чтения.
Операция `EDIT` создаёт отдельную копию персонажа и не предоставляет право записи в исходный `Character`.

## Ограничения связей

В MVP отсутствуют связи с отдельной сущностью версии игровой системы.
`CharacterVersion` не связан напрямую с `User`.
Автор сохранения не хранится в `CharacterVersion`.

Связь между `Character` и `Sheet Schema` является обязательной для определения структуры данных персонажа.
При создании новой несовместимой структуры используется новая `Sheet Schema`, а не изменение существующей схемы.
