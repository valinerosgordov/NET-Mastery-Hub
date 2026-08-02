---
tags: [cqrs, mediator, mediatr, fastendpoints, vsa, pipeline-behaviors]
level: Senior
date: 2026-08-02
---

# CQRS и Mediator pattern

> Разделение Command/Query и развязка sender'а от handler'а через mediator с pipeline behaviors (validation, logging, transaction). Учитывает лицензионный сдвиг MediatR 13+ (dual-license с июля 2025) и альтернативы: свой dispatcher, `Mediator` (source-gen), FastEndpoints.

## Что это, зачем и когда

### Что такое CQRS?
**Command Query Responsibility Segregation** — разделение операций чтения (Query) и записи (Command). Команды меняют state и не возвращают данных, queries возвращают данные и не меняют state.

**Аналогия:** В кафе официант принимает заказы (Commands — изменяют состояние кухни) и отдельно выдаёт меню для просмотра (Queries — только читают). Один поток на запись, второй на чтение — не путаются.

### Зачем

| Без CQRS | С CQRS |
|----------|--------|
| Один service делает всё: validate, save, query, map | Каждый handler — одна ответственность |
| Сложно оптимизировать read paths — связаны с write | Read и write модели независимы (разные DB!) |
| Validation, logging, retry размазаны по сервисам | Pipeline behaviors применяются ко всем |
| Тесты проверяют сценарии целиком | Каждый handler тестируется отдельно |

### Уровни CQRS

| Уровень | Что | Когда |
|--------|-----|-------|
| **Light** | Просто разделение Command/Query классов в коде | Любой проект |
| **Medium** | Read и Write используют разные DB-проекции (read model) | Сложные системы с разной нагрузкой |
| **Heavy** | Event sourcing — write — events, read — projections | Large-scale, audit, replay |

В большинстве проектов — **Light CQRS**. Полноценный Event Sourcing — overkill для 90% backends.

### Когда применять

| Применять | Не применять |
|-----------|--------------|
| Vertical Slice Architecture (одна feature — одна папка) | Простой CRUD без логики |
| Несколько разных consumers одного действия | API на 10 endpoints |
| Pipeline cross-cutting concerns (validation, logging, retry) | Тривиальные сервисы |
| Большая команда — нужна изоляция features | Solo / 2-person team |

---

## Mediator pattern — что это

```
Клиент ─────► Mediator ─────► Handler1
              │              ─► Handler2
              │              ─► Handler3
              └─────► Pipeline behaviors (cross-cutting)
```

Sender не знает о Handler. Mediator смотрит тип Request → находит Handler → вызывает. Декouples отправителя от получателя.

В .NET — `MediatR` исторически стандарт.

---

## ⚠️ MediatR — лицензионный сдвиг (2025)

**2 апреля 2025** Jimmy Bogard анонсировал переход MediatR и AutoMapper под коммерческую модель (компания **Lucky Penny Software**); **2 июля 2025** — коммерческий запуск. Это **не форк** — сами библиотеки сменили модель: **MediatR 13+** и **AutoMapper 15+** распространяются под dual-license (RPL-1.5 или commercial).

| | MediatR ≤ 12.x | MediatR 13+ |
|--|----------------|-------------|
| Лицензия | Apache 2.0 — навсегда | Dual-license: RPL-1.5 или commercial |
| Community edition | — | Бесплатно при выручке < $5M **и** привлечённом капитале < $10M |
| Платные тиры | — | По размеру команды (не per-developer) |
| Новые features | Нет — версия заморожена | Только здесь |

Примеры кода ниже валидны для MediatR ≤ 12.x (и для одноимённых абстракций альтернатив).

### Что делать

**Стратегии для existing projects:**

1. **Проверить Community edition** — при выручке < $5M и капитале < $10M MediatR 13+ бесплатен. Зафиксировать в ADR: порог можно перерасти.
2. **Остаться на 12.x** (Apache 2.0 навсегда) — работает, но новых фич и активной разработки не будет.
3. **Мигрировать на альтернативу** — свой dispatcher, `Mediator` (source-gen), Wolverine, FastEndpoints.
4. **Заплатить** — тиры по команде; иногда дешевле миграции, если MediatR глубоко embedded.

Позиция этого vault — см. [[choosing-dependencies|Choosing Dependencies]]: дефолт — **свой in-process dispatcher (~50 строк) или прямые вызовы handler'а из endpoint'а**; mediator-абстракция нужна реже, чем кажется.

### Open-source alternatives

