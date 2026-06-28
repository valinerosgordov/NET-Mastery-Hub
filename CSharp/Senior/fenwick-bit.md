---
tags: [csharp, fenwick, bit, data-structures, senior, algorithms, prefix-sum, performance]
level: Senior
date: 2026-06-19
---

# Fenwick Tree (BIT) — running aggregates за O(log n)

> **Binary Indexed Tree: prefix-sum + point-update за O(log n) на плоском `int[]`, без node objects и без аллокаций.** Закрывает задачу «нужны бегущие агрегаты по изменяемому массиву» — leaderboards, sliding metrics, rank/order-statistics. Проще segment tree, когда нужны именно prefix-aggregate + point-update. Закрывает пробел: «знаю про префиксные суммы, но при каждом update пересчитываю весь массив за O(n)».

---

## 0. Как читать

Если впервые — раздел 1 (зачем) → 2 (lowest-set-bit trick, это всё ядро структуры) → 4 (реализация). Если уже знаешь BIT — раздел 5 (use cases), 6 (vs segment tree).

---

## 1. Зачем, и почему не наивно

Дано: массив, по которому надо отвечать на запрос «сумма первых k элементов» (prefix sum) И изменять отдельные элементы. Две наивные крайности, обе плохие:

```csharp
// Вариант A: хранить сам массив.
// query prefixSum(k) — O(n) (суммируем заново)
// update(i, delta) — O(1)

// Вариант B: хранить готовые префиксные суммы.
// query prefixSum(k) — O(1)
// update(i, delta) — O(n) (сдвигаем все суммы после i)
```

Когда updates и queries перемешаны (live leaderboard: счёт игрока меняется, тут же спрашивают его ранг), любая из крайностей деградирует до O(n) на операцию. Fenwick tree даёт **обе** операции за O(log n) — баланс вместо перекоса.

Range-sum `[l, r]` выводится из двух prefix-запросов: `sum(l, r) = prefix(r) - prefix(l - 1)`.

---

## 2. Ядро: lowest-set-bit и span покрытия

Вся структура держится на одном трюке. `i & -i` выделяет младший установленный бит числа (lowest set bit):

```csharp
int i = 12;            // 0b1100
int low = i & -i;      // 0b0100 == 4
```

Почему так: `-i` в two's complement — это `~i + 1`, то есть инверсия всех битов плюс единица. После инверсии+1 все биты ниже младшей единицы остаются как были, сама младшая единица сохраняется, а всё выше неё инвертируется. `AND` оставляет ровно младший бит.

Идея дерева: ячейка `tree[i]` хранит частичную сумму **диапазона длиной `i & -i`, заканчивающегося в `i`**. То есть `tree[i]` покрывает индексы `(i - (i & -i), i]`. Чем больше младший бит у `i`, тем длиннее span — за счёт этого любой префикс собирается из `O(log n)` непересекающихся блоков.

```text
1-based индексы, i & -i = длина span ячейки:
tree[1] -> span 1  -> покрывает (0,1]   = [1]
tree[2] -> span 2  -> покрывает (0,2]   = [1,2]
tree[3] -> span 1  -> покрывает (2,3]   = [3]
tree[4] -> span 4  -> покрывает (0,4]   = [1..4]
tree[6] -> span 2  -> покрывает (4,6]   = [5,6]
tree[8] -> span 8  -> покрывает (0,8]   = [1..8]
```

> [!warning]
> Fenwick tree **1-based**. Индекс 0 не используется как ячейка (`0 & -0 == 0` дало бы бесконечный цикл при walk). Внешний 0-based API мапится прибавлением 1 на входе.

### 2.1. Update walk: `index += index & -index`

Обновляя элемент, надо поправить все ячейки, чьи span **накрывают** этот индекс. Двигаемся вверх, каждый раз перепрыгивая через свой младший бит:

```csharp
void Add(int index, int delta)
{
    for (; index <= _n; index += index & -index)
        _tree[index] += delta;
}
```

`index += index & -index` гасит младший бит и поднимает на бит выше — за `O(log n)` шагов доходим до конца массива.

### 2.2. Query walk: `index -= index & -index`

Префиксная сумма `[1..index]` собирается из непересекающихся блоков; каждый шаг вычитает младший бит, отрезая покрытый блок и переходя к остатку:

```csharp
long Prefix(int index)
{
    long sum = 0;
    for (; index > 0; index -= index & -index)
        sum += _tree[index];
    return sum;
}
```

`index -= index & -index` уводит к 0 за `O(log n)` шагов. Update идёт «вверх» (+), query — «вниз» (−). Это зеркальность — главное, что стоит запомнить.

---

## 3. Сложность и память

| Операция | Время | Память |
|----------|-------|--------|
| `Add(i, delta)` | O(log n) | — |
| `Prefix(i)` | O(log n) | — |
| `RangeSum(l, r)` | O(log n) | — |
| Построение из массива | O(n) | — |
| Хранение | — | `int[n+1]` / `long[n+1]`, одна аллокация |

Backing store — один плоский массив (`int[]` или `long[]`), **никаких node-объектов**, никаких ссылок, отличная cache locality. После конструктора — zero allocation на операциях.

> [!info]
> Для агрегата суммы выбирай тип аккумулятора с запасом. Счётчики на `int` легко переполняются: миллион значений по 10k уже > `int.MaxValue`. Для prefix-сумм бери `long[]`.

---

## 4. Реализация

