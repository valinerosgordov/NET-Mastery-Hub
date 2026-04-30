---
tags: [performance, lazy, eager, loading-strategies]
level: Middle
date: 2026-04-30
---

# Lazy vs Eager Loading

> **Когда загружать данные сразу, когда откладывать**. Trade-offs для EF Core, lazy initialization, eager configuration.

---

## Что это, зачем и когда

### Концепция

**Eager** — загрузить **всё сразу**, использовать когда нужно.
**Lazy** — загрузить **только когда понадобится**.

**Аналогия:**
- Eager: купить продукты на неделю заранее (один поход)
- Lazy: бегать в магазин каждый раз когда что-то нужно (много мелких походов)

### Trade-offs

| | Eager | Lazy |
|--|-------|------|
| Initial latency | Higher (load всё) | Lower (load только нужное) |
| Total resources | Predictable | Может быть N+1 |
| Memory | Higher upfront | Spread в time |
| Когда лучше | Знаем что **всё** понадобится | Не знаем что **именно** нужно |

---

## 1. EF Core loading strategies

### Eager loading — `Include`

```csharp
var orders = await _db.Orders
    .Include(o => o.Items)
    .Include(o => o.Customer)
    .ToListAsync();
```

**Плюсы:**
- 1 (или 2) query
- No N+1
- All data ready

**Минусы:**
- Cartesian explosion (см. ниже)
- Loads even unused data

### Explicit loading

```csharp
var order = await _db.Orders.FindAsync(1);

// Load related когда нужно
await _db.Entry(order).Collection(o => o.Items).LoadAsync();
await _db.Entry(order).Reference(o => o.Customer).LoadAsync();
```

**Плюсы:**
- Контроль когда грузить
- Не каждый раз

**Минусы:**
- Дополнительные queries

### Lazy loading

```csharp
public class Order
{
    public virtual Customer Customer { get; set; }
    public virtual ICollection<Item> Items { get; set; }
}

// Configuration
optionsBuilder.UseLazyLoadingProxies();
```

```csharp
var order = await _db.Orders.FindAsync(1);
var customer = order.Customer;  // ⚠️ Triggers query!
foreach (var item in order.Items)  // ⚠️ Triggers another query!
{
}
```

**Плюсы:**
- Code просто
- Loads только использованное

**Минусы:**
- **N+1 problem** очень легко
- Hidden DB calls
- Disposed DbContext → exception

> [!warning] Lazy loading — обычно плохо
> EF Core community рекомендует **explicit/eager** instead. Lazy hides perf issues.

### Projection (selective loading)

```csharp
// Only what you need
var orders = await _db.Orders
    .Select(o => new OrderSummary
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,
        ItemCount = o.Items.Count
    })
    .ToListAsync();
```

**Best of both worlds:** load minimum data, no N+1, type-safe.

См. [[../EFCore/queries-performance|EF Performance]].

---

## 2. Cartesian explosion problem

```csharp
// One Order has many Items + many Tags
var orders = await _db.Orders
    .Include(o => o.Items)
    .Include(o => o.Tags)  // ⚠️
    .ToListAsync();

// Generated SQL — JOIN всех таблиц:
// 100 orders × 10 items × 5 tags = 5000 rows!
```

### Solution: AsSplitQuery

```csharp
var orders = await _db.Orders
    .AsSplitQuery()
    .Include(o => o.Items)
    .Include(o => o.Tags)
    .ToListAsync();

// Generates 3 queries:
// SELECT * FROM Orders
// SELECT * FROM Items WHERE OrderId IN (...)
// SELECT * FROM Tags WHERE OrderId IN (...)
```

**When use SplitQuery:**
- Multiple Include + collections
- Cartesian explosion обнаружено
- Trade: 3 queries но less data overall

См. [[../EFCore/queries-performance|EF Performance]].

---

## 3. Lazy\<T\> initialization

```csharp
// ❌ Expensive init at construction
public class Service
{
    private readonly Dictionary<string, Config> _configs = LoadAllConfigs();
    // 5 sec при каждом DI activation
}

// ✅ Lazy — init at first use
public class Service
{
    private readonly Lazy<Dictionary<string, Config>> _configs = 
        new(() => LoadAllConfigs(), LazyThreadSafetyMode.ExecutionAndPublication);
    
    public Config Get(string key) => _configs.Value[key];
    // First Get → load. Subsequent → cached.
}
```

`LazyThreadSafetyMode`:
- `None` — single thread access
- `PublicationOnly` — racing init OK, last wins
- `ExecutionAndPublication` — thread-safe, full lock (default)

---

## 4. Lazy services в DI

### Lazy resolution

