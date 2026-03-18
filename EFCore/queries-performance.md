---
tags: [efcore, queries, performance, n-plus-one, compiled-queries]
level: Senior
---

# Запросы и Performance

## Что это, зачем и когда

### Главное правило EF Core запросов
**Фильтруй В SQL, не в C#.** Каждый `.Where()`, `.Select()`, `.Take()` на `IQueryable` превращается в SQL. Но если вызвать `.ToList()` РАНЬШЕ фильтрации — вся таблица загрузится в память.

```csharp
// УЖАСНО: загружает ВСЮ таблицу, фильтрует в C#
var all = await _db.Orders.ToListAsync();              // SELECT * FROM Orders (ВСЁ!)
var active = all.Where(o => o.Status == "Active");     // фильтр в памяти

// ПРАВИЛЬНО: фильтрует В SQL
var active = await _db.Orders
    .Where(o => o.Status == "Active")                  // WHERE Status = 'Active'
    .ToListAsync();
```

### Когда что?

| Задача | Метод | Почему |
|--------|-------|--------|
| Проверить «есть ли хоть один?» | `AnyAsync()` | Останавливается на первом, быстрее Count |
| Получить один элемент по ID | `FindAsync(id)` | Сначала ищет в Change Tracker (без SQL) |
| Получить первый подходящий | `FirstOrDefaultAsync()` | `TOP 1` — быстрый |
| Убедиться что ровно один | `SingleOrDefaultAsync()` | `TOP 2` — проверяет уникальность |
| Только нужные столбцы | `.Select(o => new Dto{...})` | Не тащит лишние данные |
| Массовое обновление | `ExecuteUpdateAsync()` | Без загрузки в память (.NET 7+) |
| Массовое удаление | `ExecuteDeleteAsync()` | Без загрузки в память (.NET 7+) |

---

> [!question]- **Интервью: N+1 — суть и решения?**
> N+1: 1 запрос на основную сущность + N запросов на связанные. Решения: `Include()` (eager), `AsSplitQuery()`, проекция (Select → DTO), compiled queries для hot path.

> [!question]- **Интервью: First vs Single — разница?**
> `First()` — первый элемент, `TOP 1`. `Single()` — единственный элемент, `TOP 2` (проверяет уникальность). `Single` бросит исключение если >1 элемент. Для поиска по ID — `Single`. Для "любой подходящий" — `First`.

## Raw SQL

### FromSqlRaw / FromSqlInterpolated

Для запросов, возвращающих сущности. Результат можно комбинировать с LINQ (Where, OrderBy, Include).

```csharp
// ✓ Параметризованный запрос (безопасно)
var orders = await context.Orders
    .FromSqlInterpolated($"SELECT * FROM Orders WHERE Total > {minTotal}")
    .Where(o => o.Status == OrderStatus.Active) // LINQ поверх raw SQL
    .OrderBy(o => o.CreatedAt)
    .ToListAsync(ct);

// ✓ FromSqlRaw с параметрами
var orders = await context.Orders
    .FromSqlRaw("SELECT * FROM Orders WHERE CustomerId = {0}", customerId)
    .ToListAsync(ct);

// ✗ НИКОГДА — конкатенация строк → SQL injection
var orders = context.Orders
    .FromSqlRaw($"SELECT * FROM Orders WHERE Name = '{name}'"); // ОПАСНО!
```

### ExecuteSqlRawAsync

Для INSERT/UPDATE/DELETE без возврата сущностей:

```csharp
// Bulk update без загрузки в память
var affected = await context.Database.ExecuteSqlInterpolatedAsync(
    $"UPDATE Orders SET Status = {newStatus} WHERE CustomerId = {customerId}", ct);
```

### ExecuteUpdate / ExecuteDelete (EF Core 7+)

```csharp
// Типизированный bulk update — без загрузки сущностей
await context.Orders
    .Where(o => o.Status == OrderStatus.Expired)
    .ExecuteUpdateAsync(s => s
        .SetProperty(o => o.Status, OrderStatus.Cancelled)
        .SetProperty(o => o.UpdatedAt, DateTime.UtcNow), ct);

// Типизированный bulk delete
await context.Orders
    .Where(o => o.CreatedAt < cutoffDate)
    .ExecuteDeleteAsync(ct);
```

**Нюанс:** `ExecuteUpdate`/`ExecuteDelete` — не проходят через Change Tracker. Tracked-сущности не обновятся автоматически. Выполняется одним SQL без загрузки в память.

---

## First vs Single, Client vs Server

### First vs Single

