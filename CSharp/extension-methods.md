---
tags: [csharp, extension-methods, this-parameter, linq, junior, middle]
level: Junior to Middle
date: 2026-04-30
---

# Extension Methods — методы-расширения

> **Добавь методы к существующему типу без изменения исходника**. LINQ построен на extension methods. Daily work для C# разработчика. Closes пробел "знаю что есть, но не понимаю как и когда писать свои".

---

## Что это, зачем и когда

### Что такое extension method

**Статический метод** который "выглядит" как instance method класса.

```csharp
// Без extension — utility class
public static class StringHelper
{
    public static bool IsEmpty(string s) => string.IsNullOrEmpty(s);
}

// Использование:
StringHelper.IsEmpty(name);  // ⚠️ verbose

// Extension method
public static class StringExtensions
{
    public static bool IsEmpty(this string s) => string.IsNullOrEmpty(s);
    //                          ^^^^ — magic слово
}

// Использование:
name.IsEmpty();  // ✅ выглядит как метод string!
```

**Аналогия:** Аксессуары для авто. Машина (тип) уже выпущена с завода — но ты можешь "прикрутить" свой багажник, спойлер. Extension methods — твои "аксессуары" для чужих типов.

### Зачем

| Без extension | С extension |
|--------------|-------------|
| `StringHelper.IsEmpty(name)` | `name.IsEmpty()` |
| Utility class с длинным списком | Discoverable через IntelliSense |
| Не работает на null | Работает (статический метод) |
| Нет fluent chains | `items.Where(...).Select(...)` chains |

### Где встречаются

**LINQ — весь построен на extension methods:**

```csharp
// IEnumerable<T> не имеет Where / Select / OrderBy
// Это extension methods из System.Linq.Enumerable

using System.Linq;  // важно!

var result = numbers
    .Where(n => n > 0)      // extension on IEnumerable<T>
    .Select(n => n * 2)     // extension
    .OrderBy(n => n)        // extension
    .ToList();              // extension
```

Без `using System.Linq` — этих методов нет.

---

## 1. Базовый extension

### Объявление

```csharp
// 1. Static class
// 2. Static method
// 3. First param с "this"

public static class StringExtensions
{
    public static bool IsEmpty(this string s) =>
        string.IsNullOrEmpty(s);

    public static string Truncate(this string s, int maxLength)
    {
        if (s == null) return null;
        return s.Length <= maxLength ? s : s[..maxLength];
    }
}

// Использование
"hello".IsEmpty();          // false
"hello world".Truncate(5);  // "hello"
```

### Правила

- ✅ Class **должен быть `static`**
- ✅ Method **должен быть `static`**
- ✅ Первый параметр — **`this <Type>`**
- ✅ Любые дополнительные параметры — обычные
- ❌ Не может быть **`out`/`ref`/`in`** на первом параметре (кроме `this ref` для structs)
- ❌ Не может быть **generic constraint conflict** с extended type

### Несколько параметров

```csharp
public static string Repeat(this string s, int count)
{
    return string.Concat(Enumerable.Repeat(s, count));
}

"ab".Repeat(3);  // "ababab"
```

### Generic extensions

```csharp
public static class CollectionExtensions
{
    public static T Random<T>(this IList<T> source)
    {
        var index = System.Random.Shared.Next(source.Count);
        return source[index];
    }
}

new[] { 1, 2, 3, 4, 5 }.Random();  // случайный элемент
```

---

## 2. Под капотом

### Compiler magic

```csharp
// Ты пишешь:
"hello".IsEmpty();

// Компилятор переводит в:
StringExtensions.IsEmpty("hello");
```

Просто **синтаксический сахар**. Никакой runtime магии — обычный static call.

### Поэтому работает на null

```csharp
public static bool IsEmpty(this string s) =>
    string.IsNullOrEmpty(s);

string name = null;
name.IsEmpty();  // ✅ true — НЕ NullReferenceException!
                 // Эквивалентно StringExtensions.IsEmpty(null)

// Compare с instance method:
string name = null;
name.Length;   // 💥 NullReferenceException
```

### Discovery — `using` namespace

Extension methods видны **только** если их namespace в `using`:

```csharp
// File: MyApp/StringExtensions.cs
namespace MyApp.Extensions
{
    public static class StringExtensions
    {
        public static bool IsEmpty(this string s) => ...;
    }
}

// File: Program.cs
using MyApp.Extensions;  // ⚠️ без этого extension не видна!

"hello".IsEmpty();
```

