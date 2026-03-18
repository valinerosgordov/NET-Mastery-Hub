---
tags: [ddd, value-objects, aggregate-root, domain-events, entity]
level: Senior
---

# DDD — Domain-Driven Design на практике

## Что это, зачем и когда

### Что такое DDD?
**Подход к проектированию, где код отражает бизнес-логику.** Не «таблицы и CRUD», а «заказ можно отменить только если он не отправлен». Бизнес-правила живут в доменных объектах, а не размазаны по контроллерам и сервисам.

**Аналогия:** Обычный код — это список SQL-запросов с `if`-ами. DDD — это разговор с бизнесом: «Клиент создаёт заказ → добавляет товары → оплачивает → заказ отправляется». Код читается как бизнес-процесс.

### Зачем?

| Без DDD (Anemic Model) | С DDD (Rich Domain Model) |
|------------------------|--------------------------|
| `orderService.Cancel(orderId)` — логика в сервисе | `order.Cancel()` — логика в самом объекте |
| Валидация разбросана по контроллерам | Валидация в Value Object при создании |
| `new Order { Status = "Active" }` — можно создать невалидный | Приватный конструктор — невалидный объект невозможен |
| Ошибки через `throw new Exception("...")` | `Result<T>` — ошибки как значения |
| Нет событий — всё через прямые вызовы | Domain Events: `OrderCreated` → отправить email, обновить статистику |

### Когда нужен DDD?

| Ситуация | DDD? | Почему |
|----------|------|--------|
| Сложная бизнес-логика (финансы, логистика, медицина) | **Да** | Много правил, состояний, зависимостей |
| Простой CRUD (блог, TODO-app) | **Нет** | Оверкилл, достаточно сервисов |
| Долгоживущий проект, команда > 3 человек | **Да** | Границы модулей и единый язык спасают от хаоса |
| Прототип / MVP | **Нет** | Сначала проверить гипотезу, потом рефакторить |

---

## Строительные блоки

| Блок | Что это | Пример |
|------|---------|--------|
| **Entity** | Объект с уникальной идентичностью | `Order`, `User` (сравниваются по Id) |
| **Value Object** | Объект без идентичности, сравнивается по значению | `Email`, `Money`, `Address` |
| **Aggregate Root** | Главная Entity, точка входа для изменений | `Order` (содержит `OrderItem`-ы) |
| **Domain Event** | Уведомление «что-то произошло в домене» | `OrderCreatedEvent`, `PaymentCompletedEvent` |
| **Repository** | Абстракция для сохранения/загрузки Aggregate | `IOrderRepository` |

---

## Entity\<TId\> — базовый класс

Entity сравнивается по идентичности (Id), не по значению полей.

```csharp
// Базовый класс для всех Entity
public abstract class Entity<TId> : IEquatable<Entity<TId>>
    where TId : notnull
{
    public TId Id { get; protected init; }

    protected Entity(TId id) => Id = id;

    // Для EF Core (параметрless конструктор)
    protected Entity() => Id = default!;

    public bool Equals(Entity<TId>? other)
        => other is not null && Id.Equals(other.Id);

    public override bool Equals(object? obj)
        => obj is Entity<TId> entity && Equals(entity);

    public override int GetHashCode() => Id.GetHashCode();

    public static bool operator ==(Entity<TId>? left, Entity<TId>? right)
        => Equals(left, right);

    public static bool operator !=(Entity<TId>? left, Entity<TId>? right)
        => !Equals(left, right);
}
```

**Нюанс:** Два `Order` с одинаковым `Id` — это один и тот же заказ, даже если поля отличаются (данные могли измениться). Два `Money(100, "USD")` — равны по значению, не по ссылке.

---

## Value Object — sealed record + Result

Value Object не имеет идентичности. Сравнивается по значению. Immutable. Валидация при создании.

