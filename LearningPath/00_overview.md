---
tags: [learning-path, overview, navigation]
level: All
date: 2026-04-30
---

# 📚 Learning Path — Overview

> Главная навигация по vault. Гайд "с чего начать" для разных уровней. Все ссылки ведут на актуальные файлы в этой knowledge base.

---

## 🎯 Куда я хочу прийти?

Выбери цель — увидишь свой путь:

| Цель | Roadmap |
|------|---------|
| Системно выучить/повторить сам язык | [[01_language-map\|01 Language Map]] |
| Junior → Middle .NET Backend | [[02_junior-to-middle\|02 Junior → Middle]] |
| Middle → Senior .NET | [[03_middle-to-senior\|03 Middle → Senior]] |
| Готовлюсь к собеседованию | [[04_interview-prep\|04 Interview Prep]] |
| Прокачать конкретную тему | [[05_topics-by-priority\|05 Topics by Priority]] |
| Senior tips & tricks (cheatsheet) | [[09_senior-tips-cheatsheet\|09 Senior Tips]] |
| Behavioral / soft skills для интервью | [[10_interview-behavioral\|10 Interview Behavioral]] |
| Книги и видео | [[99_reading-list\|99 Reading List]] |

---

## 🗺️ Структура vault

```
C#/                        — язык: основы, продвинутые темы, FP, language design
├── async-threading        — Task, async/await, Channels, ValueTask
├── collections-linq       — Collections, LINQ, ImmutableArray
├── delegates-events       — Func/Action, events, weak events
├── design-patterns        — GoF и .NET-specific patterns
├── error-handling         — Exceptions vs Result<T,E>
├── functional-csharp      ★ FP стиль: records, pattern matching, monads
├── modern-features        — все language features 8→14
├── oop                    — классы, interfaces, inheritance
├── reflection-expression-trees ★ метапрограммирование, EF/AutoMapper internals
├── source-generators      — compile-time codegen
├── types-and-memory       — value/reference, struct, ref returns
├── csharp-language-design ★ эволюция C# 1.0→14, философия
├── csharp-vs-other-langs  ★ C# vs TS/Kotlin/Rust/Go/Python/F#
├── cli-tools-scripting    ★ System.CommandLine, Spectre.Console
└── desktop-frameworks     ★ WPF/Avalonia/Uno/MAUI comparison

Runtime/                   — низкий уровень CLR
├── compilation-jit        — Roslyn, IL, JIT, tiered, AOT
├── concurrency-atomics    — locks, atomics, memory model
├── diagnostics-tools      ★ dotnet-counters/trace/dump/monitor
├── gc-memory              — GC generations, regions, DATAS, leaks
├── interop-pinvoke        ★ P/Invoke, COM, marshalling
└── span-layout            — Span<T>, struct layout, stackalloc

AspNetCore/                — web framework
├── api-design             — REST, OpenAPI, versioning
├── auth-security          — JWT, OAuth, OIDC, RBAC
├── blazor-server          — Blazor Server (SignalR-based)
├── blazor-wasm            ★ Blazor WebAssembly
├── caching                — IMemoryCache, IDistributedCache, Redis
├── di-configuration       — DI lifetimes, Options, IConfiguration
├── graphql                ★ HotChocolate, DataLoader, Federation
├── hosting-background     — IHostedService, BackgroundService
├── logging-observability  — ILogger, Serilog, OpenTelemetry
├── native-aot             — AOT compilation, trimming
├── pipeline-middleware    — middleware, IExceptionHandler
├── resilience             — Polly, Polly.RateLimit, retries
├── security-practices     — OWASP top 10, security checklist
└── signalr                ★ Real-time, Hubs, Redis backplane

EFCore/                    — ORM
├── basics-tracking        — DbContext, Change Tracker, AsNoTracking
├── concurrency            — optimistic, pessimistic locks
├── migrations             — code-first migrations, idempotent SQL
├── patterns               — Repository, UoW, Specification, Soft Delete
├── queries-performance    — N+1, projections, AsSplitQuery
└── relationships          — 1:1, 1:N, N:N, TPH/TPT/TPC

SQL/                       — реляционные БД
├── optimization           — индексы, EXPLAIN ANALYZE
└── postgresql-deep        — PostgreSQL specifics, RLS, JSONB

Architecture/              — архитектура и паттерны
├── arch-tests             — NetArchTest для архитектурных правил
├── architecture-decisions — ADR, RFC, Design Docs
├── cqrs-mediatr           — CQRS pattern, MediatR
├── ddd                    — Domain-Driven Design
├── distributed-systems    — Saga, Outbox, eventual consistency
├── patterns               — Modular Monolith, Vertical Slices, Clean
├── solid                  — SOLID principles
├── system-design          — high-level system design
└── webai-csharp-architecture — AI integration patterns

Infrastructure/            — DevOps, deployment, integration
├── docker                 — Dockerfile, multi-stage, security
├── ipc-named-pipes-grpc   — IPC patterns
├── llm-rag-patterns       — LLM/RAG в .NET
├── messaging              — RabbitMQ, MassTransit, Kafka
├── observability          — OTel, Prometheus, Jaeger
├── project-setup          — Directory.Build.props, CPM, .editorconfig
├── semantic-kernel        — Microsoft Semantic Kernel
└── wpf-production         — WPF production patterns

Performance/               — performance work
├── hft-low-latency        — HFT patterns, MetaTrader
└── performance            — BenchmarkDotNet, profiling

Quality/                   — качество кода
└── code-quality           — analyzers, linting, code review

Testing/                   — тестирование
└── testing                — xUnit, integration, Testcontainers, NSubstitute

Snippets/                  — готовые рецепты
├── crud-example
├── efcore-queries
├── mediatr-handlers
├── result-pattern
└── wpf-viewmodel

LearningPath/              — этот раздел
```

