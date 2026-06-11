---
tags: [csharp, numeric, math, middle, decimal, double, float, int, biginteger]
level: Middle
date: 2026-05-07
---

# Numeric Types и Math — числа и математика

> **Integer types, floating-point, decimal, BigInteger, generic math (.NET 7+).** Когда `double`, когда `decimal`, почему `0.1 + 0.2 != 0.3`, что такое NaN/Infinity, overflow handling. Закрывает пробел: «знаю про int и double, не понимаю когда decimal обязателен и как работает Half».

---

## 0. Как читать

Если впервые работаешь с numerics — раздел 1→3. Если непонятна floating-point math — раздел 4. Decimal для money — раздел 5. Production guidance — раздел 11 (best practices), 13 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Numeric types в C#

| Тип | CLR | Размер | Range |
|-----|-----|--------|-------|
| `sbyte` | `System.SByte` | 1 | -128 to 127 |
| `byte` | `System.Byte` | 1 | 0 to 255 |
| `short` | `System.Int16` | 2 | -32,768 to 32,767 |
| `ushort` | `System.UInt16` | 2 | 0 to 65,535 |
| `int` | `System.Int32` | 4 | ±2.1 × 10⁹ |
| `uint` | `System.UInt32` | 4 | 0 to 4.3 × 10⁹ |
| `long` | `System.Int64` | 8 | ±9.2 × 10¹⁸ |
| `ulong` | `System.UInt64` | 8 | 0 to 1.8 × 10¹⁹ |
| `Int128` (.NET 7+) | `System.Int128` | 16 | huge |
| `UInt128` (.NET 7+) | `System.UInt128` | 16 | huge |
| `Half` (.NET 5+) | `System.Half` | 2 | ±65,504 (low precision) |
| `float` | `System.Single` | 4 | IEEE 754 binary32 |
| `double` | `System.Double` | 8 | IEEE 754 binary64 |
| `decimal` | `System.Decimal` | 16 | 128-bit base-10 |
| `BigInteger` | `System.Numerics.BigInteger` | varies | unlimited |
| `Complex` | `System.Numerics.Complex` | 16 | real + imaginary |

### 1.2. Главное правило

```
Деньги, финансы, любые точные decimal calculations → decimal
Science, graphics, physics, ML → double (или float для SIMD/GPU)
General-purpose integer → int
Indices, counts > 2.1B → long
File sizes, timestamps → long
Bit manipulation → uint / ulong
ML inference, GPU → Half / float
Crypto, huge numbers → BigInteger
```

### 1.3. Почему НЕ double для денег

```csharp
double a = 0.1;
double b = 0.2;
double c = a + b;
Console.WriteLine(c == 0.3);   // false! (c = 0.30000000000000004)

decimal a2 = 0.1m;
decimal b2 = 0.2m;
decimal c2 = a2 + b2;
Console.WriteLine(c2 == 0.3m);   // true
```

`double` — IEEE 754 binary64. 0.1 нельзя представить exactly в binary (как 1/3 в decimal — infinite). Накапливается ошибка.

`decimal` — 128-bit base-10. 0.1 хранится exactly. **Единственно правильный** для денег.

### 1.4. Эволюция

| Версия | Что |
|--------|-----|
| **.NET 1.0** | int/long/double/decimal, BCL Math |
| **.NET 4.0** | `BigInteger`, `Complex` |
| **.NET 5+** | `Half` type (16-bit float) |
| **.NET 7+** | `Int128`/`UInt128`, **Generic Math** interfaces |
| **.NET 8+** | Performance улучшения, AVX-512 для SIMD |

> [!info]- Если ты знаешь Java / Python / Rust / JavaScript
> **Java:** `int`/`long`/`double`/`BigDecimal`/`BigInteger`. `BigDecimal` = C# `decimal`, но boxed (heap). Generic Math — partial через autoboxing.
>
> **Python:** `int` (unlimited! как BigInteger), `float` (= double), `decimal.Decimal`, `fractions.Fraction`. Native int не overflow.
>
> **Rust:** `i8`/`u8`/`i16`/`u16`/`i32`/`u32`/`i64`/`u64`/`i128`/`u128`/`f32`/`f64`. `usize`/`isize` для memory. Generic math через traits (`Add`, `Sub`).
>
> **JavaScript:** только `number` (= double, IEEE 754 binary64) и `BigInt`. Все int operations через double — limited precision (up to 2⁵³).

> [!question]- Интервью: почему `double` не подходит для денег?
> `double` — IEEE 754 binary64 floating-point. Многие decimal numbers (0.1, 0.2, 0.3) **не представимы exactly** в binary — накапливается round-off error. `0.1 + 0.2 != 0.3` (becomes 0.30000000000000004). На больших операциях ошибка растёт. `decimal` — 128-bit base-10, 28-29 significant digits, exact representation для decimal fractions. **Обязателен** для финансов, бухгалтерии, money calculations. Производительность хуже (~10x slower than double), но точность критична. В JSON/DB — string или decimal column.

---

## 2. Integer types

### 2.1. Сigned vs unsigned

