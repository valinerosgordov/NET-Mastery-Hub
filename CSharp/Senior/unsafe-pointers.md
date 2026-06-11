---
tags: [csharp, unsafe, pointers, senior, fixed, stackalloc, interop, native, aot]
level: Senior
date: 2026-05-09
---

# Unsafe code и Pointers — управление памятью напрямую

> **`unsafe` context, raw pointers, `fixed`, `stackalloc`, function pointers, P/Invoke marshaling.** Когда managed code не достаточно: native interop, performance hot path, custom allocators. Закрывает пробел: «знаю что unsafe существует, не понимаю когда оправдан и почему `fixed` нужен для GC pinning».

---

## 0. Как читать

Если впервые — раздел 1→3 (зачем + базовый pointer syntax). Если уже работал с native — раздел 5 (P/Invoke marshaling). Function pointers — раздел 7. Production guidance — раздел 11 (best practices), 13 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Что такое unsafe code

```csharp
public unsafe void Process(byte[] data)
{
    fixed (byte* ptr = data)
    {
        for (int i = 0; i < data.Length; i++)
            *(ptr + i) = (byte)(*(ptr + i) ^ 0xFF);
    }
}
```

`unsafe` — позволяет:
- **Pointer arithmetic** (`*ptr`, `ptr + i`).
- **`fixed`** — pin memory от GC.
- **`stackalloc`** — stack allocation.
- **`sizeof`** на user types.
- **Function pointers** (C# 9+).

### 1.2. Когда unsafe оправдан

```
✅ Используй когда:
  - P/Invoke с native libraries
  - Hot path performance critical (10x speedup measured)
  - Image / audio / video processing (pixel access)
  - Cryptography, compression
  - Custom allocators / memory layouts
  - Interop с C/C++ structs

❌ Не используй когда:
  - Span<T>/Memory<T> сделают то же samely safely
  - Нет measured performance benefit
  - Premature optimization
  - Maintainability важнее, чем 5% speed gain
```

### 1.3. Главное правило

```
unsafe — last resort, не first choice.

Modern C# (Span<T>, ref struct, ArrayPool, Marshal) покрывает 95% случаев safely.

Если используешь unsafe:
  - Profile measure benefit (must be > 2x improvement)
  - Document why
  - Isolate в helper class
  - Code review extra strict
  - Tests cover boundaries
```

### 1.4. Project setup

```xml
<PropertyGroup>
  <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
</PropertyGroup>
```

Без этого `unsafe` блок = compile error. Opt-in flag.

### 1.5. Эволюция

| Версия | Что |
|--------|-----|
| **C# 1.0** | `unsafe`, pointers, `fixed`, `stackalloc` |
| **C# 7.2** | `ref struct`, `Span<T>` (mostly replaces unsafe) |
| **C# 7.3** | `unmanaged` constraint, `fixed` для custom types |
| **C# 8.0** | `stackalloc` в expressions |
| **C# 9.0** | **Function pointers** `delegate*` |
| **C# 9.0** | `[UnmanagedCallersOnly]` для callbacks из native |
| **.NET 7+** | `[LibraryImport]` source generator (replace P/Invoke) |
| **.NET 8+** | Native AOT — больше unsafe scenarios important |

> [!info]- Если ты знаешь C / C++ / Rust / Go
> **C / C++:** pointers — основа. C# unsafe — порт C-style pointers в .NET. Same syntax (`*ptr`, `&var`, `ptr->field`).
>
> **Rust:** unsafe blocks — same idea, raw pointers. Rust stricter — borrow checker bypass requires unsafe. C# unsafe больше permissive.
>
> **Go:** `unsafe.Pointer`, `unsafe.Sizeof`. Similar concept, less ergonomic. `cgo` для C interop.
>
> **Java:** **нет** unsafe pointers (officially). `sun.misc.Unsafe` — internal API, deprecated в Java 9+. JNI для native interop. C# имеет full unsafe support.

> [!question]- Интервью: что такое unsafe context в C#?
> Block code где разрешены **pointers**, **`fixed`** для GC pinning, **`stackalloc`**, **`sizeof`** на user types, **function pointers** (`delegate*` C# 9+). Требует `<AllowUnsafeBlocks>true</AllowUnsafeBlocks>` в csproj. Use cases: P/Invoke с native libs, performance hot paths (image/audio processing), custom allocators, interop с C/C++. Trade-offs: bypass type safety, manual memory management, harder debugging, security risk (buffer overflows). Best practice 2024+: `Span<T>`/`Memory<T>` covers 95% случаев safely. Unsafe — last resort с measured benefit.

---

## 2. Pointer types

### 2.1. Базовые pointer types

```csharp
unsafe
{
    int x = 42;
    int* ptr = &x;          // pointer to int
    Console.WriteLine(*ptr); // dereference: 42
    *ptr = 100;              // write through pointer
    Console.WriteLine(x);    // 100
    
    int** pp = &ptr;         // pointer to pointer
    void* vp = ptr;          // untyped pointer (like void* в C)
}
```

Syntax — same как C: `T*`, `*ptr`, `&var`.

### 2.2. Pointer arithmetic

```csharp
unsafe
{
    int* arr = stackalloc int[10];
    for (int i = 0; i < 10; i++)
        *(arr + i) = i * i;   // arr[i] = i*i
    
    // Equivalent
    arr[0] = 0;
    arr[1] = 1;
    
    // Manual stride
    int* p = arr;
    p++;            // p += sizeof(int)
    p += 3;         // p += 3 * sizeof(int)
}
```

`ptr + n` — adds `n * sizeof(T)` bytes. `++`/`--` — single element step. `[n]` — same as `*(ptr+n)`.

### 2.3. Sizeof

```csharp
unsafe
{
    int sz1 = sizeof(int);          // 4 — works without unsafe для primitives
    int sz2 = sizeof(MyStruct);     // requires unsafe для user types
}

[StructLayout(LayoutKind.Sequential)]
public struct MyStruct
{
    public int X, Y, Z;
}
```

`sizeof(T)` для user types требует unsafe (compile-time size).

### 2.4. Address-of operator

```csharp
unsafe
{
    int x = 42;
    int* ptr = &x;   // address of local
    
    int[] arr = { 1, 2, 3 };
    // ❌ int* p = &arr[0];   — managed array, нужен fixed
}
```

Address-of (`&`) works на:
- Local variables (stack — safe).
- Fields (через `fixed`).
- Array elements (через `fixed`).
- Parameters.

Не работает на: properties, expressions, managed types.

### 2.5. Pointer types restrictions

```csharp
// ❌ pointer не может быть generic type param
List<int*> bad;   // Compile error

// ❌ pointer не может быть field в class (managed type)
class C { int* ptr; }   // Compile error (без unsafe + readonly)

// ✅ pointer field в struct OK
unsafe struct S { public int* ptr; }
```

Pointers — **unmanaged** by nature. Cannot mix freely с managed types.

### 2.6. nint / nuint — native-sized integers

```csharp
nint n = 100;       // IntPtr alias — 32-bit на x86, 64-bit на x64
nuint un = 200;     // UIntPtr alias

unsafe
{
    void* ptr = (void*)n;   // convert nint → pointer
    nint addr = (nint)ptr;   // pointer → nint
}
```

`nint`/`nuint` — для interop с native APIs где address size matters.

> [!question]- Интервью: чем `nint` отличается от `int`?
> **`int`** — always 32-bit (System.Int32), platform-independent. **`nint`** — native-sized integer (System.IntPtr alias), 32-bit на x86, 64-bit на x64. Для **interop** с native APIs где address-sized integers нужны (HWND, file handles, pointers). До C# 9 — IntPtr. C# 9+ — `nint` keyword + arithmetic operators (раньше IntPtr не поддерживал `+`/`-` напрямую). Use cases: P/Invoke parameters, pointer arithmetic, sizes больше 2GB. Best practice: `nint` для address-related, `int` для regular numerics.

---

## 3. fixed — pin memory от GC

### 3.1. Зачем pinning

GC moves objects в memory (compaction). Pointer to managed object becomes invalid. `fixed` — **pin** object so GC won't move.

```csharp
public unsafe void Process(byte[] data)
{
    fixed (byte* ptr = data)
    {
        // GC не moves data пока в fixed scope
        for (int i = 0; i < data.Length; i++)
            ptr[i] ^= 0xFF;
    }
    // После — GC может move
}
```

### 3.2. fixed на array

```csharp
unsafe
{
    int[] arr = new int[10];
    fixed (int* ptr = arr)
    {
        for (int i = 0; i < 10; i++)
            ptr[i] = i;
    }
}
```

`fixed (T* ptr = array)` — `ptr` указывает на element 0.

### 3.3. fixed на element

```csharp
unsafe
{
    int[] arr = new int[10];
    fixed (int* ptr = &arr[5])
    {
        // ptr — на element 5
        ptr[0] = 100;   // arr[5] = 100
        ptr[1] = 200;   // arr[6] = 200
    }
}
```

### 3.4. fixed на string

```csharp
unsafe
{
    string s = "hello";
    fixed (char* ptr = s)
    {
        // ptr — на null-terminated chars
        for (int i = 0; ptr[i] != 0; i++)
            Console.WriteLine(ptr[i]);
    }
}
```

`fixed` на string — char pointer. **Не modify через ptr** — strings immutable, undefined behavior.

### 3.5. Multiple fixed

```csharp
unsafe
{
    byte[] src = ...;
    byte[] dst = ...;
    
    fixed (byte* pSrc = src)
    fixed (byte* pDst = dst)
    {
        Buffer.MemoryCopy(pSrc, pDst, dst.Length, src.Length);
    }
}
```

Или через nested.

### 3.6. fixed-size buffer в struct (C# 7.3+)

```csharp
unsafe struct Buffer128
{
    public fixed byte Data[128];   // inline array внутри struct
}

unsafe
{
    var buf = new Buffer128();
    buf.Data[0] = 1;
    buf.Data[127] = 255;
    
    fixed (byte* ptr = buf.Data)
    {
        // pointer to data
    }
}
```

`fixed` field — **inline array** в struct memory. Used для C-style structs с embedded arrays.

### 3.7. Performance pinning

```
| Operation | Time |
|-----------|------|
| fixed entry/exit | ~5ns |
| GC.AllocateUninitialized + Pin (Pinned object heap) | ~50ns init |
| GCHandle.Alloc(arr, GCHandleType.Pinned) | ~100ns |
```

`fixed` — fastest для short-lived pinning. **Pinned Object Heap (POH)** — для long-lived objects (.NET 5+).

### 3.8. Don't pin too long

```csharp
// ❌ Pin для long-running operation — GC не может efficient compact
fixed (byte* ptr = veryLargeArray)
{
    while (true)   // long loop
    {
        await SomeAsync();   // ❌ + fixed не allowed across await!
    }
}
```

Pinning — **GC pause hazard** при large objects. Best: short-lived, в tight loops.

> [!question]- Интервью: зачем нужен `fixed` keyword?
> GC moves managed objects во время compaction. Pointer to managed object **becomes invalid** после move. `fixed (T* ptr = obj)` — **pin** object: GC не moves пока в scope. Use cases: 1) Pointer arithmetic над array/string (managed memory). 2) P/Invoke с pointer parameters. 3) Interop с native libs. **Performance**: `fixed` cheap (~5ns entry/exit). **Pitfalls**: 1) Не pin long-living — GC compacts inefficient. 2) `fixed` не cross await boundary. 3) Pinned Object Heap (.NET 5+) для long-lived. Alternative: `Span<T>` через `MemoryMarshal.GetReference` — без pinning для most cases.

