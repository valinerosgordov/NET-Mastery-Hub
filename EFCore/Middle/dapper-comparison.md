---
tags: [efcore, dapper, micro-orm, raw-sql, performance, middle, senior]
level: Middle
date: 2026-04-30
---

# Dapper vs EF Core — когда что

> **Dapper — micro-ORM**: тонкий wrapper над ADO.NET, raw SQL + mapping. Часто требуют в вакансиях как альтернативу EF Core. Closes пробел "знаю EF, не знаю когда переходить на Dapper".

---

## Что это, зачем и когда

### Эволюция доступа к данным в .NET

```
ADO.NET (~2002)              # raw SqlConnection / SqlCommand — verbose
  ↓
LINQ to SQL (~2008)           # первый ORM от Microsoft
  ↓
Entity Framework (~2008)      # full ORM
  ↓
Dapper (~2011)                # Stack Overflow создали — fast, simple
  ↓
EF Core (~2016)               # cross-platform, performance улучшен
  ↓
Dapper.AOT (~2024)            # source generators для AOT
```

**Сегодня в .NET ecosystem:**
- **EF Core** — full ORM (90% projects)
- **Dapper** — micro-ORM (specific scenarios или alongside EF)
- **Raw ADO.NET** — only когда нужен maximum control

### Что такое Dapper

Тонкий wrapper над ADO.NET — пишешь **сам SQL**, Dapper делает **mapping** результатов в objects.

```csharp
// Dapper
var users = await connection.QueryAsync<User>(
    "SELECT Id, Name, Email FROM Users WHERE IsActive = @IsActive",
    new { IsActive = true });

// EF Core
var users = await db.Users
    .Where(u => u.IsActive)
    .ToListAsync();
```

---

## 1. Сравнение по аспектам

### Performance

```
Benchmark: SELECT 500 rows
  Raw ADO.NET:    1.0x  (baseline, ~5ms)
  Dapper:         1.0x  (~5ms — практически identical to raw)
  Dapper.AOT:     1.0x  (с source generators)
  EF Core:        1.5x  (~7-8ms — overhead change tracking, query translation)
  EF Core (no-tracking): 1.2x (~6ms)
```

Dapper ~30-50% быстрее EF в read scenarios. Для **многих writes** разница меньше.

### Простота

| | EF Core | Dapper |
|--|---------|--------|
| Setup | DbContext + migrations | Connection string only |
| Query | LINQ (compiled) | Raw SQL string |
| Schema | Code-first | Manual / DB-first |
| Migrations | EF migrations | Custom tool (DbUp, Flyway) |

### Что делает каждый

| Feature | EF Core | Dapper |
|---------|---------|--------|
| Object mapping | ✅ | ✅ |
| LINQ queries | ✅ | ❌ (raw SQL) |
| Change tracking | ✅ | ❌ |
| Migrations | ✅ | ❌ |
| Lazy loading | ✅ | ❌ |
| Eager loading | ✅ Include | ❌ (вручную) |
| Transactions | ✅ | ✅ (через connection) |
| Concurrency control | ✅ Optimistic | ❌ (ручное) |
| Batching | ✅ | Multiple queries |
| Stored procedures | ✅ (limited) | ✅ (excellent) |
| Performance | Good | Excellent |
| Learning curve | Medium-High | Low |

---

## 2. Dapper basics

### Setup

```bash
dotnet add package Dapper
dotnet add package Microsoft.Data.SqlClient    # для SQL Server
# или Npgsql для PostgreSQL
```

### Connection

```csharp
using Dapper;
using Microsoft.Data.SqlClient;

string connStr = "Server=localhost;Database=mydb;Trusted_Connection=true;";

using var connection = new SqlConnection(connStr);
// connection.Open() не нужен — Dapper сам managing
```

### Query — single result

```csharp
var user = await connection.QueryFirstOrDefaultAsync<User>(
    "SELECT * FROM Users WHERE Id = @Id",
    new { Id = 1 });

// Если ровно один — QuerySingleAsync (throws если 0 или >1)
var user = await connection.QuerySingleAsync<User>(
    "SELECT * FROM Users WHERE Id = @Id",
    new { Id = 1 });

// QuerySingleOrDefault — допускает 0 или 1
```

### Query — multiple