```csharp
// Email — Value Object
public sealed record Email
{
    public string Value { get; init; }

    private Email(string value) => Value = value;

    public static Result<Email> Create(string? input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return Result<Email>.Fail(Error.Validation("Email.Empty", "Email is required"));

        var trimmed = input.Trim().ToLowerInvariant();

        if (!trimmed.Contains('@') || trimmed.Length > 256)
            return Result<Email>.Fail(Error.Validation("Email.Invalid", "Invalid email format"));

        // Санитизация пользовательского ввода
        var sanitized = WebUtility.HtmlEncode(trimmed);

        return Result<Email>.Ok(new Email(sanitized));
    }

    public override string ToString() => Value;
}

// Money — Value Object с арифметикой
public sealed record Money
{
    public decimal Amount { get; init; }
    public string Currency { get; init; }

    private static readonly FrozenSet<string> AllowedCurrencies =
        new[] { "USD", "EUR", "RUB", "GBP" }.ToFrozenSet();

    private Money(decimal amount, string currency)
        => (Amount, Currency) = (amount, currency);

    public static Result<Money> Create(decimal amount, string currency)
    {
        if (amount < 0)
            return Result<Money>.Fail(Error.Validation("Money.Negative", "Amount cannot be negative"));

        if (!AllowedCurrencies.Contains(currency))
            return Result<Money>.Fail(Error.Validation("Money.Currency", $"Unsupported currency: {currency}"));

        return Result<Money>.Ok(new Money(amount, currency));
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot add {Currency} and {other.Currency}");
        return new Money(Amount + other.Amount, Currency);
    }

    public override string ToString() => $"{Amount:F2} {Currency}";
}

// Использование
var emailResult = Email.Create(userInput);
// emailResult.IsSuccess → true/false, без исключений
```

### Почему именно так?

| Решение | Почему |
|---------|--------|
| `sealed record` | Immutable + value equality из коробки + нельзя наследовать |
| Приватный конструктор | Нельзя создать невалидный объект в обход `Create()` |
| `Create()` → `Result<T>` | Ошибки как значения, не исключения |
| `FrozenSet` для валидации enum | O(1) lookup, неизменяемый, оптимален для hot path |
| `WebUtility.HtmlEncode` | Защита от XSS при пользовательском вводе |
| `init` свойства | Immutable после создания, но EF Core может маппить |

---

## Aggregate Root — точка входа

Aggregate Root — Entity, через которую проходят ВСЕ изменения. Дочерние объекты (OrderItem) недоступны напрямую — только через Root.

