---
tags: [csharp, async, threading, senior, task, parallelism, concurrency, synchronization, channels]
level: Senior
date: 2026-05-10
---

# Async и Threading — concurrency в C#

> **`async/await` глубоко, Task vs ValueTask, threading primitives, `lock`/Mutex/Semaphore, `Parallel`, `Channel<T>`, `IAsyncEnumerable<T>`, deadlocks.** Когда async vs threads vs parallel, как избегать deadlocks, race conditions, что выбрать для производительности. Закрывает пробел: «знаю про `await`, не понимаю SynchronizationContext, ConfigureAwait, и почему ASP.NET Core не имеет deadlock проблем а WPF имеет».

---

## 0. Как читать

Если впервые — раздел 1 (mental model) → 3 (async/await deep) → 6 (synchronization). Parallelism — раздел 8. Channels / streams — раздел 9. Production guidance — раздел 11 (best practices), 13 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Three concepts

```
Concurrency — multiple things happening "at once" (logical)
  - Async I/O (await network call)
  - One thread switching между tasks

Parallelism — physical simultaneity (multiple cores)
  - Parallel.ForEach
  - PLINQ
  - Task.Run on different cores

Threading — manual thread management
  - Thread class
  - Locks, signals, semaphores
  - Lower-level than async/parallel
```

### 1.2. Когда что использовать

```
Async/await:
✅ I/O bound (network, disk, database)
✅ Releasing thread during wait
✅ ASP.NET requests, UI responsiveness

Parallelism:
✅ CPU bound (calculations, image processing)
✅ Independent work splittable
✅ Multiple cores available

Threading (manual):
✅ Long-running background work
✅ Custom synchronization
✅ Performance critical lock-free code
✅ Producer/consumer queues

Не путай:
❌ async ≠ parallel (async один thread, parallel many threads)
❌ async НЕ делает code faster (just frees threads during I/O)
❌ Parallel.ForEach НЕ всегда faster (overhead, contention)
```

### 1.3. Главное правило

```
I/O bound → async/await + Task<T>
CPU bound + parallel → Parallel.ForEach или PLINQ
Long-running background → Task.Run + CancellationToken или BackgroundService
Producer/consumer → Channel<T> или BlockingCollection<T>
Synchronization → SemaphoreSlim (async-aware) или lock (sync)
Streaming → IAsyncEnumerable<T>

Avoid:
- Thread.Sleep в production (use Task.Delay)
- .Result / .Wait (deadlock potential — use await)
- lock в async method
- Manual Thread management (use Task)
```

### 1.4. Эволюция

| Версия | Что |
|--------|-----|
| **.NET 1.0** | Thread, ThreadPool, ManualResetEvent |
| **.NET 4.0** | TPL (Task Parallel Library), `Task<T>`, `Parallel.ForEach`, `CancellationToken` |
| **C# 5.0** | `async/await` keywords |
| **.NET 4.5** | `ConfigureAwait`, `TaskCompletionSource` |
| **.NET Core 2.1** | `ValueTask<T>`, performance improvements |
| **C# 8.0** | `IAsyncEnumerable<T>`, `await foreach`, `await using` |
| **.NET Core 3.0** | `Channel<T>`, async-friendly primitives |
| **.NET 6+** | `[AsyncMethodBuilder]` improvements, `ParallelLoopState` |
| **.NET 8+** | Better thread pool, AOT improvements |

> [!info]- Если ты знаешь Java / Kotlin / Go / Rust / Python
> **Java:** `CompletableFuture` chain composition, virtual threads (Java 21+, similar to goroutines).
>
> **Kotlin:** coroutines (structured concurrency, scope-based cancellation). Different mental model.
>
> **Go:** goroutines + channels (CSP). Lightweight, preemptive. C# `Channel<T>` similar concept.
>
> **Rust:** `async/await` + executors (tokio). No GC, ownership prevents data races. C# borrowed syntax.
>
> **Python:** `asyncio`, `async/await`. Similar but GIL prevents true parallelism.

> [!question]- Интервью: чем async отличается от threading?
> **Async/await** — **logical concurrency** через state machine. One thread может handle many awaiting tasks (releases thread during I/O wait). No new thread per await. **Threading** — physical OS threads. Each Thread = OS resource (~1MB stack). Thread.Sleep blocks thread. **Parallel** — uses thread pool, splits CPU work across cores. **When async**: I/O bound (network, disk, DB) — frees thread during wait. **When threads/parallel**: CPU bound — needs actual cores. **When manual Thread**: long-running background, custom synchronization. **Async НЕ faster** — just frees threads. **Common mistake**: `await Task.Run(() => ...)` для CPU bound в ASP.NET — uses thread pool worker, blocks request handling thread.

---

## 2. Threads basics

### 2.1. Thread class

```csharp
var thread = new Thread(() =>
{
    Console.WriteLine($"Hello from thread {Environment.CurrentManagedThreadId}");
});
thread.Start();
thread.Join();   // wait for completion

// With parameter
var thread2 = new Thread(p =>
{
    Console.WriteLine($"Got: {p}");
});
thread2.Start("Hello");

// Background thread (doesn't prevent app exit)
var bg = new Thread(() => { /* ... */ }) { IsBackground = true };
bg.Start();
```

`Thread` — low-level, rarely used directly в modern C#. Prefer `Task`.

### 2.2. ThreadPool

```csharp
ThreadPool.QueueUserWorkItem(state =>
{
    Console.WriteLine("Running on pool thread");
});
```

Pre-allocated pool of threads. Reused для short tasks. `Task.Run` uses ThreadPool internally.

### 2.3. Thread cost

```
Thread:
- Stack ~1MB by default
- OS thread (kernel resource)
- Context switch ~5-10μs
- Limited (~100s feasible)

ThreadPool worker:
- Reused
- Lower overhead
- Limited (default min = #cores, max ~32k)

async/await на ThreadPool:
- One thread handles many tasks
- Effectively "millions" of concurrent operations
```

### 2.4. Thread safety

```csharp
// ❌ Race condition
int counter = 0;
Parallel.For(0, 1000, _ => counter++);
// Result: НЕ обязательно 1000 — race

// ✅ Interlocked
int safeCounter = 0;
Parallel.For(0, 1000, _ => Interlocked.Increment(ref safeCounter));
// Result: 1000

// ✅ lock
object lockObj = new();
int locked = 0;
Parallel.For(0, 1000, _ =>
{
    lock (lockObj) { locked++; }
});
```

### 2.5. Thread.Sleep vs Task.Delay

```csharp
// ❌ Thread.Sleep — blocks thread
Thread.Sleep(1000);   // thread sits doing nothing

// ✅ Task.Delay — releases thread
await Task.Delay(1000);   // thread freed для другой work
```

В production — **always Task.Delay**. Thread.Sleep только для unit tests или special cases.

