---
tags: [csharp, functional, senior, lambda, linq, immutability, monad, composition, fp]
level: Senior
date: 2026-08-02
---

# Functional C# — функциональные техники в OOP-языке

> **Pure functions, immutability, higher-order functions, monads (`Option<T>`, `Result<T>`), function composition, partial application.** Как применять FP-style в C# для cleaner code, less bugs, better testability. Закрывает пробел: «знаю про LINQ, не понимаю когда `Option<T>` лучше null, и что такое Bind/Map в практике».

---

## 0. Как читать

Если впервые — раздел 1→3 (mental model + pure functions). Monads — раздел 5→6. Production guidance — раздел 9 (best practices), 11 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. FP в C#

C# — multi-paradigm OOP-first язык. Functional features добавлялись incrementally:

```
C# 3 (2008) — LINQ, lambda, expression trees, extension methods
C# 6 — null-conditional ?., expression-bodied members
C# 7 — tuples, deconstruction, pattern matching
C# 8 — switch expression, pattern matching enhancements
C# 9 — records, init, top-level statements
C# 10 — record struct, with-expression on structs
C# 11 — list patterns, required, raw strings
C# 12 — primary constructors, collection expressions
```

C# 2024+ — much more functional-friendly чем C# 1.0.

### 1.2. Функциональные принципы

```
1. Pure functions
   - Same input → same output
   - No side effects (no mutation, no I/O)

2. Immutability
   - Data не changes после creation
   - "Updates" produce new objects (with-expression)

3. First-class functions
   - Functions as values (Func<T>, Action<T>)
   - Higher-order functions (take/return functions)

4. Composition
   - Build complex from simple
   - f >> g = x => g(f(x))

5. Declarative
   - Describe what, not how
   - LINQ vs manual loops
```

### 1.3. Когда FP оправдан

```
✅ Use FP когда:
  - Data transformation (LINQ pipelines)
  - Immutable domain models (Value Objects, DTOs)
  - Concurrent code (no shared mutable state)
  - Testable business logic (pure functions)
  - Composable validation

❌ Не используй когда:
  - UI imperative state machines (event-driven)
  - Performance critical hot paths (FP overhead)
  - Heavy I/O (effects unavoidable)
  - Team unfamiliar (readability suffers)
```

### 1.4. Главное правило

```
Pragmatic FP в C#:
  - Pure functions для business rules
  - Records для Value Objects
  - LINQ для data transformations
  - Optional / Result для error handling
  - Immutability where possible
  - Mutation where necessary

Не F# converts. Use FP techniques, не replace OOP entirely.
```

### 1.5. F# — full FP альтернатива

Если хочешь true functional-first language на .NET — F#:

```fsharp
type Person = { Name: string; Age: int }
let p = { Name = "Alice"; Age = 30 }
let updated = { p with Name = "Bob" }

let add a b = a + b
let increment = add 1   // partial application
let nums = [1..10] |> List.map increment |> List.filter (fun x -> x > 5)
```

C# борется с verbosity vs F# — но C# closes gap (records, pattern matching, primary ctors).

> [!info]- Если ты знаешь Haskell / F# / Scala / TypeScript / Rust
> **Haskell:** pure FP, lazy by default, type classes. C# borrows ideas, но runtime-side effects allowed.
>
> **F#:** .NET FP cousin. Shares CLR. C# и F# interop seamless. F# more concise для FP, C# more familiar.
>
> **Scala:** OOP+FP hybrid like C#, но более functional-leaning. Type system stronger (HKT).
>
> **TypeScript:** structural typing + FP popular (Ramda, lodash/fp). Similar to C# в practical FP usage.
>
> **Rust:** ownership + traits + Option/Result — FP-influenced. C# `Result<T,E>` emulates Rust.

> [!question]- Интервью: можно ли писать functional code в C#?
> Да, C# 2024+ supports many FP techniques: 1) **Lambda + LINQ** — first-class functions, declarative pipelines. 2) **Records** — immutable Value Objects. 3) **Pattern matching** — exhaustive type dispatch. 4) **`Func<T>` / `Action<T>`** — higher-order functions. 5) **Expression trees** — code as data. **However**: C# не pure FP — side effects, mutation, exceptions allowed. Pragmatic mix: FP для data transformation + business rules, OOP для structure + state. **F#** — true FP-first .NET language. **C# closes gap** с every version. Practical advice: FP techniques selectively (pure functions, immutability, LINQ) — не replace OOP.

---

## 2. Pure functions

### 2.1. Что pure

```csharp
// ✅ Pure — same input → same output, no side effects
public static int Add(int a, int b) => a + b;
public static decimal CalculateTax(decimal amount, decimal rate) => amount * rate;
public static string Greet(string name) => $"Hello, {name}!";

// ❌ Impure — side effect (I/O)
public static void PrintGreeting(string name) =>
    Console.WriteLine($"Hello, {name}!");

// ❌ Impure — depends on global state
private static int _counter = 0;
public static int IncrementCounter() => ++_counter;

// ❌ Impure — mutation
public static List<int> AppendOne(List<int> list)
{
    list.Add(1);   // mutates argument
    return list;
}
```

### 2.2. Преимущества pure

```
✅ Easy to test — no setup, just inputs/outputs
✅ Predictable — given X, always Y
✅ Composable — combine fearlessly
✅ Cacheable / memoizable
✅ Parallel-safe (no shared state)
✅ Reasoning easier — no hidden dependencies

✅ Real-world impact:
  - Bug rates lower
  - Refactoring safer
  - Concurrency simpler
```

### 2.3. Push side effects to edges

```csharp
// ❌ Mixed pure + impure
public class OrderService
{
    public void ProcessOrder(int orderId)
    {
        var order = _db.Orders.Find(orderId);   // I/O
        var tax = order.Total * 0.08m;           // pure
        var withTax = order.Total + tax;         // pure
        order.Total = withTax;                    // mutation
        _db.SaveChanges();                        // I/O
    }
}

// ✅ Separate pure logic from I/O
public static class TaxCalculator
{
    public static decimal CalculateTotal(decimal subtotal, decimal taxRate) =>
        subtotal + (subtotal * taxRate);   // pure
}

public class OrderService
{
    public async Task ProcessOrderAsync(int orderId)
    {
        var order = await _db.Orders.FindAsync(orderId);   // I/O at edge
        var newTotal = TaxCalculator.CalculateTotal(order.Total, 0.08m);   // pure
        order.SetTotal(newTotal);
        await _db.SaveChangesAsync();   // I/O at edge
    }
}
```