```csharp
sbyte sb = -100;   // -128 to 127
byte b = 200;      // 0 to 255

short s = -1000;   // -32,768 to 32,767
ushort us = 50000; // 0 to 65,535

int i = -1_000_000;
uint ui = 4_000_000_000U;

long l = 9_000_000_000_000L;
ulong ul = 18_000_000_000_000_000_000UL;
```

Suffixes: `U` для uint, `L` для long, `UL` для ulong. Underscores для readability (C# 7+).

### 2.2. Literals

```csharp
int decimal_ = 42;
int hex = 0xFF;            // 255
int binary = 0b1010_0011;  // C# 7+ — 163
int sep = 1_000_000;       // C# 7+ — separators

long ticks = 10_000_000L;
double pi = 3.14_15_92;
```

### 2.3. Default values

```csharp
int x = default;     // 0
long y = default;    // 0L
uint z = default;    // 0U
```

Для unitialized (uninitialized fields) — 0.

### 2.4. Conversions

```csharp
// Implicit (widening — no info loss)
int i = 100;
long l = i;        // OK
double d = i;       // OK
decimal m = i;      // OK

// Explicit (narrowing — может потеря)
long big = 5_000_000_000;
int small = (int)big;   // overflow, wraps

double pi = 3.14;
int n = (int)pi;        // 3 — truncation
```

### 2.5. Overflow

```csharp
int max = int.MaxValue;
int x = max + 1;   // -2147483648 (wraparound) — default unchecked!

checked
{
    int y = max + 1;   // throws OverflowException
}

// Per expression
int a = checked(b + c);
```

`<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>` в csproj — global.

### 2.6. Int128 / UInt128 (.NET 7+)

```csharp
Int128 huge = Int128.MaxValue;
UInt128 hugeU = UInt128.Parse("123456789012345678901234567890");
```

Для cryptography, huge numbers без BigInteger overhead.

### 2.7. nint / nuint — native-sized int

```csharp
nint n = 100;       // 32-bit на x86, 64-bit на x64
nuint nu = 200;
```

Для interop / pointer arithmetic. Использовать редко.

### 2.8. Bitwise operations

```csharp
int a = 0b1010;
int b = 0b1100;

a & b;   // 0b1000 — AND
a | b;   // 0b1110 — OR
a ^ b;   // 0b0110 — XOR
~a;       // 0b...0101 — NOT (all bits flipped)
a << 2;   // 0b101000 — left shift
a >> 1;   // 0b101 — right shift

// .NET 5+
int rotated = BitOperations.RotateLeft(a, 2);
int popCount = BitOperations.PopCount((uint)a);   // count of 1 bits
int leadingZeros = BitOperations.LeadingZeroCount((uint)a);
```

`System.Numerics.BitOperations` — efficient bit manipulation.

> [!question]- Интервью: что происходит при overflow для `int`?
> By default — **unchecked**, value **wraps** (modular arithmetic). `int.MaxValue + 1 = int.MinValue`. Это потенциальный source of bugs (security vulnerabilities в particular). `checked { ... }` или `checked(expr)` — throws `OverflowException`. `<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>` в csproj — globally enable. Best practice: `checked` для critical math (financial, sensitive), unchecked для perf-sensitive (hashing). `BigInteger` для unbounded numbers без overflow. .NET 7+ — `Int128`/`UInt128` для huge numbers без BigInteger overhead.

---

## 3. Floating-point — float, double, Half

### 3.1. IEEE 754 representation

| Тип | Размер | Mantissa | Exponent | Precision |
|-----|--------|----------|----------|-----------|
| `Half` | 16 bit | 10 bit | 5 bit | ~3-4 decimal digits |
| `float` | 32 bit | 23 bit | 8 bit | ~7 decimal digits |
| `double` | 64 bit | 52 bit | 11 bit | ~15-17 decimal digits |

Sign + exponent + mantissa = float / double / Half. IEEE 754 standard.

### 3.2. Special values

```csharp
double pi = 3.14;
double inf = double.PositiveInfinity;
double ninf = double.NegativeInfinity;
double nan = double.NaN;

double.IsNaN(nan);           // true
double.IsInfinity(inf);       // true
double.IsFinite(pi);           // true (.NET Core 2.1+)

// NaN special — never equal!
nan == nan;          // false
nan == double.NaN;   // false
nan.Equals(nan);     // true (overridden in C#!)
```

### 3.3. Operations с infinity и NaN

```csharp
1.0 / 0.0;            // Infinity
-1.0 / 0.0;           // -Infinity
0.0 / 0.0;            // NaN
double.NaN + 5;        // NaN
Math.Sqrt(-1);         // NaN
```

Operations propagate NaN. Compare с NaN — всегда false.

### 3.4. Precision pitfalls

```csharp
double sum = 0;
for (int i = 0; i < 1000; i++) sum += 0.1;
Console.WriteLine(sum);   // НЕ 100, а 99.9999999999986...

// ToString может скрыть
Console.WriteLine(0.1 + 0.2);              // "0.30000000000000004"
Console.WriteLine((0.1 + 0.2).ToString("F2"));   // "0.30"
```

### 3.5. Comparison floats

```csharp
// ❌ Direct equality fragile
if (a == b) { }

// ✅ Tolerance
const double Epsilon = 1e-9;
if (Math.Abs(a - b) < Epsilon) { }

// .NET 5+
if (Math.Abs(a - b) < double.Epsilon) { }   // double.Epsilon = smallest positive double
```

### 3.6. float (Single) — для perf

```csharp
float f = 3.14f;   // suffix 'f' обязателен
float pi = (float)Math.PI;   // explicit cast double → float
```

Используется в graphics (GPU), SIMD (`Vector<T>`), ML inference. Точность ~7 digits.

### 3.7. Half (.NET 5+) — 16-bit float

```csharp
Half h = (Half)1.5f;
Half h2 = Half.Parse("1.5");

// Arithmetic — нужен conversion
float result = (float)h + (float)h2;
```

Для ML inference (Neural network weights), GPU compute. Memory-efficient.

### 3.8. Math operations

```csharp
Math.Abs(-5);              // 5
Math.Sqrt(16);              // 4
Math.Pow(2, 10);            // 1024
Math.Log(Math.E);            // 1
Math.Log10(100);             // 2
Math.Sin(Math.PI / 2);       // 1
Math.Round(3.5);             // 4 (banker's rounding!)
Math.Round(3.5, MidpointRounding.AwayFromZero);   // 4
Math.Floor(3.7);             // 3
Math.Ceiling(3.2);           // 4
Math.Min(1, 2);              // 1
Math.Max(1, 2);              // 2

// .NET 5+ — MathF для float
MathF.Sqrt(16f);            // 4f
```

> [!question]- Интервью: что такое NaN и почему `NaN == NaN` false?
> **NaN** (Not a Number) — IEEE 754 special value, результат invalid operations: `0.0/0.0`, `Math.Sqrt(-1)`, `0 * Infinity`. `NaN == NaN` returns **false** по IEEE 754 — NaN не equal никому, включая себя. `double.IsNaN(x)` — правильный check. Pitfall: `Equals` overridden в .NET — `double.NaN.Equals(double.NaN) == true` (для consistency с Dictionary/HashSet semantics). `==` operator не overridden, IEEE-compliant. NaN propagates через arithmetic — любая operation с NaN returns NaN.

---

## 4. Floating-point pitfalls deep

### 4.1. Накопление ошибки

```csharp
double sum = 0;
for (int i = 0; i < 10_000_000; i++)
    sum += 0.000_1;
Console.WriteLine(sum);   // ожидание 1000, реально ~999.9999...
```

Каждая operation — small rounding error. Million operations — visible drift.

### 4.2. Subtraction катастрофическая loss

```csharp
double a = 1e10;
double b = a + 1.0;
double diff = b - a;   // должно быть 1.0, но reality: ~0
```

Близкие large numbers — вычитание теряет precision. Workaround: rearrange algorithm.

### 4.3. Сравнение с tolerance — сложно

```csharp
// Absolute tolerance
Math.Abs(a - b) < 1e-9;   // ❌ для small numbers may не work

// Relative tolerance
double tol = 1e-9 * Math.Max(Math.Abs(a), Math.Abs(b));
Math.Abs(a - b) < tol;    // лучше для wide range

// .NET — нет built-in approximate equality, обычно домашняя реализация
```

### 4.4. Banker's rounding

```csharp
Math.Round(0.5);    // 0! (banker's rounding — round half to even)
Math.Round(1.5);    // 2
Math.Round(2.5);    // 2! (round to even)
Math.Round(3.5);    // 4

Math.Round(0.5, MidpointRounding.AwayFromZero);   // 1 (traditional)
Math.Round(2.5, MidpointRounding.AwayFromZero);   // 3
```

Default — banker's (statistically unbiased). Для traditional rounding — `MidpointRounding.AwayFromZero`.

### 4.5. ToString и locale

```csharp
double pi = 3.14;
pi.ToString();   // "3,14" в RU locale, "3.14" в US
```

**Использовать `InvariantCulture`** для wire format / parsing:

```csharp
pi.ToString(CultureInfo.InvariantCulture);   // "3.14"
double.Parse("3.14", CultureInfo.InvariantCulture);   // 3.14
```

См. [[strings-regex]] раздел 6.

### 4.6. Parsing edge cases

```csharp
double.Parse("1e308");        // 1e308 — large
double.Parse("1e309");        // ❌ OverflowException
double.Parse("NaN");           // double.NaN
double.Parse("Infinity");      // double.PositiveInfinity

double.TryParse("not a number", out var x);   // false, x = 0
```

> [!question]- Интервью: что такое banker's rounding?
> **Round half to even** — стандарт IEEE 754. `0.5 → 0`, `1.5 → 2`, `2.5 → 2`, `3.5 → 4`. Идея: если half exactly, round к ближайшему even number. Statistically unbiased (no systematic drift up/down при aggregation). `Math.Round` default. Traditional (`0.5 → 1`, `2.5 → 3`) — `Math.Round(x, MidpointRounding.AwayFromZero)`. Для финансов выбор зависит от country/regulations — некоторые требуют banker's, некоторые traditional.

---

## 5. Decimal — для money

### 5.1. Создание

```csharp
decimal m = 100m;            // suffix 'm' обязателен
decimal price = 19.99m;
decimal big = 1_000_000_000m;

// Из double — explicit (loss possible)
decimal d = (decimal)3.14;   // 3.14m с potential round-off

// Из string — best для precise
decimal exact = decimal.Parse("0.1", CultureInfo.InvariantCulture);
```

### 5.2. Range и precision

```
decimal: 128 bits
  - 96-bit mantissa (28-29 significant digits)
  - 5-bit exponent (10⁻²⁸ to 10²⁸)
  - 1-bit sign

±7.9 × 10²⁸ to ±7.9 × 10²⁸
Precision: 28-29 decimal digits
```

Для финансов более чем достаточно (compare: world GDP ~ 10¹⁴).

### 5.3. Operations

```csharp
decimal a = 0.1m;
decimal b = 0.2m;
decimal c = a + b;          // 0.3m exact!
decimal d = a * b;           // 0.02m exact

decimal e = 1m / 3m;        // 0.3333...3 (28 digits)
// Не infinite, но truncated на precision limit

decimal.Round(3.14159m, 2, MidpointRounding.ToEven);   // 3.14m
decimal.Truncate(3.7m);     // 3m
decimal.Floor(3.7m);         // 3m
decimal.Ceiling(3.2m);       // 4m
```

### 5.4. Performance

```
| Operation       | double | decimal | ratio |
|-----------------|--------|---------|-------|
| Addition        | 1ns    | 10ns    | 10x   |
| Multiplication  | 1ns    | 30ns    | 30x   |
```

Decimal ~10-30× slower. Для UI calculations / batch processing — fine. Для high-throughput math (graphics, ML) — never decimal.

### 5.5. Money pattern

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public static Money Zero(string currency) => new(0m, currency);
    
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch");
        return new Money(Amount + other.Amount, Currency);
    }
    
    public Money operator +(Money a, Money b) => a.Add(b);
}

