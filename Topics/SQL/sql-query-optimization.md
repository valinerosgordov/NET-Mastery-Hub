# Оптимизация SQL

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