### 2.6. Manual Thread сейчас редко

```
Use Thread directly когда:
- Long-running dedicated thread (rare)
- Specific OS thread requirements (apartment state COM)

Otherwise:
- async/await — для I/O
- Task.Run — для CPU work
- BackgroundService — long-running work
- Parallel.ForEach — parallel CPU
- Channel<T> — producer/consumer
```

> [!question]- Интервью: что такое ThreadPool?
> **Pre-allocated pool of OS threads**, reused для work items. Avoids overhead of creating/destroying threads per task. **Used by**: `Task.Run`, `async/await` continuations (по default), ASP.NET Core requests, `Parallel.ForEach`. **Configuration**: min threads (default ~#cores) + max threads (~32k). **`ThreadPool.SetMinThreads`** — pre-warm для startup spikes (rare). **Common issue**: thread pool starvation — все workers blocked, new tasks queue. Causes: blocking calls (`.Result`, `.Wait`), too much synchronous work. **Fix**: avoid blocking, use async, increase min threads если justified. **Best practice**: don't manage threads manually — Task abstraction better.

---

## 3. async/await deep

### 3.1. Что async/await делает

```csharp
public async Task<string> FetchAsync(string url)
{
    var response = await client.GetAsync(url);
    var content = await response.Content.ReadAsStringAsync();
    return content;
}

// Compiler transforms ↓ (simplified)
public Task<string> FetchAsync(string url)
{
    var stateMachine = new FetchStateMachine();
    stateMachine.Builder = AsyncTaskMethodBuilder<string>.Create();
    stateMachine.url = url;
    stateMachine.State = -1;
    stateMachine.Builder.Start(ref stateMachine);
    return stateMachine.Builder.Task;
}
```

`async/await` — compiler transforms method into **state machine**. Each `await` = potential suspension point.

### 3.2. State machine

```
async method execution:
1. Start synchronously
2. Hit await — check if Task already completed
   - Yes: continue synchronously
   - No: register continuation, return Task
3. When awaited Task completes — continuation invoked
4. Continue до next await или completion
5. Return final result
```

**Key insight**: `async` method doesn't necessarily run on different thread. Runs synchronously until first incomplete await.

### 3.3. SynchronizationContext

```csharp
// WPF / WinUI / WinForms — UI SynchronizationContext
public async Task ButtonClick()
{
    var data = await client.GetStringAsync(url);   // background thread
    label.Text = data;   // UI thread (auto-marshaled by SyncContext!)
}

// ASP.NET Core — null SynchronizationContext (with .NET Core)
public async Task<IActionResult> Get()
{
    var data = await client.GetStringAsync(url);
    return Ok(data);   // any thread (no marshaling needed)
}
```

`SynchronizationContext` — controls where continuation runs:
- WPF/WinUI/MAUI: UI thread.
- ASP.NET (legacy .NET Framework): request thread (default).
- ASP.NET Core: null (no special context).
- Console: null.

### 3.4. ConfigureAwait

```csharp
// Marshal back to original context (default)
var data = await client.GetAsync(url);   // SyncContext respected

// Don't marshal — стay на pool thread
var data = await client.GetAsync(url).ConfigureAwait(false);
```

**`ConfigureAwait(false)`**:
- ✅ Library code (no UI access needed) — recommended.
- ❌ ASP.NET Core — irrelevant (null SyncContext anyway).
- ❌ UI app (need to access UI после await) — would break.

### 3.5. Task vs ValueTask

```csharp
public async Task<int> SlowMethodAsync()
{
    await Task.Delay(100);
    return 42;
}

public async ValueTask<int> FastMethodAsync()
{
    if (_cache.TryGetValue(key, out var value)) return value;   // sync hot path
    return await SlowFetchAsync();
}
```

| | `Task<T>` | `ValueTask<T>` |
|---|---------|---------------|
| Type | reference type (heap) | struct (no allocation if sync-completed) |
| Cost | allocation per call | zero alloc для sync paths |
| Use case | always async | mostly sync (cache hit) |
| Restriction | reusable | **cannot await twice** |

`ValueTask<T>` — micro-optimization для hot paths где often completes synchronously.

### 3.6. Common return types

```csharp
async Task DoWork() { /* no return value */ }
async Task<int> GetCount() { /* return int */ return 42; }
async ValueTask<int> FastGet() { /* mostly sync */ return _value; }
async IAsyncEnumerable<int> Stream() { yield return 1; await Task.Delay(100); yield return 2; }
async void HandleClick(...) { /* event handler ONLY */ }   // ❌ Avoid otherwise
```

### 3.7. async void — avoid!

```csharp
// ❌ async void — no Task to await, exceptions crash app
public async void Bad() { await SomeAsync(); }

// ❌ Caller can't await
Bad();   // returns void
// Exceptions in async void → unhandled, crash process

// ✅ async Task
public async Task Good() { await SomeAsync(); }
```

**Only acceptable**: event handlers (UI button click) where `void` required by signature.

> [!question]- Интервью: что такое ConfigureAwait(false)?
> Tells `await` **не marshal continuation back to original SynchronizationContext**. Continuation runs на ThreadPool worker. **When use**: 1) **Library code** — don't know if caller is UI/ASP.NET/Console. ConfigureAwait(false) prevents unintended UI thread access + improves performance. 2) **Server code** — but ASP.NET Core has no SyncContext, so irrelevant. **When NOT use**: 1) **UI app code** — need to access UI после await. ConfigureAwait(true) (default) ensures continuation на UI thread. 2) **ASP.NET Core controllers** — no SyncContext, doesn't matter. **Best practice 2024+**: in ASP.NET Core — don't bother. In libraries — always ConfigureAwait(false). In UI apps — only use в helper методы that don't touch UI.

---

## 4. Task fundamentals

### 4.1. Creating Tasks

```csharp
// Task.Run — schedule на ThreadPool
Task<int> task = Task.Run(() => Compute());

// Task.FromResult — already-completed
Task<int> ready = Task.FromResult(42);

// Task.CompletedTask — completed Task (no result)
Task done = Task.CompletedTask;

// Task.Delay — completes after delay
await Task.Delay(1000);

// TaskCompletionSource — manual control
var tcs = new TaskCompletionSource<int>();
// later:
tcs.SetResult(42);   // or SetException, SetCanceled
Task<int> futureTask = tcs.Task;
```

### 4.2. Combinators

```csharp
// WhenAll — wait для всех
Task<string> t1 = client.GetStringAsync(url1);
Task<string> t2 = client.GetStringAsync(url2);
Task<string> t3 = client.GetStringAsync(url3);
string[] results = await Task.WhenAll(t1, t2, t3);

// WhenAny — wait для первого
Task<string> winner = await Task.WhenAny(t1, t2, t3);
string result = await winner;

// Cancellation после timeout
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
try { await SomeOperation(cts.Token); }
catch (OperationCanceledException) { /* timed out */ }
```

