---
tags: [csharp, extension-methods, junior, syntax-sugar, fluent-api, linq]
level: Junior
date: 2026-08-02
---

# Extension Methods — методы-расширения

> **`obj.MyMethod()` для типов, которые ты не контролируешь.** Под капотом — обычные статические методы. Сахарный синтаксис, на котором построен LINQ, fluent API ASP.NET Core, builder pattern, и почти каждая современная C#-библиотека. Закрывает пробел: «вижу `services.AddControllers()`, не понимаю, откуда метод вообще берётся».

---

## 0. Как читать этот файл

Если ты впервые видишь `static class` с `this`-параметром — читай разделы 1→4 подряд: получишь рабочую модель. Если уже пишешь extensions, но непонятно «как они компилируются и почему именно так» — раздел 3 (механика), 8 (resolution rules), 11 (null this). Если интересна архитектура (fluent API, DI registrations, builder pattern) — раздел 13-15.

Все примеры самостоятельные и компилируются. `// expected: ...` показывает ожидаемый вывод. Cross-language якоря (`> [!info]-`) свёрнуты — раскрывай, если переходишь из Kotlin / Swift / Rust / Python / Ruby. Interview-вопросы (`> [!question]-`) встроены рядом с теорией.

---

## 1. Что это, зачем и когда

### 1.1. Что такое extension method

**Extension method** — это статический метод, который вызывается так, как будто это обычный метод инстанса:

```csharp
public static class StringExtensions
{
    public static bool IsNullOrEmpty(this string s) => string.IsNullOrEmpty(s);
    //                                  ^^^^ ключевое: this перед первым параметром
}

// Использование
"".IsNullOrEmpty();          // True — выглядит как метод string
"hello".IsNullOrEmpty();     // False
```

Под капотом компилятор переписывает `"".IsNullOrEmpty()` в `StringExtensions.IsNullOrEmpty("")` — обычный статический вызов. Никакой магии runtime, всё разруливается на этапе компиляции.

Это даёт два эффекта:

1. **Синтаксический сахар** — читается как метод объекта, без громоздкого `Helper.DoX(obj, ...)`.
2. **Можно расширять чужие типы** — добавить метод к `string`, `int`, `List<T>`, типу из NuGet-пакета без модификации исходников.

### 1.2. Зачем extensions появились

Extension methods появились в **C# 3.0 (2007)** — той же версии, что принесла LINQ. Это не совпадение: они были **технологическим фундаментом LINQ**.

Вся фишка LINQ — `collection.Where(...).Select(...).ToList()` — построена на extensions. Без них пришлось бы писать:

```csharp
// Без extensions
var result = Enumerable.ToList(
    Enumerable.Select(
        Enumerable.Where(collection, predicate),
        selector));

// С extensions
var result = collection.Where(predicate).Select(selector).ToList();
```

Второй вариант читается слева направо, как естественный язык. Первый — обратно, через вложенные скобки. Extensions сделали fluent API возможным.

### 1.3. Главные применения

```csharp
// 1. Расширение чужих типов (BCL, NuGet)
"hello".IsNullOrEmpty();
DateTime.Now.IsWeekend();

// 2. LINQ
list.Where(x => x > 0).Select(x => x * 2).ToList();

// 3. Fluent API (builder pattern, DI)
services.AddControllers()
        .AddJsonOptions(opts => ...);

// 4. ASP.NET Core middleware
app.UseRouting()
   .UseAuthorization()
   .UseEndpoints(...);

// 5. Domain-friendly API
order.Total.AsCurrency("USD");
3.Hours().Plus(15.Minutes());

// 6. Adapters
httpClient.AsTypedClient<IUserApi>();
```

Везде, где видишь chain методов на объекте — почти наверняка extensions.

### 1.4. Эволюция: C# 3.0 → C# 14

| Версия | Год | Что появилось |
|--------|-----|---------------|
| **C# 3.0** | 2007 | Extensions methods, LINQ построен на них |
| **C# 6.0** | 2015 | `using static` — упрощает import класса с extensions |
| **C# 7.2** | 2017 | `in` параметр (read-only ref для structs) — `this in` extension |
| **C# 7.3** | 2018 | `ref` параметр для `this` (mutating extensions для structs) |
| **C# 9.0** | 2020 | `static abstract` в interfaces (но не для extensions) |
| **C# 12** | 2023 | Primary constructors для классов — упрощает паттерн helper-классов |
| **C# 13** | 2024 | `params` для коллекций — `params ReadOnlySpan<int>` параметр |
| **C# 14** | 2025 | **Extension members** (релиз с .NET 10) — extension-блоки: properties, static-члены, операторы |

С **C# 14** концепция расширилась: можно делать не только методы, но и properties, static-члены и операторы. Раздел 16 — об этом.

### 1.5. Когда применять, когда нет

✅ **Используй когда:**
- Расширяешь чужой тип, исходники которого недоступны (BCL, NuGet).
- Строишь fluent API (builder, configuration).
- Хочешь дать тип-специфичный метод без наследования.
- LINQ-style операции над коллекциями / sequences.
- Adapter / converter методы (`.AsX()`, `.ToY()`).

❌ **Не используй когда:**
- Можешь добавить метод в **свой** класс — добавь, не разделяй логику.
- Метод выглядит как обычная утилита (`Math.Sqrt`-типа) — это просто `static`-метод, без `this`.
- Создаёт illusion of method, который **должен быть** на типе, но добавить нельзя — это часто запах архитектуры (попробуй composition / wrapper class).
- Хочешь "переопределить" поведение чужого метода — extensions имеют **низкий приоритет**, instance method чужого класса всегда выиграет (раздел 8).

### 1.6. Mental model — это просто статический метод

Самое важное для понимания: **extension method — это синтаксический сахар над static методом**. После компиляции разницы нет.

```csharp
public static class IntExt
{
    public static int Squared(this int n) => n * n;
}

// Эти два вызова идентичны
int a = 5.Squared();
int b = IntExt.Squared(5);
// IL для обоих — один и тот же call
```

Это значит:
- Никакого override чужих методов.
- Никакой полиморфизм.
- Никакого dispatch на runtime — всё резолвится на этапе компиляции.
- Никаких дополнительных allocations.
- Никакого overhead vs обычный static call.

Если ты понял эту mental model — все остальные правила следуют логично.

> [!info]- Если ты знаешь Kotlin / Swift / Rust / Python / Ruby
> **Kotlin:** `fun String.isEmpty() = ...` — extensions работают так же, как в C#, тоже compile-time резолюция через статический метод. Очень близкая семантика. Kotlin был сильно вдохновлён C# extensions.
>
> **Swift:** `extension String { func isEmpty() -> Bool { ... } }` — extensions могут добавлять методы, properties, computed-свойства, conformance к протоколам. Мощнее C# (можно добавить protocol conformance). C# 14 догоняет с extension members.
>
> **Rust:** `impl Trait for Type { ... }` — близко по идее (добавить методы внешнему типу), но через trait + impl. Без trait в Rust нельзя — это часть type system, а не сахар.
>
> **Python:** «monkey patching» — `String.foo = lambda self: ...`. Изменяет класс в runtime. Хрупко, может сломать чужой код. C# extensions безопаснее — compile-time, не модифицируют тип.
>
> **Ruby:** «refinements» (`refine String do ... end`) — лексически-ограниченные monkey patches. Концепция близка к extensions, но через scope.

> [!question]- Интервью: чем extension method отличается от обычного статического метода?
> Технически — ничем, после компиляции это **тот же** статический вызов. Разница только синтаксическая: extension с ключевым словом `this` перед первым параметром можно вызвать через точку (`obj.Method()`), обычный статический метод требует `Class.Method(obj)`. IL и performance идентичны. Поэтому extension — это «синтаксический сахар над static», а не отдельная сущность языка.

---

## 2. Синтаксис и базовые правила

### 2.1. Минимальный пример

```csharp
public static class StringExtensions   // 1. static class
{
    public static int WordCount(this string s)   // 2. static method, this первый параметр
    {
        if (string.IsNullOrWhiteSpace(s)) return 0;
        return s.Split([' ', '\t', '\n'], StringSplitOptions.RemoveEmptyEntries).Length;
    }
}

// Использование (там где StringExtensions виден через using)
"hello world".WordCount();   // 2
"".WordCount();              // 0
((string?)null).WordCount(); // 0 — extensions работают на null!
```

### 2.2. Обязательные требования

Чтобы C# распознал метод как extension, **все** условия должны выполняться:

1. **Класс — `static`.** `public static class StringExtensions` — без static не скомпилируется.
2. **Метод — `static`.** Внутри static класса все методы static, но указать обязательно явно.
3. **Первый параметр — `this T`** где `T` — расширяемый тип.
4. **Класс не generic.** `static class StringExt<T>` нельзя; methods могут быть generic, класс — нет.
5. **Не nested class.** Extension methods могут быть только в top-level static-классе (ну или в другом static-классе, но не в обычном классе).

```csharp
// ❌ Не extension — класс не static
public class StringExt
{
    public static int WordCount(this string s) => ...;   // CS1106
}

// ❌ Не extension — метод не static
public static class StringExt
{
    public int WordCount(this string s) => ...;   // CS1105
}

// ❌ Не extension — generic class
public static class StringExt<T>
{
    public static int WordCount(this string s) => ...;   // CS1106
}

// ✅ OK — generic метод в non-generic классе
public static class CollectionExt
{
    public static T First<T>(this IEnumerable<T> source) => ...;
}
```

### 2.3. Множественные параметры

После `this`-параметра идут обычные:

```csharp
public static class StringExt
{
    public static string Repeat(this string s, int times)
    {
        if (times <= 0) return "";
        var sb = new StringBuilder(s.Length * times);
        for (int i = 0; i < times; i++) sb.Append(s);
        return sb.ToString();
    }
}

"abc".Repeat(3);   // "abcabcabc"
```

Значение, на котором вызвали (`"abc"`), идёт в `this`-параметр (`s`). Остальные — после.

### 2.4. Доступ к `using`

Extensions «видны», только если их namespace импортирован:

```csharp
// MyApp.Helpers/StringExt.cs
namespace MyApp.Helpers;

public static class StringExt
{
    public static int WordCount(this string s) => ...;
}
```

```csharp
// MyApp.Program.cs
using MyApp.Helpers;   // ← без этого WordCount не виден

class Program
{
    static void Main()
    {
        "hello world".WordCount();   // OK с using
    }
}
```

Это работает потому, что compiler ищет extension methods в **импортированных namespace'ах**. Если import (`using`) не сделан — компилятор не найдёт метод и выдаст CS1061: `string does not contain a definition for 'WordCount'`.

### 2.5. Global usings (.NET 6+)

Если extension используется по всему проекту, добавь global using в `GlobalUsings.cs`:

```csharp
global using MyApp.Helpers;
```

Теперь все файлы автоматически видят `MyApp.Helpers.*` без явного `using`.

### 2.6. ImplicitUsings — автоматические импорты

В csproj с `<ImplicitUsings>enable</ImplicitUsings>` SDK сам импортирует часто используемые namespace'ы:

- `System`
- `System.Collections.Generic`
- `System.Linq` ← LINQ extensions сюда
- `System.Net.Http`
- `System.Text`
- `System.Threading.Tasks`

Поэтому LINQ работает «из коробки» в новых проектах — `Linq.Enumerable.Where` импортирован неявно.

