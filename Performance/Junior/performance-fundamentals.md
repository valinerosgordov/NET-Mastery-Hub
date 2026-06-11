---
tags: [performance, fundamentals, junior, basics]
level: Junior
date: 2026-04-30
---

# Performance Fundamentals — что такое производительность

> **Базовое понимание perfomance**. Что такое "медленно", как мерить, почему оптимизировать преждевременно вредно. Введение перед глубоким `performance.md`.

---

## Что это, зачем и когда

### Что такое perfomance?

**Время и ресурсы которые код тратит на работу** — CPU, память, диск, сеть.

**Аналогия:** Машина. Скорость (CPU), вместимость багажника (память), расход топлива (electricity), время до точки B (latency). Нельзя оптимизировать "скорость" не зная **что мерить**: max speed? acceleration? fuel efficiency?

### Виды performance

| Метрика | Что это | Пример |
|---------|---------|--------|
| **Latency** | Сколько времени на 1 операцию | API response 150ms |
| **Throughput** | Сколько операций в секунду | 1000 RPS |
| **Memory** | Сколько RAM используется | 200 MB heap |
| **CPU** | % загрузки процессора | 45% CPU |
| **Allocation rate** | Скорость создания объектов | 10 MB/sec |
| **GC pressure** | Как часто срабатывает GC | 100 Gen0/min |

### Зачем мерить?

Без замеров — "интуитивная оптимизация" = угадывание:
- "Я думаю это медленно" → переписал → **стало медленнее**
- "StringBuilder быстрее string +" → для 5 строк дольше + аллокация
- "LINQ медленный" → в production hot path неважно, в loop — критично

**Правило:** **No measurement = no optimization**.

---

## 1. Big O — сложность алгоритмов

### Зачем

Знаешь сложность — предсказываешь как код поведёт на больших данных.

```csharp
// O(1) — константное время, не зависит от размера
int x = list[5];

// O(n) — линейно, n операций для n элементов
foreach (var item in list) { ... }

// O(n²) — квадратично, n*n операций. На 1000 = 1M операций!
foreach (var a in list)
    foreach (var b in list)
        if (a == b) ...

// O(log n) — логарифмически, очень быстро. 1M элементов = 20 операций
binarySearch(sortedList, target);

// O(n log n) — sort
list.Sort();
```

### Practical таблица

| n=10 | O(1) | O(log n) | O(n) | O(n log n) | O(n²) | O(2^n) |
|------|------|----------|------|------------|-------|--------|
| 10 | 1 | 3 | 10 | 33 | 100 | 1024 |
| 100 | 1 | 7 | 100 | 664 | 10k | 10^30 |
| 1000 | 1 | 10 | 1k | 10k | 1M | ∞ |
| 1M | 1 | 20 | 1M | 20M | 10^12 | ∞ |

### Сложности .NET коллекций

| Operation | List\<T\> | Dictionary | HashSet | LinkedList |
|-----------|-----------|-----------|---------|------------|
| Add | O(1) amort | O(1) avg | O(1) avg | O(1) |
| Lookup by index | O(1) | — | — | O(n) |
| Lookup by key | O(n) | O(1) avg | O(1) avg | O(n) |
| Insert at start | O(n) | — | — | O(1) |
| Remove by value | O(n) | O(1) | O(1) | O(n) |
| Contains | O(n) | O(1) | O(1) | O(n) |

```csharp
// ❌ O(n²) — для каждого item ищем в list
var ids = new List<int> { 1, 2, 3, /* ... 10000 items */ };
var matches = new List<int>();
foreach (var item in items)
{
    if (ids.Contains(item.Id))  // O(n) Contains в List!
        matches.Add(item.Id);
}
// Total: O(items × ids) = O(n × m)

// ✅ O(n) — HashSet даёт O(1) Contains
var idsSet = new HashSet<int>(ids);
foreach (var item in items)
{
    if (idsSet.Contains(item.Id))  // O(1)
        matches.Add(item.Id);
}
// Total: O(n + m)
```

---

## 2. Workflow optimization

```
┌─────────────────────────────────────────────────┐
│  1. MEASURE — есть ли проблема?                 │
│     - Profile production / load test            │
│     - Найди slowest endpoint / function         │
└─────────────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────────────┐
│  2. ANALYZE — где bottleneck?                   │
│     - dotTrace / PerfView / dotnet-trace        │
│     - dotnet-counters live monitoring           │
│     - Memory leak? CPU bound? I/O bound?        │
└─────────────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────────────┐
│  3. OPTIMIZE — фикси одно место                 │
│     - Algorithm change (O(n²) → O(n))           │
│     - Caching                                   │
│     - Async I/O                                 │
└─────────────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────────────┐
│  4. MEASURE AGAIN — стало лучше?                │
│     - Сравни до/после                           │
│     - Если хуже — откатывай                     │
└─────────────────────────────────────────────────┘
```

