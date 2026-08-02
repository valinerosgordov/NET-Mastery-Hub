---
tags: [csharp, dispose, idisposable, middle, resource-management, gc, finalizer, using]
level: Middle
date: 2026-05-07
---

# IDisposable и Dispose Pattern — управление ресурсами

> **Детерминированное освобождение неуправляемых ресурсов в managed runtime.** `IDisposable`, `using`, `IAsyncDisposable`, finalizer pattern, `SafeHandle`, GC interaction. Закрывает пробел: «знаю про `using`, не понимаю когда писать finalizer, и почему `using var` лучше `using ()`».

---

## 0. Как читать этот файл

Если ты впервые работаешь с `IDisposable` — раздел 1→4: получишь рабочую модель и поймёшь ресурсы. Если уже пишешь Dispose, но непонятно про finalizer — раздел 6 (Dispose pattern), 7 (SafeHandle). Если строишь production систему — раздел 9 (async dispose), 11 (best practices), 13 (pitfalls).

Cross-language якоря (`> [!info]-`) свёрнуты — раскрывай если переходишь из Python / Java / Rust / Go / C++. Interview-вопросы (`> [!question]-`) встроены.

---

## 1. Что это, зачем и когда

### 1.1. Managed vs unmanaged ресурсы

В .NET runtime есть **garbage collector** (GC), который освобождает managed memory автоматически. Но есть ресурсы, **которые GC не знает**:

- File handles (OS дескрипторы файлов)
- Network sockets, HTTP connections
- Database connections
- Mutex / Semaphore handles
- GDI handles (Windows graphics)
- Memory-mapped files
- Native memory (через `Marshal.AllocHGlobal`)

GC обнаружит объект-обёртку (managed) когда нет references, но handle внутри **не освободит автоматически**. Ресурс leak'ает до перезапуска процесса.

### 1.2. Зачем IDisposable

`IDisposable` — стандартный contract для типов, **владеющих unmanaged ресурсом** или composing других IDisposable:

```csharp
public interface IDisposable
{
    void Dispose();
}
```

Caller вызывает `Dispose()` — instance освобождает ресурсы. Это **deterministic** cleanup — точка освобождения предсказуема, в отличие от GC (когда захочет).

### 1.3. Главное правило

```
Если класс владеет:
  - Unmanaged resource (handle, native memory) → IMPLEMENT IDisposable + finalizer
  - Только managed IDisposable fields → IMPLEMENT IDisposable (без finalizer)
  - Только managed memory (без unmanaged) → НЕ implement IDisposable

Если consume IDisposable объект:
  - Always wrap в using / using var / try/finally
  - Не вызывай Dispose вручную — using дешевле и безопаснее
```

### 1.4. Эволюция

| Версия | Что |
|--------|-----|
| **.NET 1.0** | `IDisposable`, `using` statement |
| **.NET 2.0** | `SafeHandle` — recommended replacement для finalizers |
| **C# 8.0** | `IAsyncDisposable`, `await using` |
| **C# 8.0** | `using var` — implicit scope (без braces) |
| **.NET 6+** | Performance improvements в Dispose paths |
| **.NET 8+** | `ConfigureAwait` improvements для `await using` |

> [!info]- Если ты знаешь Python / Java / Rust / Go / C++
> **Python:** context managers (`with open(f) as f:`) ↔ `using`. Protocol: `__enter__`/`__exit__`. Для async: `async with`.
>
> **Java:** `AutoCloseable` ↔ `IDisposable`, try-with-resources ↔ `using`. С Java 7+. Finalizers deprecated, recommended cleaner API.
>
> **Rust:** RAII через `Drop` trait — automatic cleanup при scope exit. Безопаснее C# (compile-time guaranteed), нет `using` keyword нужно.
>
> **Go:** `defer` keyword для cleanup at function exit. Не такой strict как RAII, но работает.
>
> **C++:** RAII через destructor — automatic. C# выбрал managed runtime + explicit Dispose, потому что GC не deterministic.

> [!question]- Интервью: зачем нужен `IDisposable` если есть GC?
> GC освобождает managed memory автоматически когда нет references. Но GC **не знает** про unmanaged ресурсы (file handles, sockets, native memory) — они leak'ают до перезапуска процесса. `IDisposable` — contract для **deterministic cleanup**: caller вызывает `Dispose()`, ресурс освобождается **сразу**, не ждёт GC. Также важно для shared resources (DB connections, locks) — нельзя ждать unspecified GC time. `using` statement — syntactic sugar для try/finally + Dispose.

---

## 2. using — основной инструмент

### 2.1. using statement (старый стиль)

```csharp
using (var stream = new FileStream("file.txt", FileMode.Open))
{
    // работа со stream
}
// stream.Dispose() вызван автоматически в finally
```

Компилятор разворачивает в:

```csharp
{
    var stream = new FileStream("file.txt", FileMode.Open);
    try
    {
        // работа со stream
    }
    finally
    {
        if (stream != null) stream.Dispose();
    }
}
```

`Dispose` вызывается **всегда** — даже при exception.

### 2.2. using var (C# 8+)

```csharp
public void ReadFile()
{
    using var stream = new FileStream("file.txt", FileMode.Open);
    using var reader = new StreamReader(stream);
    
    var content = reader.ReadToEnd();
    // ... другая работа
}
// reader.Dispose() и stream.Dispose() — в обратном порядке (LIFO) при выходе из метода
```

