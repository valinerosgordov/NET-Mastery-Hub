---
tags: [efcore, loading, eager, lazy, explicit, split-query, middle, n-plus-one]
level: Middle
date: 2026-08-02
---

# EF Core Loading Strategies — eager, lazy, explicit, split queries

> **Как загружать связанные данные правильно: Include vs Select vs SplitQuery vs explicit loading.** Closes пробел между Junior basics (что такое навигация) и Senior performance (cartesian explosion fixing). Frequent interview topic.

---

## 0. Как читать

После `Junior/ef-crud-queries.md` (LINQ basics, Include intro). Перед `Senior/queries-performance.md` (deep N+1, compiled queries). Здесь — practical decisions: когда что выбирать, какие trade-offs.

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Проблема — N+1 query

```csharp
// Кажется невинным — но catastrophic в production
var users = await _db.Users.ToListAsync();   // 1 query
foreach (var user in users)
{
    Console.WriteLine($"{user.Name}: {user.Orders.Count}");
    // Каждый доступ к user.Orders → отдельный SQL!
    // 100 users → 1 + 100 = 101 queries
    // 10K users → 10K+ queries → API timeout
}
```

Это **главная причина** "медленных" EF Core API. Решается выбором правильной loading strategy.

### 1.2. Четыре стратегии

```
1. Eager loading       — Include    — загрузить связанное СРАЗУ с основной query
2. Explicit loading    — Load       — загрузить связанное ПОЗЖЕ по требованию
3. Lazy loading        — virtual    — загружается АВТОМАТИЧЕСКИ при доступе (опасно)
4. Projection          — Select     — загрузить ТОЛЬКО нужные поля (часто лучше всех)
```

### 1.3. Главное правило

```
По убыванию приоритета:
1. Projection (Select) — для read-only API responses
2. Eager loading (Include) — когда нужны полные entities
3. Split queries (AsSplitQuery) — для multiple collection includes
4. Explicit loading — для conditional loading
5. Lazy loading — НИКОГДА в production (включено по умолчанию)

Никогда:
- Не полагайся на lazy loading в hot paths
- Не Include всё подряд
- Не забывай AsNoTracking для read-only
```

> [!info]- Если ты приходил из Hibernate / SQLAlchemy
> Концепция та же. Hibernate FetchType.EAGER ↔ EF Core Include. FetchType.LAZY ↔ virtual + Proxies. SQLAlchemy `joinedload()` ↔ Include, `selectinload()` ↔ AsSplitQuery, `lazyload` ↔ proxies. Главное отличие EF Core — projection (`.Select(u => new Dto(...))`) translates в SQL очень умно, часто **лучше** чем Include + manual mapping.

> [!question]- **Интервью: какие виды loading в EF Core?**
> Четыре: 1) **Eager** (`Include`) — загрузка связанных вместе с main query через JOIN. 2) **Explicit** (`Entry().Collection().LoadAsync()`) — manual load позже. 3) **Lazy** (virtual properties + Proxies) — auto-load при первом доступе, **скрывает N+1**. 4) **Projection** (`Select`) — только нужные поля, часто **самый эффективный**. **Production rule**: 1) projection для read paths, 2) Include для CRUD, 3) explicit для conditional cases, 4) **никогда lazy** в server apps. **Why no lazy**: hidden SQL queries, hard to predict performance, N+1 risk.

---

## 2. Eager Loading — Include

### 2.1. Базовое использование

```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public List<Order> Orders { get; set; } = new();   // collection navigation
    public Address? Address { get; set; }              // reference navigation
}

public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public User User { get; set; } = null!;
    public List<OrderItem> Items { get; set; } = new();
    public decimal Total { get; set; }
}

// Single Include
var users = await _db.Users
    .Include(u => u.Orders)
    .ToListAsync();

// Multiple Include
var users = await _db.Users
    .Include(u => u.Orders)
    .Include(u => u.Address)
    .ToListAsync();
```

### 2.2. ThenInclude — глубже

