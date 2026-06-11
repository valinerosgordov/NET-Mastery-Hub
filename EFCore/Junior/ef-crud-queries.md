---
tags: [efcore, queries, junior, linq, crud, filtering]
level: Junior
date: 2026-05-10
---

# EF Core Queries — LINQ basics, фильтры, сортировка, projection

> **Where, Select, OrderBy, FirstOrDefault, joins basics, includes, aggregations.** Practical querying для daily work. Companion к `ef-basics.md`.

---

## 0. Как читать

После `ef-basics.md` (DbContext setup + базовый CRUD). Здесь — углубление в queries: фильтрация, сортировка, joins, projection. Перед `Senior/queries-performance.md`.

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. LINQ + EF Core — overview

### 1.1. Как это работает

```csharp
// Ты пишешь LINQ query
var users = await _db.Users
    .Where(u => u.Age >= 18 && u.IsActive)
    .OrderBy(u => u.Name)
    .ToListAsync();

// EF Core преобразует в SQL:
// SELECT * FROM Users
// WHERE Age >= 18 AND IsActive = 1
// ORDER BY Name
// (выполняется в БД, возвращает результат)
```

LINQ-методы сразу не выполняются — они **строят дерево** запроса. SQL генерируется и отправляется в БД при `ToListAsync()` / `FirstOrDefaultAsync()` / etc.

### 1.2. `IQueryable<T>` vs `IEnumerable<T>`

```csharp
// IQueryable<T> — query, ещё не выполнился
IQueryable<User> query = _db.Users.Where(u => u.IsActive);
// SQL ещё не отправлен!

// Добавляешь больше операций → query growing
query = query.OrderBy(u => u.Name);
// Всё ещё нет SQL

// Только когда вызываешь terminal operation — query выполняется
List<User> result = await query.ToListAsync();
// ВОТ ТУТ SQL отправляется
```

```csharp
// ❌ ToList() рано → дальнейший Where выполняется в C#!
var users = await _db.Users
    .ToListAsync();   // ❌ загрузил ВСЕ users в память
var adults = users.Where(u => u.Age >= 18).ToList();   // фильтрация в C#

// ✅ Where перед ToList → фильтр в SQL
var adults = await _db.Users
    .Where(u => u.Age >= 18)
    .ToListAsync();   // только взрослые загружены
```

### 1.3. Deferred execution

```csharp
var query = _db.Users.Where(u => u.Age > 18);

// Можно добавлять условия
if (filterByActive) query = query.Where(u => u.IsActive);
if (sortByName) query = query.OrderBy(u => u.Name);

// SQL выполнится только сейчас
var result = await query.ToListAsync();
```

Composable queries — собираешь до execution.

> [!info]- Если ты знаешь Hibernate / SQLAlchemy / Sequelize
> Концепция `IQueryable<T>` похожа на: Hibernate Criteria API, SQLAlchemy QuerySet, Sequelize.findAll options. Composable, lazy. Преимущество EF Core — type-safe LINQ, IDE проверяет на compile-time.

> [!question]- Интервью: чем `IQueryable<T>` отличается от `IEnumerable<T>` в EF Core?
> **`IQueryable<T>`** — query expression tree, **не выполнен**. Дальнейшие LINQ операции добавляют узлы в tree. Когда вызываешь `ToListAsync()` / `FirstOrDefaultAsync()` — EF Core преобразует tree в SQL. **`IEnumerable<T>`** — already-loaded data в памяти. Дальнейшие LINQ операции выполняются в C#. **Discovery**: после `.ToList()` / `.AsEnumerable()` query выполняется, дальнейшие фильтры — in-memory. **Best practice**: фильтрация и сортировка в `IQueryable<T>` (передаются в SQL). `.ToList()` только когда готов получить результат.

---

## 2. Filtering — Where

### 2.1. Простые условия

