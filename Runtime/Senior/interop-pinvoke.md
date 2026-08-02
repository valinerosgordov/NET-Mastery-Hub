---
tags: [interop, pinvoke, com, marshalling, libraryimport, safehandle, native, com-interop]
level: Senior
date: 2026-04-30
---

# Interop: P/Invoke, COM, Marshalling

> Полный гайд по работе с native кодом из .NET. Закрывает: P/Invoke, `LibraryImport` source generator (.NET 7+), marshalling типов, SafeHandle, COM Interop, calling conventions, AOT-compat, common pitfalls. Critical для WPF/desktop, native libraries (CUDA, OpenCV, ffmpeg), Windows API, MetaTrader, hardware integration.

---

## Что это, зачем и когда

### Что такое Interop?
**Способность managed (.NET) кода вызывать native (C/C++/COM) код** и наоборот. Native — это код напрямую исполняемый CPU без CLR.

**Аналогия:** Переводчик между двумя языками. .NET говорит на C# → переводчик (P/Invoke / COM Interop) → native говорит на C/C++. Marshalling — это правила перевода типов.

### Зачем?

| Сценарий | Без interop | С interop |
|----------|-------------|-----------|
| Использовать ffmpeg для видео | Писать кодек на C# | `[LibraryImport("ffmpeg")]` — миллион строк готового кода |
| Windows API (registry, services) | Невозможно | `kernel32.dll`, `user32.dll` через P/Invoke |
| OpenCV для computer vision | Написать с нуля | Emgu.CV / OpenCvSharp обёртки |
| MetaTrader 5 для trading | Нет API на C# | COM Interop с MT5 |
|硬件 (sensors, devices) | Нет драйверов | Native C SDK + P/Invoke |
| GPU (CUDA, DirectML) | Невозможно | Native bindings |

### Когда что использовать

| Инструмент | Когда |
|-----------|-------|
| **`[DllImport]`** (legacy) | Простой P/Invoke до .NET 7 |
| **`[LibraryImport]`** (.NET 7+) | Modern P/Invoke с source generator — **default для нового кода** |
| **`SafeHandle`** | Любые native handles (IntPtr) — обязательно для GC safety |
| **COM Interop (`ComImport`)** | Office, MetaTrader, Windows shell, ActiveX |
| **`[GeneratedComInterface]`** (.NET 8+) | COM с source generator, AOT-friendly |
| **`UnmanagedCallersOnly`** (.NET 5+) | Callbacks из native в managed |
| **`NativeAOT` экспорт** | Сборка native library на C# для использования из C/Python |

---

## 1. P/Invoke — основы

### Простой вызов Windows API

```csharp
using System.Runtime.InteropServices;

class Program
{
    // Legacy way (.NET до 7)
    [DllImport("user32.dll", CharSet = CharSet.Unicode, SetLastError = true)]
    private static extern int MessageBox(IntPtr hWnd, string text, string caption, uint type);
    
    static void Main()
    {
        MessageBox(IntPtr.Zero, "Hello from .NET!", "Caption", 0);
    }
}
```

### `LibraryImport` — modern way (.NET 7+)

```csharp
using System.Runtime.InteropServices;

partial class Program
{
    // Modern — source generator вместо runtime marshalling
    [LibraryImport("user32.dll", StringMarshalling = StringMarshalling.Utf16, SetLastError = true)]
    private static partial int MessageBox(IntPtr hWnd, string text, string caption, uint type);
    
    static void Main()
    {
        MessageBox(IntPtr.Zero, "Hello!", "Title", 0);
    }
}
```

### `DllImport` vs `LibraryImport` — что лучше

| | `DllImport` | `LibraryImport` |
|--|-------------|-----------------|
| Когда | .NET Framework, .NET до 7 | **.NET 7+** |
| Marshalling | Runtime (через JIT-stubs) | Compile-time (source generator) |
| AOT-friendly | ❌ | ✅ |
| Performance | Чуть медленнее | Чуть быстрее (no JIT cost) |
| Readability | Implicit marshalling | Явный, видно что генерируется |
| Method modifier | `static extern` | `static partial` |

