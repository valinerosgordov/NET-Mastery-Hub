---
tags: [collections, linq, dictionary, hashset, generics]
level: Junior
---

# Collections Рё LINQ

## Р§С‚Рѕ СЌС‚Рѕ, Р·Р°С‡РµРј Рё РєРѕРіРґР°

### Р§С‚Рѕ С‚Р°РєРѕРµ РєРѕР»Р»РµРєС†РёРё?
**РљРѕРЅС‚РµР№РЅРµСЂС‹ РґР»СЏ С…СЂР°РЅРµРЅРёСЏ РЅР°Р±РѕСЂР° РѕР±СЉРµРєС‚РѕРІ.** РљР°Р¶РґС‹Р№ С‚РёРї РєРѕР»Р»РµРєС†РёРё РѕРїС‚РёРјРёР·РёСЂРѕРІР°РЅ РїРѕРґ РѕРїСЂРµРґРµР»С‘РЅРЅС‹Рµ РѕРїРµСЂР°С†РёРё: РїРѕРёСЃРє, РІСЃС‚Р°РІРєР°, РїРѕСЂСЏРґРѕРє.

**РђРЅР°Р»РѕРіРёСЏ:** РљРѕР»Р»РµРєС†РёСЏ вЂ” СЌС‚Рѕ С€РєР°С„. `List` вЂ” РїРѕР»РєР° (РїРѕ РїРѕСЂСЏРґРєСѓ). `Dictionary` вЂ” СЏС‰РёРє СЃ РїРѕРґРїРёСЃСЏРјРё (Р±С‹СЃС‚СЂС‹Р№ РїРѕРёСЃРє РїРѕ РёРјРµРЅРё). `HashSet` вЂ” РјРµС€РѕРє СѓРЅРёРєР°Р»СЊРЅС‹С… РїСЂРµРґРјРµС‚РѕРІ. `Queue` вЂ” РѕС‡РµСЂРµРґСЊ РІ РјР°РіР°Р·РёРЅРµ.

### Р§С‚Рѕ С‚Р°РєРѕРµ LINQ?
**Language Integrated Query** вЂ” SQL-РїРѕРґРѕР±РЅС‹Р№ СЃРёРЅС‚Р°РєСЃРёСЃ РїСЂСЏРјРѕ РІ C#. Р¤РёР»СЊС‚СЂР°С†РёСЏ, СЃРѕСЂС‚РёСЂРѕРІРєР°, РіСЂСѓРїРїРёСЂРѕРІРєР° Р»СЋР±С‹С… РєРѕР»Р»РµРєС†РёР№ РѕРґРЅРёРј СЃС‚РёР»РµРј.

### РљРѕРіРґР° РєР°РєСѓСЋ РєРѕР»Р»РµРєС†РёСЋ?

| Р—Р°РґР°С‡Р° | РљРѕР»Р»РµРєС†РёСЏ | РџРѕС‡РµРјСѓ |
|--------|-----------|--------|
| РџРѕСЃР»РµРґРѕРІР°С‚РµР»СЊРЅС‹Р№ СЃРїРёСЃРѕРє, РґРѕСЃС‚СѓРї РїРѕ РёРЅРґРµРєСЃСѓ | **List\<T\>** | O(1) РїРѕ РёРЅРґРµРєСЃСѓ, O(n) РїРѕРёСЃРє |
| Р‘С‹СЃС‚СЂС‹Р№ РїРѕРёСЃРє РїРѕ РєР»СЋС‡Сѓ | **Dictionary\<K,V\>** | O(1) РїРѕРёСЃРє, РІСЃС‚Р°РІРєР°, СѓРґР°Р»РµРЅРёРµ |
| РЈРЅРёРєР°Р»СЊРЅС‹Рµ СЌР»РµРјРµРЅС‚С‹, РїСЂРѕРІРµСЂРєР° В«СЃРѕРґРµСЂР¶РёС‚ Р»Рё?В» | **HashSet\<T\>** | O(1) Contains |
| РћС‡РµСЂРµРґСЊ (РїРµСЂРІС‹Р№ РїСЂРёС€С‘Р» вЂ” РїРµСЂРІС‹Р№ РІС‹С€РµР») | **Queue\<T\>** | FIFO |
| РЎС‚РµРє (РїРѕСЃР»РµРґРЅРёР№ РїСЂРёС€С‘Р» вЂ” РїРµСЂРІС‹Р№ РІС‹С€РµР») | **Stack\<T\>** | LIFO |
| РќРµРёР·РјРµРЅСЏРµРјС‹Р№ СЃР»РѕРІР°СЂСЊ (read-only hot path) | **FrozenDictionary** | Р‘С‹СЃС‚СЂРµРµ Dictionary РґР»СЏ С‡С‚РµРЅРёСЏ |
| РџРѕС‚РѕРєРѕР±РµР·РѕРїР°СЃРЅР°СЏ РєРѕР»Р»РµРєС†РёСЏ | **ConcurrentDictionary** | Lock-free РґР»СЏ РјРЅРѕРіРѕРїРѕС‚РѕС‡РЅРѕСЃС‚Рё |
| РўРѕР»СЊРєРѕ РїРµСЂРµС‡РёСЃР»РµРЅРёРµ (lazy) | **IEnumerable\<T\>** | РћС‚Р»РѕР¶РµРЅРЅРѕРµ РІС‹РїРѕР»РЅРµРЅРёРµ, yield |

---

> РЎРїСЂР°РІРѕС‡РЅРёРє РїРѕ РєРѕР»Р»РµРєС†РёСЏРј, LINQ Рё generics. C# 13 / .NET 9.
> РўРµРѕСЂРёСЏ в†’ РїСЂР°РєС‚РёРєР° в†’ senior-level РєРѕРґ в†’ РІРѕРїСЂРѕСЃС‹ РёРЅС‚РµСЂРІСЊСЋ.

---

## РћР±Р·РѕСЂ РєРѕР»Р»РµРєС†РёР№

### РРµСЂР°СЂС…РёСЏ РёРЅС‚РµСЂС„РµР№СЃРѕРІ

```
IEnumerable<T>          вЂ” С‚РѕР»СЊРєРѕ РїРµСЂРµС‡РёСЃР»РµРЅРёРµ (foreach, yield)
  в””в”Ђ ICollection<T>     вЂ” Count, Add, Remove, Contains
       в””в”Ђ IList<T>      вЂ” РёРЅРґРµРєСЃР°С‚РѕСЂ [i], Insert, RemoveAt
       в””в”Ђ ISet<T>       вЂ” РѕРїРµСЂР°С†РёРё РЅР°Рґ РјРЅРѕР¶РµСЃС‚РІР°РјРё
  в””в”Ђ IReadOnlyCollection<T>
       в””в”Ђ IReadOnlyList<T>
       в””в”Ђ IReadOnlySet<T>
       в””в”Ђ IReadOnlyDictionary<TKey, TValue>
```

```csharp
// IEnumerable<T> вЂ” РјРёРЅРёРјР°Р»СЊРЅС‹Р№ РєРѕРЅС‚СЂР°РєС‚, Р»РµРЅРёРІРѕРµ РїРµСЂРµС‡РёСЃР»РµРЅРёРµ
IEnumerable<int> LazyRange(int count)
{
    for (var i = 0; i < count; i++)
        yield return i;
}

// ICollection<T> вЂ” РєРѕРіРґР° РЅСѓР¶РЅРѕ Р·РЅР°С‚СЊ Count Рё РёР·РјРµРЅСЏС‚СЊ РєРѕР»Р»РµРєС†РёСЋ
void Process(ICollection<string> items)
{
    Console.WriteLine(items.Count);
    items.Add("new");
    items.Remove("old");
}

// IList<T> вЂ” РєРѕРіРґР° РЅСѓР¶РµРЅ РґРѕСЃС‚СѓРї РїРѕ РёРЅРґРµРєСЃСѓ
void ProcessList(IList<Order> orders)
{
    var first = orders[0];
    orders.Insert(0, new Order());
    orders.RemoveAt(orders.Count - 1);
}
```

**РџСЂР°РІРёР»Рѕ РІС‹Р±РѕСЂР° РїР°СЂР°РјРµС‚СЂР° РјРµС‚РѕРґР°:** РїСЂРёРЅРёРјР°Р№ СЃР°РјС‹Р№ СѓР·РєРёР№ РёРЅС‚РµСЂС„РµР№СЃ, РєРѕС‚РѕСЂС‹Р№ С‚РµР±Рµ РЅСѓР¶РµРЅ. Р•СЃР»Рё РґРѕСЃС‚Р°С‚РѕС‡РЅРѕ РїРµСЂРµС‡РёСЃР»РёС‚СЊ вЂ” `IEnumerable<T>`. РќСѓР¶РµРЅ Count вЂ” `IReadOnlyCollection<T>`. РќСѓР¶РµРЅ РёРЅРґРµРєСЃ вЂ” `IReadOnlyList<T>`.

### IEnumerable vs IQueryable вЂ” РєР»СЋС‡РµРІРѕРµ СЂР°Р·Р»РёС‡РёРµ

```csharp
// IEnumerable<T> вЂ” РІС‹РїРѕР»РЅРµРЅРёРµ in-memory (LINQ to Objects)
// Р”РµР»РµРіР°С‚С‹: Func<T, bool>
IEnumerable<Order> orders = dbContext.Orders.AsEnumerable();
var filtered = orders.Where(o => o.Total > 1000); // С„РёР»СЊС‚СЂР°С†РёСЏ РІ РїР°РјСЏС‚Рё C#

// IQueryable<T> вЂ” С‚СЂР°РЅСЃР»СЏС†РёСЏ РІ РїСЂРѕРІР°Р№РґРµСЂ (SQL, MongoDB Рё С‚.Рґ.)
// Expression trees: Expression<Func<T, bool>>
IQueryable<Order> query = dbContext.Orders;
var filtered2 = query.Where(o => o.Total > 1000); // С‚СЂР°РЅСЃР»РёСЂСѓРµС‚СЃСЏ РІ SQL: WHERE Total > 1000
```

