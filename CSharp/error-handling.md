---
tags: [exceptions, error-handling, strings, json, regex]
level: Middle to Senior
---

# РћР±СЂР°Р±РѕС‚РєР° РѕС€РёР±РѕРє, СЃС‚СЂРѕРєРё Рё I/O

## Р§С‚Рѕ СЌС‚Рѕ, Р·Р°С‡РµРј Рё РєРѕРіРґР°

### Р§С‚Рѕ С‚Р°РєРѕРµ Exception?
**РЎРёРіРЅР°Р» В«С‡С‚Рѕ-С‚Рѕ РїРѕС€Р»Рѕ РЅРµ С‚Р°РєВ».** РџСЂРµСЂС‹РІР°РµС‚ РЅРѕСЂРјР°Р»СЊРЅС‹Р№ РїРѕС‚РѕРє, РїРµСЂРµРґР°С‘С‚ СѓРїСЂР°РІР»РµРЅРёРµ РІ catch-Р±Р»РѕРє. РЎРѕРґРµСЂР¶РёС‚ СЃРѕРѕР±С‰РµРЅРёРµ, СЃС‚РµРєС‚СЂРµР№СЃ, С‚РёРї РѕС€РёР±РєРё.

**РђРЅР°Р»РѕРіРёСЏ:** РџРѕР¶Р°СЂРЅР°СЏ СЃРёРіРЅР°Р»РёР·Р°С†РёСЏ. Р Р°Р±РѕС‚Р° РѕСЃС‚Р°РЅР°РІР»РёРІР°РµС‚СЃСЏ, РІСЃРµ СЂРµР°РіРёСЂСѓСЋС‚ РЅР° С‚СЂРµРІРѕРіСѓ. Р”РѕСЂРѕРіРѕ (СЃС‚РµРєС‚СЂРµР№СЃ = СЂР°СЃСЃР»РµРґРѕРІР°РЅРёРµ), РїРѕСЌС‚РѕРјСѓ РЅРµ РґС‘СЂРіР°Р№ Р±РµР· РїРѕРІРѕРґР°.

### РљРѕРіРґР° С‡С‚Рѕ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ?

| РЎРёС‚СѓР°С†РёСЏ | РџРѕРґС…РѕРґ | РџРѕС‡РµРјСѓ |
|----------|--------|--------|
| Р‘Р” СѓРїР°Р»Р°, СЃРµС‚СЊ РЅРµРґРѕСЃС‚СѓРїРЅР° | **Exception** | РРЅС„СЂР°СЃС‚СЂСѓРєС‚СѓСЂРЅС‹Р№ СЃР±РѕР№ вЂ” РёСЃРєР»СЋС‡РёС‚РµР»СЊРЅР°СЏ СЃРёС‚СѓР°С†РёСЏ |
| Email СѓР¶Рµ Р·Р°РЅСЏС‚, РЅРµРІР°Р»РёРґРЅС‹Р№ РІРІРѕРґ | **Result Pattern** | РћР¶РёРґР°РµРјР°СЏ РѕС€РёР±РєР° вЂ” РЅРµ РёСЃРєР»СЋС‡РµРЅРёРµ, Р° Р±РёР·РЅРµСЃ-Р»РѕРіРёРєР° |
| РњРµС‚РѕРґ РјРѕР¶РµС‚ РІРµСЂРЅСѓС‚СЊ null | **Nullable + ?. ??** | `Order? order = Find(id); order?.Cancel();` |
| РџР°СЂСЃРёРЅРі СЃС‚СЂРѕРєРё РІ С‡РёСЃР»Рѕ | **TryParse** | `int.TryParse(s, out var n)` вЂ” Р±РµР· exception |
| Р¤Р°Р№Р» РЅРµ РЅР°Р№РґРµРЅ, JSON Р±РёС‚С‹Р№ | **try/catch РєРѕРЅРєСЂРµС‚РЅРѕРіРѕ С‚РёРїР°** | `catch (FileNotFoundException)` вЂ” РЅРµ РіРѕР»С‹Р№ `catch` |

### РЎС‚СЂРѕРєРё вЂ” РєРѕРіРґР° С‡С‚Рѕ?

| Р—Р°РґР°С‡Р° | РРЅСЃС‚СЂСѓРјРµРЅС‚ | РџРѕС‡РµРјСѓ |
|--------|-----------|--------|
| РЎРєР»РµРёС‚СЊ 2-3 СЃС‚СЂРѕРєРё | **РРЅС‚РµСЂРїРѕР»СЏС†РёСЏ** `$"Hello {name}"` | Р§РёС‚Р°РµРјРѕ, Р±С‹СЃС‚СЂРѕ |
| РЎРєР»РµРёС‚СЊ РІ С†РёРєР»Рµ (100+ СЂР°Р·) | **StringBuilder** | String immutable в†’ РєР°Р¶РґР°СЏ РєРѕРЅРєР°С‚РµРЅР°С†РёСЏ = РЅРѕРІС‹Р№ РѕР±СЉРµРєС‚ |
| РњРЅРѕРіРѕСЃС‚СЂРѕС‡РЅС‹Р№ С‚РµРєСЃС‚ / JSON | **Raw string literals** `"""..."""` | Р‘РµР· СЌРєСЂР°РЅРёСЂРѕРІР°РЅРёСЏ |
| РџР°СЂСЃРёРЅРі Р±РµР· Р°Р»Р»РѕРєР°С†РёР№ | **Span\<char\>** | Hot path, zero-alloc |

---

> РЎРїСЂР°РІРѕС‡РЅРёРє РїРѕ exception handling, СЃС‚СЂРѕРєР°Рј, I/O, JSON Рё Regex. C# 13 / .NET 9.
> РўРµРѕСЂРёСЏ в†’ РїСЂР°РєС‚РёРєР° в†’ senior-level РєРѕРґ в†’ РІРѕРїСЂРѕСЃС‹ РёРЅС‚РµСЂРІСЊСЋ.

---

## Exception Handling

### try / catch / finally вЂ” РїРѕС‚РѕРє РІС‹РїРѕР»РЅРµРЅРёСЏ

Р‘Р°Р·РѕРІС‹Р№ РјРµС…Р°РЅРёР·Рј РѕР±СЂР°Р±РѕС‚РєРё РѕС€РёР±РѕРє. `finally` РІС‹РїРѕР»РЅСЏРµС‚СЃСЏ **РІСЃРµРіРґР°** вЂ” РґР°Р¶Рµ РїСЂРё `return` РёР· `try`.

```csharp
try
{
    int result = int.Parse("not_a_number");
    Console.WriteLine(result);
}
catch (FormatException ex)
{
    // Р›РѕРІРёРј РєРѕРЅРєСЂРµС‚РЅС‹Р№ С‚РёРї вЂ” РџР РђР’РР›Р¬РќРћ
    Console.WriteLine($"РћС€РёР±РєР° С„РѕСЂРјР°С‚Р°: {ex.Message}");
}
catch (Exception ex)
{
    // РћР±С‰РёР№ catch вЂ” С‚РѕР»СЊРєРѕ РІ РєРѕРЅС†Рµ С†РµРїРѕС‡РєРё
    Console.WriteLine($"РќРµРёР·РІРµСЃС‚РЅР°СЏ РѕС€РёР±РєР°: {ex.Message}");
}
finally
{
    // Р’С‹РїРѕР»РЅСЏРµС‚СЃСЏ Р’РЎР•Р“Р”Рђ: Рё РїСЂРё СѓСЃРїРµС…Рµ, Рё РїСЂРё РѕС€РёР±РєРµ, Рё РїСЂРё return
    Console.WriteLine("РћС‡РёСЃС‚РєР° СЂРµСЃСѓСЂСЃРѕРІ");
}
```

