---
tags: [csharp, indexers, operators, overloading, conversion, middle]
level: Middle
date: 2026-04-30
---

# Indexers и Operator Overloading

> **Кастомные `[]` индексаторы и перегрузка операторов**. Делает твои типы выглядящими как built-in: `matrix[1,2]`, `point + point`, `(int)money`. Closes пробел "знаю что есть, но не понимаю когда использовать и как избежать абуза".

---

## Что это, зачем и когда

### Что это

Возможность сделать **custom типы себя как primitive** — index access, арифметика, conversions.

```csharp
// Без перегрузки — verbose API
var sum = vector1.Add(vector2);
var item = list.GetItem(5);

// С перегрузкой — natural syntax
var sum = vector1 + vector2;
var item = list[5];
```

### Когда применять

✅ **Хорошо для:**
- **Math types** — Vector, Matrix, Point, Complex, Money
- **Containers** — Custom collections (List-like с indexers)
- **Domain types** — Currency, Distance, Angle
- **Smart strings** — case-sensitive vs case-insensitive

❌ **Плохо для:**
- "Просто потому что можно"
- Когда semantics non-intuitive (`User + User = ?`)
- Когда `Add()` метод понятнее `+`
- Side effects в operators

---

## 1. Indexers — `[index]`

### Базовый indexer

```csharp
public class WeekSchedule
{
    private readonly Dictionary<DayOfWeek, string> _activities = new();

    // Indexer — `this` syntax
    public string this[DayOfWeek day]
    {
        get => _activities.TryGetValue(day, out var v) ? v : "Free";
        set => _activities[day] = value;
    }
}

// Использование
var schedule = new WeekSchedule();
schedule[DayOfWeek.Monday] = "Gym";
Console.WriteLine(schedule[DayOfWeek.Monday]);  // "Gym"
Console.WriteLine(schedule[DayOfWeek.Sunday]);  // "Free"
```

### Несколько типов параметров (overloads)

```csharp
public class UserCollection
{
    private readonly List<User> _users = new();

    // По индексу
    public User this[int index] => _users[index];

    // По email
    public User this[string email] =>
        _users.FirstOrDefault(u => u.Email == email);

    // Несколько параметров — multidimensional
    public User this[string firstName, string lastName] =>
        _users.FirstOrDefault(u => u.FirstName == firstName && u.LastName == lastName);
}

users[0];                       // by index
users["john@example.com"];      // by email
users["John", "Doe"];            // by name
```

### Read-only indexer

```csharp
public class ImmutableList<T>
{
    private readonly T[] _items;

    public ImmutableList(T[] items) => _items = items;

    public T this[int index] => _items[index];  // только get!
}
```

### Multi-dimensional

```csharp
public class Matrix
{
    private readonly double[,] _data;

    public Matrix(int rows, int cols) => _data = new double[rows, cols];

    public double this[int row, int col]
    {
        get => _data[row, col];
        set => _data[row, col] = value;
    }
}

var m = new Matrix(3, 3);
m[0, 0] = 1.5;
m[1, 2] = m[0, 0] + 2;
```

### Index from end (^) и Range (..)

C# 8+ поддерживает `Index` и `Range`:

```csharp
public class MyList
{
    private readonly List<int> _items = new();

    // Index
    public int this[Index index] => _items[index];

    // Range
    public IEnumerable<int> this[Range range]
    {
        get
        {
            var (start, length) = range.GetOffsetAndLength(_items.Count);
            return _items.Skip(start).Take(length);
        }
    }
}

var list = new MyList();
var last = list[^1];           // Index from end
var slice = list[1..5];         // Range
```

> [!info] String, Array, List уже поддерживают
> Большинство встроенных коллекций уже имеют indexers с Index/Range.

### Const-key dictionary pattern

```csharp
public class Config
{
    private readonly Dictionary<string, object> _values = new();

    public T this<T>(string key)  // ❌ Compile error — generic indexer не поддерживается!
}

// ✅ Альтернатива — generic method
public T Get<T>(string key) => (T)_values[key];

config.Get<int>("Port");
```

> [!warning] Generic indexers нельзя
> Нужно делать через regular method.

---

## 2. Operator overloading — арифметика

### Бинарные операторы

