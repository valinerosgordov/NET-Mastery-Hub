---
tags: [distributed-systems, outbox, idempotency, saga, masstransit, cap, eventual-consistency, tech-lead]
level: Senior
date: 2026-08-02
---

# Distributed Systems Patterns в .NET — Outbox, Idempotency, Saga

## Что это, зачем и когда

### Что такое distributed system?
**Несколько процессов на разных машинах, общающихся по сети, ведущих себя как одна система с точки зрения пользователя.** Микросервисы, кластер БД, событийная архитектура с RabbitMQ — всё это distributed.

**Аналогия:** Одиночный процесс — это разговор с собой. Несколько потоков в процессе — разговор семьи за одним столом (можно слышать всех). Distributed system — это переписка по почте: письма теряются, доходят с задержкой, могут прийти не в том порядке, ответ может разойтись с ожиданием. И ты должен спроектировать систему которая всё равно работает.

### Главные сложности

| Что | В одиночной системе | В распределённой |
|-----|---------------------|------------------|
| Network failures | Их нет | Norma — сообщения теряются, retry'ятся, дублируются |
| Clock | Один источник времени | Часы расходятся между узлами |
| Транзакции | ACID через БД | Нет глобальной транзакции — нужен Saga / two-phase commit |
| Состояние | Точно знаешь | Snapshot отстаёт, eventual consistency |
| Failure detection | Process crash → ясно | Не отвечает = не работает или сеть лагает? |

### Когда применять эти паттерны
Когда у тебя **больше одного процесса, который должен оставаться согласованным**:
- Микросервисы общающиеся через RabbitMQ/Kafka
- Web API + database + redis cache
- API + email service + payment gateway
- Multi-region active-active

Когда **не нужно**: монолит с одной БД, один процесс, синхронные вызовы.

> [!warning]- MassTransit v9 — коммерческий (Q1 2026)
> Примеры в этом файле написаны на MassTransit **v8 (Apache 2.0)** — код валиден. Но **v9** (компания Massient) — платный: ~$400/мес SMB, ~$1200/мес enterprise; есть 100%-discount тир для организаций с выручкой < $1M/год. v8 получает только security-патчи, **EOL — конец 2026**. Альтернативы: **Wolverine** (MIT, ядро OSS — mediator + saga + outbox + брокеры одним инструментом), Rebus, Brighter, raw-клиенты брокеров. Карта решений — [[choosing-dependencies|Choosing Dependencies]].

---

## CAP-теорема и её практическое применение

**Теорема:** В распределённой системе при network partition можно гарантировать только два из трёх:

- **C**onsistency — все ноды видят одни и те же данные
- **A**vailability — каждый запрос получает ответ
- **P**artition tolerance — система продолжает работать при сетевых сбоях

**В реальности P неизбежен** — сеть может разорваться. Поэтому выбираешь между **CP** (консистентность + доступность только когда сеть в порядке) и **AP** (доступность всегда, но возможны устаревшие данные).

| | CP — выбираем consistency | AP — выбираем availability |
|--|---------------------------|---------------------------|
| Примеры | PostgreSQL primary-replica с sync repl, Zookeeper, etcd | DynamoDB, Cassandra, eventual consistency caches |
| Поведение при partition | Часть нод не отвечает (отказ в обслуживании) | Все ноды отвечают, но могут показывать старые данные |
| Когда | Деньги, инвентарь, бронирование | Лента новостей, рекомендации, аналитика |

### PACELC — расширение CAP

Ещё один параметр: при отсутствии partition (E — Else) — выбираешь между **L**atency и **C**onsistency. PostgreSQL с async-replication: PA/EL (если partition — оставляешь availability через replicas, в норме оптимизируешь latency, теряешь strict consistency для replicas).

> [!question]- **Интервью: SQL базы — это CP или AP?**
> Один инстанс — N/A (нет distributed). Master-replica с **synchronous** replication — CP (writes ждут реплики, при partition — недоступны). С **async** replication — AP (writes идут сразу, при partition реплики отстают). Большинство production-баз — async-replication как default + опциональный sync для критичных запросов. Postgres именно так.

---

## Идемпотентность

**Идемпотентная операция — повторный вызов с теми же параметрами не меняет результат.** Базовое требование надёжной distributed system.

### Зачем
В сети **сообщения дублируются, пропадают, retry'ятся**. Если "Создай заказ" не идемпотентно — retry даст 2 заказа. С идемпотентностью — два запроса дают тот же результат, второй просто возвращает ID первого.

### Идемпотентные ключи (Idempotency-Key)

```http
POST /api/orders
Idempotency-Key: f47ac10b-58cc-4372-a567-0e02b2c3d479
Content-Type: application/json

{ "items": [...], "amount": 100 }
```

### Реализация на ASP.NET Core

```csharp
public sealed class IdempotencyMiddleware(
    RequestDelegate next,
    IIdempotencyStore store,
    ILogger<IdempotencyMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (!HttpMethods.IsPost(context.Request.Method))
        {
            await next(context);
            return;
        }

        var key = context.Request.Headers["Idempotency-Key"].FirstOrDefault();
        if (string.IsNullOrEmpty(key))
        {
            await next(context);
            return;
        }

        // Check cache
        var cached = await store.GetAsync(key, context.RequestAborted);
        if (cached is not null)
        {
            context.Response.StatusCode = cached.StatusCode;
            await context.Response.WriteAsync(cached.Body, context.RequestAborted);
            return;
        }

        // Capture response
        var originalBody = context.Response.Body;
        using var ms = new MemoryStream();
        context.Response.Body = ms;

        try
        {
            await next(context);

            ms.Position = 0;
            var body = await new StreamReader(ms).ReadToEndAsync(context.RequestAborted);

            await store.SetAsync(key, new CachedResponse(context.Response.StatusCode, body),
                ttl: TimeSpan.FromHours(24), context.RequestAborted);

            ms.Position = 0;
            await ms.CopyToAsync(originalBody, context.RequestAborted);
        }
        finally
        {
            context.Response.Body = originalBody;
        }
    }
}

public interface IIdempotencyStore
{
    Task<CachedResponse?> GetAsync(string key, CancellationToken ct);
    Task SetAsync(string key, CachedResponse response, TimeSpan ttl, CancellationToken ct);
}

public sealed record CachedResponse(int StatusCode, string Body);

// Storage в Redis
public sealed class RedisIdempotencyStore(IConnectionMultiplexer redis) : IIdempotencyStore
{
    private readonly IDatabase _db = redis.GetDatabase();

    public async Task<CachedResponse?> GetAsync(string key, CancellationToken ct)
    {
        var json = await _db.StringGetAsync($"idem:{key}");
        return json.HasValue ? JsonSerializer.Deserialize<CachedResponse>(json!) : null;
    }

    public Task SetAsync(string key, CachedResponse response, TimeSpan ttl, CancellationToken ct)
    {
        var json = JsonSerializer.Serialize(response);
        return _db.StringSetAsync($"idem:{key}", json, expiry: ttl);
    }
}
```

