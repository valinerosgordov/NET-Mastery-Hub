---
tags: [efcore, concurrency, transactions, optimistic-locking, pessimistic-locking, deadlocks, isolation, advisory-locks]
level: Senior
date: 2026-04-30
---

# Concurrency, Transactions и Locks

> Полный гайд по concurrent доступу к данным через EF Core. Закрывает: optimistic / pessimistic locking, isolation levels, deadlock detection и retry, advisory locks (Postgres), distributed locks (Redis), connection pooling, batching, savepoints, distributed transactions, ловушки async transactions.

---

## Что это, зачем и когда

### Что такое Concurrency?
Ситуация, когда **два пользователя** одновременно меняют **одни и те же данные**. Без защиты — один перезапишет изменения другого, и данные потеряются (lost update).

**Аналогия:** Два человека редактируют один Google Doc одновременно. Если нет синхронизации — один потеряет свои изменения.

### Когда нужна защита?
- Несколько пользователей могут редактировать один ресурс (заказ, профиль)
- Фоновые процессы и HTTP-запросы могут менять одну запись одновременно
- Финансовые операции (баланс счёта — нельзя потерять списание)
- Inventory (резервирование товаров)

### Какая защита бывает?

| Тип | Как работает | Когда |
|-----|-------------|-------|
| **Optimistic** | Проверяет «не изменились ли данные» при сохранении. Если изменились — ошибка | Конфликты РЕДКИ. Веб-приложения (99% случаев) |
| **Pessimistic** | Блокирует строку в БД, пока один пользователь работает | Конфликты ЧАСТЫ. Финансы, склады |
| **Advisory locks** | Application-level lock в БД, не привязан к таблице | Distributed coordination |
| **Distributed locks (Redis)** | Lock через внешнее хранилище | Multi-service coordination |
| **Transactions** | Группа операций — либо ВСЕ выполняются, либо НИЧЕГО | Несколько связанных изменений (перевод денег) |

---

> [!question]- **Интервью: Optimistic concurrency — как реализовать?**
> `[ConcurrencyCheck]` или `[Timestamp]` на свойстве → EF добавляет `WHERE RowVersion = @old` в UPDATE. При конфликте 0 rows affected → `DbUpdateConcurrencyException`. Обработка: перечитать (DB wins), перезаписать (client wins) или merge.

## Optimistic Concurrency

Предполагаем, что конфликты редки. При `SaveChanges` EF проверяет, не изменилась ли строка с момента чтения.

### Реализация

#### Вариант 1: RowVersion (timestamp)

```csharp
// SQL Server — rowversion / timestamp (auto-incremented binary)
public class Order
{
    public Guid Id { get; set; }
    public decimal Total { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; } = null!;
}

// PostgreSQL — система xmin
modelBuilder.Entity<Order>()
    .UseXminAsConcurrencyToken();
// Использует system column xmin (transaction id) — нет дополнительной колонки
```

#### Вариант 2: ConcurrencyCheck на конкретном поле

```csharp
public class Order
{
    public Guid Id { get; set; }

    [ConcurrencyCheck]  // только эта колонка проверяется
    public decimal Total { get; set; }

    public string? Notes { get; set; }  // не проверяется
}
```

Полезно когда `Total` критичен, но `Notes` можно перезаписывать.

#### Вариант 3: Manual version counter

```csharp
public class Order
{
    public Guid Id { get; set; }
    public decimal Total { get; set; }
    public int Version { get; set; }
}

modelBuilder.Entity<Order>()
    .Property(o => o.Version)
    .IsConcurrencyToken();

// При каждом save — увеличиваем version вручную или через interceptor
public override int SaveChanges()
{
    foreach (var entry in ChangeTracker.Entries<IVersioned>().Where(e => e.State == EntityState.Modified))
    {
        entry.Entity.Version++;
    }
    return base.SaveChanges();
}
```

### Что генерирует EF

```sql
-- При UPDATE
UPDATE "Orders"
SET "Total" = @newTotal,
    "RowVersion" = DEFAULT  -- или version + 1
WHERE "Id" = @id 
  AND "RowVersion" = @oldVersion;  -- ← проверка

-- Если 0 rows affected → DbUpdateConcurrencyException
```

