---
tags: [design-patterns, gof, strategy, factory, decorator, observer, builder]
level: Senior
---

# Design Patterns (GoF)

## Что это, зачем и когда

### Что такое Design Patterns?
**Проверенные шаблоны решения типовых задач.** Не код для копирования, а идеи: «если у тебя ТАКАЯ проблема — решай ТАК». Gang of Four (GoF) описали 23 паттерна в 1994, но в современном C# многие реализуются проще.

**Аналогия:** Кулинарные рецепты. Можно каждый раз изобретать борщ с нуля, а можно открыть рецепт. Паттерн — это рецепт: проверенный, понятный другим разработчикам, с известными trade-offs.

### Зачем?

| Без паттернов | С паттернами |
|--------------|-------------|
| «Я написал свою систему уведомлений» — 500 строк багов | Observer / Event — 20 строк, работает |
| «Как создать объект с 15 параметрами?» | Builder — пошаговая сборка |
| «Как добавить логирование ко всем HTTP вызовам?» | Decorator — оборачиваешь, не меняя оригинал |
| «Как переключать алгоритм на лету?» | Strategy — подставляешь нужную реализацию |

### Когда какой?

| Проблема | Паттерн | Пример в .NET |
|----------|---------|---------------|
| Создать сложный объект пошагово | **Builder** | `IHostBuilder`, `WebApplicationBuilder` |
| Создать объект, не зная конкретный тип | **Factory** | `IHttpClientFactory`, `IServiceProvider` |
| Один интерфейс, разные реализации | **Strategy** | `IPaymentProcessor`, DI |
| Добавить поведение без изменения класса | **Decorator** | `DelegatingHandler`, middleware |
| Уведомить подписчиков о событии | **Observer** | `event`, `IObservable<T>`, Domain Events |
| Гарантировать один экземпляр | **Singleton** | `builder.Services.AddSingleton<T>()` |
| Обеспечить единый доступ к коллекции | **Iterator** | `IEnumerable<T>`, `yield return` |

---

## Strategy — подмена алгоритма

**Проблема:** нужно переключать алгоритм (скидки, сортировка, валидация) без `if/switch`.

```csharp
// Интерфейс стратегии
public interface IDiscountStrategy
{
    decimal Calculate(decimal price, int quantity);
}

// Конкретные стратегии
public sealed class NoDiscount : IDiscountStrategy
{
    public decimal Calculate(decimal price, int quantity) => 0;
}

public sealed class BulkDiscount : IDiscountStrategy
{
    public decimal Calculate(decimal price, int quantity)
        => quantity >= 10 ? price * quantity * 0.1m : 0;
}

public sealed class VipDiscount : IDiscountStrategy
{
    public decimal Calculate(decimal price, int quantity)
        => price * quantity * 0.2m;
}

// Контекст — использует стратегию
public sealed class PriceCalculator(IDiscountStrategy discount)
{
    public decimal CalculateTotal(decimal price, int quantity)
    {
        var subtotal = price * quantity;
        return subtotal - discount.Calculate(price, quantity);
    }
}

// DI — подключаем нужную стратегию
builder.Services.AddScoped<IDiscountStrategy, BulkDiscount>();
// Или через Keyed Services:
builder.Services.AddKeyedScoped<IDiscountStrategy, NoDiscount>("regular");
builder.Services.AddKeyedScoped<IDiscountStrategy, VipDiscount>("vip");
```

### Strategy через Func (упрощённый вариант)

```csharp
// Когда не нужен отдельный класс — Func достаточно
public sealed class Sorter<T>
{
    public IReadOnlyList<T> Sort(IEnumerable<T> items, Func<T, object> keySelector)
        => items.OrderBy(keySelector).ToList();
}

// Использование
var sorted = sorter.Sort(orders, o => o.Total);       // по сумме
var sorted = sorter.Sort(orders, o => o.CreatedAt);    // по дате
```

