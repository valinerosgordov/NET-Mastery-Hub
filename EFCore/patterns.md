---
tags: [efcore, patterns, repository, soft-delete, audit, specification, bulk-operations, multi-tenant, cqrs]
level: Senior
date: 2026-04-30
---

# EF Core Patterns

> Production-grade паттерны на EF Core. Закрывает: Repository / Unit of Work, Soft Delete, Audit interceptors, Specification pattern, Bulk operations (EFCore.BulkExtensions, ExecuteUpdate/Delete), Multi-Tenancy (RLS, query filters, separate DBs), CQRS с EF, Domain Events publishing, Outbox pattern integration.

---

## Что это, зачем и когда

### Обзор паттернов

| Паттерн | Что делает | Когда нужен |
|---------|-----------|-------------|
| **Soft Delete** | Логическое удаление через флаг | Аудит, восстановление, юр. требования |
| **TPH/TPT/TPC** | Маппинг наследования | Иерархии типов в домене |
| **Repository** | Абстракция над DbContext | Тестируемость, замена ORM (редко нужно) |
| **Unit of Work** | Атомарная группа изменений | Несколько репозиториев в одной TX |
| **Specification** | Encapsulate query logic | Сложные queries с переиспользованием |
| **Audit Interceptors** | Auto CreatedAt/UpdatedAt/UpdatedBy | Compliance, debugging, change tracking |
| **Value Converters** | C# тип ↔ SQL тип | Strongly-typed IDs, encryption, JSON |
| **Global Query Filters** | Авто-WHERE для всех queries | Soft Delete, Multi-tenancy |
| **Domain Events** | Публикация событий доменной логики | Decoupling, side effects, CQRS |
| **Outbox** | Atomic commit + reliable messaging | Distributed systems |
| **Bulk Operations** | INSERT/UPDATE/DELETE миллионов записей | ETL, миграции, очистки |

---

## Soft Delete

Логическое удаление — пометка вместо физического DELETE. Данные остаются для аудита, восстановления.

### Базовая реализация

```csharp
// Marker interface
public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTime? DeletedAt { get; set; }
    string? DeletedBy { get; set; }
}

public class Order : ISoftDeletable
{
    public Guid Id { get; set; }
    public bool IsDeleted { get; set; }
    public DateTime? DeletedAt { get; set; }
    public string? DeletedBy { get; set; }
}
```

### Global Query Filter — авто-исключение

```csharp
modelBuilder.Entity<Order>()
    .HasQueryFilter(o => !o.IsDeleted);

// Все queries теперь автоматически: WHERE IsDeleted = false
var orders = await context.Orders.ToListAsync();  // только живые
```

### Auto soft-delete через interceptor

```csharp
public class SoftDeleteInterceptor(ICurrentUserService currentUser) 
    : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        if (eventData.Context is null) return base.SavingChangesAsync(eventData, result, ct);

        foreach (var entry in eventData.Context.ChangeTracker.Entries<ISoftDeletable>()
                     .Where(e => e.State == EntityState.Deleted))
        {
            entry.State = EntityState.Modified;
            entry.Entity.IsDeleted = true;
            entry.Entity.DeletedAt = DateTime.UtcNow;
            entry.Entity.DeletedBy = currentUser.UserId?.ToString();
        }

        return base.SavingChangesAsync(eventData, result, ct);
    }
}

// Регистрация
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    var currentUser = sp.GetRequiredService<ICurrentUserService>();
    options
        .UseNpgsql(connStr)
        .AddInterceptors(new SoftDeleteInterceptor(currentUser));
});

// Использование — обычный Remove делает soft delete
context.Orders.Remove(order);
await context.SaveChangesAsync();
// SQL: UPDATE Orders SET IsDeleted=true, DeletedAt='...' WHERE Id=...
```

### Включить удалённые в query

```csharp
// Для админки / restore feature
var allIncludingDeleted = await context.Orders
    .IgnoreQueryFilters()
    .ToListAsync();

var deletedOnly = await context.Orders
    .IgnoreQueryFilters()
    .Where(o => o.IsDeleted)
    .ToListAsync();
```

### Pitfalls Soft Delete

#### 1. Unique index конфликтует

```sql
CREATE UNIQUE INDEX ux_users_email ON Users (Email);
```

Soft-deleted user с email `john@example.com` — никто другой не может зарегистрироваться с этим email!

**Решение: filtered unique index**

