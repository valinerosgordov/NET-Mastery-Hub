---
tags: [diagnostics, dotnet-counters, dotnet-trace, dotnet-dump, eventpipe, perfview, troubleshooting, production]
level: Senior
date: 2026-04-30
---

# .NET Diagnostics Tools

> Полный гайд по production troubleshooting в .NET. Закрывает: dotnet-counters/trace/dump/gcdump/monitor, EventPipe, EventListener, ETW, perf, PerfView, dotMemory, dotTrace, integration с Datadog/Grafana, диагностика в Docker/Kubernetes без attach debugger, dotnet-stack, dotnet-symbol.

---

## Что это, зачем и когда

### Что такое diagnostics tools?
**Командлайн-утилиты от Microsoft для observability и troubleshooting** running .NET процессов **без перекомпиляции и без debugger attach** — критично в production.

**Аналогия:** Стетоскоп, тонометр, пульсометр у врача. Без вскрытия пациента можно много диагностировать.

### Зачем?

| Без tools | С tools |
|-----------|---------|
| "App тормозит, перезапусти pod" | `dotnet-counters` — видно CPU/GC/threads метрики live |
| "Memory leak — проверь логи" | `dotnet-gcdump` → анализ heap в dotMemory |
| "API даёт 500" | `dotnet-trace` → видно все методы / SQL / HTTP в trace |
| "Crash random" | `dotnet-dump` → core dump → анализ в Visual Studio |
| "GC pauses" | `dotnet-trace --providers=gc` → точные measurements |
| "Latency спайки" | EventPipe + flame graph |

### Когда что использовать

| Tool | Use case | Output |
|------|----------|--------|
| **dotnet-counters** | Live monitoring metrics (CPU, GC, HTTP, exceptions) | Console / JSON |
| **dotnet-trace** | Profiling — где время / allocations | nettrace / speedscope |
| **dotnet-dump** | Snapshot процесса при crash | core dump → analyze |
| **dotnet-gcdump** | Heap snapshot для memory leaks | gcdump → dotMemory / VS |
| **dotnet-monitor** | Sidecar для k8s — все вышеперечисленное по REST API | HTTP endpoints |
| **dotnet-stack** | Snapshot stack traces всех threads | Console |
| **dotnet-symbol** | Скачать debug symbols (.pdb) | Locally cached |
| **PerfView** | ETW + heap analysis (Windows) | GUI tool |
| **dotTrace / dotMemory** | JetBrains GUI profilers | Cross-platform |

---

## 1. Установка tools

```bash
# Установка как global tools
dotnet tool install --global dotnet-counters
dotnet tool install --global dotnet-trace
dotnet tool install --global dotnet-dump
dotnet tool install --global dotnet-gcdump
dotnet tool install --global dotnet-monitor
dotnet tool install --global dotnet-stack
dotnet tool install --global dotnet-symbol

# Update
dotnet tool update --global dotnet-counters

# В Docker — установить в RUN-stage
RUN dotnet tool install --global dotnet-counters && \
    echo 'export PATH="$PATH:/root/.dotnet/tools"' >> /root/.bashrc
```

---

## 2. dotnet-counters — live metrics

Показывает counters (метрики) running процесса в реальном времени.

### Базовое использование

```bash
# Список процессов
dotnet-counters ps

# Output:
# 12345  myapp  /app/myapp.exe
# 67890  worker /app/worker.dll

# Мониторинг — default counters (CPU, GC, ThreadPool, HTTP)
dotnet-counters monitor --process-id 12345

# Конкретные провайдеры
dotnet-counters monitor --process-id 12345 \
    --counters System.Runtime,Microsoft.AspNetCore.Hosting,Microsoft.AspNetCore.Server.Kestrel
```

### Доступные counter providers

| Provider | Метрики |
|----------|---------|
| `System.Runtime` | CPU, GC, exceptions, threads, allocations |
| `Microsoft.AspNetCore.Hosting` | requests-per-second, total-requests, current-requests, failed-requests |
| `Microsoft.AspNetCore.Server.Kestrel` | connections, queued connections |
| `System.Net.Http` | requests-failed, requests-started |
| `System.Net.NameResolution` | DNS lookups |
| `System.Net.Security` | TLS handshakes |
| `System.Net.Sockets` | bytes sent/received |
| `Microsoft.Data.SqlClient.EventSource` | SQL operations |
| `Npgsql` | Postgres-specific |