**Когда Strategy:** несколько алгоритмов для одной задачи, выбор в runtime.
**В .NET:** DI + интерфейс = Strategy из коробки.

---

## Factory — создание объектов

**Проблема:** клиент не должен знать конкретный тип создаваемого объекта.

### Factory Method

```csharp
// Абстрактный метод создания
public interface INotificationFactory
{
    INotification Create(string type, string message);
}

public sealed class NotificationFactory : INotificationFactory
{
    public INotification Create(string type, string message) => type switch
    {
        "email" => new EmailNotification(message),
        "sms" => new SmsNotification(message),
        "push" => new PushNotification(message),
        _ => throw new ArgumentOutOfRangeException(nameof(type))
    };
}
```

### Factory через DI (современный подход)

```csharp
// Вместо Factory class — DI резолвит нужную реализацию
builder.Services.AddKeyedScoped<INotification, EmailNotification>("email");
builder.Services.AddKeyedScoped<INotification, SmsNotification>("sms");
builder.Services.AddKeyedScoped<INotification, PushNotification>("push");

// В handler-е
public sealed class SendNotificationHandler(IServiceProvider provider)
{
    public async Task<Result> HandleAsync(string channel, string message, CancellationToken ct)
    {
        var notification = provider.GetKeyedService<INotification>(channel);
        if (notification is null)
            return Result.Fail(Error.Validation("Notification.Unknown", $"Unknown channel: {channel}"));

        await notification.SendAsync(message, ct);
        return Result.Ok();
    }
}
```

### Встроенные Factory в .NET

```csharp
// IHttpClientFactory — не new HttpClient(), а фабрика
builder.Services.AddHttpClient<IPaymentGateway, StripeGateway>();
// Внутри: factory создаёт HttpClient с правильным lifecycle

// IServiceScopeFactory — создание DI scope вручную
public sealed class BackgroundWorker(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var scope = scopeFactory.CreateScope();
        var repo = scope.ServiceProvider.GetRequiredService<IOrderRepository>();
        // ...
    }
}

// IDbContextFactory — создание DbContext вне Scoped lifetime
builder.Services.AddDbContextFactory<AppDbContext>();
```

**Когда Factory:** создание объекта сложное, зависит от конфигурации, или тип определяется в runtime.

---

## Decorator — обёртка поведения

**Проблема:** добавить функциональность (логирование, кеширование, retry) без изменения исходного класса.

```csharp
// Базовый интерфейс
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
}

// Основная реализация
public sealed class OrderRepository(AppDbContext context) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => await context.Orders.FindAsync([id], ct);
}

// Decorator 1: Логирование
public sealed class LoggingOrderRepository(
    IOrderRepository inner,
    ILogger<LoggingOrderRepository> logger) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
    {
        logger.LogInformation("Getting order {OrderId}", id);
        var order = await inner.GetByIdAsync(id, ct);
        if (order is null) logger.LogWarning("Order {OrderId} not found", id);
        return order;
    }
}

// Decorator 2: Кеширование
public sealed class CachedOrderRepository(
    IOrderRepository inner,
    IMemoryCache cache) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => await cache.GetOrCreateAsync($"order:{id}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return await inner.GetByIdAsync(id, ct);
        });
}

// Регистрация — декораторы оборачивают друг друга
builder.Services.AddScoped<OrderRepository>(); // base
builder.Services.AddScoped<IOrderRepository>(sp =>
    new CachedOrderRepository(
        new LoggingOrderRepository(
            sp.GetRequiredService<OrderRepository>(),
            sp.GetRequiredService<ILogger<LoggingOrderRepository>>()),
        sp.GetRequiredService<IMemoryCache>()));
// Цепочка: Cache → Logging → Repository
```

### Decorator в ASP.NET: DelegatingHandler