Functional core / imperative shell — pure logic isolated, I/O at boundaries.

### 2.4. Memoization

```csharp
public static Func<TArg, TResult> Memoize<TArg, TResult>(Func<TArg, TResult> fn)
    where TArg : notnull
{
    var cache = new ConcurrentDictionary<TArg, TResult>();
    return arg => cache.GetOrAdd(arg, fn);
}

Func<int, int> expensiveCalc = n =>
{
    Thread.Sleep(1000);   // simulated cost
    return n * n;
};

var memoized = Memoize(expensiveCalc);
memoized(5);   // 1 second
memoized(5);   // immediate — cached
memoized(6);   // 1 second
```

Memoization works only для pure functions.

### 2.5. Testing pure

```csharp
// Pure — easy test
[Test]
public void CalculateTotal_AddsTax()
{
    var result = TaxCalculator.CalculateTotal(100, 0.08m);
    Assert.That(result, Is.EqualTo(108m));
}

// vs Impure — needs mocks, setup
[Test]
public async Task ProcessOrder_AppliesTax()
{
    var dbMock = new Mock<DbContext>();
    var orderMock = new Order { Id = 1, Total = 100 };
    dbMock.Setup(d => d.Orders.FindAsync(1)).ReturnsAsync(orderMock);
    
    var service = new OrderService(dbMock.Object);
    await service.ProcessOrderAsync(1);
    
    dbMock.Verify(d => d.SaveChangesAsync(), Times.Once);
    // ... more setup
}
```

> [!question]- Интервью: что такое pure function и почему важна?
> Function где: 1) **Same input → same output** (deterministic). 2) **No side effects** (no I/O, no mutation, no global state). **Examples**: `Add(2, 3)` always returns 5. **Counter-examples**: `DateTime.Now` (changes), `Random.Next()` (non-deterministic), `File.ReadAllText` (I/O), method that mutates argument. **Why important**: 1) **Testable** without mocks. 2) **Memoizable** (cache results). 3) **Parallel-safe** (no shared state). 4) **Reasoning** — given X always Y. 5) **Composable** fearlessly. **C# practice**: separate pure business logic from I/O ("functional core, imperative shell"). I/O at boundaries (controllers, repositories), pure logic в middle.

---

## 3. Immutability

### 3.1. Records (C# 9+)

```csharp
public record User(int Id, string Name, string Email);

var u1 = new User(1, "Alice", "a@x.com");
var u2 = u1 with { Name = "Bob" };   // new instance, не mutation

u1.Name;   // "Alice" — unchanged
u2.Name;   // "Bob"
```

См. [[modern-features]] раздел 2.

### 3.2. Immutable collections

```csharp
using System.Collections.Immutable;

ImmutableList<int> list = ImmutableList.Create(1, 2, 3);
ImmutableList<int> updated = list.Add(4);   // new list

list.Count;     // 3
updated.Count;  // 4

// Immutable dictionary
ImmutableDictionary<string, int> dict = ImmutableDictionary<string, int>.Empty
    .Add("a", 1)
    .Add("b", 2);
var newDict = dict.SetItem("a", 100);   // new dict

// Immutable HashSet
ImmutableHashSet<int> set = ImmutableHashSet.Create(1, 2, 3);
```

`System.Collections.Immutable` — efficient persistent data structures.

### 3.3. ReadOnly views

```csharp
List<int> mutable = new() { 1, 2, 3 };
IReadOnlyList<int> view = mutable;   // view, не copy

// view не can mutate
// view.Add(4);   // compile error

// But underlying still mutable
mutable.Add(4);
view.Count;   // 4 — view sees changes
```

`IReadOnlyList<T>`/`IReadOnlyCollection<T>` — interfaces для read-only access. Не deep immutability — view над mutable.

### 3.4. Frozen collections (.NET 8+)

```csharp
using System.Collections.Frozen;

FrozenDictionary<string, int> frozen = source.ToFrozenDictionary();
FrozenSet<string> frozenSet = source.ToFrozenSet();

// Initialization slow, lookup faster than regular Dictionary
// Use case: build once, query many times
```

`Frozen*` — read-optimized immutable collections. Faster than ImmutableDictionary для read-only scenarios.

### 3.5. Records деструктуризация

```csharp
public record Point(double X, double Y);

var p = new Point(3.0, 4.0);
var (x, y) = p;   // deconstruction
double distance = Math.Sqrt(x * x + y * y);
```

### 3.6. Mutation через transformation

```csharp
// Imperative — mutation
var users = new List<User>();
foreach (var u in source)
{
    if (u.IsActive) users.Add(new User(u.Id, u.Name.ToUpper(), u.Email));
}

// Functional — transformation
var users = source
    .Where(u => u.IsActive)
    .Select(u => u with { Name = u.Name.ToUpper() })
    .ToList();
```

LINQ + records — functional pipeline.

### 3.7. Когда immutability cost

```
Cost:
- Allocation per change (new object)
- More memory
- GC pressure

When acceptable:
- Domain entities (correctness > perf)
- DTOs (transient)
- Configuration (rarely changes)

When too expensive:
- Hot path tight loops (use mutation)
- Massive collections (use ImmutableList с structural sharing, или mutate locally)
- Performance critical code
```

> [!question]- Интервью: какие альтернативы Mutation в C#?
> 1) **`record`** + `with` expression — non-destructive update. `var u2 = u1 with { Name = "Bob" }`. 2) **`init` properties** — settable только через constructor / object initializer. После — read-only. 3) **`ImmutableList<T>` / `ImmutableDictionary<TK, TV>`** — persistent data structures. Add/Remove returns new collection. 4) **`FrozenDictionary<TK, TV>`** (.NET 8+) — read-optimized immutable. 5) **Functional pipeline** через LINQ — transformation instead of mutation. 6) **`ReadOnlyCollection<T>`** — view (не deep immutable). **Cost**: allocation per change. **Worth**: testability, concurrency safety, reasoning. **Pragmatic**: immutability where correctness matters, mutation в hot paths.

---

## 4. Higher-order functions

### 4.1. Functions as values

```csharp
// Function stored in variable
Func<int, int> square = x => x * x;
Func<int, int, int> add = (a, b) => a + b;
Action<string> log = msg => Console.WriteLine(msg);

// Pass as argument
public static int Apply(int x, Func<int, int> f) => f(x);
Apply(5, square);   // 25
Apply(5, x => x + 10);   // 15

// Return from function
public static Func<int, int> CreateMultiplier(int factor) =>
    x => x * factor;

var times3 = CreateMultiplier(3);
times3(5);   // 15
```

