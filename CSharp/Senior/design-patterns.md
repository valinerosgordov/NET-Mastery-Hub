---
tags: [csharp, design-patterns, senior, solid, dependency-injection, repository, factory, observer]
level: Senior
date: 2026-08-02
---

# Design Patterns в C# — практический обзор

> **SOLID principles, GoF patterns, modern .NET equivalents.** Какие patterns актуальны до сих пор, какие устарели, и где language features заменили patterns. Закрывает пробел: «знаю названия Singleton/Factory/Observer, не понимаю когда применять и какие современные альтернативы».

---

## 0. Как читать

Если впервые — раздел 1 (SOLID) → раздел 4 (creational) → раздел 5 (structural) → раздел 6 (behavioral). Modern alternatives — раздел 8. Production guidance — раздел 11 (best practices), 13 (pitfalls).

GoF patterns deep с code examples — отдельный файл `gof-patterns-extended.md`. Этот — overview + practical decisions.

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. SOLID — фундамент (кратко)

Полный разбор пяти принципов — с production-примерами на Result pattern, DI и Clean Architecture — в [[solid|SOLID, DRY, KISS, YAGNI]]. Дублировать его здесь незачем; ниже — сводка, чтобы дальше говорить о паттернах на общем языке.

| Принцип | Суть | Маркер нарушения |
|---------|------|------------------|
| **S**RP | Одна причина для изменения | «и» в описании класса: «создаёт заказ **и** шлёт email» |
| **O**CP | Расширяй новым кодом, не правь старый | `switch` по типу, растущий с каждой новой фичей |
| **L**SP | Подтип заменяет базовый без сюрпризов | `NotSupportedException` в override; классика — `Square : Rectangle` |
| **I**SP | Маленькие сфокусированные интерфейсы | `Robot`, вынужденный реализовать `Eat()` из fat-интерфейса |
| **D**IP | Зависимость от абстракций, не от конкретики | `new SqlServerRepository()` внутри бизнес-логики |

Связь с паттернами: большинство GoF-паттернов — инструменты соблюдения SOLID. Strategy и Decorator обслуживают OCP, Adapter и Facade — ISP/DIP, Factory — DIP.

> [!question]- Интервью: что такое SOLID и почему важно?
> Acronym для **5 OOP design principles**: 1) **S**ingle Responsibility — class should have one reason to change. 2) **O**pen/Closed — extend через inheritance/polymorphism, не modification. 3) **L**iskov Substitution — derived can substitute base без breaking behavior. 4) **I**nterface Segregation — small focused interfaces, not fat ones. 5) **D**ependency Inversion — depend on abstractions, not concretions. **Why**: testability (DI mocks), maintainability (changes localized), extensibility (new features без rewrite). **Pragmatic application**: don't over-apply (SRP не каждый method отдельный class). Coined by Robert C. Martin (Uncle Bob).

---

## 2. Pattern categories

### 2.1. Three groups (GoF)

```
Creational — object creation
- Factory Method, Abstract Factory
- Singleton
- Builder
- Prototype

Structural — composition
- Adapter, Bridge
- Composite, Decorator
- Facade, Proxy, Flyweight

Behavioral — communication
- Observer, Strategy
- Command, Iterator
- Mediator, Memento
- State, Template Method
- Visitor, Chain of Responsibility
```

### 2.2. Modern .NET — что заменено

| GoF Pattern | .NET equivalent / language feature |
|-------------|-------------------------------------|
| Iterator | `IEnumerable<T>` + `yield return` |
| Observer | `event` + `INotifyPropertyChanged` + Reactive Extensions |
| Strategy | `Func<T>` / `Action<T>` delegate |
| Command | `ICommand` (WPF) / `IRequest` (MediatR) |
| Singleton | DI container `AddSingleton<T>` |
| Factory Method | DI container resolution |
| Visitor | Pattern matching switch (C# 8+) |
| Template Method | Abstract base class + virtual methods (still relevant) |
| Memento | `record` immutability + `with` expressions |

Many "GoF patterns" — language sugar в C#. Don't over-engineer.

### 2.3. Когда patterns

```
✅ Use pattern когда:
  - Recurring problem с known solution
  - Communication shorthand с team
  - Library / framework design
  - Domain complexity warrants abstraction

❌ Не use когда:
  - Simple problem doesn't need it
  - Pattern adds cognitive overhead
  - Language feature solves elegantly
  - Premature abstraction
```

### 2.4. Anti-patterns

```
- God Class — class with too many responsibilities
- Singleton abuse — global state mutability
- Anemic Domain Model — entities только properties, logic в services
- Premature Abstraction — IRepository<T> для every entity
- Pattern Soup — mixing patterns без cause
- Service Locator — anti-pattern (use DI instead)
```

> [!question]- Интервью: какие GoF patterns сегодня устарели?
> 1) **Iterator** — заменён `IEnumerable<T>` + `yield return`. Просто declare. 2) **Observer** — `event` keyword + `INotifyPropertyChanged` для UI; Reactive Extensions для streams. 3) **Strategy** — `Func<T>` / `Action<T>` delegates вместо отдельных classes. 4) **Singleton** — DI container `AddSingleton<T>` вместо global static. 5) **Factory Method** — DI container resolution. 6) **Visitor** — pattern matching `switch` expression (C# 8+) намного cleaner. **Still relevant**: Template Method (abstract + virtual), Composite (file system trees), Decorator (HTTP middleware), Specification (DDD queries), CQRS, Repository (mostly). Pattern thinking важно, но direct GoF reference часто дате code.

---

## 3. SOLID в практике (DI + interfaces)

DIP в production .NET реализуется DI-контейнером — `Microsoft.Extensions.DependencyInjection` встроен, сторонние (Autofac) нужны редко. Полный разбор контейнера — регистрация, lifetimes, scope-ловушки — в [[aspnet-dependency-injection-deep|DI Deep Dive]]; применение принципов на слоях — в [[solid|SOLID, DRY, KISS, YAGNI]]. Резюме для контекста паттернов:

- **Constructor injection — стандарт.** Зависимости видны в сигнатуре, подменяются моками в тестах. С C# 12 — primary constructor: `public class Service(IRepository repo, ILogger<Service> logger)`.
- **Lifetimes:** `Singleton` — stateless-сервисы, кэши, конфигурация; `Scoped` — `DbContext`, репозитории (один на HTTP-запрос); `Transient` — лёгкие объекты без состояния.
- **Interface vs abstract class:** interface — для DI-контрактов; abstract class — для Template Method (общая логика + защищённые точки расширения).
- **Service Locator — анти-паттерн.** `ServiceLocator.Get<T>()` прячет зависимости: их не видно в сигнатуре, тесты требуют глобального state, ошибки всплывают в runtime вместо compile-time. Всегда constructor injection; Service Locator — только legacy / специфичные framework-нужды.

> [!question]- Интервью: чем DI отличается от Service Locator?
> **Dependency Injection** — dependencies passed **explicitly** через constructor (или property). Class declares what it needs. **Service Locator** — class **asks** для dependencies через global accessor: `ServiceLocator.Get<T>()`. **DI advantages**: 1) Explicit dependencies (visible в signature). 2) Testable (mock injected). 3) Compile-time check (constructor params). 4) Lifetime managed by container. **Service Locator anti-pattern**: hidden dependencies, tests need to setup global state, hard to refactor, runtime errors. Best practice: **always DI**, Service Locator только для legacy / specific framework needs.

