---
tags: [learning-path, junior, middle, roadmap]
level: Junior to Middle
date: 2026-04-30
---

# 🎓 Junior → Middle .NET Backend

> Roadmap для перехода из junior в middle. Ориентир: **3-6 месяцев** active learning. Все ссылки ведут на актуальные файлы vault. После каждой phase — практика на pet-проекте.

---

## 📅 Time estimate

| Режим | Часов в день | Длительность |
|-------|--------------|--------------|
| Part-time (вечерами) | 1-2 ч | ~5-6 месяцев |
| Part-time активно | 2-3 ч | ~3-4 месяца |
| Full-time | 6-8 ч | ~1.5-2 месяца |

**Чистое время на материалы:** ~120-150 часов. Практика добавляет 50-100% сверху.

---

## Phase 1 — C# Fundamentals (3-4 недели)

**Цель:** уверенно писать на C#, понимать базовые механики.

### Обязательно

| # | Тема | Файл | Время |
|---|------|------|-------|
| 1 | Типы и память | [[types-and-memory\|types-and-memory]] | 4-5 ч |
| 2 | OOP | [[oop\|oop]] | 5-6 ч |
| 3 | Modern features (8→14) | [[modern-features\|modern-features]] | 5-6 ч |
| 4 | Collections и LINQ | [[collections-linq\|collections-linq]] | 6-7 ч |
| 5 | Delegates и Events | [[delegates-events\|delegates-events]] | 4-5 ч |

### Practice

- Console приложения с LINQ
- Простой CRUD без БД (in-memory)
- Решить 30-50 задач на LeetCode на C# (Easy/Medium)

### Чек-лист

- [ ] Понимаю разницу class vs struct vs record
- [ ] Знаю когда `var` vs explicit type
- [ ] Использую records для DTO
- [ ] LINQ читается как родной (Where/Select/GroupBy/Aggregate)
- [ ] Понимаю closures и captured variables в lambda

---

## Phase 2 — Async и продвинутый C# (2-3 недели)

**Цель:** не бояться async, понимать concurrency.

| # | Тема | Файл | Время |
|---|------|------|-------|
| 6 | Async и Threading | [[async-threading\|async-threading]] | 7-8 ч |
| 7 | Error handling | [[error-handling\|error-handling]] | 3-4 ч |
| 8 | Concurrency atomics | [[concurrency-atomics\|concurrency-atomics]] | 4-5 ч |

### Practice

- Async file processor (читать файлы parallel)
- Web scraper (Task.WhenAll + SemaphoreSlim)
- Producer-consumer через Channel<T>

### Чек-лист

- [ ] Никогда не пишу `.Result` / `.Wait()` (deadlock!)
- [ ] CancellationToken прокидываю везде
- [ ] Понимаю ConfigureAwait(false) — когда нужен
- [ ] Использую Result<T,E> вместо exceptions для business logic

---

## Phase 3 — ASP.NET Core (4-5 недель)

**Цель:** строить REST API.

| # | Тема | Файл | Время |
|---|------|------|-------|
| 9 | Pipeline & Middleware | [[pipeline-middleware\|pipeline-middleware]] | 4-5 ч |
| 10 | DI & Configuration | [[di-configuration\|di-configuration]] | 4-5 ч |
| 11 | API Design | [[api-design\|api-design]] | 4-5 ч |
| 12 | Authentication & Security | [[auth-security\|auth-security]] | 5-6 ч |
| 13 | Caching | [[caching\|caching]] | 3-4 ч |
| 14 | Logging & Observability | [[logging-observability\|logging-observability]] | 3-4 ч |
| 15 | Hosting & Background | [[hosting-background\|hosting-background]] | 3-4 ч |
| 16 | Resilience (Polly) | [[resilience\|resilience]] | 3-4 ч |

### Practice

- **Pet-project: Task API** — CRUD задачи с auth
- Добавить JWT auth
- Подключить логирование (Serilog)
- Добавить Polly для retry на внешние API

### Чек-лист

- [ ] Понимаю DI lifetimes (Singleton/Scoped/Transient) и их ловушки
- [ ] Минимум 3 типа auth знаю (cookies / JWT / API key)
- [ ] Структурированные логи (`logger.LogInformation("{Id}", id)`)
- [ ] Знаю когда Memory vs Distributed cache

---

## Phase 4 — EF Core и SQL (3-4 недели)

**Цель:** работать с БД эффективно.

