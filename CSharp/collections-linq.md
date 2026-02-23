---
tags: [collections, linq, dictionary, hashset, generics]
level: Senior
---

# Collections и LINQ

> Справочник по коллекциям, LINQ и generics. C# 13 / .NET 9.
> Теория → практика → senior-level код → вопросы интервью.

---

## Обзор коллекций

### Иерархия интерфейсов

```
IEnumerable<T>          — только перечисление (foreach, yield)
  └─ ICollection<T>     — Count, Add, Remove, Contains
       └─ IList<T>      — индексатор [i], Insert, RemoveAt
       └─ ISet<T>       — операции над множествами
  └─ IReadOnlyCollection<T>
       └─ IReadOnlyList<T>
       └─ IReadOnlySet<T>
       └─ IReadOnlyDictionary<TKey, TValue>
```

```csharp
// IEnumerable<T> — минимальный контракт, ленивое перечисление
IEnumerable<int> LazyRange(int count)
{
    for (var i = 0; i < count; i++)
        yield return i;
}

// ICollection<T> — когда нужно знать Count и изменять коллекцию
void Process(ICollection<string> items)
{
    Console.WriteLine(items.Count);
    items.Add("new");
    items.Remove("old");
}

// IList<T> — когда нужен доступ по индексу
void ProcessList(IList<Order> orders)
{
    var first = orders[0];
    orders.Insert(0, new Order());
    orders.RemoveAt(orders.Count - 1);
}
```

**Правило выбора параметра метода:** принимай самый узкий интерфейс, который тебе нужен. Если достаточно перечислить — `IEnumerable<T>`. Нужен Count — `IReadOnlyCollection<T>`. Нужен индекс — `IReadOnlyList<T>`.

### IEnumerable vs IQueryable — ключевое различие

```csharp
// IEnumerable<T> — выполнение in-memory (LINQ to Objects)
// Делегаты: Func<T, bool>
IEnumerable<Order> orders = dbContext.Orders.AsEnumerable();
var filtered = orders.Where(o => o.Total > 1000); // фильтрация в памяти C#

// IQueryable<T> — трансляция в провайдер (SQL, MongoDB и т.д.)
// Expression trees: Expression<Func<T, bool>>
IQueryable<Order> query = dbContext.Orders;
var filtered2 = query.Where(o => o.Total > 1000); // транслируется в SQL: WHERE Total > 1000
```

> **Критично:** Не вызывай `.AsEnumerable()` или `.ToList()` до финальной фильтрации — иначе вся таблица загрузится в память.

> [!question]- **Интервью: IEnumerable vs IQueryable — когда что?**
> **IEnumerable** — выполнение в памяти (LINQ to Objects). Фильтрация на стороне приложения.
>
> **IQueryable** — дерево выражений → SQL. Выполнение на сервере БД. Используй с EF Core.
>
> **Правило:** не возвращай `IQueryable` из repository наружу — утечка абстракции. Материализуй через `ToListAsync()`.

---

## Generic Collections

### List\<T\> — основная коллекция

Внутри — массив `T[]`. При переполнении capacity удваивается.

```csharp
// Создание
var numbers = new List<int> { 1, 2, 3 };
var empty = new List<string>(capacity: 100); // предвыделение — меньше аллокаций
List<int> fromRange = [1, 2, 3, 4, 5]; // collection expression (C# 12)

// Основные методы
numbers.Add(4);
numbers.AddRange([5, 6, 7]);
numbers.Insert(0, 0);              // вставка по индексу — O(n)
numbers.Remove(3);                 // удаление первого вхождения — O(n)
numbers.RemoveAt(0);               // удаление по индексу — O(n)
numbers.RemoveAll(x => x % 2 == 0); // удаление по предикату

// Поиск
var idx = numbers.IndexOf(5);
var found = numbers.Find(x => x > 3);         // первый подходящий или default
var all = numbers.FindAll(x => x > 3);        // все подходящие
bool exists = numbers.Exists(x => x == 5);

// Сортировка
numbers.Sort();                                // in-place, Span-based (быстро)
numbers.Sort((a, b) => b.CompareTo(a));       // по убыванию
numbers.Sort(Comparer<int>.Create((a, b) => a - b));

// Конвертация
int[] array = numbers.ToArray();
ReadOnlyCollection<int> ro = numbers.AsReadOnly();

// Capacity vs Count
Console.WriteLine($"Count: {numbers.Count}, Capacity: {numbers.Capacity}");
numbers.TrimExcess(); // уменьшить Capacity до Count
```

**Когда использовать:** в 90% случаев. Быстрый доступ по индексу O(1), добавление в конец O(1) amortized.

### Dictionary\<TKey, TValue\> — хеширование

```csharp
// Создание
var dict = new Dictionary<string, int>
{
    ["apple"] = 5,
    ["banana"] = 3,
    ["cherry"] = 8
};

// Безопасное чтение — TryGetValue (предпочтительный способ)
if (dict.TryGetValue("apple", out var count))
    Console.WriteLine($"Apple: {count}");

// GetValueOrDefault (.NET Core 2.0+) — удобно для value types
int qty = dict.GetValueOrDefault("mango", defaultValue: 0);

// CollectionsMarshal.GetValueRefOrAddDefault — zero-copy доступ
ref var val = ref System.Runtime.InteropServices.CollectionsMarshal
    .GetValueRefOrAddDefault(dict, "grape", out bool existed);
if (!existed) val = 10;
val += 5; // изменяем значение in-place без повторного lookup

// Добавление
dict.Add("date", 2);                // ArgumentException если ключ уже есть
dict.TryAdd("date", 99);            // false если ключ есть, без исключений
dict["elderberry"] = 1;             // добавит или перезапишет

// Удаление
dict.Remove("banana");
dict.Remove("banana", out var removed); // получить удалённое значение

// Перебор
foreach (var (key, value) in dict)
    Console.WriteLine($"{key}: {value}");

// Создание из LINQ
var lookup = new[] { "a", "bb", "ccc", "dd" }
    .ToDictionary(s => s, s => s.Length);
```

