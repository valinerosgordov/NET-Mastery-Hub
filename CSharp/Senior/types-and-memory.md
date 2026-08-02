---
tags: [csharp, types, memory, senior, gc, stack, heap, value-types, reference-types, generations]
level: Senior
date: 2026-05-10
---

# Types и Memory — value vs reference, stack/heap, GC

> **Value types vs reference types, stack vs heap, boxing/unboxing, struct semantics, ref struct, escape analysis, GC generations, LOH, POH, ref returns.** Когда struct vs class, как избежать allocations, какие GC modes работают для твоего workload. Закрывает пробел: «знаю что есть GC, не понимаю когда struct vs class и почему `ref struct` не может cross await».

---

## 0. Как читать

Если впервые — раздел 1→3 (mental model + value/reference). GC — раздел 6→8. Performance hot path — раздел 9. Production guidance — раздел 11 (best practices), 13 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Категории типов в C#

```
Value types (struct):
- bool, byte, short, int, long, float, double, decimal
- enum
- struct (user-defined)
- record struct (C# 10+)
- ref struct (Span<T>, ReadOnlySpan<T>)
- Tuples (ValueTuple<T1, T2, ...>)

Reference types (class):
- class (user-defined)
- record (class — C# 9+)
- string (immutable reference type)
- arrays (any T[])
- delegate types (Func<T>, Action<T>, ...)
- interface

Pointer types (unsafe):
- T*
- void*
```

### 1.2. Главное различие — semantics

```
Value type:
- Copied by value when assigned / passed to method
- Two variables — independent copies
- Stored on stack (often) or inline в parent type
- Eligible для stack allocation (escape analysis)

Reference type:
- Copied by reference when assigned
- Two variables — share same object
- Object on heap, variable on stack (the reference itself)
- Always heap-allocated (mostly)
```

```csharp
// Value type — copy
struct Point { public int X, Y; }
var p1 = new Point { X = 1, Y = 2 };
var p2 = p1;   // copy
p2.X = 100;
Console.WriteLine(p1.X);   // 1 — unchanged

// Reference type — share
class PointClass { public int X, Y; }
var c1 = new PointClass { X = 1, Y = 2 };
var c2 = c1;   // both reference same object
c2.X = 100;
Console.WriteLine(c1.X);   // 100!
```

### 1.3. Главное правило

```
Choose struct когда:
  - Logically a single value (Point, Money, DateTime)
  - Small (< 16 bytes typical, < 32 max)
  - Immutable
  - Не replicated в lots of collections (boxing risk)
  - Performance critical (avoid heap allocation)

Choose class когда:
  - Has identity (User, Order)
  - Mutable
  - Large (> 32 bytes)
  - Inheritance needed
  - Nullable semantically meaningful
```

### 1.4. Эволюция

| Версия | Что |
|--------|-----|
| **C# 1.0** | struct, class, basic value/reference dichotomy |
| **C# 2.0** | Generics, `Nullable<T>` (boxing reduction) |
| **C# 7.0** | `ref` returns, `ref struct` |
| **C# 7.2** | `readonly struct`, `Span<T>`, `ref readonly` |
| **C# 8.0** | Default interface methods, `using` declarations |
| **C# 9.0** | Records (class + struct), init properties |
| **C# 10.0** | `record struct`, `with` expression на structs |
| **C# 11.0** | `required` members, list patterns, generic attributes |
| **.NET 5+** | Pinned Object Heap, GC improvements |
| **.NET 8+** | `FrozenDictionary` (read-optimized) |

> [!info]- Если ты знаешь Java / Kotlin / Rust / C++ / Go
> **Java:** primitives (int/long/double) — value-like, but **no struct**. Everything else heap. Project Valhalla long-term added value types.
>
> **Kotlin:** `data class` always heap. `inline class` (value class) — single-field, struct-like.
>
> **Rust:** struct stack-allocated by default, ownership prevents many GC problems. `Vec<T>`/`Box<T>` — heap when needed.
>
> **C++:** explicit stack vs heap (`Point p` vs `new Point`). C# similar but more controlled.
>
> **Go:** value vs pointer types similar. Escape analysis по compile time. C# JIT does similar.

> [!question]- Интервью: чем struct отличается от class в C#?
> 1) **Memory**: struct stored inline (stack или внутри parent type), class — heap allocation. 2) **Semantics**: struct copied by value (each variable — copy), class shared by reference (same instance). 3) **Inheritance**: struct cannot inherit (only interfaces), class supports inheritance. 4) **Default**: struct cannot be null (без `Nullable<T>`), class can. 5) **Equality**: struct uses field-by-field equality (default), class uses reference equality (default). 6) **Performance**: struct avoids heap allocation (faster для small data), class better для large/mutable/inherited. **Best practice**: struct для < 32 bytes immutable values (Point, Money, DateTime). Class для everything else. **C# 9+ records**: `record class` (default) immutable reference, `record struct` immutable value (C# 10+).

---

## 2. Value types — struct deep

### 2.1. Custom struct

```csharp
public struct Point
{
    public int X { get; set; }
    public int Y { get; set; }
    
    public double DistanceTo(Point other)
    {
        var dx = other.X - X;
        var dy = other.Y - Y;
        return Math.Sqrt(dx * dx + dy * dy);
    }
}

// Use
var p = new Point { X = 1, Y = 2 };
Console.WriteLine(p.DistanceTo(new Point { X = 4, Y = 6 }));   // 5.0
```

### 2.2. readonly struct (C# 7.2+)

```csharp
public readonly struct ImmutablePoint
{
    public int X { get; }
    public int Y { get; }
    
    public ImmutablePoint(int x, int y) => (X, Y) = (x, y);
    
    public ImmutablePoint Translate(int dx, int dy) => new(X + dx, Y + dy);
}
```

`readonly struct` — все fields readonly, compiler optimizes (no defensive copies). **Best practice 2024+**: always `readonly` for value types.

### 2.3. record struct (C# 10+)

```csharp
public record struct Point(int X, int Y);

var p1 = new Point(1, 2);
var p2 = p1 with { X = 100 };   // value copy + modification

p1 == p2;   // false — value equality
```

`record struct` — value type с auto-generated `Equals` (field-by-field), `GetHashCode`, `ToString`, `with`. Best для small immutable Value Objects.

```csharp
// Or readonly record struct
public readonly record struct Money(decimal Amount, string Currency);
```

### 2.4. Value vs reference struct copy cost

```
Small struct (< 16 bytes):
- Copy ~ 1-2ns
- Stack-friendly
- Cache-friendly

Large struct (> 32 bytes):
- Copy expensive
- Performance worse than class (frequent copying)
- Recommend pass by ref / in
```

### 2.5. Default values

```csharp
struct Point { public int X, Y; }

var p = default(Point);   // X=0, Y=0
var p2 = new Point();      // X=0, Y=0 — same as default

class PointClass { public int X, Y; }
PointClass c = default;   // null!
```

Value types — never null by default (always zeroed). Reference types — null by default.

### 2.6. Boxing

