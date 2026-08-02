---
tags: [performance, caching, redis, memory-cache, distributed-cache, strategies]
level: Middle to Senior
date: 2026-08-02
---

# Caching Strategies — стратегии кеширования

> **Какие виды кеша есть, когда какой применять, как избежать stale data, как не сделать хуже**. Глубокий разбор cache patterns.

---

## Что это, зачем и когда

### Что такое кеш?

**Сохранение результата вычисления чтобы не вычислять заново**. Trade time для memory.

**Аналогия:** Запомнить, что 7×8=56, чтобы не считать каждый раз. Память (твой мозг) дешевле чем вычисление с нуля.

### Зачем

| Без cache | С cache |
|-----------|---------|
| Каждый запрос — поход в БД (~10-100ms) | Из памяти (~1ms) |
| Сложный отчёт пересчитывается каждый раз | Cached на 5 мин |
| 1000 RPS = 1000 DB queries/sec | 100 DB queries + 900 cache hits |
| External API hit limit / cost | Reduce calls drastically |

### Когда **не** надо кешировать

❌ **Плохие кандидаты:**
- Часто меняется (caсhe всегда stale)
- Per-user уникальные данные (low hit rate)
- Critical correctness (банковский баланс — лучше всегда из БД)
- Маленький dataset который и так помещается в memory

✅ **Хорошие кандидаты:**
- Read-heavy + изменяется редко (settings, configs)
- Expensive вычисления (reports, aggregations)
- Reference data (countries list, currencies)
- API responses который дороги

---

## 1. Уровни кеша

```
┌──────────────────────────────────────┐
│ L1 CPU cache (~5 ns)                 │  ← managed by hardware
├──────────────────────────────────────┤
│ Process memory (RAM, ~100 ns)        │  ← IMemoryCache
├──────────────────────────────────────┤
│ Distributed cache (~1 ms)            │  ← Redis, Memcached
│   - Same datacenter                   │
├──────────────────────────────────────┤
│ Database (~10-100 ms)                 │  ← последний resort
└──────────────────────────────────────┘
```

Каждый уровень в **10-100x медленнее** предыдущего но **больше capacity**.

---

## 2. Cache patterns

### Pattern 1: Cache-Aside (Lazy Loading)

Самый частый pattern.

```csharp
public async Task<User?> GetUserAsync(int id)
{
    // 1. Пробуй cache
    if (_cache.TryGetValue($"user_{id}", out User? cached))
        return cached;
    
    // 2. Cache miss — fetch from DB
    var user = await _db.Users.FindAsync(id);
    
    // 3. Save в cache
    if (user != null)
        _cache.Set($"user_{id}", user, TimeSpan.FromMinutes(5));
    
    return user;
}
```

**Плюсы:**
- Простой
- Cache содержит только что реально запрошено
- Resilient — если cache упал, fallback to DB

**Минусы:**
- First request — slow (cache miss)
- Stale data возможна (cache не знает про update в DB)

### Pattern 2: Write-Through

Каждый write идёт И в DB И в cache.

```csharp
public async Task UpdateUserAsync(User user)
{
    // 1. Update DB
    _db.Users.Update(user);
    await _db.SaveChangesAsync();
    
    // 2. Update cache
    _cache.Set($"user_{user.Id}", user, TimeSpan.FromMinutes(5));
}
```

**Плюсы:**
- Cache всегда актуален
- Consistency

**Минусы:**
- Slower writes
- Если cache fail после DB write — inconsistency

### Pattern 3: Write-Behind (Write-Back)

Write идёт в cache, потом асинхронно в DB.

```csharp
public Task UpdateUserAsync(User user)
{
    // 1. Update cache (быстро)
    _cache.Set($"user_{user.Id}", user, TimeSpan.FromMinutes(5));
    
    // 2. Queue для async DB update
    _writeQueue.Enqueue(user);
    
    return Task.CompletedTask;
}

// Background worker
async Task ProcessWriteQueue()
{
    await foreach (var user in _writeQueue.ReadAllAsync())
        await _db.SaveChangesAsync();
}
```

