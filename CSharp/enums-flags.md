---
tags: [csharp, enums, flags, bitwise, parsing, junior, middle]
level: Junior
date: 2026-04-30
---

# Enums и Flags — перечисления

> **Полный гайд по enum в C#**: базовое использование, [Flags] для битовых масок, parsing, pattern matching на enums, common pitfalls. Daily work для любого Middle разработчика.

---

## Что это, зачем и когда

### Что такое enum

**Перечисление** — именованные константы одного типа.

```csharp
public enum Status
{
    Active,
    Pending,
    Closed
}

Status s = Status.Active;
```

Под капотом — `int` (по дефолту). Каждое значение получает порядковый номер: `Active=0, Pending=1, Closed=2`.

### Зачем

| Без enum | С enum |
|----------|--------|
| Magic strings: `if (status == "active")` | `if (status == Status.Active)` |
| Magic numbers: `if (level == 3)` | `if (level == LogLevel.Warning)` |
| Typo не ловится: `"actve"` | Compiler error |
| IntelliSense ничего не подсказывает | Все значения видны |
| Refactoring сложно | F2 переименует везде |

**Аналогия:** Кнопки в лифте. Можешь подписать "1, 2, 3, 4" — но "Lobby, Office, Restaurant, Roof" понятнее. Enum — это понятные подписи.

### Где применяются

- **Status fields**: `Active`, `Pending`, `Closed`
- **Log levels**: `Debug`, `Info`, `Warning`, `Error`
- **Permissions** (с `[Flags]`): `Read | Write | Delete`
- **Days of week**: `Monday`, `Tuesday`, ...
- **HTTP methods, status codes**
- **Configuration choices**

---

## 1. Базовый enum

### Объявление

```csharp
public enum Color
{
    Red,
    Green,
    Blue
}

Color c = Color.Red;
Console.WriteLine(c);          // "Red"
Console.WriteLine((int)c);      // 0
```

### Custom underlying values

```csharp
public enum HttpStatus
{
    OK = 200,
    NotFound = 404,
    InternalError = 500
}

int code = (int)HttpStatus.OK;  // 200
```

### Underlying type

По дефолту — `int`. Можно изменить:

```csharp
public enum SmallEnum : byte    // 0..255
{
    A, B, C
}

public enum BigEnum : long      // ±9 quintillion
{
    A = 1L,
    B = 2_000_000_000L
}

// Available types: byte, sbyte, short, ushort, int, uint, long, ulong
```

### Auto-numbering

```csharp
public enum Status
{
    Active,        // 0
    Pending,       // 1
    Closed = 10,   // explicit
    Archived       // 11 (continues from previous)
}
```

---

## 2. Conversion между enum и int / string

### Enum → int

```csharp
Status s = Status.Active;
int n = (int)s;  // 0

// Generic — если не знаешь underlying type
long n = Convert.ToInt64(s);
```

### Int → enum

```csharp
int code = 1;
Status s = (Status)code;  // Pending

// ⚠️ Cast не проверяет валидность!
Status invalid = (Status)999;  // не throws — просто 999 как value
```

### Enum.IsDefined — проверка валидности

```csharp
int code = 999;

if (Enum.IsDefined(typeof(Status), code))
{
    Status s = (Status)code;
}
else
{
    throw new ArgumentException("Invalid status code");
}

// Generic version (.NET 5+)
if (Enum.IsDefined<Status>(code))
{
    // ...
}
```

> [!warning] `IsDefined` — slow
> Internally uses reflection. Не используй в hot path.

### Enum → string

```csharp
Status s = Status.Active;

// ToString
string name = s.ToString();              // "Active"
string name = $"{s}";                     // "Active"

// Format
string formatted = s.ToString("D");      // "0" (decimal value)
string formatted = s.ToString("X");      // "00000000" (hex)
string formatted = s.ToString("G");      // "Active" (default)
```

### String → enum

```csharp
// Parse — throws при invalid
Status s = Enum.Parse<Status>("Active");

// Case-insensitive
Status s = Enum.Parse<Status>("active", ignoreCase: true);

// TryParse — safe
if (Enum.TryParse<Status>("Active", out Status result))
{
    Console.WriteLine(result);
}

// Numeric string тоже работает
Status s = Enum.Parse<Status>("0");  // Status.Active
```

