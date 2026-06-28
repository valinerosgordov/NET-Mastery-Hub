---
tags: [architecture, scenarios, real-world, decision-making, e-commerce, performance, senior]
level: Middle to Senior
date: 2026-04-30
---

# Real-World Scenarios — какая архитектура для какой задачи

> **Сценарий → решение**. От меню навигации до микросервисного e-commerce. От чего зависит выбор архитектуры, как она влияет на нагрузку, plus/minus каждого подхода. Companion к [[patterns-decision-guide|Patterns Decision Guide]] — там framework, тут конкретные case studies.

---

## Что это, зачем и когда

### Главное правило

> **Архитектура — это не "что лучше", это "что лучше для МОЕЙ задачи СЕЙЧАС"**.

Один и тот же паттерн может быть genius для одной задачи и over-engineering для другой. Этот файл — карта "если задача X, то начни с Y".

### Структура

1. **От чего зависит выбор** — 10 факторов
2. **Архитектура vs нагрузка** — performance impact
3. **UI / компоненты** — мелкие задачи (меню, формы, wizard)
4. **Модули** — средние задачи (корзина, поиск, reporting)
5. **Системы целиком** — большие проекты (e-commerce, портал, SaaS)
6. **Эволюция** — как меняется по мере роста
7. **Cheat sheet**

---

## 1. От чего зависит выбор архитектуры

### 10 факторов

| # | Фактор | Малое значение | Большое значение |
|---|--------|----------------|------------------|
| 1 | **Размер команды** | 1-2 dev → simple monolith | 50+ → microservices / modular |
| 2 | **Сложность домена** | CRUD → N-Layer | DDD-worthy → Clean + DDD |
| 3 | **Time to market** | MVP за неделю → minimal API | Long-term → правильная архитектура |
| 4 | **Ожидаемая нагрузка** | < 100 RPS → monolith ОК | > 10K RPS → split + cache |
| 5 | **Стабильность requirements** | Часто меняется → VSA | Стабильно → Clean Architecture |
| 6 | **Бюджет / cost** | Малый → monolith (1 server) | Большой → cloud-native |
| 7 | **SLA / availability** | 99% — restart OK | 99.99% — HA, redundancy |
| 8 | **Compliance** | Internal tool — minimal | GDPR/PCI/HIPAA → audit, encryption |
| 9 | **Опыт команды** | Junior majority → проще | Senior → можно сложнее |
| 10 | **Ecosystem** | .NET only | Polyglot → microservices justified |

### Decision matrix — quick filter

```
Хотите быстро запустить MVP?
  → Single project, ASP.NET Core minimal API, EF Core, SQLite
  → No architecture для прототипа. YAGNI.

Стабильный internal tool, < 5 dev?
  → N-Layer (3 проекта: API + Domain + Infrastructure)
  → EF Core + Built-in DI

Customer-facing SaaS, 5-15 dev, домен сложный?
  → Clean Architecture + Feature folders
  → MediatR + FluentValidation + Result pattern

Multi-team large product?
  → Modular Monolith с DDD
  → Modules через MediatR + outbox для cross-module events

Independent teams, scale, polyglot stack?
  → Microservices (carefully)
  → Не раньше чем когда реально болит monolith
```

---

## 2. Архитектура → влияние на нагрузку

### Сравнительная таблица performance

| Архитектура | Throughput (RPS) | Latency p50 | Latency p99 | Operational cost | Когда |
|-------------|-------------------|-------------|-------------|------------------|-------|
| **Single monolith** | 1K–50K | 5–20 ms | 50–100 ms | $ | Стартап, internal tool |
| **Modular monolith** | 1K–50K | 5–20 ms | 50–100 ms | $ | SaaS, B2B |
| **Microservices** | 100–100K per service | 10–50 ms (+network) | 100–500 ms | $$$$ | Крупный продукт |
| **Serverless (Azure Functions / Lambda)** | 10–10K | холодный старт 1–3s | 5–10s холодный | pay-per-use | Sporadic load |
| **HFT / hot-path** | 1M+ | < 1 ms | < 5 ms | $$$ | Trading, ad-tech |

### Influence patterns на performance

| Pattern | Performance impact | Когда оправдан |
|---------|--------------------|----------------|
| **Repository поверх EF** | 0% (просто wrapper) | Только если несколько data sources |
| **MediatR pipeline** | +5–10% overhead на reflection | Decoupling > performance |
| **CQRS с одной БД** | 0% sync, +10% сложность | Чёткая разница read/write |
| **CQRS с разными БД** | Read scale × 10 | Read-heavy app |
| **Event Sourcing** | Write 2–5x slower, read fast (projections) | Audit, replay, time-travel |
| **Dapper vs EF Core** | +30–50% read speed | Hot-path queries |
| **gRPC vs REST** | +5–10x speed, +60% smaller payload | Internal services |
| **Caching (Redis)** | +100–1000x для cached reads | Read-heavy, expensive queries |
| **Async/await** | +10x throughput на I/O | Web apps (must) |
| **Memory pooling** | -50% GC pauses | Hot paths, high-throughput |

### Главное правило performance

> **Cache > Read replica > Index > Faster query > Faster ORM > Microservices.**
>
> Cache даёт 100x boost. Microservices не делают код faster — добавляют network overhead. Они помогают **scale**, не **speed**.

См. [[performance|Performance]] и [[queries-performance|EF Queries Performance]].

---

## 3. Сценарии: UI / компоненты (мелкие задачи)

### Сценарий 1: Меню навигации с правами доступа

**Задача:** Sidebar с пунктами меню. Разные пользователи видят разные пункты (Admin видит "Пользователи", обычный — нет).

**Простой подход (90% кейсов):**

```csharp
// Декларативно через атрибуты + DI
public class MenuService
{
    private readonly ICurrentUser _user;

    public IEnumerable<MenuItem> GetMenu()
    {
        if (_user.IsInRole("Admin"))
            yield return new("Users", "/admin/users", "users-icon");
        if (_user.IsInRole("Admin") || _user.IsInRole("Manager"))
            yield return new("Reports", "/reports", "report-icon");
        yield return new("Profile", "/profile", "profile-icon");
    }
}
```

