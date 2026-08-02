---
tags: [learning-path, interview, prep, checklist]
level: All
date: 2026-08-02
---

# 🎤 Interview Prep

> Подготовка к техническому собеседованию на .NET Backend позиции. От Junior до Senior. Что повторить, какие задачи решать, на чём фокус.

---

## Что это, зачем и когда

> Подготовка к техническому собеседованию на .NET Backend позиции. От Junior до Senior. Что повторить, какие задачи решать, на чём фокус.

### Зачем эта страница

Собес на Senior .NET — стресс. Без чёткого плана:
- Учишь хаотично — забываешь что прошло, что нет
- Учишь не то — узнают не базу, а advanced topics
- Не practiced live coding — ступор у whiteboard
- Не подготовлен behavioral — рассказы беспорядочные

**Решение:** structured prep по weeks/days с links на нужные файлы vault'а.

### Структура prep

```
2 недели:
├── Week 1: Knowledge review (повторение по topics)
└── Week 2: Practice (coding, system design, mock interviews)

Day-by-day:
├── 1-2 дня — C# fundamentals
├── 3 — async / concurrency
├── 4 — ASP.NET Core
├── 5 — EF Core / SQL
├── 6 — Architecture / patterns
├── 7 — Testing
├── 8-9 — System design study
├── 10-11 — Live coding practice
├── 12-13 — Mock interviews
└── 14 — Behavioral (STAR stories)
```

### Уровни сложности

| Level | Focus |
|-------|-------|
| **Junior** | C# basics, simple LINQ, basic ASP.NET Core, basic SQL |
| **Middle** | + async, design patterns, REST, testing, EF Core |
| **Senior** | + architecture, performance, distributed systems, system design |
| **Lead/Architect** | + team management, technical strategy, decision making |

> [!info] Главный совет
> **Honesty > pretending to know**. Senior interviewer видит насквозь. Лучше "не знаю, но я бы подошёл так..." чем neuverenный bullshit.

---

## Case Studies — реальные собеседования

### Case Study #1 — Senior .NET в Product company

**Сценарий:** 3-этапный собес — screening, technical, system design.

**Этап 1: Screening (45 мин)**

Вопросы:
- "Расскажи о себе и последнем проекте"
- "Что было самым сложным?"
- "Почему уходишь из текущего места?"

Pitfalls:
- Слишком long ответ ("про себя 15 минут")
- Negative о предыдущем employer (red flag)
- Vague "сложности" без конкретики

**Что делать:** STAR метод (Situation, Task, Action, Result). 2-3 минуты на ответ.

---

**Этап 2: Technical deep-dive (90 мин)**

Вопросы:
- "Как работает async/await под капотом?"
- "Расскажи про GC. Что такое Gen 0/1/2/LOH?"
- "Live coding: написать thread-safe cache с TTL"
- "Что выберешь — `Channel<T>` или `BlockingCollection`?"

**Что готовить:** [[async-threading|async-threading]], [[gc-memory|gc-memory]], [[concurrency-atomics|concurrency-atomics]].

---

**Этап 3: System design (60 мин)**

Задача: "Спроектируй URL shortener на 1B requests/day"

Что нужно покрыть:
1. Functional requirements (что делает)
2. Non-functional (scale, latency, availability)
3. Capacity estimation (storage, RPS, bandwidth)
4. API design
5. Data model
6. High-level architecture
7. Deep dive в bottlenecks
8. Trade-offs

См. [[real-world-scenarios|real-world-scenarios]] и [[system-design|system-design]].

---

### Case Study #2 — Outsourcing company

**Сценарий:** Broad knowledge тестируют. Less depth, more breadth.

**Format:**
- 60 мин — Q&A по wide topics
- 30 мин — small coding task

**Topics covered:**
- C# basics (nullable types, patterns, LINQ)
- ASP.NET Core (middleware, DI lifetimes)
- EF Core (tracking, migrations, performance)
- SQL (joins, indexes, transactions)
- Patterns (когда какой)
- DevOps awareness (Docker, CI/CD basics)