| Alternative | Особенности |
|-------------|-------------|
| **Mediator (Martin Othamar)** | Source-generated, AOT-friendly, performance better than MediatR. Разработка active |
| **Brighter** | Polymorphic mediator с command processor + outbox built-in |
| **Wolverine** | Полный microservices framework: mediator + sagas + scheduling |
| **DispatchR** | Минималистичный, no-frills mediator |
| **Roll your own** | Простой mediator пишется в 50 строк |

### Roll your own — простой mediator

```csharp
public interface IRequest<TResponse> { }
public interface IRequestHandler<TRequest, TResponse> where TRequest : IRequest<TResponse>
{
    Task<TResponse> Handle(TRequest request, CancellationToken ct);
}

public interface IMediator
{
    Task<TResponse> Send<TResponse>(IRequest<TResponse> request, CancellationToken ct = default);
}

public sealed class Mediator(IServiceProvider sp) : IMediator
{
    public async Task<TResponse> Send<TResponse>(IRequest<TResponse> request, CancellationToken ct = default)
    {
        var handlerType = typeof(IRequestHandler<,>).MakeGenericType(request.GetType(), typeof(TResponse));
        dynamic handler = sp.GetRequiredService(handlerType);
        return await handler.Handle((dynamic)request, ct);
    }
}

services.AddScoped<IMediator, Mediator>();
services.Scan(s => s.FromAssemblyOf<Program>()
    .AddClasses(c => c.AssignableTo(typeof(IRequestHandler<,>)))
    .AsImplementedInterfaces()
    .WithScopedLifetime());
```

50 строк — этого достаточно для 80% use cases. Боли с commercial license нет.

### Mediator (open-source альтернатива от Martin Othamar)

```bash
dotnet add package Mediator.SourceGenerator
dotnet add package Mediator.Abstractions
```

```csharp
public sealed record GetOrderQuery(Guid Id) : IRequest<OrderDto>;

public sealed class GetOrderHandler : IRequestHandler<GetOrderQuery, OrderDto>
{
    public ValueTask<OrderDto> Handle(GetOrderQuery query, CancellationToken ct)
    {
        // ...
    }
}

// Registration
services.AddMediator();  // source-generated registrations + handlers
```

API почти идентичен MediatR — миграция простая. Performance лучше (source-gen, no reflection).

> [!question]- **Интервью: что делать, если проект на MediatR, а 13+ стал коммерческим?**
> Это не форк — сама библиотека сменила модель (dual-license с июля 2025). Четыре опции:
> 1. Проверить Community edition — при выручке < $5M **и** привлечённом капитале < $10M MediatR 13+ бесплатен; зафиксировать порог в ADR
> 2. Остаться на 12.x (Apache 2.0 навсегда) — работает, но версия заморожена: через 2-3 года это security-долг
> 3. Мигрировать на open-source: `Mediator` (Martin Othamar) — самая близкая по API замена, source-gen, MIT; либо свой dispatcher ~50 строк
> 4. Заплатить (тиры по размеру команды, не per-developer) — если MediatR глубоко embedded, иногда дешевле миграции
>
> Главное — **принять решение явно** в ADR, не игнорировать. Это license risk и technical debt. Подробно — [[choosing-dependencies|Choosing Dependencies]].

---

## CQRS с Mediator — структура

### Vertical Slice Architecture (рекомендуется)

```
Features/
├── Orders/
│   ├── Create/
│   │   ├── CreateOrderCommand.cs
│   │   ├── CreateOrderHandler.cs
│   │   ├── CreateOrderValidator.cs
│   │   └── CreateOrderEndpoint.cs
│   ├── Get/
│   │   ├── GetOrderQuery.cs
│   │   ├── GetOrderHandler.cs
│   │   └── GetOrderEndpoint.cs
│   └── List/
│       └── ...
└── Users/
    └── ...
```

Все код одной фичи — рядом. Меняешь "Create Order" — открываешь одну папку.

### Command (write)

```csharp
public sealed record CreateOrderCommand(string Customer, decimal Total, List<OrderItemDto> Items)
    : IRequest<Result<Guid>>;

public sealed class CreateOrderHandler(
    AppDbContext db,
    ICurrentUserService user,
    IPublishEndpoint publisher,
    ILogger<CreateOrderHandler> logger)
    : IRequestHandler<CreateOrderCommand, Result<Guid>>
{
    public async Task<Result<Guid>> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Create(cmd.Customer, cmd.Total, user.UserId);
        if (!order.IsSuccess) return Result.Failure<Guid>(order.Error);

        db.Orders.Add(order.Value);
        await db.SaveChangesAsync(ct);

        await publisher.Publish(new OrderCreated(order.Value.Id), ct);
        logger.LogInformation("Order {OrderId} created", order.Value.Id);

        return Result.Success(order.Value.Id);
    }
}
```