### 2.7. Naming convention

Соглашения для extension-классов:
- Класс — `<Тип>Extensions` или `<Цель>Extensions`: `StringExtensions`, `EnumerableExtensions`, `HttpClientExtensions`.
- Namespace — обычно тот же, что у расширяемого типа, или специальный helper-namespace.
- Один тип — один extension-класс (помогает поиску).

```csharp
// Хорошо
public static class StringExtensions { ... }
public static class HttpResponseMessageExtensions { ... }

// Плохо — Util ничего не говорит
public static class Util { ... }

// Плохо — Helpers сборная солянка
public static class Helpers { ... }
```

> [!question]- Интервью: что должно быть, чтобы метод стал extension?
> 1) Класс должен быть `static` (top-level или в другом static-классе, не nested в обычном). 2) Метод должен быть `static`. 3) Первый параметр должен начинаться с ключевого слова `this` перед типом. 4) Класс не должен быть generic. 5) Namespace класса должен быть импортирован через `using` (или global using) в месте вызова. Если хоть одно условие нарушено — компилятор не распознаёт метод как extension и выдаёт `CS1061: 'Type' does not contain a definition for 'X'`.

---

## 3. Внутреннее устройство

### 3.1. Что генерирует компилятор

Возьмём простую extension:

```csharp
public static class StringExt
{
    public static int WordCount(this string s) => s.Split(' ').Length;
}

class Program
{
    static void Main()
    {
        var n = "hello world".WordCount();
    }
}
```

В IL `Main` будет содержать вызов:

```il
ldstr "hello world"
call int32 StringExt::WordCount(string)   // обычный static call
```

То есть `"hello world".WordCount()` компилируется ровно так же, как если бы ты написал `StringExt.WordCount("hello world")`. **Никакой разницы в IL.**

### 3.2. Атрибут [Extension]

Чтобы compiler в **другой** assembly смог распознать твой метод как extension (например, при использовании DLL из NuGet), он должен пометить его специальным атрибутом. Это делается автоматически:

```csharp
public static class StringExt
{
    [System.Runtime.CompilerServices.Extension]   // компилятор сам ставит
    public static int WordCount(this string s) => ...;
}

[System.Runtime.CompilerServices.Extension]   // и на классе тоже
public static class StringExt { ... }
```

Поэтому формально **ключевое слово `this`** — это синтаксис для C#-компилятора, который в IL транслируется в `[ExtensionAttribute]`. Другие .NET-языки (F#, VB) этот атрибут читают и тоже распознают метод как extension.

### 3.3. Compile-time resolution — никакого runtime overhead

Когда ты пишешь `obj.Method()`, компилятор делает следующее:

1. Смотрит, есть ли у `obj.Method()` instance-метод с таким именем и совместимыми параметрами.
2. Если да — резолвит как обычный virtual / non-virtual call.
3. Если нет — ищет extension-методы в импортированных namespace'ах.
4. Среди найденных выбирает best match по правилам overload resolution.
5. Заменяет вызов на статический call.

Всё это — **на этапе компиляции**. Runtime не делает никаких дополнительных проверок, нет dispatch-таблицы для extensions, нет reflection.

### 3.4. Никакого override / virtual

Поскольку extensions — статические методы, у них **нет полиморфизма**:

```csharp
public class Animal { }
public class Dog : Animal { }

public static class AnimalExt
{
    public static string Name(this Animal a) => "Animal";
    public static string Name(this Dog d) => "Dog";
}

Animal a = new Dog();
Console.WriteLine(a.Name());   // "Animal" — не "Dog"!
```

Расскажу почему. Компилятор видит тип переменной `a` как `Animal` (а не `Dog`). Резолвит вызов `Name()` к `AnimalExt.Name(Animal)` на этапе компиляции. Реальный runtime-тип `Dog` не учитывается.

Это одна из ключевых ловушек. **Если хочешь полиморфизм — пиши обычный virtual метод**, не extension.

### 3.5. Когда compiler не находит метод

```csharp
"hello".WordCount();   // CS1061: 'string' does not contain a definition for 'WordCount'
                       // and no accessible extension method 'WordCount' accepting a first
                       // argument of type 'string' could be found (are you missing a using?)
```

Сообщение прямо подсказывает: «нет ни instance-метода, ни видимого extension». Чаще всего — забыл `using`. Иногда — `WordCount` определён в другой assembly, которую не подключил.

### 3.6. ExtensionAttribute и сторонние языки

`[Extension]` не часть language spec C#, а .NET runtime-атрибут. Это значит, что F# или VB.NET тоже могут видеть твои extensions:

```fsharp
// F# может вызвать твою extension
"hello".WordCount()   // работает, если StringExt в подключённой assembly
```

Кроссплатформенность через атрибуты — типично для .NET.

### 3.7. Performance — ноль overhead

```csharp
// Эти три варианта компилируются в идентичный IL
"hello".WordCount();
StringExt.WordCount("hello");
StringExt.WordCount(s: "hello");
```

Поэтому **нет performance-причин избегать extensions**. Это синтаксис, а не runtime-механизм.

> [!question]- Интервью: есть ли overhead у extension method по сравнению с instance method?
> Нет. Extension method компилируется в обычный статический вызов, в IL не остаётся никаких следов «extension». Performance идентичен static method'у. Instance method может быть virtual (через vtable lookup) — это чуть-чуть медленнее статического. Поэтому extension может быть **быстрее** virtual instance method, если расширяемый класс имеет много override-ов. На практике разница неизмерима, выбор между extension и instance делается по архитектуре, не по perf.

---

## 4. Generic extensions

### 4.1. Generic метод в non-generic классе

Самый частый паттерн — generic extension для generic коллекции:

```csharp
public static class EnumerableExt
{
    public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> source) where T : class
    {
        foreach (var item in source)
            if (item is not null)
                yield return item;
    }
}

string?[] arr = ["a", null, "b", null, "c"];
var nonNull = arr.WhereNotNull().ToList();   // ["a", "b", "c"]
```

Generic-параметр `T` — на методе. Класс остаётся обычным static.

### 4.2. Type inference

Компилятор обычно сам выводит generic-параметры из аргументов:

```csharp
public static T FirstOrDefault<T>(this IEnumerable<T> source, Func<T, bool> predicate) { ... }

var nums = new[] { 1, 2, 3 };
var first = nums.FirstOrDefault(x => x > 1);   // T выведено как int
```

Иногда compiler не может вывести — тогда указываешь явно:

```csharp
var json = "{}".FromJson<MyType>();   // FromJson<T>(this string)
```

### 4.3. Generic constraints

Можно ограничить тип T:

```csharp
public static class ComparableExt
{
    public static T Max<T>(this IEnumerable<T> source) where T : IComparable<T>
    {
        T max = source.First();
        foreach (var item in source)
            if (item.CompareTo(max) > 0) max = item;
        return max;
    }
}

new[] { 3, 1, 4, 1, 5, 9, 2, 6 }.Max();   // 9
```

### 4.4. Notnull / class / struct constraints

Полезные ограничения:

```csharp
// notnull — T не может быть nullable type
public static T Coalesce<T>(this T value, T fallback) where T : notnull
    => value;

// class — только reference types
public static T? FirstOrDefault<T>(this IEnumerable<T> source) where T : class { ... }

// struct — только value types
public static T? AsNullable<T>(this T value) where T : struct => value;

// new() — должен быть default constructor
public static T CreateClone<T>(this T source) where T : new() => new T();
```

### 4.5. Multiple constraints

```csharp
public static T DeepClone<T>(this T source)
    where T : class, ICloneable, new()
{
    return (T)source.Clone();
}
```

### 4.6. Constraint на nested generic

Можно ограничить generic-параметр другим generic:

```csharp
public static List<TResult> MapEach<T, TResult>(
    this IEnumerable<T> source,
    Func<T, TResult> selector)
    where TResult : new()
{
    return source.Select(selector).ToList();
}
```

### 4.7. Multiple generic-параметров

```csharp
public static Dictionary<TKey, TValue> ToDictionary<T, TKey, TValue>(
    this IEnumerable<T> source,
    Func<T, TKey> keySelector,
    Func<T, TValue> valueSelector)
    where TKey : notnull
{
    var dict = new Dictionary<TKey, TValue>();
    foreach (var item in source)
        dict.Add(keySelector(item), valueSelector(item));
    return dict;
}

new[] { "alice", "bob" }.ToDictionary(s => s.Length, s => s.ToUpper());
// Выводится: ToDictionary<string, int, string>
```

> [!question]- Интервью: как объявить generic extension method?
> Класс должен быть **обычным static (не generic)**, generic-параметр объявляется на методе: `public static T DoSomething<T>(this T value) ...`. Можно добавлять constraints через `where` (`class`, `struct`, `notnull`, `new()`, `interface`, `class : Base`). Type inference обычно резолвит generic-параметр из аргумента; если нет — нужно указать явно `obj.Method<T>()`. Generic class для extensions запрещён (CS1106).

---

## 5. Extensions для null

### 5.1. Можно вызывать на null!

Это удивляет джунов, но extension методы можно вызывать на `null` без NRE:

```csharp
public static class StringExt
{
    public static bool IsEmpty(this string? s) => string.IsNullOrEmpty(s);
}

string? x = null;
x.IsEmpty();   // True — НЕ NullReferenceException!
```

**Почему работает:** компилятор переводит `x.IsEmpty()` в `StringExt.IsEmpty(x)`. Это статический вызов с `null` в качестве аргумента — обычный путь, никакого dispatch на null-объект.

Сравни с instance-методом:

```csharp
string? x = null;
x.ToLower();   // NullReferenceException — virtual call на null
```

Здесь virtual call требует разыменования `x` (получить vtable), которое крашится на null.

### 5.2. Это и плюс, и ловушка

**Плюс:** удобно строить null-safe API:

```csharp
public static int SafeLength(this string? s) => s?.Length ?? 0;

string? name = null;
name.SafeLength();   // 0 — без NRE
```

**Ловушка:** если ты в extension-методе делаешь что-то с `this`-параметром, не проверив на null:

```csharp
public static int WordCount(this string s)
{
    return s.Split(' ').Length;   // ❌ NRE если s == null
}

((string?)null).WordCount();   // NullReferenceException
```

Extension может быть вызван на null. Метод **должен** это учитывать.

### 5.3. Best practice — null-aware

```csharp
public static class StringExt
{
    public static int WordCount(this string? s)
    {
        if (s is null) return 0;
        return s.Split(' ').Length;
    }
}

((string?)null).WordCount();   // 0
"hello world".WordCount();     // 2
```

Тип параметра — `string?` (nullable). Метод явно обрабатывает null.

Альтернатива — кинуть `ArgumentNullException`, если null недопустим:

```csharp
public static int WordCount(this string s)
{
    ArgumentNullException.ThrowIfNull(s);
    return s.Split(' ').Length;
}
```

Это явный контракт: «null не разрешён, передал — получай exception».

### 5.4. Nullable Reference Types и extensions

С NRT (Nullable Reference Types) ты должен **явно указать**, принимает ли extension nullable:

```csharp
#nullable enable

// Не принимает null
public static int WordCount(this string s) { ... }

// Принимает null
public static int WordCountOrZero(this string? s) { ... }

string? maybeNull = GetString();
maybeNull.WordCount();           // ⚠️ warning CS8604: возможный null
maybeNull.WordCountOrZero();     // OK
```

