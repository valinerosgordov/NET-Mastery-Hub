# Паттерны и продвинутое

## Soft Delete

Свойство IsDeleted/DeletedAt. Global query filter: HasQueryFilter(o => !o.IsDeleted). SaveChanges — при Delete помечать, не удалять. IgnoreQueryFilters() для админки.

---

## TPH, TPT, TPC

| Стратегия | Таблицы | Плюсы | Минусы |
|-----------|---------|-------|--------|
| TPH | Одна, discriminator | Простота | Nullable для подтипов |
| TPT | Таблица на тип | Нормализовано | Много JOIN |
| TPC | Таблица на конкретный тип | Нет nullable | Дублирование |

---

## Repository, Unit of Work

DbContext уже UoW + Repository. Явный слой — при изоляции доменной логики от EF. Не оборачивать каждый DbSet «для единообразия».

---

## CQRS, Interceptors

**Read/Write separation**: два DbContext (primary + replica). Read — только запросы, AsNoTracking.

**Interceptors**: SaveChangesInterceptor, перехват команд. Аудит, soft delete, мультитенантность, логирование SQL.

---

## DDD Aggregate Root

Один DbSet на aggregate root. Дочерние — через навигации, нет прямого доступа извне. Транзакционная граница — один aggregate. Root инкапсулирует инварианты.
