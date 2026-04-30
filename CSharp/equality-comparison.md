---
tags: [csharp, equality, comparison, equals, gethashcode, iequatable, icomparable, middle]
level: Middle
date: 2026-04-30
---

# Equality и Comparison — равенство и сравнение

> **Глубокий гайд по равенству и сравнению в C#**. Equals, GetHashCode, == operator, IEquatable\<T\>, IComparable\<T\>, IComparer\<T\>, EqualityComparer\<T\>.Default. Closes частую source багов в коллекциях, dictionaries, sorts.

---

## Что это, зачем и когда

### Зачем это нужно

```csharp
class Person
{
    public string Name;
    public int Age;
}

var p1 = new Person { Name = "John", Age = 30 };
var p2 = new Person { Name = "John", Age = 30 };

p1 == p2;            // false! Разные объекты
p1.Equals(p2);       // false! По default — reference equality

var set = new HashSet<Person> { p1 };
set.Contains(p2);    // false! Same data, разный reference

var dict = new Dictionary<Person, string>();
dict[p1] = "first";
dict[p2];            // ❌ KeyNotFoundException!
```

Ситуация решается через правильную реализацию `Equals` + `GetHashCode`.

### Главное правило

> **Когда переопределяешь `Equals` — обязан переопределить `GetHashCode`. И наоборот.**

Иначе — баги в HashSet, Dictionary, LINQ.Distinct, etc.

---

## 1. Reference vs Value Equality

### Reference equality (default для classes)

```csharp
class Person
{
    public string Name;
}

var p1 = new Person { Name = "John" };
var p2 = new Person { Name = "John" };
var p3 = p1;

p1 == p2;             // false (разные объекты в памяти)
p1 == p3;             // true (та же ссылка)
ReferenceEquals(p1, p3);  // true
```

### Value equality (default для structs / records / primitives)

```csharp
int a = 5;
int b = 5;
a == b;               // true (сравнение значений)

struct Point { public int X, Y; }
var p1 = new Point { X = 1, Y = 2 };
var p2 = new Point { X = 1, Y = 2 };
p1.Equals(p2);        // true! Default struct equality сравнивает все fields

// Records (C# 9+)
record Person(string Name);
var r1 = new Person("John");
var r2 = new Person("John");
r1 == r2;             // true! Records — value equality по default
```

### Strings — special case

```csharp
string a = "hello";
string b = "hello";

a == b;               // true (string overrides ==)
a.Equals(b);          // true
ReferenceEquals(a, b);  // true (compiler interned literals)
                        // Но если runtime-created — false
```

---

## 2. Object.Equals, GetHashCode, ==

### Дефолтная реализация

```csharp
// От Object — class root:
public virtual bool Equals(object? obj);
public virtual int GetHashCode();
public static bool operator ==(object? left, object? right);  // через Equals для overridden types
```

**Default behavior:**

| Тип | `Equals` | `GetHashCode` | `==` |
|-----|----------|---------------|------|
| `class` | Reference | Random per instance | Reference |
| `struct` | Reflection-based field comparison (slow!) | Hash of fields | Не определён |
| `record` | Value (compiler-generated) | Value (compiler-generated) | Value |
| `record struct` | Value | Value | Value |

### Правильная реализация для class

```csharp
public class Person : IEquatable<Person>
{
    public string Name { get; }
    public int Age { get; }

    public Person(string name, int age) { Name = name; Age = age; }

    // Override object.Equals
    public override bool Equals(object? obj) => Equals(obj as Person);

    // IEquatable<T> — typed, без boxing
    public bool Equals(Person? other)
    {
        if (other is null) return false;
        if (ReferenceEquals(this, other)) return true;

        return Name == other.Name && Age == other.Age;
    }

    // GetHashCode — должен быть consistent с Equals
    public override int GetHashCode() => HashCode.Combine(Name, Age);

    // Operators (опционально)
    public static bool operator ==(Person? left, Person? right) =>
        left?.Equals(right) ?? right is null;

    public static bool operator !=(Person? left, Person? right) =>
        !(left == right);
}
```

