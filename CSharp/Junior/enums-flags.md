---
tags: [csharp, enums, flags, junior, bitwise, conversion, ef-core]
level: Junior
date: 2026-08-02
---

# Enums и Flags — перечисления и битовые маски

> **Типобезопасные константы вместо magic numbers и строк.** `enum`, `[Flags]`, bitwise операции, конверсии в строки и обратно, EF Core mapping, JSON serialization, ловушки с persistence. Закрывает пробел: «вижу `enum` в коде, не понимаю, чем он лучше `int`, и почему `[Flags]` — это особенный случай».

---

## 0. Как читать этот файл

Если ты впервые видишь `enum` в C# — читай разделы 1→4 подряд: получишь рабочую модель и поймёшь, **зачем не использовать просто `int`**. Если уже пишешь enum'ы, но непонятно про `[Flags]` и bitwise — раздел 6 (Flags), 7 (bitwise операции). Если интересна интеграция (БД, JSON, API) — раздел 11-13. Если строишь production систему — раздел 14 (best practices), 15 (decision tree).

Все примеры самостоятельные. `// expected: ...` показывает ожидаемый вывод. Cross-language якоря (`> [!info]-`) свёрнуты — раскрывай, если переходишь из Java / TypeScript / Rust / Python / Swift. Interview-вопросы (`> [!question]-`) встроены рядом с теорией.

---

## 1. Что это, зачем и когда

### 1.1. Что такое enum

**Enum (enumeration)** — это именованный набор связанных констант, образующих собственный тип. Каждый member имеет имя и числовое значение:

```csharp
public enum OrderStatus
{
    Pending,    // 0
    Paid,       // 1
    Shipped,    // 2
    Delivered,  // 3
    Cancelled   // 4
}

OrderStatus s = OrderStatus.Paid;
Console.WriteLine(s);          // "Paid"
Console.WriteLine((int)s);     // 1
```

Под капотом enum — это просто **value type**, базирующийся на целочисленном типе (по умолчанию `int`). Но C# даёт ему имя, member-имена, и компилятор не позволяет случайно перепутать с обычным `int`.

### 1.2. Зачем enum, когда есть int / string

До enum люди использовали:

```csharp
// Магические числа — нечитаемо, ошибочно
public class Order
{
    public int Status { get; set; }   // 0=pending, 1=paid, 2=...
}

if (order.Status == 1)   // что значит 1?
    SendEmail();

// Магические строки — лучше, но typo не ловится
public class Order
{
    public string Status { get; set; } = "pending";
}

if (order.Status == "Paid")   // typo "paid" vs "Paid" — silent bug
    SendEmail();
```

С enum:

```csharp
public class Order
{
    public OrderStatus Status { get; set; }
}

if (order.Status == OrderStatus.Paid)   // typo поймает компилятор
    SendEmail();
```

Преимущества:

1. **Type safety** — нельзя присвоить произвольный `int` без явного cast.
2. **Самодокументируется** — имя члена говорит больше, чем магическое число.
3. **IntelliSense** — IDE показывает все возможные значения.
4. **Refactoring** — переименовать member один раз, IDE обновит везде.
5. **Switch exhaustiveness** — компилятор может предупредить о missing case.
6. **Performance** — сравнение `int` vs `int`, не string compare.

### 1.3. Главное правило

```
Используй enum для замкнутого набора значений, известного на compile-time.

✅ Order status, payment method, log level, day of week
❌ User ID, country (если в БД таблица countries), произвольные tags
```

Если набор значений **меняется в runtime** (приходят из БД, конфига, пользовательского ввода) — это не enum, это lookup table или строка.

### 1.4. Типы enum

```csharp
// С underlying типом int (по умолчанию)
public enum Color
{
    Red,
    Green,
    Blue
}

// С другим underlying типом
public enum Status : byte    // 0-255, экономия памяти
{
    Active = 1,
    Inactive = 2
}

public enum LargeId : long   // если нужны большие значения
{
    Min = 0L,
    Max = long.MaxValue
}
```

Допустимые underlying типы: `byte`, `sbyte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`. По умолчанию — `int`.

### 1.5. Эволюция: .NET 1.0 → .NET 10

| Версия | Год | Что появилось |
|--------|-----|---------------|
| **.NET 1.0** | 2002 | `enum`, `[Flags]` атрибут, `Enum` базовый класс |
| **.NET 2.0** | 2005 | `Nullable<T>` — `OrderStatus?` поддерживается |
| **.NET 4.0** | 2010 | `Enum.HasFlag`, generic constraints улучшения |
| **C# 7.3** | 2018 | `where T : Enum` constraint |
| **.NET 5** | 2020 | Performance улучшения для enum operations |
| **.NET 7** | 2022 | Generic Math, эффективные enum операции |
| **.NET 8** | 2023 | `JsonStringEnumConverter` улучшения |

### 1.6. Когда что использовать

```
Замкнутый набор статусов / типов / категорий
  → simple enum
  
Набор флагов, которые могут комбинироваться
  → enum с [Flags]
  
Набор + дополнительная логика (display name, behaviour)
  → smart enum (sealed class) или Records
  
Динамический набор из БД
  → lookup table, не enum
  
Integration с external system, у которого свои коды
  → enum + maps в строки/числа external system
```

