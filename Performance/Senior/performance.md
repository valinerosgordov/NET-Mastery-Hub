---
tags: [performance, benchmarkdotnet, profiling, perfview, dotmemory, dotnet-monitor, eventpipe]
level: Senior
---

# Performance — Benchmarking, Profiling, Memory Leaks

## Что это, зачем и когда

### Что такое performance work?
Workflow из трёх стадий:
1. **Measure** — замерить, что есть сейчас (нет числа = нет проблемы)
2. **Analyze** — найти bottleneck (где тормозит, где аллоцирует)
3. **Optimize** — изменить и **снова замерить** что стало лучше

**Аналогия:** Без замеров оптимизация — как лечить машину "молотком и интуицией". С BenchmarkDotNet/PerfView — как сначала компьютерная диагностика, потом точечный ремонт.

### Главные правила

| Правило | Почему |
|---------|--------|
| Не оптимизируй без замеров | "Я думаю это медленно" — половина проблем в другом месте |
| Оптимизируй hot path | 90% времени в 10% кода (Pareto). Остальное — пустая работа |
| Сначала корректность, потом перформанс | Быстрый неправильный ответ хуже медленного правильного |
| Перформанс — non-functional requirement | Должен быть в acceptance criteria, иначе сполжёт |
| Continuous profiling в проде | Лабораторные тесты не ловят production patterns |

### Когда нужен perf-work

- p99 latency растёт со временем — проблема либо в коде либо в данных (table scan на растущей таблице)
- Memory растёт без падений — leak
- CPU > 80% постоянно — нужна оптимизация горячего пути
- Pre-launch: убедиться что система выдержит prod-нагрузку
- Post-incident: что было причиной, как предотвратить

### Когда **не** нужен

- "Просто так оптимизировать" — потратишь время, ухудшишь читаемость
- На код, выполняющийся раз в день (даже 10 секунд roundtrip OK)
- На non-critical CRUD на 100 RPS (даже EF без оптимизаций тянет)

---

## BenchmarkDotNet — точные замеры

```bash
dotnet add package BenchmarkDotNet
```

### Минимальный benchmark

```csharp
[MemoryDiagnoser]                    // показывает аллокации
[SimpleJob(RuntimeMoniker.Net100)]   // явный фрейм-таргет
public class StringConcatBenchmark
{
    private readonly string[] _items = Enumerable.Range(1, 100).Select(i => $"item-{i}").ToArray();

    [Benchmark(Baseline = true)]
    public string PlusOperator()
    {
        var result = "";
        foreach (var item in _items) result += item;
        return result;
    }

    [Benchmark]
    public string StringBuilder()
    {
        var sb = new StringBuilder();
        foreach (var item in _items) sb.Append(item);
        return sb.ToString();
    }

    [Benchmark]
    public string StringJoin() => string.Join("", _items);

    [Benchmark]
    public string StringConcat() => string.Concat(_items);
}

// Запуск
public static class Program
{
    public static void Main(string[] args) =>
        BenchmarkRunner.Run<StringConcatBenchmark>();
}
```

```bash
dotnet run -c Release --project Benchmarks
```

Output:
```
| Method        | Mean      | Ratio | Allocated |
|-------------- |---------- |------ |---------- |
| PlusOperator  | 12,345 ns | 1.00  | 50,000 B  |
| StringBuilder |    234 ns | 0.02  |    520 B  |
| StringJoin    |    198 ns | 0.02  |    480 B  |
| StringConcat  |    187 ns | 0.02  |    480 B  |
```

### Параметры

```csharp
public class Bench
{
    [Params(10, 100, 1000, 10_000)]
    public int N;

    [GlobalSetup]
    public void Setup()
    {
        _data = Enumerable.Range(0, N).ToArray();
    }

    [Benchmark]
    public int Sum() => _data.Sum();
}
```

`[Params(...)]` — генерирует separate row для каждого значения. Видишь как метод scale'ится.

### Diagnosers

```csharp
[MemoryDiagnoser]                  // Allocations
[ThreadingDiagnoser]               // Lock contention, completed work items
[EventPipeProfiler(EventPipeProfile.CpuSampling)]  // CPU profile
[NativeMemoryProfiler]             // Native memory
[SimpleJob(RunStrategy.Throughput, runtimeMoniker: RuntimeMoniker.Net100)]
public class HeavyBench { /* ... */ }
```

