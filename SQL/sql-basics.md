---
tags: [sql, basics, fundamentals, ddl, dml, joins, transactions, normalization]
level: Junior to Senior
date: 2026-04-30
---

# SQL Basics — фундаментальные концепции

> Базовые понятия реляционных БД и SQL для разработчиков. Закрывает пробел "что такое SQL" перед optimization. Темы: DDL/DML/DCL/TCL, joins, transactions, ACID, normalization, типы данных, constraints, views, stored procedures, чем отличаются БД (PostgreSQL/SQL Server/MySQL).

---

## Что это, зачем и когда

### Что такое SQL?

**Structured Query Language** — язык для работы с **реляционными базами данных**. Декларативный — описываешь *что* нужно, не *как* получить.

**Аналогия:** Excel в виде языка. Таблицы со строками, можно фильтровать, сортировать, группировать, объединять. SQL — это команды для Excel, которые работают на миллионах строк.

### Зачем

| Без SQL | С SQL |
|---------|-------|
| Хранишь данные в файлах / памяти | Структурированно в таблицах |
| Каждый раз пишешь алгоритм поиска | Один запрос, БД оптимизирует |
| Не масштабируется | До терабайт данных |
| Нет ACID гарантий | Транзакции из коробки |
| Анализ — ad-hoc Python | Один SELECT — отчёт готов |

### Когда какая БД

| Задача | БД |
|--------|-----|
| Сложные queries, JSONB | **PostgreSQL** ⭐ — default 2026 |
| Microsoft / .NET ecosystem | SQL Server / Azure SQL |
| Простой web app, MySQL hosting повсюду | MySQL / MariaDB |
| Embedded / mobile | SQLite |
| Огромные таблицы (100M+ rows) | PostgreSQL с partitioning, или ClickHouse для analytics |
| Time-series | TimescaleDB / InfluxDB |
| Графы | Neo4j |
| Schema-less документы | MongoDB / DynamoDB (NoSQL) |
| Очереди / pub-sub | Redis (in-memory) |

> [!info] PostgreSQL — современный default
> Для нового проекта в 2026 — PostgreSQL практически всегда лучший выбор. Open-source, мощный, JSONB, RLS, full-text search, extensions, рост популярности. См. [[postgresql-deep|postgresql-deep]].

---

## 1. Структура реляционной БД

### Таблица (Table)

Двумерная структура: строки + колонки.

```
┌────────────────────────────────────────┐
│ users                                  │
├──────┬───────────┬─────────────────────┤
│ id   │ name      │ email               │  ← columns
├──────┼───────────┼─────────────────────┤
│ 1    │ Alice     │ alice@example.com   │  ← rows
│ 2    │ Bob       │ bob@example.com     │
│ 3    │ Charlie   │ charlie@example.com │
└──────┴───────────┴─────────────────────┘
```

### Связи между таблицами

```
users (1) ─────────< (N) orders
   id ─────fk────── user_id
```

Один user может иметь много orders. `user_id` в `orders` — **foreign key (FK)** ссылается на `users.id` (**primary key, PK**).

### Schema

Группа связанных таблиц. В PostgreSQL — namespace для таблиц. По умолчанию `public`.

```sql
CREATE SCHEMA shop;
CREATE TABLE shop.products (id INT, name TEXT);
SELECT * FROM shop.products;
```

---

## 2. SQL команды — 4 группы

### DDL — Data Definition Language (структура)

Создание / изменение схемы.

```sql
-- Создать таблицу
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Изменить таблицу
ALTER TABLE users ADD COLUMN age INT;
ALTER TABLE users DROP COLUMN age;
ALTER TABLE users ALTER COLUMN name TYPE TEXT;

-- Удалить таблицу
DROP TABLE users;

-- Truncate — удалить все данные но сохранить структуру
TRUNCATE TABLE users;
```

### DML — Data Manipulation Language (данные)

Работа с данными.

