# Performance — производительность

> 11 файлов / 189 KB. Profiling, optimization, caching, HFT. От Junior basics до hot path optimization.

[← Главный README]() · [Полный INDEX]()

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Junior, что такое perf | [`performance-fundamentals.md`]() |
| Senior, нужны tools | [`performance.md`]() — BenchmarkDotNet, dotMemory |
| Slow application | [`bottleneck-analysis.md`]() → [`memory-profiling.md`]() |
| Нужен cache | [`caching-strategies.md`]() |
| Sub-millisecond latency | [`hft-low-latency.md`]() |
| Capacity planning | [`capacity-planning.md`](), [`performance-budgets.md`]() |

---

## 📚 Все 11 файлов

### Fundamentals (Junior to Senior)

| Файл | Уровень | Описание |
|------|---------|----------|
| [`performance-fundamentals.md`]() | Junior/Middle | Что такое производительность, basics |
| [`performance.md`]() | Senior | BenchmarkDotNet, profiling tools (30 KB) |

> ⚠️ **`performance-fundamentals.md` ≠ `performance.md`** — это разные уровни одной темы.

### Profiling

| Файл | Описание |
|------|----------|
| [`bottleneck-analysis.md`]() | Profiling и поиск узких мест |
| [`memory-profiling.md`]() | dotMemory, PerfView, memory leaks |

### Optimization patterns

| Файл | Описание |
|------|----------|
| [`optimization-patterns.md`]() | Performance optimization patterns |
| [`async-performance.md`]() | Async/await performance pitfalls |
| [`lazy-eager-loading.md`]() | Lazy\<T\>, EF Include vs Lazy |

### Caching

| Файл | Описание |
|------|----------|
| [`caching-strategies.md`]() | Cache-aside, write-through, invalidation |

### Capacity & limits

| Файл | Описание |
|------|----------|
| [`capacity-planning.md`]() | Sizing, scaling, capacity math |
| [`performance-budgets.md`]() | SLA, performance budgets |

### Specialized

| Файл | Описание |
|------|----------|
| [`hft-low-latency.md`]() | HFT, hot paths, < 1 ms (40 KB) ⭐ |

---

## 🔗 Связанные папки

- [`Runtime/gc-memory`]() — GC влияние на perf
- [`Runtime/span-layout`]() — zero-allocation patterns
- [`EFCore/queries-performance`]() — DB perf
- [`SQL/optimization`](), [`SQL/indexes-deep`]() — DB optimization
- [`AspNetCore/caching`]() — practical caching в Web
