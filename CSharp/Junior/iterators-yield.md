---
tags: [csharp, iterators, yield, junior, ienumerable, lazy-evaluation, async-streams]
level: Junior
date: 2026-08-02
---

# Iterators и yield — итераторы и ленивая генерация

> **Метод, который возвращает по одному элементу за раз, а не материализует всю коллекцию.** `yield return`, `IEnumerable`, `IEnumerator`, state machine, lazy evaluation, `IAsyncEnumerable` (.NET Core 3.0+ / C# 8). Закрывает пробел: «знаю про `foreach`, не понимаю, как написать свой собственный source данных».

---

## 0. Как читать этот файл

Если ты впервые видишь `yield return` — читай разделы 1→4: получишь рабочую модель и поймёшь, **почему `yield` экономит память**. Если уже пишешь iterator-методы, но непонятно «как они компилируются» — раздел 3 (state machine), 7 (deferred execution). Если строишь production систему — раздел 11 (async streams), 13 (производительность), 14 (best practices).

Все примеры самостоятельные. `// expected: ...` показывает ожидаемый вывод. Cross-language якоря (`> [!info]-`) свёрнуты — раскрывай, если переходишь из Python / JavaScript / Rust / Kotlin / Java. Interview-вопросы (`> [!question]-`) встроены рядом с теорией.

---

## 1. Что это, зачем и когда

### 1.1. Что такое iterator

**Iterator** — это метод, который возвращает последовательность значений **по одному**, не строя полную коллекцию в памяти. Объявляется как обычный метод с `IEnumerable<T>` / `IEnumerator<T>` возвращаемым типом, но внутри использует ключевое слово `yield return`:

```csharp
public IEnumerable<int> CountTo(int n)
{
    for (int i = 1; i <= n; i++)
        yield return i;
}

foreach (var x in CountTo(5))
    Console.WriteLine(x);
// 1, 2, 3, 4, 5
```

Под капотом C#-компилятор превращает метод в **state machine** (класс), который реализует `IEnumerable<T>` и `IEnumerator<T>`. Каждый `yield return` сохраняет state и возвращает значение caller-у. На следующий `MoveNext()` выполнение продолжается с того же места.

### 1.2. Зачем yield, когда есть `List<T>`

Без yield:

```csharp
public List<int> CountTo(int n)
{
    var list = new List<int>(n);
    for (int i = 1; i <= n; i++)
        list.Add(i);
    return list;
}

foreach (var x in CountTo(1_000_000))
    Process(x);
```

Проблемы:

1. **Материализация всех элементов** — для `n = 1_000_000` создаётся `List` на 1M элементов, ~4MB heap.
2. **Eager evaluation** — даже если caller хочет только первые 10, посчитаются все 1M.
3. **Side effects** — если генерация дорогая (HTTP, DB), все запросы случатся, даже если их не используют.

С yield:

```csharp
public IEnumerable<int> CountTo(int n)
{
    for (int i = 1; i <= n; i++)
        yield return i;
}

foreach (var x in CountTo(1_000_000).Take(10))
    Process(x);   // только 10 итераций!
```

- **Lazy** — элементы генерируются по запросу.
- **Memory-efficient** — в памяти только текущий элемент + state machine.
- **Composable** — `Take`, `Where`, `Select` делают только то, что нужно.

### 1.3. Главное правило

```
Используй yield когда:
- Источник большой или неизвестного размера
- Caller может прервать на середине (Take, First, FirstOrDefault)
- Генерация элемента дорогая (don't compute what's not needed)
- Бесконечная последовательность

НЕ используй когда:
- Маленький известный набор (просто верни List)
- Нужен индексный доступ или Count (yield не даёт)
- Caller итерирует много раз — каждая итерация будет re-execute
```

### 1.4. yield return vs yield break

```csharp
public IEnumerable<int> Numbers()
{
    yield return 1;
    yield return 2;
    yield break;     // прервать iteration
    yield return 3;  // не достигается
}

foreach (var n in Numbers())
    Console.WriteLine(n);
// 1, 2 — Output: 3 не выводится
```

`yield return X` — отдать значение и приостановить.
`yield break` — закончить итерацию (как `return` в обычном методе).

### 1.5. Эволюция: C# 2.0 → C# 13

| Версия | Год | Что появилось |
|--------|-----|---------------|
| **C# 2.0** | 2005 | `yield return`, `yield break` — первое появление iterators |
| **C# 5.0** | 2012 | `async/await` — но **не** для iterators ещё |
| **C# 7.0** | 2017 | Local functions могут быть iterators |
| **C# 8.0** | 2019 | `IAsyncEnumerable<T>`, `await foreach`, `yield` в async |
| **.NET 5+** | 2020 | Performance optimizations для iterators |
| **C# 11+** | 2022 | Pattern matching улучшения для итераторов |
| **C# 13** | 2024 | `params ReadOnlySpan<T>` — связано через перегрузки |

### 1.6. Когда что использовать

```
Малый известный набор (5-50 элементов)
  → return new List<T> { ... }
  
Большой набор / lazy generation
  → `IEnumerable<T>` с yield return
  
Бесконечная / unbounded последовательность
  → `IEnumerable<T>` с yield return + caller использует Take
  
Async source (HTTP stream, DB cursor, file lines)
  → `IAsyncEnumerable<T>` с async + yield (.NET Core 3.0+ / C# 8)
  
Stream обработка с pipeline
  → `IEnumerable<T>` + LINQ (Where/Select/Aggregate)
```

> [!info]- Если ты знаешь Python / JavaScript / Rust / Kotlin / Java
> **Python:** `yield` идентичен. Generator-функции — точно так же state-machine, lazy evaluation. C# `yield return` ↔ Python `yield`. Async generators (Python 3.6+) ↔ C# `IAsyncEnumerable`.
>
> **JavaScript:** `function*` (generator function) с `yield`. Семантика такая же — state machine, lazy. Async generators (`async function*` + `for await`) ↔ C# `IAsyncEnumerable`.
>
> **Rust:** нет `yield` keyword (предложен в pre-design). Альтернатива — реализовать `Iterator` trait вручную с `next()` методом, или использовать `iter::from_fn`. Stable iterators через trait — мощно, но более verbose.
>
> **Kotlin:** `sequence { yield(...) }` — функциональный builder, эмулирует C#-style yield. Coroutines + Flow для async cases.
>
> **Java:** до Java 8 — итераторы вручную через `Iterator` interface. Java 8+ — Stream API (eager или lazy в зависимости от operations). C# `IEnumerable` ближе к Java `Stream` по идее, но Stream нельзя iterate несколько раз.

> [!question]- Интервью: чем yield лучше возврата `List<T>`?
> 1) **Lazy evaluation** — элементы вычисляются по запросу, не все сразу. 2) **Memory-efficient** — в памяти только текущий элемент + state, не вся коллекция. 3) **Composability** — caller может комбинировать с `Take`, `Where`, `First` без вычисления лишнего. 4) **Бесконечные последовательности** — невозможны через `List`. 5) **Side effects** генерации (HTTP, DB) случаются только для нужных элементов. Минусы: caller не знает заранее `Count`, нет индексного доступа, повторная итерация = повторное выполнение.

---

## 2. Базовый синтаксис

### 2.1. Простой iterator

```csharp
public IEnumerable<int> Range(int start, int count)
{
    for (int i = 0; i < count; i++)
        yield return start + i;
}

foreach (var x in Range(10, 5))
    Console.WriteLine(x);
// 10, 11, 12, 13, 14
```

Метод **должен** возвращать `IEnumerable<T>` или `IEnumerator<T>` (или их generic варианты). Без этого `yield` не компилируется.

### 2.2. Возможные return types

```csharp
public IEnumerable<int> Method1() { yield return 1; }    // generic, идиоматично
public IEnumerator<int> Method2() { yield return 1; }    // тоже OK, реже
public IEnumerable Method3() { yield return 1; }          // non-generic, legacy
public IEnumerator Method4() { yield return 1; }          // non-generic, legacy
```

Idiomatic — `IEnumerable<T>`. `IEnumerator<T>` имеет смысл, когда метод сам реализует enumerator контракт (например, для custom `foreach` без `IEnumerable`).

### 2.3. Несколько yield в методе

```csharp
public IEnumerable<string> Greetings()
{
    yield return "Hello";
    yield return "Hi";
    yield return "Hey";
}

foreach (var g in Greetings())
    Console.WriteLine(g);
// Hello, Hi, Hey
```

Каждый `yield return` — отдельная "пауза". Между ними может быть произвольный код.

### 2.4. Условный yield

```csharp
public IEnumerable<int> Evens(int n)
{
    for (int i = 1; i <= n; i++)
    {
        if (i % 2 == 0)
            yield return i;
    }
}

foreach (var x in Evens(10))
    Console.WriteLine(x);
// 2, 4, 6, 8, 10
```

`yield` может быть в `if`, `for`, `while`, `switch`. Ограничение — не в `try` с catch (детали в разделе 5).

### 2.5. Бесконечная последовательность

```csharp
public IEnumerable<int> NaturalNumbers()
{
    int i = 1;
    while (true)
        yield return i++;
}

// Caller использует Take для ограничения
foreach (var x in NaturalNumbers().Take(5))
    Console.WriteLine(x);
// 1, 2, 3, 4, 5
```

Iterator может быть бесконечным — `while (true)` нормально. `List<int>` так не сможет.

### 2.6. Вызов другого iterator-метода

```csharp
public IEnumerable<int> First10Evens()
{
    foreach (var n in NaturalNumbers())
    {
        if (n % 2 == 0)
            yield return n;
        if (n >= 20) yield break;   // защита
    }
}
```

Iterator может потреблять другой iterator. State machine оборачивает оба корректно.

### 2.7. Локальная функция как iterator

```csharp
public IEnumerable<int> EvensInRange(int from, int to)
{
    return EvensIterator();

    IEnumerable<int> EvensIterator()
    {
        for (int i = from; i <= to; i++)
            if (i % 2 == 0)
                yield return i;
    }
}
```

Это паттерн **eager validation + lazy execution** — внешний метод проверяет аргументы, локальная функция выполняет ленивую часть. Подробнее в разделе 7.

> [!question]- Интервью: что должен возвращать метод с yield?
> `IEnumerable`, `IEnumerable<T>`, `IEnumerator` или `IEnumerator<T>`. Без одного из них компилятор не позволит использовать `yield`. Идиоматично — `IEnumerable<T>` (generic). `IEnumerator<T>` — реже, для случаев, когда метод сам выступает enumerator'ом. С .NET Core 3.0 / C# 8 (2019) для async iterators — `IAsyncEnumerable<T>` или `IAsyncEnumerator<T>`.

---

## 3. Внутреннее устройство — state machine

### 3.1. Что генерирует компилятор

Когда ты пишешь:

```csharp
public IEnumerable<int> CountTo(int n)
{
    for (int i = 1; i <= n; i++)
        yield return i;
}
```

Компилятор превращает это в **класс** примерно такого вида (упрощённо):