Compiler через NRT помогает поймать ошибки до runtime.

### 5.5. Reading: контракт extension должен быть очевиден

Когда видишь сигнатуру extension с `this T?` (nullable) — знаешь, что null разрешён. С `this T` (non-nullable + NRT) — null недопустим.

Это часть документации. В FCL:

```csharp
// string.IsNullOrEmpty принимает null
public static bool IsNullOrEmpty([NotNullWhen(false)] string? value);

// string.Length — instance, не extension; на null не вызовешь
```

> [!question]- Интервью: можно ли вызвать extension method на null?
> Да. Extension компилируется в статический метод, и compiler заменяет `x.Method()` на `Class.Method(x)`. Если `x == null`, статический метод получает null в качестве аргумента, без NRE. Метод **должен** это учитывать — обычно через nullable-параметр (`this string?`) и явную проверку на null. С instance-методом такого нет: virtual call на null крашится в момент vtable lookup. Это одна из killer-фич extensions для null-safe API.

---

## 6. Extensions для structs

### 6.1. Обычный extension — копирует struct

```csharp
public struct Point
{
    public int X;
    public int Y;
}

public static class PointExt
{
    public static void MoveRight(this Point p, int dx)
    {
        p.X += dx;   // ❌ меняет КОПИЮ
    }
}

var p = new Point { X = 0, Y = 0 };
p.MoveRight(10);
Console.WriteLine(p.X);   // 0! Не изменилось.
```

**Почему:** struct передаётся **по значению**. `this Point p` — это копия. Изменения в методе не видны вызывающему.

### 6.2. ref this — мутирующий extension (C# 7.3+)

Для мутаций нужен `ref this`:

```csharp
public static class PointExt
{
    public static void MoveRight(this ref Point p, int dx)
    {
        p.X += dx;   // меняет оригинал
    }
}

var p = new Point { X = 0, Y = 0 };
p.MoveRight(10);
Console.WriteLine(p.X);   // 10 — изменилось!
```

Ограничения `ref this`:
- Только для **structs**, не для классов (классы и так передаются по ссылке).
- Можно вызывать только на **mutable variable**, не на rvalue:
  ```csharp
  GetPoint().MoveRight(10);   // ❌ rvalue, не lvalue
  new Point().MoveRight(10);   // ❌
  ```

### 6.3. in this — read-only ref (C# 7.2+)

Если struct большая (несколько полей), копирование дорогое. `in this` передаёт по ссылке без права изменения:

```csharp
public struct LargeStruct
{
    public int A, B, C, D, E, F, G, H;
}

public static class LargeStructExt
{
    public static int Sum(this in LargeStruct s)
    {
        return s.A + s.B + s.C + s.D + s.E + s.F + s.G + s.H;
        // ❌ s.A = 0;  // CS8332: cannot assign to 'A', it's readonly
    }
}
```

Это микрооптимизация. Имеет смысл только для **больших** struct (> 16-24 байт) и **горячего** кода (миллионы вызовов).

### 6.4. Boxing в extensions для structs

```csharp
public interface IPrintable
{
    void Print();
}

public static class PrintableExt
{
    public static void PrintTwice(this IPrintable p)
    {
        p.Print();
        p.Print();
    }
}

public struct MyStruct : IPrintable
{
    public void Print() => Console.WriteLine("X");
}

new MyStruct().PrintTwice();   // ⚠️ boxing — struct → IPrintable
```

`IPrintable` — interface, ссылочный контракт. Передача `MyStruct` через `IPrintable` требует **boxing** (создание heap-объекта). На горячем коде это дорого.

Решение — generic constraint:

```csharp
public static void PrintTwice<T>(this T p) where T : IPrintable
{
    p.Print();
    p.Print();
}

new MyStruct().PrintTwice();   // без boxing — JIT генерирует специализацию для MyStruct
```

JIT генерирует отдельную версию метода для каждого value-type T, минуя boxing. Это паттерн `where T : IInterface` для perf-критичных extensions.

### 6.5. ref struct extensions

`ref struct` (например, `Span<T>`) — особый тип, который живёт только на стеке. Extensions для них работают, но с ограничениями:

```csharp
public static class SpanExt
{
    public static int SumWithBonus(this Span<int> span, int bonus)
    {
        int sum = bonus;
        foreach (var x in span) sum += x;
        return sum;
    }
}

Span<int> nums = stackalloc int[] { 1, 2, 3, 4, 5 };
nums.SumWithBonus(100);   // 115
```

`ref struct` параметр в extension — обычный, без специального ключевого слова.

