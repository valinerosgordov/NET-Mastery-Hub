---
tags: [rabbitmq, masstransit, kafka, messaging, message-broker, outbox, distributed]
level: Senior
date: 2026-08-02
---

# Очереди и Message Brokers

> Асинхронная коммуникация через message brokers: RabbitMQ vs Kafka, MassTransit, Outbox pattern и Dead Letter Queue — когда брать очередь ради loose coupling, retry и устойчивости к падению консьюмера.

> [!warning] Лицензия MassTransit — v9 коммерческий (2026)
> MassTransit v9 (Q1 2026, компания Massient) — коммерческий: ~$400/мес SMB, ~$1200/мес enterprise; для организаций с выручкой < $1M/год есть 100%-discount тир. **v8 остаётся Apache 2.0**, но получает только security-патчи — EOL конец 2026. Все примеры кода в этом файле валидны для v8.
> Альтернативы: **Wolverine** (MIT, ядро OSS — mediator + messaging + saga + outbox одним инструментом; платный только аддон CritterWatch), **Rebus**, **Brighter**, **NServiceBus** (давно коммерческий) или raw-клиенты брокеров (`RabbitMQ.Client`, `Confluent.Kafka`). Как принимать такие решения системно — [[choosing-dependencies|Choosing Dependencies]].

## Что это, зачем и когда

### Что такое Message Broker?
Промежуточное хранилище сообщений между частями приложения. Один сервис **отправляет** сообщение в очередь, другой **забирает** когда готов. Если получатель упал — сообщение **ждёт** в очереди.

**Аналогия:** Почтовый ящик. Ты кидаешь письмо — почтальон доставит когда сможет. Ты НЕ стоишь у двери получателя и не ждёшь.

### Зачем нужен?

| Без очереди (синхронно) | С очередью (асинхронно) |
|------------------------|------------------------|
| API вызывает EmailService НАПРЯМУЮ | API кладёт задачу в очередь, отвечает сразу |
| EmailService упал → API ломается | EmailService упал → сообщение ждёт в очереди |
| EmailService медленный → пользователь ждёт | API отвечает за 50мс, email отправится в фоне |
| Нет retry — письмо потерялось | Автоматический retry, Dead Letter Queue |
| Tight coupling между компонентами | Loose coupling — продюсер не знает о консьюмерах |

### Когда использовать?
- Email/SMS уведомления — не блокировать пользователя
- Обработка файлов — resize, конвертация, генерация PDF
- Интеграция между сервисами — один не знает о другом
- Event-driven архитектура — `OrderCreated` → склад + оплата + email
- Тяжёлые операции — всё, что > 1 секунды
- Pub-Sub broadcasting — одно событие, много подписчиков

### Когда НЕ нужен?
- Простое CRUD-приложение с одним сервером
- Синхронный ответ обязателен (пользователь ждёт результат)
- MVP / прототип (избыточная сложность)

---

## RabbitMQ vs Kafka — главный выбор

| | **RabbitMQ** | **Kafka** |
|--|--------------|-----------|
| **Парадигма** | Smart broker, dumb consumer | Dumb broker, smart consumer |
| **Модель** | Очереди + exchanges | Append-only log + topics |
| **Throughput** | 10-50K msg/sec | 1M+ msg/sec |
| **Latency** | Низкая (~ms) | Низкая, но через batch |
| **Ordering** | Per-queue | Per-partition |
| **Storage** | Оптимизирован для коротких сообщений | Оптимизирован для долгого хранения (days/weeks) |
| **Replay сообщений** | Сложно (специальная очередь) | Из коробки (consumer offset) |
| **Use case** | Task queue, RPC, command/event | Event streaming, log aggregation, analytics |

**Когда RabbitMQ:**
- Task queues (email, image processing) — короткие задачи, быстрая доставка
- RPC pattern — request/response через очереди
- Сложная маршрутизация по headers/routing keys
- Web app сценарии — большинство задач

**Когда Kafka:**
- Event sourcing — храним всю историю событий
- Аналитика, ML pipelines — миллионы событий/сек
- Microservices с большим throughput
- Replay events — пересчитать read model из истории

**В сомнениях — начинай с RabbitMQ.** Перейти на Kafka позже проще, чем сразу взять Kafka и страдать с его сложностью.

---

## RabbitMQ — модель

```
Producer → Exchange → Binding → Queue → Consumer
```

### Exchange types

| Тип | Маршрутизация | Пример |
|-----|---------------|--------|
| **Direct** | По точному routing key | `order.created` → конкретная очередь |
| **Topic** | По паттерну (`order.*`, `#.error`) | `order.created.eu` → несколько очередей |
| **Fanout** | Все привязанные очереди (broadcast) | Логирование, уведомления |
| **Headers** | По заголовкам сообщения | Редко используется |

### Гарантии доставки

| Setting | Что даёт |
|---------|----------|
| **Durable queue** | Очередь переживает рестарт брокера |
| **Persistent message** (delivery_mode=2) | Сообщение пишется на диск (не в RAM) |
| **Publisher confirms** | Брокер подтверждает получение сообщения |
| **Consumer acknowledgment** | Consumer явно подтверждает (`BasicAck`) — иначе redeliver |
| **Prefetch count** | Сколько сообщений consumer держит unack'ed одновременно |

