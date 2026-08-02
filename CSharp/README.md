# CSharp — язык

> 42 файла / ~2.4 MB. Всё про C# как язык: от basics до advanced internals. Самая большая папка vault'а.

[[README|← Главный README]] · [[INDEX|Полный INDEX]]

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Никогда не писал на C# | [[csharp-basics|`Junior/csharp-basics.md`]] → [[dotnet-cli-getting-started|`Junior/dotnet-cli-getting-started.md`]] |
| Знаю Java/Python, перехожу | [[csharp-vs-other-langs|`Senior/csharp-vs-other-langs.md`]] |
| Junior — daily tools | [[debugging-basics|`Junior/debugging-basics.md`]], [[naming-conventions|`Junior/naming-conventions.md`]] |
| Junior хочу в Middle | [[oop|`Junior/oop.md`]] → [[error-handling|`Middle/error-handling.md`]] → [[collections-linq|`Junior/collections-linq.md`]] |
| Middle хочу в Senior | [[async-threading|`Senior/async-threading.md`]] → [[types-and-memory|`Senior/types-and-memory.md`]] → [[generics-deep|`Middle/generics-deep.md`]] |
| Готовлюсь к собесу | async-threading, types-and-memory, collections-linq, design-patterns, gof-patterns-extended |
| Senior performance | [[memory-pooling|`Senior/memory-pooling.md`]], [[unsafe-pointers|`Senior/unsafe-pointers.md`]], [[numeric-types-math|`Middle/numeric-types-math.md`]] |

---

## 📚 Все файлы по уровню

> Гиганты Junior-папки (100+ KB) читаются по диапазонам из roadmap'ов, не подряд — см. [[02_junior-to-middle|`LearningPath/02_junior-to-middle.md`]].

### 🌱 Junior (14)

| Файл | Описание |
|------|----------|
| [[csharp-basics|`csharp-basics.md`]] | Стартовая точка: переменные, типы, control flow |
| [[dotnet-cli-getting-started|`dotnet-cli-getting-started.md`]] | dotnet CLI, templates, package management |
| [[debugging-basics|`debugging-basics.md`]] | Отладка: breakpoints, watch, immediate, logging |
| [[naming-conventions|`naming-conventions.md`]] | Naming: PascalCase, camelCase, conventions |
| [[oop|`oop.md`]] | Inheritance, polymorphism, abstract — Junior→Senior разделы |
| [[collections-linq|`collections-linq.md`]] | Collections, LINQ deep — Junior→Senior разделы |
| [[datetime-timezones|`datetime-timezones.md`]] | DateTime, TimeZoneInfo, NodaTime |
| [[strings-regex|`strings-regex.md`]] | String operations, Regex, performance |
| [[enums-flags|`enums-flags.md`]] | Enum, [Flags], parsing, serialization |
| [[tuples-deconstruction|`tuples-deconstruction.md`]] | ValueTuple, deconstruction |
| [[anonymous-types|`anonymous-types.md`]] | Anonymous types, when и зачем |
| [[extension-methods|`extension-methods.md`]] | this-параметр, fluent APIs, LINQ-like |
| [[iterators-yield|`iterators-yield.md`]] | yield return, IAsyncEnumerable |
| [[async-await-basics|`async-await-basics.md`]] ⭐ NEW | async/await вход: Task, await, CancellationToken — мост к async-threading |

### 🌿 Middle (14)

