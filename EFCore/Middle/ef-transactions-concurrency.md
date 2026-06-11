---
tags: [efcore, transactions, concurrency, isolation, optimistic, pessimistic, middle]
level: Middle
date: 2026-05-10
---

# EF Core Transactions & Concurrency — ACID, isolation, optimistic locking

> **Транзакции в EF Core, isolation levels, optimistic concurrency tokens, deadlock handling, distributed transactions.** Critical для финансовых, e-commerce, любых систем с concurrent writes.

---

## 0. Как читать

После `Junior/ef-basics.md` (SaveChanges basics). Перед `Senior/concurrency.md` (deep production patterns). Здесь — practical: SaveChanges как transaction, explicit BeginTransaction, optimistic concurrency через RowVersion, deadlock retry strategies.

---

## 1. SaveChanges — implicit transaction

### 1.1. Что делает SaveChanges

```csharp
public async Task ProcessOrderAsync(int orderId, decimal payment)
{
    var order = await _db.Orders.FindAsync(orderId);
    order.Status = OrderStatus.Paid;
    order.PaidAt = DateTime.UtcNow;
    
    var account = await _db.Accounts.FindAsync(order.AccountId);
    account.Balance -= payment;
    
    _db.PaymentLogs.Add(new PaymentLog { OrderId = orderId, Amount = payment });
    
    await _db.SaveChangesAsync();
    // ATOMIC: либо ВСЕ 3 операции (UPDATE Order + UPDATE Account + INSERT PaymentLog) commit,
    // либо НИЧЕГО (rollback).
}
```

EF Core автоматически:
1. Открывает transaction
2. Выполняет SQL для всех tracked changes
3. Commit при success
4. Rollback при exception

### 1.2. Почему это важно

```csharp
// Без transaction — половинчатое состояние возможно
public async Task ProcessOrderBroken(int orderId, decimal payment)
{
    var order = await _db.Orders.FindAsync(orderId);
    order.Status = OrderStatus.Paid;
    await _db.SaveChangesAsync();   // commit 1
    
    // 💥 Crash здесь!
    
    var account = await _db.Accounts.FindAsync(order.AccountId);
    account.Balance -= payment;
    await _db.SaveChangesAsync();   // не вызвался
}
// Order оплачен в БД, но account не списан → inconsistent state
```

```csharp
// С one SaveChanges — atomic
public async Task ProcessOrderCorrect(int orderId, decimal payment)
{
    var order = await _db.Orders.FindAsync(orderId);
    order.Status = OrderStatus.Paid;
    
    var account = await _db.Accounts.FindAsync(order.AccountId);
    account.Balance -= payment;
    
    await _db.SaveChangesAsync();
    // Atomic — либо обе change, либо ни одна
}
```

> [!question]- **Интервью: SaveChanges это transaction?**
> **Да** — implicit transaction внутри одного SaveChanges. EF Core открывает SQL transaction, выполняет все pending changes (INSERT/UPDATE/DELETE), commits. При exception — rollback всех. **Best practice**: один SaveChanges на logical operation. Несколько Add/Remove/property changes → один SaveChanges. **Не** делай SaveChanges в loop — каждый = отдельная transaction. **Cross-DbContext**: Different DbContexts → different transactions. Если нужна shared transaction → explicit BeginTransaction (см. ниже) или TransactionScope.

---

## 2. Explicit Transactions — BeginTransaction

### 2.1. Когда нужно

- Multiple SaveChanges в одной transaction
- Mix EF Core + Dapper
- Custom SQL execution
- Save-point patterns

### 2.2. Базовая форма

```csharp
public async Task<Result> TransferMoneyAsync(int fromId, int toId, decimal amount)
{
    using var transaction = await _db.Database.BeginTransactionAsync();
    try
    {
        var from = await _db.Accounts.FindAsync(fromId);
        var to = await _db.Accounts.FindAsync(toId);
        
        if (from.Balance < amount)
        {
            await transaction.RollbackAsync();
            return Result.Fail("Insufficient funds");
        }
        
        from.Balance -= amount;
        await _db.SaveChangesAsync();   // UPDATE в transaction
        
        to.Balance += amount;
        await _db.SaveChangesAsync();   // ещё UPDATE в той же transaction
        
        await transaction.CommitAsync();
        return Result.Ok();
    }
    catch
    {
        await transaction.RollbackAsync();
        throw;
    }
}
```