| # | Тема | Файл | Время |
|---|------|------|-------|
| 17 | EF Core Basics | [[basics-tracking\|basics-tracking]] | 4-5 ч |
| 18 | Migrations | [[migrations\|migrations]] | 3-4 ч |
| 19 | Relationships | [[relationships\|relationships]] | 4-5 ч |
| 20 | Queries Performance | [[queries-performance\|queries-performance]] | 4-5 ч |
| 21 | Concurrency | [[concurrency\|concurrency]] | 3-4 ч |
| 22 | EF Patterns | [[C# and NET/EFCore/patterns\|patterns]] | 3-4 ч |
| 23 | SQL Optimization | [[optimization\|optimization]] | 4-5 ч |
| 24 | PostgreSQL Deep | [[postgresql-deep\|postgresql-deep]] | 5-6 ч |

### Practice

- Добавить EF Core к pet-проекту
- Migrations: создать модели, обновить, откатить
- Найти и пофиксить N+1 query (с EF Logging)
- AsNoTracking для read-only endpoints
- Написать 10+ EXPLAIN ANALYZE для PostgreSQL

### Чек-лист

- [ ] Понимаю N+1 проблему и как её ловить (EF Logging / dotnet-counters)
- [ ] AsNoTracking всегда для read-only
- [ ] Use Projection (`.Select`) вместо Include когда возможно
- [ ] Написать индекс на колонку которая в WHERE / JOIN
- [ ] Optimistic concurrency через RowVersion / xmin

---

## Phase 5 — Testing и DevOps (2-3 недели)

**Цель:** писать тесты и деплоить.

| # | Тема | Файл | Время |
|---|------|------|-------|
| 25 | Testing | [[testing\|testing]] | 5-6 ч |
| 26 | Code Quality | [[code-quality\|code-quality]] | 3-4 ч |
| 27 | Project Setup | [[project-setup\|project-setup]] | 3-4 ч |
| 28 | Docker | [[docker\|docker]] | 4-5 ч |

### Practice

- Покрыть pet-проект тестами (>50% coverage)
- xUnit unit-tests для domain logic
- Integration test через WebApplicationFactory + Testcontainers
- Dockerfile для проекта (multi-stage)
- GitHub Actions CI: build + test + Docker push

### Чек-лист

- [ ] xUnit + Shouldly для assertions
- [ ] Testcontainers вместо in-memory DB
- [ ] CI прогоняет тесты на каждый PR
- [ ] Docker image размером < 100 MB
- [ ] EditorConfig + analyzers (Sonar, Meziantou) включены

---

## Phase 6 — Architecture introduction (2-3 недели)

**Цель:** структурировать код, понимать паттерны.

| # | Тема | Файл | Время |
|---|------|------|-------|
| 29 | SOLID | [[solid\|solid]] | 3-4 ч |
| 30 | Architecture Patterns | [[C# and NET/Architecture/patterns\|patterns]] | 6-7 ч |
| 31 | CQRS + MediatR | [[cqrs-mediatr\|cqrs-mediatr]] | 4-5 ч |
| 32 | DDD intro | [[ddd\|ddd]] | 5-6 ч |
| 33 | Design Patterns | [[design-patterns\|design-patterns]] | 4-5 ч |

### Practice

- Перестроить pet-project в Vertical Slices
- MediatR для command/query
- Один DDD aggregate с invariants
- Один Saga / state machine

### Чек-лист

- [ ] Понимаю модели Clean / Onion / Hexagonal
- [ ] Vertical Slices > N-layer для небольших проектов
- [ ] CQRS как разделить read/write
- [ ] DDD: Aggregate, Value Object, Domain Events

---

## ✅ Итоговый Pet-Project (заключительные 1-2 недели)

После всех phases твой pet-project должен иметь:

1. ✅ REST API на ASP.NET Core 10
2. ✅ EF Core + PostgreSQL (Testcontainers для тестов)
3. ✅ JWT auth с refresh tokens
4. ✅ MediatR для Commands/Queries
5. ✅ FluentValidation для requests
6. ✅ Result<T,E> вместо exceptions
7. ✅ Serilog с structured logging
8. ✅ OpenTelemetry traces
9. ✅ xUnit + Testcontainers (>50% coverage)
10. ✅ Dockerfile + GitHub Actions CI
11. ✅ Resilience через Polly для внешних HTTP
12. ✅ Vertical Slices structure

**Это middle-уровень.** Можно идти на собесы.

---

## 🎯 Идеи для pet-project

| Проект | Сложность | Что прокачаешь |
|--------|-----------|----------------|
| **Task API** ⭐ | Easy | CRUD, auth, EF Core (старт-проект) |
| **Блог** | Medium | Posts, comments, pagination, search |
| **E-commerce** | Medium | Orders, cart, payments (mock) |
| **URL shortener** | Easy | High-throughput design, caching |
| **Real-time chat** | Hard | SignalR, WebSockets |
| **Job board** | Medium | Search, filters, notifications |
| **Habit tracker** | Easy | CRUD, streaks, gamification |

**Рекомендация:** Начни с **Task API**. После всех phases расширяй ту же codebase, не начинай новый.

---

## 📚 Книги (must-read для middle)

См. [[99_reading-list\|99 Reading List]] — full list.

Топ-3 для середины пути:

1. **C# in Depth** — Jon Skeet (язык deeply)
2. **EF Core in Action** — Jon P Smith (EF Core от автора)
3. **Code That Fits in Your Head** — Mark Seemann (clean code современный)

---

## ⏭️ Что дальше — путь в Senior

После завершения этого roadmap → переход в [[03_middle-to-senior\|03 Middle → Senior]].

Senior отличается от Middle не количеством знаний, а:
- Системным мышлением (архитектура целой системы)
- Опытом с production incidents (debugging, troubleshooting)
- Менторингом и code review
- Trade-off thinking (всегда compromise, нет idealов)
- Business awareness (зачем код пишем)
