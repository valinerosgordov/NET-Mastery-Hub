---
tags: [learning-path, interview, prep, checklist]
level: All
date: 2026-04-30
---

# 🎤 Interview Prep

> Подготовка к техническому собеседованию на .NET Backend позиции. От Junior до Senior. Что повторить, какие задачи решать, на чём фокус.

---

## 📋 За 1-2 недели до собеса

### Day 1-2: C# fundamentals review

- [[CSharp/types-and-memory|types-and-memory]] — value vs reference, struct semantics, ref returns
- [[CSharp/oop|oop]] — inheritance, interfaces, virtual/sealed/abstract
- [[CSharp/collections-linq|collections-linq]] — когда какая коллекция, LINQ deep
- [[CSharp/modern-features|modern-features]] — records, pattern matching, primary constructors

**Likely questions:**
- "Расскажи про разницу class vs struct vs record"
- "Когда String.Empty vs ''?"
- "Как работает `string.Intern()`?"
- "Что такое boxing/unboxing? Когда происходит?"
- "Equals и GetHashCode — контракт"
- "Когда используешь LINQ to Entities vs LINQ to Objects? В чём разница?"

### Day 3: Async / Concurrency

- [[CSharp/async-threading|async-threading]] — Task, async/await internals, sync context
- [[Runtime/concurrency-atomics|concurrency-atomics]] — locks, atomics, Channel<T>

**Likely questions:**
- "Как работает async/await под капотом?"
- "Что такое SynchronizationContext?"
- "ConfigureAwait(false) — когда нужно?"
- "Разница ValueTask vs Task — когда что?"
- "Deadlock в async коде — пример"
- "Channel vs BlockingCollection vs ConcurrentQueue"

### Day 4: ASP.NET Core

- [[AspNetCore/pipeline-middleware|pipeline-middleware]] — middleware chain, exceptions handler
- [[AspNetCore/di-configuration|di-configuration]] — Singleton/Scoped/Transient pitfalls
- [[AspNetCore/auth-security|auth-security]] — JWT, OAuth, OIDC

**Likely questions:**
- "Как работает middleware pipeline?"
- "DI lifetimes — captive dependency как избежать"
- "JWT vs cookies — когда что?"
- "Refresh token — зачем и как"
- "RBAC vs ABAC vs PBAC"
- "Как реализовать rate limiting?"

### Day 5: EF Core / SQL

- [[EFCore/basics-tracking|basics-tracking]] — Change Tracker, AsNoTracking
- [[EFCore/queries-performance|queries-performance]] — N+1, Include vs Select
- [[SQL/optimization|SQL optimization]] — indexes, EXPLAIN

**Likely questions:**
- "Что такое N+1 проблема? Как её ловить?"
- "AsNoTracking — зачем"
- "Eager / Explicit / Lazy loading — когда что"
- "Что такое Cartesian explosion и AsSplitQuery"
- "Optimistic vs pessimistic concurrency"
- "Как написать индекс, чтобы query летал"
- "Что такое Index-only scan?"

### Day 6: Architecture

- [[Architecture/solid|solid]] — принципы
- [[Architecture/patterns|patterns]] — Clean / Onion / Vertical Slices
- [[Architecture/cqrs-mediatr|cqrs-mediatr]] — CQRS pattern
- [[Architecture/ddd|ddd]] — DDD concepts

**Likely questions:**
- "SOLID — приведи примеры из своего кода"
- "Clean / Onion / Hexagonal — в чём разница?"
- "Когда CQRS оправдан, когда — over-engineering"
- "Aggregate root — зачем"
- "Domain Event — пример"
- "Saga vs Process Manager"

### Day 7: Testing

- [[Testing/testing|testing]] — xUnit, integration, Testcontainers

**Likely questions:**
- "Unit vs Integration tests — когда какие"
- "Mocks vs stubs vs fakes"
- "Testcontainers — зачем"
- "Test pyramid — сколько каких"

### Day 8-10: System Design

- [[Architecture/system-design|system-design]] — high-level design

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

### Topic 1: Cloud-native .NET

- Native AOT — кратко зачем
- Containerization — Docker best practices
- Kubernetes basics — Pod / Service / Deployment
- Health checks — Liveness vs Readiness
- 12-factor app principles

### Topic 2: Modern C#

- Records (C# 9) — что и зачем
- Pattern matching evolution (8→13)
- Primary constructors (C# 12)
- Collection expressions (C# 12)
- Nullable reference types
- Source generators basics

### Topic 3: Modern ASP.NET

- Minimal APIs vs MVC
- Endpoint routing
- Output Caching (.NET 7+)
- Rate Limiting middleware (.NET 7+)
- IExceptionHandler (.NET 8+)
- Authentication через external IdP

### Topic 4: Distributed systems

- Eventual consistency
- Saga vs distributed transactions
- Outbox pattern
- Idempotency keys
- Circuit breaker / Retry / Timeout
- Service mesh basics (Istio / Linkerd)

### Topic 5: Observability

- OpenTelemetry — основа
- Distributed tracing (TraceId / SpanId)
- Structured logging (`logger.LogInfo("{X}", x)`)
- Metrics: Counter / Gauge / Histogram
- SLI / SLO / SLA

### Topic 6: AI Integration

- LLM API patterns (function calling, tool use)
- RAG (vector DB + embeddings)
- Semantic Kernel basics
- ML.NET для inference

---

## ✅ Senior-level чек-лист

Для Senior позиций ожидается:

### Must know

- [ ] Все типы GC: generations, regions, DATAS, pinned, frozen
- [ ] Span<T>, Memory<T>, ArrayPool<T>, stackalloc
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

- Implement Result<T, E> with fluent API
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