```csharp
[CompilerGenerated]
private sealed class <CountTo>d__0 : IEnumerable<int>, IEnumerator<int>, IDisposable
{
    private int <>1__state;       // текущее состояние
    private int <>2__current;     // текущее значение
    private int n;                  // параметр
    private int <i>5__1;           // local i

    public int Current => <>2__current;

    public bool MoveNext()
    {
        switch (<>1__state)
        {
            case 0:
                <>1__state = -1;
                <i>5__1 = 1;
                goto loopCheck;
            case 1:
                <>1__state = -1;
                <i>5__1++;
                goto loopCheck;
            loopCheck:
                if (<i>5__1 <= n)
                {
                    <>2__current = <i>5__1;
                    <>1__state = 1;
                    return true;
                }
                return false;
        }
        return false;
    }

    public IEnumerator<int> GetEnumerator() { /* ... */ }
    public void Dispose() { <>1__state = -1; }
    public void Reset() => throw new NotSupportedException();
    // ...
}
```

Компилятор:
1. Генерирует класс `<CountTo>d__0` (имя машины — `<MethodName>d__N`).
2. Реализует `IEnumerable<T>`, `IEnumerator<T>`, `IDisposable`.
3. Локальные переменные становятся полями класса.
4. Тело метода превращается в `switch` по состоянию.
5. Каждый `yield return` — отдельный case + сохранение state + return true.

### 3.2. Что значит "пауза" между yield

`MoveNext()` — единственный метод, который двигает iterator. Когда `foreach` вызывает `MoveNext()`:
1. State machine заходит в `switch` по текущему `<>1__state`.
2. Выполняет код до следующего `yield return`.
3. Сохраняет значение в `<>2__current`.
4. Сохраняет следующее состояние в `<>1__state`.
5. Возвращает `true`.

При следующем `MoveNext()` всё повторяется с правильного места.

`yield break` — устанавливает `<>1__state = -1` и возвращает `false`. `foreach` останавливается.

### 3.3. Один instance до GetEnumerator

Тонкий момент: один и тот же класс реализует **и** `IEnumerable<T>`, **и** `IEnumerator<T>`. Это оптимизация — не создавать два объекта.

```csharp
var iter = CountTo(5);   // создан класс, но MoveNext не вызван
var en1 = iter.GetEnumerator();   // вернёт this (если еще не enumerator)
var en2 = iter.GetEnumerator();   // вернёт новый instance!
```

Первый `GetEnumerator()` возвращает сам класс. Второй — новый instance с теми же параметрами. Это значит, что **два независимых foreach** на одной IEnumerable работают независимо.

### 3.4. Как foreach вызывает iterator

```csharp
foreach (var x in CountTo(5))
    Console.WriteLine(x);
```

Компилятор разворачивает в:

```csharp
var enumerable = CountTo(5);
var enumerator = enumerable.GetEnumerator();
try
{
    while (enumerator.MoveNext())
    {
        var x = enumerator.Current;
        Console.WriteLine(x);
    }
}
finally
{
    enumerator?.Dispose();
}
```

`Dispose` обязательно вызывается через `try/finally`. Это важно — iterator может держать ресурсы (file handle, DB cursor), которые надо освободить.

### 3.5. Performance overhead state machine

Каждый iterator-метод = один heap-аллоцированный класс. Это:
- ~40-80 байт на instance.
- Allocation на каждый вызов метода.
- GC pressure при частом вызове в hot loop.

В .NET 5+ есть оптимизации (struct enumerators для некоторых cases), но обычные iterator-методы всегда class-based. Если perf критичен — можно реализовать `IEnumerable<T>` вручную через struct (раздел 6).

### 3.6. Замыкания на параметры

Параметры iterator-метода **захватываются** в state machine — становятся полями класса. Это значит:

```csharp
public IEnumerable<int> Range(int start, int count)
{
    for (int i = 0; i < count; i++)
        yield return start + i;
}

var iter = Range(10, 5);
// start=10, count=5 захвачены в state machine
// Если caller теперь меняет свой local 'count', это не влияет
```

### 3.7. Посмотреть IL — sharplab.io