> **РљСЂРёС‚РёС‡РЅРѕ:** РќРµ РІС‹Р·С‹РІР°Р№ `.AsEnumerable()` РёР»Рё `.ToList()` РґРѕ С„РёРЅР°Р»СЊРЅРѕР№ С„РёР»СЊС‚СЂР°С†РёРё вЂ” РёРЅР°С‡Рµ РІСЃСЏ С‚Р°Р±Р»РёС†Р° Р·Р°РіСЂСѓР·РёС‚СЃСЏ РІ РїР°РјСЏС‚СЊ.

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: IEnumerable vs IQueryable вЂ” РєРѕРіРґР° С‡С‚Рѕ?**
> **IEnumerable** вЂ” РІС‹РїРѕР»РЅРµРЅРёРµ РІ РїР°РјСЏС‚Рё (LINQ to Objects). Р¤РёР»СЊС‚СЂР°С†РёСЏ РЅР° СЃС‚РѕСЂРѕРЅРµ РїСЂРёР»РѕР¶РµРЅРёСЏ.
>
> **IQueryable** вЂ” РґРµСЂРµРІРѕ РІС‹СЂР°Р¶РµРЅРёР№ в†’ SQL. Р’С‹РїРѕР»РЅРµРЅРёРµ РЅР° СЃРµСЂРІРµСЂРµ Р‘Р”. РСЃРїРѕР»СЊР·СѓР№ СЃ EF Core.
>
> **РџСЂР°РІРёР»Рѕ:** РЅРµ РІРѕР·РІСЂР°С‰Р°Р№ `IQueryable` РёР· repository РЅР°СЂСѓР¶Сѓ вЂ” СѓС‚РµС‡РєР° Р°Р±СЃС‚СЂР°РєС†РёРё. РњР°С‚РµСЂРёР°Р»РёР·СѓР№ С‡РµСЂРµР· `ToListAsync()`.

---

## Generic Collections

### List\<T\> вЂ” РѕСЃРЅРѕРІРЅР°СЏ РєРѕР»Р»РµРєС†РёСЏ

Р’РЅСѓС‚СЂРё вЂ” РјР°СЃСЃРёРІ `T[]`. РџСЂРё РїРµСЂРµРїРѕР»РЅРµРЅРёРё capacity СѓРґРІР°РёРІР°РµС‚СЃСЏ.

```csharp
// РЎРѕР·РґР°РЅРёРµ
var numbers = new List<int> { 1, 2, 3 };
var empty = new List<string>(capacity: 100); // РїСЂРµРґРІС‹РґРµР»РµРЅРёРµ вЂ” РјРµРЅСЊС€Рµ Р°Р»Р»РѕРєР°С†РёР№
List<int> fromRange = [1, 2, 3, 4, 5]; // collection expression (C# 12)

// РћСЃРЅРѕРІРЅС‹Рµ РјРµС‚РѕРґС‹
numbers.Add(4);
numbers.AddRange([5, 6, 7]);
numbers.Insert(0, 0);              // РІСЃС‚Р°РІРєР° РїРѕ РёРЅРґРµРєСЃСѓ вЂ” O(n)
numbers.Remove(3);                 // СѓРґР°Р»РµРЅРёРµ РїРµСЂРІРѕРіРѕ РІС…РѕР¶РґРµРЅРёСЏ вЂ” O(n)
numbers.RemoveAt(0);               // СѓРґР°Р»РµРЅРёРµ РїРѕ РёРЅРґРµРєСЃСѓ вЂ” O(n)
numbers.RemoveAll(x => x % 2 == 0); // СѓРґР°Р»РµРЅРёРµ РїРѕ РїСЂРµРґРёРєР°С‚Сѓ

// РџРѕРёСЃРє
var idx = numbers.IndexOf(5);
var found = numbers.Find(x => x > 3);         // РїРµСЂРІС‹Р№ РїРѕРґС…РѕРґСЏС‰РёР№ РёР»Рё default
var all = numbers.FindAll(x => x > 3);        // РІСЃРµ РїРѕРґС…РѕРґСЏС‰РёРµ
bool exists = numbers.Exists(x => x == 5);

// РЎРѕСЂС‚РёСЂРѕРІРєР°
numbers.Sort();                                // in-place, Span-based (Р±С‹СЃС‚СЂРѕ)
numbers.Sort((a, b) => b.CompareTo(a));       // РїРѕ СѓР±С‹РІР°РЅРёСЋ
numbers.Sort(Comparer<int>.Create((a, b) => a - b));

// РљРѕРЅРІРµСЂС‚Р°С†РёСЏ
int[] array = numbers.ToArray();
ReadOnlyCollection<int> ro = numbers.AsReadOnly();

// Capacity vs Count
Console.WriteLine($"Count: {numbers.Count}, Capacity: {numbers.Capacity}");
numbers.TrimExcess(); // СѓРјРµРЅСЊС€РёС‚СЊ Capacity РґРѕ Count
```

**РљРѕРіРґР° РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ:** РІ 90% СЃР»СѓС‡Р°РµРІ. Р‘С‹СЃС‚СЂС‹Р№ РґРѕСЃС‚СѓРї РїРѕ РёРЅРґРµРєСЃСѓ O(1), РґРѕР±Р°РІР»РµРЅРёРµ РІ РєРѕРЅРµС† O(1) amortized.

### Dictionary\<TKey, TValue\> вЂ” С…РµС€РёСЂРѕРІР°РЅРёРµ

```csharp
// РЎРѕР·РґР°РЅРёРµ
var dict = new Dictionary<string, int>
{
    ["apple"] = 5,
    ["banana"] = 3,
    ["cherry"] = 8
};

// Р‘РµР·РѕРїР°СЃРЅРѕРµ С‡С‚РµРЅРёРµ вЂ” TryGetValue (РїСЂРµРґРїРѕС‡С‚РёС‚РµР»СЊРЅС‹Р№ СЃРїРѕСЃРѕР±)
if (dict.TryGetValue("apple", out var count))
    Console.WriteLine($"Apple: {count}");

// GetValueOrDefault (.NET Core 2.0+) вЂ” СѓРґРѕР±РЅРѕ РґР»СЏ value types
int qty = dict.GetValueOrDefault("mango", defaultValue: 0);

// CollectionsMarshal.GetValueRefOrAddDefault вЂ” zero-copy РґРѕСЃС‚СѓРї
ref var val = ref System.Runtime.InteropServices.CollectionsMarshal
    .GetValueRefOrAddDefault(dict, "grape", out bool existed);
if (!existed) val = 10;
val += 5; // РёР·РјРµРЅСЏРµРј Р·РЅР°С‡РµРЅРёРµ in-place Р±РµР· РїРѕРІС‚РѕСЂРЅРѕРіРѕ lookup

// Р”РѕР±Р°РІР»РµРЅРёРµ
dict.Add("date", 2);                // ArgumentException РµСЃР»Рё РєР»СЋС‡ СѓР¶Рµ РµСЃС‚СЊ
dict.TryAdd("date", 99);            // false РµСЃР»Рё РєР»СЋС‡ РµСЃС‚СЊ, Р±РµР· РёСЃРєР»СЋС‡РµРЅРёР№
dict["elderberry"] = 1;             // РґРѕР±Р°РІРёС‚ РёР»Рё РїРµСЂРµР·Р°РїРёС€РµС‚

// РЈРґР°Р»РµРЅРёРµ
dict.Remove("banana");
dict.Remove("banana", out var removed); // РїРѕР»СѓС‡РёС‚СЊ СѓРґР°Р»С‘РЅРЅРѕРµ Р·РЅР°С‡РµРЅРёРµ

// РџРµСЂРµР±РѕСЂ
foreach (var (key, value) in dict)
    Console.WriteLine($"{key}: {value}");

// РЎРѕР·РґР°РЅРёРµ РёР· LINQ
var lookup = new[] { "a", "bb", "ccc", "dd" }
    .ToDictionary(s => s, s => s.Length);
```

**РЎР»РѕР¶РЅРѕСЃС‚СЊ:** Get/Set/Add/Remove вЂ” O(1) amortized. Р—Р°РІРёСЃРёС‚ РѕС‚ РєР°С‡РµСЃС‚РІР° `GetHashCode()`.

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: РљР°Рє СЂР°Р±РѕС‚Р°РµС‚ Dictionary? GetHashCode + Equals?**
> Dictionary РёСЃРїРѕР»СЊР·СѓРµС‚ С…РµС€ РґР»СЏ РІС‹Р±РѕСЂР° bucket-Р°. `GetHashCode()` в†’ bucket, `Equals()` в†’ СЂР°Р·СЂРµС€РµРЅРёРµ РєРѕР»Р»РёР·РёР№ РІРЅСѓС‚СЂРё bucket-Р°.
>
> **РљРѕРЅС‚СЂР°РєС‚:** РµСЃР»Рё `Equals(a,b) == true`, С‚Рѕ `GetHashCode(a) == GetHashCode(b)`. РќР°СЂСѓС€РµРЅРёРµ в†’ РїРѕС‚РµСЂСЏ СЌР»РµРјРµРЅС‚РѕРІ. РџР»РѕС…РѕРµ СЂР°СЃРїСЂРµРґРµР»РµРЅРёРµ в†’ РґРµРіСЂР°РґР°С†РёСЏ O(1) РґРѕ O(n).
>
> **РџСЂР°РєС‚РёРєР°:** РІСЃРµРіРґР° РїРµСЂРµРѕРїСЂРµРґРµР»СЏР№ РѕР±Р° РјРµС‚РѕРґР° РІРјРµСЃС‚Рµ. `HashCode.Combine()`. РќРµ РјСѓС‚Р°Р±РµР»СЊРЅС‹Рµ РїРѕР»СЏ РІ РєР»СЋС‡Р°С….