> [!info]- Если ты знаешь Java / TypeScript / Rust / Python / Swift
> **Java:** `enum` гораздо мощнее C# — может содержать методы, конструкторы, реализовывать interface'ы. С C# 7+ это эмулируется через "smart enum" pattern (sealed class с static instances).
>
> **TypeScript:** `enum` имеет два режима — number-based (как C#) и string-based (`enum X { A = "A", B = "B" }`). Const enum — compile-time only. C# enum похож на number-based TypeScript enum.
>
> **Rust:** `enum` — это algebraic data type (sum type), может содержать ассоциированные данные. C# enum проще — только именованные числа, но C# 9+ records + pattern matching эмулируют похожее.
>
> **Python:** `enum.Enum` (Python 3.4+) — близкая семантика к C# enum. `enum.Flag` — для битовых флагов (как C# `[Flags]`). `enum.IntEnum` — наследник `int` (как C# enum под капотом).
>
> **Swift:** `enum` мощнее C# — может иметь raw values, ассоциированные значения, методы. Для C#-аналога нужна smart enum или sealed class.

> [!question]- Интервью: чем enum лучше string или int constants?
> **Type safety** — компилятор не даст присвоить произвольное значение. **Самодокументирование** — `OrderStatus.Paid` понятнее, чем `1` или `"Paid"`. **IntelliSense** — IDE показывает варианты. **Refactoring безопасен** — переименование одного member обновляет везде. **Performance** — `int` сравнение быстрее string compare. **Switch exhaustiveness** — компилятор видит missing cases. Используй для замкнутых наборов известных на compile-time. Не используй для динамических данных (countries, tags).

---

## 2. Объявление и использование

### 2.1. Базовый синтаксис

```csharp
public enum DayOfWeek
{
    Monday,      // 0
    Tuesday,     // 1
    Wednesday,   // 2
    Thursday,    // 3
    Friday,      // 4
    Saturday,    // 5
    Sunday       // 6
}

DayOfWeek today = DayOfWeek.Wednesday;
Console.WriteLine(today);          // "Wednesday"
Console.WriteLine((int)today);     // 2
```

По умолчанию первый member = 0, каждый следующий +1.

### 2.2. Явные значения

```csharp
public enum HttpStatus
{
    OK = 200,
    Created = 201,
    BadRequest = 400,
    Unauthorized = 401,
    NotFound = 404,
    InternalServerError = 500
}

HttpStatus s = HttpStatus.NotFound;
Console.WriteLine((int)s);   // 404
```

Лучше всегда указывать явные значения — это **стабильный контракт**. Без явных значений добавление member'а в середину сдвинет все последующие, что **breaking change** для serialized данных в БД / wire format.

```csharp
// ❌ Опасно — добавление 'Cancelled' в середину сдвинет 'Shipped' и 'Delivered'
public enum OrderStatus
{
    Pending,     // 0
    Paid,        // 1
    Cancelled,   // 2 — было 'Shipped' (2)!
    Shipped,     // 3 — было 'Delivered' (3)!
    Delivered    // 4 — было 'Cancelled' (4)? 
}

// ✅ Безопасно — порядок объявления не влияет на хранение
public enum OrderStatus
{
    Pending = 1,
    Paid = 2,
    Shipped = 3,
    Delivered = 4,
    Cancelled = 5
}
```

### 2.3. Дублирующие значения

```csharp
public enum Color
{
    Red = 1,
    Crimson = 1,    // ← такое же значение
    Green = 2,
    Blue = 3
}

(int)Color.Red == (int)Color.Crimson;   // true
Color.Red == Color.Crimson;              // true
Console.WriteLine(Color.Red);            // "Red" или "Crimson" — компилятор выберет
```

Иногда полезно для алиасов, но бывает источник путаницы. Избегай без явной причины.

### 2.4. Объявление nested и в namespace

```csharp
namespace MyApp.Domain
{
    // Top-level
    public enum OrderStatus { Pending, Paid }

    public class Order
    {
        // Nested
        public enum Priority { Low, Medium, High }

        public Priority Pri { get; set; }   // OK
    }
}

// Внешнее использование
var p = MyApp.Domain.Order.Priority.High;
```

Nested enum полезен, когда он логически принадлежит одному классу. Top-level — общий.

### 2.5. Namespace вырастает — изолируй enum

В большом проекте лучше группировать enum'ы:

```
MyApp/
├── Domain/
│   ├── Order/
│   │   ├── Order.cs
│   │   ├── OrderStatus.cs       ← один enum = один файл
│   │   └── OrderPriority.cs
│   └── Customer/
│       └── CustomerType.cs
```

Это упрощает поиск и refactoring.

### 2.6. Default value enum'а

```csharp
public enum Status
{
    Active = 1,
    Inactive = 2
}

Status s = default;          // 0 — Status (не существует!)
Status s2 = new Status();    // 0 — то же самое
Console.WriteLine(s);        // "0" — без member-имени
Console.WriteLine((int)s);   // 0
```

`default(Enum)` всегда `0`, **независимо от того, существует ли такой member**. Это типичная ловушка.

**Решение:** определи `0` как смысловой default:

```csharp
public enum Status
{
    Unknown = 0,    // ← default-value
    Active = 1,
    Inactive = 2
}

Status s = default;          // Unknown — теперь имеет смысл
```

Обязательное правило для production enum'ов.

> [!question]- Интервью: что произойдёт, если объявить `enum Status { Active = 1, Inactive = 2 }` и сделать `Status s = default`?
> `s` получит значение `0`, которому **не соответствует** ни один member. `Console.WriteLine(s)` напечатает `"0"`, а не имя. Это типичная ошибка — забыли определить значение для `0`. Best practice: всегда первым member делать `Unknown = 0` или `None = 0`, чтобы default имел смысл. Особенно важно для DB-маппинга, где новая запись с дефолтом получит `0`.

---

## 3. Конверсии — enum ↔ int ↔ string

### 3.1. enum → int

```csharp
OrderStatus s = OrderStatus.Paid;

int i = (int)s;            // explicit cast — OK
int j = s;                 // ❌ CS0266 — нужен явный cast
```

C# требует явный cast для предотвращения случайных ошибок. Это типобезопасность.

### 3.2. int → enum

```csharp
int i = 2;

OrderStatus s = (OrderStatus)i;   // explicit cast
Console.WriteLine(s);              // "Shipped" если 2 = Shipped

// ⚠️ Cast НЕ проверяет валидность
OrderStatus invalid = (OrderStatus)999;
Console.WriteLine(invalid);          // "999" — нет такого member, но cast прошёл
Console.WriteLine((int)invalid);     // 999

Enum.IsDefined(typeof(OrderStatus), invalid);   // false — проверка
```

`Enum.IsDefined` или generic `Enum.IsDefined<T>` (.NET 5+) — единственный способ убедиться, что int соответствует реальному member:

```csharp
if (!Enum.IsDefined(typeof(OrderStatus), i))
    throw new ArgumentException($"Invalid OrderStatus value: {i}");

OrderStatus s = (OrderStatus)i;
```

### 3.3. enum → string

```csharp
OrderStatus s = OrderStatus.Paid;

string str = s.ToString();         // "Paid"
string str2 = $"{s}";              // "Paid"
string str3 = nameof(OrderStatus.Paid);   // "Paid" — compile-time!
```

`nameof()` — лучший способ для compile-time строк. Не зависит от runtime, поддерживает refactoring.

### 3.4. string → enum

```csharp
// Parse — throw если не подошло
OrderStatus s = Enum.Parse<OrderStatus>("Paid");           // .NET 5+
OrderStatus s2 = (OrderStatus)Enum.Parse(typeof(OrderStatus), "Paid");   // legacy

// TryParse — для пользовательского input
if (Enum.TryParse<OrderStatus>("Paid", out var status))
{
    Console.WriteLine(status);   // "Paid"
}

// Case-insensitive
Enum.TryParse<OrderStatus>("paid", ignoreCase: true, out var s3);
```

Используй `TryParse` для пользовательского input — безопасно, не throw.

### 3.5. **Ловушка**: TryParse принимает любой int как string

```csharp
// "999" не имя member, но это валидный int
Enum.TryParse<OrderStatus>("999", out var s);   // returns true!
Console.WriteLine(s);          // "999"
Console.WriteLine((int)s);     // 999

// Защита через IsDefined
if (Enum.TryParse<OrderStatus>(input, out var status) && Enum.IsDefined(status))
{
    // OK — валидный member
}
```

### 3.6. Получить все значения

```csharp
// .NET 5+
OrderStatus[] all = Enum.GetValues<OrderStatus>();
foreach (var s in all)
    Console.WriteLine($"{s} = {(int)s}");

// Старый стиль
foreach (OrderStatus s in Enum.GetValues(typeof(OrderStatus)))
    Console.WriteLine(s);

// Имена
string[] names = Enum.GetNames<OrderStatus>();
```

Полезно для UI dropdown'ов, miграций БД, генерации документации.

### 3.7. Performance — обзор

```csharp
// Самые быстрые
(int)enumValue;             // ноль overhead
nameof(EnumType.Member);    // compile-time
enumA == enumB;             // int compare

// Медленные (reflection)
enumValue.ToString();       // в .NET 5+ optimized, раньше был slow
Enum.Parse<T>(...);          // reflection
Enum.IsDefined(...);        // reflection
Enum.GetValues<T>();        // в .NET 5+ optimized
```

В hot path — кэшируй результаты `Enum.GetValues`, не вызывай в цикле.

> [!question]- Интервью: что произойдёт при `(OrderStatus)999` если в enum нет такого значения?
> Cast пройдёт без ошибки — C# не проверяет валидность при cast int → enum. `s.ToString()` вернёт `"999"`, не member-имя. `(int)s == 999` вернёт `true`. Чтобы проверить валидность — `Enum.IsDefined(typeof(OrderStatus), s)`. Это типичная ловушка при чтении из БД legacy данных, где могут быть стары/удалённые коды. Best practice: всегда валидировать после cast.

---

## 4. Pattern matching и enum

### 4.1. Switch expression

```csharp
public enum OrderStatus { Pending, Paid, Shipped, Delivered, Cancelled }

string Describe(OrderStatus s) => s switch
{
    OrderStatus.Pending => "Waiting for payment",
    OrderStatus.Paid => "Payment received",
    OrderStatus.Shipped => "On the way",
    OrderStatus.Delivered => "Completed",
    OrderStatus.Cancelled => "Cancelled by user",
    _ => "Unknown"
};
```

`switch` expression идеален для enum — компактно, exhaustive (если есть `_`).

### 4.2. Switch без discard — компилятор поможет

С .NET 5+ компилятор может предупредить о non-exhaustive switch:

```csharp
string Describe(OrderStatus s) => s switch
{
    OrderStatus.Pending => "Waiting",
    OrderStatus.Paid => "Paid",
    // warning CS8524: switch expression does not handle all possible values
};
```

Если без `_` забыл case — warning. Ставь `_ =>` или `default =>` для production safety.

### 4.3. Pattern matching combos

```csharp
public enum Priority { Low, Medium, High, Critical }

string GetSlackChannel(Order order) => (order.Status, order.Priority) switch
{
    (OrderStatus.Cancelled, _) => "#cancelled-orders",
    (_, Priority.Critical) => "#urgent-orders",
    (OrderStatus.Paid, Priority.High) => "#paid-orders",
    _ => "#general"
};
```

Tuple patterns — мощная техника для multi-dimensional решений.

### 4.4. Relational patterns с enum

```csharp
public enum LogLevel { Trace = 0, Debug = 1, Info = 2, Warn = 3, Error = 4, Critical = 5 }

bool ShouldLog(LogLevel level, LogLevel minLevel) => level switch
{
    >= LogLevel.Warn => true,    // Warn, Error, Critical всегда логируем
    _ => level >= minLevel
};
```

Relational patterns (`>=`, `<`, etc.) работают для enum'ов с числовыми значениями.

### 4.5. is pattern

```csharp
if (order.Status is OrderStatus.Paid or OrderStatus.Shipped)
{
    // активный заказ
}

if (order.Status is not OrderStatus.Cancelled)
{
    // не отменён
}
```

`or` / `and` / `not` — выразительный синтаксис для нескольких enum-значений.

### 4.6. Switch statement (старый стиль)

```csharp
switch (s)
{
    case OrderStatus.Pending:
    case OrderStatus.Paid:
        return "active";
    case OrderStatus.Shipped:
        return "in transit";
    case OrderStatus.Delivered:
    case OrderStatus.Cancelled:
        return "complete";
    default:
        throw new ArgumentOutOfRangeException();
}
```

Используй switch expression в новом коде. Switch statement — для побочных эффектов (не возврата значения) или legacy.

> [!question]- Интервью: как обработать все возможные значения enum в switch?
> Через switch expression с `_ => default` или `_ => throw`. С .NET 5+ компилятор может предупредить о non-exhaustive switch (CS8524). Это catch-all защищает от появления нового member'а в enum, который не был учтён. Альтернатива — кидать `ArgumentOutOfRangeException` в `_ =>` для fail-fast поведения. Pattern matching с `is OrderStatus.X or OrderStatus.Y` — для группировки нескольких значений в одно условие.

---

## 5. Underlying type и storage

### 5.1. Выбор underlying типа

```csharp
public enum SmallEnum : byte    // 1 байт, 0-255 значений
public enum MediumEnum : short  // 2 байта, до 65535
public enum DefaultEnum         // 4 байта (int), до 2^31
public enum LargeEnum : long    // 8 байт, до 2^63
```

В большинстве случаев `int` (default) подходит. `byte` имеет смысл если:

- В collection миллионы экземпляров (экономия памяти).
- Sent to wire / БД и размер критичен.
- Mapping к C struct из interop.

`long` — редко, только если значений больше 2 миллиардов (нереалистично для enum) или mapping к external system с long IDs.

### 5.2. Размер в memory

```csharp
public enum Status : byte { Active = 1, Inactive = 2 }

Marshal.SizeOf<Status>();   // 1 byte
sizeof(Status);              // 1 (compile-time)

public class Order
{
    public int Id;            // 4 bytes
    public Status Status;     // 1 byte (но padding до 4)
    public DateTime CreatedAt;// 8 bytes
}

// Реальный размер с padding'ом — больше, чем сумма полей
```

Для одного объекта это не важно. Для arrays/lists миллионов элементов — может быть значительно (10x экономия).

### 5.3. Проверка underlying type

```csharp
Type underlying = Enum.GetUnderlyingType(typeof(OrderStatus));
Console.WriteLine(underlying);   // "System.Int32"

// Generic
Type underlying2 = Enum.GetUnderlyingType<OrderStatus>();   // .NET 5+
```

Полезно для serializer'ов, generic кода.

### 5.4. **Ловушка**: смена underlying type — breaking change

Если enum хранится в БД как `int`, а ты решил поменять на `byte`:

```csharp
// До
public enum Status : int { Active = 1, Inactive = 2 }

// После — если в БД int колонка, EF Core не справится прозрачно
public enum Status : byte { Active = 1, Inactive = 2 }
```

Это **breaking change** для:
- БД схемы (нужна миграция).
- Wire format (если enum сериализован как int).
- Inter-service communication.

В public API лучше зафиксировать underlying type заранее и не менять.

### 5.5. Performance: enum vs int

```csharp
// Equivalent IL
status == OrderStatus.Paid;   // → cmp r0, 1 (int compare)
i == 1;                        // → cmp r0, 1
```

Идентично. Никакого overhead.

```csharp
// Аллокация при ToString — была проблемой до .NET 5
status.ToString();             // в .NET 5+ optimized
status.ToString("D");          // числовое — быстрее
status.ToString("G");          // имя — slower (reflection)
```

В .NET 5+ enum.ToString() значительно ускорен — в большинстве случаев hot path не страдает.

### 5.6. Equality и сравнение

```csharp
OrderStatus a = OrderStatus.Paid;
OrderStatus b = OrderStatus.Paid;

a == b;                  // true — int compare
a.Equals(b);             // true
a.CompareTo(b);          // 0

a.GetHashCode() == b.GetHashCode();   // true — int hash
```

Enum реализует `IComparable`, `IEquatable<T>`, `IFormattable`, `IConvertible` через базовый `Enum`. Всё через underlying int.

> [!question]- Интервью: какой underlying type у `enum` по умолчанию?
> `int` (System.Int32) — 4 байта, диапазон около ±2 миллиарда. Можно явно указать через `: byte`/`: short`/`: long`/`: uint` и т.д. Изменение underlying type после первой публикации = breaking change для БД и wire format. Лучше сразу выбирать с запасом — в большинстве случаев `int` оптимален. `byte` стоит брать только если enum хранится в БД таблицах с миллионами записей и память критична.

---

## 6. [Flags] — битовые маски

### 6.1. Зачем Flags

Иногда сущность может иметь **несколько** атрибутов одновременно:

```csharp
// Без [Flags] — нужен список или несколько boolean
public class FileAccess
{
    public bool CanRead { get; set; }
    public bool CanWrite { get; set; }
    public bool CanDelete { get; set; }
}

// С [Flags] — один enum-значение содержит все
[Flags]
public enum FileAccess
{
    None = 0,
    Read = 1,
    Write = 2,
    Delete = 4,
    All = Read | Write | Delete
}

FileAccess access = FileAccess.Read | FileAccess.Write;
```

`[Flags]` enum — это **bitmask**: каждый member занимает один bit, можно комбинировать через bitwise OR.

### 6.2. Правила объявления

```csharp
[Flags]
public enum Permission
{
    None    = 0,        // ✅ Всегда определи 0
    Read    = 1,        // 0001 = 2^0
    Write   = 2,        // 0010 = 2^1
    Delete  = 4,        // 0100 = 2^2
    Execute = 8,        // 1000 = 2^3
    
    // Комбинации (для удобства)
    ReadWrite = Read | Write,   // 0011 = 3
    All = Read | Write | Delete | Execute   // 1111 = 15
}
```

**Обязательные правила:**

1. **Каждый base flag — степень двойки** (1, 2, 4, 8, 16, ...). Иначе bit overlap, не сможешь различить flags.
2. **`None = 0`** — всегда. Default value, ноль bits set.
3. **`[Flags]` атрибут** — компилятор сам не догадается. Без атрибута `ToString()` показывает int, не имена.
4. **Комбинации после base** — `ReadWrite`, `All` объявляются через `|` после base flags.

### 6.3. Использование bit shift для readability

```csharp
[Flags]
public enum Permission
{
    None    = 0,
    Read    = 1 << 0,    // 1
    Write   = 1 << 1,    // 2
    Delete  = 1 << 2,    // 4
    Execute = 1 << 3,    // 8
    Admin   = 1 << 4,    // 16
    Owner   = 1 << 5,    // 32
}
```

`1 << n` = `2^n`. Читается как «n-й bit». Удобно для длинных enum'ов — не надо считать степени двойки.

### 6.4. Установка флагов

```csharp
Permission p = Permission.Read | Permission.Write;
// Binary: 0001 | 0010 = 0011

p |= Permission.Delete;   // добавить flag
// Binary: 0011 | 0100 = 0111
```

OR (`|`) — устанавливает bits. Идемпотентен (повторное OR того же flag ничего не меняет).

### 6.5. Проверка флагов — HasFlag

```csharp
Permission p = Permission.Read | Permission.Write;

p.HasFlag(Permission.Read);    // true
p.HasFlag(Permission.Delete);  // false

// Проверка нескольких — все должны быть установлены
p.HasFlag(Permission.Read | Permission.Write);   // true (оба установлены)
p.HasFlag(Permission.Read | Permission.Delete);  // false (Delete не установлен)
```

`HasFlag` — readable, но **slower** (boxing до .NET 5, потом optimized). В hot path можно через bitwise:

```csharp
(p & Permission.Read) == Permission.Read;   // эквивалентно HasFlag(Read)
```

В .NET 5+ `HasFlag` оптимизирован, можно использовать без оглядки на perf.

### 6.6. Удаление флагов

```csharp
Permission p = Permission.Read | Permission.Write | Permission.Delete;

p &= ~Permission.Delete;   // убрать flag
// p теперь Read | Write
```

`& ~flag` — стандартный паттерн удаления. `~Permission.Delete` инвертирует bits (все bits кроме Delete = 1), AND с этим оставляет всё кроме Delete.

### 6.7. Toggle флага

```csharp
p ^= Permission.Read;   // inverts Read flag
```

XOR — переключает bit. Если был — убирает, не было — добавляет.

### 6.8. Печать комбинации

```csharp
Permission p = Permission.Read | Permission.Write;
Console.WriteLine(p);   // "Read, Write" — благодаря [Flags] атрибуту

// Без [Flags]
public enum NoFlagsPermission
{
    Read = 1,
    Write = 2
}
NoFlagsPermission np = (NoFlagsPermission)3;
Console.WriteLine(np);   // "3" — не имена!
```

`[Flags]` атрибут говорит `ToString()` форматировать как combined.

### 6.9. None vs HasFlag

```csharp
Permission p = Permission.None;

p.HasFlag(Permission.None);   // ⚠️ true — всегда!
```

`HasFlag(None)` всегда `true`, потому что `None = 0`, и `(any & 0) == 0`. Не используй для проверки «нет ли флагов».

```csharp
// Правильная проверка «нет флагов»
p == Permission.None;
```

### 6.10. **Ловушка**: значения не степени двойки

```csharp
[Flags]
public enum BadPermission
{
    None = 0,
    Read = 1,
    Write = 2,
    Delete = 3,      // ❌ ОШИБКА — 3 = Read | Write
    Execute = 4
}

BadPermission p = BadPermission.Delete;
p.HasFlag(BadPermission.Read);    // true! 3 содержит bit 1 (Read)
p.HasFlag(BadPermission.Write);   // true! 3 содержит bit 2 (Write)
```

Если member не степень двойки, он перекрывает другие flags. Bit логика ломается. Всегда `1, 2, 4, 8, 16, ...`.

> [!question]- Интервью: как объявить enum для битовых масок?
> 1) Атрибут `[Flags]` на enum. 2) Каждый base member — степень двойки (1, 2, 4, 8, 16) или `1 << n`. 3) Обязательно `None = 0` для default. 4) Комбинации (`ReadWrite = Read | Write`) — необязательно, но удобно. Установка через `|`, проверка через `HasFlag` или `& flag != 0`, удаление через `& ~flag`. `[Flags]` атрибут влияет на `ToString()` — печатает имена через запятую вместо числа. Без атрибута enum работает как обычно, но представление будет числом.

