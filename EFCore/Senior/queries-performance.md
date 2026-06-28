---
tags: [ef-core, queries, performance, n-plus-1, projection, compiled-queries, bulk]
level: Senior
---

# EF Core — Queries и Performance

> Performance-разбор read-side EF Core: устранение N+1 через `Include`/projection/split queries, `AsNoTracking`, compiled queries, `ExecuteUpdate`/`ExecuteDelete`, bulk insert, keyset pagination и DbContext pooling.

## Что это, зачем и когда

### Что такое EF Core query?
Любая операция чтения через EF Core (`db.Users.Where(...)`, `FirstOrDefaultAsync`, `ToListAsync`). EF транслирует C# в SQL через Expression Tree → LINQ provider Postgres/SQL Server → SQL запрос → возвращает entities.

**Аналогия:** EF — это переводчик с русского на английский. Ты пишешь "найди пользователей старше 18", EF говорит DB "SELECT * FROM users WHERE age > 18". Большинство performance-проблем — это плохо переведённые "русские" фразы (LINQ-выражения), которые DB не может оптимизировать.

### Почему performance важен
- N+1 query — самая частая причина "почему API тормозит"
- Один лишний `.ToListAsync()` после фильтра = full table scan
- Cartesian explosion в `Include` может вернуть миллионы строк вместо тысячи
- На staging всё ОК (10 records), на проде падает (10M)

---

## N+1 Problem

### Что это

```csharp
// 1 запрос: SELECT * FROM Users
var users = await db.Users.ToListAsync();

// N запросов: для каждого user — SELECT * FROM Orders WHERE UserId = X
foreach (var user in users)
{
    var orders = user.Orders.ToList();  // Lazy load каждого user!
}
```

100 пользователей → **101 SQL запрос**. На production → DB горит.

### Решения

```csharp
// 1. Eager loading — Include
var users = await db.Users
    .Include(u => u.Orders)
    .ToListAsync();
// Один запрос с JOIN

// 2. Explicit projection — выбираем только нужное
var data = await db.Users
    .Select(u => new
    {
        u.Id, u.Name,
        OrderCount = u.Orders.Count(),
        LastOrderDate = u.Orders.Max(o => (DateTime?)o.CreatedAt)
    })
    .ToListAsync();
// Один запрос с агрегацией
```

### Detection

Включи sensitive logging в Development:
```csharp
options.EnableSensitiveDataLogging();
options.LogTo(Console.WriteLine, LogLevel.Information);
```

В логах увидишь сколько запросов → если десятки на одну операцию — N+1.

В .NET 8+ EF Core по умолчанию **варнингает** в logs если detected:
```
warn: Microsoft.EntityFrameworkCore.Query[20504]
      Compiling a query which loads related collections for more than one collection navigation...
```

> [!question]- **Интервью: что такое N+1 и как с ним бороться?**
> Запрос к коллекции (1 запрос), затем lazy-load связанной для каждого элемента (N запросов).
> Решения:
> 1. **`Include(...)`** — eager loading, один запрос с JOIN. Минус: cartesian explosion при множественных collection-navigations
> 2. **Projection через `.Select(...)`** — выбираем только нужные поля. Самое эффективное
> 3. **Split queries** — `.AsSplitQuery()` для multiple collection includes (без cartesian, но 2+ запроса)
> 4. **Batch loading** — собрал IDs, потом одним запросом достал related: `var orders = await db.Orders.Where(o => userIds.Contains(o.UserId)).ToListAsync()`

---

## Projection — selective loading

### Зачем

```csharp
// ❌ Тащим всю entity (50 полей включая большие text-поля)
var user = await db.Users.FirstAsync(u => u.Id == id);
return new { user.Id, user.Name };

// ✅ Запрос только нужных полей
var user = await db.Users
    .Where(u => u.Id == id)
    .Select(u => new { u.Id, u.Name })
    .FirstAsync();
```

DB читает меньше данных с диска, в memory загружается меньше.

### DTO projection

