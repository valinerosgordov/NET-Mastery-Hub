---
tags: [postgresql, sql, npgsql, jsonb, row-level-security, pgvector, indexes, partitioning]
level: Senior
---

# PostgreSQL Deep для .NET-разработчика

## Что это, зачем и когда

### Что такое PostgreSQL?
**Open-source реляционная СУБД с ACID, MVCC, расширяемой системой типов и индексов.** В 2026 — фактический стандарт для backend-приложений: от стартапов до Apple/Instagram. В .NET-мире — главный кандидат для PostgreSQL'а из коробки (через `Npgsql`).

**Аналогия:** SQL Server — это шкаф IKEA «всё из коробки, но платно». MySQL — сборная мебель за $30. PostgreSQL — модульный шкаф, можешь собрать что хочешь, расширения покрывают экзотические задачи (vector search, time-series, geo).

### Зачем именно Postgres

| | Postgres | SQL Server | MySQL |
|--|----------|------------|-------|
| Лицензия | Permissive (PostgreSQL License) | Платный (для prod-нагрузок) | GPL (страх ~~Oracle~~) |
| JSONB | Лучший в индустрии | JSON есть, индексы хуже | JSON есть, индексы слабее |
| Row-Level Security | Native, мощный | Native | Нет (только через VIEWs) |
| Vector search (RAG) | pgvector — стандарт | через CLR — костыль | нет нативно |
| Полнотекстовый поиск | Native (`tsvector`) | Native (`CONTAINS`) | Native (`MATCH AGAINST`) |
| Партиционирование | Declarative (PG 10+) | Native | Native |
| Расширения | 1000+ (pg_partman, citus, timescale, postgis) | Скромно | Скромно |
| Cloud-native | Все провайдеры (AWS RDS, Azure, Neon, Supabase) | Azure-first | Все |

### Когда **не** Postgres
- **Time-series at scale** — TimescaleDB (расширение Postgres) или ClickHouse / InfluxDB
- **Сверхбольшие OLAP** — ClickHouse
- **Document-first** без реляционной схемы — MongoDB
- **Embedded** в десктоп-приложении — SQLite (как DailyPlanner и Nuts)
- **Key-value** с TTL и пабсабом — Redis

---

## Npgsql — production patterns

### Connection string

```csharp
var connStr = "Host=localhost;Port=5432;Database=app;Username=app;Password=...;" +
              "Pooling=true;Maximum Pool Size=100;Minimum Pool Size=5;" +
              "Connection Idle Lifetime=300;Connection Pruning Interval=10;" +
              "Multiplexing=true;Max Auto Prepare=100;Auto Prepare Min Usages=2;" +
              "Application Name=NexusAI;" +
              "SSL Mode=Require;Trust Server Certificate=true;";
```

Ключевые параметры:
- `Pooling=true` — встроенный pool, всегда оставляй включённым
- `Maximum Pool Size=100` — обычно по числу ядер БД-сервера × 2-4
- `Multiplexing=true` — отправка нескольких команд через одно соединение (быстрее в high-throughput)
- `Max Auto Prepare=100` — кэш prepared statements (готовых запросов)
- `Application Name=NexusAI` — видно в `pg_stat_activity`, помогает дебажить

### NpgsqlDataSource (рекомендуется с .NET 7+)

```csharp
// Program.cs
var dataSource = new NpgsqlDataSourceBuilder(connStr)
    .EnableDynamicJson()           // для JSONB serialization
    .UseLoggerFactory(loggerFactory)
    .Build();

builder.Services.AddSingleton(dataSource);

// Usage
public class TaskRepository(NpgsqlDataSource dataSource)
{
    public async Task<Task?> GetAsync(Guid id, CancellationToken ct)
    {
        await using var conn = await dataSource.OpenConnectionAsync(ct);
        await using var cmd = new NpgsqlCommand("SELECT id, title FROM tasks WHERE id = @id", conn);
        cmd.Parameters.AddWithValue("id", id);

        await using var reader = await cmd.ExecuteReaderAsync(ct);
        if (!await reader.ReadAsync(ct)) return null;

        return new Task(reader.GetGuid(0), reader.GetString(1));
    }
}
```

`NpgsqlDataSource` — singleton, переиспользуется через всё приложение. **Не путай с `NpgsqlConnection`** — соединения создаёшь короткими через `OpenConnectionAsync()`.

### EF Core с PostgreSQL

```csharp
builder.Services.AddDbContextPool<AppDbContext>((sp, options) =>
{
    var dataSource = sp.GetRequiredService<NpgsqlDataSource>();
    options.UseNpgsql(dataSource, npgsql =>
    {
        npgsql.EnableRetryOnFailure(maxRetryCount: 3);
        npgsql.CommandTimeout(30);
    });

    options.UseSnakeCaseNamingConvention();  // Title -> title в БД
});
```

`AddDbContextPool` — пул `DbContext` для high-throughput сценариев (10x быстрее на простых запросах в high-RPS системах).

### Bulk copy — в десятки раз быстрее INSERT

