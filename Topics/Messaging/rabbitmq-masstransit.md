# Очереди и Message Brokers

## Зачем

Асинхронная обработка, decoupling, масштабирование. Producer отправляет сообщение, Consumer обрабатывает. Retry, dead letter.

---

## RabbitMQ

AMQP broker. Exchanges (direct, topic, fanout) → Queues → Consumers. Долговечность (durable), acknowledgments.

**Библиотека**: RabbitMQ.Client или MassTransit (абстракция).

---

## MassTransit

Абстракция над RabbitMQ, Azure Service Bus, Amazon SQS. Mediator-подобный API. Consumers, Sagas, Retry policies.

```csharp
services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer>();
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("localhost", "/", h => { h.Username("guest"); h.Password("guest"); });
        cfg.ConfigureEndpoints(ctx);
    });
});
```

---

## Azure Service Bus

Managed в Azure. Queues, Topics (pub/sub), Sessions. Подходит для enterprise, интеграции с Azure.

---

## Background Jobs

HostedService + очередь в памяти (Channel) — для простых сценариев. Hangfire — persistent, dashboard. Quartz.NET — scheduling.

---

## См. также

- [[Topics/Docker/docker-deploy|Docker и CI/CD]]
- [[Topics/Performance/dotnet-performance|.NET Performance]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
