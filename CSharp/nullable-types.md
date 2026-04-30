---
tags: [csharp, nullable, null-safety, nrt, junior, middle]
level: Middle
date: 2026-04-30
---

# Nullable Types и Null Safety

> **Полный гайд по null в C#**: Nullable\<T\> для value types, Nullable Reference Types (NRT), null operators, как избегать NullReferenceException, как включать NRT в legacy коде, attributes для аннотации.

---

## Что это, зачем и когда

### Что такое null

**`null`** — отсутствие значения. "Здесь ничего нет."

```csharp
string? name = null;        // переменная есть, но значения нет
User? user = GetUser();      // может вернуть null если не нашёл
```

### Проблема: NullReferenceException

```csharp
string s = null;
int len = s.Length;  // 💥 NullReferenceException!
```

> **The Billion Dollar Mistake** — Tony Hoare (создатель null) назвал введение null своей самой большой ошибкой. Это причина 30%+ всех bugs в production.

### Решения в C#

```
C# 1.0 — 7.x:
  - null может быть в любой reference type
  - Никаких compile-time проверок
  - NRE — runtime crash

C# 8.0+ (Nullable Reference Types):
  - Reference types — non-nullable by default
  - string? — nullable, string — non-nullable
  - Compiler warns при потенциальных null deref
  - Не runtime enforcement — только compile hints
```

---

## 1. Nullable\<T\> — для value types

Value types (`int`, `bool`, `DateTime`) **не могут быть null** by default:

```csharp
int x = null;     // ❌ Compile error
bool b = null;    // ❌
```

Чтобы разрешить null — `Nullable<T>` или сокращение `T?`:

```csharp
int? x = null;            // OK
Nullable<int> y = null;   // то же самое

bool? isReady = null;
DateTime? lastLogin = null;
```

### Использование

```csharp
int? age = null;

// Check
if (age.HasValue)
{
    int actualAge = age.Value;
    // или
    int actualAge = age.Value;
}

// Pattern matching (C# 8+)
if (age is int actualAge)
{
    Console.WriteLine(actualAge);
}

// Default value
int actualAge = age ?? 0;        // null coalescing
int actualAge = age.GetValueOrDefault();  // 0 если null
int actualAge = age.GetValueOrDefault(18); // 18 если null
```

### Operations с Nullable

```csharp
int? a = 5;
int? b = null;

int? sum = a + b;       // null (lifted operator — null propagates)
int? c = a + 1;         // 6
bool eq = a == b;       // false
bool eq2 = a == 5;      // true

// Comparison с null:
a < null;               // false (всегда false для < > <= >=)
a == null;              // false
```

### Под капотом

```csharp
// Nullable<T> — это struct
public struct Nullable<T> where T : struct
{
    public bool HasValue { get; }
    public T Value { get; }
    // ...
}

// int? — syntactic sugar для Nullable<int>
int? x = 5;          // эквивалентно
Nullable<int> y = 5;
```

### Boxing nullable

```csharp
int? a = 5;
object boxed = a;        // boxes как int (5), не как Nullable<int>!

int? b = null;
object boxed2 = b;       // boxes как null reference

if (boxed is int unboxed) { /* ... */ }  // OK
```

---

## 2. Nullable Reference Types (NRT) — C# 8+

### Включение

```xml
<!-- csproj -->
<PropertyGroup>
  <Nullable>enable</Nullable>
</PropertyGroup>
```

Или per-file:
```csharp
#nullable enable
```

После включения:
- `string` — **non-nullable** (warning при null assignment)
- `string?` — **nullable** (явно разрешает null)

### Что меняется

```csharp
#nullable enable

string name = null;         // ⚠️ CS8600: Cannot convert null to non-nullable
string? maybeNull = null;   // ✅ OK

// Method signatures
string GetName() { return null; }       // ⚠️ Warning
string? GetNameMaybe() { return null; } // ✅ OK

// Parameters
void Process(string s)        // s — non-nullable
void Process(string? s)       // s — может быть null
```

### Использование nullable

```csharp
void PrintLength(string? s)
{
    Console.WriteLine(s.Length);  // ⚠️ Warning: possible null

    if (s is not null)
    {
        Console.WriteLine(s.Length);  // ✅ Compiler знает не null
    }

    Console.WriteLine(s?.Length);  // ✅ null-conditional
    Console.WriteLine(s?.Length ?? 0);  // ✅
    Console.WriteLine(s!.Length);  // ✅ "trust me, not null"
}
```

### Flow analysis

Компилятор отслеживает nullability через control flow:

