---
tags: [async, await, task, cancellation, junior]
level: Junior
date: 2026-08-02
---

# async/await — базовый уровень

> **Вход в асинхронность для Junior.** Здесь — рабочая ментальная модель: зачем async существует, что реально делает `await`, как выглядит типовой код и какие четыре ошибки роняют production. Внутренности (state machine, `SynchronizationContext`, `ConfigureAwait`, синхронизация потоков) намеренно вынесены в canonical deep-dive [[async-threading|async-threading]] — этот файл мост к нему, а не замена.

---

## 0. Как читать этот файл

Каждый раздел построен по схеме: **зачем → как выглядит в коде → где ломается**. Если видишь фразу «подробнее в [[async-threading|async-threading]]» — значит здесь дан минимум для правильной картины, а полное объяснение живёт в deep-dive. Дочитай этот файл до конца, попиши async-код пару недель — и только потом иди в deep-dive: он ляжет гораздо ровнее.

Что тебе нужно до этого файла: [[csharp-basics|C# Basics]] (методы, классы, exceptions) и хотя бы шапочное знакомство с [[collections-linq|Collections и LINQ]].

---

## 1. Зачем async — какую проблему он решает

### 1.1. Аналогия: официант

Представь ресторан, где официант принимает заказ, относит его на кухню — и **стоит у плиты, ожидая, пока блюдо приготовится**. Всё это время остальные столики не обслуживаются. Чтобы обслужить 10 столиков, нужно 10 официантов.

Так работает **синхронный** код: поток (официант) отдал запрос к базе данных (кухне) и заблокировался — стоит и ждёт, не делая ничего полезного.

Асинхронный официант поступает иначе: отнёс заказ на кухню, **вернулся в зал и обслуживает другие столики**. Когда кухня звонит в колокольчик «готово» — любой свободный официант забирает блюдо и несёт его на нужный столик. 10 столиков обслуживают 2 официанта.

`async`/`await` — это и есть «вернуться в зал»: пока I/O-операция (БД, HTTP, диск) выполняется *вне* твоего процесса, поток не стоит у плиты, а обслуживает другие запросы.

### 1.2. Что такое «блокирующий» вызов — и почему это дорого

```csharp
// СИНХРОННО: поток заблокирован на всё время запроса
using var client = new HttpClient();
string html = client.GetStringAsync("https://example.com").Result; // ← НЕ ДЕЛАЙ ТАК (§6.1)
Console.WriteLine(html.Length);
```

Пока сервер отвечает (допустим, 300 мс), поток **ничего не делает** — он просто занят. В консольной утилите это незаметно. В web-приложении — фатально:

- ASP.NET Core обслуживает запросы потоками из **thread pool** — их не бесконечно много (стартово ≈ по числу ядер CPU).
- Каждый заблокированный поток = минус один обработчик для остальных запросов.
- 100 одновременных запросов, каждый блокирует поток на 300 мс → пул исчерпан, новые запросы стоят в очереди, latency растёт лавинообразно. Это называется **thread pool starvation**.

Асинхронная версия отпускает поток на время ожидания:

```csharp
// АСИНХРОННО: на время сетевого ожидания поток возвращается в пул
using var client = new HttpClient();
string html = await client.GetStringAsync("https://example.com");
Console.WriteLine(html.Length);
```

Тот же поток, пока сервер думает, обрабатывает другие запросы. Поэтому async-сервер на тех же ресурсах держит на порядки больше одновременных подключений — не потому что «async быстрее» (один запрос выполняется столько же), а потому что **потоки не простаивают**.

> [!info]- «А кто тогда ждёт ответ, если поток ушёл?»
> Никто. Настоящий сетевой/дисковый I/O выполняет операционная система и железо (сетевая карта, контроллер диска) — без участия .NET-потока. Когда ОС сигнализирует «данные пришли», планировщик .NET берёт свободный поток из пула и продолжает твой метод. Знаменитая формулировка Stephen Cleary: *«There is no thread»*. Механика — в [[async-threading|async-threading]].

### 1.3. Где Junior встречает async в первый же день

Async — не «продвинутая тема на потом». Все современные I/O-API в .NET асинхронные, и ты столкнёшься с ними в первую неделю:

| Область | API | Файл vault'а |
|---------|-----|--------------|
| HTTP-запросы | `HttpClient.GetAsync`, `PostAsJsonAsync` | [[http-client-resilience|HttpClient Resilience]] |
| База данных | EF Core: `ToListAsync`, `SaveChangesAsync` | [[ef-basics|EF Basics]] |
| Файлы | `File.ReadAllTextAsync`, `Stream.ReadAsync` | [[io-streams|IO Streams]] |
| Web-endpoint'ы | handler'ы Minimal API / контроллеров возвращают `Task<...>` | AspNetCore-раздел |
| Пауза | `Task.Delay` (не `Thread.Sleep`, §6.5) | — |

Правило простое: **если операция ходит за пределы процесса (сеть, диск, БД) — она асинхронная**. Синхронные двойники (`File.ReadAllText`, `client.Send`) существуют для legacy и скриптов, в серверном коде им не место.

---

## 2. `Task` и `Task<T>` — ментальная модель

### 2.1. «Обещание результата»

`Task<T>` — это **обещание, что результат типа `T` появится позже**. Не сам результат, а квитанция на него (в JS это называется Promise — модель та же):

```csharp
using var client = new HttpClient();

// GetStringAsync вернул управление СРАЗУ — скачивание ещё идёт
Task<string> promise = client.GetStringAsync("https://example.com");

Console.WriteLine("Запрос отправлен, можно делать что-то ещё...");

// await = «когда обещание исполнится — дай мне результат»
string html = await promise;
Console.WriteLine($"Готово: {html.Length} символов");
```

Состояния у задачи три (упрощённо): **выполняется** → **завершилась успешно** (результат готов) / **завершилась с ошибкой** (внутри лежит exception, §3.3) / **отменена** (§5).

- `Task<T>` — обещание значения: `Task<string>`, `Task<List<Order>>`.
- `Task` (без `T`) — обещание, что **работа завершится**, без значения. Асинхронный аналог `void`: `SaveChangesAsync` возвращает `Task` — «сохранение когда-нибудь закончится».

### 2.2. async-метод — как это пишется

```csharp
public sealed class PageSizeService(HttpClient client)
{
    // async в сигнатуре + Task<T> в возвращаемом типе + await внутри
    public async Task<int> GetPageSizeAsync(string url)
    {
        string html = await client.GetStringAsync(url);
        return html.Length; // пишешь return int — компилятор сам завернёт в Task<int>
    }
}
```

Три вещи, которые сбивают новичка:

1. **`async` — деталь реализации, а не часть контракта.** Модификатор лишь разрешает использовать `await` внутри метода. Вызывающий видит только `Task<int>`.
2. **`return html.Length`, а не `return Task.FromResult(html.Length)`.** Внутри async-метода возвращаешь голое значение — упаковку в `Task` делает компилятор.
3. **Суффикс `Async` — конвенция имени** для всех методов, возвращающих `Task`/`Task<T>`. Не опускай его: по имени вызова видно, что метод нужно await'ить.

### 2.3. async Main и top-level statements

Точка входа тоже умеет в async — иначе нельзя было бы await'ить в консольных утилитах:

```csharp
// Top-level statements (современный стиль): await прямо в Program.cs
using var client = new HttpClient();
string status = await client.GetStringAsync("https://example.com");
Console.WriteLine(status.Length);
```

В старом коде встретишь эквивалент с явной точкой входа — `private static async Task Main()`: сигнатура `async Task` вместо `void` разрешает await внутри `Main`.

> [!question]- Интервью: что такое `Task<T>` своими словами?
> «Объект-обещание: операция запущена, результат типа `T` появится позже. У задачи есть состояние (выполняется / успех / ошибка / отменена), результат забирается через `await`. `Task` без `T` — то же самое, но без значения, только факт завершения». Если хочешь добавить глубины: exception внутри задачи хранится до момента await — поэтому забытый await проглатывает ошибки (§6.3).

---

## 3. await — что происходит на самом деле

### 3.1. На паузу ставится метод, а не поток

Ключевая фраза, которую стоит выучить дословно: **`await` приостанавливает метод, но не блокирует поток**.

```csharp
public sealed class ReportService(HttpClient client)
{
    public async Task<string> BuildReportAsync()
    {
        Console.WriteLine("Шаг 1: до await");           // выполняет поток A

        string data = await client.GetStringAsync("https://example.com");
        // ← здесь метод «замораживается», поток A уходит работать дальше

        Console.WriteLine("Шаг 2: после await");         // выполняет поток из пула (возможно, другой)
        return $"Report: {data.Length} bytes";
    }
}
```

Что происходит в точке `await`, по шагам:

1. Началась операция (`GetStringAsync` отправил запрос). Если она **уже завершена** (данные в кэше, файл маленький) — метод просто продолжается синхронно, никакой магии.
2. Если нет — метод **возвращает управление вызывающему** (тот получает незавершённый `Task`), а текущий поток освобождается: в web-приложении уходит обслуживать другие запросы.
3. Когда операция завершилась, **продолжение метода** (всё, что после `await`) выполняется на свободном потоке из thread pool.

Из пункта 3 следствие, о которое спотыкаются: код до и после `await` может выполняться **разными потоками**. Для обычного backend-кода это неважно (ты не привязан к потоку). Важно это становится в UI-приложениях и при работе с thread-affinity — разбор в [[async-threading|async-threading]].

> Как именно компилятор «замораживает» метод (перепись в state machine), кто решает, на каком потоке продолжить (`SynchronizationContext`, `TaskScheduler`), и что делает `ConfigureAwait(false)` — это уровень Middle/Senior: [[async-threading|async-threading]] §3. Для написания корректного кода достаточно модели выше.

### 3.2. await — это ещё и распаковка результата

```csharp
using var client = new HttpClient();

Task<string> task = client.GetStringAsync("https://example.com");
string html = await task;   // await «снимает» Task<string> → string

int length = html.Length;   // дальше работаешь с обычным значением
Console.WriteLine(length);
```

Поэтому async-цепочки читаются линейно, сверху вниз, как синхронный код: «скачал профиль → достал URL аватарки → скачал аватарку» — три `await` подряд, без единого callback'а. В этом главная победа async/await над callback-стилем, который до C# 5 был единственным вариантом.

### 3.3. Исключения проходят как обычно

Если внутри асинхронной операции случился exception, он сохраняется в `Task` и **перебрасывается в точке `await`**. Для тебя это выглядит как обычный try/catch:

```csharp
using var client = new HttpClient();

try
{
    string html = await client.GetStringAsync("https://invalid.example.invalid");
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"Сеть недоступна: {ex.Message}");
}
```

Ловится, пробрасывается, логируется — всё как в синхронном коде. Ломается эта простота только в двух случаях: забытый `await` (§6.3) и `async void` (§6.2). Общая стратегия работы с ошибками — [[error-handling|Error Handling]].

> [!question]- Интервью: «await блокирует поток?»
> Нет — и это главный вопрос-фильтр по теме. `await` приостанавливает **метод**: если операция не завершена, метод возвращает управление вызывающему, а поток освобождается (в сервере — уходит обслуживать другие запросы). Продолжение метода после завершения операции выполнит поток из thread pool — возможно, другой. Блокирует поток как раз `.Result`/`.Wait()` — то, чем await не является.

---

## 4. Типовой код — что ты будешь писать каждый день

### 4.1. HTTP-запрос

```csharp
public sealed record WeatherDto(string City, double TemperatureC);

public sealed class WeatherService(HttpClient client)
{
    public async Task<WeatherDto?> GetWeatherAsync(string city, CancellationToken ct)
    {
        // GetFromJsonAsync: запрос + десериализация JSON одним вызовом
        return await client.GetFromJsonAsync<WeatherDto>(
            $"https://api.example.com/weather/{city}", ct);
    }
}
```

Заметь: `HttpClient` приходит через конструктор (DI + `IHttpClientFactory`), а не создаётся `new` на каждый вызов — почему это важно и как настроить, см. [[http-client-resilience|HttpClient Resilience]].

### 4.2. База данных — EF Core

```csharp
using Microsoft.EntityFrameworkCore;

public sealed class Order
{
    public Guid Id { get; init; }
    public decimal Total { get; set; }
    public bool IsPaid { get; set; }
}

public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
}

public sealed class OrderService(AppDbContext db)
{
    public async Task<List<Order>> GetUnpaidAsync(CancellationToken ct)
    {
        return await db.Orders
            .Where(o => !o.IsPaid)
            .ToListAsync(ct);          // запрос уходит в БД здесь
    }

    public async Task MarkPaidAsync(Guid orderId, CancellationToken ct)
    {
        Order? order = await db.Orders.FindAsync([orderId], ct);
        if (order is null) return;

        order.IsPaid = true;
        await db.SaveChangesAsync(ct); // INSERT/UPDATE уходят в БД здесь
    }
}
```

Всё, что реально ходит в базу, — асинхронное: `ToListAsync`, `FirstOrDefaultAsync`, `SaveChangesAsync`. Основы EF — [[ef-basics|EF Basics]].

### 4.3. Файлы

```csharp
string path = Path.Combine(Path.GetTempPath(), "notes.txt");

await File.WriteAllTextAsync(path, "первая строка");
string content = await File.ReadAllTextAsync(path);
Console.WriteLine(content);
```

### 4.4. Последовательно vs параллельно — `Task.WhenAll`

Если операции **не зависят друг от друга**, await'ить их по очереди — значит складывать их время:

```csharp
using var client = new HttpClient();

// ПОСЛЕДОВАТЕЛЬНО: ~300мс + ~300мс = ~600мс
string first = await client.GetStringAsync("https://example.com/a");
string second = await client.GetStringAsync("https://example.com/b");
Console.WriteLine(first.Length + second.Length);
```

```csharp
using var client = new HttpClient();

// ПАРАЛЛЕЛЬНО: обе операции запущены ДО первого await → ~300мс суммарно
Task<string> firstTask = client.GetStringAsync("https://example.com/a");
Task<string> secondTask = client.GetStringAsync("https://example.com/b");

string[] pages = await Task.WhenAll(firstTask, secondTask);
Console.WriteLine(pages[0].Length + pages[1].Length);
```

Механика: задача **запускается в момент вызова метода**, а не в момент `await`. Вызвал оба метода подряд — обе операции уже в полёте; `Task.WhenAll` ждёт завершения всех и отдаёт массив результатов.

> [!warning] EF Core: не параллель на одном DbContext
> `DbContext` не потокобезопасен — **два одновременных запроса через один контекст запрещены** (`InvalidOperationException: A second operation was started...`). `Task.WhenAll` годится для HTTP, файлов, независимых сервисов, но два запроса к БД через один и тот же `db` выполняй последовательно (или через отдельные контексты — Middle-тема).

---

## 5. `CancellationToken` — почему всегда

### 5.1. Зачем

Пользователь закрыл вкладку, клиент оборвал HTTP-запрос, приложение останавливается — а твой метод продолжает крутить запрос к БД на 30 секунд, занимая соединение и CPU впустую. `CancellationToken` — это способ сказать всей цепочке вызовов: «результат больше никому не нужен, сворачивайтесь».

Правило vault'а (и любого серьёзного code review): **каждый async-метод принимает `CancellationToken` и пробрасывает его дальше**. Не «когда понадобится» — всегда: добавить параметр потом означает поменять все сигнатуры цепочки.

### 5.2. Как пробрасывать

Токен ты обычно **не создаёшь, а передаёшь насквозь** — из аргумента метода в каждый вызываемый async-API:

```csharp
public sealed class ExportService(HttpClient client)
{
    public async Task ExportAsync(string[] ids, CancellationToken ct)   // 1. принял
    {
        foreach (string id in ids)
        {
            string json = await client.GetStringAsync(
                $"https://api.example.com/items/{id}", ct);             // 2. пробросил

            await File.AppendAllTextAsync("export.jsonl", json + '\n', ct); // 3. пробросил
        }
    }
}
```

Отмена работает кооперативно: когда токен «сработал», ближайший await'нутый API бросает `OperationCanceledException`, и цепочка разматывается вверх. Это **штатное** исключение — глобальный обработчик обычно просто не логирует его как ошибку.

### 5.3. Откуда токен берётся

```csharp
// Ручное создание — например, таймаут в консольной утилите:
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
using var client = new HttpClient();

try
{
    string html = await client.GetStringAsync("https://example.com", cts.Token);
    Console.WriteLine(html.Length);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Не уложились в 10 секунд — отменено.");
}
```

В ASP.NET Core создавать ничего не нужно: объяви параметр `CancellationToken ct` в handler'е endpoint'а — фреймворк подставит токен, который сработает при обрыве соединения клиентом. Твоя работа — только пробросить его вниз.

> [!question]- Интервью: зачем каждому async-методу `CancellationToken`?
> Чтобы прекращать работу, результат которой уже никому не нужен: клиент оборвал запрос, истёк таймаут, приложение останавливается (graceful shutdown). Без токена запрос к БД доработает до конца, впустую держа соединение и поток. Отмена кооперативная: токен пробрасывается по всей цепочке, сработал — ближайший API бросает `OperationCanceledException`. Добавлять параметр «потом» дорого — меняются все сигнатуры, поэтому токен принимают сразу.

---

## 6. Смертные грехи — с механизмом

Четыре ошибки, за которые бьют на code review. Важно понимать не «так нельзя», а **почему именно ломается**.

### 6.1. `.Result` / `.Wait()` — блокировка на задаче

```csharp
public sealed class BadReportService(HttpClient client)
{
    public string BuildReport()
    {
        // ГРЕХ: синхронно ждём асинхронную операцию
        string data = client.GetStringAsync("https://example.com").Result;
        return data[..100];
    }
}
```

Механизм поломки — два сценария:

- **Deadlock (UI, старый ASP.NET).** Поток с «контекстом» (UI-поток) блокируется в `.Result` и ждёт задачу. Задача завершилась, но её продолжение должно выполниться **на том же потоке** — а он занят ожиданием. Оба ждут друг друга вечно. Классический разбор — Stephen Cleary, *Don't Block on Async Code* (reading list).
- **Thread pool starvation (ASP.NET Core).** Дедлока нет (контекста нет), но каждый `.Result` держит поток заблокированным — ровно та проблема из §1.2, ради которой async и существует. Под нагрузкой пул исчерпывается, сервис деградирует. Это называют **sync-over-async**.

Лечение всегда одно — асинхронность «заразна» вверх по стеку, прими это:

```csharp
public sealed class GoodReportService(HttpClient client)
{
    public async Task<string> BuildReportAsync(CancellationToken ct)
    {
        string data = await client.GetStringAsync("https://example.com", ct);
        return data[..100];
    }
}
```

> [!question]- Интервью: почему `.Result` может привести к deadlock?
> `.Result` блокирует текущий поток до завершения задачи. Если код выполняется в контексте, который требует продолжений на этом же потоке (UI-поток WPF/WinForms, legacy ASP.NET), продолжение задачи встаёт в очередь к заблокированному потоку — задача не может завершиться, поток не может разблокироваться, взаимное ожидание. В ASP.NET Core контекста нет и дедлока не будет, но `.Result` всё равно вреден: это sync-over-async, под нагрузкой выжирающий thread pool.

### 6.2. `async void` — исключения без страховки

```csharp
public sealed class Mailer
{
    // ГРЕХ: async void
    public async void SendWelcomeEmail(string email)
    {
        await Task.Delay(100);                     // имитация отправки
        throw new InvalidOperationException("SMTP down");
        // ← это исключение НЕКОМУ поймать: нет Task, в котором оно бы сохранилось.
        //   Оно всплывает как необработанное и РОНЯЕТ ПРОЦЕСС.
    }
}
```

Механизм: исключение из async-метода сохраняется в возвращаемом `Task` и ждёт своего `await` (§3.3). У `async void` возвращаемого `Task` **нет** — исключению негде жить, вызывающий не может ни await'ить метод, ни поймать ошибку, ни узнать о завершении. Правильно:

```csharp
public sealed class SafeMailer
{
    // Task вместо void — вызывающий может await и поймать исключение
    public async Task SendWelcomeEmailAsync(string email, CancellationToken ct)
    {
        await Task.Delay(100, ct);
        // ... отправка
    }
}
```

Единственное легитимное место `async void` — **event handler'ы** (WPF/WinForms `button.Click += async (s, e) => ...`), потому что сигнатура события требует `void`. Там весь код оборачивают в try/catch целиком.

### 6.3. Забытый `await` — проглоченное исключение

```csharp
public sealed class AuditService(HttpClient client)
{
    public async Task ProcessAsync(CancellationToken ct)
    {
        // ГРЕХ: вызвали async-метод и не await'нули (компилятор предупредит: CS4014)
        LogToRemoteAsync("processing started", ct);

        // ... основная работа продолжается, НЕ дождавшись логирования
        await Task.Delay(500, ct);
    }

    private async Task LogToRemoteAsync(string message, CancellationToken ct)
    {
        await client.PostAsync($"https://log.example.com?m={message}", content: null, ct);
        // если здесь будет exception — он останется внутри забытого Task и исчезнет молча
    }
}
```

Что ломается: метод продолжает выполняться, **не дождавшись** операции (гонки: «почему лог пустой?», «почему запись в БД иногда не успевает?»), а исключение из незавершённой задачи **никогда не перебросится** — некому await'ить. Ошибки исчезают бесследно. Компилятор помогает — warning **CS4014** «call is not awaited»; в vault-проектах он поднят до ошибки. Фикс — одно слово: `await LogToRemoteAsync(...)`.

### 6.4. `await` в цикле — когда можно `WhenAll`

```csharp
using var client = new HttpClient();
string[] urls = ["https://example.com/1", "https://example.com/2", "https://example.com/3"];

// МЕДЛЕННО: каждая итерация ждёт предыдущую → время = сумма всех запросов
var slowResults = new List<string>();
foreach (string url in urls)
{
    slowResults.Add(await client.GetStringAsync(url));
}
Console.WriteLine(slowResults.Count);
```

```csharp
using var client = new HttpClient();
string[] urls = ["https://example.com/1", "https://example.com/2", "https://example.com/3"];

// БЫСТРО: все запросы стартуют сразу, ждём один раз
IEnumerable<Task<string>> downloadTasks = urls.Select(url => client.GetStringAsync(url));
string[] fastResults = await Task.WhenAll(downloadTasks);
Console.WriteLine(fastResults.Length);
```

Это не всегда грех: если итерации **зависят друг от друга** (следующий запрос строится на ответе предыдущего) или ресурс не терпит параллели (один `DbContext`, §4.4) — цикл с await корректен. Грех — последовательное ожидание **независимых** операций. Для сотен/тысяч задач параллель ограничивают (semaphore, `Parallel.ForEachAsync`) — это уже [[async-threading|async-threading]] §5.

### 6.5. Бонусный грех: `Thread.Sleep` вместо `Task.Delay`

```csharp
// ГРЕХ в async-коде: блокирует поток — перечёркивает смысл async
// Thread.Sleep(1000);

// Правильно: «пауза без потока»
await Task.Delay(TimeSpan.FromSeconds(1));
Console.WriteLine("прошла секунда, поток всё это время работал на других задачах");
```

`Thread.Sleep` усыпляет поток (официант спит в зале), `Task.Delay` — это таймер: поток свободен, метод продолжится по истечении времени.

---

## 7. `ValueTask` — увидишь в сигнатурах

В BCL и библиотеках тебе встретится `ValueTask<T>` вместо `Task<T>` (например, `FindAsync` в EF Core, `ReadAsync` у каналов). Для потребителя разница нулевая: **так же await'ишь, и всё**. Это оптимизация аллокаций для методов, которые часто завершаются синхронно; зачем она и какие у неё ограничения (нельзя await'ить дважды) — [[async-threading|async-threading]] §3.5. Писать свои `ValueTask`-методы на Junior-уровне не нужно.

---

## 8. Decision tree — когда метод должен быть async

```
Метод делает I/O (БД, HTTP, файлы, очереди)?
  │
  ├── Да → async Task / async Task<T>, суффикс Async,
  │        CancellationToken параметром, await всех async-вызовов
  │
  ├── Нет, но вызывает хотя бы один async-метод
  │      → тоже async Task (асинхронность заразна вверх по стеку;
  │        НЕ «гасить» её через .Result — §6.1)
  │
  └── Нет, чистые вычисления в памяти (маппинг, валидация, математика)
         → обычный синхронный метод.
           НЕ заворачивай в Task.Run «чтобы было асинхронно» —
           это просто занимает второй поток без выгоды (детали: async-threading §5)
```

И выбор внутри async-метода:

```
Несколько async-операций подряд?
  │
  ├── Результат первой нужен второй → последовательные await
  │
  ├── Независимы, разные ресурсы (HTTP, файлы) → Task.WhenAll
  │
  └── Независимы, но общий DbContext → последовательные await (§4.4)
```

---

## 9. Cheat sheet

### Сигнатуры

```csharp
public sealed class CheatSheetService(HttpClient client)
{
    // Возвращает значение
    public async Task<string> LoadAsync(CancellationToken ct) =>
        await client.GetStringAsync("https://example.com", ct);

    // Не возвращает значение (аналог void)
    public async Task NotifyAsync(CancellationToken ct) =>
        await client.PostAsync("https://example.com/ping", content: null, ct);

    // async void — ТОЛЬКО event handlers, больше нигде
}
```

### Вызовы, отмена, ошибки

```csharp
using var client = new HttpClient();
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30)); // таймаут-токен

try
{
    // Обычный вызов
    string page = await client.GetStringAsync("https://example.com", cts.Token);

    // Параллельно (независимые операции)
    Task<string> taskA = client.GetStringAsync("https://example.com/a", cts.Token);
    Task<string> taskB = client.GetStringAsync("https://example.com/b", cts.Token);
    string[] both = await Task.WhenAll(taskA, taskB);

    // Пауза без блокировки потока
    await Task.Delay(TimeSpan.FromMilliseconds(200), cts.Token);

    Console.WriteLine($"{page.Length} / {both.Length}");
}
catch (OperationCanceledException)
{
    Console.WriteLine("отменено (таймаут/пользователь) — штатная ситуация");
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"сетевая ошибка: {ex.Message}");
}
```

### Запреты

| Не делай | Делай | Почему |
|----------|-------|--------|
| `.Result` / `.Wait()` / `.GetAwaiter().GetResult()` | `await` | deadlock / starvation (§6.1) |
| `async void MyMethod()` | `async Task MyMethodAsync()` | исключение роняет процесс (§6.2) |
| `DoAsync();` без await | `await DoAsync();` | гонки + проглоченный exception (§6.3) |
| `await` в цикле для независимых операций | `Task.WhenAll` | время = сумма вместо максимума (§6.4) |
| `Thread.Sleep` в async-коде | `await Task.Delay` | блокирует поток (§6.5) |
| async-метод без `CancellationToken` | принять и пробросить `ct` | работа-зомби после отмены (§5) |

---

## 10. Практика — упражнения с разбором

Решения полные — читай после своей попытки.

### 10.1. Три страницы параллельно

**Задача:** скачать три URL параллельно и вывести суммарный размер в символах.

```csharp
using var client = new HttpClient();
string[] urls =
[
    "https://example.com",
    "https://example.org",
    "https://example.net"
];

Task<string>[] downloads = [.. urls.Select(url => client.GetStringAsync(url))];
string[] pages = await Task.WhenAll(downloads);

int totalChars = pages.Sum(p => p.Length);
Console.WriteLine($"Всего символов: {totalChars}");
```

**Что демонстрируем:** задачи стартуют в `Select` (до await), `WhenAll` собирает результаты, суммарное время ≈ времени самого медленного запроса.

### 10.2. Починить sync-over-async

**Задача:** метод писали до знакомства с async. Найди все проблемы и перепиши.

```csharp
// ДО: три греха в шести строках
public sealed class LegacyUserService(HttpClient client)
{
    public string GetUserName(string id)
    {
        var json = client.GetStringAsync($"https://api.example.com/users/{id}").Result; // грех 1: .Result
        client.PostAsync("https://audit.example.com/log", content: null);               // грех 2: забытый await
        return json.ToUpperInvariant();
        // грех 3: нет CancellationToken
    }
}
```

```csharp
// ПОСЛЕ
public sealed class ModernUserService(HttpClient client)
{
    public async Task<string> GetUserNameAsync(string id, CancellationToken ct)
    {
        string json = await client.GetStringAsync($"https://api.example.com/users/{id}", ct);
        await client.PostAsync("https://audit.example.com/log", content: null, ct);
        return json.ToUpperInvariant();
    }
}
```

**Что важно:** сигнатура изменилась (`string` → `Task<string>`, `+ Async`, `+ ct`) — значит, изменятся и все вызывающие. Асинхронность заразна, и это нормально.

### 10.3. Code review: найди баг

**Задача:** метод компилируется и «обычно работает». Что с ним не так и когда он выстрелит?

```csharp
public sealed class NewsletterService(HttpClient client)
{
    public async void SendAll(List<string> emails)
    {
        foreach (string email in emails)
        {
            await client.PostAsync($"https://mail.example.com/send?to={email}", content: null);
        }
    }
}
```

**Разбор:**
1. **`async void`** — вызывающий не может дождаться завершения и не увидит исключений; любой сбой SMTP-шлюза уронит процесс (§6.2). Нужен `async Task SendAllAsync(...)`.
2. **Нет `CancellationToken`** — при остановке приложения рассылка продолжит крутиться (§5).
3. Спорное место для обсуждения: последовательный цикл. Здесь он может быть **осознанным** (не заваливать SMTP-шлюз параллелью), но это решение должно быть явным — комментарий или ограниченная параллель через `Parallel.ForEachAsync`.

### 10.4. RetryAsync — маленький помощник

**Задача:** написать метод, который выполняет async-операцию до 3 раз с паузой 500 мс между попытками; если все попытки провалились — пробросить последнее исключение.

```csharp
// Использование (top-level statements — до объявления класса):
using var client = new HttpClient();
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));

string html = await Retry.RunAsync(
    token => client.GetStringAsync("https://example.com", token),
    maxAttempts: 3,
    delay: TimeSpan.FromMilliseconds(500),
    ct: cts.Token);

Console.WriteLine(html.Length);

public static class Retry
{
    public static async Task<T> RunAsync<T>(
        Func<CancellationToken, Task<T>> action,
        int maxAttempts,
        TimeSpan delay,
        CancellationToken ct)
    {
        for (int attempt = 1; ; attempt++)
        {
            try
            {
                return await action(ct);
            }
            catch (Exception) when (attempt < maxAttempts)
            {
                // ловим только если остались попытки; иначе исключение летит вызывающему
                await Task.Delay(delay, ct);
            }
        }
    }
}
```

**Что демонстрируем:** `Func<CancellationToken, Task<T>>` — «асинхронная операция как параметр»; exception filter `when` — повторяем только пока есть попытки; токен пробрасывается и в операцию, и в `Task.Delay`. В production для retry используют Polly ([[http-client-resilience|HttpClient Resilience]]), но понимать механику нужно руками.

---

## 11. См. также

**Canonical deep-dive (следующий шаг после этого файла):**
- [[async-threading|Async and Threading]] — state machine, `SynchronizationContext`, `ConfigureAwait`, `ValueTask`, `TaskCompletionSource`, синхронизация, deadlock-разборы

**Смежное на Junior-уровне:**
- [[iterators-yield|Iterators / yield]] — `IAsyncEnumerable` и `await foreach`: асинхронные потоки данных
- [[error-handling|Error Handling]] — исключения, Result pattern; как ловить ошибки async-цепочек
- [[io-streams|IO Streams]] — асинхронное чтение/запись файлов и потоков
- [[ef-basics|EF Basics]] — откуда берутся `ToListAsync`/`SaveChangesAsync`
- [[http-client-resilience|HttpClient Resilience]] — правильная жизнь `HttpClient`, retry, таймауты
- [[threading-basics|Threading Basics]] — потоки и thread pool, на которых всё это работает
- [[dispose-pattern|Dispose Pattern]] — `IAsyncDisposable` и `await using`

---

## 12. Reading list

**Документация (начни отсюда):**
- **Asynchronous programming with async and await** — `learn.microsoft.com/dotnet/csharp/asynchronous-programming/` — официальный вводный курс, включая аналогию с завтраком (наш официант — её родня).
- **Task asynchronous programming model** — `learn.microsoft.com/dotnet/csharp/asynchronous-programming/task-asynchronous-programming-model` — модель `Task`/`await` подробнее.

**Stephen Cleary (лучшее объяснение «на пальцах»):**
- **Async and Await** — `blog.stephencleary.com/2012/02/async-and-await.html` — канонический вводный пост.
- **Don't Block on Async Code** — `blog.stephencleary.com/2012/07/dont-block-on-async-code.html` — почему `.Result` дедлочит, с картинками.
- **There Is No Thread** — `blog.stephencleary.com/2013/11/there-is-no-thread.html` — кто «ждёт» I/O на самом деле.
- **Concurrency in C# Cookbook** (книга, O'Reilly) — когда захочешь систематизировать; рецептами, без воды.

**Когда будешь готов к deep-dive (после [[async-threading|async-threading]]):**
- **David Fowler — Async Guidance** — `github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md` — свод do/don't от архитектора ASP.NET Core.
- **Stephen Toub — How Async/Await Really Works in C#** — `devblogs.microsoft.com/dotnet/how-async-await-really-works/` — state machine изнутри; читать не раньше Middle.