```csharp
public readonly struct Vector
{
    public double X { get; }
    public double Y { get; }

    public Vector(double x, double y) => (X, Y) = (x, y);

    // Перегрузка +
    public static Vector operator +(Vector a, Vector b) =>
        new(a.X + b.X, a.Y + b.Y);

    // Перегрузка -
    public static Vector operator -(Vector a, Vector b) =>
        new(a.X - b.X, a.Y - b.Y);

    // Умножение на скаляр
    public static Vector operator *(Vector v, double scalar) =>
        new(v.X * scalar, v.Y * scalar);

    public static Vector operator *(double scalar, Vector v) =>
        v * scalar;  // commutative — обе стороны

    // Деление
    public static Vector operator /(Vector v, double scalar) =>
        new(v.X / scalar, v.Y / scalar);
}

// Использование
var a = new Vector(1, 2);
var b = new Vector(3, 4);

var sum = a + b;        // Vector(4, 6)
var diff = a - b;        // Vector(-2, -2)
var scaled = a * 2;      // Vector(2, 4)
var scaled2 = 2 * a;     // Vector(2, 4) — commutative
```

### Унарные операторы

```csharp
public readonly struct Vector
{
    // Унарный минус
    public static Vector operator -(Vector v) => new(-v.X, -v.Y);

    // Унарный плюс (редко имеет смысл)
    public static Vector operator +(Vector v) => v;

    // Increment / decrement
    public static Vector operator ++(Vector v) => new(v.X + 1, v.Y + 1);
    public static Vector operator --(Vector v) => new(v.X - 1, v.Y - 1);
}

var v = new Vector(1, 2);
var negated = -v;     // Vector(-1, -2)
var inc = v++;        // returns v, потом v becomes Vector(2, 3)
```

### Comparison operators

Должны быть **парами** (`<` и `>`, `<=` и `>=`):

```csharp
public class Money : IComparable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency) =>
        (Amount, Currency) = (amount, currency);

    public int CompareTo(Money? other)
    {
        if (other is null) return 1;
        if (Currency != other.Currency)
            throw new InvalidOperationException("Different currencies");
        return Amount.CompareTo(other.Amount);
    }

    public static bool operator <(Money a, Money b) => a.CompareTo(b) < 0;
    public static bool operator >(Money a, Money b) => a.CompareTo(b) > 0;
    public static bool operator <=(Money a, Money b) => a.CompareTo(b) <= 0;
    public static bool operator >=(Money a, Money b) => a.CompareTo(b) >= 0;
}
```

### Equality operators

```csharp
public class Point : IEquatable<Point>
{
    public int X { get; }
    public int Y { get; }

    public Point(int x, int y) => (X, Y) = (x, y);

    public bool Equals(Point? other) =>
        other is not null && X == other.X && Y == other.Y;

    public override bool Equals(object? obj) => Equals(obj as Point);
    public override int GetHashCode() => HashCode.Combine(X, Y);

    // == и != — пара! Нельзя одно без другого
    public static bool operator ==(Point? a, Point? b)
    {
        if (a is null) return b is null;
        return a.Equals(b);
    }

    public static bool operator !=(Point? a, Point? b) => !(a == b);
}
```

См. [[equality-comparison|Equality и Comparison]].

> [!info] Records — auto-generated
> ```csharp
> public record Point(int X, int Y);
> // == != Equals GetHashCode — все auto!
> ```

---

## 3. Что МОЖНО перегружать

| Operator | Notes |
|----------|-------|
| `+`, `-`, `*`, `/`, `%` | Бинарные арифметические |
| `+`, `-` | Унарные (отдельно от бинарных) |
| `++`, `--` | Increment / decrement |
| `==`, `!=` | Equality (parami!) |
| `<`, `>`, `<=`, `>=` | Comparison (parami!) |
| `&`, `\|`, `^` | Bitwise / logical |
| `<<`, `>>` | Shift |
| `~`, `!` | Унарные bitwise / logical NOT |
| `true`, `false` | Special — для conditional logic |
| `(T)` | Conversion (implicit/explicit) |

## 4. Что НЕЛЬЗЯ перегружать

```csharp
=          // assignment
&&  ||      // short-circuit (но & | можно — они вызываются)
??          // null coalescing
?:          // ternary
.           // member access
->          // pointer
[]          // используй indexer (this[])
new
typeof, sizeof, is, as, checked, unchecked
```

---

## 5. Implicit и Explicit conversions

### Implicit — auto-conversion

```csharp
public readonly struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency) =>
        (Amount, Currency) = (amount, currency);

    // decimal → Money (implicit USD)
    public static implicit operator Money(decimal amount) =>
        new(amount, "USD");
}

// Использование
Money price = 19.99m;  // implicit conversion
```