### Query (read)

```csharp
public sealed record GetOrderQuery(Guid Id) : IRequest<OrderDto?>;

public sealed class GetOrderHandler(AppDbContext db) : IRequestHandler<GetOrderQuery, OrderDto?>
{
    public async Task<OrderDto?> Handle(GetOrderQuery query, CancellationToken ct)
    {
        return await db.Orders
            .AsNoTracking()
            .Where(o => o.Id == query.Id)
            .Select(o => new OrderDto(o.Id, o.Customer, o.Total))
            .FirstOrDefaultAsync(ct);
    }
}
```

### Endpoint (Minimal API)

```csharp
public sealed class CreateOrderEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app) =>
        app.MapPost("/api/orders", async (
            CreateOrderRequest request,
            IMediator mediator,
            CancellationToken ct) =>
        {
            var cmd = new CreateOrderCommand(request.Customer, request.Total, request.Items);
            var result = await mediator.Send(cmd, ct);

            return result.IsSuccess
                ? Results.Created($"/api/orders/{result.Value}", result.Value)
                : Results.UnprocessableEntity(result.Error);
        }).WithTags("Orders");
}

public sealed record CreateOrderRequest(string Customer, decimal Total, List<OrderItemDto> Items);
```

См. подробно в [[api-design|API Design]] — `IEndpoint` pattern.

---

## Pipeline Behaviors

Mediator pattern даёт **cross-cutting concerns** — действия, применяемые **ко всем** handlers без копирования кода. Через pipeline behaviors.

```
Request ─► [Logging] ─► [Validation] ─► [Transaction] ─► Handler ─► Response
                                                            │
                                          [Caching] ◄────────┘
```

### Validation pipeline

```csharp
public sealed class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        if (!validators.Any())
            return await next();

        var context = new ValidationContext<TRequest>(request);
        var failures = (await Task.WhenAll(validators.Select(v => v.ValidateAsync(context, ct))))
            .SelectMany(r => r.Errors)
            .Where(f => f != null)
            .ToList();

        if (failures.Count != 0)
            throw new ValidationException(failures);

        return await next();
    }
}

services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```

Все commands проходят через validation.

### Logging behavior

```csharp
public sealed class LoggingBehavior<TRequest, TResponse>(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        var sw = Stopwatch.StartNew();

        try
        {
            logger.LogInformation("Handling {RequestName}", requestName);
            var response = await next();
            logger.LogInformation("Handled {RequestName} in {Ms}ms", requestName, sw.ElapsedMilliseconds);
            return response;
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Failed {RequestName}", requestName);
            throw;
        }
    }
}
```

### Transactional behavior

```csharp
public sealed class TransactionalBehavior<TRequest, TResponse>(AppDbContext db)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICommand  // marker interface для commands
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        await using var tx = await db.Database.BeginTransactionAsync(ct);
        try
        {
            var response = await next();
            await tx.CommitAsync(ct);
            return response;
        }
        catch
        {
            await tx.RollbackAsync(ct);
            throw;
        }
    }
}
```

Commands в одной транзакции — Queries без. Разделение через marker interface (`ICommand` vs `IQuery`).

### Retry behavior

```csharp
public sealed class RetryBehavior<TRequest, TResponse>(ILogger<RetryBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRetryable  // marker
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var pipeline = new ResiliencePipelineBuilder()
            .AddRetry(new RetryStrategyOptions
            {
                MaxRetryAttempts = 3,
                BackoffType = DelayBackoffType.Exponential,
                ShouldHandle = new PredicateBuilder().Handle<DbUpdateConcurrencyException>(),
            })
            .Build();

        return await pipeline.ExecuteAsync(async _ => await next(), ct);
    }
}
```

См. [[resilience|Resilience]] — Polly v8 deep.

### Caching behavior

```csharp
public sealed class CachingBehavior<TRequest, TResponse>(HybridCache cache)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : ICacheable
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var cacheable = (ICacheable)request;
        return await cache.GetOrCreateAsync(
            cacheable.CacheKey,
            async _ => await next(),
            new HybridCacheEntryOptions { Expiration = cacheable.CacheTime },
            tags: cacheable.Tags,
            cancellationToken: ct);
    }
}

public interface ICacheable
{
    string CacheKey { get; }
    TimeSpan CacheTime { get; }
    string[] Tags { get; }
}

public sealed record GetOrderQuery(Guid Id) : IRequest<OrderDto?>, ICacheable
{
    public string CacheKey => $"order:{Id}";
    public TimeSpan CacheTime => TimeSpan.FromMinutes(5);
    public string[] Tags => ["orders", $"order:{Id}"];
}
```

