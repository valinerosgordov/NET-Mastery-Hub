---
tags: [csharp, equality, comparison, middle, equals, gethashcode, icomparable, iequatable]
level: Middle
date: 2026-05-07
---

# Equality и Comparison — равенство и сравнение

> **Reference equality vs value equality, контракт `Equals`/`GetHashCode`, `IEquatable<T>`, `IComparable<T>`, custom comparers.** Закрывает пробел: «знаю про `==`, не понимаю когда переопределять `Equals` и почему `GetHashCode` обязателен парой».

---

## 0. Как читать

Если впервые работаешь с equality — раздел 1→4 подряд. Если уже override-ил Equals, но непонятно contract — раздел 3 (rules). Если строишь production — раздел 7 (custom comparers), 9 (best practices), 11 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Reference vs value equality

```csharp
class Person { public string Name = ""; }

var p1 = new Person { Name = "Alice" };
var p2 = new Person { Name = "Alice" };
var p3 = p1;

p1 == p2;          // false — разные references (по умолчанию для class)
p1 == p3;          // true — same reference
p1.Equals(p2);     // false (default Object.Equals = reference)
ReferenceEquals(p1, p2);   // false
ReferenceEquals(p1, p3);   // true
```

По умолчанию для class:
- `==` — reference equality
- `Equals(Object)` — reference equality (Object.Equals default)
- `ReferenceEquals` — всегда reference equality

Для **value equality** (по содержимому полей) нужно override.

### 1.2. Value types — value equality by default

```csharp
struct Point { public int X, Y; }

var p1 = new Point { X = 1, Y = 2 };
var p2 = new Point { X = 1, Y = 2 };

p1.Equals(p2);    // true — default ValueType.Equals использует reflection!
p1 == p2;         // ❌ Compile error — operator не определён по умолчанию
```

`ValueType.Equals` — default через **reflection** (медленно). Best practice: override + implement `IEquatable<T>` для perf.

### 1.3. Records — value equality автоматически

```csharp
record User(int Id, string Name);

var u1 = new User(1, "Alice");
var u2 = new User(1, "Alice");

u1 == u2;          // true!
u1.Equals(u2);     // true
```

