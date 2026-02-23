---
tags: [modern-csharp, pattern-matching, records, nullable, generics]
level: Senior
---

# Современные возможности C# (8–14)

> Справочник по фичам языка C# 8–14.
> Теория → практика → senior-level код → вопросы интервью.

---

## Pattern Matching (C# 8–11)

### Type Patterns (C# 8)

Проверка типа с одновременным объявлением переменной:

```csharp
object obj = "Hello, World!";

// Классический type pattern
if (obj is string s)
{
    Console.WriteLine(s.ToUpper()); // HELLO, WORLD!
}

// В switch expression
string result = obj switch
{
    string s => $"Строка длиной {s.Length}",
    int i    => $"Число: {i}",
    null     => "null",
    _        => $"Неизвестный тип: {obj.GetType().Name}"
};
```

### Declaration и Constant Patterns (C# 8)

```csharp
// Declaration pattern — объявляем переменную при совпадении
if (GetOrder() is Order { Status: OrderStatus.Completed } order)
{
    ProcessCompleted(order);
}

// Constant pattern — сравнение с константой
if (statusCode is 200)
{
    // OK
}

// null check через constant pattern
if (name is null)
{
    throw new ArgumentNullException(nameof(name));
}

// not null — обратная проверка
if (name is not null)
{
    Console.WriteLine(name);
}
```

### Property Patterns (C# 8, расширены в C# 10)

```csharp
public record Person(string Name, int Age, Address? Address);
public record Address(string City, string Country);

// Простой property pattern
if (person is { Age: > 18, Name: not null })
{
    Console.WriteLine("Совершеннолетний с именем");
}

// Вложенный property pattern (C# 10 — extended property patterns)
if (person is { Address.City: "Moscow" })
{
    Console.WriteLine("Москвич");
}

// Эквивалент до C# 10 (вложенная запись):
if (person is { Address: { City: "Moscow" } })
{
    Console.WriteLine("Москвич");
}

// Комбинация в switch expression
string category = person switch
{
    { Age: < 13 }                          => "Ребёнок",
    { Age: >= 13 and < 18 }               => "Подросток",
    { Age: >= 18, Address.Country: "RU" }  => "Взрослый (РФ)",
    { Age: >= 18 }                         => "Взрослый",
    _                                       => "Неизвестно"
};
```

### Relational и Logical Patterns (C# 9)

Операторы сравнения `<`, `>`, `<=`, `>=` и логические комбинаторы `and`, `or`, `not`:

```csharp
// Relational patterns
string GetTemperatureDescription(double temp) => temp switch
{
    < -20                 => "Экстремальный холод",
    >= -20 and < 0        => "Мороз",
    >= 0 and < 15         => "Прохладно",
    >= 15 and < 25        => "Комфортно",
    >= 25 and < 35        => "Жарко",
    >= 35                 => "Экстремальная жара"
};

// Logical pattern: not
if (statusCode is not (200 or 201 or 204))
{
    LogError(statusCode);
}

// Комбинация and / or
if (value is > 0 and < 100 or 999)
{
    // value в (0, 100) или равно 999
}
```

### Positional Patterns с деконструкцией (C# 8)

```csharp
public readonly record struct Point(double X, double Y);

// Positional pattern — деконструкция через Deconstruct
string GetQuadrant(Point point) => point switch
{
    (0, 0)       => "Начало координат",
    ( > 0, > 0)  => "Первая четверть",
    ( < 0, > 0)  => "Вторая четверть",
    ( < 0, < 0)  => "Третья четверть",
    ( > 0, < 0)  => "Четвёртая четверть",
    (_, 0) or (0, _) => "На оси"
};

// Деконструкция tuple
(int status, string? body) = GetResponse();
var message = (status, body) switch
{
    (200, not null) => $"OK: {body}",
    (200, null)     => "OK: пустой ответ",
    (404, _)        => "Не найдено",
    (>= 500, _)     => "Ошибка сервера",
    _               => $"Статус: {status}"
};
```

### List Patterns (C# 11)

Мощный pattern matching по коллекциям и массивам:

```csharp
int[] numbers = [1, 2, 3, 4, 5];

// Точное совпадение
if (numbers is [1, 2, 3, 4, 5])
{
    Console.WriteLine("Точное совпадение");
}

// Discard и slice
if (numbers is [1, _, _, _, 5])
{
    Console.WriteLine("Начинается с 1, заканчивается на 5");
}

// Slice pattern (..) — «любое количество элементов»
if (numbers is [1, .., 5])
{
    Console.WriteLine("Начинается с 1, заканчивается на 5 (любая длина)");
}

// Захват значений
if (numbers is [var first, .., var last])
{
    Console.WriteLine($"Первый: {first}, Последний: {last}");
}

// Захват slice
if (numbers is [_, .. var middle, _])
{
    // middle = [2, 3, 4]
}

// Вложенные patterns внутри list pattern
string[] args = ["--verbose", "--output", "result.json"];
var config = args switch
{
    ["--help"]                        => new Config(ShowHelp: true),
    ["--verbose", "--output", var f]  => new Config(Verbose: true, OutputFile: f),
    [var cmd, ..]                     => new Config(Command: cmd),
    []                                => Config.Default
};
```