> [!warning] Implicit — только когда **lossless и predictable**
> User не ожидает потерю данных или surprises.

### Explicit — требует cast

```csharp
public readonly struct Celsius
{
    public double Value { get; }
    public Celsius(double v) => Value = v;

    public static explicit operator Fahrenheit(Celsius c) =>
        new(c.Value * 9 / 5 + 32);
}

public readonly struct Fahrenheit
{
    public double Value { get; }
    public Fahrenheit(double v) => Value = v;
}

var c = new Celsius(100);
var f = (Fahrenheit)c;  // explicit — нужен cast
// Fahrenheit f = c;   // ❌ Compile error — explicit нужен
```

### Conversion guidelines

| Когда implicit | Когда explicit |
|----------------|----------------|
| `int → long` | `long → int` (потеря данных) |
| `Subclass → Base` | `Base → Subclass` |
| Lossless | Lossy |
| Always succeeds | Может throw |
| Same conceptual type | Разные concepts (Celsius → Fahrenheit) |

### Multiple conversions

```csharp
public readonly struct Distance
{
    public double Meters { get; }
    public Distance(double m) => Meters = m;

    public static implicit operator double(Distance d) => d.Meters;
    public static implicit operator Distance(double m) => new(m);
}

Distance d = 100;          // double → Distance (implicit)
double m = d;              // Distance → double (implicit)
double total = d + 50;      // Distance → double, then arithmetic
```

---

## 6. Real-world examples

### Money type

```csharp
public readonly struct Money : IEquatable<Money>, IComparable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency ?? throw new ArgumentNullException();
    }

    // Arithmetic
    public static Money operator +(Money a, Money b)
    {
        EnsureSameCurrency(a, b);
        return new Money(a.Amount + b.Amount, a.Currency);
    }

    public static Money operator -(Money a, Money b)
    {
        EnsureSameCurrency(a, b);
        return new Money(a.Amount - b.Amount, a.Currency);
    }

    public static Money operator *(Money m, decimal factor) =>
        new(m.Amount * factor, m.Currency);

    public static Money operator /(Money m, decimal divisor) =>
        new(m.Amount / divisor, m.Currency);

    // Comparison
    public int CompareTo(Money other)
    {
        EnsureSameCurrency(this, other);
        return Amount.CompareTo(other.Amount);
    }

    public static bool operator <(Money a, Money b) => a.CompareTo(b) < 0;
    public static bool operator >(Money a, Money b) => a.CompareTo(b) > 0;
    public static bool operator <=(Money a, Money b) => a.CompareTo(b) <= 0;
    public static bool operator >=(Money a, Money b) => a.CompareTo(b) >= 0;

    // Equality
    public bool Equals(Money other) => Amount == other.Amount && Currency == other.Currency;
    public override bool Equals(object? obj) => obj is Money m && Equals(m);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    public static bool operator ==(Money a, Money b) => a.Equals(b);
    public static bool operator !=(Money a, Money b) => !a.Equals(b);

    // String representation
    public override string ToString() => $"{Amount:F2} {Currency}";

    private static void EnsureSameCurrency(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException("Different currencies");
    }
}

// Use
var price = new Money(99.99m, "USD");
var tax = price * 0.08m;
var total = price + tax;
Console.WriteLine(total);  // "107.99 USD"
```

### Custom Collection с indexer

```csharp
public class CircularBuffer<T>
{
    private readonly T[] _buffer;
    private int _start = 0;
    public int Count { get; private set; } = 0;
    public int Capacity => _buffer.Length;

    public CircularBuffer(int capacity) => _buffer = new T[capacity];

    public T this[int index]
    {
        get
        {
            if (index < 0 || index >= Count)
                throw new ArgumentOutOfRangeException();
            return _buffer[(_start + index) % Capacity];
        }
    }

    public void Add(T item)
    {
        if (Count < Capacity)
        {
            _buffer[(_start + Count) % Capacity] = item;
            Count++;
        }
        else
        {
            _buffer[_start] = item;
            _start = (_start + 1) % Capacity;
        }
    }
}

var buffer = new CircularBuffer<int>(3);
buffer.Add(1); buffer.Add(2); buffer.Add(3); buffer.Add(4);
buffer[0];  // 2 (1 был overwritten)
buffer[1];  // 3
buffer[2];  // 4
```

### Type-safe IDs