### Job — сравниваем фреймворки

```csharp
[SimpleJob(RuntimeMoniker.Net80)]
[SimpleJob(RuntimeMoniker.Net100)]
public class Bench { /* запустится на обоих ! */ }
```

Полезно для: "стало ли быстрее на новом .NET", "AOT vs JIT".

### Pitfalls

```csharp
// ❌ JIT может выкинуть весь код — return ничего не делает
[Benchmark]
public void NoOp()
{
    var x = 1 + 1;
}

// ✅ Возвращай результат, чтобы JIT не оптимизировал
[Benchmark]
public int Op() => 1 + 1;
```

```csharp
// ❌ Запуск в Debug
dotnet run

// ✅ Release
dotnet run -c Release
// + БЕЗ debugger attached!
```

```csharp
// ❌ Меряешь со state из предыдущей итерации
private List<int> _list = new();

[Benchmark]
public void Add()
{
    _list.Add(1);  // Растёт между iterations
}

// ✅ IterationSetup для очистки
[IterationSetup]
public void Reset() => _list.Clear();
```

> [!question]- **Интервью: как BenchmarkDotNet получает точные замеры?**
> 1. **Pilot** — несколько итераций для stabilize
> 2. **Warmup** — JIT компилит, кэши прогреваются, tiered compilation reaches Tier1
> 3. **Workload** — основные measurements (десятки/сотни прогонов)
> 4. **Statistics** — Mean, StdDev, Percentiles. Outliers удаляются
> 5. **Изоляция** — каждый бенч в отдельном процессе, без cross-contamination
> 6. **Защита от dead code elimination** — return values + `Consume()` через `Consumer` где нужно

---

## Профилирование — какой инструмент когда

```
┌─────────────────────────────────────────────────────────────┐
│                  Какая проблема?                             │
└─────────────────────────────────────────────────────────────┘
        │              │             │              │
   "Slow code"    "Memory leak"  "Production    "Live RAM/CPU"
                                  crash"
        │              │             │              │
        ▼              ▼             ▼              ▼
  dotTrace /     dotMemory /    PerfView /      dotnet-counters
  VS Profiler /  dotnet-       dotnet-dump
  PerfView       gcdump
```

### Сравнительная таблица

| Tool | Когда | Платформа | Сложность |
|------|-------|-----------|-----------|
| **dotnet-counters** | Real-time мониторинг (RAM, GC, threads) | Cross | Easy |
| **dotnet-trace** | CPU profiling без рестарта | Cross | Medium |
| **dotnet-dump** | Memory dump для post-mortem | Cross | Medium |
| **dotnet-gcdump** | Heap snapshot, лёгкий | Cross | Easy |
| **dotnet-monitor** | Continuous production profiling | Cross | Medium |
| **PerfView** | Production crash dumps, ETW deep analysis | Windows | Hard |
| **dotMemory** | Heap analysis с UI | Win/Mac | Easy |
| **dotTrace** | CPU profiling с UI, sampling/tracing | Win/Mac | Easy |
| **VS Profiler** | Dev-time, integrated | Win/Mac | Easy |
| **EventPipe API** | Programmatic access из кода | Cross | Hard |

---

## dotnet-counters — real-time

```bash
dotnet tool install -g dotnet-counters

# Список доступных counters
dotnet-counters list

# Стандартные .NET counters
dotnet-counters monitor --process-id <pid> System.Runtime

# Конкретные counters
dotnet-counters monitor --process-id <pid> \
    System.Runtime[cpu-usage,working-set,gc-heap-size,exception-count] \
    Microsoft.AspNetCore.Hosting[requests-per-second,total-requests]

# Custom Meters (см. observability.md)
dotnet-counters monitor --process-id <pid> MyApp.Orders
```

Output:
```
[System.Runtime]
    CPU Usage (%)                         12
    Working Set (MB)                     245
    GC Heap Size (MB)                    150
    Gen 0 GC / sec                         5
    Exception Count / sec                  0
```

Лёгкий, без stop-the-world, безопасный для production. Используй чтобы быстро понять "это GC overhead или CPU-bound work".

### Привязка к Prometheus

