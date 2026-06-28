---
tags: [eip, integration, content-based-router, routing, messaging, pipeline, dsl, architecture]
level: Senior
date: 2026-06-19
---

# EIP: Content-Based Router в C#

## Что это, зачем и когда

### Что такое Content-Based Router?

Content-Based Router (CBR) — паттерн из каталога Enterprise Integration Patterns (Hohpe & Woolf). Маршрутизатор, который **смотрит внутрь сообщения** и направляет его одному из нескольких получателей в зависимости от содержимого. Не по адресу, не по транспорту — именно по данным внутри.

**Аналогия:** сортировочный центр на почте. На конверте написан индекс — сотрудник читает индекс и кладёт письмо в нужную ячейку. Письмо одно, ячеек много, решение принимается по содержимому, а не по тому, кто принёс письмо.

### Зачем?

| Без роутера | С роутером |
|-------------|------------|
| `if/else` с бизнес-логикой обработки прямо в точке приёма | Решение «куда» отделено от «что делать» |
| Добавить новый тип заказа — править центральный метод | Добавить ветку `When(...)` — остальное не трогаем |
| Невозможно протестировать маршрутизацию отдельно | Роутер — самостоятельный компонент, тестируется изолированно |
| Отправитель должен знать всех получателей | Отправитель знает только роутер |

### Когда применять

- Входящий поток гетерогенный: один endpoint, но сообщения разных типов/приоритетов/тенантов.
- Решение о next-hop зависит от **полей сообщения**, а не от URL или заголовка транспорта.
- Набор получателей меняется чаще, чем сам канал приёма.

Когда **не нужно**: маршрут детерминирован транспортом (разные очереди под разные типы — это уже сделал брокер), либо ветка всего одна (тогда это просто обработчик).

---

> [!question]- **Интервью: чем Content-Based Router отличается от Message Filter и Recipient List?**
> CBR выбирает **ровно один** маршрут из многих (first-match-wins). Message Filter — частный случай с одной веткой: пропустить или отбросить. Recipient List — шлёт сообщение **нескольким** получателям сразу (fan-out), а не одному. Путаница CBR и Recipient List — классический источник дублей или потерь.

---

## Содержание