### Exhaustive Matching и Discard

```csharp
// Компилятор предупредит, если не все случаи покрыты (для enum)
public enum PaymentStatus { Pending, Completed, Failed, Refunded }

string GetMessage(PaymentStatus status) => status switch
{
    PaymentStatus.Pending   => "Ожидание",
    PaymentStatus.Completed => "Завершён",
    PaymentStatus.Failed    => "Ошибка",
    PaymentStatus.Refunded  => "Возврат",
    // _ => "Неизвестно" — discard для exhaustive matching
    // Без _ компилятор выдаст warning, если enum расширят
};
```

### Nested Patterns

```csharp
public record Order(
    int Id,
    OrderStatus Status,
    decimal Total,
    Customer? Customer);

public record Customer(string Name, CustomerTier Tier);
public enum CustomerTier { Regular, Silver, Gold, Platinum }
public enum OrderStatus { New, Processing, Shipped, Delivered, Cancelled }

// Глубоко вложенный pattern matching
decimal CalculateDiscount(Order order) => order switch
{
    { Status: OrderStatus.Cancelled } => 0m,
    { Total: < 100m }                 => 0m,
    { Customer: null }                 => 0m,

    { Total: >= 1000m, Customer: { Tier: CustomerTier.Platinum } }
        => order.Total * 0.20m,

    { Total: >= 500m, Customer: { Tier: CustomerTier.Gold or CustomerTier.Platinum } }
        => order.Total * 0.15m,

    { Customer: { Tier: CustomerTier.Gold } }
        => order.Total * 0.10m,

    { Customer: { Tier: CustomerTier.Silver } }
        => order.Total * 0.05m,

    _ => 0m
};
```

### Практические примеры: валидация, парсинг, state machines

```csharp
// --- Валидация ---
public static Result<CreateOrderCommand> Validate(CreateOrderCommand cmd) => cmd switch
{
    { CustomerId: <= 0 }       => Result<CreateOrderCommand>.Failure("Невалидный CustomerId"),
    { Items: [] }              => Result<CreateOrderCommand>.Failure("Заказ без товаров"),
    { Items: [.., { Qty: <= 0 }] } => Result<CreateOrderCommand>.Failure("Количество <= 0"),
    _                          => Result<CreateOrderCommand>.Success(cmd)
};

// --- Парсинг CLI-аргументов ---
static CliOptions ParseArgs(string[] args) => args switch
{
    ["-v" or "--version"]            => new(ShowVersion: true),
    ["-h" or "--help"]               => new(ShowHelp: true),
    ["run", var file]                => new(Command: "run", File: file),
    ["run", var file, "--watch"]     => new(Command: "run", File: file, Watch: true),
    ["build", "--release"]           => new(Command: "build", Release: true),
    []                               => CliOptions.Default,
    _                                => throw new ArgumentException($"Неизвестные аргументы: {string.Join(' ', args)}")
};

// --- State Machine ---
public static OrderStatus NextState(OrderStatus current, OrderEvent @event) =>
    (current, @event) switch
    {
        (OrderStatus.New, OrderEvent.Pay)               => OrderStatus.Processing,
        (OrderStatus.Processing, OrderEvent.Ship)       => OrderStatus.Shipped,
        (OrderStatus.Shipped, OrderEvent.Deliver)       => OrderStatus.Delivered,
        (OrderStatus.New, OrderEvent.Cancel)             => OrderStatus.Cancelled,
        (OrderStatus.Processing, OrderEvent.Cancel)      => OrderStatus.Cancelled,
        _ => throw new InvalidOperationException(
            $"Невалидный переход: {current} + {@event}")
    };
```

> [!question]- **Интервью: Switch expression — exhaustive matching?**
> Компилятор проверяет полноту для enum, bool, sealed hierarchy. Для открытых типов — `_` (discard). Warning при неполном покрытии. В production: всегда `_` с throw для защиты от новых значений.

---

## Nullable Reference Types (C# 8)

### Включение

```xml
<!-- В .csproj -->
<PropertyGroup>
    <Nullable>enable</Nullable>
</PropertyGroup>
```

```csharp
// Или на уровне файла
#nullable enable
```

### Аннотация `?` — nullable vs non-nullable