**Не пропускай шаг 1 и 4!** Это половина успеха.

---

## 3. Profiling tools для Junior

### dotnet-counters — live monitoring

```bash
# Установить
dotnet tool install -g dotnet-counters

# Find process
dotnet-counters ps

# Monitor
dotnet-counters monitor -p 12345 --providers System.Runtime
```

```
[System.Runtime]
    CPU Usage (%)                       45
    GC Heap Size (MB)                   320
    Gen 0 GC Count (Count / 1 sec)      5
    Gen 1 GC Count (Count / 1 sec)      2
    Gen 2 GC Count (Count / 1 sec)      0
    ThreadPool Thread Count             16
    Working Set (MB)                    480
```

Если CPU 100% → CPU bound problem. Если GC Count high → too many allocations.

### BenchmarkDotNet — micro-benchmarks

```csharp
[MemoryDiagnoser]
public class StringConcatBench
{
    private readonly string[] _items = Enumerable.Range(0, 100).Select(i => $"item-{i}").ToArray();

    [Benchmark(Baseline = true)]
    public string PlusOperator()
    {
        var s = "";
        foreach (var item in _items) s += item;
        return s;
    }

    [Benchmark]
    public string StringBuilder()
    {
        var sb = new System.Text.StringBuilder();
        foreach (var item in _items) sb.Append(item);
        return sb.ToString();
    }

    [Benchmark]
    public string StringJoin() => string.Join("", _items);
}
```

```bash
dotnet run -c Release
```

```
| Method        | Mean      | Allocated |
|-------------- |----------:|----------:|
| PlusOperator  | 95.32 us  |  100 KB   | ← медленно, много аллокаций
| StringBuilder |  3.45 us  |    2 KB   |
| StringJoin    |  2.10 us  |    1 KB   | ← лучший
```

См. [[performance|Performance — глубоко]] для полного гайда.

---

## 4. Common bottlenecks для Junior

### 1. N+1 queries

```csharp
// ❌ 1 + N queries
var orders = db.Orders.ToList();        // 1 query
foreach (var order in orders)
{
    var user = db.Users.Find(order.UserId);  // N queries!
    Console.WriteLine($"{user.Name}: {order.Total}");
}

// ✅ 1 query
var orders = db.Orders.Include(o => o.User).ToList();
```

См.[[queries-performance|EF Queries Performance]].

### 2. String concatenation в loop

```csharp
// ❌ O(n²) — каждое + создаёт новую строку
var s = "";
for (int i = 0; i < 10_000; i++)
    s += i.ToString();

// ✅ StringBuilder
var sb = new StringBuilder();
for (int i = 0; i < 10_000; i++)
    sb.Append(i);
var s = sb.ToString();
```

### 3. Synchronous I/O (file, HTTP, DB)

```csharp
// ❌ Блокирует thread на 1 сек = thread не обрабатывает другие requests
public string GetData()
{
    return File.ReadAllText("data.txt");
}

// ✅ Async — thread свободен пока ждём диск
public async Task<string> GetDataAsync()
{
    return await File.ReadAllTextAsync("data.txt");
}
```

### 4. Не нужный allocations

```csharp
// ❌ Создаёт новый list каждый вызов
public List<int> GetActiveIds()
{
    return _items.Where(i => i.IsActive).Select(i => i.Id).ToList();
}

// ✅ IEnumerable — lazy, без list allocation
public IEnumerable<int> GetActiveIds()
{
    return _items.Where(i => i.IsActive).Select(i => i.Id);
}
```

### 5. LINQ в hot path с allocations

```csharp
// ❌ В hot loop — каждый Where/Select создаёт enumerator
foreach (var item in items.Where(i => i.IsActive).Select(i => i.Value))
{ ... }

// ✅ Если perf критичен — простой for с Span
foreach (var item in items.AsSpan())
{
    if (!item.IsActive) continue;
    ProcessValue(item.Value);
}
```

### 6. ToList() / ToArray() ненужно

```csharp
// ❌ Material всё в list, потом считаешь — extra allocation
var count = items.Where(i => i.IsActive).ToList().Count;

// ✅ Count прямо
var count = items.Count(i => i.IsActive);

// ❌ Двойная enumeration
var data = GetData().ToList();  // первая
return data.Count == 0 ? null : data.First();  // вторая

// ✅ Один проход
var first = GetData().FirstOrDefault();
return first;
```

### 7. Cache для повторяющихся вычислений

