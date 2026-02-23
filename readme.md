# .NET Knowledge Base

> Персональная база знаний по C#, .NET, архитектуре и подготовке к собеседованиям.
> **42 заметки** — единый справочник с встроенными вопросами интервью, production-ready кодом и Senior-level разборами.

---

## Структура

```
C#/
├── CSharp/           — Язык C# (7 заметок)
├── Runtime/          — .NET Internals: JIT, GC, Span, Concurrency (4)
├── AspNetCore/       — ASP.NET Core: pipeline → logging (7)
├── EFCore/           — Entity Framework Core (6)
├── Architecture/     — Clean, VSA, CQRS, архтесты (3)
├── Testing/          — xUnit, Testcontainers (1)
├── Infrastructure/   — Docker, Messaging, Observability (4)
├── Performance/      — BenchmarkDotNet, profiling, leaks (1)
├── SQL/              — Индексы, планы запросов (1)
├── Quality/          — Analyzers, EditorConfig (1)
├── Snippets/         — Готовые сниппеты кода (4)
└── Meta/             — Learning Path, Behavioral (2)
```

---

## C# Language

| # | Тема | Ключевое |
|---|------|----------|
| 1 | [Типы и память](CSharp/types-and-memory.md) | Value/Reference, Stack/Heap, Boxing, Span, struct vs class |
| 2 | [ООП и классы](CSharp/oop.md) | Наследование, интерфейсы, полиморфизм, records, IDisposable |
| 3 | [Collections и LINQ](CSharp/collections-linq.md) | List, Dictionary, HashSet, Concurrent, LINQ, Expression Trees |
| 4 | [Delegates и Events](CSharp/delegates-events.md) | Delegate, Action/Func, лямбды, events, замыкания |
| 5 | [Ошибки, строки, I/O](CSharp/error-handling.md) | Exceptions, строки, JSON, файлы, Regex |
| 6 | [Async и Threading](CSharp/async-threading.md) | Task, async/await, CancellationToken, Channel, синхронизация |
| 7 | [Modern C# 8–14](CSharp/modern-features.md) | Pattern matching, nullable, records, primary constructors |

---

## .NET Runtime (Deep Dive)

| # | Тема | Ключевое |
|---|------|----------|
| 1 | [Компиляция и JIT](Runtime/compilation-jit.md) | Roslyn, IL, JIT, Tiered Compilation, R2R, NativeAOT, Dynamic PGO |
| 2 | [GC, LOH и POH](Runtime/gc-memory.md) | Поколения, Mark-Sweep-Compact, LOH-фрагментация, Finalization |
| 3 | [Span, Memory, Layout](Runtime/span-layout.md) | ref struct, stackalloc, Data Alignment, StructLayout |
| 4 | [Concurrency и Atomics](Runtime/concurrency-atomics.md) | CAS, volatile, Lock-free, Memory Barriers |

---

## ASP.NET Core

| # | Тема | Ключевое |
|---|------|----------|
| 1 | [Pipeline и Middleware](AspNetCore/pipeline-middleware.md) | Request pipeline, routing, middleware, filters |
| 2 | [DI и Configuration](AspNetCore/di-configuration.md) | ServiceLifetime, Options pattern, validation |
| 3 | [Auth и Security](AspNetCore/auth-security.md) | JWT, CORS, policies, claims, Data Protection |
| 4 | [Hosting и Background](AspNetCore/hosting-background.md) | BackgroundService, IHostedService, Kestrel |
| 5 | [Caching](AspNetCore/caching.md) | IMemoryCache, IDistributedCache, Rate Limiting |
| 6 | [API Design](AspNetCore/api-design.md) | Controllers, Minimal API, versioning, content negotiation |
| 7 | [Logging и Observability](AspNetCore/logging-observability.md) | ILogger, Serilog, OpenTelemetry, Jaeger, Seq |

---

## Entity Framework Core

| # | Тема | Ключевое |
|---|------|----------|
| 1 | [Basics и Tracking](EFCore/basics-tracking.md) | DbContext, Change Tracker, loading strategies |
| 2 | [Queries и Performance](EFCore/queries-performance.md) | N+1, compiled queries, проекции, split queries |
| 3 | [Relationships](EFCore/relationships.md) | FK, navigation, owned types, many-to-many |
| 4 | [Migrations](EFCore/migrations.md) | Schema management, seed data, idempotent scripts |
| 5 | [Concurrency](EFCore/concurrency.md) | Optimistic concurrency, transactions, retry |
| 6 | [Patterns](EFCore/patterns.md) | Repository, TPH/TPT, soft delete, audit |

---

## Architecture

| # | Тема | Ключевое |
|---|------|----------|
| 1 | [Patterns](Architecture/patterns.md) | Clean Architecture, VSA, N-Layered, масштабирование |
| 2 | [CQRS и MediatR](Architecture/cqrs-mediatr.md) | Result Pattern, Command/Query, pipeline behaviors |
| 3 | [Архитектурные тесты](Architecture/arch-tests.md) | NetArchTest, проверка слоёв, конвенции |

---

## Специализированные темы

| Тема | Ключевое |
|------|----------|
| [Testing](Testing/testing.md) | Пирамида тестов, xUnit, Testcontainers, mocking |
| [Docker](Infrastructure/docker.md) | Dockerfile, multi-stage, docker-compose |
| [Messaging](Infrastructure/messaging.md) | RabbitMQ, MassTransit, Azure Service Bus |
| [Observability](Infrastructure/observability.md) | OpenTelemetry, Jaeger, Seq, метрики |
| [Project Setup](Infrastructure/project-setup.md) | Шаблон .NET проекта 2026, CI/CD |
| [Performance](Performance/performance.md) | BenchmarkDotNet, profiling, memory leaks |
| [SQL Optimization](SQL/optimization.md) | Индексы, планы запросов, транзакции |
| [Code Quality](Quality/code-quality.md) | Roslyn Analyzers, SonarQube, EditorConfig |

---

## Snippets — Готовый код

| Сниппет | Описание |
|---------|----------|
| [MediatR Handlers](Snippets/mediatr-handlers.md) | Command/Query handler с Result |
| [Result Pattern](Snippets/result-pattern.md) | Примеры Result/Option |
| [EF Core Queries](Snippets/efcore-queries.md) | Запросы, Include, проекции |
| [WPF ViewModel](Snippets/wpf-viewmodel.md) | MVVM Toolkit, ObservableProperty |

---

## Meta

| Заметка | Описание |
|---------|----------|
| [Learning Path](Meta/learning-path.md) | Пошаговый план обучения с оценкой времени |
| [Behavioral](Meta/behavioral.md) | Подготовка к behavioral интервью |

---

## Формат заметок

Каждая заметка следует единому формату:

```
---
tags: [тема1, тема2]
level: Senior
---

# Название

## Теория
...

### Пример (production-ready код)

> [!question]- Интервью: Вопрос?
> Развёрнутый ответ встроен РЯДОМ с теорией.

## См. также
- [Ссылка](../path.md)
```

Вопросы интервью встроены в темы как сворачиваемые callouts `> [!question]-` — рядом с теорией, к которой относятся.
