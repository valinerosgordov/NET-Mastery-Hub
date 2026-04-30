---
tags:
  - span
  - memory
  - performance
  - zero-allocation
  - struct-layout
  - simd
  - pipelines
  - utf8
  - inline-arrays
  - deepdive
complexity: Senior
date: 2026-04-30
---

# Zero-allocation: Span, Memory, Layout, SIMD, Pipelines

> Самая глубокая заметка по работе с памятью без аллокаций. Закрывает Span/Memory, struct layout, SIMD/Vector, hardware intrinsics, pipelines, inline arrays — всё что нужно для performance-критичного кода на Senior уровне.

---

## Что это, зачем и когда

### Что такое zero-allocation?
**Код, который не создаёт новых объектов в куче.** Нет аллокаций → нет нагрузки на GC → нет пауз → максимальная скорость.

**Аналогия:** Вместо ксерокопирования страницы книги (аллокация), указываешь пальцем на нужный абзац (Span). Книга одна, копий нет.

### Зачем?

| С аллокациями | Zero-allocation |
|--------------|-----------------|
| `string.Substring()` → новая строка в куче | `span.Slice()` → окно в ту же память |
| Парсинг CSV: 1M строк → 1M аллокаций | Span: 0 аллокаций |
| GC собирает мусор → паузы 50-200мс | GC почти не запускается |
| Проверка через 50 разделителей → 50 проходов | SearchValues + SIMD: один проход 16 байт за раз |

### Когда использовать?

| Инструмент | Когда | Ограничения |
|-----------|-------|-------------|
| **Span\<T\>** | Синхронный hot path, парсинг, слайсинг | ref struct — только стек, нельзя в async |
| **Memory\<T\>** | Нужно хранить в поле / передать в async | Чуть медленнее Span |
| **stackalloc** | Маленький буфер (< 1 KB) на стеке | Фиксированный размер |
| **ArrayPool\<T\>** | Переиспользование массивов | Не забыть Return() |
| **Vector\<T\> / Vector256\<T\>** | SIMD batch операции | Размер зависит от CPU |
| **Pipelines** | Сетевой/файловый I/O с буферами | Сложнее в использовании |
| **StructLayout** | Контроль расположения полей в памяти | Interop, низкоуровневая оптимизация |
| **SearchValues\<T\>** (.NET 8+) | Поиск любого из набора символов | Только для search-patterns |

**Правило:** используй zero-allocation только на hot path (1000+ ops/sec). На cold path — читаемость важнее.

---

## 1. Span\<T\> — окно в память без аллокаций

### Что это

`Span<T>` — **ссылка на непрерывный участок памяти** любого происхождения: массив, стек, native. Без копирования, без аллокаций.

### Внутреннее устройство

```csharp
public readonly ref struct Span<T>
{
    internal readonly ref T _reference;  // managed pointer (.NET 7+ — ref field)
    private readonly int _length;
    // Всего 2 поля: 16 bytes на x64
}
```

```
Массив в памяти:
[Header][Length][ 0 ][ 1 ][ 2 ][ 3 ][ 4 ][ 5 ][ 6 ][ 7 ]

Span<int> span = array.AsSpan(2, 4):
                        ↓ _reference          ↓ _length = 4
                  [ 0 ][ 1 ][ 2 ][ 3 ][ 4 ][ 5 ][ 6 ][ 7 ]
                              ^^^^^^^^^^^^^^^^^^^
```

### Почему ref struct

`Span<T>` содержит **managed pointer** (`ref T`) — ссылку, которая может указывать на стек. Если Span попадёт на heap (поле класса, boxing), GC не сможет отследить → memory corruption.

| Ограничение ref struct | Причина |
|------------------------|---------|
| Не может быть полем класса | Класс на heap, ref на stack → dangling pointer |
| Не может быть в async/await | State machine — класс на heap |
| Не может быть в замыкании | Замыкание — объект на heap |
| Не может быть boxed | Boxing → heap |
| Не может быть в `IEnumerable` | yield → state machine на heap |
| Не может быть generic argument | T для класса = на heap |

### .NET 7+: ref fields внутри ref struct

С .NET 7 появились **ref fields** — теперь `Span<T>` действительно содержит `ref T`, не `ByReference<T>`. Можно создавать свои ref structs:

```csharp
public readonly ref struct Window<T>
{
    public readonly ref T First;
    public readonly int Length;

    public Window(ref T reference, int length)
    {
        First = ref reference;  // ref field assignment
        Length = length;
    }
}
```

### Span vs ReadOnlySpan