---

## 4. Creational patterns

### 4.1. Factory Method

```csharp
public interface IDocument
{
    void Print();
}

public abstract class DocumentFactory
{
    public IDocument CreateAndPrint()
    {
        var doc = CreateDocument();   // factory method
        doc.Print();
        return doc;
    }
    
    protected abstract IDocument CreateDocument();
}

public class PdfFactory : DocumentFactory
{
    protected override IDocument CreateDocument() => new PdfDocument();
}
```

**В .NET:** обычно DI container resolves. Rarely write Factory Method explicitly.

### 4.2. Abstract Factory

```csharp
public interface IUIFactory
{
    IButton CreateButton();
    ITextBox CreateTextBox();
}

public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton() => new WinButton();
    public ITextBox CreateTextBox() => new WinTextBox();
}

public class MacUIFactory : IUIFactory
{
    public IButton CreateButton() => new MacButton();
    public ITextBox CreateTextBox() => new MacTextBox();
}

// Use
IUIFactory factory = OperatingSystem.IsWindows() ? new WindowsUIFactory() : new MacUIFactory();
var btn = factory.CreateButton();
```

Used в cross-platform UI frameworks (MAUI, Avalonia).

### 4.3. Singleton

```csharp
// ❌ Old-school manual Singleton
public sealed class OldSingleton
{
    private static readonly Lazy<OldSingleton> _instance = new(() => new OldSingleton());
    public static OldSingleton Instance => _instance.Value;
    private OldSingleton() { }
}

// ✅ Modern — DI container
builder.Services.AddSingleton<IMyService, MyService>();
```

Singleton has bad reputation — global mutable state, testability issues. **Use DI** instead.

### 4.4. Builder

```csharp
// Fluent builder
public class HttpRequestBuilder
{
    private string _url = "";
    private string _method = "GET";
    private Dictionary<string, string> _headers = new();
    
    public HttpRequestBuilder WithUrl(string url) { _url = url; return this; }
    public HttpRequestBuilder WithMethod(string method) { _method = method; return this; }
    public HttpRequestBuilder WithHeader(string key, string value)
    {
        _headers[key] = value;
        return this;
    }
    
    public HttpRequestMessage Build()
    {
        var req = new HttpRequestMessage(new HttpMethod(_method), _url);
        foreach (var (k, v) in _headers) req.Headers.Add(k, v);
        return req;
    }
}

// Use
var request = new HttpRequestBuilder()
    .WithUrl("https://api.example.com")
    .WithMethod("POST")
    .WithHeader("Authorization", "Bearer token")
    .Build();
```

C# 9+ records с `with` expression — alternative для immutable builders:

```csharp
public record HttpConfig(string Url, string Method, Dictionary<string, string> Headers);

var config = new HttpConfig("", "GET", new()) with
{
    Url = "https://api.example.com",
    Method = "POST"
};
```

### 4.5. Object Pool — built-in

```csharp
// .NET provides ObjectPool / ArrayPool
using Microsoft.Extensions.ObjectPool;

var pool = new DefaultObjectPool<StringBuilder>(new StringBuilderPooledObjectPolicy());
var sb = pool.Get();
try
{
    sb.Append("Hello");
    var result = sb.ToString();
}
finally
{
    pool.Return(sb);
}
```

См. [[memory-pooling]].

### 4.6. Lazy initialization

```csharp
public class Service
{
    private readonly Lazy<ExpensiveResource> _resource = new(() => new ExpensiveResource());
    
    public void Use() => _resource.Value.DoSomething();
    // Constructor создаёт Service быстро, ExpensiveResource создаётся при first access
}
```

`Lazy<T>` — built-in, thread-safe by default.

> [!question]- Интервью: почему Singleton anti-pattern?
> 1) **Global mutable state** — testing isolation impossible. Tests share state, flaky. 2) **Hidden dependencies** — class accesses singleton internally, signature doesn't reveal. 3) **Lifetime coupling** — early/late init issues. 4) **Concurrency** — multiple threads access same state. 5) **Hard to mock** — cannot replace для testing. **Modern alternative**: DI container `AddSingleton<IService, Service>()`. Container manages lifetime, tests inject mock. **Legitimate Singletons**: stateless utilities (formatters, encoders), well-justified caches с careful concurrency. Even those — better through DI.

---

## 5. Structural patterns

### 5.1. Adapter

```csharp
// Third-party API
public class LegacyShipping
{
    public void ShipItem(string itemId, double weightInPounds, string addressLine);
}

// Our system
public interface IShippingService
{
    Task<TrackingId> ShipAsync(ShipRequest request);
}

// Adapter
public class LegacyShippingAdapter : IShippingService
{
    private readonly LegacyShipping _legacy;
    
    public LegacyShippingAdapter(LegacyShipping legacy) => _legacy = legacy;
    
    public Task<TrackingId> ShipAsync(ShipRequest request)
    {
        var weightInPounds = request.WeightKg * 2.20462;
        var address = $"{request.Address.Street}, {request.Address.City}";
        _legacy.ShipItem(request.ItemId, weightInPounds, address);
        return Task.FromResult(new TrackingId(Guid.NewGuid().ToString()));
    }
}
```

Wrap legacy API в modern interface.

### 5.2. Decorator

```csharp
public interface IRepository
{
    Task<User?> GetByIdAsync(int id);
}

// Concrete
public class SqlRepository : IRepository
{
    public async Task<User?> GetByIdAsync(int id)
    {
        // SQL query
    }
}

// Decorator — adds caching
public class CachingRepository : IRepository
{
    private readonly IRepository _inner;
    private readonly IMemoryCache _cache;
    
    public CachingRepository(IRepository inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }
    
    public async Task<User?> GetByIdAsync(int id)
    {
        if (_cache.TryGetValue(id, out User? cached)) return cached;
        var user = await _inner.GetByIdAsync(id);
        _cache.Set(id, user);
        return user;
    }
}

// Decorator — adds logging
public class LoggingRepository : IRepository
{
    private readonly IRepository _inner;
    private readonly ILogger _logger;
    
    public LoggingRepository(IRepository inner, ILogger<LoggingRepository> logger)
    {
        _inner = inner;
        _logger = logger;
    }
    
    public async Task<User?> GetByIdAsync(int id)
    {
        _logger.LogInformation("Getting user {Id}", id);
        var user = await _inner.GetByIdAsync(id);
        _logger.LogInformation("Got user {Id}: {Found}", id, user != null);
        return user;
    }
}

// Compose
services.AddScoped<IRepository>(sp =>
{
    var sql = new SqlRepository();
    var cached = new CachingRepository(sql, sp.GetRequiredService<IMemoryCache>());
    var logged = new LoggingRepository(cached, sp.GetRequiredService<ILogger<LoggingRepository>>());
    return logged;
});
```

