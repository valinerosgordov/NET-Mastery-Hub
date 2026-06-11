---
tags: [csharp, naming-conventions, junior, code-style, framework-design-guidelines, stylecop]
level: Junior
date: 2026-05-04
---

# Naming conventions — соглашения по именованию

> **PascalCase, camelCase, и почему `Id` лучше `ID`.** Microsoft Framework Design Guidelines, naming patterns (Async suffix, Exception suffix, IDisposable interface), Hungarian notation как anti-pattern, EditorConfig + StyleCop. Закрывает пробел: «знаю про camelCase, не понимаю, почему `httpClient`, а не `HTTPClient`».

---

## 0. Как читать этот файл

Если ты впервые пишешь C# — читай разделы 1→4 подряд: получишь рабочую модель и поймёшь, **где какой стиль**. Если уже пишешь, но непонятно про edge cases (HTTPClient vs HttpClient, ID vs Id) — раздел 8 (аббревиатуры). Если нужно настроить tooling — раздел 15 (.editorconfig + StyleCop + analyzers). Если работаешь в команде с разнобоем — раздел 17 (refactoring).

Все примеры самостоятельные и compile-able. Cross-language якоря (`> [!info]-`) свёрнуты — раскрывай если переходишь из Java / Python / Rust / Go / Kotlin. Interview-вопросы (`> [!question]-`) встроены рядом с теорией.

---

## 1. Что это, зачем и когда

### 1.1. Проблема разнобоя

Сравни два проекта:

```csharp
// Проект А
public class user_service
{
    private readonly DataBase DB;
    public USER GetUser(int user_id) { ... }
    public void delete_user(int ID) { ... }
}

// Проект Б
public class UserService
{
    private readonly IDatabase _db;
    public User GetUser(int userId) { ... }
    public void DeleteUser(int id) { ... }
}
```

Оба компилируются. Оба работают. Но **читать** проект А — мука: каждый раз тратишь mental cycles на парсинг "а это камелкейс или нет, это публичное или приватное". Проект Б читается с одного взгляда — мозг знает, чего ожидать.

Naming convention — это **shared mental model в команде**. Без неё каждый файл — новая загадка.

### 1.2. Зачем единый стиль

1. **Читаемость** — глаз обучается шаблонам. `_field`, `Property`, `localVar` мгновенно понятны без чтения.
2. **Code review proceedings** — без споров о стиле, фокус на логике.
3. **Refactoring безопасен** — IDE rename работает по токенам, согласованные имена легко находить.
4. **Onboarding** — новые разработчики знают conventions из BCL (которая огромный реальный пример).
5. **Tooling** — analyzers (StyleCop, FxCop, Roslyn) могут автоматически проверять.

### 1.3. Microsoft Framework Design Guidelines

C# имеет официальные Microsoft Framework Design Guidelines, которые применяет команда .NET к BCL и `System.*` namespace. Эти правила — **де-факто стандарт** для всего C# мира:

- Книга «Framework Design Guidelines» (Krzysztof Cwalina, Brad Abrams) — третье издание 2020.
- Microsoft Docs — обзор.
- BCL сама — самый большой example.

Это не просто «как Microsoft делает». Это правила, которые миллионы разработчиков прочно знают. Отступление от них = когнитивная нагрузка.

### 1.4. Главные правила

```
PascalCase   — types, members, namespaces (UserService, GetById, DataAccess)
camelCase    — local variables, parameters, private fields (с префиксом _)
SCREAMING_   — нет в C# (Java/JS-стиль)
_underscore  — приватные поля (_database)
I-prefix     — interfaces (IRepository)
T-prefix     — generic-параметры (TKey, TValue, T)
Async-suffix — async-методы (LoadAsync, SaveAsync)
Exception    — exception типы (NotFoundException)
```

### 1.5. Эволюция: .NET Framework 1.0 → .NET 10

| Период | Стиль |
|--------|-------|
| **VB6 / pre-.NET** | Hungarian notation (`strName`, `intCount`) |
| **.NET Framework 1.0 (2002)** | Microsoft Framework Design Guidelines — отказ от Hungarian, PascalCase / camelCase |
| **.NET Framework 2.0+** | Generic constraints, `<T>` префикс для generic-параметров |
| **C# 4-7** | Async-suffix принят (.NET 4.5 TPL) |
| **.NET Core 1.0 (2016)** | Унификация — те же rules для cross-platform |
| **.NET 5+** | Stable Framework Design Guidelines |
| **C# 9-12** | Records — те же naming rules (PascalCase для type, properties) |
| **.NET 8/9/10** | Roslyn analyzers по умолчанию — стиль автоматически проверяется |

### 1.6. Когда использовать какой стиль

```
Тип (class, struct, interface, enum, delegate, record)
  → PascalCase

Метод, свойство (Property), event, public field, constant
  → PascalCase

Параметр, локальная переменная
  → camelCase

Приватное поле
  → _camelCase (с подчёркиванием)

Generic параметр
  → T (один) или TKey, TValue (многозначные)

Namespace
  → PascalCase, dot-separated (MyApp.Domain.Orders)

Файл
  → PascalCase.cs (соответствует имени основного типа)
```

