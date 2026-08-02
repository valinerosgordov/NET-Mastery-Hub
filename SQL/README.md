# SQL — relational databases

> 9 файлов / ~253 KB. SQL fundamentals + index internals + PostgreSQL deep features + concurrency + safe migrations + security + EAV indexing.

[[README|← Главный README]] · [[INDEX|Полный INDEX]]

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Junior, только начинаю | [[sql-basics|`sql-basics.md`]] |
| Slow queries, нужно оптимизировать | [[optimization|`optimization.md`]] → [[indexes-deep|`indexes-deep.md`]] |
| Работаю с PostgreSQL | [[postgresql-deep|`postgresql-deep.md`]] |
| Хранимки / триггеры / raw SQL | [[postgres-functions-triggers|`postgres-functions-triggers.md`]] |
| Конкурентность, блокировки, очереди задач | [[mvcc-and-locking|`mvcc-and-locking.md`]] |
| Меняю схему на проде без даунтайма | [[zero-downtime-migrations|`zero-downtime-migrations.md`]] |
| Безопасность: инъекции, роли, привилегии | [[sql-security|`sql-security.md`]] |
| Гибкая схема / EAV, индексы под неё | [[eav-flexible-store-indexing|`eav-flexible-store-indexing.md`]] |
| Готовлюсь к Senior собесу | Все 9 файлов |

---

## 📚 Все 9 файлов

| Файл | Уровень | Описание |
|------|---------|----------|
| [[sql-basics|`sql-basics.md`]] | Junior/Middle | JOIN, transactions, isolation, ACID, normalization |
| [[indexes-deep|`indexes-deep.md`]] | Middle/Senior | B-tree, query plans, index types ⭐ |
| [[optimization|`optimization.md`]] | Senior | Query optimization, EXPLAIN, partitioning, PgBouncer |
| [[postgresql-deep|`postgresql-deep.md`]] | Senior | JSONB, RLS, pgvector, VACUUM, advanced PG features |
| [[postgres-functions-triggers|`postgres-functions-triggers.md`]] | Senior | PL/pgSQL функции, процедуры, триггеры, вызов из Npgsql/EF |
| [[mvcc-and-locking|`mvcc-and-locking.md`]] | Senior | MVCC, xmin/xmax, lock modes, SKIP LOCKED, UPSERT, LISTEN/NOTIFY ⭐ |
| [[zero-downtime-migrations|`zero-downtime-migrations.md`]] | Senior | Безопасный DDL, lock_timeout, CONCURRENTLY, expand-contract, EF Core ⭐ |
| [[sql-security|`sql-security.md`]] | Senior | SQL injection, roles/GRANT, least privilege, SECURITY DEFINER, TLS |
| [[eav-flexible-store-indexing|`eav-flexible-store-indexing.md`]] | Senior | EAV-хранилище: фиксированный набор индексов, covering/partial, type-segregated UNIQUE |

---

## 🗺️ Как связаны Senior-файлы

- **MVCC** ([[mvcc-and-locking]]) объясняет dead tuples → **VACUUM/bloat** ([[postgresql-deep]])
- **Lock modes** ([[mvcc-and-locking]]) — фундамент для **безопасного DDL** ([[zero-downtime-migrations]])
- **Триггеры** ([[postgres-functions-triggers]]) часто шлют **LISTEN/NOTIFY** ([[mvcc-and-locking]]) и используются для **аудита** ([[sql-security]])
- **RLS** ([[postgresql-deep]]) + **роли** ([[sql-security]]) = multi-tenant безопасность
- **Backfill** ([[zero-downtime-migrations]]) использует **процедуры с COMMIT** ([[postgres-functions-triggers]])

---

## 🔗 Связанные папки

- [`EFCore/`](../EFCore/) — ORM поверх SQL
- [[dapper-comparison|`EFCore/Middle/dapper-comparison.md`]] — raw SQL alternatives
- [`Performance/`](../Performance/) — DB performance в общем контексте