```csharp
// non-nullable — компилятор гарантирует, что null не будет
string name = "Иван";
// name = null; // CS8600 warning

// nullable — явно говорим: «здесь может быть null»
string? middleName = null; // OK

// Возврат nullable
public string? FindUserName(int id)
{
    return _users.TryGetValue(id, out var user) ? user.Name : null;
}
```

### Null-forgiving operator `!`

Используем **только** когда точно знаем, что значение не null, но компилятор не может это доказать:

```csharp
// Допустимо: после проверки в другом месте
var item = cache.Get(key); // возвращает T?
Debug.Assert(item is not null);
Process(item!); // мы уже проверили

// Допустимо: EF Core navigation property (заполняется ORM)
public class Order
{
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!; // EF заполнит
}

// ЗАПРЕЩЕНО: слепое подавление warnings
string? x = GetValue();
Console.WriteLine(x!.Length); // опасно — может быть null
```

### Null Guards

```csharp
// .NET 6+ — самый короткий способ
public void Process(string name, Order order)
{
    ArgumentNullException.ThrowIfNull(name);
    ArgumentNullException.ThrowIfNull(order);
    // ...
}

// .NET 7+ — для строк
public void SetName(string name)
{
    ArgumentException.ThrowIfNullOrEmpty(name);
    ArgumentException.ThrowIfNullOrWhiteSpace(name); // .NET 8
}
```

### Операторы `??`, `??=`, `?.`, `?[]`

```csharp
// ?? — null-coalescing
string displayName = user.NickName ?? user.FullName ?? "Аноним";

// ??= — null-coalescing assignment
_cache ??= new Dictionary<string, object>();

// ?. — null-conditional member access
int? length = text?.Length;
string? upper = text?.ToUpper();

// ?[] — null-conditional element access
int? first = numbers?[0];

// Цепочка
string? city = order?.Customer?.Address?.City;

// Комбинация ?. и ??
string city = order?.Customer?.Address?.City ?? "Неизвестно";
```

### Nullable-атрибуты (System.Diagnostics.CodeAnalysis)

```csharp
using System.Diagnostics.CodeAnalysis;

// [NotNull] — после вызова метода параметр гарантированно не null
public void EnsureInitialized([NotNull] ref string? value)
{
    value ??= "default";
}

// [MaybeNull] — generic T может вернуть null
[return: MaybeNull]
public T Find<T>(int id) where T : class
{
    return _store.TryGetValue(id, out var item) ? (T)item : default;
}

// [NotNullWhen] — параметр не null, когда метод возвращает true/false
public bool TryGetUser(int id, [NotNullWhen(true)] out User? user)
{
    return _users.TryGetValue(id, out user);
}

// [MemberNotNull] — после вызова метода поле гарантированно не null
[MemberNotNull(nameof(_connection))]
private void EnsureConnected()
{
    _connection ??= CreateConnection();
}

// [AllowNull] — можно передать null, даже если тип non-nullable
public string Title
{
    get => _title;
    [param: AllowNull]
    set => _title = value ?? "Без названия";
}
```

### Лучшие практики

```csharp
// 1. Всегда включать <Nullable>enable</Nullable> в новых проектах.
// 2. Стремиться к нулю nullable warnings.
// 3. Не злоупотреблять ! — каждое использование = потенциальный баг.
// 4. Для DTO/API models — string? для опциональных полей.
// 5. Для domain models — string (non-nullable), инварианты в конструкторе.
// 6. Использовать [NotNullWhen], [MaybeNull] для точного контракта API.
```

> [!question]- **Интервью: Nullable Reference Types — как включить и что даёт?**
> `<Nullable>enable</Nullable>` в csproj. Компилятор предупреждает при присвоении null в non-nullable, разыменовании без проверки. Это **только аннотации** — в runtime проверок нет.
>
> **Атрибуты:** `[NotNull]`, `[MaybeNull]`, `[NotNullWhen(true)]` — тонкий контроль flow analysis.

---

## Records (C# 9–10)

### record class — reference type (C# 9)

```csharp
// Positional syntax — самый частый вариант
public record Person(string Name, int Age);

// Компилятор генерирует:
// - Конструктор Person(string, int)
// - Свойства Name { get; init; } и Age { get; init; }
// - Deconstruct(out string, out int)
// - Value equality (Equals, GetHashCode, ==, !=)
// - ToString() → "Person { Name = Иван, Age = 30 }"
// - with-expression support

// Расширенная запись с телом
public record Person(string Name, int Age)
{
    // Дополнительное свойство
    public string DisplayName => $"{Name} ({Age})";

    // Переопределение ToString через PrintMembers
    protected virtual bool PrintMembers(StringBuilder sb)
    {
        sb.Append($"Name = {Name}, Age = {Age}");
        return true;
    }
}
```

### record struct — value type (C# 10)

