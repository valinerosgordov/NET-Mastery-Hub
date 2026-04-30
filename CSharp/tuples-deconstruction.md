---
tags: [csharp, tuples, deconstruction, value-tuple, junior, middle]
level: Junior
date: 2026-04-30
---

# Tuples и Deconstruction — кортежи и деконструкция

> **Modern C# everyday feature**: ValueTuple, named tuples, deconstruction, multiple return values. Появилось в C# 7.0 и сильно изменило стиль кода. Closes пробел "когда tuple, когда class/record".

---

## Что это, зачем и когда

### Что такое tuple

**Группа значений упакованных вместе** без создания отдельного класса.

```csharp
// Без tuples — создавай class для каждой группы
class Point { public int X; public int Y; }
var p = new Point { X = 1, Y = 2 };

// С tuples — на месте
var p = (X: 1, Y: 2);
Console.WriteLine($"{p.X}, {p.Y}");  // "1, 2"
```

### Зачем

| Без tuples | С tuples |
|-----------|----------|
| `class` для multiple returns | `(T1, T2)` return type |
| `out` parameters везде | `(value, success)` returns |
| `Tuple<T1, T2>` (старый) — `Item1`, `Item2` — нечитабельно | Named tuples — `(int Count, decimal Total)` |
| Helper class для group of values | Inline tuple |

### Эволюция

- **C# 7.0 (2017)** — ValueTuple, deconstruction
- **C# 7.1+** — tuple equality
- **C# 12 (2023)** — alias `using` для tuple types

```csharp
// C# 12 — alias
using Coord = (int X, int Y);

Coord MakeCoord() => (1, 2);
```

---

## 1. Базовые tuples

### Создание

```csharp
// Named tuple
var person = (Name: "John", Age: 30);
person.Name;     // "John"
person.Age;      // 30

// Без имён — Item1, Item2, ...
var pair = (1, "hello");
pair.Item1;      // 1
pair.Item2;      // "hello"

// Mixed
var data = (Id: 1, "value", Year: 2024);
data.Id;         // 1
data.Item2;      // "value"  (нет имени)
data.Year;       // 2024

// Type explicit
(int Count, decimal Total) result = (5, 100.50m);
```

### Tuples — это struct

```csharp
// ValueTuple<T1, T2> — это struct (value type, stack-allocated)
var t = (1, 2);
// эквивалентно: new ValueTuple<int, int>(1, 2)

// Старый Tuple<T1, T2> — class (reference type, heap)
// Использовать ValueTuple, не Tuple. Tuple — legacy.
```

### До 8 элементов

```csharp
var sevenElement = (1, 2, 3, 4, 5, 6, 7);

// > 7 — поддерживается через nesting (ValueTuple<..., TRest>)
var eight = (1, 2, 3, 4, 5, 6, 7, 8);
// 8th item — Rest.Item1
```

---

## 2. Multiple return values

### Без tuples — out parameters

```csharp
// Старый стиль
public bool TryParseCoord(string input, out int x, out int y)
{
    var parts = input.Split(',');
    x = int.Parse(parts[0]);
    y = int.Parse(parts[1]);
    return true;
}

// Использование
if (TryParseCoord("1,2", out int x, out int y))
{
    Console.WriteLine($"{x}, {y}");
}
```

### С tuples

```csharp
// Modern
public (int X, int Y) ParseCoord(string input)
{
    var parts = input.Split(',');
    return (int.Parse(parts[0]), int.Parse(parts[1]));
}

// Использование
var (x, y) = ParseCoord("1,2");
Console.WriteLine($"{x}, {y}");

// Или через имена
var coord = ParseCoord("1,2");
Console.WriteLine($"{coord.X}, {coord.Y}");
```

### Pattern: TryGet с tuple

```csharp
// Tuple-based "TryGet"
public (bool Success, User? User) TryGetUser(int id)
{
    var user = _users.FirstOrDefault(u => u.Id == id);
    return (user != null, user);
}

// Использование
var (success, user) = TryGetUser(1);
if (success)
{
    Console.WriteLine(user.Name);
}
```

> [!info] tuple vs out
> **Tuple** — readable, immutable, можно ignore parts.
> **Out** — был стандартом, теперь меньше используется.
> **Result\<T, E\>** или Maybe для domain logic — best.

---

## 3. Deconstruction

### Декомпозиция tuple

```csharp
var person = (Name: "John", Age: 30);

// Все sразу
var (name, age) = person;
Console.WriteLine($"{name}, {age}");  // "John, 30"

// Discard через _
var (name, _) = person;  // ignore age

// Существующие variables
string n;
int a;
(n, a) = person;
```

