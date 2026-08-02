---
tags: [ef-core, queries, performance, pagination, concurrency, dapper]
level: Middle to Senior
---

# Snippets — EF Core Queries

> Производительные EF Core read-паттерны: `AsNoTracking` + проекция в DTO, offset- и keyset-пагинация, `AsSplitQuery` против Cartesian explosion, optimistic concurrency, compiled queries и Dapper для отчётов.

## Базовые best practices

```csharp
// AsNoTracking для read-only запросов — всегда
var orders = await context.Orders
    .AsNoTracking()
    .TagWith("GetOrdersByCustomer")      // видно в SQL профайлере
    .Where(o => o.CustomerId == customerId && o.Status != OrderStatus.Cancelled)
    .Select(o => new OrderSummaryDto     // проекция в DTO — не тянем лишние поля
    {
        Id = o.Id,
        CreatedAt = o.CreatedAt,
        TotalAmount = o.TotalAmount,
        Status = o.Status
    })
    .OrderByDescending(o => o.CreatedAt)
    .ToListAsync(cancellationToken)
    .ConfigureAwait(false);
```

## Пагинация — Offset (простая)

```csharp
public async Task<PagedResult<OrderSummaryDto>> GetPagedAsync(
    int page, int pageSize, CancellationToken ct)
{
    var query = context.Orders
        .AsNoTracking()
        .Where(o => o.Status == OrderStatus.Active);

    var totalCount = await query.CountAsync(ct).ConfigureAwait(false);

    var items = await query
        .OrderByDescending(o => o.CreatedAt)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(o => new OrderSummaryDto { /* ... */ })
        .ToListAsync(ct)
        .ConfigureAwait(false);

    return new PagedResult<T>(items, totalCount, page, pageSize);
}
```

## Пагинация — Keyset (производительная, для больших таблиц)

```csharp
// Передаём cursor (последний Id предыдущей страницы), не offset
public async Task<List<OrderSummaryDto>> GetNextPageAsync(
    Guid? afterId, int pageSize, CancellationToken ct)
{
    var query = context.Orders.AsNoTracking().AsQueryable();

    if (afterId.HasValue)
        query = query.Where(o => o.Id > afterId.Value); // требует сортировки по Id

    return await query
        .OrderBy(o => o.Id)
        .Take(pageSize)
        .Select(o => new OrderSummaryDto { /* ... */ })
        .ToListAsync(ct)
        .ConfigureAwait(false);
}
```

## Include без Cartesian explosion

```csharp
// ПЛОХО — два Include коллекций = декартово произведение
var bad = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments)  // ← Cartesian explosion!
    .ToListAsync(ct);

// ХОРОШО — AsSplitQuery разбивает на отдельные SQL запросы
var good = await context.Orders
    .AsSplitQuery()
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .ToListAsync(ct);
```

## Оптимистичный параллелизм

```csharp
// Entity
public sealed class Order
{
    public Guid Id { get; private set; }
    [Timestamp] public byte[] RowVersion { get; private set; } = [];
}

// Обработка конфликта
try
{
    await context.SaveChangesAsync(ct).ConfigureAwait(false);
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();
    var dbValues = await entry.GetDatabaseValuesAsync(ct).ConfigureAwait(false);

    if (dbValues is null)
        return Result.Fail<Order>(Errors.NotFound("Order", id));

    // Решаем конфликт: database wins или client wins
    entry.OriginalValues.SetValues(dbValues);
    return Result.Fail<Order>(new Error("Conflict", "Order was modified by another user"));
}
```

## Compiled Query (для горячих путей)

```csharp
// Объявляем один раз как static readonly
private static readonly Func<AppDbContext, Guid, Task<Order?>> GetByIdCompiled =
    EF.CompileAsyncQuery((AppDbContext ctx, Guid id) =>
        ctx.Orders.FirstOrDefault(o => o.Id == id));

// Используем
var order = await GetByIdCompiled(context, orderId).ConfigureAwait(false);
```

## Raw SQL через Dapper (сложные запросы)

```csharp
public async Task<List<OrderReportDto>> GetMonthlyReportAsync(
    int year, int month, CancellationToken ct)
{
    const string sql = """
        SELECT
            DATE_TRUNC('day', created_at) AS date,
            COUNT(*) AS order_count,
            SUM(total_amount) AS revenue
        FROM orders
        WHERE EXTRACT(YEAR FROM created_at) = @Year
          AND EXTRACT(MONTH FROM created_at) = @Month
          AND status = 'Completed'
        GROUP BY DATE_TRUNC('day', created_at)
        ORDER BY date
        """;

    var conn = context.Database.GetDbConnection();
    var result = await conn
        .QueryAsync<OrderReportDto>(sql, new { Year = year, Month = month })
        .ConfigureAwait(false);

    return result.AsList();
}
```

---

## См. также

- [[optimization|SQL Optimization]]
