---
tags: [learning-path, topics, priority, value]
level: All
date: 2026-04-30
---

# 📊 Topics by Priority — что учить first

> Рейтинг тем по value vs effort. Если хочешь точечно прокачать — здесь приоритеты.

---

## ⭐⭐⭐ Critical — must know

Без этих знаний middle/senior быть не может. Учить **в первую очередь**.

| # | Тема | Файл | Effort | ROI |
|---|------|------|--------|-----|
| 1 | Async/await deep | [[CSharp/async-threading\|async-threading]] | High | ⭐⭐⭐⭐⭐ |
| 2 | EF Core Basics + Tracking | [[EFCore/basics-tracking\|basics-tracking]] | Medium | ⭐⭐⭐⭐⭐ |
| 3 | Queries Performance (N+1) | [[EFCore/queries-performance\|queries-performance]] | Medium | ⭐⭐⭐⭐⭐ |
| 4 | DI lifetimes pitfalls | [[AspNetCore/di-configuration\|di-configuration]] | Low | ⭐⭐⭐⭐⭐ |
| 5 | Pipeline & Middleware | [[AspNetCore/pipeline-middleware\|pipeline-middleware]] | Medium | ⭐⭐⭐⭐⭐ |
| 6 | Authentication / JWT | [[AspNetCore/auth-security\|auth-security]] | High | ⭐⭐⭐⭐⭐ |
| 7 | SOLID principles | [[Architecture/solid\|solid]] | Low | ⭐⭐⭐⭐⭐ |
| 8 | Architecture Patterns | [[Architecture/patterns\|patterns]] | High | ⭐⭐⭐⭐⭐ |
| 9 | SQL Optimization | [[SQL/optimization\|optimization]] | Medium | ⭐⭐⭐⭐⭐ |
| 10 | Testing | [[Testing/testing\|testing]] | Medium | ⭐⭐⭐⭐⭐ |

**Время на освоение critical:** ~40-60 часов чистого reading + practice.

---

## ⭐⭐ High value — учить во вторую очередь

Не критично сразу, но **отделяет middle от senior**.

| # | Тема | Файл | Effort | ROI |
|---|------|------|--------|-----|
| 11 | GC и память | [[Runtime/gc-memory\|gc-memory]] | High | ⭐⭐⭐⭐ |
| 12 | Span<T> и Layout | [[Runtime/span-layout\|span-layout]] | High | ⭐⭐⭐⭐ |
| 13 | Concurrency atomics | [[Runtime/concurrency-atomics\|concurrency-atomics]] | High | ⭐⭐⭐⭐ |
| 14 | Modern C# features | [[CSharp/modern-features\|modern-features]] | Medium | ⭐⭐⭐⭐ |
| 15 | Functional C# | [[CSharp/functional-csharp\|functional-csharp]] | Medium | ⭐⭐⭐⭐ |
| 16 | DDD | [[Architecture/ddd\|ddd]] | High | ⭐⭐⭐⭐ |
| 17 | CQRS + MediatR | [[Architecture/cqrs-mediatr\|cqrs-mediatr]] | Medium | ⭐⭐⭐⭐ |
| 18 | Distributed Systems | [[Architecture/distributed-systems\|distributed-systems]] | High | ⭐⭐⭐⭐ |
| 19 | EF Migrations | [[EFCore/migrations\|migrations]] | Medium | ⭐⭐⭐⭐ |
| 20 | EF Patterns | [[EFCore/patterns\|patterns]] | High | ⭐⭐⭐⭐ |
| 21 | Caching | [[AspNetCore/caching\|caching]] | Medium | ⭐⭐⭐⭐ |
| 22 | Resilience (Polly) | [[AspNetCore/resilience\|resilience]] | Low | ⭐⭐⭐⭐ |
| 23 | Logging Observability | [[AspNetCore/logging-observability\|logging-observability]] | Medium | ⭐⭐⭐⭐ |
| 24 | OpenTelemetry | [[Infrastructure/observability\|observability]] | High | ⭐⭐⭐⭐ |
| 25 | Docker | [[Infrastructure/docker\|docker]] | Medium | ⭐⭐⭐⭐ |
| 26 | Project Setup | [[Infrastructure/project-setup\|project-setup]] | Low | ⭐⭐⭐⭐ |
| 27 | PostgreSQL Deep | [[SQL/postgresql-deep\|postgresql-deep]] | High | ⭐⭐⭐⭐ |
| 28 | Performance | [[Performance/performance\|performance]] | High | ⭐⭐⭐⭐ |
| 29 | Diagnostics tools | [[Runtime/diagnostics-tools\|diagnostics-tools]] | Medium | ⭐⭐⭐⭐ |

