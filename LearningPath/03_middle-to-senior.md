---
tags: [learning-path, middle, senior, roadmap, advanced]
level: Middle to Senior
date: 2026-04-30
---

# 🚀 Middle → Senior .NET

> Roadmap для Senior-уровня. Senior — не "знает всё", а **думает системно** и **принимает trade-offs**. Этот путь идёт глубже в каждую тему, добавляет архитектурное мышление, performance, production patterns.

---

## 🎯 Что отличает Senior от Middle

| | Middle | Senior |
|--|--------|--------|
| Знание language | Использует | **Понимает почему** |
| Архитектура | Применяет паттерны | **Выбирает между паттернами** |
| Performance | Знает что важно | **Профилирует и оптимизирует** |
| Production issues | Гасит огни | **Предотвращает** |
| Code review | Замечает баги | **Замечает design issues** |
| Mentoring | — | **Помогает команде расти** |
| Business | Пишет код | **Понимает зачем** |

---

## 📅 Time estimate

| Уровень фокуса | Длительность |
|----------------|--------------|
| Идти параллельно с работой | 8-12 месяцев |
| Active investment (вечера + выходные) | 4-6 месяцев |
| Фокус на росте 100% | 2-3 месяца |

Senior не достигается просто чтением — нужен **production experience с complex systems**.

---

## Phase 1 — Deep CLR / Runtime (3-4 недели)

**Цель:** понимать что происходит под капотом.

| # | Тема | Файл | Время |
|---|------|------|-------|
| 1 | GC и память | [[gc-memory\|gc-memory]] | 8-10 ч |
| 1a | Stack vs Heap | [[memory-stack-heap\|memory-stack-heap]] | 2-3 ч |
| 2 | Compilation / JIT | [[compilation-jit\|compilation-jit]] | 4-5 ч |
| 3 | Span и Layout | [[span-layout\|span-layout]] | 6-7 ч |
| 4 | Concurrency atomics | [[concurrency-atomics\|concurrency-atomics]] | 6-7 ч |
| 5 | Diagnostics tools | [[diagnostics-tools\|diagnostics-tools]] | 5-6 ч |
| 6 | Interop / P/Invoke | [[interop-pinvoke\|interop-pinvoke]] | 4-5 ч |
| 7 | Reflection & Expression Trees | [[reflection-expression-trees\|reflection-expression-trees]] | 6-7 ч |
| 8 | Source Generators | [[source-generators\|source-generators]] | 4-5 ч |

### Practice

- Запустить `dotnet-counters` на pet-project в production
- Найти memory leak с `dotnet-gcdump` (умышленно создать)
- Написать BenchmarkDotNet test для критичной функции
- Span<T> рефакторинг строк parsing
- Свой Source Generator (простой mapper)

### Чек-лист

- [ ] Понимаю generations / regions / DATAS GC
- [ ] Знаю когда struct vs class (allocation, copy cost)
- [ ] Span/Memory/ArrayPool применяю в hot paths
- [ ] dotnet-trace и анализ flame graph
- [ ] Reflection cached + Compiled Expression
- [ ] AOT-friendly код пишу

---

## Phase 2 — Functional C# и language design (2-3 недели)

**Цель:** modern multi-paradigm C#.

| # | Тема | Файл | Время |
|---|------|------|-------|
| 9 | Functional C# | [[functional-csharp\|functional-csharp]] | 5-6 ч |
| 10 | C# Language Design | [[csharp-language-design\|csharp-language-design]] | 3-4 ч |
| 11 | C# vs other languages | [[csharp-vs-other-langs\|csharp-vs-other-langs]] | 3-4 ч |

### Practice

- Применить Result<T,E> везде в pet-project
- Заменить exceptions на разные viable errors
- Pattern matching для разных типов входных данных
- Один pet-feature на F# (если интересно)

---

## Phase 3 — Architecture deep (4-5 недель)

**Цель:** проектировать целые системы.

| # | Тема | Файл | Время |
|---|------|------|-------|
| 12 | Architecture Patterns deep | [[architecture-patterns\|architecture-patterns]] | 8-10 ч |
| 13 | DDD на практике | [[ddd\|ddd]] | 7-8 ч |
| 14 | CQRS + MediatR глубоко | [[cqrs-mediatr\|cqrs-mediatr]] | 5-6 ч |
| 15 | Distributed Systems | [[distributed-systems\|distributed-systems]] | 8-10 ч |
| 16 | System Design | [[system-design\|system-design]] | 6-8 ч |
| 17 | Architecture Tests | [[arch-tests\|arch-tests]] | 2-3 ч |
| 18 | ADRs | [[architecture-decisions\|architecture-decisions]] | 3-4 ч |
| 18a | Agent-Safe Architecture | [[agent-safe-architecture\|agent-safe-architecture]] | 3-4 ч |
| 18b | EIP: Content-Based Router | [[eip-content-based-router\|eip-content-based-router]] | 2-3 ч |
| 19 | EF Patterns | [[ef-patterns\|ef-patterns]] | 5-6 ч |
| 19a | EF Value Converters | [[ef-value-converters\|ef-value-converters]] | 2-3 ч |
| 19b | Lazy vs Eager Loading | [[lazy-eager-loading\|lazy-eager-loading]] | 2-3 ч |