```csharp
public sealed record UserDto(Guid Id, string Name, string Email, int OrderCount);

var users = await db.Users
    .Select(u => new UserDto(
        u.Id,
        u.Name,
        u.Email,
        u.Orders.Count(o => !o.IsCancelled)
    ))
    .ToListAsync();
```

EF транслирует это в **один SQL** с подзапросом для `OrderCount`. Возвращает только 4 поля, не вся entity.

### AutoMapper / Mapperly + Project

```csharp
// AutoMapper
var users = await db.Users
    .ProjectTo<UserDto>(_mapper.ConfigurationProvider)
    .ToListAsync();

// Mapperly
[Mapper]
public partial class UserMapper
{
    public partial IQueryable<UserDto> ProjectToDto(IQueryable<User> query);
}

var users = await mapper.ProjectToDto(db.Users).ToListAsync();
```

`ProjectTo` транслируется в SQL — никаких `.ToList()` потом `.Select()` в памяти.

---

## Include — eager loading

### Базовый Include

```csharp
var orders = await db.Orders
    .Include(o => o.Customer)        // 1-to-1 / many-to-1
    .Include(o => o.OrderItems)      // 1-to-many
        .ThenInclude(oi => oi.Product)  // дальше по цепочке
    .Where(o => o.CreatedAt > DateTime.Today.AddDays(-7))
    .ToListAsync();
```

### Cartesian Explosion

```csharp
var users = await db.Users
    .Include(u => u.Orders)        // 100 orders per user
    .Include(u => u.Reviews)       // 100 reviews per user
    .ToListAsync();
```

DB возвращает 100 × 100 = **10,000 строк** на каждого user (cross product). EF потом дедупит в памяти, но SQL уже отжарил DB.

При **двух+ collection includes** — переключайся на split query:

```csharp
var users = await db.Users
    .Include(u => u.Orders)
    .Include(u => u.Reviews)
    .AsSplitQuery()
    .ToListAsync();
// Теперь 3 запроса: users, orders WHERE UserId IN (...), reviews WHERE UserId IN (...)
```

### Глобально включить split queries

```csharp
options.UseNpgsql(connStr, b =>
{
    b.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery);
});
```

Default — single query. Лучше явно ставить per-query.

### Filtered Include (.NET 5+)

```csharp
var users = await db.Users
    .Include(u => u.Orders.Where(o => !o.IsCancelled).OrderByDescending(o => o.CreatedAt).Take(10))
    .ToListAsync();
```

Загружаем не все Orders, а топ-10 не-cancelled. **Только в Include, не в дальнейших операциях**.

### Когда Include vs Projection

| | Include | Projection |
|--|---------|------------|
| Когда | Нужна вся entity для writes | Нужны только некоторые поля для read |
| Tracking | По умолчанию tracked | Anonymous types — untracked |
| Performance | OK | Лучше |
| Cartesian explosion | Возможен | Нет |
| Use case | Сложные UI редактирующие entity | API endpoints, dashboards, lists |

**Правило:** для read-only API — projection. Для CRUD UI — Include.

---

## LeftJoin / RightJoin (EF Core 10)

EF Core 10 даёт first-class операторы `Queryable.LeftJoin` и `Queryable.RightJoin`, которые транслируются напрямую в SQL `LEFT JOIN` / `RIGHT JOIN`.

```csharp
// ✅ EF Core 10 — явный LEFT JOIN
var rows = await db.Users
    .LeftJoin(
        db.Orders,
        u => u.Id,
        o => o.UserId,
        (u, o) => new { u.Id, u.Name, OrderId = (Guid?)o.Id })
    .ToListAsync();
// SQL: SELECT ... FROM users u LEFT JOIN orders o ON u.id = o.user_id
```

> [!warning] Снимай legacy GroupJoin-обходной приём при апгрейде
> До EF Core 10 left join писали через `GroupJoin(...).SelectMany(..., DefaultIfEmpty())`. Этот приём хрупкий: при неверной форме `SelectMany` легко получить **случайный CROSS JOIN** (декартово произведение) или материализацию групп с **N+1** подгрузкой. Новые операторы убирают эту ловушку — выражение намерения («левое соединение») совпадает с генерируемым SQL.

