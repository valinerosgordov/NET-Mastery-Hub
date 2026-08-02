---
tags: [efcore, bulk, executeupdate, executedelete, performance, middle, batch]
level: Middle
date: 2026-08-02
---

# EF Core Bulk Operations — ExecuteUpdate, ExecuteDelete, batch operations

> **Bulk INSERT/UPDATE/DELETE без change tracking. ExecuteUpdate/ExecuteDelete (EF Core 7+), EFCore.BulkExtensions, raw SQL bulk patterns.** Critical для production migrations, mass updates, ETL processes.

---

## 0. Как читать

После `Junior/ef-basics.md` (basic CRUD). Здесь — performance: что делать когда нужно обновить миллион rows. Перед `Senior/queries-performance.md` (N+1 deep).

---

## 1. Проблема — change tracking overhead

### 1.1. Standard EF подход

```csharp
// Mark all inactive users as deleted
var inactiveUsers = await _db.Users
    .Where(u => !u.IsActive && u.LastLogin < DateTime.UtcNow.AddYears(-1))
    .ToListAsync();   // загружает в memory ВСЕ matching rows!

foreach (var user in inactiveUsers)
{
    user.IsDeleted = true;
}

await _db.SaveChangesAsync();   // N UPDATE statements (по одному на user)
```

Проблемы:
- 1M users → 1M objects в memory → memory exhausted
- 1M UPDATE statements → slow
- Change tracker holds 1M snapshots
- Один SaveChanges медленный (хоть один transaction)

### 1.2. Что нужно в production

```sql
-- One SQL для миллионов rows
UPDATE Users SET IsDeleted = 1 
WHERE IsActive = 0 AND LastLogin < '2025-01-01';
```

Один statement, БД делает работу, никаких C# objects.

EF Core 7+ имеет это **встроенно**.

---

## 2. ExecuteUpdate (EF Core 7+)

### 2.1. Простой UPDATE

```csharp
// Mass update — один SQL statement
int affectedRows = await _db.Users
    .Where(u => !u.IsActive && u.LastLogin < DateTime.UtcNow.AddYears(-1))
    .ExecuteUpdateAsync(u => u.SetProperty(x => x.IsDeleted, true));

// SQL: UPDATE Users SET IsDeleted = 1 WHERE IsActive = 0 AND LastLogin < '2025-01-01'
// Returns: count of affected rows
```

### 2.2. Multiple SetProperty

```csharp
await _db.Users
    .Where(u => u.Email == oldEmail)
    .ExecuteUpdateAsync(u => u
        .SetProperty(x => x.Email, newEmail)
        .SetProperty(x => x.UpdatedAt, DateTime.UtcNow)
        .SetProperty(x => x.EmailVerified, false));
```

### 2.3. Computed values

```csharp
// Increment counter
await _db.Products
    .Where(p => p.CategoryId == categoryId)
    .ExecuteUpdateAsync(p => p
        .SetProperty(x => x.ViewCount, x => x.ViewCount + 1));

// Conditional update
await _db.Orders
    .Where(o => o.Status == OrderStatus.Pending && o.CreatedAt < cutoff)
    .ExecuteUpdateAsync(o => o
        .SetProperty(x => x.Status, OrderStatus.Expired)
        .SetProperty(x => x.UpdatedAt, x => DateTime.UtcNow));
```

### 2.4. SetProperty с связанными полями

```csharp
// Discount всем продуктам категории на 10%
await _db.Products
    .Where(p => p.CategoryId == categoryId)
    .ExecuteUpdateAsync(p => p
        .SetProperty(x => x.Price, x => x.Price * 0.9m));
```

### 2.5. Returns affected count

```csharp
int rowsUpdated = await _db.Users
    .Where(u => u.IsDeleted)
    .ExecuteUpdateAsync(u => u.SetProperty(x => x.PurgeAt, DateTime.UtcNow.AddDays(30)));

_logger.LogInformation("Marked {Count} users for purge", rowsUpdated);
```

### 2.6. Преимущества

