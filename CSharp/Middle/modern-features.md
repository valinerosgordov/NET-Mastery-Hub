---
tags: [csharp, modern-features, middle, records, init, pattern-matching, primary-constructors, raw-strings, extension-members]
level: Middle
date: 2026-08-02
---

# Modern C# Features (8-14) — что нового

> **Records, `init`, pattern matching, primary constructors, raw string literals, collection expressions, generic math, extension members.** Все ключевые фичи C# 8.0 → 14.0. Закрывает пробел: «знаю про records, не помню что нового в C# 13 vs 14, и зачем `field` keyword».

---

## 0. Как читать

Этот файл — мост от basic C# к современному. Если уже знаком с большинством — читай раздел 1 для overview, дальше cherry-pick. Если pre-C# 8 codebase — раздел 1→3 для приоритетных upgrades. Production guidance — раздел 14 (best practices), 16 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Эволюция C# 8 → 14

| Версия | Год | .NET | Ключевые фичи |
|--------|-----|------|---------------|
| **C# 8.0** | 2019 | .NET Core 3.0 | NRT, default interface methods, async streams, ranges/indices, switch expression, `using var`, pattern matching enhancements |
| **C# 9.0** | 2020 | .NET 5 | **Records**, `init`, top-level statements, pattern matching enhancements, target-typed `new`, source generators |
| **C# 10.0** | 2021 | .NET 6 | Global usings, file-scoped namespaces, record struct, primary constructor (records), lambda improvements, `with` для structs |
| **C# 11.0** | 2022 | .NET 7 | Raw string literals, `required` members, list patterns, generic attributes, static abstract в interfaces, `file` keyword |
| **C# 12.0** | 2023 | .NET 8 | **Primary constructors** (для всех classes/structs), collection expressions `[1,2,3]`, alias any type, default lambda parameters |
| **C# 13.0** | 2024 | .NET 9 | `params Span<T>`, partial properties, lock object type, escape sequence `\e`, `field` keyword (preview) |
| **C# 14.0** | 2025 | .NET 10 | **Extension members** (properties/static/operators), `field` (stable), null-conditional assignment `?.=`, `nameof` для unbound generics, first-class span conversions, partial constructors/events |

### 1.2. Главное правило

```
Включи в новых проектах:
  - <LangVersion>latest</LangVersion>
  - <Nullable>enable</Nullable>
  - <ImplicitUsings>enable</ImplicitUsings>

Используй активно (high impact):
  - records (C# 9) — DTO, value objects
  - init / required (C# 9-11) — immutable APIs
  - pattern matching (C# 8-11) — clean conditionals
  - primary constructors (C# 12) — DI без boilerplate
  - collection expressions [1,2,3] (C# 12)
  - file-scoped namespaces (C# 10)
  - raw string literals (C# 11)
  - global using (C# 10)

Selectively (specific scenarios):
  - default interface methods (C# 8) — non-breaking extension
  - generic math (C# 11 + .NET 7+) — math libraries
  - static abstract в interfaces (C# 11) — generic dispatch
  - field keyword (C# 14, preview в C# 13) — auto-property с logic в setter
  - extension members (C# 14) — properties/operators поверх чужих типов
```

> [!info]- Если ты знаешь Java / Kotlin / Swift / TypeScript
> **Java:** Records (Java 14+) — direct equivalent. Pattern matching (Java 21) — каtching up к C#. Sealed classes (Java 17) ↔ C# 11+ generic inference.
>
> **Kotlin:** data class ↔ record. Smart casts ↔ pattern matching. Очень похоже на C# modern.
>
> **Swift:** structs + protocols ↔ records + interfaces. Pattern matching очень mature.
>
> **TypeScript:** interfaces + type guards ↔ pattern matching. Discriminated unions ↔ exhaustive switch.

> [!question]- Интервью: какие самые важные фичи C# 8-14?
> **C# 8**: Nullable Reference Types (compile-time null safety), `using var`, switch expressions, ranges/indices, async streams (`IAsyncEnumerable<T>`), default interface methods. **C# 9**: **Records** (auto equality + with), `init` properties, top-level statements, target-typed `new()`, pattern enhancements. **C# 10**: file-scoped namespaces, global usings, record struct, lambda improvements. **C# 11**: raw string literals, `required` members, list patterns, generic attributes. **C# 12**: **Primary constructors для всех types**, collection expressions `[1,2,3]`. **C# 13**: partial properties, `params Span<T>`, `System.Threading.Lock`. **C# 14**: **extension members**, `field` keyword (stable), null-conditional assignment `?.=`.

---

## 2. Records (C# 9+)

### 2.1. Базовый record

```csharp
public record User(int Id, string Name, string Email);

var u1 = new User(1, "Alice", "a@x.com");
var u2 = new User(1, "Alice", "a@x.com");

u1 == u2;          // true — value equality автоматически
u1.Equals(u2);     // true
u1.GetHashCode();  // совпадает с u2

Console.WriteLine(u1);   // "User { Id = 1, Name = Alice, Email = a@x.com }"
```

Compiler генерирует: constructor, init properties, Equals/GetHashCode, ToString, Deconstruct, ==/!=.

### 2.2. with-expression

```csharp
var u1 = new User(1, "Alice", "a@x.com");
var u2 = u1 with { Name = "Bob" };   // copy с замененным Name

u1.Name;   // "Alice"
u2.Name;   // "Bob"
```

`with` — **non-destructive mutation**. Создаёт shallow copy с изменёнными полями.

