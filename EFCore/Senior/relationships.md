---
tags: [efcore, relationships, navigation, owned-types, json-columns, value-converters, computed-columns]
level: Senior
date: 2026-08-02
---

# Relationships, типы данных и продвинутый mapping

> Полный гайд по связям и типам в EF Core. Закрывает: все relationships (1:1/1:N/N:N), TPH/TPT/TPC inheritance, Owned types, Shadow properties, Value Converters (включая Strongly-Typed IDs), JSON columns (EF Core 7+), computed columns, generated values, backing fields, indexes deep, schema separation.

---

## Что это, зачем и когда

### Что такое Relationships?
Связи между таблицами в БД, описанные через C#-свойства. `Order` имеет `Customer` → в БД это FK `CustomerId`.

**Аналогия:** Папки с документами. Заказ «лежит» в папке клиента. Связь = ссылка «этот заказ принадлежит этому клиенту».

### Когда какой тип связи?

| Связь | Пример | В БД |
|-------|--------|------|
| **One-to-Many** | Customer → Orders | FK `CustomerId` в Orders |
| **Many-to-Many** | Student ↔ Course | Join-таблица `StudentCourse` |
| **One-to-One** | User → UserProfile | FK + Unique index |
| **Self-Referencing** | Employee → Manager (Employee) | FK `ManagerId` к той же таблице |
| **Owned Type** | Order → Address (Value Object) | Столбцы Address_Street в Orders |
| **TPH/TPT/TPC inheritance** | Payment → CreditCard, BankTransfer | По стратегии (см. ниже) |

---

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

> [!warning] Shadow FK vs Explicit FK
> Если FK не объявлен явно — EF создаёт **shadow property**. Это работает, но:
> - Не виден в коде (только через `EF.Property<Guid>(o, "CustomerId")`)
> - Сложнее тесты и debugging
> - Невозможно установить значение FK без загрузки родителя
>
> **Всегда объявляй FK явно.**

### Required vs Optional Navigation

```csharp
// Required (NOT NULL FK) — заказ всегда принадлежит клиенту
public Guid CustomerId { get; set; }
public Customer Customer { get; set; } = null!;

// Optional (NULL FK) — у заказа может не быть купона
public Guid? CouponId { get; set; }
public Coupon? Coupon { get; set; }
```

EF Core 6+ выводит required/optional из nullability аннотаций. С `<Nullable>enable</Nullable>` — поведение точное.

---

## Конфигурация связей

### One-to-Many

```csharp
modelBuilder.Entity<Order>(entity =>
{
    entity.HasOne(o => o.Customer)
        .WithMany(c => c.Orders)
        .HasForeignKey(o => o.CustomerId)
        .OnDelete(DeleteBehavior.Restrict);
});
```

#### DeleteBehavior — все варианты

| Behavior | Поведение | Когда использовать |
|----------|-----------|---------------------|
| `Cascade` | Удаление parent → удаление children | Aggregate (Order → OrderItems) |
| `ClientCascade` | Cascade в client, не в DB | Когда DB не поддерживает FK cascades (rare) |
| `Restrict` | Запрет удаления parent при наличии children (DB throws) | По умолчанию для безопасности |
| `SetNull` | FK устанавливается в NULL | Optional FK (Comment.AuthorId nullable) |
| `ClientSetNull` | SetNull в client, no action в DB | Когда DB не поддерживает |
| `NoAction` | Как Restrict, но без runtime check на клиенте | Для perf — verify constraint только в DB |

> [!warning] Cascade Cycles в SQL Server
> SQL Server запрещает несколько cascade paths к одной таблице (потенциальный infinite loop при FK cycle). Решение: `OnDelete(DeleteBehavior.Restrict)` для всех путей кроме одного, либо обрабатывать удаление вручную.

```csharp
// Postgres — поддерживает cycles, можно cascade везде
// SQL Server — может выбросить "may cause cycles or multiple cascade paths"
```

### Many-to-Many

#### Без явной join entity (EF Core 5+)

```csharp
modelBuilder.Entity<Student>()
    .HasMany(s => s.Courses)
    .WithMany(c => c.Students);
// EF создаст join-таблицу StudentCourse автоматически
```

#### С явной join entity (когда нужны дополнительные поля)

```csharp
public class Enrollment
{
    public Guid StudentId { get; set; }
    public Student Student { get; set; } = null!;
    
    public Guid CourseId { get; set; }
    public Course Course { get; set; } = null!;
    
    public DateTime EnrolledAt { get; set; }
    public Grade? Grade { get; set; }
}

modelBuilder.Entity<Enrollment>(entity =>
{
    entity.HasKey(e => new { e.StudentId, e.CourseId });
    
    entity.HasOne(e => e.Student)
        .WithMany(s => s.Enrollments)
        .HasForeignKey(e => e.StudentId);
    
    entity.HasOne(e => e.Course)
        .WithMany(c => c.Enrollments)
        .HasForeignKey(e => e.CourseId);
});
```

#### Skip navigations + named join

