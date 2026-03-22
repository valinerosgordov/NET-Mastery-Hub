---
tags: [design-patterns, gof, strategy, factory, decorator, observer, builder, chain-of-responsibility, state, adapter, specification]
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

## Chain of Responsibility — цепочка обработчиков

**Проблема:** запрос проходит через несколько обработчиков, каждый решает — обработать или передать дальше.

**Аналогия:** Бюрократия. Заявка идёт: секретарь → начальник отдела → директор. Каждый либо решает, либо передаёт выше.

```csharp
// Это и есть middleware в ASP.NET Core!
app.Use(async (context, next) =>
{
    // Обработчик 1: проверка API ключа
    if (!context.Request.Headers.ContainsKey("X-Api-Key"))
    {
        context.Response.StatusCode = 401;
        return; // Прерываем цепочку
    }
    await next(); // Передаём дальше
});

app.Use(async (context, next) =>
{
    // Обработчик 2: логирование
    var sw = Stopwatch.StartNew();
    await next(); // Передаём дальше
    sw.Stop();
    // Логируем после ответа
    Console.WriteLine($"{context.Request.Path} — {sw.ElapsedMilliseconds}ms");
});
```

### Кастомная цепочка (не middleware)

```csharp
// Пример: обработка заявки на поддержку
public abstract class SupportHandler
{
    private SupportHandler? _next;

    public SupportHandler SetNext(SupportHandler next)
    {
        _next = next;
        return next; // fluent chain
    }

    public virtual Result Handle(SupportTicket ticket)
        => _next?.Handle(ticket)
           ?? Result.Fail(Error.Validation("Support.Unhandled", "No handler for this ticket"));
}

public sealed class BotHandler : SupportHandler
{
    public override Result Handle(SupportTicket ticket)
    {
        if (ticket.Category == "FAQ")
        {
            // Бот отвечает на FAQ
            return Result.Ok();
        }
        return base.Handle(ticket); // передаём дальше
    }
}

public sealed class Level1Handler : SupportHandler
{
    public override Result Handle(SupportTicket ticket)
    {
        if (ticket.Priority < 3)
        {
            // L1 саппорт решает простые тикеты
            return Result.Ok();
        }
        return base.Handle(ticket);
    }
}

public sealed class Level2Handler : SupportHandler
{
    public override Result Handle(SupportTicket ticket)
    {
        // L2 решает всё остальное
        return Result.Ok();
    }
}

// Сборка цепочки
var bot = new BotHandler();
bot.SetNext(new Level1Handler())
   .SetNext(new Level2Handler());

var result = bot.Handle(ticket);
// FAQ → бот. Простой → L1. Сложный → L2.
```

**Когда Chain of Responsibility:** запрос проходит через N обработчиков, порядок важен.
**В .NET:** Middleware, DelegatingHandler, MediatR Pipeline Behaviors — всё это Chain of Responsibility.

---

## Template Method — алгоритм с переопределяемыми шагами

**Проблема:** алгоритм один, но некоторые шаги отличаются в наследниках.

**Аналогия:** Рецепт пиццы. Шаги одинаковые (замесить тесто → раскатать → добавить начинку → запечь), но начинка разная.

