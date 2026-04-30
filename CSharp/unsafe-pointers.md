---
tags: [csharp, unsafe, pointers, fixed, stackalloc, ref-struct, interop, hot-paths, senior]
level: Senior
date: 2026-04-30
---

# Unsafe и Pointers — низкоуровневый C#

> **`unsafe`, `fixed`, `stackalloc`, pointer arithmetic, `ref struct`**. Для interop с native кодом и hot paths где надо избежать GC. Senior топик — мало кто реально пишет, но **многие интервьюеры спрашивают**.

---

## Что это, зачем и когда

### Зачем unsafe

C# — managed язык. GC, bounds checking, type safety — всё за тебя. **Но иногда** этого мало:

- **Interop** с native libs (C/C++ DLL)
- **Hot path** где bounds checking стоит cycles
- **Direct memory manipulation** — image processing, network protocols
- **Kernel bypass** — networking, GPUs

### Trade-off

```
Managed C#         vs      Unsafe C#
───────────────             ───────────────
Type safety       ✓               ✗ (можно повредить память)
Bounds check      ✓               ✗ (можно выйти за границы)
GC                ✓               ⚠ (нужно pin'ить)
Cross-platform    ✓               ⚠ (зависит от platform)
Compile flag      Нет             /unsafe
Performance       Good            Maximum (10-100x в hot path)
```

### Когда применять

✅ **Используй когда:**
- Native interop (P/Invoke, COM)
- Image / audio / video processing pixel-by-pixel
- Network protocols (parse binary headers)
- HFT / sub-millisecond latency
- Cryptography (constant-time operations)

❌ **НЕ используй когда:**
- Просто чтобы "быстрее" — измерь сначала с Span/Memory
- Cross-platform code критичен
- Junior team без supervision
- Code должен быть verifiable

> [!warning] Modern alternative — `Span<T>` и `ref struct`
> 
> **80% случаев unsafe заменяется на `Span<T>` без потерь по перформансу!** Используй unsafe только когда Span недостаточно.

См. [[../Runtime/span-layout|Span\<T\> и layout]] и [[../Runtime/interop-pinvoke|Interop & PInvoke]].

---

## 1. Включаем unsafe

### В csproj

```xml
<PropertyGroup>
  <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
</PropertyGroup>
```

### В коде

```csharp
// Метод
public unsafe void Process(byte* data, int length) { /* ... */ }

// Блок
public void Method()
{
    unsafe
    {
        int x = 5;
        int* p = &x;
        *p = 10;
        Console.WriteLine(x); // 10
    }
}

// Класс/struct
public unsafe struct Buffer
{
    public byte* Data;
    public int Length;
}
```

---

## 2. Указатели — basics

### Объявление и операции

```csharp
unsafe
{
    int x = 42;
    int* p = &x;        // получить адрес
    
    Console.WriteLine(*p);     // 42 — dereference
    *p = 100;                   // запись через указатель
    Console.WriteLine(x);      // 100
    
    // Pointer arithmetic
    int[] arr = { 1, 2, 3, 4, 5 };
    fixed (int* start = arr)
    {
        int* current = start;
        for (int i = 0; i < arr.Length; i++)
        {
            Console.WriteLine(*current);
            current++;  // двигаем на sizeof(int) = 4 bytes
        }
    }
}
```

### Что разрешено

```csharp
T*              // pointer to T (только blittable types!)
*p              // dereference
&x              // address of
p[i]            // pointer indexing
p++  p--  p+1   // arithmetic
p1 == p2        // comparison
(byte*)p        // cast pointers
sizeof(T)       // размер типа в байтах
```

### Blittable types (только эти)

```csharp
// ✅ Pointer-friendly
byte, sbyte, short, ushort, int, uint, long, ulong, float, double, decimal,
bool (но 1 byte! не как managed bool),
struct из blittable полей

// ❌ Нельзя
string, object, любой class, struct с reference полями
int*, byte* как поле struct без pinning
```

---

## 3. `fixed` — pinning managed memory

### Зачем

Managed objects могут быть **перемещены GC'ом**. Если у тебя pointer на managed array — после move pointer указывает на мусор.

