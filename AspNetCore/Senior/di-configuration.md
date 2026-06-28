---
tags: [di, dependency-injection, configuration, options, keyed-services, ioptions, aot]
level: Senior
---

# Dependency Injection и Configuration

> DI-контейнер ASP.NET Core, lifetimes, Keyed Services и типизированная конфигурация через `IOptions` с учётом AOT-ограничений.

## Что это, зачем и когда

### Что такое DI?
**Dependency Injection** — паттерн где зависимости передаются объекту извне (через constructor / setter), а не создаются им внутри.

```csharp
// ❌ Без DI — tight coupling
public class OrderService
{
    private readonly SqlOrderRepository _repo = new();  // hard-coded зависимость

    public Task ProcessAsync(Order o) => _repo.SaveAsync(o);
}

// ✅ С DI
public class OrderService(IOrderRepository repo)
{
    public Task ProcessAsync(Order o) => repo.SaveAsync(o);
}
```

**Аналогия:** Без DI ты сам ищешь батарейки в своих устройствах. С DI — "батарейка" приходит к тебе через единый источник (DI container), и ты можешь её заменить (mock в тестах, разные реализации в разных environments).

### Зачем

| Без DI | С DI |
|--------|------|
| Tight coupling — не поменять реализацию | Loose coupling — меняем через DI |
| Сложно тестировать (моки невозможны) | Easy mocking |
| Lifecycle руками везде | DI управляет (singleton/scoped/transient) |
| Конфигурация размазана по коду | Централизована в Program.cs |

### Когда применять
- Любой нетривиальный сервис .NET — стандарт
- Не применяй для value-objects, DTOs, простых моделей данных

---

## Service Lifetimes

| Lifetime | Когда создаётся | Когда удаляется |
|----------|----------------|-----------------|
| **Singleton** | Один раз на startup | На shutdown app |
| **Scoped** | Один раз на scope (HTTP request) | На end of scope |
| **Transient** | Каждый раз | Сразу после использования |

```csharp
builder.Services.AddSingleton<IClock, SystemClock>();         // глобальный
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();        // per-request
builder.Services.AddTransient<IEmailService, EmailService>(); // каждый раз новый
```

### Сюрпризы lifetime

```csharp
// ❌ Singleton содержит Scoped — captured в первый scope, использован вторым
public class CachedService(IUserRepository userRepo)  // userRepo Scoped
{
}
builder.Services.AddSingleton<CachedService>();  // капчурим scoped как singleton

// ✅ IServiceScopeFactory — создаём scope явно
public class CachedService(IServiceScopeFactory scopeFactory)
{
    public async Task DoWorkAsync()
    {
        using var scope = scopeFactory.CreateScope();
        var userRepo = scope.ServiceProvider.GetRequiredService<IUserRepository>();
        // ...
    }
}
```

ASP.NET Core валидирует это в Development и кидает `InvalidOperationException`. В production не валидируется по умолчанию — обязательно включи:

```csharp
builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateScopes = true;          // not allow scoped → singleton capture
    options.ValidateOnBuild = true;          // catch issues at startup
});
```

> [!question]- **Интервью: что произойдёт если Singleton инжектит Scoped?**
> **Captive dependency.** Scoped service инициализируется в первом запросе, но Singleton его держит → второй запрос читает state первого, может быть disposed.
> С `ValidateScopes = true` (Dev default) — `InvalidOperationException` при попытке resolve.
> Решение: либо переделать Singleton в Scoped (если возможно), либо использовать `IServiceScopeFactory` для создания scope вручную, либо передавать factory `Func<T>`.

### Validate at startup

```csharp
// Проверка что все services корректно резолвятся ДО первого запроса
builder.Services.AddOptions<MyOptions>()
    .ValidateDataAnnotations()
    .ValidateOnStart();  // assert at startup, fail fast

builder.Host.UseDefaultServiceProvider(options =>
{
    options.ValidateOnBuild = true;
    options.ValidateScopes = true;
});
```