```csharp
// Span<T> — чтение и запись
Span<byte> writable = stackalloc byte[256];
writable[0] = 0xFF;

// ReadOnlySpan<T> — только чтение
ReadOnlySpan<char> text = "Hello, World!".AsSpan();
ReadOnlySpan<char> hello = text[..5];

// string → ReadOnlySpan<char> неявно
void Process(ReadOnlySpan<char> data) { }
Process("some text");  // без аллокации
```

### Парсинг без аллокаций — пример

```csharp
// Парсинг CSV строки без string.Split
public static void ParseCsvLine(ReadOnlySpan<char> line)
{
    while (!line.IsEmpty)
    {
        int comma = line.IndexOf(',');
        ReadOnlySpan<char> field = comma >= 0 ? line[..comma] : line;

        if (int.TryParse(field, out int value))
            ProcessValue(value);

        line = comma >= 0 ? line[(comma + 1)..] : ReadOnlySpan<char>.Empty;
    }
}

// Парсинг бинарного протокола
public static (ReadOnlySpan<byte> payload, ushort checksum) ParsePacket(
    ReadOnlySpan<byte> raw)
{
    int length = BinaryPrimitives.ReadInt32LittleEndian(raw);
    var payload = raw.Slice(4, length);
    ushort checksum = BinaryPrimitives.ReadUInt16LittleEndian(raw[(4 + length)..]);
    return (payload, checksum);
}
```

> [!warning] Span и async несовместимы
> `Span<T>` нельзя использовать в `async` методах. Решение: `Memory<T>` или выделить синхронную часть в отдельный метод.

---

## 2. Memory\<T\> — Span для async мира

### Когда Span недостаточно

`Memory<T>` — **обёртка**, которая может жить на heap. Можно передавать в async методы, хранить в полях.

```csharp
public readonly struct Memory<T>
{
    private readonly object? _object;  // Array | MemoryManager<T> | string
    private readonly int _index;
    private readonly int _length;

    public Span<T> Span => /* создаёт Span из _object + _index + _length */;
}
```

### Использование в async

```csharp
public async Task ProcessAsync(Memory<byte> buffer, CancellationToken ct)
{
    int bytesRead = await stream.ReadAsync(buffer, ct);

    // Для обработки — получаем Span (только в sync контексте)
    ProcessData(buffer.Span[..bytesRead]);
}

private void ProcessData(Span<byte> data)
{
    BinaryPrimitives.ReadInt32LittleEndian(data);
}
```

### MemoryManager\<T\> — кастомный owner

```csharp
public sealed class NativeMemoryManager : MemoryManager<byte>
{
    private IntPtr _ptr;
    private readonly int _length;

    public NativeMemoryManager(int size)
    {
        _length = size;
        _ptr = Marshal.AllocHGlobal(size);
    }

    public override Span<byte> GetSpan() =>
        new Span<byte>((void*)_ptr, _length);

    public override MemoryHandle Pin(int elementIndex = 0) =>
        new MemoryHandle((byte*)_ptr + elementIndex, default, this);

    public override void Unpin() { }

    protected override void Dispose(bool disposing)
    {
        if (_ptr != IntPtr.Zero)
        {
            Marshal.FreeHGlobal(_ptr);
            _ptr = IntPtr.Zero;
        }
    }
}

// Использование
using var nativeMem = new NativeMemoryManager(1024);
Memory<byte> mem = nativeMem.Memory;  // Memory над native памятью
```

### Span vs Memory

| Критерий | `Span<T>` | `Memory<T>` |
|----------|-----------|-------------|
| Аллокации | Zero | Zero (struct обёртка) |
| async/await | ❌ | ✅ |
| Поле класса | ❌ | ✅ |
| Производительность | Максимум | Чуть медленнее (.Span) |
| stackalloc | ✅ | ❌ |
| Источники | Array, stack, native | Array, MemoryManager |
| Pin для interop | `fixed` | `Pin()` |

---

## 3. stackalloc — память на стеке

```csharp
Span<byte> buffer = stackalloc byte[256];

// Порог безопасности — не больше ~1 KB на стеке
const int StackThreshold = 1024;

Span<byte> buf = size <= StackThreshold
    ? stackalloc byte[size]
    : new byte[size];

// stackalloc + инициализация
Span<int> ids = stackalloc int[] { 1, 2, 3, 4, 5 };
```

| Характеристика | `stackalloc` | `new T[]` | `ArrayPool.Rent` |
|----------------|-------------|-----------|-------------------|
| Где | Stack | Heap (Gen 0) | Heap (pooled) |
| Скорость | Мгновенно (mov rsp) | Быстро | Мгновенно (reuse) |
| GC pressure | Нет | Да | Нет |
| Размер | ~1 KB безопасно | Неограничен | Неограничен |
| Освобождение | Автоматически | GC | Manual (Return) |

> [!warning] Stack Overflow
> Стек потока ~1 MB по умолчанию. `stackalloc byte[1_000_000]` → StackOverflowException. Всегда проверять размер и fallback на heap.