```csharp
// Базовый класс Aggregate Root с Domain Events
public abstract class AggregateRoot<TId> : Entity<TId>
    where TId : notnull
{
    private readonly List<IDomainEvent> _domainEvents = [];

    public IReadOnlyCollection<IDomainEvent> DomainEvents
        => _domainEvents.AsReadOnly();

    protected AggregateRoot(TId id) : base(id) { }
    protected AggregateRoot() { } // EF Core

    protected void Raise(IDomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);

    public void ClearDomainEvents()
        => _domainEvents.Clear();
}

// Order — Aggregate Root
public sealed class Order : AggregateRoot<Guid>
{
    private readonly List<OrderItem> _items = [];

    public Guid CustomerId { get; private init; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public Money Total => CalculateTotal();

    private Order() { } // EF Core

    // Фабричный метод — единственный способ создать Order
    public static Result<Order> Create(Guid customerId)
    {
        if (customerId == Guid.Empty)
            return Result<Order>.Fail(Error.Validation("Order.CustomerId", "Customer ID is required"));

        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Draft
        };

        order.Raise(new OrderCreatedEvent(order.Id, customerId));

        return Result<Order>.Ok(order);
    }

    // Бизнес-метод — возвращает Result, не бросает exception
    public Result AddItem(Guid productId, int quantity, Money price)
    {
        if (Status != OrderStatus.Draft)
            return Result.Fail(Error.Validation("Order.NotDraft", "Can only add items to draft orders"));

        if (quantity <= 0)
            return Result.Fail(Error.Validation("Order.Quantity", "Quantity must be positive"));

        var existingItem = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existingItem is not null)
        {
            existingItem.IncreaseQuantity(quantity);
            return Result.Ok();
        }

        _items.Add(OrderItem.Create(productId, quantity, price));
        return Result.Ok();
    }

    public Result Cancel()
    {
        if (Status == OrderStatus.Shipped)
            return Result.Fail(Error.Validation("Order.AlreadyShipped", "Cannot cancel shipped order"));

        if (Status == OrderStatus.Cancelled)
            return Result.Fail(Error.Validation("Order.AlreadyCancelled", "Order is already cancelled"));

        Status = OrderStatus.Cancelled;
        Raise(new OrderCancelledEvent(Id));
        return Result.Ok();
    }

    public Result Confirm()
    {
        if (Status != OrderStatus.Draft)
            return Result.Fail(Error.Validation("Order.NotDraft", "Can only confirm draft orders"));

        if (_items.Count == 0)
            return Result.Fail(Error.Validation("Order.NoItems", "Order must have at least one item"));

        Status = OrderStatus.Confirmed;
        Raise(new OrderConfirmedEvent(Id, Total));
        return Result.Ok();
    }

    private Money CalculateTotal()
        => _items.Aggregate(
            Money.Create(0, "USD").Value!,
            (sum, item) => sum.Add(item.Subtotal));
}

// OrderItem — Entity внутри Aggregate (не Root!)
public sealed class OrderItem : Entity<Guid>
{
    public Guid ProductId { get; private init; }
    public int Quantity { get; private set; }
    public Money Price { get; private init; } = null!;
    public Money Subtotal => Money.Create(Price.Amount * Quantity, Price.Currency).Value!;

    private OrderItem() { } // EF Core

    internal static OrderItem Create(Guid productId, int quantity, Money price)
        => new()
        {
            Id = Guid.NewGuid(),
            ProductId = productId,
            Quantity = quantity,
            Price = price
        };

    internal void IncreaseQuantity(int amount) => Quantity += amount;
}

public enum OrderStatus { Draft, Confirmed, Shipped, Cancelled }
```

### Правила Aggregate

| Правило | Почему |
|---------|--------|
| Один `SaveChanges` = один Aggregate | Транзакционная граница = Aggregate |
| Дочерние Entity недоступны через DbSet | Только через Root: `order.AddItem(...)`, не `context.OrderItems.Add(...)` |
| Между Aggregates — только по Id | `order.CustomerId` (Guid), не `order.Customer` (navigation) |
| Бизнес-методы возвращают `Result` | Без исключений в бизнес-логике |
| Фабричный `Create()` + приватный конструктор | Невозможно создать невалидный объект |

---

## Domain Events

Уведомление о том, что произошло в домене. Обрабатываются после SaveChanges.

```csharp
// Интерфейс
public interface IDomainEvent
{
    Guid Id { get; }
    DateTime OccurredAt { get; }
}

// Базовая реализация
public abstract record DomainEvent : IDomainEvent
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

// Конкретные события
public sealed record OrderCreatedEvent(Guid OrderId, Guid CustomerId) : DomainEvent;
public sealed record OrderConfirmedEvent(Guid OrderId, Money Total) : DomainEvent;
public sealed record OrderCancelledEvent(Guid OrderId) : DomainEvent;

// Обработчик события
public sealed class OrderCreatedEventHandler(
    IEmailService emailService,
    ILogger<OrderCreatedEventHandler> logger)
{
    public async Task HandleAsync(OrderCreatedEvent @event, CancellationToken ct)
    {
        logger.LogInformation("Order {OrderId} created for customer {CustomerId}",
            @event.OrderId, @event.CustomerId);

        await emailService.SendOrderConfirmationAsync(@event.OrderId, ct);
    }
}
```

### Диспетчеризация через SaveChangesInterceptor

