---
tags: [learning-path, topics, priority, value]
level: All
date: 2026-08-02
---

# 📊 Topics by Priority — что учить first

> Рейтинг тем по value vs effort. Если хочешь точечно прокачать — здесь приоритеты.

---

## ⭐⭐⭐ Critical — must know

Без этих знаний middle/senior быть не может. Учить **в первую очередь**.

| # | Тема | Файл | Effort | ROI |
|---|------|------|--------|-----|
| 1 | Async/await deep | [[async-threading\|async-threading]] | High | ⭐⭐⭐⭐⭐ |
| 2 | EF Core Basics + Tracking | [[basics-tracking\|basics-tracking]] | Medium | ⭐⭐⭐⭐⭐ |
| 3 | Queries Performance (N+1) | [[queries-performance\|queries-performance]] | Medium | ⭐⭐⭐⭐⭐ |
| 4 | DI lifetimes pitfalls | [[di-configuration\|di-configuration]] | Low | ⭐⭐⭐⭐⭐ |
| 5 | Pipeline & Middleware | [[pipeline-middleware\|pipeline-middleware]] | Medium | ⭐⭐⭐⭐⭐ |
| 6 | Authentication / JWT | [[auth-security\|auth-security]] | High | ⭐⭐⭐⭐⭐ |
| 7 | SOLID principles | [[solid\|solid]] | Low | ⭐⭐⭐⭐⭐ |
| 8 | Architecture Patterns | [[architecture-patterns\|architecture-patterns]] | High | ⭐⭐⭐⭐⭐ |
| 9 | SQL Optimization | [[optimization\|optimization]] | Medium | ⭐⭐⭐⭐⭐ |
| 10 | Testing | [[testing\|testing]] | Medium | ⭐⭐⭐⭐⭐ |

**Время на освоение critical:** ~40-60 часов чистого reading + practice.

---

## ⭐⭐ High value — учить во вторую очередь

Не критично сразу, но **отделяет middle от senior**.

| # | Тема | Файл | Effort | ROI |
|---|------|------|--------|-----|
| 11 | GC и память | [[gc-memory\|gc-memory]] | High | ⭐⭐⭐⭐ |
| 12 | `Span<T>` и Layout | [[span-layout\|span-layout]] | High | ⭐⭐⭐⭐ |
| 13 | Concurrency atomics | [[concurrency-atomics\|concurrency-atomics]] | High | ⭐⭐⭐⭐ |
| 14 | Modern C# features | [[modern-features\|modern-features]] | Medium | ⭐⭐⭐⭐ |
| 15 | Functional C# | [[functional-csharp\|functional-csharp]] | Medium | ⭐⭐⭐⭐ |
| 16 | DDD | [[ddd\|ddd]] | High | ⭐⭐⭐⭐ |
| 17 | CQRS / mediator-паттерн (сам паттерн знать обязательно; MediatR 13+ коммерческий — см. [[choosing-dependencies\|Choosing Dependencies]]) | [[cqrs-mediatr\|cqrs-mediatr]] | Medium | ⭐⭐⭐⭐ |
| 18 | Distributed Systems | [[distributed-systems\|distributed-systems]] | High | ⭐⭐⭐⭐ |
| 19 | EF Migrations | [[migrations\|migrations]] | Medium | ⭐⭐⭐⭐ |
| 20 | EF Patterns | [[ef-patterns\|ef-patterns]] | High | ⭐⭐⭐⭐ |
| 21 | Caching | [[caching\|caching]] | Medium | ⭐⭐⭐⭐ |
| 22 | Resilience (Polly) | [[resilience\|resilience]] | Low | ⭐⭐⭐⭐ |
| 23 | Logging Observability | [[logging-observability\|logging-observability]] | Medium | ⭐⭐⭐⭐ |
| 24 | OpenTelemetry | [[observability\|observability]] | High | ⭐⭐⭐⭐ |
| 25 | Docker | [[docker\|docker]] | Medium | ⭐⭐⭐⭐ |
| 26 | Project Setup | [[project-setup\|project-setup]] | Low | ⭐⭐⭐⭐ |
| 27 | PostgreSQL Deep | [[postgresql-deep\|postgresql-deep]] | High | ⭐⭐⭐⭐ |
| 28 | Performance | [[performance\|performance]] | High | ⭐⭐⭐⭐ |
| 29 | Diagnostics tools | [[diagnostics-tools\|diagnostics-tools]] | Medium | ⭐⭐⭐⭐ |
| 29a | ThreadPool Starvation / Hill-Climbing | [[threadpool-starvation-hill-climbing\|threadpool-starvation-hill-climbing]] | High | ⭐⭐⭐⭐ |
| 29b | Agent-Safe Architecture | [[agent-safe-architecture\|agent-safe-architecture]] | Medium | ⭐⭐⭐⭐ |
| 29c | Stack vs Heap | [[memory-stack-heap\|memory-stack-heap]] | Low | ⭐⭐⭐⭐ |
| 29d | EF Value Converters | [[ef-value-converters\|ef-value-converters]] | Low | ⭐⭐⭐⭐ |
| 29e | Lazy vs Eager Loading | [[lazy-eager-loading\|lazy-eager-loading]] | Low | ⭐⭐⭐⭐ |
| 29f | HttpClient Resilience | [[http-client-resilience\|http-client-resilience]] | Low | ⭐⭐⭐⭐ |
| 29g | Rate Limiting (ASP.NET) | [[aspnet-rate-limiting\|aspnet-rate-limiting]] | Low | ⭐⭐⭐⭐ |
| 29h | Performance Budgets | [[performance-budgets\|performance-budgets]] | Medium | ⭐⭐⭐⭐ |
| 29i | SQL Security | [[sql-security\|sql-security]] | Medium | ⭐⭐⭐⭐ |