### 4.2. LINQ — full of HOFs

```csharp
var result = numbers
    .Where(x => x > 0)         // takes predicate function
    .Select(x => x * 2)        // takes mapper function
    .OrderBy(x => x)           // takes key selector
    .Aggregate(0, (a, b) => a + b);   // takes reducer function

// All these are higher-order functions (HOFs)
```

### 4.3. Custom HOFs

```csharp
public static class Funcs
{
    public static Func<T, T> Compose<T>(Func<T, T> f, Func<T, T> g) =>
        x => g(f(x));
    
    public static Func<TArg, TResult> Apply<TArg, TArgPartial, TResult>(
        Func<TArgPartial, TArg, TResult> fn,
        TArgPartial partialArg) =>
        arg => fn(partialArg, arg);
    
    public static Action<T> Tap<T>(Action<T> sideEffect) => x =>
    {
        sideEffect(x);
    };
}

// Compose
Func<int, int> plus1 = x => x + 1;
Func<int, int> times2 = x => x * 2;
var combined = Funcs.Compose(plus1, times2);   // (x+1)*2
combined(5);   // 12

// Tap — side effect в pipeline
var data = source
    .Select(x => x * 2)
    .Select(x => { Console.WriteLine(x); return x; })   // tap pattern
    .ToList();
```

### 4.4. Function composition

```csharp
public static class FuncExtensions
{
    // f >> g = x => g(f(x))
    public static Func<TIn, TOut2> Compose<TIn, TMid, TOut2>(
        this Func<TIn, TMid> first,
        Func<TMid, TOut2> second) =>
        x => second(first(x));
}

// Use
Func<string, string> trim = s => s.Trim();
Func<string, string> upper = s => s.ToUpper();
Func<string, int> length = s => s.Length;

var pipeline = trim.Compose(upper).Compose(length);
pipeline("  hello  ");   // 5

// Pipeline operator (style — F# style)
"  hello  ".Trim().ToUpper().Length;   // C# uses method chaining
```

C# не имеет F# `|>` operator natively. Method chaining — closest equivalent.

### 4.5. Partial application

```csharp
public static class Partial
{
    public static Func<T2, TResult> Apply<T1, T2, TResult>(
        this Func<T1, T2, TResult> fn,
        T1 t1) =>
        t2 => fn(t1, t2);
}

Func<int, int, int> add = (a, b) => a + b;
var increment = add.Apply(1);   // partial application
increment(5);   // 6
increment(10);  // 11
```

### 4.6. Currying

```csharp
public static class Curry
{
    public static Func<T1, Func<T2, TResult>> Curry<T1, T2, TResult>(
        this Func<T1, T2, TResult> fn) =>
        t1 => t2 => fn(t1, t2);
}

Func<int, int, int> add = (a, b) => a + b;
var curried = add.Curry();
var add5 = curried(5);
add5(3);   // 8
```

C# rarely uses true currying (verbose). F# has natural curry. C# works с partial application.

> [!question]- Интервью: что такое higher-order function?
> Function которая **takes function как parameter** или **returns function**. Examples: 1) **`LINQ.Where(Func<T, bool>)`** — takes predicate. 2) **`Select(Func<T, U>)`** — takes mapper. 3) **`Aggregate(seed, Func<TAcc, T, TAcc>)`** — takes reducer. 4) **`CreateMultiplier(int factor)` returning `Func<int, int>`** — function factory. **Use cases**: 1) **Strategy pattern** through delegate. 2) **Composable pipelines** (LINQ, middleware). 3) **Partial application** (specialized functions из general). 4) **Composition** (`f.Compose(g)`). **Real-world**: ASP.NET Core middleware, LINQ, RX (Reactive Extensions), validation chains. C# supports HOFs through `Func<T>` / `Action<T>` / `Predicate<T>`.

---

## 5. Option / Maybe pattern

### 5.1. Проблема null

```csharp
// ❌ Null fragile
public User? FindUser(int id) =>
    _users.FirstOrDefault(u => u.Id == id);

var user = FindUser(42);
Console.WriteLine(user.Name);   // NRE if not found!

// NRT helps но не enforces
var user = FindUser(42);
Console.WriteLine(user?.Name ?? "Unknown");   // verbose
```

### 5.2. `Option<T>` — Maybe Monad

```csharp
public abstract record Option<T>
{
    public sealed record Some(T Value) : Option<T>;
    public sealed record None : Option<T>;
    
    public static Option<T> Of(T? value) =>
        value is null ? new None() : new Some(value);
    
    public static implicit operator Option<T>(T value) => new Some(value);
    
    public bool IsSome => this is Some;
    public bool IsNone => this is None;
    
    public T GetValueOrDefault(T fallback) => this switch
    {
        Some s => s.Value,
        _ => fallback
    };
    
    public Option<U> Map<U>(Func<T, U> mapper) => this switch
    {
        Some s => new Option<U>.Some(mapper(s.Value)),
        _ => new Option<U>.None()
    };
    
    public Option<U> Bind<U>(Func<T, Option<U>> binder) => this switch
    {
        Some s => binder(s.Value),
        _ => new Option<U>.None()
    };
    
    public TResult Match<TResult>(Func<T, TResult> some, Func<TResult> none) => this switch
    {
        Some s => some(s.Value),
        _ => none()
    };
}
```

### 5.3. Использование Option

```csharp
public Option<User> FindUser(int id)
{
    var user = _users.FirstOrDefault(u => u.Id == id);
    return Option<User>.Of(user);
}

// Functional chain
var nameLength = FindUser(42)
    .Map(u => u.Name)
    .Map(name => name.Length)
    .GetValueOrDefault(0);

// Pattern matching
var message = FindUser(42).Match(
    some: u => $"Found {u.Name}",
    none: () => "Not found");

// With Bind (chainable)
public Option<Address> GetUserAddress(int id) =>
    FindUser(id).Bind(u => u.Address is null
        ? new Option<Address>.None()
        : new Option<Address>.Some(u.Address));
```

### 5.4. LanguageExt library

```xml
<PackageReference Include="LanguageExt.Core" Version="4.4.9" />
```

```csharp
using LanguageExt;
using static LanguageExt.Prelude;

Option<User> user = Some(new User { Id = 1, Name = "Alice" });
Option<User> noUser = None;

var name = user.Map(u => u.Name).IfNone("Unknown");

// Чейн через bind
var result = user
    .Bind(u => u.Email is null ? Option<string>.None : Some(u.Email))
    .Map(email => email.ToUpper());
```

