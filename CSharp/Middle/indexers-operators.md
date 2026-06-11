---
tags: [csharp, indexers, operators, middle, operator-overloading, this-indexer]
level: Middle
date: 2026-05-07
---

# Indexers и Operator Overloading

> **Объект ведёт себя как массив или встроенный тип.** `this[index]` indexer + operator overloading (`+`, `-`, `==`, `>`, conversion). Когда уместно, когда anti-pattern. Закрывает пробел: «знаю про `dict[key]`, не понимаю когда писать свой `this[]` и зачем переопределять `+` для Money».

---

## 0. Как читать

Если впервые видишь indexer — раздел 1→3. Если уже писал operator overload, но непонятно implicit/explicit — раздел 6. Production guidance — раздел 8 (best practices), 10 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Indexer — `this[]` как член класса

```csharp
public class WordList
{
    private readonly List<string> _words = [];
    
    public string this[int index]   // indexer
    {
        get => _words[index];
        set => _words[index] = value;
    }
}

var list = new WordList();
list[0] = "hello";   // setter
var w = list[0];      // getter
```

Indexer позволяет объекту **вести себя как массив**. Под капотом — методы `get_Item(int)` / `set_Item(int, T)`.

### 1.2. Operator overloading — custom поведение для `+`, `-`, etc.

```csharp
public readonly struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal amount, string currency) { Amount = amount; Currency = currency; }
    
    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency) throw new InvalidOperationException();
        return new Money(a.Amount + b.Amount, a.Currency);
    }
}

var sum = new Money(100, "USD") + new Money(50, "USD");
```

### 1.3. Когда использовать

```
Indexer:
  ✅ Collection-like класс (List, Dictionary wrapper)
  ✅ Multi-dimensional access (Matrix[r, c])
  ✅ String-key lookup (Configuration["key"])
  ❌ Method который принимает один параметр — лучше named method

Operator overload:
  ✅ Math types (Money, Vector, Matrix, Complex)
  ✅ Domain types где `+` natural (TimeSpan, DateTime)
  ✅ Comparison для sortable types (Version)
  ❌ Бизнес-логика — `user1 + user2` confuses
  ❌ Performance-impossible operations
```

### 1.4. Главное правило

```
Indexers — для collection/dictionary semantics.
Operators — для math-like / value-object types.

Если читатель кода не догадается без documentation — НЕ overload.

Records и primitive value types — основные кандидаты на operators.
```

> [!info]- Если ты знаешь Python / C++ / Kotlin / Rust
> **Python:** `__getitem__(self, key)` ↔ indexer; magic methods `__add__`, `__sub__` ↔ operator overload. Очень похоже.
>
> **C++:** `operator[]` для indexer, `operator+`, `operator==` etc. — direct equivalents. C# borrowed syntax из C++.
>
> **Kotlin:** `operator fun get(index)` ↔ indexer; `operator fun plus(other)` ↔ operator overload. Compile-time pattern.
>
> **Rust:** `Index` trait для `[]`, `Add`/`Sub` traits для arithmetic. Очень similar concept, traits вместо `operator` keyword.

> [!question]- Интервью: что такое indexer в C#?
> Special property с syntax `this[parameters]` — позволяет объекту использоваться как массив (`obj[index]`). Под капотом: компилятор генерирует методы `get_Item(...)` / `set_Item(..., value)`. Может принимать любые параметры (int, string, multi-key). Имеет accessors `get`/`set`/`init`. Используется для collection wrappers, dictionaries, multi-dimensional arrays, string-key configurations. Не путай с indexer Property — обычное property не имеет `this`.

---

## 2. Indexer basics

### 2.1. Простой indexer

```csharp
public class StringCollection
{
    private readonly List<string> _items = [];
    
    public string this[int index]
    {
        get => _items[index];
        set => _items[index] = value;
    }
    
    public int Count => _items.Count;
    public void Add(string item) => _items.Add(item);
}

var c = new StringCollection();
c.Add("hello");
c[0] = "world";
Console.WriteLine(c[0]);
```

### 2.2. Множественные параметры

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
```

### 2.3. Non-int параметр

```csharp
public class Settings
{
    private readonly Dictionary<string, string> _data = new();
    
