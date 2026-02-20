# Performance

## N+1

Один запрос за список + N дополнительных запросов при обращении к навигациям.

```csharp
// ✗ N+1 — каждый order.Customer → отдельный запрос
var orders = await context.Orders.ToListAsync();
foreach (var order in orders)
    Console.WriteLine(order.Customer.Name); // SELECT для каждого!

// ✓ Решение 1: Include
var orders = await context.Orders
    .Include(o => o.Customer)
    .ToListAsync(ct);

// ✓ Решение 2: Проекция (лучше!)
var dtos = await context.Orders
    .Select(o => new OrderDto(o.Id, o.Customer.Name, o.Total))
    .ToListAsync(ct);

// ✓ Решение 3: Явная загрузка для конкретных случаев
var orders = await context.Orders.ToListAsync(ct);
var customerIds = orders.Select(o => o.CustomerId).Distinct();
await context.Customers.Where(c => customerIds.Contains(c.Id)).LoadAsync(ct);
// EF автоматически привяжет через Change Tracker
```

**Нюанс:** проекция в DTO (`Select`) — самый эффективный подход. Загружаются только нужные поля, нет tracking overhead, один оптимальный SQL.

---

## Оптимизация reads

### Чек-лист оптимизации

| Приём | Когда | Эффект |
|-------|-------|--------|
| `AsNoTracking()` | Все read-only запросы | -30% памяти, +20% скорость |
| `Select()` в DTO | Нужны не все поля | Меньше данных по сети |
| `AsSplitQuery()` | Несколько Include-коллекций | Нет Cartesian explosion |
| Compiled queries | Hot path, частое выполнение | Нет парсинга Expression |
| Индексы | WHERE, JOIN, ORDER BY | Из scan → seek |
| Кэширование | Часто читаемые, редко меняемые | Нет запроса к БД |
| Keyset pagination | Большие offset | Стабильная скорость |

```csharp
// Пример оптимизированного запроса
var orders = await context.Orders
    .AsNoTracking()
    .TagWith("Dashboard:RecentOrders")
    .Where(o => o.CustomerId == customerId && o.CreatedAt > cutoff)
    .OrderByDescending(o => o.CreatedAt)
    .Select(o => new OrderSummaryDto
    {
        Id = o.Id,
        Total = o.Total,
        ItemCount = o.Items.Count,
        CustomerName = o.Customer.Name
    })
    .Take(20)
    .ToListAsync(ct);
```

---

## Compiled Queries

```csharp
// Компиляция один раз — повторные вызовы без парсинга Expression
private static readonly Func<AppDbContext, Guid, CancellationToken, Task<Order?>>
    GetOrderById = EF.CompileAsyncQuery(
        (AppDbContext ctx, Guid id, CancellationToken ct) =>
            ctx.Orders
                .Include(o => o.Items)
                .FirstOrDefault(o => o.Id == id));

// Использование
var order = await GetOrderById(context, orderId, ct);
```

**Нюанс:** compiled queries экономят на построении Expression Tree и генерации SQL. Эффект заметен при:
- Высоко нагруженных путях (тысячи запросов/секунду)
- Сложных запросах (много Join, Where)

Для простых запросов EF Core уже кэширует план — эффект минимален.

---

## AutoInclude

```csharp
// Навигация загружается автоматически с каждым запросом
modelBuilder.Entity<Order>()
    .Navigation(o => o.Customer)
    .AutoInclude();

// Для owned types — AutoInclude по умолчанию

// Отключить для конкретного запроса
var orders = context.Orders
    .IgnoreAutoIncludes()
    .ToListAsync(ct);
```

**Нюанс:** AutoInclude опасен:
- Лишние данные в запросах, где навигация не нужна
- Cartesian explosion при нескольких AutoInclude
- Конфликт с проекцией (`Select`)
- Предпочитать явный `Include` — предсказуемее

---

## Пагинация

### Offset-based (простая, но деградирует)

```csharp
var page = await context.Orders
    .OrderBy(o => o.CreatedAt)
    .Skip((pageNumber - 1) * pageSize) // OFFSET — сканирует и пропускает
    .Take(pageSize)                     // LIMIT
    .ToListAsync(ct);
```

### Keyset (cursor) pagination — стабильная скорость

```csharp
var page = await context.Orders
    .OrderBy(o => o.CreatedAt)
    .ThenBy(o => o.Id) // disambiguator
    .Where(o => o.CreatedAt > lastCreatedAt
             || (o.CreatedAt == lastCreatedAt && o.Id > lastId))
    .Take(pageSize)
    .ToListAsync(ct);
```

**Нюанс:** OFFSET 100000 → БД просканирует 100000 строк и выбросит. Keyset — всегда index seek. Для API — keyset. Для UI с «перейти на страницу N» — offset (но лимитировать max page).

---

## Batch Operations (EF Core 7+)

```csharp
// Bulk update — один SQL, без загрузки сущностей
await context.Orders
    .Where(o => o.Status == OrderStatus.Expired)
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, OrderStatus.Cancelled), ct);

// Bulk delete
await context.Orders
    .Where(o => o.CreatedAt < cutoffDate)
    .ExecuteDeleteAsync(ct);

// SaveChanges batching — EF автоматически группирует команды
// Настройка: options.MaxBatchSize(100)
context.AddRange(newOrders);
await context.SaveChangesAsync(ct); // один round-trip для множества INSERT
```

---

## Диагностика

```csharp
// Логирование SQL
options.LogTo(Console.WriteLine, LogLevel.Information);

// Чувствительные данные (только dev!)
options.EnableSensitiveDataLogging();

// Детальная диагностика
options.EnableDetailedErrors();

// Счётчики в runtime
// dotnet-counters monitor Microsoft.EntityFrameworkCore
```

**Нюанс:** в production — логировать медленные запросы через `DbCommandInterceptor`:

```csharp
public class SlowQueryInterceptor : DbCommandInterceptor
{
    public override ValueTask<DbDataReader> ReaderExecutedAsync(
        DbCommand command, CommandExecutedEventData eventData,
        DbDataReader result, CancellationToken ct)
    {
        if (eventData.Duration > TimeSpan.FromSeconds(1))
            Log.Warning("Slow query ({Duration}ms): {Sql}",
                eventData.Duration.TotalMilliseconds, command.CommandText);
        return new(result);
    }
}
```

---

## См. также

- [.NET Performance](../../Performance/dotnet-performance.md)
- [SQL Optimization](../../SQL/sql-query-optimization.md)