```csharp
// Загрузка миллиона строк
await using var conn = await dataSource.OpenConnectionAsync(ct);
await using var writer = await conn.BeginBinaryImportAsync(
    "COPY tasks (id, title, due_date) FROM STDIN (FORMAT BINARY)", ct);

foreach (var task in tasks)
{
    await writer.StartRowAsync(ct);
    await writer.WriteAsync(task.Id, NpgsqlDbType.Uuid, ct);
    await writer.WriteAsync(task.Title, NpgsqlDbType.Text, ct);
    await writer.WriteAsync(task.DueDate, NpgsqlDbType.Timestamp, ct);
}

await writer.CompleteAsync(ct);
```

Для 1M строк: обычные `INSERT` — минуты, `COPY ... BINARY` — секунды.

### Pipelining

```csharp
// Несколько команд на одном round-trip
await using var batch = new NpgsqlBatch(conn)
{
    BatchCommands =
    {
        new("INSERT INTO logs (msg) VALUES (@msg)") { Parameters = { new("msg", "first") } },
        new("INSERT INTO logs (msg) VALUES (@msg)") { Parameters = { new("msg", "second") } },
        new("UPDATE counters SET value = value + 1") { },
    },
};
await batch.ExecuteNonQueryAsync(ct);
```

Один TCP round-trip → драматически быстрее на дальних БД (RDS из другого региона).

> [!question]- **Интервью: что такое multiplexing в Npgsql и когда применять?**
> Multiplexing — отправка команд от **разных logical connections** через **одно физическое соединение**. Включается в connection string.
>
> Когда применять:
> - High RPS приложения с короткими запросами (>1000 RPS)
> - Когда упираешься в pool (`Pool Exhaustion` ошибки)
>
> Когда **не**:
> - Длинные транзакции (multiplexing их не разруливает — они эксклюзивно занимают connection)
> - LISTEN/NOTIFY (он pinned к connection)
> - Когда используется `RaiseNotice` для прогресса
>
> По умолчанию выключен — это потому что несовместим с описанными выше use cases.

---

## JSONB — полу-структурированные данные

### JSONB vs JSON vs TEXT

| | TEXT | JSON | JSONB |
|--|------|------|-------|
| Хранение | Строка как есть | Строка с проверкой валидности | Бинарное представление (parsed) |
| Запросы | Только LIKE | По синтаксису + операторы | По синтаксису + операторы |
| Индексы | B-tree (whole text) | — | **GIN** (по полям!) |
| Производительность чтения | Быстро (raw) | Парсинг каждый раз | Pre-parsed |
| Производительность записи | Быстро | Чуть медленнее (парс) | Медленнее (бинарный кодинг) |
| Когда применять | Logs, raw markup | Никогда — выбирай JSONB | Документы, метаданные, конфиги |

**Правило:** в .NET-приложении на PG всегда `jsonb`. `json` — legacy, `text` — для строк.

### Операторы JSONB

```sql
-- Получить поле как JSONB
SELECT data->'profile' FROM users;

-- Получить поле как text (для сравнений)
SELECT data->>'email' FROM users WHERE data->>'email' = 'a@b.com';

-- Содержит подобъект
SELECT * FROM users WHERE data @> '{"role": "admin"}';

-- Существует ли ключ
SELECT * FROM users WHERE data ? 'phone';

-- Существует любой из ключей
SELECT * FROM users WHERE data ?| array['phone', 'email'];

-- Существуют все ключи
SELECT * FROM users WHERE data ?& array['phone', 'email'];

-- Удаление поля
UPDATE users SET data = data - 'phone' WHERE id = 1;

-- Установка вложенного поля
UPDATE users SET data = jsonb_set(data, '{profile,age}', '30', create_missing := true);
```

### Индексы для JSONB

```sql
-- GIN на всё поле — поддерживает @>, ?, ?|, ?&
CREATE INDEX idx_users_data ON users USING GIN (data);

-- GIN с jsonb_path_ops — компактнее, но только @>
CREATE INDEX idx_users_data_path ON users USING GIN (data jsonb_path_ops);

-- B-tree на extracted поле — для конкретного query pattern
CREATE INDEX idx_users_email ON users ((data->>'email'));

-- Partial index — для подмножества
CREATE INDEX idx_active_admins ON users ((data->>'role'))
WHERE data @> '{"active": true}';
```

### JSONB в EF Core с Npgsql

```csharp
public sealed class User
{
    public Guid Id { get; set; }
    public string Email { get; set; } = "";

    // Способ 1: typed JSON document
    public UserProfile Profile { get; set; } = new();

    // Способ 2: dynamic JsonDocument
    public JsonDocument? Metadata { get; set; }
}

public sealed class UserProfile
{
    public string FirstName { get; set; } = "";
    public string LastName { get; set; } = "";
    public List<string> Tags { get; set; } = new();
}

// OnModelCreating
modelBuilder.Entity<User>(b =>
{
    b.Property(u => u.Profile).HasColumnType("jsonb");
    b.Property(u => u.Metadata).HasColumnType("jsonb");
});
```

