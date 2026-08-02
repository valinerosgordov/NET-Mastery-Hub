---
tags: [csharp, tuples, deconstruction, value-tuple, junior]
level: Junior
date: 2026-05-04
---

# Tuples и Deconstruction — кортежи и деконструкция

> **Modern C# everyday feature.** ValueTuple, named tuples, deconstruction, multiple return values. Появилось в C# 7.0, стало повседневным инструментом и закрыло целый класс боли — `out`-параметры, `Tuple<T1,T2>` с безымянными `Item1`/`Item2`, и временные классы-помощники.

---

## 0. Как читать этот файл

Файл линейный: можно читать с начала, можно прыгать. Если ты впервые видишь tuple в C# — читай разделы 1→6 подряд: получишь рабочую модель. Если уже пользуешься, но хочется глубины — раздел 2 (внутреннее устройство), 3 (где живут имена), 10 (equality), 14 (performance), 18 (advanced).

Все примеры — самостоятельные, копируются в `dotnet run` и запускаются. Где есть `// expected: ...` — это вывод, который ты должен увидеть.

Cross-language якоря (`> [!info]-`) свёрнуты по умолчанию — открывай, если переходишь из Python / JavaScript / TypeScript / Rust / Go.

Interview-вопросы (`> [!question]-`) встроены рядом с теорией — формат «как ответить на собесе за 30 секунд».

---

## 1. Что это, зачем и когда

### 1.1. Что такое tuple

**Tuple** — это упакованная вместе группа значений без отдельного типа.

```csharp
// Без tuple — приходится заводить класс или использовать out
class Coord { public int X; public int Y; }
var p = new Coord { X = 1, Y = 2 };

// С tuple — на месте, без объявления типа
var p = (X: 1, Y: 2);
Console.WriteLine($"{p.X}, {p.Y}");   // 1, 2
```

Tuple — это не магическое существительное, а конкретный тип: `System.ValueTuple<T1, T2, ...>` (struct из стандартной библиотеки). Когда ты пишешь `(1, 2)`, компилятор разворачивает это в `new ValueTuple<int, int>(1, 2)`. Никакого скрытого аллокатора, никаких proxy-объектов.

### 1.2. Зачем tuples появились — какая боль решилась

До C# 7.0 для возврата нескольких значений было три плохих варианта:

```csharp
// Вариант 1: out-параметры — императивный стиль, портит сигнатуру
public bool TryDivide(int a, int b, out int quotient, out int remainder)
{
    if (b == 0) { quotient = 0; remainder = 0; return false; }
    quotient = a / b;
    remainder = a % b;
    return true;
}

// Использование — нужно объявить переменные заранее (раньше — обязательно вне вызова)
int q, r;
if (TryDivide(10, 3, out q, out r))
    Console.WriteLine($"{q} ост. {r}");
```

```csharp
// Вариант 2: Tuple<T1, T2> — тип из .NET 4.0, class на куче, поля Item1/Item2
public Tuple<int, int> Divide(int a, int b) => Tuple.Create(a / b, a % b);

var result = Divide(10, 3);
Console.WriteLine($"{result.Item1} ост. {result.Item2}");
// Что такое Item1? Что такое Item2? Без документации — никак.
```

```csharp
// Вариант 3: завести отдельный класс / struct
public class DivisionResult
{
    public int Quotient { get; init; }
    public int Remainder { get; init; }
}

public DivisionResult Divide(int a, int b) =>
    new() { Quotient = a / b, Remainder = a % b };
```

У каждого варианта своя цена:

| Подход | Цена |
|--------|------|
| `out` | Портит сигнатуру, нельзя использовать в LINQ, нельзя `await`, императивный стиль |
| `Tuple<T1, T2>` | Heap-аллокация, безымянные `Item1`/`Item2`, immutable но без value-equality |
| Класс на каждый return | Одноразовый тип, размывает домен, лишний файл |

