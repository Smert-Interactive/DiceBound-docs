# SheetSchema and Character Data

Документ фиксирует универсальный формат `SheetSchema` и
`Character.data` для schema-driven архитектуры DiceBound.
Формат не зависит от конкретной игровой системы.

`SheetSchema` хранится в `sheet_schemas.schema` типа JSONB.
`Character.data` хранит значения полей в формате JSONB.

## SheetSchema

`SheetSchema` содержит `version`, `name` и список секций.
Секция содержит поля с уникальными `id`.

```json
{
  "version": 1,
  "name": "D&D 5e",
  "sections": [
    {
      "id": "abilities",
      "label": "Abilities",
      "fields": [
        {
          "id": "strength",
          "type": "number",
          "label": "Strength"
        }
      ]
    }
  ]
}
```

### section

Секция содержит `id`, `label` и `fields`.
`section.id` используется для идентификации секции.
Значения персонажа по `section.id` не хранятся.

### Field

Каждое поле содержит `id`, `type` и `label`.
`field.id` должен быть уникальным в `SheetSchema`.

Поддерживаются типы:

- `text`;
- `number`;
- `select`;
- `checkbox`;
- `resource`;
- `calculated`.

## Типы полей

### text

Хранит строковое значение.

```json
{
  "id": "character_name",
  "type": "text",
  "label": "Name",
  "required": true,
  "maxLength": 100
}
```

Проверяются тип, обязательность и `maxLength`, если он задан.

### number

Хранит числовое значение.

```json
{
  "id": "strength",
  "type": "number",
  "label": "Strength",
  "min": 1,
  "max": 30
}
```

Проверяются тип и ограничения `min` и `max`, если они заданы.

### select

Хранит одно значение из `options`.

```json
{
  "id": "class",
  "type": "select",
  "label": "Class",
  "options": [
    {
      "value": "fighter",
      "label": "Fighter"
    },
    {
      "value": "wizard",
      "label": "Wizard"
    }
  ]
}
```

Значение должно совпадать с одним из `options[].value`.

### checkbox

Хранит значение `true` или `false`.

```json
{
  "id": "inspiration",
  "type": "checkbox",
  "label": "Inspiration"
}
```

### resource

Хранит текущий и максимальный ресурс.

```json
{
  "id": "hit_points",
  "type": "resource",
  "label": "Hit Points",
  "min": 0
}
```

Значение в `Character.data`:

```json
{
  "current": 18,
  "max": 24
}
```

`current` не может быть меньше `min` или больше `max`.
`max` не может быть меньше `min`.

### calculated

Хранит значение, вычисляемое Formula Engine.
Пользователь не изменяет такое поле напрямую.

```json
{
  "id": "strength_modifier",
  "type": "calculated",
  "label": "Strength modifier",
  "formula": {
    "op": "floor",
    "args": [
      {
        "op": "div",
        "args": [
          {
            "op": "sub",
            "args": [
              {
                "ref": "strength"
              },
              10
            ]
          },
          2
        ]
      }
    ]
  }
}
```

## Character.data

`Character.data` содержит значения полей связанной `SheetSchema`.

Если схема содержит поле `strength`, данные содержат:

```json
{
  "strength": 14
}
```

Для `resource` используется объект `current` и `max`.
Для `calculated` ключ также совпадает с `field.id`.

`field.id` является контрактом между `SheetSchema` и `Character.data`.
Секции в `Character.data` не хранятся.

## Базовая валидация

Перед сохранением backend проверяет `Character.data` по
связанной `SheetSchema`.

Проверяются:

- обязательные поля;
- тип значения;
- `min` и `max` для `number`;
- `maxLength` для `text`;
- допустимое значение `select`;
- boolean для `checkbox`;
- `current` и `max` для `resource`;
- отсутствие пользовательского значения для `calculated`.

Неизвестные `field.id` не должны добавляться в `Character.data`.
`calculated` поля вычисляются backend через Formula Engine.

## Formula Engine

Formula Engine вычисляет поля типа `calculated`.
Формула содержит `op` и массив `args`.
Ссылка на другое поле задаётся через `ref`.