```csharp
// Equality
var alice = await _db.Users.Where(u => u.Name == "Alice").ToListAsync();

// Inequality
var nonAdmins = await _db.Users.Where(u => u.Role != "Admin").ToListAsync();

// Comparison
var adults = await _db.Users.Where(u => u.Age >= 18).ToListAsync();

// Range
var teens = await _db.Users.Where(u => u.Age >= 13 && u.Age <= 19).ToListAsync();

// Multiple conditions
var activeAdults = await _db.Users
    .Where(u => u.IsActive && u.Age >= 18)
    .ToListAsync();

// OR
var criticalUsers = await _db.Users
    .Where(u => u.IsAdmin || u.IsModerator)
    .ToListAsync();
```

### 2.2. String operations

```csharp
// StartsWith / EndsWith / Contains
var alphaUsers = await _db.Users.Where(u => u.Name.StartsWith("A")).ToListAsync();
var gmailUsers = await _db.Users.Where(u => u.Email.EndsWith("@gmail.com")).ToListAsync();
var searchResults = await _db.Users.Where(u => u.Name.Contains("ohn")).ToListAsync();

// Case insensitive (SQL Server case insensitive по default; PostgreSQL — case sensitive)
var users = await _db.Users
    .Where(u => u.Name.ToLower() == "alice")
    .ToListAsync();
// Или EF.Functions.Like для wildcard
var likeUsers = await _db.Users
    .Where(u => EF.Functions.Like(u.Name, "%ali%"))
    .ToListAsync();

// Length
var longNames = await _db.Users
    .Where(u => u.Name.Length > 20)
    .ToListAsync();
```

### 2.3. Date filters

```csharp
// Recent users (last 7 days)
var weekAgo = DateTime.UtcNow.AddDays(-7);
var recent = await _db.Users
    .Where(u => u.CreatedAt > weekAgo)
    .ToListAsync();

// By year
var users2024 = await _db.Users
    .Where(u => u.CreatedAt.Year == 2024)
    .ToListAsync();

// Date components
var januaryUsers = await _db.Users
    .Where(u => u.CreatedAt.Month == 1)
    .ToListAsync();
```

### 2.4. Null checks

```csharp
// Where Email is NULL
var noEmail = await _db.Users
    .Where(u => u.Email == null)
    .ToListAsync();

// Where Email is NOT NULL
var withEmail = await _db.Users
    .Where(u => u.Email != null)
    .ToListAsync();

// Null-safe сравнение
var users = await _db.Users
    .Where(u => u.PhoneNumber != null && u.PhoneNumber.Length > 0)
    .ToListAsync();
```

### 2.5. List/Array IN

```csharp
// "WHERE Id IN (1, 2, 3)"
var ids = new[] { 1, 2, 3 };
var users = await _db.Users
    .Where(u => ids.Contains(u.Id))
    .ToListAsync();

// "WHERE Name IN ('Alice', 'Bob')"
var names = new List<string> { "Alice", "Bob" };
var matched = await _db.Users
    .Where(u => names.Contains(u.Name))
    .ToListAsync();
```

### 2.6. Multiple Where — AND chain

```csharp
// Эти два эквивалентны
var query1 = _db.Users
    .Where(u => u.IsActive)
    .Where(u => u.Age >= 18)
    .Where(u => u.Email != null);

var query2 = _db.Users
    .Where(u => u.IsActive && u.Age >= 18 && u.Email != null);

// EF Core генерирует одинаковый SQL
```

### 2.7. Conditional filtering

```csharp
public async Task<List<User>> SearchAsync(
    string? name = null,
    int? minAge = null,
    bool? activeOnly = null)
{
    var query = _db.Users.AsQueryable();
    
    if (!string.IsNullOrEmpty(name))
        query = query.Where(u => u.Name.Contains(name));
    
    if (minAge.HasValue)
        query = query.Where(u => u.Age >= minAge.Value);
    
    if (activeOnly == true)
        query = query.Where(u => u.IsActive);
    
    return await query.ToListAsync();
}

// EF Core строит query только с применёнными фильтрами
```

> [!question]- Интервью: как фильтровать по optional параметрам?
> Composable queries — building через if statements: `var query = _db.Users.AsQueryable(); if (filter1) query = query.Where(...); ... return query.ToListAsync();`. EF Core **строит SQL динамически**, добавляя только применённые фильтры. **Не делать**: `Where(u => name == null || u.Name.Contains(name))` — генерирует sub-optimal SQL с условными branches. **Не делать**: загрузить всё через `ToList`, потом фильтровать в C#. **Best practice**: declarative composition через AsQueryable + conditional Where.

