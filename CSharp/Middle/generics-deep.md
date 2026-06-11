---
tags: [csharp, generics, middle, type-parameters, constraints, variance, covariance]
level: Middle
date: 2026-05-07
---

# Generics deep — параметризованные типы и методы

> **Type parameters, constraints (`where T : ...`), variance (`in`/`out`), generic methods, generic math (.NET 7+).** Закрывает пробел: «знаю про `List<T>`, не понимаю когда `where T : class` vs `notnull`, и почему `IEnumerable<out T>` covariant».

---

## 0. Как читать

Если впервые — раздел 1→4. Constraints deep — раздел 5. Variance — раздел 6. Production guidance — раздел 11 (best practices), 13 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Что такое generics

Generics — type parameters, позволяющие писать **type-safe** reusable code:

```csharp
// Без generics
public class IntList { public void Add(int x); public int Get(int i); }
public class StringList { public void Add(string x); public string Get(int i); }
// Дублирование!

// С generics
public class List<T>
{
    public void Add(T item) { /* ... */ }
    public T Get(int i) { /* ... */ }
}

var ints = new List<int>();
var strs = new List<string>();
```

### 1.2. Зачем

1. **Type safety** — compile-time check, нет casting.
2. **No boxing** для value types — performance.
3. **Reuse** — один код работает с разными типами.
4. **Self-documenting** — `Dictionary<int, User>` clearer чем `Hashtable`.

### 1.3. Главное правило

```
Generic class / method когда:
  - Один code работает с many types
  - Type safety важна
  - Performance — no boxing для value types

Constraints (where T : ...) когда:
  - Нужны specific operations (Equals, comparison)
  - Только reference types или value types
  - Только specific interface

Variance (in / out) когда:
  - Generic interface / delegate
  - IEnumerable<out T>, Func<in T, out R>
```

### 1.4. Эволюция

| Версия | Что |
|--------|-----|
| **.NET 2.0** | Generics introduced (`List<T>`, `Dictionary<TKey,TValue>`) |
| **C# 4.0** | Variance (`in`/`out`) |
| **C# 7.3** | `Enum`, `Delegate`, `unmanaged` constraints |
| **C# 8** | NRT — `T?` для unconstrained, `where T : notnull` |
| **C# 9** | Static abstract members в interfaces (preview) |
| **C# 11** | Static abstract members stable, generic math |
| **.NET 7+** | Generic Math interfaces (`INumber<T>`) |