```csharp
public readonly struct UserId : IEquatable<UserId>
{
    public Guid Value { get; }
    public UserId(Guid value) => Value = value;

    public static implicit operator Guid(UserId id) => id.Value;
    public static explicit operator UserId(Guid g) => new(g);

    public bool Equals(UserId other) => Value == other.Value;
    public override bool Equals(object? obj) => obj is UserId id && Equals(id);
    public override int GetHashCode() => Value.GetHashCode();
    public static bool operator ==(UserId a, UserId b) => a.Equals(b);
    public static bool operator !=(UserId a, UserId b) => !a.Equals(b);

    public override string ToString() => Value.ToString();
}

public readonly struct OrderId
{
    public Guid Value { get; }
    public OrderId(Guid value) => Value = value;
    // ... аналогично
}

// Type safety
public Order GetOrder(OrderId id) { /* ... */ }

UserId userId = ...;
GetOrder(userId);  // ❌ Compile error — won't accept UserId

OrderId orderId = ...;
GetOrder(orderId);  // ✅
```

См. [[oop|OOP]] — strong types.

---

## 7. Best Practices

### Indexers

- **Имя класса collection-like** — User`s`, Items
- **Documented behavior** — что возвращает если не найден?
- **`Count` / `Length` property** для iteration
- **Implement IEnumerable\<T\>** для foreach
- **Consider `Index` / `Range`** support (C# 8+)

### Arithmetic operators

- **Только когда math имеет смысл** — Vector, Money, не User
- **`readonly struct` для value types** — immutable
- **Возвращай новый instance** — не mutate
- **Все основные operators парами** (`+ -`, `* /`)
- **Commutative explicit** (`a*b` и `b*a`)

### Comparison

- **Implement `IComparable<T>`** ALWAYS если operators есть
- **`<` и `>`** — пара
- **`<=` и `>=`** — пара
- **Consistent с CompareTo**

### Equality

- **Records если возможно** — auto-generated
- **`==` и `!=`** — пара
- **Override Equals + GetHashCode** ALWAYS если `==`
- **`IEquatable<T>`** для performance (no boxing)

### Conversions

- **Implicit только если safe** — lossless, intuitive
- **Explicit для lossy / risky**
- **Documented** — что conversion делает
- **Не конвертируй в несвязанные типы**

---

## 8. Common Pitfalls

### 1. Operators не reflect logic types

```csharp
// ❌ User + User = ? Не имеет смысла
public static User operator +(User a, User b) { /* ??? */ }

// ✅ Используй методы для domain operations
public class Team
{
    public void AddMember(User user) { /* ... */ }
}
team.AddMember(user);
```

### 2. Equality operator без Equals override

```csharp
public class Point
{
    public static bool operator ==(Point? a, Point? b) => a?.Equals(b) ?? b is null;
    // ⚠️ Equals не overridden — uses default reference equality!
    // a == b может расходиться с a.Equals(b)
}

// ✅ Override и Equals и operators (или используй record)
public record Point(int X, int Y);
```

### 3. Indexer для нелинейного access

```csharp
// ❌ User[id] но lookup по списку — O(N)!
public class Users
{
    private List<User> _users = new();
    public User this[int id] => _users.First(u => u.Id == id);  // slow
}

// ✅ Dictionary под капотом для O(1)
public class Users
{
    private Dictionary<int, User> _users = new();
    public User this[int id] => _users[id];
}
```

### 4. Side effects в operator

```csharp
// ❌ Side effect — log, save, etc
public static Money operator +(Money a, Money b)
{
    Log($"Adding {a} + {b}");  // ⚠️ pure operations expected
    Database.Save(...);          // ⚠️ surprise!
    return new Money(a.Amount + b.Amount, a.Currency);
}

// ✅ Pure — predictable
public static Money operator +(Money a, Money b) =>
    new(a.Amount + b.Amount, a.Currency);
```

### 5. Different types в operators — ambiguity

```csharp
public class Distance
{
    public static implicit operator double(Distance d) => d.Meters;
    public static implicit operator Distance(double m) => new Distance(m);

    public static Distance operator +(Distance a, Distance b) => new(a.Meters + b.Meters);
}

Distance d = 10;
double x = 5;
var sum = d + x;  // ⚠️ Ambiguous! d → double + double, OR x → Distance + Distance?
```

**Лечение:** explicit conversions, или избегай dual implicit.

### 6. `<` и `>` без CompareTo логики

```csharp
public static bool operator <(Money a, Money b) =>
    a.Amount < b.Amount;  // ⚠️ а если currencies разные?

// ✅
public static bool operator <(Money a, Money b)
{
    if (a.Currency != b.Currency)
        throw new InvalidOperationException();
    return a.Amount < b.Amount;
}
```

### 7. Overflow в arithmetic

```csharp
public static Vector operator *(Vector v, int scalar) =>
    new(v.X * scalar, v.Y * scalar);  // ⚠️ может overflow

// ✅ checked
public static Vector operator *(Vector v, int scalar) =>
    new(checked(v.X * scalar), checked(v.Y * scalar));

// Или используй decimal / long в storage
```

### 8. Implicit conversion везде

```csharp
public class Distance
{
    public static implicit operator string(Distance d) => $"{d.Meters}m";
    public static implicit operator double(Distance d) => d.Meters;
    public static implicit operator int(Distance d) => (int)d.Meters;
    public static implicit operator Distance(string s) => new(double.Parse(s));
}

// User гадает — что compile вызовет:
Console.WriteLine(distance);  // string? double? int?
```

**Лечение:** избегай many implicit conversions. Использовать **explicit** для не-trivial.

### 9. Indexer без bounds check

```csharp
public T this[int index] => _items[index];  // ⚠️ throws ArgumentOutOfRangeException

// ✅ Documented behavior + custom message
public T this[int index]
{
    get
    {
        if (index < 0 || index >= Count)
            throw new ArgumentOutOfRangeException(nameof(index), $"Index must be 0..{Count-1}");
        return _items[index];
    }
}
```

### 10. Operators на reference types — nullable issues

```csharp
public class Money
{
    public static Money operator +(Money a, Money b) =>
        new(a.Amount + b.Amount, a.Currency);  // ⚠️ NRE if any null
}

Money? m1 = null;
Money? m2 = new(10, "USD");
var sum = m1 + m2;  // 💥

// ✅ Handle null или используй struct
public static Money operator +(Money? a, Money? b) =>
    (a, b) switch
    {
        (null, null) => default,
        (var x, null) => x.Value,
        (null, var y) => y.Value,
        var (x, y) => new Money(x.Value.Amount + y.Value.Amount, x.Value.Currency)
    };
```

---

## 9. Cheat sheet

| Сценарий | Solution |
|----------|----------|
| Index access | `public T this[int index] { get; set; }` |
| Multi-key indexer | `public T this[int row, int col]` |
| String / int dual indexer | Multiple `this[T]` overloads |
| Range support | `public T this[Range r]` (C# 8+) |
| Math type | `public static T operator +(T a, T b)` |
| Conversion lossless | `public static implicit operator T(F f)` |
| Conversion lossy | `public static explicit operator T(F f)` |
| Equality | Records или manual `==`/`!=` + Equals + GHC |
| Comparison | `IComparable<T>` + `<`,`>`,`<=`,`>=` parами |
| Type-safe ID | `readonly struct ID { Guid Value }` + ops |
| Custom collection | indexer + Count + IEnumerable\<T\> |

---

## 10. Decision tree

```
Хочешь дать типу [] access?
│
├── Indexer — прямой index
│   ├── По int → list-like
│   ├── По string → dict-like
│   └── По multiple keys → multi-dim
│
└── Просто property с array? → public T[] Items { get; } 

Хочешь арифметику с типом?
│
├── Math semantic? (Vector, Matrix, Money)
│   → Operator overloading + readonly struct
│
├── Domain operation? (User join Team)
│   → Method, не operator
│
└── Conversion?
    ├── Lossless / safe → implicit
    └── Lossy / explicit → explicit

Equality?
│
├── Value type? → record / record struct
├── Class with value semantics? → manual ops + IEquatable
└── Identity-based? → не overload (default reference =)
```

---

## См. также

- [[csharp-basics|C# Basics]] — operators intro
- [[equality-comparison|Equality и Comparison]] — `==`, Equals deep
- [[oop|OOP]] — methods vs operators
- [[functional-csharp|Functional C#]] — records (built-in equality)
- [[modern-features|Modern Features]] — Index, Range
- [[generics-deep|Generics Deep]] — INumber\<T\> (.NET 7+)

## Reading list

- **Microsoft Docs — Operator overloading** — learn.microsoft.com/dotnet/csharp/language-reference/operators/operator-overloading
- **Microsoft Docs — Indexers** — learn.microsoft.com/dotnet/csharp/programming-guide/indexers
- **Eric Lippert — Operator design** — ericlippert.com (классика)
- **Effective C#** — Bill Wagner (operators chapter)
- **Framework Design Guidelines** — Cwalina & Abrams
