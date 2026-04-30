---
tags: [csharp, dispose, idisposable, using, async-disposable, resource-management, middle]
level: Middle
date: 2026-04-30
---

# Dispose Pattern — управление ресурсами

> **Как правильно освобождать ресурсы** в C#: IDisposable, using, IAsyncDisposable, finalizers, Dispose pattern. Daily work для middle разработчика. Closes пробел "знаю про using, но не понимаю когда писать свой Dispose".

---

## Что это, зачем и когда

### Зачем

C# имеет **garbage collection** для управляемой памяти. Но **некоторые ресурсы не управляются GC**:

- File handles (open files)
- Network connections (sockets, HTTP)
- DB connections
- Window handles, GDI objects
- Native memory (`Marshal.AllocHGlobal`)

Эти ресурсы **должны быть освобождены явно** — иначе:
- File locked
- DB connection pool иссякает
- OS handles исчерпываются
- Memory leaks

### IDisposable — контракт

```csharp
public interface IDisposable
{
    void Dispose();
}
```

Объект implements `IDisposable` → при не нужности **обязан** вызвать `Dispose()`.

### Когда нужен Dispose

✅ **Класс должен implement IDisposable если:**
- Содержит unmanaged ресурсы (HFILE, native pointers)
- Содержит managed `IDisposable` поля
- Subscribed на events другого long-living объекта (memory leak иначе!)
- Имеет CancellationTokenSource / Timer / Task с cleanup

❌ **НЕ нужен:**
- Pure data class (DTO, record)
- Stateless services (без resources)
- Math types (Vector, Money)

---

## 1. Базовое использование IDisposable

### `using` statement

```csharp
// ✅ Гарантированный Dispose даже при exception
using (var stream = new FileStream("file.txt", FileMode.Open))
{
    // ... read ...
}  // stream.Dispose() вызывается автоматически

// stream здесь уже disposed
```

Эквивалентно:

```csharp
var stream = new FileStream("file.txt", FileMode.Open);
try
{
    // ... read ...
}
finally
{
    stream.Dispose();
}
```

### `using` declaration (C# 8+)

```csharp
public void ProcessFile(string path)
{
    using var stream = new FileStream(path, FileMode.Open);
    using var reader = new StreamReader(stream);
    // ... use ...
}  // оба disposed at end of method (в обратном порядке)
```

### Multiple resources

```csharp
// Старый стиль — nested
using (var conn = new SqlConnection(connStr))
using (var cmd = conn.CreateCommand())
using (var reader = cmd.ExecuteReader())
{
    // ...
}

// C# 8+ — declarations
using var conn = new SqlConnection(connStr);
using var cmd = conn.CreateCommand();
using var reader = cmd.ExecuteReader();
// ... все disposed at end of scope
```

### Dispose order

```csharp
{
    using var a = new A();   // creates
    using var b = new B();   // creates
}  // Dispose в обратном порядке: b.Dispose(), потом a.Dispose()
```

LIFO — last in, first out.

---

## 2. Implementing IDisposable — простой случай

### Только managed ресурсы

```csharp
public class FileLogger : IDisposable
{
    private readonly StreamWriter _writer;
    private bool _disposed;

    public FileLogger(string path)
    {
        _writer = new StreamWriter(path, append: true);
    }

    public void Log(string message) =>
        _writer.WriteLine($"[{DateTime.UtcNow:o}] {message}");

    public void Dispose()
    {
        if (_disposed) return;
        _writer?.Dispose();
        _disposed = true;
    }
}

// Use
using var logger = new FileLogger("app.log");
logger.Log("Started");
```

### Pattern: Dispose multiple times — safe

```csharp
public void Dispose()
{
    if (_disposed) return;  // ⚠️ Idempotent!
    _writer?.Dispose();
    _disposed = true;
}

// Multiple Dispose calls — OK
var logger = new FileLogger("a.log");
logger.Dispose();
logger.Dispose();  // ✅ no double-dispose error
logger.Dispose();
```

> [!info] Microsoft guideline
> `Dispose()` должен быть **idempotent** — multiple calls не throw.

### ObjectDisposedException

```csharp
public void Log(string message)
{
    if (_disposed)
        throw new ObjectDisposedException(nameof(FileLogger));
    _writer.WriteLine(message);
}
```

После Dispose — операции должны throw. Иначе bugs от использования "мёртвого" объекта.