```csharp
// User → Orders → OrderItems → Product
var users = await _db.Users
    .Include(u => u.Orders)
        .ThenInclude(o => o.Items)
            .ThenInclude(i => i.Product)
    .ToListAsync();
```

Indent (4 spaces) — convention для readability.

### 2.3. Filtered Include (EF Core 5+)

```csharp
// Загрузить ТОЛЬКО paid orders
var users = await _db.Users
    .Include(u => u.Orders.Where(o => o.Status == OrderStatus.Paid))
    .ToListAsync();

// Сортировка внутри Include
var users = await _db.Users
    .Include(u => u.Orders.OrderByDescending(o => o.CreatedAt).Take(10))
    .ToListAsync();
```

Поддерживаемые operators в filtered Include: `Where`, `OrderBy`/`OrderByDescending`, `ThenBy`/`ThenByDescending`, `Skip`, `Take`.

### 2.4. SQL который генерируется

```csharp
var users = await _db.Users
    .Include(u => u.Orders)
    .Where(u => u.IsActive)
    .ToListAsync();
```

```sql
-- Single LEFT JOIN
SELECT u.Id, u.Name, u.IsActive, o.Id, o.UserId, o.Total
FROM Users u
LEFT JOIN Orders o ON o.UserId = u.Id
WHERE u.IsActive = 1
ORDER BY u.Id, o.Id
```

EF потом дедуплицирует rows и формирует object graph.

### 2.5. Проблема — Cartesian Explosion

```csharp
// User имеет 10 Orders + 10 Comments + 10 Reviews
var users = await _db.Users
    .Include(u => u.Orders)
    .Include(u => u.Comments)
    .Include(u => u.Reviews)
    .ToListAsync();
```

Сгенерированный SQL делает **CROSS JOIN** между всеми collections:

```sql
-- 10 Orders × 10 Comments × 10 Reviews = 1000 rows на ОДНОГО user!
-- 100 users → 100,000 rows
```

Размер result set взрывается. Memory + network bandwidth + DB load.

### 2.6. Решение — AsSplitQuery (EF Core 5+)

```csharp
var users = await _db.Users
    .Include(u => u.Orders)
    .Include(u => u.Comments)
    .Include(u => u.Reviews)
    .AsSplitQuery()
    .ToListAsync();
```

Генерирует **отдельную query на каждую collection**:

```sql
-- Query 1: Users
SELECT * FROM Users;

-- Query 2: Orders для всех users
SELECT * FROM Orders WHERE UserId IN (1, 2, 3, ...);

-- Query 3: Comments
SELECT * FROM Comments WHERE UserId IN (1, 2, 3, ...);

-- Query 4: Reviews
SELECT * FROM Reviews WHERE UserId IN (1, 2, 3, ...);
```

4 round-trips, но каждая **маленькая**. Total data передано меньше.

**Trade-off:**
- Single query: 1 round-trip, но cartesian explosion possible
- Split: N round-trips, но каждая efficient

```
Когда AsSplitQuery:
✅ Multiple collection includes (3+ collections)
✅ Большие collections (10+ items each)
✅ Производительность важнее latency

Когда single query (default):
✅ Reference navigations only (One-to-One)
✅ Small collections (1-2 items typically)
✅ Latency важнее (минимум round-trips)
```

### 2.7. Глобальный AsSplitQuery

```csharp
// Program.cs
builder.Services.AddDbContext<AppDbContext>(opt =>
{
    opt.UseSqlServer(connStr, b => b.UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery));
});
```

Все queries по default используют split. Override на конкретной query через `.AsSingleQuery()`.

> [!question]- **Интервью: что такое cartesian explosion и как fix?**
> Когда `Include` загружает несколько collections, EF Core генерирует SQL с JOIN для каждой → SQL **умножает** rows: User × Orders × Comments × Reviews. Если у user 10 Orders, 10 Comments, 10 Reviews — то 1000 rows на одного user. **Fix**: `AsSplitQuery()` — генерирует отдельную SELECT для каждой collection (EF Core 5+). Trade-off: 1 query → N queries, но каждая **меньше**. **Глобально**: `UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)` в DbContext options. **Когда не нужно**: reference navigations (One-to-One/Many-to-One) — там JOIN не множит.