LanguageExt — most popular FP library для C#. Provides `Option<T>`, `Either<L, R>`, `Try<T>`, etc.

### 5.5. NRT vs `Option<T>`

```
NRT (T?):
✅ Built-in language feature
✅ No external dependency
✅ Compile-time warnings
❌ Best-effort (holes)
❌ No composition (Map/Bind)

Option<T>:
✅ True functional approach
✅ Composable (Map/Bind/Match)
✅ Forces handling
❌ Adds dependency (LanguageExt)
❌ Verbose syntax sometimes
```

NRT — pragmatic для most codebases. `Option<T>` — true FP когда composition heavy.

### 5.6. Option в practice

```csharp
public Option<User> FindByEmail(string email) => /* ... */;
public Option<Order> GetActiveOrder(User user) => /* ... */;

// Chain — return None если any step fails
var totalDue = FindByEmail("a@x.com")
    .Bind(GetActiveOrder)
    .Map(order => order.Total);

// totalDue : Option<decimal>
```

> [!question]- Интервью: чем `Option<T>` отличается от `T?`?
> **`T?`** (NRT) — language feature, **opt-in compile-time** null safety. Warnings, не errors. Holes (generics, reflection). **`Option<T>`** — type explicitly representing Some(value) | None. **Forces handling** (no NullReferenceException possible). **Composable** через Map/Bind/Match. **Comparison**: NRT — pragmatic для C# codebases (built-in, familiar). `Option<T>` — true FP approach (forces explicit handling, composable). LanguageExt library popular для production. **Pragmatic choice**: NRT для most code, `Option<T>` когда composing many maybe-failing operations (e.g., user lookup → address lookup → coordinate parse). Combine: NRT для simple, `Option<T>` для chains.

---

## 6. Result / Either — error handling

### 6.1. Проблема exceptions

```csharp
// Exceptions — hidden control flow
public User GetUser(int id)
{
    if (id <= 0) throw new ArgumentException();
    var user = _db.Find(id);
    if (user is null) throw new NotFoundException();
    if (!user.IsActive) throw new UserBannedException();
    return user;
}

// Caller doesn't see what can throw
var user = GetUser(42);   // ?
```

### 6.2. `Result<T, E>`

```csharp
public abstract record Result<T, E>
{
    public sealed record Success(T Value) : Result<T, E>;
    public sealed record Failure(E Error) : Result<T, E>;
    
    public bool IsSuccess => this is Success;
    public bool IsFailure => this is Failure;
    
    public Result<U, E> Map<U>(Func<T, U> mapper) => this switch
    {
        Success s => new Result<U, E>.Success(mapper(s.Value)),
        Failure f => new Result<U, E>.Failure(f.Error),
        _ => throw new InvalidOperationException()
    };
    
    public Result<U, E> Bind<U>(Func<T, Result<U, E>> binder) => this switch
    {
        Success s => binder(s.Value),
        Failure f => new Result<U, E>.Failure(f.Error),
        _ => throw new InvalidOperationException()
    };
    
    public TResult Match<TResult>(
        Func<T, TResult> success,
        Func<E, TResult> failure) => this switch
    {
        Success s => success(s.Value),
        Failure f => failure(f.Error),
        _ => throw new InvalidOperationException()
    };
}
```

### 6.3. Использование Result

```csharp
public enum FindUserError { InvalidId, NotFound, Banned }

public Result<User, FindUserError> FindUser(int id)
{
    if (id <= 0) return new Result<User, FindUserError>.Failure(FindUserError.InvalidId);
    
    var user = _users.FirstOrDefault(u => u.Id == id);
    if (user is null) return new Result<User, FindUserError>.Failure(FindUserError.NotFound);
    if (!user.IsActive) return new Result<User, FindUserError>.Failure(FindUserError.Banned);
    
    return new Result<User, FindUserError>.Success(user);
}

// Caller forced to handle
var result = FindUser(42);
var message = result.Match(
    success: u => $"Hello {u.Name}",
    failure: e => e switch
    {
        FindUserError.InvalidId => "Invalid ID",
        FindUserError.NotFound => "User not found",
        FindUserError.Banned => "User banned",
        _ => "Unknown error"
    });
```

### 6.4. Railway-Oriented Programming

```csharp
// Сложный flow через Bind chains
public Result<Order, OrderError> PlaceOrder(OrderRequest request) =>
    ValidateRequest(request)
        .Bind(GetCustomer)
        .Bind(CheckInventory)
        .Bind(CalculatePricing)
        .Bind(ChargePayment)
        .Bind(SaveOrder);

// Each step может fail — chain stops, error propagated
// Success path "stays on rails" through transformations
```

Railway-oriented programming (Scott Wlaschin) — composable error handling.

### 6.5. LanguageExt `Either<L, R>`

```csharp
using LanguageExt;
using static LanguageExt.Prelude;

Either<string, int> Divide(int a, int b) =>
    b == 0 ? Left("Division by zero") : Right(a / b);

var result = Divide(10, 2)
    .Map(x => x * 2)
    .Bind(x => Divide(x, 0))   // chains, fails if any step fails
    .IfLeft(0);
```

`Either<L, R>` — left is error, right is success. Equivalent Result pattern.

### 6.6. Result vs Exception

```
Exception:
✅ Established C# pattern
✅ Stack trace automatic
✅ Easy для unrecoverable errors
❌ Hidden control flow
❌ Performance cost (10,000+ ns)
❌ Can be ignored (silent swallow)

Result<T, E>:
✅ Explicit error в signature
✅ Forces handling
✅ Composable (Bind/Map)
✅ Performance fast
❌ Verbose syntax
❌ Library dependency for nice APIs
```

Best practice: **Result для expected errors** (validation, business rules, "not found"), **Exception для unexpected** (I/O failure, programming bugs).