---

## 3. Full Dispose pattern (классический)

Для **базовых классов** с **unmanaged ресурсами** или для классов которые могут быть наследованы:

```csharp
public class ResourceHolder : IDisposable
{
    private bool _disposed;
    private IntPtr _nativeHandle;       // unmanaged
    private StreamWriter? _writer;       // managed

    public ResourceHolder()
    {
        _nativeHandle = AllocateNative();
        _writer = new StreamWriter("log.txt");
    }

    // Public Dispose
    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);  // не запускать finalizer (мы уже cleaned up)
    }

    // Protected virtual — для наследников override
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // Free managed resources
            _writer?.Dispose();
            _writer = null;
        }

        // Free unmanaged resources (всегда, даже если disposing=false)
        if (_nativeHandle != IntPtr.Zero)
        {
            FreeNative(_nativeHandle);
            _nativeHandle = IntPtr.Zero;
        }

        _disposed = true;
    }

    // Finalizer — fallback если Dispose не вызвали
    ~ResourceHolder()
    {
        Dispose(disposing: false);
    }

    [DllImport("native.dll")] private static extern IntPtr AllocateNative();
    [DllImport("native.dll")] private static extern void FreeNative(IntPtr h);
}
```

### Зачем `disposing` параметр

| `disposing = true` | `disposing = false` |
|---------------------|---------------------|
| Вызван из `Dispose()` — приложение явно cleanup | Вызван из finalizer — GC cleanup |
| Можно trogать managed objects (`_writer.Dispose()`) | НЕЛЬЗЯ — managed objects уже finalized возможно |
| Free всё | Free только unmanaged |

> [!warning] Finalizer не trogать managed!
> Когда finalizer запущен — другие объекты могут быть уже finalized. Trogать их → undefined behavior.

### Inheritance

```csharp
public class DerivedResource : ResourceHolder
{
    private FileStream? _stream;

    protected override void Dispose(bool disposing)
    {
        if (disposing)
        {
            _stream?.Dispose();
            _stream = null;
        }

        base.Dispose(disposing);  // ⚠️ Не забудь base!
    }
}
```

---

## 4. SafeHandle — modern way для unmanaged

`SafeHandle` — лучше чем raw IntPtr для unmanaged handles.

```csharp
public class FileWrapper : IDisposable
{
    private readonly SafeFileHandle _handle;

    public FileWrapper(string path)
    {
        _handle = File.OpenHandle(path);
    }

    public void Dispose()
    {
        _handle?.Dispose();  // SafeHandle thread-safe + auto-cleanup
    }
}
```

`SafeHandle` сам managing finalizer + thread safety + reference counting. **Используй вместо IntPtr** где возможно.

См. [[../Runtime/interop-pinvoke|Interop & PInvoke]].

---

## 5. IAsyncDisposable (.NET Core 3+)

Для async cleanup — например flush stream, send goodbye message.

```csharp
public class AsyncResource : IAsyncDisposable
{
    private readonly Stream _stream;
    private readonly HttpClient _http;

    public AsyncResource()
    {
        _stream = new FileStream("file.txt", FileMode.Create);
        _http = new HttpClient();
    }

    public async ValueTask DisposeAsync()
    {
        await _stream.DisposeAsync();   // async dispose
        _http.Dispose();                  // sync (HttpClient не async)
    }
}

// Use
await using var resource = new AsyncResource();
// auto async dispose at end of scope
```

### `await using` vs `using`

```csharp
// Sync
using var stream = new FileStream("f.txt", FileMode.Open);

// Async
await using var stream = new FileStream("f.txt", FileMode.Open);
// stream.DisposeAsync() будет awaited
```

### Implement и Dispose, и DisposeAsync

```csharp
public class DualResource : IDisposable, IAsyncDisposable
{
    private readonly Stream _stream;

    public DualResource() => _stream = new FileStream(...);

    // Sync
    public void Dispose()
    {
        _stream?.Dispose();
    }

    // Async
    public async ValueTask DisposeAsync()
    {
        if (_stream is not null)
            await _stream.DisposeAsync();
    }
}
```

> [!info] DisposeAsync делает то же что Dispose async-friendly
> Если есть и тот и тот — выбирай DisposeAsync в async коде.

---

## 6. Common patterns

### Pattern 1: Wrap external resource

