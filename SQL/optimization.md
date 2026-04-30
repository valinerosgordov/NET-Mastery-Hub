---
tags: [sql, indexes, query-plans, optimization]
level: Senior
---

# Оптимизация SQL

## Что это, зачем и когда

### Что такое оптимизация SQL?
**Заставить запросы к БД работать быстрее** — через правильные индексы, структуру запросов и понимание execution plan.

**Аналогия:** Книга без оглавления — чтобы найти главу, листаешь все 500 страниц (Full Table Scan). С оглавлением (индексом) — сразу на нужную страницу.

### Зачем?

| Без оптимизации | С оптимизацией |
|-----------------|---------------|
| `SELECT * FROM Orders WHERE CustomerId = X` → 5 сек (сканирует 10 млн строк) | Индекс на `CustomerId` → 2 мс |
| Запрос из 10 JOIN → timeout | Правильные индексы + execution plan → 50 мс |
| N+1: 1000 запросов к БД на одну страницу | Include / проекция → 1-2 запроса |
| «БД тормозит, давайте купим сервер помощнее» | Добавили 3 индекса — скорость x100 |

### Когда заниматься?

| Ситуация | Действие |
|----------|----------|
| Проектируешь схему | Индексы на FK, колонки в WHERE/ORDER BY |
| Запрос > 100 мс | Посмотри execution plan (`EXPLAIN ANALYZE`) |
| Таблица > 100K строк | Проверь наличие нужных индексов |
| N+1 в логах | Добавь `Include()` или проекцию |
| **НЕ нужно:** маленькая таблица (< 1000 строк) | Full scan быстрее, чем использование индекса |

---

## Индексы

**Clustered index** — определяет физический порядок данных на диске. Один на таблицу. По умолчанию — Primary Key. Данные хранятся в листьях B-tree. Range scan — последовательное чтение (быстро).

**Non-clustered index** — отдельная B-tree структура. Листья содержат указатели на строки (RID или clustered key). Может быть несколько на таблицу. Lookup в основную таблицу — дополнительный I/O.

### Когда создавать

- Колонки в `WHERE`, `JOIN ON`, `ORDER BY`, `GROUP BY`
- Колонки с высокой селективностью (много уникальных значений)
- FK-колонки — JOIN по ним частый
- **Не** создавать на колонках с малой селективностью (`bool`, `status` с 3 значениями) — full scan может быть быстрее

### Составные индексы — leftmost prefix rule

Индекс `(A, B, C)` работает для:
- `WHERE A = ?` ✓
- `WHERE A = ? AND B = ?` ✓
- `WHERE A = ? AND B = ? AND C = ?` ✓
- `ORDER BY A, B` ✓

Не работает для:
- `WHERE B = ?` ✗ (без A)
- `WHERE C = ?` ✗ (без A и B)
- `ORDER BY B, A` ✗ (другой порядок)

```sql
-- Составной индекс для частого запроса
CREATE INDEX IX_Orders_CustomerId_CreatedAt
ON Orders (CustomerId, CreatedAt DESC);

-- Использует индекс полностью
SELECT * FROM Orders
WHERE CustomerId = @id ORDER BY CreatedAt DESC;
```

### Covering Index (Include)

Добавляет колонки в листья индекса без включения в ключ. Позволяет **index-only scan** — нет обращения к основной таблице.

```sql
CREATE INDEX IX_Orders_Status_Include
ON Orders (Status)
INCLUDE (CustomerId, Total, CreatedAt);

-- Index-only scan — быстро
SELECT CustomerId, Total, CreatedAt
FROM Orders WHERE Status = 'Active';
```

**Нюанс:** Include-колонки увеличивают размер индекса. Не добавлять всё подряд.

### Filtered Index

Индекс с условием `WHERE`. Меньше размер, быстрее обновление.

```sql
CREATE INDEX IX_Orders_Active
ON Orders (CreatedAt)
WHERE IsDeleted = 0 AND Status != 'Cancelled';
```

**Нюанс:** запрос должен содержать то же условие, иначе оптимизатор не выберет filtered index.

---

## Execution Plan

### Как читать