```csharp
// Добавление заголовка ко всем HTTP запросам
public sealed class ApiKeyHandler(IOptions<ApiSettings> settings)
    : DelegatingHandler
{
    protected override Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        request.Headers.Add("X-Api-Key", settings.Value.Key);
        return base.SendAsync(request, ct); // передаёт дальше по цепочке
    }
}

// Логирование всех HTTP запросов
public sealed class LoggingHandler(ILogger<LoggingHandler> logger)
    : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        logger.LogInformation("HTTP {Method} {Uri}", request.Method, request.RequestUri);
        var response = await base.SendAsync(request, ct);
        logger.LogInformation("HTTP {Status} in {Elapsed}ms",
            response.StatusCode, /* elapsed */);
        return response;
    }
}

// Цепочка DelegatingHandler
builder.Services.AddTransient<ApiKeyHandler>();
builder.Services.AddTransient<LoggingHandler>();
builder.Services
    .AddHttpClient<IExternalApi, ExternalApi>()
    .AddHttpMessageHandler<ApiKeyHandler>()   // 1-й decorator
    .AddHttpMessageHandler<LoggingHandler>(); // 2-й decorator
```

**Когда Decorator:** cross-cutting concerns (логирование, кеш, retry, авторизация) без изменения основного кода.
**В .NET:** Middleware, DelegatingHandler, Pipeline Behaviors — всё это Decorator.

---

## Observer — уведомление подписчиков

**Проблема:** объект изменился — нужно уведомить других, не создавая жёсткой связи.

### C# Events (классический Observer)

```csharp
// Publisher
public sealed class StockTicker
{
    public event EventHandler<PriceChangedEventArgs>? PriceChanged;

    public void UpdatePrice(string symbol, decimal price)
    {
        // ... бизнес-логика ...
        PriceChanged?.Invoke(this, new PriceChangedEventArgs(symbol, price));
    }
}

public sealed record PriceChangedEventArgs(string Symbol, decimal Price) : EventArgs;

// Subscribers
var ticker = new StockTicker();
ticker.PriceChanged += (_, e) => Console.WriteLine($"LOG: {e.Symbol} = {e.Price}");
ticker.PriceChanged += (_, e) =>
{
    if (e.Price > 1000) Console.WriteLine($"ALERT: {e.Symbol} above $1000!");
};
```

### Domain Events (современный Observer)

```csharp
// Вместо C# events — Domain Events через Interceptor (см. ddd.md)
// Publisher: Aggregate Root
public sealed class Order : AggregateRoot<Guid>
{
    public Result Confirm()
    {
        Status = OrderStatus.Confirmed;
        Raise(new OrderConfirmedEvent(Id, Total)); // «публикация»
        return Result.Ok();
    }
}

// Subscriber 1: отправить email
public sealed class SendOrderEmailHandler : IDomainEventHandler<OrderConfirmedEvent>
{
    public async Task HandleAsync(OrderConfirmedEvent @event, CancellationToken ct)
        => await emailService.SendAsync(@event.OrderId, ct);
}

// Subscriber 2: обновить статистику
public sealed class UpdateStatsHandler : IDomainEventHandler<OrderConfirmedEvent>
{
    public async Task HandleAsync(OrderConfirmedEvent @event, CancellationToken ct)
        => await statsService.IncrementAsync(@event.Total, ct);
}
// Добавить subscriber = добавить класс. Publisher не знает о подписчиках.
```

**Когда Observer:** «когда X произошло, нужно сделать Y, Z и, может быть, W» — и список реакций может расти.

---

## Builder — пошаговая сборка

**Проблема:** объект с множеством параметров. Конструктор с 10 аргументами нечитаем.

