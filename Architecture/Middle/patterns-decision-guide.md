---
tags: [architecture, patterns, decision-guide, design-patterns, integration, senior]
level: Middle to Senior
date: 2026-08-02
---

# Patterns & Architecture — Decision Guide

> **Главный навигационный файл vault'а**: под какую задачу что выбрать. Связывает GoF паттерны, архитектурные стили, DDD, CQRS, Mediator, Repository — в один decision framework. Closes пробел "знаю отдельные patterns, не понимаю как они складываются вместе и когда какой выбрать".

---

## Что это, зачем и когда

### Зачем этот файл существует

Vault содержит **много отдельных файлов** про patterns:

- `CSharp/design-patterns.md` — 13 GoF patterns (Strategy, Factory, Decorator, ...)
- `Architecture/Senior/architecture-patterns.md` — N-Layer / Clean / VSA / Hybrid
- `Architecture/solid.md` — SOLID, DRY, KISS, YAGNI
- `Architecture/ddd.md` — Bounded Contexts, Aggregates, Value Objects
- `Architecture/cqrs-mediatr.md` — Command/Query separation
- `Architecture/microservices-vs-monolith.md` — when split
- `EFCore/Senior/ef-patterns.md` — Repository, UoW, Specification
- `Snippets/result-pattern.md` — Result\<T, E\>

**Проблема:** Senior должен знать **когда что использовать**, не просто "что это". Нет одного места которое объединяет всё.

### Уровни паттернов

```
┌─────────────────────────────────────────┐
│ System / Distributed                    │  microservices, saga, CQRS, event        sourcing                                │
│  ↑                                      │
├─────────────────────────────────────────┤
│ Application architecture                │  Clean, VSA, N-Layer
│  ↑                                      │
├─────────────────────────────────────────┤
│ Domain modeling                         │  DDD, Aggregates, Value Objects
│  ↑                                      │
├─────────────────────────────────────────┤
│ Code organization                       │  SOLID, Repository, UoW
│  ↑                                      │
├─────────────────────────────────────────┤
│ Implementation (GoF)                    │  Strategy, Factory, Observer, ...
└─────────────────────────────────────────┘
```

**Каждый уровень** имеет свои patterns. Senior выбирает на каждом уровне правильно.

### Главное правило выбора

> **Не паттерн ради паттерна.** Сначала задача → потом простейшее решение → если есть pain point → паттерн.

Большинство bugs от **over-engineering** (слишком много паттернов рано) **или** **under-engineering** (никаких patterns — спагетти).

---

## 1. Какая архитектура для проекта

### Decision matrix

| Размер / тип                               | Архитектура                                             |
| ------------------------------------------ | ------------------------------------------------------- |
| Прототип, MVP, demo                        | **Single project + minimal API** — нет архитектуры      |
| Маленькая внутренняя tool, < 5 entities    | **N-Layer (3 проекта)** — Controllers / Services / Data |
| Stable domain, 5–30 entities               | **Clean Architecture** — Domain centric                 |
| Много feature-команд, fast iteration       | **Vertical Slices (VSA)** — feature folders             |
| Крупный продукт, complex domain            | **Hybrid: Clean + VSA + DDD** — лучшее обоих            |
| Несколько bounded contexts, разные команды | **Modular Monolith** — modules + DI boundaries          |
| Independent teams, scaling, polyglot       | **Microservices** (но осторожно — costs)                |

См. [[architecture-patterns|Architecture Patterns]] для глубокого сравнения.

### Триггеры — когда менять

```
Прототип → N-Layer:
  - Уже 3+ entity, бизнес-логика появилась

N-Layer → Clean Architecture:
  - Сложная domain логика
  - Тестирование стало больно (setup мега-длинный)
  - Unit-тесты требуют DB / HTTP mocks везде

Clean → Hybrid (+ VSA):
  - Много teams parallel
  - Cross-cutting features размазываются по слоям
  - Open-close violations (часто менять много слоев)

Modular Monolith → Microservices:
  - Independent scaling нужен
  - Polyglot persistence / languages
  - Team boundaries clear
  - Готовы к network costs
```

> [!warning] Microservices — last resort
> Operational complexity растёт **экспоненциально**. Сначала Modular Monolith, только потом split.

См. [[microservices-vs-monolith|Microservices vs Monolith]].

---

## 2. Под какую задачу — какой GoF паттерн

### Algorithm / behavior selection