```csharp
// Шаблон — абстрактный класс
public abstract class DataImporter
{
    // Template Method — финальный алгоритм, нельзя переопределить
    public async Task<Result> ImportAsync(Stream source, CancellationToken ct)
    {
        var rawData = await ReadAsync(source, ct);      // шаг 1
        var validated = Validate(rawData);                // шаг 2
        if (validated.IsFailure) return validated;
        var transformed = Transform(rawData);             // шаг 3
        await SaveAsync(transformed, ct);                 // шаг 4
        return Result.Ok();
    }

    // Шаги, которые переопределяют наследники
    protected abstract Task<RawData> ReadAsync(Stream source, CancellationToken ct);
    protected abstract Result Validate(RawData data);
    protected abstract ProcessedData Transform(RawData data);
    protected abstract Task SaveAsync(ProcessedData data, CancellationToken ct);
}

// CSV импорт
public sealed class CsvImporter(AppDbContext context) : DataImporter
{
    protected override async Task<RawData> ReadAsync(Stream source, CancellationToken ct)
    {
        using var reader = new StreamReader(source);
        var content = await reader.ReadToEndAsync(ct);
        return new RawData(content.Split('\n'));
    }

    protected override Result Validate(RawData data)
        => data.Lines.Length > 0 ? Result.Ok() : Result.Fail(Error.Validation("Csv.Empty", "File is empty"));

    protected override ProcessedData Transform(RawData data)
        => new(data.Lines.Skip(1).Select(ParseLine).ToList()); // пропустить header

    protected override async Task SaveAsync(ProcessedData data, CancellationToken ct)
    {
        context.Products.AddRange(data.Items);
        await context.SaveChangesAsync(ct);
    }
}

// JSON импорт — тот же алгоритм, другие шаги
public sealed class JsonImporter(AppDbContext context) : DataImporter
{
    protected override async Task<RawData> ReadAsync(Stream source, CancellationToken ct)
        => new(await JsonSerializer.DeserializeAsync<string[]>(source, cancellationToken: ct) ?? []);

    // ... остальные шаги
}
```

**Когда Template Method:** один и тот же алгоритм, но с разными деталями (импорт CSV/JSON/XML, генерация отчётов в разных форматах).

---

## Adapter — совместимость интерфейсов

**Проблема:** внешний сервис / библиотека имеет свой интерфейс, а тебе нужен твой.

**Аналогия:** Переходник для розетки. Европейская вилка не лезет в американскую розетку — нужен адаптер.

```csharp
// Внешний сервис (нельзя менять)
public class StripePaymentSdk
{
    public StripeResult Charge(string cardToken, long amountCents, string currency) { /* ... */ }
}

// Наш интерфейс (domain layer)
public interface IPaymentGateway
{
    Task<Result> ChargeAsync(Money amount, string cardToken, CancellationToken ct);
}

// Adapter — оборачивает чужой интерфейс в наш
public sealed class StripePaymentAdapter(StripePaymentSdk stripe) : IPaymentGateway
{
    public Task<Result> ChargeAsync(Money amount, string cardToken, CancellationToken ct)
    {
        var amountCents = (long)(amount.Amount * 100);
        var stripeResult = stripe.Charge(cardToken, amountCents, amount.Currency);

        return Task.FromResult(stripeResult.Success
            ? Result.Ok()
            : Result.Fail(Error.Internal("Payment.Failed", stripeResult.ErrorMessage)));
    }
}

// DI — переключение платёжной системы = замена одной строки
builder.Services.AddScoped<IPaymentGateway, StripePaymentAdapter>();
// Завтра перешли на PayPal:
// builder.Services.AddScoped<IPaymentGateway, PayPalAdapter>();
```

**Когда Adapter:** интеграция с внешними API, библиотеками, legacy-кодом. Защита домена от чужих типов.
**В .NET:** Typed HttpClient — по сути адаптер над `HttpClient`.

---

## Facade — простой интерфейс для сложной системы

**Проблема:** клиент не хочет знать о 10 сервисах — ему нужна одна точка входа.

**Аналогия:** Официант в ресторане. Тебе не нужно знать про повара, кладовку и мойку — говоришь «стейк средней прожарки» и всё.

