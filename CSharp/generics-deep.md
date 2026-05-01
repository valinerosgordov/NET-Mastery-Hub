---
tags: [csharp, generics, variance, generic-math, constraints, type-system]
level: Middle to Senior
date: 2026-04-30
---

# Generics Deep — дженерики досконально

> **Дженерики на глубоком уровне**: variance (co/contra), generic math (.NET 7+ static abstract members), constraints, generic specialization vs erasure, C# generics vs Java/C++. Closes пробел в "почему generics в C# работают так, а в Java/C++ иначе".

---

## Что это, зачем и когда

### Что такое generics

**Параметризация типа** — пишешь код один раз, работает с разными типами.

```csharp
// Без generics — code duplication
public class IntList
{
    public void Add(int item) { }
    public int Get(int index) { return 0; }
}

public class StringList
{
    public void Add(string item) { }
    public string Get(int index) { return ""; }
}

// С generics — один класс
public class List<T>
{
    public void Add(T item) { }
    public T Get(int index) { return default!; }
}
```

### Зачем

| Без generics | С generics |
|--------------|------------|
| Code duplication | Один class для всех типов |
| Boxing для value types в `object` collections | Zero boxing |
| Cast everywhere — runtime errors | Type safety compile-time |
| `object` based — slow | Type-specialized — fast |

### Brief history

- **C# 1.0 (2002)** — generics НЕ было! `ArrayList`, `Hashtable` — `object` based, slow + cast everywhere
- **C# 2.0 (2005)** — generics добавлены. Революция: `List<T>`, `Dictionary<K,V>`
- **C# 4.0 (2010)** — variance (in/out modifiers)
- **C# 11 (2022)** — **static abstract members в interfaces** → generic math!
- **C# 12+ (2023+)** — primary constructors с generics, alias generic types

---

## 1. Базовые generics

### Generic class

```csharp
public class Box<T>
{
    public T Value { get; set; } = default!;
    
    public void Set(T value) => Value = value;
    public T Get() => Value;
}

// Использование
var intBox = new Box<int> { Value = 42 };
var stringBox = new Box<string> { Value = "hello" };
```

### Generic method

```csharp
public static T First<T>(IEnumerable<T> items)
{
    foreach (var item in items)
        return item;
    throw new InvalidOperationException();
}

// Type inference — компилятор сам определяет T
int firstInt = First(new[] { 1, 2, 3 });        // T = int
string firstStr = First(new[] { "a", "b" });    // T = string
```

### Generic interface

```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(int id);
    Task<IEnumerable<T>> GetAllAsync();
    Task SaveAsync(T entity);
}

public class UserRepository : IRepository<User> { /* ... */ }
public class OrderRepository : IRepository<Order> { /* ... */ }
```

### Multiple type parameters

```csharp
public class Dictionary<TKey, TValue>
    where TKey : notnull
{
    public TValue this[TKey key] { get; set; }
}

public class Result<TSuccess, TError>
{
    public TSuccess? Success { get; }
    public TError? Error { get; }
}
```

---

## 2. Constraints — `where T : ...`

Ограничения на type parameter — что T умеет.

### Базовые constraints

```csharp
// where T : class — reference type
public class Repository<T> where T : class { }

// where T : struct — value type (non-nullable)
public class Vector<T> where T : struct { }

// where T : new() — параметрless constructor
public T Create<T>() where T : new() => new T();

// where T : SomeBaseClass — наследник класса
public class AnimalShelter<T> where T : Animal { }

// where T : ISomeInterface — implements интерфейс
public T Process<T>(T item) where T : IComparable<T> { }

// where T : SomeOtherType — substitutable for другой generic
public void Copy<T, U>(T source, U dest) where T : U { }

// where T : notnull — не nullable (C# 8+)
public class Cache<T> where T : notnull { }

// where T : unmanaged — без managed references (C# 7.3+)
public unsafe void Process<T>(T value) where T : unmanaged { }
```

### Combining constraints

```csharp
public class Repository<T>
    where T : class, IEntity, new()
{
    public T Create() => new T();
}
```

Порядок важен: `class`/`struct` → constraint type → `new()`.

### Constraints на multiple parameters