### 2.3. Mix с Dapper в одной transaction

```csharp
public async Task ProcessAsync(int orderId)
{
    using var transaction = await _db.Database.BeginTransactionAsync();
    
    // 1. EF Core operation
    var order = await _db.Orders.FindAsync(orderId);
    order.Status = OrderStatus.Processing;
    await _db.SaveChangesAsync();
    
    // 2. Dapper operation в той же transaction
    var connection = _db.Database.GetDbConnection();
    var dbTransaction = transaction.GetDbTransaction();
    
    await connection.ExecuteAsync(
        "INSERT INTO AuditLog (OrderId, Action, Time) VALUES (@id, @act, @time)",
        new { id = orderId, act = "Processing", time = DateTime.UtcNow },
        transaction: dbTransaction);
    
    await transaction.CommitAsync();
}
```

### 2.4. SaveChanges с auto-rollback при exception

```csharp
// SaveChanges НЕ нужен manual rollback при exception
// EF Core сам rollback transaction при exception
public async Task UpdateMultipleAsync()
{
    var users = await _db.Users.ToListAsync();
    foreach (var user in users)
    {
        user.LastLogin = DateTime.UtcNow;
    }
    
    try
    {
        await _db.SaveChangesAsync();
        // если throws — transaction rollback автоматически
    }
    catch (DbUpdateException ex)
    {
        _logger.LogError(ex, "Update failed");
        // No need to rollback — already done
    }
}
```

Manual rollback нужен **только** для explicit BeginTransaction.

---

## 3. Isolation Levels — что видит concurrent code

### 3.1. SQL Standard isolation levels

```
                     Dirty   Non-Repeatable   Phantom
                     Read    Read              Read
Read Uncommitted     ✓       ✓                 ✓
Read Committed       ✗       ✓                 ✓
Repeatable Read      ✗       ✗                 ✓
Serializable         ✗       ✗                 ✗
Snapshot (MVCC)      ✗       ✗                 ✗
```

### 3.2. Что значит каждое явление

```
Dirty Read:
  T1: UPDATE Account SET Balance = 0 WHERE Id = 1
  T2: SELECT Balance FROM Account WHERE Id = 1   → 0
  T1: ROLLBACK
  T2 видел uncommitted данные (которые откатились)

Non-Repeatable Read:
  T1: SELECT Balance FROM Account WHERE Id = 1   → 100
  T2: UPDATE Account SET Balance = 200 WHERE Id = 1; COMMIT
  T1: SELECT Balance FROM Account WHERE Id = 1   → 200
  Тот же row дал разный результат в одной transaction

Phantom Read:
  T1: SELECT * FROM Orders WHERE Total > 100   → 5 rows
  T2: INSERT INTO Orders (Total) VALUES (200); COMMIT
  T1: SELECT * FROM Orders WHERE Total > 100   → 6 rows
  Появилась новая row в той же transaction
```

### 3.3. Defaults

```
PostgreSQL: Read Committed (с MVCC под капотом)
SQL Server: Read Committed (с pessimistic locking)
SQLite: Serializable (single-writer model)
MySQL InnoDB: Repeatable Read (в .NET через Pomelo)
```

### 3.4. Изменение isolation level

```csharp
using var transaction = await _db.Database.BeginTransactionAsync(
    IsolationLevel.RepeatableRead);

// All reads/writes использующие transaction follow RepeatableRead
var order = await _db.Orders.FindAsync(orderId);
// ... 

await transaction.CommitAsync();
```

### 3.5. Когда какой level

```
✅ Read Committed (default):
- 95% случаев
- Acceptable для большинства apps

✅ Repeatable Read:
- Финансовые операции
- "Прочитать → подумать → решить → записать" patterns
- Inventory checks

✅ Serializable:
- Only когда нужна полная strict consistency
- Reporting consistency
- Cost: контестность locks высокая

✅ Snapshot (MVCC, PostgreSQL default):
- Read-heavy workloads
- Long-running reads не блокируют writers
- Trade-off: storage для row versions
```