### 4.3. Cancellation

```csharp
public async Task DownloadAsync(string url, CancellationToken ct)
{
    using var response = await client.GetAsync(url, ct);
    using var stream = await response.Content.ReadAsStreamAsync(ct);
    
    var buffer = new byte[8192];
    int read;
    while ((read = await stream.ReadAsync(buffer, ct)) > 0)
    {
        ct.ThrowIfCancellationRequested();   // periodic check
        // process
    }
}

// Caller
using var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(30));   // auto-cancel after 30s
try
{
    await DownloadAsync("https://...", cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Cancelled");
}
```

`CancellationToken` — cooperative cancellation. Method must check periodically (`ct.ThrowIfCancellationRequested()`) или pass to async APIs.

### 4.4. Exceptions

```csharp
public async Task DoAsync()
{
    try
    {
        await client.GetStringAsync(url);
    }
    catch (HttpRequestException ex)
    {
        // handle
    }
}

// Aggregate exceptions с WhenAll
try
{
    await Task.WhenAll(t1, t2, t3);
}
catch (Exception ex)
{
    // Only first exception thrown
    // To get all:
    var allEx = (t1.IsFaulted ? t1.Exception : null);
    // ... combine
}
```

`async/await` unwraps `AggregateException` — caller sees first inner exception.

### 4.5. Continuations

```csharp
// Old style (pre-async) — ContinueWith
Task<string> task = client.GetStringAsync(url);
task.ContinueWith(t =>
{
    if (t.IsFaulted) Console.WriteLine($"Error: {t.Exception}");
    else Console.WriteLine($"Got: {t.Result}");
});

// Modern — async/await
try { var result = await client.GetStringAsync(url); }
catch (Exception ex) { /* ... */ }
```

`async/await` strictly cleaner than `ContinueWith` chains.

### 4.6. Hot vs Cold tasks

```csharp
// Tasks created with Task.Run / async methods — HOT (already running)
Task<int> hot = Task.Run(() => Compute());
// Compute already running на ThreadPool!

// Lazy<Task<T>> — cold (only starts when accessed)
Lazy<Task<int>> lazy = new(() => Task.Run(() => Compute()));
// Not running yet
var result = await lazy.Value;   // starts now
```

C# Tasks — almost always **hot** (started). For lazy — wrap в `Lazy<Task<T>>`.

> [!question]- Интервью: что такое CancellationToken и зачем он?
> **Cooperative cancellation** mechanism. **`CancellationTokenSource`** owner creates + cancels. **`CancellationToken`** consumer checks periodically. **Pattern**: 1) Caller creates `CancellationTokenSource`. 2) Pass `cts.Token` to async method. 3) Method checks `ct.ThrowIfCancellationRequested()` periodically OR passes to inner async APIs. 4) Caller calls `cts.Cancel()` when needed. **Auto-cancel**: `cts.CancelAfter(timeout)`. **Linked tokens**: `CancellationTokenSource.CreateLinkedTokenSource(token1, token2)` — cancel when any source cancels. **Best practice**: pass through chain, every async method accepts CancellationToken parameter (default for last position). **Throws**: `OperationCanceledException` when cancelled. Catch separately from regular errors.

---

## 5. async patterns

### 5.1. Sync over async — danger

```csharp
// ❌ DEADLOCK risk в WPF/WinUI/legacy ASP.NET
public string GetData()
{
    return GetDataAsync().Result;   // blocks UI thread
    // .GetAwaiter().GetResult() same problem
}

// In WPF: UI thread waits для Task. Task continuation needs UI thread (SyncContext). Deadlock!

// ✅ Make caller async
public async Task<string> GetData()
{
    return await GetDataAsync();
}
```

**Rule**: don't `.Result` / `.Wait()` async methods. ASP.NET Core safer (no SyncContext) but still degrades thread pool.

### 5.2. Async over sync

```csharp
// CPU bound — wrap в Task.Run
public async Task<int> ComputeAsync(int n)
{
    return await Task.Run(() => HeavyComputation(n));   // moves to pool
}

// Use case: don't block ASP.NET request thread
// Caveat: don't do this in libraries — caller may not want offloading
```

### 5.3. Fire-and-forget — careful

```csharp
// ❌ Anti-pattern — exceptions lost
public void HandleRequest()
{
    Task.Run(async () =>
    {
        await SendNotificationAsync();   // if throws — silent
    });
}

// ✅ Pattern — Task tracking
public Task BackgroundWorkAsync()
{
    return Task.Run(async () =>
    {
        try { await SendNotificationAsync(); }
        catch (Exception ex) { _logger.LogError(ex, "Failed"); }
    });
}

// Caller: var bg = BackgroundWorkAsync();   // tracked Task
// Or just: _ = BackgroundWorkAsync();   // explicit fire-and-forget с logging
```

### 5.4. Async lazy

```csharp
public class Service
{
    private readonly AsyncLazy<HttpClient> _client = new(async () =>
    {
        var httpClient = new HttpClient();
        await ConfigureAsync(httpClient);
        return httpClient;
    });
    
    public async Task UseAsync()
    {
        var client = await _client.Value;
        await client.GetAsync(...);
    }
}

// Or simple Lazy<Task<T>>
private readonly Lazy<Task<HttpClient>> _client = new(async () =>
{
    var c = new HttpClient();
    await Task.Delay(100);
    return c;
});
```

### 5.5. Throttling parallel async

```csharp
// ❌ Unbounded — может start 1000s tasks at once
var tasks = urls.Select(u => client.GetStringAsync(u));
var results = await Task.WhenAll(tasks);   // all in flight!

// ✅ SemaphoreSlim throttle
var semaphore = new SemaphoreSlim(10);   // max 10 concurrent
var tasks = urls.Select(async u =>
{
    await semaphore.WaitAsync();
    try { return await client.GetStringAsync(u); }
    finally { semaphore.Release(); }
});
var results = await Task.WhenAll(tasks);

// .NET 6+: Parallel.ForEachAsync
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (url, ct) =>
    {
        var data = await client.GetStringAsync(url, ct);
        // process
    });
```

### 5.6. Retry с exponential backoff

```csharp
public static async Task<T> RetryAsync<T>(
    Func<Task<T>> action,
    int maxAttempts = 3,
    TimeSpan? baseDelay = null)
{
    baseDelay ??= TimeSpan.FromMilliseconds(100);
    
    for (int attempt = 1; attempt <= maxAttempts; attempt++)
    {
        try { return await action(); }
        catch (TransientException) when (attempt < maxAttempts)
        {
            var delay = baseDelay.Value * Math.Pow(2, attempt - 1);
            await Task.Delay(delay);
        }
    }
    throw new InvalidOperationException("Should not reach here");
}

// Use
var data = await RetryAsync(() => client.GetStringAsync(url));
```