Чтобы увидеть, что реально генерируется — открой [sharplab.io](https://sharplab.io), вставь код, выбери "C#" в Decompile. Увидишь сгенерированный класс state machine. Полезно для понимания механики.

> [!question]- Интервью: что генерирует компилятор для метода с yield?
> Класс `<MethodName>d__N` с `[CompilerGenerated]` атрибутом, реализующий `IEnumerable<T>`, `IEnumerator<T>`, `IDisposable`. Локальные переменные → поля класса. Тело метода → switch по `<>1__state` в `MoveNext()`. Каждый `yield return` — отдельный case: сохраняет значение в `<>2__current`, сохраняет следующее состояние, возвращает true. `yield break` — state = -1, return false. Каждый вызов iterator-метода создаёт новый instance этого класса. Один объект реализует и Enumerable, и Enumerator (оптимизация).

---

## 4. Lazy evaluation deep

### 4.1. Iterator не выполняется до foreach

```csharp
public IEnumerable<int> WithLogging()
{
    Console.WriteLine("Iterator method body started");
    for (int i = 1; i <= 3; i++)
    {
        Console.WriteLine($"  Yielding {i}");
        yield return i;
    }
    Console.WriteLine("Iterator method body ended");
}

Console.WriteLine("=== Calling iterator ===");
var iter = WithLogging();   // ← НИЧЕГО не печатается!
Console.WriteLine("=== Iterator obtained ===");

foreach (var x in iter)      // ← теперь печатается
    Console.WriteLine($"Got {x}");
```

Output:
```
=== Calling iterator ===
=== Iterator obtained ===
Iterator method body started
  Yielding 1
Got 1
  Yielding 2
Got 2
  Yielding 3
Got 3
Iterator method body ended
```

Тело iterator-метода **не выполняется** при вызове `WithLogging()`. Только при первой `MoveNext()` (которую `foreach` делает в начале).

### 4.2. Side effects тоже отложены

Это критично для side effects:

```csharp
public IEnumerable<User> LoadUsers()
{
    Console.WriteLine("Connecting to DB");
    using var conn = new SqlConnection(connStr);
    conn.Open();   // ← происходит при первой MoveNext, не при вызове LoadUsers()

    using var cmd = new SqlCommand("SELECT * FROM Users", conn);
    using var reader = cmd.ExecuteReader();
    while (reader.Read())
    {
        yield return new User { /* ... */ };
    }
}

var iter = LoadUsers();   // соединение НЕ открыто
// ... позже
foreach (var u in iter) { ... }   // только сейчас открывается
```

Это иногда сюрприз — если caller думает, что вызов = выполнение, он не ловит exception на месте.

### 4.3. Deferred execution в LINQ

Такая же семантика у LINQ:

```csharp
var query = users.Where(u => u.IsActive);   // ничего не вычислено!
// ...
foreach (var u in query) { ... }   // вычисляется здесь
```

`Where`, `Select`, `Take`, `OrderBy` — все iterator-методы под капотом, все deferred.

Терминальные операторы (`ToList`, `ToArray`, `Count`, `First`, `Sum`, etc.) — eager, форсируют выполнение.

### 4.4. Eager validation pattern

Defer execution создаёт проблему: `ArgumentException` из тела iterator-метода **не поднимется** при вызове, только при первой `MoveNext`:

```csharp
public IEnumerable<int> Range(int from, int to)
{
    if (from > to)
        throw new ArgumentException("from > to");   // ← throws при MoveNext, не при вызове!
    for (int i = from; i <= to; i++)
        yield return i;
}

var iter = Range(10, 5);   // OK — НЕ throws!
foreach (var x in iter) { ... }   // throws здесь
```

Это **запутывает** caller-а. Правильный паттерн — eager validation + lazy execution через локальную функцию:

```csharp
public IEnumerable<int> Range(int from, int to)
{
    if (from > to)
        throw new ArgumentException("from > to");   // ← throws сразу
    return RangeIterator(from, to);

    static IEnumerable<int> RangeIterator(int from, int to)
    {
        for (int i = from; i <= to; i++)
            yield return i;
    }
}

var iter = Range(10, 5);   // throws сразу!
```

Внешний метод обычный — eager. Локальная функция — iterator (lazy). Validation срабатывает на вызове, генерация — на foreach.

### 4.5. Защита от double-iteration

Iterator не имеет встроенного state «уже итерирован»:

```csharp
var iter = CountTo(5);

// Первая итерация
foreach (var x in iter) Console.WriteLine(x);  // 1, 2, 3, 4, 5

// Вторая — снова работает! Создаётся новый enumerator
foreach (var x in iter) Console.WriteLine(x);  // 1, 2, 3, 4, 5 (повторно)
```

Каждый `foreach` вызывает `GetEnumerator()` — получает новый instance, тело метода выполняется заново. Side effects тоже повторяются:

```csharp
public IEnumerable<int> WithSideEffect()
{
    Console.WriteLine("Computing!");
    yield return 1;
    yield return 2;
}

var iter = WithSideEffect();
foreach (var _ in iter) { }  // "Computing!" — раз
foreach (var _ in iter) { }  // "Computing!" — два!
```

Если хочешь однократного вычисления — материализуй:

```csharp
var cached = WithSideEffect().ToList();
foreach (var _ in cached) { }   // без побочных эффектов
foreach (var _ in cached) { }   // тоже
```

### 4.6. Lazy chains — pipeline без промежуточных коллекций

Гениальная идея LINQ — chain из iterator-методов **не материализует** промежуточные результаты:

```csharp
var result = Enumerable.Range(1, 1_000_000)
    .Where(x => x % 2 == 0)
    .Select(x => x * x)
    .Take(10)
    .ToList();
```

Что происходит на каждый элемент:
1. `Range` yield-ит число 1.
2. `Where` проверяет `1 % 2 == 0` — false, не передаёт дальше.
3. `Range` yield-ит 2.
4. `Where` пропускает 2.
5. `Select` возвращает 4.
6. `Take(10)` запоминает: 1 из 10.
7. ... продолжается до Take(10) hit лимит.
8. `ToList` собирает 10 элементов.

Total iterations: ~20 (не миллион). Никаких промежуточных `List` на 500K чётных.

> [!question]- Интервью: почему iterator не выполняется до foreach?
> Тело iterator-метода компилируется в `MoveNext()` метод state machine. Когда caller вызывает iterator-метод, он получает только instance state machine, тело не выполнено. Первый `MoveNext()` (который `foreach` делает) запускает выполнение до первого `yield return`. Это deferred execution. Side effects (DB connection, validation) тоже отложены. Чтобы validation срабатывала сразу — wrap iterator в обычный метод и используй локальную функцию для тела.

---

## 5. yield в try/catch/finally

### 5.1. Что разрешено

```csharp
public IEnumerable<int> WithFinally()
{
    try
    {
        yield return 1;     // ✅ OK — yield в try с finally
        yield return 2;
    }
    finally
    {
        Console.WriteLine("Cleanup");
    }
}
```

`yield return` в `try` с **только** `finally` — разрешён. Это даёт правильный cleanup: `finally` выполняется при `Dispose()` enumerator-а (включая случай прерванного foreach).

### 5.2. Что запрещено

```csharp
public IEnumerable<int> WithCatch()
{
    try
    {
        yield return 1;     // ❌ CS1626: cannot yield в try с catch
    }
    catch (Exception)
    {
        // ...
    }
}
```

`yield return` в `try` с **`catch`** — **запрещён**. Причина: state machine не может правильно сериализовать состояние через границу catch.

```csharp
public IEnumerable<int> WithFinallyOnly()
{
    yield return 1;        // вне try

    try
    {
        var x = ProcessSomething();   // не yield — OK
        yield return x;     // ❌ если бы был catch — запрещено
    }
    catch (Exception ex)
    {
        // catch без yield внутри — OK, но yield в этом try — нет
        Log(ex);
    }
}
```

### 5.3. Workaround — eager catch

Если нужен exception handling вокруг генерации — лови ДО yield:

```csharp
public IEnumerable<Item> LoadItems()
{
    foreach (var id in ids)
    {
        Item? item = null;
        try
        {
            item = LoadFromDb(id);   // try НЕ содержит yield
        }
        catch (DbException ex)
        {
            Log(ex);
            continue;   // skip этот ID
        }

        if (item is not null)
            yield return item;   // yield ВНЕ try
    }
}
```

Pattern: try выполняет загрузку → результат в локалку → yield снаружи.

### 5.4. Finally вызывается при Dispose

```csharp
public IEnumerable<int> WithCleanup()
{
    try
    {
        for (int i = 1; i <= 100; i++)
            yield return i;
    }
    finally
    {
        Console.WriteLine("Cleanup!");
    }
}

foreach (var x in WithCleanup().Take(3))
    Console.WriteLine(x);
// 1
// 2
// 3
// Cleanup!  ← Take(3) разорвал foreach, Dispose вызвал finally
```

Это критично для ресурсов:

```csharp
public IEnumerable<string> ReadLines(string path)
{
    using var reader = new StreamReader(path);
    string? line;
    while ((line = reader.ReadLine()) is not null)
        yield return line;
    // Dispose StreamReader при завершении или прерывании foreach
}
```

`using` внутри iterator работает правильно — file закрывается даже если caller break-ит на середине.

### 5.5. Multiple yield в try/finally

```csharp
public IEnumerable<int> Multi()
{
    try
    {
        yield return 1;
        yield return 2;
        yield return 3;
    }
    finally
    {
        Console.WriteLine("Done");
    }
}
```

OK — все три yield в одном try с finally. Cleanup вызывается один раз при завершении.

### 5.6. Nested try

```csharp
public IEnumerable<int> Nested()
{
    try
    {
        yield return 1;
        try
        {
            yield return 2;
        }
        finally
        {
            Console.WriteLine("Inner cleanup");
        }
    }
    finally
    {
        Console.WriteLine("Outer cleanup");
    }
}
```

OK — nested `try/finally` с yield. Cleanup в правильном порядке (LIFO).

> [!question]- Интервью: почему yield запрещён в try с catch?
> State machine компилятора не умеет правильно сериализовать состояние через границу catch — слишком сложная семантика для control flow в `MoveNext()`. Yield в try с **только** finally — разрешён, finally выполняется через `Dispose()` enumerator-а (даже при прерывании foreach). Workaround для catch: try выполняет работу (без yield внутри), результат в локалку, yield снаружи try.

---

## 6. Custom IEnumerable без yield

### 6.1. Зачем

99% случаев — `yield` достаточно. Но иногда нужен ручной контроль:

- Структурный enumerator (`struct`) для perf — без heap allocation на каждом GetEnumerator.
- Особый Reset() поведение.
- Custom GetEnumerator с дополнительными методами (Skip, JumpTo).

### 6.2. Минимальная реализация

```csharp
public class MyRange : IEnumerable<int>
{
    private readonly int _start;
    private readonly int _count;

    public MyRange(int start, int count) { _start = start; _count = count; }

    public IEnumerator<int> GetEnumerator() => new Enumerator(_start, _count);
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator() => GetEnumerator();

    private class Enumerator : IEnumerator<int>
    {
        private readonly int _start;
        private readonly int _count;
        private int _index = -1;

        public Enumerator(int start, int count) { _start = start; _count = count; }

        public int Current => _start + _index;
        object System.Collections.IEnumerator.Current => Current;

        public bool MoveNext()
        {
            _index++;
            return _index < _count;
        }

        public void Reset() => _index = -1;
        public void Dispose() { }
    }
}

foreach (var x in new MyRange(10, 5))
    Console.WriteLine(x);
// 10, 11, 12, 13, 14
```

Эквивалент iterator-метода, но **полный контроль** над struct/class, fields, lifetime.

### 6.3. Struct enumerator — для perf

Heap-allocation enumerator-а на каждом foreach — проблема в hot path. Решение — struct enumerator:

```csharp
public readonly struct FastRange
{
    private readonly int _start;
    private readonly int _count;

    public FastRange(int start, int count) { _start = start; _count = count; }

    public Enumerator GetEnumerator() => new(_start, _count);

    public struct Enumerator
    {
        private readonly int _start;
        private readonly int _count;
        private int _index;

        public Enumerator(int start, int count)
        {
            _start = start;
            _count = count;
            _index = -1;
        }

        public int Current => _start + _index;

        public bool MoveNext()
        {
            _index++;
            return _index < _count;
        }
    }
}

foreach (var x in new FastRange(10, 5))   // struct, без allocation
    Console.WriteLine(x);
```

Заметь: `FastRange` **не** реализует `IEnumerable<T>`. C# foreach pattern-based — компилятор просто ищет метод с именем `GetEnumerator()`, который возвращает struct/class с `MoveNext()` и `Current`. Boxing избегается.

Это паттерн `List<T>.Enumerator`, `Dictionary<K,V>.Enumerator` в BCL.

### 6.4. Когда struct enumerator оправдан

✅ Hot loop в performance-критичном коде (миллионы операций/сек).
✅ Кастомная коллекция в библиотеке — public API.
❌ Обычные сценарии — yield проще и читаемее.

В .NET 5+ компилятор иногда оптимизирует iterator state machine в struct, если возможно. Но не всегда.

### 6.5. Yield vs ручная реализация — сравнение

| | yield | Ручная реализация |
|---|-------|-------------------|
| Простота | ✅ простой синтаксис | ❌ много boilerplate |
| Heap allocation | ❌ class всегда | ✅ можно struct |
| State management | ✅ автоматически | ❌ вручную |
| Try/catch flexibility | ❌ catch запрещён | ✅ полная свобода |
| Reset() | ❌ throws | ✅ можно реализовать |
| Custom methods | ❌ нет | ✅ Skip, Jump, etc. |

### 6.6. IEnumerator вручную для async-like flow

```csharp
public class FibonacciEnumerator : IEnumerator<long>
{
    private long _prev = 0, _curr = 1;
    private int _step = 0;

    public long Current => _prev;

    public bool MoveNext()
    {
        if (_step++ > 0)
        {
            (_prev, _curr) = (_curr, _prev + _curr);
        }
        return _prev < long.MaxValue / 2;
    }

    public void Reset() { _prev = 0; _curr = 1; _step = 0; }
    public void Dispose() { }
    object System.Collections.IEnumerator.Current => Current;
}

public class Fibonacci : IEnumerable<long>
{
    public IEnumerator<long> GetEnumerator() => new FibonacciEnumerator();
    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator() => GetEnumerator();
}

foreach (var n in new Fibonacci().Take(10))
    Console.WriteLine(n);
// 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
```

Иногда ручная реализация выразительнее — state простой, нет вложенных `for` или sub-iterators.

> [!question]- Интервью: когда стоит реализовывать `IEnumerable<T>` вручную вместо yield?
> 1) Когда нужен struct enumerator для избежания heap allocation в hot path. 2) Когда нужен Reset() (yield throws NotSupportedException). 3) Когда нужны custom methods на enumerator (Skip, JumpTo). 4) Когда yield ограничения мешают (yield в try/catch). В большинстве случаев yield проще и читаемее, ручная реализация — оптимизация для библиотек или критичного кода. Pattern-based foreach даёт unboxed struct enumerators (как List/Dictionary).

---

## 7. Eager validation + lazy execution

### 7.1. Проблема — defer execution скрывает ошибки

```csharp
public IEnumerable<int> Take<T>(IEnumerable<int> source, int count)
{
    if (count < 0)
        throw new ArgumentOutOfRangeException(nameof(count));

    foreach (var x in source)
    {
        if (count-- > 0)
            yield return x;
        else
            yield break;
    }
}

// Caller
var iter = Take(numbers, -1);   // НЕ throws здесь!
foreach (var x in iter) { }       // throws здесь — surprise!
```

Validation отложена до первого `MoveNext()`. Caller получает unhelpful stack trace.

### 7.2. Решение — wrapper + local function

```csharp
public IEnumerable<int> Take<T>(IEnumerable<int> source, int count)
{
    if (count < 0)
        throw new ArgumentOutOfRangeException(nameof(count));   // ← throws сразу

    return TakeIterator(source, count);

    static IEnumerable<int> TakeIterator(IEnumerable<int> source, int count)
    {
        foreach (var x in source)
        {
            if (count-- > 0)
                yield return x;
            else
                yield break;
        }
    }
}

// Caller
var iter = Take(numbers, -1);   // throws сразу!
```

Внешний метод `Take` — обычный (не iterator), eager validation. Локальная функция `TakeIterator` — iterator (lazy execution).

`static` на локальной функции (опционально) — гарантирует, что она не захватывает неявно `this` или внешние локалы.

### 7.3. ArgumentNullException + null-aware

```csharp
public `IEnumerable<T>` Where<T>(this `IEnumerable<T>` source, Func<T, bool> predicate)
{
    ArgumentNullException.ThrowIfNull(source);
    ArgumentNullException.ThrowIfNull(predicate);

    return WhereIterator(source, predicate);

    static `IEnumerable<T>` WhereIterator(`IEnumerable<T>` source, Func<T, bool> predicate)
    {
        foreach (var x in source)
            if (predicate(x))
                yield return x;
    }
}
```

Это паттерн всех LINQ-методов в BCL.

### 7.4. Когда оборачивать

✅ **Public API** — caller получает понятную ошибку без mystery.
✅ **Library code** — пользователи библиотеки не знают про defer execution.
✅ **Validation важна для correctness** — разница между null source и пустой коллекцией.

❌ **Internal helper** — если знаешь, что caller всегда foreach сразу — overkill.

### 7.5. Альтернатива — null-tolerant без validation

Иногда вообще не нужно throw, можно tolerantly:

```csharp
public `IEnumerable<T>` WhereOrEmpty<T>(this `IEnumerable<T>`? source, Func<T, bool> predicate)
{
    if (source is null) return [];   // empty
    return source.Where(predicate);
}
```

Зависит от контракта.

### 7.6. Eager + lazy combined

```csharp
public IEnumerable<Order> GetActiveOrders(int userId, DateTime? since)
{
    if (userId <= 0)
        throw new ArgumentException("userId must be positive", nameof(userId));

    var fromDate = since ?? DateTime.UtcNow.AddDays(-30);

    return Iterator();

    IEnumerable<Order> Iterator()
    {
        // Используется захваченный fromDate
        foreach (var order in QueryDb(userId, fromDate))
            if (order.Status != OrderStatus.Cancelled)
                yield return order;
    }
}
```

Локальная функция захватывает `userId`, `fromDate` из enclosing scope. Validation eager, computation lazy. Один из самых полезных паттернов C#.