### HashSet\<T\> вЂ” СѓРЅРёРєР°Р»СЊРЅС‹Рµ СЌР»РµРјРµРЅС‚С‹

```csharp
var set = new HashSet<string>(StringComparer.OrdinalIgnoreCase)
{
    "Alice", "Bob", "Charlie"
};

// Р”РѕР±Р°РІР»РµРЅРёРµ вЂ” РІРµСЂРЅС‘С‚ false РµСЃР»Рё СЌР»РµРјРµРЅС‚ СѓР¶Рµ РµСЃС‚СЊ
bool added = set.Add("alice"); // false вЂ” OrdinalIgnoreCase

// РџСЂРѕРІРµСЂРєР°
bool contains = set.Contains("BOB"); // true

// РћРїРµСЂР°С†РёРё РЅР°Рґ РјРЅРѕР¶РµСЃС‚РІР°РјРё
var setA = new HashSet<int> { 1, 2, 3, 4, 5 };
var setB = new HashSet<int> { 3, 4, 5, 6, 7 };

setA.UnionWith(setB);          // {1,2,3,4,5,6,7} вЂ” РѕР±СЉРµРґРёРЅРµРЅРёРµ
setA.IntersectWith(setB);      // {3,4,5} вЂ” РїРµСЂРµСЃРµС‡РµРЅРёРµ
setA.ExceptWith(setB);         // {1,2} вЂ” СЂР°Р·РЅРѕСЃС‚СЊ
setA.SymmetricExceptWith(setB);// {1,2,6,7} вЂ” СЃРёРјРјРµС‚СЂРёС‡РµСЃРєР°СЏ СЂР°Р·РЅРѕСЃС‚СЊ

bool isSubset = setA.IsSubsetOf(setB);
bool overlaps = setA.Overlaps(setB);
bool equal = setA.SetEquals(setB);
```

### Queue\<T\> Рё Stack\<T\>

```csharp
// Queue вЂ” FIFO (РїРµСЂРІС‹Р№ РІРѕС€С‘Р» вЂ” РїРµСЂРІС‹Р№ РІС‹С€РµР»)
var queue = new Queue<string>();
queue.Enqueue("first");
queue.Enqueue("second");
queue.Enqueue("third");

string next = queue.Dequeue();    // "first"
string peek = queue.Peek();       // "second" (Р±РµР· СѓРґР°Р»РµРЅРёСЏ)
bool has = queue.TryDequeue(out var item); // Р±РµР·РѕРїР°СЃРЅС‹Р№ РІР°СЂРёР°РЅС‚

// Stack вЂ” LIFO (РїРѕСЃР»РµРґРЅРёР№ РІРѕС€С‘Р» вЂ” РїРµСЂРІС‹Р№ РІС‹С€РµР»)
var stack = new Stack<int>();
stack.Push(1);
stack.Push(2);
stack.Push(3);

int top = stack.Pop();           // 3
int peekTop = stack.Peek();      // 2
bool popped = stack.TryPop(out var val); // Р±РµР·РѕРїР°СЃРЅС‹Р№ РІР°СЂРёР°РЅС‚
```

### LinkedList\<T\> вЂ” РґРІСѓСЃРІСЏР·РЅС‹Р№ СЃРїРёСЃРѕРє

```csharp
var list = new LinkedList<string>();

var node1 = list.AddFirst("A");
var node2 = list.AddLast("C");
var node3 = list.AddAfter(node1, "B"); // A -> B -> C

// РќР°РІРёРіР°С†РёСЏ
LinkedListNode<string>? current = list.First;
while (current is not null)
{
    Console.Write($"{current.Value} -> ");
    current = current.Next;
}

// РЈРґР°Р»РµРЅРёРµ вЂ” O(1) РµСЃР»Рё РµСЃС‚СЊ СЃСЃС‹Р»РєР° РЅР° СѓР·РµР»
list.Remove(node3);
```

**РљРѕРіРґР° РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ:** С‡Р°СЃС‚С‹Рµ РІСЃС‚Р°РІРєРё/СѓРґР°Р»РµРЅРёСЏ РІ СЃРµСЂРµРґРёРЅРµ. РќР° РїСЂР°РєС‚РёРєРµ вЂ” РїРѕС‡С‚Рё РЅРёРєРѕРіРґР°. `List<T>` Р±С‹СЃС‚СЂРµРµ РёР·-Р·Р° cache locality.

### SortedSet\<T\>, SortedDictionary\<TKey, TValue\>

```csharp
// SortedSet вЂ” РІСЃРµРіРґР° РѕС‚СЃРѕСЂС‚РёСЂРѕРІР°РЅРЅС‹Р№ РЅР°Р±РѕСЂ СѓРЅРёРєР°Р»СЊРЅС‹С… СЌР»РµРјРµРЅС‚РѕРІ (Red-Black Tree)
var sorted = new SortedSet<int> { 5, 3, 8, 1, 9 };
// РџРµСЂРµР±РѕСЂ: 1, 3, 5, 8, 9

var range = sorted.GetViewBetween(3, 8); // {3, 5, 8} вЂ” РїРѕРґРјРЅРѕР¶РµСЃС‚РІРѕ
int min = sorted.Min; // 1
int max = sorted.Max; // 9

// SortedDictionary вЂ” РєР»СЋС‡Рё РІСЃРµРіРґР° РѕС‚СЃРѕСЂС‚РёСЂРѕРІР°РЅС‹ (Red-Black Tree)
var sortedDict = new SortedDictionary<string, int>
{
    ["banana"] = 3,
    ["apple"] = 5,
    ["cherry"] = 1
};
// РџРµСЂРµР±РѕСЂ: apple -> banana -> cherry (Р°Р»С„Р°РІРёС‚РЅС‹Р№ РїРѕСЂСЏРґРѕРє РєР»СЋС‡РµР№)
foreach (var (key, value) in sortedDict)
    Console.WriteLine($"{key}: {value}");
```

**РЎР»РѕР¶РЅРѕСЃС‚СЊ:** РІСЃРµ РѕРїРµСЂР°С†РёРё O(log n). РСЃРїРѕР»СЊР·СѓР№ РєРѕРіРґР° РЅСѓР¶РµРЅ РїРѕСЃС‚РѕСЏРЅРЅС‹Р№ РѕС‚СЃРѕСЂС‚РёСЂРѕРІР°РЅРЅС‹Р№ РїРѕСЂСЏРґРѕРє.

### PriorityQueue\<TElement, TPriority\> (.NET 6)

```csharp
var pq = new PriorityQueue<string, int>();

pq.Enqueue("Low priority task", priority: 10);
pq.Enqueue("Critical task", priority: 1);
pq.Enqueue("Medium task", priority: 5);

// РР·РІР»РµС‡РµРЅРёРµ вЂ” РІСЃРµРіРґР° СЌР»РµРјРµРЅС‚ СЃ РЅР°РёРјРµРЅСЊС€РёРј priority
while (pq.TryDequeue(out var task, out var priority))
    Console.WriteLine($"[{priority}] {task}");
// [1] Critical task
// [5] Medium task
// [10] Low priority task

// EnqueueDequeue вЂ” Р°С‚РѕРјР°СЂРЅРѕ: РґРѕР±Р°РІРёС‚СЊ Рё СЃСЂР°Р·Сѓ РёР·РІР»РµС‡СЊ РјРёРЅРёРјР°Р»СЊРЅС‹Р№
var dequeued = pq.EnqueueDequeue("New task", 3);
```

---

## Concurrent Collections

### ConcurrentDictionary\<TKey, TValue\>

```csharp
var cache = new ConcurrentDictionary<string, int>();

// РџРѕС‚РѕРєРѕР±РµР·РѕРїР°СЃРЅС‹Рµ РѕРїРµСЂР°С†РёРё
cache.TryAdd("key", 1);
cache.TryRemove("key", out var removed);

// GetOrAdd вЂ” РґРѕР±Р°РІРёС‚СЊ РµСЃР»Рё РЅРµС‚ (factory РјРѕР¶РµС‚ РІС‹Р·РІР°С‚СЊСЃСЏ РЅРµСЃРєРѕР»СЊРєРѕ СЂР°Р·!)
var value = cache.GetOrAdd("counter", key => ExpensiveComputation(key));

// AddOrUpdate вЂ” Р°С‚РѕРјР°СЂРЅРѕРµ РѕР±РЅРѕРІР»РµРЅРёРµ
cache.AddOrUpdate(
    key: "counter",
    addValue: 1,
    updateValueFactory: (key, oldValue) => oldValue + 1);

// Р’РђР–РќРћ: factory РІС‹Р·РѕРІС‹ РќР• Р°С‚РѕРјР°СЂРЅС‹ вЂ” С‚РѕР»СЊРєРѕ Р·Р°РїРёСЃСЊ Р°С‚РѕРјР°СЂРЅР°
// Р”Р»СЏ РґРѕСЂРѕРіРёС… factory РёСЃРїРѕР»СЊР·СѓР№ Lazy<T>:
var lazyCache = new ConcurrentDictionary<string, Lazy<int>>();
var result = lazyCache.GetOrAdd("key", k => new Lazy<int>(() => ExpensiveComputation(k)));
Console.WriteLine(result.Value);
```

### ConcurrentQueue\<T\>, ConcurrentBag\<T\>

```csharp
// ConcurrentQueue вЂ” РїРѕС‚РѕРєРѕР±РµР·РѕРїР°СЃРЅС‹Р№ FIFO
var cq = new ConcurrentQueue<WorkItem>();
cq.Enqueue(new WorkItem("task1"));
if (cq.TryDequeue(out var item))
    Process(item);

// ConcurrentBag вЂ” РЅРµСѓРїРѕСЂСЏРґРѕС‡РµРЅРЅР°СЏ РїРѕС‚РѕРєРѕР±РµР·РѕРїР°СЃРЅР°СЏ РєРѕР»Р»РµРєС†РёСЏ
// РћРїС‚РёРјРёР·РёСЂРѕРІР°РЅР° РґР»СЏ СЃС†РµРЅР°СЂРёСЏ: РєР°Р¶РґС‹Р№ РїРѕС‚РѕРє РґРѕР±Р°РІР»СЏРµС‚ Рё Р·Р°Р±РёСЂР°РµС‚ СЃРІРѕРё СЌР»РµРјРµРЅС‚С‹
var bag = new ConcurrentBag<int>();
Parallel.For(0, 100, i => bag.Add(i));
Console.WriteLine(bag.Count); // 100
```