> [!question]- Интервью: как сделать extension, который меняет struct?
> Использовать `this ref T` (C# 7.3+). Без `ref` extension получает копию struct, изменения не видны caller-у. С `ref` — оригинал, изменения сохраняются. Ограничения: можно вызывать только на mutable variable (не на возвращённом из метода значении или rvalue). Для read-only оптимизации — `this in T`, передаёт по ссылке без возможности модификации (избегает копирования больших struct). Это всё работает только для structs; для классов передача по ссылке естественна.

---

## 7. Конфликты имён и resolution

### 7.1. Instance method vs extension

Если есть и instance, и extension с одинаковой сигнатурой — **instance выигрывает всегда**:

```csharp
public class Box
{
    public int Count() => 42;   // instance
}

public static class BoxExt
{
    public static int Count(this Box b) => 100;   // extension
}

new Box().Count();   // 42 — instance method
```

Это design rationale: extensions не должны «хайджекать» поведение типа.

Также instance method **скрывает** extensions, даже если параметры разные:

```csharp
public class Box
{
    public int Count(string s) => s.Length;
}

public static class BoxExt
{
    public static int Count(this Box b) => 100;   // → доступен
}

new Box().Count();         // CS7036 — нужны параметры, потому что compiler нашёл метод (с string), а лучшего match нет
new Box().Count("test");   // 4 — instance
```

В реальности это не такая большая проблема, потому что compiler даст ошибку.

### 7.2. Несколько extensions с одинаковой сигнатурой

```csharp
namespace A
{
    public static class StringExt
    {
        public static string Foo(this string s) => "from A";
    }
}

namespace B
{
    public static class StringExt
    {
        public static string Foo(this string s) => "from B";
    }
}

// Если оба using — конфликт!
using A;
using B;

"x".Foo();   // CS0121: ambiguous between A.StringExt.Foo and B.StringExt.Foo
```

Решение — disambiguate через explicit static call:

```csharp
A.StringExt.Foo("x");   // OK
B.StringExt.Foo("x");   // OK
```

Или extension alias (C# 12 для типов, не для extensions, но можно через using static):

```csharp
using static A.StringExt;   // импортирует только из A
"x".Foo();   // вызовет A.StringExt.Foo
```

### 7.3. Resolution scope

Compiler ищет extensions в порядке:

1. **Inner namespace** (где код пишется).
2. **Nested namespaces** через using.
3. **`global using`-ы**.
4. **Implicit usings из SDK**.

Чем «ближе» extension к месту вызова, тем выше приоритет. Поэтому конфликты с BCL (`System.Linq`) в проектном коде маловероятны — твой `using` ближе.

### 7.4. Overload resolution внутри одного класса

```csharp
public static class Ext
{
    public static void Do(this object o) => Console.WriteLine("object");
    public static void Do(this string s) => Console.WriteLine("string");
}

((object)"hello").Do();   // "object"
"hello".Do();             // "string" — более специфичный тип выигрывает
```

Compiler выбирает **более специфичный** тип. Это стандартные правила overload resolution: `string` ближе к `object`, поэтому выигрывает при типе переменной `string`.

### 7.5. Conflict с методом из base class

```csharp
public class Animal
{
    public void Speak() => Console.WriteLine("Animal speaks");
}

public class Dog : Animal { }

public static class DogExt
{
    public static void Speak(this Dog d) => Console.WriteLine("Woof");
}

new Dog().Speak();   // "Animal speaks" — instance method из base class!
```

Extension не побеждает instance method даже из base class. Это поддерживает «extensions не хайджекают поведение».

### 7.6. Best practice — отдельный namespace для controversial extensions

Если extension может конфликтовать или ты не хочешь, чтобы он автоматически попадал в код:

```csharp
namespace MyApp.Helpers.Extras
{
    public static class StringExt
    {
        public static string Slugify(this string s) => ...;
    }
}
```

Чтобы использовать — нужен явный `using MyApp.Helpers.Extras;`. Это «opt-in», не «всегда видно».

В BCL: `System.Linq.Queryable` extensions для `IQueryable<T>` живут в отдельном namespace от `Enumerable` для `IEnumerable<T>` — чтобы можно было импортировать только нужное.

> [!question]- Интервью: что произойдёт, если есть и instance method, и extension с одинаковой сигнатурой?
> Instance method **всегда** побеждает. Это design rationale: extensions не должны хайджекать поведение типа. Compiler ищет instance methods первыми; только если ничего не нашёл (включая base class методы) — переходит к extensions. Это значит, что добавление instance method в новой версии библиотеки может «перекрыть» твою extension с тем же именем (но не сломает компиляцию — instance method просто будет вызываться вместо extension).

---

## 8. LINQ — построен на extensions

### 8.1. Минимальная LINQ-цепочка

```csharp
var result = numbers
    .Where(x => x > 0)
    .Select(x => x * 2)
    .ToList();
```

Каждый из `Where`, `Select`, `ToList` — extension method:

```csharp
namespace System.Linq;

public static class Enumerable
{
    public static IEnumerable<T> Where<T>(this IEnumerable<T> source, Func<T, bool> predicate) { ... }
    public static IEnumerable<TResult> Select<T, TResult>(this IEnumerable<T> source, Func<T, TResult> selector) { ... }
    public static List<T> ToList<T>(this IEnumerable<T> source) { ... }
}
```

Все возвращают `IEnumerable<T>` (или `List<T>` для terminal), поэтому можно chain'ить дальше.

### 8.2. IEnumerable vs IQueryable

В LINQ есть **две** параллельные иерархии extensions:

```csharp
// LINQ to Objects — System.Linq.Enumerable
IEnumerable<T> source = ...;
source.Where(...)   // выполняется in-memory

// LINQ to SQL / EF Core — System.Linq.Queryable
IQueryable<T> source = ...;
source.Where(...)   // транслируется в SQL
```

Один и тот же синтаксис, разные namespaces, разное поведение. EF Core видит, что `dbContext.Users` — это `IQueryable<User>`, и вызывает `Queryable.Where`, который преобразует expression tree в SQL.

### 8.3. Deferred execution через extensions

```csharp
var query = numbers.Where(x => x > 0);
// query — IEnumerable<int>, ничего ещё не вычислено

foreach (var n in query)   // здесь начинается итерация
    Console.WriteLine(n);
```

Это работает потому, что `Where` возвращает **lazy iterator** (внутренний класс с `yield return`). Каждый `MoveNext` вызывает predicate. Никакой материализации в коллекцию.

Терминальные методы (`ToList`, `ToArray`, `Sum`, `Count`, `First`) форсируют выполнение.

### 8.4. Custom LINQ — свой extension

Можно расширить LINQ своими методами:

```csharp
public static class MyLinqExt
{
    public static IEnumerable<T> WhereNot<T>(this IEnumerable<T> source, Func<T, bool> predicate)
    {
        return source.Where(item => !predicate(item));
    }

    public static IEnumerable<TResult> SelectIndexed<T, TResult>(
        this IEnumerable<T> source,
        Func<T, int, TResult> selector)
    {
        int index = 0;
        foreach (var item in source)
            yield return selector(item, index++);
    }
}

new[] { 1, 2, 3, 4, 5 }
    .WhereNot(x => x % 2 == 0)        // [1, 3, 5]
    .SelectIndexed((n, i) => $"{i}:{n}")
    .ToList();
// ["0:1", "1:3", "2:5"]
```

Многие проекты делают свой `LinqExtensions` с domain-specific помощниками.

### 8.5. Pipeline-like обработка

LINQ extensions позволяют выразить пошаговую обработку как композицию:

```csharp
var stats = orders
    .Where(o => o.Status == OrderStatus.Paid)
    .Where(o => o.Date >= DateTime.Today.AddDays(-30))
    .GroupBy(o => o.Customer.Country)
    .Select(g => new
    {
        Country = g.Key,
        Total = g.Sum(o => o.Amount),
        Count = g.Count(),
        Avg = g.Average(o => o.Amount)
    })
    .OrderByDescending(s => s.Total)
    .Take(10)
    .ToList();
```

Каждый шаг — отдельный extension. Pipeline читается сверху вниз, как steps.

### 8.6. Performance — chained extensions ≠ multiple iterations

```csharp
var result = list
    .Where(x => x > 0)
    .Where(x => x < 100)
    .Select(x => x * 2)
    .ToList();
```

Может показаться, что коллекция обходится 3 раза. Но **нет** — благодаря deferred execution через `yield return`, на каждом item все три фильтра/трансформации применяются в одном проходе. Финальный `ToList()` итерирует один раз.

Это ключ к производительности LINQ. Не путай с `.ToList().Where(...).ToList().Select(...).ToList()` — это уже 3 прохода.

> [!question]- Интервью: почему LINQ невозможен без extension methods?
> LINQ строится на fluent-цепочках `source.Where(...).Select(...).ToList()`. Без extensions пришлось бы писать `Enumerable.ToList(Enumerable.Select(Enumerable.Where(source, p), s))` — обратная вложенность, нечитаемо. Extensions делают `Where`, `Select`, `ToList` похожими на методы коллекции, которые можно chain'ить слева направо. Также extensions работают для **любого** `IEnumerable<T>` без модификации интерфейса — расширили BCL, не меняя `IEnumerable`. Это краеугольный камень LINQ design.

---

## 9. Fluent API через extensions

### 9.1. ASP.NET Core middleware pipeline

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddControllers()
    .AddJsonOptions(opts => opts.JsonSerializerOptions.PropertyNameCaseInsensitive = true);

builder.Services.AddDbContext<AppDb>(opts =>
    opts.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

var app = builder.Build();

app.UseHttpsRedirection()
   .UseAuthentication()
   .UseAuthorization()
   .MapControllers();

app.Run();
```

Большинство методов здесь — extensions. Например:

```csharp
public static class WebApplicationBuilderExtensions
{
    public static IServiceCollection AddControllers(this IServiceCollection services) { ... }
}

public static class JsonMvcCoreBuilderExtensions
{
    public static IMvcCoreBuilder AddJsonOptions(this IMvcCoreBuilder builder, Action<JsonOptions> opts) { ... }
}
```

Они возвращают тот же объект (или другой builder) для продолжения цепочки.

### 9.2. Builder pattern

```csharp
public class HttpRequestBuilder
{
    private string _url = "";
    private HttpMethod _method = HttpMethod.Get;
    private readonly Dictionary<string, string> _headers = new();
    private object? _body;

    public HttpRequest Build() => new(...);
}

public static class HttpRequestBuilderExtensions
{
    public static HttpRequestBuilder Url(this HttpRequestBuilder b, string url) { b._url = url; return b; }
    public static HttpRequestBuilder Method(this HttpRequestBuilder b, HttpMethod m) { b._method = m; return b; }
    public static HttpRequestBuilder Header(this HttpRequestBuilder b, string key, string value) { b._headers[key] = value; return b; }
    public static HttpRequestBuilder Body(this HttpRequestBuilder b, object body) { b._body = body; return b; }
    public static HttpRequestBuilder ContentType(this HttpRequestBuilder b, string ct) => b.Header("Content-Type", ct);
}

var request = new HttpRequestBuilder()
    .Url("https://api.example.com")
    .Method(HttpMethod.Post)
    .ContentType("application/json")
    .Body(new { name = "Alice" })
    .Build();
```

Extensions удобны тем, что:
- Можно добавлять новые методы **без** модификации основного класса.
- Domain-specific extensions держат основной класс маленьким.
- Тесты могут использовать свои extensions для readability.

### 9.3. Возврат `this` для chain

Главное правило fluent API: метод **возвращает this** (или другой builder, готовый к chain).

```csharp
public static class StringBuilderExt
{
    public static StringBuilder AppendIfNotEmpty(this StringBuilder sb, string s)
    {
        if (!string.IsNullOrEmpty(s)) sb.Append(s);
        return sb;   // ← возвращаем sb, чтобы можно было chain
    }

    public static StringBuilder AppendLineIf(this StringBuilder sb, bool condition, string s)
    {
        if (condition) sb.AppendLine(s);
        return sb;
    }
}

var result = new StringBuilder()
    .Append("Hello, ")
    .AppendIfNotEmpty(name)
    .AppendLine()
    .AppendLineIf(includeTime, $"Time: {DateTime.Now}")
    .ToString();
```

### 9.4. Dependency Injection registrations

```csharp
public static class ServiceCollectionExtensions
{
    public static IServiceCollection AddOrderService(this IServiceCollection services, IConfiguration config)
    {
        services.AddScoped<IOrderService, OrderService>();
        services.AddScoped<IOrderValidator, OrderValidator>();
        services.AddSingleton<IOrderPriceCalculator, DefaultOrderPriceCalculator>();
        services.Configure<OrderOptions>(config.GetSection("Orders"));
        return services;
    }

    public static IServiceCollection AddPaymentService(this IServiceCollection services, IConfiguration config)
    {
        services.AddScoped<IPaymentGateway, StripeGateway>();
        services.Configure<StripeOptions>(config.GetSection("Stripe"));
        return services;
    }
}

// Использование
builder.Services
    .AddOrderService(builder.Configuration)
    .AddPaymentService(builder.Configuration)
    .AddCustomerService(builder.Configuration);
```

Это стандартный паттерн в .NET-проектах: каждая «фича» — свой extension `Add<Feature>Service`, регистрирующий все её зависимости и опции в одном месте. `Program.cs` остаётся коротким.

### 9.5. Configuration через extensions

EF Core — пример:

```csharp
builder.Services.AddDbContext<AppDb>(options => options
    .UseSqlServer(connStr, sql => sql
        .EnableRetryOnFailure(3)
        .CommandTimeout(30))
    .UseSnakeCaseNamingConvention()
    .EnableSensitiveDataLogging(builder.Environment.IsDevelopment()));
```

Все эти `Use*` и `Enable*` — extension methods на `DbContextOptionsBuilder`.

### 9.6. Когда fluent API оправдан

✅ **Хорошо для:**
- Builder pattern (нужно собрать сложный объект пошагово).
- Configuration-heavy API.
- Pipeline (LINQ-стиль).
- Test setup (Arrange step становится читаемее).

❌ **Не нужен для:**
- Простых CRUD-операций.
- Когда параметры все обязательны (лучше конструктор / record).
- Domain-логики, где порядок операций не важен.

Не переусердствуй. Fluent API хорош для сложного API, но если у тебя метод с двумя параметрами — обычная сигнатура лучше.

> [!question]- Интервью: как реализовать builder pattern через extension methods?
> 1) Главный класс хранит state (поля). 2) Extension methods на этом классе принимают `this Builder b`, изменяют state и возвращают тот же builder для цепочки. 3) Финальный метод `Build()` создаёт целевой объект из state. Почему extensions, а не методы класса: можно добавлять новые методы конфигурации без модификации основного класса (open-closed principle), domain-specific extensions для разных модулей, тесты могут расширять builder своими методами для readability. ASP.NET Core, EF Core, AutoMapper, FluentValidation — всё построено на этом паттерне.

---

## 10. Domain-friendly extensions

### 10.1. Time / Duration

```csharp
public static class TimeExt
{
    public static TimeSpan Seconds(this int n) => TimeSpan.FromSeconds(n);
    public static TimeSpan Minutes(this int n) => TimeSpan.FromMinutes(n);
    public static TimeSpan Hours(this int n) => TimeSpan.FromHours(n);
    public static TimeSpan Days(this int n) => TimeSpan.FromDays(n);

    public static DateTime Ago(this TimeSpan t) => DateTime.Now - t;
    public static DateTime FromNow(this TimeSpan t) => DateTime.Now + t;
}

await Task.Delay(5.Seconds());
var tomorrow = 1.Days().FromNow();
var lastWeek = 7.Days().Ago();
var deadline = DateTime.Now + 2.Hours() + 30.Minutes();
```

Это вдохновлено Ruby on Rails. Делает код читаемым на грани естественного языка. Используется во многих .NET-проектах (Quartz.NET, Polly, Hangfire).

### 10.2. Money / Currency

```csharp
public readonly record struct Money(decimal Amount, string Currency);

public static class MoneyExt
{
    public static Money USD(this decimal amount) => new(amount, "USD");
    public static Money EUR(this decimal amount) => new(amount, "EUR");
    public static Money RUB(this decimal amount) => new(amount, "RUB");

    public static Money Plus(this Money a, Money b)
    {
        if (a.Currency != b.Currency) throw new InvalidOperationException("Currency mismatch");
        return new Money(a.Amount + b.Amount, a.Currency);
    }
}

var total = 100m.USD().Plus(50m.USD());   // Money { 150, "USD" }
var bug = 100m.USD().Plus(50m.EUR());     // throws — типобезопасный для одного currency
```

Доменные конструкторы через extensions делают код декларативным.

### 10.3. Validation / Guards

```csharp
public static class GuardExt
{
    public static T NotNull<T>(this T? value, string paramName) where T : class
        => value ?? throw new ArgumentNullException(paramName);

    public static string NotEmpty(this string? value, string paramName)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("must not be empty", paramName);
        return value;
    }

    public static int Positive(this int value, string paramName)
    {
        if (value <= 0) throw new ArgumentOutOfRangeException(paramName, "must be positive");
        return value;
    }
}

public class OrderService
{
    public OrderService(IOrderRepo repo, ILogger<OrderService> logger)
    {
        _repo = repo.NotNull(nameof(repo));
        _logger = logger.NotNull(nameof(logger));
    }

    public Order Create(string customer, int quantity)
    {
        customer.NotEmpty(nameof(customer));
        quantity.Positive(nameof(quantity));
        // ...
    }
}
```

Это альтернатива `if (x == null) throw ...` boilerplate. С .NET 6+ есть `ArgumentNullException.ThrowIfNull(...)` — встроенный аналог.

### 10.4. Collection helpers

```csharp
public static class CollectionExt
{
    public static bool IsNullOrEmpty<T>(this IEnumerable<T>? source)
        => source is null || !source.Any();

    public static bool HasAny<T>(this IEnumerable<T>? source)
        => source is not null && source.Any();

    public static IEnumerable<T> EmptyIfNull<T>(this IEnumerable<T>? source)
        => source ?? [];

    public static void AddRange<T>(this ICollection<T> target, IEnumerable<T> items)
    {
        foreach (var item in items) target.Add(item);
    }

    public static int IndexOf<T>(this IEnumerable<T> source, Func<T, bool> predicate)
    {
        int i = 0;
        foreach (var item in source)
        {
            if (predicate(item)) return i;
            i++;
        }
        return -1;
    }
}
```

`IsNullOrEmpty<T>` — доменный аналог `string.IsNullOrEmpty` для коллекций. `EmptyIfNull` — defensive programming, удобно перед foreach.

### 10.5. Enum extensions

```csharp
public static class EnumExt
{
    public static bool In<T>(this T value, params T[] options) where T : Enum
        => options.Contains(value);

    public static string ToHumanReadable<T>(this T value) where T : Enum
    {
        var field = typeof(T).GetField(value.ToString());
        var attr = field?.GetCustomAttribute<DescriptionAttribute>();
        return attr?.Description ?? value.ToString();
    }
}

enum OrderStatus { Pending, Paid, Shipped, Delivered, Cancelled }

OrderStatus s = OrderStatus.Paid;
s.In(OrderStatus.Paid, OrderStatus.Shipped);   // True
s.ToHumanReadable();                            // "Paid"
```

Удобно для UI / serialization / business rules.

### 10.6. Conversion / parsing

```csharp
public static class ConvertExt
{
    public static int? TryParseInt(this string? s) =>
        int.TryParse(s, out var n) ? n : null;

    public static decimal? TryParseDecimal(this string? s) =>
        decimal.TryParse(s, NumberStyles.Any, CultureInfo.InvariantCulture, out var d) ? d : null;

    public static T FromJson<T>(this string json) =>
        JsonSerializer.Deserialize<T>(json) ?? throw new InvalidOperationException("Failed to parse JSON");

    public static string ToJson<T>(this T value) =>
        JsonSerializer.Serialize(value);
}

"42".TryParseInt();           // 42
"hello".TryParseInt();         // null
data.ToJson();                 // serialize
"{...}".FromJson<MyType>();    // deserialize
```

### 10.7. Boundary — где остановиться

Не превращай весь код в extension methods. Хорошие domain extensions:

- Связаны с одним доменным понятием (Time, Money, Order).
- Имеют явное имя, читаемое в контексте.
- Не дублируют instance-метод.
- Помещаются в одной команде разработчиков (не из «общей» библиотеки).

Плохие — `int.IsEvenAndPositive()`, который применяется один раз в одном месте.

> [!question]- Интервью: как сделать DSL (domain-specific language) через extensions?
> Через extension methods на примитивных или domain-типах, которые читаются как естественный язык. Примеры: `5.Seconds()` (Ruby on Rails-style), `100m.USD()` для Money, `value.NotNull(name)` для guards. Главное — придумать читаемый синтаксис, чтобы цепочка выглядела декларативно. Не злоупотреблять: extensions хороши для domain-критичных операций, не для разовых утилит. В .NET примеры DSL через extensions: Quartz.NET для расписаний, Polly для retry-policies, FluentAssertions для тестов, AutoMapper для маппинга.

---

## 11. Adapter / Converter pattern

### 11.1. .AsX() — view conversion

```csharp
public static class StringExt
{
    public static ReadOnlySpan<char> AsSpan(this string s) => s.AsSpan();   // встроенно
    public static IEnumerable<char> AsChars(this string s) => s;
}
```

`As`-extensions создают **view** на существующий объект, без копирования данных. В BCL — `string.AsSpan()`, `Memory<T>.AsMemory()`.

### 11.2. .ToX() — копирующая конверсия

```csharp
public static class EnumerableExt
{
    public static List<T> ToList<T>(this IEnumerable<T> source) => new(source);
    public static T[] ToArray<T>(this IEnumerable<T> source) => source.ToArray();
    public static HashSet<T> ToHashSet<T>(this IEnumerable<T> source) => new(source);
}
```

`To`-extensions создают **новую** коллекцию из существующей. Семантика: «копия».

Соглашение по naming:
- `As*` → view, не копирует, та же память.
- `To*` → копия, новый объект.

Это часть BCL convention. Следуй ей в своих extensions.

### 11.3. Conversion DTO ↔ Domain

```csharp
public class User { public int Id; public string Name = ""; public string Email = ""; }
public record UserDto(int Id, string Name, string Email);

public static class UserMapping
{
    public static UserDto ToDto(this User user) => new(user.Id, user.Name, user.Email);

    public static User ToDomain(this UserDto dto)
        => new() { Id = dto.Id, Name = dto.Name, Email = dto.Email };

    public static IEnumerable<UserDto> ToDto(this IEnumerable<User> users)
        => users.Select(u => u.ToDto());
}

User u = ...;
UserDto dto = u.ToDto();
List<UserDto> dtos = users.ToDto().ToList();
```

Это альтернатива AutoMapper для простых случаев. Преимущество: полная типобезопасность, видно компилятору, можно отлаживать.

### 11.4. Async-friendly

```csharp
public static class HttpClientExt
{
    public static async Task<T?> GetJsonAsync<T>(this HttpClient client, string url, CancellationToken ct = default)
    {
        var response = await client.GetAsync(url, ct);
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<T>(cancellationToken: ct);
    }

    public static async Task<HttpResponseMessage> PostJsonAsync<T>(
        this HttpClient client, string url, T data, CancellationToken ct = default)
    {
        var json = JsonSerializer.Serialize(data);
        var content = new StringContent(json, Encoding.UTF8, "application/json");
        return await client.PostAsync(url, content, ct);
    }
}

var user = await httpClient.GetJsonAsync<User>("/users/42");
await httpClient.PostJsonAsync("/orders", new { items = new[] { 1, 2, 3 } });
```

Async extensions — стандартный паттерн для wrapper над HttpClient / EF Core / DbConnection.

### 11.5. ServiceProvider / DI extensions

```csharp
public static class ServiceProviderExt
{
    public static T GetRequired<T>(this IServiceProvider sp) where T : notnull
        => sp.GetService<T>() ?? throw new InvalidOperationException($"Service {typeof(T).Name} not registered");

    public static async Task<T> WithScopeAsync<T>(
        this IServiceProvider sp,
        Func<IServiceScope, Task<T>> action)
    {
        await using var scope = sp.CreateAsyncScope();
        return await action(scope);
    }
}

var db = sp.GetRequired<AppDb>();

await sp.WithScopeAsync(async scope =>
{
    var db = scope.ServiceProvider.GetRequired<AppDb>();
    return await db.Users.ToListAsync();
});
```

Wrapping повторяющиеся patterns в extensions.

### 11.6. Result / Option (functional)

```csharp
public abstract record `Result<T>`;
public record Ok<T>(T Value) : `Result<T>`;
public record Err<T>(string Error) : `Result<T>`;

public static class ResultExt
{
    public static Result<TResult> Map<T, TResult>(this `Result<T>` result, Func<T, TResult> mapper)
        => result switch
        {
            Ok<T> ok => new Ok<TResult>(mapper(ok.Value)),
            Err<T> err => new Err<TResult>(err.Error),
            _ => throw new InvalidOperationException()
        };

    public static Result<TResult> Bind<T, TResult>(this `Result<T>` result, Func<T, Result<TResult>> binder)
        => result switch
        {
            Ok<T> ok => binder(ok.Value),
            Err<T> err => new Err<TResult>(err.Error),
            _ => throw new InvalidOperationException()
        };

    public static T ValueOr<T>(this `Result<T>` result, T fallback)
        => result is Ok<T> ok ? ok.Value : fallback;
}

Result<int> ParseAndDouble(string s) =>
    int.TryParse(s, out var n)
        ? new Ok<int>(n).Map(x => x * 2)
        : new Err<int>("Not a number");
```

Функциональный стиль — full extensions для composition.

> [!question]- Интервью: чем `As*` отличается от `To*` в naming convention?
> `As*` — view конверсия, не копирует данные, возвращает обёртку над тем же memory. Пример: `string.AsSpan()` создаёт `ReadOnlySpan<char>`, ссылающийся на ту же память. Дешёвая операция (O(1)), но обёртка живёт пока живёт оригинал. `To*` — копирующая конверсия, создаёт новый объект с собственными данными. Пример: `IEnumerable.ToList()` создаёт новый `List<T>` с копиями элементов. Дороже (O(n)), но независимая от оригинала. BCL строго следует этой конвенции, своим extensions тоже стоит.

---

## 12. Async и cancellation

### 12.1. Async extension methods

```csharp
public static class TaskExt
{
    public static async Task<T> WithTimeout<T>(this Task<T> task, TimeSpan timeout)
    {
        using var cts = new CancellationTokenSource(timeout);
        try
        {
            return await task.WaitAsync(cts.Token);
        }
        catch (OperationCanceledException) when (cts.IsCancellationRequested)
        {
            throw new TimeoutException();
        }
    }

    public static Task<T> WithFallback<T>(this Task<T> task, T fallback) =>
        task.ContinueWith(t => t.IsFaulted ? fallback : t.Result);
}

var data = await SlowOperation().WithTimeout(5.Seconds());
var result = await RiskyOperation().WithFallback(default!);
```

Это естественный паттерн обогащения Task без модификации BCL.

### 12.2. CancellationToken — last param convention

```csharp
public static async Task<T> RetryAsync<T>(
    this Func<Task<T>> action,
    int maxAttempts = 3,
    CancellationToken ct = default)   // ← всегда последний, default = none
{
    for (int i = 0; i < maxAttempts; i++)
    {
        try
        {
            return await action();
        }
        catch when (i < maxAttempts - 1 && !ct.IsCancellationRequested)
        {
            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, i)), ct);
        }
    }
    throw new InvalidOperationException("Max attempts exceeded");
}

