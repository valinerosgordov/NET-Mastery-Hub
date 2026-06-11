---
tags: [csharp, gof, design-patterns, senior, creational, structural, behavioral, encyclopedia]
level: Senior
date: 2026-05-10
---

# GoF Patterns Extended — все 23 паттерна с примерами

> **Полный справочник 23 GoF design patterns с C# implementation, modern .NET equivalents, real-world usage.** Reference companion к `design-patterns.md` (overview + SOLID + DDD). Закрывает пробел: «нужен code example каждого pattern, intent, structure, when not to use».

---

## 0. Как читать

**Формат каждого pattern**: Intent → Problem → C# implementation → Modern alternative → When apply / When avoid → Real-world examples.

Раздел 1 — Creational (5 patterns). Раздел 2 — Structural (7 patterns). Раздел 3 — Behavioral (11 patterns). Раздел 4 — comparison + decision tree.

Для overview SOLID + DDD + modern alternatives → `design-patterns.md`.

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Creational Patterns

### 1.1. Singleton

**Intent**: Ensure class has only one instance, provide global access.

```csharp
// Classic — Lazy<T> thread-safe
public sealed class Logger
{
    private static readonly Lazy<Logger> _instance = new(() => new Logger());
    public static Logger Instance => _instance.Value;
    
    private Logger() { }
    
    public void Log(string message) => Console.WriteLine($"[{DateTime.Now}] {message}");
}

// Use
Logger.Instance.Log("Hello");

// Modern — DI container (preferred)
services.AddSingleton<ILogger, Logger>();
```

**Modern alternative**: `services.AddSingleton<T>()` через DI container.

**When apply**:
- Stateless services (formatters, encoders).
- Caches с careful concurrency.
- Configuration objects.

**When avoid**:
- Mutable global state (testing нightmare).
- Hidden dependencies.
- Anything that should be testable.

**Real .NET**: `Console`, `Environment`, `HttpClient` (when used as singleton).

### 1.2. Factory Method

**Intent**: Define interface for creating object, let subclasses decide which class to instantiate.

```csharp
public abstract class DocumentCreator
{
    public IDocument CreateAndOpen()
    {
        var doc = CreateDocument();   // factory method
        doc.Open();
        return doc;
    }
    
    protected abstract IDocument CreateDocument();
}

public class PdfCreator : DocumentCreator
{
    protected override IDocument CreateDocument() => new PdfDocument();
}

public class WordCreator : DocumentCreator
{
    protected override IDocument CreateDocument() => new WordDocument();
}
```

**Modern alternative**: DI container с `Func<T>` factories.

```csharp
services.AddTransient<Func<DocumentType, IDocument>>(sp => type => type switch
{
    DocumentType.Pdf => sp.GetRequiredService<PdfDocument>(),
    DocumentType.Word => sp.GetRequiredService<WordDocument>(),
    _ => throw new ArgumentException()
});
```

**When apply**:
- Subclass decides concrete type.
- Plugin systems.
- Framework hooks.

**Real .NET**: `WebRequest.Create(url)`, `HttpClientFactory.CreateClient()`.

### 1.3. Abstract Factory

**Intent**: Provide interface для creating families of related objects без specifying concrete classes.

```csharp
public interface IUIFactory
{
    IButton CreateButton();
    ITextBox CreateTextBox();
    IDialog CreateDialog();
}

public class WindowsUIFactory : IUIFactory
{
    public IButton CreateButton() => new WindowsButton();
    public ITextBox CreateTextBox() => new WindowsTextBox();
    public IDialog CreateDialog() => new WindowsDialog();
}

public class MacUIFactory : IUIFactory
{
    public IButton CreateButton() => new MacButton();
    public ITextBox CreateTextBox() => new MacTextBox();
    public IDialog CreateDialog() => new MacDialog();
}

// Use
IUIFactory factory = OperatingSystem.IsWindows() ? new WindowsUIFactory() : new MacUIFactory();
var btn = factory.CreateButton();
var box = factory.CreateTextBox();
```

**When apply**:
- Cross-platform UI (MAUI, Avalonia).
- Database providers (SqlClient vs Postgres).
- Logging implementations.

**Real .NET**: `DbProviderFactory` (ADO.NET), MAUI cross-platform abstractions.

### 1.4. Builder

**Intent**: Separate construction of complex object from its representation.

```csharp
// Fluent builder
public class HttpRequestBuilder
{
    private string _url = "";
    private HttpMethod _method = HttpMethod.Get;
    private readonly Dictionary<string, string> _headers = new();
    private object? _body;
    private TimeSpan _timeout = TimeSpan.FromSeconds(30);
    
    public HttpRequestBuilder Url(string url) { _url = url; return this; }
    public HttpRequestBuilder Method(HttpMethod method) { _method = method; return this; }
    public HttpRequestBuilder Header(string key, string value) { _headers[key] = value; return this; }
    public HttpRequestBuilder Body(object body) { _body = body; return this; }
    public HttpRequestBuilder Timeout(TimeSpan timeout) { _timeout = timeout; return this; }
    
    public HttpRequest Build()
    {
        if (string.IsNullOrEmpty(_url)) throw new InvalidOperationException("URL required");
        return new HttpRequest(_url, _method, _headers, _body, _timeout);
    }
}

// Use
var request = new HttpRequestBuilder()
    .Url("https://api.example.com")
    .Method(HttpMethod.Post)
    .Header("Authorization", "Bearer token")
    .Body(new { name = "Alice" })
    .Timeout(TimeSpan.FromSeconds(10))
    .Build();
```

**Modern alternative**: records `with` expressions для immutable data.

```csharp
public record HttpRequest(
    string Url,
    HttpMethod Method,
    Dictionary<string, string> Headers,
    object? Body,
    TimeSpan Timeout);

var request = new HttpRequest("", HttpMethod.Get, new(), null, TimeSpan.FromSeconds(30))
    with { Url = "https://api.example.com", Method = HttpMethod.Post };
```

**When apply**:
- Many optional parameters.
- Step-by-step construction.
- Immutable objects с complex setup.

**Real .NET**: `StringBuilder`, `WebApplicationBuilder`, `HostBuilder`, EF Core `OnModelCreating(ModelBuilder)`.

### 1.5. Prototype

**Intent**: Specify kinds of objects to create using prototypical instance, create new objects by copying.

```csharp
public abstract class Shape
{
    public string Color { get; set; } = "";
    
    public abstract Shape Clone();
}

public class Circle : Shape
{
    public double Radius { get; set; }
    
    public override Shape Clone() => new Circle { Color = Color, Radius = Radius };
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }
    
    public override Shape Clone() => new Rectangle { Color = Color, Width = Width, Height = Height };
}

// Use
var prototype = new Circle { Color = "Red", Radius = 5 };
var copy1 = prototype.Clone();
var copy2 = prototype.Clone();
```

**Modern alternative**: records с `with` expression — automatic.

```csharp
public record Circle(string Color, double Radius);

var prototype = new Circle("Red", 5);
var copy1 = prototype with { };   // auto Clone
var copy2 = prototype with { Color = "Blue" };   // Clone + modify
```

**When apply**:
- Cloning expensive-to-create objects.
- Avoid factory class proliferation.

**Real .NET**: `ICloneable` (legacy, avoid in new code), records `with`, `MemberwiseClone`.