### Idempotency на write через unique constraint

Альтернатива middleware: при создании сущности используй idempotency key как primary key или уникальный индекс. Дубликат → unique constraint violation → ловим как success (запрос уже обработан).

```csharp
public async Task<Result<Guid>> CreateOrderAsync(string idempotencyKey, OrderInput input, CancellationToken ct)
{
    try
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            IdempotencyKey = idempotencyKey,
            // ...
        };
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(ct);
        return Result.Ok(order.Id);
    }
    catch (DbUpdateException ex) when (IsDuplicate(ex))
    {
        // Кто-то уже создал. Возвращаем существующий
        var existing = await _db.Orders
            .Where(o => o.IdempotencyKey == idempotencyKey)
            .Select(o => o.Id)
            .FirstAsync(ct);
        return Result.Ok(existing);
    }
}
```

> [!question]- **Интервью: HTTP-методы и идемпотентность — что есть что?**
> - **GET** — идемпотентен, безопасен (no side effects)
> - **PUT** — идемпотентен (одинаковая операция, тот же результат)
> - **DELETE** — идемпотентен (повторный delete уже удалённого = ничего не происходит)
> - **POST** — **НЕ идемпотентен** (по умолчанию). Поэтому для POST нужен Idempotency-Key, чтобы не создавать дубликаты при retry
> - **PATCH** — может быть идемпотентен (если задаёт абсолютные значения) или нет (если increment)

---

## Exactly-once delivery — миф

**В distributed system невозможно достичь exactly-once delivery.** Можно только:

| | Что | Реализуемо |
|--|-----|-----------|
| At-most-once | Сообщение либо доставлено, либо нет (drop при ошибке) | Да, тривиально |
| At-least-once | Сообщение доставлено хотя бы раз, могут быть дубли | Да, через retries |
| Exactly-once **delivery** | Точно один раз | **НЕТ** |
| Exactly-once **processing** | Получатель выполнил эффект ровно один раз | Да — через at-least-once delivery + idempotency |

**Стандартный паттерн:** Кафка/RabbitMQ дают at-least-once, твой консьюмер — идемпотентен, эффект применяется один раз.

```csharp
// Consumer обрабатывает один раз даже при дубликатах
public class OrderShippedConsumer : IConsumer<OrderShipped>
{
    public async Task Consume(ConsumeContext<OrderShipped> ctx)
    {
        var msgId = ctx.MessageId!.Value;

        // Idempotency check — мы уже обработали это сообщение?
        if (await _processed.ExistsAsync(msgId, ctx.CancellationToken))
        {
            _logger.LogInformation("Skipping duplicate {MessageId}", msgId);
            return;
        }

        await using var tx = await _db.Database.BeginTransactionAsync(ctx.CancellationToken);

        // Effect
        await _db.Notifications.AddAsync(new Notification(...), ctx.CancellationToken);
        await _processed.AddAsync(msgId, ctx.CancellationToken);  // в той же транзакции!

        await _db.SaveChangesAsync(ctx.CancellationToken);
        await tx.CommitAsync(ctx.CancellationToken);
    }
}
```

Ключ — **обновить таблицу processed-сообщений и эффект в одной транзакции**. Иначе разрыв между ними даст либо дубликат, либо потерю.

---

## Outbox Pattern — solving the dual-write problem

### Проблема

```csharp
// ❌ Распределённая транзакция — нельзя
public async Task PlaceOrderAsync(Order order)
{
    await _db.Orders.AddAsync(order);   // 1. БД
    await _bus.Publish(new OrderPlaced(order.Id));  // 2. RabbitMQ
    await _db.SaveChangesAsync();
}
```

Что может пойти не так:
1. БД сохранила, RabbitMQ упал → email не отправлен, нет события для downstream'а
2. БД упала, RabbitMQ доставил → событие про несуществующий заказ
3. Распределённая транзакция (2PC) — медленная, не работает с большинством message broker'ов

### Решение — Outbox

В одной БД-транзакции пишем И сущность И событие в **outbox-таблицу** в той же БД. Отдельный процесс читает outbox и публикует в брокер. **Атомарность гарантирована БД.**

```sql
CREATE TABLE outbox_messages (
    id          uuid PRIMARY KEY,
    type        text NOT NULL,
    payload     jsonb NOT NULL,
    created_at  timestamptz DEFAULT now(),
    processed_at timestamptz
);

CREATE INDEX ON outbox_messages (processed_at) WHERE processed_at IS NULL;
```

```csharp
// Domain-логика — кладём событие в outbox в той же транзакции
public async Task PlaceOrderAsync(Order order)
{
    await _db.Orders.AddAsync(order);
    await _db.OutboxMessages.AddAsync(new OutboxMessage
    {
        Id = Guid.NewGuid(),
        Type = nameof(OrderPlaced),
        Payload = JsonSerializer.SerializeToDocument(new OrderPlaced(order.Id, order.Total)),
        CreatedAt = DateTime.UtcNow,
    });
    await _db.SaveChangesAsync();  // одна БД-транзакция
}

// Background service — публикует
public class OutboxPublisher(
    AppDbContext db,
    IBus bus,
    ILogger<OutboxPublisher> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var batch = await db.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .OrderBy(m => m.CreatedAt)
                .Take(100)
                .ToListAsync(ct);

            foreach (var msg in batch)
            {
                try
                {
                    var type = Type.GetType(msg.Type)!;
                    var evt = msg.Payload.Deserialize(type)!;

                    await bus.Publish(evt, evt.GetType(), c => c.MessageId = msg.Id, ct);

                    msg.ProcessedAt = DateTime.UtcNow;
                }
                catch (Exception ex)
                {
                    logger.LogError(ex, "Failed to publish {Id}, will retry", msg.Id);
                    // Не помечаем processed → retry в следующей итерации
                }
            }

            if (batch.Count > 0)
                await db.SaveChangesAsync(ct);

            await Task.Delay(TimeSpan.FromSeconds(1), ct);
        }
    }
}
```