### Проще через record

```csharp
// Все методы выше — auto-generated для record!
public record Person(string Name, int Age);

var p1 = new Person("John", 30);
var p2 = new Person("John", 30);

p1 == p2;             // true ✅
p1.Equals(p2);        // true ✅
p1.GetHashCode() == p2.GetHashCode();  // true ✅
```

> [!info] Records — лучший способ для DTO / value objects
> Если value equality нужен — record. Не пиши Equals/GetHashCode вручную.

---

## 3. Контракт Equals — invariants

### Equals правила

`Equals` должен быть:

1. **Reflexive** — `x.Equals(x)` → true
2. **Symmetric** — `x.Equals(y) == y.Equals(x)`
3. **Transitive** — `x.Equals(y) && y.Equals(z)` → `x.Equals(z)`
4. **Consistent** — multiple вызовов одинаковый результат если objects не изменились
5. **Null check** — `x.Equals(null)` → false

```csharp
// ❌ Нарушение symmetric — bad inheritance equality
class Base
{
    public override bool Equals(object? obj) => obj is Base;
}
class Derived : Base
{
    public override bool Equals(object? obj) => obj is Derived;
}

var b = new Base();
var d = new Derived();

b.Equals(d);  // true (d is Base)
d.Equals(b);  // false (b is not Derived)
// ⚠️ Asymmetric — undefined behavior в HashSets!
```

### GetHashCode правила

1. **Consistency с Equals** — `x.Equals(y)` → `x.GetHashCode() == y.GetHashCode()`
2. **Stability** — same object → same hash во время lifetime (если не mutated)
3. **Distribution** — разные objects должны иметь разные hashes (по возможности)

> [!warning] Mutable objects + collections
> Если object используется в HashSet/Dictionary, **не меняй fields** которые участвуют в Equals/GetHashCode! Hash protocol сломается, find не найдёт object.

```csharp
var person = new Person("John", 30);
var set = new HashSet<Person> { person };

person.Age = 31;  // ⚠️ Mutated!
set.Contains(person);  // может вернуть false!
```

**Лечение:** делай value objects immutable. Use records.

---

## 4. HashCode.Combine — modern way

```csharp
public override int GetHashCode() => HashCode.Combine(Name, Age, City);

// Для коллекций
public override int GetHashCode()
{
    var hash = new HashCode();
    foreach (var item in _items) hash.Add(item);
    return hash.ToHashCode();
}
```

`HashCode.Combine` — .NET Core 2.1+, лучшая дистрибуция чем manual `^` (XOR).

### Old way — не используй

```csharp
// ❌ Old style XOR — плохая distribution
public override int GetHashCode() => Name.GetHashCode() ^ Age.GetHashCode();

// HashCode("a", 1) == HashCode("b", 1) если NameHash совпал ⚠️
```

---

## 5. IEquatable\<T\> — strongly-typed equality

```csharp
public class Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    // Strongly-typed — без boxing для structs / без reflection для classes
    public bool Equals(Money? other) =>
        other is not null && Amount == other.Amount && Currency == other.Currency;

    // Override object.Equals — возвращает к typed
    public override bool Equals(object? obj) => Equals(obj as Money);

    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
}
```

### Зачем

```csharp
// Без IEquatable<T> — boxing!
struct Money { /* без IEquatable */ public decimal Amount, Currency; }

var m1 = new Money { Amount = 100 };
var m2 = new Money { Amount = 100 };

m1.Equals(m2);  // calls object.Equals — boxes m2 в object!
                 // Use reflection — slow

// С IEquatable<Money>
struct Money : IEquatable<Money> { /* ... */ }

m1.Equals(m2);  // calls Equals(Money) — no boxing, type-safe ✅
```