```csharp
int x = 42;        // value type — stack
object o = x;      // ❌ BOXING — heap allocation, copy x to heap

int y = (int)o;    // unboxing — copy back from heap

// Common boxing scenarios
Console.WriteLine(x);                     // ❌ object overload — boxes
List<object> list = new() { x };           // ❌ boxing
Dictionary<int, object> d = ...;           // ❌ value → object value boxes
```

Boxing — **expensive** (~30ns + GC pressure). Avoid в hot paths.

### 2.7. Avoiding boxing

```csharp
// ❌ Boxing
List<object> list = new() { 1, 2, 3 };
Console.WriteLine($"Result: {x}");   // boxing of x

// ✅ No boxing
List<int> list = new() { 1, 2, 3 };
Console.WriteLine($"Result: {x}");   // no boxing — string interpolation generic

// Use generic constraints
void Process<T>(T item) where T : struct
{
    // T not boxed
}

// Use specific overloads
List<int>.Add(int) — no boxing
List<object>.Add(object) — boxes int

// IEquatable<T> для structs
struct Point : IEquatable<Point>
{
    public bool Equals(Point other) => X == other.X && Y == other.Y;
    // Generic Equals — no boxing
}
```

> [!question]- Интервью: что такое boxing?
> Конверсия **value type → object** (или interface). Allocates heap memory, copies value into heap object. **Cost**: ~30ns + GC pressure. **Examples**: 1) `object o = 42` — int boxed. 2) `List<object>.Add(intValue)` — boxes. 3) `string.Format("{0}", intValue)` — boxes. 4) `IComparable<int>.CompareTo` — interface dispatch на struct boxes если без generic constraint. **Avoiding**: 1) Generic methods с `where T : struct`. 2) Specific overloads (`Console.WriteLine(int)` exists separate of `Console.WriteLine(object)`). 3) `IEquatable<T>` interface для struct. 4) `Span<T>` для slicing без boxing. 5) `Nullable<T>` (HasValue check) для optional values. **Modern**: most BCL APIs avoid boxing через generics. С# generics reified — no boxing для primitives (vs Java erasure).

### 2.8. Struct → interface variable BOXES (и каждый вызов идёт по копии)

Почему: интерфейс — reference type. Присвоить struct в переменную типа интерфейса значит дать ссылку, а ссылаться можно только на heap-объект — поэтому JIT кладёт **копию** struct в box на куче. Дальше каждый interface-dispatched вызов выполняется **на этой boxed-копии**, а не на исходном значении. Это и аллокация, и тихий баг с мутабельным struct: меняешь box, оригинал не двигается.

```csharp
public interface IShape
{
    double Area();
}

public struct Circle(double radius) : IShape
{
    public double Area() => Math.PI * radius * radius;
}
```

```csharp
var circle = new Circle(2.0);

// ❌ struct → interface variable: BOXES once, копия Circle уходит на кучу
IShape shape = circle;
double a = shape.Area();   // virtual dispatch на boxed-копии

// ✅ переменная конкретного типа: вызов прямой, без box
Circle direct = circle;
double b = direct.Area();   // no box, no heap
```

Хуже всего — приём `IShape` в сигнатуру: каждый аргумент-struct боксится на входе:

```csharp
// ❌ boxes каждый переданный struct на границе вызова
static double SumAreas(IReadOnlyList<IShape> shapes)
{
    double total = 0;
    foreach (var s in shapes) total += s.Area();
    return total;
}

// ✅ generic constraint → JIT монорфизирует под Circle, value-type dispatch, no box
static double SumAreas<T>(IReadOnlyList<T> shapes) where T : struct, IShape
{
    double total = 0;
    for (int i = 0; i < shapes.Count; i++) total += shapes[i].Area();
    return total;
}
```

`where T : struct, IShape` — ключ: для value-типа JIT генерирует отдельную специализацию метода (monomorphization), вызывает `Area()` напрямую на значении, без box и без virtual dispatch.

> [!info]- Как доказать: MemoryDiagnoser delta + IL в ILSpy

> ```csharp
> [MemoryDiagnoser]
> public class ShapeDispatchBench
> {
>     private readonly Circle[] _circles =
>         Enumerable.Range(1, 1000).Select(i => new Circle(i)).ToArray();
>
>     [Benchmark(Baseline = true)]
>     public double ViaInterface()        // boxes 1000 раз
>     {
>         double total = 0;
>         foreach (Circle c in _circles)
>         {
>             IShape s = c;               // box на каждой итерации
>             total += s.Area();
>         }
>         return total;
>     }
>
>     [Benchmark]
>     public double ViaGeneric()          // 0 B
>     {
>         double total = 0;
>         foreach (Circle c in _circles) total += AreaOf(c);
>         return total;
>     }
>
>     private static double AreaOf<T>(T shape) where T : struct, IShape => shape.Area();
> }
> ```
>
> Ожидаемый delta: `ViaInterface` показывает `Allocated ≈ 1000 × sizeof(box)` (≈ 24 KB на x64 для бокса с double-полем), `ViaGeneric` — `0 B`. В ILSpy у `ViaInterface` на месте `IShape s = c;` видна инструкция `box Circle`; у `ViaGeneric` она отсутствует — вместо неё прямой `call`/`callvirt` без бокса на специализированной версии метода.

---

## 3. Reference types — class deep

### 3.1. Object header

```
Reference type object в memory:
- Object header (~16 bytes на 64-bit):
  - Sync block index (lock state)
  - Method table pointer (type info)
- Fields (alignment-padded)

Total overhead: 16-24 bytes per object
For small classes — overhead may exceed actual data!
```

### 3.2. Reference equality vs value equality

```csharp
class User { public int Id { get; set; } }
var u1 = new User { Id = 1 };
var u2 = new User { Id = 1 };

u1 == u2;          // false — different references!
u1.Equals(u2);     // false — default class Equals is reference equality
ReferenceEquals(u1, u2);   // false

// Override для value equality
public class User : IEquatable<User>
{
    public int Id { get; init; }
    public bool Equals(User? other) => other != null && Id == other.Id;
    public override bool Equals(object? obj) => Equals(obj as User);
    public override int GetHashCode() => Id.GetHashCode();
    public static bool operator ==(User? a, User? b) => a is null ? b is null : a.Equals(b);
    public static bool operator !=(User? a, User? b) => !(a == b);
}

// Or just use record!
public record User(int Id);   // auto value equality
```

### 3.3. record class (C# 9+)

```csharp
public record User(int Id, string Name);

var u1 = new User(1, "Alice");
var u2 = new User(1, "Alice");
u1 == u2;   // true — value equality auto-generated
```

`record` (class) — reference type but value semantics. Best для DTOs / Value Objects.

### 3.4. String — special

```csharp
// String — reference type, but BEHAVES like value
string a = "hello";
string b = a;
b += " world";
Console.WriteLine(a);   // "hello" — unchanged!

// String immutable — concat creates new string
// Reference equality:
string s1 = "hello";
string s2 = "hello";
ReferenceEquals(s1, s2);   // true! — string interning

string s3 = new string(new[] { 'h', 'e', 'l', 'l', 'o' });
ReferenceEquals(s1, s3);   // false (different object)
s1 == s3;                  // true — value equality (string overrides ==)
```