### MassTransit Outbox — встроенный

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddEntityFrameworkOutbox<AppDbContext>(o =>
    {
        o.QueryDelay = TimeSpan.FromSeconds(1);
        o.UsePostgres();
        o.UseBusOutbox();  // публикация только после COMMIT
    });

    x.AddConsumers(typeof(Program).Assembly);
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host("rabbitmq");
        cfg.ConfigureEndpoints(ctx);
    });
});
```

Теперь любой `await _publisher.Publish(...)` внутри `_db.SaveChanges` сохраняется в outbox. Background-процесс MassTransit'а доставляет.

В NexusAI это **рекомендуемая** конфигурация — `UseBusOutbox()` стоит как baseline.

> [!question]- **Интервью: какие альтернативы Outbox для решения dual-write?**
> 1. **Transactional Outbox** (этот) — в той же БД, надёжно, простая реализация
> 2. **Change Data Capture (CDC)** — Debezium читает WAL Postgres, конвертирует commit'ы в Kafka events. Меньше boilerplate, но добавляет инфраструктуру (Kafka, Debezium)
> 3. **Saga / event-driven** — нет dual-write, всё через события (но сложнее проектировать)
> 4. **Event Sourcing** — события — единственный source of truth, БД-снимки строятся из событий

---

## Legacy migration — синхронизатор как постоянная подсистема

### Почему «временный мостик» становится вечным

Классический сценарий: рядом со старой (legacy) системой поднимают новую и хотят держать их в синхроне на период миграции — strangler fig, dual-run, постепенный cutover. Первая мысль — **CDC (Debezium читает WAL, льёт изменения в новую БД)**. Для чистого «один-к-одному копирования таблиц» это работает и действительно временно.

Ловушка в том, что **как только синхронизации требуется трансформация — CDC перестаёт быть достаточным, а синхронизатор перестаёт быть временным**. Трансформация появляется почти всегда:

- разные схемы — у legacy одна таблица `customers`, в новой системе `Customer` + `Address` + `ContactInfo` (нормализация/денормализация);
- разные идентификаторы — legacy `int`-автоинкремент, новая система `Guid`/ULID;
- разные инварианты — новая модель отвергает данные, которые legacy спокойно хранила (пустой email, отрицательная цена);
- бизнес-смысл изменения важнее самого diff'а строки — «цена снижена» это не то же самое, что `UPDATE price`.

CDC отдаёт **row-level diff без бизнес-смысла**: «строка 42, колонка price, было 100, стало 90». Восстанавливать намерение из diff'а на принимающей стороне — это парсить смысл из последствий, и оно ломается на каждой нетривиальной трансформации (мульти-табличные изменения в одной бизнес-операции приедут как несколько несвязанных row-событий). Поэтому при наличии трансформаций синхронизатор — это **полноценная долгоживущая подсистема со своим кодом, схемой, мониторингом и SLO**, а не скрипт, который удалят после cutover. Проектируй его соответственно.

> [!warning]

> «Временный синхронизатор» — самообман в 9 случаях из 10. Cutover буксует месяцами/годами, обе системы пишут параллельно дольше, чем планировалось, а часто двусторонняя синхронизация остаётся навсегда (legacy умеет то, что новую систему ещё не научили). Закладывай его как продукт: с тестами, версионированием контрактов и on-call, а не как throwaway-скрипт.

### Эмитить доменные события через outbox, а не читать чужой WAL

Источником синхронизации должны быть **доменные события** («`PriceReduced`», «`CustomerOnboarded`»), а не сырой row-diff. Бизнес-операция в системе-источнике в одной транзакции пишет и своё состояние, и доменное событие в outbox (см. раздел [[#Outbox Pattern — solving the dual-write problem]] выше). Синхронизатор — это обычный идемпотентный consumer этих событий, который применяет трансформацию и пишет в целевую систему.

Это ровно та же связка outbox + идемпотентный consumer, что выше, только consumer'ом выступает миграционный синхронизатор:

```csharp
public sealed class LegacyToNewSyncConsumer(
    NewDbContext db,
    IIdMappingStore idMap,
    ILogger<LegacyToNewSyncConsumer> logger) : IConsumer<PriceReduced>
{
    public async Task Consume(ConsumeContext<PriceReduced> ctx)
    {
        var ct = ctx.CancellationToken;
        var msgId = ctx.MessageId!.Value;

        await using var tx = await db.Database.BeginTransactionAsync(ct);

        // Idempotency — это at-least-once, дубликаты норма
        if (await db.InboxMessages.AnyAsync(m => m.Id == msgId, ct))
            return;

        // old -> new id mapping: legacy int -> new Guid
        var newId = await idMap.ResolveAsync("Product", ctx.Message.LegacyProductId, ct);
        if (newId is null)
        {
            // Порядок событий не гарантирован — родитель ещё не приехал.
            // Бросаем -> retry -> при исчерпании уйдёт в DLQ на разбор.
            throw new MappingNotReadyException(ctx.Message.LegacyProductId);
        }

        var product = await db.Products.FindAsync([newId.Value], ct);
        product!.ApplyPriceReduction(ctx.Message.NewPrice);   // трансформация = доменный метод

        await db.InboxMessages.AddAsync(new InboxMessage(msgId, DateTime.UtcNow), ct);
        await db.SaveChangesAsync(ct);
        await tx.CommitAsync(ct);
    }
}
```

Почему именно так, а не CDC напрямую:

- **намерение, а не diff** — событие несёт бизнес-смысл, трансформация в коде явная и тестируемая;
- **атомарность источника** — outbox гарантирует, что событие не разойдётся с состоянием legacy (dual-write problem решена в источнике);
- **идемпотентность** — at-least-once + inbox/unique constraint: повторная доставка не задваивает запись.

### Таблица маппинга идентификаторов old -> new

При смене схемы ID нужна **персистентная таблица соответствия**, а не вычисление на лету. Это и переводчик ссылок (FK из legacy указывают на старые ID), и журнал того, что уже мигрировано.

```sql
CREATE TABLE id_mapping (
    entity_type   text   NOT NULL,        -- 'Customer', 'Product', 'Order'
    legacy_id     text   NOT NULL,        -- исходный ключ как строка (int/composite)
    new_id        uuid   NOT NULL,
    migrated_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (entity_type, legacy_id),
    UNIQUE (entity_type, new_id)
);
```

Зачем таблица, а не детерминированная функция `legacy_id -> Guid`:

- **разрешение FK** — событие `OrderPlaced(legacyCustomerId)` нужно превратить в `new_customer_id`; без таблицы это невозможно для сущностей, созданных напрямую в новой системе;
- **двусторонний lookup** — при двусторонней синхронизации нужно ходить и new -> legacy (поэтому `UNIQUE` на оба столбца);
- **наблюдаемость прогресса** — `COUNT(*)` по `entity_type` показывает, сколько реально перенесено, и ловит пропуски;
- **детерминированный хэш ID не спасает** — он не покрывает сущности, рождённые в новой системе, и не даёт обратного направления.

`ResolveAsync` возвращающий `null` — это сигнал «событие пришло раньше своей сущности» (нет гарантии порядка между топиками/партициями). Корректная реакция — retry, затем DLQ, **не** молчаливый `INSERT` с выдуманным ID.

### Reconciliation: проверять КОНСИСТЕНТНОСТЬ ДАННЫХ, а не доставку сообщений

Главная ошибка — считать миграцию успешной по «все сообщения за-ack-аны / лаг consumer'а нулевой». **Ack доказывает доставку, а не корректность.** Трансформация могла отработать с багом, событие могло потеряться до outbox (дыра в инструментировании источника), `ApplyPriceReduction` мог тихо упасть на инварианте. Очередь при этом пустая и зелёная.

Поэтому нужен отдельный **reconciliation job**, который периодически сверяет фактические данные двух систем — независимо от шины:

```csharp
public sealed class ReconciliationJob(
    ILegacyReader legacy,
    NewDbContext newDb,
    IIdMappingStore idMap,
    IReconciliationReporter reporter,
    TimeProvider time) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromHours(1), time);
        while (await timer.WaitForNextTickAsync(ct))
        {
            // 1) Count drift: сколько строк должно быть и сколько есть
            var legacyCount = await legacy.CountProductsAsync(ct);
            var newCount = await newDb.Products.CountAsync(ct);

            // 2) Content drift: сверяем checksum по бизнес-полям на сэмпле/окне
            var mismatches = new List<ReconciliationMismatch>();
            await foreach (var row in legacy.StreamProductsAsync(ct))
            {
                var newId = await idMap.ResolveAsync("Product", row.Id, ct);
                if (newId is null) { mismatches.Add(ReconciliationMismatch.Missing(row.Id)); continue; }

                var target = await newDb.Products.FindAsync([newId.Value], ct);
                if (target is null) { mismatches.Add(ReconciliationMismatch.Missing(row.Id)); continue; }

                // checksum по БИЗНЕС-полям после трансформации, не побайтовое сравнение строк
                if (BusinessChecksum(row) != BusinessChecksum(target))
                    mismatches.Add(ReconciliationMismatch.Diverged(row.Id, newId.Value));
            }

            await reporter.PublishAsync(legacyCount, newCount, mismatches, ct);
        }
    }
}
```

Что проверяет reconciliation:

- **count drift** — расхождение числа сущностей (что-то не доехало или задвоилось);
- **content drift** — checksum по бизнес-полям *после* трансформации расходится (баг трансформации, потерянное событие, частичный сбой);
- **orphans / dangling FK** — ссылки в одной системе без цели в другой;
- **направление расхождения** — кто прав при двусторонней синхронизации (нужна политика разрешения конфликтов: last-writer-wins по версии, либо legacy-as-source-of-truth до cutover).

Reconciliation сравнивает **бизнес-смысл после трансформации**, поэтому это не побайтовое сравнение строк (схемы и так разные) — считай checksum по нормализованным бизнес-полям. Найденные расхождения — это либо автоматический re-emit пропущенного события, либо тикет на разбор, но в любом случае это **независимый от шины контур доверия**: именно он, а не зелёный дашборд RabbitMQ, даёт право нажать cutover.

> [!info]

> Метрика готовности к cutover — не «лаг consumer'а = 0», а «reconciliation N циклов подряд показывает нулевой drift». Первое говорит «почту разнесли», второе — «в обеих системах одни и те же данные». Путать их — классический способ переключиться на новую систему с тихо разъехавшимися данными.

---

## Inbox Pattern — для consumer'ов

Зеркальный паттерн на стороне консьюмера: в той же транзакции что и эффект, помечаешь сообщение как обработанное.

```csharp
public class OrderShippedConsumer : IConsumer<OrderShipped>
{
    public async Task Consume(ConsumeContext<OrderShipped> ctx)
    {
        var msgId = ctx.MessageId!.Value;

        await using var tx = await _db.Database.BeginTransactionAsync(ctx.CancellationToken);

        // Idempotency через inbox-таблицу
        var alreadyProcessed = await _db.InboxMessages.AnyAsync(m => m.Id == msgId, ctx.CancellationToken);
        if (alreadyProcessed) return;

        // Apply effect
        await _db.Notifications.AddAsync(new Notification(ctx.Message), ctx.CancellationToken);

        // Mark processed (in same transaction!)
        await _db.InboxMessages.AddAsync(new InboxMessage(msgId, DateTime.UtcNow), ctx.CancellationToken);

        await _db.SaveChangesAsync(ctx.CancellationToken);
        await tx.CommitAsync(ctx.CancellationToken);
    }
}
```

MassTransit `AddEntityFrameworkOutbox` автоматически также включает inbox для consumer'ов в том же DbContext'е.

---

## Saga Pattern

Saga — координация распределённой транзакции через **последовательность локальных транзакций + компенсирующих действий** при failure.

### Choreography vs Orchestration

| | Choreography | Orchestration |
|--|--------------|---------------|
| Координация | Каждый сервис подписан на события и реагирует | Центральный orchestrator (state machine) |
| Связность | Низкая (сервисы независимы) | Средняя (orchestrator знает все шаги) |
| Видимость flow | Сложно отследить (events разбросаны по логам) | Линейная история в state machine |
| Когда | Простые saga (2-3 шага) | Сложные (5+ шагов, conditional branches) |

### Choreography пример

```
Order Service           Payment Service          Shipping Service
─────────────────       ─────────────────        ────────────────
PlaceOrder ──────▶ OrderPlaced
                                       ▼
                                  Charge Card
                                       │
                  PaymentSucceeded ◀────┤
                  PaymentFailed   ◀────┤
                                       ▼
                                                 Ship Order
                                                  │
                                          OrderShipped ◀──