```
✅ Один SQL statement для миллионов rows
✅ Никаких objects в memory
✅ Никакого change tracking overhead
✅ Atomic (один transaction)
✅ Returns affected count
✅ Type-safe LINQ syntax
```

### 2.7. Ограничения

```
❌ Нельзя обновлять через navigation properties (только SetProperty на сам entity)
❌ Не trigger Domain Events (если есть)
❌ Не interceptors save changes
❌ Не optimistic concurrency check
❌ Не auditing через DbContext SaveChanges hooks
```

> [!question]- **Интервью: ExecuteUpdate vs traditional update?**
> **ExecuteUpdate** (EF Core 7+) — генерирует **один UPDATE SQL** для всех matching rows. **Traditional**: load entities → modify → SaveChanges → N UPDATEs. **ExecuteUpdate plus**: 1) **Performance** — миллионы rows в один statement. 2) **Memory** — не загружает в C#. 3) **Atomicity** — один SQL transaction. **Minus**: 1) **Не triggers Domain Events / Interceptors / Auditing** через DbContext. 2) **Не optimistic concurrency check**. 3) **Сложно для multi-table updates**. **Use cases**: data cleanup, mass status changes, ETL, cache warming. **Когда не use**: critical business operations с auditing, complex graph updates.

### 2.8. Эволюция ExecuteUpdate (EF Core 9 → 10)

API не стоит на месте — каждая версия снимает часть кейсов, ради которых раньше брали EFCore.BulkExtensions или raw SQL:

- **EF Core 9** — `SetProperty` принимает **complex type целиком**: EF сам разворачивает его в UPDATE всех замапленных колонок (раньше каждый член перечислялся вручную).
- **EF Core 10** — сеттеры принимаются как **обычная лямбда, а не expression tree**: сеттеры можно строить динамически (условный `if` внутри), без ручной сборки `Expression`. Плюс ExecuteUpdate работает по **JSON-колонкам через complex types** — bulk update отдельного свойства внутри JSON одним SQL.

```csharp
// EF Core 9 — complex type одним SetProperty
var newAddress = new Address("Line 1", null, "Beetley", "Norfolk", "NR20 4DR");
await _db.Stores
    .Where(s => s.Region == "Germany")
    .ExecuteUpdateAsync(s => s.SetProperty(x => x.StoreAddress, newAddress));
// SQL: UPDATE ... SET StoreAddress_Line1 = ..., StoreAddress_City = ..., ...

// EF Core 10 — обычная лямбда: сеттеры добавляются условно
await _db.Blogs.ExecuteUpdateAsync(s =>
{
    s.SetProperty(b => b.Views, 0);
    if (resetName)
    {
        s.SetProperty(b => b.Name, "Unnamed");   // добавляется только по условию
    }
});
```

До EF Core 10 условные сеттеры требовали ручной композиции expression tree — error-prone boilerplate. Каждый `SetProperty` по-прежнему должен транслироваться в SQL.

---

## 3. ExecuteDelete (EF Core 7+)

### 3.1. Простой DELETE

```csharp
// Удалить inactive users
int deleted = await _db.Users
    .Where(u => !u.IsActive && u.LastLogin < DateTime.UtcNow.AddYears(-2))
    .ExecuteDeleteAsync();

// SQL: DELETE FROM Users WHERE IsActive = 0 AND LastLogin < '2024-01-01'
```

### 3.2. Cascade delete через related

```csharp
// Удалить orders → SQL cascade удалит OrderItems (если FK с CASCADE)
int deleted = await _db.Orders
    .Where(o => o.CreatedAt < archiveDate)
    .ExecuteDeleteAsync();
```

Если FK без CASCADE — нужно DELETE children сначала или будет FK violation.

### 3.3. Soft delete pattern

```csharp
// Soft delete instead of hard delete
await _db.Users
    .Where(u => u.LastLogin < cutoff)
    .ExecuteUpdateAsync(u => u
        .SetProperty(x => x.IsDeleted, true)
        .SetProperty(x => x.DeletedAt, DateTime.UtcNow));
```

С Global Query Filter в DbContext:

```csharp
modelBuilder.Entity<User>()
    .HasQueryFilter(u => !u.IsDeleted);
// Все queries автоматически filter out deleted
```