### 3.5. Nullable reference

```csharp
public class Service
{
    public string? Name { get; set; }   // can be null (NRT enabled)
    public string Description { get; set; } = "";   // non-nullable, default ""
}
```

NRT — compile-time annotation only. Runtime — все references могут быть null.

### 3.6. Inheritance vs composition

```csharp
// Inheritance
public class Animal { public virtual void Speak() => Console.WriteLine("?"); }
public class Dog : Animal { public override void Speak() => Console.WriteLine("Woof"); }

// Composition (preferred)
public class Engine { public void Start() => Console.WriteLine("Vroom"); }
public class Car { private readonly Engine _engine = new(); public void Drive() => _engine.Start(); }
```

Best practice 2024+: **composition over inheritance** (rule of thumb).

> [!question]- Интервью: чем `record` отличается от `class`?
> **`class`** — reference type, default reference equality (compares object addresses). Mutable by default. Inheritance default. **`record`** (C# 9+) — reference type **с value semantics**: 1) **Auto-generated `Equals` / `GetHashCode`** — field-by-field comparison. 2) **`with`-expression** — non-destructive update (returns new instance). 3) **`Deconstruct`** for positional records. 4) **`ToString`** — formatted output. 5) **`==` / `!=`** — value equality. **Same as class**: heap-allocated, can inherit (other records), nullable. **Use cases**: DTOs, Value Objects, immutable data. **Don't use** для entities (need identity, не value equality). **C# 10+ `record struct`** — value type version. Best practice: records для immutable data, classes для mutable entities с identity.

---

## 4. ref struct, `Span<T>`, escape analysis

### 4.1. ref struct

```csharp
public ref struct Stack<T>
{
    private Span<T> _data;
    private int _size;
    
    public Stack(Span<T> initial)
    {
        _data = initial;
        _size = 0;
    }
    
    public void Push(T item)
    {
        if (_size >= _data.Length) throw new InvalidOperationException();
        _data[_size++] = item;
    }
}
```

`ref struct` — special struct type **stack-only**:
- Cannot be field of class or non-ref struct
- Cannot be boxed
- Cannot be generic type parameter (mostly)
- Cannot be in async / iterator (no await crossing)
- Cannot be heap-allocated

Used by `Span<T>`, `ReadOnlySpan<T>`, `ValueStringBuilder`.

### 4.2. `Span<T>`

```csharp
public readonly ref struct Span<T>
{
    // ref struct — stack-only
    // Provides safe access to contiguous memory
}

byte[] arr = new byte[1024];
Span<byte> span = arr.AsSpan();
Span<byte> slice = span.Slice(0, 100);   // no allocation!
```

См. [[memory-pooling]].

### 4.3. ref struct ограничения

```csharp
// ❌ ref struct cannot be class field
public class C
{
    private Span<int> _span;   // ❌ Compile error
}

// ❌ Cannot be in generic type
List<Span<int>> bad;   // ❌

// ❌ Cannot cross await
public async Task M()
{
    Span<int> s = stackalloc int[10];
    await Task.Delay(100);   // ❌
    s[0] = 1;
}

// ❌ Cannot be boxed
object o = mySpan;   // ❌
```

### 4.4. Escape analysis

JIT determines если object escapes method scope:

```csharp
public Point CreatePoint()
{
    var p = new Point { X = 1, Y = 2 };
    return p;   // ESCAPES (returned)
}

public double Distance()
{
    var p1 = new Point { X = 0, Y = 0 };
    var p2 = new Point { X = 3, Y = 4 };
    return p1.DistanceTo(p2);   // doesn't escape — JIT can stack-allocate
}
```

Escape analysis — JIT decides stack vs heap. .NET conservative compared to Go.

### 4.5. ref returns (C# 7+)

```csharp
public ref int FindFirst(int[] arr, int value)
{
    for (int i = 0; i < arr.Length; i++)
    {
        if (arr[i] == value) return ref arr[i];   // return reference
    }
    throw new InvalidOperationException();
}

int[] data = { 1, 2, 3, 4, 5 };
ref int first = ref FindFirst(data, 3);
first = 100;   // modifies array!
Console.WriteLine(data[2]);   // 100
```

`ref return` — return reference to memory. Used in performance-critical code (`Span<T>`, Dictionary internal).

### 4.6. ref locals

```csharp
int[] arr = { 1, 2, 3 };
ref int first = ref arr[0];   // alias to arr[0]
first = 100;
Console.WriteLine(arr[0]);   // 100

// Doesn't allocate — direct memory access
```

### 4.7. ref readonly (C# 7.2+)

```csharp
public readonly struct LargeStruct
{
    public readonly int A, B, C, D, E, F, G, H;   // 32 bytes
}

public ref readonly LargeStruct Get(int index, ref readonly LargeStruct[] arr) =>
    ref arr[index];

// Read-only reference — no copy, no modification
```

`ref readonly` — read-only reference. Avoids defensive copies for large readonly structs.

> [!question]- Интервью: что такое `ref struct` и когда использовать?
> Special struct type **forced stack-only**. Cannot be: 1) Class field. 2) Non-ref struct field. 3) Generic type argument (mostly). 4) Boxed. 5) In async / iterator (cross await forbidden). **Use cases**: 1) **`Span<T>` / `ReadOnlySpan<T>`** — safe stack-bound memory access. 2) **`ValueStringBuilder`** — stack-allocated string building. 3) Custom stack-only types для performance. **Why**: prevents heap allocation, ensures lifetime within method scope, JIT can optimize (no GC tracking). **Trade-offs**: cannot store as field, async limitations. **Best practice**: rarely write own `ref struct`. Use `Span<T>` / `Memory<T>` / `ValueStringBuilder` (provided by BCL).

---

## 5. Stack vs Heap

### 5.1. Stack

```
Stack:
- LIFO data structure
- Per-thread (one stack per thread)
- Small (~1 MB default)
- Fast allocation (just bump pointer)
- Auto-cleanup при method return
- No GC overhead
```

Stores:
- Local value type variables
- Method parameters (often)
- Return addresses
- `stackalloc` allocations

### 5.2. Heap

```
Heap:
- Random allocation
- Shared across threads
- Large (limited by RAM)
- Slower allocation (find free block, GC overhead)
- Cleanup by GC
```

Stores:
- All reference type instances
- Boxed value types
- Strings
- Arrays
- Closure captures (lambda)

### 5.3. Стack overflow

```csharp
public int Recurse(int n)
{
    return Recurse(n + 1);   // ❌ Eventually stack overflow!
}

// Default ~1MB stack:
// 64KB / function frame = 16,000 calls before overflow
```

### 5.4. Heap allocation visible

```csharp
// All heap allocations
var list = new List<int>();   // List object
var array = new int[100];     // int[] array
var str = "hello";            // string (interned)
var lambda = () => x;          // closure object capturing x

Action a = () => Console.WriteLine();   // delegate object
```

### 5.5. Closures и captures

