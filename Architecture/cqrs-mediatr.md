---
tags: [cqrs, mediatr, result-pattern, vertical-slices]
level: Senior
---

# Result/Option, MediatR и CQRS

## Что это, зачем и когда

### Что такое Result Pattern?
**Возврат ошибки как значения** вместо бросания исключений. Метод возвращает `Result<T>` — либо успех с данными, либо ошибка с описанием.

**Аналогия:** Вместо того чтобы кричать «ПОЖАР!» (exception), ты спокойно передаёшь записку: «не получилось, потому что...» (Result с Error).

### Что такое CQRS?
**Command Query Responsibility Segregation** — разделение операций чтения и записи. Запросы (Query) и команды (Command) идут через разные модели/пути.

**Аналогия:** В банке одно окно для «узнать баланс» (Query), другое — для «перевести деньги» (Command). Каждое оптимизировано под свою задачу.

### Что такое MediatR?
**Посредник** — вместо прямого вызова сервиса, отправляешь запрос через MediatR, и он находит нужный обработчик. Decoupling между отправителем и получателем.

### Зачем?

| Подход | Проблема без него | Что даёт |
|--------|-------------------|----------|
| **Result Pattern** | Exception для «email занят» — тяжело, неудобно, стектрейс лишний | Явные ошибки, типобезопасность, функциональная цепочка |
| **CQRS** | Один толстый сервис и для чтения, и для записи | Read-модель оптимизирована (AsNoTracking, View), Write-модель — с валидацией и бизнес-логикой |
| **MediatR** | Endpoint → Service → Repository — жёсткая связка | Endpoint → MediatR → Handler. Добавляешь логирование/валидацию через pipeline behaviors |

### Когда использовать?

| Паттерн | Нужен когда | НЕ нужен когда |
|---------|-------------|----------------|
| **Result** | Бизнес-ошибки (валидация, «не найден», «нет доступа») | Инфраструктурные сбои (БД упала, сеть) — для них исключения |
| **CQRS** | Разная нагрузка на чтение/запись, сложный домен | Простой CRUD, одна модель для всего достаточна |
| **MediatR** | Нужны cross-cutting concerns (логирование, валидация, кеш) через pipeline | Простое приложение, 5 endpoints — лишняя абстракция |

---

## Result\<T\> — ошибки как значения

Вместо исключений для ожидаемых ошибок (не найден, валидация, бизнес-правило) — возвращаем Result. Railway Oriented Programming: цепочка операций, каждая возвращает Result. При ошибке — цепочка прерывается.

### Минимальная реализация

```csharp
public sealed class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public Error? Error { get; }

    private Result(bool isSuccess, T? value, Error? error)
        => (IsSuccess, Value, Error) = (isSuccess, value, error);

    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(Error error) => new(false, default, error);

    // Функциональные методы
    public Result<TOut> Map<TOut>(Func<T, TOut> mapper)
        => IsSuccess ? Result<TOut>.Ok(mapper(Value!)) : Result<TOut>.Fail(Error!);

    public async Task<Result<TOut>> BindAsync<TOut>(Func<T, Task<Result<TOut>>> func)
        => IsSuccess ? await func(Value!) : Result<TOut>.Fail(Error!);

    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<Error, TOut> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

public record Error(string Code, string Message)
{
    public static Error NotFound(string entity, object id)
        => new("not_found", $"{entity} with ID {id} not found");
    public static Error Validation(string message)
        => new("validation", message);
    public static Error Conflict(string message)
        => new("conflict", message);
}
```

### Использование в контроллере

```csharp
app.MapGet("/orders/{id}", async (Guid id, IMediator mediator) =>
{
    var result = await mediator.Send(new GetOrderQuery(id));

    return result.Match(
        order => Results.Ok(order),
        error => error.Code switch
        {
            "not_found" => Results.NotFound(error.Message),
            "validation" => Results.BadRequest(error.Message),
            _ => Results.Problem(error.Message)
        });
});
```

### Библиотеки

- **FluentResults** — `Result<T>`, цепочки, множественные ошибки
- **OneOf** — discriminated unions (`OneOf<Success, NotFound, Error>`)
- **ErrorOr** — `ErrorOr<T>` с typed errors
- **Свой тип** — минимум кода, полный контроль

**Нюанс:** Result для ожидаемых ошибок (бизнес-логика, валидация). Исключения — для неожиданных (null reference, connection lost, out of memory).

---

## Option\<T\> — альтернатива null

Значение может отсутствовать. Явный контракт — вызывающий обязан обработать отсутствие.