`dotnet-counters` — для ad-hoc. В prod лучше OpenTelemetry → Prometheus (см.[Observability]())).

---

## dotnet-trace — CPU profiling

```bash
dotnet tool install -g dotnet-trace

# Запись 30 секунд CPU traces
dotnet-trace collect --process-id <pid> --duration 00:00:30 --output cpu.nettrace

# С конкретными providers (events)
dotnet-trace collect --process-id <pid> \
    --providers Microsoft-Windows-DotNETRuntime:0x14C14FCCBD:5,\
                Microsoft-DotNETCore-SampleProfiler

# Specific profile
dotnet-trace collect --process-id <pid> --profile cpu-sampling --duration 00:01:00
```

### Анализ результата

```bash
# Конвертировать в формат который понимают разные tools
dotnet-trace convert cpu.nettrace --format Speedscope -o cpu.speedscope.json
# Открыть в speedscope.app

# Или через PerfView
PerfView /noView cpu.nettrace
```

Хорош когда:
- Не можешь attached debugger в production
- Нужен снапшот performance в моменте проблемы
- Хочешь сравнить before/after оптимизации

---

## dotnet-dump + dotnet-gcdump

### dotnet-dump — full process dump

```bash
dotnet tool install -g dotnet-dump

# Снимок всего process state
dotnet-dump collect --process-id <pid> --output dump.dmp

# Анализ
dotnet-dump analyze dump.dmp
> threads          # все managed threads
> clrstack         # stack текущего thread'а
> dumpheap -stat   # group objects by type
> gcroot 0x123...  # кто держит этот объект
> sos              # помощь по SOS commands
```

Heavy operation — process замораживается. Production usage — только при investigation crash.

### dotnet-gcdump — лёгкий heap snapshot

```bash
dotnet tool install -g dotnet-gcdump

dotnet-gcdump collect --process-id <pid>  # → ./<pid>.gcdump

# Анализ — открой в Visual Studio или PerfView

```

Намного легче full dump — содержит только GC heap (managed objects), без stack traces / native heap. Достаточно для большинства memory-проблем.

---

## dotMemory — UI для heap

JetBrains, paid (с trial). Делает то же что gcdump + analyzer, но **намного удобнее**:

- Visual heap snapshot
- Snapshot diff (что добавилось между snapshots)
- Retention path — кто держит объект
- Distribution by type
- Memory traffic — кто аллоцирует
- Inspections — auto-detect leaks

**Workflow для memory leak hunt:**
1. Снять snapshot 1 (baseline)
2. Воспроизвести подозрительную операцию N раз
3. Снять snapshot 2
4. Diff: что выросло? → look at retention path

---

## PerfView — heavy artillery (Windows)

Free Microsoft tool. Самый мощный, но steep learning curve.

```
PerfView.exe
> File → Run → запускаешь приложение
> Файл .etl содержит ETW events
> Stack views → CPU, GC, JIT, Thread Time
```

### Когда PerfView

| Сценарий | Почему |
|----------|--------|
| Production crash dump | Понимает .NET dumps лучше всех |
| GC паузы детально | Видишь каждый GC event с reason |
| Lock contention | Thread Time view + blocked time |
| JIT overhead | Метрики компиляции |
| Memory leak в native interop | ETW + native stacks |

PerfView не для повседневной работы. Open it когда другие tools недостаточно.

---

## dotnet-monitor — continuous profiling в проде

Sidecar инструмент Microsoft. Запускается рядом с приложением, экспонирует HTTP API для diagnostic operations без stop-the-world.

```bash
# Docker
docker pull mcr.microsoft.com/dotnet/monitor:8

# Sidecar pattern в Kubernetes
spec:
  containers:
    - name: app
      image: myapp:latest
    - name: monitor
      image: mcr.microsoft.com/dotnet/monitor:8
      ports: [{ containerPort: 52323 }]
```

### API endpoints

```bash
# Текущая info
GET /info
GET /processes

# Snapshot dump
POST /dump?type=Mini

# CPU profile 30s
GET /trace?durationSeconds=30

# Heap snapshot
GET /gcdump

# Live counters
GET /livemetrics
```

### Триггеры — auto-collect при condition

