---
tags: [sql, indexes, btree, hash, performance, query-plan, postgresql]
level: Middle to Senior
date: 2026-08-02
---

# Indexes Deep — индексы досконально

> **Глубокий гайд по индексам в реляционных БД**. Что такое индекс изнутри, какие виды бывают, когда применять, как читать query plan, типичные ошибки. Фокус на PostgreSQL, применимо к SQL Server / MySQL.

---

## Что это, зачем и когда

### Что такое индекс — внутри

**Структура данных** для быстрого поиска. Хранится **рядом с таблицей** в виде отсортированного дерева (B-tree) или другой структуры.

**Аналогия:** Индекс в книге. Без — листаешь все страницы (full table scan). С индексом — открываешь нужную главу сразу (index seek).

```
Таблица users (1М строк):
  id │ email           │ name
  1  │ alice@x.com     │ Alice
  2  │ bob@y.com       │ Bob
  ...
  
Index idx_email на email:
  alice@x.com  → row 1
  bob@y.com    → row 2
  carol@z.com  → row 3
  ...
  (отсортировано по email!)
```

Запрос `WHERE email = 'bob@y.com'`:
- **Без index:** 1М сравнений → 500ms
- **С index:** ~20 сравнений в B-tree → 0.1ms

### Цена индекса

Индекс — **trade-off**:

| + | − |
|---|---|
| SELECT быстрее (если используется) | INSERT/UPDATE/DELETE медленнее (нужно обновить index) |
| ORDER BY быстрее (если соответствует) | Disk space (10-30% от таблицы) |
| JOIN быстрее (по indexed column) | RAM в кэше |

**Правило:** не индексируй "на всякий случай". Только колонки которые **реально** используются в WHERE, JOIN, ORDER BY часто выполняемых queries.

---

## 1. B-tree — главная структура

### Как работает

Сбалансированное дерево с N ключами в каждом узле. Поиск — `O(log n)`.

```
                    [50, 100]
                   /    |    \
              [10,30] [60,80] [120,150]
              /  |  \  /|\    /|\
            ...  ... ...
```

Поиск `value = 75`:
1. Корень: 50 < 75 < 100 → средняя ветвь
2. `[60, 80]`: 60 < 75 < 80 → средняя ветвь
3. Лист с `75` → найден

### Что хранится в листьях

| BD | В листьях |
|----|-----------|
| **PostgreSQL** | TID (tuple identifier) — указатель на строку |
| **SQL Server** (clustered) | Данные строки целиком |
| **MySQL InnoDB** (clustered PK) | Данные строки целиком |

### Когда B-tree подходит

✅ **Эффективен:**
- Equality: `WHERE id = 5`
- Range: `WHERE created_at > '2024-01-01'`
- Sorting: `ORDER BY id`
- Prefix matching: `WHERE name LIKE 'John%'`
- IN: `WHERE id IN (1, 2, 3)`

❌ **Не помогает:**
- Suffix / contains: `WHERE name LIKE '%son'` или `LIKE '%mid%'`
- Function on column: `WHERE LOWER(email) = 'x'`
- Unrelated WHERE: индекс на `email`, ищем `WHERE name = 'X'`

---

## 2. Создание индексов

### Базовый синтаксис

```sql
-- Простой
CREATE INDEX idx_users_email ON users(email);

-- Уникальный
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Composite (несколько колонок)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- Удалить
DROP INDEX idx_users_email;
```

### Naming conventions

```
idx_<table>_<columns>           — обычный
idx_<table>_<columns>_unique    — уникальный
fk_<table>_<column>             — на foreign key (часто авто-creates)
pk_<table>                      — primary key (auto)
```

### CREATE INDEX CONCURRENTLY (PostgreSQL)