### 2.3. record class vs record struct (C# 10+)

```csharp
public record class User(int Id, string Name);   // class — heap (default)
public record struct Point(double X, double Y);  // struct — stack
public readonly record struct Money(decimal Amount, string Currency);   // immutable struct
```

`record class` — reference, value equality. `record struct` — value type с record features. `readonly record struct` — fully immutable.

### 2.4. Inheritance в records

```csharp
public abstract record Animal(string Name);
public record Dog(string Name, string Breed) : Animal(Name);
public record Cat(string Name, bool IsIndoor) : Animal(Name);

var d = new Dog("Rex", "Labrador");
var a = (Animal)d;   // upcast
a == d;              // true — value equality with type check

new Animal("Rex") == new Dog("Rex", "Lab");   // false — different types
```

Records делают **type-aware equality** автоматически — Dog ≠ Animal даже с same Name.

### 2.5. Custom record body

```csharp
public record Order(int Id, decimal Total)
{
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    
    public bool IsLargeOrder => Total > 1000;
    
    public Order Discount(decimal percent) =>
        this with { Total = Total * (1 - percent / 100) };
}
```

После primary constructor — обычное body класса.

### 2.6. Когда record vs class

```
record когда:
  - DTO / API request/response
  - Value object (Money, Coordinate)
  - Immutable domain events
  - Equality по value важна
  - With-expression для evolution

class когда:
  - Domain entity с identity (User с changing state)
  - Mutable state с invariants
  - Service / Repository / Controller
  - Inheritance hierarchy с behavior
```