```csharp
modelBuilder.Entity<User>()
    .HasIndex(u => u.Email)
    .IsUnique()
    .HasFilter("\"IsDeleted\" = false");

// SQL: CREATE UNIQUE INDEX ... WHERE IsDeleted = false
```

#### 2. FK к удалённой записи

Order ссылается на soft-deleted Customer. EF не знает что Customer "удалён" → загружает.

**Решение:** Filter на навигации тоже:

```csharp
modelBuilder.Entity<Order>()
    .HasQueryFilter(o => !o.IsDeleted && !o.Customer.IsDeleted);
```

#### 3. Cascade soft delete

```csharp
public class SoftDeleteInterceptor : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(...)
    {
        foreach (var entry in entries)
        {
            entry.State = EntityState.Modified;
            entry.Entity.IsDeleted = true;
            
            // Cascade — soft-delete детей aggregate
            if (entry.Entity is Order order)
            {
                foreach (var item in order.Items)
                {
                    context.Entry(item).Property("IsDeleted").CurrentValue = true;
                }
            }
        }
        // ...
    }
}
```

#### 4. Permanent purge job

Soft-deleted данные накапливаются. Нужен фоновый job для физического удаления старых записей:

```csharp
public class PurgeOldDeletedJob : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var period = new PeriodicTimer(TimeSpan.FromDays(1));
        while (await period.WaitForNextTickAsync(ct))
        {
            using var scope = serviceProvider.CreateScope();
            var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            
            var cutoff = DateTime.UtcNow.AddDays(-90);
            await context.Orders
                .IgnoreQueryFilters()
                .Where(o => o.IsDeleted && o.DeletedAt < cutoff)
                .ExecuteDeleteAsync(ct);  // hard delete
        }
    }
}
```

---

## Audit Interceptors

### Auto CreatedAt / UpdatedAt / UpdatedBy

```csharp
public interface IAuditable
{
    DateTime CreatedAt { get; set; }
    string? CreatedBy { get; set; }
    DateTime? UpdatedAt { get; set; }
    string? UpdatedBy { get; set; }
}

public class AuditInterceptor(ICurrentUserService user, TimeProvider timeProvider) 
    : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData,
        InterceptionResult<int> result,
        CancellationToken ct = default)
    {
        if (eventData.Context is null) return base.SavingChangesAsync(eventData, result, ct);

        var now = timeProvider.GetUtcNow().UtcDateTime;
        var userId = user.UserId?.ToString() ?? "system";

        foreach (var entry in eventData.Context.ChangeTracker.Entries<IAuditable>())
        {
            switch (entry.State)
            {
                case EntityState.Added:
                    entry.Entity.CreatedAt = now;
                    entry.Entity.CreatedBy = userId;
                    break;
                case EntityState.Modified:
                    entry.Entity.UpdatedAt = now;
                    entry.Entity.UpdatedBy = userId;
                    // Защита от изменения CreatedAt
                    entry.Property(nameof(IAuditable.CreatedAt)).IsModified = false;
                    entry.Property(nameof(IAuditable.CreatedBy)).IsModified = false;
                    break;
            }
        }

        return base.SavingChangesAsync(eventData, result, ct);
    }
}
```

### Audit log таблица — full change tracking

Записываем каждое изменение в отдельную таблицу:

```csharp
public class AuditLog
{
    public Guid Id { get; set; }
    public string EntityType { get; set; } = "";
    public string EntityId { get; set; } = "";
    public string Action { get; set; } = "";  // Insert / Update / Delete
    public string? OldValues { get; set; }    // JSON
    public string? NewValues { get; set; }    // JSON
    public string? UserId { get; set; }
    public DateTime Timestamp { get; set; }
}

public class AuditLogInterceptor(ICurrentUserService user) : SaveChangesInterceptor
{
    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct = default)
    {
        if (eventData.Context is not AppDbContext context) return result;

        var auditLogs = new List<AuditLog>();

        foreach (var entry in context.ChangeTracker.Entries()
                     .Where(e => e.Entity is not AuditLog &&
                                 e.State is EntityState.Added or 
                                            EntityState.Modified or 
                                            EntityState.Deleted))
        {
            var log = new AuditLog
            {
                Id = Guid.NewGuid(),
                EntityType = entry.Entity.GetType().Name,
                EntityId = entry.Properties.First(p => p.Metadata.IsPrimaryKey()).CurrentValue?.ToString() ?? "",
                Action = entry.State.ToString(),
                Timestamp = DateTime.UtcNow,
                UserId = user.UserId?.ToString(),
                OldValues = entry.State != EntityState.Added 
                    ? JsonSerializer.Serialize(entry.OriginalValues.ToObject()) 
                    : null,
                NewValues = entry.State != EntityState.Deleted 
                    ? JsonSerializer.Serialize(entry.CurrentValues.ToObject()) 
                    : null
            };
            
            auditLogs.Add(log);
        }

        if (auditLogs.Count > 0)
        {
            context.Set<AuditLog>().AddRange(auditLogs);
            await context.SaveChangesAsync(ct);  // recursive save — без interceptor (через AsNoTracking?)
        }

        return result;
    }
}
```