### Practice

- Перестроить pet-project в Modular Monolith с 2-3 модулями
- Outbox pattern для domain events
- Saga для multi-step business process (e.g. checkout)
- Написать 5 ADR для решений принятых ранее
- NetArchTest для архитектурных правил

### Чек-лист

- [ ] DDD aggregates с invariants и domain events
- [ ] Outbox pattern для at-least-once delivery
- [ ] Знаю когда CQRS vs simpler pattern
- [ ] Eventual consistency понятна и принята
- [ ] Saga / Process manager для distributed transactions
- [ ] System design на 1-100 RPS / 1k-10k RPS / 100k+ RPS

---

## Phase 4 — Modern API styles (2-3 недели)

**Цель:** не только REST.

| # | Тема | Файл | Время |
|---|------|------|-------|
| 20 | GraphQL | [[graphql\|graphql]] | 5-6 ч |
| 21 | SignalR | [[signalr\|signalr]] | 5-6 ч |
| 22 | gRPC / IPC | [[ipc-named-pipes-grpc\|ipc-named-pipes-grpc]] | 4-5 ч |
| 22a | Kestrel как raw HTTP host | [[kestrel-as-raw-host\|kestrel-as-raw-host]] | 3-4 ч |
| 23 | Native AOT | [[native-aot\|native-aot]] | 3-4 ч |
| 24 | Blazor WASM | [[blazor-wasm\|blazor-wasm]] | 4-5 ч |
| 25 | Blazor Server | [[blazor-server\|blazor-server]] | 3-4 ч |

### Practice

- Один из API styles в pet-project (GraphQL или gRPC)
- Real-time updates через SignalR
- Native AOT version проекта (если возможно)

---

## Phase 5 — Production patterns (3-4 недели)

**Цель:** что нужно для production-ready.

| # | Тема | Файл | Время |
|---|------|------|-------|
| 26 | Observability deep | [[observability\|observability]] | 5-6 ч |
| 27 | Messaging | [[messaging\|messaging]] | 5-6 ч |
| 28 | Performance | [[performance\|performance]] | 5-6 ч |
| 28a | ThreadPool Starvation / Hill-Climbing | [[threadpool-starvation-hill-climbing\|threadpool-starvation-hill-climbing]] | 3-4 ч |
| 28b | Performance Budgets | [[performance-budgets\|performance-budgets]] | 2-3 ч |
| 28c | HttpClient Resilience | [[http-client-resilience\|http-client-resilience]] | 2-3 ч |
| 28d | Fenwick Tree / BIT | [[fenwick-bit\|fenwick-bit]] | 2-3 ч |
| 29 | HFT / Low Latency | [[hft-low-latency\|hft-low-latency]] | 4-5 ч |
| 30 | Docker production | [[docker\|docker]] | 5-6 ч |
| 31 | LLM/RAG patterns | [[llm-rag-patterns\|llm-rag-patterns]] | 5-6 ч |
| 32 | Security practices | [[security-practices\|security-practices]] | 4-5 ч |
| 32a | Rate Limiting (ASP.NET) | [[aspnet-rate-limiting\|aspnet-rate-limiting]] | 2-3 ч |
| 32b | SQL Security | [[sql-security\|sql-security]] | 2-3 ч |
| 33 | Testing deep | [[testing\|testing]] | 5-6 ч |

### Practice

- OpenTelemetry в pet-project + Jaeger UI
- RabbitMQ / Kafka для async messaging
- BenchmarkDotNet для critical paths
- Profile production traffic с Pyroscope
- LLM integration через Semantic Kernel

### Чек-лист

- [ ] Distributed tracing работает end-to-end
- [ ] Metrics: SLO/SLI/SLA понятие применяю
- [ ] Performance regression detection
- [ ] Container security (non-root user, distroless)
- [ ] Vault / Key management

---

## Phase 6 — Cross-cutting (2-3 недели)

**Цель:** темы которые везде.

