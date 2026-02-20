# Паттерны и продвинутое

## Soft Delete

Логическое удаление — пометка вместо физического DELETE. Данные остаются для аудита, восстановления.

```csharp
// Модель
public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
}

public class Order : ISoftDeletable
{
    public Guid Id { get; set; }
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
}

// Global Query Filter — автоматически скрывает удалённые
modelBuilder.Entity<Order>()
    .HasQueryFilter(o => !o.IsDeleted);

// Interceptor для автоматического soft delete
public class SoftDeleteInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken ct)
    {
        var context = eventData.Context!;
        foreach (var entry in context.ChangeTracker.Entries<ISoftDeletable>()
            .Where(e => e.State == EntityState.Deleted))
        {
            entry.State = EntityState.Modified;
            entry.Entity.IsDeleted = true;
            entry.Entity.DeletedAt = DateTime.UtcNow;
        }
        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// Регистрация
options.AddInterceptors(new SoftDeleteInterceptor());

// Для админки — показать удалённые
var allOrders = await context.Orders
    .IgnoreQueryFilters()
    .ToListAsync(ct);
```

**Нюанс:** soft delete + unique index → конфликт (удалённая запись блокирует уникальное значение). Решение: filtered unique index `WHERE IsDeleted = 0`.

---

## TPH, TPT, TPC

Стратегии маппинга наследования на таблицы.

### TPH (Table Per Hierarchy) — default

Одна таблица, дискриминатор. Самая быстрая стратегия.

```csharp
public abstract class Payment
{
    public Guid Id { get; set; }
    public decimal Amount { get; set; }
}
public class CreditCardPayment : Payment
{
    public string CardNumber { get; set; } = null!;
}
public class BankTransferPayment : Payment
{
    public string IBAN { get; set; } = null!;
}

// Конфигурация (TPH — по умолчанию)
modelBuilder.Entity<Payment>()
    .HasDiscriminator<string>("PaymentType")
    .HasValue<CreditCardPayment>("CreditCard")
    .HasValue<BankTransferPayment>("BankTransfer");
```

### TPT (Table Per Type)

Отдельная таблица для каждого типа. JOIN при запросе.

```csharp
modelBuilder.Entity<CreditCardPayment>().ToTable("CreditCardPayments");
modelBuilder.Entity<BankTransferPayment>().ToTable("BankTransferPayments");
```

### TPC (Table Per Concrete type) — EF Core 7+

Отдельная таблица для каждого конкретного типа. Без JOIN, но дублирование базовых полей.

```csharp
modelBuilder.Entity<Payment>().UseTpcMappingStrategy();
modelBuilder.Entity<CreditCardPayment>().ToTable("CreditCardPayments");
modelBuilder.Entity<BankTransferPayment>().ToTable("BankTransferPayments");
```

| Стратегия | Таблицы | Запрос подтипа | Запрос базового | Nullable |
|-----------|---------|----------------|-----------------|----------|
| **TPH** | 1 | Быстро (filter) | Быстро | Да (подтипы) |
| **TPT** | N+1 | Быстро | Медленно (JOIN) | Нет |
| **TPC** | N | Быстро | Средне (UNION) | Нет |

**Нюанс:** TPH — лучший по умолчанию для производительности. TPT — если много подтипов с уникальными полями и нужна нормализация. TPC — компромисс.

---

## Repository и Unit of Work

### DbContext уже является UoW + Repository