### 3.6. Read Uncommitted — почти никогда

```csharp
// SELECT WITH (NOLOCK) в SQL Server
using var transaction = await _db.Database.BeginTransactionAsync(
    IsolationLevel.ReadUncommitted);

var stats = await _db.Orders.CountAsync();
// Может вернуть incorrect count (uncommitted data)
```

Use cases: **monitoring dashboards** где приблизительно OK. Никогда для business logic.

> [!question]- **Интервью: какой isolation level выбрать для финансовой operation?**
> **Repeatable Read** или выше для transactions с pattern "read-decide-write" (например списание баланса). **Read Committed** (default) недостаточно — между прочтением баланса и обновлением другая transaction может изменить значение → race condition. **Snapshot/Serializable** для extreme consistency. **Better alternative**: optimistic concurrency с RowVersion — без блокировок, при conflict retry. **Production**: combination — Repeatable Read + RowVersion для defense in depth.

---

## 4. Optimistic Concurrency — RowVersion

### 4.1. Проблема

```csharp
// Two users editing the same product simultaneously
// User A: Load product (version 1)
// User B: Load product (version 1)
// User A: Update price to 100, save (DB version 2)
// User B: Update description, save (DB version 3, but A's price overwritten!)

// Lost update problem
```

### 4.2. Решение — RowVersion / Timestamp

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public decimal Price { get; set; }
    
    [Timestamp]   // SQL Server timestamp / rowversion
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}

// EF Core configuration
modelBuilder.Entity<Product>()
    .Property(p => p.RowVersion)
    .IsRowVersion();   // Fluent API alternative
```

### 4.3. Как это работает

```csharp
// User A
var product = await _db.Products.FindAsync(1);   // RowVersion = 0x00000001
product.Price = 100;
await _db.SaveChangesAsync();
// SQL: UPDATE Products SET Price = 100, RowVersion = 0x00000002 WHERE Id = 1 AND RowVersion = 0x00000001
// Successfully updated

// User B (had same loaded version)
var product2 = await _db.Products.FindAsync(1);   // загружен раньше, RowVersion = 0x00000001 в memory
product2.Description = "...";
await _db.SaveChangesAsync();
// SQL: UPDATE Products SET Description = '...', RowVersion = 0x00000003 WHERE Id = 1 AND RowVersion = 0x00000001
// 0 rows affected → DbUpdateConcurrencyException!
```

### 4.4. Handle concurrency exception

```csharp
public async Task<Result> UpdateProductAsync(int id, decimal newPrice)
{
    try
    {
        var product = await _db.Products.FindAsync(id);
        product.Price = newPrice;
        await _db.SaveChangesAsync();
        return Result.Ok();
    }
    catch (DbUpdateConcurrencyException ex)
    {
        // Someone modified between our load and save
        return Result.Fail("Product was modified by another user. Reload and try again.");
    }
}
```

### 4.5. Resolve conflict — manual merge

```csharp
public async Task UpdateWithConflictResolutionAsync(int id, Action<Product> changes)
{
    var product = await _db.Products.FindAsync(id);
    
    while (true)
    {
        try
        {
            changes(product);
            await _db.SaveChangesAsync();
            return;
        }
        catch (DbUpdateConcurrencyException ex)
        {
            foreach (var entry in ex.Entries)
            {
                var dbValues = await entry.GetDatabaseValuesAsync();
                if (dbValues == null)
                {
                    // Row was deleted
                    throw;
                }
                
                // Strategy 1: client wins — overwrite DB values with current
                // entry.OriginalValues.SetValues(dbValues);
                
                // Strategy 2: store wins — abort our changes
                // entry.CurrentValues.SetValues(dbValues);
                
                // Strategy 3: merge — keep DB values for conflicting fields
                var dbProduct = dbValues.ToObject() as Product;
                product.RowVersion = dbProduct.RowVersion;   // accept new version
                // Re-apply our changes
            }
        }
    }
}
```

### 4.6. Property-level concurrency tokens

```csharp
public class Product
{
    public int Id { get; set; }
    