### Custom EventCounter из своего кода

```csharp
public class MyMetrics
{
    private readonly Meter _meter = new("MyApp.Orders", "1.0.0");
    private readonly Counter<int> _ordersProcessed;
    private readonly Histogram<double> _orderProcessingTime;

    public MyMetrics()
    {
        _ordersProcessed = _meter.CreateCounter<int>("orders.processed");
        _orderProcessingTime = _meter.CreateHistogram<double>("orders.processing_time_ms");
    }

    public void RecordOrderProcessed(double durationMs)
    {
        _ordersProcessed.Add(1);
        _orderProcessingTime.Record(durationMs);
    }
}

// Регистрация в DI
builder.Services.AddSingleton<MyMetrics>();
```

```bash
# Просмотр
dotnet-counters monitor --process-id 12345 --counters MyApp.Orders
```

### Export в JSON

```bash
dotnet-counters collect --process-id 12345 \
    --counters System.Runtime \
    --output metrics.json \
    --format json
```

### В Docker / Kubernetes

```bash
# В running container
docker exec -it mycontainer dotnet-counters monitor --process-id 1

# В k8s pod
kubectl exec -it mypod -- dotnet-counters monitor --process-id 1
```

> [!info] PID = 1 в контейнере
> Если .NET процесс — entry point контейнера (правильный паттерн), у него PID = 1. Удобно для script-based monitoring.

---

## 3. dotnet-trace — профилирование

Записывает trace всего что происходит в процессе — методы, IO, GC, HTTP, SQL.

### Профилирование

```bash
# Базовый trace (cpu-sampling профиль)
dotnet-trace collect --process-id 12345

# Конкретный duration
dotnet-trace collect --process-id 12345 --duration 00:00:30

# Запуск + trace новый процесс
dotnet-trace collect -- dotnet myapp.dll
```

### Profiles

```bash
# Available profiles
dotnet-trace list-profiles

# cpu-sampling      — CPU usage by method (default)
# gc-verbose        — все GC events
# gc-collect        — лёгкий GC monitoring
# database          — SQL operations

```

```bash
# CPU profile
dotnet-trace collect --process-id 12345 --profile cpu-sampling --duration 00:00:30

# GC profile
dotnet-trace collect --process-id 12345 --profile gc-verbose --duration 00:01:00
```

### Custom providers — что включить

```bash
# Все Microsoft.AspNetCore events
dotnet-trace collect --process-id 12345 \
    --providers "Microsoft-AspNetCore-Server-Kestrel:0xFFFFFFFF:Verbose,Microsoft.AspNetCore.Hosting"

# Verbose level: Critical, Error, Warning, Informational, Verbose, LogAlways
# Keywords: hex flag, всё — 0xFFFFFFFFFFFFFFFF

```

### Анализ trace

#### speedscope.app (browser-based, cross-platform)

```bash
# Convert nettrace → speedscope JSON
dotnet-trace convert trace.nettrace --format speedscope

# Open speedscope.app, drag trace.speedscope.json

```

Flame graph — видно где время тратится. Sandwich view, Left-heavy view.

#### Visual Studio (Windows)

Открыть `.nettrace` файл в VS → анализ CPU/Memory/Allocations.

#### PerfView (Windows, advanced)

Лучший tool для deep analysis ETW + .NET events.

### Trace в production

```bash
# Низкоуровневое профилирование (~5% overhead)
dotnet-trace collect --process-id 12345 \
    --buffersize 256 \
    --duration 00:00:30 \
    --output trace-prod.nettrace
```

> [!warning] Profile production carefully
> `gc-verbose` может добавить 10-20% overhead. CPU-sampling — обычно <5%. Тестируй overhead на staging перед production.

---

## 4. dotnet-dump — core dumps

Snapshot всего процесса для post-mortem analysis (после crash или live).

### Capture dump