> [!question]- Интервью: что такое eager validation + lazy execution?
> Паттерн для iterator-методов с validation параметров. Внешний метод (обычный, не iterator) — eager: проверяет параметры, throw exception если не валидны. Возвращает результат вызова локальной функции — iterator. Это решает проблему deferred execution: validation срабатывает сразу при вызове, генерация — отложена до первого MoveNext. Используется во всех LINQ-методах BCL. Без этого паттерна ArgumentException появляется при foreach, что путает caller-а.

---

## 8. Multiple iteration

### 8.1. Каждый foreach — новый enumerator

```csharp
public IEnumerable<int> WithLog()
{
    Console.WriteLine("Start");
    yield return 1;
    yield return 2;
    Console.WriteLine("End");
}

var iter = WithLog();

foreach (var _ in iter) { }
// Start
// End

foreach (var _ in iter) { }   // полный re-execute!
// Start
// End
```

Каждый `foreach` вызывает `GetEnumerator()` → новый instance state machine → тело выполняется с нуля.

### 8.2. Когда это проблема

```csharp
public IEnumerable<int> SlowQuery()
{
    Console.WriteLine("Query DB...");
    Thread.Sleep(1000);   // имитация запроса
    yield return 1;
    yield return 2;
}

var iter = SlowQuery();

foreach (var _ in iter) { }   // 1 секунда
foreach (var _ in iter) { }   // ещё 1 секунда!

var count = iter.Count();      // ещё 1 секунда — Count тоже итерирует!
var first = iter.First();       // ещё 1 секунда
```

**Memory** — iterator не кэширует. **Speed** — каждая итерация = полное выполнение.

### 8.3. Решение — материализовать через ToList/ToArray

```csharp
var cached = SlowQuery().ToList();   // 1 секунда — один раз

foreach (var _ in cached) { }   // мгновенно
foreach (var _ in cached) { }   // мгновенно
var count = cached.Count;        // O(1)
var first = cached[0];           // O(1)
```

Trade-off: память на List, но скорость и repeatability.

### 8.4. Decision rule

```
Caller итерирует один раз?
  → Iterator (yield), пусть будет lazy.

Caller итерирует много раз / нужен Count / нужен индекс?
  → Материализуй через ToList в caller-е.
  → Или возвращай List/`IReadOnlyList<T>` сразу.

Не уверен?
  → Возвращай `IEnumerable<T>` — caller сам решит ToList или нет.
```

### 8.5. `IReadOnlyCollection<T>` как middle ground

```csharp
public IReadOnlyCollection<int> GetItems()
{
    return Iterator().ToList();   // материализовано

    static IEnumerable<int> Iterator()
    {
        for (int i = 1; i <= 100; i++)
            yield return i;
    }
}
```

Возврат `IReadOnlyCollection<T>` сигнализирует caller-у: «здесь Count есть, репетитивная итерация дёшева».

### 8.6. Lazy vs cached — выбор

```csharp
// Lazy — для больших sources, потенциально не все нужны
public IEnumerable<Log> StreamLogs() { /* yield */ }

// Cached — для маленьких known sets, повторно нужно
public IReadOnlyList<Country> GetCountries() => _countries;   // List

// Generator с кешированием — Lazy<T>
private readonly Lazy<List<Config>> _configs = new(() => LoadConfigs());
public IReadOnlyList<Config> GetConfigs() => _configs.Value;
```

`Lazy<T>` — стандартная техника для thread-safe single-init.

> [!question]- Интервью: почему `iter.Count()` после foreach снова выполняет iterator?
> Каждое обращение к iterator (foreach, Count, ToList, First, любой LINQ) вызывает `GetEnumerator()` — создаётся новый instance state machine, тело iterator-метода выполняется заново с нуля. Side effects тоже повторяются. Это **deferred + non-cached** семантика. Решение: материализуй один раз через `ToList()` или `ToArray()`, дальше работай с коллекцией. Возвращай `IReadOnlyCollection<T>` или `IReadOnlyList<T>` если caller заведомо нужны Count/index.

---

## 9. yield + LINQ — как Where/Select построены

### 9.1. Все LINQ-методы — iterators

```csharp
// Упрощённая реализация Where
public static `IEnumerable<T>` Where<T>(this `IEnumerable<T>` source, Func<T, bool> predicate)
{
    foreach (var item in source)
        if (predicate(item))
            yield return item;
}

// Упрощённая Select
public static IEnumerable<TResult> Select<T, TResult>(this `IEnumerable<T>` source, Func<T, TResult> selector)
{
    foreach (var item in source)
        yield return selector(item);
}

// Take
public static `IEnumerable<T>` Take<T>(this `IEnumerable<T>` source, int count)
{
    if (count <= 0) yield break;
    int taken = 0;
    foreach (var item in source)
    {
        yield return item;
        if (++taken >= count) yield break;
    }
}
```

Реальные BCL-реализации сложнее (оптимизации для `IList<T>`, `Array`, `List<T>` через runtime checks), но семантика та же — все iterator-методы.

### 9.2. Цепочка iterator-ов = pipeline

```csharp
var result = numbers
    .Where(x => x > 0)        // iterator
    .Select(x => x * 2)        // iterator
    .Take(5)                    // iterator
    .ToList();                  // terminal — материализует
```

Между Where, Select, Take **нет** промежуточных коллекций. Каждый элемент проходит через всю цепочку, прежде чем перейти к следующему.

```
Поток данных:
numbers.MoveNext() → 1
  → Where: 1 > 0? yes → передаёт дальше
  → Select: 1 * 2 = 2 → передаёт дальше
  → Take: 1 из 5
  → ToList: добавляет 2

numbers.MoveNext() → -1
  → Where: -1 > 0? no → НЕ передаёт
  → Select не вызван
  → Take не вызван
  
numbers.MoveNext() → 3
  → Where: yes
  → Select: 6
  → Take: 2 из 5
  → ToList: добавляет 6

... продолжается до Take hit лимит
ToList завершает foreach.
```

### 9.3. Custom LINQ — свои iterator-extensions

```csharp
public static `IEnumerable<T>` WhereIndexed<T>(
    this `IEnumerable<T>` source,
    Func<T, int, bool> predicate)
{
    int index = 0;
    foreach (var item in source)
    {
        if (predicate(item, index))
            yield return item;
        index++;
    }
}

public static IEnumerable<TResult> SelectMany2<T, TResult>(
    this `IEnumerable<T>` source,
    Func<T, IEnumerable<TResult>> selector)
{
    foreach (var item in source)
        foreach (var sub in selector(item))
            yield return sub;
}

// Использование
var firstThree = numbers.WhereIndexed((x, i) => i < 3);
var allTags = posts.SelectMany2(p => p.Tags);
```

Простая реализация — несколько строк. LINQ — это набор iterator-extensions поверх IEnumerable.

### 9.4. Pipeline компонуется хорошо

Каждый iterator-extension независим — можно собирать pipeline как Lego:

```csharp
public static `IEnumerable<T>` WithoutDuplicates<T>(this `IEnumerable<T>` source)
{
    var seen = new HashSet<T>();
    foreach (var item in source)
        if (seen.Add(item))
            yield return item;
}

public static `IEnumerable<T>` Throttle<T>(this `IEnumerable<T>` source, TimeSpan delay)
{
    foreach (var item in source)
    {
        yield return item;
        Thread.Sleep(delay);
    }
}

// Composable
var processed = events
    .WithoutDuplicates()
    .Throttle(TimeSpan.FromMilliseconds(100))
    .Take(10);
```

### 9.5. Performance — chain не хуже одного цикла

Iterator chain имеет небольшой overhead (state machine на каждый этап), но в большинстве случаев пренебрежим. Главное преимущество — **читаемость и composability**.

Если perf критичен:

```csharp
// LINQ — readable, slight overhead
var sum = numbers.Where(x => x > 0).Sum();

// Manual loop — fastest
long sum = 0;
foreach (var x in numbers)
    if (x > 0) sum += x;
```

Разница обычно < 2x. В hot path ROI — manual loop. В обычном коде — LINQ.

### 9.6. Where после Where — компилятор не оптимизирует

```csharp
var result = numbers
    .Where(x => x > 0)
    .Where(x => x < 100);
```

Каждый Where создаёт свой iterator. **Компилятор не сливает их в один**. Можно слить вручную:

```csharp
var result = numbers.Where(x => x > 0 && x < 100);
```

Микро-оптимизация, существенна только в очень hot loop.

> [!question]- Интервью: как LINQ обрабатывает 1 миллион элементов без промежуточных коллекций?
> Каждый LINQ-метод (Where, Select, Take, OrderBy, и т.д.) — это iterator-extension с `yield return`. Они **не материализуют** результат — возвращают `IEnumerable<T>` с deferred execution. Когда terminal operator (ToList, Sum, First, foreach) запускает выполнение, элементы проходят через всю цепочку **по одному**. Никаких промежуточных List на 500K элементов. Это дает memory efficiency и возможность early exit (Take, First, Any). Цена — небольшой overhead state machines на каждый этап.

---

## 10. Exception handling

### 10.1. Throw в iterator — что происходит

```csharp
public IEnumerable<int> Buggy()
{
    yield return 1;
    yield return 2;
    throw new InvalidOperationException("Oops");
    yield return 3;   // никогда
}

try
{
    foreach (var x in Buggy())
        Console.WriteLine(x);
}
catch (InvalidOperationException ex)
{
    Console.WriteLine($"Caught: {ex.Message}");
}
// 1
// 2
// Caught: Oops
```

Throw происходит при `MoveNext()` — на момент когда state machine хочет вычислить следующий элемент. Caller получает exception в обычном catch вокруг foreach.

### 10.2. Cleanup через finally

```csharp
public IEnumerable<int> WithResource()
{
    var resource = AcquireResource();
    try
    {
        foreach (var x in resource.Items)
        {
            if (x < 0) throw new InvalidOperationException();
            yield return x;
        }
    }
    finally
    {
        resource.Dispose();
    }
}
```

Если throw случится — `finally` всё равно вызовется через `Dispose()` enumerator-а. Resource освободится.

### 10.3. using в iterator — как работает

```csharp
public IEnumerable<string> ReadLines(string path)
{
    using var reader = new StreamReader(path);   // ← OK даже с yield
    string? line;
    while ((line = reader.ReadLine()) is not null)
        yield return line;
}
```

`using` компилируется в `try/finally` с `Dispose()` в finally. С yield это работает: state machine вызывает Dispose реально только когда enumerator закрыт (в Dispose() самого enumerator-а).

Это критично для file handles, DB connections — закрываются правильно.

### 10.4. Exception в predicate / selector

```csharp
var result = numbers
    .Where(x => 10 / x > 0)   // throws DivideByZeroException if x == 0
    .ToList();
```

Exception в lambda для LINQ-методов — обычный case. Propagates через Where iterator вверх. Обрабатывай в catch вокруг foreach или ToList.

### 10.5. Aggregate exceptions — нет в обычном yield

В отличие от async, обычный iterator не накапливает exceptions. Первая throw останавливает выполнение. Если хочешь продолжить после ошибки:

```csharp
public IEnumerable<(T? Value, Exception? Error)> SafeIter<T>(IEnumerable<Func<T>> factories)
{
    foreach (var factory in factories)
    {
        T? value = default;
        Exception? error = null;
        try
        {
            value = factory();
        }
        catch (Exception ex)
        {
            error = ex;
        }
        yield return (value, error);   // ← yield ВНЕ try
    }
}
```

Применяется паттерн eager catch + lazy yield (раздел 5.3).

