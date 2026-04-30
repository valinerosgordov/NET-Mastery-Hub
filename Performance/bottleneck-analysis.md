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

См. [[../SQL/optimization|SQL Optimization]].

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

См. [[../CSharp/async-threading|Async Threading]].

### N+1 в EF Core

```csharp
// Symptoms: slow API endpoint, DB CPU low, lots of small queries
// Detect: EF Core logging, dotnet-trace с Microsoft-EntityFrameworkCore-Diagnostics
// Fix: Include / Select projection
```

См. [[../EFCore/queries-performance|EF Performance]].

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

## См. также

- [[performance-fundamentals|Performance Fundamentals]]
- [[memory-profiling|Memory Profiling]]
- [[../Runtime/diagnostics-tools|Diagnostics Tools]]
- [[../Infrastructure/observability|Observability]]

## Reading list

- **Systems Performance** — Brendan Gregg (definitive)
- **Brendan Gregg blog** — brendangregg.com
- **USE Method** — brendangregg.com/usemethod.html
- **The Art of Capacity Planning** — John Allspaw