### EqualityComparer<T>.Default

```csharp
// Используется внутри HashSet, Dictionary, LINQ
var comparer = EqualityComparer<Money>.Default;
comparer.Equals(m1, m2);

// Под капотом: если T : IEquatable<T> — uses Equals(T)
//              иначе — Equals(object) (boxing!)
```

**Always implement `IEquatable<T>`** для types которые попадают в коллекции.

---

## 6. IComparable\<T\> — порядок (sorting)

Для **сортировки** — implement `IComparable<T>`.

```csharp
public class Version : IComparable<Version>
{
    public int Major { get; }
    public int Minor { get; }

    public int CompareTo(Version? other)
    {
        if (other is null) return 1;

        int majorCompare = Major.CompareTo(other.Major);
        if (majorCompare != 0) return majorCompare;

        return Minor.CompareTo(other.Minor);
    }
}

// Sort works automatically
var versions = new List<Version> { new(2,0), new(1,5), new(1,0) };
versions.Sort();  // 1.0, 1.5, 2.0
```

### CompareTo возвращает

- `< 0` — this **меньше** other
- `0` — равны (для sort)
- `> 0` — this **больше** other

### Override comparison operators

```csharp
public class Version : IComparable<Version>
{
    // ... CompareTo выше ...

    public static bool operator <(Version a, Version b) => a.CompareTo(b) < 0;
    public static bool operator >(Version a, Version b) => a.CompareTo(b) > 0;
    public static bool operator <=(Version a, Version b) => a.CompareTo(b) <= 0;
    public static bool operator >=(Version a, Version b) => a.CompareTo(b) >= 0;
}
```

---

## 7. IComparer\<T\> — кастомная сортировка

Внешний comparer — не нужно изменять класс.

```csharp
public class PersonAgeComparer : IComparer<Person>
{
    public int Compare(Person? x, Person? y)
    {
        if (x is null) return y is null ? 0 : -1;
        if (y is null) return 1;
        return x.Age.CompareTo(y.Age);
    }
}

// Sort by age
var people = new List<Person> { /* ... */ };
people.Sort(new PersonAgeComparer());

// Lambda — short
people.Sort((a, b) => a.Age.CompareTo(b.Age));

// LINQ
var sortedByAge = people.OrderBy(p => p.Age);
var sortedByAgeDesc = people.OrderByDescending(p => p.Age);

// Multiple criteria
var sorted = people
    .OrderBy(p => p.LastName)
    .ThenBy(p => p.FirstName)
    .ThenByDescending(p => p.Age);
```

### Comparer<T>.Create — inline

```csharp
var comparer = Comparer<Person>.Create((a, b) => a.Age.CompareTo(b.Age));
people.Sort(comparer);
```

---

## 8. IEqualityComparer\<T\> — custom equality

Custom equality для использования в HashSet / Dictionary.

```csharp
public class CaseInsensitiveStringComparer : IEqualityComparer<string>
{
    public bool Equals(string? x, string? y) =>
        string.Equals(x, y, StringComparison.OrdinalIgnoreCase);

    public int GetHashCode(string obj) =>
        obj?.ToLowerInvariant().GetHashCode() ?? 0;
}

// HashSet с custom equality
var set = new HashSet<string>(new CaseInsensitiveStringComparer());
set.Add("Hello");
set.Contains("HELLO");  // true!

// Dictionary
var dict = new Dictionary<string, int>(new CaseInsensitiveStringComparer());
dict["Hello"] = 1;
dict["HELLO"];  // 1
```

### Built-in StringComparer

```csharp
// Готовые comparers для строк
StringComparer.Ordinal              // binary, fastest
StringComparer.OrdinalIgnoreCase    // ignore case
StringComparer.InvariantCulture
StringComparer.InvariantCultureIgnoreCase
StringComparer.CurrentCulture       // depends on locale!
StringComparer.CurrentCultureIgnoreCase

// Использование
var dict = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
```

