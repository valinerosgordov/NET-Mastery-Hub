---
tags: [csharp, anonymous-types, linq, projections, junior]
level: Junior
date: 2026-08-02
---

# Anonymous Types — анонимные типы

> **`new { Name = "John", Age = 30 }`** — типы, рождённые компилятором на лету, без явного `class` объявления. Встречаются повсюду в LINQ-проекциях, structured logging, EF Core запросах. Закрывают пробел: «знаю, что есть, не понимаю отличий от tuples и records».

---

## 0. Как читать этот файл

Если ты впервые видишь `new { ... }` в коде — читай разделы 1→4 подряд: получишь рабочую модель и поймёшь, почему такая штука вообще нужна. Если уже пользуешься, но интересно «как оно устроено» — раздел 3 (внутреннее устройство), 9 (property order ловушка), 14 (EF Core deep). Если важна навигация по соседям — раздел 10 (vs Tuple / vs Record / vs Class).

Все примеры самостоятельные и компилируются. `// expected: ...` показывает ожидаемый вывод. Cross-language якоря (`> [!info]-`) свёрнуты — открывай, если переходишь из Python / JavaScript / TypeScript / Rust / Go. Interview-вопросы (`> [!question]-`) встроены рядом с теорией.

---

## 1. Что это, зачем и когда

### 1.1. Что такое анонимный тип

**Анонимный тип** — это `class`, который создаёт компилятор на основе outline-выражения `new { ... }`. У него нет имени, которое ты можешь написать в коде, но в IL у него есть имя (что-то вроде `<>f__AnonymousType0`) и обычная C# class-структура с свойствами, конструктором, `Equals`, `GetHashCode`, `ToString`.

```csharp
var person = new { Name = "John", Age = 30 };

// Под капотом компилятор сгенерировал примерно:
// internal sealed class <>f__AnonymousType0<TName, TAge>
// {
//     public TName Name { get; }
//     public TAge Age { get; }
//     public <>f__AnonymousType0(TName name, TAge age) { Name = name; Age = age; }
//     public override bool Equals(object o) { /* по полям */ }
//     public override int GetHashCode() { /* combined */ }
//     public override string ToString() { /* "{ Name = ..., Age = ... }" */ }
// }
```

Ключевое:

1. **Это class, не struct.** Anonymous type живёт на куче, имеет reference equality по умолчанию (но `Equals` переопределён на value).
2. **Internal sealed.** Видим только в текущей assembly, наследовать от него нельзя.
3. **Read-only properties.** Поля помечены `readonly`, свойства — без `set`. Всё immutable.
4. **Compiler reuse.** Если ты дважды напишешь `new { Name = "X", Age = 30 }` — оба выражения дадут один и тот же сгенерированный тип (с одинаковыми именами свойств в одинаковом порядке).

### 1.2. Зачем анонимные типы появились

Anonymous types появились в **C# 3.0 (2007)** — той же версии, что принесла LINQ. Это не совпадение: они решали конкретную проблему LINQ-проекций.

Вот как выглядел код **до** anonymous types:

```csharp
// Хочется получить (Id, Total) для каждого order — и больше ничего
public class OrderSummary
{
    public int Id { get; set; }
    public decimal Total { get; set; }
}

var summaries = orders
    .Select(o => new OrderSummary { Id = o.Id, Total = o.Total })
    .ToList();
```

Проблема: **на каждую LINQ-проекцию приходилось заводить класс**. Десятки одноразовых классов на проект, каждый используется в одном месте, замусоривает namespace. Компилятор это понимал: «если тип нужен ровно для одного выражения — пусть я сам его сделаю».

С C# 3.0:

```csharp
var summaries = orders
    .Select(o => new { o.Id, o.Total })
    .ToList();
// Тип элемента — anonymous, существует только внутри этого выражения
```

Никаких лишних классов. Имена свойств — компилятор берёт из имён полей объекта.

### 1.3. Главные свойства

```csharp
var p = new { Name = "John", Age = 30 };

// 1. Read-only properties
// p.Name = "Jane";   // ❌ CS0200: Property has no setter

// 2. Имена свойств обязательны
// var bad = new { "John", 30 };   // ❌ — нужны Name = / Age =

// 3. Value equality (override Equals + GetHashCode)
var p1 = new { Name = "John", Age = 30 };
var p2 = new { Name = "John", Age = 30 };
Console.WriteLine(p1.Equals(p2));   // True
Console.WriteLine(p1 == p2);        // False — оператор == не override-нут!

// 4. ToString с именами полей
Console.WriteLine(p);   // { Name = John, Age = 30 }

// 5. Только в текущем method scope (нельзя вернуть как typed)
```

Особенно тонкая деталь: `Equals` переопределён, но **оператор `==` — нет**. Anonymous type не реализует custom `==`, поэтому `==` сравнивает по reference. Это ловушка (раздел 15.8).

### 1.4. Эволюция: C# 3.0 → C# 14

| Версия | Год | Что добавили |
|--------|-----|--------------|
| **C# 3.0** | 2007 | Anonymous types, object initializers, `var`, LINQ |
| **C# 7.0** | 2017 | ValueTuple — частичная альтернатива anonymous |
| **C# 9.0** | 2020 | Records — серьёзная альтернатива для cross-method случаев |
| **C# 12** | 2023 | Collection expressions упростили inline-данные, но anonymous остался |
| **C# 13–14 / .NET 10** | 2024–2025 | Сами anonymous types не менялись — стабильная фича; альтернативы (tuples, records) тоже без изменений |

С C# 7+ часть сценариев (multi-return, локальные группировки) перешла на tuples. С C# 9+ — на records. Но **внутри LINQ-выражения** anonymous types по-прежнему канон, особенно в EF Core (см. раздел 14).

### 1.5. Когда применять

✅ **Используй когда:**
- LINQ-проекция внутри одного метода (`Select`, `GroupBy`, `Join`).
- Composite key для `GroupBy` / `OrderBy` / `HashSet` (но tuple тоже OK).
- Structured logging context (`logger.LogInformation("...", new { ... })`).
- Quick experiments в скрипте / `dotnet run --project`.

❌ **Не используй когда:**
- Нужно вернуть из метода — заводи `record`.
- Передаёшь между методами или классами — `record`.
- Сериализуешь как часть public API — `record` / DTO.
- Нужна mutability — `class`.
- Нужна inheritance / поведение — `class` / `record`.

> [!info]- Если ты знаешь Python / JavaScript / TypeScript / Rust / Go
> **Python:** ближайший аналог — `dict` (`{"name": "John", "age": 30}`) или `dataclass`. Но Python — динамический, у dict нет типа на этапе компиляции. C# anonymous — настоящий typed class, IDE видит свойства, autocomplete работает.
>
> **JavaScript:** object literal `{ name: "John", age: 30 }` — близкая идея, но нет типизации в runtime. TypeScript добавляет structural typing, который похож на C# anonymous types по духу.
>
> **TypeScript:** `{ name: string; age: number }` — структурный тип. Похоже на C# anonymous, но: TS типы — compile-time only (стираются), C# anonymous — настоящий runtime-class. С другой стороны, TS позволяет назвать тип через `type Foo = { ... }`, C# anonymous — нет.
>
> **Rust:** аналога нет. Rust требует именованную `struct` для каждого типа. Ближайшее по идее — tuple struct `struct Foo(i32, String)`, но имена полей через позицию.
>
> **Go:** struct literal `struct{ Name string; Age int }{Name: "John", Age: 30}` — можно объявлять inline. Это ближе всего к C# anonymous, но синтаксис громоздкий.

> [!question]- Интервью: чем anonymous type отличается от tuple?
> Anonymous — class на куче с typed read-only properties. Tuple — `ValueTuple<...>` struct на стеке с mutable fields. Anonymous требует имена полей, tuple может без (`Item1`, `Item2`). Anonymous **нельзя** вернуть из метода как typed (нет имени типа), tuple — можно. Anonymous идиоматичен в LINQ-проекциях (особенно EF Core), tuple — для multi-return, composite keys, swap. Records — superset обоих, если результат уезжает за пределы одного метода.

---

## 2. Создание и синтаксис

### 2.1. Литерал — основной способ

```csharp
// Все имена явно
var person = new { Name = "John", Age = 30 };
Console.WriteLine($"{person.Name}, {person.Age}");

// Имена + выражения
var calculated = new { Sum = 1 + 2, Product = 2 * 3 };
Console.WriteLine($"{calculated.Sum}, {calculated.Product}");   // 3, 6
```

### 2.2. Property promotion — capture из имён

Если значение — это переменная или поле/свойство объекта, имя свойства anonymous-типа можно опустить:

```csharp
string name = "John";
int age = 30;

// Имена выводятся из переменных
var person = new { name, age };
Console.WriteLine($"{person.name}, {person.age}");   // lowercase!
```

⚠️ **Внимание к регистру.** Если переменная называется `name` (lowercase), у anonymous-типа будет свойство `name`. Это нарушает PascalCase-конвенцию C#. Чаще пишут явные имена:

```csharp
string name = "John";
var person = new { Name = name };   // ✅ PascalCase
```

С полями объектов — то же самое, имя свойства совпадает с полем:

```csharp
class Order
{
    public int Id { get; set; }
    public decimal Amount { get; set; }
}

Order o = new() { Id = 1, Amount = 100m };
var summary = new { o.Id, o.Amount };
Console.WriteLine($"{summary.Id}, {summary.Amount}");
```

С вложенными свойствами имя берётся от **последнего** сегмента:

```csharp
var data = new { o.Customer.Name };   // имя свойства = "Name", не "Customer.Name"
```

### 2.3. Mixed синтаксис

Можно комбинировать promotion и явные имена:

```csharp
var summary = new
{
    o.Id,                    // promoted (имя = "Id")
    Customer = o.Customer.Name,   // явное имя
    o.Amount,                // promoted (имя = "Amount")
    Tax = o.Amount * 0.2m    // явное имя + выражение
};
```

### 2.4. Nested anonymous types

Свойство anonymous-типа может быть anonymous-типом:

```csharp
var employee = new
{
    Name = "John",
    Address = new
    {
        City = "Moscow",
        Country = "RU"
    },
    Skills = new[] { "C#", "SQL" },
    Stats = new
    {
        Salary = 100_000m,
        StartDate = new DateTime(2020, 1, 15)
    }
};

Console.WriteLine(employee.Address.City);     // Moscow
Console.WriteLine(employee.Skills[0]);        // C#
Console.WriteLine(employee.Stats.Salary);     // 100000
```

Это нормальный паттерн в EF Core при материализации сложных деревьев — но если глубина больше 2 уровней, читаемость падает. Для глубоких структур — `record` с composition.

### 2.5. Список / массив anonymous-типов

```csharp
var people = new[]
{
    new { Name = "Alice", Age = 30 },
    new { Name = "Bob", Age = 25 },
    new { Name = "Carol", Age = 35 }
};

foreach (var p in people)
    Console.WriteLine($"{p.Name}: {p.Age}");
```

Тип массива — `<>f__AnonymousType0<string, int>[]`. Ключевое условие: **все элементы должны быть одинакового anonymous-типа** (одинаковые имена свойств в одинаковом порядке с совместимыми типами):

```csharp
// ❌ Compile error — разные типы
var bad = new[]
{
    new { Name = "Alice", Age = 30 },
    new { Name = "Bob", Email = "b@x.com" }   // другой shape
};
```

### 2.6. Нельзя — что не работает