РџРѕСЂСЏРґРѕРє РІС‹РїРѕР»РЅРµРЅРёСЏ РїСЂРё РѕС€РёР±РєРµ:
```
1. try в†’ РІС‹РїРѕР»РЅСЏРµС‚СЃСЏ РґРѕ СЃС‚СЂРѕРєРё СЃ РѕС€РёР±РєРѕР№
2. catch в†’ РїРµСЂРІС‹Р№ РїРѕРґС…РѕРґСЏС‰РёР№ РїРѕ С‚РёРїСѓ (СЃРІРµСЂС…Сѓ РІРЅРёР·)
3. finally в†’ РІСЃРµРіРґР°
```

### Exception Hierarchy

```
                        Exception
                           |
              +------------+------------+
              |                         |
      SystemException           ApplicationException (СѓСЃС‚Р°СЂРµР»)
              |
  +-----------+-----------+-----------+
  |           |           |           |
ArgumentException  InvalidOp...  IOException
  |                                   |
ArgumentNullException          FileNotFoundException
ArgumentOutOfRangeException    DirectoryNotFoundException
```

### Common Exceptions вЂ” РєРѕРіРґР° РєР°РєРѕР№ Р±СЂРѕСЃР°С‚СЊ

```csharp
// ArgumentNullException вЂ” Р°СЂРіСѓРјРµРЅС‚ null, Р° РЅРµ РґРѕР»Р¶РµРЅ Р±С‹С‚СЊ
public decimal CalculateDiscount(Order order)
{
    ArgumentNullException.ThrowIfNull(order);           // .NET 6+
    ArgumentException.ThrowIfNullOrEmpty(order.Id);     // .NET 7+ (ArgumentException, РЅРµ ArgumentNullException!)
    return order.Total * 0.1m;
}

// ArgumentException вЂ” Р°СЂРіСѓРјРµРЅС‚ РЅРµРІР°Р»РёРґРЅС‹Р№ (РЅРµ null, РЅРѕ РЅРµРїСЂР°РІРёР»СЊРЅС‹Р№)
public void SetAge(int age)
{
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(age); // .NET 8+
    ArgumentOutOfRangeException.ThrowIfGreaterThan(age, 150);
    _age = age;
}

// InvalidOperationException вЂ” РѕР±СЉРµРєС‚ РІ РЅРµРїСЂР°РІРёР»СЊРЅРѕРј СЃРѕСЃС‚РѕСЏРЅРёРё
public void Complete()
{
    if (_status == OrderStatus.Cancelled)
        throw new InvalidOperationException(
            $"РќРµР»СЊР·СЏ Р·Р°РІРµСЂС€РёС‚СЊ РѕС‚РјРµРЅС‘РЅРЅС‹Р№ Р·Р°РєР°Р·. РўРµРєСѓС‰РёР№ СЃС‚Р°С‚СѓСЃ: {_status}");
    _status = OrderStatus.Completed;
}

// NotSupportedException вЂ” РѕРїРµСЂР°С†РёСЏ РЅРµ РїРѕРґРґРµСЂР¶РёРІР°РµС‚СЃСЏ С‚РёРїРѕРј
public override void Write(byte[] buffer, int offset, int count)
    => throw new NotSupportedException("РџРѕС‚РѕРє С‚РѕР»СЊРєРѕ РґР»СЏ С‡С‚РµРЅРёСЏ");

// KeyNotFoundException вЂ” РєР»СЋС‡ РЅРµ РЅР°Р№РґРµРЅ РІ РєРѕР»Р»РµРєС†РёРё
public User GetUser(string id)
    => _users.TryGetValue(id, out var user)
        ? user
        : throw new KeyNotFoundException($"РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ '{id}' РЅРµ РЅР°Р№РґРµРЅ");

// NotImplementedException вЂ” Р·Р°РіР»СѓС€РєР° (TODO РІ РєРѕРґРµ)
public decimal CalculateTax()
    => throw new NotImplementedException("TODO: СЂРµР°Р»РёР·РѕРІР°С‚СЊ СЂР°СЃС‡С‘С‚ РЅР°Р»РѕРіР°");
```

### throw vs throw ex вЂ” СЃРѕС…СЂР°РЅРµРЅРёРµ Stack Trace

**РљР РРўРР§Р•РЎРљР Р’РђР–РќРћ:** `throw ex` С‚РµСЂСЏРµС‚ РѕСЂРёРіРёРЅР°Р»СЊРЅС‹Р№ stack trace.

```csharp
try
{
    SomeRiskyOperation();
}
catch (Exception ex)
{
    LogError(ex);
    throw;      // РџР РђР’РР›Р¬РќРћ: СЃРѕС…СЂР°РЅСЏРµС‚ РѕСЂРёРіРёРЅР°Р»СЊРЅС‹Р№ stack trace
    // throw ex; // РќР•РџР РђР’РР›Р¬РќРћ: stack trace РЅР°С‡РЅС‘С‚СЃСЏ СЃ СЌС‚РѕР№ СЃС‚СЂРѕРєРё
}
```

**ExceptionDispatchInfo** вЂ” РїРѕРІС‚РѕСЂРЅС‹Р№ throw СЃ РїРѕР»РЅС‹Рј СЃРѕС…СЂР°РЅРµРЅРёРµРј СЃС‚РµРєР°:

```csharp
using System.Runtime.ExceptionServices;

ExceptionDispatchInfo? capturedException = null;

try
{
    await ProcessPaymentAsync();
}
catch (Exception ex)
{
    capturedException = ExceptionDispatchInfo.Capture(ex);
}

// ... РїРѕР·Р¶Рµ, РјРѕР¶РµС‚ Р±С‹С‚СЊ РґР°Р¶Рµ РІ РґСЂСѓРіРѕРј РјРµС‚РѕРґРµ
if (capturedException is not null)
{
    capturedException.Throw(); // РџРѕР»РЅС‹Р№ РѕСЂРёРіРёРЅР°Р»СЊРЅС‹Р№ stack trace СЃРѕС…СЂР°РЅС‘РЅ
}
```

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: throw vs throw ex вЂ” С‡С‚Рѕ С‚РµСЂСЏРµС‚СЃСЏ?**
> `throw;` вЂ” РїРµСЂРµР±СЂР°СЃС‹РІР°РµС‚ РёСЃРєР»СЋС‡РµРЅРёРµ СЃ РѕСЂРёРіРёРЅР°Р»СЊРЅС‹Рј stack trace. `throw ex;` вЂ” СЃР±СЂР°СЃС‹РІР°РµС‚ stack trace РЅР° С‚РµРєСѓС‰СѓСЋ РїРѕР·РёС†РёСЋ, С‚РµСЂСЏСЏ РёРЅС„РѕСЂРјР°С†РёСЋ Рѕ РјРµСЃС‚Рµ РІРѕР·РЅРёРєРЅРѕРІРµРЅРёСЏ.
>
> **РџСЂР°РІРёР»Рѕ:** РІСЃРµРіРґР° `throw;` РІ catch. `throw new WrapperException("msg", ex);` РґР»СЏ РѕР±С‘СЂС‚РєРё СЃ СЃРѕС…СЂР°РЅРµРЅРёРµРј inner exception. `ExceptionDispatchInfo.Capture(ex).Throw()` вЂ” РґР»СЏ РѕС‚Р»РѕР¶РµРЅРЅРѕРіРѕ РїРµСЂРµР±СЂР°СЃС‹РІР°РЅРёСЏ.

### Exception Filters вЂ” `when`

Р¤РёР»СЊС‚СЂС‹ РїСЂРѕРІРµСЂСЏСЋС‚СЃСЏ **РґРѕ** СЂР°СЃРєСЂСѓС‚РєРё СЃС‚РµРєР°. Р•СЃР»Рё `when` РІРµСЂРЅСѓР» `false`, catch РїСЂРѕРїСѓСЃРєР°РµС‚СЃСЏ.