ASP.NET Core middleware — Decorator pattern на HTTP pipeline level.

### 5.3. Facade

```csharp
// Facade — simplifies complex subsystem
public class OrderFacade
{
    private readonly IInventory _inventory;
    private readonly IPayment _payment;
    private readonly IShipping _shipping;
    private readonly INotification _notification;
    
    public OrderFacade(IInventory inv, IPayment pay, IShipping ship, INotification notif)
    {
        _inventory = inv; _payment = pay; _shipping = ship; _notification = notif;
    }
    
    public async Task<OrderResult> PlaceOrderAsync(OrderRequest request)
    {
        if (!await _inventory.ReserveAsync(request.Items)) return OrderResult.OutOfStock;
        if (!await _payment.ChargeAsync(request.Customer, request.Total)) return OrderResult.PaymentFailed;
        var trackingId = await _shipping.ScheduleAsync(request);
        await _notification.SendAsync(request.Customer, trackingId);
        return OrderResult.Success(trackingId);
    }
}

// Client doesn't deal с 4 services — one method PlaceOrderAsync
```

### 5.4. Proxy

```csharp
public interface IExpensiveResource
{
    string GetData();
}

public class RealResource : IExpensiveResource
{
    public RealResource()
    {
        Console.WriteLine("Loading expensive resource...");
        Thread.Sleep(2000);   // simulated cost
    }
    
    public string GetData() => "Real data";
}

public class LazyProxy : IExpensiveResource
{
    private RealResource? _real;
    
    public string GetData()
    {
        _real ??= new RealResource();   // create on first access
        return _real.GetData();
    }
}

// Use
IExpensiveResource resource = new LazyProxy();
// Constructor cheap
Console.WriteLine(resource.GetData());   // expensive on first call
```

### 5.5. Composite

```csharp
public abstract class FileSystemNode
{
    public string Name { get; }
    protected FileSystemNode(string name) => Name = name;
    
    public abstract long GetSize();
}

public class File : FileSystemNode
{
    private readonly long _size;
    public File(string name, long size) : base(name) => _size = size;
    public override long GetSize() => _size;
}

public class Directory : FileSystemNode
{
    private readonly List<FileSystemNode> _children = new();
    public Directory(string name) : base(name) { }
    public void Add(FileSystemNode child) => _children.Add(child);
    public override long GetSize() => _children.Sum(c => c.GetSize());   // recursive!
}

// Use — tree of files/dirs treated uniformly
var root = new Directory("root");
root.Add(new File("a.txt", 100));
var sub = new Directory("sub");
sub.Add(new File("b.txt", 200));
root.Add(sub);
Console.WriteLine(root.GetSize());   // 300
```

> [!question]- Интервью: что такое Decorator pattern и где используется в .NET?
> Wraps object внутри другого implementing same interface, adding behavior **без modifying** wrapped class. **Pattern**: `interface I` → `BaseImpl : I` → `Decorator : I { I _inner; }` → пере-вызывает `_inner.Method()` + adds logic. **Examples в .NET**: 1) **ASP.NET Core middleware** — chain of decorators on HTTP pipeline (`UseAuthentication` → `UseAuthorization` → `UseRouting` → endpoint). 2) **Stream decorators** — `BufferedStream`, `GZipStream`, `CryptoStream` wrap base Stream. 3) **EF Core query interceptors**. 4) **HttpClient handlers** — DelegatingHandler chain. 5) **Logging/caching/retry** wrappers around repositories. **Composition** через DI: `services.AddScoped<IService, RealService>(); services.Decorate<IService, CachingDecorator>();` (Scrutor).

---

## 6. Behavioral patterns

### 6.1. Observer — `event` keyword

```csharp
// Modern C# — event keyword (Observer Pattern built-in)
public class OrderService
{
    public event EventHandler<OrderCreatedEventArgs>? OrderCreated;
    
    public void PlaceOrder(Order order)
    {
        // ... save
        OrderCreated?.Invoke(this, new OrderCreatedEventArgs(order));
    }
}

// Subscribers
service.OrderCreated += (s, e) => SendEmail(e.Order);
service.OrderCreated += (s, e) => UpdateInventory(e.Order);
```

См. [[delegates-events]].

### 6.2. Strategy — `Func<T>` / `Action<T>`

```csharp
// Old-school Strategy — interface + classes
public interface ITaxCalculator { decimal Calculate(decimal amount); }
public class USTax : ITaxCalculator { public decimal Calculate(decimal amount) => amount * 0.08m; }
public class EUTax : ITaxCalculator { public decimal Calculate(decimal amount) => amount * 0.20m; }

public class Order
{
    private readonly ITaxCalculator _tax;
    public Order(ITaxCalculator tax) => _tax = tax;
    public decimal GetTotal(decimal subtotal) => subtotal + _tax.Calculate(subtotal);
}

// Modern C# — Func<T> delegate
public class Order
{
    private readonly Func<decimal, decimal> _tax;
    public Order(Func<decimal, decimal> tax) => _tax = tax;
    public decimal GetTotal(decimal subtotal) => subtotal + _tax(subtotal);
}

new Order(amount => amount * 0.08m);   // US
new Order(amount => amount * 0.20m);   // EU
```

Strategy с одной method — `Func<T>` cleaner. Multi-method — interface still better.

### 6.3. Command — `ICommand` / MediatR

```csharp
// Simple Command — ICommand для WPF
public interface ICommand
{
    void Execute(object? parameter);
}

// MediatR — request/handler pattern
public record CreateUserCommand(string Email, string Name) : IRequest<int>;

public class CreateUserHandler : IRequestHandler<CreateUserCommand, int>
{
    public async Task<int> Handle(CreateUserCommand request, CancellationToken ct)
    {
        // create user
        return userId;
    }
}

// Dispatch
var userId = await mediator.Send(new CreateUserCommand("a@x.com", "Alice"));
```

MediatR — request/handler-реализация Command pattern поверх DI.

> [!warning]- License: MediatR 13+ — коммерческий
> С 2025 MediatR перешёл к Lucky Penny Software: 13+ — dual-license (RPL-1.5 / commercial), ≤12.x остаётся свободным навсегда — код выше валиден. Дефолт для новых проектов — **свой in-process dispatcher** (~50 строк) или **Mediator** (source-gen, martinothamar) как drop-in с той же `IRequest`-семантикой. Разбор — [[choosing-dependencies|Choosing Dependencies]].

### 6.4. Iterator — `IEnumerable<T>` + `yield`

```csharp
// Built-in Iterator
public IEnumerable<int> Fibonacci()
{
    int a = 0, b = 1;
    while (true)
    {
        yield return a;
        (a, b) = (b, a + b);
    }
}

foreach (var n in Fibonacci().Take(10))
    Console.WriteLine(n);
```