- **PostgreSQL:** `EXPLAIN ANALYZE SELECT ...` — реальный план с фактическим временем
- **SQL Server:** `SET STATISTICS IO ON` + Actual Execution Plan (Ctrl+M в SSMS)
- **MySQL:** `EXPLAIN SELECT ...`

### На что смотреть

| Операция | Проблема | Решение |
|----------|----------|---------|
| Seq Scan / Table Scan | Полный просмотр таблицы | Добавить индекс |
| Nested Loop с большими таблицами | O(n×m) | Hash Join, индексы на JOIN |
| Sort с высоким cost | Сортировка на диске | Индекс с ORDER BY |
| Key Lookup / Bookmark Lookup | Лишний I/O после index scan | Covering index (INCLUDE) |
| Estimated vs Actual rows расходятся | Устаревшая статистика | `ANALYZE` / `UPDATE STATISTICS` |

### Обновление статистики

```sql
-- PostgreSQL
ANALYZE orders;

-- SQL Server
UPDATE STATISTICS Orders;
```

---

### Join-алгоритмы — когда что выбирает оптимизатор

SQL-движок выбирает один из трёх алгоритмов соединения таблиц в зависимости от размеров, индексов и доступной памяти. Понимание, что именно использует твой запрос — ключ к оптимизации.

#### Nested Loops

```
for each row_a in A:
    for each row_b in B where row_b.key == row_a.key:
        emit(row_a, row_b)
```

**Работает отлично:** когда одна таблица маленькая (десятки-сотни строк), у второй есть индекс по join-ключу. Поиск по индексу занимает O(log N), общая сложность O(M × log N).

**Плохо:** обе таблицы крупные и/или нет индекса — сложность вырождается в O(M × N).

#### Hash Join

1. **Build phase** — строим hash-таблицу по ключу в памяти из меньшей таблицы
2. **Probe phase** — пробегаемся по второй таблице, ищем через hash за O(1)

**Работает отлично:** обе таблицы большие, индексов нет, равнозначное сравнение (`=`).

**Цена:** расход памяти. Если hash-таблица не влезает в `work_mem` (PostgreSQL) — spill на диск, резкое падение производительности.

**Не работает:** с `<`, `>`, `BETWEEN` — только равенство.

#### Merge Join

Обе таблицы отсортированы по join-ключу → идём двумя указателями.

**Работает отлично:** обе стороны уже отсортированы (индексы покрывают ORDER BY). Типично для `INNER JOIN` между таблицами с clustered index по join-ключу.

**Цена:** если сортировки нет — sort сам по себе O(N log N), теряем преимущество.

#### Как диагностировать

**PostgreSQL:**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > '2026-01-01';

-- В плане ищем:
-- "Nested Loop"  → N × M, хорошо только с индексом
-- "Hash Join"    → память съедается на build-фазе
-- "Merge Join"   → обе стороны отсортированы
```

**SQL Server (SSMS):** Ctrl+M → Include Actual Execution Plan → видны иконки Nested Loops / Hash Match / Merge Join.

#### Правила толщиной в 3 пункта

| Ситуация | Ожидаемый выбор | Что проверить |
|----------|-----------------|---------------|
| Маленькое × большое + индекс на FK | Nested Loops | `EXPLAIN` показывает `Index Scan` на большой таблице |
| Большое × большое, нет индекса | Hash Join | `work_mem` достаточно, чтобы build table не ушёл на диск |
| Обе большие, обе отсортированы | Merge Join | Индексы покрывают ORDER BY |

#### Типичные проблемы

- **Hash Spill на диск** — в Postgres план покажет `Hash Batches: 4 Memory Usage: 1024kB Disk: 4096kB`. Увеличь `work_mem` (локально для запроса: `SET LOCAL work_mem = '256MB'`) либо добавь индекс для Nested Loops.
- **Nested Loops без индекса на большой таблице** — O(N×M) катастрофа. План покажет `Seq Scan` внутри цикла. Лечится индексом.
- **Оптимизатор выбрал не то** — устаревшая статистика. `ANALYZE table_name` обновляет распределения для планировщика.

> [!question]- **Интервью: В каких случаях Hash Join медленнее Nested Loops?**
> Когда одна таблица маленькая (скажем, 100 строк), у второй есть B-Tree индекс по join-ключу. Nested Loops делает 100 индексных lookup-ов — по O(log N) каждый. Hash Join вынужден сначала построить hash-таблицу (даже маленькую), потом пробежаться по большой — это больше работы. Плюс если памяти мало и hash уходит на диск (spill), Hash Join деградирует катастрофически. Мораль: маленькое × большое + индекс → Nested Loops. Большое × большое без индекса или неравенство → Hash Join. Обе отсортированы → Merge Join.

---

## Паттерны оптимизации

### Пагинация

**OFFSET/LIMIT** — медленно при большом offset (БД читает и отбрасывает строки).

```sql
-- Плохо при offset > 10000
SELECT * FROM Orders ORDER BY Id OFFSET 100000 LIMIT 20;