**Базовое правило:** `BasicAck` после успешной обработки. Если consumer упадёт до ACK — сообщение вернётся в очередь автоматически. **Идемпотентность consumer обязательна**.

### Quorum queues vs Classic mirrored queues

В RabbitMQ 3.8+ добавили **quorum queues** — replicated queues на основе Raft consensus. Они заменяют старые "mirrored queues".

| | Classic mirrored | Quorum |
|--|------------------|--------|
| Алгоритм | Custom replication | Raft (proven) |
| Recovery после split-brain | Manual | Auto |
| Performance | Хуже при N replicas | Лучше |
| Ordering при failover | Может теряться | Гарантировано |
| Когда | Legacy | **Стандарт для production 2026** |

```bash
# При создании queue
rabbitmqctl declare queue --type=quorum name=orders
```

### Lazy queues — для очень больших объёмов

Если очередь может расти до миллионов сообщений (slow consumer / batch processing), используй lazy queue. Они хранят все на диске, не пытаясь держать в RAM:

```bash
rabbitmqctl declare queue name=batch x-queue-mode=lazy
```

Trade-off: latency выше, но не упрётся в RAM.

### Dead Letter Queue (DLQ)

Сообщения попадают в DLQ при:
- Отклонении consumer'ом (`BasicNack` с `requeue=false`)
- Истечении message TTL
- Переполнении queue (max-length)
- Превышении max-retries после re-publish

```csharp
// Setup queue с DLQ
var args = new Dictionary<string, object>
{
    ["x-dead-letter-exchange"] = "dlx-orders",
    ["x-dead-letter-routing-key"] = "orders.failed",
    ["x-message-ttl"] = 60000,  // 1 minute TTL
    ["x-max-length"] = 100000,
};
channel.QueueDeclare("orders", durable: true, exclusive: false, autoDelete: false, arguments: args);
```

Мониторинг DLQ — критичен. Alert если depth > 0 на 5+ минут.

> [!question]- **Интервью: что такое poison message и как с ним бороться?**
> Сообщение, которое consumer не может обработать (bad payload, нарушенный invariant). Без защиты consumer вечно retry'ится, тормозит очередь.
> **Решения:**
> 1. **Max retry attempts** — после N failed → отправить в DLQ
> 2. **DLQ + alert** — мониторинг depth DLQ, ручной разбор
> 3. **Quarantine queue** — отдельная очередь для poison, отдельный low-priority worker может попытаться обработать через час

---

## MassTransit — основной .NET-фреймворк

Абстракция над RabbitMQ, Azure Service Bus, Amazon SQS, Kafka. Mediator-подобный API. ⚠️ v9 — коммерческий (см. callout в начале файла); примеры ниже — v8 (Apache 2.0).

### Setup

```csharp
builder.Services.AddMassTransit(x =>
{
    // Auto-discovery всех consumers в assembly
    x.AddConsumers(typeof(Program).Assembly);

    x.SetKebabCaseEndpointNameFormatter();  // OrderCreatedConsumer → order-created

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });

        // Common pipeline для всех consumers
        cfg.UseMessageRetry(r => r.Exponential(
            retryLimit: 5,
            minInterval: TimeSpan.FromSeconds(1),
            maxInterval: TimeSpan.FromMinutes(5),
            intervalDelta: TimeSpan.FromSeconds(2)));

        cfg.UseInMemoryOutbox(ctx);  // Если без EF outbox

        cfg.ConfigureEndpoints(ctx);
    });
});
```

### Consumer

```csharp
public sealed class OrderCreatedConsumer(
    IOrderProcessingService service,
    ILogger<OrderCreatedConsumer> logger) : IConsumer<OrderCreated>
{
    public async Task Consume(ConsumeContext<OrderCreated> context)
    {
        var msg = context.Message;
        logger.LogInformation("Processing OrderCreated {OrderId}", msg.OrderId);

        await service.ProcessAsync(msg.OrderId, context.CancellationToken);

        // Опционально — publish следующее событие
        await context.Publish(new OrderProcessed(msg.OrderId));
    }
}

public sealed record OrderCreated(Guid OrderId, decimal Total);
public sealed record OrderProcessed(Guid OrderId);
```

### Publish vs Send

| | Publish | Send |
|--|---------|------|
| Семантика | Event broadcasting (pub-sub) | Command (один получатель) |
| Куда | На все queues, привязанные к message type | В конкретный endpoint |
| Использование | "Произошло X" | "Сделай Y" |

```csharp
// Publish — не знаем сколько получателей
await _publishEndpoint.Publish(new OrderCreated(...));

// Send — знаем конкретный target
await _sendEndpointProvider
    .GetSendEndpoint(new Uri("queue:order-processing"))
    .Send(new ProcessOrder(...));
```

### Concurrency и prefetch

