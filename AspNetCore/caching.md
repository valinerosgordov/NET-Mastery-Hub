---
tags: [aspnet, caching, hybrid-cache, redis, rate-limiting, output-cache, cdn]
level: Senior
---

# Caching и Rate Limiting

## Что это, зачем и когда

### Что такое кэширование?
Сохранение результата дорогой операции, чтобы **не выполнять её повторно**. Первый запрос — вычисляем и сохраняем. Следующие — отдаём из кэша.

**Аналогия:** Ты звонишь другу спросить номер пиццерии. Первый раз — он ищет, диктует. Ты **записываешь** (кэшируешь). Второй раз — смотришь в записях, не звонишь.

### Зачем?
- 100 пользователей запрашивают «популярные товары» → без кэша = 100 одинаковых SQL → БД перегружена
- С кэшем: первый запрос → SQL → 200мс → кэш на 5 мин → следующие 99 → из кэша → 1мс
- Снижение нагрузки на DB, internal services, third-party API
- Быстрее → лучше UX → больше конверсия (Amazon: 100ms latency = -1% revenue)

### Когда что?

| Тип кэша | Где хранит | Скорость | Когда |
|----------|-----------|----------|-------|
| **IMemoryCache** | В памяти приложения | Наносекунды | Один сервер, локальные данные |
| **IDistributedCache (Redis)** | Отдельный Redis | 1-5мс (сеть) | Несколько серверов, shared state |
| **HybridCache (.NET 9+)** | L1 memory + L2 Redis | Лучшее из обоих | Новые проекты — рекомендуется |
| **Output Cache** | HTTP response целиком | Мгновенно | GET endpoints без персонализации |
| **CDN** | Edge servers по миру | Мгновенно для users | Static assets, public API responses |
| **Browser cache** | На клиенте | 0 ms | Static files, immutable resources |

### Что такое Rate Limiting?
Ограничение количества запросов от одного клиента за период времени.

**Зачем:** защита от DDoS, ботов, злоупотребления API, защита downstream сервисов.

---

## Уровни кэширования (защита в глубину)

```
Browser Cache (клиент) — Cache-Control headers
    ↓ miss
CDN / Edge Cache — статика, public API
    ↓ miss
Reverse Proxy Cache (nginx, Varnish)
    ↓ miss
ASP.NET Output Cache — целые HTTP responses
    ↓ miss
Application Cache (HybridCache: L1 + L2)
    ↓ miss
Database (исходный источник)
```

Каждый уровень снимает нагрузку с следующего. **Главный принцип** — кэшируй как можно ближе к пользователю, инвалидируй как можно ближе к источнику изменений.

---

## HybridCache (.NET 9+) — рекомендуется

Новый встроенный API в `Microsoft.Extensions.Caching.Hybrid`. Объединяет L1 (memory) и L2 (Redis) с **автоматическим stampede protection**.

```bash
dotnet add package Microsoft.Extensions.Caching.Hybrid
```

```csharp
builder.Services.AddHybridCache(options =>
{
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(10),
        LocalCacheExpiration = TimeSpan.FromMinutes(2),
    };
});

// L2 — Redis
builder.Services.AddStackExchangeRedisCache(opts =>
{
    opts.Configuration = builder.Configuration.GetConnectionString("Redis");
    opts.InstanceName = "MyApp:";
});

// Service
public class ProductService(HybridCache cache, AppDbContext db)
{
    public async Task<Product?> GetAsync(int id, CancellationToken ct = default)
    {
        return await cache.GetOrCreateAsync(
            $"product:{id}",
            async cancellationToken =>
            {
                return await db.Products.FirstOrDefaultAsync(p => p.Id == id, cancellationToken);
            },
            tags: [$"product:{id}", "products"],
            cancellationToken: ct);
    }

    public async Task UpdateAsync(Product product, CancellationToken ct = default)
    {
        await db.SaveChangesAsync(ct);
        await cache.RemoveByTagAsync($"product:{product.Id}", ct);
    }
}
```

### Что HybridCache даёт из коробки

