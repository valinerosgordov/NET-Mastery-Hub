---
tags: [efcore, value-converters, owned-types, json, middle, configuration]
level: Middle
date: 2026-05-10
---

# EF Core Value Converters & Owned Types — типы и schema

> **ValueConverter, ValueComparer, owned types, JSON columns (.NET 7+), backing fields, computed columns.** Всё что нужно для нестандартного mapping между C# и DB schema.

---

## 0. Как читать

После `Junior/ef-basics.md` (Data Annotations vs Fluent). Здесь — sophisticated mapping: enum → string, Money VO → 2 columns, JSON serialize, backing fields для DDD. Перед `Senior/relationships.md` (deep relationships).

---

## 1. Value Converters — базовое

### 1.1. Зачем

C# тип не fits в SQL колонку напрямую:

```csharp
public class User
{
    public UserStatus Status { get; set; }   // enum
    public Email Email { get; set; }          // Value Object
    public List<string> Tags { get; set; }    // collection
    public DateTimeOffset CreatedAt { get; set; }
}
```

Value Converters — **mapping между C# representation и DB representation**.

### 1.2. Built-in: enum → string

```csharp
public enum UserStatus { Active, Suspended, Banned }

public class User
{
    public int Id { get; set; }
    public UserStatus Status { get; set; }
}

// Default: enum → int (Active=0, Suspended=1)
// Если хочешь string:
modelBuilder.Entity<User>()
    .Property(u => u.Status)
    .HasConversion<string>();   // 'Active', 'Suspended'
```

**Зачем string:** Readable в БД, migration safer (re-order enum values не ломает), reports без JOIN.
**Зачем int (default):** Меньше storage, faster comparison.

### 1.3. Custom converter — Value Object

```csharp
public sealed record Email(string Value);

public class User
{
    public int Id { get; set; }
    public Email Email { get; set; } = new("");
}

modelBuilder.Entity<User>()
    .Property(u => u.Email)
    .HasConversion(
        email => email.Value,            // C# → DB
        value => new Email(value));      // DB → C#
```

```sql
CREATE TABLE Users (Id INT, Email VARCHAR(255));
```

### 1.4. List → JSON

```csharp
public class Article
{
    public int Id { get; set; }
    public List<string> Tags { get; set; } = new();
}

modelBuilder.Entity<Article>()
    .Property(a => a.Tags)
    .HasConversion(
        tags => JsonSerializer.Serialize(tags, (JsonSerializerOptions?)null),
        json => JsonSerializer.Deserialize<List<string>>(json, (JsonSerializerOptions?)null) ?? new());
```

### 1.5. Decimal с precision

```csharp
modelBuilder.Entity<Product>()
    .Property(p => p.Price)
    .HasPrecision(18, 2);   // decimal(18, 2)
```

> [!question]- **Интервью: зачем Value Converters?**
> Mapping между C# representation и SQL когда default не подходит. **Use cases**: 1) Enum → string (readable, safe migrations). 2) Value Object → primitive. 3) Collection → JSON. 4) Custom date handling. 5) Encryption. **Configuration**: `HasConversion(toDb, fromDb)` Fluent API. **Performance**: conversion runs на каждом read/write — keep cheap.

---

## 2. ValueComparer — для mutable types

### 2.1. Проблема

```csharp
// List<string> с converter, но без comparer
var article = await _db.Articles.FindAsync(1);
article.Tags.Add("new-tag");   // Mutate existing list
await _db.SaveChangesAsync();
// ❌ Tags не обновлены! EF сравнивает по reference — list тот же объект
```

### 2.2. Решение

```csharp
modelBuilder.Entity<Article>()
    .Property(a => a.Tags)
    .HasConversion(
        v => JsonSerializer.Serialize(v, (JsonSerializerOptions?)null),
        v => JsonSerializer.Deserialize<List<string>>(v, (JsonSerializerOptions?)null) ?? new())
    .Metadata.SetValueComparer(new ValueComparer<List<string>>(
        (a, b) => a!.SequenceEqual(b!),
        c => c.Aggregate(0, (acc, v) => HashCode.Combine(acc, v.GetHashCode())),
        c => c.ToList()));   // snapshot — deep copy
```