> [!question]- Интервью: чем `record` отличается от `class`?
> Record — **синтаксический сахар над class** с автоматически generated: parameterized constructor, **init-only** properties, **value equality** (Equals/GetHashCode/== по полям), **ToString** с named properties, **Deconstruct**, **`with` expression** для non-destructive mutation. Type-aware equality (Dog ≠ Animal в inheritance). Class — manual everything, mutable by default. Best practice: record для immutable DTO / value objects / events. Class — для domain entities с behavior + identity. `record struct` — value type с record features (C# 10+).

---

## 3. init и required

### 3.1. init (C# 9+)

```csharp
public class User
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    public string Email { get; init; } = "";
}

var u = new User { Id = 1, Name = "Alice", Email = "a@x.com" };
// u.Name = "Bob";   // ❌ — init only allows constructor / object initializer
```

`init` — like `set`, но **доступен только в constructor или object initializer**. После — read-only.

### 3.2. Зачем init vs set

```csharp
// Без init — нужен constructor для immutability
public class User
{
    public int Id { get; }
    public string Name { get; }
    public User(int id, string name) { Id = id; Name = name; }
}

// С init — flexibility object initializer + immutability
public class User
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
}

new User { Id = 1, Name = "Alice" };   // immutable, but flexible init
```

Combine с `required` (C# 11+) — compile-time guarantee.

### 3.3. required (C# 11+)

```csharp
public class User
{
    public required string Email { get; init; }
    public required string Name { get; init; }
    public int Age { get; init; } = 0;
}

var u = new User { Email = "a@x.com", Name = "Alice" };   // OK
// var u2 = new User { Email = "..." };   // ❌ Compile error — Name required
// var u3 = new User();                    // ❌ Compile error
```

`required` — compile-time guarantee что caller задал поле. Combine `required init` — immutable + non-null guaranteed.

### 3.4. SetsRequiredMembers attribute

```csharp
public class User
{
    public required string Email { get; init; }
    public required string Name { get; init; }
    
    [SetsRequiredMembers]
    public User(string email, string name)
    {
        Email = email;
        Name = name;
    }
}

new User("a@x.com", "Alice");   // OK — constructor sets required
```

Constructor с `[SetsRequiredMembers]` снимает с caller обязанность задавать required.

### 3.5. init + records

```csharp
public record User(int Id, string Name)
{
    public required string Email { get; init; }   // required + init в record
}

new User(1, "Alice") { Email = "a@x.com" };   // OK
// new User(1, "Alice");                        // ❌ Email required
```

> [!question]- Интервью: чем `init` отличается от `set`?
> **`set`** — assignable anytime после construction. Mutable property. **`init`** — assignable только в constructor или object initializer. После — read-only. Cleaner immutability vs read-only `{ get; }` (которое требует constructor для assignment) — позволяет flexible object initializer syntax. Combine с **`required`** (C# 11+) — caller обязан задать поле в object initializer (compile-time check). `[SetsRequiredMembers]` на constructor — снимает требование если ctor задаёт.

---

## 4. Pattern matching (C# 8-11)

### 4.1. is patterns (C# 8+)

```csharp
// Type pattern
if (obj is string s) Console.WriteLine(s.Length);

// Null pattern
if (obj is null) return;
if (obj is not null) DoSomething();

// Property pattern (C# 8)
if (user is { Age: > 18 }) AllowEntry();
if (order is { Status: "Pending", Total: > 100 }) NotifyManager();

// Recursive
if (point is { X: 0, Y: 0 }) Origin();

// Nested
if (response is { Data: { Status: "OK" } }) ProcessSuccess();
```

### 4.2. Switch expression (C# 8+)

```csharp
var description = day switch
{
    DayOfWeek.Saturday or DayOfWeek.Sunday => "Weekend",
    DayOfWeek.Friday => "Almost weekend",
    _ => "Weekday"
};

// С property patterns
var status = order switch
{
    { Status: "Paid", Total: > 1000 } => "VIP",
    { Status: "Paid" } => "Regular",
    { Status: "Pending" } => "Waiting",
    _ => "Unknown"
};

// С type patterns
var area = shape switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Square s => s.Side * s.Side,
    Rectangle { Width: var w, Height: var h } => w * h,
    null => 0,
    _ => throw new InvalidOperationException()
};
```

### 4.3. Relational patterns (C# 9+)

```csharp
var category = age switch
{
    < 0 => throw new ArgumentException(),
    < 13 => "Child",
    >= 13 and < 18 => "Teen",
    >= 18 and < 65 => "Adult",
    _ => "Senior"
};
```

### 4.4. Logical patterns — and / or / not

```csharp
if (n is > 0 and < 100) { }
if (s is "yes" or "y" or "true") { }
if (ch is not (',' or ';' or ':')) { }
```

### 4.5. List patterns (C# 11+)

```csharp
int[] arr = [1, 2, 3];

var description = arr switch
{
    [] => "empty",
    [_] => "single",
    [_, _] => "pair",
    [1, _, _] => "starts with 1",
    [_, .., 3] => "ends with 3",
    [var first, .., var last] => $"first={first}, last={last}",
    _ => "other"
};
```

### 4.6. var patterns

```csharp
if (DateTime.Now is var now && now.Hour > 17)
    SendEveningEmail(now);

// Capture intermediate
var result = obj switch
{
    { Value: var v } when v > 100 => $"big {v}",
    _ => "small"
};
```

### 4.7. when clauses

```csharp
var category = number switch
{
    int n when n < 0 => "negative",
    int n when n == 0 => "zero",
    int n when n > 0 => "positive",
    _ => "unknown"
};

// В catch
try { }
catch (HttpException ex) when (ex.StatusCode == 404) { }
```

> [!question]- Интервью: что такое pattern matching в C#?
> Composite система проверок — type, property values, structure — через unified syntax. Возрастал C# 7 → 11. **Type patterns** (`obj is string s`), **property patterns** (`{ Age: > 18 }`), **list patterns** (C# 11+: `[1, .., 3]`), **relational** (`< 100`), **logical** (`and`/`or`/`not`), **var** (capture intermediate), **when** clauses (additional guards). Используется в `if`, `switch` expression, `catch` filter. Cleaner альтернатива nested ifs + casts. Exhaustiveness check — compiler warning если не все cases covered.

---

## 5. Primary constructors (C# 12+)

### 5.1. Базовый

```csharp
public class UserService(IUserRepository repo, ILogger<UserService> logger)
{
    public async Task<User?> GetByIdAsync(int id)
    {
        logger.LogInformation("Getting user {Id}", id);
        return await repo.GetByIdAsync(id);
    }
}
```

`(IUserRepository repo, ILogger<UserService> logger)` — primary constructor parameters. Доступны во всём теле class как captured variables.

### 5.2. До C# 12 — boilerplate

```csharp
public class UserService
{
    private readonly IUserRepository _repo;
    private readonly ILogger<UserService> _logger;
    
    public UserService(IUserRepository repo, ILogger<UserService> logger)
    {
        _repo = repo;
        _logger = logger;
    }
    
    public async Task<User?> GetByIdAsync(int id)
    {
        _logger.LogInformation(...);
        return await _repo.GetByIdAsync(id);
    }
}
```

C# 12 убирает 6 lines boilerplate.

### 5.3. Properties через primary constructor

```csharp
public class User(int id, string name)
{
    public int Id { get; } = id;
    public string Name { get; } = name;
}
```

Параметры primary constructor — **не properties автоматически** (в class). Manual properties с initialization.

В **records** primary constructor params автоматически становятся properties:

```csharp
public record User(int Id, string Name);   // Id и Name — public init properties
```

### 5.4. Validation в primary constructor

```csharp
public class Money(decimal amount, string currency)
{
    public decimal Amount { get; } = amount >= 0
        ? amount
        : throw new ArgumentException("Amount must be non-negative");
    public string Currency { get; } = !string.IsNullOrWhiteSpace(currency)
        ? currency
        : throw new ArgumentException("Currency required");
}
```

Validation в property initializer. Cleaner альтернатива через explicit constructor:

```csharp
public class Money
{
    public Money(decimal amount, string currency)
    {
        if (amount < 0) throw new ArgumentException();
        // ...
    }
}
```

### 5.5. base() call

```csharp
public abstract class Animal(string name)
{
    public string Name { get; } = name;
}

public class Dog(string name, string breed) : Animal(name)
{
    public string Breed { get; } = breed;
}
```

`: Animal(name)` — passes parameter to base primary constructor.

### 5.6. Pitfalls

```csharp
// ❌ Captured variable, не readonly field
public class Service(IRepository repo)
{
    public void Method1() => repo.Save();   // OK
    public void Method2() => repo = null;   // ❌ Не работает — readonly capture
}

// Field semantics — параметры immutable вне assignments в declaration
```

Primary constructor parameters — **immutable** в class body. Если нужен mutable — explicit field.

### 5.7. Когда primary constructor

✅ **Используй когда:**
- DI dependencies (большинство classes).
- Records (главное use case).
- Simple value types (Money, Coordinate).

❌ **Не используй когда:**
- Complex initialization logic (parsing, multi-step).
- Multiple constructors с разной semantics.
- Conditional branching в init.

> [!question]- Интервью: что нового в primary constructors C# 12?
> До C# 12 primary constructors были **только в records** (C# 9+). C# 12 расширяет на **все class и struct**. Параметры доступны во всём теле как **captured variables** (immutable в body). Class **не делает** их properties автоматически (в отличие от records) — manual `public int Id { get; } = id;`. Use case #1: **DI dependencies** — убирает boilerplate (private readonly + ctor + assign). Properties с validation: `public T X { get; } = validate(input)`. Base passing: `: Animal(name)`. Не подходит для complex init / multi-step / conditional.

---

## 6. Collection expressions (C# 12+)

### 6.1. Базовый syntax

```csharp
int[] arr = [1, 2, 3, 4, 5];                           // array
List<int> list = [1, 2, 3];                             // List<T>
Span<int> span = [1, 2, 3];                              // Span<T>
ReadOnlySpan<char> chars = ['a', 'b', 'c'];              // ReadOnlySpan<char>
ImmutableArray<int> immutable = [1, 2, 3];               // ImmutableArray
HashSet<int> set = [1, 2, 3];                             // HashSet<T>
```

Compiler выводит type из target.

### 6.2. Spread operator

```csharp
int[] a = [1, 2, 3];
int[] b = [4, 5, 6];
int[] c = [..a, ..b, 7, 8];   // [1, 2, 3, 4, 5, 6, 7, 8]
```

`..source` — spread elements. Concatenate множество.

### 6.3. Empty / single

```csharp
List<int> empty = [];
List<string> one = ["hello"];
```

### 6.4. С constructor params

```csharp
public class Vector(double x, double y)
{
    public double X { get; } = x;
    public double Y { get; } = y;
}

var v = new Vector(1.0, 2.0);

// Nested
public record Path(string[] Segments);
var p = new Path(["users", "alice", "documents"]);
```

### 6.5. Performance

Collection expressions — **compiler optimization**. Для `int[]` / `List<T>` — direct allocation, no LINQ overhead.

```csharp
// До C# 12
List<int> list = new List<int>(new[] { 1, 2, 3 });   // двойная allocation

// C# 12+
List<int> list = [1, 2, 3];   // direct, optimal
```

### 6.6. Ограничения

- Type должен поддерживать collection expressions (`IEnumerable<T>`, indexer + Add, etc.).
- Custom types — нужен `[CollectionBuilder]` attribute.

> [!question]- Интервью: что такое collection expressions C# 12?
> Unified syntax `[1, 2, 3]` для **разных collection types** — array, List, Span, ReadOnlySpan, HashSet, ImmutableArray, custom через `[CollectionBuilder]`. Compiler выводит type из target. **Spread operator `..source`** для concatenation. Преимущества: 1) **Cleaner syntax** — `[1, 2, 3]` vs `new List<int> { 1, 2, 3 }`. 2) **Performance** — compiler optimizes (direct allocation, no LINQ overhead). 3) **Polymorphic** — same syntax для разных types. Limitations: type должен support — `IEnumerable<T>` + indexer/Add, или [CollectionBuilder] attribute для custom.