```csharp
// Inject Lazy<IService> — не инициализируется пока не вызовут .Value
public class Controller(Lazy<IExpensiveService> service)
{
    public IActionResult Get()
    {
        // service не создан до этого момента
        return Ok(service.Value.GetData());
    }
}

// Регистрация
builder.Services.AddScoped<Lazy<IExpensiveService>>(sp =>
    new Lazy<IExpensiveService>(() => sp.GetRequiredService<IExpensiveService>()));
```

**Когда:**
- Some endpoints don't use this service
- Expensive initialization
- Avoid loading unused dependencies

> [!info] Trade-off
> Adds complexity. Если service used часто — overhead не оправдан.

---

## 5. Lazy/eager в кеше

### Cache-aside (lazy)

```csharp
// Loaded только при request
public async Task<User?> GetAsync(int id)
{
    return await _cache.GetOrCreateAsync($"user_{id}", async _ =>
        await _db.Users.FindAsync(id));
}
```

### Cache warming (eager)

```csharp
// Load всё в cache при startup
public class CacheWarmupService(IMemoryCache cache, AppDbContext db) : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        var allConfigs = await db.Configs.ToListAsync(ct);
        foreach (var config in allConfigs)
        {
            cache.Set($"config_{config.Key}", config, TimeSpan.FromHours(1));
        }
    }
    
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}

builder.Services.AddHostedService<CacheWarmupService>();
```

**Когда warming:**
- Read-heavy + не меняется
- Predictable startup time допустим
- Cold cache hurts first users

---

## 6. Module loading в больших apps

### Lazy assemblies в Blazor WASM

```xml
<ItemGroup>
  <BlazorWebAssemblyLazyLoad Include="HeavyFeature.dll" />
</ItemGroup>
```

```csharp
@inject LazyAssemblyLoader Loader

@code {
    private async Task LoadFeature()
    {
        var assemblies = await Loader.LoadAssembliesAsync(["HeavyFeature.dll"]);
        // Module loaded only когда user navigates to feature
    }
}
```

---

## 7. ORM-агностичные паттерны

### IEnumerable vs IList в API

```csharp
// ❌ Materialized — caller получает full list
public List<Order> GetActive()
{
    return _db.Orders.Where(o => o.IsActive).ToList();
}

// ✅ IEnumerable — lazy, caller выбирает
public IEnumerable<Order> GetActive()
{
    return _db.Orders.Where(o => o.IsActive);
}

// Caller:
// Хочет первые 5 — only 5 загрузятся
var first5 = service.GetActive().Take(5).ToList();
```

> [!warning] IEnumerable can re-enumerate
> Each `foreach` re-runs query. Если caller использует несколько раз — материalize в ToList().

---

## 8. Common Pitfalls

### 1. Lazy loading в API responses

```csharp
// ❌ Returns Order с virtual ICollection<Item> Items
public async Task<Order> GetOrder(int id)
{
    return await _db.Orders.FindAsync(id);
}

// JSON serializer triggers lazy load → дополнительные queries в middle of serialization
// Может быть disposed DbContext → exception
```

**Лечение:** explicit Include, или DTO projection.

### 2. Multiple Include — Cartesian explosion

См. выше.

### 3. Lazy init в hot path

```csharp
private Lazy<Service> _lazy = new(...);

// Each call — overhead Lazy.Value check
public void DoSomething()
{
    _lazy.Value.Method();
}
```

В hot path — `Lazy<T>.Value` имеет overhead. Если service used always — direct field.

### 4. Eager loading всего что можно

```csharp
// ❌ Loads 50 фкtors даже если не нужно
var order = await _db.Orders
    .Include(o => o.Items)
    .Include(o => o.Customer)
    .Include(o => o.Customer.Address)
    .Include(o => o.PaymentMethod)
    .Include(o => o.Discounts)
    // ... ещё 20 Include
    .FirstAsync(o => o.Id == id);
```

**Лечение:** только то что нужно для конкретного use case (use Projection / multiple methods).

---

## 9. Best Practices

- **Eager (Include) для known relationships** в read scenarios
- **Projection (Select)** для DTOs — minimum data
- **AsSplitQuery** при multiple collections
- **Avoid lazy loading в EF** — usually hides issues
- **Lazy\<T\>** для expensive optional deps
- **Cache warming** для read-heavy + slow init
- **IEnumerable**: lazy chain в pipeline, **List** для random access

---

## См. также

- [[performance-fundamentals|Performance Fundamentals]]
- [[caching-strategies|Caching Strategies]]
- [[optimization-patterns|Optimization Patterns]]
- [[../EFCore/queries-performance|EF Queries Performance]]
- [[../EFCore/basics-tracking|EF Basics & Tracking]]
- [[../AspNetCore/blazor-wasm|Blazor WASM]] (lazy assemblies)

## Reading list

- **EF Core docs — Loading related data** — learn.microsoft.com/ef/core/querying/related-data
- **Jon P Smith — EF Core in Action**
- **Entity Framework Performance** — by various authors