---

## 3. Projection — Select (часто лучше Include)

### 3.1. Зачем projection

Для **read paths** (API responses, list views) — projection часто **лучше** Include:

```csharp
// ❌ Loading full entities + manual mapping
var users = await _db.Users
    .Include(u => u.Orders)
    .Where(u => u.IsActive)
    .ToListAsync();

var dtos = users.Select(u => new UserDto(
    u.Id,
    u.Name,
    u.Orders.Count,
    u.Orders.Sum(o => o.Total)
)).ToList();

// 1) Loaded все колонки User + все колонки Orders
// 2) Memory: 100 users × ~50 orders × ~10 fields = 50K objects + properties
// 3) Mapping in C# memory
```

```csharp
// ✅ Projection — load только нужное, mapping в SQL
var dtos = await _db.Users
    .Where(u => u.IsActive)
    .Select(u => new UserDto(
        u.Id,
        u.Name,
        u.Orders.Count,
        u.Orders.Sum(o => o.Total)
    ))
    .ToListAsync();

// 1) SQL: SELECT Id, Name, (SELECT COUNT(*) FROM Orders WHERE UserId = u.Id),
//                          (SELECT SUM(Total) FROM Orders WHERE UserId = u.Id)
//          FROM Users WHERE IsActive = 1
// 2) Memory: 100 dto objects only
// 3) Aggregation в DB
```

### 3.2. AsNoTracking — automatic для projection

```csharp
// Projection в anonymous types / DTOs автоматически no-tracking
var dtos = await _db.Users
    .Select(u => new UserDto(u.Id, u.Name))
    .ToListAsync();
// EF не tracks эти objects — нечего tracking (не entity)

// Projection в entity → tracking сохраняется!
var users = await _db.Users
    .Select(u => new User { Id = u.Id, Name = u.Name })
    .ToListAsync();
// ⚠️ Эти User entities tracked
```

### 3.3. Сложные projections

```csharp
public record OrderSummaryDto(
    int OrderId,
    string CustomerName,
    decimal Total,
    int ItemCount,
    string TopProductName);

var summaries = await _db.Orders
    .Where(o => o.Status == OrderStatus.Paid)
    .Select(o => new OrderSummaryDto(
        o.Id,
        o.User.Name,                                          // navigation projection
        o.Total,
        o.Items.Count,                                        // collection count
        o.Items.OrderByDescending(i => i.Quantity)            // nested projection
              .Select(i => i.Product.Name)
              .FirstOrDefault() ?? "N/A"
    ))
    .ToListAsync();
```

EF Core translates это в **один efficient SQL** с subqueries.

### 3.4. Когда projection лучше Include

```
✅ Use projection когда:
- Read path (API GET, list views)
- Нужны только некоторые поля
- Aggregations (Count, Sum, Avg)
- Computed fields

❌ Не использовать projection когда:
- Нужно изменить и сохранить (need tracking)
- CRUD operations
- Domain logic operations
```

### 3.5. AutoMapper / Mapster — ProjectTo

Если используешь AutoMapper:

```csharp
var dtos = await _db.Users
    .ProjectTo<UserDto>(_mapper.ConfigurationProvider)
    .ToListAsync();
// Translates AutoMapper config в SQL projection
```

Mapster аналогично:

```csharp
var dtos = await _db.Users
    .ProjectToType<UserDto>()
    .ToListAsync();
```