---

## ⭐ Medium value — для глубокого Senior

Учить когда **уже Senior** или для специфичных задач.

| # | Тема | Файл | Когда |
|---|------|------|-------|
| 30 | Reflection & Expression Trees | [[reflection-expression-trees\|reflection-expression-trees]] | Lib authors, ORM internals |
| 31 | Source Generators | [[source-generators\|source-generators]] | AOT-friendly libs |
| 32 | Compilation / JIT | [[compilation-jit\|compilation-jit]] | Performance work |
| 33 | Interop / P/Invoke | [[interop-pinvoke\|interop-pinvoke]] | Native libs, MetaTrader |
| 34 | Native AOT | [[native-aot\|native-aot]] | CLI tools, microservices |
| 35 | GraphQL | [[graphql\|graphql]] | Сложные queries / BFF |
| 36 | SignalR | [[signalr\|signalr]] | Real-time features |
| 37 | gRPC / IPC | [[ipc-named-pipes-grpc\|ipc-named-pipes-grpc]] | Microservice-to-microservice |
| 38 | Messaging | [[messaging\|messaging]] | Async architecture |
| 39 | Code Quality | [[code-quality\|code-quality]] | Tech leads |
| 40 | Architecture Tests | [[arch-tests\|arch-tests]] | Большие проекты |
| 41 | ADRs | [[architecture-decisions\|architecture-decisions]] | Tech leads |
| 42 | System Design | [[system-design\|system-design]] | FAANG interviews |
| 43 | EIP: Content-Based Router | [[eip-content-based-router\|eip-content-based-router]] | Integration / messaging pipelines |
| 44 | Fenwick Tree / BIT | [[fenwick-bit\|fenwick-bit]] | Leaderboards, running aggregates |
| 45 | Kestrel как raw HTTP host | [[kestrel-as-raw-host\|kestrel-as-raw-host]] | Framework / gateway / proxy authors |

---

## 🌐 Specialized — для специфических ниш

Только если работаешь в этой нише.