---

## 7. Bitwise операции

### 7.1. Все операторы

| Оператор | Что делает | Пример |
|----------|-----------|--------|
| `|` | OR — устанавливает bits | `a | b` устанавливает все bits из `a` и `b` |
| `&` | AND — пересечение bits | `a & b` — bits, установленные и в `a`, и в `b` |
| `^` | XOR — exclusive OR | `a ^ b` — bits в одном, но не в другом |
| `~` | NOT — инверсия | `~a` — все bits, кроме установленных в `a` |
| `<<` | Left shift | `1 << 3` = `1000` (8) |
| `>>` | Right shift | `8 >> 1` = `0100` (4) |

### 7.2. Truth table

```
   |  0  1
---+------
 0 |  0  1     ←  OR (|) — true если любой
 1 |  1  1

   |  0  1
---+------
 0 |  0  0     ←  AND (&) — true если оба
 1 |  0  1

   |  0  1
---+------
 0 |  0  1     ←  XOR (^) — true если разные
 1 |  1  0
```

### 7.3. Установка флага

```csharp
permissions = permissions | Permission.Read;
permissions |= Permission.Read;   // shorthand
```

Идемпотентно — повторение не меняет.

### 7.4. Проверка флага

```csharp
// Способ 1 — HasFlag (readable)
if (permissions.HasFlag(Permission.Read))

// Способ 2 — bitwise (fastest, до .NET 5)
if ((permissions & Permission.Read) == Permission.Read)
if ((permissions & Permission.Read) != 0)   // эквивалентно для single flag
```

В .NET 5+ оба варианта одинаковые по perf. Используй `HasFlag` для читаемости.

### 7.5. Удаление флага

```csharp
permissions &= ~Permission.Delete;
```

Стандартный паттерн, без альтернативы. `~Permission.Delete` инвертирует все bits, AND оставляет всё кроме Delete.

### 7.6. Toggle флага

```csharp
permissions ^= Permission.Read;
```

XOR с самим собой = 0, XOR с другим = combined.

### 7.7. Любой из флагов (OR check)

```csharp
// Есть хотя бы один из флагов
if ((permissions & (Permission.Read | Permission.Write)) != 0)
{
    // Read OR Write
}
```

### 7.8. Все из флагов (AND check)

```csharp
var required = Permission.Read | Permission.Write;
if ((permissions & required) == required)
{
    // Read AND Write
}

// Эквивалент через HasFlag
if (permissions.HasFlag(required))
```

### 7.9. Очистить все флаги

```csharp
permissions = Permission.None;
permissions &= 0;   // тоже очищает (в зависимости от underlying type)
```

### 7.10. Использовать Generic Math (.NET 7+)

```csharp
// Можно писать generic helper
public static bool HasAnyFlag<T>(T value, T flags) where T : Enum
{
    var v = Convert.ToUInt64(value);
    var f = Convert.ToUInt64(flags);
    return (v & f) != 0;
}

// .NET 7+ через Generic Math
public static T AddFlag<T>(T value, T flag) where T : struct, Enum, IBitwiseOperators<T, T, T>
{
    return value | flag;
}
```

В большинстве случаев конкретный enum + bitwise операции проще и быстрее.

> [!question]- Интервью: чем отличается `permissions | Permission.Read` от `permissions & Permission.Read`?
> `|` (OR) **устанавливает** flag — добавляет bit в bitmask, не трогая остальные. Используется для добавления разрешения. `&` (AND) **проверяет** flag — возвращает только те bits, которые установлены и в `permissions`, и в `Permission.Read`. Если результат `!= 0` (для single flag) или `== flag` (для multiple), значит flag установлен. Удаление flag через `& ~flag` — AND с инверсией оставляет всё кроме `flag`.

---

## 8. Smart enum pattern

### 8.1. Зачем smart enum

Стандартный C# enum — это просто числа с именами. Не может содержать поведение, дополнительные данные, валидацию. Для сложных доменных понятий это ограничение.

Пример: `OrderStatus` хочешь дополнить:
- Display name (для UI на разных языках).
- Список разрешённых переходов (Pending → Paid OK, Delivered → Pending — нет).
- Цвет для UI.

С обычным enum — отдельные dictionary / extension methods. С smart enum — всё в одном месте.

### 8.2. Базовая реализация

```csharp
public sealed class OrderStatus : IEquatable<OrderStatus>
{
    public static readonly OrderStatus Pending = new(1, "Pending", "Waiting for payment");
    public static readonly OrderStatus Paid = new(2, "Paid", "Payment received");
    public static readonly OrderStatus Shipped = new(3, "Shipped", "On the way");
    public static readonly OrderStatus Delivered = new(4, "Delivered", "Completed");
    public static readonly OrderStatus Cancelled = new(5, "Cancelled", "Cancelled");

    public int Id { get; }
    public string Name { get; }
    public string DisplayName { get; }

    private OrderStatus(int id, string name, string displayName)
    {
        Id = id;
        Name = name;
        DisplayName = displayName;
    }

    public static IReadOnlyList<OrderStatus> All { get; } = [Pending, Paid, Shipped, Delivered, Cancelled];

    public static OrderStatus FromId(int id) =>
        All.FirstOrDefault(s => s.Id == id)
        ?? throw new ArgumentException($"Unknown OrderStatus id: {id}");

    public bool Equals(OrderStatus? other) => other is not null && Id == other.Id;
    public override bool Equals(object? obj) => obj is OrderStatus s && Equals(s);
    public override int GetHashCode() => Id;
    public override string ToString() => Name;
}
```

Использование:

```csharp
OrderStatus s = OrderStatus.Paid;
Console.WriteLine(s);              // "Paid"
Console.WriteLine(s.DisplayName);  // "Payment received"

// Из БД (по int)
var status = OrderStatus.FromId(2);   // Paid
```

### 8.3. С методами поведения

```csharp
public sealed class OrderStatus
{
    public static readonly OrderStatus Pending = new(1, "Pending", canTransitionTo: [2, 5]);    // → Paid, Cancelled
    public static readonly OrderStatus Paid = new(2, "Paid", canTransitionTo: [3, 5]);          // → Shipped, Cancelled
    public static readonly OrderStatus Shipped = new(3, "Shipped", canTransitionTo: [4]);       // → Delivered
    public static readonly OrderStatus Delivered = new(4, "Delivered", canTransitionTo: []);    // финальный
    public static readonly OrderStatus Cancelled = new(5, "Cancelled", canTransitionTo: []);    // финальный

    public int Id { get; }
    public string Name { get; }
    private readonly int[] _canTransitionTo;

    private OrderStatus(int id, string name, int[] canTransitionTo)
    {
        Id = id;
        Name = name;
        _canTransitionTo = canTransitionTo;
    }

    public bool CanTransitionTo(OrderStatus target) =>
        _canTransitionTo.Contains(target.Id);
}

// Использование
var current = OrderStatus.Pending;
var next = OrderStatus.Paid;

if (current.CanTransitionTo(next))
{
    // OK to transition
}
```

Логика валидации переходов — в самом enum'е, не разбросана по сервисам.

### 8.4. С generic базовым классом

Для нескольких smart enum'ов в проекте можно сделать базовый класс:

```csharp
public abstract class Enumeration<TEnum, TKey> : IEquatable<Enumeration<TEnum, TKey>>
    where TEnum : Enumeration<TEnum, TKey>
    where TKey : IEquatable<TKey>
{
    public TKey Id { get; }
    public string Name { get; }

    protected Enumeration(TKey id, string name) { Id = id; Name = name; }

    public bool Equals(Enumeration<TEnum, TKey>? other) =>
        other is not null && Id.Equals(other.Id);

    public override bool Equals(object? obj) =>
        obj is Enumeration<TEnum, TKey> e && Equals(e);

    public override int GetHashCode() => Id.GetHashCode();
    public override string ToString() => Name;
}

public sealed class OrderStatus : Enumeration<OrderStatus, int>
{
    public static readonly OrderStatus Pending = new(1, "Pending");
    public static readonly OrderStatus Paid = new(2, "Paid");

    private OrderStatus(int id, string name) : base(id, name) { }
}
```

Стандартный паттерн в DDD-проектах.

### 8.5. Когда smart enum

✅ **Используй когда:**

- Доменное понятие имеет дополнительные атрибуты (display name, color, behaviour).
- Нужна валидация переходов (state machine).
- Сложная логика, которая иначе разрослась бы по extension methods.

❌ **Не используй когда:**

- Простой enum для UI dropdown — overkill.
- Performance-критичный код (smart enum — class на heap, обычный enum — value type).
- Тестирование сравнения через bit operations.

### 8.6. Известные библиотеки

- **Ardalis.SmartEnum** — популярная NuGet-библиотека.
- **Vogen** — source generator для type-safe value objects, частично покрывает smart enum use case.

### 8.7. Сравнение с обычным enum

| | Обычный enum | Smart enum |
|---|---|---|
| Type safety | ✅ | ✅ |
| Performance | ✅ value type | ❌ class on heap |
| Pattern matching | ✅ exhaustive | ❌ нет (но через type pattern) |
| Дополнительные поля | ❌ | ✅ |
| Methods | ❌ (только extensions) | ✅ |
| EF Core mapping | Простой | Сложнее (через Conversion) |
| JSON serialization | Встроенно | Нужен converter |
| Switch exhaustiveness | ✅ | ❌ |

> [!question]- Интервью: что такое smart enum и когда его использовать?
> Smart enum — это `sealed class` с `static readonly` instances, эмулирующий enum, но с возможностью содержать поля, методы и сложную логику. Используется когда домен требует, например, display name на разных языках, валидацию переходов между статусами (state machine), цвет для UI. Преимущества: вся логика в одном месте, не разбросана по extension'ам. Недостатки: class на heap (vs enum как value type), нет встроенной поддержки в EF Core / JSON (нужны converter'ы), нет switch exhaustiveness. Используй для сложных доменных enum'ов, обычный enum — для простых наборов.

---

## 9. Enum как ключ — Dictionary, HashSet

### 9.1. Enum как Dictionary key

```csharp
public enum LogLevel { Trace, Debug, Info, Warn, Error }

var counters = new Dictionary<LogLevel, int>
{
    [LogLevel.Info] = 0,
    [LogLevel.Warn] = 0,
    [LogLevel.Error] = 0
};

counters[LogLevel.Info]++;
```

Работает out-of-box — `enum` имеет `Equals` и `GetHashCode` через underlying int. Идентично использованию int как ключа.

### 9.2. Performance

Dictionary с enum key — почти как с int (одинаковый hash, одинаковое сравнение). Реальная разница незначительна.

В .NET 5+ для enum hash code и equality нет boxing. До .NET 5 был — потенциальная perf проблема в hot path.

### 9.3. EnumDictionary alternative

Если enum имеет малое количество member'ов (5-20), можно использовать array вместо Dictionary для максимальной скорости:

```csharp
public enum Priority { Low, Medium, High, Critical }   // 4 value

int[] counts = new int[4];   // index = (int)priority

counts[(int)Priority.High]++;
int total = counts.Sum();
```

Trade-off: больше памяти, если enum sparse (значения 0, 100, 200 — 200 ячеек), но O(1) и без hash.

### 9.4. HashSet с enum

```csharp
var allowedTypes = new HashSet<MediaType>
{
    MediaType.Image,
    MediaType.Video
};

if (allowedTypes.Contains(file.Type))
    Process(file);
```

То же что и Dictionary — работает естественно.

### 9.5. EnumSet через [Flags] enum — альтернатива HashSet

Для маленьких наборов flag-комбинаций бывает быстрее использовать `[Flags]` enum как «set»:

```csharp
[Flags]
public enum AllowedFeature
{
    None = 0,
    Search = 1,
    Filter = 2,
    Sort = 4,
    Export = 8
}

AllowedFeature allowed = AllowedFeature.Search | AllowedFeature.Filter;

// Эквивалент HashSet.Contains
bool canSearch = allowed.HasFlag(AllowedFeature.Search);
```

Это compact (один int вместо HashSet allocation), но ограничено числом member'ов (32 для int, 64 для long).

> [!question]- Интервью: какие perf-преимущества у enum как Dictionary key?
> До .NET 5 enum как key страдал от boxing на каждом hash/equality lookup, что было медленнее int. С .NET 5+ это исправлено — perf эквивалентен int. Для маленьких enum'ов (5-20 member'ов) можно использовать array `int[]` или `T[]` с index = `(int)enumValue` — O(1), no hashing, no allocations. Для маленьких flag-наборов — `[Flags]` enum работает как compact set (один int вместо HashSet с allocation).

---

## 10. Enum в коллекциях

### 10.1. Сериализация в JSON массив

```csharp
public class User
{
    public int Id { get; set; }
    public List<UserRole> Roles { get; set; } = [];
}

public enum UserRole { Admin, Editor, Viewer }

// JSON
// { "id": 1, "roles": ["Admin", "Editor"] }
```

С `JsonStringEnumConverter` массив enum'ов сериализуется как массив строк (см. раздел 13 JSON).

### 10.2. Many-to-many связи

В БД:

```csharp
public class User
{
    public int Id { get; set; }
    public List<UserRoleAssignment> Roles { get; set; } = [];
}

public class UserRoleAssignment
{
    public int UserId { get; set; }
    public UserRole Role { get; set; }   // enum, маппится в int или string
}
```

EF Core автоматически справляется — каждый Role хранится отдельной строкой в junction table.

### 10.3. Bitmap permissions через [Flags]

Альтернатива many-to-many для simple permissions:

```csharp
public class User
{
    public int Id { get; set; }
    public Permissions Perms { get; set; }   // один int
}

[Flags]
public enum Permissions { None=0, Read=1, Write=2, Delete=4, Admin=8 }
```

Trade-off:
- ✅ Compact (один int вместо junction table).
- ✅ Быстрая проверка (bit AND).
- ❌ Ограничено 64 flags (long).
- ❌ Нельзя добавлять/удалять member'ы между деплоями без миграции данных.
- ❌ Сложнее для отчётов (запросы по подмножеству permissions).

Для < 20 permissions, не меняющихся часто — годная техника.

### 10.4. Entity-attribute pattern

Если permissions могут меняться часто или их много — обычный many-to-many лучше:

```csharp
public class Permission
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Description { get; set; } = "";
}

public class User
{
    public List<Permission> Permissions { get; set; } = [];
}
```

> [!question]- Интервью: когда использовать `[Flags]` enum вместо many-to-many таблицы?
> `[Flags]` подходит для **малого** (< 20-30) **стабильного** набора флагов, которые часто проверяются в hot path. Преимущества: compact (один int), быстрая проверка через bitwise. Недостатки: ограничение 64 flag'а (long), сложно добавлять/удалять, неудобно для отчётов. Many-to-many таблица лучше для динамических наборов permissions, где admin может создавать новые роли без deploy. Compromise: hybrid — core permissions как `[Flags]`, custom — через таблицу.

---

## 11. EF Core mapping

### 11.1. Default mapping — int

По умолчанию EF Core мапит enum в **int** колонку:

```csharp
public class Order
{
    public int Id { get; set; }
    public OrderStatus Status { get; set; }   // → int колонка
}

public enum OrderStatus { Pending = 1, Paid = 2, Shipped = 3 }
```

SQL:
```sql
CREATE TABLE Orders (
    Id INT PRIMARY KEY,
    Status INT NOT NULL   -- 1, 2, 3
);
```

Самый эффективный — int compare быстрый, маленький размер, indexed.

### 11.2. Mapping в string

Для читаемости в БД:

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>();   // → "Pending", "Paid", "Shipped"
```

SQL:
```sql
CREATE TABLE Orders (
    Status NVARCHAR(20) NOT NULL   -- "Paid"
);
```

✅ **Преимущества string mapping:**
- Читаемо при ручном просмотре БД.
- Защищено от breaking change при сдвиге enum-значений.

❌ **Недостатки:**
- Больше места.
- Slower compare (string vs int).
- Опасно при опечатке member-имени (хотя C# не даст).

### 11.3. Convert через `HasConversion<T, T>`

Кастомные converter'ы:

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion(
        v => v.ToString(),                    // в БД (как string)
        v => Enum.Parse<OrderStatus>(v)       // из БД
    );
```

Эквивалент `HasConversion<string>()`, но даёт контроль (например, lower-case, custom mapping).

### 11.4. Value Converter для enum

Для всех enum'ов проекта — base class:

```csharp
public class EnumStringConverter<T> : ValueConverter<T, string> where T : struct, Enum
{
    public EnumStringConverter()
        : base(
            v => v.ToString(),
            v => Enum.Parse<T>(v))
    { }
}

// Использование
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion(new EnumStringConverter<OrderStatus>());
```

Reusable, type-safe.

### 11.5. Index по enum-колонке

```csharp
modelBuilder.Entity<Order>()
    .HasIndex(o => o.Status);
```

Index на int-колонке — как обычный numeric index. На string — лексикографический. Оба работают.

Для частых фильтров по статусу:

```csharp
db.Orders.Where(o => o.Status == OrderStatus.Paid).ToList();
```

EF Core генерирует:
```sql
SELECT * FROM Orders WHERE Status = 2   -- если int
SELECT * FROM Orders WHERE Status = 'Paid'   -- если string
```

Index ускоряет оба.

### 11.6. **Ловушка**: переименование member ломает string mapping

```csharp
// V1
public enum Status { Pending, Paid, Shipped }
// БД содержит: "Pending", "Paid", "Shipped"

// V2 — переименовали Paid → Confirmed
public enum Status { Pending, Confirmed, Shipped }
// БД всё ещё содержит: "Pending", "Paid", "Shipped"
// При чтении: Enum.Parse<Status>("Paid") → Exception!
```

При string mapping переименование member = breaking change для существующих БД-данных. Решения:

1. **Не переименовывать** — добавлять новый member, deprecate старый.
2. **Миграция данных** — `UPDATE Orders SET Status = 'Confirmed' WHERE Status = 'Paid';`
3. **Custom converter с aliases**:

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion(
        v => v == OrderStatus.Confirmed ? "Paid" : v.ToString(),   // legacy compatibility
        v => v == "Paid" ? OrderStatus.Confirmed : Enum.Parse<OrderStatus>(v)
    );
```

С int mapping та же проблема — переименование не страшно (значения те же), но **изменение значений** breaking.

### 11.7. NodaTime / Smart Enum mapping

Smart enum (раздел 8) — `class`, не enum. EF Core маппит через:

```csharp
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion(
        v => v.Id,                    // в БД (int)
        v => OrderStatus.FromId(v)    // из БД
    );
```

Дальше всё как с обычным enum'ом.

### 11.8. Filter по enum в queries

```csharp
// IN clause
var activeStatuses = new[] { OrderStatus.Pending, OrderStatus.Paid };
var activeOrders = await db.Orders
    .Where(o => activeStatuses.Contains(o.Status))
    .ToListAsync();
// SQL: WHERE Status IN (1, 2)