---

## 4. stackalloc deep

### 4.1. Базовый stackalloc

```csharp
unsafe
{
    int* buffer = stackalloc int[100];
    for (int i = 0; i < 100; i++)
        buffer[i] = i * i;
}
```

`stackalloc T[n]` — allocate `n * sizeof(T)` bytes на stack. **No GC**, very fast (~5ns).

### 4.2. `Span<T>` stackalloc (C# 7.2+, no unsafe needed!)

```csharp
// Без unsafe!
Span<int> buffer = stackalloc int[100];
for (int i = 0; i < 100; i++)
    buffer[i] = i * i;
```

`stackalloc` returning `Span<T>` — safe wrapper. **Не нужен `unsafe`** keyword. Recommended approach 2024+.

### 4.3. Lifetime — current method

```csharp
unsafe int* GetBuffer()   // ❌ возвращаешь pointer на stack memory!
{
    int* buf = stackalloc int[100];
    return buf;   // UB — caller использует освобождённую память
}
```

Stack memory **freed when method returns**. Pointer becomes invalid. **Never return** stackalloc pointer.

`Span<T>` enforces compile-time:

```csharp
Span<int> Bad()
{
    Span<int> span = stackalloc int[100];
    return span;   // ❌ Compile error — span captures stack-only ref
}
```

