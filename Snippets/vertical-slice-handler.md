---
tags: [snippets, vertical-slices, minimal-api, cqrs]
level: All
date: 2026-08-02
---

# Snippets — Vertical Slice без MediatR

> Copy-paste слайс без mediator-библиотеки: feature-класс с Command/Response records и Handler (primary constructor DI), Minimal API endpoint с `TypedResults` + ProblemDetails, валидация endpoint-фильтром, `Result<T>`-flow. Второй вариант — минимальный свой `IDispatcher` (~30 строк) для тех, кому нужна единая точка Send. Почему не MediatR 13+ — [[choosing-dependencies|Choosing Dependencies]].

## 1. Слайс целиком: endpoint → handler напрямую

Endpoint вызывает handler напрямую через DI — без `IRequest`, `ISender` и рефлексии. Стектрейс читается, «go to definition» ведёт в handler, а не в библиотеку, и лицензионный вопрос MediatR 13+ просто не возникает.

```csharp
namespace MyApp.Features.Orders.CreateOrder;

// Один static-класс = один слайс: контракт, handler и endpoint в одном файле
public static class CreateOrder
{
    public sealed record Command(Guid CustomerId, List<OrderItemDto> Items);

    public sealed record Response(Guid OrderId);

    // Обычный класс в DI — никакого IRequestHandler
    public sealed class Handler(
        IOrderRepository repo,
        IUnitOfWork uow,
        ILogger<Handler> logger)
    {
        public async Task<Result<Response>> Handle(Command command, CancellationToken ct)
        {
            var customer = await repo.GetCustomerByIdAsync(command.CustomerId, ct);
            if (customer is null)
                return Result.Fail<Response>(Errors.NotFound("Customer", command.CustomerId));

            var order = Order.Create(customer.Id, command.Items);

            await repo.AddAsync(order, ct);
            await uow.SaveChangesAsync(ct);

            logger.LogInformation("Order {OrderId} created", order.Id);
            return Result.Ok(new Response(order.Id));
        }
    }

    public static void MapEndpoint(IEndpointRouteBuilder app) =>
        app.MapPost("/api/orders", async (
                Command command,
                Handler handler,
                CancellationToken ct) =>
            {
                var result = await handler.Handle(command, ct);

                return result.Match<IResult>(
                    onSuccess: r => TypedResults.Created($"/api/orders/{r.OrderId}", r),
                    onFailure: error => error.Code switch
                    {
                        "NotFound"   => TypedResults.NotFound(new ProblemDetails { Detail = error.Message }),
                        "Validation" => TypedResults.UnprocessableEntity(new ProblemDetails { Detail = error.Message }),
                        "Conflict"   => TypedResults.Conflict(new ProblemDetails { Detail = error.Message }),
                        _            => TypedResults.Problem(error.Message)
                    });
            })
            .AddEndpointFilter<ValidationFilter<Command>>()
            .WithTags("Orders");
}
```

Реализация `Result<T>` и `Errors` — в [[result-pattern|Result Pattern]].

### 1.1. Регистрация (Program.cs)

```csharp
builder.Services.AddScoped<CreateOrder.Handler>();
builder.Services.AddValidatorsFromAssembly(typeof(CreateOrder).Assembly);

var app = builder.Build();
CreateOrder.MapEndpoint(app);
```

При десятках слайсов регистрацию хэндлеров собирают сканированием (Scrutor), но начинать стоит с явных строк — сразу видно, какие слайсы живы.

## 2. Валидация — endpoint-фильтр вместо PipelineBehavior

FluentValidation остаётся свободной (Apache 2.0); меняется только точка подключения: вместо `IPipelineBehavior` — `IEndpointFilter`, срабатывает после binding, до вызова handler'а.

