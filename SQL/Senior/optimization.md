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


---

## Partitioning — горизонтальное разделение больших таблиц

### Зачем partitioning

```
Table: Orders, 500M rows, 800 GB
- Index seek работает, но maintenance painful (index rebuild часы)
- Backup долгий
- Старые данные редко используются, но занимают место
- Statistics устаревают на больших таблицах
```

Partitioning делит таблицу на **независимые pieces** (partitions) — БД управляет ими отдельно, но запросы видят как одну таблицу.

### Виды partitioning

```
Range partitioning — по диапазону (типично по дате):
  partition_2024 — orders за 2024
  partition_2025 — orders за 2025
  partition_2026 — orders за 2026
  
List partitioning — по конкретным значениям:
  partition_eu  — country IN ('DE', 'FR', 'IT', ...)
  partition_us  — country IN ('US', 'CA')
  partition_asia — country IN ('JP', 'CN', 'KR')
  
Hash partitioning — равномерно по hash:
  partition_0 — hash(customer_id) % 4 == 0
  partition_1 — hash(customer_id) % 4 == 1
  ... etc
```

### PostgreSQL — declarative partitioning

```sql
-- Range partitioning по дате
CREATE TABLE orders (
    id BIGSERIAL,
    customer_id INT,
    created_at TIMESTAMPTZ NOT NULL,
    total NUMERIC(18, 2),
    PRIMARY KEY (id, created_at)   -- partition key должен быть в PK
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE orders_2026 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- INSERT автоматически routes в нужную partition
INSERT INTO orders (customer_id, created_at, total)
VALUES (123, '2026-05-10', 99.99);   -- → orders_2026
```

### Hash partitioning для load balancing

```sql
CREATE TABLE events (
    id BIGSERIAL,
    user_id INT,
    event_type TEXT,
    PRIMARY KEY (id, user_id)
) PARTITION BY HASH (user_id);

CREATE TABLE events_p0 PARTITION OF events FOR VALUES WITH (modulus 4, remainder 0);
CREATE TABLE events_p1 PARTITION OF events FOR VALUES WITH (modulus 4, remainder 1);
CREATE TABLE events_p2 PARTITION OF events FOR VALUES WITH (modulus 4, remainder 2);
CREATE TABLE events_p3 PARTITION OF events FOR VALUES WITH (modulus 4, remainder 3);

-- Hash(user_id) routes evenly across partitions
```

### Partition pruning — performance benefit

```sql
EXPLAIN SELECT * FROM orders 
WHERE created_at >= '2026-01-01' AND created_at < '2026-06-01';

-- Plan показывает что только orders_2026 сканируется!
-- orders_2024 и orders_2025 skipped → 3x меньше работы
```

Это **partition pruning**. Critical для performance — без него benefit от partitioning минимальный.

### Operations на partitions

```sql
-- Drop старых данных = drop partition (instant!)
DROP TABLE orders_2024;
-- vs DELETE FROM orders WHERE created_at < '2025-01-01' (часы для 100M rows)

-- Detach partition (для archiving)
ALTER TABLE orders DETACH PARTITION orders_2024;
-- orders_2024 становится отдельной таблицей, можно move на slower storage

-- Attach новой partition
CREATE TABLE orders_2027 (...) PARTITION OF orders FOR VALUES FROM ('2027-01-01') TO ('2028-01-01');
```

### Когда partitioning

```
✅ Use cases:
- Большие таблицы (>100M rows)
- Time-series data (logs, events, orders)
- Data lifecycle (drop старых месяцев)
- Multi-tenant (partition by tenant_id)
- Parallel queries по partitions

❌ Не use:
- Маленькие таблицы (<10M rows) — overhead
- Queries без partition key в WHERE
- Сложные cross-partition joins
- Non-uniform data distribution (skewed hash)
```

### Common pitfalls

- **Partition key в WHERE обязателен** для pruning. `SELECT WHERE customer_id = X` без `created_at` → full scan всех partitions.
- **Foreign keys между partitioned tables limited** в PostgreSQL.
- **Indexes на partition уровне** — каждая partition имеет свой index.
- **Statistics per partition** — `ANALYZE partition_name` отдельно.

> [!question]- **Интервью: когда нужно partitioning?**
> Большие таблицы (>100M rows) с предсказуемой query pattern по partition key. **Use cases**: time-series (orders, logs, events) — partition by date; multi-tenant — partition by tenant_id; load balancing — hash partition. **Benefits**: 1) **Partition pruning** — query touches только нужные partitions. 2) **Maintenance** — drop старых partition вместо DELETE миллионов rows. 3) **Parallel scans** по partitions. **Critical**: partition key должен быть в WHERE clause для pruning. **Trade-offs**: overhead для small tables, complex для cross-partition joins.

