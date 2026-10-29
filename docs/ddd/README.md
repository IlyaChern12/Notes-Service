# DDD-модель Notes Service (ЛР2)

Предметная модель построена методами Domain-Driven Design, чтобы снизить риск потери заметок, неполной синхронизации и неявной перезаписи изменений, выявленных при [анализе отзывов](../analysis/user-reviews-analysis.md).

| Документ | Содержание |
|---|---|
| [glossary.md](glossary.md) | единый язык: 23 термина и 4 разрешённых конфликта |
| [context-map.md](context-map.md) | 4 ограниченных контекста, связи L1–L8, ACL, outbox |
| [aggregates.md](aggregates.md) | агрегаты Note, SyncSession, Attachment; правила N1–N5, S1–S5, A1–A5 |
| [scenarios.md](scenarios.md) | сценарии С1–С4, схема компонентов, карты потоков, 7 доменных событий |

Модель сохраняет решения архитектурного паспорта: модульный монолит, PostgreSQL, REST API, JWT, история версий и мягкое удаление.