Records (C# 9+) автоматически генерируют value equality по всем полям (см. [[oop|OOP]] раздел 11).

### 1.4. Главное правило

```
Override Equals когда:
  - Domain entity с identity (Id) → equals by Id
  - Value object — равенство по полям
  - Иначе — оставь default reference equality

Если override Equals:
  - ОБЯЗАТЕЛЬНО override GetHashCode
  - Implement IEquatable<T> для perf (no boxing для value types)
  - Override == и != operators (optional но conventional)
  - Equality contract — reflexive, symmetric, transitive, consistent

Используй record вместо ручного override
  если value equality по всем полям подойдёт.
```

### 1.5. Эволюция

| Версия | Что |
|--------|-----|
| **.NET 1.0** | `Object.Equals`, `Object.GetHashCode`, `IComparable` |
| **.NET 2.0** | `IEquatable<T>`, `IComparable<T>`, `IComparer<T>`, `IEqualityComparer<T>` |
| **.NET 4.5** | `EqualityComparer<T>.Default`, `Comparer<T>.Default` |
| **C# 7+** | tuple equality (`(1,2) == (1,2)`) |
| **C# 9** | Records — auto-generated equality |
| **.NET 8+** | Generic Math interfaces — `IEqualityOperators`, `IComparisonOperators` |

> [!info]- Если ты знаешь Java / Python / Rust / JavaScript
> **Java:** `equals()` + `hashCode()` (пара обязательна, как в C#). `compareTo` соответствует `IComparable`. `Comparator` соответствует `IComparer`. `==` всегда reference (нет operator overload).
>
> **Python:** `__eq__`, `__hash__` методы. Tuples / dataclasses — value equality автоматически. `==` использует `__eq__`.
>
> **Rust:** `PartialEq`/`Eq` traits, `Hash` trait, `PartialOrd`/`Ord`. Compiler-derived через `#[derive(PartialEq, Eq, Hash)]`.
>
> **JavaScript:** `===` strict equality (reference для objects), `==` с coercion. Нет встроенного value equality для objects — manual или libraries (lodash isEqual).

> [!question]- Интервью: чем reference equality отличается от value equality?
> **Reference equality** — `==` сравнивает **references** (адреса в heap). Default для class. `ReferenceEquals(a, b)` всегда reference. **Value equality** — сравнение **по содержимому** полей. Default для struct (через reflection — медленно). Override `Equals` + `GetHashCode` чтобы определить custom equality. Records (C# 9+) автоматически value по полям. String — special case: class, но `==` value (overloaded). DateTime, decimal, primitives — value (struct). Best practice: явно override + `IEquatable<T>` для domain types.

---

## 2. Equals и GetHashCode — base implementation

### 2.1. Object.Equals base

```csharp
// От Object — virtual
public virtual bool Equals(object? obj);
public virtual int GetHashCode();
```

Default: reference equality + identity-based hash. Любой класс может override.

### 2.2. Минимальный override для class

```csharp
public class Person : IEquatable<Person>
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    
    public bool Equals(Person? other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;
        return Id == other.Id;
    }
    
    public override bool Equals(object? obj) => Equals(obj as Person);
    
    public override int GetHashCode() => Id.GetHashCode();
    
    public static bool operator ==(Person? a, Person? b)
    {
        if (a is null) return b is null;
        return a.Equals(b);
    }
    
    public static bool operator !=(Person? a, Person? b) => !(a == b);
}
```

Полный pattern: `IEquatable<T>.Equals(T)`, `Object.Equals(object)`, `GetHashCode`, `==`, `!=`.

### 2.3. HashCode.Combine

```csharp
public class Address : IEquatable<Address>
{
    public string Street { get; init; } = "";
    public string City { get; init; } = "";
    public string Country { get; init; } = "";
    
    public bool Equals(Address? other) =>
        other is not null &&
        Street == other.Street &&
        City == other.City &&
        Country == other.Country;
    
    public override bool Equals(object? obj) => Equals(obj as Address);
    
    public override int GetHashCode() =>
        HashCode.Combine(Street, City, Country);
}
```

`HashCode.Combine(...)` — built-in helper для combining нескольких полей в hash. Лучше ручной XOR-magic.

### 2.4. Старый стиль hash combination

```csharp
public override int GetHashCode()
{
    unchecked
    {
        int hash = 17;
        hash = hash * 31 + (Street?.GetHashCode() ?? 0);
        hash = hash * 31 + (City?.GetHashCode() ?? 0);
        hash = hash * 31 + (Country?.GetHashCode() ?? 0);
        return hash;
    }
}
```

`unchecked` — overflow при умножении OK. Числа 17, 31 — традиционные primes. Используй `HashCode.Combine` если возможно.

### 2.5. Records — generates всё

```csharp
public record User(int Id, string Name);
// Compiler генерирует:
//   public virtual bool Equals(User? other)
//   public override bool Equals(object? obj)
//   public override int GetHashCode()
//   public static bool operator ==(User? a, User? b)
//   public static bool operator !=(User? a, User? b)
```

### 2.6. Tuple equality

```csharp
var t1 = (1, "a");
var t2 = (1, "a");
t1 == t2;   // true (C# 7+)

var t3 = (Id: 1, Name: "a");
var t4 = (Id: 1, Name: "a");
t3 == t4;   // true
```

ValueTuple сравнивается покомпонентно. Удобно для composite keys.

> [!question]- Интервью: почему override Equals требует override GetHashCode?
> Contract: **equal objects must have equal hash codes**. Если нарушить — Dictionary/HashSet поломаются. Lookup использует hash для bucket, потом equals для finalize match. Если equal objects имеют разный hash — попадут в разные buckets, lookup не найдёт. Compiler warning CS0659 предупреждает. Best practice: override **both** или ни one. Используй `HashCode.Combine(field1, field2, ...)` для clean hash из нескольких полей.

---

## 3. Equality contract — 4 правила

### 3.1. Reflexivity

```csharp
x.Equals(x) == true;   // ВСЕГДА true (для non-null x)
```

Объект всегда equal себе. Даже NaN — `double.NaN.Equals(double.NaN) == true` (special!), хотя `double.NaN == double.NaN == false`.

### 3.2. Symmetry

```csharp
x.Equals(y) == y.Equals(x);
```

Если `a.Equals(b)` true — `b.Equals(a)` тоже true.

**Ловушка:** в inheritance, derived с extra полями ломает symmetry.

### 3.3. Transitivity

```csharp
if (x.Equals(y) && y.Equals(z)) → x.Equals(z) == true
```

Цепочка equals должна work.

### 3.4. Consistency

Multiple calls с unchanged state дают тот же результат. Equality не должна меняться рандомно.

### 3.5. null handling

```csharp
x.Equals(null) == false;   // (для non-null x)
```

Static `Object.Equals(a, b)` handles nulls — `Object.Equals(null, null) == true`.

### 3.6. Equal objects → equal hash codes

```csharp
if (a.Equals(b)) → a.GetHashCode() == b.GetHashCode()
```

**Reverse не required** — equal hash не значит equal objects (collisions OK). Но equal objects MUST have equal hash.

> [!question]- Интервью: какие 4 правила equality contract?
> 1) **Reflexive** — `x.Equals(x) == true` всегда. 2) **Symmetric** — `x.Equals(y) == y.Equals(x)`. 3) **Transitive** — если a==b и b==c, то a==c. 4) **Consistent** — multiple calls with unchanged state дают тот же результат. Дополнительно: `x.Equals(null) == false`, equal objects must have equal hash codes (reverse не требуется). Нарушение contract ломает collections (Dictionary, HashSet) и LINQ операторы (Distinct, GroupBy, Contains).