---

## 7. Raw string literals (C# 11+)

### 7.1. Triple quotes

```csharp
string json = """
    {
        "name": "Alice",
        "age": 30
    }
    """;
```

`"""..."""` — raw string. Никакой escaping не нужен.

### 7.2. Несколько кавычек если нужен `"""` внутри

```csharp
string code = """"
    This contains """ triple quotes
    """";

// Или ещё больше
string deep = """""
    This has """" four quotes
    """"";
```

Open/close — **больше** того что встречается внутри. 4-quote raw string handles 3-quote sequences.

### 7.3. Indentation handling

```csharp
string s = """
    Line 1
    Line 2
    """;

// Compiler автоматически удаляет common indentation
// (закрывающие """)
// Result:
// Line 1
// Line 2
```

Indentation closing `"""` — **base** для удаления. Все lines с **меньшим** indent — error.

### 7.4. Multiline / single-line

```csharp
string single = """quoted text""";   // single line (без newlines между)

string multi = """
    multi
    line
    """;
```

### 7.5. Interpolation в raw strings

```csharp
var name = "Alice";

// Single $: regular interpolation rules
string s1 = $"""
    Hello {name}
    """;

// Multiple $: больше braces для interpolation
string s2 = $$"""
    JSON: { "name": "{{name}}" }
    """;
// Two $ — нужно {{ }} для interpolation, single { stays literal
```

### 7.6. Когда use

✅ **Используй когда:**
- JSON, XML, regex literals (escaping pain).
- SQL queries multi-line.
- Code snippets / templates.

❌ **Не используй когда:**
- Simple short strings — `"Hello"`.
- One-line с simple escaping — verbose.

> [!question]- Интервью: что такое raw string literals?
> Triple-quote `"""..."""` (или больше) — string без escaping. **No `\\` for backslash, `\"` for quote**. Идеально для JSON, XML, regex, SQL. Multi-line — compiler удаляет common indentation на основе closing `"""`. **Interpolation**: `$"""..."""` (single $) — `{var}`. `$$"""..."""` (two $) — `{{var}}` (single `{` stays literal). C# 11+. Replaces `@"..."` (verbatim) для most cases — verbatim не handles `"` без doubling.

---

## 8. File-scoped namespaces и global using (C# 10+)

### 8.1. File-scoped namespace

```csharp
// До C# 10
namespace MyApp.Domain
{
    public class Order { }
    public class User { }
}

// C# 10+
namespace MyApp.Domain;

public class Order { }
public class User { }
```

Reduces один level indentation. **Стандарт** в new code.

### 8.2. global using

