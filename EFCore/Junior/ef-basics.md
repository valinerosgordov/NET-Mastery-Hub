---
tags: [efcore, basics, junior, orm, dbcontext]
level: Junior
date: 2026-08-02
---

# EF Core Basics — что такое ORM и как им пользоваться

> **DbContext, DbSet, основные CRUD операции, миграции, конфигурация.** Введение перед `Middle/dapper-comparison.md` и `Senior/basics-tracking.md`.

---

## 0. Как читать

Если впервые видишь EF — раздел 1→4 last (basics + setup + CRUD). Если знаешь ORMs из других языков (Hibernate/Sequelize/SQLAlchemy) — пропусти 1.1, перейди к 2 (specifics).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Что такое ORM

**Object-Relational Mapper** — библиотека которая превращает таблицы БД в C# классы и обратно. Вместо SQL — пишешь LINQ.

```csharp
// Без ORM — raw SQL
var conn = new SqlConnection("...");
conn.Open();
var cmd = new SqlCommand("SELECT Id, Name, Email FROM Users WHERE Id = @id", conn);
cmd.Parameters.AddWithValue("@id", 42);
var reader = cmd.ExecuteReader();
reader.Read();
var user = new User
{
    Id = reader.GetInt32(0),
    Name = reader.GetString(1),
    Email = reader.GetString(2)
};

// С EF Core
var user = await _db.Users.FindAsync(42);
```

EF Core генерирует SQL под капотом. Ты работаешь с C# объектами.

> [!warning] В raw-примере выше — `AddWithValue`, это анти-паттерн
> Параметр он добавляет правильно (защита от инъекций есть), но **тип угадывает из значения**: `string` уходит как `nvarchar`, и при колонке `varchar` SQL Server делает implicit conversion на каждой строке → index scan вместо seek. В реальном ADO.NET-коде задавай тип явно: `cmd.Parameters.Add("@id", SqlDbType.Int).Value = 42`. Полный разбор — [[security-practices|AspNetCore/Senior/security-practices]]. С EF Core этой проблемы нет — провайдер берёт тип из модели.

### 1.2. EF Core vs альтернативы

```
EF Core (Microsoft):
✅ Встроен в .NET ecosystem
✅ Type-safe LINQ queries
✅ Автоматические migrations
✅ Change tracking
✅ Большое community
❌ Может генерить плохие SQL queries (изучается)
❌ Performance overhead vs raw SQL

Dapper (micro-ORM):
✅ Очень быстрый (~ raw ADO.NET)
✅ Контроль над SQL
❌ Сам пишешь SQL
❌ Без change tracking, migrations
✅ Хорош для read-heavy / specific queries

Raw ADO.NET / SqlConnection:
✅ Максимальный control
✅ Самый быстрый
❌ Boilerplate
❌ SQL injection risk если не parameterize
❌ Manual mapping

NHibernate:
✅ Powerful (Java Hibernate port)
❌ Сложнее EF Core
❌ Меньше community сейчас
```

**В 2024+ для большинства .NET проектов: EF Core.**

### 1.3. Когда EF Core оправдан

```
✅ Use EF Core когда:
- Standard CRUD приложение (~80% случаев)
- Маленькие/средние team
- Schema эволюционирует (migrations важны)
- Type safety приоритет
- Не write-heavy benchmark

❌ Не use когда:
- Read-only reporting / analytics → Dapper
- Performance-critical hot path → Dapper / raw SQL
- Простой CLI script → ADO.NET
- Сложный legacy SQL который EF не понимает → Dapper
```

### 1.4. Главное правило

```
EF Core — для повседневной работы с БД в C# приложении.
Dapper — для специфичных performance-critical queries.
Raw SQL — для редких edge cases.

В одном приложении можно mix: EF для CRUD + Dapper для отчётов.
```

> [!info]- Если ты знаешь Hibernate / Sequelize / SQLAlchemy / TypeORM
> Концепция та же — Object-Relational Mapping. EF Core ближе к Hibernate чем Sequelize. **DbContext** ≈ Hibernate Session / SQLAlchemy session. **`DbSet<T>`** ≈ Hibernate `Session.createQuery` / SQLAlchemy `session.query(Model)`. Migrations ≈ Alembic / Hibernate dbm. Главное отличие — LINQ queries (compile-time checked).