```

Каждый сервис обрабатывает event и испускает следующий. Если PaymentFailed — Order Service слушает и компенсирует (cancel order).

### Orchestration через MassTransit Saga

```csharp
public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = "";

    public Guid OrderId { get; set; }
    public decimal Total { get; set; }
    public Guid? PaymentId { get; set; }
}

public class OrderSaga : MassTransitStateMachine<OrderState>
{
    public State PendingPayment { get; private set; } = null!;
    public State PendingShipment { get; private set; } = null!;
    public State Completed { get; private set; } = null!;
    public State Failed { get; private set; } = null!;

    public Event<OrderPlaced> OrderPlaced { get; private set; } = null!;
    public Event<PaymentSucceeded> PaymentSucceeded { get; private set; } = null!;
    public Event<PaymentFailed> PaymentFailed { get; private set; } = null!;
    public Event<OrderShipped> OrderShipped { get; private set; } = null!;

    public OrderSaga()
    {
        InstanceState(x => x.CurrentState);

        Event(() => OrderPlaced, x => x.CorrelateById(c => c.Message.OrderId));
        Event(() => PaymentSucceeded, x => x.CorrelateById(c => c.Message.OrderId));
        Event(() => PaymentFailed, x => x.CorrelateById(c => c.Message.OrderId));
        Event(() => OrderShipped, x => x.CorrelateById(c => c.Message.OrderId));

        Initially(
            When(OrderPlaced)
                .Then(ctx => { ctx.Saga.OrderId = ctx.Message.OrderId; ctx.Saga.Total = ctx.Message.Total; })
                .Publish(ctx => new ChargeCard(ctx.Saga.OrderId, ctx.Saga.Total))
                .TransitionTo(PendingPayment));

        During(PendingPayment,
            When(PaymentSucceeded)
                .Then(ctx => ctx.Saga.PaymentId = ctx.Message.PaymentId)
                .Publish(ctx => new ShipOrder(ctx.Saga.OrderId))
                .TransitionTo(PendingShipment),
            When(PaymentFailed)
                .Publish(ctx => new CancelOrder(ctx.Saga.OrderId))
                .TransitionTo(Failed));

        During(PendingShipment,
            When(OrderShipped)
                .TransitionTo(Completed)
                .Finalize());

        SetCompletedWhenFinalized();
    }
}

