---
tags: [csharp, memory-pooling, senior, performance, arraypool, span, memorypool, gc, allocation]
level: Senior
date: 2026-05-09
---

# Memory Pooling — переиспользование буферов

> **Сокращение allocation pressure через reuse: `ArrayPool<T>`, `MemoryPool<T>`, `ObjectPool<T>`, `Span<T>`/`Memory<T>` для zero-copy.** Когда 90%+ allocations — short-lived buffers (parsers, serializers, network I/O), pooling даёт 10-100x throughput. Закрывает пробел: «знаю про GC, не понимаю когда `ArrayPool.Shared.Rent` оправдан и почему `Span<T>` может избежать allocation вообще».

---

## 0. Как читать

Если впервые — раздел 1→3 (mental model + ArrayPool basics). Если уже используешь pooling — раздел 5 (Span/Memory zero-copy), 6 (custom ObjectPool). Production guidance — раздел 9 (best practices), 11 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Allocation pressure problem

```csharp
// Hot path — миллионы вызовов в секунду
public string Process(string input)
{
    byte[] buffer = new byte[8192];   // allocation per call!
    Encoding.UTF8.GetBytes(input, buffer);
    return Convert.ToBase64String(buffer);
}
```

Каждый `new byte[8192]` — heap allocation. GC должен потом освободить:
- **Gen 0 collection** — fast (~1ms), но frequent.
- **Gen 1/2** — slower при promotion.
- **LOH** (Large Object Heap, > 85,000 bytes) — Gen 2 only, fragmentation.

В hot path — **GC pauses становятся bottleneck**.

### 1.2. Pooling — переиспользование

```csharp
public string Process(string input)
{
    byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);
    try
    {
        Encoding.UTF8.GetBytes(input, buffer);
        return Convert.ToBase64String(buffer, 0, input.Length);
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

Buffer **rented** (взят из пула), used, **returned**. Pool reuses arrays между calls — нет allocation.

### 1.3. Когда pooling justified

```
✅ Pool когда:
  - Buffer > ~1 KB (small allocation cheap, pooling overhead не оправдывается)
  - Hot path (миллионы calls/sec)
  - Buffer lifetime короткое и predictable
  - Profiler shows allocation pressure

❌ Не pool когда:
  - Tiny buffers (< 1KB) — allocate напрямую
  - Long-lived buffers (cache them yourself)
  - Rare path (1 раз в секунду — экономия meaningless)
  - Multi-threaded shared state (sync overhead)
```

### 1.4. Главное правило

```
ArrayPool<T>.Shared — для byte[], char[], int[] etc.
MemoryPool<T> — для Memory<T> с custom-shaped buffers
ObjectPool<T> (Microsoft.Extensions.ObjectPool) — для StringBuilder, custom объекты
Span<T> / Memory<T> — slice без copy
stackalloc — для small buffers (< 1KB) на stack

Always use try/finally для Return — leaks без него.
ArrayPool.Return clearArray=true — если был sensitive data.
```

### 1.5. Эволюция

| Версия | Что |
|--------|-----|
| **.NET Core 1.0 (2016)** | `ArrayPool<T>` |
| **.NET Core 2.0** | `MemoryPool<T>`, `IMemoryOwner<T>` |
| **.NET Core 2.1** | `Span<T>`, `Memory<T>`, `ReadOnlySpan<T>` |
| **.NET Core 3.0** | `ObjectPool<T>` (Microsoft.Extensions.ObjectPool) |
| **.NET 5+** | More BCL уses pooling internally |
| **.NET 6+** | `IRecyclable`, performance improvements |
| **.NET 8+** | Better escape analysis, less manual pooling needed |

> [!info]- Если ты знаешь Java / Rust / C++ / Go
> **Java:** `ByteBuffer.allocate(n)` no built-in pool. Frameworks (Netty) implement их own pools. JVM escape analysis handles некоторые cases automatically.
>
> **Rust:** ownership semantics — no GC. Allocations explicit, `Vec<u8>` reuse manual. `bytes` crate — pool implementations.
>
> **C++:** RAII + custom allocators. Boost.Pool, jemalloc — common. Manual lifetime management standard.
>
> **Go:** `sync.Pool` — analog `ObjectPool<T>`. Used везде где hot path. GC similar to .NET, pooling решает same problem.

> [!question]- Интервью: зачем нужен memory pooling если есть GC?
> GC handles managed memory automatically, но **allocation/deallocation pressure** в hot path causes: 1) **GC pauses** — Gen 0 collections frequent, ~1ms each — accumulate. 2) **LOH fragmentation** для buffers > 85,000 bytes — Gen 2 only, slow. 3) **Cache misses** — new allocations cold. **Pooling** rents buffers from pool, returns после use. Net result: 10-100x throughput в parsers, serializers, network I/O. Examples in BCL: `JsonSerializer`, `HttpClient`, Kestrel — all use pooling internally. Trade-off: complexity (must Return), bugs если не handle properly.

---

## 2. `ArrayPool<T>` basics

### 2.1. Shared instance

```csharp
using System.Buffers;

byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);
try
{
    // use buffer
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

`ArrayPool<T>.Shared` — singleton, thread-safe, ready to use.

### 2.2. Rent — может вернуть larger buffer

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);
Console.WriteLine(buffer.Length);   // может быть 8192, 16384 или больше!
```

`Rent(n)` — returns buffer **at least** n bytes. Pool implementation rounds up to power of 2 typically. **Always use** explicit length parameter в operations:

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);
int actualLength = 8192;   // track separately
// Не buffer.Length — может быть больше!

Encoding.UTF8.GetBytes(input, 0, input.Length, buffer, 0);
// Или через Span
input.AsSpan().CopyTo(buffer.AsSpan(0, actualLength));
```

### 2.3. Return — clearArray for sensitive data

```csharp
byte[] passwordBuffer = ArrayPool<byte>.Shared.Rent(64);
try
{
    Encoding.UTF8.GetBytes(password, passwordBuffer);
    // ... use
}
finally
{
    ArrayPool<byte>.Shared.Return(passwordBuffer, clearArray: true);
}
```

`clearArray: true` — zeros buffer перед return в pool. Other callers won't see your data. Cost: small overhead. Always use для passwords, tokens, encryption keys.

### 2.4. Custom pool

```csharp
// Default Shared pool — fits most cases
ArrayPool<byte>.Shared

// Custom pool — для specific scenarios
var customPool = ArrayPool<byte>.Create(maxArrayLength: 1024 * 1024, maxArraysPerBucket: 50);
```

`Create()` — separate pool, не shares state с Shared. Used когда:
- Specific size requirements.
- Isolation (тесты, специфические workloads).

### 2.5. Buffer rent/return overhead

```
| Allocation pattern | Time per op |
|---------------------|--------------|
| new byte[8192]      | ~50-200ns + GC pressure |
| Pool.Rent(8192)     | ~20-50ns |
| Pool.Return(buf)    | ~20-50ns |
| stackalloc byte[1024] | ~5ns (no GC) |
```

Pool — 2-4x faster than allocation, plus no GC pressure.

### 2.6. Pool sizing

```
ArrayPool.Shared:
- Buckets по 16, 32, 64, 128, ... 1MB
- Per-bucket: ~50 arrays cached per CPU core
- Excess returns — discarded (becomes GC)
- Total memory: bounded
```

> [!question]- Интервью: что вернёт `ArrayPool.Rent(8192)`?
> Buffer **at least** 8192 bytes. Может быть 8192, 16384 — pool implementation rounds **up** to power of 2 (или specific bucket size). **Always track actual length separately** — `buffer.Length` может быть больше requested. Use explicit length параметр в operations или `Span<T>` slice. Forgetting → buffer overflow логические errors. Best practice: pass length parameter explicitly: `Encoding.UTF8.GetBytes(s, 0, s.Length, buffer, 0)` или `input.AsSpan().CopyTo(buffer.AsSpan(0, length))`.

---

## 3. ArrayPool patterns

### 3.1. Basic try/finally

```csharp
public byte[] Compress(byte[] data)
{
    byte[] buffer = ArrayPool<byte>.Shared.Rent(data.Length * 2);
    try
    {
        int compressedLength = CompressInternal(data, buffer);
        var result = new byte[compressedLength];
        Buffer.BlockCopy(buffer, 0, result, 0, compressedLength);
        return result;
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

### 3.2. RAII через `using` + custom struct

```csharp
public readonly ref struct PooledArray<T>
{
    public readonly T[] Array;
    public readonly int Length;
    
    public PooledArray(int length)
    {
        Array = ArrayPool<T>.Shared.Rent(length);
        Length = length;
    }
    
    public Span<T> AsSpan() => Array.AsSpan(0, Length);
    
    public void Dispose() => ArrayPool<T>.Shared.Return(Array);
}