```csharp
// Сложная подсистема — 4 сервиса
public interface IInventoryService { Task<bool> CheckAvailabilityAsync(Guid productId, int qty, CancellationToken ct); }
public interface IPaymentGateway { Task<Result> ChargeAsync(Money amount, string token, CancellationToken ct); }
public interface IShippingService { Task<string> CreateShipmentAsync(Address address, CancellationToken ct); }
public interface INotificationService { Task SendOrderConfirmationAsync(Guid orderId, CancellationToken ct); }

// Facade — одна точка входа
public sealed class CheckoutFacade(
    IInventoryService inventory,
    IPaymentGateway payment,
    IShippingService shipping,
    INotificationService notifications,
    IOrderRepository orders,
    IUnitOfWork unitOfWork)
{
    public async Task<Result<CheckoutResult>> ProcessAsync(
        CheckoutRequest request, CancellationToken ct)
    {
        // 1. Проверить наличие
        var available = await inventory.CheckAvailabilityAsync(
            request.ProductId, request.Quantity, ct);
        if (!available)
            return Result<CheckoutResult>.Fail(
                Error.Validation("Checkout.OutOfStock", "Product is out of stock"));

        // 2. Списать оплату
        var chargeResult = await payment.ChargeAsync(request.Amount, request.CardToken, ct);
        if (chargeResult.IsFailure)
            return Result<CheckoutResult>.Fail(chargeResult.Error!);

        // 3. Создать заказ
        var orderResult = Order.Create(request.CustomerId);
        if (orderResult.IsFailure)
            return Result<CheckoutResult>.Fail(orderResult.Error!);

        orders.Add(orderResult.Value!);
        await unitOfWork.SaveChangesAsync(ct);

        // 4. Отправка
        var trackingNumber = await shipping.CreateShipmentAsync(request.Address, ct);

        // 5. Уведомление
        await notifications.SendOrderConfirmationAsync(orderResult.Value!.Id, ct);

        return Result<CheckoutResult>.Ok(
            new CheckoutResult(orderResult.Value!.Id, trackingNumber));
    }
}

// Endpoint знает только о Facade — не о 5 сервисах
app.MapPost("/api/checkout", async (
    CheckoutRequest request,
    CheckoutFacade checkout,
    CancellationToken ct) =>
{
    var result = await checkout.ProcessAsync(request, ct);
    return result.ToResponse(r => TypedResults.Ok(r));
});
```

**Когда Facade:** оркестрация нескольких сервисов, API для внешних клиентов, упрощение подсистемы.

---

## State — машина состояний

**Проблема:** объект ведёт себя по-разному в зависимости от состояния. Без паттерна — гора `if/switch`.

**Аналогия:** Банкомат. В режиме ожидания — принимает карту. С картой — просит PIN. С PIN-ом — показывает баланс. Одна и та же кнопка «ОК» делает разное в зависимости от состояния.

```csharp
// ✗ Без State — switch-ад
public class Order
{
    public string Status { get; set; } = "Draft";

    public void Pay()
    {
        if (Status == "Draft") Status = "Paid";
        else if (Status == "Paid") throw new Exception("Already paid");
        else if (Status == "Cancelled") throw new Exception("Cannot pay cancelled");
        else if (Status == "Shipped") throw new Exception("Already shipped");
        // и так для каждого метода...
    }
}

// ✓ State Pattern — каждое состояние = отдельный класс
public interface IOrderState
{
    Result Pay(Order order);
    Result Ship(Order order);
    Result Cancel(Order order);
}

public sealed class DraftState : IOrderState
{
    public Result Pay(Order order)
    {
        order.TransitionTo(new PaidState());
        return Result.Ok();
    }
    public Result Ship(Order order)
        => Result.Fail(Error.Validation("Order.NotPaid", "Must pay before shipping"));
    public Result Cancel(Order order)
    {
        order.TransitionTo(new CancelledState());
        return Result.Ok();
    }
}

public sealed class PaidState : IOrderState
{
    public Result Pay(Order order)
        => Result.Fail(Error.Validation("Order.AlreadyPaid", "Already paid"));
    public Result Ship(Order order)
    {
        order.TransitionTo(new ShippedState());
        return Result.Ok();
    }
    public Result Cancel(Order order)
    {
        // Нужен refund перед отменой
        order.TransitionTo(new CancelledState());
        return Result.Ok();
    }
}

public sealed class ShippedState : IOrderState
{
    public Result Pay(Order order)
        => Result.Fail(Error.Validation("Order.Shipped", "Already shipped"));
    public Result Ship(Order order)
        => Result.Fail(Error.Validation("Order.Shipped", "Already shipped"));
    public Result Cancel(Order order)
        => Result.Fail(Error.Validation("Order.Shipped", "Cannot cancel shipped order"));
}

public sealed class CancelledState : IOrderState
{
    public Result Pay(Order order)
        => Result.Fail(Error.Validation("Order.Cancelled", "Order is cancelled"));
    public Result Ship(Order order)
        => Result.Fail(Error.Validation("Order.Cancelled", "Order is cancelled"));
    public Result Cancel(Order order)
        => Result.Fail(Error.Validation("Order.Cancelled", "Already cancelled"));
}

// Order делегирует текущему состоянию
public sealed class Order
{
    private IOrderState _state = new DraftState();
    public string Status => _state.GetType().Name.Replace("State", "");

    internal void TransitionTo(IOrderState state) => _state = state;

    public Result Pay() => _state.Pay(this);
    public Result Ship() => _state.Ship(this);
    public Result Cancel() => _state.Cancel(this);
}

// Использование — чистое
var order = new Order();
order.Pay();    // OK: Draft → Paid
order.Ship();   // OK: Paid → Shipped
order.Cancel(); // Error: "Cannot cancel shipped order"
```