`using var` — без braces, scope = до конца enclosing block. **Стандарт в новом коде**.

### 2.3. Multiple resources

```csharp
// Старый стиль — стек braces
using (var conn = new SqlConnection(connStr))
using (var cmd = new SqlCommand(sql, conn))
using (var reader = cmd.ExecuteReader())
{
    while (reader.Read()) { /* ... */ }
}

// Новый стиль — flat
using var conn = new SqlConnection(connStr);
using var cmd = new SqlCommand(sql, conn);
using var reader = cmd.ExecuteReader();

while (reader.Read()) { /* ... */ }
// Dispose в LIFO: reader → cmd → conn
```

### 2.4. await using (C# 8+, .NET Core 3+)

```csharp
await using var stream = new FileStream("file.txt", FileMode.Open);
await using var reader = new StreamReader(stream);

var content = await reader.ReadToEndAsync();
// stream.DisposeAsync() ← await
// reader.DisposeAsync() ← await
```

`await using` для `IAsyncDisposable` — async cleanup. Подробнее раздел 9.

### 2.5. ConfigureAwait в await using

```csharp
// Library code
await using var stream = new FileStream(...).ConfigureAwait(false);
```

Wait — `await using x = expr.ConfigureAwait(false)` не работает напрямую. Используй вместо этого:

```csharp
var stream = new FileStream(...);
await using (stream.ConfigureAwait(false))
{
    // ...
}
```

Или явно `IAsyncDisposable` cast.

### 2.6. using не для всех types

```csharp
// using требует IDisposable
using var x = new MyClass();   // ❌ Compile error если MyClass нет IDisposable

// IDE подскажет — implement IDisposable или убери using
```

Compiler проверяет: тип должен реализовать `IDisposable` (или duck-typed `Dispose()` для refs structs в .NET 5+).

### 2.7. using ref struct (C# 8+)

```csharp
public ref struct MyRefStruct
{
    public void Dispose() { /* ... */ }
}

// Compile-time check — ref struct не может реализовать interface,
// но если есть Dispose() метод — using работает
using var x = new MyRefStruct();
```

Для `Span<T>` и similar — pattern-based using без IDisposable.

