---
tags: [performance, budget, slo, sla, monitoring]
level: Senior
date: 2026-04-30
---

# Performance Budgets — целевые показатели

> **Performance как functional requirement** — устанавливаем cost targets, мониторим, alert когда превышаем.

---

## Что это, зачем и когда

### Performance budget — что это

**Заранее установленные пределы** на performance metrics. Build / PR fails если превышаются.

**Аналогия:** Финансовый budget. Заранее решил тратить на еду $X в месяц. Превышаешь — alert. Без budget — потратишь сколько потратится.

### Зачем budgets

Без budgets:
- Performance медленно деградирует с каждым PR
- Никто не замечает пока юзеры не жалуются
- "Где наш p99 5 лет назад? — не помним"

С budgets:
- Каждый PR проверяется — стало хуже?
- Regression блокируется ДО merge
- Команда aware о cost изменений

---

## 1. Какие метрики

### API endpoints

```
POST /api/orders
  p50: 50ms
  p95: 200ms
  p99: 500ms
  RPS: ≥ 1000 sustained

GET /api/products
  p50: 20ms
  p95: 100ms
  p99: 250ms
  RPS: ≥ 5000 sustained
```

### Frontend (Web Vitals)

```
LCP (Largest Contentful Paint):  < 2.5s
FID (First Input Delay):           < 100ms
CLS (Cumulative Layout Shift):     < 0.1
TTI (Time To Interactive):         < 3.8s
Bundle size:                      < 200 KB
```

### Resource budgets

```
CPU (per pod):     < 70% sustained
Memory:            < 70% of limit
GC pause time:     < 50ms p99
Allocation rate:   < 10 MB/sec
DB connections:    < 80% of pool
```

---

## 2. Setting budgets

### Approach 1: SLO-based

```
Customer SLA: 99% of API calls < 500ms

Working backwards:
- p99 budget: 500ms total

Sub-budgets:
- DB query:         200ms
- Business logic:    50ms
- External API:     200ms
- Serialization:     20ms
- Other:             30ms
```

### Approach 2: Historical baseline

```
Look at production:
- p99 today: 250ms
- Best month last year: 200ms

Budget: 220ms (slightly worse than best, but achievable)
```

### Approach 3: Competitive

```
Industry standard для checkout: 1 sec
Our budget: 800ms (10% better than competitors)
```

---

## 3. Enforcement в CI

### Unit-level: BenchmarkDotNet thresholds

```csharp
public class PerformanceBudget
{
    [Fact]
    public void CalculatePrice_under_5ms()
    {
        var sw = Stopwatch.StartNew();
        for (int i = 0; i < 1000; i++)
            calculator.Calculate(orders);
        sw.Stop();
        
        var avgMs = sw.ElapsedMilliseconds / 1000.0;
        avgMs.Should().BeLessThan(5);
    }
}
```

### Integration: load test в CI

```csharp
[Fact]
public async Task GetOrders_p95_under_200ms()
{
    var scenario = Scenario.Create("get_orders", async ctx =>
    {
        var resp = await client.GetAsync("/api/orders");
        return resp.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
    })
    .WithLoadSimulations(
        Simulation.KeepConstant(100, TimeSpan.FromMinutes(1))
    );
    
    var stats = NBomberRunner.RegisterScenarios(scenario).Run();
    var p95 = stats.ScenarioStats[0].Ok.Latency.Percent95;
    
    p95.Should().BeLessThan(200);  // Budget: p95 < 200ms
}
```

См.[[mutation-load-testing|Load Testing]].

### CI workflow

```yaml
# .github/workflows/perf-test.yml
- name: Performance budget check
  run: dotnet test --filter Category=Performance
  
# Если budget exceeded — workflow fails → PR не merge
```

---

## 4. Frontend budgets

### Webpack / Vite warnings

```javascript
// vite.config.js
export default {
    build: {
        chunkSizeWarningLimit: 500,  // KB warning
        rollupOptions: {
            output: {
                manualChunks: { /* split */ }
            }
        }
    }
};
```

### Lighthouse CI

```yaml
- name: Lighthouse
  uses: treosh/lighthouse-ci-action@v10
  with:
    urls: |
      https://staging.example.com/
    budgetPath: ./lighthouse-budget.json
    uploadArtifacts: true
```