---

## 3. Sorting — OrderBy

### 3.1. Single column

```csharp
// По возрастанию
var sorted = await _db.Users
    .OrderBy(u => u.Name)
    .ToListAsync();

// По убыванию
var sortedDesc = await _db.Users
    .OrderByDescending(u => u.CreatedAt)
    .ToListAsync();
```

### 3.2. Multiple columns

```csharp
// По возрасту (asc), затем по имени (asc)
var sorted = await _db.Users
    .OrderBy(u => u.Age)
    .ThenBy(u => u.Name)
    .ToListAsync();

// Mix asc / desc
var sorted2 = await _db.Users
    .OrderBy(u => u.Department)
    .ThenByDescending(u => u.Salary)
    .ThenBy(u => u.Name)
    .ToListAsync();
```

### 3.3. Pagination — Skip + Take

```csharp
const int pageSize = 20;
int page = 1;   // 1-based

var users = await _db.Users
    .OrderBy(u => u.Id)   // !!! ВСЕГДА ORDER BY перед Skip
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

⚠️ **Без OrderBy результаты Skip+Take непредсказуемы** — каждый раз другие записи могут вернуться. SQL стандарт не гарантирует ordering без `ORDER BY`.

### 3.4. Конфликт sort + paging

```csharp
// Получить всего записей для расчёта total pages
var total = await _db.Users.CountAsync();
var totalPages = (int)Math.Ceiling(total / (double)pageSize);

// Получить страницу
var users = await _db.Users
    .OrderBy(u => u.Id)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

return new PagedResult<User>
{
    Items = users,
    Page = page,
    PageSize = pageSize,
    TotalCount = total,
    TotalPages = totalPages
};
```

> [!question]- Интервью: всегда нужен OrderBy перед Skip/Take?
> **Да, всегда** — SQL Server warning: "The query uses a row limiting operator (OFFSET/FETCH/TOP) and does not have an ORDER BY clause; results may not be deterministic". В практике: записи могут возвращаться в **разном порядке** между вызовами, что ломает pagination (страница 2 возвращает дубли страницы 1 или пропускает записи). **Всегда**: `_db.Users.OrderBy(u => u.Id).Skip((page-1)*size).Take(size)`. Best PK для ordering — index по Id (auto-increment) — самый быстрый.

---

## 4. First / Single / Find — получить одну запись

### 4.1. FindAsync — по PK, fastest

```csharp
// Best для primary key lookup
var user = await _db.Users.FindAsync(42);
// Возвращает null если не найдено

// Преимущества:
// - Сначала проверяет change tracker (no SQL если уже загружен)
// - Затем SQL: SELECT * FROM Users WHERE Id = 42
// - Самый быстрый для PK lookup
```

### 4.2. FirstOrDefault — первая по предикату или null

```csharp
var alice = await _db.Users
    .FirstOrDefaultAsync(u => u.Email == "alice@x.com");

if (alice == null)
{
    Console.WriteLine("Not found");
    return;
}

// SQL: SELECT TOP 1 * FROM Users WHERE Email = 'alice@x.com'
```

### 4.3. SingleOrDefault — единственная или null

```csharp
var user = await _db.Users
    .SingleOrDefaultAsync(u => u.Email == "alice@x.com");

// Возвращает null если 0 записей
// THROWS InvalidOperationException если 2+ записи!
// Используй когда уверен что результат уникальный (по unique index)

// SQL: SELECT TOP 2 * FROM Users WHERE Email = 'alice@x.com'
//      (TOP 2 для проверки uniqueness)
```

### 4.4. First / Single — без default (throw if not found)

```csharp
// FirstAsync — throws InvalidOperationException если нет результата
var user = await _db.Users.FirstAsync(u => u.Id == 42);

// Используй когда УВЕРЕН что запись существует (например, после Add)
```

### 4.5. Когда что использовать

```
FindAsync(id):
✅ Lookup by primary key
✅ Fastest (использует change tracker cache)