```csharp
void Process(string? s)
{
    if (s is null) return;
    Console.WriteLine(s.Length);  // ✅ Compiler knows s != null после if-return

    // Или через guard
    s ??= "default";
    Console.WriteLine(s.Length);  // ✅ guaranteed not null
}

string? GetUser() { return null; }

void Use()
{
    var user = GetUser();
    if (user != null && user.IsActive)
    {
        Console.WriteLine(user.Name);  // ✅ short-circuit OK
    }
}
```

### NRT — это **warnings**, не errors

```csharp
string s = null;  // CS8600 — warning, code compiles!
Console.WriteLine(s.Length);  // CS8602 — warning, runtime NRE при run!
```

NRT — **compile-time hints**. Runtime по-прежнему может получить null.

> [!info] Warnings as errors
> В production включи `<TreatWarningsAsErrors>true</TreatWarningsAsErrors>` чтобы NRT warnings блокировали build.

См. [[../Quality/static-analysis|Static Analysis]].

---

## 3. Null operators

### `?.` — null-conditional access

```csharp
string? name = GetName();

// Без оператора — много if-ов
if (name != null)
{
    int len = name.Length;
    Console.WriteLine(len);
}

// С ?.
int? len = name?.Length;  // null если name null
Console.WriteLine(name?.Length);

// Chained
User? user = GetUser();
string? city = user?.Address?.City?.Name;  // null если любой null
```

### `??` — null coalescing

```csharp
string? maybeNull = GetName();
string display = maybeNull ?? "Anonymous";

int? age = GetAge();
int actualAge = age ?? 0;

// Throw if null
string name = GetName() ?? throw new ArgumentNullException();
```

### `??=` — null coalescing assignment

```csharp
List<int>? items = null;
items ??= new();   // создать если null
items.Add(1);

// Эквивалентно:
if (items is null) items = new();
items.Add(1);

// Lazy init
private List<string>? _cache;
public List<string> Cache => _cache ??= LoadCache();
```

### `!` — null forgiving

"Trust me, это не null":

```csharp
string? maybeNull = GetName();
int len = maybeNull!.Length;  // suppress warning
                              // Runtime: NRE если null

// Когда использовать:
// 1. Знаешь что не null but compiler не может вывести
public string Name => _name!;  // _name установлен в constructor

// 2. Library которая не имеет nullable annotations
var connection = config.GetConnectionString("Default")!;

// 3. Test code
Assert.NotNull(result);
result!.Property.Should().Be(...);
```

> [!warning] Не злоупотребляй `!`
> `!` отключает null check. Если ошибся — runtime NRE. Используй с care.

### `?[...]` — null-conditional indexer

```csharp
List<int>? items = null;
int? first = items?[0];  // null если items null
```

---

## 4. Patterns для null handling

### Pattern 1: Guard clauses

```csharp
public void Process(User user)
{
    if (user is null) throw new ArgumentNullException(nameof(user));

    // ... use user safely
}

// .NET 6+ — ArgumentNullException.ThrowIfNull
public void Process(User user)
{
    ArgumentNullException.ThrowIfNull(user);
    // ...
}
```

### Pattern 2: TryGet / TryParse

```csharp
public bool TryGetUser(int id, out User? user)
{
    user = _users.FirstOrDefault(u => u.Id == id);
    return user != null;
}

if (TryGetUser(1, out var user))
{
    Console.WriteLine(user.Name);
}
```

### Pattern 3: Result\<T, E\> вместо null

```csharp
public Result<User, string> GetUser(int id)
{
    var user = _users.FirstOrDefault(u => u.Id == id);
    return user != null
        ? Result.Ok(user)
        : Result.Error("User not found");
}

// Caller — explicit handling, нельзя забыть про null
var result = GetUser(1);
if (result.IsSuccess)
{
    Console.WriteLine(result.Value.Name);
}
```

См. [[error-handling|Error Handling]] для детального гайда.

### Pattern 4: Maybe / Option type

```csharp
public Option<User> GetUser(int id)
{
    var user = _users.FirstOrDefault(u => u.Id == id);
    return user != null ? Option.Some(user) : Option.None<User>();
}

// LanguageExt library — наиболее популярная implementation
```

### Pattern 5: Empty collection вместо null

```csharp
// ❌ null collection — caller везде должен проверять
public List<Order>? GetOrders(int userId)
{
    if (userId < 0) return null;
    // ...
}

// ✅ Empty list
public List<Order> GetOrders(int userId)
{
    if (userId < 0) return [];  // empty
    // ...
}

// Caller безопасно:
var orders = GetOrders(id);
foreach (var order in orders) { }  // OK даже если empty
```

