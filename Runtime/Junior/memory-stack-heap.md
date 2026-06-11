---
tags: [runtime, memory, stack, heap, junior, value-type, reference-type, boxing]
level: Junior
date: 2026-05-10
---

# Memory — Stack, Heap, Value/Reference Types

> **Где живут переменные, как работает копирование, что такое boxing, почему `string` immutable.** Введение перед `Runtime/gc-memory.md` (deep GC) и `CSharp/Senior/types-and-memory.md`.

---

## 0. Как читать

Если впервые — раздел 1 (mental model) → 2 (value/ref) → 3 (stack/heap). После — 4 (passing semantics), 5 (boxing). Глубокий treatment GC generations + memory layouts — Senior файлы.

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Mental model — где живут переменные

### 1.1. Главная картинка

```
Process Memory Layout (simplified):

┌─────────────────────────────────┐
│  Thread Stack (per thread)      │  ← LIFO, fast, ~1MB
│  ─ local variables (value types)│
│  ─ method parameters            │
│  ─ return addresses             │
│                                 │
│            ↓ growing            │
│                                 │
│            ↑ growing            │
│                                 │
│  Managed Heap                   │  ← Shared, GC-managed
│  ─ all reference type objects   │
│  ─ boxed value types            │
└─────────────────────────────────┘
```

### 1.2. Value type vs Reference type — grand picture

```csharp
// Value types — на stack (если local) или inline в containing object
int age = 25;
DateTime now = DateTime.Now;
struct Point { int X, Y; }

// Reference types — на heap, на stack только pointer
class User { ... }
string name = "Alice";
List<int> list = new();
```

```
After: int age = 25; User user = new("Alice");

Stack (current method):
┌──────────────┐
│ age = 25     │  ← value прямо в stack
│ user = 0x... │  ← pointer (8 bytes на x64)
└──────────────┘
                  │
                  ↓
Heap:        ┌────────────────────────┐
             │ User { Name="Alice" }  │  ← actual object
             └────────────────────────┘
```

### 1.3. Простой тест

```csharp
// Что напечатает?
int a = 10;
int b = a;
b = 20;
Console.WriteLine(a);   // ?

// Ответ: 10
// b — копия a. Изменение b не affects a.
// Value type semantics.

// Vs:
class Box { public int Value; }

Box x = new() { Value = 10 };
Box y = x;
y.Value = 20;
Console.WriteLine(x.Value);   // ?

// Ответ: 20!
// y и x — два указателя на ОДИН объект. Изменение через y видно через x.
// Reference type semantics.
```

> [!info]- Если ты приходил из C/C++
> `int x = 10` — то же что C. Reference type ≈ pointer, но неявный (нет `*` или `&`). `User u = new()` ≈ `User* u = new User()` в C++. Главное отличие — нет manual `delete` (GC). Также нет stack-allocated objects (`User u;` в C++ — на stack; в C# class всегда на heap).

> [!info]- Если ты приходил из Python / JavaScript
> Концепция reference types похожа: dict / list / object — by reference. Главное отличие — value types (struct, primitives) копируются by value, не by reference. В Python `int` immutable но передаётся "по value semantics" в практике. C# value types более ярко выражены через `struct`.

> [!question]- Интервью: где живут переменные в .NET?
> **Value types** (struct, int, bool, DateTime, enum) — на **stack** если локальные variables в method, **inline** внутри containing object если поля. **Reference types** (class, string, array, delegate, interface) — объект на **heap**, reference (pointer) на stack или inline. **Edge cases**: 1) Boxed value type (`object obj = 42`) — на heap. 2) Captured locals в lambda — могут переместиться на heap (closure object). 3) Async method state machine — frame на heap. **Performance impact**: stack super fast (microseconds), heap slower (~10-100x), GC overhead. **Memory pressure**: heap allocations triggers GC.

---

## 2. Value types

### 2.1. Что такое value type

```csharp
// Built-in value types:
int, long, short, byte           // integers
float, double, decimal           // floating point
bool                             // boolean
char                             // single character
DateTime, DateTimeOffset, TimeSpan
Guid
enum                             // enumerated types

// User-defined: struct
struct Point
{
    public int X;
    public int Y;
    public double Distance() => Math.Sqrt(X * X + Y * Y);
}
```