### stackalloc в async — нельзя?

Прямо в async — нельзя. Но можно через Span-only метод:

```csharp
public async Task ProcessAsync(byte[] buffer)
{
    await Task.Yield();
    
    // Можно вызвать sync метод, который использует stackalloc
    var hash = ComputeHash(buffer);
}

private static int ComputeHash(byte[] buffer)
{
    Span<byte> temp = stackalloc byte[256];
    // ...
    return result;
}
```

---

## 4. ArrayPool\<T\> и MemoryPool\<T\>

### ArrayPool — переиспользование массивов

```csharp
var pool = ArrayPool<byte>.Shared;

byte[] buffer = pool.Rent(minimumLength: 8192);
try
{
    // используем buffer (может быть длиннее запрошенного — buffer.Length >= 8192)
    int read = stream.Read(buffer, 0, 8192);
    Process(buffer, 0, read);
}
finally
{
    pool.Return(buffer, clearArray: true);  // clear если массив содержит секреты
}
```

### MemoryPool\<T\> — для Memory\<T\>

```csharp
var pool = MemoryPool<byte>.Shared;

using IMemoryOwner<byte> owner = pool.Rent(minBufferSize: 8192);
Memory<byte> memory = owner.Memory;

await stream.ReadAsync(memory);
Process(memory.Span);

// owner.Dispose() автоматически возвращает в пул
```

### Custom buckets

```csharp
// Свой ArrayPool с фиксированными bucket sizes
var customPool = ArrayPool<byte>.Create(maxArrayLength: 1024 * 1024, maxArraysPerBucket: 50);
```

### `RecyclableMemoryStream` — Microsoft решение для MemoryStream

```csharp
private static readonly RecyclableMemoryStreamManager Manager = new();

using var ms = Manager.GetStream("tag");  // не аллоцирует byte[]
// работаем с ms как с обычным MemoryStream
// при Dispose — буфер возвращается в пул
```

---

## 5. SIMD: Vector\<T\>, Vector256\<T\>, Hardware Intrinsics

### Что такое SIMD

**Single Instruction Multiple Data** — одна CPU инструкция работает над несколькими элементами одновременно. На x86 — `SSE`/`AVX`/`AVX-512`. На ARM — `NEON`/`SVE`.

```
Сложение 8 int за одну инструкцию (AVX2):
[ 1  2  3  4  5  6  7  8 ]   ← Vector256<int>
+
[10 20 30 40 50 60 70 80 ]
=
[11 22 33 44 55 66 77 88 ]
```

### Vector\<T\> — портативный SIMD

`Vector<T>` адаптируется к runtime: на AVX2 — 8 int за раз, на ARM NEON — 4 int.

```csharp
using System.Numerics;

public static int SumVector(int[] data)
{
    if (Vector.IsHardwareAccelerated)
    {
        Vector<int> sum = Vector<int>.Zero;
        int i = 0;
        int simdLen = Vector<int>.Count;  // 4 / 8 / 16 в зависимости от CPU
        
        for (; i <= data.Length - simdLen; i += simdLen)
        {
            sum += new Vector<int>(data, i);
        }

        int total = 0;
        for (int j = 0; j < simdLen; j++) total += sum[j];

        // Хвост
        for (; i < data.Length; i++) total += data[i];

        return total;
    }
    else
    {
        // Fallback
        int total = 0;
        foreach (var x in data) total += x;
        return total;
    }
}
```

Простой scalar sum: ~2 ns/element. SIMD: ~0.3 ns/element. **6-7x ускорение**.

### Vector128 / Vector256 / Vector512 — фиксированный размер

```csharp
using System.Runtime.Intrinsics;
using System.Runtime.Intrinsics.X86;

public static int SumAvx2(ReadOnlySpan<int> data)
{
    if (!Avx2.IsSupported) return SumScalar(data);

    Vector256<int> sum = Vector256<int>.Zero;
    int i = 0;
    
    for (; i <= data.Length - 8; i += 8)
    {
        // Загрузка 8 int
        Vector256<int> v = Vector256.Create(data.Slice(i, 8));
        sum = Avx2.Add(sum, v);
    }

    // Horizontal sum
    int total = Vector256.Sum(sum);

    for (; i < data.Length; i++) total += data[i];
    return total;
}
```

| Тип | Bits | Где |
|-----|------|-----|
| `Vector64<T>` | 64 | ARM NEON, частично x86 |
| `Vector128<T>` | 128 | SSE/AdvSimd — большинство CPU |
| `Vector256<T>` | 256 | AVX/AVX2 — Intel/AMD 2013+ |
| `Vector512<T>` | 512 | AVX-512 — Intel server, недавние client (.NET 8+) |
| `Vector<T>` | depends | Адаптируется |