```csharp
public Action CreateClosure()
{
    int x = 10;
    return () => Console.WriteLine(x);   // captures x — heap allocation!
}

// Compiler generates anonymous class:
// class Closure { public int x; public void Method() => Console.WriteLine(x); }
// new Closure { x = 10 }.Method
```

Closures = heap allocation per call (creating closure). Hot path → avoid.

### 5.6. struct в array

```csharp
// Reference types — array of pointers
class Item { public int X; }
Item[] items = new Item[100];   // 100 nulls (8 bytes each = 800B)
items[0] = new Item();           // separate heap allocation per item

// Struct — inline в array
struct ItemStruct { public int X; }
ItemStruct[] structs = new ItemStruct[100];   // 100 inline ints (400B)
structs[0].X = 5;   // direct access
```

Struct arrays — much more cache-friendly. Used heavily в game engines (ECS pattern).

> [!question]- Интервью: где value types аллоцируются?
> **Не only stack** — depends on context: 1) **Local variables** in method — stack. 2) **Method parameters** — stack (часто). 3) **Field of class** — heap (inside parent class). 4) **Field of struct** — wherever parent struct is. 5) **Array element** — heap (array on heap, but inline within array). 6) **Boxed** — heap (in box object). 7) **Captured by closure** — heap (in compiler-generated closure class). **JIT escape analysis**: even reference types may stack-allocate when JIT proves no escape (rarely в .NET vs Go). Best practice: don't assume "value = stack"; struct stored wherever logically belongs. For guaranteed stack: `stackalloc Span<T>`.

---

## 6. Garbage Collector (GC)

### 6.1. Generations

```
Gen 0:
- New objects allocated here
- Frequent collection (~ms)
- Most objects die here

Gen 1:
- Objects survived Gen 0 collection
- Less frequent
- Buffer between Gen 0 и Gen 2

Gen 2:
- Long-lived objects
- Infrequent collection (slow)
- Includes Large Object Heap

LOH (Large Object Heap):
- Objects > 85,000 bytes
- Goes directly to Gen 2
- Not compacted (default)
- Fragmentation issues

POH (Pinned Object Heap, .NET 5+):
- Pinned objects (для interop)
- Long-lived pinned data
```

### 6.2. Generational hypothesis

```
Most objects die young:
- Tempo variables (LINQ chains)
- Closures
- Boxed values
- Strings concatenations

Long-lived объекты:
- Caches
- Singletons
- App configuration

Generational GC — frequent fast collection of new objects.
```

### 6.3. GC modes

```
Workstation GC (default for desktop):
- One GC thread per process
- Optimized for low-latency interactive apps

Server GC (default for ASP.NET Core):
- Multiple GC threads (one per CPU core)
- Larger heaps
- Better throughput
- Higher memory usage

Background GC:
- Gen 2 collection in background thread
- Reduces pauses

Concurrent GC (legacy term — same as Background GC)
```

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
</PropertyGroup>
```

### 6.4. Allocation cost

```
Allocation:
- Bump pointer (~1ns)
- Cheap

Collection cost:
- Gen 0: ~1ms
- Gen 1: ~5-10ms
- Gen 2: ~50-200ms
- Full GC: even slower

Goal: reduce promotions to Gen 1/2.
```

### 6.5. GC tuning

```xml
<!-- ASP.NET Core production settings -->
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
  <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
  <RetainVMGarbageCollection>true</RetainVMGarbageCollection>
</PropertyGroup>
```

```csharp
// Force GC (rarely needed!)
GC.Collect();   // ❌ usually anti-pattern

// Suggest collection
GC.Collect(2, GCCollectionMode.Optimized);

// Settings
GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;
```

### 6.6. Finalizers vs IDisposable

```csharp
public class Resource : IDisposable
{
    private bool _disposed;
    
    ~Resource()   // finalizer
    {
        Dispose(false);
    }
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        if (disposing)
        {
            // cleanup managed resources
        }
        // cleanup unmanaged resources
        _disposed = true;
    }
}
```

Finalizers — slow (Gen 2 + extra GC cycle). Avoid если possible. `SafeHandle` лучше.

### 6.7. Weak references

```csharp
var data = new LargeData();
var weak = new WeakReference<LargeData>(data);
data = null;   // strong reference gone

// Later
if (weak.TryGetTarget(out var d))
{
    // d still alive — use it
}
else
{
    // d collected — recompute
}
```

`WeakReference<T>` — allows GC to collect target. Used in caches.

> [!question]- Интервью: что такое generational GC?
> .NET GC организует heap в **generations** based on hypothesis "most objects die young": 1) **Gen 0** — new allocations. Frequent fast collection (~1ms). Most objects die here. 2) **Gen 1** — survived Gen 0. Buffer between 0 и 2. 3) **Gen 2** — long-lived objects + LOH (objects > 85,000 bytes). Slow collection (50-200ms). 4) **POH** (.NET 5+) — pinned objects. **Why**: collecting all heap every time slow. Generational allows frequent fast Gen 0 collection без touching long-lived objects. **Modes**: Workstation (single thread, low-latency) vs Server (multi-thread, throughput). **Best practice**: avoid promoting young objects to Gen 1/2 (use ArrayPool, avoid closures в hot path, use struct для ephemeral data). LOH allocations — costly (Gen 2 only).

---

## 7. LOH и POH — special heaps

### 7.1. Large Object Heap (LOH)

```
LOH:
- Objects > 85,000 bytes
- Allocations directly to Gen 2
- Not compacted by default (fragmentation!)
- Slow allocation (search for free block)
- Slow collection (full Gen 2)
```

### 7.2. LOH fragmentation

```csharp
// LOH fragments over time
for (int i = 0; i < 1000; i++)
{
    var big = new byte[100_000];   // LOH
    // big released
}

// Free space но fragmented — может cause OutOfMemoryException даже с RAM
```

### 7.3. LOH компактификация (.NET 4.5.1+)

```csharp
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect();   // performs LOH compaction
```

Compact LOH manually. **Slow** — only when fragmentation problem detected.

### 7.4. Avoiding LOH

```csharp
// ❌ Common LOH allocations
byte[] huge = new byte[1_000_000];   // 1MB, LOH
string megaString = new string('x', 100_000);   // ~200KB, LOH

// ✅ Use ArrayPool for large buffers
byte[] buffer = ArrayPool<byte>.Shared.Rent(1_000_000);
try { /* use buffer */ }
finally { ArrayPool<byte>.Shared.Return(buffer); }

// ✅ Or chunked processing
const int ChunkSize = 8192;
for (int offset = 0; offset < total; offset += ChunkSize)
{
    var chunk = new byte[Math.Min(ChunkSize, total - offset)];
    // process
}
```

### 7.5. Pinned Object Heap (POH, .NET 5+)

```csharp
// Pinned objects via GC.AllocateUninitializedArray
byte[] pinnedBuffer = GC.AllocateUninitializedArray<byte>(1024, pinned: true);