> [!question]- Интервью: какие creational patterns остались актуальны?
> 1) **Builder** — still common для complex configuration (HostBuilder, WebApplicationBuilder, StringBuilder). Records `with` partially replaces для immutable. 2) **Abstract Factory** — cross-platform UI (MAUI/Avalonia), database providers. 3) **Factory Method** — mostly заменён DI container resolution + `Func<T>` factories. 4) **Singleton** — manual implementation anti-pattern; `services.AddSingleton<T>()` правильный путь. 5) **Prototype** — заменён records `with`. **Direction 2024+**: DI container handles most "creational" concerns. Builder остаётся essential для complex objects (especially fluent APIs).

---

## 2. Structural Patterns

### 2.1. Adapter

**Intent**: Convert interface of class into another interface clients expect.

```csharp
// Third-party legacy API
public class LegacyShipping
{
    public void ShipItem(string itemId, double weightInPounds, string addressLine);
}

// Our system interface
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

**When apply**:
- Wrap legacy / third-party APIs.
- Make incompatible interfaces work together.
- Migration boundaries.

**Real .NET**: `TextReader`/`TextWriter` adapters над `Stream`, `StreamReader`/`StreamWriter`.

### 2.2. Bridge

**Intent**: Decouple abstraction from implementation so both can vary independently.

```csharp
// Implementation hierarchy
public interface IRenderer
{
    void Render(string shape);
}

public class VectorRenderer : IRenderer
{
    public void Render(string shape) => Console.WriteLine($"Drawing {shape} as vector");
}

public class RasterRenderer : IRenderer
{
    public void Render(string shape) => Console.WriteLine($"Drawing {shape} as pixels");
}

// Abstraction hierarchy
public abstract class Shape
{
    protected readonly IRenderer Renderer;
    protected Shape(IRenderer renderer) => Renderer = renderer;
    public abstract void Draw();
}

public class Circle : Shape
{
    public Circle(IRenderer renderer) : base(renderer) { }
    public override void Draw() => Renderer.Render("circle");
}

public class Square : Shape
{
    public Square(IRenderer renderer) : base(renderer) { }
    public override void Draw() => Renderer.Render("square");
}

// Use — Shape × Renderer combinations без class explosion
new Circle(new VectorRenderer()).Draw();
new Circle(new RasterRenderer()).Draw();
new Square(new VectorRenderer()).Draw();
```

**When apply**:
- Two orthogonal hierarchies.
- Avoid combinatorial explosion (NxM classes).
- Cross-platform abstractions.

**Real .NET**: ADO.NET (DbCommand × DbConnection providers), Logging (ILogger × ILoggerProvider).

### 2.3. Composite

**Intent**: Compose objects into tree structures, treat individual + composition uniformly.

```csharp
public abstract class FileSystemNode
{
    public string Name { get; }
    protected FileSystemNode(string name) => Name = name;
    
    public abstract long GetSize();
    public abstract void Print(int indent = 0);
}

// Leaf
public class FileNode : FileSystemNode
{
    private readonly long _size;
    
    public FileNode(string name, long size) : base(name) => _size = size;
    
    public override long GetSize() => _size;
    public override void Print(int indent = 0) =>
        Console.WriteLine($"{new string(' ', indent)}{Name} ({_size} bytes)");
}

// Composite
public class DirectoryNode : FileSystemNode
{
    private readonly List<FileSystemNode> _children = new();
    
    public DirectoryNode(string name) : base(name) { }
    
    public void Add(FileSystemNode child) => _children.Add(child);
    public void Remove(FileSystemNode child) => _children.Remove(child);
    
    public override long GetSize() => _children.Sum(c => c.GetSize());
    
    public override void Print(int indent = 0)
    {
        Console.WriteLine($"{new string(' ', indent)}{Name}/");
        foreach (var child in _children) child.Print(indent + 2);
    }
}

// Use
var root = new DirectoryNode("root");
var docs = new DirectoryNode("docs");
docs.Add(new FileNode("readme.md", 100));
docs.Add(new FileNode("guide.pdf", 5000));
root.Add(docs);
root.Add(new FileNode("config.json", 200));

Console.WriteLine($"Total size: {root.GetSize()}");   // 5300
root.Print();
```

**When apply**:
- Tree structures (file system, UI hierarchies, organization charts).
- Recursive operations.

**Real .NET**: WPF visual tree, XmlNode hierarchy, Roslyn SyntaxNode tree, expression trees.

### 2.4. Decorator

**Intent**: Attach additional responsibilities to object dynamically. Alternative to subclassing.

```csharp
public interface IDataService
{
    Task<string> GetAsync(int id);
}

// Concrete
public class DatabaseDataService : IDataService
{
    public async Task<string> GetAsync(int id)
    {
        await Task.Delay(100);   // simulate DB
        return $"Data for {id}";
    }
}

// Caching decorator
public class CachingDecorator : IDataService
{
    private readonly IDataService _inner;
    private readonly IMemoryCache _cache;
    
    public CachingDecorator(IDataService inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }
    
    public async Task<string> GetAsync(int id)
    {
        if (_cache.TryGetValue(id, out string? cached)) return cached!;
        var value = await _inner.GetAsync(id);
        _cache.Set(id, value, TimeSpan.FromMinutes(5));
        return value;
    }
}

// Logging decorator
public class LoggingDecorator : IDataService
{
    private readonly IDataService _inner;
    private readonly ILogger<LoggingDecorator> _logger;
    
    public LoggingDecorator(IDataService inner, ILogger<LoggingDecorator> logger)
    {
        _inner = inner;
        _logger = logger;
    }
    
    public async Task<string> GetAsync(int id)
    {
        _logger.LogInformation("Getting {Id}", id);
        var sw = Stopwatch.StartNew();
        try
        {
            return await _inner.GetAsync(id);
        }
        finally
        {
            _logger.LogInformation("Got {Id} in {Ms}ms", id, sw.ElapsedMilliseconds);
        }
    }
}

// Retry decorator
public class RetryDecorator : IDataService
{
    private readonly IDataService _inner;
    private readonly int _maxAttempts;
    
    public RetryDecorator(IDataService inner, int maxAttempts = 3)
    {
        _inner = inner;
        _maxAttempts = maxAttempts;
    }
    
    public async Task<string> GetAsync(int id)
    {
        for (int attempt = 1; attempt <= _maxAttempts; attempt++)
        {
            try { return await _inner.GetAsync(id); }
            catch (TransientException) when (attempt < _maxAttempts) { await Task.Delay(100 * attempt); }
        }
        throw new InvalidOperationException();
    }
}

// Compose — Scrutor library
services.AddScoped<IDataService, DatabaseDataService>();
services.Decorate<IDataService, CachingDecorator>();
services.Decorate<IDataService, RetryDecorator>();
services.Decorate<IDataService, LoggingDecorator>();
// Order: Logging → Retry → Caching → Database
```

**Modern .NET**: ASP.NET Core middleware pipeline — Decorator pattern на HTTP requests.

**When apply**:
- Cross-cutting concerns (logging, caching, retry, validation).
- Adding behavior без modifying core.
- Compose multiple behaviors.

**Real .NET**: Stream decorators (`BufferedStream`, `GZipStream`, `CryptoStream`), HttpClient `DelegatingHandler`, ASP.NET Core middleware.

### 2.5. Facade

**Intent**: Provide unified interface to set of interfaces в subsystem.

```csharp
// Complex subsystem
public class InventoryService { /* check stock */ }
public class PricingService { /* calculate prices, taxes */ }
public class PaymentService { /* charge card */ }
public class ShippingService { /* schedule delivery */ }
public class NotificationService { /* email/SMS */ }