**Strategy:** review broad topics, не углубляйся в один. Vault [[02_junior-to-middle|Junior to Middle]] + [[03_middle-to-senior|Middle to Senior]] roadmaps хороши.

---

### Case Study #3 — Startup

**Сценарий:** Pragmatic, full-stack thinking ценится.

**Format:**
- 45 мин — обсуждение experience
- 60 мин — pair programming на real problem
- 30 мин — Q&A о products

**Что ценится:**
- Ability to ship быстро
- Pragmatism (не over-engineering)
- Product thinking (зачем feature?)
- Comfort с uncertainty
- Wide skill set (frontend, deploy, monitoring — не только backend)

**Strategy:** покажи что MVP можешь shipnть, бизнес-понимание есть, не перфекционист.

---

### Case Study #4 — Behavioral round

**Сценарий:** "Tell me about a time when..." вопросы. Часто Senior+ позиции имеют separate раунд.

**Common questions:**
- "Когда тебе пришлось делать unpopular decision?"
- "Конфликт в команде — как разрешил?"
- "Самый сложный bug который ты решал?"
- "Времена когда ты ошибся — что узнал?"
- "Mentor'инг junior'а — твой подход?"

**STAR template:**
```
Situation — context (1-2 sentence)
Task — что нужно было решить
Action — что ты сделал (главное!)
Result — что получилось, что узнал
```

**Подготовь 5-7 stories** покрывающих:
1. Технический challenge
2. Conflict resolution
3. Leadership / mentoring
4. Failure + lesson
5. Process improvement

См. [[10_interview-behavioral|Interview Behavioral]].

---

## 📋 За 1-2 недели до собеса
### Day 1-2: C# fundamentals review

- [[types-and-memory|types-and-memory]] — value vs reference, struct semantics, ref returns
- [[oop|oop]] — inheritance, interfaces, virtual/sealed/abstract
- [[collections-linq|collections-linq]] — когда какая коллекция, LINQ deep
- [[modern-features|modern-features]] — records, pattern matching, primary constructors

**Likely questions:**
- "Расскажи про разницу class vs struct vs record"
- "Когда String.Empty vs ''?"
- "Как работает `string.Intern()`?"
- "Что такое boxing/unboxing? Когда происходит?"
- "Equals и GetHashCode — контракт"
- "Когда используешь LINQ to Entities vs LINQ to Objects? В чём разница?"

### Day 3: Async / Concurrency

- [[async-threading|async-threading]] — Task, async/await internals, sync context
- [[concurrency-atomics|concurrency-atomics]] — locks, atomics, `Channel<T>`

**Likely questions:**
- "Как работает async/await под капотом?"
- "Что такое SynchronizationContext?"
- "ConfigureAwait(false) — когда нужно?"
- "Разница ValueTask vs Task — когда что?"
- "Deadlock в async коде — пример"
- "Channel vs BlockingCollection vs ConcurrentQueue"

### Day 4: ASP.NET Core

- [[pipeline-middleware|pipeline-middleware]] — middleware chain, exceptions handler
- [[di-configuration|di-configuration]] — Singleton/Scoped/Transient pitfalls
- [[auth-security|auth-security]] — JWT, OAuth, OIDC

**Likely questions:**
- "Как работает middleware pipeline?"
- "DI lifetimes — captive dependency как избежать"
- "JWT vs cookies — когда что?"
- "Refresh token — зачем и как"
- "RBAC vs ABAC vs PBAC"
- "Как реализовать rate limiting?"

### Day 5: EF Core / SQL

- [[basics-tracking|basics-tracking]] — Change Tracker, AsNoTracking
- [[queries-performance|queries-performance]] — N+1, Include vs Select
- [[optimization|SQL optimization]] — indexes, EXPLAIN