### BlockingCollection\<T\>

```csharp
// Producer-consumer СЃ РѕРіСЂР°РЅРёС‡РµРЅРЅРѕР№ С‘РјРєРѕСЃС‚СЊСЋ
using var bc = new BlockingCollection<string>(boundedCapacity: 10);

// Producer (РІ РґСЂСѓРіРѕРј РїРѕС‚РѕРєРµ)
Task.Run(() =>
{
    for (var i = 0; i < 50; i++)
    {
        bc.Add($"item-{i}"); // Р±Р»РѕРєРёСЂСѓРµС‚СЃСЏ РµСЃР»Рё РєРѕР»Р»РµРєС†РёСЏ РїРѕР»РЅР°СЏ
    }
    bc.CompleteAdding(); // СЃРёРіРЅР°Р»: Р±РѕР»СЊС€Рµ РЅРµ Р±СѓРґРµС‚ СЌР»РµРјРµРЅС‚РѕРІ
});

// Consumer вЂ” GetConsumingEnumerable Р±Р»РѕРєРёСЂСѓРµС‚ РґРѕ CompleteAdding
foreach (var item in bc.GetConsumingEnumerable())
    Console.WriteLine(item);
```

### Channel\<T\> вЂ” modern async producer/consumer

```csharp
// РџСЂРµРґРїРѕС‡С‚РёС‚РµР»СЊРЅС‹Р№ РІР°СЂРёР°РЅС‚ РґР»СЏ async РєРѕРґР° (РІРјРµСЃС‚Рѕ BlockingCollection)
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

// Р—Р°РїСѓСЃРє
var cts = new CancellationTokenSource();
await Task.WhenAll(
    ProduceAsync(channel.Writer, cts.Token),
    ConsumeAsync(channel.Reader, cts.Token));
```

### Immutable Collections

```csharp
using System.Collections.Immutable;

// РљР°Р¶РґР°СЏ РѕРїРµСЂР°С†РёСЏ РІРѕР·РІСЂР°С‰Р°РµС‚ РќРћР’РЈР® РєРѕР»Р»РµРєС†РёСЋ вЂ” СЃС‚Р°СЂР°СЏ РЅРµ РјРµРЅСЏРµС‚СЃСЏ
var list = ImmutableList<int>.Empty;
var list2 = list.Add(1).Add(2).Add(3);
var list3 = list2.RemoveAt(0); // [2, 3]
// list2 РїРѕ-РїСЂРµР¶РЅРµРјСѓ [1, 2, 3]

// Builder вЂ” РґР»СЏ РјР°СЃСЃРѕРІС‹С… РјСѓС‚Р°С†РёР№ (СЌС„С„РµРєС‚РёРІРЅРµРµ С‡РµРј С†РµРїРѕС‡РєР° Add)
var builder = ImmutableList.CreateBuilder<string>();
builder.Add("A");
builder.Add("B");
builder.Add("C");
ImmutableList<string> immutable = builder.ToImmutable();

// ImmutableDictionary
var dict = ImmutableDictionary<string, int>.Empty
    .Add("x", 1)
    .Add("y", 2)
    .SetItem("x", 10); // РїРµСЂРµР·Р°РїРёСЃСЊ вЂ” РЅРѕРІС‹Р№ СЃР»РѕРІР°СЂСЊ

// ImmutableArray вЂ” РЅР°РёР±РѕР»РµРµ СЌС„С„РµРєС‚РёРІРЅР°СЏ immutable РєРѕР»Р»РµРєС†РёСЏ
var arr = ImmutableArray.Create(1, 2, 3);
var arr2 = arr.Add(4); // [1, 2, 3, 4]
```

---

## Frozen Collections (.NET 8)

РћРїС‚РёРјРёР·РёСЂРѕРІР°РЅС‹ РґР»СЏ **С‡С‚РµРЅРёСЏ**. РЎРѕР·РґР°СЋС‚СЃСЏ РѕРґРёРЅ СЂР°Р· вЂ” РїРѕС‚РѕРј С‚РѕР»СЊРєРѕ С‡РёС‚Р°СЋС‚СЃСЏ. Р‘С‹СЃС‚СЂРµРµ Dictionary/HashSet РґР»СЏ lookup.

```csharp
using System.Collections.Frozen;

// РЎРѕР·РґР°РЅРёРµ вЂ” РёР· СЃСѓС‰РµСЃС‚РІСѓСЋС‰РµР№ РєРѕР»Р»РµРєС†РёРё
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

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ вЂ” РѕР±С‹С‡РЅС‹Р№ API СЃР»РѕРІР°СЂСЏ
var val = frozen["GET"];
bool found = frozen.TryGetValue("POST", out var v);

// FrozenSet
FrozenSet<string> allowedMethods = new[] { "GET", "POST", "PUT", "DELETE" }
    .ToFrozenSet(StringComparer.OrdinalIgnoreCase);

bool allowed = allowedMethods.Contains("get"); // true
```

**РљРѕРіРґР° РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ:**
- РљРѕРЅС„РёРіСѓСЂР°С†РёСЏ, СЃРїСЂР°РІРѕС‡РЅРёРєРё, РјР°РїРїРёРЅРіРё, РєРѕС‚РѕСЂС‹Рµ РЅРµ РјРµРЅСЏСЋС‚СЃСЏ РїРѕСЃР»Рµ СЃС‚Р°СЂС‚Р°
- HTTP method routing, enum-to-string РјР°РїРїРёРЅРіРё
- Р”РѕСЂРѕР¶Рµ СЃРѕР·РґР°С‚СЊ, РґРµС€РµРІР»Рµ С‡РёС‚Р°С‚СЊ вЂ” РёРґРµР°Р»СЊРЅРѕ РґР»СЏ hot path

---

## Collection Expressions (C# 12)

```csharp
// Р›РёС‚РµСЂР°Р»С‹ РєРѕР»Р»РµРєС†РёР№ вЂ” РµРґРёРЅС‹Р№ СЃРёРЅС‚Р°РєСЃРёСЃ РґР»СЏ РІСЃРµС… С‚РёРїРѕРІ
int[] array = [1, 2, 3];
List<int> list = [1, 2, 3];
Span<int> span = [1, 2, 3];
ReadOnlySpan<int> roSpan = [1, 2, 3];
ImmutableArray<int> immArr = [1, 2, 3];
HashSet<int> set = [1, 2, 3];

// РџСѓСЃС‚Р°СЏ РєРѕР»Р»РµРєС†РёСЏ
List<string> empty = [];

// Spread operator вЂ” РѕР±СЉРµРґРёРЅРµРЅРёРµ РєРѕР»Р»РµРєС†РёР№
int[] first = [1, 2, 3];
int[] second = [4, 5, 6];
int[] combined = [..first, ..second]; // [1, 2, 3, 4, 5, 6]

// Spread СЃ С„РёР»СЊС‚СЂР°С†РёРµР№
int[] extras = [10, 20];
int[] all = [0, ..first, ..extras, 99]; // [0, 1, 2, 3, 10, 20, 99]

// Р’ РїР°СЂР°РјРµС‚СЂР°С… РјРµС‚РѕРґР°
PrintAll([1, 2, 3]);

void PrintAll(IEnumerable<int> items)
{
    foreach (var item in items)
        Console.Write($"{item} ");
}

// Target-typed new вЂ” РєРѕРјРїРёР»СЏС‚РѕСЂ РѕРїСЂРµРґРµР»СЏРµС‚ С‚РёРї
Dictionary<string, List<int>> map = new()
{
    ["odds"] = [1, 3, 5],
    ["evens"] = [2, 4, 6]
};
```

---

## LINQ вЂ” РћСЃРЅРѕРІС‹

### Method syntax vs Query syntax

```csharp
var products = GetProducts();

// Method syntax (РїСЂРµРґРїРѕС‡С‚РёС‚РµР»СЊРЅС‹Р№ вЂ” Р±РѕР»РµРµ РіРёР±РєРёР№)
var expensive = products
    .Where(p => p.Price > 100)
    .OrderByDescending(p => p.Price)
    .Select(p => new { p.Name, p.Price });

// Query syntax (СѓРґРѕР±РЅРµРµ РґР»СЏ join Рё let)
var expensiveQuery =
    from p in products
    where p.Price > 100
    orderby p.Price descending
    select new { p.Name, p.Price };

// Query syntax СЃ let вЂ” РїСЂРѕРјРµР¶СѓС‚РѕС‡РЅР°СЏ РїРµСЂРµРјРµРЅРЅР°СЏ
var discounted =
    from p in products
    let discountPrice = p.Price * 0.9m
    where discountPrice > 50
    select new { p.Name, Original = p.Price, Discounted = discountPrice };
```

### Deferred Execution (РѕС‚Р»РѕР¶РµРЅРЅРѕРµ РІС‹РїРѕР»РЅРµРЅРёРµ)

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// Р—Р°РїСЂРѕСЃ РќР• РІС‹РїРѕР»РЅСЏРµС‚СЃСЏ Р·РґРµСЃСЊ вЂ” СЌС‚Рѕ РѕРїРёСЃР°РЅРёРµ РѕРїРµСЂР°С†РёРё
var query = numbers.Where(n => n > 2);

numbers.Add(6); // РґРѕР±Р°РІР»СЏРµРј СЌР»РµРјРµРЅС‚ РџРћРЎР›Р• СЃРѕР·РґР°РЅРёСЏ Р·Р°РїСЂРѕСЃР°