// Facade — single entry point
public class OrderProcessingFacade
{
    private readonly InventoryService _inventory;
    private readonly PricingService _pricing;
    private readonly PaymentService _payment;
    private readonly ShippingService _shipping;
    private readonly NotificationService _notification;
    
    public OrderProcessingFacade(/* DI */) { /* ... */ }
    
    public async Task<OrderResult> PlaceOrderAsync(OrderRequest request)
    {
        if (!await _inventory.ReserveAsync(request.Items))
            return OrderResult.OutOfStock;
        
        var pricing = _pricing.Calculate(request);
        
        if (!await _payment.ChargeAsync(request.Customer, pricing.Total))
            return OrderResult.PaymentFailed;
        
        var trackingId = await _shipping.ScheduleAsync(request, pricing);
        await _notification.SendOrderConfirmationAsync(request.Customer, trackingId);
        
        return OrderResult.Success(trackingId);
    }
}

// Client uses facade — doesn't see 5 services
await orderFacade.PlaceOrderAsync(request);
```

**When apply**:
- Hide complex subsystem.
- Reduce coupling between client и subsystem.
- Provide simple API.

**Real .NET**: `WebApplication`, `HttpClient` (facade over HttpMessageHandler chain), `EntityFrameworkCore.DbContext`.

### 2.6. Flyweight

**Intent**: Use sharing для efficiently support large numbers of fine-grained objects.

```csharp
// Intrinsic state — shared
public class TreeType
{
    public string Name { get; }
    public string Texture { get; }
    public string Color { get; }
    
    public TreeType(string name, string texture, string color)
    {
        Name = name;
        Texture = texture;
        Color = color;
    }
    
    public void Draw(int x, int y) => Console.WriteLine($"Drawing {Name} at ({x},{y})");
}

// Flyweight factory
public static class TreeTypeFactory
{
    private static readonly Dictionary<string, TreeType> _cache = new();
    
    public static TreeType Get(string name, string texture, string color)
    {
        var key = $"{name}-{texture}-{color}";
        if (!_cache.TryGetValue(key, out var type))
        {
            type = new TreeType(name, texture, color);
            _cache[key] = type;
        }
        return type;
    }
}

// Extrinsic state — per instance
public class Tree
{
    public int X { get; }
    public int Y { get; }
    public TreeType Type { get; }   // shared!
    
    public Tree(int x, int y, TreeType type) => (X, Y, Type) = (x, y, type);
    
    public void Draw() => Type.Draw(X, Y);
}

// Use — millions of trees, only few unique TreeTypes
var oak = TreeTypeFactory.Get("Oak", "oak.png", "Green");
var pine = TreeTypeFactory.Get("Pine", "pine.png", "DarkGreen");

var forest = new List<Tree>();
for (int i = 0; i < 1_000_000; i++)
    forest.Add(new Tree(rand.Next(), rand.Next(), i % 2 == 0 ? oak : pine));

// 1M trees, but only 2 TreeType instances в memory
```

**Modern .NET**: String interning is built-in flyweight (`string.Intern`).

**When apply**:
- Many similar objects (millions of game entities).
- High memory pressure.
- Most state is shared / cacheable.

**Real .NET**: String interning (`string.Intern`), `Char` boxing cache, `Boolean.True`/`False`, integer caching `(byte)0..(byte)255`.

### 2.7. Proxy

**Intent**: Provide surrogate or placeholder for another object to control access.

```csharp
public interface IExpensiveResource
{
    string GetData();
}

// Real subject — expensive
public class RealResource : IExpensiveResource
{
    public RealResource()
    {
        Console.WriteLine("Loading expensive resource...");
        Thread.Sleep(2000);
    }
    
    public string GetData() => "Real data";
}

// Lazy proxy
public class LazyProxy : IExpensiveResource
{
    private RealResource? _real;
    
    public string GetData()
    {
        _real ??= new RealResource();   // create on first access
        return _real.GetData();
    }
}

// Protection proxy — access control
public class ProtectionProxy : IExpensiveResource
{
    private readonly IExpensiveResource _real;
    private readonly string _userRole;
    
    public ProtectionProxy(IExpensiveResource real, string userRole)
    {
        _real = real;
        _userRole = userRole;
    }
    
    public string GetData()
    {
        if (_userRole != "Admin") throw new UnauthorizedAccessException();
        return _real.GetData();
    }
}

// Remote proxy — network
public class HttpProxy : IExpensiveResource
{
    private readonly HttpClient _client;
    
    public HttpProxy(HttpClient client) => _client = client;
    
    public string GetData() =>
        _client.GetStringAsync("https://api.example.com/data").Result;
}
```

**Variants**:
- **Lazy** — defer creation.
- **Protection** — access control.
- **Remote** — network communication.
- **Virtual** — placeholder for heavy resources.
- **Smart** — additional behavior (reference counting, locking).

**Modern .NET**: `Lazy<T>` built-in для lazy proxy, EF Core lazy loading proxies, Castle.DynamicProxy for AOP.

**When apply**:
- Lazy initialization.
- Access control.
- Logging / caching wrapping.
- Remote services.

**Real .NET**: `Lazy<T>`, EF Core lazy loading, ServiceFabric remoting, gRPC client proxies.

> [!question]- Интервью: чем Proxy отличается от Decorator?
> Both wrap object same interface. **Decorator**: adds behavior — caller knows wrapped object exists, decorators stack (CachingDecorator + LoggingDecorator + RetryDecorator). Composition focused. **Proxy**: controls access — caller treats proxy as the real thing (transparent). Single proxy typically. **Intent**: Decorator extends, Proxy controls/replaces. **Examples Proxy**: `Lazy<T>` (defer creation), EF Core lazy loading (load on access), gRPC client (network call hidden), security check (refuse if not authorized). **Examples Decorator**: cache repository results, log every call, retry on failure, validate input. Code structure similar — intent differs. Both useful.

---

## 3. Behavioral Patterns

### 3.1. Chain of Responsibility

**Intent**: Pass request along chain of handlers; each decides to process или pass further.

```csharp
public abstract class ApprovalHandler
{
    private ApprovalHandler? _next;
    
    public ApprovalHandler SetNext(ApprovalHandler next)
    {
        _next = next;
        return next;
    }
    
    public abstract void Approve(decimal amount);
    
    protected void PassToNext(decimal amount) => _next?.Approve(amount);
}

public class TeamLead : ApprovalHandler
{
    public override void Approve(decimal amount)
    {
        if (amount <= 1000) Console.WriteLine($"TeamLead approved {amount}");
        else PassToNext(amount);
    }
}

public class Manager : ApprovalHandler
{
    public override void Approve(decimal amount)
    {
        if (amount <= 10_000) Console.WriteLine($"Manager approved {amount}");
        else PassToNext(amount);
    }
}

public class Director : ApprovalHandler
{
    public override void Approve(decimal amount)
    {
        if (amount <= 100_000) Console.WriteLine($"Director approved {amount}");
        else PassToNext(amount);
    }
}

// Use
var lead = new TeamLead();
var mgr = new Manager();
var dir = new Director();
lead.SetNext(mgr).SetNext(dir);

lead.Approve(500);    // TeamLead approved
lead.Approve(5000);   // Manager approved
lead.Approve(50000);  // Director approved
```

**Modern .NET**: ASP.NET Core middleware = perfect Chain of Responsibility.

```csharp
app.UseExceptionHandler();   // first
app.UseAuthentication();
app.UseAuthorization();
app.UseRouting();
app.UseEndpoints(...);       // last

