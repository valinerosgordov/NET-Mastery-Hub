---
tags: [aspnetcore, dependency-injection, lifetimes, keyed-services, middle, di]
level: Middle
date: 2026-05-10
---

# ASP.NET Core DI Deep — lifetimes, scopes, keyed services, captive deps

> **DI lifetimes deep, IServiceScopeFactory, keyed services (.NET 8+), TimeProvider, captive dependencies, async DI.** Closes пробел между Junior basics и Senior di-configuration.

---

## 0. Как читать

После Junior знакомства с DI. Перед `Senior/di-configuration.md`. Здесь — production patterns: lifetimes pitfalls, scope factory, keyed services .NET 8, async DI patterns.

---

## 1. Service Lifetimes — recap + nuances

### 1.1. Три lifetimes

```csharp
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddTransient<IDateProvider, SystemDateProvider>();
```

### 1.2. Singleton

```
- ОДИН instance на ВСЁ приложение
- Создаётся при первом запросе
- Живёт до завершения app
- Shared между ВСЕМИ requests + threads

Use cases:
✅ Stateless services (formatters, calculators)
✅ Caches, configuration
✅ HttpClient (через IHttpClientFactory!)
✅ Loggers
✅ Performance-critical (avoid construction overhead)

❌ Не use:
- Per-request state
- DbContext (НИКОГДА!)
- Anything that uses scoped dependencies
- Stateful services с user-specific data
```

```csharp
public class GlobalCacheService : IGlobalCacheService
{
    private readonly ConcurrentDictionary<string, object> _cache = new();
    
    // Thread-safe — используем concurrent collections
    public T? Get<T>(string key) => _cache.TryGetValue(key, out var v) ? (T)v : default;
    public void Set<T>(string key, T value) => _cache[key] = value!;
}

builder.Services.AddSingleton<IGlobalCacheService, GlobalCacheService>();
```

### 1.3. Scoped

```
- ОДИН instance на HTTP request (или scope)
- Создаётся при start request
- Disposed при end request
- Shared между всеми сервисами одного request

Use cases:
✅ DbContext (стандарт!)
✅ Repositories
✅ UnitOfWork
✅ Per-request state (current user, tenant)
✅ Request-scoped logger context
```

```csharp
public class UserService : IUserService
{
    private readonly AppDbContext _db;   // scoped
    private readonly ICurrentUser _user;  // scoped
    
    public UserService(AppDbContext db, ICurrentUser user)
    {
        _db = db;
        _user = user;
    }
}

builder.Services.AddDbContext<AppDbContext>(...);   // implicitly scoped
builder.Services.AddScoped<IUserService, UserService>();
```

### 1.4. Transient

```
- НОВЫЙ instance каждый раз когда запрашивается
- Disposed при end of containing scope
- Никогда не shared между запросами

Use cases:
✅ Lightweight stateless services
✅ Date/time providers
✅ Validators (если используют request-scoped data)

❌ Не use:
- Heavy construction (creates many instances)
- Anything с meaningful state
```

### 1.5. Decision rule

```
Singleton:
- Stateless / immutable state
- Thread-safe
- Heavy to construct
- Shared resources

Scoped:
- Per-request lifecycle
- Uses DbContext / scoped deps
- Default choice если не уверен!

Transient:
- Lightweight
- Stateless
- Default для simple helpers
```

> [!question]- **Интервью: lifetimes — что когда?**
> **Singleton** — один instance на app lifetime. Stateless services, caches, config. Threadsafe обязательно. **Scoped** — один на HTTP request. DbContext, repositories, request state. **Transient** — новый каждый раз. Lightweight stateless helpers. **Default rule**: если не уверен — Scoped (safest, most common). **Container detection**: ASP.NET Core checks scope при resolving — throws если Singleton зависит от Scoped (captive dependency).

---

## 2. Captive Dependencies — главная проблема

### 2.1. Что это

Singleton который **держит scoped dependency**. Scoped живёт дольше своего scope → memory leak / wrong state.

```csharp
public class CacheService   // Singleton
{
    private readonly AppDbContext _db;   // ❌ Scoped!
    
    public CacheService(AppDbContext db)
    {
        _db = db;
    }
}

builder.Services.AddSingleton<CacheService>();
builder.Services.AddDbContext<AppDbContext>();   // scoped
```