```json
{
  "CollectionRules": {
    "HighCpu": {
      "Trigger": {
        "Type": "EventCounter",
        "Settings": {
          "ProviderName": "System.Runtime",
          "CounterName": "cpu-usage",
          "GreaterThan": 80,
          "SlidingWindowDuration": "00:00:30"
        }
      },
      "Actions": [
        { "Type": "CollectTrace", "Settings": { "Profile": "Cpu", "Duration": "00:00:30" } }
      ]
    }
  }
}
```

CPU > 80% за 30 секунд → автоматически снимается trace и пишется в S3. Утром investigate.

---

## EventPipe — programmatic profiling

API для inproc trace generation. Использовать когда хочется контроль из кода:

```csharp
using Microsoft.Diagnostics.NETCore.Client;
using Microsoft.Diagnostics.Tracing;
using Microsoft.Diagnostics.Tracing.Parsers;

var providers = new[]
{
    new EventPipeProvider("Microsoft-Windows-DotNETRuntime",
        EventLevel.Informational, (long)ClrTraceEventParser.Keywords.GC)
};

var client = new DiagnosticsClient(processId);
using var session = client.StartEventPipeSession(providers);

await using var output = File.Create("trace.nettrace");
await session.EventStream.CopyToAsync(output);
```

Используют **OpenTelemetry**, **dotnet-trace**, **dotnet-monitor** — все на этом API. Тебе самому редко нужен прямой доступ.

---

## Memory leak hunting workflow

### 1. Detect — есть ли leak?

```bash
# Long-running process
dotnet-counters monitor --process-id <pid> System.Runtime[gc-heap-size,working-set,gc-fragmentation]
```

Симптомы:
- `gc-heap-size` растёт линейно при стабильной нагрузке
- `working-set` растёт быстрее `gc-heap-size` → native leak / unmanaged
- `gen2-gc-count` растёт быстро → объекты живут долго (попадают в Gen2)

### 2. Capture snapshots

```bash
# Baseline
dotnet-gcdump collect --process-id <pid>  # → snapshot1.gcdump

# Воспроизводим подозрительную операцию N раз
curl -X POST .../api/heavy 100 раз

# After
dotnet-gcdump collect --process-id <pid>  # → snapshot2.gcdump
```

### 3. Compare в dotMemory или Visual Studio

Открой оба snapshots → "Compare" → видишь:
- New objects (created between snapshots and still alive)
- Survivors (existed in both but expected to be released)
- Type breakdown — кто растёт быстрее всех

### 4. Retention path — кто держит

В dotMemory: на любом объекте → "Show Roots". Видишь цепочку до GC root:
```
GC root (static field)
  → SomeService (singleton)
    → _eventHandlers (List<EventHandler>)
      → MyComponent.OnDataChanged (delegate)
        → MyComponent (виновник, должен быть собран)
```

### 5. Common leak causes

| Cause | Fix |
|-------|-----|
| `event += handler` без `-=` | `Dispose` отписывает, `WeakEventManager`, `WeakReferenceMessenger` |
| Singleton держит scoped service | Pass dependency как factory, не direct ref |
| `Task` без `await` живёт бесконечно | Always await, use `CancellationToken` |
| Static collection растёт | TTL, eviction policy, periodic cleanup |
| `IDisposable` не задиспоузится | `using`, IDisposable analyzer |
| `IAsyncEnumerable` не закрывается | `await using` for enumerator |
| Long-running `DbContext` накапливает change tracker | Scoped per operation |
| Native handles (file, socket) не освобождаются | SafeHandle pattern |
| Closures захватывают heavy outer variables | Локальные копии, factor out closure |
| LOH fragmentation | ArrayPool вместо new byte[100_000] каждый раз |

### 6. Verify — повторный snapshot

После fix — повтори снимок. Лекажные объекты должны исчезнуть. Если нет — проблема глубже или несколько проблем одновременно.

> [!question]- **Интервью: твой production app медленно жрёт memory. Workflow?**
> 1. **dotnet-counters** проверить growing `gc-heap-size` или `working-set`
> 2. **gcdump** — два snapshot'а с интервалом
> 3. **Compare snapshots** в dotMemory или PerfView
> 4. **Find growing types** — кто увеличивается между snapshots
> 5. **Retention path** — кто держит этот тип
> 6. **Fix** — обычно event handlers, static state, или DbContext lifecycle
> 7. **Verify** — повтор snapshot

