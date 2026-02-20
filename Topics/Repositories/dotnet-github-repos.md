# Полезные .NET репозитории

## По темам

### Validation
- [Modern.FluentValidation.Extensions](https://github.com/anton-martyniuk/Modern.FluentValidation.Extensions) — кастомные валидаторы для FluentValidation (Apache 2.0)

### Result / Discriminated Unions
- [OneOf.Deconstruct](https://github.com/anton-martyniuk/OneOf.Deconstruct) — deconstruction для OneOf, F#-style unions в C#

### Modern .NET Tools
- [Modern](https://github.com/anton-martyniuk/Modern) — инструменты для быстрой разработки: services, controllers, CQRS, MongoDB (Apache 2.0)

### EF Core
- [efcore-extensions-examples](https://github.com/anton-martyniuk/efcore-extensions-examples) — примеры Entity Framework Extensions (Bulk Insert, Update, Delete)

### Architecture
- [modular-monolith-abp-framework](https://github.com/anton-martyniuk/modular-monolith-abp-framework) — Modular Monolith на ABP Framework, Shipments + Stocks, sync/async, MVC Razor

### Full-stack
- [invoice-builder](https://github.com/anton-martyniuk/invoice-builder) — генератор инвойсов: ASP.NET Core 10, EF Core, Postgres, React, TypeScript, TailwindCSS, IronPDF (MIT)

---

## Best Practices

- **Modern** — смотреть структуру и паттерны. Не копировать слепо — адаптировать под свой домен.
- **FluentValidation.Extensions** — использовать кастомные валидаторы вместо дублирования правил.
- **modular-monolith-abp** — референс по границам модулей. ABP — тяжёлый фреймворк, идеи применимы и без него.
- **efcore-extensions** — BulkInsert для миграций и batch-операций. Не для обычного CRUD.

---

## См. также

- [[Architecture/architecture-tutorial|Архитектуры]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