```sql
-- INSERT — добавить
INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');

INSERT INTO users (name, email) VALUES 
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com');

-- SELECT — прочитать
SELECT * FROM users;
SELECT name, email FROM users WHERE id = 1;

-- UPDATE — изменить
UPDATE users SET name = 'Alicia' WHERE id = 1;

-- DELETE — удалить
DELETE FROM users WHERE id = 1;
```

### DCL — Data Control Language (права)

```sql
GRANT SELECT, INSERT ON users TO app_user;
REVOKE DELETE ON users FROM app_user;
```

### TCL — Transaction Control Language (транзакции)

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- или ROLLBACK
```

---

## 3. SELECT — основа

### Базовый SELECT

```sql
SELECT column1, column2 FROM table_name;
SELECT * FROM users;  -- * = все колонки (избегай в production)
```

### WHERE — фильтрация

```sql
SELECT * FROM users WHERE age > 18;
SELECT * FROM users WHERE name = 'Alice';
SELECT * FROM users WHERE age BETWEEN 18 AND 65;
SELECT * FROM users WHERE name IN ('Alice', 'Bob');
SELECT * FROM users WHERE name LIKE 'A%';  -- начинается с A
SELECT * FROM users WHERE email IS NOT NULL;
SELECT * FROM users WHERE age > 18 AND city = 'Moscow';
```

### ORDER BY — сортировка

```sql
SELECT * FROM users ORDER BY name;             -- ASC по умолчанию
SELECT * FROM users ORDER BY age DESC;
SELECT * FROM users ORDER BY age DESC, name ASC;  -- multiple columns
```

### LIMIT / OFFSET — pagination

```sql
SELECT * FROM users LIMIT 10;             -- первые 10
SELECT * FROM users LIMIT 10 OFFSET 20;   -- 21-30
```

### DISTINCT — уникальные значения

```sql
SELECT DISTINCT city FROM users;
SELECT COUNT(DISTINCT city) FROM users;  -- сколько уникальных
```

### GROUP BY — группировка

```sql
-- Сколько users в каждом городе
SELECT city, COUNT(*) AS user_count
FROM users
GROUP BY city
ORDER BY user_count DESC;

