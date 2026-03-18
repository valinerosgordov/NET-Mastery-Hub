---
tags: [efcore, concurrency, transactions, optimistic-locking]
level: Senior
---

# Concurrency и Transactions

## Что это, зачем и когда

### Что такое Concurrency (конкурентность)?
Ситуация, когда **два пользователя** одновременно меняют **одни и те же данные**. Без защиты — один перезапишет изменения другого, и данные потеряются.

**Аналогия:** Два человека редактируют один Google Doc одновременно. Если нет синхронизации — один потеряет свои изменения.

### Когда нужна защита?
- Несколько пользователей могут редактировать один ресурс (заказ, профиль)
- Фоновые процессы и HTTP-запросы могут менять одну запись одновременно
- Финансовые операции (баланс счёта — нельзя потерять списание)

### Какая защита бывает?

| Тип | Как работает | Когда |
|-----|-------------|-------|
| **Optimistic** (оптимистичная) | Проверяет «не изменились ли данные» при сохранении. Если изменились — ошибка | Конфликты РЕДКИ. Веб-приложения (99% случаев) |
| **Pessimistic** (пессимистичная) | Блокирует строку в БД, пока один пользователь работает | Конфликты ЧАСТЫ. Финансы, склады |
| **Transactions** | Группа операций — либо ВСЕ выполняются, либо НИЧЕГО | Несколько связанных изменений (перевод денег) |

---

> [!question]- **Интервью: Optimistic concurrency — как реализовать?**
> `[ConcurrencyCheck]` или `[Timestamp]` на свойстве → EF добавляет `WHERE RowVersion = @old` в UPDATE. При конфликте — `DbUpdateConcurrencyException`. Обработка: перечитать, merge или retry.

## Optimistic Concurrency

Предполагаем, что конфликты редки. При `SaveChanges` EF проверяет, не изменилась ли строка с момента чтения.

### Реализация

```csharp
// Вариант 1: RowVersion (SQL Server — rowversion/timestamp)
public class Order
{
    public Guid Id { get; set; }
    public decimal Total { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; } = null!;
}

// Вариант 2: ConcurrencyCheck на конкретном поле
public class Order
{
    public Guid Id { get; set; }

    [ConcurrencyCheck]
    public decimal Total { get; set; }
}

// Fluent API
modelBuilder.Entity<Order>()
    .Property(o => o.RowVersion)
    .IsRowVersion(); // SQL: WHERE RowVersion = @old в UPDATE
```

### Обработка конфликта

```csharp
try
{
    order.Total = newTotal;
    await context.SaveChangesAsync(ct);
}
catch (DbUpdateConcurrencyException ex)
{
    var entry = ex.Entries.Single();

    // Стратегия 1: Database wins — перезагрузить из БД
    await entry.ReloadAsync(ct);

    // Стратегия 2: Client wins — перезаписать
    entry.OriginalValues.SetValues(await entry.GetDatabaseValuesAsync(ct));
    await context.SaveChangesAsync(ct);

    // Стратегия 3: Merge — сравнить и показать пользователю
    var dbValues = await entry.GetDatabaseValuesAsync(ct);
    // Сравнить entry.CurrentValues, entry.OriginalValues, dbValues
}
```

**Нюанс:** EF генерирует `UPDATE ... WHERE Id = @id AND RowVersion = @oldVersion`. Если 0 rows affected → `DbUpdateConcurrencyException`. PostgreSQL: `xmin` system column вместо RowVersion.

---

## Transactions

### Implicit (по умолчанию)

```csharp
// SaveChanges = одна транзакция автоматически
context.Orders.Add(order);
context.OrderItems.AddRange(items);
await context.SaveChangesAsync(ct); // всё или ничего
```

### Explicit — несколько SaveChanges в одной транзакции

```csharp
await using var transaction = await context.Database.BeginTransactionAsync(ct);
try
{
    context.Orders.Add(order);
    await context.SaveChangesAsync(ct); // генерирует ID

    context.Payments.Add(new Payment { OrderId = order.Id, Amount = order.Total });
    await context.SaveChangesAsync(ct);

    await transaction.CommitAsync(ct);
}
catch
{
    await transaction.RollbackAsync(ct);
    throw;
}
```

### Кросс-контекстные транзакции

```csharp
// Вариант 1: Общее соединение
var connection = new SqlConnection(connString);
await connection.OpenAsync(ct);
await using var transaction = await connection.BeginTransactionAsync(ct);

var options1 = new DbContextOptionsBuilder<Context1>()
    .UseSqlServer(connection).Options;
var options2 = new DbContextOptionsBuilder<Context2>()
    .UseSqlServer(connection).Options;

await using var ctx1 = new Context1(options1);
await using var ctx2 = new Context2(options2);

ctx1.Database.UseTransaction(transaction);
ctx2.Database.UseTransaction(transaction);

// ... операции в обоих контекстах
await transaction.CommitAsync(ct);

// Вариант 2: TransactionScope (distributed)
using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);
// ... операции
scope.Complete();
```

**Нюанс:** `TransactionScope` с `TransactionScopeAsyncFlowOption.Enabled` — обязательно для async. Без этого флага контекст транзакции теряется при переключении потоков.

---

## Isolation Levels

```csharp
await using var transaction = await context.Database
    .BeginTransactionAsync(IsolationLevel.RepeatableRead, ct);
```

| Уровень | Dirty Read | Non-repeatable Read | Phantom Read |
|---------|-----------|---------------------|--------------|
| Read Uncommitted | Да | Да | Да |
| Read Committed (default) | Нет | Да | Да |
| Repeatable Read | Нет | Нет | Да |
| Serializable | Нет | Нет | Нет |
| Snapshot (SQL Server) | Нет | Нет | Нет (MVCC) |

**Нюанс:** PostgreSQL по умолчанию использует MVCC (Read Committed с snapshot-подобным поведением). Serializable — дороже всего, но гарантирует полную изоляцию. Для большинства CRUD — Read Committed достаточно.

---

## Connection Pooling

ADO.NET автоматически пулит соединения. Не нужно управлять вручную.

```csharp
// Соединение возвращается в пул при Dispose DbContext
// DbContext Scoped — один на HTTP-запрос, соединение берётся из пула

// Настройка пула в connection string
"Server=...;Database=...;Max Pool Size=100;Min Pool Size=5;"

// DbContext pooling (EF Core) — пулинг самих контекстов
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseNpgsql(connectionString), poolSize: 128);
```

**Нюанс:** `AddDbContextPool` — повторное использование DbContext (очистка Change Tracker). Для stateless read-only API — значительный прирост. Ограничение: нельзя внедрять Scoped-сервисы через конструктор DbContext (context пулится как Singleton-подобный).

### Connection Exhaustion

```csharp
// ✗ Утечка соединений — не Dispose контекст
var context = new AppDbContext(options); // соединение не возвращается в пул!

// ✓ Правильно — using или DI Scoped
await using var context = factory.CreateDbContext();
// Или через DI: Scoped lifetime → Dispose в конце запроса
```

---

## SaveChanges Batching

EF Core автоматически группирует команды INSERT/UPDATE/DELETE в один round-trip.

```csharp
// 1000 сущностей → ~10 batches (по умолчанию MaxBatchSize ≈ 42 для SQL Server)
context.AddRange(thousandEntities);
await context.SaveChangesAsync(ct); // не 1000 INSERT, а ~10 batched commands

// Настройка
options.UseSqlServer(connStr, o => o.MaxBatchSize(100));
```

---

## См. также

- [Interview: EF Core и SQL](../../../Interview/5-ef-core-sql.md)