**Плюсы:**
- Очень быстрые writes
- Batch writes к DB

**Минусы:**
- Risk потери данных (cache crash до flush)
- Сложно (нужна durable queue)

**Когда:** high-write systems где OK to lose few seconds of writes.

### Pattern 4: Read-Through

Cache **сам** загружает данные. Wrapper над cache.

```csharp
public class ReadThroughCache<T>
{
    private readonly IMemoryCache _cache;
    private readonly Func<string, Task<T?>> _loader;
    
    public async Task<T?> GetAsync(string key)
    {
        return await _cache.GetOrCreateAsync(key, async entry =>
        {
            entry.SetAbsoluteExpiration(TimeSpan.FromMinutes(5));
            return await _loader(key);
        });
    }
}

// Использование
var userCache = new ReadThroughCache<User>(_cache, async key =>
{
    var id = int.Parse(key.Split('_')[1]);
    return await _db.Users.FindAsync(id);
});

var user = await userCache.GetAsync("user_42");
```

**Плюсы:**
- Encapsulation — caller не знает про cache
- Stampede protection (см. ниже)

### Pattern 5: Refresh-Ahead

Refresh **до** expiration — никаких cache miss.

```csharp
public class RefreshAheadCache<T>
{
    private T _value;
    private DateTime _expiry;
    private readonly Timer _refreshTimer;
    private readonly Func<Task<T>> _loader;
    
    public RefreshAheadCache(Func<Task<T>> loader, TimeSpan ttl)
    {
        _loader = loader;
        // Refresh за 80% от TTL
        var refreshInterval = ttl * 0.8;
        _refreshTimer = new Timer(async _ => await Refresh(), null, refreshInterval, refreshInterval);
    }
    
    private async Task Refresh()
    {
        _value = await _loader();
        _expiry = DateTime.UtcNow.Add(TimeSpan.FromMinutes(5));
    }
    
    public T Get() => _value;
}
```

**Плюсы:**
- Никаких cache miss
- Latency consistent

**Минусы:**
- Сложнее
- Refresh даже если не запрашивают

---

## 3. Реализация в .NET — кратко

Технические детали всех механизмов — setup, опции, тонкости, production-checklist — в canonical-файле [[caching|Caching и Rate Limiting]]. Здесь — только выбор:

| Механизм | Что это | Когда |
|----------|---------|-------|
| `IMemoryCache` | In-process, микросекунды | Single-instance, небольшие горячие данные |
| `IDistributedCache` (Redis) | Shared между репликами, ~1-2 ms (сериализация + network) | Multi-instance, общий кэш |
| `HybridCache` | L1 (memory) + L2 (Redis) + stampede protection одним API | Новые проекты — дефолт |
| Output Caching (.NET 7+) | Кэш целого HTTP-ответа | Одинаковые ответы: public API, анонимные endpoints |

`HybridCache` — это NuGet-пакет `Microsoft.Extensions.Caching.Hybrid` (GA 2025), а не часть runtime: работает вплоть до `netstandard2.0` / .NET Framework 4.7.2; в документации и шаблонах позиционируется с .NET 9.

Expiration: **Absolute** — reference data (обновляется периодически), **Sliding** — сессии и recent items, оба вместе — sliding для активного использования с absolute-потолком.

---

## 4. Cache invalidation — самая сложная часть

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### Strategy 1: TTL (time-based)

```csharp
cache.Set(key, value, TimeSpan.FromMinutes(5));
// Через 5 минут — expires автоматически
```

**Плюсы:** простой, garantирует bounded staleness.
**Минусы:** stale data до expiration.

### Strategy 2: Manual invalidation

```csharp
public async Task UpdateUserAsync(User user)
{
    _db.Users.Update(user);
    await _db.SaveChangesAsync();
    
    // Invalidate cache
    cache.Remove($"user_{user.Id}");
}
```

**Плюсы:** instant consistency.
**Минусы:** легко забыть удалить cache. Невозможно если update идёт извне (другой service).

### Strategy 3: Pub/Sub invalidation