```csharp
try
{
    await httpClient.GetAsync(url, cancellationToken);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    return Result.Failure<Order>("Р—Р°РєР°Р· РЅРµ РЅР°Р№РґРµРЅ");
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.TooManyRequests)
{
    await Task.Delay(TimeSpan.FromSeconds(5));
    return await RetryAsync();
}
catch (Exception ex) when (ex is not OperationCanceledException)
{
    // Р›РѕРІРёРј РІСЃС‘, РљР РћРњР• РѕС‚РјРµРЅС‹ вЂ” РїСѓСЃС‚СЊ OperationCanceledException РїСЂРѕР»РµС‚РёС‚ РІС‹С€Рµ
    _logger.LogError(ex, "РћС€РёР±РєР° HTTP-Р·Р°РїСЂРѕСЃР°");
    return Result.Failure<Order>(ex.Message);
}

// Р¤РёР»СЊС‚СЂ РґР»СЏ Р»РѕРіРёСЂРѕРІР°РЅРёСЏ Р±РµР· РїРµСЂРµС…РІР°С‚Р° (when РІСЃРµРіРґР° false)
catch (Exception ex) when (LogException(ex))
{
    // РЎСЋРґР° РЅРёРєРѕРіРґР° РЅРµ РїРѕРїР°РґС‘Рј
}

static bool LogException(Exception ex)
{
    Console.Error.WriteLine(ex);
    return false; // Р’СЃРµРіРґР° false в†’ catch РЅРµ СЃСЂР°Р±Р°С‚С‹РІР°РµС‚, exception Р»РµС‚РёС‚ РґР°Р»СЊС€Рµ
}
```

### Custom Exceptions вЂ” РїСЂР°РІРёР»СЊРЅРѕРµ СЃРѕР·РґР°РЅРёРµ

```csharp
/// <summary>
/// Р”РѕРјРµРЅРЅРѕРµ РёСЃРєР»СЋС‡РµРЅРёРµ: РЅРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ СЃСЂРµРґСЃС‚РІ РЅР° СЃС‡С‘С‚Рµ.
/// </summary>
public sealed class InsufficientFundsException : Exception
{
    public decimal CurrentBalance { get; }
    public decimal RequestedAmount { get; }

    public InsufficientFundsException(decimal currentBalance, decimal requestedAmount)
        : base($"РќРµРґРѕСЃС‚Р°С‚РѕС‡РЅРѕ СЃСЂРµРґСЃС‚РІ. Р‘Р°Р»Р°РЅСЃ: {currentBalance:C}, Р·Р°РїСЂРѕС€РµРЅРѕ: {requestedAmount:C}")
    {
        CurrentBalance = currentBalance;
        RequestedAmount = requestedAmount;
    }

    public InsufficientFundsException(string message) : base(message) { }

    public InsufficientFundsException(string message, Exception innerException)
        : base(message, innerException) { }
}

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ
public void Withdraw(decimal amount)
{
    if (amount > Balance)
        throw new InsufficientFundsException(Balance, amount);

    Balance -= amount;
}
```

### Best Practices вЂ” РёС‚РѕРіРѕРІР°СЏ С‚Р°Р±Р»РёС†Р°

| РџСЂР°РІРёР»Рѕ | РџСЂРёРјРµСЂ |
|---------|--------|
| Р›РѕРІРёС‚СЊ РєРѕРЅРєСЂРµС‚РЅС‹Рµ С‚РёРїС‹ | `catch (FormatException)`, РЅРµ `catch (Exception)` |
| РќРµ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ exceptions РґР»СЏ control flow | РСЃРїРѕР»СЊР·РѕРІР°С‚СЊ `TryParse`, `TryGetValue`, `Result<T>` |
| Р’СЃРµРіРґР° РІРєР»СЋС‡Р°С‚СЊ inner exception | `throw new AppException("msg", ex)` |
| `throw`, РЅРµ `throw ex` | РЎРѕС…СЂР°РЅСЏРµС‚ stack trace |
| `finally` РґР»СЏ РѕС‡РёСЃС‚РєРё | РР»Рё Р»СѓС‡С€Рµ вЂ” `using` / `await using` |
| Exception filters РІРјРµСЃС‚Рѕ catch + rethrow | `catch (Exception ex) when (ShouldHandle(ex))` |
| `ArgumentNullException.ThrowIfNull()` | Р’РјРµСЃС‚Рѕ СЂСѓС‡РЅС‹С… `if (x is null) throw` |

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: РљРѕРіРґР° СЃРѕР·РґР°РІР°С‚СЊ РєР°СЃС‚РѕРјРЅС‹Рµ РёСЃРєР»СЋС‡РµРЅРёСЏ?**
> РљРѕРіРґР° СЃС‚Р°РЅРґР°СЂС‚РЅС‹Рµ РЅРµ РїРµСЂРµРґР°СЋС‚ Р±РёР·РЅРµСЃ-СЃРјС‹СЃР»: `OrderNotFoundException`, `InsufficientBalanceException`. РќР°СЃР»РµРґСѓР№ РѕС‚ `Exception` (РЅРµ `ApplicationException`). РљРѕРЅСЃС‚СЂСѓРєС‚РѕСЂС‹ СЃ `message` Рё `innerException`. РќРµ РёСЃРїРѕР»СЊР·СѓР№ РёСЃРєР»СЋС‡РµРЅРёСЏ РґР»СЏ flow control вЂ” stack unwinding РґРѕСЂРѕРі.

---

## Strings (СѓРіР»СѓР±Р»С‘РЅРЅРѕ)

### Immutability Рё String Pool

РЎС‚СЂРѕРєРё РІ C# **РЅРµРёР·РјРµРЅСЏРµРјС‹** (immutable). Р›СЋР±Р°СЏ В«РјРѕРґРёС„РёРєР°С†РёСЏВ» СЃРѕР·РґР°С‘С‚ **РЅРѕРІС‹Р№** РѕР±СЉРµРєС‚.

```csharp
string greeting = "Hello";
string modified = greeting.Replace("Hello", "Hi"); // РќРѕРІС‹Р№ РѕР±СЉРµРєС‚ РІ heap
// greeting РїРѕ-РїСЂРµР¶РЅРµРјСѓ "Hello"

// String interning вЂ” .NET Р°РІС‚РѕРјР°С‚РёС‡РµСЃРєРё РёРЅС‚РµСЂРЅРёСЂСѓРµС‚ Р»РёС‚РµСЂР°Р»С‹
string a = "hello";
string b = "hello";
Console.WriteLine(ReferenceEquals(a, b)); // True вЂ” РѕРґРёРЅ РѕР±СЉРµРєС‚ РІ РїР°РјСЏС‚Рё

// Р СѓС‡РЅРѕРµ РёРЅС‚РµСЂРЅРёСЂРѕРІР°РЅРёРµ
string dynamic1 = new string(['h', 'e', 'l', 'l', 'o']);
string dynamic2 = new string(['h', 'e', 'l', 'l', 'o']);
Console.WriteLine(ReferenceEquals(dynamic1, dynamic2));               // False
Console.WriteLine(ReferenceEquals(string.Intern(dynamic1),
                                  string.Intern(dynamic2)));          // True

// РџСЂРѕРІРµСЂРєР°, РёРЅС‚РµСЂРЅРёСЂРѕРІР°РЅР° Р»Рё СЃС‚СЂРѕРєР°
bool isInterned = string.IsInterned(dynamic1) is not null;
```

### StringBuilder вЂ” РєРѕРіРґР° РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ

**РџСЂР°РІРёР»Рѕ:** РµСЃР»Рё РєРѕРЅРєР°С‚РµРЅР°С†РёСЏ СЃС‚СЂРѕРє РІ С†РёРєР»Рµ РёР»Рё > 3вЂ“4 РѕРїРµСЂР°С†РёР№ вЂ” РёСЃРїРѕР»СЊР·СѓР№ `StringBuilder`.