var data = await ((Func<Task<string>>)(() => httpClient.GetStringAsync(url))).RetryAsync(maxAttempts: 5, ct);
```

`CancellationToken` — последний параметр с default value. Это стандарт BCL и сообщества.

### 12.3. ValueTask extensions

```csharp
public static class ValueTaskExt
{
    public static async ValueTask<T> WithFallbackAsync<T>(
        this ValueTask<T> vt, T fallback)
    {
        try { return await vt; }
        catch { return fallback; }
    }
}
```

`ValueTask` для hot-path, где аллокация Task критична (cache hits, sync-completing operations). Extensions работают так же.

### 12.4. ConfigureAwait в extensions

```csharp
public static async Task<T> WithLogging<T>(this Task<T> task, ILogger logger)
{
    var result = await task.ConfigureAwait(false);
    logger.LogDebug("Operation completed");
    return result;
}
```

В библиотечных extensions — всегда `ConfigureAwait(false)` (раздел async-debugging). Не зависят от SynchronizationContext caller-а.

В application-коде (особенно ASP.NET Core где SynchronizationContext null) — не обязательно, но и не вредит.

### 12.5. IAsyncEnumerable extensions

```csharp
public static class AsyncEnumerableExt
{
    public static async IAsyncEnumerable<T> WhereAsync<T>(
        this IAsyncEnumerable<T> source,
        Func<T, ValueTask<bool>> predicate,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        await foreach (var item in source.WithCancellation(ct))
        {
            if (await predicate(item))
                yield return item;
        }
    }
}

