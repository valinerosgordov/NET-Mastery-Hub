# 📑 INDEX — Полное оглавление vault'а

> Все 119 файлов с короткими описаниями. **Три способа найти**: по алфавиту / по уровню / по теме.

[← Назад к главному README](README.md)

---

## 🔤 По алфавиту

### A

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Architecture/architecture-decisions.md`](Architecture/architecture-decisions.md) | Senior | ADRs — как документировать архитектурные решения |
| [`Architecture/arch-tests.md`](Architecture/arch-tests.md) | Senior | NetArchTest, ArchUnit — фитнес-функции для архитектуры |
| [`AspNetCore/api-design.md`](AspNetCore/api-design.md) | Middle/Senior | REST design, versioning, OpenAPI |
| [`Performance/async-performance.md`](Performance/async-performance.md) | Senior | Async/await performance pitfalls |
| [`CSharp/async-threading.md`](CSharp/async-threading.md) | Senior | Task, async/await internals (58 KB) |
| [`CSharp/attributes-metadata.md`](CSharp/attributes-metadata.md) | Middle | Attributes, custom attributes, reflection |
| [`AspNetCore/auth-security.md`](AspNetCore/auth-security.md) | Middle/Senior | JWT, OAuth, OIDC, RBAC |

### B

| Файл | Уровень | Описание |
|------|---------|----------|
| [`EFCore/basics-tracking.md`](EFCore/basics-tracking.md) | Middle | DbContext, ChangeTracker, AsNoTracking |
| [`AspNetCore/blazor-server.md`](AspNetCore/blazor-server.md) | Senior | Blazor Server architecture |
| [`AspNetCore/blazor-wasm.md`](AspNetCore/blazor-wasm.md) | Senior | Blazor WebAssembly |
| [`Performance/bottleneck-analysis.md`](Performance/bottleneck-analysis.md) | Senior | Profiling и поиск узких мест |

### C

| Файл | Уровень | Описание |
|------|---------|----------|
| [`AspNetCore/caching.md`](AspNetCore/caching.md) | Middle/Senior | IDistributedCache, OutputCaching, Redis |
| [`Performance/caching-strategies.md`](Performance/caching-strategies.md) | Senior | Cache-aside, write-through, invalidation |
| [`Performance/capacity-planning.md`](Performance/capacity-planning.md) | Senior | Sizing, scaling, capacity math |
| [`Infrastructure/cicd-github-actions.md`](Infrastructure/cicd-github-actions.md) | Middle | CI/CD pipelines, secrets, environments |
| [`CSharp/cli-tools-scripting.md`](CSharp/cli-tools-scripting.md) | Senior | System.CommandLine, scripting |
| [`Quality/clean-code.md`](Quality/clean-code.md) | Junior/Senior | Naming, principles, fundamentals |
| [`CSharp/collections-linq.md`](CSharp/collections-linq.md) | Senior | Collections, LINQ deep (47 KB) |
| [`Quality/code-quality.md`](Quality/code-quality.md) | Senior | Analyzers, EditorConfig, SonarCloud |
| [`Quality/code-review.md`](Quality/code-review.md) | Junior/Senior | Code review process & culture |
| [`Runtime/compilation-jit.md`](Runtime/compilation-jit.md) | Senior | JIT, AOT, ReadyToRun, tiered compilation |
| [`EFCore/concurrency.md`](EFCore/concurrency.md) | Middle/Senior | Optimistic locking, RowVersion |
| [`Runtime/concurrency-atomics.md`](Runtime/concurrency-atomics.md) | Senior | Memory model, Interlocked, lock-free |
| [`Architecture/cqrs-mediatr.md`](Architecture/cqrs-mediatr.md) | Senior | CQRS, MediatR, pipeline behaviors |
| [`Snippets/crud-example.md`](Snippets/crud-example.md) | Middle | Полный CRUD пример |
| [`CSharp/csharp-basics.md`](CSharp/csharp-basics.md) | Junior | Стартовая точка C#: переменные, типы, control flow |
| [`CSharp/csharp-language-design.md`](CSharp/csharp-language-design.md) | Senior | History, design decisions, evolution |
| [`CSharp/csharp-vs-other-langs.md`](CSharp/csharp-vs-other-langs.md) | Senior | C# vs Java/Go/Rust/Python — comparison |

### D

| Файл | Уровень | Описание |
|------|---------|----------|
| [`EFCore/dapper-comparison.md`](EFCore/dapper-comparison.md) | Middle | Dapper vs EF, hybrid patterns |
| [`CSharp/datetime-timezones.md`](CSharp/datetime-timezones.md) | Junior/Senior | DateTime, TimeZoneInfo, NodaTime |
| [`CSharp/delegates-events.md`](CSharp/delegates-events.md) | Senior | Delegates, events, Func/Action |
| [`CSharp/design-patterns.md`](CSharp/design-patterns.md) | Senior | 13 GoF patterns в C# |
| [`CSharp/desktop-frameworks.md`](CSharp/desktop-frameworks.md) | Senior | WPF, MAUI, Avalonia |
| [`Runtime/diagnostics-tools.md`](Runtime/diagnostics-tools.md) | Senior | dotnet-counters, trace, dump, PerfView |
| [`AspNetCore/di-configuration.md`](AspNetCore/di-configuration.md) | Middle/Senior | DI lifetimes, configuration system |
| [`CSharp/dispose-pattern.md`](CSharp/dispose-pattern.md) | Middle | IDisposable, IAsyncDisposable, SafeHandle |
| [`Architecture/distributed-systems.md`](Architecture/distributed-systems.md) | Senior | Saga, outbox, CAP, consistency |
| [`Infrastructure/docker.md`](Infrastructure/docker.md) | Middle/Senior | Docker, multistage, optimization (63 KB) |

### E

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Snippets/efcore-queries.md`](Snippets/efcore-queries.md) | Middle | EF query patterns snippets |
| [`CSharp/enums-flags.md`](CSharp/enums-flags.md) | Junior/Middle | Enum, [Flags], parsing |
| [`CSharp/equality-comparison.md`](CSharp/equality-comparison.md) | Middle | Equals, GetHashCode, IEquatable |
| [`CSharp/error-handling.md`](CSharp/error-handling.md) | Senior | Exceptions, Result, OneOf |
| [`CSharp/extension-methods.md`](CSharp/extension-methods.md) | Junior/Middle | Extension methods, fluent APIs |