// Использование
public void Process(byte[] data)
{
    using var pooled = new PooledArray<byte>(8192);
    var span = pooled.AsSpan();
    // ... process span
    // auto Return через Dispose
}
```

### 3.3. Per-call с finally

```csharp
public async Task<string> ReadAsync(Stream stream, int length)
{
    byte[] buffer = ArrayPool<byte>.Shared.Rent(length);
    try
    {
        int read = await stream.ReadAsync(buffer.AsMemory(0, length));
        return Encoding.UTF8.GetString(buffer, 0, read);
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

### 3.4. Streaming — re-rent при growth

```csharp
public byte[] ReadAll(Stream stream)
{
    byte[] buffer = ArrayPool<byte>.Shared.Rent(4096);
    int totalRead = 0;
    
    try
    {
        while (true)
        {
            int read = stream.Read(buffer, totalRead, buffer.Length - totalRead);
            if (read == 0) break;
            totalRead += read;
            
            if (totalRead == buffer.Length)
            {
                // Need bigger — rent new + copy + return old
                byte[] bigger = ArrayPool<byte>.Shared.Rent(buffer.Length * 2);
                Buffer.BlockCopy(buffer, 0, bigger, 0, totalRead);
                ArrayPool<byte>.Shared.Return(buffer);
                buffer = bigger;
            }
        }
        
        var result = new byte[totalRead];
        Buffer.BlockCopy(buffer, 0, result, 0, totalRead);
        return result;
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

### 3.5. ArrayPool в LINQ-like

```csharp
public static class PooledLinq
{
    public static byte[] ToPooledArray<T>(this IEnumerable<T> source)
    {
        // Implementation using ArrayPool
        // ... pattern
    }
}
```

LINQ extensions с pooling — common в performance libraries.

> [!question]- Интервью: чем `ArrayPool.Shared` отличается от `ArrayPool.Create()`?
> **`Shared`** — singleton instance, optimized для general use, thread-safe, ready to use. **`Create(maxArrayLength, maxArraysPerBucket)`** — custom pool с separate state. Useful для: 1) **Isolation** — тесты не должны share с production code. 2) **Specific bucket sizes** — если known buffer sizes. 3) **Bounded memory** — limit total pool size. Best practice 99% case: use `Shared`. Custom pool — micro-optimization, обычно overkill. **Never** create custom pool в hot path (each `Create` allocates).

---

## 4. `MemoryPool<T>`

### 4.1. Зачем

`MemoryPool<T>` — abstraction над allocation, returns `IMemoryOwner<T>` (RAII):

```csharp
using System.Buffers;

using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(8192);
Memory<byte> memory = owner.Memory;
// use memory
// auto-release при Dispose
```

`IMemoryOwner<T>` — **disposable wrapper**, return automatic.

### 4.2. MemoryPool vs ArrayPool

| | `ArrayPool<T>` | `MemoryPool<T>` |
|---|--------------|----------------|
| Returns | `T[]` | `IMemoryOwner<T>` (`Memory<T>`) |
| Lifetime | Manual try/finally | RAII через using |
| Performance | Faster (direct array) | Slight indirection |
| Use case | Hot path, low overhead | Async / long-lived owners |

`MemoryPool<T>` builds on `ArrayPool<T>` internally. `Memory<T>` — convenient API.

### 4.3. With async

```csharp
public async Task<int> ReadAsync(Stream stream)
{
    using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(8192);
    Memory<byte> memory = owner.Memory;
    int bytesRead = await stream.ReadAsync(memory);
    return bytesRead;
}
// Auto Return при Dispose
```

`Memory<T>` works в async (unlike `Span<T>`!).

### 4.4. Custom IMemoryOwner

```csharp
public sealed class CustomOwner : IMemoryOwner<byte>
{
    private byte[] _buffer;
    public Memory<byte> Memory => _buffer;
    
    public CustomOwner(int size) => _buffer = ArrayPool<byte>.Shared.Rent(size);
    
    public void Dispose()
    {
        ArrayPool<byte>.Shared.Return(_buffer);
        _buffer = null!;
    }
}
```

### 4.5. Когда MemoryPool

✅ **Используй когда:**
- Async cleanup needed.
- Buffer travels через layers (return value).
- RAII pattern preferred.

❌ **Не нужно когда:**
- Sync hot path — `ArrayPool` direct cheaper.
- Stack allocation possible (`stackalloc` для < 1KB).

> [!question]- Интервью: чем `MemoryPool<T>` отличается от `ArrayPool<T>`?
> **`ArrayPool<T>`** — returns `T[]` directly, manual `Return` через try/finally. Faster, less abstraction. **`MemoryPool<T>`** — returns `IMemoryOwner<T>` wrapper с `Memory<T>` property + `Dispose()` для return. RAII — `using` block. Internally uses ArrayPool. Use cases: async paths (where `Memory<T>` works, `Span<T>` не can pass через await), buffer ownership transfer (return IMemoryOwner from method), cleaner code. Trade-off: small overhead (extra wrapper), но negligible. Best practice: ArrayPool в hot path tight loops, MemoryPool в API surface / async.

---

## 5. `Span<T>` и `Memory<T>` — zero-copy

### 5.1. `Span<T>` basics

```csharp
byte[] buffer = new byte[1024];
Span<byte> span = buffer.AsSpan();   // view над buffer
Span<byte> slice = span.Slice(100, 200);   // sub-view
slice[0] = 1;    // modifies original buffer

// String view
ReadOnlySpan<char> chars = "hello".AsSpan();
ReadOnlySpan<char> sub = chars.Slice(1, 3);   // "ell"
```

`Span<T>` — **stack-only** view над memory. **No allocation** для slicing. Read-write, mutable.

### 5.2. `ReadOnlySpan<T>` — для immutable data

```csharp
ReadOnlySpan<char> input = "hello world".AsSpan();
ReadOnlySpan<char> first = input.Slice(0, 5);   // "hello", no allocation
```

`ReadOnlySpan<T>` — read-only view. Critical для string slicing без allocation.

### 5.3. stackalloc — на stack

```csharp
Span<byte> stackBuffer = stackalloc byte[256];   // на stack, no GC!
stackBuffer[0] = 1;
// Lifetime — current method, не escape
```

`stackalloc` — direct stack allocation. **No GC**, fast. Limit ~1KB safe (stack overflow risk).

### 5.4. Combined — large stack или pool

```csharp
public void Process(string input)
{
    const int StackThreshold = 256;
    
    Span<byte> buffer = input.Length <= StackThreshold
        ? stackalloc byte[StackThreshold]
        : new byte[input.Length * 2];   // или ArrayPool
    
    int written = Encoding.UTF8.GetBytes(input, buffer);
    // process buffer.Slice(0, written)
}
```

Common pattern: `stackalloc` для small, heap (или pool) для large.

### 5.5. `Span<T>` ограничения

```csharp
// ❌ Span<T> не может cross await boundary
public async Task M()
{
    Span<byte> span = stackalloc byte[100];
    await Task.Delay(100);   // ❌ Compile error!
    span[0] = 1;
}

// ❌ Span<T> не может быть field
public class C
{
    private Span<byte> _span;   // ❌ Compile error
}

// ❌ Span<T> не может быть в generic
List<Span<byte>>;   // ❌ Compile error
```

`Span<T>` — `ref struct`, stack-only by design. Для async / async storage — use `Memory<T>`.

### 5.6. `Memory<T>` — heap-friendly

```csharp
public async Task ProcessAsync(byte[] data)
{
    Memory<byte> memory = data.AsMemory();
    await stream.WriteAsync(memory);   // ✅ Memory<T> works в async
    
    Memory<byte> slice = memory.Slice(100, 200);   // no allocation
}
```

`Memory<T>` — heap-friendly view. Slower than Span (extra indirection), но works в async.

### 5.7. ToArray, no-alloc operations

```csharp
ReadOnlySpan<char> input = "hello".AsSpan();

// Common operations no-alloc
int idx = input.IndexOf('l');           // no alloc
bool starts = input.StartsWith("he".AsSpan());   // no alloc
int hash = string.GetHashCode(input);   // no alloc

// Allocating — explicit
string s = input.ToString();             // alloc
char[] arr = input.ToArray();             // alloc
```

Best practice: stay в `Span<T>` сколько возможно, только на boundary конвертируй в string/array.

### 5.8. `Span<T>` в parsers

```csharp
public static int ParseInt(ReadOnlySpan<char> input)
{
    int result = 0;
    foreach (var ch in input)
    {
        if (ch < '0' || ch > '9') throw new FormatException();
        result = result * 10 + (ch - '0');
    }
    return result;
}

// Use без substring allocation
ReadOnlySpan<char> data = "123,456,789".AsSpan();
int comma1 = data.IndexOf(',');
int n1 = ParseInt(data.Slice(0, comma1));   // no string alloc!
int n2 = ParseInt(data.Slice(comma1 + 1, data.IndexOf(',', comma1 + 1) - comma1 - 1));
```

Modern parsers (UTF8, JSON, CSV) heavily use `Span<T>` — zero allocation.

> [!question]- Интервью: чем `Span<T>` отличается от `Memory<T>`?
> **`Span<T>`** — `ref struct`, stack-only. **No async** (cross await boundary forbidden), не может быть field/generic, lifetime — current method. Maximum performance — direct memory access, JIT optimizations. **`Memory<T>`** — regular struct, heap-friendly. **Works in async**, can be field, generic. Slight indirection (extra ptr resolve). Use case Span: hot path tight loops, parsers, sync. Memory: async I/O, API boundaries, owned buffers (`IMemoryOwner<T>`). Common pattern: API takes/returns Memory, internally converts to Span для performance. **Never** field of class — `Span<T>` (compile error), `Memory<T>` OK.

---

## 6. `ObjectPool<T>` — объекты, не buffers

### 6.1. Microsoft.Extensions.ObjectPool

```csharp
using Microsoft.Extensions.ObjectPool;

var pool = new DefaultObjectPool<StringBuilder>(new StringBuilderPooledObjectPolicy());

StringBuilder sb = pool.Get();
try
{
    sb.Append("Hello, ");
    sb.Append("world!");
    string result = sb.ToString();
}
finally
{
    pool.Return(sb);   // Reset+ return
}
```

NuGet: `Microsoft.Extensions.ObjectPool`.

### 6.2. PooledObjectPolicy

```csharp
public class StringBuilderPooledObjectPolicy : PooledObjectPolicy<StringBuilder>
{
    public int InitialCapacity { get; set; } = 100;
    public int MaximumRetainedCapacity { get; set; } = 4096;
    
    public override StringBuilder Create() => new StringBuilder(InitialCapacity);
    
    public override bool Return(StringBuilder obj)
    {
        if (obj.Capacity > MaximumRetainedCapacity)
        {
            return false;   // не cache too-big objects
        }
        obj.Clear();
        return true;
    }
}
```

Policy controls: how to create, how to reset, when to discard.

### 6.3. Custom pooled object

```csharp
public class MyParser
{
    public StringBuilder Buffer { get; } = new();
    public List<string> Tokens { get; } = new();
    
    public void Reset()
    {
        Buffer.Clear();
        Tokens.Clear();
    }
}

public class MyParserPolicy : PooledObjectPolicy<MyParser>
{
    public override MyParser Create() => new MyParser();
    
    public override bool Return(MyParser parser)
    {
        parser.Reset();
        return true;
    }
}

var pool = new DefaultObjectPool<MyParser>(new MyParserPolicy());
```

### 6.4. ASP.NET Core uses

ASP.NET Core internally pools:
- `StringBuilder` для responses.
- `HttpContext` features.
- DI scope objects (under specific conditions).

Not exposed direct, но affects performance.

### 6.5. Когда ObjectPool

✅ **Используй когда:**
- Object **expensive** to create (StringBuilder, ParserContext).
- High allocation rate.
- Reset semantics clear.

❌ **Не используй когда:**
- Object cheap to create.
- Reset complex / error-prone.
- Lifetime unclear.

> [!question]- Интервью: чем `ObjectPool<T>` отличается от `ArrayPool<T>`?
> **`ArrayPool<T>`** — для arrays of value types (byte[], char[], int[]). Specialized для buffer reuse. **`ObjectPool<T>`** — для **arbitrary objects** (StringBuilder, parsers, contexts). Generic over reference type. **Policy-based**: how to create, reset, when to discard. Examples: pool StringBuilder (avoid `new StringBuilder(...)` per call), pool parser contexts, pool DI scopes. ASP.NET Core uses internally. Best practice: ObjectPool для expensive-to-create objects, ArrayPool для buffers, oba для same goal — reduce allocation pressure.

---

## 7. Real-world patterns

### 7.1. JsonSerializer + ArrayPool

`System.Text.Json` use ArrayPool internally:

```csharp
// Underneath JsonSerializer.Serialize:
byte[] buffer = ArrayPool<byte>.Shared.Rent(...);
try
{
    Utf8JsonWriter writer = new(buffer);
    writer.WriteStartObject();
    // ... serialize
    return Encoding.UTF8.GetString(buffer, 0, writer.BytesWritten);
}
finally { ArrayPool<byte>.Shared.Return(buffer); }
```

User не aware — но performance gain significant.

### 7.2. Pipelines + buffers

```csharp
PipeReader reader = ...;

while (true)
{
    ReadResult result = await reader.ReadAsync();
    ReadOnlySequence<byte> buffer = result.Buffer;
    
    // process buffer (may span multiple ArrayPool segments)
    SequencePosition? newline = buffer.PositionOf((byte)'\n');
    if (newline != null)
    {
        ProcessLine(buffer.Slice(0, newline.Value));
    }
    
    reader.AdvanceTo(buffer.GetPosition(1, newline.Value));
    if (result.IsCompleted) break;
}

await reader.CompleteAsync();
```

`System.IO.Pipelines` — entire pipeline pooled buffers, zero-copy.

### 7.3. HttpClient streaming

```csharp
using HttpResponseMessage response = await client.GetAsync(url, HttpCompletionOption.ResponseHeadersRead);
using Stream stream = await response.Content.ReadAsStreamAsync();

byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);
try
{
    int read;
    while ((read = await stream.ReadAsync(buffer)) > 0)
    {
        // process buffer.AsSpan(0, read)
    }
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);
}
```

Streaming + pool — no large response buffer in memory.

### 7.4. Custom serializer

```csharp
public byte[] Serialize<T>(T data)
{
    byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
    int written = 0;
    try
    {
        // serialize в buffer
        while (true)
        {
            try
            {
                written = WriteToBuffer(data, buffer);
                break;
            }
            catch (BufferTooSmallException)
            {
                // Grow
                ArrayPool<byte>.Shared.Return(buffer);
                buffer = ArrayPool<byte>.Shared.Rent(buffer.Length * 2);
            }
        }
        
        // Copy actual bytes
        var result = new byte[written];
        Buffer.BlockCopy(buffer, 0, result, 0, written);
        return result;
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buffer);
    }
}
```

### 7.5. Parser hot path

```csharp
public List<string> ParseCsvLine(ReadOnlySpan<char> line)
{
    var result = new List<string>();
    int start = 0;
    for (int i = 0; i < line.Length; i++)
    {
        if (line[i] == ',')
        {
            // Substring-like, но zero allocation для slicing
            result.Add(line.Slice(start, i - start).ToString());   // ToString allocates
            start = i + 1;
        }
    }
    result.Add(line.Slice(start).ToString());
    return result;
}
```

Spans для slicing, ToString только когда нужен string output.

### 7.6. Logging + ObjectPool

```csharp
public class LogFormatter
{
    private static readonly ObjectPool<StringBuilder> _pool =
        new DefaultObjectPool<StringBuilder>(new StringBuilderPooledObjectPolicy());
    
