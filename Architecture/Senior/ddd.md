---
tags: [ddd, value-objects, aggregate-root, domain-events, entity]
level: Senior
date: 2026-06-28
---

# DDD — Domain-Driven Design на практике

## Что это, зачем и когда

### Что такое DDD?
**Подход к проектированию, где код отражает бизнес-логику.** Не «таблицы и CRUD», а «заказ можно отменить только если он не отправлен». Бизнес-правила живут в доменных объектах, а не размазаны по контроллерам и сервисам.

**Аналогия:** Обычный код — это список SQL-запросов с `if`-ами. DDD — это разговор с бизнесом: «Клиент создаёт заказ → добавляет товары → оплачивает → заказ отправляется». Код читается как бизнес-процесс.

### Зачем?

| Без DDD (Anemic Model) | С DDD (Rich Domain Model) |
|------------------------|--------------------------|
| `orderService.Cancel(orderId)` — логика в сервисе | `order.Cancel()` — логика в самом объекте |
| Валидация разбросана по контроллерам | Валидация в Value Object при создании |
| `new Order { Status = "Active" }` — можно создать невалидный | Приватный конструктор — невалидный объект невозможен |
| Ошибки через `throw new Exception("...")` | `Result<T>` — ошибки как значения |
| Нет событий — всё через прямые вызовы | Domain Events: `OrderCreated` → отправить email, обновить статистику |

### Когда нужен DDD?

| Ситуация | DDD? | Почему |
|----------|------|--------|
| Сложная бизнес-логика (финансы, логистика, медицина) | **Да** | Много правил, состояний, зависимостей |
| Простой CRUD (блог, TODO-app) | **Нет** | Оверкилл, достаточно сервисов |
| Долгоживущий проект, команда > 3 человек | **Да** | Границы модулей и единый язык спасают от хаоса |
| Прототип / MVP | **Нет** | Сначала проверить гипотезу, потом рефакторить |

---

---

## Strategic DDD — Bounded Contexts и Ubiquitous Language

> **Tactical DDD (Entity, VO, Aggregate)** — это про **код**.
> **Strategic DDD (Bounded Context, Ubiquitous Language, Context Maps)** — это про **границы и общение между командами**.
> На Senior interview спрашивают именно strategic. Без него ответ "что такое DDD" неполный.

### Зачем strategic DDD

В большой системе одно и то же слово в разных частях значит **разное**. "Order" в e-commerce:
- В каталоге: line item с product, quantity, price
- В платежах: транзакция с card, amount, status
- В доставке: упакованная коробка с address, tracking number
- В аналитике: запись в data warehouse с conversion attribution

Если все используют **одну модель `Order`** — она пухнет до 50 полей, половина ненужных в каждом конкретном context. Появляются `if (context == "shipping")` ветки. Coupling через данные.

**Strategic DDD говорит:** не пытайся слепить одну модель. Раздели на **Bounded Contexts**, в каждом — свой `Order` со своим smaller scope.

### Ubiquitous Language — единый язык

**Внутри Bounded Context** все используют одни термины: business, dev, тесты, БД-схема, API.

```
Плохо (нет ubiquitous language):
- Бизнес: "клиент покупает курс"
- Frontend: const buyer = ...
- Backend domain: class Customer { ... PurchaseProduct(...) }
- БД таблица: users
- API: POST /api/orders
- Логи: "user_id=42 placed order"
```

Каждый слой имеет **свой словарь**. Communication ломается, баги в переводе.

```
Хорошо (ubiquitous language в контексте Sales):
- Бизнес: "Customer enrolls in Course"
- Frontend: const customer = ...; customer.enroll(course)
- Backend: class Customer { void EnrollIn(Course course) }
- БД таблица: Customers, CourseEnrollments
- API: POST /api/customers/{id}/enrollments
- Логи: "Customer 42 enrolled in Course 7"
```

Один термин — одно значение. Code = бизнес-описание. Документация не нужна.

> [!question]- **Интервью: что такое Ubiquitous Language?**
> Единый словарь между бизнесом, разработчиками и кодом **внутри одного Bounded Context**. Если бизнес говорит "Subscription expires" — значит в коде `Subscription.Expire()`, в БД `expires_at`, в API `POST /subscriptions/{id}/expire`, в логах `"Subscription 42 expired"`. Не "User", "Account", "Membership" в разных местах для одного и того же. **Boundary важна** — в другом Bounded Context (например Billing) тот же объект может называться `Invoice` со своими методами. Это нормально, главное чтобы внутри одного BC терминология единая.

### Bounded Context — границы модели

**Bounded Context (BC)** — explicit boundary, внутри которого domain model (модель + язык) consistent. За границей — другая модель, другой язык, другое значение слов.

#### Пример: e-commerce

```
┌──────────────────────────────────────────────────────────────────┐
│ E-commerce system                                                 │
│                                                                    │
│  ┌──────────────────┐  ┌────────────────┐  ┌──────────────────┐ │
│  │ Catalog Context  │  │ Sales Context  │  │ Shipping Context │ │
│  │                   │  │                │  │                   │ │
│  │ Product:          │  │ Order:         │  │ Package:          │ │
│  │ - Id              │  │ - Id           │  │ - Id              │ │
│  │ - Title           │  │ - LineItems    │  │ - OrderId         │ │
│  │ - Description     │  │ - Total        │  │ - Address         │ │
│  │ - Images          │  │ - PaymentStat  │  │ - Tracking        │ │
│  │ - Categories      │  │ - CustomerId   │  │ - Weight, Size    │ │
│  │ - SEO meta        │  │                │  │ - Carrier         │ │
│  │ - Reviews         │  │ Customer:      │  │                   │ │
│  │                   │  │ - Id           │  │ Recipient:        │ │
│  │                   │  │ - Email        │  │ - Name, Phone     │ │
│  │                   │  │ - PaymentInfo  │  │ - Address         │ │
│  └──────────────────┘  └────────────────┘  └──────────────────┘ │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

В каждом BC:
- **Своя модель** (Product в Catalog ≠ "что-то в корзине" в Sales)
- **Свой язык** (Catalog: "Product", Sales: "LineItem", Shipping: "Item")
- **Своя БД-схема** (или хотя бы своя schema/namespace)
- **Своя команда** (often) или **свой module**

#### Как определять Bounded Context

```
Признак отдельного BC:
1. Разный язык бизнеса для одной "сущности"
   "Customer" в Sales имеет PaymentMethod
   "Customer" в Support имеет TicketHistory
   → Разные BC

2. Разные actors / users используют
   Catalog: editors добавляют товары
   Sales: customers покупают
   Shipping: warehouse staff пакует
   → Разные BC

