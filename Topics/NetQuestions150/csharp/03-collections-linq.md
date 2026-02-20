# Collections и LINQ

## HashSet, Dictionary, ToLookup

### HashSet&lt;T&gt;

Множество уникальных элементов. O(1) для Add, Remove, Contains. Основан на хэш-таблице.

```csharp
var ids = new HashSet<int> { 1, 2, 3 };
ids.Add(2);          // false — уже есть
ids.Contains(3);     // true — O(1)

// Операции над множествами
ids.UnionWith(other);        // объединение
ids.IntersectWith(other);    // пересечение
ids.ExceptWith(other);       // разность
ids.IsSubsetOf(other);       // подмножество?
```

**Нюанс:** для корректной работы тип `T` должен правильно реализовывать `GetHashCode()` и `Equals()`. Для record это автоматически. Для class — нужно переопределять или использовать `IEqualityComparer<T>`.

### Dictionary&lt;TKey, TValue&gt;

O(1) для доступа по ключу. Дубликаты ключей запрещены.

```csharp
var dict = new Dictionary<string, int>(capacity: 100); // указать capacity

// Безопасный доступ
if (dict.TryGetValue("key", out var value)) { }

// .NET 6+ — TryAdd, GetValueOrDefault
dict.TryAdd("key", 42);                  // false если ключ есть
var val = dict.GetValueOrDefault("key");  // default если нет

// CollectionsMarshal.GetValueRefOrAddDefault — zero-copy update
ref var entry = ref CollectionsMarshal.GetValueRefOrAddDefault(dict, "key", out bool exists);
entry += 10; // изменяем значение без повторного хэширования
```

### ToLookup

`ILookup<TKey, TElement>` — immutable, ключ → коллекция значений. Материализуется сразу.

```csharp
var lookup = orders.ToLookup(o => o.CustomerId);
IEnumerable<Order> customerOrders = lookup[customerId]; // пустая коллекция если нет
```

**Отличие от GroupBy:** `GroupBy` — ленивый (deferred), `ToLookup` — материализованный (immediate). `ToLookup` безопасен для повторного перечисления.

---

## IEnumerable vs IQueryable