// Range
var enumNumeric = (int)OrderStatus.Shipped;
var ordersAfterShipped = await db.Orders
    .Where(o => (int)o.Status >= enumNumeric)
    .ToListAsync();
```

EF Core корректно транслирует.

> [!question]- Интервью: int vs string mapping для enum в БД?
> **int** — быстрее (numeric compare, smaller size), стандартное поведение EF Core, защищено от опечаток. Недостатки: непонятно при ручном просмотре БД (`Status = 2` — что это?). **string** — читаемо для DBA, но slower и больше размер. Главная ловушка string: переименование member = breaking change для legacy данных. Решение: использовать `HasConversion<string>()` только если читаемость критична для ops, иначе int. Если переходить — миграция данных через UPDATE или custom converter с aliases.

---

## 12. JSON serialization

### 12.1. Default — int

System.Text.Json по умолчанию сериализует enum как **int**:

```csharp
public class Order
{
    public int Id { get; set; }
    public OrderStatus Status { get; set; }
}

JsonSerializer.Serialize(new Order { Id = 1, Status = OrderStatus.Paid });
// {"Id":1,"Status":2}
```

Компактно, но непонятно. Внешний клиент должен знать mapping `2 = Paid`.

### 12.2. JsonStringEnumConverter — как строки

```csharp
var options = new JsonSerializerOptions
{
    Converters = { new JsonStringEnumConverter() }
};

JsonSerializer.Serialize(new Order { Status = OrderStatus.Paid }, options);
// {"Id":1,"Status":"Paid"}
```

Лучше для public API — клиент видит читаемые значения.

### 12.3. Per-property converter

```csharp
public class Order
{
    public int Id { get; set; }

    [JsonConverter(typeof(JsonStringEnumConverter))]
    public OrderStatus Status { get; set; }
}
```

Атрибут на одном свойстве — глобальные options не нужны.

### 12.4. Naming policies

```csharp
var options = new JsonSerializerOptions
{
    Converters = { new JsonStringEnumConverter(JsonNamingPolicy.CamelCase) }
};

JsonSerializer.Serialize(new Order { Status = OrderStatus.PartiallyShipped }, options);
// {"status":"partiallyShipped"}
```

Naming policies (CamelCase, SnakeCaseLower и т.д.) применяются и к property names, и к enum-значениям.

### 12.5. Кастомный converter

```csharp
public class LowerCaseEnumConverter<T> : JsonConverter<T> where T : struct, Enum
{
    public override T Read(ref Utf8JsonReader reader, Type type, JsonSerializerOptions opts)
    {
        var s = reader.GetString();
        return Enum.Parse<T>(s!, ignoreCase: true);
    }

    public override void Write(Utf8JsonWriter writer, T value, JsonSerializerOptions opts)
    {
        writer.WriteStringValue(value.ToString().ToLowerInvariant());
    }
}

// Использование
[JsonConverter(typeof(LowerCaseEnumConverter<OrderStatus>))]
public OrderStatus Status { get; set; }
// "paid", "shipped", ...
```

Полный контроль над форматом.

### 12.6. Newtonsoft.Json

```csharp
using Newtonsoft.Json;
using Newtonsoft.Json.Converters;

JsonConvert.SerializeObject(order, new JsonSerializerSettings
{
    Converters = { new StringEnumConverter() }
});
// {"Id":1,"Status":"Paid"}

// Per-property
public class Order
{
    [JsonConverter(typeof(StringEnumConverter))]
    public OrderStatus Status { get; set; }
}
```

Functional эквивалент.

### 12.7. Flags enum в JSON

```csharp
[Flags]
public enum Permission { None=0, Read=1, Write=2, Delete=4 }

var p = Permission.Read | Permission.Write;
JsonSerializer.Serialize(p, new JsonSerializerOptions { Converters = { new JsonStringEnumConverter() } });
// "Read, Write" — comma-separated combined string
```

Деsearilization этого формата работает обратно. Но чаще для flags используют **массив**:

```csharp
public class User
{
    public List<Permission> Permissions { get; set; } = [];   // массив enum'ов
}

// JSON: { "permissions": ["Read", "Write"] }
```

Это обычный List, не [Flags]. Более читаемо в API.

### 12.8. OpenAPI / Swagger schema

`JsonStringEnumConverter` влияет на OpenAPI schema:

```yaml
# Без converter — int с описанием
OrderStatus:
  type: integer
  enum: [1, 2, 3, 4, 5]

# С converter — string enum
OrderStatus:
  type: string
  enum: [Pending, Paid, Shipped, Delivered, Cancelled]
```

String версия лучше для:
- Клиент-генераторы (NSwag, OpenAPI Generator).
- Документации.
- Тестирования через Postman / Swagger UI.

### 12.9. Versioning enum в API

Когда добавляешь новый member в API enum:

```csharp
public enum OrderStatus
{
    Pending, Paid, Shipped, Delivered, Cancelled,
    Returned   // ← новый в V2
}
```

Старые клиенты получат `"Returned"` в JSON и не смогут парсить — `JsonException`. Решения:

1. **Default value на клиенте** — если получили unknown member, mapping в "Unknown".
2. **Tolerant deserialization** — `JsonStringEnumConverter` имеет опцию для unknown:

```csharp
new JsonStringEnumConverter(allowIntegerValues: true)
```

3. **Сначала добавить на сервер**, потом клиенты обновляются — старые продолжают работать с известными им values.

> [!question]- Интервью: чем отличается int и string serialization для enum в JSON?
> **int** — компактнее, но требует от клиента знать mapping (2 = Paid). Не self-documenting. Легко ошибиться при изменении enum-значений. **string** — читаемо в API, self-documenting, OpenAPI генерирует понятную schema. Недостатки: чуть больше размер, требует case-insensitive parsing для tolerant API. Best practice для public API: `JsonStringEnumConverter` (string), для internal high-volume APIs (микросервисы) — int OK для compactness.

---

## 13. Validation и API contracts

### 13.1. Валидация enum в input

```csharp
[HttpPost]
public IActionResult UpdateStatus(int orderId, [FromBody] UpdateStatusRequest req)
{
    if (!Enum.IsDefined(req.Status))
        return BadRequest($"Invalid OrderStatus: {req.Status}");

    // ...
}

public class UpdateStatusRequest
{
    public OrderStatus Status { get; set; }
}
```

`Enum.IsDefined<T>` (.NET 5+) — проверяет, что значение соответствует реальному member.

### 13.2. Data Annotations

```csharp
public class UpdateStatusRequest
{
    [Required]
    [EnumDataType(typeof(OrderStatus))]
    public OrderStatus Status { get; set; }
}
```

`EnumDataType` атрибут проверяет валидность при ASP.NET Core model binding.

### 13.3. FluentValidation

```csharp
public class UpdateStatusRequestValidator : AbstractValidator<UpdateStatusRequest>
{
    public UpdateStatusRequestValidator()
    {
        RuleFor(x => x.Status).IsInEnum()
            .WithMessage("Invalid OrderStatus value");
    }
}
```

`IsInEnum` — встроенный rule для проверки.

### 13.4. **Ловушка** [Flags] enum валидация

```csharp
[Flags]
public enum Permission { None=0, Read=1, Write=2, Delete=4 }

Enum.IsDefined<Permission>(Permission.Read | Permission.Write);   // false!
// Combined value 3 не определён как member, хотя валидный для Flags
```

`Enum.IsDefined` для [Flags] enum только проверяет single-member values. Для combined нужна кастомная валидация:

```csharp
public static bool IsValidFlagsCombination<T>(T value) where T : struct, Enum
{
    long allFlags = 0;
    foreach (T flag in Enum.GetValues<T>())
        allFlags |= Convert.ToInt64(flag);

    long val = Convert.ToInt64(value);
    return (val & ~allFlags) == 0;   // нет bits вне определённых
}

IsValidFlagsCombination(Permission.Read | Permission.Write);   // true
IsValidFlagsCombination((Permission)999);                        // false
```

### 13.5. Версионирование enum в public API

Добавление новых member'ов — backward compatible:

```csharp
// V1
public enum OrderStatus { Pending, Paid, Shipped }

// V2 — добавили Cancelled
public enum OrderStatus { Pending, Paid, Shipped, Cancelled }
// Старые клиенты не отправят Cancelled, всё работает
```

Удаление / переименование — **breaking change**. Требует API versioning или backward-compat aliases.

```csharp
// Старая API V1: { "status": "Confirmed" }
// Новая V2: { "status": "Paid" }
// Решение — accept оба варианта
public class StatusConverter : JsonConverter<OrderStatus>
{
    public override OrderStatus Read(...)
    {
        var s = reader.GetString();
        return s switch
        {
            "Confirmed" => OrderStatus.Paid,   // legacy alias
            _ => Enum.Parse<OrderStatus>(s!, true)
        };
    }

    public override void Write(...)
    {
        writer.WriteStringValue(value.ToString());
    }
}
```

> [!question]- Интервью: как валидировать enum input в API?
> 1) `Enum.IsDefined<T>(value)` — проверяет валидность member для обычных enum (для [Flags] не работает с combined). 2) `[EnumDataType(typeof(T))]` атрибут на DTO — ASP.NET Core валидирует автоматически. 3) FluentValidation `RuleFor(x => x.Status).IsInEnum()`. 4) Для [Flags] — кастомная валидация через bitmask `(value & ~allFlags) == 0`. Защищает от отправки клиентом произвольных int / unknown member'ов через JSON.

---

## 14. Best Practices

### 14.1. Объявление

- ✅ **Всегда `None = 0`** или `Unknown = 0` — default value должен быть осмысленным.
- ✅ **Явные значения** для production enum (`Active = 1, Inactive = 2`) — защита от breaking change при изменении порядка.
- ✅ **`[Flags]` атрибут** для bitmask enum — иначе `ToString()` не показывает имена.
- ✅ **Степени двойки для Flags** — `1, 2, 4, 8, 16` или `1 << n`.
- ✅ **Underlying type явно**, если не int — для clarity (`enum Color : byte { ... }`).
- ❌ Не дублируй значения без явной причины.
- ❌ Не меняй существующие значения после deploy — breaking change для БД и wire format.

### 14.2. Использование

- ✅ **`(int)enumValue`** для cast в число — explicit, безопасно.
- ✅ **`Enum.IsDefined<T>(value)`** после cast int → enum (для не-Flags).
- ✅ **`nameof()`** для compile-time строк — refactoring-safe.
- ✅ **`Enum.TryParse<T>` + `IsDefined`** для parsing user input (защита от int-as-string).
- ✅ **Switch expression** с `_ =>` для всех случаев.
- ❌ Не используй `Convert.ChangeType` для enum — медленно.
- ❌ Не используй `Enum.Parse` для пользовательского input без `TryParse`.

### 14.3. [Flags]

- ✅ **Все base members — степени двойки**.
- ✅ **`None = 0`** обязательно.
- ✅ **`HasFlag` или `& flag != 0`** для проверки (одинаково в .NET 5+).
- ✅ **`& ~flag`** для удаления.
- ❌ Не используй `HasFlag(None)` — всегда true.
- ❌ Не превышай 64 bit (long) — это limit.
- ❌ Не клади combined значения как base members — `Delete = 3` ломает bit logic.

### 14.4. Persistence

- ✅ **Default — int mapping** — быстрее, надёжнее.
- ✅ **String mapping только** если читаемость в БД критична для ops.
- ✅ **Index на enum-колонку**, если фильтрация частая.
- ✅ **Index в БД** для частых WHERE clauses.
- ✅ **Migration scripts** при переименовании member'ов.
- ❌ Не меняй underlying type после deploy.
- ❌ Не полагайся на default IDs `0, 1, 2, ...` — задавай явно.

### 14.5. JSON / API

- ✅ **`JsonStringEnumConverter`** для public API.
- ✅ **`JsonNamingPolicy.CamelCase`** или другая стандартная.
- ✅ **OpenAPI schema** через String.
- ✅ **Tolerant deserialization** на клиенте (unknown → default).
- ❌ Не менять existing member'ов в public API — добавлять новые.

### 14.6. Smart enum (если нужен)

- ✅ Используй для domain-rich сценариев (state transitions, display names).
- ✅ Через **Ardalis.SmartEnum** — стандарт в DDD проектах.
- ❌ Не используй для простых enum — overkill.

### 14.7. Performance

- ✅ Cast `(int)enumValue` — zero overhead.
- ✅ `nameof()` — compile-time.
- ✅ Bitwise операции — быстрее, чем `HasFlag` до .NET 5 (после — одинаково).
- ❌ Не вызывай `Enum.GetValues` в hot loop — кэшируй.
- ❌ Не используй `Enum.ToString()` миллионы раз — slow до .NET 5.

---

## 15. Decision tree

```
Что нужно?
│
├── Замкнутый набор статусов / категорий
│   ├── Простые → enum
│   ├── С display name / behaviour → smart enum
│   └── Динамический набор (БД) → не enum, lookup table
│
├── Несколько атрибутов одновременно
│   ├── Compact (< 30 flags, стабильный) → [Flags] enum
│   └── Динамический → many-to-many table
│
├── Хранение в БД
│   ├── Default → int mapping
│   ├── Нужна читаемость → string mapping (HasConversion<string>)
│   └── Smart enum → HasConversion(v => v.Id, v => Type.FromId(v))
│
├── JSON / API contract
│   ├── Public API → JsonStringEnumConverter
│   ├── Internal high-volume → int default
│   └── Tolerant к unknown → custom converter с fallback
│
├── Валидация input
│   ├── Простой enum → Enum.IsDefined
│   ├── [Flags] → custom mask check
│   └── ASP.NET Core → [EnumDataType] / FluentValidation
│
└── Performance critical
    ├── Bitwise операции для flags
    ├── int[] вместо Dictionary<Enum, T>
    └── (int)enumValue для cast в число
