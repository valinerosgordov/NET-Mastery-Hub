# Collections и LINQ

## HashSet, Dictionary, ToLookup

HashSet — множество без дубликатов, O(1) lookup. Dictionary — ключ → значение. ToLookup — ключ → последовательность, материализация при создании.

---

## IEnumerable, IQueryable

IEnumerable — в памяти, отложенное выполнение. IQueryable — у провайдера (БД), дерево выражений до перечисления. IQueryable наследует IEnumerable.

---

## Deferred Execution, yield, Cast

Запрос выполняется при перечислении (foreach, ToList). yield return — ленивая генерация, state machine. Cast — приведение при перечислении, не создаёт объекты. OfType — фильтр по типу.

---

## Expression Trees

Expression<T> — код как данные. EF обходит и строит SQL. Отличие от делегата — можно анализировать и преобразовывать.

---

## Immutable, Frozen, Thread-safe

ImmutableList — «мутация» возвращает новый экземпляр. FrozenDictionary/FrozenSet (.NET 8) — неизменяемые, оптимизированы для чтения. Concurrent*, BlockingCollection, Channel.