```csharp
modelBuilder.Entity<Student>()
    .HasMany(s => s.Courses)
    .WithMany(c => c.Students)
    .UsingEntity<Enrollment>(
        j => j.HasOne(e => e.Course).WithMany(c => c.Enrollments),
        j => j.HasOne(e => e.Student).WithMany(s => s.Enrollments),
        j => 
        {
            j.HasKey(e => new { e.StudentId, e.CourseId });
            j.Property(e => e.EnrolledAt).HasDefaultValueSql("now()");
        });

// Можно работать "напрямую" с Student.Courses (skip), и через Enrollment
student.Courses.Add(course);  // unfilled EnrolledAt — будет default
```

### One-to-One

```csharp
public class User
{
    public Guid Id { get; set; }
    public UserProfile? Profile { get; set; }
}

public class UserProfile
{
    public Guid Id { get; set; }
    public Guid UserId { get; set; }     // FK + Unique
    public User User { get; set; } = null!;
    public string Bio { get; set; } = "";
}

modelBuilder.Entity<User>()
    .HasOne(u => u.Profile)
    .WithOne(p => p.User)
    .HasForeignKey<UserProfile>(p => p.UserId);  // явно указываем dependent
```

> [!info] Кто dependent / principal в 1:1
> EF не может вывести автоматически — нужно указать через `HasForeignKey<T>(...)`. T — это **dependent** (тот у кого FK). Principal — другой.

### Self-Referencing

Сотрудник имеет менеджера (тоже сотрудник):

```csharp
public class Employee
{
    public Guid Id { get; set; }
    public string Name { get; set; } = "";
    
    // FK на самого себя
    public Guid? ManagerId { get; set; }
    public Employee? Manager { get; set; }
    
    public ICollection<Employee> DirectReports { get; set; } = [];
}

modelBuilder.Entity<Employee>()
    .HasOne(e => e.Manager)
    .WithMany(e => e.DirectReports)
    .HasForeignKey(e => e.ManagerId)
    .OnDelete(DeleteBehavior.Restrict);  // не удалять подчинённых при удалении менеджера
```

#### Recursive query — все подчинённые

```csharp
// Загрузить всю иерархию через recursive CTE
var hierarchy = await context.Database
    .SqlQuery<Employee>($@"
        WITH RECURSIVE hierarchy AS (
            SELECT * FROM ""Employees"" WHERE ""Id"" = {rootId}
            UNION ALL
            SELECT e.* FROM ""Employees"" e
            INNER JOIN hierarchy h ON e.""ManagerId"" = h.""Id""
        )
        SELECT * FROM hierarchy
    ")
    .ToListAsync(ct);
```

См. [[postgresql-deep|PostgreSQL Deep — Recursive CTE]].

---

## Inheritance: TPH, TPT, TPC

Стратегии маппинга наследования на таблицы.

### TPH (Table Per Hierarchy) — default

**Одна таблица, дискриминатор.** Самая быстрая стратегия.

```csharp
public abstract class Payment
{
    public Guid Id { get; set; }
    public decimal Amount { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class CreditCardPayment : Payment
{
    public string CardLast4 { get; set; } = "";
    public string CardholderName { get; set; } = "";
}

public class BankTransferPayment : Payment
{
    public string IBAN { get; set; } = "";
    public string SwiftCode { get; set; } = "";
}

public class CryptoPayment : Payment
{
    public string WalletAddress { get; set; } = "";
    public string Currency { get; set; } = "";
}

// Конфигурация
modelBuilder.Entity<Payment>()
    .HasDiscriminator<string>("PaymentType")
    .HasValue<CreditCardPayment>("CreditCard")
    .HasValue<BankTransferPayment>("BankTransfer")
    .HasValue<CryptoPayment>("Crypto");

// Auto-discriminator — если не указано вручную
// EF использует имя класса как discriminator value автоматически
```

#### Структура таблицы

```sql
CREATE TABLE "Payments" (
    "Id" UUID PRIMARY KEY,
    "Amount" DECIMAL(18,2) NOT NULL,
    "CreatedAt" TIMESTAMPTZ NOT NULL,
    "PaymentType" VARCHAR(50) NOT NULL,
    -- CreditCard fields (NULL для других типов)
    "CardLast4" VARCHAR(4),
    "CardholderName" VARCHAR(100),
    -- BankTransfer fields
    "IBAN" VARCHAR(34),
    "SwiftCode" VARCHAR(11),
    -- Crypto fields
    "WalletAddress" VARCHAR(64),
    "Currency" VARCHAR(10)
);
```

#### Запросы

```csharp
// Все payments
var all = await context.Payments.ToListAsync();

// Только credit cards
var ccPayments = await context.Payments.OfType<CreditCardPayment>().ToListAsync();

// SQL: SELECT * FROM "Payments" WHERE "PaymentType" = 'CreditCard'
```

### TPT (Table Per Type)

**Отдельная таблица для каждого типа.** JOIN при запросе.

```csharp
modelBuilder.Entity<Payment>().UseTptMappingStrategy();

modelBuilder.Entity<Payment>().ToTable("Payments");
modelBuilder.Entity<CreditCardPayment>().ToTable("CreditCardPayments");
modelBuilder.Entity<BankTransferPayment>().ToTable("BankTransferPayments");
modelBuilder.Entity<CryptoPayment>().ToTable("CryptoPayments");
```

