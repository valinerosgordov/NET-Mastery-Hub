---
tags: [csharp, language-design, evolution, history, philosophy, breaking-changes, c-sharp-versions]
level: Senior
date: 2026-04-30
---

# C# Language Design — эволюция и философия

> История развития C# 1.0 → 14 (2026), почему добавлялись фичи, какие принципы. Закрывает: каждая версия с killer features, breaking changes, deprecation, language design philosophy, comparison с Java/TypeScript/Kotlin/Rust, что грядёт в C# 15+, как фичи влияют на performance/AOT/безопасность.

---

## Что это, зачем и когда

### Зачем знать эволюцию языка

Senior C# должен понимать **почему** язык такой, а не просто **как** им пользоваться:
- Какие проблемы решала каждая фича — context decision-making
- Что **deprecated** и почему — избегать legacy patterns
- Куда движется язык — готовиться к C# 15+
- Почему некоторые фичи противоречат друг другу (например, nullable reference types vs structs)
- Когда какую фичу применять — design choices

### Аналогия

Знать только последнюю версию языка как знать только сегодняшнее настроение друга. Не понимаешь характер, мотивы, прошлые ошибки. Senior знает language как сложную личность с историей.

---

## 1. Краткая история

| Год | Версия | .NET | Killer features |
|-----|--------|------|-----------------|
| 2002 | 1.0 | .NET 1.0 | Basic OOP, GC |
| 2005 | 2.0 | .NET 2.0 | **Generics**, nullable types, anonymous methods, iterators |
| 2007 | 3.0 | .NET 3.5 | **LINQ**, lambda, anonymous types, extension methods, var |
| 2010 | 4.0 | .NET 4.0 | **dynamic**, named/optional args, generic variance |
| 2012 | 5.0 | .NET 4.5 | **async/await** |
| 2015 | 6.0 | .NET 4.6 | string interpolation, expression-bodied, null-conditional |
| 2017 | 7.0-7.3 | .NET Core 2 | **tuples**, pattern matching basics, ref returns, `Span<T>` |
| 2019 | 8.0 | .NET Core 3 | **nullable reference types**, async streams, default interface methods |
| 2020 | 9.0 | .NET 5 | **records**, top-level statements, init setters, target-typed new |
| 2021 | 10.0 | .NET 6 | **global usings**, file-scoped namespaces, record structs |
| 2022 | 11.0 | .NET 7 | **list patterns**, raw strings, generic math (static abstract) |
| 2023 | 12.0 | .NET 8 | **primary constructors**, collection expressions, alias any type |
| 2024 | 13.0 | .NET 9 | **params collections**, `Lock` type, partial properties |
| 2025/26 | 14.0 | .NET 10 | extension types, type unions (preview), null-conditional assignment |

---

## 2. Дизайн-принципы C#

### Anders Hejlsberg's principles (Lead architect)

1. **General-purpose** — не привязан к одной нише
2. **Multi-paradigm** — OOP + functional + procedural
3. **Type-safe** — ошибки в compile-time
4. **Backward compatible** — старый код продолжает работать
5. **Pragmatic** — добавляем фичи когда есть real need
6. **Performance-conscious** — близко к C++ на критичных путях

### Compared с философиями других

| | C# | Java | TypeScript | Kotlin | Rust |
|--|--|------|-----------|--------|------|
| Компилируется в | IL → JIT/AOT | Bytecode | JS | JVM/JS | Native |
| GC | ✅ | ✅ | (browser) | ✅ | ❌ ownership |
| Type system | Static, sound | Static, sound | Gradual, unsound | Static, sound | Static, sound, ownership |
| Null safety | C# 8+ | ❌ | ✅ strict | ✅ | ✅ |
| OOP-first | ✅ | ✅ | ✅ | ✅ | ❌ trait-based |
| Functional | Strong | Weak (improving) | OK | Strong | Strong |
| Pattern matching | C# 8+ growing | Switch pattern (Java 21+) | Tagged unions | Sealed classes | Built-in |

### Что C# уникально хорошо делает

- **Performance + productivity** — ближе к C++ чем Java на критичных задачах + simplest IDE
- **Multi-paradigm balance** — OOP комфортный + functional growing
- **Tooling** — VS / Rider / VS Code лучшие в индустрии
- **Cross-platform native** — .NET 6+ Linux/macOS first-class
- **AOT compilation** — конкурент Go / Rust для cloud / CLI

### Что C# делает хуже Kotlin / Rust

