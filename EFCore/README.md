# EFCore — Entity Framework Core

> 13 файлов / ~420 KB. EF Core deep dive: tracking, queries, migrations, patterns + сравнение с Dapper.

[[README|← Главный README]] · [[INDEX|Полный INDEX]]

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Первый раз с EF | [[ef-basics|`Junior/ef-basics.md`]] → [[ef-crud-queries|`Junior/ef-crud-queries.md`]] |
| Как реально работает tracking | [[basics-tracking|`Senior/basics-tracking.md`]] |
| N+1 проблема? | [[queries-performance|`Senior/queries-performance.md`]] |
| Include vs split vs lazy | [[ef-loading-strategies|`Middle/ef-loading-strategies.md`]] |
| Связи one-to-many | [[relationships|`Senior/relationships.md`]] |
| Когда EF, когда Dapper | [[dapper-comparison|`Middle/dapper-comparison.md`]] |
| Нужен Repository? | [[ef-patterns|`Senior/ef-patterns.md`]] |

---

## 📚 Все 13 файлов

### 🌱 Junior

| Файл | Описание |
|------|----------|
| [[ef-basics|`ef-basics.md`]] | DbContext, DbSet, первая модель — вход в EF |
| [[ef-crud-queries|`ef-crud-queries.md`]] | CRUD-операции, базовые LINQ-запросы |

### 🌿 Middle

| Файл | Описание |
|------|----------|
| [[ef-loading-strategies|`ef-loading-strategies.md`]] | Eager / lazy / explicit loading, AsSplitQuery |
| [[ef-transactions-concurrency|`ef-transactions-concurrency.md`]] | Транзакции, SaveChanges, изоляция |
| [[ef-bulk-operations|`ef-bulk-operations.md`]] | ExecuteUpdate/Delete, bulk insert |
| [[ef-value-converters|`ef-value-converters.md`]] | Value converters, owned types, JSON columns |
| [[dapper-comparison|`dapper-comparison.md`]] | Dapper vs EF, hybrid patterns |

### 🏆 Senior

| Файл | Описание |
|------|----------|
| [[basics-tracking|`basics-tracking.md`]] | DbContext, ChangeTracker, AsNoTracking ⭐ |
| [[queries-performance|`queries-performance.md`]] | N+1, Include, projections ⭐ |
| [[relationships|`relationships.md`]] | One-to-many, many-to-many, owned types, TPH/TPT/TPC |
| [[migrations|`migrations.md`]] | EF migrations, scripts, rollback |
| [[concurrency|`concurrency.md`]] | Optimistic locking, RowVersion |
| [[ef-patterns|`ef-patterns.md`]] | Repository, UoW, Specification, Soft Delete, audit |

---

## 🔗 Связанные папки

- [`SQL/`](../SQL/) — что под капотом EF
- [[cqrs-mediatr|`Architecture/Senior/cqrs-mediatr.md`]] — CQRS с EF
- [[lazy-eager-loading|`Performance/Middle/lazy-eager-loading.md`]] — strategies
- [[efcore-queries|`Snippets/efcore-queries.md`]] — готовые запросы