| Feature | Что |
|---------|-----|
| **L1 + L2 автоматически** | Сначала memory, miss → Redis, miss → factory |
| **Stampede protection** | 1000 параллельных запросов → 1 раз вызывается factory |
| **Tag-based invalidation** | `RemoveByTagAsync("products")` сбрасывает все продукты |
| **Type-safe** | Generic API без сериализации руками |
| **Cancellation propagation** | Через `CancellationToken` |

### Тонкости

```csharp
var options = new HybridCacheEntryOptions
{
    Expiration = TimeSpan.FromMinutes(10),       // Total TTL
    LocalCacheExpiration = TimeSpan.FromMinutes(2),  // L1 короче для свежести при invalidation
    Flags = HybridCacheEntryFlags.DisableLocalCache,  // Skip L1
};
```

**LocalCacheExpiration < Expiration** — потому что L1 живёт в каждом процессе и не получает invalidation мгновенно. Короткий L1 = меньше staleness.

> [!question]- **Интервью: HybridCache vs ручная связка IMemoryCache + IDistributedCache?**
> Раньше делали:
> ```csharp
> // Manual
> if (memCache.TryGetValue(key, out var value)) return value;
> var fromRedis = await distributedCache.GetAsync(key);
> if (fromRedis != null) { memCache.Set(key, fromRedis, ttl); return fromRedis; }
> // ... factory call, populate both layers
> ```
> Минусы:
> 1. **Stampede** — 100 параллельных GET'ов → 100 factory calls
> 2. **Сериализация** руками каждый раз
> 3. **Cache key conventions** размазаны по сервисам
>
> HybridCache решает всё это в одном API. После .NET 9 — стандартный выбор для новых проектов.

---

## IMemoryCache — детально

```csharp
builder.Services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024;  // В единицах "size" (определяешь сам)
    options.CompactionPercentage = 0.25;  // При достижении лимита удаляем 25% самых старых
});

public class ProductService(IMemoryCache cache, AppDbContext db)
{
    public async Task<Product?> GetAsync(int id)
    {
        return await cache.GetOrCreateAsync($"product:{id}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            entry.SlidingExpiration = TimeSpan.FromMinutes(2);
            entry.Size = 1;  // Учёт в SizeLimit
            entry.Priority = CacheItemPriority.High;  // Удаляется последним

            // Колбэк при удалении
            entry.RegisterPostEvictionCallback((key, value, reason, _) =>
            {
                _logger.LogInformation("Evicted {Key}: {Reason}", key, reason);
            });

            return await db.Products.FirstOrDefaultAsync(p => p.Id == id);
        });
    }
}
```

### Expiration policies

| | Что |
|--|-----|
| **AbsoluteExpiration** | Точное время истечения |
| **AbsoluteExpirationRelativeToNow** | Истечёт через X (от момента set) |
| **SlidingExpiration** | Сбрасывается на каждый access. "Если не используется 5 минут — удалить" |
| **Combined** | `Sliding=2min, Absolute=10min` → sliding до cap'а в 10 минут |

### CancellationChangeToken для invalidation

```csharp
var cts = new CancellationTokenSource();

cache.Set("products", products, new MemoryCacheEntryOptions()
    .AddExpirationToken(new CancellationChangeToken(cts.Token)));

// Invalidate всех "products" одной операцией
cts.Cancel();
```

---

## IDistributedCache (Redis) — детально

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "redis:6379,abortConnect=false,connectTimeout=5000,syncTimeout=5000";
    options.InstanceName = "MyApp:";  // Префикс ключей
});

public class ProductService(IDistributedCache cache, AppDbContext db)
{
    public async Task<Product?> GetAsync(int id, CancellationToken ct = default)
    {
        var json = await cache.GetStringAsync($"product:{id}", ct);
        if (json is not null) return JsonSerializer.Deserialize<Product>(json);

        var product = await db.Products.FirstOrDefaultAsync(p => p.Id == id, ct);
        if (product is null) return null;

        await cache.SetStringAsync(
            $"product:{id}",
            JsonSerializer.Serialize(product),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
            },
            ct);

