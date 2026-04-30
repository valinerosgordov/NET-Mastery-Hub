---
tags: [csharp, math, biginteger, half, vector, simd, math, random, bitconverter, middle, senior]
level: Middle to Senior
date: 2026-04-30
---

# Numeric Types & Math

> **Полный гайд по числовым типам и математике в .NET**: BigInteger, Half, Decimal, Vector\<T\> (SIMD), Math/MathF, Random.Shared, BitConverter, BinaryPrimitives, generic math (.NET 7+). Closes пробел "знаю int/double, не знаю что ещё есть".

---

## Что это, зачем и когда

### Зачем разные числовые типы

Разные задачи требуют разной точности, range, semantics:

| Тип | Размер | Range / Точность | Зачем |
|-----|--------|-------------------|-------|
| `byte`, `sbyte` | 1 B | 0-255 / -128..127 | Bytes, маленькие counters |
| `short`, `ushort` | 2 B | ±32K | Network protocols, image |
| `int`, `uint` | 4 B | ±2.1B | Default integer |
| `long`, `ulong` | 8 B | ±9 quintillion | Timestamps (Ticks), large counts |
| `float` (Single) | 4 B | ~7 digits | Game math, GPU |
| `double` | 8 B | ~15 digits | Default float |
| `decimal` | 16 B | 28-29 digits | **Money** (без round errors) |
| `Half` | 2 B | ~3 digits | ML, GPU memory savings |
| `BigInteger` | varies | ∞ | Crypto, mathematics |
| `Complex` | 16 B | double precision | Engineering, signal processing |
| `nint`, `nuint` | platform | varies | Pointer-sized integer (32 or 64 bit) |

### Главный вопрос Senior

> **"Почему нельзя использовать `double` для денег?"**
>
> Потому что `0.1 + 0.2 = 0.30000000000000004` (binary float не может точно представить десятичные дроби). Для денег — **`decimal`** всегда.

```csharp
double a = 0.1, b = 0.2;
Console.WriteLine(a + b);  // 0.30000000000000004 ⚠️

decimal x = 0.1m, y = 0.2m;
Console.WriteLine(x + y);  // 0.3 ✓
```

---

## 1. Integer types

### Выбор типа

```csharp
byte      // 0..255       — RGB pixel, byte arrays
sbyte     // -128..127    — почти не нужен
short     // ±32K         — file headers, audio samples
ushort    // 0..65535     — port numbers, char codes
int       // ±2.1B        — DEFAULT для целых
uint      // 0..4.3B      — file sizes, counters
long      // ±9 quint.   — timestamps, IDs
ulong     // 0..18 quint. — hashes, large counters
nint      // platform     — pointer arithmetic
nuint     // platform     — pointer-sized unsigned
```

### Default рекомендация

```csharp
// ✅ Используй int для большинства задач
int count = 100;

// ✅ long для timestamps / IDs
long ticks = DateTime.UtcNow.Ticks;
long userId = 1234567890123L;

// ❌ short / byte для loop variables — micro-optimization, vendor lock
short i;  // не быстрее int!
```

### Overflow поведение

```csharp
int max = int.MaxValue;       // 2,147,483,647
int overflow = max + 1;        // -2,147,483,648 (silent wrap!)

// ✅ checked — throws
int x = checked(max + 1);  // OverflowException

// Для всего файла — в csproj
// <CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>

// ✅ unchecked — explicit silent
int y = unchecked(max + 1);  // -2147483648

// Проверки safe
if (Math.AddCheck(a, b, out int result)) { /* ... */ }
// .NET 7+ — Int32.AddChecked
```

### `nint` / `nuint` — platform-sized

```csharp
// Размер == sizeof(IntPtr): 4 на x86, 8 на x64/ARM64
nint platformInt = 100;
nuint size = (nuint)array.Length;

// Полезно для interop
public static unsafe void Process(byte* ptr, nint count) { /* ... */ }
```

---

## 2. Floating-point types

### `float` vs `double` vs `decimal`

```csharp
float f = 3.14f;          // 4 bytes, ~7 digits — НЕ для финансов
double d = 3.14;           // 8 bytes, ~15 digits — DEFAULT для float
decimal m = 3.14m;         // 16 bytes, 28 digits — для финансов

// Размер влияет на performance
float[] floats = new float[1_000_000];     // 4 MB
double[] doubles = new double[1_000_000];  // 8 MB
decimal[] decimals = new decimal[1_000_000]; // 16 MB
```