3. Разные lifecycle / consistency boundaries
   Order создаётся → платится → отправляется → доставляется
   На каждом этапе разные business rules → возможно разные BC

4. Разные команды владеют
   Если код двух фичей трогают разные команды → подсказка к BC

5. Разная technology rationale
   Catalog нужен full-text search → Elasticsearch
   Sales нужны транзакции → PostgreSQL
   Analytics нужна column store → ClickHouse
   → Технические разные BC
```

#### Bounded Context ≠ Microservice

Распространённое заблуждение. **BC — логическая граница**, microservice — physical deployment unit.

| | Bounded Context | Microservice |
|--|-----------------|--------------|
| Что | Domain boundary (логика) | Deployment boundary (process) |
| Кто решает | Бизнес-аналитик + tech lead | DevOps + tech lead |
| Может ли BC быть внутри monolith | **ДА** — Modular Monolith | — |
| Один BC = один MS? | Часто, но не обязательно | — |
| Два BC в одном MS? | Возможно если они sharing infrastructure | — |

```
Правильно:
1. Сначала определи Bounded Contexts (domain modeling)
2. Потом реши: monolith с modules, или microservices
3. Microservices — это deployment-выбор, не domain-выбор

Неправильно:
1. "Делаем microservices"
2. Делим как попало по техническим axes
3. Получаем distributed monolith
```

> [!question]- **Интервью: Bounded Context vs Microservice?**
> **Bounded Context** — domain boundary (один language, одна model). **Microservice** — deployment boundary (отдельный process, БД, команда). BC можно implement как microservice, но не обязательно — Modular Monolith тоже валиден. Один microservice может содержать несколько BC если они tightly related. Главное правило: **сначала определяешь BC через domain analysis, потом решаешь deployment strategy**. Иначе получишь distributed monolith — все недостатки microservices без преимуществ.

### Context Map — как Bounded Contexts общаются

Когда есть несколько BC, важно **explicitly** определить как они общаются. Eric Evans описал 7 типов отношений:

#### 1. Partnership (партнёрство)
Две команды работают вместе на одну цель, координируют изменения.

```
Sales BC ←→ Inventory BC
Любое изменение в одном — синхронизация с другим
Высокий coupling между командами
```

#### 2. Shared Kernel (общее ядро)
Часть domain model используется обоими BC. **Опасно** — изменения требуют согласования.

```
Sales BC ┐
         ├── Shared: Customer (Id, Email basic info)
Support BC ┘
```

Используй **минимально**. Только для truly shared concepts.

#### 3. Customer-Supplier (заказчик-поставщик)
Upstream (supplier) поставляет данные/сервисы downstream (customer). Downstream может влиять на upstream API, но supplier владеет реализацией.

```
Catalog (upstream) → Sales (downstream)
Sales просит "нужны товары с ценой"
Catalog приоритизирует это в roadmap
```

#### 4. Conformist (конформист)
Downstream **просто принимает** API upstream без возможности влиять. Если upstream поменяет API — downstream подстроится.

```
External CRM (upstream) → Our Sales (downstream conformist)
Мы не контролируем CRM
Просто адаптируемся под их API
```

#### 5. Anti-Corruption Layer (ACL) — защитный слой ⭐

Downstream **изолирует** свою domain model от upstream. ACL **переводит** upstream concepts в свой ubiquitous language.

```
Legacy CRM (chaos) → ACL (translator) → Clean Sales BC

В ACL:
- Adapter — реализация upstream API (HTTP client, etc.)
- Translator — мапит legacy DTOs → clean domain models
- Facade — простой interface для нашего BC
```

```csharp
// ACL для legacy CRM
public sealed class LegacyCrmAdapter : ICustomerRepository
{
    private readonly LegacyCrmHttpClient _client;

    public async Task<Customer?> GetByIdAsync(CustomerId id, CancellationToken ct)
    {
        // Грязный legacy DTO — приходит как есть
        var legacyDto = await _client.GetUserByLegacyIdAsync(id.Value, ct);
        if (legacyDto == null) return null;

        // Translation в наш clean domain model
        return new Customer(
            new CustomerId(legacyDto.UserPk),
            Email.Create(legacyDto.EmailAddr ?? legacyDto.AltEmail).Value!,
            FullName.Create(legacyDto.FName, legacyDto.LName).Value!
        );
    }
}

// Наш domain не знает про legacy CRM!
public sealed class CustomerService(ICustomerRepository repository)
{
    public async Task<Result<Customer>> GetCustomerAsync(CustomerId id, CancellationToken ct)
    {
        var customer = await repository.GetByIdAsync(id, ct);
        return customer is null
            ? Result<Customer>.Fail(Errors.NotFound("Customer", id))
            : Result<Customer>.Ok(customer);
    }
}
```

**Когда обязательно ACL:**
- Legacy systems (старый CRM, ERP)
- External APIs которые ты не контролируешь
- Third-party services (Stripe, Twilio, SendGrid)
- Любая интеграция где их model ≠ твоя model

**ACL — самый частый strategic DDD pattern в реальной работе.**

#### 6. Open Host Service (OHS)
Upstream предоставляет stable API для multiple downstream. Не подстраивается под одного конкретного.

```
Public Catalog API → multiple consumers (web, mobile, partners)
API stable, versioned
Catalog не знает кто consumer
```

#### 7. Published Language
Standard format для общения между BC. Часто JSON Schema, Protobuf, Avro.

```
OrderCreated event published as Protobuf schema
Все consumers parse the same
Schema versioned
```

#### Separate Ways
Два BC решают **не интегрироваться**. Дублируем данные если нужно.

```
Marketing BC и Sales BC оба хранят Customer
Нет общего языка, нет ACL
Eventual consistency через events если требуется
```

> [!question]- **Интервью: что такое Anti-Corruption Layer?**
> Защитный слой между нашим Bounded Context и **внешней системой** (legacy / third-party / другой BC с непохожим языком). Состоит из: 1) **Adapter** — технический клиент (HTTP, gRPC). 2) **Translator** — мапит chaos external DTOs в чистый domain model. 3) **Facade** — clean interface для нашего code. **Зачем**: внешние API могут быть кошмарны (legacy CRM с полями `usr_pk`, `e_addr_1`, `is_dl`). Без ACL этот хаос проникает в нашу domain. С ACL наш `Customer` чистый, ACL делает грязную translation работу. **Cost**: extra layer, но **защита от cascade changes** когда external API меняется.

### ACL как набор seam-ов — декомпозиция для Strangler-Fig без миграции схемы

Каноничный ACL выше — **один монолитный слой** (Adapter + Translator + Facade). Это правильная отправная точка, но в реальной Strangler-Fig миграции, где **схему БД намеренно НЕ трогают** (старый монолит ещё пишет в те же таблицы, дешевле параллелить, чем останавливать прод на DDL), монолитный translator превращается в комок: один метод читает legacy-строки, разбирает JSONB, собирает агрегат, проверяет инварианты — и его нельзя протестировать без живой БД.

**Почему так получается.** Когда схема заморожена, вся «грязь» legacy (nullable-комбинации, полиморфный JSON, неявные состояния через счётчики) остаётся *на входе*. Если размазать её разбор по translator-у, каждый баг в маппинге требует поднять Postgres, налить фикстуру, прогнать integration-тест. Цикл обратной связи — минуты вместо миллисекунд.

**Что делать.** Разрезать монолитный ACL на несколько тонких **seam-ов** (швов) — каждый одна чистая функция «вход → выход», без I/O. Seam — это **единственное место**, где конкретный вид legacy-хаоса превращается в типы домена. Тогда:

- каждый seam — pure function: `row → domain` или `(row, context) → Result<Aggregate>`;
- юнит-тест без БД и без моков — на вход подаёшь POCO/`JsonElement`, на выходе сравниваешь домен;
- баг локализован: «состояние посчиталось неверно» → seam 1, «неизвестный discriminator» → seam 2, «инвариант не сработал» → seam 3.

> [!info] Seam (шов) vs Adapter
> **Adapter** держит I/O (HTTP/SQL) — его тестируют интеграционно. **Seam** — это чистая трансформация *после* I/O: данные уже в памяти. Граница проходит ровно там, где `IDataReader`/`NpgsqlDataReader` отдал `object?`-ы. Всё, что левее — Adapter (мокается/интегрируется), всё, что правее — seam (тестируется in-memory). Поэтому seam-ы и есть та часть ACL, которую можно покрыть быстрыми тестами на 100%.

Граница потока в Strangler-Fig ACL:

```text
                         ┌──────────── ACL ───────────────────────────────┐