```csharp
// DbContext = Unit of Work (SaveChanges — одна транзакция)
// DbSet<T> = Repository (Add, Remove, Find, LINQ)

// ✗ Тонкая обёртка без добавленной ценности
public class OrderRepository
{
    private readonly AppDbContext _context;
    public Task<Order?> GetByIdAsync(Guid id)
        => _context.Orders.FindAsync(id).AsTask(); // просто проксирует DbSet
}

// ✓ Полезный репозиторий — инкапсулирует сложную логику
public class OrderRepository(AppDbContext context) : IOrderRepository
{
    public async Task<Order?> GetWithItemsAsync(Guid id, CancellationToken ct)
        => await context.Orders
            .Include(o => o.Items)
            .TagWith("OrderRepo:GetWithItems")
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<OrderSummary>> GetDashboardAsync(
        Guid customerId, CancellationToken ct)
        => await context.Orders
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId)
            .Select(o => new OrderSummary(o.Id, o.Total, o.Items.Count))
            .ToListAsync(ct);
}
```

**Когда репозиторий уместен:**
- Сложная доменная логика доступа к данным (DDD Aggregates)
- Необходимость смены ORM (EF → Dapper)
- Тестируемость (мок интерфейса репозитория)

---

## CQRS с EF Core

```csharp
// Read DbContext — реплика, AsNoTracking, оптимизирован для чтения
public class ReadDbContext(DbContextOptions<ReadDbContext> options) : DbContext(options)
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<OrderReadModel>().ToView("vw_Orders"); // View вместо Table
    }
}

// Write DbContext — primary, tracking, полная модель
public class WriteDbContext(DbContextOptions<WriteDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
}

// Регистрация
builder.Services.AddDbContext<WriteDbContext>(o => o.UseNpgsql(primaryConnStr));
builder.Services.AddDbContext<ReadDbContext>(o => o.UseNpgsql(replicaConnStr));
```

---

## Interceptors

Перехват операций EF для cross-cutting concerns.

```csharp
// Аудит — автоматическое заполнение CreatedAt/UpdatedAt
public class AuditInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken ct)
    {
        var context = eventData.Context!;
        var now = DateTime.UtcNow;

        foreach (var entry in context.ChangeTracker.Entries<IAuditable>())
        {
            if (entry.State == EntityState.Added)
                entry.Entity.CreatedAt = now;

            if (entry.State is EntityState.Added or EntityState.Modified)
                entry.Entity.UpdatedAt = now;
        }
        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// Регистрация
options.AddInterceptors(new AuditInterceptor(), new SoftDeleteInterceptor());
```

---

## DDD Aggregate Root

Один `DbSet` на aggregate root. Дочерние сущности — через навигации, без прямого доступа.

```csharp
// Aggregate Root
public class Order
{
    public Guid Id { get; private set; }
    private readonly List<OrderItem> _items = [];
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();

    public void AddItem(Guid productId, int quantity, decimal price)
    {
        if (quantity <= 0) throw new DomainException("Invalid quantity");
        _items.Add(new OrderItem(productId, quantity, price));
    }

    public decimal Total => _items.Sum(i => i.Price * i.Quantity);
}

// Конфигурация — backing field
modelBuilder.Entity<Order>(entity =>
{
    entity.HasMany(o => o.Items)
        .WithOne()
        .HasForeignKey("OrderId");

    entity.Navigation(o => o.Items)
        .UsePropertyAccessMode(PropertyAccessMode.Field); // _items
});
```

**Нюанс:** транзакционная граница — один aggregate. Межагрегатное взаимодействие — через Domain Events, не прямые ссылки. Один `SaveChanges` = один aggregate.

---

## Value Converters

```csharp
// Конвертация типов при чтении/записи в БД
modelBuilder.Entity<Order>()
    .Property(o => o.Tags)
    .HasConversion(
        v => JsonSerializer.Serialize(v, (JsonSerializerOptions?)null),
        v => JsonSerializer.Deserialize<List<string>>(v, (JsonSerializerOptions?)null)!
    );

// Enum → string
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>();

// Strongly-typed ID
modelBuilder.Entity<Order>()
    .Property(o => o.Id)
    .HasConversion(
        id => id.Value,
        value => new OrderId(value));
```

---

## См. также

- [Interview: Architecture](../../../Interview/7-architecture.md)
- [Interview: EF Core и SQL](../../../Interview/5-ef-core-sql.md)