---

## Materialized Views — pre-computed aggregations

### Что такое materialized view

Обычный view = stored query. Materialized view = **stored query + cached result**. Result обновляется через REFRESH (manual или automatic).

```sql
-- Regular view — recomputes каждый раз
CREATE VIEW order_summary AS
SELECT 
    customer_id,
    COUNT(*) as order_count,
    SUM(total) as total_revenue
FROM orders
GROUP BY customer_id;

-- Materialized view — stored result
CREATE MATERIALIZED VIEW order_summary_mv AS
SELECT 
    customer_id,
    COUNT(*) as order_count,
    SUM(total) as total_revenue
FROM orders
GROUP BY customer_id;

-- Use as table
SELECT * FROM order_summary_mv WHERE customer_id = 123;   -- O(log N) с индексом

-- Refresh когда данные изменились
REFRESH MATERIALIZED VIEW order_summary_mv;

-- Concurrent refresh (без блокировок)
REFRESH MATERIALIZED VIEW CONCURRENTLY order_summary_mv;   -- требует UNIQUE INDEX
```

### Когда materialized view

```
✅ Use cases:
- Heavy aggregations часто читаемые
- Dashboard queries
- Reports с pre-computed metrics
- Read-heavy workload (reads >> writes)
- Slow queries which можно cache

❌ Не use:
- Real-time данные (mv stale)
- Write-heavy workload
- Маленькие aggregations (recompute fast)
```

### SQL Server — Indexed Views

SQL Server не имеет materialized views, но имеет **Indexed Views** (similar):

```sql
CREATE VIEW order_summary
WITH SCHEMABINDING
AS
SELECT 
    customer_id,
    COUNT_BIG(*) as order_count,
    SUM(total) as total_revenue
FROM dbo.orders
GROUP BY customer_id;

-- Создание indexed view = create unique clustered index
CREATE UNIQUE CLUSTERED INDEX IX_order_summary 
ON order_summary (customer_id);

-- Индекс автоматически updates при INSERT/UPDATE/DELETE на orders
```

В отличие от PostgreSQL — SQL Server indexed view auto-refresh при изменениях, но имеет много restrictions (no LEFT JOIN, no subqueries, etc).

### Refresh strategies

```sql
-- 1. Manual — ad-hoc
REFRESH MATERIALIZED VIEW order_summary_mv;

-- 2. Scheduled — pg_cron extension
SELECT cron.schedule(
    'refresh-order-summary',
    '*/15 * * * *',   -- каждые 15 минут
    'REFRESH MATERIALIZED VIEW CONCURRENTLY order_summary_mv'
);

-- 3. Triggered — refresh after batch of changes
CREATE OR REPLACE FUNCTION refresh_summary() RETURNS TRIGGER AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY order_summary_mv;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER on_orders_change
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH STATEMENT EXECUTE FUNCTION refresh_summary();
-- ⚠️ Может быть slow если orders updates часто
```

### Trade-offs

```
✅ MV plus:
- Fast reads (pre-computed)
- Indexable
- Reduces load on base tables

❌ Cons:
- Storage overhead
- Stale data между refreshes
- Refresh cost (full vs incremental)
- Can't update MV directly
```

---

## CTE — Common Table Expressions

### Базовый CTE

```sql
-- Без CTE — repeated subquery
SELECT customer_id, count
FROM (SELECT customer_id, COUNT(*) as count FROM orders GROUP BY customer_id) sub
WHERE count > 10
ORDER BY count DESC;

-- С CTE — readable + reusable
WITH order_counts AS (
    SELECT customer_id, COUNT(*) as count
    FROM orders
    GROUP BY customer_id
)
SELECT * FROM order_counts WHERE count > 10 ORDER BY count DESC;
```

### Multiple CTEs

```sql
WITH 
recent_orders AS (
    SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '30 days'
),
top_customers AS (
    SELECT customer_id, COUNT(*) as order_count, SUM(total) as revenue
    FROM recent_orders
    GROUP BY customer_id
    ORDER BY revenue DESC
    LIMIT 10
)
SELECT 
    tc.customer_id,
    tc.order_count,
    tc.revenue,
    c.name,
    c.email
FROM top_customers tc
JOIN customers c ON c.id = tc.customer_id;
```

### Recursive CTE — для иерархий и графов

