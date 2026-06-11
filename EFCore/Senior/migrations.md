---
tags: [efcore, migrations, schema, seed-data, zero-downtime, dbup, fluent-migrator]
level: Senior
date: 2026-04-30
---

# Migrations и схема БД

> Полный гайд по управлению схемой БД в .NET. Закрывает: EF Core migrations, idempotent scripts, zero-downtime deployments (expand-contract pattern), distributed lock для concurrent app instances, multi-context, alternatives (DbUp, FluentMigrator, Grate).

---

## Что это, зачем и когда

### Что такое миграции?
**Git для схемы БД.** Каждое изменение (новая таблица, новый столбец, переименование) — отдельный файл-миграция. Можно применить, откатить, перенести на другой сервер.

**Аналогия:** Журнал ремонта квартиры. «10 января — добавили розетку. 15 января — перенесли дверь.» Можно «откатить» ремонт до любого состояния.

### Зачем?
- **Без миграций:** «Я добавил столбец на своей БД, забыл на сервере — упало»
- **С миграциями:** Запуск `dotnet ef database update` — БД обновляется до актуального состояния автоматически
- Все изменения в git → code review → CI/CD применяет автоматически
- Аудит: кто и когда изменил схему

### Когда создавать миграцию?
- Добавил/удалил/изменил Entity или свойство
- Добавил индекс, изменил тип столбца
- **НЕ** создавай миграцию, если менял только C#-код без изменения модели данных

### Подходы к управлению схемой

| Инструмент | Плюс | Минус |
|------------|------|-------|
| **EF Core Migrations** | Интегрировано, автогенерация | Привязка к EF, шумные миграции, проблемы с rebase |
| **DbUp** | Просто, SQL-first | Только применение, не откат |
| **FluentMigrator** | C# DSL, версионирование, rollback | Параллельный код к EF |
| **Grate / RoundhousE** | Run-once + every-time scripts | Кривая обучения |
| **Liquibase** | Cross-platform, XML/YAML | Сложнее, не C# |
| **Flyway** | Простой, version-based SQL | Community edition ограничен |

> [!info] Когда что выбрать
> **EF Core Migrations** — стартовый вариант для CRUD проектов. **DbUp + raw SQL** — когда команда DBA хочет видеть SQL и rebase в miigrations становится болью. **FluentMigrator** — когда нужны полноценные rollback и пересечение БД (Postgres + SQL Server). **Grate** — для организаций с большими SQL-стандартами.

---

## EF Core Migrations: команды

```bash
# Создать миграцию (после изменения модели)
dotnet ef migrations add AddOrderTable
dotnet ef migrations add AddOrderTable --output-dir Data/Migrations

# Применить все pending миграции
dotnet ef database update

# Применить до конкретной миграции
dotnet ef database update AddOrderTable

# Откатить до предыдущей (применит Down() последней)
dotnet ef database update PreviousMigrationName

# Откатить ВСЕ миграции (БД пуста, но __EFMigrationsHistory остаётся)
dotnet ef database update 0

# Удалить последнюю миграцию (только если ещё не применена!)
dotnet ef migrations remove

# Список применённых миграций
dotnet ef migrations list

# Сгенерировать idempotent SQL-скрипт (для production)
dotnet ef migrations script --idempotent -o migration.sql

# Скрипт от-до (для CI/CD)
dotnet ef migrations script FromMigration ToMigration --idempotent -o migration.sql

# Bundle (.NET 7+) — самодостаточный exe-файл с миграциями
dotnet ef migrations bundle -o ./efbundle
./efbundle --connection "Host=...;Database=..."
```

### Multiple DbContext

```bash
# Указать какой context
dotnet ef migrations add Init --context AppDbContext --output-dir Data/Migrations/AppDb
dotnet ef migrations add Init --context AuditDbContext --output-dir Data/Migrations/AuditDb

dotnet ef database update --context AppDbContext
```

### --startup-project и --project

Если migrations и DbContext в разных проектах:

```bash
dotnet ef migrations add Init \
    --project src/MyApp.Infrastructure \
    --startup-project src/MyApp.Api
```

---

