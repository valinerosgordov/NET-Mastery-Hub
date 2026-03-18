---
tags: [oop, classes, interfaces, inheritance, records]
level: Senior
---

# ООП и классы

## Что это, зачем и когда

### Что такое ООП?
**Способ организации кода через объекты** — каждый объект имеет данные (свойства) и поведение (методы). Четыре столпа: инкапсуляция, наследование, полиморфизм, абстракция.

**Аналогия:** Конструктор LEGO. Каждый кубик (объект) — самостоятельный, имеет форму (свойства) и способы крепления (методы). Из кубиков собираешь что угодно.

### Зачем?

| Принцип | Что даёт | Пример |
|---------|----------|--------|
| **Инкапсуляция** | Скрываешь детали, показываешь только нужное | `private decimal _balance` + `public void Withdraw()` |
| **Наследование** | Переиспользуешь код базового класса | `Animal` → `Dog`, `Cat` |
| **Полиморфизм** | Один интерфейс, разные реализации | `IPayment.Process()` — карта или перевод |
| **Абстракция** | Работаешь с «что делает», не «как делает» | `ILogger.Log()` — не важно, куда пишет |

### Когда что использовать?

| Конструкция | Когда | Когда НЕ нужно |
|-------------|-------|----------------|
| **class** | Объект с идентичностью и поведением | Простые данные без логики |
| **interface** | Контракт, несколько реализаций, DI | Одна реализация навсегда |
| **abstract class** | Общая логика + шаблон для подклассов | Если достаточно interface |
| **sealed class** | По умолчанию всё sealed (запрет наследования) | Только если нужно наследование |
| **record** | DTO, Value Object, immutable данные | Mutable объекты с поведением |

---

> Справочник по объектно-ориентированному программированию в C# 13.
> Теория → практика → senior-level код → вопросы интервью.

---

## Классы

### Объявление класса, поля, свойства, методы

```csharp
namespace Domain.Models;

public class Product
{
    // Поле (field) — хранит состояние
    private decimal _price;

    // Свойство (property) — контролируемый доступ к данным
    public string Name { get; set; } = string.Empty;

    // Read-only свойство с backing field
    public decimal Price
    {
        get => _price;
        set => _price = value >= 0
            ? value
            : throw new ArgumentOutOfRangeException(nameof(value));
    }

    // Метод — поведение объекта
    public decimal CalculateDiscount(decimal percentage)
        => Price * (percentage / 100m);

    // Expression-bodied метод
    public override string ToString() => $"{Name}: {Price:C}";
}
```

### Access Modifiers

| Modifier             | Класс | Сборка | Наследник (та же сборка) | Наследник (другая сборка) | Все |
| -------------------- | ----- | ------ | ------------------------ | ------------------------- | --- |
| `public`             | +     | +      | +                        | +                         | +   |
| `private`            | +     | -      | -                        | -                         | -   |
| `protected`          | +     | -      | +                        | +                         | -   |
| `internal`           | +     | +      | +                        | -                         | -   |
| `protected internal` | +     | +      | +                        | +                         | -   |
| `private protected`  | +     | -      | +                        | -                         | -   |

```csharp
namespace Domain.Models;

public class Account
{
    public string Owner { get; init; } = string.Empty;           // Виден всем
    private decimal _balance;                                     // Только внутри класса
    protected int TransactionCount { get; set; }                  // Класс + наследники
    internal string InternalCode { get; set; } = string.Empty;   // Только в пределах сборки
    protected internal bool IsActive { get; set; }                // Сборка ИЛИ наследник
    private protected byte Priority { get; set; }                 // Наследник в ТОЙ ЖЕ сборке
}
```