### 2.2. Поведение copy-by-value

```csharp
struct Point { public int X; public int Y; }

Point p1 = new() { X = 1, Y = 2 };
Point p2 = p1;        // КОПИЯ! Не reference.
p2.X = 100;
Console.WriteLine(p1.X);   // 1 (не 100)
Console.WriteLine(p2.X);   // 100
```

Каждое присваивание / передача в method = **полная копия**.

### 2.3. Где они живут

```
Local в method:
- На stack текущего method frame
- Auto-deallocated при return

Поле объекта:
- Inline внутри объекта на heap
- Не отдельно

Поле struct:
- Inline внутри parent struct

Element массива:
- Inline в массиве (для int[], double[], Point[])
```

### 2.4. Когда struct оправдан

```
✅ Use struct когда:
- Маленький размер (< 16 bytes typically)
- Immutable (или rarely modified)
- Single value semantics (Point, Color, DateTime)
- Performance-critical (avoid heap allocation)

❌ Не use struct когда:
- Большой размер (> 32 bytes — copying overhead)
- Mutable с complex logic
- Inheritance нужен (struct не наследуется)
- Many properties — лучше class
```

### 2.5. Default value

```csharp
int x;            // default 0 (если field/array element)
bool b;           // default false
DateTime d;       // default DateTime.MinValue
Point p;          // все fields default

// В method body — ОБЯЗАТЕЛЬНО initialize перед use!
int y;
Console.WriteLine(y);   // ❌ Compile error: использование uninitialized variable
```

### 2.6. Nullable value types

```csharp
int x = null;       // ❌ Compile error — int не nullable
int? y = null;      // ✅ Nullable<int>

// Nullable — generic struct: Nullable<int>
public struct Nullable<T> where T : struct
{
    public bool HasValue { get; }
    public T Value { get; }
}

// Use
int? age = null;
if (age.HasValue) Console.WriteLine(age.Value);

// Null-coalescing
int realAge = age ?? 0;
```

> [!question]- Интервью: что такое struct и когда использовать?
> **Struct** — value type, container для related data. **Storage**: stack если локальный, inline в parent object если поле. **Copy semantics**: assignment / passing = full copy. **Не наследуется** (нет inheritance hierarchy). **Use cases**: маленькие immutable data (Point, Color, Money, DateTime). **Avoid когда**: 1) Big size (>32 bytes — copying overhead). 2) Mutable с complex behavior. 3) Inheritance нужен. **Best practice 2024+**: `readonly struct` для immutability, `record struct` для value-semantics records (.NET 5+). Built-in примеры: `int`, `double`, `bool`, `DateTime`, `Guid`, `KeyValuePair<K, V>`.

---

## 3. Reference types

### 3.1. Что такое reference type

```csharp
class User
{
    public string Name { get; set; } = "";
    public int Age { get; set; }
}

interface IService { }
delegate void EventHandler();
record Order(int Id, decimal Total);    // record — class под капотом

// Built-in:
string                  // immutable reference type
object                  // base class всех types
Array (int[], etc.)     // arrays
List<T>, Dictionary<K, V>, etc.
```

### 3.2. Поведение copy-by-reference

```csharp
class User { public string Name = ""; }

User u1 = new User { Name = "Alice" };
User u2 = u1;        // u2 теперь указывает на ТОТ ЖЕ объект
u2.Name = "Bob";

Console.WriteLine(u1.Name);   // "Bob" !!! (изменили через u2 — видно через u1)
Console.WriteLine(u2.Name);   // "Bob"
Console.WriteLine(ReferenceEquals(u1, u2));   // true
```

### 3.3. Stack frame (схема)

```
class User { public string Name; public int Age; }

void Method()
{
    User u = new User { Name = "Alice", Age = 30 };
}

Stack:                    Heap:
┌──────────────┐          ┌─────────────────┐
│ Method frame │          │ User instance   │
│   u = 0x100  │ ───────→ │ Name = "Alice"  │
│              │          │ Age = 30        │
└──────────────┘          └─────────────────┘
                                    │
                                    ↓
                          ┌─────────────────┐
                          │ string "Alice"  │  ← string тоже reference type!
                          └─────────────────┘
```

### 3.4. null