```csharp
// ❌ ОПАСНО — GC может переместить arr
int[] arr = { 1, 2, 3 };
unsafe
{
    int* p = (int*)/* как-то взять адрес? */;
    DoSomething(p);  // может crash после GC!
}

// ✅ fixed — закрепляет на время блока
unsafe
{
    fixed (int* p = arr)
    {
        // arr не двинется пока выполняется блок
        DoSomething(p);
    }
    // После закрытия блока — pointer invalid
}
```

### Несколько fixed

```csharp
fixed (byte* src = sourceArray, dst = destArray)
{
    Buffer.MemoryCopy(src, dst, destArray.Length, sourceArray.Length);
}
```

### Pin строки

```csharp
string s = "Hello";
fixed (char* p = s)
{
    // p указывает на UTF-16 chars
    for (int i = 0; i < s.Length; i++)
        Console.WriteLine(*(p + i));
}
```

---

## 4. `stackalloc` — память на стеке

### Зачем

GC отвечает за heap. Stack — выделяется и освобождается **автоматически** при выходе из функции, **никаких аллокаций для GC**.

```csharp
public unsafe void ProcessSmall(int n)
{
    // Heap allocation — GC pressure
    int[] arr = new int[n];
    
    // Stack allocation — instant cleanup
    int* arr = stackalloc int[n];
    
    for (int i = 0; i < n; i++)
        arr[i] = i * i;
}
```

### Modern syntax — Span\<T\>

```csharp
// Без unsafe!
public void ProcessSmall(int n)
{
    Span<int> arr = stackalloc int[n];
    
    for (int i = 0; i < n; i++)
        arr[i] = i * i;
}

// Это не unsafe, и работает на .NET Core 2.1+
```

> [!info] Modern way
> `Span<int> = stackalloc int[n]` — не требует `unsafe`, type-safe, bounds-checked. **Используй это вместо raw pointers** в 90% случаев.

### Размер ограничен!

```csharp
// ❌ Stack overflow!
Span<byte> huge = stackalloc byte[10_000_000];

// Стек обычно 1 MB. Stackalloc — макс ~1 KB safe.
// Для большего — ArrayPool или heap.

// ✅ Hybrid pattern
Span<byte> buf = size <= 256
    ? stackalloc byte[256]
    : new byte[size];  // или ArrayPool
```

См. [[memory-pooling|Memory Pooling]] и [[../Runtime/span-layout|Span layout]].

---

## 5. Case Study #1 — Fast string parsing

### Задача

Parse `"123,456,789"` в `int[]`. 10K вызовов/сек. С `Split` + `Parse` — слишком много allocations.

### ❌ Naive

```csharp
public int[] Parse(string s)
{
    return s.Split(',').Select(int.Parse).ToArray();
    // Allocations: string[] from Split, IEnumerable, int[]
    // ~120 bytes per call × 10K calls/sec = 1.2 MB/sec
}
```

### ✅ Span + stackalloc

```csharp
public void Parse(ReadOnlySpan<char> input, Span<int> output, out int count)
{
    count = 0;
    int start = 0;
    
    for (int i = 0; i <= input.Length; i++)
    {
        if (i == input.Length || input[i] == ',')
        {
            output[count++] = int.Parse(input.Slice(start, i - start));
            start = i + 1;
        }
    }
}

// Use
public int Sum(string s)
{
    Span<int> nums = stackalloc int[16];  // max 16 numbers
    Parse(s.AsSpan(), nums, out int count);
    
    int sum = 0;
    for (int i = 0; i < count; i++) sum += nums[i];
    return sum;
}
```

**Результат:**

| Approach | Time | Allocations |
|----------|------|-------------|
| Split + Parse | 850 ns | 120 bytes |
| Span + stackalloc | 95 ns | 0 bytes |

**9x speedup, 0 allocations.**

---

## 6. Case Study #2 — Image processing (grayscale)

### Задача

Convert 4K image (3840×2160 RGB) в grayscale. Per-pixel операция.

### ❌ Naive — managed