```csharp
cfg.ReceiveEndpoint("order-processing", e =>
{
    e.PrefetchCount = 16;        // Сколько unack messages в полёте
    e.ConcurrentMessageLimit = 8; // Сколько обрабатывается параллельно
    e.UseConcurrencyLimit(8);
    e.ConfigureConsumer<OrderCreatedConsumer>(ctx);
});
```

| Setting | Что даёт |
|---------|----------|
| **PrefetchCount** | Сколько брокер шлёт в этот consumer-канал. Высокий = меньше round-trips, но накопление при slow consumer |
| **ConcurrentMessageLimit** | Сколько messages обрабатывается параллельно в .NET. Ограничь чтобы не съесть всю CPU |

**Правило:** PrefetchCount ≥ ConcurrentMessageLimit × 2. Например: 8 параллельных + 16 prefetch = всегда есть запас.

### Retry policies

```csharp
cfg.UseMessageRetry(r =>
{
    r.Immediate(2);                                                         // 2 immediate retries
    r.Interval(3, TimeSpan.FromSeconds(5));                                 // 3 retries по 5 сек
    r.Exponential(5, TimeSpan.FromSeconds(1), TimeSpan.FromMinutes(5),
                  TimeSpan.FromSeconds(2));                                  // exponential 5 раз
    r.Incremental(5, TimeSpan.FromSeconds(2), TimeSpan.FromSeconds(2));     // 2, 4, 6, 8, 10 sec
});

// Per-consumer
public sealed class OrderConsumerDefinition : ConsumerDefinition<OrderCreatedConsumer>
{
    protected override void ConfigureConsumer(IReceiveEndpointConfigurator endpoint, IConsumerConfigurator<OrderCreatedConsumer> consumer)
    {
        endpoint.UseMessageRetry(r => r.Interval(3, TimeSpan.FromSeconds(10)));
    }
}
```

### Redelivery (delayed retry)

`UseMessageRetry` — synchronous retry в текущем consumer. **Redelivery** — schedule message на потом, освобождая поток:

```csharp
cfg.UseDelayedRedelivery(r => r.Intervals(
    TimeSpan.FromMinutes(5),
    TimeSpan.FromMinutes(15),
    TimeSpan.FromMinutes(30),
    TimeSpan.FromHours(1)));
```

Это требует **scheduler** — RabbitMQ delayed-message exchange plugin или Quartz.NET integration. Используется когда retry с большой задержкой (downstream сервис лежит, не нужно блокировать workers).

### Outbox через EntityFramework

Решает dual-write problem (См. [[distributed-systems|Distributed Systems]]).

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.QueryDelay = TimeSpan.FromSeconds(1);
        o.UsePostgres();
        o.UseBusOutbox();  // Publish после COMMIT
    });

    x.AddConsumers(typeof(Program).Assembly);
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("rabbitmq");
        cfg.ConfigureEndpoints(ctx);
    });
});
```

Теперь `await _publisher.Publish(...)` внутри `_db.SaveChanges` атомарен. В NexusAI — рекомендуемая baseline-конфигурация.

**Inbox автоматически тоже включается** — consumer'ы становятся идемпотентными через MassTransit-managed таблицу `InboxState`.

### Saga state machines

```csharp
public sealed class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = "";
    public Guid OrderId { get; set; }
}

public sealed class OrderSaga : MassTransitStateMachine<OrderState>
{
    public State PendingPayment { get; private set; } = null!;
    public State Completed { get; private set; } = null!;

    public Event<OrderCreated> OrderCreated { get; private set; } = null!;
    public Event<PaymentSucceeded> PaymentSucceeded { get; private set; } = null!;

    public OrderSaga()
    {
        InstanceState(x => x.CurrentState);
        Event(() => OrderCreated, x => x.CorrelateById(c => c.Message.OrderId));
        Event(() => PaymentSucceeded, x => x.CorrelateById(c => c.Message.OrderId));

        Initially(
            When(OrderCreated)
                .Then(ctx => ctx.Saga.OrderId = ctx.Message.OrderId)
                .Publish(ctx => new ChargePayment(ctx.Saga.OrderId))
                .TransitionTo(PendingPayment));

        During(PendingPayment,
            When(PaymentSucceeded)
                .TransitionTo(Completed)
                .Finalize());

        SetCompletedWhenFinalized();
    }
}

builder.Services.AddMassTransit(x =>
{
    x.AddSagaStateMachine<OrderSaga, OrderState>()
        .EntityFrameworkRepository(r =>
        {
            r.ConcurrencyMode = ConcurrencyMode.Pessimistic;
            r.ExistingDbContext<AppDbContext>();
        });
});
```

См. подробно в[[distributed-systems|Distributed Systems]].

### Scheduling

```csharp
// Schedule message на через 30 минут
await _scheduler.SchedulePublish(
    DateTime.UtcNow.AddMinutes(30),
    new SendReminderEmail(userId));

// Cancel
var scheduledMessage = await _scheduler.SchedulePublish(...);
await _scheduler.CancelScheduledPublish(scheduledMessage);
```

Требует Quartz integration или Hangfire или RabbitMQ delayed-message plugin:
```csharp
x.AddDelayedMessageScheduler();  // RabbitMQ plugin
// или
x.AddQuartzConsumers();          // Quartz.NET
```

### Request-Response

```csharp
// Request side
var client = bus.CreateRequestClient<GetOrderStatus>();
var response = await client.GetResponse<OrderStatusResponse>(
    new GetOrderStatus(orderId),
    timeout: TimeSpan.FromSeconds(5));