> [!question]- Интервью: чем `using var` отличается от `using ()`?
> `using ()` (statement) — explicit scope с braces, Dispose вызывается на выходе из braces. `using var` (declaration, C# 8+) — без braces, scope = до конца enclosing method/block. `using var` короче, читаемее, особенно при multiple resources. Compiler генерирует тот же try/finally + Dispose. Best practice: `using var` в новом коде. `using ()` оправдан только когда нужен explicit scope **меньше** enclosing метода (например, Dispose **внутри** loop iteration).

---

## 3. Простой IDisposable — managed resources only

### 3.1. Базовая реализация

Если класс **не владеет unmanaged ресурсом** напрямую, а только composing других IDisposable — простая реализация:

```csharp
public class FileProcessor : IDisposable
{
    private readonly StreamReader _reader;
    private readonly StreamWriter _writer;
    private bool _disposed;
    
    public FileProcessor(string inputPath, string outputPath)
    {
        _reader = new StreamReader(inputPath);
        _writer = new StreamWriter(outputPath);
    }
    
    public void Dispose()
    {
        if (_disposed) return;
        
        _reader.Dispose();
        _writer.Dispose();
        
        _disposed = true;
    }
}
```

### 3.2. Idempotent Dispose

`Dispose()` должен быть **idempotent** — безопасным при multiple calls:

```csharp
var p = new FileProcessor(...);
p.Dispose();
p.Dispose();   // не должен throw / corrupt
```

Стандартный приём — `_disposed` flag:

```csharp
public void Dispose()
{
    if (_disposed) return;   // skip если уже disposed
    
    // ... cleanup
    
    _disposed = true;
}
```

### 3.3. ObjectDisposedException

Каждый public method должен проверять `_disposed`:

```csharp
public void Process()
{
    ObjectDisposedException.ThrowIf(_disposed, this);   // .NET 7+
    // или старый стиль:
    // if (_disposed) throw new ObjectDisposedException(nameof(FileProcessor));
    
    // ... работа
}
```

Caller получает clear exception вместо непонятного `NullReferenceException` или corrupted state.

### 3.4. Без finalizer для simple case

Простой `IDisposable` (composing managed) **не нуждается в finalizer**. Finalizer — только для unmanaged resources, которые `Dispose` забыли вызвать.

```csharp
public class SimpleWrapper : IDisposable
{
    private readonly IDisposable _inner;
    
    public SimpleWrapper(IDisposable inner) { _inner = inner; }
    
    public void Dispose()   // достаточно — без finalizer, без GC.SuppressFinalize
    {
        _inner.Dispose();
    }
}
```

Finalizer добавляет overhead (объект попадает в finalization queue, живёт лишний GC cycle).

### 3.5. Naming convention

Public method — `Dispose()` (per IDisposable contract). Если есть extra cleanup logic — protected `virtual Dispose(bool disposing)` (раздел 6).

### 3.6. Disposing сторонней responsibility

```csharp
public class Decorator : IDisposable
{
    private readonly Stream _inner;
    private readonly bool _ownsInner;
    
    public Decorator(Stream inner, bool ownsInner = true)
    {
        _inner = inner;
        _ownsInner = ownsInner;
    }
    
    public void Dispose()
    {
        if (_ownsInner) _inner.Dispose();
    }
}
```

Pattern из BCL — `StreamReader` / `StreamWriter` имеют `leaveOpen` параметр. Не закрывают base stream если caller передал ownership = false.

> [!question]- Интервью: что значит "idempotent Dispose"?
> Method можно вызвать **многократно** без побочных эффектов после первого вызова. `Dispose` обязан быть idempotent: после первого call ресурс освобождён, повторные calls — no-op. Реализация через `_disposed` flag: первый call cleanup + sets flag, последующие skip. Это важно потому что `using` может вызвать Dispose, потом caller тоже вручную, или finalizer добежит. Несколько освобождений того же handle = double-free → undefined behaviour. Idempotent Dispose защищает.

---

## 4. Composition — disposing children

### 4.1. Owner disposes children

Если класс **создаёт** disposable fields — owner должен их dispose:

```csharp
public class HttpService : IDisposable
{
    private readonly HttpClient _client;
    
    public HttpService()
    {
        _client = new HttpClient();   // владеем — должны dispose
    }
    
    public void Dispose() => _client.Dispose();
}
```

### 4.2. Injected dependencies — НЕ disposing

Если dependency **передан в constructor** — обычно **не** disposить. Caller / DI container управляет lifetime:

```csharp
public class HttpService
{
    private readonly HttpClient _client;
    
    public HttpService(HttpClient client)
    {
        _client = client;   // НЕ владеем — не dispose
    }
    
    // Не имплементируем IDisposable если только composing
}
```

### 4.3. DI lifetimes и Dispose

ASP.NET Core DI container:
- **Transient** — service разрешается каждый раз. DI tracks scope, dispose в конце scope.
- **Scoped** — один instance на HTTP request. Dispose в конце request.
- **Singleton** — один instance на app. Dispose при shutdown.

```csharp
// DI container сам вызовет Dispose
services.AddScoped<IUserService, UserService>();   // если UserService : IDisposable
```

Не вызывай Dispose сам — container делает.

### 4.4. Disposing в правильном порядке

```csharp
public class Manager : IDisposable
{
    private readonly Stream _output;
    private readonly StreamWriter _writer;
    
    public Manager()
    {
        _output = File.Create("out.txt");
        _writer = new StreamWriter(_output);
    }
    
    public void Dispose()
    {
        _writer?.Dispose();   // first — flushes buffer
        _output?.Dispose();   // then — closes file handle
    }
}
```

LIFO order — обычно. `using var` автоматически делает LIFO. Manual cleanup тоже должен.

### 4.5. Try/finally в Dispose

```csharp
public void Dispose()
{
    if (_disposed) return;
    
    try { _resource1?.Dispose(); }
    catch { /* swallow или log */ }
    
    try { _resource2?.Dispose(); }
    catch { }
    
    _disposed = true;
}
```

Один Dispose не должен влиять на другие. Часто swallow exceptions в Dispose — иначе теряется ошибка предыдущей операции (которая, скорее всего, была более important).

> [!question]- Интервью: должен ли DI container dispose-ить services?
> Да — Microsoft.Extensions.DependencyInjection (стандартный ASP.NET Core DI) **автоматически вызывает `Dispose`** на services, реализующих `IDisposable` / `IAsyncDisposable`, при выходе из scope. Scoped — в конце HTTP request. Singleton — при shutdown app. Transient — tracked в parent scope (важно: не abuse transient disposable, может расти memory). Не вызывай Dispose сам если services из DI — container делает. Manual Dispose только для explicitly created `new` объектов.

---

## 5. try/finally вместо using — когда

### 5.1. Conditional Dispose

```csharp
// Когда не всегда нужно disposить
Stream? stream = null;
try
{
    if (someCondition) stream = OpenStream();
    // ... работа
}
finally
{
    stream?.Dispose();
}
```

### 5.2. Dispose в callback

```csharp
public void Process(Action<Stream> callback)
{
    var stream = OpenStream();
    try
    {
        callback(stream);   // callback может throw
    }
    finally
    {
        stream.Dispose();
    }
}
```

### 5.3. Manual Dispose в long-running

```csharp
public async Task ProcessManyFiles(IEnumerable<string> paths)
{
    foreach (var path in paths)
    {
        var stream = File.OpenRead(path);
        try
        {
            await ProcessAsync(stream);
        }
        finally
        {
            await stream.DisposeAsync();
        }
        // file закрыт здесь, до next iteration
    }
}

// Альтернатива через using:
await foreach (var path in paths)
{
    await using var stream = File.OpenRead(path);   // dispose at iteration end
    await ProcessAsync(stream);
}
```

`using var` внутри loop — disposes на каждой iteration.

### 5.4. Из библиотек — пример StreamReader

```csharp
public class StreamReader : IDisposable
{
    private Stream _stream;
    private bool _leaveOpen;
    
    protected override void Dispose(bool disposing)
    {
        if (disposing && !_leaveOpen)
            _stream?.Dispose();
        base.Dispose(disposing);
    }
}
```

`leaveOpen` — каллер сохраняет ownership. Полезно для adapters / decorators.

> [!question]- Интервью: когда использовать try/finally вместо using?
> 1) **Conditional dispose** — disposable создаётся не всегда (`if`). 2) **Передача в callback** — нужно ensure cleanup при exception в callback. 3) **Custom logic в finally** — logging, error tracking. 4) **Pattern-based использование** для типов с `Dispose()` методом, которые не реализуют `IDisposable`. В большинстве случаев `using var` лучше — короче, ошибиться сложнее. try/finally — explicit control flow, exception handling в Dispose path.