### Упрощённый State через enum + switch expression

```csharp
// Когда полноценный State Pattern = оверкилл
public sealed class Order
{
    public OrderStatus Status { get; private set; } = OrderStatus.Draft;

    // Таблица переходов — одно место, легко читать
    public Result Pay() => Transition(OrderStatus.Paid,
        [OrderStatus.Draft]);

    public Result Ship() => Transition(OrderStatus.Shipped,
        [OrderStatus.Paid]);

    public Result Cancel() => Transition(OrderStatus.Cancelled,
        [OrderStatus.Draft, OrderStatus.Paid]);

    private Result Transition(OrderStatus target, OrderStatus[] allowedFrom)
    {
        if (!allowedFrom.Contains(Status))
            return Result.Fail(Error.Validation("Order.InvalidTransition",
                $"Cannot transition from {Status} to {target}"));

        Status = target;
        return Result.Ok();
    }
}
```

**Когда State:** объект с 4+ состояниями и правилами переходов.
**Упрощённый вариант:** enum + таблица переходов (для простых случаев).
**Полный паттерн:** отдельные классы состояний (когда поведение сильно отличается).

---

## Specification — фильтрация запросов

**Проблема:** сложная фильтрация, которая комбинируется: «активные товары дороже $100 из категории Electronics».

```csharp
// Базовая спецификация
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    // Комбинаторы
    public Specification<T> And(Specification<T> other)
        => new AndSpecification<T>(this, other);

    public Specification<T> Or(Specification<T> other)
        => new OrSpecification<T>(this, other);
}

// Конкретные спецификации
public sealed class ActiveProductSpec : Specification<Product>
{
    public override Expression<Func<Product, bool>> ToExpression()
        => p => p.IsActive;
}

public sealed class PriceAboveSpec(decimal minPrice) : Specification<Product>
{
    public override Expression<Func<Product, bool>> ToExpression()
        => p => p.Price > minPrice;
}

public sealed class InCategorySpec(string category) : Specification<Product>
{
    public override Expression<Func<Product, bool>> ToExpression()
        => p => p.Category == category;
}

// And комбинатор
internal sealed class AndSpecification<T>(
    Specification<T> left, Specification<T> right) : Specification<T>
{
    public override Expression<Func<T, bool>> ToExpression()
    {
        var leftExpr = left.ToExpression();
        var rightExpr = right.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            Expression.Invoke(leftExpr, param),
            Expression.Invoke(rightExpr, param));
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

// Использование
var spec = new ActiveProductSpec()
    .And(new PriceAboveSpec(100))
    .And(new InCategorySpec("Electronics"));

var products = await context.Products
    .Where(spec.ToExpression())
    .ToListAsync(ct);
```

### Упрощённая версия (без Specification класса)

```csharp
// Для большинства случаев — extension methods проще
public static class ProductFilters
{
    public static IQueryable<Product> Active(this IQueryable<Product> query)
        => query.Where(p => p.IsActive);

    public static IQueryable<Product> PriceAbove(this IQueryable<Product> query, decimal min)
        => query.Where(p => p.Price > min);

    public static IQueryable<Product> InCategory(this IQueryable<Product> query, string category)
        => query.Where(p => p.Category == category);
}

// Чейнинг — читаемо и просто
var products = await context.Products
    .Active()
    .PriceAbove(100)
    .InCategory("Electronics")
    .ToListAsync(ct);
// Транслируется в один SQL с тремя WHERE условиями
```

