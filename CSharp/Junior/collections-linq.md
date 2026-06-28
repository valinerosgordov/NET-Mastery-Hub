---
tags: [csharp, collections, linq, junior, list, dictionary, hashset, lazy-evaluation, performance]
level: Junior
date: 2026-05-04
---

# Collections и LINQ — коллекции и язык запросов

> **Главные коллекции BCL и LINQ как универсальный язык pipeline'ов.** `List<T>`, `Dictionary<TKey,TValue>`, `HashSet<T>`, специальные коллекции, LINQ операторы (Where/Select/GroupBy/Aggregate), deferred execution, performance characteristics. Закрывает пробел: «знаю про List, не понимаю когда HashSet, и почему LINQ chain не выполняется до ToList».

---

## 0. Как читать этот файл

Если ты впервые работаешь с коллекциями в C# — читай разделы 1→4 подряд: получишь рабочую модель. Если уже пишешь LINQ, но непонятно про deferred execution — раздел 7. Если интересна performance — раздел 11. Production-ready — раздел 12 (best practices), 14 (pitfalls).

Все примеры самостоятельные. Cross-language якоря (`> [!info]-`) — раскрывай если переходишь из Python / Java / JavaScript / Rust. Interview-вопросы (`> [!question]-`) встроены.

---

## 1. Что это, зачем и когда

### 1.1. Что такое коллекция

**Коллекция** — структура данных, хранящая множество элементов одного типа. В .NET:

```csharp
var list = new List<int> { 1, 2, 3, 4, 5 };           // динамический массив
var dict = new Dictionary<string, int> { ["a"] = 1 };   // hash map
var set = new HashSet<int> { 1, 2, 3 };                  // unique items
var queue = new Queue<int>();                            // FIFO
var stack = new Stack<int>();                            // LIFO
```

Все они реализуют `IEnumerable<T>` — общий интерфейс iteration.

### 1.2. Что такое LINQ

**LINQ** (Language Integrated Query) — set extension methods на `IEnumerable<T>` для transformation, filtering, aggregation:

```csharp
var users = GetUsers();

var activeAdminEmails = users
    .Where(u => u.IsActive)              // filter
    .Where(u => u.Role == "Admin")        // filter
    .Select(u => u.Email)                 // projection
    .OrderBy(e => e)                      // sort
    .ToList();                            // materialize
```

LINQ — declarative, composable, lazy. Один pipeline на любой `IEnumerable<T>` (in-memory list, DB query, stream).

### 1.3. Главное правило

```
Используй List<T> для:
  - Sequential данных, indexed access
  - Default choice для коллекции

Используй Dictionary<TKey, TValue> для:
  - Lookup by key (O(1))
  - Mapping key → value

Используй HashSet<T> для:
  - Unique items, membership check (O(1))
  - Set operations (union, intersect)

Используй LINQ для:
  - Transformations и aggregations
  - Декларативный подход вместо loops
  - Composable pipeline шагов

Используй обычный foreach для:
  - Side effects (mutation, I/O)
  - Performance-критичный код
  - Когда LINQ скрывает intent
```

### 1.4. Эволюция: .NET 1.0 → C# 13

| Версия | Год | Что появилось |
|--------|-----|---------------|
| **.NET 1.0** | 2002 | `ArrayList`, `Hashtable` (non-generic) |
| **.NET 2.0** | 2005 | Generic collections (`List<T>`, `Dictionary<TKey,TValue>`) |
| **.NET 3.5** | 2008 | **LINQ** + extension methods + lambdas |
| **.NET 4.0** | 2010 | `ConcurrentDictionary`, `BlockingCollection` |
| **.NET Core 2.1** | 2018 | `Span<T>`, optimized collection internals |
| **.NET 6** | 2021 | `PriorityQueue<T>`, performance улучшения |
| **.NET 8** | 2023 | `FrozenDictionary`, `FrozenSet` (immutable optimized) |
| **C# 12** | 2023 | **Collection expressions** `[1, 2, 3]` |
| **C# 13** | 2024 | `params ReadOnlySpan<T>` |

### 1.5. Quick reference

| Коллекция | Lookup | Insert | Remove | Use case |
|-----------|--------|--------|--------|----------|
| `List<T>` | O(n) | O(1) amortized | O(n) | Sequential, indexed |
| `Dictionary<K,V>` | O(1) | O(1) | O(1) | Key-value mapping |
| `HashSet<T>` | O(1) | O(1) | O(1) | Unique items, membership |
| `SortedDictionary<K,V>` | O(log n) | O(log n) | O(log n) | Sorted by key |
| `SortedSet<T>` | O(log n) | O(log n) | O(log n) | Sorted unique |
| `LinkedList<T>` | O(n) | O(1) at head/tail | O(1) by node | Frequent insertions middle |
| `Queue<T>` | — | O(1) | O(1) | FIFO |
| `Stack<T>` | — | O(1) | O(1) | LIFO |
| `PriorityQueue<T,P>` | — | O(log n) | O(log n) | Sorted by priority |

> [!info]- Если ты знаешь Python / Java / JavaScript / Rust
> **Python:** `list` ↔ `List<T>`, `dict` ↔ `Dictionary<K,V>`, `set` ↔ `HashSet<T>`. List comprehensions и `map`/`filter` ↔ LINQ Select/Where. Generators ↔ deferred execution.
>
> **Java:** `ArrayList<T>` ↔ `List<T>`, `HashMap<K,V>` ↔ `Dictionary<K,V>`, `HashSet<T>` ↔ `HashSet<T>`. Streams API (Java 8+) ↔ LINQ — same idea, разный синтаксис.
>
> **JavaScript:** `Array` ↔ `List<T>`, `Map` ↔ `Dictionary<K,V>`, `Set` ↔ `HashSet<T>`. Array methods (.map, .filter, .reduce) — eager (LINQ lazy).
>
> **Rust:** `Vec<T>` ↔ `List<T>`, `HashMap<K,V>` ↔ `Dictionary<K,V>`, `HashSet<T>` ↔ `HashSet<T>`. Iterators trait ↔ LINQ — lazy + composable.

> [!question]- Интервью: чем `List<T>` отличается от `T[]`?
> `T[]` — fixed-size array на heap, размер задаётся при создании. `List<T>` — dynamic array, использует internal `T[]` который растёт автоматически (capacity удваивается при overflow). `List<T>.Add` — O(1) amortized. Доступ по индексу одинаков (O(1)). Для unknown size с insertions — `List<T>`. Для known fixed size + perf — `T[]`. `List<T>` имеет более богатый API (Add, Remove, IndexOf), array — basic indexing.

---

## 2. `List<T>` — динамический массив

### 2.1. Создание и инициализация

```csharp
// Empty
var list1 = new List<int>();

// С initial capacity (предзаполняет buffer, не elements)
var list2 = new List<int>(capacity: 100);

// С initial elements
var list3 = new List<int> { 1, 2, 3 };
var list4 = new List<int>([1, 2, 3]);   // collection expression (C# 12+)

// Из IEnumerable
var list5 = new List<int>(Enumerable.Range(1, 5));

// Collection expression (C# 12+)
List<int> list6 = [1, 2, 3, 4, 5];
```