-- Средний возраст по городу, только города с >10 users
SELECT city, AVG(age) AS avg_age
FROM users
GROUP BY city
HAVING COUNT(*) > 10;
```

> [!info] WHERE vs HAVING
> - `WHERE` — фильтрует строки **до** GROUP BY
> - `HAVING` — фильтрует группы **после** GROUP BY
> - `WHERE` не может использовать aggregate functions (`COUNT`, `SUM`, `AVG`)

### Aggregate functions

```sql
SELECT COUNT(*) FROM users;
SELECT COUNT(DISTINCT city) FROM users;
SELECT SUM(amount) FROM orders;
SELECT AVG(age) FROM users;
SELECT MIN(price), MAX(price) FROM products;
```

---

## 4. JOINs — объединение таблиц

Главный механизм работы с реляционными данными.

### INNER JOIN — пересечение

```sql
-- Только users у которых есть orders
SELECT u.name, o.id AS order_id, o.total
FROM users u
INNER JOIN orders o ON u.id = o.user_id;
```

### LEFT JOIN — все из левой + матчи из правой

```sql
-- Все users + их orders (NULL если orders нет)
SELECT u.name, o.id AS order_id, o.total
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- User без orders → row с NULL в order_id, total
```

### RIGHT JOIN — все из правой

```sql
-- Все orders + users (NULL если orphan order — редко)
SELECT u.name, o.id
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;
```

В практике почти всегда LEFT JOIN — переписать RIGHT можно как LEFT поменяв таблицы.

### FULL OUTER JOIN — всё из обеих

```sql
SELECT u.name, o.id
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;
```

### CROSS JOIN — Декартово произведение

```sql
-- Каждый user × каждый product (для combinations)
SELECT u.name, p.name
FROM users u
CROSS JOIN products p;
```

⚠️ Опасно — 1000 users × 10000 products = 10M rows.

### Self join

```sql
-- Найти users с одинаковым city
SELECT u1.name, u2.name, u1.city
FROM users u1
INNER JOIN users u2 ON u1.city = u2.city AND u1.id < u2.id;
```

### Несколько JOIN

```sql
SELECT u.name, o.id, p.name AS product, oi.quantity
FROM users u
INNER JOIN orders o ON u.id = o.user_id
INNER JOIN order_items oi ON o.id = oi.order_id
INNER JOIN products p ON oi.product_id = p.id
WHERE u.city = 'Moscow';
```

---

## 5. Subqueries и CTEs

### Subquery — подзапрос

```sql
-- Users с ценой ордера выше среднего
SELECT name FROM users WHERE id IN (
    SELECT user_id FROM orders WHERE total > (
        SELECT AVG(total) FROM orders
    )
);
```

### CTE — Common Table Expression (читабельнее)

```sql
WITH high_value_orders AS (
    SELECT user_id, total
    FROM orders
    WHERE total > 1000
),
avg_order AS (
    SELECT AVG(total) AS avg_total FROM orders
)
SELECT u.name
FROM users u
INNER JOIN high_value_orders hvo ON u.id = hvo.user_id
WHERE hvo.total > (SELECT avg_total FROM avg_order);
```

CTE = "временные таблицы" в рамках одного запроса. Можно ссылаться несколько раз.

### Recursive CTE — для иерархий

```sql
-- Все потомки category id=1 (categories.parent_id ссылается на parent)
WITH RECURSIVE descendants AS (
    SELECT id, name, parent_id FROM categories WHERE id = 1
    UNION ALL
    SELECT c.id, c.name, c.parent_id
    FROM categories c
    INNER JOIN descendants d ON c.parent_id = d.id
)
SELECT * FROM descendants;
```

---

## 6. Window functions — мощь без GROUP BY

```sql
-- Top 3 orders по каждому user
SELECT user_id, id, total,
       ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY total DESC) AS rn
FROM orders
WHERE rn <= 3;  -- ОШИБКА — rn не доступно в WHERE

-- Правильно — через subquery
SELECT * FROM (
    SELECT user_id, id, total,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY total DESC) AS rn
    FROM orders
) ranked
WHERE rn <= 3;
```

### Полезные window functions

```sql
-- Running total (накопительная сумма)
SELECT date, amount,
       SUM(amount) OVER (ORDER BY date) AS running_total
FROM transactions;

-- Сравнение с предыдущей строкой
SELECT date, amount,
       amount - LAG(amount) OVER (ORDER BY date) AS diff_from_prev
FROM transactions;

-- Rank
SELECT name, score,
       RANK() OVER (ORDER BY score DESC) AS rank,
       DENSE_RANK() OVER (ORDER BY score DESC) AS dense_rank,
       ROW_NUMBER() OVER (ORDER BY score DESC) AS row_number
FROM students;
-- RANK: 1, 2, 2, 4 (skip)
-- DENSE_RANK: 1, 2, 2, 3 (no skip)
-- ROW_NUMBER: 1, 2, 3, 4 (always unique)

-- Percentile
SELECT salary,
       PERCENT_RANK() OVER (ORDER BY salary) AS percentile
FROM employees;
```

---

## 7. Constraints — ограничения

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,                              -- PK
    email VARCHAR(255) UNIQUE NOT NULL,                 -- UNIQUE + NOT NULL
    age INT CHECK (age >= 0 AND age <= 150),           -- CHECK
    country_id INT REFERENCES countries(id),            -- FK
    created_at TIMESTAMP DEFAULT NOW(),                 -- DEFAULT
    
    CONSTRAINT email_format CHECK (email LIKE '%@%')   -- named constraint
);
```