```csharp
namespace MyApp.Api.Filters;

public sealed class ValidationFilter<TRequest>(IValidator<TRequest>? validator = null)
    : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context,
        EndpointFilterDelegate next)
    {
        // Валидатор не зарегистрирован — пропускаем
        if (validator is null)
            return await next(context);

        if (context.Arguments.OfType<TRequest>().FirstOrDefault() is not { } request)
            return await next(context);

        var validation = await validator.ValidateAsync(
            request, context.HttpContext.RequestAborted);

        if (!validation.IsValid)
            return TypedResults.ValidationProblem(validation.ToDictionary());

        return await next(context);
    }
}

// Валидатор — обычный FluentValidation, живёт рядом со слайсом
public sealed class CreateOrderCommandValidator : AbstractValidator<CreateOrder.Command>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty();
    }
}
```

## 3. Вариант 2: минимальный свой IDispatcher

Когда нужна единая точка `Send` (декораторы, логирование, метрики) или декомпозиция «команда объявляет свой результат» — mediator пишется руками за ~30 строк:

```csharp
namespace MyApp.Application.Dispatching;

public interface ICommand<TResult>;

public interface ICommandHandler<in TCommand, TResult>
    where TCommand : ICommand<TResult>
{
    Task<TResult> Handle(TCommand command, CancellationToken ct);
}

public interface IDispatcher
{
    Task<TResult> Send<TResult>(ICommand<TResult> command, CancellationToken ct);
}

public sealed class Dispatcher(IServiceProvider services) : IDispatcher
{
    public async Task<TResult> Send<TResult>(ICommand<TResult> command, CancellationToken ct)
    {
        var handlerType = typeof(ICommandHandler<,>)
            .MakeGenericType(command.GetType(), typeof(TResult));

        var handler = services.GetRequiredService(handlerType);

        // Reflection-вызов: ок для типового API; на hot path — кэш MethodInfo
        // (ConcurrentDictionary) или Mediator.SourceGenerator
        var handle = handlerType.GetMethod("Handle")!;
        return await (Task<TResult>)handle.Invoke(handler, [command, ct])!;
    }
}
```

### 3.1. Использование

```csharp
// Command объявляет свой результат — семантика та же, что у IRequest<T> в MediatR
public sealed record CreateOrderCommand(Guid CustomerId)
    : ICommand<Result<Guid>>;

public sealed class CreateOrderHandler(IOrderRepository repo, IUnitOfWork uow)
    : ICommandHandler<CreateOrderCommand, Result<Guid>>
{
    public async Task<Result<Guid>> Handle(CreateOrderCommand command, CancellationToken ct)
    {
        var order = Order.Create(command.CustomerId);
        await repo.AddAsync(order, ct);
        await uow.SaveChangesAsync(ct);
        return Result.Ok(order.Id);
    }
}
```

```csharp
// Program.cs + endpoint
builder.Services.AddScoped<IDispatcher, Dispatcher>();
builder.Services.AddScoped<
    ICommandHandler<CreateOrderCommand, Result<Guid>>,
    CreateOrderHandler>();

app.MapPost("/api/orders", async (
    CreateOrderCommand command,
    IDispatcher dispatcher,
    CancellationToken ct) =>
{
    var result = await dispatcher.Send(command, ct);

    return result.Match<IResult>(
        onSuccess: id => TypedResults.Created($"/api/orders/{id}", new { Id = id }),
        onFailure: error => TypedResults.Problem(error.Message));
});
```

## 4. Когда что брать

| Ситуация | Решение |
|----------|---------|
| Обычный CRUD / CQRS-декомпозиция | Прямой вызов handler'а (раздел 1) — дефолт |
| Единая точка `Send`, декораторы, метрики | Свой `IDispatcher` (раздел 3) |
| Нужна `IRequest`-семантика без рефлексии | Mediator.SourceGenerator (martinothamar) — drop-in, source-gen |
| Durable messaging, saga, outbox | Wolverine (MIT) — mediator + messaging одним инструментом |
| Легаси на MediatR | Остаться на 12.x (свободная лицензия навсегда) или проверить условия community edition 13+ |

---

## См. также

- [[result-pattern|Result Pattern]] — `Result<T>`, `Error`, маппинг в HTTP
- [[mediatr-handlers|MediatR Handlers]] — тот же слайс на MediatR 12.x
- [[crud-example|CRUD Example]] — полный CRUD от endpoint до EF Core в этом же стиле
- [[choosing-dependencies|Choosing Dependencies]] — лицензии и критерии выбора зависимостей