> [!question]- **Интервью: Include vs Select для read API?**
> Для read-only paths **Select лучше**. **Include** загружает все колонки entities + tracks их (memory + GC pressure). **Select** загружает только нужные поля + automatic no-tracking (anonymous/DTO types). Performance: Select на read-heavy endpoints **2-5x быстрее** чем Include + manual mapping. **Когда Include**: CRUD operations где нужны полные entities для modify+save. **Best practice 2024+**: 1) `_db.Users.AsNoTracking().Where(...).Select(...)` для read APIs. 2) `_db.Users.Include(...).Where(...)` для CRUD. 3) `ProjectTo` (AutoMapper) или `ProjectToType` (Mapster) если уже используешь mapper.

---

## 4. Lazy Loading — почему НЕ в production

### 4.1. Что это

```csharp
public class User
{
    public int Id { get; set; }
    public virtual List<Order> Orders { get; set; } = new();   // virtual!
}
```

Сетап в DbContext:

```csharp
builder.Services.AddDbContext<AppDbContext>(opt =>
{
    opt.UseLazyLoadingProxies()   // включает proxy generation
       .UseSqlServer(connStr);
});
```

```bash
dotnet add package Microsoft.EntityFrameworkCore.Proxies
```

Теперь:

```csharp
var user = await _db.Users.FindAsync(1);
Console.WriteLine(user.Orders.Count);
// ↑ Hidden SQL: SELECT * FROM Orders WHERE UserId = 1
```

### 4.2. Почему опасно

```csharp
// Loop через users
var users = await _db.Users.ToListAsync();
foreach (var user in users)
{
    Console.WriteLine($"{user.Name}: {user.Orders.Count}");
    // Каждый user.Orders → SQL query!
    // 1000 users = 1001 queries (N+1 hidden!)
}
```

**Симптомы lazy loading hell:**
- API endpoint slow без видимой причины
- Tests быстрые (in-memory), production падает
- DB connections exhausted
- N+1 не видно в коде

### 4.3. Когда lazy НЕ катастрофа

```
Только в нишевых случаях:
- Desktop app (WPF/WinForms) с UI binding
- Single-user scenarios
- Console scripts с известным data set
- Хорошо тестировано performance

Никогда:
- ASP.NET Core APIs
- High-traffic web apps
- Microservices
- Cloud apps с per-request DI scope
```

### 4.4. Disable lazy loading

Не включай Proxies package и не делай properties virtual. EF Core **по default НЕ имеет** lazy loading — это explicit opt-in.

Если случайно включил:

```csharp
// Disable globally
builder.Services.AddDbContext<AppDbContext>(opt =>
{
    opt.UseLazyLoadingProxies(useLazyLoadingProxies: false)
       .UseSqlServer(connStr);
});

// Or remove UseLazyLoadingProxies() call entirely
```

> [!question]- **Интервью: lazy loading в EF Core — за или против?**
> **Против** для server apps. **Минусы**: 1) **N+1 queries hidden** — нельзя увидеть в коде. 2) **Performance unpredictable** — простой `.Count` может trigger SQL. 3) **DbContext lifetime ловушки** — если DbContext disposed, lazy load throws. 4) **Tests pass** (in-memory loads instantly), production fails. 5) **Concurrent access issues** в async code. **Plus** только для desktop apps с UI binding или single-user scripts. **Best practice 2024+**: disabled (default), use explicit Include или Select для каждого read path. Если приходишь в legacy app с lazy loading — план на migration.

---

## 5. Explicit Loading — для conditional cases

### 5.1. Что это

Загрузить связанное **позже**, после получения main entity.

```csharp
var user = await _db.Users.FindAsync(1);

// Условная загрузка
if (showOrders)
{
    await _db.Entry(user)
        .Collection(u => u.Orders)
        .LoadAsync();
}

// Reference (One-to-One)
await _db.Entry(user)
    .Reference(u => u.Address)
    .LoadAsync();
```

### 5.2. С фильтрацией и aggregation

```csharp
// Load только paid orders
await _db.Entry(user)
    .Collection(u => u.Orders)
    .Query()
    .Where(o => o.Status == OrderStatus.Paid)
    .LoadAsync();

// Aggregations без loading entities
var orderCount = await _db.Entry(user)
    .Collection(u => u.Orders)
    .Query()
    .CountAsync();

var totalRevenue = await _db.Entry(user)
    .Collection(u => u.Orders)
    .Query()
    .SumAsync(o => o.Total);
```