```csharp
var users = await connection.QueryAsync<User>(
    "SELECT Id, Name, Email FROM Users WHERE IsActive = @IsActive",
    new { IsActive = true });

// Returns IEnumerable<User>, обычно `.ToList()`
```

### Execute (INSERT / UPDATE / DELETE)

```csharp
int rowsAffected = await connection.ExecuteAsync(
    "UPDATE Users SET Name = @Name WHERE Id = @Id",
    new { Name = "John", Id = 1 });
```

### Insert + return ID

```csharp
// SQL Server
var newId = await connection.ExecuteScalarAsync<int>(@"
    INSERT INTO Users (Name, Email) 
    VALUES (@Name, @Email);
    SELECT CAST(SCOPE_IDENTITY() AS INT);",
    new { Name = "John", Email = "j@x.com" });

// PostgreSQL
var newId = await connection.ExecuteScalarAsync<int>(@"
    INSERT INTO users (name, email) 
    VALUES (@Name, @Email)
    RETURNING id;",
    new { Name = "John", Email = "j@x.com" });
```

### Batch insert

```csharp
var users = new[]
{
    new { Name = "Alice", Email = "a@x.com" },
    new { Name = "Bob", Email = "b@x.com" }
};

await connection.ExecuteAsync(
    "INSERT INTO Users (Name, Email) VALUES (@Name, @Email)",
    users);
// Dapper выполнит INSERT для каждого элемента
```

### Multi-mapping (JOIN)

```csharp
var orders = await connection.QueryAsync<Order, User, Order>(
    @"SELECT o.*, u.* 
      FROM Orders o 
      JOIN Users u ON u.Id = o.UserId",
    (order, user) =>
    {
        order.User = user;
        return order;
    },
    splitOn: "Id");  // на каком column splittить мапинг
```

### Multiple result sets

```csharp
var sql = @"
    SELECT * FROM Users WHERE Id = @Id;
    SELECT * FROM Orders WHERE UserId = @Id;";

using var reader = await connection.QueryMultipleAsync(sql, new { Id = 1 });
var user = await reader.ReadFirstOrDefaultAsync<User>();
var orders = (await reader.ReadAsync<Order>()).ToList();
```

### Transactions

```csharp
using var connection = new SqlConnection(connStr);
await connection.OpenAsync();
using var transaction = await connection.BeginTransactionAsync();

try
{
    await connection.ExecuteAsync(
        "INSERT INTO ...",
        new { ... },
        transaction: transaction);
    
    await connection.ExecuteAsync(
        "UPDATE ...",
        new { ... },
        transaction: transaction);
    
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

### Stored procedures

```csharp
var users = await connection.QueryAsync<User>(
    "GetActiveUsers",                       // proc name
    new { CompanyId = 1 },                   // parameters
    commandType: CommandType.StoredProcedure);
```

---

## 3. Когда Dapper > EF Core

### 1. Read-heavy scenarios

```csharp
// Dashboard queries — много reads, no need for tracking
public class DashboardService
{
    private readonly IDbConnection _conn;
    
    public async Task<DashboardData> GetData(int userId)
    {
        var stats = await _conn.QueryFirstAsync<Stats>(@"
            SELECT 
                COUNT(*) as TotalOrders,
                SUM(Total) as Revenue,
                AVG(Total) as AvgOrder
            FROM Orders 
            WHERE UserId = @UserId AND CreatedAt > @SinceDate",
            new { UserId = userId, SinceDate = DateTime.UtcNow.AddDays(-30) });
        
        return new DashboardData { Stats = stats };
    }
}
```

### 2. Complex SQL queries

```csharp
// Recursive CTE, window functions, full-text search
const string sql = @"
WITH OrderRanks AS (
    SELECT 
        o.Id, o.UserId, o.Total,
        ROW_NUMBER() OVER (PARTITION BY o.UserId ORDER BY o.Total DESC) as Rank
    FROM Orders o
)
SELECT TOP 10 u.Name, o.Total, o.Rank
FROM OrderRanks o
JOIN Users u ON u.Id = o.UserId
WHERE o.Rank <= 3
ORDER BY o.Total DESC";