```csharp
User u = null;
Console.WriteLine(u.Name);   // ❌ NullReferenceException

// Null check
if (u != null)
    Console.WriteLine(u.Name);

// Null-conditional (C# 6+)
Console.WriteLine(u?.Name);   // null если u null

// Null-coalescing
Console.WriteLine(u?.Name ?? "Unknown");
```

### 3.5. Reference equality vs Value equality

```csharp
class User { public string Name = ""; }

User u1 = new() { Name = "Alice" };
User u2 = new() { Name = "Alice" };

Console.WriteLine(u1 == u2);                // false (default — reference compare)
Console.WriteLine(ReferenceEquals(u1, u2)); // false

// Override Equals для value comparison:
class User
{
    public string Name = "";
    
    public override bool Equals(object? obj) =>
        obj is User other && Name == other.Name;
    
    public override int GetHashCode() => Name.GetHashCode();
}

// Or use record (auto-generates Equals/GetHashCode):
record User(string Name);

User u1 = new("Alice");
User u2 = new("Alice");
Console.WriteLine(u1 == u2);   // true! (record value equality)
```

> [!question]- Интервью: чем отличается `==` для class и для struct?
> **Class** (reference type): `==` по default = **reference equality** (та ли это самая instance в памяти). Можно override через `==` operator + `Equals`. **Record class** — auto-generates value equality. **Struct** (value type): `==` НЕ доступен по default (compile error). Нужно implement самому. **`record struct`** (.NET 5+) — auto-generates value equality. **`Equals` method** — value comparison по default для records, нужно override для regular classes/structs. **Best practice**: для domain entities — records (auto value equality), для regular classes — explicit override Equals + GetHashCode + `==` operator (если нужно).

---

## 4. Passing — by value vs by reference

### 4.1. Default — passing by value (но nuance)

```csharp
// Для value types — копия value
void Increment(int x) { x = x + 1; }

int n = 5;
Increment(n);
Console.WriteLine(n);   // 5 (не 6!)

// Для reference types — копия reference (но указывает на тот же объект)
class Box { public int Value; }

void IncrementBox(Box b) { b.Value = b.Value + 1; }

Box box = new() { Value = 5 };
IncrementBox(box);
Console.WriteLine(box.Value);   // 6 (изменился!)
```

⚠️ Важно: для reference types **копируется reference**, но он указывает на **тот же объект**. Изменение через копию reference видно снаружи.

### 4.2. Замена reference внутри method — НЕ работает

```csharp
class Box { public int Value; }

void Replace(Box b)
{
    b = new Box { Value = 999 };   // меняем local copy reference
}

Box box = new() { Value = 5 };
Replace(box);
Console.WriteLine(box.Value);   // 5 (не 999!)
// Local b теперь указывает на новый объект, но снаружи box остался прежним
```

### 4.3. ref keyword — pass reference

```csharp
void Increment(ref int x) { x = x + 1; }

int n = 5;
Increment(ref n);
Console.WriteLine(n);   // 6!

// Для замены reference внутри method:
void Replace(ref Box b)
{
    b = new Box { Value = 999 };
}

Box box = new() { Value = 5 };
Replace(ref box);
Console.WriteLine(box.Value);   // 999!
```

### 4.4. out keyword — output parameter

```csharp
// Return multiple values
bool TryParse(string s, out int result)
{
    if (int.TryParse(s, out result)) return true;
    result = 0;
    return false;
}

if (TryParse("42", out int n))
    Console.WriteLine(n);   // 42
```

### 4.5. in keyword — readonly reference

```csharp
// Pass by reference (avoid copy для big structs) но read-only
struct BigStruct { public long A, B, C, D, E, F, G, H; }

void Print(in BigStruct s)
{
    Console.WriteLine(s.A);
    // s.A = 0;   // ❌ Compile error — in = readonly
}

BigStruct big = new();
Print(in big);   // pass by reference, no copy, but immutable inside
```

### 4.6. Когда что использовать

```
default (без modifier):
- Value types — copy
- Reference types — copy reference (объект shared)

ref:
- Хочешь изменения внутри method видеть снаружи
- Hot path — для big structs (avoid copy)

out:
- Method возвращает ДОПОЛНИТЕЛЬНОЕ значение (TryParse pattern)
- Variable инициализируется внутри method (compiler enforces)

in:
- Big struct, нужно pass by reference но без modifications
- Performance optimization
```

