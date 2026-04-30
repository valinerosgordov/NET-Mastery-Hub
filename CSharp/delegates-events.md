---
tags: [delegates, events, lambdas, func, action]
level: Middle to Senior
---

# Delegates, Events Рё Lambdas

## Р§С‚Рѕ СЌС‚Рѕ, Р·Р°С‡РµРј Рё РєРѕРіРґР°

### Р§С‚Рѕ С‚Р°РєРѕРµ РґРµР»РµРіР°С‚?
**РџРµСЂРµРјРµРЅРЅР°СЏ, РєРѕС‚РѕСЂР°СЏ С…СЂР°РЅРёС‚ РјРµС‚РѕРґ.** РњРѕР¶РЅРѕ РїРµСЂРµРґР°РІР°С‚СЊ РјРµС‚РѕРґС‹ РєР°Рє РїР°СЂР°РјРµС‚СЂС‹, С…СЂР°РЅРёС‚СЊ РІ РєРѕР»Р»РµРєС†РёСЏС…, РІС‹Р·С‹РІР°С‚СЊ РїРѕР·Р¶Рµ.

**РђРЅР°Р»РѕРіРёСЏ:** Р”РµР»РµРіР°С‚ вЂ” СЌС‚Рѕ Р·Р°РїРёСЃРєР° В«РїРѕР·РІРѕРЅРё РњР°С€Рµ РїРѕ РЅРѕРјРµСЂСѓ 555-1234В». РўС‹ РЅРµ Р·РІРѕРЅРёС€СЊ СЃР°Рј вЂ” РїРµСЂРµРґР°С‘С€СЊ Р·Р°РїРёСЃРєСѓ (РґРµР»РµРіР°С‚) РєРѕРјСѓ-С‚Рѕ, Рё РѕРЅ РїРѕР·РІРѕРЅРёС‚ РєРѕРіРґР° РЅСѓР¶РЅРѕ.

### Р§С‚Рѕ С‚Р°РєРѕРµ СЃРѕР±С‹С‚РёРµ (event)?
**РЈРІРµРґРѕРјР»РµРЅРёРµ В«С‡С‚Рѕ-С‚Рѕ РїСЂРѕРёР·РѕС€Р»РѕВ».** РћРґРёРЅ РѕР±СЉРµРєС‚ РєСЂРёС‡РёС‚ В«Р·Р°РєР°Р· СЃРѕР·РґР°РЅ!В», РґСЂСѓРіРёРµ СЃР»СѓС€Р°СЋС‚ Рё СЂРµР°РіРёСЂСѓСЋС‚. Publisher-Subscriber РїР°С‚С‚РµСЂРЅ.

### Р§С‚Рѕ С‚Р°РєРѕРµ Р»СЏРјР±РґР°?
**РђРЅРѕРЅРёРјРЅС‹Р№ РјРµС‚РѕРґ РІ РѕРґРЅСѓ СЃС‚СЂРѕРєСѓ.** Р’РјРµСЃС‚Рѕ `private bool IsAdult(Person p) { return p.Age >= 18; }` в†’ `p => p.Age >= 18`.

### РљРѕРіРґР° С‡С‚Рѕ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ?

| РљРѕРЅСЃС‚СЂСѓРєС†РёСЏ | РљРѕРіРґР° | РџСЂРёРјРµСЂ |
|-------------|-------|--------|
| **Action\<T\>** | РњРµС‚РѕРґ Р±РµР· РІРѕР·РІСЂР°С‚Р° (void) | Callback: `onComplete: () => Log("done")` |
| **Func\<T, TResult\>** | РњРµС‚РѕРґ СЃ РІРѕР·РІСЂР°С‚РѕРј | LINQ: `.Where(x => x.Age > 18)` |
| **Event** | РЈРІРµРґРѕРјР»РµРЅРёРµ РїРѕРґРїРёСЃС‡РёРєРѕРІ (1 в†’ N) | `OrderCreated`, `ButtonClicked` |
| **Р›СЏРјР±РґР°** | РљРѕСЂРѕС‚РєРёР№ inline РјРµС‚РѕРґ | `.Select(x => x.Name)` |
| **Delegate** | РљР°СЃС‚РѕРјРЅР°СЏ СЃРёРіРЅР°С‚СѓСЂР° (СЂРµРґРєРѕ) | РћР±С‹С‡РЅРѕ С…РІР°С‚Р°РµС‚ Action/Func |

---

> РЎРїСЂР°РІРѕС‡РЅРёРє РїРѕ РґРµР»РµРіР°С‚Р°Рј, СЃРѕР±С‹С‚РёСЏРј Рё Р»СЏРјР±РґР°-РІС‹СЂР°Р¶РµРЅРёСЏРј. C# 13 / .NET 9.
> РўРµРѕСЂРёСЏ в†’ РїСЂР°РєС‚РёРєР° в†’ senior-level РєРѕРґ в†’ РІРѕРїСЂРѕСЃС‹ РёРЅС‚РµСЂРІСЊСЋ.

---

## Delegates

### Р§С‚Рѕ С‚Р°РєРѕРµ delegate

Delegate вЂ” **type-safe СѓРєР°Р·Р°С‚РµР»СЊ РЅР° РјРµС‚РѕРґ**. Р­С‚Рѕ СЃСЃС‹Р»РѕС‡РЅС‹Р№ С‚РёРї, РєРѕС‚РѕСЂС‹Р№ С…СЂР°РЅРёС‚ СЃСЃС‹Р»РєСѓ РЅР° РѕРґРёРЅ РёР»Рё РЅРµСЃРєРѕР»СЊРєРѕ РјРµС‚РѕРґРѕРІ СЃ РѕРїСЂРµРґРµР»С‘РЅРЅРѕР№ СЃРёРіРЅР°С‚СѓСЂРѕР№. РљРѕРјРїРёР»СЏС‚РѕСЂ РіРµРЅРµСЂРёСЂСѓРµС‚ РєР»Р°СЃСЃ, РЅР°СЃР»РµРґСѓСЋС‰РёР№ `System.MulticastDelegate`.

```csharp
// РћР±СЉСЏРІР»РµРЅРёРµ delegate вЂ” РѕРїСЂРµРґРµР»СЏРµРј СЃРёРіРЅР°С‚СѓСЂСѓ
delegate int MathOperation(int a, int b);

// РњРµС‚РѕРґС‹, СЃРѕРІРјРµСЃС‚РёРјС‹Рµ СЃ СЃРёРіРЅР°С‚СѓСЂРѕР№
static int Add(int a, int b) => a + b;
static int Multiply(int a, int b) => a * b;

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ
MathOperation op = Add;
int result = op(3, 4); // 7

op = Multiply;
result = op(3, 4);     // 12
```

### РћР±СЉСЏРІР»РµРЅРёРµ delegate

```csharp
// Р‘РµР· РїР°СЂР°РјРµС‚СЂРѕРІ, Р±РµР· РІРѕР·РІСЂР°С‚Р°
delegate void NotifyHandler();

// РЎ РїР°СЂР°РјРµС‚СЂР°РјРё Рё РІРѕР·РІСЂР°С‚РѕРј
delegate TResult Converter<in TInput, out TResult>(TInput input);

// РЎ ref/out РїР°СЂР°РјРµС‚СЂР°РјРё
delegate bool TryParser<T>(string input, out T result);

// Generic delegate СЃ constraints
delegate T Factory<T>() where T : new();
```

РџРѕРґ РєР°РїРѕС‚РѕРј РєРѕРјРїРёР»СЏС‚РѕСЂ РіРµРЅРµСЂРёСЂСѓРµС‚:

```csharp
// РџСЂРёРјРµСЂРЅРѕ С‚Р°РєРѕР№ РєР»Р°СЃСЃ (СѓРїСЂРѕС‰С‘РЅРЅРѕ)
sealed class MathOperation : MulticastDelegate
{
    public int Invoke(int a, int b);
    // Р’РќРРњРђРќРР•: BeginInvoke/EndInvoke РќР• РїРѕРґРґРµСЂР¶РёРІР°СЋС‚СЃСЏ РІ .NET Core / .NET 5+
    // Р’С‹Р·РѕРІ Р±СЂРѕСЃР°РµС‚ PlatformNotSupportedException. РСЃРїРѕР»СЊР·СѓР№ Task/async РІРјРµСЃС‚Рѕ СЌС‚РѕРіРѕ.
    public IAsyncResult BeginInvoke(int a, int b, AsyncCallback callback, object state);
    public int EndInvoke(IAsyncResult result);
}
```