// Used для long-lived pinned data (interop, P/Invoke)
// Doesn't fragment regular Gen 0/1/2 heaps
```

POH — separate region для pinned objects. Reduces fragmentation в normal heap.

### 7.6. Monitoring

```bash
# dotnet-counters
dotnet-counters monitor --process-id <pid> System.Runtime
# Watch:
# - gen-0-gc-count, gen-1-gc-count, gen-2-gc-count
# - loh-size
# - pause-time-percentage
```

```csharp
// Programmatic
long allocated = GC.GetTotalMemory(forceFullCollection: false);
long gen0 = GC.CollectionCount(0);
long gen1 = GC.CollectionCount(1);
long gen2 = GC.CollectionCount(2);
```

> [!question]- Интервью: что такое LOH и почему важен?
> **Large Object Heap** — heap для objects > 85,000 bytes. **Differences from Gen 0/1/2**: 1) **Direct to Gen 2** — no Gen 0/1 path. 2) **Not compacted by default** — fragmentation possible. 3) **Slow allocation** — search free block. 4) **Slow collection** — full Gen 2 GC. **Why 85,000?**: empirical threshold where copy cost > compaction overhead. **Common LOH allocations**: large `byte[]`, large `string`, large arrays, double[]. **Avoiding**: `ArrayPool<byte>` для large buffers, chunked processing, `Stream.CopyToAsync` instead of `ReadToEnd`, careful с `JsonSerializer` для large objects. **Symptoms LOH issues**: OutOfMemoryException с unused RAM, long Gen 2 pauses, growing working set. **Tools**: dotnet-counters, PerfView, dotMemory.

---

## 8. Memory profiling

### 8.1. dotnet-counters

```bash
dotnet tool install -g dotnet-counters
dotnet-counters monitor --process-id <pid> System.Runtime
```

Real-time runtime metrics: GC counts, heap size, allocation rate, working set.

### 8.2. dotnet-trace

```bash
dotnet-trace collect --process-id <pid> --providers Microsoft-DotNETCore-SampleProfiler
# Stops with Ctrl+C, generates .nettrace file
```

Performance traces — analyzed via PerfView, Speedscope.

### 8.3. dotnet-dump

```bash
dotnet-dump collect --process-id <pid>
dotnet-dump analyze core.<pid>
> dumpheap -stat
> gcroot <objaddr>
```

Memory dump analysis — find leaks, big objects, references.

### 8.4. PerfView

```
Microsoft tool — comprehensive ETW analysis:
- Memory profiling
- CPU sampling
- Allocation tracking
- GC events

Steps:
1. Collect → ETW trace
2. Open .etl file
3. GC Heap Alloc Stack — top allocations
4. GC Heap Net Mem — leaks
```

### 8.5. dotMemory (JetBrains)

GUI memory profiler. Easier than PerfView. Used during development.

### 8.6. BenchmarkDotNet

```csharp
[MemoryDiagnoser]
public class Benchmarks
{
    [Benchmark(Baseline = true)]
    public void Allocating()
    {
        var list = new List<int>();
        for (int i = 0; i < 1000; i++) list.Add(i);
    }
    
    [Benchmark]
    public void Pooled()
    {
        var buffer = ArrayPool<int>.Shared.Rent(1000);
        try
        {
            for (int i = 0; i < 1000; i++) buffer[i] = i;
        }
        finally
        {
            ArrayPool<int>.Shared.Return(buffer);
        }
    }
}

// Output:
// |    Method | Allocated |
// |---------- |----------:|
// | Allocating |   8,224 B |
// | Pooled    |       0 B |
```

`[MemoryDiagnoser]` — shows allocations per benchmark.

### 8.7. ApplicationInsights / OpenTelemetry

```csharp
// Production observability
services.AddApplicationInsightsTelemetry();
// Or
services.AddOpenTelemetry().WithMetrics(builder =>
{
    builder.AddRuntimeInstrumentation();
});
```

Metrics: GC counts, heap size, allocation rate exported to Azure Monitor / Prometheus.

> [!question]- Интервью: какие tools для memory profiling в .NET?
> 1) **dotnet-counters** (CLI) — real-time runtime metrics, GC counts, heap size, allocation rate. Free, simple. 2) **dotnet-trace** + **PerfView** — ETW traces, allocation hot paths, GC events. Free, powerful, complex. 3) **dotnet-dump** + **SOS** — memory dump analysis, find leaks/big objects. 4) **dotMemory** (JetBrains) — GUI profiler, easier learning curve, paid. 5) **BenchmarkDotNet с `[MemoryDiagnoser]`** — measure allocations per operation в micro-benchmarks. 6) **ApplicationInsights / OpenTelemetry** — production observability, GC metrics exported. **Workflow**: 1) Suspicion (slow / OOM) → counters check. 2) Hot path → BenchmarkDotNet measure. 3) Production leak → dotMemory or dump analysis. 4) Detailed allocation → PerfView trace.

---

## 9. Performance hot path optimizations

### 9.1. Avoid allocations

```csharp
// ❌ Allocates per call
public string Format(int x) => $"Value: {x}";   // string + boxing

// ✅ Cached
private static readonly string Prefix = "Value: ";
public string Format(int x) => Prefix + x;
// Still string concat but no boxing if int → string fast path
```

### 9.2. Stack vs heap для small data

```csharp
// ❌ Heap
class Point3D { public double X, Y, Z; }
var p = new Point3D { X = 1, Y = 2, Z = 3 };

// ✅ Stack
struct Point3D { public double X, Y, Z; }
var p = new Point3D { X = 1, Y = 2, Z = 3 };
```

### 9.3. `Span<T>` + stackalloc

```csharp
// ❌ Heap allocation per parse
public int Parse(string s)
{
    var trimmed = s.Trim();   // allocation!
    return int.Parse(trimmed);
}

// ✅ Span на stack
public int Parse(string s)
{
    Span<char> buf = stackalloc char[s.Length];
    int len = s.AsSpan().Trim(buf);   // no allocation
    return int.Parse(buf.Slice(0, len));
}
```

### 9.4. ArrayPool для buffers

```csharp
// ❌ New array per call
public byte[] Compress(byte[] data)
{
    var buffer = new byte[data.Length * 2];
    // process
    return buffer;
}

// ✅ Pool
public byte[] Compress(byte[] data)
{
    var buffer = ArrayPool<byte>.Shared.Rent(data.Length * 2);
    try
    {
        // process
        return buffer.AsSpan(0, length).ToArray();   // copy only what's needed
    }
    finally { ArrayPool<byte>.Shared.Return(buffer); }
}
```

### 9.5. Avoid LINQ в hot paths

```csharp
// ❌ LINQ allocations (Where, Select, ToList all allocate)
public int Sum(List<int> list) => list.Where(x => x > 0).Select(x => x * 2).Sum();

// ✅ Direct loop — no allocations
public int Sum(List<int> list)
{
    int total = 0;
    foreach (var x in list)
    {
        if (x > 0) total += x * 2;
    }
    return total;
}
```

LINQ — convenient но allocates iterators. Avoid в tight loops (game frame, request handling).

### 9.6. Avoid closures в hot paths

```csharp
// ❌ Closure allocates per call
public void Process(List<int> items, int threshold)
{
    items.Where(x => x > threshold)   // closure captures threshold
        .ToList();
}