Что произойдёт:
1. App start → CacheService создаётся (singleton)
2. CacheService держит DbContext (scoped) — но scope первого request!
3. После request scope закрывается → DbContext disposed
4. CacheService продолжает использовать **disposed DbContext** → ObjectDisposedException

### 2.2. Detection

ASP.NET Core (Development environment) детектирует captive dependencies:

```csharp
// Program.cs — explicit validation
builder.Services.BuildServiceProvider(new ServiceProviderOptions
{
    ValidateScopes = true,    // detect captive deps
    ValidateOnBuild = true     // detect on app start
});

// В Development это уже включено по default
```

Запуск даст:

```
InvalidOperationException: Cannot consume scoped service 'AppDbContext' 
from singleton 'CacheService'
```

### 2.3. Решение — IServiceScopeFactory

Singleton не должен держать scoped service напрямую. Используй scope factory для создания scope on-demand:

```csharp
public class CacheService   // Singleton
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public CacheService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }
    
    public async Task RefreshAsync()
    {
        // Создаём scope on-demand
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        
        var data = await db.SomeTable.ToListAsync();
        // Process data...
        
    }   // scope disposed → DbContext disposed
}

builder.Services.AddSingleton<CacheService>();
```

### 2.4. Background Services pattern

`BackgroundService` (HostedService) is **Singleton** by default. Same pitfall:

```csharp
public class OrderProcessor : BackgroundService
{
    private readonly AppDbContext _db;   // ❌ NEVER!
    
    public OrderProcessor(AppDbContext db) => _db = db;
}
```

**Fix**: IServiceScopeFactory:

```csharp
public class OrderProcessor : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    
    public OrderProcessor(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using (var scope = _scopeFactory.CreateScope())
            {
                var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
                await ProcessOrdersAsync(db, stoppingToken);
            }
            
            await Task.Delay(TimeSpan.FromSeconds(30), stoppingToken);
        }
    }
}
```

### 2.5. Async scope (.NET 8+)

```csharp
await using var scope = _scopeFactory.CreateAsyncScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
// ...
// При dispose — async cleanup (DbContext.DisposeAsync)
```

`CreateAsyncScope` invokes `IAsyncDisposable.DisposeAsync` properly.

> [!question]- **Интервью: captive dependency что это?**
> Когда сервис **долгого lifetime** (Singleton) держит ссылку на сервис **более короткого lifetime** (Scoped). Scoped service выживает дольше своего natural scope → memory leak, ObjectDisposedException, wrong state. **Detection**: ASP.NET Core с `ValidateScopes = true` (default в Development) бросает на startup. **Fix**: использовать `IServiceScopeFactory.CreateScope()` чтобы create scope on-demand, resolve scoped services внутри scope, dispose. **Critical для**: BackgroundService (singleton), HostedService, long-running tasks.

---

## 3. Keyed Services (.NET 8+)

### 3.1. Зачем

Раньше — multiple реализации одного interface не было встроенно:

```csharp
// До .NET 8 — manual registry
public interface INotificationService
{
    Task SendAsync(string message);
}

builder.Services.AddScoped<EmailNotificationService>();
builder.Services.AddScoped<SmsNotificationService>();
builder.Services.AddScoped<PushNotificationService>();

// В коде нужен factory:
public class NotificationFactory
{
    private readonly IServiceProvider _provider;
    
    public INotificationService Create(string type) => type switch
    {
        "email" => _provider.GetRequiredService<EmailNotificationService>(),
        "sms" => _provider.GetRequiredService<SmsNotificationService>(),
        _ => throw new ArgumentException()
    };
}
```

### 3.2. .NET 8+ — keyed services

```csharp
builder.Services.AddKeyedScoped<INotificationService, EmailNotificationService>("email");
builder.Services.AddKeyedScoped<INotificationService, SmsNotificationService>("sms");
builder.Services.AddKeyedScoped<INotificationService, PushNotificationService>("push");
```

Inject специфичную реализацию:

```csharp
public class OrderService
{
    private readonly INotificationService _email;
    private readonly INotificationService _sms;
    
    public OrderService(
        [FromKeyedServices("email")] INotificationService email,
        [FromKeyedServices("sms")] INotificationService sms)
    {
        _email = email;
        _sms = sms;
    }
    
    public async Task NotifyOrderShipped(Order order)
    {
        await _email.SendAsync($"Order {order.Id} shipped");
        await _sms.SendAsync($"Order {order.Id} shipped");
    }
}
```