### Когда что

| Задача | Тип | Зачем |
|--------|-----|-------|
| Money, accounting | `decimal` | Точные операции |
| Graphics, game physics | `float` | GPU работает с float, скорость |
| Scientific calculations | `double` | Балансs precision/speed |
| ML training | `float` или `Half` | Memory + speed |
| ML inference | `Half` (на GPU) | 2x memory savings |
| Distance / angles | `double` | Достаточно точности |
| Currency exchange | `decimal` | Точность важна |

### Floating-point pitfalls

```csharp
// 1. Сравнение через ==  — НЕНАДЁЖНО
double a = 0.1 + 0.2;
double b = 0.3;
a == b  // false! 0.30000000000000004 ≠ 0.3

// ✅ Сравнение с tolerance
const double Epsilon = 1e-10;
Math.Abs(a - b) < Epsilon  // true

// 2. Накопление error
double sum = 0;
for (int i = 0; i < 1000; i++)
    sum += 0.1;
// sum != 100.0!

// ✅ Используй decimal для accumulators
decimal sum = 0;
for (int i = 0; i < 1000; i++)
    sum += 0.1m;  // = 100m exactly

// 3. NaN, Infinity
double nan = 0.0 / 0.0;
double inf = 1.0 / 0.0;
double.IsNaN(nan)        // true
double.IsInfinity(inf)   // true
```

### `Half` — half-precision float (.NET 5+)

```csharp
using System;

// 16-bit float — ~3 digits точности
Half h = (Half)3.14f;
float back = (float)h;  // 3.14062f (потеря точности)

// Когда использовать:
// - ML/AI inference (GPU memory savings)
// - Compressed storage
// - Где precision не критична
```

> [!info] Half — в основном для interop с ML / GPU
> На CPU — медленнее float (нет hardware support обычно). Используй когда **memory** важна больше чем speed.

---

## 3. Case Study #1 — Финансовая система

### Задача

Online банкинг — расчёт балансов, начисление процентов. Точность critical.

### ❌ С float/double — bugs!

```csharp
public decimal CalculateInterest(decimal balance, double rate, int days)
{
    // Mixed types — точность теряется
    return balance * (decimal)(rate * days / 365.0);
}

// rate = 0.05, days = 30:
// (rate * days / 365.0) = 0.004109589041095890... (binary error!)
// При больших суммах — копейки накапливаются ⇒ legal issues
```

### ✅ Все decimal

```csharp
public decimal CalculateInterest(decimal balance, decimal rate, int days)
{
    return balance * rate * days / 365m;
}

// Use
decimal balance = 1_000_000.00m;
decimal annualRate = 0.05m;
decimal interest = CalculateInterest(balance, annualRate, 30);
// = 4109.59 точно
```

### Округление money

```csharp
decimal price = 19.995m;

// ❌ ToString("F2") — banker's rounding, может surprise
price.ToString("F2")  // "20.00" — но может быть "19.99" в зависимости от культуры

// ✅ Math.Round с явным mode
Math.Round(price, 2, MidpointRounding.AwayFromZero)  // 20.00

// Для money — обычно AwayFromZero (округляем .5 наверх)
Math.Round(0.5m, 0, MidpointRounding.AwayFromZero)   // 1
Math.Round(0.5m, 0, MidpointRounding.ToEven)         // 0 (banker's!)
```

### Storage в DB

```csharp
// EF Core
public class Account
{
    [Column(TypeName = "decimal(18, 4)")]  // 4 decimal places
    public decimal Balance { get; set; }
}

// SQL: DECIMAL(18, 4) — до 18 digits total, 4 после запятой
// Достаточно для $99 trillion с 4 decimals
```

См. [[../EFCore/basics-tracking|EF Core Basics]].

---

## 4. `BigInteger` — целые произвольной длины

### Когда

- Cryptography (RSA, EC)
- Mathematical computations
- Large factorials, fibonacci numbers
- Сomputing с numbers > 9 quintillion

```csharp
using System.Numerics;

// 100! — никаким long не вместишь
BigInteger factorial = 1;
for (int i = 2; i <= 100; i++)
    factorial *= i;

Console.WriteLine(factorial);
// 93326215443944152681699238856266700490715968264381621468592963895217599993229915608941463976156518286253697920827223758251185210916864000000000000000000000000
```