await foreach (var user in db.Users.AsAsyncEnumerable()
    .WhereAsync(async u => await IsActiveAsync(u, ct), ct))
{
    // обработка
}
```

`[EnumeratorCancellation]` атрибут — позволяет caller-у через `WithCancellation(ct)` передать токен в extension.

> [!question]- Интервью: как правильно объявлять CancellationToken в extension method?
> Последний параметр с default value: `CancellationToken ct = default`. Это стандарт BCL и .NET community. Внутри метода передавай `ct` всем awaited операциям (`HttpClient.GetAsync(url, ct)`, `Task.Delay(time, ct)`). Для `IAsyncEnumerable` — параметр должен быть с атрибутом `[EnumeratorCancellation]`, чтобы caller мог передать ct через `.WithCancellation(ct)`. Игнорировать ct — анти-паттерн: операция не отменяется, держит ресурсы.

---

## 13. Common Pitfalls — с механизмами

### 13.1. Extension не виден — забыл using

```csharp
// MyApp.Program.cs
"hello".WordCount();   // CS1061
```

**Механизм:** compiler ищет extensions только в импортированных namespace'ах. Без `using MyApp.Helpers;` — не находит.

**Фикс:**
```csharp
using MyApp.Helpers;
```

Или `global using` в проекте.

### 13.2. Полиморфизм не работает

```csharp
Animal a = new Dog();
a.Name();   // вызовет Animal-версию, не Dog
```

**Механизм:** extension резолвится по **compile-time** типу переменной. Runtime-тип не учитывается.

**Фикс:** если нужен полиморфизм — virtual instance method, не extension. Или explicit pattern matching:

```csharp
string GetName(Animal a) => a switch
{
    Dog d => "Dog",
    Cat c => "Cat",
    _ => "Animal"
};
```

### 13.3. NRE на `this` без проверки

```csharp
public static int WordCount(this string s) => s.Split(' ').Length;

((string?)null).WordCount();   // NullReferenceException на s.Split
```

**Механизм:** extension может быть вызван на null. Метод **не получает** null-protection бесплатно.

**Фикс:**
```csharp
public static int WordCount(this string? s)
{
    if (s is null) return 0;
    return s.Split(' ').Length;
}
```

### 13.4. Mutating struct без ref this

```csharp
public static void Increment(this Point p) => p.X++;

var p = new Point();
p.Increment();   // p.X всё ещё 0!
```

**Механизм:** struct передаётся по значению. `p.X++` мутирует копию.

**Фикс:** `this ref Point p` (C# 7.3+).

### 13.5. Instance method перекрывает extension

```csharp
// Твоя extension
public static void DoStuff(this MyClass m) { ... }

// В новой версии библиотеки добавили MyClass.DoStuff() instance method
// Твоя extension перестаёт вызываться (без compile error!)
```

**Механизм:** instance method всегда выигрывает, extension становится «dead code».

**Фикс:** переименовать extension или явно вызывать `MyClassExt.DoStuff(obj)`.

### 13.6. Несколько extensions с одинаковой сигнатурой

```csharp
using A;
using B;

"x".Foo();   // CS0121: ambiguous
```

**Механизм:** compiler не может выбрать между двумя одинаково подходящими.

**Фикс:** explicit static call (`A.StringExt.Foo("x")`) или `using static A.StringExt;` чтобы импортировать только из одного namespace.

### 13.7. Generic extension не выводит тип

```csharp
public static T Cast<T>(this object obj) => (T)obj;

obj.Cast();   // CS0411: Cannot infer T
```

**Механизм:** в этой сигнатуре T нельзя вывести из аргумента (он `object`). Нужно явно указать.

**Фикс:**
```csharp
obj.Cast<MyType>();
```

### 13.8. Extension на interface vs concrete type

```csharp
public static class EnumerableExt
{
    public static int CountFast<T>(this IEnumerable<T> source) => source.Count();
}

public static class ListExt
{
    public static int CountFast<T>(this List<T> list) => list.Count;
}

new List<int>().CountFast();   // List version (более специфичный)
```

**Механизм:** compiler выбирает более специфичный тип. `List<int>` ближе чем `IEnumerable<int>`.

Это часто используется в BCL для оптимизаций (`List.Count` — O(1), `IEnumerable.Count()` — O(n) в общем случае).

### 13.9. Mutating extension в LINQ

```csharp
public static int IncrementAndReturn(this int n) => n + 1;

var list = new[] { 1, 2, 3 }.Select(x => x.IncrementAndReturn()).ToList();
// OK — extension просто вычисляет значение

public static int IncrementInPlace(this int[] arr, int idx)
{
    arr[idx]++;
    return arr[idx];
}

// ⚠️ Side effect внутри LINQ — антипаттерн
var result = new[] { 1, 2, 3 }.Select(x => arr.IncrementInPlace(0)).ToList();
```

**Механизм:** LINQ ожидает чистые функции. Side effects внутри Select/Where ведут к непредсказуемому поведению.

**Фикс:** не делай side effects в LINQ. Если нужно — `foreach` или `ForEach` extension.

### 13.10. Hidden allocations в naive extensions

```csharp
// Аллоцирует List на каждый вызов
public static IEnumerable<T> WithoutNulls<T>(this IEnumerable<T?> source) where T : class
{
    var result = new List<T>();
    foreach (var item in source)
        if (item is not null) result.Add(item);
    return result;
}

// Лучше — yield return, deferred execution
public static IEnumerable<T> WithoutNulls<T>(this IEnumerable<T?> source) where T : class
{
    foreach (var item in source)
        if (item is not null) yield return item;
}
```

**Механизм:** `new List<T>()` — heap allocation. Если extension вызывается в hot loop, это GC pressure.

**Фикс:** использовать `yield return` для lazy evaluation (или возвращать `Span<T>`/`ReadOnlySpan<T>` для perf-critical случаев).

> [!question]- Интервью: топ-3 ловушки extension methods.
> 1) Polymorphism не работает — extension резолвится по compile-time типу переменной, не runtime. Animal a = new Dog(); a.Method() вызовет Animal-версию. 2) NRE на null this не возникает автоматически — метод компилируется как static call с null-аргументом. Метод сам должен проверять. 3) Instance method всегда побеждает extension — добавление instance в новой версии библиотеки делает твою extension dead code (без compile error).

---

## 14. Best Practices

- **Naming:** класс — `<Тип>Extensions` (`StringExtensions`, `HttpClientExtensions`). Метод — глагол или адекватное имя (`WordCount`, `ToDto`, `WithTimeout`).
- **Namespace:** обычно тот же, что у расширяемого типа, или специальный helper. Для BCL extensions — отдельный namespace, не загромождать root.
- **`As*` vs `To*`:** `As*` — view (no copy), `To*` — copy (new object). Следуй BCL convention.
- **`this T?` для null-aware extensions** — явный контракт, что null разрешён.
- **`this ref T` для mutating struct** — иначе изменения теряются.
- **`this in T` для read-only ref больших struct** — оптимизация, без копии.
- **Возвращай `this`** для fluent API — даёт возможность chain.
- **CancellationToken — последний параметр с `= default`** — convention.
- **Extension на interface** (`IEnumerable<T>`) обычно полезнее, чем на конкретном типе.
- **Generic constraint `where T : IInterface`** — избегает boxing для structs.
- **Deferred execution через `yield return`** — для коллекций.
- **Не злоупотребляй DSL-стилем** — `5.Seconds()` хорошо в правильных местах, плохо везде.
- **Один extension class на расширяемый тип** — `StringExt`, не сборная солянка.
- **Документируй XML-комментариями** — extensions появляются в IntelliSense, и contributors не знают намерения.
- **Не добавляй extension там, где можешь добавить instance method** — extension только для чужих типов или fluent API.
- **Тестируй extensions как обычные методы** — они и есть обычные static методы.
- **Не делай side effects в LINQ-style extensions** — нарушает ожидания caller'а.
- **Избегай `new List<T>()` без причины** — `yield return` лучше для lazy evaluation.
- **NRT-аннотации обязательны** — `this string?` или `this string` определяет контракт.

---

## 15. Decision tree

```
Нужно добавить функциональность к типу?
│
├── Это твой тип, исходники доступны
│   ├── Поведение — добавь instance method
│   ├── Утилита, относящаяся к типу — добавь instance или static
│   └── Helper, не относящийся к типу — обычный static (не extension)
│
├── Чужой тип (BCL, NuGet, third-party)
│   ├── Нужен fluent API — extension с `return this`
│   ├── Domain-specific helper — extension в своём namespace
│   ├── Adapter / converter — extension с `As*`/`To*` naming
│   └── Заменяет instance method — НЕЛЬЗЯ (instance всегда побеждает)
│
└── Generic коллекции / sequences
    ├── LINQ-style операция — extension на IEnumerable<T>
    ├── С constraint — `where T : ...` для специфичных типов
    └── Async — extension на IAsyncEnumerable<T> с EnumeratorCancellation