-- Keyset pagination — стабильная производительность
SELECT * FROM Orders
WHERE Id > @lastId ORDER BY Id LIMIT 20;
```

**Нюанс:** keyset требует уникальный и стабильный ключ сортировки. Для составной сортировки: `WHERE (CreatedAt, Id) > (@lastDate, @lastId)`.

### N+1 проблема

```sql
-- Плохо: 1 + N запросов
SELECT * FROM Orders;                              -- 1
SELECT * FROM OrderItems WHERE OrderId = @id;      -- × N

-- Хорошо: один запрос
SELECT o.*, oi.*
FROM Orders o
LEFT JOIN OrderItems oi ON o.Id = oi.OrderId
WHERE o.CustomerId = @customerId;
```

В EF Core: `.Include(o => o.Items)` или проекция через `.Select()`.

### Bulk операции

```sql
-- Плохо: 10000 INSERT по одному
INSERT INTO Logs (Message) VALUES ('log1');  -- × 10000

-- Хорошо: batch insert
INSERT INTO Logs (Message) VALUES ('log1'), ('log2'), ('log3'), ...;

-- PostgreSQL COPY — самый быстрый
COPY Logs (Message) FROM STDIN;
```

В .NET: `EFCore.BulkExtensions`, `Npgsql COPY`, `SqlBulkCopy`.

### SELECT * vs конкретные колонки

```sql
-- Плохо: загружает все колонки (включая BLOB/TEXT)
SELECT * FROM Users;

-- Хорошо: только нужные
SELECT Id, Name, Email FROM Users;
```

### Параметризация

```sql
-- Плохо: конкатенация → SQL injection + нет кэша плана
"SELECT * FROM Users WHERE Name = '" + name + "'"

-- Хорошо: параметры → безопасность + кэш плана
SELECT * FROM Users WHERE Name = @name
```

---

## Транзакции и блокировки

### Уровни изоляции

| Уровень | Dirty Read | Non-repeatable | Phantom |
|---------|-----------|----------------|---------|
| Read Uncommitted | Да | Да | Да |
| Read Committed | Нет | Да | Да |
| Repeatable Read | Нет | Нет | Да |
| Serializable | Нет | Нет | Нет |
| Snapshot (MVCC) | Нет | Нет | Нет |

**Read Committed** — default в PostgreSQL и SQL Server. Достаточно для большинства случаев.

**Snapshot / MVCC** — читатели не блокируют писателей. PostgreSQL — MVCC по умолчанию.

### Deadlock

Два процесса ждут ресурсы друг друга. Защита:
- Фиксированный порядок блокировки ресурсов
- Короткие транзакции
- `SET LOCK_TIMEOUT` — не ждать бесконечно
- Retry логика при deadlock exception

---

## Мониторинг в production

```sql
-- PostgreSQL: pg_stat_statements — топ медленных запросов
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;

-- SQL Server: DMV
SELECT TOP 10
    qs.total_elapsed_time / qs.execution_count AS avg_time,
    qs.execution_count,
    SUBSTRING(st.text, qs.statement_start_offset/2 + 1, 100)
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_time DESC;
```

**postgresql.conf:** `log_min_duration_statement = 200` — логировать запросы дольше 200 мс.

---

## См. также

- [[Topics/Performance/dotnet-performance|.NET Performance]]
- [[Topics/NetQuestions150/efcore/05-performance|EF Core Performance]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