```csharp
public class TempFile : IDisposable
{
    public string Path { get; }
    private bool _disposed;

    public TempFile(string content)
    {
        Path = System.IO.Path.GetTempFileName();
        File.WriteAllText(Path, content);
    }

    public void Dispose()
    {
        if (_disposed) return;
        try { File.Delete(Path); } catch { /* ignore */ }
        _disposed = true;
    }
}

// Use
using var tmp = new TempFile("test data");
ProcessFile(tmp.Path);
// File автоматически удалится
```

### Pattern 2: Scope timing

```csharp
public class TimingScope : IDisposable
{
    private readonly string _operation;
    private readonly Stopwatch _stopwatch;

    public TimingScope(string operation)
    {
        _operation = operation;
        _stopwatch = Stopwatch.StartNew();
        Console.WriteLine($"Starting: {_operation}");
    }

    public void Dispose()
    {
        _stopwatch.Stop();
        Console.WriteLine($"Finished: {_operation} in {_stopwatch.ElapsedMilliseconds}ms");
    }
}

// Use
using (new TimingScope("Process orders"))
{
    foreach (var order in orders) Process(order);
}
// Output:
// Starting: Process orders
// Finished: Process orders in 1234ms
```

### Pattern 3: Subscription cleanup (event memory leak prevention)

```csharp
public class Subscriber : IDisposable
{
    private readonly Publisher _publisher;
    private readonly EventHandler _handler;

    public Subscriber(Publisher publisher)
    {
        _publisher = publisher;
        _handler = OnEvent;
        _publisher.SomeEvent += _handler;  // subscribe
    }

    private void OnEvent(object sender, EventArgs e) { /* ... */ }

    public void Dispose()
    {
        _publisher.SomeEvent -= _handler;  // unsubscribe — KEY!
    }
}

// Без Dispose — Publisher держит reference на Subscriber через event
// → Subscriber не GC'd → memory leak
```

См. [[delegates-events|Delegates и Events]].

### Pattern 4: CancellationTokenSource cleanup

```csharp
public class BackgroundProcessor : IDisposable
{
    private readonly CancellationTokenSource _cts = new();
    private Task? _backgroundTask;

    public void Start() =>
        _backgroundTask = Task.Run(() => Process(_cts.Token));

    private async Task Process(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            await DoWorkAsync();
            await Task.Delay(1000, ct);
        }
    }

    public void Dispose()
    {
        _cts.Cancel();
        try { _backgroundTask?.Wait(TimeSpan.FromSeconds(5)); }
        catch (AggregateException) { /* expected — cancellation */ }
        _cts.Dispose();
    }
}
```

### Pattern 5: Reader-Writer lock

```csharp
public class ReadLockScope : IDisposable
{
    private readonly ReaderWriterLockSlim _lock;

    public ReadLockScope(ReaderWriterLockSlim @lock)
    {
        _lock = @lock;
        _lock.EnterReadLock();
    }

    public void Dispose() => _lock.ExitReadLock();
}

public class Cache
{
    private readonly ReaderWriterLockSlim _lock = new();

    public T Get<T>(string key)
    {
        using var _ = new ReadLockScope(_lock);
        return Lookup<T>(key);
    }
}
```

### Pattern 6: Compound IDisposable

```csharp
public sealed class CompositeDisposable : IDisposable
{
    private readonly List<IDisposable> _disposables = new();

    public void Add(IDisposable disposable) => _disposables.Add(disposable);

    public void Dispose()
    {
        foreach (var d in _disposables.AsEnumerable().Reverse())
        {
            try { d.Dispose(); }
            catch { /* swallow — все должны быть disposed */ }
        }
        _disposables.Clear();
    }
}

// Use
using var composite = new CompositeDisposable();
composite.Add(new FileStream(...));
composite.Add(new StreamReader(...));
composite.Add(new SqlConnection(...));
// All disposed in reverse order
```

---

## 7. Common Pitfalls

### 1. Не dispose'ить вообще

```csharp
// ❌ Stream остаётся открытым — file locked
public void Read()
{
    var stream = new FileStream("f.txt", FileMode.Open);
    var content = new StreamReader(stream).ReadToEnd();
    return content;
}

// ✅ using
public string Read()
{
    using var stream = new FileStream("f.txt", FileMode.Open);
    using var reader = new StreamReader(stream);
    return reader.ReadToEnd();
}
```

### 2. Forgot unsubscribe events

