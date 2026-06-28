---
tags: [solid, srp, ocp, lsp, isp, dip, dry, kiss, yagni, architecture]
level: Senior
---

# SOLID, DRY, KISS, YAGNI

> Пять принципов SOLID (SRP, OCP, LSP, ISP, DIP) плюс DRY/KISS/YAGNI с примерами на C#, маркерами нарушений и решениями — ориентиры для гибкого, тестируемого и поддерживаемого кода.

## Что это, зачем и когда

### Что такое SOLID?
**5 принципов проектирования**, которые делают код гибким, тестируемым и поддерживаемым. Не правила — а ориентиры. Можно нарушать осознанно, но нельзя игнорировать.

**Аналогия:** SOLID — как правила дорожного движения. Можно ехать без них, но на перекрёстке будет авария. Чем больше проект, тем важнее правила.

### Зачем?

| Без SOLID | С SOLID |
|----------|---------|
| Класс на 2000 строк — делает всё | Маленькие классы, каждый отвечает за одно |
| Добавил фичу — сломалось в 5 местах | Расширяешь, не меняя существующий код |
| Невозможно тестировать — всё связано | Каждый класс тестируется отдельно через интерфейс |
| Заменить компонент = переписать половину | Заменяешь реализацию, не трогая остальное |

### Когда применять?
- **Всегда** в production-коде
- **Не фанатично** — прототипы, скрипты, простые CRUD могут обойтись без строгого следования
- Если класс > 200 строк или > 3 зависимостей — почти наверняка нарушен SRP

---

## S — Single Responsibility Principle

**Класс должен иметь одну причину для изменения.** Не «делает одну вещь», а «меняется по одной причине».

```csharp
// ✗ Нарушение — OrderService делает ВСЁ
public class OrderService
{
    public Order Create(CreateOrderRequest request) { /* бизнес-логика */ }
    public void SendConfirmationEmail(Order order) { /* отправка email */ }
    public string GeneratePdf(Order order) { /* генерация PDF */ }
    public void SaveToDatabase(Order order) { /* работа с БД */ }
}
// Причины для изменения:
// 1. Бизнес-правила заказа
// 2. Формат email
// 3. Шаблон PDF
// 4. Структура БД
// → 4 причины = нарушение SRP

// ✓ Правильно — каждый класс = одна ответственность
public sealed class CreateOrderHandler(
    IOrderRepository repository,
    IUnitOfWork unitOfWork)
{
    public async Task<Result<Order>> HandleAsync(
        CreateOrderRequest request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId);
        if (order.IsFailure) return order;

        repository.Add(order.Value!);
        await unitOfWork.SaveChangesAsync(ct);
        // Domain Event → email отправится отдельным handler-ом
        return order;
    }
}

public sealed class OrderConfirmationEmailHandler(IEmailService email)
    : IDomainEventHandler<OrderCreatedEvent>
{
    public async Task HandleAsync(OrderCreatedEvent @event, CancellationToken ct)
        => await email.SendOrderConfirmationAsync(@event.OrderId, ct);
}

public sealed class OrderPdfGenerator(ITemplateEngine templates)
{
    public byte[] Generate(Order order) { /* только PDF */ }
}
```

**Маркер нарушения:** слова «и» в описании класса. «Создаёт заказ **и** отправляет email **и** генерирует PDF» → 3 класса.

---

## O — Open/Closed Principle

**Открыт для расширения, закрыт для изменения.** Добавляешь новое поведение без правки существующего кода.

