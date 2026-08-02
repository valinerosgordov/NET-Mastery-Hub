---
tags: [architecture, microservices, monolith, modular-monolith, decision]
level: Middle to Senior
date: 2026-08-02
---

# Microservices vs Monolith — выбор архитектуры

> **Когда монолит, когда микросервисы, когда modular monolith**. Реальные trade-offs, не маркетинг. Закрывает: типы архитектур, decision criteria, миграция, anti-patterns ("distributed monolith").

---

## Что это, зачем и когда

### Главный вопрос

Нет правильного ответа "monolith vs microservices". Есть **trade-offs**, которые меняются в зависимости от:
- Размер команды
- Размер системы
- Domain complexity
- Performance / availability requirements
- Operational maturity (DevOps, monitoring)

**Senior signal:** не "я люблю микросервисы", а "какой trade-off правильнее в этом контексте?".

---

## 1. Виды архитектур

### Single-process Monolith

Один deployable artifact. Одна database. Один process.

```
┌─────────────────────────────┐
│      One .NET App           │
│  ┌────────────────────────┐ │
│  │ Users / Orders /       │ │
│  │ Products / Payments    │ │
│  │ (всё в одной БД)       │ │
│  └────────────────────────┘ │
└─────────────────────────────┘
              │
        ┌─────────┐
        │   DB    │
        └─────────┘
```

✅ **Хорошо когда:**
- Стартап / MVP / прототип
- Команда 1-5 разработчиков
- Простой domain (CRUD)
- Не знаешь bounded contexts ещё

❌ **Плохо когда:**
- 10+ разработчиков (merge conflicts)
- Большой domain без separation
- Разные части требуют разной scale
- Tight coupling между modules

### Modular Monolith ⭐ (recommended default)

Один deployable, но **внутри логически разделён** на модули.

```
┌──────────────────────────────────────┐
│         One .NET App                 │
│  ┌──────────────┐  ┌──────────────┐ │
│  │ Users module │  │ Orders mod.  │ │
│  │  - Domain    │  │  - Domain    │ │
│  │  - DB schema │  │  - DB schema │ │
│  └──────┬───────┘  └──────┬───────┘ │
│         │ events           │        │
│         ▼ via mediator     ▼        │
│  ┌──────────────┐  ┌──────────────┐ │
│  │ Products mod │  │ Payments mod │ │
│  └──────────────┘  └──────────────┘ │
└──────────────────────────────────────┘
              │
     ┌────────────────┐
     │  One DB        │
     │  schema_users  │
     │  schema_orders │
     │  ...           │
     └────────────────┘
```

**Принципы:**
- Каждый модуль = **bounded context** (DDD)
- Модули общаются только через **public API** (commands, queries, events)
- Internal types — `internal` для модуля
- Отдельные DB schemas (psql) или prefixed tables
- NetArchTest enforces правила

✅ **Хорошо когда:**
- Domain имеет чёткие boundaries
- Нужна гибкость "potentially split в micro" later
- Команда 5-30 разработчиков
- **Default для большинства новых проектов 2026**

См. [[architecture-patterns|Architecture Patterns]] — детальный гайд по Modular Monolith.

### Microservices

Несколько independent сервисов. Каждый — own DB, own deploy, own team.

```
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Users svc    │ │ Orders svc   │ │ Products svc │
│  + DB        │ │  + DB        │ │  + DB        │
└──────┬───────┘ └──────┬───────┘ └──────┬───────┘
       │                │                │
       └────────────────┼────────────────┘
                        │
              Message Bus / API Gateway
```

✅ **Хорошо когда:**
- Очень большая система (50+ developers)
- Разные части независимы (можно deploy independently)
- Different scale requirements (Search 1000 RPS vs Admin 1 RPS)
- Different tech stacks нужны (Python ML + .NET API)
- Team Topologies — Conway's law in play

❌ **Плохо когда:**
- Стартап (overhead убьёт скорость)
- DevOps незрелый (можешь не handle complexity)
- Domain не чёткий (bad boundaries → constantly redrawing)

### Distributed Monolith ❌ (anti-pattern)

Микросервисы по форме, но **tight coupling** в реальности:

- Sync calls между всеми services
- Shared DB (или shared schema)
- Deploy всех вместе обязательно
- Cascading failures
- Race conditions через все services

**Worst of both worlds** — complexity микросервисов без преимуществ.