---

## 4. IEquatable&lt;T&gt; — typed equality

### 4.1. Зачем

`Object.Equals(object)` принимает `object` — для value types это **boxing** на каждом call:

```csharp
Point p1 = new(1, 2), p2 = new(1, 2);
p1.Equals(p2);   // boxes p2 → object → heap allocation
```

`IEquatable<T>` — typed version без boxing:

```csharp
public struct Point : IEquatable<Point>
{
    public int X, Y;
    
    public bool Equals(Point other) => X == other.X && Y == other.Y;
    public override bool Equals(object? obj) => obj is Point p && Equals(p);
    public override int GetHashCode() => HashCode.Combine(X, Y);
}

p1.Equals(p2);   // вызывает IEquatable.Equals(Point) — no boxing
```

### 4.2. Generic collections используют IEquatable

```csharp
List<Point>.Contains(p);   // если IEquatable<Point> — fast
HashSet<Point>             // тоже использует IEquatable
Dictionary<Point, int>     // hash + IEquatable
```

`EqualityComparer<T>.Default` автоматически выбирает `IEquatable<T>.Equals` если implemented.

### 4.3. Реализация для class

```csharp
public class Order : IEquatable<Order>
{
    public int Id { get; init; }
    
    public bool Equals(Order? other) =>
        other is not null && Id == other.Id;
    
    public override bool Equals(object? obj) => Equals(obj as Order);
    public override int GetHashCode() => Id.GetHashCode();
}
```

Для class — `IEquatable<T>` тоже полезен (clean, без `obj as T` cast).

### 4.4. Records — IEquatable&lt;T&gt; automatically

```csharp
public record User(int Id, string Name);
// User : IEquatable<User> — автоматически
```

### 4.5. Когда IEquatable