FirstOrDefaultAsync(predicate):
✅ Lookup by other field (Email, Name)
✅ Множественные результаты возможны — берём первый
✅ Возможно 0 results — null OK

SingleOrDefaultAsync(predicate):
✅ Должен быть UNIQUE результат (или 0)
✅ Throw если найдено 2+ — это bug
✅ Чуть медленнее (TOP 2 проверка)

FirstAsync / SingleAsync:
✅ Точно знаешь что найдётся
❌ Throws если не найдено (часто не надо)

LastOrDefaultAsync:
⚠️ Требует OrderBy (иначе throws)
```

> [!question]- Интервью: разница `FirstOrDefault` и `SingleOrDefault`?
> **`FirstOrDefault`** — берёт **первую** запись соответствующую предикату (или null). Не проверяет uniqueness. SQL: `TOP 1`. **`SingleOrDefault`** — ожидает **0 или 1** запись. **Throws InvalidOperationException** если найдено 2+. SQL: `TOP 2` (проверяет uniqueness). **Use cases**: 1) `FirstOrDefault` — first match по non-unique field, или когда не важен ли uniqueness. 2) `SingleOrDefault` — точно ожидаешь единственный результат (lookup по unique index — Email, OrderNumber). Throw сигнализирует data corruption / bug. **Performance**: First чуть быстрее (TOP 1 vs TOP 2).

---

## 5. Projection — Select

### 5.1. Зачем projection

Не загружай **все колонки** если нужно только несколько:

```csharp
// ❌ Loading 20+ columns just for name and email
var users = await _db.Users.ToListAsync();
foreach (var user in users)
{
    Console.WriteLine($"{user.Name}: {user.Email}");
}

// ✅ Load only needed
var users = await _db.Users
    .Select(u => new { u.Name, u.Email })
    .ToListAsync();
foreach (var u in users)
{
    Console.WriteLine($"{u.Name}: {u.Email}");
}
```

SQL difference:
- `SELECT * FROM Users` → 20+ columns
- `SELECT Name, Email FROM Users` → 2 columns

### 5.2. Projection в DTO

```csharp
public record UserDto(int Id, string Name, string Email);

var dtos = await _db.Users
    .Where(u => u.IsActive)
    .Select(u => new UserDto(u.Id, u.Name, u.Email))
    .ToListAsync();

// Преимущества:
// 1. Только нужные колонки в SQL
// 2. Сразу DTO без mapping
// 3. AsNoTracking автоматически (нет entity tracking)
```

### 5.3. Anonymous types

```csharp
// Quick projection без DTO class
var summary = await _db.Users
    .Select(u => new
    {
        u.Id,
        u.Name,
        FullName = $"{u.FirstName} {u.LastName}",   // computed
        OrderCount = u.Orders.Count   // aggregation!
    })
    .ToListAsync();
```

### 5.4. Computed fields в Select

```csharp
var userInfo = await _db.Users
    .Select(u => new
    {
        u.Id,
        u.Name,
        IsAdult = u.Age >= 18,
        DaysSinceCreated = (DateTime.UtcNow - u.CreatedAt).Days,
        OrderTotal = u.Orders.Sum(o => o.Total)   // sub-aggregation
    })
    .ToListAsync();

// EF Core преобразует это в эффективный SQL с CASE / DATEDIFF / SUM
```

### 5.5. Select Many — flatten collections

```csharp
// User → Orders relationship
public class User
{
    public int Id { get; set; }
    public List<Order> Orders { get; set; } = new();
}

// Все orders всех users
var allOrders = await _db.Users
    .SelectMany(u => u.Orders)
    .ToListAsync();
// Эквивалент: _db.Orders.ToListAsync() (через navigation)
```

### 5.6. Не делай

```csharp
// ❌ Select после ToList — ничего не оптимизирует
var users = await _db.Users.ToListAsync();
var dtos = users.Select(u => new UserDto(u.Id, u.Name, u.Email)).ToList();
// SQL: SELECT * FROM Users (все колонки!)
// Mapping в C#

// ✅ Select до ToList — projection в SQL
var dtos = await _db.Users
    .Select(u => new UserDto(u.Id, u.Name, u.Email))
    .ToListAsync();