```csharp
public TResult Map<T, TResult>(T input, Func<T, TResult> mapper)
    where T : class
    where TResult : class
{
    return mapper(input);
}
```

### Default constraints — что значит "не указано"

```csharp
public class Box<T> { }  // T может быть anything (включая nullable)
```

### `where T : default` (C# 9+)

```csharp
public abstract class Base<T>
{
    public abstract void Method(T value);
}

public class Derived<T> : Base<T>
{
    public override void Method(T value)
    {
        if (value is default) { }  // Error в general case
    }
}

// Решается через
public class Derived<T> : Base<T> where T : default { /* ... */ }
```

### Practical examples

```csharp
// IComparable constraint — для сравнения
public T MaxOf<T>(T a, T b) where T : IComparable<T> =>
    a.CompareTo(b) > 0 ? a : b;

// Где T : IDisposable — для resource management
public void UseDisposable<T>(T item) where T : IDisposable
{
    using (item)
    {
        // work
    }
}

// Generic factory с new()
public class Factory<T> where T : new()
{
    public T Create() => new();
}

// Numeric с struct + IComparable
public T Sum<T>(IEnumerable<T> items)
    where T : struct, IComparable<T>
{
    // Limited — нельзя `+` без INumber<T> (.NET 7+)
}
```

---

## 3. Variance — covariance и contravariance

Variance — **как generic types relate друг к другу** при наследовании.

### Проблема

```csharp
// String : object — но что про IEnumerable<string> и IEnumerable<object>?
IEnumerable<string> strings = new List<string> { "a", "b" };
IEnumerable<object> objects = strings;  // ?
```

В C# 4.0+ — **OK** благодаря covariance.

### Covariance — `out`

`out T` — **только output** (return values). Тогда `IEnumerable<Derived>` подставляется где ожидается `IEnumerable<Base>`.

```csharp
public interface IEnumerable<out T>  // out — covariant
{
    IEnumerator<T> GetEnumerator();
}

// Это OK
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings;  // ✅ string : object → IEnumerable<string> : IEnumerable<object>

// Concept: если из контейнера можно только читать — safe convert к base
foreach (object o in objects)
{
    // Получаем object — но реально string. OK.
}
```

### Contravariance — `in`

`in T` — **только input** (parameters). `Action<Base>` подставляется где ожидается `Action<Derived>`.

```csharp
public interface IComparer<in T>  // in — contravariant
{
    int Compare(T x, T y);
}

// IComparer<object> может работать с string'ами
IComparer<object> objComparer = ...;
IComparer<string> strComparer = objComparer;  // ✅

List<string> list = new();
list.Sort(strComparer);  // OK — comparer принимает string (как object)
```

### Invariance (default)

`T` без модификаторов — **invariant**. Никакой substitution.

```csharp
List<T>  // T invariant
List<string> ≠ List<object>  // нельзя подставить друг к другу

List<string> strs = new();
List<object> objs = strs;  // ❌ Compile error!

// Почему? List<T> позволяет ADD — если бы был List<object> = List<string>,
// можно было бы добавить int в List<string>!
```

### Variance rules

| Modifier | Position | Substitution |
|----------|----------|--------------|
| **`out T`** (covariant) | Только return / output | `Derived → Base` |
| **`in T`** (contravariant) | Только parameter / input | `Base → Derived` |
| **`T`** (invariant) | Both | Нет substitution |

### Common variant types в .NET

```csharp
// Covariant (out T)
IEnumerable<out T>
IEnumerator<out T>
IReadOnlyCollection<out T>
IReadOnlyList<out T>
IQueryable<out T>
Func<out TResult>
Func<in T, out TResult>  // оба!

// Contravariant (in T)
IComparer<in T>
IComparable<in T>
IEqualityComparer<in T>
Action<in T>
Predicate<in T>

// Invariant (T)
List<T>
Dictionary<TKey, TValue>
Func<in T1, in T2, ...>  // input args contravariant
```

### Func<T1, T2, T3, TResult> — mixed

