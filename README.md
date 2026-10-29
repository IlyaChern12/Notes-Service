# Notes Service

## О проекте

Notes Service – клиент-серверная система для создания, хранения,
редактирования и синхронизации пользовательских заметок между несколькими
устройствами.

Проект разработан в рамках дисциплины «Создание клиент-серверных приложений»
и является продолжением архитектурного анализа приложений для работы
с заметками. Цель проектирования – архитектура, обеспечивающая
надёжное хранение пользовательских данных, согласованность состояния между
устройствами и формализованное взаимодействие клиентской и серверной частей.

## Основные возможности

- регистрация, вход, обновление и отзыв токенов;
- создание, получение, редактирование и мягкое удаление заметок;
- история версий заметки и обнаружение конфликтов при конкурентном редактировании;
- теги заметок;
- синхронизация изменений между устройствами с подтверждением обработки;
- вложения во внешнем файловом хранилище;
- безопасный повтор изменяющих запросов (`Idempotency-Key`);
- работа ИИ-ассистента с заметками по протоколу MCP (этап проектирования).

## Архитектура

Серверная часть – модульный монолит ([ADR-001](docs/architecture/ADR-001.md)), разделённый на четыре ограниченных контекста ([ADR-002](docs/architecture/ADR-002-bounded-contexts.md)):

- **Identity & Access** – регистрация, вход и управление токенами;
- **Notes** – заметки, история версий, теги, проверка владельца;
- **Synchronization** – доставка изменений устройствам и подтверждения;
- **Attachments** – метаданные и хранение вложений через ACL внешнего хранилища.

Клиенты взаимодействуют с сервером по REST API (OpenAPI 3.0), ИИ-клиенты – по MCP. Источник истины – PostgreSQL; Redis используется как необязательный кэш снимков версий; файлы хранятся в S3-совместимом хранилище. Копии серверного приложения не хранят состояние и масштабируются горизонтально за балансировщиком ([ADR-005](docs/architecture/ADR-005-scaling.md)).

![Container](docs/architecture/c4-container.png)

## Архитектурные решения

| ADR | Решение |
|---|---|
| [ADR-001](docs/architecture/ADR-001.md) | Модульный монолит, REST API, PostgreSQL, JWT, версионирование заметок |
| [ADR-002](docs/architecture/ADR-002-bounded-contexts.md) | Ограниченные контексты и модуль Identity & Access |
| [ADR-003](docs/architecture/ADR-003-concurrency-control.md) | Управление конкурентным доступом: версии, блокировки, ключи идемпотентности, outbox |
| [ADR-004](docs/architecture/ADR-004-tokens.md) | Токены доступа и обновления, ротация, CSRF-защита |
| [ADR-005](docs/architecture/ADR-005-scaling.md) | Горизонтальное масштабирование, кэш, фоновые обработчики |
| [ADR-006](docs/architecture/ADR-006-mcp-adapter.md) | MCP-адаптер для ИИ-клиентов |

## API

Контракт описан в [openapi.yaml](docs/architecture/openapi.yaml) (версия 1.1.0, обратно совместима с 1.0.0). Базовый адрес – `/api/v1`.

| Метод | Endpoint | Назначение |
|---|---|---|
| POST | `/auth/register` | Регистрация пользователя |
| POST | `/auth/login` | Вход: access token и refresh-cookie |
| POST | `/auth/refresh` | Обновление access token с ротацией refresh |
| POST | `/auth/logout` | Завершение текущего входа |
| GET | `/notes` | Список заметок (ключевая пагинация) |
| POST | `/notes` | Создание заметки |
| GET | `/notes/{noteId}` | Получение заметки |
| PATCH | `/notes/{noteId}` | Изменение заметки с проверкой версии |
| DELETE | `/notes/{noteId}` | Мягкое удаление заметки |
| PUT | `/notes/{noteId}/tags` | Установка тегов заметки |
| GET | `/tags` | Справочник тегов пользователя |
| GET | `/sync` | Получение изменений (since или cursor) |
| POST | `/sync/{sessionId}/ack` | Подтверждение обработки изменений устройством |
| POST | `/notes/{noteId}/attachments` | Начало прикрепления файла |
| POST | `/attachments/{attachmentId}/attempts/{attemptId}/confirm` | Подтверждение хранения и прикрепление |
| GET | `/attachments/{attachmentId}` | Сведения о вложении и ссылка на файл |

## Аутентификация

Access token – JWT на 15 минут (`sub`, `iat`, `exp`), передаётся в `Authorization: Bearer <token>`. Refresh token – непрозрачное значение на 30 дней в cookie `Secure; HttpOnly; SameSite=Strict; Path=/api/v1/auth`, ротируется при каждом обновлении. Подробнее – [ADR-004](docs/architecture/ADR-004-tokens.md).

## Синхронизация и конкурентный доступ

Каждая заметка содержит номер версии. Изменение применяется условным `UPDATE … WHERE version = ожидаемая`; если заметку уже изменили с другого устройства, данные не перезаписываются и клиент получает `409 VERSION_CONFLICT` с текущей версией. Изменение заметки и событие для синхронизации фиксируются одной транзакцией (outbox); устройство подтверждает обработку изменений, поэтому обрыв связи не теряет прогресс. Разбор гонок – [race-conditions.md](docs/concurrency/race-conditions.md).

## Документация

| Раздел | Содержание |
|---|---|
| [docs/analysis](docs/analysis/user-reviews-analysis.md) | ЛР1: анализ отзывов Evernote, дефекты R1–R10 |
| [docs/ddd](docs/ddd/README.md) | ЛР2: единый язык, карта контекстов, агрегаты, сценарии, события |
| [docs/architecture](docs/architecture) | ЛР3: архитектурный паспорт (ADR, C4, OpenAPI, ER) и ADR последующих работ |
| [docs/concurrency](docs/concurrency/race-conditions.md) | ЛР4: состояния гонки и блокировки |
| [docs/mcp](docs/mcp/README.md) | ЛР5: спецификация MCP-интерфейса, трассируемость, реестр данных |

## Структура репозитория

```
repo/
├── README.md
├── .gitignore
└── docs/
    ├── analysis/
    │   └── user-reviews-analysis.md
    ├── ddd/
    │   ├── README.md
    │   ├── glossary.md
    │   ├── context-map.md, context-map.png
    │   ├── aggregates.md, aggregate-*.png
    │   ├── scenarios.md, components.png, flow-*.png
    ├── architecture/
    │   ├── ADR-001.md … ADR-006-mcp-adapter.md
    │   ├── c4-diagram.drawio, c4-context.png, c4-container.png
    │   ├── openapi.yaml
    │   ├── er-diagram.drawio, er-diagram.png
    │   ├── deployment.png
    │   └── auth-login.png, auth-refresh-logout.png
    ├── concurrency/
    │   ├── race-conditions.md
    │   └── race-note.png, race-refresh.png
    └── mcp/
        ├── README.md
        ├── tools.json, resources.json
        ├── traceability.md
        └── data-registry.md
```

## Архитектурные артефакты

### C4

[c4-diagram.drawio](docs/architecture/c4-diagram.drawio) – диаграммы System Context и Container.

![System Context](docs/architecture/c4-context.png)

### ER-диаграмма

[er-diagram.png](docs/architecture/er-diagram.png) (исходник – [er-diagram.drawio](docs/architecture/er-diagram.drawio)) – модель данных после ЛР2–ЛР5: сущности паспорта и таблицы токенов обновления, ключей идемпотентности, outbox, ленты и подтверждений синхронизации, попыток загрузки вложений.

![ER-диаграмма](docs/architecture/er-diagram.png)