```sql
-- ❌ Lock на таблицу — блокирует writes
CREATE INDEX idx_users_email ON users(email);

-- ✅ Без lock — можно делать в production
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

CONCURRENTLY медленнее, но не блокирует — критично для prod.

---

## 3. Composite indexes — порядок колонок

### Leftmost prefix rule

Индекс `(A, B, C)` помогает запросам по:
- ✅ `WHERE A = ?`
- ✅ `WHERE A = ? AND B = ?`
- ✅ `WHERE A = ? AND B = ? AND C = ?`
- ❌ `WHERE B = ?`
- ❌ `WHERE C = ?`
- ❌ `WHERE B = ? AND C = ?`

> [!info] PostgreSQL 18: skip scan размывает «абсолют»
> B-tree **skip scan** (PG 18) умеет использовать составной индекс и без условия на ведущую колонку: если у `A` низкая cardinality (например, 4 значения status), планировщик сам перебирает каждое значение `A` и делает точечные пробы по `B`. Т.е. `WHERE B = ?` может пойти по индексу `(A, B)`. Это спасательный круг, а не замена дизайна: чем выше cardinality ведущей колонки, тем бесполезнее skip scan. **Правило leftmost prefix остаётся эвристикой проектирования индексов.**

### Пример

```sql
CREATE INDEX idx_orders_user_date_status 
ON orders(user_id, created_at, status);

-- ✅ Использует index
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 1 AND created_at > '2024-01-01';
SELECT * FROM orders WHERE user_id = 1 AND created_at > '2024-01-01' AND status = 'active';

-- ✅ Использует частично (только user_id)
SELECT * FROM orders WHERE user_id = 1 AND status = 'active';

-- ❌ Не использует — нет user_id в WHERE
SELECT * FROM orders WHERE created_at > '2024-01-01';
SELECT * FROM orders WHERE status = 'active';
```

### Какой column первый

**Правило:** колонка с **самой высокой селективностью** (cardinality) — первая.

```
Cardinality = unique values / total rows
  high cardinality: email (миллион unique)     ← первой
  low cardinality:  status (5 значений)         ← последней
  низкая:           is_active (2 значения)      ← последней
```

```sql
-- ✅ user_id (high cardinality) первым
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- ❌ status (low) первым — индекс почти бесполезен
CREATE INDEX idx_orders_status_user ON orders(status, user_id);
```

### Equality > Range

В composite — equality колонки **до** range:

```sql
-- Запрос: WHERE user_id = 1 AND created_at BETWEEN ... AND ...

-- ✅ user_id (=) первым, created_at (range) вторым
CREATE INDEX ON orders(user_id, created_at);

-- ❌ Range первым — index использует только до created_at
CREATE INDEX ON orders(created_at, user_id);
```

---

## 4. Виды индексов в PostgreSQL

### B-tree (default)

Описано выше. 95% использования.

```sql
CREATE INDEX idx_users_email ON users(email);
-- эквивалентно:
CREATE INDEX idx_users_email ON users USING BTREE(email);
```

### Hash

Только equality (=), не range / sort. Редко лучше B-tree в Postgres.

```sql
CREATE INDEX idx_users_email_hash ON users USING HASH(email);
```

### GIN (Generalized Inverted Index)

Для **массивов**, **JSONB**, **full-text search**.

```sql
-- Array
CREATE INDEX idx_posts_tags ON posts USING GIN(tags);
SELECT * FROM posts WHERE tags @> ARRAY['sql'];

-- JSONB
CREATE INDEX idx_products_metadata ON products USING GIN(metadata);
SELECT * FROM products WHERE metadata @> '{"color": "red"}';

-- Full-text search
CREATE INDEX idx_articles_search ON articles USING GIN(to_tsvector('english', body));
SELECT * FROM articles WHERE to_tsvector('english', body) @@ to_tsquery('postgresql');
```

### GiST (Generalized Search Tree)

Для **геометрических** (PostGIS), **range types**, custom ordering.

```sql
CREATE INDEX idx_points_geo ON points USING GIST(coordinates);
SELECT * FROM points WHERE coordinates && box '((0,0),(10,10))';
```

### BRIN (Block Range Index)

Для **очень больших** таблиц (млрд строк) с natural ordering — времена, sequential IDs.

Хранит min/max для каждых N последовательных blocks. Маленький, но эффективен только если данные упорядочены.

```sql
CREATE INDEX idx_logs_created ON logs USING BRIN(created_at);
-- 10x меньше места чем B-tree
-- Эффективен если строки insertтятся по порядку времени
```

### SP-GiST

Для unbalanced data (деревья, hierarchies, spatial).

---

## 5. Special index types

### Partial index

Индекс **только подмножества** строк.

```sql
-- Только active users
CREATE INDEX idx_users_email_active 
ON users(email) 
WHERE is_active = true;