---

## 6. Full Dispose pattern — для unmanaged ресурсов

### 6.1. Когда нужен полный pattern

Полный pattern (с finalizer) — только когда класс **владеет unmanaged resource напрямую**:

```csharp
public class NativeBufferOwner : IDisposable
{
    private IntPtr _buffer;
    private bool _disposed;
    
    public NativeBufferOwner(int size)
    {
        _buffer = Marshal.AllocHGlobal(size);   // unmanaged!
    }
    
    // Public Dispose — for callers
    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
    
    // Protected virtual — для derived classes
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        
        if (disposing)
        {
            // Free managed resources here (если есть)
        }
        
        // Free unmanaged
        if (_buffer != IntPtr.Zero)
        {
            Marshal.FreeHGlobal(_buffer);
            _buffer = IntPtr.Zero;
        }
        
        _disposed = true;
    }
    
    // Finalizer — последний шанс если Dispose forgotten
    ~NativeBufferOwner()
    {
        Dispose(disposing: false);
    }
}
```

### 6.2. disposing=true vs false

- **`disposing=true`** — caller вызвал `Dispose()`. Можно safely освобождать **managed** resources (other IDisposable fields).
- **`disposing=false`** — finalizer вызвал. **НЕ трогай managed resources** — они могут быть уже finalized (порядок finalizer'ов недетерминирован).

```csharp
protected virtual void Dispose(bool disposing)
{
    if (_disposed) return;
    
    if (disposing)
    {
        _managedField?.Dispose();   // OK only here
    }
    
    // Unmanaged cleanup — always
    if (_handle != IntPtr.Zero)
    {
        NativeFree(_handle);
        _handle = IntPtr.Zero;
    }
    
    _disposed = true;
}
```

### 6.3. GC.SuppressFinalize

```csharp
public void Dispose()
{
    Dispose(true);
    GC.SuppressFinalize(this);   // ← убрать из finalization queue
}
```

Если caller вызвал `Dispose()`, ресурс уже free — finalizer не нужен. `SuppressFinalize` снимает объект с finalization queue, экономя один GC cycle.

### 6.4. Finalizer cost

```
Без finalizer:
  - Allocation, use, GC collect → объект gone

С finalizer:
  - Allocation → объект попадает в finalization queue
  - GC collect → объект НЕ collected, добавлен в f-reachable queue
  - Finalizer thread вызывает finalizer (асинхронно)
  - Next GC collect → объект collected
  → 2x работа, lives longer
```

Finalizer = expensive. Используй **только** когда необходимо.

### 6.5. Sealed класс — упрощение

```csharp
public sealed class NativeOwner : IDisposable
{
    private IntPtr _handle;
    
    public NativeOwner(int size) => _handle = Marshal.AllocHGlobal(size);
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    private void Dispose(bool disposing)
    {
        // ... cleanup
    }
    
    ~NativeOwner() => Dispose(false);
}
```

Sealed — нет derived → `private` вместо `protected virtual` для `Dispose(bool)`. Чуть проще.

### 6.6. Inheritable case — virtual

```csharp
public class Base : IDisposable
{
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing) { /* ... */ }
    
    ~Base() => Dispose(false);
}

public class Derived : Base
{
    private IDisposable? _myResource;
    
    protected override void Dispose(bool disposing)
    {
        if (disposing)
        {
            _myResource?.Dispose();
        }
        base.Dispose(disposing);   // ВАЖНО — chain
    }
}
```

Derived overrides `Dispose(bool)`, **не** `Dispose()` — base уже handles. Chain через `base.Dispose(disposing)`.

> [!question]- Интервью: что делает `GC.SuppressFinalize`?
> Удаляет объект из finalization queue — finalizer не будет вызван. Используется в `Dispose()` после освобождения ресурсов: caller сам делает cleanup, finalizer не нужен. Без SuppressFinalize объект попадает в finalization queue → переживает один GC cycle, finalizer thread вызывает finalizer асинхронно → next GC collect. С SuppressFinalize — collected на первом cycle. Performance: один GC cycle меньше + не tracked. Вызывать **всегда** в Dispose() для классов с finalizer.

---

## 7. SafeHandle — рекомендованная альтернатива

### 7.1. Зачем SafeHandle

`SafeHandle` (.NET 2.0+) — abstract class из BCL, замена ручному finalizer pattern для handle resources:

```csharp
public abstract class SafeHandle : CriticalFinalizerObject, IDisposable
{
    protected IntPtr handle;
    public abstract bool IsInvalid { get; }
    protected abstract bool ReleaseHandle();
    // ...
}
```

Преимущества:
- **Built-in finalizer** — не нужно писать свой.
- **Critical finalization** — ResolveHandle вызывается даже при abort, через `CriticalFinalizerObject`.
- **Reference counting** — protect от premature finalization при concurrent use.
- **Recommended Microsoft** для всех handle resources.

### 7.2. Готовые SafeHandle subclasses

```csharp
// Microsoft.Win32.SafeHandles namespace
SafeFileHandle             // file handle
SafeWaitHandle              // wait handle (mutex, semaphore)
SafeProcessHandle           // process handle
SafeRegistryHandle          // registry key
SafeAccessTokenHandle       // Windows access token
```

Используй готовые когда возможно.

### 7.3. Custom SafeHandle

```csharp
public sealed class MySafeHandle : SafeHandleZeroOrMinusOneIsInvalid
{
    public MySafeHandle() : base(ownsHandle: true) { }
    
    public MySafeHandle(IntPtr handle) : base(ownsHandle: true)
    {
        SetHandle(handle);
    }
    
    protected override bool ReleaseHandle()
    {
        return NativeMethods.CloseHandle(handle);
    }
}

// Использование
public class Resource : IDisposable
{
    private readonly MySafeHandle _handle;
    
    public Resource()
    {
        _handle = NativeMethods.OpenResource();
    }
    
    public void Dispose() => _handle.Dispose();   // автоматически делает all
}
```

Класс, использующий `SafeHandle`, **не** нуждается в собственном finalizer — SafeHandle сам управляет.

### 7.4. SafeHandle vs raw IntPtr

```csharp
// ❌ Raw IntPtr — нужен ручной pattern
public class BadResource : IDisposable
{
    private IntPtr _handle;
    private bool _disposed;
    
    public void Dispose() { /* boilerplate */ }
    ~BadResource() { /* boilerplate */ }
}

// ✅ SafeHandle — намного меньше кода
public class GoodResource : IDisposable
{
    private readonly SafeFileHandle _handle;
    public void Dispose() => _handle.Dispose();
}
```

> [!question]- Интервью: почему SafeHandle лучше custom finalizer?
> 1) **Built-in finalizer** — не пишешь boilerplate. 2) **CriticalFinalizerObject** — finalizer вызывается даже при abort/unhandled exception (rare paths где normal finalizer skipped). 3) **Reference counting** — защита от race conditions при concurrent Dispose / use. 4) **Recommended Microsoft** для handle resources — стандарт. Используй SafeFileHandle, SafeWaitHandle и др. готовые subclasses. Custom SafeHandle через `SafeHandleZeroOrMinusOneIsInvalid` или `SafeHandleMinusOneIsInvalid`. Только raw IntPtr + custom finalizer — для legacy / educational только.

