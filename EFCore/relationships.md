---
tags: [efcore, relationships, navigation, owned-types]
level: Senior
---

# Relationships и типы

## Navigation Properties

Свойства, ссылающиеся на связанную сущность (reference) или коллекцию (collection). EF использует для построения JOIN, Include, Change Tracking.

```csharp
public class Order
{
    public Guid Id { get; set; }

    // FK — явно объявлен (рекомендуется)
    public Guid CustomerId { get; set; }

    // Reference navigation
    public Customer Customer { get; set; } = null!;

    // Collection navigation
    public ICollection<OrderItem> Items { get; set; } = [];
}

public class Customer
{
    public Guid Id { get; set; }
    public string Name { get; set; } = null!;

    // Inverse navigation
    public ICollection<Order> Orders { get; set; } = [];
}
```

**Нюанс:** если FK не объявлен явно — EF создаёт shadow property. Явный FK даёт контроль и предсказуемость. Инициализация коллекций `= []` предотвращает NullReferenceException.

---

## Конфигурация связей

### One-to-Many

```csharp
modelBuilder.Entity<Order>(entity =>
{
    entity.HasOne(o => o.Customer)
        .WithMany(c => c.Orders)
        .HasForeignKey(o => o.CustomerId)
        .OnDelete(DeleteBehavior.Restrict);  // запрет каскадного удаления
});
```

| DeleteBehavior | Эффект |
|----------------|--------|
| `Cascade` | Удаление parent → удаление children |
| `Restrict` | Запрет удаления parent при наличии children |
| `SetNull` | FK устанавливается в NULL |
| `NoAction` | Как Restrict, но без проверки на клиенте |

### Many-to-Many (EF Core 5+)

```csharp
// Без явной join-таблицы (EF создаст автоматически)
modelBuilder.Entity<Student>()
    .HasMany(s => s.Courses)
    .WithMany(c => c.Students);

// С явной join entity (когда нужны дополнительные поля)
modelBuilder.Entity<Enrollment>(entity =>
{
    entity.HasKey(e => new { e.StudentId, e.CourseId });
    entity.HasOne(e => e.Student).WithMany(s => s.Enrollments);
    entity.HasOne(e => e.Course).WithMany(c => c.Enrollments);
});
```

### One-to-One

```csharp
modelBuilder.Entity<Order>()
    .HasOne(o => o.ShippingAddress)
    .WithOne()
    .HasForeignKey<ShippingAddress>(a => a.OrderId);
```

---

## Owned Types (Value Objects)

Без собственной идентичности. Принадлежат владельцу. Хранятся в таблице владельца или в отдельной таблице.

```csharp
// Доменная модель
public class Address
{
    public string Street { get; init; } = null!;
    public string City { get; init; } = null!;
    public string ZipCode { get; init; } = null!;
}

public class Customer
{
    public Guid Id { get; set; }
    public Address ShippingAddress { get; set; } = null!;
    public Address? BillingAddress { get; set; }
}

// Конфигурация
modelBuilder.Entity<Customer>(entity =>
{
    entity.OwnsOne(c => c.ShippingAddress, addr =>
    {
        addr.Property(a => a.Street).HasColumnName("ShippingStreet");
        addr.Property(a => a.City).HasColumnName("ShippingCity");
    });
    entity.OwnsOne(c => c.BillingAddress); // nullable owned type
});
```

**Нюанс:** Owned types не имеют DbSet. Загружаются всегда с владельцем (автоматический Include). `OwnsMany` — для коллекции (отдельная таблица).

---

## Shadow Properties

Свойства, которые существуют только в модели EF и в БД, но не в C# классе.

```csharp
// Конфигурация
modelBuilder.Entity<Order>()
    .Property<DateTime>("CreatedAt")
    .HasDefaultValueSql("GETUTCDATE()");

modelBuilder.Entity<Order>()
    .Property<string>("TenantId");

// Доступ
var createdAt = context.Entry(order).Property<DateTime>("CreatedAt").CurrentValue;

// Фильтрация
context.Orders.Where(o => EF.Property<string>(o, "TenantId") == tenantId);
```

**Применение:** аудит (CreatedAt, UpdatedAt), мультитенантность (TenantId), soft delete (IsDeleted). Модель остаётся чистой.

---

## Enum Mapping

```csharp
public enum OrderStatus { New, Processing, Shipped, Delivered }

// По умолчанию — int в БД
// Для хранения как string:
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>()   // "New", "Processing" и т.д.
    .HasMaxLength(20);

// Или кастомный конвертер
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion(
        v => v.ToString(),
        v => Enum.Parse<OrderStatus>(v));
```

**Нюанс:** string хранение — читаемо в БД, но занимает больше места. Int — компактно, но при переименовании enum значения ломаются. Выбор зависит от контекста.

---

## Indexes

```csharp
modelBuilder.Entity<Order>(entity =>
{
    // Простой индекс
    entity.HasIndex(o => o.CustomerId);

    // Составной индекс (порядок колонок важен!)
    entity.HasIndex(o => new { o.CustomerId, o.CreatedAt });

    // Уникальный индекс
    entity.HasIndex(o => o.OrderNumber).IsUnique();

    // Filtered index (PostgreSQL / SQL Server)
    entity.HasIndex(o => o.Status)
        .HasFilter("[Status] <> 'Deleted'");

    // Covering index (include columns)
    entity.HasIndex(o => o.CustomerId)
        .IncludeProperties(o => new { o.Total, o.Status });
});
```

**Нюанс:** индекс на `(A, B, C)` используется для запросов по `A`, `A+B`, `A+B+C`, но НЕ для `B` или `C` отдельно (leftmost prefix rule).

---

## См. также

- [Interview: EF Core и SQL](../../../Interview/5-ef-core-sql.md)