`yield return` — automatic Iterator implementation. См. [[iterators-yield]].

### 6.5. Mediator

```csharp
// Decouples N components
public interface IMediator
{
    Task PublishAsync<TEvent>(TEvent ev);
}

public class OrderCreatedHandler { /* listens to OrderCreated */ }
public class InventoryHandler { /* listens to OrderCreated */ }
public class EmailHandler { /* listens to OrderCreated */ }

// MediatR library
public record OrderCreated(int OrderId) : INotification;

public class EmailHandler : INotificationHandler<OrderCreated>
{
    public Task Handle(OrderCreated notification, CancellationToken ct)
    {
        // send email
    }
}

// All handlers automatically receive
await mediator.Publish(new OrderCreated(orderId));
```

Про лицензию MediatR 13+ и альтернативы (свой dispatcher, Mediator source-gen) — caveat в 6.3.

### 6.6. Template Method — abstract + virtual

```csharp
public abstract class ReportGenerator
{
    public string Generate()
    {
        var data = LoadData();
        var processed = ProcessData(data);
        return FormatOutput(processed);
    }
    
    protected abstract IEnumerable<DataRow> LoadData();
    protected virtual List<DataRow> ProcessData(IEnumerable<DataRow> data) => data.ToList();
    protected abstract string FormatOutput(List<DataRow> data);
}

public class HtmlReport : ReportGenerator
{
    protected override IEnumerable<DataRow> LoadData() => /* DB query */;
    protected override string FormatOutput(List<DataRow> data) => /* HTML */;
}

public class CsvReport : ReportGenerator
{
    protected override IEnumerable<DataRow> LoadData() => /* DB query */;
    protected override string FormatOutput(List<DataRow> data) => /* CSV */;
}
```

Still relevant — abstract base + virtual override points.

### 6.7. State — pattern matching

```csharp
public enum OrderStatus { Pending, Paid, Shipped, Delivered, Cancelled }

public record Order(int Id, OrderStatus Status, decimal Total);

// Old-school — State pattern with classes
public abstract class OrderState
{
    public abstract Order Pay(Order order);
    public abstract Order Ship(Order order);
}

public class PendingState : OrderState
{
    public override Order Pay(Order order) => order with { Status = OrderStatus.Paid };
    public override Order Ship(Order order) => throw new InvalidOperationException();
}

// Modern C# — pattern matching
public static Order Transition(Order order, string action) => (order.Status, action) switch
{
    (OrderStatus.Pending, "pay") => order with { Status = OrderStatus.Paid },
    (OrderStatus.Paid, "ship") => order with { Status = OrderStatus.Shipped },
    (OrderStatus.Shipped, "deliver") => order with { Status = OrderStatus.Delivered },
    (OrderStatus.Pending or OrderStatus.Paid, "cancel") => order with { Status = OrderStatus.Cancelled },
    _ => throw new InvalidOperationException($"Cannot {action} from {order.Status}")
};
```

C# 8+ pattern matching — cleaner для simple state machines.

### 6.8. Visitor — pattern matching

```csharp
public abstract record Shape;
public record Circle(double Radius) : Shape;
public record Square(double Side) : Shape;
public record Rectangle(double Width, double Height) : Shape;

// Old-school Visitor — separate visitor per operation
public interface IShapeVisitor<T>
{
    T Visit(Circle c);
    T Visit(Square s);
    T Visit(Rectangle r);
}

// Modern — pattern matching
public static double Area(Shape shape) => shape switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Square s => s.Side * s.Side,
    Rectangle r => r.Width * r.Height,
    _ => throw new InvalidOperationException()
};

public static double Perimeter(Shape shape) => shape switch
{
    Circle c => 2 * Math.PI * c.Radius,
    Square s => 4 * s.Side,
    Rectangle r => 2 * (r.Width + r.Height),
    _ => throw new InvalidOperationException()
};
```

Pattern matching switch — Visitor для Closed hierarchy. Discriminated unions (preview в C# 15 / .NET 11, GA ~ноябрь 2026) сделают ещё cleaner.

### 6.9. Chain of Responsibility — middleware

```csharp
// ASP.NET Core middleware = Chain of Responsibility
app.UseExceptionHandler();        // first
app.UseAuthentication();
app.UseAuthorization();
app.UseRouting();
app.UseEndpoints(...);            // last

// Each middleware:
public class CustomMiddleware
{
    private readonly RequestDelegate _next;
    
    public CustomMiddleware(RequestDelegate next) => _next = next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        // Pre-processing
        await _next(context);   // delegate to next в chain
        // Post-processing
    }
}
```

> [!question]- Интервью: какие GoF patterns остаются актуальны в современном C#?
> 1) **Template Method** — abstract base + virtual overrides. Still common (ReportGenerator с abstract LoadData/FormatOutput). 2) **Decorator** — wrap interface implementations (logging/caching/retry around repositories, ASP.NET middleware). 3) **Composite** — recursive structures (file system trees, UI hierarchies). 4) **Specification** — composable query predicates (DDD). 5) **Mediator** — decoupling через in-process dispatcher (CQRS, domain events; MediatR — классическая реализация, 13+ коммерческий). 6) **Chain of Responsibility** — ASP.NET middleware pipeline. 7) **Adapter** — wrapping third-party APIs. **Replaced by language**: Iterator (yield), Observer (event), Strategy (`Func<T>`), Singleton (DI), Factory Method (DI), Visitor (pattern matching), State (pattern matching). Pattern thinking важно, GoF reference часто означает не использовать language features.

---

## 7. Domain-Driven Design patterns

### 7.1. Entity vs Value Object

```csharp
// Entity — has identity (Id), can change over time
public class User
{
    public int Id { get; set; }
    public string Email { get; set; }
    public string Name { get; set; }
    
    public override bool Equals(object? obj) => obj is User u && Id == u.Id;
    public override int GetHashCode() => Id.GetHashCode();
}

// Value Object — defined by attributes, immutable
public record Money(decimal Amount, string Currency);
public record Address(string Street, string City, string ZipCode);

// User has Address (Value Object) and id (Entity identity)
public class User
{
    public int Id { get; init; }
    public Address Address { get; init; }
    public Money Balance { get; init; }
}
```

Records — perfect для Value Objects.

### 7.2. Aggregate Root

```csharp
public class Order   // aggregate root
{
    private readonly List<OrderLine> _lines = new();
    
    public int Id { get; init; }
    public IReadOnlyList<OrderLine> Lines => _lines;
    
    // Only AggregateRoot exposes mutation
    public void AddLine(Product product, int quantity)
    {
        if (Status != OrderStatus.Draft) throw new InvalidOperationException();
        _lines.Add(new OrderLine(product, quantity));
    }
    
    public OrderStatus Status { get; private set; }
}

public record OrderLine(Product Product, int Quantity);   // entity inside aggregate
```

Aggregates маинтайн invariants — single transaction unit.

### 7.3. Repository