> [!question]- Интервью: что такое EF Core?
> **Entity Framework Core** — Microsoft's official ORM для .NET. Преобразует C# objects в SQL queries и обратно. Provides: 1) **DbContext** — sessions / unit of work. 2) **`DbSet<T>`** — табличные коллекции. 3) **LINQ queries** — type-safe SQL generation. 4) **Migrations** — schema evolution. 5) **Change tracking** — automatic update detection. **Use case**: повседневная работа с БД в .NET приложении. Cross-platform, supports SQL Server, PostgreSQL, MySQL, SQLite, Cosmos DB.

---

## 2. Setup — как начать

### 2.1. NuGet packages

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.0" />
  <!-- Или Postgres: -->
  <!-- <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.0" /> -->
  
  <!-- Для миграций и dotnet ef CLI -->
  <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.0" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="8.0.0" />
</ItemGroup>
```

### 2.2. Установка dotnet ef tool

```bash
# Глобально
dotnet tool install --global dotnet-ef

# Или локально в проект
dotnet tool install --local dotnet-ef
```

`dotnet ef` — CLI для миграций, scaffolding, etc.

### 2.3. Создание DbContext

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }
    
    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }
}

// Entity classes
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public DateTime CreatedAt { get; set; }
}

public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public decimal Total { get; set; }
    public DateTime CreatedAt { get; set; }
}
```

### 2.4. Регистрация в ASP.NET Core

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

var app = builder.Build();
```

`appsettings.json`:

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=MyApp;Trusted_Connection=True;TrustServerCertificate=True;"
  }
}
```

### 2.5. Создание первой миграции

```bash
# Создать миграцию (генерируется код в Migrations/)
dotnet ef migrations add InitialCreate

# Применить к БД (создаст БД + таблицы)
dotnet ef database update

# Откатить последнюю миграцию
dotnet ef migrations remove

# Откатить применённую миграцию (rollback в БД)
dotnet ef database update <PreviousMigrationName>
```

EF Core генерирует SQL для создания таблиц по твоим C# классам.

### 2.6. Console приложение setup

Без ASP.NET Core — простой console app:

```csharp
// Program.cs
using Microsoft.EntityFrameworkCore;

var optionsBuilder = new DbContextOptionsBuilder<AppDbContext>();
optionsBuilder.UseSqlServer("Server=localhost;Database=MyApp;Trusted_Connection=True;TrustServerCertificate=True;");

using var db = new AppDbContext(optionsBuilder.Options);

// Используй db
var users = db.Users.ToList();
foreach (var user in users) Console.WriteLine(user.Name);
```

> [!question]- Интервью: что такое DbContext?
> **DbContext** — central class EF Core, представляет **session с базой данных**. Содержит: 1) **`DbSet<T>`** properties — каждая = таблица в БД. 2) **Change tracker** — отслеживает изменения сущностей. 3) **Connection** к БД. 4) **Configuration** через `OnModelCreating` или `DbContextOptions`. **Lifecycle**: scoped (один на HTTP запрос). **Pattern**: Unit of Work — все изменения собираются и сохраняются вместе через `SaveChangesAsync`. **Не делать**: long-lived DbContext (memory leak), shared между threads (not thread-safe).

---

## 3. CRUD operations

### 3.1. Create — добавить запись

```csharp
public async Task<int> CreateUserAsync(string name, string email)
{
    var user = new User
    {
        Name = name,
        Email = email,
        CreatedAt = DateTime.UtcNow
    };
    
    _db.Users.Add(user);
    await _db.SaveChangesAsync();
    
    return user.Id;   // Id заполнится автоматически после Save
}
```

```csharp
// Add multiple
_db.Users.AddRange(new[]
{
    new User { Name = "Alice", Email = "a@x.com" },
    new User { Name = "Bob", Email = "b@x.com" },
});
await _db.SaveChangesAsync();
```

### 3.2. Read — найти запись