```bash
# Live process dump
dotnet-dump collect --process-id 12345

# Output: core_20260430_140000_12345

# Тип dump
dotnet-dump collect --process-id 12345 --type Heap   # все managed objects + threads
dotnet-dump collect --process-id 12345 --type Mini   # минимальный, threads + stacks
dotnet-dump collect --process-id 12345 --type Full   # всё (большой!)
```

### Crash dumps — auto

```bash
# Environment vars для capture при crash
export DOTNET_DbgEnableMiniDump=1
export DOTNET_DbgMiniDumpType=4    # 4 = Heap dump (рекомендуется)
export DOTNET_DbgMiniDumpName=/tmp/dumps/core.%d.%t

# Запуск app — при crash автоматически создаётся dump
dotnet myapp.dll

# В Dockerfile
ENV DOTNET_DbgEnableMiniDump=1
ENV DOTNET_DbgMiniDumpType=4
ENV DOTNET_DbgMiniDumpName=/dumps/core.%d
```

### Анализ dump

```bash
# Open analyzer (interactive shell)
dotnet-dump analyze core_20260430_140000_12345

# Команды (как в WinDbg / SOS):
> threads               # все threads
> clrstack              # managed stack текущего thread
> dumpheap -stat        # статистика heap
> dumpheap -mt <addr>   # все объекты типа
> gcroot <addr>         # кто держит объект (поиск утечки)
> threadpool            # ThreadPool state
> syncblk               # все locks
> exit
```

### Visual Studio для analysis

`File → Open → Crash Dump File`. Полноценный stepping и inspection.

---

## 5. dotnet-gcdump — heap dumps для memory leaks

Только managed heap — компактнее чем full dump (~50-80% меньше).

```bash
# Capture
dotnet-gcdump collect --process-id 12345
# Output: 20260430_140000_12345.gcdump

# Анализ — JetBrains dotMemory или Visual Studio (Diagnostic Tools)

```

### Workflow для memory leak

```
1. dotnet-counters — видно что memory растёт
2. dotnet-gcdump — снимок 1 (baseline)
3. wait 30 min или вызвать suspect operation N раз
4. dotnet-gcdump — снимок 2
5. Открыть оба в dotMemory, compare
6. Видно какой тип объектов растёт → находим утечку
```

### Альтернативно — `GC.GetTotalMemory(forceFullCollection: true)`

```csharp
// Вызвать после полной GC
long totalMemory = GC.GetTotalMemory(forceFullCollection: true);
Console.WriteLine($"Heap: {totalMemory / 1024 / 1024} MB");

// Посмотреть GCMemoryInfo
var info = GC.GetGCMemoryInfo();
Console.WriteLine($"Heap size: {info.HeapSizeBytes / 1024 / 1024} MB");
Console.WriteLine($"Memory load: {info.MemoryLoadBytes / 1024 / 1024} MB");
```

См. [GC и память](gc-memory.md).

---

## 6. dotnet-monitor — sidecar для production

`dotnet-monitor` — REST API для diagnostics. Развёртывается как sidecar в k8s pod рядом с приложением.

### Setup

```bash
# Standalone
dotnet-monitor collect --urls http://localhost:52323
```

### Endpoints

```bash
# List processes
curl http://localhost:52323/processes

# Live metrics
curl http://localhost:52323/livemetrics?pid=12345

# Trace 30 sec
curl http://localhost:52323/trace?pid=12345&durationSeconds=30 -o trace.nettrace

# GC dump
curl http://localhost:52323/gcdump?pid=12345 -o memory.gcdump

# Crash dump
curl http://localhost:52323/dump?pid=12345 -o crash.dmp

# Logs
curl http://localhost:52323/logs?pid=12345&durationSeconds=60 -o logs.txt
```

### Kubernetes sidecar

```yaml
# pod-with-monitor.yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-with-monitor
spec:
  shareProcessNamespace: true   # ← важно для access к процессу
  containers:
    - name: myapp
      image: myapp:latest
      env:
        - name: DOTNET_DiagnosticPorts
          value: "/diag/dotnet-monitor.sock"
      volumeMounts:
        - mountPath: /diag
          name: diagnostics
    
    - name: monitor
      image: mcr.microsoft.com/dotnet/monitor:latest
      args: ["collect", "--urls", "http://+:52323", "--metrics-urls", "http://+:52325"]
      ports:
        - containerPort: 52323
          name: monitor-api
        - containerPort: 52325
          name: prometheus
      volumeMounts:
        - mountPath: /diag
          name: diagnostics
  
  volumes:
    - name: diagnostics
      emptyDir: {}
```