✅ **Используй когда:**
- Custom value equality для type.
- Type используется в generic collections.
- Performance важен (avoid boxing для value types).

❌ **Не нужно когда:**
- Default reference equality достаточен.
- Records (auto-generated).

> [!question]- Интервью: зачем `IEquatable<T>` если есть `Equals(object)`?
> 1) **No boxing** для value types — typed `Equals(T)` принимает T напрямую. 2) **Type safety** — compile-time check что compare правильным типом. 3) **Generic collections** (`HashSet<T>`, `Dictionary<TKey,TValue>`) предпочитают `IEquatable<T>.Equals` через `EqualityComparer<T>.Default`. 4) **Cleaner code**. Standard pattern: implement `IEquatable<T>` + override `Equals(object)` + override `GetHashCode`. Records делают всё это автоматически.

---

## 5. == и != operators

### 5.1. Override operators

```csharp
public sealed class Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal amount, string currency) { Amount = amount; Currency = currency; }
    
    public bool Equals(Money? other) =>
        other is not null && Amount == other.Amount && Currency == other.Currency;
    
    public override bool Equals(object? obj) => Equals(obj as Money);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    
    public static bool operator ==(Money? a, Money? b)
    {
        if (a is null) return b is null;
        return a.Equals(b);
    }
    
    public static bool operator !=(Money? a, Money? b) => !(a == b);
}
```

### 5.2. Соглашение для class

Microsoft Framework Design Guidelines: для **immutable value types** → operator overload OK. Для mutable classes — обычно **избегать** (`==` может confuse с reference equality).

### 5.3. Records — automatic

```csharp
public record User(int Id, string Name);
// == и != сгенерированы автоматически
```