-- Запрос — использует index
SELECT * FROM users WHERE email = 'x@y.com' AND is_active = true;
```

**Применение:**
- Soft-delete: индекс на `WHERE deleted_at IS NULL`
- Status filtering: `WHERE status = 'active'`
- Sparse data: `WHERE archived_at IS NULL`

### Covering index (INCLUDE)

Дополнительные колонки в индексе — для **index-only scan** (без обращения к таблице).

```sql
-- B-tree по email + name/age в leaf nodes
CREATE INDEX idx_users_email_covering 
ON users(email) 
INCLUDE (name, age);

-- Index-only scan — таблица не читается
SELECT email, name, age FROM users WHERE email = 'x@y.com';
```

⚠️ Trade-off: индекс жирнее (хранятся доп. колонки).

### Functional index

Индекс на **результат функции**.

```sql
-- Без — не использует idx_email
SELECT * FROM users WHERE LOWER(email) = 'x@y.com';

-- С functional index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Использует
SELECT * FROM users WHERE LOWER(email) = 'x@y.com';
```

**Применение:**
- Case-insensitive search
- Computed values (`(price * quantity)`)
- Date extraction (`DATE_TRUNC('day', created_at)`)

### Expression index

```sql
-- На выражение
CREATE INDEX idx_orders_total 
ON orders((price * quantity));

SELECT * FROM orders WHERE price * quantity > 1000;
```

---

## 6. Index-only scan

Самый быстрый тип — **только index**, таблица не читается.

```sql
CREATE INDEX idx_users_email_name ON users(email) INCLUDE (name);

-- Index-only scan: всё в indexed колонках
EXPLAIN SELECT email, name FROM users WHERE email = 'x';
-- → Index Only Scan using idx_users_email_name

-- Index scan: нужно читать таблицу для других колонок
EXPLAIN SELECT email, name, age FROM users WHERE email = 'x';
-- → Index Scan using idx_users_email_name (heap fetch для age)
```

В PostgreSQL — **VACUUM** обновляет visibility map (нужен для index-only scan).

---

## 7. EXPLAIN — читаем query plan

### Базовый EXPLAIN

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 1;

-- Output:
-- Index Scan using idx_orders_user_id on orders
--   (cost=0.43..8.45 rows=10 width=64)
--   Index Cond: (user_id = 1)
```

| Поле | Что |
|------|-----|
| `cost=startup..total` | Estimated cost (планировщик) |
| `rows` | Estimated rows |
| `width` | Estimated bytes per row |

### EXPLAIN ANALYZE — реальные данные

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 1;

-- Output:
-- Index Scan using idx_orders_user_id on orders
--   (cost=0.43..8.45 rows=10 width=64) 
--   (actual time=0.025..0.118 rows=8 loops=1)
--   Index Cond: (user_id = 1)
-- Planning Time: 0.234 ms
-- Execution Time: 0.156 ms
```

`actual time=startup..total rows=actual loops=N` — реальные числа.

### Типы scan

| Scan | Когда |
|------|-------|
| **Seq Scan** (sequential) | Полная таблица — нет индекса |
| **Index Scan** | Index найден, потом таблица для всех колонок |
| **Index Only Scan** | Всё в индексе — fastest |
| **Bitmap Heap Scan** | Много строк через index — оптимизация |
| **Bitmap Index Scan** | Часть Bitmap Heap |

### Признаки проблем

```sql
EXPLAIN ANALYZE SELECT ... ;
```

Ищи:

1. **Seq Scan на больших таблицах** → нужен index
2. **rows estimate vs actual** сильно отличаются → ANALYZE table нужен
3. **Loops > 1** → возможно nested loop с индекс scan нужен hash join
4. **Filter** после Index Scan → колонка из filter не в index, добавь
5. **Sort** в plan → ORDER BY без index можно ускорить добавив index

### EXPLAIN с opциями

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON) 
SELECT ... ;

-- BUFFERS — сколько blocks read из памяти / диска
-- VERBOSE — больше деталей  
-- FORMAT JSON — для tooling (pgAdmin, JetBrains)
```