var status = response.Message.Status;

// Response side
public class GetOrderStatusConsumer : IConsumer<GetOrderStatus>
{
    public async Task Consume(ConsumeContext<GetOrderStatus> ctx)
    {
        var status = await GetStatusAsync(ctx.Message.OrderId);
        await ctx.RespondAsync(new OrderStatusResponse(status));
    }
}
```

Внутри — auto-generated reply-to queue, MassTransit разруливает correlation. Удобно для RPC через очереди (с timeout!).

### Observability

```csharp
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddSource("MassTransit")  // ВАЖНО
        .AddOtlpExporter())
    .WithMetrics(metrics => metrics
        .AddMeter("MassTransit")
        .AddPrometheusExporter());

// Custom filter — в Pipeline
cfg.UseConsumeFilter(typeof(LoggingFilter<>), ctx);

public class LoggingFilter<T>(ILogger<LoggingFilter<T>> logger) : IFilter<ConsumeContext<T>> where T : class
{
    public async Task Send(ConsumeContext<T> context, IPipe<ConsumeContext<T>> next)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            await next.Send(context);
            logger.LogInformation("Consumed {Type} in {Ms}ms", typeof(T).Name, sw.ElapsedMilliseconds);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Consumer {Type} failed", typeof(T).Name);
            throw;
        }
    }

    public void Probe(ProbeContext context) { }
}
```

См. полную обвязку в [[observability|Observability]].

### Batching

Когда задачи маленькие но многочисленные — обрабатывай батчами:

```csharp
public class OrderBatchConsumer : IConsumer<Batch<OrderCreated>>
{
    public async Task Consume(ConsumeContext<Batch<OrderCreated>> context)
    {
        var orderIds = context.Message.Select(c => c.Message.OrderId).ToList();
        await _service.ProcessBatchAsync(orderIds, context.CancellationToken);
    }
}

// Setup
cfg.ReceiveEndpoint("orders-batch", e =>
{
    e.PrefetchCount = 100;
    e.UseBatch<OrderCreated>(b =>
    {
        b.MessageLimit = 50;          // или
        b.TimeLimit = TimeSpan.FromSeconds(5);  // первое срабатывает
    });
    e.ConfigureConsumer<OrderBatchConsumer>(ctx);
});
```

Один `Insert ... VALUES (50 rows)` вместо 50 отдельных insert'ов = драматическое снижение нагрузки на БД.

---

## Kafka — основные концепции

### Топики и партиции

```
Topic: order-events
├── Partition 0: [msg1, msg2, msg3, ...]
├── Partition 1: [msg4, msg5, msg6, ...]
└── Partition 2: [msg7, msg8, msg9, ...]
```

**Ordering гарантирован per-partition.** Сообщения с одинаковым ключом (например, `order_id`) идут в один partition → consumer обрабатывает их по порядку.

### Consumer groups

```
Topic order-events (3 partitions)

Consumer Group "billing":
  Consumer A → Partition 0
  Consumer B → Partition 1
  Consumer C → Partition 2

Consumer Group "analytics":
  Consumer X → Partition 0, 1, 2 (один consumer на 3 partitions)
```

- Один partition обрабатывается **одним consumer в группе** (parallelism limited by partitions)
- Несколько групп читают независимо — каждая со своим offset
- При добавлении/удалении consumer — rebalance автоматический

### Offset commit

```csharp
// Auto-commit (опасно — можешь потерять message при крэше до обработки)
config.EnableAutoCommit = true;

// Manual commit — после успешной обработки
config.EnableAutoCommit = false;
consumer.Subscribe("order-events");

while (true)
{
    var result = consumer.Consume(ct);
    await ProcessAsync(result.Message.Value);
    consumer.Commit(result);
}
```

### Confluent.Kafka SDK для .NET

```csharp
var producerConfig = new ProducerConfig
{
    BootstrapServers = "kafka:9092",
    Acks = Acks.All,           // Wait for all replicas
    EnableIdempotence = true,   // Дедупликация на producer side
    CompressionType = CompressionType.Lz4,
};

using var producer = new ProducerBuilder<string, string>(producerConfig).Build();

await producer.ProduceAsync("order-events", new Message<string, string>
{
    Key = orderId.ToString(),
    Value = JsonSerializer.Serialize(orderEvent),
    Headers = [new Header("trace-id", Encoding.UTF8.GetBytes(traceId))],
});
```

### MassTransit + Kafka

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddRider(rider =>
    {
        rider.AddConsumer<OrderEventConsumer>();

        rider.UsingKafka((ctx, k) =>
        {
            k.Host("kafka:9092");

            k.TopicEndpoint<OrderEvent>("order-events", "billing-group", e =>
            {
                e.ConfigureConsumer<OrderEventConsumer>(ctx);
                e.AutoOffsetReset = AutoOffsetReset.Earliest;
            });
        });
    });
});
```

