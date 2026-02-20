# .NET Learning Path — Todo List

Пошаговый план обучения с оценкой времени и целевого уровня.

---

## Learn Path: с чего начать

```
1. C# Fundamentals (2–3 недели)
   ↓
2. C# Advanced (2 недели)
   ↓
3. ASP.NET Core (3–4 недели)
   ↓
4. EF Core (2–3 недели)
   ↓
5. Architecture + Infrastructure (2–3 недели)
   ↓
6. Testing + DevOps (1–2 недели)
   ↓
7. Advanced Topics (1–2 недели)
   ↓
8. Interview Prep (1–2 недели)
```

**Старт**: [[Topics/NetQuestions150/csharp/01-types-memory|01 Types & Memory]] или [[Interview/01-csharp-fundamentals|Interview: C# Fundamentals]].

---

## Phase 1: C# Fundamentals (2–3 недели)

| #   | Тема                 | Материал                                                                     | Время | Статус |
| --- | -------------------- | ---------------------------------------------------------------------------- | ----- | ------ |
| 1   | Типы и память        | [[Topics/NetQuestions150/csharp/01-types-memory\|01 Types & Memory]]         | 3–4 ч | ⬜      |
| 2   | OOP                  | [[Topics/NetQuestions150/csharp/02-oop\|02 OOP]]                             | 4–5 ч | ⬜      |
| 3   | Collections и LINQ   | [[Topics/NetQuestions150/csharp/03-collections-linq\|03 Collections & LINQ]] | 5–6 ч | ⬜      |
| 4   | Языковые конструкции | [[Topics/NetQuestions150/csharp/05-language\|05 Language]]                   | 4–5 ч | ⬜      |

**Практика**: консольные приложения, LINQ-запросы, работа с коллекциями.

---

## Phase 2: C# Advanced (2 недели)

| # | Тема | Материал | Время | Статус |
|---|------|----------|-------|--------|
| 5 | Async и Concurrency | [[Topics/NetQuestions150/csharp/04-async-concurrency\|04 Async & Concurrency]] | 6–8 ч | ⬜ |
| 6 | Async углублённо | [[Interview/02-async-threading\|Interview: Async & Threading]] | 3–4 ч | ⬜ |

**Практика**: async/await, Task.WhenAll, Channel, SemaphoreSlim.

---

## Phase 3: ASP.NET Core (3–4 недели)

| # | Тема | Материал | Время | Статус |
|---|------|----------|-------|--------|
| 7 | Pipeline и Routing | [[Topics/NetQuestions150/aspnet/01-pipeline-routing\|01 Pipeline & Routing]] | 4–5 ч | ⬜ |
| 8 | DI и Configuration | [[Topics/NetQuestions150/aspnet/02-di-configuration\|02 DI & Configuration]] | 3–4 ч | ⬜ |
| 9 | Options и Validation | [[Topics/NetQuestions150/aspnet/03-options-validation\|03 Options & Validation]] | 3–4 ч | ⬜ |
| 10 | Auth | [[Topics/NetQuestions150/aspnet/04-auth\|04 Auth]] | 5–6 ч | ⬜ |
| 11 | Hosting | [[Topics/NetQuestions150/aspnet/05-hosting\|05 Hosting]] | 2–3 ч | ⬜ |
| 12 | Caching | [[Topics/NetQuestions150/aspnet/06-caching\|06 Caching]] | 3–4 ч | ⬜ |
| 13 | API | [[Topics/NetQuestions150/aspnet/07-api\|07 API]] | 5–6 ч | ⬜ |
| 14 | Logging | [[Topics/NetQuestions150/aspnet/08-logging\|08 Logging]] | 2–3 ч | ⬜ |

**Практика**: REST API с CRUD, JWT auth, Minimal API или MVC.

---

## Phase 4: EF Core (2–3 недели)

| # | Тема | Материал | Время | Статус |
|---|------|----------|-------|--------|
| 15 | Migrations | [[Topics/NetQuestions150/efcore/01-migrations-schema\|01 Migrations]] | 2–3 ч | ⬜ |
| 16 | Loading и Tracking | [[Topics/NetQuestions150/efcore/02-loading-tracking\|02 Loading & Tracking]] | 4–5 ч | ⬜ |
| 17 | Relationships | [[Topics/NetQuestions150/efcore/03-relationships\|03 Relationships]] | 4–5 ч | ⬜ |
| 18 | Queries | [[Topics/NetQuestions150/efcore/04-queries\|04 Queries]] | 3–4 ч | ⬜ |
| 19 | Performance | [[Topics/NetQuestions150/efcore/05-performance\|05 Performance]] | 4–5 ч | ⬜ |
| 20 | Concurrency и Transactions | [[Topics/NetQuestions150/efcore/06-concurrency-transactions\|06 Concurrency & Transactions]] | 3–4 ч | ⬜ |
| 21 | Patterns | [[Topics/NetQuestions150/efcore/07-patterns\|07 Patterns]] | 3–4 ч | ⬜ |
| 22 | SQL оптимизация | [[Topics/SQL/sql-query-optimization\|SQL Optimization]] | 2–3 ч | ⬜ |

**Практика**: API с БД, миграции, Include, AsNoTracking, N+1 fix.

---

## Phase 5: Architecture и окружение (2–3 недели)

| # | Тема | Материал | Время | Статус |
|---|------|----------|-------|--------|
| 23 | Архитектуры | [[Architecture/architecture-tutorial\|Architecture Tutorial]] | 6–8 ч | ⬜ |
| 24 | Соглашения и тесты | [[Architecture/architecture-conventions-and-tests\|Conventions & Tests]] | 2–3 ч | ⬜ |
| 25 | Project Setup | [[Topics/ProjectSetup/start-dotnet-project-2026\|Start .NET Project 2026]] | 2–3 ч | ⬜ |
| 26 | Top 10 things | [[Topics/ProjectSetup/top-10-things-dotnet-2026\|Top 10 .NET 2026]] | 1–2 ч | ⬜ |
| 27 | Code Quality | [[Topics/CodeQuality/code-quality-best-practices\|Code Quality]] | 2–3 ч | ⬜ |
| 28 | Security | [[Interview/04-security\|Interview: Security]] | 3–4 ч | ⬜ |
| 29 | Observability | [[Topics/Observability/opentelemetry-jaeger-seq\|OpenTelemetry, Jaeger, Seq]] | 4–5 ч | ⬜ |
| 30 | Result и MediatR | [[Topics/ResultPattern/result-pattern-cqrs\|Result/Option и MediatR]] | 3–4 ч | ⬜ |

---

## Phase 6: Testing и DevOps (1–2 недели)

| # | Тема | Материал | Время | Статус |
|---|------|----------|-------|--------|
| 31 | Unit и Integration тесты | [[Topics/Testing/testing-xunit-testcontainers\|Testing]] | 5–6 ч | ⬜ |
| 32 | Testing Interview | [[Interview/08-testing\|Interview: Testing]] | 2–3 ч | ⬜ |
| 33 | Docker и деплой | [[Topics/Docker/docker-deploy\|Docker & CI/CD]] | 4–5 ч | ⬜ |
| 34 | Git, CI/CD basics | [[Topics/Docker/docker-deploy\|Docker & CI/CD]] (Git в том же) | 2–3 ч | ⬜ |

**Практика**: тесты для pet-проекта, Dockerfile, GitHub Actions.

---

## Phase 7: Advanced Topics (1–2 недели)

| # | Тема | Материал | Время | Статус |
|---|------|----------|-------|--------|
| 35 | Messaging | [[Topics/Messaging/rabbitmq-masstransit\|RabbitMQ, MassTransit]] | 4–5 ч | ⬜ |
| 36 | Performance | [[Topics/Performance/dotnet-performance\|.NET Performance]] | 3–4 ч | ⬜ |
| 37 | Architecture Interview | [[Interview/07-architecture\|Interview: Architecture]] | 2–3 ч | ⬜ |

---

## Phase 8: Interview Prep (1–2 недели)

| # | Тема | Материал | Время | Статус |
|---|------|----------|-------|--------|
| 38 | Все категории Interview | [[Interview/interview-index\|Interview Index]] | 8–12 ч | ⬜ |
| 39 | Logging, Metrics | [[Interview/06-logging-metrics\|Logging & Metrics]] | 2–3 ч | ⬜ |
| 40 | Behavioral | [[Interview/09-behavioral\|Behavioral]] | 2–3 ч | ⬜ |
| 41 | NetQuestions150 повтор | [[Topics/NetQuestions150/net-questions-150\|150 Questions]] | 6–8 ч | ⬜ |

---

## Чеклист перед интервью (за 1–2 дня)

- [ ] C#: class vs struct vs record, async, LINQ, generics
- [ ] ASP.NET: pipeline, DI lifetimes, JWT, Options
- [ ] EF Core: migrations, N+1, AsNoTracking, transactions
- [ ] Архитектура: Clean, VSA, паттерны
- [ ] Тесты: xUnit, моки, Testcontainers
- [ ] Behavioral: STAR, конфликты, сложные задачи

---

## Pet-проект: идеи

| Проект | Сложность | Что прокачаешь |
|--------|-----------|----------------|
| Task API | Легко | CRUD, auth, EF Core |
| Блог | Средне | Посты, комментарии, пагинация |
| E-commerce | Средне | Заказы, корзина, платежи (mock) |
| Real-time чат | Сложно | SignalR, WebSockets |
| Modular Monolith | Сложно | Модули, messaging, CQRS |

---

## Книги и курсы

| Ресурс | Назначение |
|--------|------------|
| **CLR via C#** (Рихтер) | Глубокое понимание CLR, GC, типов |
| **Pro C# and .NET** | Актуальный C# и .NET |
| **Designing Data-Intensive Applications** | БД, масштабирование, distributed systems |
| **Clean Code** (Мартин) | Читаемость, рефакторинг |
| Microsoft Learn | Бесплатные модули по .NET |

---

## Оценка времени

| Режим | Часов в день | Общее время |
|-------|--------------|-------------|
| **Part-time** | 1–2 ч | 16–20 недель (~4–5 месяцев) |
| **Part-time** | 2–3 ч | 12–16 недель (~3–4 месяца) |
| **Full-time** | 6–8 ч | 6–8 недель (~1.5–2 месяца) |

**Чистое время на материалы**: ~120–150 часов. Практика (pet-проект) добавляет 50–100%.

---

## Уровень после прохождения

| Phase | Уровень | Что умеешь |
|-------|---------|------------|
| 1–2 | **Junior** | C# синтаксис, OOP, LINQ, async. Пишешь код по заданию. |
| 3–4 | **Junior+** | REST API, EF Core, CRUD. Самостоятельно делаешь простые фичи. |
| 5 | **Middle** | Архитектура, Observability, Result/MediatR. Настраиваешь проект. |
| 6–7 | **Middle+** | Тесты, Docker, CI/CD, Messaging, Performance. Полный цикл разработки. |
| 8 | **Middle+** | Проходишь технические интервью, объясняешь решения. |

**Полный путь** → **Middle .NET Backend Developer**: API, БД, auth, тесты, DevOps, архитектура.

Для **Senior**: + системный дизайн, масштабирование, лидерство, менторинг.

---

## Рекомендации

1. **Практика важнее теории** — после каждого модуля делай мини-проект или задачу.
2. **Pet-проект** — один проект на весь путь, наращивай фичи по мере изучения.
3. **Порядок** — не перескакивай: C# → ASP.NET → EF Core. Async можно изучать параллельно с ASP.NET.
4. **Повтор** — перед интервью пройди NetQuestions150 за 1–2 недели.
5. **Git** — с первого дня: ветки, коммиты, PR. CI/CD — когда есть работающий проект.