```csharp
// РџР›РћРҐРћ: РєР°Р¶РґР°СЏ += СЃРѕР·РґР°С‘С‚ РЅРѕРІС‹Р№ РѕР±СЉРµРєС‚ (O(nВІ) РїРѕ РїР°РјСЏС‚Рё)
string result = "";
for (int i = 0; i < 10_000; i++)
    result += i.ToString();

// РҐРћР РћРЁРћ: StringBuilder вЂ” РјСѓС‚Р°Р±РµР»СЊРЅС‹Р№ Р±СѓС„РµСЂ (O(n))
var sb = new StringBuilder(capacity: 64_000); // РЈРєР°Р·С‹РІР°РµРј capacity Р·Р°СЂР°РЅРµРµ
for (int i = 0; i < 10_000; i++)
    sb.Append(i);

string output = sb.ToString();

// Fluent API
string html = new StringBuilder(256)
    .Append("<div class=\"card\">")
    .Append("<h2>").Append(title).Append("</h2>")
    .Append("<p>").Append(description).Append("</p>")
    .Append("</div>")
    .ToString();
```

### ReadOnlySpan<char> вЂ” РїР°СЂСЃРёРЅРі Р±РµР· Р°Р»Р»РѕРєР°С†РёР№

```csharp
// РџР°СЂСЃРёРЅРі СЃС‚СЂРѕРєРё "2024-01-15" Р±РµР· СЃРѕР·РґР°РЅРёСЏ РїРѕРґСЃС‚СЂРѕРє
ReadOnlySpan<char> date = "2024-01-15".AsSpan();

int year  = int.Parse(date[..4]);       // "2024" вЂ” Р±РµР· Р°Р»Р»РѕРєР°С†РёРё
int month = int.Parse(date[5..7]);      // "01"
int day   = int.Parse(date[8..10]);     // "15"

// Р Р°Р·РґРµР»РµРЅРёРµ РїРѕ С‚РѕРєРµРЅР°Рј Р±РµР· Р°Р»Р»РѕРєР°С†РёР№
ReadOnlySpan<char> csv = "Alice,30,Developer".AsSpan();
int firstComma = csv.IndexOf(',');
ReadOnlySpan<char> name = csv[..firstComma]; // "Alice" вЂ” zero-alloc

// РЎСЂР°РІРЅРµРЅРёРµ Р±РµР· Р°Р»Р»РѕРєР°С†РёР№
bool startsWith = csv.StartsWith("Alice");
bool contains = csv.Contains("Developer", StringComparison.Ordinal);
```

### StringComparison вЂ” РїСЂРѕРёР·РІРѕРґРёС‚РµР»СЊРЅРѕСЃС‚СЊ Рё РєРѕСЂСЂРµРєС‚РЅРѕСЃС‚СЊ

```csharp
string a = "hello";
string b = "HELLO";

// Ordinal вЂ” РїРѕР±Р°Р№С‚РѕРІРѕРµ СЃСЂР°РІРЅРµРЅРёРµ, РЎРђРњРћР• Р‘Р«РЎРўР РћР•
a.Equals(b, StringComparison.Ordinal);                // false
a.Equals(b, StringComparison.OrdinalIgnoreCase);      // true вЂ” РґР»СЏ case-insensitive

// CurrentCulture вЂ” СѓС‡РёС‚С‹РІР°РµС‚ Р»РѕРєР°Р»СЊ (РјРµРґР»РµРЅРЅРµРµ, РґР»СЏ UI)
a.Equals(b, StringComparison.CurrentCultureIgnoreCase); // true

// РџР РђР’РР›Рћ: РґР»СЏ РІРЅСѓС‚СЂРµРЅРЅРµР№ Р»РѕРіРёРєРё вЂ” Ordinal, РґР»СЏ РѕС‚РѕР±СЂР°Р¶РµРЅРёСЏ РїРѕР»СЊР·РѕРІР°С‚РµР»СЋ вЂ” CurrentCulture
// Р”Р»СЏ Dictionary РєР»СЋС‡РµР№:
var dict = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
dict["Hello"] = 1;
Console.WriteLine(dict["hello"]); // 1 вЂ” СЂРµРіРёСЃС‚СЂРѕРЅРµР·Р°РІРёСЃРёРјС‹Р№ РїРѕРёСЃРє
```

### Raw String Literals (C# 11) Рё UTF-8 Literals

```csharp
// Raw string literals вЂ” Р±РµР· СЌРєСЂР°РЅРёСЂРѕРІР°РЅРёСЏ
string json = """
    {
        "name": "Alice",
        "age": 30,
        "tags": ["dev", "lead"]
    }
    """;

// РРЅС‚РµСЂРїРѕР»СЏС†РёСЏ РІ raw strings (РєРѕР»РёС‡РµСЃС‚РІРѕ $ = РєРѕР»РёС‡РµСЃС‚РІРѕ { РґР»СЏ РёРЅС‚РµСЂРїРѕР»СЏС†РёРё)
string name = "Alice";
string template = $$"""
    {
        "name": "{{name}}",
        "query": "SELECT * FROM users WHERE name = '{name}'"
    }
    """;

// UTF-8 string literals (C# 11) вЂ” РІРѕР·РІСЂР°С‰Р°РµС‚ ReadOnlySpan<byte>
ReadOnlySpan<byte> utf8 = "Content-Type: application/json"u8;
// Р‘РµР· Р°Р»Р»РѕРєР°С†РёРё СЃС‚СЂРѕРєРё вЂ” РёРґРµР°Р»СЊРЅРѕ РґР»СЏ HTTP headers, РїСЂРѕС‚РѕРєРѕР»РѕРІ
```

### String.Create вЂ” РѕРїС‚РёРјРёР·Р°С†РёСЏ СЃРѕР·РґР°РЅРёСЏ СЃС‚СЂРѕРє

```csharp
// РЎРѕР·РґР°С‘С‚ СЃС‚СЂРѕРєСѓ С„РёРєСЃРёСЂРѕРІР°РЅРЅРѕР№ РґР»РёРЅС‹, Р·Р°РїРѕР»РЅСЏСЏ С‡РµСЂРµР· Span (РѕРґРЅР° Р°Р»Р»РѕРєР°С†РёСЏ)
string hex = string.Create(8, 0xDEADBEEF, static (span, value) =>
{
    for (int i = span.Length - 1; i >= 0; i--)
    {
        int nibble = (int)(value & 0xF);
        span[i] = (char)(nibble < 10 ? '0' + nibble : 'A' + nibble - 10);
        value >>= 4;
    }
});
// hex == "DEADBEEF"

// Р“РµРЅРµСЂР°С†РёСЏ СЃР»СѓС‡Р°Р№РЅРѕР№ СЃС‚СЂРѕРєРё Р±РµР· РїСЂРѕРјРµР¶СѓС‚РѕС‡РЅС‹С… Р°Р»Р»РѕРєР°С†РёР№
string randomId = string.Create(16, Random.Shared, static (span, rng) =>
{
    const string chars = "abcdefghijklmnopqrstuvwxyz0123456789";
    for (int i = 0; i < span.Length; i++)
        span[i] = chars[rng.Next(chars.Length)];
});
```

### РћСЃРЅРѕРІРЅС‹Рµ РјРµС‚РѕРґС‹ СЃС‚СЂРѕРє