    public string this[string key]
    {
        get => _data.TryGetValue(key, out var v) ? v : "";
        set => _data[key] = value;
    }
}

var s = new Settings();
s["theme"] = "dark";
var t = s["theme"];
```

### 2.4. Read-only indexer

```csharp
public class ReadOnlyList<T>
{
    private readonly T[] _items;
    
    public ReadOnlyList(T[] items) => _items = items;
    
    public T this[int index] => _items[index];   // только getter
}
```

Без setter — read-only. Caller не может assign.

### 2.5. Init-only indexer (C# 9+)

```csharp
public class Builder
{
    private readonly Dictionary<string, string> _data = new();
    
    public string this[string key]
    {
        get => _data[key];
        init => _data[key] = value;   // только в object initializer
    }
}

var b = new Builder { ["a"] = "1", ["b"] = "2" };
// b["c"] = "3";   // ❌ — init only
```

### 2.6. Multiple indexers — overloading

```csharp
public class Library
{
    public Book this[int id] => /* ... */;       // by id
    public Book this[string title] => /* ... */; // by title
    public List<Book> this[Author author] => /* ... */;   // by author
}
```

Overload по типу параметров. Compiler resolves по argument type.

### 2.7. Indexer vs method

```csharp
// Indexer — collection-like
public string this[int index] => _items[index];

// Method — explicit
public string GetByIndex(int index) => _items[index];
```

Indexer — когда object **семантически коллекция**. Method — когда retrieval имеет other meaning (computed, side effects).

### 2.8. Compiled metadata

```csharp
// Indexer компилируется в:
public string get_Item(int index) { /* ... */ }
public void set_Item(int index, string value) { /* ... */ }
[IndexerName("Item")]   // default name
```

Имя `Item` — convention. `IndexerName` attribute меняет (используется в BCL для backward compat).

> [!question]- Интервью: чем indexer отличается от обычного property?
> 1) **Параметры** — indexer принимает arguments (`this[int]`), property — nullary. 2) **Syntax** — `obj[args]` vs `obj.Property`. 3) **Multiple overloads** — indexer может иметь несколько overloads по типам параметров. 4) **`this` keyword** в declaration. 5) **Name** — компилируется как `Item` (или through `[IndexerName]` attribute). 6) **Use cases** — indexer для collection/dictionary semantics, property для object state. Pattern matching и LINQ работают с indexers через `IList<T>.this[int]`.

---

## 3. Indexer patterns

### 3.1. Cache wrapper

```csharp
public class MemoryCache<TKey, TValue> where TKey : notnull
{
    private readonly Dictionary<TKey, TValue> _store = new();
    
    public TValue this[TKey key]
    {
        get
        {
            if (!_store.TryGetValue(key, out var v))
                throw new KeyNotFoundException(key.ToString());
            return v;
        }
        set => _store[key] = value;
    }
    
    public bool TryGet(TKey key, out TValue? value) =>
        _store.TryGetValue(key, out value);
}
```

### 3.2. Lazy materialization

```csharp
public class LazyList<T>
{
    private readonly Func<int, T> _factory;
    private readonly Dictionary<int, T> _cache = new();
    
    public LazyList(Func<int, T> factory) => _factory = factory;
    
    public T this[int index]
    {
        get
        {
            if (!_cache.TryGetValue(index, out var v))
                _cache[index] = v = _factory(index);
            return v;
        }
    }
}
```

### 3.3. Multi-dimensional accessor

```csharp
public class Grid<T>
{
    private readonly T[,] _cells;
    public int Rows { get; }
    public int Cols { get; }
    
    public Grid(int rows, int cols)
    {
        Rows = rows; Cols = cols;
        _cells = new T[rows, cols];
    }
    
    public T this[int row, int col]
    {
        get => _cells[row, col];
        set => _cells[row, col] = value;
    }
}
```

### 3.4. Range и индексы (C# 8+)

```csharp
public class Buffer
{
    private readonly byte[] _data;
    
    public Buffer(int size) => _data = new byte[size];
    
    public byte this[int index] => _data[index];
    public byte this[Index index] => _data[index];
    public byte[] this[Range range] => _data[range];
}

