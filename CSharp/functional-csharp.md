---
tags: [csharp, functional, immutability, monads, records, pattern-matching, railway-oriented, languageext]
level: Senior
date: 2026-04-30
---

# Functional C#

> Полный гайд по функциональному стилю в C#. Применим в любом C#-проекте — ASP.NET, console, Unity, mobile. Закрывает: immutability через records, pattern matching evolution (8→13), expressions vs statements, pure functions, function composition, Result/Option/Either monads, railway-oriented programming, LanguageExt библиотека, F# для C# devs, performance trade-offs.

---

## Что это, зачем и когда

### Что такое функциональный стиль?

**Подход где данные — immutable, функции — без side effects, и программа = композиция чистых преобразований.**

| | Imperative (классический OOP) | Functional |
|--|------------------------------|-----------|
| Состояние | Mutable, меняется через методы | Immutable, новый объект каждый раз |
| Side effects | Везде (DB, logs, mutations) | Изолированы на границе системы |
| Control flow | if/else, for, throw | Pattern matching, expressions |
| Data flow | `obj.SetX(); obj.SetY()` | `obj with { X = 1, Y = 2 }` |
| Error handling | Exceptions | Result\<T, E\> / Either |
| Null | `null` reference | Option\<T\> / Maybe\<T\> |

**Аналогия:** Imperative — это рецепт типа "положи в кастрюлю, нагрей, помешай, добавь, посоли". Functional — это конвейер: `ингредиенты → cleanup() → mix() → cook() → garnish() → блюдо`. Каждая стадия принимает input, возвращает output, не меняет input.

### Зачем functional стиль?

| Imperative | Functional |
|-----------|-----------|
| `if (user != null && user.IsActive) { ... }` повсюду | Option\<User\>: `user.IfActive().Map(...)` явно |
| Race conditions от shared mutable state | Immutable → thread-safe by default |
| Exceptions ломают control flow незаметно | Result\<T\> явно в сигнатуре, нельзя забыть |
| Тестировать сложно — много setup state | Pure functions: input → output, легко тестировать |
| Что вернёт метод? Надо читать тело | Сигнатура говорит всё: `Option<User>` = может не быть |

### Когда applicable

✅ **Хорошо для:**
- Доменная логика (ядро бизнес-правил) — pure, predictable
- Data transformations / ETL pipelines
- LINQ queries (уже functional)
- Concurrent / parallel — immutable безопасно
- Event sourcing — events immutable by design
- DDD value objects — natural fit
- Validation chains
- Configuration / settings объекты

❌ **Сложно для:**
- Heavy I/O (UI, networking) — нужны side effects
- Performance hot paths — copying immutable data дорого
- Полностью переписать legacy mutable codebase — слишком дорого

### Functional vs OOP — не противопоставление

C# — **multi-paradigm**. Лучшие codebases смешивают:
- OOP для structure (классы, encapsulation, DI)
- Functional для data (records, immutability, pattern matching, LINQ)

```csharp
// Класс — для structure и DI
public class OrderProcessor(IOrderRepository repo, ILogger<OrderProcessor> log)
{
    // Pure functional core
    public Result<ProcessedOrder> Process(Order order) =>
        ValidateOrder(order)
            .Bind(CalculatePrice)
            .Bind(ApplyDiscounts)
            .Map(ToProcessed);
    
    // OOP wrapper для I/O
    public async Task<Result<ProcessedOrder>> ProcessAsync(Guid id)
    {
        var order = await repo.GetAsync(id);
        return order is null
            ? Result.Fail("Not found")
            : Process(order);
    }
}
```

---

## 1. Immutability через Records (C# 9+)

### Record vs Class

```csharp
// ❌ Mutable class — изменяется
public class Address
{
    public string Street { get; set; } = "";
    public string City { get; set; } = "";
}

var addr = new Address { Street = "Main", City = "NYC" };
addr.Street = "Broadway";  // mutation — два объекта ссылаются на один

// ✅ Record — immutable + value equality
public record Address(string Street, string City);

var addr1 = new Address("Main", "NYC");
var addr2 = addr1 with { Street = "Broadway" };  // новый объект
// addr1 нетронут!

// Value equality
new Address("Main", "NYC") == new Address("Main", "NYC");  // true!
```

### Record types — три варианта

```csharp
// Positional record — самый компактный
public record Person(string Name, int Age);

// Property-based record
public record User
{
    public string Name { get; init; } = "";
    public int Age { get; init; }
}

// Mutable record — `record` с set'ами (anti-pattern, но возможно)
public record Mutable
{
    public string Name { get; set; } = "";  // не functional!
}

// Record struct (C# 10+) — value type
public record struct Point(int X, int Y);
// Stack-allocated, no GC
```