| Тип | Что |
|-----|-----|
| **PRIMARY KEY** | Уникально + NOT NULL + кластеризованный индекс (обычно) |
| **UNIQUE** | Уникально (но может быть NULL) |
| **NOT NULL** | Не может быть NULL |
| **CHECK** | Произвольное условие |
| **FOREIGN KEY** | Ссылка на другую таблицу |
| **DEFAULT** | Значение по умолчанию если не указано |

### Foreign key actions

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id) 
        ON DELETE CASCADE      -- удалить orders при удалении user
        ON UPDATE CASCADE      -- обновить если user.id изменится
);

-- Варианты ON DELETE:
-- CASCADE   — удалить дочерние
-- SET NULL  — установить FK в NULL
-- SET DEFAULT — установить default value
-- RESTRICT  — запретить удаление если есть дочерние
-- NO ACTION — то же что RESTRICT (default)
```

---

## 8. Транзакции и ACID

### Что такое транзакция

Группа операций, которая выполняется **атомарно** — либо все, либо ни одной.

```sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Если что-то пойдёт не так здесь, до COMMIT — мы можем ROLLBACK
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;  -- или ROLLBACK;
```

### ACID

| | Что значит |
|--|-----------|
| **Atomicity** | Всё или ничего — частичный commit невозможен |
| **Consistency** | После транзакции БД в валидном state (constraints проверены) |
| **Isolation** | Параллельные транзакции "не видят" друг друга (зависит от уровня) |
| **Durability** | После COMMIT — данные на диске, не потеряются при crash |

### Isolation Levels

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
-- Доступные:
-- READ UNCOMMITTED — самый слабый (dirty reads возможны)
-- READ COMMITTED   — default в PostgreSQL и SQL Server
-- REPEATABLE READ  — данные внутри transaction stable
-- SERIALIZABLE     — самый строгий (как будто single-threaded)
```

| Level | Dirty Read | Non-repeatable Read | Phantom Read |
|-------|-----------|----------------------|---------------|
| READ UNCOMMITTED | ✓ может | ✓ | ✓ |
| READ COMMITTED | ✗ | ✓ | ✓ |
| REPEATABLE READ | ✗ | ✗ | ✓ (SQL Server) / ✗ (PG) |
| SERIALIZABLE | ✗ | ✗ | ✗ |

См. [[../EFCore/concurrency|EF Core concurrency]] — детали isolation levels.

### Savepoints

```sql
BEGIN;
INSERT INTO logs VALUES (...);

SAVEPOINT before_update;
UPDATE users SET ... WHERE ...;

-- Что-то не так — откат только update
ROLLBACK TO SAVEPOINT before_update;

-- log остался, можно продолжить
COMMIT;
```

---

## 9. Индексы — введение

> Подробнее: [[optimization|SQL Optimization]] и [[postgresql-deep|PostgreSQL Deep]].

### Что такое индекс

**Структура данных** для быстрого поиска. Аналогия: индекс в книге — позволяет найти страницу с темой не читая всё подряд.

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);
```

### Когда индекс помогает

```sql
-- ✓ Использует индекс если есть idx_users_email
SELECT * FROM users WHERE email = 'alice@example.com';

-- ✓ Использует
SELECT * FROM users WHERE email LIKE 'alice%';

-- ✗ Не использует (LIKE начинается с %)
SELECT * FROM users WHERE email LIKE '%example.com';

-- ✗ Не использует — функция от колонки
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
-- Решение: functional index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```

### Composite index

```sql
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at DESC);

-- ✓ Использует
SELECT * FROM orders WHERE user_id = 1;
SELECT * FROM orders WHERE user_id = 1 AND created_at > '2024-01-01';
SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC;

-- ✗ Не использует — leading column не указана
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

> [!info] Leftmost prefix rule
> Composite index `(A, B, C)` помогает запросам по `A`, `(A, B)`, `(A, B, C)`. Не помогает по `B`, `C`, `(B, C)`.