### 2.2. Основные операции

```csharp
var list = new List<int> { 1, 2, 3, 4, 5 };

// Добавление
list.Add(6);                       // в конец — O(1) amortized
list.Insert(0, 0);                 // в позицию — O(n)
list.AddRange([7, 8, 9]);          // несколько

// Удаление
list.Remove(3);                    // первое вхождение — O(n)
list.RemoveAt(0);                  // по индексу — O(n)
list.RemoveAll(x => x % 2 == 0);    // по predicate
list.Clear();                       // все

// Доступ
int first = list[0];                // O(1)
list[0] = 99;                       // O(1)
list.Count;                         // current size
list.Capacity;                      // buffer size

// Поиск
bool contains = list.Contains(5);                   // O(n)
int idx = list.IndexOf(3);                           // O(n)
int found = list.Find(x => x > 10) ?? 0;             // O(n)
List<int> all = list.FindAll(x => x > 5);            // O(n)
bool any = list.Exists(x => x > 100);                // O(n)
bool all2 = list.TrueForAll(x => x > 0);              // O(n)

// Sort / Reverse
list.Sort();                                          // in-place
list.Sort((a, b) => b.CompareTo(a));                  // custom comparer (descending)
list.Reverse();
```

### 2.3. Capacity vs Count

```csharp
var list = new List<int>(capacity: 100);
list.Count;       // 0 — пустой
list.Capacity;    // 100 — buffer есть

list.Add(1);
list.Count;       // 1
list.Capacity;    // 100 — не растёт

list.AddRange(Enumerable.Range(1, 200));
list.Capacity;    // выросло (обычно удваивается: 100→200→400...)

list.TrimExcess();   // освободить unused buffer
list.Capacity;    // ≈ Count
```

Pre-allocate capacity если знаешь размер — избегаешь нескольких realloc.

### 2.4. Iteration

```csharp
var list = new List<int> { 1, 2, 3 };

// foreach — безопасный
foreach (var x in list)
    Console.WriteLine(x);

// for — для index-based access
for (int i = 0; i < list.Count; i++)
    list[i] *= 2;

// LINQ — для трансформации
var doubled = list.Select(x => x * 2).ToList();
```

### 2.5. **Ловушка**: модификация во время iteration

```csharp
var list = new List<int> { 1, 2, 3, 4, 5 };

// ❌ — InvalidOperationException
foreach (var x in list)
{
    if (x % 2 == 0) list.Remove(x);
}

// ✅ — итерируй копию или используй RemoveAll
list.RemoveAll(x => x % 2 == 0);

// или through index с end-to-start
for (int i = list.Count - 1; i >= 0; i--)
    if (list[i] % 2 == 0) list.RemoveAt(i);

// или собирай to-remove items first
var toRemove = list.Where(x => x % 2 == 0).ToList();
foreach (var x in toRemove) list.Remove(x);
```

`foreach` создаёт enumerator; модификация underlying collection invalidates его → exception.

### 2.6. ToArray / ToList — копии

```csharp
var list = new List<int> { 1, 2, 3 };
int[] array = list.ToArray();      // копия
List<int> copy = list.ToList();    // копия

// Mutation copy не affects original
array[0] = 99;
list[0];   // 1 — без изменений
```

### 2.7. AsReadOnly / ImmutableList

```csharp
var list = new List<int> { 1, 2, 3 };

// Read-only view (не копия — отражает changes в original)
IReadOnlyList<int> readOnly = list.AsReadOnly();
// readOnly.Add(4);   // ❌ no Add method
list.Add(4);
readOnly.Count;       // 4 — same underlying

// Immutable копия
ImmutableList<int> immutable = list.ToImmutableList();
var newList = immutable.Add(99);   // returns new ImmutableList
```

`IReadOnlyList<T>` — view на mutable list. `ImmutableList<T>` — настоящая immutable structure (slow, но safe).

### 2.8. `List<T>` vs `T[]`

| | `List<T>` | `T[]` |
|---|----------|-------|
| Size | dynamic | fixed |
| API | rich (Add, Remove, Find...) | basic indexing |
| Performance | similar (small overhead на indexing) | slightly faster |
| Use case | Default | Known size, perf hot path |

> [!question]- Интервью: что такое Capacity в `List<T>`?
> Размер internal `T[]` buffer, в котором хранятся elements. Может быть больше `Count`. При `Add` если `Count == Capacity` — buffer удваивается (новый array, копирование). Pre-allocate через `new List<T>(capacity)` избегает several realloc если known approximate size. `TrimExcess()` уменьшает Capacity до Count (после массовых удалений). `Capacity` — implementation detail, обычно не важен для correctness, но важен для perf в hot path.

---

## 3. `Dictionary<TKey, TValue>` — hash map

### 3.1. Создание

```csharp
var dict = new Dictionary<string, int>();
var dict2 = new Dictionary<string, int>
{
    ["alice"] = 1,
    ["bob"] = 2
};
var dict3 = new Dictionary<string, int>
{
    { "alice", 1 },
    { "bob", 2 }
};

// С comparer
var caseInsensitive = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
caseInsensitive["Alice"] = 1;
caseInsensitive["alice"];   // 1 — found
```

### 3.2. Операции

```csharp
var dict = new Dictionary<string, int>();

// Add / set
dict.Add("alice", 1);            // throws если key exists
dict["alice"] = 1;                // upsert (set or add)

// Get
int value = dict["alice"];        // throws KeyNotFoundException если нет
if (dict.TryGetValue("alice", out var v)) { /* v = 1 */ }

// Check
bool has = dict.ContainsKey("alice");
bool hasValue = dict.ContainsValue(1);   // O(n)!

// Remove
dict.Remove("alice");
dict.Remove("alice", out var removed);   // .NET 6+ возвращает value

// Iteration
foreach (var kvp in dict)
    Console.WriteLine($"{kvp.Key} = {kvp.Value}");

// Только keys или values
foreach (var k in dict.Keys) { }
foreach (var v in dict.Values) { }
```

### 3.3. TryGetValue vs indexer

```csharp
// ❌ Throws если нет ключа
int v = dict["missing"];

// ✅ Safe
if (dict.TryGetValue("missing", out var v))
{
    // используй v
}

// ✅ С default
int value = dict.GetValueOrDefault("missing", -1);   // .NET 5+
```

### 3.4. Ключ должен быть immutable + правильный hash

```csharp
public class BadKey
{
    public int Id { get; set; }   // mutable!
}

var bad = new BadKey { Id = 1 };
var dict = new Dictionary<BadKey, string> { [bad] = "value" };

bad.Id = 99;   // ⚠️ изменили key!
dict[bad];     // KeyNotFoundException — hash изменился!
```

Ключ не должен mutate после insert. Используй immutable typesЫ (record, readonly properties, value types).

### 3.5. Custom comparer

```csharp
public class CaseInsensitiveComparer : IEqualityComparer<string>
{
    public bool Equals(string? x, string? y) =>
        string.Equals(x, y, StringComparison.OrdinalIgnoreCase);
    public int GetHashCode(string obj) =>
        StringComparer.OrdinalIgnoreCase.GetHashCode(obj);
}

var dict = new Dictionary<string, int>(new CaseInsensitiveComparer());
```