Запросы с EF Core (Npgsql предоставляет JSONB-aware translation):
```csharp
var admins = await db.Users
    .Where(u => EF.Functions.JsonContains(u.Profile, """{"role":"admin"}"""))
    .ToListAsync(ct);

// Доступ к полю
var firstNames = await db.Users
    .Select(u => u.Profile.FirstName)
    .ToListAsync(ct);
```

> [!question]- **Интервью: когда JSONB, когда отдельные колонки?**
> JSONB хорош для:
> - Полей, которые могут отличаться между записями (метаданные, профили)
> - Документов, которые читаются/пишутся целиком (конфиги, settings)
> - Прототипирования, когда схема ещё не устаканилась
>
> Отдельные колонки нужны, когда:
> - По полю часто фильтруют или сортируют (B-tree индекс эффективнее)
> - Поле требует FK constraint
> - Поле имеет фиксированный тип, проверки (numeric range, enum)
>
> На NexusAI / [anonymized] используется гибрид: критичные поля (id, created_at, tenant_id) — отдельные колонки, неструктурированные опции — JSONB.

---

## Row-Level Security (RLS) — multi-tenant из коробки

**RLS позволяет настроить policy: какие строки видит конкретный пользователь сессии.** Это **не application-level фильтрация** (`WHERE tenant_id = @tid`), а уровень БД — даже если разработчик забыл фильтр, БД сама не вернёт чужие данные.

В NexusAI это **locked architecture decision (ADR-0001)** — multi-tenancy from day one без opt-out.

### Базовая настройка

```sql
-- Таблица с tenant_id
CREATE TABLE presentations (
    id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   uuid NOT NULL,
    title       text NOT NULL,
    created_at  timestamptz DEFAULT now()
);

-- Включаем RLS
ALTER TABLE presentations ENABLE ROW LEVEL SECURITY;

-- Создаём policy
CREATE POLICY tenant_isolation ON presentations
USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

Теперь любой `SELECT FROM presentations` отфильтрован по `app.tenant_id` сессии. Без правильно установленного setting'а — пустой результат.

### Установка tenant_id из приложения

```csharp
public class TenantConnectionInterceptor(ITenantContext tenantContext)
    : DbConnectionInterceptor
{
    public override async ValueTask<DbConnection> ConnectionOpenedAsync(
        DbConnection connection,
        ConnectionEndEventData eventData,
        CancellationToken ct = default)
    {
        var tenantId = tenantContext.TenantId;
        if (tenantId is null)
            throw new InvalidOperationException("Tenant id is required for any DB operation.");

        var npgsql = (NpgsqlConnection)connection;
        await using var cmd = npgsql.CreateCommand();
        cmd.CommandText = $"SET app.tenant_id = '{tenantId}'";  // безопасно — мы валидируем что это Guid
        await cmd.ExecuteNonQueryAsync(ct);

        return connection;
    }
}

// Регистрация
builder.Services.AddScoped<ITenantContext, TenantContext>();

builder.Services.AddDbContext<AppDbContext>((sp, opts) =>
{
    opts.UseNpgsql(connStr);
    opts.AddInterceptors(sp.GetRequiredService<TenantConnectionInterceptor>());
});

// TenantContext извлекается из JWT/HttpContext
public sealed class TenantContext(IHttpContextAccessor http) : ITenantContext
{
    public Guid? TenantId
    {
        get
        {
            var claim = http.HttpContext?.User.FindFirst("tenant_id")?.Value;
            return Guid.TryParse(claim, out var id) ? id : null;
        }
    }
}
```

### Bypass для системных операций

Иногда нужен legitimate доступ ко всем строкам (миграции, фоновые задачи, админ-эндпойнты). Решения:

**1. Атрибут BYPASSRLS на пользователе БД:**
```sql
CREATE ROLE app_admin WITH LOGIN BYPASSRLS PASSWORD '...';
```
Этот пользователь игнорирует все policy. Используй **только** для миграций и админ-таскo.

**2. SECURITY DEFINER function:**
```sql
CREATE FUNCTION admin_get_all_presentations()
RETURNS TABLE (id uuid, tenant_id uuid, title text)
SECURITY DEFINER
LANGUAGE sql
AS $$ SELECT id, tenant_id, title FROM presentations $$;
```
Функция выполняется с правами того кто её создал (не вызывает) — игнорирует RLS если creator имеет BYPASSRLS.

### INSERT-policy и UPDATE-policy

```sql
-- USING — для SELECT/UPDATE/DELETE: какие строки видим
-- WITH CHECK — для INSERT/UPDATE: какие значения можно записать
CREATE POLICY tenant_isolation ON presentations
USING (tenant_id = current_setting('app.tenant_id')::uuid)
WITH CHECK (tenant_id = current_setting('app.tenant_id')::uuid);
```

**Без `WITH CHECK`:** клиент может вставить запись с **другим** tenant_id (дыра!).

> [!question]- **Интервью: какие подводные камни RLS в production?**
> 1. **`SECURITY DEFINER` functions обходят RLS** — если ты вызываешь такую функцию из app-code, RLS не сработает. Применяй с осторожностью.
> 2. **Connection pooling** — если соединения переиспользуются, `SET app.tenant_id` от предыдущего пользователя может «утечь». Решение: всегда устанавливай setting на каждом open connection (через interceptor), или используй `SET LOCAL` в транзакции (rollback после `COMMIT`).
> 3. **Performance** — RLS добавляет фильтр в каждый план запроса. На сложных запросах (несколько JOIN с RLS-таблицами) это может удвоить план. Пиши explicit `tenant_id = ...` filters в запросах + RLS как safety-net.
> 4. **Миграции** — обычные миграции от пользователя без BYPASSRLS могут не суметь обновить чужие строки. Запускай миграции от admin-пользователя.

---

## Партиционирование

Партиционирование — разделение большой таблицы на физические "куски" (partitions) по ключу. Postgres в plan'е читает только нужные куски, что ускоряет запросы и упрощает управление (легче drop'ать старые данные).

### Когда нужно

| Сигнал | Решение |
|--------|---------|
| Таблица > 100M строк, queries фильтруют по диапазону (date) | Range partitioning |
| Multi-tenant с большими тенантами | List partitioning по tenant_id |
| Партишены растут равномерно, нужен parallel insert | Hash partitioning |
| Таблица < 10M строк, нет проблем с performance | **Не нужно** — партиционирование добавляет complexity |

### Range partitioning по дате

```sql
-- Главная таблица (partitioned)
CREATE TABLE events (
    id          uuid NOT NULL,
    occurred_at timestamptz NOT NULL,
    payload     jsonb,
    PRIMARY KEY (id, occurred_at)  -- partition key должен быть в PK
) PARTITION BY RANGE (occurred_at);

