# OOP

## Access Modifiers

| Модификатор | Область видимости |
|-------------|-------------------|
| `public` | Везде |
| `internal` | В пределах сборки (default для top-level типов) |
| `private` | В пределах класса (default для членов) |
| `protected` | Класс + наследники |
| `protected internal` | Класс + наследники ИЛИ та же сборка |
| `private protected` | Класс + наследники В той же сборке |
| `file` (C# 11) | В пределах файла |

```csharp
// file-scoped type — виден только в этом файле
file class InternalHelper { }

// private protected — наследник должен быть в той же сборке
public class Base
{
    private protected void SecretMethod() { }
}
```

**Нюанс:** `internal` — основной модификатор для скрытия деталей реализации между проектами. `[InternalsVisibleTo("Tests")]` — открытие internal для тестов.

---

## Interface vs Abstract Class

| Аспект | Interface | Abstract Class |
|--------|-----------|----------------|
| Множественное наследование | Да | Нет (только один класс) |
| Конструктор | Нет | Да |
| Поля | Нет | Да |
| Default implementation | Да (C# 8+) | Да |
| Состояние | Нет | Да |
| Назначение | «Может делать» (контракт) | «Является» (общая база) |

```csharp
// Interface — контракт поведения
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
}

// Abstract class — общая логика + шаблонный метод
public abstract class BaseEntity
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;

    public abstract void Validate(); // обязан реализовать наследник
}
```

**Нюанс:** Default Interface Methods (C# 8+) позволяют добавлять методы в интерфейс без поломки существующих реализаций. Но вызвать можно только через интерфейс, не через класс.

---

## sealed, Inheritance, Composition

**sealed** — запрет наследования класса или override метода. JIT может девиртуализировать вызовы sealed методов — performance gain.

```csharp
public sealed class OrderService : IOrderService { } // нельзя наследовать

public class Base
{
    public virtual void Process() { }
}
public class Derived : Base
{
    public sealed override void Process() { } // дальше переопределить нельзя
}
```

### Inheritance vs Composition

```csharp
// ✗ Inheritance — жёсткая связь, хрупкая иерархия
public class LoggingOrderService : OrderService { }

// ✓ Composition — гибкость, DI, тестируемость
public class LoggingOrderService(IOrderService inner, ILogger logger) : IOrderService
{
    public async Task<Order> GetAsync(Guid id, CancellationToken ct)
    {
        logger.LogInformation("Getting order {Id}", id);
        return await inner.GetAsync(id, ct);
    }
}
```

**Правило:** наследование — когда есть настоящая иерархия «is-a» с общим поведением. Composition — во всех остальных случаях (Decorator, Strategy, DI).

---

## Polymorphism, Encapsulation

### Polymorphism

**Runtime (virtual/override):** вызов по реальному типу объекта. Требует `virtual` на базовом методе.

**Compile-time (overloading):** выбор метода по сигнатуре при компиляции.

```csharp
// Runtime polymorphism
public abstract class Shape
{
    public abstract double Area();
}
public class Circle(double radius) : Shape
{
    public override double Area() => Math.PI * radius * radius;
}

// Interface polymorphism — предпочтительный способ в .NET
IEnumerable<Shape> shapes = [new Circle(5), new Rectangle(3, 4)];
double totalArea = shapes.Sum(s => s.Area());
```

**Нюанс:** `new` (hiding) vs `override`: `new` скрывает метод базового класса, но при вызове через базовый тип вызовется старый метод. Почти всегда нужен `override`.

### Encapsulation

```csharp
public class BankAccount
{
    private decimal _balance;

    public decimal Balance => _balance; // readonly доступ

    public void Deposit(decimal amount)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(amount);
        _balance += amount;
    }
}
```

---

## Static Constructor, Extension Methods

### Static Constructor

Вызывается один раз перед первым доступом к типу. Thread-safe (гарантия CLR). Нет параметров.

```csharp
public class ConnectionPool
{
    private static readonly Dictionary<string, string> _defaults;

    static ConnectionPool()
    {
        // Вызывается один раз, thread-safe
        _defaults = LoadDefaults();
    }
}
```

**Нюанс:** исключение в static constructor → `TypeInitializationException`. Тип становится непригодным до перезапуска AppDomain. Не делать тяжёлых операций.

### Extension Methods

```csharp
public static class StringExtensions
{
    // this — первый параметр, указывает расширяемый тип
    public static bool IsNullOrEmpty(this string? value)
        => string.IsNullOrEmpty(value);

    // Generic extension
    public static T? FirstOrNull<T>(this IEnumerable<T> source) where T : struct
        => source.Any() ? source.First() : null;
}

// Использование
"hello".IsNullOrEmpty(); // false
```

**Нюанс:** extension method — обычный статический метод с синтаксическим сахаром. При наличии instance-метода с той же сигнатурой, instance-метод имеет приоритет. LINQ — целиком построен на extension methods.

---

## IDisposable и Finalizer

```csharp
public class FileProcessor : IDisposable
{
    private readonly StreamReader _reader;
    private bool _disposed;

    public void Dispose()
    {
        if (_disposed) return;
        _reader.Dispose();
        _disposed = true;
        GC.SuppressFinalize(this); // не вызывать finalizer
    }
}

// using — гарантия Dispose
using var processor = new FileProcessor("data.txt");
// Dispose вызовется при выходе из scope
```

**Нюанс:** `IAsyncDisposable` + `await using` — для async cleanup (например, закрытие сетевого соединения). Finalizer (`~ClassName`) — крайний случай для unmanaged ресурсов.

---

## См. также

- [C# Reference: ООП и классы](../../../Reference/csharp-oop-classes.md)