```csharp
// ✗ Нарушение — добавление нового способа оплаты = правка switch
public class PaymentProcessor
{
    public Result Process(Payment payment)
    {
        return payment.Type switch
        {
            "CreditCard" => ProcessCreditCard(payment),
            "BankTransfer" => ProcessBankTransfer(payment),
            "Crypto" => ProcessCrypto(payment), // ← каждый новый тип = правка ЭТОГО класса
            _ => Result.Fail(Error.Validation("Payment.Unknown", "Unknown type"))
        };
    }
}

// ✓ Правильно — новый способ = новый класс, старый код не меняется
public interface IPaymentProcessor
{
    string PaymentType { get; }
    Task<Result> ProcessAsync(Payment payment, CancellationToken ct);
}

public sealed class CreditCardProcessor : IPaymentProcessor
{
    public string PaymentType => "CreditCard";
    public async Task<Result> ProcessAsync(Payment payment, CancellationToken ct)
    {
        // логика кредитной карты
        return Result.Ok();
    }
}

public sealed class BankTransferProcessor : IPaymentProcessor
{
    public string PaymentType => "BankTransfer";
    public async Task<Result> ProcessAsync(Payment payment, CancellationToken ct)
    {
        // логика банковского перевода
        return Result.Ok();
    }
}

// Добавить Crypto → создать CryptoProcessor : IPaymentProcessor
// Ничего не менять в существующем коде!

// Резолвер — через DI
public sealed class PaymentProcessorResolver(
    IEnumerable<IPaymentProcessor> processors)
{
    private readonly FrozenDictionary<string, IPaymentProcessor> _map =
        processors.ToFrozenDictionary(p => p.PaymentType);

    public Result<IPaymentProcessor> Resolve(string paymentType)
        => _map.TryGetValue(paymentType, out var processor)
            ? Result<IPaymentProcessor>.Ok(processor)
            : Result<IPaymentProcessor>.Fail(
                Error.Validation("Payment.Unknown", $"Unknown type: {paymentType}"));
}

// Регистрация
builder.Services.AddScoped<IPaymentProcessor, CreditCardProcessor>();
builder.Services.AddScoped<IPaymentProcessor, BankTransferProcessor>();
builder.Services.AddSingleton<PaymentProcessorResolver>();
```

**Инструменты OCP:** интерфейсы, стратегии, DI, полиморфизм. Не `switch` по типам.

---

## L — Liskov Substitution Principle

**Подкласс должен быть заменяем базовым классом без поломки.** Если `Dog : Animal`, то везде где используется `Animal`, можно подставить `Dog` и ничего не сломается.

```csharp
// ✗ Нарушение — квадрат ломает контракт прямоугольника
public class Rectangle
{
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    public int Area => Width * Height;
}

public class Square : Rectangle
{
    public override int Width
    {
        get => base.Width;
        set { base.Width = value; base.Height = value; } // ← неожиданное поведение!
    }
    public override int Height
    {
        get => base.Height;
        set { base.Width = value; base.Height = value; }
    }
}

// Клиентский код ожидает:
Rectangle rect = new Square();
rect.Width = 5;
rect.Height = 10;
// rect.Area == 50? Нет! Area == 100, потому что Width тоже стал 10
// → Подстановка Square вместо Rectangle СЛОМАЛА логику

// ✓ Правильно — отдельные типы
public interface IShape
{
    int Area { get; }
}

public sealed record Rectangle(int Width, int Height) : IShape
{
    public int Area => Width * Height;
}

public sealed record Square(int Side) : IShape
{
    public int Area => Side * Side;
}
```

```csharp
// ✗ Нарушение LSP в реальном коде — NotImplementedException
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct);
    Task AddAsync(T entity, CancellationToken ct);
    Task DeleteAsync(T entity, CancellationToken ct);
}

public class ReadOnlyRepository<T> : IRepository<T>
{
    public Task<T?> GetByIdAsync(Guid id, CancellationToken ct) => /* OK */;
    public Task AddAsync(T entity, CancellationToken ct)
        => throw new NotImplementedException(); // ← НАРУШЕНИЕ LSP!
    public Task DeleteAsync(T entity, CancellationToken ct)
        => throw new NotImplementedException(); // ← НАРУШЕНИЕ LSP!
}

// ✓ Правильно — ISP: разделить интерфейс
public interface IReadRepository<T>
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct);
}

public interface IWriteRepository<T> : IReadRepository<T>
{
    Task AddAsync(T entity, CancellationToken ct);
    Task DeleteAsync(T entity, CancellationToken ct);
}
```

**Маркер нарушения:** `throw new NotImplementedException()`, `NotSupportedException`, пустые методы. Если подкласс не может выполнить контракт — он не должен наследовать.