```csharp
// FindAsync — по primary key, fastest
var user = await _db.Users.FindAsync(42);

// FirstOrDefaultAsync — первая по предикату или null
var user = await _db.Users.FirstOrDefaultAsync(u => u.Email == "a@x.com");

// SingleOrDefaultAsync — единственная по предикату или null
// (бросает если найдено больше одной)
var user = await _db.Users.SingleOrDefaultAsync(u => u.Email == "a@x.com");

// ToListAsync — все записи
var users = await _db.Users.ToListAsync();

// Filtered list
var adults = await _db.Users.Where(u => u.Age >= 18).ToListAsync();

// Count
var count = await _db.Users.CountAsync();
var adultCount = await _db.Users.CountAsync(u => u.Age >= 18);

// Any / All
var hasUsers = await _db.Users.AnyAsync();
var allActive = await _db.Users.AllAsync(u => u.IsActive);
```

### 3.3. Update — изменить запись

```csharp
// Способ 1: Load → modify → Save
public async Task UpdateUserNameAsync(int id, string newName)
{
    var user = await _db.Users.FindAsync(id);
    if (user == null) throw new InvalidOperationException("User not found");
    
    user.Name = newName;   // Change tracker заметит
    await _db.SaveChangesAsync();
}

// Change tracker автоматически генерирует UPDATE SQL только для изменённых полей.
```

```csharp
// Способ 2: Update без load (optimization, EF Core 7+)
public async Task UpdateUserNameAsync(int id, string newName)
{
    await _db.Users
        .Where(u => u.Id == id)
        .ExecuteUpdateAsync(u => u.SetProperty(x => x.Name, newName));
}
// Один UPDATE запрос, без load
```

```csharp
// Способ 3: Detached entity attach
public async Task UpdateUserAsync(User user)
{
    _db.Users.Update(user);   // attaches and marks all properties as modified
    await _db.SaveChangesAsync();
}
// Менее efficient — UPDATE всех полей даже не изменённых
```

### 3.4. Delete — удалить запись

```csharp
// Способ 1: Load → Remove → Save
public async Task DeleteUserAsync(int id)
{
    var user = await _db.Users.FindAsync(id);
    if (user == null) return;
    
    _db.Users.Remove(user);
    await _db.SaveChangesAsync();
}

// Способ 2: Bulk delete без load (EF Core 7+)
public async Task DeleteUserAsync(int id)
{
    await _db.Users
        .Where(u => u.Id == id)
        .ExecuteDeleteAsync();
}

// Способ 3: Bulk delete по условию
public async Task DeleteInactiveUsersAsync()
{
    await _db.Users
        .Where(u => !u.IsActive && u.CreatedAt < DateTime.UtcNow.AddYears(-1))
        .ExecuteDeleteAsync();
}
```

### 3.5. Полный CRUD пример

```csharp
public class UserService
{
    private readonly AppDbContext _db;
    
    public UserService(AppDbContext db) => _db = db;
    
    public async Task<int> CreateAsync(string name, string email)
    {
        var user = new User { Name = name, Email = email, CreatedAt = DateTime.UtcNow };
        _db.Users.Add(user);
        await _db.SaveChangesAsync();
        return user.Id;
    }
    
    public async Task<User?> GetByIdAsync(int id) =>
        await _db.Users.FindAsync(id);
    
    public async Task<List<User>> GetAllAsync() =>
        await _db.Users.OrderBy(u => u.CreatedAt).ToListAsync();
    
    public async Task UpdateAsync(int id, string newName)
    {
        var user = await _db.Users.FindAsync(id);
        if (user == null) throw new InvalidOperationException("Not found");
        user.Name = newName;
        await _db.SaveChangesAsync();
    }
    
    public async Task DeleteAsync(int id) =>
        await _db.Users
            .Where(u => u.Id == id)
            .ExecuteDeleteAsync();
}
```

> [!question]- Интервью: что делает SaveChangesAsync?
> Запускает **commit всех изменений** в DbContext в одну транзакцию. EF Core: 1) Анализирует Change Tracker. 2) Генерирует SQL (`INSERT`/`UPDATE`/`DELETE`). 3) Открывает transaction. 4) Выполняет SQL. 5) Возвращает количество affected rows. 6) Если ошибка — rollback всей транзакции. **Без SaveChanges** — `Add`/`Remove`/property changes только в памяти. **Pattern**: один SaveChanges на logical operation (один HTTP request типично). Несколько Add/Remove → один SaveChanges.

---

## 4. Configuration — Data Annotations vs Fluent API

### 4.1. Data Annotations — на классе

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

public class User
{
    [Key]
    public int Id { get; set; }
    
    [Required]
    [MaxLength(100)]
    public string Name { get; set; } = "";
    