```csharp
public readonly struct Option<T>
{
    private readonly T? _value;
    public bool HasValue { get; }

    private Option(T value) => (_value, HasValue) = (value, true);

    public static Option<T> Some(T value) => new(value);
    public static Option<T> None => default;

    public TOut Match<TOut>(Func<T, TOut> onSome, Func<TOut> onNone)
        => HasValue ? onSome(_value!) : onNone();

    public Option<TOut> Map<TOut>(Func<T, TOut> mapper)
        => HasValue ? Option<TOut>.Some(mapper(_value!)) : Option<TOut>.None;
}

// Использование
Option<User> user = await repo.FindByEmailAsync(email);
string greeting = user.Match(
    u => $"Привет, {u.Name}!",
    () => "Пользователь не найден");
```

---

## MediatR — Mediator Pattern

Декомпозиция: один handler на один use case. Контроллер не зависит от сервисов напрямую.

### Настройка

```csharp
services.AddMediatR(cfg => cfg.RegisterServicesFromAssembly(typeof(Program).Assembly));
```

### Command (изменяет состояние)

```csharp
public record CreateOrderCommand(string CustomerName, decimal Total) : IRequest<Result<Guid>>;

public sealed class CreateOrderHandler(
    AppDbContext db,
    ILogger<CreateOrderHandler> logger)
    : IRequestHandler<CreateOrderCommand, Result<Guid>>
{
    public async Task<Result<Guid>> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        if (string.IsNullOrWhiteSpace(request.CustomerName))
            return Result<Guid>.Fail(Error.Validation("Customer name is required"));

        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerName = request.CustomerName,
            Total = request.Total
        };

        db.Orders.Add(order);
        await db.SaveChangesAsync(ct);

        logger.LogInformation("Order {OrderId} created", order.Id);
        return Result<Guid>.Ok(order.Id);
    }
}
```

### Query (только чтение)

```csharp
public record GetOrderQuery(Guid Id) : IRequest<Result<OrderDto>>;

public sealed class GetOrderHandler(AppDbContext db)
    : IRequestHandler<GetOrderQuery, Result<OrderDto>>
{
    public async Task<Result<OrderDto>> Handle(GetOrderQuery request, CancellationToken ct)
    {
        var order = await db.Orders
            .Where(o => o.Id == request.Id)
            .Select(o => new OrderDto(o.Id, o.CustomerName, o.Total))
            .FirstOrDefaultAsync(ct);

        return order is not null
            ? Result<OrderDto>.Ok(order)
            : Result<OrderDto>.Fail(Error.NotFound("Order", request.Id));
    }
}
```

### Pipeline Behaviors (Cross-cutting concerns)

```csharp
// Logging — логирует каждый request
public sealed class LoggingBehavior<TRequest, TResponse>(
    ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        logger.LogInformation("Handling {Request}", typeof(TRequest).Name);
        var response = await next();
        logger.LogInformation("Handled {Request}", typeof(TRequest).Name);
        return response;
    }
}

// Validation — FluentValidation + MediatR
public sealed class ValidationBehavior<TRequest, TResponse>(
    IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        var failures = validators
            .Select(v => v.Validate(request))
            .SelectMany(r => r.Errors)
            .Where(f => f is not null)
            .ToList();

        if (failures.Count > 0)
            throw new ValidationException(failures);

        return await next();
    }
}

// Регистрация
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
services.AddTransient(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
```

---

## CQRS — Command Query Responsibility Segregation

Разделение моделей чтения и записи. Command — изменяет состояние, не возвращает данные (или возвращает ID). Query — читает, не изменяет.

**Простой CQRS (один БД):**
- Command → Write model (полная entity с валидацией)
- Query → Read model (проекция / DTO, без лишних данных)
- Разные DbContext или разные методы

**Полный CQRS (два хранилища):**
- Command → Write DB (нормализованная)
- Event → Sync → Read DB (денормализованная, оптимизированная для чтения)
- Eventual consistency

**Нюанс:** полный CQRS — сложность. Начинай с простого (один БД, разные модели). Усложняй только при явной необходимости.

---

## Vertical Slice Architecture

Один «срез» = одна фича. Handler + Validator + DTO в одной папке/файле. Минимум shared кода между срезами.

```
Features/
├── Orders/
│   ├── CreateOrder.cs     (Command + Handler + Validator)
│   ├── GetOrder.cs        (Query + Handler)
│   └── GetOrdersList.cs   (Query + Handler)
├── Customers/
│   ├── CreateCustomer.cs
│   └── GetCustomer.cs
```

**Преимущество:** изменение одной фичи не затрагивает другие. Легко удалять, рефакторить.

---

## См. также

- [Архитектуры](patterns.md)
- [MediatR Handlers](../Snippets/mediatr-handlers.md)
- [[Topics/Snippets/snippet-result-pattern|Result Usage]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