> [!info]- Если ты знаешь Java / Kotlin / TypeScript / Rust
> **Java:** generics через type erasure — runtime типы стёрты, есть только `Object`. C# — **runtime types preserved** (reified). Performance better, no boxing для value types.
>
> **Kotlin:** Очень similar к C#. Variance через `in`/`out` modifiers ровно как в C#.
>
> **TypeScript:** type erasure (compile-time only). Variance inferred. Похоже syntax, но нет runtime types.
>
> **Rust:** generics через monomorphization (как C# для value types). Trait bounds ↔ constraints. Сильнее compile-time guarantees.

> [!question]- Интервью: как generics в C# отличаются от Java?
> **Java erasure** — runtime типы стёрты (`List<String>` становится `List<Object>` после compile). Performance — boxing для primitives, no compile-time speed gain. **C# reification** — generic types preserved в runtime. JIT generates **specialized code** для each value type T (separate machine code), reuses code для reference types. Result: 1) `typeof(T)` works в runtime. 2) No boxing для `List<int>`. 3) Performance gains от specialization. Cost: bigger binary footprint при много специализаций.

---

## 2. Generic classes

### 2.1. Один type parameter

```csharp
public class Container<T>
{
    private T _value;
    
    public Container(T value) => _value = value;
    
    public T Get() => _value;
    public void Set(T value) => _value = value;
}

var c = new Container<int>(42);
var s = new Container<string>("hello");
```

### 2.2. Multiple type parameters

```csharp
public class Pair<TKey, TValue>
{
    public TKey Key { get; }
    public TValue Value { get; }
    
    public Pair(TKey key, TValue value)
    {
        Key = key;
        Value = value;
    }
}

var p = new Pair<int, string>(1, "one");
```

### 2.3. Type inference в constructor (C# 9+ + variants)

```csharp
// Ранее — explicit
var p = new Pair<int, string>(1, "one");

// .NET / C# inference в некоторых scenarios
// (через factory methods чаще)
public static class Pair
{
    public static Pair<TK, TV> Create<TK, TV>(TK key, TV value) => new(key, value);
}

var p = Pair.Create(1, "one");   // inferred
```

### 2.4. Default value

```csharp
public class Container<T>
{
    private T _value = default!;   // 0 для int, null для string
    
    public T Default() => default!;
    public T Default2() => default(T)!;
}
```

`default(T)` — type-specific default. С NRT — `default!` чтобы suppress warning.

### 2.5. Static members per type

```csharp
public class Cache<T>
{
    public static List<T> Items { get; } = new();
}

Cache<int>.Items.Add(1);
Cache<string>.Items.Add("hello");
// Items в Cache<int> и Cache<string> — РАЗНЫЕ list (per-type static)
```

Каждая generic specialization — separate static state.

### 2.6. Inheritance

```csharp
public class BaseRepository<T> { }
public class UserRepository : BaseRepository<User> { }   // closed
public class GenericRepo<T> : BaseRepository<T> { }       // open

public class GenericRepo<T> where T : Entity { }   // с constraint
```

### 2.7. Generic interface

```csharp
public interface IRepository<T>
{
    T? GetById(int id);
    void Save(T item);
}

public class UserRepository : IRepository<User>
{
    public User? GetById(int id) { /* ... */ }
    public void Save(User item) { /* ... */ }
}
```

> [!question]- Интервью: генерик класс с static field — каждая specialization имеет свой?
> **Да**. `Cache<int>.Items` и `Cache<string>.Items` — **отдельные** lists. JIT специализирует код для each generic type, static fields включены в specialization. Это feature, не bug — позволяет per-type cache, per-type configuration. Pitfall: если writing thread-safe singleton через `static`, помни что `Singleton<T>.Instance` создаст per-T singleton, не global. Для truly global — non-generic class.

---

## 3. Generic methods

### 3.1. Базовый

```csharp
public static T Max<T>(T a, T b) where T : IComparable<T>
{
    return a.CompareTo(b) > 0 ? a : b;
}

int x = Max(3, 5);                  // T inferred — int
string s = Max("hello", "world");    // T inferred — string
```

### 3.2. Type inference

```csharp
public T First<T>(IEnumerable<T> source) => source.GetEnumerator().Current;

var x = First(new[] { 1, 2, 3 });   // T = int (inferred from arg)

// Если no arg — explicit
var y = default(int);   // 0
```

Compiler infers T из arguments. Если не может — explicit `Method<T>(...)`.

### 3.3. Multiple parameters

```csharp
public TResult Convert<TIn, TResult>(TIn input, Func<TIn, TResult> selector) =>
    selector(input);

int len = Convert("hello", s => s.Length);   // TIn=string, TResult=int
```

### 3.4. Generic method в non-generic class

```csharp
public static class Helpers
{
    public static T Identity<T>(T x) => x;
    
    public static List<T> SingleItemList<T>(T item) => new() { item };
}

var x = Helpers.Identity(42);              // int
var l = Helpers.SingleItemList("hello");   // List<string>
```

### 3.5. Generic method в generic class

```csharp
public class Container<T>
{
    private T _value;
    
    public TOut Convert<TOut>(Func<T, TOut> mapper) => mapper(_value);
}

var c = new Container<int>(42);
string s = c.Convert(x => x.ToString());   // TOut = string
```

### 3.6. Constraints на method

```csharp
public T MaxOrDefault<T>(IEnumerable<T> source, T fallback)
    where T : IComparable<T>
{
    T best = fallback;
    bool any = false;
    foreach (var item in source)
    {
        if (!any || item.CompareTo(best) > 0) { best = item; any = true; }
    }
    return best;
}
```

> [!question]- Интервью: как compiler выводит type parameter?
> **Type inference** — compiler анализирует argument types. Для `Method<T>(T x, T y)`, при вызове `Method(1, 2)` — `T = int`. Если arguments разных типов — выбирает common base (`int + double` → не выводит, ambiguous). Если no arguments или all type parameters в return type — нужен explicit. Pitfall: type inference не работает для **return-only** generics. Тогда explicit: `Method<int>(...)`. С C# 9+ target-typed `new` упрощает: `List<int> l = new();`.

---

## 4. Open vs closed types

### 4.1. Терминология

- **Open type** — `List<T>` где T unbound.
- **Closed type** — `List<int>` где T = конкретный int.
- **Open generic** — `List<>` (без T) — для reflection.

### 4.2. typeof — open vs closed

```csharp
Type closed = typeof(List<int>);       // closed
Type open = typeof(List<>);             // open generic — без T

// Закрытие через MakeGenericType
Type t = open.MakeGenericType(typeof(int));   // List<int>

// Создание instance
var list = Activator.CreateInstance(t);   // List<int>
```

### 4.3. Reflection

```csharp
Type type = typeof(Dictionary<,>);   // open, 2 type params
Type[] args = new[] { typeof(string), typeof(int) };
Type closed = type.MakeGenericType(args);   // Dictionary<string, int>

// Generic method via reflection
MethodInfo method = typeof(Helpers).GetMethod("Identity")!;
MethodInfo generic = method.MakeGenericMethod(typeof(int));
generic.Invoke(null, new object[] { 42 });
```

### 4.4. nameof для generics

```csharp
nameof(List<int>);   // "List" — без type args
typeof(List<int>).Name;   // "List`1" (backtick + arity)
typeof(List<int>).FullName;   // "System.Collections.Generic.List`1[[System.Int32, ...]]"
```

> [!question]- Интервью: чем `typeof(List<>)` отличается от `typeof(List<int>)`?
> **`typeof(List<>)`** — **open generic** type, T unbound. Используется для reflection, Generic Type Definitions. **`typeof(List<int>)`** — **closed** specific type. `MakeGenericType(typeof(int))` закрывает open в closed. Activator.CreateInstance работает с closed types. Reflection metadata: `Type.IsGenericType` для closed, `Type.IsGenericTypeDefinition` для open. Naming: closed `List`1[Int32]`, open `List`1`. Use case open: DI containers (`AddScoped(typeof(IRepository<>), typeof(Repository<>))`).

---

## 5. Constraints — `where T : ...`

### 5.1. Reference type

```csharp
public T Find<T>(int id) where T : class
{
    // T — reference type
    // Может быть null
    return null!;
}
```

### 5.2. Value type

```csharp
public T Compute<T>() where T : struct
{
    // T — value type, не nullable
    return default;
}

// Nullable<T> для value types
public T? FindNullable<T>() where T : struct
{
    return null;   // Nullable<T>
}
```

### 5.3. notnull (C# 8+)

```csharp
public class MyClass<T> where T : notnull
{
    // T — non-null (reference или value)
    // Compile-time NRT enforcement
}
```

### 5.4. unmanaged (C# 7.3+)

```csharp
public unsafe Span<T> Wrap<T>(T* ptr, int length) where T : unmanaged
{
    return new Span<T>(ptr, length);
}
```

`unmanaged` — value type без reference fields (можно использовать с pointers).

### 5.5. new() — parameterless constructor

```csharp
public T CreateNew<T>() where T : new()
{
    return new T();   // ОК
}
```

### 5.6. Specific class

```csharp
public class Repository<T> where T : Entity
{
    public void Save(T item) { item.UpdatedAt = DateTime.UtcNow; }
}
```

T должен быть Entity или derived.

### 5.7. Interface

```csharp
public T Max<T>(T a, T b) where T : IComparable<T>
{
    return a.CompareTo(b) > 0 ? a : b;
}
```

### 5.8. Multiple constraints

```csharp
public T Process<T>(T input)
    where T : class, IDisposable, new()
{
    // T — class, IDisposable, имеет parameterless ctor
}

// Order: class/struct/notnull/unmanaged FIRST, then interfaces, then new() LAST
public class Service<T, U>
    where T : class, IRepository<U>, new()
    where U : Entity, IEquatable<U>
{
}
```

### 5.9. Type parameter constraint

```csharp
public class Cache<TKey, TValue> where TKey : IEquatable<TKey>
{
}

public T Convert<T, U>(U input) where T : U   // T inherits from U
{
}
```

### 5.10. Enum / Delegate (C# 7.3+)

```csharp
public T ParseEnum<T>(string s) where T : Enum
{
    return (T)Enum.Parse(typeof(T), s);
}

public void Wrap<T>(T del) where T : Delegate { }
```

> [!question]- Интервью: какие constraints доступны в C#?
> 1) **`where T : class`** — reference type. 2) **`where T : struct`** — value type (non-null). 3) **`where T : notnull`** (C# 8+) — non-null any kind. 4) **`where T : unmanaged`** (C# 7.3+) — value type без reference fields. 5) **`where T : new()`** — parameterless constructor. 6) **`where T : SomeClass`** — specific base class. 7) **`where T : IInterface`** — implements interface. 8) **`where T : Enum / Delegate`** (C# 7.3+) — enum / delegate. 9) **`where T : U`** — type parameter constraint. Multiple: order class/struct first, then interfaces, then new() last.

---

## 6. Variance — `in` / `out`

### 6.1. Зачем

```csharp
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings;   // ✅ — covariant!

// Без variance — было бы compile error (List<string> не List<object>)
```

`IEnumerable<T>` covariant в T (out T) — `IEnumerable<string>` assignable `IEnumerable<object>`.

### 6.2. out — covariance

```csharp
public interface IProducer<out T>   // T в OUTPUT positions
{
    T Produce();
    // void Consume(T item);   // ❌ — T в input
}

IProducer<string> sp = new Producer<string>();
IProducer<object> op = sp;   // ✅ covariant
```

`out` — T только в **output** positions (return values, get accessors).

### 6.3. in — contravariance

```csharp
public interface IConsumer<in T>   // T в INPUT positions
{
    void Consume(T item);
    // T Produce();   // ❌ — T в output
}

IConsumer<object> co = new Consumer<object>();
IConsumer<string> cs = co;   // ✅ contravariant — string accepted в object handler
```

`in` — T только в **input** positions (parameters).

### 6.4. invariance — default

```csharp
public interface IList<T>   // T и input и output
{
    void Add(T item);    // input
    T Get(int i);         // output
}

// IList<string> assignable IList<object>?
// НЕТ — invariant, нельзя!
```

Если T используется и input и output — invariant (default).

### 6.5. Built-in variance

| Type | Variance |
|------|----------|
| `IEnumerable<out T>` | Covariant |
| `IEnumerator<out T>` | Covariant |
| `IReadOnlyList<out T>` | Covariant |
| `IReadOnlyCollection<out T>` | Covariant |
| `IList<T>` | Invariant |
| `ICollection<T>` | Invariant |
| `Func<in T1, ..., out TResult>` | Contravariant in inputs, covariant in result |
| `Action<in T>` | Contravariant |
| `IComparer<in T>` | Contravariant |
| `IEqualityComparer<in T>` | Contravariant |

### 6.6. Custom variance

```csharp
public interface IMapper<in TIn, out TOut>
{
    TOut Map(TIn input);
}

IMapper<object, string> objMapper = ...;
IMapper<string, object> strMapper = objMapper;
//   contravariant ↑               covariant ↑
```

### 6.7. Variance не для classes

```csharp
public class List<T> { }   // ❌ нельзя List<out T>
```

Variance работает только на **interfaces** и **delegates**.

### 6.8. Array variance — historical

```csharp
string[] strs = new[] { "a", "b" };
object[] objs = strs;   // ✅ covariant array (legacy)
objs[0] = 1;             // ❌ ArrayTypeMismatchException at runtime!
```

Array covariance — мoжет вызвать runtime exception. Anti-pattern. Используй `IReadOnlyList<T>`.

> [!question]- Интервью: разница между covariance и contravariance?
> **Covariance (`out T`)** — `IEnumerable<string>` assignable `IEnumerable<object>`. T в **output** positions only (return, get). Sub-type подходит где требуется super-type. **Contravariance (`in T`)** — `Action<object>` assignable `Action<string>`. T в **input** positions only (parameters). Super-type подходит где требуется sub-type. **Invariance** — default, T в обе positions, no assignment compatibility. Practical: `IEnumerable<out T>` covariant, `Func<in T, out R>` contravariant в input + covariant в result. `IList<T>` invariant потому что Add (input) + Get (output).

---

## 7. Generic delegates

### 7.1. Func / Action / Predicate

```csharp
Func<int, int> square = x => x * x;
Func<int, int, int> add = (a, b) => a + b;
Func<string, int> len = s => s.Length;

Action printX = () => Console.WriteLine("X");
Action<int> printNum = x => Console.WriteLine(x);
Action<string, int> log = (msg, code) => Console.WriteLine($"{code}: {msg}");

Predicate<int> isEven = x => x % 2 == 0;
```

`Func<...>` — last type parameter is return. `Action<...>` — void. `Predicate<T>` — bool result.

### 7.2. Custom generic delegate

```csharp
public delegate TResult Mapper<in TIn, out TResult>(TIn input);

Mapper<int, string> intToStr = x => x.ToString();
```

### 7.3. `EventHandler<T>`

```csharp
public class OrderService
{
    public event EventHandler<OrderCreatedEventArgs>? OrderCreated;
}

public class OrderCreatedEventArgs : EventArgs
{
    public Order Order { get; init; } = null!;
}

service.OrderCreated += (s, e) => Console.WriteLine(e.Order.Id);
```

См. [[delegates-events]].

---

## 8. Generic constraints with operators (.NET 7+)

### 8.1. До .NET 7 — невозможно

```csharp
// ❌ Не работало до .NET 7
public T Sum<T>(IEnumerable<T> source)
{
    T sum = default;
    foreach (var x in source) sum += x;   // Error — нет + для T
    return sum;
}
```

### 8.2. С .NET 7 — generic math

```csharp
public T Sum<T>(IEnumerable<T> source) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var x in source) sum += x;   // ✅
    return sum;
}
```

См. [[numeric-types-math]] раздел 7.

### 8.3. Static abstract members

```csharp
public interface IShape
{
    static abstract IShape Create();
    abstract double Area();
}