### Pattern 6: Null Object pattern

```csharp
// Полиморфный "null"
public class NullCustomer : Customer
{
    public override string Name => "Guest";
    public override decimal Discount => 0;
    public override bool CanShip() => false;
}

public Customer GetCustomer(int id) =>
    _customers.FirstOrDefault(c => c.Id == id) ?? new NullCustomer();
```

### Pattern 7: Default values

```csharp
public string GetGreeting(string? name = null)
{
    return $"Hello, {name ?? "Guest"}!";
}

// Configuration
public class Config
{
    public string DbConnection { get; set; } = "Server=localhost";
    public int Timeout { get; set; } = 30;
}
```

---

## 5. NRT в реальных classes

### Constructor initialization

```csharp
public class User
{
    public string Name { get; set; }  // ⚠️ Warning: not initialized
    public string Email { get; set; }
}

// ✅ Init в constructor
public class User
{
    public string Name { get; set; }
    public string Email { get; set; }

    public User(string name, string email)
    {
        Name = name;
        Email = email;
    }
}

// ✅ Required (C# 11+)
public class User
{
    public required string Name { get; set; }
    public required string Email { get; set; }
}

var user = new User { Name = "John", Email = "x@y" };
// var u = new User();  // ❌ required not set

// ✅ Init с default
public class User
{
    public string Name { get; set; } = "Unknown";
    public string Email { get; set; } = string.Empty;
}

// ✅ Records (C# 9+) — auto-handled
public record User(string Name, string Email);
```

### Nullable поля

```csharp
public class User
{
    public string Name { get; set; } = "";        // non-nullable, default
    public string? MiddleName { get; set; }       // nullable
    public DateTime? LastLogin { get; set; }       // nullable value type
}
```

### Generic types

```csharp
public class Repository<T> where T : class
{
    public T? FindById(int id)  // returns null if not found
    {
        // ...
    }

    // Без constraint — `T?` не работает корректно для всех T
    public T? Get<T>() where T : class { return null; }  // OK
    public T? GetVal<T>() where T : struct { return null; }  // returns Nullable<T>
}
```

---

## 6. Attributes для nullability

Для precise annotation — `System.Diagnostics.CodeAnalysis`.

### `[NotNull]` — output никогда null

```csharp
public void Init([NotNull] out string value)
{
    value = "initialized";  // OK
    // value = null;  // ⚠️ Caller думает что не null
}

// Caller
Init(out string s);
Console.WriteLine(s.Length);  // ✅ no warning
```

### `[MaybeNull]` — может быть null даже если type non-nullable

```csharp
[return: MaybeNull]
public T Find<T>() where T : class
{
    // returns T? но type system не может выразить
    return null;  // suppressing warning
}
```

### `[NotNullWhen(true)]` — TryGet pattern

```csharp
public bool TryGet([NotNullWhen(true)] out User? user)
{
    user = ...;
    return user != null;
}

// Compiler понимает:
if (TryGet(out var user))
{
    Console.WriteLine(user.Name);  // ✅ user не null!
}
```

### `[NotNullIfNotNull]` — output null iff input null

```csharp
[return: NotNullIfNotNull(nameof(input))]
public string? Trim(string? input) =>
    input?.Trim();

// Caller:
string? s = "hello";
string trimmed = Trim(s);  // ✅ compiler знает: s не null → result не null

string? n = null;
string? trimmed2 = Trim(n);  // null
```

### `[MemberNotNull]` — после метода поле не null

```csharp
public class Service
{
    private string? _name;

    public string Name
    {
        get
        {
            EnsureName();
            return _name;  // ✅ _name not null после EnsureName
        }
    }

    [MemberNotNull(nameof(_name))]
    private void EnsureName()
    {
        _name ??= "default";
    }
}
```

### `[DisallowNull]` / `[AllowNull]`

```csharp
// Property non-nullable, но setter принимает null (для backwards compat)
public class Config
{
    private string _name = "default";

    [AllowNull]
    public string Name
    {
        get => _name;
        set => _name = value ?? "default";
    }
}

config.Name = null;  // ✅ allowed, sets к "default"
```

---

## 7. Включение NRT в legacy проект

Постепенное включение — лучшая стратегия.

### Шаг 1: Per-project включить

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>
  <!-- или warnings only без annotations -->
  <Nullable>warnings</Nullable>
  <!-- или annotations only без warnings -->
  <Nullable>annotations</Nullable>
