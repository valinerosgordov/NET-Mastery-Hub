---
tags: [oop, classes, interfaces, inheritance, records]
level: Junior to Senior
---

# РћРћРџ Рё РєР»Р°СЃСЃС‹

## Р§С‚Рѕ СЌС‚Рѕ, Р·Р°С‡РµРј Рё РєРѕРіРґР°

### Р§С‚Рѕ С‚Р°РєРѕРµ РћРћРџ?
**РЎРїРѕСЃРѕР± РѕСЂРіР°РЅРёР·Р°С†РёРё РєРѕРґР° С‡РµСЂРµР· РѕР±СЉРµРєС‚С‹** вЂ” РєР°Р¶РґС‹Р№ РѕР±СЉРµРєС‚ РёРјРµРµС‚ РґР°РЅРЅС‹Рµ (СЃРІРѕР№СЃС‚РІР°) Рё РїРѕРІРµРґРµРЅРёРµ (РјРµС‚РѕРґС‹). Р§РµС‚С‹СЂРµ СЃС‚РѕР»РїР°: РёРЅРєР°РїСЃСѓР»СЏС†РёСЏ, РЅР°СЃР»РµРґРѕРІР°РЅРёРµ, РїРѕР»РёРјРѕСЂС„РёР·Рј, Р°Р±СЃС‚СЂР°РєС†РёСЏ.

**РђРЅР°Р»РѕРіРёСЏ:** РљРѕРЅСЃС‚СЂСѓРєС‚РѕСЂ LEGO. РљР°Р¶РґС‹Р№ РєСѓР±РёРє (РѕР±СЉРµРєС‚) вЂ” СЃР°РјРѕСЃС‚РѕСЏС‚РµР»СЊРЅС‹Р№, РёРјРµРµС‚ С„РѕСЂРјСѓ (СЃРІРѕР№СЃС‚РІР°) Рё СЃРїРѕСЃРѕР±С‹ РєСЂРµРїР»РµРЅРёСЏ (РјРµС‚РѕРґС‹). РР· РєСѓР±РёРєРѕРІ СЃРѕР±РёСЂР°РµС€СЊ С‡С‚Рѕ СѓРіРѕРґРЅРѕ.

### Р—Р°С‡РµРј?

| РџСЂРёРЅС†РёРї | Р§С‚Рѕ РґР°С‘С‚ | РџСЂРёРјРµСЂ |
|---------|----------|--------|
| **РРЅРєР°РїСЃСѓР»СЏС†РёСЏ** | РЎРєСЂС‹РІР°РµС€СЊ РґРµС‚Р°Р»Рё, РїРѕРєР°Р·С‹РІР°РµС€СЊ С‚РѕР»СЊРєРѕ РЅСѓР¶РЅРѕРµ | `private decimal _balance` + `public void Withdraw()` |
| **РќР°СЃР»РµРґРѕРІР°РЅРёРµ** | РџРµСЂРµРёСЃРїРѕР»СЊР·СѓРµС€СЊ РєРѕРґ Р±Р°Р·РѕРІРѕРіРѕ РєР»Р°СЃСЃР° | `Animal` в†’ `Dog`, `Cat` |
| **РџРѕР»РёРјРѕСЂС„РёР·Рј** | РћРґРёРЅ РёРЅС‚РµСЂС„РµР№СЃ, СЂР°Р·РЅС‹Рµ СЂРµР°Р»РёР·Р°С†РёРё | `IPayment.Process()` вЂ” РєР°СЂС‚Р° РёР»Рё РїРµСЂРµРІРѕРґ |
| **РђР±СЃС‚СЂР°РєС†РёСЏ** | Р Р°Р±РѕС‚Р°РµС€СЊ СЃ В«С‡С‚Рѕ РґРµР»Р°РµС‚В», РЅРµ В«РєР°Рє РґРµР»Р°РµС‚В» | `ILogger.Log()` вЂ” РЅРµ РІР°Р¶РЅРѕ, РєСѓРґР° РїРёС€РµС‚ |

### РљРѕРіРґР° С‡С‚Рѕ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ?

| РљРѕРЅСЃС‚СЂСѓРєС†РёСЏ | РљРѕРіРґР° | РљРѕРіРґР° РќР• РЅСѓР¶РЅРѕ |
|-------------|-------|----------------|
| **class** | РћР±СЉРµРєС‚ СЃ РёРґРµРЅС‚РёС‡РЅРѕСЃС‚СЊСЋ Рё РїРѕРІРµРґРµРЅРёРµРј | РџСЂРѕСЃС‚С‹Рµ РґР°РЅРЅС‹Рµ Р±РµР· Р»РѕРіРёРєРё |
| **interface** | РљРѕРЅС‚СЂР°РєС‚, РЅРµСЃРєРѕР»СЊРєРѕ СЂРµР°Р»РёР·Р°С†РёР№, DI | РћРґРЅР° СЂРµР°Р»РёР·Р°С†РёСЏ РЅР°РІСЃРµРіРґР° |
| **abstract class** | РћР±С‰Р°СЏ Р»РѕРіРёРєР° + С€Р°Р±Р»РѕРЅ РґР»СЏ РїРѕРґРєР»Р°СЃСЃРѕРІ | Р•СЃР»Рё РґРѕСЃС‚Р°С‚РѕС‡РЅРѕ interface |
| **sealed class** | РџРѕ СѓРјРѕР»С‡Р°РЅРёСЋ РІСЃС‘ sealed (Р·Р°РїСЂРµС‚ РЅР°СЃР»РµРґРѕРІР°РЅРёСЏ) | РўРѕР»СЊРєРѕ РµСЃР»Рё РЅСѓР¶РЅРѕ РЅР°СЃР»РµРґРѕРІР°РЅРёРµ |
| **record** | DTO, Value Object, immutable РґР°РЅРЅС‹Рµ | Mutable РѕР±СЉРµРєС‚С‹ СЃ РїРѕРІРµРґРµРЅРёРµРј |

---

> РЎРїСЂР°РІРѕС‡РЅРёРє РїРѕ РѕР±СЉРµРєС‚РЅРѕ-РѕСЂРёРµРЅС‚РёСЂРѕРІР°РЅРЅРѕРјСѓ РїСЂРѕРіСЂР°РјРјРёСЂРѕРІР°РЅРёСЋ РІ C# 13.
> РўРµРѕСЂРёСЏ в†’ РїСЂР°РєС‚РёРєР° в†’ senior-level РєРѕРґ в†’ РІРѕРїСЂРѕСЃС‹ РёРЅС‚РµСЂРІСЊСЋ.

---

## РљР»Р°СЃСЃС‹

### РћР±СЉСЏРІР»РµРЅРёРµ РєР»Р°СЃСЃР°, РїРѕР»СЏ, СЃРІРѕР№СЃС‚РІР°, РјРµС‚РѕРґС‹

```csharp
namespace Domain.Models;

public class Product
{
    // РџРѕР»Рµ (field) вЂ” С…СЂР°РЅРёС‚ СЃРѕСЃС‚РѕСЏРЅРёРµ
    private decimal _price;

    // РЎРІРѕР№СЃС‚РІРѕ (property) вЂ” РєРѕРЅС‚СЂРѕР»РёСЂСѓРµРјС‹Р№ РґРѕСЃС‚СѓРї Рє РґР°РЅРЅС‹Рј
    public string Name { get; set; } = string.Empty;

    // Read-only СЃРІРѕР№СЃС‚РІРѕ СЃ backing field
    public decimal Price
    {
        get => _price;
        set => _price = value >= 0
            ? value
            : throw new ArgumentOutOfRangeException(nameof(value));
    }

    // РњРµС‚РѕРґ вЂ” РїРѕРІРµРґРµРЅРёРµ РѕР±СЉРµРєС‚Р°
    public decimal CalculateDiscount(decimal percentage)
        => Price * (percentage / 100m);

    // Expression-bodied РјРµС‚РѕРґ
    public override string ToString() => $"{Name}: {Price:C}";
}
```

### Access Modifiers

| Modifier             | РљР»Р°СЃСЃ | РЎР±РѕСЂРєР° | РќР°СЃР»РµРґРЅРёРє (С‚Р° Р¶Рµ СЃР±РѕСЂРєР°) | РќР°СЃР»РµРґРЅРёРє (РґСЂСѓРіР°СЏ СЃР±РѕСЂРєР°) | Р’СЃРµ |
| -------------------- | ----- | ------ | ------------------------ | ------------------------- | --- |
| `public`             | +     | +      | +                        | +                         | +   |
| `private`            | +     | -      | -                        | -                         | -   |
| `protected`          | +     | -      | +                        | +                         | -   |
| `internal`           | +     | +      | +                        | -                         | -   |
| `protected internal` | +     | +      | +                        | +                         | -   |
| `private protected`  | +     | -      | +                        | -                         | -   |