```csharp
namespace Algorithms.Structures;

/// <summary>
/// Fenwick tree (Binary Indexed Tree): point-update + prefix-sum за O(log n).
/// 1-based внутри, 0-based наружу. Backing store — плоский long[], allocation-free на операциях.
/// </summary>
public sealed class FenwickTree
{
    private readonly long[] _tree;
    private readonly int _n;

    public FenwickTree(int size)
    {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(size);
        _n = size;
        _tree = new long[size + 1]; // index 0 не используется
    }

    /// <summary>O(n)-построение из готовых значений (быстрее n × Add).</summary>
    public FenwickTree(ReadOnlySpan<long> values) : this(values.Length)
    {
        for (int i = 0; i < values.Length; i++)
        {
            int idx = i + 1;
            _tree[idx] += values[i];
            int parent = idx + (idx & -idx);
            if (parent <= _n)
                _tree[parent] += _tree[idx];
        }
    }

    /// <summary>Прибавить delta к элементу с 0-based индексом index.</summary>
    public void Add(int index, long delta)
    {
        for (int i = index + 1; i <= _n; i += i & -i)
            _tree[i] += delta;
    }

    /// <summary>Сумма элементов [0 .. index] включительно (0-based).</summary>
    public long PrefixSum(int index)
    {
        long sum = 0;
        for (int i = index + 1; i > 0; i -= i & -i)
            sum += _tree[i];
        return sum;
    }

    /// <summary>Сумма на отрезке [left .. right] включительно (0-based).</summary>
    public long RangeSum(int left, int right) =>
        PrefixSum(right) - (left > 0 ? PrefixSum(left - 1) : 0);
}
```

Использование:

```csharp
var bit = new FenwickTree(8);
bit.Add(2, 5);          // элемент[2] += 5
bit.Add(5, 3);          // элемент[5] += 3
long s = bit.RangeSum(0, 5); // 8
```

---

## 5. Use cases

- **Leaderboards / live ranking.** Очко игрока меняется → `Add`; «сколько игроков с очками ниже X» → prefix-count. Ранг за `O(log n)` вместо пересортировки.
- **Sliding metrics.** Скользящие суммы/средние по потоку событий с произвольной точечной корректировкой бакетов.
- **Rank / order-statistics.** На частотном массиве: «k-й по порядку элемент» — бинарный поиск по дереву (`O(log n)` или `O(log^2 n)`), count distinct в окне, количество инверсий при сортировке.
- **2D-расширение.** Вложенный BIT → prefix-сумма по прямоугольнику за `O(log^2 n)` (тепловые карты, агрегаты по сетке).

> [!question]- Интервью: как BIT решает order-statistics («найти k-й элемент»)?
> Строим частотный BIT по диапазону значений (`tree[v]` = сколько раз значение `v` встречалось). Точечная вставка/удаление значения — `Add`. «k-й по возрастанию» = найти наименьший индекс с `prefix == k`. Делается бинарным спуском прямо по дереву (идти по битам от старшего, накапливая сумму, пока не превысим k) за `O(log n)`, либо обычным бинпоиском поверх `PrefixSum` за `O(log^2 n)`.

---

## 6. vs Segment Tree

Fenwick — это «урезанный» сегментный для одного частного, но очень частого случая.

| | Fenwick (BIT) | Segment Tree |
|---|---------------|--------------|
| Память | `int[n+1]`, 1 массив | ~`4n` узлов / массив |
| Константа | очень малая (плоский массив) | выше (рекурсия / индексация детей) |
| Код | ~3 коротких цикла | заметно длиннее |
| Point-update + prefix/range-query | ✅ идеально | ✅ |
| Range-update + range-query | ⚠️ только с трюком (два BIT) | ✅ через lazy propagation |
| Произвольные операции (min/max/gcd) | ❌ обратимость нужна (sum, xor) | ✅ любой ассоциативный моноид |

Эвристика выбора:

- **Sum / xor + point-update + prefix-query** → Fenwick. Меньше памяти, меньше кода, быстрее по константе.
- **Range-update + range-query**, или **min/max/gcd** (нет обратной операции для «вычесть префикс») → **segment tree** (с lazy propagation для range-update).

> [!warning]
> Базовый Fenwick опирается на **обратимость**: `range = prefix(r) - prefix(l-1)`. Для `min`/`max` вычитания нет — наивно перенести нельзя. Это типичная ловушка «возьму BIT для range-min» — бери segment tree.

---

## 7. Common mistakes

- **0-based внутри дерева.** `i & -i` при `i == 0` равно 0 → `Prefix` мгновенно выходит, `Add` зацикливается/не двигается. Всегда +1 на границе API, ячейка 0 — мёртвая.
- **`int` под аккумулятор сумм.** Тихое переполнение при больших объёмах. Для prefix-сумм — `long`.
- **Перепутанное направление walk.** Update += младший бит (вверх), query −= младший бит (вниз). Зеркально перепутать — получите правдоподобный, но неверный результат на части входов; ловится только тестом со смешанными update/query.
- **`n × Add` для построения.** `O(n log n)`. Конструктор из span строит за `O(n)` — используй его для bulk-инициализации.
- **BIT для range-min/max.** См. callout выше — нужна обратимость.

---

## 8. См. также

- [[types-and-memory|Types and Memory]] — почему плоский `int[]` без node-объектов даёт cache locality и zero allocation
- [[collections-linq|Collections и LINQ]] — стандартные коллекции, когда O(log n)-структура не нужна
- [[memory-pooling|Memory Pooling]] — переиспользование буферов для allocation-free hot path

---

## 9. Reading list

- **Peter Fenwick (1994)** — «A New Data Structure for Cumulative Frequency Tables» (оригинальная статья)
- **CP-Algorithms — Fenwick Tree** — cp-algorithms.com/data_structures/fenwick.html
- **Halim, "Competitive Programming"** — раздел Fenwick / Segment Tree