var results = await connection.QueryAsync<UserOrderRank>(sql);
```

EF Core LINQ для такого — verbose и не всегда правильно translate.

### 3. Stored procedures

```csharp
// Complex business logic в SP
var result = await connection.QueryAsync<ReportRow>(
    "GenerateMonthlyReport",
    new 
    { 
        Year = 2024, 
        Month = 1,
        IncludeArchived = false
    },
    commandType: CommandType.StoredProcedure);
```

### 4. Hot paths требующих maximum performance

```csharp
// API endpoint вызывается 10K+ RPS
public async Task<UserDto> GetUserAsync(int id)
{
    return await _conn.QuerySingleOrDefaultAsync<UserDto>(
        "SELECT Id, Name, Email FROM Users WHERE Id = @Id",
        new { Id = id });
}
// 30-50% быстрее чем EF в этом scenario
```

### 5. Legacy DB / DBA-controlled schemas

Когда DB schema control НЕ у разработчиков (DBA team), EF migrations не применимы.

### 6. Reporting / data warehousing

```csharp
// Сложные analytics queries — SQL естественнее чем LINQ
const string sql = @"
SELECT 
    DATE_TRUNC('month', created_at) as Month,
    category,
    SUM(amount) as Total,
    COUNT(*) as Count,
    AVG(amount) as Average
FROM orders
WHERE created_at >= @StartDate
GROUP BY DATE_TRUNC('month', created_at), category
ORDER BY Month, category";

var report = await connection.QueryAsync<ReportRow>(sql, new { StartDate });
```

---

## 4. Когда EF Core > Dapper

### 1. CRUD-heavy applications

```csharp
// EF: change tracking автоматически генерит SQL
var user = await db.Users.FindAsync(1);
user.Name = "Updated";
user.Email = "new@x.com";
await db.SaveChangesAsync();
// Auto: UPDATE Users SET Name=@Name, Email=@Email WHERE Id=1

// Dapper — manual
var user = await conn.QueryFirstAsync<User>("SELECT * FROM Users WHERE Id=1");
user.Name = "Updated";
user.Email = "new@x.com";
await conn.ExecuteAsync(
    "UPDATE Users SET Name=@Name, Email=@Email WHERE Id=@Id", user);
```

### 2. Domain-driven design

```csharp
// EF: aggregates, domain events, навигационные свойства
public class Order
{
    public List<OrderItem> Items { get; }
    
    public void AddItem(Product product, int qty)
    {
        Items.Add(new OrderItem(product, qty));
        AddDomainEvent(new ItemAddedEvent(...));
    }
}

await db.SaveChangesAsync();  // auto-saves entire aggregate
```

### 3. Migrations нужны

```bash
dotnet ef migrations add AddUserTable
dotnet ef database update
```

Dapper не имеет встроенного migrations.

### 4. Проект small-to-medium

EF Core — стандарт. Меньше кода. Больше people знают. Faster development.

---

## 5. Гибрид — лучшее обоих

Использовать обa в одном проекте — best practice 2026.

```csharp
public class UserService
{
    private readonly AppDbContext _db;            // EF для CRUD
    private readonly IDbConnection _conn;          // Dapper для read-heavy
    
    public UserService(AppDbContext db, IDbConnection conn)
    {
        _db = db;
        _conn = conn;
    }
    
    // CRUD — EF (change tracking, validation)
    public async Task<User> CreateUserAsync(CreateUserCommand cmd)
    {
        var user = new User(cmd.Name, cmd.Email);
        _db.Users.Add(user);
        await _db.SaveChangesAsync();
        return user;
    }
    
    // Complex read — Dapper (performance)
    public async Task<UserDashboardData> GetDashboardAsync(int userId)
    {
        const string sql = @"
            SELECT 
                u.Name,
                COUNT(o.Id) as OrderCount,
                COALESCE(SUM(o.Total), 0) as Revenue
            FROM Users u
            LEFT JOIN Orders o ON o.UserId = u.Id
            WHERE u.Id = @UserId
            GROUP BY u.Name";
        
        return await _conn.QuerySingleAsync<UserDashboardData>(
            sql, new { UserId = userId });
    }
}
```

### DI setup

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(connStr));

builder.Services.AddScoped<IDbConnection>(sp =>
    new SqlConnection(connStr));

// Альтернатива — same connection
builder.Services.AddScoped<IDbConnection>(sp =>
{
    var ctx = sp.GetRequiredService<AppDbContext>();
    return ctx.Database.GetDbConnection();
});
```