var buf = new Buffer(10);
var b = buf[5];
var last = buf[^1];          // through Index
var slice = buf[2..5];       // through Range
```

`Index`/`Range` — built-in для ^N (from end) и slicing.

### 3.5. Validation в indexer

```csharp
public class ValidatedList
{
    private readonly List<string> _items = [];
    
    public string this[int index]
    {
        get
        {
            if (index < 0 || index >= _items.Count)
                throw new IndexOutOfRangeException();
            return _items[index];
        }
        set
        {
            if (string.IsNullOrEmpty(value))
                throw new ArgumentException("Empty value not allowed");
            _items[index] = value;
        }
    }
}
```

### 3.6. Async indexer? — нет

```csharp
public Task<T> this[int index] => /* ... */;   // ⚠ можно но usually anti-pattern
```

Indexer должен быть fast. Для async — explicit `GetAsync(int)` method.

> [!question]- Интервью: можно ли иметь несколько indexers в одном классе?
> Да — overload по типам параметров. Например, `Library` может иметь `this[int id]`, `this[string title]`, `this[Author author]`. Compiler resolves по argument type. Нельзя иметь два indexers с identical signature. Также можно overload по count параметров (`this[int]` и `this[int, int]`). Pattern из BCL: `string` имеет `this[int]` и `this[Index]` (C# 8+).

---

## 4. Operator overloading basics

### 4.1. Какие operators можно overload

```
Unary:    + - ! ~ ++ -- true false
Binary:   + - * / % & | ^ << >>
Comparison: == != < > <= >=
Conversion: (T) implicit, explicit
```

**Нельзя**: `=`, `&&`, `||`, `??`, `??=`, `?:`, `()`, `[]` (через indexer), member access.

### 4.2. Базовый синтаксис

```csharp
public readonly struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }
    
    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException("Currency mismatch");
        return new Money(a.Amount + b.Amount, a.Currency);
    }
    
    public static Money operator -(Money a, Money b)
    {
        if (a.Currency != b.Currency)
            throw new InvalidOperationException("Currency mismatch");
        return new Money(a.Amount - b.Amount, a.Currency);
    }
    
    public static Money operator *(Money m, decimal multiplier) =>
        new(m.Amount * multiplier, m.Currency);
}

var m1 = new Money(100, "USD");
var m2 = new Money(50, "USD");
var sum = m1 + m2;            // 150 USD
var doubled = m1 * 2;          // 200 USD
```

### 4.3. Operators must be `static`

```csharp
public static Money operator +(Money a, Money b);   // ✅ static
// public Money operator +(Money other);           // ❌ Compile error
```

Operators — statics. Левый и правый operands передаются как parameters.

### 4.4. Pairs обязательны

```csharp
// Если override == ОБЯЗАТЕЛЬНО override !=
public static bool operator ==(Money a, Money b) => /* ... */;
public static bool operator !=(Money a, Money b) => !(a == b);

// > и < — pair
// <= и >= — pair
```

Pairs:
- `==` / `!=`
- `<` / `>`
- `<=` / `>=`
- `true` / `false`

### 4.5. Implicit Equals + GetHashCode

```csharp
public static bool operator ==(Money a, Money b) => /* ... */;
// Также override Equals и GetHashCode!
public override bool Equals(object? obj) => /* ... */;
public override int GetHashCode() => /* ... */;
```

См. [[equality-comparison|Equality]] раздел 5.

> [!question]- Интервью: какие операторы нельзя overload?
> `=` (assignment — special), `&&`/`||` (short-circuit logical — overloadable indirectly через `true`/`false` operators), `??` / `??=` / `?:` (null/conditional), `()` (function call — нет в C#), `[]` (через indexer), `.` (member access), `=>` (arrow). Compound operators (`+=`, `-=`) auto-derived из binary (`+`, `-`). Все остальные binary, unary, comparison, conversion — overloadable.

---

## 5. Comparison operators

### 5.1. < > <= >= pair

```csharp
public sealed class Version : IComparable<Version>
{
    public int Major, Minor, Patch;
    
    public int CompareTo(Version? other) { /* ... */ }
    