> [!info] TryParse не проверяет IsDefined
> ```csharp
> Enum.TryParse<Status>("999", out var s);  // returns true, s = (Status)999
> // ⚠️ Используй IsDefined дополнительно если нужна валидность
> ```

### Все значения enum

```csharp
// Все values
Status[] all = Enum.GetValues<Status>();    // .NET 5+
Status[] all = (Status[])Enum.GetValues(typeof(Status));

// Все names
string[] names = Enum.GetNames<Status>();   // .NET 5+
string[] names = Enum.GetNames(typeof(Status));

foreach (var s in Enum.GetValues<Status>())
{
    Console.WriteLine($"{s} = {(int)s}");
}
```

---

## 3. [Flags] — битовые маски

### Зачем

Когда нужно хранить **комбинацию** значений в одной переменной.

```csharp
[Flags]
public enum Permissions
{
    None    = 0,
    Read    = 1,    // 0001
    Write   = 2,    // 0010
    Delete  = 4,    // 0100
    Admin   = 8,    // 1000

    ReadWrite = Read | Write,    // 3 (0011)
    All       = Read | Write | Delete | Admin  // 15 (1111)
}

// Combine
Permissions p = Permissions.Read | Permissions.Write;
Console.WriteLine(p);  // "Read, Write" (благодаря [Flags])

// Check
bool canRead = p.HasFlag(Permissions.Read);  // true
bool canDelete = p.HasFlag(Permissions.Delete);  // false
```

### Правила [Flags] enum

1. **Values — степени двойки** (1, 2, 4, 8, 16, ...)
2. **None = 0** — обязательно
3. **Combinations** — можно как именованные shortcuts (`ReadWrite = Read | Write`)
4. **Атрибут [Flags]** — для правильного `ToString()`

### Bitwise операции

```csharp
[Flags]
public enum Permissions { None = 0, Read = 1, Write = 2, Delete = 4 }

Permissions p = Permissions.Read | Permissions.Write;  // OR — добавить

// Add flag
p |= Permissions.Delete;        // p = Read | Write | Delete

// Remove flag
p &= ~Permissions.Write;        // p = Read | Delete

// Check flag (modern, fast)
if ((p & Permissions.Read) != 0)
{
    // has Read
}

// Check flag (HasFlag — slow до .NET Core, fast в новых)
if (p.HasFlag(Permissions.Read))
{
    // has Read
}

// Toggle flag
p ^= Permissions.Write;          // если был — убрать, если не был — добавить
```

### Несколько flags за раз

```csharp
// Has all
Permissions required = Permissions.Read | Permissions.Write;
bool hasAll = (p & required) == required;

// Has any
bool hasAny = (p & required) != 0;
```

### .NET 5+ — `Enum.HasFlag` оптимизирован

```csharp
// До .NET Core 2.1 — slow (boxing, reflection)
p.HasFlag(Permissions.Read);

// .NET 5+ — JIT inlining делает быстрым
p.HasFlag(Permissions.Read);  // ✅ fast

// Manual check (always fast)
(p & Permissions.Read) == Permissions.Read;
```

---

## 4. Pattern matching на enums

### Switch expression (C# 8+)

```csharp
public string Describe(Status s) => s switch
{
    Status.Active => "Currently active",
    Status.Pending => "Waiting for review",
    Status.Closed => "No longer active",
    _ => "Unknown"
};
```

### Multiple values

```csharp
public bool IsTerminal(Status s) => s switch
{
    Status.Closed or Status.Archived => true,
    _ => false
};
```

### С when clauses

```csharp
public string Action(Status s, bool isAdmin) => (s, isAdmin) switch
{
    (Status.Active, _) => "view",
    (Status.Pending, true) => "approve",
    (Status.Pending, false) => "wait",
    (Status.Closed, true) => "reopen",
    _ => "denied"
};
```

### Flags pattern matching