`ValidateOnBuild` — в startup пытается резолвить ВСЕ зарегистрированные services. Падает с понятной ошибкой если что-то missing.

---

## Регистрация services

### Базовые методы

```csharp
// По интерфейсу
builder.Services.AddScoped<IOrderService, OrderService>();

// По конкретному типу (auto-discovery через ctor)
builder.Services.AddScoped<OrderService>();

// Factory function — для сложного init
builder.Services.AddScoped<IClient>(sp =>
{
    var config = sp.GetRequiredService<IConfiguration>();
    return new ApiClient(config["ApiKey"]!);
});

// Existing instance
builder.Services.AddSingleton<IClock>(new SystemClock());
```

### TryAdd vs Add

```csharp
// Add — всегда добавляет (несколько регистраций — последняя выигрывает в resolve, но все live в коллекции)
services.AddSingleton<IService, ServiceA>();
services.AddSingleton<IService, ServiceB>();
// Resolve IService → ServiceB. IEnumerable<IService> → both

// TryAdd — добавляет только если не зарегистрирован
services.TryAddSingleton<IService, ServiceA>();
services.TryAddSingleton<IService, ServiceB>();  // skipped
```

`TryAdd` для библиотек: позволяет user'у override default registration.

### Multiple implementations

```csharp
services.AddScoped<INotificationHandler, EmailHandler>();
services.AddScoped<INotificationHandler, SmsHandler>();
services.AddScoped<INotificationHandler, PushHandler>();

// Inject all
public class Notifier(IEnumerable<INotificationHandler> handlers)
{
    public async Task NotifyAsync(...)
    {
        foreach (var h in handlers) await h.SendAsync(...);
    }
}
```

---

## Keyed Services (.NET 8+)

Несколько реализаций одного интерфейса с явным ключом:

```csharp
services.AddKeyedSingleton<ICache, MemoryCache>("memory");
services.AddKeyedSingleton<ICache, RedisCache>("redis");

// Inject specific
public class Service([FromKeyedServices("redis")] ICache cache)
{
    // ...
}

// Resolve manually
var cache = serviceProvider.GetRequiredKeyedService<ICache>("redis");
```

### Когда применять

| Применять | Не применять |
|-----------|--------------|
| Множество стратегий одного интерфейса (различные payment providers) | Если хватает `IEnumerable<T>` |
| Per-tenant конфигурация | Когда логика выбора implementation сложная |
| Per-environment fallback (in-memory dev, Redis prod) | Type-based dispatch вместо string keys |

```csharp
// Production-ready пример — payment processors
services.AddKeyedScoped<IPaymentProcessor, StripeProcessor>("stripe");
services.AddKeyedScoped<IPaymentProcessor, PayPalProcessor>("paypal");
services.AddKeyedScoped<IPaymentProcessor, CryptoProcessor>("crypto");

public class CheckoutService(IServiceProvider sp)
{
    public async Task ProcessAsync(string method, ...)
    {
        var processor = sp.GetRequiredKeyedService<IPaymentProcessor>(method);
        await processor.ChargeAsync(...);
    }
}
```

> [!question]- **Интервью: чем keyed services лучше factory pattern?**
> Factory: ты пишешь `IPaymentFactory`, он switch'ит по string → возвращает implementation. Boilerplate.
> Keyed: DI container делает то же сам. Меньше кода, type-safe registration, native integration с DI lifecycle.
> Pre-.NET 8 — приходилось через `Func<string, IService>` или factory. Теперь нативно.

---

## ActivatorUtilities — manual instantiation с DI

```csharp
public class MyClass(IDependency dep, string param) { }

// Создаём instance с DI-резолвом части ctor params
var instance = ActivatorUtilities.CreateInstance<MyClass>(serviceProvider, "my-string");
// IDependency инжектится из DI, "my-string" передаётся как параметр
```