> [!warning] Recursive SaveChanges
> Вызов SaveChanges из интерсептора → recursion. Защита: использовать отдельный DbContext или флаг.

---

## Specification Pattern

Encapsulate query logic в reusable объекты:

```csharp
public abstract class Specification<T>
{
    public Expression<Func<T, bool>> Criteria { get; protected set; } = _ => true;
    public List<Expression<Func<T, object>>> Includes { get; } = [];
    public Expression<Func<T, object>>? OrderBy { get; protected set; }
    public Expression<Func<T, object>>? OrderByDescending { get; protected set; }
    public int? Take { get; protected set; }
    public int? Skip { get; protected set; }
    public bool AsNoTracking { get; protected set; }
}

public class ActiveOrdersForCustomer : Specification<Order>
{
    public ActiveOrdersForCustomer(Guid customerId)
    {
        Criteria = o => o.CustomerId == customerId && o.Status == OrderStatus.Active;
        Includes.Add(o => o.Items);
        OrderByDescending = o => o.CreatedAt;
        Take = 10;
        AsNoTracking = true;
    }
}

// Evaluator — применяет spec к IQueryable
public static class SpecificationEvaluator
{
    public static IQueryable<T> ApplySpecification<T>(
        IQueryable<T> query, Specification<T> spec) where T : class
    {
        if (spec.AsNoTracking) query = query.AsNoTracking();
        
        query = query.Where(spec.Criteria);
        
        foreach (var include in spec.Includes)
            query = query.Include(include);
        
        if (spec.OrderBy is not null) query = query.OrderBy(spec.OrderBy);
        if (spec.OrderByDescending is not null) query = query.OrderByDescending(spec.OrderByDescending);
        
        if (spec.Skip.HasValue) query = query.Skip(spec.Skip.Value);
        if (spec.Take.HasValue) query = query.Take(spec.Take.Value);
        
        return query;
    }
}

// Использование
var spec = new ActiveOrdersForCustomer(customerId);
var orders = await SpecificationEvaluator
    .ApplySpecification(context.Orders, spec)
    .ToListAsync();
```

### Composable specs

```csharp
public class AndSpecification<T>(Specification<T> a, Specification<T> b) : Specification<T>
{
    public new Expression<Func<T, bool>> Criteria => 
        a.Criteria.AndAlso(b.Criteria);
}

// Combine: активные + дорогие
var spec = new ActiveOrdersSpec().And(new HighValueOrdersSpec(threshold: 1000));
```

### Когда Specification

✅ **Хорошо для:**
- Сложная query logic, переиспользуемая в нескольких местах
- DDD — encapsulate domain-specific queries
- Тестируемость — spec тестируется отдельно

❌ **Не нужно для:**
- Простых CRUD queries — overengineering
- Когда есть Repository с готовыми методами

См. также: библиотека Ardalis.Specification.

---

## Repository и Unit of Work

### DbContext = UoW + Repository

```csharp
// DbContext = Unit of Work (SaveChanges — одна транзакция)
// DbSet<T> = Repository (Add, Remove, Find, LINQ)

// ❌ Тонкая обёртка без ценности
public class OrderRepository
{
    private readonly AppDbContext _context;
    public Task<Order?> GetByIdAsync(Guid id) => _context.Orders.FindAsync(id).AsTask();
}
```

### Полезный Repository — encapsulates complex logic