**Используй `LibraryImport` для нового кода.** `DllImport` — только legacy.

---

## 2. Marshalling — правила перевода типов

### Blittable типы — без marshalling

Типы у которых memory layout одинаковый в .NET и C/C++ → передаются без преобразования (zero overhead).

| Blittable | Не blittable |
|-----------|-------------|
| `byte`, `sbyte` | `bool` (4 bytes в .NET, 1 в C) |
| `short`, `ushort` | `string` (нужна конвертация encoding) |
| `int`, `uint`, `long`, `ulong` | `char` (UTF-16 в .NET) |
| `float`, `double` | `decimal` (custom layout) |
| `IntPtr`, `UIntPtr` | `DateTime` |
| `nint`, `nuint` (.NET 5+) | `object` |
| Указатели (unsafe) | Массивы (нужно знать длину) |
| Struct из blittable полей | Class (managed objects) |

### String marshalling

```csharp
// LibraryImport — explicit StringMarshalling
[LibraryImport("native.dll", StringMarshalling = StringMarshalling.Utf8)]
private static partial int ProcessString(string input);

// Doors per encoding:
// StringMarshalling.Utf8        — для Linux/POSIX и многих Linux libraries
// StringMarshalling.Utf16       — Windows API (W functions: MessageBoxW)
// StringMarshalling.Custom      — кастомный marshaller

// Custom marshaller для UTF-8 в .NET 6:
[DllImport("native.dll")]
private static extern int Process(
    [MarshalAs(UnmanagedType.LPUTF8Str)] string input);
```

### Struct marshalling

```csharp
// C struct
// typedef struct {
//     int x;
//     int y;
//     char name[32];
// } Point;

// .NET equivalent
[StructLayout(LayoutKind.Sequential, CharSet = CharSet.Ansi, Pack = 1)]
public struct Point
{
    public int X;
    public int Y;
    
    [MarshalAs(UnmanagedType.ByValTStr, SizeConst = 32)]
    public string Name;
}

[LibraryImport("native.dll")]
private static partial void ProcessPoint(ref Point p);

// Использование
var point = new Point { X = 1, Y = 2, Name = "Hello" };
ProcessPoint(ref point);
```

### LayoutKind — варианты

| Layout | Когда |
|--------|-------|
| `Sequential` | Поля в порядке объявления — стандарт для interop |
| `Explicit` | Точное расположение через `[FieldOffset]` — union, специфичные форматы |
| `Auto` (default for class) | CLR может реordered поля — **не использовать** для interop |

### Pack — alignment

```csharp
// Без Pack (default) — natural alignment по platform (8 на x64)
[StructLayout(LayoutKind.Sequential)]
struct A { byte b; int i; }  // sizeof = 8 (3 bytes padding)

// Pack=1 — packed, без padding
[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct B { byte b; int i; }  // sizeof = 5
```

`Pack = 1` критично когда native структура packed (чаще всего сетевые протоколы). Иначе — оставить default.

### Massive marshalling

```csharp
// C function: void process(int* arr, int length)

// LibraryImport
[LibraryImport("native.dll")]
private static partial void Process(
    [In] int[] arr,
    int length);

// Использование
int[] data = [1, 2, 3, 4, 5];
Process(data, data.Length);

// In/Out direction
[LibraryImport("native.dll")]
private static partial void Fill([In, Out] int[] buffer, int size);
// Для buffer который заполняется native кодом
```

### `Span<T>` в P/Invoke (.NET 7+)

```csharp
[LibraryImport("native.dll")]
private static partial int ProcessBuffer(ReadOnlySpan<byte> data, int length);

byte[] buffer = new byte[1024];
ProcessBuffer(buffer, buffer.Length);

// stackalloc — zero allocation interop
Span<byte> stack = stackalloc byte[256];
ProcessBuffer(stack, stack.Length);
```

---

## 3. SafeHandle — безопасное управление native handles

### Зачем

Native handles (file handles, socket handles, GDI handles) — **критичный resource**. Утечка → exhaustion системных ресурсов.