var price = new Money(19.99m, "USD");
var tax = new Money(1.60m, "USD");
var total = price + tax;   // 21.59 USD exact
```

### 5.6. Database / JSON

```csharp
// EF Core — всегда decimal column для money
public class Order
{
    public decimal Total { get; set; }   // SQL DECIMAL(18,2)
}

// JSON — System.Text.Json deserializes decimal correctly
// Newtonsoft.Json — тоже OK
{
  "total": 19.99
}
```

Никогда не storing money как `double`. SQL DECIMAL/NUMERIC.

> [!question]- Интервью: почему `decimal` точен, а `double` нет?
> **decimal** — 128-bit **base-10** representation: 96-bit mantissa + 5-bit exponent (10ⁿ). Decimal fractions (0.1, 0.01, 0.5) представимы exactly — нет round-off. **double** — IEEE 754 **base-2** representation. 0.1 в binary = 0.000110011001100... (infinite repeating). Mantissa truncates → small error. Errors accumulate в operations. decimal trade-off: ~10x slower, 16 bytes vs 8. Для money / финансов — единственно correct. Для science / graphics — double (precision достаточно, perf важнее).

---

## 6. BigInteger — unlimited integers

### 6.1. Создание

```csharp
using System.Numerics;

BigInteger huge = BigInteger.Parse("123456789012345678901234567890");
BigInteger fromInt = new BigInteger(42);
BigInteger fromBytes = new BigInteger(new byte[] { 1, 2, 3, 4 });
BigInteger pow = BigInteger.Pow(2, 1000);   // 2^1000
```

### 6.2. Operations

```csharp
var a = BigInteger.Parse("100000000000000000000");
var b = BigInteger.Parse("50000000000000000000");