См. [[strings-regex|Strings — culture comparison]].

---

## 9. == operator — overloading

### Для классов

```csharp
public class Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public override bool Equals(object? obj) => Equals(obj as Money);
    public bool Equals(Money? other) => /* ... */;
    public override int GetHashCode() => /* ... */;

    // Operator overload
    public static bool operator ==(Money? a, Money? b)
    {
        if (a is null) return b is null;
        return a.Equals(b);
    }

    public static bool operator !=(Money? a, Money? b) => !(a == b);
}
```

### Pitfall: == не virtual!

```csharp
class Animal
{
    public static bool operator ==(Animal a, Animal b) { /* ... */ }
}

class Dog : Animal
{
    public static bool operator ==(Dog a, Dog b) { /* different impl */ }
}

Animal a = new Dog();
Animal b = new Dog();
a == b;  // ⚠️ Calls Animal == Animal, не Dog == Dog!
         // Compile-time dispatch, не virtual
```

`Equals` — virtual. `==` — нет. Используй `Equals` для polymorphic equality.

---

## 10. Records — auto-generated equality

C# 9+ records — компилятор генерирует **всё**:

```csharp
public record Person(string Name, int Age);

// Auto-generated:
// - Equals(object?)
// - Equals(Person?)
// - GetHashCode()
// - operator ==(Person, Person)
// - operator !=(Person, Person)
// - PrintMembers (для ToString)
// - Deconstruct
```

### Inheritance в records

```csharp
public record Animal(string Name);
public record Dog(string Name, string Breed) : Animal(Name);

var a = new Animal("Rex");
var d = new Dog("Rex", "Labrador");

a == d;  // false — different runtime types
        // Even если Name same!
```

Records compare **runtime types** + members. Безопасное наследование (symmetry preserved).

### record struct (.NET 6+)

```csharp
public record struct Point(int X, int Y);

var p1 = new Point(1, 2);
var p2 = new Point(1, 2);

p1 == p2;  // true ✅
// Stack-allocated, value semantics
```