```csharp
public interface IOrderRepository
{
    Task<Order?> GetWithItemsAsync(Guid id, CancellationToken ct);
    Task<IReadOnlyList<OrderSummary>> GetCustomerDashboardAsync(Guid customerId, CancellationToken ct);
    Task<int> BulkCancelByCustomerAsync(Guid customerId, CancellationToken ct);
}

public class OrderRepository(AppDbContext context) : IOrderRepository
{
    public async Task<Order?> GetWithItemsAsync(Guid id, CancellationToken ct)
        => await context.Orders
            .Include(o => o.Items).ThenInclude(i => i.Product)
            .Include(o => o.Customer)
            .TagWith("OrderRepo:GetWithItems")
            .FirstOrDefaultAsync(o => o.Id == id, ct);

    public async Task<IReadOnlyList<OrderSummary>> GetCustomerDashboardAsync(
        Guid customerId, CancellationToken ct)
        => await context.Orders
            .AsNoTracking()
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.CreatedAt)
            .Take(20)
            .Select(o => new OrderSummary(
                o.Id, 
                o.Total, 
                o.Status, 
                o.Items.Count, 
                o.CreatedAt))
            .ToListAsync(ct);

    public async Task<int> BulkCancelByCustomerAsync(Guid customerId, CancellationToken ct)
        => await context.Orders
            .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Pending)
            .ExecuteUpdateAsync(s => s
                .SetProperty(o => o.Status, OrderStatus.Cancelled)
                .SetProperty(o => o.CancelledAt, DateTime.UtcNow), ct);
}
```

### Когда Repository уместен

✅ **Полезен когда:**
- Сложная доменная логика доступа к данным (DDD Aggregates)
- Тестируемость — мок интерфейса репозитория
- Возможная замена ORM (EF → Dapper)
- Цепочка специфичных queries которые иначе будут дублироваться

❌ **Не нужен когда:**
- Простой CRUD — DbContext достаточен
- В Vertical Slices — каждый Handler работает напрямую с DbContext

### TagWith — для query observability

```csharp
var order = await context.Orders
    .TagWith("OrderRepo:GetWithItems by user " + userId)
    .Where(o => o.Id == id)
    .FirstOrDefaultAsync();

// SQL: -- OrderRepo:GetWithItems by user abc
//      SELECT ...
```

В логах БД видно от какого репо/метода пришёл query. Полезно для performance debugging.

---

## CQRS с EF Core

### Read и Write Contexts

```csharp
// Write — primary, tracking, full model
public class WriteDbContext(DbContextOptions<WriteDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(OrderConfiguration).Assembly);
    }
}

// Read — replica, NoTracking globally, view-based
public class ReadDbContext(DbContextOptions<ReadDbContext> options) : DbContext(options)
{
    public DbSet<OrderListItem> OrderList => Set<OrderListItem>();
    public DbSet<OrderDetails> OrderDetails => Set<OrderDetails>();

    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking);
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<OrderListItem>().ToView("vw_OrderList");
        modelBuilder.Entity<OrderDetails>().ToView("vw_OrderDetails");
    }
}

// Регистрация
builder.Services.AddDbContext<WriteDbContext>(o => 
    o.UseNpgsql(primaryConnStr));
builder.Services.AddDbContextPool<ReadDbContext>(o => 
    o.UseNpgsql(replicaConnStr));
```

### Database Views для Read Models

```sql
-- migration или DDL
CREATE VIEW vw_OrderList AS
SELECT 
    o.Id,
    o.OrderNumber,
    c.Name AS CustomerName,
    o.Total,
    o.Status,
    COUNT(oi.Id) AS ItemCount,
    o.CreatedAt
FROM Orders o
JOIN Customers c ON o.CustomerId = c.Id
LEFT JOIN OrderItems oi ON oi.OrderId = o.Id
WHERE o.IsDeleted = false
GROUP BY o.Id, o.OrderNumber, c.Name, o.Total, o.Status, o.CreatedAt;
```

```csharp
[Keyless]
public class OrderListItem
{
    public Guid Id { get; set; }
    public string OrderNumber { get; set; } = "";
    public string CustomerName { get; set; } = "";
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; }
    public int ItemCount { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

Преимущества:
- Read-side оптимизирован SQL'ом DBA
- Нет N+1, нет cartesian — view denormalizes
- Read context может работать на replica

---

## Multi-Tenancy

### Стратегии

| Стратегия | Database | Schema | Когда |
|-----------|----------|--------|-------|
| **Database per tenant** | По одной | Свой | Strong isolation, compliance, paid SaaS |
| **Schema per tenant** | Общая | Per-tenant schema | Mid isolation |
| **Shared DB + TenantId column** | Общая | Общая, фильтрация по TenantId | Maximum density, low cost |
| **Postgres RLS** | Общая | Общая, RLS policies | Best of shared + isolation |

### Stratrgy: Shared DB + TenantId + Query Filter

```csharp
public interface ITenantProvider
{
    Guid TenantId { get; }
}