---

## 4. Bulk INSERT — варианты

### 4.1. EF Core встроенный — AddRange

```csharp
var users = new List<User>();
for (int i = 0; i < 10_000; i++)
{
    users.Add(new User { Name = $"User{i}", Email = $"user{i}@x.com" });
}

_db.Users.AddRange(users);
await _db.SaveChangesAsync();
```

EF Core batches INSERT statements (до 42 rows per batch для SQL Server). Лучше чем foreach + Add + SaveChanges, но **всё равно медленно** для миллионов rows.

```sql
-- Generated:
INSERT INTO Users (Name, Email) VALUES (@p0, @p1), (@p2, @p3), ...   -- 42 rows
INSERT INTO Users (Name, Email) VALUES (@p84, @p85), ...
-- ...
```

### 4.2. EFCore.BulkExtensions — для миллионов

> [!warning]- License: EFCore.BulkExtensions — cFOSS с января 2024
> Dual license (cFOSS): бесплатно только personal use, non-profit и компании с выручкой < $1M/год — остальным платная лицензия. На NuGet есть MIT-форк **EFCore.BulkExtensions.MIT** с тем же API. И прежде чем брать библиотеку вообще — проверь, не закрывает ли кейс нативный `ExecuteUpdate`/`ExecuteDelete` (§2) или PG binary `COPY` (§4.4): бесплатно и без зависимости. Framework выбора — [[choosing-dependencies|Choosing Dependencies]].

```bash
dotnet add package EFCore.BulkExtensions
```

```csharp
using EFCore.BulkExtensions;

var users = GenerateUsers(1_000_000);

await _db.BulkInsertAsync(users);
// Использует SqlBulkCopy (SQL Server) / COPY (PostgreSQL) под капотом
// Performance: 100x быстрее AddRange + SaveChanges
```

```
Benchmark (insert 1M rows):
- foreach + Add + SaveChanges:  ~5 minutes
- AddRange + SaveChanges:        ~30 seconds
- BulkInsertAsync:               ~3 seconds
```

### 4.3. Bulk update / delete с EFCore.BulkExtensions

```csharp
// Bulk update (analogue ExecuteUpdate но с custom logic)
var users = await _db.Users.ToListAsync();
foreach (var user in users)
{
    user.LastSeenAt = DateTime.UtcNow;
}
await _db.BulkUpdateAsync(users);

// Bulk delete entities
await _db.BulkDeleteAsync(usersToDelete);

// Upsert (insert or update)
await _db.BulkInsertOrUpdateAsync(users);
```

### 4.4. Raw COPY — самый быстрый для PostgreSQL

```csharp
using Npgsql;

await using var connection = (NpgsqlConnection)_db.Database.GetDbConnection();
await connection.OpenAsync();

using var writer = await connection.BeginBinaryImportAsync(
    "COPY users (name, email) FROM STDIN (FORMAT BINARY)");

foreach (var user in users)
{
    await writer.StartRowAsync();
    await writer.WriteAsync(user.Name, NpgsqlDbType.Text);
    await writer.WriteAsync(user.Email, NpgsqlDbType.Text);
}

await writer.CompleteAsync();
// Performance: insert 1M rows за ~1 second
```

### 4.5. SqlBulkCopy — самый быстрый для SQL Server

```csharp
using Microsoft.Data.SqlClient;
using System.Data;

var dataTable = new DataTable();
dataTable.Columns.Add("Name", typeof(string));
dataTable.Columns.Add("Email", typeof(string));

foreach (var user in users)
{
    dataTable.Rows.Add(user.Name, user.Email);
}

using var bulkCopy = new SqlBulkCopy(connection)
{
    DestinationTableName = "Users",
    BatchSize = 10_000
};

await bulkCopy.WriteToServerAsync(dataTable);
```

### 4.6. Decision tree

```
Сколько rows?
│
├── < 100 → AddRange + SaveChanges
├── 100 - 10,000 → AddRange + SaveChanges (batched)
├── 10,000 - 100,000 → EFCore.BulkExtensions (BulkInsertAsync; лицензия — см. 4.2)
└── 100,000+ → 
    ├── PostgreSQL → COPY (Npgsql binary import)
    └── SQL Server → SqlBulkCopy
```