</PropertyGroup>
```

### Шаг 2: Per-file директивы

```csharp
// Top of file
#nullable enable

// Или disable в legacy file
#nullable disable

// Reset
#nullable restore

// Granular
#nullable enable warnings
#nullable disable annotations
```

### Шаг 3: Suppress warnings временно

```csharp
#nullable disable warnings
// legacy code
#nullable restore warnings
```

### Постепенный план

```
Week 1:  Включить <Nullable>annotations</Nullable> в одном проекте
         Добавить ? где нужно, no warnings yet
Week 2:  Switch на <Nullable>enable</Nullable>
         Получишь massive warnings count
Week 3+: Постепенно фиксать warnings (10-50 в день)
         Не пытайся всё сразу — годовая работа в legacy
```

---

## 8. Nullable + EF Core

```csharp
public class User
{
    public int Id { get; set; }
    public required string Name { get; set; }              // NOT NULL в БД
    public string? MiddleName { get; set; }                 // nullable column
    public DateTime CreatedAt { get; set; }                 // not null value type
    public DateTime? DeletedAt { get; set; }                 // nullable value type
}
```

EF Core читает nullability:
- `string` → `NOT NULL` column
- `string?` → `NULL` column
- `int` → `NOT NULL`
- `int?` → `NULL`

### Navigation properties

```csharp
public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }

    // Required navigation — non-null после Include
    public required Customer Customer { get; set; }

    // Optional navigation — может быть null
    public Customer? Customer { get; set; }
}
```

См. [[../EFCore/relationships|EF Core Relationships]].

---

## 9. Common Pitfalls

### 1. Compiler "обманывают"

```csharp
public class Config
{
    public string ApiKey { get; set; }  // ⚠️ not initialized!
}

var c = new Config();
Console.WriteLine(c.ApiKey.Length);  // 💥 NRE despite no warning!
```

**Лечение:** `required` или `= "";` или constructor init.

### 2. Async + nullable confusion

```csharp
public async Task<User?> GetUserAsync(int id)
{
    var user = await _repo.FindAsync(id);
    return user;  // user is User? — OK
}

// Caller
var user = await GetUserAsync(1);
Console.WriteLine(user.Name);  // ⚠️ Warning: possible null

if (user is null) return;
Console.WriteLine(user.Name);  // ✅
```

### 3. JSON deserialization

```csharp
public class User
{
    public string Name { get; set; }  // [JsonPropertyName("name")]
}

var json = """{"name": null}""";
var user = JsonSerializer.Deserialize<User>(json);
Console.WriteLine(user.Name.Length);  // 💥 NRE!
```

**Лечение:** mark nullable, или use `required` + validation.

### 4. Default forgiving в old code

```csharp
public string Name { get; set; } = null!;  // ⚠️ Lying

var user = new User();
Console.WriteLine(user.Name.Length);  // 💥 NRE
```

`= null!` — convenience hack для legacy. Используй `required` или constructor.

### 5. Generic type without constraint

```csharp
public T Get<T>() => default!;

string s = Get<string>();   // s — null!
int x = Get<int>();          // x — 0
int? y = Get<int?>();        // y — null
```

`default(T)` для reference type — null. С `default!` подавляется warning.

### 6. Null в LINQ

```csharp
List<User?> users = ...;
var names = users.Select(u => u.Name);  // ⚠️ if some users null

var safe = users.Where(u => u != null).Select(u => u!.Name);

// Или
var names = users.OfType<User>().Select(u => u.Name);  // filter null
```

### 7. struct fields

```csharp
public struct Point
{
    public string Name;  // ⚠️ struct fields всегда default-initialized
}

var p = new Point();
Console.WriteLine(p.Name.Length);  // 💥 NRE — Name is null!
```

Structs — default zeroed. Reference fields — null. NRT не помогает.

### 8. Late initialization patterns

```csharp
public class Service
{
    public DatabaseConnection Connection { get; private set; }  // ⚠️

    public async Task InitializeAsync()
    {
        Connection = await Connect();  // late init
    }
}

// ✅ Через [MemberNotNullWhen] или factory pattern
public static async Task<Service> CreateAsync()
{
    var s = new Service();
    s.Connection = await Connect();
    return s;
}
```

### 9. Library без NRT annotations

```csharp
// Library код без NRT
public class LegacyApi
{
    public string GetData() => null;  // returns null but signature не nullable
}