public class HttpTenantProvider(IHttpContextAccessor accessor) : ITenantProvider
{
    public Guid TenantId => 
        Guid.Parse(accessor.HttpContext?.User.FindFirst("TenantId")?.Value 
                   ?? throw new InvalidOperationException("No tenant"));
}

public class Order
{
    public Guid Id { get; set; }
    public Guid TenantId { get; set; }  // ← важно
    // ...
}

// DbContext с tenant filter
public class AppDbContext(
    DbContextOptions<AppDbContext> options,
    ITenantProvider tenantProvider) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Global filter — все queries фильтруются по TenantId
        modelBuilder.Entity<Order>()
            .HasQueryFilter(o => o.TenantId == tenantProvider.TenantId);
    }

    public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Auto-fill TenantId при INSERT
        foreach (var entry in ChangeTracker.Entries<Order>().Where(e => e.State == EntityState.Added))
        {
            entry.Entity.TenantId = tenantProvider.TenantId;
        }
        return await base.SaveChangesAsync(ct);
    }
}
```

### Strategy: Postgres Row-Level Security

См. подробно [PostgreSQL Deep — RLS](../SQL/postgresql-deep.md).

Кратко: filter на уровне БД, не на уровне C#:

```sql
ALTER TABLE Orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON Orders
    USING (tenant_id = current_setting('app.tenant_id')::uuid);
```

В C#: устанавливаем `app.tenant_id` через connection interceptor:

```csharp
public class TenantConnectionInterceptor(ITenantProvider provider) : DbConnectionInterceptor
{
    public override async ValueTask<InterceptionResult> ConnectionOpeningAsync(
        DbConnection connection,
        ConnectionEventData eventData,
        InterceptionResult result,
        CancellationToken ct = default)
    {
        return result;
    }

    public override async Task ConnectionOpenedAsync(
        DbConnection connection,
        ConnectionEndEventData eventData,
        CancellationToken ct = default)
    {
        await using var cmd = connection.CreateCommand();
        cmd.CommandText = $"SET app.tenant_id = '{provider.TenantId}'";
        await cmd.ExecuteNonQueryAsync(ct);
    }
}
```

**Преимущества RLS:**
- Невозможно случайно загрузить чужие данные (даже если забыл WHERE)
- Защита от SQL injection в multi-tenant
- DBA-уровень contract

### Strategy: Database per tenant

```csharp
public class TenantConnectionStringResolver
{
    private readonly Dictionary<Guid, string> _connStrings;
    
    public TenantConnectionStringResolver(IConfiguration config)
    {
        _connStrings = config.GetSection("TenantConnections")
            .Get<Dictionary<Guid, string>>() ?? [];
    }
    
    public string GetConnectionString(Guid tenantId) =>
        _connStrings.TryGetValue(tenantId, out var connStr) 
            ? connStr 
            : throw new InvalidOperationException($"No connection for tenant {tenantId}");
}

builder.Services.AddDbContext<AppDbContext>((sp, options) =>
{
    var tenantProvider = sp.GetRequiredService<ITenantProvider>();
    var resolver = sp.GetRequiredService<TenantConnectionStringResolver>();
    options.UseNpgsql(resolver.GetConnectionString(tenantProvider.TenantId));
});
```

**Compliance:** для GDPR, HIPAA — иногда требуется database per tenant (full data isolation).

См. также [Architecture/distributed-systems.md](../Architecture/distributed-systems.md) и [Kubernetes — multi-tenant](../Infrastructure/kubernetes.md).

---

## Bulk Operations

### EF Core 7+ ExecuteUpdate / ExecuteDelete (native)

```csharp
// Bulk update — 1 SQL без загрузки entities в память
await context.Orders
    .Where(o => o.Status == "Pending" && o.CreatedAt < DateTime.UtcNow.AddDays(-30))
    .ExecuteUpdateAsync(setters => setters
        .SetProperty(o => o.Status, "Cancelled")
        .SetProperty(o => o.CancelledAt, DateTime.UtcNow), ct);