### Когда Kafka выигрывает

| Сценарий | Почему Kafka |
|----------|-------------|
| Event sourcing | Persistent log, replay через offset |
| Аналитика real-time | High throughput, partitioning by user/tenant |
| Stream processing (Kafka Streams / KStreams.NET) | Native windowing, joins, aggregations |
| Long retention (7 дней+) | Disk-based storage, не память |
| Backpressure из-за burst'ов | Producer не блокируется, consumer догоняет |

### Kafka retention & consumer liveness pitfalls

Здесь ломаются почти все, кто пришёл из RabbitMQ: в Kafka «прочитал» и «удалил» — это **разные оси**, и обе устроены не так, как подсказывает интуиция.

**Retention удаляет сегменты целиком, не отдельные сообщения.** Партиция физически — это цепочка сегмент-файлов (`*.log`). Активный сегмент пишется до тех пор, пока не накопит `segment.bytes` или не проживёт `segment.ms`, после чего **роллится** (закрывается) и появляется новый. `retention.ms` / `retention.bytes` применяются **только к закрытым (rolled) сегментам** и работают по принципу «удалить самый старый сегмент, когда он стал старше/тяжелее лимита». Это значит:

- Ретеншн **никогда не урезает уже записанный сегмент задним числом** — он либо целиком хранится, либо целиком удаляется.
- Активный сегмент **не удаляется никогда**, пока не закроется. Если трафик низкий, данные старше `retention.ms` могут лежать на диске часами просто потому, что сегмент ещё не докатился до ролла.
- **Всплеск продюсера переполнит диск независимо от `retention.ms`.** Лимиты считаются на закрытых сегментах постфактум; пока burst пишется в активный сегмент, удалять нечего. Диск кончится раньше, чем сработает ретеншн. Защита — `segment.ms` поменьше (чтобы сегменты роллились чаще и попадали под удаление), отдельный disk-usage alert и квоты продюсера.

> [!warning] «Read != delete»
> Чтение сообщения консьюмером **ничего не удаляет**. Данные живут ровно столько, сколько диктует retention, и читаются сколько угодно раз сколько угодно группами. Это и есть основа replay — но это же значит, что «я всё вычитал» **не освобождает диск**. За место отвечает только retention/compaction, а не консьюмеры.

**Consumer group масштабируется только до числа партиций.** Внутри одной группы партиция закреплена строго за одним консьюмером (1:1). Подняли `partitions = 6`, запустили 10 подов — 6 работают, **4 простаивают** вечно. Параллелизм потребления = `min(consumers, partitions)`. Хотите больше воркеров — поднимайте число партиций заранее (уменьшить его потом нельзя), закладывая будущий рост.

```csharp
// Confluent.Kafka: ручной коммит ПОСЛЕ обработки — at-least-once
var config = new ConsumerConfig
{
    BootstrapServers = "kafka:9092",
    GroupId = "billing-group",
    EnableAutoCommit = false,                    // коммитим сами
    AutoOffsetReset = AutoOffsetReset.Earliest,
    // Дать брокеру понять, сколько РЕАЛЬНО занимает обработка одного сообщения:
    MaxPollIntervalMs = 300_000,                 // верхняя граница времени между poll'ами
    SessionTimeoutMs = 45_000,
};

using var consumer = new ConsumerBuilder<string, string>(config).Build();
consumer.Subscribe("order-events");

while (!ct.IsCancellationRequested)
{
    var cr = consumer.Consume(ct);
    await ProcessAsync(cr.Message.Value, ct);    // обработка ДО коммита
    consumer.Commit(cr);                         // offset двигаем только после успеха
}
```

**`MaxPollIntervalMs` должен превышать реальное время обработки одного сообщения (worst-case), иначе брокер выкинет консьюмера прямо посреди работы.** Liveness в Kafka определяется не «жив ли процесс», а «как давно консьюмер вызывал `poll()`». Если между двумя `poll()` прошло больше `MaxPollIntervalMs` (например, обработка батча упёрлась в медленный downstream), брокер считает консьюмера зависшим, исключает его из группы и запускает **rebalance** — партиции переедут на других, а текущая работа может быть выполнена повторно. Классический failure mode: тяжёлый хэндлер + дефолтный `MaxPollIntervalMs` = бесконечный цикл «обработал половину → ребаланс → начали заново». Лечится либо ростом `MaxPollIntervalMs`, либо `max.poll.records`/уменьшением батча, либо выносом тяжёлой работы из poll-петли.

> [!tip] Lag != business latency
> Consumer lag (`high-water-mark − committed-offset`) — это **сколько сообщений ещё не вычитано**, а не «насколько устарел бизнес-результат». Нулевой lag при at-least-once + ручном коммите после обработки означает «всё обработано и закоммичено». Но lag растёт и когда продюсер просто выдал burst (consumer догонит), и когда хэндлер реально встал — различить помогает только связка lag + throughput + время обработки. Алертить на голый lag без контекста = ложные срабатывания на каждом всплеске трафика.