// SQL: SELECT Id, Name, Email FROM Users (только нужное)
```

> [!question]- Интервью: зачем Select в EF Core?
> 1) **Projection** — загрузка только нужных колонок (вместо `SELECT *`). 2) **DTO mapping** — сразу в DTO без manual mapping. 3) **Computed fields** — `IsAdult = u.Age >= 18`, переводится в SQL `CASE`. 4) **Aggregations** — `OrderCount = u.Orders.Count` в одной query. 5) **Performance** — меньше data в network, меньше memory. 6) **AsNoTracking automatic** — anonymous/DTO types не tracked. **Best practice 2024+**: `Select` для read-only paths (API responses, list views). Без Select — entity tracking + все колонки.

---

## 6. Aggregations — Count / Sum / Average / Min / Max

### 6.1. Count

```csharp
// Total count
var totalUsers = await _db.Users.CountAsync();

// Conditional count
var adultCount = await _db.Users.CountAsync(u => u.Age >= 18);

// LongCountAsync — для больших таблиц
var huge = await _db.LogEntries.LongCountAsync();
// long вместо int (если > 2 billion)
```

### 6.2. Any / All

```csharp
// Existence check (faster than Count > 0)
var hasUsers = await _db.Users.AnyAsync();
var hasAdmins = await _db.Users.AnyAsync(u => u.IsAdmin);

// Universal check
var allActive = await _db.Users.AllAsync(u => u.IsActive);
```

```csharp
// ❌ Slow — counts all matching
if (await _db.Users.CountAsync(u => u.Email == email) > 0) { ... }

// ✅ Fast — stops at first match
if (await _db.Users.AnyAsync(u => u.Email == email)) { ... }
```

### 6.3. Sum / Average / Min / Max

```csharp
var totalRevenue = await _db.Orders.SumAsync(o => o.Total);
var avgOrder = await _db.Orders.AverageAsync(o => o.Total);
var maxOrder = await _db.Orders.MaxAsync(o => o.Total);
var minOrder = await _db.Orders.MinAsync(o => o.Total);

// Combined в одном query через GroupBy (см. ниже)
```

### 6.4. GroupBy — агрегация по группам

```csharp
// Count orders per user
var orderCounts = await _db.Orders
    .GroupBy(o => o.UserId)
    .Select(g => new
    {
        UserId = g.Key,
        Count = g.Count(),
        Total = g.Sum(o => o.Total),
        Avg = g.Average(o => o.Total)
    })
    .ToListAsync();

// SQL:
// SELECT UserId, COUNT(*), SUM(Total), AVG(Total)
// FROM Orders
// GROUP BY UserId
```

### 6.5. Group по нескольким полям

```csharp
var stats = await _db.Orders
    .GroupBy(o => new { o.UserId, o.Status })
    .Select(g => new
    {
        g.Key.UserId,
        g.Key.Status,
        Count = g.Count(),
        Total = g.Sum(o => o.Total)
    })
    .ToListAsync();
```

### 6.6. Having — фильтр после GroupBy

```csharp
// Users with > 5 orders
var topUsers = await _db.Orders
    .GroupBy(o => o.UserId)
    .Where(g => g.Count() > 5)
    .Select(g => new { UserId = g.Key, Count = g.Count() })
    .ToListAsync();

// SQL: ... GROUP BY UserId HAVING COUNT(*) > 5
```

> [!question]- Интервью: разница `Any` и `Count > 0`?
> **`Any()`** — `SELECT 1 FROM ... WHERE ...` с stop-on-first-match. **Stops** при первой найденной записи. **`Count() > 0`** — `SELECT COUNT(*) FROM ...` — считает **все** matching records, потом сравнивает. **Performance**: Any namного быстрее на больших таблицах (sub-millisecond vs seconds для миллионов rows). **Use cases**: 1) **Existence check** — Any. 2) **Точное число** — Count. **Bonus**: `IEnumerable.Any()` (in-memory) тоже быстрее Count потому что enumerator.MoveNext() и stop.

---

## 7. Joins — связанные таблицы

### 7.1. Navigation properties — рекомендуется

```csharp
public class User
{
    public int Id { get; set; }
    public List<Order> Orders { get; set; } = new();   // navigation
}

