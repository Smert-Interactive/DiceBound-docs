# ERD DiceBound

Документ содержит принципиальную ERD актуальной модели данных MVP DiceBound.

Диаграмма показывает основные сущности, ключи и cardinality.

```mermaid
erDiagram
    USER ||--o{ GAME_SYSTEM : owns
    USER ||--o{ CHARACTER : owns
    USER ||--o{ CHARACTER_SHARE : receives

    GAME_SYSTEM ||--o{ SHEET_SCHEMA : defines
    GAME_SYSTEM ||--o{ CHARACTER : uses
    SHEET_SCHEMA ||--o{ CHARACTER : defines

    CHARACTER ||--o{ CHARACTER_VERSION : has
    CHARACTER ||--o{ CHARACTER_SHARE : shares

    USER {
        uuid id PK
    }

    GAME_SYSTEM {
        uuid id PK
        uuid owner_id FK
        string name
        timestamp created_at
    }

    SHEET_SCHEMA {
        uuid id PK
        uuid game_system_id FK
        string name
        jsonb schema
        timestamp created_at
    }

    CHARACTER {
        uuid id PK
        uuid owner_id FK
        uuid game_system_id FK
        uuid sheet_schema_id FK
        string name
        jsonb data
        timestamp created_at
        timestamp updated_at
    }

    CHARACTER_VERSION {
        uuid id PK
        uuid character_id FK
        integer version_number
        jsonb data
        timestamp created_at
    }

    CHARACTER_SHARE {
        uuid id PK
        uuid character_id FK
        uuid user_id FK
        string permission
        timestamp created_at
```

## Сущности

В ERD используются шесть основных таблиц MVP.

| Таблица | Назначение |
| --- | --- |
| `User` | Пользователь системы. |
| `GameSystem` | Игровая система. |
| `SheetSchema` | Схема листа персонажа. |
| `Character` | Текущие данные персонажа. |
| `CharacterVersion` | Снимок персонажа при сохранении. |
| `CharacterShare` | Доступ пользователя к персонажу. |

## Ключи

PK и основные FK представлены в следующей таблице.

| Таблица | PK | FK |
| --- | --- | --- |
| `User` | `id` | — |
| `GameSystem` | `id` | `owner_id` |
| `SheetSchema` | `id` | `game_system_id` |
| `Character` | `id` | `owner_id`, `game_system_id`, `sheet_schema_id` |
| `CharacterVersion` | `id` | `character_id` |
| `CharacterShare` | `id` | `character_id`, `user_id` |

### Cardinality

Основные связи имеют следующую cardinality.

| Связь | Cardinality |
| --- | --- |
| `User` -> `GameSystem` | 1:N |
| `User` -> `Character` | 1:N |
| `User` -> `CharacterShare` | 1:N |
| `GameSystem` -> `SheetSchema` | 1:N |
| `GameSystem` -> `Character` | 1:N |
| `SheetSchema` -> `Character` | 1:N |
| `Character` -> `CharacterVersion` | 1:N |
| `Character` -> `CharacterShare` | 1:N |

## Character

`Character` связан с владельцем через `owner_id`.

`Character` связан с `GameSystem` через `game_system_id`.

`Character` связан с конкретным `SheetSchema` через
`sheet_schema_id`.

Текущие данные персонажа хранятся в поле `data` типа JSONB.

## CharacterVersion

`CharacterVersion` связан с `Character` через `character_id`.

Каждая версия содержит полный снимок данных персонажа в поле `data`.

Новая версия создаётся при логическом сохранении персонажа.

При восстановлении версии более поздние версии удаляются.

Восстановление не создаёт новую версию.

## CharacterShare

`CharacterShare` связывает персонажа с пользователем, которому
предоставлен доступ.

Поле `permission` определяет уровень доступа.

При VIEW пользователь получает доступ только для чтения.

При EDIT создаётся отдельная копия `Character` для получателя.

Копия имеет собственные данные и историю версий.

## SheetSchema

`SheetSchema` связан с `GameSystem` через `game_system_id`.

Поле `schema` хранит структуру листа персонажа в формате JSONB.

Используемая в MVP схема считается неизменяемой.

При несовместимом изменении структуры создаётся новый `SheetSchema`.

`Character` продолжает ссылаться на выбранный `SheetSchema`.

Полная модель `SystemVersion` и миграция персонажей между схемами
не входят в MVP.

## Ограничения MVP

`SystemVersion` не является частью текущей ERD.

Миграция существующих персонажей между версиями схемы не входит в MVP.

Инвентарь и заклинания не являются частью функциональности MVP.

`Schema` не является отдельной таблицей.

Структура схемы хранится в `SheetSchema.schema`.

`Permission` не является отдельной таблицей.

Уровень доступа хранится в `CharacterShare.permission`.

## Связанные документы

- [`entities.md`](entities.md)
- [`relationships.md`](relationships.md)