### 4.7. Content-hash change tracking (skip DML wholesale)

**Проблема, которую EF не решает сам.** ChangeTracker делает **per-property dirty-checking**: на каждую загруженную entity он держит снапшот original values и при `SaveChanges` сравнивает текущее состояние property-by-property. Это спасает от лишних UPDATE для **отдельных** колонок, но:

- снапшот **всё равно материализуется** для каждой загруженной row (память + CPU на сравнение);
- решение «писать или нет» принимается EF **внутри** `SaveChanges`, когда граф уже собран, навигации прицеплены, value-converters прогнаны;
- при ETL/sync ты обычно **перезаписываешь весь аггрегат** из источника (re-present), поэтому ChangeTracker честно видит «всё поменялось» даже если бизнес-смысл идентичен.

Идея content-hash: **до** записи посчитать хэш контента аггрегата **до и после** enrichment; если совпало — пропустить весь write-path для этой row целиком. Это не замена dirty-checking, а **обход решения о записи на уровне аггрегата** для идемпотентно пере-представленных строк — раньше и грубее, чем per-property сравнение EF.

> [!info] Отличие от per-property dirty-checking
> EF dirty-checking работает **внутри** `SaveChanges` и всё равно снапшотит каждую загруженную entity. Content-hash отсекает row **до** того, как она вообще попадёт в bulk-batch или в `ChangeTracker`-граф: для unchanged rows нет ни снапшота, ни DML-решения, ни round-trip.

Хэш считаем по стабильной канонической проекции (порядок полей фиксирован, без волатильных полей типа `UpdatedAt`):

```csharp
using System.IO.Hashing;   // XxHash3 — fast non-crypto hash

// Stable content fingerprint of the aggregate (exclude volatile audit fields)
private static ulong ComputeContentHash(ProductAggregate p)
{
    var hash = new XxHash3();

    hash.Append(MemoryMarshal.AsBytes(p.Sku.AsSpan()));
    hash.Append(MemoryMarshal.AsBytes(p.Name.AsSpan()));
    hash.Append(MemoryMarshal.AsBytes(stackalloc[] { p.Price }));
    hash.Append(MemoryMarshal.AsBytes(stackalloc[] { p.CategoryId }));

    foreach (var v in p.Variants.OrderBy(static x => x.Id))   // deterministic order
        hash.Append(MemoryMarshal.AsBytes(stackalloc[] { v.Id, v.Stock }));

    return hash.GetCurrentHashAsUInt64();
}
```

Pipeline: хэш существующего состояния хранится в колонке (`content_hash bigint`), сравниваем с хэшом после enrichment:

```csharp
var changed = new List<ProductAggregate>();

foreach (var incoming in source)   // 999 re-presented aggregates
{
    Enrich(incoming);   // normalize, attach computed fields

    ulong newHash = ComputeContentHash(incoming);

    if (existingHashes.TryGetValue(incoming.Sku, out ulong oldHash) && oldHash == newHash)
        continue;   // ← skip the WRITE decision entirely; never enters the DML path

    incoming.ContentHash = newHash;
    changed.Add(incoming);
}

// Only the genuinely-changed subset hits the DB
```

**Измеренный кейс (sync 999 аггрегатов):** 967 строк пропущены по совпадению хэша, в БД ушли только 32 изменённые. End-to-end примерно `991ms -> 154ms` (≈6.4x): исчезли материализация снапшотов, генерация DML и round-trip'ы для 96.7% строк.

**Комбинируем с bulk-путями в ОДНОЙ транзакции.** Из 32 changed часть — новые (INSERT через binary COPY / `SqlBulkCopy`), часть — существующие (batched UPDATE по PK). Всё под одним `BeginTransactionAsync`, чтобы sync был атомарным:

```csharp
await using var tx = await _db.Database.BeginTransactionAsync(ct);
try
{
    var (toInsert, toUpdate) = Partition(changed, existingKeys);

    // Inserts — Npgsql binary COPY (fastest path, см. 4.4)
    if (toInsert.Count > 0)
    {
        await using var writer = await connection.BeginBinaryImportAsync(
            "COPY products (sku, name, price, category_id, content_hash) FROM STDIN (FORMAT BINARY)", ct);

        foreach (var p in toInsert)
        {
            await writer.StartRowAsync(ct);
            await writer.WriteAsync(p.Sku, NpgsqlDbType.Text, ct);
            await writer.WriteAsync(p.Name, NpgsqlDbType.Text, ct);
            await writer.WriteAsync(p.Price, NpgsqlDbType.Numeric, ct);
            await writer.WriteAsync(p.CategoryId, NpgsqlDbType.Integer, ct);
            await writer.WriteAsync((long)p.ContentHash, NpgsqlDbType.Bigint, ct);
        }

        await writer.CompleteAsync(ct);
    }

    // Updates — batched ExecuteUpdate per row group (or temp-table + UPDATE...FROM for big sets)
    foreach (var p in toUpdate)
    {
        await _db.Products
            .Where(x => x.Sku == p.Sku)
            .ExecuteUpdateAsync(s => s
                .SetProperty(x => x.Name, p.Name)
                .SetProperty(x => x.Price, p.Price)
                .SetProperty(x => x.ContentHash, (long)p.ContentHash), ct);
    }

    await tx.CommitAsync(ct);
}
catch
{
    await tx.RollbackAsync(ct);
    throw;
}
```

> [!warning] COPY и ExecuteUpdate делят connection, но НЕ транзакцию автоматически
> `ExecuteUpdate` идёт через `DbContext`, binary COPY — через `NpgsqlConnection`. Чтобы оба попали в один `tx`, бери connection из `_db.Database.GetDbConnection()` (тот же, что enlistнут в транзакцию EF), а не открывай новый. Иначе COPY закоммитится отдельно и rollback его не откатит.

**Когда применять:**

- high-volume **sync / ETL**, где **большинство** строк пере-представляются без реальных изменений (идемпотентный источник: внешний каталог, CDC-feed, nightly full-refresh);
- DML — **доказанное** узкое место (профайл показывает время в INSERT/UPDATE + round-trip, а не в чтении источника);
- есть стабильная каноническая проекция контента (детерминированный порядок коллекций, исключённые волатильные поля).

**Когда НЕ применять:** малые объёмы (хэш — лишняя сложность); источник, где почти всё меняется (хэш совпадёт редко → только overhead); аггрегаты без устойчивой канонизации (false negatives → пропущенные изменения = data drift).

> [!tip] Связь с single-property no-op guard
> Это **аггрегатное обобщение** point-guard из [[optimization-patterns|Performance/Middle/optimization-patterns]] (раздел «Skip unchanged work»): там `if (entity.Status == status) return;` отсекает запись одной property; здесь один хэш отсекает запись **целого аггрегата** ещё до входа в DML-путь. Та же идея «не делай работу, результат которой уже достигнут», поднятая с поля на граф.

---

## 5. Best Practices

### 5.1. Batch large operations

```csharp
// ❌ Try to update 10M rows одной командой
await _db.LargeTable
    .Where(x => x.Status == "old")
    .ExecuteUpdateAsync(x => x.SetProperty(p => p.Status, "new"));
// Может lock'нуть всю таблицу на 30+ минут!
// Transaction log переполнится.

// ✅ Batch по 50K rows
const int batchSize = 50_000;
int totalUpdated = 0;

while (true)
{
    int updated = await _db.LargeTable
        .Where(x => x.Status == "old")
        .Take(batchSize)
        .ExecuteUpdateAsync(x => x.SetProperty(p => p.Status, "new"));
    
    if (updated == 0) break;
    totalUpdated += updated;
    
    _logger.LogInformation("Updated {Total} rows so far", totalUpdated);
    await Task.Delay(TimeSpan.FromMilliseconds(100));   // дать БД breathing room
}
```

### 5.2. Use indexed columns в WHERE