public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public User User { get; set; } = null!;   // navigation
    public decimal Total { get; set; }
}

// EF Core понимает relationship по conventions
```

### 7.2. Include — eager loading

```csharp
// Загрузить orders вместе с users
var users = await _db.Users
    .Include(u => u.Orders)
    .ToListAsync();

// Доступ — без extra query
foreach (var user in users)
{
    foreach (var order in user.Orders)
        Console.WriteLine($"{user.Name}: {order.Total}");
}

// SQL: один JOIN query (или несколько optimized — EF Core решает)
```

### 7.3. ThenInclude — глубокие navigation

```csharp
// User → Orders → OrderItems → Product
var users = await _db.Users
    .Include(u => u.Orders)
        .ThenInclude(o => o.Items)
            .ThenInclude(i => i.Product)
    .ToListAsync();
```

### 7.4. Filtered Include (.NET 5+)

```csharp
// Include только paid orders
var users = await _db.Users
    .Include(u => u.Orders.Where(o => o.Status == "Paid"))
    .ToListAsync();
```

### 7.5. Manual join — когда нет navigation

```csharp
// Если не хочешь добавить navigation property
var query = await _db.Users
    .Join(_db.Orders,
        u => u.Id,
        o => o.UserId,
        (u, o) => new { User = u, Order = o })
    .Where(joined => joined.Order.Total > 100)
    .Select(joined => new
    {
        UserName = joined.User.Name,
        OrderTotal = joined.Order.Total
    })
    .ToListAsync();

// Verbose — обычно лучше использовать navigation
```

### 7.6. Explicit loading — отдельный запрос

```csharp
// Сначала users, потом orders отдельным запросом
var users = await _db.Users.ToListAsync();

foreach (var user in users)
{
    await _db.Entry(user)
        .Collection(u => u.Orders)
        .LoadAsync();
}

// ❌ N+1 problem — N запросов!
// Используй Include в большинстве случаев
```

### 7.7. Lazy loading — опасно для production

```csharp
// Если включён lazy loading proxies
public virtual List<Order> Orders { get; set; } = new();

// Доступ к Orders → автоматический SQL запрос
foreach (var user in users)
{
    Console.WriteLine(user.Orders.Count);   // SQL запрос!
}
// 100 users → 101 query (1 + 100)
```

**Best practice 2024+**: lazy loading **отключи**. Используй Include explicit. Меньше bugs, меньше hidden N+1.

> [!question]- Интервью: что такое eager / lazy / explicit loading?
> 1) **Eager** — загрузить связанные данные вместе через `Include()`. Один query (или несколько optimized). Recommended. 2) **Lazy** — autoload при доступе к navigation property. Скрытые SQL queries — N+1 risk. Требует `virtual` properties + Microsoft.EntityFrameworkCore.Proxies. **Anti-pattern для production** — disable. 3) **Explicit** — manual load после загрузки entity через `_db.Entry(entity).Collection(...).LoadAsync()`. Useful когда не знаешь нужно ли заранее. **Best practice 2024+**: eager loading с `Include()` + `AsNoTracking()` для read paths.

---

## 8. Common pitfalls

### 8.1. ToList перед фильтрацией

```csharp
// ❌ Загружает все users
var users = await _db.Users.ToListAsync();
var adults = users.Where(u => u.Age >= 18).ToList();
```

**Фикс:** Where перед ToList.

### 8.2. Включая ВСЕ navigation

```csharp
// ❌ Loading массивные связанные данные
var users = await _db.Users
    .Include(u => u.Orders)
    .Include(u => u.Comments)
    .Include(u => u.Reviews)
    .Include(u => u.LoginHistory)
    .ToListAsync();