```json
// lighthouse-budget.json
[
  {
    "resourceSizes": [
      { "resourceType": "script", "budget": 300 },
      { "resourceType": "total", "budget": 500 }
    ],
    "timings": [
      { "metric": "interactive", "budget": 3000 }
    ]
  }
]
```

---

## 5. Production monitoring

### SLI tracking

```
SLI: API endpoint p99 latency
SLO: < 500ms за 28 days
Error budget: 1% of requests can be > 500ms
```

### Burn rate alerts

```yaml
- alert: SLOBudgetBurningTooFast
  expr: rate(http_requests_slow_total[1h]) > 0.01
  for: 30m
  annotations:
    summary: "Error budget consumed 30x faster than allowed"
```

См.[[observability|Observability]].

### Dashboards

```
Grafana panels:
- p50/p95/p99 latency by endpoint
- RPS / error rate
- Database query times
- GC metrics
- Memory / CPU usage
- Performance budget vs actual
```

---

## 6. When violated

### Immediate

1. **Check** — true regression или anomaly?
2. **Find cause** — git bisect, check recent deploys
3. **Decide** — rollback, fix forward, accept

### Process

```
1. Alert fires
2. On-call investigates
3. Root cause analysis
4. Fix or rollback
5. Post-mortem
6. Update budget if needed
```

---

## 7. Budget evolution

Budgets — не immutable.

### Loosening (when justified)

- New feature легитимно дороже
- Trade-off: feature value > perf cost
- Document why

### Tightening

- Optimization made — capture gain
- New competitive pressure
- Customer SLA tightened

---

## 8. Common pitfalls

### Setting budgets без data

```
"p99 should be < 100ms" 

But:
- Current p99: 500ms
- Load: 100 RPS
- DB query: avg 80ms

Budget: impossible to meet
```

**Лечение:** Baseline first, set realistic target.

### Budgets only on perf-critical paths

Budget на `/health` нет смысла. Budget на `/api/orders` — да.

Focus на user-facing critical paths.

### No remediation plan

```
"We have budget. p99 violated. Now what?"
```

**Лечение:**
- Runbook для violations
- Ownership defined
- Remediation strategies

### Budget shifting blame

```
"Backend team: API budget violated"
"Backend: it's DB team's fault"
```

**Лечение:** budget — team responsibility. Blameless RCA.

---

## 9. Best Practices

- **Set budgets early** — before drift
- **Realistic targets** — based on data
- **Sub-budgets** — breakdown by component
- **CI enforcement** — block regressions
- **Production monitoring** — alerts on violation
- **Burn rate** — early warning
- **Document rationale** — why this number
- **Review quarterly** — adjust as needed
- **Blameless** culture
- **Post-mortems** для violations

---

## 10. Example budget document

```markdown
# Performance Budgets — Order Service

## SLOs

- p99 latency для POST /api/orders < 500ms (99% time)
- Availability: 99.9%
- RPS sustained: ≥ 1000

## Sub-budgets

| Component | Budget |
|-----------|--------|
| Validation | 20ms |
| DB query (write) | 50ms |
| Inventory check | 100ms |
| Payment external API | 200ms |
| Email queue | 30ms |
| Total + buffer | 500ms |

## Resources

- CPU: < 70% sustained
- Memory: < 1.5 GB / 2 GB limit
- DB connections: < 50 / 100

## Monitoring

- Datadog dashboard: [link]
- Alerts: PagerDuty rotation

## Owners

- Tech: backend team @user
- Product: @pm-name
- Last review: 2026-01

## History

- 2026-01: Tightened from 800ms to 500ms (after caching)
- 2025-09: Initial: 800ms
```

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
- [[bottleneck-analysis|Bottleneck Analysis]]
- [[capacity-planning|Capacity Planning]]
-[[observability|Observability]]
-[[mutation-load-testing|Load Testing]]

## Reading list

- **Site Reliability Engineering** — Google (sre.google/books)
- **The Site Reliability Workbook** — Google
- **Google's SLO documents** — sre.google/sre-book/service-level-objectives
- **Brendan Gregg blog** — brendangregg.com