```csharp
// ❌ Каждый запрос — поход в БД
public Settings GetSettings(string key)
{
    return _db.Settings.First(s => s.Key == key);
}

// ✅ Cache (settings меняются редко)
private readonly IMemoryCache _cache;

public Settings GetSettings(string key) =>
    _cache.GetOrCreate($"settings_{key}", entry =>
    {
        entry.SlidingExpiration = TimeSpan.FromMinutes(5);
        return _db.Settings.First(s => s.Key == key);
    });
```

См.[[caching|Caching]].

---

## 5. Premature optimization

> "Premature optimization is the root of all evil." — Donald Knuth

### Почему преждевременная оптимизация плохо

```csharp
// "Я слышал что массивы быстрее List"
public class WeirdCode
{
    private readonly int[] _items = new int[1000];
    private int _count = 0;

    public void Add(int x)
    {
        if (_count == _items.Length)
        {
            var newArr = new int[_items.Length * 2];
            Array.Copy(_items, newArr, _items.Length);
            // Stop! Ты пишешь List вручную.
        }
        _items[_count++] = x;
    }
}

// ✅ Просто
public class CleanCode
{
    private readonly List<int> _items = new();
    public void Add(int x) => _items.Add(x);
}
```

**Затраты premature optimization:**
- Сложный код, читать сложно
- Бывает медленнее (сложнее = больше bugs)
- Нет measurement → может быть оптимизировал не то

### Когда оптимизировать

```
1. Есть problem (slow user feedback / monitoring alert)
2. Profiled и нашёл bottleneck
3. Bottleneck реально big (>10% времени)
4. Понятна alternative
5. Тесты есть
```

---

## 6. Memory vs CPU optimization — баланс

### CPU bound

Algorithm slow, considerные циклы. Profile показывает CPU 100%.

**Лечение:**
- Algorithm improvement (O(n²) → O(n))
- Parallel processing (Parallel.ForEach, PLINQ)
- SIMD (Vector\<T\>)

### Memory bound

Лишние allocations, GC занят, heap растёт.

**Лечение:**
- Reuse объектов (ArrayPool\<T\>)
- Span\<T\> вместо substring
- Struct вместо class где можно
- Avoid LINQ in hot path

### I/O bound

Ждём диск / сеть / DB.

**Лечение:**
- Async (нет блокировки threads)
- Batch operations
- Caching

---

## 7. Memory allocation — откуда берутся аллокации

```csharp
// ❌ Hidden allocations
public string GetGreeting(string name)
{
    return "Hello, " + name + "!";  // 1 string allocation
}

public List<int> Doubled(IEnumerable<int> items)
{
    return items.Select(x => x * 2).ToList();  // List + Select enumerator
}

public void Log(string action, int userId)
{
    _logger.LogInformation($"{action} by user {userId}");  // string interpolation = allocation
}

// ✅ Optimized для hot path
public string GetGreeting(ReadOnlySpan<char> name)
{
    return string.Concat("Hello, ", name, "!");
}

// LoggerMessage source generator — zero allocation
[LoggerMessage(Level = LogLevel.Information, Message = "{Action} by user {UserId}")]
public static partial void LogAction(this ILogger logger, string action, int userId);
```

> [!info] Не оптимизируй случайно
> Если код вне hot path — readability > perf. `$"..."` interpolation OK для логов которые редко.

---

## 8. SLA, SLO, SLI — performance в production

| | Что |
|--|-----|
| **SLI** (Service Level Indicator) | Метрика что мы измеряем (latency p99) |
| **SLO** (Service Level Objective) | Цель (p99 < 500ms 99% времени) |
| **SLA** (Service Level Agreement) | Контракт с клиентом (uptime 99.9%) |

```
SLI: API latency p99
SLO: < 500ms за 28 дней
SLA: 99.9% uptime
```

Если **SLO нарушен** — performance work становится приоритетом.

См.[[observability|Observability]].

---

## 9. Mental models

### Latency numbers (Jeff Dean)

```
L1 cache reference                       0.5 ns
Branch mispredict                        5   ns
L2 cache reference                       7   ns
Mutex lock/unlock                        25  ns
Main memory reference                    100 ns
Compress 1 KB with Zippy                 3,000 ns
Send 1 KB over 1 Gbps network            10,000 ns       = 10 µs
Read 4 KB randomly from SSD              150,000 ns      = 150 µs
Read 1 MB sequentially from memory       250,000 ns      = 250 µs
Round trip within same datacenter        500,000 ns      = 500 µs
Read 1 MB sequentially from SSD          1,000,000 ns    = 1 ms
Disk seek                                10,000,000 ns   = 10 ms
Read 1 MB sequentially from disk         20,000,000 ns   = 20 ms
Send packet CA->Netherlands->CA          150,000,000 ns  = 150 ms
```

