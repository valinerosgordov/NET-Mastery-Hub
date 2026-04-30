---
tags: [csharp, iterators, yield, ienumerable, lazy, junior, middle]
level: Junior to Middle
date: 2026-04-30
---

# Iterators и yield — ленивая итерация

> **Полный гайд по `yield return`**: что это, как работает, когда использовать. Под капотом IEnumerable\<T\>, custom iterators, async streams (IAsyncEnumerable). Closes пробел "почему `yield` важен и как избежать частых ошибок".

---

## Что это, зачем и когда

### Что такое iterator

**Метод который возвращает sequence значений** через `yield return` — без создания полной коллекции в памяти.

```csharp
// Без yield — нужно создать всю коллекцию
public IEnumerable<int> GetNumbers()
{
    var list = new List<int>();
    for (int i = 0; i < 1_000_000; i++)
        list.Add(i);
    return list;
}
// 1M ints в памяти сразу!

// С yield — lazy, по одному
public IEnumerable<int> GetNumbers()
{
    for (int i = 0; i < 1_000_000; i++)
        yield return i;
}
// 0 памяти до итерации, потом по 1 int за раз
```

### Когда использовать

✅ **Yield хорош когда:**
- Большие sequences (миллионы элементов)
- Бесконечные sequences
- Дорогие elements (загрузка файлов, HTTP calls)
- Ранний exit (`break`, `Take(10)`)
- Streaming данных (file lines, DB results)

❌ **Yield не нужен когда:**
- Маленькая фиксированная коллекция (10 элементов)
- Multiple iteration of same sequence
- Index access нужен (yield — только forward)
- Caller хочет `Count` без enumeration

### Под капотом

Компилятор генерит **state machine** — class, реализующий `IEnumerator<T>`:

```csharp
public IEnumerable<int> Range(int n)
{
    for (int i = 0; i < n; i++)
        yield return i;
}

// Компилятор генерит примерно:
private sealed class RangeStateMachine : IEnumerable<int>, IEnumerator<int>
{
    private int _state = 0;
    private int _current;
    private int _i;
    private int _n;

    public bool MoveNext()
    {
        switch (_state)
        {
            case 0:
                _i = 0;
                _state = 1;
                goto case 1;
            case 1:
                if (_i < _n)
                {
                    _current = _i;
                    _i++;
                    return true;
                }
                _state = -1;
                return false;
        }
        return false;
    }

    public int Current => _current;
    // ... и т.д.
}
```

Это **похоже на async/await state machine** — те же compiler tricks.

См. [[reflection-expression-trees|Reflection и Expression Trees]] для метапрограммирования.

---

## 1. Базовый yield

### `yield return` — выдать значение

```csharp
public IEnumerable<int> Squares(int max)
{
    for (int i = 1; i <= max; i++)
    {
        yield return i * i;  // выдаёт значение, ждёт следующий MoveNext
    }
}

// Использование
foreach (var sq in Squares(5))
{
    Console.WriteLine(sq);  // 1, 4, 9, 16, 25
}
```

Когда `foreach` вызывает `MoveNext`:
1. Метод выполняется до `yield return`
2. Текущее значение становится `Current`
3. Метод **приостанавливается**
4. На следующий `MoveNext` — продолжает с того места
5. Когда метод доходит до конца или `yield break` — итерация заканчивается

### `yield break` — досрочный выход

```csharp
public IEnumerable<int> ReadUntilNegative(int[] arr)
{
    foreach (var n in arr)
    {
        if (n < 0) yield break;  // прерываем итерацию
        yield return n;
    }
}

ReadUntilNegative(new[] { 1, 2, 3, -1, 4, 5 });
// 1, 2, 3
```

---

## 2. Lazy execution — главное преимущество

### Deferred execution

```csharp
public IEnumerable<int> Numbers()
{
    Console.WriteLine("Starting");
    for (int i = 0; i < 5; i++)
    {
        Console.WriteLine($"Yielding {i}");
        yield return i;
    }
    Console.WriteLine("Done");
}

// Создание iterator'а — НЕ выполняет код!
var nums = Numbers();
Console.WriteLine("Created iterator");

// Только foreach запускает выполнение
foreach (var n in nums)
{
    Console.WriteLine($"Got {n}");
}
```