```csharp
public byte[] ToGrayscale(byte[] rgb)
{
    var result = new byte[rgb.Length / 3];
    
    for (int i = 0; i < rgb.Length; i += 3)
    {
        byte r = rgb[i];
        byte g = rgb[i + 1];
        byte b = rgb[i + 2];
        result[i / 3] = (byte)(r * 0.3 + g * 0.59 + b * 0.11);
    }
    
    return result;
}

// 4K image: ~25 MB input, ~8 MB output
// Time: 80 ms, GC pause: 5-10 ms
```

### ✅ Unsafe — pointers

```csharp
public unsafe void ToGrayscale(byte* rgb, byte* gray, int pixelCount)
{
    byte* src = rgb;
    byte* dst = gray;
    
    for (int i = 0; i < pixelCount; i++)
    {
        // No bounds check — на 30% быстрее
        // r * 77 + g * 150 + b * 29  (integer math, ÷ 256 ≈ * 0.3, * 0.59, * 0.11)
        int gray_val = (src[0] * 77 + src[1] * 150 + src[2] * 29) >> 8;
        *dst = (byte)gray_val;
        
        src += 3;
        dst++;
    }
}

// Use
byte[] rgb = LoadImage(...);
byte[] gray = new byte[rgb.Length / 3];

unsafe
{
    fixed (byte* pRgb = rgb, pGray = gray)
    {
        ToGrayscale(pRgb, pGray, rgb.Length / 3);
    }
}

// Time: 18 ms — 4.4x faster
```

### ✅✅ Span — managed но fast

```csharp
public void ToGrayscale(ReadOnlySpan<byte> rgb, Span<byte> gray)
{
    int pixelCount = rgb.Length / 3;
    
    for (int i = 0; i < pixelCount; i++)
    {
        int idx = i * 3;
        int gray_val = (rgb[idx] * 77 + rgb[idx + 1] * 150 + rgb[idx + 2] * 29) >> 8;
        gray[i] = (byte)gray_val;
    }
}

// Time: 22 ms — 3.6x faster, без unsafe!
// JIT eliminate bounds checks для предсказуемых паттернов
```

### Benchmark на 4K image

| Method | Time | Allocations | Safety |
|--------|------|-------------|--------|
| Managed naive | 80 ms | 8 MB heap | ✓ |
| Span | 22 ms | 8 MB heap (output) | ✓ |
| Unsafe pointers | 18 ms | 8 MB heap (output) | ✗ |
| Unsafe + SIMD | 5 ms | 8 MB heap (output) | ✗ |

**Lesson:** Span даёт 80% benefit unsafe без рисков. Только если нужен максимум — unsafe + SIMD.

См. [[../Performance/optimization-patterns|Optimization]] и `System.Numerics.Vector<T>`.

---

## 7. Case Study #3 — Network protocol parsing

### Задача

TCP сервер парсит binary header (16 bytes): version (1B), flags (1B), length (4B BE), checksum (8B), reserved (2B).

### ❌ Naive

```csharp
public class Header
{
    public byte Version;
    public byte Flags;
    public uint Length;
    public ulong Checksum;
}

public Header Parse(byte[] data)
{
    var h = new Header();
    h.Version = data[0];
    h.Flags = data[1];
    h.Length = (uint)((data[2] << 24) | (data[3] << 16) | (data[4] << 8) | data[5]);
    h.Checksum = BitConverter.ToUInt64(data, 6);  // alloc!
    return h;
}
```

### ✅ Unsafe — direct cast

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public unsafe struct Header
{
    public byte Version;
    public byte Flags;
    public uint Length;       // big-endian!
    public ulong Checksum;
    public ushort Reserved;
}

public unsafe Header ParseHeader(ReadOnlySpan<byte> data)
{
    fixed (byte* p = data)
    {
        Header h = *(Header*)p;
        // Convert BE → LE для multi-byte fields
        h.Length = BinaryPrimitives.ReverseEndianness(h.Length);
        h.Checksum = BinaryPrimitives.ReverseEndianness(h.Checksum);
        return h;
    }
}

// Time: 4 ns vs 35 ns — 9x speedup
```

### ✅✅ MemoryMarshal — без unsafe!

```csharp
public Header ParseHeader(ReadOnlySpan<byte> data)
{
    Header h = MemoryMarshal.Read<Header>(data);
    h.Length = BinaryPrimitives.ReverseEndianness(h.Length);
    h.Checksum = BinaryPrimitives.ReverseEndianness(h.Checksum);
    return h;
}