```sql
CREATE TABLE "Payments" (Id, Amount, CreatedAt);
CREATE TABLE "CreditCardPayments" (Id REFERENCES "Payments", CardLast4, CardholderName);
CREATE TABLE "BankTransferPayments" (Id REFERENCES "Payments", IBAN, SwiftCode);
```

Запрос требует JOIN:
```sql
SELECT p.*, cc.* FROM "Payments" p
LEFT JOIN "CreditCardPayments" cc ON p."Id" = cc."Id"
LEFT JOIN "BankTransferPayments" bt ON p."Id" = bt."Id"
LEFT JOIN "CryptoPayments" cr ON p."Id" = cr."Id";
```

### TPC (Table Per Concrete type) — EF Core 7+

**Отдельная таблица для каждого конкретного типа без базовой таблицы.**

```csharp
modelBuilder.Entity<Payment>().UseTpcMappingStrategy();

modelBuilder.Entity<CreditCardPayment>().ToTable("CreditCardPayments");
modelBuilder.Entity<BankTransferPayment>().ToTable("BankTransferPayments");
modelBuilder.Entity<CryptoPayment>().ToTable("CryptoPayments");
```

```sql
CREATE TABLE "CreditCardPayments" (Id, Amount, CreatedAt, CardLast4, CardholderName);
CREATE TABLE "BankTransferPayments" (Id, Amount, CreatedAt, IBAN, SwiftCode);
-- Базовая таблица "Payments" НЕ существует
```

Запрос всех типов — UNION:
```sql
SELECT * FROM "CreditCardPayments"
UNION ALL
SELECT * FROM "BankTransferPayments"
UNION ALL
SELECT * FROM "CryptoPayments";
```

### Сравнение стратегий

| Стратегия | Таблиц | Запрос подтипа | Запрос базового | Nullable cols | Когда |
|-----------|--------|----------------|-----------------|---------------|-------|
| **TPH** | 1 | Быстро (filter) | Быстро | Да | Default — лучше всего по perf |
| **TPT** | N+1 | JOIN — медленно | JOIN — медленно | Нет | Нормализация, много уникальных полей |
| **TPC** | N | Быстро | UNION — средне | Нет | Когда базовый тип почти не запрашивается |

**Default — TPH.** TPT/TPC — когда:
- TPT: иерархия широкая, каждый подтип имеет 10+ уникальных полей → много NULL columns в TPH
- TPC: запросы почти всегда по подтипам, базовый класс редко используется

> [!warning] Изменение стратегии — ломает данные
> Переход с TPH на TPT (или наоборот) на live БД — это сложная миграция (нужно перенести данные между таблицами). Лучше выбирать стратегию на старте.

### Mapping в "плоский" объект (без inheritance)

Если иерархия — артефакт C# модели и нет в БД:

```csharp
// Исключить из inheritance hierarchy
modelBuilder.Ignore<BasePayment>();

// Или: простой enum-based discriminator
public class Payment
{
    public Guid Id { get; set; }
    public decimal Amount { get; set; }
    public PaymentType Type { get; set; }
    public string? CardLast4 { get; set; }
    public string? IBAN { get; set; }
}
```

---

## Owned Types (Value Objects)

Без собственной идентичности. Принадлежат владельцу. Хранятся в таблице владельца или в отдельной таблице.

### OwnsOne — single owned

```csharp
public class Address
{
    public string Street { get; init; } = "";
    public string City { get; init; } = "";
    public string ZipCode { get; init; } = "";
    public string Country { get; init; } = "";
}

public class Customer
{
    public Guid Id { get; set; }
    public Address ShippingAddress { get; set; } = null!;
    public Address? BillingAddress { get; set; }
}

modelBuilder.Entity<Customer>(entity =>
{
    entity.OwnsOne(c => c.ShippingAddress, addr =>
    {
        addr.Property(a => a.Street).HasColumnName("ShippingStreet");
        addr.Property(a => a.City).HasColumnName("ShippingCity");
        addr.Property(a => a.ZipCode).HasColumnName("ShippingZip");
        addr.Property(a => a.Country).HasColumnName("ShippingCountry");
    });
    
    entity.OwnsOne(c => c.BillingAddress);  // nullable owned type
});
```

В БД — все поля Address колонками в `Customers`:

```sql
CREATE TABLE "Customers" (
    "Id" UUID,
    "ShippingStreet" TEXT,
    "ShippingCity" TEXT,
    "ShippingZip" TEXT,
    "ShippingCountry" TEXT,
    "BillingAddress_Street" TEXT,  -- nullable
    "BillingAddress_City" TEXT,
    ...
);
```

### OwnsMany — collection of owned

```csharp
public class Order
{
    public Guid Id { get; set; }
    public List<OrderLine> Lines { get; set; } = [];
}

public class OrderLine
{
    public string ProductName { get; init; } = "";
    public int Quantity { get; init; }
    public decimal UnitPrice { get; init; }
}

modelBuilder.Entity<Order>().OwnsMany(o => o.Lines, lines =>
{
    lines.WithOwner().HasForeignKey("OrderId");
    lines.HasKey("OrderId", "Id");
});
```

В БД — отдельная таблица `OrderLine`, но **доступная только через Order**.

### Owned vs Regular Entity — когда что

