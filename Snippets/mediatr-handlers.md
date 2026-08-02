---
tags: [mediatr, cqrs, handlers, result-pattern, pipeline-behavior, validation]
level: Middle to Senior
---

# Snippets — MediatR Handlers

> CQRS-обвязка на MediatR: Command/Query handlers, возвращающие `Result<T>`, FluentValidation validator и `IPipelineBehavior<TRequest, TResponse>` для валидации и логирования, плюс регистрация в Program.cs.

## Command Handler (с Result<T>)

```csharp
namespace MyApp.Features.Orders.CreateOrder;

public sealed record CreateOrderCommand : IRequest<Result<Guid>>
{
    public required Guid CustomerId { get; init; }
    public required List<OrderItemDto> Items { get; init; }
}

public sealed class CreateOrderCommandHandler(
    IOrderRepository orderRepo,
    IUnitOfWork uow,
    ILogger<CreateOrderCommandHandler> logger)
    : IRequestHandler<CreateOrderCommand, Result<Guid>>
{
    public async Task<Result<Guid>> Handle(
        CreateOrderCommand command,
        CancellationToken cancellationToken)
    {
        var customer = await orderRepo
            .GetCustomerByIdAsync(command.CustomerId, cancellationToken)
            .ConfigureAwait(false);

        if (customer is null)
            return Result.Fail<Guid>(new NotFoundError($"Customer {command.CustomerId} not found"));

        var order = Order.Create(customer.Id, command.Items);

        await orderRepo.AddAsync(order, cancellationToken).ConfigureAwait(false);
        await uow.SaveChangesAsync(cancellationToken).ConfigureAwait(false);

        logger.LogInformation("Order {OrderId} created for customer {CustomerId}",
            order.Id, customer.Id);

        return Result.Ok(order.Id);
    }
}
```

## Query Handler

```csharp
namespace MyApp.Features.Orders.GetOrder;

public sealed record GetOrderByIdQuery(Guid OrderId) : IRequest<Result<OrderDto>>;

public sealed class GetOrderByIdQueryHandler(IOrderRepository repo)
    : IRequestHandler<GetOrderByIdQuery, Result<OrderDto>>
{
    public async Task<Result<OrderDto>> Handle(
        GetOrderByIdQuery query,
        CancellationToken cancellationToken)
    {
        var order = await repo
            .GetByIdAsync(query.OrderId, cancellationToken)
            .ConfigureAwait(false);

        if (order is null)
            return Result.Fail<OrderDto>(new NotFoundError($"Order {query.OrderId} not found"));

        return Result.Ok(order.ToDto());
    }
}
```

## Validator (FluentValidation)

```csharp
namespace MyApp.Features.Orders.CreateOrder;

public sealed class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId)
            .NotEmpty().WithMessage("CustomerId is required");

        RuleFor(x => x.Items)
            .NotEmpty().WithMessage("Order must have at least one item")
            .ForEach(item => item
                .ChildRules(i =>
                {
                    i.RuleFor(x => x.ProductId).NotEmpty();
                    i.RuleFor(x => x.Quantity).GreaterThan(0);
                }));
    }
}
```

## Pipeline Behaviour (Validation + Logging)

```csharp
namespace MyApp.Application.Behaviours;

public sealed class ValidationBehaviour<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken cancellationToken)
    {
        if (!validators.Any())
            return await next().ConfigureAwait(false);

        var context = new ValidationContext<TRequest>(request);
        var failures = validators
            .Select(v => v.Validate(context))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count > 0)
            throw new ValidationException(failures);

        return await next().ConfigureAwait(false);
    }
}
```

## MediatR Registration (Program.cs)

```csharp
builder.Services.AddMediatR(cfg =>
{
    cfg.RegisterServicesFromAssembly(typeof(CreateOrderCommand).Assembly);
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehaviour<,>));
    cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehaviour<,>));
});

builder.Services.AddValidatorsFromAssembly(typeof(CreateOrderCommandValidator).Assembly);
```

---

## См. также

- [[result-pattern|Result Pattern + CQRS]]
