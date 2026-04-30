# NET-Mastery-Hub

> **Comprehensive C# / .NET knowledge base** вЂ” from Junior fundamentals to Senior architecture and distributed systems.
>
> **142 deep-dive notes / ~3.6 MB**, organized by topic. Russian primary, English technical terms.

---

## рџ“‹ Table of Contents

- [Quick Start вЂ” РєСѓРґР° СЃРјРѕС‚СЂРµС‚СЊ РїРµСЂРІС‹Рј РґРµР»РѕРј](#quick-start)
- [How this vault is organized](#how-this-vault-is-organized)
- [рџ“Ѓ The 12 sections](#-the-12-sections)
- [рџЊџ Top-20 must-read РґР»СЏ Senior](#-top-20-must-read-РґР»СЏ-senior)
- [рџ—єпёЏ Special navigation files](#пёЏ-special-navigation-files)
- [рџЋЇ Quick navigation вЂ” by use case](#-quick-navigation--by-use-case)
- [рџ“ђ Conventions & format](#-conventions--format)
- [рџ“Љ Stats](#-stats)
- [License](#license)

---

## Quick Start

### РљСѓРґР° РёРґС‚Рё РїРµСЂРІС‹Рј РґРµР»РѕРј?

| РўС‹ РєС‚Рѕ | РљСѓРґР° РёРґС‚Рё |
|--------|-----------|
| рџЊ± **РќРёРєРѕРіРґР° РЅРµ РїРёСЃР°Р» РЅР° C#** | [`CSharp/csharp-basics.md`](CSharp/csharp-basics.md) |
| рџЊї **Junior С…РѕС‡РµС‚ РІ Middle** | [`LearningPath/02_junior-to-middle.md`](LearningPath/02_junior-to-middle.md) (3-6 РјРµСЃСЏС†РµРІ РїР»Р°РЅ) |
| рџЊі **Middle С…РѕС‡РµС‚ РІ Senior** | [`LearningPath/03_middle-to-senior.md`](LearningPath/03_middle-to-senior.md) |
| рџЋ¤ **Р“РѕС‚РѕРІР»СЋСЃСЊ Рє СЃРѕР±РµСЃСѓ** | [`LearningPath/04_interview-prep.md`](LearningPath/04_interview-prep.md) (1-2 РЅРµРґРµР»Рё СЃРїСЂРёРЅС‚) |
| рџЏ—пёЏ **РџСЂРѕРµРєС‚РёСЂСѓСЋ РЅРѕРІРѕРµ РїСЂРёР»РѕР¶РµРЅРёРµ** | [`Architecture/real-world-scenarios.md`](Architecture/real-world-scenarios.md) вЂ” 18 СЃС†РµРЅР°СЂРёРµРІ СЃ СЂРµС€РµРЅРёСЏРјРё |
| рџ¤” **РљР°РєРѕР№ РїР°С‚С‚РµСЂРЅ / Р°СЂС…РёС‚РµРєС‚СѓСЂСѓ РІС‹Р±СЂР°С‚СЊ?** | [`Architecture/patterns-decision-guide.md`](Architecture/patterns-decision-guide.md) |
| рџ“љ **Reference / lookup** | [РџРѕР»РЅРѕРµ РѕРіР»Р°РІР»РµРЅРёРµ РЅРёР¶Рµ](#-the-12-sections) |

### РљР°Рє РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ

1. **Р’ Obsidian** вЂ” РѕС‚РєСЂРѕР№ РїР°РїРєСѓ РєР°Рє vault РґР»СЏ full backlink navigation
2. **РќР° GitHub** вЂ” markdown render OK, СЃСЃС‹Р»РєРё СЂР°Р±РѕС‚Р°СЋС‚ С‡РµСЂРµР· relative paths
3. **Р›РѕРєР°Р»СЊРЅРѕ** вЂ” VSCode + Markdown Preview

---

## How this vault is organized

```
NET-Mastery-Hub/
в”‚
в”њв”Ђв”Ђ рџЋ“ LearningPath/      Roadmaps Junior в†’ Middle в†’ Senior + interview prep
в”‚
в”њв”Ђв”Ђ рџ’Ћ CSharp/             РЎР°Рј СЏР·С‹Рє вЂ” fundamentals в†’ advanced (29 С„Р°Р№Р»РѕРІ)
в”њв”Ђв”Ђ вљ™пёЏ  Runtime/            CLR internals, GC, JIT, threading (7 С„Р°Р№Р»РѕРІ)
в”‚
в”њв”Ђв”Ђ рџЊђ AspNetCore/         Web framework вЂ” auth, middleware, GraphQL, SignalR (14)
в”њв”Ђв”Ђ рџ’ѕ EFCore/             ORM вЂ” basics, performance, patterns (7)
в”њв”Ђв”Ђ рџ—„пёЏ  SQL/                SQL fundamentals + indexes + Postgres (4)
в”‚
в”њв”Ђв”Ђ рџЏ›пёЏ  Architecture/       SOLID, DDD, CQRS, microservices, decision guides (12)
в”њв”Ђв”Ђ вњ… Quality/            Clean code, refactoring, code review, static analysis (5)
в”њв”Ђв”Ђ рџ§Є Testing/            Unit, integration, mocking, mutation, fundamentals (5)
в”њв”Ђв”Ђ вљЎ Performance/        Profiling, optimization, caching, HFT (11)
в”њв”Ђв”Ђ рџљў Infrastructure/     Docker, k8s, CI/CD, observability, messaging (10)
в”‚
в”њв”Ђв”Ђ рџ“‹ Snippets/           Ready-to-copy code patterns (5)
в”њв”Ђв”Ђ рџ“њ Scripts/            Maintenance PowerShell scripts
в””в”Ђв”Ђ рџ“– _changelog.md       What changed when
```

---

## рџ“Ѓ The 12 sections

> РљР°Р¶РґР°СЏ РїР°РїРєР° РёРјРµРµС‚ СЃРІРѕР№ `README.md` СЃ РґРµС‚Р°Р»СЊРЅРѕР№ РЅР°РІРёРіР°С†РёРµР№ РІРЅСѓС‚СЂРё.

### рџЋ“ [LearningPath](LearningPath/) вЂ” РіРґРµ РЅР°С‡Р°С‚СЊ (8 С„Р°Р№Р»РѕРІ)

Roadmaps РґР»СЏ СЂРѕСЃС‚Р°: РєР°РєРёРµ С‚РµРјС‹ РёР·СѓС‡Р°С‚СЊ РІ РєР°РєРѕРј РїРѕСЂСЏРґРєРµ + interview prep.

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ LearningPath/README.md](LearningPath/README.md)

### рџ’Ћ [CSharp](CSharp/) вЂ” СЏР·С‹Рє (29 С„Р°Р№Р»РѕРІ / 933 KB) в­ђ СЃР°РјР°СЏ Р±РѕР»СЊС€Р°СЏ

РџРѕР»РЅРѕРµ РїРѕРєСЂС‹С‚РёРµ C# СЏР·С‹РєР° РѕС‚ basics РґРѕ Senior:

- **Junior:** csharp-basics, datetime-timezones, strings-regex, enums-flags, tuples-deconstruction
- **Middle:** oop, modern-features, async-threading, collections-linq, error-handling, nullable-types, io-streams, equality-comparison, attributes-metadata, indexers-operators, dispose-pattern, extension-methods, iterators-yield, delegates-events
- **Senior:** generics-deep, functional-csharp, design-patterns, types-and-memory, reflection-expression-trees, source-generators, csharp-language-design, csharp-vs-other-langs, cli-tools-scripting, desktop-frameworks

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ CSharp/README.md](CSharp/README.md)

### вљ™пёЏ [Runtime](Runtime/) вЂ” CLR internals (7 С„Р°Р№Р»РѕРІ)

`gc-memory`, `compilation-jit`, `concurrency-atomics`, `span-layout`, `threading-basics`, `interop-pinvoke`, `diagnostics-tools`

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ Runtime/README.md](Runtime/README.md)

### рџЊђ [AspNetCore](AspNetCore/) вЂ” web framework (14 С„Р°Р№Р»РѕРІ / 392 KB)

`api-design`, `auth-security`, `pipeline-middleware`, `di-configuration`, `caching`, `logging-observability`, `hosting-background`, `resilience`, `security-practices`, `signalr`, `graphql`, `blazor-server`, `blazor-wasm`, `native-aot`

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ AspNetCore/README.md](AspNetCore/README.md)

### рџ’ѕ [EFCore](EFCore/) вЂ” ORM (7 С„Р°Р№Р»РѕРІ)

`basics-tracking`, `queries-performance`, `relationships`, `migrations`, `concurrency`, `patterns`, `dapper-comparison`

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ EFCore/README.md](EFCore/README.md)

### рџ—„пёЏ [SQL](SQL/) вЂ” relational DB (4 С„Р°Р№Р»Р°)

`sql-basics`, `indexes-deep`, `optimization`, `postgresql-deep`

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ SQL/README.md](SQL/README.md)

### рџЏ›пёЏ [Architecture](Architecture/) вЂ” patterns & systems (12 С„Р°Р№Р»РѕРІ / 385 KB)

`patterns` (N-Layer/Clean/VSA), `solid`, `ddd`, `cqrs-mediatr`, `distributed-systems`, `microservices-vs-monolith`, `system-design`, `architecture-decisions`, `arch-tests`, **`patterns-decision-guide`** (РЅРѕРІС‹Р№!), **`real-world-scenarios`** (18 case studies), `webai-csharp-architecture`

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ Architecture/README.md](Architecture/README.md)

### вњ… [Quality](Quality/) вЂ” clean code (5 С„Р°Р№Р»РѕРІ)

`clean-code` (Junior basics), `code-quality` (Senior tools вЂ” analyzers/SonarCloud), `refactoring`, `code-review`, `static-analysis`

> вљ пёЏ `clean-code.md` в‰  `code-quality.md` вЂ” СЌС‚Рѕ СЂР°Р·РЅС‹Рµ СѓСЂРѕРІРЅРё РѕРґРЅРѕР№ С‚РµРјС‹ (Junior РїСЂРёРЅС†РёРїС‹ vs Senior tooling).

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ Quality/README.md](Quality/README.md)

### рџ§Є [Testing](Testing/) вЂ” testing strategies (5 С„Р°Р№Р»РѕРІ)

`testing-fundamentals` (Junior basics), `testing` (Senior вЂ” xUnit/TUnit/TestContainers), `integration-testing`, `mocking-strategies`, `mutation-load-testing`

> вљ пёЏ `testing.md` в‰  `testing-fundamentals.md` вЂ” СЌС‚Рѕ СЂР°Р·РЅС‹Рµ СѓСЂРѕРІРЅРё (Senior tools vs Junior basics).

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ Testing/README.md](Testing/README.md)

### вљЎ [Performance](Performance/) вЂ” РїСЂРѕРёР·РІРѕРґРёС‚РµР»СЊРЅРѕСЃС‚СЊ (11 С„Р°Р№Р»РѕРІ)

`performance-fundamentals` (Junior basics), `performance` (Senior вЂ” BenchmarkDotNet/PerfView), `optimization-patterns`, `caching-strategies`, `memory-profiling`, `async-performance`, `lazy-eager-loading`, `hft-low-latency`, `bottleneck-analysis`, `capacity-planning`, `performance-budgets`

> вљ пёЏ `performance.md` в‰  `performance-fundamentals.md` вЂ” Senior tools vs Junior basics.

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ Performance/README.md](Performance/README.md)

