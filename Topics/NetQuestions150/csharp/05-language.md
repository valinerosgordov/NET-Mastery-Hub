# Языковые конструкции

## Generics

Параметризация типов. Type safety без boxing. Компилятор генерирует специализации для value types.

### Constraints

```csharp
public class Repository<T> where T : class, IEntity, new()
{
    // where T : class      — reference type
    // where T : struct      — value type
    // where T : new()       — параметрless конструктор
    // where T : IEntity     — реализует интерфейс
    // where T : Base        — наследует класс
    // where T : notnull     — не nullable
    // where T : unmanaged   — unmanaged type (для interop)
}
```

### Covariance / Contravariance

```csharp
// out — covariance: IEnumerable<Derived> → IEnumerable<Base>
IEnumerable<object> objects = new List<string>(); // ✓

// in — contravariance: Action<Base> → Action<Derived>
Action<Base> baseAction = b => Console.WriteLine(b);
Action<Derived> derivedAction = baseAction; // ✓

// Собственный пример
public interface IProducer<out T> { T Produce(); }      // covariant
public interface IConsumer<in T>  { void Consume(T item); } // contravariant
```

**Нюанс:** `List<T>` — не covariant (`List<string>` нельзя присвоить `List<object>`), потому что `List` и производитель, и потребитель. Только `IEnumerable<out T>`, `IReadOnlyList<out T>` — covariant.

---

## delegate, Func, Action, Event

### Delegate

```csharp
// Встроенные делегаты:
Func<int, int, bool> isGreater = (a, b) => a > b;   // возвращает значение
Action<string> log = msg => Console.WriteLine(msg);   // void
Predicate<int> isEven = x => x % 2 == 0;              // → bool

// Multicast delegate
Action<string> handler = Console.WriteLine;
handler += msg => File.AppendAllText("log.txt", msg);
handler("test"); // вызовутся оба
```

### Event — инкапсулированный delegate

```csharp
public class OrderService
{
    public event EventHandler<OrderCreatedArgs>? OrderCreated;

    public void CreateOrder(Order order)
    {
        // ... бизнес-логика
        OrderCreated?.Invoke(this, new OrderCreatedArgs(order));
    }
}

// Подписка
service.OrderCreated += (sender, args) => logger.Log(args.Order);
```

**Нюанс:** event — только `+=` и `-=` извне. Без event подписчик мог бы вызвать `Invoke` или присвоить `= null`, сломав других подписчиков.

### ref, out, in

```csharp
// ref — вход + выход, инициализация ДО вызова
void Update(ref int value) => value *= 2;

// out — только выход, ОБЯЗАН присвоить внутри метода
bool TryParse(string s, out int result);
if (int.TryParse("42", out var num)) { } // inline declaration

// in — readonly ref, без копирования для больших struct
void Process(in LargeStruct data) { } // передача по ссылке, без копии
```

**Нюанс:** `in` для struct < 16 bytes может быть медленнее, чем копия (indirection overhead). Полезно для больших readonly struct.

---

## Exception Handling

```csharp
try
{
    await ProcessAsync(ct);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    // when — фильтр, не ловит если условие false
    // НЕ раскручивает стек — можно видеть в отладчике
    return null;
}
catch (Exception ex)
{
    logger.LogError(ex, "Processing failed");
    throw;          // ✓ сохраняет stack trace
    // throw ex;    // ✗ сбрасывает stack trace
    // throw new AppException("msg", ex); // ✓ обёртка с inner exception
}
finally
{
    // Выполняется ВСЕГДА (даже при throw)
    Cleanup();
}
```

**Нюанс:** `catch when` — фильтр, выполняется ДО раскрутки стека. Полезно для логирования без перехвата:

```csharp
catch (Exception ex) when (LogAndReturnFalse(ex)) { }
// Логирует, но не ловит — исключение продолжает лететь
```

### Кастомные исключения

```csharp
public class DomainException(string message, string code)
    : Exception(message)
{
    public string Code { get; } = code;
}

public class NotFoundException(string entity, object id)
    : DomainException($"{entity} with id {id} not found", "NOT_FOUND");
```