> [!info]- Если ты знаешь Java / Python / Rust / Go / Kotlin
> **Java:** ровно та же модель — PascalCase для классов, camelCase для методов/полей. Главное отличие — Java использует `final` для const, нет underscore-prefix для private (по convention). Constants — `SCREAMING_SNAKE_CASE` (в C# — PascalCase).
>
> **Python:** другой стиль — snake_case для всего (методы, поля, файлы), `PascalCase` только для классов. C# convention выглядит чуждо. Ключевое — `_field` для приватных (как в C#) и `__name_mangling`.
>
> **Rust:** snake_case для функций, переменных, файлов; PascalCase для типов; SCREAMING_SNAKE для constants. Compiler даёт warning при отступлении (`#[warn(non_snake_case)]`). Очень строгий tooling.
>
> **Go:** PascalCase для public (exported), camelCase для private — visibility через case. Файлы — snake_case.go. Compiler enforced.
>
> **Kotlin:** copies Java + некоторые улучшения. PascalCase для classes/interfaces, camelCase для functions/properties. Сильное предпочтение immutable values.

> [!question]- Интервью: что такое Microsoft Framework Design Guidelines?
> Это документированные правила Microsoft для проектирования API и naming в .NET BCL. Оригинальная книга Krzysztof Cwalina + Brad Abrams (2008, 3-е издание 2020), также в Microsoft Docs. Описывает: PascalCase / camelCase правила, naming patterns (Async-suffix, Exception-suffix, IDisposable), abbreviations (Id vs ID, Http vs HTTP), API design (immutability, async, exceptions). Все .NET-разработчики знакомы с этими conventions через BCL. Отступление = когнитивная нагрузка для team-mates.

---

## 2. Базовые правила case

### 2.1. PascalCase

Каждое слово начинается с большой буквы, без разделителей:

```
UserService
GetUserById
LoadDataAsync
DateTime
HttpClient
```

В C# применяется к:
- Типам (class, struct, interface, enum, delegate, record).
- Public/protected/internal members (method, property, field, event, constant).
- Namespace (`MyApp.Domain.Orders`).
- Generic-параметрам (`TKey`, `TValue`).
- Файлам (`UserService.cs`).

### 2.2. camelCase

Первое слово с маленькой буквы, остальные с большой, без разделителей:

```
userService    (local)
loadData       (parameter)
totalAmount    (local)
isActive       (local boolean)
```

В C# применяется к:
- Локальным переменным.
- Параметрам методов / конструкторов.
- Приватным полям с префиксом `_` (`_database`, `_logger`).

### 2.3. _camelCase (приватные поля)

Подчёркивание + camelCase:

```csharp
public class UserService
{
    private readonly IDatabase _database;
    private readonly ILogger<UserService> _logger;
    private int _cacheHits;
}
```

Это **convention** Microsoft (и большинства проектов) для приватных полей класса. Зачем:

1. **Читаемость** — сразу видно, что это поле, а не локалка.
2. **Disambiguation** — отличает от parameter одинаковым именем:
   ```csharp
   public UserService(IDatabase database, ILogger<UserService> logger)
   {
       _database = database;       // _database vs database — ясно
       _logger = logger;
   }
   ```
3. **No `this.` в коде** — без префикса нужно `this.database = database;`, что многословно.

Альтернативный стиль (без `_`) тоже встречается, но менее распространён в C#.

### 2.4. SCREAMING_SNAKE_CASE — нет в C#

В Java / JavaScript / Python для constants:

```java
// Java
public static final int MAX_RETRY_COUNT = 3;
```

В C# **нет** этого — constants и static readonly используют PascalCase:

```csharp
// C#
public const int MaxRetryCount = 3;
public static readonly TimeSpan DefaultTimeout = TimeSpan.FromSeconds(30);
```

Это исторически отличается от Java и часто вызывает удивление у переходящих с других языков. Microsoft Framework Design Guidelines явно: const = PascalCase.

### 2.5. ALL_CAPS — нет в C#

Никогда не используй:

```csharp
public const int MAX_USERS = 100;     // ❌ не C# style
public string DB_CONNECTION = "...";   // ❌
```

Вместо:

```csharp
public const int MaxUsers = 100;       // ✅
public string DbConnection = "...";    // ✅
```

### 2.6. Никаких префиксов типа

Hungarian notation (vbScript / pre-.NET MFC) — **НЕ применяется** в C#:

```csharp
// ❌ Hungarian — не C#
string strName;
int intCount;
bool bIsActive;

// ✅ C# style — type clear from declaration
string name;
int count;
bool isActive;
```

C# имеет статическую типизацию — IDE и компилятор знают типы. Префикс не добавляет информации, только шум.

### 2.7. Без разделителей-символов

```csharp
// ❌ snake_case в C# — не идиоматично
public class user_service { }
public int user_id;
public void load_data() { }

// ✅
public class UserService { }
public int UserId;
public void LoadData() { }
```

Snake_case подходит Python/Rust, но не C#.

> [!question]- Интервью: чем PascalCase отличается от camelCase?
> PascalCase — все слова начинаются с большой буквы, без разделителей: `UserService`, `GetById`. camelCase — первое слово маленькое, остальные большие: `userService`, `getById`. В C#: PascalCase для типов и публичных членов (классы, методы, свойства), camelCase для локальных переменных и параметров, `_camelCase` для приватных полей. Microsoft Framework Design Guidelines строго определяет, где какой стиль. Отличие от Java (та же модель, но без underscore для private) и Python/Rust (snake_case).

---

## 3. Соглашения по типам

### 3.1. Class — PascalCase, существительное

```csharp
public class UserService { }
public class OrderRepository { }
public class CustomerNotFoundException { }
public class HttpRequestBuilder { }
```

**Правило:** имя класса = существительное, описывающее, **что это**, а не **что делает**. Действия — на методах.

```csharp
// ❌ Плохо — глагольная форма для класса
public class CalculatePrice { }

// ✅ Хорошо — существительное
public class PriceCalculator { }
```

### 3.2. Suffix patterns — стандартные окончания

Microsoft FDG предлагает стандартные suffixes для категорий типов:

| Suffix | Что означает | Примеры |
|--------|--------------|---------|
| **`Service`** | Бизнес-логика | `OrderService`, `EmailService` |
| **`Repository`** | Data access | `UserRepository`, `OrderRepository` |
| **`Controller`** | Web API endpoints | `UsersController`, `OrdersController` |
| **`Manager`** | Координация (используй умеренно) | `ConnectionManager` |
| **`Provider`** | Источник данных / configuration | `ConfigurationProvider`, `LoggerProvider` |
| **`Builder`** | Builder pattern | `HttpRequestBuilder`, `QueryBuilder` |
| **`Factory`** | Factory pattern | `OrderFactory`, `ConnectionFactory` |
| **`Handler`** | Event/command handler | `MessageHandler`, `KeyPressHandler` |
| **`Helper` / `Util`** | Static helpers (используй умеренно) | `StringHelper`, `MathUtil` |
| **`Exception`** | Исключение | `NotFoundException`, `ValidationException` |
| **`Attribute`** | Атрибут | `RequiredAttribute`, `RouteAttribute` |
| **`EventArgs`** | Аргументы события | `OrderCreatedEventArgs` |
| **`Options`** | DI-options | `JwtOptions`, `EmailOptions` |
| **`Settings` / `Configuration`** | Конфигурация | `DatabaseSettings` |
| **`Dto`** | Data Transfer Object | `UserDto`, `OrderDto` |
| **`ViewModel`** | UI binding model (MVVM) | `LoginViewModel` |
| **`Request` / `Response`** | API contracts | `CreateOrderRequest`, `OrderResponse` |
| **`Command` / `Query`** | CQRS | `CreateOrderCommand`, `GetOrderQuery` |

### 3.3. Interface — `I` префикс + PascalCase

```csharp
public interface IRepository<T> { }
public interface IDisposable { }
public interface IEnumerable<out T> { }
public interface IUserService { }
```

**`I` префикс** — единственное оставшееся место Hungarian-style в C#. Убрать из BCL уже невозможно (legacy), и Microsoft в FDG явно предписывает использовать.

Обоснование:
- Сразу видно, что это interface, а не class.
- При `IList<T>` vs `List<T>` — visually контрастно.
- Convention BCL — все знают.

Альтернативный стиль (Java-like, без `I` префикса) встречается в некоторых проектах, но **сильно меньшинство** в C# мире.

### 3.4. Struct — PascalCase, существительное

```csharp
public struct Point { ... }
public struct Money { ... }
public readonly struct Coordinate { ... }
```

Никаких префиксов. Имя описывает данные, которые struct хранит.

### 3.5. Enum — PascalCase + единственное число

```csharp
public enum OrderStatus { ... }      // ✅ единственное число
public enum DayOfWeek { ... }        // ✅
public enum LogLevel { ... }         // ✅

public enum OrderStatuses { ... }    // ❌ множественное
public enum DaysOfWeek { ... }       // ❌
```

Исключение — `[Flags]` enum, где имя описывает набор флагов:

```csharp
[Flags]
public enum Permissions { Read, Write, Delete }   // ✅ множественное OK
```

### 3.6. Delegate — PascalCase, описывает signature

```csharp
public delegate void EventHandler<T>(object sender, T args);
public delegate TResult Func<in T, out TResult>(T arg);
public delegate bool Predicate<in T>(T item);
public delegate void Action<in T>(T arg);
```

Если delegate описывает specific event handler — суффикс `Handler` или `Callback`:

```csharp
public delegate void OrderCreatedHandler(Order order);
public delegate Task<Result> AsyncOperationCallback(int id);
```

### 3.7. Record — как class

Record — это синтаксический сахар над class (или struct в record struct). Naming идентичен:

```csharp
public record User(int Id, string Name);
public record CreateOrderCommand(int UserId, decimal Amount);
public record struct Point(int X, int Y);
```

Параметры primary constructor — PascalCase (они становятся public properties).

### 3.8. Generic-параметры — `T` или `T<Description>`

```csharp
public class List<T> { ... }                          // ✅ один параметр — T
public class Dictionary<TKey, TValue> { ... }          // ✅ многозначный — описательно
public delegate TResult Func<in T, out TResult>();    // ✅
public class Repository<TEntity, TKey> { ... }
```

**Правило:**
- Один generic-параметр — просто `T`.
- Несколько — описательные имена с префиксом `T`: `TKey`, `TValue`, `TEntity`, `TResult`.
- Никогда не одна буква: `K`, `V`, `R` — нечитаемо.

Префикс `T` — convention Microsoft (как `I` для interface). Видно, что параметр generic, а не обычный тип.

### 3.9. Abstract class — без префикса `A`

В .NET Framework 1.x иногда были `AbstractList`, `BaseRepository`. Сейчас Microsoft FDG **не рекомендует** префикс:

```csharp
// ❌ Старый стиль
public abstract class AbstractRepository<T> { }
public abstract class BaseService { }

// ✅ Современный стиль
public abstract class Repository<T> { }
public abstract class ServiceBase { }   // если нужно — суффикс
```

Если очень нужно подчеркнуть «base class» — суффикс `Base`. Но обычно не нужно: `Repository<T>` без префикса понятен по контексту наследования.

> [!question]- Интервью: почему интерфейсы в C# именуются с префикса `I`?
> Историческое наследие COM (Component Object Model) и Visual Studio C++ traditions. Microsoft Framework Design Guidelines явно прописывает: `I` префикс для interface'ов. Цели: 1) визуально отличать interface от class в declarations и usages (`IRepository<T>` vs `Repository<T>` сразу различимы). 2) Convention BCL — `IList`, `IEnumerable`, `IDisposable` — все знают паттерн. 3) Disambiguation в API: `class List : IList<T>` — ясно, что один реализует другой. В Java/Python/Rust такого префикса нет, но в C# — стандарт. Отступление = когнитивная нагрузка для других разработчиков.

---

## 4. Соглашения по членам

### 4.1. Method — PascalCase, глагол или verb phrase

```csharp
public void Save() { }                  // ✅ глагол
public User GetById(int id) { }         // ✅ verb phrase
public bool IsValid() { }               // ✅ для bool — Is/Has/Can
public async Task LoadAsync() { }       // ✅ Async-suffix
```

**Boolean-методы** — префикс `Is`, `Has`, `Can`, `Should`:

```csharp
public bool IsActive() { }
public bool HasPermission(string p) { }
public bool CanDelete() { }
public bool ShouldRetry(Exception ex) { }
```

**Get/Set** — для accessor-методов:

```csharp
public User GetById(int id) { }
public void SetActive(bool isActive) { }
```

### 4.2. Property — PascalCase, существительное

```csharp
public string Name { get; set; }
public int Age { get; set; }
public DateTime CreatedAt { get; init; }
public bool IsActive { get; set; }              // boolean — Is/Has/Can
public List<Order> Orders { get; } = [];        // collection — plural
```

**Правило:** Property имя = существительное (что хранит), не глагол:

```csharp
// ❌ Глагольное имя для property
public bool GetActive { get; set; }

// ✅ Существительное / прилагательное
public bool IsActive { get; set; }
```

### 4.3. Field — категории и стиль

| Тип поля | Стиль | Пример |
|----------|-------|--------|
| **Public field** | PascalCase | `public int Count;` (редко используется!) |
| **Public const** | PascalCase | `public const int MaxRetry = 3;` |
| **Public static readonly** | PascalCase | `public static readonly DateTime Epoch = ...;` |
| **Private field** | `_camelCase` | `private int _count;` |
| **Private const** | PascalCase | `private const int MaxBuffer = 1024;` |
| **Private static readonly** | PascalCase | `private static readonly Regex PathRegex = ...;` |

```csharp
public class Service
{
    public const int MaxRetries = 3;                    // public const → Pascal
    private const int InternalBuffer = 1024;            // private const → Pascal (тоже!)
    
    public static readonly DateTime ServiceStartedAt = DateTime.UtcNow;
    private static readonly ILogger Log = ...;          // static readonly → Pascal
    
    private readonly IDatabase _database;               // private field → _camel
    private int _retryCount;
}
```

**Ключевое:** const и static readonly **всегда** PascalCase, независимо от visibility. Private fields (instance) — `_camelCase`.

### 4.4. Public field — почти никогда

В C# public fields почти никогда не используются — вместо них **properties**:

```csharp
// ❌ Public field — не C# style
public class User
{
    public string Name;
    public int Age;
}

// ✅ Properties
public class User
{
    public string Name { get; set; } = "";
    public int Age { get; set; }
}
```

Properties дают: encapsulation (можно добавить validation), interface compatibility (можно реализовать), data binding, lazy initialization. Field — только для structs с очень простой семантикой.

### 4.5. Event — PascalCase

```csharp
public event EventHandler<OrderCreatedEventArgs> OrderCreated;
public event Action<int> ItemSelected;
public event PropertyChangedEventHandler PropertyChanged;
```

**Правило:** event — существительное или past participle (что произошло):

```csharp
public event EventHandler Closing;       // ✅ in-progress
public event EventHandler Closed;        // ✅ past tense — событие случилось
public event EventHandler OrderPlaced;   // ✅
```

EventArgs — суффикс `EventArgs`:

```csharp
public class OrderCreatedEventArgs : EventArgs
{
    public Order Order { get; init; }
}
```

### 4.6. Constants — PascalCase

```csharp
public const int MaxBufferSize = 1024;
public const string DefaultGreeting = "Hello";
public const double Pi = 3.14159;
private const int InternalLimit = 100;
```

**Не SCREAMING_SNAKE_CASE** — это Java/JS стиль. В C# — PascalCase везде.

Если константа — это набор связанных значений, рассмотри `enum`:

```csharp
// ❌ Группа констант
public const int OrderStatusPending = 1;
public const int OrderStatusPaid = 2;
public const int OrderStatusShipped = 3;

// ✅ Enum
public enum OrderStatus
{
    Pending = 1,
    Paid = 2,
    Shipped = 3
}
```

### 4.7. Indexer — стандартная сигнатура

```csharp
public class MyCollection<T>
{
    public T this[int index] { get; set; }   // index naming OK
    public T this[string key] { get; set; }
}
```

Индексаторы используют параметр — обычно `index` или `key`. PascalCase здесь не нужен — это параметр, не свойство.

### 4.8. Operator overload — стандартные имена

```csharp
public static Money operator +(Money a, Money b) { ... }
public static bool operator ==(Money a, Money b) { ... }
```

Operators имеют фиксированные имена. Можно реализовать через named methods (`Add`, `Equals`) — это convention, например для languages, не поддерживающих operator overload (VB.NET).

> [!question]- Интервью: должны ли constants быть SCREAMING_SNAKE_CASE в C#?
> Нет — это **Java/JavaScript/Python** convention. Microsoft Framework Design Guidelines для C# явно: const и static readonly всегда **PascalCase**, независимо от visibility. Например, `MaxBufferSize`, `DefaultTimeout`, `Pi`. SCREAMING_SNAKE в C# смотрится чужеродно (хотя компилируется). Это одно из самых частых заблуждений у разработчиков, переходящих с Java. BCL во всех своих constants использует PascalCase: `int.MaxValue`, `Math.PI`, `TimeSpan.Zero`.

---

## 5. Параметры и локальные переменные

### 5.1. Параметры — camelCase, описательно

```csharp
public User GetById(int id) { }
public void Update(int userId, string newEmail) { }
public void Calculate(decimal amount, decimal taxRate, bool includeShipping) { }
```

**Правило:** параметры всегда `camelCase`. Имя описывает **что** передаётся, не **как**.

### 5.2. Длина имени — domain context

```csharp
// Короткие — для общеизвестных
public User GetById(int id) { }                  // ✅ id — clear
public string Format(string s) { }                // ✅ s — общеприятно для string

// Длинные — для domain-specific
public void ProcessOrder(int orderId, decimal totalAmount, string customerEmail) { }
```

Контекстно: `i` приемлем для loop counter (`for (int i = 0; ...)`), но не для bookkeeping в business logic.

### 5.3. Boolean параметры — Is/Has/Can

```csharp
public void Update(User user, bool sendNotification) { }     // ✅
public void Save(Order order, bool isFinal) { }              // ✅

// Лучше — с предыдущим описательным префиксом
public void Save(Order order, bool isPersisted = true) { }
public void Process(Item item, bool shouldValidate = true) { }
```

### 5.4. Default values — обозначь дефолты

```csharp
public async Task<List<User>> GetAsync(
    int pageSize = 20,
    int page = 1,
    bool includeInactive = false,
    CancellationToken cancellationToken = default)   // CancellationToken always last
{
    // ...
}
```

Conventions:
- **CancellationToken** — последний параметр, имя `cancellationToken` или `ct`.
- **Optional params** — после required, с разумными defaults.
- **Avoid magic numbers** — если default-значение значимое, лучше named constant.

### 5.5. params — для переменного числа аргументов

```csharp
public void Log(string message, params object[] args) { }

Log("User {0} logged in", username);
Log("Items: {0}, {1}, {2}", a, b, c);
```

### 5.6. out/ref/in параметры

```csharp
public bool TryParse(string s, out int value)        // out — обычный case
public void Modify(ref User user)                     // ref — редко
public int Compute(in LargeStruct data)               // in — read-only ref для perf
```

Naming не отличается — camelCase. Modifier (`out`/`ref`/`in`) на месте параметра, не в имени.

### 5.7. Локальные переменные — camelCase

```csharp
public void Process()
{
    var totalAmount = 0m;
    var orderCount = orders.Count;
    var firstUser = users.FirstOrDefault();
    
    for (int i = 0; i < orderCount; i++)
    {
        var order = orders[i];
        // ...
    }
}
```

**Правило:** camelCase, описательно. Длина зависит от scope — короткие для tight scope (`i` в for), длинные для function-level state.

### 5.8. var — когда уместно

```csharp
// ✅ Тип очевиден из правой части
var customer = new Customer();
var users = new List<User>();
var count = orders.Count();

// ⚠️ Спорно — тип неочевиден
var result = service.Process();   // что возвращает Process? Type unclear

// ✅ Явный тип когда полезно
List<User> users = LoadUsers();
HashSet<int> seen = new();
```

`var` — convention `prefer var when type is apparent` в стиле Microsoft. С .NET 9+ есть `var` analyzer в IDE, который подсказывает.

### 5.9. Discard `_` — для unused

```csharp
var (_, value) = GetTuple();           // не нужен первый элемент
if (int.TryParse(s, out _)) { }        // не нужен out
foreach (var _ in items) { count++; }  // counts items без обработки
```

`_` сигнализирует — намеренно игнорируется.

### 5.10. Loop variables — i / j / k для tight loops

```csharp
// ✅ Tight loop — i OK
for (int i = 0; i < n; i++) { }

// Nested — i, j, k
for (int i = 0; i < rows; i++)
    for (int j = 0; j < cols; j++)
        matrix[i, j] = i + j;

// ❌ Long loop body — стоит описательное имя
for (int orderIndex = 0; orderIndex < orders.Count; orderIndex++)
{
    // 50 строк работы с orderIndex...
}
```

> [!question]- Интервью: где использовать `var`, где явный тип?
> Microsoft style: «prefer var when type is apparent from the right side» — когда тип очевиден из присвоения. `var customer = new Customer();` — OK, тип читается. `var result = service.Process();` — спорно, тип неочевиден без знания API. Правила хорошего тона: использовать `var` для конструкторов, factories, LINQ выражений; явный тип когда type signature важен (interface vs concrete, API contract). Команды иногда фиксируют свой стиль через .editorconfig (`csharp_style_var_for_built_in_types`).

---

## 6. Namespaces и assemblies

### 6.1. Namespace — PascalCase, точки как разделители

```csharp
namespace MyCompany.MyApp.Domain.Orders;
namespace System.Collections.Generic;
namespace Microsoft.Extensions.DependencyInjection;
```

**Правило:** dot-separated PascalCase. Иерархия от общего к частному.

### 6.2. Convention `Company.Product.Component`

Microsoft FDG предписывает структуру:

```
<Company or Organization>.<Product or Project>.<Functional Area>
```

Примеры:
- `Microsoft.AspNetCore.Mvc`
- `Newtonsoft.Json.Linq`
- `System.Collections.Generic`
- `MyCompany.OrderManagement.Domain`

### 6.3. Совпадение с типами — избегай

```csharp
// ❌ Namespace и type конфликтуют
namespace System.Threading.Tasks
{
    public class Task { }   // конфликтует с System.Threading.Tasks.Task!
}

// ✅
namespace MyApp.Tasks
{
    public class TaskManager { }
}
```

Namespace и type внутри него с одинаковым именем — баг. IDE confused, errors.

### 6.4. File-scoped namespace (C# 10+)

```csharp
// Старый стиль — block-scoped
namespace MyApp.Services
{
    public class UserService
    {
        // ...
    }
}

// C# 10+ — file-scoped (без braces)
namespace MyApp.Services;

public class UserService
{
    // ...
}
```

Файл-scoped — меньше indentation. Стандарт в новом коде. Microsoft analyzer `IDE0161` предупреждает о block-scoped.

### 6.5. Assembly naming — соответствует namespace

Convention: `MyCompany.MyApp.dll` соответствует `namespace MyCompany.MyApp.*`.

```
MyApp.Domain.dll       → namespace MyApp.Domain.*
MyApp.Infrastructure.dll → namespace MyApp.Infrastructure.*
MyApp.Web.dll          → namespace MyApp.Web.*
```

### 6.6. Subnamespaces — функциональные группы

```
MyApp/
├── Domain/
│   ├── Users/
│   │   ├── User.cs           → namespace MyApp.Domain.Users
│   │   └── UserService.cs
│   └── Orders/
│       ├── Order.cs           → namespace MyApp.Domain.Orders
│       └── OrderRepository.cs
├── Infrastructure/
│   └── Persistence/           → namespace MyApp.Infrastructure.Persistence
└── Web/
    └── Controllers/           → namespace MyApp.Web.Controllers
```

Структура папок = структура namespaces. Стандарт.

### 6.7. Internal организация

```csharp
namespace MyApp.Services.Internal;   // или просто .Helpers, .Utilities
```

`Internal` или `Helpers` для namespace, не для public API. Сигнализирует — это вспомогательный код, не для consumers.

### 6.8. Avoid generic words

```csharp
// ❌ Слишком общие
namespace MyApp.Helpers;
namespace MyApp.Common;
namespace MyApp.Utilities;

// ✅ Конкретные
namespace MyApp.Validation;
namespace MyApp.Logging;
namespace MyApp.Persistence;
```

«Helpers» / «Utilities» — частая помойка. Лучше выделить понятные функциональные области.

> [!question]- Интервью: как именовать namespaces?
> PascalCase, dot-separated, иерархия от общего к частному. Convention: `<Company>.<Product>.<Component>`. Например: `Microsoft.AspNetCore.Mvc`, `MyCompany.OrderManagement.Domain.Orders`. Структура папок проекта обычно зеркалит namespaces (`Domain/Orders/Order.cs` → `MyApp.Domain.Orders`). С C# 10+ — file-scoped namespace без braces. Избегать generic слов («Helpers», «Common», «Utilities»). Не дублировать имена namespace и типов внутри.

---

## 7. Файлы и папки

### 7.1. Один тип — один файл

```
UserService.cs       → class UserService
IUserService.cs      → interface IUserService
User.cs               → class User
OrderStatus.cs       → enum OrderStatus
```

**Правило:** имя файла = имя основного типа в нём. Это не C# requirement (компилятор не enforced), но стандарт всех проектов.

### 7.2. Соответствие namespace и пути

```
src/
├── MyApp.Domain/
│   └── Users/
│       ├── User.cs                  → MyApp.Domain.Users.User
│       └── UserService.cs           → MyApp.Domain.Users.UserService
└── MyApp.Web/
    └── Controllers/
        └── UsersController.cs       → MyApp.Web.Controllers.UsersController
```

Пути совпадают с namespaces. IDE refactoring (rename file → namespace) автоматически работает.

### 7.3. Partial classes — особый case

```csharp
// User.cs
public partial class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

// User.Generated.cs (auto-generated)
public partial class User
{
    public DateTime CreatedAt { get; set; }
}
```

Partial — для разделения большого класса или для сгенерированного кода. Suffix `.Generated.cs` или `.Designer.cs` — стандарт.

### 7.4. Tests — соответствуют source

```
src/
└── MyApp.Domain/
    └── Users/
        └── UserService.cs

tests/
└── MyApp.Domain.Tests/
    └── Users/
        └── UserServiceTests.cs   ← добавлен суффикс Tests
```

Test class именуется `<TypeName>Tests`. Файл — соответственно.

### 7.5. Имена проектов

```
MyApp.Domain.csproj           ← business logic
MyApp.Domain.Tests.csproj     ← тесты домена
MyApp.Infrastructure.csproj   ← infrastructure (DB, HTTP)
MyApp.Web.csproj              ← API endpoints
MyApp.Cli.csproj              ← console app
MyApp.IntegrationTests.csproj ← интеграционные тесты
```

Conventional patterns. Каждый проект — отдельный assembly.

### 7.6. Solution структура

```
MyApp/
├── MyApp.sln
├── src/
│   ├── MyApp.Domain/
│   ├── MyApp.Infrastructure/
│   └── MyApp.Web/
├── tests/
│   ├── MyApp.Domain.Tests/
│   └── MyApp.IntegrationTests/
├── docs/
├── .editorconfig
├── Directory.Build.props
└── README.md
```

`src/` и `tests/` — стандартное разделение. `Directory.Build.props` — общие MSBuild settings для всех проектов.

### 7.7. Naming для НЕ-кодовых файлов

```
README.md              ← capital-uppercase для top-level docs
LICENSE                ← не md (исторически), без extension
CHANGELOG.md
.editorconfig          ← с точкой, без extension
.gitignore
Directory.Build.props
appsettings.json       ← lowercase (config)
appsettings.Development.json
docker-compose.yml     ← kebab-case (Docker convention)
```

Не C# convention — следуй стандарту индустрии.

---

## 8. Аббревиатуры — главный source of confusion

### 8.1. Правило двух букв

Аббревиатура **до 2 букв включительно** — все заглавные. **3+ букв** — только первая заглавная (Pascal-style):

```csharp
// 2 буквы — UPPERCASE
public class IOStream { }           // IO — 2 буквы
public Uri MyUri { get; set; }       // Uri в imports, но 3 буквы → Pascal-style
public string Id { get; set; }       // Id

// 3+ букв — Pascal-style
public class HttpClient { }          // не HTTPClient
public string Html { get; set; }     // не HTML
public XmlDocument Xml { get; set; } // 3 буквы
public class HtmlParser { }          // не HTMLParser
```

### 8.2. Известные ловушки

| Аббревиатура | Букв | Правильно (PascalCase context) | Неправильно |
|--------------|------|-------------------------------|-------------|
| **Id** | 2 | `Id`, `UserId` | `ID`, `UserID` |
| **IO** | 2 | `IOException`, `MyIOStream` | `IoException` |
| **OK** | 2 | `IsOK`, `MarkAsOK` | `IsOk` |
| **DB** | 2 | `DbConnection`, `DbContext` | `DBConnection` |
| **UI** | 2 | `IUIElement`, `UIThread` | `IUiElement` |
| **HTTP** | 4 | `HttpClient`, `HttpRequest` | `HTTPClient` |
| **HTML** | 4 | `HtmlParser`, `HtmlNode` | `HTMLParser` |
| **XML** | 3 | `XmlDocument`, `XmlSerializer` | `XMLDocument` |
| **JSON** | 4 | `JsonSerializer`, `JsonOptions` | `JSONSerializer` |
| **API** | 3 | `ApiController`, `WebApi` | `APIController` |
| **URL** | 3 | `UrlEncoder`, `BaseUrl` | `URL` |
| **URI** | 3 | `Uri` (как тип), `RequestUri` | `URI` |
| **SQL** | 3 | `SqlConnection`, `SqlCommand` | `SQLConnection` |
| **TCP** | 3 | `TcpClient`, `TcpListener` | `TCPClient` |
| **CSV** | 3 | `CsvReader`, `CsvFile` | `CSVReader` |
| **PDF** | 3 | `PdfDocument`, `PdfReader` | `PDFDocument` |

**Запоминание:** BCL имеет `Uri`, `Id`, `Db`, `Http`, `Xml`, `Json`, `Api` везде. Микрософт строго следует правилу.

### 8.3. camelCase для аббревиатур

В camelCase (параметры, локалки) — двух-буквенные **не** все заглавные, потому что начинаются с маленькой:

```csharp
// 2 буквы в camelCase
int userId;            // ✅ не userID
Uri requestUri;         // ✅
DbConnection dbConnection;   // ✅

// 3+ букв
HttpClient httpClient;
JsonOptions jsonOptions;
ApiController apiController;
```

### 8.4. Acronyms внутри имён

```csharp
public class UserDbContext { }       // 2 buckвы Db → middle of word
public class HttpClientFactory { }    // 4 буквы Http → middle
public string MyHtmlContent { get; set; }   // middle of word
```

Если аббревиатура в середине слова — следуй правилу длины. `Db` в `UserDbContext` — Pascal-style (потому что 2 буквы — 2 заглавных в начале identifier'а, но в середине — `Db` capitalized one-time).

### 8.5. Историческое: почему ID, не Id

В .NET 1.x на ранних версиях `ID` встречалось:

```csharp
// .NET Framework 1.x
public class System.Data.SqlClient.SqlCommandBuilder
{
    // ...
}
public Guid Id { get; set; }   // но Id, не ID — даже тогда
```

Microsoft явно ввёл Id (не ID) с самого начала FDG. Но многие codebase'ы (особенно legacy enterprise) сохраняют `ID`. Это legacy, не стандарт.

> [!question]- Интервью: почему `HttpClient`, а не `HTTPClient`?
> Microsoft Framework Design Guidelines: аббревиатуры **3+ букв** используют Pascal-style (только первая заглавная), **2 букв** — все заглавные. HTTP — 4 буквы → `Http`. HTML, XML, JSON, API, URL, SQL — все 3-4 буквы → Pascal style. Исключения для 2 букв: `IO`, `Id`, `Db`, `UI`, `OK` — все заглавные. Это правило — **convention BCL**. Все .NET API следуют ему: `HttpClient`, `JsonSerializer`, `XmlDocument`, `SqlConnection`. Нарушение = когнитивная нагрузка.

---

## 9. Naming patterns — стандартные паттерны

### 9.1. Async-suffix

**Правило:** все методы, возвращающие `Task` или `ValueTask`, — суффикс `Async`:

```csharp
public Task<User> GetUserAsync(int id) { }
public ValueTask<bool> ExistsAsync(int id) { }
public Task SaveAsync() { }
public IAsyncEnumerable<User> StreamUsersAsync() { }
```

Зачем:
- Caller сразу видит — это async, нужен `await`.
- Compiler не подскажет если забыл await (`fire and forget` как warning, но не error).
- Конвенция BCL — все 1000+ async API имеют suffix.

Исключение — async iterator (`IAsyncEnumerable<T>`). Там `Async` суффикс тоже стандарт, хотя метод не возвращает `Task`.

### 9.2. Try-prefix — для безопасных конверсий

```csharp
public bool TryParse(string s, out int value) { }
public bool TryGetValue(int key, out User user) { }
```

**Правило:** `Try` префикс — метод не throws, возвращает bool + out param. Безопасная альтернатива throw-варианту:

```csharp
public int Parse(string s) { /* throws FormatException */ }
public bool TryParse(string s, out int value) { /* never throws */ }

public T Get(int id) { /* throws KeyNotFoundException */ }
public bool TryGet(int id, out T item) { /* never throws */ }
```

В BCL: `int.TryParse`, `Dictionary.TryGetValue`, `Queue.TryDequeue`.

### 9.3. With-prefix / Set-prefix — для immutable mutations

```csharp
// Immutable record
public record User(string Name, int Age)
{
    public User WithName(string newName) => this with { Name = newName };
    public User WithAge(int newAge) => this with { Age = newAge };
}

// Builder pattern
public OrderBuilder WithCustomer(int id) { ... }
public OrderBuilder WithDiscount(decimal amount) { ... }
```

`With` — для immutable copy with change. `Set` — для mutable setter (но в C# обычно property).

### 9.4. To-prefix — конверсии

```csharp
public List<T> ToList() { }
public T[] ToArray() { }
public string ToString() { }
public DateTime ToUniversalTime() { }
public Dictionary<TKey, TValue> ToDictionary(...) { }
```

**Convention:** `To*` создаёт **новую** коллекцию / тип с копированием данных. Семантика «копия».

### 9.5. As-prefix — view conversion

```csharp
public Span<T> AsSpan() { }
public Memory<T> AsMemory() { }
public ReadOnlySpan<char> AsSpan(this string s) { }
public IEnumerable<T> AsEnumerable() { }
```

**Convention:** `As*` создаёт **обёртку** над тем же memory, без копирования. Семантика «view».

```csharp
string s = "hello";
ReadOnlySpan<char> span = s.AsSpan();   // O(1), no copy
char[] arr = s.ToCharArray();            // O(n), копия
```

`As*` дешевле `To*`, но обёртка живёт пока живёт оригинал.

### 9.6. Get-prefix — accessor с возможной computation

```csharp
public User GetById(int id) { }            // может бросить
public User? FindById(int id) { }           // может вернуть null
public bool TryGetById(int id, out User u) { }   // не throws
```

**Правило:** `Get*` для accessor методов. Может throws (NotFound). Альтернативы — `Find*` (returns nullable), `Try*` (returns bool).

### 9.7. Create-prefix — factory methods

```csharp
public static Logger CreateLogger(string category) { }
public static HttpClient CreateClient() { }
public static User CreateAnonymous() { }
```

`Create*` — factory pattern. Метод создаёт новый instance.

В Microsoft FDG:
- `Create` — factory.
- `New` — иногда тоже, но `Create` доминирует.

### 9.8. EventArgs-suffix

```csharp
public class OrderCreatedEventArgs : EventArgs
{
    public Order Order { get; init; }
    public DateTime CreatedAt { get; init; }
}

public event EventHandler<OrderCreatedEventArgs> OrderCreated;
```

Все event arg classes — суффикс `EventArgs`. Convention .NET, все BCL events следуют.

### 9.9. Exception-suffix

```csharp
public class NotFoundException : Exception { }
public class ValidationException : Exception { }
public class PaymentDeclinedException : Exception { }
public class TimeoutException : Exception { }   // BCL
```

Все exception types — суффикс `Exception`. Convention .NET. Без него непонятно, что это исключение.

### 9.10. Attribute-suffix

```csharp
public class RequiredAttribute : Attribute { }
public class RouteAttribute : Attribute { }
public class JsonPropertyAttribute : Attribute { }
```

В коде используется без суффикса (compiler допускает):

```csharp
[Required]              // компилятор ищет [RequiredAttribute]
public string Name { get; set; }

[Route("api/users")]
public class UsersController { }
```

Convention: класс с суффиксом, использование без. Так все знают, что это атрибут.

### 9.11. Options / Settings / Configuration

```csharp
public class JwtOptions
{
    public string SecretKey { get; set; } = "";
    public int ExpirationMinutes { get; set; } = 60;
}

services.Configure<JwtOptions>(config.GetSection("Jwt"));
```

`Options` — для DI options pattern (`IOptions<T>`). `Settings` или `Configuration` — менее формальные, тот же концепт.

### 9.12. ViewModel / Request / Response / Dto

```csharp
public class UserDto                           // data transfer
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

public class CreateOrderRequest                // API input
{
    public int CustomerId { get; set; }
    public List<int> ItemIds { get; set; } = [];
}

public class OrderResponse                     // API output
{
    public int Id { get; set; }
    public OrderStatus Status { get; set; }
}

public class OrderViewModel                    // UI binding
{
    public int Id { get; set; }
    public string DisplayName { get; set; } = "";
}
```

Соглашения по слою:
- `Dto` — generic data transfer (между слоями).
- `Request` / `Response` — API endpoint contracts.
- `ViewModel` — UI binding (MVVM, ASP.NET MVC).
- `Command` / `Query` — CQRS.

### 9.13. -er / -or — agent наме

```csharp
public class PriceCalculator { }      // calculates prices
public class OrderProcessor { }        // processes orders
public class FileWriter { }             // writes files
public class EventDispatcher { }        // dispatches events
```

`-er` или `-or` суффикс — класс который **делает** что-то. Анти-паттерн «manager / utility», когда нет конкретного действия.

### 9.14. -able / -ible — capability

```csharp
public interface IDisposable { }
public interface IComparable<T> { }
public interface IEnumerable<T> { }
public interface ISerializable { }
```

`-able` суффикс для interface — capability marker. «Этот тип обладает свойством X».

### 9.15. -ed / -ing — event semantics

```csharp
public event EventHandler Closing;       // ✅ in-progress (можно отменить)
public event EventHandler Closed;        // ✅ past tense (уже произошло)

public event EventHandler OrderCreated;  // past tense
public event EventHandler OrderCreating; // pre-creation
```

`-ing` — событие в процессе (часто cancellable). `-ed` — событие после завершения.

> [!question]- Интервью: что значит Async-суффикс в C#?
> Convention для async методов: метод возвращает `Task`, `Task<T>`, `ValueTask` или `ValueTask<T>` — добавляется суффикс `Async`. Например: `GetUserAsync`, `LoadDataAsync`, `SaveAsync`. Зачем: caller визуально понимает, что нужен `await`, и compiler warning при «fire and forget» (вызов без await). Это **convention BCL** — все async API в .NET 4.5+ следуют. Исключение: async iterators (`IAsyncEnumerable<T>`) — суффикс `Async` тоже стандарт, хотя метод не возвращает Task. В Java эквивалент — `CompletionStage<T>` без специального суффикса.

---

## 10. Hungarian notation — anti-pattern

### 10.1. Что это

Hungarian notation — стиль, где **префикс кодирует тип** переменной:

```csharp
// VBScript / pre-.NET MFC style
string strName;       // str = string
int intCount;          // int = integer
bool bIsActive;        // b = boolean
double dblPrice;       // dbl = double
char[] arrChars;       // arr = array
```

Это правило **давно мертво** в C#. Microsoft ввёл FDG в 2002 явно отказавшись от Hungarian.

### 10.2. Почему было

В 1980-х (Microsoft Word, Office, Excel core teams) C / C++ не имели IDE с типовой подсказкой. Префикс помогал увидеть тип, не идя смотреть declaration. Charles Simonyi (Hungarian Microsoft) предложил convention — отсюда название.

### 10.3. Почему умерло

1. **IDE** — Visual Studio показывает тип через hover, IntelliSense. Префикс лишний.
2. **Компилятор enforced** — переменная имеет один тип всю жизнь. Нет «string или int?».
3. **Шум** — `strName` хуже читается чем `name`.
4. **Refactoring** — изменил тип? Надо переименовать всю переменную.
5. **Неправильно** — Apps Hungarian (исходный) был о semantic intent (`xpos`, `ypos`), не о типах. Microsoft исказил его в Systems Hungarian (типы).

### 10.4. Что осталось

Только два «псевдо-Hungarian» в C# — convention:
- **`I` префикс** для interface (`IRepository`, `IDisposable`).
- **`T` префикс** для generic-параметра (`TKey`, `TEntity`).

Оба — про **role**, не про тип. И оба — established convention BCL.

### 10.5. Anti-pattern в современном коде

```csharp
// ❌ Hungarian — не C# style
public class UserService
{
    private string strConnectionString;
    private int intMaxRetries;
    private bool bIsConnected;
    private List<User> lstUsers;

    public bool blnSaveUser(User objUser)
    {
        string strQuery = "...";
        // ...
    }
}

// ✅ C# style
public class UserService
{
    private readonly string _connectionString;
    private readonly int _maxRetries;
    private bool _isConnected;
    private List<User> _users;

    public bool SaveUser(User user)
    {
        string query = "...";
        // ...
    }
}
```

### 10.6. Прямой запрет в FDG

Microsoft Framework Design Guidelines прямо: **«Do not use Hungarian notation in identifiers»**. Это прямое предписание, не рекомендация.

### 10.7. Когда я вижу Hungarian — что делать

Если работаешь с legacy codebase где Hungarian — постепенно refactoring:
1. Переименуй один класс / модуль за раз.
2. Используй IDE Rename (F2 в Visual Studio).
3. Не делай sweeping change всего проекта сразу — ломает blame, code review.
4. В `.editorconfig` можно настроить warning на Hungarian-стиль.

> [!question]- Интервью: что такое Hungarian notation и почему её не используют в C#?
> Hungarian notation — стиль, где префикс имени кодирует тип (`strName`, `intCount`, `bIsActive`). Использовалась в C / C++ / pre-.NET MFC. **В C# не применяется**: 1) IDE показывает тип через hover/IntelliSense — префикс лишний шум. 2) Static typing уже даёт type info. 3) Refactoring сложнее (изменил тип → переименовать). 4) Microsoft FDG явно: «Do not use Hungarian notation». Единственные «псевдо-Hungarian» в C# — `I` префикс для interface и `T` для generic — но они про **role**, не про тип, и established convention BCL.

---

## 11. Domain-specific naming

### 11.1. DDD — Domain layer

В Domain-Driven Design:

```csharp
// Aggregate root
public class Order { ... }

// Entity (с identity)
public class OrderLine { public int Id; ... }

// Value Object (immutable, equal by value)
public record Money(decimal Amount, string Currency);
public record Address(string Street, string City);

// Domain Event
public record OrderCreatedEvent(int OrderId, DateTime OccurredAt);

// Domain Service
public class OrderPriceCalculationService { ... }

// Specification
public class ActiveCustomersSpecification : ISpecification<Customer> { ... }
```

Naming передаёт role в архитектуре.

### 11.2. CQRS

```csharp
// Command — изменяет state
public record CreateOrderCommand(int CustomerId, List<int> ItemIds);

// Command Handler
public class CreateOrderCommandHandler : ICommandHandler<CreateOrderCommand>
{
    public Task<Result> Handle(CreateOrderCommand cmd, CancellationToken ct) { }
}

// Query — читает state
public record GetOrderByIdQuery(int OrderId);

// Query Handler
public class GetOrderByIdQueryHandler : IQueryHandler<GetOrderByIdQuery, OrderDto>
{
    public Task<OrderDto> Handle(GetOrderByIdQuery query, CancellationToken ct) { }
}
```

`Command` / `Query` суффиксы. `Handler` для обработчиков.

### 11.3. Repository pattern

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id, CancellationToken ct = default);
    Task<List<Order>> FindByStatusAsync(OrderStatus status, CancellationToken ct = default);
    Task AddAsync(Order order, CancellationToken ct = default);
    Task UpdateAsync(Order order, CancellationToken ct = default);
    Task RemoveAsync(int id, CancellationToken ct = default);
}
```

Standard CRUD method naming:
- `GetById*` — strict get (throws / returns nullable).
- `Find*` — search by criteria.
- `Add*` — create new.
- `Update*` — modify existing.
- `Remove*` / `Delete*` — delete.

### 11.4. Service layer

```csharp
public class OrderService                  // application service
{
    public async Task<OrderDto> PlaceOrderAsync(...) { }
    public async Task CancelOrderAsync(int id) { }
}

public class IEmailService                 // infrastructure service
{
    public async Task SendAsync(string to, string subject, string body) { }
}

public class IPaymentGateway              // external integration
{
    public async Task<PaymentResult> ProcessAsync(Payment p) { }
}
```

`Service` — application слой. `Gateway` — adapter к external system. `Provider` — источник данных / configuration.

### 11.5. ASP.NET Core conventions

```csharp
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<ActionResult<UserDto>> Get(int id) { }

    [HttpPost]
    public async Task<ActionResult<UserDto>> Create(CreateUserRequest request) { }

    [HttpPut("{id}")]
    public async Task<ActionResult> Update(int id, UpdateUserRequest request) { }

    [HttpDelete("{id}")]
    public async Task<ActionResult> Delete(int id) { }
}
```

Convention:
- Controller — суффикс `Controller`, plural noun (`Users`, `Orders`).
- Method names — REST verbs (`Get`, `Create`, `Update`, `Delete`).
- Routes via attributes — kebab-case URLs.

### 11.6. Microservices / Bounded contexts

```
OrderService.Api/          ← API project
OrderService.Domain/        ← business logic
OrderService.Infrastructure/← persistence, external