var sum = a + b;              // 150000000000000000000
var product = a * b;           // 5e39
var quotient = a / b;          // 2
var remainder = a % b;         // 0
var power = BigInteger.Pow(a, 5);
var gcd = BigInteger.GreatestCommonDivisor(a, b);
var modPow = BigInteger.ModPow(a, b, BigInteger.Parse("1000000007"));
```

### 6.3. Performance

```
Размер | Time per add
2⁶⁴   | 1ns (~ long)
2¹²⁸  | 50ns
2²⁵⁶  | 100ns
2¹⁰²⁴ | 1000ns
```

Linear с размером digits. Для huge numbers всё равно usable.

### 6.4. Use cases

```csharp
// Crypto — RSA modular arithmetic
BigInteger publicKey = ...;
BigInteger encrypted = BigInteger.ModPow(message, publicKey, modulus);

// Combinatorics
BigInteger Factorial(int n)
{
    BigInteger result = 1;
    for (int i = 2; i <= n; i++) result *= i;
    return result;
}
Factorial(100);   // 100! = 9.33e157

// Game scoring без overflow
BigInteger totalScore = 0;
foreach (var p in players) totalScore += p.Score;
```

### 6.5. Conversions

```csharp
BigInteger big = BigInteger.Pow(2, 100);