```csharp
public class Subscriber
{
    public Subscriber(Publisher p)
    {
        p.Event += OnEvent;  // ❌ never unsubscribe → memory leak
    }
}
```

См. [[delegates-events|Delegates и Events]] — weak event pattern.

### 3. Async + sync Dispose

```csharp
public class AsyncService : IDisposable
{
    public async Task SaveAsync() { /* async work */ }

    public void Dispose()
    {
        SaveAsync().Wait();  // ⚠️ Blocks thread, deadlock potential!
    }
}

// ✅ Implement IAsyncDisposable
public class AsyncService : IAsyncDisposable
{
    public async ValueTask DisposeAsync()
    {
        await SaveAsync();
    }
}
```

### 4. Dispose не idempotent

```csharp
// ❌
public void Dispose()
{
    _stream.Dispose();  // 2-й call → ObjectDisposedException на _stream!
}

// ✅
private bool _disposed;
public void Dispose()
{
    if (_disposed) return;
    _stream.Dispose();
    _disposed = true;
}
```

### 5. Wrong order

```csharp
// ❌ stream закрыт, reader не может flush
public void Dispose()
{
    _stream.Dispose();  // closed first
    _reader.Dispose();   // tries flush → throws
}

// ✅ Reverse order — closer dependencies first
public void Dispose()
{
    _reader.Dispose();   // flushes to stream
    _stream.Dispose();    // closes stream
}
```

> [!info] `using` declarations авто LIFO
> `using var a = ...; using var b = ...;` → b disposed first.

### 6. Dispose в exception

```csharp
public void Process()
{
    var resource = new Resource();
    DoWork(resource);  // throws!
    resource.Dispose();  // ⚠️ Никогда не вызовется
}

// ✅ using
public void Process()
{
    using var resource = new Resource();
    DoWork(resource);
}  // Dispose даже при exception
```

### 7. Field assignment в Dispose

```csharp
public void Dispose()
{
    if (_writer != null)
    {
        _writer.Dispose();
        _writer = null;  // ⚠️ helps GC ON LARGE OBJECTS
    }
}
// Обычно не нужно — но для long-living objects полезно
```

### 8. Capturing `this` в lambda → не disposed

```csharp
public class Service : IDisposable
{
    public void Subscribe(Publisher p)
    {
        p.Event += (s, e) => HandleEvent(this);  // ⚠️ captures this!
    }
    // Publisher держит lambda → держит this → Service never GC'd
}

// ✅ Save handler reference for unsubscribe
private EventHandler? _handler;

public void Subscribe(Publisher p)
{
    _handler = (s, e) => HandleEvent(this);
    p.Event += _handler;
}

public void Dispose()
{
    _publisher.Event -= _handler;
}
```

### 9. ASP.NET Core service Disposable lifetime

```csharp
// Если ваш Service implements IDisposable, ASP.NET Core автоматически dispose
services.AddScoped<MyService>();   // disposed at end of request
services.AddSingleton<MyService>(); // disposed at app shutdown
services.AddTransient<MyService>(); // ⚠️ disposed at end of scope (но scope = request)
```

### 10. Forget GC.SuppressFinalize

```csharp
public class WithFinalizer : IDisposable
{
    ~WithFinalizer() { Dispose(false); }  // finalizer

    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);  // ⚠️ Без этого — finalizer всё равно запустится!
    }

    protected virtual void Dispose(bool disposing) { /* ... */ }
}
```

`SuppressFinalize` — говорит GC: "не запускай finalizer, я уже cleaned up". Performance — finalizer expensive.

---

## 8. Best Practices

### Implementation