### рџљў [Infrastructure](Infrastructure/) вЂ” DevOps & deploy (10 С„Р°Р№Р»РѕРІ / 331 KB)

`docker`, `kubernetes`, `cicd-github-actions`, `observability`, `messaging`, `project-setup`, `ipc-named-pipes-grpc`, `wpf-production`, `llm-rag-patterns`, `semantic-kernel`

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ Infrastructure/README.md](Infrastructure/README.md)

### рџ“‹ [Snippets](Snippets/) вЂ” ready-to-copy (5 С„Р°Р№Р»РѕРІ)

`crud-example`, `efcore-queries`, `mediatr-handlers`, `result-pattern`, `wpf-viewmodel`

в†’ [РџРѕРґСЂРѕР±РЅРµРµ РІ Snippets/README.md](Snippets/README.md)

### рџ“њ [Scripts](Scripts/) вЂ” maintenance

`format_audit.ps1`, `fix_formatting.ps1` вЂ” РґР»СЏ Р°РІС‚РѕРјРµР№РЅС‚РµРЅР°РЅСЃР° С„РѕСЂРјР°С‚Р° РєРѕРґР° РІ `.md` С„Р°Р№Р»Р°С….

```powershell
# Р—Р°РїСѓСЃРєР°С‚СЊ РёР· РєРѕСЂРЅСЏ vault
& "Scripts/format_audit.ps1"
& "Scripts/fix_formatting.ps1"
```