```csharp
// ❌ Legacy (до EF Core 10) — многословно и провоцирует cross-join/N+1
var rows = await db.Users
    .GroupJoin(db.Orders, u => u.Id, o => o.UserId, (u, orders) => new { u, orders })
    .SelectMany(x => x.orders.DefaultIfEmpty(), (x, o) => new { x.u.Id, x.u.Name, OrderId = (Guid?)o.Id })
    .ToListAsync();
```

**Adopt-on-upgrade:** после перехода на EF Core 10 заменяй `GroupJoin().SelectMany(..., DefaultIfEmpty())` на `LeftJoin` (новый код пиши сразу на нём). Разбор старого `GroupJoin`-обходного приёма — см. [[collections-linq]].

---

## AsNoTracking и AsNoTrackingWithIdentityResolution

### Что такое tracking

EF держит change tracker — все loaded entities в памяти, отслеживает изменения. На `SaveChanges()` смотрит что изменилось → генерирует UPDATE/DELETE.

**Минусы tracking'а:**
- Memory overhead — копия entity для baseline
- Performance — каждое чтение проверяет identity map
- Не нужно для read-only

```csharp
// ✅ Read-only — отключаем tracking
var users = await db.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .ToListAsync();
```

В .NET 8+ можно глобально:
```csharp
options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
// Tracking явно где нужно через .AsTracking()
```

### AsNoTrackingWithIdentityResolution

Default `AsNoTracking()` — **не дедупит** entities с одинаковым ID. Если у тебя `Include(u => u.Orders)` и 100 orders ссылаются на одного customer, в результате будет 100 копий customer.

```csharp
// ❌ AsNoTracking без identity resolution
var orders = await db.Orders
    .Include(o => o.Customer)
    .AsNoTracking()
    .ToListAsync();
// 100 copies of Customer object

// ✅ С identity resolution
var orders = await db.Orders
    .Include(o => o.Customer)
    .AsNoTrackingWithIdentityResolution()
    .ToListAsync();
// Только уникальные Customers, разделяются ссылками
```

Memory savings драматический при большом количестве shared related entities. Но performance cost — нужен dictionary lookup.

> [!question]- **Интервью: AsNoTracking vs AsNoTrackingWithIdentityResolution?**
> `AsNoTracking()` — не отслеживает изменения, но **не делает identity resolution** — если связанная entity встречается N раз, будет N разных C# объектов с одинаковыми данными.
> `AsNoTrackingWithIdentityResolution()` — additionally дедупит по ID, возвращает same C# instance для same database row. Полезно когда entity обходится в коде и === важен.
> Performance: первая быстрее (один lookup на entity), вторая memory-friendlier для navigation-heavy запросов.

---

## Compiled Queries — для high-throughput

EF транслирует LINQ → SQL **на каждый вызов**. Это медленно (microseconds, но при миллионах операций суммарно много).

```csharp
// ❌ Каждый вызов — translation overhead
public Task<User?> GetByEmailAsync(string email) =>
    db.Users.FirstOrDefaultAsync(u => u.Email == email);

// ✅ Compiled — translation один раз
private static readonly Func<AppDbContext, string, Task<User?>> GetByEmailQuery =
    EF.CompileAsyncQuery((AppDbContext ctx, string email) =>
        ctx.Users.FirstOrDefault(u => u.Email == email));

public Task<User?> GetByEmailAsync(string email) =>
    GetByEmailQuery(db, email);
```

5-10x speedup на hot path. Используй для часто-вызываемых query (login, identity, hot endpoints).

### Limitations
- Не поддерживает `Include` — только projection
- Не поддерживает dynamic queries (предикаты в runtime)
- Compiled query держит **сильную ссылку** на DbContext type — может мешать unloading test assemblies

---

## ExecuteUpdate / ExecuteDelete (.NET 7+)