```csharp
// GlobalUsings.cs
global using System;
global using System.Collections.Generic;
global using System.Threading.Tasks;
global using static System.Math;
```

Все usings — applied **во всех файлах** проекта. Reduce repetition.

### 8.3. Implicit usings

```xml
<!-- .csproj -->
<PropertyGroup>
  <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

`<ImplicitUsings>enable</ImplicitUsings>` — automatically adds common usings (System, System.Linq, System.Threading.Tasks, etc.). По SDK type (Web, Console).

### 8.4. Best practice

```csharp
// GlobalUsings.cs
global using System.Threading.Channels;
global using MyApp.Domain;
global using MyApp.Common;

// project files — clean от common usings
namespace MyApp.Services;

public class OrderService { /* ... */ }
```

---

## 9. Pattern matching new (C# 11)

### 9.1. List patterns deep

```csharp
int[] arr = [1, 2, 3, 4, 5];

bool isCorrect = arr switch
{
    [] => false,
    [_] => false,
    [.., 5] => true,
    [1, ..] => true,
    [1, _, _, _, 5] => true,
    _ => false
};
```

### 9.2. Slice patterns

```csharp
var (first, rest) = arr switch
{
    [var f, .. var r] => (f, r),
    _ => (0, Array.Empty<int>())
};

// .. captures rest as same type
```

### 9.3. Property patterns nested

```csharp
public record Address(string Street, string City);
public record User(string Name, Address? Address);

var description = user switch
{
    { Address: { City: "NYC" } } => "New Yorker",
    { Address: { City: "LA" } } => "LAer",
    { Address: null } => "No address",
    _ => "Other"
};

// Cleaner с extended property patterns (C# 10+)
var description2 = user switch
{
    { Address.City: "NYC" } => "New Yorker",
    { Address.City: "LA" } => "LAer",
    { Address: null } => "No address",
    _ => "Other"
};
```

---

## 10. Generic attributes и static abstract (C# 11+)

### 10.1. Generic attributes

```csharp
// До C# 11
[Handler(typeof(CreateUserCommand))]
public class CreateUserHandler { }

// C# 11+
[Handler<CreateUserCommand>]
public class CreateUserHandler { }

[AttributeUsage(AttributeTargets.Class)]
public class HandlerAttribute<T> : Attribute where T : ICommand
{
    public Type CommandType => typeof(T);
}
```

См. [[attributes-metadata]] раздел 4.

### 10.2. Static abstract в interfaces

```csharp
public interface IShape
{
    static abstract IShape Create();
    abstract double Area();
}

public class Circle : IShape
{
    public double Radius { get; init; }
    public static IShape Create() => new Circle { Radius = 1 };
    public double Area() => Math.PI * Radius * Radius;
}

T MakeShape<T>() where T : IShape => (T)T.Create();
```

Enables **generic dispatch на static methods** — base для generic math.

### 10.3. Generic math

```csharp
public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var v in values) sum += v;
    return sum;
}

Sum(new[] { 1, 2, 3 });        // works для int
Sum(new[] { 1.5, 2.5 });        // works для double
Sum(new[] { 1m, 2m, 3m });      // works для decimal
```

См. [[numeric-types-math]] раздел 7.

---

## 11. C# 13 (.NET 9)

### 11.1. `params Span<T>`

```csharp
// До C# 13
public void Log(params object[] args) { }   // allocates array

// C# 13+
public void Log(params ReadOnlySpan<object> args) { }   // no allocation для small
```

### 11.2. partial properties

```csharp
public partial class ViewModel
{
    public partial string Name { get; set; }
}

// Generated code (source generator):
public partial class ViewModel
{
    private string _name = "";
    public partial string Name
    {
        get => _name;
        set { if (_name != value) { _name = value; OnPropertyChanged(); } }
    }
}
```

### 11.3. field keyword

```csharp
// До C# 13
private string _name = "";
public string Name
{
    get => _name;
    set => _name = value?.Trim() ?? "";
}

// C# 13 (preview) → C# 14 (stable)
public string Name
{
    get;
    set => field = value?.Trim() ?? "";
}
```

`field` — contextual keyword referencing **auto-generated backing field**. Без явного declaration. В C# 13 фича была доступна только с `<LangVersion>preview</LangVersion>`; **стабильна с C# 14** — см. раздел 12.

### 11.4. Lock object type

```csharp
// До C# 13 — обычный object
private readonly object _lock = new();

// C# 13+ — System.Threading.Lock optimized
private readonly Lock _lock = new();

lock (_lock) { /* ... */ }
```

`Lock` type — JIT optimizes для better performance.

### 11.5. Escape sequence \e

```csharp
string ansiColor = "\e[31m";   // ESC character (0x1B)
// До C# 13: "\u001b[31m"
```

> [!question]- Интервью: что нового в C# 13?
> 1) **partial properties** — declare в один файл, implement в другом (для source generators). 2) **`params ReadOnlySpan<T>`** — params без heap allocation для small collections. 3) **`System.Threading.Lock`** — оптимизированный `lock` object type. 4) **`allows ref struct`** — `ref struct` как generic type argument. 5) **`\e` escape** для ESC character (ANSI codes), implicit index access в object initializers. 6) **`field` keyword** — в C# 13 только preview, стабилен с C# 14. C# 13 фокусируется на **performance** и **clean syntax**.

---

## 12. C# 14 (.NET 10) — самое новое

### 12.1. Extension members

C# 14 расширяет extension methods до **properties, static members и operators**. Новый синтаксис — `extension`-блок внутри static class:

```csharp
public static class StringExtensions
{
    // Instance extension members — receiver с именем
    extension(string s)
    {
        public bool IsBlank => string.IsNullOrWhiteSpace(s);            // extension property
        public string Truncate(int max) => s.Length <= max ? s : s[..max];
    }