### Inheritance в records

```csharp
public abstract record Shape;
public record Circle(double Radius) : Shape;
public record Rectangle(double Width, double Height) : Shape;
public record Square(double Side) : Rectangle(Side, Side);

// Pattern matching работает идеально
double Area(Shape shape) => shape switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Square s => s.Side * s.Side,
    Rectangle r => r.Width * r.Height,
    _ => throw new ArgumentException()
};
```

### `init` setter — write-once

```csharp
public class User
{
    public string Email { get; init; } = "";  // только в object initializer / ctor
    public DateTime CreatedAt { get; init; }
}

var u = new User { Email = "a@b.com" };  // OK
u.Email = "c@d.com";  // ❌ compile error — init only

// `with` создаёт новый объект
var u2 = u with { Email = "c@d.com" };  // OK, новый instance
```

### Глубокая immutability — collections

```csharp
// ❌ Псевдо-immutable — список mutable
public record Order(string Id, List<OrderItem> Items);

var order = new Order("1", new List<OrderItem>());
order.Items.Add(new OrderItem());  // ⚠️ mutate list — обходим immutability!

// ✅ ImmutableList / ImmutableArray
public record Order(string Id, ImmutableList<OrderItem> Items);

var order = new Order("1", ImmutableList<OrderItem>.Empty);
var newOrder = order with { Items = order.Items.Add(new OrderItem()) };

// ✅ Read-only collection
public record Order(string Id, IReadOnlyList<OrderItem> Items);
```

См. [Collections и LINQ](collections-linq.md) — ImmutableArray vs ImmutableList performance.

---

## 2. Pattern Matching — evolution C# 8 → 13

### C# 8 — switch expression

```csharp
// Old way (statement)
public string GetDayType(DayOfWeek day)
{
    switch (day)
    {
        case DayOfWeek.Saturday:
        case DayOfWeek.Sunday:
            return "Weekend";
        default:
            return "Weekday";
    }
}

// C# 8 — expression
public string GetDayType(DayOfWeek day) => day switch
{
    DayOfWeek.Saturday or DayOfWeek.Sunday => "Weekend",
    _ => "Weekday"
};
```

### C# 9 — pattern combinators (and / or / not)

```csharp
public string GetTemperatureStatus(int temp) => temp switch
{
    < 0 => "Freezing",
    >= 0 and < 15 => "Cold",
    >= 15 and < 25 => "Comfortable",
    >= 25 and < 35 => "Hot",
    >= 35 => "Extreme heat"
};

// Type pattern с logical
public bool IsValidPayment(object payment) => payment switch
{
    Cash { Amount: > 0 } => true,
    CreditCard { IsExpired: false, IsBlocked: false } => true,
    null or { } => false,
    _ => false
};
```

### C# 10 — extended property patterns

```csharp
public record Person(string Name, Address Address);
public record Address(string City, string Country);

// C# 10+ — nested без вложенных скобок
public bool IsFromMoscow(Person p) => p is { Address.City: "Moscow" };
public bool IsRussian(Person p) => p is { Address.Country: "Russia" };

// Combined
public bool IsMoscowResident(Person p) => 
    p is { Address: { City: "Moscow", Country: "Russia" } };
// или короче
public bool IsMoscowResident(Person p) => 
    p is { Address.City: "Moscow", Address.Country: "Russia" };
```

### C# 11 — list patterns

```csharp
public string DescribeArray(int[] arr) => arr switch
{
    [] => "empty",
    [var single] => $"single element: {single}",
    [_, _] => "two elements",
    [var first, .., var last] => $"first {first}, last {last}",
    [1, 2, 3] => "exactly 1, 2, 3",
    [1, ..] => "starts with 1",
    [.., 99] => "ends with 99",
    _ => "other"
};

// Slice pattern
public bool IsValidCommand(string[] args) => args switch
{
    ["help"] or ["help", _] => true,
    ["add", var name, ..] when !string.IsNullOrEmpty(name) => true,
    _ => false
};
```

### C# 13 — params для collections, primary ctors patterns

```csharp
public record Range(int Start, int End)
{
    public bool Contains(int value) => value is >= Start and <= End;
}
```

### Real-world: Result handling без exceptions

```csharp
public abstract record Result<T>;
public record Ok<T>(T Value) : Result<T>;
public record Failure<T>(string Error) : Result<T>;

// Использование
Result<User> result = GetUser(id);

string message = result switch
{
    Ok<User>(var user) when user.IsActive => $"Active: {user.Name}",
    Ok<User>(var user) => $"Inactive: {user.Name}",
    Failure<User>(var error) => $"Error: {error}",
    _ => "Unknown"
};
```