```csharp
// ❌ IntPtr — нет автоматической очистки
[LibraryImport("kernel32.dll", SetLastError = true)]
private static partial IntPtr CreateFileW(string filename, uint access, uint share, ...);

[LibraryImport("kernel32.dll")]
private static partial int CloseHandle(IntPtr handle);

IntPtr file = CreateFileW("test.txt", ...);
// Если exception — CloseHandle не вызовется → leak
CloseHandle(file);
```

### SafeHandle — RAII pattern

```csharp
public sealed class SafeFileHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    public SafeFileHandle() : base(ownsHandle: true) { }
    
    public SafeFileHandle(IntPtr handle, bool ownsHandle) : base(ownsHandle)
    {
        SetHandle(handle);
    }
    
    protected override bool ReleaseHandle()
    {
        // Этот метод гарантированно вызовется CLR при finalization
        // даже если application crash, GC убирает SafeHandle первым
        return CloseHandle(handle) != 0;
    }
    
    [LibraryImport("kernel32.dll")]
    private static partial int CloseHandle(IntPtr handle);
}

// P/Invoke возвращает SafeHandle
[LibraryImport("kernel32.dll", StringMarshalling = StringMarshalling.Utf16, SetLastError = true)]
private static partial SafeFileHandle CreateFileW(string filename, ...);

// Использование
using var file = CreateFileW("test.txt", ...);
// Автоматический ReleaseHandle при Dispose или GC
```

### Существующие SafeHandle в .NET

- `Microsoft.Win32.SafeHandles.SafeFileHandle` — Windows file handle
- `Microsoft.Win32.SafeHandles.SafeWaitHandle` — wait events
- `System.Runtime.InteropServices.SafeHandle` — base
- `SafeHandleZeroOrMinusOneIsInvalid` — для handle где 0 или -1 = invalid
- `SafeHandleMinusOneIsInvalid` — где -1 = invalid

> [!info] CriticalFinalizerObject и финализация
> SafeHandle наследует от CriticalFinalizerObject — финализатор всегда выполняется при shutdown CLR (даже после AppDomain.Unload). Гарантирует cleanup даже в катастрофических ситуациях.

См. [[gc-memory|GC и память — SafeHandle]].

---

## 4. Calling Conventions

```csharp
[LibraryImport("native.dll")]
[UnmanagedCallConv(CallConvs = new[] { typeof(CallConvCdecl) })]
private static partial int CFunction(int x);

[LibraryImport("user32.dll")]
[UnmanagedCallConv(CallConvs = new[] { typeof(CallConvStdcall) })]
private static partial int WindowsApi(int x);
```

### Conventions

| Convention | Где используется |
|-----------|------------------|
| **Cdecl** | Стандарт C/C++ на Linux/macOS, ffmpeg, OpenCV |
| **Stdcall** | Windows API (kernel32.dll, user32.dll) |
| **Thiscall** | C++ instance methods (rare) |
| **Fastcall** | Регистровая передача параметров |
| **Vectorcall** | Microsoft, для SIMD |

В x64 — единая calling convention (Microsoft x64 на Windows, System V на Linux), различия только на x86. Для x64 кода обычно не нужно указывать explicitly.

---

## 5. Callbacks — native → managed

### Function pointer

```csharp
// C function принимает callback:
// typedef void (*Callback)(int value);
// void RegisterCallback(Callback cb);

// .NET 5+ — UnmanagedCallersOnly
[UnmanagedCallersOnly(CallConvs = new[] { typeof(CallConvCdecl) })]
private static void OnNative(int value)
{
    Console.WriteLine($"Called from native: {value}");
}

[LibraryImport("native.dll")]
private static unsafe partial void RegisterCallback(delegate* unmanaged[Cdecl]<int, void> callback);

unsafe
{
    RegisterCallback(&OnNative);
}
```

### Через Delegate (legacy)

```csharp
public delegate void NativeCallback(int value);

[DllImport("native.dll")]
private static extern void RegisterCallback(NativeCallback cb);

private static NativeCallback _callback = OnEvent;  // ⚠️ Сохрани reference!

private static void OnEvent(int value) { ... }

RegisterCallback(_callback);
```