> [!question]- **Интервью: Перечислите модификаторы доступа. Когда internal?**
> `public`, `private`, `protected`, `internal`, `protected internal`, `private protected`, `file` (C# 11).
> `internal` — доступен только внутри сборки. Используется для инфраструктурных классов, которые не должны быть частью публичного API. `protected internal` — наследникам ИЛИ коду в той же сборке.

### Конструкторы

#### Default конструктор

```csharp
namespace Domain.Models;

public class Customer
{
    public string Name { get; set; } = "Unknown";
    public string Email { get; set; } = string.Empty;

    // Если не объявить никакой конструктор, компилятор сгенерирует пустой default.
    // Если объявить любой — default НЕ генерируется автоматически.
}
```

#### Параметризованный конструктор

```csharp
namespace Domain.Models;

public class Order
{
    public Guid Id { get; }
    public string CustomerName { get; }
    public DateTime CreatedAt { get; }

    public Order(string customerName)
    {
        Id = Guid.NewGuid();
        CustomerName = customerName;
        CreatedAt = DateTime.UtcNow;
    }

    // Перегрузка конструктора с chaining через this(...)
    public Order(string customerName, DateTime createdAt)
        : this(customerName)
    {
        CreatedAt = createdAt;
    }
}
```

#### Static конструктор

```csharp
namespace Infrastructure.Configuration;

public class AppSettings
{
    // Вызывается ОДИН раз, до первого обращения к типу.
    // Нельзя вызвать вручную, нет access modifier, нет параметров.
    static AppSettings()
    {
        DefaultTimeout = TimeSpan.FromSeconds(30);
        MachineName = Environment.MachineName;
    }

    public static TimeSpan DefaultTimeout { get; }
    public static string MachineName { get; }
}
```

#### Primary Constructor (C# 12)

Параметры primary constructor — это **не свойства и не поля**, а captured параметры, доступные во всём теле класса.

```csharp
namespace Application.Services;

// Параметры primary constructor захватываются как переменные.
// Идеально для DI — лаконично, без boilerplate.
public class OrderService(
    IOrderRepository orderRepository,
    ILogger<OrderService> logger)
{
    public async Task<Result<Order>> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        logger.LogInformation("Получение заказа {OrderId}", id);
        var order = await orderRepository.GetByIdAsync(id, ct).ConfigureAwait(false);

        return order is null
            ? Result<Order>.Failure($"Заказ {id} не найден")
            : Result<Order>.Success(order);
    }
}
```

> **Важно:** параметры primary constructor мутабельны. Если нужна иммутабельность — присвоить `readonly` полю вручную:

```csharp
namespace Application.Services;

public class PaymentService(IPaymentGateway gateway)
{
    // Явное readonly поле — гарантия иммутабельности
    private readonly IPaymentGateway _gateway = gateway;

    public Task ChargeAsync(decimal amount)
        => _gateway.ChargeAsync(amount);
}
```

### Деконструкторы (Deconstruct)

Метод `Deconstruct` позволяет разложить объект на составные части через tuple syntax.

```csharp
namespace Domain.Models;

public class Point(double x, double y)
{
    public double X { get; } = x;
    public double Y { get; } = y;

    // Деконструктор — не путать с финализатором!
    public void Deconstruct(out double x, out double y)
    {
        x = X;
        y = Y;
    }
}

// Использование:
// var point = new Point(3.0, 4.0);
// var (x, y) = point;  // x = 3.0, y = 4.0
```

### Финализаторы (~ClassName)

Финализатор (finalizer / destructor) вызывается GC перед освобождением объекта. **Не используй** — непредсказуемое время вызова, замедляет GC (объект попадает в finalization queue), невозможно гарантировать порядок.

```csharp
namespace Legacy;

// АНТИПАТТЕРН — показано для справки
public class ResourceHolder
{
    private IntPtr _handle;

    ~ResourceHolder()
    {
        // Вызывается GC — время вызова не определено
        // Используй IDisposable вместо этого!
        ReleaseHandle(_handle);
    }

    private static void ReleaseHandle(IntPtr handle)
    {
        // Освобождение unmanaged ресурса
    }
}
```

> **Правило:** вместо финализаторов используй `IDisposable` + `using`. См. секцию "IDisposable и using pattern" ниже.

### Partial Classes

Разделение одного класса на несколько файлов. Часто используется для code generation (EF Core, source generators).

```csharp
// Файл: Models/User.cs
namespace Domain.Models;

public partial class User
{
    public Guid Id { get; init; }
    public string Name { get; set; } = string.Empty;
}

// Файл: Models/User.Validation.cs
namespace Domain.Models;

public partial class User
{
    public bool IsValid() => !string.IsNullOrWhiteSpace(Name);
}
```

C# 13 — **partial properties**. Partial methods доступны с C# 3 (расширенные — с C# 9):

```csharp
namespace Domain.Models;

// Файл 1: определение
public partial class Customer
{
    public partial string FullName { get; }
    public partial bool Validate();
}

// Файл 2: реализация
public partial class Customer
{
    public partial string FullName => $"{FirstName} {LastName}";
    public partial bool Validate() => !string.IsNullOrWhiteSpace(FirstName);
}
```

### Static Classes и Static Members

Static класс — не создаётся через `new`, не участвует в наследовании.

```csharp
namespace Application.Helpers;

// Static class — утилитные функции, extension methods
public static class StringExtensions
{
    public static string Truncate(this string value, int maxLength)
        => value.Length <= maxLength ? value : value[..maxLength] + "...";

    public static bool IsNullOrEmpty(this string? value)
        => string.IsNullOrEmpty(value);
}
```

Static members в обычном классе:

```csharp
namespace Domain.Models;

public class ConnectionPool
{
    // Shared между всеми экземплярами
    private static int _activeConnections;
    public static int MaxConnections { get; set; } = 100;

    public static int ActiveConnections => _activeConnections;

    public void Open()
    {
        Interlocked.Increment(ref _activeConnections);
    }

    public void Close()
    {
        Interlocked.Decrement(ref _activeConnections);
    }
}
```

### this и base

```csharp
namespace Domain.Models;

public class Entity
{
    public Guid Id { get; init; }

    public Entity(Guid id) => Id = id;
    public Entity() : this(Guid.NewGuid()) { } // this — вызов другого конструктора
}

public class AuditableEntity : Entity
{
    public DateTime CreatedAt { get; init; }

    // base — вызов конструктора базового класса
    public AuditableEntity(Guid id) : base(id)
    {
        CreatedAt = DateTime.UtcNow;
    }
}
```

`this` для fluent API:

```csharp
namespace Application.Builders;

public class QueryBuilder
{
    private string _table = string.Empty;
    private string _where = string.Empty;

    // Возвращаем this для chaining
    public QueryBuilder From(string table) { _table = table; return this; }
    public QueryBuilder Where(string condition) { _where = condition; return this; }
    public string Build() => $"SELECT * FROM {_table} WHERE {_where}";
}

// Использование:
// var sql = new QueryBuilder().From("Orders").Where("Id = @id").Build();
```

---

## Свойства

### Auto-Properties

```csharp
namespace Domain.Models;

public class Item
{
    // Полная авто-свойство: getter + setter
    public string Name { get; set; } = string.Empty;

    // Read-only авто-свойство — можно присвоить только в конструкторе
    public Guid Id { get; } = Guid.NewGuid();

    // Авто-свойство с private setter
    public DateTime CreatedAt { get; private set; } = DateTime.UtcNow;
}
```

### Init-Only Properties (C# 9)

Можно задать при инициализации (`new { ... }`), но не изменить после.

```csharp
namespace Domain.Models;

public class Address
{
    public string Street { get; init; } = string.Empty;
    public string City { get; init; } = string.Empty;
    public string PostalCode { get; init; } = string.Empty;
}

// var addr = new Address { Street = "Main St", City = "Moscow", PostalCode = "101000" };
// addr.City = "SPb"; // Ошибка компиляции — init-only!
```

### Required Properties (C# 11)

Компилятор **заставляет** указать значение при создании объекта.

```csharp
namespace Application.Contracts;

public class CreateOrderRequest
{
    public required string CustomerName { get; init; }
    public required decimal Amount { get; init; }
    public string? Notes { get; init; }
}

// var request = new CreateOrderRequest(); // Ошибка: required members не заданы
// var request = new CreateOrderRequest { CustomerName = "John", Amount = 99.99m }; // OK
```

### Computed Properties

```csharp
namespace Domain.Models;

public class Rectangle(double width, double height)
{
    public double Width { get; } = width;
    public double Height { get; } = height;

    // Computed — вычисляется каждый раз при обращении
    public double Area => Width * Height;
    public double Perimeter => 2 * (Width + Height);
    public bool IsSquare => Math.Abs(Width - Height) < 0.0001;
}
```

### Expression-Bodied Members

```csharp
namespace Domain.Models;

public class Temperature
{
    private double _celsius;

    // Expression-bodied property
    public double Celsius
    {
        get => _celsius;
        set => _celsius = value;
    }

    // Expression-bodied read-only property
    public double Fahrenheit => _celsius * 9 / 5 + 32;

    // Expression-bodied method
    public static Temperature FromFahrenheit(double f) => new() { Celsius = (f - 32) * 5 / 9 };

    // Expression-bodied constructor
    public Temperature(double celsius) => _celsius = celsius;

    // Expression-bodied finalizer (для примера — не использовать!)
    // ~Temperature() => Console.WriteLine("Finalized");

    public override string ToString() => $"{Celsius:F1}°C ({Fahrenheit:F1}°F)";
}
```

### `field` keyword (C# 14 preview)

Автоматический доступ к backing field свойства без явного объявления.

```csharp
namespace Domain.Models;

public class Sensor
{
    // field — ссылка на auto-generated backing field
    // Нет нужды объявлять private double _temperature;
    public double Temperature
    {
        get => field;
        set => field = value < -273.15 ? -273.15 : value; // валидация прямо в setter
    }

    public string Label
    {
        get => field;
        set => field = value.Trim(); // нормализация в setter
    } = "Default";
}
```

---

## Наследование

### Базовый и производный класс

```csharp
namespace Domain.Models;

// Базовый класс
public class Shape
{
    public string Color { get; init; } = "Black";

    public virtual double GetArea() => 0;

    public override string ToString() => $"{GetType().Name} [{Color}] Area={GetArea():F2}";
}

// Производный класс
public class Circle(double radius) : Shape
{
    public double Radius { get; } = radius;

    public override double GetArea() => Math.PI * Radius * Radius;
}

public class Square(double side) : Shape
{
    public double Side { get; } = side;

    public override double GetArea() => Side * Side;
}
```

### Method Overriding: virtual / override / sealed

```csharp
namespace Domain.Models;

public class Animal
{
    public virtual string Speak() => "...";
}

public class Dog : Animal
{
    // override — переопределение виртуального метода
    public override string Speak() => "Woof!";
}

public class GuideDog : Dog
{
    // sealed override — запрет дальнейшего переопределения в наследниках
    public sealed override string Speak() => "Bark (calmly)";
}

// public class SuperGuideDog : GuideDog
// {
//     public override string Speak() => "..."; // Ошибка: sealed!
// }
```

### Method Hiding: new keyword

```csharp
namespace Domain.Models;

public class BaseLogger
{
    public void Log(string message) => Console.WriteLine($"[Base] {message}");
}

public class CustomLogger : BaseLogger
{
    // new — скрывает метод базового класса (НЕ переопределяет).
    // При вызове через ссылку базового типа — вызовется метод базового класса!
    public new void Log(string message) => Console.WriteLine($"[Custom] {message}");
}

// CustomLogger logger = new();
// logger.Log("test");            // [Custom] test
// ((BaseLogger)logger).Log("test"); // [Base] test — ВНИМАНИЕ!
```

> **Правило:** `new` hiding — почти всегда признак плохого дизайна. Используй `virtual` / `override`.

### Abstract Classes и Abstract Methods

```csharp
namespace Domain.Models;

// Нельзя создать экземпляр abstract класса
public abstract class Notification
{
    public required string Recipient { get; init; }
    public DateTime SentAt { get; private set; }

    // Abstract метод — нет реализации, наследник ОБЯЗАН реализовать
    public abstract Task SendAsync(CancellationToken ct = default);

    // Обычный метод — общая логика для всех наследников
    protected void MarkAsSent() => SentAt = DateTime.UtcNow;
}

public class EmailNotification(IEmailSender sender) : Notification
{
    public required string Subject { get; init; }

    public override async Task SendAsync(CancellationToken ct)
    {
        await sender.SendAsync(Recipient, Subject, ct).ConfigureAwait(false);
        MarkAsSent();
    }
}
```

### Порядок вызова конструкторов в иерархии

Конструкторы вызываются **от базового к производному**. Деструкторы — наоборот.

```csharp
namespace Examples;

public class A
{
    public A() => Console.WriteLine("A ctor");
}

public class B : A
{
    public B() => Console.WriteLine("B ctor");
}

public class C : B
{
    public C() => Console.WriteLine("C ctor");
}

// new C() выведет:
// A ctor
// B ctor
// C ctor
```

С параметрами:

```csharp
namespace Domain.Models;

public class Entity(Guid id)
{
    public Guid Id { get; } = id;
}

public class AuditableEntity(Guid id, string createdBy) : Entity(id)
{
    public string CreatedBy { get; } = createdBy;
    public DateTime CreatedAt { get; } = DateTime.UtcNow;
}

public class Order(Guid id, string createdBy, string customerName)
    : AuditableEntity(id, createdBy)
{
    public string CustomerName { get; } = customerName;
}
```

---

## Интерфейсы

### Объявление и реализация

```csharp
namespace Domain.Abstractions;

public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(Guid id, CancellationToken ct = default);
}
```

```csharp
namespace Infrastructure.Persistence;

public class OrderRepository(AppDbContext context) : IRepository<Order>
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => await context.Orders.FindAsync([id], ct).ConfigureAwait(false);

    public async Task<IReadOnlyList<Order>> GetAllAsync(CancellationToken ct)
        => await context.Orders.ToListAsync(ct).ConfigureAwait(false);

    public async Task AddAsync(Order entity, CancellationToken ct)
        => await context.Orders.AddAsync(entity, ct).ConfigureAwait(false);

    public Task UpdateAsync(Order entity, CancellationToken ct)
    {
        context.Orders.Update(entity);
        return Task.CompletedTask;
    }

    public async Task DeleteAsync(Guid id, CancellationToken ct)
    {
        var entity = await GetByIdAsync(id, ct).ConfigureAwait(false);
        if (entity is not null)
            context.Orders.Remove(entity);
    }
}
```

### Default Interface Methods (C# 8)

Интерфейсы могут содержать реализацию по умолчанию — не ломает существующих потребителей при добавлении нового метода.

```csharp
namespace Domain.Abstractions;

public interface ILogger
{
    void Log(string message);

    // Default implementation — потребитель может не реализовывать
    void LogError(string message) => Log($"[ERROR] {message}");
    void LogWarning(string message) => Log($"[WARN] {message}");
}

public class ConsoleLogger : ILogger
{
    // Реализуем только Log — LogError и LogWarning получаем бесплатно
    public void Log(string message) => Console.WriteLine(message);
}
```

> **Нюанс:** default methods доступны только через ссылку на интерфейс:

```csharp
// var logger = new ConsoleLogger();
// logger.LogError("fail"); // Ошибка компиляции!
// ((ILogger)logger).LogError("fail"); // OK
```

### Static Abstract Members (C# 11)

Позволяет требовать static члены в generic constraints — основа для generic math.

```csharp
namespace Domain.Abstractions;

public interface IParseable<TSelf> where TSelf : IParseable<TSelf>
{
    static abstract TSelf Parse(string input);
    static abstract bool TryParse(string input, out TSelf? result);
}

public readonly record struct Money(decimal Amount, string Currency) : IParseable<Money>
{
    public static Money Parse(string input)
    {
        // Формат: "100.50 USD"
        var parts = input.Split(' ');
        return new Money(decimal.Parse(parts[0]), parts[1]);
    }

    public static bool TryParse(string input, out Money result)
    {
        try { result = Parse(input); return true; }
        catch { result = default; return false; }
    }
}

// Generic метод, использующий static abstract:
// T ParseValue<T>(string input) where T : IParseable<T> => T.Parse(input);
```

### Interface vs Abstract Class — когда что

| Критерий                        | Interface                     | Abstract Class                  |
| ------------------------------- | ----------------------------- | ------------------------------- |
| Множественное наследование      | Да                            | Нет                             |
| Поля (state)                    | Нет (только static)           | Да                              |
| Конструкторы                    | Нет                           | Да                              |
| Access modifiers на members     | Да (C# 8+)                    | Да                              |
| Default implementation          | Да (C# 8+)                    | Да                              |
| Когда использовать              | Контракт / capability         | Общая база + shared state/logic |

**Эмпирическое правило:**
- **Interface** — "что объект умеет" (`IDisposable`, `IComparable`, `IRepository`)
- **Abstract class** — "что объект является" (`Notification`, `Shape`)

> [!question]- **Интервью: Interface vs Abstract Class — когда что?**
> **Interface** — контракт поведения. Несколько реализаций, DI, тестируемость (мокирование). C# 8+ — default implementations.
>
> **Abstract class** — частичная реализация + общее состояние. Один базовый класс. Template Method pattern.
>
> **Правило:** prefer composition over inheritance. Интерфейсы + DI в 90% случаев лучше наследования.

### Множественная реализация

```csharp
namespace Domain.Models;

public interface ISerializable
{
    string Serialize();
}

public interface ICloneable<T>
{
    T Clone();
}

public interface IAuditable
{
    DateTime CreatedAt { get; }
    DateTime? UpdatedAt { get; }
}

// Класс реализует несколько интерфейсов
public class Document : ISerializable, ICloneable<Document>, IAuditable
{
    public required string Title { get; init; }
    public required string Content { get; init; }
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; init; }

    public string Serialize()
        => System.Text.Json.JsonSerializer.Serialize(this);

    public Document Clone()
        => new() { Title = Title, Content = Content, CreatedAt = DateTime.UtcNow };
}
```

### Explicit Interface Implementation

Когда два интерфейса имеют одноимённые члены, или нужно скрыть метод с основного API класса.

```csharp
namespace Domain.Models;

public interface IFileReader
{
    string Read(string path);
}

public interface INetworkReader
{
    string Read(string url);
}

public class UniversalReader : IFileReader, INetworkReader
{
    // Explicit — метод доступен только через интерфейсную ссылку
    string IFileReader.Read(string path) => File.ReadAllText(path);
    // ⚠️ new HttpClient() + .Result — антипаттерн! Только для демо синтаксиса.
    // В реальном коде: IHttpClientFactory + async/await.
    string INetworkReader.Read(string url) => new HttpClient().GetStringAsync(url).Result;

    // Общий public метод
    public string Read(string source)
        => source.StartsWith("http")
            ? ((INetworkReader)this).Read(source)
            : ((IFileReader)this).Read(source);
}
```

---

## Полиморфизм

### Compile-Time (Overloading) vs Runtime (Overriding)

```csharp
namespace Examples;

// Compile-time полиморфизм — method overloading
public class Calculator
{
    public int Add(int a, int b) => a + b;
    public double Add(double a, double b) => a + b;
    public decimal Add(decimal a, decimal b) => a + b;
    // Компилятор выбирает нужный метод на этапе компиляции
}

// Runtime полиморфизм — method overriding
public abstract class PaymentProcessor
{
    public abstract Task<bool> ProcessAsync(decimal amount);
}

public class StripeProcessor : PaymentProcessor
{
    public override Task<bool> ProcessAsync(decimal amount)
        => Task.FromResult(true); // Stripe API call
}

public class PayPalProcessor : PaymentProcessor
{
    public override Task<bool> ProcessAsync(decimal amount)
        => Task.FromResult(true); // PayPal API call
}

// PaymentProcessor processor = GetProcessor(); // Runtime — какой конкретный тип?
// await processor.ProcessAsync(100m);           // Вызовется нужный override
```

### Covariance и Contravariance

```csharp
namespace Examples;

// Covariance (out) — можно вернуть более конкретный тип
public interface IProducer<out T>
{
    T Produce();
}

// Contravariance (in) — можно принять более базовый тип
public interface IConsumer<in T>
{
    void Consume(T item);
}

public class Animal { public string Name { get; init; } = ""; }
public class Dog : Animal { }

public class DogProducer : IProducer<Dog>
{
    public Dog Produce() => new() { Name = "Rex" };
}

public class AnimalConsumer : IConsumer<Animal>
{
    public void Consume(Animal item) => Console.WriteLine(item.Name);
}

// Covariance: IProducer<Dog> -> IProducer<Animal>
// IProducer<Animal> producer = new DogProducer(); // OK

// Contravariance: IConsumer<Animal> -> IConsumer<Dog>
// IConsumer<Dog> consumer = new AnimalConsumer(); // OK
```

### Upcasting и Downcasting

```csharp
namespace Examples;

public class Vehicle { public string Brand { get; init; } = ""; }
public class Car : Vehicle { public int Doors { get; init; } }
public class Truck : Vehicle { public double Payload { get; init; } }

public static class CastingExamples
{
    public static void Demo()
    {
        Car car = new() { Brand = "Toyota", Doors = 4 };

        // Upcast — всегда безопасен, неявный
        Vehicle vehicle = car;

        // Downcast — требует явный cast, может бросить InvalidCastException
        Car sameCar = (Car)vehicle; // OK — объект реально Car

        // Безопасный downcast с is
        if (vehicle is Car c)
        {
            Console.WriteLine($"Car with {c.Doors} doors");
        }

        // Безопасный downcast с as (возвращает null, если не удалось)
        Truck? truck = vehicle as Truck; // null — vehicle не Truck

        // Pattern matching (предпочтительный способ)
        string description = vehicle switch
        {
            Car { Doors: > 2 } sedan => $"Sedan {sedan.Brand}",
            Car compact => $"Compact {compact.Brand}",
            Truck { Payload: > 10 } heavy => $"Heavy truck {heavy.Brand}",
            Truck t => $"Light truck {t.Brand}",
            _ => $"Vehicle {vehicle.Brand}"
        };
    }
}
```

---

## Records

### Record Class (C# 9)

Immutable reference type с value-based equality. Идеален для DTO, events, value objects.

```csharp
namespace Domain.ValueObjects;

// Positional record — компилятор генерирует:
// - properties (init-only)
// - конструктор
// - Deconstruct
// - Equals / GetHashCode (value-based)
// - ToString
// - оператор == / !=
public record Address(string Street, string City, string PostalCode);
```

```csharp
namespace Application.Contracts;

// Не-positional record с ручными свойствами
public record OrderCreatedEvent
{
    public required Guid OrderId { get; init; }
    public required string CustomerName { get; init; }
    public required decimal TotalAmount { get; init; }
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}
```

### Record Struct (C# 10)

Value type с value-based equality. Хранится на стеке, нет аллокации в heap.

```csharp
namespace Domain.ValueObjects;

// Positional record struct — мутабельный по умолчанию!
public record struct Coordinate(double Latitude, double Longitude);

// readonly record struct — иммутабельный (предпочтительный)
public readonly record struct Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch");

        return this with { Amount = Amount + other.Amount };
    }
}
```

### With Expressions

Создание копии record с изменёнными полями (non-destructive mutation).

```csharp
namespace Examples;

public record UserProfile(string Name, string Email, int Age);

public static class WithExpressionDemo
{
    public static void Demo()
    {
        var original = new UserProfile("Alice", "alice@mail.com", 30);

        // with — создаёт копию с изменённым Email
        var updated = original with { Email = "newalice@mail.com" };

        // original.Email == "alice@mail.com"  — не изменился
        // updated.Email  == "newalice@mail.com"
        // original == updated // false — Email различается
    }
}
```

### Positional Records и Deconstruct

```csharp
namespace Domain.ValueObjects;

public record Range(int Start, int End)
{
    // Computed property
    public int Length => End - Start;

    // Deconstruct генерируется автоматически для positional records
    // public void Deconstruct(out int start, out int end) => (start, end) = (Start, End);
}

// var range = new Range(10, 50);
// var (start, end) = range; // Deconstruct: start = 10, end = 50
```

### Equality: Value-Based vs Reference-Based

```csharp
namespace Examples;

public class PersonClass
{
    public string Name { get; init; } = "";
    public int Age { get; init; }
}

public record PersonRecord(string Name, int Age);

public static class EqualityDemo
{
    public static void Demo()
    {
        // class — reference equality по умолчанию
        var c1 = new PersonClass { Name = "Bob", Age = 25 };
        var c2 = new PersonClass { Name = "Bob", Age = 25 };
        var classEqual = c1 == c2;      // false — разные объекты в heap

        // record — value-based equality
        var r1 = new PersonRecord("Bob", 25);
        var r2 = new PersonRecord("Bob", 25);
        var recordEqual = r1 == r2;     // true — одинаковые значения полей

        // ReferenceEquals всегда проверяет ссылку
        var sameRef = ReferenceEquals(r1, r2); // false — разные объекты
    }
}
```

### Record vs Class vs Struct — таблица сравнения

| Характеристика     | `class`          | `record class`     | `struct`         | `record struct`    |
| ------------------ | ---------------- | ------------------ | ---------------- | ------------------ |
| Тип                | Reference        | Reference          | Value            | Value              |
| Heap/Stack         | Heap             | Heap               | Inline*          | Inline*            |
| Equality           | Reference        | Value-based        | Value (Equals**) | Value-based        |
| Immutable          | Нет (вручную)    | Да (по умолчанию)  | Нет (вручную)    | readonly = да      |
| Наследование       | Да               | Да (только record) | Нет              | Нет                |
| `with` expression  | Нет              | Да                 | Да (C# 10+)      | Да                 |
| Deconstruct        | Вручную          | Авто (positional)  | Вручную          | Авто (positional)  |
| Когда использовать | Mutable entities | DTO, events, VO    | Маленькие данные | VO без аллокаций   |

> \* **Inline** — на стеке для локальных переменных, внутри объекта-владельца в heap для полей класса. Отдельной heap-аллокации нет (если нет boxing).
> \*\* Struct по умолчанию получает value equality через `ValueType.Equals` (reflection, медленно). Оператор `==` **не генерируется** автоматически — нужна ручная перегрузка. В `record struct` оператор `==` генерируется.

---

## Вложенные типы

### Nested Classes

```csharp
namespace Domain.Models;

public class OrderAggregate
{
    private readonly List<OrderLine> _lines = [];

    public IReadOnlyList<OrderLine> Lines => _lines;

    public void AddLine(string product, int quantity, decimal price)
        => _lines.Add(new OrderLine(product, quantity, price));

    public decimal Total => _lines.Sum(l => l.Subtotal);

    // Nested class — тесно связан с родителем, не используется отдельно
    public class OrderLine(string product, int quantity, decimal unitPrice)
    {
        public string Product { get; } = product;
        public int Quantity { get; } = quantity;
        public decimal UnitPrice { get; } = unitPrice;
        public decimal Subtotal => Quantity * UnitPrice;
    }
}
```

### Когда использовать

- Тип имеет смысл **только** в контексте родительского класса
- Builder pattern — `Order.Builder`
- Реализация интерфейса, не предназначенная для внешнего потребления
- State machine states — `Connection.ConnectedState`

```csharp
namespace Domain.Models;

// Пример: state machine с nested types
public abstract class ConnectionState
{
    public abstract string Status { get; }
    public virtual bool CanConnect => false;
    public virtual bool CanDisconnect => false;

    public sealed class Disconnected : ConnectionState
    {
        public override string Status => "Disconnected";
        public override bool CanConnect => true;
    }

    public sealed class Connected : ConnectionState
    {
        public override string Status => "Connected";
        public override bool CanDisconnect => true;
    }

    public sealed class Error(string message) : ConnectionState
    {
        public override string Status => $"Error: {message}";
        public override bool CanConnect => true;
    }
}
```

---

## Ключевые паттерны

### Composition over Inheritance

Наследование создаёт жёсткую связь. Композиция — гибкость через интерфейсы.

```csharp
namespace Domain.Models;

// ПЛОХО: наследование — жёсткая иерархия
// public class FlyingSwimmingDuck : FlyingAnimal, SwimmingAnimal { } // Невозможно!

// ХОРОШО: композиция через интерфейсы
public interface IFlyable
{
    void Fly();
}

public interface ISwimmable
{
    void Swim();
}

// Реализации поведения
public class WingFlight : IFlyable
{
    public void Fly() => Console.WriteLine("Flying with wings");
}

public class Floating : ISwimmable
{
    public void Swim() => Console.WriteLine("Floating on water");
}

// Композиция: duck ИМЕЕТ способности, а не ЯВЛЯЕТСЯ чем-то
public class Duck(IFlyable flyBehavior, ISwimmable swimBehavior)
{
    public void Fly() => flyBehavior.Fly();
    public void Swim() => swimBehavior.Swim();
}

// var duck = new Duck(new WingFlight(), new Floating());
// duck.Fly();  // "Flying with wings"
// duck.Swim(); // "Floating on water"
```

### Sealed Classes — производительность и дизайн

`sealed` запрещает наследование. JIT применяет devirtualization — прямой вызов вместо виртуальной таблицы.

```csharp
namespace Domain.Models;

// sealed — нельзя наследовать, JIT оптимизирует virtual/override вызовы
public sealed class InMemoryCache
{
    private readonly Dictionary<string, object> _store = [];

    public void Set(string key, object value) => _store[key] = value;

    public T? Get<T>(string key)
        => _store.TryGetValue(key, out var value) ? (T)value : default;

    public bool Remove(string key) => _store.Remove(key);
}
```

> **Рекомендация:** помечай `sealed` все классы, которые **не предназначены** для наследования. Это и производительность, и явный дизайн-интент. В .NET runtime большинство internal классов — sealed.

> [!question]- **Интервью: sealed — зачем запечатывать класс?**
> Запрещает наследование. Причины: (1) безопасность — гарантия контракта, (2) производительность — JIT девиртуализирует вызовы sealed-методов, (3) дизайн — явное указание что класс не для расширения.
>
> **Практика:** в .NET Runtime многие internal классы sealed для производительности. В ASP.NET Core handler-ы часто sealed.

### Object Initializers

```csharp
namespace Application.Contracts;

public class OrderDto
{
    public required Guid Id { get; init; }
    public required string CustomerName { get; init; }
    public List<string> Items { get; init; } = [];
    public decimal Total { get; init; }
}

public static class InitializerDemo
{
    public static OrderDto Create()
    {
        // Object initializer — задаём свойства при создании
        return new OrderDto
        {
            Id = Guid.NewGuid(),
            CustomerName = "Alice",
            Items = ["Widget", "Gadget", "Doohickey"],
            Total = 149.99m
        };
    }

    // Collection initializer (C# 12 — collection expressions)
    public static List<int> Numbers() => [1, 2, 3, 4, 5];
    public static int[] Array() => [10, 20, 30];
}
```

### IDisposable и using pattern

Для детерминированного освобождения ресурсов (connections, files, handles).

```csharp
namespace Infrastructure.Services;

// Базовый паттерн IDisposable
public sealed class DatabaseConnection : IDisposable
{
    private bool _disposed;
    private readonly NpgsqlConnection _connection;

    public DatabaseConnection(string connectionString)
    {
        _connection = new NpgsqlConnection(connectionString);
        _connection.Open();
    }

    public NpgsqlCommand CreateCommand(string sql)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        return new NpgsqlCommand(sql, _connection);
    }

    public void Dispose()
    {
        if (_disposed) return;
        _connection.Dispose();
        _disposed = true;
    }
}
```

```csharp
namespace Infrastructure.Services;

// IAsyncDisposable — для асинхронного освобождения
public sealed class FileProcessor : IAsyncDisposable
{
    private readonly StreamWriter _writer;

    public FileProcessor(string path)
    {
        _writer = new StreamWriter(path, append: true);
    }

    public async Task WriteLineAsync(string line)
        => await _writer.WriteLineAsync(line).ConfigureAwait(false);

    public async ValueTask DisposeAsync()
    {
        await _writer.DisposeAsync().ConfigureAwait(false);
    }
}
```

Использование `using`:

```csharp
namespace Examples;

public static class UsingDemo
{
    // Классический using block
    public static void Classic()
    {
        using (var connection = new DatabaseConnection("connstr"))
        {
            var cmd = connection.CreateCommand("SELECT 1");
            // ...
        } // Dispose() вызывается здесь

    }

    // using declaration (C# 8) — Dispose при выходе из enclosing scope (блока {})
    public static void Declaration()
    {
        using var connection = new DatabaseConnection("connstr");
        var cmd = connection.CreateCommand("SELECT 1");
        // Dispose() вызывается при выходе из текущего scope (здесь — метод)
    }

    // await using — для IAsyncDisposable
    public static async Task AsyncUsing()
    {
        await using var processor = new FileProcessor("log.txt");
        await processor.WriteLineAsync("Hello");
        // DisposeAsync() вызывается при выходе из метода
    }
}
```

---

> [!question]- **Интервью: Dispose pattern — зачем GC.SuppressFinalize?**
> Без `SuppressFinalize` объект проходит через Finalization Queue даже после Dispose: лишняя сборка GC, промоция в следующее поколение. `SuppressFinalize` убирает объект из очереди финализации.
>
> **Паттерн:** `IDisposable` + `IAsyncDisposable`. Всегда `using` / `await using`. Финализатор — только safety net для unmanaged ресурсов.

## См. также

- [Типы и память](types-and-memory.md)
- [Collections и LINQ](collections-linq.md)
- [Delegates и Events](delegates-events.md)
- [Modern C#](modern-features.md)