-- Партишены по месяцам
CREATE TABLE events_2026_04 PARTITION OF events
    FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
CREATE TABLE events_2026_05 PARTITION OF events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Default партишен — куда падают записи вне диапазонов
CREATE TABLE events_default PARTITION OF events DEFAULT;

-- Индекс на каждом партишене
CREATE INDEX ON events_2026_04 (occurred_at);
CREATE INDEX ON events_2026_05 (occurred_at);
```

### Автоматическое создание партишенов через pg_partman

```sql
CREATE EXTENSION pg_partman;

SELECT partman.create_parent(
    p_parent_table := 'public.events',
    p_control := 'occurred_at',
    p_type := 'native',
    p_interval := 'monthly',
    p_premake := 4   -- предсоздать партишены на 4 месяца вперёд
);

-- Setup retention — удалять партишены старше 6 месяцев
UPDATE partman.part_config
SET retention = '6 months', retention_keep_table = false
WHERE parent_table = 'public.events';

-- Cron-подобный maintenance (запускается по расписанию)
SELECT partman.run_maintenance('public.events');
```

### Drop старых партишенов = O(1)

```sql
-- Старый способ для не-партиционированной таблицы — медленно
DELETE FROM events WHERE occurred_at < '2025-01-01';
VACUUM events;

-- С партишенами — мгновенно
DROP TABLE events_2024_12;
```

---

## Window functions deep

Window functions — агрегации **без коллапса строк**: считаем значения по окну (части набора), но возвращаем все исходные строки.

### Базовые: ROW_NUMBER, RANK, DENSE_RANK

```sql
-- Top-3 заказа на пользователя
SELECT id, user_id, amount, created_at,
       ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY amount DESC) AS rn
FROM orders
WHERE rn <= 3;
```

| Function | Поведение для одинаковых значений |
|----------|----------------------------------|
| `ROW_NUMBER()` | 1, 2, 3, 4 (всегда уникальные) |
| `RANK()` | 1, 2, 2, 4 (пропуск после ties) |
| `DENSE_RANK()` | 1, 2, 2, 3 (нет пропусков) |
| `PERCENT_RANK()` | от 0 до 1, по позиции |
| `NTILE(n)` | разбиение на n равных квантилей |

### LAG / LEAD — соседние строки

```sql
-- Считаем разницу с предыдущим тиком
SELECT
    occurred_at,
    bid,
    bid - LAG(bid) OVER (ORDER BY occurred_at) AS bid_delta,
    LEAD(bid) OVER (ORDER BY occurred_at) - bid AS bid_next_delta
FROM ticks
ORDER BY occurred_at;
```

Без LAG — нужен self-join, медленно. С LAG — оптимизатор делает один проход.

### Frame: ROWS / RANGE / GROUPS BETWEEN

```sql
-- Скользящее среднее за 7 дней
SELECT day, total_orders,
       AVG(total_orders) OVER (
           ORDER BY day
           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ) AS rolling_avg_7d
FROM daily_stats;