UserService.Api/
UserService.Domain/
UserService.Infrastructure/
```

Каждый bounded context — отдельная solution или group projects. Naming с context prefix.

### 11.7. Test naming patterns

```csharp
// Pattern: MethodName_Scenario_ExpectedBehavior
public class UserServiceTests
{
    [Fact]
    public void GetById_WhenUserExists_ReturnsUser() { }

    [Fact]
    public void GetById_WhenUserNotFound_ThrowsException() { }

    [Fact]
    public async Task SaveAsync_WhenValidUser_PersistsToDatabase() { }
}

// Alternative: Should syntax
[Fact]
public void Should_ReturnUser_When_UserExists() { }

// Alternative: Given/When/Then
[Fact]
public void Given_UserDoesNotExist_When_GetById_Then_ThrowsException() { }
```

Команды выбирают один стиль. Самый распространённый — `MethodName_Scenario_ExpectedBehavior`.

> [!question]- Интервью: как именовать tests?
> Самый распространённый паттерн: `MethodName_Scenario_ExpectedBehavior`. Например: `GetById_WhenUserExists_ReturnsUser`, `SaveAsync_WhenInvalidInput_ThrowsValidationException`. Альтернативы: BDD-style `Should_X_When_Y` или Given/When/Then. Главное — единый стиль в команде. Обоснование: при failing test мгновенно ясно «какой метод, какой сценарий, что должно было произойти». Test class — `<TypeName>Tests`. Файл соответственно.

---

## 12. EditorConfig + StyleCop + Roslyn analyzers

### 12.1. .editorconfig — единый стиль команды

`.editorconfig` — стандарт VS / Rider / VS Code, описывающий code style для всего проекта:

```ini
# Корень проекта (top-most)
root = true