---

## 8. ObjectDisposedException и `_disposed` flag

### 8.1. Стандартный pattern

```csharp
public class Service : IDisposable
{
    private bool _disposed;
    
    public void DoWork()
    {
        ObjectDisposedException.ThrowIf(_disposed, this);   // .NET 7+
        // ... work
    }
    
    public void Dispose()
    {
        if (_disposed) return;
        _disposed = true;
        // ... cleanup
    }
}
```

`.NET 7+` имеет `ObjectDisposedException.ThrowIf(condition, instance)` — convenience method.

### 8.2. Старый стиль

```csharp
public void DoWork()
{
    if (_disposed) throw new ObjectDisposedException(nameof(Service));
}
```

### 8.3. ThrowIfDisposed extension (если нет .NET 7)

```csharp
public static class DisposableExtensions
{
    public static void ThrowIfDisposed(this object obj, bool disposed)
    {
        if (disposed) throw new ObjectDisposedException(obj.GetType().FullName);
    }
}
```

### 8.4. Когда проверять

```csharp
public void PublicMethod() { ThrowIfDisposed(); /* work */ }   // ✅ public — check
public Task<T> PublicAsync() { ThrowIfDisposed(); /* work */ }  // ✅
internal void InternalMethod() { /* skip check — internal trust */ }   // optional
private void Helper() { /* no check — trusted */ }
```

Public methods — обязательно. Internal/private — usually trust callers (your own code).

### 8.5. Ловушка — Dispose не должен throw

```csharp
public void Dispose()
{
    ObjectDisposedException.ThrowIf(_disposed, this);   // ❌ Bad — Dispose должен быть idempotent
    // ...
}

// ✅ Правильно
public void Dispose()
{
    if (_disposed) return;   // skip, не throw
    // ...
    _disposed = true;
}
```

`Dispose` должен быть idempotent — silent skip при second call. Throwing breaks `using` semantics (try/finally вызовет Dispose даже при exception в body — не usable если Dispose throws).

> [!question]- Интервью: должен ли `Dispose` throw `ObjectDisposedException`?
> Нет — `Dispose` должен быть **idempotent**, повторный вызов — no-op. Throw нарушает `using` semantics: compiler генерирует try/finally + Dispose, при exception в body finally вызывает Dispose — если Dispose throws, original exception теряется. **Public methods other than Dispose** — да, throw `ObjectDisposedException` через `_disposed` check. Это сигнализирует caller'у — instance мёртв, не используй.

---

## 9. IAsyncDisposable и await using

### 9.1. Зачем async dispose

Иногда cleanup сам async — flush сетевого буфера, close DB connection с rollback, finalize HTTP stream:

```csharp
public interface IAsyncDisposable
{
    ValueTask DisposeAsync();
}
```

C# 8+ / .NET Core 3+. `ValueTask` — для случаев, когда synchronous завершение возможно (без allocation).

### 9.2. await using

```csharp
public async Task ProcessAsync()
{
    await using var stream = new FileStream("file.txt", FileMode.Open);
    await using var reader = new StreamReader(stream);
    
    var content = await reader.ReadToEndAsync();
    // await stream.DisposeAsync() в finally
}
```

`await using` — sugar для async finally + `await DisposeAsync()`.

