# Continuous Integration

Документ описывает CI для `DiceBound-backend` и `DiceBound-frontend`.
Он фиксирует условия запуска workflow, выполняемые проверки, required status checks и требования для merge в `main`.

## Запуск CI

Backend и frontend используют GitHub Actions.

Workflow запускается:

- для Pull Request в `main`;
- после push в `main`;
- вручную через `workflow_dispatch`.

Для нескольких запусков одного workflow на одной Git reference используется `concurrency`.
Новый запуск отменяет предыдущий незавершённый запуск для той же reference.

Каждый CI job имеет timeout 15 минут.
Workflow использует разрешение `contents: read`.

Node.js настраивается по `.nvmrc`.
В обоих репозиториях используется Node.js `24`.

Перед запуском Node.js checks workflow проверяет наличие `package.json` и `package-lock.json`.
Если присутствуют оба файла, проверки продолжаются.
Если отсутствуют оба файла, Node.js checks пропускаются.
Если присутствует только один файл, workflow завершается с ошибкой.

## Backend CI

Workflow находится в `DiceBound-backend/.github/workflows/ci.yml`.

Backend workflow:
[CI](/Smert-Interactive/DiceBound-backend/blob/main/.github/workflows/ci.yml)

Название workflow — `Backend CI`.
Название job и status check — `backend-ci`.

После checkout и настройки Node.js выполняются:

```bash
npm ci
npm run lint
npm test
npm run build
```

Команды соответствуют следующим scripts из `package.json`:

```text
lint  -> oxlint --type-aware src/ test/
test  -> vitest run
build -> nest build
```

Ошибка любой команды завершает `backend-ci` с ошибкой.

## Frontend CI

Workflow находится в `DiceBound-frontend/.github/workflows/ci.yml`.

Frontend workflow:
[CI](/Smert-Interactive/DiceBound-frontend/blob/main/.github/workflows/ci.yml)

Название workflow — `Frontend CI`.
Название job и status check — `frontend-ci`.

После checkout и настройки Node.js выполняются:

```bash
npm ci
npm run lint
npm run test
npm run build
```

Команды соответствуют следующим scripts из `package.json`:

```text
lint  -> eslint .
test  -> vitest
build -> tsc -b && vite build
```

Ошибка любой команды завершает `frontend-ci` с ошибкой.

## Required status checks

Для `main` обоих репозиториев действует ruleset `Protect main`.

В `DiceBound-backend` required status check:

```text
backend-ci
```

В `DiceBound-frontend` required status check:

```text
frontend-ci
```

Required status check должен завершиться успешно перед обычным merge Pull Request.

Strict status checks policy отключена.
Ruleset не требует предварительно обновлять branch относительно последнего состояния `main`
только для повторного выполнения required status check.

## Требования для merge

Ruleset `Protect main` требует:

- Pull Request;
- минимум один approving review;
- успешный required status check;
- разрешение всех review threads;
- linear history;
- merge методом `squash`.

После нового push предыдущие approvals считаются устаревшими и сбрасываются.

Для unattributed changes включено дополнительное требование approval.

Ruleset запрещает удаление `main` и non-fast-forward updates.

## Диагностика падения CI

Откройте неуспешный workflow в GitHub Actions или Checks Pull Request и найдите первый
step со статусом failure.

Основные steps соответствуют локальным командам:

```text
Install dependencies -> npm ci
Lint                 -> npm run lint
Test                 -> npm test / npm run test
Build                -> npm run build
```

Для воспроизведения Backend CI из корня `DiceBound-backend`:

```bash
npm ci
npm run lint
npm test
npm run build
```

Для воспроизведения Frontend CI из корня `DiceBound-frontend`:

```bash
npm ci
npm run lint
npm run test
npm run build
```

Если падает `Detect Node.js project`, проверьте наличие `package.json` и `package-lock.json`.
Оба файла должны быть либо закоммичены вместе, либо отсутствовать вместе.

Если падает `Set up Node.js`, проверьте `.nvmrc`.

Если падает `npm ci`, проверьте соответствие `package.json` и `package-lock.json`.

Если падает `Lint`, `Test` или `Build`, воспроизведите соответствующую команду локально
и исправьте ошибку.

Статус `cancelled` после нового push может быть результатом работы `concurrency`, поскольку
предыдущий незавершённый запуск автоматически отменяется.

Повторный запуск workflow не заменяет исправление ошибок lint, tests или build.

## Источники конфигурации

Фактическое поведение Backend CI определяется:

```text
DiceBound-backend/.github/workflows/ci.yml
DiceBound-backend/package.json
DiceBound-backend/.nvmrc
```

Фактическое поведение Frontend CI определяется:

```text
DiceBound-frontend/.github/workflows/ci.yml
DiceBound-frontend/package.json
DiceBound-frontend/.nvmrc
```

Требования для merge определяются ruleset `Protect main` соответствующего репозитория.

При изменении workflow, npm scripts или ruleset этот документ должен обновляться, если
описанное поведение перестаёт соответствовать фактической конфигурации.