Output:
```
Created iterator         ← код Numbers() ещё не выполнялся!
Starting
Yielding 0
Got 0
Yielding 1
Got 1
...
Done
```

### Bесконечные sequences

```csharp
public IEnumerable<int> Naturals()
{
    int i = 1;
    while (true)
    {
        yield return i++;
    }
}

// Берём только первые 10
var first10 = Naturals().Take(10);
foreach (var n in first10)
{
    Console.WriteLine(n);  // 1..10
}
// Не зависает! `yield` останавливается после 10 значений.
```

### Pipeline of operations

```csharp
public IEnumerable<int> ReadFileNumbers(string path)
{
    foreach (var line in File.ReadLines(path))  // lazy
    {
        if (int.TryParse(line, out int n))
            yield return n;
    }
}

var result = ReadFileNumbers("data.txt")  // lazy
    .Where(n => n > 0)                     // lazy (LINQ)
    .Take(100)                              // lazy
    .Sum();                                 // ← выполнение!

// Только для первых 100 positive чисел читается файл!
// Если они в первой 1000 строк — остальные 999000 не читаются.
```

LINQ построен на iterators. Большинство LINQ методов — lazy.

См. [[collections-linq|Collections и LINQ]].

---

## 3. Re-iteration

### Iterator можно проитерировать **несколько раз**

```csharp
public IEnumerable<int> GetNumbers()
{
    Console.WriteLine("Starting");
    for (int i = 0; i < 3; i++)
        yield return i;
}

var nums = GetNumbers();

foreach (var n in nums) { Console.WriteLine($"First: {n}"); }
foreach (var n in nums) { Console.WriteLine($"Second: {n}"); }
```

Output:
```
Starting
First: 0
First: 1
First: 2
Starting        ← код выполняется ЗАНОВО!
Second: 0
Second: 1
Second: 2
```

Каждый `foreach` создаёт **новый enumerator** и выполняет метод заново.

### Pitfall: side effects при re-iteration

```csharp
public IEnumerable<int> ReadAndProcess()
{
    using var conn = new DatabaseConnection();  // ⚠️ открывается каждый foreach!
    foreach (var row in conn.Query())
        yield return row.Number;
}

// Caller
var nums = ReadAndProcess();
nums.Count();   // Open + Read + Close
nums.Count();   // ОПЯТЬ Open + Read + Close!
```

**Решение:** материализуй результат если нужно несколько раз:

```csharp
var nums = ReadAndProcess().ToList();  // выполнение один раз
nums.Count();   // ✅ uses list
nums.Count();   // ✅ uses list
```

---

## 4. Iterator state — каждый foreach свой

```csharp
public IEnumerable<int> Range(int n)
{
    for (int i = 0; i < n; i++)
        yield return i;
}

var seq = Range(3);

var iter1 = seq.GetEnumerator();
var iter2 = seq.GetEnumerator();

iter1.MoveNext(); iter1.Current;  // 0
iter1.MoveNext(); iter1.Current;  // 1

iter2.MoveNext(); iter2.Current;  // 0 ← свой state!
```

Каждый `GetEnumerator` — independent state machine.

---

## 5. Yield в практике

### Pattern 1: Generate sequence

```csharp
public IEnumerable<int> Fibonacci()
{
    int a = 0, b = 1;
    while (true)
    {
        yield return a;
        (a, b) = (b, a + b);
    }
}

var first10Fib = Fibonacci().Take(10).ToList();
// 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
```

### Pattern 2: Parse stream / file

```csharp
public IEnumerable<LogEntry> ParseLog(string path)
{
    foreach (var line in File.ReadLines(path))
    {
        if (TryParseLine(line, out var entry))
            yield return entry;
    }
}

// Использование — даже на 100GB log file работает!
var errors = ParseLog("huge.log")
    .Where(e => e.Level == "ERROR")
    .Take(100)
    .ToList();
```

### Pattern 3: Tree traversal