### 10.6. Cancellation — не встроено

Обычный iterator не поддерживает CancellationToken — нет места проверять. Workaround:

```csharp
public IEnumerable<int> CancellableRange(int start, int count, CancellationToken ct)
{
    for (int i = 0; i < count; i++)
    {
        ct.ThrowIfCancellationRequested();   // вручную
        yield return start + i;
    }
}

// Caller
var cts = new CancellationTokenSource(timeout: TimeSpan.FromSeconds(5));
foreach (var x in CancellableRange(0, 1000, cts.Token))
    Process(x);
```

Cancellation встраивается **вручную**. Для structured cancellation — IAsyncEnumerable (раздел 11).

> [!question]- Интервью: куда полетит exception, если throw в iterator-методе?
> Throw происходит при `MoveNext()` — то есть когда foreach пытается вычислить следующий элемент. Exception propagates как обычный — вверх по stack, попадает в catch вокруг foreach. После throw enumerator считается «закрытым», `Dispose()` вызывается через try/finally в foreach. Если в iterator есть try/finally — finally вызывается. Cleanup ресурсов работает правильно через `using` внутри iterator-метода.

---

## 11. IAsyncEnumerable (.NET Core 3.0+ / C# 8)

### 11.1. Зачем

Обычный iterator синхронный. Для async source (HTTP stream, DB cursor с await, file lines с async read) нужен `IAsyncEnumerable<T>`:

```csharp
// Синхронный iterator
public IEnumerable<string> ReadLines(string path)
{
    using var reader = new StreamReader(path);
    string? line;
    while ((line = reader.ReadLine()) is not null)
        yield return line;
}

// Async iterator
public async IAsyncEnumerable<string> ReadLinesAsync(string path)
{
    using var reader = new StreamReader(path);
    string? line;
    while ((line = await reader.ReadLineAsync()) is not null)
        yield return line;
}

await foreach (var line in ReadLinesAsync("big.txt"))
    Console.WriteLine(line);
```

`async IAsyncEnumerable<T>` + `yield return` + `await foreach` — три кита.

### 11.2. await foreach

```csharp
await foreach (var item in source)
{
    Process(item);
}
```

Компилятор разворачивает в:

```csharp
var enumerator = source.GetAsyncEnumerator();
try
{
    while (await enumerator.MoveNextAsync())
    {
        var item = enumerator.Current;
        Process(item);
    }
}
finally
{
    await enumerator.DisposeAsync();
}
```

Каждый `MoveNextAsync()` возвращает `ValueTask<bool>` — может быть completed синхронно (быстрый case) или await-ить (медленный).

### 11.3. CancellationToken и [EnumeratorCancellation]

```csharp
public async IAsyncEnumerable<T> Stream<T>(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    while (HasMore())
    {
        ct.ThrowIfCancellationRequested();
        var item = await FetchNext(ct);
        yield return item;
    }
}

// Caller
await foreach (var x in Stream<int>().WithCancellation(cts.Token))
    Process(x);
```

`[EnumeratorCancellation]` атрибут — позволяет caller-у через `WithCancellation(token)` пробросить токен в iterator. Без него токен проигнорируется.

### 11.4. ConfigureAwait в async iterator

```csharp
public async IAsyncEnumerable<int> Source([EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 100; i++)
    {
        await Task.Delay(10, ct).ConfigureAwait(false);   // в библиотечном коде
        yield return i;
    }
}

// Caller — может ConfigureAwait(false) в await foreach
await foreach (var x in Source().ConfigureAwait(false).WithCancellation(ct))
    Process(x);
```

`ConfigureAwait(false)` в библиотечном async iterator — стандарт. На await foreach — для UI / classic ASP.NET.

### 11.5. Where/Select для IAsyncEnumerable

В System.Linq.Async (NuGet):

```bash
dotnet add package System.Linq.Async
```

```csharp
using System.Linq;

await foreach (var user in db.Users.AsAsyncEnumerable()
    .Where(u => u.IsActive)
    .Select(u => new UserDto(u.Id, u.Name))
    .Take(100))
{
    Console.WriteLine(user);
}
```

LINQ для IAsyncEnumerable — отдельный пакет. EF Core 7+ предоставляет `.AsAsyncEnumerable()` для DbSet — стримит результаты.

### 11.6. Composing — async pipeline

```csharp
public async IAsyncEnumerable<Order> EnrichOrders(
    IAsyncEnumerable<int> orderIds,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var id in orderIds.WithCancellation(ct))
    {
        var order = await LoadOrder(id, ct);
        order.Items = await LoadItems(order.Id, ct);
        yield return order;
    }
}

// Использование
await foreach (var order in EnrichOrders(GetOrderIds(), cts.Token))
    Process(order);
```

Pipeline async-ий — каждый этап может await. Nice for streaming: первый order доступен ещё до загрузки всех IDs.

### 11.7. Когда IAsyncEnumerable

✅ **Используй когда:**
- Source данных async (HTTP stream, EF Core query, file with ReadLineAsync).
- Стрим неизвестного размера или большой.
- Хочешь backpressure (caller контролирует темп через await).
- Real-time data (logs, events).

❌ **Не нужно когда:**
- Все данные уже в памяти — обычный `IEnumerable<T>`.
- Нужны все элементы сразу — `Task<List<T>>` проще.

> [!question]- Интервью: чем `IAsyncEnumerable<T>` отличается от `Task<List<T>>`?
> `Task<List<T>>` — async-метод возвращает **всю** коллекцию когда готова. Caller ждёт всё. Память для всех элементов сразу. `IAsyncEnumerable<T>` — async iterator, элементы стримятся **по одному**. Caller обрабатывает по мере получения через `await foreach`. Память для одного элемента + state. Backpressure встроена — caller контролирует темп. Используй `IAsyncEnumerable` для большого/неизвестного размера source, real-time данных, EF Core стримов. `Task<List<T>>` — для маленьких known-size коллекций.

---

## 12. `Channel<T>` — альтернатива

### 12.1. Когда iterator не подходит

Iterator модель — **single producer, single consumer** в одном потоке. Не работает для:
- Multi-producer (несколько потоков пишут в стрим).
- Multi-consumer (несколько обработчиков читают).
- Producer и consumer в разных потоках с координацией.

Решение — `System.Threading.Channels`.

### 12.2. Базовое использование

```csharp
using System.Threading.Channels;

var channel = Channel.CreateUnbounded<int>();

// Producer
_ = Task.Run(async () =>
{
    for (int i = 1; i <= 100; i++)
    {
        await channel.Writer.WriteAsync(i);
    }
    channel.Writer.Complete();
});

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync())
{
    Console.WriteLine(item);
}
```

Channel — двунаправленный pipe с writer и reader. `ReadAllAsync()` возвращает `IAsyncEnumerable<T>` — можно использовать `await foreach`.

### 12.3. Bounded vs Unbounded

```csharp
// Unbounded — без ограничений (memory growth!)
var c1 = Channel.CreateUnbounded<int>();

// Bounded — фиксированный буфер, backpressure
var c2 = Channel.CreateBounded<int>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait,   // producer ждёт если full
    SingleReader = true,
    SingleWriter = false
});
```

Bounded дает backpressure — producer блокируется, если consumer не успевает. Unbounded удобнее, но рискует OutOfMemory если producer быстрее consumer.

### 12.4. Multi-producer, multi-consumer

```csharp
var channel = Channel.CreateBounded<WorkItem>(100);

// 4 producer'а
for (int i = 0; i < 4; i++)
{
    int producerId = i;
    _ = Task.Run(async () =>
    {
        while (true)
        {
            var item = await ProduceWork(producerId);
            await channel.Writer.WriteAsync(item);
        }
    });
}

// 8 consumer'ов
for (int i = 0; i < 8; i++)
{
    _ = Task.Run(async () =>
    {
        await foreach (var item in channel.Reader.ReadAllAsync())
        {
            await ProcessItem(item);
        }
    });
}
```

Channel сам синхронизирует доступ — не нужны locks. Production-grade инструмент.

### 12.5. Channel vs IAsyncEnumerable — когда что

| | IAsyncEnumerable | Channel |
|---|------------------|---------|
| Producer | Один (yield) | Любое количество |
| Consumer | Один (await foreach) | Любое количество |
| Backpressure | Через caller pacing | Bounded mode |
| Buffering | Нет | Да (bounded/unbounded) |
| Декларативность | Высокая | Средняя |
| Use case | Streaming source | Worker pool, message bus |

`IAsyncEnumerable` — для источников данных. `Channel` — для координации между задачами.

> [!question]- Интервью: когда использовать Channel вместо IAsyncEnumerable?
> Когда нужно multi-producer / multi-consumer координация между потоками. `IAsyncEnumerable` — single producer / single consumer iterator, идеален для streaming source данных (DB cursor, HTTP stream). `Channel<T>` — общий буфер с writer и reader API, поддерживает несколько producer'ов / consumer'ов одновременно, имеет встроенный backpressure через bounded mode. Используй для worker pool patterns, in-memory message bus, координации тяжёлых workflow.

---

## 13. Performance — overhead и оптимизации

### 13.1. State machine allocation

Каждый вызов iterator-метода создаёт class на heap:

```csharp
public IEnumerable<int> Range(int n) { for (int i = 0; i < n; i++) yield return i; }

// Каждый вызов = одна heap allocation (~40-60 байт)
var iter1 = Range(10);
var iter2 = Range(10);
```

В hot path с миллионами вызовов — заметная GC pressure.

### 13.2. Сравнение — yield vs обычный List

```csharp
[Benchmark]
public int YieldSum()
{
    int sum = 0;
    foreach (var x in YieldRange(1000)) sum += x;
    return sum;
}

[Benchmark]
public int ListSum()
{
    int sum = 0;
    foreach (var x in ListRange(1000)) sum += x;
    return sum;
}

[Benchmark]
public int LoopSum()
{
    int sum = 0;
    for (int i = 0; i < 1000; i++) sum += i;
    return sum;
}

IEnumerable<int> YieldRange(int n) { for (int i = 0; i < n; i++) yield return i; }
List<int> ListRange(int n) { var list = new List<int>(n); for (int i = 0; i < n; i++) list.Add(i); return list; }
```

Типичные результаты (.NET 8):
```
| Method   |     Mean | Allocated |
|--------- |---------:|----------:|
| LoopSum  |  0.5 us  |       0 B |    ← fastest
| YieldSum |  4.5 us  |      48 B |    ← state machine
| ListSum  |  3.0 us  |   4,024 B |    ← List на 1000 int
```

`LoopSum` быстрее всех. `YieldSum` имеет state machine allocation, но не материализует. `ListSum` аллоцирует огромный массив, медленнее на cache misses.

### 13.3. ValueTask и async iterators

`IAsyncEnumerable.MoveNextAsync()` возвращает `ValueTask<bool>`:

```csharp
public interface IAsyncEnumerator<out T> : IAsyncDisposable
{
    T Current { get; }
    ValueTask<bool> MoveNextAsync();
}
```

`ValueTask` оптимизирован для синхронно-завершающихся операций — без allocation, если результат сразу готов. Это важно для streaming, где большинство `MoveNextAsync` могут быть быстрыми.

### 13.4. Pooled iterators (advanced)

Для горячих сценариев можно использовать struct enumerator + ArrayPool:

```csharp
public struct PooledRange : IEnumerable<int>, IEnumerator<int>
{
    private int _start;
    private int _end;
    private int _current;

    public PooledRange(int start, int end) { _start = start; _end = end; _current = start - 1; }

    public int Current => _current;
    object IEnumerator.Current => _current;

    public IEnumerator<int> GetEnumerator() => this;
    IEnumerator IEnumerable.GetEnumerator() => this;

    public bool MoveNext() => ++_current < _end;
    public void Reset() => _current = _start - 1;
    public void Dispose() { }
}
```

Struct = no heap allocation. Но boxing случится, если caller возьмёт `IEnumerable<int>` через interface — переменная стает reference. Без interface (foreach pattern matching) — без boxing.

### 13.5. `Span<T>` и `Memory<T>` — нельзя в iterator

```csharp
public IEnumerable<int> NotAllowed(Span<int> data)   // ❌ CS8345
{
    foreach (var x in data) yield return x;
}
```

`Span<T>` — `ref struct`, не может быть полем класса. State machine — это class. Поэтому iterator не может принимать `Span<T>` параметр. Workaround: материализовать в `T[]` или передавать через `Memory<T>`.

### 13.6. Кэширование результата

Если iterator вызывается часто и результат стабилен:

```csharp
private static readonly List<int> _cached = ComputeOnce().ToList();
public static IEnumerable<int> ComputeOnce() { for (int i = 0; i < 100; i++) yield return i; }

// Caller использует _cached
foreach (var x in _cached) Process(x);
```

Trade-off: память на List vs скорость + repeatability.

### 13.7. yield в `foreach` — упрощение цикла

```csharp
// Без оптимизации
public IEnumerable<int> Doubled(IEnumerable<int> source)
{
    foreach (var x in source)
        yield return x * 2;
}

// LINQ — то же самое
public IEnumerable<int> Doubled2(IEnumerable<int> source) =>
    source.Select(x => x * 2);
```

Версия с LINQ короче, но использует Select-iterator под капотом — overhead тот же. Yield версия чуть прямолинейнее. Выбор стилистический.

> [!question]- Интервью: какой overhead у yield-метода?
> 1) Heap allocation state machine (~40-80 байт) на каждый вызов iterator-метода. 2) Виртуальные вызовы `MoveNext` через `IEnumerator<T>` interface (если caller использует `IEnumerable<T>` тип, не concrete). 3) Замыкание параметров и локалов в поля state machine. В hot loop с миллионами вызовов — заметная GC pressure. Оптимизации: pattern-based foreach с struct enumerator, кэширование результата через ToList, materialize если perf критичен. В большинстве случаев overhead приемлем — readability важнее.

---

## 14. Best Practices

### 14.1. Когда yield

- ✅ **Большой/неизвестный размер** — миллионы элементов, потоки.
- ✅ **Lazy generation** — caller может прервать (Take, First).
- ✅ **Бесконечные последовательности** — Fibonacci, natural numbers.
- ✅ **Pipeline** — chain через Where, Select, etc.
- ✅ **Streaming source** — file lines, DB cursor, HTTP stream.
- ❌ **Маленький known set** — верни List сразу.
- ❌ **Нужен Count / index** — материализуй или верни IReadOnlyList.
- ❌ **Multiple iteration** — кэшируй через ToList в caller.

### 14.2. Eager validation

- ✅ **Wrap iterator в обычный метод** — внешний делает validation, локальная функция yield.
- ✅ **`ArgumentNullException.ThrowIfNull`** для параметров.
- ✅ **`static` на локальной функции** — гарантирует не захватывает this.

### 14.3. Disposal

- ✅ **`using` внутри iterator** — закроет ресурсы при Dispose.
- ✅ **`try/finally`** — для cleanup без using.
- ❌ **`yield в try/catch`** — запрещено компилятором.

### 14.4. Контракт возвращаемого типа

- ✅ **`IEnumerable<T>`** — стандарт для public API.
- ✅ **`IReadOnlyList<T>`** — если знаешь Count и index доступен.
- ✅ **`IAsyncEnumerable<T>`** — для async source.
- ❌ **`IEnumerator<T>` напрямую** — редко уместно.

### 14.5. LINQ + yield

- ✅ **Custom iterator extensions** — на `IEnumerable<T>` для domain-специфичных операций.
- ✅ **Lazy chain** — Where → Select → Take без материализации.
- ✅ **Materialize в конце** — ToList/ToArray один раз для caller.

### 14.6. Async iterators

- ✅ **`async IAsyncEnumerable<T>`** + `await foreach`.
- ✅ **`[EnumeratorCancellation]`** на CancellationToken параметре.
- ✅ **`ConfigureAwait(false)`** в библиотечном коде.
- ✅ **`WithCancellation(token)`** на caller side.

### 14.7. Performance

- ✅ **`yield` для readability** — overhead < 5% в типичном коде.
- ✅ **Struct enumerator + pattern foreach** — для hot path.
- ❌ **Не вызывай iterator много раз** — материализуй.
- ❌ **Не используй для маленьких known sets** — overhead не оправдан.

### 14.8. Не делай

- ❌ Yield в try с catch.
- ❌ Side effects (DB, HTTP) без понимания deferred execution.
- ❌ Iterator с side effects + multiple iteration без cache.
- ❌ Захват `Span<T>` / `ref struct` параметров.
- ❌ Возврат `IEnumerable<T>` для маленьких known sets.

---

## 15. Decision tree

```
Что нужно?
│
├── Источник данных
│   ├── Малый known set (5-50) → return List<T>
│   ├── Большой / неизвестный → `IEnumerable<T>` с yield
│   ├── Бесконечный → `IEnumerable<T>` с yield
│   └── Async source → `IAsyncEnumerable<T>` с async + yield
│
├── Pipeline данных
│   ├── Chain трансформаций → LINQ (Where/Select/Take)
│   ├── Custom step → свой extension с yield
│   └── Multi-producer / consumer → `Channel<T>`
│
├── Caller pattern
│   ├── Один foreach → yield идеально
│   ├── Many iterations / Count / index → ToList в caller или IReadOnlyList
│   └── Прерывание (Take, First) → yield с lazy
│
├── Validation параметров
│   ├── Eager (нужно сразу throw) → wrapper + local function
│   └── Lazy (защита от мисъюза) → throw в iterator теле
│
├── Resources (file, DB)
│   ├── using внутри iterator → правильный cleanup
│   ├── try/finally → manual cleanup
│   └── try с catch и yield → REWRITE — eager catch + lazy yield снаружи
│
├── Cancellation
│   ├── Sync iterator → ct.ThrowIfCancellationRequested() вручную
│   └── Async → [EnumeratorCancellation] + WithCancellation
│
└── Performance critical
    ├── Struct enumerator + pattern foreach
    ├── Кэшировать результат через ToList
    └── Manual loop без iterator overhead
```

---

## 16. Cheat sheet

```csharp
// === Базовый iterator ===
public IEnumerable<int> Range(int start, int count)
{
    for (int i = 0; i < count; i++)
        yield return start + i;
}

// === yield break ===
public IEnumerable<int> UntilNegative(IEnumerable<int> source)
{
    foreach (var x in source)
    {
        if (x < 0) yield break;
        yield return x;
    }
}

// === Eager validation + lazy execution ===
public IEnumerable<int> Take(IEnumerable<int> source, int count)
{
    ArgumentNullException.ThrowIfNull(source);
    if (count < 0) throw new ArgumentOutOfRangeException(nameof(count));

    return Iterator();
    static IEnumerable<int> Iterator() { /* yield тут */ }
}

// === Resource management ===
public IEnumerable<string> ReadLines(string path)
{
    using var reader = new StreamReader(path);
    string? line;
    while ((line = reader.ReadLine()) is not null)
        yield return line;
}

// === Async iterator ===
public async IAsyncEnumerable<string> ReadLinesAsync(
    string path,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var reader = new StreamReader(path);
    string? line;
    while ((line = await reader.ReadLineAsync(ct)) is not null)
        yield return line;
}

// Caller
await foreach (var line in ReadLinesAsync(path).WithCancellation(cts.Token))
    Process(line);

// === Custom LINQ-style ===
public static `IEnumerable<T>` WhereIndexed<T>(
    this `IEnumerable<T>` source,
    Func<T, int, bool> predicate)
{
    int i = 0;
    foreach (var x in source)
    {
        if (predicate(x, i)) yield return x;
        i++;
    }
}

// === Materialize для repeatable iteration ===
var items = source.ToList();   // один раз
foreach (var x in items) Process1(x);
foreach (var x in items) Process2(x);

// === Channel для multi-producer ===
var channel = Channel.CreateBounded<int>(100);
// Producer: await channel.Writer.WriteAsync(item)
// Consumer: await foreach (var item in channel.Reader.ReadAllAsync())
```

| Сценарий | Решение |
|----------|---------|
| Простой генератор | yield return в `IEnumerable<T>` |
| Lazy pipeline | LINQ chain |
| Eager validation | wrapper + local function |
| Resource cleanup | using inside yield |
| Async source | IAsyncEnumerable + await foreach |
| Multi-iteration | ToList в caller |
| Cancellation | [EnumeratorCancellation] |
| Multi-producer | `Channel<T>` |
| Hot path perf | struct enumerator |

---

## 17. Common Pitfalls — с механизмами

### 17.1. Validation срабатывает поздно

```csharp
public IEnumerable<int> Take(IEnumerable<int> source, int count)
{
    if (count < 0) throw new ArgumentException();   // ❌ throws при MoveNext, не при вызове
    foreach (var x in source) { /* ... */ yield return x; }
}
```

**Механизм:** тело iterator-метода = `MoveNext()` state machine. Throw происходит при первой MoveNext, не при вызове метода.

**Фикс:** wrapper + local function (раздел 7).

### 17.2. yield в try с catch

```csharp
public IEnumerable<int> Buggy()
{
    try { yield return 1; }   // ❌ CS1626
    catch { }
}
```

**Механизм:** state machine не умеет правильно обработать catch вокруг yield.

**Фикс:** eager catch (try без yield внутри), yield снаружи try.

### 17.3. Multiple iteration вызывает повторное выполнение

```csharp
var iter = SlowQuery();   // не выполнено
foreach (var _ in iter) { }   // 1 секунда
foreach (var _ in iter) { }   // ещё 1 секунда!
```

**Механизм:** каждый foreach = новый GetEnumerator = новый instance state machine = тело выполняется заново.

**Фикс:** `var cached = SlowQuery().ToList();` один раз.

### 17.4. Side effects срабатывают неожиданно

```csharp
public IEnumerable<User> LoadUsers()
{
    Console.WriteLine("Loading...");   // ← НЕ при вызове LoadUsers
    yield return new User();
}

var iter = LoadUsers();   // тихо
foreach (var _ in iter) { }   // "Loading..." здесь!
```

**Механизм:** тело iterator выполняется лениво.

**Фикс:** если важно eager — материализуй через ToList сразу.

### 17.5. Reset не работает

```csharp
public IEnumerable<int> Range() { yield return 1; yield return 2; }

var iter = Range();
var en = iter.GetEnumerator();
en.MoveNext();
en.Reset();   // ❌ NotSupportedException
```

**Механизм:** auto-generated state machine не реализует Reset (по design).

