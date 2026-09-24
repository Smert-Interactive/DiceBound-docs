# Связи модели данных DiceBound

## Назначение

Этот документ описывает связи между основными сущностями модели данных
DiceBound, их cardinality, обязательность, правила удаления и правила
сохранения истории.

Документ относится к универсальному ядру DiceBound и не вводит специфичные для
RPG таблицы вроде `skills`, `spells`, `items`, `races` или `classes`.

## Основные сущности

В документе используются следующие сущности:

- `User`
- `GameSystem`
- `SystemVersion`
- `Character`
- `CharacterVersion`
- `Share`

## Принципы связей

- Владение, связи, права доступа и версии хранятся реляционно.
- Внешние ключи не хранятся внутри `JSONB`.
- `SystemVersion.schema` описывает структуру листа персонажа.
- `CharacterVersion.data` хранит значения конкретного состояния персонажа.
- Каждая `CharacterVersion` должна ссылаться на `SystemVersion`, чтобы старые
  версии персонажей можно было интерпретировать по правильной схеме.
- Исторические записи `SystemVersion` и `CharacterVersion` не должны молча
  переписываться или удаляться обычным пользовательским изменением.

## Сводка связей

- `User -> GameSystem`
  - Cardinality: `1:N`
  - Обязательность: обязательная для `GameSystem`
  - FK: `game_systems.owner_id -> users.id`
  - Назначение: владелец RPG-системы
- `User -> Character`
  - Cardinality: `1:N`
  - Обязательность: обязательная для `Character`
  - FK: `characters.owner_id -> users.id`
  - Назначение: владелец персонажа
- `GameSystem -> SystemVersion`
  - Cardinality: `1:N`
  - Обязательность: обязательная для `SystemVersion`
  - FK: `system_versions.game_system_id -> game_systems.id`
  - Назначение: версии RPG-системы
- `GameSystem -> Character`
  - Cardinality: `1:N`
  - Обязательность: обязательная для `Character`
  - FK: `characters.game_system_id -> game_systems.id`
  - Назначение: RPG-система персонажа
- `Character -> CharacterVersion`
  - Cardinality: `1:N`
  - Обязательность: обязательная для `CharacterVersion`
  - FK: `character_versions.character_id -> characters.id`
  - Назначение: история состояний персонажа
- `SystemVersion -> CharacterVersion`
  - Cardinality: `1:N`
  - Обязательность: обязательная для `CharacterVersion`
  - FK: `character_versions.system_version_id -> system_versions.id`
  - Назначение: схема для интерпретации состояния персонажа
- `User -> CharacterVersion`
  - Cardinality: `1:N`
  - Обязательность: необязательная для `CharacterVersion`
  - FK: `character_versions.created_by_user_id -> users.id`
  - Назначение: автор изменения, если отслеживается
- `Character -> Share`
  - Cardinality: `1:N`
  - Обязательность: обязательная для `Share`
  - FK: `shares.character_id -> characters.id`
  - Назначение: доступы к персонажу
- `User -> Share`
  - Cardinality: `1:N`
  - Обязательность: обязательная для `Share`
  - FK: `shares.user_id -> users.id`
  - Назначение: пользователь, получивший доступ
- `Character -> User` через `Share`
  - Cardinality: `N:M`
  - Обязательность: необязательная для `Character`
  - FK: `shares.character_id`, `shares.user_id`
  - Назначение: совместный доступ к персонажу

## Обязательные и необязательные связи

### Обязательные связи

Эти связи должны существовать для каждой дочерней записи:

- `GameSystem` должен иметь владельца: `game_systems.owner_id`.
- `Character` должен иметь владельца: `characters.owner_id`.
- `Character` должен быть связан с RPG-системой: `characters.game_system_id`.
- `SystemVersion` должен быть связан с RPG-системой:
  `system_versions.game_system_id`.
- `CharacterVersion` должен быть связан с персонажем:
  `character_versions.character_id`.
- `CharacterVersion` должен быть связан с версией RPG-системы:
  `character_versions.system_version_id`.
- `Share` должен быть связан с персонажем: `shares.character_id`.
- `Share` должен быть связан с пользователем-получателем: `shares.user_id`.

### Необязательные связи

Эти связи могут отсутствовать:

- `character_versions.created_by_user_id` может быть `NULL`, если система не
  отслеживает автора конкретного изменения.
- `system_versions.created_by_user_id` может быть `NULL`, если автор версии
  системы не отслеживается отдельно от владельца `GameSystem`.
- У `Character` может не быть ни одной записи `Share`. Это означает, что доступ
  есть только у владельца.