// Bulk delete
await context.AuditLogs
    .Where(l => l.Timestamp < DateTime.UtcNow.AddYears(-1))
    .ExecuteDeleteAsync(ct);
```

> [!warning] ExecuteUpdate/Delete пропускает interceptors!
> - SoftDeleteInterceptor НЕ сработает → реальный DELETE
> - AuditInterceptor НЕ сработает → нет audit log
> - Domain Events не публикуются
>
> Используй когда **точно** не нужны эти эффекты (например, purge старых audit log самих).

### EFCore.BulkExtensions — для миллионов записей

```csharp
// NuGet: EFCore.BulkExtensions

// Bulk Insert
var orders = Enumerable.Range(0, 1_000_000).Select(i => new Order { ... }).ToList();
await context.BulkInsertAsync(orders);
// Использует SqlBulkCopy / COPY — на порядки быстрее AddRange + SaveChanges

// Bulk Update
await context.BulkUpdateAsync(modifiedOrders);

// Bulk Insert or Update (upsert)
await context.BulkInsertOrUpdateAsync(orders);

// Conditional bulk
await context.BulkUpdateAsync(orders, options =>
{
    options.PropertiesToInclude = ["Status", "UpdatedAt"];
    options.UpdateByProperties = ["Id"];
});

// Bulk Save (auto detects what's added/modified)
await context.BulkSaveChangesAsync();
```

### Postgres COPY для maximum throughput

```csharp
// 100M строк — в считанные минуты
using var connection = (NpgsqlConnection)context.Database.GetDbConnection();
await connection.OpenAsync(ct);

await using var writer = await connection.BeginBinaryImportAsync(
    "COPY \"Orders\" (\"Id\", \"Total\", \"CustomerId\") FROM STDIN (FORMAT BINARY)", ct);

foreach (var order in orders)
{
    await writer.StartRowAsync(ct);
    await writer.WriteAsync(order.Id, NpgsqlDbType.Uuid, ct);
    await writer.WriteAsync(order.Total, NpgsqlDbType.Numeric, ct);
    await writer.WriteAsync(order.CustomerId, NpgsqlDbType.Uuid, ct);
}

await writer.CompleteAsync(ct);
```

См. [PostgreSQL Deep — Bulk operations](../SQL/postgresql-deep.md).

---

## Domain Events Publishing

DDD паттерн — domain логика публикует события, инфраструктура их обрабатывает.

```csharp
public abstract class DomainEvent
{
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime OccurredAt { get; } = DateTime.UtcNow;
}

public class OrderPlacedEvent(Guid orderId, Guid customerId, decimal total) : DomainEvent
{
    public Guid OrderId { get; } = orderId;
    public Guid CustomerId { get; } = customerId;
    public decimal Total { get; } = total;
}

// Aggregate с events
public class Order
{
    private readonly List<DomainEvent> _domainEvents = [];
    
    [NotMapped]
    public IReadOnlyCollection<DomainEvent> DomainEvents => _domainEvents.AsReadOnly();
    
    public void ClearDomainEvents() => _domainEvents.Clear();
    
    public static Order Place(Guid customerId, IEnumerable<OrderItem> items)
    {
        var order = new Order { /* ... */ };
        order._domainEvents.Add(new OrderPlacedEvent(order.Id, customerId, order.Total));
        return order;
    }
}
```

### Publish через interceptor

```csharp
public class DomainEventsInterceptor(IPublisher publisher) : SaveChangesInterceptor
{
    public override async ValueTask<int> SavedChangesAsync(
        SaveChangesCompletedEventData eventData,
        int result,
        CancellationToken ct = default)
    {
        if (eventData.Context is null) return result;

        var entitiesWithEvents = eventData.Context.ChangeTracker.Entries()
            .Select(e => e.Entity)
            .OfType<IHasDomainEvents>()
            .Where(e => e.DomainEvents.Any())
            .ToList();

        var domainEvents = entitiesWithEvents
            .SelectMany(e => e.DomainEvents)
            .ToList();

        // Clear до публикации
        foreach (var entity in entitiesWithEvents)
            entity.ClearDomainEvents();

        // Publish (через MediatR / IPublisher)
        foreach (var domainEvent in domainEvents)
            await publisher.Publish(domainEvent, ct);

        return result;
    }
}
```

> [!warning] Publish после SaveChanges — at-most-once
> Если SaveChanges прошёл, но publisher упал → событие потеряно. Для at-least-once → Outbox pattern.

### Outbox Integration

См. [Distributed Systems — Outbox](../Architecture/distributed-systems.md).

Кратко: вместо прямой публикации — записываем event в `OutboxMessages` таблицу в той же транзакции:

```csharp
public class OutboxInterceptor : SaveChangesInterceptor
{
    public override async ValueTask<InterceptionResult<int>> SavingChangesAsync(...)
    {
        if (eventData.Context is not AppDbContext context) return ...;

        var domainEvents = context.ChangeTracker.Entries()
            .Select(e => e.Entity)
            .OfType<IHasDomainEvents>()
            .SelectMany(e => e.DomainEvents)
            .ToList();

        if (domainEvents.Count == 0) return ...;

        var outboxMessages = domainEvents.Select(e => new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = e.GetType().AssemblyQualifiedName!,
            Payload = JsonSerializer.Serialize(e, e.GetType()),
            CreatedAt = DateTime.UtcNow
        });