// Регистрация
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

State machine хранит state в БД (через EF), MassTransit гарантирует что только одно событие обрабатывается на инстанс саги одновременно (через row-level locking).

### Compensating actions

Если payment failed после ship — нужна compensation: refund, return inventory, notify user.

```csharp
During(PendingShipment,
    When(ShipmentFailed)
        .Publish(ctx => new RefundPayment(ctx.Saga.PaymentId!.Value))
        .Publish(ctx => new CancelOrder(ctx.Saga.OrderId))
        .TransitionTo(Failed));
```

**Compensating транзакции — не идеальный rollback.** Они меняют мир обратно, но между forward и compensating действиями могло пройти время и другие действия (видимый эффект клиенту, отправленный email). Это часть Eventually Consistent дизайна — UI должен показывать "в обработке" пока saga не завершилась.

---

## Retry + DLQ (Dead Letter Queue)

### Exponential backoff + jitter

```csharp
cfg.UseMessageRetry(r =>
{
    r.Exponential(
        retryLimit: 5,
        minInterval: TimeSpan.FromSeconds(1),
        maxInterval: TimeSpan.FromMinutes(5),
        intervalDelta: TimeSpan.FromSeconds(2));
    // Дополнительно — jitter, чтобы 1000 retry не пошли одновременно
});
```

### Poison messages → DLQ

После исчерпания retry — сообщение уходит в _error queue. MassTransit делает это автоматически. Мониторинг DLQ — критично:

```csharp
// Custom consumer для error queue (для разбора poison messages)
public class ErrorMessageConsumer : IConsumer<Fault<OrderPlaced>>
{
    public async Task Consume(ConsumeContext<Fault<OrderPlaced>> ctx)
    {
        await _alertService.SendAsync(new AlertMessage
        {
            Title = "Poison message in OrderPlaced",
            Body = ctx.Message.Exceptions.First().Message,
            MessageId = ctx.Message.FaultMessageId,
        });

        // Опционально — сохранить в БД для manual review
        await _poisonStore.SaveAsync(ctx.Message);
    }
}
```

### Dashboard / monitoring

- Grafana панель: count of messages in `_error` queue, alert если > 0 на 5 минут
- Sentry / Application Insights: каждый Fault как отдельная error-trace
- Slack/Telegram notifications на любое попадание в DLQ

---

## Distributed locking

### Redis (RedLock-net)

```csharp
public sealed class DistributedLock(IConnectionMultiplexer redis)
{
    public async Task<bool> WithLockAsync(string key, TimeSpan ttl, Func<Task> action, CancellationToken ct)
    {
        var redLockFactory = RedLockFactory.Create(new[] { redis });
        await using var redLock = await redLockFactory.CreateLockAsync(
            resource: key,
            expiryTime: ttl,
            waitTime: TimeSpan.FromSeconds(5),
            retryTime: TimeSpan.FromMilliseconds(200),
            cancellationToken: ct);

        if (!redLock.IsAcquired) return false;

        await action();
        return true;
    }
}

// Usage
var locked = await _lock.WithLockAsync($"deck-generation:{deckId}", TimeSpan.FromMinutes(5),
    () => GenerateDeckAsync(deckId), ct);
```

### PostgreSQL advisory locks

```csharp
public async Task<T> WithAdvisoryLockAsync<T>(long lockId, Func<Task<T>> action, CancellationToken ct)
{
    await using var conn = await _dataSource.OpenConnectionAsync(ct);
    await using var tx = await conn.BeginTransactionAsync(ct);

    // pg_advisory_xact_lock — auto-release on commit/rollback
    await using (var cmd = new NpgsqlCommand($"SELECT pg_advisory_xact_lock({lockId})", conn, tx))
        await cmd.ExecuteNonQueryAsync(ct);

    try
    {
        var result = await action();
        await tx.CommitAsync(ct);
        return result;
    }
    catch
    {
        await tx.RollbackAsync(ct);
        throw;
    }
}
```

### Когда что

| | Redis (RedLock) | PG advisory lock |
|--|------------------|-------------------|
| Нужен Redis | Да | Нет |
| Производительность | Хорошо | Хорошо |
| Семантика lease | TTL, может exceed deadline | Auto-release on commit |
| Когда | Уже есть Redis | Уже есть Postgres, не хочется новой инфры |

---

## Distributed tracing

### OpenTelemetry context propagation

```csharp
// Producer — context инжектится в headers сообщения
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddSource("MassTransit")  // critical
        .AddOtlpExporter());

builder.Services.AddMassTransit(x =>
{
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.UseRawJsonSerializer();  // включает W3C trace context в headers
        cfg.ConfigureEndpoints(ctx);
    });
});
```