Better: **Polly** library для production retry / circuit breaker / bulkhead.

### 5.7. Timeout

```csharp
public async Task<string> GetWithTimeoutAsync(string url, TimeSpan timeout)
{
    using var cts = new CancellationTokenSource(timeout);
    try { return await client.GetStringAsync(url, cts.Token); }
    catch (OperationCanceledException) when (cts.IsCancellationRequested)
    {
        throw new TimeoutException($"Timed out after {timeout}");
    }
}

// Or via WaitAsync (.NET 6+)
public async Task<string> GetAsync(string url)
{
    return await client.GetStringAsync(url).WaitAsync(TimeSpan.FromSeconds(30));
}
```

`Task.WaitAsync` (.NET 6+) — clean timeout. Throws `TimeoutException`.

> [!question]- Интервью: почему `.Result` опасно?
> 1) **Deadlock в SyncContext environments** (WPF/WinUI/MAUI/legacy ASP.NET): UI thread blocks waiting на Task. Task continuation needs UI thread (SyncContext). Неfunable. ASP.NET Core OK (null SyncContext) but still: 2) **Thread pool starvation** — blocked thread = wasted resource. Many `.Result` calls drain pool. 3) **Exceptions wrapped в AggregateException** — must unwrap. 4) **Synchronous blocking** — defeats async benefits. **Best practice**: never `.Result` / `.Wait()` в production. Make caller async (`async Task` instead of `string`). Если absolutely necessary — `.GetAwaiter().GetResult()` (unwraps AggregateException). Common case: `Main` method — use `async Task Main` (C# 7.1+). Console app сценарии — `.GetAwaiter().GetResult()` acceptable.

---

## 6. Synchronization primitives

### 6.1. lock keyword

```csharp
private readonly object _lock = new();
private int _counter;

public void Increment()
{
    lock (_lock)
    {
        _counter++;
    }
}

// Compiler transforms:
// Monitor.Enter(_lock);
// try { _counter++; }
// finally { Monitor.Exit(_lock); }
```

`lock` — Monitor-based, **synchronous**. Cannot await inside!

### 6.2. lock в async — anti-pattern

```csharp
// ❌ Cannot await inside lock
public async Task BadAsync()
{
    lock (_lock)
    {
        await SomeAsync();   // ❌ Compile error — await inside lock
    }
}

// Alternative: SemaphoreSlim
private readonly SemaphoreSlim _semaphore = new(1, 1);

public async Task GoodAsync()
{
    await _semaphore.WaitAsync();
    try { await SomeAsync(); }
    finally { _semaphore.Release(); }
}
```

### 6.3. SemaphoreSlim — async-friendly

```csharp
// As mutex (count = 1)
private readonly SemaphoreSlim _mutex = new(1, 1);

public async Task ProtectedAsync()
{
    await _mutex.WaitAsync();
    try { /* critical section */ }
    finally { _mutex.Release(); }
}

// As throttle (count = N)
private readonly SemaphoreSlim _throttle = new(10, 10);

public async Task LimitedAsync()
{
    await _throttle.WaitAsync();
    try { await DoWorkAsync(); }
    finally { _throttle.Release(); }
}
```

### 6.4. Mutex — cross-process

```csharp
// Cross-process synchronization
using var mutex = new Mutex(false, "Global\\MyAppMutex");

if (!mutex.WaitOne(TimeSpan.FromSeconds(5)))
{
    Console.WriteLine("Another instance running");
    return;
}

try { /* exclusive work */ }
finally { mutex.ReleaseMutex(); }
```

`Mutex` — OS-level, slow (~1μs vs lock ~10ns). Use only для cross-process.

### 6.5. ReaderWriterLockSlim

```csharp
private readonly ReaderWriterLockSlim _lock = new();
private readonly Dictionary<int, string> _cache = new();

public string? Read(int key)
{
    _lock.EnterReadLock();
    try { return _cache.GetValueOrDefault(key); }
    finally { _lock.ExitReadLock(); }
}

public void Write(int key, string value)
{
    _lock.EnterWriteLock();
    try { _cache[key] = value; }
    finally { _lock.ExitWriteLock(); }
}
```

Многие readers concurrent / single writer exclusive. Use case: read-heavy cache.

### 6.6. Interlocked

```csharp
private int _counter = 0;

// Atomic increment
Interlocked.Increment(ref _counter);
Interlocked.Decrement(ref _counter);
Interlocked.Add(ref _counter, 5);

// Atomic read/write 64-bit
Interlocked.Exchange(ref _value, newValue);

// Compare and swap
int original = Interlocked.CompareExchange(ref _value, newValue, expected);
```

**Lock-free** atomic operations. Faster than `lock` для simple counters.

### 6.7. ConcurrentDictionary / ConcurrentBag / ConcurrentQueue

```csharp
// Thread-safe collections
var dict = new ConcurrentDictionary<int, string>();

// AddOrUpdate
dict.AddOrUpdate(1, "A", (key, oldValue) => "B");

// GetOrAdd
var value = dict.GetOrAdd(1, key => $"Value-{key}");

// TryAdd, TryRemove, TryUpdate
dict.TryAdd(2, "X");
dict.TryRemove(1, out var removed);

// Concurrent queue (FIFO)
var queue = new ConcurrentQueue<int>();
queue.Enqueue(42);
queue.TryDequeue(out var item);

// Concurrent bag (unordered)
var bag = new ConcurrentBag<string>();
bag.Add("hello");
```

Thread-safe alternatives к Dictionary / List / Queue.

### 6.8. ReadyToRun / AsyncLocal

```csharp
// AsyncLocal — context flows через async calls
private static readonly AsyncLocal<string?> _user = new();

public async Task DoWork()
{
    _user.Value = "Alice";
    await DoSubWork();   // _user.Value still "Alice" inside!
}

public async Task DoSubWork()
{
    Console.WriteLine($"Working as {_user.Value}");   // "Alice"
}
```

`AsyncLocal<T>` — like ThreadLocal но flows через `await`. Used для request context (correlation IDs, current user).

> [!question]- Интервью: чем `lock` отличается от `SemaphoreSlim`?
> **`lock`** — Monitor-based, **synchronous only**. Cannot await inside. Fast (~10ns). Single-threaded entry. **`SemaphoreSlim`** — async-aware (`WaitAsync()` releases thread). Counter-based (allow N concurrent). Slower (~50-100ns) but works в async. **`SemaphoreSlim(1, 1)`** — async-friendly mutex (initial 1, max 1). **`SemaphoreSlim(N, N)`** — throttle для N concurrent operations. **Use cases lock**: synchronous critical sections (counters, dictionaries). **Use cases SemaphoreSlim**: async critical sections, throttling concurrent operations (HTTP requests, DB connections). **Cannot mix**: `lock(obj) { await ...; }` compile error. **Best practice 2024+**: lock для sync hot paths, SemaphoreSlim для async / throttling, `Interlocked` для simple atomic operations.

---

## 7. Deadlocks и race conditions

### 7.1. Classic deadlock

```csharp
// ❌ Two threads acquire locks в opposite order
object lockA = new(), lockB = new();

// Thread 1
lock (lockA) { lock (lockB) { /* ... */ } }

// Thread 2
lock (lockB) { lock (lockA) { /* ... */ } }

// Possible: T1 holds lockA, waits for lockB.
//           T2 holds lockB, waits for lockA.
// → Deadlock
```

**Fix:** consistent lock ordering (always A before B).

### 7.2. async/sync mix deadlock

```csharp
// ❌ WPF deadlock
public string GetData() => GetDataAsync().Result;

public async Task<string> GetDataAsync()
{
    var response = await client.GetStringAsync(url);
    return response;
}

// In WPF: UI thread calls GetData
// GetData blocks UI thread on .Result
// GetDataAsync await needs to return UI thread (SyncContext)
// UI thread blocked → never returns → deadlock
```

**Fix:** make caller async, use `await`.

### 7.3. Cross-thread access (UI)

```csharp
// ❌ Background thread updates UI
await Task.Run(() =>
{
    Items.Add(newItem);   // crash в WPF/WinUI
});

// ✅ Marshal to UI
await Task.Run(() =>
{
    var data = LoadData();
    Application.Current.Dispatcher.Invoke(() => Items.Add(data));
});
```

См. [[desktop-frameworks]] раздел 12.

### 7.4. Race condition on shared state

```csharp
// ❌ Race
private List<int> _items = new();

public void Add(int item) => _items.Add(item);   // concurrent add corrupts List<T>!

// ✅ Lock
private readonly object _lock = new();
public void Add(int item)
{
    lock (_lock) { _items.Add(item); }
}

// ✅ ConcurrentBag / ConcurrentQueue
private readonly ConcurrentBag<int> _items = new();
public void Add(int item) => _items.Add(item);
```

### 7.5. Read-modify-write races

```csharp
// ❌ Race
public void Increment() => _counter++;   // Read-Modify-Write — not atomic!

// ✅ Interlocked
public void Increment() => Interlocked.Increment(ref _counter);

// ✅ Lock
private readonly object _lock = new();
public void Increment() { lock (_lock) { _counter++; } }
```

### 7.6. Detecting deadlocks

```
Tools:
- Visual Studio Debugger — Parallel Tasks window, Threads window
- dotnet-stack — print all threads' stacks
- ANTS Performance Profiler
- ETW traces (deadlock events)

Symptoms:
- Process hangs without progress
- CPU near 0% (threads blocked, not spinning)
- Same threads stuck on lock acquire stacks

Prevention:
- Don't .Result / .Wait async methods
- Don't lock в async methods (use SemaphoreSlim)
- Avoid nested locks
- If nested necessary — consistent ordering
- Use timeouts (Monitor.TryEnter с timeout)
```

> [!question]- Интервью: как избежать deadlock в async коде?
> 1) **Don't `.Result` / `.Wait()`** async methods — primary deadlock source. Make caller async (`async Task` instead of sync return). 2) **Don't `lock` в async методах** — `lock(obj) { await ...; }` не compile (good). Use `SemaphoreSlim.WaitAsync()` instead. 3) **Avoid nested locks** — minimize. 4) **Consistent lock ordering** if nested necessary (always A before B). 5) **`ConfigureAwait(false)`** в libraries — не marshal to original SyncContext. 6) **Timeouts on lock acquire** — `Monitor.TryEnter(obj, timeout)`. 7) **Use immutable data** где possible — eliminates lock need. **Production**: ASP.NET Core (no SyncContext) reduces deadlock risk vs WPF/WinUI. Still: never sync-over-async.