### Обработка конфликта — три стратегии

```csharp
public async Task<Result> UpdateOrderAsync(Guid id, decimal newTotal, CancellationToken ct)
{
    const int maxRetries = 3;
    
    for (int attempt = 0; attempt < maxRetries; attempt++)
    {
        try
        {
            var order = await _context.Orders.FindAsync([id], ct);
            if (order is null) return Result.NotFound();
            
            order.Total = newTotal;
            await _context.SaveChangesAsync(ct);
            return Result.Success();
        }
        catch (DbUpdateConcurrencyException ex)
        {
            var entry = ex.Entries.Single();
            
            // Стратегия 1: Database wins — перезагрузить из БД
            await entry.ReloadAsync(ct);
            // → пользователь видит свежие данные, делает изменения заново
            return Result.Conflict("Data was modified by another user, please retry");
            
            // ИЛИ Стратегия 2: Client wins — перезаписать
            entry.OriginalValues.SetValues(await entry.GetDatabaseValuesAsync(ct));
            await _context.SaveChangesAsync(ct);
            return Result.Success();
            
            // ИЛИ Стратегия 3: Merge — для каждого поля решить кто wins
            var dbValues = await entry.GetDatabaseValuesAsync(ct);
            var clientValues = entry.CurrentValues;
            var resolvedValues = MergeValues(dbValues, clientValues, entry.OriginalValues);
            entry.CurrentValues.SetValues(resolvedValues);
            entry.OriginalValues.SetValues(dbValues);  // обновляем concurrency token
            await _context.SaveChangesAsync(ct);
            return Result.Success();
        }
    }
    
    return Result.Failed("Too many conflicts");
}
```

### Custom Merge Logic

```csharp
private static PropertyValues MergeValues(
    PropertyValues db,
    PropertyValues client,
    PropertyValues original)
{
    var merged = db.Clone();
    
    foreach (var prop in db.Properties)
    {
        // Если client изменил это поле (отличается от original) — берём client
        if (!Equals(client[prop.Name], original[prop.Name]))
        {
            merged[prop.Name] = client[prop.Name];
        }
        // Иначе берём db (он мог измениться другим пользователем)
    }
    
    return merged;
}
```

> [!info] Когда какая стратегия
> - **DB wins** — для UI, где пользователь хочет видеть актуальное и решать заново. Финансы, инвентаризация.
> - **Client wins** — для idempotent updates, где порядок не важен. Settings, profile.
> - **Merge** — collaborative editing (Google Docs-style). Сложно реализовать правильно, требует field-level diff.

### Optimistic concurrency для bulk operations

```csharp
// ❌ EF Core 7+ ExecuteUpdate не проверяет concurrency token!
await context.Orders
    .Where(o => o.CustomerId == customerId)
    .ExecuteUpdateAsync(setters => setters.SetProperty(o => o.Status, "Cancelled"));
// Никаких WHERE RowVersion = ...

// ✅ Если нужна optimistic — итерировать вручную
var orders = await context.Orders.Where(o => o.CustomerId == customerId).ToListAsync(ct);
foreach (var order in orders) order.Status = "Cancelled";
await context.SaveChangesAsync(ct);  // здесь проверка работает
```

---

## Pessimistic Concurrency — SELECT FOR UPDATE

Для критичных операций — заблокировать строку прямо в БД, чтобы никто не читал/писал её одновременно.

### EF Core 8+ — `SELECT ... FOR UPDATE`

EF Core нативно не поддерживает FOR UPDATE — только через raw SQL:

```csharp
public async Task<Result> TransferAsync(
    Guid fromAccountId, 
    Guid toAccountId, 
    decimal amount, 
    CancellationToken ct)
{
    await using var transaction = await _context.Database.BeginTransactionAsync(ct);
    
    try
    {
        // 1. Lock both rows (порядок важен — фиксированный для избежания deadlock!)
        var ids = new[] { fromAccountId, toAccountId }.OrderBy(id => id).ToArray();
        
        var lockedAccounts = await _context.Accounts
            .FromSqlRaw(@"
                SELECT * FROM ""Accounts"" 
                WHERE ""Id"" = ANY({0}) 
                ORDER BY ""Id""
                FOR UPDATE", ids)
            .ToListAsync(ct);
        
        var from = lockedAccounts.First(a => a.Id == fromAccountId);
        var to = lockedAccounts.First(a => a.Id == toAccountId);
        
        if (from.Balance < amount)
            return Result.Failed("Insufficient funds");
        
        from.Balance -= amount;
        to.Balance += amount;
        
        await _context.SaveChangesAsync(ct);
        await transaction.CommitAsync(ct);
        
        return Result.Success();
    }
    catch
    {
        await transaction.RollbackAsync(ct);
        throw;
    }
}
```