long l = (long)big;   // ❌ OverflowException — too big
double d = (double)big;   // OK — loss of precision
string s = big.ToString();
byte[] bytes = big.ToByteArray();
```

> [!question]- Интервью: когда использовать `BigInteger`?
> Когда integer **превышает `long`** (2⁶³ ~ 9.2e18). Use cases: 1) **Cryptography** — RSA, ECC, modular exponentiation. 2) **Combinatorics** — factorials, combinations (n! быстро превышает long). 3) **Number theory** — primes, GCD больших чисел. 4) **Scientific** — astronomy, particle physics. Performance: linear с size of digits, ~50-100ns per operation для 2¹²⁸-2²⁵⁶. Для 2⁶⁴ — `long` enough, BigInteger overkill (10x slower). .NET 7+ `Int128`/`UInt128` — fixed-size huge numbers, faster than BigInteger для known size.

---

## 7. Generic Math (.NET 7+)

### 7.1. Что это

`.NET 7` ввёл **generic math interfaces** — позволяют писать generic code, который работает с любым numeric type:

```csharp
public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var v in values) sum += v;
    return sum;
}

Sum(new[] { 1, 2, 3 });               // works для int
Sum(new[] { 1.5, 2.5, 3.5 });          // works для double
Sum(new[] { 1m, 2m, 3m });             // works для decimal
```

Раньше — невозможно. Operator-based generic constraints не было.

### 7.2. Иерархия interfaces

```
INumberBase<T>           — Zero, One, parse
├── INumber<T>            — Comparable + arithmetic + radix
│   ├── IBinaryInteger<T>  — bit operations
│   ├── ISignedNumber<T>   — neg, abs
│   ├── IUnsignedNumber<T>
│   ├── IFloatingPoint<T>  — IsInfinity, IsNaN
│   └── IFixedPoint<T>     — decimal-like
└── IAdditionOperators<T>, ISubtractionOperators<T>, etc. — отдельные
```

### 7.3. Базовые operators

```csharp
public interface IAdditionOperators<TSelf, TOther, TResult>
{
    static abstract TResult operator +(TSelf left, TOther right);
}

public interface ISubtractionOperators<TSelf, TOther, TResult> { /* ... */ }
public interface IMultiplyOperators<TSelf, TOther, TResult> { /* ... */ }
public interface IDivisionOperators<TSelf, TOther, TResult> { /* ... */ }
public interface IComparisonOperators<TSelf, TOther, TResult> { /* ... */ }
public interface IEqualityOperators<TSelf, TOther, TResult> { /* ... */ }
```

`static abstract` (C# 11+) — interface может декларировать static methods/operators.

### 7.4. Constraint `INumber<T>`

```csharp
public static T Average<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    int count = 0;
    foreach (var v in values) { sum += v; count++; }
    return count == 0 ? T.Zero : sum / T.CreateChecked(count);
}

Average(new[] { 1.0, 2.0, 3.0 });   // 2.0 (double)
Average(new[] { 1m, 2m, 3m });       // 2m (decimal)
```

`T.CreateChecked(int)` — convert int → T (с overflow check).

### 7.5. Custom math types

```csharp
public readonly record struct Vector2(float X, float Y) :
    IAdditionOperators<Vector2, Vector2, Vector2>,
    IMultiplyOperators<Vector2, float, Vector2>
{
    public static Vector2 operator +(Vector2 a, Vector2 b) =>
        new(a.X + b.X, a.Y + b.Y);
    
    public static Vector2 operator *(Vector2 a, float b) =>
        new(a.X * b, a.Y * b);
}
```

Custom type integrated с generic math.

> [!question]- Интервью: что такое generic math в .NET 7+?
> Set of interfaces (`INumber<T>`, `IBinaryInteger<T>`, `IFloatingPoint<T>`, `IAdditionOperators<T,T,T>` etc.), которые позволяют **generic code работать с numeric types**. Базируется на `static abstract` interface members (C# 11+). До .NET 7 — невозможно: нельзя было declare generic constraint для `+` operator. Теперь: `where T : INumber<T>` enables `T.Zero`, `T.One`, arithmetic operators, parsing. Use cases: math libraries (matrix, vector), statistical functions, custom numeric types (Money, Coordinate). Performance: comparable native (JIT specializes per T).

---

## 8. Random и hashing

### 8.1. Random

```csharp
var rnd = new Random();
int n = rnd.Next(0, 100);     // 0-99
double d = rnd.NextDouble();   // 0.0-1.0
byte[] buf = new byte[16];
rnd.NextBytes(buf);

// Thread-safe (с .NET 6+)
Random.Shared.Next(100);   // recommended для most cases
```

`Random.Shared` (.NET 6+) — singleton thread-safe. Для seeded reproducible — `new Random(seed)`.

### 8.2. Cryptographic random

```csharp
using System.Security.Cryptography;

byte[] secure = new byte[32];
RandomNumberGenerator.Fill(secure);   // .NET Core 3+

int n = RandomNumberGenerator.GetInt32(0, 100);   // .NET 6+
```

Для security-sensitive (tokens, passwords, IDs). `Random` predictable — не для security.

### 8.3. Hash codes

```csharp
int hash = HashCode.Combine(field1, field2, field3);