---

## I — Interface Segregation Principle

**Клиент не должен зависеть от методов, которые не использует.** Лучше много маленьких интерфейсов, чем один толстый.

```csharp
// ✗ Нарушение — толстый интерфейс
public interface IUserService
{
    Task<User> GetByIdAsync(Guid id, CancellationToken ct);
    Task<User> CreateAsync(CreateUserRequest request, CancellationToken ct);
    Task UpdateAsync(User user, CancellationToken ct);
    Task DeleteAsync(Guid id, CancellationToken ct);
    Task SendEmailAsync(Guid userId, string message, CancellationToken ct);
    Task<byte[]> GenerateReportAsync(CancellationToken ct);
    Task ImportFromCsvAsync(Stream csv, CancellationToken ct);
}
// ProfilePage нужен только GetById, но зависит от ВСЕГО

// ✓ Правильно — маленькие, сфокусированные интерфейсы
public interface IUserReader
{
    Task<User?> GetByIdAsync(Guid id, CancellationToken ct);
}

public interface IUserWriter
{
    Task<Result<User>> CreateAsync(CreateUserRequest request, CancellationToken ct);
    Task<Result> UpdateAsync(User user, CancellationToken ct);
    Task<Result> DeleteAsync(Guid id, CancellationToken ct);
}

public interface IUserNotifier
{
    Task SendEmailAsync(Guid userId, string message, CancellationToken ct);
}

// ProfilePage зависит ТОЛЬКО от IUserReader — минимальная связность
public sealed class GetUserProfileHandler(IUserReader users)
{
    public async Task<Result<UserProfile>> HandleAsync(Guid userId, CancellationToken ct)
    {
        var user = await users.GetByIdAsync(userId, ct);
        return user is null
            ? Result<UserProfile>.Fail(Error.NotFound("User.NotFound", "User not found"))
            : Result<UserProfile>.Ok(new UserProfile(user.Id, user.Name, user.Email));
    }
}
```

**Практика в .NET:**
- `IReadRepository<T>` + `IWriteRepository<T>` вместо одного `IRepository<T>`
- `IEmailSender` отдельно от `IUserService`
- Handlers зависят от конкретного интерфейса, не от «бога-сервиса»

---

## D — Dependency Inversion Principle

**Модули верхнего уровня не зависят от модулей нижнего. Оба зависят от абстракций.**

```
✗ Без DIP:
OrderHandler → PostgresOrderRepository → Npgsql
(высокий уровень зависит от низкого — если сменить БД, переписывать handler)

✓ С DIP:
OrderHandler → IOrderRepository ← PostgresOrderRepository
(оба зависят от интерфейса — меняешь реализацию, handler не трогаешь)
```

```csharp
// Domain Layer (ядро) — определяет интерфейс
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id, CancellationToken ct);
    void Add(Order order);
}

// Application Layer — использует интерфейс (не знает про БД)
public sealed class ConfirmOrderHandler(
    IOrderRepository repository,
    IUnitOfWork unitOfWork,
    TimeProvider timeProvider)
{
    public async Task<Result> HandleAsync(Guid orderId, CancellationToken ct)
    {
        var order = await repository.GetByIdAsync(orderId, ct);
        if (order is null)
            return Result.Fail(Error.NotFound("Order.NotFound", "Order not found"));

        var result = order.Confirm();
        if (result.IsFailure) return result;

        await unitOfWork.SaveChangesAsync(ct);
        return Result.Ok();
    }
}

// Infrastructure Layer — реализует интерфейс (знает про EF Core)
public sealed class OrderRepository(AppDbContext context) : IOrderRepository
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => await context.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public void Add(Order order) => context.Orders.Add(order);
}

// Регистрация — Infrastructure знает про конкретику, Domain/Application — нет
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
```

### Направление зависимостей в Clean Architecture

```
Api → Application → Domain
 ↓                    ↑
Infrastructure ───────┘ (реализует интерфейсы Domain)
```

| Слой | Зависит от | Не знает о |
|------|-----------|-----------|
| **Domain** | Ничего | Application, Infrastructure, Api |
| **Application** | Domain | Infrastructure, Api |
| **Infrastructure** | Domain, Application | Api |
| **Api** | Application (и иногда Infrastructure для DI) | — |