    [ConcurrencyCheck]   // Этот property проверяется как RowVersion
    public decimal Price { get; set; }
}
```

```sql
-- EF generates:
UPDATE Products SET Price = 200 WHERE Id = 1 AND Price = 100;
-- Если другой transaction изменил Price → 0 rows → exception
```

Useful когда не хочешь использовать row version (e.g. legacy schema без RowVersion column).

> [!question]- **Интервью: optimistic vs pessimistic concurrency?**
> **Optimistic** — assumes conflicts rare. Read entity → modify → save с проверкой "никто не изменил с момента read". В .NET через `[Timestamp]` / RowVersion. При conflict — **retry или fail**. Plus: no locks, high concurrency. Minus: required retry logic. **Pessimistic** — explicit lock на read (`SELECT ... FOR UPDATE` в Postgres / `WITH (UPDLOCK)` в SQL Server). Lock держится до commit. Plus: no conflicts. Minus: blocking, deadlocks, scalability issues. **Best practice 2024+**: optimistic как default + retry, pessimistic только для specific scenarios (queue processing, inventory reservation).

---

## 5. Pessimistic Locking — explicit locks

### 5.1. Когда нужно

- Inventory reservation (lock product before payment)
- Queue processing (one worker exclusively)
- Long transactions с manual coordination
- Когда optimistic постоянно конфликтует

### 5.2. SELECT FOR UPDATE через raw SQL

```csharp
// PostgreSQL
public async Task<Product?> GetForUpdateAsync(int id)
{
    return await _db.Products
        .FromSqlRaw("SELECT * FROM Products WHERE Id = {0} FOR UPDATE", id)
        .FirstOrDefaultAsync();
}

// SQL Server
public async Task<Product?> GetForUpdateAsync(int id)
{
    return await _db.Products
        .FromSqlRaw("SELECT * FROM Products WITH (UPDLOCK, ROWLOCK) WHERE Id = {0}", id)
        .FirstOrDefaultAsync();
}
```

Lock держится до commit transaction. Other transactions ждут.

### 5.3. Inventory reservation pattern

```csharp
public async Task<Result> ReserveProductAsync(int productId, int quantity)
{
    using var transaction = await _db.Database.BeginTransactionAsync(IsolationLevel.RepeatableRead);
    
    // Lock product row
    var product = await _db.Products
        .FromSqlRaw("SELECT * FROM Products WHERE Id = {0} FOR UPDATE", productId)
        .FirstOrDefaultAsync();
    
    if (product == null)
    {
        await transaction.RollbackAsync();
        return Result.Fail("Product not found");
    }
    
    if (product.AvailableStock < quantity)
    {
        await transaction.RollbackAsync();
        return Result.Fail("Insufficient stock");
    }
    
    product.AvailableStock -= quantity;
    product.ReservedStock += quantity;
    
    await _db.SaveChangesAsync();
    await transaction.CommitAsync();
    
    return Result.Ok();
}
```

### 5.4. Deadlocks

```
Transaction A: lock Product 1, ждёт Product 2
Transaction B: lock Product 2, ждёт Product 1
→ DEADLOCK
```

DBMS detects и убивает одну transaction (обычно "victim"). Получаешь exception.

### 5.5. Avoid deadlocks

```
Rules:
1. Always lock в одинаковом порядке
   if (id1 < id2) { lock(id1); lock(id2); }
   else { lock(id2); lock(id1); }

2. Short transactions — minimize lock duration
3. Lock on minimum scope — not whole table
4. Use indexes для быстрых locks
5. Retry deadlock-victims с exponential backoff
```

### 5.6. Deadlock retry pattern

```csharp
public async Task ExecuteWithRetryAsync(Func<Task> operation)
{
    const int maxRetries = 3;
    for (int attempt = 0; attempt < maxRetries; attempt++)
    {
        try
        {
            await operation();
            return;
        }
        catch (DbUpdateException ex) when (IsDeadlock(ex))
        {
            if (attempt == maxRetries - 1) throw;
            await Task.Delay(TimeSpan.FromMilliseconds(Math.Pow(2, attempt) * 100));
        }
    }
}