public class CustomMiddleware
{
    private readonly RequestDelegate _next;
    public CustomMiddleware(RequestDelegate next) => _next = next;
    
    public async Task InvokeAsync(HttpContext context)
    {
        // pre-processing
        await _next(context);   // pass to next
        // post-processing
    }
}
```

**When apply**:
- Multiple handlers can process request.
- Order matters.
- Pipeline / middleware.

**Real .NET**: ASP.NET Core middleware, HttpClient `DelegatingHandler`, Roslyn analyzers.

### 3.2. Command

**Intent**: Encapsulate request as object, parameterize clients with different requests, queue / log / undo.

```csharp
public interface ICommand
{
    void Execute();
    void Undo();
}

public class TextEditor
{
    public string Text { get; set; } = "";
}

public class InsertTextCommand : ICommand
{
    private readonly TextEditor _editor;
    private readonly string _text;
    private readonly int _position;
    
    public InsertTextCommand(TextEditor editor, string text, int position)
    {
        _editor = editor;
        _text = text;
        _position = position;
    }
    
    public void Execute() =>
        _editor.Text = _editor.Text.Insert(_position, _text);
    
    public void Undo() =>
        _editor.Text = _editor.Text.Remove(_position, _text.Length);
}

public class CommandManager
{
    private readonly Stack<ICommand> _history = new();
    
    public void Execute(ICommand command)
    {
        command.Execute();
        _history.Push(command);
    }
    
    public void Undo()
    {
        if (_history.TryPop(out var cmd)) cmd.Undo();
    }
}

// Use
var editor = new TextEditor();
var manager = new CommandManager();

manager.Execute(new InsertTextCommand(editor, "Hello", 0));
manager.Execute(new InsertTextCommand(editor, " World", 5));
Console.WriteLine(editor.Text);   // "Hello World"
manager.Undo();
Console.WriteLine(editor.Text);   // "Hello"
```

**Modern .NET**: WPF `ICommand`, MediatR `IRequest<TResponse>`.

```csharp
// MediatR
public record CreateUserCommand(string Email, string Name) : IRequest<int>;

public class CreateUserHandler : IRequestHandler<CreateUserCommand, int>
{
    public async Task<int> Handle(CreateUserCommand cmd, CancellationToken ct)
    {
        // create user
        return userId;
    }
}

var userId = await mediator.Send(new CreateUserCommand("a@x.com", "Alice"));
```

**When apply**:
- Undo/redo functionality.
- Queueable operations (job queue).
- Decoupling sender от receiver.
- CQRS architecture.

**Real .NET**: WPF `RoutedCommand`, MediatR, `MessagePublisher` patterns, Hangfire jobs.

### 3.3. Iterator

**Intent**: Provide way to access elements of aggregate sequentially без exposing representation.

```csharp
// Manual implementation (instructive)
public class Tree<T>
{
    private readonly Node<T>? _root;
    public Tree(Node<T>? root) => _root = root;
    
    public IEnumerable<T> InOrderTraversal()
    {
        if (_root == null) yield break;
        foreach (var item in TraverseInOrder(_root)) yield return item;
    }
    
    private IEnumerable<T> TraverseInOrder(Node<T> node)
    {
        if (node.Left != null)
            foreach (var item in TraverseInOrder(node.Left)) yield return item;
        yield return node.Value;
        if (node.Right != null)
            foreach (var item in TraverseInOrder(node.Right)) yield return item;
    }
}

public class Node<T>
{
    public T Value { get; set; }
    public Node<T>? Left { get; set; }
    public Node<T>? Right { get; set; }
}
```

**Modern .NET**: `IEnumerable<T>` + `yield return` — built-in Iterator. См. [[iterators-yield]].

```csharp
public IEnumerable<int> Fibonacci()
{
    int a = 0, b = 1;
    while (true)
    {
        yield return a;
        (a, b) = (b, a + b);
    }
}

foreach (var n in Fibonacci().Take(10)) Console.WriteLine(n);
```

**When apply**:
- Custom traversal logic.
- Lazy enumeration.
- Hide internal structure.

**Real .NET**: `IEnumerable<T>`, LINQ, custom collections с `GetEnumerator`.

### 3.4. Mediator

**Intent**: Define object encapsulating how set of objects interact, promoting loose coupling.

```csharp
public interface IChatMediator
{
    void Send(string message, User sender);
    void Register(User user);
}

public class ChatRoom : IChatMediator
{
    private readonly List<User> _users = new();
    
    public void Register(User user)
    {
        _users.Add(user);
        user.Mediator = this;
    }
    
    public void Send(string message, User sender)
    {
        foreach (var user in _users)
            if (user != sender) user.Receive(message, sender);
    }
}

public class User
{
    public string Name { get; }
    public IChatMediator? Mediator { get; set; }
    
    public User(string name) => Name = name;
    
    public void Send(string message) => Mediator?.Send(message, this);
    public void Receive(string message, User from) =>
        Console.WriteLine($"{Name} received from {from.Name}: {message}");
}

// Use
var room = new ChatRoom();
var alice = new User("Alice");
var bob = new User("Bob");
var charlie = new User("Charlie");

room.Register(alice);
room.Register(bob);
room.Register(charlie);

alice.Send("Hello!");   // Bob и Charlie receive
```

**Modern .NET**: MediatR library — Command/Query/Notification dispatching через Mediator.

```csharp
// MediatR — request/response
var result = await mediator.Send(new GetUserQuery(42));

// MediatR — notification (1 publisher, N handlers)
await mediator.Publish(new OrderPlaced(orderId));
```

**When apply**:
- Many objects interact in complex ways.
- Want to decouple.
- Domain events.

**Real .NET**: MediatR, ASP.NET Core SignalR (hub mediates clients), event aggregator patterns.

### 3.5. Memento

**Intent**: Capture and externalize object's internal state без violating encapsulation, restore later.

```csharp
public class Editor
{
    public string Text { get; private set; } = "";
    public int CursorPosition { get; private set; }
    
    public void Type(string text)
    {
        Text = Text.Insert(CursorPosition, text);
        CursorPosition += text.Length;
    }
    
    public void MoveCursor(int position) => CursorPosition = position;
    
    // Save state
    public Memento Save() => new(Text, CursorPosition);
    
    // Restore
    public void Restore(Memento memento)
    {
        Text = memento.Text;
        CursorPosition = memento.CursorPosition;
    }
    
    public sealed class Memento
    {
        public string Text { get; }
        public int CursorPosition { get; }
        
        internal Memento(string text, int cursor)
        {
            Text = text;
            CursorPosition = cursor;
        }
    }
}

// Use
var editor = new Editor();
editor.Type("Hello");
var save1 = editor.Save();
editor.Type(" World");
editor.Restore(save1);   // back to "Hello"
```

**Modern .NET**: records + `with` для simple cases.

```csharp
public record EditorState(string Text, int CursorPosition);

public class Editor
{
    public EditorState State { get; private set; } = new("", 0);
    
    public void Type(string text) =>
        State = State with
        {
            Text = State.Text.Insert(State.CursorPosition, text),
            CursorPosition = State.CursorPosition + text.Length
        };
    