**Сложность:** Get/Set/Add/Remove — O(1) amortized. Зависит от качества `GetHashCode()`.

> [!question]- **Интервью: Как работает Dictionary? GetHashCode + Equals?**
> Dictionary использует хеш для выбора bucket-а. `GetHashCode()` → bucket, `Equals()` → разрешение коллизий внутри bucket-а.
>
> **Контракт:** если `Equals(a,b) == true`, то `GetHashCode(a) == GetHashCode(b)`. Нарушение → потеря элементов. Плохое распределение → деградация O(1) до O(n).
>
> **Практика:** всегда переопределяй оба метода вместе. `HashCode.Combine()`. Не мутабельные поля в ключах.

### HashSet\<T\> — уникальные элементы

```csharp
var set = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
{
    "Alice", "Bob", "Charlie"
};

// Добавление — вернёт false если элемент уже есть
bool added = set.Add("alice"); // false — OrdinalIgnoreCase

// Проверка
bool contains = set.Contains("BOB"); // true

// Операции над множествами
var setA = new HashSet<int> { 1, 2, 3, 4, 5 };
var setB = new HashSet<int> { 3, 4, 5, 6, 7 };

setA.UnionWith(setB);          // {1,2,3,4,5,6,7} — объединение
setA.IntersectWith(setB);      // {3,4,5} — пересечение
setA.ExceptWith(setB);         // {1,2} — разность
setA.SymmetricExceptWith(setB);// {1,2,6,7} — симметрическая разность

bool isSubset = setA.IsSubsetOf(setB);
bool overlaps = setA.Overlaps(setB);
bool equal = setA.SetEquals(setB);
```

### Queue\<T\> и Stack\<T\>

```csharp
// Queue — FIFO (первый вошёл — первый вышел)
var queue = new Queue<string>();
queue.Enqueue("first");
queue.Enqueue("second");
queue.Enqueue("third");

string next = queue.Dequeue();    // "first"
string peek = queue.Peek();       // "second" (без удаления)
bool has = queue.TryDequeue(out var item); // безопасный вариант

// Stack — LIFO (последний вошёл — первый вышел)
var stack = new Stack<int>();
stack.Push(1);
stack.Push(2);
stack.Push(3);

int top = stack.Pop();           // 3
int peekTop = stack.Peek();      // 2
bool popped = stack.TryPop(out var val); // безопасный вариант
```

### LinkedList\<T\> — двусвязный список

```csharp
var list = new LinkedList<string>();

var node1 = list.AddFirst("A");
var node2 = list.AddLast("C");
var node3 = list.AddAfter(node1, "B"); // A -> B -> C

// Навигация
LinkedListNode<string>? current = list.First;
while (current is not null)
{
    Console.Write($"{current.Value} -> ");
    current = current.Next;
}

// Удаление — O(1) если есть ссылка на узел
list.Remove(node3);
```

**Когда использовать:** частые вставки/удаления в середине. На практике — почти никогда. `List<T>` быстрее из-за cache locality.

### SortedSet\<T\>, SortedDictionary\<TKey, TValue\>

```csharp
// SortedSet — всегда отсортированный набор уникальных элементов (Red-Black Tree)
var sorted = new SortedSet<int> { 5, 3, 8, 1, 9 };
// Перебор: 1, 3, 5, 8, 9

var range = sorted.GetViewBetween(3, 8); // {3, 5, 8} — подмножество
int min = sorted.Min; // 1
int max = sorted.Max; // 9

// SortedDictionary — ключи всегда отсортированы (Red-Black Tree)
var sortedDict = new SortedDictionary<string, int>
{
    ["banana"] = 3,
    ["apple"] = 5,
    ["cherry"] = 1
};
// Перебор: apple -> banana -> cherry (алфавитный порядок ключей)
foreach (var (key, value) in sortedDict)
    Console.WriteLine($"{key}: {value}");
```

**Сложность:** все операции O(log n). Используй когда нужен постоянный отсортированный порядок.

### PriorityQueue\<TElement, TPriority\> (.NET 6)

```csharp
var pq = new PriorityQueue<string, int>();

pq.Enqueue("Low priority task", priority: 10);
pq.Enqueue("Critical task", priority: 1);
pq.Enqueue("Medium task", priority: 5);

// Извлечение — всегда элемент с наименьшим priority
while (pq.TryDequeue(out var task, out var priority))
    Console.WriteLine($"[{priority}] {task}");
// [1] Critical task
// [5] Medium task
// [10] Low priority task

// EnqueueDequeue — атомарно: добавить и сразу извлечь минимальный
var dequeued = pq.EnqueueDequeue("New task", 3);
```

---

## Concurrent Collections

### ConcurrentDictionary\<TKey, TValue\>