private static bool IsDeadlock(DbUpdateException ex) =>
    // SQL Server error 1205, PostgreSQL 40P01
    ex.InnerException?.Message.Contains("deadlock", StringComparison.OrdinalIgnoreCase) == true;
```

EF Core 6+ имеет встроенный retry через `EnableRetryOnFailure`:

```csharp
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(connStr, sqlOpt =>
    {
        sqlOpt.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorNumbersToAdd: null);   // SQL Server-specific transient errors
    }));
```

---

## 6. Distributed Transactions — TransactionScope

### 6.1. Когда нужно

Multiple DBs / services — нужна atomic consistency между ними.

```csharp
using (var scope = new TransactionScope(
    TransactionScopeAsyncFlowOption.Enabled))
{
    // Operation 1: SQL Server
    await _sqlContext.SaveChangesAsync();
    
    // Operation 2: PostgreSQL
    await _pgContext.SaveChangesAsync();
    
    scope.Complete();   // commit both atomically (через MSDTC)
}
```

### 6.2. Проблемы

- Требует MSDTC (Microsoft Distributed Transaction Coordinator) или PostgreSQL prepared transactions
- Performance penalty
- Сложность configuration
- На Linux — limited support
- В cloud — часто недоступен

### 6.3. Alternative — Outbox Pattern

Лучшее решение для distributed scenarios — **Outbox pattern**:

```csharp
public async Task ProcessOrderAsync(Order order)
{
    using var transaction = await _db.Database.BeginTransactionAsync();
    
    // 1. Save order
    _db.Orders.Add(order);
    
    // 2. Save outbox message (в той же DB transaction!)
    _db.OutboxMessages.Add(new OutboxMessage
    {
        Type = "OrderCreated",
        Payload = JsonSerializer.Serialize(new { order.Id, order.Total }),
        CreatedAt = DateTime.UtcNow
    });
    
    await _db.SaveChangesAsync();
    await transaction.CommitAsync();
    
    // 3. Background worker реад outbox и публикует в queue (eventually consistent)
}
```

Detailed treatment в `Architecture/Senior/distributed-systems.md`.

> [!question]- **Интервью: distributed transactions в .NET — за или против?**
> **Против**, кроме narrow cases. **TransactionScope** работает (через MSDTC на Windows), но: 1) **Performance penalty** (2-phase commit). 2) **Не работает в Linux/cloud** надёжно. 3) **Coordinator failure** — система может зависнуть. **Modern alternative**: **Outbox pattern** + eventual consistency. Save в outbox внутри single-DB transaction, отдельный worker публикует events в queue. Если worker fails — retry. Если consumer fails — retry. Eventually consistent, scalable, cloud-friendly. **Microservices**: Saga pattern (orchestration или choreography) — explicit compensation вместо ACID.

---

## 7. Common pitfalls

### 7.1. SaveChanges в loop

```csharp
// ❌ N transactions
foreach (var user in users)
{
    user.LastLogin = DateTime.UtcNow;
    await _db.SaveChangesAsync();
}

// ✅ Один transaction
foreach (var user in users)
{
    user.LastLogin = DateTime.UtcNow;
}
await _db.SaveChangesAsync();
```

### 7.2. Long-running transaction

```csharp
// ❌ Transaction открыт пока обрабатываем 1M rows
using var transaction = await _db.Database.BeginTransactionAsync();
var allRows = await _db.LargeTable.ToListAsync();   // 1M rows
foreach (var row in allRows)
{
    // ... long processing ...
}
await _db.SaveChangesAsync();
await transaction.CommitAsync();
// Locks held 30+ minutes — другие transactions блокируются
```

**Фикс**: batch processing с smaller transactions.

### 7.3. Nested transactions

```csharp
// ❌ Вложенные transactions не supported в EF Core
using var outer = await _db.Database.BeginTransactionAsync();
using var inner = await _db.Database.BeginTransactionAsync();   // throws!
```

**Фикс**: использовать `Savepoints`:

```csharp
using var transaction = await _db.Database.BeginTransactionAsync();
await transaction.CreateSavepointAsync("BeforeRiskyOp");