```csharp
// Generic Repository — mostly outdated, EF Core IS repository
public interface IGenericRepository<T> where T : Entity
{
    Task<T?> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task AddAsync(T entity);
    // ...
}

// Better — specific repository
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
    Task<List<Order>> GetByCustomerAsync(int customerId);
    Task<List<Order>> GetPendingAsync();
    Task SaveAsync(Order order);
}

public class SqlOrderRepository : IOrderRepository
{
    private readonly DbContext _db;
    public SqlOrderRepository(DbContext db) => _db = db;
    
    public Task<Order?> GetByIdAsync(int id) => _db.Orders.FindAsync(id).AsTask();
    // ...
}
```

Specific repositories — better intent. Generic Repository — abstraction over abstraction.

### 7.4. Specification

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();
    
    public Specification<T> And(Specification<T> other) => new AndSpec<T>(this, other);
    public Specification<T> Or(Specification<T> other) => new OrSpec<T>(this, other);
}

public class IsAdultUser : Specification<User>
{
    public override Expression<Func<User, bool>> ToExpression() => u => u.Age >= 18;
}

public class IsActiveUser : Specification<User>
{
    public override Expression<Func<User, bool>> ToExpression() => u => u.IsActive;
}

// Use
var spec = new IsAdultUser().And(new IsActiveUser());
var users = ctx.Users.Where(spec.ToExpression()).ToList();
```

См. [[reflection-expression-trees]] раздел 6.

### 7.5. Domain Events

```csharp
public abstract record DomainEvent(DateTime OccurredOn);
public record OrderPlaced(int OrderId, decimal Total, DateTime OccurredOn) : DomainEvent(OccurredOn);

public class Order
{
    private readonly List<DomainEvent> _events = new();
    public IReadOnlyList<DomainEvent> Events => _events;
    
    public void Place()
    {
        Status = OrderStatus.Placed;
        _events.Add(new OrderPlaced(Id, Total, DateTime.UtcNow));
    }
    
    public void ClearEvents() => _events.Clear();
}

// In EF Core SaveChanges interceptor — publish events после commit
foreach (var entity in changedEntities)
{
    if (entity is IHasDomainEvents hasEvents)
    {
        foreach (var ev in hasEvents.Events)
            await mediator.Publish(ev);
    }
}
```

### 7.6. CQRS — Command Query Responsibility Segregation

```csharp
// Commands — modify state
public record CreateUserCommand(string Email, string Name) : IRequest<int>;

public class CreateUserHandler : IRequestHandler<CreateUserCommand, int>
{
    public async Task<int> Handle(CreateUserCommand cmd, CancellationToken ct) { /* ... */ }
}

// Queries — read state
public record GetUserByIdQuery(int Id) : IRequest<UserDto?>;

public class GetUserByIdHandler : IRequestHandler<GetUserByIdQuery, UserDto?>
{
    public async Task<UserDto?> Handle(GetUserByIdQuery query, CancellationToken ct) { /* ... */ }
}

// Use
var userId = await mediator.Send(new CreateUserCommand("a@x.com", "Alice"));
var user = await mediator.Send(new GetUserByIdQuery(userId));
```

CQRS — separate read и write paths. Dispatch — свой dispatcher / Mediator (source-gen) / MediatR ≤12.x (13+ коммерческий, см. [[choosing-dependencies|Choosing Dependencies]]). Скейлится — отдельные DBs, projections.

> [!question]- Интервью: чем Entity отличается от Value Object?
> **Entity** — has **identity** (Id), distinct existence over time. Two Users с same Name — different entities (different Ids). Mutable state (within aggregate). **Value Object** — defined by **attributes**, no identity. Two Money(100, "USD") — equal. Immutable. **Examples**: Entity — User, Order, Product. Value Object — Money, Address, Coordinates. **C# implementation**: Entity — class с Id, Equals overridden by Id. Value Object — `record` (auto value equality). **DDD principle**: identify Entities, treat Value Objects as immutable. Aggregate Root maintains consistency boundary.

---

## 8. Modern .NET — language replaces patterns

### 8.1. Sum types через records + sealed

```csharp
// "Sum type" emulation
public abstract record Result<T>;
public record Success<T>(T Value) : Result<T>;
public record Failure<T>(string Error) : Result<T>;

Func<int, int, Result<int>> divide = (a, b) =>
    b == 0 ? new Failure<int>("Div by zero") : new Success<int>(a / b);

var result = divide(10, 2);
var msg = result switch
{
    Success<int> s => $"Got {s.Value}",
    Failure<int> f => $"Error: {f.Error}",
    _ => throw new InvalidOperationException()   // compiler не знает, что hierarchy закрыта
};
```

Discriminated unions (preview в C# 15 / .NET 11) сделают ещё cleaner — native `union` с compiler-enforced exhaustiveness, без `_`-ветки.

### 8.2. Functional pipeline через LINQ

```csharp
// Imperative
var result = new List<int>();
foreach (var x in data)
{
    if (x > 0)
    {
        var y = x * 2;
        result.Add(y);
    }
}

// Functional
var result2 = data
    .Where(x => x > 0)
    .Select(x => x * 2)
    .ToList();
```

LINQ — functional style alternative many imperative loops.

### 8.3. Immutable types через records

```csharp
public record User(int Id, string Name, string Email);

var u1 = new User(1, "Alice", "a@x.com");
var u2 = u1 with { Name = "Bob" };   // immutable evolution
```

Builder pattern often не нужен с records.

### 8.4. Memento через records

```csharp
public record GameSnapshot(int Score, int Level, List<string> Inventory);

public class Game
{
    public int Score { get; private set; }
    public int Level { get; private set; }
    public List<string> Inventory { get; } = new();
    
    public GameSnapshot Save() => new(Score, Level, new(Inventory));
    
    public void Restore(GameSnapshot snapshot)
    {
        Score = snapshot.Score;
        Level = snapshot.Level;
        Inventory.Clear();
        Inventory.AddRange(snapshot.Inventory);
    }
}

// Save / restore
var save = game.Save();
game.PlayDangerously();
if (game.Lost) game.Restore(save);
```

### 8.5. Observer через INotifyPropertyChanged + Reactive

```csharp
// Modern Observer — Rx (Reactive Extensions)
using System.Reactive.Subjects;
using System.Reactive.Linq;

public class StockTicker
{
    private readonly Subject<PriceChange> _prices = new();
    public IObservable<PriceChange> Prices => _prices;
    
    public void UpdatePrice(string symbol, decimal newPrice)
    {
        _prices.OnNext(new PriceChange(symbol, newPrice));
    }
}

// Subscribe with filtering
var ticker = new StockTicker();
ticker.Prices
    .Where(p => p.Symbol == "AAPL")
    .Throttle(TimeSpan.FromSeconds(1))
    .Subscribe(p => Console.WriteLine($"Throttled: {p.Symbol} {p.Price}"));
```

### 8.6. Source generators replace Reflection

```csharp
// Old — reflection-based JSON
JsonConvert.SerializeObject(user);