    public static bool operator <(Version? a, Version? b) =>
        Comparer<Version>.Default.Compare(a, b) < 0;
    public static bool operator >(Version? a, Version? b) =>
        Comparer<Version>.Default.Compare(a, b) > 0;
    public static bool operator <=(Version? a, Version? b) =>
        Comparer<Version>.Default.Compare(a, b) <= 0;
    public static bool operator >=(Version? a, Version? b) =>
        Comparer<Version>.Default.Compare(a, b) >= 0;
}
```

`Comparer<T>.Default.Compare(a, b)` handles null safely — нет NRE.

### 5.2. == через Equals

```csharp
public sealed class Money : IEquatable<Money>
{
    public bool Equals(Money? other) => /* ... */;
    public override bool Equals(object? obj) => Equals(obj as Money);
    public override int GetHashCode() => /* ... */;
    
    public static bool operator ==(Money? a, Money? b)
    {
        if (a is null) return b is null;
        return a.Equals(b);
    }
    public static bool operator !=(Money? a, Money? b) => !(a == b);
}
```

### 5.3. Records — automatic

```csharp
public record User(int Id, string Name);
// Compiler генерирует:
//   == и != (через value equality)
//   <, >, <=, >= НЕ генерирует (records не natural-ordered)
```

### 5.4. IComparisonOperators (.NET 7+)

```csharp
public interface IComparisonOperators<TSelf, TOther, TResult>
    where TSelf : IComparisonOperators<TSelf, TOther, TResult>?
{
    static abstract TResult operator <(TSelf left, TOther right);
    static abstract TResult operator >(TSelf left, TOther right);
    // ...
}
```

Generic Math interfaces для perf-critical numeric code.

> [!question]- Интервью: должны ли `==` и `!=` operators returns совпадать с `Equals`?
> Да — это **contract**. `a == b` должен = `a.Equals(b)` (для same type). Иначе collections поломаются (HashSet, Dictionary lookup используют Equals, но developer ожидает == match). Standard pattern: implement `Equals(object)`, `Equals(T)`, `GetHashCode`, потом `==` через `Equals`. Records делают это автоматически. Microsoft Framework Design Guidelines требует.

---

## 6. Conversion operators

### 6.1. Implicit vs explicit

```csharp
public readonly struct Celsius
{
    public double Value { get; }
    public Celsius(double v) => Value = v;
    
    // Implicit — compiler сам конвертирует, безопасный
    public static implicit operator double(Celsius c) => c.Value;
    
    // Explicit — нужен cast `(Celsius)`, риск information loss
    public static explicit operator Celsius(double d) => new(d);
}

Celsius temp = new(25);
double t = temp;          // implicit — OK
Celsius t2 = (Celsius)25; // explicit — нужен cast
```

### 6.2. Когда implicit

✅ **Implicit когда:**
- Конверсия безопасна (no info loss).
- Нет throw возможен.
- Семантически естественно (Celsius → double — value).

```csharp
// String → ReadOnlySpan<char>
public static implicit operator ReadOnlySpan<char>(string s) => s.AsSpan();
```

### 6.3. Когда explicit

✅ **Explicit когда:**
- Возможна потеря информации (`double → int`).
- Throw возможен (invalid input).
- Семантическая разница important (degrees → radians).

```csharp
// double → int — может потерять precision
int x = (int)3.7;   // 3, лишняя дробная часть отброшена
```

### 6.4. Записи в обе стороны

```csharp
public readonly struct Temperature
{
    public double Celsius { get; }
    public Temperature(double c) => Celsius = c;
    
    public static implicit operator double(Temperature t) => t.Celsius;
    public static explicit operator Temperature(double d) => new(d);
}
```

### 6.5. Ограничения

- Нельзя override `=` (assignment).
- Нельзя iz/to interface (только classes/structs).
- Нельзя iz/to base class (special-cased).

### 6.6. ToString — не conversion operator

```csharp
public override string ToString() => /* ... */;   // virtual метод, не operator
```

`ToString()` — не conversion operator. `Object.ToString()` virtual метод, override как обычно.

> [!question]- Интервью: чем implicit conversion отличается от explicit?
> **Implicit** — compiler конвертирует автоматически (`double t = celsius;`). Используется когда: безопасно, нет info loss, нет throw. **Explicit** — caller обязан написать cast (`(Celsius)25`). Используется когда: возможна потеря данных (double → int), throw возможен, semantic conversion (degrees ↔ radians). Best practice: implicit для widening (subset → superset), explicit для narrowing.

---

## 7. true и false operators

### 7.1. Зачем

Позволяет тип использоваться в boolean context (`if`, `while`):

```csharp
public readonly struct Maybe<T>
{
    private readonly T? _value;
    private readonly bool _hasValue;
    
    public Maybe(T value) { _value = value; _hasValue = true; }
    
    public static bool operator true(Maybe<T> m) => m._hasValue;
    public static bool operator false(Maybe<T> m) => !m._hasValue;
}