var hc = new HashCode();
hc.Add(value1);
hc.Add(value2);
int finalHash = hc.ToHashCode();
```

См. [[equality-comparison]] раздел 2.

> [!question]- Интервью: чем `Random` отличается от `RandomNumberGenerator`?
> **`Random`** (System.Random) — **pseudo-random**, predictable если знаешь seed. Algorithm — fast (LCG в .NET Framework, Xoshiro256** в .NET 6+). НЕ для security. **`RandomNumberGenerator`** (System.Security.Cryptography) — **cryptographically secure** (CSPRNG). Использует OS entropy. Slower, но non-predictable. Для security tokens, passwords, session IDs. `Random.Shared` (.NET 6+) — thread-safe singleton, использует new algorithm. Best practice: `Random.Shared` для general purpose, `RandomNumberGenerator` для security.

---

## 9. SIMD и `Vector<T>`

### 9.1. Что это

SIMD (Single Instruction Multiple Data) — CPU инструкции, которые operate на multiple values одновременно (4 floats при 128-bit SSE, 8 при 256-bit AVX, 16 при 512-bit AVX-512).

```csharp
using System.Numerics;

// Vector<T> auto-sizes под CPU support
Vector<float> a = new Vector<float>(arr1);
Vector<float> b = new Vector<float>(arr2);
Vector<float> result = a + b;   // одна SIMD instruction для multiple floats
result.CopyTo(output);
```

### 9.2. Vector256/512 (.NET 8+)

```csharp
using System.Runtime.Intrinsics;

Vector256<float> v1 = Vector256.LoadUnsafe(...);
Vector256<float> v2 = Vector256.LoadUnsafe(...);
Vector256<float> sum = v1 + v2;
```

Explicit-sized vectors. Для max perf hot path.

### 9.3. Когда

✅ **Используй когда:**
- Math hot path (matrix mult, image processing).
- Large arrays of floats / ints.
- Profiled bottleneck.

❌ **Не используй когда:**
- Простой код — overhead conversion.
- Малые arrays (< 16 elements).
- Без profiling — premature optimization.

---

## 10. Conversion между numeric types

### 10.1. Implicit (widening)

```csharp
int i = 100;
long l = i;          // OK
float f = i;          // OK (info loss possible для very large int)
double d = i;         // OK
decimal m = i;        // OK

float f2 = 3.14f;
double d2 = f2;       // OK
```

### 10.2. Explicit (narrowing)

```csharp
double d = 3.14;
int i = (int)d;       // 3 — truncation
float f = (float)d;    // OK с potential precision loss

long big = 5_000_000_000;
int small = (int)big;  // overflow → wraps (или throws с checked)
```

### 10.3. Convert class

```csharp
int x = Convert.ToInt32(3.7);    // 4 — rounds (banker's)!
int y = (int)3.7;                 // 3 — truncates
int z = Convert.ToInt32("42");   // 42 — string parse

double d = Convert.ToDouble("3.14", CultureInfo.InvariantCulture);
```

`Convert` rounds, cast truncates. Catch — easy mistake.

### 10.4. Parse / TryParse

```csharp
int n = int.Parse("42");
double d = double.Parse("3.14", CultureInfo.InvariantCulture);

if (int.TryParse("42", out var x)) { /* use x */ }

// Span overload (.NET 7+ no allocation)
int n2 = int.Parse("42".AsSpan());
```

### 10.5. CreateChecked / CreateSaturating / CreateTruncating (.NET 7+)

```csharp
// Для INumber<T>
int big = 1_000_000;

T checked_t = T.CreateChecked(big);    // throws OverflowException если не fits
T saturated = T.CreateSaturating(big);  // clamps to T.MinValue/MaxValue
T truncated = T.CreateTruncating(big);  // wraps (modulo)
```

> [!question]- Интервью: чем `(int)3.7` отличается от `Convert.ToInt32(3.7)`?
> **`(int)3.7`** — explicit cast, **truncates** (drops fractional part). Result: 3. **`Convert.ToInt32(3.7)`** — uses **banker's rounding** (round half to even). Result: 4 (для 3.5 — round to 4; для 2.5 — round to 2). Common mistake: assume cast rounds. Cast always truncates toward zero. `Math.Round` separate function. `Convert` — rounds. Для **explicit truncation** — cast. Для **explicit rounding** — `Math.Round` или `Convert`.

---

## 11. Best Practices

### 11.1. Type selection

- ✅ **`decimal`** для money, всегда.
- ✅ **`int`** default для integers.
- ✅ **`long`** для large counts, file sizes, ticks.
- ✅ **`double`** для science / graphics / general math.
- ✅ **`float`** для GPU / SIMD / ML inference.
- ✅ **`BigInteger`** для unlimited (crypto).
- ❌ **`double` для money** — fundamental error.
- ❌ **`uint`/`ulong`** для public API без причины (interop issues).

### 11.2. Math

- ✅ **`InvariantCulture`** для parse / format в wire format.
- ✅ **`MidpointRounding.AwayFromZero`** если не banker's.
- ✅ **`checked`** для critical math.
- ✅ **`Math.Abs(a - b) < epsilon`** для double comparison.
- ❌ **Direct `==` для floats**.
- ❌ **`Convert.ToInt32`** когда нужно truncation (используй cast).

### 11.3. Performance

- ✅ **Pre-allocated buffers** в numerical hot path.
- ✅ **`Vector<T>` / SIMD** для large arrays.
- ✅ **`MathF`** для float (no double conversion).
- ✅ **`Random.Shared`** thread-safe.
- ❌ **`new Random()`** в loop — same seed (clock).
- ❌ **`decimal` в SIMD path** — slow.

### 11.4. Не делай

- ❌ Money через double / float.
- ❌ ID как float / double.
- ❌ Hash code на mutable numeric.
- ❌ Random для security tokens.

---

## 12. Decision tree

```
Что нужно?
│
├── Integer
│   ├── Default → int
│   ├── Large (> 2.1B) → long
│   ├── Bit manipulation → uint / ulong
│   ├── Huge (> 9e18) → Int128 / BigInteger
│   ├── Native size (interop) → nint / nuint
│   └── Tiny (memory-critical) → byte / sbyte / short
│
├── Floating-point
│   ├── Money / finance → decimal (НИКОГДА double)
│   ├── Science / general → double
│   ├── Graphics / SIMD / GPU → float
│   ├── ML inference → Half (.NET 5+)
│   └── Unlimited precision → BigInteger (integer) или decimal
│
├── Math operations
│   ├── Standard → Math (double) / MathF (float)
│   ├── SIMD → Vector<T>
│   ├── Generic → INumber<T> (.NET 7+)
│   └── Crypto → BigInteger.ModPow
│
├── Random
│   ├── General → Random.Shared
│   ├── Reproducible → new Random(seed)
│   └── Security → RandomNumberGenerator
│
└── Conversion
    ├── Widening → implicit
    ├── Narrowing с loss → cast (truncates)
    ├── Rounding → Convert.ToInt32 или Math.Round
    └── Generic checked → T.CreateChecked