---

## 3. Expressions vs Statements

### Expressions — возвращают значение

```csharp
// Statement — нет return value
int x;
if (condition)
    x = 1;
else
    x = 2;

// Expression — есть value
int x = condition ? 1 : 2;

// Switch expression
int x = day switch
{
    "Monday" => 1,
    "Tuesday" => 2,
    _ => 0
};
```

### Expression-bodied members

```csharp
public class Calculator
{
    // Expression-bodied method
    public int Add(int a, int b) => a + b;
    
    // Expression-bodied property
    public bool IsValid => _value > 0 && _value < 100;
    
    // Expression-bodied constructor (C# 7+)
    public Calculator(int value) => _value = value;
    
    // Local function expression
    public int Compute(int x)
    {
        int square(int n) => n * n;
        return square(x) + square(x);
    }
}
```

> [!info] Expression bias
> Functional code тяготеет к expressions — каждая операция возвращает значение, можно chain'ать. Statements — break the flow.

---

## 4. Pure Functions

**Pure** — функция, у которой:
1. Output зависит **только** от input (нет hidden state)
2. **Нет side effects** (не меняет global state, не делает I/O, не throw)

```csharp
// ✅ Pure
public static int Add(int a, int b) => a + b;
public static IEnumerable<int> Squares(IEnumerable<int> nums) => nums.Select(n => n * n);

// ❌ Impure — depends on hidden state
private int _counter = 0;
public int Next() => ++_counter;  // отвечает по-разному при тех же inputs

// ❌ Impure — side effect (logging)
public int Add(int a, int b)
{
    Console.WriteLine("Adding...");  // side effect
    return a + b;
}

// ❌ Impure — throws
public int Divide(int a, int b)
{
    if (b == 0) throw new DivideByZeroException();  // skip control flow
    return a / b;
}
```

### Преимущества pure functions

- **Тестируемость** — нет setup, mock'ов; input → output
- **Concurrency** — нет shared state, безопасно в parallel
- **Memoization** — кешировать результат можно (тот же input = тот же output)
- **Reasoning** — легко понять что делает (не нужно знать всю иерархию)
- **Composition** — pure + pure = pure

### Functional core, imperative shell

Современный pattern: ядро системы — pure, snowboard — imperative (I/O, DB, UI).

```csharp
// Imperative shell — связь с внешним миром
public async Task<Result> ProcessOrderAsync(Guid orderId)
{
    var order = await _repo.GetAsync(orderId);   // I/O
    var customer = await _repo.GetCustomerAsync(order.CustomerId);  // I/O
    
    var result = ProcessOrder(order, customer);  // ← Pure functional core
    
    if (result is Ok<ProcessedOrder>(var processed))
    {
        await _repo.SaveAsync(processed);  // I/O
        await _emailService.SendAsync(processed);  // I/O
    }
    
    return result;
}

// Pure core — testable без mocks
private static Result<ProcessedOrder> ProcessOrder(Order order, Customer customer)
{
    return ValidateOrder(order)
        .Bind(o => CheckCreditLimit(o, customer))
        .Bind(ApplyDiscounts)
        .Map(o => new ProcessedOrder(o));
}
```

См. [Architecture Patterns](../Architecture/patterns.md).

---

## 5. Option / Maybe — вместо null

### Проблема null

```csharp
public User? FindUser(string email) => /* ... */;

// Caller забывает проверить
var user = FindUser("a@b.com");
Console.WriteLine(user.Name);  // 💥 NullReferenceException!

// Nullable reference types помогают, но всё равно null pollution в коде
if (user is not null) { ... }  // ветвление везде
```

### Решение — Option\<T\>

```csharp
public abstract record Option<T>
{
    public abstract Option<TResult> Map<TResult>(Func<T, TResult> f);
    public abstract Option<TResult> Bind<TResult>(Func<T, Option<TResult>> f);
    public abstract T GetOrElse(T defaultValue);
    public abstract bool IsSome { get; }
    public abstract bool IsNone => !IsSome;
}

public record Some<T>(T Value) : Option<T>
{
    public override Option<TResult> Map<TResult>(Func<T, TResult> f) => new Some<TResult>(f(Value));
    public override Option<TResult> Bind<TResult>(Func<T, Option<TResult>> f) => f(Value);
    public override T GetOrElse(T def) => Value;
    public override bool IsSome => true;
}

public record None<T> : Option<T>
{
    public override Option<TResult> Map<TResult>(Func<T, TResult> f) => new None<TResult>();
    public override Option<TResult> Bind<TResult>(Func<T, Option<TResult>> f) => new None<TResult>();
    public override T GetOrElse(T def) => def;
    public override bool IsSome => false;
}

public static class Option
{
    public static Option<T> Some<T>(T value) => new Some<T>(value);
    public static Option<T> None<T>() => new None<T>();
    public static Option<T> FromNullable<T>(T? value) where T : class =>
        value is null ? None<T>() : Some(value);
}
```