Maybe<int> m = new(42);
if (m) Console.WriteLine("Has value");   // works через operator true
```

### 7.2. С `&&` и `||`

```csharp
// Если есть operator true/false, можно overload & и |
public static Maybe<T> operator &(Maybe<T> a, Maybe<T> b) => /* ... */;
public static Maybe<T> operator |(Maybe<T> a, Maybe<T> b) => /* ... */;

var combined = a && b;   // compiler использует true/false + & operators
```

Short-circuit `&&`/`||` работают через `true`/`false` operators.

### 7.3. Когда уместно

Очень редко. Idiom: `Maybe<T>`, `Option<T>` (custom), iZ-checking types. Большинство кода без этого обходится — boolean expressions explicit.

> [!question]- Интервью: зачем `operator true` и `operator false`?
> Позволяют типу использоваться в boolean context (`if (obj)`, `while (obj)`). Также enable overload `&&`/`||` operators через `true`/`false` + `&`/`|` (short-circuit). Использование редкое — большинство C# кода explicit `if (obj.IsValid)`. Idiom для optional types (`Maybe<T>`, `Option<T>`). В BCL — `DBNull` и nullable bool. Microsoft FDG не рекомендует — confusing.

---

## 8. Best Practices

### 8.1. Indexers

- ✅ Для collection / dictionary semantics.
- ✅ `IndexOutOfRangeException` для invalid int index.
- ✅ `KeyNotFoundException` для missing string key.
- ✅ Validation в setter.
- ✅ Range/Index support (C# 8+).
- ❌ Async indexer — anti-pattern.
- ❌ Side effects в getter.

### 8.2. Operator overloading

- ✅ **Math types** (Money, Vector, Coordinate).
- ✅ **Pairs** — `==`/`!=`, `<`/`>`, `<=`/`>=`.
- ✅ **`Equals` + `GetHashCode`** при `==` overload.
- ✅ **Records** для simple value-based equality.
- ❌ **Бизнес-логика** — `user1 + user2` confusing.
- ❌ **Throw в operator** — surprise (но иногда OK для invalid input).
- ❌ **Asymmetric operators** — `a + b != b + a` без обоснования.

### 8.3. Conversion operators

- ✅ **Implicit** для safe, info-preserving conversions.
- ✅ **Explicit** для narrowing, lossy, semantic conversions.
- ❌ **Implicit с info loss** — surprise narrowing.
- ❌ **Conversion из/в multiple types** — confusing.

### 8.4. Не делай

- ❌ Operator overload где не очевидно.
- ❌ Indexer для random side effects.
- ❌ `+` для concatenation если нет clear meaning.
- ❌ Implicit conversion которая может throw.

---

## 9. Decision tree

```
Что нужно?
│
├── Object вёл себя как collection / dictionary
│   └── Indexer this[...]
│       ├── int index → array-like
│       ├── string key → dictionary-like
│       ├── multi-dim → matrix-like
│       ├── Index/Range → slicing (C# 8+)
│       └── overloads — multiple lookup strategies
│
├── Math operations над domain type
│   └── Operator + / - / * /
│       ├── Pairs обязательны (== / !=, < / >)
│       ├── + Equals + GetHashCode
│       └── Records — automatic для == / !=
│
├── Тип может быть convert to/from другому
│   ├── Безопасно, no loss → implicit
│   ├── Lossy / throws → explicit
│   └── В обе стороны — оба operators
│
└── Boolean-like type → operator true / false
    └── Очень редко
```

---

## 10. Cheat sheet

```csharp
// === Indexer ===
public class MyList<T>
{
    private readonly List<T> _items = [];
    
    public T this[int i]
    {
        get => _items[i];
        set => _items[i] = value;
    }
    
    public T this[Index i] => _items[i];
    public List<T> this[Range r] => [.. _items[r]];
}

// === Operators ===
public readonly struct Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }
    
    public Money(decimal a, string c) { Amount = a; Currency = c; }
    
    // Arithmetic
    public static Money operator +(Money a, Money b) => /* ... */;
    public static Money operator -(Money a, Money b) => /* ... */;
    public static Money operator *(Money m, decimal x) => new(m.Amount * x, m.Currency);
    public static Money operator -(Money m) => new(-m.Amount, m.Currency);   // unary
    
    // Equality
    public bool Equals(Money other) => Amount == other.Amount && Currency == other.Currency;
    public override bool Equals(object? obj) => obj is Money m && Equals(m);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    public static bool operator ==(Money a, Money b) => a.Equals(b);
    public static bool operator !=(Money a, Money b) => !(a == b);
    
    // Conversion
    public static implicit operator decimal(Money m) => m.Amount;
    public static explicit operator Money(decimal d) => new(d, "USD");
}
```

---

## 11. Common Pitfalls

### 11.1. Operator overload без Equals

```csharp
public static bool operator ==(Money a, Money b) => /* ... */;
// ❌ нет override Equals — collections ломаются
```

**Фикс:** `==` через `Equals`, override Equals + GetHashCode.

### 11.2. Implicit conversion с loss

```csharp
public static implicit operator int(BigDecimal b) => (int)b.Value;   // ❌ loss possible
```

**Фикс:** explicit для narrowing.

### 11.3. Asymmetric operators

```csharp
public static Money operator +(Money m, decimal d);   // OK
public static Money operator +(decimal d, Money m);   // ❌ забыл — m + 5 OK, 5 + m fail
```

**Фикс:** define both directions.

### 11.4. Indexer throw в getter

```csharp
public T this[int i] => i >= Count ? throw new InvalidOperationException() : _items[i];
```

**Механизм:** confusing — обычно `IndexOutOfRangeException` ожидается.

**Фикс:** `IndexOutOfRangeException` или `ArgumentOutOfRangeException`.

### 11.5. Operator overload для confusing semantics

```csharp
public static User operator +(User a, User b);   // ❌ что значит "сумма users"?
```

**Фикс:** explicit method `Merge(User a, User b)`.

### 11.6. `==` reference comparison surprise

```csharp
public class User { public int Id; }
var u1 = new User { Id = 1 };
var u2 = new User { Id = 1 };
u1 == u2;   // false — reference, не Id!
```

**Фикс:** override `==` или используй record.

### 11.7. Operator throws в hot path

```csharp
public static Money operator +(Money a, Money b)
{
    if (a.Currency != b.Currency) throw new InvalidOperationException();   // expensive
    /* ... */
}
```

**Фикс:** validate данные раньше, перед operator вызовом.

### 11.8. Indexer без validation в setter

```csharp
public string this[int i] { set => _items[i] = value; }   // null OK?
```

**Фикс:** explicit validation.

### 11.9. Implicit conversion + nullable surprises

```csharp
public static implicit operator string(MyType m) => m.ToString();
MyType? x = null;
string s = x;   // ⚠ NRE
```

**Фикс:** explicit или null check.

### 11.10. `a += b` через operator + creates copy

```csharp
public class List
{
    public static List operator +(List a, List b) => /* new List */;
}