См. [[functional-csharp|Functional C#]] и [[modern-features|Modern Features]].

---

## 11. struct equality — pitfalls

### Default struct equality — slow

```csharp
public struct Point
{
    public int X, Y;
}

var p1 = new Point { X = 1, Y = 2 };
var p2 = new Point { X = 1, Y = 2 };

p1.Equals(p2);  // true — но reflection-based!
```

Default struct `Equals`:
1. Boxes obj to ValueType
2. Use reflection to compare fields one by one
3. Slow (~100x slower than custom)

### Implement IEquatable<T> для structs

```csharp
public struct Point : IEquatable<Point>
{
    public int X { get; }
    public int Y { get; }

    public Point(int x, int y) { X = x; Y = y; }

    public bool Equals(Point other) => X == other.X && Y == other.Y;
    public override bool Equals(object? obj) => obj is Point p && Equals(p);
    public override int GetHashCode() => HashCode.Combine(X, Y);

    public static bool operator ==(Point a, Point b) => a.Equals(b);
    public static bool operator !=(Point a, Point b) => !a.Equals(b);
}

// 100x faster — нет boxing, нет reflection
```

### Лучше — record struct

```csharp
public record struct Point(int X, int Y);
// Всё auto-generated, быстрое equality
```

---

## 12. Common Pitfalls

### 1. Override Equals без GetHashCode

```csharp
// ❌
class Person
{
    public string Name;
    public override bool Equals(object? obj) =>
        obj is Person p && Name == p.Name;
    // GetHashCode не overridden! Default — random per instance.
}

var p1 = new Person { Name = "John" };
var p2 = new Person { Name = "John" };
var set = new HashSet<Person> { p1 };
set.Contains(p2);
// p1 и p2 равны (Equals=true), но GetHashCode разные!
// HashSet ищет в bucket по hash, не находит.
// Result: false — incorrect!
```

**Лечение:** override обоих, или используй record.

### 2. GetHashCode на mutable fields

```csharp
class Person
{
    public string Name { get; set; }  // mutable!

    public override int GetHashCode() => Name.GetHashCode();
}

var p = new Person { Name = "John" };
var dict = new Dictionary<Person, int> { [p] = 1 };

p.Name = "Jane";  // ⚠️ Mutated!
dict[p];  // ❌ KeyNotFoundException — hash изменился
```

**Лечение:** value objects immutable.

### 3. == для floats

```csharp
double a = 0.1 + 0.2;
double b = 0.3;

a == b;  // false! Floating-point precision

// ✅ Сравнивай с epsilon
const double Epsilon = 0.0001;
Math.Abs(a - b) < Epsilon;  // true

// ✅ Или decimal
decimal a = 0.1m + 0.2m;
decimal b = 0.3m;
a == b;  // true
```

### 4. Comparing strings с culture issues

```csharp
"file" == "FILE";  // false (case-sensitive)
"file".Equals("FILE", StringComparison.OrdinalIgnoreCase);  // true ✅

// Turkish I problem
"file".ToUpper() == "FILE";  // зависит от culture!
"file".ToUpperInvariant() == "FILE";  // ✅ predictable
```

### 5. CompareTo асимметрично

```csharp
public int CompareTo(Person? other)
{
    return Name.Length - other.Name.Length;  // ⚠️ может overflow!
}

// "a" длина 1, name длина MAXINT → MAXINT - 1 = OK
// "a" длина 1, name длина -INT_MAX (impossible но в general case)

// ✅ Используй CompareTo вместо subtract
public int CompareTo(Person? other) => 
    Name.Length.CompareTo(other.Name.Length);
```

### 6. == operator на null

```csharp
public class Money
{
    public static bool operator ==(Money a, Money b)
    {
        return a.Amount == b.Amount;  // ⚠️ NullReferenceException если a null!
    }
}

// ✅
public static bool operator ==(Money? a, Money? b)
{
    if (a is null) return b is null;
    return a.Equals(b);
}
```

### 7. Equality в LINQ

```csharp
// LINQ Distinct, Except, Contains, etc — uses EqualityComparer<T>.Default

class Person { /* без IEquatable */ }

var people = new[] { 
    new Person { Name = "John" },
    new Person { Name = "John" }  // same data
};

people.Distinct().Count();  // 2 — different references!

// ✅ Custom comparer
people.Distinct(new PersonNameComparer()).Count();  // 1

// Или используй record:
record Person(string Name);
people.Distinct().Count();  // 1 ✅
```

### 8. Inheritance equality breaks symmetric

См. секцию **3. Equals правила** выше.

### 9. Boxing в struct equality

```csharp
struct Point { public int X, Y; }

Point p = new() { X = 1, Y = 2 };
object o = p;  // boxed!
p.Equals(o);   // OK но allocation для boxing

// ✅ IEquatable<Point> + Equals(Point)
struct Point : IEquatable<Point> { ... public bool Equals(Point other) }
p.Equals(p2);  // direct call, no boxing
```

### 10. GetHashCode в derived classes

```csharp
class Base
{
    public string Name;
    public override int GetHashCode() => Name.GetHashCode();
}

class Derived : Base
{
    public int Age;
    public override int GetHashCode() => Age.GetHashCode();  // ❌ ignores Name!
}

var d = new Derived { Name = "John", Age = 30 };
// hash только по Age — collision с другими Derived с тем же Age
```

**Лечение:** include base hash:

```csharp
public override int GetHashCode() => HashCode.Combine(base.GetHashCode(), Age);
```

---

## 13. Best Practices

### Equality

- **Records для value objects** — auto-generated всё
- **`IEquatable<T>`** для классов в коллекциях (avoid boxing)
- **Override Equals + GetHashCode together** (или используй record)
- **`HashCode.Combine`** для hash
- **Immutable fields** в hash — никаких setters
- **Operator ==** — overload вместе с Equals для public types

### Comparison

- **`IComparable<T>`** для natural ordering
- **`IComparer<T>`** для custom sort criteria
- **LINQ OrderBy** для simple cases
- **Multiple criteria** через ThenBy

### Strings

- **Always specify StringComparison**
- **`StringComparer.Ordinal`** для internal data
- **`StringComparer.CurrentCulture`** для display

### Common patterns

```csharp
// Value object
public record Money(decimal Amount, string Currency);

// Entity (identity-based equality)
public class User
{
    public Guid Id { get; }

    public override bool Equals(object? obj) => 
        obj is User u && Id == u.Id;

    public override int GetHashCode() => Id.GetHashCode();
}

// Comparison-aware
public record Version(int Major, int Minor) : IComparable<Version>
{
    public int CompareTo(Version? other)
    {
        if (other is null) return 1;
        var c = Major.CompareTo(other.Major);
        return c != 0 ? c : Minor.CompareTo(other.Minor);
    }
}
```

---

## 14. Cheat sheet

| Сценарий | Solution |
|----------|----------|
| Value equality для DTO | `record` |
| Value equality для struct | `record struct` |
| Custom equality на class | `IEquatable<T>` + override Equals/GHC |
| Sort sequence | `IComparable<T>` |
| Custom sort criteria | `IComparer<T>` или `OrderBy(x => ...)` |
| Equality в Dictionary/HashSet | `IEqualityComparer<T>` или type implements IEquatable |
| Case-insensitive strings | `StringComparer.OrdinalIgnoreCase` |
| Hash combine | `HashCode.Combine(field1, field2, ...)` |
| Compare floats | `Math.Abs(a - b) < epsilon` |
| Compare safely with null | `is null` checks first |
| Reference check | `ReferenceEquals(a, b)` |

---

## 15. Decision tree

```
Нужна equality?
│
├── DTO / Value Object?
│   → record / record struct
│
├── Entity (identity-based)?
│   → Equals + GetHashCode по Id
│
├── Custom class в HashSet/Dictionary?
│   → IEquatable<T> + override Equals/GHC
│
└── Compare strings?
    → StringComparison.Ordinal/IgnoreCase

Нужна comparison?
│
├── Natural order (Version, Date)?
│   → IComparable<T>
│
├── Multiple sort criteria?
│   → OrderBy.ThenBy
│
└── External comparer?
    → IComparer<T>

Нужна custom equality в collection?
│
└── IEqualityComparer<T> + pass в HashSet/Dictionary ctor
```

---

## См. также

- [[csharp-basics|C# Basics]] — == operators intro
- [[oop|OOP]] — Equals в inheritance
- [[functional-csharp|Functional C#]] — records value semantics
- [[generics-deep|Generics Deep]] — EqualityComparer\<T\>.Default
- [[collections-linq|Collections и LINQ]] — Distinct, GroupBy используют equality
- [[strings-regex|Strings и Regex]] — culture в comparison
- [[modern-features|Modern Features]] — records

## Reading list

- **Microsoft Docs — Equals method** — learn.microsoft.com/dotnet/api/system.object.equals
- **Microsoft Docs — IEquatable\<T\>** — learn.microsoft.com/dotnet/api/system.iequatable-1
- **Microsoft Docs — IComparable\<T\>** — learn.microsoft.com/dotnet/api/system.icomparable-1
- **Eric Lippert — Equality and identity** — ericlippert.com
- **Jon Skeet — Equals and GetHashCode** — codeblog.jonskeet.uk
- **Effective C#** — Bill Wagner (item про equality)
- **CLR via C#** — Jeffrey Richter (chapter про reference identity)