// Р’С‹РїРѕР»РЅРµРЅРёРµ РїСЂРѕРёСЃС…РѕРґРёС‚ Р—Р”Р•РЎР¬ РїСЂРё РёС‚РµСЂР°С†РёРё
foreach (var n in query)
    Console.Write($"{n} "); // 3 4 5 6 вЂ” РІРєР»СЋС‡Р°СЏ 6!

// РћРџРђРЎРќРћРЎРўР¬: РјРЅРѕРіРѕРєСЂР°С‚РЅРѕРµ РїРµСЂРµС‡РёСЃР»РµРЅРёРµ
var filtered = numbers.Where(n => ExpensiveCheck(n));
var count = filtered.Count();    // 1-СЏ РёС‚РµСЂР°С†РёСЏ
var first = filtered.First();    // 2-СЏ РёС‚РµСЂР°С†РёСЏ вЂ” ExpensiveCheck РІС‹Р·РѕРІРµС‚СЃСЏ СЃРЅРѕРІР°!

// Р Р•РЁР•РќРР•: РјР°С‚РµСЂРёР°Р»РёР·Р°С†РёСЏ
var materialized = numbers.Where(n => ExpensiveCheck(n)).ToList();
var count2 = materialized.Count;   // Р±РµР· РїРѕРІС‚РѕСЂРЅРѕРіРѕ РІС‹С‡РёСЃР»РµРЅРёСЏ
var first2 = materialized[0];
```

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: yield return вЂ” РєР°Рє СЂР°Р±РѕС‚Р°РµС‚ Рё РѕРіСЂР°РЅРёС‡РµРЅРёСЏ?**
> РљРѕРјРїРёР»СЏС‚РѕСЂ РіРµРЅРµСЂРёСЂСѓРµС‚ state machine. РџСЂРё РєР°Р¶РґРѕРј `MoveNext()` вЂ” РїСЂРѕРґРѕР»Р¶РµРЅРёРµ СЃ С‚РѕС‡РєРё РїРѕСЃР»РµРґРЅРµРіРѕ `yield return`. Р­Р»РµРјРµРЅС‚С‹ СЃРѕР·РґР°СЋС‚СЃСЏ Р»РµРЅРёРІРѕ.
>
> **РћРіСЂР°РЅРёС‡РµРЅРёСЏ:** РЅРµР»СЊР·СЏ `yield return` РІРЅСѓС‚СЂРё `try` СЃ `catch`. РќРµР»СЊР·СЏ `ref`/`out` РїР°СЂР°РјРµС‚СЂС‹. `finally` РІС‹РїРѕР»РЅСЏРµС‚СЃСЏ РїСЂРё `Dispose` РїРµСЂРµС‡РёСЃР»РёС‚РµР»СЏ.
>
> **РљРѕРіРґР°:** СЃС‚СЂРёРјРёРЅРі Р±РѕР»СЊС€РёС… РґР°РЅРЅС‹С…, Р±РµСЃРєРѕРЅРµС‡РЅС‹Рµ РїРѕСЃР»РµРґРѕРІР°С‚РµР»СЊРЅРѕСЃС‚Рё, РєР°СЃС‚РѕРјРЅР°СЏ Р»РѕРіРёРєР° РїРµСЂРµС‡РёСЃР»РµРЅРёСЏ.

### Immediate Execution (РЅРµРјРµРґР»РµРЅРЅРѕРµ РІС‹РїРѕР»РЅРµРЅРёРµ)

```csharp
var items = Enumerable.Range(1, 100);

// РњР°С‚РµСЂРёР°Р»РёР·Р°С†РёСЏ РІ РєРѕРЅРєСЂРµС‚РЅСѓСЋ РєРѕР»Р»РµРєС†РёСЋ
List<int> list = items.ToList();
int[] array = items.ToArray();
Dictionary<int, string> dict = items.ToDictionary(i => i, i => $"item-{i}");
HashSet<int> set = items.ToHashSet();
ILookup<bool, int> lookup = items.ToLookup(i => i % 2 == 0); // РіСЂСѓРїРїРёСЂРѕРІРєР°

// РђРіСЂРµРіР°С†РёСЏ вЂ” С‚РѕР¶Рµ РЅРµРјРµРґР»РµРЅРЅРѕРµ
int count = items.Count();
int sum = items.Sum();
int max = items.Max();
```

---

## LINQ вЂ” РћРїРµСЂР°С‚РѕСЂС‹

### Р¤РёР»СЊС‚СЂР°С†РёСЏ

```csharp
var people = GetPeople();

// Where вЂ” С„РёР»СЊС‚СЂР°С†РёСЏ РїРѕ РїСЂРµРґРёРєР°С‚Сѓ
var adults = people.Where(p => p.Age >= 18);

// Where СЃ РёРЅРґРµРєСЃРѕРј
var firstThreeExpensive = products
    .Where((p, index) => p.Price > 100 && index < 3);

// OfType вЂ” С„РёР»СЊС‚СЂР°С†РёСЏ РїРѕ С‚РёРїСѓ (Р±РµР·РѕРїР°СЃРЅС‹Р№ cast)
object[] mixed = [1, "hello", 2.5, "world", 42];
IEnumerable<string> strings = mixed.OfType<string>(); // "hello", "world"

// OfType РїРѕР»РµР·РµРЅ РґР»СЏ РёРµСЂР°СЂС…РёР№ С‚РёРїРѕРІ
var animals = GetAnimals();
var dogs = animals.OfType<Dog>(); // С‚РѕР»СЊРєРѕ Dog, Р±РµР· InvalidCastException
```

### РџСЂРѕРµРєС†РёСЏ

```csharp
// Select вЂ” С‚СЂР°РЅСЃС„РѕСЂРјР°С†РёСЏ РєР°Р¶РґРѕРіРѕ СЌР»РµРјРµРЅС‚Р°
var names = people.Select(p => p.Name);
var dtos = people.Select(p => new PersonDto
{
    FullName = $"{p.FirstName} {p.LastName}",
    Age = p.Age
});

// Select СЃ РёРЅРґРµРєСЃРѕРј
var indexed = people.Select((p, i) => new { Index = i, p.Name });

// SelectMany вЂ” flatten РІР»РѕР¶РµРЅРЅС‹С… РєРѕР»Р»РµРєС†РёР№
var departments = GetDepartments(); // РєР°Р¶РґС‹Р№ РѕС‚РґРµР» СЃРѕРґРµСЂР¶РёС‚ List<Employee>

// Р‘РµР· SelectMany:
IEnumerable<IEnumerable<Employee>> nested = departments.Select(d => d.Employees);

// РЎ SelectMany вЂ” РїР»РѕСЃРєРёР№ СЃРїРёСЃРѕРє РІСЃРµС… СЃРѕС‚СЂСѓРґРЅРёРєРѕРІ:
IEnumerable<Employee> allEmployees = departments.SelectMany(d => d.Employees);

// SelectMany СЃ СЂРµР·СѓР»СЊС‚РёСЂСѓСЋС‰РёРј СЃРµР»РµРєС‚РѕСЂРѕРј
var empWithDept = departments.SelectMany(
    d => d.Employees,
    (dept, emp) => new { dept.Name, Employee = emp.FullName });

// РџСЂРёРјРµСЂ: СЂР°Р·Р±РёС‚СЊ СЃС‚СЂРѕРєРё РЅР° СЃР»РѕРІР°
string[] sentences = ["Hello world", "Foo bar baz"];
var words = sentences.SelectMany(s => s.Split(' ')); // ["Hello","world","Foo","bar","baz"]
```

### РЎРѕСЂС‚РёСЂРѕРІРєР°

```csharp
var orders = GetOrders();

// OrderBy / OrderByDescending
var byDate = orders.OrderBy(o => o.CreatedAt);
var byDateDesc = orders.OrderByDescending(o => o.CreatedAt);

// ThenBy / ThenByDescending вЂ” РІС‚РѕСЂРёС‡РЅР°СЏ СЃРѕСЂС‚РёСЂРѕРІРєР°
var sorted = orders
    .OrderBy(o => o.Status)
    .ThenByDescending(o => o.Total)
    .ThenBy(o => o.CustomerName);

// Order / OrderDescending (.NET 7) вЂ” Р±РµР· keySelector, РґР»СЏ РїСЂРѕСЃС‚С‹С… С‚РёРїРѕРІ
int[] nums = [5, 3, 8, 1, 9];
var asc = nums.Order();           // [1, 3, 5, 8, 9]
var desc = nums.OrderDescending(); // [9, 8, 5, 3, 1]

// Reverse
var reversed = orders.OrderBy(o => o.Id).Reverse();
```

### Р“СЂСѓРїРїРёСЂРѕРІРєР°

```csharp
var transactions = GetTransactions();

// GroupBy вЂ” РіСЂСѓРїРїРёСЂРѕРІРєР° РїРѕ РєР»СЋС‡Сѓ
var byCategory = transactions.GroupBy(t => t.Category);

foreach (var group in byCategory)
{
    Console.WriteLine($"Category: {group.Key}, Count: {group.Count()}");
    foreach (var tx in group)
        Console.WriteLine($"  {tx.Description}: {tx.Amount:C}");
}

// GroupBy СЃ СЂРµР·СѓР»СЊС‚РёСЂСѓСЋС‰РёРј СЃРµР»РµРєС‚РѕСЂРѕРј
var summary = transactions.GroupBy(
    t => t.Category,
    (category, items) => new
    {
        Category = category,
        Total = items.Sum(i => i.Amount),
        Count = items.Count()
    });

// GroupBy СЃ elementSelector
var namesByCity = people.GroupBy(
    p => p.City,
    p => p.Name); // IGrouping<string, string>