**Likely questions:**
- "Что такое N+1 проблема? Как её ловить?"
- "AsNoTracking — зачем"
- "Eager / Explicit / Lazy loading — когда что"
- "Что такое Cartesian explosion и AsSplitQuery"
- "Optimistic vs pessimistic concurrency"
- "Как написать индекс, чтобы query летал"
- "Что такое Index-only scan?"

### Day 6: Architecture

- [[solid|solid]] — принципы
- [[architecture-patterns|Architecture Patterns]] — Clean / Onion / Vertical Slices
- [[cqrs-mediatr|cqrs-mediatr]] — CQRS pattern (MediatR 13+ коммерческий — замены см. [[choosing-dependencies|Choosing Dependencies]])
- [[ddd|ddd]] — DDD concepts

**Likely questions:**
- "SOLID — приведи примеры из своего кода"
- "Clean / Onion / Hexagonal — в чём разница?"
- "Когда CQRS оправдан, когда — over-engineering"
- "Aggregate root — зачем"
- "Domain Event — пример"
- "Saga vs Process Manager"

### Day 7: Testing

- [[testing|testing]] — xUnit v3, integration, Testcontainers

**Likely questions:**
- "Unit vs Integration tests — когда какие"
- "Mocks vs stubs vs fakes"
- "Testcontainers — зачем"
- "Test pyramid — сколько каких"

### Day 8-10: System Design

- [[system-design|system-design]] — high-level design

**Practice tasks:**
1. Design URL shortener (TinyURL clone)
2. Design Twitter feed
3. Design rate limiter
4. Design distributed cache
5. Design notification system
6. Design payment processor

Для каждой:
- Functional requirements
- Non-functional (RPS, latency, availability)
- High-level design
- Database schema + индексы
- Caching strategy
- Scaling strategy
- Failure scenarios

### Day 11-12: Behavioral

- [[10_interview-behavioral|Interview Behavioral]] — STAR method, examples

---

## 🔥 Hot topics 2026 — обязательно знать

### Topic 1: .NET 10 LTS и миграция с 8/9

- Спросят: «на какой версии production и что с EOL?» .NET 10 — текущий LTS (ноябрь 2025, поддержка до ноября 2028); **.NET 8 и .NET 9 оба EOL 10.11.2026** — миграция в 2026 не опция, а дедлайн.
- Ответ строишь как процесс: bump TFM → пройтись по breaking changes → обновить пакеты → прогнать тесты; главный риск не в runtime, а в сторонних пакетах (см. Topic 7).

### Topic 2: C# 14

- Extension members (extension-блоки: свойства и static-члены, не только методы), `field` keyword (логика в accessor без ручного backing field), null-conditional assignment `obj?.Prop = value`.
- Спросят: «что нового в C# 14 и что из этого реально используешь» — отвечать на своих примерах, не перечислением. См. [[modern-features|modern-features]].

### Topic 3: HybridCache

- Единый API вместо связки `IMemoryCache` + `IDistributedCache`: L1 in-process + L2 (Redis), stampede protection из коробки, инвалидация по тегам.
- Спросят: «как защитить кэш от cache stampede» — раньше ждали ручной `SemaphoreSlim`-паттерн, теперь правильный ответ — HybridCache. См. [[caching|caching]].

### Topic 4: Встроенный OpenAPI + Scalar

- Swashbuckle убран из шаблонов: генерация документа — `Microsoft.AspNetCore.OpenApi` (`AddOpenApi()` / `MapOpenApi()`), UI из коробки нет — де-факто стандарт Scalar.
- Спросят: «чем заменили Swagger и почему» — уметь объяснить сам разлом (Swashbuckle стагнировал, Microsoft забрала генерацию документа в platform). См. [[api-design|api-design]].

### Topic 5: Aspire

- Оркестрация распределённого solution на dev-времени: AppHost на C#, service discovery, OpenTelemetry + dashboard из коробки. Не замена Kubernetes — это dev experience и deploy-генерация.
- Спросят: «как локально поднимаешь систему из N сервисов» — ответ уровня 2026: Aspire, а не рукописный docker-compose. См. [[aspire|Aspire]] (canonical), [[project-setup|project-setup]] и [[docker|docker]].