        context.Set<OutboxMessage>().AddRange(outboxMessages);
        return await base.SavingChangesAsync(eventData, result, ct);
    }
}

// Background job читает OutboxMessages, публикует, помечает как processed
```

---

## DDD Aggregate Root

Один `DbSet` на aggregate root. Дочерние сущности — через навигации, без прямого доступа.

```csharp
public class Order  // Aggregate Root
{
    public Guid Id { get; private set; }
    public Guid CustomerId { get; private set; }
    public OrderStatus Status { get; private set; }
    
    private readonly List<OrderItem> _items = [];
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    
    private Order() { }
    
    public static Order Place(Guid customerId, IEnumerable<OrderItem> items)
    {
        var order = new Order
        {
            Id = Guid.NewGuid(),
            CustomerId = customerId,
            Status = OrderStatus.Pending
        };
        order._items.AddRange(items);
        return order;
    }
    
    public void AddItem(Guid productId, int quantity, decimal price)
    {
        if (Status != OrderStatus.Pending) 
            throw new DomainException("Cannot modify confirmed order");
        if (quantity <= 0) 
            throw new DomainException("Invalid quantity");
        
        _items.Add(new OrderItem(productId, quantity, price));
    }

    public decimal Total => _items.Sum(i => i.Price * i.Quantity);
}

public class OrderItem  // Entity inside aggregate, NO own DbSet
{
    public Guid Id { get; private set; }
    public Guid ProductId { get; private set; }
    public int Quantity { get; private set; }
    public decimal Price { get; private set; }
    
    public OrderItem(Guid productId, int quantity, decimal price)
    {
        Id = Guid.NewGuid();
        ProductId = productId;
        Quantity = quantity;
        Price = price;
    }
}

// DbContext — только Order, не OrderItem!
public DbSet<Order> Orders => Set<Order>();
// НЕ public DbSet<OrderItem> OrderItems

// Конфигурация
modelBuilder.Entity<Order>(entity =>
{
    entity.HasMany(o => o.Items)
        .WithOne()
        .HasForeignKey("OrderId");
    
    // Backing field — через _items, не публичный Items
    entity.Navigation(o => o.Items)
        .UsePropertyAccessMode(PropertyAccessMode.Field);
    
    // Computed Total
    entity.Ignore(o => o.Total);  // не сохраняется в БД
});
```

> [!info] Транзакционная граница = один Aggregate
> Один SaveChanges = один aggregate. Изменения в нескольких aggregates → саги или domain events. См. [DDD на практике](../Architecture/ddd.md).

---

## Common Pitfalls

### 1. Soft Delete без cascade

```csharp
// ❌ Order помечен deleted, но OrderItems остались
order.IsDeleted = true;
await context.SaveChangesAsync();

// ✅ Cascade в interceptor или явно
foreach (var item in order.Items)
    context.Entry(item).Property("IsDeleted").CurrentValue = true;
```

### 2. Audit interceptor без ICurrentUserService — null reference

```csharp
// При запуске migrations или в console — нет HttpContext
// CurrentUserService.UserId → throws

// ✅ Defensive
public string? UserId => 
    accessor.HttpContext?.User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
// Возвращаем "system" если null
var userId = currentUser.UserId?.ToString() ?? "system";
```

### 3. Audit log перезаписывает SaveChanges

```csharp
public override async ValueTask<int> SavedChangesAsync(...)
{
    // ❌ Бесконечный recursion
    context.AuditLogs.AddRange(logs);
    await context.SaveChangesAsync();  // → опять SavedChanges → ...
}