NpgsqlDataReader ──row──▶│ Seam 1: read mapper   (nullable-combo → DU)     │
(frozen legacy            │ Seam 2: parse ctor    (JSONB N-discr → kind)    │──▶ clean
 schema, Adapter)         │ Seam 3: write factory (ctx + DU → Aggregate)    │    Domain
                         └─────────────────────────────────────────────────┘
   I/O boundary  ────────────────────────▲ pure functions, unit-testable
```

#### Seam 1 — read-side mapper: nullable-комбо → закрытый DU

Legacy-таблица `quiz_attempts` кодирует состояние **тремя** nullable/неявными столбцами: `score int NULL`, `completed_at timestamptz NULL`, `attempts int NOT NULL` (где `0` означает «ещё не начинал»). Это классический *implicit state machine*: валидные состояния — `NotStarted` / `InProgress` / `Completed`, но в схеме они выражены **комбинацией** значений, и есть невыразимые комбинации (`score` есть, а `completed_at` — нет).

Анти-паттерн — растащить `int?`/`DateTime?` по всему домену и плодить `if (score is not null && completedAt is not null)` в десятке мест:

```csharp
// BAD: nullable-комбо протекает в домен, проверка состояния дублируется
public sealed class AttemptRowBad
{
    public int? Score { get; init; }
    public DateTime? CompletedAt { get; init; }
    public int Attempts { get; init; }

    // каждый вызывающий заново выводит состояние — рассинхрон гарантирован
    public bool IsDone => Score is not null && CompletedAt is not null;
}
```

Seam 1 — **единственное место**, где nullable превращаются в тип. Закрытый sealed-record DU делает невыразимые состояния непредставимыми:

```csharp
// Closed discriminated union — три состояния, ничего сверх
public abstract record AttemptState
{
    private AttemptState() { } // запрет внешних наследников => union закрыт

    public sealed record NotStarted : AttemptState;
    public sealed record InProgress(int Attempts) : AttemptState;
    public sealed record Completed(int Score, DateTimeOffset CompletedAt, int Attempts) : AttemptState;
}
```

```csharp
// Seam 1: ЕДИНСТВЕННАЯ точка, где nullable-комбо становится типом.
// Pure function: row (POCO) -> DU. Ни I/O, ни моков для теста.
public static class AttemptStateMapper
{
    public static AttemptState Map(int? score, DateTimeOffset? completedAt, int attempts) =>
        (score, completedAt, attempts) switch
        {
            (null, null, 0)               => new AttemptState.NotStarted(),
            (null, null, > 0 and var a)   => new AttemptState.InProgress(a),
            ({ } s, { } at, var a)        => new AttemptState.Completed(s, at, a),

            // невыразимая комбинация в legacy => fail fast, не молчаливый null
            _ => throw new CorruptLegacyStateException(score, completedAt, attempts)
        };
}

public sealed class CorruptLegacyStateException(int? score, DateTimeOffset? completedAt, int attempts)
    : Exception($"Unmappable attempt row: score={score}, completedAt={completedAt}, attempts={attempts}");
```

Тест — без БД, миллисекунды:

```csharp
[Fact]
public void Map_score_without_completedAt_is_corrupt()
{
    var act = () => AttemptStateMapper.Map(score: 42, completedAt: null, attempts: 1);
    act.Should().Throw<CorruptLegacyStateException>();
}

[Fact]
public void Map_zero_attempts_is_NotStarted() =>
    AttemptStateMapper.Map(null, null, 0).Should().Be(new AttemptState.NotStarted());
```

#### Seam 2 — parse constructor: полиморфный JSONB → один `Parse(row)` с типизированным исключением

Legacy-столбец `payload jsonb` хранит N разных видов событий, и за годы накопились **синонимы дискриминатора**: `"kind"`, `"type"`, `"event_type"`, а значения дрейфовали (`"signup"` vs `"sign_up"` vs `"registered"`). Распределённый разбор (где попало `payload["type"]`) — гарантированный источник тихих `null`.

Seam 2 — один `Parse`, который **централизованно** сводит синонимы и при неизвестном виде бросает типизированное исключение (fail fast), а не возвращает `null` для дальнейшего null-propagation:

```csharp
public enum LegacyEventKind { Signup, Purchase, Refund }

public sealed class UnknownLegacyKindException(string? rawKind, string rawJson)
    : Exception($"Unknown legacy event kind '{rawKind}'. Raw: {rawJson}")
{
    public string? RawKind { get; } = rawKind;
    public string RawJson { get; } = rawJson;
}
```

```csharp
using System.Collections.Frozen;
using System.Text.Json;

// Seam 2: ЕДИНСТВЕННАЯ точка разбора legacy-вида. Синонимы агрегируются здесь.
public static class LegacyEventParser
{
    // дискриминатор мог лежать под любым из этих ключей
    private static readonly string[] KindKeys = ["kind", "type", "event_type"];