### 9.3. Реализация IAsyncDisposable

```csharp
public class AsyncResource : IAsyncDisposable, IDisposable
{
    private NetworkStream? _stream;
    private bool _disposed;
    
    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        
        if (_stream != null)
        {
            await _stream.FlushAsync();
            await _stream.DisposeAsync();
            _stream = null;
        }
        
        _disposed = true;
        GC.SuppressFinalize(this);   // если был finalizer
    }
    
    // Sync version для backward compat
    public void Dispose()
    {
        if (_disposed) return;
        _stream?.Dispose();
        _disposed = true;
        GC.SuppressFinalize(this);
    }
}
```

Best practice: implement **both** для compat. Sync version — fallback.

### 9.4. ConfigureAwait в await using

```csharp
public async Task LibraryMethod()
{
    var resource = CreateResource();
    var awaitable = resource.DisposeAsync().ConfigureAwait(false);
    
    try
    {
        await DoWorkAsync().ConfigureAwait(false);
    }
    finally
    {
        await awaitable;
    }
}
```

Это verbose. В library code часто проще — sync `using` если возможно.

### 9.5. Microsoft.Extensions.DependencyInjection и async dispose

```csharp
// DI container поддерживает async dispose
public class MyService : IAsyncDisposable
{
    public async ValueTask DisposeAsync() { /* ... */ }
}

services.AddScoped<MyService>();

// При выходе из scope — DisposeAsync вызывается
await using var scope = serviceProvider.CreateAsyncScope();
var service = scope.ServiceProvider.GetRequiredService<MyService>();
// ... use
// scope.DisposeAsync() автоматически calls services.DisposeAsync()
```

### 9.6. Когда IAsyncDisposable

✅ **Используй когда:**
- Cleanup действительно async (flush/finalize требует await).
- Стандарт Microsoft для new APIs.

❌ **Не используй когда:**
- Cleanup полностью sync — обычный `IDisposable` достаточно.
- Возникает соблазн добавить async везде "на всякий случай".

> [!question]- Интервью: чем `IAsyncDisposable` отличается от `IDisposable`?
> `IAsyncDisposable.DisposeAsync()` возвращает `ValueTask` — позволяет await flush/cleanup operations. `IDisposable.Dispose()` — sync. C# 8+ ввёл `await using` для async cleanup. Best practice — реализуй **оба** интерфейса в типе с async cleanup: async для full path, sync — для legacy callers + finalizer fallback. `ValueTask` (vs Task) — может завершиться synchronously без allocation.

---

## 10. Dispose в наследовании

### 10.1. Base реализует IDisposable

```csharp
public class Base : IDisposable
{
    private bool _disposed;
    private IntPtr _handle;
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        
        if (disposing)
        {
            // managed cleanup
        }
        
        // unmanaged
        if (_handle != IntPtr.Zero)
        {
            NativeFree(_handle);
            _handle = IntPtr.Zero;
        }
        
        _disposed = true;
    }
    
    ~Base() => Dispose(false);
}
```

### 10.2. Derived overrides Dispose(bool)

```csharp
public class Derived : Base
{
    private IDisposable? _myField;
    
    protected override void Dispose(bool disposing)
    {
        if (disposing)
        {
            _myField?.Dispose();
        }
        
        base.Dispose(disposing);   // ОБЯЗАТЕЛЬНО chain
    }
    
    // Не перезаписывай public Dispose() и не пиши свой finalizer —
    // base уже handles
}
```

### 10.3. Неправильно — override public Dispose

```csharp
public class BadDerived : Base
{
    public new void Dispose() { /* ... */ }   // ❌ hides parent — bug
}
```

`Dispose()` в base **не virtual** by design. Derived overrides только `Dispose(bool)`.

### 10.4. Skip finalizer in derived (sealed)

```csharp
public sealed class FinalDerived : Base
{
    protected override void Dispose(bool disposing)
    {
        // ... extra cleanup
        base.Dispose(disposing);
    }
    // No new finalizer — base уже имеет
}
```

> [!question]- Интервью: что override-ить в derived классе с Dispose pattern?
> **Только `Dispose(bool disposing)`** — protected virtual метод от base. Public `Dispose()` уже handles GC.SuppressFinalize и chains в Dispose(bool). Finalizer тоже не overridden — base finalizer вызовет Dispose(false), которое вызовет твой override через virtual dispatch. **Обязательно chain `base.Dispose(disposing)`** в конце своего Dispose(bool) — иначе base cleanup пропустится.

---

## 11. Best Practices

### 11.1. Implementing IDisposable

- ✅ **`using var`** в новом коде.
- ✅ **Idempotent** Dispose через `_disposed` flag.
- ✅ **`ObjectDisposedException.ThrowIf`** в public methods.
- ✅ **`SafeHandle`** для unmanaged handles вместо custom finalizer.
- ✅ **`IAsyncDisposable`** для async cleanup (+ sync `IDisposable` fallback).
- ✅ **`GC.SuppressFinalize(this)`** в Dispose() если есть finalizer.
- ❌ **Finalizer без unmanaged resource** — overhead без пользы.
- ❌ **Throw в Dispose** — ломает `using` semantics.
- ❌ **Dispose() override в derived** — overrides Dispose(bool) instead.

### 11.2. Consuming IDisposable