public class Circle : IShape
{
    public static IShape Create() => new Circle();
    public double Area() => Math.PI;
}

T MakeShape<T>() where T : IShape => T.Create();
```

C# 11+ позволяет interfaces декларировать static methods, которые derived обязан implement.

---

## 9. Common patterns

### 9.1. Factory pattern

```csharp
public class Factory<T> where T : new()
{
    public T Create() => new T();
}

// С dependency
public class Factory<T> where T : new()
{
    private readonly Action<T> _initializer;
    public Factory(Action<T> initializer) => _initializer = initializer;
    
    public T Create()
    {
        var instance = new T();
        _initializer(instance);
        return instance;
    }
}
```

### 9.2. Repository

```csharp
public interface IRepository<T> where T : Entity
{
    Task<T?> GetByIdAsync(int id);
    Task<List<T>> GetAllAsync();
    Task AddAsync(T item);
    Task UpdateAsync(T item);
    Task DeleteAsync(int id);
}

public class Repository<T> : IRepository<T> where T : Entity
{
    private readonly DbContext _db;
    public Repository(DbContext db) => _db = db;
    
    public async Task<T?> GetByIdAsync(int id) =>
        await _db.Set<T>().FindAsync(id);
    
    // ... другие
}
```

### 9.3. Generic Builder

```csharp
public class QueryBuilder<T> where T : Entity
{
    private IQueryable<T> _query;
    public QueryBuilder(IQueryable<T> source) => _query = source;
    
