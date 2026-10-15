# Технические контракты DiceBound MVP

> Status: Proposed — контракт для реализации, а не описание уже работающего API.

Документ фиксирует минимальный HTTP API, необходимый для утверждённого MVP DiceBound.
Источник продуктовых требований — [MVP scope](./mvp-scope.md).
Структура листа и модель данных описаны в [SheetSchema and Character Data](./data-model/sheet-schema.md) и [Entities](./data-model/entities.md).

## Общие соглашения

- Архитектура: JSON REST API, базовый путь `/api/v1`.
- Все `id` — UUID, даты — ISO 8601 в UTC, например `2026-10-10T12:00:00Z`.
- В JSON используются имена полей `snake_case`; HTTP-методы имеют стандартную семантику.
- Тело запросов и ответов — `application/json`, если тело предусмотрено.
- Успешные операции возвращают `200 OK`, создание — `201 Created`, удаление или выход без тела — `204 No Content`.
- Списки в MVP возвращаются массивами без обязательной пагинации.
- Авторизация осуществляется через защищённую сессионную cookie: `HttpOnly`, `SameSite=Lax`, `Secure` в production.
  Для изменяющих запросов сервер проверяет источник запроса (Origin/CSRF-защита).
- `GET /auth/me`, все запросы к системам, схемам и персонажам требуют входа, кроме регистрации и входа.
- Права проверяет backend на каждом запросе, независимо от состояния кнопок на frontend.
- Для недоступного или несуществующего персонажа клиент получает одинаковый `404`, чтобы не раскрывать чужие данные.
- `GET /health` — существующая инфраструктурная проверка, не часть бизнес-API MVP.

## Auth

| Метод | Endpoint | Назначение | Успех |
| --- | --- | --- | --- |
| `POST` | `/auth/register` | Регистрация по email и паролю | `201` + `User` |
| `POST` | `/auth/login` | Создать сессию | `200` + `User`, cookie |
| `POST` | `/auth/logout` | Завершить текущую сессию | `204` |
| `GET` | `/auth/me` | Текущий пользователь и показатели профиля | `200` + `CurrentUser` |

**RegisterRequest:** `email: string`, `password: string`, `password_confirmation: string`.
Пароль содержит минимум 8 символов; пароль и подтверждение должны совпадать.
Поле `username` из модели `User` формируется сервером, поскольку в утверждённом сценарии регистрации вводятся только email и пароль.

**LoginRequest:** `email: string`, `password: string`.
При ошибке входа сервер возвращает нейтральное сообщение, не уточняя, неверен email или пароль.

**User:** `id`, `username`, `email`, `created_at`.
**CurrentUser:** поля `User` + `owned_characters_count: number` и `shared_characters_count: number` для страницы профиля.
Пароли и их хэши никогда не передаются в ответах.

Пример регистрации:

```http
POST /api/v1/auth/register
Content-Type: application/json

{"email":"player@example.com","password":"safe-password-123","password_confirmation":"safe-password-123"}
```

```json
{
  "id": "11111111-1111-4111-8111-111111111111",
  "username": "player",
  "email": "player@example.com",
  "created_at": "2026-10-10T12:00:00Z"
}
```

Истёкшая или отсутствующая сессия: `401 UNAUTHORIZED`.
Frontend перенаправляет пользователя на вход и после входа возвращает на исходную страницу.

## Game Systems и импорт JSON

| Метод | Endpoint | Назначение | Успех |
| --- | --- | --- | --- |
| `GET` | `/systems` | Встроенные и собственные импортированные системы | `200` + `GameSystemSummary[]` |
| `GET` | `/systems/{system_id}` | Данные доступной системы | `200` + `GameSystemSummary` |
| `POST` | `/systems/import/validate` | Проверить текст JSON и правила `SheetSchema` | `200` + `SchemaValidationResult` |
| `POST` | `/systems/import` | Создать импортированную систему вместе со схемой | `201` + `ImportedSystemResult` |
| `DELETE` | `/systems/{system_id}` | Скрыть собственную импортированную систему | `204` |

**GameSystemSummary:**

- `id: UUID`;
- `owner_id: UUID | null` (`null` для встроенной системы);
- `name: string`;
- `description: string | null`;
- `source: "BUILT_IN" | "IMPORTED"`;
- `character_count: number`;
- `created_at: string`.

Встроенные системы — упрощённые D&D 5e и Call of Cthulhu.
Удалять их нельзя.
Импортированные системы видны владельцу; чужие импортированные системы не становятся общедоступной библиотекой.