```csharp
var cache = new ConcurrentDictionary<string, int>();

// Потокобезопасные операции
cache.TryAdd("key", 1);
cache.TryRemove("key", out var removed);

// GetOrAdd — добавить если нет (factory может вызваться несколько раз!)
var value = cache.GetOrAdd("counter", key => ExpensiveComputation(key));

// AddOrUpdate — атомарное обновление
cache.AddOrUpdate(
    key: "counter",
    addValue: 1,
    updateValueFactory: (key, oldValue) => oldValue + 1);

// ВАЖНО: factory вызовы НЕ атомарны — только запись атомарна
// Для дорогих factory используй Lazy<T>:
var lazyCache = new ConcurrentDictionary<string, Lazy<int>>();
var result = lazyCache.GetOrAdd("key", k => new Lazy<int>(() => ExpensiveComputation(k)));
Console.WriteLine(result.Value);
```

### ConcurrentQueue\<T\>, ConcurrentBag\<T\>

```csharp
// ConcurrentQueue — потокобезопасный FIFO
var cq = new ConcurrentQueue<WorkItem>();
cq.Enqueue(new WorkItem("task1"));
if (cq.TryDequeue(out var item))
    Process(item);

// ConcurrentBag — неупорядоченная потокобезопасная коллекция
// Оптимизирована для сценария: каждый поток добавляет и забирает свои элементы
var bag = new ConcurrentBag<int>();
Parallel.For(0, 100, i => bag.Add(i));
Console.WriteLine(bag.Count); // 100
```

### BlockingCollection\<T\>

```csharp
// Producer-consumer с ограниченной ёмкостью
using var bc = new BlockingCollection<string>(boundedCapacity: 10);

// Producer (в другом потоке)
Task.Run(() =>
{
    for (var i = 0; i < 50; i++)
    {
        bc.Add($"item-{i}"); // блокируется если коллекция полная
    }
    bc.CompleteAdding(); // сигнал: больше не будет элементов
});

// Consumer — GetConsumingEnumerable блокирует до CompleteAdding
foreach (var item in bc.GetConsumingEnumerable())
    Console.WriteLine(item);
```

### Channel\<T\> — modern async producer/consumer

```csharp
// Предпочтительный вариант для async кода (вместо BlockingCollection)
var channel = Channel.CreateBounded<Order>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleReader = false,
    SingleWriter = false
});

// Producer
async Task ProduceAsync(ChannelWriter<Order> writer, CancellationToken ct)
{
    try
    {
        while (!ct.IsCancellationRequested)
        {
            var order = await GetNextOrderAsync(ct).ConfigureAwait(false);
            await writer.WriteAsync(order, ct).ConfigureAwait(false);
        }
    }
    finally
    {
        writer.Complete();
    }
}

// Consumer
async Task ConsumeAsync(ChannelReader<Order> reader, CancellationToken ct)
{
    await foreach (var order in reader.ReadAllAsync(ct).ConfigureAwait(false))
    {
        await ProcessOrderAsync(order, ct).ConfigureAwait(false);
    }
}

// Запуск
var cts = new CancellationTokenSource();
await Task.WhenAll(
    ProduceAsync(channel.Writer, cts.Token),
    ConsumeAsync(channel.Reader, cts.Token));
```

### Immutable Collections

```csharp
using System.Collections.Immutable;

// Каждая операция возвращает НОВУЮ коллекцию — старая не меняется
var list = ImmutableList<int>.Empty;
var list2 = list.Add(1).Add(2).Add(3);
var list3 = list2.RemoveAt(0); // [2, 3]
// list2 по-прежнему [1, 2, 3]

// Builder — для массовых мутаций (эффективнее чем цепочка Add)
var builder = ImmutableList.CreateBuilder<string>();
builder.Add("A");
builder.Add("B");
builder.Add("C");
ImmutableList<string> immutable = builder.ToImmutable();

// ImmutableDictionary
var dict = ImmutableDictionary<string, int>.Empty
    .Add("x", 1)
    .Add("y", 2)
    .SetItem("x", 10); // перезапись — новый словарь

// ImmutableArray — наиболее эффективная immutable коллекция
var arr = ImmutableArray.Create(1, 2, 3);
var arr2 = arr.Add(4); // [1, 2, 3, 4]
```

---

## Frozen Collections (.NET 8)

Оптимизированы для **чтения**. Создаются один раз — потом только читаются. Быстрее Dictionary/HashSet для lookup.

```csharp
using System.Collections.Frozen;

// Создание — из существующей коллекции
var source = new Dictionary<string, int>
{
    ["GET"] = 1,
    ["POST"] = 2,
    ["PUT"] = 3,
    ["DELETE"] = 4
};

FrozenDictionary<string, int> frozen = source.ToFrozenDictionary();
FrozenDictionary<string, int> frozenOrdinal = source
    .ToFrozenDictionary(StringComparer.OrdinalIgnoreCase);

// Использование — обычный API словаря
var val = frozen["GET"];
bool found = frozen.TryGetValue("POST", out var v);

// FrozenSet
FrozenSet<string> allowedMethods = new[] { "GET", "POST", "PUT", "DELETE" }
    .ToFrozenSet(StringComparer.OrdinalIgnoreCase);

bool allowed = allowedMethods.Contains("get"); // true
```

**Когда использовать:**
- Конфигурация, справочники, маппинги, которые не меняются после старта
- HTTP method routing, enum-to-string маппинги
- Дороже создать, дешевле читать — идеально для hot path

---

## Collection Expressions (C# 12)