    public EditorState Save() => State;
    public void Restore(EditorState state) => State = state;
}
```

**When apply**:
- Undo/redo functionality.
- Save points (game saves).
- Snapshots для transactions.

**Real .NET**: WPF DependencyObject, EF Core change tracking, IMemoryCache snapshots.

### 3.6. Observer

**Intent**: Define one-to-many dependency, when one changes — all dependents notified.

```csharp
// Manual implementation (instructive)
public interface IObserver<T>
{
    void Update(T data);
}

public class Subject<T>
{
    private readonly List<IObserver<T>> _observers = new();
    
    public void Subscribe(IObserver<T> observer) => _observers.Add(observer);
    public void Unsubscribe(IObserver<T> observer) => _observers.Remove(observer);
    
    public void Notify(T data)
    {
        foreach (var observer in _observers) observer.Update(data);
    }
}
```

**Modern .NET**: `event` keyword — Observer Pattern built-in.

```csharp
public class StockTicker
{
    public event EventHandler<PriceChangedEventArgs>? PriceChanged;
    
    public void UpdatePrice(string symbol, decimal price) =>
        PriceChanged?.Invoke(this, new PriceChangedEventArgs(symbol, price));
}

public class PriceChangedEventArgs : EventArgs
{
    public string Symbol { get; }
    public decimal Price { get; }
    public PriceChangedEventArgs(string symbol, decimal price) =>
        (Symbol, Price) = (symbol, price);
}

// Subscribe
var ticker = new StockTicker();
ticker.PriceChanged += (s, e) => Console.WriteLine($"{e.Symbol}: {e.Price}");
ticker.UpdatePrice("AAPL", 150);
```

**Reactive Extensions** — full-featured observer:

```csharp
using System.Reactive.Subjects;
using System.Reactive.Linq;

var subject = new Subject<PriceChange>();

subject
    .Where(p => p.Symbol == "AAPL")
    .Throttle(TimeSpan.FromSeconds(1))
    .Subscribe(p => Console.WriteLine($"Throttled: {p.Symbol} {p.Price}"));

subject.OnNext(new PriceChange("AAPL", 150));
```

**When apply**:
- One-to-many notifications.
- Loose coupling.
- Event-driven systems.
- Reactive programming.

**Real .NET**: `event`, `INotifyPropertyChanged`, Reactive Extensions, EventArgs in WPF/WinUI/MAUI.

### 3.7. State

**Intent**: Allow object to alter behavior when internal state changes. Object appears to change class.

```csharp
public abstract class OrderState
{
    public abstract void Pay(Order order);
    public abstract void Ship(Order order);
    public abstract void Cancel(Order order);
}

public class PendingState : OrderState
{
    public override void Pay(Order order)
    {
        Console.WriteLine("Paying...");
        order.SetState(new PaidState());
    }
    
    public override void Ship(Order order) =>
        throw new InvalidOperationException("Cannot ship unpaid order");
    
    public override void Cancel(Order order)
    {
        Console.WriteLine("Cancelling pending order");
        order.SetState(new CancelledState());
    }
}

public class PaidState : OrderState
{
    public override void Pay(Order order) =>
        throw new InvalidOperationException("Already paid");
    
    public override void Ship(Order order)
    {
        Console.WriteLine("Shipping...");
        order.SetState(new ShippedState());
    }
    
    public override void Cancel(Order order)
    {
        Console.WriteLine("Refunding paid order");
        order.SetState(new CancelledState());
    }
}

public class ShippedState : OrderState { /* ... */ }
public class CancelledState : OrderState { /* ... */ }

public class Order
{
    private OrderState _state = new PendingState();
    public void SetState(OrderState state) => _state = state;
    public void Pay() => _state.Pay(this);
    public void Ship() => _state.Ship(this);
    public void Cancel() => _state.Cancel(this);
}
```

**Modern C#**: pattern matching switch — cleaner для simple state machines.

```csharp
public enum OrderStatus { Pending, Paid, Shipped, Cancelled }

public record Order(int Id, OrderStatus Status);

public static Order Pay(Order order) => order.Status switch
{
    OrderStatus.Pending => order with { Status = OrderStatus.Paid },
    _ => throw new InvalidOperationException($"Cannot pay from {order.Status}")
};

public static Order Ship(Order order) => order.Status switch
{
    OrderStatus.Paid => order with { Status = OrderStatus.Shipped },
    _ => throw new InvalidOperationException($"Cannot ship from {order.Status}")
};
```

**When apply**:
- Object behavior depends on state.
- Many state-dependent conditionals.
- Workflow engines.

**Real .NET**: WPF VisualStateManager, workflow engines (Stateless library), TCP connection states.

### 3.8. Strategy

**Intent**: Define family of algorithms, encapsulate each, make interchangeable.

```csharp
public interface ICompressionStrategy
{
    byte[] Compress(byte[] data);
    byte[] Decompress(byte[] data);
}

public class GZipStrategy : ICompressionStrategy
{
    public byte[] Compress(byte[] data) { /* GZip */ return data; }
    public byte[] Decompress(byte[] data) { /* GZip */ return data; }
}

public class ZipStrategy : ICompressionStrategy
{
    public byte[] Compress(byte[] data) { /* Zip */ return data; }
    public byte[] Decompress(byte[] data) { /* Zip */ return data; }
}

public class FileArchiver
{
    private readonly ICompressionStrategy _strategy;
    
    public FileArchiver(ICompressionStrategy strategy) => _strategy = strategy;
    
    public byte[] Archive(byte[] data) => _strategy.Compress(data);
}

// Use
var gzip = new FileArchiver(new GZipStrategy());
var zip = new FileArchiver(new ZipStrategy());
```

**Modern C#**: `Func<T>` / `Action<T>` — single-method strategy.

```csharp
// Strategy с одной method — Func<T>
public class FileArchiver
{
    private readonly Func<byte[], byte[]> _compress;
    
    public FileArchiver(Func<byte[], byte[]> compress) => _compress = compress;
    
    public byte[] Archive(byte[] data) => _compress(data);
}

// Use
var gzip = new FileArchiver(data => /* GZip */ data);
var zip = new FileArchiver(data => /* Zip */ data);
```

**When apply**:
- Multiple algorithms для same task.
- Runtime algorithm selection.
- Replace conditionals с polymorphism.

**Real .NET**: `IComparer<T>` (sorting strategies), `IEqualityComparer<T>`, `JsonSerializerOptions` settings.

### 3.9. Template Method

**Intent**: Define skeleton of algorithm in operation, defer some steps to subclasses.

```csharp
public abstract class ReportGenerator
{
    // Template method — algorithm skeleton
    public string Generate()
    {
        var data = LoadData();
        var processed = ProcessData(data);
        var formatted = FormatOutput(processed);
        return formatted;
    }
    
    // Required overrides
    protected abstract IEnumerable<DataRow> LoadData();
    protected abstract string FormatOutput(List<DataRow> data);
    
    // Optional override (hook)
    protected virtual List<DataRow> ProcessData(IEnumerable<DataRow> data) => data.ToList();
}

public class HtmlReport : ReportGenerator
{
    protected override IEnumerable<DataRow> LoadData() => /* DB query */ new List<DataRow>();
    protected override string FormatOutput(List<DataRow> data) => "<html>...</html>";
}

public class CsvReport : ReportGenerator
{
    protected override IEnumerable<DataRow> LoadData() => /* DB query */ new List<DataRow>();
    protected override string FormatOutput(List<DataRow> data) =>
        string.Join("\n", data.Select(r => r.ToString()));
    