        return product;
    }

    public async Task InvalidateAsync(int id, CancellationToken ct = default)
    {
        await cache.RemoveAsync($"product:{id}", ct);
    }
}
```

### Прямой ConnectionMultiplexer для advanced

```csharp
builder.Services.AddSingleton<IConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect("redis:6379"));

public class AdvancedCache(IConnectionMultiplexer redis)
{
    public async Task IncrementAsync(string key, long by = 1)
    {
        var db = redis.GetDatabase();
        await db.StringIncrementAsync(key, by);
    }

    public async Task PublishAsync(string channel, string message)
    {
        var sub = redis.GetSubscriber();
        await sub.PublishAsync(RedisChannel.Literal(channel), message);
    }
}
```

`IDistributedCache` — упрощённый API. `ConnectionMultiplexer` — полная мощь Redis (см. подробно в [Redis Advanced](../Infrastructure/redis-advanced.md)).

---

## Cache Patterns

### 1. Cache-Aside (Lazy Loading)

```csharp
public async Task<T?> GetAsync<T>(string key, Func<Task<T>> factory)
{
    var cached = await cache.GetAsync<T>(key);
    if (cached is not null) return cached;

    var fresh = await factory();
    if (fresh is not null) await cache.SetAsync(key, fresh, ttl);
    return fresh;
}
```

**Самый частый**. Простой, robust. Минус — на cache miss первый запрос медленный.

### 2. Write-Through

При записи — пишем И в cache И в DB. Cache всегда synced.

```csharp
public async Task UpdateAsync(Product p)
{
    await db.SaveChangesAsync();
    await cache.SetAsync($"product:{p.Id}", p);
}
```

Pros: всегда свежий cache. Cons: 2 hops на каждую запись.

### 3. Write-Behind (Write-Back)

Пишем в cache, eventually flush в DB. Ну то есть **рискованно**.

```csharp
public async Task UpdateAsync(Product p)
{
    await cache.SetAsync($"product:{p.Id}", p);
    await queue.PublishAsync(new ProductUpdated(p));  // Async flush
}
```

Pros: fast writes. Cons: cache crash = data loss. Использовать только для non-critical (analytics counters).

### 4. Read-Through

Cache как proxy перед DB. Cache знает как load из source.

```csharp
public class CachedProductRepository(IMemoryCache cache, AppDbContext db) : IProductRepository
{
    public Task<Product?> GetAsync(int id) =>
        cache.GetOrCreateAsync($"product:{id}", async _ =>
            await db.Products.FirstOrDefaultAsync(p => p.Id == id));
}
```

Decorator pattern. App не знает что кэширует.

> [!question]- **Интервью: какие проблемы у write-back cache?**
> 1. **Data loss при сбое** — между write в cache и flush в DB. Если кэш упал — потеряно.
> 2. **Out-of-sync чтения с другого instance** — другой сервис читает напрямую из DB → видит старые данные.
> 3. **Сложность recovery** — как restore консистентность?
> Пример где OK: счётчики просмотров. Если потеряли пару увеличений — не критично, бизнес не пострадает. Где **не OK** — финансы, заказы, любые transactional данные.

---

## Cache Stampede / Thundering Herd

**Проблема:** TTL истёк, 1000 параллельных GET'ов — все идут в DB. БД падает.

### 1. Lock через distributed mutex

```csharp
public async Task<T> GetOrComputeWithLockAsync<T>(
    string key, Func<Task<T>> factory, TimeSpan ttl, CancellationToken ct)
{
    var cached = await cache.GetAsync<T>(key);
    if (cached is not null) return cached;

    using var redLock = await lockFactory.CreateLockAsync(
        $"lock:{key}", TimeSpan.FromSeconds(10), ct);

    if (redLock.IsAcquired)
    {
        // Double-check после получения lock
        cached = await cache.GetAsync<T>(key);
        if (cached is not null) return cached;

        var fresh = await factory();
        await cache.SetAsync(key, fresh, ttl);
        return fresh;
    }

    // Не получили lock — другой instance computes; ждём и читаем
    await Task.Delay(50, ct);
    return await GetOrComputeWithLockAsync(key, factory, ttl, ct);
}
```

Просто, но добавляет round-trip к Redis при каждом cache miss.

### 2. Probabilistic early refresh

Refresh заранее с малой вероятностью — distribute load.

```csharp
public async Task<T> GetWithEarlyRefreshAsync<T>(
    string key, Func<Task<T>> factory, TimeSpan ttl, CancellationToken ct)
{
    var (value, expiresAt) = await cache.GetWithExpiryAsync<T>(key, ct);
    if (value is null) return await ComputeAndCacheAsync(key, factory, ttl, ct);

    var remaining = expiresAt - DateTime.UtcNow;
    var refreshThreshold = ttl * 0.1;  // 10% от TTL до истечения

    if (remaining < refreshThreshold)
    {
        // Random chance: один из ~10 запросов делает refresh
        if (Random.Shared.NextDouble() < 0.1)
            _ = ComputeAndCacheAsync(key, factory, ttl, ct);  // background
    }

    return value;
}
```

Без round-trips к Redis lock'у. Но если плохо повезло — несколько одновременных refreshes возможны.

### 3. SingleFlight (in-process)

Внутри одного process — `Lazy<Task<T>>` гарантирует один factory call:

```csharp
private readonly ConcurrentDictionary<string, Lazy<Task<object>>> _inFlight = new();