# Все файлы
[*]
indent_style = space
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
charset = utf-8

# C# files
[*.cs]
indent_size = 4
csharp_using_directive_placement = outside_namespace
csharp_prefer_braces = true:warning

# Naming rules
dotnet_naming_rule.private_fields_with_underscore.severity = warning
dotnet_naming_rule.private_fields_with_underscore.symbols = private_fields
dotnet_naming_rule.private_fields_with_underscore.style = underscore_camel_case

dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private

dotnet_naming_style.underscore_camel_case.required_prefix = _
dotnet_naming_style.underscore_camel_case.capitalization = camel_case

# Var preferences
csharp_style_var_for_built_in_types = false:warning
csharp_style_var_when_type_is_apparent = true:warning
csharp_style_var_elsewhere = true:warning

# Project files
[*.{csproj,xml,config}]
indent_size = 2
```

Размещается в корне проекта (или solution). IDE автоматически применяет при работе с файлами.

### 12.2. StyleCop.Analyzers — NuGet package

Установка:

```bash
dotnet add package StyleCop.Analyzers
```

Roslyn analyzer, проверяет StyleCop правила:
- SA1200: Using directives placement.
- SA1208: System using directives must come first.
- SA1300: Element should begin with uppercase letter.
- SA1303: Const field names should begin with uppercase letter.
- SA1310: Field names should not contain underscore (можно отключить).
- SA1500: Braces for multi-line statements.

Конфигурация через `stylecop.json`:

```json
{
  "$schema": "https://raw.githubusercontent.com/DotNetAnalyzers/StyleCopAnalyzers/master/StyleCop.Analyzers/StyleCop.Analyzers/Settings/stylecop.schema.json",
  "settings": {
    "documentationRules": {
      "companyName": "MyCompany",
      "copyrightText": "Copyright (c) MyCompany. All rights reserved."
    },
    "namingRules": {
      "allowedHungarianPrefixes": [],
      "allowCommonHungarianPrefixes": false
    }
  }
}
```

### 12.3. Roslyn IDE analyzers

Microsoft предоставляет встроенные analyzers через VS / SDK:

- **IDE0001**: Simplify name.
- **IDE0007**: Use `var`.
- **IDE0090**: Use `new()` (target-typed new).
- **IDE0161**: Use file-scoped namespace.
- **IDE0044**: Make field readonly.
- **IDE0040**: Add accessibility modifiers.

Включаются через `.editorconfig`:

```ini
[*.cs]
dotnet_diagnostic.IDE0007.severity = warning
dotnet_diagnostic.IDE0161.severity = error
dotnet_diagnostic.IDE0044.severity = warning
```

### 12.4. .editorconfig + naming rules

Полная конфигурация naming rules:

```ini
# Private fields → _camelCase
dotnet_naming_rule.private_fields.severity = warning
dotnet_naming_rule.private_fields.symbols = private_field_symbol
dotnet_naming_rule.private_fields.style = underscore_camel_case_style