Bulk operations **без загрузки entities в память**.

```csharp
// ❌ Старый способ — загружаем 10K rows, потом save
var inactive = await db.Users.Where(u => u.LastLoginAt < cutoff).ToListAsync();
foreach (var u in inactive) u.IsActive = false;
await db.SaveChangesAsync();

// ✅ ExecuteUpdate — один SQL UPDATE
await db.Users
    .Where(u => u.LastLoginAt < cutoff)
    .ExecuteUpdateAsync(s => s
        .SetProperty(u => u.IsActive, false)
        .SetProperty(u => u.UpdatedAt, DateTime.UtcNow));
```

```csharp
// Bulk delete
await db.Logs
    .Where(l => l.CreatedAt < DateTime.UtcNow.AddDays(-30))
    .ExecuteDeleteAsync();
```

**Не вызывает change tracker** — не триггерит SaveChanges interceptors, AuditLog handlers, и т.п. Применяй когда уверен что эти эффекты не нужны.

---

## Bulk Insert

EF не оптимизирует много INSERT'ов в один. Для тысяч записей:

### Вариант 1: EFCore.BulkExtensions (open-source)

```bash
dotnet add package EFCore.BulkExtensions
```

```csharp
await db.BulkInsertAsync(users);  // Использует SqlBulkCopy / COPY для PG
await db.BulkUpdateAsync(users);
await db.BulkInsertOrUpdateAsync(users);  // upsert
```

10-50x быстрее `db.AddRange + SaveChanges` для тысяч записей.

### Вариант 2: Npgsql BinaryImport (Postgres)

См. [PostgreSQL Deep]() — `BeginBinaryImportAsync`. Это нативный COPY protocol PG, ещё быстрее `EFCore.BulkExtensions`.

```csharp
await using var conn = await dataSource.OpenConnectionAsync(ct);
await using var writer = await conn.BeginBinaryImportAsync(
    "COPY users (id, name, email) FROM STDIN (FORMAT BINARY)", ct);

foreach (var u in users)
{
    await writer.StartRowAsync(ct);
    await writer.WriteAsync(u.Id, NpgsqlDbType.Uuid, ct);
    await writer.WriteAsync(u.Name, NpgsqlDbType.Text, ct);
    await writer.WriteAsync(u.Email, NpgsqlDbType.Text, ct);
}
await writer.CompleteAsync(ct);
```

### Вариант 3: Z.EntityFramework.Extensions (commercial)

Платная, но самая фичерная — bulk insert/update/merge с поддержкой всех типов БД.

| | EF AddRange | EFCore.BulkExtensions | Npgsql COPY |
|--|-------------|----------------------|-------------|
| 1K rows | ~500ms | ~50ms | ~30ms |
| 10K rows | ~5s | ~200ms | ~80ms |
| 100K rows | ~50s | ~2s | ~500ms |
| Лицензия | Free | MIT | Free (пакет Postgres) |

---

## Raw SQL — когда EF недостаточно

### `FromSqlRaw` / `FromSqlInterpolated`

```csharp
// Параметризованный (защита от SQL injection)
var users = await db.Users
    .FromSqlInterpolated($"SELECT * FROM users WHERE created_at > {cutoff}")
    .Where(u => u.IsActive)  // Можно дальше LINQ-композицию
    .ToListAsync();

// Stored procedure
var stats = await db.Database
    .SqlQueryRaw<UserStat>("CALL get_user_stats({0})", userId)
    .ToListAsync();
```

### `ExecuteSqlRaw` для DDL/non-query

```csharp
await db.Database.ExecuteSqlRawAsync("CREATE INDEX CONCURRENTLY idx_users_email ON users(email)");
```

### Когда raw SQL

| Когда |
|-------|
| Сложные window functions / CTE которые EF не транслирует |
| DB-specific функции (PostgreSQL JSONB операторы, FullText через tsvector) |
| Оптимизация конкретного hot query |
| Migration / DDL |