public async Task<T> GetOrComputeAsync<T>(string key, Func<Task<T>> factory)
{
    var lazy = _inFlight.GetOrAdd(key, _ => new Lazy<Task<object>>(async () =>
    {
        try { return (await factory())!; }
        finally { _inFlight.TryRemove(key, out _); }
    }));

    return (T)await lazy.Value;
}
```

Combined with HybridCache — она это делает встроенно.

### 4. HybridCache solves it for you

Просто используй HybridCache — stampede protection встроен.

---

## Output Caching (.NET 7+)

Кэширование полного HTTP-ответа. Повторные запросы получают response без выполнения handler'а.

```csharp
builder.Services.AddOutputCache(opts =>
{
    opts.AddBasePolicy(b => b.Expire(TimeSpan.FromMinutes(5)));

    opts.AddPolicy("ProductList", b => b
        .Expire(TimeSpan.FromMinutes(10))
        .Tag("products")
        .SetVaryByQuery("page", "sort"));

    opts.AddPolicy("UserSpecific", b => b
        .Expire(TimeSpan.FromMinutes(2))
        .Tag("users")
        .VaryByValue(http => http.User.FindFirstValue("sub") ?? ""));
});

app.UseOutputCache();

app.MapGet("/api/products", GetProducts).CacheOutput("ProductList");

app.MapPost("/api/products", async (IOutputCacheStore store, ...) =>
{
    // создание продукта
    await store.EvictByTagAsync("products", default);
});
```

### Когда Output Cache НЕ работает

- Запросы с `Authorization` header (по умолчанию — отключено)
- POST/PUT/DELETE запросы
- Ответы с `Set-Cookie`
- Ответы со статусом != 200

Для authenticated endpoints — `b.AllowCachingAuthenticated()` + `VaryByValue` per-user.

---

## CDN + Cache-Control headers

Для static assets и public API responses — пусть кэширует CDN, не твой сервис.

```csharp
app.MapGet("/api/categories", () => GetCategories())
   .WithMetadata(new ResponseCacheAttribute
   {
       Duration = 3600,           // 1 час
       Location = ResponseCacheLocation.Any,  // CDN + browser
       VaryByQueryKeys = ["lang"]
   });