### Deconstruction для классов

```csharp
public class Person
{
    public string Name { get; }
    public int Age { get; }

    public Person(string name, int age) { Name = name; Age = age; }

    // Deconstruct method — может быть extension!
    public void Deconstruct(out string name, out int age)
    {
        name = Name;
        age = Age;
    }
}

// Использование
var person = new Person("John", 30);
var (name, age) = person;  // вызывает Deconstruct
```

### Records — auto deconstruction

```csharp
public record Person(string Name, int Age);

var p = new Person("John", 30);
var (name, age) = p;  // auto-generated Deconstruct!
```

### Deconstruction в `foreach`

```csharp
Dictionary<string, int> ages = new() { ["Alice"] = 30, ["Bob"] = 25 };

foreach (var (key, value) in ages)
{
    Console.WriteLine($"{key}: {value}");
}

// vs старый style
foreach (var kvp in ages)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}
```

### Pattern matching + deconstruction

```csharp
var point = (X: 1, Y: 2);

string description = point switch
{
    (0, 0) => "origin",
    (var x, 0) => $"on X axis at {x}",
    (0, var y) => $"on Y axis at {y}",
    (var x, var y) => $"at ({x}, {y})"
};
```

---

## 4. Tuples в LINQ

### GroupBy / Project с tuples

```csharp
var orders = new[] {
    new { CustomerId = 1, Amount = 100m, Year = 2024 },
    new { CustomerId = 1, Amount = 50m, Year = 2024 },
    new { CustomerId = 2, Amount = 200m, Year = 2023 }
};

// Group by composite key
var byCustomerYear = orders.GroupBy(o => (o.CustomerId, o.Year))
    .Select(g => new
    {
        Key = g.Key,
        Total = g.Sum(o => o.Amount)
    });

foreach (var group in byCustomerYear)
{
    Console.WriteLine($"Customer {group.Key.CustomerId} in {group.Key.Year}: {group.Total}");
}
```

### Возврат multiple values из LINQ

```csharp
var stats = numbers.Aggregate(
    seed: (Min: int.MaxValue, Max: int.MinValue, Sum: 0),
    func: (acc, n) => (Math.Min(acc.Min, n), Math.Max(acc.Max, n), acc.Sum + n)
);

Console.WriteLine($"Min={stats.Min}, Max={stats.Max}, Sum={stats.Sum}");
```

---

## 5. Tuples vs альтернативы

### tuple vs class

```csharp
// Class — formal, reusable
public class Coordinate
{
    public int X { get; }
    public int Y { get; }

    public Coordinate(int x, int y) { X = x; Y = y; }

    public double Distance(Coordinate other) =>
        Math.Sqrt(Math.Pow(X - other.X, 2) + Math.Pow(Y - other.Y, 2));
}

// Tuple — quick, local
(int X, int Y) coord = (1, 2);
```

**Когда class:**
- Domain concept
- Имеет behavior (методы)
- Reused across проект
- Validation в constructor
- Identity matters

**Когда tuple:**
- Quick local grouping
- Multiple returns
- Internal helper
- Throwaway

### tuple vs record

```csharp
// Tuple — anonymous structure
var person = (Name: "John", Age: 30);

// Record — named type
public record Person(string Name, int Age);
var person = new Person("John", 30);
```

**Когда tuple:**
- Local scope
- Не reused
- Quick prototyping

**Когда record:**
- Public API
- Across files / methods
- Имеет behavior
- Documentation needed

### tuple vs anonymous type

```csharp
// Anonymous type — обычно для LINQ projection
var anon = new { Name = "John", Age = 30 };

// Tuple — можно return из методов!
var tuple = (Name: "John", Age: 30);

// Anonymous types нельзя return как тип
public ??? GetData()
{
    return new { Name = "John", Age = 30 };  // ❌ no type to declare
}

// ✅ Tuple
public (string Name, int Age) GetData()
{
    return ("John", 30);
}
```

---

## 6. Tuple equality

```csharp
var a = (1, 2);
var b = (1, 2);

a == b;       // true ✅ (value equality, since C# 7.3)
a.Equals(b);  // true

// С different names — same structure
var x = (X: 1, Y: 2);
var y = (A: 1, B: 2);

x == y;  // true! Names ignored, only types and values

// С different types
var i = (1, 2);
var l = (1L, 2L);
i == l;  // ❌ Compile error — different types
```

### В collections