| Owned | Regular |
|-------|---------|
| Нет identity (Address — это "Москва, Тверская 1", не "Address #42") | Есть identity (Customer #42) |
| Принадлежит одному owner | Может быть shared между владельцами |
| Загружается всегда с owner | Может быть запрошен напрямую |
| Нет DbSet | Есть DbSet |
| Меняется immutably (record) | Mutable, tracked |

> [!info] Когда Owned — правильный выбор
> Address, Money, DateRange, Coordinates — все они Value Objects в DDD. Используй OwnsOne для записи в туже таблицу, OwnsMany — для коллекции но без separate management.

---

## Shadow Properties

Свойства, которые существуют только в модели EF и в БД, но не в C# классе.

```csharp
modelBuilder.Entity<Order>()
    .Property<DateTime>("CreatedAt")
    .HasDefaultValueSql("now()");

modelBuilder.Entity<Order>()
    .Property<string>("TenantId");

modelBuilder.Entity<Order>()
    .Property<int>("Version")
    .IsConcurrencyToken();

// Доступ через context.Entry().Property<T>()
var createdAt = context.Entry(order).Property<DateTime>("CreatedAt").CurrentValue;
context.Entry(order).Property<string>("TenantId").CurrentValue = "tenant-1";

// В LINQ — через EF.Property<T>()
var orders = await context.Orders
    .Where(o => EF.Property<string>(o, "TenantId") == tenantId)
    .ToListAsync();
```

**Применение:** аудит (CreatedAt, UpdatedAt, ModifiedBy), мультитенантность (TenantId), soft delete (IsDeleted), version (для optimistic concurrency). Модель остаётся чистой.

---

## Backing Fields — для encapsulation

DDD aggregate должен скрывать internal collection:

```csharp
public class Order
{
    public Guid Id { get; private set; }
    
    private readonly List<OrderItem> _items = [];
    
    // Public read-only view
    public IReadOnlyCollection<OrderItem> Items => _items.AsReadOnly();
    
    // Domain method — единственный способ изменить
    public void AddItem(Guid productId, int quantity, decimal price)
    {
        if (quantity <= 0) throw new DomainException("Invalid quantity");
        _items.Add(new OrderItem(productId, quantity, price));
    }
}

// Конфигурация — EF использует backing field _items
modelBuilder.Entity<Order>(entity =>
{
    entity.HasMany(o => o.Items)
        .WithOne()
        .HasForeignKey("OrderId");
    
    entity.Navigation(o => o.Items)
        .UsePropertyAccessMode(PropertyAccessMode.Field);  // _items, не Items
});
```

Это критично для DDD — клиентский код не может `order.Items.Add(...)` напрямую, обходя domain logic.

---

## Value Converters — type mapping advanced

### Strongly-Typed IDs

Вместо `Guid OrderId` — типизированный wrapper, защищает от путаницы:

```csharp
public readonly record struct OrderId(Guid Value);
public readonly record struct CustomerId(Guid Value);

public class Order
{
    public OrderId Id { get; set; }
    public CustomerId CustomerId { get; set; }
}

// Без converter — компилятор: 
//   "Cannot convert OrderId to Guid"
public Order GetOrder(CustomerId customerId)  // нельзя случайно передать OrderId!
{
    return context.Orders.First(o => o.CustomerId == customerId);
}

// Конфигурация
modelBuilder.Entity<Order>()
    .Property(o => o.Id)
    .HasConversion(
        id => id.Value,                    // C# → DB
        value => new OrderId(value));       // DB → C#

// Reusable converter
public class OrderIdConverter : ValueConverter<OrderId, Guid>
{
    public OrderIdConverter() : base(id => id.Value, value => new OrderId(value)) { }
}

modelBuilder.Entity<Order>()
    .Property(o => o.Id)
    .HasConversion<OrderIdConverter>();
```

### Enum → string

```csharp
public enum OrderStatus { New, Processing, Shipped, Delivered, Cancelled }

modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>()
    .HasMaxLength(20);

// SQL: "Status" VARCHAR(20)  -- "New", "Processing", etc
```

#### Enum → string vs int

| | int | string |
|--|-----|--------|
| Размер | 4 bytes | 10-20 bytes |
| Читаемость в БД | Плохо | Хорошо |
| Renaming enum | Не ломается | **Ломается** (нужна миграция данных) |
| Reordering enum | **Ломается** | Не ломается |
| Performance index | Чуть быстрее | Чуть медленнее |

**Рекомендация:** string — для читаемости. Если performance критичен и enum стабилен — int.

### List → JSON (если нет JSON columns поддержки)

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.Tags)
    .HasConversion(
        tags => JsonSerializer.Serialize(tags, JsonSerializerOptions.Default),
        json => JsonSerializer.Deserialize<List<string>>(json, JsonSerializerOptions.Default)!,
        new ValueComparer<List<string>>(
            (c1, c2) => c1!.SequenceEqual(c2!),
            c => c.Aggregate(0, (a, v) => HashCode.Combine(a, v.GetHashCode())),
            c => c.ToList()));