    public QueryBuilder<T> Where(Expression<Func<T, bool>> predicate)
    {
        _query = _query.Where(predicate);
        return this;
    }
    
    public QueryBuilder<T> OrderBy<TKey>(Expression<Func<T, TKey>> selector)
    {
        _query = _query.OrderBy(selector);
        return this;
    }
    
    public List<T> Build() => _query.ToList();
}
```

### 9.4. Generic Singleton (per-type)

```csharp
public sealed class Singleton<T> where T : new()
{
    private static readonly Lazy<T> _instance = new(() => new T());
    public static T Instance => _instance.Value;
}

var s1 = Singleton<MyService>.Instance;
```

### 9.5. Result type

```csharp
public abstract record Result<T>
{
    public sealed record Success(T Value) : Result<T>;
    public sealed record Failure(string Error) : Result<T>;
}

public Result<int> Divide(int a, int b) =>
    b == 0 ? new Result<int>.Failure("div by zero") : new Result<int>.Success(a / b);
```

> [!question]- Интервью: как реализовать generic singleton?
> ```csharp
> public sealed class Singleton<T> where T : new()
> {
>     private static readonly Lazy<T> _instance = new(() => new T());
>     public static T Instance => _instance.Value;
> }
> ```
> `Lazy<T>` — thread-safe initialization. Constraint `new()` — generic нужен parameterless ctor. Per-T instance: `Singleton<A>.Instance != Singleton<B>.Instance` (separate static field for each generic specialization). Pitfall: если нужен truly global singleton, не делай generic — обычный sealed class. Best practice 2024+: DI container `AddSingleton<TService>` лучше manual singleton (testable, configurable).

---

## 10. Generic constraints in nested types

### 10.1. Nested generic

```csharp
public class Outer<T>
{
    public class Inner   // implicitly knows T
    {
        public T Value { get; set; } = default!;
    }
}