```

---

## 16. Cheat sheet

```csharp
// === Объявление ===
public enum OrderStatus
{
    Unknown = 0,    // ← всегда определи 0
    Pending = 1,
    Paid = 2,
    Shipped = 3
}

[Flags]
public enum Permission
{
    None = 0,
    Read = 1 << 0,    // 1
    Write = 1 << 1,   // 2
    Delete = 1 << 2,  // 4
    All = Read | Write | Delete
}

// === Конверсии ===
(int)OrderStatus.Paid                            // 2
OrderStatus.Paid.ToString()                       // "Paid"
nameof(OrderStatus.Paid)                          // "Paid" (compile-time)

(OrderStatus)2                                    // Paid (без проверки!)
Enum.IsDefined(typeof(OrderStatus), 2)            // true
Enum.IsDefined<OrderStatus>(2)                    // .NET 5+

Enum.Parse<OrderStatus>("Paid")                   // Paid
Enum.TryParse<OrderStatus>("Paid", out var s)     // true, s = Paid
Enum.TryParse<OrderStatus>("paid", true, out var s2) // case-insensitive

Enum.GetValues<OrderStatus>()                     // массив всех
Enum.GetNames<OrderStatus>()                      // массив имён

// === Pattern matching ===
string Describe(OrderStatus s) => s switch
{
    OrderStatus.Pending => "waiting",
    OrderStatus.Paid => "paid",
    OrderStatus.Shipped => "shipping",
    _ => "unknown"
};

if (s is OrderStatus.Paid or OrderStatus.Shipped) { ... }

// === Flags ===
Permission p = Permission.Read | Permission.Write;
p |= Permission.Delete;                  // добавить
p &= ~Permission.Delete;                 // удалить
p ^= Permission.Read;                    // toggle
p.HasFlag(Permission.Read);              // проверить
(p & Permission.Read) != 0;              // проверить (faster pre-NET5)

// === EF Core ===
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>();            // как строка в БД

modelBuilder.Entity<Order>()
    .HasIndex(o => o.Status);             // индекс

// === JSON ===
new JsonSerializerOptions { Converters = { new JsonStringEnumConverter() } }

// === Validation ===
Enum.IsDefined<OrderStatus>(value)
[EnumDataType(typeof(OrderStatus))] на свойстве
RuleFor(x => x.Status).IsInEnum()  // FluentValidation
```

| Сценарий | Решение |
|----------|---------|
| Простой набор статусов | `enum` |
| С display name + transitions | smart enum (sealed class) |
| Битовые маски | `[Flags]` enum |
| Хранение в БД | int mapping (default) |
| Читаемая БД для DBA | string mapping |
| Public API JSON | JsonStringEnumConverter |
| Internal high-volume | int default |
| Validation input | Enum.IsDefined / EnumDataType |
| Switch | switch expression с `_` |
| Default value | `Unknown = 0` или `None = 0` |

---

## 17. Common Pitfalls — с механизмами

### 17.1. default(Enum) без member 0

```csharp
public enum Status { Active = 1, Inactive = 2 }
Status s = default;   // 0 — нет такого member!
```

**Механизм:** `default` для value type = все байты `0`. Для enum это `(EnumType)0` независимо от объявленных members.

**Фикс:** всегда объявляй `Unknown = 0` или `None = 0`.

### 17.2. Cast int → enum не валидирует

```csharp
OrderStatus s = (OrderStatus)999;   // OK, no exception
```

**Механизм:** C# не проверяет валидность при cast int → enum для скорости.

**Фикс:** `if (!Enum.IsDefined<OrderStatus>(s)) throw ...`.

### 17.3. Переименование member = breaking change

```csharp
// V1
public enum Status { Pending, Paid, Shipped }
// V2 — переименовали Paid → Confirmed
public enum Status { Pending, Confirmed, Shipped }
```

**Механизм:** при string mapping в БД старые записи `"Paid"` не парсятся в новый member.

**Фикс:** не переименовывай. Если нужно — миграция данных или alias через custom converter.

### 17.4. [Flags] не степень двойки

```csharp
[Flags]
public enum BadPermission
{
    Read = 1, Write = 2, Delete = 3   // ❌ 3 = Read | Write
}
```

**Механизм:** значения flags должны занимать **отдельные** bits. Если value не степень двойки, оно перекрывается с комбинациями.

**Фикс:** всегда `1, 2, 4, 8, 16` или `1 << n`.

### 17.5. HasFlag(None) всегда true

```csharp
permissions.HasFlag(Permission.None);   // всегда true!
```

**Механизм:** `HasFlag(None)` эквивалентно `(any & 0) == 0`, что всегда true.

**Фикс:** для проверки «нет флагов» — `permissions == Permission.None`.

### 17.6. Mutating bitwise без =

```csharp
permissions & ~Permission.Delete;   // ❌ ничего не меняет — выражение без присвоения
```

**Механизм:** bitwise операторы возвращают новое значение, не мутируют переменную.

**Фикс:** `permissions &= ~Permission.Delete;`.

### 17.7. Переключение int → byte breaking

```csharp
// V1
public enum Status : int { ... }
// БД int колонка
// V2 — поменяли на byte
public enum Status : byte { ... }
```

**Механизм:** EF Core ожидает int, БД хранит byte — миграция обязательна.

**Фикс:** не меняй underlying type после deploy. Если нужно — отдельная миграция БД.

### 17.8. Dynamic enum from DB anti-pattern

```csharp
public class Status
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}
// ... и используешь как enum
```

**Механизм:** «enum» из БД = lookup table, не enum. Type safety теряется.

**Фикс:** если values fixed на compile-time → enum. Если динамические → lookup table с FK constraint.

### 17.9. Switch без default

```csharp
string Describe(OrderStatus s) => s switch
{
    OrderStatus.Pending => "P",
    OrderStatus.Paid => "$"
    // нет default!
};
// При новом member'е → SwitchExpressionException на runtime
```

**Механизм:** switch expression без `_` throws при несовпадении.

**Фикс:** всегда `_ => default` или `_ => throw new ArgumentOutOfRangeException()`.

### 17.10. JSON int vs string contract

```csharp
// Сервер: JsonStringEnumConverter — отправляет "Paid"
// Клиент: ожидает 2 — fails
```

**Механизм:** клиент и сервер должны договориться о формате enum в JSON.

**Фикс:** OpenAPI / Swagger как contract source. Документировать в API docs. Использовать одинаковый converter на обеих сторонах.

> [!question]- Интервью: топ-3 ловушки enum в .NET.
> 1) `default(Enum)` всегда `0`, даже если member 0 не определён — может получить «invalid» enum-значение. Решение: всегда `Unknown = 0`. 2) Cast `(EnumType)int` не проверяет валидность — `(OrderStatus)999` пройдёт без ошибки. Решение: `Enum.IsDefined` после cast. 3) `[Flags]` enum с не-степенями двойки (`Delete = 3`) — bit overlap, HasFlag даёт неожиданные результаты. Решение: всегда степени двойки или `1 << n`.

---

## 18. Practice — упражнения с разбором

### 18.1. Bitmask permission system

**Задача.** Создать `[Flags]` enum для permissions (Read, Write, Delete, Admin), методы для добавления/удаления/проверки. Hash для пользовательского storage.

```csharp
[Flags]
public enum Permission
{
    None    = 0,
    Read    = 1 << 0,    // 1
    Write   = 1 << 1,    // 2
    Delete  = 1 << 2,    // 4
    Admin   = 1 << 3,    // 8
    
    // Удобные комбинации
    ReadWrite = Read | Write,
    All = Read | Write | Delete | Admin
}

public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public Permission Permissions { get; set; } = Permission.None;
    
    public void Grant(Permission p) => Permissions |= p;
    public void Revoke(Permission p) => Permissions &= ~p;
    public bool Has(Permission p) => Permissions.HasFlag(p);
    public bool HasAny(Permission flags) => (Permissions & flags) != 0;
    public bool HasAll(Permission flags) => (Permissions & flags) == flags;
}

// Использование
var user = new User { Name = "Alice" };
user.Grant(Permission.Read);
user.Grant(Permission.Write);

user.Has(Permission.Read);             // true
user.Has(Permission.Delete);           // false
user.HasAny(Permission.ReadWrite);     // true
user.HasAll(Permission.ReadWrite);     // true

user.Revoke(Permission.Write);
user.Has(Permission.Write);            // false

Console.WriteLine(user.Permissions);    // "Read"

// Сериализация
JsonSerializer.Serialize(user, new JsonSerializerOptions
{
    Converters = { new JsonStringEnumConverter() }
});
// {"Id":0,"Name":"Alice","Permissions":"Read"}
```

**Разбор:** `1 << n` для readability. `Has` использует `HasFlag` (читаемее), `HasAny`/`HasAll` через bitwise для multiple flags. Combined values (`ReadWrite`, `All`) удобны для типичных групп. Хранится один int в БД (compact), но всё равно читаемо при serialization.

### 18.2. Smart enum для OrderStatus

**Задача.** Реализовать `OrderStatus` как smart enum с `DisplayName`, `Color`, `CanTransitionTo`.

```csharp
public sealed class OrderStatus : IEquatable<OrderStatus>
{
    public static readonly OrderStatus Pending = new(1, "Pending", "Waiting for payment", "#FFA500", [2, 5]);
    public static readonly OrderStatus Paid = new(2, "Paid", "Payment received", "#90EE90", [3, 5]);
    public static readonly OrderStatus Shipped = new(3, "Shipped", "On the way", "#87CEEB", [4]);
    public static readonly OrderStatus Delivered = new(4, "Delivered", "Order completed", "#00FF00", []);
    public static readonly OrderStatus Cancelled = new(5, "Cancelled", "Cancelled by user", "#FF0000", []);
    