- У новой `GameSystem` теоретически может временно не быть `SystemVersion`, если
  система создана как черновик. Для полноценного создания персонажей нужна как
  минимум одна `SystemVersion`.
- У нового `Character` теоретически может временно не быть `CharacterVersion`,
  если персонаж создан как черновик. Для сохраненного состояния нужна как
  минимум одна `CharacterVersion`.

## Подробное описание связей

### User 1:N GameSystem

Один пользователь может владеть несколькими RPG-системами.

```text
users.id
  -> game_systems.owner_id
```

Cardinality:

- один `User` может иметь `0..N` записей `GameSystem`;
- каждая `GameSystem` должна иметь ровно одного владельца.

Правило удаления:

- удаление `User`, у которого есть `GameSystem`, должно быть ограничено или
  заменено soft delete;
- автоматическое каскадное удаление `GameSystem` нежелательно, потому что за
  ней могут находиться `SystemVersion`, `Character` и исторические данные.

### User 1:N Character

Один пользователь может владеть несколькими персонажами.

```text
users.id
  -> characters.owner_id
```

Cardinality:

- один `User` может иметь `0..N` записей `Character`;
- каждый `Character` должен иметь ровно одного владельца.

Правило удаления:

- удаление владельца персонажа должно быть ограничено или заменено soft delete;
- персонажи и их версии не должны теряться из-за случайного удаления
  пользователя.

### GameSystem 1:N SystemVersion

Одна RPG-система может иметь несколько версий.

```text
game_systems.id
  -> system_versions.game_system_id
```

Cardinality:

- одна `GameSystem` может иметь `0..N` записей `SystemVersion`;
- каждая `SystemVersion` должна относиться ровно к одной `GameSystem`.

Ограничения:

```text
UNIQUE (game_system_id, version_number)
```

Правило удаления:

- удаление `GameSystem`, у которой есть `SystemVersion`, должно быть ограничено;
- если `SystemVersion` уже используется в `CharacterVersion`, удаление
  недопустимо, иначе исторические версии персонажей потеряют схему
  интерпретации;
- для скрытия устаревших систем предпочтительнее использовать архивирование или
  soft delete.

Правило сохранения истории:

- изменение структуры RPG-системы должно создавать новую `SystemVersion`;
- уже используемая `SystemVersion.schema` не должна молча переписываться.

### GameSystem 1:N Character

Один персонаж создается в рамках одной RPG-системы.

```text
game_systems.id
  -> characters.game_system_id
```

Cardinality:

- одна `GameSystem` может иметь `0..N` персонажей;
- каждый `Character` должен быть связан ровно с одной `GameSystem`.

Правило удаления:

- удаление `GameSystem`, к которой привязаны персонажи, должно быть ограничено;
- персонаж не должен оставаться без RPG-системы.

Примечание:

- конкретная версия схемы для отдельного состояния персонажа определяется не
  через `characters.game_system_id`, а через
  `character_versions.system_version_id`.

### Character 1:N CharacterVersion

`Character` является стабильной сущностью персонажа, а `CharacterVersion`
хранит конкретные исторические состояния.

```text
characters.id
  -> character_versions.character_id
```

Cardinality:

- один `Character` может иметь `0..N` записей `CharacterVersion`;
- каждая `CharacterVersion` должна относиться ровно к одному `Character`.

Ограничения:

```text
UNIQUE (character_id, version_number)
```

Правило удаления:

- удаление `Character`, у которого есть версии, должно быть ограничено или
  заменено soft delete;
- каскадное удаление `CharacterVersion` нежелательно, потому что это уничтожает
  историю персонажа.

Правило сохранения истории:

- изменение персонажа создает новую `CharacterVersion`;
- старые `CharacterVersion` не должны перезаписываться обычным обновлением;
- старые версии могут использоваться для истории, сравнения, отката и аудита.

### SystemVersion 1:N CharacterVersion

Каждая версия персонажа должна знать, по какой версии RPG-системы нужно
интерпретировать ее `data JSONB`.

```text
system_versions.id
  -> character_versions.system_version_id
```

Cardinality:

- одна `SystemVersion` может использоваться в `0..N` записях
  `CharacterVersion`;
- каждая `CharacterVersion` должна ссылаться ровно на одну `SystemVersion`.

Правило удаления:

- удаление `SystemVersion`, на которую ссылается хотя бы одна
  `CharacterVersion`, недопустимо;
- иначе станет невозможно однозначно понять структуру старого
  `CharacterVersion.data`.

Правило сохранения истории:

- `CharacterVersion.system_version_id` фиксирует схему интерпретации на момент
  создания версии персонажа;
- новая версия RPG-системы не должна менять смысл старых версий персонажей.