var inner = new Outer<int>.Inner();
```

### 10.2. Independent type parameters

```csharp
public class Outer<T>
{
    public class Inner<U>
    {
        public T Outer { get; set; } = default!;
        public U Inner { get; set; } = default!;
    }
}

var x = new Outer<int>.Inner<string>();
```

---

## 11. Best Practices

### 11.1. Implementing generics

- ✅ **Type parameters meaningful names** — `T` for single, `TKey`/`TValue` для multiple, `TIn`/`TOut` для transformations.
- ✅ **`where T : notnull`** для NRT compatibility.
- ✅ **`out`/`in` variance** для interfaces.
- ✅ **Constraints minimum** — only what needed.
- ❌ **Many type parameters** (> 3) — refactor.
- ❌ **Generic for sake of generic** — concrete types если только один use case.

### 11.2. Performance

- ✅ **Generic specialization** для value types — no boxing.
- ✅ **`EqualityComparer<T>.Default`** в generic code (uses `IEquatable<T>`).
- ✅ **Source generators** для metadata-heavy generic patterns.
- ❌ **Reflection on generic types** в hot path — slow.

### 11.3. API design

- ✅ **Pre-LINQ-style** generics (`IRepository<T>`).
- ✅ **Variance correctly** — `IProducer<out T>` / `IConsumer<in T>`.
- ✅ **Generic interfaces** для DI flexibility.
- ❌ **Generic class где interface достаточен**.

### 11.4. Не делай

- ❌ Open generic типы как parameters в public API.
- ❌ `dynamic` вместо generic (lose type safety + perf).
- ❌ `object` параметры там где generic подойдёт.
- ❌ Reified types for runtime polymorphism — interface лучше.

---

## 12. Decision tree

```
Что нужно?
│
├── Reuse code для many types
│   ├── С compile-time type safety → generic class / method
│   ├── С interface contract → generic interface
│   └── Concrete static helpers → static class + generic method
│
├── Constraints
│   ├── Reference type → where T : class
│   ├── Value type → where T : struct
│   ├── Non-null any kind → where T : notnull (C# 8+)
│   ├── Specific behavior → where T : IInterface
│   ├── Need to construct → where T : new()
│   ├── Numeric → where T : INumber<T> (.NET 7+)
│   └── Combine — order class/struct, interfaces, new() last
│
├── Variance
│   ├── Output only (T в return) → out T (covariant)
│   ├── Input only (T в params) → in T (contravariant)
│   └── Both → invariant (default)
│
└── Generic patterns
    ├── Repository<T> → DI container
    ├── Factory<T> где T : new() → instances
    ├── Singleton<T> per-T → reuse
    ├── Result<T> — wrapping computations
    └── EventHandler<T> — typed events
```

---

## 13. Cheat sheet

```csharp
// === Generic class ===
public class Container<T>
{
    private T _value;
    public Container(T value) => _value = value;
    public T Get() => _value;
}

// === Generic method ===
public static T Max<T>(T a, T b) where T : IComparable<T> =>
    a.CompareTo(b) > 0 ? a : b;

// === Multiple type params ===
public TResult Convert<TIn, TResult>(TIn input, Func<TIn, TResult> map) =>
    map(input);

// === Constraints ===
where T : class                           // reference type
where T : struct                          // value type
where T : notnull                         // non-null (C# 8+)
where T : unmanaged                       // unmanaged value type
where T : new()                           // parameterless ctor
where T : SomeBase                        // specific base
where T : IComparable<T>                  // interface
where T : Enum                            // enum (C# 7.3+)
where T : Delegate                        // delegate
where T : U                               // type param

// Combined (order: class/struct first, interfaces, new() last)
where T : class, IDisposable, new()

// === Variance ===
public interface IProducer<out T>          // covariant (output)
{
    T Produce();
}

public interface IConsumer<in T>           // contravariant (input)
{
    void Consume(T item);
}

// === Open vs closed ===
typeof(List<>)                              // open
typeof(List<int>)                           // closed
type.MakeGenericType(typeof(int))           // open → closed

// === Generic delegate ===
Func<int, int> square = x => x * x;
Action<string> log = msg => Console.WriteLine(msg);
EventHandler<MyEventArgs> handler = (s, e) => { };

// === Generic Math (.NET 7+) ===
public T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var v in values) sum += v;
    return sum;
}
```

---

## 14. Common Pitfalls

### 14.1. T? без constraint — confusing

```csharp
public T? FirstOrDefault<T>(IEnumerable<T> source)
{
    // Если T = string → T? = string?
    // Если T = int    → T? = int? (Nullable<int>)
    // Если T = struct → T? = T? (Nullable<T>)
}
```

**Фикс:** explicit `where T : class` или `where T : struct` constraint, или use `[MaybeNull]` annotations.

### 14.2. new T() с required parameters

```csharp
public T Create<T>() where T : new() => new T();   // ✅ только parameterless