-- Cumulative sum
SELECT day, revenue,
       SUM(revenue) OVER (
           PARTITION BY EXTRACT(YEAR FROM day)
           ORDER BY day
           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS ytd_revenue
FROM daily_stats;
```

> [!question]- **Интервью: разница между ROWS BETWEEN и RANGE BETWEEN?**
> `ROWS` — фиксированное число физических строк (X PRECEDING / X FOLLOWING).
> `RANGE` — все строки с **значением** в диапазоне (по ORDER BY column).
>
> Пример: при ORDER BY date, если есть несколько строк на одну дату:
> - `ROWS BETWEEN 1 PRECEDING AND CURRENT ROW` — текущая + ровно 1 предыдущая физическая строка
> - `RANGE BETWEEN 1 DAY PRECEDING AND CURRENT ROW` — все строки за последние сутки (включая ties на текущей дате)
>
> `GROUPS` (PG 11+) — группы из равных значений ORDER BY (не строк).

---

## CTE и Recursive CTE

### Базовый CTE (Common Table Expression)

```sql
WITH recent_orders AS (
    SELECT * FROM orders WHERE created_at > now() - interval '7 days'
)
SELECT user_id, COUNT(*) FROM recent_orders GROUP BY user_id;
```

CTE как `temp view` — улучшает читаемость. **Раньше** были materialized по-умолчанию — медленные. С PG 12+ CTE inline'ятся в основной запрос (как subquery), производительность как у subquery.

### Recursive CTE — иерархии

```sql
-- Все потомки в дереве
WITH RECURSIVE descendants AS (
    -- ANCHOR — стартовая точка
    SELECT id, parent_id, name, 1 AS depth
    FROM categories
    WHERE id = '00000000-0000-0000-0000-000000000001'

    UNION ALL

    -- RECURSIVE — итеративно расширяем
    SELECT c.id, c.parent_id, c.name, d.depth + 1
    FROM categories c
    JOIN descendants d ON c.parent_id = d.id
)
SELECT * FROM descendants ORDER BY depth, name;
```

### Recursive CTE для генерации серий

```sql
-- Все даты за месяц
WITH RECURSIVE dates AS (
    SELECT '2026-04-01'::date AS d
    UNION ALL
    SELECT d + 1 FROM dates WHERE d < '2026-04-30'
)
SELECT d FROM dates;

-- Альтернатива через generate_series (нативная функция, проще)
SELECT generate_series('2026-04-01'::date, '2026-04-30'::date, '1 day'::interval);
```

---

## EXPLAIN ANALYZE — план выполнения

### Базовый разбор

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)
SELECT u.email, COUNT(o.id) AS orders
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2026-01-01'
GROUP BY u.email;
```

Вывод:
```
HashAggregate  (cost=15234.50..15334.50 rows=10000 width=44) (actual time=125.123..127.456 rows=8423 loops=1)
  Group Key: u.email
  Buffers: shared hit=1234 read=567
  ->  Hash Right Join  (cost=234.00..14234.50 rows=200000 width=36) (actual time=2.345..98.123 rows=156234 loops=1)
        Hash Cond: (o.user_id = u.id)
        ->  Seq Scan on orders o  (cost=0.00..12000.00 rows=200000 width=24) (actual time=0.012..45.678 rows=200000 loops=1)
        ->  Hash  (cost=200.00..200.00 rows=10000 width=20) (actual time=2.300..2.300 rows=10000 loops=1)
              ->  Seq Scan on users u  (cost=0.00..200.00 rows=10000 width=20) (actual time=0.005..1.234 rows=10000 loops=1)
                    Filter: (created_at > '2026-01-01'::date)
                    Rows Removed by Filter: 5000
Planning Time: 0.345 ms
Execution Time: 128.123 ms
```

### Что искать

| Что | Что значит | Что делать |
|-----|-----------|-----------|
| `Seq Scan` на большой таблице | Полный скан вместо индекса | Добавь индекс / убери функцию из WHERE |
| Большой `Rows Removed by Filter` | Индекс не помог отфильтровать | Composite index по фильтру и ORDER BY |
| `actual rows` >> estimated `rows` | Statistics устарели | `ANALYZE table_name;` |
| `Buffers: shared read=X` | X страниц с диска (не из cache) | Прогрев или больше shared_buffers |
| `Sort` или `Hash` с диском | Не влезло в work_mem | Подними `work_mem` |
| `Nested Loop` на больших таблицах | Часто catastrophic | Хочешь Hash или Merge Join → ANALYZE |

### pg_stat_statements

```sql
CREATE EXTENSION pg_stat_statements;

-- Топ-10 самых медленных запросов
SELECT query, calls, total_exec_time / calls AS avg_ms, mean_exec_time, max_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

Это **первое что делаешь** на тормозящей системе — ищешь топ запросов по `total_exec_time`. Не смотри только `mean_exec_time` — может быть запрос на 10ms, но за час вызывается миллион раз.

### Plan caching pitfall

`PREPARE` или Npgsql auto-prepare кэширует план на основе **первого вызова**. Если параметр сильно влияет на selectivity — может быть subтоптimal:

```csharp
// Первый вызов с маленьким tenant — план через index scan
// Затем тот же prepared с большим tenant — план не пересчитан, всё ещё index scan по всем строкам tenant'а
await cmd.ExecuteAsync(new { tenantId = smallTenant });
await cmd.ExecuteAsync(new { tenantId = bigTenant });  // медленно!
```

Решение — `npgsql.UsePlanFlushAfter(...)` или `Max Auto Prepare=0` для problematic queries.

---

## Индексы — детали

### B-tree (default)

Балансированное дерево, поддерживает: `=`, `<`, `<=`, `>`, `>=`, `BETWEEN`, `IN`, `IS NULL`. Используй для большинства колонок.

```sql
-- Composite — порядок важен!
CREATE INDEX ON orders (user_id, created_at DESC);

-- Запросы которые выиграют:
-- WHERE user_id = X (только первый столбец)
-- WHERE user_id = X AND created_at > ... (оба)
-- WHERE user_id = X ORDER BY created_at DESC

-- Запрос НЕ выиграет:
-- WHERE created_at > ... (без user_id) — composite не работает
```

### Partial index

```sql
-- Только активные пользователи (если 90% inactive — индекс в 10x меньше)
CREATE INDEX ON users (email) WHERE is_active = true;

-- Только undeleted
CREATE INDEX ON tasks (created_at) WHERE deleted_at IS NULL;
```

### Expression index

```sql
-- Поиск по lower-case email (case-insensitive)
CREATE INDEX ON users (LOWER(email));
SELECT * FROM users WHERE LOWER(email) = LOWER('A@B.com');
```

### Include columns (covering index)

```sql
-- B-tree с дополнительными "несортированными" столбцами в листьях
CREATE INDEX ON orders (user_id) INCLUDE (total, status);
-- Запросы только по этим столбцам не делают heap fetch — быстрее
SELECT total, status FROM orders WHERE user_id = X;
```

### GIN — для массивов, JSONB, full-text

```sql
CREATE INDEX ON articles USING GIN (tags);          -- text[] tags
CREATE INDEX ON articles USING GIN (metadata);      -- jsonb
CREATE INDEX ON articles USING GIN (search_tsv);    -- full-text
```

### BRIN — time-series

Block Range Index — хранит min/max для блоков таблицы. Маленький, эффективен для столбцов, **которые коррелируют с физическим порядком** (timestamp в insertion-order).

```sql
CREATE INDEX ON events USING BRIN (occurred_at) WITH (pages_per_range = 128);
```

100x меньше B-tree, годен только для approximate range scans, но на time-series данных эффективен.

### HNSW / IVFFlat — для pgvector

Подробнее ниже.

---

## Полнотекстовый поиск

### tsvector / tsquery

```sql
-- Колонка для индексации (генерируемая)
ALTER TABLE articles ADD COLUMN search_tsv tsvector
    GENERATED ALWAYS AS (
        to_tsvector('russian', coalesce(title, '') || ' ' || coalesce(body, ''))
    ) STORED;

CREATE INDEX ON articles USING GIN (search_tsv);

-- Запрос
SELECT id, title
FROM articles
WHERE search_tsv @@ to_tsquery('russian', 'постгрес & оптимизация')
ORDER BY ts_rank(search_tsv, to_tsquery('russian', 'постгрес & оптимизация')) DESC
LIMIT 10;
```

### Конфигурации (dictionaries)

```sql
SELECT * FROM pg_ts_config;  -- список доступных
-- english, russian, simple — стандартные
-- Можно создавать свои (с собственными стеммерами, синонимами)
```

### Highlighting

```sql
SELECT id, ts_headline('russian', body, query, 'StartSel=<mark>, StopSel=</mark>')
FROM articles, to_tsquery('russian', 'постгрес') query
WHERE search_tsv @@ query;
```

---

## pgvector — vector similarity

### Установка и базовое использование

```sql
CREATE EXTENSION vector;

CREATE TABLE chunks (
    id          uuid PRIMARY KEY,
    content     text NOT NULL,
    embedding   vector(1536)  -- OpenAI text-embedding-3-small
);
```

### Distance functions

| Оператор | Что считает | Когда |
|----------|------------|-------|
| `<->` | L2 (Euclidean) | Когда модель НЕ нормализует векторы |
| `<=>` | Cosine distance | Стандарт для embedding-моделей (OpenAI, Cohere, etc.) |
| `<#>` | Negative inner product | Когда нормализация и важна скорость (нет вычисления нормы) |

```sql
-- Top-10 семантически близких к query embedding
SELECT id, content, embedding <=> '[0.1, 0.2, ...]'::vector AS distance
FROM chunks
ORDER BY embedding <=> '[0.1, 0.2, ...]'::vector
LIMIT 10;
```

### Index — HNSW vs IVFFlat

| | HNSW | IVFFlat |
|--|------|---------|
| Тип | Hierarchical Navigable Small World (graph) | Inverted File с k-means кластеризацией |
| Память | Больше | Меньше |
| Build time | Медленно | Быстро |
| Query time | Очень быстро | Быстро |
| Recall | Высокий (>95% при правильных параметрах) | Зависит от probes |
| Когда | Production, low-latency | Когда HNSW не помещается в RAM |

```sql
-- HNSW (рекомендую по умолчанию для < 10M векторов)
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- IVFFlat — нужно ANALYZE сначала, чтобы ML работал
CREATE INDEX ON chunks USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- При запросе — настройка точности vs скорость
SET hnsw.ef_search = 100;       -- HNSW
SET ivfflat.probes = 10;        -- IVFFlat
```

> [!question]- **Интервью: как выбрать `lists` для IVFFlat?**
> Эмпирическое правило: `lists = sqrt(rows)` для < 1M строк, `lists = rows / 1000` для > 1M.
> Например, 100K векторов → lists ~316. После создания индекса — `SET ivfflat.probes = sqrt(lists)` (10-30 обычно даёт recall > 95%).
> ⚠️ IVFFlat зависит от ANALYZE — после большого insert/delete нужно `REINDEX`.

---

## Транзакции и блокировки

### Isolation levels

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
-- или
BEGIN ISOLATION LEVEL SERIALIZABLE;
```

| | Read Uncommitted | Read Committed | Repeatable Read | Serializable |
|--|------------------|----------------|-----------------|--------------|
| Dirty read | Возможен | Нет | Нет | Нет |
| Non-repeatable read | Возможен | Возможен | Нет | Нет |
| Phantom read | Возможен | Возможен | Возможен в стандарте, **нет в Postgres** | Нет |
| Concurrency | Максимум | Хорошо | Хуже | Хуже всего (откаты!) |

**Postgres default — Read Committed.** Для большинства приложений достаточно.

### SELECT ... FOR UPDATE / FOR SHARE

```sql
BEGIN;

-- Лочим строку для эксклюзивного апдейта (другие транзакции ждут)
SELECT balance FROM accounts WHERE id = $1 FOR UPDATE;

-- Бизнес-логика на стороне приложения
-- ...

UPDATE accounts SET balance = balance - 100 WHERE id = $1;

COMMIT;
```

`FOR UPDATE SKIP LOCKED` — пропускать залоченные строки (классический паттерн job queue).

### Advisory locks — application-level locks через БД

```sql
-- Acquire (NULL если не получилось)
SELECT pg_try_advisory_lock(12345);

-- Обязательно release в той же сессии
SELECT pg_advisory_unlock(12345);

-- Или transaction-scoped — auto-release при commit/rollback
SELECT pg_advisory_xact_lock(12345);
```

Применение: distributed locking (только один воркер обрабатывает определённый ID), prevent concurrent migrations, etc.

### Optimistic concurrency через row-version (xmin)

```sql
-- Postgres хранит ID транзакции последней мутации в системной колонке xmin
SELECT id, balance, xmin FROM accounts WHERE id = $1;
-- ... апдейт с проверкой
UPDATE accounts SET balance = $1 WHERE id = $2 AND xmin = $3;
-- Если 0 rows updated — кто-то изменил между read и write
```

EF Core поддерживает: `IsRowVersion` mapped к `xmin`:
```csharp
modelBuilder.Entity<Account>()
    .Property("xmin")
    .HasColumnType("xid")
    .ValueGeneratedOnAddOrUpdate()
    .IsConcurrencyToken();
```

---

## VACUUM / autovacuum / bloat

В Postgres `UPDATE` не меняет строку in-place — создаёт новую версию (MVCC). Старые версии (dead tuples) удаляет VACUUM. Без него таблица растёт, индексы пухнут, query plans портятся.

### Что показывает bloat

```sql
-- Через pgstattuple
CREATE EXTENSION pgstattuple;
SELECT * FROM pgstattuple('orders');
-- dead_tuple_percent > 20% — пора VACUUM

-- Размер таблицы и индексов
SELECT pg_size_pretty(pg_relation_size('orders')) AS table_size,
       pg_size_pretty(pg_indexes_size('orders')) AS indexes_size,
       pg_size_pretty(pg_total_relation_size('orders')) AS total;
```

### Manual VACUUM в проде

```sql
-- Cleanup dead tuples (default)
VACUUM orders;

-- Compact indexes (medium) — может блокировать
VACUUM (FULL) orders;  -- НЕ запускай в проде, exclusive lock

-- Аналитика — обновляет статистику для планировщика
ANALYZE orders;

-- Совмещённый
VACUUM (ANALYZE) orders;
```

### Autovacuum tuning

```sql
-- Per-table
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.1,    -- VACUUM когда 10% таблицы dead
    autovacuum_analyze_scale_factor = 0.05   -- ANALYZE когда 5% изменено
);
```

Для high-write таблиц — настрой агрессивнее (default 20% vacuum часто слишком поздно).

---

## Production checklist

- [ ] `Pooling=true` в connection string
- [ ] `NpgsqlDataSource` как singleton (`AddDbContextPool` для high-throughput)
- [ ] `ApplicationName=` указано — видно в pg_stat_activity
- [ ] `pg_stat_statements` extension включено
- [ ] `EnableRetryOnFailure` для transient errors
- [ ] Все JSONB колонки через `jsonb` (не `json`)
- [ ] RLS включена на multi-tenant таблицах + interceptor для setting tenant_id
- [ ] Composite индексы по фактическим WHERE + ORDER BY паттернам
- [ ] Partial indexes для разреженных bool-полей (`is_active`, `deleted_at IS NULL`)
- [ ] Партиционирование для time-series таблиц > 50M строк
- [ ] Backup автоматический (pg_dump per day) + WAL-archiving для PITR
- [ ] `autovacuum` мониторится, тяжёлые таблицы с настроенным scale_factor
- [ ] `shared_buffers` = 25% RAM, `effective_cache_size` = 75% RAM (на dedicated сервере)
- [ ] `work_mem` достаточно чтобы Hash/Sort не уходили на диск (см. EXPLAIN BUFFERS)
- [ ] SSL включён (`SSL Mode=Require` в conn string)
- [ ] Slow query log включён (`log_min_duration_statement = 1000`)

---

## Common pitfalls

### 1. Connection leak

```csharp
// ❌ NpgsqlConnection не задиспоузен — pool exhaustion
var conn = new NpgsqlConnection(connStr);
conn.Open();
var cmd = conn.CreateCommand();
// exception — pool занят навсегда