// ToLookup вЂ” РЅРµРјРµРґР»РµРЅРЅР°СЏ РјР°С‚РµСЂРёР°Р»РёР·Р°С†РёСЏ РіСЂСѓРїРїРёСЂРѕРІРєРё
ILookup<string, Transaction> lookup = transactions.ToLookup(t => t.Category);
var foodTx = lookup["Food"]; // РїСѓСЃС‚Р°СЏ РєРѕР»Р»РµРєС†РёСЏ РµСЃР»Рё РєР»СЋС‡Р° РЅРµС‚ (РЅРµ РёСЃРєР»СЋС‡РµРЅРёРµ!)

// CountBy (.NET 9) вЂ” РїРѕРґСЃС‡С‘С‚ РїРѕ РєР»СЋС‡Сѓ
IEnumerable<KeyValuePair<string, int>> counts = transactions.CountBy(t => t.Category);
foreach (var (category, count) in counts)
    Console.WriteLine($"{category}: {count}");

// AggregateBy (.NET 9) вЂ” Р°РіСЂРµРіР°С†РёСЏ РїРѕ РєР»СЋС‡Сѓ
var totalsByCategory = transactions.AggregateBy(
    t => t.Category,
    seed: 0m,
    (total, tx) => total + tx.Amount);
```

### РЎРѕРµРґРёРЅРµРЅРёРµ

```csharp
var customers = GetCustomers();
var orders = GetOrders();

// Join вЂ” inner join
var customerOrders = customers.Join(
    orders,
    c => c.Id,          // outer key
    o => o.CustomerId,   // inner key
    (c, o) => new { Customer = c.Name, o.Total, o.CreatedAt });

// РўРѕ Р¶Рµ СЃР°РјРѕРµ РІ query syntax (С‡РёС‚Р°Р±РµР»СЊРЅРµРµ РґР»СЏ join)
var customerOrdersQuery =
    from c in customers
    join o in orders on c.Id equals o.CustomerId
    select new { Customer = c.Name, o.Total, o.CreatedAt };

// GroupJoin вЂ” left join (РѕРґРёРЅ-РєРѕ-РјРЅРѕРіРёРј)
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

// Left join С‡РµСЂРµР· query syntax + DefaultIfEmpty
var leftJoin =
    from c in customers
    join o in orders on c.Id equals o.CustomerId into orderGroup
    from o in orderGroup.DefaultIfEmpty()
    select new
    {
        Customer = c.Name,
        OrderTotal = o?.Total ?? 0
    };

// Zip вЂ” СЃРѕРµРґРёРЅРµРЅРёРµ РїРѕ РїРѕР·РёС†РёРё
var names2 = new[] { "Alice", "Bob", "Charlie" };
var scores = new[] { 95, 87, 92 };
var zipped = names2.Zip(scores, (name, score) => $"{name}: {score}");
// "Alice: 95", "Bob: 87", "Charlie: 92"

// Zip Р±РµР· selector (.NET Core 3.0+) вЂ” РІРѕР·РІСЂР°С‰Р°РµС‚ ValueTuple
var tuples = names2.Zip(scores); // (Alice, 95), (Bob, 87), (Charlie, 92)
```

### РђРіСЂРµРіР°С†РёСЏ

```csharp
var values = new[] { 10, 20, 30, 40, 50 };

int count = values.Count();
int countFiltered = values.Count(v => v > 25); // 3
long longCount = values.LongCount();

int sum = values.Sum();
double avg = values.Average();
int min = values.Min();
int max = values.Max();

// MinBy / MaxBy (.NET 6) вЂ” РІРѕР·РІСЂР°С‰Р°РµС‚ СЌР»РµРјРµРЅС‚, Р° РЅРµ Р·РЅР°С‡РµРЅРёРµ РєР»СЋС‡Р°
var cheapest = products.MinBy(p => p.Price);   // Product, РЅРµ decimal
var mostExpensive = products.MaxBy(p => p.Price);

// Aggregate вЂ” СѓРЅРёРІРµСЂСЃР°Р»СЊРЅР°СЏ СЃРІС‘СЂС‚РєР° (reduce/fold)
int product = values.Aggregate((acc, x) => acc * x); // 10*20*30*40*50

// Aggregate СЃ seed
string csv = values.Aggregate(
    seed: new StringBuilder(),
    (sb, val) => sb.Length == 0 ? sb.Append(val) : sb.Append(',').Append(val),
    sb => sb.ToString()); // "10,20,30,40,50"
```

### Р­Р»РµРјРµРЅС‚

```csharp
var items = new[] { 10, 20, 30, 40, 50 };

// First вЂ” РїРµСЂРІС‹Р№ СЌР»РµРјРµРЅС‚ (InvalidOperationException РµСЃР»Рё РїСѓСЃС‚Рѕ)
int first = items.First();
int firstOver30 = items.First(x => x > 30); // 40

// FirstOrDefault вЂ” default(T) РµСЃР»Рё РїСѓСЃС‚Рѕ
int firstOrZero = items.FirstOrDefault(x => x > 100); // 0
int firstOrNeg = items.FirstOrDefault(x => x > 100, defaultValue: -1); // -1 (.NET 6)

// Single вЂ” СЂРѕРІРЅРѕ РѕРґРёРЅ СЌР»РµРјРµРЅС‚ (РёСЃРєР»СЋС‡РµРЅРёРµ РµСЃР»Рё 0 РёР»Рё >1)
int single = items.Where(x => x == 30).Single();

// SingleOrDefault вЂ” 0 РёР»Рё 1 СЌР»РµРјРµРЅС‚ (РёСЃРєР»СЋС‡РµРЅРёРµ РµСЃР»Рё >1)
int singleOr = items.SingleOrDefault(x => x == 999); // 0

// Last / LastOrDefault
int last = items.Last(); // 50
int lastSmall = items.LastOrDefault(x => x < 25); // 20

// ElementAt / ElementAtOrDefault
int third = items.ElementAt(2);          // 30
int outOfRange = items.ElementAtOrDefault(100); // 0

// Index (.NET 9) вЂ” СЃ РїРѕРґРґРµСЂР¶РєРѕР№ ^from-end
int fromEnd = items.ElementAt(^1); // 50 (РїРѕСЃР»РµРґРЅРёР№)
```

**РљРѕРіРґР° С‡С‚Рѕ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ:**
| РњРµС‚РѕРґ | РћР¶РёРґР°РЅРёРµ | РџСѓСЃС‚Рѕ | РњРЅРѕРіРѕ |
|-------|----------|-------|-------|
| `First` | >= 1 СЌР»РµРјРµРЅС‚ | Exception | OK (Р±РµСЂС‘С‚ РїРµСЂРІС‹Р№) |
| `Single` | СЂРѕРІРЅРѕ 1 | Exception | Exception |
| `FirstOrDefault` | 0 РёР»Рё Р±РѕР»РµРµ | default | OK |
| `SingleOrDefault` | 0 РёР»Рё 1 | default | Exception |

### РљРѕР»РёС‡РµСЃС‚РІРѕ Рё РїСЂРѕРІРµСЂРєР°

```csharp
var orders = GetOrders();

// Any вЂ” РµСЃС‚СЊ Р»Рё С…РѕС‚СЏ Р±С‹ РѕРґРёРЅ СЌР»РµРјРµРЅС‚ (СЌС„С„РµРєС‚РёРІРЅРµРµ Count() > 0)
bool hasOrders = orders.Any();
bool hasExpensive = orders.Any(o => o.Total > 1000);

// All вЂ” РІСЃРµ Р»Рё СЌР»РµРјРµРЅС‚С‹ СѓРґРѕРІР»РµС‚РІРѕСЂСЏСЋС‚ СѓСЃР»РѕРІРёСЋ
bool allPaid = orders.All(o => o.IsPaid);

// Contains вЂ” СЃРѕРґРµСЂР¶РёС‚ Р»Рё СЌР»РµРјРµРЅС‚
bool has42 = new[] { 1, 2, 42, 100 }.Contains(42);

// РђРќРўРРџРђРўРўР•Р Рќ:
if (orders.Count() > 0) { } // РїР»РѕС…Рѕ вЂ” РїРµСЂРµС‡РёСЃР»СЏРµС‚ РІСЃСЋ РєРѕР»Р»РµРєС†РёСЋ
if (orders.Any()) { }       // С…РѕСЂРѕС€Рѕ вЂ” РѕСЃС‚Р°РЅР°РІР»РёРІР°РµС‚СЃСЏ РЅР° РїРµСЂРІРѕРј СЌР»РµРјРµРЅС‚Рµ
```

### РњРЅРѕР¶РµСЃС‚РІР°

```csharp
var a = new[] { 1, 2, 3, 4, 5 };
var b = new[] { 3, 4, 5, 6, 7 };

// Distinct вЂ” СѓРЅРёРєР°Р»СЊРЅС‹Рµ СЌР»РµРјРµРЅС‚С‹
int[] unique = new[] { 1, 1, 2, 2, 3 }.Distinct().ToArray(); // [1, 2, 3]

// DistinctBy (.NET 6) вЂ” СѓРЅРёРєР°Р»СЊРЅС‹Рµ РїРѕ РєР»СЋС‡Сѓ
var uniqueByCity = people.DistinctBy(p => p.City);

// Union вЂ” РѕР±СЉРµРґРёРЅРµРЅРёРµ (СѓРЅРёРєР°Р»СЊРЅС‹Рµ РёР· РѕР±РµРёС…)
var union = a.Union(b); // [1, 2, 3, 4, 5, 6, 7]

// Intersect вЂ” РїРµСЂРµСЃРµС‡РµРЅРёРµ
var intersect = a.Intersect(b); // [3, 4, 5]

// Except вЂ” СЂР°Р·РЅРѕСЃС‚СЊ (РІ a, РЅРѕ РЅРµ РІ b)
var except = a.Except(b); // [1, 2]

// *By РІР°СЂРёР°РЅС‚С‹ (.NET 6) вЂ” СЃСЂР°РІРЅРµРЅРёРµ РїРѕ РєР»СЋС‡Сѓ
var exceptByCity = peopleA.ExceptBy(
    peopleB.Select(p => p.City),
    p => p.City);