// Если нужны parameters — Factory pattern
public T Create<T>(Func<T> factory) => factory();
```

### 14.3. Casting T → конкретный тип

```csharp
public void Process<T>(T item)
{
    if (item is string s) { /* ... */ }   // OK
    var x = (string)(object)item;           // ❌ runtime cast, может throw
}
```

**Фикс:** type pattern matching (`is`).

### 14.4. Variance в classes

```csharp
public class List<out T> { }   // ❌ Compile error — variance only interfaces/delegates
```

**Фикс:** определи interface с variance, class implements.

### 14.5. Forgot constraint — нельзя оператор

```csharp
public T Add<T>(T a, T b) => a + b;   // ❌ нет + для unconstrained T
```

**Фикс:** `where T : INumber<T>` (.NET 7+) или specific overloads.

### 14.6. Generic specialization performance surprise

```csharp
var list = new List<MyStruct>();
// Each MyStruct в list — copied (struct semantics)
// Generic specialized код для MyStruct — separate machine code
```

Generally good (no boxing), но binary size grows. Для most apps — neglible.

### 14.7. Static field per-T — leak

```csharp
public class Cache<T>
{
    public static List<T> Items { get; } = new();
}

// Cache<int>.Items, Cache<string>.Items, Cache<User>.Items — все живут до AppDomain end
```

**Фикс:** non-generic + Dictionary<Type, ...>.

### 14.8. Reflection MakeGenericType slow

```csharp
foreach (var t in types)
{
    var closed = openType.MakeGenericType(t);   // ❌ slow в loop
    var instance = Activator.CreateInstance(closed);
}
```

**Фикс:** cache в Dictionary<Type, Func<object>>.

### 14.9. Array variance runtime

```csharp
string[] strs = new[] { "a" };
object[] objs = strs;
objs[0] = 1;   // ❌ ArrayTypeMismatchException at runtime!
```

**Фикс:** `IReadOnlyList<T>` (covariant safely).

### 14.10. Constraints не satisfied — не сообщает в use site

```csharp
public class Foo<T> where T : class { }
var f = new Foo<int>();   // CS0452 error при INSTANTIATION
```

OK actually — compile-time error. Pitfall больше runtime когда reflection.

> [!question]- Интервью: топ-3 ошибки с generics?
> 1) **`T?` без constraint** — `T?` означает разное для class vs struct. Без `where T : class/struct` — confusing semantics. Always specify constraint. 2) **Casting `T → конкретный тип`** через `(string)(object)item` — runtime cast, может throw. Use pattern matching `if (item is string s)`. 3) **Variance only on interfaces/delegates** — `class List<out T>` compile error. Variance through interface (`IReadOnlyList<out T>`), implementation invariant.

---

## 15. Practice exercises

### 15.1. Generic Repository

```csharp
public abstract class Entity
{
    public int Id { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}

public interface IRepository<T> where T : Entity
{
    Task<T?> GetByIdAsync(int id);
    Task<IReadOnlyList<T>> GetAllAsync();
    Task AddAsync(T item);
    Task RemoveAsync(int id);
}

public class Repository<T> : IRepository<T> where T : Entity
{
    private readonly DbContext _db;
    public Repository(DbContext db) => _db = db;
    