    // Static extension members — receiver без имени (только тип)
    extension(string)
    {
        public static string Repeat(char c, int count) => new(c, count);
    }
}

// Использование:
// "  ".IsBlank            → true — property «на строке»
// string.Repeat('-', 3)   → "---" — static member «на типе»
```

Generic receiver и extension operators:

```csharp
public static class SequenceExtensions
{
    extension<T>(IEnumerable<T> source) where T : INumber<T>
    {
        public T Sum() { T acc = T.Zero; foreach (var v in source) acc += v; return acc; }

        public static IEnumerable<T> operator +(IEnumerable<T> left, IEnumerable<T> right) =>
            left.Zip(right, (a, b) => a + b);   // extension operator
    }
}
```

Механизм: compiler генерирует обычные static методы (property → `get_`/`set_`), поэтому extension members видны из любого C#-кода, но state добавить нельзя — это по-прежнему sugar над static calls. Старый синтаксис `static R Method(this T x)` остаётся валидным и компилируется в то же самое — миграция не нужна.

### 12.2. Null-conditional assignment `?.=`

```csharp
// До C# 14
if (customer is not null)
    customer.Order = GetCurrentOrder();

// C# 14
customer?.Order = GetCurrentOrder();   // assignment только если customer != null
customer?.LoyaltyPoints += 10;         // работает и с compound assignment
```

Важно: RHS **не вычисляется**, если receiver == null — `GetCurrentOrder()` не вызовется.

### 12.3. field стал стабильным

`field` keyword (см. 11.3) в C# 14 доступен без `<LangVersion>preview</LangVersion>` — можно использовать в production.

### 12.4. Мелкие, но полезные

```csharp
// nameof для unbound generics
string a = nameof(List<>);          // "List" — до C# 14 требовалось nameof(List<int>)
string b = nameof(Dictionary<,>);   // "Dictionary"
```

```csharp
// Partial constructors и events (дополняет partial properties из C# 13)
public partial class ViewModel
{
    public partial ViewModel(IAppServices services);   // defining declaration
    public partial event EventHandler? Changed;        // defining declaration
}

public partial class ViewModel   // вторая часть — например, от source generator
{
    private EventHandler? _changed;

    public partial ViewModel(IAppServices services) => Services = services;

    public partial event EventHandler? Changed
    {
        add => _changed += value;
        remove => _changed -= value;
    }

    public IAppServices Services { get; }
}
```

**First-class span conversions** — implicit conversions `T[]` → `Span<T>`/`ReadOnlySpan<T>` и `Span<T>` → `ReadOnlySpan<T>` теперь на уровне языка: меньше ceremony, лучше overload resolution — span-overload выбирается без явного `.AsSpan()` в большем числе случаев.

**User-defined compound assignment** — тип может объявить instance-оператор `public void operator +=(T other)`, мутирующий значение in-place вместо «создать новое + присвоить» (важно для больших structs, tensor/money-типов).

> [!question]- Интервью: что нового в C# 14?
> 1) **Extension members** — `extension(T receiver) { }` блоки в static class: extension **properties**, **static members** и **operators**, не только методы; компилируются в static методы, старый `this`-синтаксис остаётся валиден. 2) **`field` keyword** стабилен (в C# 13 был preview). 3) **Null-conditional assignment** `?.=` и `?.+=` — RHS не вычисляется при null receiver. 4) **`nameof` для unbound generics** — `nameof(List<>)`. 5) **First-class span conversions** — implicit array/Span/ReadOnlySpan conversions на уровне языка. 6) **Partial constructors/events**. 7) **User-defined compound assignment** — `void operator +=` in-place.

---

## 13. NRT и async streams (C# 8 — review)

### 13.1. Nullable reference types

См. [[nullable-types]].

### 13.2. async streams — `IAsyncEnumerable<T>`

```csharp
public async IAsyncEnumerable<int> GenerateAsync([EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 10; i++)
    {
        await Task.Delay(100, ct);
        yield return i;
    }
}

// Consumer
await foreach (var n in GenerateAsync(ct))
{
    Console.WriteLine(n);
}
```

`IAsyncEnumerable<T>` + `await foreach` — async streaming. Лучше чем `Task<List<T>>` для **streaming**.

### 13.3. async streams + cancellation

```csharp
// Producer
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct)
{
    while (await orderQueue.WaitToReadAsync(ct))
    {
        if (orderQueue.TryRead(out var order))
            yield return order;
    }
}

