---
tags: [performance, benchmarkdotnet, profiling, memory-leaks, diagnostics]
level: Senior
---

# .NET Performance и диагностика

## Что это, зачем и когда

### Что такое оптимизация производительности?
**Заставить код работать быстрее и расходовать меньше памяти.** Не угадывать, а измерять — находить узкие места и устранять их.

**Аналогия:** Настройка гоночного автомобиля. Не «красить быстрее», а замерить, где теряется время (шины? двигатель? аэродинамика?) и улучшить именно это.

### Зачем?

| Без оптимизации | С оптимизацией |
|-----------------|---------------|
| API отвечает 2 секунды — пользователи уходят | API отвечает 50мс — быстрый отклик |
| GC паузы по 200мс — приложение «подвисает» | Минимум аллокаций → GC почти не запускается |
| Сервер на 8 ядрах держит 100 RPS | Тот же сервер держит 10 000 RPS |
| «У нас тормозит, но не знаем где» | BenchmarkDotNet + профайлер → точно знаем |

### Когда оптимизировать?

| Ситуация | Действие |
|----------|----------|
| Код ещё не написан | **НЕ оптимизируй** — сначала работающий код |
| Код работает, но медленно | Измерь (BenchmarkDotNet), найди bottleneck, оптимизируй |
| Hot path (вызывается 1000+ раз/сек) | Span, ArrayPool, stackalloc, zero-alloc |
| Cold path (вызывается редко) | **НЕ оптимизируй** — читаемость важнее |
| Утечка памяти | dotnet-counters → dotnet-gcdump → анализ |

**Золотое правило:** «Premature optimization is the root of all evil» — Donald Knuth. Сначала правильно, потом быстро.

---

> [!question]- **Интервью: Как найти утечку памяти в .NET?**
> 1. `dotnet-counters` — мониторинг GC Gen sizes в real-time
> 2. `dotnet-gcdump` — снимок heap для анализа
> 3. `dotnet-dump` + SOS — глубокий анализ (`dumpheap -stat`, `gcroot`)
> 4. dotMemory / PerfView — визуальный анализ
>
> **5 главных причин:** Events (подписка без отписки), Static fields, Closures, Timers, Unmanaged resources.

## Span\<T\> и Memory\<T\>

**Span\<T\>** / **ReadOnlySpan\<T\>** — окно в непрерывную память без копирования. `ref struct` — только на стеке.

```csharp
// Парсинг без аллокаций
ReadOnlySpan<char> line = "2026-02-20;Order;1499.99".AsSpan();
int first = line.IndexOf(';');
int last = line.LastIndexOf(';');

ReadOnlySpan<char> date = line[..first];           // "2026-02-20"
ReadOnlySpan<char> type = line[(first+1)..last];   // "Order"
ReadOnlySpan<char> price = line[(last+1)..];       // "1499.99"

if (decimal.TryParse(price, out var amount))
    Console.WriteLine(amount); // 1499.99 — ноль аллокаций
```

**Memory\<T\>** — как Span, но можно хранить в heap (поля класса, async методы). Используй когда нужно передать буфер через `await`.

```csharp
// Memory в async — Span нельзя
async Task ProcessAsync(Memory<byte> buffer)
{
    await stream.ReadAsync(buffer);
    Span<byte> span = buffer.Span; // Span только в синхронных участках
    Process(span);
}
```

**Нюанс:** `Span<T>` нельзя в async методах, лямбдах, полях класса. Если нужно — используй `Memory<T>`.

---

## ArrayPool\<T\>

Пул массивов — аренда вместо `new byte[]`. Меньше давления на GC.

```csharp
var buffer = ArrayPool<byte>.Shared.Rent(4096);
// ВАЖНО: Rent может вернуть массив БОЛЬШЕ запрошенного
try
{
    int bytesRead = await stream.ReadAsync(buffer.AsMemory(0, 4096));
    ProcessData(buffer.AsSpan(0, bytesRead));
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer, clearArray: true);
    // clearArray: true — обнулить данные (security)
}
```

**Нюанс:** обязательно возвращать в `finally`. Иначе — утечка пула. Не использовать массив после `Return`.

---

## ValueTask vs Task

| Аспект | Task | ValueTask |
|--------|------|-----------|
| Тип | class (heap) | struct (stack) |
| Аллокация | Всегда (кэш для некоторых) | Нет при sync completion |
| Await несколько раз | Да | **Нет** (undefined behavior) |
| Публичный API | Предпочтительно | Осторожно |