    protected override List<DataRow> ProcessData(IEnumerable<DataRow> data) =>
        data.Where(r => r.IsValid).ToList();   // CSV filters invalid rows
}
```

**When apply**:
- Algorithm с invariant + variant steps.
- Code reuse через inheritance.
- Frameworks.

**Real .NET**: ASP.NET Core `Controller`, EF Core `DbContext.OnConfiguring`, BackgroundService base class, Test fixtures.

### 3.10. Visitor

**Intent**: Represent operation to be performed on elements of object structure. Define new operation без changing classes.

```csharp
// Element hierarchy
public interface IShapeVisitor
{
    void Visit(Circle circle);
    void Visit(Square square);
    void Visit(Rectangle rectangle);
}

public abstract class Shape
{
    public abstract void Accept(IShapeVisitor visitor);
}

public class Circle : Shape
{
    public double Radius { get; set; }
    public override void Accept(IShapeVisitor visitor) => visitor.Visit(this);
}

public class Square : Shape
{
    public double Side { get; set; }
    public override void Accept(IShapeVisitor visitor) => visitor.Visit(this);
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }
    public override void Accept(IShapeVisitor visitor) => visitor.Visit(this);
}

// Visitor implementations
public class AreaVisitor : IShapeVisitor
{
    public double Total { get; private set; }
    public void Visit(Circle c) => Total += Math.PI * c.Radius * c.Radius;
    public void Visit(Square s) => Total += s.Side * s.Side;
    public void Visit(Rectangle r) => Total += r.Width * r.Height;
}

public class PerimeterVisitor : IShapeVisitor
{
    public double Total { get; private set; }
    public void Visit(Circle c) => Total += 2 * Math.PI * c.Radius;
    public void Visit(Square s) => Total += 4 * s.Side;
    public void Visit(Rectangle r) => Total += 2 * (r.Width + r.Height);
}

// Use
var shapes = new List<Shape> { new Circle { Radius = 5 }, new Square { Side = 3 } };
var areaVisitor = new AreaVisitor();
foreach (var shape in shapes) shape.Accept(areaVisitor);
Console.WriteLine($"Total area: {areaVisitor.Total}");
```

**Modern C#**: pattern matching switch — much cleaner.

```csharp
public abstract record Shape;
public record Circle(double Radius) : Shape;
public record Square(double Side) : Shape;
public record Rectangle(double Width, double Height) : Shape;

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

**When apply**:
- Operations on heterogeneous object hierarchies.
- Add operations без modifying classes.
- AST processing.

**Real .NET**: Roslyn `ExpressionVisitor` (LINQ-to-SQL translation), `SyntaxVisitor` (compilers), pattern matching switch (modern equivalent).

### 3.11. Interpreter

**Intent**: Define grammar representation, interpret sentences in language.

```csharp
public interface IExpression
{
    int Interpret(Dictionary<string, int> context);
}

public class Number : IExpression
{
    private readonly int _value;
    public Number(int value) => _value = value;
    public int Interpret(Dictionary<string, int> context) => _value;
}

public class Variable : IExpression
{
    private readonly string _name;
    public Variable(string name) => _name = name;
    public int Interpret(Dictionary<string, int> context) => context[_name];
}

public class Add : IExpression
{
    private readonly IExpression _left, _right;
    public Add(IExpression left, IExpression right) => (_left, _right) = (left, right);
    public int Interpret(Dictionary<string, int> context) =>
        _left.Interpret(context) + _right.Interpret(context);
}

// Use — represent (x + 5)
var expr = new Add(new Variable("x"), new Number(5));
var ctx = new Dictionary<string, int> { ["x"] = 10 };
Console.WriteLine(expr.Interpret(ctx));   // 15
```

**Modern .NET**: Expression Trees built-in (LINQ to SQL, EF Core).

```csharp
Expression<Func<int, int>> expr = x => x + 5;
// Same idea, more powerful — analyzable + compilable
```

**When apply**:
- DSL implementation.
- Rule engines.
- Configuration languages.

**Real .NET**: LINQ Expression Trees, regex engine, Roslyn syntax trees.

> [!question]- Интервью: какие behavioral patterns остались актуальны?
> 1) **Chain of Responsibility** — ASP.NET Core middleware (perfect example). 2) **Mediator** — MediatR библиотека (CQRS, domain events). 3) **Command** — MediatR `IRequest<T>`, WPF `ICommand`. 4) **Template Method** — abstract base + virtual methods (BackgroundService, Controller). 5) **Strategy** — `Func<T>` для simple cases, interface для complex. 6) **Observer** — `event` keyword + `INotifyPropertyChanged`, Reactive Extensions. **Replaced by language**: Iterator (`yield return`), State (pattern matching switch), Visitor (pattern matching switch), Memento (records + `with`). **Real .NET examples** every behavioral pattern имеет.

---

## 4. Pattern relationships + decision tree

### 4.1. Pattern relationships

```
Creational:
- Abstract Factory often returns Builder products
- Singleton + Factory Method common combination
- Builder для complex construction (alternative Factory с many params)

Structural:
- Adapter — make incompatible work
- Decorator — add behavior dynamically
- Proxy — control access
- Facade — simplify subsystem
- Composite — tree structures
- Flyweight — share instances
- Bridge — separate hierarchies

Behavioral:
- Command + Memento — undo/redo
- Observer + Mediator — decouple components
- Strategy + Template Method — algorithmic variation
- Chain of Responsibility + Composite — pipeline на tree
- Iterator + Composite — traversal trees
- Visitor + Composite — operations on trees
```

### 4.2. Quick decision

```
Need to create object?
├── Single instance → Singleton (DI AddSingleton<T>)
├── Subclass decides type → Factory Method (or DI Func<T>)
├── Family of related → Abstract Factory
├── Complex configuration → Builder (StringBuilder, HostBuilder)
└── Copy existing → Prototype (record with)

Need to compose objects?
├── Tree structure → Composite
├── Wrap legacy → Adapter
├── Add behavior dynamically → Decorator
├── Control access → Proxy
├── Simplify subsystem → Facade
├── Share many instances → Flyweight
└── Two orthogonal hierarchies → Bridge

Need to communicate?
├── One-to-many notifications → Observer (event)
├── Decouple components → Mediator (MediatR)
├── Encapsulate request → Command (MediatR IRequest)
├── Algorithm variation → Strategy (Func<T>)
├── Algorithm template → Template Method (abstract + virtual)
├── State machine → State (pattern matching)
├── Pipeline → Chain of Responsibility (middleware)
├── Operations on hierarchy → Visitor (pattern matching)
├── Custom traversal → Iterator (yield)
├── Snapshot/undo → Memento (records)
└── DSL/grammar → Interpreter (Expression Trees)
```

### 4.3. Pattern frequency в production .NET

```
Used часто:
✅ Decorator (Stream, middleware, Scrutor)
✅ Observer (event, INotifyPropertyChanged, RX)
✅ Composite (UI hierarchies, file systems, AST)
✅ Template Method (BackgroundService, Controller)
✅ Builder (HostBuilder, StringBuilder)
✅ Strategy (IComparer, Func<T>)
✅ Mediator (MediatR, SignalR)
✅ Chain of Responsibility (middleware)
✅ Adapter (legacy wrapping)
✅ Proxy (Lazy<T>, EF Core lazy)

Used редко manually (language replaces):
⚠️ Iterator (yield)
⚠️ State (pattern matching)
⚠️ Visitor (pattern matching)
⚠️ Memento (records + with)
⚠️ Singleton (DI container)
⚠️ Factory Method (DI container)

Edge cases:
- Flyweight (string interning automatic)
- Bridge (cross-platform abstractions)
- Interpreter (Expression Trees, parsers)
- Prototype (records with)
- Abstract Factory (UI cross-platform, DB providers)
```