- ✅ **`using var`** для всех disposable.
- ✅ **DI container** управляет lifetime — не вызывай Dispose сам.
- ✅ **`leaveOpen` параметр** для adapters/decorators.
- ❌ **Manual Dispose забывание** — leak.
- ❌ **Dispose в loop без using** — easy to forget при exception.

### 11.3. Async dispose

- ✅ **`await using`** для IAsyncDisposable.
- ✅ **Implement both** IDisposable + IAsyncDisposable для compat.
- ✅ **`ValueTask`** возврат — sync completion без allocation.
- ❌ **Sync Dispose с blocking wait** на async operation — deadlock risk.

---

## 12. Decision tree

```
Класс владеет ресурсом?
│
├── Только managed memory (без unmanaged) → НЕТ IDisposable
│
├── Composing managed IDisposable fields →
│   IDisposable (simple) — нет finalizer, нет SuppressFinalize
│
├── Unmanaged handle / native memory →
│   Используй SafeHandle (recommended)
│   ИЛИ полный pattern: IDisposable + Dispose(bool) + finalizer + SuppressFinalize
│
├── Async cleanup нужен (flush, network) →
│   IAsyncDisposable + await using
│   (+ IDisposable fallback)
│
└── Inherit IDisposable parent →
    Override только Dispose(bool)
    Chain через base.Dispose(disposing)

Caller использует?
│
├── using var (C# 8+) — default choice
├── using ()  — для explicit scope меньше метода
├── try/finally — conditional / callback
└── DI container — auto-Dispose в scope
```

---

## 13. Cheat sheet

```csharp
// === Simple IDisposable (managed only) ===
public class Simple : IDisposable
{
    private readonly IDisposable _inner;
    private bool _disposed;
    
    public Simple(IDisposable inner) => _inner = inner;
    
    public void Dispose()
    {
        if (_disposed) return;
        _inner.Dispose();
        _disposed = true;
    }
}

// === Full pattern (with unmanaged) ===
public class Full : IDisposable
{
    private IntPtr _handle;
    private bool _disposed;
    
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }
    
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;
        
        if (disposing) { /* managed */ }
        
        if (_handle != IntPtr.Zero)
        {
            NativeFree(_handle);
            _handle = IntPtr.Zero;
        }
        
        _disposed = true;
    }
    
    ~Full() => Dispose(false);
}

// === Better — SafeHandle ===
public class WithSafeHandle : IDisposable
{
    private readonly SafeFileHandle _handle;
    public void Dispose() => _handle.Dispose();
}

// === IAsyncDisposable ===
public class Async : IAsyncDisposable, IDisposable
{
    public async ValueTask DisposeAsync()
    {
        await CleanupAsync();
        Dispose();
    }
    
    public void Dispose() { /* sync cleanup */ }
}

// === Consuming ===
using var resource = CreateResource();
await using var asyncResource = CreateAsync();

// Multiple
using var a = CreateA();
using var b = CreateB();
using var c = CreateC();
// LIFO dispose: c → b → a

// === Public method check ===
public void DoWork()
{
    ObjectDisposedException.ThrowIf(_disposed, this);   // .NET 7+
    // work
}
```

| Сценарий | Решение |
|----------|---------|
| Owned managed disposable | Simple IDisposable |
| Unmanaged handle | SafeHandle (recommended) |
| Native memory ручное | Full pattern + finalizer |
| Async cleanup | IAsyncDisposable |
| Inheritance | Override Dispose(bool) + chain base |
| Consume | `using var` |
| DI service | DI container auto-disposes |
| Conditional cleanup | try/finally |

---

## 14. Common Pitfalls

### 14.1. Забыли Dispose

```csharp
var stream = File.OpenRead(path);   // ❌ no using
ProcessStream(stream);
// stream НЕ disposed — leak до GC
```

**Фикс:** `using var stream = File.OpenRead(path);`.

### 14.2. Throw в Dispose

```csharp
public void Dispose()
{
    throw new InvalidOperationException();   // ❌
}
```

**Механизм:** breaks `using` — original exception в body lost.

**Фикс:** swallow или log в Dispose, не throw.

### 14.3. Dispose в callback без try

```csharp
var stream = File.OpenRead(path);
DoSomething(stream);   // если throw — stream не disposed!
stream.Dispose();
```

**Фикс:** `using var` или try/finally.

### 14.4. Finalizer без SuppressFinalize

```csharp
public class Bad : IDisposable
{
    ~Bad() { /* ... */ }
    public void Dispose() { /* без SuppressFinalize */ }
}
```

**Механизм:** объект попадает в finalization queue даже после Dispose — лишний GC cycle.

**Фикс:** `GC.SuppressFinalize(this)` в Dispose().

### 14.5. Disposing managed в finalizer

```csharp
~Resource()
{
    _managedField.Dispose();   // ❌ finalize order undefined
}
```

**Механизм:** managed field может быть already finalized.

**Фикс:** `Dispose(disposing=false)` — touch только unmanaged.

### 14.6. Dispose чужого resource

```csharp
public class Wrapper
{
    private Stream _injected;
    public Wrapper(Stream s) { _injected = s; }
    
    public void Dispose() => _injected.Dispose();   // ❌ caller владеет
}
```

**Фикс:** `leaveOpen` параметр или просто не disposing injected.

### 14.7. Sync Dispose ждёт async

```csharp
public void Dispose()
{
    _stream.FlushAsync().Wait();   // ❌ deadlock risk
}
```