// Same performance, no unsafe — best!
```

См. [[../Runtime/interop-pinvoke|Interop & PInvoke]] и [[../Infrastructure/ipc-named-pipes-grpc|IPC]].

---

## 8. `ref struct` — stack-only types

### Зачем

Гарантировать что struct **никогда не попадёт на heap** (no boxing, no fields в class).

```csharp
public ref struct StackBuffer
{
    private Span<byte> _buffer;
    
    public StackBuffer(Span<byte> buffer) => _buffer = buffer;
    
    public void Write(byte b) { /* ... */ }
}

// ❌ Compile errors — гарантия что не попадёт на heap
class Holder { public StackBuffer Buf; }   // ❌
StackBuffer[] arr = new StackBuffer[10];   // ❌
async Task DoAsync() { var buf = new StackBuffer(); }  // ❌ async makes it field

// ✅ ОК — only on stack
public void Method()
{
    Span<byte> mem = stackalloc byte[256];
    var buf = new StackBuffer(mem);
    Process(buf);
}
```

### Built-in ref structs

```csharp
Span<T>           // ref struct
ReadOnlySpan<T>   // ref struct
Utf8JsonReader    // ref struct
ValueStringBuilder // ref struct in System.Text (internal)
```

### Custom ref struct

```csharp
public ref struct ValueListBuilder<T>
{
    private Span<T> _span;
    private int _count;
    
    public ValueListBuilder(Span<T> initialSpan)
    {
        _span = initialSpan;
        _count = 0;
    }
    
    public void Add(T item)
    {
        if (_count == _span.Length) GrowIfNeeded();
        _span[_count++] = item;
    }
    
    public ReadOnlySpan<T> AsSpan() => _span[.._count];
    
    private void GrowIfNeeded()
    {
        // ... realloc logic если нужно ...
    }
}

// Use
public string BuildString(IEnumerable<int> items)
{
    Span<char> initial = stackalloc char[256];
    var builder = new ValueListBuilder<char>(initial);
    
    foreach (var item in items)
    {
        Span<char> buf = stackalloc char[16];
        item.TryFormat(buf, out int written);
        for (int i = 0; i < written; i++) builder.Add(buf[i]);
        builder.Add(',');
    }
    
    return new string(builder.AsSpan());
}
```

См. [[../Runtime/span-layout|Span\<T\> и layout]].

---

## 9. Case Study #4 — Cryptographic constant-time compare

### Задача

Compare two byte arrays — но **constant time** (защита от timing attacks).

### ❌ Naive — ранний выход

```csharp
public bool CompareBad(byte[] a, byte[] b)
{
    if (a.Length != b.Length) return false;
    for (int i = 0; i < a.Length; i++)
        if (a[i] != b[i]) return false;  // ❌ timing leak!
    return true;
}

// Attacker measures time → находит первый разный byte → cracks token
```

### ✅ Constant-time

```csharp
public unsafe bool CompareConstantTime(ReadOnlySpan<byte> a, ReadOnlySpan<byte> b)
{
    if (a.Length != b.Length) return false;
    
    fixed (byte* pa = a, pb = b)
    {
        int diff = 0;
        for (int i = 0; i < a.Length; i++)
        {
            diff |= pa[i] ^ pb[i];   // XOR — 0 если равны
        }
        return diff == 0;
    }
}

// Время одинаковое независимо где разница → no timing leak
```

> [!info] Modern alternative
> `CryptographicOperations.FixedTimeEquals(a, b)` — встроено в .NET 5+. Без unsafe.

---

## 10. Case Study #5 — P/Invoke с native DLL

### Задача

Вызвать C функцию `int sum_array(int* arr, int len)` из native DLL.

### Реализация

```csharp
public static class NativeMath
{
    [DllImport("native_math.dll", CallingConvention = CallingConvention.Cdecl)]
    private static extern unsafe int sum_array(int* arr, int len);
    