> [!question]- Интервью: Railway-Oriented Programming?
> Coined by Scott Wlaschin (F#). **Pattern**: chain operations through `Bind` / `Map` где each step может succeed (continue) или fail (short-circuit, error propagates). **Like railway**: success — main track, failure — switches to error track, stays там. **C# implementation** через `Result<T, E>` или `Either<L, R>`. **Example**: `ValidateRequest → GetCustomer → CheckInventory → ChargePayment → SaveOrder`. Each returns `Result<T, Error>`. Bind chains them. If any fails — chain stops, error propagated to caller. **Benefits**: cleaner than try/catch nesting, explicit error handling, composable. **F# native syntax** (`>>=`, `bind`) — C# verbose но достижимо. Library: LanguageExt (`Either<L, R>`).

---

## 7. Functional patterns

### 7.1. Pipeline pattern

```csharp
public static class Pipeline
{
    public static T2 Pipe<T1, T2>(this T1 source, Func<T1, T2> fn) => fn(source);
}

// Use
var result = "  hello world  "
    .Pipe(s => s.Trim())
    .Pipe(s => s.ToUpper())
    .Pipe(s => s.Length);
// 11
```

C# не имеет `|>` operator из F#, но extension methods + chain work similar.

### 7.2. Functor (Map)

```csharp
// Anything с Map — functor
// Option<T>, Result<T>, IEnumerable<T>, Task<T>

// Examples
Option<int>.Some(5).Map(x => x * 2);              // Option<int> Some(10)
Result<int, string>.Success(5).Map(x => x * 2);   // Result Success(10)
list.Select(x => x * 2);                           // IEnumerable Map
await task.ContinueWith(t => t.Result * 2);        // Task Map
```

Functor — abstraction для "container с Map operation".

### 7.3. Monad (Bind)

```csharp
// Anything с Bind — monad
// Option<T>, Result<T>, IEnumerable<T> (SelectMany), Task<T>

Option<User> user = FindUser(42);
Option<Address> address = user.Bind(u =>
    u.Address is null
        ? Option<Address>.None
        : Option<Address>.Some(u.Address));

// SelectMany — IEnumerable monadic Bind
List<int> numbers = new() { 1, 2, 3 };
List<int> doubled = numbers.SelectMany(x => new[] { x, x * 2 }).ToList();
// [1, 2, 2, 4, 3, 6]
```

Monad — Functor + Bind. Allows chaining computations that may fail или have effects.

### 7.4. LINQ как monad syntax

```csharp
// IEnumerable<T> — monad. Select = Map, SelectMany = Bind.
var users = new[] { user1, user2, user3 };

// Query syntax — sugar для SelectMany / Select / Where
var emails = from user in users
             where user.IsActive
             from email in user.Emails   // SelectMany — Bind!
             select email;

// Equivalent
var emails2 = users
    .Where(u => u.IsActive)
    .SelectMany(u => u.Emails);
```

LINQ query syntax — C# monadic syntax (similar к Haskell `do` notation, F# computation expressions).

### 7.5. Validation applicative

```csharp
// Multiple errors collected vs short-circuit
public abstract record Validation<T>
{
    public sealed record Valid(T Value) : Validation<T>;
    public sealed record Invalid(List<string> Errors) : Validation<T>;
    
    public Validation<U> Map<U>(Func<T, U> mapper) => this switch
    {
        Valid v => new Validation<U>.Valid(mapper(v.Value)),
        Invalid i => new Validation<U>.Invalid(i.Errors),
        _ => throw new InvalidOperationException()
    };
    
    // Apply combines two Validations, accumulating errors
    public Validation<TResult> Apply<TResult>(
        Validation<Func<T, TResult>> fn) => (this, fn) switch
    {
        (Valid v, Validation<Func<T, TResult>>.Valid f) => new Validation<TResult>.Valid(f.Value(v.Value)),
        (Invalid i, Validation<Func<T, TResult>>.Invalid fi) => new Validation<TResult>.Invalid(i.Errors.Concat(fi.Errors).ToList()),
        (Invalid i, _) => new Validation<TResult>.Invalid(i.Errors),
        (_, Validation<Func<T, TResult>>.Invalid fi) => new Validation<TResult>.Invalid(fi.Errors),
        _ => throw new InvalidOperationException()
    };
}

// Use — collects ALL validation errors
public Validation<User> ValidateUser(string name, int age, string email)
{
    var nameVal = !string.IsNullOrEmpty(name)
        ? new Validation<string>.Valid(name)
        : new Validation<string>.Invalid(new() { "Name required" });
    
    var ageVal = age > 0
        ? new Validation<int>.Valid(age)
        : new Validation<int>.Invalid(new() { "Age must be positive" });
    
    var emailVal = email.Contains("@")
        ? new Validation<string>.Valid(email)
        : new Validation<string>.Invalid(new() { "Invalid email" });
    
    // ... combine with Apply
}
```

`Validation` — applicative functor accumulates errors. `Result` short-circuits.

### 7.6. Functional composition в pipelines

```csharp
public static class Func
{
    public static Func<T, TOut2> Then<T, TMid, TOut2>(
        this Func<T, TMid> first,
        Func<TMid, TOut2> next) =>
        x => next(first(x));
}

// Build pipeline
Func<string, string> trim = s => s.Trim();
Func<string, string> lower = s => s.ToLower();
Func<string, string[]> split = s => s.Split(' ');

var process = trim.Then(lower).Then(split);

string[] words = process("  Hello World  ");
// ["hello", "world"]
```

> [!question]- Интервью: что такое monad?
> Type с **two operations**: 1) **`Return`** (constructor) — wrap value: `T → M<T>`. 2) **`Bind`** (flatMap) — chain operations: `M<T> + (T → M<U>) → M<U>`. **Examples в C#**: 1) **`IEnumerable<T>`** — `SelectMany` is bind. 2) **`Task<T>`** — `ContinueWith` (or async/await syntax). 3) **`Option<T>`** — Maybe monad. 4) **`Result<T, E>`** — Either monad. **Why**: composability. Chain operations that may have effects (failure, async, multiple values) without unwrapping/checking each step. **Mathematical laws**: identity, associativity. **Pragmatic в C#**: don't worry о laws, use Map/Bind для composing. LINQ query syntax — C#'s monad syntax (similar Haskell `do`-notation).

---

## 8. F# influence

### 8.1. F# vs C# functional

```
F# offers:
✅ Discriminated unions
✅ Pattern matching deeper
✅ Record types (since 1.0)
✅ Type providers (DB schemas)
✅ Computation expressions (custom monad syntax)
✅ Pipe operator |>
✅ Forward composition >>
✅ Hindley-Milner type inference
✅ Immutability default

C# closes gap:
- Records (C# 9+)
- Pattern matching (C# 8+)
- Init properties (C# 9+)
- Discriminated unions (preview в C# 15 / .NET 11, GA ~ноябрь 2026)
```

### 8.2. F# example (для context)

