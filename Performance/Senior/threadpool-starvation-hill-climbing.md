---
tags: [performance, threadpool, starvation, hill-climbing, async, channels, diagnostics]
level: Senior
date: 2026-06-19
---

# ThreadPool Starvation и Hill-Climbing — почему пул раздувается, а CPU простаивает

> Самый коварный production-инцидент в .NET: throughput падает в разы, p99 уходит в секунды, а CPU при этом 8%. Виноват не пул как таковой, а его алгоритм авто-тюнинга, которого обманывает заблокированный воркер. Здесь — механизм hill-climbing, сигнатура аварии и почему `SetMinThreads` лечит симптом, а не причину.

## 1. Hill-climbing как контур обратной связи

ThreadPool .NET не держит фиксированное число воркеров. Сверх `minThreads` он добавляет потоки через алгоритм **hill-climbing** (восхождение на холм) — это управляющий контур с обратной связью, который ищет такое число потоков, при котором throughput (завершённых work-item в секунду) максимален.

### 1.1 Как работает контур

Алгоритм работает дискретными шагами:

- **Окно измерения** — примерно 200 мс или 32 завершённых сэмпла (что наступит раньше). На каждом окне пул фиксирует две величины: текущее число потоков и достигнутый throughput.
- **Внутренний DFT** — над историей сэмплов считается дискретное преобразование Фурье (discrete Fourier transform). Оно выделяет полезный сигнал «зависимость throughput от числа потоков» из шума случайных колебаний нагрузки. По сути алгоритм спрашивает: «когда я в прошлый раз добавил поток, throughput коррелированно вырос или это случайность?»
- **Управляющее воздействие** — если производная throughput по числу потоков положительна, пул добавляет потоки, но **не более ~4 потоков в секунду** (намеренно медленный темп, чтобы не разогнать систему в осцилляцию). Если добавление потоков throughput не улучшает — пул откатывается назад.

> [!info]
> Темп ~4 потока/сек — ключевая цифра. Она объясняет «лестницу» `ThreadCount` на графике (см. раздел 4) и то, почему деградация при starvation разворачивается за десятки секунд, а не мгновенно.

### 1.2 Почему именно hill-climbing

Оптимальное число потоков заранее неизвестно: оно зависит от соотношения CPU- и I/O-работы, числа ядер, характера нагрузки. Захардкодить нельзя. Hill-climbing — это адаптивный поиск максимума функции `throughput(threadCount)` без знания её формы. Для нормальной (неблокирующей) нагрузки он сходится к разумному числу потоков, близкому к числу ядер для CPU-bound и выше для смешанной нагрузки.

## 2. Почему заблокированный воркер обманывает алгоритм

Контур измеряет throughput как **число завершённых work-item**, а не как полезную работу процессора. В этом и дыра.

Рассмотрим воркер, который вытащил из очереди задачу и внутри неё сделал синхронное ожидание async-операции:

```csharp
// Этот work-item блокирует поток ThreadPool на длительность I/O
public IActionResult GetReport(int id)
{
    // .Result паркует поток пула на всё время сетевого вызова
    var data = _httpClient.GetStringAsync($"/upstream/{id}").Result;
    return Ok(Parse(data));
}
```

Что видит hill-climbing:

1. Воркер взял work-item и в конце концов его **завершил** (вернул результат после разблокировки). Для алгоритма это «+1 к завершённым work-item» — положительный вклад в throughput.
2. Пока воркер стоял на `.Result`, очередь продолжала наполняться новыми запросами, а свободных потоков нет — они все висят на блокировках.
3. Производная throughput по числу потоков оказывается **положительной**: «добавил поток → завершилось больше work-item». Алгоритм делает вывод «мне не хватает потоков» и добавляет ещё ~4/сек.
4. Новые потоки тоже мгновенно встают на `.Result`. Пул раздувается, **CPU стоит почти на нуле** — все потоки спят на блокировках, а не считают.