**SchemaValidationRequest:** `schema_text: string`.
`schema_text` — исходный JSON-текст, вставленный пользователем или прочитанный frontend из `.json`-файла.
Сервер проверяет синтаксис JSON, допустимые секции/поля, уникальность `field.id`, допустимые типы, ограничения и формулы.

**SchemaValidationResult:**

```json
{
  "valid": true,
  "errors": [],
  "summary": {"sections_count": 1, "fields_count": 2},
  "schema": {
    "sections": [
      {
        "id": "basic",
        "label": "Basic",
        "fields": [
          {"id": "strength", "type": "number", "label": "Strength", "min": 1, "max": 30},
          {"id": "modifier", "type": "calculated", "label": "Modifier", "formula": "floor((strength - 10) / 2)"}
        ]
      }
    ]
  }
}
```

Для невалидного текста: `200` с `valid: false`, `schema: null`, `summary: null` и массивом `errors`.
Каждый элемент `errors`: `path: string`, `message: string`, `line: number | null`.
Номер строки заполняется, когда его можно определить по исходному тексту, в остальных случаях используется `null`.
Предпросмотр корректной схемы frontend строит через существующий `Dynamic Sheet Renderer` из `schema` в ответе.

**ImportGameSystemRequest:** `system_name: string`, `sheet_schema_name: string`, `schema_text: string`.

**ImportedSystemResult:**

```json
{
  "system": {
    "id": "22222222-2222-4222-8222-222222222222",
    "owner_id": "11111111-1111-4111-8111-111111111111",
    "name": "My RPG",
    "description": null,
    "source": "IMPORTED",
    "character_count": 0,
    "created_at": "2026-10-10T12:00:00Z"
  },
  "sheet_schema": {
    "id": "33333333-3333-4333-8333-333333333333",
    "game_system_id": "22222222-2222-4222-8222-222222222222",
    "name": "Basic Sheet",
    "schema": {"sections": [{"id": "basic", "label": "Basic", "fields": [{"id": "strength", "type": "number", "label": "Strength", "min": 1, "max": 30}]}]},
    "created_at": "2026-10-10T12:00:00Z"
  }
}
```

`sheet_schema.schema` в ответе содержит полную валидированную структуру импортированного листа.
Импорт создаёт `GameSystem` и `SheetSchema` атомарно.
Неверная схема при импорте: `422 VALIDATION_ERROR`, ничего не создаётся.

Удаление импортированной системы означает исключение её из списка и запрет выбора для *новых* персонажей.
Её `SheetSchema` и метаданные сохраняются для существующих персонажей, их историй и копий через `EDIT`.
Физическое каскадное удаление недопустимо.

## SheetSchema

| Метод | Endpoint | Назначение | Успех |
| --- | --- | --- | --- |
| `GET` | `/systems/{system_id}/schemas` | Доступные схемы выбранной системы | `200` + `SheetSchemaSummary[]` |
| `GET` | `/schemas/{schema_id}` | Полная схема для `Dynamic Sheet Renderer` | `200` + `SheetSchema` |

**SheetSchemaSummary:** `id`, `game_system_id`, `name`, `created_at`.

**SheetSchema:** `id`, `game_system_id`, `name`, `schema: object`, `created_at`.
`schema` имеет форму `{ "sections": [{ "id": "...", "label": "...", "fields": [...] }] }`.
Каждое поле имеет уникальный в пределах схемы `id`, `type`, `label` и параметры своего типа.
Поддерживаются ровно шесть типов:

| Тип | Значение в `Character.data` | Основные правила |
| --- | --- | --- |
| `text` | `string` | `required`, `maxLength` |
| `number` | `number` | `min`, `max` |
| `select` | выбранный `options[].value` | значение из списка |
| `checkbox` | `boolean` | `true` / `false` |
| `resource` | `{ "current": number, "max": number }` | `min <= current <= max` |
| `calculated` | рассчитанное значение | только формула; не вводится пользователем |

Формулы поддерживают `+`, `-`, `*`, `/`, скобки, `floor`, `ceil`, `round`, `min`, `max`.
Произвольный JavaScript и `eval()` запрещены.
Ошибки формул, неизвестные ссылки, циклы зависимостей и деление на ноль отклоняются при валидации/вычислении.

Опубликованная используемая `SheetSchema` неизменяема.
Редактирование, удаление схем и визуальный Schema Editor отдельными endpoints в MVP не предусмотрены.
Несовместимые схемы подключаются новым импортом.

