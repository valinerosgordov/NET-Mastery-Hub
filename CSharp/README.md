# CSharp — язык

> 29 файлов / 933 KB. Всё про C# как язык: от basics до advanced internals.

[← Главный README](../README.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Никогда не писал на C# | [`csharp-basics.md`](csharp-basics.md) |
| Знаю Java/Python, перехожу | [`csharp-vs-other-langs.md`](csharp-vs-other-langs.md) |
| Junior хочу в Middle | [`oop.md`](oop.md) → [`error-handling.md`](error-handling.md) → [`collections-linq.md`](collections-linq.md) |
| Middle хочу в Senior | [`async-threading.md`](async-threading.md) → [`types-and-memory.md`](types-and-memory.md) → [`generics-deep.md`](generics-deep.md) |
| Готовлюсь к собесу | [`async-threading.md`](async-threading.md), [`types-and-memory.md`](types-and-memory.md), [`collections-linq.md`](collections-linq.md), [`design-patterns.md`](design-patterns.md) |

---

## 📚 Все 29 файлов

### 🌱 Junior — основы

| Файл | Описание |
|------|----------|
| [`csharp-basics.md`](csharp-basics.md) | Стартовая точка: переменные, типы, control flow |

### 🌿 Junior to Middle — daily work

| Файл | Описание |
|------|----------|
| [`datetime-timezones.md`](datetime-timezones.md) | DateTime, TimeZoneInfo, NodaTime |
| [`strings-regex.md`](strings-regex.md) | String operations, Regex, performance |
| [`enums-flags.md`](enums-flags.md) | Enum, [Flags], parsing, serialization |
| [`tuples-deconstruction.md`](tuples-deconstruction.md) | ValueTuple, deconstruction |
| [`extension-methods.md`](extension-methods.md) | this-параметр, fluent APIs, LINQ-like |
| [`iterators-yield.md`](iterators-yield.md) | yield return, IAsyncEnumerable |

### 🌳 Middle — production work

| Файл | Описание |
|------|----------|
| [`oop.md`](oop.md) | Inheritance, polymorphism, abstract |
| [`error-handling.md`](error-handling.md) | Exceptions, Result, OneOf |
| [`io-streams.md`](io-streams.md) | File, Stream, async I/O |
| [`nullable-types.md`](nullable-types.md) | Nullable\<T\>, NRT, null operators |
| [`equality-comparison.md`](equality-comparison.md) | Equals, GetHashCode, IEquatable |
| [`attributes-metadata.md`](attributes-metadata.md) | Attributes, custom attributes, reflection |
| [`indexers-operators.md`](indexers-operators.md) | Custom indexers, operator overloading |
| [`dispose-pattern.md`](dispose-pattern.md) | IDisposable, IAsyncDisposable, SafeHandle |
| [`delegates-events.md`](delegates-events.md) | Delegates, events, Func/Action |
| [`collections-linq.md`](collections-linq.md) | Collections, LINQ deep (47 KB) |

### 🏔️ Middle to Senior

| Файл | Описание |
|------|----------|
| [`generics-deep.md`](generics-deep.md) | Variance, INumber\<T\>, generic math |
| [`modern-features.md`](modern-features.md) | Records, primary ctors, raw strings |

### 🏆 Senior

| Файл | Описание |
|------|----------|
| [`async-threading.md`](async-threading.md) | Task, async/await internals (58 KB) ⭐ |
| [`types-and-memory.md`](types-and-memory.md) | Value vs reference, boxing, struct (53 KB) ⭐ |
| [`functional-csharp.md`](functional-csharp.md) | Records, pattern matching, FP в C# |
| [`design-patterns.md`](design-patterns.md) | 13 GoF patterns в C# |
| [`reflection-expression-trees.md`](reflection-expression-trees.md) | Reflection, expression trees, dynamic |
| [`source-generators.md`](source-generators.md) | Source generators (.NET 5+) |
| [`csharp-language-design.md`](csharp-language-design.md) | History, design decisions, evolution |
| [`csharp-vs-other-langs.md`](csharp-vs-other-langs.md) | C# vs Java/Go/Rust/Python |
| [`cli-tools-scripting.md`](cli-tools-scripting.md) | System.CommandLine, scripting |
| [`desktop-frameworks.md`](desktop-frameworks.md) | WPF, MAUI, Avalonia |

---

## 🔗 Связанные папки

- [`Runtime/`](../Runtime/) — CLR internals (GC, JIT, threading) — расширение глубже C#
- [`Performance/`](../Performance/) — performance optimization
- [`Architecture/design-patterns`](../Architecture/) — patterns в архитектурном контексте