```csharp
public delegate TResult Func<in T1, in T2, out TResult>(T1 arg1, T2 arg2);

Func<object, string> a = (object o) => o.ToString();
Func<string, string> b = a;  // ✅ contravariant T1 (object → string OK для input)

Func<int, string> c = (int i) => i.ToString();
Func<int, object> d = c;  // ✅ covariant TResult (string → object OK для output)
```

### Когда нужно знать variance

✅ **Practical scenarios:**
- Sort с `IComparer<T>` для базового типа — contravariance
- LINQ `.Cast<T>()` — covariance под капотом
- Event handlers с `Action<TEventArgs>` — contravariance
- Создание custom generic interfaces — решаешь in/out

✅ **Senior signal:** Объяснить почему `List<string>` нельзя присвоить к `List<object>`, но `IEnumerable<string>` — можно.

---

## 4. Generic Math (.NET 7+) — революция

### Проблема до .NET 7

```csharp
// Можно ли написать generic Sum?
public static T Sum<T>(IEnumerable<T> items)
    where T : ???
{
    T total = ???;
    foreach (var item in items)
        total += item;  // ❌ Compile error — нет операторов для generic T
    return total;
}
```

До .NET 7 — **невозможно**. Решали через дублирование (Sum для int, double, decimal отдельно).

### Static abstract members в interfaces (C# 11 / .NET 7)

Можно объявить **static** методы / операторы в interface:

```csharp
public interface INumber<TSelf>
    where TSelf : INumber<TSelf>
{
    static abstract TSelf operator +(TSelf left, TSelf right);
    static abstract TSelf operator -(TSelf left, TSelf right);
    static abstract TSelf Zero { get; }
    static abstract TSelf One { get; }
    // ...
}
```

### Решение

```csharp
public static T Sum<T>(IEnumerable<T> items) where T : INumber<T>
{
    T total = T.Zero;
    foreach (var item in items)
        total += item;
    return total;
}

// Работает с любым numeric type!
Sum(new[] { 1, 2, 3 });               // 6 (int)
Sum(new[] { 1.5, 2.5 });               // 4.0 (double)
Sum(new[] { 1.5m, 2.5m });             // 4.0 (decimal)
Sum(new[] { 1f, 2f });                  // 3.0f (float)
```

### `INumber<T>` hierarchy

```
IComparable<TSelf>
    └─ INumberBase<TSelf>
        ├─ IBinaryNumber<TSelf>
        │   ├─ IBinaryInteger<TSelf>  ← byte, short, int, long, uint, ulong
        │   └─ IFloatingPointIeee754<TSelf>  ← float, double, Half
        └─ IFloatingPoint<TSelf>  ← decimal
```

### Полезные операции через generics

```csharp
public static T Average<T>(IEnumerable<T> items)
    where T : INumber<T>
{
    T sum = T.Zero;
    int count = 0;
    foreach (var item in items)
    {
        sum += item;
        count++;
    }
    return sum / T.CreateChecked(count);
}

public static T Max<T>(IEnumerable<T> items)
    where T : IComparable<T>
{
    T max = items.First();
    foreach (var item in items.Skip(1))
        if (item.CompareTo(max) > 0) max = item;
    return max;
}

// Vector math
public static T DotProduct<T>(T[] a, T[] b) where T : INumber<T>
{
    T sum = T.Zero;
    for (int i = 0; i < a.Length; i++)
        sum += a[i] * b[i];
    return sum;
}
```

### `T.CreateChecked` / `CreateSaturating` / `CreateTruncating`

Конверсия чисел через generic:

```csharp
T value = T.CreateChecked(42);    // throws при overflow
T value = T.CreateSaturating(42); // clamp к min/max
T value = T.CreateTruncating(42); // обрезает bits
```

### Operators в generic interfaces

```csharp
public interface IShape<TSelf> where TSelf : IShape<TSelf>
{
    static abstract TSelf operator +(TSelf left, TSelf right);
    static abstract TSelf Identity { get; }
    abstract double Area { get; }
}

public class Rectangle : IShape<Rectangle>
{
    public double Width { get; set; }
    public double Height { get; set; }
    public double Area => Width * Height;
    
    public static Rectangle Identity => new() { Width = 0, Height = 0 };
    public static Rectangle operator +(Rectangle a, Rectangle b) =>
        new() { Width = a.Width + b.Width, Height = a.Height + b.Height };
}
```

