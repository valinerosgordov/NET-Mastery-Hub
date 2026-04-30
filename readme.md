---
tags: [index, navigation, readme]
level: All
date: 2026-04-30
---

# 📚 C# Senior Knowledge Base

> Production-grade .NET knowledge base для Senior+ разработчиков. **97 файлов, ~2.6 MB**, coverage Junior→Senior.

---

## 🚀 Куда идти?

### Хочешь учиться по плану

→ **[[LearningPath/00_overview|Learning Path Overview]]**

| Цель | Ссылка |
|------|--------|
| Junior → Middle | [[LearningPath/02_junior-to-middle\|02 Junior → Middle]] |
| Middle → Senior | [[LearningPath/03_middle-to-senior\|03 Middle → Senior]] |
| Готовлюсь к собесу | [[LearningPath/04_interview-prep\|04 Interview Prep]] |
| Что учить first (priority) | [[LearningPath/05_topics-by-priority\|05 Topics by Priority]] |
| Senior cheatsheet | [[LearningPath/09_senior-tips-cheatsheet\|09 Senior Tips]] |
| Behavioral для интервью | [[LearningPath/10_interview-behavioral\|10 Interview Behavioral]] |
| Книги и курсы | [[LearningPath/99_reading-list\|99 Reading List]] |

---

## 🗺️ Структура vault (12 разделов, 97 файлов)

### LearningPath/ (8 files) — гайды как учить

```
00_overview                 — главная навигация
02_junior-to-middle         — roadmap 3-6 месяцев
03_middle-to-senior         — roadmap для Senior
04_interview-prep           — за 1-2 недели до собеса
05_topics-by-priority       — рейтинг по value/effort
09_senior-tips-cheatsheet   — quick reference
10_interview-behavioral     — soft skills, STAR
99_reading-list             — книги, blogs, conferences
```

### CSharp/ (15 files) — язык

```
async-threading             — Task, async/await, Channels
collections-linq            — Collections, LINQ, Immutable
delegates-events            — Func/Action, events
design-patterns             — GoF и .NET-specific
error-handling              — Exceptions vs Result<T,E>
functional-csharp           — FP стиль: records, monads, railway
modern-features             — все language features 8→14
oop                         — классы, interfaces, inheritance
reflection-expression-trees — метапрограммирование
source-generators           — compile-time codegen
types-and-memory            — value/reference, struct
csharp-language-design      — эволюция C# 1.0→14
csharp-vs-other-langs       — C# vs TS/Kotlin/Rust/Go/Python/F#
cli-tools-scripting         — System.CommandLine, Spectre
desktop-frameworks          — WPF/Avalonia/Uno/MAUI
```

### Runtime/ (6 files) — низкий уровень CLR

```
compilation-jit             — Roslyn, IL, JIT, tiered, AOT
concurrency-atomics         — locks, atomics, memory model
diagnostics-tools           — dotnet-counters/trace/dump
gc-memory                   — GC generations, regions, leaks
interop-pinvoke             — P/Invoke, COM, marshalling
span-layout                 — Span<T>, struct layout
```

### AspNetCore/ (14 files) — web framework

```
api-design                  — REST, OpenAPI, versioning
auth-security               — JWT, OAuth, OIDC, RBAC
blazor-server               — Blazor Server (SignalR)
blazor-wasm                 — Blazor WebAssembly
caching                     — IMemoryCache, IDistributedCache
di-configuration            — DI lifetimes, Options
graphql                     — HotChocolate, DataLoader
hosting-background          — IHostedService, BackgroundService
logging-observability       — ILogger, Serilog, OTel
native-aot                  — AOT compilation, trimming
pipeline-middleware         — middleware, IExceptionHandler
resilience                  — Polly, retries
security-practices          — OWASP top 10
signalr                     — Real-time, Hubs
```

### EFCore/ (6 files) — ORM

```
basics-tracking             — DbContext, Change Tracker
concurrency                 — optimistic, pessimistic locks
migrations                  — code-first migrations
patterns                    — Repository, UoW, Soft Delete
queries-performance         — N+1, projections
relationships               — 1:1, 1:N, N:N, TPH/TPT
```

### SQL/ (4 files) — реляционные БД

```
sql-basics                  — DDL/DML, JOINs, transactions
indexes-deep                — индексы досконально, EXPLAIN
optimization                — query plans, performance
postgresql-deep             — PG specifics, RLS, JSONB
```

### Architecture/ (10 files) — паттерны

```
arch-tests                  — NetArchTest
architecture-decisions      — ADR, RFC, Design Docs
cqrs-mediatr                — CQRS pattern, MediatR
ddd                         — Domain-Driven Design
distributed-systems         — Saga, Outbox
microservices-vs-monolith   — выбор архитектуры
patterns                    — Modular Monolith, VSA, Clean
solid                       — SOLID
system-design               — high-level design
webai-csharp-architecture   — AI integration
```

### Infrastructure/ (8 files) — DevOps

```
docker                      — Dockerfile, multi-stage
ipc-named-pipes-grpc        — IPC patterns
llm-rag-patterns            — LLM/RAG в .NET
messaging                   — RabbitMQ, MassTransit
observability               — OTel, Prometheus, Jaeger
project-setup               — Directory.Build.props, CPM
semantic-kernel             — Microsoft Semantic Kernel
wpf-production              — WPF production patterns
```

### Performance/ (11 files) — performance

