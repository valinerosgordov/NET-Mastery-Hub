# Performance — производительность

> 12 файлов / ~252 KB. Profiling, optimization, caching, HFT, ThreadPool starvation. От Junior basics до hot path optimization.

[[README|← Главный README]] · [[INDEX|Полный INDEX]]

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Junior, что такое perf | [[performance-fundamentals|`performance-fundamentals.md`]] |
| Senior, нужны tools | [[performance|`performance.md`]] — BenchmarkDotNet, dotMemory |
| Slow application | [[bottleneck-analysis|`bottleneck-analysis.md`]] → [[memory-profiling|`memory-profiling.md`]] |
| Нужен cache | [[caching-strategies|`caching-strategies.md`]] |
| Sub-millisecond latency | [[hft-low-latency|`hft-low-latency.md`]] |
| Throughput падает при низком CPU | [[threadpool-starvation-hill-climbing|`threadpool-starvation-hill-climbing.md`]] |
| Capacity planning | [[capacity-planning|`capacity-planning.md`]], [[performance-budgets|`performance-budgets.md`]] |

---

## 📚 Все 12 файлов

### Fundamentals (Junior to Senior)

| Файл | Уровень | Описание |
|------|---------|----------|
| [[performance-fundamentals|`performance-fundamentals.md`]] | Junior/Middle | Что такое производительность, basics |
| [[performance|`performance.md`]] | Senior | BenchmarkDotNet, profiling tools (30 KB) |

> ⚠️ **`performance-fundamentals.md` ≠ `performance.md`** — это разные уровни одной темы.

### Profiling

| Файл | Описание |
|------|----------|
| [[bottleneck-analysis|`bottleneck-analysis.md`]] | Profiling и поиск узких мест |
| [[memory-profiling|`memory-profiling.md`]] | dotMemory, PerfView, memory leaks |

### Optimization patterns

| Файл | Описание |
|------|----------|
| [[optimization-patterns|`optimization-patterns.md`]] | Performance optimization patterns |
| [[async-performance|`async-performance.md`]] | Async/await performance pitfalls |
| [[lazy-eager-loading|`lazy-eager-loading.md`]] | Lazy\<T\>, EF Include vs Lazy |

### Caching

| Файл | Описание |
|------|----------|
| [[caching-strategies|`caching-strategies.md`]] | Cache-aside, write-through, invalidation |

### Capacity & limits

| Файл | Описание |
|------|----------|
| [[capacity-planning|`capacity-planning.md`]] | Sizing, scaling, capacity math |
| [[performance-budgets|`performance-budgets.md`]] | SLA, performance budgets |

### Specialized

| Файл | Описание |
|------|----------|
| [[hft-low-latency|`hft-low-latency.md`]] | HFT, hot paths, < 1 ms (40 KB) ⭐ |
| [[threadpool-starvation-hill-climbing|`threadpool-starvation-hill-climbing.md`]] | ThreadPool starvation, hill-climbing, bulkhead-паттерн (20 KB) ⭐ NEW |

---

## 🔗 Связанные папки

- [[gc-memory|`Runtime/Senior/gc-memory.md`]] — GC влияние на perf
- [[span-layout|`Runtime/Senior/span-layout.md`]] — zero-allocation patterns
- [[queries-performance|`EFCore/Senior/queries-performance.md`]] — DB perf
- [[optimization|`SQL/Senior/optimization.md`]], [[indexes-deep|`SQL/Middle/indexes-deep.md`]] — DB optimization
- [[caching|`AspNetCore/Senior/caching.md`]] — practical caching в Web