> [!question]- Интервью: на собеседовании говорят "напиши Singleton" — что ответить?
> Спросить **зачем** — это часто trick question. Best answer: 1) **Show classic implementation** (`Lazy<T>` thread-safe). 2) **Recommend modern alternative** — `services.AddSingleton<T>()` через DI container. 3) **Объясни why anti-pattern**: global mutable state, hidden dependencies, untestable. **Stay flexible**: if interviewer wants pure pattern — provide. If wants production knowledge — explain DI. **Show language understanding**: many GoF patterns заменены C# features (yield, event, pattern matching, records). Pattern thinking важно, но writing them manually часто означает не использовать language features.

---

## 5. Anti-patterns to recognize

### 5.1. God Object

```csharp
// ❌ Single class doing everything
public class OrderManager
{
    public void ProcessOrder(int id) { /* ... */ }
    public void GenerateInvoice(int id) { /* ... */ }
    public void SendEmail(int id) { /* ... */ }
    public void UpdateInventory(int id) { /* ... */ }
    public void ChargeCard(int id) { /* ... */ }
    // ... 50 more methods
}
```

**Fix**: split по responsibility (SRP).

### 5.2. Singleton Abuse

```csharp
public static class GlobalState
{
    public static Dictionary<string, object> Cache = new();   // ❌ mutable global
    public static User CurrentUser { get; set; }              // ❌ global state
    public static DbConnection DbConnection { get; set; }     // ❌ shared mutable
}
```

**Fix**: DI container with appropriate lifetimes.

### 5.3. Service Locator

```csharp
public class Service
{
    public void Process()
    {
        var repo = ServiceLocator.GetService<IRepository>();   // ❌ hidden dep
    }
}
```

**Fix**: constructor injection.

### 5.4. Anemic Domain Model

```csharp
public class Order   // ❌ only data
{
    public int Id { get; set; }
    public List<OrderLine> Lines { get; set; } = new();
    public OrderStatus Status { get; set; }
    public decimal Total { get; set; }
}

public class OrderService   // all logic here
{
    public void Pay(Order o) { o.Status = OrderStatus.Paid; }
}
```

**Fix**: rich entity (Order.Pay() enforces invariants).

### 5.5. Circular Dependency

```csharp
public class A { B B; public A(B b) => B = b; }
public class B { A A; public B(A a) => A = a; }
// ❌ A depends on B, B depends on A — DI container fails
```

**Fix**: refactor — extract shared interface, или use events.

### 5.6. Premature Generalization

```csharp
public interface IRepository<TEntity, TId, TFilter, TOrderBy>   // ❌ over-generic
{
    Task<List<TEntity>> Get(TFilter filter, TOrderBy order);
}
```

**Fix**: start simple, generalize when needed.

> [!question]- Интервью: топ-3 OOP anti-patterns в C#?
> 1) **God Object** — single class с 30+ methods, mixed responsibilities. Fix: split по SRP. Example: OrderManager doing payment + email + inventory + invoice. 2) **Anemic Domain Model** — entities только properties (data bags), all logic в services. Fix: rich entities (Order.Pay() enforces invariants). DDD principle. 3) **Service Locator** — `ServiceLocator.GetService<T>()` global access. Hidden dependencies, untestable. Fix: constructor injection (DI). Бонус: **Singleton with mutable state** — global concurrency nightmare. Use DI `AddSingleton<T>` для stateless services.

---

## 6. Cheat sheet — pattern одной строкой

```
=== Creational ===
Singleton          — single instance shared (DI AddSingleton<T>)
Factory Method     — subclass decides concrete type (DI Func<T>)
Abstract Factory   — family of related products (cross-platform UI)
Builder            — step-by-step complex construction (HostBuilder)
Prototype          — clone existing instance (record with)

=== Structural ===
Adapter            — incompatible interface → expected interface
Bridge             — separate abstraction from implementation (orthogonal hierarchies)
Composite          — tree structure, uniform handling (file system)
Decorator          — add behavior dynamically (caching, logging, retry)
Facade             — simple interface to complex subsystem
Flyweight          — share instances для memory (string interning)
Proxy              — surrogate / placeholder (Lazy<T>, remote)

=== Behavioral ===
Chain of Responsibility — pipeline of handlers (middleware)
Command            — encapsulate request as object (MediatR)
Iterator           — sequential access (yield, IEnumerable<T>)
Mediator           — central coordinator (MediatR, SignalR hubs)
Memento            — capture state snapshot (records, undo/redo)
Observer           — one-to-many notifications (event, RX)
State              — behavior depends on state (pattern matching)
Strategy           — interchangeable algorithms (Func<T>, IComparer<T>)
Template Method    — algorithm skeleton + override hooks (BackgroundService)
Visitor            — operations on hierarchies (pattern matching, ExpressionVisitor)
Interpreter        — DSL grammar (Expression Trees)
```

---

## 7. Common pitfalls

### 7.1. Pattern soup

Combining 3+ patterns в same class — usually over-engineering. Start simple.

### 7.2. Pattern для name's sake

"Use Strategy для tax calculation" → write 5 classes when `Func<decimal, decimal>` lambda enough.

### 7.3. Manual when language has feature

```csharp
// ❌ Manual Iterator
public class CustomCollection : IEnumerable<int>
{
    public IEnumerator<int> GetEnumerator() => new CustomEnumerator();
    // Custom IEnumerator с manual MoveNext, Current
}
```

**Fix:** `yield return`.

### 7.4. Singleton instead of DI

```csharp
// ❌ Manual
public sealed class CacheService
{
    private static CacheService? _instance;
    public static CacheService Instance => _instance ??= new();
}
```

**Fix:** `services.AddSingleton<ICacheService, CacheService>()`.

### 7.5. Generic `Repository<T>` abuse

```csharp
public interface IRepository<T> { Task<T?> GetByIdAsync(int id); /* ... */ }
public class UserRepository : IRepository<User> { /* ... */ }
public class OrderRepository : IRepository<Order> { /* ... */ }
// ... 50 more identical
```

**Fix:** EF Core IS the repository. Specific repos если specific queries нужны.

### 7.6. Decorator chain без сause

```csharp
services.AddScoped<IService, RealService>();
services.Decorate<IService, LoggingDecorator>();
services.Decorate<IService, CachingDecorator>();
services.Decorate<IService, RetryDecorator>();
services.Decorate<IService, ValidationDecorator>();
services.Decorate<IService, MetricsDecorator>();
// 6 layers — debugging hell
```

**Fix:** combine cross-cutting concerns wisely. ASP.NET middleware OK, repository decorator chains 2-3 max.

### 7.7. State pattern для simple enum

```csharp
// ❌ Overkill — class per state
public abstract class OrderState { /* ... */ }
public class PendingState : OrderState { /* ... */ }
public class PaidState : OrderState { /* ... */ }
public class ShippedState : OrderState { /* ... */ }
public class CancelledState : OrderState { /* ... */ }
public class DeliveredState : OrderState { /* ... */ }
```

**Fix:** pattern matching switch для simple enums.

### 7.8. Mediator everywhere

```csharp
// ❌ MediatR для каждого method call
public async Task<User> GetUser(int id) =>
    await _mediator.Send(new GetUserQuery(id));
// Just call repository directly
```