### Visualizers

- **pgMustard** — paid, лучший
- **explain.depesz.com** — бесплатный, paste plan
- **JetBrains Rider DataGrip** — built-in
- **pgAdmin** — встроенный

---

## 8. Типичные ошибки

### 1. Index не используется из-за функции

```sql
-- ❌ idx_email не используется
SELECT * FROM users WHERE LOWER(email) = 'x';

-- ✅ Functional index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```

### 2. Implicit type cast

```sql
-- В Postgres user_id is BIGINT, передаём string
SELECT * FROM users WHERE user_id = '123';  -- неявный cast → seq scan!

-- ✅ Правильный type
SELECT * FROM users WHERE user_id = 123;
```

### 3. LIKE с leading %

```sql
-- ❌ Не использует B-tree
WHERE email LIKE '%@gmail.com';

-- ✅ Trigram index (extension pg_trgm)
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_users_email_trgm ON users USING GIN(email gin_trgm_ops);
WHERE email LIKE '%@gmail.com';  -- теперь использует
```

### 4. OR conditions

```sql
-- ❌ OR — может не использовать индексы оптимально
WHERE email = 'x' OR phone = 'y';

-- ✅ UNION ALL быстрее иногда
SELECT * FROM users WHERE email = 'x'
UNION ALL
SELECT * FROM users WHERE phone = 'y';
```

### 5. NULL в индексах

```sql
-- В Postgres — NULL индексируется (отличие от Oracle)
CREATE INDEX idx ON users(email);
WHERE email IS NULL;  -- работает

-- В PG NULL "больше" любого значения по умолчанию
CREATE INDEX idx ON orders(created_at NULLS FIRST);
```

### 6. Слишком много индексов

```sql
-- 15 индексов на одну таблицу
-- INSERT: 100ms (без) → 800ms (с 15 индексами)
```

Каждый INSERT обновляет каждый index. **Хотя бы 1 SELECT/INSERT ratio** → не имеет смысла индекс.

### 7. Дублирующие индексы

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_email_name ON users(email, name);
-- idx_users_email избыточен — leftmost prefix rule покрывает email queries

-- ✅ Удалить избыточный
DROP INDEX idx_users_email;
```

### 8. ORDER BY без index

```sql
-- ❌ Sort step после fetching
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
-- → Seq Scan + Sort (или Index Scan + Sort если нет DESC index)

-- ✅ Index с правильным ordering
CREATE INDEX idx_orders_created_desc ON orders(created_at DESC);
SELECT * FROM orders ORDER BY created_at DESC LIMIT 10;
-- → Index Scan (no sort)
```

---

## 9. Maintenance

### Index bloat

PostgreSQL — MVCC. Updates создают new tuple, old не сразу удаляется. Indexes "толстеют" — bloat.

```sql
-- Размер index (с bloat)
SELECT pg_size_pretty(pg_relation_size('idx_users_email'));

-- Reindex (без lock в Postgres 12+)
REINDEX INDEX CONCURRENTLY idx_users_email;
```

Регулярный `VACUUM ANALYZE` (autovacuum обычно достаточно).

### Unused indexes

```sql
-- Найти неиспользуемые
SELECT 
  schemaname, tablename, indexname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- ни разу не использован
ORDER BY pg_relation_size(indexrelid) DESC;

-- Удалить
DROP INDEX idx_unused;
```

### Statistics

PostgreSQL собирает статистики (cardinality, distribution). Stale stats → bad plans.

```sql
-- Manual analyze
ANALYZE users;

