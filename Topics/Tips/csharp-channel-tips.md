# C# Tips & Tricks

Полезные советы и паттерны из сообщества. Источник: @csharp_1001_notes и практика.

---

## .NET 10 & C# 14 — Ключевые фичи

### C# 14
- **Extension Members** — расширения не только методы, но и свойства/индексаторы
- **Null-Conditional Assignment** — `obj?.Prop = value;`
- **`field` keyword** — доступ к backing field в auto-property
- **Lambda parameter modifiers** — `ref`, `in`, `out` в лямбдах
- **Partial constructors/events** — partial теперь для конструкторов и событий

### ASP.NET Core 10
- Minimal APIs: встроенная валидация
- JSON Patch поддержка
- SSE (Server-Sent Events) из коробки
- OpenAPI 3.1

### EF Core 10
- Optional Complex Types
- JSON в Complex Types
- `LeftJoin` / `RightJoin` LINQ операторы
- Named Query Filters
- Улучшенный `ExecuteUpdate`

---

## Feature Flags

Runtime-контроль поведения без редеплоя. Включение/выключение фич по условиям.

```csharp
// NuGet: Microsoft.FeatureManagement.AspNetCore
builder.Services.AddFeatureManagement();

// appsettings.json
{
  "FeatureManagement": {
    "NewDashboard": true,
    "BetaSearch": {
      "EnabledFor": [
        { "Name": "Percentage", "Parameters": { "Value": 30 } }
      ]
    }
  }
}

// Использование
app.MapGet("/dashboard", async (IFeatureManager fm) =>
{
    if (await fm.IsEnabledAsync("NewDashboard"))
        return Results.Ok("New dashboard");
    return Results.Ok("Legacy dashboard");
});
```

**Инструменты:** Microsoft.FeatureManagement, Azure App Configuration, LaunchDarkly, Unleash.

**Когда использовать:**
- Progressive rollout (постепенный выкат)
- A/B тестирование
- Kill switch для проблемных фич
- User targeting (по юзеру/группе/проценту)

---

## Minimal APIs — Auto-Registration Endpoints

Избавляет от ручной регистрации каждого endpoint.

```csharp
// Интерфейс
public interface IEndpoint
{
    void Map(IEndpointRouteBuilder app);
}

// Пример endpoint
public sealed class GetOrderEndpoint : IEndpoint
{
    public void Map(IEndpointRouteBuilder app)
    {
        app.MapGet("/api/orders/{id:guid}", async (
            Guid id,
            ISender sender,
            CancellationToken ct) =>
        {
            var result = await sender.Send(new GetOrderByIdQuery { Id = id }, ct);
            return result.IsSuccess
                ? Results.Ok(result.Value)
                : Results.NotFound(result.Error);
        });
    }
}

// Регистрация в Program.cs
public static IServiceCollection AddEndpoints(this IServiceCollection services)
{
    var endpointTypes = typeof(Program).Assembly
        .GetTypes()
        .Where(t => t is { IsAbstract: false, IsInterface: false }
                     && t.IsAssignableTo(typeof(IEndpoint)));

    foreach (var type in endpointTypes)
        services.AddSingleton(typeof(IEndpoint), type);

    return services;
}

public static IApplicationBuilder MapEndpoints(this WebApplication app)
{
    var endpoints = app.Services.GetRequiredService<IEnumerable<IEndpoint>>();
    foreach (var endpoint in endpoints)
        endpoint.Map(app);
    return app;
}

// Program.cs
builder.Services.AddEndpoints();
// ...
app.MapEndpoints();
```

---

## FluentValidation vs Data Annotations

| Критерий | Data Annotations | FluentValidation |
|----------|-----------------|------------------|
| DI | Нет | Да |
| Динамические правила | Нет | Да |
| Проверка по БД | Нет | Да |
| Сложная логика | Ограничено | Полная |
| Конфигурация | Атрибуты на модели | Отдельный класс |

**Когда FluentValidation лучше:**