```
Service A ──sync──▶ Service B ──sync──▶ Service C ──sync──▶ Service D
                                                                 │
                            All share same DB ◀──────────────────┘
```

**Признаки:**
- "Чтобы добавить feature нужно деплоить 5 services"
- "Если один сервис лежит — всё падает"
- "Тесты гоняем всё вместе"
- "Один из микросервисов меняем — приходится менять других"

---

## 2. Trade-offs — детально

### Independence (independent deployment)

| | Monolith | Modular | Micro |
|--|---------|---------|-------|
| Deploy independently | ❌ | ❌ | ✅ |
| Release cadence | Same | Same | Independent |
| Rollback impact | All | All | Per service |
| Coordination overhead | Low | Low | High (versioning) |

### Database

| | Monolith | Modular | Micro |
|--|---------|---------|-------|
| One DB | ✅ | ✅ (logical separation) | ❌ |
| Cross-module transactions | ✅ ACID | ✅ ACID | ❌ Saga / Outbox |
| Foreign keys cross-module | ✅ | ⚠️ Avoid | ❌ |
| Schema migrations | Simple | Simple | Coordinated |

См. [[distributed-systems|Distributed Systems]] — Saga, Outbox.

### Performance

| | Monolith | Modular | Micro |
|--|---------|---------|-------|
| Inter-module call | In-process (ns) | In-process (ns) | Network (ms) |
| Latency | Low | Low | Higher |
| Network overhead | None | None | Significant |
| Caching | Easy (in-memory) | Easy | Distributed cache |

### Scalability

| | Monolith | Modular | Micro |
|--|---------|---------|-------|
| Scale whole app | ✅ | ✅ | — |
| Scale specific feature | ❌ | ❌ | ✅ |
| Resource efficiency | ✅ | ✅ | Often wasteful (overhead per service) |

### Failure isolation

| | Monolith | Modular | Micro |
|--|---------|---------|-------|
| One module crashes | All down | All down | Only that service |
| Cascading failure | N/A | N/A | High risk без circuit breakers |
| Recovery | Restart app | Restart app | Restart service |

### Development velocity

| | Monolith | Modular | Micro |
|--|---------|---------|-------|
| Initial setup | Fast | Medium | Slow |
| Team friction | High at scale | Low | Lowest (ownership) |
| Cross-cutting changes | Single PR | Single PR | Multiple PRs / coordination |
| Onboarding | Easy (one repo) | Easy | Harder (which service?) |

### Testing

| | Monolith | Modular | Micro |
|--|---------|---------|-------|
| Unit tests | Easy | Easy | Easy |
| Integration tests | Easy | Easy | Harder (mock services) |
| E2E tests | Easy | Easy | Hard (multiple services up) |
| Contract tests | N/A | N/A | Pact / similar |

### Operational complexity

| | Monolith | Modular | Micro |
|--|---------|---------|-------|
| Logs aggregation | Simple | Simple | Hard (correlation) |
| Distributed tracing | Not needed | Not needed | Required (OTel) |
| Service discovery | N/A | N/A | Required |
| API gateway | Optional | Optional | Required |
| Secrets management | One config | One config | Per service |
| Monitoring overhead | Low | Low | High |

См. [[observability|Observability]].

---

## 3. Decision criteria

### Размер команды

```
1-5 devs:    Monolith / Modular Monolith
5-20 devs:   Modular Monolith ⭐
20-50 devs:  Modular Monolith → split важных частей
50+ devs:    Microservices (Conway's law forces it)
```

### Domain complexity

```
Simple CRUD:      Monolith
Medium domain:    Modular Monolith ⭐
Complex domain:   Modular Monolith с ясными boundaries
                  → micro если boundaries stable
Multiple domains: Microservices
```

### Scale requirements

```
< 100 RPS:        Monolith fine
100-1000 RPS:     Monolith / Modular OK
1000-10k RPS:     Modular / strategic micro
10k+ RPS:         Micro для hot endpoints
Variable scale:   Micro (different parts need different scale)
```

### Operational maturity

```
No DevOps team:       Monolith only
Junior DevOps:        Modular Monolith
Mature DevOps:        Microservices possible
Enterprise platform:  Microservices likely needed
```

---

## 4. Когда modular monolith — лучший выбор

В 2026 — **modular monolith это default** для большинства новых проектов.