var l = new List();
l += otherList;   // создаёт NEW list (не mutates)
```

**Фикс:** use mutable methods (`.Add`, `.AddRange`) для in-place.

> [!question]- Интервью: топ-3 ошибки с operator overload?
> 1) **Override `==` без Equals/GetHashCode** — collections (HashSet, Dictionary) ломаются. Always pair operators с Equals/GetHashCode (или record). 2) **Implicit conversion с info loss** — `decimal → int` через implicit surprises caller. Use explicit для narrowing. 3) **Asymmetric arithmetic** — `Money + decimal` defined, но `decimal + Money` нет → user confused. Define both directions для commutative operators.

---

## 12. Practice exercises

### 12.1. Complex number type

```csharp
public readonly record struct Complex(double Real, double Imaginary)
{
    public static Complex operator +(Complex a, Complex b) =>
        new(a.Real + b.Real, a.Imaginary + b.Imaginary);
    
    public static Complex operator -(Complex a, Complex b) =>
        new(a.Real - b.Real, a.Imaginary - b.Imaginary);
    
    public static Complex operator *(Complex a, Complex b) =>
        new(a.Real * b.Real - a.Imaginary * b.Imaginary,
            a.Real * b.Imaginary + a.Imaginary * b.Real);
    
    public double Magnitude => Math.Sqrt(Real * Real + Imaginary * Imaginary);
    
    public static explicit operator double(Complex c) => c.Magnitude;
}