★ = недавно добавлено / расширено

---

## 📊 Статистика vault

- **Всего файлов:** ~80
- **Общий объём:** ~2.3 MB
- **Уровень:** Senior+ (большинство файлов)
- **Покрытие:** ~92% Senior .NET topics 2026
- **Языки:** русский (основной), технические термины — английские

---

## 🚀 Quick Start

**Если ты новичок в этом vault:**

1. Открой [[02_junior-to-middle\|02 Junior → Middle]] или [[03_middle-to-senior\|03 Middle → Senior]] в зависимости от уровня
2. Открой соответствующий файл в нужной папке
3. После прочтения — пиши код по теме (pet-project!)
4. Возвращайся к [[09_senior-tips-cheatsheet\|09 Senior Tips]] для повторения

**Если готовишься к собесу:**

1. [[04_interview-prep\|04 Interview Prep]] — что повторить
2. [[10_interview-behavioral\|10 Interview Behavioral]] — soft skills
3. [[09_senior-tips-cheatsheet\|09 Senior Tips]] — быстрый review

**Если хочешь в конкретную тему:**

[[05_topics-by-priority\|05 Topics by Priority]] — рейтинг тем по value/effort

---

## 🎓 Уровни

| Уровень | Что значит | Как достичь |
|---------|-----------|-------------|
| **Junior** | Пишет CRUD по заданию | C# fundamentals + ASP.NET basics |
| **Junior+** | Самостоятельно делает простые фичи | + EF Core + auth |
| **Middle** | Полный цикл разработки | + architecture + testing + DevOps |
| **Middle+** | Проектирует фичи, делает code review | + DDD + observability + performance |
| **Senior** | Архитектурные решения, mentoring | + system design + leadership |
| **Senior+** | Cross-functional impact, R&D | + business + cross-team |

См. [[02_junior-to-middle\|02 Junior → Middle]] и [[03_middle-to-senior\|03 Middle → Senior]] для детальных roadmap.