Обычно достаточно `StringComparer.OrdinalIgnoreCase` встроенного.

### 3.6. Performance

```
Operation       | Average | Worst |
----------------|---------|-------|
Add             | O(1)    | O(n)  | (resize)
Lookup (Index)  | O(1)    | O(n)  | (hash collisions)
Remove          | O(1)    | O(n)  |
ContainsKey     | O(1)    | O(n)  |
ContainsValue   | O(n)    | O(n)  | (linear scan!)
```

Hash collisions редки с good hash function. ContainsValue O(n) — не для frequent use.

> [!question]- Интервью: почему Dictionary key не должен mutate?
> Dictionary использует hash code ключа для быстрого lookup. При insert hash вычисляется и item помещается в bucket. Если key мутируется после insert — hash изменится, но item остаётся в старом bucket. Lookup через новый hash не найдёт item — KeyNotFoundException. Используй immutable types для keys: records, value types (struct), readonly properties. String — immutable, отличный key. Custom class с mutable properties — bad key.

---

## 4. `HashSet<T>` — unique items

### 4.1. Создание и операции

```csharp
var set = new HashSet<int> { 1, 2, 3, 4, 5 };

// Add — returns bool (true если добавили, false если уже было)
bool added = set.Add(3);   // false
set.Add(99);                // true

// Membership — O(1)
bool has = set.Contains(5);

// Remove
set.Remove(3);

// Iteration
foreach (var x in set)
    Console.WriteLine(x);

// Без guarantee порядка!
```

### 4.2. Set operations

```csharp
var a = new HashSet<int> { 1, 2, 3, 4 };
var b = new HashSet<int> { 3, 4, 5, 6 };

// Union (объединение) — modifies in-place
a.UnionWith(b);   // a = { 1, 2, 3, 4, 5, 6 }

// Intersection — modifies in-place
var c = new HashSet<int>(a);
c.IntersectWith(b);   // c = { 3, 4 }

// Difference — modifies in-place
var d = new HashSet<int>(a);
d.ExceptWith(b);   // d = { 1, 2 }

// SymmetricExcept — XOR
var e = new HashSet<int>(a);
e.SymmetricExceptWith(b);   // e = { 1, 2, 5, 6 }

// Subset / superset checks
a.IsSubsetOf(b);
a.IsSupersetOf(b);
a.SetEquals(b);
```

### 4.3. Когда HashSet vs List

```csharp
// ❌ List для membership check — O(n) каждый раз
var list = new List<int>();
foreach (var x in items)
    if (!list.Contains(x))   // O(n) на каждый item
        list.Add(x);
// Total: O(n²)

// ✅ HashSet — O(1) per check
var set = new HashSet<int>();
foreach (var x in items)
    set.Add(x);   // O(1)
// Total: O(n)
```

Для unique items / membership check — HashSet всегда. List только если нужен порядок + indexed access.

### 4.4. Distinct() в LINQ

```csharp
var nums = new[] { 1, 2, 2, 3, 3, 3, 4 };
var unique = nums.Distinct().ToList();   // [1, 2, 3, 4]

// Под капотом — использует HashSet
```

### 4.5. Custom equality

```csharp
public record User(int Id, string Email);

// Default — value equality (record auto)
var users = new HashSet<User>
{
    new User(1, "a@x.com"),
    new User(1, "a@x.com"),   // duplicate!
    new User(2, "b@x.com")
};
users.Count;   // 2

// Custom comparer
public class UserByIdComparer : IEqualityComparer<User>
{
    public bool Equals(User? x, User? y) => x?.Id == y?.Id;
    public int GetHashCode(User obj) => obj.Id;
}

var byId = new HashSet<User>(new UserByIdComparer());
```

> [!question]- Интервью: когда HashSet vs Dictionary vs List?
> **List** — sequential данные, indexed access, **не для membership** (O(n) Contains). **Dictionary** — key→value mapping, O(1) lookup. **HashSet** — unique items, O(1) membership/add/remove, set operations. Decision: нужны pairs (key+value) → Dictionary; только unique items, проверка существования → HashSet; sequential / indexed → List. Для duplicate detection — HashSet вместо List.Contains в loop (O(n²) → O(n)).

---

## 5. Специальные коллекции

### 5.1. `Queue<T>` — FIFO

```csharp
var queue = new Queue<int>();
queue.Enqueue(1);
queue.Enqueue(2);
queue.Enqueue(3);

int first = queue.Dequeue();   // 1 — first in, first out
int peek = queue.Peek();        // 2 — без удаления
queue.Count;                    // 2
```

### 5.2. `Stack<T>` — LIFO

```csharp
var stack = new Stack<int>();
stack.Push(1);
stack.Push(2);
stack.Push(3);

int top = stack.Pop();          // 3 — last in, first out
int peek = stack.Peek();        // 2
```

### 5.3. `PriorityQueue<TElement, TPriority>` (.NET 6+)

```csharp
var pq = new PriorityQueue<string, int>();
pq.Enqueue("low", 5);
pq.Enqueue("high", 1);          // lower priority value = higher priority
pq.Enqueue("medium", 3);

string first = pq.Dequeue();    // "high"
string second = pq.Dequeue();   // "medium"
string third = pq.Dequeue();    // "low"
```

Min-heap based. Lower priority value = served first. Полезно для task scheduling, Dijkstra, A*.

### 5.4. `LinkedList<T>`

```csharp
var ll = new LinkedList<int>();
ll.AddLast(1);
ll.AddLast(2);
ll.AddFirst(0);
// 0 ↔ 1 ↔ 2

var node = ll.Find(1);
ll.AddBefore(node!, 99);
// 0 ↔ 99 ↔ 1 ↔ 2

ll.Remove(node!);
// 0 ↔ 99 ↔ 2
```

LinkedList — для frequent insertions/deletions в середине. В большинстве cases `List<T>` лучше (cache-friendly, lower overhead).

### 5.5. SortedDictionary / SortedSet

```csharp
var sortedDict = new SortedDictionary<string, int>
{
    ["charlie"] = 3,
    ["alice"] = 1,
    ["bob"] = 2
};
// Iteration в порядке keys: alice, bob, charlie

var sortedSet = new SortedSet<int> { 5, 2, 8, 1, 3 };
// Iteration: 1, 2, 3, 5, 8
```

O(log n) operations (red-black tree). Использовать когда нужен sorted order при iteration. Для one-shot sort — `List<T>.Sort()` faster.

### 5.6. ConcurrentDictionary, ConcurrentBag, BlockingCollection

```csharp
// Thread-safe dictionary
var concDict = new ConcurrentDictionary<string, int>();

// AddOrUpdate atomic
concDict.AddOrUpdate("key", 1, (k, oldValue) => oldValue + 1);

// GetOrAdd atomic
int v = concDict.GetOrAdd("key", k => ComputeValue(k));

// TryAdd / TryRemove / TryUpdate — атомарные
```

Для multi-threaded scenarios. Подробнее — отдельный topic про concurrency.