1. [Единая абстракция IProcessor](#1-единая-абстракция-iprocessor)
2. [Fluent DSL: Choice / When / Otherwise](#2-fluent-dsl-choice--when--otherwise)
3. [Роутер — это тоже processor](#3-роутер--это-тоже-processor)
4. [InOnly vs InOut: тонкий баг request/reply](#4-inonly-vs-inout-тонкий-баг-requestreply)
5. [Контраст с ASP.NET Core middleware pipeline](#5-контраст-с-aspnet-core-middleware-pipeline)

---

## 1. Единая абстракция IProcessor

### Почему сначала абстракция

Главная идея EIP-style маршрутизации: **всё — processor**. Обработчик заказа, логгер, трансформер, и сам роутер — все реализуют один интерфейс. Тогда из них можно собирать конвейеры и деревья как из кубиков, и роутер ничем не отличается от листа.

`Exchange` — конверт сообщения (терминология Camel/EIP). Он несёт входящее сообщение (`In`), место для ответа (`Out`), свойства и паттерн обмена. Processor мутирует `Exchange` или передаёт дальше.

```csharp
public enum ExchangePattern
{
    InOnly, // fire-and-forget: ответ не ожидается
    InOut   // request/reply: вызывающий ждёт Out
}

public sealed class Exchange
{
    public required object In { get; init; }
    public object? Out { get; set; }
    public ExchangePattern Pattern { get; init; } = ExchangePattern.InOnly;
    public Dictionary<string, object?> Properties { get; } = [];
    public Exception? Exception { get; set; }
}

public interface IProcessor
{
    Task Process(Exchange exchange, CancellationToken ct);
}
```

Один метод, `Task` (не `ValueTask` — большинство процессоров реально асинхронны: I/O, сеть), `CancellationToken` обязателен.

### Лист — обычный обработчик

```csharp
public sealed class FulfillOrderProcessor(ILogger<FulfillOrderProcessor> logger) : IProcessor
{
    public async Task Process(Exchange exchange, CancellationToken ct)
    {
        var order = (Order)exchange.In;
        logger.LogInformation("Fulfilling order {OrderId}", order.Id);
        await Task.Yield(); // здесь реальная работа: запись в БД, вызов сервиса
    }
}
```

С точки зрения вызывающего нет разницы между этим листом и целым деревом маршрутизации — оба за интерфейсом `IProcessor`. Это и есть рычаг композиции.

---

## 2. Fluent DSL: Choice / When / Otherwise

### Зачем DSL, а не голый switch

`switch`-выражение по типу сообщения работает, пока веток мало и они в одном месте. Как только появляется потребность собирать маршрут из частей, переиспользовать предикаты, вкладывать роутеры друг в друга — нужна **композируемая запись**. Fluent DSL даёт декларативное «дерево решений», которое само является `IProcessor`.

Контракт DSL:

- `.Choice()` открывает блок выбора;
- `.When(predicate)` задаёт ветку с предикатом `Func<Exchange, bool>` и обработчиком;
- ветки проверяются **сверху вниз, first-match-wins** — выигрывает первая истинная;
- `.Otherwise(processor)` — fallback, если ни одна ветка не сработала;
- `.EndChoice()` закрывает блок и возвращает готовый `IProcessor`.

> [!warning] First-match-wins, не best-match
> Ветки оцениваются по порядку объявления. Более общий предикат, поставленный выше частного, «съест» сообщение, и частная ветка никогда не выполнится. Порядок — это и есть приоритет; держите специфичные предикаты выше общих.

### Реализация

Сам роутер — `sealed class`, реализующий `IProcessor`. Билдер копит ветки и собирает роутер на `EndChoice()`.

```csharp
public sealed class ChoiceRouter(
    IReadOnlyList<(Func<Exchange, bool> Predicate, IProcessor Processor)> branches,
    IProcessor? otherwise) : IProcessor
{
    public async Task Process(Exchange exchange, CancellationToken ct)
    {
        foreach (var (predicate, processor) in branches)
        {
            if (predicate(exchange))
            {
                await processor.Process(exchange, ct);
                return; // first-match-wins
            }
        }

        if (otherwise is not null)
        {
            await otherwise.Process(exchange, ct);
            return;
        }

        // Нет ветки и нет Otherwise — сообщение «провалилось».
        // Решение по умолчанию: пометить как необработанное, не бросать исключение.
        exchange.Properties["unrouted"] = true;
    }
}
```

Билдер с extension-методами на `IProcessor` (точка входа `.Choice()`):

```csharp
public sealed class ChoiceBuilder
{
    private readonly List<(Func<Exchange, bool>, IProcessor)> _branches = [];
    private IProcessor? _otherwise;

    public ChoiceBuilder When(Func<Exchange, bool> predicate, IProcessor processor)
    {
        _branches.Add((predicate, processor));
        return this;
    }

    public ChoiceBuilder Otherwise(IProcessor processor)
    {
        _otherwise = processor;
        return this;
    }

    public IProcessor EndChoice() => new ChoiceRouter(_branches, _otherwise);
}

public static class RouteDsl
{
    public static ChoiceBuilder Choice() => new();
}
```

> [!info] `When(...)` принимает `Func<Exchange, bool>`, а не `Func<TMessage, bool>`

предикат видит весь `Exchange` (свойства, паттерн обмена, заголовки), а не только тело. Это позволяет маршрутизировать по метаданным, не разворачивая сообщение.

### Использование

```csharp
IProcessor router = RouteDsl.Choice()
    .When(e => e.In is Order { Total: > 10_000 }, vipProcessor)
    .When(e => e.In is Order { IsExport: true }, exportProcessor)
    .When(e => e.In is Order, standardProcessor)
    .Otherwise(deadLetterProcessor)
    .EndChoice();

await router.Process(new Exchange { In = order }, ct);
```

VIP-ветка выше экспортной: VIP-экспортный заказ уйдёт в VIP. Поменяете порядок — поменяется приоритет. `Otherwise` ловит всё, что не Order (или Order, не подошедший ни под одну ветку).

---

## 3. Роутер — это тоже processor

### Композиция без специальных случаев

`ChoiceRouter` реализует `IProcessor`, поэтому его можно:

- поставить **листом** в другой `Choice()` — получаем вложенные роутеры (дерево решений);
- вставить в линейный конвейер между другими процессорами;
- передать туда, где ждут любой `IProcessor` — вызывающий не знает, что внутри дерево.

```csharp
IProcessor exportRouting = RouteDsl.Choice()
    .When(e => ((Order)e.In).Country == "US", usExportProcessor)
    .When(e => ((Order)e.In).Country == "EU", euExportProcessor)
    .Otherwise(manualReviewProcessor)
    .EndChoice();

IProcessor topRouter = RouteDsl.Choice()
    .When(e => e.In is Order { Total: > 10_000 }, vipProcessor)
    .When(e => e.In is Order { IsExport: true }, exportRouting) // вложенный роутер как ветка
    .When(e => e.In is Order, standardProcessor)
    .Otherwise(deadLetterProcessor)
    .EndChoice();
```

Никакого `IRouter` отдельно от `IProcessor` заводить не нужно — это нарушило бы единообразие и вернуло «специальные случаи». Роутер композируется ровно потому, что он неотличим от обычного процессора. Это прямое следствие LSP: где ждут `IProcessor`, подставляется и лист, и дерево (см. [[solid|SOLID]]).

### Линейный pipeline как processor

Для полноты — конвейер тоже processor, прогоняющий цепочку по порядку:

```csharp
public sealed class Pipeline(params IProcessor[] steps) : IProcessor
{
    public async Task Process(Exchange exchange, CancellationToken ct)
    {
        foreach (var step in steps)
        {
            await step.Process(exchange, ct);
            if (exchange.Exception is not null) return; // прерываем на ошибке
        }
    }
}
```

Теперь `Pipeline` можно положить веткой в `Choice()`, а `Choice()` — шагом в `Pipeline`. Любая комбинация — снова `IProcessor`.

---

## 4. InOnly vs InOut: тонкий баг request/reply

### Два паттерна обмена

Это **самый частый и самый коварный** баг при ручной маршрутизации. Роутер один и тот же, но семантика обмена принципиально разная.

| | InOnly (fire-and-forget) | InOut (request/reply) |
|--|--------------------------|-----------------------|
| Ожидание ответа | Нет | Да — вызывающий ждёт `Out` |
| Что вернуть клиенту | Сразу `200/202`, не дожидаясь маршрута | Результат из `Exchange.Out` |
| Запуск маршрута | Можно offload в фон | Должен завершиться до ответа |
| Типичный транспорт | Очередь, событие | HTTP request/reply, gRPC |

### InOnly — отвязали маршрут, ответили сразу

```csharp
// Endpoint при InOnly: ставим в фон, отвечаем немедленно
app.MapPost("/orders", (Order order, IBackgroundQueue queue) =>
{
    var exchange = new Exchange { In = order, Pattern = ExchangePattern.InOnly };
    queue.Enqueue(exchange); // маршрут отработает асинхронно
    return Results.Accepted(); // 202, не дожидаясь обработки
});
```

### InOut — сервер обязан дождаться маршрута

```csharp
// Endpoint при InOut: ждём маршрут, возвращаем Exchange.Out
app.MapPost("/quote", async (QuoteRequest req, IProcessor router, CancellationToken ct) =>
{
    var exchange = new Exchange { In = req, Pattern = ExchangePattern.InOut };
    await router.Process(exchange, ct); // ОБЯЗАТЕЛЬНО await до ответа
    return exchange.Out is QuoteResponse resp
        ? Results.Ok(resp)
        : Results.Problem("No reply produced");
});
```

### Где именно ломается

Баг рождается, когда request/reply-сценарий случайно обрабатывают как InOnly — например, переиспользовали «быстрый» fire-and-forget endpoint для запроса, который на самом деле ждёт ответ.

```csharp
// ❌ Баг: запрос требует ответа, а мы вернули 202 не дождавшись маршрута
app.MapPost("/quote", (QuoteRequest req, IBackgroundQueue queue) =>
{
    var exchange = new Exchange { In = req, Pattern = ExchangePattern.InOut };
    queue.Enqueue(exchange);  // маршрут ещё не выполнился
    return Results.Ok(exchange.Out); // Out == null ВСЕГДА → клиент получает пустоту
});
```

Механизм: `Out` заполняется **внутри** `Process`, а здесь мы вернули ответ раньше, чем фоновая очередь вообще запустила маршрут. Гонки нет даже — `Out` гарантированно `null` в момент `return`. Клиент стабильно получает пустой/неверный ответ, а в логах всё «успешно». Поэтому **request/reply требует InOut**: вызывающий обязан `await`-ить маршрут, иначе `Exchange.Out` физически не существует к моменту ответа.

> [!warning] Не offload-ить InOut в фон
> Если паттерн `InOut`, маршрут нельзя ставить в `Channel`/`BackgroundService` и сразу отвечать. Либо `await` в синхронном relative-к-запросу пути, либо честный async request/reply поверх корреляции (correlation id + ожидание ответного сообщения). Тихая подмена InOut на InOnly = молчаливая потеря ответа.

---

## 5. Контраст с ASP.NET Core middleware pipeline

И CBR, и middleware pipeline — про «прохождение сообщения через цепочку». Но это разные звери, и смешивать их ментальные модели опасно.

| Аспект | Content-Based Router | ASP.NET Core middleware |
|--------|----------------------|-------------------------|
| Топология | Дерево: выбор **одной** ветки из многих | Линейная цепочка, проходят все |
| Решение | По **содержимому** сообщения (`When` предикаты) | По порядку регистрации + опционально `MapWhen`/`UseWhen` |
| Обратный путь | Нет «возврата вверх» — выбрали ветку и ушли | Есть: запрос вниз, ответ вверх через те же слои |
| Цель | Доставить сообщение **одному** получателю | Обернуть обработку cross-cutting concerns |
| Завершение | first-match-wins → `return` | `next()` или short-circuit |

Ключевое различие: middleware — это **луковица** (`Request → M1 → M2 → Endpoint → M2 → M1 → Response`), каждый слой видит и запрос, и ответ. CBR — это **развилка**: сообщение уходит в одну ветку и не возвращается «вверх по дереву». Branching через `MapWhen`/`UseWhen` в ASP.NET Core — это уже шаг в сторону CBR (выбор ветки по условию), но предикат там обычно смотрит на путь/заголовки, а не на десериализованное тело.

Разбор pipeline, порядка слоёв и `MapWhen`/`UseWhen` — в [[pipeline-middleware|Pipeline, Middleware и Routing]]. Место CBR среди архитектурных паттернов — в [[architecture-patterns|Архитектура и паттерны проектирования]].

> [!info] Practical note

в .NET не пишут CBR с нуля для прода — берут Apache Camel-аналоги (NServiceBus/MassTransit routing, Rebus, или Camel напрямую через Kamelets). Самописный `ChoiceRouter` оправдан внутри модульного монолита, где не хочется тащить шину ради трёх веток маршрутизации.

---

## Common Pitfalls

| Ошибка | Механизм | Фикс |
|--------|----------|------|
| Общий предикат выше частного | first-match-wins съедает сообщение раньше частной ветки | Специфичные `When` выше общих |
| InOut offload-нут в фон | `Exchange.Out` ещё `null` в момент ответа | `await` маршрут до ответа |
| Нет `Otherwise` | Неподходящее сообщение «проваливается» молча | Всегда задавать fallback (dead-letter / manual review) |
| Роутер шлёт нескольким | Это уже Recipient List, не CBR — дубли обработки | CBR = ровно один получатель; нужен fan-out → другой паттерн |
| Предикат с side-effect | `When(e => Mutate(e))` меняет `Exchange` при оценке | Предикаты — чистые, мутация только в процессорах |

---

## См. также

- [[architecture-patterns|Архитектура и паттерны проектирования]]
- [[pipeline-middleware|Pipeline, Middleware и Routing]]
- [[solid|SOLID]]
- [[distributed-systems|Distributed Systems Patterns]]