Raw SQL — fallback. Большинство задач EF решает достойно. Не пиши SQL "из старой привычки" — сначала попробуй LINQ.

---

## Pagination — правильная

### `OFFSET` — медленный для больших offsets

```csharp
// ❌ В page 1000 (offset 100000) — DB читает 100K строк прежде чем отбросить
var page = await db.Orders
    .OrderBy(o => o.CreatedAt)
    .Skip(100_000)
    .Take(50)
    .ToListAsync();
```

PG/SQL Server для `OFFSET 100K` физически читает 100K rows + 50, отбрасывает первые 100K.

### Cursor-based — O(1)

```csharp
// ✅ После предыдущей page имеем lastSeen
public async Task<IReadOnlyList<Order>> GetPageAsync(DateTime? cursor, int pageSize, CancellationToken ct)
{
    var query = db.Orders.OrderByDescending(o => o.CreatedAt);

    if (cursor.HasValue)
        query = query.Where(o => o.CreatedAt < cursor.Value);

    return await query.Take(pageSize).AsNoTracking().ToListAsync(ct);
}
```

API возвращает `{ items: [...], nextCursor: "2026-04-28T10:00:00" }`. Клиент шлёт cursor для следующей page. Используется в Twitter, Stripe, Shopify, везде.

**Limitation:** не позволяет jump to page N — только sequential. Если нужно UI с "page 1, 2, 3, 4, ...", используй OFFSET с `LIMIT`.

### Keyset pagination

Расширенная версия cursor — для composite ordering. Composite-ключ — это семантически кортеж `(CreatedAt, Id)`, и хочется сравнивать его как row-value: `(CreatedAt, Id) <= (@d, @lastId)`. SQL это поддерживает нативно (row-value comparison) и может пройти по составному индексу одним range scan.

> [!warning] EF не транслирует row-value кортежи — OR-форма сканирует широко
> EF Core пока **не умеет** транслировать кортежное сравнение `(a, b) < (c, d)` в SQL row-value предикат. Приходится раскрывать его в OR-форму (ниже). Проблема в том, что планировщик БД по OR-предикату часто **не использует** composite-индекс эффективно: вместо одного range scan по `(CreatedAt, Id)` он делает более широкий скан и фильтрует. На больших таблицах это сводит на нет весь смысл keyset pagination. Если профиль показывает, что индекс не задействован — переходи на raw-SQL row-value вариант ниже.

OR-форма (то, что EF способен сгенерировать сегодня) — корректна, но субоптимальна по плану:

```csharp
public async Task<List<Order>> GetPageAsync(DateTime? lastCreatedAt, Guid? lastId, int pageSize, CancellationToken ct)
{
    var query = db.Orders.OrderByDescending(o => o.CreatedAt).ThenBy(o => o.Id);

    if (lastCreatedAt.HasValue && lastId.HasValue)
    {
        // OR-decomposition: EF не умеет (CreatedAt, Id) <= (@d, @lastId).
        // Планировщик может НЕ задействовать composite-индекс по этому предикату.
        query = query.Where(o =>
            o.CreatedAt < lastCreatedAt ||
            (o.CreatedAt == lastCreatedAt && o.Id.CompareTo(lastId.Value) > 0));
    }

    return await query.Take(pageSize).ToListAsync(ct);
}
```

### Keyset pagination — раздельный raw-SQL вариант (рекомендуемый на горячем пути)

Чтобы получить настоящий row-value предикат и гарантированный range scan по индексу — обойди трансляцию через `FromSqlInterpolated`. Параметры по-прежнему параметризуются (защита от SQL injection), а дальше можно дописывать LINQ-композицию:

```csharp
public async Task<List<Order>> GetPageKeysetAsync(DateTime lastCreatedAt, Guid lastId, int pageSize, CancellationToken ct)
{
    // Row-value comparison: один range scan по индексу (created_at, id).
    // Колонки/типы должны совпадать с маппингом сущности (см. сгенерированный SQL).
    return await db.Orders
        .FromSqlInterpolated($"""
            SELECT * FROM orders
            WHERE (created_at, id) <= ({lastCreatedAt}, {lastId})
            ORDER BY created_at DESC, id DESC
            LIMIT {pageSize}
            """)
        .AsNoTracking()
        .ToListAsync(ct);
}
```