---

## 5. C# generics vs Java vs C++

### Java — type erasure

```java
// Java
List<String> list = new ArrayList<>();
// Runtime — это просто List, type info стёрта (erasure)

class Container<T> {
    void method() {
        // T x = new T();  // ❌ Cannot — нет runtime type info
        // if (obj instanceof T) { }  // ❌ Cannot
    }
}
```

**Pros:**
- Backward compat (бывший `List` не ломается)
- Простой компилятор

**Cons:**
- `List<int>` невозможен — только `List<Integer>` (boxing!)
- Нельзя `new T()`, `T.class`
- Нельзя array of T — `T[]`

### C++ — templates

```cpp
template<typename T>
class Container {
    T value;
};

// Compile time — generates separate class per T
Container<int> intCont;     // class _Z9Container_int
Container<string> strCont;  // class _Z9Container_string
```

**Pros:**
- Maximum performance (specialized code)
- Можно ANY operation на T (compile checks)
- "Duck typing" — если T имеет нужный метод, OK

**Cons:**
- Code bloat (separate copy per T)
- Cryptic compile errors
- Slower compilation

### C# — лучшее обоих миров

C# generics — **runtime aware**:

```csharp
public class Container<T>
{
    public Type GetTypeOfT() => typeof(T);  // ✅ runtime type info
    public T CreateInstance() => Activator.CreateInstance<T>();  // ✅ if T : new()
}

var intCont = new Container<int>();
intCont.GetTypeOfT();  // System.Int32

// Reflection видит generics
var listType = typeof(List<>);
var intListType = listType.MakeGenericType(typeof(int));
```

### Generic specialization для value types

В CLR — **разное поведение** для reference и value types:

```csharp
List<string>  // reference type T
List<int>     // value type T
```

| | Reference type T | Value type T |
|--|------------------|--------------|
| **CIL код** | Один shared (через type erasure-like sharing) | Specialized per T |
| **Boxing** | Нет (already references) | **Нет** (avoid boxing!) |
| **Performance** | Standard | Native (как C++ template) |
| **Memory** | Shared метакод | Separate JIT'd code per T |

```csharp
// List<int> — int хранится прямо в array, no boxing
var list = new List<int>();
list.Add(42);  // 42 stored as int directly

// До generics — ArrayList — boxing!
var old = new ArrayList();
old.Add(42);  // 42 boxed в object → heap allocation
```

### Resume: C# vs Java vs C++

```
Java:   Runtime erasure → simple, but slow для primitives
C++:    Compile-time templates → fast, but code bloat
C#:     Hybrid — reified generics для value types,
         shared CIL для reference types → best of both
```

---

## 6. Variance в classes — нет

```csharp
// ❌ Variance применима только к interfaces и delegates
public class List<out T> { }  // Compile error!

// ✅ Только interfaces / delegates
public interface IEnumerable<out T> { }
public delegate TResult Func<out TResult>();
```

**Workaround для classes:** explicit cast методы или wrapper interfaces.

---

## 7. Generic type inference

### Compiler infers type из arguments

```csharp
public T First<T>(IEnumerable<T> items) => items.First();

// Type inference
First(new[] { 1, 2, 3 });        // T = int
First(new[] { "a", "b" });        // T = string
First<double>(new[] { 1, 2, 3 }); // explicit T = double (int → double conversion)
```

### Когда inference fails

```csharp
public T Get<T>() => default!;

Get();              // ❌ Cannot infer T from empty arg list
Get<int>();         // ✅ explicit
```

### Method group inference

```csharp
public TResult Map<T, TResult>(T input, Func<T, TResult> mapper) => mapper(input);

string s = Map(42, x => x.ToString());  // T=int, TResult=string inferred
```

### `default!` vs constraint

```csharp
public T GetDefault<T>() => default!;  // works for any T

public T GetDefault<T>() where T : new() => new();  // works only with new() constraint

public T GetDefault<T>() where T : class => null!;  // returns null for ref types
```

---

## 8. Generic patterns

### Repository\<T\>