| Файл | Описание |
|------|----------|
| [[modern-features|`modern-features.md`]] | Records, primary ctors, raw strings, C# 8→14 |
| [[error-handling|`error-handling.md`]] | Exceptions, Result, OneOf |
| [[nullable-types|`nullable-types.md`]] | Nullable\<T\>, NRT, null operators |
| [[generics-deep|`generics-deep.md`]] | Variance, INumber\<T\>, generic math |
| [[delegates-events|`delegates-events.md`]] | Delegates, events, Func/Action |
| [[equality-comparison|`equality-comparison.md`]] | Equals, GetHashCode, IEquatable |
| [[attributes-metadata|`attributes-metadata.md`]] | Attributes, custom attributes, reflection |
| [[indexers-operators|`indexers-operators.md`]] | Custom indexers, operator overloading |
| [[dispose-pattern|`dispose-pattern.md`]] | IDisposable, IAsyncDisposable, SafeHandle |
| [[io-streams|`io-streams.md`]] | File, Stream, async I/O |
| [[serialization-deep|`serialization-deep.md`]] ⭐ NEW | System.Text.Json deep, source-gen, XML, MessagePack/protobuf |
| [[bcl-essentials|`bcl-essentials.md`]] ⭐ NEW | Guid (v4/v7, ключи БД), Random.Shared, TimeSpan, Stopwatch, TimeProvider |
| [[numeric-types-math|`numeric-types-math.md`]] | BigInteger, Half, Vector\<T\>, SIMD |
| [[keywords-reference|`keywords-reference.md`]] | C# keywords reference (ref/in/out/scoped/etc) |

### 🏆 Senior (14)

| Файл | Описание |
|------|----------|
| [[async-threading|`async-threading.md`]] | Task, async/await internals ⭐ |
| [[types-and-memory|`types-and-memory.md`]] | Value vs reference, boxing, struct ⭐ |
| [[functional-csharp|`functional-csharp.md`]] | Records, pattern matching, FP в C# |
| [[design-patterns|`design-patterns.md`]] | SOLID, обзор GoF, DI, Repository/CQRS/DDD, modern-замены паттернов |
| [[gof-patterns-extended|`gof-patterns-extended.md`]] | Все 23 GoF pattern'а подробно + anti-patterns |
| [[reflection-expression-trees|`reflection-expression-trees.md`]] | Reflection, expression trees, dynamic |
| [[source-generators|`source-generators.md`]] | Source generators (.NET 5+) |
| [[memory-pooling|`memory-pooling.md`]] | ArrayPool, ObjectPool, MemoryPool |
| [[unsafe-pointers|`unsafe-pointers.md`]] | unsafe, fixed, stackalloc, ref struct |
| [[csharp-language-design|`csharp-language-design.md`]] | History, design decisions, evolution |
| [[csharp-vs-other-langs|`csharp-vs-other-langs.md`]] | C# vs Java/Go/Rust/Python/TS |
| [[cli-tools-scripting|`cli-tools-scripting.md`]] | System.CommandLine, scripting |
| [[desktop-frameworks|`desktop-frameworks.md`]] | WPF, MAUI, Avalonia |
| [[fenwick-bit|`fenwick-bit.md`]] ⭐ NEW | Fenwick Tree (BIT): префиксные суммы O(log n), трюк i & -i |

---

## 🎓 Рекомендуемый порядок изучения

### Junior path (3-6 мес)

```
1. csharp-basics
2. dotnet-cli-getting-started  ← запустить первый проект
3. debugging-basics             ← отлаживать с первого дня
4. naming-conventions           ← привычки правильно
5. oop (Junior разделы)
6. collections-linq (Junior разделы)
7. error-handling (intro)
8. iterators-yield, enums-flags, strings-regex
9. nullable-types
10. dispose-pattern (IDisposable, using)
11. async-await-basics          ← мост к async-threading
```

### Middle path (6-12 мес)

```
1. async-threading (must!)
2. types-and-memory
3. delegates-events
4. modern-features (records, pattern matching)
5. attributes-metadata, equality-comparison
6. extension-methods, indexers-operators
7. io-streams + serialization-deep + bcl-essentials
8. generics-deep
9. design-patterns (SOLID + обзор GoF)
10. functional-csharp
```

### Senior path

```
1. reflection-expression-trees
2. source-generators
3. memory-pooling, unsafe-pointers
4. gof-patterns-extended (все 23 GoF)
5. numeric-types-math (SIMD, generic math)
6. csharp-language-design (понять design decisions)
7. cli-tools-scripting, desktop-frameworks (specific domains)
```

---

## 🔗 Связанные папки

- [`Runtime/`](../Runtime/) — CLR internals (GC, JIT, threading) — расширение глубже C#
- [`Performance/`](../Performance/) — performance optimization
- [`Architecture/`](../Architecture/) — паттерны в архитектурном контексте
- [[case-studies-top7|`LearningPath/case-studies-top7.md`]] — production case studies