// ValueComparer обязателен для коллекций — иначе change tracking не сработает
```

### DateTime UTC enforcer

```csharp
public class UtcDateTimeConverter : ValueConverter<DateTime, DateTime>
{
    public UtcDateTimeConverter() : base(
        v => v.Kind == DateTimeKind.Utc ? v : v.ToUniversalTime(),
        v => DateTime.SpecifyKind(v, DateTimeKind.Utc))
    { }
}

// Применить ко всем DateTime properties
foreach (var entityType in modelBuilder.Model.GetEntityTypes())
{
    foreach (var property in entityType.GetProperties())
    {
        if (property.ClrType == typeof(DateTime) || property.ClrType == typeof(DateTime?))
        {
            property.SetValueConverter(new UtcDateTimeConverter());
        }
    }
}
```

### Encryption converter

```csharp
public class EncryptedStringConverter : ValueConverter<string, string>
{
    public EncryptedStringConverter(IDataProtector protector) : base(
        plain => protector.Protect(plain),
        encrypted => protector.Unprotect(encrypted))
    { }
}

modelBuilder.Entity<User>()
    .Property(u => u.SocialSecurityNumber)
    .HasConversion(new EncryptedStringConverter(protector));

// В БД — encrypted, в коде — plain text
```

---

## JSON Columns — конспект

Полный deep-dive (запросы, JSON-массивы, индексация, complex types) — [[ef-value-converters|EF Core Value Converters — JSON Columns]]. Здесь только суть для контекста mapping'а.

Версии: `ToJson()` для owned types появился в **EF Core 7** (SQL Server); **EF Core 8** — Npgsql (jsonb) и SQLite; **EF Core 10** — рекомендуемый путь через **complex types** (`ComplexProperty().ToJson()`), owned `ToJson()` — legacy.

```csharp
public class Customer
{
    public Guid Id { get; set; }
    public Address Address { get; set; } = null!;  // как JSON
}

modelBuilder.Entity<Customer>()
    .OwnsOne(c => c.Address, addr => addr.ToJson());  // вся Address в одной JSON-колонке
```

```sql
CREATE TABLE "Customers" (
    "Id" UUID PRIMARY KEY,
    "Address" JSONB  -- {"Street": "...", "City": "...", "ZipCode": "..."}
);
```

Запросы по JSON-свойствам (`.Where(c => c.Address.City == "Moscow")`) транслируются в `->>` / `JSON_VALUE`.

✅ **Хорошо когда:** schema-flexible части (settings, metadata), owned type не нужен в JOIN'ах, реже изменяемые данные.
❌ **Плохо когда:** частые фильтры/JOIN по полям, нужны индексы на individual fields (в Postgres спасает GIN), большие JSON (KB+).

См. также [[postgresql-deep|PostgreSQL Deep — JSONB]].

---

## Computed Columns

```csharp
public class Order
{
    public decimal SubTotal { get; set; }
    public decimal Tax { get; set; }
    public decimal Total { get; set; }  // computed
}

modelBuilder.Entity<Order>()
    .Property(o => o.Total)
    .HasComputedColumnSql("\"SubTotal\" + \"Tax\"", stored: true);
```

| Stored / Persisted | Поведение |
|-------|-----------|
| `stored: true` (PERSISTED / STORED) | Значение вычислено при INSERT/UPDATE и хранится | 
| `stored: false` (default в EF API) | Вычисляется при каждом SELECT (virtual) |

> [!warning] Postgres до 18 — только STORED
> Virtual generated columns в PostgreSQL появились лишь в **PG 18** (и там VIRTUAL — дефолт для `GENERATED ALWAYS AS`). До PG 18 Npgsql поддерживал только `stored: true` — `stored: false` на старых версиях не сработает. В SQL Server наоборот: non-persisted computed — исторический дефолт.

**Когда какой:** stored — для индексирования и частых reads. Non-stored — для редких запросов и экономии storage.

> [!warning] Computed columns — read-only с EF
> Не пытайся писать в `Order.Total` через EF — он будет игнорироваться. EF знает что это computed.

---

## Generated Values

### Auto-generated при insert

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.Id)
    .HasDefaultValueSql("gen_random_uuid()");  // PG 13+
    // SQL Server: NEWID() / NEWSEQUENTIALID()
    // MySQL: UUID()

modelBuilder.Entity<Order>()
    .Property(o => o.CreatedAt)
    .HasDefaultValueSql("now() at time zone 'utc'");

// .ValueGeneratedOnAdd() — EF знает что значение придёт из DB после INSERT
modelBuilder.Entity<Order>()
    .Property(o => o.OrderNumber)
    .HasDefaultValueSql("nextval('order_seq')")
    .ValueGeneratedOnAdd();
```

### Auto-update при update

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.UpdatedAt)
    .ValueGeneratedOnAddOrUpdate()
    .HasComputedColumnSql("now()", stored: false);  // не работает для триггеров