// Огромный SQL JOIN или несколько queries
```

**Фикс:** load только что нужно. Используй projection (Select) для list views.

### 8.3. Count > 0 вместо Any

```csharp
if (await _db.Users.CountAsync(u => u.Email == email) > 0) { ... }
// ❌ Slow на больших таблицах
```

**Фикс:** `await _db.Users.AnyAsync(u => u.Email == email)`.

### 8.4. Skip без OrderBy

```csharp
var page = await _db.Users.Skip(20).Take(10).ToListAsync();
// ❌ Непредсказуемый порядок
```

**Фикс:** `OrderBy(u => u.Id).Skip(...).Take(...)`.

### 8.5. SingleOrDefault на non-unique field

```csharp
var user = await _db.Users.SingleOrDefaultAsync(u => u.Name == "Alice");
// ❌ Throws если 2 Alice's в БД
```

**Фикс:** FirstOrDefault для non-unique, Single только для unique field.

### 8.6. Eager loading lazy navigation

```csharp
foreach (var user in users)
{
    Console.WriteLine(user.Orders.Count);   // ❌ N+1 если lazy
}
```

**Фикс:** Include + disable lazy loading.

### 8.7. Strings concat в Where

```csharp
// ❌ ToString() / методы которые EF не понимает
var users = await _db.Users
    .Where(u => u.Id.ToString().Contains("42"))   // ❌ Может не translate
    .ToListAsync();
```

**Фикс:** `Where(u => u.Id == 42)` или используй EF.Functions.Like.

### 8.8. Custom C# methods в Where

```csharp
public bool IsValid(User u) => u.Email != null && u.Email.Contains("@");

// ❌ Не translate в SQL
var valid = await _db.Users.Where(u => IsValid(u)).ToListAsync();
```

**Фикс:** inline предикат в Where.

### 8.9. Date.Now вместо UtcNow

```csharp
var recent = await _db.Users
    .Where(u => u.CreatedAt > DateTime.Now.AddDays(-7))
    .ToListAsync();
// ❌ Local time может вызвать problems в production (timezones)
```

**Фикс:** всегда UTC: `DateTime.UtcNow.AddDays(-7)`.

### 8.10. Forgot AsNoTracking

```csharp
// API endpoint — read-only
public async Task<List<UserDto>> GetAllAsync() =>
    await _db.Users
        .Select(u => new UserDto(u.Id, u.Name))
        .ToListAsync();
// Хорошо — projection в DTO, но если без projection:

public async Task<List<User>> GetAllAsync() =>
    await _db.Users.ToListAsync();
// ❌ Без AsNoTracking — overhead change tracker
```

**Фикс:** `.AsNoTracking()` для read-only.

> [!question]- Интервью: топ-3 query mistakes?
> 1) **N+1 query** — loop по entities + access navigation = N+1 SQL queries. Fix: `Include()` для eager loading. 2) **ToList() слишком рано** — загружаешь ВСЕ записи, фильтруешь в C#. Fix: фильтр в SQL через `Where()` ДО `ToList()`. 3) **Count > 0 вместо Any()** — Count выполняет full scan, Any stops at first match. Особенно critical на больших таблицах. **Bonus**: SingleOrDefault на non-unique field — throws при 2+ matches. Use FirstOrDefault если non-unique.

---

## 9. Cheat sheet

```csharp
// === Filtering ===
var query = _db.Users
    .Where(u => u.IsActive)
    .Where(u => u.Age >= 18);

// String
.Where(u => u.Name.StartsWith("A"))
.Where(u => u.Email.Contains("@gmail"))
.Where(u => u.Name.Length > 5)

// Date
.Where(u => u.CreatedAt > DateTime.UtcNow.AddDays(-7))
.Where(u => u.CreatedAt.Year == 2024)

// IN clause
.Where(u => ids.Contains(u.Id))

// === Sorting ===
.OrderBy(u => u.Name)
.OrderByDescending(u => u.CreatedAt)
.OrderBy(u => u.Department).ThenBy(u => u.Name)

// === Pagination ===
.OrderBy(u => u.Id)   // !!! always
.Skip((page - 1) * pageSize)
.Take(pageSize)

// === Single record ===
await _db.Users.FindAsync(id);                              // by PK
await _db.Users.FirstOrDefaultAsync(u => u.Email == "x");   // by predicate
await _db.Users.SingleOrDefaultAsync(u => u.Email == "x"); // unique
await _db.Users.FirstAsync(u => u.Id == id);               // throws if not found