### F

| Файл | Уровень | Описание |
|------|---------|----------|
| [`CSharp/functional-csharp.md`](CSharp/functional-csharp.md) | Senior | Records, pattern matching, FP в C# |

### G

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Runtime/gc-memory.md`](Runtime/gc-memory.md) | Senior | GC generations, regions, leaks (56 KB) |
| [`CSharp/generics-deep.md`](CSharp/generics-deep.md) | Middle/Senior | Variance, INumber\<T\>, generic math |
| [`AspNetCore/graphql.md`](AspNetCore/graphql.md) | Senior | HotChocolate, schemas, federation |

### H

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Performance/hft-low-latency.md`](Performance/hft-low-latency.md) | Senior | HFT, hot paths, < 1 ms |
| [`AspNetCore/hosting-background.md`](AspNetCore/hosting-background.md) | Middle/Senior | BackgroundService, IHostedService |

### I

| Файл | Уровень | Описание |
|------|---------|----------|
| [`CSharp/indexers-operators.md`](CSharp/indexers-operators.md) | Middle | Custom indexers, operator overloading |
| [`SQL/indexes-deep.md`](SQL/indexes-deep.md) | Senior | B-tree, query plans, index types |
| [`Testing/integration-testing.md`](Testing/integration-testing.md) | Senior | TestContainers, WebApplicationFactory |
| [`Runtime/interop-pinvoke.md`](Runtime/interop-pinvoke.md) | Senior | P/Invoke, marshalling, native interop |
| [`CSharp/io-streams.md`](CSharp/io-streams.md) | Middle | File, Stream, StreamReader, async I/O |
| [`Infrastructure/ipc-named-pipes-grpc.md`](Infrastructure/ipc-named-pipes-grpc.md) | Senior | Named pipes, gRPC, IPC |
| [`CSharp/iterators-yield.md`](CSharp/iterators-yield.md) | Junior/Middle | yield return, IAsyncEnumerable |

### K

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Infrastructure/kubernetes.md`](Infrastructure/kubernetes.md) | Middle/Senior | Pod/Deployment/Service, .NET deploy, Helm |

### L

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Performance/lazy-eager-loading.md`](Performance/lazy-eager-loading.md) | Middle | Lazy\<T\>, EF Include vs Lazy |
| [`Infrastructure/llm-rag-patterns.md`](Infrastructure/llm-rag-patterns.md) | Senior | LLM integration, RAG, vector DBs |
| [`AspNetCore/logging-observability.md`](AspNetCore/logging-observability.md) | Middle/Senior | Serilog, structured logging |