### Transaction across both

```csharp
using var transaction = await _db.Database.BeginTransactionAsync();
try
{
    // EF
    _db.Users.Add(new User(...));
    await _db.SaveChangesAsync();
    
    // Dapper в той же transaction
    var conn = _db.Database.GetDbConnection();
    var dbTransaction = transaction.GetDbTransaction();
    
    await conn.ExecuteAsync(
        "INSERT INTO AuditLog (...) VALUES (...)",
        new { ... },
        transaction: dbTransaction);
    
    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

---

## 6. Dapper extensions / helpers

### Dapper.Contrib — basic CRUD

```bash
dotnet add package Dapper.Contrib
```

```csharp
// CRUD без писать SQL
var user = new User { Name = "John" };

await connection.InsertAsync(user);                  // INSERT
var fetched = await connection.GetAsync<User>(1);     // SELECT WHERE Id=1
var allUsers = await connection.GetAllAsync<User>(); // SELECT *
await connection.UpdateAsync(user);                   // UPDATE
await connection.DeleteAsync(user);                    // DELETE
```

С атрибутами:
```csharp
[Table("users")]
public class User
{
    [Key]
    public int Id { get; set; }
    public string Name { get; set; }
    [Computed]
    public DateTime CreatedAt { get; set; }   // не INSERT/UPDATE
}
```

### Dapper.AOT (.NET 8+) — source generators

```bash
dotnet add package Dapper.AOT
```

```csharp
[DapperAot]   // включить AOT
public class UserRepository
{
    public async Task<User> GetById(IDbConnection conn, int id) =>
        await conn.QuerySingleAsync<User>(
            "SELECT * FROM Users WHERE Id = @id",
            new { id });
}
```

Source generator создаст strongly-typed code без runtime reflection — AOT-friendly, faster.

См. [[source-generators|Source Generators]].

### SqlKata — query builder

```bash
dotnet add package SqlKata.Execution
```

```csharp
var query = new Query("Users")
    .Where("IsActive", true)
    .Where("Age", ">", 18)
    .OrderBy("Name");

var users = await query.GetAsync<User>();
```

Альтернатива raw SQL — typed query builder с Dapper под капотом.

---

## 7. Common patterns

### Pattern 1: Repository с Dapper

```csharp
public interface IUserRepository
{
    Task<User?> GetByIdAsync(int id);
    Task<IEnumerable<User>> GetAllAsync();
    Task<int> CreateAsync(User user);
}

public class UserRepository : IUserRepository
{
    private readonly IDbConnection _conn;
    
    public UserRepository(IDbConnection conn) => _conn = conn;
    
    public async Task<User?> GetByIdAsync(int id) =>
        await _conn.QueryFirstOrDefaultAsync<User>(
            "SELECT * FROM Users WHERE Id = @Id",
            new { Id = id });
    
    public async Task<IEnumerable<User>> GetAllAsync() =>
        await _conn.QueryAsync<User>("SELECT * FROM Users");
    
    public async Task<int> CreateAsync(User user) =>
        await _conn.ExecuteScalarAsync<int>(@"
            INSERT INTO Users (Name, Email) 
            VALUES (@Name, @Email);
            SELECT CAST(SCOPE_IDENTITY() AS INT);",
            user);
}
```

### Pattern 2: Async streaming для больших results

```csharp
// Не загружает всё в память
await foreach (var user in connection.QueryUnbufferedAsync<User>(
    "SELECT * FROM HugeTable"))
{
    Process(user);
}
```

### Pattern 3: Dynamic queries

```csharp
// Sometimes нужно строить query dynamically
var sb = new StringBuilder("SELECT * FROM Users WHERE 1=1");
var parameters = new DynamicParameters();

if (!string.IsNullOrEmpty(name))
{
    sb.Append(" AND Name LIKE @Name");
    parameters.Add("Name", $"%{name}%");
}
if (minAge.HasValue)
{
    sb.Append(" AND Age >= @MinAge");
    parameters.Add("MinAge", minAge.Value);
}