// ✅
await using var conn = await dataSource.OpenConnectionAsync();
```

### 2. EF + JSONB-фильтр без EF.Functions

```csharp
// ❌ Падает в runtime — выражение не транслируется
var users = db.Users.Where(u => u.Profile.Tags.Contains("admin")).ToList();

// ✅ Через EF.Functions для JSONB-aware translation
var users = db.Users.Where(u => EF.Functions.JsonContains(u.Profile, """{"tags":["admin"]}""")).ToList();
```

### 3. Boolean column без индекса в partial query

```sql
-- ❌ Полный scan на is_published = true (если 90% true)
SELECT * FROM articles WHERE is_published = true ORDER BY created_at DESC LIMIT 10;

-- ✅ Partial index
CREATE INDEX ON articles (created_at DESC) WHERE is_published = true;
```

### 4. UPDATE на много строк без LIMIT

```sql
-- ❌ Длинная блокировка, риск deadlock
UPDATE orders SET status = 'archived' WHERE created_at < '2024-01-01';

-- ✅ Батчами
WITH chunk AS (
    SELECT id FROM orders WHERE created_at < '2024-01-01' AND status != 'archived'
    LIMIT 10000 FOR UPDATE
)
UPDATE orders SET status = 'archived' WHERE id IN (SELECT id FROM chunk);
-- Цикл из приложения пока RowsAffected > 0
```

### 5. Использование `OFFSET` для пагинации больших таблиц

```sql
-- ❌ Медленно с ростом offset (O(N))
SELECT * FROM orders ORDER BY created_at DESC OFFSET 1000000 LIMIT 50;