### M

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Snippets/mediatr-handlers.md`](Snippets/mediatr-handlers.md) | Middle | MediatR handler patterns |
| [`Performance/memory-profiling.md`](Performance/memory-profiling.md) | Senior | dotMemory, PerfView, memory leaks |
| [`Infrastructure/messaging.md`](Infrastructure/messaging.md) | Senior | RabbitMQ, Kafka, MassTransit |
| [`Architecture/microservices-vs-monolith.md`](Architecture/microservices-vs-monolith.md) | Senior | When microservices vs monolith |
| [`EFCore/migrations.md`](EFCore/migrations.md) | Middle | EF migrations, scripts, rollback |
| [`Testing/mocking-strategies.md`](Testing/mocking-strategies.md) | Middle/Senior | Moq, NSubstitute, fakes vs mocks |
| [`CSharp/modern-features.md`](CSharp/modern-features.md) | Senior | Records, primary ctors, raw strings |
| [`Testing/mutation-load-testing.md`](Testing/mutation-load-testing.md) | Senior | Stryker.NET, NBomber |

### N

| Файл | Уровень | Описание |
|------|---------|----------|
| [`AspNetCore/native-aot.md`](AspNetCore/native-aot.md) | Senior | Native AOT compilation |
| [`CSharp/nullable-types.md`](CSharp/nullable-types.md) | Middle | Nullable\<T\>, NRT, null operators |

### O

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Infrastructure/observability.md`](Infrastructure/observability.md) | Senior | OpenTelemetry, Prometheus, Jaeger |
| [`CSharp/oop.md`](CSharp/oop.md) | Senior | Inheritance, polymorphism, abstract |
| [`Performance/optimization-patterns.md`](Performance/optimization-patterns.md) | Senior | Performance optimization patterns |
| [`SQL/optimization.md`](SQL/optimization.md) | Senior | Query optimization, EXPLAIN |

### P

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Architecture/patterns.md`](Architecture/patterns.md) | Senior | N-Layer / Clean / VSA / Hybrid (52 KB) |
| [`Architecture/patterns-decision-guide.md`](Architecture/patterns-decision-guide.md) ⭐ | Senior | Какой паттерн под какую задачу |
| [`EFCore/patterns.md`](EFCore/patterns.md) | Middle/Senior | Repository, UoW, Specification |
| [`Performance/performance.md`](Performance/performance.md) | Senior | BenchmarkDotNet, profiling tools |
| [`Performance/performance-budgets.md`](Performance/performance-budgets.md) | Senior | SLA, performance budgets |
| [`Performance/performance-fundamentals.md`](Performance/performance-fundamentals.md) | Junior/Middle | Performance basics |
| [`AspNetCore/pipeline-middleware.md`](AspNetCore/pipeline-middleware.md) | Middle/Senior | Request pipeline, custom middleware |
| [`SQL/postgresql-deep.md`](SQL/postgresql-deep.md) | Senior | JSONB, RLS, advanced PG features |
| [`Infrastructure/project-setup.md`](Infrastructure/project-setup.md) | Middle | csproj, Directory.Build.props, packaging |

### Q

| Файл | Уровень | Описание |
|------|---------|----------|
| [`EFCore/queries-performance.md`](EFCore/queries-performance.md) | Senior | N+1, Include, projections |

### R

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Architecture/real-world-scenarios.md`](Architecture/real-world-scenarios.md) ⭐ | Senior | 18 case studies: меню → e-commerce → HFT |
| [`Quality/refactoring.md`](Quality/refactoring.md) | Senior | Refactoring techniques (Fowler-style) |
| [`CSharp/reflection-expression-trees.md`](CSharp/reflection-expression-trees.md) | Senior | Reflection, expression trees, dynamic |
| [`EFCore/relationships.md`](EFCore/relationships.md) | Middle | One-to-many, many-to-many, owned types |
| [`AspNetCore/resilience.md`](AspNetCore/resilience.md) | Senior | Polly, retry, circuit breaker |
| [`Snippets/result-pattern.md`](Snippets/result-pattern.md) | Middle | Result\<T, E\> snippet |

### S