### Преимущества

✅ Простота операций (один deploy, один process)
✅ ACID транзакции внутри модулей
✅ In-process calls — fast
✅ Refactor границ модулей **легко** (один codebase)
✅ Рост команды поддерживается до 30+ разработчиков
✅ **Можно разделить на micro потом** (если действительно надо)

### Структура

```
src/
├── Modules/
│   ├── Users/
│   │   ├── Users.Domain/         # entities, value objects
│   │   ├── Users.Application/    # commands, queries, handlers
│   │   ├── Users.Infrastructure/ # EF, repositories
│   │   └── Users.Api/            # controllers / minimal APIs
│   ├── Orders/
│   │   ├── Orders.Domain/
│   │   └── ...
│   └── Products/
│       └── ...
├── Shared/
│   ├── Shared.Kernel/            # cross-cutting (Result, DomainEvent)
│   └── Shared.Infrastructure/    # logging, telemetry
└── Bootstrap/
    └── WebApi/                   # composition root
```

### Inter-module communication

Публичные contracts модуля (events, commands) выносятся в отдельную **contracts-assembly**; доставка in-process — **свой dispatcher** (~50 строк) или **Wolverine**; для гарантированной cross-module доставки — **integration events через outbox** (см. [[distributed-systems|Distributed Systems]]).

> [!warning] MediatR 13+ — коммерческий (dual-license с июля 2025)
> Код ниже использует интерфейсы `INotification`/`IRequestHandler` — API валиден для MediatR ≤12.x (Apache 2.0 навсегда) и для `Mediator` (source-gen, MIT). Линия замен — [[choosing-dependencies|Choosing Dependencies]].

```csharp
// Module Users — публичный contract (contracts-assembly)
public record UserCreatedEvent(Guid UserId, string Email) : INotification;

// Module Users — internal implementation
internal class CreateUserHandler : IRequestHandler<CreateUserCommand, Guid>
{
    public async Task<Guid> Handle(CreateUserCommand cmd, CancellationToken ct)
    {
        var user = User.Create(cmd.Email);
        await _users.AddAsync(user);
        await _mediator.Publish(new UserCreatedEvent(user.Id, user.Email));
        return user.Id;
    }
}

// Module Notifications — слушает event
internal class UserCreatedHandler : INotificationHandler<UserCreatedEvent>
{
    public async Task Handle(UserCreatedEvent evt, CancellationToken ct)
    {
        await _email.SendWelcomeAsync(evt.Email);
    }
}
```

Другой модуль **не знает** internals — только event-contract из contracts-assembly.

### Architecture tests enforce правила

```csharp
[Fact]
public void Users_module_should_not_reference_Orders_internals()
{
    Types.InAssembly(typeof(UsersModule).Assembly)
        .ShouldNot()
        .HaveDependencyOn("MyApp.Modules.Orders")
        .GetResult().IsSuccessful.Should().BeTrue();
}
```

См. [[arch-tests|Architecture Tests]].

---

## 5. Migration strategies

### Monolith → Modular Monolith

**Постепенно**:

1. Identify bounded contexts (domain analysis)
2. Move classes в module folders (`Modules/Users/`, etc.)
3. Mark internal — `internal` modifier
4. Replace direct calls с commands/queries через in-process dispatcher
5. Add architecture tests

Можно **в течение года** при работе над feature requests, не "большой rewrite".

### Modular Monolith → Microservices

**Только если:**
- Modular monolith **реально не справляется** (proof, не предположение)
- Bounded contexts **stable** (не меняются ежемесячно)
- Operational team **готова**
- Business value > complexity cost

**Strangler Fig pattern:**

```
Step 1: Monolith handles all
  ┌──────────────┐
  │  Monolith    │
  └──────────────┘

Step 2: Extract один module как микро
  ┌──────────────┐  ┌─────────┐
  │  Monolith    │  │ Users   │
  │  (without    │←─┤ service │
  │   users)     │  └─────────┘
  └──────────────┘

Step 3: Continue extracting
  ┌──────────┐ ┌─────────┐ ┌─────────┐
  │ Monolith │ │ Users   │ │ Orders  │
  │ (small)  │ │ service │ │ service │
  └──────────┘ └─────────┘ └─────────┘

Step 4: Eventually monolith disappears
```

Слой между — **API gateway** или **message broker**.

### Microservices → Monolith (yes, this happens!)