    [Required]
    [EmailAddress]
    [MaxLength(200)]
    [Column("EmailAddress")]   // имя колонки в БД
    public string Email { get; set; } = "";
    
    [Range(0, 150)]
    public int Age { get; set; }
    
    [DataType(DataType.DateTime)]
    public DateTime CreatedAt { get; set; }
    
    [NotMapped]   // не сохраняется в БД
    public string DisplayName => $"{Name} ({Email})";
}

[Table("UserOrders")]   // имя таблицы в БД
public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }
    [Precision(18, 2)]   // decimal(18, 2)
    public decimal Total { get; set; }
}
```

### 4.2. Fluent API — в OnModelCreating

```csharp
public class AppDbContext : DbContext
{
    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<User>(entity =>
        {
            entity.HasKey(u => u.Id);
            entity.Property(u => u.Name).IsRequired().HasMaxLength(100);
            entity.Property(u => u.Email).IsRequired().HasMaxLength(200);
            entity.HasIndex(u => u.Email).IsUnique();   // unique index
        });
        
        modelBuilder.Entity<Order>(entity =>
        {
            entity.ToTable("UserOrders");
            entity.Property(o => o.Total).HasPrecision(18, 2);
            entity.HasOne<User>()
                .WithMany()
                .HasForeignKey(o => o.UserId);
        });
    }
}
```

### 4.3. Что выбрать

```
Data Annotations:
✅ Простые ограничения (Required, MaxLength)
✅ Видны прямо в Entity class
❌ Не все features поддерживаются
❌ Mixing data + persistence concerns

Fluent API:
✅ Полный набор features
✅ Сложные scenarios (composite keys, complex relationships)
✅ Separation: Entity = data, Configuration = persistence
❌ Конфигурация в OnModelCreating (далеко от Entity)

Best practice 2024+:
- Data Annotations для simple cases
- Fluent API для complex
- Если конфликт — Fluent API побеждает
- Используй IEntityTypeConfiguration<T> для крупных проектов
```

### 4.4. IEntityTypeConfiguration — отдельный конфиг файл

```csharp
public class UserConfiguration : IEntityTypeConfiguration<User>
{
    public void Configure(EntityTypeBuilder<User> builder)
    {
        builder.HasKey(u => u.Id);
        builder.Property(u => u.Name).IsRequired().HasMaxLength(100);
        builder.HasIndex(u => u.Email).IsUnique();
        // вся конфигурация User в одном файле
    }
}

// В DbContext:
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    // Auto-discover all IEntityTypeConfiguration<T>
}
```

Лучшая практика для серьёзных проектов.

> [!question]- Интервью: Data Annotations или Fluent API?
> **Data Annotations** — атрибуты на classes (`[Required]`, `[MaxLength]`). **Простые** ограничения, **видны прямо в Entity**. Минус: mixing data model с persistence concerns. **Fluent API** — конфигурация в `OnModelCreating` через `modelBuilder`. **Полный** набор features, separation of concerns. **Best practice 2024+**: 1) **Простые** ограничения (Required, MaxLength) — Data Annotations. 2) **Сложные** (composite keys, complex relationships, custom converters) — Fluent API. 3) **Большие проекты** — `IEntityTypeConfiguration<T>` separate file для каждой entity.

---

## 5. Conventions — что EF Core делает "магически"

### 5.1. Primary Key

```csharp
public class User
{
    public int Id { get; set; }   // автоматически PK (по convention "Id" or "<TypeName>Id")
}

// Or
public class User
{
    public int UserId { get; set; }   // автоматически PK
}
```

EF Core ищет property "Id" или "`<ClassName>Id`" → primary key.

### 5.2. Type → SQL conversion

```
C# type           → SQL type (SQL Server)
int                int (NOT NULL)
int?               int (NULL)
long               bigint
short              smallint
byte               tinyint
bool               bit
string             nvarchar(MAX) (NULL by default)
string [Required]  nvarchar(MAX) NOT NULL
string [MaxLength(100)]  nvarchar(100)
decimal            decimal(18, 2)
double             float
DateTime           datetime2
DateOnly           date
TimeOnly           time
Guid               uniqueidentifier
byte[]             varbinary(MAX)
```

### 5.3. Имя таблицы

```csharp
public DbSet<User> Users { get; set; }
// → Table name: "Users" (имя property)

