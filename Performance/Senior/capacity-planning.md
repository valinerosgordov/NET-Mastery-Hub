---
tags: [performance, capacity-planning, scalability, sla]
level: Senior
date: 2026-04-30
---

# Capacity Planning — планирование мощностей

> **Сколько ресурсов нужно для текущей и будущей нагрузки**. Расчёт CPU, memory, network, БД для заданного traffic.

---

## Что это, зачем и когда

### Capacity planning — зачем

Без planning:
- Over-provisioned: тратишь деньги впустую
- Under-provisioned: системы падают под нагрузкой

### Когда планировать

- Перед launch продукта
- Before known traffic spike (sale, event)
- Quarterly review при росте
- After major architecture change

---

## 1. Определи единицу нагрузки

```
Web API:   requests / second (RPS)
Database:  queries / second (QPS)  
Worker:    jobs / hour
Storage:   GB / day
Network:   Mbps / Gbps
```

---

## 2. Измерь текущую нагрузку

### Production метрики

```
Today:
- Peak RPS:        1,500
- Average RPS:       300
- p99 latency:    250 ms
- CPU usage:       45%
- Memory:          60% of 4 GB
- DB connections:  50 / 100 max
```

Из Prometheus / Datadog.

---

## 3. Расчёт capacity

### Little's Law

```
L = λ × W

L = number of requests in system at any moment
λ = arrival rate (RPS)
W = average time in system (latency)
```

**Пример:**
```
λ = 1000 RPS
W = 100 ms = 0.1 s
L = 1000 × 0.1 = 100 concurrent requests
```

Нужно threads / connections поддерживать 100 одновременно. Для async ASP.NET — это 100 in-flight tasks (легко). Sync — 100 threads.

### CPU capacity

```
1 core = 1000 ms / sec computation

If request takes 50ms CPU:
  per core = 1000/50 = 20 RPS

Need 1000 RPS = 1000/20 = 50 cores
+ headroom 30% = 65 cores

= 8 nodes × 8 cores
```

### Memory capacity

```
Per request memory: ~5 MB (working set)
Concurrent requests: 100
Total: 500 MB + base 1 GB = 1.5 GB

Per pod: 2 GB request, 4 GB limit
```

### Database capacity

```
QPS: 5000
Connection pool: 100
Max QPS per connection: 50

Need: 5000/50 = 100 connections (just at limit!)

Solution:
- Read replicas (split read load)
- Connection pooling (PgBouncer)
- Caching (reduce DB hits)
```

### Network capacity

```
Average response: 50 KB
RPS: 1000
Bandwidth: 50 KB × 1000 = 50 MB/s = 400 Mbps

Need 1 Gbps minimum.
```

---

## 4. Growth projection

```
Year 1: 1000 RPS peak
Year 2: ?

Growth assumptions:
- User growth: 50%/year
- Traffic per user: 1.2x (more usage)
- Total: 1.5 × 1.2 = 1.8x

Year 2: 1800 RPS
Year 3: 1800 × 1.8 = 3240 RPS
```

Plan capacity для year+1.

---

## 5. Headroom

```
Provision for: peak × buffer

Buffer = 30-50% headroom

Reasons:
- Bursts beyond peak
- One node failure
- Maintenance/deploys
- Unexpected traffic
```

---

## 6. Vertical vs Horizontal scaling

### Vertical (scale up)

Bigger machine. Limits:
- Hardware ceiling
- Cost grows nonlinearly
- Single point of failure

### Horizontal (scale out)

More machines. Better for cloud:
- Linear cost
- HA built-in
- Need stateless app

Most modern cloud apps — horizontal первым делом.

---

## 7. Database scaling

### Read replicas

```
Primary  ←─ writes
   ↓ replication
Replica 1  ←─ reads
Replica 2
Replica 3
```

Eventual consistency между primary и replicas.

### Sharding

Partition data by key:
```
Users 1-1000:    Shard A
Users 1001-2000: Shard B
Users 2001-3000: Shard C
```

### Caching layer

1 cache hit ≠ 1 DB query → reduce DB load 80-90%.

См. [[caching-strategies|Caching Strategies]].

---

## 8. Cost optimization

### Reserved vs On-demand (cloud)

Reserved (1-3 year commit) = 30-60% cheaper.
On-demand = pay as you go.

**Strategy:** baseline traffic → reserved. Spikes → on-demand / spot.

### Spot instances (AWS) / Preemptible (GCP)

70-90% дешевле but могут disappear с 30s warning.

Хорошо для:
- Batch jobs
- Stateless workers
- Test environments

### Right-sizing

```
Currently: 8 cores, 32 GB RAM
Actual usage: 30% CPU, 40% RAM peak

→ Right-size to 4 cores, 16 GB
→ 50% cost savings
```

---

## 9. Load testing для validation

См. [[mutation-load-testing|Load Testing]].

```bash
# k6 example
k6 run --vus 1000 --duration 5m load-test.js
```

If система выдерживает X RPS — capacity confirmed.

---

## 10. Common pitfalls

### Plan for average, not peak

Peak ≠ average. Plan for peak + buffer.

### Ignore growth

Plan for current — через 6 месяцев упадёт.

### Forget about background jobs

API capacity 1000 RPS — но workers тоже жрут ресурсы.

### Single load test ≠ reality

Load tests synthetic. Production patterns могут differ.

### No headroom

100% utilization → first hiccup → cascade failure.

---

## Best Practices

- **Monitor** continuously (Prometheus + alerts)
- **Set SLOs** — known performance targets
- **Plan для peak × 1.5** — headroom
- **Project growth** — year+1 minimum
- **Load test** quarterly
- **Document assumptions** — review when invalid
- **Scale horizontally** when possible
- **Cost optimization** — reserved + spot
- **Right-size periodically** — don't overprovision

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
- [[bottleneck-analysis|Bottleneck Analysis]]
- [[mutation-load-testing|Load Testing]]
- [[distributed-systems|Distributed Systems]]
- [[observability|Observability]]

## Reading list

- **The Art of Capacity Planning** — John Allspaw
- **Site Reliability Engineering** — Google (free online: sre.google/books)
- **Designing Data-Intensive Applications** — Kleppmann