Используется когда:
- Создаёшь instance в runtime (плагины, dynamic types)
- Часть аргументов из DI, часть — runtime-defined
- Factory pattern — фабрика принимает параметр + DI зависимости

```csharp
public class HandlerFactory(IServiceProvider sp)
{
    public IHandler<T> Create<T>(string name) =>
        ActivatorUtilities.CreateInstance<TypedHandler<T>>(sp, name);
}
```

---

## AsyncServiceScope (.NET 6+)

```csharp
// Старый API (синхронный Dispose)
using var scope = serviceProvider.CreateScope();
var service = scope.ServiceProvider.GetRequiredService<IService>();
await service.DoAsync();
// scope.Dispose() — синхронно, ждёт async cleanup'ы

// ✅ Async API
await using var scope = serviceProvider.CreateAsyncScope();
var service = scope.ServiceProvider.GetRequiredService<IService>();
await service.DoAsync();
// await scope.DisposeAsync() — корректно ждёт IAsyncDisposable services
```

Critical если scoped services реализуют `IAsyncDisposable` (например, DbContext с pending async cleanup).

---

## IOptions — конфигурация

```csharp
public sealed class JwtOptions
{
    public const string SectionName = "Jwt";

    [Required, MinLength(32)]
    public string Key { get; init; } = "";

    [Required]
    public string Issuer { get; init; } = "";

    [Required]
    public string Audience { get; init; } = "";

    [Range(1, 60)]
    public int ExpirationMinutes { get; init; } = 15;
}
```

### Регистрация

```csharp
// Базовая
builder.Services.Configure<JwtOptions>(
    builder.Configuration.GetSection(JwtOptions.SectionName));

// С валидацией + ValidateOnStart
builder.Services
    .AddOptions<JwtOptions>()
    .Bind(builder.Configuration.GetSection(JwtOptions.SectionName))
    .ValidateDataAnnotations()
    .Validate(opts => opts.ExpirationMinutes <= 60, "Expiration must be ≤ 60 minutes")
    .ValidateOnStart();
```

### Использование

```csharp
public class TokenService(IOptions<JwtOptions> options)
{
    private readonly JwtOptions _options = options.Value;

    public string Generate(...) => /* using _options.Key, _options.Issuer */;
}
```

### IOptions vs IOptionsSnapshot vs IOptionsMonitor

| | IOptions | IOptionsSnapshot | IOptionsMonitor |
|--|----------|------------------|-----------------|
| Lifetime | Singleton | Scoped | Singleton |
| Перечитывает config | Никогда | На каждом scope (request) | Live (через `OnChange`) |
| Когда | Static config | Per-request feature flags | Background services с reload |

```csharp
// IOptionsMonitor — для long-running с hot-reload
public class FeatureFlagService(IOptionsMonitor<FeatureFlags> monitor)
{
    public FeatureFlagService(IOptionsMonitor<FeatureFlags> monitor)
    {
        monitor.OnChange(flags => Console.WriteLine($"Flags changed: {flags.Enabled}"));
    }

    public bool IsEnabled() => monitor.CurrentValue.Enabled;
}
```

> [!question]- **Интервью: разница между IOptions, IOptionsSnapshot, IOptionsMonitor?**
> **`IOptions<T>`** — Singleton. Bind'ится один раз. Используй когда конфигурация **не** меняется в runtime (Jwt secret).
> **`IOptionsSnapshot<T>`** — Scoped. Перечитывается на каждый request. Используй для **per-request** feature flags / A/B test settings.
> **`IOptionsMonitor<T>`** — Singleton. **Live updates** через `OnChange`. Используй в Singleton/BackgroundService где нужно реагировать на config-изменения без рестарта.

### Configuration Sources