public DbSet<Order> AllOrders { get; set; }
// → Table name: "AllOrders"
```

EF Core использует имя `DbSet<T>` property как имя таблицы. Override через `[Table("CustomName")]` или Fluent API.

### 5.4. Pluralization

EF Core НЕ pluralize automatically (в отличие от Hibernate с some configs). Имя property в `DbSet<T>` определяет имя таблицы. Принято `Users`, `Orders`, `Products`.

### 5.5. Foreign Keys по convention

```csharp
public class User
{
    public int Id { get; set; }
    public List<Order> Orders { get; set; } = new();   // navigation
}

public class Order
{
    public int Id { get; set; }
    public int UserId { get; set; }   // FK по convention
    public User User { get; set; } = null!;   // navigation
}
```

EF Core находит relationship: `Order.UserId` → `User.Id`. Foreign key создаётся автоматически.

### 5.6. Override conventions

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Snake_case columns globally
    foreach (var entity in modelBuilder.Model.GetEntityTypes())
    {
        foreach (var property in entity.GetProperties())
        {
            property.SetColumnName(ToSnakeCase(property.Name));
        }
    }
}
```

> [!question]- Интервью: что такое conventions в EF Core?
> Default rules которые EF Core применяет автоматически: 1) **PK detection** — property named "Id" или "`<TypeName>Id`". 2) **Type mapping** — C# `string` → `nvarchar(MAX)`, `int` → `int`. 3) **Table name** — DbSet property name. 4) **FK detection** — `<RelatedTypeName>Id` linked to navigation. 5) **Index** — none by default (need explicit `[Index]` или Fluent). **Override**: Data Annotations на class, Fluent API в OnModelCreating, или global conventions (EF Core 7+ `ConfigureConventions`). **Best practice**: rely on conventions для типичных случаев, override только когда нужно.

---

## 6. Migrations — schema evolution

### 6.1. Зачем миграции

Когда изменяешь Entity (добавил property / поле / класс), БД должна адаптироваться. Миграция = SQL script для применения изменения.

```csharp
// Initial state
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

// Изменил — добавил Email
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";   // NEW
}
```

Создать миграцию:

```bash
dotnet ef migrations add AddEmailToUser
```

EF Core генерирует:

```csharp
// Migrations/20260510120000_AddEmailToUser.cs
public partial class AddEmailToUser : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(
            name: "Email",
            table: "Users",
            type: "nvarchar(max)",
            nullable: false,
            defaultValue: "");
    }
    
    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(name: "Email", table: "Users");
    }
}
```

Применить к БД:

```bash
dotnet ef database update
```

### 6.2. Workflow

```
1. Изменил Entity (добавил/удалил/переименовал property)
2. dotnet ef migrations add <DescriptiveName>
3. Просмотрел сгенерированный код — корректен ли?
4. dotnet ef database update — применил
5. Проверил БД
```

### 6.3. Полезные команды

```bash
# Создать миграцию
dotnet ef migrations add AddOrders

# Применить все pending
dotnet ef database update

# Применить до specific миграции
dotnet ef database update AddOrders

# Откатить (применить предыдущую)
dotnet ef database update PreviousMigrationName

# Удалить последнюю миграцию (если ещё не применена!)
dotnet ef migrations remove

# Сгенерировать SQL без применения (для review/production)
dotnet ef migrations script
dotnet ef migrations script PreviousMigrationName CurrentMigrationName

# Список миграций
dotnet ef migrations list

# Drop всю БД
dotnet ef database drop
```

### 6.4. Production migrations

```
В production НЕ запускай `dotnet ef database update` напрямую!

Лучше:
1. Сгенерировать SQL: dotnet ef migrations script -i -o migration.sql
   (-i = idempotent, выполнится только нужные)
2. Review SQL
3. DBA / DevOps применяет в production
4. Code deploy
```

### 6.5. Connection string для миграций

```csharp
// В Program.cs DI registration:
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// dotnet ef использует Default connection из appsettings.Development.json (по convention)
```

Или явно:

```bash
dotnet ef database update --connection "Server=...;..."
```