### Lock modes (Postgres)

| Mode | SQL | Что блокирует |
|------|-----|--------------|
| `FOR UPDATE` | `SELECT ... FOR UPDATE` | Полная блокировка — другие SELECT FOR UPDATE и UPDATE/DELETE ждут |
| `FOR NO KEY UPDATE` | `... FOR NO KEY UPDATE` | Слабее — позволяет FK references |
| `FOR SHARE` | `... FOR SHARE` | Read lock — другие могут читать, но не писать |
| `FOR KEY SHARE` | `... FOR KEY SHARE` | Слабый share — для FK validation |
| `NOWAIT` | `... FOR UPDATE NOWAIT` | Если строка заблокирована — сразу exception |
| `SKIP LOCKED` | `... FOR UPDATE SKIP LOCKED` | Пропустить заблокированные — для job queue |

### SELECT FOR UPDATE SKIP LOCKED — паттерн job queue

Идеально для распределённой queue, где каждый worker берёт следующую доступную задачу:

```csharp
public async Task<Job?> GetNextJobAsync(CancellationToken ct)
{
    await using var transaction = await _context.Database.BeginTransactionAsync(ct);
    
    var job = await _context.Jobs
        .FromSqlRaw(@"
            SELECT * FROM ""Jobs""
            WHERE ""Status"" = 'Pending'
            ORDER BY ""CreatedAt""
            LIMIT 1
            FOR UPDATE SKIP LOCKED")
        .FirstOrDefaultAsync(ct);
    
    if (job is null)
    {
        await transaction.CommitAsync(ct);
        return null;
    }
    
    job.Status = "Processing";
    job.WorkerId = Environment.MachineName;
    await _context.SaveChangesAsync(ct);
    await transaction.CommitAsync(ct);
    
    return job;
}
```

10 worker'ов одновременно — каждый получит **разную** задачу, без duplicate processing.

> [!info] Когда Pessimistic
> - Финансовые транзакции (баланс счёта)
> - Inventory (резервирование товара)
> - Job queue (один worker — одна задача)
> - Booking (билеты, переговорки)

> [!warning] Pessimistic блокирует — short transactions only
> Lock держится **до commit/rollback**. Если транзакция зависла на 30 секунд — все остальные ждут. Делай транзакции короткими, не делай в них HTTP calls / file I/O.

---

## Transactions

### Implicit (по умолчанию)

```csharp
context.Orders.Add(order);
context.OrderItems.AddRange(items);
await context.SaveChangesAsync(ct);  // одна транзакция автоматически: всё или ничего
```

### Explicit — несколько SaveChanges в одной транзакции

```csharp
await using var transaction = await context.Database.BeginTransactionAsync(ct);
try
{
    context.Orders.Add(order);
    await context.SaveChangesAsync(ct);  // INSERT, генерирует ID
    
    context.Payments.Add(new Payment { OrderId = order.Id, Amount = order.Total });
    await context.SaveChangesAsync(ct);  // INSERT с FK на order.Id
    
    await transaction.CommitAsync(ct);
}
catch
{
    await transaction.RollbackAsync(ct);
    throw;
}
```

### Savepoints — частичный rollback

```csharp
await using var transaction = await context.Database.BeginTransactionAsync(ct);

context.Orders.Add(order);
await context.SaveChangesAsync(ct);

await transaction.CreateSavepointAsync("BeforePayment", ct);

try
{
    context.Payments.Add(payment);
    await context.SaveChangesAsync(ct);
}
catch (PaymentException)
{
    // Откатить только payment, оставить order
    await transaction.RollbackToSavepointAsync("BeforePayment", ct);
    
    order.Status = OrderStatus.AwaitingPayment;
    await context.SaveChangesAsync(ct);
}

await transaction.CommitAsync(ct);
```