---

## 8. Parallelism

### 8.1. Parallel.For / Parallel.ForEach

```csharp
// CPU-bound parallelism
Parallel.For(0, 1_000_000, i =>
{
    var result = HeavyComputation(i);
    // process
});

Parallel.ForEach(items, item =>
{
    Process(item);
});

// With options
Parallel.ForEach(items,
    new ParallelOptions { MaxDegreeOfParallelism = 4 },
    item => Process(item));
```

`Parallel` distributes work across cores. Internal partitioning, ThreadPool execution.

### 8.2. Parallel.ForEachAsync (.NET 6+)

```csharp
// Async-aware parallelism
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (url, ct) =>
    {
        var data = await client.GetStringAsync(url, ct);
        await ProcessAsync(data);
    });
```

Modern equivalent для async work с throttling. Replaces SemaphoreSlim pattern для simple cases.

### 8.3. PLINQ

```csharp
// Parallel LINQ
var result = numbers
    .AsParallel()
    .Where(x => IsPrime(x))
    .Select(x => x * 2)
    .ToList();

// Specify parallelism
var result2 = numbers
    .AsParallel()
    .WithDegreeOfParallelism(4)
    .WithCancellation(token)
    .Where(x => IsPrime(x))
    .ToArray();

// Preserve order (slower)
var ordered = numbers.AsParallel().AsOrdered().Where(x => x > 10).ToList();
```

PLINQ — parallel LINQ. Auto-partitions work. Best для CPU-bound transformations.

### 8.4. When parallel НЕ faster

```csharp
// ❌ Overhead > benefit
var result = items.Take(100).AsParallel().Select(x => x * 2).ToList();
// Tiny work, parallel setup overhead > computation

// ❌ Contention on shared state
var shared = new List<int>();
Parallel.ForEach(items, x => shared.Add(x));   // Lock contention или race

// ❌ Memory bound
Parallel.ForEach(hugeArrays, arr => Sum(arr));   // limited by memory bandwidth
```

**Profile**: ensure parallel actually faster. Tools: BenchmarkDotNet.

### 8.5. Partitioner

```csharp
// Custom partitioning
var partitioner = Partitioner.Create(0, 1_000_000, 1000);   // chunks of 1000

Parallel.ForEach(partitioner, range =>
{
    for (int i = range.Item1; i < range.Item2; i++)
    {
        // process
    }
});
```

For tight loops — chunk-based partitioning reduces overhead.

### 8.6. Thread-local state

```csharp
// Per-thread accumulator
int totalSum = 0;
Parallel.ForEach(items,
    () => 0,                                              // thread-local init
    (item, state, threadLocalSum) => threadLocalSum + item,   // body
    finalSum => Interlocked.Add(ref totalSum, finalSum));     // merge

Console.WriteLine(totalSum);
```

Avoids contention by giving each thread own accumulator, merging at end.