Это положительная обратная связь в худшем смысле: чем хуже становится (больше блокированных потоков), тем активнее алгоритм добавляет потоки, усугубляя ситуацию. Получается **ThreadPool starvation** — голодание пула: на полезную работу потоков нет, хотя их формально сотни.

> [!warning]
> Корень проблемы — измерение «завершённых work-item» как прокси для «полезного throughput». Блокирующий код ломает это допущение: work-item «завершается», но процессор при этом простаивал. Любой sync-over-async (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`, синхронный `File.ReadAllText` в async-пути, блокирующий `lock` вокруг долгой операции) запускает этот сценарий.

## 3. Production-сигнатура

Авария по starvation узнаётся по характерному набору симптомов. Типичная картина перегруженного блокирующими вызовами сервиса:

| Метрика | Норма | При starvation |
|---------|-------|----------------|
| Throughput | 50k RPS | падает до ~12k RPS |
| p99 latency | 40 мс | взлетает до ~4 s |
| CPU | 40-60% | ~8% (потоки спят) |
| ThreadCount | стабилен | растёт «лестницей» |
| Queue length | ~0 | тысячи pending |

Контринтуитивный маркер: **p99 растёт, пока CPU падает**. В обычной перегрузке (CPU-bound) latency растёт ВМЕСТЕ с CPU. Здесь наоборот — это почти однозначная подпись starvation, а не нехватки железа. Докидывать ядра бесполезно: новые ядра тоже будут простаивать.

## 4. Диагностика через dotnet-counters (dashboard-ready)

Подтверждается starvation за минуту через `dotnet-counters` без рестарта процесса:

```bash
dotnet-counters monitor --process-id <pid> System.Runtime
```

Три объективных сигнала, которые надо вынести на дашборд:

- **ThreadPool Queue Length** (счётчик `ThreadPool Queue Length`, программно `ThreadPool.PendingWorkItemCount`) `>=` 1000, **при том что** CPU `<` 50%. Большая очередь без нагрузки на процессор = потоки заняты ожиданием, а не работой. Это главный сигнал.
- **ThreadPool Thread Count** растёт «лестницей» (ступеньками по ~4/сек — почерк hill-climbing, см. раздел 1.1), вместо того чтобы стабилизироваться.
- **p99 растёт по мере падения CPU** — собирается из вашей APM/OpenTelemetry-гистограммы латентности в паре с `cpu-usage`.

```csharp
// Программная проверка очереди — можно повесить на health-check / метрику
long pending = ThreadPool.PendingWorkItemCount;
ThreadPool.GetAvailableThreads(out int workerAvail, out _);
// pending растёт, а workerAvail около нуля при низком CPU → starvation
```

> [!info]
> Дашбордное правило для алерта: `ThreadPool Queue Length >= 1000 AND cpu-usage < 50%` в течение 30 секунд. Этот составной предикат почти не даёт ложных срабатываний — обычная CPU-перегрузка его не триггерит, потому что там CPU высокий.

## 5. Настоящее лечение — await end-to-end

Единственное корректное исправление — убрать блокировку, сделать путь асинхронным **на всю глубину стека** (async all the way). Тогда воркер не паркуется на I/O: на `await` он возвращается в пул и берёт следующий work-item, а continuation выполнится по готовности I/O. Число занятых потоков перестаёт зависеть от длительности I/O, и hill-climbing снова видит честный throughput.

```csharp
// Тот же endpoint, но поток пула не блокируется на время сетевого вызова
public async Task<IActionResult> GetReportAsync(int id, CancellationToken ct)
{
    var data = await _httpClient.GetStringAsync($"/upstream/{id}", ct);
    return Ok(Parse(data));
}
```

Подробно про механику state machine, `ConfigureAwait`, ловушки sync-over-async — [[async-threading|async/await]] и [[threading-basics|Threading Basics]].

## 6. Bulkhead для неизбежной legacy-блокировки

Иногда блокировку убрать нельзя: синхронный драйвер, COM-объект, нативная библиотека, старый SDK без async API. Тогда задачу **нельзя** отдавать в общий ThreadPool — она заразит hill-climbing тем же образом. Решение — паттерн **bulkhead** (переборка): изолировать блокирующую работу на выделенных потоках, скормив её через `Channel<T>`.

```csharp
// Выделенные long-running потоки + Channel<T> как очередь работ.
// Блокировка живёт ТОЛЬКО здесь и не трогает общий пул.
public sealed class BlockingLegacyBulkhead : IAsyncDisposable
{
    private readonly Channel<WorkItem> _channel =
        Channel.CreateBounded<WorkItem>(new BoundedChannelOptions(1000)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = false,
        });

    private readonly Thread[] _workers;

    public BlockingLegacyBulkhead(int workerCount)
    {
        _workers = new Thread[workerCount];
        for (var i = 0; i < workerCount; i++)
        {
            // LongRunning => отдельный поток, НЕ из ThreadPool
            _workers[i] = new Thread(RunWorkerLoop)
            {
                IsBackground = true,
                Name = $"legacy-bulkhead-{i}",
            };
            _workers[i].Start();
        }
    }

    public ValueTask EnqueueAsync(WorkItem item, CancellationToken ct) =>
        _channel.Writer.WriteAsync(item, ct);

    private void RunWorkerLoop()
    {
        // Синхронный consume: поток выделенный, блокироваться ему можно
        foreach (var item in _channel.Reader.ReadAllAsync().ToBlockingEnumerable())
            item.RunBlockingLegacyCall(); // здесь живёт .Result / sync driver
    }

    public async ValueTask DisposeAsync()
    {
        _channel.Writer.Complete();
        await Task.CompletedTask;
        foreach (var t in _workers) t.Join(TimeSpan.FromSeconds(5));
    }
}
```

Если предпочитаете создавать поток через TPL, используйте `Task.Factory.StartNew(loop, TaskCreationOptions.LongRunning)` — это тоже даёт **отдельный** поток вне пула:

```csharp
// LongRunning подсказывает планировщику выделить dedicated-поток, а не брать из пула
_ = Task.Factory.StartNew(
    RunWorkerLoop,
    CancellationToken.None,
    TaskCreationOptions.LongRunning,
    TaskScheduler.Default);