```csharp
// ❌ Slow — no index на Status
await _db.Orders
    .Where(o => o.Status == OrderStatus.Pending)
    .ExecuteUpdateAsync(...);
// Full table scan для миллионов rows

// ✅ Index на Status column
// CREATE INDEX IX_Orders_Status ON Orders(Status);
```

### 5.3. Transaction для multiple bulk operations

```csharp
using var transaction = await _db.Database.BeginTransactionAsync();
try
{
    await _db.OrderItems
        .Where(oi => oi.OrderId == orderId)
        .ExecuteDeleteAsync();
    
    await _db.Orders
        .Where(o => o.Id == orderId)
        .ExecuteDeleteAsync();
    
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

### 5.4. Logging audit trail отдельно

ExecuteUpdate не triggers SaveChanges interceptors. Логируй вручную:

```csharp
public async Task<int> BulkUpdateOrdersAsync(...)
{
    int updated = await _db.Orders
        .Where(...)
        .ExecuteUpdateAsync(...);
    
    _db.AuditLogs.Add(new AuditLog
    {
        Action = "BulkUpdate",
        Entity = "Order",
        AffectedCount = updated,
        Timestamp = DateTime.UtcNow,
        UserId = _currentUser.Id
    });
    
    await _db.SaveChangesAsync();
    return updated;
}
```

### 5.5. Verify before destructive operations

```csharp
// Always count BEFORE delete на prod
var matchCount = await _db.Users
    .Where(u => !u.IsActive && u.LastLogin < cutoff)
    .CountAsync();

if (matchCount > expectedThreshold)
{
    _logger.LogWarning("Would delete {Count} users — exceeds threshold", matchCount);
    return;   // safety check
}

await _db.Users
    .Where(u => !u.IsActive && u.LastLogin < cutoff)
    .ExecuteDeleteAsync();
```

---

## 6. Common pitfalls

### 6.1. ExecuteUpdate не trigger SaveChanges hooks

```csharp
// SaveChanges interceptor для auditing
public class AuditInterceptor : SaveChangesInterceptor
{
    public override async Task<InterceptionResult<int>> SavingChangesAsync(...)
    {
        // Log all changes
    }
}

// ExecuteUpdate не trigger этот interceptor!
await _db.Users.Where(...).ExecuteUpdateAsync(...);
// Auditing skipped
```

**Фикс**: manual logging для bulk operations или используй raw SQL triggers в DB.

### 6.2. Cascade delete без FK constraints

```csharp
// ❌ DELETE orders без сначала удалить OrderItems
await _db.Orders.Where(o => ...).ExecuteDeleteAsync();
// FK violation если no CASCADE
```

**Фикс**: configure CASCADE в migrations или delete children сначала.

### 6.3. Lock everything

```csharp
// ❌ Update 10M rows одной командой
await _db.LargeTable.ExecuteUpdateAsync(...);
// Locks таблицы на длительное время
// Other transactions block / timeout
```

**Фикс**: batch processing.

### 6.4. ExecuteUpdate вместо ExecuteUpdateAsync

```csharp
// ❌ Sync version в async context
_db.Users.Where(...).ExecuteUpdate(...);   // blocks thread
```

**Фикс**: всегда `ExecuteUpdateAsync` в async code.

### 6.5. Memory issues с AddRange миллионов

```csharp
// ❌ Memory exhausted
var allRows = ReadFromCsv(filePath);   // 10M rows!
_db.Users.AddRange(allRows);
await _db.SaveChangesAsync();
// OutOfMemoryException
```

**Фикс**: batch processing или streaming bulk insert.

### 6.6. Не нужно загружать ДЛЯ ExecuteUpdate

```csharp
// ❌ Загрузил → modify → SaveChanges
var users = await _db.Users.Where(u => ...).ToListAsync();
foreach (var u in users) u.LastLogin = DateTime.UtcNow;
await _db.SaveChangesAsync();

// ✅ ExecuteUpdate без load
await _db.Users
    .Where(u => ...)
    .ExecuteUpdateAsync(u => u.SetProperty(x => x.LastLogin, DateTime.UtcNow));