**Фикс:** `iter.GetEnumerator()` ещё раз для нового enumerator. Или ручная реализация (раздел 6).

### 17.6. Boxing в `IEnumerable<T>`

```csharp
public List<int>.Enumerator GetEnumerator() { /* struct */ }

// Pattern-based foreach — без boxing
foreach (var x in list) { }

// Через interface — boxing
IEnumerable<int> ie = list;
foreach (var x in ie) { }   // struct enumerator boxed
```

**Механизм:** struct → interface = boxing. Caller использует interface variable.

**Фикс:** keep concrete type где возможно. Pattern-based foreach использует struct без interface.

### 17.7. `Span<T>` не может быть параметром iterator

```csharp
public IEnumerable<int> Process(Span<int> data) { /* ❌ */ }
```

**Механизм:** Span — ref struct, не может быть полем class (state machine).

**Фикс:** `T[]`, `Memory<T>`, или скопировать в массив до iterator.

### 17.8. Async iterator без EnumeratorCancellation

```csharp
public async IAsyncEnumerable<int> Stream(CancellationToken ct = default)
{
    while (true)
    {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(100);   // не использует caller-ский ct!
        yield return 1;
    }
}

await foreach (var x in Stream().WithCancellation(cts.Token))   // ct не пробрасывается
    Process(x);
```

**Механизм:** без `[EnumeratorCancellation]` атрибута компилятор не знает, какой параметр получит token из `WithCancellation`.

**Фикс:** `[EnumeratorCancellation] CancellationToken ct = default`.

### 17.9. yield в lambda — не работает

```csharp
Func<IEnumerable<int>> f = () => { yield return 1; };   // ❌
```

**Механизм:** lambdas не могут быть iterators.

**Фикс:** локальная функция вместо lambda.

```csharp
IEnumerable<int> Local() { yield return 1; }
Func<IEnumerable<int>> f = Local;
```

### 17.10. Возврат `IEnumerable<T>` для known small set

```csharp
public IEnumerable<int> GetTopThree() { yield return 1; yield return 2; yield return 3; }
```

**Механизм:** state machine allocation overhead, caller вынужден ToList для Count/index.

**Фикс:** `return new[] { 1, 2, 3 };` или `IReadOnlyList<int>`.

> [!question]- Интервью: топ-3 ловушки yield в C#?
> 1) **Validation срабатывает поздно** — throw в теле iterator-метода = throw при первом MoveNext, не при вызове метода. Caller получает unhelpful stack. Решение: wrapper + local function. 2) **Multiple iteration** — каждый foreach создаёт новый enumerator, тело выполняется заново со всеми side effects. Решение: материализуй через ToList в caller. 3) **yield в try/catch запрещён** — state machine не умеет catch вокруг yield. Решение: eager catch (try без yield внутри), yield снаружи try.

---

## 18. Practice — упражнения с разбором

### 18.1. Custom Range с step

**Задача.** Реализовать `Range(start, end, step)` через yield, поддерживая negative step.

```csharp
public static IEnumerable<int> Range(int start, int end, int step = 1)
{
    if (step == 0)
        throw new ArgumentException("Step cannot be zero", nameof(step));

    return Iterator(start, end, step);

    static IEnumerable<int> Iterator(int start, int end, int step)
    {
        if (step > 0)
        {
            for (int i = start; i < end; i += step)
                yield return i;
        }
        else
        {
            for (int i = start; i > end; i += step)
                yield return i;
        }
    }
}

foreach (var x in Range(0, 10, 2))
    Console.WriteLine(x);
// 0, 2, 4, 6, 8

foreach (var x in Range(10, 0, -2))
    Console.WriteLine(x);
// 10, 8, 6, 4, 2

Range(0, 10, 0);   // throws ArgumentException сразу!
```

**Разбор:** eager validation в wrapper, lazy generation в local function. Step может быть positive или negative — два случая в loop. Caller получает understandable exception сразу при step=0, не отложенно.

### 18.2. Custom LINQ — DistinctBy

**Задача.** Реализовать `DistinctBy<T, TKey>(source, keySelector)` — distinct по custom ключу.

```csharp
public static `IEnumerable<T>` DistinctBy<T, TKey>(
    this `IEnumerable<T>` source,
    Func<T, TKey> keySelector)
{
    ArgumentNullException.ThrowIfNull(source);
    ArgumentNullException.ThrowIfNull(keySelector);

    return Iterator(source, keySelector);

    static `IEnumerable<T>` Iterator(`IEnumerable<T>` source, Func<T, TKey> keySelector)
    {
        var seen = new HashSet<TKey>();
        foreach (var item in source)
        {
            if (seen.Add(keySelector(item)))
                yield return item;
        }
    }
}

var users = new[]
{
    new { Id = 1, Email = "a@x.com" },
    new { Id = 2, Email = "a@x.com" },   // duplicate by email
    new { Id = 3, Email = "b@x.com" }
};

var unique = users.DistinctBy(u => u.Email);
foreach (var u in unique)
    Console.WriteLine($"{u.Id}: {u.Email}");
// 1: a@x.com
// 3: b@x.com
```

**Разбор:** state — `HashSet<TKey>` — сохраняется между yield-ами в state machine. `Add` возвращает true если новый, тогда yield. .NET 6+ имеет встроенный `DistinctBy`, но это пример того, как писать LINQ-style extensions.

### 18.3. Async stream с paging API

**Задача.** Стримить все items из paged API через IAsyncEnumerable.

```csharp
public async IAsyncEnumerable<Item> StreamAllItems(
    HttpClient http,
    string baseUrl,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    int page = 1;
    while (true)
    {
        ct.ThrowIfCancellationRequested();

        var response = await http.GetFromJsonAsync<PagedResponse<Item>>(
            $"{baseUrl}?page={page}&pageSize=100",
            ct);

        if (response is null || response.Items.Count == 0)
            yield break;

        foreach (var item in response.Items)
            yield return item;

        if (!response.HasMore) yield break;
        page++;
    }
}

public record PagedResponse<T>(List<T> Items, bool HasMore);
public record Item(int Id, string Name);

// Caller
var cts = new CancellationTokenSource(TimeSpan.FromMinutes(5));
await foreach (var item in StreamAllItems(http, "https://api.com/items").WithCancellation(cts.Token))
{
    Console.WriteLine(item);
    if (item.Id > 1000) break;   // early exit OK
}
```

**Разбор:** classic паттерн async streaming через paged API. Caller получает первую страницу мгновенно, не ждёт всех. Может прервать через break — последующие страницы не загрузятся (cancel HTTP). `[EnumeratorCancellation]` пробрасывает токен через WithCancellation.

### 18.4. Eager validation pattern

**Задача.** Реализовать `ChunkBy<T>(source, size)` с правильным eager validation.

```csharp
public static IEnumerable<`IReadOnlyList<T>`> ChunkBy<T>(
    this `IEnumerable<T>` source,
    int size)
{
    ArgumentNullException.ThrowIfNull(source);
    if (size <= 0)
        throw new ArgumentOutOfRangeException(nameof(size), "Size must be positive");

    return Iterator(source, size);

    static IEnumerable<`IReadOnlyList<T>`> Iterator(`IEnumerable<T>` source, int size)
    {
        var chunk = new List<T>(size);
        foreach (var item in source)
        {
            chunk.Add(item);
            if (chunk.Count == size)
            {
                yield return chunk;
                chunk = new List<T>(size);
            }
        }
        if (chunk.Count > 0)
            yield return chunk;
    }
}

var nums = Enumerable.Range(1, 10);
foreach (var chunk in nums.ChunkBy(3))
    Console.WriteLine(string.Join(",", chunk));
// 1,2,3
// 4,5,6
// 7,8,9
// 10

nums.ChunkBy(0);   // throws ArgumentOutOfRangeException сразу!
```

**Разбор:** в .NET 6+ есть `Chunk(size)` встроенно — пример того, как писать. Validation срабатывает сразу при вызове, генерация ленивая. Note: `IReadOnlyList<T>` в return — caller знает Count для каждого chunk.

### 18.5. Resource-safe file streaming

**Задача.** Стримить строки из CSV-файла, парсить в DTO, безопасно закрывать file даже при exception.

```csharp
public record CsvRow(string Id, string Name, decimal Price);

public static IEnumerable<CsvRow> StreamCsv(string path)
{
    ArgumentNullException.ThrowIfNull(path);
    if (!File.Exists(path))
        throw new FileNotFoundException("CSV file not found", path);

    return Iterator(path);

    static IEnumerable<CsvRow> Iterator(string path)
    {
        using var reader = new StreamReader(path);

        var header = reader.ReadLine();   // skip header
        if (header is null) yield break;

        string? line;
        int lineNum = 1;
        while ((line = reader.ReadLine()) is not null)
        {
            lineNum++;
            var parts = line.Split(',');
            if (parts.Length != 3)
                throw new FormatException($"Line {lineNum}: expected 3 columns");

            // Eager parsing инcide try НЕ нужно — yield снаружи
            var row = new CsvRow(parts[0], parts[1], decimal.Parse(parts[2], CultureInfo.InvariantCulture));
            yield return row;
        }
    }
}

// Caller
foreach (var row in StreamCsv("products.csv").Take(10))   // только первые 10
    Console.WriteLine(row);
// File автоматически закрывается через using в Dispose
```

**Разбор:** `using var reader` внутри iterator — компилируется в try/finally в state machine. При Dispose() (включая прерывание через Take/break) reader корректно закрывается. Eager validation file existence в wrapper. `decimal.Parse(InvariantCulture)` — стабильный парсинг для wire format.

---

## 19. Bonus practice — продвинутые упражнения

### 19.1. Buffered iterator с lookahead

**Задача.** Реализовать iterator, который позволяет «заглянуть» на N элементов вперёд (peek) без потребления.

```csharp
public class PeekableEnumerator<T> : IDisposable
{
    private readonly IEnumerator<T> _source;
    private readonly Queue<T> _buffer = new();

    public PeekableEnumerator(IEnumerable<T> source)
    {
        _source = source.GetEnumerator();
    }

    public bool MoveNext()
    {
        if (_buffer.Count > 0)
        {
            _buffer.Dequeue();
            return _buffer.Count > 0 || _source.MoveNext();
        }
        return _source.MoveNext();
    }

    public T Current => _buffer.Count > 0 ? _buffer.Peek() : _source.Current;

    public bool TryPeek(int offset, out T? value)
    {
        while (_buffer.Count <= offset)
        {
            if (!_source.MoveNext())
            {
                value = default;
                return false;
            }
            _buffer.Enqueue(_source.Current);
        }
        value = _buffer.ElementAt(offset);
        return true;
    }

    public void Dispose() => _source.Dispose();
}

// Использование
var nums = Enumerable.Range(1, 10);
using var peek = new PeekableEnumerator<int>(nums);

while (peek.MoveNext())
{
    if (peek.TryPeek(1, out var next))
        Console.WriteLine($"Current: {peek.Current}, Next: {next}");
    else
        Console.WriteLine($"Current: {peek.Current}, last");
}
```

**Разбор:** обычный iterator не позволяет `peek` — caller обязан consume. Custom enumerator с buffer queue даёт lookahead. Полезно для парсеров, лексеров, stream-обработки с context. Не yield-based — слишком сложно для state machine.

### 19.2. Producer-consumer через Channel и async iterator