| Аспект | IEnumerable&lt;T&gt; | IQueryable&lt;T&gt; |
|--------|---------------------|---------------------|
| Выполнение | В памяти (C#) | У провайдера (SQL) |
| Представление | Делегат | Expression Tree |
| Фильтрация | В памяти | На стороне БД |
| Провайдер | Нет | IQueryProvider |

```csharp
// IQueryable — фильтр выполняется в SQL
IQueryable<Order> query = context.Orders
    .Where(o => o.Total > 100);  // WHERE Total > 100 в SQL

// IEnumerable — ВСЕ данные загружаются, фильтр в памяти
IEnumerable<Order> all = context.Orders;
var filtered = all.Where(o => o.Total > 100); // фильтр в C#, не в SQL!
```

**Нюанс:** `IQueryable` наследует `IEnumerable`. Ошибка — вызвать `.AsEnumerable()` или `.ToList()` до фильтрации. Тогда фильтрация идёт в памяти, загружая всё из БД.

---

## Deferred Execution, yield, Cast

### Deferred Execution

Запрос создаёт pipeline, но не выполняется до перечисления (foreach, ToList, First, Count).

```csharp
var query = list.Where(x => x > 5).Select(x => x * 2); // ничего не выполнено
var result = query.ToList();  // выполняется здесь

// Опасность: множественное перечисление
IEnumerable<int> filtered = GetData().Where(x => x > 0);
var count = filtered.Count();  // перечисление 1
var first = filtered.First();  // перечисление 2 — данные могут измениться!
```

### yield return

Ленивая генерация последовательности. Компилятор создаёт state machine.

```csharp
public static IEnumerable<int> Fibonacci()
{
    int a = 0, b = 1;
    while (true)
    {
        yield return a;
        (a, b) = (b, a + b);
    }
}

// Берём только 10 первых — бесконечная последовательность безопасна
var first10 = Fibonacci().Take(10).ToList();
```

**Нюанс:** код до первого `yield return` выполняется при первом `MoveNext()`, не при вызове метода. Для eager валидации параметров — отдельный метод-обёртка.

### Cast и OfType

```csharp
// Cast<T> — приведение, бросает InvalidCastException если тип не совпадает
ArrayList legacy = new() { 1, 2, "oops" };
legacy.Cast<int>().ToList(); // InvalidCastException на "oops"

// OfType<T> — фильтрация по типу, пропускает несовпадения
legacy.OfType<int>().ToList(); // [1, 2] — "oops" пропущен
```

---

## Expression Trees

`Expression<Func<T, bool>>` — код как структура данных. EF Core анализирует дерево и генерирует SQL.

```csharp
// Делегат — скомпилированный код, «чёрный ящик»
Func<Order, bool> func = o => o.Total > 100;

// Expression — дерево, можно анализировать и преобразовывать
Expression<Func<Order, bool>> expr = o => o.Total > 100;
// expr.Body = BinaryExpression { Left = MemberAccess(Total), Right = Constant(100) }

// Динамическое построение фильтров
public static Expression<Func<T, bool>> And<T>(
    this Expression<Func<T, bool>> left,
    Expression<Func<T, bool>> right)
{
    var param = Expression.Parameter(typeof(T));
    var body = Expression.AndAlso(
        Expression.Invoke(left, param),
        Expression.Invoke(right, param));
    return Expression.Lambda<Func<T, bool>>(body, param);
}
```

**Нюанс:** нельзя использовать в Expression Tree: операторы присваивания, try-catch, await, dynamic. Только «чистые» выражения.

---

## Immutable, Frozen, Thread-safe коллекции

### Immutable (System.Collections.Immutable)

«Мутация» возвращает новый экземпляр. Structural sharing — новая версия разделяет данные со старой.

```csharp
var list = ImmutableList.Create(1, 2, 3);
var newList = list.Add(4);      // list не изменился, newList = [1,2,3,4]
var builder = list.ToBuilder();  // мутабельный builder для пакетных изменений
builder.Add(5);
var result = builder.ToImmutable();
```

### Frozen (.NET 8+)

`FrozenDictionary<TKey, TValue>`, `FrozenSet<T>` — оптимизированы для чтения. Создание медленнее, чтение быстрее обычного Dictionary.

```csharp
// Создать один раз, читать много
FrozenDictionary<string, int> cache = data.ToFrozenDictionary(x => x.Key, x => x.Value);
```

### Thread-safe (System.Collections.Concurrent)

```csharp
ConcurrentDictionary<string, int> dict = new();
dict.AddOrUpdate("key", 1, (_, old) => old + 1); // атомарный upsert

ConcurrentQueue<T>      // lock-free FIFO
ConcurrentBag<T>        // unordered, оптимизирован для producer=consumer
BlockingCollection<T>   // обёртка с блокировкой, bounded
Channel<T>              // async producer-consumer (предпочтительно)
```

**Нюанс:** `ConcurrentDictionary` — `GetOrAdd` не гарантирует однократный вызов factory. Для дорогих операций — `Lazy<T>` как значение: `dict.GetOrAdd(key, _ => new Lazy<T>(...)).Value`.

---

## LINQ Best Practices

```csharp
// ✗ Плохо — множественное перечисление
var data = GetExpensiveData();
if (data.Any()) Process(data.First()); // 2 перечисления

// ✓ Хорошо — одно перечисление
var first = GetExpensiveData().FirstOrDefault();
if (first is not null) Process(first);

// ✗ Плохо — Select + Where вместо одного прохода
items.Select(x => Transform(x)).Where(x => x != null)

// ✓ Хорошо — один проход с pattern matching
items.Select(x => Transform(x)).OfType<Result>()
```

---

## См. также

- [C# Reference: Коллекции и LINQ](../../../Reference/csharp-collections-linq.md)