**Фикс:** `IAsyncDisposable.DisposeAsync` через `await`.

### 14.8. ObjectDisposedException в Dispose

```csharp
public void Dispose()
{
    if (_disposed) throw new ObjectDisposedException(...);   // ❌ ломает using
}
```

**Фикс:** silent return при `_disposed`.

### 14.9. Не dispose в parameter exception

```csharp
public Resource(IDisposable dep)
{
    _dep = dep;
    if (...) throw new ArgumentException();   // ❌ dep leaks
}
```

**Фикс:** validate **до** assignment.

### 14.10. Multiple dispose race

```csharp
// Thread 1: resource.Dispose();
// Thread 2: resource.Dispose();
// Без _disposed flag — double-free
```

**Фикс:** idempotent `_disposed` flag (volatile или Interlocked для thread-safety).

> [!question]- Интервью: топ-3 ошибки с IDisposable?
> 1) **Забыли Dispose** — manual `new Stream()` без `using` → handle leak до GC. Always `using var`. 2) **Throw в Dispose** — ломает using semantics, теряется original exception. Dispose должен swallow/log. 3) **Finalizer без SuppressFinalize** — объект живёт лишний GC cycle. Always `GC.SuppressFinalize(this)` в Dispose() если есть finalizer. Бонус: dispose injected dependency — каллер должен владеть, не ты.

---

## 15. Practice — упражнения

### 15.1. Resource wrapper с правильным pattern

```csharp
public sealed class CacheFile : IDisposable
{
    private readonly FileStream _stream;
    private readonly StreamWriter _writer;
    private bool _disposed;
    
    public CacheFile(string path)
    {
        _stream = new FileStream(path, FileMode.Create);
        _writer = new StreamWriter(_stream);
    }
    
    public void Write(string line)
    {
        ObjectDisposedException.ThrowIf(_disposed, this);
        _writer.WriteLine(line);
    }
    
    public void Dispose()
    {
        if (_disposed) return;
        
        try { _writer.Dispose(); }   // flushes + closes
        catch { }
        
        try { _stream.Dispose(); }
        catch { }
        
        _disposed = true;
    }
}

using var cache = new CacheFile("cache.txt");
cache.Write("line 1");
cache.Write("line 2");
// auto-disposed
```

### 15.2. Async resource

```csharp
public sealed class HttpStreamWrapper : IAsyncDisposable, IDisposable
{
    private readonly HttpClient _client;
    private readonly Stream _stream;
    private bool _disposed;
    
    public HttpStreamWrapper(HttpClient client, Stream stream)
    {
        _client = client;
        _stream = stream;
    }
    
    public async ValueTask DisposeAsync()
    {
        if (_disposed) return;
        await _stream.DisposeAsync();
        _client.Dispose();
        _disposed = true;
        GC.SuppressFinalize(this);
    }
    
    public void Dispose()
    {
        if (_disposed) return;
        _stream.Dispose();
        _client.Dispose();
        _disposed = true;
        GC.SuppressFinalize(this);
    }
}

await using var wrapper = new HttpStreamWrapper(client, stream);
```

### 15.3. SafeHandle для native API

```csharp
public sealed class SafeWin32Handle : SafeHandleZeroOrMinusOneIsInvalid
{
    public SafeWin32Handle() : base(true) { }
    public SafeWin32Handle(IntPtr h) : base(true) => SetHandle(h);
    
    protected override bool ReleaseHandle()
    {
        return NativeMethods.CloseHandle(handle);
    }
}

public class NativeService : IDisposable
{
    private readonly SafeWin32Handle _h;
    public NativeService() => _h = NativeMethods.OpenHandle();
    public void Dispose() => _h.Dispose();
}
```

---

## 16. Что читать дальше

1. **GC и finalization** — `GC.Collect`, generations.
2. **Memory management** — managed heap, large object heap.
3. **`Span<T>` и stack allocation** — perf alternatives.
4. **System.IO Streams** — главный consumer IDisposable.
5. **HttpClient lifetime** — IHttpClientFactory.
6. **EF Core DbContext lifetime** — Scoped + automatic dispose.

---

## 17. См. также

- [[csharp-basics|C# Basics]]
- [[error-handling|Error Handling]] — exception in Dispose
- [[io-streams|IO Streams]]
- [[oop|OOP]] — inheritance + Dispose
- SafeHandle classes (Microsoft.Win32.SafeHandles)
- Microsoft.Extensions.DependencyInjection lifetimes

---

## 18. Reading list

- **Microsoft Docs — IDisposable** — learn.microsoft.com/dotnet/api/system.idisposable
- **Microsoft Docs — Dispose pattern** — learn.microsoft.com/dotnet/standard/garbage-collection/implementing-dispose
- **Microsoft Docs — IAsyncDisposable** — learn.microsoft.com/dotnet/api/system.iasyncdisposable
- **Microsoft Docs — SafeHandle** — learn.microsoft.com/dotnet/api/system.runtime.interopservices.safehandle
- **Microsoft Docs — using statement** — learn.microsoft.com/dotnet/csharp/language-reference/statements/using
- **Stephen Cleary — IAsyncDisposable** — blog.stephencleary.com
- **Eric Lippert — Dispose patterns** — ericlippert.com
- **Joe Duffy — Concurrent Programming on Windows** — book chapter on finalization
- **Jeffrey Richter — CLR via C# (4th ed.)** — chapters on memory management
