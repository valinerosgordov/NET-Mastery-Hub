---
tags: [rabbitmq, masstransit, kafka, messaging, message-broker, outbox, distributed]
level: Senior
---

# Очереди и Message Brokers

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

Абстракция над RabbitMQ, Azure Service Bus, Amazon SQS, Kafka. Mediator-подобный API.

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

Решает dual-write problem (см. [Distributed Systems](../Architecture/distributed-systems.md)).

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

См. подробно в [Distributed Systems](../Architecture/distributed-systems.md).

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

См. полную обвязку в [Observability](observability.md).

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
**Решение:** inbox-таблица с message_id, см. [Distributed Systems](../Architecture/distributed-systems.md).

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
**Решение:** нumeric version в payload, upcasters для старых версий. См. [Distributed Systems / Event versioning](../Architecture/distributed-systems.md#versioning-events).

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

## См. также

- [Distributed Systems](../Architecture/distributed-systems.md) — Outbox, Inbox, Saga, идемпотентность глубоко
- [Observability](observability.md) — OpenTelemetry для tracing through messaging
- [Resilience](../AspNetCore/resilience.md) — Polly паттерны, применимы к message handling
- [PostgreSQL Deep](../SQL/postgresql-deep.md) — outbox-таблица, advisory locks
- [System Design](../Architecture/system-design.md) — выбор Kafka vs RabbitMQ, capacity planning
- [Hangfire Deep](hangfire-deep.md) — альтернатива для simple background jobs

## Reading list

- **MassTransit docs** — masstransit.io (особенно Configuration, Sagas, Outbox, RetryPolicy)
- **RabbitMQ in Depth** — Gavin M. Roy (главное о брокере)
- **Kafka: The Definitive Guide** — Confluent (про Kafka + Kafka Streams)
- **Designing Event-Driven Systems** — Ben Stopford (бесплатная книга от Confluent)
- **Microservices Patterns** — Chris Richardson (главы про messaging, Saga, CQRS)
- **CloudAMQP blog** — cloudamqp.com/blog (real-world performance, troubleshooting RabbitMQ)
- **Confluent blog** — confluent.io/blog (Kafka deep-dives)
- **MassTransit Discord** — самое живое community для troubleshooting
