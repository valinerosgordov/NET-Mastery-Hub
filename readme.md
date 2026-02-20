# .NET — База знаний

Полный справочник по C#, .NET, архитектуре и подготовке к интервью.

---

## C# Reference — Полный справочник языка

> Детальные заметки с примерами кода по каждой теме. Открывай нужный раздел как шпаргалку.

| # | Тема | Заметка | Что внутри |
|---|------|---------|------------|
| 1 | Типы и основы | [[Reference/csharp-types-and-basics\|Типы, переменные, основы]] | Value/Reference types, Stack/Heap, Boxing, операторы, строки, массивы, enum, struct |
| 2 | ООП и классы | [[Reference/csharp-oop-classes\|ООП и классы]] | Классы, свойства, наследование, интерфейсы, полиморфизм, records, IDisposable |
| 3 | Collections и LINQ | [[Reference/csharp-collections-linq\|Collections и LINQ]] | List, Dictionary, HashSet, Concurrent collections, LINQ операторы, Generics |
| 4 | Delegates и Events | [[Reference/csharp-delegates-events\|Delegates, Events, Lambdas]] | Delegate, Action/Func, лямбды, events, замыкания, паттерны |
| 5 | Async и Threading | [[Reference/csharp-async-threading\|Async и Threading]] | Task, async/await, CancellationToken, Channel, синхронизация, BackgroundService |
| 6 | Ошибки, строки, I/O | [[Reference/csharp-error-handling\|Обработка ошибок, строки, I/O]] | Exceptions, строки, файлы, JSON, Regex |
| 7 | Modern C# | [[Reference/csharp-modern-features\|Современные возможности C# 8–14]] | Pattern matching, nullable, records, primary constructors, collection expressions |

---

## Architecture

- [[Architecture/architecture-tutorial|Архитектуры: Clean, VSA, N-Layered]] — туториал, плюсы и минусы, гибрид
- [[Architecture/architecture-conventions-and-tests|Соглашения и тесты]] — именование, архитектурные тесты
- [[Architecture/architecture-tests-netarchtest|NetArchTest]] — проверка слоёв, Modular Monolith

## Interview

- [[Interview/interview-index|Ответы на интервью]] — 9 категорий: C#, Async, ASP.NET, Security, EF Core, Logging, Architecture, Testing, Behavioral

## Learning Path

- [[Topics/LearningPath/learning-path|Learning Path]] — Todo list для обучения, уровни, оценка времени

## 150 .NET Questions

- [[Topics/NetQuestions150/net-questions-150|150 вопросов]] — C#, ASP.NET Core, EF Core по модулям

## Topics

| Тема | Заметка | Описание |
|------|---------|----------|
| Code Quality | [[Topics/CodeQuality/code-quality-best-practices\|Code Quality]] | Analyzers, EditorConfig |
| Observability | [[Topics/Observability/opentelemetry-jaeger-seq\|OpenTelemetry + Jaeger + Seq]] | Трассировка, метрики |
| Testing | [[Topics/Testing/testing-xunit-testcontainers\|Testing: xUnit + Testcontainers]] | Unit, Integration |
| Docker | [[Topics/Docker/docker-deploy\|Docker и CI/CD]] | Dockerfile, docker-compose |
| Messaging | [[Topics/Messaging/rabbitmq-masstransit\|RabbitMQ + MassTransit]] | Очереди, Azure Service Bus |
| Performance | [[Topics/Performance/dotnet-performance\|.NET Performance]] | Span, ArrayPool, BenchmarkDotNet |
| Result Pattern | [[Topics/ResultPattern/result-pattern-cqrs\|Result Pattern + CQRS]] | Railway, MediatR |
| SQL | [[Topics/SQL/sql-query-optimization\|SQL Optimization]] | Индексы, планы запросов |
| Project Setup | [[Topics/ProjectSetup/start-dotnet-project-2026\|Start .NET Project 2026]] | Шаблон проекта |
| Project Setup | [[Topics/ProjectSetup/top-10-things-dotnet-2026\|Top 10 .NET 2026]] | Ключевые практики |
| Repositories | [[Topics/Repositories/dotnet-github-repos\|.NET GitHub Repos]] | Полезные репозитории |
| Tips & Tricks | [[Topics/Tips/csharp-channel-tips\|C# Tips & Tricks]] | Feature Flags, Auto-Registration, Load Balancing, C# 14 |

## Snippets

- [[Topics/Snippets/snippet-mediatr-handlers|MediatR Handlers]] — Command/Query handler с Result
- [[Topics/Snippets/snippet-result-pattern|Result Usage]] — примеры Result/Option
- [[Topics/Snippets/snippet-efcore-queries|EF Core Queries]] — запросы, Include, проекции
- [[Topics/Snippets/snippet-wpf-viewmodel|WPF ViewModel]] — MVVM Toolkit, ObservableProperty
