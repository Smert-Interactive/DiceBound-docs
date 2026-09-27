# Entities

Документ описывает основные сущности актуальной модели данных MVP
DiceBound.

## User

### Ответственность User

Представляет пользователя платформы DiceBound.

Пользователь может создавать `GameSystem` и `Character`.
Пользователь также может получать доступ к чужим персонажам через
`CharacterShare`.

### Атрибуты User

- `id`
- `username`
- `email`
- `created_at`
- `updated_at`

### Ключ User

Primary key: `users.id`.

### Связи User

На `users.id` ссылаются:

- `game_systems.owner_id`;
- `characters.owner_id`;
- `character_shares.user_id`.

## GameSystem

### Ответственность GameSystem

Представляет RPG-систему, используемую для создания персонажей.

### Атрибуты GameSystem

- `id`
- `owner_id`
- `name`
- `description`
- `created_at`
- `updated_at`

### Ключ GameSystem

Primary key: `game_systems.id`.

### Внешний ключ GameSystem

`owner_id -> users.id`.

### Связи GameSystem

Один `User` может владеть несколькими `GameSystem`.
Один `GameSystem` может иметь несколько `SheetSchema`.
Один `GameSystem` может использоваться несколькими `Character`.

## Sheet Schema

### Ответственность Sheet Schema

Представляет структуру листа персонажа для конкретного
`GameSystem`.

`SheetSchema` хранит описание структуры листа в поле `schema`.

### Атрибуты Sheet Schema

- `id`
- `game_system_id`
- `name`
- `schema`
- `created_at`

### Ключ Sheet Schema

Primary key: `sheet_schemas.id`.

### Внешний ключ Sheet Schema

`game_system_id -> game_systems.id`.

### Хранение Sheet Schema

Поле `schema` имеет тип PostgreSQL `JSONB`.

### Ограничения Sheet Schema

Используемый `SheetSchema` считается неизменяемым в рамках MVP.
При несовместимом изменении структуры создаётся новый
`SheetSchema`.

## Character

### Ответственность Character

Представляет постоянную сущность персонажа.

`Character` хранит идентичность и основные метаданные персонажа.
Текущее состояние персонажа хранится в поле `data`.

### Атрибуты Character

- `id`
- `owner_id`
- `game_system_id`
- `sheet_schema_id`
- `name`
- `data`
- `created_at`
- `updated_at`

### Ключ Character

Primary key: `characters.id`.

### Внешние ключи Character

- `owner_id -> users.id`;
- `game_system_id -> game_systems.id`;
- `sheet_schema_id -> sheet_schemas.id`.

### Хранение Character

Поле `data` имеет тип PostgreSQL `JSONB`.

### Связи Character

Один `User` может владеть несколькими `Character`.
Один `GameSystem` может использоваться несколькими `Character`.
Один `SheetSchema` может использоваться несколькими `Character`.
Один `Character` может иметь несколько `CharacterVersion`.
Один `Character` может иметь несколько `CharacterShare`.

## CharacterVersion

### Ответственность CharacterVersion

Представляет сохранённый снимок состояния `Character`.

`CharacterVersion` хранит полное состояние персонажа в момент
логического сохранения.

### Атрибуты CharacterVersion

- `id`
- `character_id`
- `version_number`
- `data`
- `created_at`

### Ключ CharacterVersion

Primary key: `character_versions.id`.

### Внешний ключ CharacterVersion

`character_id -> characters.id`.

### Хранение CharacterVersion

Поле `data` имеет тип PostgreSQL `JSONB`.

### Ограничения CharacterVersion

Номер версии уникален в пределах одного персонажа:

`UNIQUE(character_id, version_number)`.

Новая версия создаётся при логическом сохранении персонажа.

### Восстановление CharacterVersion

При восстановлении выбранной версии более поздние версии удаляются.
Новая `CharacterVersion` при восстановлении не создаётся.

## CharacterShare

### Ответственность CharacterShare

Представляет предоставление доступа пользователя к `Character`.

`CharacterShare` используется для связи `Character` с другим
`User`.

### Атрибуты CharacterShare

- `id`
- `character_id`
- `user_id`
- `permission`
- `created_at`

### Ключ CharacterShare

Primary key: `character_shares.id`.

### Внешние ключи CharacterShare

- `character_id -> characters.id`;
- `user_id -> users.id`.

### Ограничения CharacterShare

Для одного пользователя действует одна запись доступа к персонажу:

`UNIQUE(character_id, user_id)`.

### Типы доступа CharacterShare

`VIEW` предоставляет доступ только к просмотру исходного
`Character`.

`EDIT` не предоставляет право изменять исходный `Character`.
При `EDIT` создаётся отдельная копия `Character` для получателя.

## Общая модель

Актуальная MVP-модель содержит следующие таблицы:

- `User`;
- `GameSystem`;
- `SheetSchema`;
- `Character`;
- `CharacterVersion`;
- `CharacterShare`.

`SystemVersion` не входит в модель MVP.
`Schema` является полем `SheetSchema.schema`, а не отдельной таблицей.
`Permission` является полем `CharacterShare.permission`, а не
отдельной таблицей.

## Связанные документы

- [Связи модели данных](relationships.md)
- [Стратегия хранения данных](storage-strategy.md)
- [Версионирование персонажей](character-versioning.md)