- **Null safety** — flag-driven, не enforced (compared to Kotlin's `String?`)
- **Memory safety** — GC ловит, но не предотвращает race conditions (Rust ownership)
- **Async cancellation** — manual через CancellationToken (vs Kotlin coroutines structured concurrency)
- **Sum types** — discriminated unions через records hierarchy (vs Kotlin sealed class или Rust enum)

---

## 3. C# 1.0 — 2.0: Foundation

### C# 1.0 (2002) — start

```csharp
// Boilerplate-heavy, all collections — non-generic
ArrayList list = new ArrayList();
list.Add(1);
int x = (int)list[0];  // boxing!

// Delegates
public delegate void Handler(string msg);
event Handler OnEvent;

// Properties (revolutionary at the time)
public string Name
{
    get { return _name; }
    set { _name = value; }
}
```

### C# 2.0 — Generics (2005, killer feature)

```csharp
// Type-safe collections
List<int> numbers = new List<int>();
numbers.Add(1);
int x = numbers[0];  // no cast, no boxing

// Generic methods
public T First<T>(IEnumerable<T> source) => source.First();

// Constraints
public T MaxOf<T>(T a, T b) where T : IComparable<T> =>
    a.CompareTo(b) > 0 ? a : b;
```

**Почему важно:** generics — тип-безопасные коллекции и алгоритмы без `object` boxing. Foundation для LINQ, ASP.NET Core, EF Core.

### Также в 2.0
- Nullable types (`int?`)
- Anonymous methods `delegate(int x) { return x * 2; }`
- Iterators `yield return`
- Partial classes

---

## 4. C# 3.0 — LINQ revolution (2007)

```csharp
// Lambda expressions
Func<int, int> square = x => x * x;

// Extension methods
public static class StringExtensions
{
    public static bool IsEmpty(this string s) => string.IsNullOrEmpty(s);
}
"".IsEmpty();  // true

// var (type inference)
var dict = new Dictionary<string, List<int>>();

// Object initializers
var person = new Person { Name = "John", Age = 30 };

// Anonymous types
var anon = new { Name = "John", Age = 30 };

// LINQ! — query syntax + method syntax
var query = from p in people
            where p.Age > 18
            select p.Name;

var query2 = people
    .Where(p => p.Age > 18)
    .Select(p => p.Name);
```

**Историческое значение:** LINQ — первый mainstream functional pattern в imperative языке. Идея пришла из F# / Haskell / SQL, гениально интегрирована в C#.

---

## 5. C# 4.0 — dynamic (2010)

```csharp
dynamic obj = GetSomething();
obj.Whatever();  // resolved at runtime

// COM Interop простел в разы
dynamic excel = Activator.CreateInstance(Type.GetTypeFromProgID("Excel.Application"));
excel.Workbooks.Add();

// Named/optional arguments
public void Method(string name, int age = 0, string? city = null) { }

Method(name: "John", city: "NYC");
```

**Зачем:** legacy Office automation, Python integration via IronPython.

> [!warning] Dynamic — не общая практика
> Используется только для interop / scripting. Type safety теряется.

---

## 6. C# 5.0 — async/await (2012, game-changer)

```csharp
// До 5.0 — callback hell
void DownloadData(string url, Action<string> onComplete)
{
    httpClient.BeginGetStringAsync(url, ar =>
    {
        var result = httpClient.EndGetString(ar);
        onComplete(result);
    });
}

// C# 5.0 — async/await
async Task<string> DownloadDataAsync(string url) =>
    await httpClient.GetStringAsync(url);

// Композиция
async Task ProcessAsync()
{
    var data1 = await DownloadAsync(url1);
    var data2 = await DownloadAsync(url2);
    return Combine(data1, data2);
}
```

**Почему революция:** async код выглядит как синхронный, но не блокирует thread. Compiler генерирует state machine из linear-looking code. Inspiration для async/await в JavaScript, Rust, Python, Kotlin coroutines.

См. [Async и Threading](async-threading.md).

---

## 7. C# 6.0 — Quality of Life (2015)

```csharp
// String interpolation
var greeting = $"Hello, {name}!";

// Null-conditional operator
var length = user?.Name?.Length ?? 0;

// Expression-bodied members
public string FullName => $"{FirstName} {LastName}";

// Auto-properties с initializer
public string Name { get; set; } = "Default";

// using static
using static System.Math;
var x = Sqrt(16);  // вместо Math.Sqrt

// nameof — refactor-safe strings
throw new ArgumentNullException(nameof(value));
```

**Тренд:** меньше boilerplate, expression-first style.

---

## 8. C# 7.0 — Tuples и pattern matching (2017)

```csharp
// Tuples (C# 7.0)
public (int min, int max) Range(int[] arr) => (arr.Min(), arr.Max());

var (min, max) = Range([1, 2, 3]);

// Pattern matching basics (C# 7.0)
public string Describe(object obj) => obj switch  // C# 8.0 syntax
{
    int i when i < 0 => "negative",
    int i => "non-negative",
    string s => $"string: {s}",
    _ => "unknown"
};

// Local functions
public int Process(int x)
{
    return Compute(x) + Compute(-x);
    
    int Compute(int n) => n * n;  // local function
}

// out variables (C# 7.0)
if (int.TryParse(input, out var number))
{
    Console.WriteLine(number);  // не нужно объявлять до if
}

// Span<T> (C# 7.2)
ReadOnlySpan<char> slice = "hello".AsSpan().Slice(0, 3);
```

См. [Span и Layout](../Runtime/span-layout.md).

---

## 9. C# 8.0 — Nullable + Async streams (2019)

```csharp
// Nullable reference types — turn on в csproj
#nullable enable

string name = null;        // Warning! string is non-nullable
string? maybeNull = null;  // OK

string FetchName()
{
    return null;  // Warning! return type is non-nullable
}

// Использование
void Print(string? name)
{
    Console.WriteLine(name.Length);  // Warning! possible null
    
    if (name is not null)
    {
        Console.WriteLine(name.Length);  // OK
    }
    
    Console.WriteLine(name?.Length ?? 0);  // OK
    Console.WriteLine(name!.Length);        // override warning (you take responsibility)
}
```

> [!warning] Nullable — warnings, не errors
> NRT не enforced runtime — `null` всё ещё может попасть. Это compile-time hint только.

### Async streams

```csharp
// IAsyncEnumerable<T>
public async IAsyncEnumerable<int> GenerateAsync(
    [EnumeratorCancellation] CancellationToken ct)
{
    for (int i = 0; ; i++)
    {
        await Task.Delay(100, ct);
        yield return i;
    }
}

// Consume
await foreach (var item in GenerateAsync(ct))
{
    Console.WriteLine(item);
    if (item > 100) break;
}
```

### Default interface methods

```csharp
public interface ILogger
{
    void Log(string message);
    
    // Default method — interface evolution без breaking changes
    void LogError(string error) => Log($"ERROR: {error}");
}
```

**Тренд:** safety + structured concurrency.

---

## 10. C# 9.0 — Records (2020, big shift to functional)

```csharp
// Positional records — immutable DTO в одну строку
public record Person(string Name, int Age);

var p = new Person("John", 30);
var p2 = p with { Age = 31 };  // copy with modification

// Value equality
new Person("John", 30) == new Person("John", 30);  // true!

// Top-level statements
// Program.cs
Console.WriteLine("Hello!");  // No Main!

// Init-only setters
public class User
{
    public string Name { get; init; } = "";  // только при construction
}

// Target-typed new
List<Person> people = new() { new("John", 30) };  // type inferred

// Pattern matching enhanced
if (obj is { Name.Length: > 5 } person) { /* ... */ }
```

**Тренд:** immutability, functional features, modern syntax.

См. [Functional C#](functional-csharp.md).

---

## 11. C# 10.0 — Cleanup (2021)

```csharp
// Global usings — в одном файле / в csproj
// GlobalUsings.cs
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using Microsoft.Extensions.DependencyInjection;

// Теперь во всех файлах не нужны эти using

// File-scoped namespaces — без extra indent
namespace MyApp.Services;

public class Service { }

// vs old:
// namespace MyApp.Services {
//     public class Service { }
// }

// Record structs (value-type records)
public record struct Point(int X, int Y);
// Stack-allocated, value equality, with expressions

// Lambda improvements
var lambda = (int x) => x + 1;          // explicit param types
Func<int, int> typed = x => x + 1;       // target-typed
[Required] (int x) => x + 1;             // attributes!

// Constant interpolated strings
const string PREFIX = "MyApp";
const string LOG_TAG = $"{PREFIX}.Logger";  // C# 10!
```

---

## 12. C# 11.0 — List patterns + Generic math (2022)

```csharp
// List patterns
public string Describe(int[] arr) => arr switch
{
    [] => "empty",
    [var x] => $"single: {x}",
    [_, _] => "two",
    [var first, .., var last] => $"first={first}, last={last}",
    [1, 2, 3] => "exactly 1,2,3",
    _ => "other"
};

// Raw string literals — без escaping!
var json = """
    {
        "name": "John",
        "path": "C:\Users\John"
    }
    """;

var multiline = """""
    Even strings with """ quotes work
    """"";  // 5 quotes outer

// Required members
public class User
{
    public required string Name { get; init; }
    public required string Email { get; init; }
}

var u = new User { Name = "John", Email = "x@y" };
// var u = new User();  // Error — required not set

// File-local types — internal в одном файле
file class Helper { /* ... */ }

// Generic math (static abstract в interfaces!)
public interface IAddable<TSelf> where TSelf : IAddable<TSelf>
{
    static abstract TSelf operator +(TSelf a, TSelf b);
}

public T Sum<T>(IEnumerable<T> items) where T : IAddable<T>, INumber<T>
{
    var sum = T.Zero;
    foreach (var item in items) sum += item;
    return sum;
}

Sum(new[] { 1, 2, 3 });        // 6
Sum(new[] { 1.5, 2.5 });        // 4.0
Sum(new[] { 1m, 2m });          // 3 (decimal)
```

**Generic math** — революция: Sum, Average, etc. могут работать с любым numeric типом без code duplication.

---

## 13. C# 12.0 — Primary constructors + Collection expressions (2023)

```csharp
// Primary constructors на классах (раньше только records)
public class OrderService(IOrderRepo repo, ILogger<OrderService> log)
{
    public Task<Order?> GetAsync(Guid id)
    {
        log.LogInformation("Fetching {Id}", id);
        return repo.GetByIdAsync(id);
    }
    
    // Поля доступны напрямую как captured из ctor
}

// Collection expressions — унифицированный синтаксис
int[] arr = [1, 2, 3];                    // array
List<int> list = [1, 2, 3];                // List
ImmutableArray<int> imm = [1, 2, 3];      // ImmutableArray
HashSet<int> set = [1, 2, 3];              // HashSet

// Spread
int[] more = [.. existing, 4, 5];
var combined = [..arr1, ..arr2, ..arr3];

// Alias any type
using Vec = (int X, int Y, int Z);

Vec MakeVec() => (1, 2, 3);

// Default lambda parameters
var greet = (string name = "World") => $"Hello, {name}!";
greet();          // "Hello, World!"
greet("John");    // "Hello, John!"

// `params ReadOnlySpan<T>` — high-performance varargs
public void Log(string format, params ReadOnlySpan<object?> args) { }
// Avoid object[] allocation!
```

---

## 14. C# 13.0 — Lock и params collections (2024)

```csharp
// New Lock type (.NET 9) — explicit lock с RAII
private readonly Lock _lock = new();

public void DoWork()
{
    using (_lock.EnterScope())
    {
        // critical section
    }
}

// params collections — больше вариантов
public void Print(params ReadOnlySpan<int> values) { }  // C# 12
public void Print(params IEnumerable<int> values) { }    // C# 13!
public void Print(params List<int> values) { }            // C# 13

Print(1, 2, 3);  // works for all overloads

// Partial properties (раньше только methods)
public partial class MyClass
{
    public partial string Name { get; set; }
}

public partial class MyClass
{
    public partial string Name 
    { 
        get => _backing; 
        set => _backing = value?.Trim() ?? ""; 
    }
}

// Implicit index в object initializers
var arr = new int[5] { [0] = 10, [4] = 50 };

// new escape sequence \e for ESC (0x1B)
Console.Write("\e[31mRed\e[0m");  // ANSI escape
```

---

## 15. C# 14.0 — Extension types и Type unions (2025/26 preview)

```csharp
// Extension types — добавляем поведение существующим типам без inheritance
extension type DistanceExtension for double
{
    public double Kilometers => this * 1000;
    public double Miles => this * 1609.34;
}

(2.5).Kilometers;  // 2500
3.0.Miles;         // 4828.02

// Implicit extension types — добавляют conversion
implicit extension type Celsius for double
{
    public Celsius ToFahrenheit() => new((this * 9 / 5) + 32);
}

// Type unions (preview!)
public T1 | T2 GetValueOrError<T1, T2>() { ... }

string | int Result(bool ok) => ok ? "good" : 42;

var r = Result(true);
if (r is string s) { /* string branch */ }
else if (r is int i) { /* int branch */ }

// Null-conditional assignment
user?.Name = "John";  // nothing if user is null

// readonly anonymous types
readonly var dto = new { Name = "John", Age = 30 };

// Field keyword в auto-properties (preview)
public string Name
{
    get => field?.Trim();  // 'field' refers to backing field
    set => field = value;
}
```

**Тренд C# 14:** structural extensions без inheritance, type-level expressiveness, безопасность.

---

## 16. Breaking changes и deprecation history

### .NET Framework → .NET Core/5+ (2020)

- `AppDomain` → `AssemblyLoadContext`
- `Configuration Manager (web.config)` → `IConfiguration`
- `WCF` → gRPC / REST
- `WebForms` → Razor Pages / Blazor
- `Remoting` → gRPC / IPC
- `Code Access Security` (CAS) — gone

### Patterns deprecated в C# evolution

| Раньше | Теперь |
|--------|--------|
| Mutable DTO classes | Records |
| `if (x != null && x.Length > 0)` | `if (x is { Length: > 0 })` |
| Synchronous over async (`.Result`) | `await` everywhere |
| `null` для absent value | Nullable reference types или `Option<T>` |
| `try/catch` для control flow | `Result<T, E>` |
| `static` utility classes | Extension methods |
| `Hashtable`, `ArrayList` | `Dictionary<K,V>`, `List<T>` |
| Fields вместо properties | Auto-properties |
| `WebClient` | `HttpClient` |
| `Console.WriteLine` для logging | `ILogger` |

### Compiler warnings → errors over time

- `CS8600-CS8625` (nullable warnings) — turn into errors в production code
- `CS1591` (XML doc) — для library projects
- `CA1062` — argument validation в public APIs
- `CA2007` — ConfigureAwait в library code

---

## 17. C# vs другие современные языки

### C# vs Kotlin (JVM-альтернатива)

| | C# | Kotlin |
|--|-----|--------|
| Null safety | Nullable annotation (warnings) | Built-in (compiler errors) |
| Coroutines | async/await | Coroutines (более structured) |
| Records | C# 9 records | Data classes |
| Sealed types | C# 17 sealed records hierarchy | Sealed classes (cleaner) |
| Smart casts | Pattern matching | Smart cast после `is` |
| Top-level functions | Static methods в classes | Just functions |
| Functional | LINQ | Excellent (стандарт library) |
| Performance | Native + AOT | JIT (slower startup) |

### C# vs TypeScript

| | C# | TypeScript |
|--|-----|-----------|
| Type system | Sound, static | Gradual, structural, **unsound** |
| Compilation | IL → JIT/AOT | JS (no runtime types) |
| Async | Task / async | Promise / async |
| Records | C# 9 | Object types (no equality) |
| Discriminated unions | Sealed records hierarchy | First-class (better) |
| Decorators | Attributes (richer) | Decorators |
| Browser native | ❌ (Blazor WASM) | ✅ |
| Performance | Native | V8 JS engine |

### C# vs Rust

| | C# | Rust |
|--|-----|------|
| Memory | GC | Ownership / borrowing (no GC) |
| Safety | Type-safe | Memory + thread safe |
| Performance | Good | Excellent (zero-cost abstractions) |
| Learning curve | Low-medium | Steep |
| Ecosystem | Massive | Growing |
| Use cases | Apps, services, ML, web | Systems, embedded, performance |
| Async | Task-based | Future-based, more explicit |

### Когда выбирать C#

✅ **C# лучше когда:**
- Нужен GC и productivity
- Microsoft / Azure ecosystem
- Mixed paradigm (OOP + functional)
- ASP.NET Core web apps
- Cross-platform desktop (Avalonia)
- ML.NET / scientific
- Game dev (Unity)

✅ **Другой язык лучше когда:**
- Browser native → TypeScript
- JVM ecosystem → Kotlin
- No GC, max performance → Rust / C++
- Functional first → F#
- Statistics → R / Python
- Mobile native → Swift / Kotlin

---

## 18. Что грядёт в C# 15+

### Confirmed roadmap

- **Type unions** (полноценная support)
- **Discriminated unions** (sealed types built-in)
- **Pipe operator** `|>` (как в F#)
- **Roles** — interfaces over existing types (как traits)
- **Dictionary expressions** — `{ ["key"] = value }`
- **Improved AOT** — больше features compatible

### Speculative

- **Effects** — controlled side effects (как Koka)
- **Parameterized modules**
- **Lightweight type aliases**

---

## 19. Языковая безопасность — overview

### Тип безопасности vs Production safety

| Уровень | Что проверяется | C# поддержка |
|---------|-----------------|--------------|
| Type safety | Типы совпадают | ✅ Strong |
| Null safety | Null dereference | ⚠️ Warnings (NRT) |
| Memory safety | No use-after-free | ✅ via GC |
| Thread safety | Race conditions | ❌ Programmer's job |
| Initialization safety | Required fields | ✅ `required` keyword |
| Pattern exhaustiveness | All cases covered | ⚠️ Partial (sealed types help) |
| Lifetime safety | Reference validity | ⚠️ via `ref` rules |

C# силён в type/memory safety, средний в null safety, слаб в thread safety (по сравнению с Rust).

---

## 20. Best Practices по эволюции

- **Use latest stable** — `<LangVersion>latest</LangVersion>` в csproj
- **Adopt nullable reference types** — incrementally в legacy code
- **Records для DTO/Events/Value Objects** — стандарт с C# 9
- **Pattern matching > if/else chains** — readability + correctness
- **Primary constructors для simple cases** — меньше boilerplate
- **Collection expressions** — `[1, 2, 3]` вместо `new[] { 1, 2, 3 }`
- **Top-level statements** для CLI и tests
- **File-scoped namespaces** — стандарт после C# 10
- **Global usings** — для commonly used in solution
- **Required members** для critical-init properties
- **Generic math** для math libraries
- **Extension types** (C# 14+) для добавления behavior без inheritance
- **Не используй `dynamic`** если можно типизировано
- **Не используй `ArrayList` / `Hashtable`** — generics есть с 2005

### Migration guide для legacy

```
Phase 1 (легко):
- Update <LangVersion> и target framework
- File-scoped namespaces (auto-fix через VS)
- Primary ctors для классов с простым DI

Phase 2 (средне):
- DTO → Records
- if/else цепи → switch expressions
- Extract to extension methods

Phase 3 (сложно):
- Включить nullable reference types (по проектам)
- Result<T, E> вместо exceptions для business logic
- Functional core / imperative shell
```

---

## 21. Common Misconceptions

### "C# = .NET Framework"

❌ С 2014 — .NET Core, потом .NET 5+. Cross-platform, open-source, JIT/AOT. .NET Framework 4.8 — legacy (не deprecated, но не развивается).

### "Async всегда быстрее"

❌ Async добавляет overhead (state machine, allocations). Для CPU-bound работы — `Parallel.ForEach`. Для I/O-bound — async wins.

### "Generics = C++ templates"

❌ В C# generics реализованы через **runtime type erasure** для ref types и **runtime specialization** для value types. C++ templates — pure compile-time generation. C# generics безопаснее, но C++ — мощнее (any type, even literals).

### "C# медленнее C++"

⚠️ Зависит. На typical business app — близко. На SIMD / hand-tuned C++ — C++ всё ещё быстрее. С Native AOT и Vector intrinsics — C# стало competitive.

### "Записи (records) — immutable forever"

❌ Records по умолчанию immutable, но можно сделать mutable через `set`. Reference type record можно изменять через captured reference. Только value-type records (record struct) — действительно immutable.

---

## См. также

- [Modern C# Features](modern-features.md) — детали features
- [Functional C#](functional-csharp.md) — records, pattern matching
- [OOP](oop.md) — classes, interfaces evolution
- [Async и Threading](async-threading.md) — async/await deep
- [Reflection и Expression Trees](reflection-expression-trees.md) — generic math support
- [Native AOT](../AspNetCore/native-aot.md) — современный compilation

## Reading list

- **Mads Torgersen — C# Language Design** — github.com/dotnet/csharplang (proposals)
- **Microsoft Docs — What's new in C#** — learn.microsoft.com/dotnet/csharp/whats-new
- **CLR via C#** — Jeffrey Richter (deep CLR/C# internals)
- **C# in Depth** — Jon Skeet (язык в деталях, обновляется к каждой major version)
- **Anders Hejlsberg interviews** — youtube (insights от author)
- **C# Language Spec** — github.com/dotnet/csharpstandard
- **Jared Parsons blog** — github.com/jaredpar (C# compiler engineer)
- **Mads Torgersen — language design talks** — youtube
- **Stephen Toub — performance series** — devblogs.microsoft.com/dotnet