Под это нужен composite-индекс `(created_at, id)` в том же порядке, что и `ORDER BY`. Направление сравнения (`<=` vs `>=`) выбирается под направление сортировки страницы. Детальнее про то, как row-value предикат ложится на составной индекс и почему OR-форма теряет index seek — см. [[optimization]].

> [!tip] Когда оставаться на LINQ-форме
> Для небольших/средних таблиц OR-форма читается лучше и план приемлем — не усложняй. Raw-SQL row-value оправдан, когда таблица большая, keyset на горячем пути, и `EXPLAIN` подтвердил, что OR-предикат не берёт composite-индекс.

---

## Async streaming через AsAsyncEnumerable

Для больших датасетов которые не помещаются в память:

```csharp
public async IAsyncEnumerable<UserDto> StreamActiveUsersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var user in db.Users
        .Where(u => u.IsActive)
        .Select(u => new UserDto(u.Id, u.Name))
        .AsAsyncEnumerable()
        .WithCancellation(ct))
    {
        yield return user;
    }
}

// Использование — не загружаем всё в память сразу
await foreach (var user in service.StreamActiveUsersAsync(ct))
{
    await ProcessAsync(user);
}
```

Connection держится открытым на время enumeration — для коротких операций OK, для долгой обработки в loop'е — лучше batch'и.

---

## Concurrency token

См. [Concurrency](concurrency.md). Кратко — добавь rowversion / xmin для optimistic concurrency:

```csharp
public class Account
{
    public Guid Id { get; set; }
    public decimal Balance { get; set; }

    [Timestamp]  // SQL Server rowversion
    public byte[] RowVersion { get; set; } = null!;
}

// Postgres — xmin
modelBuilder.Entity<Account>()
    .Property("xmin")
    .HasColumnType("xid")
    .ValueGeneratedOnAddOrUpdate()
    .IsConcurrencyToken();
```

---

## Query plan analysis

### EF logging

```csharp
options.LogTo(Console.WriteLine, [DbLoggerCategory.Database.Command.Name], LogLevel.Information);
options.EnableSensitiveDataLogging();  // Параметры в plain text — Dev only!
```

Видишь генерируемый SQL → копируешь в pgAdmin/SSMS → `EXPLAIN ANALYZE`.

### EXPLAIN ANALYZE

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT u.email, COUNT(o.id)
FROM users u LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2026-01-01'
GROUP BY u.email;
```

См. [PostgreSQL Deep]() — детальный разбор плана.

### Query Tags для tracing

```csharp
var users = await db.Users
    .TagWith("API: GetActiveUsers - high-traffic")
    .Where(u => u.IsActive)
    .ToListAsync();
// SQL содержит /* API: GetActiveUsers - high-traffic */ — видно в pg_stat_statements / slow log
```

Помогает идентифицировать какой код породил конкретный slow query.

---

## DbContext lifecycle и pooling

### Per-request scope (default ASP.NET)

`AddDbContext` регистрирует scoped — один DbContext на HTTP request. На каждом request создаётся новый.

### `AddDbContextPool` — для high-throughput

```csharp
builder.Services.AddDbContextPool<AppDbContext>((sp, options) =>
{
    options.UseNpgsql(connStr);
}, poolSize: 128);
```

Pool готовых DbContext — на request получаем reset'нутый instance из pool, освобождаем в pool после. Экономит создание/уничтожение DbContext.

**Pitfall:** если ты сохраняешь state в DbContext (instance fields, scoped services через ctor) — это state сохранится между requests! Используй pool только если context чистый.

### `AddDbContextFactory` — для Blazor Server

См. [Blazor Server](). В Blazor Server scoped DbContext = один DbContext на circuit (длинный сессионный объект). Параллельные async-методы из компонентов конфликтуют → "DbContext is not thread-safe".

```csharp
// ❌ В Blazor Server — НЕ scoped DbContext
builder.Services.AddDbContext<AppDbContext>(...);