```csharp
// По умолчанию — mutable!
public record struct Coordinate(double Lat, double Lon);

// Для immutability — добавляем readonly
public readonly record struct Money(decimal Amount, string Currency);

var m = new Money(100m, "RUB");
// m.Amount = 200m; // CS8852 — readonly
```

### with-expressions

```csharp
var person = new Person("Иван", 30);
var older = person with { Age = 31 };

Console.WriteLine(person); // Person { Name = Иван, Age = 30 }
Console.WriteLine(older);  // Person { Name = Иван, Age = 31 }
Console.WriteLine(person == older); // False

// Deep copy с одинаковыми значениями
var copy = person with { };
Console.WriteLine(person == copy);           // True (value equality)
Console.WriteLine(ReferenceEquals(person, copy)); // False (разные объекты)
```

### Value Equality

```csharp
var p1 = new Person("Иван", 30);
var p2 = new Person("Иван", 30);

Console.WriteLine(p1 == p2);      // True — value equality
Console.WriteLine(p1.Equals(p2)); // True
Console.WriteLine(ReferenceEquals(p1, p2)); // False

// class — reference equality по умолчанию
// record — value equality по умолчанию
```

### Inheritance с records

```csharp
public record Animal(string Name);
public record Dog(string Name, string Breed) : Animal(Name);

var dog = new Dog("Шарик", "Лабрадор");
Animal animal = dog;

// Equality учитывает runtime-тип (EqualityContract)
var animal2 = new Animal("Шарик");
Console.WriteLine(dog == animal2); // False — разные типы

// ОГРАНИЧЕНИЯ:
// - record class может наследоваться только от record class
// - record struct не поддерживает наследование (как любой struct)
// - sealed record — запрет наследования
public sealed record Immutable(string Value);
```

### record vs class vs struct — таблица сравнения

```
| Аспект              | class          | record class   | struct         | record struct     |
|---------------------|----------------|----------------|----------------|-------------------|
| Тип                 | Reference      | Reference      | Value          | Value             |
| Equality            | Reference      | Value          | Value          | Value             |
| Immutable           | Нет            | По умолчанию   | Нет            | Нет (readonly да) |
| Inheritance         | Да             | Да (records)   | Нет            | Нет               |
| with-expression     | Нет            | Да             | Нет            | Да                |
| Deconstruct         | Ручной         | Авто           | Ручной         | Авто              |
| ToString            | Тип            | Все свойства   | Тип            | Все свойства      |
| Heap/Stack          | Heap           | Heap           | Stack*         | Stack*            |
```

> [!question]- **Интервью: record class vs record struct — различия?**
> **record class** — reference (heap), value equality, наследование, `with` копирует объект. Для DTO, API-моделей.
>
> **record struct** — value (stack), value equality, без наследования. Для маленьких value-объектов в hot path. `readonly record struct` — предпочтительная форма.

> [!question]- **Интервью: Ковариантность и контравариантность?**
> **Ковариантность (`out T`):** `IEnumerable<Cat>` → `IEnumerable<Animal>`. Только чтение.
>
> **Контравариантность (`in T`):** `IComparer<Animal>` → `IComparer<Cat>`. Только приём.
>
> **Инвариантность:** `List<Cat>` НЕ → `List<Animal>` — List и пишет, и читает.

---

## Init-only и Required (C# 9, 11)

### init accessor (C# 9)

```csharp
public class UserDto
{
    public string Name { get; init; }    // можно задать только при инициализации
    public string Email { get; init; }
    public int Age { get; init; }
}

var user = new UserDto { Name = "Иван", Email = "ivan@mail.ru", Age = 30 };
// user.Name = "Пётр"; // CS8852 — init-only
```

### required modifier (C# 11)

```csharp
public class CreateOrderDto
{
    public required string ProductName { get; init; }
    public required int Quantity { get; init; }
    public required decimal Price { get; init; }
    public string? Notes { get; init; } // опциональное
}

// Компилятор заставит указать все required свойства:
var dto = new CreateOrderDto
{
    ProductName = "Клавиатура",
    Quantity = 1,
    Price = 5000m
};

// var bad = new CreateOrderDto { ProductName = "Мышь" };
// CS9035 — Quantity и Price обязательны
```

### SetsRequiredMembers

```csharp
public class OrderEntity
{
    public required int Id { get; init; }
    public required string Name { get; init; }

    // Конструктор, который заполняет все required members
    [System.Diagnostics.CodeAnalysis.SetsRequiredMembers]
    public OrderEntity(int id, string name)
    {
        Id = id;
        Name = name;
    }

    // Параметрless constructor для EF Core — без SetsRequiredMembers
    public OrderEntity() { }
}

// Оба варианта компилируются:
var a = new OrderEntity(1, "Test");
var b = new OrderEntity { Id = 2, Name = "Test2" };
```