-- Auto: autovacuum по достижении threshold
```

---

## 10. Strategy — когда индексировать

### Индексируй

✅ **Всегда:**
- Primary keys (auto)
- Foreign keys
- Колонки в WHERE часто-выполняемых queries
- ORDER BY на больших таблицах
- JOIN keys

✅ **Часто:**
- UNIQUE constraints
- Колонки в GROUP BY
- Range queries (BETWEEN, >, <)
- LIKE 'prefix%'

⚠️ **Иногда:**
- Boolean columns (только если selective)
- Status columns (низкая cardinality — partial index)

❌ **Никогда:**
- Колонки которые редко в WHERE
- Маленькие таблицы (<1000 rows — seq scan быстрее)
- Колонки которые часто меняются (write overhead)

### Decision matrix

```
Selectivity    │ Always WHERE │ Sometimes WHERE │ Rarely WHERE
───────────────┼──────────────┼─────────────────┼──────────────
High (>90%)    │     YES      │      YES        │   Maybe
Medium         │     YES      │     Maybe       │     NO
Low (<10%)     │  Partial     │  Composite      │     NO
```

---

## 11. Real-world examples

### Pagination

```sql
-- Naive — медленный для глубокой страницы
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 100000;
-- скан 100010 строк!

-- Cursor-based (keyset)
SELECT * FROM orders 
WHERE id > 100000 
ORDER BY id 
LIMIT 10;
-- O(log n) если index на id (PK уже есть)
```

См. [[queries-performance|EF Queries Performance]].

### Soft delete

```sql
CREATE INDEX idx_orders_active ON orders(created_at)
WHERE deleted_at IS NULL;

-- Все queries на active orders используют тонкий partial index
SELECT * FROM orders WHERE deleted_at IS NULL ORDER BY created_at;
```

### Multi-tenant

```sql
-- Каждый запрос фильтруется по tenant
CREATE INDEX idx_orders_tenant_user 
ON orders(tenant_id, user_id, created_at DESC);

SELECT * FROM orders 
WHERE tenant_id = 'a' AND user_id = 1 
ORDER BY created_at DESC LIMIT 10;
```

---

## 12. Best Practices

- **Index foreign keys** — JOINs летают
- **Composite в правильном порядке** — высокая selectivity сначала, equality перед range
- **CREATE INDEX CONCURRENTLY** в production
- **Partial indexes** для filtered queries
- **EXPLAIN ANALYZE** для проблемных queries
- **pg_stat_user_indexes** — найти unused
- **Regular VACUUM** для visibility map
- **Не over-index** — каждый замедляет writes
- **REINDEX CONCURRENTLY** при bloat
- **Functional indexes** для transformations
- **Trigram (pg_trgm)** для LIKE '%pattern%'
- **GIN для JSONB / arrays / full-text**
- **BRIN для time-series** (миллиарды строк)

---

## Case Studies

### Case Study #1 — Sub-second search для e-commerce каталога

**Сценарий:** 1M товаров. Search query `LIKE '%shirt%'` — 5 sec.

**❌ SQL LIKE:**
```sql
SELECT * FROM products WHERE name LIKE '%shirt%'
-- Full table scan, 5 sec
```

**✅ Postgres FTS:**
```sql
ALTER TABLE products ADD COLUMN search_vector tsvector;
CREATE INDEX idx_search ON products USING GIN(search_vector);

SELECT * FROM products WHERE search_vector @@ plainto_tsquery('shirt');
-- 50 ms
```

**✅ Better — Elasticsearch:**
```csharp
var response = await _es.SearchAsync<Product>(s => s
    .Query(q => q.MultiMatch(m => m.Query("shirt").Fields(f => f.Field(p => p.Name).Field(p => p.Description))))
    .Size(20));
// 10 ms + relevance scoring + faceting
```

См. [[nosql-databases|NoSQL Databases]] и [[postgresql-deep|PostgreSQL Deep]].

---

### Case Study #2 — Composite index strategy

**Сценарий:** Query `WHERE customer_id = ? AND status = ? ORDER BY created_at DESC`.

**❌ Single column indexes:**
```sql
CREATE INDEX idx_customer ON orders(customer_id);
CREATE INDEX idx_status ON orders(status);
-- Query plan: bitmap scan, не оптимально
```

**✅ Composite index в правильном порядке:**
```sql
CREATE INDEX idx_customer_status_date 
    ON orders(customer_id, status, created_at DESC);