```

> [!warning]
> **Никогда** не оборачивайте блокирующую legacy-работу в `Task.Run(() => legacy.BlockingCall())`. `Task.Run` уходит в **тот же** общий ThreadPool — вы просто переместили блокировку с одного пулового потока на другой и снова кормите hill-climbing ложным сигналом. Bulkhead работает только потому, что его потоки физически отделены от пула.

## 7. SetMinThreads лечит симптом, а не причину

`ThreadPool.SetMinThreads(n, n)` поднимает нижнюю границу: пул держит `n` потоков всегда, **минуя hill-climbing** на участке `threadCount <= n` (внутри минимума потоки создаются мгновенно, без темпа ~4/сек). Это снимает острый симптом — очередь временно рассасывается, потому что есть запас уже готовых потоков, чтобы поглотить блокировки.

Но это **не лечение**, а дорогая анестезия:

- **Память.** Каждый поток — это стек, по умолчанию ~1 MB. `SetMinThreads(512, 512)` = ~512 MB резерва только под стеки воркеров, ещё до полезной работы.
- **Убит авто-тюнинг.** Пока `threadCount <= n`, hill-climbing не управляет числом потоков. Пул теряет эластичность: под низкой нагрузкой он не сжимается (держит `n`), под высокой — выше `n` снова включается тот же медленный (~4/сек) и легко обманываемый алгоритм. Вы зафиксировали worst-case вместо адаптации.
- **Маскировка.** Поднятый минимум прячет настоящую причину (sync-over-async). Под бОльшей нагрузкой starvation вернётся, и придётся снова крутить число вверх — гонка, которую не выиграть.

> [!info]
> Единственное оправданное применение `SetMinThreads` — **cold-start priming**: дать пулу стартовый запас потоков, чтобы первые секунды после деплоя не упирались в медленный разгон hill-climbing (~4/сек). Разумный ориентир — порядка `Environment.ProcessorCount * 4`, не сотни. Это оптимизация прогрева, а не средство от starvation.

```csharp
// OK: умеренный прайминг на старте, чтобы пережить первый всплеск трафика
int prime = Environment.ProcessorCount * 4;
ThreadPool.SetMinThreads(prime, prime);