**Дефолт — at-least-once + идемпотентный хэндлер.** Коммить offset **после** успешной обработки (как в примере). Тогда краш до коммита приводит к повторной доставке, а не к потере — а повтор безопасен, потому что хэндлер идемпотентен (dedup по `message_id`, upsert вместо insert). Коммит **до** обработки даёт at-most-once и тихую потерю сообщений при краше. Идемпотентность не опция, а условие корректности at-least-once.

### Producer-side batch compression

Сжатие в Kafka — это свойство **продюсера на уровне батча**, а не «заархивируй тело руками». Понимание этой границы экономит и CPU, и нервы на дебаге.

**Не зипуй каждое тело вручную — включай нативное сжатие батча.** Ручной gzip каждого `Value` ломает replay и debugging (в топике лежат не читаемые человеком байты, любой консьюмер обязан знать про твой формат и разжимать сам, инструменты вроде `kafka-console-consumer` показывают мусор) и при этом проигрывает по эффективности. Нативное сжатие (`CompressionType`) жмёт **весь батч целиком** прозрачно: продюсер сжимает на отправке, брокер хранит сжатым, консьюмер разжимает автоматически драйвером — код хэндлера видит обычный JSON.

**Коэффициент сжатия растёт с размером батча.** JSON повторяет имена полей в каждом сообщении (`"orderId"`, `"total"`, `"status"`…), и чем больше однотипных сообщений в одном батче, тем лучше словарь компрессора переиспользует эти повторы. Один объект на батч сжимается слабо; 500 похожих объектов — отлично. Поэтому ради ratio **специально укрупняют батч**: поднимают `LingerMs` (продюсер ждёт пару миллисекунд, накапливая сообщения) и `batch.size`.

```csharp
var producerConfig = new ProducerConfig
{
    BootstrapServers = "kafka:9092",
    // Durability pairing — без этого сжатие просто ускоряет потерю данных:
    Acks = Acks.All,                       // ждём подтверждения всех in-sync реплик
    EnableIdempotence = true,              // ровно-однократная запись, без дублей при retry

    // Батч-сжатие:
    CompressionType = CompressionType.Zstd,
    LingerMs = 5,                          // ~5мс ожидания ради укрупнения батча
    BatchSize = 256 * 1024,                // 256 KiB — больше сообщений в одном сжатом батче
};
```

**Выбор кодека** (на JSON-нагрузке):

| Кодек | Профиль | Когда брать |
|-------|---------|-------------|
| `Gzip` | Лучший ratio, дороже по CPU | Дорогой трафик/хранилище, CPU в запасе |
| `Lz4` | Баланс ratio/скорость | Дефолт общего назначения |
| `Zstd` | Ratio близко к Gzip, скорость ближе к Lz4 | Лучший ratio, если CPU позволяет |

**Брокер хранит то, что прислал продюсер, и НЕ перекомпрессирует — кроме одного случая.** Обычно `compression.type` на брокере = `producer`, и батч ложится на диск в том кодеке, что выбрал продюсер. Перекомпрессия происходит, **только если брокерский (топиковый) `compression.type` явно задан и отличается** от продюсерского — тогда брокер разожмёт и пережмёт батч на запись (дополнительная нагрузка на брокер). Держите брокер на `producer`, если не нужна принудительная унификация.

> [!warning] Tradeoff: CPU растёт с обеих сторон
> Сжатие переносит нагрузку с сети и диска на CPU — **и продюсера, и консьюмера** (разжатие). На CPU-bound сервисах агрессивный `Gzip` может стать новым узким местом. Меряйте: цель — суммарный выигрыш (меньше байт по сети, выше throughput на партицию, меньше места), а не «ratio любой ценой».

> [!tip] Durability pairing
> Сжатие касается эффективности, а не надёжности. Боевой продюсер всегда сочетает его с `Acks.All` + `EnableIdempotence = true`: идемпотентность глушит дубли при ретраях (включая ретраи сжатых батчей), `Acks.All` гарантирует, что батч принят всеми in-sync репликами до подтверждения. Сжатие без этой пары просто быстрее теряет данные.

---

## Real-world deployment

### RabbitMQ cluster

Минимум 3 ноды для quorum. На production:
- 3 ноды в одном datacenter (low latency между нодами)
- Quorum queues (не classic mirrored)
- Managed offering — Amazon MQ, CloudAMQP — берут на себя upgrades, backups
- **Federated queues** — между датацентрами/регионами (но это сложнее)

```
Node 1 (AZ1) ←→ Node 2 (AZ2)
       ↘    ↗
       Node 3 (AZ3)
```

### Memory and disk tuning

```ini
# rabbitmq.conf
vm_memory_high_watermark.relative = 0.6        # alarm на 60% RAM
disk_free_limit.relative = 2.0                 # alarm если disk < 2 × RAM
queue_master_locator = client-local            # ассеssimately co-locate queue с producer
heartbeat = 60                                 # detect dead connections
```

### Monitoring critical metrics

| Метрика | Alert когда |
|---------|------------|
| `rabbitmq_queue_messages_ready` | > 10000 (consumer отстаёт) |
| `rabbitmq_queue_messages_unacknowledged` | > prefetch × consumers (consumer stuck) |
| `rabbitmq_connections` | резкий рост (utilization issue) |
| `rabbitmq_disk_free_bytes` | < threshold |
| `rabbitmq_node_mem_used` | > 80% |
| Messages в DLQ | > 0 на 5 минут |