---

## Overloading, Overriding, using

### Overloading — одно имя, разные сигнатуры

```csharp
public void Send(string message) { }
public void Send(string message, Priority priority) { }
public void Send(Email email) { }
// Компилятор выбирает перегрузку при компиляции по типам аргументов
```

### Overriding — замена virtual метода

```csharp
public class Base
{
    public virtual string Describe() => "Base";
}
public class Derived : Base
{
    public override string Describe() => "Derived";
}

Base obj = new Derived();
obj.Describe(); // "Derived" — вызов по реальному типу (vtable)
```

### using — IDisposable / IAsyncDisposable

```csharp
// Classic using
using (var connection = new SqlConnection(connStr))
{
    await connection.OpenAsync(ct);
}

// using declaration (C# 8) — Dispose при выходе из scope
using var stream = File.OpenRead("data.bin");
// stream.Dispose() вызовется при выходе из метода

// await using — IAsyncDisposable
await using var context = await factory.CreateDbContextAsync(ct);
```

---

## Switch Expression, NRT, Primary Constructors

### Switch Expression (C# 8+)

```csharp
// Возвращает значение, все ветки должны покрываться
string status = order.State switch
{
    OrderState.New => "Новый",
    OrderState.Processing => "В обработке",
    OrderState.Shipped => "Отправлен",
    _ => throw new ArgumentOutOfRangeException()  // _ — обязательная ветка
};

// Pattern matching
string Classify(object obj) => obj switch
{
    int n when n > 0 => "positive",
    int n => "non-positive",
    string { Length: > 10 } s => $"long string: {s[..10]}...",
    null => "null",
    _ => obj.GetType().Name
};
```

### Nullable Reference Types (NRT)

```csharp
#nullable enable

string name = null;   // ⚠ warning: assigning null to non-nullable
string? nullable = null; // ✓ explicit nullable

// Null-forgiving operator
string definitelyNotNull = possiblyNull!; // подавляет warning (осторожно!)

// Полезные операторы
string result = input ?? "default";       // null-coalescing
int length = input?.Length ?? 0;           // null-conditional
input ??= "fallback";                     // null-coalescing assignment
```

### Primary Constructors (C# 12)

```csharp
// Class — параметры захватываются как поля (не свойства!)
public class OrderService(IOrderRepository repo, ILogger<OrderService> logger)
{
    public async Task<Order> GetAsync(Guid id, CancellationToken ct)
    {
        logger.LogInformation("Getting order {Id}", id);
        return await repo.GetByIdAsync(id, ct)
            ?? throw new NotFoundException("Order", id);
    }
}

// Record — параметры становятся init-свойствами
public record OrderDto(Guid Id, string Customer, decimal Total);
```

**Нюанс:** в class primary constructor параметры — это captured fields, не свойства. Они не видны извне. В record — это публичные init-свойства.

---

## Culture и DateTime

```csharp
// Всегда указывать culture при форматировании
decimal price = 1234.56m;
price.ToString("C", CultureInfo.GetCultureInfo("ru-RU")); // "1 234,56 ₽"
price.ToString("C", CultureInfo.InvariantCulture);          // "¤1,234.56"

// Для хранения и передачи — всегда UTC
DateTime.UtcNow;                    // ✓ UTC
DateTimeOffset.UtcNow;              // ✓ с offset
DateTime.Now;                       // ✗ зависит от сервера

// TimeZoneInfo — конвертация для отображения
var msk = TimeZoneInfo.FindSystemTimeZoneById("Russian Standard Time");
var local = TimeZoneInfo.ConvertTimeFromUtc(DateTime.UtcNow, msk);
```

**Нюанс:** `DateTime.Kind` — Unspecified, Utc, Local. При сериализации Kind теряется. Предпочитать `DateTimeOffset` — хранит offset явно. Для дат без времени — `DateOnly` (.NET 6+).

---

## См. также

- [C# Reference: Делегаты и события](../../../Reference/csharp-delegates-events.md)
- [C# Reference: Современные фичи](../../../Reference/csharp-modern-features.md)
