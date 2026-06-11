---
tags: [performance, async-patterns, parallelism]
level: Middle to Senior
date: 2026-04-30
---

# Async Performance Patterns

> **Async-specific performance: ValueTask, TaskCompletionSource, async pitfalls, ConfigureAwait**. Когда async помогает, когда мешает.

---

## Что это, зачем и когда

### Async — не silver bullet

Async часто **тратит ресурсы** если использован неправильно. Помогает только для I/O-bound, а не CPU-bound.

```csharp
// ❌ async для CPU work — overhead без выгоды
public async Task<int> CalculateAsync(int x)
{
    return await Task.FromResult(x * x);  // overhead async/await + Task allocation
}

// ✅
public int Calculate(int x) => x * x;
```

См.[[async-threading|Async Threading]].

---

## 1. Task vs ValueTask

```csharp
// Task — always allocates (reference type)
public async Task<int> GetAsync(int id)
{
    return _cache.TryGetValue(id, out var v) ? v : await LoadAsync(id);
}

// ValueTask — struct, no allocation если результат sync
public async ValueTask<int> GetAsync(int id)
{
    return _cache.TryGetValue(id, out var v) ? v : await LoadAsync(id);
}
```

**Когда ValueTask:**
- Often returns synchronously (cache hits)
- Hot path
- Single await per call

**Когда Task:**
- Default — простота важнее
- Result awaited multiple times
- Stored in collection

> [!warning] ValueTask — restrictions
> - Can be awaited only ОДИН раз
> - Cannot be in `Task.WhenAll`
> - Easier to misuse

---

## 2. ConfigureAwait(false)

```csharp
public async Task<string> ReadAsync()
{
    var data = await File.ReadAllTextAsync("file.txt").ConfigureAwait(false);
    return data;
}
```

**Когда:**
- **Library code** — yes, always
- **ASP.NET Core app code** — НЕ нужно (no SynchronizationContext)
- **WPF / WinForms** — yes, везде кроме UI thread

> [!info] ASP.NET Core 6+ — без SyncContext
> ConfigureAwait(false) в app code = no benefit но шум.

---

## 3. Avoiding deadlocks

```csharp
// ❌ DEADLOCK в WPF / WinForms / older ASP.NET
public string GetData()
{
    return GetDataAsync().Result;  // 💀 Deadlock!
}

// ✅ async all the way
public async Task<string> GetDataAsync()
{
    return await GetDataInternalAsync();
}
```

См.[[async-threading|Async Threading]] — async deadlock detail.

---

## 4. Parallel async — правильно

```csharp
// ❌ Sequential await
foreach (var url in urls)
    results.Add(await client.GetStringAsync(url));

// ✅ Concurrent (one-by-one starting)
var tasks = urls.Select(u => client.GetStringAsync(u));
var results = await Task.WhenAll(tasks);

// ✅ Bounded parallelism
var semaphore = new SemaphoreSlim(10);
var tasks = urls.Select(async url =>
{
    await semaphore.WaitAsync();
    try { return await client.GetStringAsync(url); }
    finally { semaphore.Release(); }
});
var results = await Task.WhenAll(tasks);

// ✅✅ Parallel.ForEachAsync (.NET 6+)
await Parallel.ForEachAsync(
    urls,
    new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (url, ct) =>
    {
        var data = await client.GetStringAsync(url, ct);
        ProcessAsync(data);
    });
```

---

## 5. async void — only event handlers

```csharp
// ❌ async void в general code
public async void DoWork()
{
    await Task.Delay(1000);
    throw new Exception();  // 💥 Crashes process!
}

// ✅ async Task
public async Task DoWork()
{
    await Task.Delay(1000);
    throw new Exception();  // caller catches
}

// ✅ OK для event handlers
button.Click += async (s, e) =>
{
    await ProcessAsync();
    // Если throw — не crash, но event handler must catch
};
```

---

## 6. CancellationToken — пропускай везде

```csharp
// ❌ Невозможно отменить
public async Task<List<Order>> GetAsync()
{
    return await _db.Orders.ToListAsync();
}

// ✅ Cancellable
public async Task<List<Order>> GetAsync(CancellationToken ct)
{
    return await _db.Orders.ToListAsync(ct);
}

// In controller:
public async Task<IActionResult> Get(CancellationToken ct)
{
    // ASP.NET cancels when client disconnects
    var orders = await _service.GetAsync(ct);
    return Ok(orders);
}
```

Если client отключился — DB query отменится, ресурсы свободны.

---

## 7. fire-and-forget — danger zone