**Нюанс:** DIP в .NET реализуется через DI-контейнер. Интерфейс в Domain, реализация в Infrastructure, регистрация в `AddInfrastructure()` extension method.

---

## SOLID — сводная таблица

| Принцип | Одним словом | Маркер нарушения | Решение |
|---------|-------------|------------------|---------|
| **S**RP | Одна ответственность | Класс > 200 строк, «и» в описании | Разбить на классы по ответственности |
| **O**CP | Расширяй, не меняй | `switch` по типу, постоянные правки одного класса | Интерфейсы, стратегии, DI |
| **L**SP | Подстановка без поломки | `NotImplementedException`, пустые методы | Правильная иерархия, ISP |
| **I**SP | Маленькие интерфейсы | Класс реализует интерфейс, но использует 2 из 10 методов | Разбить интерфейс |
| **D**IP | Зависимость от абстракции | `new ConcreteClass()` в бизнес-логике | Интерфейс + DI |

---

## DRY — Don't Repeat Yourself

**Каждое знание должно иметь единственное представление в системе.** Не дублируй логику, данные, конфигурацию.

```csharp
// ✗ Дублирование — одна и та же валидация в 3 местах
public class CreateOrderHandler
{
    public Result Handle(CreateOrderRequest req)
    {
        if (req.Email is null || !req.Email.Contains('@') || req.Email.Length > 256)
            return Result.Fail(Error.Validation("Email.Invalid", "Bad email"));
        // ...
    }
}

public class UpdateProfileHandler
{
    public Result Handle(UpdateProfileRequest req)
    {
        if (req.Email is null || !req.Email.Contains('@') || req.Email.Length > 256) // ДУБЛЬ!
            return Result.Fail(Error.Validation("Email.Invalid", "Bad email"));
        // ...
    }
}

// ✓ Одно место — Value Object
public sealed record Email
{
    public string Value { get; init; }
    private Email(string value) => Value = value;

    public static Result<Email> Create(string? input)
    {
        if (string.IsNullOrWhiteSpace(input) || !input.Contains('@') || input.Length > 256)
            return Result<Email>.Fail(Error.Validation("Email.Invalid", "Bad email"));
        return Result<Email>.Ok(new Email(input.Trim().ToLowerInvariant()));
    }
}

// Теперь в обоих handlers:
var email = Email.Create(req.Email); // Логика в ОДНОМ месте
```

### Что считать дублированием?

| Дубль | Не дубль |
|-------|---------|
| Одна и та же валидация в 3 handler-ах | Похожий, но РАЗНЫЙ SQL в 2 репозиториях |
| Один и тот же маппинг Entity → DTO в 5 местах | Два цикла `for` с разной логикой |
| Connection string в 3 файлах конфигурации | Две функции по 3 строки, которые случайно похожи |

**Правило:** дублирование логики — это DRY violation. Дублирование структуры (два похожих класса с разной логикой) — это нормально. **Не создавай абстракцию ради устранения случайного совпадения.**

---

## KISS — Keep It Simple, Stupid

**Простейшее решение, которое работает.** Не усложняй без причины.

```csharp
// ✗ Over-engineering — абстракция ради абстракции
public interface IDateTimeProvider { DateTime UtcNow { get; } }
public interface IDateTimeProviderFactory { IDateTimeProvider Create(string timezone); }
public class DateTimeProviderFactoryBuilder
{
    public IDateTimeProviderFactory Build() => /* 50 строк... */;
}
// Для чего? Чтобы получить DateTime.UtcNow...

// ✓ KISS — используй встроенный TimeProvider
public sealed class OrderService(TimeProvider timeProvider)
{
    public Order Create() => new() { CreatedAt = timeProvider.GetUtcNow() };
}
// В prod: TimeProvider.System. В тестах: FakeTimeProvider. Всё.
```