### RSA-style operations

```csharp
// Generate large prime (упрощённо)
BigInteger p = BigInteger.Parse("104729");  // small prime для примера
BigInteger q = BigInteger.Parse("104723");
BigInteger n = p * q;  // = 10970530267
BigInteger phi = (p - 1) * (q - 1);

// Modular exponentiation
BigInteger message = 12345;
BigInteger e = 65537;
BigInteger encrypted = BigInteger.ModPow(message, e, n);
```

### Performance trade-off

```csharp
// BigInteger медленнее обычных int
BigInteger slow = 1;
for (int i = 0; i < 1_000_000; i++) slow += 1;  // 100x slower than int

// ✅ Используй BigInteger ТОЛЬКО когда правда нужен >ulong range
```

См. [[../Performance/optimization-patterns|Optimization]].

---

## 5. SIMD — `Vector<T>`, `Vector128`, `Vector256`

### Что такое SIMD

**Single Instruction, Multiple Data** — одна инструкция обрабатывает несколько значений параллельно.

```
Scalar:                  SIMD (4-wide):
add a, b → c             add [a1,a2,a3,a4], [b1,b2,b3,b4] → [c1,c2,c3,c4]
1 cycle                   1 cycle (но 4 операции одновременно!)
```

### `Vector<T>` — portable SIMD

```csharp
using System.Numerics;

public float[] AddArrays(float[] a, float[] b)
{
    var result = new float[a.Length];
    int vectorSize = Vector<float>.Count;  // 4 на SSE, 8 на AVX, 16 на AVX-512
    int i = 0;
    
    // SIMD chunks
    for (; i <= a.Length - vectorSize; i += vectorSize)
    {
        var va = new Vector<float>(a, i);
        var vb = new Vector<float>(b, i);
        var vc = va + vb;
        vc.CopyTo(result, i);
    }
    
    // Tail
    for (; i < a.Length; i++)
        result[i] = a[i] + b[i];
    
    return result;
}
```

### Benchmark

```
| Method  | Length | Time    | Speedup |
|---------|--------|---------|---------|
| Scalar  | 1M     | 2.5 ms  | 1x      |
| SIMD    | 1M     | 0.6 ms  | 4x      |
```

---

## 6. Case Study #2 — Image processing с SIMD

### Задача

Brighten image: `pixel.R = min(255, R * 1.2)` для миллионов пикселей.

### ❌ Scalar

```csharp
public void Brighten(byte[] pixels)
{
    for (int i = 0; i < pixels.Length; i++)
    {
        int b = (int)(pixels[i] * 1.2f);
        pixels[i] = (byte)Math.Min(255, b);
    }
}

// 4K image (33M pixels): 60 ms
```

### ✅ SIMD

```csharp
using System.Numerics;
using System.Runtime.Intrinsics;

public unsafe void BrightenSimd(byte[] pixels)
{
    int vectorSize = Vector<byte>.Count;  // 32 bytes на AVX2
    int i = 0;
    
    var multiplier = new Vector<byte>(120);  // ~1.2 * 100
    
    fixed (byte* p = pixels)
    {
        for (; i <= pixels.Length - vectorSize; i += vectorSize)
        {
            var v = Unsafe.Read<Vector<byte>>(p + i);
            // Multiplication + saturation logic
            var result = SaturatingMultiply(v, multiplier);
            Unsafe.Write(p + i, result);
        }
    }
    
    // Tail
    for (; i < pixels.Length; i++)
    {
        int b = (int)(pixels[i] * 1.2f);
        pixels[i] = (byte)Math.Min(255, b);
    }
}

// 4K image: 12 ms — 5x speedup
```

### `Vector128`, `Vector256`, `Vector512` — explicit width

```csharp
using System.Runtime.Intrinsics;
using System.Runtime.Intrinsics.X86;

public unsafe void Sum(int* a, int* b, int* dst, int len)
{
    int i = 0;
    
    if (Avx2.IsSupported)
    {
        for (; i <= len - 8; i += 8)
        {
            var va = Avx2.LoadVector256(a + i);
            var vb = Avx2.LoadVector256(b + i);
            var vc = Avx2.Add(va, vb);
            Avx2.Store(dst + i, vc);
        }
    }
    
    // Tail scalar
    for (; i < len; i++)
        dst[i] = a[i] + b[i];
}
```