## Анатомия миграции

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
                CustomerId = table.Column<Guid>(nullable: false),
                Total = table.Column<decimal>(type: "decimal(18,2)", nullable: false),
                CreatedAt = table.Column<DateTime>(
                    nullable: false,
                    defaultValueSql: "now() at time zone 'utc'")
            },
            constraints: table => 
            {
                table.PrimaryKey("PK_Orders", x => x.Id);
                table.ForeignKey(
                    name: "FK_Orders_Customers_CustomerId",
                    column: x => x.CustomerId,
                    principalTable: "Customers",
                    principalColumn: "Id",
                    onDelete: ReferentialAction.Restrict);
            });

        migrationBuilder.CreateIndex(
            name: "IX_Orders_CustomerId_CreatedAt",
            table: "Orders",
            columns: new[] { "CustomerId", "CreatedAt" });

        // Raw SQL — для специфичной логики
        migrationBuilder.Sql(@"
            CREATE INDEX IX_Orders_HighValue 
            ON ""Orders""(""Total"") 
            WHERE ""Total"" > 1000
        ");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropTable(name: "Orders");
    }
}
```

### __EFMigrationsHistory

Таблица в БД с историей применённых миграций:

```sql
SELECT * FROM "__EFMigrationsHistory";
-- MigrationId             | ProductVersion
-- 20260430000001_AddOrders | 9.0.0
-- 20260430000002_AddIndex  | 9.0.0
```

EF при `database update` сравнивает миграции в коде с этой таблицей и применяет только pending.

### Custom history table location (multi-tenant)

```csharp
options.UseNpgsql(connStr, b => 
    b.MigrationsHistoryTable("__MyMigrationsHistory", schema: "tenant1"));
```

---

## Idempotent SQL Scripts — production way

В production применять миграции через `dotnet ef database update` **опасно**:
- Требует .NET SDK + source code на prod ноде
- Нет ревью SQL перед применением
- Race conditions при нескольких instances

**Правильный подход — idempotent скрипт:**

```bash
dotnet ef migrations script --idempotent -o migrations.sql
```

Idempotent SQL содержит проверки `IF NOT EXISTS` для каждой миграции:

```sql
-- Каждая миграция обёрнута в проверку
DO $EF$
BEGIN
    IF NOT EXISTS(
        SELECT 1 FROM "__EFMigrationsHistory" 
        WHERE "MigrationId" = '20260430000001_AddOrders')
    THEN
        CREATE TABLE "Orders" ( ... );
        INSERT INTO "__EFMigrationsHistory" ("MigrationId", "ProductVersion") 
        VALUES ('20260430000001_AddOrders', '9.0.0');
    END IF;
END $EF$;

DO $EF$
BEGIN
    IF NOT EXISTS(
        SELECT 1 FROM "__EFMigrationsHistory" 
        WHERE "MigrationId" = '20260430000002_AddIndex')
    THEN
        CREATE INDEX "IX_Orders_CustomerId" ON "Orders"("CustomerId");
        INSERT INTO "__EFMigrationsHistory" VALUES ('20260430000002_AddIndex', '9.0.0');
    END IF;
END $EF$;
```

**Преимущества:**
- Можно прогнать дважды — ничего не сломается
- DBA может прочитать перед применением
- Применяется через `psql -f migrations.sql` без .NET
- Легко включить в CI/CD pipeline

### CI/CD pipeline пример (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 10.0.x
      
      - name: Install EF Core CLI
        run: dotnet tool install --global dotnet-ef
      
      - name: Generate migration script
        run: |
          dotnet ef migrations script --idempotent \
            --project src/MyApp.Infrastructure \
            --startup-project src/MyApp.Api \
            -o migrations.sql
      
      - name: Apply migrations
        env:
          PGPASSWORD: ${{ secrets.DB_PASSWORD }}
        run: |
          psql -h ${{ secrets.DB_HOST }} \
               -U ${{ secrets.DB_USER }} \
               -d ${{ secrets.DB_NAME }} \
               -f migrations.sql
      
      - name: Deploy app
        run: # ... deploy after migrations succeeded
```

---

## EF Migrations Bundle (.NET 7+)

Самодостаточный исполняемый файл со всеми миграциями. Не нужен .NET SDK на target машине.

```bash
# Сборка bundle
dotnet ef migrations bundle \
    --self-contained \
    --target-runtime linux-x64 \
    --output ./efbundle \
    --project src/MyApp.Infrastructure \
    --startup-project src/MyApp.Api

# Применение на target
./efbundle --connection "Host=prod-db;Database=app;Username=migrator"
```

**Применение:** Container с миграциями отдельно от main app:

```dockerfile
# Dockerfile.migrations
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet tool install --global dotnet-ef
ENV PATH="$PATH:/root/.dotnet/tools"
RUN dotnet ef migrations bundle --self-contained -r linux-x64 \
    --project src/MyApp.Infrastructure \
    --startup-project src/MyApp.Api \
    -o /app/efbundle

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-chiseled
COPY --from=build /app/efbundle /efbundle
ENTRYPOINT ["/efbundle"]
```