// === Projection ===
.Select(u => new { u.Id, u.Name })
.Select(u => new UserDto(u.Id, u.Name, u.Email))

// === Computed ===
.Select(u => new
{
    u.Id,
    IsAdult = u.Age >= 18,
    OrderCount = u.Orders.Count
})

// === Aggregations ===
await _db.Users.CountAsync();
await _db.Users.CountAsync(u => u.IsActive);
await _db.Users.AnyAsync(u => u.Email == email);
await _db.Orders.SumAsync(o => o.Total);
await _db.Orders.AverageAsync(o => o.Total);

// === GroupBy ===
.GroupBy(o => o.UserId)
.Select(g => new
{
    UserId = g.Key,
    Count = g.Count(),
    Total = g.Sum(o => o.Total)
})

// === Eager loading ===
.Include(u => u.Orders)
.Include(u => u.Orders).ThenInclude(o => o.Items)
.Include(u => u.Orders.Where(o => o.Status == "Paid"))

// === No-tracking для read-only ===
.AsNoTracking()

// === Common pattern: List API endpoint ===
var users = await _db.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .Select(u => new UserDto(u.Id, u.Name, u.Email))
    .ToListAsync();
```

---

## 10. Practice exercises

### 10.1. Search endpoint

Реализуй `SearchUsersAsync`:

```csharp
public async Task<List<UserDto>> SearchUsersAsync(
    string? namePattern = null,
    int? minAge = null,
    int? maxAge = null,
    bool? activeOnly = null,
    int page = 1,
    int pageSize = 20,
    string orderBy = "Name");   // "Name" / "CreatedAt" / "Age"
```

Требования:
- Только применяемые фильтры идут в SQL
- Pagination правильная (с OrderBy)
- AsNoTracking
- Projection в DTO
- Sort direction asc/desc через знак минус (`"-CreatedAt"`)

### 10.2. Stats endpoint

```csharp
public class UserStatsDto
{
    public int TotalUsers { get; set; }
    public int ActiveUsers { get; set; }
    public int AdultUsers { get; set; }
    public decimal AverageAge { get; set; }
    public Dictionary<string, int> UsersByCountry { get; set; } = new();
}

public async Task<UserStatsDto> GetStatsAsync()
{
    // ?
}
```

Один или несколько queries — оптимизируй.

### 10.3. Top customers

```csharp
public class TopCustomerDto
{
    public int UserId { get; set; }
    public string Name { get; set; } = "";
    public int OrderCount { get; set; }
    public decimal TotalSpent { get; set; }
    public decimal AverageOrder { get; set; }
}

public async Task<List<TopCustomerDto>> GetTopCustomersAsync(int top = 10)
{
    // Top N customers by total spent
    // Включить только paid orders
    // OrderCount > 0
}
```

Используй GroupBy + Sum + OrderByDescending + Take.

---

## 11. Что читать дальше

1. **`EFCore/Senior/queries-performance.md`** — N+1, optimization, profiling
2. **`EFCore/Senior/relationships.md`** — one-to-many, many-to-many, owned types
3. **`EFCore/Senior/basics-tracking.md`** — change tracker deep dive
4. **`EFCore/Middle/dapper-comparison.md`** — когда EF, когда Dapper

---

## 12. См. также

- [[ef-basics|EFCore/Junior/ef-basics]] — DbContext setup, basic CRUD
- [[queries-performance|EFCore/Senior/queries-performance]] — optimization
- [[relationships|EFCore/Senior/relationships]] — entity relationships
- [[basics-tracking|EFCore/Senior/basics-tracking]] — change tracking
- [[collections-linq|CSharp/Junior/collections-linq]] — LINQ basics

---

## 13. Reading list

- **Microsoft Docs — Querying Data** — learn.microsoft.com/ef/core/querying/
- **Microsoft Docs — LINQ Query Operators** — learn.microsoft.com/dotnet/csharp/linq/
- **"Entity Framework Core in Action" — Jon Smith** (queries chapter)
- **EF Core blog** — devblogs.microsoft.com/dotnet/category/entity-framework