```csharp
builder.Configuration
    .AddJsonFile("appsettings.json", optional: false, reloadOnChange: true)
    .AddJsonFile($"appsettings.{builder.Environment.EnvironmentName}.json",
                 optional: true, reloadOnChange: true)
    .AddEnvironmentVariables(prefix: "MYAPP_")
    .AddCommandLine(args)
    .AddUserSecrets<Program>(optional: true)  // Dev only
    .AddAzureKeyVault(...)                     // Production secrets
    .AddCustomConfigSource();
```

**Order matters!** Last source wins. Standard pattern:
1. appsettings.json (base)
2. appsettings.{Env}.json (env-specific)
3. User secrets (dev only)
4. Environment variables (production overrides)
5. Command line (highest priority for dev/debugging)

### Hot reload

Default `reloadOnChange: true` для JSON files. ASP.NET Core watches файл, при изменении — пересоздаёт `IOptionsSnapshot`/`IOptionsMonitor`. Singleton `IOptions` **не** обновляется (это by design).

---

## Configuration Binding Source Generator (AOT)

В runtime обычный `Configuration.Bind<T>()` использует **reflection** → не AOT-friendly.

```xml
<!-- .csproj -->
<PropertyGroup>
  <EnableConfigurationBindingGenerator>true</EnableConfigurationBindingGenerator>
</PropertyGroup>
```

После этого binding транслируется в source-generated code → AOT-compatible, faster startup.

```csharp
// Работает с source-gen
builder.Services
    .AddOptions<JwtOptions>()
    .Bind(builder.Configuration.GetSection(JwtOptions.SectionName))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

См. [Native AOT](native-aot.md) и [Source Generators]().

---

## Strongly-typed configuration patterns

### 1. Per-feature options

```csharp
public sealed class EmailOptions { /* ... */ }
public sealed class JwtOptions { /* ... */ }
public sealed class CacheOptions { /* ... */ }

services.Configure<EmailOptions>(config.GetSection("Email"));
services.Configure<JwtOptions>(config.GetSection("Jwt"));
services.Configure<CacheOptions>(config.GetSection("Cache"));
```

Чище чем монолитный `AppOptions` со всем подряд.

### 2. Nested options

```csharp
public sealed class CacheOptions
{
    public RedisOptions Redis { get; init; } = new();
    public MemoryOptions Memory { get; init; } = new();
}

public sealed class RedisOptions
{
    public string ConnectionString { get; init; } = "";
    public int DatabaseId { get; init; } = 0;
}
```

```json
{
  "Cache": {
    "Redis": {
      "ConnectionString": "...",
      "DatabaseId": 0
    },
    "Memory": {
      "MaxSizeMB": 100
    }
  }
}
```

### 3. Named options

```csharp
services.Configure<HttpClientOptions>("WeatherApi", config.GetSection("HttpClients:Weather"));
services.Configure<HttpClientOptions>("PaymentApi", config.GetSection("HttpClients:Payment"));

public class WeatherClient(IOptionsSnapshot<HttpClientOptions> options)
{
    private readonly HttpClientOptions _opts = options.Get("WeatherApi");
}
```

Удобно когда есть однотипные options (несколько HTTP clients, multiple DBs).

### 4. Validation через `IValidateOptions<T>`

Для сложной валидации (cross-field, async):

```csharp
public class JwtOptionsValidator : IValidateOptions<JwtOptions>
{
    public ValidateOptionsResult Validate(string? name, JwtOptions options)
    {
        if (options.Key.Length < 32)
            return ValidateOptionsResult.Fail("Key too short");

        if (Encoding.UTF8.GetByteCount(options.Key) < 32)
            return ValidateOptionsResult.Fail("Key UTF-8 byte count < 32");

        return ValidateOptionsResult.Success;
    }
}

services.AddSingleton<IValidateOptions<JwtOptions>, JwtOptionsValidator>();
```

---

## Decorator pattern через DI

```csharp
public interface IRepository { Task<T> GetAsync<T>(int id); }
public class SqlRepository : IRepository { /* ... */ }