    public string Format(LogEntry entry)
    {
        StringBuilder sb = _pool.Get();
        try
        {
            sb.Append(entry.Timestamp);
            sb.Append(" [");
            sb.Append(entry.Level);
            sb.Append("] ");
            sb.Append(entry.Message);
            return sb.ToString();
        }
        finally
        {
            _pool.Return(sb);
        }
    }
}
```

Logging — common pooling target (high frequency).

---

## 8. Measurement и profiling

### 8.1. Benchmark с BenchmarkDotNet

```csharp
[MemoryDiagnoser]
public class PoolingBenchmark
{
    private readonly byte[] _data = new byte[8192];
    
    [Benchmark(Baseline = true)]
    public void NewArray()
    {
        byte[] buffer = new byte[8192];
        // simulate work
        for (int i = 0; i < buffer.Length; i++) buffer[i] = (byte)i;
    }
    
    [Benchmark]
    public void Pooled()
    {
        byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);
        try
        {
            for (int i = 0; i < buffer.Length; i++) buffer[i] = (byte)i;
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(buffer);
        }
    }
}

// Output:
// |  Method  |     Mean | Allocated |
// |--------- |---------:|----------:|
// | NewArray | 1.234 us |    8192 B |
// | Pooled   | 0.456 us |       0 B |
```

`[MemoryDiagnoser]` — show allocations.

### 8.2. dotnet-counters runtime

```bash
dotnet-counters monitor --process-id <pid> System.Runtime
```

Watch: `gen-0-gc-count`, `gen-1-gc-count`, `gen-2-gc-count`, `working-set`. Pooling должен снизить GC counts.

### 8.3. PerfView / dotMemory

```
Trace allocation hot paths:
1. PerfView Collect → Run app → Stop
2. Open trace → GC Heap Alloc Stack
3. Find top allocation sources
4. Apply pooling
5. Re-measure
```

### 8.4. ETW counters

```csharp
// Вручную instrument
public class PoolMetrics
{
    public long Rented;
    public long Returned;
    public long ArraySize;
}
```

Production observability — track pool effectiveness.

> [!question]- Интервью: как измерить эффективность pooling?
> 1) **BenchmarkDotNet с `[MemoryDiagnoser]`** — show allocated bytes per operation. Compare baseline (`new`) vs pooled. 2) **dotnet-counters runtime** — gen-0/1/2 GC counts per second. Pooling drops counts dramatically. 3) **PerfView / dotMemory** — allocation hot paths, find где дорого аллоцируется. 4) **Application throughput** — req/sec в load test. Pooling should improve. **Метрики**: 1) **Allocations** per operation drop ~100x. 2) **GC time** % drop. 3) **Latency P99** improvement (no GC pauses). 4) **Throughput** rises. Без profile — pooling **может ухудшить** (overhead для small allocations).

---

## 9. Best practices

### 9.1. ArrayPool

- ✅ **`Rent` + `try/finally Return`** always.
- ✅ **`clearArray: true`** для sensitive data.
- ✅ **Track actual length** separately (Length может быть больше).
- ✅ **`Shared`** в 99% cases.
- ❌ **Don't pool small buffers** (< 1KB).
- ❌ **Don't keep rented buffer** longer than needed.

### 9.2. Span/Memory

- ✅ **`Span<T>`** в hot path tight loops (no async, no field).
- ✅ **`Memory<T>`** на API boundaries / async.
- ✅ **`stackalloc`** для < 1KB.
- ✅ **`ReadOnlySpan<char>`** для substring без alloc.
- ❌ **`Span<T>` в class field** — compile error.
- ❌ **`Span<T>` cross await** — compile error.

### 9.3. ObjectPool

- ✅ **Expensive create** объектов (StringBuilder, parsers).
- ✅ **PooledObjectPolicy.Return** resets state.
- ✅ **DefaultObjectPool** для most cases.
- ❌ **Don't pool tiny objects** (overhead).
- ❌ **Don't pool stateful objects** без clear reset semantics.

### 9.4. Profiling first

- ✅ **Profile before pool** — allocations real bottleneck?
- ✅ **Measure after** — actual improvement?
- ❌ **Premature pooling** — adds complexity без gain.

### 9.5. Не делай

- ❌ Forget `Return` — leak (pool grows or array becomes GC).
- ❌ Use buffer **after** Return — corruption (other code uses).
- ❌ Multiple `Return` calls для same buffer.
- ❌ Pool в low-frequency paths.
- ❌ Custom pool без real need.

---

## 10. Decision tree

```
Allocation pressure?
│
├── Profiled bottleneck (PerfView / BenchmarkDotNet)?
│   ├── Yes — продолжаем
│   └── No — skip pooling
│
├── Buffer тип?
│   ├── byte[]/char[]/int[] (arrays of value types)
│   │   ├── Hot path tight loop → ArrayPool<T>.Shared
│   │   ├── API boundary / async → MemoryPool<T>
│   │   └── Small (< 1KB) → stackalloc
│   │
│   ├── StringBuilder / parser / context (objects)
│   │   ├── Expensive create → ObjectPool<T>
│   │   └── Cheap → just `new`
│   │
│   └── Span<T> view (already pooled buffer)
│       ├── Sync hot path → Span<T>
│       └── Async / field → Memory<T>
│
├── Lifetime
│   ├── Tight scope → using / try-finally
│   └── Cross-method → IMemoryOwner<T>
│
└── Sensitive data?
    ├── Yes → clearArray: true
    └── No → default Return