```fsharp
type Shape =
    | Circle of float
    | Square of float
    | Rectangle of float * float

let area shape =
    match shape with
    | Circle r -> System.Math.PI * r * r
    | Square s -> s * s
    | Rectangle (w, h) -> w * h

let pipeline = [1..10]
    |> List.filter (fun x -> x > 5)
    |> List.map (fun x -> x * 2)
    |> List.sum
```

### 8.3. C# equivalent

```csharp
public abstract record Shape;
public record Circle(double Radius) : Shape;
public record Square(double Side) : Shape;
public record Rectangle(double Width, double Height) : Shape;

public static double Area(Shape shape) => shape switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Square s => s.Side * s.Side,
    Rectangle r => r.Width * r.Height,
    _ => throw new InvalidOperationException()
};

var pipeline = Enumerable.Range(1, 10)
    .Where(x => x > 5)
    .Select(x => x * 2)
    .Sum();
```

C# verbose чем F# но достижимо. Discriminated unions (C# 15 preview) уберут `_`-ветку — compiler-enforced exhaustiveness.

### 8.4. Когда F# vs C#

```
F# когда:
✅ DDD-heavy domain
✅ Financial / actuarial calculations
✅ Compilers / parsers
✅ Data transformations
✅ Pure functional preferred

C# когда:
✅ Larger team (popularity)
✅ Mainstream ecosystem
✅ OO-heavy domain
✅ Microsoft stack integration
✅ Tooling (VS for F# weaker)
```

### 8.5. Mixing C# и F#

```fsharp
// F# — Pure logic
namespace Domain

module TaxCalculator =
    let calculateTax amount rate = amount * rate
    let total amount rate = amount + calculateTax amount rate
```

```csharp
// C# — Plumbing / integration
public class OrderService
{
    public async Task ProcessAsync(Order order)
    {
        var total = TaxCalculator.total(order.Subtotal, 0.08m);   // F# function
        order.SetTotal(total);
        await _db.SaveChangesAsync();
    }
}
```

Same CLR — seamless interop. F# для domain logic, C# для I/O / framework integration.

> [!question]- Интервью: какие F# features missing в C#?
> 1) **Discriminated unions native** — `type Shape = Circle | Square | Rectangle`. C# workaround через `abstract record + sealed`; native unions — preview в C# 15 (.NET 11). 2) **Hindley-Milner type inference** — F# almost no annotations, C# limited (`var`, target-typed). 3) **Pipe operator `|>`** — F# `data |> Filter |> Map |> Sum`. C# method chaining alternative. 4) **Computation expressions** — F# generic monadic syntax (any monad). C# only specific (async, LINQ). 5) **Records mature longer** (F# 1.0 vs C# 9+). 6) **Forward composition `>>`** — F# `f >> g`. C# verbose. 7) **Active patterns** — extensible pattern matching. **C# advantages**: bigger ecosystem, more popular, easier hiring, better tooling (VS), records C# enough modern. Mix: F# для domain logic, C# для plumbing.

---

## 9. Best practices

### 9.1. Pure functions

- ✅ **Separate pure logic from I/O**.
- ✅ **Records для Value Objects**.
- ✅ **Functional core, imperative shell** pattern.
- ✅ **Test pure functions без mocks**.
- ❌ **Mutate parameters**.
- ❌ **Hidden global state**.

### 9.2. Immutability

- ✅ **`record` для DTOs / Value Objects**.
- ✅ **`init` properties** — immutable after construction.
- ✅ **`ImmutableList<T>` / `FrozenDictionary<TK, TV>`** для shared collections.
- ✅ **Functional pipelines** через LINQ.
- ❌ **Mutate в hot path** for perf.
- ❌ **Deep mutation** через nested objects.

### 9.3. Higher-order functions

- ✅ **`Func<T>` / `Action<T>`** для simple strategies.
- ✅ **LINQ chains** для transformations.
- ✅ **Composition** через method chaining.
- ❌ **Currying** — verbose в C#.
- ❌ **Heavy partial application** — explicit functions cleaner.

### 9.4. Error handling

- ✅ **`Result<T, E>`** для expected errors (validation, business).
- ✅ **Exception** для unexpected (I/O, bugs).
- ✅ **`Option<T>`** для maybe-values (alternative null).
- ❌ **Mix exception/Result** в same API без cause.

### 9.5. FP libraries

- ✅ **LanguageExt** — full FP library (Option, Either, Try).
- ✅ **MoreLINQ** — additional LINQ operators.
- ❌ **Force FP everywhere** — pragmatic mix.

### 9.6. Не делай

- ❌ Mutation hidden inside pure-looking functions.
- ❌ FP techniques без team buy-in.
- ❌ Over-abstraction (every function generic + composable).
- ❌ Replace OOP entirely — pragmatic mix.

---

## 10. Decision tree

```
Какую FP technique?
│
├── Side effects
│   ├── Pure logic — extract to static method
│   ├── I/O at boundaries — controllers, repos
│   └── "Functional core, imperative shell"
│
├── Data
│   ├── DTO / Value Object → record
│   ├── Mutable entity → class с private setters + methods
│   ├── Read-only collection → IReadOnlyList<T>
│   ├── Immutable collection → ImmutableList<T>
│   └── Read-optimized (.NET 8+) → FrozenDictionary
│
├── Functions
│   ├── Strategy single method → Func<T>
│   ├── Multiple operations → interface
│   ├── Composition → method chaining (или Pipe extension)
│   ├── Specialized → partial application или explicit method
│   └── HOF — accept/return Func<T>
│
├── Maybe-value
│   ├── Simple → T? (NRT)
│   ├── Composable chains → Option<T> (LanguageExt)
│   └── Pragmatic — NRT в most cases
│
├── Error handling
│   ├── Expected business errors → Result<T, E>
│   ├── Unexpected (I/O, bugs) → exception
│   ├── Validation collecting errors → Validation<T>
│   ├── Composable chains → Railway-oriented (Bind chains)
│   └── Library: LanguageExt Either<L, R>
│
└── Pipeline
    ├── LINQ для transformations
    ├── Method chaining instead of |>
    └── ASP.NET Core middleware (Chain of Responsibility)
```

---

## 11. Cheat sheet