> [!question]- Интервью: что такое ref / out / in?
> **`ref`** — pass argument **by reference** (не by copy). Вызывающий код видит изменения. Variable должна быть **initialized** до call. **`out`** — output parameter. Method **обязан** assign перед return. Используется для multiple return values (TryParse pattern). Variable **не нужна** initialization до call. **`in`** — pass by reference **read-only**. Для big structs — avoid copy overhead, но immutable inside. **C# 7.2+**. **Use cases**: 1) `ref` — modify input. 2) `out` — return additional value. 3) `in` — performance для big structs. **Modern alternative**: tuples `(bool ok, int value) TryParse(...)` или records — cleaner чем `out`.

---

## 5. Boxing и unboxing

### 5.1. Что это

```csharp
int x = 42;          // value type, на stack
object obj = x;      // BOXING — копия в heap, obj указывает туда
int y = (int)obj;    // UNBOXING — копия из heap обратно в stack
```

```
До boxing:
Stack:           Heap:
┌─────────┐
│ x = 42  │
└─────────┘

После obj = x:
Stack:           Heap:
┌──────────┐     ┌──────────────┐
│ x = 42   │     │ Object Header│
│ obj = ──┼────→ │ Type info    │
└──────────┘     │ value: 42    │
                 └──────────────┘
```

Стоимость boxing:
- Heap allocation (~ns)
- GC pressure (объект надо собрать потом)
- Memory overhead (24+ bytes на header даже для int)
- Cache miss potential (heap ≠ cached как stack)

### 5.2. Когда boxing случается

```csharp
// 1. Cast value type в object
int x = 42;
object obj = x;   // boxing

// 2. Cast в interface (если value type implements)
struct Point : IComparable
{
    public int X, Y;
    public int CompareTo(object? obj) => 0;
}

IComparable c = new Point();   // boxing!

// 3. Non-generic collections
ArrayList list = new();
list.Add(42);   // boxing — ArrayList stores object

// 4. String formatting
Console.WriteLine($"Value: {42}");   // boxing для int → object
// Modern .NET 6+ оптимизировал многие случаи interpolation

// 5. Calling Object methods on struct
struct S { public int X; }
S s = new() { X = 5 };
s.ToString();    // virtual call — boxing!
s.GetType();     // boxing
s.Equals(other); // boxing если override отсутствует
```

### 5.3. Когда boxing НЕ случается

```csharp
// Generic collections — typed
List<int> nums = new();
nums.Add(42);   // no boxing!

// Generic methods
T DoStuff<T>(T value) where T : struct => value;
DoStuff(42);   // no boxing

// Generic comparers
List<int>.Sort();   // no boxing для int
```

### 5.4. Boxing в hot path — performance killer

```csharp
// ❌ Million boxings
ArrayList list = new();
for (int i = 0; i < 1_000_000; i++)
    list.Add(i);   // boxing каждый раз!

// Memory: ~24 MB вместо ~4 MB. GC pressure. Slow.

// ✅ Generic
List<int> list = new();
for (int i = 0; i < 1_000_000; i++)
    list.Add(i);
```

### 5.5. Detect boxing

```bash
# Через ILSpy / dotPeek
# Look for: box / unbox.any IL instructions

# В runtime — BenchmarkDotNet с MemoryDiagnoser
[MemoryDiagnoser]
public class BoxingBench
{
    [Benchmark]
    public object Box() => 42;   // boxing
}
// Output: Allocated: 24 B
```

> [!question]- Интервью: что такое boxing?
> Преобразование **value type → reference type** (object). Создаёт **копию value на heap** с object header (~24 bytes overhead) + сам value. **Когда происходит**: 1) Cast в `object` (`object obj = 42`). 2) Cast в interface если struct implements. 3) Non-generic collections (ArrayList). 4) Calling object methods (ToString, GetType) на struct без override. **Performance impact**: heap allocation, GC pressure, cache miss. **Avoid**: generic collections (`List<int>` не `List<object>`), generic methods, `IComparable<T>` instead of `IComparable`. **Modern C#**: `dynamic`, string interpolation в .NET 6+ — many boxings оптимизированы. **Detection**: BenchmarkDotNet `MemoryDiagnoser`, ILSpy для IL inspection.

