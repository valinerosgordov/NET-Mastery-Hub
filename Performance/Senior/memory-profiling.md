---
tags: [performance, profiling, memory-leaks, gcdump, dotmemory]
level: Senior
date: 2026-04-30
---

# Memory Profiling — поиск утечек и проблем с памятью

> **Как находить memory leaks, GC pressure, оптимизировать heap usage**. Tools, workflow, decoding heap snapshots.

---

## Что это, зачем и когда

### Когда нужно memory profiling

✅ **Признаки memory problems:**
- Heap size растёт со временем (memory leak)
- Frequent Gen2 GC (slow!)
- High allocation rate (GC pressure)
- OOM (Out of Memory) errors
- Process memory usage > expected
- Slow response time из-за GC pauses

### Workflow

```
1. Confirm problem  — counter says memory growing?
2. Capture snapshots — gcdump baseline + after issue
3. Compare          — что выросло?
4. Find root cause  — кто держит references?
5. Fix              — break the chain
6. Verify           — снимок снова, growth остановлен?
```

---

## 1. dotnet-counters — first signal

```bash
dotnet-counters monitor -p 12345 --counters System.Runtime
```

```
GC Heap Size (MB)               : 320
Gen 0 GC Count (Count / 1 sec)  : 5
Gen 1 GC Count (Count / 1 sec)  : 2
Gen 2 GC Count (Count / 1 sec)  : 1     ← проблема если часто
Allocation Rate (B / 1 sec)     : 50,000,000   ← 50 MB/s!
Working Set (MB)                : 480
```

**Tell-tale signs:**
- Gen2 GC > 1/sec → high pressure
- Allocation rate > 50 MB/s → lots of garbage
- Heap size monotonically растёт → leak

---

## 2. dotnet-gcdump — heap snapshot

```bash
# Capture
dotnet-gcdump collect -p 12345
# Output: 20260430_140000_12345.gcdump
```

Open в **dotMemory** (JetBrains, $) или **Visual Studio Diagnostic Tools**.

### Workflow для leak

```
1. Take baseline snapshot
2. Wait for problem to develop (e.g. run 1000 requests)
3. Take 2nd snapshot
4. Compare:
   - Какие types увеличились в количестве?
   - Кто держит references на эти объекты (GC roots)?
5. Find root cause в коде
```

### Пример: event handler leak

```csharp
public class Service
{
    public event EventHandler? OnUpdate;
}

public class Subscriber
{
    public Subscriber(Service service)
    {
        service.OnUpdate += Handle;  // 🔴 Не отписан!
    }
    
    private void Handle(object? s, EventArgs e) { }
}

// Создаём 1000 subscribers, всё subscribed
for (int i = 0; i < 1000; i++)
    new Subscriber(service);

// Хотя локальные refs ушли — Service держит references на Subscriber через event
// 1000 Subscriber instances лежат в памяти
```

В dotMemory будет видно:
- `Subscriber` instances: 1000
- GC root: `Service.OnUpdate` event

**Лечение:** unsubscribe или WeakReference.

---

## 3. PerfView — Windows powerful tool