Compiler prevents.

### 4.4. Stack overflow risk

```csharp
unsafe void Bad(int size)
{
    int* buf = stackalloc int[size];   // ❌ size = huge → stack overflow!
}
```

Stack — typically 1MB. `stackalloc int[300_000]` ~ 1.2MB → crash.

**Best practice:** check threshold:

```csharp
const int MaxStackSize = 256;   // элементов

Span<int> buffer = size <= MaxStackSize
    ? stackalloc int[MaxStackSize]
    : new int[size];
```

### 4.5. SkipLocalsInit (.NET 5+)

```csharp
[SkipLocalsInit]
unsafe void Fast()
{
    int* buf = stackalloc int[100];   // не zero-initialized!
    // Faster — но careful, может быть garbage
}
```

`[SkipLocalsInit]` — skip zero-init. Faster, но buffer contains garbage.

### 4.6. Inline arrays (C# 12+)

```csharp
[System.Runtime.CompilerServices.InlineArray(128)]
public struct Buffer128
{
    private byte _element0;
}

void Use()
{
    Buffer128 buf = default;
    buf[0] = 1;
    buf[127] = 255;
    Span<byte> span = buf;   // implicit conversion
}
```

Inline arrays — modern alternative `fixed` buffers. Fully safe, supports any element type.

### 4.7. stackalloc в expressions (C# 8+)

```csharp
// До C# 8 — только в local
int* buf = stackalloc int[100];

// C# 8+ — в expressions
ProcessSpan(stackalloc int[100]);   // как argument
```