```csharp
string text = "  Hello, World! Hello, C#!  ";

// РџРѕРёСЃРє
bool has      = text.Contains("World");                          // true
int idx       = text.IndexOf("Hello");                           // 2
int lastIdx   = text.LastIndexOf("Hello");                       // 17
bool starts   = text.TrimStart().StartsWith("Hello");            // true
bool ends     = text.TrimEnd().EndsWith("C#!");                  // true

// РњРѕРґРёС„РёРєР°С†РёСЏ (РєР°Р¶РґС‹Р№ РјРµС‚РѕРґ РІРѕР·РІСЂР°С‰Р°РµС‚ РќРћР’РЈР® СЃС‚СЂРѕРєСѓ)
string replaced = text.Replace("Hello", "Hi");                   // "  Hi, World! Hi, C#!  "
string trimmed  = text.Trim();                                   // "Hello, World! Hello, C#!"
string upper    = text.ToUpperInvariant();                        // "  HELLO, WORLD! HELLO, C#!  "
string lower    = text.ToLowerInvariant();                        // "  hello, world! hello, c#!  "
string sub      = "Hello, World!".Substring(7, 5);               // "World" (legacy)
string sub2     = "Hello, World!"[7..12];                        // "World" (modern вЂ” Range)

// Р Р°Р·РґРµР»РµРЅРёРµ Рё РѕР±СЉРµРґРёРЅРµРЅРёРµ
string[] parts  = "one,two,three".Split(',');                    // ["one", "two", "three"]
string[] parts2 = "one,,two".Split(',', StringSplitOptions.RemoveEmptyEntries); // ["one", "two"]
string joined   = string.Join(" | ", parts);                     // "one | two | three"

// РџСЂРѕРІРµСЂРєРё
bool empty      = string.IsNullOrEmpty("");                      // true
bool blank      = string.IsNullOrWhiteSpace("   ");              // true
```

### String Formatting

```csharp
string name = "Alice";
decimal price = 1234.5m;
DateTime now = DateTime.Now;

// РРЅС‚РµСЂРїРѕР»СЏС†РёСЏ вЂ” РїСЂРµРґРїРѕС‡С‚РёС‚РµР»СЊРЅС‹Р№ СЃРїРѕСЃРѕР±
string msg1 = $"РџСЂРёРІРµС‚, {name}! Р¦РµРЅР°: {price:C2}. Р”Р°С‚Р°: {now:yyyy-MM-dd}";

// Р’С‹СЂР°РІРЅРёРІР°РЅРёРµ РІ РёРЅС‚РµСЂРїРѕР»СЏС†РёРё
string aligned = $"|{"Name",-20}|{"Price",10:C2}|";
// |Name                |  $1,234.50|

// String.Format (РґР»СЏ С€Р°Р±Р»РѕРЅРѕРІ РёР· СЂРµСЃСѓСЂСЃРѕРІ)
string msg2 = string.Format("РџРѕР»СЊР·РѕРІР°С‚РµР»СЊ {0} Р·Р°РїР»Р°С‚РёР» {1:C2}", name, price);

// Р§РёСЃР»РѕРІС‹Рµ С„РѕСЂРјР°С‚С‹
$"{1234567:N0}"    // "1,234,567"      вЂ” СЃ СЂР°Р·РґРµР»РёС‚РµР»СЏРјРё
$"{0.856:P1}"      // "85.6%"          вЂ” РїСЂРѕС†РµРЅС‚С‹
$"{255:X2}"        // "FF"             вЂ” hex
$"{42:D5}"         // "00042"          вЂ” РґРѕРїРѕР»РЅРµРЅРёРµ РЅСѓР»СЏРјРё
```

### Char вЂ” СЂР°Р±РѕС‚Р° СЃ СЃРёРјРІРѕР»Р°РјРё

```csharp
char c = 'A';

char.IsDigit('5');           // true
char.IsLetter('A');          // true
char.IsWhiteSpace(' ');      // true
char.IsUpper('A');           // true
char.IsLower('a');           // true
char.IsLetterOrDigit('3');   // true
char.IsPunctuation('.');     // true

char upper = char.ToUpperInvariant('a');  // 'A'
char lower = char.ToLowerInvariant('Z');  // 'z'
int numeric = (int)char.GetNumericValue('7'); // 7
```

---

## File I/O

### File вЂ” СЃС‚Р°С‚РёС‡РµСЃРєРёРµ РјРµС‚РѕРґС‹ (Р±С‹СЃС‚СЂС‹Р№ РґРѕСЃС‚СѓРї)

```csharp
string path = @"C:\Data\config.txt";

// Р—Р°РїРёСЃСЊ
File.WriteAllText(path, "Hello, World!");
File.WriteAllLines(path, ["line1", "line2", "line3"]);
File.AppendAllText(path, "\nnew line");

// Р§С‚РµРЅРёРµ
string content      = File.ReadAllText(path);
string[] lines      = File.ReadAllLines(path);
byte[] bytes        = File.ReadAllBytes(path);

// РџСЂРѕРІРµСЂРєРё
bool exists         = File.Exists(path);
DateTime created    = File.GetCreationTime(path);
DateTime modified   = File.GetLastWriteTime(path);

// РћРїРµСЂР°С†РёРё
File.Copy(path, @"C:\Data\config_backup.txt", overwrite: true);
File.Move(path, @"C:\Data\config_new.txt", overwrite: true);
File.Delete(path);
```

### Async File Operations

```csharp
// Р’РЎР•Р“Р”Рђ РёСЃРїРѕР»СЊР·СѓР№ async РґР»СЏ I/O РІ ASP.NET Core / GUI РїСЂРёР»РѕР¶РµРЅРёСЏС…
string content = await File.ReadAllTextAsync("data.json", cancellationToken);
string[] lines = await File.ReadAllLinesAsync("log.txt", cancellationToken);

await File.WriteAllTextAsync("output.txt", result, cancellationToken);
await File.WriteAllLinesAsync("output.txt", lines, cancellationToken);
await File.WriteAllBytesAsync("image.png", imageBytes, cancellationToken);

// Append
await File.AppendAllTextAsync("log.txt", $"{DateTime.UtcNow}: СЃРѕР±С‹С‚РёРµ\n");
```

### FileInfo вЂ” instance-based

```csharp
var fileInfo = new FileInfo(@"C:\Data\report.pdf");

if (fileInfo.Exists)
{
    Console.WriteLine($"РРјСЏ:       {fileInfo.Name}");           // report.pdf
    Console.WriteLine($"Р Р°Р·РјРµСЂ:    {fileInfo.Length} Р±Р°Р№С‚");    // 1234567
    Console.WriteLine($"РљР°С‚Р°Р»РѕРі:   {fileInfo.DirectoryName}");  // C:\Data
    Console.WriteLine($"Р Р°СЃС€.:     {fileInfo.Extension}");      // .pdf
    Console.WriteLine($"РЎРѕР·РґР°РЅ:    {fileInfo.CreationTime}");
    Console.WriteLine($"РР·РјРµРЅС‘РЅ:   {fileInfo.LastWriteTime}");
    Console.WriteLine($"ReadOnly:  {fileInfo.IsReadOnly}");

    fileInfo.CopyTo(@"C:\Backup\report.pdf", overwrite: true);
}
```

### Directory Рё DirectoryInfo

```csharp
string dir = @"C:\Projects\MyApp";

// Directory (static)
Directory.CreateDirectory(dir);                         // РЎРѕР·РґР°С‘С‚ РІР»РѕР¶РµРЅРЅС‹Рµ РєР°С‚Р°Р»РѕРіРё
bool dirExists      = Directory.Exists(dir);
string[] files      = Directory.GetFiles(dir, "*.cs");
string[] subDirs    = Directory.GetDirectories(dir);
string[] allCsFiles = Directory.GetFiles(dir, "*.cs", SearchOption.AllDirectories);

// EnumerateFiles вЂ” lazy, РЅРµ Р·Р°РіСЂСѓР¶Р°РµС‚ РІСЃС‘ РІ РїР°РјСЏС‚СЊ
foreach (string file in Directory.EnumerateFiles(dir, "*.log", SearchOption.AllDirectories))
{
    Console.WriteLine(file);
}

Directory.Move(@"C:\Old", @"C:\New");
Directory.Delete(dir, recursive: true); // recursive СѓРґР°Р»СЏРµС‚ СЃРѕРґРµСЂР¶РёРјРѕРµ

// DirectoryInfo (instance)
var dirInfo = new DirectoryInfo(dir);
FileInfo[] csFiles = dirInfo.GetFiles("*.cs", SearchOption.AllDirectories);
DirectoryInfo[] subDirectories = dirInfo.GetDirectories();
```