### Topic 6: xUnit v3 + Microsoft.Testing.Platform

- xUnit v3: тест-проект — самостоятельный executable, работает поверх Microsoft.Testing.Platform (MTP), который вытесняет VSTest; шаблоны .NET 10 генерят уже на MTP.
- Смежный вопрос: FluentAssertions с v8 тоже стал платным → Shouldly. См. [[testing|testing]].

### Topic 7: Лицензионный разлом OSS

- Частый вопрос 2026: «MediatR / AutoMapper / MassTransit стали коммерческими — что делать?» MediatR 13+ и AutoMapper 15+ — коммерческие (Lucky Penny, июль 2025; Community edition при выручке < $5M, версии ≤12.x / ≤14.x свободны); MassTransit v9 — коммерческий (Q1 2026), v8 — security-only до конца 2026.
- Ответ: свой in-process dispatcher или Mediator.SourceGenerator, Wolverine (messaging), Mapperly (mapping); главное — показать процесс выбора зависимости, а не заученный список. См. [[choosing-dependencies|Choosing Dependencies]].

### Topic 8: AI-слой в .NET

- Стандарт 2026 — не голый Semantic Kernel: абстракции `Microsoft.Extensions.AI` (`IChatClient`, embeddings) + Agent Framework (наследник SK + AutoGen) + MCP (официальный C# SDK) для инструментов.
- Спросят: «как подключить LLM в API и не прибить гвоздями к вендору» — DI поверх `IChatClient`, tool calling, structured output. См. [[llm-rag-patterns|llm-rag-patterns]] и [[mcp-csharp|mcp-csharp]].

### Topic 9: Vector search / pgvector

- RAG-вопрос: «где хранить embeddings» — базовый ответ: pgvector в уже имеющемся PostgreSQL (EF Core mapping, HNSW-индекс); отдельная vector DB — только под спец-требования.
- Уметь объяснить chunking → embedding → похожесть (cosine) → подмешивание в контекст. См. [[llm-rag-patterns|llm-rag-patterns]].

---

## ✅ Senior-level чек-лист

Для Senior позиций ожидается:

### Must know

- [ ] Все типы GC: generations, regions, DATAS, pinned, frozen
- [ ] `Span<T>`, `Memory<T>`, `ArrayPool<T>`, stackalloc
- [ ] async/await internals — state machine
- [ ] EF Core Change Tracker, identity map
- [ ] PostgreSQL: RLS, MVCC, EXPLAIN ANALYZE, indexes types
- [ ] Distributed transactions: Outbox, Saga, 2PC drawbacks
- [ ] CAP theorem implications
- [ ] Idempotency, eventual consistency
- [ ] OpenTelemetry полный stack
- [ ] Native AOT trade-offs

### Nice to have

- [ ] Roslyn API basics
- [ ] Source generators (свой написал)
- [ ] gRPC, GraphQL, SignalR
- [ ] Avalonia / MAUI
- [ ] F# basics (для understanding FP)
- [ ] Other languages: TypeScript, Kotlin, Go, Rust hello-world

### Architecture

- [ ] DDD aggregates
- [ ] CQRS + Event Sourcing tradeoffs
- [ ] Microservices vs Modular Monolith decision matrix
- [ ] Saga pattern implementations
- [ ] Async messaging (RabbitMQ / Kafka)

### Senior leadership

- [ ] ADRs — могу написать
- [ ] Code review — focus на design, не на nits
- [ ] Mentoring junior через pair programming
- [ ] Tech talks внутри команды

---

## 🎲 Coding tasks для practice

### Easy (junior-level warmup)

- FizzBuzz
- Reverse string / linked list
- Fibonacci
- Find duplicate in array
- Two sum

### Medium (middle-level)

- LRU cache implementation
- Rate limiter (token bucket / leaky bucket)
- Deep clone of arbitrary object graph
- Implement basic event bus
- Async timeout wrapper

### Hard (senior-level)

- Implement `Result<T, E>` with fluent API
- Write a simple ORM-like LINQ provider
- Build a Source Generator
- Optimize hot path с Span (real example)
- Design distributed lock с Redis

---

## 🎤 Live coding tips

- **Думай вслух** — interviewer хочет видеть процесс мышления, не finished solution
- **Уточняй requirements** — "Сколько пользователей? Concurrent? Latency требования?"
- **Сначала корректность, потом оптимизация** — "Сначала naive, потом will optimize"
- **Не молчи** — даже если думаешь, говори "Я сейчас рассматриваю варианты..."
- **Тесты в голове** — "Edge case: empty input, null, large data"

---

## 📞 За день до собеса

- [ ] Просмотреть [[09_senior-tips-cheatsheet|Senior Tips Cheatsheet]]
- [ ] Посмотреть company website / GitHub — какой stack используют
- [ ] Поспать достаточно
- [ ] Подготовить 3-5 вопросов **interviewer'у** (про команду, процесс, технологии)
- [ ] LinkedIn / резюме — освежить
- [ ] За 30 минут до — кофе, вода, тихое место

---

## 💬 Вопросы interviewer'у — обязательны

**В конце собеса всегда спрашивают "У тебя есть вопросы?". Подготовь:**

### Про команду

- "Сколько разработчиков в команде? Уровни?"
- "Как организована работа — Scrum / Kanban?"
- "Code review — сколько approvers?"

### Про технологии

- "Какие .NET версии в production? Когда обновляются?"
- "Microservices / monolith?"
- "CI/CD — что используется?"
- "Как с tech debt — есть ли время на refactoring?"

### Про культуру

- "Что больше всего мотивирует команду?"
- "Сложности с которыми сейчас сталкиваетесь?"
- "Возможности роста — куда могу развиваться через 1-2 года?"

### Про продукт

- "Как принимаются продуктовые решения?"
- "Метрики продукта — что важно?"
- "Roadmap на 6-12 месяцев — что в приоритете?"

---

## 🎯 Tips для собесов в FAANG / Big Tech

Если идёшь в Microsoft / Google / Amazon / Meta / Apple:

- **System Design rounds** — обязательны, готовься 2-4 недели
- **Behavioral / Leadership** — STAR method, examples из опыта
- **Coding rounds** — LeetCode hard уровень, обычно 2-3 раунда
- **Onsite** — 4-6 раундов в один день

Ресурсы:

- [Designing Data-Intensive Applications](https://dataintensive.net/) — book
- [System Design Interview](https://amzn.to/3hwLP0c) — Alex Xu's book
- [LeetCode](https://leetcode.com) — Top 75 Easy/Medium/Hard
- [Pramp / Interviewing.io](https://www.pramp.com) — practice mocks

---

## 🎯 Tips для собесов в startups

Startups часто ценят больше:

- **Practical experience** > academic знания
- **Pet-projects на GitHub** — must
- **Готовность wear multiple hats** — full-stack thinking
- **Communication** — small team, нужно объяснять решения

В резюме фокус на:

- Production opтыт (uptime, scale)
- Cross-functional работа
- Self-driven projects
- Open source contributions

---

См. также:

- [[09_senior-tips-cheatsheet|09 Senior Tips Cheatsheet]] — quick reference
- [[10_interview-behavioral|10 Interview Behavioral]] — soft skills
- [[99_reading-list|99 Reading List]] — books, blogs, courses
- [[02_junior-to-middle|02 Junior → Middle]] — если ещё не middle
- [[03_middle-to-senior|03 Middle → Senior]] — если хочешь Senior

---

## Cheat sheet

| Cheatsheet — Interview Quick Reference |
|---|
| **Algorithms** — O(n log n) sorts, hash table O(1), binary search O(log n) |
| **C# basics** — value vs reference types, string is reference but immutable |
| **GC generations** — Gen 0 (short-lived), Gen 1, Gen 2 (long-lived), LOH (>85KB) |
| **async/await** — Task is reference type, ValueTask for hot paths sync-completing |
| **Nullable types** — `int?` vs `string?` (NRT), null forgiveness `!` |
| **LINQ** — deferred execution, ToList materializes, AsEnumerable breaks IQueryable |
| **EF Core** — AsNoTracking for reads, Include for navigation, ProjectTo for DTO |
| **DI** — Singleton/Scoped/Transient lifetimes, captive dependency anti-pattern |
| **Threading** — Thread vs Task, ThreadPool, lock vs Monitor vs Semaphore |
| **Async pitfall** — `.Result` deadlock в UI/ASP.NET Classic; в ASP.NET Core дедлока нет, но sync-over-async всё равно вреден (thread pool starvation) |
| **Memory** — struct stack alloc (если local), class always heap, boxing cost |
| **HTTP** — REST verbs idempotency, status codes 4xx vs 5xx, CORS preflight |
| **SOLID** — SRP, OCP, LSP, ISP, DIP |
| **Patterns** — Strategy (interchangeable algo), Factory (creation), Observer (events), Decorator (wrap) |
| **CAP theorem** — Consistency/Availability/Partition tolerance, choose 2 |
| **ACID** — Atomic, Consistent, Isolated, Durable |
| **Indexes** — composite order matters, covering index, B-tree default |
| **N+1 problem** — Include или Projection в EF |
| **JWT structure** — Header.Payload.Signature, base64url encoded |
| **CORS** — browser-only, не security boundary |
| **Microservices vs Monolith** — distributed adds complexity, justify with team size |

| Common interview Q | Answer |
|---|---|
| Difference between Class and Struct | Reference vs value, heap vs stack, default semantics |
| What is Boxing | Wrapping value type into object reference (heap alloc) |
| async/await vs Task | async/await is syntactic sugar over Task continuations |
| ConfigureAwait(false) | In libraries — avoid context capture |
| `ref` vs `out` | ref must be initialized, out must be assigned in method |
| `using` statement | IDisposable pattern, deterministic cleanup |
| What is GC | Garbage Collector — auto memory management, generational |
| Singleton vs Static | Singleton testable (DI), Static — global state |
| Mocking — when | Tests need to isolate from external dependencies |
| Microservices — when | Big team, independent deployment, justifiable complexity |

---

## Decision tree

```
Подготовка к собесу?
│
├── Position level?
│   ├── Junior → Focus на basics: types, OOP, LINQ, EF basics
│   ├── Middle → + async, design patterns, REST, testing
│   └── Senior → + architecture, performance, distributed systems
│
├── Company type?
│   ├── Product → focus на architecture, system design
│   ├── Outsourcing → broad knowledge, multiple stacks
│   ├── Startup → pragmatic, full-stack thinking
│   └── Enterprise → patterns, processes, code quality
│
├── Interview type?
│   ├── Technical screen → algorithms, basic C# Q&A
│   ├── Live coding → practice на LeetCode (mid level)
│   ├── System design → study real-world architectures
│   ├── Behavioral → STAR method, prepare 5-7 stories
│   └── Take-home → quality > speed, tests obligatory
│
├── Topics to study (по priority):
│   1. C# fundamentals (types, async, LINQ, OOP)
│   2. .NET runtime (GC, JIT, threading)
│   3. EF Core (queries, tracking, performance)
│   4. ASP.NET Core (middleware, DI, auth)
│   5. SQL (joins, indexes, query optimization)
│   6. Architecture (SOLID, patterns, microservices)
│   7. Testing (unit, integration, mocking)
│   8. DevOps (Docker, CI/CD, k8s basics)
│
└── Practice:
    ├── LeetCode 50-100 medium problems
    ├── System design 5-10 case studies
    ├── Mock interviews с peers
    └── Read this vault top-20 files
```