dotnet_naming_symbols.private_field_symbol.applicable_kinds = field
dotnet_naming_symbols.private_field_symbol.applicable_accessibilities = private

dotnet_naming_style.underscore_camel_case_style.required_prefix = _
dotnet_naming_style.underscore_camel_case_style.capitalization = camel_case

# Constants → PascalCase
dotnet_naming_rule.constants.severity = warning
dotnet_naming_rule.constants.symbols = constant_symbol
dotnet_naming_rule.constants.style = pascal_case_style

dotnet_naming_symbols.constant_symbol.applicable_kinds = field
dotnet_naming_symbols.constant_symbol.required_modifiers = const

dotnet_naming_style.pascal_case_style.capitalization = pascal_case

# Interfaces → I prefix
dotnet_naming_rule.interfaces.severity = warning
dotnet_naming_rule.interfaces.symbols = interface_symbol
dotnet_naming_rule.interfaces.style = i_prefix_pascal_case_style

dotnet_naming_symbols.interface_symbol.applicable_kinds = interface

dotnet_naming_style.i_prefix_pascal_case_style.required_prefix = I
dotnet_naming_style.i_prefix_pascal_case_style.capitalization = pascal_case
```

Это standard pattern для команд, которые хотят strict enforcement.

### 12.5. SonarQube / SonarLint

Дополнительные code-quality правила сверх StyleCop:
- Code smells (длинные методы, deep nesting).
- Bug detection (null reference paths).
- Security (SQL injection, secrets in code).
- Coverage tracking.

Платные / on-prem versions для enterprise. SonarLint — бесплатный IDE plugin для warnings.

### 12.6. Workflow в команде

```
1. .editorconfig в корне solution — version-controlled
2. CI build with treat warnings as errors
3. Pre-commit hook (или PR check) проверяет style
4. IDE auto-format on save
5. Code review только по логике, не по стилю
```

С таким setup команда не спорит о стиле — всё автоматически.

### 12.7. Treat warnings as errors в CI

```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <WarningsNotAsErrors>CS1591</WarningsNotAsErrors>  <!-- missing XML doc — warning -->
  </PropertyGroup>
</Project>
```

В CI build падает при нарушении naming rules. Локально IDE показывает warnings в реальном времени.

> [!question]- Интервью: как enforced naming conventions в команде?
> Через `.editorconfig` (наиболее распространённый способ): описаны naming rules, IDE (VS, Rider, VS Code) auto-применяет. Дополнительно — StyleCop.Analyzers (NuGet package) для расширенных правил. Roslyn IDE analyzers (IDE0XXX) встроены. В CI — `TreatWarningsAsErrors=true` ловит violations при build. Pre-commit hooks (через Husky.NET) могут проверять перед commit. Результат: команда не спорит о стиле, всё автоматически. Code review фокусируется на логике, не на пробелах.

---

## 13. Bad smells — что переименовать

### 13.1. Single-letter names в широком scope

```csharp
// ❌ Function-level: a, b, c — ничего не значат
public void Process(int a, int b)
{
    var c = a + b;
    Save(c);
}

// ✅
public void ProcessOrder(int orderId, int userId)
{
    var totalAmount = orderId + userId;
    Save(totalAmount);
}
```

OK для tight loops (`i`, `j`), не OK для function parameters / fields.

### 13.2. Generic «Manager» / «Helper» / «Utility»

```csharp
// ❌ Что именно делает Manager?
public class UserManager { ... }
public class OrderHelper { ... }
public class StringUtility { ... }