### Path вЂ” Р±РµР·РѕРїР°СЃРЅР°СЏ СЂР°Р±РѕС‚Р° СЃ РїСѓС‚СЏРјРё

```csharp
// Р’РЎР•Р“Р”Рђ РёСЃРїРѕР»СЊР·СѓР№ Path.Combine вЂ” РѕРЅ РїСЂР°РІРёР»СЊРЅРѕ РѕР±СЂР°Р±Р°С‚С‹РІР°РµС‚ СЂР°Р·РґРµР»РёС‚РµР»Рё
string full = Path.Combine("C:", "Users", "mos26", "file.txt");
// C:\Users\mos26\file.txt

Path.GetFileName(@"C:\Data\report.pdf");       // "report.pdf"
Path.GetFileNameWithoutExtension(@"report.pdf"); // "report"
Path.GetExtension(@"report.pdf");               // ".pdf"
Path.GetDirectoryName(@"C:\Data\report.pdf");   // "C:\Data"
Path.GetFullPath("relative/path.txt");          // РђР±СЃРѕР»СЋС‚РЅС‹Р№ РїСѓС‚СЊ

Path.GetTempPath();                             // РџСѓС‚СЊ Рє temp-РєР°С‚Р°Р»РѕРіСѓ
Path.GetTempFileName();                         // РЎРѕР·РґР°С‘С‚ РІСЂРµРјРµРЅРЅС‹Р№ С„Р°Р№Р»
Path.GetRandomFileName();                       // "a3b2c1d4.tmp" (РЅРµ СЃРѕР·РґР°С‘С‚ С„Р°Р№Р»)

// .NET 8+
Path.Exists(@"C:\Data\file.txt");               // РџСЂРѕРІРµСЂСЏРµС‚ Рё С„Р°Р№Р», Рё РєР°С‚Р°Р»РѕРі
```

### StreamReader / StreamWriter вЂ” РґР»СЏ Р±РѕР»СЊС€РёС… С„Р°Р№Р»РѕРІ

```csharp
// Р§С‚РµРЅРёРµ РїРѕСЃС‚СЂРѕС‡РЅРѕ вЂ” РЅРµ Р·Р°РіСЂСѓР¶Р°РµС‚ РІРµСЃСЊ С„Р°Р№Р» РІ РїР°РјСЏС‚СЊ
await using var reader = new StreamReader("large_file.csv", Encoding.UTF8);
while (await reader.ReadLineAsync() is { } line)
{
    ProcessLine(line);
}

// Р—Р°РїРёСЃСЊ
await using var writer = new StreamWriter("output.txt", append: false, Encoding.UTF8);
await writer.WriteLineAsync("РџРµСЂРІР°СЏ СЃС‚СЂРѕРєР°");
await writer.WriteLineAsync("Р’С‚РѕСЂР°СЏ СЃС‚СЂРѕРєР°");
await writer.FlushAsync(); // РџСЂРёРЅСѓРґРёС‚РµР»СЊРЅС‹Р№ СЃР±СЂРѕСЃ Р±СѓС„РµСЂР°
```

### FileStream вЂ” binary РґР°РЅРЅС‹Рµ

```csharp
// Р—Р°РїРёСЃСЊ binary
await using var writeStream = new FileStream(
    "data.bin",
    FileMode.Create,
    FileAccess.Write,
    FileShare.None,
    bufferSize: 4096,
    useAsync: true);

byte[] data = [0x48, 0x65, 0x6C, 0x6C, 0x6F];
await writeStream.WriteAsync(data, cancellationToken);

// Р§С‚РµРЅРёРµ binary
await using var readStream = new FileStream("data.bin", FileMode.Open, FileAccess.Read,
    FileShare.Read, bufferSize: 4096, useAsync: true);

byte[] buffer = new byte[(int)readStream.Length]; // Length вЂ” long, РЅСѓР¶РµРЅ РєР°СЃС‚
int bytesRead = await readStream.ReadAsync(buffer, cancellationToken);
```

### using Рё IDisposable

```csharp
// using statement вЂ” Р°РІС‚РѕРјР°С‚РёС‡РµСЃРєРёР№ Dispose() РІ finally
using (var reader = new StreamReader("file.txt"))
{
    string content = await reader.ReadToEndAsync();
} // reader.Dispose() РІС‹Р·С‹РІР°РµС‚СЃСЏ Р·РґРµСЃСЊ

// using declaration (C# 8) вЂ” Dispose РїСЂРё РІС‹С…РѕРґРµ РёР· scope
using var writer = new StreamWriter("log.txt", append: true);
await writer.WriteLineAsync("log entry");
// writer.Dispose() РїСЂРё РІС‹С…РѕРґРµ РёР· РјРµС‚РѕРґР°

// await using вЂ” РґР»СЏ IAsyncDisposable
await using var dbConnection = new NpgsqlConnection(connectionString);
await dbConnection.OpenAsync(cancellationToken);

// РЎРІРѕР№ IDisposable
public sealed class TempFileScope : IDisposable
{
    public string FilePath { get; } = Path.GetTempFileName();

    public void Dispose()
    {
        if (File.Exists(FilePath))
            File.Delete(FilePath);
    }
}

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ
using var temp = new TempFileScope();
await File.WriteAllTextAsync(temp.FilePath, "РІСЂРµРјРµРЅРЅС‹Рµ РґР°РЅРЅС‹Рµ");
// Р¤Р°Р№Р» Р°РІС‚РѕРјР°С‚РёС‡РµСЃРєРё СѓРґР°Р»РёС‚СЃСЏ
```

---

## JSON

### System.Text.Json вЂ” СЃРµСЂРёР°Р»РёР·Р°С†РёСЏ / РґРµСЃРµСЂРёР°Р»РёР·Р°С†РёСЏ

```csharp
using System.Text.Json;
using System.Text.Json.Serialization;

// Р‘Р°Р·РѕРІР°СЏ СЃРµСЂРёР°Р»РёР·Р°С†РёСЏ
var order = new Order("ORD-001", "Alice", 299.99m);
string json = JsonSerializer.Serialize(order);

// Р”РµСЃРµСЂРёР°Р»РёР·Р°С†РёСЏ
Order? deserialized = JsonSerializer.Deserialize<Order>(json);

// РЎ РЅР°СЃС‚СЂРѕР№РєР°РјРё
var options = new JsonSerializerOptions
{
    PropertyNamingPolicy   = JsonNamingPolicy.CamelCase,    // camelCase РІ JSON
    WriteIndented          = true,                           // РљСЂР°СЃРёРІС‹Р№ РІС‹РІРѕРґ
    PropertyNameCaseInsensitive = true,                      // РќРµС‡СѓРІСЃС‚РІРёС‚РµР»РµРЅ Рє СЂРµРіРёСЃС‚СЂСѓ РїСЂРё С‡С‚РµРЅРёРё
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull, // РџСЂРѕРїСѓСЃРєР°С‚СЊ null
    Converters             = { new JsonStringEnumConverter() },   // Enum РєР°Рє СЃС‚СЂРѕРєРё
};

string prettyJson = JsonSerializer.Serialize(order, options);
Order? parsed     = JsonSerializer.Deserialize<Order>(prettyJson, options);
```

### РђС‚СЂРёР±СѓС‚С‹