### Стоимость индексов

- **+ Read speed** — быстрее SELECT
- **− Write overhead** — INSERT/UPDATE/DELETE медленнее (нужно обновить индекс)
- **− Disk space** — индекс занимает место (~10-30% от размера таблицы)

**Правило:** не индексируй "на всякий случай". Только колонки в WHERE, JOIN, ORDER BY часто используемых queries.

---

## 10. Views — virtual tables

```sql
CREATE VIEW active_users AS
SELECT id, name, email
FROM users
WHERE deleted_at IS NULL AND last_login > NOW() - INTERVAL '90 days';

-- Использовать как обычную таблицу
SELECT * FROM active_users;
```

### Materialized views (PostgreSQL)

```sql
-- Pre-computed view — данные хранятся
CREATE MATERIALIZED VIEW user_stats AS
SELECT u.id, u.name, COUNT(o.id) AS order_count, SUM(o.total) AS total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- Обновить (manually или by cron)
REFRESH MATERIALIZED VIEW user_stats;
REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats;  -- без блокировки
```

**Когда materialized view:**
- Aggregations на больших таблицах (медленно вычислять каждый раз)
- Read-heavy + старые данные OK

---

## 11. Stored procedures и functions

### PostgreSQL function

```sql
CREATE OR REPLACE FUNCTION transfer_funds(
    sender_id INT,
    recipient_id INT,
    amount DECIMAL
) RETURNS BOOLEAN AS $$
BEGIN
    UPDATE accounts SET balance = balance - amount WHERE id = sender_id;
    UPDATE accounts SET balance = balance + amount WHERE id = recipient_id;
    RETURN TRUE;
EXCEPTION
    WHEN OTHERS THEN
        RAISE;
        RETURN FALSE;
END;
$$ LANGUAGE plpgsql;

-- Использование
SELECT transfer_funds(1, 2, 100.00);
```

### Когда (не) использовать stored procedures

✅ **Когда:**
- Очень специфичная DB-bound логика (e.g. complex aggregations)
- Performance-critical (avoid round-trips)
- Triggered events (BEFORE INSERT)

❌ **Когда нет:**
- Бизнес-логика (хочешь иметь её в коде, не в БД)
- Тестируемость важна (SP сложно тестировать)
- Multi-DB поддержка (vendor lock-in)
- Versioning logic — git for code, миграции для SP

В большинстве случаев — **держи логику в C#**, не в БД.

---

## 12. Triggers

```sql
CREATE OR REPLACE FUNCTION audit_user_changes()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, action, user_id, changed_at)
    VALUES ('users', TG_OP, COALESCE(NEW.id, OLD.id), NOW());
    RETURN COALESCE(NEW, OLD);
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER users_audit
AFTER INSERT OR UPDATE OR DELETE ON users
FOR EACH ROW
EXECUTE FUNCTION audit_user_changes();
```

