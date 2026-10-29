# Трассируемость требований MCP-интерфейса

## Матрица трассируемости

| Бизнес-требование | Сущность БД | Атрибуты | Инструмент / ресурс MCP |
|---|---|---|---|
| ТР-01. Пользователь находит свои заметки по словам, тегу и времени изменения | notes, tags, note_tags | notes.title, content, updated_at, is_deleted; tags.name | `search_notes` |
| ТР-02. Пользователь просматривает заметку с тегами и списком вложений | notes, tags, note_tags, attachments | notes.id, title, content, version, created_at, updated_at; tags.name; attachments.id, file_name, mime_type, size_bytes | `get_note`; `notes://{note_id}` |
| ТР-03. Ассистент создаёт заметку по просьбе пользователя | notes, idempotency_keys | notes.title, content, version = 1; idempotency_keys.key, request_hash | `create_note` |
| ТР-04. Ассистент дополняет заметку, не переписывая прежний текст | notes, note_versions | notes.content, version; note_versions.* | `append_to_note` |
| ТР-05. Правка не перезаписывает изменения с другого устройства (R6, R7) | notes, note_versions | notes.version, title, content; note_versions.version, changed_via | `update_note` |
| ТР-06. Пользователь размечает заметки тегами | tags, note_tags | tags.name; note_tags.note_id, tag_id | `set_note_tags`; `tags://me` |
| ТР-07. Удалённая заметка исчезает со всех устройств и не «воскресает» | notes | notes.is_deleted, deleted_at, version | `delete_note` |
| ТР-08. Пользователь видит историю и автора правок | note_versions | note_versions.version, title, created_at, changed_via | `list_note_versions`; `notes://{note_id}/versions/{version}` |
| ТР-09. Пользователь возвращает прежнюю версию (R4, R6) | notes, note_versions | note_versions.title, content; notes.version | `restore_note_version` |
| ТР-10. Ассистент сообщает о вложениях без передачи файла (R5, R9) | attachments | attachments.id, note_id, file_name, mime_type, size_bytes, created_at | `attachments://{attachment_id}`; `get_note` |
| ТР-11. Ассистент объясняет состояние синхронизации устройств (R7) | devices, sync_sessions | devices.platform, last_sync_at; sync_sessions.status, synchronized_at | `sync://devices` |
| ТР-12. Повтор вызова после сбоя не создаёт второй эффект | idempotency_keys | user_id, key, request_hash, response, created_at | `create_note`, `append_to_note`, `update_note`, `delete_note`, `restore_note_version` |
| ТР-13. Ассистент получает только данные пользователя, от имени которого действует | users, notes | users.id = sub токена; notes.user_id | все инструменты и ресурсы |

## Обратная проверка: атрибуты, не раскрываемые через MCP

| Атрибут | Причина |
|---|---|
| users.id, notes.user_id | субъект определяется по токену |
| users.email, users.name | персональные данные, не нужны ни одной задаче |
| users.password_hash, auth_refresh_tokens.* | аутентификационные секреты |
| devices.name, devices.id, note_versions.device_id | название может содержать имя владельца; заменено меткой |
| attachments.storage_path, содержимое файла | адрес хранилища скрыт ACL; файл не передаётся в контекст модели |
| sync_sessions.since, changes_count, checkpoint | служебные данные протокола синхронизации |
| idempotency_keys.request_hash, response | служебные данные повторов |
| outbox, processed_events, change_feed, upload_attempts | внутренняя инфраструктура |

## Соответствие полей схем модели данных

| Поле схемы | Тип в схеме | Атрибут и тип в БД | Вывод |
|---|---|---|---|
| note_id | string uuid | notes.id UUID | совпадает |
| title | string 1…255 | notes.title VARCHAR(255) | совпадает |
| content, text | string ≤ 50 000, ≤ 10 000 | notes.content TEXT | схема строже |
| expected_version, version, before_version | integer ≥ 1 | notes.version, note_versions.version INTEGER | совпадает |
| tags[], tag | string 1…50 | tags.name VARCHAR(50) | совпадает |
| updated_after | string date-time | notes.updated_at TIMESTAMPTZ | совпадает |
| idempotency_key | string uuid | idempotency_keys.key UUID | совпадает |
| query | string 2…200 | полнотекстовый индекс по title, content | параметр запроса |
| sort, limit, cursor | enum, integer, string | – | параметр протокола |

## Изменения модели данных, внесённые для MCP

- `note_versions.changed_via VARCHAR(10) NOT NULL DEFAULT 'device'`, `CHECK (changed_via IN ('device', 'mcp'))`;
- уникальность тегов без учёта регистра: `UNIQUE (user_id, lower(name))` (миграция expand – migrate – contract с объединением совпадающих тегов);
- индекс `GIN (to_tsvector('russian', title || ' ' || content)) WHERE NOT is_deleted`;
- новых таблиц не требуется: `idempotency_keys` и `outbox` введены в ЛР4.
