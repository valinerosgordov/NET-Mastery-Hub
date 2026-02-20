# Caching и Rate Limiting

## Output Caching (.NET 7+)

Кэширование полного HTTP-ответа на стороне сервера. Повторные запросы к тому же endpoint получают ответ из кэша без выполнения handler'а.

```csharp
builder.Services.AddOutputCache(opts =>
{
    opts.AddBasePolicy(b => b.Expire(TimeSpan.FromMinutes(5)));

    opts.AddPolicy("ProductList", b => b
        .Expire(TimeSpan.FromMinutes(10))
        .Tag("products")
        .SetVaryByQuery("page", "sort"));
});

app.UseOutputCache(); // Между Routing и Endpoints

// Применение
app.MapGet("/api/products", GetProducts).CacheOutput("ProductList");

// Инвалидация по тегу
app.MapPost("/api/products", async (IOutputCacheStore store, ...) =>
{
    // ... создание продукта
    await store.EvictByTagAsync("products", default); // Сбросить все кэшированные ответы с тегом
});
```

### Когда Output Cache НЕ работает

- Запросы с `Authorization` header (по умолчанию)
- POST/PUT/DELETE запросы
- Ответы с `Set-Cookie`
- Ответы со статусом != 200

---

## IMemoryCache vs IDistributedCache

| Аспект | IMemoryCache | IDistributedCache |
|--------|--------------|-------------------|
| Хранение | In-process memory | Redis, SQL Server, NCache |
| Масштабирование | Один инстанс | Общий для всех инстансов |
| Сериализация | Нет (объекты as-is) | `byte[]` (нужна сериализация) |
| Скорость | Очень быстро (~наносекунды) | Сетевой вызов (~миллисекунды) |
| Потеря при рестарте | Да | Нет (Redis persistent) |

### IMemoryCache

```csharp
builder.Services.AddMemoryCache();

public class ProductService
{
    private readonly IMemoryCache _cache;

    public async Task<Product?> GetByIdAsync(int id)
    {
        return await _cache.GetOrCreateAsync($"product:{id}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
            entry.SlidingExpiration = TimeSpan.FromMinutes(2);
            entry.Size = 1; // Если настроен SizeLimit
            entry.Priority = CacheItemPriority.High;

            return await _repo.GetByIdAsync(id);
        });
    }
}
```

**Типы expiration**:
- `AbsoluteExpiration` — точное время удаления
- `AbsoluteExpirationRelativeToNow` — через N минут после создания записи
- `SlidingExpiration` — продлевается при каждом обращении (но не дольше AbsoluteExpiration)

### IDistributedCache

```csharp
builder.Services.AddStackExchangeRedisCache(opts =>
{
    opts.Configuration = "localhost:6379";
    opts.InstanceName = "myapp:";
});

public class CatalogService
{
    private readonly IDistributedCache _cache;

    public async Task<List<Category>> GetCategoriesAsync()
    {
        var cached = await _cache.GetStringAsync("categories");
        if (cached is not null)
            return JsonSerializer.Deserialize<List<Category>>(cached)!;

        var categories = await _repo.GetAllAsync();

        await _cache.SetStringAsync("categories",
            JsonSerializer.Serialize(categories),
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1)
            });

        return categories;
    }
}
```

### HybridCache (.NET 9)

Двухуровневый кэш: L1 (in-memory) + L2 (distributed). Снижает латентность за счёт L1 и обеспечивает консистентность через L2.

```csharp
builder.Services.AddHybridCache(opts =>
{
    opts.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(30),
        LocalCacheExpiration = TimeSpan.FromMinutes(5)
    };
});

public class ProductService
{
    private readonly HybridCache _cache;

    public async Task<Product> GetByIdAsync(int id, CancellationToken ct)
    {
        return await _cache.GetOrCreateAsync(
            $"product:{id}",
            async token => await _repo.GetByIdAsync(id, token),
            cancellationToken: ct);
    }
}
```