```

---

## 11. Cheat sheet

```csharp
using System.Buffers;
using Microsoft.Extensions.ObjectPool;

// === ArrayPool ===
byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);
try
{
    int actualLength = 8192;
    // work — track actualLength separately!
    process(buffer.AsSpan(0, actualLength));
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer, clearArray: false);
}

// === MemoryPool — async-friendly ===
using IMemoryOwner<byte> owner = MemoryPool<byte>.Shared.Rent(8192);
Memory<byte> memory = owner.Memory;
await stream.WriteAsync(memory);

// === stackalloc — small buffers (< 1KB) ===
Span<byte> stack = stackalloc byte[256];
process(stack);

// === Hybrid stack/heap ===
Span<byte> buffer = length <= 256
    ? stackalloc byte[256]
    : new byte[length];

// === Span<T> view (no allocation) ===
ReadOnlySpan<char> input = "hello world".AsSpan();
ReadOnlySpan<char> first = input.Slice(0, 5);   // "hello"
int idx = input.IndexOf(' ');                    // no alloc
bool starts = input.StartsWith("he".AsSpan());   // no alloc

// === Memory<T> — async-friendly ===
public async Task<int> ReadAsync(Memory<byte> destination)
{
    return await stream.ReadAsync(destination);
}

// === ObjectPool ===
var pool = new DefaultObjectPool<StringBuilder>(new StringBuilderPooledObjectPolicy());