### Кросс-контекстные транзакции

```csharp
// Вариант 1: Общее соединение
var connection = new NpgsqlConnection(connString);
await connection.OpenAsync(ct);
await using var transaction = await connection.BeginTransactionAsync(ct);

var options1 = new DbContextOptionsBuilder<Context1>()
    .UseNpgsql(connection).Options;
var options2 = new DbContextOptionsBuilder<Context2>()
    .UseNpgsql(connection).Options;

await using var ctx1 = new Context1(options1);
await using var ctx2 = new Context2(options2);

await ctx1.Database.UseTransactionAsync(transaction, ct);
await ctx2.Database.UseTransactionAsync(transaction, ct);

// ... операции в обоих контекстах
await transaction.CommitAsync(ct);
```

```csharp
// Вариант 2: TransactionScope (legacy, distributed)
using var scope = new TransactionScope(
    TransactionScopeOption.Required,
    new TransactionOptions { IsolationLevel = IsolationLevel.ReadCommitted },
    TransactionScopeAsyncFlowOption.Enabled);  // ← обязательно для async!

await ctx1.SaveChangesAsync();
await ctx2.SaveChangesAsync();

scope.Complete();
```

> [!warning] TransactionScope с async — обязателен флаг
> Без `TransactionScopeAsyncFlowOption.Enabled` контекст транзакции теряется при переключении потоков → silent transaction abort.

### Distributed Transactions (XA / 2PC)

С .NET Core убрана поддержка `MSDTC`. Distributed transactions через `TransactionScope` работают только в .NET Framework / Windows + специальные ситуации .NET 7+.

**Правильное решение в современных системах** — **Outbox pattern** для cross-service consistency. См. [Distributed Systems](../Architecture/distributed-systems.md).

---

## Isolation Levels

```csharp
await using var transaction = await context.Database
    .BeginTransactionAsync(IsolationLevel.RepeatableRead, ct);
```

### Феномены и уровни

| Уровень | Dirty Read | Non-repeatable | Phantom | Cost |
|---------|-----------|----------------|---------|------|
| **Read Uncommitted** | Да | Да | Да | Минимальный |
| **Read Committed** (default PG/SQL Server) | Нет | Да | Да | Низкий |
| **Repeatable Read** | Нет | Нет | Да | Средний |
| **Serializable** | Нет | Нет | Нет | Высокий (часто abort) |
| **Snapshot** (SQL Server, PG MVCC) | Нет | Нет | Нет | Низкий (через MVCC) |

### Объяснения феноменов

**Dirty Read** — TX1 читает данные, которые TX2 изменил, но ещё не commit'нул. Если TX2 откатится — TX1 видел "phantom" данные.

```sql
-- TX1                       -- TX2
                              UPDATE Account SET Balance = 1000 WHERE Id = 1;
SELECT Balance FROM Account WHERE Id = 1;  -- видит 1000 (но это unpushed)
                              ROLLBACK;
                              -- баланс на самом деле остался прежним!
```