| # | Тема | Файл | Время |
|---|------|------|-------|
| 34 | CLI tools / scripting | [[cli-tools-scripting\|cli-tools-scripting]] | 3-4 ч |
| 35 | Desktop frameworks | [[desktop-frameworks\|desktop-frameworks]] | 3-4 ч |
| 36 | WPF Production (если применимо) | [[wpf-production\|wpf-production]] | 4-5 ч |
| 37 | Web/AI architecture | [[webai-csharp-architecture\|webai-csharp-architecture]] | 3-4 ч |
| 38 | Semantic Kernel | [[semantic-kernel\|semantic-kernel]] | 2-3 ч |

---

## 🎯 Senior pet-project — должно быть

К моменту "я Senior" pet-project должен показывать:

1. ✅ Modular Monolith с 3+ bounded contexts
2. ✅ DDD aggregates с domain events
3. ✅ CQRS с separate read models
4. ✅ Outbox pattern для events
5. ✅ At least one Saga/Process Manager
6. ✅ EF Core с complex relationships, value converters
7. ✅ PostgreSQL с RLS / advanced features
8. ✅ Distributed tracing (OTel + Jaeger)
9. ✅ Structured logging с correlation IDs
10. ✅ Circuit breaker + retry + timeout (Polly)
11. ✅ Rate limiting (распределённый, не in-memory)
12. ✅ Authentication через external IdP (Auth0/Keycloak)
13. ✅ RBAC + ABAC где сложнее
14. ✅ Background jobs (Hangfire / Quartz)
15. ✅ Real-time через SignalR
16. ✅ GraphQL **или** gRPC API
17. ✅ Native AOT version (для CLI tools минимум)
18. ✅ Tests: unit / integration / architecture / load
19. ✅ Docker с multi-stage builds
20. ✅ k8s deployment (Helm / Kustomize)
21. ✅ CI/CD: tests + lint + Docker push + deploy
22. ✅ ADRs для каждого major decision

---

## 🧠 Senior soft-skills

Технические знания недостаточно. Senior это про:

### 1. System thinking

- Видеть систему целиком, а не один service
- Понимать blast radius изменений
- Cross-team dependencies — proactively manage

### 2. Communication

- Объяснять сложное **простыми словами** (newcomer на проект)
- Писать **clear ADRs** и design docs
- Code review с фокусом на **growth**, не на "why didn't you do X"
- Disagree and commit

### 3. Trade-off thinking

- "Идеального решения нет" — выбирай trade-offs **сознательно**
- Document **почему** выбрал, не только **что**
- Reversible vs irreversible decisions

### 4. Business awareness

- Зачем фича — какой problem solve'им?
- Cost / benefit
- Time to market vs technical debt

### 5. Mentoring

- Ставить growth-oriented задачи juniors
- Pair programming с коллегами
- "Teach to fish, not give fish"

---

## 📊 Self-assessment — ты Senior если

- [ ] Можешь объяснить любой пункт из vault — **сходу, без подсказок**
- [ ] Прошёл production incident от alert до root cause analysis
- [ ] Спроектировал систему с >5 services
- [ ] Делал technical interview как interviewer
- [ ] Менторил минимум 1 junior до middle
- [ ] Презентовал tech talk внутри команды или externally
- [ ] Написал минимум 5 ADRs которые приняты командой
- [ ] Видел EF / ASP.NET / .NET через 2+ major versions (e.g. 6→8 changes)
- [ ] Можешь обсудить design choices с trade-offs (не "лучшее решение")

---

## 📚 Senior books — must-read

См. [[99_reading-list\|99 Reading List]] — full list.

Топ-5 для Senior:

1. **Designing Data-Intensive Applications** — Martin Kleppmann (must!)
2. **Domain-Driven Design** — Eric Evans (классика)
3. **Implementing Domain-Driven Design** — Vaughn Vernon (более практичная)
4. **Building Microservices** — Sam Newman
5. **Designing Distributed Systems** — Brendan Burns

Plus:

6. **Pro .NET Memory Management** — Konrad Kokosa (если deep CLR)
7. **CLR via C#** — Jeffrey Richter (классика)
8. **Performance** — Stephen Cleary blog + Stephen Toub posts

---

## 🎤 Conferences / talks

- **NDC Conferences** (Oslo, London) — best .NET community
- **dotnetConf** — Microsoft official
- **GOTO** conferences — broader software design
- **InfoQ** — articles + videos
- **Stephen Toub posts** — performance series
- **Andrew Lock blog** — ASP.NET deep
- **Jon Skeet blog** — language deep

См. [[99_reading-list\|99 Reading List]] для полного списка.

---

## ⏭️ После Senior — Senior+, Staff, Principal

Senior — не финал. Дальше:

- **Senior+ / Tech Lead** — тиражирование impact в команде
- **Staff Engineer** — cross-team, technical strategy
- **Principal** — organization-wide, technical vision

Эти уровни **не про код**, а про:
- Strategy
- Influence без authority
- Cross-functional partnership
- Technology bets and roadmaps