## Characters

| Метод | Endpoint | Назначение | Успех |
| --- | --- | --- | --- |
| `GET` | `/characters` | Свои персонажи, копии и доступные по `VIEW` | `200` + `CharacterList` |
| `POST` | `/characters` | Создать персонажа и версию `v1` | `201` + `Character` |
| `GET` | `/characters/{character_id}` | Получить лист и права текущего пользователя | `200` + `CharacterDetails` |
| `PUT` | `/characters/{character_id}` | Логическое сохранение листа | `200` + `CharacterDetails` |
| `DELETE` | `/characters/{character_id}` | Удалить своего персонажа с историей и VIEW-доступами | `204` |

**CreateCharacterRequest:**

```json
{
  "game_system_id": "22222222-2222-4222-8222-222222222222",
  "sheet_schema_id": "33333333-3333-4333-8333-333333333333",
  "name": "Arin"
}
```

Сервер проверяет принадлежность `sheet_schema_id` указанной системе.
При создании `Character.data` инициализируется из полей/значений по умолчанию схемы, вычисляются доступные формулы и создаётся `CharacterVersion` с номером `1`.
Создание чернового листа не равнозначно пользовательскому сохранению заполненного листа: обязательные поля окончательно проверяются при `PUT`.

**SaveCharacterRequest:** `data: Record<string, unknown>` — *полный* набор пользовательских значений, без значений типа `calculated`.
`PUT` — одна операция «Сохранить», а не autosave каждого поля.
Backend валидирует `data`, вычисляет `calculated`, затем в одной транзакции обновляет `Character.data` и добавляет новую `CharacterVersion`.
При ошибке не изменяет текущее состояние и не создаёт версию.

Пример запроса сохранения:

```http
PUT /api/v1/characters/44444444-4444-4444-8444-444444444444
Content-Type: application/json

{"data":{"strength":16}}
```

Пример данных после вычисления:

```json
{
  "strength": 16,
  "modifier": 3
}
```

**Character:** `id`, `owner_id`, `game_system_id`, `sheet_schema_id`, `name`, `data`, `created_at`, `updated_at`, `current_version_number`.
`data` — плоский JSON-объект со значениями по `field.id`, без вложенности по секциям.
`resource` хранит объект `{current,max}`; `calculated` присутствует в ответах, но не принимается от пользователя.

**CharacterDetails:** `Character` + `access: "OWNER" | "VIEW"`, `can_edit: boolean`, `can_view_history: boolean`, `owner_username: string`.
Для своей копии, полученной через `EDIT`, `access = "OWNER"`: это самостоятельный персонаж.

**CharacterSummary:** `id`, `name`, `game_system_id`, `game_system_name`, `updated_at`, `access`, `owner_username`, `copy_source_owner_name: string | null`.
`copy_source_owner_name` заполняется для копий, чтобы показать «Копия от …».

**CharacterList:**

```json
{
  "owned": [],
  "shared_view": []
}
```

`owned` включает обычных собственных персонажей и независимые копии через `EDIT`.
`shared_view` содержит только оригиналы, доступные по `VIEW`.
Каждый элемент массива имеет тип `CharacterSummary`.

Редактирование и удаление доступны только `OWNER`.
При удалении удаляются данные, история и действующие VIEW-доступы владельца.
Ранее переданные через `EDIT` копии остаются у получателей.

## Character Version History и restore

| Метод | Endpoint | Назначение | Успех |
| --- | --- | --- | --- |
| `GET` | `/characters/{character_id}/versions` | Список версий от новых к старым | `200` + `CharacterVersionSummary[]` |
| `GET` | `/characters/{character_id}/versions/{version_number}` | Снимок выбранной версии | `200` + `CharacterVersion` |
| `POST` | `/characters/{character_id}/versions/{version_number}/restore` | Восстановить версию, удалив более поздние | `200` + `RestoreResult` |

**CharacterVersionSummary:** `id`, `version_number`, `created_at`, `is_current`.
Версия `1` на frontend отображается как «создание персонажа».

**CharacterVersion:** `id`, `character_id`, `version_number`, `data`, `created_at`.
`data` содержит *полный* снимок данных листа, включая вычисленные поля.
При просмотре история отображается тем же `Dynamic Sheet Renderer` в режиме read-only.

**RestoreResult:**

```json
{
  "character": {
    "id": "44444444-4444-4444-8444-444444444444",
    "current_version_number": 4,
    "data": {"strength": 16, "modifier": 3}
  },
  "deleted_versions_count": 3
}
```