    // канонизация значений-синонимов в один enum
    private static readonly FrozenDictionary<string, LegacyEventKind> KindMap =
        new Dictionary<string, LegacyEventKind>(StringComparer.OrdinalIgnoreCase)
        {
            ["signup"]     = LegacyEventKind.Signup,
            ["sign_up"]    = LegacyEventKind.Signup,
            ["registered"] = LegacyEventKind.Signup,
            ["purchase"]   = LegacyEventKind.Purchase,
            ["order_paid"] = LegacyEventKind.Purchase,
            ["refund"]     = LegacyEventKind.Refund,
            ["chargeback"] = LegacyEventKind.Refund,
        }.ToFrozenDictionary();

    public static LegacyEventKind Parse(JsonElement row)
    {
        var raw = ExtractKind(row);

        return raw is not null && KindMap.TryGetValue(raw, out var kind)
            ? kind
            : throw new UnknownLegacyKindException(raw, row.GetRawText());
    }

    private static string? ExtractKind(JsonElement row)
    {
        foreach (var key in KindKeys)
            if (row.TryGetProperty(key, out var prop) && prop.ValueKind is JsonValueKind.String)
                return prop.GetString();

        return null;
    }
}
```

> [!warning] Не возвращать `null` из `Parse`
> Соблазн — `TryParse(row, out kind)` и тихо пропустить нераспознанную строку. В Strangler-Fig это маскирует **schema drift**: legacy-монолит выкатил новый вид события, а ты его молча потерял. Типизированный `UnknownLegacyKindException` с `RawJson` останавливает миграцию на первой же незнакомой строке и даёт точный repro. Fail fast здесь дешевле, чем расследование «почему пропали 3% событий».

#### Seam 3 — write-side aggregate factory: инварианты на ПРЕ-загруженном контексте

При записи в новый домен агрегат должен проверить инвариант, требующий внешние данные — например, бронь не должна пересекаться с уже занятыми `TimeSlot`-ами. Анти-паттерн — дать фабрике лезть в БД самой: домен становится зависим от persistence, фабрику нельзя протестировать без репозитория.

```csharp
// BAD: фабрика домена тянет данные сама => coupling домена с инфраструктурой
public static class BookingFactoryBad
{
    public static Booking Create(Guid roomId, DateTimeOffset start, ISlotRepository repo) // <-- repo в домене
    {
        var taken = repo.GetSlots(roomId); // I/O внутри домена, тест требует мок
        // ...
        return new Booking();
    }
}
```

Seam 3 принимает **уже загруженный** контекст параметром (`IReadOnlyList<TimeSlot>`). Загрузку делает Application-слой *до* вызова; домен остаётся чистым и проверяет инвариант над переданными данными:

```csharp
public sealed record TimeSlot(DateTimeOffset Start, DateTimeOffset End);

public sealed class Booking
{
    public Guid Id { get; }
    public Guid RoomId { get; }
    public TimeSlot Slot { get; }

    private Booking(Guid id, Guid roomId, TimeSlot slot) =>
        (Id, RoomId, Slot) = (id, roomId, slot);

    // Seam 3: инвариант на ПРЕ-загруженном контексте. Ни репозитория, ни DbContext.
    public static Result<Booking> Create(
        Guid roomId,
        TimeSlot desired,
        IReadOnlyList<TimeSlot> existingSlots) // pre-fetched контекст как параметр
    {
        foreach (var slot in existingSlots)
            if (Overlaps(desired, slot))
                return Result<Booking>.Fail(
                    Error.Conflict("Booking.Overlap", "Time slot is already taken"));

        return Result<Booking>.Ok(new Booking(Guid.NewGuid(), roomId, desired));
    }

    private static bool Overlaps(TimeSlot a, TimeSlot b) =>
        a.Start < b.End && b.Start < a.End;
}
```

Application-слой связывает I/O и чистую фабрику:

```csharp
public sealed class CreateBookingHandler(ISlotRepository slots, IUnitOfWork unitOfWork)
{
    public async Task<Result<Guid>> HandleAsync(CreateBookingCommand cmd, CancellationToken ct)
    {
        // I/O — в Application, ДО фабрики
        var existing = await slots.GetForRoomAsync(cmd.RoomId, ct);

        var slot = new TimeSlot(cmd.Start, cmd.End);
        var result = Booking.Create(cmd.RoomId, slot, existing); // seam 3, без I/O
        if (result.IsFailure)
            return Result<Guid>.Fail(result.Error!);

        slots.Add(result.Value!);
        await unitOfWork.SaveChangesAsync(ct);
        return Result<Guid>.Ok(result.Value!.Id);
    }
}
```

Тест фабрики — чистый, контекст подаётся списком:

```csharp
[Fact]
public void Create_overlapping_slot_fails()
{
    var existing = new[] { new TimeSlot(At(9), At(10)) };
    var result = Booking.Create(Guid.NewGuid(), new TimeSlot(At(9, 30), At(10, 30)), existing);

    result.IsFailure.Should().BeTrue();
    result.Error!.Code.Should().Be("Booking.Overlap");

    static DateTimeOffset At(int h, int m = 0) => new(2026, 1, 1, h, m, 0, TimeSpan.Zero);
}
```

#### Snapshot-тесты границ разбора — ловушка для schema drift

Юнит-тесты seam-ов проверяют *известные* формы. Чтобы поймать момент, когда legacy-монолит **молча** изменил данные (новый дискриминатор, новая nullable-комбинация), на каждую границу разбора ставят **snapshot-тест против реальных прод-строк**: выгружаешь репрезентативную выборку (анонимизированную) из прод-реплики, прогоняешь через seam, фиксируешь результат как snapshot. Дрейф схемы → snapshot не совпал/`Parse` бросил → красный CI до того, как миграция дойдёт до прода.

```csharp
// Verify (snapshot): прод-строки как фикстуры. Дрейф схемы => упавший snapshot.
[Theory]
[MemberData(nameof(ProductionPayloads))] // загружены из anonymized prod dump
public Task Parse_matches_snapshot(string rawJson)
{
    using var doc = JsonDocument.Parse(rawJson);
    var kind = LegacyEventParser.Parse(doc.RootElement);
    return Verify(kind).UseParameters(rawJson);
}
```

> [!tip] Откуда брать прод-строки
> Анонимизированный дамп из read-реплики (никакого PII в фикстурах) или Testcontainers-снимок. Обновлять выборку при каждом релизе legacy-монолита — именно она первой покажет, что upstream поменял контракт. Это дешёвый «канарейка»-тест на разрыв ACL-контракта между двумя живыми системами.

> [!question]- **Интервью: как мигрировать модуль Strangler-Fig'ом, не меняя схему БД?**
> Поднимаю новый чистый домен рядом со старым, а на стыке ставлю **ACL из тонких seam-ов** (не один translator): seam 1 — read-mapper, который в одном месте сворачивает nullable-комбо столбцов в **закрытый DU** состояний (невыразимые комбинации → исключение, не `null`); seam 2 — `Parse(row)`, централизованно сводящий синонимы полиморфного JSONB-дискриминатора в enum и бросающий **типизированный** `UnknownLegacyKindException` при незнакомом виде (fail fast против тихого null-propagation); seam 3 — фабрика агрегата, проверяющая инварианты над **пре-загруженным** контекстом (`IReadOnlyList<TimeSlot>` параметром), а не лезущая в БД сама (домен не зависит от persistence). Каждый seam — pure function, юнит-тест без БД и моков. Поверх — **snapshot-тесты против прод-строк** на каждой границе разбора: они краснеют, когда legacy молча дрейфует схему. Схему не трогаю намеренно: старый монолит ещё пишет в те же таблицы.

### Domain Services — где живёт логика не fitting в Entity/VO

Иногда бизнес-операция касается **нескольких** Aggregates или не принадлежит одной Entity. Тогда — **Domain Service** (НЕ Application Service!).

```csharp
// ❌ Логика "перевести деньги между accounts" не fits в один Account
public class Account
{
    public void Transfer(Account other, Money amount) { /* ... */ }
    // Account.Transfer трогает другой Account — нарушение Aggregate boundary
}