-- Все 3 columns в одном index, idx scan
```

**Правило order:**
1. Equality columns (=) первыми
2. Range columns (>, <) последними
3. Sort columns в матching order

---

### Case Study #3 — Index maintenance overhead

**Сценарий:** Table с 10 indexes. Inserts замедлились 5x.

**Проблема:** каждый INSERT обновляет ВСЕ indexes.

**Solution:**
```sql
-- Identify unused indexes
SELECT indexrelname, idx_scan FROM pg_stat_user_indexes 
WHERE idx_scan = 0 ORDER BY pg_relation_size(indexrelid) DESC;

-- Drop unused
DROP INDEX idx_old_unused;

-- Insert performance восстанавливается
```

**Lesson:** indexes — trade-off. Read fast, write slow. Audit periodically.


---

## Cheat sheet

| Index type | Use case |
|------------|----------|
| B-tree (default) | Equality + range queries |
| Hash | Equality only (rare practical use) |
| GiST | Full-text search, geometric |
| GIN | Full-text, arrays, JSONB |
| BRIN | Very large tables, time-series |
| Bloom (Postgres extension) | Multi-column equality |

| Indexing strategy | When |
|-------------------|------|
| Single-column index | Common WHERE на одно поле |
| Composite index | WHERE + ORDER BY combined |
| Covering index | Query reads только indexed columns |
| Partial index | Filter условие фиксирован (`WHERE status = 'active'`) |
| Expression index | WHERE expression(col) (e.g., LOWER(email)) |
| Unique index | Enforces uniqueness + speed |

**Common Mistakes:**
- ❌ Index на каждый column (write penalty)
- ❌ Composite index с wrong column order
- ❌ Forgot to ANALYZE after big import
- ❌ Index не используется (planner не выбирает)
- ❌ Trailing wildcard `LIKE 'abc%'` works, leading `LIKE '%abc'` не работает с B-tree

| Diagnostic | SQL |
|-----------|-----|
| Query plan | `EXPLAIN ANALYZE SELECT ...` |
| Index size | `SELECT pg_size_pretty(pg_relation_size('idx_name'))` |
| Unused indexes | `SELECT * FROM pg_stat_user_indexes WHERE idx_scan = 0` |
| Slow queries | `pg_stat_statements` extension |


---

## Decision tree

```
Index стратегия?
│
├── Какой query type?
│   ├── Single column equality → B-tree single column
│   ├── Multi column WHERE → Composite index
│   ├── Range queries → B-tree
│   ├── Full-text → GIN с tsvector
│   ├── JSON queries → GIN на JSONB
│   ├── Geographic → PostGIS GiST
│   └── Very large historical → BRIN
│
├── Index covers query?
│   ├── Reads только indexed columns → covering index
│   └── Joins many tables → consider denormalization
│
├── Index size становится проблемой?
│   ├── Large rare values → partial index с WHERE
│   ├── Functional usage (LOWER) → expression index
│   └── Multi-tenant → per-tenant partition
│
├── Maintenance?
│   ├── Tables с >1M rows → REINDEX periodically
│   ├── ANALYZE после bulk imports
│   └── Drop unused indexes (audit ежеquarter)
│
└── Альтернативы?
    ├── Materialized view (precomputed query result)
    ├── Cache layer (Redis для hot queries)
    └── Specialized DB (Elasticsearch для search)
```


---

## См. также

- [[sql-basics|SQL Basics]] — fundamentals
- [[optimization|SQL Optimization]] — query plan deep
- [[postgresql-deep|PostgreSQL Deep]] — PG specifics, RLS, JSONB
- [[eav-flexible-store-indexing|EAV Flexible Store Indexing]] — один набор индексов на любую схему, partial/covering для EAV
- [[queries-performance|EF Queries Performance]] — N+1, projections
- [[migrations|EF Migrations]] — index creation in migrations

## Reading list

- **Use The Index, Luke** — use-the-index-luke.com (must-read!)
- **PostgreSQL Documentation — Indexes** — postgresql.org/docs/current/indexes.html
- **SQL Performance Explained** — Markus Winand (книга по indexes)
- **The Art of PostgreSQL** — Dimitri Fontaine
- **PostgreSQL Index Types** — pganalyze.com/blog
- **Postgres Weekly** — postgresweekly.com — newsletter