### Configuration

```yaml
# /etc/dotnet-monitor/settings.json
{
  "Authentication": {
    "MonitorApiKey": {
      "Subject": "...",
      "PublicKey": "..."
    }
  },
  "DiagnosticPort": {
    "ConnectionMode": "Listen",
    "EndpointName": "/diag/dotnet-monitor.sock"
  },
  "GlobalCounter": {
    "IntervalSeconds": 5
  },
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

Auto-collect trace при CPU > 80%. Огонь!

См. [Docker — dotnet-monitor sidecar](../Infrastructure/docker.md).

---

## 7. EventPipe и EventListener (in-process)

EventPipe — cross-platform mechanism для emit и consume событий. Используется всеми tools выше.

### EventListener — слушаем события in-process

```csharp
public class GcEventListener : EventListener
{
    protected override void OnEventSourceCreated(EventSource source)
    {
        if (source.Name == "Microsoft-Windows-DotNETRuntime")
        {
            EnableEvents(source, EventLevel.Verbose, (EventKeywords)0x1);  // GC events
        }
    }
    
    protected override void OnEventWritten(EventWrittenEventArgs eventData)
    {
        if (eventData.EventName == "GCStart_V2")
        {
            // Generation, Reason, Type
            int gen = (int)eventData.Payload![1]!;
            uint reason = (uint)eventData.Payload[2]!;
            Console.WriteLine($"GC Gen{gen}, reason={reason}");
        }
    }
}

// Запустить
var listener = new GcEventListener();
// Подписка active до dispose
```

### Custom EventSource

```csharp
[EventSource(Name = "MyApp-OrderProcessing")]
public sealed class OrderEventSource : EventSource
{
    public static readonly OrderEventSource Log = new();
    
    [Event(1, Level = EventLevel.Informational)]
    public void OrderReceived(string orderId, decimal total) =>
        WriteEvent(1, orderId, total);
    
    [Event(2, Level = EventLevel.Warning)]
    public void OrderProcessingSlow(string orderId, double durationMs) =>
        WriteEvent(2, orderId, durationMs);
}

// В коде
OrderEventSource.Log.OrderReceived(order.Id.ToString(), order.Total);

// В dotnet-trace
// dotnet-trace collect --providers MyApp-OrderProcessing
```

### .NET 6+ — System.Diagnostics.Metrics

Modern API для metrics (вместо EventCounter):

```csharp
public class OrderMetrics
{
    private readonly Meter _meter = new("MyApp.Orders", "1.0.0");
    private readonly Counter<int> _ordersCount;
    private readonly Histogram<double> _processingTime;
    
    public OrderMetrics()
    {
        _ordersCount = _meter.CreateCounter<int>("orders.count", description: "Total orders");
        _processingTime = _meter.CreateHistogram<double>("orders.processing_ms");
    }
    
    public void RecordOrder(double processingMs)
    {
        _ordersCount.Add(1);
        _processingTime.Record(processingMs);
    }
}
```

OpenTelemetry автоматически экспортирует эти metrics → Prometheus/Datadog/Jaeger.

См. [Observability](../Infrastructure/observability.md).

---

## 8. dotnet-stack — quick stack snapshot

```bash
# Snapshot всех threads
dotnet-stack report --process-id 12345

# Output:
# Thread (0x12345) "Main thread"
#   System.Threading.Thread.Sleep(int)
#   MyApp.Program.Main()
#
# Thread (0x67890) "ThreadPool worker"
#   System.Net.Sockets.Socket.AcceptAsync(...)
#   ...

```

Полезно для quick check — "что сейчас делают threads". Особенно при hangs / deadlocks.

---

## 9. Troubleshooting workflows

### Workflow 1: High CPU

```bash
# 1. Confirm — counters показывают CPU > 80%
dotnet-counters monitor --process-id 12345