### Hardware Intrinsics

`System.Runtime.Intrinsics.X86` и `.Arm` — прямые wrapper'ы CPU инструкций:

```csharp
using System.Runtime.Intrinsics.X86;

if (Sse2.IsSupported)
{
    var a = Vector128.Create(1, 2, 3, 4);
    var b = Vector128.Create(10, 20, 30, 40);
    var c = Sse2.Add(a, b);  // direct PADDD instruction
}

if (Avx2.IsSupported)
{
    // AVX2: shuffle, gather, FMA
    var v = Avx2.GatherVector256(basePtr, indices, scale: 4);
}

if (Bmi1.IsSupported)
{
    // Bit Manipulation Instructions
    int trailingZeros = (int)Bmi1.TrailingZeroCount(value);
}

if (Aes.IsSupported)
{
    // AES-NI
    var encrypted = Aes.Encrypt(state, roundKey);
}
```

ARM:
```csharp
using System.Runtime.Intrinsics.Arm;

if (AdvSimd.IsSupported)
{
    // ARM NEON
    var sum = AdvSimd.Add(a, b);
}
```

### Готовые API на SIMD

В .NET 8+ многие методы автоматически используют SIMD под капотом:

```csharp
// Все эти — SIMD-accelerated
ReadOnlySpan<byte> data = ...;

int idx = data.IndexOf((byte)0xFF);          // SIMD scan
int idx2 = data.IndexOfAny((byte)0, (byte)10, (byte)13);  // SIMD multi-byte search
bool eq = span1.SequenceEqual(span2);        // SIMD compare
data.Reverse();                              // SIMD shuffle

// String operations
string s = ...;
int idx3 = s.AsSpan().IndexOf("needle");     // SIMD-аccelerated
```

### SearchValues\<T\> (.NET 8+)

Pre-computed table для быстрого поиска любого из набора символов:

```csharp
private static readonly SearchValues<char> InvalidChars = 
    SearchValues.Create("<>:\"|?*\0");

public bool IsValidFileName(string name) =>
    name.AsSpan().IndexOfAny(InvalidChars) < 0;
```

Для маленьких наборов (≤ 8) — сразу SIMD. Для больших — bit table + SIMD. **2-10x быстрее** ручного `IndexOf` по каждому символу.

```csharp
// Byte search
private static readonly SearchValues<byte> Whitespace = 
    SearchValues.Create([(byte)' ', (byte)'\t', (byte)'\r', (byte)'\n']);

ReadOnlySpan<byte> buffer = ...;
int idx = buffer.IndexOfAny(Whitespace);
```

---

## 6. Tensor primitives (.NET 9+)

`System.Numerics.Tensors` — high-level API для тензорной математики, под капотом SIMD:

```csharp
using System.Numerics.Tensors;

ReadOnlySpan<float> a = ...;
ReadOnlySpan<float> b = ...;
Span<float> result = stackalloc float[a.Length];

TensorPrimitives.Add(a, b, result);             // result = a + b
TensorPrimitives.Multiply(a, b, result);        // element-wise
float dot = TensorPrimitives.Dot(a, b);
float l2 = TensorPrimitives.Norm(a);
TensorPrimitives.Sigmoid(a, result);            // ML activation
TensorPrimitives.Softmax(a, result);
```

Используется в ML.NET, Microsoft.Extensions.AI для embeddings.

---

## 7. UTF-8 strings и `u8` literals (.NET 7+)

В сетевых протоколах, JSON, web — данные часто в UTF-8. Конвертация в `string` (UTF-16) — лишняя аллокация.

### `u8` literals — ReadOnlySpan\<byte\> в compile-time

```csharp
ReadOnlySpan<byte> hello = "Hello"u8;  // 5 bytes UTF-8 в .data секции
ReadOnlySpan<byte> contentType = "application/json"u8;

// Сравнение без аллокаций
public bool IsJson(ReadOnlySpan<byte> contentTypeHeader) =>
    contentTypeHeader.StartsWith("application/json"u8);
```

### Utf8Parser / Utf8Formatter

```csharp
ReadOnlySpan<byte> input = "12345"u8;
if (Utf8Parser.TryParse(input, out int value, out int consumed))
{
    Console.WriteLine(value);  // 12345
}

Span<byte> buffer = stackalloc byte[16];
if (Utf8Formatter.TryFormat(42, buffer, out int written))
{
    // buffer[..written] содержит "42" в UTF-8
}
```

Полезно для: JSON parsing, HTTP headers, Kafka messages.

### IUtf8SpanFormattable / IUtf8SpanParsable\<T\> (.NET 8+)

Стандартизованные интерфейсы для UTF-8:

```csharp
public readonly struct OrderId : IUtf8SpanFormattable
{
    private readonly Guid _value;

    public bool TryFormat(Span<byte> destination, out int bytesWritten,
                          ReadOnlySpan<char> format, IFormatProvider? provider)
    {
        return _value.TryFormat(destination, out bytesWritten, format);
    }
}
```

`System.Text.Json` использует это автоматически — нет лишней конвертации UTF-8 → string → byte[].

---

## 8. Inline Arrays (C# 12)

Раньше fixed-size buffers требовали `unsafe`. C# 12 ввёл **inline arrays** — managed альтернатива:

```csharp
[InlineArray(16)]
public struct Buffer16<T>
{
    private T _element0;
}

// Использование
Buffer16<int> buffer = default;
buffer[0] = 42;
buffer[15] = 100;

ReadOnlySpan<int> span = buffer;  // implicit conversion
```

**Применение:**
- Fixed-size buffers без `unsafe`
- Embedded arrays в struct (cache-friendly layout)
- AOT-compatible (без heap allocation)

```csharp
// Performance counter — все cells в одной struct, нет heap
[InlineArray(64)]
public struct CellBuffer
{
    private long _v;
}

public class StripedCounter
{
    private CellBuffer _cells;

    public void Increment()
    {
        int idx = Thread.GetCurrentProcessorId() & 63;
        ref long cell = ref _cells[idx];
        Interlocked.Increment(ref cell);
    }
}
```

---

## 9. Data Alignment и Padding

### Как процессор читает память

CPU читает память **словами** (8 bytes на x64). Если данные не выровнены по границе своего размера — два чтения вместо одного.

**Правило выравнивания:** `Offset % Size == 0`.

```
int (4 bytes) должен начинаться по адресу, кратному 4:
  0x00 → ✓, 0x04 → ✓, 0x03 → ✗
```

### Padding в структурах

```csharp
struct BadLayout  // Ожидаем 13 bytes, реально 24 bytes!
{
    byte a;       // offset 0, size 1
    // [padding 7 bytes]
    long b;       // offset 8, size 8
    byte c;       // offset 16, size 1
    // [padding 3 bytes]
    int d;        // offset 20, size 4
}
// Total: 24 bytes (11 bytes — padding!)

struct GoodLayout  // 16 bytes
{
    long b;       // offset 0
    int d;        // offset 8
    byte a;       // offset 12
    byte c;       // offset 13
    // [padding 2 bytes]
}
// Total: 16 bytes
```

> [!info] Правило: располагай поля от большего к меньшему
> `long` → `int` → `short` → `byte`. Минимизирует padding. Для hot structs (массивы, tight loops) — разница в производительности значительная (cache line utilization).

### Cache line awareness

L1 cache line обычно 64 байта (на ARM Apple — 128). Если struct ≤ 64 — поместится в одну cache line, fast access.

```csharp
// ❌ Struct 100 байт — не помещается в cache line
struct LargeStruct
{
    public Guid A, B, C;  // 16 × 3 = 48
    public long D, E, F, G; // 8 × 4 = 32
    public int H, I, J, K, L;  // 4 × 5 = 20
    // Total: 100 bytes
}

// ✅ Группировать hot fields в первые 64 байта
struct HotPath
{
    public long Counter;       // hot, в cache line 0
    public long Timestamp;     // hot
    public Guid Id;            // hot
    public int Status;         // hot, в cache line 0 (52 bytes)
    
    public string LongDescription;  // cold, в cache line 1+
    public Dictionary<string, string> Tags;
}
```

---

## 10. StructLayout и FieldOffset

### Explicit Layout — union

```csharp
[StructLayout(LayoutKind.Explicit)]
struct Packet
{
    [FieldOffset(0)] public int RawValue;     // 4 bytes
    [FieldOffset(0)] public byte Byte0;
    [FieldOffset(1)] public byte Byte1;
    [FieldOffset(2)] public byte Byte2;
    [FieldOffset(3)] public byte Byte3;
}

var p = new Packet { RawValue = 0xAABBCCDD };
Console.WriteLine($"0x{p.Byte0:X2}");  // 0xDD (little-endian)
```

### Sequential Layout для interop

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]  // без padding
struct NetworkHeader
{
    public byte Version;
    public byte Type;
    public ushort Length;
    public uint SequenceId;
}
// Sizeof = 8 bytes