### Multicast delegates (С†РµРїРѕС‡РєР° РІС‹Р·РѕРІРѕРІ)

Delegate РјРѕР¶РµС‚ С…СЂР°РЅРёС‚СЊ **РЅРµСЃРєРѕР»СЊРєРѕ** РјРµС‚РѕРґРѕРІ. РџСЂРё РІС‹Р·РѕРІРµ РѕРЅРё РІС‹РїРѕР»РЅСЏСЋС‚СЃСЏ РїРѕСЃР»РµРґРѕРІР°С‚РµР»СЊРЅРѕ. Р’РѕР·РІСЂР°С‰Р°РµС‚СЃСЏ СЂРµР·СѓР»СЊС‚Р°С‚ **РїРѕСЃР»РµРґРЅРµРіРѕ** РјРµС‚РѕРґР° РІ С†РµРїРѕС‡РєРµ.

```csharp
delegate void Logger(string message);

static void ConsoleLog(string msg) => Console.WriteLine($"[Console] {msg}");
static void FileLog(string msg) => Console.WriteLine($"[File] {msg}");
static void DbLog(string msg) => Console.WriteLine($"[DB] {msg}");

// РљРѕРјР±РёРЅРёСЂРѕРІР°РЅРёРµ
Logger log = ConsoleLog;
log += FileLog;
log += DbLog;

log("РџСЂРёР»РѕР¶РµРЅРёРµ Р·Р°РїСѓС‰РµРЅРѕ");
// [Console] РџСЂРёР»РѕР¶РµРЅРёРµ Р·Р°РїСѓС‰РµРЅРѕ
// [File] РџСЂРёР»РѕР¶РµРЅРёРµ Р·Р°РїСѓС‰РµРЅРѕ
// [DB] РџСЂРёР»РѕР¶РµРЅРёРµ Р·Р°РїСѓС‰РµРЅРѕ

// РЈРґР°Р»РµРЅРёРµ РёР· С†РµРїРѕС‡РєРё
log -= FileLog;
log("РўРѕР»СЊРєРѕ Console Рё DB");
```

**РџРѕР»СѓС‡РµРЅРёРµ СЂРµР·СѓР»СЊС‚Р°С‚РѕРІ РєР°Р¶РґРѕРіРѕ РјРµС‚РѕРґР° РІ multicast delegate:**

```csharp
delegate int Calculator(int x);

static int Double(int x) => x * 2;
static int Square(int x) => x * x;
static int AddTen(int x) => x + 10;

Calculator calc = Double;
calc += Square;
calc += AddTen;

// РћР±С‹С‡РЅС‹Р№ РІС‹Р·РѕРІ вЂ” РІРµСЂРЅС‘С‚ С‚РѕР»СЊРєРѕ СЂРµР·СѓР»СЊС‚Р°С‚ РїРѕСЃР»РµРґРЅРµРіРѕ (AddTen)
int last = calc(5); // 15

// РџРѕР»СѓС‡РёС‚СЊ СЂРµР·СѓР»СЊС‚Р°С‚ РєР°Р¶РґРѕРіРѕ РјРµС‚РѕРґР°
foreach (Calculator d in calc.GetInvocationList().Cast<Calculator>())
{
    int result = d(5);
    Console.WriteLine(result); // 10, 25, 15
}
```

### Р’СЃС‚СЂРѕРµРЅРЅС‹Рµ РґРµР»РµРіР°С‚С‹: Action, Func, Predicate

Р”Р»СЏ 99% СЃР»СѓС‡Р°РµРІ **РЅРµ РЅСѓР¶РЅРѕ** РѕР±СЉСЏРІР»СЏС‚СЊ СЃРІРѕР№ delegate вЂ” РёСЃРїРѕР»СЊР·СѓР№ РІСЃС‚СЂРѕРµРЅРЅС‹Рµ.

```csharp
// Action вЂ” void-РјРµС‚РѕРґ, РґРѕ 16 РїР°СЂР°РјРµС‚СЂРѕРІ
Action greet = () => Console.WriteLine("РџСЂРёРІРµС‚");
Action<string> log = msg => Console.WriteLine(msg);
Action<string, int> repeat = (msg, count) =>
{
    for (int i = 0; i < count; i++)
        Console.WriteLine(msg);
};

// Func вЂ” РјРµС‚РѕРґ СЃ РІРѕР·РІСЂР°С‰Р°РµРјС‹Рј Р·РЅР°С‡РµРЅРёРµРј, РґРѕ 16 РїР°СЂР°РјРµС‚СЂРѕРІ
// РџРѕСЃР»РµРґРЅРёР№ generic-Р°СЂРіСѓРјРµРЅС‚ вЂ” С‚РёРї РІРѕР·РІСЂР°С‚Р°
Func<int> getRandom = () => Random.Shared.Next();
Func<int, int, int> add = (a, b) => a + b;
Func<string, bool> isValid = s => !string.IsNullOrWhiteSpace(s);

// Predicate вЂ” С‡Р°СЃС‚РЅС‹Р№ СЃР»СѓС‡Р°Р№ Func<T, bool>
Predicate<int> isEven = x => x % 2 == 0;

List<int> numbers = [1, 2, 3, 4, 5, 6, 7, 8];
List<int> evens = numbers.FindAll(isEven); // [2, 4, 6, 8]
```

### РљРѕРіРґР° РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ СЃРІРѕР№ delegate vs РІСЃС‚СЂРѕРµРЅРЅС‹Р№

| РЎС†РµРЅР°СЂРёР№ | Р РµС€РµРЅРёРµ |
|---|---|
| РћР±С‹С‡РЅС‹Р№ callback / pipeline | `Func<T>` РёР»Рё `Action<T>` |
| РќСѓР¶РµРЅ `ref`, `out`, `in` РїР°СЂР°РјРµС‚СЂ | РЎРІРѕР№ delegate |
| РќСѓР¶РЅР° РёРјРµРЅРѕРІР°РЅРЅР°СЏ СЃРµРјР°РЅС‚РёРєР° РґР»СЏ API | РЎРІРѕР№ delegate |
| Event handler | `EventHandler<T>` |
| Р‘РѕР»СЊС€Рµ 16 РїР°СЂР°РјРµС‚СЂРѕРІ (РЅРёРєРѕРіРґР°) | РЎРІРѕР№ delegate |

```csharp
// РЎРІРѕР№ delegate РЅСѓР¶РµРЅ РґР»СЏ ref/out
delegate bool TryParseHandler<T>(ReadOnlySpan<char> input, out T result);

TryParseHandler<int> tryParse = int.TryParse;
if (tryParse("42", out int value))
    Console.WriteLine(value); // 42
```

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: Func vs Action vs delegate вЂ” РєРѕРіРґР° СЃРІРѕР№?**
> `Func<T, TResult>` вЂ” РґРµР»РµРіР°С‚ СЃ РІРѕР·РІСЂР°С‚РѕРј. `Action<T>` вЂ” Р±РµР· РІРѕР·РІСЂР°С‚Р°. РџРѕРєСЂС‹РІР°СЋС‚ 95% СЃР»СѓС‡Р°РµРІ.
> РЎРІРѕР№ delegate вЂ” РєРѕРіРґР° РЅСѓР¶РЅС‹ `ref`/`out`/`in` РїР°СЂР°РјРµС‚СЂС‹, >16 РїР°СЂР°РјРµС‚СЂРѕРІ, РёР»Рё СЃРµРјР°РЅС‚РёС‡РµСЃРєРѕРµ РёРјСЏ РІ API.

---

## РђРЅРѕРЅРёРјРЅС‹Рµ РјРµС‚РѕРґС‹

### РЎРёРЅС‚Р°РєСЃРёСЃ delegate { }