```csharp
// Литералы коллекций — единый синтаксис для всех типов
int[] array = [1, 2, 3];
List<int> list = [1, 2, 3];
Span<int> span = [1, 2, 3];
ReadOnlySpan<int> roSpan = [1, 2, 3];
ImmutableArray<int> immArr = [1, 2, 3];
HashSet<int> set = [1, 2, 3];

// Пустая коллекция
List<string> empty = [];

// Spread operator — объединение коллекций
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];
int[] combined = [..first, ..second]; // [1, 2, 3, 4, 5, 6]

// Spread с фильтрацией
int[] extras = [10, 20];
int[] all = [0, ..first, ..extras, 99]; // [0, 1, 2, 3, 10, 20, 99]

// В параметрах метода
PrintAll([1, 2, 3]);

void PrintAll(IEnumerable<int> items)
{
    foreach (var item in items)
        Console.Write($"{item} ");
}

// Target-typed new — компилятор определяет тип
Dictionary<string, List<int>> map = new()
{
    ["odds"] = [1, 3, 5],
    ["evens"] = [2, 4, 6]
};
```

---

## LINQ — Основы

### Method syntax vs Query syntax

```csharp
var products = GetProducts();

// Method syntax (предпочтительный — более гибкий)
var expensive = products
    .Where(p => p.Price > 100)
    .OrderByDescending(p => p.Price)
    .Select(p => new { p.Name, p.Price });

// Query syntax (удобнее для join и let)
var expensiveQuery =
    from p in products
    where p.Price > 100
    orderby p.Price descending
    select new { p.Name, p.Price };

// Query syntax с let — промежуточная переменная
var discounted =
    from p in products
    let discountPrice = p.Price * 0.9m
    where discountPrice > 50
    select new { p.Name, Original = p.Price, Discounted = discountPrice };
```

### Deferred Execution (отложенное выполнение)

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// Запрос НЕ выполняется здесь — это описание операции
var query = numbers.Where(n => n > 2);

numbers.Add(6); // добавляем элемент ПОСЛЕ создания запроса

// Выполнение происходит ЗДЕСЬ при итерации
foreach (var n in query)
    Console.Write($"{n} "); // 3 4 5 6 — включая 6!

// ОПАСНОСТЬ: многократное перечисление
var filtered = numbers.Where(n => ExpensiveCheck(n));
var count = filtered.Count();    // 1-я итерация
var first = filtered.First();    // 2-я итерация — ExpensiveCheck вызовется снова!

// РЕШЕНИЕ: материализация
var materialized = numbers.Where(n => ExpensiveCheck(n)).ToList();
var count2 = materialized.Count;   // без повторного вычисления
var first2 = materialized[0];
```

> [!question]- **Интервью: yield return — как работает и ограничения?**
> Компилятор генерирует state machine. При каждом `MoveNext()` — продолжение с точки последнего `yield return`. Элементы создаются лениво.
>
> **Ограничения:** нельзя `yield return` внутри `try` с `catch`. Нельзя `ref`/`out` параметры. `finally` выполняется при `Dispose` перечислителя.
>
> **Когда:** стриминг больших данных, бесконечные последовательности, кастомная логика перечисления.

### Immediate Execution (немедленное выполнение)

```csharp
var items = Enumerable.Range(1, 100);

// Материализация в конкретную коллекцию
List<int> list = items.ToList();
int[] array = items.ToArray();
Dictionary<int, string> dict = items.ToDictionary(i => i, i => $"item-{i}");
HashSet<int> set = items.ToHashSet();
ILookup<bool, int> lookup = items.ToLookup(i => i % 2 == 0); // группировка

// Агрегация — тоже немедленное
int count = items.Count();
int sum = items.Sum();
int max = items.Max();
```

---

## LINQ — Операторы

### Фильтрация

```csharp
var people = GetPeople();

// Where — фильтрация по предикату
var adults = people.Where(p => p.Age >= 18);

// Where с индексом
var firstThreeExpensive = products
    .Where((p, index) => p.Price > 100 && index < 3);

// OfType — фильтрация по типу (безопасный cast)
object[] mixed = [1, "hello", 2.5, "world", 42];
IEnumerable<string> strings = mixed.OfType<string>(); // "hello", "world"

// OfType полезен для иерархий типов
var animals = GetAnimals();
var dogs = animals.OfType<Dog>(); // только Dog, без InvalidCastException
```

### Проекция

```csharp
// Select — трансформация каждого элемента
var names = people.Select(p => p.Name);
var dtos = people.Select(p => new PersonDto
{
    FullName = $"{p.FirstName} {p.LastName}",
    Age = p.Age
});

// Select с индексом
var indexed = people.Select((p, i) => new { Index = i, p.Name });

// SelectMany — flatten вложенных коллекций
var departments = GetDepartments(); // каждый отдел содержит List<Employee>

// Без SelectMany:
IEnumerable<IEnumerable<Employee>> nested = departments.Select(d => d.Employees);

// С SelectMany — плоский список всех сотрудников:
IEnumerable<Employee> allEmployees = departments.SelectMany(d => d.Employees);

// SelectMany с результирующим селектором
var empWithDept = departments.SelectMany(
    d => d.Employees,
    (dept, emp) => new { dept.Name, Employee = emp.FullName });

// Пример: разбить строки на слова
string[] sentences = ["Hello world", "Foo bar baz"];
var words = sentences.SelectMany(s => s.Split(' ')); // ["Hello","world","Foo","bar","baz"]
```

### Сортировка

```csharp
var orders = GetOrders();

// OrderBy / OrderByDescending
var byDate = orders.OrderBy(o => o.CreatedAt);
var byDateDesc = orders.OrderByDescending(o => o.CreatedAt);

