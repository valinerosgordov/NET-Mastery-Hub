# Performance — производительность

> 11 файлов / 189 KB. Profiling, optimization, caching, HFT. От Junior basics до hot path optimization.

[← Главный README](../README.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Junior, что такое perf | [`performance-fundamentals.md`](performance-fundamentals.md) |
| Senior, нужны tools | [`performance.md`](performance.md) — BenchmarkDotNet, dotMemory |
| Slow application | [`bottleneck-analysis.md`](bottleneck-analysis.md) → [`memory-profiling.md`](memory-profiling.md) |
| Нужен cache | [`caching-strategies.md`](caching-strategies.md) |
| Sub-millisecond latency | [`hft-low-latency.md`](hft-low-latency.md) |
| Capacity planning | [`capacity-planning.md`](capacity-planning.md), [`performance-budgets.md`](performance-budgets.md) |

---

## 📚 Все 11 файлов

### Fundamentals (Junior to Senior)

| Файл | Уровень | Описание |
|------|---------|----------|
| [`performance-fundamentals.md`](performance-fundamentals.md) | Junior/Middle | Что такое производительность, basics |
| [`performance.md`](performance.md) | Senior | BenchmarkDotNet, profiling tools (30 KB) |

> ⚠️ **`performance-fundamentals.md` ≠ `performance.md`** — это разные уровни одной темы.

### Profiling

| Файл | Описание |
|------|----------|
| [`bottleneck-analysis.md`](bottleneck-analysis.md) | Profiling и поиск узких мест |
| [`memory-profiling.md`](memory-profiling.md) | dotMemory, PerfView, memory leaks |

### Optimization patterns

| Файл | Описание |
|------|----------|
| [`optimization-patterns.md`](optimization-patterns.md) | Performance optimization patterns |
| [`async-performance.md`](async-performance.md) | Async/await performance pitfalls |
| [`lazy-eager-loading.md`](lazy-eager-loading.md) | Lazy\<T\>, EF Include vs Lazy |

### Caching

| Файл | Описание |
|------|----------|
| [`caching-strategies.md`](caching-strategies.md) | Cache-aside, write-through, invalidation |

### Capacity & limits

| Файл | Описание |
|------|----------|
| [`capacity-planning.md`](capacity-planning.md) | Sizing, scaling, capacity math |
| [`performance-budgets.md`](performance-budgets.md) | SLA, performance budgets |

### Specialized

| Файл | Описание |
|------|----------|
| [`hft-low-latency.md`](hft-low-latency.md) | HFT, hot paths, < 1 ms (40 KB) ⭐ |

---

## 🔗 Связанные папки

- [`Runtime/gc-memory`](../Runtime/gc-memory.md) — GC влияние на perf
- [`Runtime/span-layout`](../Runtime/span-layout.md) — zero-allocation patterns
- [`EFCore/queries-performance`](../EFCore/queries-performance.md) — DB perf
- [`SQL/optimization`](../SQL/optimization.md), [`SQL/indexes-deep`](../SQL/indexes-deep.md) — DB optimization
- [`AspNetCore/caching`](../AspNetCore/caching.md) — practical caching в Web