> [!question]- Интервью: что такое migrations и зачем?
> EF Core mechanism для **schema evolution**: каждое изменение Entity создаёт миграцию (C# class в Migrations/), которая знает как **обновить БД** (Up) и **откатить** (Down). **Workflow**: change Entity → `dotnet ef migrations add Name` → review generated code → `dotnet ef database update`. **В production**: не запускать update напрямую, сгенерировать SQL script, review, применить через DevOps. **Альтернативы**: ручные SQL scripts (DbUp, FluentMigrator) — больше control, меньше automation.

---

## 7. Tracking vs No-Tracking

### 7.1. Default: Tracking

```csharp
var user = await _db.Users.FindAsync(42);
user.Name = "Bob";
await _db.SaveChangesAsync();
// EF автоматически замечает изменение, генерирует UPDATE
```

`DbContext` хранит **snapshot** каждой загруженной entity. На SaveChanges сравнивает текущее состояние с snapshot — генерирует SQL только для изменённых полей.

### 7.2. AsNoTracking — для read-only queries

```csharp
// Read-only — не нужен tracking
var users = await _db.Users
    .AsNoTracking()
    .ToListAsync();

// Изменения не отслеживаются — нельзя сделать SaveChanges на этих entities
users[0].Name = "Bob";
await _db.SaveChangesAsync();   // НЕ обновится в БД
```

**Зачем AsNoTracking:**
- ✅ Faster queries (не создаёт snapshot)
- ✅ Меньше memory
- ✅ Лучше для list views, reports

```csharp
// Для read-heavy endpoints — всегда AsNoTracking
public async Task<List<UserDto>> GetAllAsync() =>
    await _db.Users
        .AsNoTracking()
        .Select(u => new UserDto(u.Id, u.Name, u.Email))
        .ToListAsync();
```

### 7.3. Tracking decision rule

```
Tracking (default) когда:
- Будешь изменять и сохранять entities
- Single entity load (FindAsync)

AsNoTracking когда:
- Read-only operations
- List views (lots of entities, no edits)
- API responses (read → DTO)
- Reports, analytics
```

### 7.4. AsNoTrackingWithIdentityResolution

```csharp
// Если в response одна entity появляется несколько раз (через JOIN)
var orders = await _db.Orders
    .AsNoTrackingWithIdentityResolution()
    .Include(o => o.User)
    .ToListAsync();
```

EF дедуплицирует одинаковые entities. Использовать когда есть identity (не для anonymous types).

> [!question]- Интервью: что такое AsNoTracking?
> EF Core хранит **snapshot загруженных entities** для change detection. **`AsNoTracking()`** отключает это — entities загружаются без snapshot. **Pros**: faster, less memory, simpler. **Cons**: нельзя сделать SaveChanges на этих entities. **Use cases**: read-only queries, list views, API responses (DTO mapping), reports. **Default**: tracking (для CRUD scenarios). **Best practice 2024+**: всегда AsNoTracking для read-only paths. Globally можно через `optionsBuilder.UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking)`.

---

## 8. Common pitfalls

### 8.1. N+1 query problem

```csharp
// ❌ N+1
var users = await _db.Users.ToListAsync();   // 1 query
foreach (var user in users)
{
    Console.WriteLine($"{user.Name}: {user.Orders.Count}");
    // Каждый Orders доступ → отдельный SQL query!
    // Если 100 users → 1 + 100 = 101 queries
}
```

**Фикс**: `Include` для eager loading:

```csharp
// ✅ 1 query
var users = await _db.Users
    .Include(u => u.Orders)
    .ToListAsync();
```

См. `Senior/queries-performance.md`.

### 8.2. SaveChanges забыл

```csharp
public async Task UpdateUserAsync(int id, string name)
{
    var user = await _db.Users.FindAsync(id);
    user.Name = name;
    // ❌ Забыл SaveChangesAsync — изменения только в памяти!
}
```

### 8.3. Tracking entity dispose

```csharp
async Task<List<User>> GetUsers()
{
    using var db = new AppDbContext(options);
    return await db.Users.ToListAsync();
    // ❌ DbContext disposed — попытки lazy load навигации crash
}
```

**Фикс**: load всё что нужно ДО dispose, или используй scoped DbContext (DI).

### 8.4. DbContext shared между threads

```csharp
// ❌ DbContext НЕ thread-safe
var db = new AppDbContext(options);
await Task.WhenAll(
    Task.Run(() => db.Users.AddAsync(user1)),
    Task.Run(() => db.Users.AddAsync(user2))
);
// Race conditions, exceptions, corrupted state
```

**Фикс**: DbContext per request / per task.

### 8.5. Long-lived DbContext

```csharp
public class UserService
{
    private static AppDbContext _db = new(...);   // ❌ Singleton DbContext
}
```

**Фикс**: scoped lifetime — DI создаёт DbContext per request.

### 8.6. SaveChanges в loop

```csharp
foreach (var user in users)
{
    _db.Users.Add(user);
    await _db.SaveChangesAsync();   // ❌ N transactions, slow
}

// ✅ Bulk
_db.Users.AddRange(users);
await _db.SaveChangesAsync();   // 1 transaction
```

### 8.7. Loading whole entity for one field

```csharp
// ❌ Loading whole user just to get email
var user = await _db.Users.FindAsync(id);
var email = user.Email;
```

**Фикс**: project только нужное поле:

```csharp
var email = await _db.Users
    .Where(u => u.Id == id)
    .Select(u => u.Email)
    .FirstOrDefaultAsync();
```

### 8.8. Не AsNoTracking для read-only

```csharp
// ❌ Tracking для list view (не нужен)
var users = await _db.Users.ToListAsync();
// Memory + snapshot overhead

// ✅
var users = await _db.Users.AsNoTracking().ToListAsync();
```

### 8.9. Strings вместо enums

```csharp
public class User
{
    public string Status { get; set; } = "Active";   // ❌ string
}
```

**Фикс**: enum (mapped to int by default, или string с conversion):

```csharp
public enum UserStatus { Active, Suspended, Deleted }

public class User
{
    public UserStatus Status { get; set; } = UserStatus.Active;
}

// EF Core по default сохраняет как int. Для string:
modelBuilder.Entity<User>()
    .Property(u => u.Status)
    .HasConversion<string>();
```

### 8.10. Migration без review

```bash
dotnet ef migrations add Stuff
dotnet ef database update
# ❌ Не посмотрел что там сгенерировалось — возможно дроп таблицы!
```

**Фикс**: всегда review сгенерированный код в `Migrations/` перед применением.

> [!question]- Интервью: топ-3 ошибки с EF Core?
> 1) **N+1 query** — loop по entities + access navigation = N queries вместо 1. Fix: `Include()` для eager loading. 2) **Long-lived/shared DbContext** — DI scoped только. Не делай static DbContext, не share между threads. 3) **Не AsNoTracking для read-only** — лишний overhead change tracker. **Bonus**: SaveChanges в loop вместо AddRange + один SaveChanges. **Bonus 2**: returning Entity напрямую из API без mapping в DTO — утечка sensitive полей.