ValueComparer:
- **Equals** — как сравнивать
- **GetHashCode** — для tracking
- **Snapshot** — как "запомнить" original (deep copy)

Без snapshot EF сравнивает с тем же reference что mutated.

---

## 3. Owned Types

### 3.1. Что это

Value Object mapping в **те же колонки** что parent — без отдельной таблицы.

```csharp
public sealed record Address(string Street, string City, string Country, string PostalCode);

public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public Address ShippingAddress { get; set; } = null!;
    public Address BillingAddress { get; set; } = null!;
}

modelBuilder.Entity<Customer>(entity =>
{
    entity.OwnsOne(c => c.ShippingAddress, addr =>
    {
        addr.Property(a => a.Street).HasColumnName("ShippingStreet");
        addr.Property(a => a.City).HasColumnName("ShippingCity");
        addr.Property(a => a.Country).HasColumnName("ShippingCountry");
    });
    
    entity.OwnsOne(c => c.BillingAddress, addr =>
    {
        addr.Property(a => a.Street).HasColumnName("BillingStreet");
    });
});
```

```sql
CREATE TABLE Customers (
    Id INT PRIMARY KEY,
    Name VARCHAR(100),
    ShippingStreet VARCHAR(200),
    ShippingCity VARCHAR(100),
    BillingStreet VARCHAR(200)
);
```

### 3.2. Owned vs Reference Navigation

| | Owned Type | Reference Navigation |
|--|-----------|---------------------|
| Lifecycle | Bound to parent | Independent |
| Identity | Не имеет (VO) | Имеет (Entity) |
| Storage | Same table | Separate table |
| Sharing | Не shared | Shared possible |
| Use case | Address, Money, Period | Customer, Product |

### 3.3. OwnsMany — collection

```csharp
public sealed record OrderLine(int ProductId, int Quantity, Money Price);

modelBuilder.Entity<Order>(entity =>
{
    entity.OwnsMany(o => o.Lines, line =>
    {
        line.WithOwner().HasForeignKey("OrderId");
        line.Property<int>("Id");
        line.HasKey("Id");
        
        line.OwnsOne(l => l.Price, price =>
        {
            price.Property(p => p.Amount).HasColumnName("PriceAmount");
            price.Property(p => p.Currency).HasColumnName("PriceCurrency");
        });
    });
});
```

> [!question]- **Интервью: Owned Type vs separate Entity?**
> **Owned Type** — Value Object, lifecycle bound to parent, без identity, mapping в **те же колонки** parent. **Use cases**: Address, Money, Period. **Entity** — independent, имеет Id, может exist без parent. **DDD**: Aggregates contain Owned Types as VOs, reference другие Aggregates через FK only. **Performance**: owned same table — нет JOIN.

---

## 4. JSON Columns (.NET 7+)

### 4.1. Native support

```csharp
public class Product
{
    public int Id { get; set; }
    public ProductMetadata Metadata { get; set; } = new();
}

public class ProductMetadata
{
    public string Brand { get; set; } = "";
    public Dictionary<string, string> Specifications { get; set; } = new();
    public List<string> Tags { get; set; } = new();
}

modelBuilder.Entity<Product>()
    .OwnsOne(p => p.Metadata, b => b.ToJson());
```

```sql
-- PostgreSQL: jsonb
-- SQL Server: nvarchar(max) с JSON validation
CREATE TABLE Products (Id INT, Metadata jsonb);
```

### 4.2. Query JSON

```csharp
var samsungProducts = await _db.Products
    .Where(p => p.Metadata.Brand == "Samsung")
    .ToListAsync();
// PostgreSQL: WHERE Metadata->>'Brand' = 'Samsung'
// SQL Server: WHERE JSON_VALUE(Metadata, '$.Brand') = 'Samsung'
```

### 4.3. Когда JSON

```
✅ Use:
- Schema-less metadata (varying fields)
- Unstructured data
- Read-mostly
- Rich nested objects

❌ Не use:
- Frequently filtered/joined fields
- Critical FK relationships
- Heavy update operations
```