| Метод | SQL | Поведение |
|-------|-----|-----------|
| `FirstOrDefault()` | `TOP 1` / `LIMIT 1` | Первый или default |
| `First()` | `TOP 1` | Первый или `InvalidOperationException` |
| `SingleOrDefault()` | `TOP 2` | Один или default. >1 → exception |
| `Single()` | `TOP 2` | Ровно один. 0 или >1 → exception |

```csharp
// Поиск по PK — FindAsync (кэширует в Change Tracker)
var order = await context.Orders.FindAsync(orderId);

// Поиск по условию — FirstOrDefault
var order = await context.Orders
    .FirstOrDefaultAsync(o => o.OrderNumber == number, ct);

// Гарантия уникальности — Single
var user = await context.Users
    .SingleOrDefaultAsync(u => u.Email == email, ct);
```

**Нюанс:** `FindAsync` сначала ищет в Change Tracker (без запроса к БД). Если сущность tracked — возвращает её. `FirstOrDefault` всегда идёт в БД.

### Server-side vs Client-side Evaluation

```csharp
// ✓ Server-side — EF переводит в SQL
var orders = context.Orders.Where(o => o.Total > 100);

// ✗ Client-side — загрузка ВСЕХ данных в память
var orders = context.Orders.Where(o => MyCustomMethod(o.Name));
// EF не может перевести MyCustomMethod в SQL → загружает всё

// Проверка: включить логирование warnings
options.ConfigureWarnings(w =>
    w.Throw(RelationalEventId.QueryPossibleUnintendedUseOfEqualsWarning));
```

**Нюанс:** EF Core 3+ по умолчанию бросает исключение при client-side evaluation в Where. В Select — допускается (данные уже загружены). Всегда проверять сгенерированный SQL через логи.

---

## Expression Trees в EF

LINQ-запрос компилируется в `Expression<Func<T, bool>>`. EF обходит дерево выражений и строит SQL.

```csharp
// Динамические фильтры через Expression
public static class QueryExtensions
{
    public static IQueryable<T> WhereIf<T>(
        this IQueryable<T> query,
        bool condition,
        Expression<Func<T, bool>> predicate)
        => condition ? query.Where(predicate) : query;
}

// Использование — фильтры применяются по условию
var orders = context.Orders
    .WhereIf(customerId.HasValue, o => o.CustomerId == customerId.Value)
    .WhereIf(minTotal.HasValue, o => o.Total >= minTotal.Value)
    .WhereIf(!string.IsNullOrEmpty(status), o => o.Status == Enum.Parse<OrderStatus>(status))
    .OrderByDescending(o => o.CreatedAt)
    .Take(pageSize)
    .ToListAsync(ct);
```

### Спецификации (Specification Pattern)

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity)
        => ToExpression().Compile()(entity);
}

public class ActiveOrderSpec : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression()
        => o => o.Status == OrderStatus.Active && !o.IsDeleted;
}

// Применение
var spec = new ActiveOrderSpec();
var orders = await context.Orders
    .Where(spec.ToExpression())
    .ToListAsync(ct);
```

---

## Query Tags

```csharp
var orders = await context.Orders
    .TagWith("GetActiveOrdersByCustomer")  // комментарий в SQL
    .TagWithCallSite()                      // + имя файла и строка (.NET 6+)
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(ct);

// В логах SQL Server / PostgreSQL:
// -- GetActiveOrdersByCustomer
// -- file: OrderRepository.cs:42
// SELECT * FROM Orders WHERE CustomerId = @p0
```

**Нюанс:** `TagWith` + `TagWithCallSite` — must-have для production. Позволяет быстро найти в логах БД, какой код сгенерировал медленный запрос.

---

## Global Query Filters

```csharp
modelBuilder.Entity<Order>()
    .HasQueryFilter(o => !o.IsDeleted);              // soft delete

modelBuilder.Entity<Order>()
    .HasQueryFilter(o => o.TenantId == _tenantId);   // мультитенантность

// Отключение для конкретного запроса
var allOrders = await context.Orders
    .IgnoreQueryFilters()
    .ToListAsync(ct);
```

**Нюанс:** фильтры стекируются — если два фильтра, оба применяются. Но EF не поддерживает несколько `HasQueryFilter` — используйте `&&`:

```csharp
entity.HasQueryFilter(o => !o.IsDeleted && o.TenantId == _tenantId);
```

---

## См. также

- [Interview: EF Core и SQL](../../../Interview/5-ef-core-sql.md)
- [SQL Optimization](../../SQL/sql-query-optimization.md)