Запуск в k8s как `Job` или Helm `pre-upgrade hook`:

```yaml
# helm/templates/migration-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrations
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: myapp/migrations:{{ .Values.image.tag }}
          args: ["--connection", "$(DB_CONNECTION)"]
          env:
            - name: DB_CONNECTION
              valueFrom:
                secretKeyRef:
                  name: db-credentials
                  key: connection-string
      restartPolicy: OnFailure
  backoffLimit: 3
```

---

## Применение миграций при старте app — НЕ рекомендуется

```csharp
// ❌ ПРОБЛЕМЫ при нескольких instances
public class Program
{
    public static async Task Main(string[] args)
    {
        var app = builder.Build();
        
        using (var scope = app.Services.CreateScope())
        {
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            await db.Database.MigrateAsync();  // ❌ Race condition!
        }
        
        await app.RunAsync();
    }
}
```

**Проблемы:**
1. **Race condition** — 3 pod'а запустились одновременно, все делают `MigrateAsync` → DDL конфликты, иногда дубликаты в `__EFMigrationsHistory`
2. **Slow startup** — миграция блокирует запуск на 30+ секунд → readiness probe timeout
3. **Crash loop** — миграция упала на третьем pod'е (уже применена) → CrashLoopBackOff
4. **Нет rollback** — миграция применилась, но app не запустился → данные в новой схеме, код старый

### Решение 1: Distributed Lock (Postgres advisory lock)

```csharp
public static class MigrationExtensions
{
    public static async Task MigrateWithLockAsync(this AppDbContext context, CancellationToken ct)
    {
        var connection = (NpgsqlConnection)context.Database.GetDbConnection();
        await connection.OpenAsync(ct);
        
        // 1. Берём advisory lock (число — любая константа, должна совпадать у всех instances)
        await using var cmd = connection.CreateCommand();
        cmd.CommandText = "SELECT pg_advisory_lock(123456)";
        await cmd.ExecuteNonQueryAsync(ct);
        
        try
        {
            // 2. Только один instance применяет миграции
            await context.Database.MigrateAsync(ct);
        }
        finally
        {
            // 3. Освобождаем lock
            cmd.CommandText = "SELECT pg_advisory_unlock(123456)";
            await cmd.ExecuteNonQueryAsync(ct);
        }
    }
}
```

Все 3 pod'а запустятся, один возьмёт lock и применит миграции, остальные подождут.

SQL Server alternative — `sp_getapplock`:

```sql
EXEC sp_getapplock @Resource = 'EFMigrations', @LockMode = 'Exclusive', @LockOwner = 'Session';
-- ... migration ...
EXEC sp_releaseapplock @Resource = 'EFMigrations';
```

### Решение 2: Отдельный migration job

Как в Helm pre-upgrade hook выше — миграции применяются **до** rollout приложения. App запускается только когда схема готова.

---

## Zero-Downtime Migrations: Expand-Contract Pattern

Проблема: при rolling update в k8s одновременно запущены **старый** и **новый** код. Миграции, ломающие совместимость, → ошибки.

### Bad: переименование колонки

```csharp
// Миграция переименовала Email → EmailAddress
migrationBuilder.RenameColumn(name: "Email", table: "Users", newName: "EmailAddress");
```

Во время rolling deploy:
- Старые pods (50%) ищут `SELECT Email FROM Users` → ColumnDoesNotExist exception
- Новые pods (50%) работают с `EmailAddress` → норм

### Expand-Contract Pattern (3 deployments)

#### Deploy 1: **Expand** — добавить новое, не трогая старое

```csharp
// Миграция: добавить EmailAddress, скопировать данные
migrationBuilder.AddColumn<string>(name: "EmailAddress", table: "Users", nullable: true);
migrationBuilder.Sql(@"UPDATE ""Users"" SET ""EmailAddress"" = ""Email""");

// Старая колонка ОСТАЁТСЯ
// Код пишет в ОБЕ колонки — Email и EmailAddress
```

```csharp
// Code: пишем в обе
public class User
{
    public string Email { get; set; }        // legacy
    public string EmailAddress { get; set; }  // new
    
    public string GetEmail() => EmailAddress ?? Email;
    
    public void SetEmail(string value)
    {
        Email = value;
        EmailAddress = value;
    }
}
```

#### Deploy 2: **Migrate** — переключить чтение на новое