```

### 6.7. Concurrency tokens не check

```csharp
public class Product
{
    [Timestamp]
    public byte[] RowVersion { get; set; }
    public decimal Price { get; set; }
}

// ❌ ExecuteUpdate не check RowVersion
await _db.Products
    .Where(p => p.Id == id)
    .ExecuteUpdateAsync(p => p.SetProperty(x => x.Price, newPrice));
// Перезаписывает изменения, сделанные другими transactions
```

**Фикс**: для optimistic concurrency используй стандартный SaveChanges flow.

### 6.8. Bulk extensions package security

```csharp
// Проверь package: EFCore.BulkExtensions vs forks
// Some forks deprecated / unmaintained
```

Основной пакет — `EFCore.BulkExtensions` от **borisdj**, но с января 2024 он cFOSS (см. §4.2): для коммерческого использования при выручке ≥ $1M нужна платная лицензия. Свободная альтернатива с тем же API — MIT-форк **EFCore.BulkExtensions.MIT**; перед добавлением любого из них проверь publisher и активность репозитория ([[choosing-dependencies|Choosing Dependencies]]).

### 6.9. ExecuteUpdate с related entities

```csharp
// ❌ Нельзя update через navigation
await _db.Orders
    .Where(o => o.Id == orderId)
    .ExecuteUpdateAsync(o => o
        .SetProperty(x => x.Customer.Email, newEmail));   // ❌ не работает

// ✅ Update target entity directly
await _db.Customers
    .Where(c => c.Orders.Any(o => o.Id == orderId))
    .ExecuteUpdateAsync(c => c.SetProperty(x => x.Email, newEmail));
```

### 6.10. Returns affected count vs row count

```csharp
int affected = await _db.Users
    .Where(u => u.Email == "old@x.com")
    .ExecuteUpdateAsync(...);

// affected == 0 не означает "user не существует"
// Может означать: user найден, но new value == old value (no change)
// На SQL Server affected = matched rows, не actually updated
```

> [!question]- **Интервью: топ-3 ошибки с bulk operations?**
> 1) **ExecuteUpdate без batching на больших таблицах** — locks всю table на длительное время. Fix: batch по 10-50K rows с pause. 2) **Не trigger interceptors / Domain Events** — auditing / SaveChanges hooks skipped. Fix: manual logging или raw SQL triggers. 3) **AddRange миллионов rows** — memory exhausted. Fix: EFCore.BulkExtensions или native bulk copy (SqlBulkCopy / COPY). **Bonus**: forgot CASCADE → FK violation when deleting parents. **Bonus 2**: ExecuteUpdate не respects optimistic concurrency tokens.

---

## 7. Cheat sheet

```csharp
// === ExecuteUpdate (EF Core 7+) ===
await _db.Users
    .Where(u => u.IsActive)
    .ExecuteUpdateAsync(u => u
        .SetProperty(x => x.LastSeen, DateTime.UtcNow)
        .SetProperty(x => x.LoginCount, x => x.LoginCount + 1));

// === ExecuteDelete (EF Core 7+) ===
await _db.Users
    .Where(u => !u.IsActive)
    .ExecuteDeleteAsync();

// === AddRange (small batches) ===
_db.Users.AddRange(users);
await _db.SaveChangesAsync();

// === EFCore.BulkExtensions (large) ===
await _db.BulkInsertAsync(users);
await _db.BulkUpdateAsync(users);
await _db.BulkDeleteAsync(users);
await _db.BulkInsertOrUpdateAsync(users);

// === Native bulk PostgreSQL ===
using var writer = await connection.BeginBinaryImportAsync(
    "COPY users (name, email) FROM STDIN (FORMAT BINARY)");
await writer.StartRowAsync();
await writer.WriteAsync(name, NpgsqlDbType.Text);
await writer.CompleteAsync();

// === Native bulk SQL Server ===
using var bulkCopy = new SqlBulkCopy(connection)
{
    DestinationTableName = "Users",
    BatchSize = 10_000
};
await bulkCopy.WriteToServerAsync(dataTable);

