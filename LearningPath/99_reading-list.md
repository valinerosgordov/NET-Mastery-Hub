---
tags: [meta, reading-list, resources, blogs, curated]
level: All
date: 2026-06-28
---

# Reading List — внешние ресурсы

> Курируемый список внешних .NET-ресурсов для Senior-level чтения: блоги (antondevtips, Milan Jovanović), Telegram-каналы, официальные источники и книги.

Курируемый список материалов для углублённого чтения. Всё проверено лично или отмечено как ожидающее прочтения. Формат: blog/article → topic → tags.

## Что это, зачем и когда

### Что такое?
Отобранные статьи и блоги, которые дают **углублённое Senior-level понимание** конкретных тем .NET. Не "подборка 1000 ссылок", а минимальный набор, от которого реально растёт уровень.

### Как пользоваться
- Раз в неделю открываешь список, выбираешь 1-2 статьи
- После прочтения — либо промоутишь в отдельную заметку в vault, либо помечаешь галочкой
- Если материал устарел/оказался слабым — вычёркиваешь (не копим мёртвые ссылки)

---

## antondevtips.com

Антон Мартыненко — один из самых качественных русскоязычных авторов по .NET. Длинные технические статьи, часто с production-сценариями.

### Архитектура

- [ ] [N-Layered vs Clean vs Vertical Slice Architecture](https://antondevtips.com/blog/n-layered-vs-clean-vs-vertical-slice-architecture) — сравнение подходов → см. [[architecture-patterns|Architecture/patterns.md]]
- [ ] [Лучшая структура .NET-проектов с Clean Architecture и Vertical Slices](https://antondevtips.com/blog/the-best-way-to-structure-your-dotnet-projects-with-clean-architecture-and-vertical-slices)
- [ ] [Зачем писать архитектурные тесты](https://antondevtips.com/blog/why-do-you-need-to-write-architecture-tests-in-dotnet) → уже в [[arch-tests|Architecture/arch-tests.md]]
- [ ] [От модульного монолита к микросервисам](https://antondevtips.com/blog/migrating-modular-monolith-to-microservices-in-dotnet)

### API и Backend

- [ ] [Best Practices для REST API](https://antondevtips.com/blog/best-practices-for-building-rest-apis)
- [ ] [90% API не RESTful — что вы упускаете](https://antondevtips.com/blog/90-of-apis-are-not-restful-what-youre-missing-and-when-it-matters)
- [ ] [Как ускорить Web API](https://antondevtips.com/blog/how-to-increase-performance-of-web-apis-in-dotnet)
- [ ] [Authentication и Authorization в ASP.NET Core](https://antondevtips.com/blog/authentication-and-authorization-best-practices-in-aspnetcore)
- [ ] [Интеграционные тесты в ASP.NET Core](https://antondevtips.com/blog/asp-net-core-integration-testing-best-practises)

### EF Core и данные

- [ ] [Почему не нужен Repository поверх EF Core](https://antondevtips.com/blog/why-you-dont-need-a-repository-in-ef-core) → см. [[ef-patterns|EFCore/patterns.md]]
- [ ] [5 скрытых NuGet-пакетов для EF Core](https://antondevtips.com/blog/5-hidden-efcore-nuget-packages) — **ценно, прочитать**
- [ ] [Кеширование в .NET](https://antondevtips.com/blog/how-to-implement-caching-strategies-in-dotnet) → см. [[caching|AspNetCore/caching.md]]

### Resilience и инфраструктура

- [ ] [Retry с Polly и Microsoft Resilience](https://antondevtips.com/blog/how-to-implement-retries-and-resilience-patterns-with-polly-and-microsoft-resilience) → см. [[resilience|AspNetCore/resilience.md]]
- [ ] [Job Scheduler TickerQ — замена Quartz и Hangfire](https://antondevtips.com/blog/tickerq-the-modern-dotnet-job-scheduler-that-beats-quartz-and-hangfire) — альтернативы для hosting-background
- [ ] [Деплой в Azure с Neon Postgres и .NET Aspire](https://antondevtips.com/blog/how-to-deploy-dotnet-application-to-azure-using-neon-postgres-and-dotnet-aspire)
- [ ] [Aspire integration testing best practices](https://antondevtips.com/blog/dotnet-aspire-integration-testing-best-practices-for-distributed-applications)

### Observability

- [ ] [Старт с OpenTelemetry, Jaeger и Seq](https://antondevtips.com/blog/getting-started-with-open-telemetry-in-dotnet-with-jaeger-and-seq) → см. [[observability|Infrastructure/observability.md]]

### Язык C#

- [ ] [Новые фичи .NET 10 и C# 14](https://antondevtips.com/blog/new-features-in-dotnet-10-and-csharp-14)
- [ ] [Extension Members в C# 14](https://antondevtips.com/blog/extension-members-in-csharp14-changed-how-we-write-code-forever) → см. [[modern-features|CSharp/modern-features.md]]
- [ ] [Как писать чище код в .NET](https://antondevtips.com/blog/how-to-write-better-and-cleaner-code-in-dotnet)

### Ошибки разработчиков (**must-read**)

- [ ] [Top 10 вещей для .NET-разработчика в 2026](https://antondevtips.com/blog/top-10-things-every-dotnet-developer-needs-to-do-in-2026) — **быстрый аудит**
- [ ] [15 ошибок .NET-разработчиков](https://antondevtips.com/blog/top-15-mistakes-dotnet-developers-make-how-to-avoid-common-pitfalls)
- [ ] [15 ошибок при создании Web API](https://antondevtips.com/blog/top-15-mistakes-developers-make-when-creating-web-apis)

### Практические проекты

- [ ] [Invoice Builder на .NET с IronPDF](https://antondevtips.com/blog/how-to-build-a-production-ready-invoice-builder-in-dotnet-using-ironpdf)

---

## Milan Jovanović (milanjovanovic.tech)

Сильные материалы по архитектуре, EF Core, Semantic Kernel.

- [ ] [Semantic Search с Amazon S3 Vectors и Semantic Kernel](https://milanjovanovic.tech/blog/building-semantic-search-with-amazon-s3-vectors-and-semantic-kernel) → см. [[semantic-kernel|Infrastructure/semantic-kernel.md]]
- [ ] [Improving code quality in C# with static code analysis](https://milanjovanovic.tech/blog/improving-code-quality-in-csharp-with-static-code-analysis) → см. [[code-quality|Quality/code-quality.md]]

---

## Telegram-каналы, за которыми стоит следить

- [@csharpproglib](https://t.me/csharpproglib) — Библиотека шарписта, релизы .NET, CVE, новые библиотеки
- [@csharp_1001_notes](https://t.me/csharp_1001_notes) — короткие заметки, ловушки, паттерны
- [@sachkov_blog](https://t.me/sachkov_blog) — практический опыт, AI в коде
- [@yeahub_c_sharp_dev](https://t.me/yeahub_c_sharp_dev) — подготовка к собесам, теория

**Как фильтровать:** каналы мешают рекламу курсов, мемы, политику с техническим контентом. Раз в 1-2 недели пройтись и выдернуть релевантное — в свои заметки или в этот reading list.

---

## Официальные источники

- [dotnet/announcements](https://github.com/dotnet/announcements/issues) — релизы, security advisories, **подписаться на RSS**
- [devblogs.microsoft.com/dotnet](https://devblogs.microsoft.com/dotnet/) — главный блог команды .NET
- [csharplang](https://github.com/dotnet/csharplang/tree/main/proposals) — proposals для новых версий C#

---

## Книги (для углублённого чтения)

- **Марк Прайс. «C# 12 и .NET 8. Современная кросс-платформенная разработка»** — свежее издание (9-е или новее)
- **Jon Skeet. «C# in Depth»** — 5-е издание, для понимания «почему так устроено»
- **Andrew Lock. «ASP.NET Core in Action»** — 3-е издание
- **Steve Smith. «Architecting Modern Web Applications with ASP.NET Core and Azure»** — бесплатно от Microsoft
- **Vladimir Khorikov. «Unit Testing Principles, Practices, and Patterns»** — must-read по тестированию

---

## См. также

- [[00_overview|Learning Path]] — пошаговый план обучения
- [NET-Mastery-Hub](https://github.com/valinerosgordov/NET-Mastery-Hub) — мои собственные гайды, куда можно промоутить проработанные темы