ReadOnlySpan<byte> raw = socket.ReceiveBuffer;
var header = MemoryMarshal.Read<NetworkHeader>(raw);
```

### Парсинг бинарного протокола — полный пример

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
readonly struct TradeMessage
{
    public readonly long Timestamp;    // 8 bytes
    public readonly int InstrumentId;  // 4 bytes
    public readonly decimal Price;     // 16 bytes
    public readonly int Quantity;      // 4 bytes
    // Total: 32 bytes
}

public static TradeMessage ParseTrade(ReadOnlySpan<byte> buffer)
{
    return MemoryMarshal.Read<TradeMessage>(buffer);
}

public static ReadOnlySpan<TradeMessage> ParseBatch(ReadOnlySpan<byte> buffer)
{
    return MemoryMarshal.Cast<byte, TradeMessage>(buffer);
}
```

> [!warning] Endianness
> `MemoryMarshal.Read` не конвертирует byte order. Если протокол big-endian, а CPU little-endian — нужен `BinaryPrimitives.ReverseEndianness()` или `ReadInt32BigEndian`.

### `[Auto]` vs `[Sequential]` vs `[Explicit]`

| Layout | Что |
|--------|-----|
| `Auto` (default для class) | CLR может реordered поля для оптимального packing |
| `Sequential` (default для struct) | Поля в порядке объявления |
| `Explicit` | Точное расположение через FieldOffset |

Для interop **всегда** Sequential или Explicit — Auto может изменить порядок.

---

## 11. Unsafe и pointer manipulation

```csharp
// fixed — пиннинг managed object
byte[] managed = new byte[1024];
unsafe
{
    fixed (byte* ptr = managed)
    {
        // ptr валиден внутри fixed блока
        // GC не двигает managed на время fixed
        *(int*)ptr = 42;
    }
}

// Unsafe.As — reinterpret cast без копирования
float f = 3.14f;
int bits = Unsafe.As<float, int>(ref f);  // 0x4048F5C3

// Unsafe.Add — pointer arithmetic для Span
ref byte start = ref MemoryMarshal.GetReference(span);
ref byte element = ref Unsafe.Add(ref start, index);

// Unsafe.SizeOf
int size = Unsafe.SizeOf<TradeMessage>();  // 32

// Unsafe.AreSame — сравнение references
bool same = Unsafe.AreSame(ref a, ref b);
```

### MemoryMarshal — продвинутые операции

```csharp
// Cast — reinterpret один Span как Span другого типа
Span<byte> bytes = ...;
Span<int> ints = MemoryMarshal.Cast<byte, int>(bytes);  // 4x меньше элементов

// AsBytes — typed Span как byte Span
Span<int> data = ...;
Span<byte> raw = MemoryMarshal.AsBytes(data);

// CreateSpan — Span из ref + length
ref int start = ref data[0];
Span<int> span = MemoryMarshal.CreateSpan(ref start, 100);

// CreateReadOnlySpan — для readonly структур
ReadOnlySpan<int> rspan = MemoryMarshal.CreateReadOnlySpan(in start, 100);
```

---

## 12. System.IO.Pipelines — high-perf I/O

### Зачем

`Stream` API имеет несколько проблем для high-performance:
- Нужно вручную управлять буфером (его размером, ростом, возвратом в пул)
- Нет backpressure — reader не знает что writer медленный
- Двойное копирование (Stream → buffer → consumer)

`System.IO.Pipelines` решает: единая memory pool + backpressure + zero-copy.

### Паттерн использования

```csharp
public static async Task ProcessFromSocketAsync(Socket socket)
{
    var pipe = new Pipe();
    
    Task writing = FillPipeAsync(socket, pipe.Writer);
    Task reading = ReadPipeAsync(pipe.Reader);
    
    await Task.WhenAll(reading, writing);
}

private static async Task FillPipeAsync(Socket socket, PipeWriter writer)
{
    while (true)
    {
        Memory<byte> memory = writer.GetMemory(sizeHint: 4096);
        int bytesRead = await socket.ReceiveAsync(memory, SocketFlags.None);
        if (bytesRead == 0) break;
        
        writer.Advance(bytesRead);  // commit что записали
        
        FlushResult result = await writer.FlushAsync();  // reader узнал
        if (result.IsCompleted) break;
    }
    
    await writer.CompleteAsync();
}

private static async Task ReadPipeAsync(PipeReader reader)
{
    while (true)
    {
        ReadResult result = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = result.Buffer;
        
        // Парсим сообщения
        while (TryParseMessage(ref buffer, out var message))
        {
            ProcessMessage(message);
        }
        
        reader.AdvanceTo(buffer.Start, buffer.End);
        
        if (result.IsCompleted) break;
    }
    
    await reader.CompleteAsync();
}

private static bool TryParseMessage(
    ref ReadOnlySequence<byte> buffer,
    out ReadOnlySequence<byte> message)
{
    var reader = new SequenceReader<byte>(buffer);
    
    if (reader.TryReadTo(out message, delimiter: (byte)'\n'))
    {
        buffer = reader.UnreadSequence;
        return true;
    }
    
    message = default;
    return false;
}
```