### 4.4. Index на JSON path (PostgreSQL)

```sql
CREATE INDEX idx_products_brand ON Products ((Metadata->>'Brand'));
CREATE INDEX idx_products_metadata_gin ON Products USING GIN (Metadata);
```

---

## 5. Backing Fields для DDD

### 5.1. Зачем

```csharp
public class Order
{
    private readonly List<OrderLine> _lines = new();
    
    public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
    
    public void AddLine(OrderLine line)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Cannot modify shipped order");
        _lines.Add(line);
    }
}
```

EF Core должен mapping в private field `_lines`, не в public read-only `Lines`.

### 5.2. Configuration

```csharp
modelBuilder.Entity<Order>()
    .Navigation(o => o.Lines)
    .HasField("_lines")
    .UsePropertyAccessMode(PropertyAccessMode.Field);
```

### 5.3. Naming conventions

EF auto-detects если naming следует convention:
- `_camelCase`
- `_CamelCase`
- `m_camelCase`

```csharp
private readonly List<OrderLine> _lines = new();
public IReadOnlyList<OrderLine> Lines => _lines.AsReadOnly();
// EF auto-detects _lines
```

> [!question]- **Интервью: backing fields в EF Core зачем?**
> **DDD encapsulation** — Aggregate Root должен control state changes. Public navigation с public setter = любой код делает `order.Lines.Add(...)` → нарушение invariants. **Решение**: private field `_lines` + public read-only `Lines` + public methods (`AddLine()`) с validation. EF Core mapping в backing field через `UsePropertyAccessMode(PropertyAccessMode.Field)` или auto-detection `_camelCase`.

---

## 6. Computed Columns

### 6.1. Definition

```csharp
public class Order
{
    public int Id { get; set; }
    public decimal Subtotal { get; set; }
    public decimal Tax { get; set; }
    public decimal Total { get; set; }
}

modelBuilder.Entity<Order>()
    .Property(o => o.Total)
    .HasComputedColumnSql("[Subtotal] + [Tax]", stored: true);
```

```sql
CREATE TABLE Orders (
    Id INT,
    Subtotal DECIMAL(18,2),
    Tax DECIMAL(18,2),
    Total AS (Subtotal + Tax) PERSISTED
);
```

### 6.2. Stored vs Virtual

```csharp
.HasComputedColumnSql("...", stored: true);    // PERSISTED, indexable
.HasComputedColumnSql("...", stored: false);   // computed at query time
```

---

## 7. Common pitfalls

### 7.1. ValueComparer пропущен

List<string> mutated → не сохранятся изменения. Fix: ValueComparer + Snapshot.

### 7.2. Owned types sharing

```csharp
var address = new Address(...);
customer1.ShippingAddress = address;
customer2.ShippingAddress = address;
// ⚠️ EF создаст две copies — owned types value semantic
```

### 7.3. Migration после change converter

```csharp
// Был int, стал string — existing values не auto-convert
// Нужен manual data migration:
// UPDATE Users SET Status = CASE WHEN Status_old = 0 THEN 'Active' ...
```

### 7.4. JSON columns без index

Filter по JSON property → seq scan. Add explicit index в migration.

### 7.5. Backing field naming wrong

```csharp
private List<OrderLine> linesList = new();   // ❌ EF не auto-find
// Rename → _lines
```

### 7.6. ValueConverter performance

Heavy converter в hot path slow. Encrypt в application layer once, store encrypted in DB.

> [!question]- **Интервью: топ-3 ошибки с value converters?**
> 1) **ValueComparer пропущен для mutable types** — изменения не detected. Fix: ValueComparer с Snapshot. 2) **Owned types использованы как entities** — share между parents → unintended copies. Fix: понимай value vs entity semantic. 3) **JSON columns без indexes** — slow queries. Fix: explicit index на JSON path. **Bonus**: backing field naming — следуй `_camelCase` convention.

---

## 8. Cheat sheet