> [!warning] «Atomic» с оговоркой: value factory у `GetOrAdd` / `AddOrUpdate` под contention может выполниться несколько раз (в словарь попадёт только один результат). Делай factory идемпотентной/дешёвой или храни `Lazy<T>`. Также `Count` / `ToArray()` берут все внутренние локи. Детали — [[async-threading]] (раздел 6.7).

### 5.7. FrozenDictionary, FrozenSet (.NET 8+)

```csharp
var frozen = new[] { 1, 2, 3, 4, 5 }.ToFrozenSet();
var frozenDict = new Dictionary<string, int>
{
    ["a"] = 1,
    ["b"] = 2
}.ToFrozenDictionary();

// Read-only, но заметно быстрее на чтение, чем Dictionary
```

**Как устроен Frozen — почему быстрее.** Обычный `Dictionary<TKey, TValue>` обязан в любой момент уметь `Add`/`Remove`, поэтому его структура — компромисс. `ToFrozenDictionary()` получает **весь набор ключей заранее** и на этапе построения анализирует его, выбирая специализированную внутреннюю реализацию:

- **Маленькие наборы** (единицы элементов) → реализация с линейным сканом: на 3–5 элементах перебор дешевле любого хеширования.
- **Строковые ключи** → анализ длин и подстрок: ищется минимальный дискриминатор (например, «все ключи различаются длиной» или «достаточно хешировать символы 2–4»), и хешируется только он, а не вся строка.
- **Плотные int-ключи** → прямое обращение по индексу в массиве, хеш не нужен вообще.
- **Общий случай** → хеш-таблица с предвычисленными при построении хеш-кодами и подобранным числом бакетов почти без коллизий (perfect-hash-подход).

**Trade-off — за что платим.** Построение на порядок-два дороже, чем у обычного `Dictionary`: вся аналитика выполняется в `ToFrozenDictionary()`. Чтение — быстрее: от десятков процентов в общем случае до разов на строковых ключах с удачным дискриминатором. Это классический build-cost vs read-speed: окупается, когда коллекция строится один раз и читается миллионы раз.

**`GetAlternateLookup` (.NET 9+)** — поиск по `ReadOnlySpan<char>` без материализации строки-ключа:

```csharp
var routes = new Dictionary<string, int>
{
    ["orders"] = 1,
    ["users"] = 2
}.ToFrozenDictionary();

var lookup = routes.GetAlternateLookup<ReadOnlySpan<char>>();

ReadOnlySpan<char> segment = "users/42".AsSpan(0, 5); // срез без аллокации
int handlerId = lookup[segment];
```

Классика применения — парсинг: ключ приходит как кусок входной строки, и раньше пришлось бы делать `Substring` (аллокация) ради одного lookup'а.

**Когда НЕ использовать Frozen:**

- данные периодически перестраиваются (кеш с рефрешем) — build-cost съест выигрыш чтения; «изменить» Frozen нельзя, только построить заново целиком;
- коллекция читается один-два раза — не окупится;
- нужна дешёвая «модифицированная копия» — это ниша `ImmutableDictionary` (структурный шеринг: O(log n) на копию-с-изменением, но и чтение медленнее). Frozen и Immutable решают разные задачи: Frozen — максимум чтения без изменений, Immutable — дешёвые снапшоты с изменениями.

> [!question]- Интервью: FrozenDictionary vs ImmutableDictionary — оба же read-only?
> Read-only — единственное сходство. `ImmutableDictionary` оптимизирован под **дешёвое создание изменённых копий**: внутри дерево со структурным шерингом, `SetItem` стоит O(log n) и не копирует всё, но и lookup — O(log n), медленнее обычного Dictionary. `FrozenDictionary` оптимизирован под **чтение**: при построении анализирует ключи и выбирает специализированную реализацию (linear scan для маленьких, дискриминатор по подстроке для строк, прямой индекс для плотных int), lookup быстрее Dictionary, но любое «изменение» = полная пересборка. Снапшоты с эволюцией → Immutable; построил-однажды-читаешь-всегда → Frozen.

Для read-only данных, инициализированных при app startup и часто читаемых (config, lookup-таблицы, роутинг).

---

## 6. LINQ — основы

### 6.1. Что такое LINQ

LINQ — **extension methods** на `IEnumerable<T>`, которые принимают functions (lambdas) и возвращают transformed sequence:

```csharp
using System.Linq;

var nums = new[] { 1, 2, 3, 4, 5 };

var evens = nums.Where(x => x % 2 == 0);   // 2, 4
var doubled = nums.Select(x => x * 2);      // 2, 4, 6, 8, 10
var first = nums.First(x => x > 3);          // 4
var sum = nums.Sum();                         // 15
```

Все LINQ методы — extensions из `System.Linq`. `using System.Linq;` обычно implicit (через `ImplicitUsings` в .NET 6+).

### 6.2. Method syntax vs query syntax

```csharp
// Method syntax (idiomatic в C#)
var result = users
    .Where(u => u.Age > 18)
    .OrderBy(u => u.Name)
    .Select(u => u.Email)
    .ToList();

// Query syntax (SQL-like)
var result2 = (from u in users
               where u.Age > 18
               orderby u.Name
               select u.Email).ToList();
```

Эквивалентны. Method syntax более популярен. Query syntax удобен для complex joins / groups.

### 6.3. Lambda syntax

```csharp
// Просто
nums.Where(x => x > 5);

// Тело
nums.Where(x => 
{
    var threshold = ComputeThreshold();
    return x > threshold;
});

// Method group
nums.Where(IsValid);

bool IsValid(int x) => x > 5;
```

Lambda — implicit conversion в `Func<T, TResult>` или `Action<T>`.

### 6.4. Deferred execution — главная особенность

```csharp
var query = nums.Where(x => 
{
    Console.WriteLine($"Filtering {x}");
    return x > 2;
});

// Нет output — query не выполнен!

foreach (var x in query)
    Console.WriteLine($"Got {x}");

// Output:
// Filtering 1
// Filtering 2
// Filtering 3
// Got 3
// Filtering 4
// Got 4
// Filtering 5
// Got 5
```

LINQ методы (Where, Select, Take, OrderBy) **не выполняются** до первой итерации. Это **lazy evaluation**.

### 6.5. Materialization — `ToList`/`ToArray`/`First`/`Sum`

```csharp
// Lazy
var query = nums.Where(x => x > 2);   // ничего не выполнено

// Eager — материализуют
var list = query.ToList();             // выполнены, копия в List
var array = query.ToArray();           // копия в array

// Terminal operations
var first = nums.First();               // материализует только до first
var sum = nums.Sum();                   // итерирует всё для агрегации
var count = nums.Count();               // итерирует (для IEnumerable; O(1) для ICollection)
```

**Правило:** материализуй (`ToList`/`ToArray`) когда нужно итерировать **много раз** или хранить.

### 6.6. Multiple iteration of lazy query

```csharp
var query = nums.Where(x => 
{
    Console.WriteLine($"Check {x}");
    return x > 2;
});

foreach (var x in query) { }   // печатает Check 1..5
foreach (var x in query) { }   // печатает Check 1..5 СНОВА!
```

