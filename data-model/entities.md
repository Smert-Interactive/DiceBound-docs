# Сущности модели данных DiceBound

## Назначение

Этот документ описывает основные сущности DiceBound, их ответственность,
ключевые атрибуты, primary key и предполагаемые foreign keys.

Документ фиксирует универсальное ядро модели данных. Он не вводит отдельные
таблицы под конкретные RPG-системы, например `skills`, `spells`, `items`,
`races` или `classes`.

## Список основных сущностей

В ядро модели входят:

- `User`
- `GameSystem`
- `SystemVersion`
- `Schema`
- `Character`
- `CharacterVersion`
- `Share`
- `Permission`

`Schema` и `Permission` в этой версии модели не являются отдельными таблицами.
`Schema` хранится как `SystemVersion.schema`. `Permission` хранится как поле
`Share.permission`.

## User

### Ответственность

`User` представляет пользователя DiceBound.

Пользователь может:

- владеть RPG-системами;
- владеть персонажами;
- создавать версии систем, если это отслеживается;
- создавать версии персонажей, если это отслеживается;
- получать доступ к чужим персонажам через `Share`.

`User` является реляционной сущностью PostgreSQL и не хранится в `JSONB`.

### Ключевые атрибуты

- `id`
- `username`
- `email`
- `created_at`
- `updated_at`

### Primary key

- `users.id`

### Предполагаемые foreign keys

У `User` нет обязательных foreign keys на другие основные сущности.

На `User` ссылаются:

- `game_systems.owner_id`
- `characters.owner_id`
- `system_versions.created_by_user_id`
- `character_versions.created_by_user_id`
- `shares.user_id`

## GameSystem

### Ответственность

`GameSystem` представляет RPG-систему как данные.

Примеры RPG-систем:

- Dungeons & Dragons
- Pathfinder
- Call of Cthulhu
- Cyberpunk RED
- пользовательская RPG-система

`GameSystem` не должна превращаться в набор отдельных таблиц под конкретную
игру. Конкретная структура листа персонажа хранится в версиях системы.

### Ключевые атрибуты

- `id`
- `owner_id`
- `name`
- `description`
- `created_at`
- `updated_at`

### Primary key

- `game_systems.id`

### Предполагаемые foreign keys

- `game_systems.owner_id -> users.id`

На `GameSystem` ссылаются:

- `system_versions.game_system_id`
- `characters.game_system_id`

## SystemVersion

### Ответственность

`SystemVersion` представляет одну версию RPG-системы.

Она нужна, чтобы изменения структуры RPG-системы не ломали старые версии
персонажей. Каждая версия системы имеет собственную `schema`.

### Ключевые атрибуты

- `id`
- `game_system_id`
- `version_number`
- `schema`
- `created_by_user_id`
- `created_at`

### Primary key

- `system_versions.id`

### Предполагаемые foreign keys

- `system_versions.game_system_id -> game_systems.id`
- `system_versions.created_by_user_id -> users.id`

На `SystemVersion` ссылаются:

- `character_versions.system_version_id`

### Ограничения

Номер версии должен быть уникален внутри одной RPG-системы:

```text
UNIQUE (game_system_id, version_number)
```

## Schema

### Ответственность

`Schema` описывает структуру листа персонажа для конкретной версии
RPG-системы.

В текущей версии модели `Schema` не является отдельной таблицей. Она хранится в
поле:

```text
system_versions.schema
```

Тип хранения:

```text
JSONB
```

### Ключевые атрибуты

Состав `schema` зависит от RPG-системы. Концептуально она может описывать:

- поля листа персонажа;
- типы полей;
- названия полей;
- правила группировки;
- дополнительные настройки отображения или валидации.

Пример:

```json
{
  "fields": {
    "health": {
      "type": "integer",
      "label": "Health"
    }
  }
}
```

### Primary key

У `Schema` нет собственного primary key, потому что это не отдельная таблица.

Схема идентифицируется через:

```text
system_versions.id
```

### Предполагаемые foreign keys

