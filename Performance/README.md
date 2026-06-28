# Performance — производительность

> 12 файлов / ~252 KB. Profiling, optimization, caching, HFT, ThreadPool starvation. От Junior basics до hot path optimization.

[← Главный README](../readme.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Junior, что такое perf | [`performance-fundamentals.md`](Junior/performance-fundamentals.md) |
| Senior, нужны tools | [`performance.md`](Senior/performance.md) — BenchmarkDotNet, dotMemory |
| Slow application | [`bottleneck-analysis.md`](Middle/bottleneck-analysis.md) → [`memory-profiling.md`](Senior/memory-profiling.md) |
| Нужен cache | [`caching-strategies.md`](Middle/caching-strategies.md) |
| Sub-millisecond latency | [`hft-low-latency.md`](Senior/hft-low-latency.md) |
| Throughput падает при низком CPU | [`threadpool-starvation-hill-climbing.md`](Senior/threadpool-starvation-hill-climbing.md) |
| Capacity planning | [`capacity-planning.md`](Senior/capacity-planning.md), [`performance-budgets.md`](Senior/performance-budgets.md) |

---

## 📚 Все 12 файлов

### Fundamentals (Junior to Senior)

| Файл | Уровень | Описание |
|------|---------|----------|
| [`performance-fundamentals.md`](Junior/performance-fundamentals.md) | Junior/Middle | Что такое производительность, basics |
| [`performance.md`](Senior/performance.md) | Senior | BenchmarkDotNet, profiling tools (30 KB) |

> ⚠️ **`performance-fundamentals.md` ≠ `performance.md`** — это разные уровни одной темы.

### Profiling

| Файл | Описание |
|------|----------|
| [`bottleneck-analysis.md`](Middle/bottleneck-analysis.md) | Profiling и поиск узких мест |
| [`memory-profiling.md`](Senior/memory-profiling.md) | dotMemory, PerfView, memory leaks |

### Optimization patterns

| Файл | Описание |
|------|----------|
| [`optimization-patterns.md`](Middle/optimization-patterns.md) | Performance optimization patterns |
| [`async-performance.md`](Middle/async-performance.md) | Async/await performance pitfalls |
| [`lazy-eager-loading.md`](Middle/lazy-eager-loading.md) | Lazy\<T\>, EF Include vs Lazy |

### Caching

| Файл | Описание |
|------|----------|
| [`caching-strategies.md`](Middle/caching-strategies.md) | Cache-aside, write-through, invalidation |

### Capacity & limits

| Файл | Описание |
|------|----------|
| [`capacity-planning.md`](Senior/capacity-planning.md) | Sizing, scaling, capacity math |
| [`performance-budgets.md`](Senior/performance-budgets.md) | SLA, performance budgets |

### Specialized

| Файл | Описание |
|------|----------|
| [`hft-low-latency.md`](Senior/hft-low-latency.md) | HFT, hot paths, < 1 ms (40 KB) ⭐ |
| [`threadpool-starvation-hill-climbing.md`](Senior/threadpool-starvation-hill-climbing.md) | ThreadPool starvation, hill-climbing, bulkhead-паттерн (20 KB) ⭐ NEW |

---

## 🔗 Связанные папки

- [`Runtime/Senior/gc-memory.md`](../Runtime/Senior/gc-memory.md) — GC влияние на perf
- [`Runtime/Senior/span-layout.md`](../Runtime/Senior/span-layout.md) — zero-allocation patterns
- [`EFCore/Senior/queries-performance.md`](../EFCore/Senior/queries-performance.md) — DB perf
- [`SQL/Senior/optimization.md`](../SQL/Senior/optimization.md), [`SQL/Middle/indexes-deep.md`](../SQL/Middle/indexes-deep.md) — DB optimization
- [`AspNetCore/Senior/caching.md`](../AspNetCore/Senior/caching.md) — practical caching в Web