    public Task<T?> GetByIdAsync(int id) => _db.Set<T>().FindAsync(id).AsTask();
    public async Task<IReadOnlyList<T>> GetAllAsync() => await _db.Set<T>().ToListAsync();
    public async Task AddAsync(T item)
    {
        _db.Set<T>().Add(item);
        await _db.SaveChangesAsync();
    }
    public async Task RemoveAsync(int id)
    {
        var item = await GetByIdAsync(id);
        if (item != null) { _db.Set<T>().Remove(item); await _db.SaveChangesAsync(); }
    }
}

public class User : Entity { public string Email { get; set; } = ""; }
// services.AddScoped(typeof(IRepository<>), typeof(Repository<>));
```

### 15.2. `Result<T>` wrapping computations

```csharp
public abstract record Result<T>
{
    public sealed record Success(T Value) : Result<T>;
    public sealed record Failure(string Error) : Result<T>;
    
    public Result<U> Map<U>(Func<T, U> mapper) => this switch
    {
        Success s => new Result<U>.Success(mapper(s.Value)),
        Failure f => new Result<U>.Failure(f.Error),
        _ => throw new InvalidOperationException()
    };
    
    public Result<U> Bind<U>(Func<T, Result<U>> binder) => this switch
    {
        Success s => binder(s.Value),
        Failure f => new Result<U>.Failure(f.Error),
        _ => throw new InvalidOperationException()
    };
}

Result<int> Divide(int a, int b) =>
    b == 0 ? new Result<int>.Failure("Division by zero") : new Result<int>.Success(a / b);

var result = Divide(10, 2)
    .Map(x => x * 2)
    .Bind(x => x > 0 ? new Result<int>.Success(x) : new Result<int>.Failure("non-positive"));
```

### 15.3. Type-safe event publisher

```csharp
public interface IEventHandler<in TEvent>
{
    Task HandleAsync(TEvent ev);
}

public class EventBus
{
    private readonly Dictionary<Type, List<object>> _handlers = new();
    