StringBuilder sb = pool.Get();
try
{
    sb.Append("...");
    return sb.ToString();
}
finally
{
    pool.Return(sb);   // reset через policy
}

// === Custom RAII через struct ===
public readonly ref struct PooledArray<T>
{
    private readonly T[] _array;
    public Span<T> Span { get; }
    
    public PooledArray(int length)
    {
        _array = ArrayPool<T>.Shared.Rent(length);
        Span = _array.AsSpan(0, length);
    }
    
    public void Dispose() => ArrayPool<T>.Shared.Return(_array);
}

using var pooled = new PooledArray<byte>(8192);
process(pooled.Span);

// === BenchmarkDotNet ===
[MemoryDiagnoser]
public class Benchmarks
{
    [Benchmark(Baseline = true)] public void Plain() => new byte[8192];
    [Benchmark] public void Pooled()
    {
        var buf = ArrayPool<byte>.Shared.Rent(8192);
        ArrayPool<byte>.Shared.Return(buf);
    }
}
```

---

## 12. Common pitfalls

### 12.1. Forget Return — leak

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
// ... work, throws exception
// buffer не returned — GC will collect, но pool не reuse
```

**Фикс:** `try/finally Return`.

### 12.2. Use after Return — corruption

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
ArrayPool<byte>.Shared.Return(buffer);
buffer[0] = 1;   // ❌ another renter может уже use!
```

**Фикс:** не trust buffer после Return.

### 12.3. Buffer.Length surprise

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(100);
Console.WriteLine(buffer.Length);   // 128 (or larger), не 100!

input.CopyTo(buffer);   // если input < 128 — ok, иначе problem
```