В Jaeger/Tempo увидишь trace, который пересекает границы сервисов: HTTP-запрос → consumer → outbox publish → другой consumer.

---

## Eventual consistency и read models (CQRS)

### Идея
Запись в одну модель (write model — нормализованная, ACID), чтение из другой (read model — денормализованная, оптимизирована под query). Sync через events.

```csharp
// Write model — order_aggregate
public class OrderCommandHandler
{
    public async Task PlaceAsync(PlaceOrderCommand cmd, CancellationToken ct)
    {
        var order = Order.Place(cmd.Items, cmd.Total);
        await _writeRepo.SaveAsync(order, ct);
        await _bus.Publish(new OrderPlaced(order.Id), ct);
    }
}

// Read model — материализованный для экрана "мои заказы"
public class OrderListProjection : IConsumer<OrderPlaced>, IConsumer<OrderShipped>
{
    public async Task Consume(ConsumeContext<OrderPlaced> ctx)
    {
        await _readDb.MyOrdersList.AddAsync(new MyOrderRow
        {
            OrderId = ctx.Message.OrderId,
            Status = "Placed",
            // ... денормализованные данные для UI
        }, ctx.CancellationToken);
        await _readDb.SaveChangesAsync(ctx.CancellationToken);
    }

    public async Task Consume(ConsumeContext<OrderShipped> ctx)
    {
        var row = await _readDb.MyOrdersList.FindAsync(ctx.Message.OrderId);
        if (row is not null)
        {
            row.Status = "Shipped";
            await _readDb.SaveChangesAsync(ctx.CancellationToken);
        }
    }
}
```

### Staleness budget

Read model отстаёт от write — на сколько допустимо? Устанавливай SLO:
- Юзеру важна актуальность баланса → < 1 секунда
- Аналитика по продажам за день → 5 минут OK
- Reporting → ежечасный пересчёт OK

UI должен **показывать состояние** ("Обрабатывается", "Готово") вместо притворства, что всё мгновенно.

---

## Versioning events

Event'ы — это контракт между сервисами. Старые версии могут быть в очередях, новые — в коде. Что делать с разногласиями?

### Forward / backward compatibility
- **Forward compatible** — новый код понимает старые события (это must-have)
- **Backward compatible** — старый код понимает новые (для постепенного rollout)

### Правила безопасных изменений