> [!warning] GC может collect delegate
> Если ты передал делегат в native, но не сохранил reference — GC соберёт его → AccessViolationException когда native вызовет callback. Всегда храни reference.

---

## 6. Memory management в interop

### Native память — `Marshal.AllocHGlobal` / `NativeMemory`

```csharp
// Legacy
IntPtr ptr = Marshal.AllocHGlobal(1024);
try
{
    Marshal.Copy(managedArray, 0, ptr, managedArray.Length);
    NativeFunction(ptr);
}
finally
{
    Marshal.FreeHGlobal(ptr);
}

// .NET 6+ — NativeMemory (более явно)
unsafe
{
    void* ptr = NativeMemory.Alloc(1024);
    try
    {
        new Span<byte>(ptr, 1024).Fill(0);
        NativeFunction((IntPtr)ptr);
    }
    finally
    {
        NativeMemory.Free(ptr);
    }
}

// Aligned allocation (.NET 6+)
void* aligned = NativeMemory.AlignedAlloc(byteCount: 4096, alignment: 64);
NativeMemory.AlignedFree(aligned);
```

### Pinning — `fixed` blocks

```csharp
byte[] managed = new byte[1024];

unsafe
{
    fixed (byte* ptr = managed)
    {
        // managed array pinned — GC не будет двигать
        NativeFunction((IntPtr)ptr, managed.Length);
    }
    // unpinned после блока
}
```

### GCHandle

```csharp
// Долгоживущий pinning (когда fixed не подходит)
GCHandle handle = GCHandle.Alloc(managedArray, GCHandleType.Pinned);
try
{
    IntPtr ptr = handle.AddrOfPinnedObject();
    NativeFunction(ptr);
}
finally
{
    handle.Free();
}
```

См. [[gc-memory|GC — GCHandle типы]].

### POH (.NET 5+) — pinned object heap

```csharp
// Аллокация на POH — GC никогда не двигает (no overhead)
byte[] buffer = GC.AllocateArray<byte>(1024, pinned: true);

// Просто передавай как обычный массив, не нужен fixed
NativeFunction(buffer, buffer.Length);
```

Для долгоживущих native buffers — лучше POH чем GCHandle.Pinned (меньше fragmentation).

---

## 7. COM Interop

### COM — что это

Component Object Model — Microsoft technology для interop между разными языками через бинарные интерфейсы (vtable). Базовая для:
- Microsoft Office Automation
- Windows shell extensions
- ActiveX controls
- DirectX
- Microsoft MetaTrader 5

### Использование existing COM

```csharp
// Excel automation — old style
Type excelType = Type.GetTypeFromProgID("Excel.Application")!;
dynamic excel = Activator.CreateInstance(excelType)!;

excel.Visible = true;
var workbook = excel.Workbooks.Add();
var sheet = workbook.ActiveSheet;
sheet.Cells[1, 1] = "Hello from .NET";
workbook.SaveAs("output.xlsx");
excel.Quit();

// Marshal.ReleaseComObject — критично!
Marshal.ReleaseComObject(sheet);
Marshal.ReleaseComObject(workbook);
Marshal.ReleaseComObject(excel);
```

### Typed COM Interop

```csharp
// Если есть type library (.tlb) — генерируем typed wrapper
// tlbimp.exe Excel.tlb /out:Excel.Interop.dll

using Microsoft.Office.Interop.Excel;

Application excel = new Application();
Workbook workbook = excel.Workbooks.Add();
Worksheet sheet = (Worksheet)workbook.ActiveSheet;
sheet.Cells[1, 1] = "Typed!";
```

### `[GeneratedComInterface]` — modern (.NET 8+)

```csharp
// Source generator — без runtime reflection, AOT-friendly
[GeneratedComInterface]
[Guid("...")]
public partial interface IMyComObject
{
    int Method1(string arg);
    void Method2();
}
```