// ✅ Использовать flag или отдельный context
private bool _isSavingAuditLog;
public override async ValueTask<int> SavedChangesAsync(...)
{
    if (_isSavingAuditLog) return result;
    
    _isSavingAuditLog = true;
    try
    {
        context.AuditLogs.AddRange(logs);
        await context.SaveChangesAsync(ct);
    }
    finally
    {
        _isSavingAuditLog = false;
    }
}
```

### 4. ExecuteUpdate без cache invalidation

```csharp
// Bulk update — кэш Orders в Redis stale
await context.Orders
    .Where(o => o.CustomerId == customerId)
    .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, "Cancelled"));

// ✅ Invalidate cache явно
await cache.RemoveAsync($"customer:{customerId}:orders");
```

### 5. Repository wrapper hides EF features

```csharp
// ❌ Repository теряет AsNoTracking, Include, AsSplitQuery
public class OrderRepository
{
    public Task<List<Order>> GetAllAsync() => _context.Orders.ToListAsync();
    // Caller не может AsNoTracking без модификации Repository
}

// ✅ Возвращай IQueryable если нужна композиция
public IQueryable<Order> Query() => _context.Orders.AsQueryable();
// Caller: repo.Query().AsNoTracking().Include(...).Where(...)
// Но это уже почти и есть DbContext...
```

### 6. Multi-tenant — запрос без TenantId фильтра

С `HasQueryFilter` это автоматически. Но если используется raw SQL:

```csharp
// ❌ TenantId не применяется к raw SQL!
var results = await context.Database
    .SqlQueryRaw<Order>("SELECT * FROM Orders WHERE Total > 100")
    .ToListAsync();
// Возвращает данные ВСЕХ tenants!
```

**Решение:** RLS на уровне Postgres + connection interceptor.

### 7. Specification pattern — over-engineering

Простой `Where(o => o.Status == "Active")` не нужно превращать в `ActiveOrdersSpecification`. Только когда query логика **переиспользуется в 3+ местах**.

### 8. CQRS — двойная регистрация миграций

ReadDbContext + WriteDbContext — оба пытаются Migrate.

```csharp
// ❌ Read context тоже мигрирует
await readContext.Database.MigrateAsync();
await writeContext.Database.MigrateAsync();

// ✅ Только Write мигрирует, Read только читает (на replica)
await writeContext.Database.MigrateAsync();
```

---

## Best Practices

- **Soft Delete через interceptor** — не разбрасывать логику по handlers
- **Filtered unique index** для soft-deleted полей
- **Audit log в отдельную таблицу** для full change tracking
- **Specification pattern** — только для переиспользуемых сложных queries
- **Repository** — только когда есть реальная польза (testability + complex logic), иначе DbContext
- **CQRS** — separate Read/Write contexts только когда нужно (replica, views, NoTracking globally)
- **Multi-tenant с RLS** — best of shared DB + isolation
- **ExecuteUpdate/Delete** — для bulk без interceptors
- **EFCore.BulkExtensions / COPY** — для миллионов записей
- **Domain Events через Outbox** — for distributed systems

---

## См. также

- [EF Core Basics & Tracking](basics-tracking.md)
- [EF Core Migrations](migrations.md)
- [EF Core Relationships](relationships.md)
- [EF Core Concurrency](concurrency.md)
- [DDD на практике](../Architecture/ddd.md) — Aggregate roots, Domain Events
- [CQRS и MediatR](../Architecture/cqrs-mediatr.md)
- [Distributed Systems — Outbox](../Architecture/distributed-systems.md)
- [PostgreSQL Deep — RLS, Bulk](../SQL/postgresql-deep.md)

## Reading list

- **Microsoft Docs — Interceptors** — learn.microsoft.com/ef/core/logging-events-diagnostics/interceptors
- **Microsoft Docs — Global Query Filters** — learn.microsoft.com/ef/core/querying/filters
- **Ardalis Specification** — github.com/ardalis/Specification
- **EFCore.BulkExtensions** — github.com/borisdj/EFCore.BulkExtensions
- **Andrew Lock — Multi-tenant с EF Core** — andrewlock.net
- **Vladimir Khorikov — DDD blog series** — enterprisecraftsmanship.com
- **Jon P Smith — DDD with EF Core** — github.com/JonPSmith/EfCore.GenericServices