Когда понимаешь что over-engineered. **Re-aggregation** — модный новый термин 2024+.

Признаки что нужен:
- Cascading failures постоянно
- Latency растёт от network calls
- Team spends >30% time на infrastructure не features
- Distributed bugs (eventual consistency issues) — основной source багов

См. **Amazon Prime Video moved monolith** — well-known case study (2023).

---

## 6. Реальный пример — кейсы

### Case 1: Стартап MVP

```
Команда: 3 разработчика
Domain: Marketplace для книг
Traffic: ~10 RPS expected
Timeline: 3 месяца до launch
```

**Выбор:** Single-process Monolith (или modular monolith если есть время).

**Не делай:** Microservices. Убивает velocity. 3 человека не справятся с infrastructure.

### Case 2: Растущий продукт

```
Команда: 15 разработчиков (3 squad по 5)
Domain: SaaS платформа CRM
Traffic: 100-500 RPS
Stage: 2 года after launch, growing
```

**Выбор:** Modular Monolith ⭐

**Структура:**
- Module: Users, Customers, Deals, Reports
- One DB с separated schemas
- One deploy
- 3 squads — каждый owns 1-2 modules

### Case 3: Большой enterprise

```
Команда: 200 разработчиков
Domain: Banking platform
Traffic: Spike до 50k RPS на payments
Compliance: PCI-DSS, SOC 2
```

**Выбор:** Microservices (с **серьёзным** infrastructure investment).

**Почему:**
- Conway's law: 200 человек физически не работают в одном codebase
- Different scale (Auth ~1000 RPS, Payments ~50k RPS)
- Compliance requires isolation
- Team Topologies — stream-aligned teams own services

### Case 4: ML platform

```
Команда: 10 разработчиков (часть DS, часть .NET)
Stack: .NET API + Python ML models
Traffic: ~50 RPS
```

**Выбор:** Hybrid — Modular Monolith (.NET) + Python service для inference.

**Почему:** разные tech stacks **forced** separation, но остальное может быть в monolith.

---

## 7. Common Pitfalls

### 1. "Microservices — modern, monolith — legacy"

False. Modular monolith — **modern** approach. Microservices — для специфичных случаев.

### 2. "Distributed monolith"

Микро по форме, tight coupling реально:
- Sync calls everywhere
- Shared DB
- Cascade failures

**Лечение:** event-driven, async messaging, true independence.

### 3. Premature microservices

```
Команда из 5 человек
Старт проекта
"Делаем микросервисы потому что Netflix"
6 месяцев на infrastructure
0 features
```

**Лечение:** start monolith, evolve.

### 4. Over-modularization

```
20 модулей в monolith
Каждый module использует 3 других
Coupling сильный
Никаких преимуществ
```

**Лечение:** ~5-10 модулей max. Модули должны быть **independent enough**.

### 5. Wrong boundaries

```
Module: "User"
Module: "Database access"
Module: "Validation"
```

Это **layers**, не **bounded contexts**! Boundaries — по domain, не по technical concern.

```
✅ Bounded contexts:
  Module: "Sales" (orders, customers, payments)
  Module: "Inventory" (products, stock)
  Module: "Shipping" (logistics)
```

### 6. Microservices без observability

Любая нетривиальная microservice architecture **требует**:
- Distributed tracing (OpenTelemetry)
- Centralized logging (ELK / Datadog)
- Metrics (Prometheus / Grafana)
- Alerting

Без этого — debugging = ад.

См. [[observability|Observability]].

### 7. Microservices с shared DB

```
Service A ──┐
Service B ──┼─→ Same DB
Service C ──┘
```

**Anti-pattern.** Service не может deploy independently — schema changes coordinated. Effectively distributed monolith.

**Правило:** один service — одна DB.

---

## 8. Best Practices

### Monolith

- Чистая структура папок (по features / domains)
- Layered architecture внутри
- Tests
- One DB достаточно

### Modular Monolith

- Modules = bounded contexts
- Public API через commands/queries/events — contracts-assembly + in-process dispatcher
- Internal types — `internal` modifier
- Architecture tests (NetArchTest)
- Per-module DB schema если возможно
- Одна solution, много projects (или один с folders)

### Microservices