### 5.3. Когда explicit loading

```
✅ Use cases:
- Conditional loading ("показать orders только если admin")
- Pagination of related collection
- Aggregation без loading items
- Lazy-style без proxies
- Debugging — точно знаешь когда query happens

❌ Не use cases:
- Foreach loop (N+1 risk)
- Simple eager scenarios — Include проще
- Read-only API — projection лучше
```

### 5.4. Anti-pattern — explicit в loop

```csharp
// ❌ N+1 через explicit
var users = await _db.Users.ToListAsync();
foreach (var user in users)
{
    await _db.Entry(user)
        .Collection(u => u.Orders)
        .LoadAsync();
    // N queries!
}
```

**Фикс**: используй Include или batch-load:

```csharp
// ✅ Batch-load всех Orders за один SQL
var users = await _db.Users.ToListAsync();
var userIds = users.Select(u => u.Id).ToList();

var orders = await _db.Orders
    .Where(o => userIds.Contains(o.UserId))
    .ToListAsync();

var ordersByUser = orders.GroupBy(o => o.UserId)
    .ToDictionary(g => g.Key, g => g.ToList());

foreach (var user in users)
{
    user.Orders = ordersByUser.GetValueOrDefault(user.Id) ?? new();
}
```

Но в большинстве случаев `.Include()` проще.

---

## 6. Decision tree — что выбирать

```
Задача — какие данные грузить
│
├── Read-only API response / List view
│   ├── Только некоторые поля → Select (projection)
│   ├── Aggregations нужны → Select с computed fields
│   └── Нужна полная entity → Include + AsNoTracking
│
├── CRUD operation (modify and save)
│   ├── Single entity + 1-2 reference navs → Include
│   ├── Multiple collections → Include + AsSplitQuery
│   └── Большие collections → Include + AsSplitQuery + filter
│
├── Conditional loading
│   ├── "Иногда нужны orders" → Explicit Load
│   └── Pagination of collection → Explicit Load + Query()
│
├── Aggregations над collection
│   ├── Без loading items → Explicit Load + CountAsync()/SumAsync()
│   └── С loading items → Select projection
│
└── Simple loops через related data
    ├── Pre-load всё → Include
    ├── Lazy loading включён → 🚨 STOP, fix architecture
    └── Performance critical → measure, choose between Include/Select/Split
```

---

## 7. Cheat sheet

```csharp
// === Eager loading ===
.Include(u => u.Orders)                          // collection navigation
.Include(u => u.Address)                         // reference navigation
.Include(u => u.Orders).ThenInclude(o => o.Items)  // глубже
.Include(u => u.Orders.Where(o => o.IsActive))   // filtered (EF Core 5+)
.Include(u => u.Orders.OrderBy(o => o.Date).Take(10))  // sorted + limited

// === Split queries ===
.Include(...).Include(...).AsSplitQuery()        // separate SQL queries
.AsSingleQuery()                                  // override global split

// === Projection (часто лучше Include) ===
.Select(u => new UserDto(u.Id, u.Name, u.Orders.Count))
.Select(u => new
{
    u.Id,
    OrderCount = u.Orders.Count,
    Revenue = u.Orders.Sum(o => o.Total)
})

// === No-tracking для read-only ===
.AsNoTracking()
.AsNoTrackingWithIdentityResolution()  // дедуп identical entities

// === Explicit loading ===
await _db.Entry(user).Collection(u => u.Orders).LoadAsync();
await _db.Entry(user).Reference(u => u.Address).LoadAsync();
await _db.Entry(user).Collection(u => u.Orders).Query().Where(...).LoadAsync();

// === Aggregations через explicit ===
var count = await _db.Entry(user).Collection(u => u.Orders).Query().CountAsync();
var sum = await _db.Entry(user).Collection(u => u.Orders).Query().SumAsync(o => o.Total);

// === Disable lazy loading (default!) ===
// Не используй Microsoft.EntityFrameworkCore.Proxies
// Не делай navigation properties virtual

// === Production setup ===
builder.Services.AddDbContext<AppDbContext>(opt =>
{
    opt.UseSqlServer(connStr);
    opt.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);  // global no-tracking
    // Override per query: .AsTracking() где нужен tracking
});
```