### Идеальный DTO: required + init

```csharp
// Паттерн для API DTOs — обязательные поля, иммутабельность
public sealed class CreateUserRequest
{
    public required string Email { get; init; }
    public required string Password { get; init; }
    public required string DisplayName { get; init; }
    public string? AvatarUrl { get; init; }
    public string? Bio { get; init; }
}
```

---

## Primary Constructors (C# 12)

### Базовый синтаксис

```csharp
// До C# 12 — явный конструктор + поля
public sealed class OrderService
{
    private readonly IOrderRepository _orderRepo;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IOrderRepository orderRepo, ILogger<OrderService> logger)
    {
        _orderRepo = orderRepo;
        _logger = logger;
    }

    public async Task<Order> GetAsync(int id)
    {
        _logger.LogInformation("Получение заказа {Id}", id);
        return await _orderRepo.GetByIdAsync(id);
    }
}

// C# 12 — primary constructor
public sealed class OrderService(
    IOrderRepository orderRepo,
    ILogger<OrderService> logger)
{
    public async Task<Order> GetAsync(int id)
    {
        logger.LogInformation("Получение заказа {Id}", id);
        return await orderRepo.GetByIdAsync(id);
    }
}
```

### Захват параметров — как работает

```csharp
// Параметры primary constructor захватываются как скрытые поля.
// Они НЕ являются readonly — это важно!

public class Counter(int initial)
{
    // initial — mutable capture, можно менять
    public int Increment() => ++initial;
    public int Value => initial;
}

var c = new Counter(0);
c.Increment(); // 1
c.Increment(); // 2
```

### Когда создавать явное поле

```csharp
public sealed class UserService(
    IUserRepository userRepo,
    ILogger<UserService> logger)
{
    // Если нужен readonly — создаём поле явно
    private readonly IUserRepository _userRepo = userRepo;

    // Не обращайтесь к `userRepo` после присвоения в поле —
    // это два разных хранилища!
    public async Task<User?> FindAsync(int id)
    {
        logger.LogInformation("Поиск пользователя {Id}", id);
        return await _userRepo.GetByIdAsync(id);
    }
}
```

### DI с primary constructors — рекомендуемый стиль

```csharp
public sealed class CreateOrderCommandHandler(
    IOrderRepository orderRepo,
    IUnitOfWork unitOfWork,
    ILogger<CreateOrderCommandHandler> logger)
    : IRequestHandler<CreateOrderCommand, Result<int>>
{
    public async Task<Result<int>> Handle(
        CreateOrderCommand request,
        CancellationToken ct)
    {
        logger.LogInformation("Создание заказа для клиента {CustomerId}", request.CustomerId);

        var order = Order.Create(request.CustomerId, request.Items);
        orderRepo.Add(order);
        await unitOfWork.SaveChangesAsync(ct).ConfigureAwait(false);

        return Result<int>.Success(order.Id);
    }
}
```

### Gotchas

```csharp
// 1. Mutable capture — параметры НЕ readonly
public class Danger(string name)
{
    public void Mutate()
    {
        name = "Changed!"; // Компилируется! Нет ошибки.
    }
}

// 2. Struct — primary constructor параметры тоже mutable
public struct Point(double x, double y)
{
    // x и y можно менять внутри методов
}

// 3. Не смешивайте capture и field assignment
public class Bad(IService service)
{
    private readonly IService _service = service;

    public void Do()
    {
        // service и _service — ДВА разных хранилища!
        // Используйте только _service.
        _service.Execute();
    }
}
```

---

## Collection Expressions (C# 12)

### Базовый синтаксис

```csharp
// Array
int[] numbers = [1, 2, 3, 4, 5];

// List<T>
List<string> names = ["Иван", "Пётр", "Анна"];

// Span<T>
Span<int> span = [10, 20, 30];

// ReadOnlySpan<T>
ReadOnlySpan<byte> bytes = [0xFF, 0x00, 0xAB];

// ImmutableArray<T>
ImmutableArray<int> immutable = [1, 2, 3];

// Empty collection
List<int> empty = [];
int[] emptyArr = [];
```

### Spread operator `..`

```csharp
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];

// Объединение коллекций
int[] all = [..first, ..second];         // [1, 2, 3, 4, 5, 6]
int[] withExtra = [0, ..first, 99];      // [0, 1, 2, 3, 99]

// Spread из любого IEnumerable
IEnumerable<int> query = Enumerable.Range(1, 5);
int[] fromQuery = [..query, 100];        // [1, 2, 3, 4, 5, 100]

// Spread в List
List<string> combined = [..existingList, "новый элемент"];
```

### Target Typing