- One service = one DB
- Async communication > sync (message broker)
- Idempotency keys везде
- Saga / Outbox для cross-service transactions
- Circuit breakers (Polly)
- Distributed tracing (OpenTelemetry)
- API gateway / BFF
- Service mesh для сложных cases (Istio)
- Schema evolution (backward compatible)
- Contract tests (Pact)

См. [[distributed-systems|Distributed Systems]].

---

## 9. Checklist — какую выбрать

```
□ Команда < 10? → Monolith / Modular Monolith
□ Domain простой (CRUD)? → Monolith
□ Domain неясен ещё? → Monolith (легко refactor)
□ Compliance требует isolation? → Microservices
□ Different scale needs? → Microservices
□ Different tech stacks? → Microservices
□ DevOps mature? → Microservices possible
□ DevOps junior? → Не микро
□ < 6 месяцев до launch? → Monolith
□ "Хотим как FAANG"? → Stop. Не повод.
□ Domain stable bounded contexts? → Modular → micro evolution
□ Domain meняется? → Stay monolith
```

---

## 10. Когда **НЕ** делай microservices

❌ Стартап с 5 разработчиками
❌ MVP / proof-of-concept
❌ Domain не чёткий
❌ Нет DevOps экспертизы
❌ Бюджет ограничен
❌ "Потому что модно"
❌ "Потому что Netflix"
❌ "Потому что хочется на собес опыт"

---

## Cheat sheet

| Need | Cheat |
|------|-------|
| One service one responsibility | **Microservices** |
| Single team, fast iteration | **Monolith** |
| Easy testing, simple deploy | **Monolith** |
| Independent scaling per service | **Microservices** |
| Polyglot tech (Python ML + .NET API) | **Microservices** |
| < 5 developers | **Monolith** (almost always) |
| > 50 developers, conflicting changes | **Microservices** |
| Strong consistency requirements | **Monolith** (или modular monolith) |
| Different uptime / SLA per part | **Microservices** |
| Complex distributed transactions | **Monolith** |
| Quick MVP / startup | **Monolith** |
| Established business, scale issues | **Migrate selectively** (strangler fig) |

**Modular Monolith** — golden middle:
- Boundaries clear (как microservices)
- Single deploy unit (как monolith)
- Refactor → microservices later if нужно


---

## Decision tree

```
Monolith vs Microservices?
│
├── Размер команды?
│   ├── 1-5 dev → Monolith
│   ├── 5-20 dev → Modular monolith
│   └── 20+ dev → Microservices начинают давать profit
│
├── Скорость изменений business?
│   ├── Frequent changes, all parts → Monolith
│   ├── Different parts разные tempo → Microservices
│   └── Slow, mature → не имеет значения
│
├── Operational maturity?
│   ├── Нет DevOps, k8s, observability → Monolith
│   └── Есть → Microservices feasible
│
├── Performance / latency?
│   ├── Sub-millisecond требуется → Monolith (no network hops)
│   ├── Independent scaling нужно → Microservices
│   └── Average → не имеет значения
│
├── Domain complexity?
│   ├── Single bounded context → Monolith
│   ├── Multiple clear contexts → Microservices follow boundaries
│   └── Unclear → Modular monolith first, split later
│
└── Deployment frequency?
    ├── Once a week → Monolith OK
    ├── Multiple times daily → Microservices помогают
    └── Continuous → Microservices feature-flagged
```

**The golden rule:** Start with monolith. Extract microservice when **specific pain** justifies overhead.


---

## См. также

- [[architecture-patterns|Architecture Patterns]] — Modular Monolith / VSA / Clean
- [[ddd|DDD]] — bounded contexts
- [[distributed-systems|Distributed Systems]] — Saga, Outbox, eventual consistency
- [[arch-tests|Architecture Tests]] — enforce module boundaries
- [[observability|Observability]] — required для micro
- [[03_middle-to-senior|Middle → Senior]]

## Reading list

- **Building Microservices** — Sam Newman (классика)
- **Microservices Patterns** — Chris Richardson
- **Monolith to Microservices** — Sam Newman
- **The Modular Monolith** — Kamil Grzybek (blog series)
- **Domain-Driven Design** — Eric Evans
- **Team Topologies** — Skelton & Pais (Conway's law modern)
- **Amazon Prime Video case study** (2023) — micro → monolith real example
- **Andrew Lock — modular monolith series** — andrewlock.net
- **Vladimir Khorikov — DDD + Modular** — enterprisecraftsmanship.com