---

## 8. Practice exercises

### 8.1. Detect cartesian explosion

```csharp
public async Task<UserDetailsDto> GetUserDetailsAsync(int userId)
{
    var user = await _db.Users
        .Include(u => u.Orders)
        .Include(u => u.Comments)
        .Include(u => u.Reviews)
        .Include(u => u.LoginHistory)
        .Where(u => u.Id == userId)
        .FirstOrDefaultAsync();
    
    return MapToDto(user);
}
```

User имеет 50 orders, 100 comments, 30 reviews, 1000 login records.

**Вопросы:**
1. Сколько rows в SQL result?
2. Какой fix?
3. Альтернатива через projection?

### 8.2. Refactor to projection

```csharp
public async Task<List<OrderListDto>> GetUserOrdersAsync(int userId)
{
    var user = await _db.Users
        .Include(u => u.Orders)
            .ThenInclude(o => o.Items)
                .ThenInclude(i => i.Product)
        .FirstOrDefaultAsync(u => u.Id == userId);
    
    return user?.Orders.Select(o => new OrderListDto
    {
        OrderId = o.Id,
        Date = o.CreatedAt,
        Total = o.Total,
        ItemCount = o.Items.Count,
        TopProduct = o.Items
            .OrderByDescending(i => i.Quantity)
            .Select(i => i.Product.Name)
            .FirstOrDefault() ?? "N/A"
    }).ToList() ?? new();
}
```

Перепиши через `.Select()` — projection вместо Include + manual mapping. Compare:
- Загруженных полей (всех vs только нужных)
- Generated SQL
- Memory consumption

### 8.3. Conditional loading

Реализуй `GetUserAsync(int id, bool includeOrders, bool includeAddress)`:
- Если `includeOrders` = true — загрузить orders с paid status
- Если `includeAddress` = true — загрузить address
- Both flags false — только user

Используй conditional Include через `IQueryable<User>` building.

---

## 9. Common pitfalls

### 9.1. Include + Where в неправильном порядке

```csharp
// ❌ Where после Include — может не filter Orders
var users = await _db.Users
    .Include(u => u.Orders)
    .Where(u => u.Orders.Any(o => o.Total > 1000))
    .ToListAsync();
// Загрузит ВСЕ Orders каждого user (не только > 1000)!

// ✅ Filter в Include
var users = await _db.Users
    .Include(u => u.Orders.Where(o => o.Total > 1000))
    .Where(u => u.Orders.Any(o => o.Total > 1000))
    .ToListAsync();
```

### 9.2. Include после ToListAsync

```csharp
// ❌ Include после ToList — игнорируется!
var users = (await _db.Users.ToListAsync())
    .Include(u => u.Orders);   // Compile error: ToListAsync returns List<T>, not IQueryable
```

### 9.3. Select + Include конфликт

```csharp
// Select не нуждается в Include — Select сам определяет что грузить
var dtos = await _db.Users
    .Include(u => u.Orders)        // ← бесполезен!
    .Select(u => new UserDto(u.Id, u.Name))
    .ToListAsync();
```

### 9.4. Lazy loading в async

```csharp
// ❌ Sync lazy load в async context
public async Task<int> GetOrderCount(int userId)
{
    var user = await _db.Users.FindAsync(userId);
    return user.Orders.Count;   // SYNC SQL query inside async method!
}
```

**Фикс**: explicit или Include.

### 9.5. Forgot AsNoTracking

