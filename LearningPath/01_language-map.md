---
tags: [learning-path, language-map, navigation, csharp]
level: All
date: 2026-06-06
---

# 🗺️ C# Language Map — язык как система, 15 блоков → файлы vault'а

> Карта **самого языка C#** (без ASP.NET / EF / архитектуры). Зеркалит таксономию «Complete C# 2026 Cheat Sheet» (15 блоков, ~90 подпунктов) и привязывает каждый блок к canonical-файлу в этой knowledge base. Цель — за один взгляд видеть: что закрыто глубоко, что тонко, чего нет.

[← 00 Overview](00_overview.md) · [Полный INDEX]() · [Главный README]()

---

## 0. Как читать карту

Карта сгруппирована в **9 слоёв** (от системы типов вниз к BCL), внутри — 15 блоков шпаргалки с исходной нумерацией, чтобы 1:1 ложилось на картинку. Для каждого блока: canonical-файл (wikilink), статус глубины, заметные подпункты.

**Легенда статуса:**

| Знак | Значение |
|------|----------|
| ✅ | Глубокая глава — механизм + примеры + pitfalls. Под Senior-bar. |
| 🟡 | Есть, но тонко / разбросано — факт покрыт, «почему» недосказано. Кандидат на углубление. |
| ❌ | Нет отдельного разбора. |

> [!info]- Зачем ещё одна навигация (помимо INDEX и overview)
> `INDEX.md` сортирует по алфавиту/уровню/теме. `00_overview` даёт roadmap по карьерным ступеням. Эта карта — **третья ось**: язык как инженерная система (типы → выражения → память → конкурентность → метапрограммирование). Полезна когда учишь/повторяешь язык системно, а не по задаче.

---

## Слой 1. Система типов

### Блок 1 — Value Types