// ThenBy / ThenByDescending — вторичная сортировка
var sorted = orders
    .OrderBy(o => o.Status)
    .ThenByDescending(o => o.Total)
    .ThenBy(o => o.CustomerName);

// Order / OrderDescending (.NET 7) — без keySelector, для простых типов
int[] nums = [5, 3, 8, 1, 9];
var asc = nums.Order();           // [1, 3, 5, 8, 9]
var desc = nums.OrderDescending(); // [9, 8, 5, 3, 1]

// Reverse
var reversed = orders.OrderBy(o => o.Id).Reverse();
```

### Группировка

```csharp
var transactions = GetTransactions();

// GroupBy — группировка по ключу
var byCategory = transactions.GroupBy(t => t.Category);

foreach (var group in byCategory)
{
    Console.WriteLine($"Category: {group.Key}, Count: {group.Count()}");
    foreach (var tx in group)
        Console.WriteLine($"  {tx.Description}: {tx.Amount:C}");
}

// GroupBy с результирующим селектором
var summary = transactions.GroupBy(
    t => t.Category,
    (category, items) => new
    {
        Category = category,
        Total = items.Sum(i => i.Amount),
        Count = items.Count()
    });

// GroupBy с elementSelector
var namesByCity = people.GroupBy(
    p => p.City,
    p => p.Name); // IGrouping<string, string>

// ToLookup — немедленная материализация группировки
ILookup<string, Transaction> lookup = transactions.ToLookup(t => t.Category);
var foodTx = lookup["Food"]; // пустая коллекция если ключа нет (не исключение!)

// CountBy (.NET 9) — подсчёт по ключу
IEnumerable<KeyValuePair<string, int>> counts = transactions.CountBy(t => t.Category);
foreach (var (category, count) in counts)
    Console.WriteLine($"{category}: {count}");

// AggregateBy (.NET 9) — агрегация по ключу
var totalsByCategory = transactions.AggregateBy(
    t => t.Category,
    seed: 0m,
    (total, tx) => total + tx.Amount);
```

### Соединение

```csharp
var customers = GetCustomers();
var orders = GetOrders();

// Join — inner join
var customerOrders = customers.Join(
    orders,
    c => c.Id,          // outer key
    o => o.CustomerId,   // inner key
    (c, o) => new { Customer = c.Name, o.Total, o.CreatedAt });

// То же самое в query syntax (читабельнее для join)
var customerOrdersQuery =
    from c in customers
    join o in orders on c.Id equals o.CustomerId
    select new { Customer = c.Name, o.Total, o.CreatedAt };

// GroupJoin — left join (один-ко-многим)
var customersWithOrders = customers.GroupJoin(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (c, orderGroup) => new
    {
        Customer = c.Name,
        OrderCount = orderGroup.Count(),
        TotalSpent = orderGroup.Sum(o => o.Total)
    });

// Left join через query syntax + DefaultIfEmpty
var leftJoin =
    from c in customers
    join o in orders on c.Id equals o.CustomerId into orderGroup
    from o in orderGroup.DefaultIfEmpty()
    select new
    {
        Customer = c.Name,
        OrderTotal = o?.Total ?? 0
    };

// Zip — соединение по позиции
var names2 = new[] { "Alice", "Bob", "Charlie" };
var scores = new[] { 95, 87, 92 };
var zipped = names2.Zip(scores, (name, score) => $"{name}: {score}");
// "Alice: 95", "Bob: 87", "Charlie: 92"

// Zip без selector (.NET Core 3.0+) — возвращает ValueTuple
var tuples = names2.Zip(scores); // (Alice, 95), (Bob, 87), (Charlie, 92)
```

### Агрегация

```csharp
var values = new[] { 10, 20, 30, 40, 50 };

int count = values.Count();
int countFiltered = values.Count(v => v > 25); // 3
long longCount = values.LongCount();

int sum = values.Sum();
double avg = values.Average();
int min = values.Min();
int max = values.Max();

// MinBy / MaxBy (.NET 6) — возвращает элемент, а не значение ключа
var cheapest = products.MinBy(p => p.Price);   // Product, не decimal
var mostExpensive = products.MaxBy(p => p.Price);

// Aggregate — универсальная свёртка (reduce/fold)
int product = values.Aggregate((acc, x) => acc * x); // 10*20*30*40*50

// Aggregate с seed
string csv = values.Aggregate(
    seed: new StringBuilder(),
    (sb, val) => sb.Length == 0 ? sb.Append(val) : sb.Append(',').Append(val),
    sb => sb.ToString()); // "10,20,30,40,50"
```

### Элемент

```csharp
var items = new[] { 10, 20, 30, 40, 50 };

// First — первый элемент (InvalidOperationException если пусто)
int first = items.First();
int firstOver30 = items.First(x => x > 30); // 40

// FirstOrDefault — default(T) если пусто
int firstOrZero = items.FirstOrDefault(x => x > 100); // 0
int firstOrNeg = items.FirstOrDefault(x => x > 100, defaultValue: -1); // -1 (.NET 6)

// Single — ровно один элемент (исключение если 0 или >1)
int single = items.Where(x => x == 30).Single();

// SingleOrDefault — 0 или 1 элемент (исключение если >1)
int singleOr = items.SingleOrDefault(x => x == 999); // 0

// Last / LastOrDefault
int last = items.Last(); // 50
int lastSmall = items.LastOrDefault(x => x < 25); // 20

// ElementAt / ElementAtOrDefault
int third = items.ElementAt(2);          // 30
int outOfRange = items.ElementAtOrDefault(100); // 0