### ReadOnlySequence\<T\> — Span для multiple buffers

`ReadOnlySequence<T>` — последовательность segments. В отличие от `Span`/`Memory`, может представлять данные в **нескольких** массивах:

```csharp
// Один segment
var single = new ReadOnlySequence<byte>(buffer);

// Multiple segments
var s1 = new BufferSegment(buf1);
var s2 = s1.Append(buf2);
var multi = new ReadOnlySequence<byte>(s1, 0, s2, buf2.Length);

// Linear traversal
foreach (var memory in multi)
{
    Process(memory.Span);
}

// Random access — медленнее
byte b = multi.Slice(100, 1).FirstSpan[0];
```

`SequenceReader<T>` — высокоуровневое чтение из ReadOnlySequence:

```csharp
var reader = new SequenceReader<byte>(sequence);

if (reader.TryReadLittleEndian(out int length) &&
    reader.TryReadExact(length, out var payload))
{
    Process(payload);
}
```

### Kestrel использует Pipelines

Внутри ASP.NET Core/Kestrel — pipelines. Поэтому `HttpContext.Request.BodyReader` возвращает `PipeReader` напрямую:

```csharp
app.MapPost("/upload", async (HttpContext ctx) =>
{
    PipeReader reader = ctx.Request.BodyReader;
    
    while (true)
    {
        var result = await reader.ReadAsync();
        var buffer = result.Buffer;
        
        // Обработка чанков без аллокации полного byte[]
        ProcessChunk(buffer);
        
        reader.AdvanceTo(buffer.End);
        if (result.IsCompleted) break;
    }
});
```

---

## 13. Object Layout class vs struct

### Class в памяти

```
class MyObj { int A; long B; byte C; }

Layout на heap:
[Sync Block][Method Table][A: 4 bytes][padding 4][B: 8 bytes][C: 1][padding 7]
                          ↑ начало fields
Total: 16 (header) + 24 (fields) = 40 bytes
```

`new MyObj()` — аллокация на heap, 40 bytes.

### Struct в памяти

```csharp
struct MyStruct { int A; long B; byte C; }

// На стеке или внутри другого объекта:
[A: 4][padding 4][B: 8][C: 1][padding 7]
// 24 bytes, без header

// В массиве MyStruct[] — компактно подряд, без header'ов
```

### Размер class vs struct

```csharp
class C { int X; }     // ~24 bytes (16 header + 4 int + 4 padding)
struct S { int X; }    // 4 bytes
```

Массив 1M элементов:
- `C[]` — 24 MB (если бы все объекты выделены) + сами объекты по 24 байта × 1M = +24 MB → **48 MB**
- `S[]` — 4 MB

### Когда struct, когда class

| | struct | class |
|--|--------|-------|
| Размер | ≤ 16 байт обычно | Любой |
| Allocation | Stack или inline в другом объекте | Heap |
| Copy semantics | By value (копируется) | By reference |
| GC pressure | Нет | Да |
| Identity | Нет (== по value) | Да |
| Hashable | По полям | По reference |

**Правило:** struct если size ≤ 16-24 byte и behave-as-value.

### readonly struct

```csharp
public readonly struct Point
{
    public readonly int X;
    public readonly int Y;
    public Point(int x, int y) { X = x; Y = y; }
}
```

Преимущества:
- JIT знает: defensive copy не нужен при `in` параметре
- При вызове методов компилятор уверен что не модифицирует
- Помогает компилятору с оптимизациями

```csharp
public void Process(in Point p)  // by ref readonly
{
    // Без `readonly struct` — JIT делает defensive copy перед каждым доступом!
    Console.WriteLine(p.X);
    Console.WriteLine(p.Y);
}
```

### `ref` readonly параметры (C# 12)

```csharp
public void Process(ref readonly Point p)  // явное readonly ref
{
    // p — ref, но read-only
}
```

Полезно для больших structs — передача по ref, но без возможности модификации.

---

## 14. Boxing / Unboxing — частый pitfall

### Что такое boxing

```csharp
int x = 42;             // value type на stack
object obj = x;         // BOXING — int копируется в новый объект на heap
int y = (int)obj;       // UNBOXING — копирование обратно
```

Boxing создаёт **новый object на heap** каждый раз. Дорого.

### Hidden boxing

```csharp
// ❌ struct в interface — boxing
struct MyStruct : IDisposable { public void Dispose() { } }

void DoWork(IDisposable d) => d.Dispose();
var s = new MyStruct();
DoWork(s);  // BOXING (s → object с MyStruct внутри)

// ❌ struct.GetHashCode без override — boxing для object.GetHashCode()
struct Point { int X; int Y; }
Dictionary<Point, string> dict;  // Point.GetHashCode() boxes! (если не override)

// ❌ string format
struct Money { decimal Value; }
var m = new Money();
Console.WriteLine($"{m}");  // BOXING если нет override ToString()
```