try
{
    // Risky operation
    await _db.SaveChangesAsync();
}
catch
{
    await transaction.RollbackToSavepointAsync("BeforeRiskyOp");
    // Continue с другой логикой
}

await transaction.CommitAsync();
```

### 7.4. Concurrency exception без catch

```csharp
// ❌ Unhandled DbUpdateConcurrencyException → 500 error
await _db.SaveChangesAsync();
```

**Фикс**: always handle для concurrent-write scenarios.

### 7.5. Async transaction без AsyncFlow

```csharp
// ❌ TransactionScope без async flow
using (var scope = new TransactionScope())
{
    await _db.SaveChangesAsync();   // ❌ Throws!
}

// ✅ С AsyncFlowOption
using (var scope = new TransactionScope(
    TransactionScopeAsyncFlowOption.Enabled))
{
    await _db.SaveChangesAsync();
    scope.Complete();
}
```

### 7.6. Forgot Commit / RollbackOnDispose

```csharp
using var transaction = await _db.Database.BeginTransactionAsync();
await _db.SaveChangesAsync();
// ❌ Forgot Commit → автоматический rollback при Dispose
```

EF Core 6+ авто-rollback unhandled transactions при dispose. Это safety net, но **always commit explicitly**.

### 7.7. RowVersion не added в migration

```csharp
public class Product
{
    [Timestamp]
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}

// Если migration не сгенерирована для existing table:
// dotnet ef migrations add AddRowVersionToProduct
// Иначе RowVersion будет всегда null → exceptions
```

### 7.8. Detached entity update без version

```csharp
// Client отправил entity (через API) с старым RowVersion
public async Task<IActionResult> Update([FromBody] Product product)
{
    _db.Products.Update(product);
    await _db.SaveChangesAsync();
    // Если client RowVersion устарел → DbUpdateConcurrencyException
    // Это GOOD — клиент имеет stale data
}
```

### 7.9. EnableRetryOnFailure + manual transactions конфликт

```csharp
// EnableRetryOnFailure не работает с manual BeginTransaction по default
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(connStr, sqlOpt => sqlOpt.EnableRetryOnFailure()));

// В коде:
using var transaction = await _db.Database.BeginTransactionAsync();   // ⚠️ throws!
```

**Фикс**: использовать execution strategy:

```csharp
var strategy = _db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    using var transaction = await _db.Database.BeginTransactionAsync();
    // ... operations ...
    await transaction.CommitAsync();
});
```

### 7.10. Mixed isolation levels

```csharp
// ❌ Меняешь isolation level в running transaction
using var transaction = await _db.Database.BeginTransactionAsync(IsolationLevel.ReadCommitted);
// ... работа ...
// нельзя upgrade на Serializable mid-transaction
```

**Фикс**: choose isolation level at start, не меняй.

> [!question]- **Интервью: топ-3 ошибки с EF transactions?**
> 1) **SaveChanges в loop** — N transactions вместо 1, slow + не atomic. Fix: collect changes → один SaveChanges. 2) **Long-running transaction** — locks held надолго → блокировка. Fix: batch processing с small transactions. 3) **Не handle DbUpdateConcurrencyException** — 500 errors при race conditions. Fix: try-catch + retry или meaningful error. **Bonus**: nested transactions не supported, use Savepoints. **Bonus 2**: EnableRetryOnFailure + manual BeginTransaction конфликт — use ExecutionStrategy.

---

## 8. Cheat sheet

```csharp
// === Implicit transaction (SaveChanges) ===
await _db.SaveChangesAsync();   // atomic для всех pending changes

// === Explicit transaction ===
using var transaction = await _db.Database.BeginTransactionAsync();
try
{
    // work...
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}

// === С isolation level ===
using var transaction = await _db.Database.BeginTransactionAsync(
    IsolationLevel.RepeatableRead);