```csharp
namespace Domain.Models;

public class Account
{
    public string Owner { get; init; } = string.Empty;           // Р’РёРґРµРЅ РІСЃРµРј
    private decimal _balance;                                     // РўРѕР»СЊРєРѕ РІРЅСѓС‚СЂРё РєР»Р°СЃСЃР°
    protected int TransactionCount { get; set; }                  // РљР»Р°СЃСЃ + РЅР°СЃР»РµРґРЅРёРєРё
    internal string InternalCode { get; set; } = string.Empty;   // РўРѕР»СЊРєРѕ РІ РїСЂРµРґРµР»Р°С… СЃР±РѕСЂРєРё
    protected internal bool IsActive { get; set; }                // РЎР±РѕСЂРєР° РР›Р РЅР°СЃР»РµРґРЅРёРє
    private protected byte Priority { get; set; }                 // РќР°СЃР»РµРґРЅРёРє РІ РўРћР™ Р–Р• СЃР±РѕСЂРєРµ
}
```

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: РџРµСЂРµС‡РёСЃР»РёС‚Рµ РјРѕРґРёС„РёРєР°С‚РѕСЂС‹ РґРѕСЃС‚СѓРїР°. РљРѕРіРґР° internal?**
> `public`, `private`, `protected`, `internal`, `protected internal`, `private protected`, `file` (C# 11).
> `internal` вЂ” РґРѕСЃС‚СѓРїРµРЅ С‚РѕР»СЊРєРѕ РІРЅСѓС‚СЂРё СЃР±РѕСЂРєРё. РСЃРїРѕР»СЊР·СѓРµС‚СЃСЏ РґР»СЏ РёРЅС„СЂР°СЃС‚СЂСѓРєС‚СѓСЂРЅС‹С… РєР»Р°СЃСЃРѕРІ, РєРѕС‚РѕСЂС‹Рµ РЅРµ РґРѕР»Р¶РЅС‹ Р±С‹С‚СЊ С‡Р°СЃС‚СЊСЋ РїСѓР±Р»РёС‡РЅРѕРіРѕ API. `protected internal` вЂ” РЅР°СЃР»РµРґРЅРёРєР°Рј РР›Р РєРѕРґСѓ РІ С‚РѕР№ Р¶Рµ СЃР±РѕСЂРєРµ.

### РљРѕРЅСЃС‚СЂСѓРєС‚РѕСЂС‹

#### Default РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂ

```csharp
namespace Domain.Models;

public class Customer
{
    public string Name { get; set; } = "Unknown";
    public string Email { get; set; } = string.Empty;

    // Р•СЃР»Рё РЅРµ РѕР±СЉСЏРІРёС‚СЊ РЅРёРєР°РєРѕР№ РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂ, РєРѕРјРїРёР»СЏС‚РѕСЂ СЃРіРµРЅРµСЂРёСЂСѓРµС‚ РїСѓСЃС‚РѕР№ default.
    // Р•СЃР»Рё РѕР±СЉСЏРІРёС‚СЊ Р»СЋР±РѕР№ вЂ” default РќР• РіРµРЅРµСЂРёСЂСѓРµС‚СЃСЏ Р°РІС‚РѕРјР°С‚РёС‡РµСЃРєРё.
}
```

#### РџР°СЂР°РјРµС‚СЂРёР·РѕРІР°РЅРЅС‹Р№ РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂ

```csharp
namespace Domain.Models;

public class Order
{
    public Guid Id { get; }
    public string CustomerName { get; }
    public DateTime CreatedAt { get; }

    public Order(string customerName)
    {
        Id = Guid.NewGuid();
        CustomerName = customerName;
        CreatedAt = DateTime.UtcNow;
    }

    // РџРµСЂРµРіСЂСѓР·РєР° РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂР° СЃ chaining С‡РµСЂРµР· this(...)
    public Order(string customerName, DateTime createdAt)
        : this(customerName)
    {
        CreatedAt = createdAt;
    }
}
```

#### Static РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂ

```csharp
namespace Infrastructure.Configuration;

public class AppSettings
{
    // Р’С‹Р·С‹РІР°РµС‚СЃСЏ РћР”РРќ СЂР°Р·, РґРѕ РїРµСЂРІРѕРіРѕ РѕР±СЂР°С‰РµРЅРёСЏ Рє С‚РёРїСѓ.
    // РќРµР»СЊР·СЏ РІС‹Р·РІР°С‚СЊ РІСЂСѓС‡РЅСѓСЋ, РЅРµС‚ access modifier, РЅРµС‚ РїР°СЂР°РјРµС‚СЂРѕРІ.
    static AppSettings()
    {
        DefaultTimeout = TimeSpan.FromSeconds(30);
        MachineName = Environment.MachineName;
    }

    public static TimeSpan DefaultTimeout { get; }
    public static string MachineName { get; }
}
```

#### Primary Constructor (C# 12)

РџР°СЂР°РјРµС‚СЂС‹ primary constructor вЂ” СЌС‚Рѕ **РЅРµ СЃРІРѕР№СЃС‚РІР° Рё РЅРµ РїРѕР»СЏ**, Р° captured РїР°СЂР°РјРµС‚СЂС‹, РґРѕСЃС‚СѓРїРЅС‹Рµ РІРѕ РІСЃС‘Рј С‚РµР»Рµ РєР»Р°СЃСЃР°.

```csharp
namespace Application.Services;

// РџР°СЂР°РјРµС‚СЂС‹ primary constructor Р·Р°С…РІР°С‚С‹РІР°СЋС‚СЃСЏ РєР°Рє РїРµСЂРµРјРµРЅРЅС‹Рµ.
// РРґРµР°Р»СЊРЅРѕ РґР»СЏ DI вЂ” Р»Р°РєРѕРЅРёС‡РЅРѕ, Р±РµР· boilerplate.
public class OrderService(
    IOrderRepository orderRepository,
    ILogger<OrderService> logger)
{
    public async Task<Result<Order>> GetByIdAsync(Guid id, CancellationToken ct = default)
    {
        logger.LogInformation("РџРѕР»СѓС‡РµРЅРёРµ Р·Р°РєР°Р·Р° {OrderId}", id);
        var order = await orderRepository.GetByIdAsync(id, ct).ConfigureAwait(false);

        return order is null
            ? Result<Order>.Failure($"Р—Р°РєР°Р· {id} РЅРµ РЅР°Р№РґРµРЅ")
            : Result<Order>.Success(order);
    }
}
```

> **Р’Р°Р¶РЅРѕ:** РїР°СЂР°РјРµС‚СЂС‹ primary constructor РјСѓС‚Р°Р±РµР»СЊРЅС‹. Р•СЃР»Рё РЅСѓР¶РЅР° РёРјРјСѓС‚Р°Р±РµР»СЊРЅРѕСЃС‚СЊ вЂ” РїСЂРёСЃРІРѕРёС‚СЊ `readonly` РїРѕР»СЋ РІСЂСѓС‡РЅСѓСЋ:

```csharp
namespace Application.Services;

public class PaymentService(IPaymentGateway gateway)
{
    // РЇРІРЅРѕРµ readonly РїРѕР»Рµ вЂ” РіР°СЂР°РЅС‚РёСЏ РёРјРјСѓС‚Р°Р±РµР»СЊРЅРѕСЃС‚Рё
    private readonly IPaymentGateway _gateway = gateway;

    public Task ChargeAsync(decimal amount)
        => _gateway.ChargeAsync(amount);
}
```

### Р”РµРєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂС‹ (Deconstruct)

РњРµС‚РѕРґ `Deconstruct` РїРѕР·РІРѕР»СЏРµС‚ СЂР°Р·Р»РѕР¶РёС‚СЊ РѕР±СЉРµРєС‚ РЅР° СЃРѕСЃС‚Р°РІРЅС‹Рµ С‡Р°СЃС‚Рё С‡РµСЂРµР· tuple syntax.

```csharp
namespace Domain.Models;

public class Point(double x, double y)
{
    public double X { get; } = x;
    public double Y { get; } = y;

    // Р”РµРєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂ вЂ” РЅРµ РїСѓС‚Р°С‚СЊ СЃ С„РёРЅР°Р»РёР·Р°С‚РѕСЂРѕРј!
    public void Deconstruct(out double x, out double y)
    {
        x = X;
        y = Y;
    }
}

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ:
// var point = new Point(3.0, 4.0);
// var (x, y) = point;  // x = 3.0, y = 4.0
```

### Р¤РёРЅР°Р»РёР·Р°С‚РѕСЂС‹ (~ClassName)

Р¤РёРЅР°Р»РёР·Р°С‚РѕСЂ (finalizer / destructor) РІС‹Р·С‹РІР°РµС‚СЃСЏ GC РїРµСЂРµРґ РѕСЃРІРѕР±РѕР¶РґРµРЅРёРµРј РѕР±СЉРµРєС‚Р°. **РќРµ РёСЃРїРѕР»СЊР·СѓР№** вЂ” РЅРµРїСЂРµРґСЃРєР°Р·СѓРµРјРѕРµ РІСЂРµРјСЏ РІС‹Р·РѕРІР°, Р·Р°РјРµРґР»СЏРµС‚ GC (РѕР±СЉРµРєС‚ РїРѕРїР°РґР°РµС‚ РІ finalization queue), РЅРµРІРѕР·РјРѕР¶РЅРѕ РіР°СЂР°РЅС‚РёСЂРѕРІР°С‚СЊ РїРѕСЂСЏРґРѕРє.

```csharp
namespace Legacy;

// РђРќРўРРџРђРўРўР•Р Рќ вЂ” РїРѕРєР°Р·Р°РЅРѕ РґР»СЏ СЃРїСЂР°РІРєРё
public class ResourceHolder
{
    private IntPtr _handle;

    ~ResourceHolder()
    {
        // Р’С‹Р·С‹РІР°РµС‚СЃСЏ GC вЂ” РІСЂРµРјСЏ РІС‹Р·РѕРІР° РЅРµ РѕРїСЂРµРґРµР»РµРЅРѕ
        // РСЃРїРѕР»СЊР·СѓР№ IDisposable РІРјРµСЃС‚Рѕ СЌС‚РѕРіРѕ!
        ReleaseHandle(_handle);
    }

    private static void ReleaseHandle(IntPtr handle)
    {
        // РћСЃРІРѕР±РѕР¶РґРµРЅРёРµ unmanaged СЂРµСЃСѓСЂСЃР°
    }
}
```

> **РџСЂР°РІРёР»Рѕ:** РІРјРµСЃС‚Рѕ С„РёРЅР°Р»РёР·Р°С‚РѕСЂРѕРІ РёСЃРїРѕР»СЊР·СѓР№ `IDisposable` + `using`. РЎРј. СЃРµРєС†РёСЋ "IDisposable Рё using pattern" РЅРёР¶Рµ.

### Partial Classes

Р Р°Р·РґРµР»РµРЅРёРµ РѕРґРЅРѕРіРѕ РєР»Р°СЃСЃР° РЅР° РЅРµСЃРєРѕР»СЊРєРѕ С„Р°Р№Р»РѕРІ. Р§Р°СЃС‚Рѕ РёСЃРїРѕР»СЊР·СѓРµС‚СЃСЏ РґР»СЏ code generation (EF Core, source generators).

```csharp
// Р¤Р°Р№Р»: Models/User.cs
namespace Domain.Models;

public partial class User
{
    public Guid Id { get; init; }
    public string Name { get; set; } = string.Empty;
}

// Р¤Р°Р№Р»: Models/User.Validation.cs
namespace Domain.Models;

public partial class User
{
    public bool IsValid() => !string.IsNullOrWhiteSpace(Name);
}
```

C# 13 вЂ” **partial properties**. Partial methods РґРѕСЃС‚СѓРїРЅС‹ СЃ C# 3 (СЂР°СЃС€РёСЂРµРЅРЅС‹Рµ вЂ” СЃ C# 9):

```csharp
namespace Domain.Models;

// Р¤Р°Р№Р» 1: РѕРїСЂРµРґРµР»РµРЅРёРµ
public partial class Customer
{
    public partial string FullName { get; }
    public partial bool Validate();
}

// Р¤Р°Р№Р» 2: СЂРµР°Р»РёР·Р°С†РёСЏ
public partial class Customer
{
    public partial string FullName => $"{FirstName} {LastName}";
    public partial bool Validate() => !string.IsNullOrWhiteSpace(FirstName);
}
```

### Static Classes Рё Static Members

Static РєР»Р°СЃСЃ вЂ” РЅРµ СЃРѕР·РґР°С‘С‚СЃСЏ С‡РµСЂРµР· `new`, РЅРµ СѓС‡Р°СЃС‚РІСѓРµС‚ РІ РЅР°СЃР»РµРґРѕРІР°РЅРёРё.

```csharp
namespace Application.Helpers;

// Static class вЂ” СѓС‚РёР»РёС‚РЅС‹Рµ С„СѓРЅРєС†РёРё, extension methods
public static class StringExtensions
{
    public static string Truncate(this string value, int maxLength)
        => value.Length <= maxLength ? value : value[..maxLength] + "...";

    public static bool IsNullOrEmpty(this string? value)
        => string.IsNullOrEmpty(value);
}
```

Static members РІ РѕР±С‹С‡РЅРѕРј РєР»Р°СЃСЃРµ:

```csharp
namespace Domain.Models;

public class ConnectionPool
{
    // Shared РјРµР¶РґСѓ РІСЃРµРјРё СЌРєР·РµРјРїР»СЏСЂР°РјРё
    private static int _activeConnections;
    public static int MaxConnections { get; set; } = 100;

    public static int ActiveConnections => _activeConnections;

    public void Open()
    {
        Interlocked.Increment(ref _activeConnections);
    }

    public void Close()
    {
        Interlocked.Decrement(ref _activeConnections);
    }
}
```

### this Рё base

```csharp
namespace Domain.Models;

public class Entity
{
    public Guid Id { get; init; }

    public Entity(Guid id) => Id = id;
    public Entity() : this(Guid.NewGuid()) { } // this вЂ” РІС‹Р·РѕРІ РґСЂСѓРіРѕРіРѕ РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂР°
}

public class AuditableEntity : Entity
{
    public DateTime CreatedAt { get; init; }

    // base вЂ” РІС‹Р·РѕРІ РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂР° Р±Р°Р·РѕРІРѕРіРѕ РєР»Р°СЃСЃР°
    public AuditableEntity(Guid id) : base(id)
    {
        CreatedAt = DateTime.UtcNow;
    }
}
```

`this` РґР»СЏ fluent API:

```csharp
namespace Application.Builders;

public class QueryBuilder
{
    private string _table = string.Empty;
    private string _where = string.Empty;

    // Р’РѕР·РІСЂР°С‰Р°РµРј this РґР»СЏ chaining
    public QueryBuilder From(string table) { _table = table; return this; }
    public QueryBuilder Where(string condition) { _where = condition; return this; }
    public string Build() => $"SELECT * FROM {_table} WHERE {_where}";
}

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ:
// var sql = new QueryBuilder().From("Orders").Where("Id = @id").Build();
```

---

## РЎРІРѕР№СЃС‚РІР°

### Auto-Properties

```csharp
namespace Domain.Models;

public class Item
{
    // РџРѕР»РЅР°СЏ Р°РІС‚Рѕ-СЃРІРѕР№СЃС‚РІРѕ: getter + setter
    public string Name { get; set; } = string.Empty;

    // Read-only Р°РІС‚Рѕ-СЃРІРѕР№СЃС‚РІРѕ вЂ” РјРѕР¶РЅРѕ РїСЂРёСЃРІРѕРёС‚СЊ С‚РѕР»СЊРєРѕ РІ РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂРµ
    public Guid Id { get; } = Guid.NewGuid();

    // РђРІС‚Рѕ-СЃРІРѕР№СЃС‚РІРѕ СЃ private setter
    public DateTime CreatedAt { get; private set; } = DateTime.UtcNow;
}
```

### Init-Only Properties (C# 9)

РњРѕР¶РЅРѕ Р·Р°РґР°С‚СЊ РїСЂРё РёРЅРёС†РёР°Р»РёР·Р°С†РёРё (`new { ... }`), РЅРѕ РЅРµ РёР·РјРµРЅРёС‚СЊ РїРѕСЃР»Рµ.

```csharp
namespace Domain.Models;

public class Address
{
    public string Street { get; init; } = string.Empty;
    public string City { get; init; } = string.Empty;
    public string PostalCode { get; init; } = string.Empty;
}

// var addr = new Address { Street = "Main St", City = "Moscow", PostalCode = "101000" };
// addr.City = "SPb"; // РћС€РёР±РєР° РєРѕРјРїРёР»СЏС†РёРё вЂ” init-only!
```

### Required Properties (C# 11)

РљРѕРјРїРёР»СЏС‚РѕСЂ **Р·Р°СЃС‚Р°РІР»СЏРµС‚** СѓРєР°Р·Р°С‚СЊ Р·РЅР°С‡РµРЅРёРµ РїСЂРё СЃРѕР·РґР°РЅРёРё РѕР±СЉРµРєС‚Р°.

```csharp
namespace Application.Contracts;

public class CreateOrderRequest
{
    public required string CustomerName { get; init; }
    public required decimal Amount { get; init; }
    public string? Notes { get; init; }
}

// var request = new CreateOrderRequest(); // РћС€РёР±РєР°: required members РЅРµ Р·Р°РґР°РЅС‹
// var request = new CreateOrderRequest { CustomerName = "John", Amount = 99.99m }; // OK
```

### Computed Properties

```csharp
namespace Domain.Models;

public class Rectangle(double width, double height)
{
    public double Width { get; } = width;
    public double Height { get; } = height;

    // Computed вЂ” РІС‹С‡РёСЃР»СЏРµС‚СЃСЏ РєР°Р¶РґС‹Р№ СЂР°Р· РїСЂРё РѕР±СЂР°С‰РµРЅРёРё
    public double Area => Width * Height;
    public double Perimeter => 2 * (Width + Height);
    public bool IsSquare => Math.Abs(Width - Height) < 0.0001;
}
```

### Expression-Bodied Members

```csharp
namespace Domain.Models;

public class Temperature
{
    private double _celsius;

    // Expression-bodied property
    public double Celsius
    {
        get => _celsius;
        set => _celsius = value;
    }

    // Expression-bodied read-only property
    public double Fahrenheit => _celsius * 9 / 5 + 32;

    // Expression-bodied method
    public static Temperature FromFahrenheit(double f) => new() { Celsius = (f - 32) * 5 / 9 };

    // Expression-bodied constructor
    public Temperature(double celsius) => _celsius = celsius;

    // Expression-bodied finalizer (РґР»СЏ РїСЂРёРјРµСЂР° вЂ” РЅРµ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ!)
    // ~Temperature() => Console.WriteLine("Finalized");

    public override string ToString() => $"{Celsius:F1}В°C ({Fahrenheit:F1}В°F)";
}
```

### `field` keyword (C# 14 preview)

РђРІС‚РѕРјР°С‚РёС‡РµСЃРєРёР№ РґРѕСЃС‚СѓРї Рє backing field СЃРІРѕР№СЃС‚РІР° Р±РµР· СЏРІРЅРѕРіРѕ РѕР±СЉСЏРІР»РµРЅРёСЏ.

```csharp
namespace Domain.Models;

public class Sensor
{
    // field вЂ” СЃСЃС‹Р»РєР° РЅР° auto-generated backing field
    // РќРµС‚ РЅСѓР¶РґС‹ РѕР±СЉСЏРІР»СЏС‚СЊ private double _temperature;
    public double Temperature
    {
        get => field;
        set => field = value < -273.15 ? -273.15 : value; // РІР°Р»РёРґР°С†РёСЏ РїСЂСЏРјРѕ РІ setter
    }

    public string Label
    {
        get => field;
        set => field = value.Trim(); // РЅРѕСЂРјР°Р»РёР·Р°С†РёСЏ РІ setter
    } = "Default";
}
```

---

## РќР°СЃР»РµРґРѕРІР°РЅРёРµ

### Р‘Р°Р·РѕРІС‹Р№ Рё РїСЂРѕРёР·РІРѕРґРЅС‹Р№ РєР»Р°СЃСЃ

```csharp
namespace Domain.Models;

// Р‘Р°Р·РѕРІС‹Р№ РєР»Р°СЃСЃ
public class Shape
{
    public string Color { get; init; } = "Black";

    public virtual double GetArea() => 0;

    public override string ToString() => $"{GetType().Name} [{Color}] Area={GetArea():F2}";
}

// РџСЂРѕРёР·РІРѕРґРЅС‹Р№ РєР»Р°СЃСЃ
public class Circle(double radius) : Shape
{
    public double Radius { get; } = radius;

    public override double GetArea() => Math.PI * Radius * Radius;
}

public class Square(double side) : Shape
{
    public double Side { get; } = side;

    public override double GetArea() => Side * Side;
}
```

### Method Overriding: virtual / override / sealed

```csharp
namespace Domain.Models;

public class Animal
{
    public virtual string Speak() => "...";
}

public class Dog : Animal
{
    // override вЂ” РїРµСЂРµРѕРїСЂРµРґРµР»РµРЅРёРµ РІРёСЂС‚СѓР°Р»СЊРЅРѕРіРѕ РјРµС‚РѕРґР°
    public override string Speak() => "Woof!";
}

public class GuideDog : Dog
{
    // sealed override вЂ” Р·Р°РїСЂРµС‚ РґР°Р»СЊРЅРµР№С€РµРіРѕ РїРµСЂРµРѕРїСЂРµРґРµР»РµРЅРёСЏ РІ РЅР°СЃР»РµРґРЅРёРєР°С…
    public sealed override string Speak() => "Bark (calmly)";
}

// public class SuperGuideDog : GuideDog
// {
//     public override string Speak() => "..."; // РћС€РёР±РєР°: sealed!
// }
```

### Method Hiding: new keyword

```csharp
namespace Domain.Models;

public class BaseLogger
{
    public void Log(string message) => Console.WriteLine($"[Base] {message}");
}

public class CustomLogger : BaseLogger
{
    // new вЂ” СЃРєСЂС‹РІР°РµС‚ РјРµС‚РѕРґ Р±Р°Р·РѕРІРѕРіРѕ РєР»Р°СЃСЃР° (РќР• РїРµСЂРµРѕРїСЂРµРґРµР»СЏРµС‚).
    // РџСЂРё РІС‹Р·РѕРІРµ С‡РµСЂРµР· СЃСЃС‹Р»РєСѓ Р±Р°Р·РѕРІРѕРіРѕ С‚РёРїР° вЂ” РІС‹Р·РѕРІРµС‚СЃСЏ РјРµС‚РѕРґ Р±Р°Р·РѕРІРѕРіРѕ РєР»Р°СЃСЃР°!
    public new void Log(string message) => Console.WriteLine($"[Custom] {message}");
}

// CustomLogger logger = new();
// logger.Log("test");            // [Custom] test
// ((BaseLogger)logger).Log("test"); // [Base] test вЂ” Р’РќРРњРђРќРР•!
```

> **РџСЂР°РІРёР»Рѕ:** `new` hiding вЂ” РїРѕС‡С‚Рё РІСЃРµРіРґР° РїСЂРёР·РЅР°Рє РїР»РѕС…РѕРіРѕ РґРёР·Р°Р№РЅР°. РСЃРїРѕР»СЊР·СѓР№ `virtual` / `override`.

### Abstract Classes Рё Abstract Methods

```csharp
namespace Domain.Models;

// РќРµР»СЊР·СЏ СЃРѕР·РґР°С‚СЊ СЌРєР·РµРјРїР»СЏСЂ abstract РєР»Р°СЃСЃР°
public abstract class Notification
{
    public required string Recipient { get; init; }
    public DateTime SentAt { get; private set; }

    // Abstract РјРµС‚РѕРґ вЂ” РЅРµС‚ СЂРµР°Р»РёР·Р°С†РёРё, РЅР°СЃР»РµРґРЅРёРє РћР‘РЇР—РђРќ СЂРµР°Р»РёР·РѕРІР°С‚СЊ
    public abstract Task SendAsync(CancellationToken ct = default);

    // РћР±С‹С‡РЅС‹Р№ РјРµС‚РѕРґ вЂ” РѕР±С‰Р°СЏ Р»РѕРіРёРєР° РґР»СЏ РІСЃРµС… РЅР°СЃР»РµРґРЅРёРєРѕРІ
    protected void MarkAsSent() => SentAt = DateTime.UtcNow;
}

public class EmailNotification(IEmailSender sender) : Notification
{
    public required string Subject { get; init; }

    public override async Task SendAsync(CancellationToken ct)
    {
        await sender.SendAsync(Recipient, Subject, ct).ConfigureAwait(false);
        MarkAsSent();
    }
}
```

### РџРѕСЂСЏРґРѕРє РІС‹Р·РѕРІР° РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂРѕРІ РІ РёРµСЂР°СЂС…РёРё

РљРѕРЅСЃС‚СЂСѓРєС‚РѕСЂС‹ РІС‹Р·С‹РІР°СЋС‚СЃСЏ **РѕС‚ Р±Р°Р·РѕРІРѕРіРѕ Рє РїСЂРѕРёР·РІРѕРґРЅРѕРјСѓ**. Р”РµСЃС‚СЂСѓРєС‚РѕСЂС‹ вЂ” РЅР°РѕР±РѕСЂРѕС‚.

```csharp
namespace Examples;

public class A
{
    public A() => Console.WriteLine("A ctor");
}

public class B : A
{
    public B() => Console.WriteLine("B ctor");
}

public class C : B
{
    public C() => Console.WriteLine("C ctor");
}

// new C() РІС‹РІРµРґРµС‚:
// A ctor
// B ctor
// C ctor
```

РЎ РїР°СЂР°РјРµС‚СЂР°РјРё:

```csharp
namespace Domain.Models;

public class Entity(Guid id)
{
    public Guid Id { get; } = id;
}

public class AuditableEntity(Guid id, string createdBy) : Entity(id)
{
    public string CreatedBy { get; } = createdBy;
    public DateTime CreatedAt { get; } = DateTime.UtcNow;
}

public class Order(Guid id, string createdBy, string customerName)
    : AuditableEntity(id, createdBy)
{
    public string CustomerName { get; } = customerName;
}
```

---

## РРЅС‚РµСЂС„РµР№СЃС‹

### РћР±СЉСЏРІР»РµРЅРёРµ Рё СЂРµР°Р»РёР·Р°С†РёСЏ

```csharp
namespace Domain.Abstractions;

public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(Guid id, CancellationToken ct = default);
}
```

```csharp
namespace Infrastructure.Persistence;

public class OrderRepository(AppDbContext context) : IRepository<Order>
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct)
        => await context.Orders.FindAsync([id], ct).ConfigureAwait(false);

    public async Task<IReadOnlyList<Order>> GetAllAsync(CancellationToken ct)
        => await context.Orders.ToListAsync(ct).ConfigureAwait(false);

    public async Task AddAsync(Order entity, CancellationToken ct)
        => await context.Orders.AddAsync(entity, ct).ConfigureAwait(false);

    public Task UpdateAsync(Order entity, CancellationToken ct)
    {
        context.Orders.Update(entity);
        return Task.CompletedTask;
    }

    public async Task DeleteAsync(Guid id, CancellationToken ct)
    {
        var entity = await GetByIdAsync(id, ct).ConfigureAwait(false);
        if (entity is not null)
            context.Orders.Remove(entity);
    }
}
```

### Default Interface Methods (C# 8)

РРЅС‚РµСЂС„РµР№СЃС‹ РјРѕРіСѓС‚ СЃРѕРґРµСЂР¶Р°С‚СЊ СЂРµР°Р»РёР·Р°С†РёСЋ РїРѕ СѓРјРѕР»С‡Р°РЅРёСЋ вЂ” РЅРµ Р»РѕРјР°РµС‚ СЃСѓС‰РµСЃС‚РІСѓСЋС‰РёС… РїРѕС‚СЂРµР±РёС‚РµР»РµР№ РїСЂРё РґРѕР±Р°РІР»РµРЅРёРё РЅРѕРІРѕРіРѕ РјРµС‚РѕРґР°.

```csharp
namespace Domain.Abstractions;

public interface ILogger
{
    void Log(string message);

    // Default implementation вЂ” РїРѕС‚СЂРµР±РёС‚РµР»СЊ РјРѕР¶РµС‚ РЅРµ СЂРµР°Р»РёР·РѕРІС‹РІР°С‚СЊ
    void LogError(string message) => Log($"[ERROR] {message}");
    void LogWarning(string message) => Log($"[WARN] {message}");
}

public class ConsoleLogger : ILogger
{
    // Р РµР°Р»РёР·СѓРµРј С‚РѕР»СЊРєРѕ Log вЂ” LogError Рё LogWarning РїРѕР»СѓС‡Р°РµРј Р±РµСЃРїР»Р°С‚РЅРѕ
    public void Log(string message) => Console.WriteLine(message);
}
```

> **РќСЋР°РЅСЃ:** default methods РґРѕСЃС‚СѓРїРЅС‹ С‚РѕР»СЊРєРѕ С‡РµСЂРµР· СЃСЃС‹Р»РєСѓ РЅР° РёРЅС‚РµСЂС„РµР№СЃ:

```csharp
// var logger = new ConsoleLogger();
// logger.LogError("fail"); // РћС€РёР±РєР° РєРѕРјРїРёР»СЏС†РёРё!
// ((ILogger)logger).LogError("fail"); // OK
```

### Static Abstract Members (C# 11)

РџРѕР·РІРѕР»СЏРµС‚ С‚СЂРµР±РѕРІР°С‚СЊ static С‡Р»РµРЅС‹ РІ generic constraints вЂ” РѕСЃРЅРѕРІР° РґР»СЏ generic math.

```csharp
namespace Domain.Abstractions;

public interface IParseable<TSelf> where TSelf : IParseable<TSelf>
{
    static abstract TSelf Parse(string input);
    static abstract bool TryParse(string input, out TSelf? result);
}

public readonly record struct Money(decimal Amount, string Currency) : IParseable<Money>
{
    public static Money Parse(string input)
    {
        // Р¤РѕСЂРјР°С‚: "100.50 USD"
        var parts = input.Split(' ');
        return new Money(decimal.Parse(parts[0]), parts[1]);
    }

    public static bool TryParse(string input, out Money result)
    {
        try { result = Parse(input); return true; }
        catch { result = default; return false; }
    }
}

// Generic РјРµС‚РѕРґ, РёСЃРїРѕР»СЊР·СѓСЋС‰РёР№ static abstract:
// T ParseValue<T>(string input) where T : IParseable<T> => T.Parse(input);
```

### Interface vs Abstract Class вЂ” РєРѕРіРґР° С‡С‚Рѕ

| РљСЂРёС‚РµСЂРёР№                        | Interface                     | Abstract Class                  |
| ------------------------------- | ----------------------------- | ------------------------------- |
| РњРЅРѕР¶РµСЃС‚РІРµРЅРЅРѕРµ РЅР°СЃР»РµРґРѕРІР°РЅРёРµ      | Р”Р°                            | РќРµС‚                             |
| РџРѕР»СЏ (state)                    | РќРµС‚ (С‚РѕР»СЊРєРѕ static)           | Р”Р°                              |
| РљРѕРЅСЃС‚СЂСѓРєС‚РѕСЂС‹                    | РќРµС‚                           | Р”Р°                              |
| Access modifiers РЅР° members     | Р”Р° (C# 8+)                    | Р”Р°                              |
| Default implementation          | Р”Р° (C# 8+)                    | Р”Р°                              |
| РљРѕРіРґР° РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ              | РљРѕРЅС‚СЂР°РєС‚ / capability         | РћР±С‰Р°СЏ Р±Р°Р·Р° + shared state/logic |

**Р­РјРїРёСЂРёС‡РµСЃРєРѕРµ РїСЂР°РІРёР»Рѕ:**
- **Interface** вЂ” "С‡С‚Рѕ РѕР±СЉРµРєС‚ СѓРјРµРµС‚" (`IDisposable`, `IComparable`, `IRepository`)
- **Abstract class** вЂ” "С‡С‚Рѕ РѕР±СЉРµРєС‚ СЏРІР»СЏРµС‚СЃСЏ" (`Notification`, `Shape`)

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: Interface vs Abstract Class вЂ” РєРѕРіРґР° С‡С‚Рѕ?**
> **Interface** вЂ” РєРѕРЅС‚СЂР°РєС‚ РїРѕРІРµРґРµРЅРёСЏ. РќРµСЃРєРѕР»СЊРєРѕ СЂРµР°Р»РёР·Р°С†РёР№, DI, С‚РµСЃС‚РёСЂСѓРµРјРѕСЃС‚СЊ (РјРѕРєРёСЂРѕРІР°РЅРёРµ). C# 8+ вЂ” default implementations.
>
> **Abstract class** вЂ” С‡Р°СЃС‚РёС‡РЅР°СЏ СЂРµР°Р»РёР·Р°С†РёСЏ + РѕР±С‰РµРµ СЃРѕСЃС‚РѕСЏРЅРёРµ. РћРґРёРЅ Р±Р°Р·РѕРІС‹Р№ РєР»Р°СЃСЃ. Template Method pattern.
>
> **РџСЂР°РІРёР»Рѕ:** prefer composition over inheritance. РРЅС‚РµСЂС„РµР№СЃС‹ + DI РІ 90% СЃР»СѓС‡Р°РµРІ Р»СѓС‡С€Рµ РЅР°СЃР»РµРґРѕРІР°РЅРёСЏ.

### РњРЅРѕР¶РµСЃС‚РІРµРЅРЅР°СЏ СЂРµР°Р»РёР·Р°С†РёСЏ

```csharp
namespace Domain.Models;

public interface ISerializable
{
    string Serialize();
}

public interface ICloneable<T>
{
    T Clone();
}

public interface IAuditable
{
    DateTime CreatedAt { get; }
    DateTime? UpdatedAt { get; }
}

// РљР»Р°СЃСЃ СЂРµР°Р»РёР·СѓРµС‚ РЅРµСЃРєРѕР»СЊРєРѕ РёРЅС‚РµСЂС„РµР№СЃРѕРІ
public class Document : ISerializable, ICloneable<Document>, IAuditable
{
    public required string Title { get; init; }
    public required string Content { get; init; }
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; init; }

    public string Serialize()
        => System.Text.Json.JsonSerializer.Serialize(this);

    public Document Clone()
        => new() { Title = Title, Content = Content, CreatedAt = DateTime.UtcNow };
}
```

### Explicit Interface Implementation

РљРѕРіРґР° РґРІР° РёРЅС‚РµСЂС„РµР№СЃР° РёРјРµСЋС‚ РѕРґРЅРѕРёРјС‘РЅРЅС‹Рµ С‡Р»РµРЅС‹, РёР»Рё РЅСѓР¶РЅРѕ СЃРєСЂС‹С‚СЊ РјРµС‚РѕРґ СЃ РѕСЃРЅРѕРІРЅРѕРіРѕ API РєР»Р°СЃСЃР°.

```csharp
namespace Domain.Models;

public interface IFileReader
{
    string Read(string path);
}

public interface INetworkReader
{
    string Read(string url);
}

public class UniversalReader : IFileReader, INetworkReader
{
    // Explicit вЂ” РјРµС‚РѕРґ РґРѕСЃС‚СѓРїРµРЅ С‚РѕР»СЊРєРѕ С‡РµСЂРµР· РёРЅС‚РµСЂС„РµР№СЃРЅСѓСЋ СЃСЃС‹Р»РєСѓ
    string IFileReader.Read(string path) => File.ReadAllText(path);
    // вљ пёЏ new HttpClient() + .Result вЂ” Р°РЅС‚РёРїР°С‚С‚РµСЂРЅ! РўРѕР»СЊРєРѕ РґР»СЏ РґРµРјРѕ СЃРёРЅС‚Р°РєСЃРёСЃР°.
    // Р’ СЂРµР°Р»СЊРЅРѕРј РєРѕРґРµ: IHttpClientFactory + async/await.
    string INetworkReader.Read(string url) => new HttpClient().GetStringAsync(url).Result;

    // РћР±С‰РёР№ public РјРµС‚РѕРґ
    public string Read(string source)
        => source.StartsWith("http")
            ? ((INetworkReader)this).Read(source)
            : ((IFileReader)this).Read(source);
}
```

---

## РџРѕР»РёРјРѕСЂС„РёР·Рј

### Compile-Time (Overloading) vs Runtime (Overriding)

```csharp
namespace Examples;

// Compile-time РїРѕР»РёРјРѕСЂС„РёР·Рј вЂ” method overloading
public class Calculator
{
    public int Add(int a, int b) => a + b;
    public double Add(double a, double b) => a + b;
    public decimal Add(decimal a, decimal b) => a + b;
    // РљРѕРјРїРёР»СЏС‚РѕСЂ РІС‹Р±РёСЂР°РµС‚ РЅСѓР¶РЅС‹Р№ РјРµС‚РѕРґ РЅР° СЌС‚Р°РїРµ РєРѕРјРїРёР»СЏС†РёРё
}

// Runtime РїРѕР»РёРјРѕСЂС„РёР·Рј вЂ” method overriding
public abstract class PaymentProcessor
{
    public abstract Task<bool> ProcessAsync(decimal amount);
}

public class StripeProcessor : PaymentProcessor
{
    public override Task<bool> ProcessAsync(decimal amount)
        => Task.FromResult(true); // Stripe API call
}

public class PayPalProcessor : PaymentProcessor
{
    public override Task<bool> ProcessAsync(decimal amount)
        => Task.FromResult(true); // PayPal API call
}

// PaymentProcessor processor = GetProcessor(); // Runtime вЂ” РєР°РєРѕР№ РєРѕРЅРєСЂРµС‚РЅС‹Р№ С‚РёРї?
// await processor.ProcessAsync(100m);           // Р’С‹Р·РѕРІРµС‚СЃСЏ РЅСѓР¶РЅС‹Р№ override
```

### Covariance Рё Contravariance

```csharp
namespace Examples;

// Covariance (out) вЂ” РјРѕР¶РЅРѕ РІРµСЂРЅСѓС‚СЊ Р±РѕР»РµРµ РєРѕРЅРєСЂРµС‚РЅС‹Р№ С‚РёРї
public interface IProducer<out T>
{
    T Produce();
}

// Contravariance (in) вЂ” РјРѕР¶РЅРѕ РїСЂРёРЅСЏС‚СЊ Р±РѕР»РµРµ Р±Р°Р·РѕРІС‹Р№ С‚РёРї
public interface IConsumer<in T>
{
    void Consume(T item);
}

public class Animal { public string Name { get; init; } = ""; }
public class Dog : Animal { }

public class DogProducer : IProducer<Dog>
{
    public Dog Produce() => new() { Name = "Rex" };
}

public class AnimalConsumer : IConsumer<Animal>
{
    public void Consume(Animal item) => Console.WriteLine(item.Name);
}

// Covariance: IProducer<Dog> -> IProducer<Animal>
// IProducer<Animal> producer = new DogProducer(); // OK

// Contravariance: IConsumer<Animal> -> IConsumer<Dog>
// IConsumer<Dog> consumer = new AnimalConsumer(); // OK
```

### Upcasting Рё Downcasting

```csharp
namespace Examples;

public class Vehicle { public string Brand { get; init; } = ""; }
public class Car : Vehicle { public int Doors { get; init; } }
public class Truck : Vehicle { public double Payload { get; init; } }

public static class CastingExamples
{
    public static void Demo()
    {
        Car car = new() { Brand = "Toyota", Doors = 4 };

        // Upcast вЂ” РІСЃРµРіРґР° Р±РµР·РѕРїР°СЃРµРЅ, РЅРµСЏРІРЅС‹Р№
        Vehicle vehicle = car;

        // Downcast вЂ” С‚СЂРµР±СѓРµС‚ СЏРІРЅС‹Р№ cast, РјРѕР¶РµС‚ Р±СЂРѕСЃРёС‚СЊ InvalidCastException
        Car sameCar = (Car)vehicle; // OK вЂ” РѕР±СЉРµРєС‚ СЂРµР°Р»СЊРЅРѕ Car

        // Р‘РµР·РѕРїР°СЃРЅС‹Р№ downcast СЃ is
        if (vehicle is Car c)
        {
            Console.WriteLine($"Car with {c.Doors} doors");
        }

        // Р‘РµР·РѕРїР°СЃРЅС‹Р№ downcast СЃ as (РІРѕР·РІСЂР°С‰Р°РµС‚ null, РµСЃР»Рё РЅРµ СѓРґР°Р»РѕСЃСЊ)
        Truck? truck = vehicle as Truck; // null вЂ” vehicle РЅРµ Truck

        // Pattern matching (РїСЂРµРґРїРѕС‡С‚РёС‚РµР»СЊРЅС‹Р№ СЃРїРѕСЃРѕР±)
        string description = vehicle switch
        {
            Car { Doors: > 2 } sedan => $"Sedan {sedan.Brand}",
            Car compact => $"Compact {compact.Brand}",
            Truck { Payload: > 10 } heavy => $"Heavy truck {heavy.Brand}",
            Truck t => $"Light truck {t.Brand}",
            _ => $"Vehicle {vehicle.Brand}"
        };
    }
}
```

---

## Records

### Record Class (C# 9)

Immutable reference type СЃ value-based equality. РРґРµР°Р»РµРЅ РґР»СЏ DTO, events, value objects.

```csharp
namespace Domain.ValueObjects;

// Positional record вЂ” РєРѕРјРїРёР»СЏС‚РѕСЂ РіРµРЅРµСЂРёСЂСѓРµС‚:
// - properties (init-only)
// - РєРѕРЅСЃС‚СЂСѓРєС‚РѕСЂ
// - Deconstruct
// - Equals / GetHashCode (value-based)
// - ToString
// - РѕРїРµСЂР°С‚РѕСЂ == / !=
public record Address(string Street, string City, string PostalCode);
```

```csharp
namespace Application.Contracts;

// РќРµ-positional record СЃ СЂСѓС‡РЅС‹РјРё СЃРІРѕР№СЃС‚РІР°РјРё
public record OrderCreatedEvent
{
    public required Guid OrderId { get; init; }
    public required string CustomerName { get; init; }
    public required decimal TotalAmount { get; init; }
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}
```

### Record Struct (C# 10)

Value type СЃ value-based equality. РҐСЂР°РЅРёС‚СЃСЏ РЅР° СЃС‚РµРєРµ, РЅРµС‚ Р°Р»Р»РѕРєР°С†РёРё РІ heap.

```csharp
namespace Domain.ValueObjects;

// Positional record struct вЂ” РјСѓС‚Р°Р±РµР»СЊРЅС‹Р№ РїРѕ СѓРјРѕР»С‡Р°РЅРёСЋ!
public record struct Coordinate(double Latitude, double Longitude);

// readonly record struct вЂ” РёРјРјСѓС‚Р°Р±РµР»СЊРЅС‹Р№ (РїСЂРµРґРїРѕС‡С‚РёС‚РµР»СЊРЅС‹Р№)
public readonly record struct Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch");

        return this with { Amount = Amount + other.Amount };
    }
}
```

### With Expressions

РЎРѕР·РґР°РЅРёРµ РєРѕРїРёРё record СЃ РёР·РјРµРЅС‘РЅРЅС‹РјРё РїРѕР»СЏРјРё (non-destructive mutation).

```csharp
namespace Examples;

public record UserProfile(string Name, string Email, int Age);

public static class WithExpressionDemo
{
    public static void Demo()
    {
        var original = new UserProfile("Alice", "alice@mail.com", 30);

        // with вЂ” СЃРѕР·РґР°С‘С‚ РєРѕРїРёСЋ СЃ РёР·РјРµРЅС‘РЅРЅС‹Рј Email
        var updated = original with { Email = "newalice@mail.com" };

        // original.Email == "alice@mail.com"  вЂ” РЅРµ РёР·РјРµРЅРёР»СЃСЏ
        // updated.Email  == "newalice@mail.com"
        // original == updated // false вЂ” Email СЂР°Р·Р»РёС‡Р°РµС‚СЃСЏ
    }
}
```

### Positional Records Рё Deconstruct

```csharp
namespace Domain.ValueObjects;

public record Range(int Start, int End)
{
    // Computed property
    public int Length => End - Start;

    // Deconstruct РіРµРЅРµСЂРёСЂСѓРµС‚СЃСЏ Р°РІС‚РѕРјР°С‚РёС‡РµСЃРєРё РґР»СЏ positional records
    // public void Deconstruct(out int start, out int end) => (start, end) = (Start, End);
}

// var range = new Range(10, 50);
// var (start, end) = range; // Deconstruct: start = 10, end = 50
```

### Equality: Value-Based vs Reference-Based

```csharp
namespace Examples;

public class PersonClass
{
    public string Name { get; init; } = "";
    public int Age { get; init; }
}

public record PersonRecord(string Name, int Age);

public static class EqualityDemo
{
    public static void Demo()
    {
        // class вЂ” reference equality РїРѕ СѓРјРѕР»С‡Р°РЅРёСЋ
        var c1 = new PersonClass { Name = "Bob", Age = 25 };
        var c2 = new PersonClass { Name = "Bob", Age = 25 };
        var classEqual = c1 == c2;      // false вЂ” СЂР°Р·РЅС‹Рµ РѕР±СЉРµРєС‚С‹ РІ heap

        // record вЂ” value-based equality
        var r1 = new PersonRecord("Bob", 25);
        var r2 = new PersonRecord("Bob", 25);
        var recordEqual = r1 == r2;     // true вЂ” РѕРґРёРЅР°РєРѕРІС‹Рµ Р·РЅР°С‡РµРЅРёСЏ РїРѕР»РµР№

        // ReferenceEquals РІСЃРµРіРґР° РїСЂРѕРІРµСЂСЏРµС‚ СЃСЃС‹Р»РєСѓ
        var sameRef = ReferenceEquals(r1, r2); // false вЂ” СЂР°Р·РЅС‹Рµ РѕР±СЉРµРєС‚С‹
    }
}
```

### Record vs Class vs Struct вЂ” С‚Р°Р±Р»РёС†Р° СЃСЂР°РІРЅРµРЅРёСЏ

| РҐР°СЂР°РєС‚РµСЂРёСЃС‚РёРєР°     | `class`          | `record class`     | `struct`         | `record struct`    |
| ------------------ | ---------------- | ------------------ | ---------------- | ------------------ |
| РўРёРї                | Reference        | Reference          | Value            | Value              |
| Heap/Stack         | Heap             | Heap               | Inline*          | Inline*            |
| Equality           | Reference        | Value-based        | Value (Equals**) | Value-based        |
| Immutable          | РќРµС‚ (РІСЂСѓС‡РЅСѓСЋ)    | Р”Р° (РїРѕ СѓРјРѕР»С‡Р°РЅРёСЋ)  | РќРµС‚ (РІСЂСѓС‡РЅСѓСЋ)    | readonly = РґР°      |
| РќР°СЃР»РµРґРѕРІР°РЅРёРµ       | Р”Р°               | Р”Р° (С‚РѕР»СЊРєРѕ record) | РќРµС‚              | РќРµС‚                |
| `with` expression  | РќРµС‚              | Р”Р°                 | Р”Р° (C# 10+)      | Р”Р°                 |
| Deconstruct        | Р’СЂСѓС‡РЅСѓСЋ          | РђРІС‚Рѕ (positional)  | Р’СЂСѓС‡РЅСѓСЋ          | РђРІС‚Рѕ (positional)  |
| РљРѕРіРґР° РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ | Mutable entities | DTO, events, VO    | РњР°Р»РµРЅСЊРєРёРµ РґР°РЅРЅС‹Рµ | VO Р±РµР· Р°Р»Р»РѕРєР°С†РёР№   |

> \* **Inline** вЂ” РЅР° СЃС‚РµРєРµ РґР»СЏ Р»РѕРєР°Р»СЊРЅС‹С… РїРµСЂРµРјРµРЅРЅС‹С…, РІРЅСѓС‚СЂРё РѕР±СЉРµРєС‚Р°-РІР»Р°РґРµР»СЊС†Р° РІ heap РґР»СЏ РїРѕР»РµР№ РєР»Р°СЃСЃР°. РћС‚РґРµР»СЊРЅРѕР№ heap-Р°Р»Р»РѕРєР°С†РёРё РЅРµС‚ (РµСЃР»Рё РЅРµС‚ boxing).
> \*\* Struct РїРѕ СѓРјРѕР»С‡Р°РЅРёСЋ РїРѕР»СѓС‡Р°РµС‚ value equality С‡РµСЂРµР· `ValueType.Equals` (reflection, РјРµРґР»РµРЅРЅРѕ). РћРїРµСЂР°С‚РѕСЂ `==` **РЅРµ РіРµРЅРµСЂРёСЂСѓРµС‚СЃСЏ** Р°РІС‚РѕРјР°С‚РёС‡РµСЃРєРё вЂ” РЅСѓР¶РЅР° СЂСѓС‡РЅР°СЏ РїРµСЂРµРіСЂСѓР·РєР°. Р’ `record struct` РѕРїРµСЂР°С‚РѕСЂ `==` РіРµРЅРµСЂРёСЂСѓРµС‚СЃСЏ.

---

## Р’Р»РѕР¶РµРЅРЅС‹Рµ С‚РёРїС‹

### Nested Classes

```csharp
namespace Domain.Models;

public class OrderAggregate
{
    private readonly List<OrderLine> _lines = [];

    public IReadOnlyList<OrderLine> Lines => _lines;

    public void AddLine(string product, int quantity, decimal price)
        => _lines.Add(new OrderLine(product, quantity, price));

    public decimal Total => _lines.Sum(l => l.Subtotal);

    // Nested class вЂ” С‚РµСЃРЅРѕ СЃРІСЏР·Р°РЅ СЃ СЂРѕРґРёС‚РµР»РµРј, РЅРµ РёСЃРїРѕР»СЊР·СѓРµС‚СЃСЏ РѕС‚РґРµР»СЊРЅРѕ
    public class OrderLine(string product, int quantity, decimal unitPrice)
    {
        public string Product { get; } = product;
        public int Quantity { get; } = quantity;
        public decimal UnitPrice { get; } = unitPrice;
        public decimal Subtotal => Quantity * UnitPrice;
    }
}
```

### РљРѕРіРґР° РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ

- РўРёРї РёРјРµРµС‚ СЃРјС‹СЃР» **С‚РѕР»СЊРєРѕ** РІ РєРѕРЅС‚РµРєСЃС‚Рµ СЂРѕРґРёС‚РµР»СЊСЃРєРѕРіРѕ РєР»Р°СЃСЃР°
- Builder pattern вЂ” `Order.Builder`
- Р РµР°Р»РёР·Р°С†РёСЏ РёРЅС‚РµСЂС„РµР№СЃР°, РЅРµ РїСЂРµРґРЅР°Р·РЅР°С‡РµРЅРЅР°СЏ РґР»СЏ РІРЅРµС€РЅРµРіРѕ РїРѕС‚СЂРµР±Р»РµРЅРёСЏ
- State machine states вЂ” `Connection.ConnectedState`

```csharp
namespace Domain.Models;

// РџСЂРёРјРµСЂ: state machine СЃ nested types
public abstract class ConnectionState
{
    public abstract string Status { get; }
    public virtual bool CanConnect => false;
    public virtual bool CanDisconnect => false;

    public sealed class Disconnected : ConnectionState
    {
        public override string Status => "Disconnected";
        public override bool CanConnect => true;
    }

    public sealed class Connected : ConnectionState
    {
        public override string Status => "Connected";
        public override bool CanDisconnect => true;
    }

    public sealed class Error(string message) : ConnectionState
    {
        public override string Status => $"Error: {message}";
        public override bool CanConnect => true;
    }
}
```

---

## РљР»СЋС‡РµРІС‹Рµ РїР°С‚С‚РµСЂРЅС‹

### Composition over Inheritance

РќР°СЃР»РµРґРѕРІР°РЅРёРµ СЃРѕР·РґР°С‘С‚ Р¶С‘СЃС‚РєСѓСЋ СЃРІСЏР·СЊ. РљРѕРјРїРѕР·РёС†РёСЏ вЂ” РіРёР±РєРѕСЃС‚СЊ С‡РµСЂРµР· РёРЅС‚РµСЂС„РµР№СЃС‹.

```csharp
namespace Domain.Models;

// РџР›РћРҐРћ: РЅР°СЃР»РµРґРѕРІР°РЅРёРµ вЂ” Р¶С‘СЃС‚РєР°СЏ РёРµСЂР°СЂС…РёСЏ
// public class FlyingSwimmingDuck : FlyingAnimal, SwimmingAnimal { } // РќРµРІРѕР·РјРѕР¶РЅРѕ!

// РҐРћР РћРЁРћ: РєРѕРјРїРѕР·РёС†РёСЏ С‡РµСЂРµР· РёРЅС‚РµСЂС„РµР№СЃС‹
public interface IFlyable
{
    void Fly();
}

public interface ISwimmable
{
    void Swim();
}

// Р РµР°Р»РёР·Р°С†РёРё РїРѕРІРµРґРµРЅРёСЏ
public class WingFlight : IFlyable
{
    public void Fly() => Console.WriteLine("Flying with wings");
}

public class Floating : ISwimmable
{
    public void Swim() => Console.WriteLine("Floating on water");
}

// РљРѕРјРїРѕР·РёС†РёСЏ: duck РРњР•Р•Рў СЃРїРѕСЃРѕР±РЅРѕСЃС‚Рё, Р° РЅРµ РЇР’Р›РЇР•РўРЎРЇ С‡РµРј-С‚Рѕ
public class Duck(IFlyable flyBehavior, ISwimmable swimBehavior)
{
    public void Fly() => flyBehavior.Fly();
    public void Swim() => swimBehavior.Swim();
}

// var duck = new Duck(new WingFlight(), new Floating());
// duck.Fly();  // "Flying with wings"
// duck.Swim(); // "Floating on water"
```

### Sealed Classes вЂ” РїСЂРѕРёР·РІРѕРґРёС‚РµР»СЊРЅРѕСЃС‚СЊ Рё РґРёР·Р°Р№РЅ

`sealed` Р·Р°РїСЂРµС‰Р°РµС‚ РЅР°СЃР»РµРґРѕРІР°РЅРёРµ. JIT РїСЂРёРјРµРЅСЏРµС‚ devirtualization вЂ” РїСЂСЏРјРѕР№ РІС‹Р·РѕРІ РІРјРµСЃС‚Рѕ РІРёСЂС‚СѓР°Р»СЊРЅРѕР№ С‚Р°Р±Р»РёС†С‹.

```csharp
namespace Domain.Models;

// sealed вЂ” РЅРµР»СЊР·СЏ РЅР°СЃР»РµРґРѕРІР°С‚СЊ, JIT РѕРїС‚РёРјРёР·РёСЂСѓРµС‚ virtual/override РІС‹Р·РѕРІС‹
public sealed class InMemoryCache
{
    private readonly Dictionary<string, object> _store = [];

    public void Set(string key, object value) => _store[key] = value;

    public T? Get<T>(string key)
        => _store.TryGetValue(key, out var value) ? (T)value : default;

    public bool Remove(string key) => _store.Remove(key);
}
```

> **Р РµРєРѕРјРµРЅРґР°С†РёСЏ:** РїРѕРјРµС‡Р°Р№ `sealed` РІСЃРµ РєР»Р°СЃСЃС‹, РєРѕС‚РѕСЂС‹Рµ **РЅРµ РїСЂРµРґРЅР°Р·РЅР°С‡РµРЅС‹** РґР»СЏ РЅР°СЃР»РµРґРѕРІР°РЅРёСЏ. Р­С‚Рѕ Рё РїСЂРѕРёР·РІРѕРґРёС‚РµР»СЊРЅРѕСЃС‚СЊ, Рё СЏРІРЅС‹Р№ РґРёР·Р°Р№РЅ-РёРЅС‚РµРЅС‚. Р’ .NET runtime Р±РѕР»СЊС€РёРЅСЃС‚РІРѕ internal РєР»Р°СЃСЃРѕРІ вЂ” sealed.

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: sealed вЂ” Р·Р°С‡РµРј Р·Р°РїРµС‡Р°С‚С‹РІР°С‚СЊ РєР»Р°СЃСЃ?**
> Р—Р°РїСЂРµС‰Р°РµС‚ РЅР°СЃР»РµРґРѕРІР°РЅРёРµ. РџСЂРёС‡РёРЅС‹: (1) Р±РµР·РѕРїР°СЃРЅРѕСЃС‚СЊ вЂ” РіР°СЂР°РЅС‚РёСЏ РєРѕРЅС‚СЂР°РєС‚Р°, (2) РїСЂРѕРёР·РІРѕРґРёС‚РµР»СЊРЅРѕСЃС‚СЊ вЂ” JIT РґРµРІРёСЂС‚СѓР°Р»РёР·РёСЂСѓРµС‚ РІС‹Р·РѕРІС‹ sealed-РјРµС‚РѕРґРѕРІ, (3) РґРёР·Р°Р№РЅ вЂ” СЏРІРЅРѕРµ СѓРєР°Р·Р°РЅРёРµ С‡С‚Рѕ РєР»Р°СЃСЃ РЅРµ РґР»СЏ СЂР°СЃС€РёСЂРµРЅРёСЏ.
>
> **РџСЂР°РєС‚РёРєР°:** РІ .NET Runtime РјРЅРѕРіРёРµ internal РєР»Р°СЃСЃС‹ sealed РґР»СЏ РїСЂРѕРёР·РІРѕРґРёС‚РµР»СЊРЅРѕСЃС‚Рё. Р’ ASP.NET Core handler-С‹ С‡Р°СЃС‚Рѕ sealed.

### Object Initializers

```csharp
namespace Application.Contracts;

public class OrderDto
{
    public required Guid Id { get; init; }
    public required string CustomerName { get; init; }
    public List<string> Items { get; init; } = [];
    public decimal Total { get; init; }
}

public static class InitializerDemo
{
    public static OrderDto Create()
    {
        // Object initializer вЂ” Р·Р°РґР°С‘Рј СЃРІРѕР№СЃС‚РІР° РїСЂРё СЃРѕР·РґР°РЅРёРё
        return new OrderDto
        {
            Id = Guid.NewGuid(),
            CustomerName = "Alice",
            Items = ["Widget", "Gadget", "Doohickey"],
            Total = 149.99m
        };
    }

    // Collection initializer (C# 12 вЂ” collection expressions)
    public static List<int> Numbers() => [1, 2, 3, 4, 5];
    public static int[] Array() => [10, 20, 30];
}
```

### IDisposable Рё using pattern

Р”Р»СЏ РґРµС‚РµСЂРјРёРЅРёСЂРѕРІР°РЅРЅРѕРіРѕ РѕСЃРІРѕР±РѕР¶РґРµРЅРёСЏ СЂРµСЃСѓСЂСЃРѕРІ (connections, files, handles).

```csharp
namespace Infrastructure.Services;

// Р‘Р°Р·РѕРІС‹Р№ РїР°С‚С‚РµСЂРЅ IDisposable
public sealed class DatabaseConnection : IDisposable
{
    private bool _disposed;
    private readonly NpgsqlConnection _connection;

    public DatabaseConnection(string connectionString)
    {
        _connection = new NpgsqlConnection(connectionString);
        _connection.Open();
    }

    public NpgsqlCommand CreateCommand(string sql)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        return new NpgsqlCommand(sql, _connection);
    }

    public void Dispose()
    {
        if (_disposed) return;
        _connection.Dispose();
        _disposed = true;
    }
}
```

```csharp
namespace Infrastructure.Services;

// IAsyncDisposable вЂ” РґР»СЏ Р°СЃРёРЅС…СЂРѕРЅРЅРѕРіРѕ РѕСЃРІРѕР±РѕР¶РґРµРЅРёСЏ
public sealed class FileProcessor : IAsyncDisposable
{
    private readonly StreamWriter _writer;

    public FileProcessor(string path)
    {
        _writer = new StreamWriter(path, append: true);
    }

    public async Task WriteLineAsync(string line)
        => await _writer.WriteLineAsync(line).ConfigureAwait(false);

    public async ValueTask DisposeAsync()
    {
        await _writer.DisposeAsync().ConfigureAwait(false);
    }
}
```

РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ `using`:

```csharp
namespace Examples;

public static class UsingDemo
{
    // РљР»Р°СЃСЃРёС‡РµСЃРєРёР№ using block
    public static void Classic()
    {
        using (var connection = new DatabaseConnection("connstr"))
        {
            var cmd = connection.CreateCommand("SELECT 1");
            // ...
        } // Dispose() РІС‹Р·С‹РІР°РµС‚СЃСЏ Р·РґРµСЃСЊ

    }

    // using declaration (C# 8) вЂ” Dispose РїСЂРё РІС‹С…РѕРґРµ РёР· enclosing scope (Р±Р»РѕРєР° {})
    public static void Declaration()
    {
        using var connection = new DatabaseConnection("connstr");
        var cmd = connection.CreateCommand("SELECT 1");
        // Dispose() РІС‹Р·С‹РІР°РµС‚СЃСЏ РїСЂРё РІС‹С…РѕРґРµ РёР· С‚РµРєСѓС‰РµРіРѕ scope (Р·РґРµСЃСЊ вЂ” РјРµС‚РѕРґ)
    }

    // await using вЂ” РґР»СЏ IAsyncDisposable
    public static async Task AsyncUsing()
    {
        await using var processor = new FileProcessor("log.txt");
        await processor.WriteLineAsync("Hello");
        // DisposeAsync() РІС‹Р·С‹РІР°РµС‚СЃСЏ РїСЂРё РІС‹С…РѕРґРµ РёР· РјРµС‚РѕРґР°
    }
}
```

---

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: Dispose pattern вЂ” Р·Р°С‡РµРј GC.SuppressFinalize?**
> Р‘РµР· `SuppressFinalize` РѕР±СЉРµРєС‚ РїСЂРѕС…РѕРґРёС‚ С‡РµСЂРµР· Finalization Queue РґР°Р¶Рµ РїРѕСЃР»Рµ Dispose: Р»РёС€РЅСЏСЏ СЃР±РѕСЂРєР° GC, РїСЂРѕРјРѕС†РёСЏ РІ СЃР»РµРґСѓСЋС‰РµРµ РїРѕРєРѕР»РµРЅРёРµ. `SuppressFinalize` СѓР±РёСЂР°РµС‚ РѕР±СЉРµРєС‚ РёР· РѕС‡РµСЂРµРґРё С„РёРЅР°Р»РёР·Р°С†РёРё.
>
> **РџР°С‚С‚РµСЂРЅ:** `IDisposable` + `IAsyncDisposable`. Р’СЃРµРіРґР° `using` / `await using`. Р¤РёРЅР°Р»РёР·Р°С‚РѕСЂ вЂ” С‚РѕР»СЊРєРѕ safety net РґР»СЏ unmanaged СЂРµСЃСѓСЂСЃРѕРІ.

## РЎРј. С‚Р°РєР¶Рµ

- [РўРёРїС‹ Рё РїР°РјСЏС‚СЊ](types-and-memory.md)
- [Collections Рё LINQ](collections-linq.md)
- [Delegates Рё Events](delegates-events.md)
- [Modern C#](modern-features.md)