```csharp
public sealed class DomainEventInterceptor(IServiceProvider serviceProvider)
    : SaveChangesInterceptor
{
    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData, int result, CancellationToken ct)
    {
        var context = eventData.Context!;

        var aggregates = context.ChangeTracker.Entries<AggregateRoot<Guid>>()
            .Select(e => e.Entity)
            .Where(a => a.DomainEvents.Count > 0)
            .ToList();

        var events = aggregates.SelectMany(a => a.DomainEvents).ToList();
        aggregates.ForEach(a => a.ClearDomainEvents());

        using var scope = serviceProvider.CreateScope();
        foreach (var domainEvent in events)
        {
            // Резолвим обработчик по типу события
            var handlerType = typeof(IDomainEventHandler<>)
                .MakeGenericType(domainEvent.GetType());

            var handlers = scope.ServiceProvider.GetServices(handlerType);
            foreach (dynamic handler in handlers)
            {
                await handler.HandleAsync((dynamic)domainEvent, ct);
            }
        }

        return result;
    }
}

// Интерфейс обработчика
public interface IDomainEventHandler<in TEvent> where TEvent : IDomainEvent
{
    Task HandleAsync(TEvent @event, CancellationToken ct);
}
```

**Нюанс:** Events диспатчатся ПОСЛЕ `SaveChanges` (в `SavedChangesAsync`), не до. Это гарантирует, что данные уже сохранены. Если handler упадёт — данные не откатятся (eventual consistency).

---

## Result Pattern — полная реализация

```csharp
// Error — описание ошибки
public sealed record Error(string Code, string Message, ErrorType Type)
{
    public static Error Validation(string code, string message)
        => new(code, message, ErrorType.Validation);

    public static Error NotFound(string code, string message)
        => new(code, message, ErrorType.NotFound);

    public static Error Unauthorized(string code, string message)
        => new(code, message, ErrorType.Unauthorized);

    public static Error Conflict(string code, string message)
        => new(code, message, ErrorType.Conflict);

    public static Error Internal(string code, string message)
        => new(code, message, ErrorType.Internal);
}

public enum ErrorType { Validation, NotFound, Unauthorized, Conflict, Internal }

// Result без значения (для void-операций)
public sealed class Result
{
    public bool IsSuccess { get; }
    public Error? Error { get; }
    public bool IsFailure => !IsSuccess;

    private Result(bool isSuccess, Error? error)
        => (IsSuccess, Error) = (isSuccess, error);

    public static Result Ok() => new(true, null);
    public static Result Fail(Error error) => new(false, error);
}

// Result<T> с значением
public sealed class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public Error? Error { get; }
    public bool IsFailure => !IsSuccess;

    private Result(bool isSuccess, T? value, Error? error)
        => (IsSuccess, Value, Error) = (isSuccess, value, error);

    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(Error error) => new(false, default, error);

    // Функциональные методы
    public Result<TOut> Map<TOut>(Func<T, TOut> mapper)
        => IsSuccess ? Result<TOut>.Ok(mapper(Value!)) : Result<TOut>.Fail(Error!);

    public async Task<Result<TOut>> MapAsync<TOut>(Func<T, Task<TOut>> mapper)
        => IsSuccess ? Result<TOut>.Ok(await mapper(Value!)) : Result<TOut>.Fail(Error!);

    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<Error, TOut> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

// Маппинг Result → HTTP response в Minimal API
public static class ResultExtensions
{
    public static IResult ToResponse<T>(this Result<T> result, Func<T, IResult> onSuccess)
        => result.Match(
            onSuccess,
            error => error.Type switch
            {
                ErrorType.Validation => TypedResults.Problem(
                    title: "Validation Error",
                    detail: error.Message,
                    statusCode: StatusCodes.Status400BadRequest),
                ErrorType.NotFound => TypedResults.NotFound(
                    new ProblemDetails { Detail = error.Message }),
                ErrorType.Unauthorized => TypedResults.Problem(
                    statusCode: StatusCodes.Status403Forbidden),
                ErrorType.Conflict => TypedResults.Conflict(
                    new ProblemDetails { Detail = error.Message }),
                _ => TypedResults.Problem(
                    title: "Internal Error",
                    statusCode: StatusCodes.Status500InternalServerError)
            });
}

// Пример в endpoint
app.MapPost("/orders", async (
    CreateOrderRequest request,
    CreateOrderHandler handler,
    CancellationToken ct) =>
{
    var result = await handler.HandleAsync(request, ct);
    return result.ToResponse(order => TypedResults.Created($"/orders/{order.Id}", order));
});
```

---

