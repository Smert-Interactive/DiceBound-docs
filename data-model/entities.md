# Entities

## User

### User: Ответственность

Представляет пользователя платформы DiceBound.

Пользователь может:

- создавать RPG-системы;
- создавать персонажей;
- создавать версии RPG-систем;
- создавать версии персонажей;
- получать доступ к персонажам других пользователей через `Share`.

### User: Ключевые атрибуты

- `id`
- `username`
- `email`
- `created_at`
- `updated_at`

### User: Primary key

`users.id`

### User: Предполагаемые foreign keys

На `users.id` ссылаются:

- `game_systems.owner_id`
- `characters.owner_id`
- `system_versions.created_by_user_id`
- `character_versions.created_by_user_id`
- `shares.user_id`

---

## GameSystem

### GameSystem: Ответственность

Представляет RPG-систему, используемую для создания персонажей.

Примеры:

- D&D;
- Pathfinder;
- Call of Cthulhu;
- Cyberpunk RED;
- пользовательская RPG-система.

RPG-система не должна представляться отдельным набором таблиц
для каждой игры. Структура листа персонажа хранится через версии
системы и поле `schema` типа JSONB.

### GameSystem: Ключевые атрибуты

- `id`
- `owner_id`
- `name`
- `description`
- `created_at`
- `updated_at`

### GameSystem: Primary key

`game_systems.id`

### GameSystem: Foreign keys

- `owner_id -> users.id`

### GameSystem: Связи

- один `User` может владеть несколькими `GameSystem`;
- один `GameSystem` может иметь несколько `SystemVersion`;
- один `GameSystem` может использоваться несколькими `Character`.

---

## SystemVersion

### SystemVersion: Ответственность

Представляет конкретную версию RPG-системы.

Версионирование необходимо для того, чтобы изменения структуры
RPG-системы не делали старые версии персонажей некорректными.

Например, если структура листа персонажа была изменена с версии
`1` на версию `2`, старая версия системы продолжает существовать
и может использоваться старыми версиями персонажей.

### SystemVersion: Ключевые атрибуты

- `id`
- `game_system_id`
- `version_number`
- `schema`
- `created_by_user_id`
- `created_at`

### SystemVersion: Primary key

`system_versions.id`

### SystemVersion: Foreign keys

- `game_system_id -> game_systems.id`
- `created_by_user_id -> users.id`

### SystemVersion: Ограничения

Для одной RPG-системы номер версии должен быть уникальным:

`UNIQUE(game_system_id, version_number)`

### SystemVersion: Связи

- один `GameSystem` может иметь много `SystemVersion`;
- одна `SystemVersion` относится только к одному `GameSystem`;
- одна `SystemVersion` может использоваться несколькими
  `CharacterVersion`.

---

## Schema

### Schema: Ответственность

`Schema` описывает структуру листа персонажа для конкретной
версии RPG-системы.

`Schema` не является отдельной таблицей базы данных. Она хранится
непосредственно в поле `system_versions.schema`.

### Schema: Тип хранения

PostgreSQL `JSONB`.

### Schema: Пример

```json
{
  "fields": {
    "health": {
      "type": "integer",
      "label": "Health"
    },
    "strength": {
      "type": "integer",
      "label": "Strength"
    }
  }
}
```

### Schema: Назначение

Через `Schema` можно описывать разные структуры персонажей
без создания новых таблиц для каждой RPG-системы.

Например, одна система может содержать:

- `health`;
- `mana`;
- `strength`;

а другая:

- `hit_points`;
- `sanity`;
- `skill_points`.

При этом структура PostgreSQL остаётся общей.

### Schema: Связь с версиями персонажа

`CharacterVersion.system_version_id` определяет, какая `Schema`
должна использоваться для интерпретации поля
`CharacterVersion.data`.

---

## Character

### Character: Ответственность

Представляет постоянную сущность персонажа.

`Character` хранит идентичность и основные метаданные персонажа,
а изменяемое состояние персонажа хранится в `CharacterVersion`.

Изменение состояния персонажа не должно перезаписывать
предыдущую версию.

### Character: Ключевые атрибуты

- `id`
- `owner_id`
- `game_system_id`
- `name`
- `created_at`
- `updated_at`

### Character: Primary key

`characters.id`

### Character: Foreign keys

- `owner_id -> users.id`
- `game_system_id -> game_systems.id`

### Character: Связи

- один `User` может владеть несколькими `Character`;
- один `GameSystem` может использоваться несколькими `Character`;
- один `Character` может иметь несколько `CharacterVersion`;
- один `Character` может иметь несколько `Share`.

### Character: Примечание по `game_system_id`

`game_system_id` является связью персонажа с RPG-системой.

При использовании `CharacterVersion.system_version_id` необходимо
сохранять согласованность между RPG-системой персонажа и RPG-системой
соответствующей версии персонажа.

---

## CharacterVersion

### CharacterVersion: Ответственность

Представляет сохранённое состояние персонажа в определённый
момент времени.

Каждое изменение состояния персонажа создаёт новую
`CharacterVersion`, поэтому предыдущие состояния сохраняются
как история.

### CharacterVersion: Ключевые атрибуты

- `id`
- `character_id`
- `system_version_id`
- `version_number`
- `data`
- `created_by_user_id`
- `created_at`