---

## ⭐ Medium value — для глубокого Senior

Учить когда **уже Senior** или для специфичных задач.

| # | Тема | Файл | Когда |
|---|------|------|-------|
| 30 | Reflection & Expression Trees | [[CSharp/reflection-expression-trees\|reflection-expression-trees]] | Lib authors, ORM internals |
| 31 | Source Generators | [[CSharp/source-generators\|source-generators]] | AOT-friendly libs |
| 32 | Compilation / JIT | [[Runtime/compilation-jit\|compilation-jit]] | Performance work |
| 33 | Interop / P/Invoke | [[Runtime/interop-pinvoke\|interop-pinvoke]] | Native libs, MetaTrader |
| 34 | Native AOT | [[AspNetCore/native-aot\|native-aot]] | CLI tools, microservices |
| 35 | GraphQL | [[AspNetCore/graphql\|graphql]] | Сложные queries / BFF |
| 36 | SignalR | [[AspNetCore/signalr\|signalr]] | Real-time features |
| 37 | gRPC / IPC | [[Infrastructure/ipc-named-pipes-grpc\|ipc-named-pipes-grpc]] | Microservice-to-microservice |
| 38 | Messaging | [[Infrastructure/messaging\|messaging]] | Async architecture |
| 39 | Code Quality | [[Quality/code-quality\|code-quality]] | Tech leads |
| 40 | Architecture Tests | [[Architecture/arch-tests\|arch-tests]] | Большие проекты |
| 41 | ADRs | [[Architecture/architecture-decisions\|architecture-decisions]] | Tech leads |
| 42 | System Design | [[Architecture/system-design\|system-design]] | FAANG interviews |

---

## 🌐 Specialized — для специфических ниш

Только если работаешь в этой нише.

| Тема | Когда |
|------|-------|
| [[Performance/hft-low-latency\|HFT / Low Latency]] | Trading, real-time analytics |
| [[Infrastructure/wpf-production\|WPF Production]] | Desktop development legacy |
| [[CSharp/desktop-frameworks\|Desktop Frameworks]] | Cross-platform desktop |
| [[CSharp/cli-tools-scripting\|CLI Tools]] | DevTools, internal automation |
| [[AspNetCore/blazor-wasm\|Blazor WASM]] | Frontend на C# |
| [[AspNetCore/blazor-server\|Blazor Server]] | Internal apps |
| [[Infrastructure/llm-rag-patterns\|LLM/RAG]] | AI integration |
| [[Infrastructure/semantic-kernel\|Semantic Kernel]] | Microsoft AI stack |
| [[Architecture/webai-csharp-architecture\|Web/AI Architecture]] | AI-powered apps |

---

## 📚 Mostly cultural / philosophical

Не делает тебя лучшим coder, но расширяет кругозор.

| Тема | Польза |
|------|--------|
| [[CSharp/csharp-language-design\|C# Language Design]] | Понимать "почему так" |
| [[CSharp/csharp-vs-other-langs\|C# vs Other Languages]] | Polyglot mindset |
| [[CSharp/error-handling\|Error Handling]] | Patterns for errors |
| [[CSharp/oop\|OOP]] | Деep OOP knowledge |
| [[CSharp/types-and-memory\|Types & Memory]] | Foundation |
| [[CSharp/collections-linq\|Collections & LINQ]] | Daily tools |
| [[CSharp/delegates-events\|Delegates & Events]] | Языковые механики |
| [[CSharp/design-patterns\|Design Patterns]] | GoF классика |

---

## 🎯 Прохождение по приоритетам

### Если у меня 1 неделя

Топ-5 critical:

1. [[CSharp/async-threading|async-threading]]
2. [[EFCore/basics-tracking|basics-tracking]]
3. [[EFCore/queries-performance|queries-performance]]
4. [[AspNetCore/di-configuration|di-configuration]]
5. [[Architecture/patterns|patterns]]

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
3. **Span<T>** для строк — zero-allocation parsing
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