**Non-repeatable Read** — TX1 читает строку, потом снова читает её — значение изменилось (TX2 commit'нул UPDATE между).

```sql
-- TX1                                     -- TX2
SELECT Balance FROM Account WHERE Id = 1;  -- видит 100
                                            UPDATE Account SET Balance = 200 WHERE Id = 1;
                                            COMMIT;
SELECT Balance FROM Account WHERE Id = 1;  -- видит 200 (изменилось!)
```

**Phantom Read** — TX1 делает запрос с WHERE, потом снова — появились новые строки (TX2 INSERT'нул).

```sql
-- TX1                                     -- TX2
SELECT COUNT(*) FROM Orders WHERE Status='Pending';  -- 5
                                            INSERT INTO Orders (..., Status) VALUES (..., 'Pending');
                                            COMMIT;
SELECT COUNT(*) FROM Orders WHERE Status='Pending';  -- 6 (phantom!)
```

### Read Committed (default)

99% случаев — этого достаточно. CRUD приложения, обычные API.

### Snapshot (PG MVCC) и SQL Server Snapshot

PG использует MVCC по умолчанию — читатели видят снимок данных на момент начала транзакции, не блокируют писателей.

```sql
-- PG: каждый SELECT в одной TX видит одну и ту же версию (snapshot)
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM Users;  -- snapshot
-- ... другая TX делает UPDATE ...
SELECT * FROM Users;  -- тот же snapshot, не видит UPDATE
COMMIT;
```

В SQL Server — нужно explicit:
```sql
ALTER DATABASE MyDb SET ALLOW_SNAPSHOT_ISOLATION ON;
ALTER DATABASE MyDb SET READ_COMMITTED_SNAPSHOT ON;  -- default Read Committed = Snapshot
```

### Serializable — самый строгий

Гарантирует что transactions выполнялись как если бы в одну линию (serial schedule).

**Цена:** PG может **abort** транзакции если detected serialization anomaly:

```csharp
try
{
    await using var tx = await context.Database
        .BeginTransactionAsync(IsolationLevel.Serializable, ct);
    
    // ... операции ...
    await context.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
}
catch (PostgresException ex) when (ex.SqlState == "40001")
{
    // serialization_failure — нужен retry
}
```

> [!info] Когда Serializable
> Финансовые расчёты, inventory с complex business rules. Иначе Read Committed достаточно.

---

## Deadlocks

### Что такое deadlock

Два процесса ждут ресурсы друг друга. БД detect'ит и выбирает **жертву** (одну из транзакций) — abort'ит её.

```
TX1:                                TX2:
LOCK Account A                      LOCK Account B
(пытается LOCK Account B)           (пытается LOCK Account A)
                wait                                  wait
                wait                                  wait
                            DEADLOCK
                       PG kills one of them
```

### Защита

```csharp
// 1. Фиксированный порядок локов
public async Task TransferAsync(Guid fromId, Guid toId, decimal amount)
{
    // ВСЕГДА lock в одинаковом порядке (по Id например)
    var ids = new[] { fromId, toId }.OrderBy(id => id).ToArray();
    
    var accounts = await _context.Accounts
        .FromSqlRaw("SELECT * FROM Accounts WHERE Id = ANY({0}) ORDER BY Id FOR UPDATE", ids)
        .ToListAsync();
    
    // ...
}

// 2. Короткие транзакции
// Не делай await HttpClient внутри transaction!

// 3. SET LOCK_TIMEOUT (PG) / SET LOCK_TIMEOUT (SQL Server)
await _context.Database.ExecuteSqlRawAsync("SET LOCAL lock_timeout = '5s'");
// если lock не получен за 5 сек → exception
```

### Retry policy для deadlocks

```csharp
public async Task<T> ExecuteWithRetryAsync<T>(
    Func<Task<T>> operation,
    int maxRetries = 3,
    CancellationToken ct = default)
{
    for (int attempt = 0; ; attempt++)
    {
        try
        {
            return await operation();
        }
        catch (DbUpdateException ex) when (IsDeadlockException(ex) && attempt < maxRetries)
        {
            // Exponential backoff с jitter
            var delay = TimeSpan.FromMilliseconds(Math.Pow(2, attempt) * 100 + Random.Shared.Next(50));
            await Task.Delay(delay, ct);
        }
    }
}

private static bool IsDeadlockException(Exception ex)
{
    return ex.InnerException switch
    {
        PostgresException pg => pg.SqlState == "40P01",  // deadlock_detected
        SqlException sql => sql.Number == 1205,           // SQL Server deadlock
        _ => false
    };
}
```

### EF Core встроенный retry — `EnableRetryOnFailure`

```csharp
options.UseNpgsql(connStr, b => b.EnableRetryOnFailure(
    maxRetryCount: 5,
    maxRetryDelay: TimeSpan.FromSeconds(30),
    errorCodesToAdd: null));

// Retries on transient errors:
// - Network failures
// - Timeouts
// - Deadlocks (40P01 в PG)
// - Connection refused
```

> [!warning] EnableRetryOnFailure + transactions = ловушка
> Если используешь `EnableRetryOnFailure` + явные транзакции — нужен retry strategy execution:
>
> ```csharp
> var strategy = context.Database.CreateExecutionStrategy();
> await strategy.ExecuteAsync(async () =>
> {
>     await using var tx = await context.Database.BeginTransactionAsync();
>     // ... операции ...
>     await tx.CommitAsync();
> });
> ```

---

## Advisory Locks (Postgres)

Postgres advisory locks — application-level lock, не привязан к таблице. Полезен для distributed coordination.

### Session-level vs Transaction-level

```csharp
// Transaction-level — освобождается при commit/rollback
await context.Database.ExecuteSqlRawAsync("SELECT pg_advisory_xact_lock(@key)", 
    new NpgsqlParameter("@key", lockKey));

// Session-level — нужно явно освобождать
await context.Database.ExecuteSqlRawAsync("SELECT pg_advisory_lock(@key)", ...);
// ... работа ...
await context.Database.ExecuteSqlRawAsync("SELECT pg_advisory_unlock(@key)", ...);

// Try-lock (не блокирующий)
var locked = await context.Database
    .SqlQueryRaw<bool>("SELECT pg_try_advisory_lock(@key)", ...)
    .FirstAsync();

if (!locked) throw new InvalidOperationException("Already locked");
```

### Применение: distributed singleton

```csharp
public class CronJobService(AppDbContext context, ILogger<CronJobService> log)
{
    public async Task RunDailyJobAsync(CancellationToken ct)
    {
        const long lockKey = 12345;  // unique key per job
        
        // Только один instance во всём кластере выполнит
        var locked = await context.Database
            .SqlQueryRaw<bool>(
                "SELECT pg_try_advisory_lock({0})", 
                lockKey)
            .FirstAsync(ct);
        
        if (!locked)
        {
            log.LogInformation("Another instance is running this job");
            return;
        }
        
        try
        {
            await DoWorkAsync(ct);
        }
        finally
        {
            await context.Database.ExecuteSqlRawAsync(
                "SELECT pg_advisory_unlock({0})", 
                new[] { (object)lockKey }, 
                ct);
        }
    }
}
```

### Применение: prevent concurrent migration

См. [EF Core Migrations](migrations.md#применение-миграций-при-старте-app--не-рекомендуется).

---

## Distributed Locks через Redis (RedLock)

Для координации между **разными сервисами** (не только instances одного) — используй Redis.

```csharp
// NuGet: RedLock.net
using RedLockNet.SERedis;
using RedLockNet.SERedis.Configuration;

var endpoints = new List<RedLockEndPoint>
{
    new() { EndPoint = new DnsEndPoint("redis-1", 6379) },
    new() { EndPoint = new DnsEndPoint("redis-2", 6379) },
    new() { EndPoint = new DnsEndPoint("redis-3", 6379) }
};

var multiplexers = endpoints.Select(e => 
    ConnectionMultiplexer.Connect(e.EndPoint.ToString())).ToList();
var redLockFactory = RedLockFactory.Create(multiplexers);

// Использование
var resource = $"order:{orderId}";
var expiry = TimeSpan.FromSeconds(30);

await using var redLock = await redLockFactory.CreateLockAsync(
    resource, 
    expiry, 
    wait: TimeSpan.FromSeconds(5),
    retry: TimeSpan.FromMilliseconds(200));

if (redLock.IsAcquired)
{
    // Critical section — только один service одновременно
    await ProcessOrderAsync(orderId);
}
else
{
    throw new InvalidOperationException("Could not acquire lock");
}
// auto-release при Dispose
```

См. [Distributed Systems — Distributed Locking](../Architecture/distributed-systems.md).

---

## Connection Pooling

ADO.NET автоматически пулит соединения.

```csharp
// Соединение возвращается в пул при Dispose DbContext
// DbContext Scoped — один на HTTP-запрос, соединение из пула

// Настройка пула в connection string
"Host=...;Database=...;Maximum Pool Size=100;Minimum Pool Size=5;Connection Idle Lifetime=300;"
```

### DbContext pooling (EF Core)

```csharp
builder.Services.AddDbContextPool<AppDbContext>(
    options => options.UseNpgsql(connectionString),
    poolSize: 128);
```

**Что даёт:** переиспользование самих DbContext'ов (с очисткой Change Tracker). Для stateless read-only API — значительный прирост (5-15% throughput).

> [!warning] DbContextPool ограничения
> - **Нельзя** инжектить Scoped-сервисы через конструктор DbContext (context пулится как Singleton-подобный)
> - Конструктор должен принимать только `DbContextOptions<T>` (без других параметров)
> - Нельзя override `OnConfiguring` (используется при создании, не при reuse)
>
> Для Scoped зависимостей (например, ICurrentUserService для multi-tenant) — использовать ContextScope или передавать через `DbContextOptionsBuilder`:
> ```csharp
> services.AddDbContextPool<AppDbContext>((sp, options) =>
> {
>     var tenantProvider = sp.GetRequiredService<ITenantProvider>();
>     options.UseNpgsql(GetConnectionString(tenantProvider));
> });
> ```

### Connection Exhaustion

```csharp
// ❌ Утечка соединений — не Dispose контекст
var context = new AppDbContext(options);
// Используется и не Dispose → соединение не возвращается в пул!

// ✅ Правильно — using или DI Scoped
await using var context = factory.CreateDbContext();
```

### Multiplexing (Npgsql)

Npgsql 6+ поддерживает multiplexing — одно физическое соединение мультиплексирует запросы от множества logical connections:

```csharp
options.UseNpgsql(connStr, b => b.UseMultiplexing());
```

**Когда:** read-heavy workload, тысячи concurrent operations. **Не подходит** когда нужны long-running connections (LISTEN/NOTIFY, COPY).

---

## SaveChanges Batching

EF Core автоматически группирует команды в один round-trip.

```csharp
// 1000 сущностей → ~10-25 batches
context.AddRange(thousandEntities);
await context.SaveChangesAsync(ct);  // не 1000 round-trips!

// Настройка
options.UseSqlServer(connStr, o => o.MaxBatchSize(100));
options.UseNpgsql(connStr, o => o.MaxBatchSize(100));
```

### MinBatchSize / MaxBatchSize

| Provider | Default MaxBatchSize |
|----------|---------------------|
| SQL Server | 42 |
| PostgreSQL | 1000 (через protocol-level batching) |
| SQLite | 1 (no batching, all in TX) |

### EF Core 7+ ExecuteUpdate / ExecuteDelete

Native bulk operations без change tracking:

```csharp
// Bulk delete — один SQL DELETE без загрузки entities
await context.Orders
    .Where(o => o.Status == "Cancelled" && o.CreatedAt < DateTime.UtcNow.AddYears(-1))
    .ExecuteDeleteAsync(ct);

// Bulk update
await context.Orders
    .Where(o => o.Status == "Pending")
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(o => o.Status, "Cancelled")
        .SetProperty(o => o.CancelledAt, DateTime.UtcNow), ct);
```

> [!warning] ExecuteUpdate/Delete пропускает interceptors и change tracker
> SaveChangesInterceptor не вызовется. Soft delete interceptor пропустится. Не вызывает domain events. Используй с пониманием.

---

## Async transactions ловушки

### 1. Забыть await commit

```csharp
// ❌ Утечка
await using var tx = await context.Database.BeginTransactionAsync(ct);
// ... операции ...
tx.CommitAsync(ct);  // забыл await → tx не закоммичен!

// ✅
await tx.CommitAsync(ct);
```

### 2. Использование context из разных потоков

DbContext **не thread-safe**. Параллельные операции:

```csharp
// ❌ Race condition
await Task.WhenAll(
    Task.Run(() => context.Orders.AddAsync(order1)),
    Task.Run(() => context.Orders.AddAsync(order2)));

// ✅ Один поток или разные context'ы
foreach (var order in orders)
    await context.Orders.AddAsync(order);

// или через factory
await Parallel.ForEachAsync(orders, async (order, ct) =>
{
    using var ctx = await contextFactory.CreateDbContextAsync(ct);
    await ctx.Orders.AddAsync(order);
    await ctx.SaveChangesAsync(ct);
});
```

### 3. Cancellation в середине transaction

```csharp
await using var tx = await context.Database.BeginTransactionAsync(ct);
try
{
    await context.SaveChangesAsync(ct);  // если cancelled
    await tx.CommitAsync(ct);
}
catch (OperationCanceledException)
{
    // tx будет автоматически rollback при Dispose
    throw;
}
```

`await using` гарантирует Dispose, который сделает rollback если не было commit.

---

## Common Pitfalls

### 1. Забыть Concurrency Token при изменении модели

Добавил optimistic concurrency на колонку `Total`, но забыл миграцию — в production EF проверяет колонку, которой нет в БД → ошибка.

### 2. Stale data + Database Wins без user notification

```csharp
catch (DbUpdateConcurrencyException ex)
{
    await ex.Entries[0].ReloadAsync();  // reload from DB
    await context.SaveChangesAsync();    // ← но мы перезаписываем своими старыми значениями!
}
```

DB wins **должен** информировать пользователя, что его изменения потерялись.

### 3. Long-running transactions

```csharp
await using var tx = await context.Database.BeginTransactionAsync();

await foreach (var item in stream)
{
    await ProcessAsync(item);  // 30 минут!
}

await tx.CommitAsync();
// Все эти 30 минут — locks в БД, другие пользователи страдают
```

**Решение:** короткие транзакции с batching, не один большой TX.

### 4. Implicit transaction в SaveChanges с rollback всех

```csharp
context.Orders.Add(goodOrder);
context.Orders.Add(badOrder);  // нарушает constraint

await context.SaveChangesAsync();  // оба rollback'нутся!
```

Если хочешь partial — отдельные SaveChanges или explicit transactions.

### 5. EF Core 7+ ExecuteUpdate без concurrency check

```csharp
// ❌ Игнорирует RowVersion
await context.Orders
    .Where(o => o.Id == id)
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.Total, newTotal));

// ✅ Если нужна optimistic — fetch + update + SaveChanges
```

### 6. Read after write в одной транзакции — может быть неконсистентно с replication

Если `Read Committed` + read replicas — read после write может попасть на replica → ещё не replicated. Требует **read-after-write consistency** (sticky session к primary).

### 7. Async + DbContext — context.SaveChangesAsync без ct

```csharp
// ❌ Не cancellable, может зависнуть
await context.SaveChangesAsync();

// ✅
await context.SaveChangesAsync(cancellationToken);
```

---

## Best Practices

- **Optimistic concurrency** — default для большинства случаев (RowVersion / xmin)
- **Pessimistic locking** — для финансов, inventory, booking
- **SELECT FOR UPDATE SKIP LOCKED** — для job queue
- **Короткие транзакции** — не делай HTTP/file I/O внутри TX
- **Фиксированный порядок локов** — избегаем deadlocks
- **EnableRetryOnFailure** — для transient errors + custom retry для deadlocks
- **CreateExecutionStrategy** — при retry + explicit transactions
- **Read Committed** — default, достаточно для 99% случаев
- **Serializable** — только для критичных financial операций (с retry)
- **Advisory locks (PG)** — для distributed coordination
- **RedLock** — для cross-service distributed locks
- **DbContextPool** — для stateless API (5-15% throughput gain)
- **Multiplexing** — для read-heavy Postgres workload
- **CancellationToken везде** — `await context.SaveChangesAsync(ct)`

---

## См. также

- [EF Core Basics & Tracking](basics-tracking.md)
- [EF Core Migrations](migrations.md)
- [EF Core Patterns](patterns.md)
- [PostgreSQL Deep — Advisory Locks, MVCC](../SQL/postgresql-deep.md)
- [SQL Optimization — Isolation, Deadlocks](../SQL/optimization.md)
- [Distributed Systems — Sagas, RedLock, Outbox](../Architecture/distributed-systems.md)
- [Concurrency и Atomics](../Runtime/concurrency-atomics.md) — низкоуровневая concurrency

## Reading list

- **Microsoft Docs — Concurrency conflicts** — learn.microsoft.com/ef/core/saving/concurrency
- **Microsoft Docs — Transactions** — learn.microsoft.com/ef/core/saving/transactions
- **Postgres docs — Concurrency Control** — postgresql.org/docs/current/mvcc.html
- **Martin Kleppmann — Designing Data-Intensive Applications** (главы 7-9 про transactions, isolation, distributed)
- **Andrew Lock — DbContextPool gotchas** — andrewlock.net
- **Postgres advisory locks** — leontrolski.github.io/postgres-advisory-locks.html
- **Redlock controversy** — антирассказ от Martin Kleppmann + ответ автора Redis