```csharp
var seen = new HashSet<(int, int)>();
seen.Add((1, 2));
seen.Contains((1, 2));  // true ✅

var byCoord = new Dictionary<(int X, int Y), string>();
byCoord[(1, 2)] = "origin";
```

---

## 7. Conversion tuples

### Implicit conversions

```csharp
// (int, int) ↔ (long, long)
(int, int) ints = (1, 2);
(long, long) longs = ints;  // ✅ implicit widening

// Different names
(int X, int Y) named = (1, 2);
(int A, int B) renamed = named;  // ✅ names не matter
```

### Tuple → struct (manual)

```csharp
public struct Point
{
    public int X, Y;

    public static implicit operator Point((int X, int Y) tuple) =>
        new() { X = tuple.X, Y = tuple.Y };
}

Point p = (1, 2);  // Auto-conversion!
```

---

## 8. Performance

### ValueTuple — stack-allocated

```csharp
// Создание tuple — no heap allocation
var t = (1, 2);  // stack
// vs
var pair = new Tuple<int, int>(1, 2);  // heap allocation!
```

### Pass tuple через method boundaries

```csharp
public (int X, int Y) Process((int X, int Y) input)
{
    return (input.X * 2, input.Y * 2);
}

// ValueTuple copy each call — для больших tuples (>4 fields) → expensive
```

### Tuple vs out — performance

```csharp
// Tuple — extra struct copy
public (bool, int) Try() => (true, 42);

// Out — direct write
public bool Try(out int value) { value = 42; return true; }

// Difference микроскопическая для small tuples. Не оптимизируй преждевременно.
```

---

## 9. Common patterns

### Pattern 1: Multi-result methods

```csharp
public (decimal Total, decimal Tax, decimal Discount) CalculateOrder(Order o)
{
    var subtotal = o.Items.Sum(i => i.Price);
    var tax = subtotal * 0.08m;
    var discount = subtotal > 1000 ? subtotal * 0.1m : 0;
    return (subtotal + tax - discount, tax, discount);
}

var (total, tax, discount) = CalculateOrder(order);
```

### Pattern 2: Swap variables

```csharp
int a = 1, b = 2;
(a, b) = (b, a);  // swap! a=2, b=1
```

### Pattern 3: Multiple init

```csharp
var (count, sum, max) = (0, 0, int.MinValue);
```

### Pattern 4: Composite keys

```csharp
var groupedBy = items.GroupBy(i => (i.Year, i.Month));

var byKey = new Dictionary<(int Year, int Month), List<Item>>();
byKey[(2024, 1)] = new();
```

### Pattern 5: Return enum + value

```csharp
public enum Result { Success, NotFound, Conflict }

public (Result Status, User? User) GetUser(int id)
{
    if (id < 0) return (Result.NotFound, null);
    var user = _users.FirstOrDefault(u => u.Id == id);
    return (user is null ? Result.NotFound : Result.Success, user);
}

// Usage
var (status, user) = GetUser(1);
switch (status)
{
    case Result.Success: Console.WriteLine(user!.Name); break;
    case Result.NotFound: Console.WriteLine("Not found"); break;
}
```

### Pattern 6: Fluent error handling

```csharp
public (bool Success, string? Error) Validate(User u)
{
    if (string.IsNullOrEmpty(u.Name)) return (false, "Name empty");
    if (u.Age < 0) return (false, "Invalid age");
    return (true, null);
}

var (ok, err) = Validate(user);
if (!ok) Console.WriteLine(err);
```

### Pattern 7: Min/max one-liner

```csharp
var (min, max) = numbers.Aggregate(
    (Min: int.MaxValue, Max: int.MinValue),
    (acc, n) => (Math.Min(acc.Min, n), Math.Max(acc.Max, n))
);
```

---

## 10. Common Pitfalls

### 1. Slow для >4 elements

```csharp
// Tuple копируется по value — больше fields, больше copy
var huge = (a, b, c, d, e, f, g);  // copied every method boundary

// ✅ Используй record / class для больших structures
public record Stats(int A, int B, int C, int D, int E, int F, int G);
```

### 2. Tuple в public API

```csharp
// ❌ Public API с tuple — sealed contract
public (int Total, int Count) GetStats()  // что значит "Total"?
{
    return (100, 5);
}

// ✅ Record — self-documenting, evolution-friendly
public record StatsResult(int Total, int Count);
public StatsResult GetStats() => new(100, 5);
```

Tuple OK для **internal** APIs. Для **public** — record.

### 3. Nameless tuples — нечитабельно

