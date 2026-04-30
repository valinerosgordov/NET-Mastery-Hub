---
tags: [performance, caching, redis, memory-cache, distributed-cache, strategies]
level: Middle to Senior
date: 2026-04-30
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

## 3. .NET — IMemoryCache (in-process)

```csharp
// Setup
builder.Services.AddMemoryCache();

// Usage
public class UserService(IMemoryCache cache, AppDbContext db)
{
    public async Task<User?> GetAsync(int id)
    {
        return await cache.GetOrCreateAsync($"user_{id}", async entry =>
        {
            entry.SlidingExpiration = TimeSpan.FromMinutes(5);
            entry.AbsoluteExpiration = DateTimeOffset.UtcNow.AddHours(1);
            entry.Size = 1;  // если SizeLimit set
            
            return await db.Users.FindAsync(id);
        });
    }
}
```

### Eviction policies

```csharp
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024;  // максимум 1024 "size units"
    options.CompactionPercentage = 0.25;  // при overflow удаляет 25%
});

// Указывать size при Set
cache.Set("key", value, new MemoryCacheEntryOptions
{
    Size = 1,  // 1 entry = 1 unit
    Priority = CacheItemPriority.High,  // last to evict
});
```

### Expiration types

```csharp
new MemoryCacheEntryOptions
{
    // Absolute — точное время expiration
    AbsoluteExpiration = DateTimeOffset.UtcNow.AddHours(1),
    AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1),
    
    // Sliding — обновляется при каждом доступе
    SlidingExpiration = TimeSpan.FromMinutes(10),
}
```

| Тип | Когда |
|-----|-------|
| **Absolute** | Reference data — обновляется periodically |
| **Sliding** | User session, recent results |
| **Both** | Sliding для активного use, Absolute как ceiling |

---

## 4. Distributed cache — Redis

Для multi-instance apps — Memory cache на каждом replicas инвалидация невозможна. Нужен **shared cache**.

```csharp
// Setup
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "MyApp:";
});

// IDistributedCache — abstraction
public class UserService(IDistributedCache cache, AppDbContext db)
{
    public async Task<User?> GetAsync(int id)
    {
        var cacheKey = $"user_{id}";
        var cached = await cache.GetStringAsync(cacheKey);
        if (cached != null)
            return JsonSerializer.Deserialize<User>(cached);
        
        var user = await db.Users.FindAsync(id);
        if (user != null)
        {
            await cache.SetStringAsync(
                cacheKey,
                JsonSerializer.Serialize(user),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
                });
        }
        return user;
    }
}
```

> [!warning] IDistributedCache slow для small values
> Сериализация + JSON + network — ~1-2 ms на operation. Для tiny values — overhead больше gain. Используй MemoryCache для frequently accessed small data.

### Redis-specific features

```csharp
// StackExchange.Redis — direct API для advanced
builder.Services.AddSingleton<IConnectionMultiplexer>(
    ConnectionMultiplexer.Connect("localhost:6379"));

public class AdvancedCache
{
    private readonly IDatabase _db;
    
    public async Task IncrementCounter(string key) =>
        await _db.StringIncrementAsync(key);
    
    public async Task<TimeSpan?> GetTtl(string key) =>
        await _db.KeyTimeToLiveAsync(key);
    
    public async Task SetWithTags(string key, string value, params string[] tags)
    {
        await _db.StringSetAsync(key, value);
        foreach (var tag in tags)
            await _db.SetAddAsync($"tag:{tag}", key);
    }
}
```

См. [[../SQL/postgresql-deep|PostgreSQL Deep]] и Redis docs.

---

## 5. HybridCache (.NET 9+)

Microsoft новый API — combines L1 (memory) + L2 (distributed).

```bash
dotnet add package Microsoft.Extensions.Caching.Hybrid
```

```csharp
builder.Services.AddHybridCache();

public class UserService(HybridCache cache, AppDbContext db)
{
    public async Task<User?> GetAsync(int id, CancellationToken ct)
    {
        return await cache.GetOrCreateAsync(
            $"user_{id}",
            async ct => await db.Users.FindAsync(id, ct),
            new HybridCacheEntryOptions
            {
                Expiration = TimeSpan.FromMinutes(5),
                LocalCacheExpiration = TimeSpan.FromMinutes(1),
            },
            cancellationToken: ct);
    }
}
```

**Преимущества:**
- L1 + L2 встроенный
- Stampede protection (см. ниже)
- Single API

**Когда использовать:** новые проекты на .NET 9+.

---

## 6. Output Caching (.NET 7+)

Cache **whole HTTP response**, not just data.

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder.Cache());
    options.AddPolicy("Expire5min", builder => builder.Expire(TimeSpan.FromMinutes(5)));
});

app.UseOutputCache();

// Endpoint
app.MapGet("/api/products", async (AppDbContext db) =>
    await db.Products.ToListAsync())
.CacheOutput("Expire5min");

// Or attribute
[OutputCache(Duration = 60)]
public async Task<IActionResult> GetProducts() { ... }
```

**Когда:** identical responses for same params (anonymous APIs, public data).

---

## 7. Cache invalidation — самая сложная часть

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

В .NET 9 HybridCache:
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

## 8. Cache stampede

### Проблема

Cache expired → 1000 concurrent requests → 1000 DB hits → DB overload.

```
Time T:    user_1 in cache
Time T+5:  user_1 expires
Time T+5:  Request 1 — cache miss, query DB
Time T+5:  Request 2 — cache miss, query DB  ← race condition!
Time T+5:  Request 3 — cache miss, query DB
...
Time T+5:  Request 1000 — cache miss, query DB
DB: 💀
```

### Solution 1: Lock (single fetch)

```csharp
private static readonly SemaphoreSlim _lock = new(1, 1);