```csharp
// ValueTask — когда часто возвращается синхронно (кэш hit)
public ValueTask<Order?> GetCachedAsync(Guid id)
{
    if (_cache.TryGetValue(id, out var order))
        return ValueTask.FromResult<Order?>(order); // Без аллокации

    return GetFromDbAsync(id); // Async path
}
```

**Правило:** `Task` по умолчанию. `ValueTask` — только в hot path с частым sync completion.

---

## stackalloc

Аллокация на стеке — без GC. Быстро, но стек ограничен (~1 MB).

```csharp
// Маленькие буферы — stackalloc
Span<byte> buffer = stackalloc byte[256];
// Большие — ArrayPool
Span<byte> bigBuffer = ArrayPool<byte>.Shared.Rent(8192);

// Паттерн: stackalloc для маленьких, pool для больших
const int StackThreshold = 256;
Span<char> buf = inputLength <= StackThreshold
    ? stackalloc char[inputLength]
    : new char[inputLength]; // или ArrayPool
```

---

## BenchmarkDotNet

Бенчмарки с warmup, статистикой, GC-метриками.

```csharp
[MemoryDiagnoser]       // аллокации
[DisassemblyDiagnoser]  // JIT asm
public class StringBenchmark
{
    private readonly string[] _items = Enumerable.Range(0, 100)
        .Select(i => i.ToString()).ToArray();

    [Benchmark(Baseline = true)]
    public string Concat()
    {
        string result = "";
        foreach (var item in _items)
            result += item;
        return result;
    }

    [Benchmark]
    public string StringBuilder()
    {
        var sb = new StringBuilder();
        foreach (var item in _items)
            sb.Append(item);
        return sb.ToString();
    }

    [Benchmark]
    public string StringJoin()
        => string.Join("", _items);
}

// Запуск: dotnet run -c Release
// BenchmarkRunner.Run<StringBenchmark>();
```

**Нюанс:** всегда запускать в Release. Debug — не показательный.

---

## Чек-лист оптимизации

### Аллокации

- [ ] `StringBuilder` вместо `+=` в циклах
- [ ] `Span<T>` / `stackalloc` вместо `Substring`, `new byte[]`
- [ ] `ArrayPool<T>` для временных буферов
- [ ] `ValueTask` для методов с частым sync return
- [ ] Collection capacity: `new List<T>(capacity)`, `new Dictionary<K,V>(capacity)`
- [ ] Static lambda (`static x => ...`) в hot path — нет closure allocation
- [ ] `string.Create()` вместо интерполяции в hot path

### LINQ

- [ ] `Any()` вместо `Count() > 0`
- [ ] `.Count` (свойство) вместо `.Count()` (метод)
- [ ] Не материализовать раньше времени: `items.Where(...).First()` а не `items.Where(...).ToList().First()`
- [ ] `HashSet<T>` для Contains в циклах вместо `List<T>.Contains` (O(1) vs O(n))
- [ ] `FrozenDictionary` для static lookup таблиц

### EF Core

- [ ] Проекция через `Select()` вместо загрузки целой entity
- [ ] `AsNoTracking()` для read-only запросов
- [ ] Pagination: keyset вместо OFFSET/LIMIT
- [ ] Избегать client-side evaluation (логировать warnings)
- [ ] Split queries для множественных Include

### GC и память

- [ ] `sealed` на классах — JIT devirtualization
- [ ] `readonly struct` — нет defensive copies
- [ ] Object pooling (`ObjectPool<T>`) для дорогих объектов
- [ ] `IDisposable` / `using` — освобождение ресурсов
- [ ] Избегать boxing value types (generic коллекции, `IEquatable<T>`)

---

## Профилирование

| Инструмент | Что измеряет |
|------------|-------------|
| **BenchmarkDotNet** | Микробенчмарки, аллокации |
| **dotnet-counters** | Live-метрики (GC, ThreadPool, HTTP) |
| **dotnet-trace** | CPU profiling, events |
| **dotnet-dump** | Heap dump, анализ памяти |
| **dotnet-gcdump** | GC heap snapshot |
| **PerfView** | Детальный анализ (ETW events) |

```bash
# Live метрики
dotnet-counters monitor --process-id <pid>

# CPU trace
dotnet-trace collect --process-id <pid> --duration 00:00:30

# Heap dump
dotnet-gcdump collect --process-id <pid>
```

---

## См. также

- [[Topics/SQL/sql-query-optimization|SQL Optimization]]
- [[Topics/Observability/opentelemetry-jaeger-seq|OpenTelemetry]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