### Использование

```csharp
public Option<User> FindUser(string email) => 
    _users.TryGetValue(email, out var user) ? Option.Some(user) : Option.None<User>();

// Composable — никаких null checks
var greeting = FindUser("a@b.com")
    .Map(u => u.Name)
    .Map(name => $"Hello, {name}!")
    .GetOrElse("Hello, stranger!");

// Bind — для chained Option-returning operations
public Option<Address> GetUserAddress(string email) =>
    FindUser(email)
        .Bind(u => GetCustomer(u.Id))
        .Bind(c => GetAddress(c.AddressId));
// Если на любом шаге None — return None, иначе Some(address)
```

### Pattern matching на Option

```csharp
string result = FindUser(email) switch
{
    Some<User>(var u) when u.IsActive => $"Active: {u.Name}",
    Some<User>(var u) => $"Inactive: {u.Name}",
    None<User> => "Not found",
    _ => "Unknown"
};
```

---

## 6. Result\<T, E\> — вместо exceptions

### Проблема exceptions для бизнес-ошибок

```csharp
// ❌ Exception для expected business outcome
public User GetUser(Guid id)
{
    var user = _db.Users.Find(id);
    if (user == null)
        throw new UserNotFoundException();  // expected!
    if (user.IsBlocked)
        throw new UserBlockedException();
    return user;
}
```

Проблемы:
- Дорого (StackTrace allocation)
- Неявно — caller не знает что throw'ит
- Ломает control flow
- Тестировать неудобно (`Assert.Throws<>`)

### Решение — Result\<T, E\>

```csharp
public abstract record Result<T, E>
{
    public abstract Result<TResult, E> Map<TResult>(Func<T, TResult> f);
    public abstract Result<TResult, E> Bind<TResult>(Func<T, Result<TResult, E>> f);
    public abstract Result<T, EResult> MapError<EResult>(Func<E, EResult> f);
    public abstract bool IsSuccess { get; }
}

public record Success<T, E>(T Value) : Result<T, E>
{
    public override Result<TResult, E> Map<TResult>(Func<T, TResult> f) => 
        new Success<TResult, E>(f(Value));
    public override Result<TResult, E> Bind<TResult>(Func<T, Result<TResult, E>> f) => 
        f(Value);
    public override Result<T, EResult> MapError<EResult>(Func<E, EResult> f) => 
        new Success<T, EResult>(Value);
    public override bool IsSuccess => true;
}

public record Failure<T, E>(E Error) : Result<T, E>
{
    public override Result<TResult, E> Map<TResult>(Func<T, TResult> f) => 
        new Failure<TResult, E>(Error);
    public override Result<TResult, E> Bind<TResult>(Func<T, Result<TResult, E>> f) => 
        new Failure<TResult, E>(Error);
    public override Result<T, EResult> MapError<EResult>(Func<E, EResult> f) => 
        new Failure<T, EResult>(f(Error));
    public override bool IsSuccess => false;
}
```

### Domain errors как типы

```csharp
public abstract record UserError;
public record UserNotFound(Guid Id) : UserError;
public record UserBlocked(Guid Id, string Reason) : UserError;
public record EmailAlreadyTaken(string Email) : UserError;

public Result<User, UserError> GetUser(Guid id)
{
    var user = _db.Users.Find(id);
    if (user is null)
        return new Failure<User, UserError>(new UserNotFound(id));
    if (user.IsBlocked)
        return new Failure<User, UserError>(new UserBlocked(id, user.BlockReason));
    return new Success<User, UserError>(user);
}

// Caller обрабатывает явно через pattern matching
return GetUser(id) switch
{
    Success<User, UserError>(var user) => Ok(user),
    Failure<User, UserError>(UserNotFound) => NotFound(),
    Failure<User, UserError>(UserBlocked(_, var reason)) => Forbid(reason),
    _ => Problem()
};
```

См. [Error Handling](error-handling.md) — детальный сравнительный анализ.

---

## 7. Railway-Oriented Programming