### CharacterVersion: Primary key

`character_versions.id`

### CharacterVersion: Foreign keys

- `character_id -> characters.id`
- `system_version_id -> system_versions.id`
- `created_by_user_id -> users.id`

### CharacterVersion: Тип хранения `data`

PostgreSQL `JSONB`.

### CharacterVersion: Назначение `data`

Поле `data` содержит фактические значения полей персонажа.

Например:

```json
{
  "health": 25,
  "strength": 14,
  "mana": 10
}
```

Структура `data` определяется связанной `SystemVersion.schema`.

Таким образом:

- `SystemVersion.schema` отвечает за то, какие поля существуют;
- `CharacterVersion.data` отвечает за значения этих полей.

### CharacterVersion: Ограничения

Номер версии должен быть уникальным в пределах одного персонажа:

`UNIQUE(character_id, version_number)`

### CharacterVersion: Связи

- один `Character` может иметь много `CharacterVersion`;
- одна `CharacterVersion` относится к одному `Character`;
- одна `CharacterVersion` использует одну `SystemVersion`.

---

## Share

### Share: Ответственность

Представляет разрешение пользователя на доступ к персонажу
другого пользователя.

`Share` используется для реализации связи многие-ко-многим
между `Character` и `User`.

Владелец персонажа хранится в `characters.owner_id`.
Запись `Share` используется для предоставления доступа
дополнительным пользователям.

### Share: Ключевые атрибуты

- `id`
- `character_id`
- `user_id`
- `permission`
- `created_at`

### Share: Primary key

`shares.id`

### Share: Foreign keys

- `character_id -> characters.id`
- `user_id -> users.id`

### Share: Ограничения

Один пользователь не должен получать несколько записей доступа
к одному и тому же персонажу:

`UNIQUE(character_id, user_id)`

### Share: Связи

- один `Character` может иметь много записей `Share`;
- один `User` может иметь много записей `Share`;
- через `Share` реализуется связь `Character N:M User`.

---

## Permission

### Permission: Ответственность

Определяет уровень доступа пользователя к персонажу.

`Permission` не является отдельной таблицей.

Значение хранится в поле `shares.permission`.

### Permission: Начальные значения

- `view` — пользователь может просматривать персонажа;
- `edit` — пользователь может изменять персонажа, если это
  разрешено логикой приложения.

### Permission: Хранение

На первом этапе значение может храниться как строковое поле.

При необходимости на уровне PostgreSQL можно добавить ограничение:

```sql
CHECK (permission IN ('view', 'edit'))
```

---

## Связи между сущностями

Основные связи модели:

`User` 1:N `GameSystem` — владеет RPG-системами
`User` 1:N `Character` — владеет персонажами
`GameSystem` 1:N `SystemVersion` — имеет версии
`Character` 1:N `CharacterVersion` — имеет историю
`SystemVersion` 1:N `CharacterVersion` — используется версиями
`User` 1:N `Share` — имеет разрешения
`Character` 1:N `Share` — предоставляет доступ
`Character` N:M `User` — через `Share`
---

## Relational and JSONB storage

Модель использует комбинацию обычных реляционных полей
PostgreSQL и `JSONB`.

### Реляционно хранятся

- идентификаторы;
- внешние ключи;
- владельцы;
- связи между сущностями;
- номера версий;
- имена;
- даты создания и изменения;
- права доступа.

### В JSONB хранятся

`system_versions.schema`:

- структура листа персонажа;
- набор полей;
- типы полей;
- настройки отображения.

`character_versions.data`:

- фактические значения полей персонажа;
- RPG-специфичные данные.

Такой подход позволяет поддерживать разные RPG-системы без
изменения структуры PostgreSQL при добавлении новых типов
персонажей.

---

## Versioning

### Версионирование RPG-систем

Связь:

`GameSystem 1:N SystemVersion`

Каждая версия системы содержит собственную `schema`.

Старая версия системы не должна изменяться при создании новой
версии. Это позволяет правильно интерпретировать старые версии
персонажей.

Пример:

```text
D&D
├── SystemVersion 1
│   └── Schema 1
│
└── SystemVersion 2
    └── Schema 2
```

### Версионирование персонажей

Связь:

`Character 1:N CharacterVersion`

Каждое сохранение изменённого состояния создаёт новую версию:

```text
Character
├── CharacterVersion 1
├── CharacterVersion 2
└── CharacterVersion 3
```

Предыдущие версии не удаляются и могут использоваться
для просмотра истории.

Каждая `CharacterVersion` дополнительно содержит
`system_version_id`. Благодаря этому известно, какая версия
RPG-системы и какая `Schema` использовались для интерпретации
сохранённых данных.

---

## Summary

Основная модель данных DiceBound состоит из следующих сущностей:

- `User` — пользователь;
- `GameSystem` — RPG-система;
- `SystemVersion` — версия RPG-системы;
- `Character` — персонаж;
- `CharacterVersion` — версия состояния персонажа;
- `Share` — предоставление доступа;
- `Schema` — структура листа персонажа в `SystemVersion.schema`;
- `Permission` — уровень доступа в `Share.permission`.

Реляционная часть модели отвечает за связи и целостность данных,
а `JSONB` используется для гибкой структуры RPG-систем и данных
персонажей.