### Avoiding boxing

```csharp
// ✅ Generic constraint
public void DoWork<T>(T disposable) where T : IDisposable
{
    disposable.Dispose();  // нет boxing — JIT specializes для T
}

// ✅ Override методы
public override int GetHashCode() => HashCode.Combine(X, Y);
public override string ToString() => $"({X}, {Y})";

// ✅ IEquatable<T>
public struct Point : IEquatable<Point>
{
    public bool Equals(Point other) => X == other.X && Y == other.Y;
    public override bool Equals(object? obj) => obj is Point p && Equals(p);
    public override int GetHashCode() => HashCode.Combine(X, Y);
}
```

---

## 15. Custom Disposable structs

```csharp
// ref struct ScopedResource — освобождается автоматически в using
public ref struct ScopedRented
{
    private byte[]? _array;
    public Span<byte> Span { get; }

    public ScopedRented(int size)
    {
        _array = ArrayPool<byte>.Shared.Rent(size);
        Span = _array.AsSpan(0, size);
    }

    public void Dispose()
    {
        if (_array != null)
        {
            ArrayPool<byte>.Shared.Return(_array);
            _array = null;
        }
    }
}

// Использование
using var rented = new ScopedRented(8192);
ProcessData(rented.Span);
// автоматический Return при выходе из scope
```

---

## Cheat Sheet: выбор подхода

```
Нужна ли память?
  │
  ├── < 1 KB, синхронно → stackalloc + Span<T>
  │
  ├── Большой буфер, переиспользуемый → ArrayPool<T>.Shared.Rent()
  │
  ├── Async контекст → Memory<T>
  │
  ├── I/O буфер, pinned для ОС → GC.AllocateArray<T>(pinned: true)
  │
  ├── Interop с native → NativeMemory.Alloc + Span<T>
  │
  ├── Network/file streaming → System.IO.Pipelines
  │
  ├── SIMD batch операции → Vector<T> / Vector256<T>
  │
  ├── Поиск любого из набора символов → SearchValues<T>
  │
  ├── UTF-8 без аллокаций → "..."u8 + Utf8Parser/Formatter
  │
  ├── Fixed-size buffer без unsafe → InlineArray<N>
  │
  ├── Парсинг бинарного формата → MemoryMarshal.Read<T> + StructLayout
  │
  └── Обычный объект → new T[] (пусть GC разбирается)
```

---

## Best Practices

- **Используй Span/Memory только на hot path** — на cold path читаемость важнее
- **stackalloc проверяй размер** — fallback на heap если > 1 KB
- **ArrayPool — не забывай Return** — иначе утечка пула
- **`u8` literals для constants** — нет runtime-аллокации
- **SearchValues для много-символного поиска** — 2-10x быстрее
- **`readonly struct` для small data** — JIT не делает defensive copy
- **`in` параметры с `readonly struct`** — pass by ref без копии
- **StructLayout для interop** — Sequential или Explicit, не Auto
- **Cache line awareness** — hot fields в первые 64 байта
- **Pipelines для streaming I/O** — лучше чем Stream + buffer вручную
- **Vector\<T\> для портативного SIMD** — fallback если не supported
- **Профилируй прежде чем оптимизировать** — `BenchmarkDotNet` обязательно

---

## См. также

- [GC, LOH и POH](gc-memory.md)
- [Concurrency и атомарность](concurrency-atomics.md)
- [Performance](../Performance/performance.md)
- [HFT/Low-Latency](../Performance/hft-low-latency.md) — Channels, ring buffers, latency budgets
- [Типы и память](../CSharp/types-and-memory.md)
- [Compilation/JIT](compilation-jit.md) — SIMD intrinsics ASM-уровень
- [IPC: Named Pipes & gRPC](../Infrastructure/ipc-named-pipes-grpc.md) — MMF ring buffer

## Reading list

- **Stephen Toub — Performance Improvements in .NET 8/9/10** — годовые обзоры на devblogs.microsoft.com
- **Adam Sitnik — Span & Memory blog** — adamsitnik.com
- **Konrad Kokosa — Pro .NET Performance** (книга)
- **Federico Lois — High-Performance .NET талки** (NDC, Build)
- **Microsoft Docs — System.IO.Pipelines** — learn.microsoft.com
- **Microsoft Docs — Hardware Intrinsics** — learn.microsoft.com/dotnet/standard/simd
- **Bartosz Adamczewski — performance posts** — leveluppp.com
- **GitHub: dotnet/runtime samples — System.Runtime.Intrinsics tests** — examples реальных SIMD алгоритмов
