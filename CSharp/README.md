# CSharp — язык

> 41 файл / ~2.4 MB. Всё про C# как язык: от basics до advanced internals. Самая большая папка vault'а.

[← Главный README](../readme.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Никогда не писал на C# | [`Junior/csharp-basics.md`](Junior/csharp-basics.md) → [`Junior/dotnet-cli-getting-started.md`](Junior/dotnet-cli-getting-started.md) |
| Знаю Java/Python, перехожу | [`Senior/csharp-vs-other-langs.md`](Senior/csharp-vs-other-langs.md) |
| Junior — daily tools | [`Junior/debugging-basics.md`](Junior/debugging-basics.md), [`Junior/naming-conventions.md`](Junior/naming-conventions.md) |
| Junior хочу в Middle | [`Junior/oop.md`](Junior/oop.md) → [`Middle/error-handling.md`](Middle/error-handling.md) → [`Junior/collections-linq.md`](Junior/collections-linq.md) |
| Middle хочу в Senior | [`Senior/async-threading.md`](Senior/async-threading.md) → [`Senior/types-and-memory.md`](Senior/types-and-memory.md) → [`Middle/generics-deep.md`](Middle/generics-deep.md) |
| Готовлюсь к собесу | async-threading, types-and-memory, collections-linq, design-patterns, gof-patterns-extended |
| Senior performance | [`Senior/memory-pooling.md`](Senior/memory-pooling.md), [`Senior/unsafe-pointers.md`](Senior/unsafe-pointers.md), [`Middle/numeric-types-math.md`](Middle/numeric-types-math.md) |

---

## 📚 Все файлы по уровню

> Гиганты Junior-папки (100+ KB) читаются по диапазонам из roadmap'ов, не подряд — см. [`LearningPath/02_junior-to-middle.md`](../LearningPath/02_junior-to-middle.md).

### 🌱 Junior (13)

| Файл | Описание |
|------|----------|
| [`csharp-basics.md`](Junior/csharp-basics.md) | Стартовая точка: переменные, типы, control flow |
| [`dotnet-cli-getting-started.md`](Junior/dotnet-cli-getting-started.md) | dotnet CLI, templates, package management |
| [`debugging-basics.md`](Junior/debugging-basics.md) | Отладка: breakpoints, watch, immediate, logging |
| [`naming-conventions.md`](Junior/naming-conventions.md) | Naming: PascalCase, camelCase, conventions |
| [`oop.md`](Junior/oop.md) | Inheritance, polymorphism, abstract — Junior→Senior разделы |
| [`collections-linq.md`](Junior/collections-linq.md) | Collections, LINQ deep — Junior→Senior разделы |
| [`datetime-timezones.md`](Junior/datetime-timezones.md) | DateTime, TimeZoneInfo, NodaTime |
| [`strings-regex.md`](Junior/strings-regex.md) | String operations, Regex, performance |
| [`enums-flags.md`](Junior/enums-flags.md) | Enum, [Flags], parsing, serialization |
| [`tuples-deconstruction.md`](Junior/tuples-deconstruction.md) | ValueTuple, deconstruction |
| [`anonymous-types.md`](Junior/anonymous-types.md) | Anonymous types, when и зачем |
| [`extension-methods.md`](Junior/extension-methods.md) | this-параметр, fluent APIs, LINQ-like |
| [`iterators-yield.md`](Junior/iterators-yield.md) | yield return, IAsyncEnumerable |

### 🌿 Middle (14)

| Файл | Описание |
|------|----------|
| [`modern-features.md`](Middle/modern-features.md) | Records, primary ctors, raw strings, C# 8→14 |
| [`error-handling.md`](Middle/error-handling.md) | Exceptions, Result, OneOf |
| [`nullable-types.md`](Middle/nullable-types.md) | Nullable\<T\>, NRT, null operators |
| [`generics-deep.md`](Middle/generics-deep.md) | Variance, INumber\<T\>, generic math |
| [`delegates-events.md`](Middle/delegates-events.md) | Delegates, events, Func/Action |
| [`equality-comparison.md`](Middle/equality-comparison.md) | Equals, GetHashCode, IEquatable |
| [`attributes-metadata.md`](Middle/attributes-metadata.md) | Attributes, custom attributes, reflection |
| [`indexers-operators.md`](Middle/indexers-operators.md) | Custom indexers, operator overloading |
| [`dispose-pattern.md`](Middle/dispose-pattern.md) | IDisposable, IAsyncDisposable, SafeHandle |
| [`io-streams.md`](Middle/io-streams.md) | File, Stream, async I/O |
| [`serialization-deep.md`](Middle/serialization-deep.md) ⭐ NEW | System.Text.Json deep, source-gen, XML, MessagePack/protobuf |
| [`bcl-essentials.md`](Middle/bcl-essentials.md) ⭐ NEW | Guid (v4/v7, ключи БД), Random.Shared, TimeSpan, Stopwatch, TimeProvider |
| [`numeric-types-math.md`](Middle/numeric-types-math.md) | BigInteger, Half, Vector\<T\>, SIMD |
| [`keywords-reference.md`](Middle/keywords-reference.md) | C# keywords reference (ref/in/out/scoped/etc) |

### 🏆 Senior (14)

| Файл | Описание |
|------|----------|
| [`async-threading.md`](Senior/async-threading.md) | Task, async/await internals ⭐ |
| [`types-and-memory.md`](Senior/types-and-memory.md) | Value vs reference, boxing, struct ⭐ |
| [`functional-csharp.md`](Senior/functional-csharp.md) | Records, pattern matching, FP в C# |
| [`design-patterns.md`](Senior/design-patterns.md) | 13 GoF patterns в C# |
| [`gof-patterns-extended.md`](Senior/gof-patterns-extended.md) | Ещё 8 GoF: Command, Visitor, Composite, Proxy и др. |
| [`reflection-expression-trees.md`](Senior/reflection-expression-trees.md) | Reflection, expression trees, dynamic |
| [`source-generators.md`](Senior/source-generators.md) | Source generators (.NET 5+) |
| [`memory-pooling.md`](Senior/memory-pooling.md) | ArrayPool, ObjectPool, MemoryPool |
| [`unsafe-pointers.md`](Senior/unsafe-pointers.md) | unsafe, fixed, stackalloc, ref struct |
| [`csharp-language-design.md`](Senior/csharp-language-design.md) | History, design decisions, evolution |
| [`csharp-vs-other-langs.md`](Senior/csharp-vs-other-langs.md) | C# vs Java/Go/Rust/Python/TS |
| [`cli-tools-scripting.md`](Senior/cli-tools-scripting.md) | System.CommandLine, scripting |
| [`desktop-frameworks.md`](Senior/desktop-frameworks.md) | WPF, MAUI, Avalonia |
| [`fenwick-bit.md`](Senior/fenwick-bit.md) ⭐ NEW | Fenwick Tree (BIT): префиксные суммы O(log n), трюк i & -i |

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
9. design-patterns (top 13 GoF)
10. functional-csharp
```

### Senior path

```
1. reflection-expression-trees
2. source-generators
3. memory-pooling, unsafe-pointers
4. gof-patterns-extended (8 advanced GoF)
5. numeric-types-math (SIMD, generic math)
6. csharp-language-design (понять design decisions)
7. cli-tools-scripting, desktop-frameworks (specific domains)
```

---

## 🔗 Связанные папки

- [`Runtime/`](../Runtime/) — CLR internals (GC, JIT, threading) — расширение глубже C#
- [`Performance/`](../Performance/) — performance optimization
- [`Architecture/`](../Architecture/) — паттерны в архитектурном контексте
- [`LearningPath/case-studies-top7.md`](../LearningPath/case-studies-top7.md) — production case studies