### Backup стратегия

RabbitMQ definitions backup (queues, exchanges, bindings):
```bash
rabbitmqctl export_definitions /backup/rabbitmq-defs.json
```

Сами сообщения — обычно не бэкапируются (короткоживущие). Если нужно — replicate critical events в S3.

---

## Anti-patterns / common mistakes

### 1. Очередь как БД
"Положу в RabbitMQ, потом разгребу" — миллионы сообщений в очереди. Брокер тормозит, restart долгий, debugging кошмар.
**Решение:** очередь — transport, не storage. Если нужна history — пиши в БД сразу.

### 2. Нет идемпотентности consumer
Retry даёт duplicate side-effects (двойной charge, двойной email).
**Решение:** inbox-таблица с message_id, См. [[distributed-systems|Distributed Systems]].

### 3. Большие payloads в message
Кладёшь 50MB файл в RabbitMQ → broker тормозит, все consumers медленные.
**Решение:** "Claim check" pattern — file в S3, в message только URL/key.

### 4. Synchronous wait на response
"Запросил через очередь, ждём 30 минут" — connections не освобождаются.
**Решение:** реальный async-flow (callback/webhook), MassTransit Request/Response с timeout, или вернуть `202 Accepted` и polling endpoint.

### 5. Один consumer на топик с 1M msg/sec
Throughput limit on single consumer.
**Решение:** Kafka — partitioning + consumer group. RabbitMQ — несколько consumers одной очереди competitively.

### 6. Catch-all `Exception` в consumer

```csharp
public async Task Consume(ConsumeContext<X> ctx)
{
    try { await ProcessAsync(ctx.Message); }
    catch (Exception) { /* swallow */ }  // ❌
}
```
Силently сбрасываешь ошибки → невозможно дебажить.
**Решение:** не лови — пускай retry/DLQ обработают. Лови только конкретные `OperationCanceledException` где нужно.

### 7. Hot consumer (один queue load на 100% единственного worker'а)
**Решение:** PrefetchCount + ConcurrentMessageLimit для параллельности; масштабировать через `dotnet run` несколько инстансов одновременно.

### 8. Не использовать `IPublishEndpoint` в DI

```csharp
// ❌ Прямой Bus — не учитывает outbox/scope
private readonly IBusControl _bus;
await _bus.Publish(...);

// ✅ Правильно
private readonly IPublishEndpoint _publisher;  // или ISendEndpointProvider
await _publisher.Publish(...);
```
`IPublishEndpoint` интегрирован с outbox-механизмом и scope-aware. `IBus` — для startup-кода и низкоуровневых сценариев.

### 9. Kafka без partition strategy
Все сообщения в один partition (если key = null) → нет parallelism.
**Решение:** key = entity ID (order_id, user_id). Тогда сообщения одной сущности в одном partition (ordering), а разные сущности — в разных (parallelism).

### 10. Нет message versioning
Изменили схему `OrderCreated` — старые сообщения в очереди ломают consumer.
**Решение:** нumeric version в payload, upcasters для старых версий. См. [[distributed-systems|Distributed Systems / Event versioning]]).

---

## Production checklist

- [ ] Quorum queues для critical workflows (не classic mirrored)
- [ ] Durable + persistent для всех важных сообщений
- [ ] Publisher confirms enabled
- [ ] Все consumers идемпотентны (inbox)
- [ ] Outbox для всех "save + publish" сценариев (MassTransit `UseBusOutbox()`)
- [ ] PrefetchCount настроен (не default 1)
- [ ] Retry policy явно сконфигурирована
- [ ] DLQ + alert на depth > 0
- [ ] Monitoring: queue depth, unack count, connection count, memory, disk
- [ ] OpenTelemetry tracing настроен (`AddSource("MassTransit")`)
- [ ] Auto-recovery connection on RabbitMQ restart
- [ ] Heartbeat настроен (60s standard)
- [ ] Cluster: минимум 3 ноды
- [ ] Backup definitions (queues, exchanges, bindings)
- [ ] Schema versioning стратегия описана
- [ ] Chaos test: kill broker mid-message — система восстанавливается

---

## Best Practices

### Best Practices for Messaging

**Reliability:**
- **Durable queues** — message не lost при broker restart
- **Acknowledgments (manual)** — confirm после processing, не на receive
- **Dead letter queues (DLQ)** — failed messages → отдельная queue для investigation
- **Idempotent consumers** — обработка дублей не вредит
- **Outbox pattern** — DB transaction + message — atomic
- **Saga pattern** — distributed transactions через events

**Performance:**
- **Batching** — group messages для throughput
- **Compression** — gzip / lz4 для big payloads
- **Partitioning** (Kafka) — parallel processing
- **Prefetch count** (RabbitMQ) — flow control
- **Connection pooling** — не open-close connection per message