// Modern — source generator (compile-time)
[JsonSerializable(typeof(User))]
internal partial class MyContext : JsonSerializerContext { }

var json = JsonSerializer.Serialize(user, MyContext.Default.User);
```

См. [[source-generators]].

> [!question]- Интервью: какие language features заменяют patterns в современном C#?
> 1) **`yield return`** заменяет Iterator pattern. 2) **`event` + `EventHandler<T>`** заменяет Observer. 3) **`Func<T>` / `Action<T>`** заменяют single-method Strategy. 4) **DI container `AddSingleton<T>`** заменяет manual Singleton. 5) **`record` + `with`** заменяет Memento, partial Builder. 6) **Pattern matching `switch`** заменяет Visitor + State. 7) **Source generators** заменяют reflection-based code generation. 8) **LINQ** заменяет many Iterator + functional combinators. 9) **`Lazy<T>`** заменяет Lazy Initialization pattern. **Direction**: language features make patterns implicit. Knowing pattern names — communication tool. Writing them explicitly — often anti-pattern в modern C#.

---

## 9. SOLID violations + refactoring

### 9.1. God Class

```csharp
// ❌ Violates SRP severely
public class OrderProcessor
{
    public void ProcessOrder(Order o) { /* 500 lines */ }
    public string GenerateInvoice(Order o) { /* 200 lines */ }
    public void SendEmail(string to, string body) { /* SMTP */ }
    public void LogEvent(string msg) { /* file IO */ }
    public List<Order> SearchOrders(...) { /* SQL */ }
    public void UpdateInventory(Order o) { /* ... */ }
    public decimal CalculateTax(Order o) { /* ... */ }
    public void ScheduleDelivery(Order o) { /* ... */ }
    // ...
}

// ✅ Split по responsibility
public class OrderService { /* orchestration */ }
public class InvoiceGenerator { /* invoice */ }
public class EmailService { /* email */ }
public class OrderRepository { /* persistence */ }
public class InventoryService { /* inventory */ }
public class TaxCalculator { /* tax */ }
public class DeliveryScheduler { /* delivery */ }
```

### 9.2. Anemic Domain Model

```csharp
// ❌ Anemic — only data
public class Order
{
    public int Id { get; set; }
    public List<OrderLine> Lines { get; set; }
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
}

// All business logic в service
public class OrderService
{
    public void Pay(Order order, decimal amount)
    {
        if (order.Status != OrderStatus.Pending) throw new InvalidOperationException();
        order.Status = OrderStatus.Paid;
        order.Total = amount;
    }
}

// ✅ Rich Domain Model
public class Order
{
    public int Id { get; private set; }
    public IReadOnlyList<OrderLine> Lines => _lines;
    public decimal Total { get; private set; }
    public OrderStatus Status { get; private set; }
    
    private readonly List<OrderLine> _lines = new();
    
    public void Pay(decimal amount)
    {
        if (Status != OrderStatus.Pending) throw new InvalidOperationException();
        Status = OrderStatus.Paid;
        Total = amount;
    }
    
    public void AddLine(Product product, int quantity)
    {
        if (Status != OrderStatus.Pending) throw new InvalidOperationException();
        _lines.Add(new OrderLine(product, quantity));
    }
}
```

### 9.3. Tight coupling

```csharp
// ❌ Coupled to concrete
public class OrderService
{
    private SqlServerOrderRepository _repo = new();   // can't test, can't swap
}

// ✅ Inject interface
public class OrderService(IOrderRepository repo)
{
    // mock в tests, swap implementations
}
```

### 9.4. Service Locator → DI

```csharp
// ❌
public class Service
{
    public void Process()
    {
        var repo = ServiceLocator.Resolve<IRepository>();
        repo.Save();
    }
}

// ✅
public class Service(IRepository repo)
{
    public void Process() => repo.Save();
}
```

### 9.5. Premature abstraction

```csharp
// ❌ IRepository<T> для каждого entity без variation
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(int id);
}
public class UserRepository : IRepository<User> { }
public class OrderRepository : IRepository<Order> { }
public class ProductRepository : IRepository<Product> { }
// ... 50 more

// ✅ EF Core IS the repository — DbContext
public class AppDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }
}

// Inject DbContext directly если specific queries не нужны
```

> [!question]- Интервью: что такое Anemic Domain Model и почему anti-pattern?
> Domain model где **entities — только bags of properties** (data only, no behavior). Business logic — в "services" (procedural code). **Why anti-pattern в DDD**: 1) **Encapsulation broken** — anyone может set entity fields. Invariants violated. 2) **Logic scattered** — same business rules в multiple services. 3) **Service classes grow** — God services с все business logic. 4) **Object-oriented advantages lost** — methods belong на entity. **Rich Domain Model**: entity has private setters, public methods enforce invariants (Order.Pay() rather than service.PayOrder(order)). **However**: pragmatic mix often best — DTOs/records для transport (anemic OK), domain entities для business logic. Don't dogmatically apply DDD — overhead не оправдывает в simple CRUD.

---

## 10. Best practices

### 10.1. Pattern application

- ✅ **Recurring problems** — apply pattern.
- ✅ **Communication shorthand** with team.
- ✅ **Library / framework design** — patterns common.
- ✅ **Modern C# features first** — language often replaces pattern.
- ❌ **Apply pattern для name's sake** — over-engineering.
- ❌ **Mix many patterns** в same class.

### 10.2. SOLID application

- ✅ **DI for dependencies** — always.
- ✅ **Interfaces for contracts** при > 2 implementations possible.
- ✅ **Small classes** — SRP.
- ✅ **Composition over inheritance**.
- ❌ **Inheritance trees > 3 levels** — review.
- ❌ **God classes** — split.

### 10.3. Modern .NET patterns

- ✅ **DI container** для object creation.
- ✅ **Records** для DTOs / Value Objects.
- ✅ **Pattern matching** для switch на types.
- ✅ **Async/await** + `Task<T>` для async patterns.
- ✅ **Source generators** вместо reflection где возможно.
- ❌ **Manual Singleton** — DI instead.
- ❌ **Iterator pattern manually** — `yield return`.
- ❌ **Observer pattern manually** — `event`.

### 10.4. DDD pragmatic

- ✅ **Records для Value Objects**.
- ✅ **Aggregate roots** — encapsulate invariants.
- ✅ **Specific Repositories** — not generic `IRepository<T>`.
- ✅ **Domain Events** для loose coupling.
- ✅ **CQRS** для complex read/write paths.
- ❌ **DDD для simple CRUD** — overhead не оправдан.
- ❌ **Anemic Domain Model**.

### 10.5. Не делай

- ❌ Pattern soup без cause.
- ❌ Singleton abuse.
- ❌ Service Locator.
- ❌ God classes.
- ❌ Premature abstraction (`IRepository<T>` везде).

---

## 11. Decision tree

```
Какой pattern?
│
├── Object creation
│   ├── Simple → DI container (AddTransient/Scoped/Singleton)
│   ├── Complex configuration → Builder
│   ├── Family of related → Abstract Factory
│   ├── Per-T instances → Generic Factory<T>
│   └── Resource pooling → ObjectPool / ArrayPool
│
├── Object structure
│   ├── Wrap legacy API → Adapter
│   ├── Add cross-cutting (cache/log/retry) → Decorator
│   ├── Simplify subsystem → Facade
│   ├── Tree structure → Composite
│   └── Lazy / proxy → Lazy<T> / Proxy
│
├── Communication
│   ├── Notifications → event / EventHandler<T>
│   ├── Strategy → Func<T>
│   ├── Command/Query → свой dispatcher / Mediator (source-gen); MediatR 13+ коммерческий
│   ├── Iteration → IEnumerable<T> + yield
│   ├── State transitions → pattern matching switch
│   ├── Type dispatch → pattern matching switch (replaces Visitor)
│   └── Pipeline → middleware (Chain of Responsibility)
│
├── DDD
│   ├── Entity (has Id, mutable) → class with private setters
│   ├── Value Object (immutable) → record
│   ├── Aggregate Root → enforce invariants
│   ├── Repository → specific (IOrderRepository), not generic
│   ├── Specification → composable Expression<Func<T, bool>>
│   ├── Domain Event → свой dispatcher / Mediator INotification (source-gen)
│   └── CQRS → Commands + Queries через свой dispatcher; MediatR 13+ коммерческий
│
└── Anti-pattern check
    ├── God class → split by SRP
    ├── Anemic Domain → move logic to entities
    ├── Service Locator → constructor injection
    ├── Manual Singleton → AddSingleton
    └── Premature abstraction → use EF Core directly