**Когда Specification:** сложная бизнес-фильтрация, правила переиспользуются.
**Упрощённый вариант:** extension methods на `IQueryable<T>` (для 90% случаев достаточно).

---

## Null Object — объект-заглушка

**Проблема:** везде проверки `if (service != null)`. Забыл — NullReferenceException.

```csharp
// ✗ Без Null Object — проверки везде
public sealed class OrderHandler
{
    private readonly ILogger? _logger;

    public void Handle(Order order)
    {
        if (_logger != null) _logger.LogInformation("Processing order {Id}", order.Id);
        // ... логика ...
        if (_logger != null) _logger.LogInformation("Order processed");
    }
}

// ✓ Null Object — объект, который «ничего не делает»
public sealed class NullLogger : ILogger
{
    public static readonly NullLogger Instance = new();
    public void Log<TState>(LogLevel logLevel, EventId eventId, TState state,
        Exception? exception, Func<TState, Exception?, string> formatter) { } // пусто
    public bool IsEnabled(LogLevel logLevel) => false;
    public IDisposable BeginScope<TState>(TState state) where TState : notnull
        => NullScope.Instance;
}

// Теперь без проверок на null
public sealed class OrderHandler(ILogger<OrderHandler> logger) // всегда не-null
{
    public void Handle(Order order)
    {
        logger.LogInformation("Processing order {Id}", order.Id);
        // ... логика ...
        logger.LogInformation("Order processed");
    }
}
```

### Null Object в .NET (встроенные)

```csharp
// NullLogger — уже есть в .NET
using Microsoft.Extensions.Logging.Abstractions;
ILogger logger = NullLogger.Instance;        // ничего не логирует
ILogger<T> logger = NullLogger<T>.Instance;  // типизированный

// Array.Empty<T>() — вместо null для пустых коллекций
public IReadOnlyList<Order> GetOrders()
    => hasOrders ? orders : Array.Empty<Order>();
// Вызывающий код: foreach работает без проверки на null

// Enumerable.Empty<T>()
public IEnumerable<Product> Search(string? query)
    => string.IsNullOrWhiteSpace(query)
        ? Enumerable.Empty<Product>()  // не null, а пустая последовательность
        : repository.Search(query);

// Task.CompletedTask — Null Object для async
public Task OnEventAsync()
    => isEnabled ? ProcessEventAsync() : Task.CompletedTask;
```

**Когда Null Object:** вместо `null` — объект-заглушка, чтобы убрать `if (x != null)` отовсюду.
**Правило:** возвращай пустую коллекцию, не `null`. Инжектируй NullLogger, не `null`.

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
| **Chain of Resp.** | Behavioral | Цепочка обработчиков | Middleware, `DelegatingHandler` |
| **Template Method** | Behavioral | Алгоритм с разными шагами | `abstract class` + `override` |
| **Adapter** | Structural | Обернуть чужой интерфейс | Wrapper для внешних API |
| **Facade** | Structural | Упростить сложную систему | Checkout/Orchestration сервис |
| **State** | Behavioral | Поведение зависит от состояния | Классы состояний или enum + переходы |
| **Specification** | Behavioral | Комбинируемая фильтрация | `Expression<Func<T, bool>>`, extension methods |
| **Null Object** | Behavioral | Убрать проверки на null | `NullLogger`, `Array.Empty<T>()` |
| **Iterator** | Behavioral | Перебор коллекции | `IEnumerable<T>`, `yield` |

---

## См. также

- [SOLID](../Architecture/solid.md) — Принципы проектирования
- [Delegates и Events](delegates-events.md) — Strategy и Observer через delegates
- [DDD на практике](../Architecture/ddd.md) — Domain Events (Observer в DDD)
- [Resilience и HttpClient](../AspNetCore/resilience.md) — DelegatingHandler (Decorator)
- [Pipeline и Middleware](../AspNetCore/pipeline-middleware.md) — Chain of Responsibility в ASP.NET