```csharp
// === Pure functions ===
public static int Add(int a, int b) => a + b;   // pure

// === Records — Value Objects ===
public record Money(decimal Amount, string Currency);
var m2 = m1 with { Amount = 100m };

// === Immutable collections ===
ImmutableList<int> list = ImmutableList.Create(1, 2, 3);
var updated = list.Add(4);   // new list

FrozenDictionary<string, int> frozen = source.ToFrozenDictionary();

// === Higher-order functions ===
Func<int, int> square = x => x * x;
Func<int, int> times3 = x => x * 3;
Func<int, int> combined = x => times3(square(x));   // composition

// === LINQ pipeline ===
var result = numbers
    .Where(x => x > 0)
    .Select(x => x * 2)
    .OrderBy(x => x)
    .Sum();

// === Option<T> (LanguageExt) ===
using LanguageExt;
using static LanguageExt.Prelude;

Option<User> user = FindUser(42);
var name = user.Map(u => u.Name).IfNone("Unknown");
var address = user.Bind(u => Optional(u.Address));

// === Result<T, E> ===
public abstract record Result<T, E>;
public sealed record Success<T, E>(T Value) : Result<T, E>;
public sealed record Failure<T, E>(E Error) : Result<T, E>;

var result = ValidateRequest(req)
    .Bind(GetCustomer)
    .Bind(CheckInventory)
    .Bind(ChargePayment);

// === Pattern matching ===
var area = shape switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Square s => s.Side * s.Side,
    Rectangle r => r.Width * r.Height,
    _ => throw new InvalidOperationException()
};

// === Composition ===
public static T2 Pipe<T1, T2>(this T1 source, Func<T1, T2> fn) => fn(source);

var result = "  hello  "
    .Pipe(s => s.Trim())
    .Pipe(s => s.ToUpper())
    .Pipe(s => s.Length);
```

---

## 12. Common pitfalls

### 12.1. Hidden mutation

```csharp
public List<int> Process(List<int> input)
{
    input.Sort();   // ❌ mutates argument
    return input;
}

// ✅ Don't mutate
public List<int> Process(List<int> input) => input.OrderBy(x => x).ToList();
```

### 12.2. Heavy currying в C#

```csharp
// ❌ Verbose
Func<int, Func<int, Func<int, int>>> add =
    a => b => c => a + b + c;
add(1)(2)(3);   // 6
```

**Фикс:** explicit method или partial application via extension.

### 12.3. Exception в "pure" function

```csharp
public int Divide(int a, int b)
{
    if (b == 0) throw new DivideByZeroException();
    return a / b;   // ❌ not really pure — может throw
}
```

**Фикс:** `Result<int, string>`.

### 12.4. `Option<T>` over null для everything

```csharp
// ❌ Overkill
public Option<int> GetCount() => Some(_items.Count);

// Just return int — Count never null
```

**Фикс:** `Option<T>` только когда semantically may be missing.

### 12.5. Forgot Bind vs Map

```csharp
// Map — wraps in container automatically
Option<int> Add(int x) => Some(x + 1);

option.Map(Add);   // ❌ Option<Option<int>>!
option.Bind(Add);  // ✅ Option<int>
```

**Фикс:** Map — `T → U`, Bind — `T → M<U>`.

### 12.6. Imperative mindset в FP code

```csharp
// ❌ FP form, imperative mindset
var sum = 0;
list.ForEach(x => sum += x);   // mutation in lambda
```

**Фикс:** `list.Sum()` или `list.Aggregate(0, (a, b) => a + b)`.

### 12.7. Heavy abstraction premature

```csharp
// ❌ Functor / Applicative / Monad для simple business logic
public abstract class TaxCalculator<TStrategy, TInput, TResult> ...
```

**Фикс:** start simple, abstract when needed.

### 12.8. Async + FP не интегрируются elegantly

```csharp
// ❌ Awkward
async Task<Option<User>> FindAsync(int id)
{
    var user = await _db.FindAsync(id);
    return Optional(user);
}

// Bind через Task<Option<T>> verbose
var result = await FindAsync(42);
var address = result.Map(u => u.Address);   // sync map after async
```

**Фикс:** TaskOptionExtensions library или ValueTask helpers.

### 12.9. Records с large data

```csharp
public record HugeReport(string Title, List<Row> Rows, Dictionary<string, byte[]> Attachments);

// Mutation expensive
var updated = report with { Rows = report.Rows.Append(newRow).ToList() };
// Allocation full new List
```

**Фикс:** mutable for large data, immutable boundaries.

### 12.10. FP без team understanding

```csharp
// Team junior — sees Bind/Map and gets confused
result.Bind(x => Validate(x).Map(v => v.Total).Bind(t => Save(t)));
```

**Фикс:** explicit if-else cleaner. Adopt FP gradually.

> [!question]- Интервью: топ-3 ошибки FP в C#?
> 1) **Hidden mutation** — pure-looking function `Process(input)` mutates argument (`input.Sort()`). Use `OrderBy().ToList()`. 2) **Map vs Bind confusion** — `option.Map(fn)` где `fn returns Option<U>` produces `Option<Option<U>>`. Use `Bind` для chain operations returning monads. 3) **Forced FP everywhere** — junior team confused by Bind/Map chains. Adopt gradually: pure functions first, immutability second, monads only когда composing many maybe-failing operations. Бонус: heavy currying — verbose в C#, explicit methods cleaner. Best practice: pragmatic FP techniques (pure + immutable + LINQ), не replace OOP.

---

## 13. Practice exercises

### 13.1. Pure function refactoring

```csharp
// Before — mixed pure + I/O
public class OrderService
{
    private readonly DbContext _db;
    private readonly IEmailService _email;
    
    public OrderService(DbContext db, IEmailService email)
    {
        _db = db;
        _email = email;
    }
    
    public async Task ProcessOrderAsync(int orderId)
    {
        var order = await _db.Orders.FindAsync(orderId);
        var taxRate = order.Country == "US" ? 0.08m : 0.20m;
        var tax = order.Subtotal * taxRate;
        order.Total = order.Subtotal + tax;
        order.Status = "Processed";
        await _db.SaveChangesAsync();
        await _email.SendAsync(order.Customer.Email, "Order processed");
    }
}

// After — pure logic extracted
public static class OrderCalculator
{
    public static decimal GetTaxRate(string country) => country switch
    {
        "US" => 0.08m,
        "EU" => 0.20m,
        _ => 0.10m
    };
    
    public static decimal CalculateTotal(decimal subtotal, decimal taxRate) =>
        subtotal + (subtotal * taxRate);
    
    public static Order Process(Order order, decimal taxRate)
    {
        var total = CalculateTotal(order.Subtotal, taxRate);
        return order with { Total = total, Status = "Processed" };
    }
}

public class OrderService
{
    public async Task ProcessOrderAsync(int orderId)
    {
        var order = await _db.Orders.FindAsync(orderId);
        var taxRate = OrderCalculator.GetTaxRate(order.Country);   // pure
        var processed = OrderCalculator.Process(order, taxRate);    // pure
        _db.Update(processed);
        await _db.SaveChangesAsync();   // I/O at edge
        await _email.SendAsync(order.Customer.Email, "Order processed");
    }
}

// Tests for OrderCalculator — no mocks, fast
[Test]
public void US_tax_rate_is_8_percent() =>
    Assert.That(OrderCalculator.GetTaxRate("US"), Is.EqualTo(0.08m));
```