**Operations:**
- **Monitoring** — queue depth, consumer lag, error rate
- **Alerting** — DLQ growth, consumer down
- **Tracing** — OpenTelemetry для message flow
- **Schema evolution** — backward-compatible changes
- **Replay capability** (Kafka) — re-process events

**Pitfalls to avoid:**
- ❌ Tight coupling через event payloads
- ❌ Synchronous wait для message reply (use request-reply pattern)
- ❌ Mixing transactional + queue without outbox
- ❌ No DLQ — messages потеряются
- ❌ Сложная routing logic в broker (move в consumer)

См. [[distributed-systems|Distributed Systems]].


---

## Cheat sheet

| Broker | Strengths | Use case |
|--------|-----------|----------|
| **RabbitMQ** | Mature, flexible routing, easy ops | General messaging, work queues |
| **Kafka** | High throughput, replay, partitioning | Event streaming, analytics |
| **Azure Service Bus** | Managed, AD integration | Azure-native enterprise |
| **AWS SQS** | Managed, simple | AWS-native simple queue |
| **Redis Streams** | Lightweight, fast | Simple event log, in cache infra |
| **NATS** | Cloud-native, JetStream | Modern microservices |
| **MassTransit** | Library — abstracts brokers (v9 коммерческий; v8 Apache 2.0 до EOL 2026) | .NET dev productivity |
| **Wolverine** | Library — MIT: mediator + messaging + saga + outbox | OSS-замена MassTransit |

| Pattern | Use |
|---------|-----|
| Work queue | Distribute tasks across workers |
| Pub/Sub | Broadcast events |
| Request-reply | Async RPC |
| Event sourcing | Event log как source of truth |
| Saga | Distributed transactions |
| Outbox | Atomic DB + message |
| CQRS | Read/write separation |

**Message types:**
- **Command** — "do this" — single consumer
- **Event** — "this happened" — multiple consumers
- **Query** — "give me this" — async if needed


---

## Decision tree

```
Какой message broker / pattern?
│
├── Need persistence?
│   ├── Yes → RabbitMQ / Kafka / Service Bus
│   └── No → Redis pub/sub (simple, fast, ephemeral)
│
├── Throughput requirement?
│   ├── < 10K msg/sec → RabbitMQ comfortable
│   ├── 10K-1M msg/sec → Kafka
│   └── > 1M msg/sec → Kafka cluster
│
├── Replay needed?
│   ├── Yes → Kafka (retention) или Event Store
│   └── No → RabbitMQ (delete on consume)
│
├── Routing complexity?
│   ├── Simple → SQS / Service Bus queue
│   ├── Topic-based → RabbitMQ topic exchange
│   ├── Header-based → RabbitMQ headers exchange
│   └── Stream / partition → Kafka topic
│
├── Cloud-managed предпочтительно?
│   ├── Azure → Service Bus / Event Hubs
│   ├── AWS → SQS / Kinesis / SNS
│   └── GCP → Pub/Sub
│
├── Use case?
│   ├── Background tasks → RabbitMQ work queue
│   ├── Microservice events → RabbitMQ pub/sub или Kafka
│   ├── Event sourcing → Kafka / Event Store
│   ├── Analytics ingestion → Kafka
│   └── Real-time browser push → SignalR (не broker)
│
└── .NET library?
    ├── Vendor-specific clients → low-level control
    ├── Wolverine → MIT: mediator + messaging + saga + outbox
    ├── MassTransit → multi-broker abstraction; v9 коммерческий (v8 Apache 2.0, EOL 2026)
    └── NServiceBus → enterprise, давно коммерческий
```


---

## См. также

- [[distributed-systems|Distributed Systems]] — Outbox, Inbox, Saga, идемпотентность глубоко
- [[observability|Observability]] — OpenTelemetry для tracing through messaging
- [[resilience|Resilience]] — Polly паттерны, применимы к message handling
- [[postgresql-deep|PostgreSQL Deep]] — outbox-таблица, advisory locks
- [[system-design|System Design]] — выбор Kafka vs RabbitMQ, capacity planning
- [[choosing-dependencies|Choosing Dependencies]] — лицензионные риски зависимостей (MassTransit v9, MediatR 13+, FluentAssertions 8+), критерии выбора
- Hangfire — альтернатива для simple background jobs (отдельного deep-dive в vault пока нет)

## Reading list

- **MassTransit docs** — masstransit.io (особенно Configuration, Sagas, Outbox, RetryPolicy) — ⚠️ v9 коммерческий, v8 EOL конец 2026
- **Wolverine docs** — wolverinefx.net (MIT-альтернатива: messaging + saga + outbox)
- **RabbitMQ in Depth** — Gavin M. Roy (главное о брокере)
- **Kafka: The Definitive Guide** — Confluent (про Kafka + Kafka Streams)
- **Designing Event-Driven Systems** — Ben Stopford (бесплатная книга от Confluent)
- **Microservices Patterns** — Chris Richardson (главы про messaging, Saga, CQRS)
- **CloudAMQP blog** — cloudamqp.com/blog (real-world performance, troubleshooting RabbitMQ)
- **Confluent blog** — confluent.io/blog (Kafka deep-dives)
- **MassTransit Discord** — самое живое community для troubleshooting