var intersectByAge = peopleA.IntersectBy(
    peopleB.Select(p => p.Age),
    p => p.Age);

var unionByName = peopleA.UnionBy(peopleB, p => p.Name);
```

### Р Р°Р·РґРµР»РµРЅРёРµ

```csharp
var items = Enumerable.Range(1, 20);

// Take / Skip
var firstFive = items.Take(5);       // [1..5]
var afterFive = items.Skip(5);       // [6..20]

// TakeLast / SkipLast
var lastThree = items.TakeLast(3);   // [18, 19, 20]
var withoutLast = items.SkipLast(3); // [1..17]

// Take СЃ Range (C# / .NET 6)
var slice = items.Take(2..5);        // [3, 4, 5]
var fromEnd = items.Take(^3..);      // [18, 19, 20]

// TakeWhile / SkipWhile вЂ” РїРѕ СѓСЃР»РѕРІРёСЋ
var takeWhile = items.TakeWhile(x => x < 5);  // [1, 2, 3, 4]
var skipWhile = items.SkipWhile(x => x < 5);  // [5, 6, ..., 20]

// Chunk (.NET 6) вЂ” СЂР°Р·Р±РёРµРЅРёРµ РЅР° Р±Р°С‚С‡Рё
int[][] chunks = items.Chunk(4).ToArray();
// [[1,2,3,4], [5,6,7,8], [9,10,11,12], [13,14,15,16], [17,18,19,20]]
```

### РќРѕРІС‹Рµ РѕРїРµСЂР°С‚РѕСЂС‹ .NET 6вЂ“9

```csharp
// Index (.NET 9) вЂ” РґРѕР±Р°РІР»СЏРµС‚ РёРЅРґРµРєСЃ Рє РєР°Р¶РґРѕРјСѓ СЌР»РµРјРµРЅС‚Сѓ
foreach (var (index, item) in products.Index())
    Console.WriteLine($"[{index}] {item.Name}");

// CountBy (.NET 9) вЂ” РїРѕРґСЃС‡С‘С‚ РїРѕ РєР»СЋС‡Сѓ (Р·Р°РјРµРЅР° GroupBy + Count)
var wordFreq = words.CountBy(w => w);
// KeyValuePair<string, int>: ("hello", 3), ("world", 2)

// AggregateBy (.NET 9) вЂ” Р°РіСЂРµРіР°С†РёСЏ РїРѕ РєР»СЋС‡Сѓ (Р·Р°РјРµРЅР° GroupBy + Aggregate)
var totalByDept = employees.AggregateBy(
    e => e.Department,
    seed: 0m,
    (total, e) => total + e.Salary);

// MinBy / MaxBy (.NET 6)
var youngest = people.MinBy(p => p.Age);
var oldest = people.MaxBy(p => p.Age);

// DistinctBy (.NET 6)
var onePerCountry = people.DistinctBy(p => p.Country);

// Order / OrderDescending (.NET 7) вЂ” Р±РµР· keySelector
var sorted = numbers.Order();          // РІРјРµСЃС‚Рѕ OrderBy(x => x)
var sortedDesc = numbers.OrderDescending();
```

---

## Expression Trees

### Р§С‚Рѕ С‚Р°РєРѕРµ Expression Tree

```csharp
using System.Linq.Expressions;

// Lambda вЂ” СЃРєРѕРјРїРёР»РёСЂРѕРІР°РЅРЅС‹Р№ РґРµР»РµРіР°С‚ (IL РєРѕРґ)
Func<int, bool> lambda = x => x > 5;

// Expression вЂ” РѕРїРёСЃР°РЅРёРµ lambda РєР°Рє СЃС‚СЂСѓРєС‚СѓСЂС‹ РґР°РЅРЅС‹С… (AST)
Expression<Func<int, bool>> expression = x => x > 5;

// Expression tree вЂ” РґРµСЂРµРІРѕ: GreaterThan(Parameter(x), Constant(5))
// EF Core Р°РЅР°Р»РёР·РёСЂСѓРµС‚ СЌС‚Рѕ РґРµСЂРµРІРѕ Рё РіРµРЅРµСЂРёСЂСѓРµС‚ SQL:
// WHERE x > 5
```

### Р—Р°С‡РµРј РЅСѓР¶РЅС‹ (EF Core, dynamic queries)

```csharp
// EF Core РёСЃРїРѕР»СЊР·СѓРµС‚ Expression<Func<T, bool>> РґР»СЏ РіРµРЅРµСЂР°С†РёРё SQL
IQueryable<Order> query = dbContext.Orders;

// Р­С‚Рѕ РІС‹СЂР°Р¶РµРЅРёРµ С‚СЂР°РЅСЃР»РёСЂСѓРµС‚СЃСЏ РІ SQL
Expression<Func<Order, bool>> filter = o => o.Total > 100 && o.Status == "Active";
var filtered = query.Where(filter); // SELECT * FROM Orders WHERE Total > 100 AND Status = 'Active'

// Р”РёРЅР°РјРёС‡РµСЃРєРѕРµ РїРѕСЃС‚СЂРѕРµРЅРёРµ Expression Trees
static Expression<Func<T, bool>> BuildFilter<T>(string propertyName, object value)
{
    var param = Expression.Parameter(typeof(T), "x");
    var property = Expression.Property(param, propertyName);
    var constant = Expression.Constant(value);
    var equal = Expression.Equal(property, constant);
    return Expression.Lambda<Func<T, bool>>(equal, param);
}

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ
var filter2 = BuildFilter<Order>("Status", "Active");
var activeOrders = dbContext.Orders.Where(filter2).ToList();

// РљРѕРјР±РёРЅРёСЂРѕРІР°РЅРёРµ С„РёР»СЊС‚СЂРѕРІ
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

> [!question]- **РРЅС‚РµСЂРІСЊСЋ: Expression Trees вЂ” Р·Р°С‡РµРј Рё РєР°Рє EF РёС… РёСЃРїРѕР»СЊР·СѓРµС‚?**
> Expression Tree вЂ” РїСЂРµРґСЃС‚Р°РІР»РµРЅРёРµ РєРѕРґР° РєР°Рє СЃС‚СЂСѓРєС‚СѓСЂС‹ РґР°РЅРЅС‹С… (AST). EF Core Р°РЅР°Р»РёР·РёСЂСѓРµС‚ РґРµСЂРµРІРѕ РІС‹СЂР°Р¶РµРЅРёР№ LINQ Рё С‚СЂР°РЅСЃР»РёСЂСѓРµС‚ РІ SQL.
>
> `IQueryable<T>` СЂР°Р±РѕС‚Р°РµС‚ СЃ `Expression<Func<T, bool>>`, Р° РЅРµ `Func<T, bool>`. Expression вЂ” РґР°РЅРЅС‹Рµ (РјРѕР¶РЅРѕ Р°РЅР°Р»РёР·РёСЂРѕРІР°С‚СЊ), Func вЂ” СЃРєРѕРјРїРёР»РёСЂРѕРІР°РЅРЅС‹Р№ РєРѕРґ.

### РљРѕРјРїРёР»СЏС†РёСЏ Expression РІ РґРµР»РµРіР°С‚

```csharp
Expression<Func<int, int, int>> addExpr = (a, b) => a + b;

// РљРѕРјРїРёР»СЏС†РёСЏ РІ РёСЃРїРѕР»РЅСЏРµРјС‹Р№ РґРµР»РµРіР°С‚
Func<int, int, int> addFunc = addExpr.Compile();

int result = addFunc(3, 4); // 7

// РџРѕР»РµР·РЅРѕ: РєРµС€РёСЂРѕРІР°С‚СЊ СЃРєРѕРјРїРёР»РёСЂРѕРІР°РЅРЅС‹Рµ РІС‹СЂР°Р¶РµРЅРёСЏ
private static readonly Func<Order, bool> _isActiveCompiled =
    ((Expression<Func<Order, bool>>)(o => o.Status == "Active")).Compile();
```

---

## Generics

### Generic РєР»Р°СЃСЃС‹, РјРµС‚РѕРґС‹, РёРЅС‚РµСЂС„РµР№СЃС‹

```csharp
// Generic РєР»Р°СЃСЃ
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

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ
var result = Result<int>.Success(42);
var message = result.Match(
    v => $"Value: {v}",
    e => $"Error: {e}");

// Generic РјРµС‚РѕРґ
public static T Max<T>(T a, T b) where T : IComparable<T> =>
    a.CompareTo(b) >= 0 ? a : b;

var maxInt = Max(10, 20);      // 20
var maxStr = Max("abc", "xyz"); // "xyz"

// Generic РёРЅС‚РµСЂС„РµР№СЃ
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id, CancellationToken ct = default);
    Task<IReadOnlyList<T>> GetAllAsync(CancellationToken ct = default);
    Task AddAsync(T entity, CancellationToken ct = default);
    Task UpdateAsync(T entity, CancellationToken ct = default);
    Task DeleteAsync(T entity, CancellationToken ct = default);
}

// Р РµР°Р»РёР·Р°С†РёСЏ
public sealed class OrderRepository(AppDbContext db) : IRepository<Order>
{
    public async Task<Order?> GetByIdAsync(Guid id, CancellationToken ct = default) =>
        await db.Orders.FindAsync([id], ct).ConfigureAwait(false);

    // ... РѕСЃС‚Р°Р»СЊРЅС‹Рµ РјРµС‚РѕРґС‹
}
```

### Generic Constraints