# 2. Profile 30 sec
dotnet-trace collect --process-id 12345 --profile cpu-sampling --duration 00:00:30

# 3. Открыть в speedscope или VS
# Видно какие методы — hot path
# Решение: optimize или async

```

### Workflow 2: Memory leak

```bash
# 1. Counter показывает рост Gen2 size
dotnet-counters monitor --process-id 12345

# 2. Snapshot 1 (baseline)
dotnet-gcdump collect --process-id 12345

# 3. Подождать / выполнить нагрузку
sleep 600

# 4. Snapshot 2
dotnet-gcdump collect --process-id 12345

# 5. Compare в dotMemory:
# - Какие объекты увеличились в количестве?
# - Кто их держит (GC roots)?
# Типичные виновники: events, static collections, ThreadStatic, closures

```

См. [GC — паттерны утечек](gc-memory.md).

### Workflow 3: GC pauses

```bash
# 1. Counter
dotnet-counters monitor --process-id 12345 \
    --counters System.Runtime[gc-fragmentation,gc-heap-size,gc-committed,gen-0-gc-count,gen-1-gc-count,gen-2-gc-count,time-in-gc]

# Видно: % времени в GC > 10% → проблема

# 2. Trace GC events
dotnet-trace collect --process-id 12345 --profile gc-verbose --duration 00:01:00

# 3. Анализ — какие generations, какие reasons (allocation pressure / ephemeral / induced)
# Решения:
# - Server GC если ещё нет
# - Регионы (.NET 7+)
# - Уменьшить allocation rate (Span, ArrayPool)
# - DATAS (.NET 8+)

```

### Workflow 4: Hang / Deadlock

```bash
# 1. Quick check — thread stacks
dotnet-stack report --process-id 12345

# 2. Если вижу threads waiting на lock — full dump
dotnet-dump collect --process-id 12345 --type Heap

# 3. dotnet-dump analyze
> syncblk           # все locks
> threads           # threads + status
> clrstack          # stack каждого thread
> !dlk              # detect deadlocks (требует SOS extension)
```

### Workflow 5: API latency spikes

```bash
# 1. Counters HTTP
dotnet-counters monitor --process-id 12345 --counters Microsoft.AspNetCore.Hosting

# 2. Trace конкретные events
dotnet-trace collect --process-id 12345 \
    --providers "Microsoft-AspNetCore-Hosting,Microsoft-Extensions-Logging" \
    --duration 00:01:00

# 3. Анализ — какие requests медленные, что в timeline (DB, HTTP, GC)
# OpenTelemetry traces дают тот же insight, но distributed

```

См. [Observability](../Infrastructure/observability.md) — OTel + traces для distributed.

### Workflow 6: Crash with no info

```bash
# 1. Включить auto-dump
export DOTNET_DbgEnableMiniDump=1
export DOTNET_DbgMiniDumpType=4
export DOTNET_DbgMiniDumpName=/dumps/core.%d

# 2. Wait for crash
# 3. Analyze
dotnet-dump analyze /dumps/core.12345
> clrstack -all     # stacks всех threads
> printexception    # Exception in failing thread
```

---

## 10. PerfView (Windows, advanced)

Лучший tool для глубокого ETW + .NET analysis. Бесплатный от Microsoft.

[github.com/microsoft/perfview](https://github.com/microsoft/perfview)

### Use cases

- **Memory** — `Heap Snapshot` → `Memory → Take Heap Snapshot`
- **GC** → `Collect → Run` с `.NET CPU + GC Allocation Tick` providers
- **Allocations** — где аллоцируется, кто держит
- **CPU** — flame graph + caller/callee
- **Network** — TCP/HTTP analysis
- **JIT** — какие методы JIT'ятся, сколько занимает

> [!info] Только Windows
> PerfView использует ETW. Для Linux — `dotnet-trace` + `perf` (Linux native).

---

## 11. JetBrains tools

### dotTrace (cross-platform profiler)

Подключается к running процессу или launches new. GUI-based.

Use cases:
- Performance profiling (CPU, allocations, timeline)
- Reveal slow methods
- Compare snapshots

### dotMemory (cross-platform memory profiler)

- Heap snapshots
- Compare heaps (find leaks)
- Object retention analysis
- Generation visualization

### Workflow

```
1. dotnet-counters / OpenTelemetry — alert
2. dotTrace — найти hot method
3. Optimize
4. dotMemory — verify нет утечки
```

---

## 12. Linux-native инструменты

### perf (Linux)

```bash
# CPU sampling
perf record -F 99 -p $(pidof dotnet) -g -- sleep 30
perf script > out.perf