---

## CPU profiling workflow

### 1. Sampling vs Tracing

| | Sampling | Tracing |
|--|----------|---------|
| Что | Снимок stack каждые N ms | Каждый method enter/exit |
| Overhead | ~1-5% | 10-50% |
| Точность | Statistical | Exact |
| Когда | Production-safe | Dev-only, hot loops |

`dotnet-trace --profile cpu-sampling` — sampling. dotTrace тоже умеет оба.

### 2. Hot path identification

В dotTrace / Speedscope / PerfView:
- **Inverted call tree** — методы, в которых программа провела больше всего времени
- **Subtree** — расходы внутри конкретного method'а
- **Plain list** — просто отсортированный список по self-time

Ищем:
- Method с самым высоким **self time** — он CPU-bound
- Method с большим **inclusive time** но низким **self** — bottleneck в downstream

### 3. Common CPU hogs

| Causes | Fix |
|--------|-----|
| LINQ в hot loop | Foreach + early break, избегать материализации |
| `string +` в loop | `StringBuilder` или `string.Concat` |
| Boxing value-types | Generic methods, типизированные коллекции |
| Reflection без cache | `MethodInfo` cache, source-gen |
| `JsonSerializer.Serialize<T>` без context | `JsonSerializerContext` |
| Synchronous I/O в async path | Truly async APIs |
| Lock contention | `Channel<T>`, lock-free через `Interlocked` |
| Heavy regex без compilation | `[GeneratedRegex]` source-gen |
| Excessive logging | Source-gen logging, IsEnabled checks |
| Crypto / hashing per request | Cache results, weaker hash где OK |

---

## Profile-Guided Optimization (PGO)

### Что это

JIT собирает данные о горячих путях, использует для re-compile с оптимизациями. **Dynamic PGO** — автоматический в .NET 8+.

```xml
<PropertyGroup>
  <TieredPGO>true</TieredPGO>  <!-- default true в .NET 8+ -->
</PropertyGroup>
```

### Static PGO

Для AOT — собираешь данные на тестовом прогоне, embed'ишь в binary:

```bash
# Собрать profile
dotnet run --configuration Release -- --profile-mode collect

# Build с использованием
dotnet publish --configuration Release /p:PgoOptimization=true
```

Production gain — 5-15% на throughput. Используется в Bing, Office, ASP.NET Core internals.

---

## GC tuning — production tips

```xml
<!-- Сервер GC — почти всегда хочешь -->
<ServerGarbageCollection>true</ServerGarbageCollection>
<ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>
```

См. подробнее в[Runtime / GC and Memory]()) и [HFT / Low-Latency](hft-low-latency.md).

### Latency mode

```csharp
GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency;
// Перед критичной операцией. Не делает Gen2 пока в этом режиме
// Возвращай в Default после
```

### Avoid LOH

`new byte[100_000]` → попадает в Large Object Heap (≥85K). LOH **не уплотняется** → fragmentation.

```csharp
// ❌
var buffer = new byte[200_000];

// ✅ ArrayPool
var buffer = ArrayPool<byte>.Shared.Rent(200_000);
try { /* use */ } finally { ArrayPool<byte>.Shared.Return(buffer); }
```

### POH (Pinned Object Heap, .NET 5+)

```csharp
var buffer = GC.AllocateArray<byte>(4096, pinned: true);
// В POH — не двигается GC, безопасно для native interop
```

---

## Span\<T\> и stackalloc — zero-alloc patterns

```csharp
// ❌ string.Split аллоцирует string[]
var parts = line.Split(';');

// ✅ Span с IndexOf
ReadOnlySpan<char> span = line.AsSpan();
var first = span.IndexOf(';');
var firstField = span[..first];
```

См. подробнее в[Span, Memory, Layout]()) и [HFT / Low-Latency](hft-low-latency.md).

```csharp
// ✅ stackalloc для маленьких temp buffers
Span<char> buffer = stackalloc char[64];
if (value.TryFormat(buffer, out var len, "F2"))
    return new string(buffer[..len]);
```