```csharp
public IEnumerable<TreeNode> Traverse(TreeNode root)
{
    yield return root;

    foreach (var child in root.Children)
    {
        foreach (var descendant in Traverse(child))  // recursive!
        {
            yield return descendant;
        }
    }
}

// Альтернатива (better perf) — через Stack
public IEnumerable<TreeNode> TraverseFast(TreeNode root)
{
    var stack = new Stack<TreeNode>();
    stack.Push(root);

    while (stack.Count > 0)
    {
        var node = stack.Pop();
        yield return node;

        foreach (var child in node.Children.Reverse())
            stack.Push(child);
    }
}
```

> [!info] Recursive yield — slow
> Каждый вложенный `yield return` — `O(depth)`. Tree из 1000 levels — 1000x slower. Стек-based — `O(1)` per yield.

### Pattern 4: Pagination

```csharp
public IEnumerable<T> GetAllPages<T>(Func<int, IEnumerable<T>> getPage)
{
    int page = 0;
    while (true)
    {
        var items = getPage(page).ToList();
        if (items.Count == 0) yield break;

        foreach (var item in items)
            yield return item;

        page++;
    }
}

// Использование
var allUsers = GetAllPages(p => api.GetUsers(page: p, size: 100))
    .Take(1000)  // только первые 1000
    .ToList();
```

### Pattern 5: Sliding window

```csharp
public IEnumerable<T[]> SlidingWindow<T>(IEnumerable<T> source, int size)
{
    var window = new Queue<T>(size);

    foreach (var item in source)
    {
        window.Enqueue(item);
        if (window.Count == size)
        {
            yield return window.ToArray();
            window.Dequeue();
        }
    }
}

// 1, 2, 3, 4, 5 → [1,2,3], [2,3,4], [3,4,5]
foreach (var w in SlidingWindow(new[] { 1, 2, 3, 4, 5 }, 3))
{
    Console.WriteLine($"[{string.Join(",", w)}]");
}
```

### Pattern 6: Async streaming (IAsyncEnumerable, .NET Core 3+)

```csharp
public async IAsyncEnumerable<string> ReadLinesAsync(
    string url,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var http = new HttpClient();
    using var stream = await http.GetStreamAsync(url, ct);
    using var reader = new StreamReader(stream);

    string? line;
    while ((line = await reader.ReadLineAsync(ct)) != null)
    {
        yield return line;
    }
}

// Использование
await foreach (var line in ReadLinesAsync("https://example.com/log", cancellationToken))
{
    Console.WriteLine(line);
}
```

См. [[async-threading|Async и Threading]] для async enumerable deep.

---

## 6. Custom IEnumerable / IEnumerator

Для full control — implement интерфейсы вручную (редко нужно):

```csharp
public class MyCollection : IEnumerable<int>
{
    private readonly int[] _items;
    public MyCollection(int[] items) => _items = items;

    public IEnumerator<int> GetEnumerator() => new MyEnumerator(_items);
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();

    private class MyEnumerator : IEnumerator<int>
    {
        private readonly int[] _items;
        private int _index = -1;

        public MyEnumerator(int[] items) => _items = items;

        public int Current => _items[_index];
        object IEnumerator.Current => Current;

        public bool MoveNext()
        {
            _index++;
            return _index < _items.Length;
        }

        public void Reset() => _index = -1;
        public void Dispose() { }
    }
}
```

В 99% случаев — **используй yield**. Custom — только когда нужен fine control над state.

---

## 7. yield + try/catch/finally

### `yield` нельзя в `catch`

```csharp
public IEnumerable<int> Process()
{
    try
    {
        yield return 1;  // ✅ OK в try
    }
    catch
    {
        // yield return 2;  // ❌ Compile error!
    }
    finally
    {
        Console.WriteLine("Cleanup");  // выполняется при Dispose enumerator
    }
}
```

### `using` в iterator — disposal at completion

```csharp
public IEnumerable<string> ReadFile(string path)
{
    using var reader = new StreamReader(path);
    string? line;
    while ((line = reader.ReadLine()) != null)
    {
        yield return line;
    }
}
// reader.Dispose() вызывается:
// 1. Когда iterator complete (foreach закончен)
// 2. Когда foreach early break
// 3. Когда enumerator.Dispose() вызван
```