**Fix:** Mediator для cross-cutting (events, complex commands), не каждый method.

### 7.9. Visitor manual

```csharp
public interface IShapeVisitor { void Visit(Circle c); /* ... */ }
public abstract class Shape { public abstract void Accept(IShapeVisitor v); }
public class Circle : Shape { public override void Accept(IShapeVisitor v) => v.Visit(this); }
// boilerplate-heavy
```

**Fix:** pattern matching switch.

### 7.10. Builder для simple objects

```csharp
public class UserBuilder
{
    private string _name;
    public UserBuilder WithName(string n) { _name = n; return this; }
    public User Build() => new(_name);
}
// User has 1 property — overkill
```

**Fix:** direct constructor or `record User(string Name)`.

> [!question]- Интервью: топ-3 ошибки с design patterns?
> 1) **Pattern для name's sake** — write Strategy + Factory + Builder для simple business logic (tax calculation). `Func<decimal, decimal>` enough. 2) **Manual when language has feature** — write Iterator instead of `yield return`, write Visitor instead of `pattern matching switch`, write Singleton instead of `services.AddSingleton<T>()`. 3) **Pattern soup** — combining 3+ patterns в same class. Each pattern adds complexity; cumulative effect = unmaintainable. **Bonus**: Generic `Repository<T>` для каждого entity без specific variation. EF Core IS the repository. **Best practice 2024+**: pattern thinking essential, manual implementation often anti-pattern. Know when language replaces pattern.

---

## 8. Practice — recognize patterns

### 8.1. Identify pattern

```csharp
// Q1
public interface ICommand { void Execute(); void Undo(); }
public class TextEditor { public Stack<ICommand> History = new(); }
// → Command pattern (with Memento implicit для Undo)

// Q2
public class StreamReader : TextReader { /* wraps Stream */ }
public class BufferedStream : Stream { /* wraps Stream */ }
public class GZipStream : Stream { /* wraps Stream */ }
// → Decorator (chain of Stream wrappers)

// Q3
public abstract class HttpHandler { protected HttpHandler? Next; }
app.UseAuthentication().UseAuthorization().UseRouting();
// → Chain of Responsibility

// Q4
public class HostBuilder
{
    public HostBuilder ConfigureServices(...) { return this; }
    public HostBuilder ConfigureLogging(...) { return this; }
    public IHost Build() { /* ... */ }
}
// → Builder

// Q5
public abstract record Shape;
public record Circle(double Radius) : Shape;
public record Square(double Side) : Shape;
double Area(Shape s) => s switch { Circle c => ..., Square sq => ... };
// → Visitor (replaced by pattern matching)

// Q6
public class StockTicker
{
    public event EventHandler<PriceChangedArgs>? PriceChanged;
}
// → Observer

// Q7
public class Logger
{
    private static readonly Lazy<Logger> _instance = new(() => new Logger());
    public static Logger Instance => _instance.Value;
}
// → Singleton

// Q8
public class HttpClient { /* facade over HttpMessageHandler chain */ }
// → Facade
```

### 8.2. Modernize legacy code

```csharp
// Legacy — manual Iterator
public class CustomCollection : IEnumerable
{
    private int[] _data;
    public IEnumerator GetEnumerator() => new CustomEnumerator(_data);
}

public class CustomEnumerator : IEnumerator
{
    private int[] _data;
    private int _index = -1;
    public CustomEnumerator(int[] data) => _data = data;
    public bool MoveNext() => ++_index < _data.Length;
    public object? Current => _data[_index];
    public void Reset() => _index = -1;
}

// Modern
public class CustomCollection
{
    private int[] _data;
    public IEnumerable<int> GetItems()
    {
        foreach (var item in _data) yield return item;
    }
}
```

```csharp
// Legacy — manual Singleton
public sealed class ConfigService
{
    private static ConfigService? _instance;
    private static readonly object _lock = new();
    public static ConfigService Instance
    {
        get
        {
            if (_instance == null)
                lock (_lock)
                    _instance ??= new ConfigService();
            return _instance;
        }
    }
}

// Modern
services.AddSingleton<ConfigService>();
```

### 8.3. Decompose God class

```csharp
// Before — God class
public class OrderProcessor
{
    public async Task Process(Order order)
    {
        // validate
        if (order.Items.Count == 0) throw new InvalidOperationException();
        
        // calculate tax
        decimal tax = order.Country == "US" ? order.Subtotal * 0.08m : order.Subtotal * 0.20m;
        order.Total = order.Subtotal + tax;
        
        // charge card
        await _stripe.ChargeAsync(order.Customer.CardToken, order.Total);
        
        // update inventory
        foreach (var item in order.Items)
            await _db.UpdateStockAsync(item.ProductId, -item.Quantity);
        
        // send email
        await _smtp.SendAsync(order.Customer.Email, "Order confirmation", $"...");
        
        // log
        _logger.LogInformation("Order {Id} processed", order.Id);
        
        // schedule delivery
        await _ups.ScheduleAsync(order.Items, order.Address);
        
        // ... 200 more lines
    }
}

// After — split + DI
public class OrderProcessor
{
    private readonly IOrderValidator _validator;
    private readonly ITaxCalculator _tax;
    private readonly IPaymentService _payment;
    private readonly IInventoryService _inventory;
    private readonly INotificationService _notification;
    private readonly IShippingService _shipping;
    private readonly ILogger<OrderProcessor> _logger;
    
    public OrderProcessor(/* DI */) { /* ... */ }
    
    public async Task Process(Order order)
    {
        _validator.Validate(order);
        var pricing = _tax.Calculate(order);
        await _payment.ChargeAsync(order.Customer, pricing.Total);
        await _inventory.ReserveAsync(order.Items);
        await _notification.SendOrderConfirmationAsync(order);
        await _shipping.ScheduleAsync(order);
        _logger.LogInformation("Order {Id} processed", order.Id);
    }
}
```

---

## 9. Что читать дальше

1. **[[design-patterns|Design Patterns]]** — overview + SOLID + DDD.
2. **[[functional-csharp|Functional C#]]** — alternatives к OOP patterns.
3. **GoF — "Design Patterns" (1994)** — original book.
4. **Robert Martin — Clean Architecture**.
5. **Refactoring.Guru** — refactoring.guru/design-patterns/csharp.
6. **Steve Smith — ASP.NET Core patterns** — ardalis.com.

---

## 10. См. также

- [[design-patterns|Design Patterns Overview]]
- [[oop|OOP]]
- [[functional-csharp|Functional C#]]
- [[reflection-expression-trees|Reflection]] — Specification, Expression Trees
- [[modern-features|Modern Features]] — pattern matching
- MediatR — github.com/jbogard/MediatR
- Scrutor — github.com/khellang/Scrutor

---

## 11. Reading list

- **GoF — "Design Patterns: Elements of Reusable OO Software"** (1994) — original
- **Robert Martin — "Clean Architecture"** (2017)
- **Robert Martin — "Clean Code"**
- **Eric Freeman — "Head First Design Patterns"** — beginner-friendly
- **Refactoring.Guru** — refactoring.guru/design-patterns/csharp
- **Sourcemaking** — sourcemaking.com/design_patterns
- **Steve Smith — ASP.NET Core** — ardalis.com
- **Jimmy Bogard — Domain Events** — jimmybogard.com
- **Vladimir Khorikov** — DDD courses, Pluralsight
- **Microsoft Docs — Design patterns** — learn.microsoft.com