### User 1:N CharacterVersion

Эта связь используется, если модель отслеживает автора конкретного изменения
персонажа.

```text
users.id
  -> character_versions.created_by_user_id
```

Cardinality:

- один `User` может быть автором `0..N` записей `CharacterVersion`;
- `CharacterVersion` может иметь одного автора или не иметь его, если авторство
  не отслеживается.

Правило удаления:

- при удалении пользователя исторические версии персонажей не должны удаляться;
- допустимые варианты: запретить удаление пользователя, использовать soft delete
  или выставлять `created_by_user_id` в `NULL`, если это не нарушает требования
  аудита.

### Character 1:N Share

`Share` описывает доступ другого пользователя к персонажу.

```text
characters.id
  -> shares.character_id
```

Cardinality:

- один `Character` может иметь `0..N` записей `Share`;
- каждая запись `Share` должна относиться ровно к одному `Character`.

Правило удаления:

- если персонаж удаляется через soft delete, связанные `Share` можно считать
  неактивными;
- физическое удаление `Share` допустимо при отзыве доступа, потому что `Share`
  не является исторической версией персонажа;
- физическое удаление персонажа вместе с `Share` допустимо только в рамках
  отдельной политики окончательного удаления данных.

### User 1:N Share

`Share.user_id` указывает пользователя, которому выдан доступ.

```text
users.id
  -> shares.user_id
```

Cardinality:

- один `User` может получить доступ к `0..N` персонажам через `Share`;
- каждая запись `Share` должна иметь ровно одного пользователя-получателя.

Ограничения:

```text
UNIQUE (character_id, user_id)
```

Правило удаления:

- при удалении пользователя записи `Share`, где он является получателем, могут
  быть удалены или деактивированы;
- удаление `Share` не должно удалять `Character` или `User`.

### Character N:M User через Share

Многие пользователи могут иметь доступ ко многим персонажам через `Share`.

```text
characters.id -> shares.character_id
users.id      -> shares.user_id
```

Cardinality:

- один `Character` может быть доступен `0..N` дополнительным пользователям;
- один `User` может иметь доступ к `0..N` чужим персонажам;
- каждая пара `character_id` и `user_id` должна быть уникальной.

Permission:

- стартовые значения: `view`, `edit`;
- новые значения прав доступа не должны добавляться без отдельного решения по
  модели доступа.

Правило удаления:

- отзыв доступа выполняется удалением или деактивацией записи `Share`;
- отзыв доступа не влияет на владельца персонажа;
- отзыв доступа не удаляет версии персонажа.

## Правила удаления

Для ядра DiceBound рекомендуется консервативная политика удаления:

- `User`: soft delete или запрет удаления при наличии связанных данных.
- `GameSystem`: soft delete или запрет удаления при наличии связанных
  `SystemVersion` или `Character`.
- `SystemVersion`: запрет удаления, если используется в `CharacterVersion`.
- `Character`: soft delete или запрет физического удаления при наличии
  `CharacterVersion`.
- `CharacterVersion`: не удалять обычными пользовательскими изменениями.
- `Share`: можно удалить или деактивировать при отзыве доступа.

Каскадное удаление нежелательно для сущностей, которые участвуют в истории:

- `SystemVersion`
- `Character`
- `CharacterVersion`

## Правила сохранения истории

### История RPG-систем

`SystemVersion` является исторической версией структуры RPG-системы.

Правила:

- изменение структуры системы создает новую `SystemVersion`;
- старая `SystemVersion.schema` не переписывается так, чтобы изменился смысл уже
  существующих `CharacterVersion`;
- старые `CharacterVersion` продолжают ссылаться на старую `SystemVersion`.

### История персонажей

`CharacterVersion` является историческим снимком состояния персонажа.

Правила:

- изменение персонажа создает новую `CharacterVersion`;
- `version_number` уникален внутри одного `Character`;
- `CharacterVersion.data` интерпретируется через связанную
  `SystemVersion.schema`;
- старые версии не перезаписываются обычным обновлением.

## Проверка на непротиворечивость

Модель связей не противоречит списку основных сущностей:

- `User` связан с владением системами, владением персонажами, созданием версий и
  получением доступа через `Share`.
- `GameSystem` связан с версиями системы и персонажами.
- `SystemVersion` связан с `GameSystem` и `CharacterVersion`.
- `Character` связан с владельцем, RPG-системой, версиями персонажа и доступами.
- `CharacterVersion` связан с `Character` и `SystemVersion`.
- `Share` связывает `Character` и `User` для прав доступа.

В модели нет связи, которая требует сущность вне утвержденного ядра.