var c1 = new Complex(3, 4);
var c2 = new Complex(1, 2);
var sum = c1 + c2;   // (4, 6)
double m = (double)c1;   // 5 — magnitude
```

### 12.2. Multi-dimensional indexer

```csharp
public class Grid<T>
{
    private readonly T[,] _data;
    public int Rows { get; }
    public int Cols { get; }
    
    public Grid(int rows, int cols)
    {
        Rows = rows; Cols = cols;
        _data = new T[rows, cols];
    }
    
    public T this[int row, int col]
    {
        get
        {
            if (row < 0 || row >= Rows) throw new ArgumentOutOfRangeException(nameof(row));
            if (col < 0 || col >= Cols) throw new ArgumentOutOfRangeException(nameof(col));
            return _data[row, col];
        }
        set
        {
            if (row < 0 || row >= Rows) throw new ArgumentOutOfRangeException(nameof(row));
            if (col < 0 || col >= Cols) throw new ArgumentOutOfRangeException(nameof(col));
            _data[row, col] = value;
        }
    }
}
```

### 12.3. Type-safe wrapper с conversions

```csharp
public readonly struct UserId : IEquatable<UserId>
{
    public int Value { get; }
    public UserId(int v) { Value = v; }
    
    public static implicit operator int(UserId id) => id.Value;
    public static explicit operator UserId(int v) => new(v);
    
    public bool Equals(UserId other) => Value == other.Value;
    public override bool Equals(object? obj) => obj is UserId u && Equals(u);
    public override int GetHashCode() => Value.GetHashCode();
    public static bool operator ==(UserId a, UserId b) => a.Equals(b);
    public static bool operator !=(UserId a, UserId b) => !(a == b);
}

UserId id = (UserId)42;   // explicit construct
int n = id;                // implicit unwrap
```

---

## 13. Что читать дальше

1. **[[equality-comparison|Equality]]** — operator `==` semantics.
2. **[[oop|OOP]]** — records auto-operators.
3. **[[generics-deep|Generics]]** — generic operator constraints (.NET 7+).
4. **Generic Math (.NET 7+)** — `IAdditionOperators`, `IComparisonOperators`.

---

## 14. См. также

- [[equality-comparison|Equality]] — `==` через Equals
- [[oop|OOP]] — records
- [[modern-features|Modern Features]] — primary constructors + operators
- BCL: `int`, `decimal`, `DateTime` — operator examples
- `IAdditionOperators<T>` (.NET 7+)
- `IComparisonOperators<T>` (.NET 7+)

---

## 15. Reading list

- **Microsoft Docs — Operator overloading** — learn.microsoft.com/dotnet/csharp/language-reference/operators/operator-overloading
- **Microsoft Docs — Indexers** — learn.microsoft.com/dotnet/csharp/programming-guide/indexers/
- **Microsoft Docs — User-defined conversion operators** — learn.microsoft.com/dotnet/csharp/language-reference/operators/user-defined-conversion-operators
- **Microsoft FDG — Operator overload guidelines** — learn.microsoft.com/dotnet/standard/design-guidelines/operator-overloads
- **Bill Wagner — Effective C# (operator chapter)**
- **Jon Skeet — C# in Depth** — operators chapter