В примере объект `character` сокращён; в реальном ответе он соответствует `Character`.
Восстановление `v4` из `v1`–`v7` копирует snapshot `v4` в `Character.data` и удаляет `v5`–`v7`.
Новая версия при restore **не создаётся**.
После следующего сохранения создаётся `v5`.
Восстановление текущей версии отклоняется (`409 RESTORE_NOT_ALLOWED`).
Операция выполняется транзакционно и требует подтверждения пользователя на frontend.

`OWNER` может просматривать историю и восстанавливать версии.
`VIEW` может только читать историю и снимки, **если** при выдаче доступа `history_visible = true`.
При скрытой истории методы возвращают `404`, как и для недоступного ресурса.

## Sharing VIEW / EDIT

| Метод | Endpoint | Назначение | Успех |
| --- | --- | --- | --- |
| `GET` | `/characters/{character_id}/shares` | Перечень предоставленных доступов | `200` + `CharacterShare[]` |
| `POST` | `/characters/{character_id}/shares` | Выдать `VIEW` либо передать копию через `EDIT` | `201` + `ShareResult` |
| `DELETE` | `/characters/{character_id}/shares/{share_id}` | Отозвать `VIEW` | `204` |

Все методы управления доступом доступны только владельцу *исходного* персонажа.
Владелец собственной копии может управлять доступом к своей копии независимо.

**CreateShareRequest:**

```json
{
  "recipient_email": "friend@example.com",
  "permission": "VIEW",
  "history_visible": false
}
```

`permission` принимает `VIEW` или `EDIT`.
`history_visible` имеет смысл только для `VIEW`, по умолчанию `false`; для `EDIT` параметр не передаётся.
Получатель должен быть зарегистрированным пользователем.
Выдача себе доступа, повторная выдача той же паре `character_id + user_id` и выдача несуществующему получателю отклоняются.
Сообщение о несуществующем email нейтральное и не подтверждает наличие аккаунта.

**CharacterShare:** `id`, `character_id`, `user_id`, `recipient_email`, `permission`, `history_visible: boolean | null`, `created_at`.

**ShareResult для VIEW:**

```json
{
  "share": {
    "id": "55555555-5555-4555-8555-555555555555",
    "character_id": "44444444-4444-4444-8444-444444444444",
    "user_id": "66666666-6666-4666-8666-666666666666",
    "recipient_email": "friend@example.com",
    "permission": "VIEW",
    "history_visible": false,
    "created_at": "2026-10-10T12:00:00Z"
  },
  "copy_character_id": null
}
```

`VIEW` предоставляет read-only доступ к оригиналу.
Получатель может открыть его из списка `shared_view`, но не может изменять, восстанавливать или удалять.
Владелец вправе отозвать `VIEW`; после отзыва оригинал исчезает из `shared_view`.
Параметры существующей выдачи не изменяются: для изменения прав нужно отозвать `VIEW` и выдать доступ повторно.

**ShareResult для EDIT:** структура та же, но `permission = "EDIT"`, `history_visible = null`, `copy_character_id` содержит UUID новой копии.
При выдаче `EDIT` backend атомарно создаёт у получателя отдельный `Character` с именем `"{исходное имя} (копия)"`, текущими `data` и собственной версией `v1`.
Копия принадлежит получателю, редактируется независимо, не синхронизируется с оригиналом и не удаляется при удалении оригинала.
`EDIT` не даёт права изменять оригинал.
Передачу `EDIT` **нельзя отозвать**, удалить или заменить обновлением записи.
Попытка вызвать `DELETE` для такой выдачи возвращает `409 SHARE_IRREVOCABLE`.

## Единый формат ошибок API

Все неуспешные запросы возвращают единый JSON-объект.
Формат сохраняет поля `statusCode`, `timestamp` и `path`, уже используемые глобальным `AllExceptionsFilter` NestJS.

```json
{
  "statusCode": 422,
  "timestamp": "2026-10-10T12:00:00Z",
  "path": "/api/v1/characters/44444444-4444-4444-8444-444444444444",
  "code": "VALIDATION_ERROR",
  "message": "Данные персонажа не соответствуют схеме",
  "details": [
    {"field": "data.strength", "message": "Значение должно быть от 1 до 30"}
  ]
}
```

**ErrorResponse:**