```csharp
// Компилятор выбирает тип коллекции по контексту
void Process(ReadOnlySpan<int> data) { /* ... */ }
Process([1, 2, 3]); // автоматически создаёт ReadOnlySpan<int>

// Возврат из метода
public IReadOnlyList<string> GetDefaults() => ["default1", "default2"];

// Тернарный оператор — оба варианта должны быть collection expression
int[] result = condition ? [1, 2] : [3, 4];
```

---

## Global и File-scoped

### File-scoped Namespaces (C# 10)

```csharp
// До C# 10 — block-scoped
namespace MyApp.Services
{
    public class UserService
    {
        // ...
    }
}

// C# 10 — file-scoped (экономит один уровень отступа)
namespace MyApp.Services;

public class UserService
{
    // ...
}
```

### Global Usings (C# 10)

```csharp
// В отдельном файле GlobalUsings.cs
global using System.Collections.Immutable;
global using System.Text.Json;
global using Microsoft.Extensions.Logging;
global using MediatR;

// В .csproj — implicit usings
// <ImplicitUsings>enable</ImplicitUsings>
// Автоматически добавляет: System, System.Linq, System.Collections.Generic и т.д.
```

```xml
<!-- Или явно в csproj -->
<ItemGroup>
    <Using Include="System.Text.Json" />
    <Using Include="Microsoft.Extensions.Logging" />
    <Using Include="MyApp.Domain.Common" />
    <!-- Static using -->
    <Using Include="System.Math" Static="true" />
    <!-- Alias -->
    <Using Include="System.Text.Json.JsonSerializer" Alias="Json" />
</ItemGroup>
```

### File-scoped Types (C# 11)

```csharp
// Тип виден только внутри файла — не экспортируется из assembly
file class InternalHelper
{
    public static int Calculate(int x) => x * 2;
}

file record struct TempResult(bool Success, string Message);

file interface ILocalProcessor
{
    void Process();
}

// Использование: source generators, вспомогательные типы,
// которые не должны утекать в публичный API.
```

---

## Raw String Literals (C# 11)

### Multi-line строки

```csharp
// Минимум три кавычки """, можно больше при необходимости
string json = """
    {
        "name": "Иван",
        "age": 30,
        "address": {
            "city": "Москва"
        }
    }
    """;
// Отступ trimming — отступ закрывающих """ определяет базовый уровень.
// Всё, что левее закрывающих """ — ошибка компиляции.

// SQL-запрос
string sql = """
    SELECT u.Id, u.Name, o.Total
    FROM Users u
    INNER JOIN Orders o ON o.UserId = u.Id
    WHERE u.IsActive = true
    ORDER BY o.Total DESC
    LIMIT 10
    """;
```

### Interpolated raw strings

```csharp
string name = "Иван";
int age = 30;

// $ перед """ — интерполяция
string json = $"""
    {{
        "name": "{name}",
        "age": {age}
    }}
    """;
// {{ }} — экранирование фигурных скобок (удваиваем)

// $$ — два доллара, тогда интерполяция через {{ }}
string template = $$"""
    {
        "name": "{{name}}",
        "jsonTemplate": "{ "key": "{value}" }"
    }
    """;
// Одиночные { } — литеральные, для интерполяции нужны {{ }}
```

---

## UTF-8 String Literals (C# 11)

```csharp
// Суффикс u8 — создаёт ReadOnlySpan<byte> в UTF-8
ReadOnlySpan<byte> greeting = "Hello, World!"u8;

// Зачем: zero-allocation при работе с HTTP, JSON, протоколами
// До C# 11:
byte[] header = Encoding.UTF8.GetBytes("Content-Type"); // аллокация

// C# 11:
ReadOnlySpan<byte> header = "Content-Type"u8; // compile-time, zero-alloc

// Пример: HTTP header
static ReadOnlySpan<byte> ContentTypeJson => "application/json"u8;
static ReadOnlySpan<byte> AuthorizationHeader => "Authorization"u8;

// Сравнение
ReadOnlySpan<byte> input = GetHeaderValue();
if (input.SequenceEqual("Bearer"u8))
{
    // ...
}
```

---

## Generic Math (C# 11)

### Интерфейсы для generic-арифметики

```csharp
// INumber<T> — базовый интерфейс для числовых операций
// IAdditionOperators, IMultiplyOperators, IComparisonOperators и т.д.

// Generic метод суммирования — работает с int, double, decimal и т.д.
public static T Sum<T>(ReadOnlySpan<T> values) where T : INumber<T>
{
    T result = T.Zero;
    foreach (var value in values)
    {
        result += value;
    }
    return result;
}

// Использование
int intSum = Sum<int>([1, 2, 3, 4, 5]);            // 15
double doubleSum = Sum<double>([1.5, 2.5, 3.0]);   // 7.0
decimal decSum = Sum<decimal>([10.5m, 20.3m]);      // 30.8

// Generic среднее
public static T Average<T>(ReadOnlySpan<T> values)
    where T : INumber<T>
{
    T sum = Sum(values);
    return sum / T.CreateChecked(values.Length);
}

// Generic clamp
public static T Clamp<T>(T value, T min, T max)
    where T : INumber<T>
{
    if (value < min) return min;
    if (value > max) return max;
    return value;
}

// Static abstract members в интерфейсах (C# 11)
public interface IFactory<TSelf> where TSelf : IFactory<TSelf>
{
    static abstract TSelf Create();
    static abstract TSelf Default { get; }
}
```