> [!question]- Интервью: чем `stackalloc Span<T>` отличается от `stackalloc T*`?
> **`stackalloc T*`** — raw pointer, требует `unsafe`. Manual memory access, no bounds checks. **`stackalloc Span<T>`** (C# 7.2+) — safe wrapper, не нужен `unsafe`. Bounds checks, compile-time prevent return из method (compiler error). Same allocation underneath. Best practice 2024+: **always `Span<T>`**. Reasons: 1) Type safety. 2) No unsafe context. 3) Bounds checks. 4) Lifetime checked compile-time. 5) Same performance. Pitfalls: stack overflow если size huge — check threshold (~1KB safe).

---

## 5. P/Invoke и marshaling

### 5.1. DllImport (legacy)

```csharp
using System.Runtime.InteropServices;

public static class NativeMethods
{
    [DllImport("user32.dll", CharSet = CharSet.Unicode)]
    public static extern int MessageBox(
        IntPtr hWnd, string text, string caption, uint type);
}

// Use
NativeMethods.MessageBox(IntPtr.Zero, "Hello", "Title", 0);
```

`[DllImport]` — declare native function. CLR generates marshaling code.

### 5.2. LibraryImport (.NET 7+ source generator)

```csharp
public static partial class NativeMethods
{
    [LibraryImport("user32.dll", StringMarshalling = StringMarshalling.Utf16)]
    public static partial int MessageBox(
        IntPtr hWnd, string text, string caption, uint type);
}
```

`[LibraryImport]` — source generator produces P/Invoke wrapper code. **AOT-compatible**, type-safe marshalling, faster than `[DllImport]`.

Best practice 2024+: **always `[LibraryImport]`** в new code.

### 5.3. Calling conventions

```csharp
[DllImport("kernel32.dll", CallingConvention = CallingConvention.StdCall)]
static extern bool CloseHandle(IntPtr handle);

// CallingConvention values:
// - StdCall (Windows API default)
// - Cdecl (C default)
// - ThisCall (C++ member methods)
// - FastCall
// - Winapi (platform default — StdCall on Windows)
```

Match native function's calling convention or crash.

### 5.4. Marshaling — string types

```csharp
[DllImport("lib.dll", CharSet = CharSet.Unicode)]
static extern void Method1(string s);   // UTF-16

[DllImport("lib.dll", CharSet = CharSet.Ansi)]
static extern void Method2(string s);   // ANSI (locale-specific)

[LibraryImport("lib.dll", StringMarshalling = StringMarshalling.Utf8)]
static partial void Method3(string s);   // UTF-8 (.NET 7+)

// StringBuilder для output
[DllImport("lib.dll")]
static extern void GetText([Out] StringBuilder buffer, int size);
```

### 5.5. Marshaling — structs

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct Point
{
    public int X;
    public int Y;
}

[DllImport("user32.dll")]
static extern bool GetCursorPos(out Point pt);

Point p;
GetCursorPos(out p);
```

`[StructLayout(LayoutKind.Sequential)]` — fields in declaration order. `LayoutKind.Explicit` + `[FieldOffset(n)]` для unions.

### 5.6. Marshaling — arrays

```csharp
[DllImport("lib.dll")]
static extern void ProcessArray(
    [In, Out] int[] data,
    int length);

// SafeArray (COM)
[DllImport("lib.dll")]
static extern void ProcessSafe(
    [MarshalAs(UnmanagedType.SafeArray, SafeArraySubType = VarEnum.VT_I4)]
    int[] data);
```

### 5.7. Marshal class — manual

```csharp
// Allocate unmanaged memory
IntPtr ptr = Marshal.AllocHGlobal(1024);
try
{
    // Use ptr...
    Marshal.WriteInt32(ptr, 0, 42);
    int x = Marshal.ReadInt32(ptr, 0);
    
    // String conversion
    IntPtr str = Marshal.StringToHGlobalUni("hello");
    string s = Marshal.PtrToStringUni(str);
    Marshal.FreeHGlobal(str);
}
finally
{
    Marshal.FreeHGlobal(ptr);
}
```

`Marshal` class — full control: allocate, copy, convert.

### 5.8. SafeHandle — RAII для native handles

```csharp
public class FileHandle : SafeHandle
{
    public FileHandle() : base(IntPtr.Zero, ownsHandle: true) { }
    
    public override bool IsInvalid => handle == IntPtr.Zero;
    
    protected override bool ReleaseHandle()
    {
        return CloseHandle(handle);
    }
    
    [DllImport("kernel32.dll")]
    static extern bool CloseHandle(IntPtr h);
}

// Use
[DllImport("kernel32.dll")]
static extern FileHandle CreateFile(...);

using FileHandle h = CreateFile(...);
// Auto release via Dispose pattern
```

`SafeHandle` — managed wrapper над native handle. Guaranteed release при finalize/dispose.

> [!question]- Интервью: чем `[LibraryImport]` лучше `[DllImport]`?
> **`[DllImport]`** (legacy) — runtime IL stubs для marshalling, slower, **не AOT-compatible** (Native AOT нельзя generate stubs at runtime). **`[LibraryImport]`** (.NET 7+) — **source generator** generates marshalling code at compile-time. Преимущества: 1) **AOT-friendly** (essential для .NET 8+ Native AOT). 2) **Type-safe marshalling** — compile-time check. 3) **Faster** — no runtime IL generation. 4) **Cleaner errors** — generator reports issues compile-time. 5) **Custom marshallers** через `[CustomMarshaller]`. Best practice: always `[LibraryImport]` new code, migrate `[DllImport]` gradually.

---

## 6. Buffer.MemoryCopy и low-level ops

### 6.1. MemoryCopy

```csharp
unsafe
{
    byte[] src = new byte[1024];
    byte[] dst = new byte[1024];
    
    fixed (byte* pSrc = src)
    fixed (byte* pDst = dst)
    {
        Buffer.MemoryCopy(pSrc, pDst, dst.Length, src.Length);
    }
}
```

`Buffer.MemoryCopy` — fast byte copy. Equivalent C `memcpy`. SIMD-optimized.

### 6.2. Span CopyTo (safe alternative)

```csharp
ReadOnlySpan<byte> src = ...;
Span<byte> dst = ...;
src.CopyTo(dst);   // safe + fast
```

`Span<T>.CopyTo` — same performance, no unsafe.

### 6.3. Unsafe.As — reinterpret cast

```csharp
using System.Runtime.CompilerServices;

byte[] bytes = new byte[4];
int value = Unsafe.As<byte, int>(ref bytes[0]);
// Reinterpret first 4 bytes как int
```

`Unsafe.As<TFrom, TTo>` — reinterpret memory как другой type. **No copy**, no allocation.

### 6.4. Unsafe.Read / Write

```csharp
unsafe
{
    byte* ptr = stackalloc byte[16];
    int value = Unsafe.Read<int>(ptr);
    Unsafe.Write(ptr + 4, 42);
}
```

### 6.5. MemoryMarshal — Span manipulation

```csharp
ReadOnlySpan<byte> bytes = ...;
ReadOnlySpan<int> ints = MemoryMarshal.Cast<byte, int>(bytes);
// Re-interpret bytes как ints (length / 4)

Span<byte> byteSpan = MemoryMarshal.AsBytes<int>(intSpan);
// int[] → byte view

ref int firstInt = ref MemoryMarshal.GetReference(intSpan);
```

`MemoryMarshal` — safe-ish low-level ops on Span. Bridge между managed и unsafe.

### 6.6. BinaryPrimitives — endianness

```csharp
using System.Buffers.Binary;

byte[] buf = new byte[4];
BinaryPrimitives.WriteInt32LittleEndian(buf, 42);
BinaryPrimitives.WriteInt32BigEndian(buf, 42);

int v1 = BinaryPrimitives.ReadInt32LittleEndian(buf);
int v2 = BinaryPrimitives.ReadInt32BigEndian(buf);
```

Cross-platform endian-aware reads/writes без unsafe.

> [!question]- Интервью: чем `Unsafe.As` отличается от cast `(int)bytes[0]`?
> **`(int)bytes[0]`** — value conversion, byte → int. Single byte → int с zero extension. **`Unsafe.As<byte, int>(ref bytes[0])`** — **reinterpret memory** as int. Reads 4 bytes starting from `bytes[0]` and treats them as int32. **No copy**, no conversion. Use case: reading struct from byte buffer (network protocols, file formats), zero-allocation parsing. Trade-offs: undefined behavior если alignment wrong (some platforms), endianness-dependent (use BinaryPrimitives для explicit), can read past array bounds. Best practice: validate bounds, use Unsafe.As для perf-critical reinterpretation.

---

## 7. Function pointers (C# 9+)

### 7.1. Зачем

Replacement для delegate в high-performance interop:

```csharp
unsafe
{
    delegate*<int, int, int> add = &Add;
    int result = add(2, 3);   // 5
}

static int Add(int a, int b) => a + b;
```

Function pointer — direct address of method. **No delegate object, no allocation, no virtual call**.

### 7.2. Performance vs delegate

```
| Operation | delegate | function pointer |
|-----------|----------|-------------------|
| Allocation | yes (heap) | no |
| Indirection | virtual call | direct call |
| Per-call cost | ~5ns | ~1ns |
| Multicast | supported | not supported |
```

5x faster + zero allocation.

### 7.3. Calling conventions

```csharp
unsafe
{
    // Default — managed convention
    delegate*<int, int> managed = &MethodA;
    
    // Native conventions
    delegate* unmanaged[Cdecl]<int, int> cdecl = &CdeclMethod;
    delegate* unmanaged[Stdcall]<int, int> stdcall = &StdcallMethod;
    delegate* unmanaged[Thiscall]<int, int> thiscall = &ThiscallMethod;
}
```

For native interop — match native calling convention.

### 7.4. UnmanagedCallersOnly

```csharp
[UnmanagedCallersOnly(CallConvs = new[] { typeof(CallConvCdecl) })]
public static int CallbackFromNative(int x)
{
    return x * 2;
}

// Native code can call this как C function
delegate* unmanaged[Cdecl]<int, int> ptr = &CallbackFromNative;
```

`[UnmanagedCallersOnly]` — managed method callable from native. AOT-compatible.

### 7.5. Limitations

```csharp
// ❌ Только static methods
delegate*<int, int> ptr = &instance.Method;   // Compile error

// ❌ No multicast
ptr += anotherPtr;   // Compile error

// ❌ No generic type parameters
delegate*<T, T> ptr;   // Compile error

// ❌ Requires unsafe
delegate*<int, int> ptr = &Add;   // Outside unsafe — error
```

Function pointers — **niche** feature для perf hot paths и interop.

### 7.6. Used в .NET runtime

`Span<T>.CopyTo`, `Buffer.MemoryCopy`, `String.IndexOf` — internally используют function pointers для dispatch. User code rarely needs.

> [!question]- Интервью: когда function pointer вместо delegate?
> **Function pointer** — direct method address, **no delegate object** (zero heap allocation). Calls direct (no virtual lookup). 5x faster ~1ns vs ~5ns delegate. Use cases: 1) **Native interop** (callbacks из C/C++). 2) **Hot path performance** (parser, dispatcher tables). 3) **Native AOT scenarios**. **Limitations**: только static methods, no multicast, no generic type params, requires `unsafe`. **Delegate** — full-featured: instance methods, multicast, captures (closures), variance, supports async/await. Best practice: 99% case — delegate. Function pointer — measured perf bottleneck или native callback only.

---

## 8. Native AOT и unsafe

### 8.1. AOT requires unsafe-friendly code

Native AOT (.NET 8+) — ahead-of-time compilation, no JIT, no reflection (mostly):

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

```bash
dotnet publish -c Release -r win-x64
```

### 8.2. Что меняется

| Feature | Regular .NET | Native AOT |
|---------|--------------|------------|
| Reflection | Full | Limited (analyzer warnings) |
| Dynamic code gen | Yes (System.Reflection.Emit) | No |
| Pluggable assemblies | Yes | No (single binary) |
| Source generators | Optional | **Essential** |
| `[DllImport]` | OK | OK but `[LibraryImport]` better |
| `unsafe` | Same | Same — first-class |

### 8.3. Source generators replace reflection

```csharp
// Before AOT — reflection-based
JsonSerializer.Serialize(user);

// AOT-friendly — source generator
[JsonSerializable(typeof(User))]
public partial class MyContext : JsonSerializerContext { }

JsonSerializer.Serialize(user, MyContext.Default.User);
```

### 8.4. unsafe в AOT — beneficial

AOT compilation — closer to C++ performance. Combined с unsafe:
- Pointer arithmetic — direct machine code.
- No JIT overhead.
- Predictable performance.
- Smaller binary.

Use cases: embedded, IoT, CLI tools, microservices.

### 8.5. Trade-offs

```
✅ AOT pros:
  - Faster startup (no JIT)
  - Smaller memory footprint
  - Single binary deployment
  - Predictable performance

❌ AOT cons:
  - Reflection limited
  - No runtime code gen
  - Larger binary size (no shared framework)
  - Slower throughput initially (JIT may catch up)
  - Build takes longer
```

> [!question]- Интервью: как Native AOT relate to unsafe code?
> Native AOT — ahead-of-time compilation, no JIT. **`unsafe`** code — first-class в AOT (compiled directly to machine code). **Reflection** ограничен — `[LibraryImport]` source generator вместо `[DllImport]`, `JsonSerializerContext` вместо runtime reflection-based serialization. **Function pointers** + `[UnmanagedCallersOnly]` — essential для callbacks. AOT particularly suited для CLI tools, microservices, IoT. Trade-offs: faster startup + smaller memory, но reflection-heavy code требует rewrite via source generators. `unsafe` patterns become more important в AOT — performance closer to C++.

---

## 9. Best practices

### 9.1. Avoid unsafe когда возможно

- ✅ **`Span<T>`** для buffer manipulation.
- ✅ **`stackalloc Span<T>`** для stack buffers.
- ✅ **`Memory<T>`** для async paths.
- ✅ **`MemoryMarshal`** для reinterpretation.
- ✅ **`BinaryPrimitives`** для endianness.
- ❌ Pointer arithmetic — Span с indexer.

### 9.2. Когда unsafe оправдан

- ✅ **P/Invoke** с native libraries (use `[LibraryImport]`).
- ✅ **Image / audio processing** — pixel access.
- ✅ **Custom serializers / parsers** — measured 2x+ improvement.
- ✅ **SafeHandle wrappers** для native resources.
- ✅ **Function pointers** для callbacks из native.

### 9.3. Pinning patterns

- ✅ **Short-lived `fixed` blocks**.
- ✅ **Pinned Object Heap** (.NET 5+) для long-lived.
- ✅ **`SafeHandle`** для native resources с RAII.
- ❌ Long-lived `fixed` — GC compaction inefficient.
- ❌ Pin large arrays in hot path.

### 9.4. Marshaling

- ✅ **`[LibraryImport]`** в .NET 7+.
- ✅ **`[StructLayout(Sequential)]`** для C-compatible structs.
- ✅ **`StringMarshalling.Utf8`** для cross-platform.
- ✅ **`SafeHandle`** для resource lifetime.
- ❌ **`[DllImport]`** в new code — `[LibraryImport]` лучше.
- ❌ Manual `Marshal.AllocHGlobal` без try/finally.

### 9.5. Не делай

- ❌ Return stackalloc pointer из method.
- ❌ Pin object на duration async operation.
- ❌ Modify string через char* (UB).
- ❌ Pointer arithmetic past buffer bounds.
- ❌ Mix managed/unmanaged без careful review.
- ❌ unsafe в production без profiling proven benefit.

---

## 10. Decision tree

```
Нужен low-level access?
│
├── Buffer manipulation
│   ├── Read / write — Span<T> / Memory<T>
│   ├── Slicing — Span<T>.Slice
│   ├── Stack allocation — stackalloc Span<T>
│   └── Reinterpret — MemoryMarshal.Cast<TFrom, TTo>
│
├── Native interop (C/C++)
│   ├── Functions → [LibraryImport] (.NET 7+)
│   ├── Structs → [StructLayout(Sequential)]
│   ├── Strings → StringMarshalling.Utf8/Utf16
│   ├── Handles → SafeHandle wrapper
│   └── Callbacks → [UnmanagedCallersOnly] + function pointer
│
├── Performance hot path
│   ├── Profile measure
│   ├── 2x+ improvement → consider unsafe
│   ├── Less → optimize via algorithms
│   └── Function pointer → measure vs delegate
│
└── Endianness / binary parsing
    ├── BinaryPrimitives — Read/WriteInt32BigEndian etc.
    ├── Unsafe.As<byte, int> — reinterpret
    └── BitConverter — legacy, platform-dependent
```

---

## 11. Cheat sheet

```csharp
// === Project setup ===
// <PropertyGroup>
//   <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
// </PropertyGroup>

// === Pointers ===
unsafe
{
    int x = 42;
    int* p = &x;
    *p = 100;
    
    int* arr = stackalloc int[10];
    arr[0] = 1;
    
    int sz = sizeof(MyStruct);
}

// === fixed ===
unsafe
{
    byte[] data = new byte[1024];
    fixed (byte* ptr = data)
    {
        // pointer arithmetic
        for (int i = 0; i < data.Length; i++)
            ptr[i] = (byte)i;
    }
}

// === stackalloc Span<T> (safe!) ===
Span<int> buffer = stackalloc int[100];
buffer[0] = 42;

// Hybrid stack/heap
Span<int> buf = size <= 256
    ? stackalloc int[256]
    : new int[size];

// === LibraryImport (.NET 7+) ===
public partial class NativeMethods
{
    [LibraryImport("user32.dll", StringMarshalling = StringMarshalling.Utf16)]
    public static partial int MessageBox(
        IntPtr hWnd, string text, string caption, uint type);
}

// === StructLayout ===
[StructLayout(LayoutKind.Sequential)]
public struct Point
{
    public int X;
    public int Y;
}

// === SafeHandle ===
public class MyHandle : SafeHandle
{
    public MyHandle() : base(IntPtr.Zero, true) { }
    public override bool IsInvalid => handle == IntPtr.Zero;
    protected override bool ReleaseHandle() => CloseHandle(handle);
}

// === Function pointer ===
unsafe
{
    delegate*<int, int, int> add = &AddStatic;
    int r = add(2, 3);
}
static int AddStatic(int a, int b) => a + b;

// === MemoryMarshal ===
ReadOnlySpan<byte> bytes = ...;
ReadOnlySpan<int> ints = MemoryMarshal.Cast<byte, int>(bytes);

// === BinaryPrimitives ===
using System.Buffers.Binary;
int v = BinaryPrimitives.ReadInt32BigEndian(bytes);
BinaryPrimitives.WriteInt32LittleEndian(buffer, 42);

// === Unsafe.As (reinterpret) ===
byte[] data = new byte[8];
int value = Unsafe.As<byte, int>(ref data[0]);

// === Marshal ===
IntPtr ptr = Marshal.AllocHGlobal(1024);
try { Marshal.WriteInt32(ptr, 0, 42); }
finally { Marshal.FreeHGlobal(ptr); }
```

---

## 12. Common pitfalls

### 12.1. Return stackalloc pointer

```csharp
unsafe int* Bad()
{
    int* p = stackalloc int[10];
    return p;   // ❌ stack memory freed
}
```

**Фикс:** return `Span<T>` (compiler prevents) или allocate на heap.

### 12.2. Pin too long

```csharp
fixed (byte* ptr = hugeArray)
{
    Thread.Sleep(60_000);   // ❌ blocks GC compaction 1 minute
}
```

**Фикс:** short fixed scope, или Pinned Object Heap.

### 12.3. fixed cross await

```csharp
public async Task M()
{
    fixed (byte* ptr = data)   // ❌ Compile error
    {
        await Task.Delay(100);
    }
}
```

**Фикс:** restructure — sync work внутри fixed, async снаружи.

### 12.4. Modify string через char*

```csharp
unsafe
{
    string s = "hello";
    fixed (char* ptr = s)
    {
        ptr[0] = 'H';   // ❌ undefined — strings immutable, may share с interned
    }
}
```

**Фикс:** `StringBuilder` или `char[]`.

### 12.5. Buffer overflow

```csharp
unsafe
{
    int* buf = stackalloc int[10];
    for (int i = 0; i < 100; i++)
        buf[i] = i;   // ❌ writes past buffer — corruption
}
```

**Фикс:** explicit length checks, prefer `Span<T>` (bounds-checked).

### 12.6. Wrong calling convention

```csharp
[DllImport("lib.dll", CallingConvention = CallingConvention.Cdecl)]
static extern int Foo(int x);

// Native function actually уses StdCall — stack corruption!
```

**Фикс:** verify native function's convention.

### 12.7. Marshal.AllocHGlobal без free

```csharp
IntPtr ptr = Marshal.AllocHGlobal(1024);
// ... work, throws
// memory leaked!
```

**Фикс:** try/finally или wrap в SafeHandle.

### 12.8. Sizeof managed type

```csharp
unsafe
{
    int sz = sizeof(string);   // ❌ Compile error — managed type
}
```

**Фикс:** `sizeof` только для unmanaged types.

### 12.9. Function pointer на instance method

```csharp
unsafe
{
    delegate*<int, int> ptr = &instance.Method;   // ❌ instance method not allowed
}
```

**Фикс:** static methods only.

### 12.10. Wrong endianness

```csharp
unsafe
{
    byte[] buf = { 0, 0, 0, 1 };
    int value = *(int*)&buf[0];   // ❌ depends on platform endianness
}
```

**Фикс:** `BinaryPrimitives.ReadInt32BigEndian` или explicit conversion.

> [!question]- Интервью: топ-3 ошибки с unsafe?
> 1) **Return stackalloc pointer** — stack memory freed после method return, caller использует invalid memory. Compile error для `Span<T>` (good!), но pointer escapes silently. 2) **Pin too long** — `fixed` block с долгой operation blocks GC compaction. Use Pinned Object Heap (.NET 5+) или short scopes. 3) **Buffer overflow** — pointer arithmetic past array bounds. No bounds check (unlike `Span<T>`). Always validate length, prefer `Span<T>`. Бонус: wrong calling convention в P/Invoke — stack corruption silent.

---

## 13. Practice exercises

### 13.1. Fast XOR cipher

```csharp
public static unsafe void XorEncrypt(Span<byte> data, byte key)
{
    fixed (byte* ptr = data)
    {
        // Process 8 bytes at a time
        ulong key8 = key | ((ulong)key << 8) | ((ulong)key << 16) | ((ulong)key << 24)
                       | ((ulong)key << 32) | ((ulong)key << 40) | ((ulong)key << 48) | ((ulong)key << 56);
        
        ulong* p64 = (ulong*)ptr;
        int chunks = data.Length / 8;
        for (int i = 0; i < chunks; i++)
            p64[i] ^= key8;
        
        // Tail
        for (int i = chunks * 8; i < data.Length; i++)
            ptr[i] ^= key;
    }
}

// Use
byte[] data = ...;
XorEncrypt(data, 0x42);
```

### 13.2. SafeHandle wrapper

```csharp
public sealed class WindowHandle : SafeHandle
{
    public WindowHandle() : base(IntPtr.Zero, ownsHandle: true) { }
    
    public WindowHandle(IntPtr existing) : base(IntPtr.Zero, true)
    {
        SetHandle(existing);
    }
    
    public override bool IsInvalid => handle == IntPtr.Zero;
    
    protected override bool ReleaseHandle()
    {
        return DestroyWindow(handle);
    }
    
    [LibraryImport("user32.dll")]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static partial bool DestroyWindow(IntPtr hWnd);
}

// Use — auto-cleanup
using WindowHandle wnd = CreateWindow(...);
```

### 13.3. Custom struct serializer

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct PacketHeader
{
    public ushort Type;
    public ushort Length;
    public uint Sequence;
    public ulong Timestamp;
}

public static class PacketSerializer
{
    public static unsafe void WriteHeader(Span<byte> buffer, in PacketHeader header)
    {
        if (buffer.Length < sizeof(PacketHeader))
            throw new ArgumentException("Buffer too small");
        
        fixed (byte* dst = buffer)
        fixed (PacketHeader* src = &header)
        {
            Buffer.MemoryCopy(src, dst, buffer.Length, sizeof(PacketHeader));
        }
    }
    
    public static unsafe PacketHeader ReadHeader(ReadOnlySpan<byte> buffer)
    {
        if (buffer.Length < sizeof(PacketHeader))
            throw new ArgumentException("Buffer too small");
        
        PacketHeader header;
        fixed (byte* src = buffer)
        {
            Buffer.MemoryCopy(src, &header, sizeof(PacketHeader), sizeof(PacketHeader));
        }
        return header;
    }
}

// Modern alternative — без unsafe
public static class PacketSerializerSafe
{
    public static void WriteHeader(Span<byte> buffer, in PacketHeader header)
    {
        MemoryMarshal.Write(buffer, in header);
    }
    
    public static PacketHeader ReadHeader(ReadOnlySpan<byte> buffer)
    {
        return MemoryMarshal.Read<PacketHeader>(buffer);
    }
}
```

`MemoryMarshal.Read/Write` — same effect без `unsafe`.

---

## 14. Что читать дальше

1. **[[memory-pooling|Memory Pooling]]** — Span/Memory deep.
2. **[[types-and-memory|Types and Memory]]** — GC, allocation.
3. **Native AOT documentation**.
4. **System.Runtime.CompilerServices.Unsafe API**.
5. **P/Invoke specification**.

---

## 15. См. также

- [[memory-pooling|Memory Pooling]] — `Span<T>`, stackalloc
- [[types-and-memory|Types and Memory]] — GC, Pinned Object Heap
- [[source-generators|Source Generators]] — LibraryImport
- System.Runtime.InteropServices — Marshal class
- System.Runtime.CompilerServices.Unsafe

---

## 16. Reading list

- **Microsoft Docs — Unsafe code** — learn.microsoft.com/dotnet/csharp/language-reference/unsafe-code
- **Microsoft Docs — Function pointers** — learn.microsoft.com/dotnet/csharp/language-reference/proposals/csharp-9.0/function-pointers
- **Microsoft Docs — P/Invoke** — learn.microsoft.com/dotnet/standard/native-interop/pinvoke
- **Microsoft Docs — LibraryImport** — learn.microsoft.com/dotnet/standard/native-interop/source-generated-marshalling
- **Microsoft Docs — Native AOT** — learn.microsoft.com/dotnet/core/deploying/native-aot/
- **Adam Sitnik — Pointers in C# 9** — adamsitnik.com
- **Stephen Toub — Performance** (devblogs.microsoft.com)
- **Konrad Kokosa — Pro .NET Memory Management** (book)
- **Marek Sarnowski — Calling C from C#** — michaelscodingspot.com