### 8.7. Choosing parallelism

```
Parallel.ForEach:
- CPU-bound, sync work
- Many items (>1000)
- Items independent

Parallel.ForEachAsync (.NET 6+):
- Async work с throttling
- HTTP requests, async I/O parallelism

PLINQ:
- LINQ-style transformation
- CPU-bound queries

Manual Task.Run + WhenAll:
- Custom logic
- Few specific tasks
- Custom error handling

Don't use parallel:
- I/O bound (use async/await)
- Few items
- Heavy contention on shared state
```

> [!question]- Интервью: чем Parallel.ForEach отличается от async/await?
> **`Parallel.ForEach`** — **CPU-bound parallelism**. Splits work across cores using ThreadPool. Synchronous wait для completion. **Same time** as if you did all work sync, just **across multiple cores**. Best для: image processing, calculations, data transformations. **`async/await`** — **logical concurrency** для I/O-bound. One thread может handle many awaiting tasks (releases during I/O wait). No new cores used unless await switches threads. Best для: network calls, DB queries, file I/O. **Combined** в `Parallel.ForEachAsync` (.NET 6+) — async work in parallel с throttling. **Wrong choice**: `Parallel.ForEach(urls, async url => await client.GetAsync(url))` — blocks pool threads (sync over async). Use `Parallel.ForEachAsync` или `Task.WhenAll(urls.Select(...))`.

---

## 9. Channels и IAsyncEnumerable

### 9.1. `Channel<T>` — producer/consumer

```csharp
using System.Threading.Channels;

// Bounded channel — backpressure
var channel = Channel.CreateBounded<int>(capacity: 100);

// Producer
async Task ProduceAsync()
{
    for (int i = 0; i < 1000; i++)
    {
        await channel.Writer.WriteAsync(i);   // backpressure если full
    }
    channel.Writer.Complete();
}

// Consumer
async Task ConsumeAsync()
{
    await foreach (var item in channel.Reader.ReadAllAsync())
    {
        await ProcessAsync(item);
    }
}

// Run in parallel
await Task.WhenAll(ProduceAsync(), ConsumeAsync());
```

`Channel<T>` — async-friendly producer/consumer queue. .NET 3.0+.

### 9.2. Bounded vs Unbounded

```csharp
// Unbounded — no backpressure (memory grows)
var unbounded = Channel.CreateUnbounded<int>();

// Bounded с different overflow strategies
var bounded = Channel.CreateBounded<int>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait,           // default — backpressure
    // FullMode = BoundedChannelFullMode.DropOldest,
    // FullMode = BoundedChannelFullMode.DropNewest,
    // FullMode = BoundedChannelFullMode.DropWrite,
});
```

### 9.3. `IAsyncEnumerable<T>`

```csharp
public async IAsyncEnumerable<int> StreamAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 100; i++)
    {
        await Task.Delay(100, ct);
        yield return i;
    }
}

// Consume
await foreach (var item in StreamAsync().WithCancellation(cts.Token))
{
    Console.WriteLine(item);
}

// LINQ для async enumerables (System.Linq.Async)
var result = await StreamAsync()
    .WhereAwait(async x => await SomeCheckAsync(x))
    .Select(x => x * 2)
    .Take(10)
    .ToListAsync();
```

`IAsyncEnumerable<T>` (C# 8+) — async streams. Lazy evaluation для async sequences.

### 9.4. Use cases

```
Channel<T>:
- Decoupled producer/consumer
- Multiple producers + multiple consumers
- Backpressure control
- Thread-safe queue

IAsyncEnumerable<T>:
- Async iteration (DB query results streaming)
- gRPC streaming
- File line-by-line async processing
- Real-time data feeds

Comparison:
- IAsyncEnumerable — pull-based (consumer drives)
- Channel — push-based (producer drives)
```

### 9.5. ASP.NET Core Server-Sent Events

```csharp
[HttpGet("stream")]
public async IAsyncEnumerable<Event> StreamEvents(
    [EnumeratorCancellation] CancellationToken ct)
{
    while (!ct.IsCancellationRequested)
    {
        yield return await GetNextEventAsync(ct);
    }
}
```

Returns `IAsyncEnumerable` directly — ASP.NET Core streams to client.

### 9.6. EF Core async streaming

```csharp
// EF Core 6+ supports IAsyncEnumerable
await foreach (var user in dbContext.Users.AsAsyncEnumerable())
{
    await ProcessAsync(user);
}
```

> [!question]- Интервью: что такое `Channel<T>` и когда использовать?
> Async-friendly **producer/consumer queue** (.NET 3.0+). **`Channel.CreateBounded<T>(capacity)`** — backpressure when full (Wait/DropOldest/DropNewest options). **`Channel.CreateUnbounded<T>()`** — no limit (memory grows). **API**: `channel.Writer.WriteAsync(item)`, `channel.Reader.ReadAllAsync()` returns IAsyncEnumerable. **Use cases**: 1) **Decoupled producer/consumer** — log processing, event streams. 2) **Throttling** — bounded backpressure. 3) **Multiple producers/consumers** — thread-safe. **vs `IAsyncEnumerable`**: Channel push-based (producer drives), IAsyncEnumerable pull-based (consumer drives). **vs BlockingCollection**: Channel async-aware, BlockingCollection synchronous. **Best practice 2024+**: `Channel<T>` для high-throughput producer/consumer. IAsyncEnumerable для streaming results.

---

## 10. Best practices

### 10.1. async/await

- ✅ **`async Task` (not `async void`)** — except event handlers.
- ✅ **`ConfigureAwait(false)`** в libraries.
- ✅ **`CancellationToken`** parameter в every async method.
- ✅ **`ValueTask<T>`** для hot paths often-sync.
- ❌ **`.Result` / `.Wait()`** — deadlock + thread pool starvation.
- ❌ **`async void`** для anything except event handlers.
- ❌ **`lock` в async** method.
- ❌ **`Thread.Sleep`** в async (use `Task.Delay`).

### 10.2. Threading

- ✅ **`Task.Run`** для CPU-bound work.
- ✅ **`Parallel.ForEach`** для embarrassingly parallel CPU work.
- ✅ **`Parallel.ForEachAsync`** (.NET 6+) для async parallel.
- ✅ **`Interlocked`** для atomic operations on primitives.
- ❌ **Manual `Thread`** — prefer Task abstraction.
- ❌ **`Thread.Abort`** — deprecated, dangerous.

### 10.3. Synchronization

- ✅ **`lock`** для sync critical sections (fast).
- ✅ **`SemaphoreSlim(1, 1)`** для async mutex.
- ✅ **`SemaphoreSlim(N, N)`** для throttling.
- ✅ **`ConcurrentDictionary`** для shared dictionaries.
- ✅ **`ReaderWriterLockSlim`** для read-heavy scenarios.
- ❌ **`Mutex`** unless cross-process needed (slow).
- ❌ **Manual synchronization** when concurrent collections work.