См. [Microsoft Docs — COM source generators](https://learn.microsoft.com/dotnet/standard/native-interop/comwrappers-source-generation).

### COM в нашем mind: MetaTrader 5

См. [[hft-low-latency|HFT/Low-Latency]] — паттерн STA-thread + COM для MT5 интеграции в .NET trading bot.

---

## 8. AOT-compatible interop

Native AOT не поддерживает runtime reflection и dynamic codegen → ограничения для interop:

| | Runtime | Native AOT |
|--|---------|-----------|
| `DllImport` | ✅ | ⚠️ limited |
| `LibraryImport` | ✅ | ✅ |
| Marshalling классов | ✅ | ❌ — только blittable |
| `Activator.CreateInstance(comType)` | ✅ | ❌ |
| `[GeneratedComInterface]` | ✅ | ✅ |
| `dynamic excel` | ✅ | ❌ |
| Custom marshallers (interface-based) | ✅ | ⚠️ Нужны generated |

**Для AOT:**
- Только `LibraryImport`
- Только blittable structs или явные marshallers
- COM через `GeneratedComInterface`
- Никакого `dynamic`

---

## 9. Producing Native Library через Native AOT

.NET 7+ позволяет компилировать .NET код **в native shared library** (.so / .dll / .dylib) для использования из C/Python/Rust.

```xml
<PropertyGroup>
  <OutputType>Library</OutputType>
  <PublishAot>true</PublishAot>
  <NativeLib>Shared</NativeLib>
</PropertyGroup>
```

```csharp
public class Calculator
{
    [UnmanagedCallersOnly(EntryPoint = "Add", CallConvs = new[] { typeof(CallConvCdecl) })]
    public static int Add(int a, int b) => a + b;
    
    [UnmanagedCallersOnly(EntryPoint = "GetVersion", CallConvs = new[] { typeof(CallConvCdecl) })]
    public static IntPtr GetVersion()
    {
        return Marshal.StringToHGlobalAnsi("1.0.0");
    }
}
```

```bash
dotnet publish -c Release -r linux-x64
# Output: bin/Release/net10.0/linux-x64/publish/MyLib.so

```

```c
// Использование из C
#include <stdio.h>
#include <dlfcn.h>

int main() {
    void* handle = dlopen("./MyLib.so", RTLD_LAZY);
    int (*Add)(int, int) = dlsym(handle, "Add");
    printf("%d\n", Add(2, 3));  // 5
    dlclose(handle);
}
```

Используется в реальных проектах:
- ML.NET модели как native library в Python
- Trading алгоритмы для использования из MT5
- Игровые движки (Unity native plugin)

---

## 10. Memory-mapped files и shared memory

```csharp
using System.IO.MemoryMappedFiles;

// Создание shared memory
using var mmf = MemoryMappedFile.CreateOrOpen("Global\\MyMmf", 1024 * 1024);
using var accessor = mmf.CreateViewAccessor();

// Запись
accessor.Write(0, 42);
accessor.Write(8, 3.14);

// Из другого процесса (или native код через CreateFileMapping)
using var existing = MemoryMappedFile.OpenExisting("Global\\MyMmf");
using var view = existing.CreateViewAccessor();
int value = view.ReadInt32(0);  // 42
```

См. [[ipc-named-pipes-grpc|IPC: Named Pipes & gRPC]] — MMF для high-perf IPC.

---

## 11. Common Pitfalls

### 1. Забыть `SetLastError = true`

```csharp
// ❌ Marshal.GetLastWin32Error() возвращает мусор
[LibraryImport("kernel32.dll")]
private static partial int CreateFile(...);

// ✅
[LibraryImport("kernel32.dll", SetLastError = true)]
private static partial int CreateFile(...);

if (result == -1)
{
    int err = Marshal.GetLastWin32Error();
    throw new Win32Exception(err);
}
```

### 2. String encoding mismatch

```csharp
// ❌ Default — ANSI на Windows, может ломать unicode
[DllImport("user32.dll")]
private static extern int MessageBox(IntPtr h, string text, string caption, uint type);

// ✅ Explicit
[LibraryImport("user32.dll", StringMarshalling = StringMarshalling.Utf16)]
private static partial int MessageBox(IntPtr h, string text, string caption, uint type);
```

### 3. Padding в struct

C struct без явного `#pragma pack` может иметь padding, отличный от .NET default:

```csharp
// C: struct { char c; int i; }  -- typically 8 bytes (3 padding)

// ❌ Pack=1 ломает binary compat
[StructLayout(LayoutKind.Sequential, Pack = 1)]
struct A { byte c; int i; }  // 5 bytes — wrong!

// ✅ Default pack
[StructLayout(LayoutKind.Sequential)]
struct A { byte c; int i; }  // 8 bytes
```

### 4. Pointer arithmetic без unsafe

```csharp
// ❌ Не работает в safe code
IntPtr ptr = Marshal.AllocHGlobal(100);
IntPtr next = ptr + 4;  // CS error

// ✅
IntPtr next = IntPtr.Add(ptr, 4);

// ✅✅ Unsafe
unsafe
{
    byte* p = (byte*)ptr;
    byte* next = p + 4;
}
```

### 5. Использование delegate без сохранения reference

```csharp
[LibraryImport("native.dll")]
private static partial void RegisterCallback(NativeCallback cb);

public void Register()
{
    NativeCallback cb = OnEvent;  // ❌ Local variable
    RegisterCallback(cb);
    // cb выходит из scope → GC может collect → native crash
}

// ✅ Сохрани в static / instance field
private static NativeCallback _callback = OnEvent;
RegisterCallback(_callback);
```

### 6. Char vs byte mismatch

```csharp
// C: char buffer[256]   — это byte[] в Linux/POSIX!
// На Windows это часто wchar_t (UTF-16)

// ✅ Зависит от platform — используй byte[] для bytes
[StructLayout(LayoutKind.Sequential)]
struct Data
{
    [MarshalAs(UnmanagedType.ByValArray, SizeConst = 256)]
    public byte[] Buffer;  // не char[]
}
```

### 7. Marshalling boolean — 1 vs 4 bytes

```csharp
// ❌ В struct — bool это 1 байт в C, 4 в .NET
struct Result { public bool Success; public int Code; }
// .NET marshalling: 4 + 4 = 8 bytes
// C struct: 1 + 3 padding + 4 = 8 bytes — совпадает на x64
// Но ARM может быть иначе

// ✅ Явное указание
struct Result
{
    [MarshalAs(UnmanagedType.U1)] public bool Success;  // 1 byte
    public int Code;
}
```

### 8. Releasing COM objects

```csharp
// ❌ COM объекты не собираются GC сразу — могут держать процесс Excel открытым
dynamic excel = ...;
excel.Quit();
// Процесс EXCEL.EXE всё ещё в task manager!

// ✅ Marshal.ReleaseComObject в finally
try { ... }
finally
{
    if (excel != null) Marshal.FinalReleaseComObject(excel);
    excel = null;
    GC.Collect();
    GC.WaitForPendingFinalizers();
    GC.Collect();
}
```

### 9. Pin страницы памяти при interop с DMA / GPU

При работе с GPU (CUDA), DMA — нужно pinned memory чтобы device не использовал swap:

```csharp
// Долгоживущий buffer для GPU
byte[] gpuBuffer = GC.AllocateArray<byte>(1024 * 1024, pinned: true);

// Или Marshal.AllocHGlobal — гарантированно вне GC heap
IntPtr nativeBuf = Marshal.AllocHGlobal(1024 * 1024);
```

### 10. P/Invoke в hot loop

P/Invoke transition stub — ~5-10 ns на x64. В loop с миллионом вызовов это 5-10 ms.

**Решения:**
- Batch operations: вместо 1M call(item) → 1 call(items)
- `[SuppressGCTransition]` (.NET 5+) — для очень коротких native функций (warning: native код не должен trigger GC)

```csharp
[LibraryImport("native.dll")]
[SuppressGCTransition]  // unsafe но быстро
private static partial int FastNativeFunc(int x);
```

---

## 12. Платформо-специфические особенности

### Windows
- `[DllImport("kernel32.dll")]` для Win32 API
- COM Interop — Office, Shell, ActiveX
- ProgID через `Type.GetTypeFromProgID`
- Registry через `Microsoft.Win32.Registry`
- Path separators — Windows specific (use `Path.Combine`)

### Linux
- `[DllImport("libc.so.6")]` для libc
- `dlopen`/`dlsym` для plugin loading
- POSIX functions — read, write, fork, exec
- File descriptors — int, не IntPtr (хотя через SafeHandle всё ок)

### macOS
- `[DllImport("libSystem.dylib")]` для system functions
- Cocoa via Objective-C — нужен ObjCRuntime (Xamarin/MAUI)

### Cross-platform
- Используй `OperatingSystem.IsWindows()`, `IsLinux()`, `IsMacOS()`
- Conditional `[LibraryImport]`:

```csharp
public static class CrossPlatform
{
    public static int GetCurrentProcessId() =>
        OperatingSystem.IsWindows() 
            ? GetCurrentProcessIdWin() 
            : GetCurrentProcessIdLinux();
    
    [LibraryImport("kernel32.dll", EntryPoint = "GetCurrentProcessId")]
    private static partial int GetCurrentProcessIdWin();
    
    [LibraryImport("libc", EntryPoint = "getpid")]
    private static partial int GetCurrentProcessIdLinux();
}
```

---

## 13. Best Practices

- **`LibraryImport` для нового кода** — AOT-compat, faster
- **`SafeHandle` для всех native handles** — гарантированный cleanup
- **Blittable structs** — `[StructLayout(Sequential)]`, явно `Pack` если нужно
- **`SetLastError = true`** для Win32 API
- **`StringMarshalling` явно** — UTF-8 для Linux, UTF-16 для Windows W functions
- **POH через `GC.AllocateArray<T>(pinned: true)`** для долгоживущих native buffers
- **Сохраняй reference на delegate** при passing в native
- **`Marshal.FinalReleaseComObject`** в finally для COM
- **Avoid `dynamic`** — only когда действительно нужно
- **Native exception ≠ C# exception** — обработай return code, не try/catch
- **Test на target platform** — Windows behavior ≠ Linux ≠ ARM
- **Profile с PerfView/dotTrace** — найти hot P/Invoke calls

---

## 14. Пример: использование OpenCV из .NET

```csharp
// Вариант 1 — готовый wrapper (preferred)
// NuGet: OpenCvSharp4
using OpenCvSharp;

var image = Cv2.ImRead("input.jpg");
var gray = new Mat();
Cv2.CvtColor(image, gray, ColorConversionCodes.BGR2GRAY);
Cv2.ImWrite("output.jpg", gray);

// Вариант 2 — прямой P/Invoke (если нет wrapper)
[LibraryImport("opencv_core.dll", StringMarshalling = StringMarshalling.Utf8)]
private static partial IntPtr cvLoadImage(string filename, int flags);

[LibraryImport("opencv_core.dll")]
private static partial void cvReleaseImage(ref IntPtr image);

IntPtr img = cvLoadImage("input.jpg", 1);
try { /* ... */ }
finally { cvReleaseImage(ref img); }
```

---

## 15. Пример: Trading bot с MetaTrader 5

```csharp
// COM Interop с MT5 (упрощённый)
using System.Runtime.InteropServices;

[ComImport]
[Guid("...")]
public interface IMtTerminal
{
    bool IsConnected();
    decimal GetBalance();
    int OrderSend(/* ... */);
}

// На STA thread (UI-thread-affinity required для COM)
[STAThread]
public static void Main()
{
    var type = Type.GetTypeFromProgID("MetaTrader.Application")!;
    dynamic mt = Activator.CreateInstance(type)!;
    
    if (mt.IsConnected())
    {
        Console.WriteLine($"Balance: {mt.GetBalance()}");
        // mt.OrderSend(...) — отправка ордера
    }
    
    Marshal.FinalReleaseComObject(mt);
}
```

См. [[hft-low-latency|HFT/Low-Latency]] — полный паттерн STA-thread + COM для MT5 trading bot.

---

## Cheat sheet

| Need | Approach |
|------|----------|
| Call C function | `[DllImport]` + extern method |
| Calling convention | `CallingConvention.Cdecl` (most common) или `Stdcall` |
| Pass struct | `[StructLayout(LayoutKind.Sequential)]` + blittable types |
| String to native | `[MarshalAs(UnmanagedType.LPStr)]` (UTF8) или `LPWStr` (UTF16) |
| Pinned array | `fixed (byte* p = arr) { ... }` |
| Long-lived pin | `GCHandle.Alloc(obj, GCHandleType.Pinned)` |
| Native pointer to managed | `Marshal.PtrToStructure<T>(ptr)` |
| Managed to native pointer | `Marshal.StructureToPtr<T>(obj, ptr, false)` |
| Free unmanaged | `Marshal.FreeHGlobal(ptr)` или `NativeMemory.Free(ptr)` |
| Callback as delegate | `[UnmanagedFunctionPointer]` attribute |
| Native AOT-compatible | `[LibraryImport]` (source generator) |
| Span ↔ native ptr | `MemoryMarshal.GetReference(span)` |
| String marshalling | UTF8 для cross-platform, UTF16 для Windows-only |
| Error handling | `Marshal.GetLastWin32Error()` (Windows) |

**Modern .NET 7+: используй `[LibraryImport]` вместо `[DllImport]`** — AOT-friendly, source-generated, faster.


---

## Decision tree

```
Native interop нужен?
│
├── Зачем?
│   ├── Call vendor SDK (only C/C++ available) → P/Invoke
│   ├── Performance hot path → SIMD intrinsics часто проще чем native
│   ├── OS-specific API → P/Invoke (но проверь .NET BCL)
│   └── Просто "хочу" → reconsider, скорее всего managed достаточно
│
├── .NET version?
│   ├── .NET 7+ → [LibraryImport] (source-gen, AOT)
│   └── < .NET 7 → [DllImport]
│
├── Что передаём?
│   ├── Primitives (int, float) → straightforward
│   ├── Strings → MarshalAs LPStr/LPWStr
│   ├── Structs → [StructLayout(Sequential)] + blittable
│   ├── Arrays → fixed для short-lived, GCHandle для long
│   └── Callbacks → [UnmanagedFunctionPointer] delegate
│
├── Cross-platform?
│   ├── Yes → DllImport "libname" (без extension)
│   └── Windows only → "kernel32.dll" / "user32.dll"
│
└── Альтернативы?
    ├── COM interop (Windows COM components)
    ├── ICustomMarshaler (complex marshaling)
    ├── C++/CLI (mixed-mode wrapper)
    └── Native library wrapper (хороший pattern)
```


---

## См. также

- [[compilation-jit|Compilation/JIT]] — как P/Invoke stubs работают
- [[gc-memory|GC и память]] — Pinning, SafeHandle, GCHandle
- [[span-layout|Span и Memory Layout]] — Span в interop, MemoryMarshal
- [[concurrency-atomics|Concurrency и Atomics]] — multithreading с native
- [[hft-low-latency|HFT/Low-Latency]] — MT5 COM, реальный case
- [[ipc-named-pipes-grpc|IPC: Named Pipes & gRPC]] — MMF для shared memory
- [[native-aot|Native AOT]] — limitations interop в AOT

## Reading list

- **Microsoft Docs — Native Interop** — learn.microsoft.com/dotnet/standard/native-interop
- **Microsoft Docs — LibraryImport source generator** — learn.microsoft.com/dotnet/standard/native-interop/pinvoke-source-generation
- **Microsoft Docs — COM Interop** — learn.microsoft.com/dotnet/standard/native-interop/cominterop
- **Adam Sitnik — P/Invoke performance** — adamsitnik.com
- **Stephen Toub — P/Invoke на devblogs** — devblogs.microsoft.com/dotnet
- **CLR via C#** (Jeffrey Richter) — главы Interop deep-dive
- **PInvoke.net** — pinvoke.net — community-maintained Win32 signatures
- **dnSpy / ILSpy** — для inspect generated marshalling stubs