// Для real auto-update — нужен trigger в БД
migrationBuilder.Sql(@"
    CREATE OR REPLACE FUNCTION update_modified_timestamp()
    RETURNS TRIGGER AS $$
    BEGIN
        NEW.""UpdatedAt"" = now();
        RETURN NEW;
    END;
    $$ LANGUAGE plpgsql;
    
    CREATE TRIGGER orders_modified
    BEFORE UPDATE ON ""Orders""
    FOR EACH ROW EXECUTE FUNCTION update_modified_timestamp();
");
```

### Sequential Guid — лучше для index

`Guid.NewGuid()` — random — плохо для clustered index (page splits).

```csharp
// PG 18+ — нативная uuidv7(), без расширений
modelBuilder.Entity<Order>()
    .Property(o => o.Id)
    .HasDefaultValueSql("uuidv7()");

// PG < 18 — стороннее расширение pg_uuidv7 (функция uuid_generate_v7())
// .HasDefaultValueSql("uuid_generate_v7()");  // требует CREATE EXTENSION pg_uuidv7

// Или генерируем в C# — нативный API, без NuGet (.NET 9+)
public static Guid NewSequentialGuid() => Guid.CreateVersion7();
```

> [!warning] SQL Server сортирует `uniqueidentifier` с последних 6 байт — v7 там почти random
> Подробный разбор порядка байтов и рабочих стратегий ключей: [[bcl-essentials]] (раздел 2.4).

---

## Indexes — детально

```csharp
modelBuilder.Entity<Order>(entity =>
{
    // Простой индекс
    entity.HasIndex(o => o.CustomerId);

    // Составной индекс (порядок колонок важен!)
    entity.HasIndex(o => new { o.CustomerId, o.CreatedAt });

    // Уникальный индекс
    entity.HasIndex(o => o.OrderNumber).IsUnique();

    // Composite unique
    entity.HasIndex(o => new { o.CustomerId, o.OrderNumber }).IsUnique();
    
    // Filtered index (PG, SQL Server)
    entity.HasIndex(o => o.Status)
        .HasFilter("\"Status\" <> 'Deleted'");
    
    // Filtered unique для soft delete
    entity.HasIndex(o => o.Email)
        .IsUnique()
        .HasFilter("\"IsDeleted\" = false");

    // Covering index (PG 11+, SQL Server)
    entity.HasIndex(o => o.CustomerId)
        .IncludeProperties(o => new { o.Total, o.Status });
    
    // Index с descending order
    entity.HasIndex(o => o.CreatedAt)
        .IsDescending();  // CREATE INDEX ... ON Orders (CreatedAt DESC)
    
    // Composite с разными directions
    entity.HasIndex(o => new { o.CustomerId, o.CreatedAt })
        .IsDescending(false, true);  // CustomerId ASC, CreatedAt DESC
    
    // Method-based (Postgres GIN, GIST)
    entity.HasIndex(o => o.SearchVector)
        .HasMethod("gin");
    
    // Index name
    entity.HasIndex(o => o.OrderNumber)
        .HasDatabaseName("ux_orders_number");
});
```

### Leftmost prefix rule

Индекс `(A, B, C)` работает для:
- `WHERE A = ?` ✓
- `WHERE A = ? AND B = ?` ✓
- `WHERE A = ? AND B = ? AND C = ?` ✓
- `ORDER BY A, B` ✓

НЕ работает для:
- `WHERE B = ?` ✗ (без A)
- `WHERE C = ?` ✗ (без A, B)
- `ORDER BY B, A` ✗ (другой порядок)

См. [[optimization|SQL Optimization — индексы]].

---

## Schema separation (Postgres)

```csharp
modelBuilder.Entity<Order>().ToTable("Orders", schema: "ordering");
modelBuilder.Entity<Customer>().ToTable("Customers", schema: "identity");

// Default schema для всех
modelBuilder.HasDefaultSchema("app");
```

В Postgres — separate schemas для bounded contexts: `ordering`, `inventory`, `identity`.

### Search path

```csharp
// Postgres — установить search_path
options.UseNpgsql(connStr, b => b.MigrationsHistoryTable("__EFMigrations", schema: "public"));

// Или через connection string
"...;SearchPath=ordering,public"
```

---

## Default Conventions Override

EF Core 8+ — кастомизация conventions:

```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    // Все decimal по умолчанию precision(18, 4)
    configurationBuilder.Properties<decimal>().HavePrecision(18, 4);
    
    // Все DateTime — UTC
    configurationBuilder.Properties<DateTime>()
        .HaveConversion<UtcDateTimeConverter>();
    
    // Все строки до 500 символов
    configurationBuilder.Properties<string>().HaveMaxLength(500);
    
    // Все enum как string
    configurationBuilder.Properties<Enum>().HaveConversion<string>();
}
```

Уменьшает boilerplate в `OnModelCreating`.

---

## Common Pitfalls

### 1. Cycle exception при сериализации

JSON serializer бросает `JsonException: A possible object cycle was detected`.

```csharp
// Order → Customer → Orders → Customer → ...

// ❌ В JSON ответе — infinite cycle
return Ok(orders);

// ✅ Решения
// 1. DTO без обратных ссылок
return Ok(orders.Select(o => new OrderDto(o.Id, o.Customer.Name)));

// 2. ReferenceHandler.IgnoreCycles
builder.Services.ConfigureHttpJsonOptions(opts =>
    opts.SerializerOptions.ReferenceHandler = ReferenceHandler.IgnoreCycles);

// 3. Не загружать инвертирующую навигацию
var orders = await context.Orders
    .Include(o => o.Customer)
    // НЕ Include обратно Customer.Orders
    .ToListAsync();