[github.com/microsoft/perfview](https://github.com/microsoft/perfview)

Free, от Microsoft. Powerful но steep learning curve.

```
Collect → take heap snapshot → analyze
```

Use cases:
- Allocation profiling (где аллоцируется)
- GC analysis (почему Gen2)
- Reference inspection

См.[[diagnostics-tools|Diagnostics Tools]].

---

## 4. dotMemory (JetBrains)

GUI heap profiler. Кросс-платформенный, $$$ commercial.

Преимущества:
- Easy compare snapshots
- Filter by type / namespace
- Object retention paths
- Generational distribution

Open `gcdump` files **directly** или attach к процессу для live analysis.

---

## 5. Common memory leaks

### Leak 1: Static collections

```csharp
// ❌ Static dict растёт навсегда
public static class Cache
{
    private static Dictionary<int, User> _users = new();
    
    public static User Get(int id)
    {
        if (!_users.ContainsKey(id))
            _users[id] = LoadFromDb(id);
        return _users[id];
    }
}
```

**Лечение:** TTL, MemoryCache с size limit.

### Leak 2: Event handlers

```csharp
// ❌ Subscriber subscribed but не unsubscribed
service.OnUpdate += subscriber.Handle;

// ✅ Unsubscribe в Dispose
public void Dispose()
{
    _service.OnUpdate -= Handle;
}

// ✅✅ Weak event pattern
WeakEventManager<Service, EventArgs>.AddHandler(service, "OnUpdate", subscriber.Handle);
```

### Leak 3: ThreadStatic / AsyncLocal

```csharp
// ❌ ThreadStatic не очищается между requests в ThreadPool
[ThreadStatic]
private static List<Item> _buffer;

public void Process(Item item)
{
    _buffer ??= new List<Item>();
    _buffer.Add(item);  // ⚠️ Когда clear?
}
```

### Leak 4: Closure captures

```csharp
public class Service
{
    private readonly LargeObject _heavy = new();
    
    public Action GetHandler() =>
        () => Console.WriteLine(_heavy.Name);
        // ⚠️ closure захватывает this → весь Service alive пока handler жив
}

// Storing handlers in static
static List<Action> _handlers = new();
_handlers.Add(service.GetHandler());
// Service не GC'd до удаления из _handlers
```

### Leak 5: Disposable not disposed

```csharp
// ❌ FileStream не закрыт
public void Process()
{
    var fs = File.Open("data.txt", FileMode.Open);
    var content = ReadContent(fs);
    // No dispose!
}

// ✅ using
public void Process()
{
    using var fs = File.Open("data.txt", FileMode.Open);
    var content = ReadContent(fs);
}
```

### Leak 6: Singleton holding state

```csharp
// ❌ Singleton accumulates state
public class CacheService
{
    private List<RequestLog> _logs = new();
    
    public void LogRequest(Request req)
    {
        _logs.Add(new RequestLog(req));
        // Никогда не очищается!
    }
}
```

### Leak 7: Captured CancellationToken

```csharp
public async Task LongOperation(CancellationToken parentCt)
{
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(parentCt);
    // ⚠️ Если parentCt никогда не отменяется и this remains — leak
}
```

### Leak 8: Timer без stop

```csharp
// ❌ Timer keeps running, captures handler
public class Service
{
    private Timer _timer;
    
    public Service()
    {
        _timer = new Timer(_ => DoWork(), null, 0, 1000);
        // Forever, никогда не Stop
    }
}
```

---

## 6. Allocation profiling

### Find hot allocation paths

```bash
dotnet-trace collect -p 12345 --providers System.Runtime --duration 30
```

Open в **speedscope.app** или **PerfView**.

Видно top allocation sites — какие методы создают most objects.

### Common allocation hot-spots

```csharp
// ❌ Boxing
int x = 5;
object o = x;  // boxing — heap allocation

List<object> list = new();
list.Add(5);   // boxing
list.Add(6);   // boxing

// ✅ Generic
List<int> list = new();

// ❌ Closure allocation в hot loop
for (int i = 0; i < 1_000_000; i++)
{
    Task.Run(() => Console.WriteLine(i));  // closure captures i!
}

// ✅ Без closure
for (int i = 0; i < 1_000_000; i++)
{
    var local = i;
    Task.Run(() => Console.WriteLine(local));
}

// ❌ String concat в loop
var s = "";
for (int i = 0; i < 1000; i++)
    s += i;  // O(n²) allocations

// ✅ StringBuilder
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++)
    sb.Append(i);
var s = sb.ToString();
```

См.[[gc-memory|GC и память]].

---

## 7. GC pressure analysis

### High Gen0 — too many allocations

```
Symptoms:
- Gen0 GC > 10/sec
- Gen0 collection pauses 5-10 ms each
- High CPU usage в GC

Causes:
- Tons of short-lived objects
- LINQ in hot path с lambdas
- String concatenation
- Boxing
```

**Лечение:**
- Reduce allocations (Span, ArrayPool, struct)
- Reuse objects (ObjectPool)

### Frequent Gen2 — survivors aging

```
Symptoms:
- Gen2 GC > 0.5/sec
- 50-200 ms pauses each Gen2
- Heap size growing

Causes:
- Mid-lived objects (выживают Gen0+Gen1, попадают в Gen2)
- Cache holding references
- Static collections growing
- LOH allocations
```

**Лечение:**
- Server GC (если ещё нет)
- Investigate cache strategy
- Find leaks (gcdump)
- Use POH (Pinned Object Heap, .NET 5+) для long-lived buffers

### LOH (Large Object Heap) issues

Объекты > 85,000 bytes идут на LOH. LOH compaction редкий → fragmentation.

```csharp
// ❌ Frequent large allocations
public byte[] Process(int size)
{
    return new byte[100_000];  // → LOH каждый вызов
}

// ✅ ArrayPool
public void Process(int size)
{
    var buffer = ArrayPool<byte>.Shared.Rent(100_000);
    try { /* use */ }
    finally { ArrayPool<byte>.Shared.Return(buffer); }
}
```

---

## 8. Production memory monitoring

### Continuous profiling

**Pyroscope** / **Datadog Continuous Profiler** / **Grafana Phlare** — постоянно собирают profile в production с минимальным overhead.

```yaml
# Pyroscope agent
- name: PYROSCOPE_SERVER_ADDRESS
  value: "http://pyroscope:4040"
- name: PYROSCOPE_APPLICATION_NAME
  value: "myapp"
```

Видишь flame graph allocations за любой период.

### Health checks

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("memory", () =>
    {
        var allocated = GC.GetTotalMemory(forceFullCollection: false);
        if (allocated > 500_000_000)  // 500 MB
            return HealthCheckResult.Degraded($"High memory: {allocated / 1_000_000} MB");
        return HealthCheckResult.Healthy();
    });
```

### Alerts

Prometheus rule:
```yaml
- alert: HighMemoryUsage
  expr: dotnet_total_memory_bytes > 1e9
  for: 10m
  annotations:
    summary: "App memory > 1 GB for 10 min"
```

См.[[observability|Observability]].

---

## 9. Common Pitfalls

### 1. Profiling production без overhead awareness

`dotnet-gcdump` — fast (<1 sec) но pauses process. Don't capture every minute в production.

### 2. Using GC.Collect() в production

```csharp
// ❌ Force GC — обычно плохая идея
GC.Collect();
GC.WaitForPendingFinalizers();
GC.Collect();
```

GC сам решает когда лучше запускать. Force = pauses + worse perf.

**Когда OK:** между tests, perf-critical sections где знаем что только что освободили много, бенчмарки.

### 3. Confusing snapshot with leak

Snapshot показывает что **сейчас** в памяти, не что **leaked**. Может быть legitimate cache.

**Лечение:** compare snapshots over time.

### 4. Looking only at managed memory

```
GC heap:   400 MB
Working set: 1.2 GB    ← где остальные 800?
```

Native memory (interop, unmanaged buffers, ASP.NET buffers) тоже учитывается. Используй `dotnet-counters` для **Working Set**, не только GC heap.

---

## 10. Workflow — finding leak step by step

```
1. Confirm leak
   - dotnet-counters: heap monotonically растёт
   - Working set растёт
   - Pod restarts из-за OOM в k8s

2. Reproduce locally или staging
   - Load test для accelerate the leak
   - Specific scenario triggers it?

3. Capture snapshots
   - Snapshot 1: app started, idle
   - Run scenario 100x
   - Snapshot 2

4. Compare in dotMemory
   - Filter "New objects" (in 2nd, not in 1st)
   - Sort by retained size
   - Top 5 — кандидаты

5. Find GC root для top suspect
   - Right-click → Show retention path
   - Trace through references

6. Identify root cause
   - Static field?
   - Event handler не unsubscribed?
   - Long-lived collection?

7. Fix and verify
   - Apply fix
   - Repeat snapshots
   - Confirm growth stopped
```

---

## 11. Best Practices

- **dotnet-counters first** — quick signal
- **gcdump compare** — leak detection
- **dotMemory** для visual analysis
- **Continuous profiling в production** — Pyroscope etc.
- **Auto memory dumps on crash** — `DOTNET_DbgEnableMiniDump=1`
- **Health checks для memory limits**
- **Alerts** на heap growth
- **Avoid GC.Collect()** — let runtime decide
- **POH for long-lived large buffers**
- **ArrayPool в hot path** — reduce LOH pressure
- **Unsubscribe events** в Dispose
- **WeakReference** где надо избежать strong refs

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
- [[performance|Performance Deep]]
-[[gc-memory|GC и память]]
-[[diagnostics-tools|Diagnostics Tools]]
-[[span-layout|Span — снижение allocations]]
- [[caching-strategies|Caching Strategies]]

## Reading list

- **Pro .NET Memory Management** — Konrad Kokosa (must-read)
- **Konrad Kokosa blog** — tooslowexception.com
- **Maoni Stephens (CLR architect) blog** — devblogs.microsoft.com/dotnet/author/maoni
- **dotMemory documentation** — jetbrains.com/help/dotmemory
- **PerfView documentation** — github.com/microsoft/perfview/tree/main/documentation