---

## 9. Cheat sheet

```csharp
// === Setup ===
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
    public DbSet<User> Users { get; set; }
    public DbSet<Order> Orders { get; set; }
}

builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(connStr));

// === Create ===
_db.Users.Add(user);
_db.Users.AddRange(users);
await _db.SaveChangesAsync();

// === Read ===
var user = await _db.Users.FindAsync(id);                          // by PK
var u2 = await _db.Users.FirstOrDefaultAsync(u => u.Email == "x"); // by predicate
var us = await _db.Users.Where(u => u.Age > 18).ToListAsync();    // filtered list
var count = await _db.Users.CountAsync();                          // count

// === Read with no tracking ===
var users = await _db.Users.AsNoTracking().ToListAsync();

// === Update ===
var user = await _db.Users.FindAsync(id);
user.Name = "Bob";
await _db.SaveChangesAsync();

// Bulk update (EF Core 7+)
await _db.Users
    .Where(u => u.IsActive)
    .ExecuteUpdateAsync(u => u.SetProperty(x => x.LastLogin, DateTime.UtcNow));

// === Delete ===
var user = await _db.Users.FindAsync(id);
_db.Users.Remove(user);
await _db.SaveChangesAsync();

// Bulk delete (EF Core 7+)
await _db.Users
    .Where(u => !u.IsActive)
    .ExecuteDeleteAsync();

// === Migrations CLI ===
// dotnet ef migrations add <Name>
// dotnet ef database update
// dotnet ef migrations remove
// dotnet ef migrations script
// dotnet ef database drop

// === Configuration — Data Annotations ===
public class User
{
    [Key]
    public int Id { get; set; }
    
    [Required]
    [MaxLength(100)]
    public string Name { get; set; } = "";
    
    [EmailAddress]
    public string Email { get; set; } = "";
}

// === Configuration — Fluent API ===
protected override void OnModelCreating(ModelBuilder mb)
{
    mb.Entity<User>(e =>
    {
        e.HasKey(u => u.Id);
        e.Property(u => u.Name).IsRequired().HasMaxLength(100);
        e.HasIndex(u => u.Email).IsUnique();
    });
}

// === Including related (eager loading) ===
var orders = await _db.Orders
    .Include(o => o.User)
    .Include(o => o.Items)
    .ToListAsync();

// === Projection (only needed columns) ===
var dtos = await _db.Users
    .Where(u => u.IsActive)
    .Select(u => new UserDto(u.Id, u.Name, u.Email))
    .ToListAsync();
```