// ✅ Factory — short-lived per operation
builder.Services.AddDbContextFactory<AppDbContext>(...);

@inject IDbContextFactory<AppDbContext> Factory
private async Task LoadAsync()
{
    using var db = await Factory.CreateDbContextAsync();
    _items = await db.Tasks.ToListAsync();
}
```

---

## Batching — `SaveChanges` оптимизация

EF Core по умолчанию **батчит** INSERT/UPDATE/DELETE в один SQL roundtrip:

```csharp
db.Users.AddRange(user1, user2, user3);
await db.SaveChangesAsync();
// SQL: INSERT INTO users (...) VALUES (...), (...), (...)
```

Параметр `MaxBatchSize` — лимит размера batch (default 100 для SQL Server, 1000 для Postgres):
```csharp
options.UseNpgsql(connStr, b => b.MaxBatchSize(1000));
```

Для очень больших объёмов — bulk insert (см. выше).

---

## Performance benchmarks — важные замеры

Benchmark.NET для EF Core query latency:

```csharp
[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net100)]
public class QueryBenchmarks
{
    private AppDbContext _db = null!;

    [GlobalSetup]
    public void Setup()
    {
        _db = CreateDbContext();
    }

    [Benchmark(Baseline = true)]
    public async Task<List<User>> GetActive_FullEntity() =>
        await _db.Users.Where(u => u.IsActive).ToListAsync();

    [Benchmark]
    public async Task<List<UserDto>> GetActive_Projection() =>
        await _db.Users.Where(u => u.IsActive)
            .Select(u => new UserDto(u.Id, u.Name))
            .ToListAsync();

    [Benchmark]
    public async Task<List<UserDto>> GetActive_NoTracking_Projection() =>
        await _db.Users.AsNoTracking().Where(u => u.IsActive)
            .Select(u => new UserDto(u.Id, u.Name))
            .ToListAsync();

    [Benchmark]
    public async Task<List<UserDto>> GetActive_Compiled() =>
        await GetActiveUsersCompiled(_db).ToListAsync();

    private static readonly Func<AppDbContext, IAsyncEnumerable<UserDto>> GetActiveUsersCompiled =
        EF.CompileAsyncQuery((AppDbContext ctx) =>
            ctx.Users.AsNoTracking()
                .Where(u => u.IsActive)
                .Select(u => new UserDto(u.Id, u.Name)));
}
```

Типичные результаты на 1K rows:
- Full entity, tracked: 5ms, 50KB allocated
- Projection, tracked: 3ms, 25KB
- Projection, no-tracking: 2ms, 15KB
- Compiled: 1.5ms, 12KB

Cumulative — 3x speedup в hot path.

---

## Production checklist

- [ ] AsNoTracking для всех read-only запросов
- [ ] Projection (`.Select(...)`) для API responses (не возвращай Entity)
- [ ] Compiled queries для hot path (login, identity, частые endpoints)
- [ ] Включён EnableRetryOnFailure для transient errors
- [ ] Split queries для multiple collection includes
- [ ] Cursor-based pagination для больших списков
- [ ] ExecuteUpdate / ExecuteDelete для bulk operations
- [ ] Bulk insert через EFCore.BulkExtensions / Npgsql COPY для больших импортов
- [ ] DbContext pooling для high-throughput сервисов
- [ ] DbContextFactory вместо scoped в Blazor Server
- [ ] Query Tags на критичных endpoints для trace
- [ ] Concurrency tokens на important entities (xmin / rowversion)
- [ ] EF logging only at Warning+ в проде (sensitive logging — никогда!)
- [ ] Monitoring slow queries в БД (pg_stat_statements / slow query log)
- [ ] Periodic ANALYZE после больших bulk inserts
- [ ] BenchmarkDotNet baseline для critical queries
- [ ] N+1 detection включён через EF logging warnings

---

## Common pitfalls

### 1. `.ToListAsync()` слишком рано

```csharp
// ❌ Тащим всю таблицу в память, потом фильтр в C#
var active = (await db.Users.ToListAsync()).Where(u => u.IsActive).ToList();