// ✅ Manual loop
public void Process(List<int> items, int threshold)
{
    var result = new List<int>();
    foreach (var x in items)
    {
        if (x > threshold) result.Add(x);
    }
}
```

Если лямбда всё-таки нужна — помечай её `static` (C# 9+). Это не «оптимизация на потом», а compile-time гарантия: компилятор **запрещает** захват любого внешнего состояния, поэтому случайный capture (а с ним `<>c__DisplayClass`-аллокация на каждый вызов) становится ошибкой сборки, а не тихой регрессией. Состояние передавай явно через state-passing перегрузки `Func<TState, T>` — в BCL их теперь много (`ConcurrentDictionary.GetOrAdd`, `MemoryCache`, `String.Create`, `Enumerable.Aggregate` и т.д.).

```csharp
// ❌ captures threshold → компилятор генерит <>c__DisplayClass, alloc на каждый вызов
public List<int> Filter(List<int> items, int threshold) =>
    items.Where(x => x > threshold).ToList();
```

```csharp
// ✅ static lambda + state-passing перегрузка: захват ЗАПРЕЩЁН компилятором, 0 closure-alloc
public bool GetOrAddFlag(ConcurrentDictionary<string, bool> cache, string key, int threshold) =>
    cache.GetOrAdd(key, static (_, t) => t > 0, threshold);   // threshold идёт аргументом, не захватом
```

```csharp
// ✅ static lambda не компилируется, если попытаться что-то захватить — баг ловится на сборке
static int CapturesNothing(int x) => x * 2;
// var bad = static () => threshold;  // CS8820: a static lambda cannot capture 'threshold'
```

> [!info]- Как найти утечку: dotnet-gcdump показывает `<>c__DisplayClass`

> Захват, доживший до Gen 2 (лямбда, подписанная на долгоживущий event/таймер/кэш), держит весь `DisplayClass` со всеми захваченными полями. В дампе `dotnet-gcdump collect --process-id <pid>` тип имеет вид `MyNamespace.MyService+<>c__DisplayClass12_0` — суффикс `<>c__DisplayClass` плюс **имя метода**, в котором объявлена лямбда, прямо называют источник. Дальше `gcroot` по этому объекту показывает удерживающий event/делегат. `static`-лямбды этого класса не создают вовсе.

### 9.7. String operations

```csharp
// ❌ Concatenation в loop
string result = "";
foreach (var x in items) result += x.ToString();   // O(n²) allocations!

// ✅ StringBuilder
var sb = new StringBuilder();
foreach (var x in items) sb.Append(x);
string result = sb.ToString();

// ✅ String.Join
string result = string.Join(",", items);
```

### 9.8. Struct enumerator

```csharp
// List<T>.Enumerator — struct (no allocation на foreach)
public struct CustomEnumerator
{
    public bool MoveNext() => /* ... */;
    public T Current => /* ... */;
}

// Custom collection с struct enumerator — no boxing
public struct CustomCollection<T> : IEnumerable<T>
{
    public CustomEnumerator GetEnumerator() => /* ... */;
    // Compiler uses struct enumerator если returns struct directly
}
```

### 9.9. Типизация коллекции как `IEnumerable<T>` боксит struct-энумератор

Почему один лишний alloc там, где «никто ничего не аллоцирует»: `List<T>.GetEnumerator()` возвращает **struct** `List<T>.Enumerator`. Когда переменную типизируют как `IEnumerable<T>`, `foreach` вынужден вызывать интерфейсный `IEnumerable<T>.GetEnumerator()`, который возвращает `IEnumerator<T>` — а это значит struct-энумератор **боксится один раз на каждый `foreach`** (одна heap-аллокация на вызов). Плюс каждый `MoveNext`/`Current` идёт через interface dispatch (virtual), теряя инлайнинг.

```csharp
// ❌ поле/параметр типа IEnumerable<int>: struct-энумератор боксится на каждом foreach
private readonly IEnumerable<int> _items;   // под капотом List<int>, но тип стёрт

public int SumBoxed()
{
    int total = 0;
    foreach (var x in _items) total += x;   // box List<int>.Enumerator + virtual MoveNext
    return total;
}
```

```csharp
// ✅ конкретный тип: foreach берёт struct-энумератор по значению, 0 аллокаций
private readonly List<int> _items;