### 10.4. Cancellation

- ✅ **Pass `CancellationToken`** through async chain.
- ✅ **Periodic `ThrowIfCancellationRequested`** в long loops.
- ✅ **Linked tokens** для combining sources.
- ✅ **Auto-cancel** через `cts.CancelAfter(timeout)`.
- ❌ **Ignore CancellationToken** в long-running async work.

### 10.5. Не делай

- ❌ Block UI thread (`.Result`, sync I/O).
- ❌ Drain thread pool (sync over async).
- ❌ Update UI from non-UI thread.
- ❌ Mix lock and await.
- ❌ Forget exception handling в Task.Run / fire-and-forget.

---

## 11. Decision tree

```
Что выбрать для concurrency?
│
├── I/O bound (network, disk, DB)
│   ├── async/await + Task<T>
│   ├── Throttle parallel I/O → SemaphoreSlim или Parallel.ForEachAsync
│   ├── Cancel → CancellationToken
│   └── Timeout → cts.CancelAfter или Task.WaitAsync
│
├── CPU bound (calculations)
│   ├── Many items → Parallel.ForEach или PLINQ
│   ├── Single computation → Task.Run (offload)
│   ├── Complex partitioning → Partitioner.Create
│   └── Async wrapper → async + Task.Run inside
│
├── Long-running background
│   ├── ASP.NET Core / Worker Service → BackgroundService
│   ├── Custom → Task.Run + CancellationToken
│   └── Periodic → Timer или PeriodicTimer (.NET 6+)
│
├── Producer/consumer
│   ├── Async → Channel<T>
│   ├── Sync → BlockingCollection<T>
│   └── Streaming results → IAsyncEnumerable<T>
│
├── Synchronization
│   ├── Sync critical section → lock
│   ├── Async critical section → SemaphoreSlim(1, 1)
│   ├── Throttle N concurrent → SemaphoreSlim(N, N)
│   ├── Read-heavy cache → ReaderWriterLockSlim
│   ├── Atomic counter → Interlocked
│   ├── Shared dictionary → ConcurrentDictionary
│   └── Cross-process → Mutex
│
└── UI updates
    ├── WPF → Application.Current.Dispatcher.Invoke
    ├── WinUI → DispatcherQueue.TryEnqueue
    ├── MAUI → MainThread.BeginInvokeOnMainThread
    └── Avalonia → Dispatcher.UIThread.InvokeAsync
```

---

## 12. Cheat sheet

```csharp
// === async/await ===
public async Task<string> FetchAsync(string url, CancellationToken ct)
{
    using var response = await client.GetAsync(url, ct);
    return await response.Content.ReadAsStringAsync(ct);
}

// === Task combinators ===
var results = await Task.WhenAll(t1, t2, t3);
var winner = await Task.WhenAny(t1, t2, t3);

// === Cancellation ===
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
try { await DoAsync(cts.Token); }
catch (OperationCanceledException) { /* timed out */ }

// === Parallel ===
Parallel.For(0, 100, i => Compute(i));
Parallel.ForEach(items, item => Process(item));
await Parallel.ForEachAsync(urls, async (url, ct) =>
{
    var data = await client.GetStringAsync(url, ct);
});

// === PLINQ ===
var result = numbers.AsParallel()
    .Where(x => x > 0)
    .Select(x => x * 2)
    .ToList();

// === Synchronization ===
private readonly object _lock = new();
lock (_lock) { /* sync critical */ }

private readonly SemaphoreSlim _sem = new(1, 1);
await _sem.WaitAsync();
try { /* async critical */ } finally { _sem.Release(); }

Interlocked.Increment(ref _counter);
Interlocked.CompareExchange(ref _value, newValue, expected);

// === Concurrent collections ===
var dict = new ConcurrentDictionary<int, string>();
dict.AddOrUpdate(1, "A", (k, old) => "B");
var v = dict.GetOrAdd(1, k => $"Value-{k}");

var queue = new ConcurrentQueue<int>();
queue.Enqueue(42);
queue.TryDequeue(out var item);

// === Channel<T> ===
var channel = Channel.CreateBounded<int>(100);

// Producer
await channel.Writer.WriteAsync(42);
channel.Writer.Complete();

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync())
{
    await ProcessAsync(item);
}

// === IAsyncEnumerable ===
public async IAsyncEnumerable<int> StreamAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 100; i++)
    {
        await Task.Delay(100, ct);
        yield return i;
    }
}

await foreach (var item in StreamAsync().WithCancellation(token))
{
    Console.WriteLine(item);
}

// === BackgroundService (ASP.NET Core) ===
public class MyWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await DoWorkAsync(stoppingToken);
            await Task.Delay(TimeSpan.FromMinutes(1), stoppingToken);
        }
    }
}

// === Throttle parallel I/O ===
var sem = new SemaphoreSlim(10);
var tasks = urls.Select(async url =>
{
    await sem.WaitAsync();
    try { return await client.GetStringAsync(url); }
    finally { sem.Release(); }
});
var results = await Task.WhenAll(tasks);
```

---

## 13. Common pitfalls

### 13.1. .Result deadlock

```csharp
// ❌ WPF deadlock
public string Get() => GetAsync().Result;
```

**Fix:** make caller async.

### 13.2. async void exceptions

```csharp
// ❌ Crashes app
public async void Bad() { throw new Exception(); }
```

**Fix:** `async Task` + try/catch.

### 13.3. Forgot await

```csharp
// ❌ Fire-and-forget по accident
public async Task DoAsync()
{
    SendNotificationAsync();   // not awaited!
}
```

**Fix:** always `await` or explicit `_ = ...` для intentional fire-and-forget.

### 13.4. Lock в async

```csharp
public async Task BadAsync()
{
    lock (_lock)
    {
        await SomeAsync();   // ❌ Compile error
    }
}
```

**Fix:** SemaphoreSlim.

### 13.5. CancellationToken не пробрасывается

```csharp
public async Task DoAsync(CancellationToken ct)
{
    var data = await client.GetStringAsync(url);   // ❌ ct ignored
}
```

**Fix:** pass `ct` to inner async APIs.

### 13.6. Race condition

```csharp
// ❌
private int _counter;
public void Increment() => _counter++;
```

**Fix:** `Interlocked.Increment(ref _counter)` или lock.

### 13.7. Update UI from background

```csharp
await Task.Run(() => label.Text = "Done");   // ❌ crash WPF
```

**Fix:** Dispatcher.Invoke.

### 13.8. ValueTask awaited twice

```csharp
ValueTask<int> task = GetAsync();
await task;
await task;   // ❌ undefined behavior
```

