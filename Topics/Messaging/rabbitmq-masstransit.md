# Очереди и Message Brokers

## Зачем нужны

Асинхронная обработка, decoupling сервисов, масштабирование. Producer отправляет сообщение в очередь, Consumer обрабатывает в своём темпе. Retry при ошибках, Dead Letter Queue для необработанных.

**Типичные сценарии:**
- Email/SMS уведомления после оформления заказа
- Обработка загруженных файлов (resize, конвертация)
- Синхронизация данных между сервисами
- Event-driven архитектура (Domain Events)

---

## RabbitMQ

AMQP-брокер. Основные концепции:

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

- **Durable queue** — очередь переживает перезапуск брокера
- **Persistent message** — сообщение записывается на диск
- **Publisher confirms** — брокер подтверждает получение
- **Consumer acknowledgment** — consumer явно подтверждает обработку (`BasicAck`)
- **Prefetch count** — сколько сообщений consumer получает до ACK (контроль нагрузки)

**Нюанс:** `BasicAck` после обработки, не до. Если consumer упадёт до ACK — сообщение вернётся в очередь (redelivery). Идемпотентность consumer — обязательна.

### Dead Letter Queue (DLQ)

Сообщения попадают в DLQ при:
- Отклонении consumer'ом (`BasicNack` без requeue)
- Истечении TTL
- Переполнении очереди

DLQ — отдельная очередь для анализа и ручной обработки ошибок.

---

## MassTransit

Абстракция над RabbitMQ, Azure Service Bus, Amazon SQS. Mediator-подобный API.

### Настройка

```csharp
services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer>();

    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });
        cfg.ConfigureEndpoints(ctx);
    });
});
```

### Consumer

```csharp
public class OrderCreatedConsumer(
    IOrderService orderService,
    ILogger<OrderCreatedConsumer> logger)
    : IConsumer<OrderCreatedEvent>
{
    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        logger.LogInformation("Processing order {OrderId}", context.Message.OrderId);
        await orderService.ProcessAsync(context.Message, context.CancellationToken);
    }
}

// Message contract — record без логики
public record OrderCreatedEvent(Guid OrderId, string CustomerName, decimal Total);
```

### Retry и Error handling

```csharp
cfg.UseMessageRetry(r => r.Intervals(
    TimeSpan.FromSeconds(1),
    TimeSpan.FromSeconds(5),
    TimeSpan.FromSeconds(15)
));

// После всех retry — сообщение в _error queue
// Можно настроить кастомную Dead Letter: cfg.UseDelayedRedelivery(...)
```

### Sagas (State Machine)

Координация длительных бизнес-процессов через state machine. Состояние хранится в БД (EF Core, Redis, MongoDB).

```csharp
public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public State Submitted { get; private set; } = null!;
    public State Paid { get; private set; } = null!;

    public Event<OrderSubmitted> OrderSubmitted { get; private set; } = null!;
    public Event<PaymentReceived> PaymentReceived { get; private set; } = null!;

    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);

        Event(() => OrderSubmitted, x => x.CorrelateById(m => m.Message.OrderId));
        Event(() => PaymentReceived, x => x.CorrelateById(m => m.Message.OrderId));

        Initially(
            When(OrderSubmitted)
                .Then(ctx => ctx.Saga.CustomerName = ctx.Message.CustomerName)
                .TransitionTo(Submitted));

        During(Submitted,
            When(PaymentReceived)
                .TransitionTo(Paid)
                .Finalize());
    }
}
```

---

## Azure Service Bus

Managed брокер в Azure. Queues (point-to-point), Topics + Subscriptions (pub/sub), Sessions (ordered, grouped).

**Отличия от RabbitMQ:**
- Managed — нет инфраструктуры для управления
- Sessions — гарантированный порядок для группы сообщений
- Scheduled delivery — отправка в будущем
- Dead letter subqueue — встроенный
- Более дорогой, но меньше ops-работы

---

## Background Jobs

### Channel + BackgroundService (in-memory)

```csharp
public class BackgroundTaskQueue
{
    private readonly Channel<Func<CancellationToken, Task>> _channel =
        Channel.CreateBounded<Func<CancellationToken, Task>>(100);

    public async ValueTask EnqueueAsync(Func<CancellationToken, Task> workItem)
        => await _channel.Writer.WriteAsync(workItem);

    public IAsyncEnumerable<Func<CancellationToken, Task>> DequeueAllAsync(CancellationToken ct)
        => _channel.Reader.ReadAllAsync(ct);
}
```

**Нюанс:** при перезапуске приложения задачи теряются. Для persistence — Hangfire, Quartz.NET.

### Hangfire — persistent jobs

Dashboard, retry, scheduled, recurring. Хранение в БД (PostgreSQL, SQL Server, Redis).

### Quartz.NET — scheduling

Cron-based расписание, кластеризация, persistent job store.

---

## Паттерны

### Outbox Pattern

Запись события и бизнес-данных в одной транзакции. Отдельный процесс читает outbox-таблицу и публикует в брокер. Гарантирует at-least-once delivery без distributed transaction.

### Idempotent Consumer

Consumer может получить одно и то же сообщение несколько раз (retry, redelivery). Хранить processed message IDs, проверять перед обработкой.

---

## См. также

- [[Topics/Docker/docker-deploy|Docker и CI/CD]]
- [[Topics/Performance/dotnet-performance|.NET Performance]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