### 3.3. Resolve programmatically

```csharp
public class NotificationDispatcher
{
    private readonly IServiceProvider _provider;
    
    public NotificationDispatcher(IServiceProvider provider) => _provider = provider;
    
    public async Task DispatchAsync(string type, string message)
    {
        var service = _provider.GetRequiredKeyedService<INotificationService>(type);
        await service.SendAsync(message);
    }
}
```

### 3.4. Use cases

```
✅ Multiple implementations:
- Storage: S3, Azure Blob, Local
- Cache: Redis, MemoryCache, Distributed
- Notification: Email, SMS, Push, Slack

✅ Strategy pattern via DI

✅ Tenant-specific services
```

```csharp
// Storage strategy
builder.Services.AddKeyedSingleton<IStorageService, S3StorageService>("s3");
builder.Services.AddKeyedSingleton<IStorageService, AzureBlobStorageService>("azure");
builder.Services.AddKeyedSingleton<IStorageService, LocalStorageService>("local");

// В config
public class FileService
{
    private readonly IStorageService _storage;
    
    public FileService(
        IConfiguration config,
        [FromKeyedServices("s3")] IStorageService s3,
        [FromKeyedServices("azure")] IStorageService azure,
        [FromKeyedServices("local")] IStorageService local)
    {
        var provider = config["Storage:Provider"];   // "s3"
        _storage = provider switch
        {
            "s3" => s3,
            "azure" => azure,
            "local" => local,
            _ => throw new InvalidOperationException()
        };
    }
}
```

> [!question]- **Интервью: keyed services .NET 8+ зачем?**
> Multiple implementations одного interface, выбираемые по **key** (string или any object). До .NET 8 нужен был manual factory. Теперь — встроенная DI feature. **Use cases**: 1) Strategy pattern (Email/SMS/Push notifications). 2) Multi-tenant (different services per tenant). 3) Storage providers (S3/Azure/Local). **Syntax**: `AddKeyedScoped<I, T>(key)`, inject через `[FromKeyedServices(key)] I service` или resolve через `provider.GetRequiredKeyedService<I>(key)`. **Trade-off vs separate interfaces**: keyed cleaner для same-interface variants, separate interfaces — для different contracts.

---

## 4. TimeProvider (.NET 8+)

### 4.1. Проблема

```csharp
public class OrderService
{
    public Order CreateOrder()
    {
        return new Order
        {
            CreatedAt = DateTime.UtcNow   // ❌ Hard-coded — нельзя test
        };
    }
}

// Test:
var order = service.CreateOrder();
Assert.Equal(expectedTime, order.CreatedAt);   // ❌ flaky — current time всегда другой
```

### 4.2. Solution: TimeProvider

```csharp
public class OrderService
{
    private readonly TimeProvider _timeProvider;
    
    public OrderService(TimeProvider timeProvider)
    {
        _timeProvider = timeProvider;
    }
    
    public Order CreateOrder()
    {
        return new Order
        {
            CreatedAt = _timeProvider.GetUtcNow().UtcDateTime
        };
    }
}

// Program.cs
builder.Services.AddSingleton(TimeProvider.System);   // production: real time

// Test:
public class OrderServiceTests
{
    [Fact]
    public void CreateOrder_SetsCreatedAt()
    {
        // Microsoft.Extensions.TimeProvider.Testing package
        var fakeTime = new FakeTimeProvider(new DateTimeOffset(2026, 5, 10, 12, 0, 0, TimeSpan.Zero));
        var service = new OrderService(fakeTime);
        
        var order = service.CreateOrder();
        
        Assert.Equal(new DateTime(2026, 5, 10, 12, 0, 0), order.CreatedAt);
    }
}
```

### 4.3. Capabilities

```csharp
TimeProvider provider = TimeProvider.System;

DateTimeOffset now = provider.GetUtcNow();
DateTimeOffset localNow = provider.GetLocalNow();
TimeZoneInfo tz = provider.LocalTimeZone;

long timestamp = provider.GetTimestamp();
TimeSpan elapsed = provider.GetElapsedTime(startTimestamp);

// Async timers — testable!
var timer = provider.CreateTimer(callback, state, dueTime, period);
```

### 4.4. Replace DateTime.UtcNow везде