// === Batched processing ===
const int batchSize = 50_000;
while (true)
{
    int updated = await _db.LargeTable
        .Where(x => x.Status == "old")
        .Take(batchSize)
        .ExecuteUpdateAsync(...);
    
    if (updated == 0) break;
    await Task.Delay(100);
}

// === Transactions для multiple bulk ===
using var transaction = await _db.Database.BeginTransactionAsync();
await _db.Children.Where(...).ExecuteDeleteAsync();
await _db.Parents.Where(...).ExecuteDeleteAsync();
await transaction.CommitAsync();
```

---

## 8. Performance comparison

```
Operation: 1,000,000 rows insert

Approach                            Time      Memory    Notes
────────────────────────────────────────────────────────────────
foreach + Add + SaveChanges         ~30 min    Low      1M transactions
foreach + Add + один SaveChanges    ~5 min     1 GB     1 transaction, 1M INSERTs
AddRange + SaveChanges              ~30 sec    1 GB     Batched INSERTs
EFCore.BulkExtensions               ~3 sec     200 MB   Wraps SqlBulkCopy/COPY
Native SqlBulkCopy / COPY           ~1 sec     50 MB    Lowest level
```

```
Operation: UPDATE 1,000,000 rows status

Approach                            Time      Memory
────────────────────────────────────────────────────
foreach + property + SaveChanges    ~3 min     500 MB
ExecuteUpdate (single SQL)          ~1 sec     ~0
EFCore.BulkExtensions BulkUpdate    ~5 sec     100 MB
```

ExecuteUpdate **выигрывает** для simple WHERE+SET scenarios.

---

## 9. Practice exercises

### 9.1. Migration script

Production database — 50M users. Нужно:
1. Mark all users без verified email as `RequiresVerification = true`
2. Set `LastNotified = NULL` для отправки notification
3. Update в один SQL statement если возможно

Ограничения: maintenance window — 30 minutes. Требуется не залочить таблицу.

### 9.2. Cleanup job

Daily background job:
1. Soft-delete все Orders старше 7 лет (compliance)
2. Hard-delete OrderItems для already-deleted Orders
3. Archive в audit table перед deletion

Используй ExecuteUpdate + ExecuteDelete + transaction.

### 9.3. Bulk import

User uploaded CSV file с 5M rows of products. Реализуй:
1. Validation (skip invalid rows, log errors)
2. Bulk insert valid rows
3. Update existing products (по SKU)
4. Return summary: inserted, updated, skipped

Цель: < 1 минуты для 5M rows.

---

## 10. Что читать дальше

1. **`EFCore/Senior/queries-performance.md`** — N+1, compiled queries
2. **`EFCore/Middle/ef-loading-strategies.md`** — loading strategies
3. **`EFCore/Middle/ef-transactions-concurrency.md`** — transactions deep
4. **`SQL/Senior/optimization.md`** — partitioning, индексы для больших таблиц

---

## 11. См. также

- [[ef-basics|EFCore/Junior/ef-basics]] — basics
- [[ef-loading-strategies|EFCore/Middle/ef-loading-strategies]] — loading
- [[ef-transactions-concurrency|EFCore/Middle/ef-transactions-concurrency]] — transactions
- [[queries-performance|EFCore/Senior/queries-performance]] — performance
- [[dapper-comparison|EFCore/Middle/dapper-comparison]] — alternatives
- [[choosing-dependencies|Choosing Dependencies]] — лицензии зависимостей (EFCore.BulkExtensions cFOSS и др.)

---

## 12. Reading list

- **Microsoft Docs — ExecuteUpdate/ExecuteDelete** — learn.microsoft.com/ef/core/saving/execute-insert-update-delete
- **EFCore.BulkExtensions** — github.com/borisdj/EFCore.BulkExtensions (cFOSS с 2024 — см. §4.2; MIT-форк: EFCore.BulkExtensions.MIT)
- **Npgsql Binary Copy** — npgsql.org/doc/copy.html
- **SqlBulkCopy** — learn.microsoft.com/dotnet/api/system.data.sqlclient.sqlbulkcopy
- **Andrew Lock — Bulk operations EF Core** — andrewlock.net