    public static int Sum(int[] data)
    {
        unsafe
        {
            fixed (int* p = data)
            {
                return sum_array(p, data.Length);
            }
        }
    }
}

// Use
int[] arr = { 1, 2, 3, 4, 5 };
int sum = NativeMath.Sum(arr);  // 15
```

### Без `unsafe` — modern API

```csharp
[DllImport("native_math.dll")]
private static extern int sum_array(IntPtr arr, int len);

// Используем GCHandle для pinning
public static int Sum(int[] data)
{
    GCHandle handle = GCHandle.Alloc(data, GCHandleType.Pinned);
    try
    {
        return sum_array(handle.AddrOfPinnedObject(), data.Length);
    }
    finally
    {
        handle.Free();
    }
}
```

См. [[../Runtime/interop-pinvoke|Interop & PInvoke]].

---

## 11. Case Study #6 — High-throughput log writer

### Задача

Write 1M log entries/sec. Каждый — 100-200 bytes. Allocation-free path обязателен.

### Решение — ref struct + stackalloc + Direct write

```csharp
public ref struct LogEntryBuilder
{
    private Span<byte> _buffer;
    private int _position;
    
    public LogEntryBuilder(Span<byte> buffer)
    {
        _buffer = buffer;
        _position = 0;
    }
    
    public void WriteString(ReadOnlySpan<char> s)
    {
        int written = Encoding.UTF8.GetBytes(s, _buffer[_position..]);
        _position += written;
    }
    
    public void WriteByte(byte b) => _buffer[_position++] = b;
    
    public void WriteInt(int value)
    {
        Utf8Formatter.TryFormat(value, _buffer[_position..], out int written);
        _position += written;
    }
    
    public ReadOnlySpan<byte> AsSpan() => _buffer[.._position];
}

public class FastLogger
{
    private readonly Stream _output;
    
    public void Log(string level, ReadOnlySpan<char> message, int code)
    {
        Span<byte> buffer = stackalloc byte[256];
        var builder = new LogEntryBuilder(buffer);
        
        builder.WriteByte((byte)'[');
        builder.WriteString(level);
        builder.WriteByte((byte)']');
        builder.WriteByte((byte)' ');
        builder.WriteString(message);
        builder.WriteByte((byte)' ');
        builder.WriteByte((byte)'(');
        builder.WriteInt(code);
        builder.WriteByte((byte)')');
        builder.WriteByte((byte)'\n');
        
        _output.Write(builder.AsSpan());
    }
}

// 1M log calls/sec:
//   Allocations: 0
//   Throughput: 5x vs StringBuilder
```

См. [[../AspNetCore/logging-observability|Logging]] и [[../Performance/hft-low-latency|HFT]].

---

## 12. Когда `unsafe` оправдан, а когда нет

### ✅ Действительно нужен unsafe

```csharp
// Native interop с raw pointers
[DllImport("kernel32.dll")]
public static extern unsafe bool ReadFile(IntPtr h, byte* buffer, int len, out int read, IntPtr overlapped);

// SIMD без built-in support
public unsafe void Sum(float* a, float* b, float* dst, int len) { /* AVX intrinsics */ }

// Custom memory allocators (rare!)
public unsafe class CustomHeap { /* ... */ }
```

### ❌ Где НЕ нужен — есть managed альтернатива

| Unsafe code | Managed альтернатива |
|-------------|----------------------|
| `byte* p = ...` для buffer | `Span<byte>` |
| `stackalloc int[n]` raw | `Span<int> s = stackalloc int[n]` |
| `*(Header*)p = h` для serialize | `MemoryMarshal.Write<Header>(span, ref h)` |
| `BitConverter.ToInt32(arr, idx)` через pointer | `MemoryMarshal.Read<int>(span)` |
| `Buffer.MemoryCopy` | `Span<T>.CopyTo` или `MemoryMarshal.AsBytes` |
| Custom string parse | `ReadOnlySpan<char>` + `int.Parse(span)` |
| Pointer arithmetic for iteration | `for + Span indexing` (JIT removes bounds check для предсказуемых) |

См. [[../Runtime/span-layout|Span\<T\> детально]].

---

## 13. Common Pitfalls

### 1. Use-after-pinning expired

```csharp
// ❌ Pointer выходит за scope fixed
unsafe
{
    int* p;
    fixed (int* fp = arr) { p = fp; }
    *p = 5;  // 💥 GC мог переместить arr — pointer invalid
}