- **`using` everywhere** — никогда не вызывай `.Dispose()` вручную
- **`using` declarations** (C# 8+) — короче чем blocks
- **Idempotent Dispose** — `if (_disposed) return;`
- **`ObjectDisposedException`** на public members после Dispose
- **`GC.SuppressFinalize(this)`** если есть finalizer
- **Dispose в обратном порядке** создания

### Design

- **Records / DTO — НЕ Disposable** — pure data
- **`SafeHandle`** вместо raw `IntPtr` для unmanaged
- **Composition** — wrap disposable, dispose в Dispose()
- **`IAsyncDisposable`** для async cleanup
- **Event unsubscription** обязательно в Dispose

### Async

- **`await using`** — для IAsyncDisposable
- **Не блокируй sync Dispose async кодом** (`.Wait()`, `.Result`)
- **Implement обоих** если нужно — IDisposable + IAsyncDisposable

### Finalizers

- **Только если есть unmanaged resources не через SafeHandle**
- **Никогда не trogай managed objects** в finalizer
- **`GC.SuppressFinalize(this)`** в Dispose
- **Используй `SafeHandle`** чтобы избежать ручного finalizer

---

## 9. IDisposable in DI

ASP.NET Core / .NET DI **автоматически вызывает Dispose**:

```csharp
public class MyService : IDisposable
{
    private readonly Connection _conn;

    public MyService() => _conn = new Connection();

    public void Dispose() => _conn.Dispose();
}

// Регистрация
services.AddScoped<MyService>();  // disposed at end of scope (request)
services.AddSingleton<MyService>(); // disposed at app shutdown

// Используешь — DI сам dispose
public class Controller(MyService svc) : ControllerBase
{
    public IActionResult Get() => Ok(svc.Process());
}
// svc.Dispose() вызовется автоматически после request
```

См. [[../AspNetCore/di-configuration|DI & Configuration]].

> [!warning] HttpClient — не dispose в обычном случае!
> ```csharp
> // ❌ Создавать HttpClient на каждый запрос — socket exhaustion
> using var http = new HttpClient();
> 
> // ✅ IHttpClientFactory — manages lifecycle
> services.AddHttpClient<MyApi>();
> ```

---

## 10. Cheat sheet

| Сценарий | Solution |
|----------|----------|
| Использовать disposable | `using var x = new T();` |
| Несколько ресурсов | `using` declarations C# 8+ |
| Implement basic | `IDisposable` + `if (_disposed) return;` |
| Async cleanup | `IAsyncDisposable` + `await using` |
| Inheritance | virtual `Dispose(bool disposing)` |
| Unmanaged resource | `SafeHandle` (preferred) или finalizer |
| Subscribe to event | unsubscribe в Dispose обязательно |
| ASP.NET service | DI вызовет Dispose автоматически |
| HttpClient | НЕ создавай — IHttpClientFactory |
| Suppress finalizer | `GC.SuppressFinalize(this)` |
| Multiple Disposables | CompositeDisposable pattern |
| Scope-based timing | Custom `IDisposable` в `using` |

---

## 11. Decision tree

```
Класс держит ресурсы?
│
├── Только managed (другие IDisposable)?
│   → Implement IDisposable
│   → В Dispose() вызвать Dispose() для каждого
│   → НЕТ finalizer
│
├── Unmanaged (IntPtr, file handles)?
│   ├── Best: SafeHandle wrapper
│   └── Иначе: full Dispose pattern + finalizer
│
├── Async cleanup нужен?
│   → IAsyncDisposable
│   → DisposeAsync() returns ValueTask
│
├── Subscribed на event?
│   → IDisposable обязательно
│   → Unsubscribe в Dispose
│
└── Чистый data?
    → НЕ нужен IDisposable

Использование:
│
├── using statement или declaration
├── НЕ вызывай Dispose() вручную
├── DI автоматически dispose
└── try/finally если using не подходит
```

---

## См. также

- [[csharp-basics|C# Basics]] — using intro
- [[oop|OOP]] — inheritance and Dispose pattern
- [[../Runtime/gc-memory|GC и память]] — finalizer queue, generations
- [[../Runtime/interop-pinvoke|Interop & PInvoke]] — SafeHandle
- [[delegates-events|Delegates и Events]] — event memory leaks
- [[async-threading|Async и Threading]] — IAsyncDisposable
- [[io-streams|I/O и Streams]] — Stream disposal
- [[../AspNetCore/di-configuration|DI & Configuration]] — service lifetimes

## Reading list

- **Microsoft Docs — Dispose pattern** — learn.microsoft.com/dotnet/standard/garbage-collection/implementing-dispose
- **Microsoft Docs — Cleanup unmanaged resources** — learn.microsoft.com
- **Microsoft Docs — IAsyncDisposable** — learn.microsoft.com/dotnet/api/system.iasyncdisposable
- **Eric Lippert — Finalizers** — ericlippert.com
- **Stephen Toub — IAsyncDisposable** — devblogs.microsoft.com/dotnet
- **CLR via C#** — Jeffrey Richter (chapter про GC и finalizers)