> [!question]- Интервью: когда переопределять `==` operator?
> Для **immutable value types** (Money, Coordinate, Color) — да. Для **mutable entities** (Order, User с changing state) — Microsoft FDG рекомендует **избегать**. Reason: `==` намекает на value equality, но если entity mutable — confuse с reference. Records — automatic operators (since C# 9). String — exception: class, но `==` overload для intuitive value compare.

---

## 6. IComparable&lt;T&gt; и IComparer&lt;T&gt;

### 6.1. IComparable — natural ordering

```csharp
public class Version : IComparable<Version>
{
    public int Major { get; init; }
    public int Minor { get; init; }
    public int Patch { get; init; }
    
    public int CompareTo(Version? other)
    {
        if (other is null) return 1;
        
        int c = Major.CompareTo(other.Major);
        if (c != 0) return c;
        
        c = Minor.CompareTo(other.Minor);
        if (c != 0) return c;
        
        return Patch.CompareTo(other.Patch);
    }
}

versions.Sort();   // использует IComparable
```

`CompareTo` returns: `< 0` — this **before** other; `= 0` — equal; `> 0` — this **after** other.

### 6.2. IComparer — external comparer

Когда нужно сравнивать без modifying тип:

```csharp
public class PersonByAgeComparer : IComparer<Person>
{
    public int Compare(Person? x, Person? y)
    {
        if (x is null && y is null) return 0;
        if (x is null) return -1;
        if (y is null) return 1;
        return x.Age.CompareTo(y.Age);
    }
}

people.Sort(new PersonByAgeComparer());
```

### 6.3. Lambda comparer

```csharp
people.Sort((a, b) => a.Age.CompareTo(b.Age));
people.Sort((a, b) => b.Age.CompareTo(a.Age));   // descending

var comparer = Comparer<Person>.Create((a, b) => a.Age.CompareTo(b.Age));
```

`Comparer<T>.Create(lambda)` — quick comparer без отдельного класса.

### 6.4. LINQ + comparison

```csharp
var sorted = people
    .OrderBy(p => p.LastName)
    .ThenBy(p => p.FirstName)
    .ToList();

var sorted2 = people.OrderBy(p => p.Name, StringComparer.OrdinalIgnoreCase);
```

### 6.5. Operators

```csharp
public class Version : IComparable<Version>
{
    public static bool operator <(Version a, Version b) => a.CompareTo(b) < 0;
    public static bool operator >(Version a, Version b) => a.CompareTo(b) > 0;
    public static bool operator <=(Version a, Version b) => a.CompareTo(b) <= 0;
    public static bool operator >=(Version a, Version b) => a.CompareTo(b) >= 0;
}
```

Microsoft FDG: рекомендует для типов, которые natural-ordered.

> [!question]- Интервью: чем `IComparable<T>` отличается от `IComparer<T>`?
> **`IComparable<T>`** — реализуется **самим типом**, defines его **natural ordering** (`CompareTo` метод). Used by `List.Sort()`, `Array.Sort()` без extra параметров. **`IComparer<T>`** — отдельный объект с `Compare(x, y)` методом. Передаётся в Sort / OrderBy. Позволяет несколько orderings для одного типа без modifying тип. `Comparer<T>.Default` использует IComparable. Lambda comparers — short-form через `Comparer<T>.Create(lambda)`.

---

## 7. IEqualityComparer&lt;T&gt; — custom equality

### 7.1. Зачем

Когда стандартный equality типа не подходит (или нельзя modify тип):

```csharp
public class UserByEmailComparer : IEqualityComparer<User>
{
    public bool Equals(User? x, User? y) =>
        string.Equals(x?.Email, y?.Email, StringComparison.OrdinalIgnoreCase);
    
    public int GetHashCode(User obj) =>
        StringComparer.OrdinalIgnoreCase.GetHashCode(obj.Email);
}

var unique = users.Distinct(new UserByEmailComparer());
var dict = new Dictionary<User, int>(new UserByEmailComparer());
```

### 7.2. Built-in StringComparer

```csharp
StringComparer.Ordinal
StringComparer.OrdinalIgnoreCase
StringComparer.InvariantCulture
StringComparer.InvariantCultureIgnoreCase
StringComparer.CurrentCulture
StringComparer.CurrentCultureIgnoreCase
```

Используются как `IEqualityComparer<string>` и `IComparer<string>`.

```csharp
var dict = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
dict["Alice"] = 1;
dict["alice"];   // 1 — case-insensitive
```

### 7.3. EqualityComparer.Default

```csharp
var cmp = EqualityComparer<int>.Default;   // использует IEquatable<int>
cmp.Equals(1, 1);   // true

var personCmp = EqualityComparer<Person>.Default;
// Использует IEquatable<Person> если implemented, иначе Object.Equals
```

Generic collections используют `EqualityComparer<T>.Default` под капотом.

### 7.4. LINQ + custom comparer

```csharp
users.Distinct(new UserByEmailComparer());
users.GroupBy(u => u.Email, StringComparer.OrdinalIgnoreCase);
list1.SequenceEqual(list2, customComparer);
list1.Except(list2, customComparer);
```

Все relevant LINQ операторы accept `IEqualityComparer<T>` overload.

### 7.5. Comparer для sorted collections

```csharp
var sortedSet = new SortedSet<Person>(
    Comparer<Person>.Create((a, b) => a.Age.CompareTo(b.Age)));
var sortedDict = new SortedDictionary<string, int>(StringComparer.OrdinalIgnoreCase);
```

> [!question]- Интервью: чем `IEqualityComparer<T>` отличается от `IEquatable<T>`?
> **`IEquatable<T>`** — реализуется **самим типом** для **естественного** equality. Один definition. **`IEqualityComparer<T>`** — отдельный объект с парой методов `Equals(x, y)` + `GetHashCode(obj)`. Позволяет alternate equality strategies без modify тип. Передаётся в LINQ операторы (Distinct, GroupBy, Except, SequenceEqual) и Dictionary/HashSet constructor. Built-in примеры: `StringComparer.OrdinalIgnoreCase`, `StringComparer.InvariantCulture`. Best practice: use built-in StringComparer для строк, custom для domain-specific.

---

## 8. Equality в наследовании

### 8.1. Проблема inheritance

```csharp
class Animal { public string Name = ""; }
class Dog : Animal { public string Breed = ""; }
```

Если `Animal.Equals` сравнивает только `Name`, а `Dog.Equals` ещё `Breed`:
- `animal.Equals(dog)` — true (Name match)
- `dog.Equals(animal)` — false (Breed missing)
- **Symmetry нарушена!**

### 8.2. Решение 1 — sealed класс

```csharp
public sealed class Money : IEquatable<Money>
{
    // ... equality по всем полям
}
```

Если класс sealed — никто не наследует, проблемы нет. **Best practice** для value types.

### 8.3. Решение 2 — exact type check

```csharp
public override bool Equals(object? obj)
{
    if (obj is null) return false;
    if (GetType() != obj.GetType()) return false;
    // ... compare fields
}
```

`GetType() == other.GetType()` — оба должны быть **точно** Animal или **точно** Dog.

### 8.4. Решение 3 — DDD entity hierarchy

```csharp
public abstract class Entity : IEquatable<Entity>
{
    public Guid Id { get; init; }
    
    public bool Equals(Entity? other) =>
        other is not null && Id == other.Id;
    
    public override bool Equals(object? obj) => Equals(obj as Entity);
    public override int GetHashCode() => Id.GetHashCode();
}

public class User : Entity { /* ... */ }
public class Order : Entity { /* ... */ }
```

Все entities equal by Id — независимо от derived type. Common DDD pattern.

### 8.5. Records и inheritance

```csharp
public abstract record Animal(string Name);
public record Dog(string Name, string Breed) : Animal(Name);
```

Records делают type-aware equality автоматически — Dog != Animal даже с same Name.

> [!question]- Интервью: какая проблема equality в inheritance?
> **Symmetry** ломается: если base сравнивает поля A, derived — A+B, то `derived.Equals(base)` returns false, но `base.Equals(derived)` true. Решения: 1) **`sealed` class** — нет inheritance проблемы. 2) **Exact type check** — `GetType() == other.GetType()`. 3) **DDD pattern** — abstract base с equality по Id. 4) **Records** — automatic runtime type check. Microsoft FDG рекомендует sealed для value types.