// Index (.NET 9) — с поддержкой ^from-end
int fromEnd = items.ElementAt(^1); // 50 (последний)
```

**Когда что использовать:**
| Метод | Ожидание | Пусто | Много |
|-------|----------|-------|-------|
| `First` | >= 1 элемент | Exception | OK (берёт первый) |
| `Single` | ровно 1 | Exception | Exception |
| `FirstOrDefault` | 0 или более | default | OK |
| `SingleOrDefault` | 0 или 1 | default | Exception |

### Количество и проверка

```csharp
var orders = GetOrders();

// Any — есть ли хотя бы один элемент (эффективнее Count() > 0)
bool hasOrders = orders.Any();
bool hasExpensive = orders.Any(o => o.Total > 1000);

// All — все ли элементы удовлетворяют условию
bool allPaid = orders.All(o => o.IsPaid);

// Contains — содержит ли элемент
bool has42 = new[] { 1, 2, 42, 100 }.Contains(42);

// АНТИПАТТЕРН:
if (orders.Count() > 0) { } // плохо — перечисляет всю коллекцию
if (orders.Any()) { }       // хорошо — останавливается на первом элементе
```

### Множества

```csharp
var a = new[] { 1, 2, 3, 4, 5 };
var b = new[] { 3, 4, 5, 6, 7 };

// Distinct — уникальные элементы
int[] unique = new[] { 1, 1, 2, 2, 3 }.Distinct().ToArray(); // [1, 2, 3]

// DistinctBy (.NET 6) — уникальные по ключу
var uniqueByCity = people.DistinctBy(p => p.City);

// Union — объединение (уникальные из обеих)
var union = a.Union(b); // [1, 2, 3, 4, 5, 6, 7]

// Intersect — пересечение
var intersect = a.Intersect(b); // [3, 4, 5]

// Except — разность (в a, но не в b)
var except = a.Except(b); // [1, 2]

// *By варианты (.NET 6) — сравнение по ключу
var exceptByCity = peopleA.ExceptBy(
    peopleB.Select(p => p.City),
    p => p.City);

var intersectByAge = peopleA.IntersectBy(
    peopleB.Select(p => p.Age),
    p => p.Age);

var unionByName = peopleA.UnionBy(peopleB, p => p.Name);
```

### Разделение

```csharp
var items = Enumerable.Range(1, 20);

// Take / Skip
var firstFive = items.Take(5);       // [1..5]
var afterFive = items.Skip(5);       // [6..20]

// TakeLast / SkipLast
var lastThree = items.TakeLast(3);   // [18, 19, 20]
var withoutLast = items.SkipLast(3); // [1..17]

// Take с Range (C# / .NET 6)
var slice = items.Take(2..5);        // [3, 4, 5]
var fromEnd = items.Take(^3..);      // [18, 19, 20]

// TakeWhile / SkipWhile — по условию
var takeWhile = items.TakeWhile(x => x < 5);  // [1, 2, 3, 4]
var skipWhile = items.SkipWhile(x => x < 5);  // [5, 6, ..., 20]

// Chunk (.NET 6) — разбиение на батчи
int[][] chunks = items.Chunk(4).ToArray();
// [[1,2,3,4], [5,6,7,8], [9,10,11,12], [13,14,15,16], [17,18,19,20]]
```

### Новые операторы .NET 6–9

```csharp
// Index (.NET 9) — добавляет индекс к каждому элементу
foreach (var (index, item) in products.Index())
    Console.WriteLine($"[{index}] {item.Name}");

// CountBy (.NET 9) — подсчёт по ключу (замена GroupBy + Count)
var wordFreq = words.CountBy(w => w);
// KeyValuePair<string, int>: ("hello", 3), ("world", 2)

// AggregateBy (.NET 9) — агрегация по ключу (замена GroupBy + Aggregate)
var totalByDept = employees.AggregateBy(
    e => e.Department,
    seed: 0m,
    (total, e) => total + e.Salary);

// MinBy / MaxBy (.NET 6)
var youngest = people.MinBy(p => p.Age);
var oldest = people.MaxBy(p => p.Age);

// DistinctBy (.NET 6)
var onePerCountry = people.DistinctBy(p => p.Country);

// Order / OrderDescending (.NET 7) — без keySelector
var sorted = numbers.Order();          // вместо OrderBy(x => x)
var sortedDesc = numbers.OrderDescending();
```

---

## Expression Trees

### Что такое Expression Tree

```csharp
using System.Linq.Expressions;

// Lambda — скомпилированный делегат (IL код)
Func<int, bool> lambda = x => x > 5;

// Expression — описание lambda как структуры данных (AST)
Expression<Func<int, bool>> expression = x => x > 5;

// Expression tree — дерево: GreaterThan(Parameter(x), Constant(5))
// EF Core анализирует это дерево и генерирует SQL:
// WHERE x > 5
```

### Зачем нужны (EF Core, dynamic queries)

```csharp
// EF Core использует Expression<Func<T, bool>> для генерации SQL
IQueryable<Order> query = dbContext.Orders;

// Это выражение транслируется в SQL
Expression<Func<Order, bool>> filter = o => o.Total > 100 && o.Status == "Active";
var filtered = query.Where(filter); // SELECT * FROM Orders WHERE Total > 100 AND Status = 'Active'