// Consumer
using var cts = new CancellationTokenSource(TimeSpan.FromMinutes(1));
await foreach (var order in StreamOrdersAsync(cts.Token))
{
    await ProcessAsync(order);
}
```

`[EnumeratorCancellation]` — token propagated в `WithCancellation` calls.

### 13.4. ConfigureAwait для streams

```csharp
await foreach (var x in stream.WithCancellation(ct).ConfigureAwait(false))
{
    // ...
}
```

В library code — `ConfigureAwait(false)` для no sync context capture.

---

## 14. Best Practices

### 14.1. Adoption priority (high impact)

- ✅ **Records** для DTO / value objects.
- ✅ **`init` + `required`** для immutable APIs.
- ✅ **Pattern matching** для clean conditionals.
- ✅ **Primary constructors** (C# 12) для DI.
- ✅ **Collection expressions** `[1,2,3]`.
- ✅ **File-scoped namespaces**.
- ✅ **Raw strings** для JSON/regex/SQL.
- ✅ **NRT** + `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>`.

### 14.2. Selectively (specific scenarios)

- ✅ **`field` keyword** (C# 14) для simple validation в setter.
- ✅ **Extension members** (C# 14) для properties/operators поверх чужих типов.
- ✅ **Generic math** (.NET 7+) для math libraries.
- ✅ **Static abstract в interfaces** для generic dispatch.
- ✅ **List patterns** для array structure check.
- ✅ **async streams** для streaming data.

### 14.3. Project setup

```xml
<PropertyGroup>
  <TargetFramework>net8.0</TargetFramework>
  <LangVersion>latest</LangVersion>
  <Nullable>enable</Nullable>
  <ImplicitUsings>enable</ImplicitUsings>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
</PropertyGroup>
```

### 14.4. Migration old → modern

- Replace `class` DTOs → `record`.
- Replace constructor-only init → `init`.
- Replace `Object.Equals` overrides → records.
- Replace nested `if`/`else` → switch expression.
- Replace `Func<T, bool>` chains → property patterns.
- Replace `IEnumerable<T> + Task` → `IAsyncEnumerable<T>`.

### 14.5. Не делай

- ❌ Переход на новые фичи без team understanding.
- ❌ Mix старых и новых patterns в одном файле без причины.
- ❌ `field` keyword там где `set` достаточен.
- ❌ Primary constructors с complex init (используй explicit ctor).

---

## 15. Cheat sheet

```csharp
// === Records ===
public record User(int Id, string Name, string Email);
public record class Order(int Id, decimal Total);   // explicit class
public record struct Point(double X, double Y);
public readonly record struct Money(decimal Amount, string Currency);

var u2 = u1 with { Name = "Bob" };

// === Init + required ===
public class User
{
    public required string Email { get; init; }
    public string Name { get; init; } = "";
}
new User { Email = "a@x.com", Name = "Alice" };

// === Pattern matching ===
var area = shape switch
{
    Circle { Radius: > 0 } c => Math.PI * c.Radius * c.Radius,
    Square { Side: var s } => s * s,
    null => 0,
    _ => throw new InvalidOperationException()
};

if (point is { X: > 0, Y: > 0 }) { }
if (arr is [1, .., 5]) { }
if (s is "yes" or "y") { }
if (n is > 0 and < 100) { }

// === Primary constructor (C# 12) ===
public class Service(IRepo repo, ILogger<Service> logger)
{
    public async Task DoAsync()
    {
        logger.LogInformation("...");
        await repo.SaveAsync();
    }
}

// === Collection expressions (C# 12) ===
int[] arr = [1, 2, 3];
List<string> list = ["a", "b", "c"];
int[] combined = [..arr1, ..arr2, 99];

// === Raw strings (C# 11) ===
string json = """
    { "name": "Alice" }
    """;

string interp = $"""
    User: {name}
    """;

// === File-scoped namespace ===
namespace MyApp.Domain;

public class Order { /* ... */ }

// === Global using ===
global using System.Threading.Channels;

// === Async streams ===
await foreach (var x in StreamAsync(ct))
{
    Process(x);
}

// === field keyword (C# 14, preview в C# 13) ===
public string Name
{
    get;
    set => field = value?.Trim() ?? "";
}

// === Extension members (C# 14) ===
public static class OrderExtensions
{
    extension(Order order)
    {
        public bool IsLarge => order.Total > 1000;
    }
}

// === Null-conditional assignment (C# 14) ===
customer?.Order = GetCurrentOrder();   // RHS не вычисляется при null