---

## Params Collections (C# 13)

```csharp
// До C# 13: params только с массивами
public void LogOld(params string[] messages) { /* аллокация массива */ }

// C# 13: params работает с Span, ReadOnlySpan, IEnumerable
public void Log(params ReadOnlySpan<string> messages)
{
    foreach (var msg in messages)
    {
        Console.WriteLine(msg);
    }
}
// Вызов — zero-allocation на стеке!
Log("info", "Запуск приложения", "OK");

// params с IEnumerable<T>
public int Sum(params IEnumerable<int> numbers) => numbers.Sum();

// params с List<T>
public void Process(params List<string> items)
{
    foreach (var item in items) { /* ... */ }
}

// Разрешение перегрузок: ReadOnlySpan > Span > Array > IEnumerable
```

---

## Lock Object (C# 13)

```csharp
// До C# 13 — lock на object
private readonly object _syncRoot = new();

public void ThreadSafeMethod()
{
    lock (_syncRoot)
    {
        // critical section
    }
}

// C# 13 — System.Threading.Lock
// Более эффективная реализация, заточенная под lock
private readonly Lock _lock = new();

public void ThreadSafeMethod()
{
    lock (_lock) // компилятор использует Lock.EnterScope()
    {
        // critical section
    }
}

// Ручное управление (Scope — ref struct, Dispose вызывает Exit)
public void ManualLock()
{
    using (_lock.EnterScope())
    {
        // critical section
    }
}

// Преимущества Lock vs object:
// 1. Нет boxing/unboxing
// 2. Компилятор генерирует более оптимальный код
// 3. Семантически ясный тип — понятно назначение
// 4. Ref struct scope — не может утечь в async контекст
```

---

## C# 14 Preview

### Extension Members

```csharp
// До C# 14 — только extension methods (static class)
public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string? s) => string.IsNullOrEmpty(s);
}

// C# 14 — extension блок: properties, indexers, static members
public extension StringExtensions for string
{
    // Extension property
    public bool IsEmpty => this.Length == 0;

    // Extension indexer
    public char FromEnd[int index] => this[^index];

    // Extension static method
    public static string Empty => string.Empty;
}

// Использование
string name = "Hello";
bool empty = name.IsEmpty;       // false
char last = name.FromEnd[1];     // 'o'
```

### field keyword в auto-properties

```csharp
// До C# 14 — нужно backing field для валидации
private string _name = "";
public string Name
{
    get => _name;
    set
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(value);
        _name = value;
    }
}

// C# 14 — field keyword ссылается на автоматическое backing field
public string Name
{
    get => field;
    set
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(value);
        field = value;
    }
}

// Удобно для lazy initialization
public List<Order> Orders
{
    get => field ??= [];
}

// Удобно для change notification (INotifyPropertyChanged)
public string Title
{
    get => field;
    set => SetProperty(ref field, value);
}
```

### Null-conditional Assignment

```csharp
// До C# 14
if (customer?.Address is not null)
{
    customer.Address.City = "Москва";
}

// C# 14 — null-conditional assignment
customer?.Address?.City = "Москва";
// Если customer == null ИЛИ Address == null — присвоение не выполняется.
// NullReferenceException не возникает. Без второго ?. при Address == null будет NRE!

// Работает и с индексаторами
list?[0] = newValue;
dictionary?["key"] = newValue;
```

### Lambda Parameter Modifiers

```csharp
// C# 14 — можно использовать ref, in, out в лямбдах с явными типами
Span<int> numbers = [3, 1, 4, 1, 5];
numbers.Sort((ref int a, ref int b) => a.CompareTo(b));
```

### Partial Constructors и Events (дополнение к partial methods/properties)

```csharp
// Partial constructor — часть реализации в одном файле, часть в другом
public partial class ViewModel
{
    public partial ViewModel(string title);
}

// В другом файле (например, generated code)
public partial class ViewModel
{
    public partial ViewModel(string title)
    {
        Title = title;
        InitializeCommands();
    }
}
```

---

## Дополнительные фичи по версиям

### Using Declarations (C# 8)

