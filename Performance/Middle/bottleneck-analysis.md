---
tags: [performance, bottleneck, analysis, troubleshooting]
level: Middle to Senior
date: 2026-04-30
---

# Bottleneck Analysis — поиск узких мест

> **Систематический подход** к нахождению где система тормозит. CPU, memory, I/O, network, DB — как определить какой ресурс ограничивает.

---

## Что это, зачем и когда

### Bottleneck — одна из причин

В любой системе есть ОДНО место которое ограничивает throughput. Like цепь — only прочна как самое слабое звено.

**USE method (Brendan Gregg):**
```
For each resource, check:
- U: Utilization      (% busy)
- S: Saturation        (queue length)
- E: Errors            (errors / sec)
```

---

## 1. Resources to monitor

### CPU

```bash
# Linux
top
htop
mpstat -P ALL 1

# Per-process
pidstat -p $(pidof dotnet) 1

# .NET-specific
dotnet-counters monitor -p PID --counters System.Runtime
```

**Bottleneck signals:**
- Sustained CPU > 80%
- Run queue length > number of cores
- Context switches >> 1000/sec
- High kernel time (% sys) — locks contention

### Memory

```bash
free -h
vmstat 1
```

**Bottleneck signals:**
- Swap usage > 0 → critical
- Page faults / sec high
- GC time > 10% (для .NET)
- OOM killer logs

### I/O (Disk)

```bash
iostat -x 1
iotop
```

**Bottleneck signals:**
- %util > 80% on disks
- await > 50ms
- High write/read queues

### Network

```bash
sar -n DEV 1
iftop
ss -s
```

**Bottleneck signals:**
- Bandwidth utilization > 70%
- TCP retransmits > 1%
- Connection refused / timeouts
- TIME_WAIT exhaustion

### Database

```sql
-- PostgreSQL
SELECT query, total_exec_time, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

**Bottleneck signals:**
- Slow queries (>100ms typical OLTP)
- Lock waits
- Connection pool exhaustion
- High WAL writes

См. [[optimization|SQL Optimization]].

---

## 2. .NET-specific bottlenecks

### CPU bound

```csharp
// Symptoms: CPU 100%, low GC, single-threaded utilization
// Fix: parallelism, algorithm improvement, SIMD

// Tools:
dotnet-trace collect -p PID --providers System.Runtime
// Open в speedscope.app для flame graph
```

### Memory pressure

```csharp
// Symptoms: GC.Gen2 frequent, allocation rate high
// Fix: ArrayPool, Span, fewer allocations

dotnet-counters monitor --counters System.Runtime[gc-heap-size,allocation-rate,gen-2-gc-count]
```

### Lock contention

```csharp
// Symptoms: high wait time, low CPU usage
// Fix: lock-free structures, smaller critical sections

// Detect через PerfView или dotnet-trace + ConcurrencyVisualizer
```

### Async deadlock

```csharp
// Symptoms: hanging, no progress
// Cause: .Result или .Wait() в async chain

// Fix: async all the way

// Detect:
dotnet-stack report -p PID
// Видно threads blocked on Task.Wait
```

### ThreadPool starvation

```csharp
// Symptoms: high request queue, SignalR/HTTP slow
// Cause: blocking calls в ThreadPool threads

// Fix:
// - Async I/O везде
// - Await не блокирует
// - Long-running CPU work — в Task.Run

// Detect:
dotnet-counters monitor --counters System.Runtime[threadpool-thread-count,threadpool-queue-length]
```

См. [[async-threading|Async Threading]].

### N+1 в EF Core

```csharp
// Symptoms: slow API endpoint, DB CPU low, lots of small queries
// Detect: EF Core logging, dotnet-trace с Microsoft-EntityFrameworkCore-Diagnostics
// Fix: Include / Select projection
```

См. [[queries-performance|EF Performance]].

---

## 3. Workflow — bottleneck hunt

```
┌─────────────────────────────┐
│ 1. Symptoms                 │
│ Slow / errors / OOM?         │
└─────────────────────────────┘
        ↓
┌─────────────────────────────┐
│ 2. USE method check          │
│ U/S/E for each resource      │
└─────────────────────────────┘
        ↓
┌─────────────────────────────┐
│ 3. Identify bottleneck      │
│ Which resource saturated?    │
└─────────────────────────────┘
        ↓
┌─────────────────────────────┐
│ 4. Drill down                │
│ Which code caused it?        │
└─────────────────────────────┘
        ↓
┌─────────────────────────────┐
│ 5. Fix                       │
│ Reduce / optimize / batch    │
└─────────────────────────────┘
        ↓
┌─────────────────────────────┐
│ 6. Verify                    │
│ Bottleneck moved? Resolved?  │
└─────────────────────────────┘
```

> [!info] Bottleneck doesn't disappear, only moves
> Fix CPU bottleneck → теперь network bottleneck. Fix network → теперь DB bottleneck. Like водопад — fix top step → next step становится bottleneck.

---

## 4. Tools matrix

| Concern | Tool |
|---------|------|
| CPU usage | `top`, `htop`, dotnet-counters |
| CPU profile | dotnet-trace, dotTrace, PerfView |
| Memory | dotnet-counters, gcdump, dotMemory |
| Allocations | dotnet-trace AllocationTick |
| GC | dotnet-counters, PerfView |
| Locks | dotnet-trace, ConcurrencyVisualizer |
| I/O | `iostat`, `iotop` |
| Network | `iftop`, `ss`, `tcpdump` |
| DB queries | EF Logging, pg_stat_statements, slow query log |
| HTTP | Postman, curl, Apache Bench |
| Distributed | OpenTelemetry, Jaeger, Datadog |

---

## 5. Common pitfalls

### Optimizing wrong thing

Profile says **CPU** is bottleneck → optimizing memory not poможет.

### Local vs production

Local: 1 user, fast disk, no contention.
Production: 1000 users, slow disk, contended.

**Test on production-like load** для real bottlenecks.

### Premature parallelization

Adding parallelism if CPU not saturated — slower (overhead) и harder to debug.

### Fixing symptom not root cause

```
Symptom: slow API
"Fix": add timeout
Real cause: N+1 query
```

---

## 6. Best Practices

- **USE method для каждого ресурса**
- **Profile production** (continuous profiling)
- **Set SLOs** — known thresholds для alerts
- **One change at a time** — измеряй diff
- **Document findings** — bottleneck reports → wiki
- **Test under load** перед production deploy
- **Monitor after fix** — bottleneck moved где?

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

См. [[async-threading|async-threading]] и [[memory-pooling|Memory Pooling]].


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
- [[memory-profiling|Memory Profiling]]
- [[diagnostics-tools|Diagnostics Tools]]
- [[observability|Observability]]

## Reading list

- **Systems Performance** — Brendan Gregg (definitive)
- **Brendan Gregg blog** — brendangregg.com
- **USE Method** — brendangregg.com/usemethod.html
- **The Art of Capacity Planning** — John Allspaw