**Задача:** "У меня есть несколько способов сделать X — нужно выбрать в runtime"

→ **Strategy**

```csharp
// Способы compress: gzip, brotli, deflate
public interface ICompressor { byte[] Compress(byte[] data); }
public class GzipCompressor : ICompressor { /* ... */ }
public class BrotliCompressor : ICompressor { /* ... */ }

// Caller выбирает в runtime
ICompressor compressor = config.Algorithm switch
{
    "gzip" => new GzipCompressor(),
    "brotli" => new BrotliCompressor(),
    _ => throw new NotSupportedException()
};
```

**Когда:** Каждое if-elif-elif которое выбирает algorithm — кандидат на Strategy.

См. [[design-patterns#Strategy|Strategy]].

### Object creation

**Задача:** "Создание объекта сложное — нужно скрыть детали"

→ **Factory** (если type меняется), **Builder** (если много параметров)

```csharp
// Factory — выбирает type
public class NotificationFactory
{
    public INotifier Create(NotificationType type) => type switch
    {
        NotificationType.Email => new EmailNotifier(...),
        NotificationType.SMS => new SmsNotifier(...),
        _ => throw new NotSupportedException()
    };
}

// Builder — много optional params
var query = new QueryBuilder()
    .From("Users")
    .Where("Age", ">", 18)
    .OrderBy("Name")
    .Limit(100)
    .Build();
```

**В .NET:**
- DI container = factory of services
- `HttpClientFactory`, `LoggerFactory` — built-in
- `StringBuilder`, `HostBuilder`, `WebApplication.CreateBuilder()` — examples Builder

### Add behavior без modify

**Задача:** "Хочу добавить логирование / caching / retry к существующему service без изменения его кода"

→ **Decorator**

```csharp
public interface IUserService { Task<User> GetById(int id); }

public class UserService : IUserService { /* main logic */ }

public class CachingUserService : IUserService
{
    private readonly IUserService _inner;
    private readonly IMemoryCache _cache;

    public async Task<User> GetById(int id) =>
        await _cache.GetOrCreateAsync(id, _ => _inner.GetById(id));
}

public class LoggingUserService : IUserService
{
    private readonly IUserService _inner;
    
    public async Task<User> GetById(int id)
    {
        _logger.LogInformation("Get user {Id}", id);
        return await _inner.GetById(id);
    }
}

// Wire в DI
services.AddScoped<IUserService>(sp =>
    new LoggingUserService(
        new CachingUserService(
            new UserService(...))));
```

**В ASP.NET:** middleware = decorator pipeline. `DelegatingHandler` для HttpClient — decorator.

См. [[pipeline-middleware|Pipeline & Middleware]].

### Notification / pub-sub

**Задача:** "Объект должен сообщать другим о событиях"

→ **Observer** (для in-process), **Mediator** (для domain), **Message bus** (для cross-service)

```csharp
// In-process: C# events (Observer built-in)
public class Order
{
    public event EventHandler<OrderPlacedEventArgs> Placed;
}

// Domain: in-process events — свой dispatcher или Mediator (source-gen)
public class OrderPlaced : INotification { /* ... */ }
public class SendEmailHandler : INotificationHandler<OrderPlaced> { /* ... */ }

// Cross-service: RabbitMQ / Kafka
await bus.Publish(new OrderPlaced(...));
```

**Когда что:**
- **Events** — same process, tight coupling OK
- **Mediator** — same process, decoupled (handlers regstrируются автоматом). Дефолт — свой dispatcher (~50 строк) или `Mediator` (source-gen, MIT); MediatR 13+ — коммерческий, см. [[choosing-dependencies|Choosing Dependencies]]
- **Message bus** — между процессами / services

См. [[cqrs-mediatr|CQRS & Mediator]] и [[messaging|Messaging]].

### Adapter — incompatible interfaces

**Задача:** "Library требует interface X, у меня объект с interface Y"

→ **Adapter**

```csharp
// Legacy API
public class LegacyPaymentApi
{
    public bool ProcessPayment(double amount, string cc) { /* ... */ }
}

// Modern interface моего app
public interface IPaymentProcessor
{
    Task<Result> Process(decimal amount, PaymentMethod method);
}

// Adapter
public class LegacyPaymentAdapter : IPaymentProcessor
{
    private readonly LegacyPaymentApi _legacy;

    public async Task<Result> Process(decimal amount, PaymentMethod method)
    {
        var success = _legacy.ProcessPayment((double)amount, method.CardNumber);
        return success ? Result.Ok() : Result.Fail("Payment failed");
    }
}
```

**Когда:** при integration со старым / 3rd-party кодом.

### Facade — упрощение complex API

**Задача:** "У меня 5 сервисов которые надо вызывать в правильном порядке для одной операции"

→ **Facade**

```csharp
// Без facade — caller вызывает 5 services
public IActionResult PlaceOrder(...)
{
    var user = userService.Validate(...);
    var inventory = inventoryService.Reserve(...);
    var payment = paymentService.Charge(...);
    var order = orderService.Create(...);
    notificationService.Send(...);
}

// Facade — один entry point
public class OrderingFacade
{
    public async Task<Order> PlaceOrder(PlaceOrderCommand cmd)
    {
        // orchestration внутри
    }
}
```

**В Clean Architecture:** Application Service / Use Case = Facade.

### State machine

**Задача:** "Объект имеет состояния, поведение зависит от состояния"

→ **State** (если состояний > 5 и сложные transitions), **switch expression** (если просто)

```csharp
// Простой — switch expression
public Order Submit() => Status switch
{
    OrderStatus.Draft => this with { Status = OrderStatus.Submitted },
    _ => throw new InvalidOperationException()
};

// Сложный — State pattern
public abstract class OrderState
{
    public abstract OrderState Submit(Order o);
    public abstract OrderState Cancel(Order o);
    public abstract OrderState Approve(Order o);
}

public class DraftState : OrderState { /* allowed: Submit */ }
public class SubmittedState : OrderState { /* allowed: Approve, Cancel */ }
// ... etc
```

См. [[design-patterns#State|State Pattern]] и [[enums-flags#State-machine|State Machine via enum]].

### Filtering / specifications

**Задача:** "Хочу composable фильтры для query"

→ **Specification**

```csharp
var query = db.Users
    .Where(new ActiveSpec().And(new AdultSpec()).IsSatisfiedBy);

// Или EF-friendly через Expression
public class ActiveSpec : Specification<User>
{
    public override Expression<Func<User, bool>> ToExpression() =>
        u => u.IsActive;
}
```

См. [[ef-patterns#Specification|Specification Pattern]].

---

## 3. Domain logic — DDD vs anemic

### Когда DDD

✅ **Используй DDD когда:**
- Сложная business logic (правила на 100+ строк)
- Domain experts говорят на специфическом языке (ubiquitous language)
- Long-living product
- Senior developers в команде

❌ **НЕ используй когда:**
- CRUD app (просто create/read/update/delete entities)
- Simple data transformation
- Прототип
- Junior team без mentor

### Anemic vs Rich domain model

```csharp
// ❌ Anemic — entity это data bag, логика в service
public class Order
{
    public OrderStatus Status { get; set; }
    public List<OrderItem> Items { get; set; }
}

public class OrderService
{
    public void Submit(Order order)
    {
        if (order.Items.Count == 0) throw new ...;
        order.Status = OrderStatus.Submitted;
    }
}

// ✅ Rich domain — логика в entity
public class Order
{
    public OrderStatus Status { get; private set; }
    private readonly List<OrderItem> _items = new();
    
    public void Submit()
    {
        if (_items.Count == 0)
            throw new InvalidOperationException("Cannot submit empty order");
        Status = OrderStatus.Submitted;
        AddDomainEvent(new OrderSubmittedEvent(this));
    }
}
```

**Anemic OK для:** simple CRUD apps, DTOs, микро-сервисы с simple domain.

**Rich OK для:** сложная domain логика, DDD проекты, long-lived products.

См. [[ddd|DDD]] и [[oop|OOP]].

---

## 4. Data access — Repository / UoW / Direct DbContext

### Decision tree

```
EF Core?
│
├── Маленький проект, < 10 entities?
│   → Direct DbContext в Controller / Handler
│   → Не оборачивай в Repository "на всякий"
│
├── Средний проект, тестируемость важна?
│   → DbContext + Specifications для queries
│   → Минимальный Repository если очень нужно
│
├── Большой проект, DDD?
│   → Generic Repository\<T\> + UoW
│   → Aggregate-specific repositories
│
└── Read-heavy / отчёты?
    → Dapper для read-only
    → EF для writes (CQRS-like)
```

### Repository pattern — overrated?

> [!info] Hot take от .NET community
> EF Core DbContext **уже implements Unit of Work + Repository**:
> - `DbSet<T>` = Repository\<T\>
> - `SaveChangesAsync` = UoW commit
> 
> Adding generic `IRepository<T>` поверх — часто **anti-pattern**.

```csharp
// ❌ Бесполезный Repository
public interface IUserRepository
{
    Task<User?> GetById(int id);
    Task<List<User>> GetAll();
    Task Save(User user);
}

public class UserRepository : IUserRepository
{
    public Task<User?> GetById(int id) => _db.Users.FindAsync(id);
    // ⚠️ Просто wrapper над DbSet — нет добавленной ценности
}

// ✅ Repository имеет смысл если:
// 1. Несколько data sources (EF + Dapper + Cache)
// 2. Aggregate с правилами (load aggregate root + children)
// 3. Mock для unit tests (но In-Memory provider обычно достаточен)
```

### Когда Repository ОК

✅ **Aggregate-specific** Repository в DDD:
```csharp
public interface IOrderRepository
{
    // Load full aggregate (Order + Items + Customer)
    Task<Order?> GetByIdAsync(OrderId id);
    
    // Domain-specific query
    Task<List<Order>> GetPendingByCustomerAsync(CustomerId id);
    
    void Add(Order order);  // SaveChanges делается в UoW
}
```

✅ **Multiple sources** (EF + Dapper):
```csharp
public class UserRepository : IUserRepository
{
    public Task<User?> GetById(int id) => _db.Users.FindAsync(id);  // EF
    public Task<UserStats> GetStats(int id) => _conn.QueryAsync<UserStats>(...);  // Dapper
}
```

См. [[ef-patterns|EF Patterns]] и [[dapper-comparison|Dapper vs EF]].

---

## 5. Cross-cutting concerns

### Где располагать общие вещи (logging, validation, caching)?

```
Cross-cutting     │ Где
──────────────────┼─────────────────────────────────────
Logging           │ Decorator + ILogger DI
                  │ Или mediator pipeline behavior
Validation        │ FluentValidation + pipeline behavior
                  │ Или DataAnnotations + ModelState
Authorization     │ ASP.NET filters / [Authorize] attribute
                  │ Или mediator pipeline behavior
Caching           │ Decorator (CachingUserService)
                  │ Или output caching middleware
Transactions      │ Pipeline behavior (BeginTransaction/Commit)
                  │ Или decorator
Retry / circuit   │ Polly через DelegatingHandler
                  │ Или Polly в HttpClientFactory
Audit             │ EF SaveChanges interceptor
                  │ Или decorator
```

### Pipeline behaviors — best для CQRS

Код ниже валиден для MediatR ≤12.x и для `Mediator` (source-gen) — API идентичен; MediatR 13+ — коммерческий (dual-license с июля 2025), дефолт vault — свой dispatcher, см. [[choosing-dependencies|Choosing Dependencies]].

```csharp
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var failures = _validators
            .Select(v => v.Validate(request))
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();
            
        if (failures.Any())
            throw new ValidationException(failures);
            
        return await next();
    }
}

services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```

См. [[cqrs-mediatr|CQRS & MediatR]].

---

## 6. Communication patterns

### Synchronous (in-process)

```
Same process, tight:    Method call
Same process, loose:    Mediator (свой dispatcher / Mediator source-gen)
Same machine, decouple: Named pipes / gRPC
```

См. [[ipc-named-pipes-grpc|IPC]].

### Synchronous (cross-service)

```
Internal services:      gRPC (faster, contract)
Public APIs:            REST (HTTP/JSON)
Real-time push:         SignalR / WebSocket
Federated query:        GraphQL (один endpoint, flexible)
```

См. [[api-design|API Design]] и [[graphql|GraphQL]].

### Asynchronous

```
Fire-and-forget:        Message queue (RabbitMQ / SQS / Kafka)
Event-driven:           Pub/Sub (Kafka / Azure Service Bus)
Workflow / saga:        Wolverine (MIT) / Dapr; MassTransit v9 — коммерческий
Reliable retry:         Outbox pattern (DB + bus)
```

См. [[messaging|Messaging]] и [[distributed-systems|Distributed Systems]].

### Decision tree

```
Нужен response?
├── Yes, fast? → gRPC / direct HTTP
├── Yes, optional? → REST
└── No → Message bus (async)

Несколько consumers?
├── Yes → Pub/Sub (Kafka topics)
└── No → Queue (RabbitMQ)

Workflow с steps?
├── Saga (compensation) → Wolverine (MIT, ядро OSS);
│     MassTransit v9 (Q1 2026) и NServiceBus — коммерческие,
│     MassTransit v8 — Apache 2.0, но security-only до EOL конец 2026
└── Sequential → Outbox + handlers

Real-time UI?
└── SignalR
```

---

## 7. Error handling — что выбрать

### Стратегии

| Подход | Когда |
|--------|-------|
| **Throw exceptions** | Critical errors, invariant violations, unexpected |
| **Result\<T, E\>** | Expected failures (validation, not found, conflict) |
| **`Maybe<T>` / nullable** | "Может быть нет значения" — без error context |
| **`OneOf<T1, T2, T3>`** | Discriminated union — multiple result types |

### Pattern by layer

```
Layer            │ Approach
─────────────────┼──────────────────────────────
Controller       │ Result → HTTP status mapping
Application      │ Result<T, Error> для business logic
Domain           │ Exception для invariants
                 │ Result для expected failures
Infrastructure   │ Exception для transient (DB, HTTP)
                 │ Polly handles retries
```

### Result pattern в практике

```csharp
public sealed record Result<T>
{
    public T? Value { get; init; }
    public string? Error { get; init; }
    public bool IsSuccess => Error is null;
    
    public static Result<T> Ok(T value) => new() { Value = value };
    public static Result<T> Fail(string error) => new() { Error = error };
}

// Application layer
public async Task<Result<Order>> PlaceOrder(PlaceOrderCommand cmd)
{
    var validation = Validate(cmd);
    if (!validation.IsSuccess) return Result<Order>.Fail(validation.Error);
    
    try
    {
        var order = await _repo.Save(...);
        return Result<Order>.Ok(order);
    }
    catch (DbUpdateConcurrencyException)
    {
        return Result<Order>.Fail("Order was modified");
    }
}

// Controller
public async Task<IActionResult> Place(PlaceOrderRequest req)
{
    var result = await _mediator.Send(req.ToCommand());
    return result.IsSuccess
        ? Created(...)
        : BadRequest(result.Error);
}
```

См. [[error-handling|Error Handling]] и [[result-pattern|Result Pattern Snippet]].

---

## 8. Concurrency / data consistency

### Optimistic vs Pessimistic locking

| | Optimistic | Pessimistic |
|--|-----------|-------------|
| Mechanism | Version / timestamp check | DB row lock |
| Conflict | Throw on commit | Wait |
| Performance | Высокая (no locks) | Низкая (locks) |
| Use case | Read-heavy, low conflict | Write-heavy, high conflict |
| EF Core | `[Timestamp]` / `IsConcurrencyToken` | `WITH (UPDLOCK)` (manual SQL) |

```csharp
// Optimistic в EF Core
public class Order
{
    public byte[] RowVersion { get; set; }   // [Timestamp]
}

try
{
    await db.SaveChangesAsync();
}
catch (DbUpdateConcurrencyException)
{
    // Кто-то ещё обновил — retry или error
}
```

См. [[concurrency|EF Concurrency]].

### Distributed transactions

```
Two-Phase Commit (2PC) → ❌ Не используй (medieval)
Saga pattern (orchestration) → workflow engine
Saga pattern (choreography) → events
Outbox pattern → guaranteed delivery
TCC (Try-Confirm-Cancel) → если нужен
```

См. [[distributed-systems|Distributed Systems]].

---

## 9. CQRS — когда нужен

### Когда CQRS оправдан

✅ **Используй CQRS:**
- Read и write **очень разные** (reports vs CRUD)
- Read scale нужен независимо от write
- Multiple read models из одного write (reporting + UI + search)
- Event sourcing уже есть

❌ **НЕ используй когда:**
- Просто хочешь "разделить query и command классы" — это **не CQRS**
- Маленький CRUD проект
- Нет performance / complexity issues

### Уровни CQRS

```
Level 1: Command/Query метод separation в одном service
  → Просто convention, easy

Level 2: Отдельные Command/Query handlers (прямой вызов или свой dispatcher)
  → Single DB, разные models — стандарт большинства проектов

Level 3: Разные read/write databases
  → SQL для writes, ElasticSearch для reads
  → Eventually consistent

Level 4: Event Sourcing + CQRS
  → Event store как source of truth
  → Read models projections
  → Высокая сложность
```

> [!warning] CQRS ≠ MediatR
> Многие думают что добавив MediatR + Commands/Queries — они "сделали CQRS". На самом деле это просто Mediator pattern. Real CQRS — separation of read and write models.

См. [[cqrs-mediatr|CQRS & MediatR]].

---

## 10. Real-world комбо: что с чем сочетается

### Combo 1: Maly internal tool (1-2 dev, 1 month)

```
Architecture:    N-Layer (3 проекта)
DI:              Built-in Microsoft.Extensions.DependencyInjection
Data:            EF Core + Direct DbContext
Patterns:        Strategy (если выбор algorithm), Decorator (logging)
Error handling:  Exceptions
Communication:   Direct method calls
Tests:           Unit + few integration
```

Не over-engineer. Простота rules.

### Combo 2: Mid-sized SaaS (3-5 dev, 1 year)

```
Architecture:    Clean Architecture + Feature Folders
DI:              + Scrutor для assembly scanning
Data:            EF Core + DbContext + Specifications
Patterns:        Strategy, Decorator, Specification, Result
Error handling:  Result<T> в Application, exceptions в Infra
Communication:   Vertical slices — handler напрямую (или свой dispatcher) + REST API
Validation:      FluentValidation в pipeline behavior / endpoint filter
Tests:           Unit + integration (TestContainers)
```

Sweet spot для большинства проектов 2026.

### Combo 3: Large product (10+ dev, multi-team)

```
Architecture:    Modular Monolith + DDD + VSA внутри modules
DI:              Per-module registration
Data:            EF Core per module, separate schemas
Patterns:        Aggregate, Value Object, Domain Events
Error handling:  Result<T> + Domain exceptions
Communication:   Modules через contracts-assembly + in-process dispatcher,
                 outbox для cross-module integration events
Validation:      FluentValidation
Tests:           Unit + integration + arch tests
Observability:   OpenTelemetry, Serilog structured
```

См. [[observability|Observability]] и [[arch-tests|Architecture Tests]].

### Combo 4: Microservices (multiple teams)

```
Architecture:    Microservices (per bounded context)
                 Каждый сервис — Clean Arch внутри
DDD:             Strategic (контексты), Tactical (Aggregates)
Data:            Per-service DB (no sharing)
Communication:   gRPC internal, REST external, Kafka events
Patterns:        Saga, Outbox, Circuit Breaker (Polly), CQRS если нужен
Deploy:          Kubernetes + Helm
Observability:   Distributed tracing (Jaeger), logs (Loki), metrics (Prom)
Tests:           Unit + integration + contract testing (Pact)
```

Только когда **реально нужно**.

---

## 11. Anti-patterns — чего избегать

### "Patterns ради patterns"

```csharp
// ❌ Generic Repository поверх EF Core (бесполезный wrapper)
public interface IRepository<T>
{
    Task<T> GetById(int id);
    Task Save(T entity);
}
// Просто DbSet<T>

// ❌ "Service Layer" который только перепроксирует Repository
public class UserService
{
    public Task<User> GetById(int id) => _repo.GetById(id);
}

// ❌ Mapping везде (DTO → Domain → DTO → ViewModel)
// 4 разных типа для одной сущности — overhead
```

### "Anemic Domain Model в DDD проекте"

```csharp
// ❌ DDD-claim, но логика в services
public class Order { public OrderStatus Status; public List<Item> Items; }
public class OrderService { public void Submit(Order o) { ... } }

// ✅ Real DDD
public class Order
{
    public void Submit() { /* validation + state change here */ }
}
```

### "God service" / "Manager class"

```csharp
// ❌ Огромный service который делает всё
public class UserManager
{
    public Task Login(...);
    public Task Register(...);
    public Task SendEmail(...);
    public Task ResetPassword(...);
    public Task UpdateProfile(...);
    public Task GenerateReport(...);
    // 50+ методов
}

// ✅ Разделить по responsibility (SOLID/SRP)
public class AuthenticationService { /* login/register */ }
public class PasswordResetService { /* reset only */ }
public class ProfileService { /* profile updates */ }
```

### "Premature abstraction"

```csharp
// ❌ Interface для чего-то что никогда не имеет 2-ю implementation
public interface ICalculator
{
    int Add(int a, int b);
}
public class Calculator : ICalculator { /* ... */ }
// YAGNI! Просто class.
```

### "Over-mocked tests"

```csharp
// ❌ Mock всего → тестируешь mock'и, не код
[Test]
public void Test()
{
    var repo = new Mock<IUserRepository>();
    var emailService = new Mock<IEmailService>();
    var cache = new Mock<ICache>();
    var validator = new Mock<IValidator>();
    // ... 10 mocks
    // Test passes но реальный код может не работать
}

// ✅ Integration test с in-memory DB / TestContainers
[Test]
public async Task Test()
{
    using var container = await StartPostgresContainer();
    var service = new UserService(realDb);
    // ...
}
```

См. [[mocking-strategies|Mocking Strategies]].

---

## 12. Decision flowchart — финальный

```
Новый проект?
│
├── Прототип / 1-week MVP?
│   → Single project, no architecture
│   → Throwaway code, lessons learned
│
├── Внутренняя tool, 1-2 dev?
│   → N-Layer (3 проекта)
│   → Built-in DI + EF Core
│   → Few patterns (Decorator для logging)
│
├── SaaS / B2B product?
│   ├── Domain простой? → Clean Architecture lite
│   └── Domain сложный? → Clean + DDD
│   → Vertical slices: handler напрямую или свой dispatcher
│     (MediatR 13+ коммерческий) + FluentValidation + Result pattern
│   → EF Core + Specifications
│
├── Multi-team large product?
│   → Modular Monolith + DDD per module
│   → Postpone microservices
│
└── Известно: needs scale, polyglot, independent teams?
    → Microservices (carefully)
    → Service mesh, observability, CI/CD must
```

---

## 13. Что почитать дальше

### Books

- **Domain-Driven Design** — Eric Evans (классика DDD)
- **Implementing Domain-Driven Design** — Vaughn Vernon (practical DDD)
- **Patterns of Enterprise Application Architecture** — Martin Fowler (PEAA)
- **Design Patterns** — Gang of Four (классика GoF)
- **Clean Architecture** — Robert Martin
- **Building Microservices** — Sam Newman
- **Designing Data-Intensive Applications** — Martin Kleppmann

### Online

- **Microsoft .NET Architecture Guide** — learn.microsoft.com/dotnet/architecture
- **Microsoft eShopOnContainers** — github.com/dotnet-architecture/eShopOnContainers (real-world reference)
- **CleanArchitecture template** — github.com/jasontaylordev/CleanArchitecture
- **Modular Monolith template** — github.com/kgrzybek/modular-monolith-with-ddd
- **Vaughn Vernon — DDD talks** — youtube
- **Andrew Lock — blog** — andrewlock.net
- **Steve Smith — blog** — ardalis.com

---

## Case Studies

### Case Study #1 — Greenfield project — что выбрать

**Сценарий:** Новый SaaS, 2-3 разработчика. Нужно решить архитектуру за 1 день.

**Decision:**
1. **Modular monolith** (не microservices) — single deploy
2. **Clean Architecture** — Domain / App / Infrastructure / Web
3. **CQRS Light** — command/query separation через прямые handler-вызовы или свой dispatcher (MediatR 13+ коммерческий, см. [[choosing-dependencies|Choosing Dependencies]]), одна DB
4. **PostgreSQL + EF Core** — proven, no premature NoSQL
5. **Vertical Slice внутри Application** — feature folders
6. **Docker compose для dev** — Postgres + Redis локально

**Why:** простота важнее модности. Refactor → microservices когда есть **specific** pain.

См. [[microservices-vs-monolith|Microservices vs Monolith]] и [[patterns-decision-guide|Patterns Decision Guide]].

---

### Case Study #2 — Strangler fig migration

**Сценарий:** Legacy ASP.NET 4.8 monolith, 5 лет, 200K LOC. Нужно migrate на .NET 10.

**Strategy:**
1. **API Gateway** перед монолитом (YARP)
2. Identify **bounded contexts** в monolith
3. Extract **первый context** → new microservice
4. Gateway routes specific paths к new service
5. Repeat — context за context
6. Monolith постепенно "сжимается"

**Timeline:** 12-18 months для full migration.

См. [[api-gateway|API Gateway]].

---

### Case Study #3 — Multi-tenant SaaS architecture (конспект)

B2B SaaS с isolation по tenant: три стратегии — shared schema + TenantId (дёшево), schema-per-tenant (middle ground), database-per-tenant (full isolation, дорого). Порог выбора — количество tenants и regulations; типовой путь — start shared, критичных tenants выносить в dedicated DBs.

Полный разбор — [[architecture-decisions|ADRs / Case Study #3]]; реализация с EF Global Query Filter и RLS — [[real-world-scenarios|Real-World Scenarios]] (Scenario 15).


---

## Cheat sheet

| Concern | Pattern |
|---------|---------|
| Separation business logic | Clean Architecture (Domain center) |
| Read vs write models | CQRS |
| Asynchronous operations | Event-driven, message queues |
| Cross-cutting | Middleware, filters, decorators |
| Domain modeling | DDD (entities, aggregates, value objects) |
| Testability | DI + interfaces |
| Deployment isolation | Microservices (с правильной мотивацией) |
| Scaling parts independently | Microservices |
| Independence per team | Microservices |
| Avoid distributed transactions | Saga / Outbox |
| Decouple услуги | Message broker |
| API versioning | URL path (/v1/) или header |
| Multi-tenant | Schema/DB per tenant |
| Long-running ops | Background workers + queue |

| Style | When |
|-------|------|
| **Monolith** | < 5 dev, MVP, simple domain |
| **Modular monolith** | 5-20 dev, mid complexity |
| **Microservices** | 20+ dev, mature ops, independent scaling |
| **Serverless** | Event-driven, sporadic load |
| **CQRS** | Different read/write requirements |
| **Event Sourcing** | Audit trail critical, temporal queries |
| **Hexagonal** | Multiple input/output adapters |


---

## Best Practices

### Best Practices for Patterns Decision Guide

**Selection process:**
1. **Identify problem** — что именно болит?
2. **Constraints first** — performance / team / time
3. **Match patterns** к characteristics
4. **Spike + measure** — небольшой proof-of-concept
5. **Document decision** — ADR (Architecture Decision Record)

**Don't:**
- ❌ Apply pattern "potому что Senior знает его"
- ❌ Multiple patterns одновременно (combinatorial complexity)
- ❌ Premature pattern application (YAGNI)
- ❌ Ignore team familiarity

**Pattern fluency growth:**
- Junior — knows 5-7 GoF basics
- Middle — knows 15-20, can apply
- Senior — knows trade-offs, when NOT to use

**Documentation pattern in ADR:**
```markdown
# ADR 003: Use Strategy Pattern for Pricing

## Status
Accepted, 2026-05-01

## Context
Multiple pricing models (regular, discounted, wholesale, B2B contract).
Currently — switch statement in OrderService, hard to extend.

## Decision
Strategy pattern: IPriceCalculator with multiple implementations.

## Consequences
+ Easy to add new pricing models
+ Testable per strategy
- More files / classes
- Need DI setup для resolving correct strategy
```


---

## См. также

### GoF Patterns (implementation)

- [[design-patterns|Design Patterns]] — 13 GoF в C# контексте
- [[oop|OOP]] — fundamentals
- [[generics-deep|Generics Deep]] — generic patterns
- [[functional-csharp|Functional C#]] — FP patterns

### Architecture (macro)

- [[architecture-patterns|Architecture Patterns]] — N-Layer / Clean / VSA / Hybrid
- [[solid|SOLID]] — основы
- [[ddd|DDD]] — Domain-Driven Design
- [[cqrs-mediatr|CQRS & Mediator]]
- [[choosing-dependencies|Choosing Dependencies]] — лицензии зависимостей, линия замен (MediatR/AutoMapper/MassTransit)
- [[microservices-vs-monolith|Microservices vs Monolith]]
- [[distributed-systems|Distributed Systems]]
- [[system-design|System Design]]
- [[architecture-decisions|ADRs]]
- [[arch-tests|Architecture Tests]]

### Implementation patterns

- [[ef-patterns|EF Patterns]] — Repository, UoW, Specification
- [[dapper-comparison|Dapper vs EF]]
- [[error-handling|Error Handling]] — Result, exceptions
- [[pipeline-middleware|Pipeline & Middleware]]
- [[messaging|Messaging Patterns]]

### Quality

- [[clean-code|Clean Code]]
- [[refactoring|Refactoring]]
- [[code-review|Code Review]]

## Reading list

- **Refactoring.Guru — Patterns** — refactoring.guru/design-patterns (visual catalog)
- **Microsoft Cloud Design Patterns** — learn.microsoft.com/azure/architecture/patterns
- **Martin Fowler bliki** — martinfowler.com/bliki
- **Mark Seemann blog** — blog.ploeh.dk (DDD + FP в C#)
- **Khalid Abuhakmeh blog** — khalidabuhakmeh.com (.NET patterns)
- **Code With Mosh — Design Patterns** — youtube