### 13.2. `Result<T, E>` для validation chain

```csharp
public abstract record Result<T, E>
{
    public sealed record Success(T Value) : Result<T, E>;
    public sealed record Failure(E Error) : Result<T, E>;
    
    public Result<U, E> Map<U>(Func<T, U> mapper) => this switch
    {
        Success s => new Result<U, E>.Success(mapper(s.Value)),
        Failure f => new Result<U, E>.Failure(f.Error),
        _ => throw new InvalidOperationException()
    };
    
    public Result<U, E> Bind<U>(Func<T, Result<U, E>> binder) => this switch
    {
        Success s => binder(s.Value),
        Failure f => new Result<U, E>.Failure(f.Error),
        _ => throw new InvalidOperationException()
    };
}

public record CreateUserCommand(string Name, int Age, string Email);
public record User(string Name, int Age, string Email);

public static Result<CreateUserCommand, string> ValidateName(CreateUserCommand cmd) =>
    string.IsNullOrWhiteSpace(cmd.Name)
        ? new Result<CreateUserCommand, string>.Failure("Name required")
        : new Result<CreateUserCommand, string>.Success(cmd);

public static Result<CreateUserCommand, string> ValidateAge(CreateUserCommand cmd) =>
    cmd.Age >= 18
        ? new Result<CreateUserCommand, string>.Success(cmd)
        : new Result<CreateUserCommand, string>.Failure("Age must be >= 18");

public static Result<CreateUserCommand, string> ValidateEmail(CreateUserCommand cmd) =>
    cmd.Email.Contains("@")
        ? new Result<CreateUserCommand, string>.Success(cmd)
        : new Result<CreateUserCommand, string>.Failure("Invalid email");

// Pipeline (railway-oriented)
public static Result<User, string> CreateUser(CreateUserCommand cmd) =>
    ValidateName(cmd)
        .Bind(ValidateAge)
        .Bind(ValidateEmail)
        .Map(c => new User(c.Name, c.Age, c.Email));

// Use
var result = CreateUser(new CreateUserCommand("Alice", 25, "a@x.com"));
var msg = result switch
{
    Result<User, string>.Success s => $"Created {s.Value.Name}",
    Result<User, string>.Failure f => $"Error: {f.Error}",
    _ => "Unknown"
};
```

### 13.3. Functional pipeline для data processing

```csharp
public record SalesRecord(DateTime Date, string Region, decimal Amount, string Product);

public class SalesAnalyzer
{
    // Pure functions
    public static IEnumerable<SalesRecord> FilterByRegion(IEnumerable<SalesRecord> data, string region) =>
        data.Where(r => r.Region == region);
    
    public static IEnumerable<SalesRecord> FilterByDateRange(IEnumerable<SalesRecord> data, DateTime from, DateTime to) =>
        data.Where(r => r.Date >= from && r.Date <= to);
    
    public static decimal CalculateTotal(IEnumerable<SalesRecord> data) =>
        data.Sum(r => r.Amount);
    
    public static IEnumerable<IGrouping<string, SalesRecord>> GroupByProduct(IEnumerable<SalesRecord> data) =>
        data.GroupBy(r => r.Product);
    
    public static IDictionary<string, decimal> SumByProduct(IEnumerable<SalesRecord> data) =>
        GroupByProduct(data)
            .ToDictionary(g => g.Key, g => g.Sum(r => r.Amount));
    
    // Composed pipeline
    public static IDictionary<string, decimal> AnalyzeRegionalSales(
        IEnumerable<SalesRecord> data,
        string region,
        DateTime from,
        DateTime to)
    {
        return data
            .Pipe(d => FilterByRegion(d, region))
            .Pipe(d => FilterByDateRange(d, from, to))
            .Pipe(SumByProduct);
    }
}

public static class PipelineExtensions
{
    public static T2 Pipe<T1, T2>(this T1 source, Func<T1, T2> fn) => fn(source);
}

// Use
var data = LoadSalesData();
var report = SalesAnalyzer.AnalyzeRegionalSales(
    data,
    region: "EU",
    from: new DateTime(2024, 1, 1),
    to: new DateTime(2024, 12, 31));
```

---

## 14. Что читать дальше

1. **[[modern-features|Modern Features]]** — records, pattern matching.
2. **[[csharp-language-design|Language Design]]** — почему features added.
3. **Scott Wlaschin — F# for Fun and Profit** — fsharpforfunandprofit.com.
4. **Vladimir Khorikov — Functional C#** — Pluralsight courses.
5. **Книга — "Functional Programming in C#" by Enrico Buonanno**.
6. **LanguageExt** — github.com/louthy/language-ext.

---

## 15. См. также

- [[modern-features|Modern Features]] — records, pattern matching
- [[csharp-language-design|Language Design]] — F# influence на C#
- [[csharp-vs-other-langs|C# vs F#]]
- [[delegates-events|Delegates]] — Func/Action
- [[error-handling|Error Handling]] — Result vs Exception
- [[generics-deep|Generics]] — type parameters
- LanguageExt — github.com/louthy/language-ext

---

## 16. Reading list

- **Scott Wlaschin — F# for Fun and Profit** — fsharpforfunandprofit.com
- **Vladimir Khorikov — Functional C#** — enterprisecraftsmanship.com
- **Enrico Buonanno — "Functional Programming in C#"** — book
- **Microsoft Docs — LINQ** — learn.microsoft.com
- **Mark Seemann — Functional Architecture** — blog.ploeh.dk
- **Tomáš Petříček — F# Deep Dives** — github.com/tpetricek
- **Don Syme — F# language designer** — fsharp.org
- **LanguageExt docs** — github.com/louthy/language-ext
- **Railway-Oriented Programming talks** — fsharpforfunandprofit.com/rop
- **Functional Reactive Programming (Rx)** — github.com/dotnet/reactive