```csharp
// Старый стиль
public class TokenService
{
    public bool IsExpired(Token token) =>
        token.ExpiresAt < DateTime.UtcNow;
}

// Modern .NET 8+
public class TokenService
{
    private readonly TimeProvider _time;
    
    public TokenService(TimeProvider time) => _time = time;
    
    public bool IsExpired(Token token) =>
        token.ExpiresAt < _time.GetUtcNow();
}
```

---

## 5. Async DI patterns

### 5.1. Async initialization

DI не supports async constructors — но можно через factory pattern:

```csharp
public interface IInitializableService
{
    Task InitializeAsync(CancellationToken ct);
}

public class CacheService : IInitializableService
{
    private Dictionary<string, object> _cache = new();
    
    public async Task InitializeAsync(CancellationToken ct)
    {
        // Async load initial data
        _cache = await LoadFromDbAsync(ct);
    }
}

// Hosted service для initialization at startup
public class StartupInitializer : IHostedService
{
    private readonly IServiceProvider _provider;
    
    public StartupInitializer(IServiceProvider provider) => _provider = provider;
    
    public async Task StartAsync(CancellationToken ct)
    {
        using var scope = _provider.CreateScope();
        var services = scope.ServiceProvider.GetServices<IInitializableService>();
        
        foreach (var service in services)
        {
            await service.InitializeAsync(ct);
        }
    }
    
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}

builder.Services.AddSingleton<CacheService>();
builder.Services.AddSingleton<IInitializableService>(sp => sp.GetRequiredService<CacheService>());
builder.Services.AddHostedService<StartupInitializer>();
```

### 5.2. Async Factory

```csharp
public class DatabaseConnectionFactory
{
    private readonly string _connStr;
    
    public DatabaseConnectionFactory(IConfiguration config)
    {
        _connStr = config.GetConnectionString("Default")!;
    }
    
    public async Task<IDbConnection> CreateAsync(CancellationToken ct = default)
    {
        var connection = new SqlConnection(_connStr);
        await connection.OpenAsync(ct);
        return connection;
    }
}

builder.Services.AddSingleton<DatabaseConnectionFactory>();

// Использование
public class OrderRepository
{
    private readonly DatabaseConnectionFactory _factory;
    
    public OrderRepository(DatabaseConnectionFactory factory) => _factory = factory;
    
    public async Task<Order?> GetByIdAsync(int id, CancellationToken ct)
    {
        await using var connection = await _factory.CreateAsync(ct);
        return await connection.QueryFirstOrDefaultAsync<Order>(...);
    }
}
```

---

## 6. Service registration patterns

### 6.1. Generic registration

```csharp
// Register multiple implementations через assembly scanning
builder.Services.Scan(scan => scan
    .FromAssemblyOf<Program>()
    .AddClasses(c => c.AssignableTo<IRepository>())
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```

(Требует `Scrutor` package.)

### 6.2. Decorator pattern

```bash
dotnet add package Scrutor
```

```csharp
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.Decorate<IUserService, CachingUserService>();
builder.Services.Decorate<IUserService, LoggingUserService>();

// Pipeline: LoggingUserService → CachingUserService → UserService
```

### 6.3. Options pattern

```csharp
// 1. Class for options
public class JwtOptions
{
    public string Issuer { get; set; } = "";
    public string Audience { get; set; } = "";
    public string Key { get; set; } = "";
    public int ExpirationMinutes { get; set; }
}

// 2. Bind from configuration
builder.Services.Configure<JwtOptions>(
    builder.Configuration.GetSection("Jwt"));

// 3. Inject via IOptions / IOptionsSnapshot / IOptionsMonitor
public class TokenService
{
    private readonly JwtOptions _options;
    
    public TokenService(IOptions<JwtOptions> options)
    {
        _options = options.Value;
    }
}
```

```
IOptions<T>          — singleton, не reload при changes
IOptionsSnapshot<T>  — scoped, reload каждый request
IOptionsMonitor<T>   — singleton с change notifications
```

### 6.4. Validation options

```csharp
builder.Services
    .AddOptions<JwtOptions>()
    .Bind(builder.Configuration.GetSection("Jwt"))
    .ValidateDataAnnotations()
    .ValidateOnStart();   // .NET 6+

public class JwtOptions
{
    [Required]
    public string Issuer { get; set; } = "";
    
    [Required]
    [MinLength(32)]
    public string Key { get; set; } = "";
}
```