У `Schema` нет собственных foreign keys.

Историческая связь с данными персонажа обеспечивается через:

```text
character_versions.system_version_id -> system_versions.id
```

## Character

### Ответственность

`Character` представляет стабильную сущность персонажа.

`Character` хранит метаданные персонажа, но не хранит все изменяемое состояние
листа. Состояния персонажа хранятся в `CharacterVersion`.

### Ключевые атрибуты

- `id`
- `owner_id`
- `game_system_id`
- `name`
- `created_at`
- `updated_at`

### Primary key

- `characters.id`

### Предполагаемые foreign keys

- `characters.owner_id -> users.id`
- `characters.game_system_id -> game_systems.id`

На `Character` ссылаются:

- `character_versions.character_id`
- `shares.character_id`

## CharacterVersion

### Ответственность

`CharacterVersion` представляет сохраненное состояние персонажа.

Каждая версия является историческим снимком. При изменении персонажа должна
создаваться новая `CharacterVersion`, а не перезаписываться старая.

`CharacterVersion` обязательно ссылается на `SystemVersion`, чтобы было понятно,
по какой `Schema` нужно интерпретировать `data`.

### Ключевые атрибуты

- `id`
- `character_id`
- `system_version_id`
- `version_number`
- `data`
- `created_by_user_id`
- `created_at`

### Primary key

- `character_versions.id`

### Предполагаемые foreign keys

- `character_versions.character_id -> characters.id`
- `character_versions.system_version_id -> system_versions.id`
- `character_versions.created_by_user_id -> users.id`

### Ограничения

Номер версии должен быть уникален внутри одного персонажа:

```text
UNIQUE (character_id, version_number)
```

### Данные персонажа

`CharacterVersion.data` хранится как `JSONB`.

`data` содержит значения конкретного состояния персонажа. Структура этих
значений определяется связанной `SystemVersion.schema`.

## Share

### Ответственность

`Share` предоставляет пользователю доступ к персонажу.

Эта сущность реализует связь многие-ко-многим между `Character` и `User`.
Владелец персонажа хранится отдельно в `characters.owner_id`; `Share` нужен для
дополнительных пользователей, которым выдан доступ.

### Ключевые атрибуты

- `id`
- `character_id`
- `user_id`
- `permission`
- `created_at`

### Primary key

- `shares.id`

### Предполагаемые foreign keys

- `shares.character_id -> characters.id`
- `shares.user_id -> users.id`

### Ограничения

Один пользователь не должен получать несколько записей доступа к одному и тому
же персонажу:

```text
UNIQUE (character_id, user_id)
```

## Permission

### Ответственность

`Permission` описывает уровень доступа пользователя к персонажу через `Share`.

В текущей версии модели `Permission` не является отдельной таблицей. Оно
хранится в поле:

```text
shares.permission
```

### Ключевые значения

Стартовые значения:

- `view`
- `edit`

`view` дает право просмотра персонажа.

`edit` дает право изменения персонажа, если это разрешено прикладной логикой.

### Primary key

У `Permission` нет собственного primary key, потому что это не отдельная
таблица.

### Предполагаемые foreign keys

У `Permission` нет собственных foreign keys.

Связь с пользователем и персонажем обеспечивается через `Share`:

```text
shares.character_id -> characters.id
shares.user_id -> users.id
```

### Ограничения

Если значения permissions фиксируются на уровне базы, можно использовать:

```text
CHECK (permission IN ('view', 'edit'))
```

## Согласованность терминов

В документации используются следующие бизнес-термины:

- `User` - пользователь DiceBound.
- `GameSystem` - RPG-система.
- `SystemVersion` - версия RPG-системы.
- `Schema` - структура листа персонажа для версии RPG-системы.
- `Character` - стабильная сущность персонажа.
- `CharacterVersion` - историческое состояние персонажа.
- `Share` - запись выдачи доступа к персонажу.
- `Permission` - уровень доступа в рамках `Share`.

Эти термины согласованы с моделью `relationships.md` и
`storage-strategy.md`.