public int SumConcrete()
{
    int total = 0;
    foreach (var x in _items) total += x;   // struct enumerator, no box, MoveNext инлайнится
    return total;
}
```

На hot path: принимай и храни конкретный `List<T>` / `T[]`, либо итерируй по индексу (`for` + `Count`/`Length`). Для публичного API не заставляй вызывающего боксить — добавь generic-перегрузку:

```csharp
// Публичный API: generic-перегрузка вместо единственной IEnumerable<T>
public static int Sum<TList>(TList items) where TList : IReadOnlyList<int>
{
    int total = 0;
    for (int i = 0; i < items.Count; i++) total += items[i];   // no enumerator box
    return total;
}
```

> [!info]- Как доказать: concrete-vs-interface MemoryDiagnoser delta

> ```csharp
> [MemoryDiagnoser]
> public class EnumeratorBoxingBench
> {
>     private readonly List<int> _list = Enumerable.Range(0, 1000).ToList();
>
>     [Benchmark(Baseline = true)]
>     public int OverInterface()
>     {
>         IEnumerable<int> seq = _list;   // тип стёрт до интерфейса
>         int total = 0;
>         foreach (var x in seq) total += x;   // boxes struct enumerator
>         return total;
>     }
>
>     [Benchmark]
>     public int OverConcrete()
>     {
>         int total = 0;
>         foreach (var x in _list) total += x;   // struct enumerator, no box
>         return total;
>     }
> }
> ```
>
> Ожидаемый delta: `OverInterface` — `Allocated ≈ 40 B` (один boxed `List<int>.Enumerator` на вызов), `OverConcrete` — `0 B`. Та же 40-байтная аллокация умножается на частоту вызова: на 1M `foreach`/сек это ~40 MB/сек постоянного Gen 0 давления буквально из-за типа переменной.

> [!question]- Интервью: как уменьшить allocations в hot path?
> 1) **Use structs для small data** (Point, Money, < 32 bytes). 2) **`Span<T>` + `stackalloc`** для temporary buffers. 3) **`ArrayPool<T>`** для large arrays (> 1KB). 4) **Avoid LINQ в tight loops** — manual foreach. 5) **Avoid closures** — closures allocate (compiler generates class). 6) **`StringBuilder`** для string concat в loops. 7) **Cache reusable objects** (StringBuilder, regex). 8) **`IEquatable<T>`** для structs — avoids boxing в comparisons. 9) **Reuse delegates** (cache as field). 10) **`ValueTask<T>`** instead of `Task<T>` для completed-sync paths. **Tools**: BenchmarkDotNet `[MemoryDiagnoser]` measures. PerfView/dotMemory finds hot allocations. **Trade-off**: optimization complexity vs benefit. Profile first.

---

## 10. Best practices

### 10.1. Type choice

- ✅ **`struct` для < 32 bytes immutable values** (Point, Money, DateTime).
- ✅ **`readonly struct`** — always for value types.
- ✅ **`record struct`** для small immutable Value Objects.
- ✅ **`class` для entities** (с identity, mutable).
- ✅ **`record class`** для DTOs / Value Objects (reference type semantics + value equality).
- ❌ **Mutable struct** — defensive copies, confusing semantics.
- ❌ **Large struct** > 32 bytes — copy cost.
- ❌ **`struct` с inheritance** — not supported.

### 10.2. Memory

- ✅ **`Span<T>` / `stackalloc`** для temporary buffers.
- ✅ **`ArrayPool<T>`** для large buffers (> 1KB).
- ✅ **`IEquatable<T>`** на structs — avoid boxing.
- ✅ **`StringBuilder`** для concat в loops.
- ❌ **Boxing** в hot path — `List<object>`, `string.Format("{0}", int)`.
- ❌ **Closures** в tight loops.
- ❌ **LINQ allocations** в performance-critical code.

### 10.3. GC

- ✅ **Server GC + Background GC** для server apps.
- ✅ **`SafeHandle`** для unmanaged resources (avoid finalizers).
- ✅ **`IDisposable` + `using`** для managed resources.
- ✅ **Avoid LOH** через ArrayPool / chunking.
- ❌ **`GC.Collect()` в production** — anti-pattern.
- ❌ **Finalizers** unless absolutely needed.
- ❌ **Manual memory management** — let GC do it.

### 10.4. Profiling

- ✅ **Profile before optimize** — assumptions wrong часто.
- ✅ **BenchmarkDotNet** для micro-benchmarks.
- ✅ **dotnet-counters** для production monitoring.
- ✅ **PerfView / dotMemory** для allocation hot paths.
- ❌ **Premature optimization** — readability matters.

### 10.5. Не делай

- ❌ Store struct в `List<object>` — boxes everything.
- ❌ Mutable struct (`public int X { get; set; }`) — confusing.
- ❌ Large struct passed by value часто — copy cost.
- ❌ `ref struct` в `async` method — compile error.
- ❌ Force GC.Collect.

---

## 11. Decision tree

```
Что выбрать?
│
├── New type — struct или class?
│   ├── < 32 bytes immutable value (Point, Money) → readonly record struct
│   ├── DTO / Value Object с value equality → record (class)
│   ├── Entity с identity (User, Order) → class
│   ├── Mutable data → class (mutable struct confusing)
│   ├── Inheritance needed → class (struct cannot inherit)
│   └── Performance critical / no allocations → struct (small) или ref struct
│
├── Buffer для temp work
│   ├── Small (< 1KB) → stackalloc Span<T>
│   ├── Medium (1KB-85KB) → ArrayPool<T>.Shared.Rent
│   ├── Large (> 85KB) → ArrayPool<T> с custom configuration
│   └── Persistent → new T[] (only если really needed)
│
├── Avoiding allocations
│   ├── String concat → StringBuilder или string.Join
│   ├── List building → reuse list или ArrayPool
│   ├── Closures → manual loop
│   ├── LINQ → foreach в hot path
│   └── Boxing → IEquatable<T>, generics
│
├── GC issues
│   ├── Server / ASP.NET → ServerGC + Background GC
│   ├── LOH fragmentation → ArrayPool, chunked processing
│   ├── Long-lived pinned → POH (.NET 5+)
│   └── Memory leak → dotMemory / PerfView analysis
│
└── Profile
    ├── Real-time → dotnet-counters
    ├── Allocation hot paths → BenchmarkDotNet [MemoryDiagnoser]
    ├── Production → PerfView + ApplicationInsights
    └── Memory dump → dotnet-dump + SOS
```

---

## 12. Cheat sheet

```csharp
// === Value types ===
public readonly record struct Point(int X, int Y);
public readonly struct Money(decimal Amount, string Currency);

var p1 = new Point(1, 2);
var p2 = p1 with { X = 10 };   // copy + modify

// === Reference types ===
public record User(int Id, string Name);   // reference type, value equality
public class Order { public int Id { get; init; } }   // entity

// === Stack allocation ===
Span<int> buffer = stackalloc int[100];   // stack
buffer[0] = 1;

// Hybrid stack/heap
Span<int> buffer = size <= 256
    ? stackalloc int[256]
    : new int[size];

// === ArrayPool ===
byte[] data = ArrayPool<byte>.Shared.Rent(8192);
try { /* use */ }
finally { ArrayPool<byte>.Shared.Return(data); }

// === ref struct ===
public ref struct ValueStringBuilder
{
    private Span<char> _chars;
    private int _pos;
}

// === ref returns ===
public ref int FindFirst(int[] arr, int value)
{
    for (int i = 0; i < arr.Length; i++)
        if (arr[i] == value) return ref arr[i];
    throw new InvalidOperationException();
}

// === Avoid boxing ===
public bool Equals<T>(T a, T b) where T : struct, IEquatable<T> =>
    a.Equals(b);   // no boxing

// === GC settings ===
// <ServerGarbageCollection>true</ServerGarbageCollection>
// <ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>

// === Benchmarking ===
[MemoryDiagnoser]
public class Bench
{
    [Benchmark] public void Allocating() => new byte[1024];
    [Benchmark] public void Pooled()
    {
        var buf = ArrayPool<byte>.Shared.Rent(1024);
        ArrayPool<byte>.Shared.Return(buf);
    }
}

// === Profiling commands ===
// dotnet-counters monitor --process-id <pid> System.Runtime
// dotnet-trace collect --process-id <pid>
// dotnet-dump collect --process-id <pid>
```

---

## 13. Common pitfalls

### 13.1. Mutable struct surprise

```csharp
struct Counter { public int Value; }

readonly Counter _counter = new();
_counter.Value = 5;   // ❌ doesn't modify _counter (defensive copy для readonly)
```

**Фикс:** `readonly struct` + `init`-only properties.

### 13.2. Struct в `List<object>`

```csharp
struct Point { public int X, Y; }
List<object> list = new();
var p = new Point { X = 1, Y = 2 };
list.Add(p);   // ❌ BOXING

var retrieved = (Point)list[0];   // unboxing — copy
```

**Фикс:** `List<Point>` (generic).

### 13.3. Foreach on collection с boxing

```csharp
ArrayList list = new();   // ❌ legacy, non-generic
list.Add(1);
foreach (int x in list)   // unbox per iteration!
{
    // process
}
```

**Фикс:** `List<int>`.

### 13.4. Closure в hot path

```csharp
public void Process(int[] items)
{
    int threshold = 10;
    items.Where(x => x > threshold).ToList();   // ❌ closure + LINQ allocations
}
```

**Фикс:** manual foreach.

### 13.5. ref struct + async

```csharp
public async Task M()
{
    Span<byte> buf = stackalloc byte[100];   // ❌ ref struct
    await Task.Delay(100);   // Compile error
    buf[0] = 1;
}
```

**Фикс:** restructure — sync work, async outside.

### 13.6. LOH allocations

```csharp
public byte[] Read(string path)
{
    return File.ReadAllBytes(path);   // ❌ файл 1MB → LOH allocation
}
```

**Фикс:** stream processing с small buffer.

### 13.7. Finalizer без SuppressFinalize

```csharp
public class Resource : IDisposable
{
    ~Resource() { /* cleanup */ }
    public void Dispose() { /* cleanup */ }   // ❌ no SuppressFinalize
}
// Resource ends на finalization queue даже if Dispose called!
```

**Фикс:** `GC.SuppressFinalize(this)` в `Dispose()`.

### 13.8. Equals не overridden для struct

```csharp
struct Point { public int X, Y; }