```csharp
// До C# 8 — using block
using (var stream = File.OpenRead("file.txt"))
{
    // ...
} // Dispose здесь

// C# 8 — using declaration (без блока)
using var stream = File.OpenRead("file.txt");
// ... используем stream
// Dispose при выходе из scope (метод, блок if/for и т.д.)
```

### Async Streams (C# 8)

```csharp
// IAsyncEnumerable<T> — асинхронная итерация
public async IAsyncEnumerable<Order> GetOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    int page = 0;
    while (true)
    {
        var batch = await _repo.GetPageAsync(page++, 100, ct).ConfigureAwait(false);
        if (batch.Count == 0) yield break;

        foreach (var order in batch)
        {
            yield return order;
        }
    }
}

// Потребление
await foreach (var order in GetOrdersAsync(ct))
{
    Process(order);
}
```

### Target-typed new (C# 9)

```csharp
// Тип выводится из контекста
Dictionary<string, List<int>> map = new();
List<string> names = new() { "Иван", "Пётр" };

// В аргументах метода
Process(new OrderOptions { Timeout = TimeSpan.FromSeconds(30) });

// В return
public CancellationTokenSource CreateCts() => new(TimeSpan.FromMinutes(5));
```

### Top-level Statements (C# 9)

```csharp
// Файл без класса Program и метода Main
using Microsoft.Extensions.Hosting;

var builder = Host.CreateDefaultBuilder(args);
// ...
var app = builder.Build();
await app.RunAsync();
```

### Constant Interpolated Strings (C# 10)

```csharp
// Интерполяция в const — если все части const
const string Scheme = "https";
const string Host = "api.example.com";
const string BaseUrl = $"{Scheme}://{Host}"; // OK в C# 10
```

### Alias Any Type (C# 12)

```csharp
// using alias для любого типа, включая tuples и generics
using Point = (double X, double Y);
using OrderId = int;
using JsonDict = System.Collections.Generic.Dictionary<string, System.Text.Json.JsonElement>;

Point origin = (0, 0);
OrderId id = 42;
JsonDict data = new() { ["key"] = JsonDocument.Parse("1").RootElement };
```

### Default Lambda Parameters (C# 12)

```csharp
// Параметры по умолчанию в лямбдах
var greet = (string name, string greeting = "Привет") => $"{greeting}, {name}!";
Console.WriteLine(greet("Иван"));            // Привет, Иван!
Console.WriteLine(greet("Иван", "Здравствуй")); // Здравствуй, Иван!
```

### ref struct Interfaces (C# 13)

```csharp
// ref struct теперь может реализовывать интерфейсы (с ограничениями)
public interface IBufferWriter
{
    void Write(ReadOnlySpan<byte> data);
}

public ref struct StackBuffer : IBufferWriter
{
    private Span<byte> _buffer;
    private int _position;

    public void Write(ReadOnlySpan<byte> data)
    {
        data.CopyTo(_buffer[_position..]);
        _position += data.Length;
    }
}

// Ограничение: нельзя использовать через boxing (IBufferWriter переменная)
// Только через generics с allows ref struct constraint
public static void WriteAll<T>(T writer, ReadOnlySpan<byte> data)
    where T : IBufferWriter, allows ref struct
{
    writer.Write(data);
}
```

### Escape Sequence `\e` (C# 13)

```csharp
// \e = ESC (0x1B) — для ANSI escape codes в терминале
Console.WriteLine("\e[31mКрасный текст\e[0m");
Console.WriteLine("\e[1;32mЖирный зелёный\e[0m");

// До C# 13: '\x1B' или '\u001B'
```

---

## Таблица фич по версиям

| Версия | Год  | .NET   | Ключевые фичи                                                                    |
|--------|------|--------|-----------------------------------------------------------------------------------|
| C# 8   | 2019 | Core 3 | Nullable refs, pattern matching, default interface methods, using declarations, async streams, indices/ranges |
| C# 9   | 2020 | 5      | Records, init-only, top-level statements, target-typed new, logical patterns      |
| C# 10  | 2021 | 6      | Global usings, file-scoped namespaces, record structs, const interpolation, extended property patterns |
| C# 11  | 2022 | 7      | Raw strings, required, list patterns, UTF-8 literals, generic math, file-scoped types, static abstract members |
| C# 12  | 2023 | 8      | Primary constructors, collection expressions, default lambda params, alias any type, inline arrays |
| C# 13  | 2024 | 9      | params collections, Lock type, ref struct interfaces, escape `\e`, method group natural type improvements |
| C# 14  | 2025 | 10     | Extension members, field keyword, null-conditional assignment, partial constructors |

---

## См. также

- [Типы и память](types-and-memory.md)
- [ООП и классы](oop.md)
- [Delegates и Events](delegates-events.md)
- [Collections и LINQ](collections-linq.md)
- [Async и потоки](async-threading.md)