// НЕ OK как "лечение" starvation: 512 потоков = ~512 MB стеков и убитый авто-тюнинг
// ThreadPool.SetMinThreads(512, 512);
```

Про прайминг в контексте latency-critical систем — [[hft-low-latency|HFT и Low-Latency]].

## Common Pitfalls

### 1. «Добавим ядер / реплик» при низком CPU

Если CPU 8%, а throughput просел — железо ни при чём. Новые ядра/поды унаследуют тот же блокирующий код и так же будут простаивать. Сначала уберите sync-over-async.

### 2. `Task.Run` вокруг блокирующего вызова как «фикс»

`Task.Run` использует общий пул. Блокировка просто переезжает на другой пуловой поток и продолжает кормить hill-climbing ложным throughput. Нужен bulkhead на выделенных потоках (раздел 6).

### 3. `SetMinThreads(большое_число)` как постоянное лечение

Снимает симптом на текущей нагрузке, жжёт ~1 MB/поток, отключает эластичность и маскирует причину. Под бОльшим трафиком starvation вернётся.

### 4. Диагностика только по CPU

Дашборд, где видна лишь загрузка CPU, пропустит starvation целиком (CPU-то низкий). Обязательно выносите `ThreadPool Queue Length` и `ThreadCount` рядом с CPU.

### 5. Один блокирующий вызов «глубоко в библиотеке»

Достаточно одного sync-over-async на горячем пути, чтобы под нагрузкой обрушить весь сервис. Ищите `.Result`/`.Wait()` не только в своём коде, но и в зависимостях.

## Cheat sheet

| Вопрос | Ответ |
|--------|-------|
| Что измеряет hill-climbing | Завершённые work-item/сек (НЕ полезную загрузку CPU) |
| Окно / темп | ~200 мс или 32 сэмпла; инжект `<=` ~4 потока/сек |
| Как distinguishes сигнал | Внутренний DFT по истории throughput |
| Почему блокировка ломает | Заблокированный поток всё равно «завершает» work-item → ложно-положительная производная |
| Подпись аварии | throughput вниз, p99 вверх, CPU вниз, ThreadCount «лестницей» |
| Главный сигнал на дашборде | `ThreadPool Queue Length >= 1000` при `cpu-usage < 50%` |
| Программная проверка | `ThreadPool.PendingWorkItemCount`, `GetAvailableThreads` |
| Настоящий фикс | `await` end-to-end (async all the way) |
| Неизбежная legacy-блокировка | Bulkhead: dedicated `LongRunning`-потоки + `Channel<T>` |
| Чего НЕ делать | `Task.Run` вокруг блокировки (тот же пул) |
| `SetMinThreads` | Симптом-only; ~1 MB/поток; убивает авто-тюнинг |
| `SetMinThreads` оправдан | Только cold-start priming, ~`ProcessorCount * 4` |

## См. также

- [[async-threading|Async и Threading]] — async/await, sync-over-async, ConfigureAwait
- [[threading-basics|Threading Basics]] — Thread, ThreadPool, TPL, starvation на уровне Middle
- [[hft-low-latency|HFT и Low-Latency]] — SetMinThreads для cold-start, dedicated threads
- [performance](performance.md) — профилирование, dotnet-counters, диагностика