### Pitfall: forgot Dispose enumerator

```csharp
var iter = ReadFile("file.txt").GetEnumerator();
iter.MoveNext();  // открыл file
// забыли iter.Dispose() — file заблокирован!

// foreach автоматически Dispose enumerator
foreach (var line in ReadFile("file.txt"))
{
    if (line.Contains("X")) break;
}  // file disposed автоматически
```

---

## 8. yield в async — IAsyncEnumerable

### Async enumerable

```csharp
public async IAsyncEnumerable<int> GenerateAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; ; i++)
    {
        await Task.Delay(100, ct);
        yield return i;
    }
}

// Consume
await foreach (var item in GenerateAsync(ct))
{
    Console.WriteLine(item);
    if (item > 100) break;
}
```

### `[EnumeratorCancellation]` — важно

Атрибут указывает компилятору **который параметр** — token для async enumerator.

```csharp
public async IAsyncEnumerable<int> Gen(
    int max,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < max; i++)
    {
        ct.ThrowIfCancellationRequested();
        yield return i;
        await Task.Delay(10, ct);
    }
}

// Caller передаёт token через WithCancellation
await foreach (var i in Gen(100).WithCancellation(myToken))
{
    // ...
}
```

### `ConfigureAwait(false)`

```csharp
await foreach (var item in source.ConfigureAwait(false))
{
    // ...
}
```

