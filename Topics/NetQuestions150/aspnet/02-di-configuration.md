# DI и Configuration

## Dependency Injection

ASP.NET Core имеет встроенный IoC-контейнер. Все сервисы регистрируются в `IServiceCollection` и разрешаются через `IServiceProvider`.

### Lifetime сервисов

| Lifetime | Создание | Область | Типичное применение |
|----------|----------|---------|---------------------|
| **Transient** | Новый экземпляр при каждом запросе к контейнеру | Один вызов `GetService` | Stateless сервисы, лёгкие операции |
| **Scoped** | Один экземпляр на scope (HTTP-запрос) | В рамках scope | DbContext, Unit of Work, текущий пользователь |
| **Singleton** | Один экземпляр на всё время жизни приложения | Глобально | Кэши, HttpClient factory, конфигурация |

### Регистрация сервисов

```csharp
var builder = WebApplication.CreateBuilder(args);

// Базовая регистрация
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddSingleton<ICacheService, InMemoryCacheService>();

// Регистрация с фабрикой
builder.Services.AddScoped<IUserContext>(sp =>
{
    var httpContext = sp.GetRequiredService<IHttpContextAccessor>().HttpContext;
    return new UserContext(httpContext?.User);
});

// Регистрация нескольких реализаций
builder.Services.AddTransient<INotifier, EmailNotifier>();
builder.Services.AddTransient<INotifier, SmsNotifier>();
// IEnumerable<INotifier> — получить все реализации

// TryAdd — не перезаписывает, если уже зарегистрировано
builder.Services.TryAddSingleton<ICache, RedisCache>();
```

### Captive Dependency (захваченная зависимость)

**Singleton НЕ должен зависеть от Scoped напрямую** — это Captive Dependency. Scoped-сервис будет жить столько же, сколько Singleton (вечно), теряя своё scoped-поведение.

```csharp
// НЕПРАВИЛЬНО — DbContext будет жить вечно, утечка соединений
public class CachingService // Singleton
{
    private readonly MyDbContext _db; // Scoped — ОШИБКА!
}

// ПРАВИЛЬНО — создаём scope в методе
public class CachingService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public CachingService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    public async Task RefreshCacheAsync()
    {
        using var scope = _scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<MyDbContext>();
        // Используем db внутри scope, dispose по завершении
    }
}
```

> В Development-режиме ASP.NET Core по умолчанию включает **ValidateScopes** — бросает исключение при Captive Dependency. В Production эта проверка отключена для производительности.

### Keyed Services (.NET 8)

Несколько реализаций одного интерфейса с выбором по ключу:

```csharp
builder.Services.AddKeyedSingleton<ICache, RedisCache>("redis");
builder.Services.AddKeyedSingleton<ICache, MemoryCache>("memory");

// В контроллере или сервисе
public class OrderService
{
    public OrderService([FromKeyedServices("redis")] ICache cache) { }
}
```

### Тонкости DI

- `GetService<T>()` возвращает `null`, если сервис не зарегистрирован; `GetRequiredService<T>()` бросает исключение — **предпочтительнее**
- `IServiceProvider` можно инжектить, но это **Service Locator anti-pattern** — скрывает зависимости, усложняет тестирование
- Открытые generic-типы: `AddScoped(typeof(IRepository<>), typeof(EfRepository<>))` — один вызов для всех `IRepository<T>`
- **Decorator pattern**: встроенный контейнер не поддерживает декораторы напрямую. Используйте Scrutor (`Decorate<IService, LoggingDecorator>()`) или фабричную регистрацию
- `IHostedService` и `BackgroundService` регистрируются как Singleton
- Dispose: контейнер вызывает `Dispose`/`DisposeAsync` для сервисов, которые он создал. Если вы передали готовый экземпляр — Dispose не вызывается

---

## Configuration

ASP.NET Core использует многослойную (layered) систему конфигурации. Последующие источники **перезаписывают** значения из предыдущих.

### Порядок источников (по умолчанию)

```
1. appsettings.json
2. appsettings.{Environment}.json    ← перезаписывает предыдущее
3. User Secrets (Development only)   ← безопасное хранение секретов
4. Environment variables             ← для контейнеров, CI/CD
5. Command-line arguments            ← наивысший приоритет
```

### Переменные окружения

Двойное подчёркивание `__` заменяет вложенность (`:` не работает в некоторых ОС):

```bash
# Эквивалент "Logging:LogLevel:Default": "Warning"
export Logging__LogLevel__Default=Warning
```

Префикс для фильтрации: `ASPNETCORE_` для хостинга, `DOTNET_` для runtime, кастомный префикс через `AddEnvironmentVariables("MyApp_")`.

### Чтение конфигурации

```csharp
// 1. Прямой доступ через IConfiguration
var connStr = configuration["ConnectionStrings:DefaultConnection"];
var connStr2 = configuration.GetConnectionString("DefaultConnection"); // shortcut

// 2. Bind к объекту
var settings = new SmtpSettings();
configuration.GetSection("Smtp").Bind(settings);

// 3. Get<T>() — Bind + создание объекта
var settings2 = configuration.GetSection("Smtp").Get<SmtpSettings>();

// 4. Options pattern (предпочтительный способ)
builder.Services.Configure<SmtpSettings>(configuration.GetSection("Smtp"));
```

### User Secrets

Хранение секретов в Development без коммита в репозиторий:

```bash
dotnet user-secrets init
dotnet user-secrets set "Smtp:Password" "my-secret-password"
```

Файл хранится в `%APPDATA%\Microsoft\UserSecrets\{guid}\secrets.json`. **Не используйте** User Secrets в Production — для production используйте Azure Key Vault, AWS Secrets Manager, HashiCorp Vault.

### Тонкости Configuration

- `IConfiguration` — Singleton, безопасен для инъекции в любой lifetime
- `reloadOnChange: true` в `AddJsonFile` — автоматическое обновление при изменении файла
- Конфигурация доступна **до** построения `IServiceProvider` — можно использовать в `ConfigureServices`
- Секции конфигурации — **case-insensitive**
- Для сложных сценариев — кастомный `IConfigurationProvider` (например, чтение из БД)
- `Configuration.GetChildren()` — получить все дочерние элементы секции (полезно для динамических списков)
