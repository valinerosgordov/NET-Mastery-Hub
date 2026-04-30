---
tags: [modern-csharp, pattern-matching, records, nullable, generics]
level: Middle to Senior
---

# РЎРѕРІСЂРµРјРµРЅРЅС‹Рµ РІРѕР·РјРѕР¶РЅРѕСЃС‚Рё C# (8вЂ“14)

## Р§С‚Рѕ СЌС‚Рѕ, Р·Р°С‡РµРј Рё РєРѕРіРґР°

### Р§С‚Рѕ С‚Р°РєРѕРµ В«СЃРѕРІСЂРµРјРµРЅРЅС‹Р№ C#В»?
**РќРѕРІС‹Рµ С„РёС‡Рё СЏР·С‹РєР°** РЅР°С‡РёРЅР°СЏ СЃ C# 8 (2019) РїРѕ C# 14 (2025). Р”РµР»Р°СЋС‚ РєРѕРґ РєРѕСЂРѕС‡Рµ, Р±РµР·РѕРїР°СЃРЅРµРµ Рё РІС‹СЂР°Р·РёС‚РµР»СЊРЅРµРµ.

**РђРЅР°Р»РѕРіРёСЏ:** РћР±РЅРѕРІР»РµРЅРёРµ РёРЅСЃС‚СЂСѓРјРµРЅС‚РѕРІ РІ РјР°СЃС‚РµСЂСЃРєРѕР№. РњРѕР¶РЅРѕ Р·Р°Р±РёРІР°С‚СЊ РіРІРѕР·РґРё РєР°РјРЅРµРј (СЃС‚Р°СЂС‹Р№ C#), Р° РјРѕР¶РЅРѕ РјРѕР»РѕС‚РєРѕРј (СЃРѕРІСЂРµРјРµРЅРЅС‹Р№ C#). Р РµР·СѓР»СЊС‚Р°С‚ РѕРґРёРЅ, РЅРѕ СЃ РјРѕР»РѕС‚РєРѕРј Р±С‹СЃС‚СЂРµРµ Рё СѓРґРѕР±РЅРµРµ.

### РљР»СЋС‡РµРІС‹Рµ С„РёС‡Рё вЂ” РєРѕРіРґР° С‡С‚Рѕ?

| Р¤РёС‡Р° | Р’РµСЂСЃРёСЏ | Р—Р°С‡РµРј | РљРѕРіРґР° |
|------|--------|-------|-------|
| **Nullable Reference Types** | C# 8 | РљРѕРјРїРёР»СЏС‚РѕСЂ РїСЂРµРґСѓРїСЂРµР¶РґР°РµС‚ Рѕ РІРѕР·РјРѕР¶РЅРѕРј null | Р’СЃРµРіРґР° РІРєР»СЋС‡Р°С‚СЊ (`<Nullable>enable</Nullable>`) |
| **Pattern Matching** | C# 8-11 | РњРѕС‰РЅР°СЏ РїСЂРѕРІРµСЂРєР° С‚РёРїРѕРІ Рё Р·РЅР°С‡РµРЅРёР№ РІ switch | Р—Р°РјРµРЅР° С†РµРїРѕС‡РєРё `if/else if`, РѕР±СЂР°Р±РѕС‚РєР° СЂР°Р·РЅС‹С… С‚РёРїРѕРІ |
| **Records** | C# 9 | Immutable С‚РёРїС‹ СЃ value equality РёР· РєРѕСЂРѕР±РєРё | DTO, Value Objects, РєРѕРЅС„РёРіСѓСЂР°С†РёРё |
| **Global Usings** | C# 10 | РЈР±СЂР°С‚СЊ РїРѕРІС‚РѕСЂСЏСЋС‰РёРµСЃСЏ `using` РёР· РєР°Р¶РґРѕРіРѕ С„Р°Р№Р»Р° | `global using System.Linq;` РІ РѕРґРЅРѕРј С„Р°Р№Р»Рµ |
| **File-scoped Namespaces** | C# 10 | РЈР±СЂР°С‚СЊ РѕРґРёРЅ СѓСЂРѕРІРµРЅСЊ РІР»РѕР¶РµРЅРЅРѕСЃС‚Рё | Р’СЃРµРіРґР°: `namespace X;` РІРјРµСЃС‚Рѕ `namespace X { }` |
| **Raw String Literals** | C# 11 | РњРЅРѕРіРѕСЃС‚СЂРѕС‡РЅС‹Рµ СЃС‚СЂРѕРєРё Р±РµР· СЌРєСЂР°РЅРёСЂРѕРІР°РЅРёСЏ | JSON, SQL, HTML РІ РєРѕРґРµ |
| **Primary Constructors** | C# 12 | DI-РёРЅСЉРµРєС†РёСЏ Р±РµР· boilerplate | `class OrderService(IRepo repo)` |
| **Collection Expressions** | C# 12 | РЈРЅРёС„РёС†РёСЂРѕРІР°РЅРЅС‹Р№ СЃРёРЅС‚Р°РєСЃРёСЃ `[1, 2, 3]` | РРЅРёС†РёР°Р»РёР·Р°С†РёСЏ Р»СЋР±С‹С… РєРѕР»Р»РµРєС†РёР№ |
| **`field` keyword** | C# 13 | Semi-auto properties Р±РµР· backing field | Р’Р°Р»РёРґР°С†РёСЏ РІ setter: `set => field = value ?? throw ...` |

---

> РЎРїСЂР°РІРѕС‡РЅРёРє РїРѕ С„РёС‡Р°Рј СЏР·С‹РєР° C# 8вЂ“14.
> РўРµРѕСЂРёСЏ в†’ РїСЂР°РєС‚РёРєР° в†’ senior-level РєРѕРґ в†’ РІРѕРїСЂРѕСЃС‹ РёРЅС‚РµСЂРІСЊСЋ.

---

## Pattern Matching (C# 8вЂ“11)

### Type Patterns (C# 8)

РџСЂРѕРІРµСЂРєР° С‚РёРїР° СЃ РѕРґРЅРѕРІСЂРµРјРµРЅРЅС‹Рј РѕР±СЉСЏРІР»РµРЅРёРµРј РїРµСЂРµРјРµРЅРЅРѕР№:

```csharp
object obj = "Hello, World!";

// РљР»Р°СЃСЃРёС‡РµСЃРєРёР№ type pattern
if (obj is string s)
{
    Console.WriteLine(s.ToUpper()); // HELLO, WORLD!
}

// Р’ switch expression
string result = obj switch
{
    string s => $"РЎС‚СЂРѕРєР° РґР»РёРЅРѕР№ {s.Length}",
    int i    => $"Р§РёСЃР»Рѕ: {i}",
    null     => "null",
    _        => $"РќРµРёР·РІРµСЃС‚РЅС‹Р№ С‚РёРї: {obj.GetType().Name}"
};
```

### Declaration Рё Constant Patterns (C# 8)

```csharp
// Declaration pattern вЂ” РѕР±СЉСЏРІР»СЏРµРј РїРµСЂРµРјРµРЅРЅСѓСЋ РїСЂРё СЃРѕРІРїР°РґРµРЅРёРё
if (GetOrder() is Order { Status: OrderStatus.Completed } order)
{
    ProcessCompleted(order);
}

// Constant pattern вЂ” СЃСЂР°РІРЅРµРЅРёРµ СЃ РєРѕРЅСЃС‚Р°РЅС‚РѕР№
if (statusCode is 200)
{
    // OK
}

// null check С‡РµСЂРµР· constant pattern
if (name is null)
{
    throw new ArgumentNullException(nameof(name));
}

// not null вЂ” РѕР±СЂР°С‚РЅР°СЏ РїСЂРѕРІРµСЂРєР°
if (name is not null)
{
    Console.WriteLine(name);
}
```

### Property Patterns (C# 8, СЂР°СЃС€РёСЂРµРЅС‹ РІ C# 10)

```csharp
public record Person(string Name, int Age, Address? Address);
public record Address(string City, string Country);

// РџСЂРѕСЃС‚РѕР№ property pattern
if (person is { Age: > 18, Name: not null })
{
    Console.WriteLine("РЎРѕРІРµСЂС€РµРЅРЅРѕР»РµС‚РЅРёР№ СЃ РёРјРµРЅРµРј");
}

// Р’Р»РѕР¶РµРЅРЅС‹Р№ property pattern (C# 10 вЂ” extended property patterns)
if (person is { Address.City: "Moscow" })
{
    Console.WriteLine("РњРѕСЃРєРІРёС‡");
}

// Р­РєРІРёРІР°Р»РµРЅС‚ РґРѕ C# 10 (РІР»РѕР¶РµРЅРЅР°СЏ Р·Р°РїРёСЃСЊ):
if (person is { Address: { City: "Moscow" } })
{
    Console.WriteLine("РњРѕСЃРєРІРёС‡");
}

// РљРѕРјР±РёРЅР°С†РёСЏ РІ switch expression
string category = person switch
{
    { Age: < 13 }                          => "Р РµР±С‘РЅРѕРє",
    { Age: >= 13 and < 18 }               => "РџРѕРґСЂРѕСЃС‚РѕРє",
    { Age: >= 18, Address.Country: "RU" }  => "Р’Р·СЂРѕСЃР»С‹Р№ (Р Р¤)",
    { Age: >= 18 }                         => "Р’Р·СЂРѕСЃР»С‹Р№",
    _                                       => "РќРµРёР·РІРµСЃС‚РЅРѕ"
};
```

### Relational Рё Logical Patterns (C# 9)

РћРїРµСЂР°С‚РѕСЂС‹ СЃСЂР°РІРЅРµРЅРёСЏ `<`, `>`, `<=`, `>=` Рё Р»РѕРіРёС‡РµСЃРєРёРµ РєРѕРјР±РёРЅР°С‚РѕСЂС‹ `and`, `or`, `not`:

```csharp
// Relational patterns
string GetTemperatureDescription(double temp) => temp switch
{
    < -20                 => "Р­РєСЃС‚СЂРµРјР°Р»СЊРЅС‹Р№ С…РѕР»РѕРґ",
    >= -20 and < 0        => "РњРѕСЂРѕР·",
    >= 0 and < 15         => "РџСЂРѕС…Р»Р°РґРЅРѕ",
    >= 15 and < 25        => "РљРѕРјС„РѕСЂС‚РЅРѕ",
    >= 25 and < 35        => "Р–Р°СЂРєРѕ",
    >= 35                 => "Р­РєСЃС‚СЂРµРјР°Р»СЊРЅР°СЏ Р¶Р°СЂР°"
};

// Logical pattern: not
if (statusCode is not (200 or 201 or 204))
{
    LogError(statusCode);
}

// РљРѕРјР±РёРЅР°С†РёСЏ and / or
if (value is > 0 and < 100 or 999)
{
    // value РІ (0, 100) РёР»Рё СЂР°РІРЅРѕ 999
}
```

### Positional Patterns СЃ РґРµРєРѕРЅСЃС‚СЂСѓРєС†РёРµР№ (C# 8)

```csharp
public readonly record struct Point(double X, double Y);

// Positional pattern вЂ” РґРµРєРѕРЅСЃС‚СЂСѓРєС†РёСЏ С‡РµСЂРµР· Deconstruct
string GetQuadrant(Point point) => point switch
{
    (0, 0)       => "РќР°С‡Р°Р»Рѕ РєРѕРѕСЂРґРёРЅР°С‚",
    ( > 0, > 0)  => "РџРµСЂРІР°СЏ С‡РµС‚РІРµСЂС‚СЊ",
    ( < 0, > 0)  => "Р’С‚РѕСЂР°СЏ С‡РµС‚РІРµСЂС‚СЊ",
    ( < 0, < 0)  => "РўСЂРµС‚СЊСЏ С‡РµС‚РІРµСЂС‚СЊ",
    ( > 0, < 0)  => "Р§РµС‚РІС‘СЂС‚Р°СЏ С‡РµС‚РІРµСЂС‚СЊ",
    (_, 0) or (0, _) => "РќР° РѕСЃРё"
};

// Р”РµРєРѕРЅСЃС‚СЂСѓРєС†РёСЏ tuple
(int status, string? body) = GetResponse();
var message = (status, body) switch
{
    (200, not null) => $"OK: {body}",
    (200, null)     => "OK: РїСѓСЃС‚РѕР№ РѕС‚РІРµС‚",
    (404, _)        => "РќРµ РЅР°Р№РґРµРЅРѕ",
    (>= 500, _)     => "РћС€РёР±РєР° СЃРµСЂРІРµСЂР°",
    _               => $"РЎС‚Р°С‚СѓСЃ: {status}"
};
```

### List Patterns (C# 11)

РњРѕС‰РЅС‹Р№ pattern matching РїРѕ РєРѕР»Р»РµРєС†РёСЏРј Рё РјР°СЃСЃРёРІР°Рј:

```csharp
int[] numbers = [1, 2, 3, 4, 5];

// РўРѕС‡РЅРѕРµ СЃРѕРІРїР°РґРµРЅРёРµ
if (numbers is [1, 2, 3, 4, 5])
{
    Console.WriteLine("РўРѕС‡РЅРѕРµ СЃРѕРІРїР°РґРµРЅРёРµ");
}

// Discard Рё slice
if (numbers is [1, _, _, _, 5])
{
    Console.WriteLine("РќР°С‡РёРЅР°РµС‚СЃСЏ СЃ 1, Р·Р°РєР°РЅС‡РёРІР°РµС‚СЃСЏ РЅР° 5");
}

// Slice pattern (..) вЂ” В«Р»СЋР±РѕРµ РєРѕР»РёС‡РµСЃС‚РІРѕ СЌР»РµРјРµРЅС‚РѕРІВ»
if (numbers is [1, .., 5])
{
    Console.WriteLine("РќР°С‡РёРЅР°РµС‚СЃСЏ СЃ 1, Р·Р°РєР°РЅС‡РёРІР°РµС‚СЃСЏ РЅР° 5 (Р»СЋР±Р°СЏ РґР»РёРЅР°)");
}

// Р—Р°С…РІР°С‚ Р·РЅР°С‡РµРЅРёР№
if (numbers is [var first, .., var last])
{
    Console.WriteLine($"РџРµСЂРІС‹Р№: {first}, РџРѕСЃР»РµРґРЅРёР№: {last}");
}

// Р—Р°С…РІР°С‚ slice
if (numbers is [_, .. var middle, _])
{
    // middle = [2, 3, 4]
}

// Р’Р»РѕР¶РµРЅРЅС‹Рµ patterns РІРЅСѓС‚СЂРё list pattern
string[] args = ["--verbose", "--output", "result.json"];
var config = args switch
{
    ["--help"]                        => new Config(ShowHelp: true),
    ["--verbose", "--output", var f]  => new Config(Verbose: true, OutputFile: f),
    [var cmd, ..]                     => new Config(Command: cmd),
    []                                => Config.Default
};
```

### Exhaustive Matching Рё Discard

```csharp
// РљРѕРјРїРёР»СЏС‚РѕСЂ РїСЂРµРґСѓРїСЂРµРґРёС‚, РµСЃР»Рё РЅРµ РІСЃРµ СЃР»СѓС‡Р°Рё РїРѕРєСЂС‹С‚С‹ (РґР»СЏ enum)
public enum PaymentStatus { Pending, Completed, Failed, Refunded }

string GetMessage(PaymentStatus status) => status switch
{
    PaymentStatus.Pending   => "РћР¶РёРґР°РЅРёРµ",
    PaymentStatus.Completed => "Р—Р°РІРµСЂС€С‘РЅ",
    PaymentStatus.Failed    => "РћС€РёР±РєР°",
    PaymentStatus.Refunded  => "Р’РѕР·РІСЂР°С‚",
    // _ => "РќРµРёР·РІРµСЃС‚РЅРѕ" вЂ” discard РґР»СЏ exhaustive matching
    // Р‘РµР· _ РєРѕРјРїРёР»СЏС‚РѕСЂ РІС‹РґР°СЃС‚ warning, РµСЃР»Рё enum СЂР°СЃС€РёСЂСЏС‚
};
```

### Nested Patterns

```csharp
public record Order(
    int Id,
    OrderStatus Status,
    decimal Total,
    Customer? Customer);

public record Customer(string Name, CustomerTier Tier);
public enum CustomerTier { Regular, Silver, Gold, Platinum }
public enum OrderStatus { New, Processing, Shipped, Delivered, Cancelled }

// Р“Р»СѓР±РѕРєРѕ РІР»РѕР¶РµРЅРЅС‹Р№ pattern matching
decimal CalculateDiscount(Order order) => order switch
{
    { Status: OrderStatus.Cancelled } => 0m,
    { Total: < 100m }                 => 0m,
    { Customer: null }                 => 0m,

    { Total: >= 1000m, Customer: { Tier: CustomerTier.Platinum } }
        => order.Total * 0.20m,

    { Total: >= 500m, Customer: { Tier: CustomerTier.Gold or CustomerTier.Platinum } }
        => order.Total * 0.15m,

    { Customer: { Tier: CustomerTier.Gold } }
        => order.Total * 0.10m,

    { Customer: { Tier: CustomerTier.Silver } }
        => order.Total * 0.05m,

    _ => 0m
};
```

### РџСЂР°РєС‚РёС‡РµСЃРєРёРµ РїСЂРёРјРµСЂС‹: РІР°Р»РёРґР°С†РёСЏ, РїР°СЂСЃРёРЅРі, state machines

```csharp
// --- Р’Р°Р»РёРґР°С†РёСЏ ---
public static Result<CreateOrderCommand> Validate(CreateOrderCommand cmd) => cmd switch
{
    { CustomerId: <= 0 }       => Result<CreateOrderCommand>.Failure("РќРµРІР°Р»РёРґРЅС‹Р№ CustomerId"),
    { Items: [] }              => Result<CreateOrderCommand>.Failure("Р—Р°РєР°Р· Р±РµР· С‚РѕРІР°СЂРѕРІ"),
    { Items: [.., { Qty: <= 0 }] } => Result<CreateOrderCommand>.Failure("РљРѕР»РёС‡РµСЃС‚РІРѕ <= 0"),
    _                          => Result<CreateOrderCommand>.Success(cmd)
};

// --- РџР°СЂСЃРёРЅРі CLI-Р°СЂРіСѓРјРµРЅС‚РѕРІ ---
static CliOptions ParseArgs(string[] args) => args switch
{
    ["-v" or "--version"]            => new(ShowVersion: true),
    ["-h" or "--help"]               => new(ShowHelp: true),
    ["run", var file]                => new(Command: "run", File: file),
    ["run", var file, "--watch"]     => new(Command: "run", File: file, Watch: true),
    ["build", "--release"]           => new(Command: "build", Release: true),
    []                               => CliOptions.Default,
    _                                => throw new ArgumentException($"РќРµРёР·РІРµСЃС‚РЅС‹Рµ Р°СЂРіСѓРјРµРЅС‚С‹: {string.Join(' ', args)}")
};

// --- State Machine ---
public static OrderStatus NextState(OrderStatus current, OrderEvent @event) =>
    (current, @event) switch
    {
        (OrderStatus.New, OrderEvent.Pay)               => OrderStatus.Processing,
        (OrderStatus.Processing, OrderEvent.Ship)       => OrderStatus.Shipped,
        (OrderStatus.Shipped, OrderEvent.Deliver)       => OrderStatus.Delivered,
        (OrderStatus.New, OrderEvent.Cancel)             => OrderStatus.Cancelled,
        (OrderStatus.Processing, OrderEvent.Cancel)      => OrderStatus.Cancelled,
        _ => throw new InvalidOperationException(
            $"РќРµРІР°Р»РёРґРЅС‹Р№ РїРµСЂРµС…РѕРґ: {current} + {@event}")
    };
```

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: Switch expression вЂ” exhaustive matching?**
> РљРѕРјРїРёР»СЏС‚РѕСЂ РїСЂРѕРІРµСЂСЏРµС‚ РїРѕР»РЅРѕС‚Сѓ РґР»СЏ enum, bool, sealed hierarchy. Р”Р»СЏ РѕС‚РєСЂС‹С‚С‹С… С‚РёРїРѕРІ вЂ” `_` (discard). Warning РїСЂРё РЅРµРїРѕР»РЅРѕРј РїРѕРєСЂС‹С‚РёРё. Р’ production: РІСЃРµРіРґР° `_` СЃ throw РґР»СЏ Р·Р°С‰РёС‚С‹ РѕС‚ РЅРѕРІС‹С… Р·РЅР°С‡РµРЅРёР№.

---

## Nullable Reference Types (C# 8)

### Р’РєР»СЋС‡РµРЅРёРµ

```xml
<!-- Р’ .csproj -->
<PropertyGroup>
    <Nullable>enable</Nullable>
</PropertyGroup>
```

```csharp
// РР»Рё РЅР° СѓСЂРѕРІРЅРµ С„Р°Р№Р»Р°
#nullable enable
```

### РђРЅРЅРѕС‚Р°С†РёСЏ `?` вЂ” nullable vs non-nullable

```csharp
// non-nullable вЂ” РєРѕРјРїРёР»СЏС‚РѕСЂ РіР°СЂР°РЅС‚РёСЂСѓРµС‚, С‡С‚Рѕ null РЅРµ Р±СѓРґРµС‚
string name = "РРІР°РЅ";
// name = null; // CS8600 warning

// nullable вЂ” СЏРІРЅРѕ РіРѕРІРѕСЂРёРј: В«Р·РґРµСЃСЊ РјРѕР¶РµС‚ Р±С‹С‚СЊ nullВ»
string? middleName = null; // OK

// Р’РѕР·РІСЂР°С‚ nullable
public string? FindUserName(int id)
{
    return _users.TryGetValue(id, out var user) ? user.Name : null;
}
```

### Null-forgiving operator `!`

РСЃРїРѕР»СЊР·СѓРµРј **С‚РѕР»СЊРєРѕ** РєРѕРіРґР° С‚РѕС‡РЅРѕ Р·РЅР°РµРј, С‡С‚Рѕ Р·РЅР°С‡РµРЅРёРµ РЅРµ null, РЅРѕ РєРѕРјРїРёР»СЏС‚РѕСЂ РЅРµ РјРѕР¶РµС‚ СЌС‚Рѕ РґРѕРєР°Р·Р°С‚СЊ:

```csharp
// Р”РѕРїСѓСЃС‚РёРјРѕ: РїРѕСЃР»Рµ РїСЂРѕРІРµСЂРєРё РІ РґСЂСѓРіРѕРј РјРµСЃС‚Рµ
var item = cache.Get(key); // РІРѕР·РІСЂР°С‰Р°РµС‚ T?
Debug.Assert(item is not null);
Process(item!); // РјС‹ СѓР¶Рµ РїСЂРѕРІРµСЂРёР»Рё

// Р”РѕРїСѓСЃС‚РёРјРѕ: EF Core navigation property (Р·Р°РїРѕР»РЅСЏРµС‚СЃСЏ ORM)
public class Order
{
    public int CustomerId { get; set; }
    public Customer Customer { get; set; } = null!; // EF Р·Р°РїРѕР»РЅРёС‚
}

// Р—РђРџР Р•Р©Р•РќРћ: СЃР»РµРїРѕРµ РїРѕРґР°РІР»РµРЅРёРµ warnings
string? x = GetValue();
Console.WriteLine(x!.Length); // РѕРїР°СЃРЅРѕ вЂ” РјРѕР¶РµС‚ Р±С‹С‚СЊ null
```

### Null Guards

```csharp
// .NET 6+ вЂ” СЃР°РјС‹Р№ РєРѕСЂРѕС‚РєРёР№ СЃРїРѕСЃРѕР±
public void Process(string name, Order order)
{
    ArgumentNullException.ThrowIfNull(name);
    ArgumentNullException.ThrowIfNull(order);
    // ...
}

// .NET 7+ вЂ” РґР»СЏ СЃС‚СЂРѕРє
public void SetName(string name)
{
    ArgumentException.ThrowIfNullOrEmpty(name);
    ArgumentException.ThrowIfNullOrWhiteSpace(name); // .NET 8
}
```

### РћРїРµСЂР°С‚РѕСЂС‹ `??`, `??=`, `?.`, `?[]`

```csharp
// ?? вЂ” null-coalescing
string displayName = user.NickName ?? user.FullName ?? "РђРЅРѕРЅРёРј";

// ??= вЂ” null-coalescing assignment
_cache ??= new Dictionary<string, object>();

// ?. вЂ” null-conditional member access
int? length = text?.Length;
string? upper = text?.ToUpper();

// ?[] вЂ” null-conditional element access
int? first = numbers?[0];

// Р¦РµРїРѕС‡РєР°
string? city = order?.Customer?.Address?.City;

// РљРѕРјР±РёРЅР°С†РёСЏ ?. Рё ??
string city = order?.Customer?.Address?.City ?? "РќРµРёР·РІРµСЃС‚РЅРѕ";
```

### Nullable-Р°С‚СЂРёР±СѓС‚С‹ (System.Diagnostics.CodeAnalysis)

```csharp
using System.Diagnostics.CodeAnalysis;

// [NotNull] вЂ” РїРѕСЃР»Рµ РІС‹Р·РѕРІР° РјРµС‚РѕРґР° РїР°СЂР°РјРµС‚СЂ РіР°СЂР°РЅС‚РёСЂРѕРІР°РЅРЅРѕ РЅРµ null
public void EnsureInitialized([NotNull] ref string? value)
{
    value ??= "default";
}

// [MaybeNull] вЂ” generic T РјРѕР¶РµС‚ РІРµСЂРЅСѓС‚СЊ null
[return: MaybeNull]
public T Find<T>(int id) where T : class
{
    return _store.TryGetValue(id, out var item) ? (T)item : default;
}

// [NotNullWhen] вЂ” РїР°СЂР°РјРµС‚СЂ РЅРµ null, РєРѕРіРґР° РјРµС‚РѕРґ РІРѕР·РІСЂР°С‰Р°РµС‚ true/false
public bool TryGetUser(int id, [NotNullWhen(true)] out User? user)
{
    return _users.TryGetValue(id, out user);
}

// [MemberNotNull] вЂ” РїРѕСЃР»Рµ РІС‹Р·РѕРІР° РјРµС‚РѕРґР° РїРѕР»Рµ РіР°СЂР°РЅС‚РёСЂРѕРІР°РЅРЅРѕ РЅРµ null
[MemberNotNull(nameof(_connection))]
private void EnsureConnected()
{
    _connection ??= CreateConnection();
}

// [AllowNull] вЂ” РјРѕР¶РЅРѕ РїРµСЂРµРґР°С‚СЊ null, РґР°Р¶Рµ РµСЃР»Рё С‚РёРї non-nullable
public string Title
{
    get => _title;
    [param: AllowNull]
    set => _title = value ?? "Р‘РµР· РЅР°Р·РІР°РЅРёСЏ";
}
```

### Р›СѓС‡С€РёРµ РїСЂР°РєС‚РёРєРё

```csharp
// 1. Р’СЃРµРіРґР° РІРєР»СЋС‡Р°С‚СЊ <Nullable>enable</Nullable> РІ РЅРѕРІС‹С… РїСЂРѕРµРєС‚Р°С….
// 2. РЎС‚СЂРµРјРёС‚СЊСЃСЏ Рє РЅСѓР»СЋ nullable warnings.
// 3. РќРµ Р·Р»РѕСѓРїРѕС‚СЂРµР±Р»СЏС‚СЊ ! вЂ” РєР°Р¶РґРѕРµ РёСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ = РїРѕС‚РµРЅС†РёР°Р»СЊРЅС‹Р№ Р±Р°Рі.
// 4. Р”Р»СЏ DTO/API models вЂ” string? РґР»СЏ РѕРїС†РёРѕРЅР°Р»СЊРЅС‹С… РїРѕР»РµР№.
// 5. Р”Р»СЏ domain models вЂ” string (non-nullable), РёРЅРІР°СЂРёР°РЅС‚С‹ РІ РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂРµ.
// 6. РСЃРїРѕР»СЊР·РѕРІР°С‚СЊ [NotNullWhen], [MaybeNull] РґР»СЏ С‚РѕС‡РЅРѕРіРѕ РєРѕРЅС‚СЂР°РєС‚Р° API.
```

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: Nullable Reference Types вЂ” РєР°Рє РІРєР»СЋС‡РёС‚СЊ Рё С‡С‚Рѕ РґР°С‘С‚?**
> `<Nullable>enable</Nullable>` РІ csproj. РљРѕРјРїРёР»СЏС‚РѕСЂ РїСЂРµРґСѓРїСЂРµР¶РґР°РµС‚ РїСЂРё РїСЂРёСЃРІРѕРµРЅРёРё null РІ non-nullable, СЂР°Р·С‹РјРµРЅРѕРІР°РЅРёРё Р±РµР· РїСЂРѕРІРµСЂРєРё. Р­С‚Рѕ **С‚РѕР»СЊРєРѕ Р°РЅРЅРѕС‚Р°С†РёРё** вЂ” РІ runtime РїСЂРѕРІРµСЂРѕРє РЅРµС‚.
>
> **РђС‚СЂРёР±СѓС‚С‹:** `[NotNull]`, `[MaybeNull]`, `[NotNullWhen(true)]` вЂ” С‚РѕРЅРєРёР№ РєРѕРЅС‚СЂРѕР»СЊ flow analysis.

---

## Records (C# 9вЂ“10)

### record class вЂ” reference type (C# 9)

```csharp
// Positional syntax вЂ” СЃР°РјС‹Р№ С‡Р°СЃС‚С‹Р№ РІР°СЂРёР°РЅС‚
public record Person(string Name, int Age);

// РљРѕРјРїРёР»СЏС‚РѕСЂ РіРµРЅРµСЂРёСЂСѓРµС‚:
// - РљРѕРЅСЃС‚СЂСѓРєС‚РѕСЂ Person(string, int)
// - РЎРІРѕР№СЃС‚РІР° Name { get; init; } Рё Age { get; init; }
// - Deconstruct(out string, out int)
// - Value equality (Equals, GetHashCode, ==, !=)
// - ToString() в†’ "Person { Name = РРІР°РЅ, Age = 30 }"
// - with-expression support

// Р Р°СЃС€РёСЂРµРЅРЅР°СЏ Р·Р°РїРёСЃСЊ СЃ С‚РµР»РѕРј
public record Person(string Name, int Age)
{
    // Р”РѕРїРѕР»РЅРёС‚РµР»СЊРЅРѕРµ СЃРІРѕР№СЃС‚РІРѕ
    public string DisplayName => $"{Name} ({Age})";

    // РџРµСЂРµРѕРїСЂРµРґРµР»РµРЅРёРµ ToString С‡РµСЂРµР· PrintMembers
    protected virtual bool PrintMembers(StringBuilder sb)
    {
        sb.Append($"Name = {Name}, Age = {Age}");
        return true;
    }
}
```

### record struct вЂ” value type (C# 10)

```csharp
// РџРѕ СѓРјРѕР»С‡Р°РЅРёСЋ вЂ” mutable!
public record struct Coordinate(double Lat, double Lon);

// Р”Р»СЏ immutability вЂ” РґРѕР±Р°РІР»СЏРµРј readonly
public readonly record struct Money(decimal Amount, string Currency);

var m = new Money(100m, "RUB");
// m.Amount = 200m; // CS8852 вЂ” readonly
```

### with-expressions

```csharp
var person = new Person("РРІР°РЅ", 30);
var older = person with { Age = 31 };

Console.WriteLine(person); // Person { Name = РРІР°РЅ, Age = 30 }
Console.WriteLine(older);  // Person { Name = РРІР°РЅ, Age = 31 }
Console.WriteLine(person == older); // False

// Deep copy СЃ РѕРґРёРЅР°РєРѕРІС‹РјРё Р·РЅР°С‡РµРЅРёСЏРјРё
var copy = person with { };
Console.WriteLine(person == copy);           // True (value equality)
Console.WriteLine(ReferenceEquals(person, copy)); // False (СЂР°Р·РЅС‹Рµ РѕР±СЉРµРєС‚С‹)
```

### Value Equality

```csharp
var p1 = new Person("РРІР°РЅ", 30);
var p2 = new Person("РРІР°РЅ", 30);

Console.WriteLine(p1 == p2);      // True вЂ” value equality
Console.WriteLine(p1.Equals(p2)); // True
Console.WriteLine(ReferenceEquals(p1, p2)); // False

// class вЂ” reference equality РїРѕ СѓРјРѕР»С‡Р°РЅРёСЋ
// record вЂ” value equality РїРѕ СѓРјРѕР»С‡Р°РЅРёСЋ
```

### Inheritance СЃ records

```csharp
public record Animal(string Name);
public record Dog(string Name, string Breed) : Animal(Name);

var dog = new Dog("РЁР°СЂРёРє", "Р›Р°Р±СЂР°РґРѕСЂ");
Animal animal = dog;

// Equality СѓС‡РёС‚С‹РІР°РµС‚ runtime-С‚РёРї (EqualityContract)
var animal2 = new Animal("РЁР°СЂРёРє");
Console.WriteLine(dog == animal2); // False вЂ” СЂР°Р·РЅС‹Рµ С‚РёРїС‹

// РћР“Р РђРќРР§Р•РќРРЇ:
// - record class РјРѕР¶РµС‚ РЅР°СЃР»РµРґРѕРІР°С‚СЊСЃСЏ С‚РѕР»СЊРєРѕ РѕС‚ record class
// - record struct РЅРµ РїРѕРґРґРµСЂР¶РёРІР°РµС‚ РЅР°СЃР»РµРґРѕРІР°РЅРёРµ (РєР°Рє Р»СЋР±РѕР№ struct)
// - sealed record вЂ” Р·Р°РїСЂРµС‚ РЅР°СЃР»РµРґРѕРІР°РЅРёСЏ
public sealed record Immutable(string Value);
```

### record vs class vs struct вЂ” С‚Р°Р±Р»РёС†Р° СЃСЂР°РІРЅРµРЅРёСЏ

```
| РђСЃРїРµРєС‚              | class          | record class   | struct         | record struct     |
|---------------------|----------------|----------------|----------------|-------------------|
| РўРёРї                 | Reference      | Reference      | Value          | Value             |
| Equality            | Reference      | Value          | Value          | Value             |
| Immutable           | РќРµС‚            | РџРѕ СѓРјРѕР»С‡Р°РЅРёСЋ   | РќРµС‚            | РќРµС‚ (readonly РґР°) |
| Inheritance         | Р”Р°             | Р”Р° (records)   | РќРµС‚            | РќРµС‚               |
| with-expression     | РќРµС‚            | Р”Р°             | РќРµС‚            | Р”Р°                |
| Deconstruct         | Р СѓС‡РЅРѕР№         | РђРІС‚Рѕ           | Р СѓС‡РЅРѕР№         | РђРІС‚Рѕ              |
| ToString            | РўРёРї            | Р’СЃРµ СЃРІРѕР№СЃС‚РІР°   | РўРёРї            | Р’СЃРµ СЃРІРѕР№СЃС‚РІР°      |
| Heap/Stack          | Heap           | Heap           | Stack*         | Stack*            |
```

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: record class vs record struct вЂ” СЂР°Р·Р»РёС‡РёСЏ?**
> **record class** вЂ” reference (heap), value equality, РЅР°СЃР»РµРґРѕРІР°РЅРёРµ, `with` РєРѕРїРёСЂСѓРµС‚ РѕР±СЉРµРєС‚. Р”Р»СЏ DTO, API-РјРѕРґРµР»РµР№.
>
> **record struct** вЂ” value (stack), value equality, Р±РµР· РЅР°СЃР»РµРґРѕРІР°РЅРёСЏ. Р”Р»СЏ РјР°Р»РµРЅСЊРєРёС… value-РѕР±СЉРµРєС‚РѕРІ РІ hot path. `readonly record struct` вЂ” РїСЂРµРґРїРѕС‡С‚РёС‚РµР»СЊРЅР°СЏ С„РѕСЂРјР°.

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: РљРѕРІР°СЂРёР°РЅС‚РЅРѕСЃС‚СЊ Рё РєРѕРЅС‚СЂР°РІР°СЂРёР°РЅС‚РЅРѕСЃС‚СЊ?**
> **РљРѕРІР°СЂРёР°РЅС‚РЅРѕСЃС‚СЊ (`out T`):** `IEnumerable<Cat>` в†’ `IEnumerable<Animal>`. РўРѕР»СЊРєРѕ С‡С‚РµРЅРёРµ.
>
> **РљРѕРЅС‚СЂР°РІР°СЂРёР°РЅС‚РЅРѕСЃС‚СЊ (`in T`):** `IComparer<Animal>` в†’ `IComparer<Cat>`. РўРѕР»СЊРєРѕ РїСЂРёС‘Рј.
>
> **РРЅРІР°СЂРёР°РЅС‚РЅРѕСЃС‚СЊ:** `List<Cat>` РќР• в†’ `List<Animal>` вЂ” List Рё РїРёС€РµС‚, Рё С‡РёС‚Р°РµС‚.

---

## Init-only Рё Required (C# 9, 11)

### init accessor (C# 9)

```csharp
public class UserDto
{
    public string Name { get; init; }    // РјРѕР¶РЅРѕ Р·Р°РґР°С‚СЊ С‚РѕР»СЊРєРѕ РїСЂРё РёРЅРёС†РёР°Р»РёР·Р°С†РёРё
    public string Email { get; init; }
    public int Age { get; init; }
}

var user = new UserDto { Name = "РРІР°РЅ", Email = "ivan@mail.ru", Age = 30 };
// user.Name = "РџС‘С‚СЂ"; // CS8852 вЂ” init-only
```

### required modifier (C# 11)

```csharp
public class CreateOrderDto
{
    public required string ProductName { get; init; }
    public required int Quantity { get; init; }
    public required decimal Price { get; init; }
    public string? Notes { get; init; } // РѕРїС†РёРѕРЅР°Р»СЊРЅРѕРµ
}

// РљРѕРјРїРёР»СЏС‚РѕСЂ Р·Р°СЃС‚Р°РІРёС‚ СѓРєР°Р·Р°С‚СЊ РІСЃРµ required СЃРІРѕР№СЃС‚РІР°:
var dto = new CreateOrderDto
{
    ProductName = "РљР»Р°РІРёР°С‚СѓСЂР°",
    Quantity = 1,
    Price = 5000m
};

// var bad = new CreateOrderDto { ProductName = "РњС‹С€СЊ" };
// CS9035 вЂ” Quantity Рё Price РѕР±СЏР·Р°С‚РµР»СЊРЅС‹
```

### SetsRequiredMembers

```csharp
public class OrderEntity
{
    public required int Id { get; init; }
    public required string Name { get; init; }

    // РљРѕРЅСЃС‚СЂСѓРєС‚РѕСЂ, РєРѕС‚РѕСЂС‹Р№ Р·Р°РїРѕР»РЅСЏРµС‚ РІСЃРµ required members
    [System.Diagnostics.CodeAnalysis.SetsRequiredMembers]
    public OrderEntity(int id, string name)
    {
        Id = id;
        Name = name;
    }

    // РџР°СЂР°РјРµС‚СЂless constructor РґР»СЏ EF Core вЂ” Р±РµР· SetsRequiredMembers
    public OrderEntity() { }
}

// РћР±Р° РІР°СЂРёР°РЅС‚Р° РєРѕРјРїРёР»РёСЂСѓСЋС‚СЃСЏ:
var a = new OrderEntity(1, "Test");
var b = new OrderEntity { Id = 2, Name = "Test2" };
```

### РРґРµР°Р»СЊРЅС‹Р№ DTO: required + init

```csharp
// РџР°С‚С‚РµСЂРЅ РґР»СЏ API DTOs вЂ” РѕР±СЏР·Р°С‚РµР»СЊРЅС‹Рµ РїРѕР»СЏ, РёРјРјСѓС‚Р°Р±РµР»СЊРЅРѕСЃС‚СЊ
public sealed class CreateUserRequest
{
    public required string Email { get; init; }
    public required string Password { get; init; }
    public required string DisplayName { get; init; }
    public string? AvatarUrl { get; init; }
    public string? Bio { get; init; }
}
```

---

## Primary Constructors (C# 12)

### Р‘Р°Р·РѕРІС‹Р№ СЃРёРЅС‚Р°РєСЃРёСЃ

```csharp
// Р”Рѕ C# 12 вЂ” СЏРІРЅС‹Р№ РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂ + РїРѕР»СЏ
public sealed class OrderService
{
    private readonly IOrderRepository _orderRepo;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IOrderRepository orderRepo, ILogger<OrderService> logger)
    {
        _orderRepo = orderRepo;
        _logger = logger;
    }

    public async Task<Order> GetAsync(int id)
    {
        _logger.LogInformation("РџРѕР»СѓС‡РµРЅРёРµ Р·Р°РєР°Р·Р° {Id}", id);
        return await _orderRepo.GetByIdAsync(id);
    }
}

// C# 12 вЂ” primary constructor
public sealed class OrderService(
    IOrderRepository orderRepo,
    ILogger<OrderService> logger)
{
    public async Task<Order> GetAsync(int id)
    {
        logger.LogInformation("РџРѕР»СѓС‡РµРЅРёРµ Р·Р°РєР°Р·Р° {Id}", id);
        return await orderRepo.GetByIdAsync(id);
    }
}
```

### Р—Р°С…РІР°С‚ РїР°СЂР°РјРµС‚СЂРѕРІ вЂ” РєР°Рє СЂР°Р±РѕС‚Р°РµС‚

```csharp
// РџР°СЂР°РјРµС‚СЂС‹ primary constructor Р·Р°С…РІР°С‚С‹РІР°СЋС‚СЃСЏ РєР°Рє СЃРєСЂС‹С‚С‹Рµ РїРѕР»СЏ.
// РћРЅРё РќР• СЏРІР»СЏСЋС‚СЃСЏ readonly вЂ” СЌС‚Рѕ РІР°Р¶РЅРѕ!

public class Counter(int initial)
{
    // initial вЂ” mutable capture, РјРѕР¶РЅРѕ РјРµРЅСЏС‚СЊ
    public int Increment() => ++initial;
    public int Value => initial;
}

var c = new Counter(0);
c.Increment(); // 1
c.Increment(); // 2
```

### РљРѕРіРґР° СЃРѕР·РґР°РІР°С‚СЊ СЏРІРЅРѕРµ РїРѕР»Рµ

```csharp
public sealed class UserService(
    IUserRepository userRepo,
    ILogger<UserService> logger)
{
    // Р•СЃР»Рё РЅСѓР¶РµРЅ readonly вЂ” СЃРѕР·РґР°С‘Рј РїРѕР»Рµ СЏРІРЅРѕ
    private readonly IUserRepository _userRepo = userRepo;

    // РќРµ РѕР±СЂР°С‰Р°Р№С‚РµСЃСЊ Рє `userRepo` РїРѕСЃР»Рµ РїСЂРёСЃРІРѕРµРЅРёСЏ РІ РїРѕР»Рµ вЂ”
    // СЌС‚Рѕ РґРІР° СЂР°Р·РЅС‹С… С…СЂР°РЅРёР»РёС‰Р°!
    public async Task<User?> FindAsync(int id)
    {
        logger.LogInformation("РџРѕРёСЃРє РїРѕР»СЊР·РѕРІР°С‚РµР»СЏ {Id}", id);
        return await _userRepo.GetByIdAsync(id);
    }
}
```

### DI СЃ primary constructors вЂ” СЂРµРєРѕРјРµРЅРґСѓРµРјС‹Р№ СЃС‚РёР»СЊ

```csharp
public sealed class CreateOrderCommandHandler(
    IOrderRepository orderRepo,
    IUnitOfWork unitOfWork,
    ILogger<CreateOrderCommandHandler> logger)
    : IRequestHandler<CreateOrderCommand, Result<int>>
{
    public async Task<Result<int>> Handle(
        CreateOrderCommand request,
        CancellationToken ct)
    {
        logger.LogInformation("РЎРѕР·РґР°РЅРёРµ Р·Р°РєР°Р·Р° РґР»СЏ РєР»РёРµРЅС‚Р° {CustomerId}", request.CustomerId);

        var order = Order.Create(request.CustomerId, request.Items);
        orderRepo.Add(order);
        await unitOfWork.SaveChangesAsync(ct).ConfigureAwait(false);

        return Result<int>.Success(order.Id);
    }
}
```

### Gotchas

```csharp
// 1. Mutable capture вЂ” РїР°СЂР°РјРµС‚СЂС‹ РќР• readonly
public class Danger(string name)
{
    public void Mutate()
    {
        name = "Changed!"; // РљРѕРјРїРёР»РёСЂСѓРµС‚СЃСЏ! РќРµС‚ РѕС€РёР±РєРё.
    }
}

// 2. Struct вЂ” primary constructor РїР°СЂР°РјРµС‚СЂС‹ С‚РѕР¶Рµ mutable
public struct Point(double x, double y)
{
    // x Рё y РјРѕР¶РЅРѕ РјРµРЅСЏС‚СЊ РІРЅСѓС‚СЂРё РјРµС‚РѕРґРѕРІ
}

// 3. РќРµ СЃРјРµС€РёРІР°Р№С‚Рµ capture Рё field assignment
public class Bad(IService service)
{
    private readonly IService _service = service;

    public void Do()
    {
        // service Рё _service вЂ” Р”Р’Рђ СЂР°Р·РЅС‹С… С…СЂР°РЅРёР»РёС‰Р°!
        // РСЃРїРѕР»СЊР·СѓР№С‚Рµ С‚РѕР»СЊРєРѕ _service.
        _service.Execute();
    }
}
```

---

## Collection Expressions (C# 12)

### Р‘Р°Р·РѕРІС‹Р№ СЃРёРЅС‚Р°РєСЃРёСЃ

```csharp
// Array
int[] numbers = [1, 2, 3, 4, 5];

// List<T>
List<string> names = ["РРІР°РЅ", "РџС‘С‚СЂ", "РђРЅРЅР°"];

// Span<T>
Span<int> span = [10, 20, 30];

// ReadOnlySpan<T>
ReadOnlySpan<byte> bytes = [0xFF, 0x00, 0xAB];

// ImmutableArray<T>
ImmutableArray<int> immutable = [1, 2, 3];

// Empty collection
List<int> empty = [];
int[] emptyArr = [];
```

### Spread operator `..`

```csharp
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];

// РћР±СЉРµРґРёРЅРµРЅРёРµ РєРѕР»Р»РµРєС†РёР№
int[] all = [..first, ..second];         // [1, 2, 3, 4, 5, 6]
int[] withExtra = [0, ..first, 99];      // [0, 1, 2, 3, 99]

// Spread РёР· Р»СЋР±РѕРіРѕ IEnumerable
IEnumerable<int> query = Enumerable.Range(1, 5);
int[] fromQuery = [..query, 100];        // [1, 2, 3, 4, 5, 100]

// Spread РІ List
List<string> combined = [..existingList, "РЅРѕРІС‹Р№ СЌР»РµРјРµРЅС‚"];
```

### Target Typing

```csharp
// РљРѕРјРїРёР»СЏС‚РѕСЂ РІС‹Р±РёСЂР°РµС‚ С‚РёРї РєРѕР»Р»РµРєС†РёРё РїРѕ РєРѕРЅС‚РµРєСЃС‚Сѓ
void Process(ReadOnlySpan<int> data) { /* ... */ }
Process([1, 2, 3]); // Р°РІС‚РѕРјР°С‚РёС‡РµСЃРєРё СЃРѕР·РґР°С‘С‚ ReadOnlySpan<int>

// Р’РѕР·РІСЂР°С‚ РёР· РјРµС‚РѕРґР°
public IReadOnlyList<string> GetDefaults() => ["default1", "default2"];

// РўРµСЂРЅР°СЂРЅС‹Р№ РѕРїРµСЂР°С‚РѕСЂ вЂ” РѕР±Р° РІР°СЂРёР°РЅС‚Р° РґРѕР»Р¶РЅС‹ Р±С‹С‚СЊ collection expression
int[] result = condition ? [1, 2] : [3, 4];
```

---

### РђСЂРіСѓРјРµРЅС‚С‹ РєРѕР»Р»РµРєС†РёРё (proposal, upcoming)

Р’ csharplang РѕР±СЃСѓР¶РґР°РµС‚СЃСЏ СЂР°СЃС€РёСЂРµРЅРёРµ: РїРµСЂРµРґР°РІР°С‚СЊ Р°СЂРіСѓРјРµРЅС‚С‹ РІ create-РјРµС‚РѕРґ РєРѕР»Р»РµРєС†РёРё РїСЂСЏРјРѕ РІРЅСѓС‚СЂРё `[ ]` вЂ” С‡С‚РѕР±С‹ РєРѕРЅС‚СЂРѕР»РёСЂРѕРІР°С‚СЊ `capacity` / `comparer` Р±РµР· РѕС‚РєР°С‚Р° Рє `new List<T>(capacity: N) { ... }`.

```csharp
// РўРµРєСѓС‰РёР№ C# вЂ” Р±РµР· СѓРїСЂР°РІР»РµРЅРёСЏ capacity РЅРµ РѕР±РѕР№С‚РёСЃСЊ
var xs = new List<int>(capacity: 32) { 1, 2, 3 };

// Proposal (РІРѕР·РјРѕР¶РЅРѕ C# 15+)
List<int> xs = [args(capacity: 32); 1, 2, 3];
//              в””в”Ђв”Ђ Р°СЂРіСѓРјРµРЅС‚С‹ РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂР° в”Ђв”  в””в”Ђв”Ђ СЌР»РµРјРµРЅС‚С‹ в”Ђв”
```

**Р—Р°С‡РµРј СЌС‚Рѕ РІР°Р¶РЅРѕ РІ hot-path РєРѕРґРµ:** РєРѕРіРґР° СЂР°Р·РјРµСЂ РєРѕР»Р»РµРєС†РёРё РёР·РІРµСЃС‚РµРЅ (РЅР°РїСЂРёРјРµСЂ, `count` РёР· Р·Р°РїСЂРѕСЃР°), `capacity` СѓР±РёСЂР°РµС‚ realloc РІРЅСѓС‚СЂРё `List<T>`. РћСЃРѕР±РµРЅРЅРѕ РєСЂРёС‚РёС‡РЅРѕ РґР»СЏ HFT / highload СЃС†РµРЅР°СЂРёРµРІ Рё batch-РѕР±СЂР°Р±РѕС‚РєРё.

Proposal: [csharplang/collection-expression-arguments](https://github.com/dotnet/csharplang/blob/main/proposals/collection-expression-arguments.md).

---

## Global Рё File-scoped

### File-scoped Namespaces (C# 10)

```csharp
// Р”Рѕ C# 10 вЂ” block-scoped
namespace MyApp.Services
{
    public class UserService
    {
        // ...
    }
}

// C# 10 вЂ” file-scoped (СЌРєРѕРЅРѕРјРёС‚ РѕРґРёРЅ СѓСЂРѕРІРµРЅСЊ РѕС‚СЃС‚СѓРїР°)
namespace MyApp.Services;

public class UserService
{
    // ...
}
```

### Global Usings (C# 10)

```csharp
// Р’ РѕС‚РґРµР»СЊРЅРѕРј С„Р°Р№Р»Рµ GlobalUsings.cs
global using System.Collections.Immutable;
global using System.Text.Json;
global using Microsoft.Extensions.Logging;
global using MediatR;

// Р’ .csproj вЂ” implicit usings
// <ImplicitUsings>enable</ImplicitUsings>
// РђРІС‚РѕРјР°С‚РёС‡РµСЃРєРё РґРѕР±Р°РІР»СЏРµС‚: System, System.Linq, System.Collections.Generic Рё С‚.Рґ.
```

```xml
<!-- РР»Рё СЏРІРЅРѕ РІ csproj -->
<ItemGroup>
    <Using Include="System.Text.Json" />
    <Using Include="Microsoft.Extensions.Logging" />
    <Using Include="MyApp.Domain.Common" />
    <!-- Static using -->
    <Using Include="System.Math" Static="true" />
    <!-- Alias -->
    <Using Include="System.Text.Json.JsonSerializer" Alias="Json" />
</ItemGroup>
```

### File-scoped Types (C# 11)

```csharp
// РўРёРї РІРёРґРµРЅ С‚РѕР»СЊРєРѕ РІРЅСѓС‚СЂРё С„Р°Р№Р»Р° вЂ” РЅРµ СЌРєСЃРїРѕСЂС‚РёСЂСѓРµС‚СЃСЏ РёР· assembly
file class InternalHelper
{
    public static int Calculate(int x) => x * 2;
}

file record struct TempResult(bool Success, string Message);

file interface ILocalProcessor
{
    void Process();
}

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ: source generators, РІСЃРїРѕРјРѕРіР°С‚РµР»СЊРЅС‹Рµ С‚РёРїС‹,
// РєРѕС‚РѕСЂС‹Рµ РЅРµ РґРѕР»Р¶РЅС‹ СѓС‚РµРєР°С‚СЊ РІ РїСѓР±Р»РёС‡РЅС‹Р№ API.
```

---

## Raw String Literals (C# 11)

### Multi-line СЃС‚СЂРѕРєРё

```csharp
// РњРёРЅРёРјСѓРј С‚СЂРё РєР°РІС‹С‡РєРё """, РјРѕР¶РЅРѕ Р±РѕР»СЊС€Рµ РїСЂРё РЅРµРѕР±С…РѕРґРёРјРѕСЃС‚Рё
string json = """
    {
        "name": "РРІР°РЅ",
        "age": 30,
        "address": {
            "city": "РњРѕСЃРєРІР°"
        }
    }
    """;
// РћС‚СЃС‚СѓРї trimming вЂ” РѕС‚СЃС‚СѓРї Р·Р°РєСЂС‹РІР°СЋС‰РёС… """ РѕРїСЂРµРґРµР»СЏРµС‚ Р±Р°Р·РѕРІС‹Р№ СѓСЂРѕРІРµРЅСЊ.
// Р’СЃС‘, С‡С‚Рѕ Р»РµРІРµРµ Р·Р°РєСЂС‹РІР°СЋС‰РёС… """ вЂ” РѕС€РёР±РєР° РєРѕРјРїРёР»СЏС†РёРё.

// SQL-Р·Р°РїСЂРѕСЃ
string sql = """
    SELECT u.Id, u.Name, o.Total
    FROM Users u
    INNER JOIN Orders o ON o.UserId = u.Id
    WHERE u.IsActive = true
    ORDER BY o.Total DESC
    LIMIT 10
    """;
```

### Interpolated raw strings

```csharp
string name = "РРІР°РЅ";
int age = 30;

// $ РїРµСЂРµРґ """ вЂ” РёРЅС‚РµСЂРїРѕР»СЏС†РёСЏ
string json = $"""
    {{
        "name": "{name}",
        "age": {age}
    }}
    """;
// {{ }} вЂ” СЌРєСЂР°РЅРёСЂРѕРІР°РЅРёРµ С„РёРіСѓСЂРЅС‹С… СЃРєРѕР±РѕРє (СѓРґРІР°РёРІР°РµРј)

// $$ вЂ” РґРІР° РґРѕР»Р»Р°СЂР°, С‚РѕРіРґР° РёРЅС‚РµСЂРїРѕР»СЏС†РёСЏ С‡РµСЂРµР· {{ }}
string template = $$"""
    {
        "name": "{{name}}",
        "jsonTemplate": "{ "key": "{value}" }"
    }
    """;
// РћРґРёРЅРѕС‡РЅС‹Рµ { } вЂ” Р»РёС‚РµСЂР°Р»СЊРЅС‹Рµ, РґР»СЏ РёРЅС‚РµСЂРїРѕР»СЏС†РёРё РЅСѓР¶РЅС‹ {{ }}
```

---

## UTF-8 String Literals (C# 11)

```csharp
// РЎСѓС„С„РёРєСЃ u8 вЂ” СЃРѕР·РґР°С‘С‚ ReadOnlySpan<byte> РІ UTF-8
ReadOnlySpan<byte> greeting = "Hello, World!"u8;

// Р—Р°С‡РµРј: zero-allocation РїСЂРё СЂР°Р±РѕС‚Рµ СЃ HTTP, JSON, РїСЂРѕС‚РѕРєРѕР»Р°РјРё
// Р”Рѕ C# 11:
byte[] header = Encoding.UTF8.GetBytes("Content-Type"); // Р°Р»Р»РѕРєР°С†РёСЏ

// C# 11:
ReadOnlySpan<byte> header = "Content-Type"u8; // compile-time, zero-alloc

// РџСЂРёРјРµСЂ: HTTP header
static ReadOnlySpan<byte> ContentTypeJson => "application/json"u8;
static ReadOnlySpan<byte> AuthorizationHeader => "Authorization"u8;

// РЎСЂР°РІРЅРµРЅРёРµ
ReadOnlySpan<byte> input = GetHeaderValue();
if (input.SequenceEqual("Bearer"u8))
{
    // ...
}
```

---

## Generic Math (C# 11)

### РРЅС‚РµСЂС„РµР№СЃС‹ РґР»СЏ generic-Р°СЂРёС„РјРµС‚РёРєРё

```csharp
// INumber<T> вЂ” Р±Р°Р·РѕРІС‹Р№ РёРЅС‚РµСЂС„РµР№СЃ РґР»СЏ С‡РёСЃР»РѕРІС‹С… РѕРїРµСЂР°С†РёР№
// IAdditionOperators, IMultiplyOperators, IComparisonOperators Рё С‚.Рґ.

// Generic РјРµС‚РѕРґ СЃСѓРјРјРёСЂРѕРІР°РЅРёСЏ вЂ” СЂР°Р±РѕС‚Р°РµС‚ СЃ int, double, decimal Рё С‚.Рґ.
public static T Sum<T>(ReadOnlySpan<T> values) where T : INumber<T>
{
    T result = T.Zero;
    foreach (var value in values)
    {
        result += value;
    }
    return result;
}

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ
int intSum = Sum<int>([1, 2, 3, 4, 5]);            // 15
double doubleSum = Sum<double>([1.5, 2.5, 3.0]);   // 7.0
decimal decSum = Sum<decimal>([10.5m, 20.3m]);      // 30.8

// Generic СЃСЂРµРґРЅРµРµ
public static T Average<T>(ReadOnlySpan<T> values)
    where T : INumber<T>
{
    T sum = Sum(values);
    return sum / T.CreateChecked(values.Length);
}

// Generic clamp
public static T Clamp<T>(T value, T min, T max)
    where T : INumber<T>
{
    if (value < min) return min;
    if (value > max) return max;
    return value;
}

// Static abstract members РІ РёРЅС‚РµСЂС„РµР№СЃР°С… (C# 11)
public interface IFactory<TSelf> where TSelf : IFactory<TSelf>
{
    static abstract TSelf Create();
    static abstract TSelf Default { get; }
}
```

---

## Params Collections (C# 13)

```csharp
// Р”Рѕ C# 13: params С‚РѕР»СЊРєРѕ СЃ РјР°СЃСЃРёРІР°РјРё
public void LogOld(params string[] messages) { /* Р°Р»Р»РѕРєР°С†РёСЏ РјР°СЃСЃРёРІР° */ }

// C# 13: params СЂР°Р±РѕС‚Р°РµС‚ СЃ Span, ReadOnlySpan, IEnumerable
public void Log(params ReadOnlySpan<string> messages)
{
    foreach (var msg in messages)
    {
        Console.WriteLine(msg);
    }
}
// Р’С‹Р·РѕРІ вЂ” zero-allocation РЅР° СЃС‚РµРєРµ!
Log("info", "Р—Р°РїСѓСЃРє РїСЂРёР»РѕР¶РµРЅРёСЏ", "OK");

// params СЃ IEnumerable<T>
public int Sum(params IEnumerable<int> numbers) => numbers.Sum();

// params СЃ List<T>
public void Process(params List<string> items)
{
    foreach (var item in items) { /* ... */ }
}

// Р Р°Р·СЂРµС€РµРЅРёРµ РїРµСЂРµРіСЂСѓР·РѕРє: ReadOnlySpan > Span > Array > IEnumerable
```

---

## Lock Object (C# 13)

```csharp
// Р”Рѕ C# 13 вЂ” lock РЅР° object
private readonly object _syncRoot = new();

public void ThreadSafeMethod()
{
    lock (_syncRoot)
    {
        // critical section
    }
}

// C# 13 вЂ” System.Threading.Lock
// Р‘РѕР»РµРµ СЌС„С„РµРєС‚РёРІРЅР°СЏ СЂРµР°Р»РёР·Р°С†РёСЏ, Р·Р°С‚РѕС‡РµРЅРЅР°СЏ РїРѕРґ lock
private readonly Lock _lock = new();

public void ThreadSafeMethod()
{
    lock (_lock) // РєРѕРјРїРёР»СЏС‚РѕСЂ РёСЃРїРѕР»СЊР·СѓРµС‚ Lock.EnterScope()
    {
        // critical section
    }
}

// Р СѓС‡РЅРѕРµ СѓРїСЂР°РІР»РµРЅРёРµ (Scope вЂ” ref struct, Dispose РІС‹Р·С‹РІР°РµС‚ Exit)
public void ManualLock()
{
    using (_lock.EnterScope())
    {
        // critical section
    }
}

// РџСЂРµРёРјСѓС‰РµСЃС‚РІР° Lock vs object:
// 1. РќРµС‚ boxing/unboxing
// 2. РљРѕРјРїРёР»СЏС‚РѕСЂ РіРµРЅРµСЂРёСЂСѓРµС‚ Р±РѕР»РµРµ РѕРїС‚РёРјР°Р»СЊРЅС‹Р№ РєРѕРґ
// 3. РЎРµРјР°РЅС‚РёС‡РµСЃРєРё СЏСЃРЅС‹Р№ С‚РёРї вЂ” РїРѕРЅСЏС‚РЅРѕ РЅР°Р·РЅР°С‡РµРЅРёРµ
// 4. Ref struct scope вЂ” РЅРµ РјРѕР¶РµС‚ СѓС‚РµС‡СЊ РІ async РєРѕРЅС‚РµРєСЃС‚
```

---

## C# 14 Preview

### Extension Members

```csharp
// Р”Рѕ C# 14 вЂ” С‚РѕР»СЊРєРѕ extension methods (static class)
public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string? s) => string.IsNullOrEmpty(s);
}

// C# 14 вЂ” extension Р±Р»РѕРє: properties, indexers, static members
public extension StringExtensions for string
{
    // Extension property
    public bool IsEmpty => this.Length == 0;

    // Extension indexer
    public char FromEnd[int index] => this[^index];

    // Extension static method
    public static string Empty => string.Empty;
}

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ
string name = "Hello";
bool empty = name.IsEmpty;       // false
char last = name.FromEnd[1];     // 'o'
```

### field keyword РІ auto-properties

```csharp
// Р”Рѕ C# 14 вЂ” РЅСѓР¶РЅРѕ backing field РґР»СЏ РІР°Р»РёРґР°С†РёРё
private string _name = "";
public string Name
{
    get => _name;
    set
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(value);
        _name = value;
    }
}

// C# 14 вЂ” field keyword СЃСЃС‹Р»Р°РµС‚СЃСЏ РЅР° Р°РІС‚РѕРјР°С‚РёС‡РµСЃРєРѕРµ backing field
public string Name
{
    get => field;
    set
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(value);
        field = value;
    }
}

// РЈРґРѕР±РЅРѕ РґР»СЏ lazy initialization
public List<Order> Orders
{
    get => field ??= [];
}

// РЈРґРѕР±РЅРѕ РґР»СЏ change notification (INotifyPropertyChanged)
public string Title
{
    get => field;
    set => SetProperty(ref field, value);
}
```

### Null-conditional Assignment

```csharp
// Р”Рѕ C# 14
if (customer?.Address is not null)
{
    customer.Address.City = "РњРѕСЃРєРІР°";
}

// C# 14 вЂ” null-conditional assignment
customer?.Address?.City = "РњРѕСЃРєРІР°";
// Р•СЃР»Рё customer == null РР›Р Address == null вЂ” РїСЂРёСЃРІРѕРµРЅРёРµ РЅРµ РІС‹РїРѕР»РЅСЏРµС‚СЃСЏ.
// NullReferenceException РЅРµ РІРѕР·РЅРёРєР°РµС‚. Р‘РµР· РІС‚РѕСЂРѕРіРѕ ?. РїСЂРё Address == null Р±СѓРґРµС‚ NRE!

// Р Р°Р±РѕС‚Р°РµС‚ Рё СЃ РёРЅРґРµРєСЃР°С‚РѕСЂР°РјРё
list?[0] = newValue;
dictionary?["key"] = newValue;
```

### Lambda Parameter Modifiers

```csharp
// C# 14 вЂ” РјРѕР¶РЅРѕ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ ref, in, out РІ Р»СЏРјР±РґР°С… СЃ СЏРІРЅС‹РјРё С‚РёРїР°РјРё
Span<int> numbers = [3, 1, 4, 1, 5];
numbers.Sort((ref int a, ref int b) => a.CompareTo(b));
```

### Partial Constructors Рё Events (РґРѕРїРѕР»РЅРµРЅРёРµ Рє partial methods/properties)

```csharp
// Partial constructor вЂ” С‡Р°СЃС‚СЊ СЂРµР°Р»РёР·Р°С†РёРё РІ РѕРґРЅРѕРј С„Р°Р№Р»Рµ, С‡Р°СЃС‚СЊ РІ РґСЂСѓРіРѕРј
public partial class ViewModel
{
    public partial ViewModel(string title);
}

// Р’ РґСЂСѓРіРѕРј С„Р°Р№Р»Рµ (РЅР°РїСЂРёРјРµСЂ, generated code)
public partial class ViewModel
{
    public partial ViewModel(string title)
    {
        Title = title;
        InitializeCommands();
    }
}
```

---

## File-based Apps (.NET 11 Preview)

### Р§С‚Рѕ

РќР°С‡РёРЅР°СЏ СЃ **.NET 11 Preview 3** РјРѕР¶РЅРѕ РїРёСЃР°С‚СЊ РїРѕР»РЅРѕС†РµРЅРЅРѕРµ ASP.NET Core Web API РІ **РѕРґРЅРѕРј `.cs` С„Р°Р№Р»Рµ** Р±РµР· `.csproj` / `.sln`. Go-like РјРёРЅРёРјР°Р»РёР·Рј, Р±РµР· РѕС‚РєР°Р·Р° РѕС‚ С‚РёРїРёР·Р°С†РёРё Рё СЌРєРѕСЃРёСЃС‚РµРјС‹ .NET.

```csharp
// app.cs
#:package Microsoft.AspNetCore.OpenApi@9.0.0

var builder = WebApplication.CreateBuilder();
var app = builder.Build();

app.MapGet("/hello/{name}", (string name) => $"Hi, {name}!");

app.Run();
```

Р—Р°РїСѓСЃРє Рё РїСѓР±Р»РёРєР°С†РёСЏ:

```bash
dotnet run app.cs                  # РїСЂСЏРјРѕР№ Р·Р°РїСѓСЃРє
dotnet publish app.cs --aot -o out # Native AOT в†’ ~30 MB single binary
```

### Р—Р°С‡РµРј

| РЎС†РµРЅР°СЂРёР№ | РџРѕС‡РµРјСѓ fit |
|----------|-----------|
| CLI-СѓС‚РёР»РёС‚С‹ | РўРёРїРёР·Р°С†РёСЏ + DI + РїР°РєРµС‚С‹ NuGet, РЅРѕ Р±РµР· ceremony. Р—Р°РјРµРЅР° PowerShell/Bash-СЃРєСЂРёРїС‚Р°Рј. |
| РџСЂРѕС‚РѕС‚РёРїС‹ РґР»СЏ РєР»РёРµРЅС‚РѕРІ / demo | РћРґРёРЅ С„Р°Р№Р» в†’ РїРѕРєР°Р·Р°Р» Р·Р°РєР°Р·С‡РёРєСѓ в†’ `dotnet run` |
| Micro-services РїРѕРґ Native AOT | РњРёРЅРёРјР°Р»СЊРЅС‹Р№ РѕР±СЂР°Р·, Р±С‹СЃС‚СЂС‹Р№ СЃС‚Р°СЂС‚ |
| РћРЅР±РѕСЂРґРёРЅРі РЅРѕРІРёС‡РєРѕРІ РІ .NET | РњРµРЅСЊС€Рµ С€СѓРјР° РІ РЅР°С‡Р°Р»Рµ вЂ” РЅРµС‚ XML, РЅРµС‚ solution, РЅРµС‚ РјРЅРѕРіРѕРїСЂРѕРµРєС‚РЅРѕР№ СЃС‚СЂСѓРєС‚СѓСЂС‹ |

### РћРіСЂР°РЅРёС‡РµРЅРёСЏ

- РњРёРіСЂР°С†РёРё EF Core С‚СЂРµР±СѓСЋС‚ РѕР±С…РѕРґРЅС‹С… СЂРµС€РµРЅРёР№ вЂ” РЅРµС‚ `.csproj`, РєРѕС‚РѕСЂС‹Р№ РЅСѓР¶РµРЅ design-time tools
- РўРµСЃС‚РёСЂРѕРІР°РЅРёРµ РІСЃС‚СЂРѕРµРЅРЅРѕРµ РЅРµ РїСЂРµРґСѓСЃРјРѕС‚СЂРµРЅРѕ вЂ” РЅСѓР¶РЅРѕ РІС‹РЅРѕСЃРёС‚СЊ РІ РѕС‚РґРµР»СЊРЅС‹Р№ РїСЂРѕРµРєС‚
- Р’ PR-Р°С… С…СѓР¶Рµ РїРѕРєР°Р· РёР·РјРµРЅРµРЅРёР№ РІ NuGet-РїР°РєРµС‚Р°С… (Р·Р°РІРёСЃРёРјРѕСЃС‚Рё РІ `#:package` РґРёСЂРµРєС‚РёРІР°С…, Р° РЅРµ РІ РѕС‚РґРµР»СЊРЅРѕРј `.csproj`)
- Production-grade СЂРµС€РµРЅРёСЏ РІСЃС‘ Р¶Рµ РѕСЃС‚Р°СЋС‚СЃСЏ СЃ РїРѕР»РЅРѕС†РµРЅРЅС‹Рј `.csproj` вЂ” СЌС‚Рѕ РґР»СЏ СЃРєСЂРёРїС‚РѕРІ Рё РїСЂРѕС‚РѕС‚РёРїРѕРІ

### РџСЂРёРјРµСЂ РїСЂРѕРґ-РїРѕР»РµР·РЅРѕРіРѕ: CLI-СѓС‚РёР»РёС‚Р° РґР»СЏ СЃРІРѕРёС… VPS

```csharp
// deploy.cs
#:package System.CommandLine@2.0.0-beta4
#:package CliWrap@3.6.6

using System.CommandLine;
using CliWrap;

var repoArg = new Argument<string>("repo");
var root = new RootCommand("Deploy helper") { repoArg };

root.SetHandler(async (string repo) =>
{
    await Cli.Wrap("git").WithArguments("pull").ExecuteAsync();
    await Cli.Wrap("docker").WithArguments("compose up -d --build").ExecuteAsync();
    Console.WriteLine($"Deployed {repo}");
}, repoArg);

await root.InvokeAsync(args);
```

Р—Р°РїСѓСЃРє: `dotnet run deploy.cs myapp`.

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: Р§С‚Рѕ РґР°С‘С‚ .NET 11 РїРѕРјРёРјРѕ РїСЂРѕРёР·РІРѕРґРёС‚РµР»СЊРЅРѕСЃС‚Рё?**
> РР· РїСЂР°РєС‚РёС‡РЅРѕРіРѕ: file-based apps (`dotnet run app.cs`) вЂ” .NET РїРµСЂРµСЃС‚Р°Р» РІС‹РіР»СЏРґРµС‚СЊ РєР°Рє Windows-only enterprise Рё РїРѕРґС‚СЏРЅСѓР»СЃСЏ Рє Go/Python РїРѕ РїРѕСЂРѕРіСѓ РІС…РѕРґР°. Р Р°Р±РѕС‚Р°РµС‚ СЃ Native AOT в†’ 30 MB single binary, СѓРґРѕР±РЅРѕ РґР»СЏ CLI Рё РјРёРєСЂРѕСЃРµСЂРІРёСЃРѕРІ. Р”Р»СЏ РїСЂРѕРґР° СЃ РјРёРіСЂР°С†РёСЏРјРё Рё РјРЅРѕРіРѕРєРѕРјРїРѕРЅРµРЅС‚РЅРѕР№ Р°СЂС…РёС‚РµРєС‚СѓСЂРѕР№ вЂ” РїРѕ-РїСЂРµР¶РЅРµРјСѓ `.csproj`, РЅРѕ РїСЂРѕС‚РѕС‚РёРїС‹, СЃРєСЂРёРїС‚С‹ Рё РґРµРјРѕ С‚РµРїРµСЂСЊ РїРёС€СѓС‚СЃСЏ РЅР° C# С‚Р°Рє Р¶Рµ Р±С‹СЃС‚СЂРѕ, РєР°Рє РЅР° Python.

---

## Р”РѕРїРѕР»РЅРёС‚РµР»СЊРЅС‹Рµ С„РёС‡Рё РїРѕ РІРµСЂСЃРёСЏРј

### Using Declarations (C# 8)

```csharp
// Р”Рѕ C# 8 вЂ” using block
using (var stream = File.OpenRead("file.txt"))
{
    // ...
} // Dispose Р·РґРµСЃСЊ

// C# 8 вЂ” using declaration (Р±РµР· Р±Р»РѕРєР°)
using var stream = File.OpenRead("file.txt");
// ... РёСЃРїРѕР»СЊР·СѓРµРј stream
// Dispose РїСЂРё РІС‹С…РѕРґРµ РёР· scope (РјРµС‚РѕРґ, Р±Р»РѕРє if/for Рё С‚.Рґ.)
```

### Async Streams (C# 8)

```csharp
// IAsyncEnumerable<T> вЂ” Р°СЃРёРЅС…СЂРѕРЅРЅР°СЏ РёС‚РµСЂР°С†РёСЏ
public async IAsyncEnumerable<Order> GetOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    int page = 0;
    while (true)
    {
        var batch = await _repo.GetPageAsync(page++, 100, ct).ConfigureAwait(false);
        if (batch.Count == 0) yield break;

        foreach (var order in batch)
        {
            yield return order;
        }
    }
}

// РџРѕС‚СЂРµР±Р»РµРЅРёРµ
await foreach (var order in GetOrdersAsync(ct))
{
    Process(order);
}
```

### Target-typed new (C# 9)

```csharp
// РўРёРї РІС‹РІРѕРґРёС‚СЃСЏ РёР· РєРѕРЅС‚РµРєСЃС‚Р°
Dictionary<string, List<int>> map = new();
List<string> names = new() { "РРІР°РЅ", "РџС‘С‚СЂ" };

// Р’ Р°СЂРіСѓРјРµРЅС‚Р°С… РјРµС‚РѕРґР°
Process(new OrderOptions { Timeout = TimeSpan.FromSeconds(30) });

// Р’ return
public CancellationTokenSource CreateCts() => new(TimeSpan.FromMinutes(5));
```

### Top-level Statements (C# 9)

```csharp
// Р¤Р°Р№Р» Р±РµР· РєР»Р°СЃСЃР° Program Рё РјРµС‚РѕРґР° Main
using Microsoft.Extensions.Hosting;

var builder = Host.CreateDefaultBuilder(args);
// ...
var app = builder.Build();
await app.RunAsync();
```

### Constant Interpolated Strings (C# 10)

```csharp
// РРЅС‚РµСЂРїРѕР»СЏС†РёСЏ РІ const вЂ” РµСЃР»Рё РІСЃРµ С‡Р°СЃС‚Рё const
const string Scheme = "https";
const string Host = "api.example.com";
const string BaseUrl = $"{Scheme}://{Host}"; // OK РІ C# 10
```

### Alias Any Type (C# 12)

```csharp
// using alias РґР»СЏ Р»СЋР±РѕРіРѕ С‚РёРїР°, РІРєР»СЋС‡Р°СЏ tuples Рё generics
using Point = (double X, double Y);
using OrderId = int;
using JsonDict = System.Collections.Generic.Dictionary<string, System.Text.Json.JsonElement>;

Point origin = (0, 0);
OrderId id = 42;
JsonDict data = new() { ["key"] = JsonDocument.Parse("1").RootElement };
```

### Default Lambda Parameters (C# 12)

```csharp
// РџР°СЂР°РјРµС‚СЂС‹ РїРѕ СѓРјРѕР»С‡Р°РЅРёСЋ РІ Р»СЏРјР±РґР°С…
var greet = (string name, string greeting = "РџСЂРёРІРµС‚") => $"{greeting}, {name}!";
Console.WriteLine(greet("РРІР°РЅ"));            // РџСЂРёРІРµС‚, РРІР°РЅ!
Console.WriteLine(greet("РРІР°РЅ", "Р—РґСЂР°РІСЃС‚РІСѓР№")); // Р—РґСЂР°РІСЃС‚РІСѓР№, РРІР°РЅ!
```

### ref struct Interfaces (C# 13)

```csharp
// ref struct С‚РµРїРµСЂСЊ РјРѕР¶РµС‚ СЂРµР°Р»РёР·РѕРІС‹РІР°С‚СЊ РёРЅС‚РµСЂС„РµР№СЃС‹ (СЃ РѕРіСЂР°РЅРёС‡РµРЅРёСЏРјРё)
public interface IBufferWriter
{
    void Write(ReadOnlySpan<byte> data);
}

public ref struct StackBuffer : IBufferWriter
{
    private Span<byte> _buffer;
    private int _position;

    public void Write(ReadOnlySpan<byte> data)
    {
        data.CopyTo(_buffer[_position..]);
        _position += data.Length;
    }
}

// РћРіСЂР°РЅРёС‡РµРЅРёРµ: РЅРµР»СЊР·СЏ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ С‡РµСЂРµР· boxing (IBufferWriter РїРµСЂРµРјРµРЅРЅР°СЏ)
// РўРѕР»СЊРєРѕ С‡РµСЂРµР· generics СЃ allows ref struct constraint
public static void WriteAll<T>(T writer, ReadOnlySpan<byte> data)
    where T : IBufferWriter, allows ref struct
{
    writer.Write(data);
}
```

### Escape Sequence `\e` (C# 13)

```csharp
// \e = ESC (0x1B) вЂ” РґР»СЏ ANSI escape codes РІ С‚РµСЂРјРёРЅР°Р»Рµ
Console.WriteLine("\e[31mРљСЂР°СЃРЅС‹Р№ С‚РµРєСЃС‚\e[0m");
Console.WriteLine("\e[1;32mР–РёСЂРЅС‹Р№ Р·РµР»С‘РЅС‹Р№\e[0m");

// Р”Рѕ C# 13: '\x1B' РёР»Рё '\u001B'
```

---

## РўР°Р±Р»РёС†Р° С„РёС‡ РїРѕ РІРµСЂСЃРёСЏРј

| Р’РµСЂСЃРёСЏ | Р“РѕРґ  | .NET   | РљР»СЋС‡РµРІС‹Рµ С„РёС‡Рё                                                                    |
|--------|------|--------|-----------------------------------------------------------------------------------|
| C# 8   | 2019 | Core 3 | Nullable refs, pattern matching, default interface methods, using declarations, async streams, indices/ranges |
| C# 9   | 2020 | 5      | Records, init-only, top-level statements, target-typed new, logical patterns      |
| C# 10  | 2021 | 6      | Global usings, file-scoped namespaces, record structs, const interpolation, extended property patterns |
| C# 11  | 2022 | 7      | Raw strings, required, list patterns, UTF-8 literals, generic math, file-scoped types, static abstract members |
| C# 12  | 2023 | 8      | Primary constructors, collection expressions, default lambda params, alias any type, inline arrays |
| C# 13  | 2024 | 9      | params collections, Lock type, ref struct interfaces, escape `\e`, method group natural type improvements |
| C# 14  | 2025 | 10     | Extension members, field keyword, null-conditional assignment, partial constructors |

---

## РЎРј. С‚Р°РєР¶Рµ

- [РўРёРїС‹ Рё РїР°РјСЏС‚СЊ](types-and-memory.md)
- [РћРћРџ Рё РєР»Р°СЃСЃС‹](oop.md)
- [Delegates Рё Events](delegates-events.md)
- [Collections Рё LINQ](collections-linq.md)
- [Async Рё РїРѕС‚РѕРєРё](async-threading.md)