См. [[async-threading#configureawait|Async — ConfigureAwait]].

---

## 9. LINQ + iterators

### Большинство LINQ — iterators

```csharp
// Все эти возвращают IEnumerable<T> через yield под капотом
items.Where(x => x > 0)
items.Select(x => x * 2)
items.Take(10)
items.Skip(5)
items.Distinct()
items.OrderBy(x => x)  // ⚠️ Это НЕ lazy полностью — должен прочитать всё

// Деferred execution — вычисление до материализации
var query = items.Where(x => Predicate(x));  // НЕ выполнено

foreach (var x in query) { }  // ← Predicate вызывается здесь
foreach (var x in query) { }  // ← Опять вызывается!

// ✅ Materialize если нужно несколько раз
var list = items.Where(...).ToList();
```

### Custom LINQ-like operator

```csharp
public static IEnumerable<T> EveryNth<T>(this IEnumerable<T> source, int n)
{
    int i = 0;
    foreach (var item in source)
    {
        if (i % n == 0) yield return item;
        i++;
    }
}

// Использование
var every3rd = Enumerable.Range(1, 20).EveryNth(3);
// 1, 4, 7, 10, 13, 16, 19
```

### Performance: yield vs ToList

```csharp
// ✅ Yield — lazy, memory-efficient
public IEnumerable<int> Squares(IEnumerable<int> nums) =>
    nums.Select(n => n * n);

// ❌ Списком — все в памяти сразу
public List<int> Squares(IEnumerable<int> nums) =>
    nums.Select(n => n * n).ToList();
```

Используй `IEnumerable<T>` в API когда возможно — caller решит когда materialize.

---

## 10. Common Pitfalls

### 1. Multiple enumeration

```csharp
public IEnumerable<int> ExpensiveOperation()
{
    Console.WriteLine("Loading...");
    return File.ReadLines("huge.txt").Select(int.Parse);
}

var data = ExpensiveOperation();
var count = data.Count();    // Loading + read
var sum = data.Sum();         // Loading + read AGAIN!

// ✅
var data = ExpensiveOperation().ToList();
var count = data.Count();    // O(1)
var sum = data.Sum();         // O(n) but no re-read
```

> [!info] ReSharper / Rider — warning IDE0058
> Reshrarper warns при "Possible multiple enumeration".

### 2. Modifying source during iteration

```csharp
List<int> list = new() { 1, 2, 3 };

// ❌ InvalidOperationException
foreach (var n in list)
{
    if (n == 2) list.Remove(n);
}

// ✅ Copy or filter
foreach (var n in list.ToList())  // copy
{
    if (n == 2) list.Remove(n);
}
```

### 3. yield в method — нельзя смешивать `return`

```csharp
public IEnumerable<int> Get(bool empty)
{
    if (empty) return [];  // ❌ Compile error — нельзя смешивать с yield

    yield return 1;
    yield return 2;
}

// ✅ Pure yield
public IEnumerable<int> Get(bool empty)
{
    if (empty) yield break;

    yield return 1;
    yield return 2;
}
```

### 4. Late binding с yield

```csharp
public IEnumerable<Func<int>> CreateClosures()
{
    for (int i = 0; i < 3; i++)
    {
        yield return () => i;  // ⚠️ Capture i by reference!
    }
}

var funcs = CreateClosures().ToList();
foreach (var f in funcs) Console.WriteLine(f());

// До C# 5: 3, 3, 3 (все captured same i)
// C# 5+: each iteration has own i — 0, 1, 2 ✅
```

С foreach в C# 5+ — каждая iteration имеет свою copy. Но в `for` loop — не!

```csharp
public IEnumerable<Func<int>> Bad()
{
    for (int i = 0; i < 3; i++)
        yield return () => i;  // ⚠️ same i
}

// Output: 3, 3, 3

// ✅ Local copy
public IEnumerable<Func<int>> Good()
{
    for (int i = 0; i < 3; i++)
    {
        int copy = i;
        yield return () => copy;
    }
}
```

### 5. Eager validation в lazy method

```csharp
public IEnumerable<int> Process(IEnumerable<int> source)
{
    if (source is null) throw new ArgumentNullException();  // ⚠️ Lazy — не throw сразу!

    foreach (var x in source)
        yield return x * 2;
}

// Caller
var result = Process(null);
// throw НЕ происходит здесь!

foreach (var x in result) { }  // ← throw здесь!
```

**Лечение:** wrap в non-iterator method:

```csharp
public IEnumerable<int> Process(IEnumerable<int> source)
{
    if (source is null) throw new ArgumentNullException();
    return ProcessImpl(source);
}

private IEnumerable<int> ProcessImpl(IEnumerable<int> source)
{
    foreach (var x in source)
        yield return x * 2;
}
```

### 6. Yield в async non-IAsyncEnumerable

```csharp
// ❌
public async Task<IEnumerable<int>> Get()
{
    yield return 1;  // ❌ Compile error
}

// ✅ IAsyncEnumerable<T>
public async IAsyncEnumerable<int> Get()
{
    yield return 1;
}
```

### 7. Performance — yield не free

```csharp
// Каждый yield return — state machine call
public IEnumerable<int> Generate1()
{
    for (int i = 0; i < 1_000_000; i++)
        yield return i;
}

// Иногда быстрее — аллоцировать список
public List<int> Generate2()
{
    var list = new List<int>(1_000_000);
    for (int i = 0; i < 1_000_000; i++)
        list.Add(i);
    return list;
}
```

Для **hot paths** — measure.

### 8. Potential infinite loop

```csharp
public IEnumerable<int> AllPositives(int start)
{
    while (true) yield return start++;
}

// ❌ Зависнет!
foreach (var n in AllPositives(1)) Console.WriteLine(n);

// ✅ С Take
foreach (var n in AllPositives(1).Take(100))
    Console.WriteLine(n);
```

---

## 11. yield vs alternatives

### yield vs Linq

```csharp
// Yield
public IEnumerable<int> EvenSquares(IEnumerable<int> nums)
{
    foreach (var n in nums)
        if (n % 2 == 0)
            yield return n * n;
}

// LINQ — уже использует yield под капотом
public IEnumerable<int> EvenSquares(IEnumerable<int> nums) =>
    nums.Where(n => n % 2 == 0).Select(n => n * n);
```

LINQ читабельнее. Yield нужен для complex логики которую сложно выразить.

### yield vs IEnumerable<T> return

```csharp
// Не yield, но возвращает IEnumerable
public IEnumerable<int> Numbers()
{
    return new[] { 1, 2, 3, 4, 5 };  // создаёт array — не lazy
}

// Yield — lazy, memory-friendly
public IEnumerable<int> Numbers()
{
    for (int i = 1; i <= 5; i++)
        yield return i;
}
```

### yield vs `IEnumerable<T>` parameters

```csharp
// API design:
public void Process(IEnumerable<int> items) { }  // accepts любой sequence

Process(new[] { 1, 2, 3 });
Process(new List<int> { 1, 2, 3 });
Process(GenerateNumbers());  // yield method
Process(query.Where(...));    // LINQ
```

`IEnumerable<T>` параметры — accepts всё. Возвращай **most specific** type, принимай **most general**.

---

## 12. Best Practices

### Использование yield

- **Lazy для больших / infinite sequences**
- **Streaming данных** (file lines, DB rows, HTTP)
- **Generator patterns** (Fibonacci, prime numbers)
- **Multi-step processing pipelines**
- **Custom LINQ operators**
- **Tree traversal** (но stack-based быстрее для глубоких)

### Возвращай IEnumerable<T>

- В API — flexibility для caller
- Caller сам решит `.ToList()`, `.ToArray()`, `Count`, etc.
- Меньше allocations если caller только iterate

### Materialize когда нужно

```csharp
// Несколько iterations — ToList
var data = source.Where(...).ToList();

// Random access по index — ToArray
var arr = source.ToArray();
var fifth = arr[4];

// Set operations — ToHashSet
var set = source.ToHashSet();
set.Contains(...);  // O(1)

// Lookup — ToDictionary / ToLookup
var byId = source.ToDictionary(x => x.Id);
```

### Avoid pitfalls

- **Не throw в iterator method** — wrap в non-iterator
- **Не modify source** during iteration
- **Don't multiple enumerate** expensive iterators
- **`[EnumeratorCancellation]`** в IAsyncEnumerable
- **`.ConfigureAwait(false)`** в library async iterators

### Performance

- **Profile** перед optimization
- **Materialize** если несколько iterations
- **Stack-based** для deep recursion
- **`Span<T>`** где допустимо для primitives

---

## 13. Cheat sheet

| Сценарий | Pattern |
|----------|---------|
| Generate sequence | `yield return value;` |
| Conditional output | `if (...) yield return value;` |
| Early exit | `yield break;` |
| Recursive (tree) | `foreach (...) yield return ...;` |
| Async streaming | `async IAsyncEnumerable<T>` + `[EnumeratorCancellation]` |
| Custom LINQ operator | extension method с `yield` |
| Cancellable | `ct.ThrowIfCancellationRequested()` |
| Resource cleanup | `using` в iterator method |
| Avoid multiple enum | `.ToList()` / `.ToArray()` |
| Validate early | wrapper method + iterator method |

---

## 14. Decision tree

```
Нужна последовательность значений?
│
├── Известное маленькое количество (<100)?
│   → array / List
│
├── Большое / infinite?
│   → yield return
│
├── Async I/O?
│   → IAsyncEnumerable + yield
│
├── Streaming из file / DB / HTTP?
│   → yield (lazy reading)
│
├── Простая трансформация?
│   → LINQ (Select / Where / Take)
│
├── Multiple iterations нужны?
│   → ToList() после yield/LINQ
│
└── Random access?
    → array / List, не yield
```

---

## См. также

- [[csharp-basics|C# Basics]] — foreach intro
- [[collections-linq|Collections и LINQ]] — LINQ построен на iterators
- [[async-threading|Async и Threading]] — IAsyncEnumerable
- [[functional-csharp|Functional C#]] — pipeline patterns
- [[io-streams|I/O и Streams]] — streaming files
- [[../Runtime/span-layout|Span и Layout]] — performance alternatives

## Reading list

- **Microsoft Docs — Iterators** — learn.microsoft.com/dotnet/csharp/iterators
- **Microsoft Docs — IAsyncEnumerable** — learn.microsoft.com/dotnet/api/system.collections.generic.iasyncenumerable-1
- **Stephen Toub — IAsyncEnumerable in C# 8** — devblogs.microsoft.com/dotnet
- **Jon Skeet — yield internals** — codeblog.jonskeet.uk
- **Eric Lippert — iterator blocks series** — ericlippert.com
- **C# in Depth** — Jon Skeet (chapter про iterators)