```csharp
public sealed class CreateOrderCommandValidator
    : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator(
        IOrderRepository orderRepo,    // DI
        IOptions<OrderSettings> opts)  // DI
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty()
            .MustAsync(async (id, ct) =>
                await orderRepo.ExistsAsync(id, ct))  // Проверка по БД
            .WithMessage("Customer not found");

        RuleFor(x => x.Items)
            .NotEmpty()
            .Must(items => items.Count <= opts.Value.MaxItems)
            .WithMessage($"Max {opts.Value.MaxItems} items per order");
    }
}
```

---

## Password Hashing — Миграция алгоритмов

Стратегия обновления хеширования без сброса паролей всех пользователей.

```csharp
public interface IPasswordHasher
{
    string Hash(string password);
    bool Verify(string password, string hash);
    bool NeedsRehash(string hash);
}

// При логине
public async Task<Result<AuthToken>> LoginAsync(LoginCommand cmd, CancellationToken ct)
{
    var user = await userRepo.GetByEmailAsync(cmd.Email, ct);
    if (user is null)
        return Result.Fail<AuthToken>(new NotFoundError("User not found"));

    if (!passwordHasher.Verify(cmd.Password, user.PasswordHash))
        return Result.Fail<AuthToken>(new AuthError("Invalid credentials"));

    // Auto-upgrade хеша при успешном логине
    if (passwordHasher.NeedsRehash(user.PasswordHash))
    {
        user.PasswordHash = passwordHasher.Hash(cmd.Password);
        await userRepo.UpdateAsync(user, ct);
    }

    return Result.Ok(tokenService.GenerateToken(user));
}
```

**Принцип:** поддерживаем несколько алгоритмов одновременно, обновляем на лету при успешном логине.

---

## Load Balancing

Распределение трафика между серверами для масштабируемости и отказоустойчивости.

```nginx
# Nginx upstream config
upstream backend {
    server app1:5000;
    server app2:5000;
    server app3:5000;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;
    }
}
```

**Алгоритмы:**
- **Round Robin** — по очереди (default)
- **Least Connections** — на сервер с меньшим кол-вом соединений
- **IP Hash** — sticky sessions по IP
- **Weighted** — приоритет серверам с большими ресурсами

---

## Event Queue в ASP.NET Core

Паттерн для decoupled компонентов через встроенные механизмы.

```csharp
// Простая in-memory очередь через Channel
public sealed class EventQueue<T>
{
    private readonly Channel<T> _channel = Channel.CreateUnbounded<T>();

    public async ValueTask PublishAsync(T @event, CancellationToken ct = default)
        => await _channel.Writer.WriteAsync(@event, ct);

    public IAsyncEnumerable<T> ReadAllAsync(CancellationToken ct = default)
        => _channel.Reader.ReadAllAsync(ct);
}

// Background consumer
public sealed class EventConsumer<T>(
    EventQueue<T> queue,
    IServiceScopeFactory scopeFactory,
    ILogger<EventConsumer<T>> logger)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var @event in queue.ReadAllAsync(ct))
        {
            await using var scope = scopeFactory.CreateAsyncScope();
            var handler = scope.ServiceProvider.GetRequiredService<IEventHandler<T>>();
            try
            {
                await handler.HandleAsync(@event, ct);
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Failed to handle event {EventType}", typeof(T).Name);
            }
        }
    }
}

// DI
builder.Services.AddSingleton<EventQueue<OrderCreatedEvent>>();
builder.Services.AddHostedService<EventConsumer<OrderCreatedEvent>>();
builder.Services.AddScoped<IEventHandler<OrderCreatedEvent>, OrderCreatedHandler>();
```

---

## См. также

- [[dotnet-knowledge-base|.NET Knowledge Base]]
- [[Topics/Snippets/snippet-mediatr-handlers|MediatR Handlers]]
- [[Topics/ResultPattern/result-pattern-cqrs|Result Pattern + CQRS]]
- [[Topics/CodeQuality/code-quality-best-practices|Code Quality]]
- [[Topics/Messaging/rabbitmq-masstransit|RabbitMQ + MassTransit]]