-- ✅ Cursor-based
SELECT * FROM orders WHERE created_at < $last_seen ORDER BY created_at DESC LIMIT 50;
```

### 6. ENUM type — pain при добавлении значения

```sql
-- ❌ Postgres ENUM — менять можно только ADD VALUE, нельзя rename/remove без переезда
CREATE TYPE order_status AS ENUM ('pending', 'paid', 'cancelled');

-- ✅ Domain или CHECK + text — гибче
CREATE TABLE orders (..., status text NOT NULL CHECK (status IN ('pending', 'paid', 'cancelled')));
```

### 7. `now()` vs `clock_timestamp()` в долгих транзакциях

```sql
-- now() = transaction_timestamp() — фиксируется в начале транзакции
-- clock_timestamp() — реальное текущее время
SELECT now(), pg_sleep(2), now();        -- одинаковые
SELECT clock_timestamp(), pg_sleep(2), clock_timestamp();  -- разные
```

### 8. Недооценка важности `ANALYZE` после bulk-insert
После `COPY ... FROM` миллион строк — статистика устарела. Запросы получают плохие планы. Запускай `ANALYZE table_name;` сразу после bulk import.

---

## См. также

- [SQL Optimization](optimization.md) — общие принципы планов выполнения, индексы, join-алгоритмы
- [EF Core Queries и Performance]() — N+1, проекции, compiled queries
- [EF Core Concurrency]() — optimistic concurrency, transactions, retry
- [LLM / RAG patterns]() — pgvector в реальной RAG-системе
- [Semantic Kernel]() — pgvector через Microsoft.Extensions.AI

## Reading list

- **Postgres docs** — postgresql.org/docs (особенно Performance Tuning, Indexes, Concurrency Control)
- **Use The Index, Luke!** — use-the-index-luke.com (фундаментальная книга по индексам, бесплатно онлайн)
- **PostgreSQL Internals** — postgrespro.com/community/books/internals (бесплатная книга по внутренностям PG)
- **Crunchy Data blog** — crunchydata.com/blog (real-world case studies)
- **Lukas Eder — jOOQ blog** — для глубоких SQL-патернов и сравнений с другими СУБД
- **Bruce Momjian talks** — momjian.us/main/presentations.html (легенда PG-сообщества, отличные слайды)
- **Postgres Weekly newsletter** — postgresweekly.com
