---
tags: [efcore, migrations, schema, seed-data]
level: Senior
---

# Migrations и схема

## Что это, зачем и когда

### Что такое миграции?
**Git для схемы БД.** Каждое изменение (новая таблица, новый столбец, переименование) — отдельный файл-миграция. Можно применить, откатить, перенести на другой сервер.

**Аналогия:** Журнал ремонта квартиры. «10 января — добавили розетку. 15 января — перенесли дверь.» Можно «откатить» ремонт до любого состояния.

### Зачем?
- **Без миграций:** «Я добавил столбец на своей БД, забыл на сервере — упало»
- **С миграциями:** Запуск `dotnet ef database update` — БД обновляется до актуального состояния автоматически
- Все изменения в git → code review → CI/CD применяет автоматически

### Когда создавать миграцию?
- Добавил/удалил/изменил Entity или свойство
- Добавил индекс, изменил тип столбца
- **НЕ** создавай миграцию, если менял только C#-код без изменения модели данных

## Миграции

Версионирование схемы БД через код. Каждая миграция — класс с `Up()` (применить) и `Down()` (откатить). Snapshot (`*ModelSnapshot.cs`) хранит текущее состояние модели.

### Команды

```bash
# Создать миграцию
dotnet ef migrations add AddOrderTable

# Применить все pending миграции
dotnet ef database update

# Откатить до конкретной миграции
dotnet ef database update PreviousMigrationName

# Удалить последнюю (ещё не применённую) миграцию
dotnet ef migrations remove

# Сгенерировать SQL-скрипт (для production)
dotnet ef migrations script --idempotent -o migration.sql
```

### Применение в production

```csharp
// ✗ Не рекомендуется — в коде приложения
await context.Database.MigrateAsync();

// ✓ Лучше — отдельный шаг CI/CD
// 1. Сгенерировать idempotent SQL-скрипт
// 2. Ревью скрипта
// 3. Применить через CI/CD pipeline
// 4. Backup БД перед миграцией
```

**Нюанс:** `MigrateAsync()` в коде — опасно при нескольких инстансах (race condition). Решения: отдельный job, distributed lock, или idempotent SQL-скрипт.

### Кастомизация миграций

```csharp
public partial class AddOrderTable : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.CreateTable(
            name: "Orders",
            columns: table => new
            {
                Id = table.Column<Guid>(nullable: false),
                Total = table.Column<decimal>(type: "decimal(18,2)")
            },
            constraints: table => table.PrimaryKey("PK_Orders", x => x.Id));

        // Raw SQL в миграции
        migrationBuilder.Sql("CREATE INDEX IX_Orders_Total ON Orders(Total) WHERE Total > 0");
    }
}
```

---

## Seed Data

### HasData — декларативный seed

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Role>().HasData(
        new Role { Id = 1, Name = "Admin" },
        new Role { Id = 2, Name = "User" }
    );
    // Данные попадают в миграцию как InsertData
}
```

**Нюанс:** `HasData` требует явного указания ключей. При изменении данных создаётся новая миграция с UpdateData/DeleteData. Для сложного seed — отдельный метод/сервис при старте.

### Custom seed

```csharp
// При старте приложения
public static class DatabaseSeeder
{
    public static async Task SeedAsync(IServiceProvider services)
    {
        using var scope = services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();

        if (!await context.Roles.AnyAsync())
        {
            context.Roles.AddRange(Role.Admin, Role.User);
            await context.SaveChangesAsync();
        }
    }
}
```

---

## Fluent API vs Data Annotations

| Подход | Где | Преимущество |
|--------|-----|--------------|
| Data Annotations | На классе (`[Required]`, `[MaxLength]`) | Просто, видно в модели |
| Fluent API | В `OnModelCreating` | Полный контроль, не загрязняет модель |

```csharp
// Fluent API — предпочтительно для DDD
modelBuilder.Entity<Order>(entity =>
{
    entity.HasKey(o => o.Id);
    entity.Property(o => o.Total).HasPrecision(18, 2);
    entity.HasIndex(o => new { o.CustomerId, o.CreatedAt });
    entity.HasQueryFilter(o => !o.IsDeleted); // global filter
});
```

**Нюанс:** Fluent API имеет приоритет над Data Annotations. Для чистой доменной модели (DDD) — Fluent API в отдельных `IEntityTypeConfiguration<T>` файлах.

---

## См. также

- [Interview: EF Core и SQL](../../../Interview/5-ef-core-sql.md)
