# Loading и Tracking

## Eager, Explicit, Lazy Loading

| Тип | Когда загружаются связи | Как | Плюсы | Минусы |
|-----|------------------------|-----|-------|--------|
| **Eager** | В основном запросе | `Include()`, `ThenInclude()` | Предсказуемо, один round-trip | Cartesian explosion |
| **Explicit** | По вызову | `Entry(e).Collection(...).Load()` | Контроль | Дополнительные запросы |
| **Lazy** | При обращении к свойству | Прокси или `ILazyLoader` | Удобно | N+1, требует virtual |

### Eager Loading

```csharp
// Загрузка связей в одном запросе
var orders = await context.Orders
    .Include(o => o.Customer)
    .Include(o => o.Items)
        .ThenInclude(i => i.Product)
    .Where(o => o.Status == OrderStatus.Active)
    .ToListAsync(ct);
```

### Explicit Loading

```csharp
var order = await context.Orders.FindAsync(orderId);

// Загрузить коллекцию позже
await context.Entry(order)
    .Collection(o => o.Items)
    .LoadAsync(ct);

// Загрузить одиночную навигацию
await context.Entry(order)
    .Reference(o => o.Customer)
    .LoadAsync(ct);
```

### Lazy Loading

```csharp
// Требует: Microsoft.EntityFrameworkCore.Proxies
// Настройка: options.UseLazyLoadingProxies()
// Свойства навигации ДОЛЖНЫ быть virtual

public class Order
{
    public virtual ICollection<OrderItem> Items { get; set; } // virtual!
}

// Или через ILazyLoader (без прокси)
public class Order(ILazyLoader lazyLoader)
{
    private ICollection<OrderItem>? _items;
    public ICollection<OrderItem> Items
        => lazyLoader.Load(this, ref _items);
}
```

**Нюанс:** Lazy loading по умолчанию отключён в EF Core. И правильно — N+1 в цикле даст сотни запросов незаметно. Предпочитать Eager + проекцию.

---

## Tracking и AsNoTracking

### Change Tracker

Для каждой tracked-сущности хранит: оригинальные значения, текущие значения, состояние (Added, Modified, Deleted, Unchanged). При `SaveChanges()` — dirty checking и генерация SQL.

```csharp
// Read-only — без tracking (быстрее, меньше памяти)
var orders = await context.Orders
    .AsNoTracking()
    .Where(o => o.Total > 100)
    .ToListAsync(ct);

// Для изменений — tracking обязателен
var order = await context.Orders.FindAsync(orderId);
order.Status = OrderStatus.Completed;
await context.SaveChangesAsync(ct); // EF знает что изменилось

// Глобально отключить tracking
protected override void OnConfiguring(DbContextOptionsBuilder options)
    => options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
```

### AsNoTrackingWithIdentityResolution

```csharp
// Как AsNoTracking, но сохраняет identity map
// Полезно при Include — одинаковые объекты не дублируются
var orders = await context.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.Customer) // Customer не дублируется для разных Order
    .ToListAsync(ct);
```

**Нюанс:** tracked entity нельзя отслеживать из двух DbContext одновременно. При Detach → Attach в другой контекст — осторожно с состоянием.

---

## Include и AsSplitQuery

### Cartesian Explosion

```csharp
// ✗ Одним запросом с JOIN — декартово произведение
var orders = await context.Orders
    .Include(o => o.Items)       // 10 items
    .Include(o => o.Payments)    // 3 payments
    .ToListAsync(ct);
// SQL вернёт 10 × 3 = 30 строк на один Order!

// ✓ Split query — отдельные запросы, объединение в памяти
var orders = await context.Orders
    .Include(o => o.Items)
    .Include(o => o.Payments)
    .AsSplitQuery()              // 3 запроса вместо 1
    .ToListAsync(ct);
```

### Когда что

- **Одна коллекция, небольшой объём** → single query (меньше round-trips)
- **Несколько коллекций** → `AsSplitQuery()` (избежать explosion)
- **Глобально** → `UseQuerySplittingBehavior(QuerySplittingBehavior.SplitQuery)` в конфигурации

**Нюанс:** при `AsSplitQuery` и параллельных изменениях в БД данные могут быть inconsistent (каждый запрос видит своё состояние). Для критичных операций — single query или транзакция.

---

## Filtered Include (.NET 5+)

```csharp
var customers = await context.Customers
    .Include(c => c.Orders.Where(o => o.Status == OrderStatus.Active))
    .ToListAsync(ct);
// Загружаются только активные заказы, не все
```

**Нюанс:** фильтр применяется к навигации, не к основной сущности. Customer загрузится даже если у него нет активных заказов.

---

## Проекция — лучше Include

```csharp
// ✓ Лучший подход — загружаем только нужные поля
var result = await context.Orders
    .Where(o => o.Status == OrderStatus.Active)
    .Select(o => new OrderDto
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,   // JOIN автоматически
        ItemCount = o.Items.Count,         // подзапрос
        Total = o.Items.Sum(i => i.Price)  // агрегация в SQL
    })
    .ToListAsync(ct);
// Нет tracking, нет лишних данных, один оптимальный SQL
```

---

## См. также

- [Interview: EF Core и SQL](../../../Interview/5-ef-core-sql.md)