---

## 6. Strings — special

### 6.1. String — reference type, immutable

```csharp
string s = "Hello";
// s — reference type (pointer на heap)
// String IMMUTABLE — раз создан, не меняется

string s2 = s + " World";   // создаёт NEW string, s не меняется

// Operations возвращают NEW strings:
s.ToUpper()        // new string
s.Replace("a", "b") // new string
s.Substring(1, 3)   // new string
```

### 6.2. String interning

```csharp
string a = "Hello";
string b = "Hello";

Console.WriteLine(ReferenceEquals(a, b));   // true!
// Компилятор и runtime используют string pool — идентичные literals share
```

```csharp
// Но dynamic strings — нет
string c = "Hel" + "lo";   // compile-time constant — interned
string d = string.Concat("Hel", "lo");   // runtime — new string

Console.WriteLine(ReferenceEquals(a, c));   // true (compile constant)
Console.WriteLine(ReferenceEquals(a, d));   // false (runtime)
```

### 6.3. == для strings — value comparison

```csharp
string a = "Hello";
string b = "Hel" + "lo";

Console.WriteLine(a == b);   // true (value equality, не reference)

// String overrides == operator для value comparison
// Equals() — то же самое
```

### 6.4. StringBuilder — mutable

```csharp
// ❌ Many string concats — quadratic memory
string result = "";
for (int i = 0; i < 1000; i++)
    result += i.ToString();
// O(n²) — каждое + создаёт new string + копирует все предыдущие

// ✅ StringBuilder — mutable buffer
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++)
    sb.Append(i);
string result = sb.ToString();
// O(n) — append в buffer, один раз allocates result
```

### 6.5. Когда StringBuilder

```
✅ Many concatenations в loop (>10)
✅ Building dynamic strings programmatically
✅ Performance-critical hot paths

❌ Несколько concatenations — string + string OK
❌ String interpolation $"{a} {b} {c}" — обычно fine
❌ String.Format — для readability приемлемо
```

> [!question]- Интервью: почему string immutable?
> **Дизайн-решение** для безопасности и performance: 1) **Thread-safe** — нельзя modify, можно безопасно share между threads. 2) **String interning** — identical literals share memory, expensive lookup tables работают. 3) **Hash code stable** — можно safely cache GetHashCode (для Dictionary keys). 4) **Security** — нельзя tamper с string после validation. **Trade-off**: many concatenations создают много intermediate strings. **Alternative**: `StringBuilder` для mutable scenarios. **Performance в .NET**: string operations heavily optimized, не бойся small concatenations. Только в hot loops или больших volumes — StringBuilder.

---

## 7. Common pitfalls

### 7.1. Big struct — copy overhead

```csharp
struct BigStruct { public long A, B, C, D, E, F, G, H; }   // 64 bytes

void Process(BigStruct s) { ... }   // ❌ Каждый call копирует 64 bytes

BigStruct big = new();
for (int i = 0; i < 1_000_000; i++)
    Process(big);   // ❌ 64 MB copying!
```

**Фикс**: pass by reference: `void Process(in BigStruct s)`. Или используй class.

### 7.2. Mutable struct в collection

```csharp
struct Point { public int X; public int Y; }

List<Point> points = new() { new() { X = 1, Y = 2 } };
points[0].X = 100;   // ❌ Compile error — нельзя modify struct в collection!

// list[0] возвращает копию, не reference
```

**Фикс**: use class, или re-assign:

```csharp
var p = points[0];
p.X = 100;
points[0] = p;
```

Или используй `record struct` / `readonly struct` — explicit immutability.

### 7.3. NullReferenceException

```csharp
User user = GetUser(id);
Console.WriteLine(user.Name);   // ❌ Crash if user null

// Modern C# (.NET 6+) — nullable reference types:
public User? GetUser(int id) { ... }   // ? = может быть null

User? user = GetUser(id);
Console.WriteLine(user.Name);   // ⚠️ Compiler warning
Console.WriteLine(user?.Name);  // ✅ Null-safe
Console.WriteLine(user!.Name);  // explicit assert non-null (use carefully)
```