// ✅ Domain Service — оперирует двумя Aggregates
public sealed class MoneyTransferService
{
    public Result Transfer(Account from, Account to, Money amount)
    {
        if (from.Balance < amount)
            return Result.Fail(Errors.InsufficientFunds());

        if (from.Currency != to.Currency)
            return Result.Fail(Errors.CurrencyMismatch());

        from.Withdraw(amount);   // each Aggregate знает свою операцию
        to.Deposit(amount);

        return Result.Ok();
    }
}
```

**Domain Service vs Application Service:**

| | Domain Service | Application Service |
|--|----------------|---------------------|
| Слой | Domain | Application (use cases) |
| Знает про | Domain entities, VOs | Repositories, UoW, external services |
| Зависимости | Только domain | Infrastructure interfaces |
| Пример | `MoneyTransferService` | `TransferMoneyHandler` (orchestrates: load from DB, call domain service, save) |
| Тесты | Unit, в memory | Integration, often mocked |

```csharp
// Application Service (Handler) использует Domain Service
public sealed class TransferMoneyHandler(
    IAccountRepository accounts,
    MoneyTransferService transferService,   // Domain Service
    IUnitOfWork unitOfWork)
{
    public async Task<Result> HandleAsync(TransferMoneyCommand cmd, CancellationToken ct)
    {
        var from = await accounts.GetByIdAsync(cmd.FromId, ct);
        var to = await accounts.GetByIdAsync(cmd.ToId, ct);

        if (from is null || to is null)
            return Result.Fail(Errors.NotFound("Account"));

        var result = transferService.Transfer(from, to, cmd.Amount);
        if (result.IsFailure) return result;

        await unitOfWork.SaveChangesAsync(ct);
        return Result.Ok();
    }
}
```

> [!question]- **Интервью: Domain Service vs Application Service?**
> **Domain Service** — содержит **бизнес-логику** которая касается **нескольких Aggregates** или не принадлежит одной Entity (перевод денег, расчёт скидки с участием Order и Customer и Promotion). Лежит в **Domain layer**, **не знает про infrastructure** (no DbContext, no HTTP). **Application Service** (Handler / Use Case) — **orchestration**: load from repository, call domain service, save through UoW, dispatch events. Лежит в **Application layer**. **Тест**: domain service — unit (in-memory), application service — integration (с моками или реальной DB).

### Specification Pattern — encapsulation сложных queries

Бизнес-правило для фильтрации часто переиспользуется в нескольких местах: query в БД, validation на add, проверка в domain logic.

```csharp
// ❌ Дублирование condition
public IEnumerable<Customer> GetPremiumCustomers() =>
    _db.Customers.Where(c => c.OrderCount >= 10 && c.TotalSpent >= 1000m && c.IsActive);

public bool IsPremium(Customer c) =>
    c.OrderCount >= 10 && c.TotalSpent >= 1000m && c.IsActive;

// Если изменится business rule (premium >= 5 orders) — забудешь поправить в одном месте
```

```csharp
// ✅ Specification encapsulates rule once
public sealed class PremiumCustomerSpec : Specification<Customer>
{
    public override Expression<Func<Customer, bool>> ToExpression() =>
        c => c.OrderCount >= 10 && c.TotalSpent >= 1000m && c.IsActive;
}

// Использование везде:
var premiums = _db.Customers.Where(new PremiumCustomerSpec().ToExpression());

if (new PremiumCustomerSpec().IsSatisfiedBy(customer))
    customer.GrantPremiumDiscount();

// Composable
var premiumNonEU = new PremiumCustomerSpec().And(new NonEUCustomerSpec());
```

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity) =>
        ToExpression().Compile().Invoke(entity);

    public Specification<T> And(Specification<T> other) =>
        new AndSpecification<T>(this, other);

    public Specification<T> Or(Specification<T> other) =>
        new OrSpecification<T>(this, other);
}

public sealed class AndSpecification<T>(Specification<T> left, Specification<T> right)
    : Specification<T>
{
    public override Expression<Func<T, bool>> ToExpression()
    {
        var l = left.ToExpression();
        var r = right.ToExpression();
        var p = Expression.Parameter(typeof(T));
        var body = Expression.AndAlso(
            Expression.Invoke(l, p),
            Expression.Invoke(r, p));
        return Expression.Lambda<Func<T, bool>>(body, p);
    }
}
```

**Когда использовать Specification:**
- ✅ Business rule переиспользуется в 3+ местах
- ✅ Сложная композиция (premium AND active AND non-EU AND NOT churned)
- ✅ Хочется тестировать rule independently
- ❌ Простой Where в одном месте — over-engineering
- ❌ Если EF Core не может translate Compile() expression — fallback на raw query

**Альтернатива в .NET ecosystem:** `Ardalis.Specification` package (готовая библиотека).

### Strategic vs Tactical DDD — что когда

```
Strategic DDD     ─→ применяешь на старте проекта
                       └ домен анализ, BC discovery
                       └ context maps
                       └ Anti-Corruption Layer для legacy
                  ─→ возвращаешься регулярно
                       └ при росте системы
                       └ при reorg команд
                       └ при новых интеграциях

Tactical DDD      ─→ применяешь когда BC уже определён
                       └ Entity, VO, Aggregate Root
                       └ Domain Events
                       └ Repository, Specification
                       └ Result pattern
```