```csharp
// where T : class вЂ” С‚РѕР»СЊРєРѕ reference types
public void Process<T>(T item) where T : class
{
    // item РјРѕР¶РµС‚ Р±С‹С‚СЊ null (РµСЃР»Рё РЅРµ РґРѕР±Р°РІРёС‚СЊ notnull)
}

// where T : struct вЂ” С‚РѕР»СЊРєРѕ value types (non-nullable)
public T ParseOrDefault<T>(string input) where T : struct
{
    // T РіР°СЂР°РЅС‚РёСЂРѕРІР°РЅРЅРѕ value type
    return default; // РЅРёРєРѕРіРґР° РЅРµ null
}

// where T : new() вЂ” РґРѕР»Р¶РµРЅ РёРјРµС‚СЊ parameterless constructor
public T CreateNew<T>() where T : new() => new T();

// where T : notnull вЂ” РЅРµ РјРѕР¶РµС‚ Р±С‹С‚СЊ null (reference РёР»Рё value)
public void Store<T>(T item) where T : notnull
{
    var key = item.GetHashCode(); // Р±РµР·РѕРїР°СЃРЅРѕ вЂ” item РЅРµ null
}

// where T : unmanaged вЂ” blittable value type (РґР»СЏ interop, Span)
public unsafe void WriteToBuffer<T>(T value, byte* buffer) where T : unmanaged
{
    *(T*)buffer = value;
}

// РљРѕРјР±РёРЅРёСЂРѕРІР°РЅРёРµ constraints
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

// where T : BaseClass, IInterface вЂ” РЅР°СЃР»РµРґРѕРІР°РЅРёРµ + РёРЅС‚РµСЂС„РµР№СЃ
public void Handle<T>(T entity)
    where T : BaseEntity, IHasTimestamp, IAuditable
{
    entity.UpdatedAt = DateTime.UtcNow;
    entity.ModifiedBy = "system";
}
```

### Covariance Рё Contravariance

```csharp
// Covariance (out T) вЂ” РјРѕР¶РЅРѕ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ Р±РѕР»РµРµ РїСЂРѕРёР·РІРѕРґРЅС‹Р№ С‚РёРї
// РўРѕР»СЊРєРѕ РґР»СЏ OUTPUT РїРѕР·РёС†РёР№ (return values)
public interface IReadOnlyRepository<out T>
{
    T GetById(int id);
    IEnumerable<T> GetAll(); // IEnumerable<out T> вЂ” РєРѕРІР°СЂРёР°РЅС‚РµРЅ
}

// РџСЂРёРјРµСЂ: IEnumerable<out T> РєРѕРІР°СЂРёР°РЅС‚РµРЅ
IEnumerable<string> strings = ["hello", "world"];
IEnumerable<object> objects = strings; // OK вЂ” string : object, out T

// Contravariance (in T) вЂ” РјРѕР¶РЅРѕ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ РјРµРЅРµРµ РїСЂРѕРёР·РІРѕРґРЅС‹Р№ С‚РёРї
// РўРѕР»СЊРєРѕ РґР»СЏ INPUT РїРѕР·РёС†РёР№ (method parameters)
public interface IComparer<in T>
{
    int Compare(T x, T y);
}

// РџСЂРёРјРµСЂ: Action<in T> РєРѕРЅС‚СЂР°РІР°СЂРёР°РЅС‚РµРЅ
Action<object> printObj = obj => Console.WriteLine(obj);
Action<string> printStr = printObj; // OK вЂ” string : object, in T
printStr("hello"); // РІС‹Р·РѕРІРµС‚ printObj

// РџСЂР°РєС‚РёС‡РµСЃРєРёР№ РїСЂРёРјРµСЂ
public interface IEventHandler<in TEvent> where TEvent : IEvent
{
    Task HandleAsync(TEvent @event, CancellationToken ct);
}

// Handler РґР»СЏ Р±Р°Р·РѕРІРѕРіРѕ С‚РёРїР°
public sealed class LoggingHandler : IEventHandler<IEvent>
{
    public Task HandleAsync(IEvent @event, CancellationToken ct)
    {
        Console.WriteLine($"Event: {@event.GetType().Name}");
        return Task.CompletedTask;
    }
}

// РњРѕР¶РЅРѕ РёСЃРїРѕР»СЊР·РѕРІР°С‚СЊ РєР°Рє handler РґР»СЏ РєРѕРЅРєСЂРµС‚РЅРѕРіРѕ СЃРѕР±С‹С‚РёСЏ
IEventHandler<OrderCreatedEvent> handler = new LoggingHandler(); // contravariance
```

### Generic Math (C# 11 / .NET 7)

```csharp
using System.Numerics;

// INumber<T> вЂ” СѓРЅРёРІРµСЂСЃР°Р»СЊРЅР°СЏ РјР°С‚РµРјР°С‚РёРєР° РґР»СЏ Р»СЋР±РѕРіРѕ С‡РёСЃР»РѕРІРѕРіРѕ С‚РёРїР°
public static T Sum<T>(ReadOnlySpan<T> values) where T : INumber<T>
{
    var result = T.Zero;
    foreach (var value in values)
        result += value;
    return result;
}

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ СЃ СЂР°Р·РЅС‹РјРё С‚РёРїР°РјРё
int intSum = Sum<int>([1, 2, 3, 4, 5]);           // 15
double dblSum = Sum<double>([1.1, 2.2, 3.3]);     // 6.6
decimal decSum = Sum<decimal>([10.5m, 20.3m]);     // 30.8

// РЎСЂРµРґРЅРµРµ РґР»СЏ Р»СЋР±РѕРіРѕ С‡РёСЃР»РѕРІРѕРіРѕ С‚РёРїР°
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

// Clamp вЂ” РѕРіСЂР°РЅРёС‡РµРЅРёРµ РґРёР°РїР°Р·РѕРЅР°
public static T Clamp<T>(T value, T min, T max) where T : INumber<T> =>
    T.Clamp(value, min, max);

// РРЅС‚РµСЂС„РµР№СЃС‹ generic math
public static T Parse<T>(string input) where T : IParsable<T> =>
    T.Parse(input, null);

public static bool TryFormat<T>(T value, Span<char> destination, out int written)
    where T : ISpanFormattable =>
    value.TryFormat(destination, out written, default, null);
```

---

## РџСЂР°РєС‚РёС‡РµСЃРєРёРµ РїР°С‚С‚РµСЂРЅС‹

### Pagination СЃ LINQ

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

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ
var result = await dbContext.Orders
    .Where(o => o.Status == "Active")
    .OrderByDescending(o => o.CreatedAt)
    .ToPagedAsync(page: 2, pageSize: 20, ct);
```

### Specification Pattern СЃ Expression Trees

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

// РСЃРїРѕР»СЊР·РѕРІР°РЅРёРµ
var spec = new ActiveOrderSpec().And(new HighValueOrderSpec(1000));
var orders = await dbContext.Orders
    .Where(spec.ToExpression())
    .ToListAsync(ct);
```

---

## РЎРѕРІРµС‚С‹ РїРѕ РїСЂРѕРёР·РІРѕРґРёС‚РµР»СЊРЅРѕСЃС‚Рё

```csharp
// 1. РСЃРїРѕР»СЊР·СѓР№ Count СЃРІРѕР№СЃС‚РІРѕ РІРјРµСЃС‚Рѕ LINQ Count() РґР»СЏ РєРѕР»Р»РµРєС†РёР№
List<int> list = [1, 2, 3];
int count = list.Count;          // O(1) вЂ” СЃРІРѕР№СЃС‚РІРѕ
int countLinq = list.Count();    // O(1) РґР»СЏ ICollection, РЅРѕ Р»РёС€РЅРёР№ РІС‹Р·РѕРІ

// 2. Any() РІРјРµСЃС‚Рѕ Count() > 0
if (list.Any()) { }              // РѕСЃС‚Р°РЅР°РІР»РёРІР°РµС‚СЃСЏ РЅР° РїРµСЂРІРѕРј
if (list.Count() > 0) { }       // РјРѕР¶РµС‚ РїРµСЂРµС‡РёСЃР»РёС‚СЊ РІСЃС‘ (РґР»СЏ IEnumerable)

// 3. HashSet РґР»СЏ Contains РІ С†РёРєР»Р°С…
var allowedIds = orders.Select(o => o.Id).ToHashSet(); // O(n) РѕРґРёРЅ СЂР°Р·
foreach (var item in items)
{
    if (allowedIds.Contains(item.OrderId)) // O(1) РєР°Р¶РґС‹Р№ СЂР°Р·
        Process(item);
}

// 4. РќРµ РјР°С‚РµСЂРёР°Р»РёР·СѓР№, РµСЃР»Рё РЅРµ РЅР°РґРѕ
// РџР»РѕС…Рѕ:
var filtered = items.Where(x => x > 0).ToList().FirstOrDefault();
// РҐРѕСЂРѕС€Рѕ:
var filtered2 = items.FirstOrDefault(x => x > 0);

// 5. Capacity РґР»СЏ List Рё Dictionary
var results = new List<OrderDto>(capacity: orders.Count); // Р±РµР· СЂРµР°Р»Р»РѕРєР°С†РёР№
var map = new Dictionary<Guid, Order>(capacity: orders.Count);

// 6. Span + stackalloc РІРјРµСЃС‚Рѕ LINQ РґР»СЏ РіРѕСЂСЏС‡РёС… РїСѓС‚РµР№
Span<int> buffer = stackalloc int[10];
// ... Р·Р°РїРѕР»РЅРµРЅРёРµ
int sum = 0;
foreach (var val in buffer)
    sum += val;

// 7. FrozenDictionary РґР»СЏ static lookup С‚Р°Р±Р»РёС†
private static readonly FrozenDictionary<string, Handler> Handlers =
    new Dictionary<string, Handler>
    {
        ["create"] = new CreateHandler(),
        ["update"] = new UpdateHandler(),
        ["delete"] = new DeleteHandler(),
    }.ToFrozenDictionary(StringComparer.OrdinalIgnoreCase);
```

---

## РЎРј. С‚Р°РєР¶Рµ

- [РўРёРїС‹ Рё РїР°РјСЏС‚СЊ](types-and-memory.md)
- [РћРћРџ Рё РєР»Р°СЃСЃС‹](oop.md)
- [Delegates Рё Events](delegates-events.md)
- [Async Рё РїРѕС‚РѕРєРё](async-threading.md)