// Через middleware
app.Use(async (ctx, next) =>
{
    if (ctx.Request.Path.StartsWithSegments("/static"))
    {
        ctx.Response.Headers.CacheControl = "public, max-age=31536000, immutable";
    }
    await next();
});
```

### Cache-Control directives

| Directive | Что |
|-----------|-----|
| `public` | Можно кэшировать proxy/CDN/browser |
| `private` | Только browser, не shared cache |
| `max-age=N` | TTL в секундах |
| `s-maxage=N` | TTL только для shared cache (CDN) |
| `no-cache` | Кэшируй, но всегда revalidate (через ETag) |
| `no-store` | Не кэшируй вообще |
| `must-revalidate` | После expiration обязательная revalidation |
| `immutable` | Не изменится никогда (versioned URLs) |
| `stale-while-revalidate=N` | Можешь отдать stale, в фоне обновляй |

### ETag + If-None-Match

Conditional GET — если контент не менялся, возвращаем `304 Not Modified` без body.

```csharp
app.MapGet("/api/feed", async (HttpContext ctx, IFeedService svc) =>
{
    var lastModified = await svc.GetLastModifiedAsync();
    var etag = $"W/\"{lastModified.Ticks}\"";

    if (ctx.Request.Headers.IfNoneMatch == etag)
    {
        return Results.StatusCode(304);
    }

    ctx.Response.Headers.ETag = etag;
    ctx.Response.Headers.LastModified = lastModified.ToString("R");

    var feed = await svc.GetFeedAsync();
    return Results.Ok(feed);
});
```

CDN автоматически выполняет revalidation через ETag.

---

## Cache invalidation — самая сложная часть

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

### 1. TTL only

Самый простой — кэш живёт N минут, потом перезапрашивается. Stale data до конца TTL.

### 2. Explicit invalidation на write

```csharp
public async Task UpdateAsync(Product p)
{
    await db.SaveChangesAsync();
    await cache.RemoveAsync($"product:{p.Id}");
    await cache.RemoveByTagAsync("products");  // Все list views
}
```

Работает в одном instance. В multi-instance — нужен pub/sub.

### 3. Pub/Sub для multi-instance

```csharp
public class CacheInvalidator(
    IMemoryCache localCache,
    IConnectionMultiplexer redis)
{
    public async Task StartAsync(CancellationToken ct)
    {
        var sub = redis.GetSubscriber();
        await sub.SubscribeAsync(RedisChannel.Literal("invalidations"), (channel, message) =>
        {
            localCache.Remove(message!);
            _logger.LogInformation("Local cache invalidated for {Key}", message);
        });
    }

    public async Task InvalidateAsync(string key)
    {
        var sub = redis.GetSubscriber();
        await sub.PublishAsync(RedisChannel.Literal("invalidations"), key);
    }
}
```

Все instances подписаны → каждый invalidate свой L1.

### 4. Tag-based invalidation

```csharp
// Insert с tags
await cache.SetAsync("product:1", product, tags: ["product:1", "products", "category:5"]);

// Invalidate all products
await cache.RemoveByTagAsync("products");

// Invalidate by category
await cache.RemoveByTagAsync("category:5");
```

HybridCache + `IOutputCacheStore` поддерживают tags из коробки.

### 5. Versioned cache keys

```csharp
public async Task<Product> GetAsync(int id)
{
    var version = await cache.GetStringAsync($"version:products:{id}") ?? "1";
    return await cache.GetOrCreateAsync($"product:{id}:v{version}", ...);
}