App не запустится если config invalid.

---

## 7. Common pitfalls

### 7.1. DbContext as Singleton

```csharp
builder.Services.AddSingleton<AppDbContext>();   // ❌ NEVER!
```

DbContext **не thread-safe**, change tracker должен resetся каждый request. Singleton DbContext:
- Memory leak (entity tracking grows)
- Race conditions
- Stale data

**Fix**: `AddDbContext` (scoped by default).

### 7.2. Captive dependency

См. раздел 2. Singleton зависит от Scoped → ObjectDisposedException.

### 7.3. Multiple registrations one interface

```csharp
builder.Services.AddScoped<IService, ServiceA>();
builder.Services.AddScoped<IService, ServiceB>();   // override A!

// При resolve — получишь B (last registration wins)
```

Хочешь оба — используй keyed services (.NET 8+) или register `IEnumerable<IService>`.

### 7.4. Disposal не вызвается

```csharp
public class MyService : IDisposable
{
    public void Dispose() { /* cleanup */ }
}

// Resolved manually:
var service = serviceProvider.GetRequiredService<MyService>();
// service.Dispose() не вызывается auto!
```

**Fix**: scope-based:

```csharp
using var scope = serviceProvider.CreateScope();
var service = scope.ServiceProvider.GetRequiredService<MyService>();
// При scope.Dispose() — service.Dispose()
```

### 7.5. Service locator anti-pattern

```csharp
// ❌
public class OrderService
{
    public void Process()
    {
        var db = ServiceLocator.GetService<AppDbContext>();   // hidden dep!
    }
}

// ✅
public class OrderService
{
    private readonly AppDbContext _db;
    
    public OrderService(AppDbContext db) => _db = db;
}
```

### 7.6. Constructor с too many dependencies

```csharp
public class OrderService
{
    public OrderService(
        IUserRepository users,
        IProductRepository products,
        IInventoryService inventory,
        IPaymentService payment,
        IShippingService shipping,
        INotificationService notification,
        ILogger<OrderService> logger,
        IUnitOfWork uow,
        IDateTimeProvider time,
        IDiscountCalculator discount,
        // ... 15 more
    ) { }
}
```

10+ deps → SRP violation. Split на smaller services / aggregate cohesive.

### 7.7. Forgot ConfigureServices order

```csharp
// ❌ DI registration ПОСЛЕ Build
var app = builder.Build();
builder.Services.AddSingleton<IService, Impl>();   // не работает!
```

Все DI registrations **до** `builder.Build()`.

### 7.8. Async constructor попытка

```csharp
public class CacheService
{
    public CacheService(IDataLoader loader)
    {
        var data = loader.LoadAsync().Result;   // ❌ Sync-over-async deadlock!
    }
}
```

**Fix**: async initialization через IHostedService или factory.

### 7.9. IOptions vs IOptionsSnapshot

```csharp
public class TokenService(IOptions<JwtOptions> options)
{
    // ⚠️ IOptions singleton — не reload при changes
}

// Если config меняется в runtime
public class TokenService(IOptionsSnapshot<JwtOptions> options)
{
    // Scoped — reload каждый request
}
```

### 7.10. Не testable design

```csharp
public class UserService
{
    public UserService()
    {
        _db = new AppDbContext();   // ❌ Не testable
        _logger = new ConsoleLogger();
    }
}

// ✅
public class UserService(AppDbContext db, ILogger<UserService> logger)
{
    // Testable — все deps injected
}
```

> [!question]- **Интервью: топ-3 ошибки с DI?**
> 1) **Captive dependency** — Singleton зависит от Scoped (DbContext в Singleton). Fix: `IServiceScopeFactory.CreateScope()`. 2) **DbContext as Singleton** — race conditions, memory leaks. Fix: `AddDbContext` (scoped by default). 3) **Service Locator anti-pattern** — `ServiceProvider.GetService<T>()` внутри methods вместо constructor injection. Hidden dependencies, hard to test. **Bonus**: 10+ constructor параметров → SRP violation. **Bonus 2**: async constructor попытка → use IHostedService для startup async work.

---

## 8. Cheat sheet