```csharp
// ✅ ArrayPool для больших variable buffers
var pool = ArrayPool<byte>.Shared;
var buffer = pool.Rent(neededSize);
try { /* use */ } finally { pool.Return(buffer); }
```

---

## Async / await overhead

| Cost | Когда |
|------|-------|
| State machine allocation | Каждый async method (если не synchronous-completion) |
| `Task` allocation | Returns Task |
| Continuation scheduling | На await |
| SyncContext capture | UI/legacy ASP.NET |

### `ValueTask` vs `Task`

```csharp
// Часто завершается синхронно (cache hit) → ValueTask
public ValueTask<int> GetAsync()
{
    if (_cache.TryGet(out var v)) return ValueTask.FromResult(v);
    return GetSlowAsync();
}
```

См. [HFT/Low-Latency](hft-low-latency.md) — детально про async overhead.

### `ConfigureAwait(false)`

В library code — да. В ASP.NET Core (без SyncContext) — необязательно. См.[Async и Threading]()).

---

## Continuous performance в CI

### Benchmark regression на каждый PR

```yaml
# .github/workflows/benchmark.yml
- name: Run benchmarks
  run: dotnet run -c Release --project Benchmarks -- --filter "*HotPath*" --exporters json

- name: Compare to baseline
  uses: benchmark-action/github-action-benchmark@v1
  with:
    tool: 'benchmarkdotnet'
    output-file-path: BenchmarkDotNet.Artifacts/results.json
    fail-on-alert: true
    alert-threshold: '110%'
    comment-on-alert: true
```

Падает PR если медианный latency hot path вырос > 10% — автоматический guard от регрессий.

### Load testing — NBomber / k6

```csharp
// NBomber — на C#, integrates with xunit
var scenario = Scenario.Create("api_load", async ctx =>
{
    var response = await http.GetAsync("/api/users");
    return response.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
})
.WithLoadSimulations(
    Simulation.RampingInject(rate: 100, interval: TimeSpan.FromSeconds(1), during: TimeSpan.FromMinutes(2)),
    Simulation.KeepConstant(copies: 100, during: TimeSpan.FromMinutes(5)));

NBomberRunner.RegisterScenarios(scenario).Run();
```

```bash
# k6 — на JS, очень популярен
k6 run --vus 100 --duration 5m loadtest.js
```

Запускай против staging перед каждым release. Метрики p99 latency и error rate за 5+ минут устойчивой нагрузки.

---

## Production checklist

- [ ] BenchmarkDotNet baseline для critical hot paths
- [ ] Benchmark regression на каждый PR (CI guard)
- [ ] Load testing на staging перед каждым release
- [ ] OpenTelemetry metrics для latency p50/p95/p99 в production
- [ ] dotnet-monitor sidecar для on-demand profiling
- [ ] Auto-collect trace при CPU > 80% sustained
- [ ] Auto-collect gcdump при memory growth
- [ ] ServerGarbageCollection включён
- [ ] ArrayPool вместо `new byte[]` для больших буферов
- [ ] Source-gen logging вместо string interpolation в hot path
- [ ] JsonSerializerContext вместо runtime serialization
- [ ] GeneratedRegex source-gen вместо `new Regex(...)`
- [ ] Compiled queries для часто-вызываемых EF queries
- [ ] AOT для CLI tools и Lambda (5-50ms cold start)
- [ ] Continuous load test раз в неделю — alerting на degradation

---

## Common pitfalls

### 1. Optimization without measurement
"Я думаю это медленно" → переписал → не быстрее → испортил читаемость. Всегда сначала benchmark.

### 2. Optimizing cold path
Метод вызывается раз в день, ты ускорил его в 10 раз → суммарная экономия 0. Профилируй сначала, оптимизируй где hot.

### 3. Premature optimization (Knuth)
"Кто-то когда-то это вызовет миллион раз" → переусложнил архитектуру → теперь и поддержка дороже. YAGNI.

### 4. Benchmarking in Debug
Без `-c Release` цифры в 10-100x хуже реальных. JIT optimizations отключены. Tiered compilation по другому.

### 5. Benchmarking with debugger attached
`dotnet run` под VS-debugger'ом → дополнительные checks → искажение замеров.

### 6. Single-shot timing вместо BenchmarkDotNet