// === Switch expression ===
var msg = day switch
{
    DayOfWeek.Saturday or DayOfWeek.Sunday => "Weekend",
    DayOfWeek.Friday => "TGIF",
    _ => "Workday"
};
```

---

## 16. Common Pitfalls

### 16.1. Records mutable properties

```csharp
public record User
{
    public string Name { get; set; } = "";   // ❌ set, не init — mutable
}
```

**Фикс:** `init` или primary constructor.

### 16.2. Required без attribute в constructor

```csharp
public class User
{
    public required string Email { get; init; }
    public User(string email) => Email = email;   // ❌ caller всё равно должен set Email
}
```

**Фикс:** `[SetsRequiredMembers]` attribute.

### 16.3. Pattern matching не exhaustive

```csharp
var x = shape switch
{
    Circle c => c.Area,
    Square s => s.Area
    // ❌ нет _ — non-exhaustive, может быть SwitchExpressionException
};
```

**Фикс:** `_ => throw new InvalidOperationException()`.

### 16.4. Primary constructor capture immutable

```csharp
public class Service(IRepo repo)
{
    public void Reset() => repo = null;   // ❌ readonly capture
}
```

**Фикс:** explicit field если нужен mutation.

### 16.5. Collection expression non-supported type

```csharp
// Custom type без [CollectionBuilder]
public class MyList<T> { }   // нет support для []
MyList<int> l = [1, 2, 3];   // ❌ compile error
```

**Фикс:** add `[CollectionBuilder]` attribute.

### 16.6. Raw string indentation mismatch

```csharp
string s = """
    line 1
  line 2   // ❌ меньше indent чем closing """
    """;
```

**Фикс:** consistent indentation.

### 16.7. Records — value equality includes all fields

```csharp
public record User(int Id, string Name)
{
    public DateTime LastSeen { get; init; }   // included в equality!
}

new User(1, "A") { LastSeen = ... } == new User(1, "A") { LastSeen = ... };
// false если LastSeen differs!
```

**Фикс:** override Equals или exclude через separate class.

### 16.8. async void в primary constructor body

```csharp
public class Service(IRepo repo)
{
    private async void Init() => await repo.LoadAsync();   // ❌ async void
    
    public Service() : this(...) { Init(); }   // fire-and-forget bad
}
```

**Фикс:** explicit factory method, не constructor.

### 16.9. Global using — namespace pollution

```csharp
// GlobalUsings.cs
global using Newtonsoft.Json;
global using System.Text.Json;
// Конфликт: какой JsonSerializer?
```

**Фикс:** alias через `using` per-file.

### 16.10. switch expression без braces

```csharp
var x = n switch
{
    > 0 => "positive",
    < 0 => "negative"
    // ❌ no _ — может throw на 0
};
```

**Фикс:** `0 => "zero"` + `_ => ...`.

> [!question]- Интервью: топ-3 ошибки с modern features?
> 1) **Records с `set` properties** — теряется immutability + value equality. Always `init`. 2) **Pattern matching не exhaustive** — `_ => throw` обязательно. SwitchExpressionException иначе. 3) **Records value equality includes ALL properties** — даже `LastSeen` timestamp ломает equality. Override Equals или separate non-record для mutable parts.

---

## 17. Practice exercises

### 17.1. DDD-style Money с modern C#

```csharp
public readonly record struct Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0m, currency);
    
    public Money Add(Money other) => Currency == other.Currency
        ? this with { Amount = Amount + other.Amount }
        : throw new InvalidOperationException("Currency mismatch");
    
    public Money Apply(decimal taxRate) => taxRate >= 0
        ? this with { Amount = Amount * (1 + taxRate) }
        : throw new ArgumentException();
    
    public static Money operator +(Money a, Money b) => a.Add(b);
}

var price = new Money(19.99m, "USD");
var withTax = price.Apply(0.085m);
```

### 17.2. Pattern matching state machine

```csharp
public enum OrderStatus { Pending, Paid, Shipped, Delivered, Cancelled }

public record Order(int Id, OrderStatus Status, decimal Total, DateTime? PaidAt = null);

public static class OrderTransitions
{
    public static Order Transition(Order order, string action) => (order.Status, action) switch
    {
        (OrderStatus.Pending, "pay") => order with { Status = OrderStatus.Paid, PaidAt = DateTime.UtcNow },
        (OrderStatus.Paid, "ship") => order with { Status = OrderStatus.Shipped },
        (OrderStatus.Shipped, "deliver") => order with { Status = OrderStatus.Delivered },
        (OrderStatus.Pending or OrderStatus.Paid, "cancel") => order with { Status = OrderStatus.Cancelled },
        _ => throw new InvalidOperationException($"Cannot {action} from {order.Status}")
    };
}
```

### 17.3. Modern service с DI

```csharp
public class UserService(
    IUserRepository repo,
    IEmailService email,
    ILogger<UserService> logger)
{
    public async Task<Result<User>> RegisterAsync(RegisterRequest req)
    {
        logger.LogInformation("Registering {Email}", req.Email);
        
        if (await repo.ExistsAsync(req.Email))
            return new Result<User>.Failure("Email already registered");
        
        var user = new User
        {
            Email = req.Email,
            Name = req.Name,
            CreatedAt = DateTime.UtcNow
        };
        
        await repo.AddAsync(user);
        await email.SendWelcomeAsync(user);
        
        return new Result<User>.Success(user);
    }
}

public record RegisterRequest
{
    public required string Email { get; init; }
    public required string Name { get; init; }
}
```

---

## 18. Что читать дальше

1. **Специфика C# 8-14** — [official docs](https://learn.microsoft.com/dotnet/csharp/whats-new/).
2. **[[oop|OOP]]** — records deep, primary constructors.
3. **[[nullable-types|Nullable Types]]** — NRT detail.
4. **[[generics-deep|Generics deep]]** — generic math, static abstract.
5. **Source generators** — partial properties, generated code.

---

## 19. См. также

- [[oop|OOP]] — records, init, sealed
- [[nullable-types|Nullable Types]] — NRT
- [[generics-deep|Generics deep]] — generic math
- [[error-handling|Error Handling]] — pattern matching в catch
- [[attributes-metadata|Attributes]] — generic attributes
- [[numeric-types-math|Numerics]] — `INumber<T>`
- [[keywords-reference|Keywords Reference]] — все modern keywords

---

## 20. Reading list

- **Microsoft Docs — What's new in C#** — learn.microsoft.com/dotnet/csharp/whats-new/
- **Microsoft Docs — C# 12 features** — learn.microsoft.com/dotnet/csharp/whats-new/csharp-12
- **Microsoft Docs — C# 13 features** — learn.microsoft.com/dotnet/csharp/whats-new/csharp-13
- **Microsoft Docs — C# 14 features** — learn.microsoft.com/dotnet/csharp/whats-new/csharp-14
- **Microsoft Docs — Records** — learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record
- **Microsoft Docs — Pattern matching** — learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching
- **Mads Torgersen** — C# language design lead, Microsoft DevBlogs
- **Andrew Lock** — andrewlock.net — modern C# patterns
- **Stephen Cleary** — blog.stephencleary.com — async streams
- **Dotnet runtime repo** — github.com/dotnet/runtime — generic math source