```csharp
public string Describe(Permissions p) => p switch
{
    Permissions.None => "No access",
    var x when x.HasFlag(Permissions.Admin) => "Full admin",
    var x when (x & Permissions.ReadWrite) == Permissions.ReadWrite => "RW user",
    var x when x.HasFlag(Permissions.Read) => "Read only",
    _ => "Limited"
};
```

См. [[modern-features|Modern C# Features]] для pattern matching deep.

---

## 5. Enum + JSON serialization

### System.Text.Json (default 2026)

```csharp
public class User
{
    public Status Status { get; set; }
}

// Default — serialize as int
var user = new User { Status = Status.Active };
JsonSerializer.Serialize(user);  // {"Status":0}

// Use string names
var options = new JsonSerializerOptions
{
    Converters = { new JsonStringEnumConverter() }
};
JsonSerializer.Serialize(user, options);  // {"Status":"Active"}

// Per-property
public class User
{
    [JsonConverter(typeof(JsonStringEnumConverter))]
    public Status Status { get; set; }
}
```

### Custom enum names в JSON

```csharp
public enum OrderStatus
{
    [JsonPropertyName("pending_payment")]
    Pending,

    [JsonPropertyName("paid")]
    Active,

    [JsonPropertyName("cancelled")]
    Closed
}
```

### Versioning enums (важно!)

```csharp
// API v1
public enum Status
{
    Active,    // 0
    Pending,   // 1
    Closed     // 2
}

// API v2 — добавили в середину! ⚠️ Breaking change
public enum Status
{
    Active,    // 0
    NewStatus, // 1  ← shifted!
    Pending,   // 2  ← было 1!
    Closed     // 3
}
// Старые клиенты с "1" получат NewStatus, не Pending!
```

> [!warning] Always use explicit values для public APIs
> ```csharp
> public enum Status
> {
>     Active = 0,
>     Pending = 1,
>     Closed = 2,
>     NewStatus = 3   // добавляй в конец!
> }
> ```

---

## 6. Enum + EF Core

### Хранение в БД

```csharp
public class Order
{
    public int Id { get; set; }
    public OrderStatus Status { get; set; }  // saved as int
}

// Save string вместо int (лучше для readability в БД)
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>();

// Custom conversion
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion(
        v => v.ToString(),
        v => (OrderStatus)Enum.Parse(typeof(OrderStatus), v));
```

### Filtering

```csharp
// Direct
var active = await db.Orders
    .Where(o => o.Status == OrderStatus.Active)
    .ToListAsync();

// Multiple values
var active = await db.Orders
    .Where(o => o.Status == OrderStatus.Active || o.Status == OrderStatus.Pending)
    .ToListAsync();

// Better — список
var states = new[] { OrderStatus.Active, OrderStatus.Pending };
var active = await db.Orders
    .Where(o => states.Contains(o.Status))
    .ToListAsync();
```

См. [[../EFCore/basics-tracking|EF Core Basics]].

---

## 7. Enum + reflection

### GetValues / GetNames

```csharp
foreach (var s in Enum.GetValues<Status>())
{
    Console.WriteLine($"{s} = {(int)s}");
}

// All names
foreach (var name in Enum.GetNames<Status>())
{
    Console.WriteLine(name);
}
```

### GetType / IsEnum

```csharp
object value = Status.Active;
Type type = value.GetType();

if (type.IsEnum)
{
    var underlying = Enum.GetUnderlyingType(type);  // typeof(int)
    Console.WriteLine($"Underlying: {underlying.Name}");
}
```

### Enum с описанием

```csharp
public enum Priority
{
    [Description("Low priority — can wait")]
    Low,

    [Description("Medium priority — normal")]
    Medium,

    [Description("High priority — urgent")]
    High
}

// Get description через reflection
public static string GetDescription<T>(T value) where T : Enum
{
    var field = typeof(T).GetField(value.ToString());
    var attr = field?.GetCustomAttribute<DescriptionAttribute>();
    return attr?.Description ?? value.ToString();
}

GetDescription(Priority.High);  // "High priority — urgent"
```

См. [[attributes-metadata|Attributes & Metadata]].

---

## 8. Common patterns

### Pattern 1: State machine

```csharp
public enum OrderState
{
    Draft,
    Submitted,
    Approved,
    Shipped,
    Delivered,
    Cancelled
}

public class Order
{
    public OrderState State { get; private set; }

    public void Submit()
    {
        if (State != OrderState.Draft)
            throw new InvalidOperationException($"Can't submit from {State}");
        State = OrderState.Submitted;
    }

    public void Approve()
    {
        if (State != OrderState.Submitted)
            throw new InvalidOperationException($"Can't approve from {State}");
        State = OrderState.Approved;
    }

    // ... etc
}
```

### Pattern 2: Type-safe enum dictionary

```csharp
private static readonly Dictionary<HttpStatus, string> StatusMessages = new()
{
    [HttpStatus.OK] = "Success",
    [HttpStatus.NotFound] = "Resource not found",
    [HttpStatus.InternalError] = "Internal server error"
};

public string GetMessage(HttpStatus status) =>
    StatusMessages.GetValueOrDefault(status, "Unknown error");
```

### Pattern 3: Permission checking helper

```csharp
[Flags]
public enum Permissions
{
    None = 0,
    Read = 1,
    Write = 2,
    Delete = 4,
    Admin = 8,
    All = Read | Write | Delete | Admin
}

public class User
{
    public Permissions Permissions { get; set; }

    public bool Can(Permissions required) =>
        (Permissions & required) == required;
}

var user = new User { Permissions = Permissions.Read | Permissions.Write };
user.Can(Permissions.Read);              // true
user.Can(Permissions.Read | Permissions.Write);  // true (has both)
user.Can(Permissions.Admin);             // false
```

### Pattern 4: Enum-driven dispatch

```csharp
public enum NotificationType { Email, SMS, Push }

public interface INotificationSender
{
    Task SendAsync(string message);
}

public class NotificationDispatcher
{
    private readonly Dictionary<NotificationType, INotificationSender> _senders;

    public NotificationDispatcher(IEnumerable<INotificationSender> senders)
    {
        _senders = new()
        {
            [NotificationType.Email] = senders.OfType<EmailSender>().First(),
            [NotificationType.SMS] = senders.OfType<SmsSender>().First(),
            [NotificationType.Push] = senders.OfType<PushSender>().First()
        };
    }

    public Task SendAsync(NotificationType type, string message) =>
        _senders[type].SendAsync(message);
}
```

### Pattern 5: Enum extension methods

```csharp
public static class StatusExtensions
{
    public static bool IsTerminal(this Status s) =>
        s is Status.Closed or Status.Archived;

    public static bool CanTransitionTo(this Status from, Status to) =>
        (from, to) switch
        {
            (Status.Active, Status.Closed) => true,
            (Status.Pending, Status.Active) => true,
            (Status.Pending, Status.Closed) => true,
            _ => false
        };
}

// Usage
if (currentStatus.IsTerminal()) { /* ... */ }
if (oldStatus.CanTransitionTo(newStatus)) { /* ... */ }
```

### Pattern 6: Enum через record (alternative для type safety)

Для maximum type safety — discriminated union через records:

```csharp
public abstract record Status
{
    public sealed record Active(DateTime Since) : Status;
    public sealed record Pending(string Reason) : Status;
    public sealed record Closed(DateTime ClosedAt, string Reason) : Status;
}

// Pattern matching
string Describe(Status s) => s switch
{
    Status.Active a => $"Active since {a.Since}",
    Status.Pending p => $"Pending: {p.Reason}",
    Status.Closed c => $"Closed at {c.ClosedAt}: {c.Reason}",
    _ => throw new UnreachableException()
};
```

См. [[functional-csharp|Functional C#]] для discriminated unions.

---

## 9. Common Pitfalls

### 1. Cast не проверяет IsDefined

```csharp
int code = 999;
Status s = (Status)code;  // ⚠️ s = (Status)999 — invalid но no exception!

// Использование может surprise:
switch (s)
{
    case Status.Active: break;
    case Status.Pending: break;
    case Status.Closed: break;
    default: 
        // ⚠️ Falls here always для invalid
        break;
}
```

**Лечение:** validate перед использованием:

```csharp
if (!Enum.IsDefined<Status>(code))
    throw new ArgumentException("Invalid status");
```

### 2. Default value enum = 0

```csharp
public enum Status
{
    Active = 1,    // ⚠️ нет 0!
    Pending = 2,
    Closed = 3
}

Status s = default;  // = 0 — НЕ valid status!

// ✅ Always include 0 как Default / Unknown / None
public enum Status
{
    Unknown = 0,
    Active = 1,
    Pending = 2,
    Closed = 3
}
```

### 3. [Flags] без степеней двойки

```csharp
// ❌ Не [Flags] semantics
[Flags]
public enum Permissions
{
    Read = 1,
    Write = 2,
    Delete = 3  // ⚠️ 3 = Read | Write — не отдельный flag!
}

Permissions p = Permissions.Read | Permissions.Write;
// p = 3 — это и есть "Delete" по value!
p == Permissions.Delete;  // true ⚠️

// ✅ Степени двойки
[Flags]
public enum Permissions
{
    None = 0,
    Read = 1,
    Write = 2,
    Delete = 4    // 4, не 3!
}
```

### 4. Forgot None = 0 в [Flags]

```csharp
[Flags]
public enum Permissions
{
    Read = 1,
    Write = 2
}

Permissions p = 0;  // что это? Нет имени!
Console.WriteLine(p);  // "0" вместо "None"

// ✅
[Flags]
public enum Permissions
{
    None = 0,    // ALWAYS добавляй
    Read = 1,
    Write = 2
}
```

### 5. ToString() — performance

```csharp
// ❌ Каждый call — reflection!
for (int i = 0; i < 10_000; i++)
{
    string s = Status.Active.ToString();  // slow!
}

// ✅ Cache если в hot path
private static readonly string ActiveString = Status.Active.ToString();
```

> [!info] .NET 5+ — `ToString()` на enum оптимизирован
> Раньше был очень slow. Сейчас — приемлемо для most cases.

### 6. HasFlag — slow до .NET Core

```csharp
// До .NET Core 2.1 — slow (boxing)
p.HasFlag(Permissions.Read);

// Manual — always fast
(p & Permissions.Read) != 0;
```

В .NET 5+ — оба varianta fast.

### 7. JsonStringEnumConverter не handles unknown values

```csharp
public enum Status { Active, Pending }

// JSON: "Closed" — нет такого в enum!
JsonSerializer.Deserialize<Status>("\"Closed\"");
// Throws JsonException

// ✅ Custom converter с fallback
public class FlexibleEnumConverter<T> : JsonConverter<T> where T : struct, Enum
{
    public override T Read(ref Utf8JsonReader reader, Type type, JsonSerializerOptions options)
    {
        var s = reader.GetString();
        if (Enum.TryParse<T>(s, ignoreCase: true, out var result))
            return result;
        return default;  // или throw, или return Unknown
    }

    public override void Write(Utf8JsonWriter writer, T value, JsonSerializerOptions options) =>
        writer.WriteStringValue(value.ToString());
}
```

### 8. Enum в Dictionary key — performance

```csharp
// До .NET 5 — Dictionary<Enum, T> использует object.GetHashCode (boxing!)
Dictionary<Status, int> dict = new();

// .NET 5+ — оптимизировано, но всё равно медленнее int

// ✅ Если нужен maximum performance:
int[] array = new int[Enum.GetValues<Status>().Length];
array[(int)Status.Active] = 42;  // O(1), no boxing
```

### 9. Enum.Parse без TryParse в production

```csharp
// ❌ User input — может throw!
public Status GetStatus(string s) => Enum.Parse<Status>(s);

// ✅ TryParse + default
public Status GetStatus(string s) =>
    Enum.TryParse<Status>(s, ignoreCase: true, out var result) 
        ? result 
        : Status.Unknown;
```

### 10. Switch не exhaustive

```csharp
public string Describe(Status s)
{
    switch (s)
    {
        case Status.Active: return "Active";
        case Status.Pending: return "Pending";
        // ⚠️ Closed забыли!
    }
    throw new ArgumentException();  // runtime error
}

// ✅ Switch expression — компилятор warning если не все
public string Describe(Status s) => s switch
{
    Status.Active => "Active",
    Status.Pending => "Pending",
    Status.Closed => "Closed",
    _ => throw new UnreachableException()  // explicit fallback
};
```

---

## 10. Best Practices

### Design

- **`None`/`Unknown` = 0** — default values valid
- **Explicit underlying values** для public APIs (versioning safety)
- **PascalCase** для names: `Status.Active`
- **Singular name**: `Status`, не `Statuses`
- **`[Flags]` только для bitmasks** — не abuse

### Conversion

- **`Enum.IsDefined`** для validation
- **`Enum.TryParse`** не Parse в production
- **`(int)e`** для cast, не `Convert.ToInt32`

### Flags

- **Степени двойки** (1, 2, 4, 8, ...)
- **`None = 0`** обязательно
- **Combinations как const**: `All = Read | Write | Delete`
- **`HasFlag` или `(x & flag) != 0`** для check

### Pattern matching

- **Switch expression** > switch statement
- **Exhaustive cases** + `_` fallback
- **`when` clauses** для complex conditions

### Performance

- **Cache `ToString()`** в hot paths
- **Manual bit ops** вместо `HasFlag` если pre-.NET 5
- **`int`-keyed array** вместо `Dictionary<Enum, T>` в hot path

### Testing

```csharp
// Test all enum values
[Theory]
[MemberData(nameof(StatusValues))]
public void Process_handles_all_statuses(Status status)
{
    var result = Process(status);
    result.Should().NotBeNull();
}

public static IEnumerable<object[]> StatusValues =>
    Enum.GetValues<Status>().Select(s => new object[] { s });
```

---

## 11. Cheat sheet

| Сценарий | Решение |
|----------|---------|
| Базовое перечисление | `enum Status { Active, Pending }` |
| Битовая маска | `[Flags] enum Permissions { Read=1, Write=2 }` |
| String → enum (safe) | `Enum.TryParse<T>(s, ignoreCase: true, out var v)` |
| Enum → int | `(int)Status.Active` |
| Int → enum (safe) | `Enum.IsDefined<T>(n) ? (T)n : T.Unknown` |
| Enum → string | `s.ToString()` |
| All values | `Enum.GetValues<T>()` |
| Has flag | `p.HasFlag(Permissions.Read)` или `(p & flag) != 0` |
| Add flag | `p \|= Permissions.Read` |
| Remove flag | `p &= ~Permissions.Read` |
| Toggle flag | `p ^= Permissions.Read` |
| Pattern matching | `s switch { Status.Active => ..., _ => ... }` |
| JSON as string | `[JsonConverter(typeof(JsonStringEnumConverter))]` |
| EF as string | `.HasConversion<string>()` |
| Description | `[Description("Hint")]` + reflection |

---

## 12. Decision tree

```
Нужен набор именованных значений?
│
├── Один из вариантов (mutually exclusive)?
│   → enum
│
├── Комбинации (несколько одновременно)?
│   → [Flags] enum + степени двойки
│
├── Нужен associated data?
│   → record / discriminated union (sealed records hierarchy)
│
└── Только 2 варианта?
    → bool (если нет смысла extension)
    → enum (если smysl: Open/Closed > true/false)

Сериализация в JSON?
│
├── Internal API → int (fast, compact)
└── Public API → string (readable, evolution-friendly)

Storage в БД?
│
├── Performance critical → int
└── Readable / debugging → string conversion
```

---

## См. также

- [[csharp-basics|C# Basics]] — типы данных
- [[modern-features|Modern C# Features]] — pattern matching
- [[attributes-metadata|Attributes & Metadata]] — Description attribute
- [[functional-csharp|Functional C#]] — discriminated unions через records
- [[../EFCore/basics-tracking|EF Core Basics]] — enum в БД
- [[oop|OOP]] — when class > enum

## Reading list

- **Microsoft Docs — Enumeration types** — learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/enum
- **Microsoft Docs — Enum guidelines** — learn.microsoft.com/dotnet/standard/design-guidelines/enum
- **Eric Lippert — Enum design** — ericlippert.com (older but classic)
- **Stephen Toub — Enum performance** — devblogs.microsoft.com
- **Andrew Lock — Enum series** — andrewlock.net
