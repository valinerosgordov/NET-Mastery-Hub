# SQL — relational databases

> 8 файлов / ~215 KB. SQL fundamentals + index internals + PostgreSQL deep features + concurrency + safe migrations + security.

[← Главный README](../readme.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Junior, только начинаю | [`sql-basics.md`](Junior/sql-basics.md) |
| Slow queries, нужно оптимизировать | [`optimization.md`](Senior/optimization.md) → [`indexes-deep.md`](Middle/indexes-deep.md) |
| Работаю с PostgreSQL | [`postgresql-deep.md`](Senior/postgresql-deep.md) |
| Хранимки / триггеры / raw SQL | [`postgres-functions-triggers.md`](Senior/postgres-functions-triggers.md) |
| Конкурентность, блокировки, очереди задач | [`mvcc-and-locking.md`](Senior/mvcc-and-locking.md) |
| Меняю схему на проде без даунтайма | [`zero-downtime-migrations.md`](Senior/zero-downtime-migrations.md) |
| Безопасность: инъекции, роли, привилегии | [`sql-security.md`](Senior/sql-security.md) |
| Готовлюсь к Senior собесу | Все 8 файлов |

---

## 📚 Все 8 файлов

| Файл | Уровень | Описание |
|------|---------|----------|
| [`sql-basics.md`](Junior/sql-basics.md) | Junior/Middle | JOIN, transactions, isolation, ACID, normalization |
| [`indexes-deep.md`](Middle/indexes-deep.md) | Middle/Senior | B-tree, query plans, index types ⭐ |
| [`optimization.md`](Senior/optimization.md) | Senior | Query optimization, EXPLAIN, partitioning, PgBouncer |
| [`postgresql-deep.md`](Senior/postgresql-deep.md) | Senior | JSONB, RLS, pgvector, VACUUM, advanced PG features |
| [`postgres-functions-triggers.md`](Senior/postgres-functions-triggers.md) | Senior | PL/pgSQL функции, процедуры, триггеры, вызов из Npgsql/EF |
| [`mvcc-and-locking.md`](Senior/mvcc-and-locking.md) | Senior | MVCC, xmin/xmax, lock modes, SKIP LOCKED, UPSERT, LISTEN/NOTIFY ⭐ |
| [`zero-downtime-migrations.md`](Senior/zero-downtime-migrations.md) | Senior | Безопасный DDL, lock_timeout, CONCURRENTLY, expand-contract, EF Core ⭐ |
| [`sql-security.md`](Senior/sql-security.md) | Senior | SQL injection, roles/GRANT, least privilege, SECURITY DEFINER, TLS |

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
- [`EFCore/Middle/dapper-comparison.md`](../EFCore/Middle/dapper-comparison.md) — raw SQL alternatives
- [`Performance/`](../Performance/) — DB performance в общем контексте