```csharp
var sw = Stopwatch.StartNew();
DoWork();
Console.WriteLine(sw.Elapsed);
```
Один прогон — статистически бессмыслен (warmup, JIT, GC noise). Используй BenchmarkDotNet который делает множественные прогоны.

### 7. EnableSensitiveDataLogging в production
Замедляет EF Core значительно. Только Dev.

### 8. Cargo cult оптимизации
"`StringBuilder` всегда быстрее" — для двух конкатенаций медленнее `+`. "`Span` всегда быстрее" — для маленьких операций overhead. Меряй.

### 9. Не учитываешь GC overhead
Метод "быстрый" сам по себе, но allocates 50KB на вызов → миллион вызовов = частые Gen2 GC pauses → суммарно медленно. `[MemoryDiagnoser]` обязательно.

### 10. Nutrient performance tests
Тесты проверяют корректность ("результат совпадает"), не performance. Performance regressions проскальзывают в production.

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

## Best Practices

### General principles

- **Measure first** — никогда не optimize without data. Profile показывает actual bottleneck, intuition обманчива.
- **Big O matters first** — algorithm complexity > micro-optimizations. O(n²) → O(n log n) даёт 1000x на 1M items, micro-opts — 2x.
- **Allocations matter** — каждый `new` — work для GC. Hot path должен быть allocation-free где возможно.
- **Cache hierarchy** — L1 (1 ns) → L2 (5 ns) → L3 (20 ns) → RAM (100 ns) → SSD (100K ns) → Network (1M ns). Локальность данных critical.

### Workflow

1. **Define performance budget** — что приемлемо? p99 < 100 ms, throughput > 1K RPS
2. **Baseline measure** — где сейчас?
3. **Identify bottleneck** — APM / profiler
4. **Hypothesize** — что ускорит?
5. **Implement minimal change**
6. **Measure** — улучшилось ли? Не стало ли хуже в другом месте?
7. **Document** — почему так, для будущих maintainers

### Optimization techniques (priority order)

1. **Algorithmic** — change Big O
2. **Caching** — repeated work eliminated
3. **Async I/O** — concurrency over parallelism
4. **Batch operations** — reduce round-trips
5. **Lazy / streaming** — process в chunks
6. **Allocations reduction** — Span<T>, pooling
7. **Inline / remove abstraction** — only proven hot path
8. **SIMD / vectorization** — numeric computations
9. **Native / unsafe** — last resort

### Anti-patterns

- ❌ Optimize without profiling
- ❌ Micro-optimize cold code
- ❌ Sacrifice readability для 1% gain
- ❌ Optimize old benchmark (.NET versions меняют performance dramatically)
- ❌ "Faster is always better" — ignoring memory / complexity costs

См.[[memory-pooling|Memory Pooling]],[[types-and-memory|Types & Memory]],[[gc-memory|GC & Memory]].


---

## См. также

- [HFT / Low-Latency](hft-low-latency.md) — глубокое погружение в low-latency .NET
-[Span, Memory, Layout]()) — Span, stackalloc, struct layout
-[GC, LOH, POH]()) — поколения, фрагментация
-[Concurrency и Atomics]()) — lock-free patterns
-[Source Generators]()) — почему source-gen быстрее reflection
-[Native AOT]()) — AOT performance (cold start, memory)
-[EF Core Queries]()) — DB query optimization
-[Observability]()) — production metrics, dotnet-monitor

## Reading list

- **Pro .NET Memory Management** — Konrad Kokosa (canonical book on .NET memory)
- **Pro .NET Performance** — Sasha Goldshtein (older but fundamental)
- **Stephen Toub's blog** — devblogs.microsoft.com/dotnet/author/toub/ (perf patterns from Microsoft)
- **BenchmarkDotNet docs** — benchmarkdotnet.org/articles/overview.html
- **PerfView tutorials** — channel9.msdn.com/Series/PerfView-Tutorial
- **Maoni Stephens blog** — maoni0.medium.com (GC architect at Microsoft)
- **Andrey Akinshin (BenchmarkDotNet author)** — aakinshin.net/posts/
- **JetBrains blog — performance** — blog.jetbrains.com/dotnet/category/performance/
- **dotnet/runtime perf issues** — github.com/dotnet/runtime/labels/area-CodeGen-coreclr