```csharp
public interface IRepository<T> where T : class, IEntity
{
    Task<T?> GetByIdAsync(Guid id);
    Task<IEnumerable<T>> GetAllAsync();
    Task SaveAsync(T entity);
    Task DeleteAsync(Guid id);
}

public class EFRepository<T>(DbContext db) : IRepository<T> where T : class, IEntity
{
    public Task<T?> GetByIdAsync(Guid id) =>
        db.Set<T>().FirstOrDefaultAsync(e => e.Id == id);
    
    public async Task<IEnumerable<T>> GetAllAsync() =>
        await db.Set<T>().ToListAsync();
    
    public Task SaveAsync(T entity) { db.Set<T>().Add(entity); return db.SaveChangesAsync(); }
    
    public async Task DeleteAsync(Guid id)
    {
        var entity = await GetByIdAsync(id);
        if (entity != null) db.Set<T>().Remove(entity);
        await db.SaveChangesAsync();
    }
}
```

### Builder\<T\>

```csharp
public class QueryBuilder<T> where T : class
{
    private readonly List<Expression<Func<T, bool>>> _filters = new();
    
    public QueryBuilder<T> Where(Expression<Func<T, bool>> filter)
    {
        _filters.Add(filter);
        return this;
    }
    
    public IQueryable<T> Build(IQueryable<T> source) =>
        _filters.Aggregate(source, (q, f) => q.Where(f));
}

// Использование
var query = new QueryBuilder<User>()
    .Where(u => u.IsActive)
    .Where(u => u.Age > 18)
    .Build(db.Users);
```

### Result\<TSuccess, TError\>

```csharp
public abstract record Result<TSuccess, TError>;

public sealed record Success<TSuccess, TError>(TSuccess Value) : Result<TSuccess, TError>;
public sealed record Failure<TSuccess, TError>(TError Error) : Result<TSuccess, TError>;

// Использование
public Result<User, string> GetUser(int id)
{
    var user = _repo.Find(id);
    return user is not null
        ? new Success<User, string>(user)
        : new Failure<User, string>("Not found");
}

// Pattern matching
var result = GetUser(1);
var message = result switch
{
    Success<User, string> { Value: var user } => $"Found: {user.Name}",
    Failure<User, string> { Error: var err } => $"Error: {err}",
    _ => throw new UnreachableException()
};
```