**Senior должен знать оба.** На interview спрашивают:
- "Что такое DDD?" → должен упомянуть **обе** части
- "Как разделить большое приложение?" → strategic answer (BC, ACL)
- "Как защитить domain от protocol-level конкретики?" → tactical (private setters, VO, Result)

### Event Storming — как обнаруживать Bounded Contexts

Workshop technique для discovery boundaries в существующем или новом domain.

```
Workshop (1 день, все стейкхолдеры):
1. На стене стикерами marks: "Domain Events" (что-то произошло)
   "Order placed", "Payment processed", "Item shipped"...

2. Группируем events по timeline (chronological)

3. Identify Commands (что users делают для events)
   "Place order" → "Order placed"

4. Identify Aggregates (что хранит state когда event происходит)
   Order Aggregate, Payment Aggregate

5. Group related aggregates → BC candidates

6. Draw context map с relationships
```

**Result:** общая картина domain + BC boundaries + integration points. Очень эффективно для new projects или legacy modernization.

> [!question]- **Интервью: как ты определяешь Bounded Contexts на новом проекте?**
> **Event Storming workshop** с бизнес-стейкхолдерами и dev team. Один день: 1) Marks all domain events на стене ("Order placed", "Payment received"). 2) Group по timeline и actors. 3) Identify aggregates (что хранит state). 4) Cluster related aggregates → BC candidates. 5) Define context map с relationships (Partnership / Customer-Supplier / ACL). **Альтернатива** для smaller scope: domain experts interviews + analysis где terminology меняется ("Customer" в Sales vs "Patient" в Healthcare BC одной системы). **Worst practice**: разбивать по техническим axes (DB, microservice, layer) — приведёт к coupling и distributed monolith.

---

## Строительные блоки

| Блок | Что это | Пример |
|------|---------|--------|
| **Entity** | Объект с уникальной идентичностью | `Order`, `User` (сравниваются по Id) |
| **Value Object** | Объект без идентичности, сравнивается по значению | `Email`, `Money`, `Address` |
| **Aggregate Root** | Главная Entity, точка входа для изменений | `Order` (содержит `OrderItem`-ы) |
| **Domain Event** | Уведомление «что-то произошло в домене» | `OrderCreatedEvent`, `PaymentCompletedEvent` |
| **Repository** | Абстракция для сохранения/загрузки Aggregate | `IOrderRepository` |

---

## Entity\<TId\> — базовый класс

Entity сравнивается по идентичности (Id), не по значению полей.

```csharp
// Базовый класс для всех Entity
public abstract class Entity<TId> : IEquatable<Entity<TId>>
    where TId : notnull
{
    public TId Id { get; protected init; }

    protected Entity(TId id) => Id = id;

    // Для EF Core (параметрless конструктор)
    protected Entity() => Id = default!;

    public bool Equals(Entity<TId>? other)
        => other is not null && Id.Equals(other.Id);

    public override bool Equals(object? obj)
        => obj is Entity<TId> entity && Equals(entity);

    public override int GetHashCode() => Id.GetHashCode();

    public static bool operator ==(Entity<TId>? left, Entity<TId>? right)
        => Equals(left, right);

    public static bool operator !=(Entity<TId>? left, Entity<TId>? right)
        => !Equals(left, right);
}
```

**Нюанс:** Два `Order` с одинаковым `Id` — это один и тот же заказ, даже если поля отличаются (данные могли измениться). Два `Money(100, "USD")` — равны по значению, не по ссылке.

---

## Value Object — sealed record + Result

Value Object не имеет идентичности. Сравнивается по значению. Immutable. Валидация при создании.

```csharp
// Email — Value Object
public sealed record Email
{
    public string Value { get; init; }

    private Email(string value) => Value = value;

    public static Result<Email> Create(string? input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return Result<Email>.Fail(Error.Validation("Email.Empty", "Email is required"));

        var trimmed = input.Trim().ToLowerInvariant();

        if (!trimmed.Contains('@') || trimmed.Length > 256)
            return Result<Email>.Fail(Error.Validation("Email.Invalid", "Invalid email format"));

        // Санитизация пользовательского ввода
        var sanitized = WebUtility.HtmlEncode(trimmed);

        return Result<Email>.Ok(new Email(sanitized));
    }

    public override string ToString() => Value;
}

// Money — Value Object с арифметикой
public sealed record Money
{
    public decimal Amount { get; init; }
    public string Currency { get; init; }

    private static readonly FrozenSet<string> AllowedCurrencies =
        new[] { "USD", "EUR", "RUB", "GBP" }.ToFrozenSet();

    private Money(decimal amount, string currency)
        => (Amount, Currency) = (amount, currency);

    public static Result<Money> Create(decimal amount, string currency)
    {
        if (amount < 0)
            return Result<Money>.Fail(Error.Validation("Money.Negative", "Amount cannot be negative"));

        if (!AllowedCurrencies.Contains(currency))
            return Result<Money>.Fail(Error.Validation("Money.Currency", $"Unsupported currency: {currency}"));

        return Result<Money>.Ok(new Money(amount, currency));
    }

    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException($"Cannot add {Currency} and {other.Currency}");
        return new Money(Amount + other.Amount, Currency);
    }

    public override string ToString() => $"{Amount:F2} {Currency}";
}

// Использование
var emailResult = Email.Create(userInput);
// emailResult.IsSuccess → true/false, без исключений
```

### Почему именно так?

| Решение | Почему |
|---------|--------|
| `sealed record` | Immutable + value equality из коробки + нельзя наследовать |
| Приватный конструктор | Нельзя создать невалидный объект в обход `Create()` |
| `Create()` → `Result<T>` | Ошибки как значения, не исключения |
| `FrozenSet` для валидации enum | O(1) lookup, неизменяемый, оптимален для hot path |
| `WebUtility.HtmlEncode` | Защита от XSS при пользовательском вводе |
| `init` свойства | Immutable после создания, но EF Core может маппить |

---

## Aggregate Root — точка входа

Aggregate Root — Entity, через которую проходят ВСЕ изменения. Дочерние объекты (OrderItem) недоступны напрямую — только через Root.