| Тема | Когда |
|------|-------|
| [[hft-low-latency\|HFT / Low Latency]] | Trading, real-time analytics |
| [[wpf-production\|WPF Production]] | Desktop development legacy |
| [[desktop-frameworks\|Desktop Frameworks]] | Cross-platform desktop |
| [[cli-tools-scripting\|CLI Tools]] | DevTools, internal automation |
| [[blazor-wasm\|Blazor WASM]] | Frontend на C# |
| [[blazor-server\|Blazor Server]] | Internal apps |
| [[llm-rag-patterns\|LLM/RAG]] | AI integration |
| [[semantic-kernel\|Semantic Kernel]] | Microsoft AI stack |
| [[webai-csharp-architecture\|Web/AI Architecture]] | AI-powered apps |

---

## 📚 Mostly cultural / philosophical

Не делает тебя лучшим coder, но расширяет кругозор.

| Тема | Польза |
|------|--------|
| [[csharp-language-design\|C# Language Design]] | Понимать "почему так" |
| [[csharp-vs-other-langs\|C# vs Other Languages]] | Polyglot mindset |
| [[error-handling\|Error Handling]] | Patterns for errors |
| [[oop\|OOP]] | Деep OOP knowledge |
| [[types-and-memory\|Types & Memory]] | Foundation |
| [[collections-linq\|Collections & LINQ]] | Daily tools |
| [[delegates-events\|Delegates & Events]] | Языковые механики |
| [[design-patterns\|Design Patterns]] | GoF классика |

---

## 🎯 Прохождение по приоритетам

### Если у меня 1 неделя

Топ-5 critical:

1. [[async-threading|async-threading]]
2. [[basics-tracking|basics-tracking]]
3. [[queries-performance|queries-performance]]
4. [[di-configuration|di-configuration]]
5. [[architecture-patterns|Architecture Patterns]]

### Если у меня месяц

Все 10 critical (раздел ⭐⭐⭐).

### Если у меня 3 месяца

Critical + High value (29 тем).

### Если у меня 6 месяцев

Critical + High + Medium = почти весь vault.

---

## 🔥 Часто упускаемые но важные

Темы которые **многие пропускают**, но которые **бьют по производительности**:

1. **EF Change Tracker** — почему DetectChanges медленный, как избежать
2. **GC generations behavior** — почему long-lived объекты в Gen2
3. **`Span<T>`** для строк — zero-allocation parsing
4. **ConfigureAwait** — когда нужен в библиотеках
5. **DI captive dependency** — Scoped в Singleton = leak
6. **PostgreSQL EXPLAIN** — чтение plans
7. **Idempotency** — must для distributed
8. **Outbox pattern** — at-least-once messaging
9. **Cancellation propagation** — каждый await должен быть cancellable
10. **Native AOT trade-offs** — что ломается

Если эти 10 тем понимаешь — уже **strong middle / weak senior**.

---

## 📈 Self-tracking — где я сейчас

Создай свой checklist:

```
Critical (10):     [▓▓▓▓░░░░░░] 4/10 пройдено
High value (19):   [▓▓░░░░░░░░] 4/19
Medium (13):       [░░░░░░░░░░] 0/13
Specialized (9):   [░░░░░░░░░░] 0/9
Cultural (8):      [▓░░░░░░░░░] 1/8
```

**Tracking template:**

```
- [ ] CSharp/async-threading
- [ ] EFCore/basics-tracking
- [ ] EFCore/queries-performance
...
```

Скопируй в свой Daily Note Obsidian, обновляй progress.

---

## 🚦 Готовность к роли

### Junior

- ✅ Critical 7+/10 (можешь писать CRUD)

### Middle

- ✅ Critical 10/10
- ✅ High value 12+/19
- 🎯 Pet-project с production patterns

### Senior

- ✅ Critical 10/10
- ✅ High value 19/19
- ✅ Medium 8+/13
- 🎯 Practical experience с production incidents

### Senior+

- ✅ All Critical + High + Medium
- ✅ 2+ Specialized (по нише компании)
- 🎯 Менторил juniors, делал tech talks
- 🎯 Архитектурные решения за плечами