## EF Core конфигурация для DDD

```csharp
// Order Configuration — маппинг Aggregate Root
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(20);

        // Value Object → отдельные столбцы
        builder.OwnsMany(o => o.Items, item =>
        {
            item.WithOwner().HasForeignKey("OrderId");
            item.HasKey(i => i.Id);

            // Nested Value Object
            item.OwnsOne(i => i.Price, price =>
            {
                price.Property(p => p.Amount).HasPrecision(18, 2);
                price.Property(p => p.Currency).HasMaxLength(3);
            });
        });

        // Navigation через backing field
        builder.Navigation(o => o.Items)
            .UsePropertyAccessMode(PropertyAccessMode.Field);

        // Domain Events НЕ маппятся в БД
        builder.Ignore(o => o.DomainEvents);
    }
}
```

---

## Полный flow: Endpoint → Handler → Domain → Persistence

```csharp
// Handler (Application Layer) — без MediatR
public sealed class CreateOrderHandler(
    IOrderRepository orderRepository,
    IUnitOfWork unitOfWork)
{
    public async Task<Result<OrderDto>> HandleAsync(
        CreateOrderRequest request, CancellationToken ct)
    {
        // 1. Создаём Value Object (валидация внутри)
        var emailResult = Email.Create(request.CustomerEmail);
        if (emailResult.IsFailure)
            return Result<OrderDto>.Fail(emailResult.Error!);

        // 2. Создаём Aggregate (валидация внутри)
        var orderResult = Order.Create(request.CustomerId);
        if (orderResult.IsFailure)
            return Result<OrderDto>.Fail(orderResult.Error!);

        var order = orderResult.Value!;

        // 3. Бизнес-операции
        foreach (var item in request.Items)
        {
            var priceResult = Money.Create(item.Price, item.Currency);
            if (priceResult.IsFailure)
                return Result<OrderDto>.Fail(priceResult.Error!);

            var addResult = order.AddItem(item.ProductId, item.Quantity, priceResult.Value!);
            if (addResult.IsFailure)
                return Result<OrderDto>.Fail(addResult.Error!);
        }

        // 4. Persist
        orderRepository.Add(order);
        await unitOfWork.SaveChangesAsync(ct);
        // Domain Events диспатчатся автоматически через Interceptor

        // 5. Map to DTO
        return Result<OrderDto>.Ok(new OrderDto(order.Id, order.Status.ToString(), order.Total.ToString()));
    }
}

public sealed record CreateOrderRequest(
    Guid CustomerId,
    string CustomerEmail,
    IReadOnlyList<OrderItemRequest> Items);

public sealed record OrderItemRequest(
    Guid ProductId,
    int Quantity,
    decimal Price,
    string Currency);

public sealed record OrderDto(Guid Id, string Status, string Total);
```

---

## Анти-паттерны

```csharp
// ✗ Anemic Model — логика в сервисе, Entity — мешок данных
public class Order
{
    public Guid Id { get; set; }            // set — любой может менять
    public OrderStatus Status { get; set; } // нет защиты
}
public class OrderService
{
    public void Cancel(Order order)
    {
        if (order.Status == OrderStatus.Shipped) throw new Exception("...");
        order.Status = OrderStatus.Cancelled; // бизнес-логика СНАРУЖИ
    }
}

// ✓ Rich Domain Model — логика ВНУТРИ Entity
public sealed class Order : AggregateRoot<Guid>
{
    public OrderStatus Status { get; private set; } // private set — защита

    public Result Cancel() // бизнес-логика ВНУТРИ
    {
        if (Status == OrderStatus.Shipped)
            return Result.Fail(Error.Validation("Order.Shipped", "Cannot cancel shipped order"));

        Status = OrderStatus.Cancelled;
        Raise(new OrderCancelledEvent(Id));
        return Result.Ok();
    }
}
```

---

## См. также

- [Architecture Patterns](patterns.md) — Clean Architecture, VSA
- [Result/CQRS](cqrs-mediatr.md) — Result Pattern детально
- [EF Core Patterns](../EFCore/patterns.md) — Aggregate Root в EF, Interceptors