```csharp
// ✗ Сложно — Generic Repository + Specification + Unit of Work + Mediator
// для простого CRUD с 5 таблицами
public interface IRepository<T> where T : Entity { /* 15 методов */ }
public interface ISpecification<T> { Expression<Func<T, bool>> ToExpression(); }
public class GenericRepository<T> : IRepository<T> { /* 200 строк */ }

// ✓ KISS — DbContext напрямую для простого CRUD
public sealed class GetProductHandler(AppDbContext context)
{
    public async Task<ProductDto?> HandleAsync(Guid id, CancellationToken ct)
        => await context.Products
            .AsNoTracking()
            .Where(p => p.Id == id)
            .Select(p => new ProductDto(p.Id, p.Name, p.Price))
            .FirstOrDefaultAsync(ct);
}
// Репозиторий нужен когда есть СЛОЖНАЯ логика доступа, не для проксирования DbSet
```

### Маркеры нарушения KISS

| Сигнал | Что делать |
|--------|-----------|
| Класс-обёртка, который просто вызывает другой класс | Убрать обёртку |
| 3 уровня абстракции для одной операции | Сократить до 1-2 |
| Паттерн ради паттерна (Repository над DbSet без логики) | Использовать DbSet напрямую |
| «А вдруг потом понадобится» | См. YAGNI ↓ |

---

## YAGNI — You Aren't Gonna Need It

**Не пиши код «на будущее».** Реализуй только то, что нужно СЕЙЧАС.

```csharp
// ✗ YAGNI violation — «а вдруг захотим поддержать 5 баз данных»
public interface IDatabase { /* абстракция над всеми БД */ }
public interface IDatabaseFactory { IDatabase Create(DatabaseType type); }
public enum DatabaseType { PostgreSQL, SqlServer, MongoDB, CosmosDB, SQLite }
// Реально используется: только PostgreSQL. Остальное НИКОГДА не понадобится.

// ✓ YAGNI — только то что нужно сейчас
builder.Services.AddDbContext<AppDbContext>(o => o.UseNpgsql(connectionString));
// Когда (ЕСЛИ) понадобится другая БД — тогда и абстрагируем.
```

```csharp
// ✗ YAGNI — «добавлю поддержку плагинов на случай если»
public interface IPlugin { void Execute(); }
public class PluginLoader { public IPlugin Load(string path) => /* reflection... */; }
public class PluginManager { /* 300 строк для управления плагинами */ }
// Никто не просил плагины. Никто не будет их писать.

// ✓ Просто сделай то, что просят
public sealed class ExportService
{
    public byte[] ExportToCsv(IEnumerable<Order> orders) { /* 20 строк */ }
}
// Когда попросят Excel — добавишь ExportToExcel. Не раньше.
```

### Правило принятия решения

```
Нужна ли эта фича СЕЙЧАС?
├── Да → Реализуй
└── Нет → Не реализуй
         ├── «Но потом будет сложнее!» → Почти всегда нет. Реализуй потом.
         └── «Но это займёт 5 минут!» → 5 минут × 50 таких решений = месяц лишнего кода
```

---

## DRY + KISS + YAGNI — как они связаны

| Ситуация | Принцип | Действие |
|----------|---------|----------|
| Одна и та же валидация в 3 местах | **DRY** | Вынеси в Value Object |
| Абстрактная фабрика для 1 реализации | **KISS** | Убери фабрику, используй класс напрямую |
| «Добавлю кеширование на всякий случай» | **YAGNI** | Добавишь когда будут проблемы с производительностью |
| 3 похожих метода, но с разной логикой | **НЕ DRY** | Это не дубль — оставь как есть (KISS) |
| Универсальный Generic Repository | **KISS + YAGNI** | DbContext достаточно для простого CRUD |

**Золотое правило:** Пиши минимальный код, который решает текущую задачу. Дублирование устраняй только когда оно реальное (одна логика в N местах). Абстракции добавляй когда они оправданы, не заранее.

---

## См. также

- [ООП и классы]() — Наследование, интерфейсы, полиморфизм
- [DI и Configuration]() — Dependency Injection в .NET
- [DDD на практике](ddd.md) — SOLID в контексте Domain-Driven Design
- [Architecture Patterns]() — Clean Architecture (применение DIP)