```

---

## 12. Cheat sheet

```csharp
// === SOLID ===
// SRP: one responsibility per class
// OCP: extend through inheritance/interfaces
// LSP: subtype substitutable for base
// ISP: small focused interfaces
// DIP: depend on abstractions

// === DI ===
services.AddSingleton<IService, Service>();
services.AddScoped<IRepo, Repo>();
services.AddTransient<IHelper, Helper>();

// === Decorator (с Scrutor) ===
services.AddScoped<IRepository, SqlRepository>();
services.Decorate<IRepository, CachingRepository>();
services.Decorate<IRepository, LoggingRepository>();

// === Builder ===
var request = new HttpRequestBuilder()
    .WithUrl("https://api.com")
    .WithMethod("POST")
    .Build();

// === Modern alternatives ===
// Iterator
public IEnumerable<int> Range(int n) { for (int i = 0; i < n; i++) yield return i; }

// Observer
public event EventHandler<MyEventArgs>? OnEvent;
OnEvent?.Invoke(this, new MyEventArgs());

// Strategy
Func<int, int> doubler = x => x * 2;

// Singleton
services.AddSingleton<ICache, RedisCache>();

// Visitor / State
var msg = obj switch
{
    Foo f => $"foo {f.X}",
    Bar b => $"bar {b.Y}",
    _ => "unknown"
};

// === DDD ===
// Value Object
public record Money(decimal Amount, string Currency);

// Entity
public class Order
{
    public int Id { get; private set; }
    private List<OrderLine> _lines = new();
    public IReadOnlyList<OrderLine> Lines => _lines;
    
    public void Pay() { /* invariants */ }
}

// Specification
public class IsAdult : Specification<User>
{
    public override Expression<Func<User, bool>> ToExpression() => u => u.Age >= 18;
}

// CQRS — IRequest-семантика (MediatR <=12.x / Mediator source-gen; MediatR 13+ commercial)
public record CreateUserCommand(string Email) : IRequest<int>;
public class Handler : IRequestHandler<CreateUserCommand, int>
{
    public Task<int> Handle(CreateUserCommand cmd, CancellationToken ct) => /* ... */;
}
```

---

## 13. Common pitfalls

### 13.1. Singleton abuse

```csharp
// ❌ Singleton с mutable state
public class Logger
{
    private static Logger _instance = new();
    public static Logger Instance => _instance;
    public List<string> Messages { get; } = new();   // ❌ shared mutable
}
```

**Фикс:** DI + `ILogger<T>` with appropriate lifetime.

### 13.2. Generic `Repository<T>`

```csharp
// ❌ Generic repository — over-abstraction
public interface IRepository<T> where T : Entity
{
    Task<T?> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task AddAsync(T entity);
}

// Hides EF Core power
```

**Фикс:** specific repositories или use DbContext directly.

### 13.3. Service Locator hidden

```csharp
public class Service
{
    public void Do()
    {
        var repo = ServiceProvider.GetRequiredService<IRepo>();   // ❌
    }
}
```

**Фикс:** constructor injection.

### 13.4. Pattern over-engineering

```csharp
// ❌ Strategy + Factory + Builder для simple thing
public class TaxStrategyFactoryBuilder
{
    public ITaxStrategyBuilder Configure(...);
}
public interface ITaxStrategyBuilder { ITaxStrategy Build(); }
public interface ITaxStrategy { decimal Calculate(decimal amount); }
public class USTaxStrategy : ITaxStrategy { ... }
// 4 classes для x => x * 0.08m
```

**Фикс:** `Func<decimal, decimal>` lambda.

### 13.5. Visitor manual

```csharp
// ❌ Manual Visitor pattern
public interface IShapeVisitor<T> { T VisitCircle(Circle c); T VisitSquare(Square s); }
```

**Фикс:** pattern matching switch.

### 13.6. God Service

```csharp
public class OrderService
{
    public void Place();
    public void Pay();
    public void Cancel();
    public void Ship();
    public void GenerateInvoice();
    public void SendEmail();
    public void UpdateInventory();
    public void Log();
    // ...
}
```

**Фикс:** split по responsibilities.

### 13.7. Anemic with services

```csharp
public class Order { public int Id; public OrderStatus Status; }   // только data
public class OrderService { public void Pay(Order o) { o.Status = Paid; } }   // logic
```

**Фикс:** rich entity (Order.Pay() method).

### 13.8. Inheritance overuse

```csharp
public class A { }
public class B : A { }
public class C : B { }
public class D : C { }
public class E : D { }
// 5+ levels
```

**Фикс:** composition over inheritance.

### 13.9. Premature interfaces

```csharp
// ❌ Interface для каждого class
public interface IUserRepository { ... }
public class UserRepository : IUserRepository { ... }
// Один implementation, никакой mock не нужен (using EF)
```

**Фикс:** add interface когда second implementation возникает.

### 13.10. Mutable singletons

```csharp
public class CacheSingleton
{
    public Dictionary<string, object> Cache { get; } = new();   // ❌ thread-unsafe
}
```

**Фикс:** ConcurrentDictionary, lock, или different lifetime.

> [!question]- Интервью: топ-3 pattern anti-patterns?
> 1) **Singleton abuse** — global mutable state, untestable, concurrency issues. Use DI container с appropriate lifetime. 2) **Service Locator** — hidden dependencies, untestable, fragile. Constructor injection всегда. 3) **Anemic Domain Model** — entities only data, all logic в services. Rich domain (entity methods enforce invariants). Бонус: **Premature Abstraction** — `IRepository<T>` для каждого entity без actual variation. EF Core IS the repository. Add interfaces когда need second implementation.

---

## 14. Practice exercises

### 14.1. Refactor God Service

```csharp
// Before
public class OrderProcessor
{
    public void ProcessOrder(int orderId)
    {
        // 1. Validate inventory
        // 2. Calculate total с tax
        // 3. Save to DB
        // 4. Charge customer
        // 5. Send confirmation email
        // 6. Update inventory
        // 7. Schedule delivery
        // 8. Log event
        // ... 500 lines
    }
}