См. [[unsafe-pointers|Unsafe & Pointers]] и [[../Performance/hft-low-latency|HFT]].

---

## 7. `Math` и `MathF`

```csharp
using System;

// double versions — Math
double Math.Sin(double x);
double Math.Cos(double x);
double Math.Sqrt(double x);
double Math.Pow(double x, double y);
double Math.Log(double x);
double Math.Floor(double x);
double Math.Ceiling(double x);
double Math.Round(double x, int digits);
double Math.Abs(double x);
double Math.Min(double a, double b);
double Math.Max(double a, double b);

// float versions — MathF (faster для float)
float MathF.Sin(float x);
float MathF.Cos(float x);
// ... все аналогично
```

### Performance — используй MathF для float

```csharp
// ❌ Implicit cast slow
float x = 1.5f;
float result = (float)Math.Sin(x);  // float → double → Math.Sin → cast back

// ✅ MathF — direct
float result = MathF.Sin(x);  // 30% faster
```

### `Math.Pow` slow — для целых степеней

```csharp
// ❌ Pow медленный
double result = Math.Pow(x, 2);  // ~30 ns

// ✅ Inline
double result = x * x;  // ~1 ns

// ✅ Для общего случая — лучше custom
public static double IntPow(double x, int n)
{
    double result = 1;
    while (n > 0)
    {
        if ((n & 1) == 1) result *= x;
        x *= x;
        n >>= 1;
    }
    return result;
}
```

---

## 8. `Random` — random numbers

### `Random.Shared` (.NET 6+) — рекомендуется

```csharp
// ❌ ДО .NET 6 — new Random() не thread-safe
var rng = new Random();
// Параллельный доступ → corrupted state!

// ✅ .NET 6+ — Random.Shared (thread-safe singleton)
int x = Random.Shared.Next(100);
double d = Random.Shared.NextDouble();
byte[] bytes = new byte[16];
Random.Shared.NextBytes(bytes);
```

### Cryptographically secure

```csharp
using System.Security.Cryptography;

// ❌ Random — predictable (для security НЕ годится)
new Random().Next();  // НЕ для tokens / keys / passwords

// ✅ Cryptographic random
byte[] key = RandomNumberGenerator.GetBytes(32);  // .NET 6+
int secureNum = RandomNumberGenerator.GetInt32(100);

// Token generation
string token = Convert.ToBase64String(RandomNumberGenerator.GetBytes(32));
```

### Seed для repeatable sequences

```csharp
// Tests — same sequence
var seeded = new Random(42);
seeded.Next();  // always same value

// Production — Random.Shared (no seed)
Random.Shared.Next();
```

См. [[../AspNetCore/security-practices|Security Practices]].

---

## 9. `BitConverter` и `BinaryPrimitives`

### `BitConverter` — старый API

```csharp
// Convert primitive to/from byte[]
byte[] bytes = BitConverter.GetBytes(12345);  // [57, 48, 0, 0] little-endian

int value = BitConverter.ToInt32(bytes, 0);  // 12345

// Endianness check
BitConverter.IsLittleEndian  // true on x86/x64
```

### `BinaryPrimitives` — modern (.NET Core 2.1+)

```csharp
using System.Buffers.Binary;

Span<byte> buffer = stackalloc byte[16];

// Write — explicit endianness
BinaryPrimitives.WriteInt32LittleEndian(buffer, 12345);
BinaryPrimitives.WriteInt64BigEndian(buffer.Slice(4), 9999L);

// Read
int x = BinaryPrimitives.ReadInt32LittleEndian(buffer);
long y = BinaryPrimitives.ReadInt64BigEndian(buffer.Slice(4));

// Endianness conversion
uint le = 0x12345678;
uint be = BinaryPrimitives.ReverseEndianness(le);  // 0x78563412
```

> [!info] Modern way
> `BinaryPrimitives` >> `BitConverter`. Возможность работать со `Span<byte>` без allocation, явная endianness.

---

## 10. Case Study #3 — Network protocol parser

### Задача

Parse binary message: `[uint32 len, byte type, uint16 flags, ulong timestamp]` (всего 15 bytes).

### Решение