```csharp
// Service A: update + publish event
public async Task UpdateUserAsync(User user)
{
    await _db.SaveChangesAsync();
    await _redis.PublishAsync("user-updated", user.Id.ToString());
}

// Service B: subscribe + invalidate
_redis.Subscribe("user-updated", (channel, message) =>
{
    var id = int.Parse(message);
    cache.Remove($"user_{id}");
});
```

**Когда:** distributed system, cross-service.

### Strategy 4: Tag-based

```csharp
// Update — invalidate all с tag "products"
cache.RemoveByTag("products");
```

В HybridCache (пакет `Microsoft.Extensions.Caching.Hybrid`):
```csharp
await cache.SetAsync(key, value, tags: new[] { "products", "category-1" });

// Invalidate by tag
await cache.RemoveByTagAsync("products");
```

### Strategy 5: Versioning

```csharp
private static int _version = 1;

public string CacheKey(int id) => $"v{_version}:user_{id}";

public void InvalidateAll()
{
    _version++;
    // Старые ключи "v1:..." больше не используются, expire по TTL
}
```

**Когда:** массовая invalidation (новый release с breaking schema).

---

## 5. Cache stampede

### Проблема

Cache expired → 1000 concurrent requests → 1000 DB hits → DB overload.

```
Time T:    user_1 in cache
Time T+5:  user_1 expires
Time T+5:  Request 1 — cache miss, query DB
Time T+5:  Request 2 — cache miss, query DB  ← race condition!
...
Time T+5:  Request 1000 — cache miss, query DB
DB: 💀
```

### Решения (реализации — в [[caching|Caching и Rate Limiting]])

- **Один загружает, остальные ждут** — `SemaphoreSlim` с double-check, in-process SingleFlight или distributed mutex.
- **HybridCache** — stampede protection из коробки: конкурентные запросы разделяют один fetch.
- **Probabilistic early refresh / XFetch** — обновление до истечения TTL с вероятностью, растущей ближе к expiry; фоновый refresh, пока отдаётся старое значение.

---

## 6. Cache в реальной архитектуре

Типовая связка — несколько слоёв, каждый ловит свою долю трафика:

```
User → CDN / edge        (static assets, public GET)
     → L1 IMemoryCache   (микросекунды; process-local, не шарится между репликами)
     → L2 Redis          (миллисекунды; shared, переживает рестарт процесса)
     → DB                (последний resort, ~10-100 ms)
```

Ручную связку L1+L2 (проверить memory → Redis → DB, наполняя кэши по пути назад) делает `HybridCache` автоматически — см. [[caching|Caching и Rate Limiting]].

---

## 7. Common Pitfalls

### 1. Cache without TTL

```csharp
// ❌ Cache forever
_cache.Set("key", value);

// ✅ Always have TTL
_cache.Set("key", value, TimeSpan.FromMinutes(5));
```

### 2. Cache thundering herd

См. cache stampede.

### 3. Cache wrong data

```csharp
// ❌ Cache user-specific data в shared cache
_cache.Set("dashboard", currentUserDashboard);  // mixed между users!

// ✅ Per-user key
_cache.Set($"dashboard_{userId}", dashboard);
```

### 4. Cache big объекты

```csharp
// ❌ Cache 100 MB list
_cache.Set("all_users", allUsers);  // sucks memory!

// ✅ Cache pages / individual items
foreach (var user in users)
    _cache.Set($"user_{user.Id}", user);
```

### 5. Cache в Singleton без cleanup

```csharp
// ❌ Никогда не expires — memory leak
public class Service
{
    private static Dictionary<int, User> _cache = new();
    
    public User Get(int id)
    {
        if (!_cache.ContainsKey(id))
            _cache[id] = LoadFromDb(id);
        return _cache[id];
    }
}
// Через год — _cache содержит миллионы entries, OOM
```

### 6. Cache changes в request

```csharp
// ❌ Modifies cached object
var user = _cache.Get<User>("user_1");
user.Name = "Modified";  // ⚠️ Меняет cached version!
```

**Лечение:** clone, immutable records, или Defensive copies.