`ValueTuple` (C# 7.0) закрывает все три:

```csharp
public (int Quotient, int Remainder) Divide(int a, int b) => (a / b, a % b);

var (q, r) = Divide(10, 3);
Console.WriteLine($"{q} ост. {r}");
// Никакой heap-аллокации, имена есть, читается как естественный код.
```

### 1.3. ValueTuple vs Tuple — почему две версии

В .NET есть **два разных типа** с похожими именами. Это смущает джунов, но различие принципиальное:

| | `Tuple<T1, T2>` (legacy, .NET 4.0) | `ValueTuple<T1, T2>` (C# 7.0+) |
|---|---|---|
| Категория | `class` (reference type) | `struct` (value type) |
| Где живёт | Heap | Stack (или внутри другого объекта) |
| Аллокация | Каждое создание = GC pressure | Без аллокации |
| Поля | `Item1`, `Item2` (read-only properties) | `Item1`, `Item2` (mutable fields) |
| Mutability | Immutable | Mutable (да, поля можно менять) |
| Equality | Reference equality по умолчанию | Value equality (auto) |
| Синтаксис создания | `Tuple.Create(1, 2)` или `new Tuple<int, int>(1, 2)` | `(1, 2)` или `new ValueTuple<int, int>(1, 2)` |
| Named-синтаксис | Нет | Да: `(X: 1, Y: 2)` |
| Deconstruction | Нет (до .NET Core 2.0) | Да |

**Правило:** в новом коде — только `ValueTuple`. `Tuple<T>` встречается в legacy-коде и в редких API (например, `Task.Run(() => Tuple.Create(...))` в очень старом коде). Если видишь `Tuple.Create` или `tuple.Item1` без подсветки именованного синтаксиса — это старый стиль, который пора рефакторить.

> [!info]- Если ты знаешь Python / JavaScript / Rust / Go
> **Python:** `(1, 2)` — встроенный `tuple`, immutable, неименованный (но есть `namedtuple` и `NamedTuple` из `typing`). C# `ValueTuple` ближе к `NamedTuple` по идее, но runtime-тип всегда есть.
>
> **JavaScript:** аналога нет, ближайшее — массив `[1, 2]` или объект `{x: 1, y: 2}`. ECMAScript Records & Tuples — proposal, в стандарте пока нет.
>
> **TypeScript:** `[number, number]` — tuple-тип, но это compile-time only; в runtime это просто массив. C# даёт настоящий runtime-тип со stack-аллокацией.
>
> **Rust:** `(1, 2)` — built-in tuple, тоже value type на стеке. Ближе всего к C# `ValueTuple`. Деконструкция через `let (x, y) = (1, 2)` — точная аналогия `var (x, y) = (1, 2)`.
>
> **Go:** multiple return values — `func div(a, b int) (int, int)` — встроено в язык, но это не tuple, а именно multiple returns. Нельзя положить в переменную как объект. C# tuple — и multiple return, и first-class value.

### 1.4. Эволюция: C# 7.0 → C# 14

| Версия | Год | Что добавили |
|--------|-----|--------------|
| **C# 7.0** | 2017 | `ValueTuple`, литеральный синтаксис `(1, 2)`, named tuples `(X: 1, Y: 2)`, deconstruction declaration `var (x, y) = ...` |
| **C# 7.1** | 2017 | Tuple projection initializers — имена выводятся из выражения (`var t = (a, b)` → `(int a, int b)`) |
| **C# 7.3** | 2018 | Tuple equality — операторы `==` и `!=` по value |
| **C# 8.0** | 2019 | Recursive patterns включая positional pattern с деконструкцией |
| **C# 9.0** | 2020 | Records — auto-generated `Deconstruct` для positional records |
| **C# 10** | 2021 | Records struct, расширение property patterns |
| **C# 11** | 2022 | List patterns, можно деконструировать массивы и списки |
| **C# 12** | 2023 | `using` alias для tuple-типов: `using Coord = (int X, int Y);` |
| **C# 13** | 2024 | `params (int X, int Y)[]` — params-коллекции с tuple-элементами |

**Что важно для джуна:** базовый синтаксис стабилен с C# 7.0 (2017). Всё, что позже — постепенные улучшения. Если работаешь с `.NET 6` или новее, у тебя точно есть всё.

### 1.5. Когда tuple, когда class / record / anonymous

Это самый частый вопрос. Короткое правило:

```
Внутри метода, локальное скопление значений?           → tuple
Возврат 2-3 значений из private/internal метода?       → tuple
Возврат из public API (особенно библиотечного)?        → record
Composite key для Dictionary<K, V> / GroupBy?          → tuple
LINQ Select projection (внутри одного запроса)?        → anonymous type
LINQ projection, но результат уезжает в Razor / API?   → record / DTO
> 4 значений?                                          → record / class
Имеет поведение (методы, валидацию)?                   → record / class
Нужна эволюция (добавлять/удалять поля)?               → record
```

Полная decision matrix — в разделе 15. Здесь — главное правило: **tuple — для коротких локальных задач. Если значение пересекает границу публичного API или живёт дольше одного метода — заводи record.**

> [!question]- Интервью: чем `ValueTuple` отличается от `Tuple<T>`?
> `ValueTuple` — `struct`, живёт на стеке, без аллокации, имеет value equality, поддерживает named-синтаксис и деконструкцию. `Tuple<T>` — `class`, живёт на куче, immutable, equality по ссылке (до C# 7.3), без именованного синтаксиса. В новом коде используем `ValueTuple` через литерал `(a, b)`. `Tuple<T>` остался ради обратной совместимости с .NET 4.x кодом.

---

## 2. ValueTuple под капотом

### 2.1. Generic struct ValueTuple<T1, ..., T7, TRest>

Тип `System.ValueTuple` — это семейство `struct`-ов с разной арностью. В исходниках .NET они выглядят примерно так (упрощённо):

```csharp
public struct ValueTuple<T1, T2>
{
    public T1 Item1;
    public T2 Item2;

    public ValueTuple(T1 item1, T2 item2)
    {
        Item1 = item1;
        Item2 = item2;
    }
}

public struct ValueTuple<T1, T2, T3> { /* ... */ }
public struct ValueTuple<T1, T2, T3, T4> { /* ... */ }
// ... до T7

public struct ValueTuple<T1, T2, T3, T4, T5, T6, T7, TRest>
    where TRest : struct
{
    public T1 Item1;
    // ...
    public T7 Item7;
    public TRest Rest;
}
```

Ключевые моменты:

1. **Это `struct`** — value type. Размещается на стеке (как локальная переменная) или внутри другого объекта (как поле).
2. **Поля, не свойства.** `Item1`, `Item2` — это публичные поля (`public T1 Item1;`), а не auto-properties. Это даёт прямой доступ без вызова геттера и позволяет передавать tuple по `ref`.
3. **Mutable.** В отличие от `Tuple<T>`, поля `ValueTuple` можно менять: `t.Item1 = 100;`. Это иногда удобно, но создаёт ловушку (см. раздел 17.4).
4. **Up to 7 + Rest.** Базовых перегрузок — от `ValueTuple<T1>` до `ValueTuple<T1...T7>`. Для большего — `T8` живёт в `Rest.Item1` следующего ValueTuple.

### 2.2. Item1..Item7 + Rest — как nesting устроено

Когда ты пишешь:

```csharp
var t = (1, 2, 3, 4, 5, 6, 7, 8, 9);
```

компилятор разворачивает это в:

```csharp
var t = new ValueTuple<int, int, int, int, int, int, int, ValueTuple<int, int>>(
    1, 2, 3, 4, 5, 6, 7,
    new ValueTuple<int, int>(8, 9)
);

// Доступ — компилятор маскирует nesting:
int a = t.Item8;   // на самом деле t.Rest.Item1
int b = t.Item9;   // на самом деле t.Rest.Item2
```

Точно так же 16-элементный tuple — это `ValueTuple<T1...T7, ValueTuple<T8...T14, ValueTuple<T15, T16>>>`. Двухуровневая вложенность.

**Зачем знать:** в редких сценариях (reflection, expression trees, debugger view) ты увидишь `Rest.Item1` вместо `Item8`. Это не баг — это физическая структура.

### 2.3. Stack allocation — кому это важно

`ValueTuple` живёт на стеке (как локальная переменная) или встраивается в layout родительского объекта (если `ValueTuple` — поле класса). Что это значит на практике:

```csharp
public (int X, int Y) GetPoint()
{
    return (1, 2);   // ноль heap-аллокаций
}

var p = GetPoint();      // tuple лежит прямо в стек-фрейме вызывающего
Console.WriteLine(p.X);
```

Сравни с `Tuple<int, int>`:

```csharp
public Tuple<int, int> GetPoint()
{
    return Tuple.Create(1, 2);   // heap-аллокация на каждом вызове
}
```

Если этот метод вызывается миллион раз в секунду (HFT, парсер, hot path) — разница ощутима: миллион выделений на куче vs ноль. Подробнее — в разделе 14.

### 2.4. Mutability — поля, а не свойства

```csharp
var t = (X: 1, Y: 2);
t.X = 100;   // ✅ OK — это field assignment
Console.WriteLine(t.X);   // 100
```

Это работает потому, что `X` транслируется в поле `Item1`, а у `ValueTuple` поля публичные. Но осторожно — при передаче в метод tuple копируется (см. раздел 17.4).

```csharp
var arr = new (int X, int Y)[3];
arr[0] = (1, 2);
arr[0].X = 100;   // ✅ Mutating element of array — это работает
// Внутри массива struct хранится по value, но array indexer возвращает managed reference,
// поэтому мутация попадает в исходный элемент.
```

Сравни с `List<(int X, int Y)>`:

```csharp
var list = new List<(int X, int Y)>();
list.Add((1, 2));
list[0].X = 100;   // ❌ Compile error: CS1612
// Indexer списка возвращает копию, а не ссылку. Менять копию бессмысленно — компилятор запрещает.
```

Это та самая боль mutable struct в коллекциях. Решение — либо менять весь tuple целиком (`list[0] = (100, 2);`), либо использовать record / immutable tuple.

### 2.5. ITuple — интерфейс для библиотек

В `System.Runtime.CompilerServices` есть интерфейс:

```csharp
public interface ITuple
{
    int Length { get; }
    object? this[int index] { get; }
}
```

Все `ValueTuple<...>` и `Tuple<...>` реализуют его. Это даёт runtime introspection — можно перебрать поля, не зная их типа на этапе компиляции:

```csharp
using System.Runtime.CompilerServices;

void Print(ITuple t)
{
    for (int i = 0; i < t.Length; i++)
        Console.WriteLine($"[{i}] = {t[i]}");
}

Print((1, "hello", 3.14));
// [0] = 1
// [1] = hello
// [2] = 3.14
```

Используется редко, но удобно для library-кода (логгеры, сериализаторы, generic dump-helpers).

> [!question]- Интервью: где живёт `ValueTuple` — на стеке или на куче?
> Зависит от контекста. Локальная переменная — на стеке (или в регистрах). Поле объекта — внутри layout этого объекта на куче. Параметр метода — копия в стек-фрейме вызываемого. Главное — `ValueTuple` сам по себе **никогда не аллоцируется отдельно на куче** (в отличие от `Tuple<T>`, который class и живёт на куче всегда).

---

## 3. Имена полей — компайл-тайм магия

### 3.1. Named tuple — это не runtime, это compile-time

Ключевая идея, которую нужно усвоить:

```csharp
var p = (X: 1, Y: 2);
Console.WriteLine(p.X);   // 1
```

`X` и `Y` **не существуют в runtime**. В IL и в `ValueTuple<int, int>` поля называются `Item1` и `Item2`. Имена `X`, `Y` — это синтаксический сахар компилятора. Когда ты пишешь `p.X`, компилятор переписывает это как `p.Item1` ещё до генерации байт-кода.

Доказательство:

```csharp
var p = (X: 1, Y: 2);
Console.WriteLine(p.GetType().GetFields().Length);   // 2
Console.WriteLine(p.GetType().GetFields()[0].Name);  // Item1
Console.WriteLine(p.GetType().GetFields()[1].Name);  // Item2
```

`GetFields()` возвращает физические поля, а они называются `Item1`/`Item2`. Имена `X`/`Y` теряются в reflection (если только не смотришь на сигнатуру метода — об этом ниже).

### 3.2. TupleElementNamesAttribute — где живут имена

Имена полей сохраняются — но в виде атрибута на сигнатуре метода / поля / параметра, а не в самом типе. Атрибут называется `TupleElementNamesAttribute`:

```csharp
public (int X, int Y) GetPoint() => (1, 2);
```

компилируется примерно в:

```csharp
[return: TupleElementNames(new[] { "X", "Y" })]
public ValueTuple<int, int> GetPoint() => new ValueTuple<int, int>(1, 2);
```

Это значит, что **имена видны через reflection на сигнатуре**, но не на самом значении:

```csharp
var method = typeof(MyClass).GetMethod("GetPoint");
var attr = method.ReturnParameter.GetCustomAttribute<TupleElementNamesAttribute>();
foreach (var name in attr.TransformNames)
    Console.WriteLine(name);
// X
// Y
```

### 3.3. Имена не сохраняются в локальных переменных

Это ловушка. Имена есть **на сигнатурах** (return type, parameters, fields), но **не на локальных переменных**:

```csharp
public (int X, int Y) GetPoint() => (1, 2);

public void Use()
{
    var p = GetPoint();      // тип p: (int X, int Y) — имена есть!
    Console.WriteLine(p.X);  // ✅

    var anonObj = new object();
    var t = ((object)p);     // тип t: object — имена потеряны
}
```

В метаданных IL для локальной переменной `p` имена сохраняются в специальной debug-информации (для подсветки в IDE и debugger), но в самом IL — типизация на уровне `ValueTuple<int, int>`.

### 3.4. Reflection доступ к именам

Получить имена tuple через reflection — нетривиально, потому что они на атрибуте сигнатуры, а не на типе:

```csharp
public (int Count, decimal Total, string Currency) GetStats() => (5, 100m, "USD");

var method = typeof(StatsService).GetMethod(nameof(StatsService.GetStats));
var attr = method.ReturnParameter
    .GetCustomAttribute<TupleElementNamesAttribute>();

if (attr != null)
{
    for (int i = 0; i < attr.TransformNames.Count; i++)
        Console.WriteLine($"Field {i}: {attr.TransformNames[i]}");
}
// Field 0: Count
// Field 1: Total
// Field 2: Currency
```

Для значения tuple, переданного как `object`, имена **получить нельзя** — они не в типе, а в сигнатуре, через которую значение прошло.

### 3.5. Name mismatch — warning CS8123

Если ты переименовываешь tuple, имена могут не совпасть. Компилятор предупредит:

```csharp
public (int X, int Y) GetPoint() => (1, 2);

(int A, int B) p = GetPoint();
// warning CS8123: The tuple element name 'X' is ignored because
// a different name or no name is specified by the target type '(int A, int B)'.
```

Это работает (имена же compile-time) — присваивание идёт по позиции. Но компилятор подсказывает: «ты переименовал, имена `X`/`Y` отброшены, дальше будут `A`/`B`».

```csharp
Console.WriteLine(p.A);   // ✅ 1
Console.WriteLine(p.X);   // ❌ Compile error: 'p' has no field 'X'
```

> [!question]- Интервью: где хранятся имена полей именованного tuple?
> Имена существуют только на этапе компиляции и в метаданных через `TupleElementNamesAttribute` на возвращаемом типе / параметре / поле. В runtime в самом `ValueTuple` всегда `Item1`, `Item2`. Поэтому через reflection имена доступны только если у тебя есть `MethodInfo` / `FieldInfo`, не само значение.

---

## 4. Создание tuple — все способы

### 4.1. Литерал — основной способ

```csharp
var a = (1, 2);                  // (int, int)
var b = (X: 1, Y: 2);            // (int X, int Y)
var c = (1, "hello", 3.14);      // (int, string, double)
var d = (Id: 1, Name: "Alice");  // (int Id, string Name)
```

Это самый идиоматичный способ. Компилятор сам выводит типы из выражения.

### 4.2. Named — рекомендуемый стиль

```csharp
// ❌ Без имён — читать тяжело
public (int, int, decimal) GetStats() => (10, 5, 100m);

var s = GetStats();
Console.WriteLine($"{s.Item1} of {s.Item2} = {s.Item3}");
// Что значит Item1? Item2? Item3? Угадай.

// ✅ С именами — самодокументируется
public (int Count, int Total, decimal Sum) GetStats() => (10, 5, 100m);

var s = GetStats();
Console.WriteLine($"{s.Count} of {s.Total} = {s.Sum}");
```

**Правило:** в публичных сигнатурах — всегда named. В одноразовых локальных — можно без имён, но если читателю придётся вспоминать «что такое Item2» через 5 строк — лучше с именами.

### 4.3. Tuple projection initializers (C# 7.1+)

С C# 7.1 имена могут «всплывать» из имён переменных или полей:

```csharp
int count = 5;
decimal total = 100m;

var stats = (count, total);
// Тип: (int count, decimal total) — имена выведены из переменных!
Console.WriteLine(stats.count);   // ✅
Console.WriteLine(stats.total);   // ✅
```

То же с полями:

```csharp
class Order
{
    public int Id;
    public decimal Amount;
}

Order o = new();
var t = (o.Id, o.Amount);
Console.WriteLine(t.Id);       // ✅ — имя поля «всплыло»
Console.WriteLine(t.Amount);   // ✅
```

Это сокращает шум в LINQ-проекциях:

```csharp
var summary = orders.Select(o => (o.Id, o.Amount));
// Тип элемента: (int Id, decimal Amount)
foreach (var s in summary)
    Console.WriteLine($"#{s.Id}: {s.Amount}");
```

### 4.4. Конструктор и Tuple.Create — устаревший стиль

```csharp
// Прямой конструктор — работает, но многословно
var a = new ValueTuple<int, string>(1, "hello");

// Tuple.Create — для legacy Tuple<T>, не для ValueTuple
var b = Tuple.Create(1, "hello");   // Tuple<int, string>, class на куче!

// ✅ Современный путь
var c = (1, "hello");   // ValueTuple<int, string>
```

Если видишь `Tuple.Create` в новом коде — это сигнал к рефакторингу. Тип результата — `Tuple<T>`, не `ValueTuple<T>`. Они **не взаимозаменяемы**: разные namespace, разная семантика, разная производительность.

### 4.5. Тип явно vs `var`

```csharp
// Type inference
var a = (1, 2);                       // (int, int)
var b = (X: 1, Y: 2);                 // (int X, int Y)

// Явный тип
(int X, int Y) c = (1, 2);            // ОК, но имена дублируются
(int, int) d = (1, 2);                // без имён

// Несовместимое количество — ошибка
(int, int) e = (1, 2, 3);             // ❌ CS0029: Cannot convert (int, int, int) to (int, int)

// Несовместимый тип элемента — ошибка
(int, string) f = (1, 2);             // ❌ CS0029: Cannot convert int to string
```

Когда стоит явный тип? Когда `var` не виден читателю как tuple — например, если возвращаемый тип неочевиден из имени метода:

```csharp
// var Result = SomeService.Compute();   // что это? int? class? tuple?
// (int Score, bool Passed) Result = SomeService.Compute();   // сразу понятно
```

### 4.6. Tuple > 7 элементов

Большие tuples технически работают, но это сигнал «возможно, это уже структура»:

```csharp
var huge = (1, 2, 3, 4, 5, 6, 7, 8, 9);
Console.WriteLine(huge.Item8);   // 8 — компилятор маскирует Rest.Item1
Console.WriteLine(huge.Item9);   // 9
```

Под капотом это `ValueTuple<int, int, int, int, int, int, int, ValueTuple<int, int>>`. Как только tuple переходит за 4-5 элементов, читаемость стремительно падает. Заводи `record`:

```csharp
public record Stats(int A, int B, int C, int D, int E, int F, int G, int H, int I);
```

> [!question]- Интервью: что произойдёт, если в tuple 10 элементов?
> Компилятор разворачивает в `ValueTuple<T1...T7, ValueTuple<T8, T9, T10>>` — nesting через поле `Rest`. Доступ через `t.Item8` маскирует `t.Rest.Item1`. На практике, если tuple > 4 элементов — это запах «нужен record / class», а не tuple.

---

## 5. Multiple return values

### 5.1. Старый out-стиль

Вот как выглядел типичный API .NET до C# 7.0 (и до сих пор — в `int.TryParse`, `Dictionary.TryGetValue`):

```csharp
public bool TryParseCoord(string input, out int x, out int y)
{
    x = 0;
    y = 0;
    var parts = input.Split(',');
    if (parts.Length != 2) return false;
    return int.TryParse(parts[0], out x) && int.TryParse(parts[1], out y);
}

// Использование (старый стиль — .NET до 4.6)
int x, y;
if (TryParseCoord("10,20", out x, out y))
    Console.WriteLine($"{x}, {y}");

// Использование (C# 6.0+)
if (TryParseCoord("10,20", out int x2, out int y2))
    Console.WriteLine($"{x2}, {y2}");
```

Проблемы:

1. **Параметры портят сигнатуру.** Метод снаружи выглядит как «принимает строку и две целых ссылки», хотя по сути возвращает три значения.
2. **Нельзя использовать в LINQ.** `orders.Select(o => TryParseCoord(o.Address, out int x, out int y))` — `out` запрещён в expression trees, ломает EF Core.
3. **Нельзя await.** `out` несовместим с `async`.
4. **Императивный стиль.** Сначала объяви переменные, потом передай — лишний шум.

### 5.2. Новый tuple-стиль

```csharp
public (bool Success, int X, int Y) TryParseCoord(string input)
{
    var parts = input.Split(',');
    if (parts.Length != 2) return (false, 0, 0);

    if (int.TryParse(parts[0], out int x) && int.TryParse(parts[1], out int y))
        return (true, x, y);

    return (false, 0, 0);
}

var (ok, x, y) = TryParseCoord("10,20");
if (ok) Console.WriteLine($"{x}, {y}");
```

Что выиграли:

- Сигнатура читается как «возвращает три значения».
- Работает в LINQ: `orders.Select(o => TryParseCoord(o.Address))`.
- Работает в `async`: `public async Task<(bool, int, int)> TryParseCoordAsync(...)`.
- Декларативный стиль.

### 5.3. Refactoring: out → tuple

Пошаговый рецепт:

```csharp
// Шаг 1 — было
public bool TryGetUser(int id, out User user)
{
    user = _users.FirstOrDefault(u => u.Id == id);
    return user != null;
}

// Шаг 2 — заменили return type на tuple
public (bool Success, User? User) TryGetUser(int id)
{
    var user = _users.FirstOrDefault(u => u.Id == id);
    return (user != null, user);
}

// Шаг 3 — обновили все callers
// Было:
if (service.TryGetUser(1, out User user))
    Console.WriteLine(user.Name);

// Стало:
var (success, user) = service.TryGetUser(1);
if (success && user != null)
    Console.WriteLine(user.Name);
```

Ловушка: если у tuple первый элемент — `bool Success`, а второй — `User?`, статический анализатор (NRT) не понимает, что `success == true` гарантирует `user != null`. Приходится писать `success && user != null` или использовать null-forgiving `user!`. Эту проблему решает `Result<T, E>` или паттерн-матчинг — см. раздел 15.5.

### 5.4. Когда `out` всё ещё лучше

`out` не умер. Он остаётся идиоматичным в одном случае — **TryParse паттерн в горячем коде**:

```csharp
// Идиоматично — все знают, как это читать
if (int.TryParse(input, out int value))
{
    // используем value
}

// vs tuple — лишний шум
var (success, value) = TryParseInt(input);
if (success)
{
    // используем value
}
```

Для `int.TryParse`, `Dictionary.TryGetValue`, `Span.TryWrite` — `out` остаётся каноном. BCL не будет переписывать эти методы, потому что:

- Они вызываются миллиардами раз — `out` чуть быстрее (нет копии tuple).
- Имя `Try*` + `out` — устоявшаяся идиома, читатель понимает мгновенно.
- Generic API — `out T` работает с любым типом включая `Span<T>`, который не помещается в обычный generic tuple (`ValueTuple<Span<T>>` запрещён — `Span` это `ref struct`).

**Эвристика:** если ты пишешь свой `Try*` метод и он работает с обычными типами — выбирай tuple. Если работает со `Span<T>` или должен быть максимально быстрым (на каждом инструкция считаем) — `out`.

### 5.5. Async + tuple

Возврат tuple из `async` метода — чисто и идиоматично:

```csharp
public async Task<(int StatusCode, string Body)> FetchAsync(string url)
{
    var response = await _http.GetAsync(url);
    var body = await response.Content.ReadAsStringAsync();
    return ((int)response.StatusCode, body);
}

var (status, body) = await FetchAsync("https://example.com");
Console.WriteLine($"HTTP {status}: {body[..50]}...");
```

Можно деконструировать прямо после `await`:

```csharp
var (status, body) = await FetchAsync(url);
```

Особенно красиво с `Task.WhenAll` для параллельных операций:

```csharp
async Task<(User User, List<Order> Orders, decimal Balance)> LoadProfileAsync(int userId)
{
    var userTask = _users.GetAsync(userId);
    var ordersTask = _orders.GetForUserAsync(userId);
    var balanceTask = _accounts.GetBalanceAsync(userId);

    await Task.WhenAll(userTask, ordersTask, balanceTask);

    return (await userTask, await ordersTask, await balanceTask);
}
```

Все три запроса идут параллельно, потом возвращаем как один tuple. На стороне клиента:

```csharp
var (user, orders, balance) = await LoadProfileAsync(42);
```

> [!info]- Если ты знаешь Go
> В Go `func GetUser(id int) (User, error)` — это встроенный язык multiple return, но не tuple. Нельзя положить в переменную, передать как один объект. C# `(User, error)` — first-class value, можно хранить, передавать, складывать в коллекции. По сути C# tuple универсальнее, но дизайн API схожий: возврат `(значение, ошибка)`.

> [!question]- Интервью: чем отличается возврат `(bool, int)` от `out int + bool return`?
> Семантически — то же. Технически: tuple копируется как struct (2 поля = 8-16 байт + IL для упаковки), `out` — прямая запись по адресу. На горячем коде `out` чуть быстрее. Tuple читабельнее в API: видно, что метод возвращает несколько значений, не имеет side-effect-параметров. Tuple работает в LINQ и async, `out` — нет. Для библиотечных hot-path методов BCL оставляет `out`. В прикладном коде — tuple идиоматичнее.

---

## 6. Deconstruction — механика

### 6.1. Что такое деконструкция

Деконструкция — это «развинтить» составной объект на отдельные переменные.

```csharp
var p = (X: 1, Y: 2);

var (x, y) = p;   // деконструкция: x = 1, y = 2
Console.WriteLine($"{x}, {y}");
```

Под капотом компилятор разворачивает это в:

```csharp
ValueTuple<int, int> __tmp = p;
int x = __tmp.Item1;
int y = __tmp.Item2;
```

Это работает для:

1. **Tuples** — встроенная деконструкция (компилятор знает структуру).
2. **Records** — авто-генерируемый `Deconstruct`.
3. **Любых типов с методом `Deconstruct(out ..., out ...)`** — твой класс может «уметь» деконструироваться.

### 6.2. Tuple — встроенная деконструкция

Tuple деконструируется без всяких объявлений `Deconstruct` — это часть языка:

```csharp
var (a, b) = (1, 2);
var (a, b, c) = (1, 2, 3);
var (a, b, c, d) = (1, 2, 3, 4);
// и т.д.
```

Арность должна совпадать:

```csharp
var (a, b) = (1, 2, 3);
// ❌ CS8132: Cannot deconstruct a tuple of '3' elements into '2' variables.
```

### 6.3. Три формы синтаксиса деконструкции

```csharp
var p = (X: 1, Y: 2);

// Форма 1: var (x, y) — все переменные с одним общим var
var (x, y) = p;

// Форма 2: (var x, var y) — каждая переменная отдельно
(var x, var y) = p;

// Форма 3: явные типы — (int x, int y)
(int x, int y) = p;

// Форма 4 (mixed): часть с var, часть с типом
(int x, var y) = p;
```

Все четыре эквивалентны для типизации. Форма 1 (`var (x, y)`) — самая частая и компактная.

**Что нельзя:** смешать **объявление** и **присваивание существующих переменных** в одной деконструкции.

```csharp
int existingX;
// Вариант 1 — все объявляем заново
var (x, y) = p;

// Вариант 2 — все существующие
(existingX, var y) = p;
// ❌ Так нельзя — должны быть либо все объявления, либо все присваивания
```

Правильно — две разные конструкции:

```csharp
// Все существующие
int x, y;
(x, y) = p;

// Все новые
var (x, y) = p;
```

### 6.4. Discard `_` — semantics

Если часть значений не нужна — используй `_`:

```csharp
var (x, _) = (1, 2);            // y отбрасываем
var (_, _, z) = (1, 2, 3);      // оставляем только z
```

Это **discard** — специальная идиома языка. `_` — не переменная, а маркер «здесь значение, но мне оно не нужно». Discard:

- Не выделяет память (компилятор пропускает).
- Не создаёт переменную (повторное использование `_` в той же области — OK).
- Семантически означает «явно проигнорировать».

```csharp
// OK — discard можно повторять
var (_, _) = (1, 2);

// Discard в switch для exhaustiveness
var result = value switch
{
    > 0 => "positive",
    < 0 => "negative",
    _ => "zero"   // discard как "default"
};
```

### 6.5. Существующие переменные — паттерн `(a, b) = ...`

Иногда переменные уже объявлены, а ты хочешь обновить их разом:

```csharp
int a = 0, b = 0;

(a, b) = (10, 20);
// a = 10, b = 20
```

Канонический пример — **swap**:

```csharp
int a = 1, b = 2;
(a, b) = (b, a);
// a = 2, b = 1
```

Это самый красивый swap в C#. Под капотом компилятор создаёт временный tuple, потом распихивает по переменным:

```csharp
// Эквивалент того, что делает компилятор
var __tmp = (b, a);
a = __tmp.Item1;
b = __tmp.Item2;
```

### 6.6. Деконструкция в `foreach` для словарей

С .NET Core 2.0 у `KeyValuePair<K, V>` есть встроенный `Deconstruct`:

```csharp
var ages = new Dictionary<string, int>
{
    ["Alice"] = 30,
    ["Bob"] = 25
};

foreach (var (name, age) in ages)
    Console.WriteLine($"{name}: {age}");

// vs старый стиль
foreach (var kvp in ages)
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
```

Это не магия Dictionary — любой тип с `Deconstruct` может разлагаться в `foreach`. Подробнее — в разделе 8.

> [!question]- Интервью: что такое discard `_` в деконструкции?
> Это специальный маркер языка, означающий «явно отбросить значение». Не создаёт переменную, не аллоцирует память. Можно повторять (`var (_, _) = ...`). Используется в деконструкции, в pattern matching (`_ =>` как «default»), в out-параметрах (`int.TryParse(s, out _)` если результат не нужен).

---

## 7. Deconstruct method — для своих типов

### 7.1. Сигнатура Deconstruct

Чтобы свой класс / struct мог деконструироваться, нужен метод `Deconstruct` с `out`-параметрами:

```csharp
public class Person
{
    public string Name { get; }
    public int Age { get; }

    public Person(string name, int age)
    {
        Name = name;
        Age = age;
    }

    // Конвенция: void Deconstruct(out ...)
    public void Deconstruct(out string name, out int age)
    {
        name = Name;
        age = Age;
    }
}

var p = new Person("Alice", 30);
var (name, age) = p;   // вызывает p.Deconstruct(out name, out age)
Console.WriteLine($"{name}, {age}");
```

Правила:

- Возвращаемый тип — `void`.
- Все параметры — `out`.
- Имя метода — строго `Deconstruct` (case-sensitive).
- Может быть instance-методом или extension-методом.
- Может быть несколько перегрузок с разной арностью.

### 7.2. Несколько Deconstruct — overloading

Один тип может иметь несколько `Deconstruct` для разной арности:

```csharp
public class Order
{
    public int Id { get; init; }
    public string Customer { get; init; } = "";
    public decimal Amount { get; init; }
    public DateTime CreatedAt { get; init; }

    public void Deconstruct(out int id, out decimal amount)
    {
        id = Id;
        amount = Amount;
    }

    public void Deconstruct(out int id, out string customer, out decimal amount)
    {
        id = Id;
        customer = Customer;
        amount = Amount;
    }

    public void Deconstruct(out int id, out string customer, out decimal amount, out DateTime created)
    {
        id = Id;
        customer = Customer;
        amount = Amount;
        created = CreatedAt;
    }
}

var o = new Order { Id = 1, Customer = "Alice", Amount = 100m, CreatedAt = DateTime.UtcNow };

var (id, amount) = o;                            // 2-arity overload
var (id2, customer, amount2) = o;                // 3-arity overload
var (id3, customer2, amount3, created) = o;      // 4-arity overload
```

Компилятор выбирает overload по количеству целевых переменных. Это удобно — можно дать читателю несколько «срезов» одного объекта.

### 7.3. Extension Deconstruct — для чужих типов

Если тип не твой (например, из NuGet-пакета или BCL), можно добавить деконструкцию через extension-метод:

```csharp
// System.Drawing.Point — не твой класс
public static class PointExtensions
{
    public static void Deconstruct(this System.Drawing.Point p, out int x, out int y)
    {
        x = p.X;
        y = p.Y;
    }
}

var p = new System.Drawing.Point(10, 20);
var (x, y) = p;   // ✅ работает через extension
```

Это особенно полезно для:

- Типов из BCL без `Deconstruct` (например, `DateTime`, `TimeSpan`).
- Сторонних библиотек, которые не дают `Deconstruct`.
- Доменных адаптеров — «деконструировать `User` в `(string, string)` для logging».

### 7.4. Records — auto-generated Deconstruct

Positional records получают `Deconstruct` бесплатно:

```csharp
public record Person(string Name, int Age);

var p = new Person("Alice", 30);
var (name, age) = p;   // ✅ компилятор сам сгенерировал Deconstruct
```

Под капотом запись:

```csharp
public record Person(string Name, int Age)
{
    public string Name { get; init; } = Name;
    public int Age { get; init; } = Age;

    // Сгенерировано компилятором:
    public void Deconstruct(out string name, out int age)
    {
        name = Name;
        age = Age;
    }
}
```

Для record class и record struct — поведение одинаковое. Это одна из причин, почему records вытеснили tuples в API: и читаются как tuple (`var (n, a) = person`), и при этом — настоящий тип с именем, equality, ToString.

### 7.5. CS8132 — mismatch arity

Если ты пытаешься деконструировать в неправильное число переменных:

```csharp
public record Person(string Name, int Age);

var p = new Person("Alice", 30);
var (a, b, c) = p;
// ❌ CS8132: Cannot deconstruct a tuple of '2' elements into '3' variables.
```

Компилятор смотрит на доступные `Deconstruct` (включая extension) и выбирает по числу out-параметров. Если ничего не подходит — ошибка.

> [!question]- Интервью: как сделать чтобы свой класс деконструировался?
> Добавить публичный метод `void Deconstruct(out T1, out T2, ...)`. Имя строго `Deconstruct`, все параметры — `out`, возврат — `void`. Можно несколько перегрузок с разной арностью. Если класс чужой — добавить через extension-метод. Records получают `Deconstruct` автоматически для positional-параметров.

---

## 8. Built-in типы с Deconstruct

### 8.1. `KeyValuePair<K, V>`

С .NET Core 2.0 у `KeyValuePair<TKey, TValue>` есть встроенный `Deconstruct`:

```csharp
var kvp = new KeyValuePair<string, int>("answer", 42);
var (key, value) = kvp;
Console.WriteLine($"{key} = {value}");
```

### 8.2. Dictionary в foreach

Самое частое применение — обход словаря:

```csharp
var prices = new Dictionary<string, decimal>
{
    ["apple"] = 1.5m,
    ["bread"] = 3.0m,
    ["milk"] = 2.0m
};

foreach (var (product, price) in prices)
    Console.WriteLine($"{product}: ${price}");

// vs старый стиль
foreach (var kvp in prices)
    Console.WriteLine($"{kvp.Key}: ${kvp.Value}");
```

Под капотом компилятор делает примерно так:

```csharp
foreach (KeyValuePair<string, decimal> __kvp in prices)
{
    __kvp.Deconstruct(out string product, out decimal price);
    Console.WriteLine($"{product}: ${price}");
}
```

### 8.3. DictionaryEntry — legacy

Старый non-generic `Hashtable` использует `DictionaryEntry` (а не `KeyValuePair`). У него **нет** `Deconstruct` в BCL. Можно добавить через extension:

```csharp
public static class DictionaryEntryExtensions
{
    public static void Deconstruct(this DictionaryEntry entry, out object key, out object? value)
    {
        key = entry.Key;
        value = entry.Value;
    }
}

var ht = new Hashtable { ["a"] = 1 };
foreach (DictionaryEntry entry in ht)
{
    var (k, v) = entry;   // через extension
    Console.WriteLine($"{k} = {v}");
}
```

В новом коде `Hashtable` не используется — есть `Dictionary<K, V>`. Но в legacy-системах встречается.

### 8.4. ValueTuple — встроенная деконструкция

`ValueTuple` деконструируется без `Deconstruct` метода — это часть языка:

```csharp
var t = (1, 2, 3);
var (a, b, c) = t;
```

Компилятор знает структуру `ValueTuple` и разворачивает деконструкцию напрямую в обращение к `Item1`, `Item2`, `Item3`.

### 8.5. Что ещё деконструируется в BCL

Список встроенной поддержки в современном .NET:

- `KeyValuePair<K, V>` — `(key, value)`
- `ValueTuple<...>` — все арности
- `Tuple<...>` — нет (но есть extension в `System.ValueTuple` пакете для legacy кода)

Большинство типов BCL — без `Deconstruct`. Это сделано осознанно: деконструкция — это спецификация публичного API, и BCL не хочет фиксировать «вот эти три поля важнейшие». Поэтому `DateTime`, `TimeSpan`, `Guid` — без деконструкции.

Но ничто не мешает добавить через extension:

```csharp
public static class DateTimeExtensions
{
    public static void Deconstruct(this DateTime dt, out int year, out int month, out int day)
    {
        year = dt.Year;
        month = dt.Month;
        day = dt.Day;
    }
}

var (year, month, day) = DateTime.Today;
Console.WriteLine($"{year}-{month}-{day}");
```

Удобно, если в проекте такой паттерн часто нужен.

> [!question]- Интервью: как `foreach (var (k, v) in dict)` работает под капотом?
> `Dictionary<K, V>.GetEnumerator()` возвращает `KeyValuePair<K, V>` на каждой итерации. У `KeyValuePair` есть метод `Deconstruct(out K, out V)`. Компилятор для каждой итерации вызывает `Deconstruct` и присваивает результат `k` и `v`. Это работает с любым типом, у которого есть `Deconstruct` правильной арности — не только с `KeyValuePair`.

---

## 9. Pattern matching и tuples — кратко

Tuples и pattern matching — близкие родственники. Полный разбор паттернов — в [[modern-features|Modern Features]], здесь — только tuple-specifics.

### 9.1. Positional pattern в switch

```csharp
var point = (X: 1, Y: 2);

string description = point switch
{
    (0, 0) => "origin",
    (var x, 0) => $"on X axis at {x}",
    (0, var y) => $"on Y axis at {y}",
    (var x, var y) when x == y => $"diagonal at {x}",
    (var x, var y) => $"at ({x}, {y})"
};
```

Каждая `case`-ветка деконструирует tuple и сравнивает по позиции. Можно смешивать константы (`0`), discard (`_`), and capture-переменные (`var x`).

### 9.2. Positional pattern для своего класса

Работает с любым типом, у которого есть `Deconstruct`:

```csharp
public record Order(int Id, string Status, decimal Amount);

string Describe(Order o) => o switch
{
    (_, "draft", _) => "черновик",
    (_, "paid", > 1000m) => "крупный оплачен",
    (_, "paid", _) => "оплачен",
    (_, "cancelled", _) => "отменён",
    _ => "неизвестно"
};
```

Под капотом — тот же `Deconstruct`. Pattern matching просто использует его как механизм декомпозиции.

### 9.3. Tuple в switch expression

Tuple естественно ложится на много условий:

```csharp
int day = 3;
bool isWeekend = (day) switch
{
    6 or 7 => true,
    _ => false
};

// vs tuple — два условия
bool gameEnded = (homeScore: 3, awayScore: 1, time: 90) switch
{
    (_, _, < 90) => false,
    (var h, var a, _) when h == a => false,   // ничья — продолжаем
    _ => true
};
```

### 9.4. List patterns + tuples (C# 11+)

```csharp
int[] arr = [1, 2, 3, 4];

string Describe(int[] a) => a switch
{
    [] => "пусто",
    [var x] => $"один элемент: {x}",
    [var first, .., var last] => $"первый {first}, последний {last}",
    _ => "что-то ещё"
};
```

Это уже не tuple, но та же ментальная модель: декомпозиция структуры по позиции. Углубляться не буду — это епархия [[modern-features|Modern Features]].

> [!question]- Интервью: как работает positional pattern для своего класса?
> Компилятор ищет метод `Deconstruct` нужной арности (включая extension). Деконструирует значение, сравнивает каждый out-параметр с соответствующим под-паттерном. Если все совпали — case срабатывает. Если `Deconstruct` нет или нет нужной арности — ошибка компиляции CS1503.

---

## 10. Tuple equality

### 10.1. Operator == — value equality (C# 7.3+)

С C# 7.3 операторы `==` и `!=` для tuples сравнивают **по значению поэлементно**:

```csharp
var a = (1, 2);
var b = (1, 2);

Console.WriteLine(a == b);   // True
Console.WriteLine(a != b);   // False
```

Под капотом компилятор разворачивает `a == b` в:

```csharp
a.Item1 == b.Item1 && a.Item2 == b.Item2
```

То есть для каждой пары используется операторы `==`/`!=` соответствующих типов. Никакой магии — обычное поэлементное сравнение.

### 10.2. Equals — то же

`Equals` тоже работает по value, как и `==`:

```csharp
var a = (1, 2);
var b = (1, 2);

a.Equals(b);   // True
```

Это потому, что `ValueTuple<T1, T2>` реализует `IEquatable<ValueTuple<T1, T2>>` с поэлементным сравнением.

Для сложных типов внутри — сравнение делегируется их `Equals`:

```csharp
var a = (Name: "Alice", Age: 30);
var b = (Name: "Alice", Age: 30);

a == b;   // True — string.Equals + int.Equals
```

### 10.3. GetHashCode — combined hash

`HashCode.Combine` (или встроенная реализация) объединяет хэши элементов:

```csharp
var t = (1, "hello", 3.14);
var hash = t.GetHashCode();
// Под капотом: HashCode.Combine(t.Item1, t.Item2, t.Item3)
```

Это даёт корректную работу в `HashSet<>` и `Dictionary<,>`:

```csharp
var seen = new HashSet<(int, int)>();
seen.Add((1, 2));
seen.Add((1, 2));
seen.Add((2, 1));
Console.WriteLine(seen.Count);   // 2 — (1,2) и (2,1) разные

var byCoord = new Dictionary<(int X, int Y), string>
{
    [(0, 0)] = "origin",
    [(1, 0)] = "right",
    [(0, 1)] = "up"
};
Console.WriteLine(byCoord[(0, 0)]);   // origin
```

### 10.4. Имена ИГНОРИРУЮТСЯ при сравнении

Это ловушка. Equality смотрит **на типы и значения**, а не на имена полей:

```csharp
(int X, int Y) a = (1, 2);
(int A, int B) b = (1, 2);

a == b;   // True — имена не учитываются
```

Это логично (имена — compile-time, в runtime их нет), но иногда удивляет:

```csharp
(int Year, int Month) date = (2024, 5);
(int Hour, int Minute) time = (2024, 5);

date == time;   // True — но это бессмыслица семантически
```

Защита — типизация. Если хочешь, чтобы `(Year, Month)` нельзя было перепутать с `(Hour, Minute)`, заводи `record`:

```csharp
public record YearMonth(int Year, int Month);
public record HourMinute(int Hour, int Minute);

YearMonth a = new(2024, 5);
HourMinute b = new(2024, 5);

a == b;   // ❌ CS0019: Operator '==' cannot be applied to 'YearMonth' and 'HourMinute'
```

### 10.5. Tuples с разной арностью — нельзя

```csharp
var a = (1, 2);
var b = (1, 2, 3);

a == b;
// ❌ CS0019: Operator '==' cannot be applied to operands of type
//            '(int, int)' and '(int, int, int)'
```

Это compile-time ошибка — арности разные, типы разные.

### 10.6. Tuple с разными типами — implicit conversion

Если типы совместимы (есть implicit conversion), сравнение работает:

```csharp
var a = (1, 2);       // (int, int)
var b = (1L, 2L);     // (long, long)

a == b;   // True — int → long widening, потом сравнение
```

Если несовместимы:

```csharp
var a = (1, 2);
var b = (1, "hello");

a == b;
// ❌ CS0019: 'int' и 'string' нельзя сравнить через ==
```

### 10.7. Custom equality — нельзя переопределить

`ValueTuple` — это struct из BCL, ты не можешь добавить свой `==`/`!=`. Если нужна особая логика равенства (например, сравнение с допуском):

```csharp
// Не tuple, а record с custom Equals
public record Point(double X, double Y)
{
    public virtual bool Equals(Point? other) =>
        other != null
        && Math.Abs(X - other.X) < 1e-9
        && Math.Abs(Y - other.Y) < 1e-9;

    public override int GetHashCode() => HashCode.Combine(X, Y);
}
```

Tuple — это «как есть». Хочешь свою логику — record / class.

> [!question]- Интервью: что вернёт `(1, 2) == (1, 2)`?
> `True`. С C# 7.3 операторы `==`/`!=` для tuples сравнивают поэлементно. Имена при этом игнорируются — `(X: 1, Y: 2) == (A: 1, B: 2)` тоже `True`. Если арность разная или типы несовместимы — compile-time ошибка.

---

## 11. Conversions

### 11.1. Implicit widening

Tuple-to-tuple конверсии работают, если для каждой пары элементов есть implicit conversion:

```csharp
(int, int) ints = (1, 2);
(long, long) longs = ints;   // ✅ int → long widening для каждого элемента
(double, double) doubles = ints;   // ✅ int → double

(int, string) mixed = (1, "hello");
(long, string) widened = mixed;   // ✅ int → long, string без изменений
```

Если хотя бы для одной пары нет implicit conversion — ошибка:

```csharp
(int, int) a = (1, 2);
(long, string) b = a;
// ❌ Cannot implicitly convert int to string
```

### 11.2. Имена меняются свободно

Имена — compile-time. Их можно «переименовать» при присваивании:

```csharp
(int X, int Y) point = (1, 2);
(int Width, int Height) size = point;   // ✅ работает, но warning CS8123
```

Семантически некорректно (точка — не размер), но компилятор это пропускает с warning. Анализаторы в проекте часто настроены поднимать CS8123 до error.

### 11.3. Custom implicit operator с tuple

Можно сделать свой тип, который автоконвертится из tuple:

```csharp
public readonly struct Coord
{
    public int X { get; }
    public int Y { get; }

    public Coord(int x, int y) { X = x; Y = y; }

    // tuple → Coord
    public static implicit operator Coord((int X, int Y) tuple) =>
        new(tuple.X, tuple.Y);

    // Coord → tuple
    public static implicit operator (int X, int Y)(Coord c) =>
        (c.X, c.Y);
}

Coord a = (1, 2);          // ✅ автоконверсия
(int X, int Y) b = a;      // ✅ обратно

Console.WriteLine(a.X);    // 1
Console.WriteLine(b.Y);    // 2
```

Это даёт «легковесный конструктор» — особенно удобно для типов значений в DDD (Value Objects).

### 11.4. Cast и as — нельзя для tuple

```csharp
object o = (1, 2);

var t = (ValueTuple<int, int>)o;   // ✅ обычный cast object → struct (unboxing)
var t2 = o as (int, int)?;         // ✅ as для nullable struct
var t3 = (int, int)o;              // ❌ синтаксис неоднозначен — это не сработает
```

Tuple — обычный struct, поэтому boxing / unboxing работает как с любым value type. Но синтаксический сахар `(...)` для tuple-литерала иногда конфликтует с cast. Если компилятор путается — пиши через `ValueTuple<>` явно.

> [!info]- Если ты знаешь Python
> В Python `(1, 2)` — встроенный `tuple`, immutable. Можно класть в `set` и в ключи `dict` (если все элементы хэшируемы). C# `ValueTuple` тут точно так же — value equality, hashable, годится как Dictionary key. Различия: C# tuple строго типизирован (`ValueTuple<int, int>` ≠ `ValueTuple<int, string>`), Python — динамический.

---

## 12. Tuples в LINQ

### 12.1. GroupBy с composite key

Самое мощное применение — composite key:

```csharp
var orders = new[]
{
    new { CustomerId = 1, Year = 2024, Amount = 100m },
    new { CustomerId = 1, Year = 2024, Amount = 50m },
    new { CustomerId = 2, Year = 2024, Amount = 200m },
    new { CustomerId = 1, Year = 2023, Amount = 75m }
};

var grouped = orders
    .GroupBy(o => (o.CustomerId, o.Year))
    .Select(g => new
    {
        Customer = g.Key.CustomerId,
        Year = g.Key.Year,
        Total = g.Sum(o => o.Amount)
    });

foreach (var g in grouped)
    Console.WriteLine($"Customer {g.Customer} in {g.Year}: ${g.Total}");
// Customer 1 in 2024: $150
// Customer 2 in 2024: $200
// Customer 1 in 2023: $75
```

`GroupBy` работает потому, что у tuple корректный `Equals` + `GetHashCode`. Composite key «бесплатно».

### 12.2. Aggregate с tuple-аккумулятором

```csharp
int[] numbers = [3, 7, 1, 9, 4, 2, 8, 5];

var (min, max, sum, count) = numbers.Aggregate(
    seed: (Min: int.MaxValue, Max: int.MinValue, Sum: 0, Count: 0),
    func: (acc, n) => (
        Math.Min(acc.Min, n),
        Math.Max(acc.Max, n),
        acc.Sum + n,
        acc.Count + 1
    )
);

Console.WriteLine($"min={min}, max={max}, avg={sum / (double)count:F2}");
// min=1, max=9, avg=4.88
```

Один проход по коллекции — четыре статистики. Без tuple пришлось бы либо четыре раза `Aggregate` (4 прохода), либо отдельный класс-аккумулятор.

### 12.3. Select projection — anonymous vs tuple

```csharp
// Вариант 1 — anonymous type
var withAnon = orders.Select(o => new { o.Id, o.Amount });

// Вариант 2 — tuple
var withTuple = orders.Select(o => (o.Id, o.Amount));
```

Когда что:

| | Anonymous type | Tuple |
|---|---|---|
| Тип элемента | `<>f__AnonymousType0<int, decimal>` | `(int Id, decimal Amount)` |
| Можно ли вернуть из метода | ❌ нет имени типа | ✅ да |
| Read-only | ✅ properties immutable | ⚠️ fields mutable |
| Equality | ✅ value equality | ✅ value equality |
| EF Core support | ✅ полная | ⚠️ ограниченная |
| ToString | `{ Id = 1, Amount = 100 }` | `(1, 100)` |

Правило: **внутри одного LINQ-выражения** — anonymous type (особенно если потом передаём в EF Core). **Если надо вернуть из метода** — tuple. **Если живёт дольше LINQ-выражения** — record.

### 12.4. EF Core ограничения

EF Core умеет переводить anonymous types в SQL, но **с tuples — частичная поддержка**:

```csharp
// ✅ Anonymous type — гарантированно работает
var query = db.Users
    .Where(u => u.IsActive)
    .Select(u => new { u.Id, u.Name });

// ⚠️ Tuple — может не транслироваться
var query2 = db.Users
    .Where(u => u.IsActive)
    .Select(u => (u.Id, u.Name));
// На некоторых провайдерах кидает InvalidOperationException
// "could not be translated"
```

Это особенность EF Core: anonymous type — давний контракт, провайдер знает, как развернуть. Tuple появился позже, и поддержка добавлялась постепенно. С EF Core 7+ многое работает, но не всё. Если получил «could not be translated» — переключись на anonymous type.

### 12.5. Composite Dictionary lookup

Tuple идеально как ключ многомерных кэшей:

```csharp
// Кэш курсов: (валюта_из, валюта_в) → курс
var rates = new Dictionary<(string From, string To), decimal>
{
    [("USD", "EUR")] = 0.92m,
    [("USD", "RUB")] = 95m,
    [("EUR", "RUB")] = 103m
};

decimal Convert(decimal amount, string from, string to)
{
    if (from == to) return amount;
    if (rates.TryGetValue((from, to), out var rate))
        return amount * rate;
    if (rates.TryGetValue((to, from), out var inverse))
        return amount / inverse;
    throw new InvalidOperationException($"No rate for {from} → {to}");
}

Console.WriteLine(Convert(100, "USD", "EUR"));   // 92
Console.WriteLine(Convert(100, "RUB", "USD"));   // 1.05 (через обратный курс)
```

### 12.6. Zip с tuple

`Zip` объединяет две последовательности в tuple:

```csharp
string[] names = ["Alice", "Bob", "Carol"];
int[] ages = [30, 25, 35];

foreach (var (name, age) in names.Zip(ages))
    Console.WriteLine($"{name}: {age}");
// Alice: 30
// Bob: 25
// Carol: 35
```

С .NET 6+ есть Zip с тремя последовательностями (возвращает 3-tuple).

> [!question]- Интервью: почему `db.Users.Select(u => new { u.Id })` работает, а `db.Users.Select(u => (u.Id, u.Name))` иногда — нет?
> Anonymous types — давний механизм EF Core, провайдер всегда знает, как разобрать выражение. ValueTuple появился в C# 7.0, поддержка в EF Core добавлялась постепенно. С EF Core 7+ большинство сценариев работает, но edge cases (вложенные tuples, deconstruction в Where) могут не транслироваться. Безопасный путь — anonymous type для проекций, tuple для многих других LINQ-сценариев.

---

## 13. JSON и сериализация

### 13.1. System.Text.Json + tuple — Item1, Item2

По умолчанию `System.Text.Json` сериализует tuple как `Item1`, `Item2`, теряя имена:

```csharp
using System.Text.Json;

var t = (Name: "Alice", Age: 30);
string json = JsonSerializer.Serialize(t);
Console.WriteLine(json);
// {"Item1":"Alice","Item2":30}
```

Это потому, что имена — compile-time, а сериализатор работает с runtime-полями `ValueTuple`. Аналогично десериализация:

```csharp
var json = "{\"Item1\":\"Alice\",\"Item2\":30}";
var t = JsonSerializer.Deserialize<(string, int)>(json);
Console.WriteLine($"{t.Item1}, {t.Item2}");   // Alice, 30
```

**Вывод:** для API, где JSON виден внешнему миру (frontend, чужие сервисы) — **никогда не возвращай tuple**. Используй record / DTO.

### 13.2. Newtonsoft.Json + tuple

То же самое — `Item1`, `Item2`:

```csharp
using Newtonsoft.Json;

var t = (Name: "Alice", Age: 30);
string json = JsonConvert.SerializeObject(t);
// {"Item1":"Alice","Item2":30}
```

Иногда видишь в legacy-кодах кастомные `JsonConverter` для tuples — не пиши такие, лучше переделать в record.

### 13.3. Custom JsonConverter — антипаттерн

Можно написать `JsonConverter`, который читает имена через reflection (`TupleElementNamesAttribute`), но:

- Имена доступны только если есть `MemberInfo` (например, на свойстве класса).
- Для голого `ValueTuple` — никаких имён.
- Сложно поддерживать.

Если уже сериализуешь tuple — сигнал «нужен record».

```csharp
// ❌ Плохо — tuple наружу, имена потеряны
public class StatsService
{
    public (int Total, decimal Sum) GetStats() => (5, 100m);
}

// ✅ Хорошо — record, имена сохранятся в JSON
public record Stats(int Total, decimal Sum);

public class StatsService
{
    public Stats GetStats() => new(5, 100m);
}

// JSON: {"Total":5,"Sum":100}
```

### 13.4. XmlSerializer и tuple

`XmlSerializer` **не поддерживает** tuples (требует public default constructor + public properties). С record — без проблем (с `[XmlElement]` или public init):

```csharp
[Serializable]
public record Stats(int Total, decimal Sum);
```

### 13.5. Best practice по сериализации

| Контекст | Используй |
|----------|-----------|
| Public REST API response | record / DTO |
| Public REST API request | record / DTO |
| gRPC / Protobuf message | сгенерированный класс |
| Internal cache (binary) | tuple OK (но проверь, что сериализатор справляется) |
| Logs / diagnostics dump | tuple OK (Item1, Item2 нормально для разработчика) |
| Сохранение в БД через EF Core | record / class (entity) |
| Json column (EF Core 8+) | record / class, не tuple |

> [!question]- Интервью: что произойдёт, если сериализовать в JSON `(Name: "Alice", Age: 30)` через `System.Text.Json`?
> Получишь `{"Item1":"Alice","Item2":30}` — имена `Name`, `Age` теряются, потому что они только compile-time. В runtime у `ValueTuple` поля называются `Item1`, `Item2`. Для публичных API tuple использовать нельзя — нужен record / DTO.

---

## 14. Performance

### 14.1. Stack vs heap — где разница

```csharp
// ValueTuple — стек, ноль аллокаций
public (int X, int Y) GetPoint() => (1, 2);

// Tuple<int, int> — куча, аллокация на каждом вызове
public Tuple<int, int> GetPointLegacy() => Tuple.Create(1, 2);
```

Бенчмарк (примерно, BenchmarkDotNet, .NET 8):

```
| Method            |     Mean | Allocated |
|------------------ |---------:|----------:|
| GetValueTuple     |  0.89 ns |       0 B |
| GetTupleLegacy    | 12.40 ns |      32 B |
```

Разница 14x по времени и 32 байта аллокации на каждый вызов у `Tuple<T>`. На 1 млн вызовов в секунду — это 32 МБ/сек GC pressure.

### 14.2. Copy cost при передаче

`ValueTuple` копируется при передаче в метод — это struct:

```csharp
public void Process((int A, int B, int C, int D, int E) data)
{
    // На входе data копируется: 5 × 4 байта = 20 байт
}
```

Для маленьких tuple (2-3 элемента) — это никогда не bottleneck. Для больших (>4 элементов с большими типами) — может стать заметно. Лечится `in`-параметром или `ref readonly`:

```csharp
// Передача по ссылке без копии
public void Process(in (int A, int B, int C, int D, int E) data)
{
    Console.WriteLine(data.A);   // через managed reference
}
```

`in` — readonly reference, обещаешь не менять. Полезно для tuples от 4-5 элементов.

### 14.3. `ValueTuple<Span<T>>` — нельзя

```csharp
ValueTuple<Span<int>, int> t = (stackalloc int[10], 5);
// ❌ CS0306: ValueTuple<T, T> requires struct, but Span<int> is a ref struct
```

`ValueTuple` — обычный generic struct. Параметры типа не могут быть `ref struct` (`Span<T>`, `ReadOnlySpan<T>`, `RefStructEnumerator`). Это ограничение языка — `ref struct` не может быть полем обычного struct.

Если нужно вернуть несколько значений включая `Span<T>` — используй `ref struct`:

```csharp
public ref struct ParseResult
{
    public Span<byte> Header;
    public Span<byte> Body;
    public int Length;
}
```

Или — обычный `out`-параметр для `Span<T>`:

```csharp
public bool TryParse(ReadOnlySpan<byte> input, out ReadOnlySpan<byte> result)
{
    // ...
}
```

### 14.4. Boxing — скрытый случай

ValueTuple — struct, поэтому в `object` или `IComparable` боксится:

```csharp
var t = (1, 2);
object o = t;            // boxing! 24 байта на куче
IComparable c = t;       // boxing!
```

Один раз — мелочь. На горячем коде с миллионами операций — заметно. Где boxing скрытый:

```csharp
// Боксинг в ToString interpolation для object
object o = (1, 2);
Console.WriteLine($"{o}");   // ToString вызывается без boxing, но...

// Боксинг при сохранении в non-generic collection
var ar = new ArrayList();
ar.Add((1, 2));   // boxing!

// Боксинг в Dictionary<T, object>
var d = new Dictionary<int, object>();
d[0] = (1, 2);   // boxing!
```

Решение — generic коллекции с типом tuple:

```csharp
var d = new Dictionary<int, (int X, int Y)>();
d[0] = (1, 2);   // без boxing, struct хранится напрямую
```

### 14.5. Tuple в hot path — что взвесить

Когда tuple в критичном по производительности коде:

✅ **Хорошо:**
- Возврат 2-3 примитивных значений из часто вызываемого метода
- Composite key в `Dictionary` / `HashSet` (вместо строки `$"{x}_{y}"`)
- Batch-обработка с агрегацией в tuple-аккумуляторе

⚠️ **Плохо:**
- Tuple > 4-5 элементов с большими структурами внутри (copy дороже)
- Boxing (в `object`, `ArrayList`, `IComparable`)
- Tuple через `IEnumerable<object>` — каждый элемент боксится

### 14.6. Микробенчмарк: tuple vs out vs result class

```csharp
// Сценарий: TryParse-like метод, вызывается миллион раз

[Benchmark]
public int OutVersion()
{
    int total = 0;
    for (int i = 0; i < 1_000_000; i++)
    {
        TryParseOut("42", out int v);
        total += v;
    }
    return total;
}

[Benchmark]
public int TupleVersion()
{
    int total = 0;
    for (int i = 0; i < 1_000_000; i++)
    {
        var (_, v) = TryParseTuple("42");
        total += v;
    }
    return total;
}

// Результат на .NET 8 (примерно):
// OutVersion:    1.2 ms,    0 B alloc
// TupleVersion:  1.4 ms,    0 B alloc
```

Разница ~15%. На обычном коде — не имеет значения. На hot path микрооптимизации tuple проигрывает `out` буквально на копии 8 байт. Для большинства кода — **выбирай tuple за читаемость**.

### 14.7. Когда брать record вместо tuple ради perf

В одной редкой ситуации `record class` бывает быстрее tuple — когда:

- Tuple передаётся ОЧЕНЬ много раз через много методов.
- Tuple большой (5+ полей).
- Передача по ссылке исключена (например, async-методы — `ref` через `await` не проходит).

В этом случае `record class` (один раз heap-аллокация, дальше передача ссылки) может выиграть у tuple (копирование на каждой границе). Но это редкий edge case — сначала меряй, потом меняй.

> [!question]- Интервью: почему `ValueTuple<Span<int>, int>` не компилируется?
> `ValueTuple` — обычный generic struct. Поля у обычного struct не могут быть `ref struct` (это `Span<T>`, `ReadOnlySpan<T>`). Ограничение языка: `ref struct` живёт только на стеке, а обычный struct может оказаться в куче (как поле класса), что нарушит инвариант. Решение — свой `ref struct` с `Span`-полями или раздельный возврат через `out`-параметр.

---

## 15. Tuples vs альтернативы — глубокое сравнение

### 15.1. Tuple vs class

```csharp
// Tuple — упаковка значений на месте
var p = (X: 1, Y: 2);

// Class — формальный тип с поведением
public class Coordinate
{
    public int X { get; }
    public int Y { get; }

    public Coordinate(int x, int y) { X = x; Y = y; }

    public double DistanceTo(Coordinate other) =>
        Math.Sqrt(Math.Pow(X - other.X, 2) + Math.Pow(Y - other.Y, 2));
}
```

| | Tuple | Class |
|---|---|---|
| Аллокация | Стек | Куча |
| Equality | Value (по полям) | Reference по умолчанию |
| Поведение (методы) | Нет | Можно |
| Inheritance | Нет | Можно |
| Validation в ctor | Нет | Можно |
| Identity | Нет | Есть (по reference) |
| Создание | `(1, 2)` | `new Coord(1, 2)` |
| Деконструкция | Встроена | Через `Deconstruct` |

**Бери class когда:**
- Доменная сущность с identity (`User`, `Order`).
- Имеет поведение (методы валидации, бизнес-логика).
- Изменяемое состояние, разделяемое между потоками.
- Иерархия (`Animal → Dog`, `Shape → Circle`).

**Бери tuple когда:**
- Локальная пара/тройка значений.
- Одноразовое использование внутри метода или класса.
- Composite key.

### 15.2. Tuple vs record

```csharp
// Tuple
var personT = (Name: "Alice", Age: 30);

// Record
public record PersonR(string Name, int Age);
var personR = new PersonR("Alice", 30);
```

| | Tuple | Record |
|---|---|---|
| Тип | `ValueTuple<T1, T2>` | Именованный record |
| Аллокация (default) | Стек | Куча (record class) или стек (record struct) |
| Equality | Value | Value |
| Имена полей | Compile-time | Runtime (свойства) |
| ToString | `(Alice, 30)` | `PersonR { Name = Alice, Age = 30 }` |
| Можно ли вернуть из API | Технически да, плохая идея | Да, идиоматично |
| With-выражения | Нет | Да: `person with { Age = 31 }` |
| Inheritance | Нет | Да (record class) |
| Defs в классе/неймспейсе | Нет | Да |
| Эволюция (добавить поле) | Сломает callers | Можно с init / required |

**Бери record когда:**
- Публичный API.
- DTO для JSON/REST/GraphQL.
- Value Object в DDD.
- Тип живёт дольше одного метода.
- Хочешь `with`-выражения для создания модифицированной копии.
- Нужна именованная сущность в системе типов.

**Бери tuple когда:**
- Локальный scope.
- Одноразовое использование.
- Не хочется заводить «ещё один файл с record».

### 15.3. Tuple vs anonymous type

Полное сравнение — в [[anonymous-types|Anonymous Types]]. Краткое:

| | Tuple | Anonymous type |
|---|---|---|
| Тип | Имеет имя `ValueTuple<...>` | Сгенерированный `<>f__AnonymousType0` |
| Возврат из метода | ✅ Да | ❌ Нет (тип не назовёшь) |
| EF Core projection | ⚠️ Частично | ✅ Полностью |
| Mutability | Mutable fields | Immutable properties |
| Реюз структуры | По типам и арности | По именам, типам, порядку (compiler reuses) |

**Эвристика:** anonymous type — внутри LINQ-выражения. Tuple — везде ещё.

### 15.4. Tuple vs out parameters

```csharp
// out
public bool TryGetUser(int id, out User user)
{
    user = _store.Find(id);
    return user != null;
}

// tuple
public (bool Found, User? User) TryGetUser(int id)
{
    var user = _store.Find(id);
    return (user != null, user);
}
```

| | out | Tuple |
|---|---|---|
| Использование в LINQ | ❌ | ✅ |
| Использование в async | ❌ | ✅ |
| Чтение сигнатуры | Сложнее | Естественнее |
| Производительность | Чуть быстрее | Чуть медленнее (копия struct) |
| NRT-связь между параметрами | Через атрибуты | Сложно |
| BCL convention | `int.TryParse`, `Dict.TryGetValue` | Прикладной код |

### 15.5. Tuple vs `Result<T, E>`

Для возврата «успех/ошибка» лучше чем tuple — `Result<T, E>` (паттерн из functional programming):

```csharp
// Tuple — слабая семантика
public (bool Ok, User? User, string? Error) GetUser(int id) { ... }

var (ok, user, error) = GetUser(1);
if (ok && user != null)
    Console.WriteLine(user.Name);
else
    Console.WriteLine(error);

// Result — сильная семантика
public Result<User, string> GetUser(int id) { ... }

var result = GetUser(1);
if (result.IsSuccess)
    Console.WriteLine(result.Value.Name);
else
    Console.WriteLine(result.Error);
```

Преимущества Result:

- Невозможны «оба null» или «и user, и error» (sum type).
- Имеет методы `Map`, `Bind`, `Match` для composability.
- Хорошо ложится на pattern matching.

Подробнее — [[error-handling|Error Handling]] раздел про Result/OneOf.

### 15.6. Decision matrix — финальная

```
Возвращаешь несколько значений?
├── Из приватного / internal метода
│   └── tuple
│
├── Из public API (библиотека, REST, gRPC)
│   └── record / DTO
│
├── Из Try*-метода BCL-стиля
│   └── out (или tuple, если новый API)
│
├── Из метода с success/failure семантикой
│   ├── только success/failure флаг
│   │   └── bool + tuple, или `Result<T, E>`
│   └── ошибка с деталями (код, сообщение)
│       └── `Result<T, E>` или OneOf<T, Error>

Группа значений на месте?
├── Внутри LINQ-проекции
│   └── anonymous type (если в EF Core) или tuple
│
├── Composite key для Dictionary / GroupBy
│   └── tuple
│
└── Локальная переменная
    └── tuple

Свой именованный тип нужен?
├── Domain concept (User, Order)
│   └── class / record
│
├── Value Object (Money, Email)
│   └── record (с валидацией в ctor)
│
└── Просто пара значений
    └── tuple

Эволюция API нужна?
├── Да — добавлять/удалять поля без breaking
│   └── record (с init/required) или class
│
└── Нет — стабильный контракт
    └── tuple допустим (если internal)
```

---

## 16. Real-world patterns — case studies

### 16.1. Multi-result service method

```csharp
public class OrderService
{
    public (decimal Subtotal, decimal Tax, decimal Discount, decimal Total) Calculate(Order order)
    {
        var subtotal = order.Items.Sum(i => i.Price * i.Quantity);
        var tax = subtotal * 0.20m;
        var discount = subtotal > 1000m ? subtotal * 0.1m : 0;
        var total = subtotal + tax - discount;
        return (subtotal, tax, discount, total);
    }
}

// Использование
var (subtotal, tax, discount, total) = service.Calculate(order);
Console.WriteLine($"Subtotal: {subtotal}, Tax: {tax}, Discount: {discount}, Total: {total}");
```

Это OK для internal сервиса. Но если результат уезжает в JSON / шаблон письма / отчёт — лучше record:

```csharp
public record OrderCalculation(decimal Subtotal, decimal Tax, decimal Discount, decimal Total);

public OrderCalculation Calculate(Order order) { ... }
```

### 16.2. Composite cache key

```csharp
public class ExchangeRateCache
{
    private readonly Dictionary<(string From, string To, DateOnly Date), decimal> _cache = new();

    public bool TryGet(string from, string to, DateOnly date, out decimal rate) =>
        _cache.TryGetValue((from, to, date), out rate);

    public void Set(string from, string to, DateOnly date, decimal rate) =>
        _cache[(from, to, date)] = rate;
}

// Использование
var cache = new ExchangeRateCache();
cache.Set("USD", "EUR", new DateOnly(2024, 5, 15), 0.92m);

if (cache.TryGet("USD", "EUR", new DateOnly(2024, 5, 15), out var rate))
    Console.WriteLine($"Rate: {rate}");
```

Без tuple пришлось бы либо строковый ключ `$"{from}_{to}_{date:yyyyMMdd}"` (медленно, alloc-heavy), либо отдельный класс ключа с `Equals`/`GetHashCode` overrides. Tuple даёт всё бесплатно.

### 16.3. Pipeline aggregation

```csharp
record SalesEvent(int ProductId, int Quantity, decimal Price);

var events = new SalesEvent[]
{
    new(1, 2, 100m),
    new(1, 1, 100m),
    new(2, 5, 50m),
    new(1, 3, 100m),
    new(2, 2, 50m)
};

// Группировка + агрегация в одном проходе
var summary = events
    .GroupBy(e => e.ProductId)
    .Select(g =>
    {
        var (totalQty, totalRevenue) = g.Aggregate(
            (Qty: 0, Revenue: 0m),
            (acc, e) => (acc.Qty + e.Quantity, acc.Revenue + e.Quantity * e.Price)
        );
        return new { ProductId = g.Key, Quantity = totalQty, Revenue = totalRevenue };
    });

foreach (var s in summary)
    Console.WriteLine($"Product {s.ProductId}: {s.Quantity} units, ${s.Revenue}");
// Product 1: 6 units, $600
// Product 2: 7 units, $350
```

Tuple-аккумулятор внутри `Aggregate` — компактно и без heap-аллокаций (ValueTuple на стеке).

### 16.4. State machine с (state, transition)

```csharp
enum OrderState { Draft, Submitted, Paid, Shipped, Delivered, Cancelled }
enum OrderEvent { Submit, Pay, Ship, Deliver, Cancel }

var transitions = new Dictionary<(OrderState, OrderEvent), OrderState>
{
    [(OrderState.Draft, OrderEvent.Submit)] = OrderState.Submitted,
    [(OrderState.Draft, OrderEvent.Cancel)] = OrderState.Cancelled,
    [(OrderState.Submitted, OrderEvent.Pay)] = OrderState.Paid,
    [(OrderState.Submitted, OrderEvent.Cancel)] = OrderState.Cancelled,
    [(OrderState.Paid, OrderEvent.Ship)] = OrderState.Shipped,
    [(OrderState.Shipped, OrderEvent.Deliver)] = OrderState.Delivered
};

OrderState? Transition(OrderState current, OrderEvent ev) =>
    transitions.TryGetValue((current, ev), out var next) ? next : null;

// Использование
var state = OrderState.Draft;
state = Transition(state, OrderEvent.Submit) ?? state;   // → Submitted
state = Transition(state, OrderEvent.Pay) ?? state;      // → Paid
state = Transition(state, OrderEvent.Cancel) ?? state;   // No transition — остаёмся в Paid
Console.WriteLine(state);   // Paid
```

Tuple `(State, Event)` как ключ — самый чистый способ описать таблицу переходов.

### 16.5. Async + Task.WhenAll + tuple deconstruction

```csharp
public class ProfileService
{
    public async Task<(User User, List<Order> Orders, decimal Balance)> LoadProfileAsync(int userId)
    {
        var userTask = _userRepo.GetAsync(userId);
        var ordersTask = _orderRepo.GetForUserAsync(userId);
        var balanceTask = _accountRepo.GetBalanceAsync(userId);

        await Task.WhenAll(userTask, ordersTask, balanceTask);

        return (await userTask, await ordersTask, await balanceTask);
    }
}

// Использование
var (user, orders, balance) = await profileService.LoadProfileAsync(42);
Console.WriteLine($"{user.Name}: {orders.Count} orders, balance ${balance}");
```

Все три запроса идут параллельно, deconstruction делает результат tidy. Альтернатива — три строки `var user = await userTask;` и т.д., что многословнее.

### 16.6. TryGet-pattern с tuple

```csharp
public class UserCache
{
    private readonly Dictionary<int, User> _store = new();

    public (bool Found, User? User) TryGet(int id) =>
        _store.TryGetValue(id, out var user)
            ? (true, user)
            : (false, null);
}

// Использование
var (found, user) = cache.TryGet(42);
if (found)
    Console.WriteLine(user!.Name);
```

Альтернатива — Maybe/Option:

```csharp
public Option<User> TryGet(int id) => /* ... */;
```

Tuple проще, Maybe строже. Выбирай по проекту.

### 16.7. Refactoring tuple → record (когда выросло)

Часто паттерн: начали с tuple для быстроты, через месяц поняли — это полноценный домен:

```csharp
// Раунд 1 — быстро запилили
public (string Name, decimal Price, int Stock) GetProductInfo(int id) { ... }

// Через 3 месяца — оказалось, нужно ещё Discount, Category, Tax
public (string Name, decimal Price, int Stock, decimal Discount, string Category, decimal Tax) GetProductInfo(int id)
{
    // ... 15 callers пришлось обновить, всех руками деконструировать заново
}

// Раунд 2 — поняли, нужен record
public record ProductInfo(
    string Name,
    decimal Price,
    int Stock,
    decimal Discount,
    string Category,
    decimal Tax
);

public ProductInfo GetProductInfo(int id) { ... }

// Дальше можно использовать with-выражения, добавлять поля с init/required
public record ProductInfo(
    string Name,
    decimal Price,
    int Stock,
    decimal Discount,
    string Category,
    decimal Tax,
    DateTime LastUpdated = default   // новое поле, default-значение → не breaking
);
```

**Правило миграции:** tuple → record когда:

- Поле начало менять смысл (не первоначальные 2-3, а уже 5+).
- Тип используется в нескольких местах.
- Возврат уезжает в JSON / lib API / БД.
- Хочется методов на нём (`product.IsAvailable()`).

Это естественная эволюция — не страшись. Лучше начать с tuple и переехать на record, чем годами тащить class из 5 строк.

---

## 17. Common Pitfalls — с механизмами

### 17.1. Item1, Item2 в production коде

```csharp
// ❌ Что значит Item2?
public (string, decimal) GetPrice(int productId) { ... }

var p = GetPrice(1);
Console.WriteLine($"Price: {p.Item2}");   // Item2? Цена? Налог?
```

**Механизм:** компилятор не требует имена. Если их нет — у tuple `Item1`, `Item2`. Это всегда читать тяжелее, чем именованные поля.

**Фикс:** именуй всё, что выходит за рамки одной строки:

```csharp
public (string Name, decimal Price) GetPrice(int productId) { ... }

var p = GetPrice(1);
Console.WriteLine($"Price: {p.Price}");
```

### 17.2. Tuple в public API — sealed contract

```csharp
// Public API — tuple
public class ReportService
{
    public (int Total, int Errors) GenerateReport() { ... }
}
```

Через 3 месяца понадобилось добавить `int Warnings`:

```csharp
public (int Total, int Errors, int Warnings) GenerateReport() { ... }
```

Это **breaking change**. Все клиенты, которые писали `var (t, e) = service.GenerateReport();`, теперь не компилируются — арность не совпадает. Всех нужно обновлять.

**Механизм:** tuple — sealed по структуре. Изменение арности = breaking. С record можно добавлять поля с `init` без слома (если callers используют свойства, а не позиционные).

**Фикс:** record для public API. Tuple — для internal/private.

### 17.3. Большой tuple — copy expensive

```csharp
public (int A, int B, int C, int D, int E, int F, int G, int H) BigStats() { ... }
```

При передаче в другие методы — копируется 32-64 байта (зависит от типов). Невидимо, но в hot path кумулятивно заметно.

**Механизм:** struct копируется по value. Большой struct → большая копия.

**Фикс:**
- Если 4+ полей — record (heap-аллокация один раз, дальше передача ссылки).
- Если очень нужна структура без heap — `in`-параметр (передача по readonly ссылке).

### 17.4. Mutating tuple field — это копия!

```csharp
var t = (X: 1, Y: 2);
ModifyTuple(t);
Console.WriteLine(t.X);   // Всё ещё 1! Не 100.

void ModifyTuple((int X, int Y) tuple)
{
    tuple.X = 100;   // мутируем КОПИЮ внутри метода
}
```

**Механизм:** tuple — struct. При передаче в метод копируется. Изменения внутри метода не видны вызывающему.

**Фикс:**
- Для мутации — `ref`:

```csharp
void ModifyTuple(ref (int X, int Y) tuple)
{
    tuple.X = 100;
}

var t = (X: 1, Y: 2);
ModifyTuple(ref t);
Console.WriteLine(t.X);   // 100
```

- Чаще — вернуть новый tuple:

```csharp
(int X, int Y) Modify((int X, int Y) input) => (input.X * 2, input.Y);

var t = (X: 1, Y: 2);
t = Modify(t);
```

### 17.5. `List<tuple>` indexer — нельзя мутировать поле

```csharp
var list = new List<(int X, int Y)> { (1, 2), (3, 4) };
list[0].X = 100;
// ❌ CS1612: Cannot modify the return value of 'List<(int X, int Y)>.this[int]'
```

**Механизм:** `List<T>.this[int]` возвращает копию (для struct). Менять копию бессмысленно — компилятор запрещает.

**Фикс:** заменить весь tuple:

```csharp
list[0] = (100, list[0].Y);
```

Или — `List<MyClass>` (reference type, нет копий).

В **массиве** работает по-другому:

```csharp
var arr = new (int X, int Y)[] { (1, 2), (3, 4) };
arr[0].X = 100;   // ✅ работает — array indexer возвращает managed reference
```

Это разница между `List<T>` (через property) и `T[]` (через index).

### 17.6. Tuple equality по позиции, не по именам

```csharp
(int Year, int Month) date = (2024, 5);
(int Hour, int Minute) time = (2024, 5);

date == time;   // True!
```

**Механизм:** equality сравнивает поля `Item1`, `Item2` по позиции. Имена — compile-time, в runtime их нет.

**Фикс:** для разных доменных понятий — разные record-ы:

```csharp
public record YearMonth(int Year, int Month);
public record HourMinute(int Hour, int Minute);

YearMonth a = new(2024, 5);
HourMinute b = new(2024, 5);

// a == b — compile error: разные типы
```

### 17.7. EF Core translation broken

```csharp
var query = db.Users
    .Where(u => u.IsActive)
    .Select(u => (u.Id, u.Name))
    .ToList();
// Иногда: InvalidOperationException could not be translated
```

**Механизм:** EF Core LINQ-провайдер исторически работал с anonymous types. Поддержка tuples добавлялась, но не для всех сценариев.

**Фикс:** anonymous type:

```csharp
var query = db.Users
    .Where(u => u.IsActive)
    .Select(u => new { u.Id, u.Name })
    .ToList();
```

Если результат нужен дальше — конвертируй в tuple на client side (`AsEnumerable()` потом `Select`).

### 17.8. JSON сериализация без имён

```csharp
public (string Name, int Age) GetPerson() => ("Alice", 30);

string json = JsonSerializer.Serialize(GetPerson());
// {"Item1":"Alice","Item2":30}
```

**Механизм:** имена — compile-time, сериализатор работает с runtime-полями.

**Фикс:** record для всего, что сериализуется:

```csharp
public record Person(string Name, int Age);
public Person GetPerson() => new("Alice", 30);
```

### 17.9. Имена теряются на boundaries

```csharp
public (int X, int Y) GetPoint() => (1, 2);

var p = GetPoint();
Console.WriteLine(p.X);   // ✅

object o = p;
var t = (ValueTuple<int, int>)o;
// Console.WriteLine(t.X);   // ❌ нет имени, поля Item1, Item2
```

**Механизм:** имена живут в `TupleElementNamesAttribute` на сигнатуре. Через `object` теряются.

**Фикс:** не приводить tuple к `object`. Если нужно — заводи record.

### 17.10. `Tuple<T1, T2>` старый — ловушка для джуна

```csharp
// Кажется одинаковым, но это разные типы:
var a = Tuple.Create(1, 2);          // Tuple<int, int> — class!
var b = (1, 2);                      // ValueTuple<int, int> — struct!

Console.WriteLine(a == b);   // ❌ Compile error — разные типы
Console.WriteLine(a.Item1);  // ✅ — у Tuple тоже есть Item1
Console.WriteLine(b.Item1);  // ✅ — у ValueTuple тоже
```

**Механизм:** два независимых типа в BCL. Старый `Tuple<T>` — class на куче. Новый `ValueTuple<T>` — struct на стеке.

**Фикс:** в новом коде `ValueTuple` через литерал `(a, b)`. `Tuple.Create` не использовать. Если в legacy-методе — рефакторить.

> [!question]- Интервью: 5 главных грабель tuple
> 1. Безымянные `Item1`, `Item2` — нечитаемо. 2. Tuple в public API — sealed contract, ломается при добавлении поля. 3. JSON-сериализация теряет имена. 4. EF Core не всегда транслирует tuple-проекции. 5. List<tuple> через indexer — нельзя мутировать поле, только заменить tuple целиком.

---

## 18. Advanced — для любопытных

### 18.1. ITuple — runtime introspection

```csharp
using System.Runtime.CompilerServices;

void DumpTuple(ITuple t)
{
    Console.WriteLine($"Length: {t.Length}");
    for (int i = 0; i < t.Length; i++)
        Console.WriteLine($"  [{i}] = {t[i]} ({t[i]?.GetType().Name})");
}

DumpTuple((1, "hello", 3.14));
// Length: 3
//   [0] = 1 (Int32)
//   [1] = hello (String)
//   [2] = 3.14 (Double)
```

`ITuple` реализуют все `ValueTuple<...>` и `Tuple<...>`. Полезно для library-кода (логгеры, дамперы).

### 18.2. Reflection: получить имена tuple-полей

Имена живут на сигнатуре:

```csharp
class Service
{
    public (int Count, decimal Total, string Currency) GetStats() => (5, 100m, "USD");
}

var method = typeof(Service).GetMethod(nameof(Service.GetStats));
var attr = method!.ReturnParameter
    .GetCustomAttribute<TupleElementNamesAttribute>();

if (attr is not null)
{
    var names = attr.TransformNames;
    for (int i = 0; i < names.Count; i++)
        Console.WriteLine($"Field {i}: {names[i]}");
}
// Field 0: Count
// Field 1: Total
// Field 2: Currency
```

Аналогично для полей класса, параметров методов — атрибут на соответствующем `MemberInfo`.

### 18.3. IL-дамп — что генерит компилятор

```csharp
public (int X, int Y) GetPoint() => (1, 2);
```

Компилируется примерно в:

```il
.method public hidebysig instance valuetype [System.Runtime]System.ValueTuple`2<int32, int32>
  GetPoint() cil managed
{
    .param [0]
    .custom instance void [System.Runtime]System.Runtime.CompilerServices.TupleElementNamesAttribute::.ctor(string[])
        = ( string[2] { "X", "Y" } )

    ldc.i4.1
    ldc.i4.2
    newobj instance void valuetype [System.Runtime]System.ValueTuple`2<int32, int32>::.ctor(!0, !1)
    ret
}
```

Видно: `ValueTuple` создаётся через `newobj` (это для struct тоже работает — создаёт значение на стеке). Атрибут `TupleElementNamesAttribute` навешан на return parameter с именами `X`, `Y`.

Можно посмотреть самостоятельно через [SharpLab](https://sharplab.io) — вставь свой C# код и увидишь IL/ASM/decompiled C#.

### 18.4. Tuple syntactic sugar — где раскрывается

Компилятор разворачивает синтаксис tuple в нескольких местах:

```csharp
// 1. Литерал
var t = (1, 2);
// → new ValueTuple<int, int>(1, 2)

// 2. Доступ по имени
t.X
// → t.Item1 (если X — alias первого поля)

// 3. Деконструкция
var (a, b) = t;
// → int a = t.Item1; int b = t.Item2;

// 4. Equality
t == other
// → t.Item1 == other.Item1 && t.Item2 == other.Item2

// 5. Pattern matching
case (var x, var y):
// → проверка типа + извлечение Item1, Item2
```

Под всем этим — обычный `ValueTuple<T1, T2, ...>`. Никакой магии.

### 18.5. Async + ValueTask + tuple

```csharp
public async ValueTask<(int Status, string Body)> FetchAsync(string url)
{
    if (TryGetCached(url, out var cached))
        return (200, cached);   // sync path — без аллокации Task

    var response = await _http.GetAsync(url);
    var body = await response.Content.ReadAsStringAsync();
    return ((int)response.StatusCode, body);
}
```

`ValueTask<(T1, T2)>` — двойная экономия: ValueTask избегает аллокации Task на sync-пути, ValueTuple избегает аллокации возвращаемой структуры. Хорошо для hot-path методов с кэшем.

### 18.6. ref returns с tuple

`ref T`-возврат позволяет возвращать ссылку на поле:

```csharp
public class Storage
{
    private (int X, int Y)[] _data = new (int, int)[100];

    public ref (int X, int Y) GetSlot(int index) => ref _data[index];
}

var storage = new Storage();
ref var slot = ref storage.GetSlot(0);
slot.X = 100;   // Изменяет напрямую _data[0].X — без копий
```

Это уже HFT/perf-уровень. Обычно избыточно.

### 18.7. Tuple в Expression Trees

Tuple можно использовать в expression trees, но с ограничениями:

```csharp
Expression<Func<(int, int)>> expr = () => (1, 2);
var compiled = expr.Compile();
var result = compiled();
Console.WriteLine($"{result.Item1}, {result.Item2}");   // 1, 2
```

В EF Core это иногда не транслируется в SQL — провайдер не всегда понимает `NewExpression` для `ValueTuple`. Поэтому в EF-запросах безопаснее anonymous types.

---

## 19. Best Practices

- **Records — для public API.** Tuples — для internal / private.
- **Имя у каждого поля tuple** — кроме совсем тривиальных одноразовых (`var (a, b) = (1, 2)` в учебном коде OK; в боевом — `(X: 1, Y: 2)`).
- **Не больше 4 элементов** в tuple. От 4-5 — заводи record.
- **`var (x, y) = method()`** — самый чистый способ читать multi-return.
- **`_` discard** для неиспользуемых значений — не тащи фейковую переменную.
- **Composite keys** в `Dictionary` / `HashSet` — это идеальное применение tuple.
- **`in`-параметр** для tuple от 4-5 элементов — избегаешь копии.
- **Anonymous type** — внутри LINQ-проекции, особенно в EF Core.
- **Record** — если результат уезжает за рамки одного метода (JSON, отчёты, лог-агрегаторы, межсервисный обмен).
- **Tuple никогда** в публичном API REST/GraphQL/gRPC — нужна стабильная схема, имена, эволюция.
- **Tuple equality по именам** — не работает. Если домены разные — заводи разные record-ы.
- **`Tuple.Create` / `Tuple<T>`** — legacy, не использовать в новом коде.
- **JSON / XML serialization** — никогда tuple, всегда record.
- **EF Core projection** — anonymous type, не tuple.

---

## 20. Decision tree

```
Что делаем?
│
├── Возвращаем несколько значений из метода
│   ├── Internal / private метод
│   │   ├── 2-4 значения → tuple с именами
│   │   └── 5+ значений → record
│   │
│   ├── Public API библиотеки / сервиса
│   │   └── record (всегда)
│   │
│   └── Try*-метод BCL-style (часто, perf-критично)
│       └── out (если работаешь со Span — обязательно out)
│
├── Composite key для коллекции
│   └── tuple (Dictionary<(T1, T2), V>, HashSet<(T1, T2)>)
│
├── LINQ projection
│   ├── Внутри EF Core запроса → anonymous type
│   └── Чисто in-memory → tuple или anonymous
│
├── Аккумулятор в Aggregate
│   └── tuple (без heap-аллокаций, читается естественно)
│
├── Состояние/переход в state machine
│   └── tuple (State, Event) → State в Dictionary
│
└── Группа значений живёт долго / используется во многих местах
    └── record / class

Декомпозиция значения?
│
├── Tuple → встроена, var (x, y) = ...
├── Record → встроена (positional records)
├── Свой класс → реализуй Deconstruct(out ..., out ...)
├── Чужой тип → extension Deconstruct
└── Dictionary entries → foreach (var (k, v) in dict)

Какие имена?
│
├── Tuple → имена в сигнатуре сохраняются, в локалках теряются
├── Record → имена — настоящие свойства, везде живут
└── Anonymous → имена есть, но тип нельзя назвать
```

---

## 21. Cheat sheet

```csharp
// Создание
var a = (1, 2);                      // (int, int)
var b = (X: 1, Y: 2);                // (int X, int Y)
(int X, int Y) c = (1, 2);           // явный тип

// Multiple return
public (int Min, int Max) MinMax(int[] arr) =>
    (arr.Min(), arr.Max());

var (min, max) = MinMax([3, 1, 4, 1, 5]);

// Swap
(a, b) = (b, a);

// Composite key
var dict = new Dictionary<(string, int), User>();
dict[("alice", 30)] = user;

// Discard
var (_, total) = GetStats();

// Foreach Dictionary
foreach (var (key, value) in dict)
    Console.WriteLine($"{key}: {value}");

// Pattern matching
var description = point switch
{
    (0, 0) => "origin",
    (var x, 0) => $"on X at {x}",
    (0, var y) => $"on Y at {y}",
    _ => "elsewhere"
};

// Equality
(1, 2) == (1, 2);   // True
(X: 1, Y: 2) == (A: 1, B: 2);   // True (имена ignored)

// Tuple alias (C# 12)
using Coord = (int X, int Y);
Coord MakeCoord() => (1, 2);

// Деконструкция своего класса
public class Person
{
    public string Name { get; init; } = "";
    public int Age { get; init; }
    public void Deconstruct(out string name, out int age)
    {
        name = Name;
        age = Age;
    }
}

// Async + WhenAll + tuple
async Task<(User, List<Order>)> LoadAsync(int id)
{
    var u = _users.GetAsync(id);
    var o = _orders.GetAsync(id);
    await Task.WhenAll(u, o);
    return (await u, await o);
}

// Refactoring: out → tuple
// Было:
public bool TryGet(int id, out User user) { ... }
// Стало:
public (bool Found, User? User) TryGet(int id) { ... }
```

| Сценарий | Решение |
|----------|---------|
| Multiple return | `(T1 Name1, T2 Name2) Method()` |
| Swap | `(a, b) = (b, a)` |
| Composite key | `Dictionary<(K1, K2), V>` |
| Foreach Dictionary | `foreach (var (k, v) in dict)` |
| Pattern matching | `case (var x, 0):` |
| Discard | `var (_, value) = ...` |
| LINQ projection в EF | anonymous type |
| LINQ projection in-memory | tuple OK |
| Async + параллельно | `Task.WhenAll` + tuple deconstruct |
| Public API | record, не tuple |
| Сериализация JSON | record, не tuple |
| > 4 полей | record, не tuple |

---

## 22. Practice — упражнения с разбором

### 22.1. Min / Max за один проход

**Задача.** Написать метод, который за один проход возвращает min, max и среднее массива.

```csharp
public static (int Min, int Max, double Average) Stats(int[] numbers)
{
    if (numbers.Length == 0)
        throw new ArgumentException("Array empty");

    return numbers.Aggregate(
        (Min: int.MaxValue, Max: int.MinValue, Sum: 0L),
        (acc, n) => (
            Math.Min(acc.Min, n),
            Math.Max(acc.Max, n),
            acc.Sum + n
        ),
        acc => (acc.Min, acc.Max, acc.Sum / (double)numbers.Length)
    );
}

// Использование
var (min, max, avg) = Stats([3, 7, 1, 9, 4]);
Console.WriteLine($"min={min}, max={max}, avg={avg:F2}");
// min=1, max=9, avg=4.80
```

**Разбор:** трёхкомпонентный tuple-аккумулятор позволяет посчитать всё за один проход. `Aggregate` с `resultSelector` (третий параметр) превращает аккумулятор `(Min, Max, Sum)` в финальный `(Min, Max, Average)`.

### 22.2. Frequency counter с tuple key

**Задача.** Подсчитать, сколько раз каждая (буква, регистр) встречается в строке.

```csharp
public static Dictionary<(char Letter, bool IsUpper), int> CountByCase(string text)
{
    var result = new Dictionary<(char, bool), int>();
    foreach (var c in text)
    {
        if (!char.IsLetter(c)) continue;
        var key = (char.ToLowerInvariant(c), char.IsUpper(c));
        result[key] = result.GetValueOrDefault(key) + 1;
    }
    return result;
}

// Использование
var counts = CountByCase("HelloWorld");
foreach (var (key, count) in counts)
    Console.WriteLine($"{key.Letter} ({(key.IsUpper ? "upper" : "lower")}): {count}");
// h (upper): 1
// e (lower): 1
// l (lower): 3
// o (lower): 2
// w (upper): 1
// r (lower): 1
// d (lower): 1
```

**Разбор:** tuple `(char, bool)` — composite key без проблем. `GetValueOrDefault(key) + 1` — идиоматичное «инкремент или начало с 1».

### 22.3. Sort by composite key

**Задача.** Отсортировать список людей по фамилии, потом по имени.

```csharp
record Person(string FirstName, string LastName);

var people = new List<Person>
{
    new("Bob", "Smith"),
    new("Alice", "Jones"),
    new("Charlie", "Smith"),
    new("Alice", "Brown")
};

var sorted = people
    .OrderBy(p => (p.LastName, p.FirstName))
    .ToList();

foreach (var p in sorted)
    Console.WriteLine($"{p.FirstName} {p.LastName}");
// Alice Brown
// Alice Jones
// Bob Smith
// Charlie Smith
```

**Разбор:** tuple реализует `IComparable`, поэтому `OrderBy(p => (p.LastName, p.FirstName))` сортирует сначала по LastName, при равенстве — по FirstName. Без tuple пришлось бы `OrderBy(...).ThenBy(...)`.

### 22.4. Refactoring out → tuple

**Задача.** Преобразовать старый стиль в новый.

```csharp
// Было
public class CoordParser
{
    public bool TryParse(string input, out int x, out int y, out string error)
    {
        x = 0; y = 0; error = "";
        var parts = input.Split(',');
        if (parts.Length != 2) { error = "Need 2 parts"; return false; }
        if (!int.TryParse(parts[0], out x)) { error = "X not number"; return false; }
        if (!int.TryParse(parts[1], out y)) { error = "Y not number"; return false; }
        return true;
    }
}

// Использование
var parser = new CoordParser();
if (parser.TryParse("10,20", out int x, out int y, out string err))
    Console.WriteLine($"{x}, {y}");
else
    Console.WriteLine($"Error: {err}");
```

**Стало:**

```csharp
public class CoordParser
{
    public (bool Success, int X, int Y, string? Error) TryParse(string input)
    {
        var parts = input.Split(',');
        if (parts.Length != 2)
            return (false, 0, 0, "Need 2 parts");
        if (!int.TryParse(parts[0], out var x))
            return (false, 0, 0, "X not number");
        if (!int.TryParse(parts[1], out var y))
            return (false, 0, 0, "Y not number");
        return (true, x, y, null);
    }
}

// Использование
var parser = new CoordParser();
var (ok, x, y, err) = parser.TryParse("10,20");
if (ok)
    Console.WriteLine($"{x}, {y}");
else
    Console.WriteLine($"Error: {err}");
```

**Разбор:** четыре out-параметра → один tuple-возврат. Стало возможным использовать в LINQ и async.

**Идём дальше:** для production кода — это сигнал «нужен `Result<T, E>`»:

```csharp
public Result<(int X, int Y), string> TryParse(string input) { ... }
```

### 22.5. Deconstruct своего класса с overloading

**Задача.** Сделать класс `Money`, который деконструируется и в `(amount, currency)`, и в `(amount, currency, formatted)`.

```csharp
public class Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency)
    {
        Amount = amount;
        Currency = currency;
    }

    public string Format() => $"{Amount:F2} {Currency}";

    public void Deconstruct(out decimal amount, out string currency)
    {
        amount = Amount;
        currency = Currency;
    }

    public void Deconstruct(out decimal amount, out string currency, out string formatted)
    {
        amount = Amount;
        currency = Currency;
        formatted = Format();
    }
}

// Использование
var m = new Money(99.95m, "USD");
var (amount, currency) = m;                          // 2-arity
var (amount2, currency2, formatted) = m;             // 3-arity
Console.WriteLine($"{amount} {currency}");           // 99.95 USD
Console.WriteLine(formatted);                        // 99.95 USD
```

**Разбор:** компилятор выбирает overload по количеству целевых переменных. Это даёт «несколько срезов» одного объекта — удобно для разных мест использования.

---

## 23. Что читать дальше — порядок и почему

1. **[[modern-features|Modern Features]]** — там полный разбор pattern matching (positional, list, recursive). Tuple — это часть большой картины.
2. **[[oop|OOP и классы]]** — `Deconstruct` для классов в общем контексте, records, Value Objects.
3. **[[anonymous-types|Anonymous Types]]** — близкий родственник tuples, особенно в LINQ-проекциях.
4. **[[error-handling|Error Handling]]** — `Result<T, E>` и почему tuple `(bool, T)` не идеален для error handling.
5. **[[collections-linq|Collections и LINQ]]** — `GroupBy` с composite keys, `Aggregate` с tuple-аккумулятором.
6. **[[functional-csharp|Functional C#]]** — pattern matching и tuples в functional-стиле.

---

## 24. См. также

- [[csharp-basics|C# Basics]] — базовые типы и переменные
- [[modern-features|Modern Features]] — pattern matching deep
- [[oop|OOP]] — Deconstruct в классах, records
- [[anonymous-types|Anonymous Types]] — для LINQ projections
- [[error-handling|Error Handling]] — `Result<T, E>` как альтернатива tuple для success/failure
- [[collections-linq|Collections и LINQ]] — composite keys, Aggregate
- [[functional-csharp|Functional C#]] — records + pattern matching
- [[generics-deep|Generics deep]] — generic constraints на tuple-параметры

---

## 25. Reading list

- **Microsoft Docs — Tuple types** — learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/value-tuples
- **Microsoft Docs — Deconstructing tuples and other types** — learn.microsoft.com/dotnet/csharp/fundamentals/functional/deconstruct
- **Microsoft Docs — Pattern matching** — learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching
- **Mads Torgersen — New features in C# 7.0** — devblogs.microsoft.com/dotnet/new-features-in-c-7-0
- **Stephen Toub — ValueTuple performance** — devblogs.microsoft.com/dotnet (поиск «ValueTuple»)
- **Jared Parsons — Tuples Design Notes** — github.com/dotnet/csharplang/blob/main/proposals/tuples.md
- **Ian Griffiths — Programming C# 12** — chapter «Tuples and ValueTuple»
- **Jon Skeet — C# in Depth (4th ed.)** — chapter про tuple equality и patterns
- **SharpLab** — sharplab.io — посмотреть, во что компилируется tuple-синтаксис
- **EF Core docs — Client vs Server evaluation** — learn.microsoft.com/ef/core/querying/client-eval
