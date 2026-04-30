# NET-Mastery-Hub

> **Comprehensive C# / .NET knowledge base** — from Junior fundamentals to Senior architecture and distributed systems.
>
> **119 deep-dive notes / ~3.2 MB**, organized by topic. Russian primary, English technical terms.

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
| 🌱 **Никогда не писал на C#** | [`CSharp/csharp-basics.md`](CSharp/csharp-basics.md) |
| 🌿 **Junior хочет в Middle** | [`LearningPath/02_junior-to-middle.md`](LearningPath/02_junior-to-middle.md) (3-6 месяцев план) |
| 🌳 **Middle хочет в Senior** | [`LearningPath/03_middle-to-senior.md`](LearningPath/03_middle-to-senior.md) |
| 🎤 **Готовлюсь к собесу** | [`LearningPath/04_interview-prep.md`](LearningPath/04_interview-prep.md) (1-2 недели спринт) |
| 🏗️ **Проектирую новое приложение** | [`Architecture/real-world-scenarios.md`](Architecture/real-world-scenarios.md) — 18 сценариев с решениями |
| 🤔 **Какой паттерн / архитектуру выбрать?** | [`Architecture/patterns-decision-guide.md`](Architecture/patterns-decision-guide.md) |
| 📚 **Reference / lookup** | [Полное оглавление ниже](#-the-12-sections) |

### Как использовать

1. **В Obsidian** — открой папку как vault для full backlink navigation
2. **На GitHub** — markdown render OK, ссылки работают через relative paths
3. **Локально** — VSCode + Markdown Preview

---

## How this vault is organized

```
NET-Mastery-Hub/
│
├── 🎓 LearningPath/      Roadmaps Junior → Middle → Senior + interview prep
│
├── 💎 CSharp/             Сам язык — fundamentals → advanced (29 файлов)
├── ⚙️  Runtime/            CLR internals, GC, JIT, threading (7 файлов)
│
├── 🌐 AspNetCore/         Web framework — auth, middleware, GraphQL, SignalR (14)
├── 💾 EFCore/             ORM — basics, performance, patterns (7)
├── 🗄️  SQL/                SQL fundamentals + indexes + Postgres (4)
│
├── 🏛️  Architecture/       SOLID, DDD, CQRS, microservices, decision guides (12)
├── ✅ Quality/            Clean code, refactoring, code review, static analysis (5)
├── 🧪 Testing/            Unit, integration, mocking, mutation, fundamentals (5)
├── ⚡ Performance/        Profiling, optimization, caching, HFT (11)
├── 🚢 Infrastructure/     Docker, k8s, CI/CD, observability, messaging (10)
│
├── 📋 Snippets/           Ready-to-copy code patterns (5)
├── 📜 Scripts/            Maintenance PowerShell scripts
└── 📖 _changelog.md       What changed when
```

---

## 📁 The 12 sections

> Каждая папка имеет свой `README.md` с детальной навигацией внутри.

### 🎓 [LearningPath](LearningPath/) — где начать (8 файлов)

Roadmaps для роста: какие темы изучать в каком порядке + interview prep.

→ [Подробнее в LearningPath/README.md](LearningPath/README.md)

### 💎 [CSharp](CSharp/) — язык (29 файлов / 933 KB) ⭐ самая большая

Полное покрытие C# языка от basics до Senior:

- **Junior:** csharp-basics, datetime-timezones, strings-regex, enums-flags, tuples-deconstruction
- **Middle:** oop, modern-features, async-threading, collections-linq, error-handling, nullable-types, io-streams, equality-comparison, attributes-metadata, indexers-operators, dispose-pattern, extension-methods, iterators-yield, delegates-events
- **Senior:** generics-deep, functional-csharp, design-patterns, types-and-memory, reflection-expression-trees, source-generators, csharp-language-design, csharp-vs-other-langs, cli-tools-scripting, desktop-frameworks

→ [Подробнее в CSharp/README.md](CSharp/README.md)

### ⚙️ [Runtime](Runtime/) — CLR internals (7 файлов)

`gc-memory`, `compilation-jit`, `concurrency-atomics`, `span-layout`, `threading-basics`, `interop-pinvoke`, `diagnostics-tools`

→ [Подробнее в Runtime/README.md](Runtime/README.md)

### 🌐 [AspNetCore](AspNetCore/) — web framework (14 файлов / 392 KB)

`api-design`, `auth-security`, `pipeline-middleware`, `di-configuration`, `caching`, `logging-observability`, `hosting-background`, `resilience`, `security-practices`, `signalr`, `graphql`, `blazor-server`, `blazor-wasm`, `native-aot`

→ [Подробнее в AspNetCore/README.md](AspNetCore/README.md)

### 💾 [EFCore](EFCore/) — ORM (7 файлов)

`basics-tracking`, `queries-performance`, `relationships`, `migrations`, `concurrency`, `patterns`, `dapper-comparison`

→ [Подробнее в EFCore/README.md](EFCore/README.md)

### 🗄️ [SQL](SQL/) — relational DB (4 файла)

`sql-basics`, `indexes-deep`, `optimization`, `postgresql-deep`

→ [Подробнее в SQL/README.md](SQL/README.md)

### 🏛️ [Architecture](Architecture/) — patterns & systems (12 файлов / 385 KB)

`patterns` (N-Layer/Clean/VSA), `solid`, `ddd`, `cqrs-mediatr`, `distributed-systems`, `microservices-vs-monolith`, `system-design`, `architecture-decisions`, `arch-tests`, **`patterns-decision-guide`** (новый!), **`real-world-scenarios`** (18 case studies), `webai-csharp-architecture`

→ [Подробнее в Architecture/README.md](Architecture/README.md)

### ✅ [Quality](Quality/) — clean code (5 файлов)

`clean-code` (Junior basics), `code-quality` (Senior tools — analyzers/SonarCloud), `refactoring`, `code-review`, `static-analysis`

> ⚠️ `clean-code.md` ≠ `code-quality.md` — это разные уровни одной темы (Junior принципы vs Senior tooling).

→ [Подробнее в Quality/README.md](Quality/README.md)

### 🧪 [Testing](Testing/) — testing strategies (5 файлов)

`testing-fundamentals` (Junior basics), `testing` (Senior — xUnit/TUnit/TestContainers), `integration-testing`, `mocking-strategies`, `mutation-load-testing`

> ⚠️ `testing.md` ≠ `testing-fundamentals.md` — это разные уровни (Senior tools vs Junior basics).

→ [Подробнее в Testing/README.md](Testing/README.md)

### ⚡ [Performance](Performance/) — производительность (11 файлов)

`performance-fundamentals` (Junior basics), `performance` (Senior — BenchmarkDotNet/PerfView), `optimization-patterns`, `caching-strategies`, `memory-profiling`, `async-performance`, `lazy-eager-loading`, `hft-low-latency`, `bottleneck-analysis`, `capacity-planning`, `performance-budgets`

> ⚠️ `performance.md` ≠ `performance-fundamentals.md` — Senior tools vs Junior basics.

→ [Подробнее в Performance/README.md](Performance/README.md)

### 🚢 [Infrastructure](Infrastructure/) — DevOps & deploy (10 файлов / 331 KB)

`docker`, `kubernetes`, `cicd-github-actions`, `observability`, `messaging`, `project-setup`, `ipc-named-pipes-grpc`, `wpf-production`, `llm-rag-patterns`, `semantic-kernel`

→ [Подробнее в Infrastructure/README.md](Infrastructure/README.md)

### 📋 [Snippets](Snippets/) — ready-to-copy (5 файлов)

`crud-example`, `efcore-queries`, `mediatr-handlers`, `result-pattern`, `wpf-viewmodel`

→ [Подробнее в Snippets/README.md](Snippets/README.md)

### 📜 [Scripts](Scripts/) — maintenance

`format_audit.ps1`, `fix_formatting.ps1` — для автомейнтенанса формата кода в `.md` файлах.

```powershell
# Запускать из корня vault
& "Scripts/format_audit.ps1"
& "Scripts/fix_formatting.ps1"
```

---

## 🌟 Top-20 must-read для Senior

Если время ограничено — это самые ценные файлы:

### Язык + Runtime (must-know internals)

1. [`CSharp/async-threading.md`](CSharp/async-threading.md) — Task, async/await под капотом (58 KB)
2. [`CSharp/types-and-memory.md`](CSharp/types-and-memory.md) — value vs reference, boxing, struct internals (53 KB)
3. [`Runtime/gc-memory.md`](Runtime/gc-memory.md) — GC generations, regions, leaks (56 KB)
4. [`Runtime/span-layout.md`](Runtime/span-layout.md) — Span\<T\>, ref struct, performance
5. [`CSharp/generics-deep.md`](CSharp/generics-deep.md) — variance, INumber\<T\>, .NET 7+

### Data + EF Core

6. [`EFCore/basics-tracking.md`](EFCore/basics-tracking.md) — Change Tracker, AsNoTracking
7. [`EFCore/queries-performance.md`](EFCore/queries-performance.md) — N+1, projections
8. [`EFCore/dapper-comparison.md`](EFCore/dapper-comparison.md) — когда EF, когда Dapper
9. [`SQL/indexes-deep.md`](SQL/indexes-deep.md) — query plans, B-tree internals

### Web framework

10. [`AspNetCore/pipeline-middleware.md`](AspNetCore/pipeline-middleware.md) — request pipeline
11. [`AspNetCore/auth-security.md`](AspNetCore/auth-security.md) — JWT, OAuth, OIDC

### Архитектура

12. [`Architecture/patterns-decision-guide.md`](Architecture/patterns-decision-guide.md) ⭐ — какой паттерн под какую задачу
13. [`Architecture/real-world-scenarios.md`](Architecture/real-world-scenarios.md) ⭐ — 18 case studies
14. [`Architecture/patterns.md`](Architecture/patterns.md) — N-Layer / Clean / VSA / Hybrid
15. [`Architecture/microservices-vs-monolith.md`](Architecture/microservices-vs-monolith.md) — когда выбирать
16. [`Architecture/ddd.md`](Architecture/ddd.md) — Bounded Contexts, Aggregates

### Quality + Testing

17. [`Testing/testing-fundamentals.md`](Testing/testing-fundamentals.md) — pyramid, FIRST principles
18. [`Quality/clean-code.md`](Quality/clean-code.md) — fundamentals
19. [`Quality/code-review.md`](Quality/code-review.md) — process & culture

### Infrastructure

20. [`Infrastructure/observability.md`](Infrastructure/observability.md) — OpenTelemetry, logs/metrics/traces

---

## 🗺️ Special navigation files

Эти файлы — **integrating hubs**, связывающие разные части vault:

| Файл | Зачем |
|------|-------|
| [`Architecture/patterns-decision-guide.md`](Architecture/patterns-decision-guide.md) | Под какую задачу — какой паттерн / архитектура |
| [`Architecture/real-world-scenarios.md`](Architecture/real-world-scenarios.md) | 18 конкретных сценариев: меню, корзина, e-commerce, HFT, IoT |
| [`LearningPath/00_overview.md`](LearningPath/00_overview.md) | Главная навигация по learning path |
| [`LearningPath/05_topics-by-priority.md`](LearningPath/05_topics-by-priority.md) | Темы по приоритету value/effort |

---

## 🎯 Quick navigation — by use case

### "Я разрабатываю..."

| Что | Куда |
|-----|------|
| Internal admin tool | [`Architecture/real-world-scenarios.md#сценарий-11`](Architecture/real-world-scenarios.md) |
| Малый интернет-магазин | [`Architecture/real-world-scenarios.md#сценарий-12`](Architecture/real-world-scenarios.md) |
| Крупный e-commerce | [`Architecture/real-world-scenarios.md#сценарий-13`](Architecture/real-world-scenarios.md) |
| Контент-портал / CMS | [`Architecture/real-world-scenarios.md#сценарий-14`](Architecture/real-world-scenarios.md) |
| SaaS B2B мульти-tenant | [`Architecture/real-world-scenarios.md#сценарий-15`](Architecture/real-world-scenarios.md) |
| HFT / Trading | [`Performance/hft-low-latency.md`](Performance/hft-low-latency.md) |
| IoT платформа | [`Architecture/real-world-scenarios.md#сценарий-17`](Architecture/real-world-scenarios.md) |
| Desktop app (WPF) | [`CSharp/desktop-frameworks.md`](CSharp/desktop-frameworks.md) |

### "Мне нужно решить..."

| Проблема | Куда |
|----------|------|
| N+1 query в EF | [`EFCore/queries-performance.md`](EFCore/queries-performance.md) |
| Memory leak | [`Runtime/diagnostics-tools.md`](Runtime/diagnostics-tools.md) + [`Runtime/gc-memory.md`](Runtime/gc-memory.md) |
| Slow database | [`SQL/optimization.md`](SQL/optimization.md) + [`SQL/indexes-deep.md`](SQL/indexes-deep.md) |
| ThreadPool starvation | [`CSharp/async-threading.md`](CSharp/async-threading.md) + [`Runtime/threading-basics.md`](Runtime/threading-basics.md) |
| Distributed transactions | [`Architecture/distributed-systems.md`](Architecture/distributed-systems.md) |
| API versioning | [`AspNetCore/api-design.md`](AspNetCore/api-design.md) |
| Caching strategy | [`AspNetCore/caching.md`](AspNetCore/caching.md) + [`Performance/caching-strategies.md`](Performance/caching-strategies.md) |
| Auth / Identity | [`AspNetCore/auth-security.md`](AspNetCore/auth-security.md) |

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
level: Junior | Middle | Senior | Junior to Senior
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

Используется Obsidian-style: `[[folder/file|display name]]`. На GitHub — рендерятся как обычные ссылки.

### Code blocks

ВСЕГДА blank line между preceding header и opening triple-backticks. Скрипт `Scripts/fix_formatting.ps1` чинит автоматом.

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
| **Total files** | 119 |
| **Total size** | ~3.2 MB |
| **Coverage** | Junior → Senior+ |
| **Largest folder** | CSharp (29 files / 933 KB) |
| **Largest file** | `CSharp/async-threading.md` (58 KB) |
| **Language** | Russian (primary), English (technical terms) |
| **Last major update** | 2026-04-30 |

### По уровню

```
Junior:           ~12 файлов  (basics, fundamentals, daily work)
Middle:           ~45 файлов  (oop, async, EF, ASP.NET, testing)
Middle to Senior: ~25 файлов  (patterns, generics, deep topics)
Senior:           ~37 файлов  (architecture, runtime, performance, advanced)
```

### По категориям

```
Language:        29 files  (CSharp/)
Runtime:          7 files  (Runtime/)
Web framework:   14 files  (AspNetCore/)
Data:            11 files  (EFCore/ + SQL/)
Architecture:    12 files  (Architecture/)
Quality:          5 files  (Quality/)
Testing:          5 files  (Testing/)
Performance:     11 files  (Performance/)
Infrastructure: 10 files  (Infrastructure/)
Learning:         8 files  (LearningPath/)
Snippets:         5 files  (Snippets/)
```

---

## License

Personal knowledge base — no formal license. Feel free to learn from it; don't republish wholesale.

If you find errors or have suggestions, [open an issue](https://github.com/valinerosgordov/NET-Mastery-Hub/issues).