**Усложнённый — Composite + Specification:**

```csharp
// Если меню большое + правил много → struct'ра дерева + спецификации доступа
public abstract record MenuNode(string Label, ISpecification<UserContext> AccessRule);

public record MenuLeaf(string Label, string Url, string Icon, ISpecification<UserContext> Rule)
    : MenuNode(Label, Rule);

public record MenuFolder(string Label, ISpecification<UserContext> Rule, IEnumerable<MenuNode> Children)
    : MenuNode(Label, Rule);

// Применение правил рекурсивно (Composite)
public IEnumerable<MenuNode> Filter(IEnumerable<MenuNode> nodes, UserContext user) =>
    nodes
        .Where(n => n.AccessRule.IsSatisfiedBy(user))
        .Select(n => n is MenuFolder f
            ? f with { Children = Filter(f.Children, user) }
            : n);
```

**Plus/minus:**

| Подход | + | − |
|--------|---|---|
| If/else в service | Простой, читаемый | Не масштабируется при сотнях rules |
| Composite + Spec | Composable, тестируемый | Overkill для < 20 пунктов |

**Когда что:** < 20 пунктов меню → if/else. Сотни → Composite + Specifications.

### Сценарий 2: Форма с валидацией

**Задача:** Регистрация пользователя — много полей, разные правила.

**Решение: FluentValidation + MediatR pipeline**

```csharp
public record RegisterUserCommand(string Email, string Password, int Age) : IRequest<Result>;

public class RegisterUserValidator : AbstractValidator<RegisterUserCommand>
{
    public RegisterUserValidator()
    {
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleFor(x => x.Password).MinimumLength(8).Matches(@"\d");
        RuleFor(x => x.Age).GreaterThanOrEqualTo(18);
    }
}

// Pipeline behavior — валидация автоматом
public class ValidationBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly IEnumerable<IValidator<TRequest>> _validators;

    public async Task<TResponse> Handle(TRequest req, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var failures = _validators.SelectMany(v => v.Validate(req).Errors).ToList();
        if (failures.Any()) throw new ValidationException(failures);
        return await next();
    }
}
```

**Не нужно:** custom builder pattern, hand-rolled validators, "Strategy" — FluentValidation покрывает 99%.

См. [[cqrs-mediatr|CQRS & MediatR]].

### Сценарий 3: Wizard / Multi-step form (Создание заказа в 3 шага)

**Задача:** Пользователь заполняет данные в 3 экрана: товар → доставка → оплата. Между шагами state нужно сохранить, может уходить и возвращаться.

**Решение: State pattern + temp storage (Redis / DB):**

```csharp
public abstract record WizardState
{
    public sealed record SelectingProduct : WizardState;
    public sealed record EnteringShipping(int ProductId, int Quantity) : WizardState;
    public sealed record EnteringPayment(int ProductId, int Quantity, ShippingAddress Address) : WizardState;
    public sealed record Completed(OrderId OrderId) : WizardState;
}

public class WizardService
{
    public async Task<WizardState> NextStep(Guid sessionId, object input)
    {
        var current = await _storage.LoadAsync(sessionId);
        return current switch
        {
            WizardState.SelectingProduct => 
                new WizardState.EnteringShipping(((ProductInput)input).Id, ((ProductInput)input).Qty),
            WizardState.EnteringShipping s => 
                new WizardState.EnteringPayment(s.ProductId, s.Quantity, (ShippingAddress)input),
            WizardState.EnteringPayment p => 
                await Submit(p, (PaymentMethod)input),
            _ => throw new InvalidOperationException()
        };
    }
}
```

**Альтернативы:**

| Подход | Когда |
|--------|-------|
| State через discriminated union (records) | Modern .NET 6+, типобезопасно |
| Workflow engine (Elsa, MassTransit Saga) | Очень сложные wizards |
| Хранить в session / cookies | Простой UI без backend state |

**Plus/minus state pattern:**
- ✅ Невозможно перейти в illegal state
- ✅ Легко тестировать каждый переход
- ✅ Persistence-friendly (serialize state)
- ❌ Boilerplate если шагов мало