**Fix:** await once, or convert to Task: `var t = task.AsTask();`.

### 13.9. Parallel.ForEach с async

```csharp
// ❌ Sync over async — pool starvation
Parallel.ForEach(urls, async url =>
{
    await client.GetAsync(url);
});
```

**Fix:** `Parallel.ForEachAsync` (.NET 6+) или `Task.WhenAll`.

### 13.10. ThreadPool starvation

```csharp
// ❌ Blocks pool threads
public async Task<string> Get() => await Task.Run(() =>
{
    return SyncHttpCall().Result;   // blocks pool worker!
});
```

**Fix:** await async APIs directly. Don't wrap sync-blocking в Task.Run.

> [!question]- Интервью: топ-3 ошибки в async коде?
> 1) **`.Result` / `.Wait()`** — deadlock в WPF/WinUI/legacy ASP.NET, thread pool starvation в ASP.NET Core. Always `await`. Make caller async. 2) **`async void`** — exceptions unhandled, crash app. Use `async Task` except event handlers. 3) **CancellationToken не пробрасывается** — outer cancellation doesn't reach inner async APIs. Pass through every async method, last parameter `CancellationToken ct = default`. **Bonus**: `lock (obj) { await ... }` — compile error (good!), use `SemaphoreSlim.WaitAsync()`. **Bonus 2**: forgot `await` — fire-and-forget by accident. Use `_ = ...` for explicit fire-and-forget с error handling.

---

## 14. Practice exercises

### 14.1. Producer/consumer с `Channel<T>`

```csharp
public class WorkProcessor
{
    private readonly Channel<WorkItem> _channel;
    
    public WorkProcessor(int capacity = 1000)
    {
        _channel = Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(capacity)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false,
            SingleWriter = false
        });
    }
    
    public async ValueTask EnqueueAsync(WorkItem item, CancellationToken ct = default) =>
        await _channel.Writer.WriteAsync(item, ct);
    
    public async Task ProcessAsync(int workerCount, CancellationToken ct = default)
    {
        var workers = Enumerable.Range(0, workerCount)
            .Select(_ => Task.Run(async () =>
            {
                await foreach (var item in _channel.Reader.ReadAllAsync(ct))
                {
                    try { await ProcessItemAsync(item); }
                    catch (Exception ex) { /* log */ }
                }
            }, ct))
            .ToList();
        
        await Task.WhenAll(workers);
    }
    
    public void Complete() => _channel.Writer.Complete();
    
    private async Task ProcessItemAsync(WorkItem item) => await Task.Delay(100);
}

public record WorkItem(int Id, string Data);
```

### 14.2. Throttled HTTP client

```csharp
public class ThrottledHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly SemaphoreSlim _throttle;
    
    public ThrottledHttpClient(HttpClient client, int maxConcurrency)
    {
        _httpClient = client;
        _throttle = new SemaphoreSlim(maxConcurrency, maxConcurrency);
    }
    
    public async Task<string> GetAsync(string url, CancellationToken ct = default)
    {
        await _throttle.WaitAsync(ct);
        try
        {
            return await _httpClient.GetStringAsync(url, ct);
        }
        finally
        {
            _throttle.Release();
        }
    }
    
    public async Task<List<string>> GetManyAsync(
        IEnumerable<string> urls,
        CancellationToken ct = default)
    {
        var tasks = urls.Select(url => GetAsync(url, ct));
        var results = await Task.WhenAll(tasks);
        return results.ToList();
    }
}

// Usage
var http = new HttpClient();
var throttled = new ThrottledHttpClient(http, maxConcurrency: 10);
var results = await throttled.GetManyAsync(thousandsOfUrls);
// Only 10 concurrent requests
```

### 14.3. Retry с exponential backoff + jitter

```csharp
public static class Retry
{
    public static async Task<T> WithBackoffAsync<T>(
        Func<CancellationToken, Task<T>> action,
        int maxAttempts = 3,
        TimeSpan? baseDelay = null,
        Func<Exception, bool>? shouldRetry = null,
        CancellationToken ct = default)
    {
        baseDelay ??= TimeSpan.FromMilliseconds(100);
        shouldRetry ??= ex => ex is HttpRequestException or TimeoutException;
        var random = new Random();
        
        for (int attempt = 1; attempt <= maxAttempts; attempt++)
        {
            try
            {
                return await action(ct);
            }
            catch (Exception ex) when (attempt < maxAttempts && shouldRetry(ex))
            {
                var jitter = random.NextDouble() * 0.3;   // 0-30% jitter
                var delay = baseDelay.Value * Math.Pow(2, attempt - 1) * (1 + jitter);
                await Task.Delay(delay, ct);
            }
        }
        
        throw new InvalidOperationException("Should not reach here");
    }
}

// Usage
var result = await Retry.WithBackoffAsync(
    async ct => await client.GetStringAsync(url, ct),
    maxAttempts: 5,
    baseDelay: TimeSpan.FromMilliseconds(200),
    shouldRetry: ex => ex is HttpRequestException);
```

---

## 15. Что читать дальше

1. **[[modern-features|Modern Features]]** — async streams, await using.
2. **[[csharp-language-design|Language Design]]** — async/await innovation.
3. **Stephen Cleary — async/await series** — blog.stephencleary.com.
4. **Stephen Toub — Performance** — devblogs.microsoft.com.
5. **Книга — "Concurrency in C# Cookbook" by Stephen Cleary**.
6. **Polly library** — github.com/App-vNext/Polly (retry/circuit breaker).

---

## 16. См. также

- [[modern-features|Modern Features]] — async streams
- [[memory-pooling|Memory Pooling]] — `Memory<T>` в async
- [[csharp-language-design|Language Design]] — async/await impact
- [[error-handling|Error Handling]] — exceptions в async
- System.Threading namespace
- System.Threading.Channels
- System.Threading.Tasks

---

## 17. Reading list

- **Microsoft Docs — async/await** — learn.microsoft.com/dotnet/csharp/asynchronous-programming
- **Microsoft Docs — Threading** — learn.microsoft.com/dotnet/standard/threading/
- **Microsoft Docs — Parallel** — learn.microsoft.com/dotnet/standard/parallel-programming/
- **Microsoft Docs — Channels** — learn.microsoft.com/dotnet/core/extensions/channels
- **Stephen Cleary — async/await series** — blog.stephencleary.com
- **Stephen Cleary — "Concurrency in C# Cookbook"** (book)
- **Stephen Toub — Performance series** — devblogs.microsoft.com
- **Joe Duffy — "Concurrent Programming on Windows"** — book (deeper)
- **Polly** — github.com/App-vNext/Polly
- **Reactive Extensions (Rx.NET)** — github.com/dotnet/reactive
- **System.IO.Pipelines** — learn.microsoft.com/dotnet/standard/io/pipelines