```csharp
// === Lifetimes ===
builder.Services.AddSingleton<ICache, RedisCache>();           // 1 instance app
builder.Services.AddScoped<IUserService, UserService>();        // 1 per request
builder.Services.AddTransient<IDateProvider, SystemDateProvider>(); // new each time

// === Keyed services (.NET 8+) ===
builder.Services.AddKeyedScoped<INotification, EmailNotification>("email");
builder.Services.AddKeyedScoped<INotification, SmsNotification>("sms");

public class Service([FromKeyedServices("email")] INotification email) { }

// === IServiceScopeFactory для Singleton → Scoped ===
public class CacheService(IServiceScopeFactory scopeFactory)
{
    public async Task RefreshAsync()
    {
        using var scope = scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    }
}

// === TimeProvider (.NET 8+) ===
builder.Services.AddSingleton(TimeProvider.System);

public class Service(TimeProvider time)
{
    public DateTime Now => time.GetUtcNow().UtcDateTime;
}

// === Options pattern ===
public class JwtOptions
{
    [Required] public string Issuer { get; set; } = "";
    [Required, MinLength(32)] public string Key { get; set; } = "";
}

builder.Services
    .AddOptions<JwtOptions>()
    .Bind(builder.Configuration.GetSection("Jwt"))
    .ValidateDataAnnotations()
    .ValidateOnStart();

// IOptions<T>          — singleton
// IOptionsSnapshot<T>  — scoped, reload per request
// IOptionsMonitor<T>   — change notifications

// === Decorator (Scrutor) ===
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.Decorate<IUserService, CachingUserService>();
builder.Services.Decorate<IUserService, LoggingUserService>();

// === Async initialization ===
public class StartupInit(IServiceProvider provider) : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        using var scope = provider.CreateScope();
        var service = scope.ServiceProvider.GetRequiredService<MyService>();
        await service.InitializeAsync(ct);
    }
    
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

---

## 9. Practice exercises

### 9.1. Fix captive dependency

Дано:
```csharp
public class CacheService : IHostedService
{
    private readonly AppDbContext _db;
    
    public CacheService(AppDbContext db) => _db = db;
    
    public async Task StartAsync(CancellationToken ct)
    {
        var data = await _db.Items.ToListAsync();
        // cache data
    }
}
```

Перепиши через IServiceScopeFactory.

### 9.2. Implement strategy через keyed services

Реализуй image storage:
- `IStorageService` interface
- `S3StorageService`, `AzureStorageService`, `LocalStorageService`
- Configure via `appsettings.json`: `Storage:Provider = "s3"`
- Auto-resolve correct implementation

### 9.3. Refactor to TimeProvider

Найди все `DateTime.UtcNow` / `DateTime.Now` в существующем коде. Refactor через `TimeProvider`. Напиши tests с `FakeTimeProvider`.

---

## 10. Что читать дальше

1. **`AspNetCore/Senior/di-configuration.md`** — DI deep production
2. **`AspNetCore/Senior/pipeline-middleware.md`** — middleware
3. **`AspNetCore/Middle/aspnet-error-handling.md`** — error handling
4. **`AspNetCore/Senior/hosting-background.md`** — BackgroundService
5. **`Architecture/Senior/solid.md`** — SOLID + DI

---

## 11. См. также

- [[di-configuration|AspNetCore/Senior/di-configuration]] — DI deep
- [[pipeline-middleware|AspNetCore/Senior/pipeline-middleware]] — middleware
- [[aspnet-controllers-routing|AspNetCore/Middle/aspnet-controllers-routing]] — controllers
- [[hosting-background|AspNetCore/Senior/hosting-background]] — BackgroundService
- [[solid|Architecture/Senior/solid]] — SOLID

---

## 12. Reading list

- **Microsoft Docs — DI** — learn.microsoft.com/aspnet/core/fundamentals/dependency-injection
- **Microsoft Docs — Keyed Services** — learn.microsoft.com/dotnet/core/extensions/dependency-injection (.NET 8 keyed)
- **Microsoft Docs — TimeProvider** — learn.microsoft.com/dotnet/standard/datetime/timeprovider-overview
- **Microsoft Docs — Options Pattern** — learn.microsoft.com/aspnet/core/fundamentals/configuration/options
- **Mark Seemann — Dependency Injection in .NET** (book)
- **Steve Smith — DI articles** — ardalis.com
- **Scrutor** — github.com/khellang/Scrutor