```csharp
// API endpoint — read-only
public async Task<List<UserDto>> GetAllAsync() =>
    await _db.Users
        .Include(u => u.Address)
        .Select(u => new UserDto(u.Id, u.Name, u.Address.City))
        .ToListAsync();
// Projection automatic no-tracking — OK

// Но если без projection:
public async Task<List<User>> GetAllAsync() =>
    await _db.Users
        .Include(u => u.Address)
        .ToListAsync();
// ⚠️ Tracking overhead для read-only path
```

**Фикс**: `.AsNoTracking()`.

### 9.6. Cartesian explosion без AsSplitQuery

```csharp
// ❌ 3 collections без split
var users = await _db.Users
    .Include(u => u.Orders)
    .Include(u => u.Comments)
    .Include(u => u.Reviews)
    .ToListAsync();
```

**Фикс**: `.AsSplitQuery()`.

### 9.7. Filtered Include с unsupported operator

```csharp
// ❌ Filtered Include поддерживает только Where/OrderBy/Skip/Take
var users = await _db.Users
    .Include(u => u.Orders.GroupBy(o => o.Status))   // ❌ GroupBy не работает
    .ToListAsync();
```

**Фикс**: используй Select вместо filtered Include для сложных aggregations.

---

## 10. Performance comparison

```
Scenario: 100 users, each с 50 orders, 100 comments, 30 reviews

Approach 1: Multiple Include без AsSplitQuery
- SQL rows: 100 × 50 × 100 × 30 = 15,000,000 rows
- Network: ~500 MB
- Memory: ~1 GB
- Time: timeout

Approach 2: Multiple Include + AsSplitQuery
- 4 separate queries
- Total rows: 100 + 5000 + 10000 + 3000 = 18,100 rows
- Network: ~5 MB
- Memory: ~50 MB
- Time: ~200ms

Approach 3: Projection через Select
- 1 query с aggregations
- Rows: 100
- Memory: ~1 MB
- Time: ~50ms

Approach 4: Lazy loading (включён)
- 1 + 100 + 100 + 100 = 301 queries (N+1)
- Memory: depends
- Time: ~3 seconds (network overhead)
```

**Lesson:** projection через `Select` ≫ split queries ≫ single query (with explosion) ≫ lazy loading.

---

## 11. Что читать дальше

1. **`EFCore/Senior/queries-performance.md`** — N+1 deep, compiled queries, profiling
2. **`EFCore/Senior/relationships.md`** — relationships configuration
3. **`EFCore/Senior/basics-tracking.md`** — change tracking deep
4. **`EFCore/Middle/dapper-comparison.md`** — когда EF, когда Dapper
5. **`AspNetCore/Middle/object-mapping.md`** — Mapperly/AutoMapper для DTO mapping

---

## 12. См. также

- [[ef-basics|EFCore/Junior/ef-basics]] — DbContext basics
- [[ef-crud-queries|EFCore/Junior/ef-crud-queries]] — LINQ basics
- [[queries-performance|EFCore/Senior/queries-performance]] — performance deep
- [[relationships|EFCore/Senior/relationships]] — entity relationships
- [[dapper-comparison|EFCore/Middle/dapper-comparison]] — alternatives
- [[object-mapping|AspNetCore/Middle/object-mapping]] — DTO mapping
- [[lazy-eager-loading|Lazy vs Eager Loading]] — общие lazy/eager trade-offs за пределами EF: `Lazy<T>`, DI, cache warming

---

## 13. Reading list

- **Microsoft Docs — Loading Related Data** — learn.microsoft.com/ef/core/querying/related-data/
- **Microsoft Docs — Single vs Split Queries** — learn.microsoft.com/ef/core/querying/single-split-queries
- **Microsoft Docs — Filtered Include** — learn.microsoft.com/ef/core/querying/related-data/eager
- **Andrew Lock — EF Core Performance** — andrewlock.net
- **Jon Smith — Entity Framework Core in Action** (book, chapters on querying)
- **EF Core source** — github.com/dotnet/efcore