См. [[error-handling|Error Handling]] и [[functional-csharp|Functional C#]].

### Generic factory

```csharp
public interface IFactory<T> where T : new()
{
    T Create();
}

public class DefaultFactory<T> : IFactory<T> where T : new()
{
    public T Create() => new();
}

public class FactoryWithConfig<T> : IFactory<T> where T : new()
{
    private readonly Action<T> _configure;
    
    public FactoryWithConfig(Action<T> configure) => _configure = configure;
    
    public T Create()
    {
        var instance = new T();
        _configure(instance);
        return instance;
    }
}
```

### Generic event handler

```csharp
public class EventBus
{
    private readonly Dictionary<Type, List<object>> _handlers = new();
    
    public void Subscribe<TEvent>(Action<TEvent> handler)
    {
        if (!_handlers.TryGetValue(typeof(TEvent), out var list))
            _handlers[typeof(TEvent)] = list = new();
        list.Add(handler);
    }
    
    public void Publish<TEvent>(TEvent evt)
    {
        if (_handlers.TryGetValue(typeof(TEvent), out var list))
            foreach (Action<TEvent> handler in list)
                handler(evt);
    }
}
```

---

## 9. Common Pitfalls

### 1. `default(T)` для reference types

```csharp
public T GetOrDefault<T>(IEnumerable<T> items)
{
    foreach (var item in items)
        return item;
    return default!;  // null для class, 0 для int
}

// Caller проблема:
string s = GetOrDefault(new string[0]);  // s = null!
int x = GetOrDefault(new int[0]);         // x = 0 — может быть valid value!

// ✅ Лечение: nullable return
public T? GetOrDefault<T>(IEnumerable<T> items) where T : class { /* ... */ }

// Или Result<T, _>
```

### 2. Type comparisons

```csharp
public bool IsType<T>(object obj) => obj is T;

IsType<int>(42);             // true
IsType<int?>(42);            // true (boxing involved)
IsType<int>(null);           // false
IsType<int?>(null);          // true (Nullable<int>)
IsType<string>(42);          // false
IsType<object>(42);          // true
```

### 3. Generic constraints не наследуются

```csharp
public class Base<T> where T : new() { }

public class Derived<T> : Base<T>
    // ⚠️ Constraint надо повторить!
    where T : new()
{ }
```

### 4. Static fields per type closure

```csharp
public class Counter<T>
{
    public static int Count = 0;
}

Counter<int>.Count = 5;
Counter<string>.Count = 10;

Counter<int>.Count;     // 5
Counter<string>.Count;  // 10  ← разные!
```

`Counter<int>` и `Counter<string>` — **разные types**, разные static fields.

### 5. Boxing с constraints

```csharp
// ❌ Constraint object — boxing для value types
public T Method<T>(T item) where T : object  // ⚠️
{
    Console.WriteLine(item.ToString());  // ToString — virtual, no boxing
}

// ✅ Без constraint или more specific
public T Method<T>(T item) { /* ... */ }
```

### 6. Generic constraints с struct

```csharp
public T Get<T>() where T : struct
{
    return default(T);  // OK — value type default
}

// Nullable<int>?
Get<int?>();  // ❌ int? is Nullable<int> — struct, OK ✅ (passed Nullable<int>)
```

### 7. Variance с arrays — broken!

```csharp
// Java-style array covariance — историческая ошибка C#
string[] strs = new string[10];
object[] objs = strs;  // ✅ compiles — но runtime broken!

objs[0] = 42;  // 💥 ArrayTypeMismatchException — runtime error!

// ✅ Используй IReadOnlyList<T> вместо arrays
IReadOnlyList<string> strs2 = new List<string>();
IReadOnlyList<object> objs2 = strs2;  // safe — covariant + read-only
```

### 8. Open generic types

```csharp
// Open generic — не bound к конкретному T
Type listOpen = typeof(List<>);
Type listClosed = typeof(List<int>);

// Reflection
var openType = typeof(IRepository<>);
var closedType = openType.MakeGenericType(typeof(User));
var instance = Activator.CreateInstance(closedType);
```

См. [[reflection-expression-trees|Reflection и Expression Trees]].

### 9. Multiple parameters and constraint conflicts

```csharp
public void Method<T, U>(T t, U u)
    where T : U  // T должен быть substituable for U
{
    U value = t;  // OK, T : U
}

// Использование
Method<string, object>("hello", "world");  // OK, string : object
Method<int, string>(42, "x");              // ❌ int : string?
```

### 10. Generic methods на virtual

```csharp
public class Base
{
    public virtual T Method<T>() => default!;
}

public class Derived : Base
{
    // ❌ Cannot specialize за T type
    public override int Method<int>() => 42;  // Compile error
    
    // ✅ Можно overrideшь, но T остаётся generic
    public override T Method<T>() => default!;
}
```

---

## 10. Performance considerations

### JIT specialization для value types

```csharp
List<int> intList = new();   // JIT generates specialized code
List<string> strList = new(); // JIT shares code между всеми ref types

// Benchmark:
intList.Add(42);     // ~5ns — specialized native code, no boxing
oldArrayList.Add(42); // ~50ns — boxing + cast
```

### Constraint-based optimization

```csharp
// Без constraint — virtual dispatch
public bool Equals<T>(T a, T b) => a.Equals(b);  // calls object.Equals (virtual)

// С constraint — прямой call
public bool Equals<T>(T a, T b) where T : IEquatable<T> => a.Equals(b);  // direct call
```

### `EqualityComparer<T>.Default`

```csharp
// Slow
public bool Compare<T>(T a, T b) => a.Equals(b);  // boxing для value types!

// Fast
public bool Compare<T>(T a, T b) => EqualityComparer<T>.Default.Equals(a, b);
// Optimized — uses IEquatable<T> if implemented
```

### Generic delegates avoid allocation

```csharp
// Func<T, TResult> shared between instances — no allocation
Func<int, int> doubler = x => x * 2;
```

### Generic specialization vs interface dispatch

```csharp
// Generic constraint — direct call (fast)
public T Add<T>(T a, T b) where T : INumber<T> => a + b;

// Interface variable — virtual call (slower)
public INumber<T> Add(INumber<T> a, INumber<T> b) where T : INumber<T> => 
    /* via virtual dispatch */;
```

---

## 11. Reflection с generics

### Open vs Closed types

```csharp
Type openList = typeof(List<>);          // open
Type closedList = typeof(List<int>);      // closed

openList.IsGenericTypeDefinition;         // true
closedList.IsGenericType;                 // true
closedList.GetGenericArguments();          // [System.Int32]
```

### Make closed type at runtime

```csharp
Type elementType = typeof(string);
Type closedListType = typeof(List<>).MakeGenericType(elementType);

object list = Activator.CreateInstance(closedListType);
// list — object reference, но реально List<string>

MethodInfo addMethod = closedListType.GetMethod("Add");
addMethod.Invoke(list, new object[] { "hello" });
```

### Generic method invocation

```csharp
public class MyClass
{
    public T GetValue<T>() => default!;
}

var instance = new MyClass();
MethodInfo method = typeof(MyClass).GetMethod("GetValue");
MethodInfo closedMethod = method.MakeGenericMethod(typeof(int));
int value = (int)closedMethod.Invoke(instance, null);
```

### Performance — cache reflection!

```csharp
// ❌ Reflection каждый call — slow
public T Get<T>() {
    var method = typeof(MyClass).GetMethod(...);  // expensive!
    var closed = method.MakeGenericMethod(typeof(T));
    return (T)closed.Invoke(...);
}

// ✅ Cache + Compiled Expression
private static readonly ConcurrentDictionary<Type, Func<object, object>> _cache = new();

public T Get<T>(object instance)
{
    var fn = _cache.GetOrAdd(typeof(T), CompileGetter);
    return (T)fn(instance);
}
```

См. [[reflection-expression-trees|Reflection и Expression Trees]].

---

## 12. Best Practices

### Constraints

- **Используй максимально specific constraint** — компилятор оптимизирует
- **`IEquatable<T>`** для equality — избегает boxing
- **`IComparable<T>`** для comparison — избегает boxing
- **`INumber<T>`** (.NET 7+) для math операций
- **`notnull`** где не должен быть null
- **`unmanaged`** для interop с native code

### Variance

- **`out T`** для return-only types (IEnumerable, IReadOnlyList)
- **`in T`** для input-only (IComparer, Action)
- **Default invariant** если оба
- **Не используй variance в classes** — нельзя

### Generic Math

- **`INumber<T>`** для general numeric operations
- **`IBinaryInteger<T>`** для bit operations
- **`IFloatingPoint<T>`** для float / double / decimal
- **`T.CreateChecked` / `Saturating` / `Truncating`** для conversions

### Performance

- **Generic specialization** для value types — automatic, just write generic
- **`EqualityComparer<T>.Default`** для equality — avoid boxing
- **Cache reflection** для generic operations
- **Avoid runtime `MakeGenericType`** в hot path

### Design

- **Repository\<T\>** для CRUD
- **Result\<T, E\>** для railway-oriented
- **Builder\<T\>** для fluent APIs
- **Factory\<T\>** для creation
- **Не over-genericize** — иногда конкретный тип читабельнее

---

## 13. Cheat sheet

| Задача | Pattern |
|--------|---------|
| Generic Sum | `T : INumber<T>` (.NET 7+) |
| Type-safe collection | `List<T>` или custom `Container<T>` |
| Repository | `IRepository<T> where T : class, IEntity` |
| Compare any T | `T : IComparable<T>` или `EqualityComparer<T>.Default` |
| Numeric conversion | `T.CreateChecked(value)` |
| Avoid boxing equality | `EqualityComparer<T>.Default.Equals(a, b)` |
| Read-only collection covariant | `IReadOnlyList<out T>` |
| Comparer contravariant | `IComparer<in T>` |
| Open generic | `typeof(List<>)` |
| Closed generic at runtime | `openType.MakeGenericType(typeof(int))` |
| Default value | `default(T)` или `default!` |
| New T() | `where T : new()` then `new T()` |
| Variance | `out T` для return, `in T` для input |
| Static abstract | C# 11+ in interfaces |

---

## 14. Decision tree

```
Что делаешь?
│
├── Math operations с любым numeric type?
│   → INumber<T> (.NET 7+)
│
├── Collection / container?
│   → List<T> / Dictionary<K, V> / etc.
│
├── Type-safe API возможны разные types?
│   → Generic interface / class
│
├── Comparison / equality?
│   → IEquatable<T> / IComparable<T>
│
├── Variance нужна?
│   → out / in modifiers (только interfaces / delegates)
│
├── Reflection с generics?
│   → typeof(T) / MakeGenericType / cache!
│
└── Generic constraint pattern?
    → where T : class, IEntity, new()
```

---

## Case Studies

### Case Study #1 — Type-safe Result\<T, E\>

**Проблема:** Methods могут fail. Throw exceptions для validation — не идиоматично.

**✅ Generic Result:**
```csharp
public readonly record struct Result<T, E>
{
    public T? Value { get; init; }
    public E? Error { get; init; }
    public bool IsSuccess { get; init; }
    
    public static Result<T, E> Ok(T value) => new() { Value = value, IsSuccess = true };
    public static Result<T, E> Fail(E error) => new() { Error = error };
}

public Result<User, string> CreateUser(string email)
{
    if (!email.Contains("@")) return Result<User, string>.Fail("Invalid email");
    var user = new User { Email = email };
    return Result<User, string>.Ok(user);
}

// Use
var result = CreateUser("test@example.com");
if (result.IsSuccess)
    Console.WriteLine($"User: {result.Value!.Email}");
else
    Console.WriteLine($"Error: {result.Error}");
```

---

### Case Study #2 — Variance — IEnumerable\<T\> covariance

**Проблема:** Хочется передать `List<Dog>` где expected `IEnumerable<Animal>`.

**✅ Covariance — out:**
```csharp
// IEnumerable<T> объявлен с out → covariant
public interface IEnumerable<out T> { /* ... */ }

List<Dog> dogs = new() { new Dog(), new Dog() };
IEnumerable<Animal> animals = dogs;  // OK — covariance!

foreach (Animal a in animals) Console.WriteLine(a.Name);
```

**vs Contravariance — in:**
```csharp
// IComparer<T> с in → contravariant
public interface IComparer<in T> { /* ... */ }

IComparer<Animal> animalComparer = new AnimalComparer();
IComparer<Dog> dogComparer = animalComparer;  // OK — contravariance!
```

---

### Case Study #3 — Generic math (.NET 7+)

**Проблема:** До .NET 7 — нельзя generic math operations.

**✅ INumber\<T\>:**
```csharp
public T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var v in values) sum += v;
    return sum;
}

// Works for any numeric type
int s1 = Sum(new[] { 1, 2, 3 });           // 6
double s2 = Sum(new[] { 1.5, 2.5, 3.5 });  // 7.5
decimal s3 = Sum(new[] { 1m, 2m, 3m });    // 6m
```

См. [[numeric-types-math|Numeric Types & Math]].


---

## См. также

- [[modern-features|Modern C# Features]] — generic features evolution
- [[csharp-language-design|C# Language Design]] — generics history
- [[reflection-expression-trees|Reflection и Expression Trees]] — runtime generics
- [[functional-csharp|Functional C#]] — `Result<T, E>`, `Option<T>`
- [[error-handling|Error Handling]] — Result pattern
- [[csharp-vs-other-langs|C# vs Other Langs]] — generics comparison
- [[../Runtime/compilation-jit|Compilation и JIT]] — JIT specialization

## Reading list

- **C# in Depth** — Jon Skeet (главы про generics)
- **Microsoft Docs — Generics** — learn.microsoft.com/dotnet/csharp/programming-guide/generics
- **Microsoft Docs — Generic Math** — learn.microsoft.com/dotnet/standard/generics/math
- **Anders Hejlsberg на generics** — youtube talks
- **Stephen Toub — Generic specialization** — devblogs.microsoft.com
- **Jon Skeet — Variance article** — codeblog.jonskeet.uk
- **CLR via C#** — Jeffrey Richter (CLR generics internals)
- **Eric Lippert — Variance series** — ericlippert.com