```

### 2. AutoInclude не работает в проекциях

```csharp
// AutoInclude — глобальный Include для всех queries
modelBuilder.Entity<Order>().Navigation(o => o.Customer).AutoInclude();

// ❌ В Select проекции AutoInclude игнорируется
var dtos = await context.Orders
    .Select(o => new { o.Id, CustomerName = o.Customer.Name })  // загрузит Customer
    .ToListAsync();

// AutoInclude работает только когда возвращаешь Entity, не projection
```

### 3. ChangeTracker не видит модификации в OwnsMany через get-only collection

```csharp
public class Order
{
    public IReadOnlyList<OrderLine> Lines => _lines;
    private readonly List<OrderLine> _lines = [];
}

// ❌ EF не отслеживает изменения если Navigation не настроена для backing field
modelBuilder.Entity<Order>()
    .OwnsMany(o => o.Lines, ...);  // Lines — readonly, EF путается

// ✅ Использовать backing field explicitly
modelBuilder.Entity<Order>()
    .OwnsMany(o => o.Lines, lines => 
    {
        lines.Navigation("_lines").UsePropertyAccessMode(PropertyAccessMode.Field);
    });
```

### 4. Лишние JOIN при OwnsOne и null

```csharp
public class Customer
{
    public Address? BillingAddress { get; set; }  // nullable owned
}

// EF делает LEFT JOIN на саму Customer таблицу — confusing query
// SQL: SELECT * FROM Customers c LEFT JOIN Customers c2 ON c.Id = c2.Id WHERE c.BillingAddress_Street IS NOT NULL
```

Решение: проверить `EXPLAIN ANALYZE` если queries неоптимальны для nullable owned types.

### 5. M:N коллекция с Add не сохраняет дополнительные поля join entity

```csharp
public class Enrollment
{
    public Guid StudentId { get; set; }
    public Guid CourseId { get; set; }
    public DateTime EnrolledAt { get; set; }
}

// Через skip navigation — не трогаем EnrolledAt
student.Courses.Add(course);
// EnrolledAt = default (1970-01-01)

// ✅ Через explicit join entity
context.Enrollments.Add(new Enrollment 
{ 
    StudentId = student.Id, 
    CourseId = course.Id, 
    EnrolledAt = DateTime.UtcNow 
});
```

### 6. Cascade delete уносит критичные данные

`DeleteBehavior.Cascade` на FK к Customers → удаление клиента сносит все его orders. **Это часто не то что хочется.**

```csharp
// ✅ Лучше — Restrict + soft delete + manual cleanup
modelBuilder.Entity<Order>()
    .HasOne(o => o.Customer)
    .WithMany(c => c.Orders)
    .HasForeignKey(o => o.CustomerId)
    .OnDelete(DeleteBehavior.Restrict);  // защита от accidental delete

// Soft delete клиента вместо физического удаления
customer.IsDeleted = true;
```

### 7. Inheritance с TPH — большая таблица с NULL колонками

Если у тебя 5 подтипов с 10 уникальных полей каждый — таблица будет иметь 50 nullable columns.

```csharp
// ✅ Лучше — TPT для широких иерархий
modelBuilder.Entity<Payment>().UseTptMappingStrategy();

// Или вынести специфичные поля в JSON
public class Payment
{
    public PaymentType Type { get; set; }
    public Dictionary<string, object> Details { get; set; } = [];  // JSON
}
```

### 8. Self-referencing с Cascade — не работает в SQL Server

SQL Server не позволяет cascade на self-reference (cycle). EF миграция упадёт.

```csharp
// ✅ Restrict для SQL Server
modelBuilder.Entity<Employee>()
    .HasOne(e => e.Manager)
    .WithMany(e => e.DirectReports)
    .OnDelete(DeleteBehavior.Restrict);
```

В Postgres — работает.

---

## Best Practices

- **Объявляй FK явно** — никаких shadow FK
- **TPH default**, TPT/TPC только при явных причинах
- **OwnsOne для Value Objects** — Address, Money, DateRange
- **JSON columns** для schema-flexible частей (EF Core 7+)
- **Strongly-typed IDs** для критичных сущностей (защита от перепутывания)
- **Backing fields для DDD** aggregates — `private readonly List<>`
- **Restrict default** для DeleteBehavior, Cascade только осознанно
- **IEntityTypeConfiguration\<T\>** для изоляции конфигурации (одна entity — один файл)
- **ConfigureConventions** для глобальных правил (UTC, decimal precision)
- **Filtered indexes** для soft delete, чтобы unique constraints работали правильно
- **Sequential Guid** для clustered index performance

---

## Case Studies

### Case Study #1 — One-to-Many: User → Orders

**Сценарий:** User имеет много Orders. Стандартный 1:N.

```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; }
    public List<Order> Orders { get; set; } = new();
}

public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }    // FK
    public User User { get; set; }
    public decimal Total { get; set; }
}

// Configuration (Fluent API)
modelBuilder.Entity<Order>()
    .HasOne(o => o.User)
    .WithMany(u => u.Orders)
    .HasForeignKey(o => o.UserId)
    .OnDelete(DeleteBehavior.Restrict);  // не cascade — orders сохранять