```csharp
// Code: читать только из EmailAddress
public class User
{
    public string Email { get; set; }
    public string EmailAddress { get; set; }
    
    public string GetEmail() => EmailAddress;  // только новая
    
    public void SetEmail(string value)
    {
        Email = value;          // ещё пишем в обе для безопасности
        EmailAddress = value;
    }
}
```

#### Deploy 3: **Contract** — удалить старое

```csharp
// Миграция: удалить Email
migrationBuilder.DropColumn(name: "Email", table: "Users");

// Code: только EmailAddress
public class User
{
    public string EmailAddress { get; set; }
}
```

### Шаблон применения для разных операций

| Операция | Expand | Migrate | Contract |
|----------|--------|---------|----------|
| **Переименовать колонку** | Добавить новую, дублировать запись | Читать из новой | Удалить старую |
| **Удалить колонку** | Перестать писать | Перестать читать | Удалить |
| **Изменить тип** | Добавить новую с правильным типом, дублировать | Читать из новой | Удалить старую |
| **Сделать NOT NULL** | Добавить с default, заполнить старые строки | (n/a) | ALTER COLUMN SET NOT NULL |
| **Добавить FK** | Добавить колонку nullable | Заполнить FK для всех | ALTER COLUMN SET NOT NULL + ADD CONSTRAINT |
| **Разбить таблицу** | Создать новую, начать дублировать запись | Читать из новой | Удалить старую |

### Пример: добавление NOT NULL на колонку с null'ами

```csharp
// ❌ ПРЯМО — упадёт если есть NULL
migrationBuilder.AlterColumn<string>(
    name: "Email",
    table: "Users",
    nullable: false);  // FAILS if NULL exists

// ✅ ТРЁХШАГОВО
// Step 1: добавить колонку nullable + backfill
migrationBuilder.AddColumn<string>("Email", "Users", nullable: true);
migrationBuilder.Sql(@"UPDATE ""Users"" SET ""Email"" = 'unknown@example.com' WHERE ""Email"" IS NULL");

// Step 2: deploy кода, который всегда устанавливает Email при INSERT/UPDATE

// Step 3: ALTER NOT NULL
migrationBuilder.AlterColumn<string>("Email", "Users", nullable: false);
```

---

## Backfill больших таблиц — без блокировок

```csharp
// ❌ ВРЕДНО — обновит миллионы строк, lock на минуты
migrationBuilder.Sql(@"UPDATE ""Orders"" SET ""Status"" = 'Pending' WHERE ""Status"" IS NULL");

// ✅ Batched update
migrationBuilder.Sql(@"
    DO $$
    DECLARE
        rows_affected INT;
    BEGIN
        LOOP
            WITH batch AS (
                SELECT ""Id"" FROM ""Orders""
                WHERE ""Status"" IS NULL
                LIMIT 10000
            )
            UPDATE ""Orders""
            SET ""Status"" = 'Pending'
            WHERE ""Id"" IN (SELECT ""Id"" FROM batch);
            
            GET DIAGNOSTICS rows_affected = ROW_COUNT;
            EXIT WHEN rows_affected = 0;
            
            -- Опционально: pg_sleep для снижения нагрузки
            PERFORM pg_sleep(0.1);
        END LOOP;
    END $$;
");
```

Для очень больших таблиц — лучше **отдельный backfill job** (BackgroundService или Hangfire), а не миграция.

---

## Кастомизация миграций

### Custom default value

```csharp
migrationBuilder.AddColumn<DateTime>(
    name: "CreatedAt",
    table: "Orders",
    type: "timestamp with time zone",
    nullable: false,
    defaultValueSql: "now() at time zone 'utc'");
```

### Computed columns

```csharp
migrationBuilder.AddColumn<decimal>(
    name: "TotalWithTax",
    table: "Orders",
    type: "decimal(18,2)",
    nullable: false,
    computedColumnSql: "\"Total\" * 1.20",
    stored: true);  // PERSISTED — рассчитывается при INSERT/UPDATE
```

### Triggers и stored procedures

```csharp
migrationBuilder.Sql(@"
    CREATE OR REPLACE FUNCTION update_modified_timestamp()
    RETURNS TRIGGER AS $$
    BEGIN
        NEW.""UpdatedAt"" = now();
        RETURN NEW;
    END;
    $$ LANGUAGE plpgsql;
    
    CREATE TRIGGER orders_modified_trigger
    BEFORE UPDATE ON ""Orders""
    FOR EACH ROW EXECUTE FUNCTION update_modified_timestamp();
");
```

### CHECK constraints