### 7.4. Default struct values

```csharp
public struct Money
{
    public decimal Amount;
    public string Currency;
}

Money m = default;   // Amount = 0, Currency = null!
Console.WriteLine(m.Currency.Length);   // ❌ NullReferenceException
```

**Фикс**: `readonly struct` с required fields, или class.

### 7.5. Concurrent modification

```csharp
List<int> nums = new() { 1, 2, 3, 4, 5 };
foreach (var n in nums)
{
    if (n % 2 == 0) nums.Remove(n);   // ❌ InvalidOperationException
}
```

**Фикс**: copy / new list или filter сначала:

```csharp
nums = nums.Where(n => n % 2 != 0).ToList();
```

### 7.6. String + (плюс) в loop

```csharp
string result = "";
for (int i = 0; i < 1000; i++)
    result += $"Item {i}\n";   // ❌ O(n²) memory
```

**Фикс**: StringBuilder.

### 7.7. Boxing через interface

```csharp
public interface IShape { double Area(); }
public struct Circle : IShape { public double R; public double Area() => Math.PI * R * R; }

void Print(IShape s) { ... }

Circle c = new() { R = 5 };
Print(c);   // ❌ Boxing! Cast struct → IShape
```

**Фикс**: generic constraint:

```csharp
void Print<T>(T s) where T : IShape { ... }   // no boxing

Circle c = new() { R = 5 };
Print(c);   // no boxing
```

### 7.8. Comparing strings case-sensitively unintentionally

```csharp
string a = "hello";
string b = "HELLO";
Console.WriteLine(a == b);   // false

// Для case-insensitive:
Console.WriteLine(string.Equals(a, b, StringComparison.OrdinalIgnoreCase));
```

### 7.9. Default capacity для Dictionary / List

```csharp
// ❌ Дефолтная capacity 4, рост 2x → много resizes если знаешь размер
var dict = new Dictionary<int, string>();
for (int i = 0; i < 100_000; i++)
    dict[i] = $"item{i}";

// ✅ Pre-size
var dict = new Dictionary<int, string>(capacity: 100_000);
```

### 7.10. Disposable забыл

```csharp
var conn = new SqlConnection(connStr);
conn.Open();
// ... work ...
// ❌ Connection лежит на heap — GC не закроет рукотворно
```

**Фикс**: `using var conn = ...;` — auto-Dispose.

> [!question]- Интервью: топ-3 memory mistakes?
> 1) **Boxing в hot path** — value types в `object` / `ArrayList` / non-generic interfaces. Million boxings = 24 MB+ overhead. Fix: generics. 2) **String concat в loop** — O(n²) memory + many allocations. Fix: StringBuilder. 3) **Big struct copying** — 64+ bytes struct passed by value много раз. Fix: `in` parameter или class. **Bonus**: NullReferenceException — enable nullable reference types (`<Nullable>enable</Nullable>` в csproj) — compiler warns. **Bonus 2**: Dictionary/List без pre-sizing — много resizes когда знаешь финальный размер.

---

## 8. Cheat sheet

```csharp
// === Value types — на stack или inline ===
int x = 42;                    // 4 bytes на stack
DateTime now = DateTime.UtcNow; // 8 bytes на stack
struct Point { int X, Y; }      // 8 bytes на stack

// === Reference types — pointer на stack, объект на heap ===
class User { ... }
User u = new();                 // 8 bytes pointer на stack, объект на heap
string s = "Hello";             // 8 bytes pointer, string на heap
int[] arr = new int[10];        // 8 bytes pointer, array на heap

// === Copy semantics ===
int a = 5; int b = a; b = 10;        // a still 5 (value copy)
User u1 = new(); User u2 = u1;        // u2 → same object as u1
u2.Name = "X"; // u1.Name == "X"

// === Pass by reference ===
void Modify(ref int x) { x++; }
int n = 5; Modify(ref n);              // n is now 6

void TryParse(string s, out int result)
{
    result = int.Parse(s);
}

void Print(in BigStruct s) { /* read-only */ }

// === Boxing avoidance ===
List<int> nums = new();          // ✅ generic
Dictionary<string, int> map = new();
T GetValue<T>() => default!;

// Boxes:
ArrayList list = new();          // ❌ legacy
list.Add(42);                    // boxing

// === Strings ===
string s = "Hello";              // immutable
string s2 = s + " World";        // new string

var sb = new StringBuilder();
for (int i = 0; i < 100; i++) sb.Append(i);
string result = sb.ToString();

// === Disposable ===
using var conn = new SqlConnection(...);
using var stream = File.Open(...);
using var doc = JsonDocument.Parse(...);

// === Null safety ===
User? user = GetUser(id);
var name = user?.Name ?? "Unknown";
if (user is not null) { /* user definitely non-null here */ }

// === Records — value equality для reference types ===
record User(int Id, string Name);
User u1 = new(1, "Alice");
User u2 = new(1, "Alice");
u1 == u2;   // true (value equality)

// === Modern struct features ===
readonly struct Money(decimal amount, string currency);
record struct Point(int X, int Y);
```