> [!warning] Triggers — implicit и опасны
> Магические side-effects, сложно дебажить. Лучше — application-level logic + EF Core interceptors. См. [[../EFCore/patterns#audit-interceptors|EF Audit Interceptors]].

---

## 13. Normalization — нормальные формы

### Зачем

Хранить каждый факт **один раз**. Избегать дублирования и аномалий обновления.

### 1NF (First Normal Form)

Каждая ячейка содержит **одно атомарное** значение.

```
❌ Не 1NF
| user_id | name  | phones                          |
|---------|-------|---------------------------------|
| 1       | Alice | 555-1234, 555-5678              |  -- два значения!

✅ 1NF — отдельная таблица phones
users: id, name
phones: id, user_id, phone_number
```

### 2NF — каждая non-key колонка зависит от **всего** PK

```
❌ Composite PK (order_id, product_id), но product_name зависит только от product_id
order_items: order_id, product_id, product_name, quantity

✅ Вынесли в products
order_items: order_id, product_id, quantity
products: product_id, product_name
```

### 3NF — non-key колонки зависят **только** от PK (transitive deps убраны)

```
❌ city_zip зависит от city, не от user.id напрямую
users: id, name, city, city_zip

✅ Вынесли city info
users: id, name, city_id
cities: id, name, zip
```

### BCNF / 4NF / 5NF

Дальше нормальные формы — академические, на практике редко нужны.

### Денормализация

В реальных high-perf системах **специально** денормализуют для скорости чтения.

```sql
-- Денормализация — храним user_name в orders для быстрого SELECT без JOIN
orders: id, user_id, user_name, total
-- Trade-off: при изменении user.name надо обновить во всех orders
```

См. [[../Architecture/distributed-systems|Distributed Systems]] — CQRS read models часто денормализованы.

---

## 14. Типы данных

### Числовые

| Тип | Размер | Range |
|-----|--------|-------|
| `SMALLINT` | 2 bytes | ±32k |
| `INT` / `INTEGER` | 4 bytes | ±2.1B |
| `BIGINT` | 8 bytes | ±9.2 × 10^18 |
| `DECIMAL(p, s)` / `NUMERIC` | variable | exact, для денег |
| `REAL` / `FLOAT4` | 4 bytes | inexact float |
| `DOUBLE PRECISION` / `FLOAT8` | 8 bytes | inexact double |

### Строки

| Тип | Когда |
|-----|-------|
| `CHAR(n)` | Fixed-length, padded с пробелами (редко используется) |
| `VARCHAR(n)` | Variable, max n chars |
| `TEXT` | Unlimited (PostgreSQL — без overhead vs VARCHAR) |
| `CITEXT` (PG) | Case-insensitive text |

### Даты / время

| Тип | Что |
|-----|-----|
| `DATE` | Только дата |
| `TIME` | Только время |
| `TIMESTAMP` | Дата + время (без TZ) |
| `TIMESTAMPTZ` | С timezone (рекомендуется!) |
| `INTERVAL` | Период (1 day, 2 hours) |

> [!info] Используй TIMESTAMPTZ
> Всегда. Хранит UTC внутри, преобразует к user TZ на read. Простоe `TIMESTAMP` без TZ — головная боль.

### UUID

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name TEXT
);
```

UUID — для distributed систем (нет central counter), 16 bytes.

### JSON / JSONB (PostgreSQL)

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    metadata JSONB  -- binary JSON, faster querying
);

INSERT INTO products (metadata) VALUES
    ('{"color": "red", "size": "L", "tags": ["sale", "new"]}');

-- Query JSON
SELECT * FROM products WHERE metadata->>'color' = 'red';
SELECT * FROM products WHERE metadata @> '{"tags": ["sale"]}';

-- Index on JSON
CREATE INDEX idx_products_color ON products ((metadata->>'color'));
```

См. [[postgresql-deep#jsonb|PostgreSQL JSONB]].

### Arrays (PostgreSQL)

```sql
CREATE TABLE posts (
    id SERIAL,
    tags TEXT[]
);

INSERT INTO posts (tags) VALUES (ARRAY['sql', 'tutorial']);
SELECT * FROM posts WHERE 'sql' = ANY(tags);
```

---

## 15. Best Practices

### Naming

```sql
-- ✅ Хорошо
users                  -- table — plural, lowercase, snake_case
user_id                -- column — singular, snake_case
idx_users_email        -- index — idx_<table>_<column>
fk_orders_user_id      -- foreign key — fk_<table>_<column>

-- ❌ Плохо
User                   -- inconsistent case
USERID                 -- ALL CAPS — unreadable
```

### Не используй SELECT *

```sql
-- ❌ Возвращает все колонки даже если не нужны
SELECT * FROM users;

-- ✅ Явно указать
SELECT id, name, email FROM users;
-- Преимущества: явный контракт, меньше I/O, видно что используется
```

### Always ORDER BY с LIMIT

```sql
-- ❌ Без ORDER BY — порядок неопределённый
SELECT * FROM users LIMIT 10;

-- ✅
SELECT * FROM users ORDER BY id LIMIT 10;
```

### Используй параметризованные queries (защита от SQL injection)

```csharp
// ❌ SQL injection risk!
var sql = $"SELECT * FROM users WHERE name = '{userInput}'";

// ✅ Параметризированный
var sql = "SELECT * FROM users WHERE name = @name";
cmd.Parameters.Add("@name", userInput);

// EF Core делает это автоматически
context.Users.Where(u => u.Name == userInput);
```

### Транзакция должна быть короткой

```sql
-- ❌ Длинная транзакция блокирует ресурсы
BEGIN;
SELECT * FROM huge_table;  -- 30 секунд!
UPDATE users SET ...;
COMMIT;

-- ✅ Сначала вычисли, потом transaction
-- (логику сделай в app)
```

### Index стратегия

- Индексы на колонки в `WHERE` (если selective)
- Индексы на FK
- Composite index по leftmost prefix rule
- Не индексировать колонки с low cardinality (boolean, enum) если не selective запрос

См. [[optimization|SQL Optimization]].

---

## 16. Когда какие БД (расширенно)

### PostgreSQL — golden default 2026

✅ **За:**
- ACID full
- JSONB — гибкость как у NoSQL
- RLS — многотенантность
- Full-text search встроен
- Extensions (PostGIS для geo, TimescaleDB для time-series)
- Open-source, без vendor lock-in
- Растущая популярность, много экспертов

❌ **Против:** более сложная настройка чем MySQL для простого VPS hosting.

### SQL Server / Azure SQL

✅ **За:**
- Microsoft ecosystem (DI, .NET, EF Core first-class)
- T-SQL мощный
- Excellent tooling (SSMS, Azure Data Studio)
- SQL Server Profiler, query plans

❌ **Против:**
- Платный (Express имеет ограничения)
- Vendor lock-in

### MySQL / MariaDB

✅ **За:**
- Простота, всё хостинги поддерживают
- Хорошо с web (LAMP stack)

❌ **Против:**
- Менее мощный SQL чем PG (нет CTE recursive до 8, нет proper JSON)
- Меньше extensions

### SQLite

✅ **За:**
- Embedded — один файл
- Идеально для desktop / mobile / прототипов
- Быстрый

❌ **Против:**
- Один writer одновременно (concurrent writes — нет)
- Не для multi-user web
- Без advanced features

### NoSQL (MongoDB / DynamoDB / Cosmos)

✅ **Когда:**
- Schema-less данные (документы разной структуры)
- Безумный scale (millions writes/sec)
- Geographic distribution

❌ **Когда нет:**
- Нужен ACID transactions
- Сложные JOINs / aggregations
- Strongly typed domain

---

## См. также

- [[optimization|SQL Optimization]] — индексы deep, EXPLAIN, query plans
- [[postgresql-deep|PostgreSQL Deep]] — RLS, JSONB, MVCC, advanced features
- [[../EFCore/basics-tracking|EF Core Basics]] — как EF translates LINQ to SQL
- [[../EFCore/queries-performance|EF Queries Performance]] — N+1, projection
- [[../EFCore/concurrency|EF Concurrency]] — isolation levels, locking
- [[../EFCore/migrations|EF Migrations]] — schema evolution

## Reading list

- **SQL Performance Explained** — Markus Winand (use-the-index-luke.com)
- **PostgreSQL Documentation** — postgresql.org/docs (best official docs)
- **Designing Data-Intensive Applications** — Kleppmann (1 раздел про БД)
- **Database Internals** — Alex Petrov (как БД работают изнутри)
- **High Performance MySQL** — O'Reilly (для MySQL focus)
- **SQL Antipatterns** — Bill Karwin (что не делать)
- **The Art of SQL** — Stephane Faroult (sql idiomatic patterns)