### Order matters

```csharp
// Регистрация в порядке выполнения (outer → inner)
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));      // outer
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(CachingBehavior<,>));
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(TransactionalBehavior<,>));
services.AddScoped(typeof(IPipelineBehavior<,>), typeof(RetryBehavior<,>));        // inner
```

Логирование снаружи всего → видим всё что внутри.
Validation до transaction → не открываем tx если invalid.
Retry внутри transaction может быть проблемой — на retry tx уже откачена → выбор зависит от семантики.

> [!question]- **Интервью: какие cross-cutting concerns стоит вынести в pipeline behavior?**
> Логирование, валидация, transaction, retry, caching, performance metrics, authorization. Главный критерий: **применяется ко многим handlers одинаково**.
> Что НЕ нужно класть: бизнес-логика конкретного handler'а, маппинг request → entity (это работа handler'а).

---

## Notification pattern (events)

Один request — один handler. Один event — N handlers.

```csharp
public sealed record OrderCreated(Guid OrderId) : INotification;

public class SendEmailHandler : INotificationHandler<OrderCreated>
{
    public async Task Handle(OrderCreated evt, CancellationToken ct)
    {
        // send email
    }
}

public class UpdateInventoryHandler : INotificationHandler<OrderCreated>
{
    public async Task Handle(OrderCreated evt, CancellationToken ct)
    {
        // decrease stock
    }
}

// Publish
await mediator.Publish(new OrderCreated(orderId), ct);
// Both handlers run
```

### Strategy: parallel или sequential

```csharp
// Sequential (default) — handlers вызываются по очереди
await mediator.Publish(notification, ct);

// Parallel
public sealed class ParallelNoPublisher : INotificationPublisher
{
    public Task Publish(IEnumerable<NotificationHandlerExecutor> handlers, INotification notification, CancellationToken ct) =>
        Task.WhenAll(handlers.Select(h => h.HandlerCallback(notification, ct)));
}

services.AddMediatR(cfg =>
{
    cfg.NotificationPublisher = new ParallelNoPublisher();
});
```

### Pitfall — exception в одном handler ломает все

В default sequential — exception у первого handler stops остальных.
**Решение:** правильно обрабатывать ошибки в каждом handler, или использовать **distributed events через message bus** (см. [[messaging|Messaging]]).

Notification pattern в Mediator — для **in-process events**. Для durable events между сервисами — RabbitMQ/MassTransit.

---

## CQRS + Read models

Light CQRS: Commands пишут в normalized DB, Queries читают из той же DB.

Medium CQRS: Commands пишут в write DB, Queries читают из **денормализованной** read model.

```
Command ─► Domain ─► Write DB
              │
              ▼
        OrderCreated event
              │
              ▼
        Projection handler ─► Read DB (денорм.)
                                  │
                                  ▼
Query ◄─────────────────────  Read DB
```

### Когда стоит делать отдельную read model

- Read и write имеют разные модели (write — normalized aggregate, read — flat для list)
- Read RPS >> write RPS (можно scale read DB independently)
- Сложные queries которые делать на write side дорого

См. [[distributed-systems|Distributed Systems / CQRS read models]] — детально.

---

## Альтернатива MediatR — FastEndpoints

Не mediator pattern, а **vertical-slice endpoint framework**. Не нужен mediator — endpoint **сам** handler.

```bash
dotnet add package FastEndpoints
```

```csharp
public sealed class CreateOrderRequest
{
    public string Customer { get; init; } = "";
    public decimal Total { get; init; }
}

public sealed class CreateOrderResponse
{
    public Guid OrderId { get; init; }
}

public sealed class CreateOrderEndpoint : Endpoint<CreateOrderRequest, CreateOrderResponse>
{
    private readonly AppDbContext _db;

    public CreateOrderEndpoint(AppDbContext db) => _db = db;

    public override void Configure()
    {
        Post("/api/orders");
        AllowAnonymous();
        Description(b => b.Produces(201));
    }

    public override async Task HandleAsync(CreateOrderRequest req, CancellationToken ct)
    {
        var order = new Order { /* ... */ };
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);

        await SendCreatedAtAsync<GetOrderEndpoint>(new { OrderId = order.Id },
            new CreateOrderResponse { OrderId = order.Id }, cancellation: ct);
    }
}

// Program.cs
builder.Services.AddFastEndpoints();
app.UseFastEndpoints();
```

### Когда FastEndpoints vs Mediator