```csharp
// Базовый класс Aggregate Root с Domain Events
public abstract class AggregateRoot<TId> : Entity<TId>
    where TId : notnull
{
    private readonly List<IDomainEvent> _domainEvents = [];

    public IReadOnlyCollection<IDomainEvent> DomainEvents
        => _domainEvents.AsReadOnly();

    protected AggregateRoot(TId id) : base(id) { }
    protected AggregateRoot() { } // EF Core

    protected void Raise(IDomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);

    public void ClearDomainEvents()
        => _domainEvents.Clear();
}

// Order — Aggregate Root
public sealed class Order : AggregateRoot<Guid>
{
    private readonly List<OrderItem> _items = [];

    public Guid CustomerId { get; private init; }
    public OrderStatus Status { get; private set; }
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    public Money Total => CalculateTotal();

    private Order() { } // EF Core

    // Фабричный метод — единственный способ создать Order
    public static Result<Order> Create(Guid customerId)
    {
        if (customerId == Guid.Empty)
            return Result<Order>.Fail(Error.Validation("Order.CustomerId", "Customer ID is required"));

        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Draft
        };

        order.Raise(new OrderCreatedEvent(order.Id, customerId));

        return Result<Order>.Ok(order);
    }

    // Бизнес-метод — возвращает Result, не бросает exception
    public Result AddItem(Guid productId, int quantity, Money price)
    {
        if (Status != OrderStatus.Draft)
            return Result.Fail(Error.Validation("Order.NotDraft", "Can only add items to draft orders"));

        if (quantity <= 0)
            return Result.Fail(Error.Validation("Order.Quantity", "Quantity must be positive"));

        var existingItem = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existingItem is not null)
        {
            existingItem.IncreaseQuantity(quantity);
            return Result.Ok();
        }

        _items.Add(OrderItem.Create(productId, quantity, price));
        return Result.Ok();
    }

    public Result Cancel()
    {
        if (Status == OrderStatus.Shipped)
            return Result.Fail(Error.Validation("Order.AlreadyShipped", "Cannot cancel shipped order"));

        if (Status == OrderStatus.Cancelled)
            return Result.Fail(Error.Validation("Order.AlreadyCancelled", "Order is already cancelled"));

        Status = OrderStatus.Cancelled;
        Raise(new OrderCancelledEvent(Id));
        return Result.Ok();
    }

    public Result Confirm()
    {
        if (Status != OrderStatus.Draft)
            return Result.Fail(Error.Validation("Order.NotDraft", "Can only confirm draft orders"));

        if (_items.Count == 0)
            return Result.Fail(Error.Validation("Order.NoItems", "Order must have at least one item"));

        Status = OrderStatus.Confirmed;
        Raise(new OrderConfirmedEvent(Id, Total));
        return Result.Ok();
    }

    private Money CalculateTotal()
        => _items.Aggregate(
            Money.Create(0, "USD").Value!,
            (sum, item) => sum.Add(item.Subtotal));
}

// OrderItem — Entity внутри Aggregate (не Root!)
public sealed class OrderItem : Entity<Guid>
{
    public Guid ProductId { get; private init; }
    public int Quantity { get; private set; }
    public Money Price { get; private init; } = null!;
    public Money Subtotal => Money.Create(Price.Amount * Quantity, Price.Currency).Value!;

    private OrderItem() { } // EF Core

    internal static OrderItem Create(Guid productId, int quantity, Money price)
        => new()
        {
            Id = Guid.NewGuid(),
            ProductId = productId,
            Quantity = quantity,
            Price = price
        };

    internal void IncreaseQuantity(int amount) => Quantity += amount;
}

public enum OrderStatus { Draft, Confirmed, Shipped, Cancelled }
```

### Правила Aggregate

| Правило | Почему |
|---------|--------|
| Один `SaveChanges` = один Aggregate | Транзакционная граница = Aggregate |
| Дочерние Entity недоступны через DbSet | Только через Root: `order.AddItem(...)`, не `context.OrderItems.Add(...)` |
| Между Aggregates — только по Id | `order.CustomerId` (Guid), не `order.Customer` (navigation) |
| Бизнес-методы возвращают `Result` | Без исключений в бизнес-логике |
| Фабричный `Create()` + приватный конструктор | Невозможно создать невалидный объект |

---

## Domain Events

Уведомление о том, что произошло в домене. Обрабатываются после SaveChanges.

```csharp
// Интерфейс
public interface IDomainEvent
{
    Guid Id { get; }
    DateTime OccurredAt { get; }
}

// Базовая реализация
public abstract record DomainEvent : IDomainEvent
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
}

// Конкретные события
public sealed record OrderCreatedEvent(Guid OrderId, Guid CustomerId) : DomainEvent;
public sealed record OrderConfirmedEvent(Guid OrderId, Money Total) : DomainEvent;
public sealed record OrderCancelledEvent(Guid OrderId) : DomainEvent;

// Обработчик события
public sealed class OrderCreatedEventHandler(
    IEmailService emailService,
    ILogger<OrderCreatedEventHandler> logger)
{
    public async Task HandleAsync(OrderCreatedEvent @event, CancellationToken ct)
    {
        logger.LogInformation("Order {OrderId} created for customer {CustomerId}",
            @event.OrderId, @event.CustomerId);

        await emailService.SendOrderConfirmationAsync(@event.OrderId, ct);
    }
}
```

### Диспетчеризация через SaveChangesInterceptor

```csharp
public sealed class DomainEventInterceptor(IServiceProvider serviceProvider)
    : SaveChangesInterceptor
{
    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData, int result, CancellationToken ct)
    {
        var context = eventData.Context!;

        var aggregates = context.ChangeTracker.Entries<AggregateRoot<Guid>>()
            .Select(e => e.Entity)
            .Where(a => a.DomainEvents.Count > 0)
            .ToList();

        var events = aggregates.SelectMany(a => a.DomainEvents).ToList();
        aggregates.ForEach(a => a.ClearDomainEvents());

        using var scope = serviceProvider.CreateScope();
        foreach (var domainEvent in events)
        {
            // Резолвим обработчик по типу события
            var handlerType = typeof(IDomainEventHandler<>)
                .MakeGenericType(domainEvent.GetType());

            var handlers = scope.ServiceProvider.GetServices(handlerType);
            foreach (dynamic handler in handlers)
            {
                await handler.HandleAsync((dynamic)domainEvent, ct);
            }
        }

        return result;
    }
}

// Интерфейс обработчика
public interface IDomainEventHandler<in TEvent> where TEvent : IDomainEvent
{
    Task HandleAsync(TEvent @event, CancellationToken ct);
}
```

**Нюанс:** Events диспатчатся ПОСЛЕ `SaveChanges` (в `SavedChangesAsync`), не до. Это гарантирует, что данные уже сохранены. Если handler упадёт — данные не откатятся (eventual consistency).

---

## Result Pattern — полная реализация