```sql
-- Employees + manager hierarchy
WITH RECURSIVE employee_tree AS (
    -- Base case: top-level managers (no boss)
    SELECT id, name, manager_id, 0 as level, name::text as path
    FROM employees
    WHERE manager_id IS NULL
    
    UNION ALL
    
    -- Recursive case: добавить subordinates
    SELECT e.id, e.name, e.manager_id, et.level + 1, et.path || ' > ' || e.name
    FROM employees e
    JOIN employee_tree et ON e.manager_id = et.id
)
SELECT * FROM employee_tree ORDER BY path;
```

```
Result:
id | name      | level | path
1  | CEO       | 0     | CEO
2  | VP Sales  | 1     | CEO > VP Sales
3  | VP Eng    | 1     | CEO > VP Eng
4  | Manager   | 2     | CEO > VP Sales > Manager
5  | Engineer  | 2     | CEO > VP Eng > Engineer
```

### Recursive use cases

- Organizational hierarchies
- Categories tree (e-commerce)
- Comment threads (parent_id)
- Bill of materials (parts containing parts)
- Network paths

### Performance — CTE materialized vs inlined

PostgreSQL 12+ — CTE по default **inlined** (treated как subquery), оптимизатор может рестрюктурировать.

```sql
WITH MATERIALIZED top_customers AS (...)   -- force materialization
WITH NOT MATERIALIZED top_customers AS (...)   -- force inline (default)
```

Materialize когда:
- CTE используется multiple раз
- CTE expensive и result reusable

---

## Window Functions

### Что такое

Применяют функции к **окну** rows без collapsing их в группы (как GROUP BY).

```sql
-- Без window function — нужны subqueries
SELECT 
    o.id,
    o.customer_id,
    o.total,
    (SELECT AVG(total) FROM orders WHERE customer_id = o.customer_id) as customer_avg
FROM orders o;

-- С window function — readable + fast
SELECT 
    id,
    customer_id,
    total,
    AVG(total) OVER (PARTITION BY customer_id) as customer_avg
FROM orders;
```

### Common window functions

```sql
-- ROW_NUMBER — уникальный rank
SELECT 
    customer_id,
    total,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total DESC) as rank
FROM orders;
-- Top order per customer = WHERE rank = 1

-- RANK / DENSE_RANK — учитывает ties
SELECT 
    customer_id,
    total,
    RANK() OVER (ORDER BY total DESC) as rank,
    DENSE_RANK() OVER (ORDER BY total DESC) as dense_rank
FROM orders;
-- RANK: 1, 2, 2, 4, 5 (ties skip numbers)
-- DENSE_RANK: 1, 2, 2, 3, 4 (ties don't skip)

-- LAG / LEAD — predecessor / successor row
SELECT 
    id,
    created_at,
    total,
    LAG(total) OVER (PARTITION BY customer_id ORDER BY created_at) as prev_total,
    LEAD(total) OVER (PARTITION BY customer_id ORDER BY created_at) as next_total
FROM orders;
-- Compare current vs previous order

-- Running totals
SELECT 
    id,
    created_at,
    total,
    SUM(total) OVER (
        PARTITION BY customer_id 
        ORDER BY created_at 
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as running_total
FROM orders;

-- Moving average
SELECT 
    created_at,
    daily_total,
    AVG(daily_total) OVER (
        ORDER BY created_at
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7days
FROM daily_sales;
```

### Top-N per group pattern

```sql
-- Top 3 orders per customer
WITH ranked AS (
    SELECT 
        customer_id,
        id,
        total,
        ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total DESC) as rn
    FROM orders
)
SELECT customer_id, id, total
FROM ranked
WHERE rn <= 3;
```

### Use cases

- Pagination с stable rankings
- Time series analytics (running totals, moving averages)
- Comparative metrics (current vs previous period)
- Top-N per group без window function = nightmare query

---

## JSON queries

### PostgreSQL JSONB

```sql
-- Schema
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    metadata JSONB
);

INSERT INTO products (name, metadata) VALUES 
('Phone', '{"brand": "Samsung", "specs": {"ram": "8GB", "storage": "128GB"}, "tags": ["mobile", "5g"]}');

-- Query JSON properties
SELECT * FROM products WHERE metadata->>'brand' = 'Samsung';

-- Nested
SELECT * FROM products WHERE metadata->'specs'->>'ram' = '8GB';

-- Array contains
SELECT * FROM products WHERE metadata->'tags' ? '5g';

-- JSONB-specific operators
SELECT * FROM products WHERE metadata @> '{"brand": "Samsung"}';   -- contains
SELECT * FROM products WHERE metadata ?| array['brand', 'manufacturer'];   -- has any key

-- Index for JSONB queries
CREATE INDEX idx_products_brand ON products ((metadata->>'brand'));
CREATE INDEX idx_products_metadata_gin ON products USING GIN (metadata);   -- general
```