```json
{
  "op": "add",
  "args": [
    {
      "ref": "strength"
    },
    2
  ]
}
```

Поддерживаются:

- `add`, `sub`, `mul`, `div`, `mod`;
- `eq`, `ne`, `gt`, `gte`, `lt`, `lte`;
- `and`, `or`, `not`;
- `min`, `max`, `abs`;
- `floor`, `ceil`, `round`;
- `if`.

Арифметические операции работают с числами.
Операции сравнения возвращают boolean.
`and`, `or` и `not` работают с boolean.
`if` принимает условие и два результата.

Formula Engine не выполняет произвольный код.
Ссылка на неизвестное поле является ошибкой.
Деление на ноль является ошибкой.
Циклическая зависимость является ошибкой валидации.

## Сохранение Character

При логическом сохранении backend:

1. получает `Character.data`;
2. проверяет данные по `SheetSchema`;
3. вычисляет `calculated fields`;
4. обновляет `Character.data`;
5. создаёт новый `CharacterVersion`.

`CharacterVersion.data` содержит полный снимок результата сохранения.

## Пример D&D 5e

```json
{
  "version": 1,
  "name": "D&D 5e",
  "sections": [
    {
      "id": "basic",
      "label": "Basic",
      "fields": [
        {
          "id": "name",
          "type": "text",
          "label": "Name",
          "required": true
        },
        {
          "id": "strength",
          "type": "number",
          "label": "Strength",
          "min": 1,
          "max": 30
        },
        {
          "id": "class",
          "type": "select",
          "label": "Class",
          "options": [
            {
              "value": "fighter",
              "label": "Fighter"
            },
            {
              "value": "wizard",
              "label": "Wizard"
            }
          ]
        }
      ]
    }
  ]
}
```

Пример `Character.data`:

```json
{
  "name": "Arin",
  "strength": 16,
  "class": "fighter"
}
```

## Пример Call of Cthulhu

```json
{
  "version": 1,
  "name": "Call of Cthulhu",
  "sections": [
    {
      "id": "basic",
      "label": "Basic",
      "fields": [
        {
          "id": "name",
          "type": "text",
          "label": "Name",
          "required": true
        },
        {
          "id": "power",
          "type": "number",
          "label": "Power",
          "min": 1,
          "max": 99
        },
        {
          "id": "unconscious",
          "type": "checkbox",
          "label": "Unconscious"
        }
      ]
    }
  ]
}
```

Пример `Character.data`:

```json
{
  "name": "Eleanor",
  "power": 60,
  "unconscious": false
}
```

Обе системы используют одинаковую структуру `SheetSchema`.
Отличается только набор полей и их значения.

## Пользовательская НРИ

Пользовательская НРИ использует тот же формат через JSON import.
Импортированный JSON становится значением `SheetSchema.schema`.

```json
{
  "version": 1,
  "name": "My RPG",
  "sections": [
    {
      "id": "basic",
      "label": "Basic",
      "fields": [
        {
          "id": "name",
          "type": "text",
          "label": "Name",
          "required": true
        },
        {
          "id": "level",
          "type": "number",
          "label": "Level",
          "min": 1
        }
      ]
    }
  ]
}
```

После import схема проходит базовую валидацию.
Она может использоваться для создания `Character`.
Значения хранятся в `Character.data` по тем же `field.id`.

## Ограничения

`SheetSchema` считается неизменяемой после публикации в MVP.
Несовместимое изменение требует создания нового `SheetSchema`.

Перенос `Character` между несовместимыми схемами не входит в MVP.
`SystemVersion` не используется в текущей MVP-модели.
Инвентарь и заклинания не входят в функциональность MVP.

## Связь с моделью данных

`GameSystem` может иметь несколько `SheetSchema`.
`Character` ссылается на конкретный `SheetSchema` через
`sheet_schema_id`.

`SheetSchema.schema` описывает структуру.
`Character.data` хранит значения.
`CharacterVersion.data` хранит снимок данных при сохранении.

Подробнее см. [entities.md](entities.md).
Связи модели описаны в [relationships.md](relationships.md).
ERD находится в [erd.md](erd.md).
