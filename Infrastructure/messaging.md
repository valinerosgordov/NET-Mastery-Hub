---
tags: [rabbitmq, masstransit, messaging, message-broker]
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
| EmailService упал → API тоже ломается | EmailService упал → сообщение ждёт в очереди |
| EmailService медленный → пользователь ждёт | API отвечает за 50мс, email отправится в фоне |
| Нет retry — письмо потерялось | Автоматический retry, Dead Letter Queue |

### Когда использовать?
- **Email/SMS уведомления** — не блокировать пользователя
- **Обработка файлов** — resize, конвертация, генерация PDF
- **Интеграция между сервисами** — один сервис не знает о другом
- **Event-driven архитектура** — OrderCreated → уведомление + склад + оплата
- **Тяжёлые операции** — всё, что занимает > 1 секунды

### Когда НЕ нужен?
- Простое CRUD-приложение с одним сервером
- Синхронный ответ обязателен (пользователь ждёт результат)
- Проект < 3 месяцев / MVP (избыточная сложность)

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