### SQL Server JSON

```sql
-- Schema
CREATE TABLE products (
    id INT PRIMARY KEY,
    name NVARCHAR(100),
    metadata NVARCHAR(MAX) CHECK (ISJSON(metadata) = 1)
);

-- Query
SELECT * FROM products WHERE JSON_VALUE(metadata, '$.brand') = 'Samsung';
SELECT * FROM products WHERE JSON_VALUE(metadata, '$.specs.ram') = '8GB';

-- JSON_QUERY for objects
SELECT JSON_QUERY(metadata, '$.specs') as specs FROM products;

-- Index — computed column
ALTER TABLE products ADD brand AS JSON_VALUE(metadata, '$.brand') PERSISTED;
CREATE INDEX idx_products_brand ON products(brand);
```

### When JSON

```
✅ Use:
- Schema-less metadata (varying per row)
- Configurations / preferences
- Event payloads
- Read-mostly data

❌ Не use:
- Frequently joined / filtered fields (use regular columns)
- Heavily updated fields
- Critical business rules (no constraints на nested fields)
```

---

## EXPLAIN ANALYZE — production debugging

### PostgreSQL EXPLAIN ANALYZE deep

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, c.name 
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.created_at > '2026-01-01'
ORDER BY o.total DESC
LIMIT 10;
```

```
Limit  (cost=12345.67..12345.69 rows=10 width=234) (actual time=45.234..45.256 rows=10 loops=1)
  Buffers: shared hit=1234 read=567
  ->  Sort  (cost=12345.67..12567.89 rows=89234 width=234) (actual time=45.232..45.245 rows=10)
        Sort Key: o.total DESC
        Sort Method: top-N heapsort  Memory: 35kB
        ->  Hash Join  (cost=234.56..11890.12 rows=89234 width=234) (actual time=2.345..40.567 rows=89234)
              Hash Cond: (o.customer_id = c.id)
              Buffers: shared hit=987 read=345
              ->  Index Scan using idx_orders_created_at on orders o  
                  (cost=0.42..10234.56 rows=89234 width=180) 
                  (actual time=0.045..15.234 rows=89234)
                    Index Cond: (created_at > '2026-01-01'::date)
              ->  Hash  (cost=123.45..123.45 rows=10000 width=54)
                    ->  Seq Scan on customers c  (cost=0.00..123.45 rows=10000 width=54)
Planning Time: 0.345 ms
Execution Time: 45.456 ms
```

### Что искать

```
1. Estimated rows vs Actual rows
   Если estimated 1000 vs actual 100000 → outdated statistics
   Fix: ANALYZE table_name;

2. Sort Method: external merge → spill to disk
   Fix: increase work_mem или add index for ORDER BY

3. Buffers: shared read=NNN → много disk I/O
   Fix: index, или возможно увеличить shared_buffers

4. Seq Scan на больших таблицах в WHERE
   Fix: add index

5. Hash Spill on disk (Hash Batches > 1)
   Fix: increase work_mem

6. Nested Loop с большой outer table
   Fix: index inner loop, или force Hash Join
```

### Real-world bottleneck case 1: missing index

```
Before:
  Seq Scan on orders (cost=0..50000) (actual time=10..2500ms rows=89234)
  Filter: customer_id = 123
  Rows Removed by Filter: 9910766

→ ANALYZE показывает Seq Scan, filter removes 9.9M rows
→ Add index: CREATE INDEX idx_orders_customer_id ON orders(customer_id);

After:
  Index Scan using idx_orders_customer_id (cost=0..234) (actual time=0.5..15ms rows=89234)
```

### Real-world bottleneck case 2: missing JOIN index

```
Before:
  Nested Loop  (actual time=0..30000ms rows=89234)
    -> Seq Scan on orders (filter created_at > '2026-01-01')
    -> Seq Scan on customers (filter id = $1)

→ Inner side не имеет index → seq scan для каждой outer row
→ Add: CREATE INDEX idx_customers_id ON customers(id);   (PK should already do this!)
→ Or change to Hash Join: SET enable_nestloop = false; (test)

After:
  Hash Join (actual time=0..50ms rows=89234)
```

### Real-world bottleneck case 3: bad cardinality

```
Estimated rows: 100
Actual rows: 1000000