// After — split by responsibility
public class OrderProcessor(
    IInventoryService inventory,
    ITaxCalculator tax,
    IOrderRepository repo,
    IPaymentService payment,
    IEmailService email,
    IDeliveryScheduler delivery,
    ILogger<OrderProcessor> logger)
{
    public async Task ProcessAsync(int orderId)
    {
        var order = await repo.GetByIdAsync(orderId)
            ?? throw new NotFoundException();
        
        if (!await inventory.ValidateAsync(order.Items))
            throw new OutOfStockException();
        
        order.SetTotal(tax.Calculate(order.Subtotal));
        
        await repo.SaveAsync(order);
        await payment.ChargeAsync(order.Customer, order.Total);
        await email.SendConfirmationAsync(order);
        await inventory.ReserveAsync(order.Items);
        await delivery.ScheduleAsync(order);
        
        logger.LogInformation("Order {Id} processed", orderId);
    }
}
```

### 14.2. Decorator chain

```csharp
public interface IDataService
{
    Task<Data> GetAsync(int id);
}

public class SqlDataService : IDataService
{
    public async Task<Data> GetAsync(int id) => /* SQL query */;
}

public class CachingDataService : IDataService
{
    private readonly IDataService _inner;
    private readonly IMemoryCache _cache;
    public CachingDataService(IDataService inner, IMemoryCache cache)
    {
        _inner = inner; _cache = cache;
    }
    public async Task<Data> GetAsync(int id)
    {
        if (_cache.TryGetValue(id, out Data cached)) return cached;
        var data = await _inner.GetAsync(id);
        _cache.Set(id, data, TimeSpan.FromMinutes(5));
        return data;
    }
}

public class RetryDataService : IDataService
{
    private readonly IDataService _inner;
    public RetryDataService(IDataService inner) => _inner = inner;
    public async Task<Data> GetAsync(int id)
    {
        for (int i = 0; i < 3; i++)
        {
            try { return await _inner.GetAsync(id); }
            catch (TransientException) when (i < 2) { await Task.Delay(100); }
        }
        throw new InvalidOperationException();
    }
}

// DI registration с Scrutor
services.AddScoped<IDataService, SqlDataService>();
services.Decorate<IDataService, CachingDataService>();
services.Decorate<IDataService, RetryDataService>();
// Order: Retry → Caching → SQL
```

### 14.3. CQRS с MediatR

Код ниже — MediatR ≤12.x (остаётся свободным); для новых проектов та же семантика доступна через свой dispatcher или Mediator (source-gen) — [[choosing-dependencies|Choosing Dependencies]].

```csharp
// Command
public record PlaceOrderCommand(int CustomerId, List<OrderLineRequest> Lines) : IRequest<int>;

public class PlaceOrderHandler(
    IOrderRepository orders,
    IInventoryService inventory,
    IMediator mediator)
    : IRequestHandler<PlaceOrderCommand, int>
{
    public async Task<int> Handle(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = new Order(cmd.CustomerId);
        foreach (var line in cmd.Lines)
            order.AddLine(line.ProductId, line.Quantity);
        
        await orders.AddAsync(order);
        
        // Domain event published
        await mediator.Publish(new OrderPlaced(order.Id, order.Total), ct);
        
        return order.Id;
    }
}

// Query
public record GetOrderByIdQuery(int OrderId) : IRequest<OrderDto?>;

public class GetOrderHandler(IDbContext db) : IRequestHandler<GetOrderByIdQuery, OrderDto?>
{
    public async Task<OrderDto?> Handle(GetOrderByIdQuery query, CancellationToken ct)
    {
        return await db.Orders
            .Where(o => o.Id == query.OrderId)
            .Select(o => new OrderDto(o.Id, o.Total, o.Status))
            .FirstOrDefaultAsync(ct);
    }
}

// Notification (Domain Event)
public record OrderPlaced(int OrderId, decimal Total) : INotification;

public class SendOrderEmailHandler : INotificationHandler<OrderPlaced>
{
    public Task Handle(OrderPlaced notification, CancellationToken ct)
    {
        // send email
    }
}

public class UpdateInventoryHandler : INotificationHandler<OrderPlaced>
{
    public Task Handle(OrderPlaced notification, CancellationToken ct)
    {
        // decrement stock
    }
}
```

---

## 15. Что читать дальше

1. **[[gof-patterns-extended|GoF Patterns Extended]]** — все 23 patterns с code.
2. **[[functional-csharp|Functional C#]]** — functional alternatives.
3. **Mediator (source-gen)** — github.com/martinothamar/Mediator — свободная drop-in замена MediatR.
4. **Vladimir Khorikov — DDD lectures** (Pluralsight).
5. **Eric Evans — Domain-Driven Design** (book — original).
6. **Robert Martin — Clean Architecture** (book).

---

## 16. См. также

- [[gof-patterns-extended|GoF Patterns Extended]]
- [[oop|OOP]] — class hierarchies
- [[functional-csharp|Functional C#]]
- [[reflection-expression-trees|Reflection]] — Specification pattern
- [[csharp-language-design|Language Design]] — features replace patterns
- MediatR — github.com/LuckyPennySoftware/MediatR (13+ коммерческий; ≤12.x свободный) — см. [[choosing-dependencies|Choosing Dependencies]]
- Mediator (source-gen) — github.com/martinothamar/Mediator
- Scrutor — github.com/khellang/Scrutor

---

## 17. Reading list

- **Robert Martin — Clean Architecture** (2017) — book, modern OOP
- **Robert Martin — Clean Code** — practices
- **Eric Evans — Domain-Driven Design** (2003) — DDD original
- **Vaughn Vernon — Implementing DDD** (2013) — practical DDD
- **Mark Seemann — Dependency Injection** (book) — DI in .NET
- **Jimmy Bogard — Domain Events** — jimmybogard.com
- **Vladimir Khorikov — Pluralsight** — DDD courses
- **Steve Smith — ASP.NET Core patterns** — ardalis.com
- **Microsoft Docs — Design patterns** — learn.microsoft.com
- **Refactoring.Guru** — refactoring.guru/design-patterns/csharp
- **Sourcemaking** — sourcemaking.com/design_patterns
- **GoF — Design Patterns** (1994) — original book