### 7. Игнорирование cache failures

```csharp
// ❌ Если Redis down — всё падает
var data = await _redis.GetStringAsync(key);
return JsonSerializer.Deserialize<User>(data);

// ✅ Graceful fallback
try
{
    var data = await _redis.GetStringAsync(key);
    if (data != null) return JsonSerializer.Deserialize<User>(data);
}
catch (RedisException) { /* log, continue */ }

return await _db.Users.FindAsync(id);
```

### 8. Cache key collisions

```csharp
// ❌ Конфликт между приложениями (same Redis instance)
_cache.Set("user_1", ...);  // App A
_cache.Set("user_1", ...);  // App B — overwrite!

// ✅ Namespace
_cache.Set("MyApp:Users:1", ...);
```

---

## 8. Best Practices

- **Cache-aside default** — простой, resilient
- **TTL обязательно** — bounded staleness
- **Per-user keys** для user-specific data
- **Namespace prefixes** в shared cache (`MyApp:Users:1`)
- **HybridCache** (NuGet, GA 2025) для multi-tier
- **Output caching** для public APIs
- **Stampede protection** на high-traffic items
- **Graceful degradation** — если cache down, fallback to DB
- **Monitor hit rate** — hit rate < 80% значит кэш, скорее всего, не окупается
- **Profile cache effectiveness** — hit/miss ratio, latency
- **Tag-based invalidation** для grouped data
- **Pub/Sub invalidation** для distributed systems
- **Не cache mutable объекты** — clone или immutable

---

## 9. Когда что — flowchart

```
Need cache?
├── No → don't cache
└── Yes
    ├── Single instance app?
    │   └── IMemoryCache
    ├── Multi-instance app?
    │   ├── Need cross-instance? → Redis (IDistributedCache)
    │   ├── Want both? → HybridCache
    │   └── Per-instance OK? → IMemoryCache
    ├── HTTP responses cache?
    │   └── Output Caching middleware
    ├── Need stampede protection?
    │   └── HybridCache or Lock
    ├── Need cross-service invalidation?
    │   └── Pub/Sub (Redis) + manual invalidation
    └── Need tags?
        └── HybridCache или custom logic
```

---

## Case Studies

### Case Study #1 — API endpoint slow на больших данных

**Сценарий:** `GET /reports/sales` возвращает данные за год — p99 latency 30 сек.

**❌ Memory + CPU bottleneck:**
```csharp
public async Task<List<SalesRow>> GetReport()
{
    var orders = await _db.Orders.ToListAsync();  // 1M rows в memory
    return orders
        .Where(o => o.Year == 2026)
        .GroupBy(o => o.ProductId)
        .Select(g => new SalesRow { ProductId = g.Key, Total = g.Sum(o => o.Total) })
        .ToList();
}
```

**✅ DB-level aggregation:**
```csharp
public async Task<List<SalesRow>> GetReport() =>
    await _db.Orders
        .Where(o => o.Year == 2026)
        .GroupBy(o => o.ProductId)
        .Select(g => new SalesRow { ProductId = g.Key, Total = g.Sum(o => o.Total) })
        .ToListAsync();
```

**Result:** 30 sec → 200 ms. SQL делает aggregation, не C#.

---

### Case Study #2 — Hot path allocations

**Сценарий:** Method вызывается 100K RPS. Profiler показывает много GC pauses.

**❌ Allocations:**
```csharp
public bool Validate(string input)
{
    var parts = input.Split(',');  // string[] alloc
    var trimmed = parts.Select(p => p.Trim()).ToList();  // List + iterations alloc
    return trimmed.All(t => !string.IsNullOrEmpty(t));
}
```

**✅ Span-based zero-alloc:**
```csharp
public bool Validate(ReadOnlySpan<char> input)
{
    foreach (var range in input.Split(','))
    {
        var part = input[range].Trim();
        if (part.IsEmpty) return false;
    }
    return true;
}
```

**Result:** 0 allocations, 3x faster, fewer GC cycles.

---

### Case Study #3 — Async overhead в hot path