var users = await connection.QueryAsync<User>(sb.ToString(), parameters);
```

### Pattern 4: Pagination

```csharp
public async Task<(IEnumerable<User> Items, int Total)> GetPagedAsync(int page, int size)
{
    const string sql = @"
        SELECT * FROM Users 
        ORDER BY Id 
        OFFSET @Offset ROWS FETCH NEXT @Size ROWS ONLY;
        
        SELECT COUNT(*) FROM Users;";
    
    using var reader = await _conn.QueryMultipleAsync(sql, new 
    { 
        Offset = (page - 1) * size, 
        Size = size 
    });
    
    var items = await reader.ReadAsync<User>();
    var total = await reader.ReadSingleAsync<int>();
    
    return (items, total);
}
```

### Pattern 5: Bulk operations через TVP / arrays

```csharp
// SQL Server — Table-Valued Parameter
var ids = new DataTable();
ids.Columns.Add("Id", typeof(int));
foreach (var id in idList) ids.Rows.Add(id);

var users = await connection.QueryAsync<User>(@"
    SELECT * FROM Users u
    INNER JOIN @Ids i ON i.Id = u.Id",
    new { Ids = ids.AsTableValuedParameter("dbo.IntList") });

// PostgreSQL — array
var users = await connection.QueryAsync<User>(@"
    SELECT * FROM users WHERE id = ANY(@Ids)",
    new { Ids = idList.ToArray() });
```

---

## 8. Common Pitfalls

### 1. SQL injection

```csharp
// ❌ String concatenation — SQL injection!
var name = userInput;
await conn.QueryAsync<User>($"SELECT * FROM Users WHERE Name = '{name}'");
// Если name = "'; DROP TABLE Users; --" — катастрофа

// ✅ Parameters
await conn.QueryAsync<User>(
    "SELECT * FROM Users WHERE Name = @Name",
    new { Name = userInput });
```

Dapper всегда parameterized — безопасно.

### 2. Connection per request — leak

```csharp
// ❌ Connection не закрывается при exception
public async Task<User> Get(int id)
{
    var conn = new SqlConnection(connStr);
    return await conn.QueryFirstAsync<User>(...);
}  // ⚠️ leak!

// ✅ using
public async Task<User> Get(int id)
{
    using var conn = new SqlConnection(connStr);
    return await conn.QueryFirstAsync<User>(...);
}
```

### 3. Multiple async без transaction

```csharp
// ❌ Half-done state если 2nd query fails
await conn.ExecuteAsync("INSERT INTO Users ...");
await conn.ExecuteAsync("INSERT INTO Audit ...");  // crash → user без audit

// ✅ Transaction
using var transaction = await conn.BeginTransactionAsync();
await conn.ExecuteAsync("INSERT INTO Users ...", transaction: transaction);
await conn.ExecuteAsync("INSERT INTO Audit ...", transaction: transaction);
await transaction.CommitAsync();
```

### 4. Mapping pitfalls

```csharp
// Class
public class User
{
    public int Id { get; set; }
    public string FullName { get; set; }
}

// SQL
SELECT Id, Name FROM Users   -- ⚠️ FullName не mapped!

// ✅ Alias
SELECT Id, Name AS FullName FROM Users
```

### 5. Performance: connection не reused

```csharp
// ❌ Каждый query — новое подключение
foreach (var id in ids)
{
    using var conn = new SqlConnection(connStr);  // открыть/закрыть!
    var user = await conn.QueryFirstAsync<User>("SELECT * FROM Users WHERE Id=@Id", new { Id = id });
}

// ✅ Один connection, или batch query
using var conn = new SqlConnection(connStr);
var users = await conn.QueryAsync<User>("SELECT * FROM Users WHERE Id IN @Ids", new { Ids = ids });
```

Connection pool управляется ADO.NET, но 1000 mini-connections — overhead.

### 6. EF Core change tracking + Dapper update

```csharp
// User loaded by EF
var user = await db.Users.FindAsync(1);

// Изменили через Dapper напрямую
await conn.ExecuteAsync("UPDATE Users SET Name='New' WHERE Id=1");

user.Name;  // ⚠️ старое значение! EF tracking не знает

// ✅ Reload
await db.Entry(user).ReloadAsync();
```

### 7. Type mismatch

```csharp
// ❌ Если в БД Id BIGINT, а в C# int
public class User
{
    public int Id { get; set; }  // ⚠️ overflow возможен!
}