→ Outdated statistics → planner chose wrong plan
→ Fix: VACUUM ANALYZE orders;
   Or: ALTER TABLE orders ALTER COLUMN customer_id SET STATISTICS 1000;   (more samples)
```

### SQL Server — execution plans

SQL Server Management Studio (SSMS) → Ctrl+M → Include Actual Execution Plan.

Same concepts: Seq scan, Hash Match, Nested Loops, Sort spilling. Иконки instead of text.

---

## Connection pooling deep — PgBouncer

### Зачем external pooler

PostgreSQL spawn process per connection. 1000 connections = 1000 processes = OOM.

`max_connections` typically 100-200 в production. Application servers с pool size 20-50 each — 4 servers × 50 = 200 connections, на грани.

### PgBouncer setup

```ini
# pgbouncer.ini
[databases]
mydb = host=postgres-primary port=5432 dbname=mydb

[pgbouncer]
listen_addr = *
listen_port = 6432

# Pool mode
pool_mode = transaction        # session / transaction / statement

# Limits
max_client_conn = 1000         # клиенты могут подключиться
default_pool_size = 25         # реальных connections к Postgres
reserve_pool_size = 5
reserve_pool_timeout = 5

# Auth
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
```

### Pool modes

```
session     — connection assigned для всей session client
              (1 client = 1 backend connection while connected)
              Compatible со всем

transaction — connection released после COMMIT/ROLLBACK
              1000 clients могут share 25 backends если transactions короткие
              Default рекомендация
              ⚠️ Не работает с prepared statements, advisory locks held cross-transaction
              
statement   — connection released после каждого statement
              Maximum throughput но multi-statement transactions broken
              Использовать редко
```

### Application setup

```csharp
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseNpgsql("Host=pgbouncer;Port=6432;Database=mydb;Pooling=true;Maximum Pool Size=20"));
```

App connections idle быстро возвращаются к PgBouncer для reuse.

### Trade-offs

```
✅ PgBouncer:
- 10x-100x more concurrent clients possible
- Saves PostgreSQL resources
- Connection overhead amortized

❌ Cons:
- Extra hop (~ms latency)
- Transaction-mode breaks некоторые features
- Failover complexity
```

### Когда нужен

```
✅ Нужен:
- Cloud Postgres (Azure, RDS) с connection limits
- Many app instances (microservices, k8s)
- Lambda-style apps с short-lived connections
- Connection storms (100+ apps starting up)

❌ Не нужен:
- Single app с stable pool
- Self-managed Postgres с large max_connections
```

> [!question]- **Интервью: зачем PgBouncer?**
> PostgreSQL spawn process per connection — expensive. Production max_connections обычно 100-200. Many app instances/microservices easily exceed это. **PgBouncer** — connection pooler ПЕРЕД Postgres: clients подключаются к PgBouncer (cheap), он maintains pool real connections к Postgres. **Pool modes**: session (1:1), transaction (release after COMMIT, default рекомендация), statement (max throughput, breaks multi-statement). **Trade-off**: extra ~ms latency, но 10x-100x more concurrent clients. **Alternative**: PgCat (newer, multi-tenant aware), AWS RDS Proxy (managed).

---

## Cleanup — sections relocations

После expansion файл logically structured:
1. Indexes (existing)
2. Execution plans + Join algorithms (existing)
3. Pagination, N+1, Bulk (existing)
4. Transactions (existing)
5. **Partitioning (NEW)**
6. **Materialized Views (NEW)**
7. **CTE (NEW)**
8. **Window Functions (NEW)**
9. **JSON queries (NEW)**
10. **EXPLAIN deep (NEW)**
11. **PgBouncer (NEW)**

---

## Reading list (extended)

- **PostgreSQL Docs — Partitioning** — postgresql.org/docs/current/ddl-partitioning.html
- **PostgreSQL Docs — Materialized Views** — postgresql.org/docs/current/rules-materializedviews.html
- **PostgreSQL Docs — Window Functions** — postgresql.org/docs/current/tutorial-window.html
- **PostgreSQL Docs — JSONB** — postgresql.org/docs/current/datatype-json.html
- **PgBouncer docs** — pgbouncer.org
- **"PostgreSQL High Performance" — Gregory Smith** (book)
- **Use The Index, Luke** — use-the-index-luke.com (free book on indexing)
- **EXPLAIN visualizer** — explain.dalibo.com (paste explain output, get visual plan)

## См. также

- [[performance|.NET Performance]]
- [[queries-performance|EF Core Performance]]
- [[index|.NET Knowledge Base]]