```csharp
// ✗ Конструктор-монстр
var report = new Report("Q1", 2026, true, false, "PDF", "A4", true, null, "admin");

// ✓ Builder — читаемая пошаговая сборка
public sealed class ReportBuilder
{
    private string _title = "";
    private int _year;
    private string _format = "PDF";
    private string _pageSize = "A4";
    private bool _includeCharts;
    private bool _includeSummary;

    public ReportBuilder WithTitle(string title) { _title = title; return this; }
    public ReportBuilder ForYear(int year) { _year = year; return this; }
    public ReportBuilder InFormat(string format) { _format = format; return this; }
    public ReportBuilder WithCharts() { _includeCharts = true; return this; }
    public ReportBuilder WithSummary() { _includeSummary = true; return this; }

    public Result<Report> Build()
    {
        if (string.IsNullOrWhiteSpace(_title))
            return Result<Report>.Fail(Error.Validation("Report.Title", "Title is required"));

        return Result<Report>.Ok(new Report(_title, _year, _format, _pageSize,
            _includeCharts, _includeSummary));
    }
}

// Использование — читается как предложение
var report = new ReportBuilder()
    .WithTitle("Q1 Sales")
    .ForYear(2026)
    .WithCharts()
    .WithSummary()
    .Build();
```

### Builder в .NET (везде!)

```csharp
// WebApplicationBuilder
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOpenApi();
builder.Services.AddDbContext<AppDbContext>();
var app = builder.Build(); // ← Build() как завершение

// IHostBuilder
Host.CreateDefaultBuilder(args)
    .ConfigureWebHostDefaults(web => web.UseStartup<Startup>())
    .Build()
    .Run();

// ConnectionStringBuilder
var connStr = new NpgsqlConnectionStringBuilder
{
    Host = "localhost",
    Database = "mydb",
    Username = "admin",
    Password = "secret",
    Pooling = true,
    MaxPoolSize = 100
}.ConnectionString;
```

**Когда Builder:** объект с 5+ параметрами, часть опциональна, нужна валидация при сборке.

---

## Singleton — один на всех

В современном .NET Singleton реализуется через DI, **не** через статическое поле.

```csharp
// ✗ Классический Singleton — антипаттерн в .NET
public class OldSingleton
{
    private static readonly OldSingleton _instance = new();
    public static OldSingleton Instance => _instance;
    private OldSingleton() { }
    // Проблемы: не тестируется, не инжектится, скрытая зависимость
}

// ✓ Singleton через DI
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
// DI гарантирует: один экземпляр, тестируемость, explicit dependency

// ✓ Если нужен lazy initialization
builder.Services.AddSingleton<ExpensiveService>(sp =>
{
    // Вызывается один раз, при первом запросе
    var config = sp.GetRequiredService<IOptions<AppSettings>>().Value;
    return new ExpensiveService(config.ConnectionString);
});
```

**Правило:** в .NET Singleton = `AddSingleton()`. Не `static`, не `Instance`, не `Lazy<T>`.

---

## Паттерны — сводная таблица

| Паттерн | Тип | Проблема | .NET реализация |
|---------|-----|----------|-----------------|
| **Strategy** | Behavioral | Переключение алгоритма | Интерфейс + DI |
| **Factory** | Creational | Создание без знания типа | `IHttpClientFactory`, Keyed Services |
| **Decorator** | Structural | Добавить поведение | `DelegatingHandler`, Middleware |
| **Observer** | Behavioral | Уведомить подписчиков | Events, Domain Events |
| **Builder** | Creational | Сложная сборка объекта | `WebApplicationBuilder`, fluent API |
| **Singleton** | Creational | Один экземпляр | `AddSingleton<T>()` |
| **Iterator** | Behavioral | Перебор коллекции | `IEnumerable<T>`, `yield` |
| **Template Method** | Behavioral | Алгоритм с переопределяемыми шагами | `abstract class` + `override` |
| **Adapter** | Structural | Несовместимый интерфейс | Wrapper class |
| **Facade** | Structural | Упростить сложную систему | Extension methods, Service class |

---

## См. также

- [SOLID](../Architecture/solid.md) — Принципы проектирования
- [Delegates и Events](delegates-events.md) — Strategy и Observer через delegates
- [DDD на практике](../Architecture/ddd.md) — Domain Events (Observer в DDD)
- [Resilience и HttpClient](../AspNetCore/resilience.md) — DelegatingHandler (Decorator)