// Твой код:
string s = legacy.GetData();  // ✅ compiler не warning
Console.WriteLine(s.Length);  // 💥 NRE
```

Solution: defensive coding, treat external API как nullable до верификации.

### 10. `string.IsNullOrEmpty`

```csharp
public void Method(string? s)
{
    if (string.IsNullOrEmpty(s))
        return;

    Console.WriteLine(s.Length);  // ✅ compiler знает! [NotNullWhen(false)]
}

// Также:
string.IsNullOrWhiteSpace(s)
```

Эти методы аннотированы — compiler понимает.

---

## 10. Best Practices

### NRT enable

- **`<Nullable>enable</Nullable>`** в новых проектах
- **TreatWarningsAsErrors** в production
- **Постепенно** в legacy (per-project, per-file)
- **Required keyword** (C# 11+) для critical properties
- **Records** автоматически handle nullability

### Patterns

- **Guard clauses** в начале методов
- **TryGet pattern** для optional results
- **Result\<T, E\>** вместо null returns для domain logic
- **Empty collections** вместо null collections
- **Init-only properties** + constructor для init guarantees

### Operators

- **`?.`** для safe member access
- **`??`** для default values
- **`??=`** для lazy init
- **`!`** только когда **точно** знаешь — sparingly
- **`is null`** > `== null` для clarity

### Methods

- **CancellationToken** non-nullable
- **out parameters** с `[NotNullWhen(true)]`
- **Nullable input** explicitly `T?`
- **Throw для invariant violations** (ArgumentNullException)
- **`ArgumentNullException.ThrowIfNull(param)`** (.NET 6+)

### Collection / API design

- **Empty collection** не null
- **Nullable элементы** ясно маркировать `List<T?>`
- **TryGet** patterns
- **Optional<T>** или Result для clear contracts

---

## 11. Cheat sheet

| Сценарий | Solution |
|----------|----------|
| Variable can be null | `string? name = null` |
| Default if null | `name ?? "default"` |
| Lazy init | `_cache ??= Load()` |
| Safe member access | `user?.Name` |
| Suppress warning (sure not null) | `user!.Name` |
| Throw if null | `name ?? throw new ArgumentNullException()` |
| Guard at method start | `ArgumentNullException.ThrowIfNull(param)` |
| Required property | `public required string Name { get; set; }` |
| Default property value | `public string Name { get; set; } = ""` |
| Try pattern | `bool TryGet(out T? value) where compiler знает` |
| Nullable value type | `int?` или `Nullable<int>` |
| Get value or default | `nullableInt ?? 0` |
| Pattern match | `if (x is not null)` |
| LINQ filter null | `.Where(x => x != null)` |
| LINQ filter nulls strict | `.OfType<T>()` |
| TryParse pattern | `int.TryParse(s, out int v)` |
| Empty collection | `[]` (C# 12) или `new List<T>()` |

---

## 12. Decision tree

```
Может ли значение быть null?
│
├── Yes (legitimately) → T? (e.g. string?)
│   ├── Result<T, E> для business logic
│   ├── Maybe<T> / Option<T> для FP
│   ├── Empty collection вместо null collection
│   └── Null Object pattern для polymorphism
│
└── No (invariant) → T (non-nullable)
    ├── Validate в constructor
    ├── ArgumentNullException.ThrowIfNull
    ├── required keyword (C# 11+)
    └── Default value: = "" / = new()

Как читать nullable:
  - Critical path → throw if null
  - Default OK → ?? defaultValue
  - Optional flow → ?. chain
  - Pattern match → is null / is not null
```

---

## См. также

- [[csharp-basics|C# Basics]] — null operators intro
- [[modern-features|Modern C# Features]] — required, NRT history
- [[error-handling|Error Handling]] — Result\<T, E\> instead of null
- [[functional-csharp|Functional C#]] — Option / Maybe
- [[../Quality/clean-code|Clean Code]] — null antipatterns
- [[../Quality/static-analysis|Static Analysis]] — TreatWarningsAsErrors
- [[generics-deep|Generics Deep]] — T? в generics
- [[../EFCore/basics-tracking|EF Core]] — nullable mappings

## Reading list

- **Microsoft Docs — Nullable reference types** — learn.microsoft.com/dotnet/csharp/nullable-references
- **Migrating to Nullable Reference Types** — learn.microsoft.com
- **Mads Torgersen — NRT design** — devblogs.microsoft.com/dotnet
- **Stephen Toub — NRT in BCL** — devblogs.microsoft.com
- **The Billion Dollar Mistake** — Tony Hoare talk (history of null)
- **Andrew Lock — NRT series** — andrewlock.net
- **LanguageExt** — github.com/louthy/language-ext (Maybe / Option / Result для C#)