```

---

## 13. Cheat sheet

```csharp
// === Integer ===
int i = 42;
long l = 9_000_000_000L;
ulong ul = 18_000_000_000UL;
Int128 i128 = Int128.MaxValue;   // .NET 7+

// === Float ===
float f = 3.14f;
double d = 3.14;
Half h = (Half)1.5f;   // .NET 5+

// === Decimal ===
decimal m = 19.99m;
decimal exact = decimal.Parse("0.1", CultureInfo.InvariantCulture);

// === BigInteger ===
using System.Numerics;
BigInteger huge = BigInteger.Pow(2, 1000);
BigInteger factorial = ...;  // calc in loop

// === Math ===
Math.Sqrt(16);
Math.Pow(2, 10);
Math.Round(3.14, 1);              // banker's rounding
Math.Round(3.5, MidpointRounding.AwayFromZero);
MathF.Sqrt(16f);                  // float version

// === Bit operations ===
int rotated = BitOperations.RotateLeft(value, 4);
int popCount = BitOperations.PopCount((uint)value);

// === Comparison ===
const double Epsilon = 1e-9;
if (Math.Abs(a - b) < Epsilon) { }

// === Parsing ===
int n = int.Parse("42");
if (int.TryParse(input, out var x)) { }
double d2 = double.Parse("3.14", CultureInfo.InvariantCulture);

// === Random ===
int rnd = Random.Shared.Next(100);
byte[] secure = new byte[32];
RandomNumberGenerator.Fill(secure);

// === Generic Math (.NET 7+) ===
public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var v in values) sum += v;
    return sum;
}

// === SIMD ===
Vector<float> v1 = new Vector<float>(arr1);
Vector<float> v2 = new Vector<float>(arr2);
Vector<float> result = v1 + v2;

// === Conversions ===
int truncated = (int)3.7;            // 3
int rounded = Convert.ToInt32(3.7);   // 4 (banker's)
T t = T.CreateChecked(value);         // .NET 7+
```

---

## 14. Common Pitfalls

### 14.1. Money через double

```csharp
double total = 0;
foreach (var item in items)
    total += item.Price;   // ❌ accumulates rounding errors
