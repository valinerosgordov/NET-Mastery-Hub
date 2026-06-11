# EFCore — Entity Framework Core

> 7 файлов / 237 KB. EF Core deep dive: tracking, queries, migrations, patterns + сравнение с Dapper.

[← Главный README]() · [Полный INDEX]()

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Первый раз с EF | [`basics-tracking.md`]() |
| N+1 проблема? | [`queries-performance.md`]() |
| Связи one-to-many | [`relationships.md`]() |
| Когда EF, когда Dapper | [`dapper-comparison.md`]() |
| Нужен Repository? | [`patterns.md`]() |

---

## 📚 Все 7 файлов

### Core

| Файл | Описание |
|------|----------|
| [`basics-tracking.md`]() | DbContext, ChangeTracker, AsNoTracking ⭐ |
| [`queries-performance.md`]() | N+1, Include, projections ⭐ |
| [`relationships.md`]() | One-to-many, many-to-many, owned types |
| [`migrations.md`]() | EF migrations, scripts, rollback |

### Advanced

| Файл | Описание |
|------|----------|
| [`concurrency.md`]() | Optimistic locking, RowVersion |
| [`patterns.md`]() | Repository, UoW, Specification |
| [`dapper-comparison.md`]() | Dapper vs EF, hybrid patterns |

---

## 🔗 Связанные папки

- [`SQL/`](../SQL/) — что под капотом EF
- [`Architecture/cqrs-mediatr`]() — CQRS с EF
- [`Performance/lazy-eager-loading`]() — strategies
- [`Snippets/efcore-queries`]() — готовые запросы
