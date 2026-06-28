# NET-Mastery-Hub

> **Comprehensive C# / .NET knowledge base** — from Junior fundamentals to Senior architecture and distributed systems.
>
> **162 deep-dive notes / ~6 MB**, organized by topic. Russian primary, English technical terms.

---

## 📋 Table of Contents

- [Quick Start — куда смотреть первым делом](#quick-start)
- [How this vault is organized](#how-this-vault-is-organized)
- [📁 The 12 sections](#-the-12-sections)
- [🌟 Top-20 must-read для Senior](#-top-20-must-read-для-senior)
- [🗺️ Special navigation files](#️-special-navigation-files)
- [🎯 Quick navigation — by use case](#-quick-navigation--by-use-case)
- [📐 Conventions & format](#-conventions--format)
- [📊 Stats](#-stats)
- [License](#license)

---

## Quick Start

### Куда идти первым делом?

| Ты кто | Куда идти |
|--------|-----------|
| 🌱 **Никогда не писал на C#** | [`CSharp/Junior/csharp-basics.md`](CSharp/Junior/csharp-basics.md) |
| 🌿 **Junior хочет в Middle** | [`LearningPath/02_junior-to-middle.md`](LearningPath/02_junior-to-middle.md) (3-6 месяцев план) |
| 🌳 **Middle хочет в Senior** | [`LearningPath/03_middle-to-senior.md`](LearningPath/03_middle-to-senior.md) |
| 🎤 **Готовлюсь к собесу** | [`LearningPath/04_interview-prep.md`](LearningPath/04_interview-prep.md) (1-2 недели спринт) |
| 🏗️ **Проектирую новое приложение** | [`Architecture/Middle/real-world-scenarios.md`](Architecture/Middle/real-world-scenarios.md) — 18 сценариев с решениями |
| 🤔 **Какой паттерн / архитектуру выбрать?** | [`Architecture/Middle/patterns-decision-guide.md`](Architecture/Middle/patterns-decision-guide.md) |
| 📚 **Reference / lookup** | [`INDEX.md`](INDEX.md) — полное оглавление с описаниями |

### Как использовать

1. **В Obsidian** — открой папку как vault для full backlink navigation
2. **На GitHub** — markdown render OK, ссылки работают через relative paths
3. **Локально** — VSCode + Markdown Preview

---

## How this vault is organized

Внутри тематических папок файлы разложены по уровням: `Junior/`, `Middle/`, `Senior/` (файлы уровня `All` лежат в корне папки). `LearningPath/` и `Snippets/` уровней не имеют.

```
NET-Mastery-Hub/
│
├── 🎓 LearningPath/      Roadmaps Junior → Middle → Senior + interview prep (10)
│
├── 💎 CSharp/             Сам язык — fundamentals → advanced (41 файл) ⭐
├── ⚙️  Runtime/            CLR internals, GC, JIT, threading (9)
│
├── 🌐 AspNetCore/         Web framework — auth, middleware, HttpClient, SignalR (23)
├── 💾 EFCore/             ORM — tracking, performance, patterns (13)
├── 🗄️  SQL/                SQL fundamentals + indexes + Postgres (9)
│
├── 🏛️  Architecture/       SOLID, DDD, CQRS, microservices, decision guides (16)
├── ✅ Quality/            Clean code, refactoring, code review, static analysis (5)
├── 🧪 Testing/            Unit, integration, mocking, mutation, fundamentals (5)
├── ⚡ Performance/        Profiling, optimization, caching, HFT (12)
├── 🚢 Infrastructure/     Docker, k8s, CI/CD, observability, messaging, AI (14)
│
├── 📋 Snippets/           Ready-to-copy code patterns (5)
└── 📑 INDEX.md            Полное оглавление (level + описание каждого файла)
```

---

## 📁 The 12 sections

> Каждая папка имеет свой `README.md` с детальной навигацией внутри. Полный список файлов с описаниями — в [`INDEX.md`](INDEX.md).

### 🎓 [LearningPath](LearningPath/) — где начать (10 файлов)

Roadmaps для роста: какие темы изучать в каком порядке + interview prep + reading list + топ-7 case studies.

→ [Подробнее в LearningPath/README.md](LearningPath/README.md)

### 💎 [CSharp](CSharp/) — язык (41 файл / ~2.4 MB) ⭐ самая большая

Полное покрытие C# языка от basics до Senior:

- **Junior:** csharp-basics, oop, collections-linq, strings-regex, datetime-timezones, enums-flags, tuples-deconstruction, anonymous-types, extension-methods, iterators-yield, naming-conventions, debugging-basics, dotnet-cli-getting-started
- **Middle:** modern-features, error-handling, nullable-types, generics-deep, delegates-events, equality-comparison, attributes-metadata, indexers-operators, dispose-pattern, io-streams, serialization-deep, bcl-essentials, numeric-types-math, keywords-reference
- **Senior:** async-threading, types-and-memory, functional-csharp, design-patterns, gof-patterns-extended, reflection-expression-trees, source-generators, memory-pooling, unsafe-pointers, fenwick-bit, csharp-language-design, csharp-vs-other-langs, cli-tools-scripting, desktop-frameworks

→ [Подробнее в CSharp/README.md](CSharp/README.md)

### ⚙️ [Runtime](Runtime/) — CLR internals (9 файлов)

- **Junior:** runtime-basics, memory-stack-heap
- **Middle:** threading-basics
- **Senior:** gc-memory, compilation-jit, concurrency-atomics, span-layout, interop-pinvoke, diagnostics-tools

→ [Подробнее в Runtime/README.md](Runtime/README.md)

### 🌐 [AspNetCore](AspNetCore/) — web framework (23 файла / ~640 KB)

- **Junior:** http-fundamentals
- **Middle:** aspnet-controllers-routing, aspnet-dependency-injection-deep, aspnet-error-handling, aspnet-rate-limiting, fluent-validation, http-client-resilience, object-mapping
- **Senior:** api-design, auth-security, pipeline-middleware, di-configuration, caching, logging-observability, hosting-background, resilience, security-practices, signalr, graphql, blazor-server, blazor-wasm, native-aot, kestrel-as-raw-host

→ [Подробнее в AspNetCore/README.md](AspNetCore/README.md)

### 💾 [EFCore](EFCore/) — ORM (13 файлов)

- **Junior:** ef-basics, ef-crud-queries
- **Middle:** ef-loading-strategies, ef-transactions-concurrency, ef-bulk-operations, ef-value-converters, dapper-comparison
- **Senior:** basics-tracking, queries-performance, relationships, migrations, concurrency, ef-patterns

→ [Подробнее в EFCore/README.md](EFCore/README.md)

### 🗄️ [SQL](SQL/) — relational DB (9 файлов)

- **Junior:** sql-basics · **Middle:** indexes-deep
- **Senior:** optimization, postgresql-deep, mvcc-and-locking, postgres-functions-triggers, sql-security, zero-downtime-migrations, eav-flexible-store-indexing

→ [Подробнее в SQL/README.md](SQL/README.md)

### 🏛️ [Architecture](Architecture/) — patterns & systems (16 файлов / ~520 KB)

- **Junior:** architecture-basics
- **Middle:** **patterns-decision-guide** (какой паттерн под задачу), **real-world-scenarios** (18 case studies), microservices-vs-monolith
- **Senior:** architecture-patterns (N-Layer/Clean/VSA), agent-safe-architecture, solid, ddd, cqrs-mediatr, distributed-systems, system-design, architecture-decisions, arch-tests, twelve-factor-app, eip-content-based-router, webai-csharp-architecture

→ [Подробнее в Architecture/README.md](Architecture/README.md)

### ✅ [Quality](Quality/) — clean code (5 файлов)

`clean-code` (Junior basics), `refactoring` (Middle), `code-quality` + `static-analysis` (Senior tools), `code-review` (All — в корне папки)

> ⚠️ `clean-code.md` ≠ `code-quality.md` — это разные уровни одной темы (Junior принципы vs Senior tooling).

→ [Подробнее в Quality/README.md](Quality/README.md)

### 🧪 [Testing](Testing/) — testing strategies (5 файлов)

`testing-fundamentals` (Junior), `mocking-strategies` (Middle), `testing` + `integration-testing` + `mutation-load-testing` (Senior)

> ⚠️ `testing.md` ≠ `testing-fundamentals.md` — это разные уровни (Senior tools vs Junior basics).

→ [Подробнее в Testing/README.md](Testing/README.md)

### ⚡ [Performance](Performance/) — производительность (12 файлов)

- **Junior:** performance-fundamentals
- **Middle:** optimization-patterns, caching-strategies, async-performance, lazy-eager-loading, bottleneck-analysis
- **Senior:** performance (BenchmarkDotNet/PerfView), memory-profiling, hft-low-latency, capacity-planning, performance-budgets, threadpool-starvation-hill-climbing

> ⚠️ `performance.md` ≠ `performance-fundamentals.md` — Senior tools vs Junior basics.

→ [Подробнее в Performance/README.md](Performance/README.md)

### 🚢 [Infrastructure](Infrastructure/) — DevOps & deploy (14 файлов / ~470 KB)

- **Junior:** docker-for-dev, project-setup-basics
- **Middle:** kubernetes, cicd-github-actions
- **Senior:** docker, observability, messaging, api-gateway, nosql-databases, ipc-named-pipes-grpc, project-setup, wpf-production, llm-rag-patterns, semantic-kernel

→ [Подробнее в Infrastructure/README.md](Infrastructure/README.md)

### 📋 [Snippets](Snippets/) — ready-to-copy (5 файлов)

`crud-example`, `efcore-queries`, `mediatr-handlers`, `result-pattern`, `wpf-viewmodel`

→ [Подробнее в Snippets/README.md](Snippets/README.md)

---

## 🌟 Top-20 must-read для Senior

Если время ограничено — это самые ценные файлы:

### Язык + Runtime (must-know internals)

1. [`CSharp/Senior/async-threading.md`](CSharp/Senior/async-threading.md) — Task, async/await под капотом
2. [`CSharp/Senior/types-and-memory.md`](CSharp/Senior/types-and-memory.md) — value vs reference, boxing, struct internals
3. [`Runtime/Senior/gc-memory.md`](Runtime/Senior/gc-memory.md) — GC generations, regions, leaks
4. [`Runtime/Senior/span-layout.md`](Runtime/Senior/span-layout.md) — Span\<T\>, ref struct, performance
5. [`CSharp/Middle/generics-deep.md`](CSharp/Middle/generics-deep.md) — variance, INumber\<T\>, .NET 7+

### Data + EF Core

6. [`EFCore/Senior/basics-tracking.md`](EFCore/Senior/basics-tracking.md) — Change Tracker, AsNoTracking
7. [`EFCore/Senior/queries-performance.md`](EFCore/Senior/queries-performance.md) — N+1, projections
8. [`EFCore/Middle/dapper-comparison.md`](EFCore/Middle/dapper-comparison.md) — когда EF, когда Dapper
9. [`SQL/Middle/indexes-deep.md`](SQL/Middle/indexes-deep.md) — query plans, B-tree internals

### Web framework

10. [`AspNetCore/Senior/pipeline-middleware.md`](AspNetCore/Senior/pipeline-middleware.md) — request pipeline
11. [`AspNetCore/Senior/auth-security.md`](AspNetCore/Senior/auth-security.md) — JWT, OAuth, OIDC
12. [`AspNetCore/Middle/http-client-resilience.md`](AspNetCore/Middle/http-client-resilience.md) — HttpClient, IHttpClientFactory, retry/breaker

### Архитектура

13. [`Architecture/Middle/patterns-decision-guide.md`](Architecture/Middle/patterns-decision-guide.md) ⭐ — какой паттерн под какую задачу
14. [`Architecture/Middle/real-world-scenarios.md`](Architecture/Middle/real-world-scenarios.md) ⭐ — 18 case studies
15. [`Architecture/Senior/architecture-patterns.md`](Architecture/Senior/architecture-patterns.md) — N-Layer / Clean / VSA / Hybrid
16. [`Architecture/Middle/microservices-vs-monolith.md`](Architecture/Middle/microservices-vs-monolith.md) — когда выбирать
17. [`Architecture/Senior/ddd.md`](Architecture/Senior/ddd.md) — Bounded Contexts, Aggregates

### Quality + Testing + Infrastructure

18. [`Testing/Junior/testing-fundamentals.md`](Testing/Junior/testing-fundamentals.md) — pyramid, FIRST principles
19. [`Quality/code-review.md`](Quality/code-review.md) — process & culture
20. [`Infrastructure/Senior/observability.md`](Infrastructure/Senior/observability.md) — OpenTelemetry, logs/metrics/traces

---

## 🗺️ Special navigation files

Эти файлы — **integrating hubs**, связывающие разные части vault:

| Файл | Зачем |
|------|-------|
| [`INDEX.md`](INDEX.md) | Полное оглавление: каждый файл + уровень + описание |
| [`Architecture/Middle/patterns-decision-guide.md`](Architecture/Middle/patterns-decision-guide.md) | Под какую задачу — какой паттерн / архитектура |
| [`Architecture/Middle/real-world-scenarios.md`](Architecture/Middle/real-world-scenarios.md) | 18 конкретных сценариев: меню, корзина, e-commerce, HFT, IoT |
| [`LearningPath/00_overview.md`](LearningPath/00_overview.md) | Главная навигация по learning path |
| [`LearningPath/05_topics-by-priority.md`](LearningPath/05_topics-by-priority.md) | Темы по приоритету value/effort |

---

## 🎯 Quick navigation — by use case

### "Я разрабатываю..."

| Что | Куда |
|-----|------|
| Internal admin tool | [`real-world-scenarios.md` → сценарий 11](Architecture/Middle/real-world-scenarios.md) |
| Малый интернет-магазин | [`real-world-scenarios.md` → сценарий 12](Architecture/Middle/real-world-scenarios.md) |
| Крупный e-commerce | [`real-world-scenarios.md` → сценарий 13](Architecture/Middle/real-world-scenarios.md) |
| Контент-портал / CMS | [`real-world-scenarios.md` → сценарий 14](Architecture/Middle/real-world-scenarios.md) |
| SaaS B2B мульти-tenant | [`real-world-scenarios.md` → сценарий 15](Architecture/Middle/real-world-scenarios.md) |
| HFT / Trading | [`Performance/Senior/hft-low-latency.md`](Performance/Senior/hft-low-latency.md) |
| IoT платформа | [`real-world-scenarios.md` → сценарий 17](Architecture/Middle/real-world-scenarios.md) |
| Desktop app (WPF) | [`CSharp/Senior/desktop-frameworks.md`](CSharp/Senior/desktop-frameworks.md) |

### "Мне нужно решить..."

| Проблема | Куда |
|----------|------|
| N+1 query в EF | [`EFCore/Senior/queries-performance.md`](EFCore/Senior/queries-performance.md) |
| Memory leak | [`Runtime/Senior/diagnostics-tools.md`](Runtime/Senior/diagnostics-tools.md) + [`Runtime/Senior/gc-memory.md`](Runtime/Senior/gc-memory.md) |
| Slow database | [`SQL/Senior/optimization.md`](SQL/Senior/optimization.md) + [`SQL/Middle/indexes-deep.md`](SQL/Middle/indexes-deep.md) |
| ThreadPool starvation | [`CSharp/Senior/async-threading.md`](CSharp/Senior/async-threading.md) + [`Runtime/Middle/threading-basics.md`](Runtime/Middle/threading-basics.md) |
| Сетевые вызовы падают / ретраи | [`AspNetCore/Middle/http-client-resilience.md`](AspNetCore/Middle/http-client-resilience.md) + [`AspNetCore/Senior/resilience.md`](AspNetCore/Senior/resilience.md) |
| JSON-контракт / сериализация | [`CSharp/Middle/serialization-deep.md`](CSharp/Middle/serialization-deep.md) |
| Distributed transactions | [`Architecture/Senior/distributed-systems.md`](Architecture/Senior/distributed-systems.md) |
| API versioning | [`AspNetCore/Senior/api-design.md`](AspNetCore/Senior/api-design.md) |
| Caching strategy | [`AspNetCore/Senior/caching.md`](AspNetCore/Senior/caching.md) + [`Performance/Middle/caching-strategies.md`](Performance/Middle/caching-strategies.md) |
| Auth / Identity | [`AspNetCore/Senior/auth-security.md`](AspNetCore/Senior/auth-security.md) |

### "Готовлюсь к интервью на..."

| Уровень | Куда |
|---------|------|
| Junior C# | [`LearningPath/02_junior-to-middle.md`](LearningPath/02_junior-to-middle.md) |
| Middle .NET | [`LearningPath/03_middle-to-senior.md`](LearningPath/03_middle-to-senior.md) + Top-20 список выше |
| Senior .NET | Top-20 список + [`Architecture/`](Architecture/) полностью |
| Behavioral / soft | [`LearningPath/10_interview-behavioral.md`](LearningPath/10_interview-behavioral.md) |
| Final sprint (1 неделя) | [`LearningPath/04_interview-prep.md`](LearningPath/04_interview-prep.md) + [`LearningPath/09_senior-tips-cheatsheet.md`](LearningPath/09_senior-tips-cheatsheet.md) |

---

## 📐 Conventions & format

### Каждый файл follows:

```markdown
---
tags: [topic1, topic2, level]
level: Junior | Middle | Senior | All
date: YYYY-MM-DD
---

# Topic Name

> Tagline — что и зачем (1-2 строки).

## Что это, зачем и когда

## 1. Базовая концепция
## 2. ... (тематические секции с примерами)
## N. Common Pitfalls
## N+1. Best Practices
## Cheat sheet
## Decision tree (если применимо)
## См. также (cross-references)
## Reading list (books, blogs, docs)
```

### Cross-references

Obsidian-style по **имени файла без пути**: `[[file-name|display name]]`. Пути в wikilinks не используются — файлы переезжают между level-папками, имена стабильны.

### Code blocks

ВСЕГДА blank line между preceding header и opening triple-backticks.

### Tags

Каждый файл имеет tags в frontmatter. Поиск по тегам:

```bash
# Все Junior темы
grep -r "level: Junior" --include="*.md" -l

# Все про async
grep -r "tags:.*async" --include="*.md" -l
```

---

## 📊 Stats

| | Value |
|---|---|
| **Total files** | 162 content notes (+ 12 section READMEs + INDEX) |
| **Total size** | ~6.3 MB |
| **Coverage** | Junior → Senior+ |
| **Largest folder** | CSharp (41 files / ~2.4 MB) |
| **Largest file** | `CSharp/Junior/tuples-deconstruction.md` (~120 KB) |
| **Language** | Russian (primary), English (technical terms) |
| **Last major update** | 2026-06-28 |

### По уровню

```
Junior:            25 файлов  (basics, fundamentals, daily work)
Junior to Middle:   1 файл
Middle:            29 файлов  (generics, EF, HttpClient, serialization, BCL, testing)
Middle to Senior:  13 файлов  (паттерны, deep topics)
Senior:            82 файла   (architecture, runtime, performance, advanced)
All / без уровня:  12 файлов  (LearningPath, Snippets, code-review)
```

### По категориям

```
Language:        41 file   (CSharp/)
Runtime:          9 files  (Runtime/)
Web framework:   23 files  (AspNetCore/)
Data:            22 files  (EFCore/ + SQL/)
Architecture:    16 files  (Architecture/)
Quality:          5 files  (Quality/)
Testing:          5 files  (Testing/)
Performance:     12 files  (Performance/)
Infrastructure:  14 files  (Infrastructure/)
Learning:        10 files  (LearningPath/)
Snippets:         5 files  (Snippets/)
```

---

## License

Personal knowledge base — no formal license. Feel free to learn from it; don't republish wholesale.

If you find errors or have suggestions, [open an issue](https://github.com/valinerosgordov/NET-Mastery-Hub/issues).