// ✅ Конкретная роль
public class UserService { ... }
public class OrderValidator { ... }
public class StringFormatter { ... }
```

`Manager` / `Helper` — часто признак unfocused класса. Если делает много — split. Если делает одно — переименуй описательно.

### 13.3. Длинные имена с redundant info

```csharp
// ❌ Лишнее
public class UserClass { ... }                  // Class в имени класса
public string UserNameString { get; set; }       // String в имени string property
public int UserAgeInteger { get; set; }          // Integer в int property
public List<User> UserList { get; set; }         // List в имени List<T>

// ✅
public class User { ... }
public string Name { get; set; }
public int Age { get; set; }
public List<User> Users { get; set; }
```

Тип уже виден из declaration. Не дублируй.

### 13.4. Misleading names

```csharp
// ❌ Имя обманывает
public List<User> GetUsers() { return GetUserById(42); }   // возвращает один, имя plural
public bool IsValid() { ... mutates state ... }              // имя suggest pure check
public void UpdateOrder(Order o) { ... возвращает result ... }   // void, но что-то возвращает (через ref/out)

// ✅
public User GetUserById(int id) { }
public bool TryValidate() { ... }
public Order Update(Order o) { ... returns updated }
```

Имя должно точно отражать поведение.

### 13.5. Boolean naming — двусмысленность

```csharp
// ❌ Что значит true?
public bool flag;
public bool data;
public bool deleted;   // is deleted? was deleted? should delete?

// ✅
public bool isDeleted;
public bool hasUnsavedChanges;
public bool canEdit;
public bool wasProcessed;
```

Boolean имена начинаются с `is`/`has`/`can`/`should`/`was` — однозначно.

### 13.6. Imperative names для свойств

```csharp
// ❌ Глагольное имя — звучит как action
public bool GetActive { get; set; }
public string GetName { get; set; }

// ✅
public bool IsActive { get; set; }
public string Name { get; set; }
```

Property — это **state**, не action. Глаголы для методов.

### 13.7. Inconsistent stylе

```csharp
// ❌ Mixed style в одном классе
public class UserService
{
    public string user_name { get; set; }       // snake_case
    public int Age { get; set; }                  // PascalCase
    public string EMAIL { get; set; }             // SCREAMING_CASE
    public bool isActive { get; set; }            // camelCase

    public void DoSomething() { }                 // PascalCase
    public void do_other_thing() { }              // snake_case
}

// ✅ Consistent
public class UserService
{
    public string UserName { get; set; }
    public int Age { get; set; }
    public string Email { get; set; }
    public bool IsActive { get; set; }

    public void DoSomething() { }
    public void DoOtherThing() { }
}
```

### 13.8. Cryptic abbreviations

```csharp
// ❌ Что значат?
public int usrCnt;
public string emailAddr;
public DateTime crtDt;

// ✅ Спеллинг полностью
public int userCount;
public string emailAddress;
public DateTime createdAt;
```

Не экономь буквы. Современные IDE имеют autocomplete — длина не проблема.

### 13.9. Numbers в названиях без causа

```csharp
// ❌ Magic numbers в имени
public class UserService2 { ... }
public class OrderManager3 { ... }
public void Process2() { ... }

// ✅ Версия через namespace или явно
namespace MyApp.V2;
public class UserService { ... }
```

Версионирование — через namespaces или suffix `V2`. Цифры в classNames без смысла.

### 13.10. Anti-pattern: суффикс типа в variable

```csharp
// ❌ Hungarian-stale
public List<int> intList = new();
public string nameString = "";
public Dictionary<int, User> userDict = new();

// ✅
public List<int> ids = new();
public string name = "";
public Dictionary<int, User> usersById = new();
```

Тип очевиден из declaration / IntelliSense. Не дублируй в имени.

> [!question]- Интервью: топ-5 bad smells в naming.
> 1) **Single-letter names** в широком scope (function parameters, fields). OK для tight loops `i,j`. 2) **Generic `Manager`/`Helper`/`Utility`** — часто unfocused класс. Замени на конкретную role (`Validator`, `Formatter`). 3) **Redundant type info** в имени (`UserClass`, `nameString`, `userList`). Тип очевиден. 4) **Misleading names** — `GetUsers()` returning single user, `IsValid()` мутирующий state. Имя должно точно описывать поведение. 5) **Inconsistent style** — mix camelCase/snake_case/SCREAMING в одном файле. Один стиль везде.

---

## 14. Refactoring — как безопасно переименовывать

### 14.1. IDE Rename (F2)

Все mainstream IDE поддерживают safe rename:
- **Visual Studio:** F2 на identifier → введи новое имя → enter. Обновит все references.
- **Rider / IntelliJ:** Shift+F6.
- **VS Code:** F2.

IDE сначала сканирует все use sites, проверяет conflicts, и атомарно переименовывает. Если есть string references (reflection, configuration files) — IDE может предупредить или не найти их.

### 14.2. Стратегия для legacy refactoring

```
Шаг 1: Один файл за раз
- Не переименовывай весь проект сразу — ломает blame, code review.
- Берёшь один класс, прогоняешь через rename.

Шаг 2: Tests первыми
- Сначала исправь tests (если они есть).
- Tests дают confidence что rename ничего не сломал.

Шаг 3: Коммит атомарно
- Один rename = один commit.
- Сообщение: "rename: UserManager -> UserService".

Шаг 4: Не меняй логику в rename-commit
- Rename-only PR — легко review.
- Логические изменения — отдельным PR.
```

### 14.3. Переход с PascalCase Hungarian к C# style

```csharp
// Before
public class user_manager
{
    private int m_userCount;
    public void GetUserData() { }
}

