# CSharp — язык

> 38 файлов / ~1.1 MB. Всё про C# как язык: от basics до advanced internals.

[← Главный README](../README.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Никогда не писал на C# | [`csharp-basics.md`](csharp-basics.md) → [`dotnet-cli-getting-started.md`](dotnet-cli-getting-started.md) |
| Знаю Java/Python, перехожу | [`csharp-vs-other-langs.md`](csharp-vs-other-langs.md) |
| Junior — daily tools | [`debugging-basics.md`](debugging-basics.md), [`naming-conventions.md`](naming-conventions.md) |
| Junior хочу в Middle | [`oop.md`](oop.md) → [`error-handling.md`](error-handling.md) → [`collections-linq.md`](collections-linq.md) |
| Middle хочу в Senior | [`async-threading.md`](async-threading.md) → [`types-and-memory.md`](types-and-memory.md) → [`generics-deep.md`](generics-deep.md) |
| Готовлюсь к собесу | [`async-threading.md`](async-threading.md), [`types-and-memory.md`](types-and-memory.md), [`collections-linq.md`](collections-linq.md), [`design-patterns.md`](design-patterns.md), [`gof-patterns-extended.md`](gof-patterns-extended.md) |
| Senior performance | [`memory-pooling.md`](memory-pooling.md), [`unsafe-pointers.md`](unsafe-pointers.md), [`numeric-types-math.md`](numeric-types-math.md) |

---

## 📚 Все файлы по уровню

### 🌱 Junior — старт и инструменты

| Файл | Описание |
|------|----------|
| [`csharp-basics.md`](csharp-basics.md) | Стартовая точка: переменные, типы, control flow |
| [`dotnet-cli-getting-started.md`](dotnet-cli-getting-started.md) ⭐ NEW | dotnet CLI, project templates, package management |
| [`debugging-basics.md`](debugging-basics.md) ⭐ NEW | Отладка: breakpoints, watch, immediate, logging |
| [`naming-conventions.md`](naming-conventions.md) ⭐ NEW | Naming: PascalCase, camelCase, conventions |

### 🌿 Junior to Middle — daily work

| Файл | Описание |
|------|----------|
| [`datetime-timezones.md`](datetime-timezones.md) | DateTime, TimeZoneInfo, NodaTime |
| [`strings-regex.md`](strings-regex.md) | String operations, Regex, performance |
| [`enums-flags.md`](enums-flags.md) | Enum, [Flags], parsing, serialization |
| [`tuples-deconstruction.md`](tuples-deconstruction.md) | ValueTuple, deconstruction |
| [`anonymous-types.md`](anonymous-types.md) | Anonymous types, when и зачем |
| [`extension-methods.md`](extension-methods.md) | this-параметр, fluent APIs, LINQ-like |
| [`iterators-yield.md`](iterators-yield.md) | yield return, IAsyncEnumerable |
| [`oop.md`](oop.md) | Inheritance, polymorphism, abstract (Junior to Senior) |
| [`collections-linq.md`](collections-linq.md) | Collections, LINQ deep (47 KB, Junior to Senior) |

### 🌳 Middle — production work

| Файл | Описание |
|------|----------|
| [`io-streams.md`](io-streams.md) | File, Stream, async I/O |
| [`nullable-types.md`](nullable-types.md) | Nullable\<T\>, NRT, null operators |
| [`equality-comparison.md`](equality-comparison.md) | Equals, GetHashCode, IEquatable |
| [`attributes-metadata.md`](attributes-metadata.md) | Attributes, custom attributes, reflection |
| [`indexers-operators.md`](indexers-operators.md) | Custom indexers, operator overloading |
| [`dispose-pattern.md`](dispose-pattern.md) | IDisposable, IAsyncDisposable, SafeHandle |
| [`keywords-reference.md`](keywords-reference.md) | C# keywords reference (ref/in/out/scoped/etc) |
| [`error-handling.md`](error-handling.md) | Exceptions, Result, OneOf (Middle to Senior) |
| [`delegates-events.md`](delegates-events.md) | Delegates, events, Func/Action (Middle to Senior) |
| [`modern-features.md`](modern-features.md) | Records, primary ctors, raw strings (Middle to Senior) |

### 🏔️ Middle to Senior

| Файл | Описание |
|------|----------|
| [`generics-deep.md`](generics-deep.md) | Variance, INumber\<T\>, generic math |
| [`numeric-types-math.md`](numeric-types-math.md) ⭐ | BigInteger, Half, Vector\<T\>, SIMD |

### 🏆 Senior

| Файл | Описание |
|------|----------|
| [`async-threading.md`](async-threading.md) | Task, async/await internals (58 KB) ⭐ |
| [`types-and-memory.md`](types-and-memory.md) | Value vs reference, boxing, struct (53 KB) ⭐ |
| [`functional-csharp.md`](functional-csharp.md) | Records, pattern matching, FP в C# |
| [`design-patterns.md`](design-patterns.md) | 13 GoF patterns в C# |
| [`gof-patterns-extended.md`](gof-patterns-extended.md) ⭐ | 8 ещё GoF: Command, Visitor, Composite, Proxy, Memento, Bridge, Flyweight, Prototype |
| [`reflection-expression-trees.md`](reflection-expression-trees.md) | Reflection, expression trees, dynamic |
| [`source-generators.md`](source-generators.md) | Source generators (.NET 5+) |
| [`memory-pooling.md`](memory-pooling.md) ⭐ | ArrayPool, ObjectPool, MemoryPool |
| [`unsafe-pointers.md`](unsafe-pointers.md) ⭐ | unsafe, fixed, stackalloc, ref struct |
| [`csharp-language-design.md`](csharp-language-design.md) | History, design decisions, evolution |
| [`csharp-vs-other-langs.md`](csharp-vs-other-langs.md) | C# vs Java/Go/Rust/Python |
| [`cli-tools-scripting.md`](cli-tools-scripting.md) | System.CommandLine, scripting |
| [`desktop-frameworks.md`](desktop-frameworks.md) | WPF, MAUI, Avalonia |

---

## 🎓 Рекомендуемый порядок изучения

### Junior path (3-6 мес)

```
1. csharp-basics
2. dotnet-cli-getting-started  ← запустить первый проект
3. debugging-basics             ← отлаживать с первого дня
4. naming-conventions           ← привычки правильно
5. oop (Junior сектор)
6. collections-linq (Junior сектор)
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
7. io-streams
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

- [`Runtime/`](../Runtime/) — CLR internals (GC, JIT, threading) — **расширение** глубже C#
- [`Performance/`](../Performance/) — performance optimization
- [`Architecture/design-patterns`](../Architecture/) — patterns в архитектурном контексте
- [`LearningPath/case-studies-top7.md`](../LearningPath/case-studies-top7.md) ⭐ — production case studies