| FastEndpoints | Mediator |
|---------------|----------|
| API-first проекты | Команды могут вызываться из разных мест |
| Без отдельных pipeline behaviors | Нужны cross-cutting concerns ко всем |
| Хочется faster startup, less abstraction | Нужна testability handler'а в isolation |
| API endpoints — единственный entry point | Background jobs тоже dispatch'ат commands |
| Source-gen, AOT-friendly | Reflection-heavy (старые версии) |

В NexusAI / [anonymized] — FastEndpoints для API + Mediator для internal cross-cutting (если нужно). Можно комбинировать.

---

## Common pitfalls

### 1. God Handler

```csharp
public class CreateOrderHandler
{
    public async Task<Result<Guid>> Handle(...)
    {
        // 200 строк: validation, mapping, business rules, save, publish, email...
    }
}
```
Handler — это координатор. Логика — в domain (`Order.Create`). Side effects — отдельные методы.

### 2. Handler вызывает другой handler через mediator

```csharp
public async Task Handle(CreateOrderCommand cmd, ...)
{
    await _db.Orders.AddAsync(...);
    await mediator.Send(new SendEmailCommand(...));  // ❌ tight coupling, тяжело тестировать
}
```
**Решение:** publish notification вместо command. Email handler сам подписан на event.

### 3. Bloated request DTO

```csharp
public record CreateOrderCommand(string Customer, decimal Total, ..., User CurrentUser, IMediator Mediator);
```
Request — это **данные**. `User`, `Mediator` инжектятся в handler через DI.

### 4. Validation в handler'е
Handler сам делает `if (request.Total < 0) return Failure(...)`. Лучше — отдельный `IValidator<CreateOrderCommand>` + ValidationBehavior.

### 5. Nested mediator calls
A → mediator.Send(B) → mediator.Send(C) → mediator.Send(D). Стек становится огромным, observability сложная.
**Решение:** один command — одна операция. Если зависимый flow — saga через message bus.

### 6. Public domain types в request

```csharp
public record CreateOrderCommand(Order Order) : IRequest<...>;  // ❌
```
Domain — за boundary. Request — DTO. Маппинг внутри handler'а.

### 7. Никаких pipeline behaviors
Validation/logging копируется во всех handlers. Нет смысла использовать mediator без pipeline.

### 8. Forget AsNoTracking в queries
Query handler меняет state? Нет — read-only. AsNoTracking обязателен.

### 9. Mediator для тривиального CRUD
3 endpoint'а, делают `db.X.Add(); db.SaveChanges()`. Mediator — overkill, прямой Endpoint Handler.

### 10. Не учитывать commercial license MediatR
"Использую MediatR в production не задумываясь" → license risk. Документируй решение в ADR (платим / migrate / freeze).

---

## Production checklist

- [ ] CQRS Light minimum: разделение Command/Query классов
- [ ] Vertical Slice — feature-based папки
- [ ] Pipeline behaviors: Logging, Validation, Transaction
- [ ] Validation через FluentValidation (или DataAnnotations)
- [ ] Result pattern для return values handlers
- [ ] Marker interfaces (`ICommand`, `IQuery`, `ICacheable`, `IRetryable`)
- [ ] Notifications для in-process events, MassTransit для distributed
- [ ] AsNoTracking в Query handlers
- [ ] Async/CancellationToken во всех handlers
- [ ] License decision документирован (MediatR commercial vs alternative) в ADR
- [ ] Tests на handlers с mock'ами зависимостей
- [ ] Architecture tests (NetArchTest) — handlers не зависят от UI/Web слоя

---

## См. также

- [[architecture-patterns|Architecture Patterns]] — Clean, VSA, modular monolith
- [[ddd|DDD на практике]] — domain logic в Aggregate, не в handler
- [[distributed-systems|Distributed Systems]] — saga, outbox, event-driven CQRS
- [[result-pattern|Result Pattern]] — return values handlers
- [[caching|Caching]] — caching pipeline behavior
- [[resilience|Resilience]] — retry pipeline behavior
- [[testing|Testing]] — тесты handlers с моками

## Reading list

- **Vertical Slice Architecture** — Jimmy Bogard talks
- **Mediator (Martin Othamar)** — github.com/martinothamar/Mediator
- **MediatR docs** — github.com/LuckyPennySoftware/MediatR (даже если migration — паттерны полезны)
- **FastEndpoints docs** — fast-endpoints.com
- **Brighter** — brighter.readthedocs.io
- **Wolverine** — wolverine.netlify.app (full microservices framework)
- **Milan Jovanović — CQRS series** — milanjovanovic.tech (особенно про Light CQRS)
- **Event Sourcing in .NET** — github.com/oskardudycz/EventSourcing.NetCore