// Rename steps:
// 1. user_manager → UserManager (IDE rename)
// 2. m_userCount → _userCount (rename field, then move PascalCase)
// 3. UserManager → UserService (если manager bad smell)
```

Не ломай blame в одном rename — commit по шагу.

### 14.4. Хорошие практики

- **Не rename в hot fix branch** — оставь для feature work. Rename меняет много файлов, conflicts при merge.
- **Document почему** — commit message объясняет (если переименовал нечеткое в чёткое).
- **Search beyond IDE** — для public API проверь external usages (NuGet consumers, configuration, scripts).
- **Backwards compat** — при public API можно оставить `[Obsolete]` старое имя:
  ```csharp
  [Obsolete("Use NewName instead", false)]
  public void OldName() => NewName();
  
  public void NewName() { ... }
  ```

### 14.5. Что НЕЛЬЗЯ rename легко

- **Public API in shipped library** — clients депендят на старое имя. Rename = breaking change.
- **Strings in code** — `"className"` через reflection. IDE не найдёт.
- **Configuration keys** — `appsettings.json` keys, environment variables.
- **DB column names** — не renamable через C# IDE.
- **Test names** — могут быть в test plan / external referrals.

В каждом case оцени impact перед rename.

### 14.6. Naming refactoring как continuous

В большом codebase нельзя достичь идеального naming за один sprint. Это **continuous activity**:
- Каждое касание (новый feature, fix bug) — refactoring наблюдаемых имён.
- Boy Scout Rule — leave it cleaner than you found it.
- Со временем — codebase сходится к единому стилю.

> [!question]- Интервью: как безопасно переименовать класс в большом проекте?
> 1) Используй IDE rename (F2 в VS, Shift+F6 в Rider) — проверяет все references, conflicts. 2) Сначала запусти tests — убедись baseline зелёный. 3) Делай rename **атомарным** commit с сообщением `rename: OldName -> NewName`. Не смешивай с логическими изменениями — review будет огромный. 4) Для public API в shipped library — `[Obsolete]` алиас на старое имя для backward compat. 5) Проверь string references (reflection, configuration, scripts) вручную. 6) Не делай sweeping rename всего проекта в одном commit — ломает blame, hard to review.

---

## 15. Best Practices

### 15.1. Базовые правила

- ✅ **PascalCase** для типов и публичных членов (классы, методы, свойства, events, constants).
- ✅ **camelCase** для локальных переменных и параметров.
- ✅ **`_camelCase`** для приватных полей.
- ✅ **`I` префикс** для interface (`IRepository`).
- ✅ **`T` префикс** для generic-параметров (`TKey`, `TValue`).
- ✅ **`Async` суффикс** для async-методов.
- ✅ **`Exception` суффикс** для exception типов.
- ❌ **Hungarian notation** — никогда (`strName`, `intCount`).
- ❌ **SCREAMING_SNAKE_CASE** для constants — это Java/JS стиль.
- ❌ **Public fields** — используй properties.

### 15.2. Аббревиатуры

- ✅ **2 буквы — все заглавные** (`Id`, `IO`, `Db`, `OK`).
- ✅ **3+ букв — Pascal-style** (`Http`, `Xml`, `Json`, `Api`, `Url`).
- ✅ **Следуй BCL** — `HttpClient`, `JsonSerializer`, `XmlDocument`.
- ❌ Не путай `ID` и `Id` — Microsoft FDG предписывает `Id`.

### 15.3. Naming patterns

- ✅ **`Try*`** — методы которые не throw, returns bool + out.
- ✅ **`To*`** — копирующая конверсия (`ToList`, `ToArray`).
- ✅ **`As*`** — view конверсия (`AsSpan`, `AsMemory`).
- ✅ **`Get*`** — accessor (может throw).
- ✅ **`Find*`** — search (может вернуть null).
- ✅ **`Create*`** — factory.
- ✅ **`-er` / `-or`** — agent (`Calculator`, `Processor`).
- ✅ **`-able` / `-ible`** — capability (`Disposable`, `Comparable`).

### 15.4. Domain naming

- ✅ **Service** — application logic.
- ✅ **Repository** — data access.
- ✅ **Controller** — web endpoints.
- ✅ **Builder** — builder pattern.
- ✅ **Factory** — creation.
- ✅ **Handler** — event/command handling.
- ✅ **Provider** — data source.
- ❌ **`Manager`/`Helper`/`Utility`** — часто bad smell, переименуй в конкретную role.

### 15.5. Tooling

- ✅ **`.editorconfig`** — version-controlled, IDE auto-применяет.
- ✅ **StyleCop.Analyzers** — расширенные правила.
- ✅ **`TreatWarningsAsErrors`** в CI — нарушения = build fail.
- ✅ **Pre-commit hook** — Husky.NET для проверки до commit.
- ✅ **Auto-format on save** — IDE setting.

### 15.6. Не делай

- ❌ Single-letter names в широком scope.
- ❌ `Manager`, `Helper`, `Utility` без конкретной роли.
- ❌ Redundant type info в имени (`UserClass`, `nameString`).
- ❌ Misleading names (plural для single, void возвращающий через ref).
- ❌ Boolean без `Is`/`Has`/`Can`/`Should` префикса.
- ❌ Глаголы для properties.
- ❌ Mixed style в одном файле.
- ❌ Cryptic abbreviations (`usrCnt`, `crtDt`).
- ❌ Numbers в classNames без причины (`UserService2`).
- ❌ Hungarian-style суффиксы типа в variable (`intList`, `userDict`).

### 15.7. Делай

- ✅ Описательные имена — длина OK, IDE help-it.
- ✅ Boolean с `Is`/`Has`/`Can`/`Should`.
- ✅ Глаголы для методов, существительные для properties и types.
- ✅ Plural для collections (`Users`, `Orders`).
- ✅ Suffix conventions (`Service`, `Repository`, `Exception`, `EventArgs`).
- ✅ Async-suffix для Task-возвращающих.
- ✅ Symmetry в API (`Add`/`Remove`, `Open`/`Close`, `Begin`/`End`).
- ✅ Domain language — терминология бизнеса в коде.
- ✅ Readability over brevity — современные IDE handle любую длину.

---

## 16. Decision tree

```
Что нужно именовать?
│
├── Тип (class, struct, interface, enum, delegate, record)
│   ├── PascalCase
│   ├── interface — добавь `I` префикс
│   ├── exception — суффикс `Exception`
│   ├── attribute — суффикс `Attribute`
│   ├── EventArgs — суффикс `EventArgs`
│   └── enum — единственное число (`OrderStatus`, не `OrderStatuses`)
│
├── Метод
│   ├── PascalCase
│   ├── глагол или verb phrase (`Save`, `GetById`)
│   ├── boolean → `Is`/`Has`/`Can`/`Should`
│   ├── async → суффикс `Async`
│   ├── try-pattern → `Try*` + out параметр
│   └── factory → `Create*`
│
├── Property
│   ├── PascalCase
│   ├── существительное / прилагательное (`Name`, `IsActive`)
│   └── collection → plural (`Users`, `Orders`)
│
├── Field
│   ├── public const / static readonly → PascalCase (`MaxRetries`)
│   ├── private const / static readonly → PascalCase (`InternalBuffer`)
│   └── private instance → `_camelCase` (`_database`)
│
├── Parameter / local
│   ├── camelCase
│   └── CancellationToken — последний параметр, `cancellationToken` или `ct`
│
├── Generic параметр
│   ├── один → `T`
│   └── несколько → `TKey`, `TValue`, `TEntity`, `TResult`
│
├── Namespace
│   ├── PascalCase, dot-separated
│   ├── `Company.Product.Component`
│   └── file-scoped (C# 10+)
│
├── Файл
│   ├── PascalCase.cs
│   └── один тип = один файл
│
└── Аббревиатура
    ├── 2 буквы → все заглавные (`Id`, `IO`, `Db`)
    └── 3+ букв → Pascal-style (`Http`, `Xml`, `Json`)
```

---

## 17. Cheat sheet

```csharp
// === Types ===
public class UserService { }
public interface IRepository<T> { }
public struct Point { }
public enum OrderStatus { Pending, Paid }
public delegate void EventHandler<T>(T args);
public record User(int Id, string Name);

// === Members ===
public class Service
{
    public const int MaxRetries = 3;             // const PascalCase
    public static readonly TimeSpan Timeout = ...;// static readonly PascalCase
    private const int InternalBuffer = 1024;      // private const тоже Pascal!
    private readonly IDb _database;                // private field _camel
    private int _count;
    
    public string Name { get; set; } = "";        // property PascalCase
    public bool IsActive { get; set; }            // boolean — Is/Has/Can/Should
    public List<Order> Orders { get; } = [];      // collection plural
    
    public event EventHandler<OrderEventArgs> OrderCreated;   // event PastTense
    
    public async Task<User> GetByIdAsync(int id)  // async + Async suffix
    {
        var user = await _database.LoadAsync(id);  // local camelCase
        return user;
    }
    
    public bool TryParse(string s, out int value) // Try-pattern
    {
        return int.TryParse(s, out value);
    }
}

// === Generic ===
public class Dictionary<TKey, TValue> { }         // T prefix
public class Repository<T> { }                     // single — T
public class Mapper<TSource, TDestination> { }    // несколько — описательно

// === Naming patterns ===
public Task<User> LoadAsync()                      // Async suffix
public List<T> ToList()                            // To = copy
public Span<T> AsSpan()                            // As = view
public bool TryGetValue(K k, out V v)              // Try = no throw
public User CreateAnonymous()                       // Create = factory
public bool HasPermission(string p)                 // Has = capability check

// === Hungarian — НЕТ ===
// ❌ string strName, int intCount, bool bIsActive, List<int> intList
// ✅ string name, int count, bool isActive, List<int> ids

// === Аббревиатуры ===
HttpClient, JsonSerializer, XmlDocument           // 3+ → Pascal
Id, IOStream, DbContext, IsOK                      // 2 → all caps в Pascal-context
userId, jsonOptions, dbConnection                  // 2 в camelCase → не all caps

// === Намespace ===
namespace MyApp.Domain.Orders;                     // file-scoped (C# 10+)
namespace MyCompany.Product.Component;             // company.product.component

// === Файлы ===
UserService.cs                                     // 1 type = 1 file
IUserService.cs
OrderStatus.cs
UserServiceTests.cs                                // tests с суффиксом Tests
```

| Что | Стиль | Пример |
|-----|-------|--------|
| Class | PascalCase | `UserService` |
| Interface | `I` + PascalCase | `IRepository<T>` |
| Struct / Enum / Record / Delegate | PascalCase | `Point`, `OrderStatus` |
| Generic param | `T` или `T<Desc>` | `T`, `TKey` |
| Method | PascalCase verb | `Save`, `GetById` |
| Property | PascalCase noun | `Name`, `IsActive` |
| Public field | PascalCase (редко) | — (используй property) |
| Const | PascalCase | `MaxRetries` |
| Static readonly | PascalCase | `DefaultTimeout` |
| Private field | `_camelCase` | `_database` |
| Local | camelCase | `currentUser` |
| Parameter | camelCase | `userId` |
| Boolean | `is`/`has`/`can` | `IsActive`, `HasPermission` |
| Async method | + `Async` suffix | `LoadAsync` |
| Exception | + `Exception` suffix | `NotFoundException` |
| EventArgs | + `EventArgs` suffix | `OrderCreatedEventArgs` |
| Attribute | + `Attribute` suffix | `RequiredAttribute` |
| Namespace | PascalCase dot | `MyApp.Domain.Orders` |

---

## 18. Common Pitfalls — частые ошибки

### 18.1. ID vs Id

```csharp
// ❌
public int UserID { get; set; }
public string OrganizationID { get; set; }

// ✅
public int UserId { get; set; }
public string OrganizationId { get; set; }
```

**Механизм:** Id — 2 буквы, в Pascal context первая заглавная, вторая маленькая. BCL имеет `Guid`, `Id` везде.

### 18.2. HTTPClient vs HttpClient

```csharp
// ❌
public class HTTPClient { }
public string HTMLContent { get; set; }

// ✅
public class HttpClient { }
public string HtmlContent { get; set; }
```

**Механизм:** 4 буквы — Pascal style. BCL `HttpClient`, `HtmlEncoder`, `JsonSerializer`.

### 18.3. SCREAMING_SNAKE для constants

```csharp
// ❌ Java style
public const int MAX_USERS = 100;
public const string DB_HOST = "localhost";

// ✅ C# style
public const int MaxUsers = 100;
public const string DbHost = "localhost";
```

**Механизм:** Microsoft FDG: const = PascalCase. SCREAMING — Java/JS, не C#.

### 18.4. Public fields вместо properties

```csharp
// ❌
public class User
{
    public int Id;
    public string Name;
}

// ✅
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}
```

**Механизм:** properties дают encapsulation, можно добавить validation, interface compatible. Public fields — anti-pattern в C#.

### 18.5. Hungarian notation

```csharp
// ❌
public class UserService
{
    private string strName;
    private int intCount;
    private bool bIsActive;
}

// ✅
public class UserService
{
    private string _name;
    private int _count;
    private bool _isActive;
}
```

**Механизм:** type очевиден из declaration. Префикс — шум, anti-pattern в C#.

### 18.6. Plural for single, single for plural

```csharp
// ❌ Misleading
public User GetUsers(int id) { }     // возвращает один user
public List<User> GetUser() { }       // возвращает collection

// ✅
public User GetUserById(int id) { }
public List<User> GetUsers() { }
```

**Механизм:** имя должно отражать количество. Single — singular, collection — plural.

### 18.7. Manager / Helper без role

```csharp
// ❌
public class UserManager { ... }       // делает что? CRUD? validation? auth?
public class StringHelper { ... }       // helps with string what?

// ✅
public class UserService { ... }       // application logic
public class UserValidator { ... }      // validation
public class StringFormatter { ... }    // formatting
```

**Механизм:** Manager / Helper — generic слова. Конкретная role — описательнее.

### 18.8. Async без suffix

```csharp
// ❌
public Task<User> GetById(int id) { }
public ValueTask<bool> Exists(int id) { }

// ✅
public Task<User> GetByIdAsync(int id) { }
public ValueTask<bool> ExistsAsync(int id) { }
```

**Механизм:** caller визуально не видит, что async. Compiler не warning о fire-and-forget.

### 18.9. Boolean без префикса

```csharp
// ❌
public bool active { get; set; }
public bool deleted { get; set; }

// ✅
public bool isActive { get; set; }   // wait, properties — PascalCase
public bool IsActive { get; set; }
public bool IsDeleted { get; set; }
```

**Механизм:** boolean имена с `Is`/`Has`/`Can`/`Should` — однозначны. Без — двусмысленно.

### 18.10. var vs explicit type

```csharp
// ❌ var когда тип неочевиден
var data = service.Process(input);   // что возвращает Process?

// ✅
ProcessResult data = service.Process(input);
// или
var customer = new Customer();   // тип очевиден из new

// ✅ var для LINQ chains
var activeUsers = users.Where(u => u.IsActive).ToList();
```

**Механизм:** `var` для apparent types, явный — когда нужно контекста. Microsoft style: «prefer var when type is apparent».

> [!question]- Интервью: топ-3 ловушки naming в C# для junior-разработчиков.
> 1) **Аббревиатуры** — `UserID` вместо `UserId` (2 буквы — first uppercase only в Pascal); `HTTPClient` вместо `HttpClient` (4 буквы — Pascal style). Microsoft FDG строгий. 2) **SCREAMING_SNAKE для constants** — это Java/JS стиль. В C# const всегда PascalCase. 3) **Hungarian notation** — `strName`, `intCount`, `lstUsers`. Anti-pattern, type очевиден из declaration. Все три — частые ошибки переходящих с других языков (Java, C++, JavaScript).

---

## 19. Practice — упражнения с разбором

### 19.1. Найди и исправь bad naming

**Задача.** Дан класс с naming issues. Перепиши.

```csharp
// До
public class user_service_class
{
    public string strFirstName;
    public string strLAST_NAME;
    public int intAge;
    public bool bIsActive;
    public List<int> intUserIDs;
    public const int MAX_RETRIES = 3;
    
    public bool blnSaveUser(string strName, int intAge)
    {
        var blnResult = ValidateUser(strName);
        if (blnResult)
        {
            return true;
        }
        return false;
    }
    
    public string GetHTML()
    {
        return "<html></html>";
    }
}

// После
public class UserService
{
    public string FirstName { get; set; } = "";
    public string LastName { get; set; } = "";
    public int Age { get; set; }
    public bool IsActive { get; set; }
    public List<int> UserIds { get; set; } = [];
    public const int MaxRetries = 3;
    
    public bool SaveUser(string name, int age)
    {
        var isValid = ValidateUser(name);
        return isValid;
    }
    
    public string GetHtml()
    {
        return "<html></html>";
    }
}
```

**Что исправили:**
- `user_service_class` → `UserService` (PascalCase, no `Class` suffix).
- `strFirstName`, `intAge` → `FirstName`, `Age` (no Hungarian, properties вместо fields).
- `strLAST_NAME` → `LastName` (PascalCase, не SCREAMING).
- `intUserIDs` → `UserIds` (no Hungarian, `Id` not `ID`).
- `MAX_RETRIES` → `MaxRetries` (PascalCase для const).
- `blnSaveUser` → `SaveUser` (no Hungarian).
- `blnResult` → `isValid` (boolean с `is` префиксом).
- `GetHTML` → `GetHtml` (4 буквы — Pascal style).

### 19.2. Naming для domain entities

**Задача.** Создать domain model для simple e-commerce, следуя naming conventions.

```csharp
// Domain types
public record Money(decimal Amount, string Currency);

public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public DateOnly DateOfBirth { get; set; }
    public bool IsActive { get; set; }
    public List<Order> Orders { get; set; } = [];
}