// ✅ Используй ТОЛЬКО внутри fixed block
unsafe
{
    fixed (int* p = arr)
    {
        *p = 5;  // safe
    }
}
```

### 2. Stack overflow от stackalloc

```csharp
// ❌ Размер из user input!
public void Method(int n)
{
    Span<byte> s = stackalloc byte[n];  // если n = 10M → stack overflow → process crash
}

// ✅ Hybrid pattern
public void Method(int n)
{
    Span<byte> s = n <= 1024
        ? stackalloc byte[1024]
        : new byte[n];  // или ArrayPool
}
```

### 3. Buffer overflow

```csharp
unsafe
{
    byte* p = stackalloc byte[10];
    for (int i = 0; i < 100; i++)  // ❌ выход за границы!
        p[i] = 0;  // повреждение stack
}
```

### 4. Endian assumptions

```csharp
unsafe
{
    int x = 1;
    byte* p = (byte*)&x;
    Console.WriteLine(*p);  // На x86 = 1 (little-endian), на ARM может быть 0!
}

// ✅ BinaryPrimitives для cross-platform
int x = BinaryPrimitives.ReadInt32LittleEndian(span);
```

### 5. Struct alignment / padding

```csharp
// ❌ Без атрибута компилятор может добавить padding
public struct Header
{
    public byte Version;    // 1 byte
    public uint Length;     // 4 bytes — но padded! offset = 4, не 1
}
sizeof(Header) // = 8 not 5!

// ✅ Pack = 1 убирает padding (но slow access на ARM)
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Header
{
    public byte Version;
    public uint Length;
}
sizeof(Header) // = 5
```

### 6. Pinning + async = bug

```csharp
// ❌ async + fixed = compile error (правильно!)
unsafe async Task Bad()
{
    fixed (int* p = arr)  // ❌ нельзя fixed через await
    {
        await SomethingAsync();  // pointer invalid после await!
    }
}

// ✅ GCHandle.Alloc Pinned для long-living
GCHandle handle = GCHandle.Alloc(arr, GCHandleType.Pinned);
try
{
    IntPtr p = handle.AddrOfPinnedObject();
    await SomethingAsync(p);
}
finally { handle.Free(); }
```

### 7. Не использовать `[DoesNotReturn]` для exit-like

Не относится напрямую к unsafe но часто рядом. Помогает analyzer.

### 8. Performance: pinning = GC penalty

```csharp
// Долгое pinning ⇒ GC не может переместить object ⇒ fragmentation
fixed (byte* p = veryLongLivedArray)
{
    while (running) { /* долгие минуты */ }
    // ⚠️ GC не может move arr → memory fragmentation
}
```

**Лечение:** короткие fixed scopes; `GCHandleType.Pinned` только когда нужен long-lived pin (с явным Free).

### 9. Mixing unsafe и safe code

```csharp
public unsafe int* GetPointer() => stackalloc int[10];

// ❌ Возврат stack-allocated pointer!
int* p = GetPointer();  // 💥 stack frame уничтожен
*p = 5;  // undefined behavior
```

`stackalloc` живёт только в текущем frame. Не возвращай.

### 10. Cross-platform issues

```csharp
// На Windows — sizeof(IntPtr) = 8 (x64)
// На Linux ARM — sizeof(IntPtr) = 8
// На Windows 32-bit — sizeof(IntPtr) = 4

unsafe
{
    long size = sizeof(IntPtr);  // varies!
}

// ✅ Используй nint / nuint (C# 9+)
nint pointer;  // platform-specific size, but type-safe
```

---

## 14. Best Practices

### Когда писать unsafe

✅ **OK:**
- Native interop (P/Invoke)
- Image / audio processing pixel-by-pixel
- Cryptography constant-time ops
- HFT / hot paths после profiling
- Custom serializers

❌ **AVOID:**
- "Просто чтобы быстрее" без измерений
- Где Span работает
- Cross-platform code критично
- Junior team без supervision
- Public APIs (verifiability важна)

### Code organization

```csharp
// ✅ Изолируй unsafe в маленькие модули
public class FastImageProcessor
{
    public void ProcessGrayscale(ReadOnlySpan<byte> input, Span<byte> output)
    {
        // Public API — managed Span
        ProcessUnsafe(input, output);
    }
    