---

## рџЊџ Top-20 must-read РґР»СЏ Senior

Р•СЃР»Рё РІСЂРµРјСЏ РѕРіСЂР°РЅРёС‡РµРЅРѕ вЂ” СЌС‚Рѕ СЃР°РјС‹Рµ С†РµРЅРЅС‹Рµ С„Р°Р№Р»С‹:

### РЇР·С‹Рє + Runtime (must-know internals)

1. [`CSharp/async-threading.md`](CSharp/async-threading.md) вЂ” Task, async/await РїРѕРґ РєР°РїРѕС‚РѕРј (58 KB)
2. [`CSharp/types-and-memory.md`](CSharp/types-and-memory.md) вЂ” value vs reference, boxing, struct internals (53 KB)
3. [`Runtime/gc-memory.md`](Runtime/gc-memory.md) вЂ” GC generations, regions, leaks (56 KB)
4. [`Runtime/span-layout.md`](Runtime/span-layout.md) вЂ” Span\<T\>, ref struct, performance
5. [`CSharp/generics-deep.md`](CSharp/generics-deep.md) вЂ” variance, INumber\<T\>, .NET 7+

### Data + EF Core

6. [`EFCore/basics-tracking.md`](EFCore/basics-tracking.md) вЂ” Change Tracker, AsNoTracking
7. [`EFCore/queries-performance.md`](EFCore/queries-performance.md) вЂ” N+1, projections
8. [`EFCore/dapper-comparison.md`](EFCore/dapper-comparison.md) вЂ” РєРѕРіРґР° EF, РєРѕРіРґР° Dapper
9. [`SQL/indexes-deep.md`](SQL/indexes-deep.md) вЂ” query plans, B-tree internals