**Фикс:** explicit length parameter.

### 12.4. `Span<T>` в async

```csharp
public async Task M()
{
    Span<byte> buf = stackalloc byte[100];   // ❌ + await дальше
    await Task.Delay(100);
    buf[0] = 1;   // Compile error
}
```

**Фикс:** `Memory<T>` или extract Span work в helper method.

### 12.5. Stack overflow от stackalloc

```csharp
public void M(int size)
{
    Span<byte> buf = stackalloc byte[size];   // ❌ если size huge
}
```

**Фикс:** size threshold check + heap fallback.

### 12.6. Pool sensitive data без clear

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(64);
Encoding.UTF8.GetBytes(password, buffer);
// Process...
ArrayPool<byte>.Shared.Return(buffer);   // ❌ password остаётся в memory!
// Other rented поднимет тот же buffer — visible!
```

**Фикс:** `Return(buffer, clearArray: true)`.

### 12.7. Double Return — pool corruption

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(1024);
ArrayPool<byte>.Shared.Return(buffer);
ArrayPool<byte>.Shared.Return(buffer);   // ❌ same buffer twice
```

**Фикс:** track ownership, single Return.

### 12.8. Pool too-small для request

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(int.MaxValue);   // ❌ huge!
// Pool скорее allocate new (без caching) — defeats purpose
```

**Фикс:** reasonable sizes только.

### 12.9. Object pool без Reset

```csharp
public class BadPolicy : PooledObjectPolicy<StringBuilder>
{
    public override StringBuilder Create() => new();
    public override bool Return(StringBuilder obj) => true;   // ❌ no Clear
}