public class CachedRepository(IRepository inner, IMemoryCache cache) : IRepository
{
    public async Task<T> GetAsync<T>(int id) =>
        await cache.GetOrCreateAsync($"{typeof(T).Name}:{id}", _ => inner.GetAsync<T>(id));
}

// Регистрация — built-in API нет, но через Scrutor:
services.AddScoped<IRepository, SqlRepository>();
services.Decorate<IRepository, CachedRepository>();
```

`Scrutor` (NuGet) добавляет decorator pattern и assembly scanning к стандартному DI.

---

## Assembly scanning через Scrutor

Без scanning — вручную регистрируешь каждый service:
```csharp
services.AddScoped<IOrderService, OrderService>();
services.AddScoped<IUserService, UserService>();
services.AddScoped<INotificationService, NotificationService>();
// ... boilerplate
```

С Scrutor:
```csharp
services.Scan(scan => scan
    .FromAssemblyOf<IService>()
    .AddClasses(c => c.AssignableTo<IService>())
        .AsImplementedInterfaces()
        .WithScopedLifetime());
```

Auto-регистрирует все классы реализующие `IService`.

### MediatR style

Большинство фреймворков (MediatR, MassTransit) делают свой scanning:
```csharp
services.AddMediatR(cfg => cfg.RegisterServicesFromAssemblyContaining<Program>());
services.AddMassTransit(x => x.AddConsumers(typeof(Program).Assembly));
```

---

## Disposal

DI container — owner всех зарегистрированных services. На end of scope (или app shutdown) — автоматически вызывает `Dispose()` или `DisposeAsync()`.

```csharp
public class Service : IDisposable
{
    public void Dispose() { /* cleanup */ }
}

services.AddScoped<Service>();

// На end of HTTP request — Dispose() вызывается
```

### Pitfall: вручную создал, но не registered

```csharp
public class OrderService(IConfiguration config)
{
    public async Task DoAsync()
    {
        var client = new HttpClient();  // ❌ DI не знает о нём — никогда не Dispose
        await client.GetAsync(...);
    }
}
```

Если нужно — `using` или register в DI:
```csharp
public class OrderService(HttpClient client) { /* ... */ }
services.AddHttpClient<OrderService>();  // managed lifecycle
```

### IAsyncDisposable

```csharp
public class AsyncService : IAsyncDisposable
{
    public async ValueTask DisposeAsync()
    {
        await CleanupAsync();
    }
}

// Default ServiceProvider supports IAsyncDisposable since .NET 6
await using var scope = serviceProvider.CreateAsyncScope();
// ↑ async dispose calls IAsyncDisposable.DisposeAsync()
```

См. [HFT/Low-Latency]() — почему `GetAwaiter().GetResult()` в Dispose — антипаттерн.

---

## Common pitfalls

### 1. Captive dependency
Singleton ← Scoped без `ValidateScopes = true` → silently broken.

### 2. Service Locator anti-pattern

```csharp
// ❌ ServiceProvider напрямую — скрытые зависимости
public class Service(IServiceProvider sp)
{
    public async Task DoAsync()
    {
        var dep = sp.GetRequiredService<IDep>();  // hidden!
    }
}

