# .NET Knowledge Base

> Персональная база знаний по C#, .NET, архитектуре и подготовке к собеседованиям.
> **38 заметок** — справочник, 150+ вопросов, сниппеты, best practices.

---

## Структура репозитория

```
├── Reference/          — Справочник языка C# (7 заметок)
├── Architecture/       — Clean Architecture, VSA, архитектурные тесты
├── Interview/          — 9 категорий вопросов для собеседований
├── Topics/
│   ├── NetQuestions150/ — 150 вопросов: C#, ASP.NET Core, EF Core
│   ├── CodeQuality/    — Analyzers, EditorConfig
│   ├── Docker/         — Dockerfile, CI/CD
│   ├── Messaging/      — RabbitMQ, MassTransit
│   ├── Observability/  — OpenTelemetry, Jaeger, Seq
│   ├── Performance/    — Span, ArrayPool, BenchmarkDotNet
│   ├── Testing/        — xUnit, Testcontainers
│   ├── ResultPattern/  — Result Pattern, CQRS, MediatR
│   ├── SQL/            — Индексы, планы запросов
│   ├── ProjectSetup/   — Шаблон .NET проекта 2026
│   ├── Repositories/   — Полезные GitHub-репозитории
│   ├── Tips/           — Feature Flags, C# 14, трюки
│   ├── Snippets/       — Готовые сниппеты кода
│   └── LearningPath/   — План обучения с оценкой времени
```

---

## C# Reference — Справочник языка