// ✅ Фильтр в SQL
var active = await db.Users.Where(u => u.IsActive).ToListAsync();
```
Главный antipattern. Особенно опасен на больших таблицах — staging работал на 100 rows, prod упал на 10M.

### 2. `Where` после `Include`

```csharp
// ❌ Include грузит ВСЕ orders, потом C# фильтрует
var users = await db.Users.Include(u => u.Orders).Where(u => u.Orders.Any(o => o.IsPaid)).ToListAsync();
// SQL: LEFT JOIN orders, потом WHERE EXISTS — orders все приходят
```
В большинстве случаев EF переписывает корректно, но проверяй SQL.

### 3. Lazy loading включён в production

```csharp
options.UseLazyLoadingProxies();  // ❌ В production — отключи
```
Магия `user.Orders` без Include — N+1 happens silently.

### 4. `Count()` перед фильтрацией

```csharp
// ❌ COUNT(*) FROM users без условия
var count = (await db.Users.ToListAsync()).Count(u => u.IsActive);

// ✅ COUNT с фильтром в SQL
var count = await db.Users.CountAsync(u => u.IsActive);
```

### 5. `Any()` vs `Count() > 0`

```csharp
// ❌ Считает все, возвращает int
if ((await db.Users.CountAsync(u => u.Email == email)) > 0) ...

// ✅ EXISTS — DB возвращает true/false на первой найденной row
if (await db.Users.AnyAsync(u => u.Email == email)) ...
```

### 6. EnableSensitiveDataLogging в production
Параметры запросов попадают в логи → утечка PII / passwords. **Только Dev**.

### 7. Long-running DbContext
DbContext, живущий часы (worker, background service), накапливает change tracker → memory leak.
**Решение:** scoped per operation, пересоздавать каждые N операций.

### 8. Async-метод не await'ится

```csharp
// ❌ Не await'ится — запрос может не выполниться, exception потеряется
db.Users.AddAsync(user);
db.SaveChangesAsync();

// ✅
await db.Users.AddAsync(user);
await db.SaveChangesAsync();
```

### 9. Множество `Contains` с большим списком

```csharp
// ❌ IN (...) с 10K элементов — может упереться в parameter limit
var ids = Enumerable.Range(1, 10_000).ToList();
await db.Users.Where(u => ids.Contains(u.Id)).ToListAsync();
```
PG лимит — 32K параметров. SQL Server — 2100. Решения: разбей на chunks, temp table, table-valued parameter.

### 10. `OrderBy` без index'а на large table
EF посылает `ORDER BY created_at DESC` — DB делает sort, возможно на диске.
**Решение:** index на column, по которому сортируешь.

---

## См. также

- [EF Core Basics и Tracking](basics-tracking.md) — основы DbContext, change tracker
- [EF Core Concurrency](concurrency.md) — optimistic locking, retry
- [EF Core Patterns]() — repository, soft delete, audit
- [PostgreSQL Deep]() — EXPLAIN ANALYZE, pg_stat_statements, индексы
- [SQL Optimization]() — общие принципы SQL performance
- [Blazor Server]() — DbContextFactory pattern
- [Performance]() — BenchmarkDotNet для query benchmarks

## Reading list

- **EF Core docs — Performance** — learn.microsoft.com/ef/core/performance/
- **Andriy Svyryd — EF Core internals talks** — отличное объяснение query pipeline
- **Shay Rojansky** — Twitter/X (Npgsql + EF Core lead, постит про performance regularly)
- **Use The Index, Luke!** — use-the-index-luke.com (важно понимать что под EF)
- **Milan Jovanović — EF Core series** — milanjovanovic.tech (compiled queries, bulk operations)
- **Jon Smith — EF Core in Action (3rd ed.)** — comprehensive book