| Поле | Тип | Описание |
| --- | --- | --- |
| `statusCode` | `number` | HTTP-статус |
| `timestamp` | `string` | Время ошибки ISO 8601 |
| `path` | `string` | Путь исходного запроса |
| `code` | `string` | Стабильный машиночитаемый код |
| `message` | `string` | Безопасное сообщение для пользователя |
| `details` | `Array<{field?: string, path?: string, line?: number, message: string}>` | Необязательные подробности валидации |

`details` опускается, если подробностей нет.
Ошибки сервера `5xx` не раскрывают stack trace или внутренние SQL-ошибки.

| HTTP | Код | Ситуация |
| --- | --- | --- |
| `400` | `BAD_REQUEST` | Некорректный JSON или параметры маршрута |
| `401` | `UNAUTHORIZED` | Отсутствующая, истёкшая или недействительная сессия |
| `403` | `FORBIDDEN` | Запрещённая операция над доступным ресурсом |
| `404` | `NOT_FOUND` | Ресурс отсутствует или должен быть скрыт |
| `409` | `CONFLICT` | Конфликт состояния или дублирование |
| `409` | `RESTORE_NOT_ALLOWED` | Попытка восстановить текущую версию |
| `409` | `SHARE_IRREVOCABLE` | Попытка отозвать `EDIT` |
| `422` | `VALIDATION_ERROR` | Невалидная схема или данные листа |
| `422` | `SHARE_UNAVAILABLE` | Получателю нельзя выдать доступ; сообщение не раскрывает наличие аккаунта |
| `500` | `INTERNAL_ERROR` | Непредвиденная ошибка backend |

Ответ `404` используется при попытке получить чужого персонажа или его скрытую историю, даже если ресурс технически существует.
Для формул сервер возвращает `422 VALIDATION_ERROR` с `details.field` или `details.path`, соответствующим проблемному полю.
Для повторной выдачи доступа допустим `409 CONFLICT` без раскрытия дополнительных данных чужих аккаунтов.

Для реализации требуется привести `AllExceptionsFilter` к этой форме: всегда добавлять `code`, нормализовать `message`, передавать `details` только в безопасном виде.

## Проверки контракта

- [ ] Регистрация, вход, выход и `GET /auth/me` работают с сессией.
- [ ] Список систем содержит две встроенные системы и импортированные пользователем.
- [ ] Проверка схемы обнаруживает неизвестные типы полей, неверные ограничения и некорректные формулы.
- [ ] Импорт создаёт систему и схему; удаление импортированной системы не ломает существующих персонажей.
- [ ] Персонаж создаётся из выбранной `SheetSchema` и получает `v1`.
- [ ] `PUT /characters/{id}` валидирует данные, считает формулы и создаёт ровно одну версию на сохранение.
- [ ] История возвращает snapshots от новых к старым.
- [ ] Restore удаляет последующие версии без создания новой.
- [ ] `VIEW` открывает оригинал только для чтения и учитывает `history_visible`.
- [ ] `EDIT` создаёт независимую копию с собственной `v1` и не может быть отозван.
- [ ] Недопустимые действия возвращают единый `ErrorResponse` без утечки чужих данных.

## Согласование с моделью данных

Контракт выше следует утверждённому поведению MVP.
В текущей принципиальной ERD ещё не указаны некоторые технические атрибуты, необходимые для такого поведения:

1. `CharacterShare.history_visible` — выбор видимости истории для `VIEW`; по умолчанию `false`.
2. Связь выданного `EDIT` с созданной копией и информация об источнике копии — для `copy_character_id` и подписи «Копия от …».
3. Признак архивирования импортированной `GameSystem`, чтобы удаление из списка не удаляло схемы и персонажей.
4. Способ идентификации встроенной `GameSystem` (`source` в DTO, nullable `owner_id` либо эквивалентный признак).

Эти уточнения относятся к реализации уже утверждённого MVP, а не к post-MVP функциям.
Они должны быть согласованы с ERD при детализации таблиц.

## Не входит в MVP

OAuth, Google/Discord login, 2FA, восстановление пароля, campaigns, команды, WebSocket, realtime-редактирование, публичные ссылки, anonymous sharing, приглашения незарегистрированных пользователей, визуальный Schema Editor, миграции схем, visual diff, marketplace, файлы и аватары.

## Связанные документы

- [DiceBound MVP](./mvp-scope.md)
- [SheetSchema and Character Data](./data-model/sheet-schema.md)
- [Entities](./data-model/entities.md)
- [Relationships](./data-model/relationships.md)
- [ERD](./data-model/erd.md)
- [UI/UX](./ui-ux.md)