Mutability нужна?
│
├── Класс — обычная extension работает
├── Struct read-only — `this in T` для perf
├── Struct мутируется — `this ref T` (C# 7.3+)
└── ref struct (Span<T>) — обычная сигнатура с T = Span<T>

Полиморфизм нужен?
│
└── НЕТ — extensions не полиморфны.
    Если нужен — virtual instance method или pattern matching в helper.

Null безопасность?
│
├── null допустим — `this T?` + проверка в начале
├── null НЕ допустим — `this T` + ArgumentNullException.ThrowIfNull
└── С NRT — компилятор поможет caller-у видеть контракт
```

---

## 16. C# 14 — Extension members

### 16.1. Что нового

C# 14 вышел в релиз вместе с .NET 10 в ноябре 2025 — extension members это **released-фича**, не preview. Теперь extensions могут быть не только методами: появились **extension properties, static extension members и extension operators**. Синтаксис — `extension`-блок **внутри обычного static class**:

```csharp
public static class StringExtensions
{
    // Extension-блок: receiver объявляется один раз для всех членов
    extension(string s)
    {
        // Extension property — до C# 14 было невозможно
        public bool IsEmpty => s.Length == 0;

        // Extension method — эквивалент старого `this string s`
        public string Truncate(int max) => s.Length <= max ? s : s[..max];
    }
}

bool a = "hello".IsEmpty;              // false — property, без скобок
bool b = "".IsEmpty;                   // true
string t = "hello world".Truncate(5);  // "hello"
```

Ключевое отличие от старого стиля: receiver (`string s`) объявлен один раз в заголовке блока и доступен всем членам — не нужно повторять `this string s` в каждой сигнатуре. Generic-блоки тоже поддерживаются: `extension<T>(IEnumerable<T> source) { ... }`. Extension-блок может жить только в non-nested, non-generic static class — то есть в том же самом классе, где живут и старые `this`-методы.

### 16.2. Зачем

До C# 14 extensions могли быть только методами — нельзя было сделать `string.IsEmpty` как property, только `IsEmpty()`. Это лимитировало API design: «родные» члены типа выглядят как properties (`Length`), а расширения — только как методы со скобками. Теперь extension-члены могут быть неотличимы от родных, и fluent API смешивает properties и methods без швов.

### 16.3. Static extension members

Блок `extension(Type)` **без имени параметра** добавляет static-члены к самому типу:

```csharp
public static class StringExtensions
{
    extension(string)
    {
        public static bool HasValue(string? value) => !string.IsNullOrEmpty(value);
    }
}

bool ok = string.HasValue("hi");   // true — вызов выглядит как static-метод string
```

### 16.4. Extension operators

Внутри static-блока можно объявлять user-defined операторы для чужих типов:

```csharp
public static class PathExtensions
{
    extension(string)
    {
        public static string operator /(string left, string right)
            => Path.Combine(left, right);
    }
}

string path = "logs" / "2026" / "app.txt";   // Path.Combine через оператор /
```

Это огромное расширение возможностей — операторы к чужим типам раньше были невозможны в принципе. Что **не** вошло в C# 14: extension indexers и extension constructors — отложены на будущие версии языка.

### 16.5. Связь со старым синтаксисом

Старый синтаксис (`this T` первым параметром) **остаётся полностью валидным и никуда не девается** — оба стиля компилируются в одинаковые статические методы и могут сосуществовать в одном static class:

```csharp
public static class MixedExtensions
{
    // Старый стиль — по-прежнему норма для одиночных методов
    public static int WordCount(this string s) => s.Split(' ').Length;

    // Новый стиль — когда нужны property/static/operator или общий receiver
    extension(string s)
    {
        public bool IsEmpty => s.Length == 0;
    }
}
```

Практическое правило: одиночный метод-расширение — старый стиль читается проще; группа членов с общим receiver, property, static-член или оператор — extension-блок.

> [!question]- Интервью: что такое extension members в C# 14?
> Расширение концепции extensions, вышедшее в релиз с C# 14 / .NET 10 (ноябрь 2025). Внутри обычного static class объявляется блок `extension(string s) { ... }`: receiver указывается один раз, а члены блока могут быть не только методами, но и properties, static-членами и user-defined операторами. Блок `extension(Type)` без имени параметра добавляет static-члены к самому типу. Это закрывает gap в API design: до C# 14 нельзя было сделать `string.IsEmpty` как property — только метод. Старый синтаксис `this T` остаётся валидным, оба стиля компилируются в одинаковые статические методы. Extension indexers и constructors в C# 14 не вошли — отложены на будущие версии.

---

## 17. Cheat sheet

```csharp
// Базовая extension
public static class StringExt
{
    public static int WordCount(this string s) => s.Split(' ').Length;
}
"hello world".WordCount();   // 2

// Generic extension
public static class CollectionExt
{
    public static T First<T>(this IEnumerable<T> src) where T : class
        => src.FirstOrDefault() ?? throw new InvalidOperationException();
}

// Mutating struct
public static class PointExt
{
    public static void Move(this ref Point p, int dx, int dy)
    {
        p.X += dx; p.Y += dy;
    }
}

// Read-only ref для большой struct
public static class LargeStructExt
{
    public static int Sum(this in LargeStruct s) => s.A + s.B + ...;
}

// Null-aware
public static class StringExt
{
    public static int LenOrZero(this string? s) => s?.Length ?? 0;
}

// LINQ-style с deferred execution
public static class LinqExt
{
    public static IEnumerable<T> WhereNotNull<T>(this IEnumerable<T?> src) where T : class
    {
        foreach (var item in src)
            if (item is not null) yield return item;
    }
}

// Async с CancellationToken
public static class TaskExt
{
    public static async Task<T> WithTimeout<T>(this Task<T> task, TimeSpan t, CancellationToken ct = default)
    {
        // ...
    }
}

// Fluent API
public static class BuilderExt
{
    public static MyBuilder WithFoo(this MyBuilder b, string foo)
    {
        b.Foo = foo;
        return b;
    }
}

// DI registration
public static class ServiceExt
{
    public static IServiceCollection AddMyFeature(this IServiceCollection s, IConfiguration c)
    {
        s.AddScoped<IMyService, MyService>();
        s.Configure<MyOptions>(c.GetSection("My"));
        return s;
    }
}

// Domain DSL
public static class TimeExt
{
    public static TimeSpan Seconds(this int n) => TimeSpan.FromSeconds(n);
    public static TimeSpan Minutes(this int n) => TimeSpan.FromMinutes(n);
}
await Task.Delay(5.Seconds());

// Adapter
public static class UserMap
{
    public static UserDto ToDto(this User u) => new(u.Id, u.Name);
}
```

| Сценарий | Решение |
|----------|---------|
| Расширить чужой тип | extension method |
| Чистая утилита, не относится к типу | static method (без `this`) |
| Fluent builder | extension с `return this` |
| LINQ-style операция | extension на `IEnumerable<T>` |
| Composite key DI | `AddMyFeature(this IServiceCollection)` |
| Domain DSL | `5.Seconds()`, `100m.USD()` |
| Adapter / converter | `As*`/`To*` extension |
| Mutate struct | `this ref T` (C# 7.3+) |
| Read-only ref для perf | `this in T` (C# 7.2+) |
| Null-aware API | `this T?` + проверка |
| Cancel-aware async | `CancellationToken ct = default` last param |

---

## 18. Practice — упражнения с разбором

### 18.1. Null-safe StringExt

**Задача.** Написать extensions для строки: `IsEmpty`, `IsBlank` (whitespace), `OrEmpty` (replace null with ""), `OrDefault` (replace null/empty with default).

```csharp
public static class StringExt
{
    public static bool IsEmpty([NotNullWhen(false)] this string? s) => string.IsNullOrEmpty(s);

    public static bool IsBlank([NotNullWhen(false)] this string? s) => string.IsNullOrWhiteSpace(s);

    public static string OrEmpty(this string? s) => s ?? "";

    public static string OrDefault(this string? s, string defaultValue)
        => s.IsBlank() ? defaultValue : s;
}

string? name = null;
name.IsEmpty();              // True
name.IsBlank();              // True
name.OrEmpty();              // ""
name.OrDefault("Anonymous"); // "Anonymous"

"   ".IsEmpty();             // False — есть пробелы
"   ".IsBlank();             // True
```

**Разбор:** `[NotNullWhen(false)]` — атрибут для NRT, говорит компилятору «если return false, то параметр точно не null». Это даёт smart flow analysis у caller. `OrEmpty` и `OrDefault` — null-safe defaults.

### 18.2. Domain DSL для длительности

**Задача.** Сделать extensions для int → TimeSpan, чтобы можно было писать `5.Seconds()`, `2.Hours().Plus(30.Minutes())`, `1.Day().Ago()`.

```csharp
public static class TimeExt
{
    public static TimeSpan Milliseconds(this int n) => TimeSpan.FromMilliseconds(n);
    public static TimeSpan Seconds(this int n) => TimeSpan.FromSeconds(n);
    public static TimeSpan Minutes(this int n) => TimeSpan.FromMinutes(n);
    public static TimeSpan Hours(this int n) => TimeSpan.FromHours(n);
    public static TimeSpan Days(this int n) => TimeSpan.FromDays(n);

    public static TimeSpan Plus(this TimeSpan a, TimeSpan b) => a + b;
    public static TimeSpan Minus(this TimeSpan a, TimeSpan b) => a - b;

    public static DateTime Ago(this TimeSpan t) => DateTime.UtcNow - t;
    public static DateTime FromNow(this TimeSpan t) => DateTime.UtcNow + t;
}

var deadline = 2.Hours().Plus(30.Minutes()).FromNow();
var since = 7.Days().Ago();
await Task.Delay(500.Milliseconds());
```

**Разбор:** простые wrappers над `TimeSpan.From*`. Главная ценность — читаемость: `2.Hours().Plus(30.Minutes())` понятнее, чем `new TimeSpan(2, 30, 0)`. В Quartz.NET, Polly, Hangfire — стандартный паттерн.

### 18.3. `Result<T>` functional extensions

**Задача.** Реализовать `Map`, `Bind`, `Match` для своего `Result<T>` типа.

```csharp
public abstract record `Result<T>`;
public sealed record Ok<T>(T Value) : `Result<T>`;
public sealed record Err<T>(string Error) : `Result<T>`;

public static class ResultExt
{
    public static Result<TResult> Map<T, TResult>(this `Result<T>` r, Func<T, TResult> f)
        => r switch
        {
            Ok<T> ok => new Ok<TResult>(f(ok.Value)),
            Err<T> err => new Err<TResult>(err.Error),
            _ => throw new InvalidOperationException()
        };

    public static Result<TResult> Bind<T, TResult>(this `Result<T>` r, Func<T, Result<TResult>> f)
        => r switch
        {
            Ok<T> ok => f(ok.Value),
            Err<T> err => new Err<TResult>(err.Error),
            _ => throw new InvalidOperationException()
        };

    public static TResult Match<T, TResult>(
        this `Result<T>` r,
        Func<T, TResult> onOk,
        Func<string, TResult> onErr)
        => r switch
        {
            Ok<T> ok => onOk(ok.Value),
            Err<T> err => onErr(err.Error),
            _ => throw new InvalidOperationException()
        };
}

// Использование — pipeline без try/catch
Result<int> ParseInt(string s) =>
    int.TryParse(s, out var n) ? new Ok<int>(n) : new Err<int>("Not a number");

Result<int> Double(int n) => new Ok<int>(n * 2);

string DescribeResult(string input) =>
    ParseInt(input)
        .Bind(Double)
        .Match(
            onOk: v => $"Got {v}",
            onErr: e => $"Failed: {e}");

DescribeResult("21");     // "Got 42"
DescribeResult("hello");  // "Failed: Not a number"
```

**Разбор:** functional pipeline через extensions. `Map` — преобразование значения в Ok-случае. `Bind` — chain операций, которые сами могут вернуть Result. `Match` — terminal, deconstruct в результат любого типа. Это альтернатива `try/catch` для recoverable-ошибок.

### 18.4. EnumerableExt для batch обработки

**Задача.** Написать `Batch<T>` extension, разбивающий `IEnumerable<T>` на коллекции по N элементов.

```csharp
public static class EnumerableExt
{
    public static IEnumerable<IReadOnlyList<T>> Batch<T>(this IEnumerable<T> source, int size)
    {
        if (size <= 0) throw new ArgumentOutOfRangeException(nameof(size));

        var batch = new List<T>(size);
        foreach (var item in source)
        {
            batch.Add(item);
            if (batch.Count == size)
            {
                yield return batch;
                batch = new List<T>(size);
            }
        }
        if (batch.Count > 0) yield return batch;
    }
}

var nums = Enumerable.Range(1, 10);
foreach (var batch in nums.Batch(3))
{
    Console.WriteLine(string.Join(", ", batch));
}
// 1, 2, 3
// 4, 5, 6
// 7, 8, 9
// 10
```

**Разбор:** classic паттерн batching для bulk-операций (insert N rows за раз, send K messages в queue). `yield return` даёт lazy evaluation — батчи генерируются по мере итерации, без материализации всей коллекции в память.

В .NET 6+ есть встроенный `Chunk(int)` — то же самое. Но приведено для понимания механики.

### 18.5. Builder через extensions

**Задача.** Реализовать HttpRequestBuilder с methods URL, Method, Header, Body, Build, используя extension methods.

```csharp
public class HttpRequestBuilder
{
    public string Url { get; set; } = "";
    public HttpMethod Method { get; set; } = HttpMethod.Get;
    public Dictionary<string, string> Headers { get; } = new();
    public string? JsonBody { get; set; }

    public HttpRequestMessage Build()
    {
        var req = new HttpRequestMessage(Method, Url);
        foreach (var h in Headers) req.Headers.TryAddWithoutValidation(h.Key, h.Value);
        if (JsonBody is not null) req.Content = new StringContent(JsonBody, Encoding.UTF8, "application/json");
        return req;
    }
}

public static class HttpRequestBuilderExt
{
    public static HttpRequestBuilder Url(this HttpRequestBuilder b, string url) { b.Url = url; return b; }
    public static HttpRequestBuilder Method(this HttpRequestBuilder b, HttpMethod m) { b.Method = m; return b; }
    public static HttpRequestBuilder Header(this HttpRequestBuilder b, string key, string value) { b.Headers[key] = value; return b; }
    public static HttpRequestBuilder Bearer(this HttpRequestBuilder b, string token) => b.Header("Authorization", $"Bearer {token}");
    public static HttpRequestBuilder JsonBody<T>(this HttpRequestBuilder b, T data) { b.JsonBody = JsonSerializer.Serialize(data); return b; }
}

// Использование
var request = new HttpRequestBuilder()
    .Url("https://api.example.com/orders")
    .Method(HttpMethod.Post)
    .Bearer("token123")
    .Header("X-Request-Id", Guid.NewGuid().ToString())
    .JsonBody(new { items = new[] { 1, 2, 3 } })
    .Build();

var response = await httpClient.SendAsync(request);
```

**Разбор:** classic builder через extensions. Главные принципы: state в основном классе, методы возвращают this, можно добавлять новые методы без модификации основного класса. `Bearer` показывает композицию — он делегирует в `Header`. Полезно для тестов: setup читается линейно.

---

## 19. Что читать дальше — порядок и почему

1. **[[csharp-basics|C# Basics]]** — generics, static, lambdas — основа для extensions.
2. **[[collections-linq|Collections и LINQ]]** — extensions в production-ситуациях.
3. **[[generics-deep|Generics deep]]** — constraints, variance, type inference подробно.
4. **[[modern-features|Modern Features]]** — pattern matching, records, primary constructors.
5. **[[anonymous-types|Anonymous Types]]** — близкий компонент для LINQ-проекций.
6. **[[delegates-events|Delegates и Events]]** — extensions часто принимают `Func`/`Action`.
7. **Async deep dive** — async extensions, IAsyncEnumerable, CancellationToken.
8. **Roslyn analyzers** — как написать analyzer для своих extensions (детектить misuse).

---

## 20. См. также

- [[csharp-basics|C# Basics]] — основы языка
- [[collections-linq|Collections и LINQ]] — где extensions main use case
- [[generics-deep|Generics deep]] — constraints, type inference
- [[anonymous-types|Anonymous Types]] — LINQ projections
- [[modern-features|Modern Features]] — pattern matching, records
- [[tuples-deconstruction|Tuples]] — связан с extension Deconstruct
- [[delegates-events|Delegates и Events]] — Func/Action для extensions
- [[error-handling|Error Handling]] — `Result<T>` и exception extensions
- Roslyn Analyzers — статический анализ extensions
- Source Generators — генерация extension methods

---

## 21. Reading list

- **Microsoft Docs — Extension methods** — learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/extension-methods
- **Microsoft Docs — Standard Query Operators (LINQ)** — learn.microsoft.com/dotnet/csharp/linq
- **Microsoft Docs — System.Linq.Enumerable** — learn.microsoft.com/dotnet/api/system.linq.enumerable
- **C# Language Specification — Extension Methods** — github.com/dotnet/csharplang/blob/main/spec/classes.md
- **Microsoft Docs — What's new in C# 14 (extension members)** — learn.microsoft.com/dotnet/csharp/whats-new/csharp-14
- **.NET Blog — C# 14: Exploring extension members** — devblogs.microsoft.com/dotnet/csharp-exploring-extension-members
- **Eric Lippert — Extension methods design notes** — ericlippert.com (поиск «extension methods»)
- **Stephen Toub — Performance of extension methods** — devblogs.microsoft.com/dotnet
- **Mark Seemann — Builder pattern в .NET** — blog.ploeh.dk
- **Vladimir Khorikov — DSL with extensions** — enterprisecraftsmanship.com
- **Andrew Lock — DI extensions patterns** — andrewlock.net
- **Jon Skeet — C# in Depth (4th ed.)** — chapter «Extension methods»
- **Bill Wagner — Effective C# (3rd ed.)** — items по extensions
- **Kotlin Extensions Documentation** — kotlinlang.org/docs/extensions.html (для сравнения design)
- **SharpLab** — sharplab.io — посмотреть IL для extension methods

---

## 22. Bonus practice — расширенные упражнения

### 22.1. Composable validation

**Задача.** Сделать chain-style валидатор: `value.NotNull(name).NotEmpty(name).MaxLength(50, name)`.

```csharp
public static class ValidationExt
{
    public static T NotNull<T>(this T? value, string paramName) where T : class
        => value ?? throw new ArgumentNullException(paramName);

    public static string NotEmpty(this string value, string paramName)
    {
        if (string.IsNullOrWhiteSpace(value))
            throw new ArgumentException("must not be empty", paramName);
        return value;
    }

    public static string MaxLength(this string value, int max, string paramName)
    {
        if (value.Length > max)
            throw new ArgumentException($"must be at most {max} chars", paramName);
        return value;
    }

    public static string MinLength(this string value, int min, string paramName)
    {
        if (value.Length < min)
            throw new ArgumentException($"must be at least {min} chars", paramName);
        return value;
    }

    public static int InRange(this int value, int min, int max, string paramName)
    {
        if (value < min || value > max)
            throw new ArgumentOutOfRangeException(paramName, $"must be between {min} and {max}");
        return value;
    }
}

public class CreateOrderCommand
{
    public string CustomerName { get; set; } = "";
    public int Quantity { get; set; }

    public void Validate()
    {
        CustomerName.NotNull(nameof(CustomerName))
                    .NotEmpty(nameof(CustomerName))
                    .MaxLength(100, nameof(CustomerName));
        Quantity.InRange(1, 1000, nameof(Quantity));
    }
}
```

**Разбор:** каждый extension возвращает значение для chain. Imperative style, но читается как декларация требований. Альтернатива — FluentValidation library (более мощный, но лишняя зависимость).

### 22.2. AsyncEnumerable batch processor

**Задача.** Обрабатывать `IAsyncEnumerable<T>` батчами по N с параллельной обработкой каждого батча.

```csharp
public static class AsyncEnumerableExt
{
    public static async IAsyncEnumerable<TResult> BatchProcess<T, TResult>(
        this IAsyncEnumerable<T> source,
        int batchSize,
        Func<T, Task<TResult>> processor,
        [EnumeratorCancellation] CancellationToken ct = default)
    {
        var batch = new List<T>(batchSize);
        await foreach (var item in source.WithCancellation(ct))
        {
            batch.Add(item);
            if (batch.Count == batchSize)
            {
                var results = await Task.WhenAll(batch.Select(processor));
                foreach (var r in results) yield return r;
                batch.Clear();
            }
        }
        if (batch.Count > 0)
        {
            var results = await Task.WhenAll(batch.Select(processor));
            foreach (var r in results) yield return r;
        }
    }
}

// Использование
var stream = GetAsyncStream();
await foreach (var result in stream.BatchProcess(
    batchSize: 10,
    processor: item => ProcessAsync(item),
    ct: cancellationToken))
{
    Console.WriteLine(result);
}
```

**Разбор:** combo `IAsyncEnumerable` + parallel processing внутри батча. `[EnumeratorCancellation]` позволяет `WithCancellation(ct)` пробросить токен. `Task.WhenAll` параллелит, `yield return` стримит результаты по мере готовности батча.

---

## 23. Связь с Source Generators и анализаторами

### 23.1. Source Generators как альтернатива extensions

С .NET 5+ есть **Source Generators** — компилятор-плагины, которые генерируют код во время компиляции. Иногда они решают те же задачи, что extensions, но мощнее.

Пример: System.Text.Json source generator генерирует typed serialization-код, чтобы в runtime не использовалась reflection:

```csharp
[JsonSerializable(typeof(User))]
public partial class UserJsonContext : JsonSerializerContext { }

string json = JsonSerializer.Serialize(user, UserJsonContext.Default.User);
```

Здесь `Default.User` — сгенерированный typed accessor. Это **не** extension, это сгенерированный код. Преимущества:
- Полная типобезопасность, работает с Native AOT.
- Нет reflection в runtime.
- Compiler знает о генерации, IntelliSense работает.

Когда что выбирать:
- **Extensions** — runtime-логика, полиморфизм через generics, DI-friendly.
- **Source Generators** — статическая генерация кода, perf-critical, AOT-friendly.

### 23.2. Roslyn Analyzers для extensions

Можно написать **analyzer**, который проверяет правильное использование твоих extensions:

```csharp
// Пример: warning, если ToList() используется без Where/Select после
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class UnnecessaryToListAnalyzer : DiagnosticAnalyzer
{
    private static readonly DiagnosticDescriptor Rule = new(
        "MYC001",
        "Unnecessary ToList()",
        "ToList() without subsequent operations is redundant",
        "Performance",
        DiagnosticSeverity.Warning,
        isEnabledByDefault: true);

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => [Rule];

    public override void Initialize(AnalysisContext ctx)
    {
        ctx.EnableConcurrentExecution();
        ctx.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        ctx.RegisterSyntaxNodeAction(AnalyzeInvocation, SyntaxKind.InvocationExpression);
    }

    private void AnalyzeInvocation(SyntaxNodeAnalysisContext ctx)
    {
        // ... детекция и emit диагностики
    }
}
```

Analyzers пакетируются в NuGet (с типом `analyzers/dotnet/cs/`). Когда консумер устанавливает пакет, IDE автоматически видит warnings.

Это часто используется библиотечными авторами для guide consumers по правильному использованию API.

### 23.3. Extensions + analyzers — DSL с проверкой

Связка особенно мощна в DSL-сценариях. Пример: FluentAssertions.

```csharp
result.Should().Be(42);   // extension
result.Should().BeGreaterThan(0).And.BeLessThan(100);
```

Если consumer пишет `result.Should().Be(42, because: "...")` без `because: "..."` — analyzer может предупредить, что лучше указать reason для отладки.

### 23.4. Extensions для analyzers самих

Roslyn API сам обильно использует extensions:

```csharp
// SyntaxNode extensions
node.AncestorsAndSelf();
node.DescendantNodes().OfType<MethodDeclarationSyntax>();
node.WithLeadingTrivia(...);

// SymbolInfo extensions
symbol.GetMembers().OfType<IMethodSymbol>();
```

Если пишешь analyzer / source generator — будешь активно с ними взаимодействовать.

> [!question]- Интервью: что использовать для расширения функциональности — extensions, source generators или analyzers?
> Все три решают разные задачи: 1) **Extensions** — runtime-логика, синтаксический сахар, fluent API, DSL, LINQ-style операции. Гибко, но всё в runtime. 2) **Source Generators** — статически сгенерированный код во время компиляции, без reflection, AOT-friendly, performance-критичный (System.Text.Json, Mapster, MediatR). 3) **Analyzers** — статический анализ кода и emit warnings/errors. Не генерирует код, направляет consumer'а к правильному использованию API. Часто комбинируется: extensions API + analyzer для проверки правильного использования + source generator для AOT-friendly реализации.

---