| Файл | Уровень | Описание |
|------|---------|----------|
| [`AspNetCore/security-practices.md`](AspNetCore/security-practices.md) | Senior | OWASP, CSP, secure headers |
| [`Infrastructure/semantic-kernel.md`](Infrastructure/semantic-kernel.md) | Senior | Microsoft Semantic Kernel |
| [`AspNetCore/signalr.md`](AspNetCore/signalr.md) | Senior | SignalR, WebSockets, real-time |
| [`Architecture/solid.md`](Architecture/solid.md) | Senior | SOLID + DRY/KISS/YAGNI |
| [`CSharp/source-generators.md`](CSharp/source-generators.md) | Senior | Source generators (.NET 5+) |
| [`Runtime/span-layout.md`](Runtime/span-layout.md) | Senior | Span\<T\>, Memory\<T\>, ref struct |
| [`SQL/sql-basics.md`](SQL/sql-basics.md) | Junior/Middle | JOIN, transactions, isolation |
| [`Quality/static-analysis.md`](Quality/static-analysis.md) | Senior | Roslyn analyzers, .editorconfig |
| [`CSharp/strings-regex.md`](CSharp/strings-regex.md) | Junior/Senior | Strings, Regex, performance |
| [`Architecture/system-design.md`](Architecture/system-design.md) | Senior | System design process & interview |