**Задача.** Worker pool: один producer читает file lines, 4 consumer'а обрабатывают параллельно.

```csharp
public async Task ProcessFile(string path, CancellationToken ct = default)
{
    var channel = Channel.CreateBounded<string>(100);

    // Producer
    var producer = Task.Run(async () =>
    {
        using var reader = new StreamReader(path);
        string? line;
        while ((line = await reader.ReadLineAsync(ct)) is not null)
        {
            await channel.Writer.WriteAsync(line, ct);
        }
        channel.Writer.Complete();
    }, ct);

    // 4 Consumers
    var consumers = Enumerable.Range(0, 4).Select(i => Task.Run(async () =>
    {
        await foreach (var line in channel.Reader.ReadAllAsync(ct))
        {
            await ProcessLine(i, line, ct);
        }
    }, ct)).ToArray();

    await Task.WhenAll(consumers.Append(producer));
}

async Task ProcessLine(int workerId, string line, CancellationToken ct)
{
    await Task.Delay(50, ct);
    Console.WriteLine($"Worker {workerId}: {line}");
}
```

**Разбор:** `IAsyncEnumerable` для one-to-one streaming не подходит для work distribution. `Channel<T>` + multiple consumers решает. `BoundedChannelOptions(100)` дает backpressure — producer блокируется когда buffer full, не съедает память на гигабайтном файле. `ReadAllAsync` возвращает `IAsyncEnumerable<T>` — все consumer'ы видят свою часть данных.

---

### 19.3. Stateful iterator — moving average

**Задача.** Реализовать iterator, вычисляющий moving average последних N элементов.

```csharp
public static IEnumerable<double> MovingAverage(this IEnumerable<double> source, int windowSize)
{
    if (windowSize <= 0)
        throw new ArgumentOutOfRangeException(nameof(windowSize));

    return Iterator(source, windowSize);

    static IEnumerable<double> Iterator(IEnumerable<double> source, int windowSize)
    {
        var window = new Queue<double>(windowSize);
        double sum = 0;

        foreach (var x in source)
        {
            window.Enqueue(x);
            sum += x;

            if (window.Count > windowSize)
                sum -= window.Dequeue();

            yield return sum / window.Count;
        }
    }
}

var prices = new[] { 10.0, 12, 14, 13, 15, 18, 20 };
foreach (var avg in prices.MovingAverage(3))
    Console.WriteLine($"{avg:F2}");
// 10.00 (only 1 element)
// 11.00 (avg 10, 12)
// 12.00 (avg 10, 12, 14)
// 13.00 (avg 12, 14, 13)
// 14.00 (avg 14, 13, 15)
// 15.33 (avg 13, 15, 18)
// 17.67 (avg 15, 18, 20)
```

**Разбор:** state — `Queue<double>` и текущая sum — сохраняются между yield в state machine. Скользящее окно: при добавлении — sum += new, при достижении лимита — sum -= old. Постоянная сложность O(1) на элемент. Это паттерн для streaming analytics, financial indicators, signal processing.

### 19.4. Zip-like iterator для двух последовательностей

**Задача.** Реализовать `Zip3<T1, T2, T3>` для трёх последовательностей.

```csharp
public static IEnumerable<(T1, T2, T3)> Zip3<T1, T2, T3>(
    IEnumerable<T1> first,
    IEnumerable<T2> second,
    IEnumerable<T3> third)
{
    ArgumentNullException.ThrowIfNull(first);
    ArgumentNullException.ThrowIfNull(second);
    ArgumentNullException.ThrowIfNull(third);

    return Iterator(first, second, third);

    static IEnumerable<(T1, T2, T3)> Iterator(
        IEnumerable<T1> first, IEnumerable<T2> second, IEnumerable<T3> third)
    {
        using var e1 = first.GetEnumerator();
        using var e2 = second.GetEnumerator();
        using var e3 = third.GetEnumerator();

        while (e1.MoveNext() && e2.MoveNext() && e3.MoveNext())
        {
            yield return (e1.Current, e2.Current, e3.Current);
        }
    }
}

var ids = new[] { 1, 2, 3 };
var names = new[] { "Alice", "Bob", "Charlie" };
var emails = new[] { "a@x.com", "b@x.com", "c@x.com" };

foreach (var (id, name, email) in Zip3(ids, names, emails))
    Console.WriteLine($"{id}: {name} ({email})");
// 1: Alice (a@x.com)
// 2: Bob (b@x.com)
// 3: Charlie (c@x.com)
```

**Разбор:** ручной контроль над enumerator'ами через `using` — все три закрываются правильно даже при exception. `MoveNext()` каждого до первого fail — останавливаемся на shortest sequence. С .NET 6+ есть встроенный `Zip` с tuple, но `Zip3` нет — пример как написать.

---

## 20. Cross-language deep — отличия от Python / JavaScript / Rust

### 20.1. Python generators

```python
# Python
def count_to(n):
    for i in range(1, n + 1):
        yield i

for x in count_to(5):
    print(x)
```

Семантически идентично C#. Различия:
- Python `yield` встроен в язык изначально (с 2.2).
- Python также имеет `yield from delegation` — `yield from other_gen()` — в C# нет такого, нужен `foreach` + `yield return`.
- `send()` метод в Python — двунаправленная коммуникация. В C# нет.
- Python generators не имеют generic-параметров (динамическая типизация).

### 20.2. JavaScript generators

```javascript
function* countTo(n) {
    for (let i = 1; i <= n; i++)
        yield i;
}

for (const x of countTo(5))
    console.log(x);
```

Тоже близко. Различия:
- `function*` syntax вместо ключевого слова в обычном методе.
- Generator имеет `next({value, done})` API — close к C# `MoveNext()` + `Current`.
- `yield expression` возвращает результат `next(arg)` от caller — двунаправленная как в Python.
- Async generators — `async function*` + `for await` — эквивалент C# `IAsyncEnumerable` + `await foreach`.

### 20.3. Rust — нет yield, есть Iterator trait

```rust
// Rust — реализация трейта вручную
struct Counter {
    count: u32,
    max: u32,
}

impl Iterator for Counter {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        if self.count < self.max {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}

for x in (Counter { count: 0, max: 5 }) {
    println!("{}", x);
}
```

Rust **не имеет** `yield` keyword (в stable). Generators в pre-design (`gen` blocks). Стандартный путь — реализовать `Iterator` trait вручную. Преимущество: zero-cost abstraction (struct, no heap), composable через iterator combinators.

### 20.4. Kotlin sequence builder

```kotlin
fun countTo(n: Int) = sequence {
    for (i in 1..n)
        yield(i)
}

countTo(5).forEach { println(it) }
```

`sequence { }` — функциональный builder. `yield()` — function call. Семантически близко к C# yield. Также есть `sequenceOf`, `generateSequence` для других случаев. Coroutines + Flow — для async.

### 20.5. Java — Stream API

```java
Stream.iterate(1, i -> i + 1)
    .limit(5)
    .forEach(System.out::println);
```

Java не имеет `yield` для iterator (только в `switch` C# 8 эквивалент). Stream API — pipeline, eager или lazy в зависимости от operations. Не позволяет custom iterators так компактно как C# yield. Для генерации — `Stream.iterate`, `Stream.generate`, `Spliterator.iterate`.

### 20.6. Comparison table

| | C# yield | Python yield | JS function* | Rust | Kotlin sequence | Java Stream |
|---|----------|--------------|--------------|------|-----------------|-------------|
| Lazy | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Two-way (send) | ❌ | ✅ | ✅ | ❌ | ❌ | ❌ |
| Async generators | ✅ IAsyncEnumerable | ✅ async def | ✅ async function* | ❌ | ✅ Flow | ❌ |
| Cancellation | ✅ ct + EnumeratorCancellation | через GeneratorExit | через AbortController | ✅ Drop trait | через coroutine scope | manual |
| Static typing | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Composable (LINQ-like) | ✅ LINQ | ✅ itertools | ✅ via libs | ✅ Iterator trait | ✅ | ✅ Stream API |

> [!question]- Интервью: чем C# yield отличается от Python generators?
> Семантически очень близко — оба создают state machine с lazy evaluation. Различия: 1) C# имеет static типизацию, Python динамическую. 2) Python поддерживает `send(value)` для двунаправленной коммуникации (caller передаёт value в generator), C# не имеет. 3) Python `yield from` для делегирования — в C# нужен `foreach` + `yield return`. 4) Async — Python `async def + yield` ↔ C# `async IAsyncEnumerable<T> + yield return`. Концепция одинаковая, синтаксис разный.

---

## 21. Что читать дальше — порядок и почему

1. **[[csharp-basics|C# Basics]]** — for/foreach, IEnumerable, generics.
2. **[[collections-linq|Collections и LINQ]]** — где iterators main use case.
3. **Async deep dive** — async/await, Task, ValueTask, async streams.
4. **`Channel<T>` deep** — multi-producer/consumer patterns.
5. **System.IO Streams** — File, Pipe, NetworkStream — основа streaming.
6. **EF Core streaming** — `AsAsyncEnumerable()` для больших queries.
7. **Reactive Extensions (Rx)** — `IObservable<T>` как push-стрим (vs IEnumerable pull).
8. **Performance & Benchmarking** — оптимизации iterators.

---

## 22. См. также

- [[csharp-basics|C# Basics]] — основы IEnumerable
- [[collections-linq|Collections и LINQ]] — практическое применение
- [[extension-methods|Extension Methods]] — для custom iterator extensions
- [[delegates-events|Delegates и Events]] — Func/Action для predicates
- Async deep dive — async/await + IAsyncEnumerable
- `Channel<T>` — multi-producer/consumer
- System.IO Streams — file/network streaming
- Rx.NET — push-based streams
- EF Core streaming — AsAsyncEnumerable
- Performance — struct enumerators, Span
- Source Generators — компилирующие генерация iterators

---

## 23. Reading list

- **Microsoft Docs — yield statement** — learn.microsoft.com/dotnet/csharp/language-reference/statements/yield
- **Microsoft Docs — `IEnumerable<T>`** — learn.microsoft.com/dotnet/api/system.collections.generic.ienumerable-1
- **Microsoft Docs — `IAsyncEnumerable<T>`** — learn.microsoft.com/dotnet/api/system.collections.generic.iasyncenumerable-1
- **Microsoft Docs — System.Threading.Channels** — learn.microsoft.com/dotnet/core/extensions/channels
- **Eric Lippert — Iterators series** — ericlippert.com (поиск «iterators»)
- **Jon Skeet — C# in Depth** — chapter «Iterators»
- **Stephen Toub — async iterators** — devblogs.microsoft.com/dotnet
- **Andrew Lock — IAsyncEnumerable patterns** — andrewlock.net
- **Adam Sitnik — performance of iterators** — adamsitnik.com
- **Marc Gravell — yield optimization** — blog.marcgravell.com
- **System.Linq.Async** — github.com/dotnet/reactive (NuGet System.Linq.Async)
- **C# Language Specification — Iterators** — github.com/dotnet/csharplang/blob/main/spec/classes.md
- **SharpLab** — sharplab.io — посмотреть IL state machine
- **BenchmarkDotNet** — benchmarkdotnet.org — измерения performance
- **Reactive Extensions (Rx.NET)** — github.com/dotnet/reactive — push-based streams

---