// Динамическое построение Expression Trees
static Expression<Func<T, bool>> BuildFilter<T>(string propertyName, object value)
{
    var param = Expression.Parameter(typeof(T), "x");
    var property = Expression.Property(param, propertyName);
    var constant = Expression.Constant(value);
    var equal = Expression.Equal(property, constant);
    return Expression.Lambda<Func<T, bool>>(equal, param);
}

// Использование
var filter2 = BuildFilter<Order>("Status", "Active");
var activeOrders = dbContext.Orders.Where(filter2).ToList();

// Комбинирование фильтров
static Expression<Func<T, bool>> And<T>(
    Expression<Func<T, bool>> left,
    Expression<Func<T, bool>> right)
{
    var param = Expression.Parameter(typeof(T), "x");
    var body = Expression.AndAlso(
        Expression.Invoke(left, param),
        Expression.Invoke(right, param));
    return Expression.Lambda<Func<T, bool>>(body, param);
}
```

> [!question]- **Интервью: Expression Trees — зачем и как EF их использует?**
> Expression Tree — представление кода как структуры данных (AST). EF Core анализирует дерево выражений LINQ и транслирует в SQL.
>
> `IQueryable<T>` работает с `Expression<Func<T, bool>>`, а не `Func<T, bool>`. Expression — данные (можно анализировать), Func — скомпилированный код.

### Компиляция Expression в делегат

```csharp
Expression<Func<int, int, int>> addExpr = (a, b) => a + b;

// Компиляция в исполняемый делегат
Func<int, int, int> addFunc = addExpr.Compile();

int result = addFunc(3, 4); // 7

// Полезно: кешировать скомпилированные выражения
private static readonly Func<Order, bool> _isActiveCompiled =
    ((Expression<Func<Order, bool>>)(o => o.Status == "Active")).Compile();
```

---

## Generics

### Generic классы, методы, интерфейсы

```csharp
// Generic класс
public sealed class Result<T>
{
    public T? Value { get; }
    public string? Error { get; }
    public bool IsSuccess => Error is null;

    private Result(T? value, string? error) => (Value, Error) = (value, error);

    public static Result<T> Success(T value) => new(value, null);
    public static Result<T> Failure(string error) => new(default, error);

    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<string, TOut> onFailure) =>
        IsSuccess ? onSuccess(Value!) : onFailure(Error!);
}

// Использование
var result = Result<int>.Success(42);
var message = result.Match(
    v => $"Value: {v}",
    e => $"Error: {e}");

// Generic метод
public static T Max<T>(T a, T b) where T : IComparable<T> =>
    a.CompareTo(b) >= 0 ? a : b;

var maxInt = Max(10, 20);      // 20
var maxStr = Max("abc", "xyz"); // "xyz"

// Generic интерфейс
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(T entity, CancellationToken ct = default);
}

// Реализация
public sealed class OrderRepository(AppDbContext db) : IRepository<Order>
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default) =>
        await db.Orders.FindAsync([id], ct).ConfigureAwait(false);

    // ... остальные методы
}
```

### Generic Constraints

```csharp
// where T : class — только reference types
public void Process<T>(T item) where T : class
{
    // item может быть null (если не добавить notnull)
}

// where T : struct — только value types (non-nullable)
public T ParseOrDefault<T>(string input) where T : struct
{
    // T гарантированно value type
    return default; // никогда не null
}

// where T : new() — должен иметь parameterless constructor
public T CreateNew<T>() where T : new() => new T();

// where T : notnull — не может быть null (reference или value)
public void Store<T>(T item) where T : notnull
{
    var key = item.GetHashCode(); // безопасно — item не null
}

// where T : unmanaged — blittable value type (для interop, Span)
public unsafe void WriteToBuffer<T>(T value, byte* buffer) where T : unmanaged
{
    *(T*)buffer = value;
}

// Комбинирование constraints
public sealed class Cache<TKey, TValue>
    where TKey : notnull, IEquatable<TKey>
    where TValue : class, new()
{
    private readonly Dictionary<TKey, TValue> _store = new();

    public TValue GetOrCreate(TKey key)
    {
        if (_store.TryGetValue(key, out var existing))
            return existing;

        var newItem = new TValue();
        _store[key] = newItem;
        return newItem;
    }
}

// where T : BaseClass, IInterface — наследование + интерфейс
public void Handle<T>(T entity)
    where T : BaseEntity, IHasTimestamp, IAuditable
{
    entity.UpdatedAt = DateTime.UtcNow;
    entity.ModifiedBy = "system";
}
```

### Covariance и Contravariance

```csharp
// Covariance (out T) — можно использовать более производный тип
// Только для OUTPUT позиций (return values)
public interface IReadOnlyRepository<out T>
{
    T GetById(int id);
    IEnumerable<T> GetAll(); // IEnumerable<out T> — ковариантен
}

// Пример: IEnumerable<out T> ковариантен
IEnumerable<string> strings = ["hello", "world"];
IEnumerable<object> objects = strings; // OK — string : object, out T

// Contravariance (in T) — можно использовать менее производный тип
// Только для INPUT позиций (method parameters)
public interface IComparer<in T>
{
    int Compare(T x, T y);
}

// Пример: Action<in T> контравариантен
Action<object> printObj = obj => Console.WriteLine(obj);
Action<string> printStr = printObj; // OK — string : object, in T
printStr("hello"); // вызовет printObj

// Практический пример
public interface IEventHandler<in TEvent> where TEvent : IEvent
{
    Task HandleAsync(TEvent @event, CancellationToken ct);
}