```csharp
migrationBuilder.AddCheckConstraint(
    name: "CK_Orders_Total_Positive",
    table: "Orders",
    sql: "\"Total\" > 0");
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
}
```

Данные попадают в миграцию как `InsertData`. При изменении — генерируется `UpdateData`/`DeleteData`.

**Ограничения:**
- Все ключи должны быть указаны явно
- Не работает с auto-generated GUID
- Не подходит для динамических данных

### Custom seed через IServiceProvider

```csharp
// MyApp.Infrastructure/DatabaseSeeder.cs
public static class DatabaseSeeder
{
    public static async Task SeedAsync(IServiceProvider services, CancellationToken ct = default)
    {
        using var scope = services.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        var logger = scope.ServiceProvider.GetRequiredService<ILogger<AppDbContext>>();
        
        // Apply migrations
        var pending = await context.Database.GetPendingMigrationsAsync(ct);
        if (pending.Any())
        {
            logger.LogInformation("Applying {Count} pending migrations", pending.Count());
            await context.Database.MigrateAsync(ct);
        }
        
        // Seed reference data
        if (!await context.Roles.AnyAsync(ct))
        {
            context.Roles.AddRange(
                new Role { Name = "Admin" },
                new Role { Name = "User" });
            await context.SaveChangesAsync(ct);
        }
        
        // Seed dev data только в Development
        var env = scope.ServiceProvider.GetRequiredService<IHostEnvironment>();
        if (env.IsDevelopment() && !await context.Users.AnyAsync(ct))
        {
            context.Users.AddRange(
                new User { Email = "admin@local", PasswordHash = "..." },
                new User { Email = "user@local", PasswordHash = "..." });
            await context.SaveChangesAsync(ct);
        }
    }
}

// Program.cs
var app = builder.Build();
await DatabaseSeeder.SeedAsync(app.Services);
await app.RunAsync();
```

### Bogus для генерации тестовых данных

```csharp
using Bogus;

var faker = new Faker<Customer>()
    .RuleFor(c => c.Id, _ => Guid.NewGuid())
    .RuleFor(c => c.Email, f => f.Internet.Email())
    .RuleFor(c => c.Name, f => f.Name.FullName())
    .RuleFor(c => c.RegisteredAt, f => f.Date.Past());

var customers = faker.Generate(1000);
context.Customers.AddRange(customers);
await context.SaveChangesAsync();
```

---

## Альтернативы EF Core Migrations

### DbUp — SQL-first

Простой подход — обычные SQL-файлы, применяются по очереди. Ничего не генерирует.

```csharp
// Program.cs
var upgrader = DeployChanges.To
    .PostgresqlDatabase(connectionString)
    .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly())
    .LogToConsole()
    .WithTransaction()
    .Build();

var result = upgrader.PerformUpgrade();
if (!result.Successful)
{
    Console.Error.WriteLine(result.Error);
    return -1;
}
```

```
Scripts/
├── 0001_CreateOrdersTable.sql
├── 0002_AddIndexOnCustomerId.sql
└── 0003_AddPaymentsTable.sql
```

```sql
-- Scripts/0001_CreateOrdersTable.sql
CREATE TABLE "Orders" (
    "Id" UUID PRIMARY KEY,
    "CustomerId" UUID NOT NULL,
    "Total" DECIMAL(18,2) NOT NULL
);
```

**Плюсы:** SQL DBA понимает, lock + checksum в `SchemaVersions` таблице, без EF.
**Минусы:** нет rollback, нет автогенерации.

### FluentMigrator — C# DSL с rollback

```csharp
[Migration(202604300001)]
public class AddOrdersTable : Migration
{
    public override void Up()
    {
        Create.Table("Orders")
            .WithColumn("Id").AsGuid().PrimaryKey()
            .WithColumn("CustomerId").AsGuid().NotNullable()
            .WithColumn("Total").AsDecimal(18, 2).NotNullable()
            .WithColumn("CreatedAt").AsDateTime2().WithDefault(SystemMethods.CurrentUTCDateTime);
        
        Create.Index("IX_Orders_CustomerId").OnTable("Orders").OnColumn("CustomerId");
    }

    public override void Down()
    {
        Delete.Table("Orders");
    }
}
```

**Плюсы:** rollback, cross-database (один код для PG/SQL Server/MySQL), теги для условного применения.
**Минусы:** параллельный код к EF Core, нет автогенерации из модели.

### Grate (бывший RoundhousE)

Run-once + every-time scripts:

```
db/
├── up/                         # run-once миграции
│   ├── 0001_CreateOrders.sql
│   └── 0002_AddIndex.sql
├── runAfterCreateDatabase/    # every-time после create
│   └── 0001_CreateExtensions.sql
├── views/                     # every-time recompile views
│   └── vw_OrderSummary.sql
├── sprocs/                    # every-time recompile sprocs
│   └── sp_GetOrderTotal.sql
└── permissions/               # every-time
    └── grant.sql
```

```bash
grate --connectionstring="..." --sqlfilesdirectory=./db --databasetype=postgresql
```

**Плюсы:** разделение run-once и every-time, идеально для views/sprocs.
**Минусы:** не все знают, кривая обучения.

---

## Multi-Tenant и Multi-DbContext миграции

### Сценарий: разные БД для разных tenant'ов

```bash
# Создание миграции одного раза
dotnet ef migrations add Init --context TenantDbContext

# Применение для каждого tenant
foreach tenant in ($(get-tenants)) {
    dotnet ef database update --context TenantDbContext \
        --connection "Host=...;Database=tenant_$tenant"
}
```

### Программно — apply per-tenant

```csharp
public class TenantMigrationService(
    IServiceProvider sp,
    ITenantStore tenants,
    ILogger<TenantMigrationService> logger) : IHostedService
{
    public async Task StartAsync(CancellationToken ct)
    {
        var allTenants = await tenants.GetAllAsync(ct);
        
        await Parallel.ForEachAsync(
            allTenants,
            new ParallelOptions { CancellationToken = ct, MaxDegreeOfParallelism = 5 },
            async (tenant, token) =>
            {
                using var scope = sp.CreateScope();
                var factory = scope.ServiceProvider
                    .GetRequiredService<IDbContextFactory<TenantDbContext>>();
                
                using var context = await factory.CreateDbContextAsync(token);
                context.Database.SetConnectionString(tenant.ConnectionString);
                
                logger.LogInformation("Migrating tenant {TenantId}", tenant.Id);
                await context.Database.MigrateAsync(token);
            });
    }
    
    public Task StopAsync(CancellationToken ct) => Task.CompletedTask;
}
```

---

## Fluent API vs Data Annotations

| Подход | Где | Преимущество |
|--------|-----|--------------|
| Data Annotations | На классе (`[Required]`, `[MaxLength]`) | Просто, видно в модели |
| Fluent API | В `OnModelCreating` или `IEntityTypeConfiguration<T>` | Полный контроль, не загрязняет модель |

```csharp
// IEntityTypeConfiguration — изоляция конфигурации
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.HasKey(o => o.Id);
        builder.Property(o => o.Total).HasPrecision(18, 2);
        builder.HasIndex(o => new { o.CustomerId, o.CreatedAt });
        builder.HasQueryFilter(o => !o.IsDeleted);
        
        // Backing field для DDD aggregate
        builder.Metadata
            .FindNavigation(nameof(Order.Items))!
            .SetPropertyAccessMode(PropertyAccessMode.Field);
    }
}

// DbContext
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(OrderConfiguration).Assembly);
}
```

**Нюанс:** Fluent API имеет приоритет над Data Annotations. Для DDD — Fluent API в отдельных конфигурациях, чистая доменная модель.

---

## Common Pitfalls

### 1. Миграция содержит `DELETE` или `DROP` — потенциальная потеря данных

```csharp
migrationBuilder.DropColumn(name: "OldField", table: "Users");
// Данные потеряны навсегда!
```

Всегда ревью миграций. Для удаления — expand-contract pattern.

### 2. Миграция блокирует таблицу на долго

`ALTER TABLE ADD COLUMN NOT NULL DEFAULT` на больших таблицах в Postgres < 11 → table rewrite → длинный lock.

В Postgres 11+ — добавление колонки с constant default не пишет данные (быстро). Но вычисляемый default — всё ещё rewrite.

**Решение:** добавить nullable, backfill batched, потом `ALTER COLUMN SET NOT NULL`.

### 3. CREATE INDEX блокирует записи

```sql
-- ❌ Блокирует INSERT/UPDATE на время создания
CREATE INDEX IX_Orders_Total ON Orders(Total);

-- ✅ Postgres CONCURRENTLY — без блокировки
CREATE INDEX CONCURRENTLY IX_Orders_Total ON Orders(Total);
```

В EF миграции:
```csharp
migrationBuilder.Sql("CREATE INDEX CONCURRENTLY IF NOT EXISTS \"IX_Orders_Total\" ON \"Orders\"(\"Total\")");
```