```csharp
// ❌ Что Item1, Item2?
var t = (5, "John");
Console.WriteLine(t.Item1);  // Хм...

// ✅ Named
var t = (Age: 5, Name: "John");
Console.WriteLine(t.Age);
```

### 4. Forget that tuple — value type

```csharp
var t = (X: 1, Y: 2);
ModifyTuple(t);  // copy!

void ModifyTuple((int X, int Y) tuple)
{
    tuple.X = 100;  // меняем copy!
}
// t.X is still 1
```

`ref` параметр если хочешь modify:

```csharp
void ModifyTuple(ref (int X, int Y) tuple)
{
    tuple.X = 100;
}
ModifyTuple(ref t);  // now t.X = 100
```

### 5. Discard misuse

```csharp
var (_, _, value) = (1, 2, 3);  // OK — ignore первые два

// ❌ Bad — Item3 имеет смысл, читать tricky
var (_, _, _, _, _, _, x) = (1, 2, 3, 4, 5, 6, 7);
```

### 6. Tuple equality имена

```csharp
(int X, int Y) a = (1, 2);
(int A, int B) b = (1, 2);

a == b;  // true — names ignored!
// Может быть surprising — A=B != X=Y conceptually
```

### 7. Tuple в LINQ + EF Core

```csharp
// EF Core может не поддерживать tuples в queries
var query = db.Users.Select(u => (u.Id, u.Name));
// ⚠️ Может throw — не translatable to SQL

// ✅ Anonymous type
var query = db.Users.Select(u => new { u.Id, u.Name });
```

### 8. Mutating tuple field

```csharp
var t = (Items: new List<int>(), Count: 0);
t.Items.Add(1);  // ✅ Mutating reference type's contents
t.Count++;       // ⚠️ Mutating tuple field (need ref or var)

// Обычно tuples — local scoping, mutation OK
// Но для API — лучше return new tuple
```

---

## 11. Best Practices

- **Records для public API** — не tuples
- **Tuples для internal / local** — quick wins
- **Named tuples всегда** — `(int X, int Y)`, не `(int, int)`
- **2-4 elements max** в tuple — больше → record/class
- **`var (x, y) = method()`** — очень читабельно для multi-returns
- **Discard `_`** для unused parts
- **Anonymous type в LINQ projections** — но для return types tuple
- **Decompose в `foreach`** для Dictionary
- **Pattern match с deconstruction** — modern C# style
- **`using` alias (C# 12)** для repeated tuple types

---

## 12. Cheat sheet

| Сценарий | Solution |
|----------|----------|
| Multiple return values | `(T1, T2) Method()` |
| Named multi-return | `(int Count, decimal Total) ...` |
| Swap variables | `(a, b) = (b, a)` |
| Foreach Dictionary | `foreach (var (k, v) in dict)` |
| Composite key | `Dictionary<(int, int), T>` |
| Pattern match | `case (var x, 0):` |
| Discard | `var (_, age) = person` |
| Local quick group | `var pair = (a, b)` |
| Public API | record, не tuple |
| EF Core queries | anonymous type, не tuple |
| Method returns >4 values | record, не tuple |

---

## 13. Decision tree

```
Нужно вернуть несколько values?
│
├── 2-3 values, internal use
│   → Tuple: (int X, int Y)
│
├── 2-4 values, public API
│   → record
│
├── >4 values
│   → record / class
│
├── Optional return (might fail)
│   → Result<T, E> или (bool Success, T? Value)
│
└── Domain concept
    → record / class

Группа values на месте?
│
├── Internal scope
│   → Tuple
│
├── Иначе
│   → Anonymous type (LINQ) или record

Декомпозиция?
│
├── Tuple → var (x, y) = tuple
├── Class → реализуй Deconstruct method
├── Record → auto Deconstruct
└── Dictionary → foreach (var (k, v) in dict)
```

---

## См. также

- [[csharp-basics|C# Basics]] — variables intro
- [[modern-features|Modern Features]] — tuples evolution
- [[functional-csharp|Functional C#]] — records, pattern matching
- [[error-handling|Error Handling]] — Result\<T, E\> alternative
- [[collections-linq|Collections и LINQ]] — anonymous types

## Reading list

- **Microsoft Docs — Tuple types** — learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/value-tuples
- **Microsoft Docs — Deconstruction** — learn.microsoft.com/dotnet/csharp/fundamentals/functional/deconstruct
- **Mads Torgersen — Tuples in C# 7** — devblogs.microsoft.com/dotnet
- **Stephen Toub — ValueTuple performance** — devblogs.microsoft.com/dotnet
- **C# in Depth** — Jon Skeet