Каждый `foreach` re-executes query. Если `Where` дорогой (DB call) — повторяется. Materialize один раз через `ToList`.

### 6.7. Закономерность LINQ — chain без allocation

```csharp
var result = source
    .Where(x => x.IsActive)
    .Where(x => x.Age > 18)
    .Select(x => x.Email)
    .Take(10)
    .ToList();
```

Между `Where`, `Select`, `Take` — **нет промежуточных коллекций**. Каждый element pipes через всю цепочку, потом next element. Только финальная `ToList` материализует.

Это **лучшее свойство LINQ** — composable без overhead.

> [!question]- Интервью: что такое deferred execution в LINQ?
> LINQ методы (Where, Select, OrderBy, Take...) не выполняют работу при вызове — возвращают `IEnumerable<T>` объект, который "знает", что делать. Реальное выполнение начинается при iteration (foreach, ToList, First, Count). Это **lazy evaluation**: 1) экономия — если только первый element нужен, не вычисляется остальное. 2) composability — chain без промежуточных коллекций. Ловушка: повторное iteration = повторное выполнение. Materialize через `ToList` один раз если нужно много раз. Для async source — `IAsyncEnumerable` + `await foreach`.

---

## 7. LINQ — операторы

### 7.1. Filtering

```csharp
nums.Where(x => x > 5);                          // filter by predicate
nums.OfType<int>();                               // filter by type
nums.Distinct();                                  // unique values
nums.DistinctBy(x => x % 10);                     // unique by key (.NET 6+)
```

### 7.2. Projection

```csharp
users.Select(u => u.Name);                        // → IEnumerable<string>
users.Select((u, i) => new { Index = i, u.Name }); // с index
users.SelectMany(u => u.Orders);                  // flatten nested
users.SelectMany(u => u.Orders, (u, o) => new { u.Name, o.Total }); // join
```

### 7.3. Sorting

```csharp
users.OrderBy(u => u.Name);                       // ascending
users.OrderByDescending(u => u.Age);              // descending
users.OrderBy(u => u.LastName).ThenBy(u => u.FirstName);   // multi-level
users.Reverse();                                   // reverse current order
```

### 7.4. Grouping

```csharp
var byCity = users.GroupBy(u => u.City);

foreach (var group in byCity)
{
    Console.WriteLine($"City: {group.Key}, count: {group.Count()}");
    foreach (var u in group)
        Console.WriteLine($"  {u.Name}");
}

// С selector
var summary = users
    .GroupBy(u => u.City)
    .Select(g => new { City = g.Key, Count = g.Count(), AvgAge = g.Average(u => u.Age) });
```

### 7.5. Aggregation

```csharp
nums.Count();                                     // total
nums.Count(x => x > 5);                            // filtered count
nums.Sum();                                        // sum
nums.Min();
nums.Max();
nums.Average();
nums.Aggregate((acc, x) => acc + x);               // generic fold
nums.Aggregate(0, (acc, x) => acc + x);            // с seed
```

### 7.6. Element access

```csharp
nums.First();                                      // first или throw
nums.First(x => x > 5);                            // first matching или throw
nums.FirstOrDefault();                             // first или default(T)
nums.FirstOrDefault(x => x > 100, -1);             // .NET 6+ default value
nums.Last();
nums.LastOrDefault();
nums.Single();                                     // exactly one or throw
nums.SingleOrDefault();                            // 0 or 1, иначе throw
nums.ElementAt(2);                                 // by index
nums.ElementAtOrDefault(99);                       // или default
```

### 7.7. Quantifiers

```csharp
nums.Any();                                        // есть хоть один?
nums.Any(x => x > 100);                            // есть matching?
nums.All(x => x > 0);                              // все matching?
nums.Contains(5);                                   // содержит value?
```

### 7.8. Set operations

```csharp
var a = new[] { 1, 2, 3, 4 };
var b = new[] { 3, 4, 5, 6 };

a.Union(b);                                        // 1, 2, 3, 4, 5, 6
a.Intersect(b);                                    // 3, 4
a.Except(b);                                       // 1, 2
a.Concat(b);                                       // 1, 2, 3, 4, 3, 4, 5, 6 (с duplicates)
```

### 7.9. Partitioning

```csharp
nums.Take(3);                                      // первые 3
nums.Skip(2);                                      // пропустить 2
nums.TakeWhile(x => x < 5);                        // пока predicate
nums.SkipWhile(x => x < 3);                        // пропускать пока
nums.Take(2..5);                                    // .NET 6+ range
nums.Chunk(3);                                      // .NET 6+ split на части по 3
```

### 7.10. Joining

```csharp
var users = new[] { new { Id = 1, Name = "Alice" } };
var orders = new[] { new { UserId = 1, Total = 100m } };

// Inner join
var joined = users.Join(
    orders,
    u => u.Id,
    o => o.UserId,
    (u, o) => new { u.Name, o.Total }
);

// Group join (one-to-many)
var grouped = users.GroupJoin(
    orders,
    u => u.Id,
    o => o.UserId,
    (u, userOrders) => new { u.Name, Orders = userOrders.ToList() }
);

// Left join эмулируется через GroupJoin + SelectMany
var leftJoin = users
    .GroupJoin(orders, u => u.Id, o => o.UserId, (u, os) => new { u, os })
    .SelectMany(x => x.os.DefaultIfEmpty(), (x, o) => new { x.u.Name, Total = o?.Total ?? 0 });
```

### 7.11. Conversion

```csharp
nums.ToList();                                     // List<T>
nums.ToArray();                                    // T[]
nums.ToHashSet();                                  // HashSet<T>
nums.ToDictionary(x => x.Id);                      // → Dictionary<TKey, T>
nums.ToDictionary(x => x.Id, x => x.Name);         // → Dictionary<TKey, TValue>
nums.ToLookup(x => x.City);                        // → ILookup<TKey, TValue> — multi-value
```

### 7.12. Generation

```csharp
Enumerable.Range(1, 5);                            // 1, 2, 3, 4, 5
Enumerable.Repeat("x", 3);                          // "x", "x", "x"
Enumerable.Empty<int>();                            // empty
```

> [!question]- Интервью: чем `Single` отличается от `First`?
> **`First`** — возвращает первый элемент (matching predicate), throws если empty. Не проверяет на other matches. **`Single`** — возвращает **единственный** элемент (matching predicate), throws если 0 или > 1 match. Используй `Single` когда логически ожидаешь exactly one (например, GetById), `First` когда just need any matching (or first sorted). `OrDefault` вариант возвращает default(T) вместо throw для empty case (но `SingleOrDefault` всё равно throws при > 1 match).

---

## 8. Deferred execution deep

### 8.1. Pipeline mechanism

```csharp
var query = nums
    .Where(x => x > 2)
    .Select(x => x * 2);

// Что происходит сейчас:
// query — это object-pipeline, не результат
```

`Where` возвращает `WhereIterator` (iterator class), `Select` возвращает `SelectIterator` обернутый вокруг WhereIterator. Нет вычислений.

### 8.2. Pull-based vs push

LINQ — **pull-based**: каждый `MoveNext()` на финальном enumerator pulls один element через всю pipeline.