```

**Production tip:** не используй `OnDelete(Cascade)` для аудит-важных entities — soft delete предпочтительнее.

---

### Case Study #2 — Many-to-Many: Students ↔ Courses

**До EF Core 5 — explicit join table.**  
**EF Core 5+ — implicit:**

```csharp
public class Student
{
    public int Id { get; set; }
    public List<Course> Courses { get; set; } = new();
}

public class Course
{
    public int Id { get; set; }
    public List<Student> Students { get; set; } = new();
}

// EF создаёт CourseStudent join table автоматически
```

**Если нужны extra поля в join (когда student enrolled, grade):**
```csharp
public class Enrollment
{
    public int StudentId { get; set; }
    public int CourseId { get; set; }
    public DateTime EnrolledAt { get; set; }
    public int? Grade { get; set; }
    
    public Student Student { get; set; }
    public Course Course { get; set; }
}

modelBuilder.Entity<Enrollment>().HasKey(e => new { e.StudentId, e.CourseId });
```

---

### Case Study #3 — Hierarchy: Comments tree

**Сценарий:** Reddit-like threading — comments имеют parent comment.

```csharp
public class Comment
{
    public int Id { get; set; }
    public string Text { get; set; }
    public int? ParentId { get; set; }   // null для top-level
    public Comment? Parent { get; set; }
    public List<Comment> Replies { get; set; } = new();
}

modelBuilder.Entity<Comment>()
    .HasOne(c => c.Parent)
    .WithMany(c => c.Replies)
    .HasForeignKey(c => c.ParentId);
```

**⚠️ Pitfall:** загружать всё дерево через Include — N levels deep = N joins. Используй recursive CTE через raw SQL для глубоких trees.


---

## Cheat sheet

| Relationship | Configuration | Note |
|--------------|---------------|------|
| One-to-One | `HasOne().WithOne().HasForeignKey<>()` | FK в одной из таблиц |
| One-to-Many | `HasOne().WithMany().HasForeignKey()` | FK в "many" side |
| Many-to-Many (simple) | `List<T>` на обеих сторонах | EF Core 5+ auto join table |
| Many-to-Many (extra fields) | Explicit join entity | Нужна composite key |
| Self-referencing | HasOne(parent).WithMany(children) | Tree / hierarchy |
| Owned (no FK) | `OwnsOne()` / `OwnsMany()` | Value objects (Address, Money) |

| Delete behavior | Что происходит |
|-----------------|----------------|
| `Cascade` | Удалить parent → удалить children (опасно!) |
| `Restrict` | Throw exception если есть children |
| `SetNull` | Set FK = null в children (FK должен быть nullable) |
| `NoAction` | Не делать ничего (DB сама решит) |

**Default conventions:**
- Required FK (non-nullable) → `Cascade` by default
- Optional FK (nullable) → `ClientSetNull` by default

**Recommended:** `OnDelete(DeleteBehavior.Restrict)` для важных entities + soft delete pattern.


---

## Decision tree

```
Какой relationship?
│
├── Сколько связанных entities?
│   ├── Один к одному → HasOne / WithOne (FK в одном из них)
│   ├── Один ко многим → HasOne / WithMany (FK в "many" side)
│   └── Многие ко многим
│       ├── Без extra полей → List<T> на обеих сторонах (auto join table)
│       └── С полями (date, role) → Explicit join entity
│
├── Hierarchy (parent → children)?
│   └── Self-reference: HasOne(parent).WithMany(children)
│
├── Value object (без identity)?
│   └── OwnsOne / OwnsMany — embedded в parent table
│
├── Cascade behavior?
│   ├── Audit-важно → Restrict + soft delete
│   ├── Truly child (нет смысла без parent) → Cascade
│   ├── Optional → SetNull (FK nullable)
│   └── Manual control → NoAction
│
└── Loading strategy?
    ├── Always need → .Include() eager
    ├── Sometimes need → Lazy loading (proxy) или explicit
    ├── Specific subset → projection (.Select)
    └── Performance critical → AsSplitQuery если cartesian explosion
```


---

## См. также

- [[basics-tracking|EF Core Basics & Tracking]]
- [[migrations|EF Core Migrations]]
- [[concurrency|EF Core Concurrency]]
- [[ef-patterns|EF Core Patterns]] — Repository, soft delete, audit interceptors
- [[ddd|DDD на практике]] — Aggregate roots, Value Objects
- [[postgresql-deep|PostgreSQL Deep — JSONB]]
- [[optimization|SQL Optimization — индексы]]

## Reading list

- **Microsoft Docs — Relationships** — learn.microsoft.com/ef/core/modeling/relationships
- **Microsoft Docs — Inheritance** — learn.microsoft.com/ef/core/modeling/inheritance
- **Microsoft Docs — Owned types** — learn.microsoft.com/ef/core/modeling/owned-entities
- **Microsoft Docs — JSON columns** — learn.microsoft.com/ef/core/modeling/relational/json-columns
- **Andrew Lock — Strongly-typed IDs in EF Core** — andrewlock.net/series/strongly-typed-ids/
- **Jon P Smith — Entity Framework Core in Action** (книга, особенно главы про owned types и inheritance)