---

## 10. Practice exercises

### 10.1. Setup blog application

Создай EF Core setup для блога:

```csharp
public class Post
{
    public int Id { get; set; }
    public string Title { get; set; } = "";
    public string Content { get; set; } = "";
    public DateTime CreatedAt { get; set; }
    public int AuthorId { get; set; }
    public Author Author { get; set; } = null!;
    public List<Comment> Comments { get; set; } = new();
}

public class Author
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public List<Post> Posts { get; set; } = new();
}

public class Comment
{
    public int Id { get; set; }
    public int PostId { get; set; }
    public Post Post { get; set; } = null!;
    public string Text { get; set; } = "";
    public DateTime CreatedAt { get; set; }
}
```

Реализуй:
- AppDbContext с DbSet'ами
- Initial migration
- BlogService с CRUD методами для Post (Create / GetById / GetByAuthor / Update / Delete)

### 10.2. Implement validation на entity level

User с правилами:
- Name required, 2-100 chars
- Email unique, valid email format
- Age 0-150
- CreatedAt всегда UTC

Реализуй через:
- Data Annotations
- Fluent API (Index unique для Email)
- Custom logic в Entity constructor

### 10.3. Optimize read-heavy endpoint

Перепиши:

```csharp
public async Task<List<UserDetailsDto>> GetAllUserDetailsAsync()
{
    var users = await _db.Users.ToListAsync();
    var result = new List<UserDetailsDto>();
    
    foreach (var user in users)
    {
        var orderCount = await _db.Orders.CountAsync(o => o.UserId == user.Id);
        result.Add(new UserDetailsDto(user.Id, user.Name, user.Email, orderCount));
    }
    
    return result;
}
```

В одну query, AsNoTracking, без N+1. Используй projection и `.Count()` в Select.

---

## 11. Что читать дальше

1. **`EFCore/Junior/ef-crud-queries.md`** — глубже про queries и filtering
2. **`EFCore/Middle/dapper-comparison.md`** — когда EF, когда Dapper
3. **`EFCore/Senior/basics-tracking.md`** — change tracking deep
4. **`EFCore/Senior/queries-performance.md`** — N+1, optimization
5. **`EFCore/Senior/relationships.md`** — one-to-many, many-to-many
6. **`EFCore/Senior/migrations.md`** — production migrations

---

## 12. См. также

- [[ef-crud-queries|EFCore/Junior/ef-crud-queries]] — следующий шаг
- [[basics-tracking|EFCore/Senior/basics-tracking]] — change tracking
- [[queries-performance|EFCore/Senior/queries-performance]] — N+1, optimization
- [[relationships|EFCore/Senior/relationships]] — entity relationships
- [[migrations|EFCore/Senior/migrations]] — production migrations
- [[dapper-comparison|EFCore/Middle/dapper-comparison]] — EF vs Dapper

---

## 13. Reading list

- **Microsoft Docs — EF Core Getting Started** — learn.microsoft.com/ef/core/get-started/
- **Microsoft Docs — DbContext** — learn.microsoft.com/ef/core/dbcontext-configuration/
- **Microsoft Docs — Migrations** — learn.microsoft.com/ef/core/managing-schemas/migrations/
- **"Entity Framework Core in Action" — Jon Smith** (book, 2nd edition)
- **Julie Lerman — EF Core blog** — thedatafarm.com
- **EF Core source code** — github.com/dotnet/efcore
- **Stack Overflow — entity-framework-core tag**