    public void Subscribe<TEvent>(IEventHandler<TEvent> handler)
    {
        var type = typeof(TEvent);
        if (!_handlers.TryGetValue(type, out var list))
            _handlers[type] = list = [];
        list.Add(handler);
    }
    
    public async Task PublishAsync<TEvent>(TEvent ev)
    {
        var type = typeof(TEvent);
        if (_handlers.TryGetValue(type, out var list))
        {
            foreach (var h in list.OfType<IEventHandler<TEvent>>())
                await h.HandleAsync(ev);
        }
    }
}

public record OrderCreated(int OrderId);

public class EmailHandler : IEventHandler<OrderCreated>
{
    public Task HandleAsync(OrderCreated ev) { /* send email */ return Task.CompletedTask; }
}
```

---

## 16. Что читать дальше

1. **[[oop|OOP]]** — generic classes + inheritance.
2. **[[delegates-events|Delegates]]** — Func/Action/EventHandler.
3. **[[numeric-types-math|Numerics]]** — generic math (`INumber<T>`).
4. **Expression trees** — generic expressions.
5. **System.Linq** — generic LINQ implementation.

---

## 17. См. также

- [[oop|OOP]]
- [[collections-linq|Collections и LINQ]] — generic collections
- [[delegates-events|Delegates]]
- [[numeric-types-math|Numerics]] — generic math
- [[equality-comparison|Equality]] — `IEquatable<T>`
- Expression trees deep
- DI container generic registration

---

## 18. Reading list

- **Microsoft Docs — Generics** — learn.microsoft.com/dotnet/csharp/programming-guide/generics/
- **Microsoft Docs — Constraints** — learn.microsoft.com/dotnet/csharp/programming-guide/generics/constraints-on-type-parameters
- **Microsoft Docs — Variance** — learn.microsoft.com/dotnet/csharp/programming-guide/concepts/covariance-contravariance/
- **Microsoft Docs — Generic Math** — learn.microsoft.com/dotnet/standard/generics/math
- **Eric Lippert — Variance series** — ericlippert.com
- **Jon Skeet — C# in Depth** — generics chapter (lengthy)
- **Bill Wagner — Effective C#** — generic items