```csharp
// Нельзя переписать поле
var p = new { Name = "John" };
p.Name = "Jane";   // ❌ CS0200: Property has no setter

// Нельзя добавить методы
var p = new
{
    Name = "John",
    SayHi = () => Console.WriteLine("Hi")
    // SayHi — это property типа Action, не method
};
p.SayHi();   // OK, но это invocation property, не вызов метода

// Нельзя generic-параметры
// var x = new<T> { Value = default(T) };   // ❌ нельзя

// Нельзя реализовать интерфейс
// var x = new : IDisposable { ... };   // ❌ нельзя
```

> [!question]- Интервью: что значит «property promotion» в anonymous types?
> Это синтаксис, при котором имя свойства anonymous-типа выводится из имени переменной или последнего сегмента доступа. `new { o.Id, o.Customer.Name }` создаёт anonymous с двумя свойствами: `Id` (от `o.Id`) и `Name` (от `o.Customer.Name`). Появилось в C# 3.0 вместе с anonymous types для краткости LINQ-проекций.

---

## 3. Внутреннее устройство

### 3.1. Что генерирует компилятор

Возьмём простое выражение:

```csharp
var p = new { Name = "John", Age = 30 };
```

Компилятор генерирует примерно следующий код в IL (упрощённо):

```csharp
[CompilerGenerated]
[DebuggerDisplay("\\{ Name = {Name}, Age = {Age} }")]
internal sealed class <>f__AnonymousType0<<Name>j__TPar, <Age>j__TPar>
{
    private readonly <Name>j__TPar <Name>i__Field;
    private readonly <Age>j__TPar <Age>i__Field;

    public <Name>j__TPar Name => <Name>i__Field;
    public <Age>j__TPar Age => <Age>i__Field;

    public <>f__AnonymousType0(<Name>j__TPar Name, <Age>j__TPar Age)
    {
        <Name>i__Field = Name;
        <Age>i__Field = Age;
    }

    public override bool Equals(object value)
    {
        var other = value as <>f__AnonymousType0<<Name>j__TPar, <Age>j__TPar>;
        return other != null
            && EqualityComparer<<Name>j__TPar>.Default.Equals(<Name>i__Field, other.<Name>i__Field)
            && EqualityComparer<<Age>j__TPar>.Default.Equals(<Age>i__Field, other.<Age>i__Field);
    }

    public override int GetHashCode()
    {
        // Combined hash от полей
        return ((-1521134295 * -1521134295 + EqualityComparer<<Name>j__TPar>.Default.GetHashCode(<Name>i__Field))
            * -1521134295 + EqualityComparer<<Age>j__TPar>.Default.GetHashCode(<Age>i__Field));
    }

    public override string ToString()
    {
        return "{ Name = " + (<Name>i__Field?.ToString() ?? "") + ", Age = " + (<Age>i__Field?.ToString() ?? "") + " }";
    }
}
```

**Что важно отметить:**