```
performance + hft + 9 топиков
(memory profiling, optimization patterns, caching strategies, etc.)
```

### Quality/ (5 files) — качество кода

```
clean-code                  — fundamentals, naming, smells
code-quality                — quality gates
code-review                 — PR review process
refactoring                 — safe refactoring techniques
static-analysis             — analyzers, SonarQube, Roslyn
```

### Testing/ (5 files) — тестирование

```
testing                     — основы xUnit
testing-fundamentals        — что такое тест, виды тестов
integration-testing         — WebApplicationFactory, Testcontainers
mocking-strategies          — NSubstitute, anti-patterns
mutation-load-testing       — Stryker.NET, NBomber, k6
```

### Snippets/ (5 files) — готовые рецепты

```
crud-example, efcore-queries, mediatr-handlers,
result-pattern, wpf-viewmodel
```

---

## 📊 Статистика

| | Значение |
|--|----------|
| Всего файлов | 97 |
| Общий объём | 2.56 MB |
| Уровень | Junior → Senior+ |
| Coverage | ~95% Senior .NET topics 2026 |
| Язык | Русский (основной), English терминология |

---

## 🎯 Что нового в этой knowledge base

- **Глубина** — каждый файл 25-60 KB, не поверхностные tutorials
- **Junior → Senior** — все уровни покрыты
- **Pitfalls везде** — что НЕ делать
- **Real production patterns** — не academic
- **Cross-references** — связи между темами (`[[file]]`)
- **2026 actual** — учитывает .NET 10, C# 14
- **Сравнение фреймворков** — не "только Microsoft way"
- **Multi-paradigm** — OOP + functional, polyglot mindset

---

## 🔑 Top-15 must-read для Senior

Если время ограничено:

1. **[[CSharp/async-threading|async-threading]]** — Task, async/await internals
2. **[[Runtime/gc-memory|GC и память]]** — поколения, regions, leaks
3. **[[EFCore/basics-tracking|EF Core Basics]]** — Change Tracker, AsNoTracking
4. **[[EFCore/queries-performance|EF Queries Performance]]** — N+1
5. **[[AspNetCore/pipeline-middleware|Pipeline & Middleware]]** — pipeline
6. **[[AspNetCore/auth-security|Auth & Security]]** — JWT, OAuth
7. **[[Architecture/patterns|Architecture Patterns]]** — Modular Monolith, VSA
8. **[[Architecture/microservices-vs-monolith|Microservices vs Monolith]]** — выбор
9. **[[Architecture/ddd|DDD]]** — aggregates, domain events
10. **[[SQL/sql-basics|SQL Basics]]** — JOIN, transactions
11. **[[SQL/indexes-deep|Indexes Deep]]** — производительность БД
12. **[[Testing/testing-fundamentals|Testing Fundamentals]]** — виды тестов
13. **[[Testing/integration-testing|Integration Testing]]** — modern stack
14. **[[Quality/clean-code|Clean Code]]** — читаемый код
15. **[[Quality/code-review|Code Review]]** — process & culture

---

## 📝 Полный список новых тем (Junior coverage)

В отличие от прошлых версий, теперь vault имеет полный **Junior intro** для:

- ✅ **Quality/clean-code** — что такое чистый код
- ✅ **Testing/testing-fundamentals** — что такое тест, виды
- ✅ **SQL/sql-basics** — DDL/DML/JOINs/transactions
- ✅ **Architecture/microservices-vs-monolith** — когда что выбирать

---

## 🛠️ Conventions

- **Russian** — основной язык
- **English** — технические термины
- **Frontmatter** — все файлы с tags / level / date
- **Code blocks** — пустая строка между header и ```
- **Cross-refs** — `[[Folder/file|display name]]`
- **Pitfalls section** — обязательно
- **Best Practices summary** — обязательно
- **Reading list** — books, docs, blogs

### Audit / maintenance

```powershell
# Проверка форматирования
& "_Claude\format_audit.ps1"

# Auto-fix code blocks
& "_Claude\fix_formatting.ps1"
```

---

## 🚀 Roadmap (что ещё может улучшиться)

Папки <10 файлов которые можно расширить:

- **EFCore** (6) — добавить: Dapper comparison, raw SQL, EF vs Dapper, stored procedures
- **Runtime** (6) — добавить: threading-basics (Junior), stack-vs-heap visual, ref-out-in, ref-struct deep
- **Snippets** (5) — добавить: logging, auth-jwt, healthcheck, docker-compose, github-actions snippets
- **Quality** (5) — добавить: technical-debt, naming-conventions, defensive-programming
- **Testing** (5) — добавить: TDD-BDD deep, e2e-playwright, contract-testing-pact, test-design-patterns
- **SQL** (4) — добавить: transactions-deep, window-functions, redis-deep, nosql-mongo, query-plans-explain
- **Infrastructure** (8) — добавить: kubernetes, cicd-github-actions, azure-cloud, secrets-management

Это incremental work, можно делать по одному файлу за раз.

---

## 📜 Changelog

См. [[_changelog|_changelog]] — полная история изменений.

Последние major updates:
- **2026-04-30 (late)** — Реструктуризация: Meta → LearningPath, +SQL basics, +Quality/Testing fundamentals
- **2026-04-30 (deep)** — Phase 7 expansion: clean-code, refactoring, code-review, testing-fundamentals, mocking-strategies, indexes-deep, microservices-vs-monolith