```

**Фикс:** `decimal`.

### 14.2. Direct float equality

```csharp
if (price == 19.99) { }   // ❌ fragile
```

**Фикс:** `Math.Abs(price - 19.99) < 0.001`.

### 14.3. Cast vs Convert confusion

```csharp
int n = (int)3.7;                   // 3 — truncate
int n2 = Convert.ToInt32(3.7);      // 4 — banker's round
// Surprise если знаешь только cast
```

**Фикс:** explicit `Math.Round` если intent.

### 14.4. Parse без culture

```csharp
double d = double.Parse("3.14");   // ❌ throws на ru-RU (запятая ожидается)
```

**Фикс:** `CultureInfo.InvariantCulture`.

### 14.5. Random в loop без seed

```csharp
for (int i = 0; i < 10; i++)
{
    var r = new Random();   // ❌ same seed (system time)
    Console.WriteLine(r.Next());
}
```

**Фикс:** `Random.Shared` или один Random instance.

### 14.6. Int overflow в hot path

```csharp
int sum = 0;
foreach (var n in millions) sum += n;   // ❌ wraps без checked
```

**Фикс:** `long sum = 0;` или `checked { ... }`.

### 14.7. Banker's rounding surprise

```csharp
Math.Round(0.5);   // 0! — banker's
Math.Round(1.5);   // 2
// Counterintuitive для non-engineers
```

**Фикс:** `MidpointRounding.AwayFromZero` если ожидание traditional.

### 14.8. NaN comparison

```csharp
if (value == double.NaN) { }   // ❌ всегда false
```

**Фикс:** `double.IsNaN(value)`.

### 14.9. BigInteger в hot path

```csharp
BigInteger sum = 0;
for (int i = 0; i < 1_000_000; i++) sum += i;   // ❌ slow
```

**Фикс:** `long sum` если fits.

### 14.10. decimal → double cast loses precision

```csharp
decimal m = 0.1m + 0.2m;   // 0.3m exact
double d = (double)m;      // 0.3 — ish (re-introduced double error)
```

**Фикс:** оставайся в decimal на всём пути.

> [!question]- Интервью: топ-3 ошибки с numerics?
> 1) **`double` для money** — accumulates rounding errors. `0.1 + 0.2 != 0.3`. Always `decimal` для финансов. 2) **Direct `==`** для floats. Используй `Math.Abs(a - b) < epsilon`. 3) **Parse / ToString без `InvariantCulture`** — locale-dependent (запятая vs точка). Для wire format / JSON — always `InvariantCulture`. Бонус: `Math.Round` использует banker's rounding (0.5 → 0!), `Convert.ToInt32` тоже. `(int)cast` — truncates, не rounds.

---

## 15. Practice exercises

### 15.1. Money с правильными rounding

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public Money Round(int decimals = 2) =>
        this with { Amount = decimal.Round(Amount, decimals, MidpointRounding.AwayFromZero) };
    
    public Money Apply(decimal taxRate)
    {
        if (taxRate < 0) throw new ArgumentException();
        return this with { Amount = Amount * (1 + taxRate) };
    }
    
    public static Money operator +(Money a, Money b)
    {
        if (a.Currency != b.Currency) throw new InvalidOperationException();
        return a with { Amount = a.Amount + b.Amount };
    }
}

var price = new Money(19.99m, "USD");
var withTax = price.Apply(0.085m).Round();   // 21.69 USD
```

### 15.2. Generic statistics (`INumber<T>`)

```csharp
public static class Stats
{
    public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
    {
        T result = T.Zero;
        foreach (var v in values) result += v;
        return result;
    }
    
    public static T Average<T>(IEnumerable<T> values) where T : INumber<T>
    {
        T sum = T.Zero;
        int count = 0;
        foreach (var v in values) { sum += v; count++; }
        if (count == 0) return T.Zero;
        return sum / T.CreateChecked(count);
    }
    
    public static T Max<T>(IEnumerable<T> values) where T : INumber<T>
    {
        bool first = true;
        T max = T.Zero;
        foreach (var v in values)
        {
            if (first) { max = v; first = false; }
            else if (v > max) max = v;
        }
        return max;
    }
}

Stats.Average(new[] { 1, 2, 3 });            // 2 (int)
Stats.Average(new[] { 1.0, 2.0, 3.0 });       // 2.0 (double)
Stats.Average(new[] { 1m, 2m, 3m });          // 2m (decimal)
```

### 15.3. Approximate equality helper

```csharp
public static class FloatHelpers
{
    public static bool ApproxEquals(double a, double b, double tolerance = 1e-9)
    {
        if (double.IsNaN(a) || double.IsNaN(b)) return false;
        if (double.IsInfinity(a) || double.IsInfinity(b)) return a == b;
        
        double diff = Math.Abs(a - b);
        if (diff < tolerance) return true;
        
        double largest = Math.Max(Math.Abs(a), Math.Abs(b));
        return diff < tolerance * largest;
    }
}

FloatHelpers.ApproxEquals(0.1 + 0.2, 0.3);   // true
FloatHelpers.ApproxEquals(1e10 + 1, 1e10);    // false (relative tolerance)
```

---

## 16. Что читать дальше

1. **[[strings-regex|Strings]]** — InvariantCulture parsing.
2. **[[equality-comparison|Equality]]** — float comparison.
3. **System.Numerics** — Vector, Matrix, Quaternion.
4. **System.Runtime.Intrinsics** — explicit SIMD.
5. **IEEE 754 specification** — floating-point standard.

---

## 17. См. также

- [[strings-regex|Strings]] — number parsing
- [[equality-comparison|Equality]] — float ==
- [[generics-deep|Generics]] — `INumber<T>` constraints
- System.Numerics namespace
- BenchmarkDotNet для perf

---

## 18. Reading list

- **Microsoft Docs — Numeric types** — learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/numeric-types
- **Microsoft Docs — Generic Math** — learn.microsoft.com/dotnet/standard/generics/math
- **Microsoft Docs — IEEE 754** — learn.microsoft.com/dotnet/api/system.double
- **Microsoft Docs — BigInteger** — learn.microsoft.com/dotnet/api/system.numerics.biginteger
- **What Every Computer Scientist Should Know About Floating-Point** (Goldberg, 1991)
- **Tanner Gooding — Generic Math design** — devblogs.microsoft.com
- **Stephen Toub — Performance numerics** — devblogs.microsoft.com
- **Bill Wagner — Effective C# (numeric items)**