- `<>f__AnonymousType0` — служебное имя (символы `<` `>` запрещены в C# идентификаторах, поэтому никто не может объявить такой тип вручную).
- Тип — **generic**. Параметры типа выводятся из выражения. Для `new { Name = "John", Age = 30 }` будет `<string, int>`.
- Конструктор принимает все поля в том же порядке, что объявлены.
- `Equals` использует `EqualityComparer<T>.Default` для каждого поля — это даёт корректную работу с null, ссылочными типами, value-типами.
- `GetHashCode` комбинирует хэши полей (точная формула отличается между версиями компилятора).
- `ToString` форматирует как `{ Name = X, Age = Y }`.

### 3.2. Type reuse — один тип для одинакового shape

Компилятор переиспользует сгенерированные типы. Если в коде встречается несколько `new { Name = ..., Age = ... }` (одинаковые имена в одинаковом порядке), это **один сгенерированный класс**:

```csharp
var a = new { Name = "Alice", Age = 30 };
var b = new { Name = "Bob", Age = 25 };

Console.WriteLine(a.GetType() == b.GetType());   // True

var c = new { Name = "Carol", Email = "c@x.com" };
Console.WriteLine(a.GetType() == c.GetType());   // False — другой shape
```

Это позволяет хранить anonymous-объекты в массиве (раздел 2.5) и сравнивать их через `Equals`.

### 3.3. Имена и порядок — оба важны

Тип определяется парой `(имя, тип, порядок)`. Изменишь порядок — другой тип:

```csharp
var a = new { Name = "X", Age = 30 };       // <>f__AnonymousType0<string, int>
var b = new { Age = 30, Name = "X" };       // <>f__AnonymousType1<int, string>

Console.WriteLine(a.GetType() == b.GetType());   // False!
Console.WriteLine(a.Equals(b));                  // False
```

**Это самая частая ловушка.** Подробно — раздел 9.

Изменишь имя поля — другой тип:

```csharp
var a = new { Name = "X" };    // AnonymousType<string>
var b = new { Title = "X" };   // AnonymousType<string> (другой!)

Console.WriteLine(a.GetType() == b.GetType());   // False
```

### 3.4. Equals по value, оператор `==` — по ссылке

```csharp
var a = new { Name = "John", Age = 30 };
var b = new { Name = "John", Age = 30 };

Console.WriteLine(a.Equals(b));   // True — value equality
Console.WriteLine(a == b);        // False — reference equality, оператор не override-нут!
```

**Механизм:** `Equals` и `GetHashCode` компилятор переопределяет (это видно в IL). Оператор `==` **не** переопределяет — он остаётся стандартным reference-сравнением для `object`.

Это значит:
- В `Dictionary<>` / `HashSet<>` / `GroupBy` — anonymous работает (там используется `Equals` + `GetHashCode`).
- Сравнение через `==` в обычном коде — **по ссылке**, не по value.

Если хочешь сравнить anonymous по value — пиши `.Equals()`.

### 3.5. Immutability — readonly fields

```csharp
var p = new { Name = "John", Age = 30 };
// p.Name = "Jane";   // ❌ CS0200: Property has no setter
```

Поля сгенерированного класса помечены `readonly`. Свойства не имеют сеттеров. Это полная immutability — никакой `with`-выражение не работает (`with` — это для records, не для anonymous):

```csharp
// ❌ Не работает для anonymous
// var p2 = p with { Age = 31 };
```

Если нужна модифицированная копия — пиши новый литерал:

```csharp
var p = new { Name = "John", Age = 30 };
var p2 = new { p.Name, Age = p.Age + 1 };
```

Это многословнее, чем `with`. Если такие модификации часты — это сигнал «нужен record».

### 3.6. ToString — зашитый формат

```csharp
var p = new { Name = "John", Age = 30 };
Console.WriteLine(p);            // { Name = John, Age = 30 }
Console.WriteLine($"{p}");       // { Name = John, Age = 30 }
Console.WriteLine(p.ToString()); // { Name = John, Age = 30 }
```

Формат не настраиваемый. Если нужен свой формат — записывай через record и переопределяй `ToString`.

### 3.7. DebuggerDisplay в IDE

Visual Studio / Rider показывают anonymous-объекты с теми же полями. Это работает потому, что компилятор генерирует `[DebuggerDisplay("\\{ Name = {Name}, Age = {Age} }")]`. В debug-окне ты видишь свойства как у обычного объекта — это удобно.

> [!question]- Интервью: почему `var a = new { X = 1 }; var b = new { X = 1 }; a == b` вернёт `false`?
> Потому что оператор `==` для anonymous типов **не** переопределён — он работает как reference equality по дефолту для object. `Equals` переопределён по value, и `a.Equals(b)` вернёт `true`. Это типичная ловушка: anonymous имеет value-equality в `Equals`/`GetHashCode` (поэтому работает в `HashSet`/`Dictionary`/`GroupBy`), но не в операторе `==`.

---

## 4. LINQ projections — главное применение

### 4.1. Зачем именно anonymous

LINQ-выражения часто превращают одну форму данных в другую. Старый стиль требовал DTO для каждого преобразования:

```csharp
// До C# 3.0 — на каждую проекцию класс
public class OrderSummary
{
    public int Id { get; set; }
    public string CustomerName { get; set; } = "";
    public decimal Total { get; set; }
}

var summaries = orders
    .Select(o => new OrderSummary
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,
        Total = o.Total
    })
    .ToList();
```

С anonymous types — на месте:

```csharp
var summaries = orders
    .Select(o => new
    {
        o.Id,
        CustomerName = o.Customer.Name,
        o.Total
    })
    .ToList();
```

Те же три поля, но без отдельного класса. Тип элемента — anonymous, существует только внутри метода. IDE видит `summaries` как `List<<>f__AnonymousTypeN<int, string, decimal>>`, autocomplete показывает `Id`, `CustomerName`, `Total`.

### 4.2. Когда DTO лучше anonymous

Anonymous OK **внутри одного метода**. Если результат уезжает наружу — нужен DTO:

```csharp
// ❌ Anonymous нельзя вернуть с типизацией
public ??? GetSummaries() => orders.Select(o => new { o.Id, o.Total }).ToList();

// Workaround: object — теряем типизацию у callers
public List<object> GetSummaries() =>
    orders.Select(o => (object)new { o.Id, o.Total }).ToList();

// Workaround: dynamic — теряем compile-time проверки
public List<dynamic> GetSummaries() =>
    orders.Select(o => (dynamic)new { o.Id, o.Total }).ToList();

// ✅ Правильно — DTO / record
public record OrderSummaryDto(int Id, decimal Total);

public List<OrderSummaryDto> GetSummaries() =>
    orders.Select(o => new OrderSummaryDto(o.Id, o.Total)).ToList();
```

**Эвристика:** anonymous — для внутренней арифметики LINQ. Возврат «наружу» — record / DTO.

### 4.3. Проекция и deferred execution

LINQ `Select` ленивый — anonymous-объекты создаются по мере итерации:

```csharp
var query = orders.Select(o => new { o.Id, o.Total });
// На этой строке ничего не выполнено

foreach (var s in query)
    Console.WriteLine($"{s.Id}: {s.Total}");
// Здесь Select вызывается для каждого элемента,
// anonymous-объекты создаются на лету
```

Если хочешь материализовать — `ToList()` / `ToArray()`:

```csharp
var summaries = orders.Select(o => new { o.Id, o.Total }).ToList();
// Anonymous-объекты созданы ВСЕ, лежат в List
```

### 4.4. Проекция с агрегацией

```csharp
var monthlyStats = orders
    .GroupBy(o => o.Date.Month)
    .Select(g => new
    {
        Month = g.Key,
        Count = g.Count(),
        Total = g.Sum(o => o.Amount),
        Avg = g.Average(o => o.Amount),
        Top = g.Max(o => o.Amount)
    })
    .ToList();

foreach (var s in monthlyStats)
    Console.WriteLine($"Month {s.Month}: {s.Count} orders, total ${s.Total}, avg ${s.Avg:F2}, top ${s.Top}");
```

Один проход через коллекцию — пять статистик в anonymous-объекте.

### 4.5. Проекция вложенная

Часто нужно «свернуть» иерархию в плоский результат с подколлекциями:

```csharp
var customerSummaries = customers
    .Select(c => new
    {
        c.Id,
        c.Name,
        OrderCount = c.Orders.Count,
        TotalSpent = c.Orders.Sum(o => o.Amount),
        RecentOrders = c.Orders
            .OrderByDescending(o => o.Date)
            .Take(3)
            .Select(o => new { o.Id, o.Date, o.Amount })   // вложенный anonymous
            .ToList()
    })
    .ToList();

foreach (var c in customerSummaries)
{
    Console.WriteLine($"{c.Name}: {c.OrderCount} orders, ${c.TotalSpent}");
    foreach (var o in c.RecentOrders)
        Console.WriteLine($"  #{o.Id} on {o.Date:d}: ${o.Amount}");
}
```

В EF Core это разворачивается в один SQL-запрос с подзапросами / JOIN — не N+1. См. раздел 14.

### 4.6. Composite projection с условиями

```csharp
var classified = products
    .Select(p => new
    {
        p.Id,
        p.Name,
        p.Price,
        Category = p.Price switch
        {
            < 100m => "cheap",
            < 1000m => "medium",
            _ => "expensive"
        },
        InStock = p.Quantity > 0
    })
    .ToList();
```

`switch`-выражение, тернарник, любая логика — внутри anonymous-литерала. Это часто чище, чем заводить отдельный класс с computed-полями.

> [!question]- Интервью: почему `var query = orders.Select(o => new { o.Id });` не загружает данные сразу?
> Потому что LINQ — deferred execution. `Select` возвращает `IEnumerable<T>`, но не выполняет проекцию до того, как кто-то итерирует (`foreach`, `ToList`, `ToArray`, `First`). Это классическая ленивость. Anonymous здесь — просто тип элемента, к ленивости отношения не имеет.

---

## 5. GroupBy с composite key

### 5.1. Проблема: группировка по нескольким полям

Хочется сгруппировать orders по году и месяцу для отчёта.

**Без anonymous types** — нужен класс с правильными `Equals` и `GetHashCode`:

```csharp
public class YearMonth : IEquatable<YearMonth>
{
    public int Year { get; init; }
    public int Month { get; init; }

    public bool Equals(YearMonth? other) =>
        other != null && Year == other.Year && Month == other.Month;

    public override bool Equals(object? obj) => Equals(obj as YearMonth);
    public override int GetHashCode() => HashCode.Combine(Year, Month);
}

var grouped = orders.GroupBy(o => new YearMonth { Year = o.Date.Year, Month = o.Date.Month });
```

Многословно. Один класс, который никому больше не нужен.

### 5.2. С anonymous types — встроенное value equality

```csharp
var grouped = orders.GroupBy(o => new { o.Date.Year, Month = o.Date.Month });

foreach (var g in grouped)
    Console.WriteLine($"{g.Key.Year}-{g.Key.Month}: {g.Count()} orders");
```

`GroupBy` использует `Equals` + `GetHashCode` ключа. У anonymous — value equality из коробки. Никакого extra-кода.

### 5.3. Multi-уровневая группировка

```csharp
var report = orders
    .GroupBy(o => new { o.Date.Year, o.Date.Month, o.CustomerType })
    .Select(g => new
    {
        g.Key.Year,
        g.Key.Month,
        g.Key.CustomerType,
        Count = g.Count(),
        Revenue = g.Sum(o => o.Amount)
    })
    .OrderBy(r => r.Year)
    .ThenBy(r => r.Month)
    .ThenBy(r => r.CustomerType)
    .ToList();

foreach (var row in report)
    Console.WriteLine($"{row.Year}-{row.Month} {row.CustomerType}: {row.Count} orders, ${row.Revenue}");
```

Три измерения группировки. Anonymous-ключ имеет встроенный equals по всем трём полям.

### 5.4. Anonymous vs tuple для composite key

Tuples тоже работают как ключи. Что выбрать?

```csharp
// Anonymous
.GroupBy(o => new { o.Date.Year, o.Date.Month })
.Select(g => new { g.Key.Year, g.Key.Month, Total = g.Sum(o => o.Amount) })

// Tuple
.GroupBy(o => (o.Date.Year, o.Date.Month))
.Select(g => new { g.Key.Year, g.Key.Month, Total = g.Sum(o => o.Amount) })
```

Что выбрать:

| | Anonymous | Tuple |
|---|---|---|
| EF Core (LINQ to SQL) | ✅ Гарантированная поддержка | ⚠️ Зависит от провайдера |
| In-memory LINQ | ✅ Работает | ✅ Работает |
| Имена в `g.Key` | `g.Key.Year`, `g.Key.Month` | `g.Key.Year`, `g.Key.Month` (если named) |
| Аллокация ключа | Heap (class) | Stack (struct) |
| Производительность | Чуть медленнее | Чуть быстрее (struct, no GC) |

**Эвристика:** в **EF Core** — anonymous (надёжнее транслируется). В **in-memory** LINQ — tuple (без heap-аллокаций ключа).

### 5.5. Sliding-window и tuples для агрегатора

Anonymous больше подходит для projection. Для аккумулятора (`Aggregate`) tuple часто удобнее, потому что деконструируется:

```csharp
// Tuple-аккумулятор — деконструкция в конце
var (min, max, sum) = numbers.Aggregate(
    (Min: int.MaxValue, Max: int.MinValue, Sum: 0),
    (acc, n) => (Math.Min(acc.Min, n), Math.Max(acc.Max, n), acc.Sum + n)
);

// Anonymous-аккумулятор — без деконструкции, через свойства
var stats = numbers.Aggregate(
    new { Min = int.MaxValue, Max = int.MinValue, Sum = 0 },
    (acc, n) => new
    {
        Min = Math.Min(acc.Min, n),
        Max = Math.Max(acc.Max, n),
        Sum = acc.Sum + n
    }
);
Console.WriteLine($"{stats.Min}, {stats.Max}, {stats.Sum}");
```

Anonymous-аккумулятор хуже tuple — на каждой итерации создаётся новый объект на куче, а tuple — struct copy на стеке. Для `Aggregate` всегда бери tuple.

> [!question]- Интервью: почему `GroupBy` работает с anonymous-ключом без дополнительного кода?
> Потому что anonymous-тип имеет переопределённые `Equals` и `GetHashCode` по value-семантике (компилятор генерирует их автоматически). `GroupBy` опирается на эти методы для разбиения по ключам. С обычным class пришлось бы переопределить их вручную или реализовать `IEquatable<T>`.

---

## 6. JOIN с anonymous projection

### 6.1. Multi-table результат без DTO

JOIN-ы часто склеивают данные из нескольких таблиц. Без anonymous пришлось бы заводить DTO для результата:

```csharp
// ❌ Скучно — лишний класс для одного JOIN
public class OrderWithCustomerDto
{
    public int OrderId { get; set; }
    public decimal Total { get; set; }
    public string CustomerName { get; set; } = "";
    public string ShippingCity { get; set; } = "";
}
```

С anonymous — на месте:

```csharp
var data = await db.Orders
    .Join(db.Customers,
        o => o.CustomerId,
        c => c.Id,
        (o, c) => new { Order = o, Customer = c })
    .Join(db.Addresses,
        oc => oc.Customer.AddressId,
        a => a.Id,
        (oc, a) => new
        {
            oc.Order.Id,
            oc.Order.Total,
            CustomerName = oc.Customer.Name,
            ShippingCity = a.City,
            ShippingZip = a.PostalCode
        })
    .ToListAsync();

foreach (var item in data)
    Console.WriteLine($"Order {item.Id} → {item.CustomerName} в {item.ShippingCity}");
```

EF Core транслирует это в один SQL-запрос с двумя JOIN. Без отдельного класса.

### 6.2. Promotion-trick — protected naming

В JOIN-проекциях имена легко конфликтуют. Если у `Order` и `Customer` обе есть свойство `Id` — promotion даст конфликт:

```csharp
var bad = orders.Join(customers,
    o => o.CustomerId,
    c => c.Id,
    (o, c) => new { o.Id, c.Id });
// ❌ CS0833: An anonymous type cannot have multiple properties with the same name
```

Решение — явные имена:

```csharp
var ok = orders.Join(customers,
    o => o.CustomerId,
    c => c.Id,
    (o, c) => new { OrderId = o.Id, CustomerId = c.Id });
```

### 6.3. Query syntax с anonymous

В query syntax `into` создаёт промежуточный anonymous-тип неявно:

```csharp
var query = from o in orders
            join c in customers on o.CustomerId equals c.Id
            join a in addresses on c.AddressId equals a.Id
            select new
            {
                o.Id,
                CustomerName = c.Name,
                City = a.City
            };
```

Промежуточные объекты в `from`/`join` — это анонимные типы, которые компилятор создаёт по мере цепочки.

### 6.4. Group join + anonymous

```csharp
var customersWithOrders = from c in customers
                          join o in orders on c.Id equals o.CustomerId into customerOrders
                          select new
                          {
                              c.Id,
                              c.Name,
                              OrderCount = customerOrders.Count(),
                              TotalSpent = customerOrders.Sum(o => o.Amount),
                              Orders = customerOrders.ToList()   // вложенная коллекция
                          };
```

`into customerOrders` создаёт `IEnumerable<Order>` для каждого customer, потом `select` упаковывает в anonymous-объект.

### 6.5. Self-join и anonymous

```csharp
// Hierarchy: parent-child relationships
var hierarchy = categories.Join(categories,
    c => c.ParentId,
    p => p.Id,
    (c, p) => new
    {
        ChildId = c.Id,
        ChildName = c.Name,
        ParentId = p.Id,
        ParentName = p.Name
    });
```

Anonymous хорошо ложится на «отношения между двумя экземплярами одной таблицы» — без DTO.

> [!question]- Интервью: что произойдёт с `new { o.Id, c.Id }` в JOIN-проекции?
> Compile-error CS0833: anonymous-тип не может иметь два свойства с одинаковым именем. Property promotion даёт оба свойства имя `Id`, что конфликтует. Решение — явные имена: `new { OrderId = o.Id, CustomerId = c.Id }`.

---

## 7. Structured logging с anonymous

### 7.1. Зачем anonymous в логировании

Современные логгеры (Serilog, ILogger, Seq, Datadog) умеют structured logging — лог это не строка, а событие с typed-полями. Anonymous types — естественный способ передать context-объект:

```csharp
_logger.LogInformation(
    "Order {OrderId} processed for customer {CustomerName} with total {Total:C}",
    order.Id, customer.Name, order.Total);
// Это работает, но именованные параметры писать нудно
```

С anonymous и `@`-prefix — компактнее:

```csharp
_logger.LogInformation("Order processed: {@OrderInfo}", new
{
    OrderId = order.Id,
    CustomerName = customer.Name,
    Total = order.Total,
    Items = order.Items.Count,
    Duration = stopwatch.ElapsedMilliseconds
});
// В Serilog → JSON-документ с полями OrderInfo.OrderId, OrderInfo.CustomerName, ...
```

### 7.2. Префиксы `@` vs `$` в message templates

В Serilog (и совместимых) у placeholder-ов есть специальные префиксы:

- **Без префикса** (`{Foo}`) — `ToString()` вызывается, в лог идёт строка.
- **`@` префикс** (`{@Foo}`) — destructuring: объект разворачивается в JSON, видны все поля.
- **`$` префикс** (`{$Foo}`) — принудительный `ToString()` (полезно для override-нутого вывода).

```csharp
var customer = new { Id = 1, Name = "Alice", Email = "a@x.com" };

// Без префикса — Serilog логирует customer.ToString() = "{ Id = 1, Name = Alice, Email = a@x.com }"
_logger.LogInformation("Customer: {Customer}", customer);

// С @ — Serilog раскрывает в JSON
_logger.LogInformation("Customer: {@Customer}", customer);
// Лог: { "Customer": { "Id": 1, "Name": "Alice", "Email": "a@x.com" } }
```

В Seq / ELK / Datadog искать по `Customer.Name = "Alice"` можно только если использовался `@`. Это решающее.

### 7.3. Реальный сценарий — try/catch с context

```csharp
public async Task PlaceOrderAsync(int customerId, List<int> productIds)
{
    var stopwatch = Stopwatch.StartNew();
    try
    {
        var order = await ProcessOrderAsync(customerId, productIds);
        _logger.LogInformation("Order placed: {@OrderInfo}", new
        {
            OrderId = order.Id,
            CustomerId = customerId,
            ItemCount = productIds.Count,
            Total = order.Total,
            DurationMs = stopwatch.ElapsedMilliseconds
        });
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Order failed: {@Context}", new
        {
            CustomerId = customerId,
            ProductIds = productIds,
            DurationMs = stopwatch.ElapsedMilliseconds
        });
        throw;
    }
}
```

В centralized log системе ты можешь:
- Найти все ордера определённого клиента: `OrderInfo.CustomerId = 42`.
- Получить все ошибки за час по продукту: `Context.ProductIds in [123, 456]`.
- Построить графики latency: `OrderInfo.DurationMs`.

Без structured logging это всё было бы парсинг строк — медленно, хрупко, непрактично.

### 7.4. ILogger vs Serilog — поведение

`Microsoft.Extensions.Logging.ILogger` сам по себе **не** делает destructuring через `@`. Для этого нужен `Serilog` (или другой провайдер с поддержкой message templates). Но синтаксис в коде один и тот же — `LoggerMessage.Define`/`LogInformation` принимают template, провайдер за кадром решает, что с ним делать.

```csharp
// Стандартный ILogger без Serilog — @ игнорируется, используется ToString()
_logger.LogInformation("Data: {@Data}", new { X = 1 });
// Лог: "Data: { X = 1 }"

// С Serilog — @ распознаётся, destructuring выполняется
// Лог (JSON): {"Message":"Data: {@Data}","Data":{"X":1}}
```

### 7.5. Не злоупотреблять `@`

`@`-destructuring — heavy operation. Reflection обходит все свойства, рекурсивно (для nested anonymous). На горячих логах это может стать узким местом.

Правило: используй `@` для богатого contextual логирования (один-два раза за HTTP-запрос). Для каждой строчки в цикле — обычные `{Var}` placeholders.

> [!question]- Интервью: что значит `{@Foo}` в Serilog message template?
> Префикс `@` говорит логгеру выполнить destructuring — обойти свойства объекта через reflection и положить их в лог как структурированные поля. Без `@` объект логируется через `ToString()`. С `@` — раскрывается в JSON-объект с searchable полями. Anonymous types — типичный input для `{@...}`.

---

## 8. Razor / Blazor view models

### 8.1. Anonymous как ViewModel — соблазнительно, но плохо

В MVC / Razor Pages соблазнительно передать в view anonymous-объект:

```csharp
public IActionResult Dashboard()
{
    var data = new
    {
        UserName = User.Identity?.Name,
        ActiveOrders = _db.Orders.Count(o => o.Status == "Active"),
        Revenue = _db.Orders.Where(o => o.Date >= DateTime.Today).Sum(o => o.Total),
        TopProducts = _db.Products.OrderByDescending(p => p.SalesCount).Take(5).ToList()
    };
    return View(data);
}
```

В Razor view:

```razor
@model dynamic

<h1>Hello @Model.UserName</h1>
<p>Active: @Model.ActiveOrders, Today's revenue: @Model.Revenue:C</p>
```

Это **работает**, но:

1. **`@model dynamic`** — отказ от типизации. IntelliSense не работает, опечатка в `@Model.UserNme` не ловится компилятором.
2. **Razor-страница в другой assembly.** Сгенерированный anonymous-тип internal в исходной assembly. Razor view (другая assembly) не имеет к нему доступа как к typed классу — отсюда `dynamic`.
3. **Reflection на каждый доступ.** `dynamic` Member access проходит через DLR (Dynamic Language Runtime) — медленно, runtime-resolved.

### 8.2. Правильный подход — ViewModel record

```csharp
public record DashboardViewModel(
    string UserName,
    int ActiveOrders,
    decimal Revenue,
    List<Product> TopProducts);

public IActionResult Dashboard()
{
    var data = new DashboardViewModel(
        UserName: User.Identity?.Name ?? "Anonymous",
        ActiveOrders: _db.Orders.Count(o => o.Status == "Active"),
        Revenue: _db.Orders.Where(o => o.Date >= DateTime.Today).Sum(o => o.Total),
        TopProducts: _db.Products.OrderByDescending(p => p.SalesCount).Take(5).ToList()
    );
    return View(data);
}
```

В Razor:

```razor
@model DashboardViewModel

<h1>Hello @Model.UserName</h1>
<p>Active: @Model.ActiveOrders, Today's revenue: @Model.Revenue:C</p>
```

Работает IntelliSense, опечатки ловятся компилятором (Razor-страница компилируется в полноценный класс).

### 8.3. Где anonymous OK в Razor

В **partial views** или **components** для маленьких кусков shape — anonymous работает через `dynamic`. Если контент тривиальный и не требует типизации:

```razor
@* _BadgeRow.cshtml *@
@model dynamic
<div class="badge">@Model.Label: @Model.Value</div>
```

```razor
@await Html.PartialAsync("_BadgeRow", new { Label = "Total", Value = "$1234" })
@await Html.PartialAsync("_BadgeRow", new { Label = "Orders", Value = "42" })
```

Но для сложных страниц всегда ViewModel.

### 8.4. Blazor — anonymous ещё хуже

В Blazor компоненты в основном принимают `[Parameter]` typed-свойства. Anonymous там почти не имеет смысла:

```csharp
@* ❌ Так не получится — нет typed-параметра *@
<MyComponent Data="new { X = 1, Y = 2 }" />
```

Blazor требует named parameters. Anonymous остаётся внутри `@code { ... }` блоков для локальных проекций (как обычный C# код). Для UI-state — фигуры с record / class.

### 8.5. Краткое резюме по UI

| Сценарий | Решение |
|----------|---------|
| MVC view model | record / class |
| MVC partial view (маленький) | anonymous + dynamic |
| Blazor component parameter | record / class |
| Blazor локальная проекция | anonymous OK (внутри `@code`) |
| Razor Pages PageModel | record / class |
| API response | record / DTO |

> [!question]- Интервью: почему передача anonymous в Razor требует `@model dynamic`?
> Потому что anonymous-тип `internal sealed` в той assembly, где он создан. Razor-страницы компилируются в отдельную assembly и не имеют доступа к internal типам. Чтобы обойти ограничение типизации, используется `dynamic` — но это убивает IntelliSense и compile-time проверки. Правильный путь — определить ViewModel record.

---

## 9. Property order matters — главная ловушка

### 9.1. Тип определяется shape: имена + типы + порядок

Это самая частая ловушка anonymous-типов. Рассмотрим:

```csharp
var a = new { Name = "X", Age = 30 };       // <>f__AnonymousType0<string, int>
var b = new { Age = 30, Name = "X" };       // <>f__AnonymousType1<int, string>

Console.WriteLine(a.GetType() == b.GetType());   // False
Console.WriteLine(a.Equals(b));                  // False
```

Объекты выглядят семантически одинаковыми (те же поля, те же значения), но **типы разные**, потому что компилятор различает их по **порядку** объявления полей.

### 9.2. Как это срабатывает в production

Чаще всего ловушка случается в условных проекциях:

```csharp
List<object> Build(bool ascending)
{
    if (ascending)
        return orders.Select(o => new { o.Id, o.Total }).Cast<object>().ToList();
    else
        return orders.Select(o => new { o.Total, o.Id }).Cast<object>().ToList();
    // Два разных anonymous-типа!
}
```

В пути «true» — тип `<int, decimal>`, в пути «false» — `<decimal, int>`. Поведение метода зависит от условия, что ломает консистентность.

### 9.3. Equals по shape, не по «семантике»

```csharp
var a = new { Year = 2024, Month = 5 };
var b = new { Month = 5, Year = 2024 };

a.Equals(b);   // False — разные типы
```

Хотя «логически» это та же дата, anonymous различает их по shape. Если нужна семантическая равность — заводи record, у которого тип имеет имя:

```csharp
public record YearMonth(int Year, int Month);

var a = new YearMonth(2024, 5);
var b = new YearMonth(Year: 2024, Month: 5);
a.Equals(b);   // True — то же
```

### 9.4. Тестирование

В юнит-тестах anonymous иногда используется как expected-объект:

```csharp
var actual = service.GetOrderSummary(1);
var expected = new { Id = 1, Total = 100m };

Assert.Equal(expected, actual);
```

Это работает только если actual и expected — **в одной assembly** и **тот же shape** (имена и порядок). Если service возвращает `List<object>` или anonymous из другой assembly — `Equals` упадёт по reference. Для тестов лучше всё равно record.

### 9.5. Best-practice: каноничный порядок

Если по проекту есть «канонические» проекции — фиксируй порядок и придерживайся:

```csharp
// В нашем проекте все order-проекции идут в порядке: Id, CustomerName, Total, Date
var summary = orders.Select(o => new
{
    o.Id,
    CustomerName = o.Customer.Name,
    o.Total,
    o.Date
});
```

Это снижает риск, что в одном месте будет `(Id, Total)`, а в другом — `(Total, Id)`.

> [!question]- Интервью: почему `new { A = 1, B = 2 }` и `new { B = 2, A = 1 }` — разные типы?
> Компилятор генерирует уникальный сгенерированный класс для каждого уникального shape, где shape = (имена полей, типы, порядок). Изменение порядка → новое объявление типа. Это не баг, это часть design rationale: anonymous types часто используются в LINQ и должны различаться по shape, чтобы EF Core / providers могли надёжно маппить их в SQL-запросы. Если нужна семантическая равность — заводи именованный record.

---

## 10. Anonymous vs Tuple vs Record vs Class

### 10.1. Все четыре похожи — но разные

```csharp
// Anonymous — local-only, immutable
var anon = new { Name = "Alice", Age = 30 };

// ValueTuple — multi-return, composite key, deconstruction
var tuple = (Name: "Alice", Age: 30);

// Record — first-class type с value equality
public record Person(string Name, int Age);
var rec = new Person("Alice", 30);

// Class — domain type с поведением, identity
public class Customer
{
    public string Name { get; init; } = "";
    public int Age { get; init; }
    public void Greet() { Console.WriteLine($"Hi, I'm {Name}"); }
}
var cls = new Customer { Name = "Alice", Age = 30 };
```

### 10.2. Сравнительная таблица

| Свойство | Anonymous | ValueTuple | Record | Class |
|----------|:---------:|:----------:|:------:|:-----:|
| Имена полей обязательны | ✅ | ❌ (Item1) | ✅ | ✅ |
| Mutability | Read-only | Mutable | Init-only | Customizable |
| Equality | Value (Equals only) | Value | Value | Reference (default) |
| Оператор `==` | Reference | Value | Value | Reference |
| Cross-method | ❌ | ✅ | ✅ | ✅ |
| Cross-assembly | ❌ | ✅ | ✅ | ✅ |
| Возврат из метода (typed) | ❌ | ✅ | ✅ | ✅ |
| Inheritance | ❌ | ❌ | ✅ (record class) | ✅ |
| `with` expression | ❌ | ❌ | ✅ | ❌ |
| Methods на типе | ❌ | ❌ | ✅ | ✅ |
| Аллокация | Heap | Stack | Heap (record class) / Stack (record struct) | Heap |
| Pattern matching positional | ❌ | ✅ | ✅ | ❌ |
| ToString с полями | ✅ | ✅ | ✅ | Default `Type.FullName` |
| Reflection — есть имена | ✅ | ⚠️ Только в сигнатуре | ✅ | ✅ |
| Generic параметры | Implicit | Implicit | Yes | Yes |

### 10.3. Когда что — практическая эвристика

```
Где живёт результат?
│
├── Внутри одного метода (LINQ, локальный shape)
│   └── Anonymous (для проекций) или Tuple (для multi-return)
│
├── Между методами / классами
│   └── Record (если value-семантика) или Class (если поведение)
│
├── Cross-assembly (NuGet, public API)
│   └── Record / Class
│
└── Сериализуется в JSON / БД
    └── Record / DTO

Mutable?
│
├── Да, нужно менять
│   └── Class (с set / public fields)
│
└── Нет, immutable
    └── Anonymous / Tuple / Record

Identity matters?
│
├── Да (User #42 — это конкретный пользователь)
│   └── Class (с Id и потенциально override Equals)
│
└── Нет (две точки (1,2) — равны)
    └── Record / Anonymous / Tuple

Behaviour (методы, валидация)?
│
├── Да
│   └── Record / Class
│
└── Нет, чисто данные
    └── Anonymous / Tuple / Record DTO
```

### 10.4. Конкретные сценарии — что выбрать

| Сценарий | Лучший выбор | Почему |
|----------|--------------|--------|
| LINQ projection в EF Core | **Anonymous** | EF гарантированно транслирует в SQL |
| LINQ projection in-memory | Anonymous или Tuple | Оба ok, anonymous чуть выразительнее |
| Composite key для GroupBy | **Anonymous** или Tuple | Оба value equality |
| Multi-return из метода | **Tuple** или Record | Tuple — для коротких, record — для public |
| Aggregate accumulator | **Tuple** | Stack-аллокация на итерации |
| Public API response | **Record / DTO** | Стабильный контракт, JSON-friendly |
| Domain entity (User, Order) | **Class** | Identity, поведение |
| Value Object (Money, Email) | **Record** | Value equality + immutable |
| Logging context | **Anonymous** | Inline, не загромождает |
| MVC ViewModel | **Record** или Class | Razor требует named тип |
| Test fixture data | **Record** | Reusable, читаемо |

### 10.5. Что не стоит делать

```csharp
// ❌ Anonymous как fake-class
var user = new { Name = "...", Age = 30 };
SomeMethod(user);   // Метод вынужден принимать object или dynamic

// ❌ Tuple для domain identity
var customer = ("Alice", 30, "alice@x.com");   // что значит каждый элемент?

// ❌ Record для mutable bag-of-properties с поведением
public record User(string Name, int Age)
{
    private List<string> _logs = new();   // mutable state в record — anti-pattern
    public void Log(string msg) => _logs.Add(msg);
}

// ❌ Class для одноразового LINQ-projection
public class TempProjection { public int Id; public decimal Total; }
orders.Select(o => new TempProjection { Id = o.Id, Total = o.Total });
// Лишний класс на одно использование — заводи anonymous
```

> [!question]- Интервью: какие 4 варианта представления группы значений в C#, и когда что выбирать?
> Anonymous, Tuple, Record, Class. Anonymous — для локальных LINQ-проекций (EF Core), не вынести из метода. Tuple — для multi-return и composite keys, struct на стеке. Record — для cross-method и cross-assembly, value-семантика, поддерживает `with`. Class — для domain entities с identity и поведением. Если нужна mutability — class. Если важна семантика типа — record. Если данные не должны существовать вне метода — anonymous или tuple.

---

## 11. Boundaries — почему anonymous нельзя вернуть

### 11.1. Корень проблемы — нет имени типа

```csharp
public ??? GetSummary()
{
    return new { Id = 1, Total = 100m };
}
```

Чтобы метод имел тип возвращаемого значения, нужно имя типа. У anonymous имя есть в IL (`<>f__AnonymousTypeN`), но оно содержит запрещённые в C# символы (`<`, `>`) — синтаксически написать его нельзя.

### 11.2. Workaround #1 — return object

```csharp
public object GetSummary() => new { Id = 1, Total = 100m };

// На стороне caller — теряем типизацию
var s = service.GetSummary();
Console.WriteLine(s);   // Работает (вызовется ToString)

// Но достучаться до полей...
// Console.WriteLine(s.Id);   // ❌ object не имеет Id
```

Для доступа — reflection (медленно) или dynamic (см. ниже).

### 11.3. Workaround #2 — return dynamic

```csharp
public dynamic GetSummary() => new { Id = 1, Total = 100m };

// Caller использует dynamic — DLR резолвит на runtime
dynamic s = service.GetSummary();
Console.WriteLine(s.Id);     // 1 — работает! Но...
Console.WriteLine(s.Idd);    // RuntimeBinderException на runtime — опечатки не ловятся!
```

`dynamic` отключает compile-time проверки. Опечатка → exception в runtime. Это очень хрупко.

### 11.4. Workaround #3 — generic с inference

Классический трюк для «передачи anonymous между методами»:

```csharp
public static class AnonymousHelper
{
    public static T Cast<T>(object obj, T anonymous) => (T)obj;
}

object data = new { Name = "Alice", Age = 30 };

// caller знает shape и передаёт «образец» для inference
var typed = AnonymousHelper.Cast(data, new { Name = "", Age = 0 });
Console.WriteLine(typed.Name);   // Alice
Console.WriteLine(typed.Age);    // 30
```

Работает потому, что компилятор reuse-ит anonymous-типы по shape — сгенерированный класс в `Helper.Cast<T>` тот же, что и при создании `data`.

Это **хрупко** и обычно сигнал «всё-таки заводи record».

### 11.5. Workaround #4 — generic метод + delegate

```csharp
public static T GetData<T>(Func<T> producer) => producer();

var data = GetData(() => new { Name = "Alice", Age = 30 });
Console.WriteLine(data.Name);   // OK
```

Тип `T` выводится из лямбды-producer. Anonymous возвращён, но метод **generic**, тип не fixed на сигнатуре.

### 11.6. Правильный путь — record

Все workaround-ы — обходы. Если ты хочешь возвращать anonymous-shape — это сигнал, что данным пора получить имя:

```csharp
public record Summary(int Id, decimal Total);

public Summary GetSummary() => new(1, 100m);

// Caller
var s = service.GetSummary();
Console.WriteLine(s.Id);     // ✅ typed
Console.WriteLine(s.Total);  // ✅ typed
```

`record` — это «anonymous, но с именем». В C# 9+ это лёгкий синтаксис, не требующий многострочного объявления:

```csharp
public record Summary(int Id, decimal Total);   // одна строка
```

> [!question]- Интервью: можно ли вернуть anonymous-тип из метода?
> Технически — только как `object` или `dynamic`. У anonymous-типа есть имя в IL, но оно содержит запрещённые в C# символы (`<>f__AnonymousType0`), его нельзя написать вручную. Через `object` теряется типизация. Через `dynamic` — DLR резолвит свойства в runtime, но опечатки не ловятся компилятором. Правильный путь — заводить record, у которого имя есть и пишется явно.

---

## 12. Reflection и dynamic — обход границы

### 12.1. Reflection для anonymous-объекта

Через reflection можно прочитать любые свойства anonymous-объекта:

```csharp
object data = new { Name = "Alice", Age = 30 };

var type = data.GetType();
Console.WriteLine($"Type: {type.Name}");   // <>f__AnonymousType0`2

foreach (var prop in type.GetProperties())
{
    var value = prop.GetValue(data);
    Console.WriteLine($"  {prop.Name} = {value}");
}
// Name = Alice
// Age = 30
```

Это работает, но:
- **Медленно.** Reflection дороже direct access на 1-2 порядка.
- **Не type-safe.** `prop.GetValue` возвращает `object`, надо приводить.

### 12.2. Generic helper для типизированного доступа

Сочетание generic + lambda для типизированного reading:

```csharp
public static TResult GetField<T, TResult>(T obj, Func<T, TResult> selector) => selector(obj);

var data = new { Name = "Alice", Age = 30 };
string name = GetField(data, x => x.Name);   // typed, без reflection
int age = GetField(data, x => x.Age);
```

Это compile-time, не reflection. Но, опять же, признак того, что нужен record.

### 12.3. dynamic для object с anonymous-shape

```csharp
public object Receive() => new { Name = "Alice", Age = 30 };

dynamic d = Receive();
Console.WriteLine(d.Name);   // Alice — DLR резолвит на runtime
Console.WriteLine(d.Age);    // 30
```

DLR (Dynamic Language Runtime) кэширует резолюции, поэтому второй вызов того же свойства быстрее первого. Но всё равно медленнее direct access.

**Что использует под капотом:**
1. Первый доступ к `d.Name` — DLR делает reflection lookup, генерирует callsite.
2. Кэширует тип объекта и binding для свойства.
3. Второй вызов — попадает в кэш, почти как direct.

### 12.4. ExpandoObject — альтернатива для динамических shape

Если нужна **mutable** dynamic-структура с произвольными свойствами:

```csharp
using System.Dynamic;

dynamic e = new ExpandoObject();
e.Name = "Alice";
e.Age = 30;
e.Greet = (Action)(() => Console.WriteLine($"Hi, I'm {e.Name}"));

e.Greet();   // Hi, I'm Alice
e.Email = "a@x.com";   // Можно добавлять свойства!
```

Это **не** anonymous-тип — это `ExpandoObject` (класс из BCL), который реализует `IDictionary<string, object>` и `IDynamicMetaObjectProvider`. Свойства живут в словаре, не в IL-структуре.

Используется для:
- Build-up DTO из конфигурации.
- Mock-объектов в тестах.
- Парсинга JSON в обходных случаях.

Дороже anonymous (heap-аллокация словаря), но гибче. Для 90% задач — `record` лучше.

### 12.5. JSON deserialization без класса

Часто нужно «распарсить JSON, не зная shape заранее»:

```csharp
using System.Text.Json;

string json = """{"name":"Alice","age":30}""";

// Через JsonElement (типобезопасно, но verbose)
JsonDocument doc = JsonDocument.Parse(json);
string name = doc.RootElement.GetProperty("name").GetString()!;
int age = doc.RootElement.GetProperty("age").GetInt32();

// Через dynamic + Newtonsoft (короче, не type-safe)
dynamic? parsed = Newtonsoft.Json.JsonConvert.DeserializeObject(json);
string name2 = parsed!.name;
int age2 = parsed.age;
```

В обоих случаях — это **не** anonymous-тип. Anonymous-тип создаётся только при `new { ... }` в исходном коде. JSON-парсеры дают тебе `JsonElement`, `JsonNode`, `Dictionary<string, object>`, `JObject` — что-то другое.

> [!question]- Интервью: чем `dynamic` отличается от `object` для anonymous-типа?
> `object` — статический тип, доступ к свойствам только через reflection (`obj.GetType().GetProperty(...).GetValue(obj)`). `dynamic` — снимает compile-time проверки и переводит резолюцию в runtime через DLR. `dynamic d = anonymous; d.Name` работает (DLR резолвит на runtime), но опечатки → `RuntimeBinderException` в runtime. `dynamic` быстрее reflection (за счёт кэширования call-site), но медленнее direct access на typed поле.

---

## 13. JSON serialization

### 13.1. System.Text.Json + anonymous works

В отличие от tuples, anonymous types сериализуются **с именами полей**:

```csharp
using System.Text.Json;

var data = new { Name = "Alice", Age = 30, Email = "a@x.com" };
string json = JsonSerializer.Serialize(data);
Console.WriteLine(json);
// {"Name":"Alice","Age":30,"Email":"a@x.com"}
```

Почему работает: anonymous-type — это обычный class с public read-only properties. `JsonSerializer` обходит свойства через reflection и берёт их имена.

### 13.2. Newtonsoft.Json — то же

```csharp
using Newtonsoft.Json;

var data = new { Name = "Alice", Age = 30 };
string json = JsonConvert.SerializeObject(data);
// {"Name":"Alice","Age":30}
```

### 13.3. Опции — camelCase, indented

```csharp
var data = new { FirstName = "Alice", LastName = "Smith", Age = 30 };

var options = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = true
};

string json = JsonSerializer.Serialize(data, options);
Console.WriteLine(json);
// {
//   "firstName": "Alice",
//   "lastName": "Smith",
//   "age": 30
// }
```

### 13.4. Десериализация — почти невозможна как typed

```csharp
string json = "{\"Name\":\"Alice\",\"Age\":30}";

// Десериализация в anonymous — нельзя без trick
// var data = JsonSerializer.Deserialize<AnonymousType???>(json);   // нет имени типа

// Trick через generic-метод с образцом
public static T? DeserializeAnonymous<T>(string json, T anonymous)
    => JsonSerializer.Deserialize<T>(json);

var data = DeserializeAnonymous(json, new { Name = "", Age = 0 });
Console.WriteLine(data?.Name);   // Alice
Console.WriteLine(data?.Age);    // 30
```

Работает, но это сигнал «нужен record»:

```csharp
public record Person(string Name, int Age);
var data = JsonSerializer.Deserialize<Person>(json);
```

### 13.5. NRT и anonymous serialization

Anonymous-type не поддерживает атрибуты (на свойства), поэтому управлять сериализацией granularly нельзя:

```csharp
// ❌ Атрибуты на anonymous-полях нельзя
var data = new
{
    [JsonPropertyName("name")] Name = "Alice",   // ❌ синтаксис не поддерживается
    Age = 30
};
```

Если нужны атрибуты сериализации (`[JsonPropertyName]`, `[JsonIgnore]`, `[JsonConverter]`) — заводи record:

```csharp
public record Person(
    [property: JsonPropertyName("first_name")] string FirstName,
    [property: JsonIgnore] int InternalId);
```

### 13.6. ASP.NET Core controller с anonymous

```csharp
[HttpGet("summary")]
public IActionResult GetSummary()
{
    return Ok(new
    {
        Total = 100,
        Date = DateTime.UtcNow,
        Items = new[] { "a", "b", "c" }
    });
}
```

Это **работает**, но:
- OpenAPI / Swagger не сможет вывести схему ответа из anonymous (нет имени типа).
- Клиент-генерируемые SDK получат `object` или `any`.
- Тестировать (`Assert.Equal(expected, response.Body)`) — больно.

В production-API всегда возвращай **named тип** (record / class):

```csharp
public record SummaryResponse(int Total, DateTime Date, string[] Items);

[HttpGet("summary")]
public ActionResult<SummaryResponse> GetSummary() =>
    new SummaryResponse(100, DateTime.UtcNow, ["a", "b", "c"]);
```

### 13.7. XmlSerializer и anonymous — нет

`XmlSerializer` требует public default constructor. У anonymous-типа конструктор только параметризованный:

```csharp
var data = new { Name = "Alice" };
new XmlSerializer(data.GetType()).Serialize(Console.Out, data);
// ❌ InvalidOperationException: <>f__AnonymousType0 cannot be serialized because it does not have a parameterless constructor.
```

Если нужен XML — record / class:

```csharp
[XmlRoot("Person")]
public class Person
{
    public string Name { get; set; } = "";
    public int Age { get; set; }
}
```

> [!question]- Интервью: что произойдёт, если сериализовать `new { Name = "Alice" }` через `JsonSerializer.Serialize`?
> Получишь `{"Name":"Alice"}` — anonymous-тип сериализуется как обычный class с публичными свойствами. Но десериализовать обратно как typed anonymous **нельзя** (нет имени типа для дженерика). Если нужен round-trip — заводи record.

---

## 14. EF Core deep — anonymous + SQL translation

### 14.1. Почему anonymous каноничен в EF Core

EF Core LINQ-провайдер видит выражение `Select(o => new { o.Id, o.Total })` и переводит его в SQL. Anonymous-типы в LINQ к EF Core поддерживались с самого начала (EF6 → EF Core 1.0). Tuples появились позже, и поддержка tuples была неполной долгое время.

```csharp
var summaries = db.Orders
    .Where(o => o.Status == "active")
    .Select(o => new { o.Id, o.Total })
    .ToList();

// EF Core генерирует SQL:
// SELECT [o].[Id], [o].[Total]
// FROM [Orders] AS [o]
// WHERE [o].[Status] = N'active'
```

Это **projection**: SQL запрашивает **только нужные колонки**, не весь Order. Это принципиально для производительности — не таскать данные, которые не нужны.

### 14.2. Без projection — full entity load

```csharp
var orders = db.Orders.Where(o => o.Status == "active").ToList();
// SELECT [o].[Id], [o].[CustomerId], [o].[Total], [o].[Date], [o].[Status], [o].[Notes], [o].[..] — все колонки
```

Если нужны только `Id` и `Total` — это лишний трафик. Always project в anonymous (или DTO), если не нужна полная сущность.

### 14.3. AsNoTracking + projection — самое быстрое чтение

```csharp
var summaries = await db.Orders
    .AsNoTracking()
    .Where(o => o.Status == "active")
    .Select(o => new { o.Id, o.Total })
    .ToListAsync();
```

`AsNoTracking()` отключает change tracker — данные не запоминаются как entities. Объекты — обычные anonymous-инстансы, не отслеживаются на изменения. Для read-only запросов это самый быстрый путь.

### 14.4. Nested projections и SQL

```csharp
var customerSummaries = await db.Customers
    .Select(c => new
    {
        c.Id,
        c.Name,
        OrderCount = c.Orders.Count(),
        TotalSpent = c.Orders.Sum(o => o.Total),
        RecentOrders = c.Orders
            .OrderByDescending(o => o.Date)
            .Take(3)
            .Select(o => new { o.Id, o.Date, o.Total })
            .ToList()
    })
    .ToListAsync();
```

EF Core 5+ умеет это разворачивать в один SQL-запрос с `JOIN` и подзапросами. Без `Include(...)` и без N+1.

```sql
SELECT [c].[Id], [c].[Name],
    (SELECT COUNT(*) FROM [Orders] AS [o] WHERE [o].[CustomerId] = [c].[Id]) AS [OrderCount],
    (SELECT COALESCE(SUM([o].[Total]), 0) FROM [Orders] AS [o] WHERE [o].[CustomerId] = [c].[Id]) AS [TotalSpent],
    [t].[Id], [t].[Date], [t].[Total]
FROM [Customers] AS [c]
LEFT JOIN (
    SELECT [o0].[Id], [o0].[Date], [o0].[Total], [o0].[CustomerId]
    FROM [Orders] AS [o0]
    WHERE [o0].[CustomerId] IS NOT NULL
    ORDER BY [o0].[Date] DESC
    -- TOP 3 per customer (через ROW_NUMBER)
) AS [t] ...
```

Это мощно — сложные структуры можно строить в LINQ, EF разбирает в эффективный SQL.

### 14.5. Когда не транслируется

Не всё транслируется в SQL. Типичные ловушки:

```csharp
// ❌ Локальный метод в проекции
public int CalcTax(decimal amount) => (int)(amount * 0.2m);

var bad = db.Orders.Select(o => new { o.Id, Tax = CalcTax(o.Total) });
// → InvalidOperationException: метод CalcTax не может быть переведён в SQL
```

Решение — либо вынести логику в SQL-совместимое выражение:

```csharp
var ok = db.Orders.Select(o => new { o.Id, Tax = o.Total * 0.2m });
```

Либо материализовать сначала, потом считать на клиенте:

```csharp
var step1 = db.Orders.Select(o => new { o.Id, o.Total }).ToList();
var step2 = step1.Select(x => new { x.Id, Tax = CalcTax(x.Total) }).ToList();
```

### 14.6. Anonymous vs tuple в EF Core — ещё раз

```csharp
// ✅ Anonymous — гарантированно работает
var a = db.Orders.Select(o => new { o.Id, o.Total }).ToList();

// ⚠️ Tuple — иногда не транслируется
var b = db.Orders.Select(o => (o.Id, o.Total)).ToList();
// На EF Core 7+ обычно работает, но edge cases есть
```

В EF Core всегда **сначала пробуй anonymous**. Tuple — для in-memory сценариев и mid-pipeline проекций после `.AsEnumerable()`.

### 14.7. DTO vs anonymous — когда DTO

Внутри запроса anonymous OK. Если результат нужно **передать наружу метода** — заводи DTO:

```csharp
// ❌ Anonymous нельзя вернуть
public async Task<???> GetSummariesAsync() =>
    await db.Orders.Select(o => new { o.Id, o.Total }).ToListAsync();

// ✅ DTO record — typed, можно возвращать
public record OrderSummary(int Id, decimal Total);

public async Task<List<OrderSummary>> GetSummariesAsync() =>
    await db.Orders
        .Select(o => new OrderSummary(o.Id, o.Total))
        .ToListAsync();
```

EF Core 7+ умеет проецировать в record-конструкторе тоже — SQL выходит тот же, что для anonymous. Так что использование DTO **не** ухудшает производительность.

> [!question]- Интервью: почему `db.Orders.Select(o => new { o.Id, o.Total }).ToList()` лучше, чем `db.Orders.ToList().Select(o => new { o.Id, o.Total })`?
> Первое — projection в SQL: запрос на сервер `SELECT Id, Total`, возвращаются только две колонки. Второе — `ToList()` загружает все колонки всех order-ов с сервера, потом локально проецирует. На больших таблицах разница огромная (по трафику, GC pressure, времени запроса). Always pushdown projection в SQL через LINQ к EF.

---

## 15. Common Pitfalls — с механизмами

### 15.1. Невозможно вернуть typed

```csharp
public ??? GetData() => new { X = 1 };
```

**Механизм:** имя anonymous-типа в IL содержит запрещённые в C# символы (`<>f__AnonymousTypeN`), вписать его в сигнатуру нельзя.

**Фикс:** `record` / `class` для возврата.

### 15.2. Operator `==` — reference, не value

```csharp
var a = new { X = 1 };
var b = new { X = 1 };
a == b;       // False!
a.Equals(b);  // True
```

**Механизм:** компилятор переопределяет `Equals`/`GetHashCode`, но **не оператор `==`**. Оператор работает как для `object` — по reference.

**Фикс:** используй `.Equals()` для value-сравнения. Если нужен `==` — заводи record (там оператор переопределён).

### 15.3. Property order matters

```csharp
var a = new { X = 1, Y = 2 };
var b = new { Y = 2, X = 1 };
a.GetType() == b.GetType();   // False
```

**Механизм:** тип определяется shape (имена + типы + порядок). Изменение порядка — другой тип.

**Фикс:** держи каноничный порядок полей в проектных конвенциях. Если важна семантическая равность — record.

### 15.4. Property promotion — нижний регистр

```csharp
string name = "Alice";
var p = new { name };
p.name;   // ❌ нижний регистр — нарушает PascalCase
```

**Механизм:** при promotion имя свойства = имя переменной. Если переменная `name` — свойство `name`.

**Фикс:** явное имя — `new { Name = name }`.

### 15.5. Razor + anonymous = dynamic = хрупкость

```csharp
return View(new { UserName = "Alice" });
// В Razor: @Model.UserNme — опечатка, RuntimeBinderException
```

**Механизм:** anonymous internal в одной assembly, Razor — в другой. Доступ только через `dynamic`. Опечатки → exception в runtime.

**Фикс:** ViewModel record для всех Razor.

### 15.6. EF Core — методы в проекции

```csharp
var data = db.Orders.Select(o => new { Tax = MyCalc(o.Total) }).ToList();
// ❌ MyCalc не переводится в SQL
```

**Механизм:** EF Core не знает, как ваш C# метод превратить в SQL.

**Фикс:** либо inline-выражение (`o.Total * 0.2m`), либо `AsEnumerable()` перед методом.

### 15.7. JSON deserialization → anonymous = trick

```csharp
var data = JsonSerializer.Deserialize<???>(json);   // что писать?
```

**Механизм:** для generic-параметра нужно имя типа. У anonymous его нет.

**Фикс:** generic-helper с образцом или (правильно) record.

### 15.8. Mutability — full immutable, нет `with`

```csharp
var p = new { Name = "Alice", Age = 30 };
p.Age = 31;   // ❌ нет setter
var p2 = p with { Age = 31 };   // ❌ with только для record
```

**Механизм:** anonymous-properties read-only, оператор `with` только для записей.

**Фикс:** `record`, если нужны модифицированные копии. Или новый литерал: `var p2 = new { p.Name, Age = 31 };`.

### 15.9. Nested + LINQ — может не транслироваться

```csharp
var data = db.Customers.Select(c => new
{
    c.Id,
    Recent = c.Orders.OrderBy(o => o.Date).Take(3).Select(o => new
    {
        Items = o.OrderItems.Select(i => new { i.ProductId, i.Qty })
    })
});
```

**Механизм:** EF Core может не справиться с такой глубиной (зависит от версии).

**Фикс:** разбить на две стадии — server-side проекция в простой shape, потом in-memory сборка.

### 15.10. Сравнение по value, но в reference-контексте

```csharp
var seen = new HashSet<object>();
seen.Add(new { X = 1 });
seen.Contains(new { X = 1 });   // ⚠️ True — но через Equals по value!
```

`HashSet<object>` использует `Equals` (override) и `GetHashCode` (override). Anonymous работает. Но `HashSet<>` с `EqualityComparer` мог бы вести себя иначе.

**Механизм:** `HashSet`/`Dictionary` опираются на `Equals`/`GetHashCode`. У anonymous они по value.

**Фикс:** обычно так и хочется — anonymous как key работает корректно.

> [!question]- Интервью: почему `var a = new { X = 1 }; var b = new { X = 1 }; a == b` вернёт `false`, а `a.Equals(b)` — `true`?
> Anonymous-тип имеет переопределённые `Equals` и `GetHashCode` по value-семантике, но **оператор `==` не переопределён** — он остаётся reference equality для `object`. Это часто ловит джунов: тест `a == b` падает, хотя данные одинаковые. Решение — использовать `.Equals()` или перейти на `record` (там оператор `==` переопределён).

---

## 16. Performance

### 16.1. Heap allocation на каждое создание

Anonymous-объект — **class на куче**. Каждое выражение `new { ... }` — это аллокация:

```csharp
for (int i = 0; i < 1_000_000; i++)
{
    var temp = new { Index = i, Square = i * i };
    // 1 миллион объектов на куче, GC будет работать
}
```

Для горячих циклов это **медленнее**, чем tuple (который struct на стеке):

```csharp
for (int i = 0; i < 1_000_000; i++)
{
    var temp = (Index: i, Square: i * i);
    // Без heap-аллокаций
}
```

### 16.2. Бенчмарк (примерно, BenchmarkDotNet; замерено на .NET 8 — на .NET 10 картина та же по порядку величин)

```
| Method               |     Mean | Allocated |
|--------------------- |---------:|----------:|
| CreateTuple          |  0.91 ns |       0 B |
| CreateAnonymous      |  3.2  ns |      24 B |
| CreateRecord         |  3.4  ns |      24 B |
```

Anonymous и record — близко (оба class). Tuple — на порядок быстрее, без аллокаций.

### 16.3. Reflection penalty

Доступ к anonymous через reflection (если ушёл за границу как `object`) — ещё в 50-100 раз медленнее direct:

```csharp
object obj = new { X = 1 };

// Direct (если бы было typed)
var x = ((typed)obj).X;   // ~1 ns

// Reflection
var x2 = obj.GetType().GetProperty("X")!.GetValue(obj);   // ~100 ns

// dynamic (DLR с кэшем)
dynamic d = obj;
var x3 = d.X;   // ~10 ns при первом вызове, ~1 ns в кэше
```

### 16.4. Когда anonymous OK с точки зрения perf

✅ **OK:**
- LINQ-проекции в EF Core — там за SQL-стороной всё равно сетевой запрос (миллисекунды).
- Logging structured data — раз в секунду / запрос.
- Build-up DTO в API-контроллере — раз на запрос.

⚠️ **Не OK:**
- Hot loop с миллионом итераций.
- Game-loop / parser / hot path в HFT.
- Tight аккумулятор в `Aggregate` — используй tuple.

### 16.5. Generic specialization не помогает

В отличие от tuple-структур, anonymous всегда **class**. Compiler не специализирует под value-types — нет `struct anonymous`. Для perf-критичных мест — tuple или specialized struct.

### 16.6. ToString — heavy

`ToString` для anonymous обходит все свойства, форматирует, конкатенирует. На каждом вызове создаётся `StringBuilder` + final string. Не вызывай `ToString` в горячих логах:

```csharp
// ❌ ToString каждый раз
for (int i = 0; i < 1_000_000; i++)
{
    var data = new { I = i };
    File.AppendAllText("log.txt", data.ToString());   // дорого
}

// ✅ Прямая запись полей
for (int i = 0; i < 1_000_000; i++)
{
    File.AppendAllText("log.txt", $"I={i}\n");
}
```

> [!question]- Интервью: что дороже — создание anonymous или tuple?
> Anonymous — class на куче, требует allocation + GC pressure. Tuple — struct на стеке, без аллокации. На простом `new { X = 1 }` vs `(X: 1)` разница 3-5x по времени и 24 vs 0 байт. На горячем коде (циклы, parsers, hot paths) tuple критически быстрее. На LINQ к EF Core разница не имеет значения — за SQL-запросом всё равно миллисекунды сетевого I/O.

---

## 17. Refactoring anonymous → record

### 17.1. Когда пора переходить

Сигналы:

1. **Передаёшь между методами** через `object` или `dynamic`.
2. **Сериализуешь как часть API** — нужен стабильный контракт.
3. **Тестируешь** — нужно `Assert.Equal(expected, actual)` с typed expected.
4. **Используешь в нескольких местах** — копируешь shape в нескольких файлах.
5. **Пишешь generic helper** «передачи anonymous между методами».

Если поймал хотя бы два — мигрируй на record.

### 17.2. Шаги миграции

**Шаг 1.** Найди shape в проекции:

```csharp
var summaries = orders.Select(o => new
{
    o.Id,
    CustomerName = o.Customer.Name,
    o.Total,
    o.Date
}).ToList();
```

**Шаг 2.** Объяви record с теми же полями (positional record):

```csharp
public record OrderSummary(int Id, string CustomerName, decimal Total, DateTime Date);
```

**Шаг 3.** Замени в проекции:

```csharp
var summaries = orders.Select(o => new OrderSummary(
    o.Id,
    o.Customer.Name,
    o.Total,
    o.Date
)).ToList();
```

**Шаг 4.** Подними тип в сигнатуру метода:

```csharp
public List<OrderSummary> GetSummaries() =>
    orders.Select(o => new OrderSummary(...)).ToList();
```

EF Core 7+ умеет проецировать в record-конструктор так же, как в anonymous. SQL получается тот же.

### 17.3. Тонкости с EF Core projection

EF Core требует, чтобы ВСЕ параметры конструктора record-а были «понятны»:

```csharp
public record OrderSummary(int Id, decimal Total, decimal Tax = 0);
//                                                          ^^^ 
// EF Core может не справиться с default-значениями. Если такая беда — без default.

public record OrderSummary(int Id, decimal Total);   // лучше
```

Либо init-only properties:

```csharp
public record OrderSummary
{
    public int Id { get; init; }
    public decimal Total { get; init; }
    public decimal Tax { get; init; }
}

orders.Select(o => new OrderSummary
{
    Id = o.Id,
    Total = o.Total,
    Tax = o.Total * 0.2m
});
```

### 17.4. Эволюция record — добавление полей

С record-ом можно добавлять поля в публичный API без breaking change для существующих callers (если используешь init и опциональные параметры с default):

```csharp
public record OrderSummary(int Id, decimal Total);

// V2 — добавили Currency, default = "USD" чтобы не сломать callers
public record OrderSummary(int Id, decimal Total, string Currency = "USD");
```

Это невозможно с anonymous — там нет имени, нет управления эволюцией.

### 17.5. Refactoring через IDE

Visual Studio / Rider имеют команду «Extract anonymous type to record». Выделяешь `new { ... }`, IDE предлагает создать record и заменить везде. Это самый быстрый путь.

### 17.6. Тесты — золотая зона для миграции

Anonymous в тестах:

```csharp
[Fact]
public void GetSummary_returns_expected()
{
    var actual = service.GetSummary(1);
    var expected = new { Id = 1, Total = 100m };
    Assert.Equal(expected, actual);
    // Работает только если actual — anonymous of same shape
}
```

С record-ом:

```csharp
public record Summary(int Id, decimal Total);

[Fact]
public void GetSummary_returns_expected()
{
    var actual = service.GetSummary(1);
    Assert.Equal(new Summary(1, 100m), actual);
}
```

Запись reusable между тестами, читается лучше.

> [!question]- Интервью: как мигрировать с anonymous на record без breaking changes?
> Объявить record с теми же полями (PascalCase, тот же порядок). Заменить `new { ... }` на `new RecordName(...)` в исходных проекциях. Поднять тип в сигнатуру метода (вместо `var` → `List<RecordName>`). EF Core 7+ умеет проецировать в record-конструктор, SQL остаётся прежним. После — можно расширять record (init-only fields с default), что невозможно с anonymous.

---

## 18. Best Practices

- **Anonymous внутри метода, record — наружу.** Если результат уезжает за границу метода (return, параметр, поле класса) — заводи record.
- **PascalCase свойства всегда.** Если promotion даёт нижний регистр (`new { name }` от переменной `name`), пиши явно: `new { Name = name }`.
- **Каноничный порядок полей** в проекте — фиксируй и придерживайся, чтобы не получать «два разных типа с теми же данными».
- **Не сравнивай через `==`** — оператор не переопределён, работает по reference. Используй `.Equals()` или мигрируй на record.
- **EF Core projections — anonymous каноничен.** Гарантированно транслируется в SQL. Tuple — для in-memory.
- **Composite keys в `GroupBy`** — anonymous через property promotion: `new { x.A, x.B }`.
- **Structured logging** — anonymous + `{@Foo}` префикс для destructuring в Serilog.
- **Razor / Blazor — нет anonymous.** Только typed ViewModel (record).
- **Public API responses — нет anonymous.** Только record / DTO. OpenAPI и client-SDK генераторам нужно имя типа.
- **Aggregate accumulator** — tuple, не anonymous. На стеке, без heap-аллокаций на каждой итерации.
- **Hot loops — нет anonymous.** Каждый `new { ... }` — heap-аллокация. Tuple или specialized struct.
- **JSON deserialization** — заводи record, не пытайся через generic-helper «приклеить» anonymous.
- **Атрибуты сериализации (`[JsonPropertyName]`, `[JsonIgnore]`)** — невозможны на anonymous, нужен record.
- **Тесты — record для expected.** Anonymous работает, но record-ы лучше пересекают границы методов.
- **Если переписываешь shape в нескольких местах** — это сигнал «пора record».

---

## 19. Decision tree

```
Где используется shape?
│
├── Внутри одного метода (LINQ projection, локальная сборка)
│   ├── EF Core / LINQ to SQL
│   │   └── Anonymous (`new { ... }`)
│   │
│   ├── In-memory LINQ — multi-return
│   │   └── Tuple (`(a, b)`)
│   │
│   ├── In-memory LINQ — projection с named полями
│   │   └── Anonymous (читаемее) или Tuple
│   │
│   └── Aggregate accumulator
│       └── Tuple (без heap allocation)
│
├── Между методами / классами
│   ├── Cross-method, value-семантика
│   │   └── Record
│   │
│   ├── Cross-method, identity / поведение
│   │   └── Class
│   │
│   └── Cross-assembly, public API
│       └── Record / DTO

Свойства мутируются?
│
├── Да
│   └── Class
│
└── Нет (read-only)
    └── Anonymous / Tuple / Record

Нужен `with`-like clone-and-modify?
│
├── Да
│   └── Record
│
└── Нет
    └── Anonymous допустим

Нужно сериализовать в JSON и десериализовать обратно?
│
├── Только сериализация
│   └── Anonymous OK
│
└── И туда, и обратно
    └── Record (десериализация требует имени типа)

Передача в Razor / Blazor?
│
├── Маленький partial / простой
│   └── Anonymous + dynamic (нет typed access)
│
└── Полноценная страница / компонент
    └── ViewModel record / class

Hot path / горячий цикл?
│
├── Да (миллион итераций)
│   └── Tuple / specialized struct
│
└── Нет (по запросу)
    └── Anonymous OK
```

---

## 20. Cheat sheet

```csharp
// Создание
var p = new { Name = "Alice", Age = 30 };
var p2 = new { user.Id, user.Name };          // promotion из полей
var p3 = new { user.Id, FullName = $"{user.First} {user.Last}" };   // mixed

// Доступ
p.Name;                                       // Alice
p.GetType().Name;                             // <>f__AnonymousType0`2

// Equality
p.Equals(new { Name = "Alice", Age = 30 });   // True (value)
p == new { Name = "Alice", Age = 30 };        // False (reference!)

// LINQ projection
orders.Select(o => new { o.Id, o.Total });

// GroupBy composite key
orders.GroupBy(o => new { o.Year, o.Month })
      .Select(g => new { g.Key.Year, g.Key.Month, Total = g.Sum(o => o.Amount) });

// JOIN result
db.Orders.Join(db.Customers,
    o => o.CustomerId, c => c.Id,
    (o, c) => new { o.Id, CustomerName = c.Name });

// Logging structured
_logger.LogInformation("Order: {@Info}", new { OrderId = 1, Total = 100m });

// Cast trick (через template object)
public static T Cast<T>(object obj, T template) => (T)obj;
var typed = Cast(receivedObject, new { Id = 0, Name = "" });

// Refactoring → record
// Было:
var summaries = orders.Select(o => new { o.Id, o.Total }).ToList();
// Стало:
public record OrderSummary(int Id, decimal Total);
var summaries = orders.Select(o => new OrderSummary(o.Id, o.Total)).ToList();
```

| Сценарий | Решение |
|----------|---------|
| LINQ Select projection | `new { ... }` |
| LINQ GroupBy composite | `g => new { g.A, g.B }` |
| LINQ Join projection | `(a, b) => new { ... }` |
| Logging context | `new { ... }` + `{@Foo}` |
| Razor partial (маленький) | anonymous + `@model dynamic` |
| Razor / Blazor полный | ViewModel record / class |
| Quick test data | `new { ... }` локально |
| Cross-method shape | record |
| Multiple values (внутри метода) | tuple |
| Public API response | record / DTO |
| Aggregate accumulator | tuple |
| Hot loop | tuple / struct |

---

## 21. Practice — упражнения с разбором

### 21.1. Top-N по продажам

**Задача.** Написать метод, который возвращает топ-3 продукта по выручке за месяц. Использовать anonymous внутри LINQ, на выходе — record (или эквивалент).

```csharp
record SaleRecord(int ProductId, decimal Amount, DateTime Date);
record TopProduct(int ProductId, decimal Revenue, int OrdersCount);

public List<TopProduct> Top3ByMonth(IEnumerable<SaleRecord> sales, int year, int month)
{
    return sales
        .Where(s => s.Date.Year == year && s.Date.Month == month)
        .GroupBy(s => s.ProductId)
        .Select(g => new   // anonymous внутри
        {
            ProductId = g.Key,
            Revenue = g.Sum(s => s.Amount),
            OrdersCount = g.Count()
        })
        .OrderByDescending(x => x.Revenue)
        .Take(3)
        .Select(x => new TopProduct(x.ProductId, x.Revenue, x.OrdersCount))
        .ToList();
}
```

**Разбор:** anonymous идеально для GroupBy + Select агрегата (имена полей сохраняются, читается). На выходе — record `TopProduct`, потому что метод public, результат уезжает наружу. Anonymous остаётся внутри тела метода.

### 21.2. JOIN customer + adresses

**Задача.** Получить orders с именем клиента и городом доставки. Один метод, anonymous внутри, DTO на выходе.

```csharp
record OrderWithDelivery(int OrderId, string Customer, string City, decimal Total);

public async Task<List<OrderWithDelivery>> GetOrdersWithDeliveryAsync()
{
    return await db.Orders
        .Join(db.Customers, o => o.CustomerId, c => c.Id, (o, c) => new { Order = o, Customer = c })
        .Join(db.Addresses, oc => oc.Customer.AddressId, a => a.Id, (oc, a) => new
        {
            oc.Order.Id,
            CustomerName = oc.Customer.Name,
            City = a.City,
            oc.Order.Total
        })
        .Select(x => new OrderWithDelivery(x.Id, x.CustomerName, x.City, x.Total))
        .ToListAsync();
}
```

**Разбор:** двойной JOIN с промежуточными anonymous (`Order + Customer`, потом плоский результат). Финальный `.Select(x => new OrderWithDelivery(...))` поднимает тип в DTO. EF Core развернёт это в один SQL с двумя JOIN.

### 21.3. Composite key — group by year+month

**Задача.** Подсчитать выручку и количество заказов помесячно, отсортировать по году/месяцу.

```csharp
public List<(int Year, int Month, int Count, decimal Revenue)> MonthlyStats(IEnumerable<SaleRecord> sales)
{
    return sales
        .GroupBy(s => new { s.Date.Year, s.Date.Month })
        .Select(g => (
            Year: g.Key.Year,
            Month: g.Key.Month,
            Count: g.Count(),
            Revenue: g.Sum(s => s.Amount)
        ))
        .OrderBy(x => x.Year)
        .ThenBy(x => x.Month)
        .ToList();
}
```

**Разбор:** anonymous ключ для `GroupBy` (две колонки + value equality), tuple на выходе для деконструкции у callers:

```csharp
foreach (var (year, month, count, revenue) in service.MonthlyStats(sales))
    Console.WriteLine($"{year}-{month:D2}: {count} orders, ${revenue}");
```

Здесь anonymous и tuple работают вместе: anonymous внутри для группировки, tuple на выходе для удобства деконструкции.

### 21.4. Logging structured order

**Задача.** Залогировать создание order с полями: ID, customer, items count, total, duration.

```csharp
public async Task<int> CreateOrderAsync(int customerId, List<int> productIds)
{
    var sw = Stopwatch.StartNew();
    var order = await ProcessAsync(customerId, productIds);
    sw.Stop();

    _logger.LogInformation("Order created: {@OrderInfo}", new
    {
        OrderId = order.Id,
        CustomerId = customerId,
        ItemsCount = productIds.Count,
        Total = order.Total,
        DurationMs = sw.ElapsedMilliseconds
    });

    return order.Id;
}
```

**Разбор:** `{@OrderInfo}` с anonymous даёт structured log с searchable полями. В Seq можно искать `OrderInfo.CustomerId = 42` или строить графики `OrderInfo.DurationMs`. Без `@` анонимный объект логировался бы как `ToString()` — строка `{ OrderId = ..., CustomerId = ... }` без structured-полей.

### 21.5. Refactoring out: anonymous → record (миграция)

**Задача.** Дано:

```csharp
public class ReportService
{
    public List<object> GetTopCustomers()   // ❌ object — теряем типизацию
    {
        return _db.Customers
            .Select(c => new
            {
                c.Id,
                c.Name,
                Revenue = c.Orders.Sum(o => o.Total)
            })
            .OrderByDescending(x => x.Revenue)
            .Take(10)
            .Cast<object>()
            .ToList();
    }
}
```

**Перепиши**, чтобы тип был typed. Рассмотри: какой тип использовать (record или class), почему, как поведение запроса на стороне БД останется прежним.

```csharp
public record TopCustomer(int Id, string Name, decimal Revenue);

public class ReportService
{
    public List<TopCustomer> GetTopCustomers()
    {
        return _db.Customers
            .Select(c => new TopCustomer(
                c.Id,
                c.Name,
                c.Orders.Sum(o => o.Total)
            ))
            .OrderByDescending(x => x.Revenue)
            .Take(10)
            .ToList();
    }
}
```

**Разбор:**

- Выбран **record** — value-семантика (для топ-листа сравнение по содержимому имеет смысл), immutable (нет необходимости менять), public API (возврат из service-метода).
- EF Core 7+ транслирует `new TopCustomer(c.Id, c.Name, c.Orders.Sum(...))` в тот же SQL, что и anonymous-проекция. Производительность не пострадает.
- Caller теперь получает typed `List<TopCustomer>` с автодополнением и compile-time проверками. Не нужен `Cast<object>` или `dynamic`.

---

## 22. Что читать дальше — порядок и почему

1. **[[tuples-deconstruction|Tuples и Deconstruction]]** — близкий родственник, особенно для multi-return и composite keys в in-memory LINQ.
2. **[[modern-features|Modern C# Features]]** — record, record struct, with-выражения, init-only properties.
3. **[[oop|OOP и классы]]** — class vs record, наследование, Equals overrides — для понимания «когда class, когда record, когда anonymous».
4. **[[collections-linq|Collections и LINQ]]** — где anonymous-проекции живут в production-коде.
5. **[[error-handling|Error Handling]]** — `Result<T, E>` как альтернатива «anonymous-возврату» с success/failure.
6. **EF Core Queries и Performance** — projection patterns, AsNoTracking, split queries.
7. **Logging и Observability** — Serilog message templates, destructuring, contextual logging.

---

## 23. См. также

- [[csharp-basics|C# Basics]] — object initializers, var
- [[tuples-deconstruction|Tuples и Deconstruction]] — близкий концепт
- [[modern-features|Modern Features]] — records detailed
- [[oop|OOP]] — class vs record
- [[collections-linq|Collections и LINQ]] — anonymous heavily used
- [[error-handling|Error Handling]] — `Result<T, E>` для cross-method success/failure

---

## 24. Reading list

- **Microsoft Docs — Anonymous Types** — learn.microsoft.com/dotnet/csharp/fundamentals/types/anonymous-types
- **Microsoft Docs — How to use anonymous types** — learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/how-to-return-subsets-of-element-properties-in-a-query
- **Eric Lippert — History of anonymous types** — ericlippert.com (поиск «anonymous type»)
- **Mads Torgersen — C# 3 design notes** — github.com/dotnet/csharplang/discussions
- **EF Core Docs — Client vs Server evaluation** — learn.microsoft.com/ef/core/querying/client-eval
- **EF Core Docs — Loading related data** — learn.microsoft.com/ef/core/querying/related-data
- **Serilog Docs — Structured Data** — github.com/serilog/serilog/wiki/Structured-Data
- **Jon Skeet — C# in Depth (4th ed.)** — chapter «Anonymous types and LINQ»
- **SharpLab** — sharplab.io — посмотреть IL для `new { ... }`, понять что генерирует компилятор