| # | Тема | Что внутри |
|---|------|------------|
| 1 | [Типы и основы](Reference/csharp-types-and-basics.md) | Value/Reference types, Stack/Heap, Boxing, строки, массивы, enum, struct |
| 2 | [ООП и классы](Reference/csharp-oop-classes.md) | Классы, наследование, интерфейсы, полиморфизм, records, IDisposable |
| 3 | [Collections и LINQ](Reference/csharp-collections-linq.md) | List, Dictionary, HashSet, Concurrent collections, LINQ, Generics |
| 4 | [Delegates и Events](Reference/csharp-delegates-events.md) | Delegate, Action/Func, лямбды, events, замыкания |
| 5 | [Async и Threading](Reference/csharp-async-threading.md) | Task, async/await, CancellationToken, Channel, BackgroundService |
| 6 | [Ошибки, строки, I/O](Reference/csharp-error-handling.md) | Exceptions, строки, файлы, JSON, Regex |
| 7 | [Modern C# 8–14](Reference/csharp-modern-features.md) | Pattern matching, nullable, records, primary constructors, collection expressions |

---

## Architecture

| Заметка | Описание |
|---------|----------|
| [Архитектуры: Clean, VSA, N-Layered](Architecture/architecture-tutorial.md) | Туториал, плюсы/минусы, гибрид |
| [Соглашения и тесты](Architecture/architecture-conventions-and-tests.md) | Именование, архитектурные тесты |
| [NetArchTest](Architecture/architecture-tests-netarchtest.md) | Проверка слоёв, Modular Monolith |

---

## Interview — Подготовка к собеседованию

9 категорий вопросов и ответов для .NET backend интервью.

| # | Тема | Ссылка |
|---|------|--------|
| 1 | C# Fundamentals | [Открыть](Interview/1-csharp-fundamentals.md) |
| 2 | Async & Threading | [Открыть](Interview/2-async-threading.md) |
| 3 | ASP.NET Core | [Открыть](Interview/3-aspnet-core.md) |
| 4 | Security | [Открыть](Interview/4-security.md) |
| 5 | EF Core & SQL | [Открыть](Interview/5-ef-core-sql.md) |
| 6 | Logging & Metrics | [Открыть](Interview/6-logging-metrics.md) |
| 7 | Architecture | [Открыть](Interview/7-architecture.md) |
| 8 | Testing | [Открыть](Interview/8-testing.md) |
| 9 | Behavioral | [Открыть](Interview/9-behavioral.md) |

---

## 150 .NET Questions

Подробные вопросы и ответы по трём направлениям: **[Оглавление](Topics/NetQuestions150/net-questions-150.md)**

<details>
<summary><strong>C# (5 модулей)</strong></summary>

| Модуль | Ссылка |
|--------|--------|
| Types & Memory | [01-types-memory](Topics/NetQuestions150/csharp/01-types-memory.md) |
| OOP | [02-oop](Topics/NetQuestions150/csharp/02-oop.md) |
| Collections & LINQ | [03-collections-linq](Topics/NetQuestions150/csharp/03-collections-linq.md) |
| Async & Concurrency | [04-async-concurrency](Topics/NetQuestions150/csharp/04-async-concurrency.md) |
| Language Features | [05-language](Topics/NetQuestions150/csharp/05-language.md) |

</details>

<details>
<summary><strong>ASP.NET Core (8 модулей)</strong></summary>

| Модуль | Ссылка |
|--------|--------|
| Pipeline & Routing | [01-pipeline-routing](Topics/NetQuestions150/aspnet/01-pipeline-routing.md) |
| DI & Configuration | [02-di-configuration](Topics/NetQuestions150/aspnet/02-di-configuration.md) |
| Options & Validation | [03-options-validation](Topics/NetQuestions150/aspnet/03-options-validation.md) |
| Auth | [04-auth](Topics/NetQuestions150/aspnet/04-auth.md) |
| Hosting | [05-hosting](Topics/NetQuestions150/aspnet/05-hosting.md) |
| Caching | [06-caching](Topics/NetQuestions150/aspnet/06-caching.md) |
| API | [07-api](Topics/NetQuestions150/aspnet/07-api.md) |
| Logging | [08-logging](Topics/NetQuestions150/aspnet/08-logging.md) |

</details>

<details>
<summary><strong>EF Core (7 модулей)</strong></summary>

| Модуль | Ссылка |
|--------|--------|
| Migrations & Schema | [01-migrations-schema](Topics/NetQuestions150/efcore/01-migrations-schema.md) |
| Loading & Tracking | [02-loading-tracking](Topics/NetQuestions150/efcore/02-loading-tracking.md) |
| Relationships | [03-relationships](Topics/NetQuestions150/efcore/03-relationships.md) |
| Queries | [04-queries](Topics/NetQuestions150/efcore/04-queries.md) |
| Performance | [05-performance](Topics/NetQuestions150/efcore/05-performance.md) |
| Concurrency & Transactions | [06-concurrency-transactions](Topics/NetQuestions150/efcore/06-concurrency-transactions.md) |
| Patterns | [07-patterns](Topics/NetQuestions150/efcore/07-patterns.md) |

</details>

---

## Topics

| Тема | Заметка | Описание |
|------|---------|----------|
| Code Quality | [Code Quality](Topics/CodeQuality/code-quality-best-practices.md) | Analyzers, EditorConfig |
| Observability | [OpenTelemetry + Jaeger + Seq](Topics/Observability/opentelemetry-jaeger-seq.md) | Трассировка, метрики |
| Testing | [xUnit + Testcontainers](Topics/Testing/testing-xunit-testcontainers.md) | Unit, Integration тесты |
| Docker | [Docker и CI/CD](Topics/Docker/docker-deploy.md) | Dockerfile, docker-compose |
| Messaging | [RabbitMQ + MassTransit](Topics/Messaging/rabbitmq-masstransit.md) | Очереди, Azure Service Bus |
| Performance | [.NET Performance](Topics/Performance/dotnet-performance.md) | Span, ArrayPool, BenchmarkDotNet |
| Result Pattern | [Result Pattern + CQRS](Topics/ResultPattern/result-pattern-cqrs.md) | Railway, MediatR |
| SQL | [SQL Optimization](Topics/SQL/sql-query-optimization.md) | Индексы, планы запросов |
| Project Setup | [Start .NET Project 2026](Topics/ProjectSetup/start-dotnet-project-2026.md) | Шаблон нового проекта |
| Project Setup | [Top 10 .NET 2026](Topics/ProjectSetup/top-10-things-dotnet-2026.md) | Ключевые практики |
| Repositories | [.NET GitHub Repos](Topics/Repositories/dotnet-github-repos.md) | Полезные репозитории |
| Tips & Tricks | [C# Tips & Tricks](Topics/Tips/csharp-channel-tips.md) | Feature Flags, C# 14, Load Balancing |

---

## Snippets — Готовый код

| Сниппет | Описание |
|---------|----------|
| [MediatR Handlers](Topics/Snippets/snippet-mediatr-handlers.md) | Command/Query handler с Result |
| [Result Usage](Topics/Snippets/snippet-result-pattern.md) | Примеры Result/Option |
| [EF Core Queries](Topics/Snippets/snippet-efcore-queries.md) | Запросы, Include, проекции |
| [WPF ViewModel](Topics/Snippets/snippet-wpf-viewmodel.md) | MVVM Toolkit, ObservableProperty |

---

## Learning Path

**[План обучения](Topics/LearningPath/learning-path.md)** — пошаговый путь от основ до продвинутых тем с оценкой времени.

```
C# Fundamentals → C# Advanced → ASP.NET Core → EF Core
    → Architecture → Testing & DevOps → Advanced Topics → Interview Prep
```