**Сценарий:** Method чаще завершается синхронно (cache hit). `Task` allocation overhead.

**❌:**
```csharp
public async Task<User> GetAsync(int id)
{
    if (_cache.TryGet(id, out var user)) return user;
    return await _db.GetAsync(id);
}
// Каждый cache hit — Task allocation
```

**✅ ValueTask:**
```csharp
public ValueTask<User> GetAsync(int id)
{
    if (_cache.TryGet(id, out var user)) return new ValueTask<User>(user);
    return new ValueTask<User>(_db.GetAsync(id));
}
// Cache hit — zero alloc
```

См. [[async-threading|async-threading]] и [[memory-pooling|Memory Pooling]].


---

## Cheat sheet

| Symptom | Tool / Approach |
|---------|-----------------|
| High CPU | dotnet-trace, dotTrace sampling |
| High memory | dotnet-dump + WinDbg, dotMemory snapshots |
| GC pauses | dotnet-counters, ETW events |
| Slow query | EF logging + Database query plan |
| Slow API | Application Insights / Datadog APM |
| Memory leak | Snapshot diffs (dotMemory, JetBrains) |
| Async deadlock | dotnet-stack threads dump |
| Lock contention | dotnet-trace + Concurrency Visualizer |
| Allocation hot path | BenchmarkDotNet `[MemoryDiagnoser]` |
| Microoptimization | BenchmarkDotNet, disasm |

| Allocation Cost | Bytes |
|-----------------|-------|
| Reference type (object) | 16-24 bytes header + fields |
| string interning | shared, no new allocation |
| boxing int → object | 24 bytes |
| `new List<T>()` empty | 40 bytes |
| `new List<T>(capacity)` | 40 + (capacity × sizeof(T)) |
| Closure | depends on captured vars |
| `async Task` state machine | ~80-200 bytes per call |
| `ValueTask` (sync complete) | 0 bytes |

| Speed | Tool |
|-------|------|
| Microsec measurements | BenchmarkDotNet |
| Millisec end-to-end | Stopwatch + LogInformation |
| Production tracing | OpenTelemetry + Jaeger |
| Real-time monitoring | dotnet-counters --refresh-interval 1 |


---

## Decision tree

```
Performance issue?
│
├── Сначала — где боль?
│   ├── Latency (p99) → APM tools (App Insights, Datadog)
│   ├── Throughput (RPS limit) → load test + profiler
│   ├── Memory → snapshots (dotMemory, dotnet-dump)
│   └── CPU → sampling profiler (dotTrace, perf)
│
├── Bottleneck identified?
│   ├── Database → query plan, indexes, N+1
│   ├── Network → batching, HTTP/2, connection pooling
│   ├── CPU → algorithmic complexity, allocations
│   ├── Memory → object pooling, struct vs class
│   └── Locks → ConcurrentDictionary, lock-free
│
├── Optimization сложность?
│   ├── Easy wins → caching, async/await, pagination
│   ├── Medium → query optimization, batch processing
│   ├── Hard → memory pooling, Span<T>, source generators
│   └── Extreme → unsafe, SIMD, native AOT, custom allocator
│
└── Проверка?
    ├── Benchmark до/после → BenchmarkDotNet
    ├── Real load test → k6, NBomber, JMeter
    └── Production canary → 5% → 50% → 100%
```

**Optimization rule:** Measure → Hypothesize → Optimize → Measure. Никогда не оптимизируй без data.


---

## См. также

- [[performance-fundamentals|Performance Fundamentals]]
- [[caching|Caching в AspNetCore]]
- [[postgresql-deep|PostgreSQL]]
- [[distributed-systems|Distributed Systems]]
- [[gc-memory|GC и память]]

## Reading list

- **Microsoft Docs — Caching** — learn.microsoft.com/aspnet/core/performance/caching
- **HybridCache docs** — learn.microsoft.com/aspnet/core/performance/caching/hybrid
- **Redis documentation** — redis.io/docs
- **Andrew Lock — Cache aside pattern** — andrewlock.net
- **Designing Data-Intensive Applications** — Kleppmann (chapter про caching)