---

## 9. Best Practices

### 9.1. Implementing equality

- ✅ **`record`** для value-equality.
- ✅ **`IEquatable<T>`** — typed, no boxing.
- ✅ **Override Equals + GetHashCode together** (CS0659 warning).
- ✅ **`HashCode.Combine(...)`** для multi-field hash.
- ✅ **`sealed`** для value types — избежать inheritance issues.
- ✅ **Operator `==`/`!=`** для immutable value types.
- ❌ **Override один из пары** — Equals без GetHashCode.
- ❌ **Mutable hash** — поля участвующие в hash не должны меняться.
- ❌ **Random / time-based hash** — нарушает consistency.

### 9.2. Comparing

- ✅ **`IComparable<T>`** для natural ordering.
- ✅ **`IComparer<T>`** для alternate orderings без modifying тип.
- ✅ **`Comparer<T>.Create(lambda)`** для quick comparers.
- ✅ **`StringComparer.OrdinalIgnoreCase`** для string keys.
- ❌ **Inconsistent ordering** — нарушение transitivity ломает sort.

### 9.3. Performance

- ✅ **Cached hash** для immutable types с expensive hash.
- ✅ **`EqualityComparer<T>.Default`** в generic code.
- ❌ **Reflection-based default ValueType.Equals** — медленно.

---

## 10. Decision tree