    public int Id { get; }
    public string Name { get; }
    public string DisplayName { get; }
    public string Color { get; }
    private readonly int[] _allowedTransitions;
    
    private OrderStatus(int id, string name, string displayName, string color, int[] allowedTransitions)
    {
        Id = id;
        Name = name;
        DisplayName = displayName;
        Color = color;
        _allowedTransitions = allowedTransitions;
    }
    
    public static IReadOnlyList<OrderStatus> All { get; } = [Pending, Paid, Shipped, Delivered, Cancelled];
    
    public static OrderStatus FromId(int id) =>
        All.FirstOrDefault(s => s.Id == id)
        ?? throw new ArgumentException($"Unknown OrderStatus id: {id}");
    
    public bool CanTransitionTo(OrderStatus target) =>
        _allowedTransitions.Contains(target.Id);
    
    public bool IsTerminal => _allowedTransitions.Length == 0;
    
    public bool Equals(OrderStatus? other) => other is not null && Id == other.Id;
    public override bool Equals(object? obj) => obj is OrderStatus s && Equals(s);
    public override int GetHashCode() => Id;
    public override string ToString() => Name;
}

// Использование
var current = OrderStatus.Pending;
var next = OrderStatus.Paid;

if (current.CanTransitionTo(next))
    Console.WriteLine($"Transition {current} → {next} OK");

OrderStatus.Delivered.IsTerminal;     // true
OrderStatus.Pending.IsTerminal;        // false

// EF Core mapping
modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion(
        v => v.Id,
        v => OrderStatus.FromId(v));
```

**Разбор:** smart enum как `sealed class` с `static readonly` instances. Каждый instance несёт metadata (color, transitions). Логика state machine инкапсулирована в `CanTransitionTo`. EF Core mapping через `HasConversion`. Для UI можно отдавать `DisplayName` и `Color` без switch'ей.

### 18.3. EF Core enum-as-string mapping

**Задача.** Настроить `OrderStatus` enum для хранения как string в БД, чтобы DBA могли читать данные напрямую.

```csharp
public enum OrderStatus
{
    Unknown = 0,
    Pending = 1,
    Paid = 2,
    Shipped = 3,
    Delivered = 4,
    Cancelled = 5
}

public class Order
{
    public int Id { get; set; }
    public OrderStatus Status { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
}

public class AppDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();
    
    protected override void OnModelCreating(ModelBuilder mb)
    {
        mb.Entity<Order>(b =>
        {
            // Как string
            b.Property(o => o.Status)
                .HasConversion<string>()
                .HasMaxLength(20)
                .IsRequired();
            
            // Index для частых фильтров
            b.HasIndex(o => o.Status);
        });
    }
}

// Запросы
var paidOrders = await db.Orders
    .Where(o => o.Status == OrderStatus.Paid)
    .ToListAsync();
// SQL: SELECT * FROM Orders WHERE Status = 'Paid'

// Migration script вручную
// CREATE INDEX IX_Orders_Status ON Orders(Status);
// При первой загрузке: existing int values → strings через UPDATE
```

**Разбор:** `HasConversion<string>()` — встроенный converter. `HasMaxLength` — оптимизация для NVARCHAR. Index ускоряет фильтры. EF Core корректно генерирует SQL с строковыми сравнениями. Trade-off: больше места и slower compare, но DBA-friendly.

### 18.4. JSON enum API versioning

**Задача.** Сделать API enum совместимый с V1 и V2 (V1: "Confirmed", V2: "Paid"). Принимаем оба, отдаём V2.

```csharp
public enum OrderStatus { Pending, Paid, Shipped }

public class TolerantOrderStatusConverter : JsonConverter<OrderStatus>
{
    public override OrderStatus Read(ref Utf8JsonReader reader, Type type, JsonSerializerOptions opts)
    {
        var value = reader.GetString();
        
        // V1 alias
        return value switch
        {
            "Confirmed" => OrderStatus.Paid,    // legacy
            null => throw new JsonException("Status is null"),
            _ => Enum.TryParse<OrderStatus>(value, ignoreCase: true, out var s)
                ? s
                : throw new JsonException($"Unknown OrderStatus: {value}")
        };
    }
    
    public override void Write(Utf8JsonWriter writer, OrderStatus value, JsonSerializerOptions opts)
    {
        writer.WriteStringValue(value.ToString());
    }
}

// Использование
var options = new JsonSerializerOptions
{
    Converters = { new TolerantOrderStatusConverter() }
};

JsonSerializer.Deserialize<OrderStatus>("\"Paid\"", options);       // Paid
JsonSerializer.Deserialize<OrderStatus>("\"Confirmed\"", options);  // Paid (legacy alias!)
JsonSerializer.Serialize(OrderStatus.Paid, options);                 // "Paid"
```

**Разбор:** custom converter `Read` принимает legacy alias `"Confirmed"` и парсит к новому member `Paid`. `Write` всегда отдаёт новое имя. Это **forward-compatible API design**: старые клиенты, отправляющие `"Confirmed"`, продолжают работать; новые видят `"Paid"`.

### 18.5. Refactoring boolean flags → [Flags] enum

**Задача.** Refactor `User` с 5 boolean полями (`IsActive`, `IsAdmin`, `CanRead`, `CanWrite`, `CanDelete`) в один `[Flags]` enum.

```csharp
// До
public class User
{
    public int Id { get; set; }
    public bool IsActive { get; set; }
    public bool IsAdmin { get; set; }
    public bool CanRead { get; set; }
    public bool CanWrite { get; set; }
    public bool CanDelete { get; set; }
}

// 5 boolean колонок в БД, разбросанная логика проверок
if (user.IsActive && (user.IsAdmin || user.CanRead)) { ... }

// После
[Flags]
public enum UserFlags
{
    None     = 0,
    Active   = 1 << 0,    // 1
    Admin    = 1 << 1,    // 2
    CanRead  = 1 << 2,    // 4
    CanWrite = 1 << 3,    // 8
    CanDelete = 1 << 4,   // 16
}

public class User
{
    public int Id { get; set; }
    public UserFlags Flags { get; set; } = UserFlags.None;
    
    public bool IsActive => Flags.HasFlag(UserFlags.Active);
    public bool IsAdmin => Flags.HasFlag(UserFlags.Admin);
    public bool CanRead => IsAdmin || Flags.HasFlag(UserFlags.CanRead);
    // Admin неявно имеет все permissions
}

// Использование
if (user.IsActive && user.CanRead) { ... }

// Migration SQL (один раз):
// ALTER TABLE Users ADD Flags INT NOT NULL DEFAULT 0;
// UPDATE Users SET Flags =
//     CASE WHEN IsActive = 1 THEN 1 ELSE 0 END +
//     CASE WHEN IsAdmin = 1 THEN 2 ELSE 0 END +
//     CASE WHEN CanRead = 1 THEN 4 ELSE 0 END +
//     CASE WHEN CanWrite = 1 THEN 8 ELSE 0 END +
//     CASE WHEN CanDelete = 1 THEN 16 ELSE 0 END;
// ALTER TABLE Users DROP COLUMN IsActive, IsAdmin, CanRead, CanWrite, CanDelete;
```

**Разбор:** боолевые колонки → один int, экономия места и упрощение API. Computed properties (`IsActive`, `IsAdmin`) сохраняют backward-compat для существующего кода. Migration script преобразует existing data в bitmask. После деплоя — refactoring продолжается удалением старых колонок.

---

## 19. Performance benchmarks

### 19.1. Cast vs HasFlag (.NET 5+)

```csharp
[Flags]
public enum Permission { Read=1, Write=2, Delete=4 }

[Benchmark]
public bool HasFlag_Method() => p.HasFlag(Permission.Read);

[Benchmark]
public bool BitwiseAnd() => (p & Permission.Read) != 0;
```

Результаты (BenchmarkDotNet, .NET 8, x64):

```
| Method        |    Mean | Allocated |
|-------------- |--------:|----------:|
| HasFlag       | 0.42 ns |       0 B |
| BitwiseAnd    | 0.38 ns |       0 B |
```

Разница пренебрежима. До .NET 5 `HasFlag` боксил, был ~50ns. Сейчас inline-нут JIT-ом.

### 19.2. ToString performance

```csharp
[Benchmark]
public string ToString_Method() => OrderStatus.Paid.ToString();

[Benchmark]
public string Nameof_Compile() => nameof(OrderStatus.Paid);

[Benchmark]
public string Format_D() => OrderStatus.Paid.ToString("D");
```

Результаты:

```
| Method            |     Mean | Allocated |
|------------------ |---------:|----------:|
| ToString_Method   |  4.5 ns |       0 B |    ← .NET 5+ optimized
| Nameof_Compile    |  0.0 ns |       0 B |    ← compile-time
| Format_D          |  8.2 ns |      32 B |    ← числовое представление, allocation
```

`nameof()` бесплатно — compile-time. `ToString()` оптимизирован для enum. `"D"` формат всегда дороже.

### 19.3. Enum.Parse vs TryParse

```csharp
[Benchmark]
public OrderStatus Parse_Static() => Enum.Parse<OrderStatus>("Paid");

[Benchmark]
public OrderStatus TryParse_Out() =>
    Enum.TryParse<OrderStatus>("Paid", out var s) ? s : default;

[Benchmark]
public OrderStatus Switch_Manual() => "Paid" switch
{
    "Pending" => OrderStatus.Pending,
    "Paid" => OrderStatus.Paid,
    "Shipped" => OrderStatus.Shipped,
    _ => default
};
```

Результаты:

```
| Method        |    Mean | Allocated |
|-------------- |--------:|----------:|
| Parse_Static  | 35.2 ns |       0 B |
| TryParse_Out  | 35.0 ns |       0 B |
| Switch_Manual |  1.1 ns |       0 B |   ← в 30x быстрее
```

Если parsing критичен в hot path — switch вручную в 30x быстрее. Reflection в `Enum.Parse` дорогая.

### 19.4. `Dictionary<Enum, T>` vs `T[]`

```csharp
private static readonly Dictionary<OrderStatus, string> Dict = new()
{
    [OrderStatus.Pending] = "P",
    [OrderStatus.Paid] = "$",
    [OrderStatus.Shipped] = "S"
};

private static readonly string[] Arr = new string[] { "U", "P", "$", "S", "D", "C" };

[Benchmark]
public string DictLookup() => Dict[OrderStatus.Paid];

[Benchmark]
public string ArrLookup() => Arr[(int)OrderStatus.Paid];
```

Результаты:

```
| Method     |    Mean | Allocated |
|----------- |--------:|----------:|
| DictLookup | 7.5 ns |       0 B |
| ArrLookup  | 0.4 ns |       0 B |    ← 18x faster
```

Для маленьких enum'ов (5-20 member'ов) array через index — оптимально. Trade-off: память (если enum sparse), но обычно того стоит.

### 19.5. Boxing проверка

```csharp
public static void TakesObject(object o) { /* boxing */ }
public static void TakesEnum<T>(T value) where T : Enum { /* без boxing */ }

[Benchmark]
public void BoxingCall() => TakesObject(OrderStatus.Paid);