---

## 9. Practice exercises

### 9.1. Boxing detection

```csharp
public void Method1(object obj) { }
public void Method2<T>(T obj) { }

int x = 42;
Method1(x);   // boxing? yes/no?
Method2(x);   // boxing? yes/no?

object o = x;       // boxing? yes/no?
ArrayList list = new();
list.Add(x);        // boxing? yes/no?

List<int> generic = new();
generic.Add(x);     // boxing? yes/no?
```

Ответы: yes / no / yes / yes / no.

### 9.2. Reference vs Value comparison

```csharp
class User { public string Name = ""; }
record UserRecord(string Name);
struct UserStruct { public string Name; }

User u1 = new() { Name = "Alice" };
User u2 = new() { Name = "Alice" };
u1 == u2;   // ?

UserRecord r1 = new("Alice");
UserRecord r2 = new("Alice");
r1 == r2;   // ?

UserStruct s1 = new() { Name = "Alice" };
UserStruct s2 = new() { Name = "Alice" };
s1 == s2;   // ?
```

Ответы:
- `u1 == u2` → false (reference equality для class)
- `r1 == r2` → true (auto-generated value equality для record)
- `s1 == s2` → compile error (struct без override == operator)

### 9.3. Memory profiling

Через BenchmarkDotNet с MemoryDiagnoser:

```csharp
[MemoryDiagnoser]
public class Bench
{
    [Benchmark]
    public string Concat() {
        string s = "";
        for (int i = 0; i < 100; i++) s += $"item{i}";
        return s;
    }
    
    [Benchmark]
    public string Builder() {
        var sb = new StringBuilder();
        for (int i = 0; i < 100; i++) sb.Append($"item{i}");
        return sb.ToString();
    }
}
```

Сравни Allocated bytes — концат много больше из-за intermediate strings.

---

## 10. Что читать дальше

1. **`Runtime/Junior/runtime-basics.md`** — CLR, JIT overview
2. **`Runtime/gc-memory.md`** — GC generations deep
3. **`Runtime/span-layout.md`** — `Span<T>`/`Memory<T>`
4. **`CSharp/Senior/types-and-memory.md`** — types deep
5. **`CSharp/Junior/oop`** — classes, struct, inheritance basics

---

## 11. См. также

- [[runtime-basics|Runtime/Junior/runtime-basics]] — CLR overview
- [[gc-memory|Runtime/gc-memory]] — GC deep
- [[span-layout|Runtime/span-layout]] — Span/Memory
- [[types-and-memory|CSharp/Senior/types-and-memory]] — types deep
- [[oop|CSharp/Junior/oop]] — class vs struct
- [[performance-fundamentals|Performance/Junior/performance-fundamentals]] — performance basics

---

## 12. Reading list

- **Microsoft Docs — Value types and reference types** — learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/value-types
- **Microsoft Docs — Boxing and Unboxing** — learn.microsoft.com/dotnet/csharp/programming-guide/types/boxing-and-unboxing
- **"Pro .NET Memory Management" — Konrad Kokosa** (deep)
- **"CLR via C#" — Jeffrey Richter** (classic, deep)
- **Stephen Toub blog** — devblogs.microsoft.com/dotnet (performance posts)
- **dotnet/runtime** — github.com/dotnet/runtime (source)
- **BenchmarkDotNet docs** — benchmarkdotnet.org