```
Что нужно?
│
├── Value-equality по всем полям → record (auto)
├── Custom equality для class
│   ├── По identity (Id) → Equals + GetHashCode + IEquatable
│   ├── По всем полям → record или manual override
│   └── По subset полей → manual + IEquatable
├── Custom equality без modify type → IEqualityComparer
├── Sorting
│   ├── Natural ordering → IComparable
│   └── External / multiple orderings → IComparer (или lambda)
├── String equality в Dictionary/HashSet → StringComparer.OrdinalIgnoreCase
└── Inheritance
    ├── Sealed class — best
    ├── DDD entity — equals by Id
    └── Generic hierarchy — exact type check (GetType())
```

---

## 11. Cheat sheet

```csharp
// === Record (auto) ===
public record User(int Id, string Name);

// === Manual class — full pattern ===
public class Person : IEquatable<Person>
{
    public int Id { get; init; }
    public string Name { get; init; } = "";
    
    public bool Equals(Person? other) =>
        other is not null && Id == other.Id;
    
    public override bool Equals(object? obj) => Equals(obj as Person);
    public override int GetHashCode() => Id.GetHashCode();
    
    public static bool operator ==(Person? a, Person? b) =>
        a is null ? b is null : a.Equals(b);
    public static bool operator !=(Person? a, Person? b) => !(a == b);
}

// === Multi-field hash ===
public override int GetHashCode() =>
    HashCode.Combine(Field1, Field2, Field3);

// === IComparable ===
public class Version : IComparable<Version>
{
    public int CompareTo(Version? other) { /* ... */ }
}

// === IComparer (lambda) ===
var byAge = Comparer<Person>.Create((a, b) => a.Age.CompareTo(b.Age));

// === IEqualityComparer ===
public class ByEmailComparer : IEqualityComparer<User>
{
    public bool Equals(User? x, User? y) =>
        string.Equals(x?.Email, y?.Email, StringComparison.OrdinalIgnoreCase);
    public int GetHashCode(User obj) =>
        StringComparer.OrdinalIgnoreCase.GetHashCode(obj.Email);
}

// === StringComparer (built-in) ===
var dict = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
```

---

## 12. Common Pitfalls

### 12.1. Override Equals без GetHashCode

```csharp
public override bool Equals(object? obj) { /* ... */ }
// ❌ CS0659 — нет GetHashCode!
```

**Механизм:** Dictionary lookup поломан (equal items в разных buckets).
**Фикс:** override **both** или используй record.

### 12.2. Mutable поля в hash

```csharp
public class BadKey
{
    public int Id { get; set; }   // mutable!
    public override int GetHashCode() => Id;
}

var dict = new Dictionary<BadKey, string>();
var k = new BadKey { Id = 1 };
dict[k] = "x";
k.Id = 2;
dict[k];   // KeyNotFoundException!
```

**Фикс:** immutable keys (init-only properties, records).

### 12.3. ValueType.Equals reflection slow

```csharp
struct Point { public int X, Y; }
// Default Equals — reflection
```

**Фикс:** override + `IEquatable<Point>`.

### 12.4. Symmetry в inheritance

```csharp
class Animal { /* equals by Name */ }
class Dog : Animal { /* equals by Name + Breed */ }

animal.Equals(dog);   // true
dog.Equals(animal);   // false — symmetry violation!
```

**Фикс:** `sealed`, exact type check, или DDD by Id.

### 12.5. Equals не handle null

```csharp
public bool Equals(Person other) => Id == other.Id;
// other может быть null
```

**Фикс:** `Person?` параметр + null check.

### 12.6. ToLower allocation в hash

```csharp
public override int GetHashCode() =>
    Email.ToLowerInvariant().GetHashCode();   // ❌ allocation
```

**Фикс:** `StringComparer.OrdinalIgnoreCase.GetHashCode(Email)`.

### 12.7. Throw в Equals

```csharp
public override bool Equals(object? obj)
{
    throw new NotImplementedException();   // ❌
}
```

**Механизм:** Dictionary/HashSet ломаются при insert.

### 12.8. Сравнение double через ==