[Benchmark]
public void NoBoxingCall() => TakesEnum(OrderStatus.Paid);
```

Результаты:

```
| Method       |    Mean | Allocated |
|------------- |--------:|----------:|
| BoxingCall   | 5.8 ns |      24 B |    ← heap allocation!
| NoBoxingCall | 0.5 ns |       0 B |
```

Передача enum как `object` всегда боксит. Generic `where T : Enum` — нет boxing. В hot path избегай `object`-параметров для enum.

> [!question]- Интервью: какие enum-операции дорогие, какие дешёвые?
> **Дешёвые** (наносекунды, без allocations): cast `(int)enumValue`, equality `enumA == enumB`, bitwise (`& | ^ ~`), `nameof()` (compile-time), `HasFlag` (.NET 5+ optimized). **Дорогие**: `Enum.Parse` / `TryParse` (reflection ~35ns), `Enum.IsDefined` (reflection), `Enum.GetValues` (reflection + allocation), боксинг через `object`-параметр (~24B heap). В hot path — кэшируй результаты `GetValues`, используй switch вместо Parse, избегай `object`-параметров для enum.

---

## 20. Source generators для enum

### 20.1. Зачем

`Enum.Parse`, `Enum.GetValues`, `Enum.IsDefined` — все используют reflection. Source generators могут сгенерировать код во время компиляции, который делает то же без reflection — быстрее, AOT-friendly.

### 20.2. EnumGenerator — пример

Популярный пакет: `Andrew Lock's NetEscapades.EnumGenerators`:

```bash
dotnet add package NetEscapades.EnumGenerators
```

```csharp
using NetEscapades.EnumGenerators;

[EnumExtensions]
public enum OrderStatus
{
    Unknown = 0,
    Pending = 1,
    Paid = 2,
    Shipped = 3,
    Delivered = 4,
    Cancelled = 5
}
```

Source generator создаёт extension class:

```csharp
// Сгенерировано
public static partial class OrderStatusExtensions
{
    public static string ToStringFast(this OrderStatus value) => value switch
    {
        OrderStatus.Unknown => nameof(OrderStatus.Unknown),
        OrderStatus.Pending => nameof(OrderStatus.Pending),
        OrderStatus.Paid => nameof(OrderStatus.Paid),
        OrderStatus.Shipped => nameof(OrderStatus.Shipped),
        OrderStatus.Delivered => nameof(OrderStatus.Delivered),
        OrderStatus.Cancelled => nameof(OrderStatus.Cancelled),
        _ => value.ToString()
    };

    public static bool IsDefinedFast(OrderStatus value) => value switch
    {
        OrderStatus.Unknown or OrderStatus.Pending or OrderStatus.Paid 
            or OrderStatus.Shipped or OrderStatus.Delivered or OrderStatus.Cancelled => true,
        _ => false
    };

    public static OrderStatus[] GetValuesFast() =>
        new[] { OrderStatus.Unknown, OrderStatus.Pending, /* ... */ };
}
```

Использование:

```csharp
OrderStatus.Paid.ToStringFast();        // "Paid", без reflection
OrderStatusExtensions.IsDefinedFast((OrderStatus)999);   // false
OrderStatusExtensions.GetValuesFast();                    // массив
```

### 20.3. Performance generators vs reflection

```
| Method                  |    Mean | Allocated |
|------------------------ |--------:|----------:|
| Enum.ToString           |  4.5 ns |       0 B |
| ToStringFast (gen)      |  0.4 ns |       0 B |    ← 11x
|                         |         |           |
| Enum.IsDefined          | 25.2 ns |       0 B |
| IsDefinedFast (gen)     |  0.3 ns |       0 B |    ← 84x
|                         |         |           |
| Enum.GetValues          | 95.0 ns |     112 B |
| GetValuesFast (gen)     |  3.2 ns |      32 B |    ← 30x
```

Source generators дают огромный perf-boost для enum-операций.

### 20.4. AOT-совместимость

Native AOT (`PublishAot=true`) ограничивает reflection. `Enum.Parse` / `Enum.GetValues` могут не работать в AOT-mode без специальных hint'ов. Source generators решают это: код сгенерирован, рефлексия не нужна.

### 20.5. Когда использовать

✅ **Используй когда:**
- Hot path с миллионами enum-операций.
- Native AOT compilation.
- Минимизация startup time / RAM.

❌ **Не нужно когда:**
- Маленький проект, perf не критичен.
- Не часто используются эти операции.

> [!question]- Интервью: что дают source generators для enum?
> Code generation во время компиляции для типичных операций (`ToString`, `IsDefined`, `GetValues`), которые иначе используют reflection. Performance: 10-80x быстрее. Allocations: меньше или ноль. AOT-compatible: работает в Native AOT без hint'ов. Популярный пакет — NetEscapades.EnumGenerators (Andrew Lock). Trade-off: дополнительная зависимость, чуть более сложный setup. Используй для perf-критичного кода и AOT публикаций.

---

## 21. Discriminated unions — что .NET НЕ умеет

### 21.1. Чего нет

В Rust / F# / Swift / Kotlin есть **discriminated unions** (sum types) — enum с ассоциированными данными:

```rust
// Rust
enum PaymentResult {
    Success { transactionId: String },
    Failed { error: String, retryAfter: u32 },
    Pending,
}
```

```fsharp
// F#
type PaymentResult =
    | Success of transactionId: string
    | Failed of error: string * retryAfter: int
    | Pending
```

В стабильном C# (C# 14 / .NET 10, на 2026-08) такого нет — только числовые enum. Но фича уже дошла до компилятора: ключевое слово `union` доступно в preview с .NET 11 Preview 2 (апрель 2026) — статус и синтаксис в 21.5.

### 21.2. Эмуляция через class hierarchy

```csharp
public abstract record PaymentResult;
public sealed record Success(string TransactionId) : PaymentResult;
public sealed record Failed(string Error, int RetryAfter) : PaymentResult;
public sealed record Pending() : PaymentResult;

// Использование
PaymentResult result = ProcessPayment();

string Describe(PaymentResult r) => r switch
{
    Success s => $"OK: {s.TransactionId}",
    Failed f => $"Error: {f.Error}, retry in {f.RetryAfter}s",
    Pending => "Waiting...",
    _ => throw new ArgumentOutOfRangeException()
};
```

`record` + pattern matching эмулирует discriminated unions. Не такой компактный синтаксис, но работает.

Trade-offs:
- ❌ Multiple types вместо одного.
- ❌ Heap allocation (records — class).
- ❌ Нет exhaustiveness checking без `_`.
- ✅ Полноценные методы / поля на каждом case.
- ✅ Совместимо с pattern matching.

### 21.3. OneOf — community library

```bash
dotnet add package OneOf
```

```csharp
using OneOf;

public OneOf<Success, Failed, Pending> ProcessPayment() { ... }

public record Success(string TransactionId);
public record Failed(string Error);
public record Pending;

// Использование
var result = ProcessPayment();
result.Switch(
    success => Console.WriteLine($"OK: {success.TransactionId}"),
    failed => Console.WriteLine($"Err: {failed.Error}"),
    pending => Console.WriteLine("Waiting..."));

// Match с возвратом
string description = result.Match(
    success => $"OK: {success.TransactionId}",
    failed => $"Err: {failed.Error}",
    pending => "Waiting...");
```

`OneOf<T1, T2, T3>` — generic-обёртка для union types. Type-safe. Используется в DDD/functional projects.

### 21.4. Когда нужна union вместо enum

✅ **Discriminated union лучше когда:**
- Каждый case имеет **разные данные** (Success содержит txId, Failed — error).
- Pattern matching по case + извлечение данных одновременно.
- Result/Either patterns (`Result<TOk, TError>`).

✅ **Простой enum лучше когда:**
- Все cases только метки без данных.
- Compact бит-mask нужен.
- Performance critical (enum — value type, union — class).

### 21.5. `union` — уже в preview (.NET 11)

Дискуссия прошла путь от [issue #113](https://github.com/dotnet/csharplang/issues/113) до актуального proposal `unions.md` в dotnet/csharplang — и до кода: с .NET 11 Preview 2 (апрель 2026) компилятор понимает контекстное ключевое слово `union`. GA ожидается с C# 15 / .NET 11 (~ноябрь 2026), но команда пока прямо пишет «active, not yet committed to a release».

```csharp
// Требуется: .NET 11 Preview SDK, net11.0, LangVersion = preview
public union PaymentResult(Success, Failed, Pending);

// Cases — обычные типы (например, records из 21.2)
string Describe(PaymentResult r) => r switch
{
    Success s => $"OK: {s.TransactionId}",
    Failed f => $"Error: {f.Error}, retry in {f.RetryAfter}s",
    Pending => "Waiting..."
    // без _ — компилятор проверяет exhaustiveness по списку cases
};
```

**Механизм:** компилятор генерирует struct с полем `object? Value` (value types боксируются) и implicit conversion из каждого case-типа. Pattern matching автоматически «разворачивает» `Value`, а switch по всем cases считается исчерпывающим — discard `_` не нужен. Когда фича дойдёт до GA, она станет встроенной заменой OneOf и эмуляции через record hierarchy.

> [!question]- Интервью: что такое discriminated union и есть ли они в C#?
> Discriminated union (sum type) — это enum, где каждый case может содержать **разные ассоциированные данные**. Есть в Rust, F#, Swift, Kotlin (sealed classes). В стабильном C# (C# 14) встроенно нет — эмулируются через `abstract record` hierarchy + pattern matching, или через `OneOf<T1, T2, T3>` библиотеку. Простые enum в C# — это лишь именованные числа без данных. Ключевое слово `union` уже в preview с .NET 11 Preview 2, GA ожидается с C# 15 (~ноябрь 2026). Пока — emulation для Result/Either patterns в DDD проектах.

---

## 22. Что читать дальше — порядок и почему

1. **[[csharp-basics|C# Basics]]** — типы и переменные.
2. **[[modern-features|Modern Features]]** — pattern matching, switch expressions с enum.
3. **[[generics-deep|Generics deep]]** — `where T : Enum` constraint.
4. **EF Core mapping** — value converters, table-per-hierarchy.
5. **[[dotnet-cli-getting-started|.NET CLI]]** — для миграций.
6. **JSON deep dive** — кастомные converter'ы.
7. **DDD value objects** — smart enum vs records.
8. **Source Generators** — для type-safe smart enum.

---

## 23. См. также

- [[csharp-basics|C# Basics]] — value types vs reference types
- [[modern-features|Modern Features]] — pattern matching, switch expressions
- [[generics-deep|Generics deep]] — Enum constraint
- [[anonymous-types|Anonymous Types]] — для projections с enum
- EF Core Mapping — ValueConverter
- JSON Serialization — JsonStringEnumConverter
- DDD Value Objects — smart enum patterns
- Roslyn Analyzers — статический анализ enum usage
- Ardalis.SmartEnum — popular smart enum library
- Vogen — type-safe value objects

---

## 24. Reading list

- **Microsoft Docs — Enumeration types** — learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/enum
- **Microsoft Docs — Enum class** — learn.microsoft.com/dotnet/api/system.enum
- **Microsoft Docs — Flags attribute** — learn.microsoft.com/dotnet/api/system.flagsattribute
- **Eric Lippert — Enum design considerations** — ericlippert.com
- **Steve Smith — Smart Enum pattern** — ardalis.com/enum-alternatives-in-c/
- **Ardalis.SmartEnum** — github.com/ardalis/SmartEnum
- **Vogen** — github.com/SteveDunn/Vogen
- **Microsoft .NET Design Guidelines — Enum Design** — learn.microsoft.com/dotnet/standard/design-guidelines/enum
- **Andrew Lock — Working with EF Core enum mappings** — andrewlock.net
- **Microsoft Docs — Pattern matching enums** — learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching
- **CSharpLang — Unions proposal** — github.com/dotnet/csharplang, proposals/unions.md (вырос из issue #113)
- **Bill Wagner — Effective C#** — items по enum design
- **SharpLab** — sharplab.io — посмотреть IL для enum operations
- **BenchmarkDotNet — Enum operations** — benchmarkdotnet.org
- **Roslyn Analyzers — Enum design rules** — learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/ca1027

---