| Подпункты | Файл | Статус |
|-----------|------|--------|
| numeric, `bool`, `char`, `enum`, `struct`, `ref struct`, default values | [[csharp-basics\|C# basics]], [[types-and-memory\|types-and-memory]] | ✅ |
| `enum`, `[Flags]`, parsing | [[enums-flags\|enums-flags]] | ✅ |
| tuples, `ValueTuple`, deconstruction | [[tuples-deconstruction\|tuples-deconstruction]] | ✅ |
| nullable value types `Nullable<T>` | [[nullable-types\|nullable-types]] | ✅ |
| `DateTime` / `DateTimeOffset` | [[datetime-timezones\|datetime-timezones]] | ✅ |
| numeric internals, `BigInteger`, `Half`, SIMD | [[numeric-types-math\|numeric-types-math]] | ✅ |

### Блок 2 — Reference Types

| Подпункты | Файл | Статус |
|-----------|------|--------|
| `class`, `object`, `string` | [[csharp-basics\|C# basics]] | ✅ |
| `record`, `record struct` | [[functional-csharp\|functional-csharp]], [[modern-features\|modern-features]] | ✅ |
| `interface` | [[oop\|oop]] | ✅ |
| `delegate`, `event` | [[delegates-events\|delegates-events]] | ✅ |
| Nullable Reference Types (NRT) | [[nullable-types\|nullable-types]] | ✅ |
| boxing / unboxing | [[types-and-memory\|types-and-memory]] | ✅ |

---

## Слой 2. Выражения и операторы

### Блок 3 — Operators & Statements

| Подпункты | Файл | Статус |
|-----------|------|--------|
| arithmetic / boolean / bitwise, ternary, comparison | [[csharp-basics\|C# basics]] | ✅ |
| `if` / `switch` / switch expressions | [[functional-csharp\|functional-csharp]], [[csharp-basics\|C# basics]] | ✅ |
| `checked` / `unchecked`, `OverflowException` | [[numeric-types-math\|numeric-types-math]], [[keywords-reference\|keywords-reference]] | ✅ |
| `for` / `while` / `do`, `foreach` / `await foreach` | [[csharp-basics\|C# basics]], [[iterators-yield\|iterators-yield]] | ✅ |

### Блок 4 — Expressions

| Подпункты | Файл | Статус |
|-----------|------|--------|
| pattern matching (type / property / list patterns) | [[functional-csharp\|functional-csharp]] | ✅ |
| collection expressions `[1, 2, 3]`, spread `..` | [[modern-features\|modern-features]], [[collections-linq\|collections-linq]] | ✅ |
| lambda expressions, member access, type testing | [[delegates-events\|delegates-events]], [[csharp-basics\|C# basics]] | ✅ |

---

## Слой 3. Структура программы и OOP

### Блок 5 — Program Structure

| Подпункты | Файл | Статус |
|-----------|------|--------|
| `Main`, top-level statements, namespaces, global usings | [[csharp-basics\|C# basics]], [[modern-features\|modern-features]] | ✅ |

### Блок 9 — OOP

| Подпункты | Файл | Статус |
|-----------|------|--------|
| classes / ctors, fields / methods / properties, inheritance, polymorphism, virtual / abstract, access modifiers, sealed / static | [[oop\|oop]] | ✅ |
| primary constructors, `field` keyword | [[modern-features\|modern-features]] | ✅ |
| constants & indexers, operator overloading | [[indexers-operators\|indexers-operators]] | ✅ |
| anonymous types | [[anonymous-types\|anonymous-types]] | ✅ |
| partial classes & **partial members** (properties C# 13) | [[modern-features\|modern-features]] §11.2 | ✅ |
| equality (`Equals` / `GetHashCode` / `IEquatable`) | [[equality-comparison\|equality-comparison]] | ✅ |

---

## Слой 4. Generics, коллекции, LINQ

### Блок 6 — Generics & Collections

| Подпункты | Файл | Статус |
|-----------|------|--------|
| generics, constraints, **variance** (`in` / `out`) | [[generics-deep\|generics-deep]] | ✅ |
| arrays & collections, readonly / immutable collections | [[collections-linq\|collections-linq]] | ✅ |
| `Span<T>` & `Memory<T>` | [[span-layout\|span-layout]] | ✅ |
| **Frozen Collections** (`FrozenDictionary` / `FrozenSet`) | [[collections-linq\|collections-linq]] §5.7 / §9.8 | 🟡 есть, но без механизма (perfect hashing, build-cost trade-off) |

### Блок 10 — LINQ

| Подпункты | Файл | Статус |
|-----------|------|--------|
| `IEnumerable` vs `IQueryable`, filtering / projection / sorting / grouping / aggregation / joins / set ops / partition | [[collections-linq\|collections-linq]] | ✅ |
| LINQ provider internals (как `Expression<Func<T>>` → SQL) | [[reflection-expression-trees\|reflection-expression-trees]] | ✅ |

---

## Слой 5. Память и GC

### Блок 7 — Memory Management & GC

| Подпункты | Файл | Статус |
|-----------|------|--------|
| finalizers, dispose pattern, `using` | [[dispose-pattern\|dispose-pattern]] | ✅ |
| GC generations, LOH / SOH / POH, regions, DATAS | [[gc-memory\|gc-memory]] | ✅ |
| `stackalloc`, struct layout, alignment / padding | [[span-layout\|span-layout]] | ✅ |
| **Inline Arrays** (`[InlineArray]`, C# 12) | [[span-layout\|span-layout]] §8, [[unsafe-pointers\|unsafe-pointers]] §4.6 | ✅ |
| `ArrayPool` / `ObjectPool` / `MemoryPool` (GC pressure) | [[memory-pooling\|memory-pooling]] | ✅ |

---

## Слой 6. Асинхронность и конкурентность

### Блок 11 — Async Programming

| Подпункты | Файл | Статус |
|-----------|------|--------|
| `Thread` / `ThreadPool`, `SynchronizationContext`, `Task` / `ValueTask`, `async`/`await`, `ConfigureAwait`, continuations, `WhenAll` / `WhenAny`, cancellation, `TaskCompletionSource`, PLINQ / `Parallel` | [[async-threading\|async-threading]] | ✅ |
| async streams `IAsyncEnumerable`, `Channel<T>` | [[iterators-yield\|iterators-yield]], [[async-threading\|async-threading]] | ✅ |
| Thread / TPL / Parallel basics | [[threading-basics\|threading-basics]] | ✅ |

### Блок 12 — Thread Synchronization

| Подпункты | Файл | Статус |
|-----------|------|--------|
| `volatile`, `Interlocked`, memory model, lock-free | [[concurrency-atomics\|concurrency-atomics]] | ✅ |
| `lock` / `Monitor`, `Mutex`, `Semaphore` / `SemaphoreSlim`, `ManualResetEvent` / `AutoResetEvent` | [[concurrency-atomics\|concurrency-atomics]], [[threading-basics\|threading-basics]] | ✅ |

---

## Слой 7. Метапрограммирование

### Блок 8 — Reflection & Dynamic Code

| Подпункты | Файл | Статус |
|-----------|------|--------|
| reflection, member discovery, `Activator`, dynamic load | [[reflection-expression-trees\|reflection-expression-trees]] | ✅ |
| `dynamic`, DLR, `ExpandoObject` (+ perf, AOT-ограничения) | [[reflection-expression-trees\|reflection-expression-trees]] §8 | ✅ |
| expression trees как data | [[reflection-expression-trees\|reflection-expression-trees]] | ✅ |
| compile-time альтернатива (source generators) | [[source-generators\|source-generators]] | ✅ |
| attributes & metadata | [[attributes-metadata\|attributes-metadata]] | ✅ |

---

## Слой 8. Ошибки, делегаты, события

### Блок 13 — Error Handling

| Подпункты | Файл | Статус |
|-----------|------|--------|
| `try` / `catch` / `finally`, **catch filters** (`when`), throwing, `Result<T>` vs exceptions | [[error-handling\|error-handling]] | ✅ |

### Блок 14 — Delegates & Events

| Подпункты | Файл | Статус |
|-----------|------|--------|
| delegates, `Func` / `Action`, raise & consume events, weak events | [[delegates-events\|delegates-events]] | ✅ |

---

## Слой 9. BCL essentials (Other Topics)

### Блок 15 — Other Topics

| Подпункты | Файл | Статус |
|-----------|------|--------|
| iterators `yield` | [[iterators-yield\|iterators-yield]] | ✅ |
| attributes | [[attributes-metadata\|attributes-metadata]] | ✅ |
| regex | [[strings-regex\|strings-regex]] | ✅ |
| file streams, async I/O | [[io-streams\|io-streams]] | ✅ |
| JSON serialization (`System.Text.Json`, source-gen) | [[serialization-deep\|serialization-deep]] | ✅ |
| **`Guid` & `Random`**, **`TimeSpan` & `Stopwatch`**, `TimeProvider` | [[bcl-essentials\|bcl-essentials]] | ✅ |
| **XML / Binary serialization** + `System.Text.Json` deep | [[serialization-deep\|serialization-deep]] | ✅ |

---

## Белые пятна — что углубить под «max background»

После сверки реальных секций (а не упоминаний) критичных языковых дыр против шпаргалки **нет**. Оба кандидата на углубление закрыты 2026-06-12:

1. ✅ **Frozen Collections — механизм.** [[collections-linq]] §5.7 теперь объясняет: анализ ключей при построении (perfect-hash подход), специализированные реализации (linear scan для маленьких наборов, дискриминатор по подстроке для строк, прямой индекс для плотных int), trade-off build-cost vs read-speed, `GetAlternateLookup` (.NET 9, lookup по `ReadOnlySpan<char>` без аллокаций) и анти-кейсы (частый rebuild → `ImmutableDictionary`).
2. ✅ **BCL essentials — канонический файл.** Создан [[bcl-essentials]]: `Guid` (v4 vs v7, ключи БД, порядок байтов SQL Server), `Random` (thread safety, `Random.Shared`, граница с `RandomNumberGenerator`), `TimeSpan`, `Stopwatch` (`GetTimestamp`/`GetElapsedTime` без аллокаций), `TimeProvider` (.NET 8, тестируемое время). XML/Binary/JSON serialization — отдельный canonical [[serialization-deep]].

> [!info]- Что НЕ покрывает эта шпаргалка (и почему карта только про язык)
> «Complete C# 2026» — чисто синтаксис языка. Senior-в-бигтехе и тимлид гейтятся тем, чего в ней нет: архитектура, distributed systems, observability, testing, leadership. Это закрыто другими папками vault'а ([[architecture-patterns]], [[ddd]], [[distributed-systems]], [[observability]], `Testing/`). Эта карта намеренно про язык — остальные оси см. в [[00_overview]] и INDEX.

---

## См. также

- [[00_overview\|00 Overview]] — навигация по карьерным ступеням
- [[05_topics-by-priority\|05 Topics by Priority]] — что учить в каком порядке по value/effort
- [[09_senior-tips-cheatsheet\|09 Senior Tips]] — быстрый review языковых фич
- [[modern-features\|modern-features]] — таблица фич C# 8 → 14 в одном месте