### Global usings (C# 10+)

```csharp
// File: GlobalUsings.cs
global using MyApp.Extensions;

// Теперь IsEmpty() доступна везде в проекте
```

См. [[modern-features|Modern C# Features]].

---

## 3. Resolution rules

### Instance method > extension

Если есть и instance method и extension с тем же именем — **выигрывает instance**:

```csharp
public static class StringExtensions
{
    public static int Length(this string s) => 999;  // глупо но допустим
}

"hello".Length;  // 5 (property string), не extension
```

### Один namespace > другой

Когда несколько extensions с одинаковым signature — нужен fully-qualified call:

```csharp
namespace A
{
    public static class Ext { public static string Foo(this string s) => "A"; }
}

namespace B
{
    public static class Ext { public static string Foo(this string s) => "B"; }
}

using A;
using B;

"x".Foo();  // ❌ Compile error — ambiguous!

// Solution
A.Ext.Foo("x");  // explicit
```

---

## 4. Common patterns

### Pattern 1: Validation

```csharp
public static class ValidationExtensions
{
    public static bool IsValidEmail(this string s)
    {
        if (string.IsNullOrWhiteSpace(s)) return false;
        return s.Contains('@') && s.Contains('.');
    }

    public static bool IsValidUrl(this string s) =>
        Uri.TryCreate(s, UriKind.Absolute, out _);

    public static T NotNull<T>(this T value, string paramName) where T : class =>
        value ?? throw new ArgumentNullException(paramName);
}

// Use
"john@example.com".IsValidEmail();
order.NotNull(nameof(order));
```

### Pattern 2: Conversion

```csharp
public static class DateTimeExtensions
{
    public static long ToUnixSeconds(this DateTime dt) =>
        new DateTimeOffset(dt).ToUnixTimeSeconds();

    public static DateTime StartOfDay(this DateTime dt) =>
        dt.Date;

    public static DateTime EndOfDay(this DateTime dt) =>
        dt.Date.AddDays(1).AddTicks(-1);

    public static bool IsWeekend(this DateTime dt) =>
        dt.DayOfWeek is DayOfWeek.Saturday or DayOfWeek.Sunday;
}

// Use
DateTime.Now.IsWeekend();
DateTime.Now.StartOfDay();
```

### Pattern 3: Fluent API

```csharp
public class QueryBuilder
{
    public string Sql { get; set; } = "";
}

public static class QueryBuilderExtensions
{
    public static QueryBuilder Select(this QueryBuilder qb, string columns)
    {
        qb.Sql += $"SELECT {columns} ";
        return qb;
    }

    public static QueryBuilder From(this QueryBuilder qb, string table)
    {
        qb.Sql += $"FROM {table} ";
        return qb;
    }

    public static QueryBuilder Where(this QueryBuilder qb, string condition)
    {
        qb.Sql += $"WHERE {condition}";
        return qb;
    }
}

// Use
var sql = new QueryBuilder()
    .Select("*")
    .From("users")
    .Where("age > 18")
    .Sql;
```

### Pattern 4: LINQ-like custom operators

```csharp
public static class EnumerableExtensions
{
    // Batch — split в порции
    public static IEnumerable<T[]> Batch<T>(this IEnumerable<T> source, int size)
    {
        var batch = new List<T>(size);
        foreach (var item in source)
        {
            batch.Add(item);
            if (batch.Count == size)
            {
                yield return batch.ToArray();
                batch.Clear();
            }
        }
        if (batch.Count > 0) yield return batch.ToArray();
    }

    // ForEach (которого нет в LINQ — намеренно)
    public static void ForEach<T>(this IEnumerable<T> source, Action<T> action)
    {
        foreach (var item in source) action(item);
    }

    // Distinct по property
    public static IEnumerable<T> DistinctBy<T, TKey>(
        this IEnumerable<T> source,
        Func<T, TKey> keySelector)
    {
        var seen = new HashSet<TKey>();
        foreach (var item in source)
            if (seen.Add(keySelector(item)))
                yield return item;
    }
}

// Use
items.Batch(100).ForEach(batch => ProcessBatch(batch));
people.DistinctBy(p => p.Email);
```

> [!info] DistinctBy в .NET 6+
> С .NET 6 — `DistinctBy` встроен в LINQ. Custom не нужен.

### Pattern 5: ASP.NET Core configuration

```csharp
// IServiceCollection extensions — бесспорный pattern в ASP.NET
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddMyServices(this IServiceCollection services)
    {
        services.AddScoped<IUserService, UserService>();
        services.AddScoped<IOrderService, OrderService>();
        services.AddSingleton<ICacheService, CacheService>();
        return services;
    }
}

// Use в Program.cs
builder.Services
    .AddMyServices()
    .AddDatabaseServices(builder.Configuration)
    .AddJwtAuth(builder.Configuration);
```

### Pattern 6: Result / nullable handling

```csharp
public static class ResultExtensions
{
    public static T OrThrow<T>(this T? value, string message) where T : class =>
        value ?? throw new InvalidOperationException(message);

    public static T OrDefault<T>(this T? value, T defaultValue) where T : class =>
        value ?? defaultValue;
}

// Use
var user = repo.FindById(1).OrThrow("User not found");
var config = settings.OrDefault(new Config());
```

### Pattern 7: String formatting

```csharp
public static class StringExtensions
{
    public static string ToTitleCase(this string s) =>
        CultureInfo.CurrentCulture.TextInfo.ToTitleCase(s.ToLower());

    public static string SnakeCase(this string s) =>
        string.Concat(s.Select((c, i) =>
            i > 0 && char.IsUpper(c) ? "_" + char.ToLower(c) : char.ToLower(c).ToString()));

    public static string SafeSubstring(this string s, int start, int length)
    {
        if (s == null) return null;
        if (start >= s.Length) return "";
        return s.Substring(start, Math.Min(length, s.Length - start));
    }
}

"hello world".ToTitleCase();   // "Hello World"
"MyVariableName".SnakeCase();  // "my_variable_name"
```

---

## 5. Extension на разные типы

### Extension на interface

```csharp
public static class EnumerableExtensions
{
    public static bool IsEmpty<T>(this IEnumerable<T> source) =>
        !source.Any();
}

// Работает на ВСЁМ что implements IEnumerable<T>
new[] { 1, 2 }.IsEmpty();         // false (array)
new List<int>().IsEmpty();         // true
new HashSet<string>().IsEmpty();   // true
```

### Extension на generic type

```csharp
public static class GenericExtensions
{
    public static T DeepClone<T>(this T source) where T : class
    {
        var json = JsonSerializer.Serialize(source);
        return JsonSerializer.Deserialize<T>(json);
    }
}

var copy = user.DeepClone();
```

### Extension на enum

```csharp
public enum LogLevel { Debug, Info, Warning, Error, Critical }

public static class LogLevelExtensions
{
    public static bool IsHigherThan(this LogLevel level, LogLevel threshold) =>
        (int)level > (int)threshold;

    public static string ToShortName(this LogLevel level) => level switch
    {
        LogLevel.Debug => "DBG",
        LogLevel.Info => "INF",
        LogLevel.Warning => "WRN",
        LogLevel.Error => "ERR",
        LogLevel.Critical => "CRT",
        _ => "???"
    };
}

LogLevel.Error.IsHigherThan(LogLevel.Warning);  // true
LogLevel.Info.ToShortName();  // "INF"
```

### Extension на struct (не recommended обычно)

```csharp
public static class IntExtensions
{
    public static bool IsEven(this int n) => n % 2 == 0;
    public static bool IsOdd(this int n) => n % 2 != 0;
}

5.IsOdd();   // true
10.IsEven(); // true

// Pitfall: ext на value types — boxing если signature object?
// В большинстве cases — компилятор оптимизирует, но осторожно
```

### Extension с `this ref` (.NET 4.7+)

Для structs — модифицировать через ref:

```csharp
public struct Point { public int X, Y; }

public static class PointExtensions
{
    public static void Translate(this ref Point p, int dx, int dy)
    {
        p.X += dx;
        p.Y += dy;
    }
}

var point = new Point { X = 1, Y = 1 };
point.Translate(5, 5);  // point.X = 6, point.Y = 6
```

Без `ref` — изменения не сохранятся (struct copy).

---

## 6. Best Practices

### Когда писать extension

✅ **Хорошо для:**
- Helper для типа который ты **не контролируешь** (`string`, `DateTime`, etc.)
- Fluent API / chains
- LINQ-like operators
- DRY — повторяющаяся логика на типе
- IServiceCollection / IApplicationBuilder в ASP.NET Core

❌ **НЕ для:**
- Сложного behavior (используй наследование / composition)
- Когда контролируешь тип (добавь instance method)
- "Магических" преобразований
- Глобального state

### Naming

```csharp
// ✅ Class заканчивается "Extensions"
public static class StringExtensions { }
public static class DateTimeExtensions { }
public static class IServiceCollectionExtensions { }

// ❌
public static class MyHelpers { }
public static class StringUtils { }
```

### Namespace organization

```csharp
// ✅ Extensions для определённого типа — в namespace того типа
namespace System.Linq          // Microsoft расширяет System.Collections.Generic
namespace MyApp.Validation     // твои validation extensions

// ❌ Все extensions свалены в один namespace
namespace MyApp.Helpers  // огромный список разных типов
```

### Discoverability

```csharp
// ✅ Логичная организация — пользователь IntelliSense легко находит
namespace MyApp.Extensions.Strings
namespace MyApp.Extensions.Collections
namespace MyApp.Extensions.AspNetCore
```

### Avoid surprises

```csharp
// ❌ Extension которая делает unexpected
public static class StringExtensions
{
    public static string Reverse(this string s)
    {
        // ⚠️ users могут не ожидать, что twice is no-op
        File.WriteAllText("log.txt", "reversed!");  // side effect!
        return new string(s.Reverse().ToArray());
    }
}

// ✅ Чистая логика
public static string Reverse(this string s) =>
    new string(s.AsEnumerable().Reverse().ToArray());
```

---

## 7. Common Pitfalls

### 1. Forgot `using`

```csharp
// File без `using System.Linq`
var result = items.Where(x => x > 0);  // ❌ Compile error
//                  ^^^^^ — extension не видна
```

### 2. Extension не видна с `??` operator

```csharp
public static class StringExtensions
{
    public static string ToLowerSafe(this string s) => s?.ToLower() ?? "";
}

string? name = null;
name.ToLowerSafe();   // ✅ Works (null-safe extension)
name?.ToLowerSafe();  // ⚠️ Это nullable! Возможен null

// Лучше использовать без ?. для extension methods
```

### 3. Conflict с instance method

```csharp
public static class StringExtensions
{
    public static string Trim(this string s) => s.Trim() + "!";
}

"hello".Trim();  // string.Trim() (instance) — не extension!
                  // Compiler выбирает instance первым
```

### 4. Extension на interface — может неэффективно вычислять

```csharp
public static class EnumerableExtensions
{
    public static bool IsEmpty<T>(this IEnumerable<T> source)
    {
        return source.Count() == 0;  // ⚠️ Iterates всю коллекцию!
    }
}

// ✅ Используй Any (early exit)
public static bool IsEmpty<T>(this IEnumerable<T> source) => !source.Any();

// ✅ Или специализация для известных типов
public static bool IsEmpty<T>(this IEnumerable<T> source)
{
    if (source is ICollection<T> c) return c.Count == 0;
    if (source is IReadOnlyCollection<T> rc) return rc.Count == 0;
    return !source.Any();
}
```

### 5. Generic constraint mismatch

```csharp
// ❌ Нельзя расширить с constraint которого нет на типе
public static T Method<T>(this T source) where T : IDisposable, new()
{
    // ...
}

// Зависит от type — может не работать на всех
```

### 6. Extension на null + chain

```csharp
public static class StringExtensions
{
    public static string EnsureNotEmpty(this string s) =>
        s.Length > 0 ? s : throw new ArgumentException();
        //  ^^^^^^ — NullReferenceException если s null!
}

// ✅
public static string EnsureNotEmpty(this string s)
{
    if (string.IsNullOrEmpty(s)) throw new ArgumentException();
    return s;
}
```

### 7. Static vs extension — выбор

```csharp
// ❌ Extension там где static call яснее
public static class MathExtensions
{
    public static double Pow(this double x, double y) => Math.Pow(x, y);
}

x.Pow(2);  // mathematicians привыкли к Pow(x, 2) — confusion

// ✅ Math.Pow(x, 2) — конвенция
```

### 8. Extension на mutable collections

```csharp
public static class ListExtensions
{
    public static List<T> RemoveDuplicates<T>(this List<T> list)
    {
        var distinct = list.Distinct().ToList();
        list.Clear();
        list.AddRange(distinct);
        return list;
    }
}

// ⚠️ Mutates input! Каллер может не ожидать.
// ✅ Возвращай новый
public static List<T> WithoutDuplicates<T>(this List<T> list) =>
    list.Distinct().ToList();
```

### 9. Performance — overhead

Extension method = static call. Никакой extra cost.

Но **первый параметр boxing** для value types если signature использует `object`:

```csharp
public static string ToStringSafe(this object o) => o?.ToString() ?? "";

5.ToStringSafe();  // boxing int → object! 
                    // (хотя для simple cases JIT может оптимизировать)
```

### 10. Extension на dynamic

```csharp
dynamic d = "hello";
d.IsEmpty();  // ⚠️ Runtime error — extensions не работают на dynamic
```

`dynamic` resolution — runtime, extension methods — compile-time. Не сочетаются.

---

## 8. Real-world examples

### LINQ — это всё extension methods

```csharp
namespace System.Linq
{
    public static class Enumerable
    {
        public static IEnumerable<T> Where<T>(
            this IEnumerable<T> source,
            Func<T, bool> predicate) { /* ... */ }

        public static IEnumerable<TResult> Select<TSource, TResult>(
            this IEnumerable<TSource> source,
            Func<TSource, TResult> selector) { /* ... */ }

        // ... 100+ методов
    }
}
```

### ASP.NET Core — везде extensions

```csharp
// IServiceCollection
services.AddControllers();
services.AddDbContext<AppDbContext>(...);
services.AddAuthentication();
services.AddAuthorization();

// IApplicationBuilder
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

// Все эти методы — extensions
```

### Entity Framework Core

```csharp
// IQueryable<T> extensions из Microsoft.EntityFrameworkCore
db.Users
    .Include(u => u.Orders)              // EF extension
    .ThenInclude(o => o.Items)            // EF extension
    .Where(u => u.IsActive)               // LINQ extension
    .AsNoTracking()                       // EF extension
    .OrderByDescending(u => u.CreatedAt)  // LINQ extension
    .ToListAsync();                        // EF extension
```

См. [[../EFCore/queries-performance|EF Queries]].

---

## 9. Cheat sheet

| Сценарий | Pattern |
|----------|---------|
| Helper на чужой тип | `public static T Helper(this T x, ...)` |
| Validation | `public static bool IsValid(this string s)` |
| Conversion | `public static long ToUnixSeconds(this DateTime dt)` |
| Fluent API | `return this` (или builder object) |
| LINQ-like | `IEnumerable<T>` extension с `yield return` |
| ASP.NET service registration | `IServiceCollection` extension |
| Null-safe | static method обрабатывает null first |
| Generic | `<T>` после имени метода |
| On enum | `(this LogLevel level)` |
| On struct (mutate) | `this ref` (C# 7.2+) |
| Custom LINQ operator | extends `IEnumerable<T>` |

---

## 10. Decision tree

```
Хочу добавить метод к типу?
│
├── Тип под моим контролем?
│   ├── Yes → Просто instance method
│   └── No → Extension method
│
├── Метод имеет behavior / state?
│   ├── Yes → Composition / decorator
│   └── No → Extension method подходит
│
├── Метод на interface?
│   ├── Default interface methods (C# 8+)
│   └── Extension method (старый подход)
│
└── Fluent chain?
    └── Extension method, return this
```

---

## См. также

- [[csharp-basics|C# Basics]] — methods intro
- [[collections-linq|Collections и LINQ]] — LINQ построен на extensions
- [[modern-features|Modern C# Features]] — global usings, default interface methods
- [[oop|OOP]] — inheritance vs composition vs extension
- [[generics-deep|Generics Deep]] — generic extensions
- [[../AspNetCore/di-configuration|DI & Configuration]] — IServiceCollection extensions

## Reading list

- **Microsoft Docs — Extension Methods** — learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/extension-methods
- **C# in Depth** — Jon Skeet (chapter про extensions)
- **Eric Lippert — Extension method design** — ericlippert.com (классический blog)
- **Stephen Cleary — Async extension methods** — blog.stephencleary.com
- **Andrew Lock — IServiceCollection extensions** — andrewlock.net