### Web framework

10. [`AspNetCore/pipeline-middleware.md`](AspNetCore/pipeline-middleware.md) вЂ” request pipeline
11. [`AspNetCore/auth-security.md`](AspNetCore/auth-security.md) вЂ” JWT, OAuth, OIDC

### РђСЂС…РёС‚РµРєС‚СѓСЂР°

12. [`Architecture/patterns-decision-guide.md`](Architecture/patterns-decision-guide.md) в­ђ вЂ” РєР°РєРѕР№ РїР°С‚С‚РµСЂРЅ РїРѕРґ РєР°РєСѓСЋ Р·Р°РґР°С‡Сѓ
13. [`Architecture/real-world-scenarios.md`](Architecture/real-world-scenarios.md) в­ђ вЂ” 18 case studies
14. [`Architecture/patterns.md`](Architecture/patterns.md) вЂ” N-Layer / Clean / VSA / Hybrid
15. [`Architecture/microservices-vs-monolith.md`](Architecture/microservices-vs-monolith.md) вЂ” РєРѕРіРґР° РІС‹Р±РёСЂР°С‚СЊ
16. [`Architecture/ddd.md`](Architecture/ddd.md) вЂ” Bounded Contexts, Aggregates

### Quality + Testing

17. [`Testing/testing-fundamentals.md`](Testing/testing-fundamentals.md) вЂ” pyramid, FIRST principles
18. [`Quality/clean-code.md`](Quality/clean-code.md) вЂ” fundamentals
19. [`Quality/code-review.md`](Quality/code-review.md) вЂ” process & culture

### Infrastructure

20. [`Infrastructure/observability.md`](Infrastructure/observability.md) вЂ” OpenTelemetry, logs/metrics/traces

---

## рџ—єпёЏ Special navigation files

Р­С‚Рё С„Р°Р№Р»С‹ вЂ” **integrating hubs**, СЃРІСЏР·С‹РІР°СЋС‰РёРµ СЂР°Р·РЅС‹Рµ С‡Р°СЃС‚Рё vault:

| Р¤Р°Р№Р» | Р—Р°С‡РµРј |
|------|-------|
| [`Architecture/patterns-decision-guide.md`](Architecture/patterns-decision-guide.md) | РџРѕРґ РєР°РєСѓСЋ Р·Р°РґР°С‡Сѓ вЂ” РєР°РєРѕР№ РїР°С‚С‚РµСЂРЅ / Р°СЂС…РёС‚РµРєС‚СѓСЂР° |
| [`Architecture/real-world-scenarios.md`](Architecture/real-world-scenarios.md) | 18 РєРѕРЅРєСЂРµС‚РЅС‹С… СЃС†РµРЅР°СЂРёРµРІ: РјРµРЅСЋ, РєРѕСЂР·РёРЅР°, e-commerce, HFT, IoT |
| [`LearningPath/00_overview.md`](LearningPath/00_overview.md) | Р“Р»Р°РІРЅР°СЏ РЅР°РІРёРіР°С†РёСЏ РїРѕ learning path |
| [`LearningPath/05_topics-by-priority.md`](LearningPath/05_topics-by-priority.md) | РўРµРјС‹ РїРѕ РїСЂРёРѕСЂРёС‚РµС‚Сѓ value/effort |

---

## рџЋЇ Quick navigation вЂ” by use case

### "РЇ СЂР°Р·СЂР°Р±Р°С‚С‹РІР°СЋ..."

| Р§С‚Рѕ | РљСѓРґР° |
|-----|------|
| Internal admin tool | [`Architecture/real-world-scenarios.md#СЃС†РµРЅР°СЂРёР№-11`](Architecture/real-world-scenarios.md) |
| РњР°Р»С‹Р№ РёРЅС‚РµСЂРЅРµС‚-РјР°РіР°Р·РёРЅ | [`Architecture/real-world-scenarios.md#СЃС†РµРЅР°СЂРёР№-12`](Architecture/real-world-scenarios.md) |
| РљСЂСѓРїРЅС‹Р№ e-commerce | [`Architecture/real-world-scenarios.md#СЃС†РµРЅР°СЂРёР№-13`](Architecture/real-world-scenarios.md) |
| РљРѕРЅС‚РµРЅС‚-РїРѕСЂС‚Р°Р» / CMS | [`Architecture/real-world-scenarios.md#СЃС†РµРЅР°СЂРёР№-14`](Architecture/real-world-scenarios.md) |
| SaaS B2B РјСѓР»СЊС‚Рё-tenant | [`Architecture/real-world-scenarios.md#СЃС†РµРЅР°СЂРёР№-15`](Architecture/real-world-scenarios.md) |
| HFT / Trading | [`Performance/hft-low-latency.md`](Performance/hft-low-latency.md) |
| IoT РїР»Р°С‚С„РѕСЂРјР° | [`Architecture/real-world-scenarios.md#СЃС†РµРЅР°СЂРёР№-17`](Architecture/real-world-scenarios.md) |
| Desktop app (WPF) | [`CSharp/desktop-frameworks.md`](CSharp/desktop-frameworks.md) |

### "РњРЅРµ РЅСѓР¶РЅРѕ СЂРµС€РёС‚СЊ..."

| РџСЂРѕР±Р»РµРјР° | РљСѓРґР° |
|----------|------|
| N+1 query РІ EF | [`EFCore/queries-performance.md`](EFCore/queries-performance.md) |
| Memory leak | [`Runtime/diagnostics-tools.md`](Runtime/diagnostics-tools.md) + [`Runtime/gc-memory.md`](Runtime/gc-memory.md) |
| Slow database | [`SQL/optimization.md`](SQL/optimization.md) + [`SQL/indexes-deep.md`](SQL/indexes-deep.md) |
| ThreadPool starvation | [`CSharp/async-threading.md`](CSharp/async-threading.md) + [`Runtime/threading-basics.md`](Runtime/threading-basics.md) |
| Distributed transactions | [`Architecture/distributed-systems.md`](Architecture/distributed-systems.md) |
| API versioning | [`AspNetCore/api-design.md`](AspNetCore/api-design.md) |
| Caching strategy | [`AspNetCore/caching.md`](AspNetCore/caching.md) + [`Performance/caching-strategies.md`](Performance/caching-strategies.md) |
| Auth / Identity | [`AspNetCore/auth-security.md`](AspNetCore/auth-security.md) |

### "Р“РѕС‚РѕРІР»СЋСЃСЊ Рє РёРЅС‚РµСЂРІСЊСЋ РЅР°..."