См. [[design-patterns#State|State Pattern]].

### Сценарий 4: Notification system (Email + SMS + Push)

**Задача:** При событиях отправлять уведомления. Канал зависит от user preferences.

**Решение: Strategy + Mediator (Domain Events)**

```csharp
public interface INotifier { Task Send(Notification n); }
public class EmailNotifier : INotifier { /* SMTP */ }
public class SmsNotifier : INotifier { /* Twilio */ }
public class PushNotifier : INotifier { /* Firebase */ }

// Domain event
public record OrderPlaced(OrderId Id, UserId Customer) : INotification;

// Handler
public class SendOrderConfirmation : INotificationHandler<OrderPlaced>
{
    private readonly IEnumerable<INotifier> _notifiers;
    private readonly IUserPrefs _prefs;

    public async Task Handle(OrderPlaced ev, CancellationToken ct)
    {
        var prefs = await _prefs.GetAsync(ev.Customer);
        var tasks = _notifiers
            .Where(n => prefs.Channels.Contains(n.Channel))
            .Select(n => n.Send(new("Order placed", $"#{ev.Id}")));
        await Task.WhenAll(tasks);
    }
}
```

**Когда message bus вместо Mediator:** если нотификации идут через границы сервисов (Order Service → Notification Service) → RabbitMQ / Kafka.

См. [[messaging|Messaging]].

### Сценарий 5: Drag-and-drop с undo/redo

**Задача:** Канбан доска. Пользователь перетаскивает карточки, нужен undo (Ctrl+Z).

**Решение: Command pattern**

```csharp
public interface ICommand
{
    void Execute();
    void Undo();
}

public class MoveCardCommand : ICommand
{
    private readonly Card _card;
    private readonly Column _from, _to;
    
    public MoveCardCommand(Card card, Column from, Column to)
    {
        _card = card; _from = from; _to = to;
    }
    
    public void Execute()
    {
        _from.Remove(_card);
        _to.Add(_card);
    }
    
    public void Undo()
    {
        _to.Remove(_card);
        _from.Add(_card);
    }
}

public class CommandHistory
{
    private readonly Stack<ICommand> _undo = new();
    private readonly Stack<ICommand> _redo = new();
    
    public void Execute(ICommand cmd)
    {
        cmd.Execute();
        _undo.Push(cmd);
        _redo.Clear();  // новый action — старый redo invalid
    }
    
    public void Undo()
    {
        if (_undo.TryPop(out var cmd)) { cmd.Undo(); _redo.Push(cmd); }
    }
    
    public void Redo()
    {
        if (_redo.TryPop(out var cmd)) { cmd.Execute(); _undo.Push(cmd); }
    }
}
```

**Когда:** любая система с undo/redo (text editors, image editors, kanban, level designers).

---

## 4. Сценарии: Модули (средние задачи)

### Сценарий 6: Корзина в e-commerce

**Задача:** Корзина пользователя. Нужно: добавить/удалить товары, посчитать total с учётом скидок, persist между сессиями.

**Архитектура:** **Aggregate (DDD)** + Repository

```csharp
// Aggregate root
public class Cart
{
    public CartId Id { get; }
    public UserId Owner { get; }
    private readonly List<CartItem> _items = new();
    public IReadOnlyList<CartItem> Items => _items;

    public void AddItem(ProductId productId, int quantity, Money price)
    {
        if (quantity <= 0) throw new DomainException("Quantity must be positive");
        
        var existing = _items.FirstOrDefault(i => i.ProductId == productId);
        if (existing != null)
            existing.IncreaseQuantity(quantity);
        else
            _items.Add(new CartItem(productId, quantity, price));
        
        AddDomainEvent(new CartItemAddedEvent(Id, productId, quantity));
    }

    public Money GetTotal(IDiscountStrategy discount) =>
        discount.Apply(_items.Sum(i => i.LineTotal));
}

// Discount как Strategy
public interface IDiscountStrategy { Money Apply(Money subtotal); }
public class NoDiscount : IDiscountStrategy { ... }
public class PercentDiscount(decimal Percent) : IDiscountStrategy { ... }
public class CouponDiscount(string Code) : IDiscountStrategy { ... }
```

**Хранение:**
- **Authenticated user** → DB (EF Core, aggregate persistence)
- **Anonymous** → Redis с TTL 7 дней
- **Hybrid** → переключение при login (merge anonymous + user cart)

**Под нагрузку:**
- Корзины — read/write heavy → **Redis** (in-memory + persistence)
- При checkout — переносим в SQL transaction

| Подход | Throughput | Когда |
|--------|------------|-------|
| EF Core + SQL | 1K RPS | Low traffic |
| Redis для активных + SQL для history | 50K+ RPS | E-commerce production |
| Полностью in-memory | 1M+ RPS | Гипер-нагрузка (но риск потери) |

См. [[ddd|DDD]] и [[ef-patterns|EF Patterns]].

### Сценарий 7: Подписка / биллинг

**Задача:** Saas с monthly subscription. Plans (Free/Pro/Enterprise), upgrade/downgrade, payment retry, cancellation, trials.

**Архитектура:** **State machine** + **Saga** + **Event Sourcing** (если audit нужен)

```csharp
public abstract record SubscriptionState
{
    public sealed record Trial(DateTime EndsAt) : SubscriptionState;
    public sealed record Active(PlanId Plan, DateTime NextBilling) : SubscriptionState;
    public sealed record PastDue(int FailedAttempts, DateTime LastTry) : SubscriptionState;
    public sealed record Cancelled(DateTime CancelledAt, string Reason) : SubscriptionState;
}

public class Subscription
{
    public SubscriptionState State { get; private set; }

    public void TryCharge() => State = State switch
    {
        Active a when ChargeSuccess() => a with { NextBilling = a.NextBilling.AddMonths(1) },
        Active a => new PastDue(1, DateTime.UtcNow),
        PastDue p when p.FailedAttempts >= 3 => new Cancelled(DateTime.UtcNow, "Payment failed"),
        PastDue p when ChargeSuccess() => new Active(GetPlan(), DateTime.UtcNow.AddMonths(1)),
        PastDue p => p with { FailedAttempts = p.FailedAttempts + 1, LastTry = DateTime.UtcNow },
        _ => throw new InvalidOperationException()
    };
}
```

**Нагрузка / scale:**
- Billing job ежедневный → BackgroundService
- Failed charges → Retry pattern (Polly) с exponential backoff
- Notifications → message bus (pub/sub)

**Compliance важно:** PCI-DSS если хранишь card data. Лучше использовать Stripe / Cloudpayments — они хранят, ты payment_method_id.

См. [[messaging|Messaging]] и [[distributed-systems|Distributed Systems]].

### Сценарий 8: Поиск по сайту

**Задача:** Пользователь ищет товары / контент. Полнотекстовый поиск, фильтры, фасеты.

**Решения по сложности:**

| Сложность | Решение | RPS | Cost |
|-----------|---------|-----|------|
| Simple | EF Core `Where + Contains` (ILIKE / LIKE) | 100 RPS | $ |
| Medium | PostgreSQL full-text search (`to_tsvector`) | 5K RPS | $$ |
| Heavy | **Elasticsearch / OpenSearch** + индексация через events | 50K+ RPS | $$$ |
| Enterprise | ES + ML ranking + personalization (vector search) | 100K+ | $$$$ |

**Архитектура с Elasticsearch (CQRS):**

```csharp
// Write model — SQL + EF Core
public class ProductService
{
    public async Task UpdateProduct(Product p)
    {
        await _db.SaveChangesAsync();
        await _bus.Publish(new ProductUpdated(p.Id));  // event
    }
}

// Read model — Elasticsearch
public class IndexProductHandler : INotificationHandler<ProductUpdated>
{
    public async Task Handle(ProductUpdated ev, CancellationToken ct)
    {
        var product = await _db.Products.FindAsync(ev.Id);
        await _es.IndexAsync(product.ToSearchDocument());
    }
}

// Search API
public class SearchService
{
    public async Task<SearchResults> Search(SearchQuery q) =>
        await _es.SearchAsync<ProductDocument>(s => s
            .Query(qq => qq.Match(m => m.Field(p => p.Name).Query(q.Text)))
            .Aggregations(...));  // фасеты
}
```

**Plus/minus CQRS с ES:**
- ✅ Search scale x100
- ✅ Faceted search out-of-box
- ✅ ML / typo tolerance
- ❌ Eventual consistency (1-5 sec delay)
- ❌ Sync errors (re-indexing pipeline)
- ❌ Operational cost ES cluster

См. [[cqrs-mediatr|CQRS]].

### Сценарий 9: Reporting / Dashboard

**Задача:** Admin dashboard с графиками — sales, users, conversions. Запросы аналитические (GROUP BY, агрегации).

**Подходы:**

| Подход | Когда |
|--------|-------|
| **EF + LINQ** | Простые отчёты, < 100K rows |
| **Dapper + raw SQL** | Сложные запросы, средние данные |
| **Materialized views в DB** | Часто меняются параметры, средняя нагрузка |
| **Read replica + Dapper** | Read scale нужен, не блокировать main DB |
| **OLAP cube / ClickHouse / TimescaleDB** | Big data, time-series, миллиарды rows |
| **Pre-computed daily aggregations** | Snapshot-style reports |

**Пример Dapper для report:**

```csharp
public async Task<MonthlyReport> GetReport(DateTime month)
{
    const string sql = @"
        SELECT 
            DATE_TRUNC('day', created_at) as Day,
            COUNT(*) as Orders,
            SUM(total) as Revenue,
            AVG(total) as AvgOrder
        FROM orders
        WHERE created_at >= @Start AND created_at < @End
        GROUP BY DATE_TRUNC('day', created_at)
        ORDER BY Day";
    
    var rows = await _conn.QueryAsync<DayStats>(sql, new { 
        Start = month, 
        End = month.AddMonths(1) 
    });
    
    return new MonthlyReport(rows.ToList());
}
```

**Для тяжёлой аналитики:**
- Read replica → Dapper
- Кеш на 5-15 минут (Redis)
- Pre-aggregated tables: `daily_sales`, `monthly_users`

См. [[dapper-comparison|Dapper vs EF]].

### Сценарий 10: Import / Export CSV

**Задача:** Загрузить CSV с 100K-1M rows, валидировать, импортировать в DB.

**Решение: Pipeline + Streaming + Batching**

```csharp
public async Task ImportAsync(Stream csvStream, CancellationToken ct)
{
    using var reader = new StreamReader(csvStream);
    using var csv = new CsvReader(reader, CultureInfo.InvariantCulture);
    
    var batch = new List<Product>(1000);
    var errors = new List<ImportError>();
    int row = 0;
    
    await foreach (var record in csv.GetRecordsAsync<ProductDto>(ct))
    {
        row++;
        
        var validation = _validator.Validate(record);
        if (!validation.IsValid)
        {
            errors.Add(new ImportError(row, validation.Errors));
            continue;
        }
        
        batch.Add(record.ToEntity());
        
        if (batch.Count == 1000)
        {
            await _db.Products.AddRangeAsync(batch, ct);
            await _db.SaveChangesAsync(ct);
            batch.Clear();
        }
    }
    
    if (batch.Any())
    {
        await _db.Products.AddRangeAsync(batch, ct);
        await _db.SaveChangesAsync(ct);
    }
}
```

**Patterns:**
- **Streaming** (`IAsyncEnumerable`) — не загружай весь файл в память
- **Batching** — `SaveChanges` каждые 1000 rows
- **Bulk insert** — для очень больших → `EFCore.BulkExtensions` или Dapper + COPY (Postgres)

**Под нагрузкой:**
- Малый файл (< 10K rows) → синхронно
- Большой → BackgroundService + progress endpoint
- Очень большой → message queue + worker service

См. [[io-streams|I/O & Streams]] и [[iterators-yield|Iterators & yield]].

---

## 5. Сценарии: Системы целиком

### Сценарий 11: Internal Admin Tool (1-2 dev, 2 weeks)

**Пример:** Внутренняя админка для саппорта — список тикетов, изменение статусов.

**Стек:**
```
Frontend:    Blazor Server (или ASP.NET MVC + Razor)
Backend:     ASP.NET Core minimal API
Architecture: 1 проект, без слоёв
Data:        EF Core + SQL Server / PostgreSQL
Auth:        Windows Auth (если internal) или Cookie auth
Deploy:      IIS / single Linux VM
```

**Patterns:**
- ❌ НЕ нужно: MediatR, CQRS, DDD, Microservices
- ✅ Нужно: Built-in DI, EF Core, минимальная structure

**Plus/minus:**
- ✅ Быстро запустить (1-2 недели)
- ✅ Низкие costs ($20/month VM)
- ❌ Не масштабируется к 10+ developers
- ❌ Тестирование boilerplate

**Когда менять:** появилась 5+ entities, бизнес-логика стала сложной, > 2 developers → перейти на N-Layer.

### Сценарий 12: Малый интернет-магазин (3-5 dev, 6 months)

**Пример:** B2C shop с 1000-10000 SKU, 100-1000 заказов/день.

**Архитектура:** **Modular Monolith + Clean Architecture lite**

```
src/
├── Shop.Web/                    # ASP.NET Core, controllers
├── Shop.Application/             # MediatR handlers, DTOs
├── Shop.Domain/                  # Entities, value objects
├── Shop.Infrastructure/          # EF Core, external APIs
├── Shop.Modules.Catalog/         # Каталог товаров
├── Shop.Modules.Cart/            # Корзина (Redis + EF)
├── Shop.Modules.Orders/          # Заказы
├── Shop.Modules.Payment/         # Stripe / эквайринг
├── Shop.Modules.Shipping/        # Доставка
└── Shop.Modules.Notification/    # Email + SMS
```

**Стек:**
```
Frontend:    React / Vue / Blazor
Backend:     ASP.NET Core, MediatR, FluentValidation
Database:    PostgreSQL (main) + Redis (cart, sessions, cache)
Search:      PostgreSQL FTS (для < 100K товаров)
Payment:     Stripe / CloudPayments
Storage:     S3 / Azure Blob (изображения)
Deploy:      Docker + 1-2 VM или Azure App Service
Monitoring:  Application Insights / Serilog → Seq
```

**Под нагрузку:**
- 100 RPS → один сервер достаточно
- 1K RPS → load balancer + 2-3 instance, Redis кеш
- 10K RPS → CDN для static, Redis cluster, read replicas

**Patterns:**
- **Domain Events** — `OrderPlaced` → email + analytics + inventory update
- **Outbox pattern** — guaranteed delivery событий
- **Saga** для checkout (reserve inventory → charge → confirm)
- **Repository per Aggregate** (Order, Product, Customer)
- **Cache-aside** для product catalog (Redis)

**Plus/minus modular monolith:**
- ✅ Простой deploy (один app)
- ✅ Простая отладка (один процесс)
- ✅ Транзакции работают в одной БД
- ✅ Можно вырезать модуль в микросервис позже
- ❌ Один процесс — failure of one module = down всего
- ❌ Все devs работают в одном repo (merge conflicts)

См. [[architecture-patterns|Architecture Patterns]] и [[ddd|DDD]].

### Сценарий 13: Крупный интернет-магазин (15+ dev)

**Пример:** Marketplace типа Wildberries scale — миллионы SKU, 100K заказов/день, 1000 продавцов.

**Архитектура:** **Microservices** по bounded contexts

```
Services:
├── Identity Service        # Auth, users, roles
├── Catalog Service         # Товары, категории (Postgres + Elasticsearch)
├── Inventory Service       # Остатки на складах
├── Cart Service            # Корзины (Redis primary)
├── Order Service           # Заказы (Postgres)
├── Payment Service         # Платежи (PCI-DSS isolated)
├── Shipping Service        # Доставка, треккинг
├── Notification Service    # Email/SMS/Push
├── Search Service          # Elasticsearch + ML ranking
├── Recommendation Service  # ML personalization
├── Analytics Service       # Events → ClickHouse
└── Seller Service          # Кабинет продавца
```

**Стек:**
```
Backend:     ASP.NET Core (или Go / Java для perf-critical)
Communication: gRPC internal, REST external, Kafka events
Databases:   Postgres per service, Redis, Elasticsearch, ClickHouse, MongoDB
Cache:       Redis cluster, CDN (Cloudflare)
Message bus: Kafka (events), RabbitMQ (commands)
API Gateway: Kong / Yarp / Ocelot
Service mesh: Istio (для production-grade)
Deploy:      Kubernetes + Helm + ArgoCD
Observability: OpenTelemetry → Jaeger + Prometheus + Grafana + Loki
CI/CD:       GitHub Actions / GitLab CI
```

**Patterns:**
- **CQRS** на уровне сервиса (read из ES, write в Postgres)
- **Event Sourcing** для Order Service (audit critical)
- **Saga** для checkout (cross-service workflow)
- **Outbox pattern** в каждом service для guaranteed delivery
- **Circuit Breaker** (Polly) на всех external calls
- **Rate limiting** на API Gateway
- **Backend for Frontend (BFF)** — отдельный API per client (web / mobile / partners)

**Под нагрузку:**
- 100K RPS overall, distributed
- Каждый сервис scale независимо
- Redis cluster для cart (1M+ active sessions)
- ES cluster для search (миллиарды queries)

**Plus/minus microservices:**
- ✅ Independent scaling per service
- ✅ Independent teams (Conway's law alignment)
- ✅ Polyglot tech (best tool per service)
- ✅ Fault isolation (один service down ≠ всё down)
- ❌ Огромная operational complexity
- ❌ Distributed transactions (sagas required)
- ❌ Network latency (gRPC помогает но не убирает)
- ❌ Debugging — distributed tracing must
- ❌ Cost — Kubernetes cluster = $$$$
- ❌ Need DevOps team

См. [[microservices-vs-monolith|Microservices vs Monolith]] и [[distributed-systems|Distributed Systems]].

### Сценарий 14: Контент-портал / CMS / Новостной сайт

**Пример:** habr.com — статьи, комментарии, голоса. Read-heavy (95% reads, 5% writes).

**Архитектура:** **Modular Monolith** + **aggressive caching** + **CQRS lite**

```
Modules:
├── Content      # Статьи, теги
├── Users        # Авторы, читатели
├── Comments     # Комментарии (с вложенностью)
├── Voting       # Лайки/дизлайки
├── Search       # Elasticsearch
└── Analytics    # Просмотры
```

**Стек:**
```
Frontend:    Next.js (SSR для SEO) + CDN
Backend:     ASP.NET Core
Database:    PostgreSQL (main), Redis (cache + counters), ES (search)
Cache:       Multi-layer (CDN → Redis → in-memory)
Storage:     S3 (картинки, видео)
Deploy:      Docker + 2-3 servers + CDN
```

**Patterns:**
- **Cache-aside** — все статьи в Redis с TTL 5-15 min
- **Read replica** — Postgres replica для тяжёлых queries
- **Eventual consistency** для счётчиков просмотров (write через message bus)
- **CDN** для static + HTML (если SSR)
- **Pagination via cursor** (не offset — для infinite scroll)

**Под нагрузку:**
- Reads: 100K RPS — почти все из cache
- Writes: 1K RPS (publish/comment/vote) — direct в Postgres
- Trending posts → **дополнительный** cache layer (in-memory)

**Plus/minus:**
- ✅ Cache даёт x100 speedup на reads
- ✅ Один service — простой deploy
- ❌ Cache invalidation сложна (классическая проблема)
- ❌ Counters (просмотры) — eventual consistency

**Кеш стратегия:**
```csharp
// Cache-aside для статьи
public async Task<Article?> GetArticle(int id)
{
    var key = $"article:{id}";
    var cached = await _cache.GetAsync<Article>(key);
    if (cached != null) return cached;
    
    var article = await _db.Articles.FindAsync(id);
    if (article != null)
        await _cache.SetAsync(key, article, TimeSpan.FromMinutes(15));
    
    return article;
}

// Invalidate при update
public async Task UpdateArticle(Article a)
{
    await _db.SaveChangesAsync();
    await _cache.RemoveAsync($"article:{a.Id}");
}
```

См. [[caching|Caching]].

### Сценарий 15: SaaS B2B мульти-tenant (e.g., CRM)

**Пример:** Salesforce-like CRM — каждая компания (tenant) видит только свои данные.

**Архитектура:** **Modular Monolith** с **tenant isolation**

**3 стратегии isolation:**

| Стратегия | Описание | Pros | Cons |
|-----------|----------|------|------|
| **Shared DB, Shared Schema** | Все tenants в одних таблицах + `TenantId` column | Дёшево, легко scale | Risk утечки данных, 1 tenant может затормозить всех |
| **Shared DB, Separate Schema** | Один Postgres, schema per tenant | Изоляция логическая | Migrations сложнее |
| **DB per Tenant** | Каждому tenant своя БД | Полная изоляция, custom backups | Дорого, миграции х tenants |

**Реализация Shared Schema (most common):**

```csharp
// Все entity имеют TenantId
public abstract class TenantEntity
{
    public Guid TenantId { get; set; }
}

public class Customer : TenantEntity
{
    public string Name { get; set; }
}

// EF Global Query Filter
modelBuilder.Entity<Customer>()
    .HasQueryFilter(c => c.TenantId == _currentTenant.Id);

// Auto при SaveChanges
public override int SaveChanges()
{
    foreach (var entry in ChangeTracker.Entries<TenantEntity>())
        if (entry.State == EntityState.Added)
            entry.Entity.TenantId = _currentTenant.Id;
    return base.SaveChanges();
}
```

**Patterns:**
- **Tenant context** — `ICurrentTenant` injected via DI per request
- **Row-Level Security** в Postgres (defense-in-depth)
- **Tenant-aware caching** — `cache-key:{tenantId}:...`
- **Per-tenant configuration** — feature flags, branding

**Compliance:** GDPR, SOC2 — требуют tenant isolation на всех уровнях.

См. [[postgresql-deep|PostgreSQL Deep]] (Row-Level Security).

### Сценарий 16: HFT / High-Frequency Trading

**Пример:** Алготорговля — нужна latency < 1 ms.

**Архитектура:** **Single process**, никаких сетевых hop'ов внутри hot path

**Стек:**
```
Language:    C# (ServerGC, ReadyToRun) или C++
Memory:      ArrayPool, ObjectPool, Span<T>, ref struct, stackalloc
Threading:   Lock-free queues (Disruptor pattern)
Networking:  Kernel bypass (DPDK), UDP multicast
Storage:     In-memory + SSD для history
Deploy:      Bare metal, кросс-rack co-location у биржи
OS:          Linux RT kernel, CPU pinning, isolated cores
```

**Patterns:**
- **Object pooling** — никаких аллокаций в hot path
- **Span\<T\> / ref struct** — без heap
- **Lock-free** structures (Interlocked, ConcurrentQueue, Disruptor)
- **Memory-mapped files** для market data
- **Source generators** вместо reflection
- **AOT compilation** — predictable JIT-free latency

**НИКОГДА не используй:**
- ❌ async/await в hot path (state machine allocation)
- ❌ Reflection
- ❌ LINQ Enumerable (allocation per query)
- ❌ String concatenation
- ❌ Boxing
- ❌ Exception-based flow

См. [[hft-low-latency|HFT Performance]] и [[types-and-memory|Types & Memory]].

### Сценарий 17: IoT платформа (миллионы устройств)

**Пример:** Smart home / industrial IoT — устройства шлют телеметрию.

**Архитектура:** **Event-driven + Time-series DB**

**Стек:**
```
Ingestion:   MQTT broker (HiveMQ / EMQX) или Azure IoT Hub
Stream:      Kafka / Azure Event Hubs (1M+ msg/sec)
Processing:  Apache Flink / Kafka Streams / .NET Worker Services
Storage:     TimescaleDB / InfluxDB / Azure Data Explorer
Cold storage: S3 / Azure Data Lake
Analytics:   ClickHouse / Druid
```

**Patterns:**
- **Event Sourcing** — каждое measurement immutable
- **CQRS** — write = ingestion, read = aggregations
- **Backpressure** (Channels, IAsyncEnumerable)
- **Idempotency** — duplicates inevitable

**Под нагрузку:**
- 1M devices × 1 msg/min = 16K msg/sec → Kafka handles easily
- Storage: 1M rows/min = 1.4 billion/day → TimescaleDB hypertables

См. [[messaging|Messaging]].

### Сценарий 18: Бухгалтерская / финансовая система

**Пример:** ERP — нужен audit, нельзя терять operations, regulatory compliance.

**Архитектура:** **Event Sourcing + CQRS + DDD**

**Зачем event sourcing здесь:**
- **Audit** — каждое изменение записано (regulators требуют)
- **Replay** — bug в логике? Replay events с фиксом
- **Time-travel** — "покажи баланс на 1 января прошлого года"
- **Multiple projections** — общий ledger + tax report + analytics

```csharp
// Events
public abstract record AccountEvent;
public record AccountOpened(AccountId Id, string Owner) : AccountEvent;
public record MoneyDeposited(AccountId Id, Money Amount, string Reference) : AccountEvent;
public record MoneyWithdrawn(AccountId Id, Money Amount, string Reference) : AccountEvent;

// Aggregate из events
public class Account
{
    private readonly List<AccountEvent> _changes = new();
    public Money Balance { get; private set; }
    
    public static Account FromHistory(IEnumerable<AccountEvent> history)
    {
        var account = new Account();
        foreach (var e in history) account.Apply(e);
        return account;
    }
    
    public void Deposit(Money amount, string reference)
    {
        if (amount.Amount <= 0) throw new DomainException();
        var ev = new MoneyDeposited(Id, amount, reference);
        Apply(ev);
        _changes.Add(ev);
    }
    
    private void Apply(AccountEvent e) => Balance = e switch
    {
        MoneyDeposited d => Balance + d.Amount,
        MoneyWithdrawn w => Balance - w.Amount,
        _ => Balance
    };
}
```

**Стек:**
```
Event store:   EventStoreDB / Marten / самодельный поверх Postgres
Read models:   Postgres (current balances), Elasticsearch (search transactions)
Compliance:    Append-only logs, retention policies
Encryption:    At-rest (TDE) + in-transit (TLS)
```

**Plus/minus event sourcing:**
- ✅ Полный audit trail
- ✅ Time-travel и replay
- ✅ Multiple read models
- ❌ Огромная сложность
- ❌ Schema evolution events painful
- ❌ Storage растёт линейно

**НЕ используй ES:** для CRUD apps. Используй когда **audit реально важен**.

См. [[ddd|DDD]].

---

## 6. Эволюция: как растёт от MVP до scale

### Стартап lifecycle

```
Day 1-30: MVP
├── Single project, ASP.NET Core minimal API
├── EF Core + SQLite / Postgres
├── Один Heroku / Azure App Service
└── Никаких patterns — YAGNI

Month 2-6: Product-market fit
├── 3-Layer architecture
├── Несколько Controllers / Services
├── Postgres + Redis для session
├── 1 backend dev
└── Patterns: Decorator (logging), Strategy если есть выбор algorithms

Month 6-18: Growth (10K users)
├── Clean Architecture lite
├── MediatR + FluentValidation
├── Domain Events
├── Background services (BackgroundService)
├── Application Insights / Serilog
├── 3-5 dev
├── Load balancer + 2 instance
└── Patterns: Result<T,E>, Repository per aggregate, CQRS lite

Month 18+: Scale (100K+ users)
├── Modular Monolith с DDD
├── Извлечение модулей в микросервисы (по 1 за раз)
├── Kafka / RabbitMQ для events
├── Redis cluster
├── Read replicas
├── Kubernetes
├── 10+ dev
└── Patterns: Saga, Outbox, Circuit Breaker, full CQRS

3+ years: Enterprise
├── Microservices по bounded contexts
├── Service mesh (Istio)
├── BFF per client
├── Event sourcing если compliance
├── 50+ dev, multi-team
└── Full distributed systems toolkit
```

### Ключевые переходы (триггеры)

```
N-Layer → Clean Architecture:
  Trigger: тестирование стало больно, mocking everywhere

Clean → Modular Monolith:
  Trigger: > 10 dev, разные команды на разных фичах

Modular Monolith → Microservices:
  Trigger: Independent deployment / scaling нужен
  ⚠️ Не раньше! Microservices — last resort

Любая → CQRS:
  Trigger: Read scale нужен ОТДЕЛЬНО от write
  
Любая → Event Sourcing:
  Trigger: Audit / compliance / replay нужен
```

> [!warning] Главная ошибка стартапов
> "Нам нужны microservices с самого начала". Нет. Microservices решают проблемы scale, но создают проблемы complexity. Если у тебя 100 users — у тебя нет проблем scale, у тебя проблемы PMF (product-market fit).

См. [[microservices-vs-monolith|Microservices vs Monolith]].

---

## 7. Финальная decision matrix

### По типу продукта

| Продукт | Архитектура | Главные patterns | Stack |
|---------|-------------|------------------|-------|
| **Внутренний tool** | Single project | DI, EF Core | ASP.NET Core + Postgres |
| **Прототип / MVP** | Minimal API | None | ASP.NET Core + SQLite |
| **B2B SaaS** | Clean Architecture | MediatR, FluentValidation, Result | + Redis + Kafka if needed |
| **Малый магазин** | Modular Monolith lite | Domain Events, Outbox, Saga | + Stripe + S3 |
| **Большой e-commerce** | Microservices | CQRS, Saga, Circuit Breaker | + Kafka + ES + K8s |
| **Контент-сайт / CMS** | Monolith + cache | Cache-aside, CDN | + Redis + CDN + ES |
| **Социальная сеть** | Microservices + event-driven | Eventually consistent, fanout | + Kafka + Cassandra |
| **CRM / SaaS multi-tenant** | Modular Monolith | Tenant isolation, RLS | + Postgres RLS |
| **Real-time chat** | Specialized | SignalR / WebSocket | + Redis pub/sub |
| **Trading / HFT** | Single process, hot-path | Object pooling, lock-free | C#/C++ + bare metal |
| **IoT** | Event-driven | Stream processing | + Kafka + TimescaleDB |
| **Финтех / банки** | Event Sourcing + CQRS + DDD | Audit, append-only | + EventStoreDB |
| **Game backend** | Microservices + WebSocket | Real-time, state sync | + Redis + UDP |

### По нагрузке

| RPS | Архитектура | Стоимость | Что важно |
|-----|-------------|-----------|-----------|
| < 100 | Monolith, 1 server | $20-50/мес | YAGNI — не оптимизируй |
| 100-1K | Monolith, 2-3 instances + LB | $100-300/мес | Базовое кеширование |
| 1K-10K | Modular monolith + Redis + read replicas | $500-2000/мес | Cache, async I/O, indices |
| 10K-100K | Microservices + Kafka + K8s | $5K-20K/мес | Distributed tracing, circuit breakers |
| 100K+ | Specialized — CDN, edge, sharding | $20K+/мес | Custom solutions, no off-the-shelf |
| 1M+ | Bare metal, kernel bypass | $$$$$$ | HFT-style, < 1ms latency |

---

## 8. Cheat sheet — final

### Какой паттерн для какой задачи

| Задача | Паттерн | Где почитать |
|--------|---------|--------------|
| Меню навигации с ролями | Composite + Specification | [[patterns-decision-guide]] |
| Форма с валидацией | FluentValidation + MediatR pipeline | [[cqrs-mediatr]] |
| Multi-step wizard | State pattern (records) |[[design-patterns]] |
| Notifications через email/SMS/push | Strategy + Mediator |[[messaging]] |
| Undo/redo в UI | Command pattern |[[design-patterns]] |
| Корзина e-commerce | Aggregate (DDD) + Redis | [[ddd]] + [[ef-patterns|EF Patterns]] |
| Биллинг / подписки | State machine + Saga | [[distributed-systems]] |
| Поиск товаров | CQRS + Elasticsearch | [[cqrs-mediatr]] |
| Reporting / аналитика | Dapper + read replica |[[dapper-comparison]] |
| Import CSV millions rows | Streaming + batching |[[io-streams]] |
| Audit для бухгалтерии | Event Sourcing | [[ddd]] |

### Какая архитектура для какого размера

| Размер | Архитектура |
|--------|-------------|
| 1-2 dev, MVP | Single project |
| 3-5 dev, internal | N-Layer |
| 5-10 dev, SaaS | Clean Architecture |
| 10-15 dev, product | Modular Monolith |
| 15-30 dev, scale | Modular Monolith → split first painful module |
| 30+ dev, multi-team | Microservices |

### Performance impact ключевые

| Оптимизация | Impact |
|-------------|--------|
| Add Redis cache | 10-100x на cached reads |
| Add async/await | 10x throughput на I/O |
| Add database index | 10-1000x на query |
| Switch EF → Dapper в hot path | 1.5-2x |
| Add CDN для static | 100x на static assets |
| Switch REST → gRPC внутри | 5-10x |
| Vertical scaling (bigger VM) | 2-5x linear |
| Horizontal scaling (more VMs) | Near-linear если stateless |
| Microservices split | 0% performance, +scaling, +complexity |

---

## 9. Anti-patterns по сценариям

| Сценарий | Anti-pattern | Что вместо |
|----------|--------------|------------|
| Малый магазин | Microservices с 1-го дня | Modular monolith |
| Internal tool | Clean Architecture с 5 проектами | Single project |
| Read-heavy site | Без cache | Cache-aside Redis |
| CRUD app | DDD с aggregates | Anemic OK для CRUD |
| Прототип | TDD + 100% coverage | Прототип = throwaway |
| HFT | async/await в hot path | Synchronous + lock-free |
| Любой | "Repository поверх EF на всякий случай" | Direct DbContext |
| Любой | "Generic abstraction для одной impl" | YAGNI |

---

## Best Practices

### Best Practices

**Architecture:**
- **Decision driven by problem** — не "найди подходящий case", а "опиши case → видеть decision"
- **18 cases охватывают типовые** scenarios — для exotic ситуаций используй [[patterns-decision-guide|Patterns Decision Guide]]
- **Periodic review** — раз в год пересматривай: что изменилось, какие новые scenarios появились

**Использование в команде:**
- При **spec review** — указывай case study reference как baseline
- В **architecture decision records (ADR)** — ссылайся на case study если decision соответствует
- **Onboarding new developer** — read top-5 относящихся к stack cases

**Когда case study НЕ применим:**
- Domain слишком specific (medical / financial regulations)
- Performance requirements выходят за обычные (HFT < 1ms)
- Compliance constraints (data sovereignty, audit trail)

→ В этих случаях — design from first principles, case study как inspiration не template.


---

## Decision tree

```
Какую архитектуру выбрать?
│
├── Маленький проект, < 5 endpoints?
│   → Сценарий 11 (Internal admin) — Minimal API, single project
│
├── E-commerce / Catalog-driven?
│   ├── 1-1000 SKU → Сценарий 12 (small shop) — Modular monolith
│   ├── 1K-100K SKU → Сценарий 13 (medium e-commerce) — VSA
│   └── Million+ SKU → Сценарий 13 + microservices (Catalog/Orders/Payment separate)
│
├── Контент-portal / CMS?
│   → Сценарий 14 (CMS) — Hybrid (read-heavy, write-light)
│
├── B2B SaaS?
│   ├── < 50 tenants → Сценарий 15 (multi-tenant) — shared schema
│   ├── 50-500 tenants → Schema-per-tenant
│   └── Enterprise (data isolation требуется) → Database-per-tenant
│
├── Real-time нужен?
│   ├── Chat / notifications → SignalR + Redis backplane
│   └── HFT / sub-millisecond → Сценарий 16 (HFT) — custom arch
│
├── IoT?
│   → Сценарий 17 (IoT) — message broker + time-series DB
│
├── ML / AI integrated?
│   → Сценарий 18 (LLM-powered) — RAG pattern + vector DB
│
└── Desktop?
    └── WPF/MAUI — отдельная decision tree (см. desktop-frameworks.md)
```


---

## См. также

### Decision-making

- [[patterns-decision-guide|Patterns Decision Guide]] — framework для выбора
- [[architecture-decisions|Architecture Decisions (ADRs)]] — как документировать
- [[system-design|System Design]] — общий процесс

### Архитектуры

- [[architecture-patterns|Architecture Patterns]] — N-Layer / Clean / VSA
- [[microservices-vs-monolith|Microservices vs Monolith]]
- [[distributed-systems|Distributed Systems]]
- [[ddd|DDD]]
- [[cqrs-mediatr|CQRS & MediatR]]

### Performance

- [[performance|Performance]]
- [[hft-low-latency|HFT]]
- [[queries-performance|EF Queries Performance]]
- [[caching|Caching]]

### Infrastructure

- [[messaging|Messaging]]
- [[observability|Observability]]
- [[kubernetes|Kubernetes]]
- [[docker|Docker]]

## Reading list

- **Designing Data-Intensive Applications** — Martin Kleppmann (must-read для любого Senior)
- **Building Microservices** — Sam Newman
- **Microsoft eShopOnContainers** — github.com/dotnet-architecture/eShopOnContainers (real-world reference architecture)
- **The Twelve-Factor App** — 12factor.net
- **Microsoft Cloud Design Patterns** — learn.microsoft.com/azure/architecture/patterns
- **System Design Primer** — github.com/donnemartin/system-design-primer
- **Mark Seemann blog** — blog.ploeh.dk
- **Vladimir Khorikov blog** — enterprisecraftsmanship.com (DDD в .NET)
- **Andrew Lock blog** — andrewlock.net (.NET deep)