> [!warning] CONCURRENTLY вне транзакции
> EF миграции по умолчанию в транзакции. `CREATE INDEX CONCURRENTLY` нельзя в транзакции. Решение: `migrationBuilder.Sql("...", suppressTransaction: true)`.

### 4. Rebase ветки с миграциями

```
main:    M1 → M2 → M3
feature: M1 → M2 → M2_feature_X

После rebase:
main:    M1 → M2 → M3 → M4 → M5
feature: ... → M5 → M2_feature_X  ← timestamp в прошлом!
```

Когда merge feature в main — `M2_feature_X` имеет timestamp **раньше** последней миграции main, но применяется **после** них. EF может пытаться применить уже применённое.

**Решения:**
1. После rebase — пересоздать миграцию с новым timestamp: `dotnet ef migrations remove` → `dotnet ef migrations add`
2. Использовать DbUp / FluentMigrator — там просто номер, без timestamp
3. Дисциплина команды — мерджить feature ASAP

### 5. Миграция в коде vs БД desync

После `git pull` появилась новая миграция от коллеги. Твоя локальная БД — на старой миграции. `dotnet ef database update` применит новую, но **`ModelSnapshot.cs`** уже изменён в pull → EF может думать что pending changes есть.

```bash
# Откатить локально к чистому состоянию
dotnet ef database update 0  # снести всё
dotnet ef database update    # применить заново
```

Для PG — иногда проще `DROP DATABASE && CREATE DATABASE && dotnet ef database update`.

### 6. Слишком много миграций в репо

Через год проекта — 200+ миграций. CI пересоздаёт БД с нуля при каждом тесте → медленно.

**Решение:** **squash** старых миграций.

```bash
# 1. Сделать snapshot текущего состояния как новую миграцию
dotnet ef migrations remove  # ... повторить для всех старых
# или удалить файлы вручную

# 2. Сгенерировать одну миграцию "Initial"
dotnet ef migrations add Initial

# 3. На существующих БД — синхронизировать __EFMigrationsHistory вручную:
INSERT INTO "__EFMigrationsHistory" ("MigrationId", "ProductVersion") 
VALUES ('20260430000001_Initial', '10.0.0');
```

Делать squash раз в год / квартал, координируя с командой.

### 7. Migration apply на read-replica

Связь к replica → ошибка `cannot execute CREATE TABLE in a read-only transaction`. Migration runner должен подключаться к **primary**.

```csharp
// connection string должен быть к primary
"Host=primary-db.example.com;..."  // не replica
```

### 8. Sensitive data в seed

```csharp
// ❌ Production пароль в коде!
modelBuilder.Entity<User>().HasData(new User 
{ 
    Email = "admin@prod.com", 
    PasswordHash = "$argon2id$..."  // прошитый хеш
});
```

Решение: seed через переменные окружения / secrets manager, не через `HasData`.

---

## Production Checklist

- [ ] Migrations генерируются в CI как idempotent SQL
- [ ] SQL-скрипт ревьювится перед применением
- [ ] Миграции применяются **отдельным шагом** до deploy app (Helm pre-upgrade hook / k8s Job)
- [ ] Distributed lock (advisory_lock / sp_getapplock) если apply в коде app
- [ ] Backups делаются **до** применения миграций
- [ ] Rollback план: `Down()` миграции + script восстановления данных
- [ ] Большие миграции (backfill, ALTER на млн строк) — отдельным фоновым job'ом, не в migration
- [ ] CREATE INDEX CONCURRENTLY (PG) или ONLINE (SQL Server) для production
- [ ] Expand-Contract pattern для breaking changes
- [ ] `__EFMigrationsHistory` мониторится (ALERT если рост остановился)
- [ ] Команда знает процедуру rebase миграций
- [ ] Multi-tenant — миграции применяются для всех tenants одной командой
- [ ] CI прогоняет `dotnet ef migrations add Test` и проверяет что нет diff (модель = БД)

---

## Cheat sheet

| Need | EF Core API |
|------|-------------|
| Read-only query | `.AsNoTracking()` |
| Read with relations | `.Include(o => o.Items)` |
| Read только некоторые fields | `.Select(o => new OrderDto { ... })` (projection) |
| Filter | `.Where(predicate)` |
| Pagination | `.OrderBy().Skip(n).Take(n)` |
| Conditional include | `.Include(o => o.Items.Where(i => i.IsActive))` |
| Bulk update (EF 7+) | `.ExecuteUpdateAsync(s => s.SetProperty(...))` |
| Bulk delete (EF 7+) | `.ExecuteDeleteAsync()` |
| Raw SQL | `.FromSqlRaw("...", params)` |
| Track changes | default (Add, Update modify entities) |
| Detach | `_db.Entry(entity).State = EntityState.Detached` |
| Stop tracking всё | `_db.ChangeTracker.Clear()` |
| Optimistic concurrency | `[Timestamp]` или `IsConcurrencyToken()` |
| Soft delete | Global query filter |
| Audit columns | `SaveChanges` override |
| Compiled query | `EF.CompileAsyncQuery(...)` |