Pattern от Scott Wlaschin (F# для funкции, applicable в C#) — chain pure operations через Result.

```
input → validate → enrich → save → output
        ↓failure   ↓failure  ↓failure
        ↓          ↓         ↓
        → → → →  error path  → → → →
```

```csharp
public Result<Order, OrderError> ProcessOrder(OrderRequest req) =>
    Validate(req)
        .Bind(CheckInventory)
        .Bind(CalculatePrice)
        .Bind(ApplyDiscounts)
        .Map(BuildOrder);

// Каждый шаг:
private Result<OrderRequest, OrderError> Validate(OrderRequest req)
{
    if (string.IsNullOrEmpty(req.CustomerId))
        return Result.Fail<OrderRequest, OrderError>(new InvalidCustomer());
    if (req.Items.Count == 0)
        return Result.Fail<OrderRequest, OrderError>(new EmptyCart());
    return Result.Ok<OrderRequest, OrderError>(req);
}

private Result<OrderRequest, OrderError> CheckInventory(OrderRequest req)
{
    foreach (var item in req.Items)
    {
        if (!_inventory.HasStock(item.ProductId, item.Quantity))
            return Result.Fail<OrderRequest, OrderError>(new OutOfStock(item.ProductId));
    }
    return Result.Ok<OrderRequest, OrderError>(req);
}

// и т.д.
```

**Магия:** если на любом шаге Failure — все последующие Bind/Map просто пропускаются. Финальный результат — первая ошибка.

### Async railway

```csharp
public async Task<Result<Order, OrderError>> ProcessAsync(OrderRequest req)
{
    var validated = Validate(req);
    if (validated is Failure<OrderRequest, OrderError> f)
        return f.MapToOrder();
    
    var inventoryChecked = await CheckInventoryAsync(validated.GetValue());
    if (inventoryChecked is Failure<OrderRequest, OrderError> f2)
        return f2.MapToOrder();
    
    // ... messy
}

// Лучше — extension methods для Task<Result>
public static class ResultAsyncExtensions
{
    public static async Task<Result<TResult, E>> BindAsync<T, TResult, E>(
        this Task<Result<T, E>> source,
        Func<T, Task<Result<TResult, E>>> bind)
    {
        var result = await source;
        return result switch
        {
            Success<T, E>(var value) => await bind(value),
            Failure<T, E>(var error) => new Failure<TResult, E>(error),
            _ => throw new ArgumentException()
        };
    }
}

public Task<Result<Order, OrderError>> ProcessAsync(OrderRequest req) =>
    Validate(req).ToTask()
        .BindAsync(CheckInventoryAsync)
        .BindAsync(CalculatePriceAsync)
        .BindAsync(ApplyDiscountsAsync)
        .MapAsync(BuildOrder);
```

---

## 8. Function Composition

```csharp
// Compose: f(g(x))
public static class FunctionExtensions
{
    public static Func<TIn, TOut> Compose<TIn, TMid, TOut>(
        this Func<TIn, TMid> first,
        Func<TMid, TOut> second) => x => second(first(x));
}

// Использование
Func<int, int> doubleIt = x => x * 2;
Func<int, int> addOne = x => x + 1;
Func<int, int> combined = doubleIt.Compose(addOne);  // (x * 2) + 1

combined(5);  // 11
```

### Currying — частичное применение

```csharp
public static Func<TArg1, Func<TArg2, TResult>> Curry<TArg1, TArg2, TResult>(
    this Func<TArg1, TArg2, TResult> f) => 
    arg1 => arg2 => f(arg1, arg2);

// Использование
Func<int, int, int> add = (a, b) => a + b;
var addCurried = add.Curry();
var add5 = addCurried(5);
add5(10);  // 15
add5(20);  // 25
```

> [!info] Когда currying полезно
> Configurable transformations (типа `.Where(predicate)` где predicate частично применённый), composition pipelines, dependency injection без классов.

---

## 9. LINQ как функциональный язык

LINQ — это **уже** functional. C# дал нам Lisp-стиль операторы.

```csharp
// Functional pipeline
var topOrders = orders
    .Where(o => o.Status == "Active")          // filter
    .Select(o => new { o.Id, o.Total })        // map
    .OrderByDescending(o => o.Total)           // sort
    .Take(10)                                   // limit
    .GroupBy(o => o.Total > 1000)              // partition
    .ToDictionary(g => g.Key, g => g.ToList()); // aggregate

// Эквивалент Haskell:
// take 10 . sortBy (Down . total) . map summarize . filter active
```

### Aggregate — fold/reduce

```csharp
var sum = numbers.Aggregate(0, (acc, n) => acc + n);
var product = numbers.Aggregate(1, (acc, n) => acc * n);

// Fold с разными accumulator types
var stats = numbers.Aggregate(
    new { Sum = 0, Count = 0 },
    (acc, n) => new { Sum = acc.Sum + n, Count = acc.Count + 1 });

// LINQ aggregations: Sum, Min, Max, Average, Count — built-in folds
```

### Custom LINQ operators

```csharp
public static class EnumerableExtensions
{
    // Chunked partitioning (.NET 6+ есть в стандарте)
    public static IEnumerable<List<T>> Chunk<T>(this IEnumerable<T> source, int size)
    {
        var chunk = new List<T>(size);
        foreach (var item in source)
        {
            chunk.Add(item);
            if (chunk.Count == size)
            {
                yield return chunk;
                chunk = new List<T>(size);
            }
        }
        if (chunk.Count > 0) yield return chunk;
    }
    
    // Tap — side effect для debugging без break flow
    public static IEnumerable<T> Tap<T>(this IEnumerable<T> source, Action<T> action)
    {
        foreach (var item in source)
        {
            action(item);
            yield return item;
        }
    }
}

// Использование
orders
    .Where(o => o.Total > 100)
    .Tap(o => log.LogDebug("Processing {OrderId}", o.Id))
    .Select(o => o.ToDto())
    .ToList();
```

См. [Collections и LINQ](collections-linq.md).

---

## 10. LanguageExt — production-ready functional

[LanguageExt](https://github.com/louthy/language-ext) — самая мощная FP library для C#.

```bash
dotnet add package LanguageExt.Core
```

### Option (built-in functional)

```csharp
using LanguageExt;
using static LanguageExt.Prelude;

Option<int> someValue = Some(42);
Option<int> noValue = None;

// Map / Bind / Match
var result = someValue
    .Map(x => x * 2)               // Some(84)
    .Bind(x => x > 100 ? Some(x) : None)
    .Match(
        Some: x => $"Got {x}",
        None: () => "Nothing");
```

### Either — Result alternative

```csharp
Either<string, int> Divide(int a, int b) =>
    b == 0 ? Left("Division by zero") : Right(a / b);

var result = Divide(10, 2)
    .Bind(x => Divide(x, 0))
    .Match(
        Right: r => $"Result: {r}",
        Left: e => $"Error: {e}");  // "Error: Division by zero"
```

### Validation — accumulate errors

```csharp
// В отличие от Result, accumulates ВСЕ errors, не только первую
Validation<string, User> ValidateUser(UserDto dto) =>
    (ValidateName(dto.Name), ValidateEmail(dto.Email), ValidateAge(dto.Age))
        .Apply((name, email, age) => new User(name, email, age));

// Все 3 проверки выполняются — список всех ошибок если что-то не так
```

### Immutable Collections

```csharp
using LanguageExt;

Lst<int> list = List(1, 2, 3, 4);  // immutable, persistent
Map<string, int> dict = Map(("a", 1), ("b", 2));
Set<int> set = Set(1, 2, 3);

var newList = list.Add(5);  // новый список, не mutation
list.Count;     // 4 (нетронут)
newList.Count;  // 5
```

### IO Monad — изолируем side effects

```csharp
public class Logger : IIO<Unit>
{
    public IO<Unit> Log(string msg) => 
        IO<Unit>.Lift(() => { Console.WriteLine(msg); return unit; });
}

public IO<int> ProcessOrder(Order order) =>
    from validated in IO<Order>.Pure(Validate(order))
    from logged in Log("Validating...")
    from result in Save(validated)
    select result;

// Не выполняется до .Run() — можно тестировать как чистую функцию
```

> [!warning] LanguageExt — overkill для большинства проектов
> Power tool с steep learning curve. Для team из 1-3 — OK. Для большой команды — может быть барьер. Используй частями (Option, Result) вместо адаптации всего.

---

## 11. F# для C# devs

F# — родной .NET functional language. Полезно даже если не пишешь на нём — учит functional thinking.

### Что даёт C# devs

| F# concept | Применение в C# |
|-----------|-----------------|
| Discriminated Unions | Sealed class hierarchies + records |
| Pattern matching | Switch expressions |
| Computation Expressions | LINQ query syntax |
| Pipe operator `\|>` | Method chaining |
| Type inference | `var`, lambda inference |
| Records | C# 9 records |
| Active Patterns | Custom matching через property patterns |

### F# inspired patterns в C#

```fsharp
// F# discriminated union
type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of a: float * b: float * c: float

let area shape =
    match shape with
    | Circle r -> Math.PI * r * r
    | Rectangle (w, h) -> w * h
    | Triangle (a, b, c) -> 
        let s = (a + b + c) / 2.0
        sqrt (s * (s - a) * (s - b) * (s - c))
```

```csharp
// C# equivalent
public abstract record Shape;
public record Circle(double Radius) : Shape;
public record Rectangle(double Width, double Height) : Shape;
public record Triangle(double A, double B, double C) : Shape;

public static double Area(Shape shape) => shape switch
{
    Circle(var r) => Math.PI * r * r,
    Rectangle(var w, var h) => w * h,
    Triangle(var a, var b, var c) => 
        Math.Sqrt(((a + b + c) / 2.0) 
            * (((a + b + c) / 2.0) - a) 
            * (((a + b + c) / 2.0) - b) 
            * (((a + b + c) / 2.0) - c)),
    _ => throw new ArgumentException()
};
```

### Когда писать F# vs C#

✅ **F# — лучший выбор:**
- Доменное моделирование (DDD-heavy)
- Финансовые расчёты, scientific computing
- Data analysis pipelines
- Compiler / parser / DSL writing
- Когда команда хочет functional-first

❌ **F# — не лучший:**
- ASP.NET Core full app (hassle с MVC)
- Unity / game dev (C# better tooling)
- Большая существующая C# codebase
- Нет F# разработчиков на рынке

---

## 12. Performance trade-offs

### Immutability cost

```csharp
[MemoryDiagnoser]
public class ImmutabilityBenchmark
{
    [Benchmark(Baseline = true)]
    public Address MutableUpdate()
    {
        var addr = new MutableAddress { Street = "Main", City = "NYC" };
        addr.Street = "Broadway";
        return addr;  // 1 allocation
    }
    
    [Benchmark]
    public ImmutableAddress ImmutableUpdate()
    {
        var addr = new ImmutableAddress("Main", "NYC");
        return addr with { Street = "Broadway" };  // 2 allocations
    }
}

// Mutable: 24 bytes per call
// Immutable: 48 bytes (двойной — orig + new)
```

### Solutions

1. **Record struct** — value type, no heap allocation
   ```csharp
   public record struct Point(int X, int Y);
   var p = new Point(1, 2);
   p = p with { X = 10 };  // stack-only!
   ```

2. **ImmutableArray** — copy-on-write, faster reads
   ```csharp
   ImmutableArray<int> arr = [1, 2, 3];
   var newArr = arr.Add(4);  // new array, but read-only access fast
   ```

3. **Persistent data structures** (LanguageExt) — share structure between versions
   ```csharp
   Lst<int> list = List(1, 2, 3, 4);
   Lst<int> updated = list.Add(5);
   // Internally — sharing common parts (tree-based)
   ```

См. [Span и Layout](../Runtime/span-layout.md) — record struct, layout, performance.

### LINQ allocation cost

```csharp
// ❌ Каждый Where/Select создаёт enumerator
var result = source
    .Where(x => x > 0)       // IEnumerable<int>
    .Select(x => x * 2)       // IEnumerable<int>
    .ToList();                // List<int>
// Allocations: ~3-4 lambda closures + IEnumerator instances

// ✅ В hot path — использовать Span / Memory
public int Sum(ReadOnlySpan<int> source)
{
    int sum = 0;
    foreach (var x in source)
        if (x > 0) sum += x * 2;
    return sum;
}
// 0 allocations
```

---

## 13. Common Pitfalls

### 1. "Functional" но с mutation внутри records

```csharp
// ❌ Records не делают глубоко immutable
public record Order(string Id, List<OrderItem> Items);

var order = new Order("1", new List<OrderItem>());
order.Items.Add(new OrderItem());  // mutates list!
```

**Лечение:** ImmutableList, IReadOnlyList, или manual защита.

### 2. Result\<T\> игнорируется

```csharp
// ❌ Caller забывает обработать
public Result<User, Error> GetUser(Guid id) { ... }

GetUser(id);  // result lost!
```

**Лечение:**
- `[MustUse]` атрибут (не стандартен, но через analyzers)
- Pattern matching обязательно
- Builder API: `GetUser(id).Match(success, failure)`

### 3. Performance hit от deep copy

```csharp
public record State(ImmutableDictionary<string, int> Counters);

// Каждое обновление — копия словаря (cheap, persistent), но всё же allocation
var newState = state with 
{ 
    Counters = state.Counters.SetItem("key", value) 
};

// В hot loop — ~ms на 1000 операций
```

**Лечение:** mutable inside, immutable outside boundary.

### 4. Over-engineering для простых случаев

```csharp
// Overkill для простой функции
Option<int> ParseSafe(string input) =>
    int.TryParse(input, out var x) ? Some(x) : None<int>();

// Достаточно
public static int? TryParse(string input) => 
    int.TryParse(input, out var x) ? x : (int?)null;
```

### 5. Throwing внутри pure functions

```csharp
// ❌ Не pure
public static decimal CalculateTax(decimal amount, string region)
{
    if (region is null) throw new ArgumentNullException();
    return amount * GetRate(region);
}

// ✅ Result или Option
public static Result<decimal, TaxError> CalculateTax(decimal amount, string region)
{
    if (string.IsNullOrEmpty(region))
        return new Failure<decimal, TaxError>(new InvalidRegion());
    return new Success<decimal, TaxError>(amount * GetRate(region));
}
```

### 6. Pattern matching exhaustiveness

```csharp
// ❌ Compiler не warning'ит если забыл case
public string Describe(Shape shape) => shape switch
{
    Circle => "round",
    Rectangle => "boxy"
    // Triangle забыли!
};
// Fall through to throw new SwitchExpressionException at runtime
```

**Лечение:**
- `_ => throw ...` или sealed types + analyzer
- В .NET 8+ — sealed records + pattern matching exhaustiveness checks

### 7. Async + Result — двойная сложность

```csharp
Task<Result<User, Error>> GetUserAsync(Guid id);

// Composition становится сложнее
var result = await GetUserAsync(id);
if (result is Success<User, Error>(var user))
{
    var addr = await GetAddressAsync(user.Id);
    // вложенность множится
}
```

**Лечение:** extension methods `BindAsync` / `MapAsync` — линейная композиция.

### 8. LINQ с side effects

```csharp
// ❌ Side effect в LINQ — surprises
items.Select(x =>
{
    Console.WriteLine(x);  // не выполнится пока не enumerate!
    return x * 2;
});
// Lazy: пока ToList не вызван — ничего не происходит
```

---

## 14. Best Practices

- **Records для DTO, Value Objects, Events** — natural fit
- **`with` expressions** для updates immutable объектов
- **`init` setters** для write-once polymorphism
- **Pattern matching** вместо if/else цепей
- **Sealed types + exhaustive switch** — compile-time safety
- **Result\<T, E\>** для expected errors, exceptions для unexpected
- **Option\<T\>** или nullable reference types — выбери одно
- **Pure functions** в domain core
- **Imperative shell** для I/O / DB / UI
- **ImmutableList / ImmutableArray** для shared collections
- **Record struct** для performance-critical small types
- **F# для domain modelling** при необходимости (можно из C# проекта reference)
- **LanguageExt частями** — Option, Either, Validation; не тащи всё
- **Тестируй pure функции** unit-тестами без mocks

---

## 15. Когда что — резюме

| Задача | Imperative C# | Functional C# |
|--------|---------------|----------------|
| ASP.NET MVC controllers | ✅ default | Result\<T\> для actions |
| EF Core CRUD | ✅ | Records для entities |
| Doman logic | OK | ✅ pure functions |
| Validation | If/else | ✅ Validation\<E, T\> accumulates |
| Business rules engine | Strategy pattern | ✅ partial application |
| ETL pipelines | foreach/loops | ✅ LINQ pipeline |
| Game logic (Unity) | ✅ MonoBehaviour state | Records для events |
| ML / Math | OK | ✅ F# или functional C# |

---

## См. также

- [Modern C# Features](modern-features.md) — pattern matching, records details
- [OOP](oop.md) — classes vs records
- [Collections и LINQ](collections-linq.md) — LINQ deep
- [Error Handling](error-handling.md) — Result vs Exceptions
- [Reflection и Expression Trees](reflection-expression-trees.md) — Expression-based composition
- [Modern Features](modern-features.md) — C# language evolution
- [Architecture Patterns](../Architecture/patterns.md) — functional core / imperative shell
- [DDD](../Architecture/ddd.md) — Value Objects через records

## Reading list

- **Scott Wlaschin — Domain Modeling Made Functional** (книга, F# но applicable to C#)
- **Scott Wlaschin — Railway Oriented Programming** — fsharpforfunandprofit.com
- **Mark Seemann — Code That Fits in Your Head** (книга, functional bias)
- **Vladimir Khorikov — Functional C# series** — enterprisecraftsmanship.com
- **Enrico Buonanno — Functional Programming in C#** (Manning, книга)
- **LanguageExt documentation** — github.com/louthy/language-ext
- **Microsoft Docs — Pattern matching** — learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching
- **F# for fun and profit** — fsharpforfunandprofit.com (обязательно)
- **Andrew Lock — Functional C# series** — andrewlock.net
- **Bartosz Adamczewski — Better C#** blog — leveluppp.com