// Next renter получит builder с предыдущим contents!
```

**Фикс:** `obj.Clear()` в Return.

### 12.10. Premature pooling

```csharp
public string Greeting() 
{
    byte[] buf = ArrayPool<byte>.Shared.Rent(16);
    try
    {
        return "Hello";
    }
    finally
    {
        ArrayPool<byte>.Shared.Return(buf);
    }
}
// Pooling overhead > allocation cost для tiny / rare
```

**Фикс:** profile first, pool только bottlenecks.

> [!question]- Интервью: топ-3 ошибки с pooling?
> 1) **Forget `Return`** — leak (buffer становится GC, pool не reused). Always try/finally или using-based RAII. 2) **Use buffer after `Return`** — другой renter уже use, corruption. Track ownership строго. 3) **`buffer.Length` surprise** — Rent returns ≥ requested, может быть larger. Always track requested length separately, use explicit length parameter в API calls. Бонус: sensitive data без `clearArray: true` — passwords visible в pool.

---

## 13. Practice exercises

### 13.1. Pooled string builder с RAII

```csharp
public sealed class PooledStringBuilder : IDisposable
{
    private static readonly ObjectPool<StringBuilder> _pool =
        new DefaultObjectPool<StringBuilder>(new StringBuilderPooledObjectPolicy());
    
    private StringBuilder? _sb;
    
    public PooledStringBuilder() => _sb = _pool.Get();
    
    public StringBuilder Builder => _sb ?? throw new ObjectDisposedException(nameof(PooledStringBuilder));
    
    public string ToStringAndDispose()
    {
        var s = Builder.ToString();
        Dispose();
        return s;
    }
    
    public void Dispose()
    {
        if (_sb != null)
        {
            _pool.Return(_sb);
            _sb = null;
        }
    }
}

// Использование
using var psb = new PooledStringBuilder();
psb.Builder.Append("Hello, ").Append("world!");
string result = psb.ToStringAndDispose();
```

### 13.2. Span-based CSV parser

```csharp
public static List<string> ParseCsvLine(ReadOnlySpan<char> line)
{
    var result = new List<string>();
    int start = 0;
    bool inQuotes = false;
    
    for (int i = 0; i < line.Length; i++)
    {
        char c = line[i];
        if (c == '"') { inQuotes = !inQuotes; continue; }
        if (c == ',' && !inQuotes)
        {
            result.Add(line.Slice(start, i - start).Trim('"').ToString());
            start = i + 1;
        }
    }
    result.Add(line.Slice(start).Trim('"').ToString());
    return result;
}

// Use
ReadOnlySpan<char> input = "Alice,30,\"NYC, NY\"".AsSpan();
var fields = ParseCsvLine(input);
// no substring allocation для slicing — только final ToString
```

### 13.3. Pooled buffer reader

```csharp
public sealed class PooledBufferReader : IAsyncDisposable
{
    private byte[] _buffer;
    private readonly Stream _stream;
    private bool _disposed;
    
    public PooledBufferReader(Stream stream, int initialSize = 4096)
    {
        _stream = stream;
        _buffer = ArrayPool<byte>.Shared.Rent(initialSize);
    }
    
    public async Task<int> ReadAsync(Memory<byte> destination)
    {
        if (_disposed) throw new ObjectDisposedException(nameof(PooledBufferReader));
        return await _stream.ReadAsync(destination);
    }
    
    public ValueTask DisposeAsync()
    {
        if (_disposed) return ValueTask.CompletedTask;
        ArrayPool<byte>.Shared.Return(_buffer);
        _buffer = null!;
        _disposed = true;
        return ValueTask.CompletedTask;
    }
}
```

---

## 14. Что читать дальше

1. **[[types-and-memory|Types and Memory]]** — GC deep, generations.
2. **[[unsafe-pointers|Unsafe Pointers]]** — direct memory.
3. **[[io-streams|IO Streams]]** — Pipelines внутри.
4. **System.IO.Pipelines deep**.
5. **Native AOT** — ahead-of-time + pooling.

---

## 15. См. также

- [[types-and-memory|Types and Memory]] — GC pressure
- [[unsafe-pointers|Unsafe Pointers]] — direct memory
- [[io-streams|IO Streams]]
- [[async-threading|Async Threading]] — `Memory<T>` в async
- System.Buffers namespace
- Microsoft.Extensions.ObjectPool

---

## 16. Reading list

- **Microsoft Docs — `ArrayPool<T>`** — learn.microsoft.com/dotnet/api/system.buffers.arraypool-1
- **Microsoft Docs — `MemoryPool<T>`** — learn.microsoft.com/dotnet/api/system.buffers.memorypool-1
- **Microsoft Docs — `Span<T>`** — learn.microsoft.com/dotnet/api/system.span-1
- **Microsoft Docs — `Memory<T>`** — learn.microsoft.com/dotnet/api/system.memory-1
- **Microsoft Docs — System.IO.Pipelines** — learn.microsoft.com/dotnet/standard/io/pipelines
- **Stephen Toub — Performance** (devblogs.microsoft.com)
- **Adam Sitnik — Pooling guide** — adamsitnik.com
- **David Fowler — Pipelines** — devblogs.microsoft.com
- **Konrad Kokosa — Pro .NET Memory Management** (book) — comprehensive
- **BenchmarkDotNet docs** — benchmarkdotnet.org