# С Brendan Gregg's flamegraph
git clone https://github.com/brendangregg/FlameGraph
./FlameGraph/stackcollapse-perf.pl out.perf | ./FlameGraph/flamegraph.pl > flame.svg
```

### bpftrace / eBPF

Для очень глубокой диагностики — kernel + userspace. Например, latency histograms всех syscalls.

```bash
# Сколько времени тратится в каждом syscall
bpftrace -e 'tracepoint:syscalls:sys_enter_* { @[probe] = count(); }'
```

### strace

```bash
# Какие syscalls делает процесс
strace -p $(pidof dotnet) -c   # summary statistics
```

---

## 13. Profiling в production без overhead

### Sampling profilers — низкий overhead (<1-5%)

`dotnet-trace --profile cpu-sampling` использует sampling — не трекает каждый вызов, а семплирует threads N раз/сек. Дешевле чем instrumentation.

### Continuous profiling

Datadog Continuous Profiler, Pyroscope, Grafana Phlare — собирают profiles **постоянно** в production с минимальным overhead.

```yaml
# pyroscope-сервер + .NET агент
- name: PYROSCOPE_AGENT_URL
  value: "http://pyroscope:4040"
```

В production видишь flame graph по любому periodу за последние N дней.

### EventPipe over diagnostic socket

```bash
# Process emits events на /tmp/dotnet-diagnostic-PID-...
# dotnet-monitor собирает их → Prometheus

```

---

## 14. Диагностика в Kubernetes — best practices

```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
        - name: app
          image: myapp:latest
          env:
            # Auto-dump on crash
            - name: DOTNET_DbgEnableMiniDump
              value: "1"
            - name: DOTNET_DbgMiniDumpType
              value: "4"  # Heap
            - name: DOTNET_DbgMiniDumpName
              value: "/dumps/core.%d"
          
          volumeMounts:
            - mountPath: /dumps
              name: dumps
            - mountPath: /tmp
              name: diagnostics
          
          # Allow exec для diagnostic
          securityContext:
            capabilities:
              add: ["SYS_PTRACE"]  # для some tools
      
      volumes:
        - name: dumps
          persistentVolumeClaim:
            claimName: crash-dumps-pvc
        - name: diagnostics
          emptyDir: {}
```

```bash
# Ad-hoc diagnostic в running pod
kubectl exec -it mypod -c app -- bash
# В pod:
dotnet-counters monitor --process-id 1
```

Лучше — sidecar `dotnet-monitor` (см. выше).

См. [Kubernetes deep](../Infrastructure/kubernetes.md).

---

## 15. Trace context для distributed systems

В микросервисной архитектуре — diagnostics одного процесса недостаточно. Нужен **distributed tracing**.

```csharp
// Activity API (.NET 5+)
private static readonly ActivitySource ActivitySource = new("MyApp.Orders");

public async Task ProcessOrder(Order order)
{
    using var activity = ActivitySource.StartActivity("ProcessOrder");
    activity?.SetTag("order.id", order.Id);
    activity?.SetTag("order.total", order.Total);
    
    // ... work ...
    
    activity?.SetStatus(ActivityStatusCode.Ok);
}
```

Через OpenTelemetry → Jaeger/Tempo/Zipkin → видно whole request flow через все сервисы.

См. [Observability](../Infrastructure/observability.md).

---

## 16. Common Pitfalls

### 1. PID = 1 в контейнере не работает с docker exec

Если используешь `tini` или `dumb-init` — PID 1 это они, не .NET. Используй `dotnet-counters ps` чтобы найти .NET process.

### 2. Memory dump огромный (full type)

`--type Full` может быть 5+ GB. Для большинства cases достаточно `Heap`.

### 3. `DOTNET_DbgEnableMiniDump` не работает

- Linux only (на Windows — другие настройки)
- Path должен быть writable процессом
- Disk должен иметь достаточно места

### 4. Профилировка — большой overhead

`gc-verbose` = 10-20% overhead. Профилируй на staging или короткими бёрстами в production.

### 5. dotnet-monitor без auth доступен из вне

```yaml
# ❌ Опасно
- name: monitor
  ports:
    - containerPort: 52323
  args: ["--no-auth"]