// === Savepoints (nested) ===
await transaction.CreateSavepointAsync("MyPoint");
await transaction.RollbackToSavepointAsync("MyPoint");

// === Optimistic concurrency ===
public class Product
{
    [Timestamp]
    public byte[] RowVersion { get; set; } = Array.Empty<byte>();
}

try
{
    await _db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException ex)
{
    // Resolve conflict
}

// === Property-level concurrency ===
public class Product
{
    [ConcurrencyCheck]
    public decimal Price { get; set; }
}

// === Pessimistic lock (raw SQL) ===
// PostgreSQL
await _db.Products
    .FromSqlRaw("SELECT * FROM Products WHERE Id = {0} FOR UPDATE", id)
    .FirstOrDefaultAsync();

// SQL Server
await _db.Products
    .FromSqlRaw("SELECT * FROM Products WITH (UPDLOCK) WHERE Id = {0}", id)
    .FirstOrDefaultAsync();

// === Retry on failure ===
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(connStr, sqlOpt =>
        sqlOpt.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorNumbersToAdd: null)));

// === Execution strategy для manual transactions ===
var strategy = _db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    using var transaction = await _db.Database.BeginTransactionAsync();
    // work...
    await transaction.CommitAsync();
});

// === Mix EF + Dapper в одной transaction ===
using var transaction = await _db.Database.BeginTransactionAsync();
await _db.SaveChangesAsync();   // EF
var conn = _db.Database.GetDbConnection();
await conn.ExecuteAsync(
    "INSERT INTO ...",
    transaction: transaction.GetDbTransaction());
await transaction.CommitAsync();
```

---

## 9. Practice exercises

### 9.1. Money transfer — implement правильно

Реализуй `TransferAsync(int fromId, int toId, decimal amount)`:
- Atomic operation (либо обе ноги trans, либо ничего)
- Проверка sufficient funds в source
- Currency match
- Optimistic concurrency для balance
- Retry на concurrency conflicts
- Logging audit trail

### 9.2. Inventory reservation

E-commerce checkout:
1. Reserve N items of product
2. Process payment (slow external API)
3. Confirm reservation OR release if payment failed

Должно быть resilient к concurrent reservations.

### 9.3. Detect concurrency issue

Найди проблему:

```csharp
public async Task<int> IncrementCounterAsync(int counterId)
{
    var counter = await _db.Counters.FindAsync(counterId);
    counter.Value++;
    await _db.SaveChangesAsync();
    return counter.Value;
}

// 100 concurrent calls → expected counter = original + 100
// Actual: counter = original + 30-50 (lost updates)
```

Что не так? Как fix?

---

## 10. Что читать дальше

1. **`EFCore/Senior/concurrency.md`** — production patterns deep
2. **`EFCore/Senior/queries-performance.md`** — performance + N+1
3. **`SQL/Middle/indexes-deep.md`** — indexes для concurrency
4. **`Architecture/Senior/distributed-systems.md`** — Outbox, Saga
5. **`SQL/Senior/optimization.md`** — isolation levels production

---

## 11. См. также

- [[ef-basics|EFCore/Junior/ef-basics]] — basics
- [[ef-loading-strategies|EFCore/Middle/ef-loading-strategies]] — loading
- [[concurrency|EFCore/Senior/concurrency]] — production patterns
- [[distributed-systems|Architecture/Senior/distributed-systems]] — Outbox pattern
- [[indexes-deep|SQL/Middle/indexes-deep]] — indexes для locks

---

## 12. Reading list

- **Microsoft Docs — Transactions** — learn.microsoft.com/ef/core/saving/transactions
- **Microsoft Docs — Concurrency Conflicts** — learn.microsoft.com/ef/core/saving/concurrency
- **PostgreSQL Docs — Concurrency Control** — postgresql.org/docs/current/mvcc.html
- **SQL Server Locking Hints** — learn.microsoft.com/sql/t-sql/queries/hints-transact-sql-table
- **"Designing Data-Intensive Applications" — Kleppmann** (chapters on transactions)
- **"Database Internals" — Alex Petrov** (locking, MVCC)