```csharp
public sealed record OrderDto
{
    [JsonPropertyName("order_id")]           // РРјСЏ РІ JSON
    public required string Id { get; init; }

    [JsonPropertyName("customer_name")]
    public required string Customer { get; init; }

    [JsonIgnore]                              // РќРµ СЃРµСЂРёР°Р»РёР·СѓРµС‚СЃСЏ
    public string InternalCode { get; init; } = "";

    [JsonInclude]                             // Р’РєР»СЋС‡РёС‚СЊ private/internal РїРѕР»Рµ
    internal decimal discount = 0.1m;

    [JsonConverter(typeof(JsonStringEnumConverter))]
    public OrderStatus Status { get; init; }

    [JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingDefault)]
    public int Priority { get; init; }        // РџСЂРѕРїСѓСЃРєР°РµС‚СЃСЏ РµСЃР»Рё 0
}
```

### Source Generators вЂ” [JsonSerializable] (AOT-friendly)

```csharp
// Р“РµРЅРµСЂР°С†РёСЏ СЃРµСЂРёР°Р»РёР·Р°С‚РѕСЂР° РЅР° СЌС‚Р°РїРµ РєРѕРјРїРёР»СЏС†РёРё вЂ” Р±С‹СЃС‚СЂРµРµ, Р±РµР· reflection
[JsonSerializable(typeof(OrderDto))]
[JsonSerializable(typeof(List<OrderDto>))]
[JsonSourceGenerationOptions(
    PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    WriteIndented = true,
    DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull)]
public partial class AppJsonContext : JsonSerializerContext;

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ
string json = JsonSerializer.Serialize(order, AppJsonContext.Default.OrderDto);
OrderDto? result = JsonSerializer.Deserialize(json, AppJsonContext.Default.OrderDto);

// Р’ ASP.NET Core Minimal API
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.TypeInfoResolverChain.Insert(0, AppJsonContext.Default);
});
```

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: System.Text.Json vs Newtonsoft вЂ” РєРѕРіРґР° С‡С‚Рѕ?**
> **STJ** вЂ” РІСЃС‚СЂРѕРµРЅ, Р±С‹СЃС‚СЂРµРµ, РјРµРЅСЊС€Рµ Р°Р»Р»РѕРєР°С†РёР№, Source Generators РґР»СЏ AOT. РџРѕ СѓРјРѕР»С‡Р°РЅРёСЋ РґР»СЏ РЅРѕРІС‹С… РїСЂРѕРµРєС‚РѕРІ.
>
> **Newtonsoft** вЂ” Р±РѕРіР°С‡Рµ С„РёС‡Р°РјРё (JToken, JsonPath, СЃР»РѕР¶РЅС‹Р№ РјР°РїРїРёРЅРі). Р”Р»СЏ legacy РёР»Рё РєРѕРіРґР° STJ РЅРµ РїРѕРєСЂС‹РІР°РµС‚ СЃС†РµРЅР°СЂРёР№.
>
> **РџСЂР°РІРёР»Рѕ:** STJ РґР»СЏ API (DTO). Newtonsoft вЂ” РґР»СЏ СЃР»РѕР¶РЅС‹С… С‚СЂР°РЅСЃС„РѕСЂРјР°С†РёР№ JSON.

### JsonDocument вЂ” DOM-based parsing

```csharp
// Р”Р»СЏ С‡С‚РµРЅРёСЏ JSON Р±РµР· РґРµСЃРµСЂРёР°Р»РёР·Р°С†РёРё РІ РѕР±СЉРµРєС‚
string json = """{"name": "Alice", "scores": [95, 87, 92], "address": {"city": "Moscow"}}""";

using JsonDocument doc = JsonDocument.Parse(json);
JsonElement root = doc.RootElement;

string name = root.GetProperty("name").GetString()!;           // "Alice"
int firstScore = root.GetProperty("scores")[0].GetInt32();     // 95
string city = root.GetProperty("address").GetProperty("city").GetString()!; // "Moscow"

// Р‘РµР·РѕРїР°СЃРЅС‹Р№ РґРѕСЃС‚СѓРї
if (root.TryGetProperty("age", out JsonElement ageElement))
{
    int age = ageElement.GetInt32();
}

// РџРµСЂРµС‡РёСЃР»РµРЅРёРµ СЃРІРѕР№СЃС‚РІ
foreach (JsonProperty prop in root.EnumerateObject())
{
    Console.WriteLine($"{prop.Name}: {prop.Value.ValueKind}");
}
```

### Utf8JsonReader / Utf8JsonWriter вЂ” high-performance

```csharp
// Utf8JsonWriter вЂ” РјРёРЅРёРјР°Р»СЊРЅС‹Рµ Р°Р»Р»РѕРєР°С†РёРё
using var stream = new MemoryStream();
using var writer = new Utf8JsonWriter(stream, new JsonWriterOptions { Indented = true }); // IDisposable, РЅРµ IAsyncDisposable

writer.WriteStartObject();
writer.WriteString("name", "Alice");
writer.WriteNumber("age", 30);
writer.WriteStartArray("tags");
writer.WriteStringValue("developer");
writer.WriteStringValue("lead");
writer.WriteEndArray();
writer.WriteEndObject();
await writer.FlushAsync();

string result = Encoding.UTF8.GetString(stream.ToArray());

// Utf8JsonReader вЂ” РїРѕС‚РѕРєРѕРІРѕРµ С‡С‚РµРЅРёРµ
byte[] jsonBytes = Encoding.UTF8.GetBytes("""{"name":"Alice","age":30}""");
var reader2 = new Utf8JsonReader(jsonBytes);

while (reader2.Read())
{
    if (reader2.TokenType == JsonTokenType.PropertyName && reader2.ValueTextEquals("name"u8))
    {
        reader2.Read();
        string value = reader2.GetString()!; // "Alice"
    }
}
```

### System.Text.Json vs Newtonsoft.Json

| РљСЂРёС‚РµСЂРёР№ | System.Text.Json | Newtonsoft.Json |
|----------|------------------|----------------|
| РџСЂРѕРёР·РІРѕРґРёС‚РµР»СЊРЅРѕСЃС‚СЊ | Р‘С‹СЃС‚СЂРµРµ РІ 1.5вЂ“2x | РњРµРґР»РµРЅРЅРµРµ |
| РђР»Р»РѕРєР°С†РёРё | РњРµРЅСЊС€Рµ (Utf8-native) | Р‘РѕР»СЊС€Рµ |
| AOT / Trimming | Source generators | РќРµ РїРѕРґРґРµСЂР¶РёРІР°РµС‚ |
| Р“РёР±РєРѕСЃС‚СЊ | РћРіСЂР°РЅРёС‡РµРЅРЅР°СЏ | РњР°РєСЃРёРјР°Р»СЊРЅР°СЏ |
| `$type` polymorphism | `[JsonDerivedType]` (.NET 7+) | `TypeNameHandling` |
| Р¦РёРєР»РёС‡РµСЃРєРёРµ СЃСЃС‹Р»РєРё | `ReferenceHandler.Preserve` | `PreserveReferencesHandling` |
| LINQ to JSON | `JsonNode` | `JObject` / `JArray` |
| Р—СЂРµР»РѕСЃС‚СЊ | Р Р°Р·РІРёРІР°РµС‚СЃСЏ | РЎС‚Р°Р±РёР»СЊРЅР°, feature-complete |
| Р РµРєРѕРјРµРЅРґР°С†РёСЏ | **РџРѕ СѓРјРѕР»С‡Р°РЅРёСЋ РґР»СЏ РЅРѕРІС‹С… РїСЂРѕРµРєС‚РѕРІ** | Legacy / СЃР»РѕР¶РЅС‹Рµ СЃС†РµРЅР°СЂРёРё |

---

## Regex

### Р‘Р°Р·РѕРІРѕРµ РёСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ

```csharp
using System.Text.RegularExpressions;

string input = "РўРµР»РµС„РѕРЅ: +7-999-123-4567, email: alice@example.com";

// РџСЂРѕРІРµСЂРєР° СЃРѕРІРїР°РґРµРЅРёСЏ
bool hasPhone = Regex.IsMatch(input, @"\+7-\d{3}-\d{3}-\d{4}");  // true

// РџРѕРёСЃРє РїРµСЂРІРѕРіРѕ СЃРѕРІРїР°РґРµРЅРёСЏ
Match match = Regex.Match(input, @"[\w.+-]+@[\w-]+\.[\w.]+");
if (match.Success)
    Console.WriteLine(match.Value); // "alice@example.com"

// Р’СЃРµ СЃРѕРІРїР°РґРµРЅРёСЏ
MatchCollection matches = Regex.Matches(input, @"\d+");
foreach (Match m in matches)
    Console.WriteLine(m.Value); // "7", "999", "123", "4567"

// Р—Р°РјРµРЅР°
string masked = Regex.Replace(input, @"\d", "*");
// "РўРµР»РµС„РѕРЅ: +*-***-***-****, email: alice@example.com"

// Р Р°Р·РґРµР»РµРЅРёРµ
string[] tokens = Regex.Split("one, two,  three", @",\s*");
// ["one", "two", "three"]
```

### Р“СЂСѓРїРїС‹ Рё РёРјРµРЅРѕРІР°РЅРЅС‹Рµ РіСЂСѓРїРїС‹

```csharp
string logLine = "2024-01-15 14:30:45 [ERROR] Connection timeout";

var pattern = @"(?<date>\d{4}-\d{2}-\d{2})\s+(?<time>\d{2}:\d{2}:\d{2})\s+\[(?<level>\w+)\]\s+(?<msg>.+)";
Match m = Regex.Match(logLine, pattern);

if (m.Success)
{
    string date  = m.Groups["date"].Value;   // "2024-01-15"
    string time  = m.Groups["time"].Value;   // "14:30:45"
    string level = m.Groups["level"].Value;  // "ERROR"
    string msg   = m.Groups["msg"].Value;    // "Connection timeout"
}
```

### Generated Regex (C# 11 / .NET 7+) вЂ” compile-time

```csharp
public partial class LogParser
{
    // РљРѕРјРїРёР»РёСЂСѓРµС‚СЃСЏ РІ IL РЅР° СЌС‚Р°РїРµ СЃР±РѕСЂРєРё вЂ” РјР°РєСЃРёРјР°Р»СЊРЅР°СЏ РїСЂРѕРёР·РІРѕРґРёС‚РµР»СЊРЅРѕСЃС‚СЊ
    [GeneratedRegex(
        @"(?<date>\d{4}-\d{2}-\d{2})\s+\[(?<level>\w+)\]\s+(?<msg>.+)",
        RegexOptions.ExplicitCapture)] // РќР• СѓРєР°Р·С‹РІР°С‚СЊ RegexOptions.Compiled вЂ” РІ [GeneratedRegex] РёРіРЅРѕСЂРёСЂСѓРµС‚СЃСЏ
    private static partial Regex LogPattern();

    [GeneratedRegex(@"^[\w.+-]+@[\w-]+\.[\w.]+$", RegexOptions.IgnoreCase)]
    private static partial Regex EmailPattern();

    [GeneratedRegex(@"^\+7\d{10}$")]
    private static partial Regex PhonePattern();

    public static bool IsValidEmail(string email) => EmailPattern().IsMatch(email);
    public static bool IsValidPhone(string phone) => PhonePattern().IsMatch(phone);

    public static LogEntry? ParseLine(string line)
    {
        Match match = LogPattern().Match(line);
        if (!match.Success) return null;

        return new LogEntry(
            Date: DateOnly.Parse(match.Groups["date"].Value),
            Level: match.Groups["level"].Value,
            Message: match.Groups["msg"].Value);
    }
}

public record LogEntry(DateOnly Date, string Level, string Message);
```

### RegexOptions

```csharp
// Compiled вЂ” РєРѕРјРїРёР»СЏС†РёСЏ РІ IL РїСЂРё РїРµСЂРІРѕРј РІС‹Р·РѕРІРµ (РјРµРґР»РµРЅРЅС‹Р№ СЃС‚Р°СЂС‚, Р±С‹СЃС‚СЂРѕРµ РІС‹РїРѕР»РЅРµРЅРёРµ)
var compiled = new Regex(@"\d+", RegexOptions.Compiled);

// IgnoreCase вЂ” СЂРµРіРёСЃС‚СЂРѕРЅРµР·Р°РІРёСЃРёРјС‹Р№ РїРѕРёСЃРє
Regex.IsMatch("Hello", @"hello", RegexOptions.IgnoreCase); // true

// Multiline вЂ” ^ Рё $ СЃРѕРІРїР°РґР°СЋС‚ СЃ РЅР°С‡Р°Р»РѕРј/РєРѕРЅС†РѕРј РљРђР–Р”РћР™ СЃС‚СЂРѕРєРё
Regex.Matches("line1\nline2\nline3", @"^line\d", RegexOptions.Multiline); // 3 СЃРѕРІРїР°РґРµРЅРёСЏ

// Singleline вЂ” С‚РѕС‡РєР° (.) СЃРѕРІРїР°РґР°РµС‚ СЃ \n
Regex.IsMatch("hello\nworld", @"hello.world", RegexOptions.Singleline); // true

// РљРѕРјР±РёРЅРёСЂРѕРІР°РЅРёРµ
var regex = new Regex(@"pattern",
    RegexOptions.Compiled | RegexOptions.IgnoreCase | RegexOptions.Multiline);

// Timeout вЂ” Р·Р°С‰РёС‚Р° РѕС‚ ReDoS
var safe = new Regex(@"(a+)+$",
    RegexOptions.None,
    matchTimeout: TimeSpan.FromMilliseconds(100));

try
{
    safe.IsMatch("aaaaaaaaaaaaaaaaaaaaaaaaaaaa!");
}
catch (RegexMatchTimeoutException)
{
    Console.WriteLine("Regex timeout вЂ” РІРѕР·РјРѕР¶РЅР°СЏ ReDoS-Р°С‚Р°РєР°");
}
```

### Р Р°СЃРїСЂРѕСЃС‚СЂР°РЅС‘РЅРЅС‹Рµ РїР°С‚С‚РµСЂРЅС‹

```csharp
public static partial class CommonPatterns
{
    // Email (СѓРїСЂРѕС‰С‘РЅРЅС‹Р№, РґР»СЏ Р±Р°Р·РѕРІРѕР№ РІР°Р»РёРґР°С†РёРё)
    [GeneratedRegex(@"^[\w.+-]+@[\w-]+\.[\w.]+$", RegexOptions.IgnoreCase)]
    public static partial Regex Email();

    // РўРµР»РµС„РѕРЅ Р Р¤: +7XXXXXXXXXX
    [GeneratedRegex(@"^\+7\d{10}$")]
    public static partial Regex PhoneRu();

    // URL
    [GeneratedRegex(@"^https?://[\w\-.]+(:\d+)?(/[\w\-./?%&=]*)?$", RegexOptions.IgnoreCase)]
    public static partial Regex Url();

    // IPv4
    [GeneratedRegex(@"^((25[0-5]|2[0-4]\d|[01]?\d\d?)\.){3}(25[0-5]|2[0-4]\d|[01]?\d\d?)$")]
    public static partial Regex IPv4();

    // GUID
    [GeneratedRegex(@"^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$",
        RegexOptions.IgnoreCase)]
    public static partial Regex Guid();
}
```

---

## РЎРј. С‚Р°РєР¶Рµ

- [РўРёРїС‹ Рё РїР°РјСЏС‚СЊ](types-and-memory.md)
- [Async Рё РїРѕС‚РѕРєРё](async-threading.md)
- [Modern C#](modern-features.md)
- [Collections Рё LINQ](collections-linq.md)