public class Order
{
    public int Id { get; set; }
    public int CustomerId { get; set; }
    public OrderStatus Status { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
    public List<OrderItem> Items { get; set; } = [];
    public Money TotalAmount { get; set; } = new(0, "USD");
}

public class OrderItem
{
    public int Id { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public Money UnitPrice { get; set; } = new(0, "USD");
}

public enum OrderStatus
{
    Pending = 1,
    Paid = 2,
    Shipped = 3,
    Delivered = 4,
    Cancelled = 5
}

// Application services
public interface IOrderService
{
    Task<Order> PlaceOrderAsync(PlaceOrderRequest request, CancellationToken ct = default);
    Task<Order?> GetByIdAsync(int orderId, CancellationToken ct = default);
    Task CancelAsync(int orderId, CancellationToken ct = default);
}

public class OrderService : IOrderService
{
    private readonly IOrderRepository _orderRepository;
    private readonly ICustomerRepository _customerRepository;
    private readonly ILogger<OrderService> _logger;
    
    // ...
}

// API contracts
public record PlaceOrderRequest(int CustomerId, List<OrderItemRequest> Items);
public record OrderItemRequest(int ProductId, int Quantity);
public record OrderResponse(int Id, OrderStatus Status, decimal Total, string Currency);

// Exceptions
public class OrderNotFoundException : Exception { }
public class InsufficientStockException : Exception { }

// Events
public record OrderPlacedEvent(int OrderId, int CustomerId, decimal Total, DateTimeOffset OccurredAt);
```

**Применённые conventions:**
- Все types — PascalCase, существительные.
- Properties — PascalCase, существительные.
- Boolean — `IsActive` с префиксом.
- Collections — plural (`Orders`, `Items`).
- Async methods — `Async` suffix + CancellationToken last.
- Interfaces — `I` префикс.
- Repository pattern — `IRepository` интерфейсы.
- Records — для immutable data (Money, requests, events).
- Suffixes: `Service`, `Repository`, `Request`, `Response`, `Exception`, `Event`.

### 19.3. .editorconfig для команды

**Задача.** Настроить `.editorconfig` для C# проекта с enforcement Microsoft conventions.

```ini
# Корень репозитория
root = true

# Все файлы — базовый стиль
[*]
indent_style = space
end_of_line = lf
insert_final_newline = true
trim_trailing_whitespace = true
charset = utf-8

# C# files
[*.cs]
indent_size = 4

# === Code style ===
csharp_using_directive_placement = outside_namespace:warning
csharp_prefer_braces = true:warning
csharp_style_namespace_declarations = file_scoped:warning
csharp_style_expression_bodied_methods = false:silent
csharp_style_var_for_built_in_types = false:warning
csharp_style_var_when_type_is_apparent = true:warning
csharp_style_var_elsewhere = false:warning

# === Naming rules ===
# Private fields → _camelCase
dotnet_naming_rule.private_fields.severity = error
dotnet_naming_rule.private_fields.symbols = private_field_symbol
dotnet_naming_rule.private_fields.style = underscore_camel_case_style

dotnet_naming_symbols.private_field_symbol.applicable_kinds = field
dotnet_naming_symbols.private_field_symbol.applicable_accessibilities = private
dotnet_naming_symbols.private_field_symbol.required_modifiers =

dotnet_naming_style.underscore_camel_case_style.required_prefix = _
dotnet_naming_style.underscore_camel_case_style.capitalization = camel_case

# Constants → PascalCase
dotnet_naming_rule.constants.severity = error
dotnet_naming_rule.constants.symbols = constant_symbol
dotnet_naming_rule.constants.style = pascal_case_style

dotnet_naming_symbols.constant_symbol.applicable_kinds = field
dotnet_naming_symbols.constant_symbol.required_modifiers = const

dotnet_naming_style.pascal_case_style.capitalization = pascal_case

# Interfaces → I prefix + PascalCase
dotnet_naming_rule.interfaces.severity = error
dotnet_naming_rule.interfaces.symbols = interface_symbol
dotnet_naming_rule.interfaces.style = i_prefix_style

dotnet_naming_symbols.interface_symbol.applicable_kinds = interface

dotnet_naming_style.i_prefix_style.required_prefix = I
dotnet_naming_style.i_prefix_style.capitalization = pascal_case

# Type parameters → T prefix
dotnet_naming_rule.type_parameters.severity = error
dotnet_naming_rule.type_parameters.symbols = type_parameter_symbol
dotnet_naming_rule.type_parameters.style = t_prefix_style

dotnet_naming_symbols.type_parameter_symbol.applicable_kinds = type_parameter

dotnet_naming_style.t_prefix_style.required_prefix = T
dotnet_naming_style.t_prefix_style.capitalization = pascal_case

# === Quality ===
dotnet_diagnostic.IDE0007.severity = warning   # Use var
dotnet_diagnostic.IDE0044.severity = warning   # Add readonly
dotnet_diagnostic.IDE0040.severity = warning   # Accessibility modifiers
dotnet_diagnostic.IDE0090.severity = warning   # Use new()
dotnet_diagnostic.IDE0161.severity = error     # File-scoped namespace
dotnet_diagnostic.CA1062.severity = warning    # Validate arguments

# Project files
[*.{csproj,xml,config,json}]
indent_size = 2
```

**Использование:**

```bash
# Проверить локально
dotnet format --verify-no-changes

# Auto-fix
dotnet format

# В CI
dotnet format --verify-no-changes
# Падает если не отформатировано
```

### 19.4. Refactoring legacy кода

**Задача.** Дан legacy класс. Перепиши под современные conventions, разбив на logical commits.

```csharp
// Legacy код
public class user_DAL
{
    public DataBase oDB;
    public string strConnectionString;
    
    public DataSet GetUsersDataSet()
    {
        // ...
    }
    
    public bool blnAddUser(string strName, string strEmail, int intAge)
    {
        // ...
    }
    
    public List<int> arrGetUserIDs()
    {
        // ...
    }
}
```

**Refactoring план:**

```
Commit 1: Class rename
  user_DAL → UserRepository
  - File rename, namespace update
  - IDE rename для refactor references

Commit 2: Field rename
  oDB → _database
  strConnectionString → _connectionString
  Make readonly если возможно

Commit 3: Methods rename
  GetUsersDataSet() → GetAllAsync() (+ async refactor если нужно)
  blnAddUser → AddAsync (+ async, no Hungarian, parameters renamed)
  arrGetUserIDs → GetAllIdsAsync (no `arr`, no Hungarian, Async, plural)

Commit 4: Modernization
  - Replace DataSet with List<User>
  - Add interface IUserRepository
  - Add CancellationToken to all async methods
  - Convert public fields to readonly fields with constructor injection
```

**Финал:**

```csharp
public interface IUserRepository
{
    Task<List<User>> GetAllAsync(CancellationToken ct = default);
    Task<int> AddAsync(User user, CancellationToken ct = default);
    Task<List<int>> GetAllIdsAsync(CancellationToken ct = default);
}

public class UserRepository : IUserRepository
{
    private readonly IDatabase _database;
    private readonly string _connectionString;
    
    public UserRepository(IDatabase database, string connectionString)
    {
        _database = database;
        _connectionString = connectionString;
    }
    
    public async Task<List<User>> GetAllAsync(CancellationToken ct = default)
    {
        // ...
    }
    
    public async Task<int> AddAsync(User user, CancellationToken ct = default)
    {
        // ...
    }
    
    public async Task<List<int>> GetAllIdsAsync(CancellationToken ct = default)
    {
        // ...
    }
}
```

**Разбор:** разбивка на atomic commits позволяет легко review каждый шаг и rollback при проблеме. Никогда не делай `git diff` на 200 файлов с 50 разными изменениями — sea of changes невозможно read.

### 19.5. Test naming pattern

**Задача.** Написать tests для UserService.GetByIdAsync, следуя naming convention.

```csharp
using Xunit;

public class UserServiceTests
{
    [Fact]
    public async Task GetByIdAsync_WhenUserExists_ReturnsUser()
    {
        // Arrange
        var repository = new Mock<IUserRepository>();
        var expectedUser = new User { Id = 42, Name = "Alice" };
        repository.Setup(r => r.GetByIdAsync(42, It.IsAny<CancellationToken>()))
                  .ReturnsAsync(expectedUser);
        var service = new UserService(repository.Object);
        
        // Act
        var actual = await service.GetByIdAsync(42);
        
        // Assert
        Assert.Equal(expectedUser, actual);
    }
    
    [Fact]
    public async Task GetByIdAsync_WhenUserNotFound_ThrowsNotFoundException()
    {
        // Arrange
        var repository = new Mock<IUserRepository>();
        repository.Setup(r => r.GetByIdAsync(It.IsAny<int>(), It.IsAny<CancellationToken>()))
                  .ReturnsAsync((User?)null);
        var service = new UserService(repository.Object);
        
        // Act & Assert
        await Assert.ThrowsAsync<NotFoundException>(() => service.GetByIdAsync(999));
    }
    
    [Fact]
    public async Task GetByIdAsync_WhenInvalidId_ThrowsArgumentException()
    {
        var service = new UserService(Mock.Of<IUserRepository>());
        
        await Assert.ThrowsAsync<ArgumentException>(() => service.GetByIdAsync(-1));
    }
    
    [Fact]
    public async Task GetByIdAsync_WhenCancelled_ThrowsOperationCanceledException()
    {
        var service = new UserService(Mock.Of<IUserRepository>());
        var cts = new CancellationTokenSource();
        cts.Cancel();
        
        await Assert.ThrowsAsync<OperationCanceledException>(
            () => service.GetByIdAsync(1, cts.Token));
    }
}
```

**Применённые conventions:**
- Test class — `<TypeName>Tests`.
- Test method — `MethodName_Scenario_ExpectedBehavior`.
- AAA pattern (Arrange / Act / Assert).
- `Async` suffix на методах + CancellationToken.
- Mock variables — описательные (`repository`, `expectedUser`).

---

## 20. Что читать дальше — порядок и почему

1. **[[csharp-basics|C# Basics]]** — основа языка с правильными именами.
2. **Microsoft Framework Design Guidelines** — официальный источник правил.
3. **C# Language Style Guide** — official Microsoft docs.
4. **Roslyn Analyzers** — встроенные правила в SDK.
5. **StyleCop.Analyzers** — расширенные style правила.
6. **[[debugging-basics|Debugging]]** — DebuggerDisplay для своих типов с правильными именами.
7. **EditorConfig spec** — editorconfig.org.
8. **Domain-Driven Design** — naming в больших codebases.

---

## 21. См. также

- [[csharp-basics|C# Basics]] — основы языка
- [[extension-methods|Extension Methods]] — naming для extensions
- [[anonymous-types|Anonymous Types]] — naming в LINQ projections
- [[debugging-basics|Debugging]] — DebuggerDisplay attribute
- [[error-handling|Error Handling]] — Exception suffix
- [[testing-fundamentals|Testing]] — test naming patterns
- Microsoft Framework Design Guidelines
- StyleCop.Analyzers — NuGet package
- EditorConfig — editorconfig.org

---

## 22. Reading list

- **Microsoft Docs — C# Identifier names** — learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/identifier-names
- **Microsoft Docs — Framework Design Guidelines** — learn.microsoft.com/dotnet/standard/design-guidelines/
- **Microsoft Docs — Naming Guidelines** — learn.microsoft.com/dotnet/standard/design-guidelines/naming-guidelines
- **Microsoft Docs — Capitalization Conventions** — learn.microsoft.com/dotnet/standard/design-guidelines/capitalization-conventions
- **Krzysztof Cwalina, Brad Abrams — Framework Design Guidelines (книга, 3rd edition 2020)**
- **Roy Osherove — Naming standards for unit tests** — osherove.com
- **Robert Martin — Clean Code (книга, chapter «Meaningful Names»)**
- **Steve McConnell — Code Complete (chapter «The Power of Variable Names»)**
- **Microsoft Docs — Coding conventions** — learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions
- **EditorConfig spec** — editorconfig.org
- **StyleCop.Analyzers wiki** — github.com/DotNetAnalyzers/StyleCopAnalyzers
- **Roslyn rules reference** — learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/
- **Andrew Lock — Naming patterns** — andrewlock.net
- **Eric Lippert — Identifier naming** — ericlippert.com
- **Steve Smith — DDD patterns naming** — ardalis.com