```csharp
// ❌ Exception потеряется
public void StartBackground()
{
    DoWorkAsync();  // ⚠️ unhandled exception → process crash
}

// ✅ Logged
public void StartBackground()
{
    _ = Task.Run(async () =>
    {
        try { await DoWorkAsync(); }
        catch (Exception ex) { _logger.LogError(ex, "Background work failed"); }
    });
}

// ✅✅ BackgroundService (proper way в ASP.NET)
public class MyBackgroundJob(IServiceProvider sp) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try { await DoWorkAsync(ct); }
            catch (Exception ex) { /* log */ }
            await Task.Delay(1000, ct);
        }
    }
}
```

См.[[hosting-background|Background Services]].

---

## 8. TaskCompletionSource — wrap callback APIs

```csharp
// Convert callback API to async
public Task<int> WaitForResponseAsync()
{
    var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);
    
    legacyApi.OnResponse += response => tcs.TrySetResult(response);
    legacyApi.OnError += error => tcs.TrySetException(new Exception(error));
    legacyApi.OnTimeout += () => tcs.TrySetCanceled();
    
    return tcs.Task;
}
```

**Critical:** `RunContinuationsAsynchronously` — иначе continuation runs sync на TaskCompletionSource thread, может deadlock.

---

## 9. async streaming — IAsyncEnumerable

```csharp
// ❌ Loads всё в memory
public async Task<List<Order>> GetAllAsync()
{
    return await _db.Orders.ToListAsync();
}

// ✅ Streams — process по мере получения
public async IAsyncEnumerable<Order> GetAllAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var order in _db.Orders.AsAsyncEnumerable().WithCancellation(ct))
        yield return order;
}

// Caller:
await foreach (var order in service.GetAllAsync())
{
    if (ShouldStop()) break;
    await ProcessAsync(order);
}
```

Полезно для large datasets, real-time streams.

---

## 10. Common Pitfalls

### 1. Creating Task для CPU work

```csharp
// ❌ Task.Run для async method — double overhead
await Task.Run(async () => await DoSomeAsync());

// ✅
await DoSomeAsync();
```

### 2. Blocking в async

```csharp
public async Task DoAsync()
{
    Thread.Sleep(1000);  // ❌ Blocks thread!
    
    // ✅
    await Task.Delay(1000);
}
```

### 3. Multiple awaits на same Task

```csharp
var task = SomeAsync();
var result1 = await task;
var result2 = await task;  // ⚠️ Returns same value, no second execution
```

### 4. Forgetting await

```csharp
// ❌ Возвращает Task, не результат
public Task<int> GetAsync() => DoAsync();  // OK если DoAsync returns Task<int>

// ❌ Compiler warning
public async Task DoBoth()
{
    Task1();  // ⚠️ Forgot await
    await Task2();
}
```

### 5. Async method without await

```csharp
// ❌ Не нужно async/await
public async Task<int> GetAsync()
{
    return await Task.FromResult(5);  // overhead state machine
}

// ✅
public Task<int> GetAsync() => Task.FromResult(5);
```

---

## 11. Best Practices

- **Async для I/O, sync для CPU**
- **CancellationToken everywhere**
- **ConfigureAwait(false) в libraries**
- **Avoid async void** — only event handlers
- **Task.WhenAll для parallel**
- **Bounded concurrency** через SemaphoreSlim
- **No fire-and-forget** в business logic
- **ValueTask только когда оправдано**
- **Don't block** — `.Result`, `.Wait()`
- **Don't sync over async** — главная причина deadlocks

---

## Case Studies

### Case Study #1 — API endpoint slow на больших данных

**Сценарий:** `GET /reports/sales` возвращает данные за год — p99 latency 30 сек.

**❌ Memory + CPU bottleneck:**
```csharp
public async Task<List<SalesRow>> GetReport()
{
    var orders = await _db.Orders.ToListAsync();  // 1M rows в memory
    return orders
        .Where(o => o.Year == 2026)
        .GroupBy(o => o.ProductId)
        .Select(g => new SalesRow { ProductId = g.Key, Total = g.Sum(o => o.Total) })
        .ToList();
}
```

**✅ DB-level aggregation:**
```csharp
public async Task<List<SalesRow>> GetReport() =>
    await _db.Orders
        .Where(o => o.Year == 2026)
        .GroupBy(o => o.ProductId)
        .Select(g => new SalesRow { ProductId = g.Key, Total = g.Sum(o => o.Total) })
        .ToListAsync();
```

**Result:** 30 sec → 200 ms. SQL делает aggregation, не C#.

---

### Case Study #2 — Hot path allocations

**Сценарий:** Method вызывается 100K RPS. Profiler показывает много GC pauses.