// ✅ Constructor injection
public class Service(IDep dep) { /* ... */ }
```

Service Locator уместен только в очень специфичных кейсах — factories, plugins. Для бизнес-логики — constructor.

### 3. Too many constructor params

```csharp
// ❌ 15 параметров — class делает слишком много
public class HugeService(IRepo a, IRepo b, IRepo c, IService d, ILogger e, ...)
```
Признак violation of SRP. Раздели на меньшие services.

### 4. Forgot register

```
InvalidOperationException: Unable to resolve service for type 'IService'
```
`ValidateOnBuild = true` ловит это в startup, не в первом request.

### 5. Mutable Singleton

```csharp
// ❌ Singleton с mutable state
public class Counter
{
    public int Count { get; set; }  // shared между всеми users!
}
services.AddSingleton<Counter>();
```
Race conditions / data leaks между requests.
**Решение:** Singleton должен быть immutable / thread-safe (Interlocked, ConcurrentCollections).

### 6. Multiple `Configure<T>`

```csharp
services.Configure<JwtOptions>(config.GetSection("Jwt"));
services.Configure<JwtOptions>(opts => opts.Key = "override");
// Оба вызываются по очереди — последний выигрывает per field
```
Может быть unexpected. Лучше один `Configure` + `IConfigureOptions<T>` для extensions.

### 7. Lazy<T> capture

```csharp
public class Service(Lazy<IDependency> dep)
{
    public void DoSomething() => dep.Value.X();  // resolved on first access
}
```
Полезно для circular dependencies или tail-loaded deps. Pitfall: scope — Lazy<T> capture'ит в момент создания scope, не на каждом access.

### 8. EnableSensitiveDataLogging в Production через config override
Случайно выкатил `appsettings.Production.json` с включённым sensitive logging → PII в логах. Всегда review configs перед deploy.

### 9. Connection string в config plain text
В `appsettings.json` хранить пароль БД — git leak.
**Решение:** secrets manager (см. [Auth & Security / Secrets Management](auth-security.md#secrets-management)).

### 10. Не использовать `IOptionsValidation`
Невалидные options ломают app в runtime когда уже late.
**Решение:** `ValidateDataAnnotations() + ValidateOnStart()`.

---

## Production checklist

- [ ] `ValidateScopes = true` + `ValidateOnBuild = true` в Production
- [ ] Все `IOptions<T>` зарегистрированы через `AddOptions<T>().ValidateOnStart()`
- [ ] Sensitive config (passwords, keys) — в secrets manager, не в JSON
- [ ] Хранение state в Singleton — only thread-safe / immutable
- [ ] Все `IDisposable` зарегистрированы в DI (managed disposal)
- [ ] Constructor injection вместо Service Locator
- [ ] Keyed services для множественных implementations с явным выбором
- [ ] `EnableConfigurationBindingGenerator` для AOT-targets
- [ ] `IOptionsMonitor` для long-running services с hot reload
- [ ] `IOptionsSnapshot` для per-request settings
- [ ] Per-feature options (не монолитный `AppOptions`)
- [ ] Tests verify DI registration (resolve test)

---

## Common pitfalls (повтор как checklist)

- [ ] Captive dependency — Singleton ← Scoped
- [ ] Service Locator pattern — обнаружить и refactor
- [ ] Mutable Singleton — добавить thread-safety или сделать immutable
- [ ] Forgotten registration — ValidateOnBuild
- [ ] Plain-text secrets в config

---

## См. также

- [Auth и Security](auth-security.md) — secrets management, configuration security
- [Hosting и Background](hosting-background.md) — `IServiceScopeFactory` в BackgroundService
- [Native AOT](native-aot.md) — ConfigurationBindingGenerator для AOT
- [Caching](caching.md) — `IOptionsMonitor` для hot-reload feature flags
- [Pipeline и Middleware](pipeline-middleware.md) — middleware order, IOC integration
- [Source Generators]() — Configuration Binding Source Generator

## Reading list

- **Microsoft Learn — Dependency Injection** — learn.microsoft.com/aspnet/core/fundamentals/dependency-injection
- **Microsoft Learn — Options Pattern** — learn.microsoft.com/dotnet/core/extensions/options
- **Andrew Lock — DI series** — andrewlock.net/series/asp-net-core-dependency-injection/
- **Mark Seemann — Dependency Injection Principles, Practices, and Patterns** — canonical book
- **Steve Smith — Decorator pattern with Scrutor** — ardalis.com
- **Khalid Abuhakmeh** — keyed services examples