```csharp
// Error — описание ошибки
public sealed record Error(string Code, string Message, ErrorType Type)
{
    public static Error Validation(string code, string message)
        => new(code, message, ErrorType.Validation);

    public static Error NotFound(string code, string message)
        => new(code, message, ErrorType.NotFound);

    public static Error Unauthorized(string code, string message)
        => new(code, message, ErrorType.Unauthorized);

    public static Error Conflict(string code, string message)
        => new(code, message, ErrorType.Conflict);

    public static Error Internal(string code, string message)
        => new(code, message, ErrorType.Internal);
}

public enum ErrorType { Validation, NotFound, Unauthorized, Conflict, Internal }

// Result без значения (для void-операций)
public sealed class Result
{
    public bool IsSuccess { get; }
    public Error? Error { get; }
    public bool IsFailure => !IsSuccess;

    private Result(bool isSuccess, Error? error)
        => (IsSuccess, Error) = (isSuccess, error);

    public static Result Ok() => new(true, null);
    public static Result Fail(Error error) => new(false, error);
}

// Result<T> с значением
public sealed class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public Error? Error { get; }
    public bool IsFailure => !IsSuccess;

    private Result(bool isSuccess, T? value, Error? error)
        => (IsSuccess, Value, Error) = (isSuccess, value, error);

    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(Error error) => new(false, default, error);

    // Функциональные методы
    public Result<TOut> Map<TOut>(Func<T, TOut> mapper)
        => IsSuccess ? Result<TOut>.Ok(mapper(Value!)) : Result<TOut>.Fail(Error!);

    public async Task<Result<TOut>> MapAsync<TOut>(Func<T, Task<TOut>> mapper)
        => IsSuccess ? Result<TOut>.Ok(await mapper(Value!)) : Result<TOut>.Fail(Error!);

    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<Error, TOut> onFailure)
        => IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

// Маппинг Result → HTTP response в Minimal API
public static class ResultExtensions
{
    public static IResult ToResponse<T>(this Result<T> result, Func<T, IResult> onSuccess)
        => result.Match(
            onSuccess,
            error => error.Type switch
            {
                ErrorType.Validation => TypedResults.Problem(
                    title: "Validation Error",
                    detail: error.Message,
                    statusCode: StatusCodes.Status400BadRequest),
                ErrorType.NotFound => TypedResults.NotFound(
                    new ProblemDetails { Detail = error.Message }),
                ErrorType.Unauthorized => TypedResults.Problem(
                    statusCode: StatusCodes.Status403Forbidden),
                ErrorType.Conflict => TypedResults.Conflict(
                    new ProblemDetails { Detail = error.Message }),
                _ => TypedResults.Problem(
                    title: "Internal Error",
                    statusCode: StatusCodes.Status500InternalServerError)
            });
}

// Пример в endpoint
app.MapPost("/orders", async (
    CreateOrderRequest request,
    CreateOrderHandler handler,
    CancellationToken ct) =>
{
    var result = await handler.HandleAsync(request, ct);
    return result.ToResponse(order => TypedResults.Created($"/orders/{order.Id}", order));
});
```

---

## EF Core конфигурация для DDD

```csharp
// Order Configuration — маппинг Aggregate Root
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(20);

        // Value Object → отдельные столбцы
        builder.OwnsMany(o => o.Items, item =>
        {
            item.WithOwner().HasForeignKey("OrderId");
            item.HasKey(i => i.Id);

            // Nested Value Object
            item.OwnsOne(i => i.Price, price =>
            {
                price.Property(p => p.Amount).HasPrecision(18, 2);
                price.Property(p => p.Currency).HasMaxLength(3);
            });
        });

        // Navigation через backing field
        builder.Navigation(o => o.Items)
            .UsePropertyAccessMode(PropertyAccessMode.Field);

        // Domain Events НЕ маппятся в БД
        builder.Ignore(o => o.DomainEvents);
    }
}
```

---

## Полный flow: Endpoint → Handler → Domain → Persistence

```csharp
// Handler (Application Layer) — без MediatR
public sealed class CreateOrderHandler(
    IOrderRepository orderRepository,
    IUnitOfWork unitOfWork)
{
    public async Task<Result<OrderDto>> HandleAsync(
        CreateOrderRequest request, CancellationToken ct)
    {
        // 1. Создаём Value Object (валидация внутри)
        var emailResult = Email.Create(request.CustomerEmail);
        if (emailResult.IsFailure)
            return Result<OrderDto>.Fail(emailResult.Error!);

        // 2. Создаём Aggregate (валидация внутри)
        var orderResult = Order.Create(request.CustomerId);
        if (orderResult.IsFailure)
            return Result<OrderDto>.Fail(orderResult.Error!);

        var order = orderResult.Value!;

        // 3. Бизнес-операции
        foreach (var item in request.Items)
        {
            var priceResult = Money.Create(item.Price, item.Currency);
            if (priceResult.IsFailure)
                return Result<OrderDto>.Fail(priceResult.Error!);

            var addResult = order.AddItem(item.ProductId, item.Quantity, priceResult.Value!);
            if (addResult.IsFailure)
                return Result<OrderDto>.Fail(addResult.Error!);
        }

        // 4. Persist
        orderRepository.Add(order);
        await unitOfWork.SaveChangesAsync(ct);
        // Domain Events диспатчатся автоматически через Interceptor

        // 5. Map to DTO
        return Result<OrderDto>.Ok(new OrderDto(order.Id, order.Status.ToString(), order.Total.ToString()));
    }
}

public sealed record CreateOrderRequest(
    Guid CustomerId,
    string CustomerEmail,
    IReadOnlyList<OrderItemRequest> Items);

public sealed record OrderItemRequest(
    Guid ProductId,
    int Quantity,
    decimal Price,
    string Currency);

public sealed record OrderDto(Guid Id, string Status, string Total);
```

---

## Анти-паттерны

```csharp
// ✗ Anemic Model — логика в сервисе, Entity — мешок данных
public class Order
{
    public Guid Id { get; set; }            // set — любой может менять
    public OrderStatus Status { get; set; } // нет защиты
}
public class OrderService
{
    public void Cancel(Order order)
    {
        if (order.Status == OrderStatus.Shipped) throw new Exception("...");
        order.Status = OrderStatus.Cancelled; // бизнес-логика СНАРУЖИ
    }
}

// ✓ Rich Domain Model — логика ВНУТРИ Entity
public sealed class Order : AggregateRoot<Guid>
{
    public OrderStatus Status { get; private set; } // private set — защита

    public Result Cancel() // бизнес-логика ВНУТРИ
    {
        if (Status == OrderStatus.Shipped)
            return Result.Fail(Error.Validation("Order.Shipped", "Cannot cancel shipped order"));

        Status = OrderStatus.Cancelled;
        Raise(new OrderCancelledEvent(Id));
        return Result.Ok();
    }
}
```

---

## См. также

- [[architecture-patterns|Architecture Patterns]] — Clean Architecture, VSA
- [[cqrs-mediatr|Result/CQRS]] — Result Pattern детально
- [[ef-patterns|EF Core Patterns]] — Aggregate Root в EF, Interceptors