**❌ Allocations:**
```csharp
public bool Validate(string input)
{
    var parts = input.Split(',');  // string[] alloc
    var trimmed = parts.Select(p => p.Trim()).ToList();  // List + iterations alloc
    return trimmed.All(t => !string.IsNullOrEmpty(t));
}
```

**✅ Span-based zero-alloc:**
```csharp
public bool Validate(ReadOnlySpan<char> input)
{
    foreach (var range in input.Split(','))
    {
        var part = input[range].Trim();
        if (part.IsEmpty) return false;
    }
    return true;
}
```

**Result:** 0 allocations, 3x faster, fewer GC cycles.

---

### Case Study #3 — Async overhead в hot path

**Сценарий:** Method чаще завершается синхронно (cache hit). `Task` allocation overhead.

**❌:**
```csharp
public async Task<User> GetAsync(int id)
{
    if (_cache.TryGet(id, out var user)) return user;
    return await _db.GetAsync(id);
}
// Каждый cache hit — Task allocation
```

**✅ ValueTask:**
```csharp
public ValueTask<User> GetAsync(int id)
{
    if (_cache.TryGet(id, out var user)) return new ValueTask<User>(user);
    return new ValueTask<User>(_db.GetAsync(id));
}
// Cache hit — zero alloc
```

См.[[async-threading|async-threading]] и [[memory-pooling|Memory Pooling]].


---

## Cheat sheet

| Symptom | Tool / Approach |
|---------|-----------------|
| High CPU | dotnet-trace, dotTrace sampling |
| High memory | dotnet-dump + WinDbg, dotMemory snapshots |
| GC pauses | dotnet-counters, ETW events |
| Slow query | EF logging + Database query plan |
| Slow API | Application Insights / Datadog APM |
| Memory leak | Snapshot diffs (dotMemory, JetBrains) |
| Async deadlock | dotnet-stack threads dump |
| Lock contention | dotnet-trace + Concurrency Visualizer |
| Allocation hot path | BenchmarkDotNet `[MemoryDiagnoser]` |
| Microoptimization | BenchmarkDotNet, disasm |

| Allocation Cost | Bytes |
|-----------------|-------|
| Reference type (object) | 16-24 bytes header + fields |
| string interning | shared, no new allocation |
| boxing int → object | 24 bytes |
| `new List<T>()` empty | 40 bytes |
| `new List<T>(capacity)` | 40 + (capacity × sizeof(T)) |
| Closure | depends on captured vars |
| `async Task` state machine | ~80-200 bytes per call |
| `ValueTask` (sync complete) | 0 bytes |

| Speed | Tool |
|-------|------|
| Microsec measurements | BenchmarkDotNet |
| Millisec end-to-end | Stopwatch + LogInformation |
| Production tracing | OpenTelemetry + Jaeger |
| Real-time monitoring | dotnet-counters --refresh-interval 1 |


---

## Decision tree

```
Performance issue?
│
├── Сначала — где боль?
│   ├── Latency (p99) → APM tools (App Insights, Datadog)
│   ├── Throughput (RPS limit) → load test + profiler
│   ├── Memory → snapshots (dotMemory, dotnet-dump)
│   └── CPU → sampling profiler (dotTrace, perf)
│
├── Bottleneck identified?
│   ├── Database → query plan, indexes, N+1
│   ├── Network → batching, HTTP/2, connection pooling
│   ├── CPU → algorithmic complexity, allocations
│   ├── Memory → object pooling, struct vs class
│   └── Locks → ConcurrentDictionary, lock-free
│
├── Optimization сложность?
│   ├── Easy wins → caching, async/await, pagination
│   ├── Medium → query optimization, batch processing
│   ├── Hard → memory pooling, Span<T>, source generators
│   └── Extreme → unsafe, SIMD, native AOT, custom allocator
│
└── Проверка?
    ├── Benchmark до/после → BenchmarkDotNet
    ├── Real load test → k6, NBomber, JMeter
    └── Production canary → 5% → 50% → 100%
```

**Optimization rule:** Measure → Hypothesize → Optimize → Measure. Никогда не оптимизируй без data.


---

## См. также

- [[performance-fundamentals|Performance Fundamentals]]
- [[optimization-patterns|Optimization Patterns]]
-[[async-threading|Async и Threading]]
-[[concurrency-atomics|Concurrency Atomics]]
-[[hosting-background|Background Services]]

## Reading list

- **Stephen Cleary blog** — blog.stephencleary.com (async expert)
- **Stephen Toub posts** — devblogs.microsoft.com/dotnet
- **Concurrency in C# Cookbook** — Stephen Cleary (book)
- **Microsoft Docs — Async/await** — learn.microsoft.com/dotnet/csharp/asynchronous-programming