```
foreach calls Select.MoveNext()
  → Select.MoveNext() calls Where.MoveNext()
    → Where.MoveNext() calls Source.MoveNext()
      → Source returns 1
    → Where checks: 1 > 2? No → MoveNext() again
      → Source returns 2 → Check: No → again
      → Source returns 3 → Check: Yes → return 3
    → Where returns 3
  → Select transforms: 3*2 = 6 → returns 6
foreach receives 6
```

### 8.3. Преимущества lazy

```csharp
// Bestcase — Take(10) останавливает pipeline после 10
var first10 = millionUsers
    .Where(u => u.IsActive)
    .Select(u => u.Email)
    .Take(10)
    .ToList();   // обработаны только первые ~10-15 active users
```

Take после Where — фильтр не проходит до конца. Огромная экономия для большого source.

### 8.4. Anti-pattern: lazy + side effects

```csharp
var query = users.Where(u => 
{
    Console.WriteLine($"Checking {u.Name}");   // side effect!
    return u.IsActive;
});

query.Count();   // выполняет, печатает all
query.Count();   // ВЫПОЛНЯЕТ СНОВА, печатает again!
query.ToList();  // ВЫПОЛНЯЕТ ТРЕТИЙ РАЗ
```

Side effects в lambdas повторяются при каждом iteration. Плохо для logging, БД calls.

### 8.5. Materialize early когда нужно

```csharp
// ❌ Lazy + multiple uses → multiple executions
var activeUsers = await db.Users.Where(u => u.IsActive);
var count = activeUsers.Count();        // SQL query #1
var first = activeUsers.First();         // SQL query #2
var emails = activeUsers.Select(u => u.Email).ToList();   // SQL query #3

// ✅ Materialize once
var activeUsers = await db.Users.Where(u => u.IsActive).ToListAsync();
var count = activeUsers.Count;          // O(1), no query
var first = activeUsers.First();         // memory
var emails = activeUsers.Select(u => u.Email).ToList();
```

### 8.6. EF Core IQueryable

В EF Core LINQ — translation в SQL:

```csharp
var query = db.Users.Where(u => u.IsActive).Select(u => u.Email);
// query — IQueryable<string>, нет SQL ещё

var list = await query.ToListAsync();
// SQL генерируется и выполняется здесь:
// SELECT Email FROM Users WHERE IsActive = 1
```

`IQueryable<T>` — extension `IEnumerable<T>` для translation в external query (SQL). Lazy + materializable.

### 8.7. Mixing client / server evaluation

```csharp
// EF Core — Server evaluation (SQL)
var serverSide = db.Users
    .Where(u => u.Age > 18)              // SQL WHERE
    .Select(u => u.Name)                  // SQL SELECT
    .ToListAsync();

// Switch на client — AsEnumerable / ToList
var mixed = await db.Users
    .Where(u => u.Age > 18)               // SQL
    .AsEnumerable()                       // переход на client-side
    .Where(u => CustomCheck(u.Email))     // C# (не SQL)
    .ToList();
```

`AsEnumerable` сигнализирует "теперь обработка in-memory". До — SQL, после — C#.

> [!question]- Интервью: что такое pull-based execution в LINQ?
> LINQ — pull-based: каждый `MoveNext()` на final enumerator вытягивает один element через всю pipeline. Source returns element → Where checks → Select transforms → caller receives. Это значит элементы processed by-one, no intermediate collections. Take(10) после Where stops pulling after 10 — Where не проходит до конца. Преимущества: memory-efficient, early exit, composable. Минус: повторное iteration = повторное выполнение pipeline. Materialize через `ToList` если нужно много раз.

---

## 9. Performance

### 9.1. Boxing для value types в non-generic

```csharp
// ❌ Non-generic — boxing для int
ArrayList list = new();   // legacy
list.Add(42);             // box int → object → heap allocation

// ✅ Generic — без boxing
List<int> list2 = new();
list2.Add(42);            // direct int, no boxing
```

Всегда используй generic collections. ArrayList / Hashtable — legacy.

### 9.2. Dictionary lookup vs List.Find

```csharp
// O(n) лookup
var found = list.FirstOrDefault(x => x.Id == 42);

// O(1) lookup
var found2 = dict.TryGetValue(42, out var v) ? v : null;
```

Для частых lookups — Dictionary с pre-built index.

### 9.3. Pre-allocate capacity

```csharp
// ❌ Default capacity 4, расти будет несколько раз
var list = new List<int>();
for (int i = 0; i < 10000; i++) list.Add(i);

// ✅ Pre-allocate
var list2 = new List<int>(capacity: 10000);
```

Hot path optimizations.

### 9.4. ToList() vs ToArray()

```
| Operation         | Speed | Memory |
|-------------------|-------|--------|
| ToList()          |  +    |  +     |  (List<T> internal array + Count)
| ToArray()         |  ++   |  ++    |  (precise size, slightly less overhead)
```

`ToArray()` чуть efficient если знаешь, что не будешь добавлять. `ToList()` flexible.

### 9.5. Avoid `.Count()` для коллекций

```csharp
// ❌ Count() — может итерировать всё
var nums = source.Where(x => x > 5);
if (nums.Count() > 0) { }

// ✅ Any() — останавливается после первого
if (nums.Any()) { }

// ✅ ICollection / Array имеют Count property (O(1))
list.Count;     // O(1) — property, не method
array.Length;   // O(1)
```

`Count()` (extension) — может быть O(n). `Count` property на List/Array — O(1).

### 9.6. Avoid OrderBy перед Filter

```csharp
// ❌ Sort всё, потом filter
var result = users.OrderBy(u => u.Name).Where(u => u.IsActive).Take(10);

// ✅ Filter сначала
var result = users.Where(u => u.IsActive).OrderBy(u => u.Name).Take(10);
```

Filter уменьшает sort workload.

### 9.7. `Span<T>` для arrays

```csharp
// Вместо Substring (allocates)
ReadOnlySpan<int> span = array.AsSpan();
ReadOnlySpan<int> slice = span[5..10];   // no allocation
```

Для arrays / strings — Span (см. strings-regex notes).

### 9.8. FrozenDictionary для read-only

```csharp
// .NET 8+ — preprocessed для faster lookup
private static readonly FrozenDictionary<string, int> _lookup =
    new Dictionary<string, int> { ["a"] = 1, ["b"] = 2 }
        .ToFrozenDictionary();

// Lookups ~30% faster than regular Dictionary
_lookup["a"];
```

Для config / lookup tables: init at startup, не меняется. Механизм ускорения (анализ ключей, специализированные реализации, `GetAlternateLookup` без аллокаций) — §5.7.

> [!question]- Интервью: чем `Count()` отличается от `Count` property?
> `Count()` — LINQ extension method, может быть O(n) (итерирует через всю collection). На `ICollection<T>` / `IReadOnlyCollection<T>` оптимизирован — O(1). `Count` — property на `List<T>`, `Array.Length`, `Dictionary<K,V>.Count` — всегда O(1). Используй property когда тип concrete (List, Array). `Count()` LINQ — для general `IEnumerable<T>`, может быть медленным для lazy queries. Для существования — `Any()` (останавливается на первом) лучше `Count() > 0`.

