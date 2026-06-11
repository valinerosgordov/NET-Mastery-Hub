# Snippets — Result<T> Pattern

## Базовая реализация Result<T>

```csharp
namespace MyApp.Domain.Common;

public sealed class Result<T>
{
    public T? Value { get; }
    public Error? Error { get; }
    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;

    private Result(T value) { Value = value; IsSuccess = true; }
    private Result(Error error) { Error = error; IsSuccess = false; }

    public static Result<T> Ok(T value) => new(value);
    public static Result<T> Fail(Error error) => new(error);

    public TResult Match<TResult>(
        Func<T, TResult> onSuccess,
        Func<Error, TResult> onFailure) =>
        IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

public static class Result
{
    public static Result<T> Ok<T>(T value) => Result<T>.Ok(value);
    public static Result<T> Fail<T>(Error error) => Result<T>.Fail(error);
}

public sealed record Error(string Code, string Message)
{
    public static readonly Error None = new(string.Empty, string.Empty);
}

// Типовые ошибки
public static class Errors
{
    public static Error NotFound(string resource, object id) =>
        new("NotFound", $"{resource} with id '{id}' was not found");

    public static Error Validation(string field, string message) =>
        new("Validation", $"{field}: {message}");

    public static Error Conflict(string message) =>
        new("Conflict", message);

    public static Error Unauthorized() =>
        new("Unauthorized", "Access denied");
}
```

## Использование в Handler

```csharp
// В handler
var result = await service.ProcessAsync(command, ct);

return result.Match(
    onSuccess: id => Result.Ok(id),
    onFailure: error => Result.Fail<Guid>(error)
);
```

## Маппинг Result → HTTP (Minimal API)

```csharp
app.MapPost("/api/orders", async (CreateOrderCommand cmd, ISender sender, CancellationToken ct) =>
{
    var result = await sender.Send(cmd, ct);

    return result.Match(
        onSuccess: id => Results.Created($"/api/orders/{id}", new { Id = id }),
        onFailure: error => error.Code switch
        {
            "NotFound"     => Results.NotFound(new ProblemDetails { Detail = error.Message }),
            "Validation"   => Results.UnprocessableEntity(new ProblemDetails { Detail = error.Message }),
            "Conflict"     => Results.Conflict(new ProblemDetails { Detail = error.Message }),
            "Unauthorized" => Results.Forbid(),
            _              => Results.Problem(error.Message)
        }
    );
});
```

## Chaining (Railway-oriented)

```csharp
// Последовательная цепочка — если любой шаг упал, дальше не идём
public async Task<Result<OrderConfirmation>> ProcessOrderAsync(
    CreateOrderCommand cmd, CancellationToken ct)
{
    var customerResult = await customerService.GetAsync(cmd.CustomerId, ct);
    if (customerResult.IsFailure) return Result.Fail<OrderConfirmation>(customerResult.Error!);

    var inventoryResult = await inventoryService.ReserveAsync(cmd.Items, ct);
    if (inventoryResult.IsFailure) return Result.Fail<OrderConfirmation>(inventoryResult.Error!);

    var order = Order.Create(customerResult.Value!, inventoryResult.Value!);
    await repo.AddAsync(order, ct);
    await uow.SaveChangesAsync(ct);

    return Result.Ok(new OrderConfirmation(order.Id, order.EstimatedDelivery));
}
```

---

## См. также

- [[Result Pattern + CQRS|Result Pattern + CQRS]]
- [[MediatR Handlers|MediatR Handlers]]