var p1 = new Point { X = 1, Y = 2 };
var p2 = new Point { X = 1, Y = 2 };
p1.Equals(p2);   // works но slow (reflection-based по default)
```

**Фикс:** `IEquatable<T>` или `record struct`.

### 13.9. Large struct по value

```csharp
public struct Matrix4x4 { public float[16] elements; }   // 64 bytes

void Process(Matrix4x4 m) { /* ... */ }   // ❌ copy 64 bytes per call
```

**Фикс:** `in` parameter (read-only ref):
```csharp
void Process(in Matrix4x4 m) { /* ... */ }
```

### 13.10. GC.Collect() в production

```csharp
public void HandleRequest()
{
    // ... work
    GC.Collect();   // ❌ stops world, slow
}
```

**Фикс:** trust GC, profile if issues.

> [!question]- Интервью: топ-3 ошибки с memory в C#?
> 1) **Boxing surprise** — `List<object>` + value type, `string.Format("{0}", int)`, mutable struct в interface variable. Each = heap allocation. Profile с BenchmarkDotNet `[MemoryDiagnoser]`. Fix: generics + `IEquatable<T>` + specific overloads. 2) **Mutable struct** — `readonly struct` field defensive copy when modifying property. Confusing semantics. Fix: `readonly struct` + `init` properties (or `record struct`). 3) **LOH allocation** — `byte[1_000_000]` → Gen 2 directly, fragmentation, slow. Fix: ArrayPool<byte> для large buffers, chunked stream processing. Бонус: closures в tight loops — compiler generates class per closure invocation. Manual loop без captures.

---

## 14. Practice exercises

### 14.1. Convert class to struct (если applicable)

```csharp
// Before — class (heap allocation per instance)
public class Vector2D
{
    public double X { get; set; }
    public double Y { get; set; }
    
    public double Magnitude => Math.Sqrt(X * X + Y * Y);
    public Vector2D Normalize() => new() { X = X / Magnitude, Y = Y / Magnitude };
}

// After — readonly record struct
public readonly record struct Vector2D(double X, double Y)
{
    public double Magnitude => Math.Sqrt(X * X + Y * Y);
    public Vector2D Normalize() => this with { X = X / Magnitude, Y = Y / Magnitude };
}

// Benchmark — struct version much faster для tight loops
[MemoryDiagnoser]
public class VectorBenchmarks
{
    private readonly Vector2D[] _vectors = Enumerable.Range(0, 10000)
        .Select(i => new Vector2D(i, i)).ToArray();
    
    [Benchmark]
    public double SumMagnitudes()
    {
        double sum = 0;
        foreach (var v in _vectors)
            sum += v.Magnitude;
        return sum;
    }
}
```

### 14.2. Reduce allocations в string processing

```csharp
// Before — many allocations
public List<string> ParseCsv(string csv)
{
    return csv.Split('\n')   // allocation
        .Select(line => line.Trim())   // allocation per line
        .Where(line => !string.IsNullOrEmpty(line))
        .SelectMany(line => line.Split(','))
        .ToList();
}

// After — span-based, no allocations until ToString
public List<string> ParseCsv(ReadOnlySpan<char> csv)
{
    var result = new List<string>();
    int lineStart = 0;
    
    for (int i = 0; i <= csv.Length; i++)
    {
        if (i == csv.Length || csv[i] == '\n')
        {
            var line = csv.Slice(lineStart, i - lineStart).Trim();
            if (line.Length > 0)
            {
                int fieldStart = 0;
                for (int j = 0; j <= line.Length; j++)
                {
                    if (j == line.Length || line[j] == ',')
                    {
                        var field = line.Slice(fieldStart, j - fieldStart).Trim();
                        if (field.Length > 0) result.Add(field.ToString());   // alloc only here
                        fieldStart = j + 1;
                    }
                }
            }
            lineStart = i + 1;
        }
    }
    
    return result;
}
```

### 14.3. Detect и fix LOH issue

```csharp
// Before — LOH allocations per request
public class FileService
{
    public async Task<byte[]> ReadFileAsync(string path)
    {
        return await File.ReadAllBytesAsync(path);   // ❌ файл 5MB → LOH
    }
}

// After — chunked streaming + ArrayPool
public class FileService
{
    public async Task<long> ProcessFileAsync(string path, Func<ReadOnlyMemory<byte>, Task> processor)
    {
        var buffer = ArrayPool<byte>.Shared.Rent(8192);
        long total = 0;
        try
        {
            using var stream = File.OpenRead(path);
            int read;
            while ((read = await stream.ReadAsync(buffer)) > 0)
            {
                await processor(buffer.AsMemory(0, read));
                total += read;
            }
            return total;
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(buffer);
        }
    }
}

// Use
await fileService.ProcessFileAsync("big.dat", chunk =>
{
    // process chunk
    return Task.CompletedTask;
});
```

---

## 15. Что читать дальше

1. **[[memory-pooling|Memory Pooling]]** — Span / ArrayPool deep.
2. **[[unsafe-pointers|Unsafe / AOT]]** — manual memory.
3. **[[generics-deep|Generics Deep]]** — boxing avoidance.
4. **Konrad Kokosa — "Pro .NET Memory Management"** (book).
5. **Maoni Stephens (.NET GC team) blog**.

---

## 16. См. также

- [[memory-pooling|Memory Pooling]]
- [[unsafe-pointers|Unsafe / AOT]]
- [[generics-deep|Generics Deep]] — value/reference distinction
- [[dispose-pattern|Dispose Pattern]] — IDisposable + finalizers
- [[modern-features|Modern Features]] — record struct
- System.Buffers — ArrayPool
- System.Memory — `Span<T>`

---

## 17. Reading list

- **Microsoft Docs — Garbage collection** — learn.microsoft.com/dotnet/standard/garbage-collection/
- **Microsoft Docs — Value types vs Reference types** — learn.microsoft.com
- **Konrad Kokosa — "Pro .NET Memory Management"** (book)
- **Maoni Stephens — .NET GC blog** — devblogs.microsoft.com (.NET GC architect)
- **Stephen Toub — Performance series** — devblogs.microsoft.com
- **Adam Sitnik — Performance** — adamsitnik.com
- **BenchmarkDotNet docs** — benchmarkdotnet.org
- **PerfView** — github.com/microsoft/perfview
- **dotnet-counters / dotnet-trace docs** — learn.microsoft.com/dotnet/core/diagnostics/
- **Sasha Goldshtein — Pro .NET Performance** (book)