// ✅ Match types
public long Id { get; set; }
```

### 8. Null mapping

```csharp
public class User
{
    public string MiddleName { get; set; }  // ⚠️ если в БД NULL — ?
}

// ✅
public string? MiddleName { get; set; }
```

### 9. Не disposing reader

```csharp
// ❌
var reader = await conn.QueryMultipleAsync(sql);
var first = await reader.ReadAsync<T>();
// reader не disposed!

// ✅
using var reader = await conn.QueryMultipleAsync(sql);
```

### 10. Не использовать async overloads

```csharp
// ❌ Sync — блокирует thread
var users = conn.Query<User>("...");

// ✅ Async
var users = await conn.QueryAsync<User>("...");
```

См. [[async-threading|Async и Threading]].

---

## 9. Best Practices

### Dapper

- **Always use parameters** — никогда string concatenation
- **using для connection** — auto-close
- **Transactions для multiple operations**
- **`QuerySingleAsync` если ровно один** — fail fast
- **`QueryFirstOrDefaultAsync` если 0 или 1**
- **Connection scopes** — короткие, не держать долго

### Гибрид EF + Dapper

- **EF для writes / domain logic** — change tracking valuable
- **Dapper для complex reads / reports** — performance + flexibility
- **Same DbContext.Database.GetDbConnection()** для shared transaction
- **Repository abstraction** — клиенты не знают какой ORM

### Performance

- **Connection pooling** уже включён в ADO.NET
- **Compiled queries** в EF Core если повторяется
- **`AsNoTracking()`** в EF для read-only
- **Dapper для hot paths** > 1000 RPS
- **Profile** перед оптимизацией!

### Maintenance

- **Centralize SQL** в repository / handler — не разбрасывай
- **DbUp / Flyway / Liquibase** для migrations если Dapper-only
- **Stored procs** — версионируй в git
- **Test queries** — integration tests с реальной DB

См. [[integration-testing|Integration Testing]].

---

## 10. Decision tree

```
Что нужно?
│
├── Simple CRUD app, small/medium?
│   → EF Core (default)
│
├── Domain-driven design?
│   → EF Core (aggregates, change tracking)
│
├── Read-heavy, complex queries?
│   ├── EF не справляется с translation → Dapper
│   ├── Нужна performance → Dapper
│   └── Иначе → EF Core с raw SQL fallback
│
├── DB-first, DBA-controlled?
│   → Dapper
│
├── Stored procs heavy?
│   → Dapper
│
├── Reporting / analytics?
│   → Dapper
│
├── Migrations нужны?
│   → EF Core (или DbUp standalone)
│
└── Best of both?
    → Hybrid: EF для writes, Dapper для reads
```

---

## 11. Cheat sheet

| Сценарий | Solution |
|----------|----------|
| Simple CRUD | EF Core |
| Hot-path read | Dapper |
| Complex JOIN/CTE | Dapper |
| Domain logic | EF Core |
| Migrations | EF Core или DbUp |
| Stored procs | Dapper |
| Bulk insert | EF Core с `AddRange` или Dapper batch |
| Real-time dashboard | Dapper |
| Reporting | Dapper |
| Audit log | Either |
| Multi-tenant | EF Core (Global Query Filters) |
| Read replicas | Dapper (легко direct) |

---

## См. также

- [[basics-tracking|EF Core Basics]] — что такое DbContext
- [[queries-performance|EF Queries Performance]] — N+1, AsNoTracking
- [[ef-patterns|EF Patterns]] — Repository, UoW
- [[sql-basics|SQL Basics]] — JOINs, transactions
- [[indexes-deep|Indexes Deep]] — production performance
- [[source-generators|Source Generators]] — Dapper.AOT
- [[architecture-patterns|Architecture Patterns]]

## Reading list

- **Dapper documentation** — github.com/DapperLib/Dapper
- **Dapper.AOT** — github.com/DapperLib/DapperAOT
- **Dapper.Contrib** — github.com/DapperLib/Dapper.Contrib
- **SqlKata** — sqlkata.com (query builder)
- **Stack Overflow blog — Dapper origins** — stackoverflow.blog
- **Microsoft Docs — EF Core vs Dapper** — learn.microsoft.com/ef/core
- **Tim Corey — Dapper tutorial** — youtube.com (recommended visual)
- **Andrew Lock — EF Core articles** — andrewlock.net