```csharp
public readonly struct Message
{
    public uint Length { get; init; }
    public byte Type { get; init; }
    public ushort Flags { get; init; }
    public ulong Timestamp { get; init; }
    
    public static Message Parse(ReadOnlySpan<byte> data)
    {
        if (data.Length < 15)
            throw new ArgumentException("Insufficient data");
        
        return new Message
        {
            Length = BinaryPrimitives.ReadUInt32BigEndian(data[..4]),
            Type = data[4],
            Flags = BinaryPrimitives.ReadUInt16BigEndian(data[5..7]),
            Timestamp = BinaryPrimitives.ReadUInt64BigEndian(data[7..15])
        };
    }
    
    public void Serialize(Span<byte> dest)
    {
        if (dest.Length < 15)
            throw new ArgumentException("Insufficient space");
        
        BinaryPrimitives.WriteUInt32BigEndian(dest[..4], Length);
        dest[4] = Type;
        BinaryPrimitives.WriteUInt16BigEndian(dest[5..7], Flags);
        BinaryPrimitives.WriteUInt64BigEndian(dest[7..15], Timestamp);
    }
}

// Use
ReadOnlySpan<byte> received = ...;
Message msg = Message.Parse(received);

Span<byte> buffer = stackalloc byte[15];
msg.Serialize(buffer);
await stream.WriteAsync(buffer);
```

См. [[unsafe-pointers|Unsafe & Pointers]] для альтернатив.

---

## 11. Generic Math (.NET 7+)

### Что это

C# 11 / .NET 7 ввели **`INumber<T>`** и связанные интерфейсы — generic math operations.

```csharp
// ❌ До .NET 7 — нельзя generic для math
public T Add<T>(T a, T b) where T : ??? 
{
    return a + b;  // compile error
}

// ✅ .NET 7+
public T Add<T>(T a, T b) where T : INumber<T>
{
    return a + b;
}

// Use
int sum = Add(1, 2);          // int
double sum = Add(1.5, 2.5);   // double
decimal sum = Add(1m, 2m);    // decimal
```

### Полезный case — generic statistics

```csharp
public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var v in values) sum += v;
    return sum;
}

public static T Average<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    int count = 0;
    foreach (var v in values) { sum += v; count++; }
    return sum / T.CreateChecked(count);
}

// Works with any numeric type
double avg = Average(new[] { 1.0, 2.0, 3.0 });   // 2.0
int avg = Average(new[] { 1, 2, 3 });             // 2
decimal avg = Average(new[] { 1m, 2m, 3m });      // 2m
```

### Иерархия

```
INumberBase<T>      — minimum (Zero, One, abs)
  ├── IBinaryInteger<T>      — int, long
  ├── IBinaryFloatingPointIeee754<T>  — float, double
  ├── ISignedNumber<T>       — sbyte, int, etc.
  ├── IUnsignedNumber<T>     — byte, uint, etc.
  └── INumber<T>             — все основные числовые
```