Преимущества HybridCache:
- Автоматическая **stampede prevention** (одновременные запросы ждут один фабричный вызов)
- Прозрачная сериализация
- Простой API — один метод вместо get + check + set

---

## Паттерны кэширования

### Cache-Aside (Lazy Loading)

Приложение само управляет кэшем: проверяет, есть ли данные → если нет, загружает и кладёт в кэш.

```csharp
var data = await cache.GetAsync(key);
if (data is null)
{
    data = await db.LoadAsync();
    await cache.SetAsync(key, data, options);
}
return data;
```

Самый распространённый паттерн. Недостаток: первый запрос всегда медленный (cache miss).

### Read-Through / Write-Through

Кэш сам обращается к источнику данных. Приложение работает только с кэшем.
- **Read-Through**: кэш загружает данные при cache miss
- **Write-Through**: кэш записывает в БД синхронно при обновлении

### Write-Behind (Write-Back)

Кэш записывает в БД **асинхронно** (с задержкой). Быстрее, но риск потери данных.

### Stampede Prevention

Проблема: множество запросов одновременно обнаруживают cache miss и все идут в БД.

Решения:
- **Lock/Semaphore**: только один запрос загружает данные, остальные ждут
- `HybridCache` решает это автоматически
- `SemaphoreSlim` + проверка кэша после получения lock

```csharp
private static readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task<Product> GetWithStampedeProtection(int id)
{
    var cached = _cache.Get<Product>($"product:{id}");
    if (cached is not null) return cached;

    await _semaphore.WaitAsync();
    try
    {
        // Повторная проверка — возможно, другой поток уже загрузил
        cached = _cache.Get<Product>($"product:{id}");
        if (cached is not null) return cached;

        var product = await _repo.GetByIdAsync(id);
        _cache.Set($"product:{id}", product, TimeSpan.FromMinutes(10));
        return product;
    }
    finally
    {
        _semaphore.Release();
    }
}
```

---

## Rate Limiting (.NET 7+)

Защита API от злоупотреблений, DDoS, обеспечение fair use.

```csharp
builder.Services.AddRateLimiter(opts =>
{
    opts.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    // Fixed Window — N запросов за фиксированное окно
    opts.AddFixedWindowLimiter("fixed", o =>
    {
        o.PermitLimit = 100;
        o.Window = TimeSpan.FromMinutes(1);
        o.QueueLimit = 10;
        o.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });

    // Sliding Window — скользящее окно, более плавное ограничение
    opts.AddSlidingWindowLimiter("sliding", o =>
    {
        o.PermitLimit = 100;
        o.Window = TimeSpan.FromMinutes(1);
        o.SegmentsPerWindow = 6; // 6 сегментов по 10 секунд
    });

    // Token Bucket — пополняемое ведро токенов (burst-friendly)
    opts.AddTokenBucketLimiter("token", o =>
    {
        o.TokenLimit = 100;          // Максимум токенов
        o.ReplenishmentPeriod = TimeSpan.FromSeconds(10);
        o.TokensPerPeriod = 20;      // Добавляется 20 каждые 10 сек
        o.AutoReplenishment = true;
    });

    // Concurrency — ограничение одновременных запросов
    opts.AddConcurrencyLimiter("concurrent", o =>
    {
        o.PermitLimit = 10;
        o.QueueLimit = 5;
    });
});

app.UseRateLimiter();

// Применение к endpoint'ам
app.MapGet("/api/data", GetData).RequireRateLimiting("fixed");
```

### Тонкости Rate Limiting

- По умолчанию ограничение **глобальное**. Для per-user/per-IP — используйте `PartitionedRateLimiter` с `RateLimitPartition.GetFixedWindowLimiter(httpContext.User.Identity?.Name, ...)`
- `QueueLimit > 0` — запросы сверх лимита ставятся в очередь вместо немедленного отказа
- В distributed-системах (несколько инстансов) встроенный rate limiter **не работает** — используйте Redis-based решения
- Для API Gateway — rate limiting лучше на уровне gateway (YARP, nginx, API Management)