```csharp
// === Value converter built-in ===
.HasConversion<string>()                       // enum → string
.HasConversion<int>()                          // enum → int (default)

// === Custom converter ===
.HasConversion(
    v => v.Value,                              // C# → DB
    v => new Email(v));                        // DB → C#

// === ValueComparer для mutable ===
.HasConversion(...)
.Metadata.SetValueComparer(new ValueComparer<List<string>>(
    (a, b) => a!.SequenceEqual(b!),
    c => c.Aggregate(0, (acc, v) => HashCode.Combine(acc, v.GetHashCode())),
    c => c.ToList()));

// === Owned Type ===
modelBuilder.Entity<Customer>()
    .OwnsOne(c => c.Address, addr =>
    {
        addr.Property(a => a.Street).HasColumnName("Street");
    });

// === Owned collection ===
modelBuilder.Entity<Order>()
    .OwnsMany(o => o.Lines, line =>
    {
        line.WithOwner().HasForeignKey("OrderId");
        line.Property<int>("Id");
        line.HasKey("Id");
    });

// === JSON column (.NET 7+) ===
modelBuilder.Entity<Product>()
    .OwnsOne(p => p.Metadata, b => b.ToJson());

// === Backing field ===
modelBuilder.Entity<Order>()
    .Navigation(o => o.Lines)
    .HasField("_lines")
    .UsePropertyAccessMode(PropertyAccessMode.Field);

// === Computed column ===
modelBuilder.Entity<Order>()
    .Property(o => o.Total)
    .HasComputedColumnSql("[Subtotal] + [Tax]", stored: true);

// === Decimal precision ===
.Property(p => p.Price).HasPrecision(18, 2);

// === Default values ===
.Property(o => o.CreatedAt).HasDefaultValueSql("GETUTCDATE()");
.Property(o => o.IsActive).HasDefaultValue(true);
```

---

## 9. Practice exercises

### 9.1. Money VO в Order

Реализуй Order с Money owned type:

```csharp
public sealed record Money(decimal Amount, string Currency);

public class Order
{
    public int Id { get; set; }
    public Money Total { get; private set; }
    public Money Tax { get; private set; }
    public Money Discount { get; private set; }
}
```

Configure: Total → TotalAmount + TotalCurrency, Currency NOT NULL MaxLength 3, Amount precision (18, 2).

### 9.2. Audit log с JSON

```csharp
public class AuditLog
{
    public int Id { get; set; }
    public string EntityType { get; set; } = "";
    public DateTime Timestamp { get; set; }
    public Dictionary<string, ChangeDetail> Changes { get; set; } = new();
}
```

Используй JSON column для Changes. Реализуй query: "найди все changes для Order Id 42 last week".

### 9.3. Status enum migration

Enum хранится как int. Migrate на string. Напиши:
1. Migration script (data preservation)
2. Configuration change
3. Test что existing orders correctly mapped

---

## 10. Что читать дальше

1. **`EFCore/Senior/relationships.md`** — relationships deep
2. **`EFCore/Middle/ef-loading-strategies.md`** — loading
3. **`EFCore/Middle/ef-transactions-concurrency.md`** — transactions
4. **`Architecture/Senior/ddd.md`** — DDD strategic + tactical

---

## 11. См. также

- [[ef-basics|EFCore/Junior/ef-basics]] — basics
- [[ef-loading-strategies|EFCore/Middle/ef-loading-strategies]] — loading
- [[ef-transactions-concurrency|EFCore/Middle/ef-transactions-concurrency]] — transactions
- [[ef-bulk-operations|EFCore/Middle/ef-bulk-operations]] — bulk
- [[relationships|EFCore/Senior/relationships]] — relationships
- [[ddd|Architecture/Senior/ddd]] — DDD context

---

## 12. Reading list

- **Microsoft Docs — Value Converters** — learn.microsoft.com/ef/core/modeling/value-conversions
- **Microsoft Docs — Owned Entity Types** — learn.microsoft.com/ef/core/modeling/owned-entities
- **Microsoft Docs — JSON Columns** — learn.microsoft.com/ef/core/modeling/relationships/owned-entities-json-columns
- **Microsoft Docs — Backing Fields** — learn.microsoft.com/ef/core/modeling/backing-field
- **Andrew Lock — EF Core configuration** — andrewlock.net