```csharp
0.1 + 0.2 == 0.3;   // ❌ false для double — floating point!
0.1m + 0.2m == 0.3m;   // ✅ true для decimal
```

**Фикс:** для double — `Math.Abs(a - b) < epsilon`.

### 12.9. NaN inequality

```csharp
double.NaN == double.NaN;          // false — IEEE 754
double.NaN.Equals(double.NaN);     // true — overrideн в C#
```

**Фикс:** `double.IsNaN(x)`.

### 12.10. Inconsistent CompareTo

```csharp
public int CompareTo(Person? other) => Random.Next(-1, 2);   // ❌
```

**Фикс:** deterministic comparison.

> [!question]- Интервью: топ-3 ошибки с equality?
> 1) **Override Equals без GetHashCode** — CS0659, ломает Dictionary/HashSet. Always pair или record. 2) **Mutable hash key** — Dictionary ставит item по hash, mutation меняет hash, lookup в wrong bucket → KeyNotFoundException. Immutable keys. 3) **Symmetry violation в inheritance** — base сравнивает A, derived A+B → asymmetry. Решения: sealed, exact type check, или DDD equals by Id.

---

## 13. Practice exercises

### 13.1. Money value object

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0, currency);
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch");
        return this with { Amount = Amount + other.Amount };
    }
}

var m1 = new Money(100, "USD");
var m2 = new Money(100, "USD");
m1 == m2;   // true
```

### 13.2. Entity с Id-based equality

```csharp
public abstract class Entity<TKey> : IEquatable<Entity<TKey>>
    where TKey : IEquatable<TKey>
{
    public TKey Id { get; init; } = default!;
    
    public bool Equals(Entity<TKey>? other) =>
        other is not null && GetType() == other.GetType() && Id.Equals(other.Id);
    
    public override bool Equals(object? obj) => Equals(obj as Entity<TKey>);
    public override int GetHashCode() => HashCode.Combine(GetType(), Id);
}

public class User : Entity<int> { public string Name { get; set; } = ""; }
```

### 13.3. Custom comparer

```csharp
var byNameThenAge = Comparer<Person>.Create((a, b) =>
{
    int c = string.Compare(a.Name, b.Name, StringComparison.OrdinalIgnoreCase);
    return c != 0 ? c : a.Age.CompareTo(b.Age);
});

people.Sort(byNameThenAge);
```

---

## 14. Что читать дальше

1. **[[oop|OOP]]** — records, immutability.
2. **[[strings-regex|Strings]]** — StringComparison.
3. **[[collections-linq|Collections]]** — Dictionary, HashSet.
4. **Generic Math interfaces (.NET 7+)**.
5. **DDD value objects** — Eric Evans.

---

## 15. См. также

- [[oop|OOP]] — records
- [[strings-regex|Strings]] — `StringComparer`
- [[collections-linq|Collections]] — equality в Dictionary/HashSet
- [[nullable-types|Nullable]] — null handling
- HashCode.Combine API
- DDD value objects pattern

---

## 16. Reading list

- **Microsoft Docs — Equality comparisons** — learn.microsoft.com/dotnet/csharp/programming-guide/statements-expressions-operators/equality-comparisons
- **Microsoft Docs — Object.Equals** — learn.microsoft.com/dotnet/api/system.object.equals
- **Microsoft Docs — IEquatable** — learn.microsoft.com/dotnet/api/system.iequatable-1
- **Microsoft Docs — IComparable** — learn.microsoft.com/dotnet/api/system.icomparable-1
- **Microsoft Docs — HashCode struct** — learn.microsoft.com/dotnet/api/system.hashcode
- **Eric Lippert — Equality blog series** — ericlippert.com
- **Jon Skeet — C# in Depth (equality chapter)**
- **Jeffrey Richter — CLR via C#** — equality semantics chapter
- **Bill Wagner — Effective C# (items on equality)**