public async Task UpdateAsync(Product p)
{
    await db.SaveChangesAsync();
    await cache.IncrementAsync($"version:products:{p.Id}");
    // Старый ключ "product:1:v1" останется в Redis до TTL — но никто его не запрашивает
}
```

Не нужно явно удалять старое — TTL вычистит. **Но** — место в Redis тратится.

### Когда что выбирать

| Pattern | Когда |
|---------|-------|
| **TTL only** | Допустимо stale ~ TTL/2 в среднем (новости, статистика) |
| **Explicit + Pub/Sub** | Multi-instance, важна свежесть после write |
| **Tag-based** | Сложные группы инвалидации (все продукты категории) |
| **Versioned keys** | High write throughput, не хочется invalidation latency |

---

## Cache как защита downstream

Полезный pattern: cache stale data при сбое downstream.

```csharp
public async Task<Product?> GetResilientAsync(int id, CancellationToken ct = default)
{
    try
    {
        var fresh = await downstreamApi.GetAsync(id, ct);
        if (fresh is not null)
        {
            await cache.SetAsync($"product:{id}", fresh, TimeSpan.FromHours(24), ct);
            return fresh;
        }
    }
    catch (Exception ex) when (ex is HttpRequestException or TimeoutException)
    {
        _logger.LogWarning(ex, "Downstream failed, falling back to cache");
    }

    // Fallback на возможно stale value
    return await cache.GetAsync<Product>($"product:{id}", ct);
}
```

Для **circuit breaker pattern** см. [Resilience](resilience.md).

---

## Rate Limiting (.NET 7+)

Built-in middleware. До этого — пакеты типа `AspNetCoreRateLimit`.

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = 429;

    // Fixed window
    options.AddFixedWindowLimiter("api", opt =>
    {
        opt.Window = TimeSpan.FromMinutes(1);
        opt.PermitLimit = 100;
        opt.QueueLimit = 0;
    });

    // Token bucket — smooth burst
    options.AddTokenBucketLimiter("burst", opt =>
    {
        opt.TokenLimit = 100;
        opt.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        opt.QueueLimit = 0;
        opt.ReplenishmentPeriod = TimeSpan.FromSeconds(1);
        opt.TokensPerPeriod = 10;
        opt.AutoReplenishment = true;
    });

    // Per-user
    options.AddPolicy("per-user", context =>
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);
        return RateLimitPartition.GetTokenBucketLimiter(
            partitionKey: userId ?? "anonymous",
            factory: _ => new TokenBucketRateLimiterOptions
            {
                TokenLimit = 100,
                ReplenishmentPeriod = TimeSpan.FromSeconds(1),
                TokensPerPeriod = 10,
            });
    });

    // Concurrency limiter — макс N параллельных
    options.AddConcurrencyLimiter("heavy", opt =>
    {
        opt.PermitLimit = 5;
        opt.QueueLimit = 100;
    });

    options.OnRejected = async (ctx, ct) =>
    {
        if (ctx.Lease.TryGetMetadata(MetadataName.RetryAfter, out var retryAfter))
        {
            ctx.HttpContext.Response.Headers.RetryAfter = retryAfter.TotalSeconds.ToString("0");
        }
        ctx.HttpContext.Response.StatusCode = 429;
        await ctx.HttpContext.Response.WriteAsync("Too many requests. Try later.", ct);
    };
});

app.UseRateLimiter();

app.MapGet("/api/data", () => "...").RequireRateLimiting("api");
app.MapPost("/api/heavy-task", () => "...").RequireRateLimiting("heavy");
```

### Algorithms

| | Fixed window | Sliding window | Token bucket | Concurrency |
|--|--------------|----------------|--------------|-------------|
| Что | N в фиксированном окне | N в скользящем окне | Токены пополняются | N одновременных |
| Burst | Большой на стыке окон | Сглажен | Bucket size | По limit |
| Use case | Простой | Точный | Most APIs | Resource-heavy ops |

**Для public API:** token bucket. Smooth.

### Distributed rate limiting

In-memory rate limiter — per-instance. За LB user может получить N × InstanceCount запросов. Для distributed:
- Redis + Lua-скрипт (см. [System Design](../Architecture/system-design.md))
- Гейт перед API (nginx limit_req, Cloudflare, AWS API Gateway)