См. [[generics-deep|Generics Deep]] и [[modern-features|Modern C# Features]].

---

## 12. Case Study #4 — Game physics с Vector3

### Задача

Игра — обработка позиций объектов: position, velocity, distance.

```csharp
using System.Numerics;

public class GameObject
{
    public Vector3 Position { get; set; }
    public Vector3 Velocity { get; set; }
    
    public void Update(float deltaTime)
    {
        Position += Velocity * deltaTime;
    }
    
    public float DistanceTo(GameObject other)
    {
        return Vector3.Distance(Position, other.Position);
    }
    
    public Vector3 DirectionTo(GameObject other)
    {
        return Vector3.Normalize(other.Position - Position);
    }
}

// Built-in operations
Vector3 a = new(1, 2, 3);
Vector3 b = new(4, 5, 6);

Vector3 sum = a + b;              // (5, 7, 9)
Vector3 scaled = a * 2.0f;        // (2, 4, 6)
float dot = Vector3.Dot(a, b);    // 32 (= 1*4 + 2*5 + 3*6)
Vector3 cross = Vector3.Cross(a, b);  // (-3, 6, -3)
float length = a.Length();         // sqrt(14) ≈ 3.74
```

### Performance

`Vector3` использует SIMD под капотом → 4x быстрее ручных операций с x/y/z.

См. [[../Performance/hft-low-latency|HFT]].

---

## 13. Полезные числовые операции

### Bit operations

```csharp
// Set bit
int value = 0;
value |= 1 << 3;  // set bit 3 → 8

// Clear bit
value &= ~(1 << 3);  // clear bit 3

// Toggle
value ^= 1 << 3;

// Check
bool isSet = (value & (1 << 3)) != 0;

// .NET 6+ — BitOperations
using System.Numerics;

int popCount = BitOperations.PopCount(0xF0F0F0F0u);  // 16 (число единичных битов)
int leadingZeros = BitOperations.LeadingZeroCount(0x000000FFu);  // 24
int trailingZeros = BitOperations.TrailingZeroCount(0x000000F0u);  // 4
int log2 = BitOperations.Log2(64);  // 6
bool isPow2 = BitOperations.IsPow2(128);  // true
uint nextPow2 = BitOperations.RoundUpToPowerOf2(100);  // 128
```

### Conversions

```csharp
// Number ↔ string
int n = int.Parse("123");
int safe = int.TryParse(s, out int x) ? x : 0;
string str = 123.ToString();

// С формат
string padded = 5.ToString("D3");      // "005"
string hex = 255.ToString("X");         // "FF"
string thousands = 1234567.ToString("N0");  // "1,234,567"
string currency = 99.99m.ToString("C", CultureInfo.GetCultureInfo("ru-RU"));
                                                  // "99,99 ₽"

// Span-based parsing (.NET 6+) — без allocation
ReadOnlySpan<char> span = "12345".AsSpan();
int parsed = int.Parse(span);
bool ok = int.TryParse(span, out int v);
```

### Min/max/clamp

```csharp
// 2 numbers
int min = Math.Min(a, b);
int max = Math.Max(a, b);

// 3+ — variadic не поддерживается, но
int min3 = Math.Min(Math.Min(a, b), c);

// .NET 7+ — INumber<T>
T min = T.Min(a, b);
T max = T.Max(a, b);

// Clamp
int clamped = Math.Clamp(value, min, max);  // .NET Core 2.0+
// Если value < min → min, > max → max, иначе value
```

---

## 14. Common Pitfalls

### 1. `double` для денег

```csharp
// ❌ КЛАССИКА
double balance = 100.0;
double withdraw = 19.99;
balance -= withdraw;  // 80.01 OR 80.00999999... ?

// ✅ ВСЕГДА decimal для money
decimal balance = 100m;
decimal withdraw = 19.99m;
balance -= withdraw;  // 80.01m exactly
```

### 2. Сравнение float через ==

```csharp
// ❌
if (a == 0.1) { /* ... */ }  // редко true!

// ✅
if (Math.Abs(a - 0.1) < 1e-10) { /* ... */ }
```

### 3. Integer overflow silent

```csharp
// ❌
int big = 2_000_000_000;
int bigger = big + big;  // -294967296 silent overflow!

// ✅ checked
int bigger = checked(big + big);  // OverflowException
```

### 4. Random не thread-safe (до .NET 6)

```csharp
// ❌ При parallel access — corrupt state
private static readonly Random _rng = new Random();
// Parallel.For — _rng.Next() returns 0 forever!

// ✅ .NET 6+
Random.Shared.Next();

// Или ThreadLocal до .NET 6
private static readonly ThreadLocal<Random> _local = 
    new ThreadLocal<Random>(() => new Random());
```

### 5. `Math.Pow` для квадратов

```csharp
// ❌ Slow
double sq = Math.Pow(x, 2);  // ~30 ns

// ✅
double sq = x * x;  // 1 ns
```

### 6. Implicit conversion `int` → `double` precision loss

```csharp
long bigNumber = 9_999_999_999_999_999L;
double d = bigNumber;  // потеря precision!
long back = (long)d;    // != bigNumber

// long has 64 bits, double has 52 bits mantissa
```

### 7. `decimal` с `Math.Sin`?

```csharp
// ❌ Math не имеет decimal overloads
Math.Sin((double)1.5m);  // приходится cast → теряем точность

// ✅ Если нужна decimal precision для trig — custom implementation или другой подход
```

### 8. Negative modulo

```csharp
// C# % — symmetric (sign of left operand)
-5 % 3  // = -2 (не 1!)

// ✅ Для математического mod (всегда positive)
public static int Mod(int x, int m) => ((x % m) + m) % m;

Mod(-5, 3)  // 1
```

### 9. Не использовать `MathF` для float

```csharp
// ❌
float result = (float)Math.Sin((double)x);  // 2 cast!

// ✅
float result = MathF.Sin(x);
```

### 10. `BitConverter` allocates

```csharp
// ❌ GetBytes аллоцирует byte[]
byte[] bytes = BitConverter.GetBytes(12345);

// ✅ BinaryPrimitives + Span
Span<byte> buf = stackalloc byte[4];
BinaryPrimitives.WriteInt32LittleEndian(buf, 12345);
// 0 allocations
```

---

## 15. Best Practices

### Choosing types

- **Money → `decimal`** всегда
- **Math/physics → `double`**, для GPU/games — `float`
- **Counters/IDs → `int`** или `long` если > 2B
- **Bytes/raw data → `byte[]`** или `Span<byte>`
- **Coordinates → `Vector2`/`Vector3`** (SIMD)
- **ML inference → `Half`** (memory savings)

### Conversion safety

- **`int.TryParse`** не Parse в production
- **`checked`** в hot paths где overflow возможен
- **`Math.Clamp`** для bounded values

### Random

- **`Random.Shared`** для general (.NET 6+)
- **`RandomNumberGenerator`** для security

### Bytes

- **`BinaryPrimitives`** > `BitConverter` (modern, no alloc)
- **Span\<byte\>** для buffers
- **`stackalloc byte[]`** для маленьких

### Performance

- **`MathF`** для `float` operations
- **`Vector<T>` / SIMD** для bulk ops
- **`x * x`** не `Math.Pow(x, 2)`
- **Generic math (.NET 7+)** для reusable algorithms

См. [[../Performance/optimization-patterns|Optimization]].

---

## 16. Cheat sheet

| Need | Use |
|------|-----|
| Default integer | `int` |
| Large counter / ID | `long` |
| Money / financial | `decimal` |
| Default float | `double` |
| Game / GPU math | `float` + `MathF` |
| ML inference | `Half` |
| Crypto / huge math | `BigInteger` |
| Pointer-sized | `nint` / `nuint` |
| 2D/3D vectors | `Vector2`, `Vector3` |
| Generic math | `INumber<T>` (.NET 7+) |
| Random general | `Random.Shared` (.NET 6+) |
| Random crypto | `RandomNumberGenerator` |
| Bytes parse/write | `BinaryPrimitives` + `Span` |
| Bit operations | `BitOperations` |
| Clamp value | `Math.Clamp` |
| Safe parse | `int.TryParse` |
| Checked overflow | `checked(...)` |
| Power of 2 check | `BitOperations.IsPow2` |

---

## 17. Decision tree

```
Что за число?
│
├── Money / финансы?
│   → decimal (всегда!)
│
├── Float math?
│   ├── Performance critical, 7 digits OK → float + MathF
│   ├── Default precision → double + Math
│   ├── ML inference → Half
│   └── Vectors / matrices → Vector2/3/4
│
├── Integer?
│   ├── Default → int
│   ├── Large IDs / timestamps → long
│   ├── Bytes / RGB → byte
│   ├── > ulong range → BigInteger
│   └── Bit manipulation → uint + BitOperations
│
├── Random?
│   ├── Security (tokens) → RandomNumberGenerator
│   └── General → Random.Shared
│
└── Bytes ↔ numbers?
    ├── Modern, alloc-free → BinaryPrimitives + Span
    └── Legacy → BitConverter
```

---

## См. также

- [[csharp-basics|C# Basics]] — value types intro
- [[types-and-memory|Types & Memory]] — внутреннее представление
- [[generics-deep|Generics Deep]] — INumber\<T\> deep
- [[modern-features|Modern C# Features]] — generic math
- [[unsafe-pointers|Unsafe & Pointers]] — raw bytes
- [[../Runtime/span-layout|Span\<T\> и layout]]
- [[../Performance/optimization-patterns|Optimization Patterns]] — SIMD
- [[../Performance/hft-low-latency|HFT]] — где precision и speed критичны

## Reading list

- **Microsoft Docs — Numerics types** — learn.microsoft.com/dotnet/standard/numerics
- **Microsoft Docs — Generic Math** — learn.microsoft.com/dotnet/standard/generics/math
- **Microsoft Docs — Vector\<T\>** — learn.microsoft.com/dotnet/api/system.numerics.vector-1
- **Stephen Toub — SIMD performance** — devblogs.microsoft.com/dotnet
- **Tanner Gooding — Generic Math design** — devblogs.microsoft.com/dotnet/preview-features-in-net-6-generic-math
- **What every computer scientist should know about floating-point** — docs.oracle.com/cd/E19957-01/806-3568
