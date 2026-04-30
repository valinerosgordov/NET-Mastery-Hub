# NET-Mastery-Hub

> Comprehensive C# / .NET knowledge base — from Junior fundamentals to Senior architecture. **111 deep-dive notes / ~2.9 MB**, organized for systematic learning, interview prep, and day-to-day reference.

Production-grade .NET coverage: language internals, runtime, ASP.NET Core, EF Core, SQL, architecture patterns, performance, testing, infrastructure. Russian primary, English technical terms.

---

## Quick Start

### What this is

A structured, deeply-researched knowledge base for .NET developers. Each topic file is **17–60 KB** of practical content with examples, common pitfalls, best practices, decision trees, and curated reading lists.

Not a tutorial. Not a course. A **reference vault** organized by topic, designed to be navigated via Obsidian-style backlinks.

### Who this is for

- **Junior** developers building foundations (start with `LearningPath/02_junior-to-middle.md`)
- **Middle** developers preparing for Senior interviews (`LearningPath/03_middle-to-senior.md`)
- **Senior** developers as a reference and refresher (Top-15 list below)
- Anyone preparing for technical interviews in C# / .NET

### How to use

1. Open the folder in [Obsidian](https://obsidian.md/) for full backlink navigation
2. Or browse on GitHub — markdown renders fine, links work via relative paths
3. Start with `LearningPath/00_overview.md` for navigation

---

## Structure (12 sections, 111 files)

```
LearningPath/    8 files   Roadmaps and study plans
CSharp/         25 files   The language itself
Runtime/         7 files   CLR, GC, JIT, threading
AspNetCore/     14 files   Web framework deep
EFCore/          6 files   ORM
SQL/             4 files   Relational DB
Architecture/   10 files   Patterns, DDD, distributed
Infrastructure/  8 files   Docker, observability, messaging
Performance/    11 files   Performance engineering
Quality/         5 files   Clean code, refactoring, review
Testing/         5 files   Unit, integration, mocking, mutation
Snippets/        5 files   Ready-to-copy patterns
```

### LearningPath — start here

| File | Purpose |
|------|---------|
| `00_overview.md` | Main navigation |
| `02_junior-to-middle.md` | 3-6 month roadmap |
| `03_middle-to-senior.md` | Senior roadmap |
| `04_interview-prep.md` | 1-2 week sprint before interviews |
| `05_topics-by-priority.md` | Topics ranked by value/effort |
| `09_senior-tips-cheatsheet.md` | Quick reference |
| `10_interview-behavioral.md` | Soft skills, STAR method |
| `99_reading-list.md` | Books, blogs, conferences |

### CSharp — the language (25 files)

Complete coverage from fundamentals to advanced:

- **Junior intro:** `csharp-basics`
- **Daily work:** `strings-regex`, `datetime-timezones`, `io-streams`, `nullable-types`, `error-handling`, `collections-linq`, `iterators-yield`, `tuples-deconstruction`, `enums-flags`, `attributes-metadata`, `equality-comparison`
- **Middle fundamentals:** `oop`, `modern-features`, `types-and-memory`, `async-threading`, `delegates-events`
- **Advanced:** `functional-csharp`, `design-patterns`, `generics-deep`
- **Senior:** `reflection-expression-trees`, `source-generators`
- **Context:** `csharp-language-design`, `csharp-vs-other-langs`
- **Domain:** `cli-tools-scripting`, `desktop-frameworks`

### Runtime — CLR internals (7 files)

`gc-memory`, `compilation-jit`, `concurrency-atomics`, `span-layout`, `threading-basics`, `interop-pinvoke`, `diagnostics-tools`

### AspNetCore (14 files)

`api-design`, `auth-security`, `pipeline-middleware`, `di-configuration`, `caching`, `logging-observability`, `hosting-background`, `resilience`, `security-practices`, `signalr`, `graphql`, `blazor-server`, `blazor-wasm`, `native-aot`

### EFCore (6 files)

`basics-tracking`, `queries-performance`, `relationships`, `migrations`, `concurrency`, `patterns`

### SQL (4 files)

`sql-basics`, `indexes-deep`, `optimization`, `postgresql-deep`

### Architecture (10 files)

`patterns` (Modular Monolith / VSA / Clean), `solid`, `ddd`, `cqrs-mediatr`, `distributed-systems`, `microservices-vs-monolith`, `system-design`, `architecture-decisions`, `arch-tests`, `webai-csharp-architecture`

### Infrastructure (8 files)

`docker`, `observability`, `messaging`, `project-setup`, `ipc-named-pipes-grpc`, `wpf-production`, `llm-rag-patterns`, `semantic-kernel`

### Performance (11 files)

`performance`, `hft`, plus 9 specialized topics on profiling, optimization patterns, caching strategies, lazy loading, memory analysis.

### Quality (5 files)

`clean-code`, `refactoring`, `code-review`, `code-quality`, `static-analysis`

### Testing (5 files)

`testing-fundamentals`, `testing` (xUnit practical), `integration-testing`, `mocking-strategies`, `mutation-load-testing`

### Snippets (5 files)

Production-ready code patterns: CRUD example, EF Core queries, MediatR handlers, Result pattern, WPF view models.

---

## Top-15 must-read for Senior

If time is short:

1. `CSharp/async-threading.md` — Task, async/await internals
2. `Runtime/gc-memory.md` — GC generations, regions, leaks
3. `EFCore/basics-tracking.md` — Change Tracker, AsNoTracking
4. `EFCore/queries-performance.md` — N+1, projections
5. `AspNetCore/pipeline-middleware.md` — request pipeline
6. `AspNetCore/auth-security.md` — JWT, OAuth, OIDC
7. `Architecture/patterns.md` — Modular Monolith, VSA, Clean
8. `Architecture/microservices-vs-monolith.md` — when to choose
9. `Architecture/ddd.md` — bounded contexts, aggregates
10. `SQL/sql-basics.md` — JOINs, transactions, isolation
11. `SQL/indexes-deep.md` — query plans, B-tree internals
12. `Testing/testing-fundamentals.md` — pyramid, FIRST principles
13. `Testing/integration-testing.md` — modern stack
14. `Quality/clean-code.md` — fundamentals
15. `Quality/code-review.md` — process and culture

---

## Conventions

### File format

Every note follows the same structure:

```
---
tags: [...]
level: Junior | Middle | Senior | All
date: YYYY-MM-DD
---

# Topic Name

> Tagline — what and why.

## What it is, why, when

## 1, 2, 3, ... topical sections with examples

## Common Pitfalls

## Best Practices

## Cheat sheet

## Decision tree

## See also (cross-references)

## Reading list
```

### Cross-references

Use Obsidian-style links: `[[folder/file|display name]]`. They render as plain links on GitHub.

### Code blocks

Always have a blank line between the preceding header and the opening triple-backticks. The `Scripts/fix_formatting.ps1` script auto-fixes this across the vault.

---

## Maintenance scripts

```powershell
# Audit formatting issues across all .md files
& "Scripts/format_audit.ps1"

# Auto-fix code block formatting
& "Scripts/fix_formatting.ps1"
```

---

## Stats

| | Value |
|---|---|
| Files | 111 |
| Total size | ~2.9 MB |
| Coverage | Junior → Senior+ |
| Language | Russian (primary), English (technical terms) |
| Last major update | 2026-04-30 |

---

## License

Personal knowledge base — no formal license. Feel free to learn from it; don't republish wholesale.

If you find errors or have suggestions, open an issue.