**Insights:**
- Memory ~ 10000x faster than disk
- Network call ~ 5000x slower than memory
- Каждый network round trip — десятки ms

Поэтому **caching** и **batching** так важны.

### Performance budget

```
Total request budget: 200ms
├── DB query        :  50ms
├── Business logic  :  20ms
├── External API    : 100ms
├── Serialization   :  10ms
└── Other           :  20ms
```

Если budget exceeded — нужна оптимизация **большого куска**, не маленьких.

---

## 10. Common Pitfalls (новички)

### 1. Optimization без measurement

См. секцию выше.

### 2. "Это быстрее" — без context

```csharp
// "for быстрее foreach"
for (int i = 0; i < list.Count; i++) { ... }
foreach (var item in list) { ... }
```

В реальности — почти equal в .NET 8+ (compiler оптимизирует foreach to for на arrays).

**Test → measure → solid evidence**.

### 3. Игнорирование GC

В .NET high allocation rate = частый GC = pauses = slow.

```csharp
// ❌ Allocates 1M strings
for (int i = 0; i < 1_000_000; i++)
    Console.WriteLine($"Item {i}");

// ✅ Reuse через StringBuilder или Span
```

См.[[gc-memory|GC и память]].

### 4. Async ради async

```csharp
// ❌ Async на CPU-bound work — overhead async без выгод
public async Task<int> CalculateAsync(int x)
{
    return await Task.FromResult(x * 2);
}

// ✅ CPU-bound — sync OK
public int Calculate(int x) => x * 2;

// ✅ I/O bound — async имеет смысл
public async Task<string> ReadFileAsync(string path) =>
    await File.ReadAllTextAsync(path);
```

### 5. Cache без invalidation

```csharp
// ❌ Cache навсегда — stale data
private static Dictionary<int, User> _cache = new();

public User Get(int id)
{
    if (!_cache.ContainsKey(id))
        _cache[id] = _db.Users.Find(id);
    return _cache[id];
}
// User обновляется в БД — cache не знает!
```

**Лечение:** TTL, invalidation events.

См.[[caching|Caching]].

### 6. ConfigureAwait в app code

```csharp
// ❌ ConfigureAwait(false) везде — cargo cult
await SomeAsync().ConfigureAwait(false);
await OtherAsync().ConfigureAwait(false);
```

В **library** — да. В **ASP.NET Core app code** — не нужен (нет SynchronizationContext).

См.[[async-threading|Async и Threading]].

---

## 11. Best Practices summary

- **Measure first** — не оптимизируй без measurement
- **Profile → fix biggest** — Pareto принцип
- **Algorithm matters most** — O(n²) → O(n) даёт 1000x на больших данных
- **Cache hot data** — но с TTL/invalidation
- **Async I/O** — не блокируй threads
- **Avoid premature optimization** — clean code first, optimize later
- **Set performance budgets** — например, p99 < 500ms
- **SLO в production** — alerts when violated
- **Monitor allocations** — high GC = problem

---

## 12. Roadmap для роста

### Junior

- [ ] Знаешь Big O
- [ ] Понимаешь N+1 problem
- [ ] Используешь async для I/O
- [ ] Знаешь BenchmarkDotNet basics

### Middle

- [ ] dotnet-counters / dotnet-trace для troubleshooting
- [ ] Profiling с dotTrace / PerfView
- [ ] Performance budget на endpoints
- [ ] Optimization patterns (caching, batching)
- [ ] Knowledge of GC behavior

### Senior

- [ ] Span\<T\> / Memory\<T\> в hot paths
- [ ] ArrayPool, ObjectPool
- [ ] SIMD / Vector\<T\>
- [ ] Native AOT для startup
- [ ] Production load testing
- [ ] Continuous profiling (Pyroscope)
- [ ] Custom GC tuning

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

- [[performance|Performance — детальный гайд]]
- [[hft-low-latency|HFT / Low Latency]]
-[[gc-memory|GC и память]]
-[[diagnostics-tools|Diagnostics Tools]]
-[[queries-performance|EF Queries Performance]]
-[[caching|Caching]]
-[[mutation-load-testing|Load Testing]]

## Reading list

- **Pro .NET Memory Management** — Konrad Kokosa
- **Writing High-Performance .NET Code** — Ben Watson
- **Stephen Toub blog series** — devblogs.microsoft.com/dotnet
- **Adam Sitnik blog** — adamsitnik.com
- **Jeff Dean — Latency Numbers Every Programmer Should Know**
- **Microsoft Docs — Performance** — learn.microsoft.com/dotnet/core/performance