---

## 10. Best Practices

### 10.1. Collections selection

- ✅ **`List<T>`** — default, sequential данные.
- ✅ **`Dictionary<K,V>`** — lookup by key.
- ✅ **`HashSet<T>`** — unique items, membership check.
- ✅ **`Queue<T>` / `Stack<T>`** — FIFO / LIFO.
- ✅ **`ConcurrentDictionary`** — multi-threaded.
- ✅ **`FrozenDictionary`** — read-only frequent lookup.
- ❌ **ArrayList / Hashtable** — non-generic legacy.
- ❌ **LinkedList** — почти никогда (List лучше).

### 10.2. LINQ

- ✅ **Method syntax** для большинства cases.
- ✅ **`Any()`** instead `Count() > 0`.
- ✅ **`FirstOrDefault`** для optional.
- ✅ **`Single`** когда expect exactly one.
- ✅ **Materialize** через `ToList` если используется много раз.
- ✅ **Filter перед sort/group** — reduce work.
- ❌ **Multiple iteration** lazy query без materialize.
- ❌ **Side effects** в lambdas.
- ❌ **`OrderBy` без `ThenBy`** для multi-level (используй ThenBy).

### 10.3. Performance

- ✅ **Pre-allocate capacity** для known size.
- ✅ **`Span<T>`** для slicing arrays.
- ✅ **Generic collections** (no boxing).
- ✅ **Frozen** для read-only.
- ❌ **Non-generic** (ArrayList).
- ❌ **`Count()` в loop**.

### 10.4. Не делай

- ❌ Модификация collection в foreach.
- ❌ Mutable keys в Dictionary.
- ❌ Multiple iteration expensive lazy query.
- ❌ List для duplicate checks.
- ❌ Concurrent mutation без synchronization.

---

## 11. Decision tree

```
Что нужно?
│
├── Storage
│   ├── Indexed sequential → List<T>
│   ├── Key-value → Dictionary<K,V>
│   ├── Unique items → HashSet<T>
│   ├── FIFO → Queue<T>
│   ├── LIFO → Stack<T>
│   ├── Sorted by priority → PriorityQueue<T,P>
│   ├── Sorted by key → SortedDictionary<K,V>
│   ├── Multi-threaded → ConcurrentDictionary<K,V>
│   └── Read-only frequent → FrozenDictionary<K,V>
│
├── Query / Transform
│   ├── Filter → Where
│   ├── Project → Select / SelectMany
│   ├── Sort → OrderBy / ThenBy
│   ├── Group → GroupBy
│   ├── Aggregate → Sum / Count / Average / Aggregate
│   ├── First match → First / FirstOrDefault
│   ├── Existence → Any
│   ├── Distinct → Distinct / DistinctBy
│   └── Join → Join / GroupJoin
│
├── Materialization
│   ├── List → ToList()
│   ├── Array → ToArray()
│   ├── Dictionary → ToDictionary()
│   ├── HashSet → ToHashSet()
│   └── Lookup (multi-value dict) → ToLookup()
│
└── Performance hot path
    ├── Pre-allocate capacity
    ├── Span<T> для arrays
    ├── FrozenDictionary для read-only
    ├── Avoid .Count() — use property
    └── Filter before sort/group
```

---

## 12. Cheat sheet

```csharp
// === Collections ===
var list = new List<int> { 1, 2, 3 };
var dict = new Dictionary<string, int> { ["a"] = 1 };
var set = new HashSet<int> { 1, 2, 3 };
var queue = new Queue<int>();
var stack = new Stack<int>();
var pq = new PriorityQueue<string, int>();   // .NET 6+

// Collection expressions (C# 12+)
List<int> nums = [1, 2, 3];
int[] arr = [1, 2, 3];

// === LINQ basics ===
var result = users
    .Where(u => u.IsActive)               // filter
    .Select(u => u.Email)                  // project
    .OrderBy(e => e)                       // sort
    .Distinct()                            // unique
    .Take(10)                              // limit
    .ToList();                             // materialize

// === Aggregates ===
list.Count(x => x > 5);
list.Sum();
list.Min();
list.Max();
list.Average();
list.Aggregate(0, (acc, x) => acc + x);

// === Element access ===
list.First();          // или throw
list.FirstOrDefault();  // или default
list.Single();          // exactly one
list.Single(x => x.Id == 42);

// === Quantifiers ===
list.Any();
list.Any(x => x > 5);
list.All(x => x > 0);
list.Contains(5);

// === Set ops ===
a.Union(b);
a.Intersect(b);
a.Except(b);

// === Grouping ===
var byCity = users.GroupBy(u => u.City);
foreach (var g in byCity) { /* g.Key, g.Count() */ }

// === Joins ===
users.Join(orders, u => u.Id, o => o.UserId, (u, o) => new { u.Name, o.Total });

// === Conversions ===
list.ToList();
list.ToArray();
list.ToDictionary(x => x.Id);
list.ToHashSet();
list.ToLookup(x => x.City);

// === Enumerable.Range / Repeat ===
Enumerable.Range(1, 100);
Enumerable.Repeat("x", 5);

// === Frozen (.NET 8+) ===
var frozen = dict.ToFrozenDictionary();
```

| Сценарий | Решение |
|----------|---------|
| Sequential storage | `List<T>` |
| Lookup by key | `Dictionary<K,V>` |
| Unique items | `HashSet<T>` |
| FIFO / LIFO | `Queue<T>` / `Stack<T>` |
| Sorted | `SortedDictionary` / `SortedSet` / `PriorityQueue` |
| Multi-threaded | `ConcurrentDictionary` |
| Read-only fast | `FrozenDictionary` (.NET 8+) |
| Filter | `.Where(predicate)` |
| Transform | `.Select(selector)` |
| Sort | `.OrderBy().ThenBy()` |
| Aggregate | `.Sum/Count/Min/Max/Average/Aggregate` |
| Group | `.GroupBy()` |
| First match | `.FirstOrDefault()` |
| Existence | `.Any()` |
| Materialize | `.ToList()` / `.ToArray()` |

---

## 13. Common pitfalls

### 13.1. Modify collection во время foreach

```csharp
// ❌ InvalidOperationException
foreach (var x in list)
    if (x % 2 == 0) list.Remove(x);
```

**Фикс:** `RemoveAll(predicate)` или iterate через index end-to-start.

### 13.2. Mutable keys в Dictionary

```csharp
public class User { public int Id { get; set; } }
var dict = new Dictionary<User, string> { [user] = "x" };
user.Id = 99;   // hash меняется
dict[user];     // KeyNotFoundException!
```

**Фикс:** immutable keys (records, value types, readonly properties).

### 13.3. Multiple iteration of lazy query

```csharp
var query = source.Where(...);   // lazy
var count = query.Count();        // executes
var first = query.First();         // executes снова
```

**Фикс:** materialize — `var list = query.ToList()`.

### 13.4. `Count() > 0` вместо `Any()`

```csharp
if (list.Count() > 0) { }   // потенциально O(n)
```

**Фикс:** `if (list.Any()) { }` — останавливается после first.

### 13.5. List.Contains для membership