public async Task<User?> GetAsync(int id)
{
    var key = $"user_{id}";
    if (_cache.TryGetValue(key, out User? cached))
        return cached;
    
    await _lock.WaitAsync();
    try
    {
        // Double-check (другой request мог уже наполнить)
        if (_cache.TryGetValue(key, out cached))
            return cached;
        
        var user = await _db.Users.FindAsync(id);
        _cache.Set(key, user, TimeSpan.FromMinutes(5));
        return user;
    }
    finally { _lock.Release(); }
}
```

Только один request загружает, остальные ждут.

### Solution 2: HybridCache stampede protection

```csharp
// Built-in! Multiple concurrent requests share single fetch
await cache.GetOrCreateAsync(key, factory, options);
```

### Solution 3: Probabilistic early refresh

Refresh **до** expiration с некоторой probability:

```csharp
public async Task<User?> GetAsync(int id)
{
    var entry = _cache.Get<CacheEntry<User>>($"user_{id}");
    if (entry == null)
        return await LoadAndCache(id);
    
    // Probabilistically refresh если близко к expiration
    var timeLeft = entry.Expiry - DateTime.UtcNow;
    if (timeLeft < TimeSpan.FromMinutes(1) && Random.Shared.NextDouble() < 0.1)
    {
        // Refresh in background, return old value
        _ = LoadAndCache(id);
    }
    
    return entry.Value;
}
```

Расширенная версия — **XFetch algorithm**.

---

## 9. Cache в реальной архитектуре

### Layer 1 — In-memory (fastest)

```csharp
IMemoryCache  // process-local
```

- Microseconds latency
- Limited by process memory
- Not shared between replicas

### Layer 2 — Distributed (shared)

```csharp
IDistributedCache (Redis)
```

- Milliseconds latency
- Shared across replicas
- Persistent (Redis with persistence)

### Layer 3 — CDN / Edge cache

For static assets, public APIs.

```
User → CloudFlare CDN → Origin server
        (cached)         (cache miss)
```

### Combined — Hybrid pattern

```csharp
public async Task<User?> GetAsync(int id)
{
    var key = $"user_{id}";
    
    // L1 — process memory
    if (_memoryCache.TryGetValue(key, out User? l1))
        return l1;
    
    // L2 — Redis
    var l2 = await _redis.GetStringAsync(key);
    if (l2 != null)
    {
        var user = JsonSerializer.Deserialize<User>(l2);
        _memoryCache.Set(key, user, TimeSpan.FromMinutes(1));  // populate L1
        return user;
    }
    
    // L3 — DB
    var fromDb = await _db.Users.FindAsync(id);
    if (fromDb != null)
    {
        await _redis.SetStringAsync(key, JsonSerializer.Serialize(fromDb));
        _memoryCache.Set(key, fromDb);
    }
    return fromDb;
}
```

`HybridCache` (.NET 9+) делает это automatically.

---

## 10. Common Pitfalls

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

## 11. Best Practices

- **Cache-aside default** — простой, resilient
- **TTL обязательно** — bounded staleness
- **Per-user keys** для user-specific data
- **Namespace prefixes** в shared cache (`MyApp:Users:1`)
- **HybridCache (.NET 9+)** для multi-tier
- **Output caching** для public APIs
- **Stampede protection** на high-traffic items
- **Graceful degradation** — если cache down, fallback to DB
- **Monitor hit rate** — < 80% = cache useless для эких
- **Profile cache effectiveness** — hit/miss ratio, latency
- **Tag-based invalidation** для grouped data
- **Pub/Sub invalidation** для distributed systems
- **Не cache mutable объекты** — clone или immutable

---

## 12. Когда что — flowchart

```
Need cache?
├── No → don't cache
└── Yes
    ├── Single instance app?
    │   └── IMemoryCache
    ├── Multi-instance app?
    │   ├── Need cross-instance? → Redis (IDistributedCache)
    │   ├── Want both? → HybridCache (.NET 9+)
    │   └── Per-instance OK? → IMemoryCache
    ├── HTTP responses cache?
    │   └── Output Caching middleware
    ├── Need stampede protection?
    │   └── HybridCache or Lock
    ├── Need cross-service invalidation?
    │   └── Pub/Sub (Redis) + manual invalidation
    └── Need tags?
        └── HybridCache (.NET 9+) или custom logic
```

---

## См. также

- [[performance-fundamentals|Performance Fundamentals]]
- [[../AspNetCore/caching|Caching в AspNetCore]]
- [[../SQL/postgresql-deep|PostgreSQL]]
- [[../Architecture/distributed-systems|Distributed Systems]]
- [[../Runtime/gc-memory|GC и память]]

## Reading list

- **Microsoft Docs — Caching** — learn.microsoft.com/aspnet/core/performance/caching
- **HybridCache docs** — learn.microsoft.com/aspnet/core/performance/caching/hybrid
- **Redis documentation** — redis.io/docs
- **Andrew Lock — Cache aside pattern** — andrewlock.net
- **Designing Data-Intensive Applications** — Kleppmann (chapter про caching)