### T

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Testing/testing.md`](Testing/testing.md) | Senior | xUnit, TUnit, TestContainers stack |
| [`Testing/testing-fundamentals.md`](Testing/testing-fundamentals.md) | Junior/Senior | Test pyramid, FIRST, TDD |
| [`Runtime/threading-basics.md`](Runtime/threading-basics.md) | Middle | Thread, ThreadPool, TPL, Parallel |
| [`CSharp/tuples-deconstruction.md`](CSharp/tuples-deconstruction.md) | Junior/Middle | ValueTuple, deconstruction |
| [`CSharp/types-and-memory.md`](CSharp/types-and-memory.md) | Senior | Value vs reference, boxing, struct |

### W

| Файл | Уровень | Описание |
|------|---------|----------|
| [`Architecture/webai-csharp-architecture.md`](Architecture/webai-csharp-architecture.md) | Senior | WebAI / [anonymized] architecture (specific project) |
| [`Snippets/wpf-viewmodel.md`](Snippets/wpf-viewmodel.md) | Middle | WPF MVVM ViewModel pattern |
| [`Infrastructure/wpf-production.md`](Infrastructure/wpf-production.md) | Senior | WPF production deployment |

---

## 📊 По уровню

### 🌱 Junior (12 файлов) — основы

Стартовые темы для тех кто только начинает:

- [`CSharp/csharp-basics.md`](CSharp/csharp-basics.md) — стартовая точка
- [`CSharp/csharp-vs-other-langs.md`](CSharp/csharp-vs-other-langs.md) — если знаешь Java/Python
- [`SQL/sql-basics.md`](SQL/sql-basics.md) — базовый SQL
- [`Quality/clean-code.md`](Quality/clean-code.md) — принципы (universal)
- [`Quality/code-review.md`](Quality/code-review.md) — для тех кто на review (universal)
- [`Performance/performance-fundamentals.md`](Performance/performance-fundamentals.md) — что такое производительность
- [`Testing/testing-fundamentals.md`](Testing/testing-fundamentals.md) — pyramid, FIRST
- [`LearningPath/02_junior-to-middle.md`](LearningPath/02_junior-to-middle.md) — roadmap

### 🌿 Junior to Middle (12 файлов) — daily work

Темы которые встречаются ежедневно у Middle разработчика:

- [`CSharp/datetime-timezones.md`](CSharp/datetime-timezones.md)
- [`CSharp/strings-regex.md`](CSharp/strings-regex.md)
- [`CSharp/enums-flags.md`](CSharp/enums-flags.md)
- [`CSharp/extension-methods.md`](CSharp/extension-methods.md)
- [`CSharp/iterators-yield.md`](CSharp/iterators-yield.md)
- [`CSharp/tuples-deconstruction.md`](CSharp/tuples-deconstruction.md)
- [`CSharp/io-streams.md`](CSharp/io-streams.md)
- [`CSharp/equality-comparison.md`](CSharp/equality-comparison.md)
- [`CSharp/dispose-pattern.md`](CSharp/dispose-pattern.md)
- [`CSharp/nullable-types.md`](CSharp/nullable-types.md)
- [`CSharp/attributes-metadata.md`](CSharp/attributes-metadata.md)
- [`CSharp/indexers-operators.md`](CSharp/indexers-operators.md)

### 🌳 Middle (примерно 30 файлов) — production work

Темы для самостоятельной работы Middle:

**C# / Runtime:**
- [`CSharp/oop.md`](CSharp/oop.md), [`CSharp/error-handling.md`](CSharp/error-handling.md), [`CSharp/collections-linq.md`](CSharp/collections-linq.md), [`CSharp/delegates-events.md`](CSharp/delegates-events.md)
- [`Runtime/threading-basics.md`](Runtime/threading-basics.md)

**Web:**
- Все [`AspNetCore/`](AspNetCore/) кроме самых advanced (native-aot, blazor-wasm)

**Data:**
- [`EFCore/basics-tracking.md`](EFCore/basics-tracking.md), [`relationships.md`](EFCore/relationships.md), [`migrations.md`](EFCore/migrations.md), [`patterns.md`](EFCore/patterns.md), [`dapper-comparison.md`](EFCore/dapper-comparison.md)

**Infrastructure:**
- [`Infrastructure/docker.md`](Infrastructure/docker.md), [`kubernetes.md`](Infrastructure/kubernetes.md), [`cicd-github-actions.md`](Infrastructure/cicd-github-actions.md), [`project-setup.md`](Infrastructure/project-setup.md)

### 🏔️ Middle to Senior — глубокие темы

- [`CSharp/generics-deep.md`](CSharp/generics-deep.md)
- [`AspNetCore/api-design.md`](AspNetCore/api-design.md), [`auth-security.md`](AspNetCore/auth-security.md), [`pipeline-middleware.md`](AspNetCore/pipeline-middleware.md), [`di-configuration.md`](AspNetCore/di-configuration.md), [`caching.md`](AspNetCore/caching.md)
- [`EFCore/queries-performance.md`](EFCore/queries-performance.md), [`concurrency.md`](EFCore/concurrency.md)

### 🏆 Senior (~37 файлов) — advanced

**C# advanced:**
- [`CSharp/async-threading.md`](CSharp/async-threading.md), [`types-and-memory.md`](CSharp/types-and-memory.md), [`modern-features.md`](CSharp/modern-features.md)
- [`functional-csharp.md`](CSharp/functional-csharp.md), [`design-patterns.md`](CSharp/design-patterns.md)
- [`reflection-expression-trees.md`](CSharp/reflection-expression-trees.md), [`source-generators.md`](CSharp/source-generators.md)
- [`csharp-language-design.md`](CSharp/csharp-language-design.md), [`csharp-vs-other-langs.md`](CSharp/csharp-vs-other-langs.md)
- [`cli-tools-scripting.md`](CSharp/cli-tools-scripting.md), [`desktop-frameworks.md`](CSharp/desktop-frameworks.md)

**Runtime:**
- Вся папка [`Runtime/`](Runtime/) кроме `threading-basics`

**Web (Senior):**
- [`AspNetCore/blazor-server.md`](AspNetCore/blazor-server.md), [`blazor-wasm.md`](AspNetCore/blazor-wasm.md), [`graphql.md`](AspNetCore/graphql.md), [`signalr.md`](AspNetCore/signalr.md), [`native-aot.md`](AspNetCore/native-aot.md), [`security-practices.md`](AspNetCore/security-practices.md), [`resilience.md`](AspNetCore/resilience.md)

**Architecture:** Вся папка [`Architecture/`](Architecture/)

**Performance:** Большинство [`Performance/`](Performance/)

**Infrastructure (Senior):**
- [`Infrastructure/observability.md`](Infrastructure/observability.md), [`messaging.md`](Infrastructure/messaging.md), [`ipc-named-pipes-grpc.md`](Infrastructure/ipc-named-pipes-grpc.md), [`llm-rag-patterns.md`](Infrastructure/llm-rag-patterns.md), [`semantic-kernel.md`](Infrastructure/semantic-kernel.md), [`wpf-production.md`](Infrastructure/wpf-production.md)

**SQL:** [`indexes-deep.md`](SQL/indexes-deep.md), [`optimization.md`](SQL/optimization.md), [`postgresql-deep.md`](SQL/postgresql-deep.md)

---

## 🏷️ По теме

### 🚀 Async / concurrency / parallelism

- [`CSharp/async-threading.md`](CSharp/async-threading.md)
- [`Runtime/threading-basics.md`](Runtime/threading-basics.md)
- [`Runtime/concurrency-atomics.md`](Runtime/concurrency-atomics.md)
- [`Performance/async-performance.md`](Performance/async-performance.md)
- [`AspNetCore/hosting-background.md`](AspNetCore/hosting-background.md)

### 🏗️ Architecture

- [`Architecture/patterns.md`](Architecture/patterns.md), [`patterns-decision-guide.md`](Architecture/patterns-decision-guide.md), [`real-world-scenarios.md`](Architecture/real-world-scenarios.md)
- [`Architecture/solid.md`](Architecture/solid.md), [`ddd.md`](Architecture/ddd.md), [`cqrs-mediatr.md`](Architecture/cqrs-mediatr.md)
- [`Architecture/distributed-systems.md`](Architecture/distributed-systems.md), [`microservices-vs-monolith.md`](Architecture/microservices-vs-monolith.md)
- [`Architecture/system-design.md`](Architecture/system-design.md), [`architecture-decisions.md`](Architecture/architecture-decisions.md), [`arch-tests.md`](Architecture/arch-tests.md)

### 💾 Data access

- Вся [`EFCore/`](EFCore/) папка
- Вся [`SQL/`](SQL/) папка
- [`AspNetCore/caching.md`](AspNetCore/caching.md)

### 🌐 Web / API

- Вся [`AspNetCore/`](AspNetCore/) папка
- [`Architecture/cqrs-mediatr.md`](Architecture/cqrs-mediatr.md)

### ⚡ Performance

- Вся [`Performance/`](Performance/) папка
- [`Runtime/gc-memory.md`](Runtime/gc-memory.md), [`span-layout.md`](Runtime/span-layout.md), [`compilation-jit.md`](Runtime/compilation-jit.md)
- [`EFCore/queries-performance.md`](EFCore/queries-performance.md)

### 🧪 Testing

- Вся [`Testing/`](Testing/) папка
- [`Architecture/arch-tests.md`](Architecture/arch-tests.md)

### 🚢 DevOps / Deploy

- [`Infrastructure/docker.md`](Infrastructure/docker.md), [`kubernetes.md`](Infrastructure/kubernetes.md), [`cicd-github-actions.md`](Infrastructure/cicd-github-actions.md)
- [`Infrastructure/observability.md`](Infrastructure/observability.md), [`project-setup.md`](Infrastructure/project-setup.md)

### 🤖 AI / LLM

- [`Infrastructure/llm-rag-patterns.md`](Infrastructure/llm-rag-patterns.md)
- [`Infrastructure/semantic-kernel.md`](Infrastructure/semantic-kernel.md)
- [`Architecture/webai-csharp-architecture.md`](Architecture/webai-csharp-architecture.md)

### 🖥️ Desktop / WPF

- [`CSharp/desktop-frameworks.md`](CSharp/desktop-frameworks.md)
- [`Infrastructure/wpf-production.md`](Infrastructure/wpf-production.md)
- [`Snippets/wpf-viewmodel.md`](Snippets/wpf-viewmodel.md)

### 🎤 Interview prep

- [`LearningPath/04_interview-prep.md`](LearningPath/04_interview-prep.md)
- [`LearningPath/09_senior-tips-cheatsheet.md`](LearningPath/09_senior-tips-cheatsheet.md)
- [`LearningPath/10_interview-behavioral.md`](LearningPath/10_interview-behavioral.md)
- [`Architecture/system-design.md`](Architecture/system-design.md)

### 📋 Patterns (cross-folder)

- [`CSharp/design-patterns.md`](CSharp/design-patterns.md) — 13 GoF
- [`CSharp/dispose-pattern.md`](CSharp/dispose-pattern.md) — IDisposable detail
- [`Architecture/patterns.md`](Architecture/patterns.md) — N-Layer, Clean, VSA
- [`Architecture/patterns-decision-guide.md`](Architecture/patterns-decision-guide.md) — выбор
- [`EFCore/patterns.md`](EFCore/patterns.md) — Repository, UoW
- [`Performance/optimization-patterns.md`](Performance/optimization-patterns.md) — perf
- [`Infrastructure/llm-rag-patterns.md`](Infrastructure/llm-rag-patterns.md) — LLM RAG
- [`Snippets/result-pattern.md`](Snippets/result-pattern.md) — Result\<T, E\>

---

[← Назад к главному README](README.md)
