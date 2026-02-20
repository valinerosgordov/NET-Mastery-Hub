# Оптимизация SQL

## Индексы

**Clustered** — порядок данных на диске (один на таблицу). **Non-clustered** — отдельная структура, указатели на строки.

**Когда**: колонки в WHERE, JOIN, ORDER BY. **Составные** — порядок колонок важен. **Covering index** — Include всех нужных колонок, index-only scan.

**Filtered index** — WHERE IsDeleted = 0. Меньше размер, быстрее для подмножества данных.

---

## Execution Plan

EXPLAIN (PostgreSQL), EXPLAIN (MySQL), Actual Execution Plan (SQL Server). Искать: Seq Scan → заменить на Index Scan, высокий cost, Nested Loops с большими таблицами.

---

## Паттерны

- **Пагинация**: OFFSET/LIMIT — медленно на больших offset. Keyset pagination (WHERE Id > @lastId) — стабильнее.
- **N+1** — один запрос с JOIN вместо N отдельных.
- **Bulk operations** — COPY, BULK INSERT вместо множества INSERT.

---

## См. также

- [[Topics/Performance/dotnet-performance|.NET Performance]]
- [[Topics/NetQuestions150/efcore/05-performance|EF Core Performance]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