| РЈСЂРѕРІРµРЅСЊ | РљСѓРґР° |
|---------|------|
| Junior C# | [`LearningPath/02_junior-to-middle.md`](LearningPath/02_junior-to-middle.md) |
| Middle .NET | [`LearningPath/03_middle-to-senior.md`](LearningPath/03_middle-to-senior.md) + Top-20 СЃРїРёСЃРѕРє РІС‹С€Рµ |
| Senior .NET | Top-20 СЃРїРёСЃРѕРє + [`Architecture/`](Architecture/) РїРѕР»РЅРѕСЃС‚СЊСЋ |
| Behavioral / soft | [`LearningPath/10_interview-behavioral.md`](LearningPath/10_interview-behavioral.md) |
| Final sprint (1 РЅРµРґРµР»СЏ) | [`LearningPath/04_interview-prep.md`](LearningPath/04_interview-prep.md) + [`LearningPath/09_senior-tips-cheatsheet.md`](LearningPath/09_senior-tips-cheatsheet.md) |

---

## рџ“ђ Conventions & format

### РљР°Р¶РґС‹Р№ С„Р°Р№Р» follows:

```markdown
---
tags: [topic1, topic2, level]
level: Junior | Middle | Senior | Junior to Senior
date: YYYY-MM-DD
---

# Topic Name

> Tagline вЂ” С‡С‚Рѕ Рё Р·Р°С‡РµРј (1-2 СЃС‚СЂРѕРєРё).

## Р§С‚Рѕ СЌС‚Рѕ, Р·Р°С‡РµРј Рё РєРѕРіРґР°

## 1. Р‘Р°Р·РѕРІР°СЏ РєРѕРЅС†РµРїС†РёСЏ
## 2. ... (С‚РµРјР°С‚РёС‡РµСЃРєРёРµ СЃРµРєС†РёРё СЃ РїСЂРёРјРµСЂР°РјРё)
## N. Common Pitfalls
## N+1. Best Practices
## Cheat sheet
## Decision tree (РµСЃР»Рё РїСЂРёРјРµРЅРёРјРѕ)
## РЎРј. С‚Р°РєР¶Рµ (cross-references)
## Reading list (books, blogs, docs)
```

### Cross-references

РСЃРїРѕР»СЊР·СѓРµС‚СЃСЏ Obsidian-style: `[[folder/file|display name]]`. РќР° GitHub вЂ” СЂРµРЅРґРµСЂСЏС‚СЃСЏ РєР°Рє РѕР±С‹С‡РЅС‹Рµ СЃСЃС‹Р»РєРё.

### Code blocks

Р’РЎР•Р“Р”Рђ blank line РјРµР¶РґСѓ preceding header Рё opening triple-backticks. РЎРєСЂРёРїС‚ `Scripts/fix_formatting.ps1` С‡РёРЅРёС‚ Р°РІС‚РѕРјР°С‚РѕРј.

### Tags

РљР°Р¶РґС‹Р№ С„Р°Р№Р» РёРјРµРµС‚ tags РІ frontmatter. РџРѕРёСЃРє РїРѕ С‚РµРіР°Рј:

```bash
# Р’СЃРµ Junior С‚РµРјС‹
grep -r "level: Junior" --include="*.md" -l

# Р’СЃРµ РїСЂРѕ async
grep -r "tags:.*async" --include="*.md" -l
```

---

## рџ“Љ Stats

| | Value |
|---|---|
| **Total files** | 138 |
| **Total size** | ~3.4 MB |
| **Coverage** | Junior в†’ Senior+ |
| **Largest folder** | CSharp (35 files / ~1.1 MB) |
| **Largest file** | `CSharp/async-threading.md` (58 KB) |
| **Language** | Russian (primary), English (technical terms) |
| **Last major update** | 2026-04-30 |

### РџРѕ СѓСЂРѕРІРЅСЋ

```
Junior:           ~12 С„Р°Р№Р»РѕРІ  (basics, fundamentals, daily work)
Middle:           ~45 С„Р°Р№Р»РѕРІ  (oop, async, EF, ASP.NET, testing)
Middle to Senior: ~25 С„Р°Р№Р»РѕРІ  (patterns, generics, deep topics)
Senior:           ~37 С„Р°Р№Р»РѕРІ  (architecture, runtime, performance, advanced)
```

### РџРѕ РєР°С‚РµРіРѕСЂРёСЏРј

```
Language:        35 files  (CSharp/)
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

Personal knowledge base вЂ” no formal license. Feel free to learn from it; don't republish wholesale.

If you find errors or have suggestions, [open an issue](https://github.com/valinerosgordov/NET-Mastery-Hub/issues).