| Loading strategy | When |
|------------------|------|
| Eager (`Include`) | Знаешь что нужны related entities |
| Explicit (`.Reference().Load()`) | Иногда нужны (lazy без proxies) |
| Lazy (proxies) | API endpoints с unpredictable access — но careful с N+1 |
| Projection (`Select`) | API DTO — самое efficient |
| Split query | Multiple Includes на разные collections |


---

## Decision tree

```
EF Core решение?
│
├── Read-only query?
│   ├── Single entity → Find / FirstOrDefault
│   ├── List → AsNoTracking + ToListAsync
│   ├── Aggregation → DB-level (Sum, Count, GroupBy)
│   └── Big result set → IAsyncEnumerable + streaming
│
├── Need related data?
│   ├── 1 collection → Include
│   ├── Multiple collections → AsSplitQuery (.NET 5+)
│   ├── Только некоторые fields → Projection (Select)
│   └── Filtered → Include + Where (filtered include)
│
├── Bulk operation?
│   ├── EF 7+ → ExecuteUpdate / ExecuteDelete
│   ├── EF 6 — → Dapper или raw SQL
│   └── Очень большой volume → SqlBulkCopy
│
├── Performance critical?
│   ├── Hot path → Compiled query (EF.CompileAsyncQuery)
│   ├── Read-heavy → Dapper (часто 2-3x faster)
│   └── Complex logic → Stored procedures
│
├── Concurrency?
│   ├── Last-write-wins → no concurrency token
│   ├── Optimistic → [Timestamp] / RowVersion
│   ├── Pessimistic → SELECT FOR UPDATE (raw SQL)
│   └── Compensating → SAGA pattern
│
└── Migration approach?
    ├── Code-first → Add-Migration → Update-Database
    ├── DB-first → Scaffold-DbContext (reverse engineering)
    └── Hybrid → manual migrations + handcraft schema
```


---

## Best Practices

### Best Practices for Migrations

- **Migrations в Git** — review как code в PR
- **One migration per feature** — не накапливай долго (труднее merge)
- **Никогда не edit applied migration** — создай новую
- **Auto-migrate в Production = anti-pattern** — explicit step в CI/CD
- **Backup перед migration** — особенно для destructive changes
- **Test migration на staging** с production-like data
- **Idempotent SQL where possible** (`IF NOT EXISTS` etc.)
- **Down migration** должна работать — иначе rollback impossible
- **Schema evolution patterns** — expand → contract:
  1. Add new column nullable
  2. Backfill data в background
  3. Make column NOT NULL
  4. Remove old column (separate migration)
- **Naming convention** — `2026_05_01_AddUserAvatarColumn` (timestamp + descriptive)
- **EF Bundle** для production deploy: `dotnet ef migrations bundle`
- **Multi-environment** — same migrations dev/staging/prod

### When migrations не нужны

- DB-first projects (DBA-managed schemas)
- Read-only databases
- NoSQL (Mongo, Cosmos) — schema-less


---

## См. также

- [EF Core Basics & Tracking](basics-tracking.md)
- [EF Core Concurrency](concurrency.md)
-[EF Core Patterns]())
-[Kubernetes — Helm pre-upgrade hooks]())
-[Docker — миграции в контейнере]())
-[PostgreSQL Deep — Advisory Locks]())
-[SQL Optimization — индексы]())
-[Distributed Systems — Outbox через миграции]())

## Reading list

- **Microsoft Docs — EF Core Migrations** — learn.microsoft.com/ef/core/managing-schemas/migrations
- **DbUp documentation** — dbup.readthedocs.io
- **FluentMigrator** — github.com/fluentmigrator/fluentmigrator
- **Grate** — github.com/erikbra/grate
- **Strong Migrations** — патерны от Andrew Lock — andrewlock.net/series/safely-migrating-databases/
- **Martin Fowler — Evolutionary Database Design** — martinfowler.com/articles/evodb.html
- **Database Reliability Engineering** — book by Laine Campbell & Charity Majors (про zero-downtime deployment)
- **Refactoring Databases** — book by Scott Ambler (классика по эволюции схемы)