    private unsafe void ProcessUnsafe(ReadOnlySpan<byte> input, Span<byte> output)
    {
        fixed (byte* pIn = input, pOut = output)
        {
            ProcessRaw(pIn, pOut, input.Length);
        }
    }
    
    private unsafe void ProcessRaw(byte* input, byte* output, int len)
    {
        // Hot path
    }
}
```

### Profile before & after

```csharp
[MemoryDiagnoser]
[Benchmark]
public void Managed() => Process_Managed(data);

[Benchmark]
public unsafe void Unsafe() => Process_Unsafe(data);

// BenchmarkDotNet results:
//   Managed:  500 μs, 1.2 KB allocations
//   Unsafe:   150 μs, 0 allocations
//   → Worth it
```

См. [[../Performance/performance|BenchmarkDotNet]].

---

## 15. Cheat sheet

| Сценарий | Solution | Unsafe нужен? |
|----------|----------|---------------|
| Buffer для I/O | `Span<byte> = stackalloc byte[256]` | Нет |
| Parse binary header | `MemoryMarshal.Read<T>(span)` | Нет |
| Image pixel ops | `Span<byte>` + индексирование | Обычно нет |
| Native lib call | P/Invoke + `fixed` или GCHandle | Да |
| SIMD operations | `Vector<T>` или intrinsics | Чаще да |
| Constant-time compare | `CryptographicOperations.FixedTimeEquals` | Нет |
| Custom struct serialize | `MemoryMarshal.Cast<byte, T>` | Нет |
| Network protocol parse | `BinaryPrimitives` + `Span` | Нет |
| Convert short ↔ bytes | `BitConverter` или `BinaryPrimitives` | Нет |
| Custom heap allocator | `NativeMemory.Alloc` (.NET 6+) | Чаще нет |
| Stack-only struct | `ref struct` | Нет (но требует осторожность) |

---

## 16. Decision tree

```
Нужно высокая производительность?
│
├── Уже измерил с Span/Memory?
│   ├── Нет → Сначала Span/Memory + profile
│   └── Да → продолжай
│
├── Span/Memory недостаточно?
│   ├── Нет → ОСТАНОВИСЬ. Span/MemoryMarshal/BinaryPrimitives хватит
│   └── Да →
│
├── Native interop нужен?
│   → unsafe + fixed + P/Invoke (justify)
│
├── SIMD / pointer arithmetic?
│   → unsafe + intrinsics (justify)
│
└── Иначе → reconsider, скорее всего Span хватит
```

---

## См. также

- [[../Runtime/span-layout|Span\<T\> и layout]] — modern alternative для 80% случаев
- [[memory-pooling|Memory Pooling]] — без unsafe
- [[../Runtime/interop-pinvoke|Interop & PInvoke]] — native libs
- [[types-and-memory|Types & Memory]] — managed memory model
- [[../Runtime/gc-memory|GC и память]] — pinning impact
- [[../Performance/hft-low-latency|HFT]] — где unsafe реально нужен
- [[../Performance/optimization-patterns|Optimization Patterns]]

## Reading list

- **Microsoft Docs — unsafe** — learn.microsoft.com/dotnet/csharp/language-reference/unsafe-code
- **Microsoft Docs — fixed statement** — learn.microsoft.com/dotnet/csharp/language-reference/statements/fixed
- **Microsoft Docs — stackalloc** — learn.microsoft.com/dotnet/csharp/language-reference/operators/stackalloc
- **Stephen Toub — Writing high-performance .NET code** — devblogs.microsoft.com
- **Federico Lois — High Performance .NET** (книга)
- **Konrad Kokosa — Pro .NET Memory Management** (книга)
- **Adam Sitnik blog** — adamsitnik.com (low-level perf)
- **EgorBo blog** — egorbogatov.com (JIT internals)