РђРЅРѕРЅРёРјРЅС‹Рµ РјРµС‚РѕРґС‹ вЂ” **СѓСЃС‚Р°СЂРµРІС€РёР№** СЃРїРѕСЃРѕР± (C# 2.0). Р›СЏРјР±РґС‹ РїРѕС‡С‚Рё РїРѕР»РЅРѕСЃС‚СЊСЋ РёС… Р·Р°РјРµРЅРёР»Рё, РЅРѕ Р·РЅР°С‚СЊ РїРѕР»РµР·РЅРѕ РґР»СЏ С‡С‚РµРЅРёСЏ legacy-РєРѕРґР°.

```csharp
// РђРЅРѕРЅРёРјРЅС‹Р№ РјРµС‚РѕРґ
Func<int, int, int> add = delegate (int a, int b)
{
    return a + b;
};

// РўРѕ Р¶Рµ СЃР°РјРѕРµ Р»СЏРјР±РґРѕР№ (РїСЂРµРґРїРѕС‡С‚РёС‚РµР»СЊРЅРѕ)
Func<int, int, int> addLambda = (a, b) => a + b;

// РђРЅРѕРЅРёРјРЅС‹Р№ РјРµС‚РѕРґ Р±РµР· РїР°СЂР°РјРµС‚СЂРѕРІ вЂ” РјРѕР¶РµС‚ РёРіРЅРѕСЂРёСЂРѕРІР°С‚СЊ РїР°СЂР°РјРµС‚СЂС‹ delegate
Action<string> ignore = delegate { Console.WriteLine("РџР°СЂР°РјРµС‚СЂ РїСЂРѕРёРіРЅРѕСЂРёСЂРѕРІР°РЅ"); };
// Р›СЏРјР±РґР° С‚Р°Рє РЅРµ РјРѕР¶РµС‚ вЂ” РЅСѓР¶РЅРѕ СЏРІРЅРѕ СѓРєР°Р·Р°С‚СЊ РїР°СЂР°РјРµС‚СЂ РёР»Рё discard
Action<string> ignoreLambda = _ => Console.WriteLine("Discard");
```

### Р—Р°РјС‹РєР°РЅРёСЏ (closures) вЂ” Р·Р°С…РІР°С‚ РїРµСЂРµРјРµРЅРЅС‹С…

Р—Р°РјС‹РєР°РЅРёРµ вЂ” Р»СЏРјР±РґР° РёР»Рё Р°РЅРѕРЅРёРјРЅС‹Р№ РјРµС‚РѕРґ, РєРѕС‚РѕСЂС‹Р№ **Р·Р°С…РІР°С‚С‹РІР°РµС‚ РїРµСЂРµРјРµРЅРЅС‹Рµ** РёР· РІРЅРµС€РЅРµР№ РѕР±Р»Р°СЃС‚Рё РІРёРґРёРјРѕСЃС‚Рё. РљРѕРјРїРёР»СЏС‚РѕСЂ РіРµРЅРµСЂРёСЂСѓРµС‚ СЃРєСЂС‹С‚С‹Р№ РєР»Р°СЃСЃ РґР»СЏ С…СЂР°РЅРµРЅРёСЏ Р·Р°С…РІР°С‡РµРЅРЅС‹С… РїРµСЂРµРјРµРЅРЅС‹С….

```csharp
int multiplier = 3;

Func<int, int> multiply = x => x * multiplier;

Console.WriteLine(multiply(5)); // 15

// Р—Р°РјС‹РєР°РЅРёРµ Р·Р°С…РІР°С‚С‹РІР°РµС‚ РџР•Р Р•РњР•РќРќРЈР®, РЅРµ Р·РЅР°С‡РµРЅРёРµ!
multiplier = 10;
Console.WriteLine(multiply(5)); // 50 вЂ” Р·РЅР°С‡РµРЅРёРµ РёР·РјРµРЅРёР»РѕСЃСЊ
```

**Р§С‚Рѕ РіРµРЅРµСЂРёСЂСѓРµС‚ РєРѕРјРїРёР»СЏС‚РѕСЂ (РїСЂРёР±Р»РёР·РёС‚РµР»СЊРЅРѕ):**

```csharp
// РљРѕРјРїРёР»СЏС‚РѕСЂ СЃРѕР·РґР°С‘С‚ DisplayClass
[CompilerGenerated]
sealed class <>c__DisplayClass
{
    public int multiplier;

    public int Method(int x) => x * multiplier;
}
```

### РћРїР°СЃРЅРѕСЃС‚Рё Р·Р°РјС‹РєР°РЅРёР№ (captured variable РІ С†РёРєР»Рµ)

РљР»Р°СЃСЃРёС‡РµСЃРєР°СЏ Р»РѕРІСѓС€РєР° вЂ” Р·Р°С…РІР°С‚ РїРµСЂРµРјРµРЅРЅРѕР№ С†РёРєР»Р°:

```csharp
// РћРЁРР‘РљРђ: РІСЃРµ Р»СЏРјР±РґС‹ Р·Р°С…РІР°С‚С‹РІР°СЋС‚ РћР”РќРЈ РїРµСЂРµРјРµРЅРЅСѓСЋ i
var actions = new List<Action>();
for (int i = 0; i < 5; i++)
{
    actions.Add(() => Console.WriteLine(i));
}
foreach (var action in actions)
    action(); // 5, 5, 5, 5, 5 вЂ” РІСЃРµ РїРµС‡Р°С‚Р°СЋС‚ 5!

// РРЎРџР РђР’Р›Р•РќРР•: Р»РѕРєР°Р»СЊРЅР°СЏ РєРѕРїРёСЏ
var actionsFixed = new List<Action>();
for (int i = 0; i < 5; i++)
{
    int local = i; // РєРѕРїРёСЏ РґР»СЏ РєР°Р¶РґРѕР№ РёС‚РµСЂР°С†РёРё
    actionsFixed.Add(() => Console.WriteLine(local));
}
foreach (var action in actionsFixed)
    action(); // 0, 1, 2, 3, 4

// foreach РќР• РёРјРµРµС‚ СЌС‚РѕР№ РїСЂРѕР±Р»РµРјС‹ (РЅР°С‡РёРЅР°СЏ СЃ C# 5)
var items = new[] { "a", "b", "c" };
var prints = new List<Action>();
foreach (var item in items)
{
    prints.Add(() => Console.WriteLine(item)); // OK вЂ” РєР°Р¶РґР°СЏ РёС‚РµСЂР°С†РёСЏ СЃРІРѕР№ item
}
```

**Р•С‰С‘ РѕРґРЅР° РѕРїР°СЃРЅРѕСЃС‚СЊ вЂ” Р·Р°С…РІР°С‚ С‚СЏР¶С‘Р»С‹С… РѕР±СЉРµРєС‚РѕРІ:**

```csharp
// РџР›РћРҐРћ: Р·Р°РјС‹РєР°РЅРёРµ РґРµСЂР¶РёС‚ СЃСЃС‹Р»РєСѓ РЅР° РІРµСЃСЊ РјР°СЃСЃРёРІ, РјРµС€Р°РµС‚ GC
byte[] hugeBuffer = new byte[10_000_000];

Func<int> getLength = () => hugeBuffer.Length;

// hugeBuffer РЅРµ Р±СѓРґРµС‚ СЃРѕР±СЂР°РЅ GC, РїРѕРєР° Р¶РёРІ getLength
```

---

## Lambda-РІС‹СЂР°Р¶РµРЅРёСЏ

### Expression lambda vs Statement lambda

```csharp
// Expression lambda вЂ” РѕРґРЅРѕ РІС‹СЂР°Р¶РµРЅРёРµ, return РЅРµСЏРІРЅС‹Р№
Func<int, int> square = x => x * x;
Func<string, string> toUpper = s => s.ToUpperInvariant();
Func<int, int, bool> isGreater = (a, b) => a > b;

// Statement lambda вЂ” Р±Р»РѕРє РєРѕРґР°, return СЏРІРЅС‹Р№
Func<int, string> classify = x =>
{
    if (x < 0) return "negative";
    if (x == 0) return "zero";
    return "positive";
};

// Expression lambda РјРѕР¶РЅРѕ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ РєР°Рє Expression Tree
Expression<Func<int, bool>> expr = x => x > 5;
// РљРѕРјРїРёР»РёСЂСѓРµС‚СЃСЏ РІ РґРµСЂРµРІРѕ РІС‹СЂР°Р¶РµРЅРёР№, Р° РЅРµ РІ delegate
// РСЃРїРѕР»СЊР·СѓРµС‚СЃСЏ РІ EF Core, OData Рё С‚.Рґ.
```

### Natural type inference (C# 10+)

РљРѕРјРїРёР»СЏС‚РѕСЂ РјРѕР¶РµС‚ РІС‹РІРµСЃС‚Рё С‚РёРї delegate РёР· Р»СЏРјР±РґС‹ Р±РµР· СЏРІРЅРѕРіРѕ СѓРєР°Р·Р°РЅРёСЏ:

```csharp
// C# 10+: natural type вЂ” РєРѕРјРїРёР»СЏС‚РѕСЂ РІС‹РІРѕРґРёС‚ Func<int, int>
var square = (int x) => x * x;

// РўРёРї РїР°СЂР°РјРµС‚СЂРѕРІ РЅСѓР¶РЅРѕ СѓРєР°Р·Р°С‚СЊ РґР»СЏ var
var greet = (string name) => $"РџСЂРёРІРµС‚, {name}!";

// Р”Р»СЏ void-Р»СЏРјР±Рґ РІС‹РІРѕРґРёС‚СЃСЏ Action
var log = (string msg) => Console.WriteLine(msg);

// РњРѕР¶РЅРѕ СѓРєР°Р·Р°С‚СЊ return type СЏРІРЅРѕ
var parse = int (string s) => int.Parse(s);

// Р Р°Р±РѕС‚Р°РµС‚ СЃ method groups С‚РѕР¶Рµ
var writeLine = Console.WriteLine; // С‚РёРї РЅРµ РІС‹РІРµРґРµС‚СЃСЏ вЂ” РїРµСЂРµРіСЂСѓР·РєРё
// РќСѓР¶РЅРѕ СЏРІРЅРѕ:
Action<string> write = Console.WriteLine; // OK
```

### Lambda attributes (C# 10+)

```csharp
// РђС‚СЂРёР±СѓС‚С‹ РЅР° Р»СЏРјР±РґРµ вЂ” РїРѕР»РµР·РЅРѕ РґР»СЏ Minimal APIs
var handler = [Authorize] (HttpContext ctx) => Results.Ok("РЎРµРєСЂРµС‚РЅС‹Рµ РґР°РЅРЅС‹Рµ");

// РђС‚СЂРёР±СѓС‚С‹ РЅР° РїР°СЂР°РјРµС‚СЂР°С…
var endpoint = ([FromQuery] string name, [FromServices] ILogger logger) =>
{
    logger.LogInformation("Р—Р°РїСЂРѕСЃ РѕС‚ {Name}", name);
    return Results.Ok($"РџСЂРёРІРµС‚, {name}");
};

// РџСЂРёРјРµСЂ РІ Minimal API
app.MapGet("/api/users", [Authorize(Roles = "Admin")]
    async ([FromServices] IMediator mediator) =>
    {
        var result = await mediator.Send(new GetUsersQuery());
        return result.IsSuccess ? Results.Ok(result.Value) : Results.Problem();
    });
```

### Lambda parameter modifiers: ref, in, out

```csharp
// ref/out РІ Р»СЏРјР±РґР°С… вЂ” C# 7.3+, in вЂ” C# 9+
// ref вЂ” С‡С‚РµРЅРёРµ Рё Р·Р°РїРёСЃСЊ РїРѕ СЃСЃС‹Р»РєРµ
var increment = (ref int x) => x++;

int value = 10;
increment(ref value);
Console.WriteLine(value); // 11

// in вЂ” С‚РѕР»СЊРєРѕ С‡С‚РµРЅРёРµ РїРѕ СЃСЃС‹Р»РєРµ (Р±РµР· РєРѕРїРёСЂРѕРІР°РЅРёСЏ)
var printLength = (in ReadOnlySpan<char> span) => Console.WriteLine(span.Length);

// out вЂ” Р·Р°РїРёСЃСЊ Р·РЅР°С‡РµРЅРёСЏ С‡РµСЂРµР· СЃСЃС‹Р»РєСѓ
var tryDivide = (int a, int b, out int result) =>
{
    if (b == 0) { result = 0; return false; }
    result = a / b;
    return true;
};

if (tryDivide(10, 3, out int res))
    Console.WriteLine(res); // 3
```

### Default parameter values РІ Р»СЏРјР±РґР°С… (C# 12)

```csharp
// C# 12: Р·РЅР°С‡РµРЅРёСЏ РїРѕ СѓРјРѕР»С‡Р°РЅРёСЋ РІ РїР°СЂР°РјРµС‚СЂР°С… Р»СЏРјР±Рґ
var greet = (string name, string greeting = "РџСЂРёРІРµС‚") => $"{greeting}, {name}!";

Console.WriteLine(greet("РњРёСЂ"));           // РџСЂРёРІРµС‚, РњРёСЂ!
Console.WriteLine(greet("РњРёСЂ", "Р—РґСЂР°РІСЃС‚РІСѓР№")); // Р—РґСЂР°РІСЃС‚РІСѓР№, РњРёСЂ!

// РџРѕР»РµР·РЅРѕ РІ Minimal APIs
app.MapGet("/search", (string query, int page = 1, int pageSize = 20) =>
{
    // page Рё pageSize РёРјРµСЋС‚ Р·РЅР°С‡РµРЅРёСЏ РїРѕ СѓРјРѕР»С‡Р°РЅРёСЋ
    return Results.Ok(new { query, page, pageSize });
});

// params С‚РѕР¶Рµ СЂР°Р±РѕС‚Р°РµС‚ (C# 13)
var sum = (params int[] numbers) => numbers.Sum();
Console.WriteLine(sum(1, 2, 3, 4, 5)); // 15
```

### Static lambdas вЂ” РёР·Р±РµР¶Р°РЅРёРµ Р°Р»Р»РѕРєР°С†РёР№

Static lambda **Р·Р°РїСЂРµС‰Р°РµС‚** Р·Р°С…РІР°С‚ РїРµСЂРµРјРµРЅРЅС‹С… РёР· РІРЅРµС€РЅРµР№ РѕР±Р»Р°СЃС‚Рё. Р­С‚Рѕ РіР°СЂР°РЅС‚РёСЂСѓРµС‚ РѕС‚СЃСѓС‚СЃС‚РІРёРµ Р°Р»Р»РѕРєР°С†РёРё closure-РѕР±СЉРµРєС‚Р°.

```csharp
int factor = 2;

// РћР±С‹С‡РЅР°СЏ Р»СЏРјР±РґР° вЂ” Р·Р°С…РІР°С‚С‹РІР°РµС‚ factor, СЃРѕР·РґР°С‘С‚ closure (Р°Р»Р»РѕРєР°С†РёСЏ)
Func<int, int> withClosure = x => x * factor;

// Static lambda вЂ” Р·Р°РїСЂРµС‰Р°РµС‚ Р·Р°С…РІР°С‚
Func<int, int> noCapture = static x => x * 2; // OK вЂ” Р»РёС‚РµСЂР°Р»

// РћРЁРР‘РљРђ РљРћРњРџРР›РЇР¦РР: static lambda РЅРµ РјРѕР¶РµС‚ Р·Р°С…РІР°С‚С‹РІР°С‚СЊ РїРµСЂРµРјРµРЅРЅС‹Рµ
// Func<int, int> error = static x => x * factor;

// РџРѕР»РµР·РЅРѕ РІ hot path РґР»СЏ РёР·Р±РµР¶Р°РЅРёСЏ Р°Р»Р»РѕРєР°С†РёР№
var numbers = new[] { 1, 2, 3, 4, 5 };

// Р‘РµР· Р°Р»Р»РѕРєР°С†РёРё closure
int[] doubled = Array.ConvertAll(numbers, static x => x * 2);

// Static local function вЂ” Р°РЅР°Р»РѕРіРёС‡РЅР°СЏ РѕРїС‚РёРјРёР·Р°С†РёСЏ
static int Square(int x) => x * x;
int[] squares = numbers.Select(Square).ToArray();
```

**РЎСЂР°РІРЅРµРЅРёРµ Р°Р»Р»РѕРєР°С†РёР№:**

```csharp
// Hot path вЂ” РєР°Р¶РґС‹Р№ РІС‹Р·РѕРІ СЃРѕР·РґР°С‘С‚ closure
void ProcessBad(int threshold)
{
    _items.Where(x => x.Value > threshold).ToList(); // Р°Р»Р»РѕРєР°С†РёСЏ DisplayClass
}

// РћРїС‚РёРјРёР·Р°С†РёСЏ вЂ” РїРµСЂРµРґР°С‘Рј threshold СЏРІРЅРѕ РёР»Рё РёСЃРїРѕР»СЊР·СѓРµРј РґСЂСѓРіРѕР№ РїРѕРґС…РѕРґ
void ProcessGood(int threshold)
{
    // Р’Р°СЂРёР°РЅС‚: С†РёРєР» РІРјРµСЃС‚Рѕ LINQ РІ hot path
    foreach (var item in _items)
    {
        if (item.Value > threshold)
            ProcessItem(item);
    }
}
```

---

## Events

### event keyword вЂ” Р·Р°С‡РµРј РЅСѓР¶РµРЅ vs РѕР±С‹С‡РЅС‹Р№ delegate

РљР»СЋС‡РµРІРѕРµ СЃР»РѕРІРѕ `event` РґРѕР±Р°РІР»СЏРµС‚ **РёРЅРєР°РїСЃСѓР»СЏС†РёСЋ** РїРѕРІРµСЂС… delegate. РЎРЅР°СЂСѓР¶Рё РєР»Р°СЃСЃР° РјРѕР¶РЅРѕ С‚РѕР»СЊРєРѕ `+=` Рё `-=`, РЅРѕ РЅРµР»СЊР·СЏ РІС‹Р·РІР°С‚СЊ РёР»Рё РїСЂРёСЃРІРѕРёС‚СЊ.

```csharp
// Р‘Р•Р— event вЂ” delegate РєР°Рє public РїРѕР»Рµ (РћРџРђРЎРќРћ)
class ButtonBad
{
    public Action? Clicked; // Р»СЋР±РѕР№ РјРѕР¶РµС‚ РІС‹Р·РІР°С‚СЊ Рё РїРµСЂРµР·Р°РїРёСЃР°С‚СЊ
}

var badButton = new ButtonBad();
badButton.Clicked = null;      // Р—РђРўРЃР  РІСЃРµ РїРѕРґРїРёСЃРєРё вЂ” РѕРїР°СЃРЅРѕ!
badButton.Clicked();           // РњРѕР¶РµС‚ РІС‹Р·РІР°С‚СЊ РёР·РІРЅРµ вЂ” РѕРїР°СЃРЅРѕ!

// РЎ event вЂ” Р±РµР·РѕРїР°СЃРЅР°СЏ РёРЅРєР°РїСЃСѓР»СЏС†РёСЏ
class Button
{
    public event Action? Clicked;

    public void SimulateClick() => Clicked?.Invoke();
}

var button = new Button();
button.Clicked += () => Console.WriteLine("РќР°Р¶Р°С‚Р°!");
// button.Clicked = null;      // РћРЁРР‘РљРђ РљРћРњРџРР›РЇР¦РР
// button.Clicked();           // РћРЁРР‘РљРђ РљРћРњРџРР›РЇР¦РР
// button.Clicked?.Invoke();   // РћРЁРР‘РљРђ РљРћРњРџРР›РЇР¦РР
button.SimulateClick();        // OK вЂ” С‚РѕР»СЊРєРѕ РёР·РЅСѓС‚СЂРё РєР»Р°СЃСЃР°
```

### EventHandler Рё EventHandler\<TEventArgs\>

РЎС‚Р°РЅРґР°СЂС‚РЅС‹Р№ РїР°С‚С‚РµСЂРЅ СЃРѕР±С‹С‚РёР№ РІ .NET:

```csharp
// РЎС‚Р°РЅРґР°СЂС‚РЅР°СЏ СЃРёРіРЅР°С‚СѓСЂР°: (object? sender, EventArgs e)
class FileWatcher
{
    // Р‘РµР· РґР°РЅРЅС‹С…
    public event EventHandler? FileDetected;

    // РЎ РґР°РЅРЅС‹РјРё
    public event EventHandler<FileChangedEventArgs>? FileChanged;

    public void Scan(string path)
    {
        // РќР°С€Р»Рё С„Р°Р№Р»
        FileDetected?.Invoke(this, EventArgs.Empty);

        // Р¤Р°Р№Р» РёР·РјРµРЅРёР»СЃСЏ
        FileChanged?.Invoke(this, new FileChangedEventArgs(path, DateTime.UtcNow));
    }
}

// Custom EventArgs
sealed class FileChangedEventArgs(string filePath, DateTime changedAt) : EventArgs
{
    public string FilePath { get; } = filePath;
    public DateTime ChangedAt { get; } = changedAt;
}

// РџРѕРґРїРёСЃРєР°
var watcher = new FileWatcher();
watcher.FileDetected += (sender, e) => Console.WriteLine("Р¤Р°Р№Р» РѕР±РЅР°СЂСѓР¶РµРЅ");
watcher.FileChanged += (sender, e) =>
    Console.WriteLine($"РР·РјРµРЅС‘РЅ: {e.FilePath} РІ {e.ChangedAt:HH:mm:ss}");
```

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: Event vs delegate вЂ” Р·Р°С‡РµРј РєР»СЋС‡РµРІРѕРµ СЃР»РѕРІРѕ event?**
> `event` РѕРіСЂР°РЅРёС‡РёРІР°РµС‚ РґРѕСЃС‚СѓРї: РїРѕРґРїРёСЃРєР°/РѕС‚РїРёСЃРєР° (`+=`/`-=`) вЂ” РёР·РІРЅРµ. Р’С‹Р·РѕРІ (`Invoke`) вЂ” С‚РѕР»СЊРєРѕ РёР·РЅСѓС‚СЂРё РєР»Р°СЃСЃР°. Р‘РµР· `event` Р»СЋР±РѕР№ РєРѕРґ РјРѕР¶РµС‚ РїРµСЂРµР·Р°РїРёСЃР°С‚СЊ РґРµР»РµРіР°С‚ (`handler = null`).
>
> **РЈС‚РµС‡РєРё РїР°РјСЏС‚Рё:** РїРѕРґРїРёСЃС‡РёРє СѓРґРµСЂР¶РёРІР°РµС‚ СЃСЃС‹Р»РєСѓ РЅР° РёР·РґР°С‚РµР»СЏ. Р•СЃР»Рё РїРѕРґРїРёСЃС‡РёРє Р¶РёРІС‘С‚ РґРѕР»СЊС€Рµ вЂ” СѓС‚РµС‡РєР°. Р РµС€РµРЅРёРµ: РѕС‚РїРёСЃРєР° РІ `Dispose`, `WeakEventManager`, `static` handlers.

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: ref vs out vs in вЂ” СЂР°Р·Р»РёС‡РёСЏ?**
> | РњРѕРґРёС„РёРєР°С‚РѕСЂ | Р§С‚РµРЅРёРµ | Р—Р°РїРёСЃСЊ | РќР°Р·РЅР°С‡РµРЅРёРµ |
> |-------------|--------|--------|------------|
> | `ref` | Р”Р° | Р”Р° | Р”РІСѓСЃС‚РѕСЂРѕРЅРЅСЏСЏ РїРµСЂРµРґР°С‡Р° |
> | `out` | Р”Рѕ РїСЂРёСЃРІРѕРµРЅРёСЏ вЂ” РЅРµС‚ | РћР±СЏР·Р°С‚РµР»СЊРЅР° | Р’РѕР·РІСЂР°С‚ РЅРµСЃРєРѕР»СЊРєРёС… Р·РЅР°С‡РµРЅРёР№ |
> | `in` | Р”Р° | РќРµС‚ | Р‘РѕР»СЊС€РёРµ struct Р±РµР· РєРѕРїРёСЂРѕРІР°РЅРёСЏ |
>
> `in` вЂ” readonly ref. РџРѕР»РµР·РЅРѕ РґР»СЏ struct > 16 Р±Р°Р№С‚ РІ hot path.

### Publish/Subscribe РїР°С‚С‚РµСЂРЅ

```csharp
// Publisher вЂ” РЅРёС‡РµРіРѕ РЅРµ Р·РЅР°РµС‚ Рѕ РїРѕРґРїРёСЃС‡РёРєР°С…
sealed class OrderService
{
    public event EventHandler<OrderCreatedEventArgs>? OrderCreated;
    public event EventHandler<OrderCreatedEventArgs>? OrderShipped;

    public void CreateOrder(string product, decimal price)
    {
        // Р‘РёР·РЅРµСЃ-Р»РѕРіРёРєР° СЃРѕР·РґР°РЅРёСЏ Р·Р°РєР°Р·Р°...
        var args = new OrderCreatedEventArgs(product, price);
        OrderCreated?.Invoke(this, args);
    }

    public void ShipOrder(string product, decimal price)
    {
        var args = new OrderCreatedEventArgs(product, price);
        OrderShipped?.Invoke(this, args);
    }
}

sealed class OrderCreatedEventArgs(string product, decimal price) : EventArgs
{
    public string Product { get; } = product;
    public decimal Price { get; } = price;
}

// Subscribers вЂ” РїРѕРґРїРёСЃС‹РІР°СЋС‚СЃСЏ РЅРµР·Р°РІРёСЃРёРјРѕ
sealed class EmailNotifier
{
    public void Subscribe(OrderService service)
    {
        service.OrderCreated += OnOrderCreated;
        service.OrderShipped += OnOrderShipped;
    }

    private void OnOrderCreated(object? sender, OrderCreatedEventArgs e) =>
        Console.WriteLine($"[Email] РќРѕРІС‹Р№ Р·Р°РєР°Р·: {e.Product} Р·Р° {e.Price:C}");

    private void OnOrderShipped(object? sender, OrderCreatedEventArgs e) =>
        Console.WriteLine($"[Email] Р—Р°РєР°Р· РѕС‚РїСЂР°РІР»РµРЅ: {e.Product}");
}

sealed class AnalyticsTracker
{
    public void Subscribe(OrderService service)
    {
        service.OrderCreated += (_, e) =>
            Console.WriteLine($"[Analytics] РџСЂРѕРґР°Р¶Р°: {e.Price:C}");
    }
}

// РЎРІСЏР·С‹РІР°РµРј
var orderService = new OrderService();
var emailNotifier = new EmailNotifier();
var analytics = new AnalyticsTracker();

emailNotifier.Subscribe(orderService);
analytics.Subscribe(orderService);

orderService.CreateOrder("РќРѕСѓС‚Р±СѓРє", 85_000m);
// [Email] РќРѕРІС‹Р№ Р·Р°РєР°Р·: РќРѕСѓС‚Р±СѓРє Р·Р° 85 000,00 в‚Ѕ
// [Analytics] РџСЂРѕРґР°Р¶Р°: 85 000,00 в‚Ѕ
```

### Custom EventArgs

```csharp
// Record-based EventArgs (C# 9+ вЂ” С‡РёСЃС‚Рѕ Рё РєСЂР°С‚РєРѕ)
sealed record ProgressEventArgs(int Current, int Total, string? Message = null) : EventArgs
{
    public double Percentage => Total > 0 ? (double)Current / Total * 100 : 0;
}

sealed class DataImporter
{
    public event EventHandler<ProgressEventArgs>? ProgressChanged;
    public event EventHandler<ImportCompletedEventArgs>? Completed;

    public async Task ImportAsync(IReadOnlyList<string> files, CancellationToken ct = default)
    {
        for (int i = 0; i < files.Count; i++)
        {
            ct.ThrowIfCancellationRequested();
            await ProcessFileAsync(files[i]);

            ProgressChanged?.Invoke(this, new(i + 1, files.Count, files[i]));
        }

        Completed?.Invoke(this, new(files.Count, true));
    }

    private Task ProcessFileAsync(string file) => Task.Delay(100); // РёРјРёС‚Р°С†РёСЏ
}

sealed record ImportCompletedEventArgs(int TotalFiles, bool Success) : EventArgs;
```

### Event accessors: add/remove

РђРЅР°Р»РѕРі get/set РґР»СЏ СЃРІРѕР№СЃС‚РІ, РЅРѕ РґР»СЏ СЃРѕР±С‹С‚РёР№. РџРѕР·РІРѕР»СЏСЋС‚ РєРѕРЅС‚СЂРѕР»РёСЂРѕРІР°С‚СЊ РїРѕРґРїРёСЃРєСѓ/РѕС‚РїРёСЃРєСѓ.

```csharp
sealed class ThreadSafePublisher
{
    private readonly object _lock = new();
    private EventHandler? _clicked;

    public event EventHandler Clicked
    {
        add
        {
            lock (_lock)
            {
                _clicked += value;
                Console.WriteLine($"РџРѕРґРїРёСЃС‡РёРє РґРѕР±Р°РІР»РµРЅ. Р’СЃРµРіРѕ: {_clicked?.GetInvocationList().Length ?? 0}");
            }
        }
        remove
        {
            lock (_lock)
            {
                _clicked -= value;
                Console.WriteLine("РџРѕРґРїРёСЃС‡РёРє СѓРґР°Р»С‘РЅ");
            }
        }
    }

    public void RaiseClick()
    {
        EventHandler? handler;
        lock (_lock)
        {
            handler = _clicked;
        }
        handler?.Invoke(this, EventArgs.Empty);
    }
}
```

### Weak events вЂ” РїСЂРµРґРѕС‚РІСЂР°С‰РµРЅРёРµ СѓС‚РµС‡РµРє РїР°РјСЏС‚Рё

РћР±С‹С‡РЅС‹Р№ event РґРµСЂР¶РёС‚ **strong reference** РЅР° РїРѕРґРїРёСЃС‡РёРєР°, РјРµС€Р°СЏ GC РµРіРѕ СЃРѕР±СЂР°С‚СЊ. Р­С‚Рѕ С‚РёРїРёС‡РЅР°СЏ РїСЂРёС‡РёРЅР° СѓС‚РµС‡РµРє РїР°РјСЏС‚Рё, РѕСЃРѕР±РµРЅРЅРѕ РІ WPF.

```csharp
// РџР РћР‘Р›Р•РњРђ: СѓС‚РµС‡РєР° РїР°РјСЏС‚Рё
class LongLivedPublisher
{
    public event EventHandler? DataReady; // Р”РµСЂР¶РёС‚ strong ref РЅР° РїРѕРґРїРёСЃС‡РёРєРѕРІ
}

class ShortLivedSubscriber
{
    public ShortLivedSubscriber(LongLivedPublisher publisher)
    {
        publisher.DataReady += OnDataReady; // РџРѕРґРїРёСЃР°Р»РёСЃСЊ вЂ” С‚РµРїРµСЂСЊ GC РЅРµ СЃРѕР±РµСЂС‘С‚
    }

    private void OnDataReady(object? sender, EventArgs e) { }

    // Р”Р°Р¶Рµ РµСЃР»Рё ShortLivedSubscriber Р±РѕР»СЊС€Рµ РЅРµ РёСЃРїРѕР»СЊР·СѓРµС‚СЃСЏ,
    // publisher РґРµСЂР¶РёС‚ РЅР° РЅРµРіРѕ СЃСЃС‹Р»РєСѓ С‡РµСЂРµР· event
}

// Р Р•РЁР•РќРР• 1: IDisposable + СЏРІРЅР°СЏ РѕС‚РїРёСЃРєР°
sealed class SafeSubscriber(LongLivedPublisher publisher) : IDisposable
{
    private bool _disposed;

    public void Subscribe() => publisher.DataReady += OnDataReady;

    private void OnDataReady(object? sender, EventArgs e)
    {
        if (_disposed) return;
        Console.WriteLine("РћР±СЂР°Р±РѕС‚РєР° РґР°РЅРЅС‹С…");
    }

    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        publisher.DataReady -= OnDataReady;
    }
}

// Р Р•РЁР•РќРР• 2: WeakEventManager (WPF)
// using System.Windows;
// WeakEventManager<LongLivedPublisher, EventArgs>
//     .AddHandler(publisher, nameof(LongLivedPublisher.DataReady), OnDataReady);

// Р Р•РЁР•РќРР• 3: РЎРѕР±СЃС‚РІРµРЅРЅР°СЏ СЂРµР°Р»РёР·Р°С†РёСЏ С‡РµСЂРµР· WeakReference
sealed class WeakEvent<TEventArgs> where TEventArgs : EventArgs
{
    private readonly List<WeakReference<EventHandler<TEventArgs>>> _handlers = [];

    public void Subscribe(EventHandler<TEventArgs> handler)
    {
        _handlers.Add(new WeakReference<EventHandler<TEventArgs>>(handler));
    }

    public void Raise(object? sender, TEventArgs args)
    {
        for (int i = _handlers.Count - 1; i >= 0; i--)
        {
            if (_handlers[i].TryGetTarget(out var handler))
            {
                handler(sender, args);
            }
            else
            {
                _handlers.RemoveAt(i); // РџРѕРґРїРёСЃС‡РёРє СЃРѕР±СЂР°РЅ GC вЂ” СѓР±РёСЂР°РµРј
            }
        }
    }
}
```

### Р›СѓС‡С€РёРµ РїСЂР°РєС‚РёРєРё: РїСЂРѕРІРµСЂРєР° null, Volatile.Read

```csharp
sealed class SafeEventPublisher
{
    public event EventHandler<MessageEventArgs>? MessageReceived;

    // РЎРїРѕСЃРѕР± 1: Null-conditional operator (СЂРµРєРѕРјРµРЅРґСѓРµРјС‹Р№ СЃ C# 6+)
    // Thread-safe вЂ” РєРѕРјРїРёР»СЏС‚РѕСЂ РєРѕРїРёСЂСѓРµС‚ delegate РїРµСЂРµРґ РІС‹Р·РѕРІРѕРј
    public void OnMessage(string text) =>
        MessageReceived?.Invoke(this, new MessageEventArgs(text));

    // РЎРїРѕСЃРѕР± 2: РЇРІРЅР°СЏ РєРѕРїРёСЏ (РєР»Р°СЃСЃРёС‡РµСЃРєРёР№ РїРѕРґС…РѕРґ)
    public void OnMessageClassic(string text)
    {
        EventHandler<MessageEventArgs>? handler = MessageReceived;
        handler?.Invoke(this, new MessageEventArgs(text));
    }

    // РЎРїРѕСЃРѕР± 3: Volatile.Read вЂ” РјР°РєСЃРёРјР°Р»СЊРЅР°СЏ РєРѕСЂСЂРµРєС‚РЅРѕСЃС‚СЊ РґР»СЏ multi-threaded
    // РўСЂРµР±СѓРµС‚ РѕС‚РґРµР»СЊРЅРѕРіРѕ backing field (РЅРµР»СЊР·СЏ ref РЅР° event РЅР°РїСЂСЏРјСѓСЋ)
    // private EventHandler<MessageEventArgs>? _messageReceived;
    // public event EventHandler<MessageEventArgs>? MessageReceived
    // {
    //     add => _messageReceived += value;
    //     remove => _messageReceived -= value;
    // }
    // public void OnMessageVolatile(string text)
    // {
    //     var handler = Volatile.Read(ref _messageReceived);
    //     handler?.Invoke(this, new MessageEventArgs(text));
    // }
    //
    // РЈРїСЂРѕС‰С‘РЅРЅС‹Р№ РІР°СЂРёР°РЅС‚ (field-like event вЂ” РєРѕРјРїРёР»СЏС‚РѕСЂ РіРµРЅРµСЂРёСЂСѓРµС‚ backing field):
    public void OnMessageVolatile(string text)
    {
        // Р”Р»СЏ field-like event РєРѕРјРїРёР»СЏС‚РѕСЂ Р°РІС‚РѕРјР°С‚РёС‡РµСЃРєРё РґРµР»Р°РµС‚ thread-safe РґРѕСЃС‚СѓРї
        // Р”РѕСЃС‚Р°С‚РѕС‡РЅРѕ РЎРїРѕСЃРѕР±Р° 2 (handler?.Invoke) вЂ” РѕРЅ СѓР¶Рµ thread-safe
        var handler = MessageReceived;
        handler?.Invoke(this, new MessageEventArgs(text));
    }

    // РќР•РџР РђР’РР›Р¬РќРћ: race condition
    // public void OnMessageBad(string text)
    // {
    //     if (MessageReceived != null)         // РњРѕР¶РµС‚ СЃС‚Р°С‚СЊ null С‚СѓС‚
    //         MessageReceived(this, new(...)); // NullReferenceException!
    // }
}

sealed record MessageEventArgs(string Text) : EventArgs;
```

**РџСЂР°РІРёР»Р° РґР»СЏ СЃРѕР±С‹С‚РёР№:**

```csharp
// 1. РњРµС‚РѕРґ-РѕР±С‘СЂС‚РєР° OnXxx вЂ” protected virtual РґР»СЏ РЅР°СЃР»РµРґРѕРІР°РЅРёСЏ
class BaseControl
{
    public event EventHandler? Clicked;

    protected virtual void OnClicked() =>
        Clicked?.Invoke(this, EventArgs.Empty);
}

class CustomButton : BaseControl
{
    protected override void OnClicked()
    {
        Console.WriteLine("CustomButton pre-processing");
        base.OnClicked();
    }
}

// 2. Р’СЃРµРіРґР° РѕС‚РїРёСЃС‹РІР°Р№СЃСЏ РІ Dispose
// 3. РќРµ РґРµР»Р°Р№ event async void (РїРѕС‚РµСЂСЏРµС€СЊ РёСЃРєР»СЋС‡РµРЅРёСЏ)
// 4. РСЃРїРѕР»СЊР·СѓР№ EventHandler<T>, Р° РЅРµ СЃРІРѕР№ delegate
```

---

## РџСЂР°РєС‚РёС‡РµСЃРєРёРµ РїР°С‚С‚РµСЂРЅС‹

### Strategy pattern С‡РµСЂРµР· delegates

Delegates вЂ” СЃР°РјС‹Р№ Р»С‘РіРєРёР№ СЃРїРѕСЃРѕР± СЂРµР°Р»РёР·РѕРІР°С‚СЊ Strategy Р±РµР· РёРЅС‚РµСЂС„РµР№СЃРѕРІ Рё РєР»Р°СЃСЃРѕРІ.

```csharp
// РЎС‚СЂР°С‚РµРіРёСЏ С†РµРЅРѕРѕР±СЂР°Р·РѕРІР°РЅРёСЏ С‡РµСЂРµР· Func<T>
sealed class PricingService
{
    public decimal CalculatePrice(
        decimal basePrice,
        Func<decimal, decimal> discountStrategy,
        Func<decimal, decimal> taxStrategy)
    {
        decimal discounted = discountStrategy(basePrice);
        return taxStrategy(discounted);
    }
}

// РЎС‚СЂР°С‚РµРіРёРё вЂ” РїСЂРѕСЃС‚Рѕ С„СѓРЅРєС†РёРё
static class DiscountStrategies
{
    public static Func<decimal, decimal> NoDiscount => price => price;
    public static Func<decimal, decimal> Percentage(decimal percent) =>
        price => price * (1 - percent / 100);
    public static Func<decimal, decimal> Fixed(decimal amount) =>
        price => Math.Max(0, price - amount);
    public static Func<decimal, decimal> BlackFriday =>
        price => price < 1000 ? price * 0.8m : price * 0.7m;
}

static class TaxStrategies
{
    public static Func<decimal, decimal> Russia => price => price * 1.20m;
    public static Func<decimal, decimal> NoTax => price => price;
}

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ вЂ” РєРѕРјР±РёРЅРёСЂСѓРµРј СЃС‚СЂР°С‚РµРіРёРё РєР°Рє С…РѕС‚РёРј
var pricing = new PricingService();

decimal finalPrice = pricing.CalculatePrice(
    basePrice: 5000m,
    discountStrategy: DiscountStrategies.Percentage(15),
    taxStrategy: TaxStrategies.Russia);

Console.WriteLine($"РС‚РѕРіРѕ: {finalPrice:C}"); // 5 100,00 в‚Ѕ
```

### Callback pattern

```csharp
// Callback РїСЂРё Р·Р°РІРµСЂС€РµРЅРёРё РѕРїРµСЂР°С†РёРё
sealed class FileDownloader
{
    public async Task DownloadAsync(
        string url,
        Action<long, long>? onProgress = null,
        Action<string>? onCompleted = null,
        Action<Exception>? onError = null)
    {
        try
        {
            long totalBytes = 10_000;
            for (long downloaded = 0; downloaded < totalBytes; downloaded += 1000)
            {
                await Task.Delay(50); // РёРјРёС‚Р°С†РёСЏ Р·Р°РіСЂСѓР·РєРё
                onProgress?.Invoke(downloaded + 1000, totalBytes);
            }

            string savedPath = $"/downloads/{Guid.NewGuid()}.dat";
            onCompleted?.Invoke(savedPath);
        }
        catch (Exception ex)
        {
            onError?.Invoke(ex);
        }
    }
}

// Р’С‹Р·РѕРІ СЃ callback
var downloader = new FileDownloader();
await downloader.DownloadAsync(
    url: "https://example.com/file.zip",
    onProgress: (current, total) =>
        Console.Write($"\rР—Р°РіСЂСѓР·РєР°: {current * 100 / total}%"),
    onCompleted: path =>
        Console.WriteLine($"\nРЎРѕС…СЂР°РЅРµРЅРѕ: {path}"),
    onError: ex =>
        Console.WriteLine($"\nРћС€РёР±РєР°: {ex.Message}"));
```

### Observer pattern С‡РµСЂРµР· events

```csharp
// РџРѕР»РЅРѕС†РµРЅРЅС‹Р№ Observer С‡РµСЂРµР· events (Р±РµР· РёРЅС‚РµСЂС„РµР№СЃРѕРІ IObservable/IObserver)
sealed class StockTicker
{
    public event EventHandler<StockPriceChangedEventArgs>? PriceChanged;

    private readonly Dictionary<string, decimal> _prices = [];

    public void UpdatePrice(string symbol, decimal newPrice)
    {
        decimal oldPrice = _prices.GetValueOrDefault(symbol);
        _prices[symbol] = newPrice;

        if (oldPrice != newPrice)
        {
            PriceChanged?.Invoke(this, new(symbol, oldPrice, newPrice));
        }
    }
}

sealed record StockPriceChangedEventArgs(
    string Symbol,
    decimal OldPrice,
    decimal NewPrice) : EventArgs
{
    public decimal Change => NewPrice - OldPrice;
    public decimal ChangePercent => OldPrice != 0 ? Change / OldPrice * 100 : 0;
}

// Observer 1: Р›РѕРіРіРµСЂ
sealed class PriceLogger : IDisposable
{
    private readonly StockTicker _ticker;

    public PriceLogger(StockTicker ticker)
    {
        _ticker = ticker;
        _ticker.PriceChanged += OnPriceChanged;
    }

    private void OnPriceChanged(object? sender, StockPriceChangedEventArgs e) =>
        Console.WriteLine(
            $"[LOG] {e.Symbol}: {e.OldPrice:F2} -> {e.NewPrice:F2} ({e.ChangePercent:+0.00;-0.00}%)");

    public void Dispose() => _ticker.PriceChanged -= OnPriceChanged;
}

// Observer 2: РђР»РµСЂС‚ РїСЂРё Р±РѕР»СЊС€РѕРј РёР·РјРµРЅРµРЅРёРё
sealed class PriceAlert(StockTicker ticker, decimal thresholdPercent) : IDisposable
{
    private bool _subscribed;

    public void Start()
    {
        if (_subscribed) return;
        ticker.PriceChanged += OnPriceChanged;
        _subscribed = true;
    }

    private void OnPriceChanged(object? sender, StockPriceChangedEventArgs e)
    {
        if (Math.Abs(e.ChangePercent) >= thresholdPercent)
        {
            Console.WriteLine(
                $"[ALERT] {e.Symbol} РёР·РјРµРЅРёР»Р°СЃСЊ РЅР° {e.ChangePercent:F2}%!");
        }
    }

    public void Dispose()
    {
        ticker.PriceChanged -= OnPriceChanged;
        _subscribed = false;
    }
}

// РЎР±РѕСЂРєР°
var ticker = new StockTicker();
using var logger = new PriceLogger(ticker);
using var alert = new PriceAlert(ticker, thresholdPercent: 5m);
alert.Start();

ticker.UpdatePrice("AAPL", 150.00m);
ticker.UpdatePrice("AAPL", 160.00m); // +6.67% вЂ” СЃСЂР°Р±РѕС‚Р°РµС‚ Р°Р»РµСЂС‚
ticker.UpdatePrice("GOOGL", 2800.00m);
ticker.UpdatePrice("GOOGL", 2810.00m); // +0.36% вЂ” Р±РµР· Р°Р»РµСЂС‚Р°
```

### Р¤СѓРЅРєС†РёРѕРЅР°Р»СЊРЅС‹Рµ С†РµРїРѕС‡РєРё: Func\<T, T\> pipeline

```csharp
// Pipeline РёР· С„СѓРЅРєС†РёР№ вЂ” РєР°Р¶РґР°СЏ С‚СЂР°РЅСЃС„РѕСЂРјРёСЂСѓРµС‚ РґР°РЅРЅС‹Рµ
static class Pipeline
{
    public static Func<T, T> Compose<T>(params Func<T, T>[] steps)
    {
        return input =>
        {
            T result = input;
            foreach (var step in steps)
                result = step(result);
            return result;
        };
    }
}

// РЎС‚СЂРѕРєРѕРІС‹Р№ pipeline
Func<string, string> processText = Pipeline.Compose<string>(
    s => s.Trim(),
    s => s.ToLowerInvariant(),
    s => System.Text.RegularExpressions.Regex.Replace(s, @"\s+", " "),
    s => System.Globalization.CultureInfo.CurrentCulture.TextInfo.ToTitleCase(s)
);

string result = processText("  hello   WORLD   from   C#  ");
Console.WriteLine(result); // "Hello World From C#"

// Р§РёСЃР»РѕРІРѕР№ pipeline
Func<decimal, decimal> calculateTotal = Pipeline.Compose<decimal>(
    price => price * 0.9m,     // СЃРєРёРґРєР° 10%
    price => price + 500m,     // РґРѕСЃС‚Р°РІРєР°
    price => price * 1.20m,    // РќР”РЎ 20%
    price => Math.Round(price, 2)
);

Console.WriteLine(calculateTotal(10_000m)); // 11 400,00

// Extension method РїРѕРґС…РѕРґ вЂ” fluent API
static class FuncExtensions
{
    public static Func<T, T> Then<T>(this Func<T, T> first, Func<T, T> next)
    {
        return input => next(first(input));
    }
}

// Fluent pipeline
Func<int, int> transform = ((Func<int, int>)(x => x * 2))
    .Then(x => x + 10)
    .Then(x => x * x);

Console.WriteLine(transform(3)); // (3*2 + 10)^2 = 256
```

**Async pipeline:**

```csharp
static class AsyncPipeline
{
    public static Func<T, Task<T>> Compose<T>(params Func<T, Task<T>>[] steps)
    {
        return async input =>
        {
            T result = input;
            foreach (var step in steps)
                result = await step(result).ConfigureAwait(false);
            return result;
        };
    }
}

// РџСЂРёРјРµСЂ: РѕР±СЂР°Р±РѕС‚РєР° Р·Р°РєР°Р·Р° РєР°Рє async pipeline
var processOrder = AsyncPipeline.Compose<Order>(
    async order => { await ValidateAsync(order); return order; },
    async order => { order.Discount = await CalculateDiscountAsync(order); return order; },
    async order => { await SaveToDbAsync(order); return order; },
    async order => { await SendConfirmationEmailAsync(order); return order; }
);

// Order order = await processOrder(new Order { ... });
```

---

## РЎРІРѕРґРЅР°СЏ С‚Р°Р±Р»РёС†Р°

| РљРѕРЅС†РµРїС‚ | РљРѕРіРґР° РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ |
|---|---|
| `Action<T>` | Void callback, Р»РѕРіРёСЂРѕРІР°РЅРёРµ, side effects |
| `Func<T, TResult>` | РўСЂР°РЅСЃС„РѕСЂРјР°С†РёСЏ, СЃС‚СЂР°С‚РµРіРёСЏ, predicate |
| `Predicate<T>` | Р¤РёР»СЊС‚СЂР°С†РёСЏ РєРѕР»Р»РµРєС†РёР№ (`List<T>.FindAll`) |
| `event EventHandler<T>` | Pub/Sub, СѓРІРµРґРѕРјР»РµРЅРёСЏ, UI events |
| Custom delegate | `ref`/`out` РїР°СЂР°РјРµС‚СЂС‹, РёРјРµРЅРѕРІР°РЅРЅР°СЏ СЃРµРјР°РЅС‚РёРєР° |
| Static lambda | Hot path Р±РµР· Р°Р»Р»РѕРєР°С†РёР№ closure |
| Expression lambda | РљСЂР°С‚РєРёРµ РѕРґРЅРѕСЃС‚СЂРѕС‡РЅС‹Рµ С‚СЂР°РЅСЃС„РѕСЂРјР°С†РёРё |
| Statement lambda | РЎР»РѕР¶РЅР°СЏ Р»РѕРіРёРєР° СЃ РІРµС‚РІР»РµРЅРёСЏРјРё |

---

## РЎРј. С‚Р°РєР¶Рµ

- [РўРёРїС‹ Рё РїР°РјСЏС‚СЊ](types-and-memory.md)
- [РћРћРџ Рё РєР»Р°СЃСЃС‹](oop.md)
- [Error Handling](error-handling.md)
- [Async Рё РїРѕС‚РѕРєРё](async-threading.md)