// Handler для базового типа
public sealed class LoggingHandler : IEventHandler<IEvent>
{
    public Task HandleAsync(IEvent @event, CancellationToken ct)
    {
        Console.WriteLine($"Event: {@event.GetType().Name}");
        return Task.CompletedTask;
    }
}

// Можно использовать как handler для конкретного события
IEventHandler<OrderCreatedEvent> handler = new LoggingHandler(); // contravariance
```

### Generic Math (C# 11 / .NET 7)

```csharp
using System.Numerics;

// INumber<T> — универсальная математика для любого числового типа
public static T Sum<T>(ReadOnlySpan<T> values) where T : INumber<T>
{
    var result = T.Zero;
    foreach (var value in values)
        result += value;
    return result;
}

// Использование с разными типами
int intSum = Sum<int>([1, 2, 3, 4, 5]);           // 15
double dblSum = Sum<double>([1.1, 2.2, 3.3]);     // 6.6
decimal decSum = Sum<decimal>([10.5m, 20.3m]);     // 30.8

// Среднее для любого числового типа
public static T Average<T>(ReadOnlySpan<T> values)
    where T : INumber<T>
{
    if (values.IsEmpty)
        throw new InvalidOperationException("Sequence is empty");

    var sum = T.Zero;
    foreach (var value in values)
        sum += value;

    return sum / T.CreateChecked(values.Length);
}

// Clamp — ограничение диапазона
public static T Clamp<T>(T value, T min, T max) where T : INumber<T> =>
    T.Clamp(value, min, max);

// Интерфейсы generic math
public static T Parse<T>(string input) where T : IParsable<T> =>
    T.Parse(input, null);

public static bool TryFormat<T>(T value, Span<char> destination, out int written)
    where T : ISpanFormattable =>
    value.TryFormat(destination, out written, default, null);
```

---

## Практические паттерны

### Pagination с LINQ

```csharp
public sealed record PagedResult<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int Page,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling((double)TotalCount / PageSize);
    public bool HasNext => Page < TotalPages;
    public bool HasPrevious => Page > 1;
}

public static async Task<PagedResult<T>> ToPagedAsync<T>(
    this IQueryable<T> query,
    int page,
    int pageSize,
    CancellationToken ct = default)
{
    var totalCount = await query.CountAsync(ct).ConfigureAwait(false);

    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync(ct)
        .ConfigureAwait(false);

    return new PagedResult<T>(items, totalCount, page, pageSize);
}

// Использование
var result = await dbContext.Orders
    .Where(o => o.Status == "Active")
    .OrderByDescending(o => o.CreatedAt)
    .ToPagedAsync(page: 2, pageSize: 20, ct);
```

### Specification Pattern с Expression Trees

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();

    public bool IsSatisfiedBy(T entity) =>
        ToExpression().Compile()(entity);

    public Specification<T> And(Specification<T> other) =>
        new AndSpecification<T>(this, other);

    public Specification<T> Or(Specification<T> other) =>
        new OrSpecification<T>(this, other);

    public Specification<T> Not() =>
        new NotSpecification<T>(this);
}

public sealed class ActiveOrderSpec : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression() =>
        o => o.Status == "Active" && !o.IsDeleted;
}

public sealed class HighValueOrderSpec : Specification<Order>
{
    private readonly decimal _minTotal;
    public HighValueOrderSpec(decimal minTotal) => _minTotal = minTotal;

    public override Expression<Func<Order, bool>> ToExpression() =>
        o => o.Total >= _minTotal;
}

// Использование
var spec = new ActiveOrderSpec().And(new HighValueOrderSpec(1000));
var orders = await dbContext.Orders
    .Where(spec.ToExpression())
    .ToListAsync(ct);
```

---

## Советы по производительности

```csharp
// 1. Используй Count свойство вместо LINQ Count() для коллекций
List<int> list = [1, 2, 3];
int count = list.Count;          // O(1) — свойство
int countLinq = list.Count();    // O(1) для ICollection, но лишний вызов

// 2. Any() вместо Count() > 0
if (list.Any()) { }              // останавливается на первом
if (list.Count() > 0) { }       // может перечислить всё (для IEnumerable)

// 3. HashSet для Contains в циклах
var allowedIds = orders.Select(o => o.Id).ToHashSet(); // O(n) один раз
foreach (var item in items)
{
    if (allowedIds.Contains(item.OrderId)) // O(1) каждый раз
        Process(item);
}

// 4. Не материализуй, если не надо
// Плохо:
var filtered = items.Where(x => x > 0).ToList().FirstOrDefault();
// Хорошо:
var filtered2 = items.FirstOrDefault(x => x > 0);

// 5. Capacity для List и Dictionary
var results = new List<OrderDto>(capacity: orders.Count); // без реаллокаций
var map = new Dictionary<Guid, Order>(capacity: orders.Count);

// 6. Span + stackalloc вместо LINQ для горячих путей
Span<int> buffer = stackalloc int[10];
// ... заполнение
int sum = 0;
foreach (var val in buffer)
    sum += val;

// 7. FrozenDictionary для static lookup таблиц
private static readonly FrozenDictionary<string, Handler> Handlers =
    new Dictionary<string, Handler>
    {
        ["create"] = new CreateHandler(),
        ["update"] = new UpdateHandler(),
        ["delete"] = new DeleteHandler(),
    }.ToFrozenDictionary(StringComparer.OrdinalIgnoreCase);
```

---

## См. также

- [Типы и память](types-and-memory.md)
- [ООП и классы](oop.md)
- [Delegates и Events](delegates-events.md)
- [Async и потоки](async-threading.md)