```csharp
foreach (var x in items)
    if (!seen.Contains(x))   // O(n) per check
        seen.Add(x);
// Total O(n²)
```

**Фикс:** HashSet — O(1) Contains.

### 13.6. Default capacity для known size

```csharp
var list = new List<int>();   // default capacity 4
for (int i = 0; i < 1_000_000; i++) list.Add(i);   // ~20 reallocations
```

**Фикс:** `new List<int>(capacity: 1_000_000)`.

### 13.7. ContainsValue для частых checks

```csharp
dict.ContainsValue("x");   // O(n) — линейный scan!
```

**Фикс:** invert dictionary или использовать второй HashSet.

### 13.8. Boxing в ArrayList

```csharp
ArrayList list = new();   // legacy, non-generic
list.Add(42);             // boxes int → object
int x = (int)list[0];      // unbox
```

**Фикс:** `List<int>` — generic, no boxing.

### 13.9. `OrderBy().OrderBy()` вместо `OrderBy().ThenBy()`

```csharp
// ❌ Только последний OrderBy эффективен
list.OrderBy(x => x.LastName).OrderBy(x => x.FirstName);

// ✅ ThenBy для secondary
list.OrderBy(x => x.LastName).ThenBy(x => x.FirstName);
```

### 13.10. Side effects в LINQ lambdas

```csharp
var query = users.Where(u =>
{
    Log(u);          // ❌ side effect — повторяется при каждом iteration
    return u.IsActive;
});
```

**Фикс:** materialize или обработка в foreach.

> [!question]- Интервью: топ-3 LINQ ловушки?
> 1) **Multiple iteration of lazy query** — каждый foreach/Count/First запускает pipeline снова. Materialize через ToList. 2) **Side effects в lambdas** — Log/DB call в Where повторяется при каждом iteration. Вынести в foreach. 3) **`Count() > 0` вместо `Any()`** — Any останавливается на первом match, Count считает все. Любая operation которая может early-exit лучше через quantifier.

---

## 14. Practice — упражнения

### 14.1. Group и aggregate

**Задача.** Сгруппировать orders по customer и посчитать total для каждого.

```csharp
public record Order(int Id, int CustomerId, decimal Total);

var orders = new List<Order>
{
    new(1, 100, 50m),
    new(2, 100, 75m),
    new(3, 200, 100m),
    new(4, 100, 25m),
    new(5, 200, 200m)
};

var summary = orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new
    {
        CustomerId = g.Key,
        OrderCount = g.Count(),
        Total = g.Sum(o => o.Total),
        Average = g.Average(o => o.Total)
    })
    .OrderByDescending(s => s.Total)
    .ToList();

foreach (var s in summary)
    Console.WriteLine($"Customer {s.CustomerId}: {s.OrderCount} orders, total {s.Total:F2}");
```

### 14.2. Custom IEqualityComparer

**Задача.** Distinct users по email (case-insensitive).

```csharp
public record User(int Id, string Email);

public class UserEmailComparer : IEqualityComparer<User>
{
    public bool Equals(User? x, User? y) =>
        string.Equals(x?.Email, y?.Email, StringComparison.OrdinalIgnoreCase);
    public int GetHashCode(User obj) =>
        obj.Email.ToLowerInvariant().GetHashCode();
}

var users = new[]
{
    new User(1, "alice@x.com"),
    new User(2, "ALICE@x.com"),
    new User(3, "bob@x.com")
};

var unique = users.Distinct(new UserEmailComparer()).ToList();
// .NET 6+: DistinctBy
var unique2 = users.DistinctBy(u => u.Email.ToLowerInvariant()).ToList();
```

### 14.3. Pivot — flat to nested

**Задача.** Из flat orders построить nested customer → orders.

```csharp
public record Customer(int Id, string Name);
public record Order(int Id, int CustomerId, decimal Total);

var customers = new List<Customer> { new(1, "Alice"), new(2, "Bob") };
var orders = new List<Order> { new(1, 1, 100m), new(2, 1, 50m), new(3, 2, 200m) };

var result = customers
    .GroupJoin(
        orders,
        c => c.Id,
        o => o.CustomerId,
        (c, os) => new
        {
            c.Name,
            Orders = os.OrderByDescending(o => o.Total).ToList()
        }
    )
    .ToList();
```

### 14.4. Top N per group

**Задача.** Топ 3 самых дорогих orders по каждому customer.

```csharp
var top3PerCustomer = orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new
    {
        CustomerId = g.Key,
        TopOrders = g.OrderByDescending(o => o.Total).Take(3).ToList()
    })
    .ToList();
```

### 14.5. Refactor non-LINQ в LINQ

```csharp
// До — imperative
var result = new List<string>();
foreach (var u in users)
{
    if (u.IsActive && u.Age > 18)
    {
        result.Add(u.Email);
    }
}
result.Sort();
var top10 = result.Take(10).ToList();

// После — LINQ
var top10 = users
    .Where(u => u.IsActive && u.Age > 18)
    .Select(u => u.Email)
    .OrderBy(e => e)
    .Take(10)
    .ToList();
```

---

## 15. Что читать дальше

1. **[[csharp-basics|C# Basics]]** — типы, generics.
2. **[[iterators-yield|Iterators и yield]]** — как LINQ работает изнутри.
3. **[[generics-deep|Generics deep]]** — generic constraints.
4. **[[anonymous-types|Anonymous Types]]** — для projections.
5. **[[delegates-events|Delegates]]** — Func/Action для LINQ.
6. **EF Core LINQ** — translation в SQL.
7. **System.Threading.Channels** — для streaming.
8. **Reactive Extensions (Rx)** — push-based streams.

---

## 16. См. также

- [[csharp-basics|C# Basics]]
- [[iterators-yield|Iterators и yield]] — yield + lazy
- [[generics-deep|Generics deep]]
- [[anonymous-types|Anonymous Types]] — projections
- [[delegates-events|Delegates]] — Func/Action
- [[strings-regex|Strings]] — Span для текста
- EF Core LINQ
- `Channel<T>` deep
- Rx.NET

---

## 17. Reading list

- **Microsoft Docs — LINQ** — learn.microsoft.com/dotnet/csharp/programming-guide/concepts/linq
- **Microsoft Docs — Collections** — learn.microsoft.com/dotnet/standard/collections/
- **Microsoft Docs — System.Linq.Enumerable** — learn.microsoft.com/dotnet/api/system.linq.enumerable
- **Microsoft Docs — FrozenDictionary** — learn.microsoft.com/dotnet/api/system.collections.frozen
- **Jon Skeet — C# in Depth (LINQ chapter)**
- **Stephen Cleary — IAsyncEnumerable patterns** — blog.stephencleary.com
- **Adam Sitnik — Performance benchmarks** — adamsitnik.com
- **Andrew Lock — LINQ patterns** — andrewlock.net
- **Bart de Smet — More LINQ patterns** — bartdesmet.net
- **MoreLINQ library** — github.com/morelinq/MoreLINQ
- **System.Linq.Async** — github.com/dotnet/reactive
- **Reactive Extensions (Rx.NET)** — github.com/dotnet/reactive
- **BenchmarkDotNet** — benchmarkdotnet.org