См. полный пример Lua-скрипта в [system-design.md / Rate Limiter](../Architecture/system-design.md#шаблон-1--rate-limiter).

---

## Cache warm-up

После deploy все кэши пустые → первые запросы медленные. Решения:

### 1. BackgroundService прогревает популярное

```csharp
public class CacheWarmupService(IServiceProvider sp, ILogger<CacheWarmupService> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await Task.Delay(TimeSpan.FromSeconds(5), ct);  // дать app стартовать

        using var scope = sp.CreateScope();
        var products = scope.ServiceProvider.GetRequiredService<IProductService>();

        log.LogInformation("Warming up product cache");
        var top = await products.GetTopSellingIdsAsync(top: 100, ct);
        await Parallel.ForEachAsync(top, ct, async (id, c) =>
        {
            await products.GetAsync(id, c);  // Заполнит cache
        });
        log.LogInformation("Cache warm-up done");
    }
}

builder.Services.AddHostedService<CacheWarmupService>();
```

### 2. Snapshot из предыдущего instance

Перед graceful shutdown сохраняем хот-keys в Redis с длинным TTL → новый instance читает готовое.

### 3. Blue-green deployment

Старый instance ещё живёт когда новый прогревается. Switchover после warm-up.

---

## Production checklist

- [ ] HybridCache для новых проектов (.NET 9+)
- [ ] L2 = Redis cluster (не single instance)
- [ ] Stampede protection (HybridCache built-in или distributed lock)
- [ ] TTL разумный — не слишком короткий (нагрузка на DB), не слишком длинный (stale)
- [ ] Tag-based invalidation для групп объектов
- [ ] Pub/Sub invalidation для multi-instance L1
- [ ] CDN на уровне статики и public API
- [ ] Cache-Control + ETag headers для conditional requests
- [ ] Output Cache для GET endpoints без персонализации
- [ ] Rate limiting на public endpoints
- [ ] Distributed rate limiting если N инстансов
- [ ] Cache warm-up service после deploy
- [ ] Resilient cache (fallback на stale при downstream failure)
- [ ] Monitoring: hit ratio, miss ratio, eviction count, memory usage

---

## Common pitfalls

### 1. Кэшировать всё подряд
"Кэш сделает быстрее" — без анализа hit rate. На low-hit ratio (< 50%) кэш только добавляет overhead.

### 2. Слишком большой TTL
Stale data → пользователи видят старую цену, старые остатки. UX/business issues.

### 3. Слишком короткий TTL
Каждые 30 секунд cache miss → DB перегружена. Особенно вместе со stampede.

### 4. Cache key collisions
`cache.Get($"user:{id}")` без префикса для tenant → один tenant видит данные другого.
**Решение:** `cache.Get($"tenant:{tenantId}:user:{id}")`.

### 5. Сериализация медленнее самого DB-запроса
В Redis кладёшь огромный объект (100KB JSON), serialize/deserialize медленнее SQL. Проверь BenchmarkDotNet'ом.

### 6. Memory leak через подписки на CancellationToken

```csharp
// ❌
options.AddExpirationToken(new CancellationChangeToken(longLivedCts.Token));
// При тысячах cache entries — тысячи registrations
```
**Решение:** короткие cts на каждую категорию данных.

### 7. Race condition: write to DB → invalidate cache
Между `SaveChanges()` и `Remove()` другой запрос успел положить старое значение в cache.
**Решение:** invalidate ДО write (set sentinel "stale") + invalidate после. Или версионирование.

### 8. Thundering herd на популярные ключи
Один hot key → один Redis-shard перегружен.
**Решение:** local L1 cache (HybridCache), sharding на N подключей, request coalescing.

### 9. Cache-Control: no-cache принимают за no-store
`no-cache` — кэшируй но revalidate. `no-store` — не кэшируй вообще. Путают.

### 10. Output cache + Authentication
Output cache по умолчанию **не** работает с `Authorization` header. Если нужен — `AllowCachingAuthenticated()` + `VaryByValue` per-user.

---

## См. также

- [Resilience](resilience.md) — Polly + cache fallback при downstream failure
- [System Design](../Architecture/system-design.md) — distributed cache patterns, rate limiter, hot key problem
- [Redis Advanced](../Infrastructure/redis-advanced.md) — pub/sub, streams, Lua, cluster — глубоко
- [Performance](../Performance/performance.md) — измерение cache hit ratio, профилирование
- [PostgreSQL Deep](../SQL/postgresql-deep.md) — чем хороши БД-индексы (часто заменяют cache)
- [Distributed Systems](../Architecture/distributed-systems.md) — eventual consistency и cache как read model

## Reading list

- **Microsoft Learn — HybridCache** — learn.microsoft.com/aspnet/core/performance/caching/hybrid
- **Microsoft Learn — Output Caching** — learn.microsoft.com/aspnet/core/performance/caching/output
- **Microsoft Learn — Rate Limiting** — learn.microsoft.com/aspnet/core/performance/rate-limit
- **Redis docs — caching patterns** — redis.io/docs/manual/patterns/
- **High-Scalability** — highscalability.com (case studies cache-стратегий крупных компаний)
- **Marc Brooker — TIme keep slipping** — brooker.co.za (про cache invalidation в распределённых системах)
- **Stripe Engineering — Online Migrations** — stripe.com/blog/online-migrations (cache invalidation в production)