✅ **Можно добавить:**
- Новое необязательное поле (default value при десериализации)
- Новый event type (consumer'ы которые не подписаны — игнорят)

❌ **Нельзя:**
- Удалять поля (старые consumer'ы упадут)
- Переименовывать поля
- Менять тип поля (string → int)
- Удалять event types если есть unprocessed messages

### Upcasting

Старая версия:
```json
{ "type": "OrderPlaced.v1", "orderId": "...", "total": 100 }
```

Новая (добавили currency):
```json
{ "type": "OrderPlaced.v2", "orderId": "...", "total": 100, "currency": "USD" }
```

Upcaster при чтении:
```csharp
public class OrderPlacedUpcaster : IEventUpcaster
{
    public bool CanUpcast(string eventType) => eventType == "OrderPlaced.v1";

    public object Upcast(JsonElement payload)
    {
        return new OrderPlacedV2
        {
            OrderId = payload.GetProperty("orderId").GetGuid(),
            Total = payload.GetProperty("total").GetDecimal(),
            Currency = "USD",  // default для legacy
        };
    }
}
```

---

## Monotonic version — детекция дыр и переупорядочивания на apply

Предыдущий раздел — про **схему** события (как `v1` дочитать до `v2`). Этот — про другое: про **порядковый номер состояния**, который позволяет получателю на стороне apply *обнаружить*, что событие пришло не в том порядке или с дырой. Это разные оси, и их легко спутать.

### Почему «Kafka же сохраняет порядок» — ложное успокоение

Kafka (и любой партиционированный лог) гарантирует порядок **только внутри одной партиции**. Как только сущность переезжает между партициями (re-keying, изменение `partition count`, два топика про одну сущность, producer с несколькими in-flight-запросами и retry без идемпотентного producer'а) — порядок между событиями одной сущности **не гарантирован** глобально. RabbitMQ без single active consumer и при конкурентных prefetch'ах — тем более.

Опасность не в дубликате (его ловит [[#Inbox Pattern — для consumer'ов|inbox/идемпотентность]]), а в **тихой перезаписи свежего состояния старым**:

> [!danger]
> Сценарий: `PriceChanged(price=90)` и следом `PriceChanged(price=100)` уезжают в разные партиции; consumer обрабатывает их в порядке `100` затем `90`. Идемпотентность не спасает — это **разные** сообщения с разными `MessageId`. Read-model останется на `90` навсегда. Сигнала нет: очередь пуста, лаг нулевой, дашборд зелёный. Это самый коварный класс багов eventual consistency — корректность теряется бесшумно.

### Решение — монотонный `Version` в событии + reject-older-than-current на apply

Источник истины ведёт монотонно растущий счётчик версии агрегата и **штампует его в каждое событие**. Получатель хранит «последнюю применённую версию» рядом с проекцией и применяет событие, **только если оно строго новее** текущего; всё «не новее текущего» отбрасывается как stale, а скачок дальше чем на единицу — это **дыра** (пропущено промежуточное событие), и она должна быть видимой.

```sql
-- write-side: версия живёт на агрегате, инкремент в той же транзакции, что и изменение
ALTER TABLE products ADD COLUMN version bigint NOT NULL DEFAULT 0;

-- read-side: проекция хранит версию последнего применённого события
ALTER TABLE product_read_model ADD COLUMN applied_version bigint NOT NULL DEFAULT 0;
```

```csharp
// Событие несёт монотонную версию агрегата на момент изменения.
// Это НЕ schema-version (.v1/.v2), а порядковый номер состояния сущности.
public sealed record PriceChanged(Guid ProductId, long Version, decimal NewPrice);

// write-side: версия инкрементится атомарно с изменением состояния
public Result PriceChange(decimal newPrice)
{
    if (newPrice <= 0)
        return Result.Fail(Error.Validation("Price must be positive"));

    Price = newPrice;
    Version++;                                  // монотонно, в той же транзакции
    Raise(new PriceChanged(Id, Version, newPrice));
    return Result.Ok();
}
```

```csharp
public sealed class ProductPriceProjection(
    ReadDbContext db,
    ILogger<ProductPriceProjection> logger) : IConsumer<PriceChanged>
{
    public async Task Consume(ConsumeContext<PriceChanged> ctx)
    {
        var ct = ctx.CancellationToken;
        var msg = ctx.Message;

        var row = await db.ProductReadModel.FindAsync([msg.ProductId], ct);
        if (row is null)
        {
            // версия > 1 у несуществующей строки = тоже дыра (пропущен create)
            logger.LogWarning(
                "Gap: PriceChanged v{Version} for unknown product {ProductId}",
                msg.Version, msg.ProductId);
            throw new OutOfOrderEventException(msg.ProductId, msg.Version);
        }

        // reject-older-than-current: ядро защиты от тихой перезаписи
        if (msg.Version <= row.AppliedVersion)
        {
            // stale/duplicate-by-order — НЕ ошибка, штатно отбрасываем
            logger.LogDebug(
                "Stale PriceChanged v{Version} <= applied v{Applied} for {ProductId}, skip",
                msg.Version, row.AppliedVersion, msg.ProductId);
            return;
        }

        if (msg.Version > row.AppliedVersion + 1)
        {
            // ДЫРА: между applied и текущим потеряно/задержано событие.
            // Бросаем -> retry (отставшее событие может приехать) -> DLQ + alert.
            logger.LogWarning(
                "Gap on {ProductId}: applied v{Applied}, got v{Version} (missing {Missing})",
                msg.ProductId, row.AppliedVersion, msg.Version, msg.Version - row.AppliedVersion - 1);
            throw new OutOfOrderEventException(msg.ProductId, msg.Version);
        }

        // monotonic forward step: применяем
        row.Price = msg.NewPrice;
        row.AppliedVersion = msg.Version;
        await db.SaveChangesAsync(ct);
    }
}
```

Три ветки на apply — это и есть весь паттерн:

| Условие на `Version` | Что это | Реакция |
|---|---|---|
| `<= applied` | дубликат по порядку / отставшее старое | штатный skip, не ошибка |
| `== applied + 1` | нормальный монотонный шаг | apply + сдвиг `applied` |
| `> applied + 1` | **дыра** — пропущено промежуточное | throw \-\> retry \-\> DLQ + alert |

### Чем версия отличается от idempotency-ключа

Это **ортогональные** механизмы, нужны оба:

- **Idempotency / inbox** отвечает на «видел ли я *именно это сообщение*?» — защита от повторной доставки одного и того же события (`MessageId`).
- **Version** отвечает на «новее ли это состояние того, что у меня уже применено?» — защита от переупорядочивания и пропусков *разных* событий одной сущности.

Дубликат имеет тот же `MessageId`, но и ту же `Version` — его поймает inbox. Переставленные события имеют **разные** `MessageId` и **разные** `Version` — их пропустит inbox, но поймает reject-older-than-current. Поэтому в полноценном consumer'е стоят обе проверки.

> [!tip]
> Не изобретай глобальный монотонный счётчик через `DateTime.UtcNow` — часы узлов расходятся (см. CAP-раздел про clock skew), два события в одну миллисекунду неразличимы, а NTP-коррекция может пойти назад. Версия должна быть **per-aggregate** и инкрементиться **в той же транзакции**, что и изменение (тот же optimistic-concurrency токен, что `xmin`/`rowversion` в EF). Тогда она строго монотонна по определению.

---

## Eventual consistency как SLO — панель метрик, а не «вроде догоняет»

[[#Staleness budget|Staleness budget]] выше задал *идею* «read-model отстаёт допустимо на N секунд». Этот раздел превращает её в **измеримый SLO с дашбордом и порогом, пробитие которого — инцидент**. Без чисел «eventual» означает «когда-нибудь, может быть» — а это не свойство системы, а отсутствие свойства.

### Почему один consumer-lag врёт

Самая частая ошибка наблюдаемости — мерить только **лаг consumer'а** (offset producer'а минус committed offset). Он отвечает на «сколько сообщений не вычитано», но **не** на «насколько устарели данные, которые видит пользователь»:

> [!warning]
> Consumer-lag может быть **нулевым при катастрофической staleness**. Consumer бодро вычитывает сообщения и сразу их ack'ает, но downstream-apply (запись в read-model, вызов внешнего API) тормозит, или проекция тихо роняет события на инварианте, или событие вообще не доехало до брокера (дыра в outbox источника). Очередь пустая, лаг нулевой, а read-model отстаёт на минуты. Один lag-график — это «почту разнесли», а не «данные совпали» (та же ловушка, что в [[#Reconciliation: проверять КОНСИСТЕНТНОСТЬ ДАННЫХ, а не доставку сообщений|reconciliation]]).

Поэтому SLO измеряется **сквозной задержкой и фактической свежестью данных**, а не глубиной очереди.

### Панель из пяти сигналов

| Метрика | Что измеряет | Как считать | Тип |
|---|---|---|---|
| **Outbox age** | возраст самого старого неотправленного события в outbox | `now() - MIN(created_at) WHERE processed_at IS NULL` | gauge, сек |
| **Apply lag** | возраст события в момент применения (E2E задержка) | `applied_at - event.OccurredAt`, гистограмма p50/p95/p99 | histogram, сек |
| **Projection errors** | проекции, упавшие на apply (включая дыры по версии) | счётчик исключений consumer'а по типу | counter |
| **DLQ size** | сколько сообщений осело в `_error` после исчерпания retry | глубина error-очереди | gauge |
| **Read-model freshness** | расхождение read-модели и write-модели по факту | `write.version - read.applied_version` на сэмпле | gauge |

Ключевые две — `apply lag` (а не consumer lag) и `read-model freshness`. Первая ловит медленный downstream при пустой очереди; вторая — это «mini-reconciliation» в реальном времени: если `applied_version` отстаёт от `version` источника, данные расходятся независимо от того, что говорит шина.

```csharp
// Apply lag и freshness снимаются в самом consumer'е — там, где видна E2E задержка.
public sealed class ProjectionMetrics
{
    private readonly Histogram<double> _applyLag = Meter.CreateHistogram<double>(
        "projection.apply_lag", unit: "s",
        description: "Возраст события в момент применения (end-to-end)");

    private static readonly Meter Meter = new("ReadModel.Projections", "1.0");

    public void RecordApply(DateTimeOffset occurredAt, TimeProvider time)
    {
        var lag = (time.GetUtcNow() - occurredAt).TotalSeconds;
        _applyLag.Record(lag);   // p95/p99 этой гистограммы — основа SLO-алерта
    }
}
```

```csharp
// Outbox age и DLQ size — gauge через ObservableGauge, опрашиваются периодически.
public sealed class OutboxHealthMetrics(IServiceScopeFactory scopes, TimeProvider time)
{
    private static readonly Meter Meter = new("Outbox.Health", "1.0");

    public OutboxHealthMetrics RegisterGauges()
    {
        Meter.CreateObservableGauge("outbox.oldest_pending_age", () =>
        {
            using var scope = scopes.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            var oldest = db.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .Min(m => (DateTimeOffset?)m.CreatedAt);
            return oldest is null ? 0d : (time.GetUtcNow() - oldest.Value).TotalSeconds;
        }, unit: "s");

        return this;
    }
}
```

### Нормальное окно лага и порог инцидента

SLO — это **число и реакция на его пробитие**, не «должно быть быстро». Зафиксируй для каждой проекции нормальное окно и эскалацию:

| Read-model | Нормальное окно (`apply lag` p95) | Warn | Инцидент (paging) |
|---|---|---|---|
| Баланс / доступный остаток | < 2 с | > 5 с на 1 мин | > 15 с на 2 мин **или** DLQ \> 0 |
| Лента заказов «мои заказы» | < 10 с | > 30 с на 5 мин | > 2 мин на 5 мин |
| Аналитика / отчёты | < 5 мин | > 15 мин | > 1 ч |

Правила, которые превращают таблицу в работающий SLO:

- **Алерт на `apply lag` p95/p99, не на consumer lag** — именно сквозная задержка отражает то, что видит пользователь.
- **`DLQ size > 0` — почти всегда сразу инцидент**: после исчерпания retry сообщение не применится само, и для версионированных проекций это означает **дыру**, которая не закроется без вмешательства.
- **`outbox age` растёт при нулевом consumer lag** = застрял publisher в источнике (см. pitfall «Outbox processor застрял»), события ещё даже не в шине — consumer-метрики этого вообще не видят.
- **`read-model freshness > 0` дольше окна** при пустой очереди = тихое расхождение (потерянное событие или упавший apply) — триггер на reconciliation/replay, а не на «подождём, догонит».

> [!info]
> Граница «warn vs инцидент» — продуктовое решение, не техническое: сколько секунд устаревшего баланса бизнес готов показать пользователю. Инженер задаёт окно вместе с владельцем продукта и **фиксирует его в том же ADR, что и event-versioning-стратегию**. SLO без записанного порога и без runbook на пробитие — это не SLO, а график, на который никто не смотрит.

---

## Production checklist

- [ ] Все POST endpoint'ы поддерживают Idempotency-Key
- [ ] Все consumer'ы идемпотентны (inbox таблица или unique constraint)
- [ ] Outbox pattern для всех "save + publish" сценариев
- [ ] Saga state хранится в БД с pessimistic locking
- [ ] Retry + DLQ настроены для всех consumer'ов
- [ ] Monitoring: alert если в DLQ появляются сообщения
- [ ] OpenTelemetry tracing по всем границам сервисов
- [ ] Distributed locks с TTL (предотвращение зависших lock'ов)
- [ ] Event versioning стратегия описана в ADR
- [ ] События несут монотонный per-aggregate `Version`; проекции делают reject-older-than-current и сигналят о дырах
- [ ] Read models имеют документированный staleness SLO с порогом-инцидентом (alert на apply-lag, не на consumer-lag)
- [ ] Дашборд eventual-consistency: outbox-age, apply-lag p95/p99, projection errors, DLQ size, read-model freshness
- [ ] Compensating actions определены для всех saga
- [ ] Tests: chaos testing с kill -9 на одном из сервисов
- [ ] Tests: replay сообщений из DLQ работает

---

## Common pitfalls

### 1. Outbox processor застрял
Один зависший processing блокирует весь FIFO outbox. Решение: timeout на processing + skip с alert.

### 2. Saga зависает в transient state
Forgot edge case или event-loss → saga в state PendingPayment навсегда. Решение: timeout-handlers (`Schedule(...)` в MassTransit), бизнес-правило "если 30 мин нет update'а — auto-cancel".

### 3. Idempotency keys без TTL
Таблица растёт бесконечно. Поставь cleanup job (старше 7 дней удаляем).

### 4. Distributed lock без timeout
Процесс упал, lock не освободился. Всегда TTL на lock'е. Если задача длится дольше — extend через keep-alive.

### 5. Не пропагируется correlation ID
Логи разных сервисов нельзя связать в один request. Решение: OpenTelemetry или хотя бы вручную через `X-Correlation-Id` header.

### 6. Single point of failure — один RabbitMQ
Cluster (3 nodes минимум для quorum). Mirrored queues для критичных. И backup стратегия.

### 7. Кафка/RabbitMQ как event store
Они — transport, не storage. Если нужен event sourcing — выделенное решение (Marten, KurrentDB — ex-EventStoreDB).

### 8. Дубликаты "одинаковые но не идентичные"
Внутри сообщения timestamp `DateTime.UtcNow` — каждый retry имеет другой timestamp → различия в payload, но семантически одно. Hash по бизнес-полям, не по всему JSON.

---

## См. также

- [[cqrs-mediatr|CQRS и MediatR]] — Command/Query разделение
- [[ddd|DDD на практике]] — domain events, aggregates как boundary
- [[messaging|Messaging]] — RabbitMQ, MassTransit основы
- [[resilience|Resilience и HttpClient]] — Polly v8, circuit breaker, retry
- [[postgresql-deep|PostgreSQL Deep]] — advisory locks, row-version, transactions
- [[observability|Logging и Observability]] — OpenTelemetry для distributed tracing

## Reading list

- **Designing Data-Intensive Applications** — Martin Kleppmann (must-read для distributed systems)
- **Microservices Patterns** — Chris Richardson (Saga, Outbox, CQRS на пальцах)
- **Release It!** — Michael Nygard (паттерны устойчивости — bulkhead, circuit breaker, timeouts)
- **MassTransit docs** — masstransit.io (особенно Saga state machines, Outbox, RetryPolicy) — ⚠️ v9 коммерческий, v8 Apache 2.0 security-only до конца 2026
- **Wolverine docs** — wolverine.netlify.app (MIT-альтернатива: mediator + saga + outbox)
- **Distributed Systems for Fun and Profit** — book.mixu.net/distsys/ (короткая бесплатная книга)
- **Jepsen reports** — jepsen.io (как реальные БД ломаются под partitions)
- **Event Sourcing на C#** — github.com/oskardudycz/EventSourcing.NetCore (примеры)
