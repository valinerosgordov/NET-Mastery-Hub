# EFCore — Entity Framework Core

> 13 файлов / ~420 KB. EF Core deep dive: tracking, queries, migrations, patterns + сравнение с Dapper.

[← Главный README](../readme.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Первый раз с EF | [`Junior/ef-basics.md`](Junior/ef-basics.md) → [`Junior/ef-crud-queries.md`](Junior/ef-crud-queries.md) |
| Как реально работает tracking | [`Senior/basics-tracking.md`](Senior/basics-tracking.md) |
| N+1 проблема? | [`Senior/queries-performance.md`](Senior/queries-performance.md) |
| Include vs split vs lazy | [`Middle/ef-loading-strategies.md`](Middle/ef-loading-strategies.md) |
| Связи one-to-many | [`Senior/relationships.md`](Senior/relationships.md) |
| Когда EF, когда Dapper | [`Middle/dapper-comparison.md`](Middle/dapper-comparison.md) |
| Нужен Repository? | [`Senior/ef-patterns.md`](Senior/ef-patterns.md) |

---

## 📚 Все 13 файлов

### 🌱 Junior

| Файл | Описание |
|------|----------|
| [`ef-basics.md`](Junior/ef-basics.md) | DbContext, DbSet, первая модель — вход в EF |
| [`ef-crud-queries.md`](Junior/ef-crud-queries.md) | CRUD-операции, базовые LINQ-запросы |

### 🌿 Middle

| Файл | Описание |
|------|----------|
| [`ef-loading-strategies.md`](Middle/ef-loading-strategies.md) | Eager / lazy / explicit loading, AsSplitQuery |
| [`ef-transactions-concurrency.md`](Middle/ef-transactions-concurrency.md) | Транзакции, SaveChanges, изоляция |
| [`ef-bulk-operations.md`](Middle/ef-bulk-operations.md) | ExecuteUpdate/Delete, bulk insert |
| [`ef-value-converters.md`](Middle/ef-value-converters.md) | Value converters, owned types, JSON columns |
| [`dapper-comparison.md`](Middle/dapper-comparison.md) | Dapper vs EF, hybrid patterns |

### 🏆 Senior

| Файл | Описание |
|------|----------|
| [`basics-tracking.md`](Senior/basics-tracking.md) | DbContext, ChangeTracker, AsNoTracking ⭐ |
| [`queries-performance.md`](Senior/queries-performance.md) | N+1, Include, projections ⭐ |
| [`relationships.md`](Senior/relationships.md) | One-to-many, many-to-many, owned types, TPH/TPT/TPC |
| [`migrations.md`](Senior/migrations.md) | EF migrations, scripts, rollback |
| [`concurrency.md`](Senior/concurrency.md) | Optimistic locking, RowVersion |
| [`ef-patterns.md`](Senior/ef-patterns.md) | Repository, UoW, Specification, Soft Delete, audit |

---

## 🔗 Связанные папки

- [`SQL/`](../SQL/) — что под капотом EF
- [`Architecture/Senior/cqrs-mediatr.md`](../Architecture/Senior/cqrs-mediatr.md) — CQRS с EF
- [`Performance/Middle/lazy-eager-loading.md`](../Performance/Middle/lazy-eager-loading.md) — strategies
- [`Snippets/efcore-queries.md`](../Snippets/efcore-queries.md) — готовые запросы