# ✅ Auth + только cluster-internal
- name: monitor
  args: ["--urls", "http://localhost:52323"]  # localhost only
```

### 6. Trace файлы путают timezone

NetTrace timestamps — UTC. Visualizers могут показывать в local time. Проверь.

### 7. Symbols не загружаются

Для analysis нужны `.pdb` файлы. Если их нет:

```bash
dotnet-symbol --symbols crash.dmp
# Скачивает symbols с Microsoft Symbol Server

```

### 8. PerfView heap не работает на .NET Core

PerfView "Heap Snapshot" работает только с .NET Framework (legacy). Для .NET Core/5+ — `dotnet-gcdump`.

### 9. Запутаться в counters / metrics / events

- **EventCounter** (legacy) — `System.Diagnostics.Tracing` namespace
- **Meter** (.NET 6+) — `System.Diagnostics.Metrics` namespace, OpenTelemetry-friendly
- **EventSource** — events для tracing
- **Counter** vs **Histogram** vs **Gauge** — типы метрик

Для нового кода — `Meter` API.

### 10. dotnet-trace переполняет буфер

Если события идут быстрее чем buffer flush — потеря событий.

```bash
dotnet-trace collect --buffersize 512  # default 256 MB
```

---

## 17. Best Practices

- **Установи tools в Docker image** — не нужно в production install
- **Auto-dump на crash** через `DOTNET_DbgEnableMiniDump=1`
- **dotnet-monitor sidecar** в k8s для on-demand diagnostics
- **OpenTelemetry с самого начала** — distributed tracing намного полезнее когда уже есть проблема
- **Continuous profiling** — Pyroscope/Datadog для production без manual triggering
- **Custom EventSource / Meter** — instrument критичные операции
- **Practice on staging** — отрабатывай workflow до production crisis
- **Document playbooks** — runbook для каждого типа incident
- **Symbols backup** — храни `.pdb` для каждого release (для post-mortem analysis)
- **Avoid println debugging** в production — используй logs + metrics + traces

---

## См. также

- [GC и память](gc-memory.md) — counters для GC, heap dumps анализ
- [Compilation/JIT](compilation-jit.md) — JIT events, tiered compilation observability
- [Concurrency и Atomics](concurrency-atomics.md) — deadlock detection
- [Span и Memory Layout](span-layout.md) — allocation profiling
- [Observability](../Infrastructure/observability.md) — OpenTelemetry, Prometheus, Jaeger
- [Performance](../Performance/performance.md) — BenchmarkDotNet integration
- [Docker](../Infrastructure/docker.md) — dotnet-monitor sidecar
- [Kubernetes](../Infrastructure/kubernetes.md) — k8s diagnostics

## Reading list

- **Microsoft Docs — Diagnostics tools** — learn.microsoft.com/dotnet/core/diagnostics
- **Microsoft Docs — dotnet-monitor** — learn.microsoft.com/dotnet/core/diagnostics/dotnet-monitor
- **Adam Sitnik — Modern .NET diagnostics** — adamsitnik.com
- **Christophe Nasarre — diagnostic blogs** — chnasarre.medium.com
- **Sasha Goldshtein — Production debugging .NET Core** (talks)
- **Konrad Kokosa — Pro .NET Memory Management** (книга)
- **PerfView documentation** — github.com/microsoft/perfview/blob/main/documentation/Documentation.md
- **Brendan Gregg — Linux Performance** — brendangregg.com (perf, eBPF)
- **Pyroscope blog** — pyroscope.io/blog (continuous profiling)
- **JetBrains dotTrace docs** — jetbrains.com/help/profiler
