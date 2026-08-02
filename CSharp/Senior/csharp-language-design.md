---
tags: [csharp, language-design, senior, csharp-evolution, philosophy, lang-design, history]
level: Senior
date: 2026-08-02
---

# C# Language Design — философия и эволюция языка

> **Принципы проектирования C#, design committee, эволюция через 14 версий, trade-offs.** Зачем добавили (и не добавили) features, как принимаются решения, что отличает C# от Java/Kotlin/F#. Закрывает пробел: «знаю синтаксис, не понимаю почему C# такой, и зачем records появились в C# 9, а primary constructors — в C# 12».

---

## 0. Как читать

Этот файл — **обзорный**. Не учит синтаксису, а объясняет philosophy. Полезен для архитекторов / тех-leads, которые принимают решения about adoption новых фич, для интервью по C# fundamentals и для понимания будущего языка.

Раздел 1 — фундаментальные principles. Раздел 2-4 — design committee и process. Раздел 5-8 — конкретные decisions. Раздел 9-10 — comparison + future.

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Что такое language design

**Language design** — set decisions определяющий: какие features add, в каком порядке, в какой syntax form, какие compromises делать. **Anders Hejlsberg** (Turbo Pascal, Delphi, J++) — chief architect C# 1.0-onwards. **Mads Torgersen** — language design lead currently.

C# не "приставлен к .NET" — designed alongside Common Language Runtime (CLR). 24 years (2002-2026) consistent direction.

### 1.2. Главные guiding principles

```
1. Pragmatic, не purist
   - Steal good ideas из других languages (Java, Haskell, F#, Rust)
   - Не fall в ML-style functional purity
   - Real-world problems > theoretical purity

2. Performance не сomprоmised
   - Generics specialized (vs Java erasure)
   - Value types (struct) — true stack allocation
   - Allow unsafe / pointers для hot paths
   - SIMD / Span<T> built-in

3. Productivity для всех level developers
   - LINQ, async/await — democratize advanced patterns
   - Records, primary constructors — boilerplate gone
   - var, target-typed new — type inference

4. Backward compatibility almost-religious
   - Old code должен компилироваться 20 years later
   - New features rarely break existing
   - Long deprecation cycles
   - Contextual keywords для new — не reserved

5. Type safety + null safety incremental
   - Nullable Reference Types (C# 8+) opt-in
   - Pattern matching + exhaustiveness
   - Strong types, минимум runtime checks
```

### 1.3. Главное правило adoption

```
Adopt feature когда:
  - Решает реальную проблему в твоём codebase
  - Team understands semantics
  - Tooling supported (IDE, analyzers)
  - Migration cost < benefit

НЕ adopt только потому что новое:
  - Premature adoption shines bug-prone
  - Mixed style hard to read
  - "Modern code" не цель — "correct code" цель
```

### 1.4. Эволюция philosophy

| Период | Focus | Examples |
|--------|-------|----------|
| **2002-2005 (C# 1-2)** | Match Java + добавить generics | generics, nullable, iterators |
| **2007-2010 (C# 3-4)** | Functional programming | LINQ, lambda, dynamic |
| **2012-2015 (C# 5-6)** | Async-first | async/await, exception filters |
| **2017-2019 (C# 7-8)** | Pattern matching, perf | tuples, ref returns, NRT, async streams |
| **2020-2022 (C# 9-11)** | Records, immutability | records, init, top-level, raw strings |
| **2023+ (C# 12+)** | Boilerplate removal | primary constructors, collection expressions, field |

> [!info]- Если ты знаешь Java / Kotlin / Swift / Rust
> **Java:** evolves slower (committee process, JEPs). Conservative. Backward compat священное. C# vs Java: more pragmatic, faster evolution, better generics, value types true.
>
> **Kotlin:** designed to fix Java's gripes. Targets JVM. Borrow from C# (extension methods, `let`/`apply` ↔ pattern matching). Less aggressive перешел performance.
>
> **Swift:** Apple-sponsored. Move pragmatic, перенял много от C# и Rust. Strong protocol-oriented programming.
>
> **Rust:** memory safety без GC — different beast. C# не aspires к ownership semantics. Both languages converge на certain patterns (nullable, pattern matching, records-like structs).

> [!question]- Интервью: какова философия C# языка?
> 5 принципов: 1) **Pragmatic** — steal good ideas из других langs (Java/Haskell/F#/Rust), не purist. 2) **Performance уважается** — generics specialized, value types real, unsafe возможен. 3) **Productivity** — LINQ/async/records reduce boilerplate. 4) **Backward compatibility** — code 20 years old должен компилироваться. New features rarely break. 5) **Type safety incremental** — NRT opt-in, pattern matching exhaustive, strong types. Lead architect Mads Torgersen ([@madstorgersen](https://twitter.com/madstorgersen)). C# evolves faster чем Java, careful чем Scala, мощнее чем Go.

---

## 2. Design committee и process

### 2.1. C# Language Design Meeting (LDM)

**LDM** — weekly meeting где обсуждаются proposals. Participants: Microsoft language team + invited community contributors.

```
Roles:
- Mads Torgersen — language design lead
- Jared Parsons — C# compiler lead
- Other Microsoft engineers
- Community members (специальные guests)
```

Notes публикуются на [GitHub csharplang](https://github.com/dotnet/csharplang/tree/main/meetings).

### 2.2. Proposal lifecycle

```
1. Discussion на GitHub csharplang
   ↓
2. Champion (LDM member) takes ownership
   ↓
3. Proposal document written
   ↓
4. LDM reviews several times — refinement
   ↓
5. Specification written
   ↓
6. Compiler implementation
   ↓
7. Preview version (.NET preview SDK)
   ↓
8. Stabilization + community feedback
   ↓
9. Final release в C# version
```

Average: 1-3 years from proposal to ship.

### 2.3. Proposal sources

```
- Microsoft engineers — internal needs (azure / .NET libraries)
- Community — GitHub issues / discussions
- Other languages — borrow from F# / Rust / Kotlin
- Conferences — feedback от MVP / users
- Customer escalations — pain points
```

Examples successfully community-driven:
- Top-level statements (proposal от MVP)
- Records (community pressure для immutability)
- Primary constructors для classes (DI boilerplate complaints)

### 2.4. Decision criteria

```
✅ Add feature если:
  - Solves common problem (not edge case)
  - Composable с existing features
  - Won't conflict с future direction
  - Implementation tractable (not too complex)
  - Educational cost reasonable

❌ Reject feature если:
  - Marginal use case (1% codebase)
  - Conflicts с другими features
  - Adds complexity без proportional benefit
  - Better solved at library level
```

### 2.5. Examples decisions

**Принято:**
- Records (C# 9) — immutability + value equality common need.
- Primary constructors (C# 12) — DI boilerplate everywhere.
- Pattern matching — extends existing switch.

**Отложено / отклонено:**
- Discriminated unions — long-discussed (since C# 7), почти десятилетие в proposals; первый preview — **C# 15** (.NET 11 Preview 2, апрель 2026), GA ожидается ~ноябрь 2026. Пример «отложено надолго ≠ отклонено».
- Type classes — too complex для mainstream.
- Operator constraints в generics — replaced static abstract members + `INumber<T>`.

### 2.6. Specs vs proposals

```
- /proposals — early ideas, in progress
- /spec — формальная language specification
- /meetings — LDM notes
```

В csharplang repo. Anyone может read decisions history.

> [!question]- Интервью: как принимаются решения в C# language design?
> **C# Language Design Meeting (LDM)** — weekly Microsoft + community participants. Proposals lifecycle: GitHub discussion → champion takes ownership → document → LDM reviews → spec → compiler → preview → final. Average 1-3 years. Decision criteria: solves common problem (not edge), composable с existing, future-compatible, implementation tractable, educational cost reasonable. Examples: records (C# 9, community pressure), primary constructors classes (C# 12, DI complaints), discriminated unions (десятилетие обсуждений → preview в C# 15). Notes публичные на github.com/dotnet/csharplang/meetings.

---

## 3. Backward compatibility — sacred cow

### 3.1. Almost-religious commitment

```
"You shouldn't have to rewrite working code when upgrading C#."
                                          — Mads Torgersen
```

Code написанный для C# 1.0 (2002) **компилируется** в C# 14 (2025). 24 years guarantee.

### 3.2. How preserved

```
1. Contextual keywords — new keywords не break existing identifiers
   - `var` (C# 3) — был bare identifier
   - `async`/`await` (C# 5) — same
   - `record` (C# 9) — same
   
2. Opt-in features через language version
   - <LangVersion>10</LangVersion>
   - Не все default — некоторые preview
   
3. Soft deprecation
   - [Obsolete] warnings, не errors
   - Old patterns continue working
   
4. Compatibility shims
   - .NET Standard provides surface для libraries
   - Type forwarding между assemblies
```

### 3.3. Breaking changes — rare

Examples (минимальные):
- C# 6: некоторые edge cases overload resolution.
- C# 8: NRT может flag existing code (warnings, не errors — opt-in).
- C# 9: `record` keyword (но contextual — не breaks identifier `record`).
- C# 11: `required` keyword (тоже contextual).

Microsoft публикует [breaking changes](https://learn.microsoft.com/dotnet/csharp/whats-new/breaking-changes) — обычно 0-2 per release.

### 3.4. Trade-off: legacy patterns

Цена backward compat:
- `delegate` keyword + `Func<>` — duplicated.
- `IEnumerable.Cast<T>()` — predates LINQ, не perfect.
- `Hashtable` (1.0) + `Dictionary<TKey,TValue>` (2.0) — duplicate.
- `ArrayList` + `List<T>` — same.
- Property syntax вариаций (auto-implemented C# 3 → init C# 9 → primary ctor C# 12).

C# carries weight of 24-year history. Not always clean.

### 3.5. ApplicationException — example deprecated

```csharp
// .NET 1.0: рекомендовали как base custom exception
public class MyException : ApplicationException { }

// Microsoft retracted: "use Exception вместо"
// Но ApplicationException остаётся navigable
// Никогда удалили — backward compat
```

### 3.6. Why это важно

Enterprise codebase: миллионы lines, decades old, shared libraries. Если updating C# breaks code — never update. Microsoft chose: **slow careful evolution** > fast clean rebuild. Stability над elegance.

> [!question]- Интервью: почему C# так сильно предан backward compatibility?
> **Microsoft enterprise customers** — крупные codebases (миллионы lines), legacy libraries, third-party dependencies. Если C# upgrade ломает code — никто не upgrade. Stability > elegance. Result: code C# 1.0 (2002) **compiles** в C# 14 (2025). New features через **contextual keywords** (`var`, `async`, `record`, `required`, `extension`) — не reserved (existing identifiers сохраняются). Opt-in через `<LangVersion>` или `<Nullable>`. Trade-offs: legacy patterns остаются (`delegate` + `Func<>`, `Hashtable` + `Dictionary`), некоторые corners ugly. Stability paid via constraints — language carries 24-year baggage.

---

## 4. Generics design — лучше Java

### 4.1. Reified vs erased

```
Java erasure:
- List<String>, List<Integer> — оба runtime List<Object>
- Cannot ask "what was T at runtime?"
- Boxing для primitives
- Один classfile compile-time

C# reification:
- `List<int>`, `List<string>` — runtime distinct types
- typeof(T) works in runtime
- No boxing для value types
- JIT specializes per value T (separate machine code)
- Reference T — shared code (one specialization)
```

### 4.2. Performance gains

```csharp
// C# — no boxing
List<int> list = new();
list.Add(42);   // direct stack-to-internal-array, no allocation

// Java — boxing
List<Integer> list = new ArrayList<>();
list.add(42);   // boxes int → Integer (heap allocation!)
```

### 4.3. Variance — `in`/`out`

```csharp
// C# — explicit variance
public interface IEnumerable<out T> { }
public interface IComparer<in T> { }

// Java — wildcards (use site)
List<? extends Number> coVariant;
List<? super Integer> contraVariant;
// Less explicit, more verbose at use site
```

C# variance — declaration-site (cleaner). Java — use-site (more flexible но verbose).

### 4.4. Constraints

```csharp
where T : class
where T : struct
where T : notnull       // C# 8+
where T : unmanaged     // C# 7.3+
where T : new()
where T : SomeBase
where T : IInterface
where T : Enum, Delegate (C# 7.3+)
where T : INumber<T> (C# 11+)
```

Java — interface constraints `<T extends Comparable<T>>`. C# — full breadth + struct/class/enum.

### 4.5. Generic Math (.NET 7+)

```csharp
public T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var v in values) sum += v;
    return sum;
}
```

`static abstract` interface members + `INumber<T>` — generic math possible. **Невозможно в Java** (no operator overloading, no static abstract).

> [!question]- Интервью: чем generics в C# лучше Java?
> 1) **Reification** — runtime types preserved (Java erasure → `List<String>` runtime `List<Object>`). 2) **No boxing** для value types — List<int> stores int directly (Java `List<Integer>` boxes int → Integer). 3) **Specialization** — JIT generates separate machine code per value T (Java один classfile). 4) **Explicit variance** через `in`/`out` (declaration site cleaner чем Java wildcards use site). 5) **Constraints** — class/struct/notnull/unmanaged/`INumber<T>` (Java только interface constraints). 6) **Generic Math** (.NET 7+) с `static abstract` members — impossible в Java. Trade-offs: bigger binary footprint при много специализаций.

---

## 5. Async/await — democratized concurrency

### 5.1. До C# 5 — callback hell

```csharp
// .NET 1.0-3.5 — IAsyncResult / Begin*/End* pattern
client.BeginRead(buffer, 0, buffer.Length, ar =>
{
    var bytesRead = client.EndRead(ar);
    // process...
    client.BeginRead(buffer, 0, buffer.Length, ar2 =>
    {
        // nested callbacks...
    }, null);
}, null);

// Или Task.ContinueWith — better but still verbose
client.ReadAsync(buffer, 0, buffer.Length)
    .ContinueWith(t =>
    {
        var bytesRead = t.Result;
        // process...
    });
```

### 5.2. C# 5 — async/await

```csharp
// Looks sync, runs async
public async Task<int> ReadAsync()
{
    var bytesRead = await client.ReadAsync(buffer, 0, buffer.Length);
    // process...
    return bytesRead;
}
```

Compiler transforms → state machine. Сохраняет stack semantics, exceptions propagate normally.

### 5.3. Innovation impact

```
До async/await:
- Threading.Thread, ManualResetEvent — primitive
- BackgroundWorker — UI specific
- Asynchronous Programming Model (APM) — verbose
- Event-based Asynchronous Pattern (EAP) — UI events

После async/await:
- ASP.NET Core полностью async
- EF Core async queries
- All BCL I/O async
- IAsyncEnumerable<T> (C# 8) — streaming
```

### 5.4. Borrowed by other langs

C# 5 (2012) — first mainstream language с async/await. Inspired:
- **JavaScript** (ES2017+) — same syntax, same semantics.
- **Python** (3.5+) — `async def`, `await`.
- **Rust** (1.39+) — `.await` operator.
- **Kotlin** — coroutines (different but similar UX).

C# influence на async programming в industry — massive.

### 5.5. Design trade-offs

```
✅ Choice: stackful vs stackless
   - Stackful (goroutines, fibers): each task own stack
   - Stackless: state machines, light-weight
   - C# chose stackless — lower memory, more tasks possible

✅ Choice: explicit await
   - Implicit (Haskell, ML) — IO type
   - Explicit (C#, JS) — visual marker где blocking
   - C#: explicit для readability

❌ Trade-off: function color
   - sync function не может await
   - "function color problem" Bob Nystrom
   - Все methods up the stack должны быть async
```

### 5.6. Future — async streams + cancellation

C# 8: `IAsyncEnumerable<T>` + `await foreach`. Streaming.
C# 8+: `[EnumeratorCancellation]` + CancellationToken.
.NET 7+: `ValueTask` improvements для no-allocation paths.

> [!question]- Интервью: какой impact async/await в industry?
> C# 5 (2012) **first mainstream language** с async/await. До этого: callbacks (APM), continuations (Task.ContinueWith), thread pool. С async/await: looks like sync code, runs async через compiler-generated state machine. Inspired JavaScript ES2017+, Python 3.5+, Rust 1.39+. Result: ASP.NET Core fully async (huge scalability gain), EF Core async queries, all BCL I/O async. Trade-offs: "function color problem" — sync не может await, всё up the stack должно быть async. C# 8 added `IAsyncEnumerable<T>` для streams. **Most impactful C# feature ever** (по influence на industry).

---

## 6. LINQ — composable queries

### 6.1. C# 3 (2008) — paradigm shift

```csharp
// До LINQ — manual loops
List<User> result = new();
foreach (var u in users)
{
    if (u.Age > 18)
        result.Add(new User { Name = u.Name.ToUpper() });
}

// LINQ
var result = users
    .Where(u => u.Age > 18)
    .Select(u => new User { Name = u.Name.ToUpper() })
    .ToList();

// LINQ query syntax (rare в практике, выглядит как SQL)
var result2 = from u in users
              where u.Age > 18
              select new User { Name = u.Name.ToUpper() };
```

### 6.2. Foundation — extension methods + lambda + IEnumerable

LINQ requires:
- **Lambda expressions** (anonymous functions).
- **Extension methods** (Where/Select на `IEnumerable<T>`).
- **Type inference** — `var result = ...`.
- **`IEnumerable<T>`** + lazy evaluation.

C# 3 added all four — synergistic effect. LINQ был **emergent** от features, не only goal.

### 6.3. Expression trees

```csharp
// Lambda → delegate (compiled)
Func<int, bool> f = x => x > 5;

// Lambda → expression tree (data structure!)
Expression<Func<int, bool>> e = x => x > 5;
// e.Body = BinaryExpression { Left=x, Operator=>, Right=5 }
```

Critical для **LINQ to SQL / EF Core**:
- Provider parses expression tree → SQL.
- Translation compile-time-ish (no SQL parsing).

### 6.4. Lazy evaluation

```csharp
var query = bigList
    .Where(x => Expensive(x))
    .Select(x => Transform(x));
// nothing executed yet!

foreach (var item in query) { /* ... */ }   // execution starts here
```

Allows composing operations без intermediate collections. Key efficiency gain.

### 6.5. Influence

LINQ inspired:
- **Java Streams** (Java 8, 2014) — direct ancestor.
- **Kotlin** sequences — similar.
- **Rust** iterator combinators — similar concept.
- **JavaScript** array methods (`map`/`filter`/`reduce`) — predates но aligned.

> [!question]- Интервью: почему LINQ был revolution в C#?
> LINQ (C# 3, 2008) — **first mainstream language** с unified query syntax для in-memory + DB. **Foundation pieces**: lambda expressions, extension methods, type inference (var), IEnumerable + IQueryable. **Lazy evaluation** — composable operations без intermediate copies. **Expression trees** — lambdas как data structures, parseable providers (LINQ to SQL, EF Core translates to SQL). **Inspired Java Streams** (2014), Kotlin sequences. До LINQ — manual loops + intermediate lists. После — declarative pipelines. **Most adopted feature** после async/await.

---

## 7. Records — immutability done right

### 7.1. Long road к records

C# 9 (2020) добавил records — но **обсуждалось** с C# 6 (2014). 6 years discussion, prototypes, design iterations.

```
Discussions:
- C# 6 — initial proposal (rejected — too complex)
- C# 7 — "tuple types" partial answer
- C# 8 — preview records
- C# 9 — final shipping
```

### 7.2. Tension — class vs struct

```
Records должен быть:
- Class (reference) — for inheritance
- Struct (value) — for immutability
- Both? — eventually

C# 9 — record class only
C# 10 — record struct (по requests)
```

### 7.3. Auto-generated functionality

```csharp
public record User(int Id, string Name);
// Compiler generates:
// - Constructor User(int Id, string Name)
// - Init properties Id, Name
// - Equals(User?), Equals(object?), GetHashCode
// - operator ==, !=
// - Deconstruct(out int Id, out string Name)
// - ToString → "User { Id = ..., Name = ... }"
// - with-expression support
// - PrintMembers virtual для inheritance
```

Hours of boilerplate gone.

### 7.4. Inheritance + value equality

```csharp
public record Base(int Id);
public record Derived(int Id, string Name) : Base(Id);

new Base(1) == new Derived(1, "x");   // false — type-aware
```

`EqualityContract` virtual property — runtime type check. Prevents Liskov violations в equality.

### 7.5. Trade-offs

```
✅ Pros:
   - Boilerplate gone
   - Immutable by convention (init properties)
   - Value equality automatic
   - with-expression для evolution

❌ Cons:
   - Не для mutable entities (records — value semantics)
   - Inheritance hierarchies become complex
   - Performance cost vs class — slight (auto-generated code не free)
   - Equals iterates all fields (performance с many fields)
```

### 7.6. Records vs F# / Scala / Kotlin

```
F# records:
type User = { Id: int; Name: string }
// Even less syntax, immutable by default

Scala case classes:
case class User(id: Int, name: String)
// Same idea, value equality, copy(...) instead of with

Kotlin data classes:
data class User(val id: Int, val name: String)
// Almost identical to C# records
```

C# records most similar to Scala case classes / Kotlin data classes. F# records simpler but more limited.

> [!question]- Интервью: почему records появились только в C# 9?
> **6 years discussion** (C# 6 → C# 9). Tensions: class vs struct (semantics), inheritance + value equality (subtle problem), syntax (positional vs named), interaction с existing features (auto-properties, init). Inspired by Scala case classes, Kotlin data classes, F# records. C# 9 chose **record class** (reference type, value equality). C# 10 added **record struct**. Auto-generated: ctor, init properties, Equals/GetHashCode/==, Deconstruct, ToString, PrintMembers (для inheritance). `EqualityContract` — runtime type check для type-aware equality. Trade-off: not для mutable entities, slight perf cost vs class.

---

## 8. NRT — incremental null safety

### 8.1. The problem

NullReferenceException — most common bug в .NET. Hoare called null reference his "billion-dollar mistake" (1965 invention).

C# 1-7: всё могло быть null. Compile-time checks нет. Runtime NRE everywhere.

### 8.2. C# 8 — Nullable Reference Types (opt-in)

```csharp
#nullable enable

string a = "hello";       // non-null promise
string? b = null;          // may be null

a = null;                  // ⚠ CS8600 warning
a.Length;                  // ⚠ CS8602 если a unknown null
```

**Compile-time only** — runtime semantics unchanged. Warnings, не errors (по default).

### 8.3. Why opt-in

Не break existing code:
- Old assemblies без NRT — `string` ambiguous.
- Migration gradual через `#nullable enable`.
- New projects — `<Nullable>enable</Nullable>` default (.NET 6+).

### 8.4. Limitations

NRT **не sound** — есть holes:
- Generic `T?` без constraint — confusing semantics.
- Reflection bypasses checks.
- Late initialization (`= null!`) escape hatch.
- Constructors с required fields — pre-C# 11 проблема.

Microsoft acknowledges: NRT не bulletproof, но "good enough" — catches majority bugs.

### 8.5. C# 11 `required` — closes gaps

```csharp
public class User
{
    public required string Email { get; init; }   // compile-time guarantee
    public required string Name { get; init; }
}

new User { Email = "x", Name = "Alice" };   // OK
new User { };   // ❌ compile error
```

Fills hole "uninitialized non-null property in constructor".

### 8.6. Future — null safety

NRT continue evolving:
- Better generic nullability.
- Flow analysis improvements.
- Maybe — sound nullable system в future C# (long discussion).

### 8.7. Comparison Kotlin / TypeScript

```kotlin
// Kotlin — sound nullability built-in
val a: String = "hello"      // non-null
val b: String? = null         // nullable
b.length    // compile error
b?.length   // OK через safe call

// Compile-time GUARANTEED — no exception
```

Kotlin nullability — **sound** (no NRE if не используешь `!!` force unwrap). C# NRT — **best effort**, warnings, escapes via `!`. Kotlin stricter, C# pragmatic.

```typescript
// TypeScript — same opt-in approach as C#
let a: string = "hello";
let b: string | null = null;
// strictNullChecks flag controls
```

TypeScript nullable — also opt-in (`strictNullChecks`). Inspired C# NRT design.

> [!question]- Интервью: почему C# NRT opt-in, не sound by default?
> 1) **Backward compatibility** — old assemblies без annotations — `string` ambiguous, нельзя default treat as non-null без breaking. 2) **Migration gradient** — large codebases need `#nullable enable` per-file для incremental adoption. 3) **NRT compile-time only** — runtime semantics unchanged (no perf cost, но also no runtime guarantee). 4) **Holes**: generics без constraint, reflection bypasses, `null!` escape hatch. 5) **Compromise**: warnings, не errors — pragmatic over purist. Kotlin **sound** by default (no NRE без `!!`), но designed from scratch (1.0 in 2016) — no legacy. TypeScript NRT — same opt-in approach as C#.

---

## 9. C# vs other modern languages

### 9.1. Performance hierarchy

```
Native C/C++/Rust  >  C# (Native AOT)  >  C# (JIT)  >  Java  >  JavaScript  >  Python
   1x                    1.05x              1.1-1.3x      1.2-1.5x   2-10x         50-100x
```

C# perf comparable to Java, Native AOT closes gap to native. Reified generics + value types — key advantages.

### 9.2. Productivity hierarchy

```
Python ≈ JavaScript  >  Kotlin ≈ Swift  >  C#  >  Java  >  Rust  >  C++
```

C# — middle ground: more ceremonious than Python, less than Java/Rust.

### 9.3. Type system

| | C# | Java | Kotlin | TypeScript | F# | Rust |
|---|----|------|--------|------------|-----|------|
| **Generics** | Reified | Erased | JVM erased | Structural | Hindley-Milner | Monomorphized |
| **Null safety** | Opt-in NRT | Optional | Sound | Opt-in | `Option<T>` | `Option<T>` |
| **Pattern matching** | Yes (8+) | Switch (21+) | When | Yes | Yes | Yes |
| **Discriminated unions** | Preview (C# 15) | Sealed (17+) | Sealed | Yes (union types) | Yes | Yes |
| **Records** | Yes (9+) | Yes (14+) | Data class | No | Yes | Yes (struct) |

### 9.4. Ecosystem comparison

```
C#:
✅ NuGet — 400,000+ packages
✅ ASP.NET Core, EF Core, Blazor
✅ Visual Studio + Rider — top IDEs
✅ Azure first-class
❌ Cross-platform desktop weaker (MAUI, Avalonia еще growing)
❌ Mobile - second-class (Xamarin → MAUI)

Java:
✅ Mature ecosystem (30 years)
✅ Spring, Hibernate, Apache projects
✅ Android (Java/Kotlin)
❌ Slower evolution
❌ JVM startup time

Kotlin:
✅ JVM compatible (Java libs)
✅ Modern syntax
✅ Multiplatform (Android, iOS, JS, Native)
❌ Smaller ecosystem
❌ Tooling less mature

Rust:
✅ Safety без GC
✅ Performance native
✅ Crates.io growing
❌ Steep learning curve
❌ Slower development cycle
```

### 9.5. C# strengths

```
✅ Backward compat (24 years stable)
✅ Performance (reified generics, value types, AOT)
✅ Tooling (VS, Rider top tier)
✅ Microsoft backing — enterprise reliability
✅ Cross-platform (.NET 5+ — Windows/Linux/Mac/iOS/Android)
✅ Pragmatic feature additions
```

### 9.6. C# weaknesses

```
❌ Verbose vs Python/Kotlin — boilerplate в places (хотя 9-14 уменьшил)
❌ Discriminated unions — пока только preview (C# 15 / .NET 11, GA ~ноябрь 2026)
❌ Higher kinded types missing
❌ AOT ecosystem early (2024+)
❌ Mobile second-class
❌ Type inference более ограничен чем F# / Rust
```

> [!question]- Интервью: что отличает C# от Java?
> 1) **Reified generics** (vs Java erasure) — no boxing, runtime type info, JIT specialization. 2) **Value types** (struct) — true stack allocation, no boxing для primitives. 3) **Faster evolution** — C# 8-14 added: NRT, records, init, primary constructors, raw strings, collection expressions, extension members. Java equivalent slower (records 14, sealed 17). 4) **async/await** built-in (C# 5). Java только CompletableFuture / virtual threads (21+). 5) **LINQ** — Java added Streams (8) inspired by LINQ. 6) **Better tooling** — Visual Studio + Rider. **Java strengths**: more mature ecosystem, better Android, JVM ecosystem (Scala, Kotlin, Clojure share JVM).

---

## 10. Future — discriminated unions, more

### 10.1. Long-anticipated — discriminated unions: уже в preview

Дошло до реализации: первый preview — **C# 15** (.NET 11 Preview 2, апрель 2026), за `<LangVersion>preview</LangVersion>` + `net11.0`. GA ожидается с .NET 11 (~ноябрь 2026); синтаксис до GA может измениться.

```csharp
// C# 15 preview — union поверх существующих case-типов
public record Circle(double Radius);
public record Square(double Side);

public union Shape(Circle, Square);

// Compiler enforces exhaustiveness при pattern matching
static double Area(Shape shape) => shape switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Square s => s.Side * s.Side
    // нет _ — компилятор знает, что set закрыт
};
```

Модель C# — **type union** (композиция существующих standalone-типов), а не tag union как в F#/Rust; value types в дефолтной реализации боксятся — simplicity over max performance. Workaround до GA — records + abstract base (sealed hierarchy) + `_ => throw`.

### 10.2. Type classes / higher-kinded types

```fsharp
// F# / Haskell — generic over type constructors
val map : ('a -> 'b) -> M<'a> -> M<'b> when M : Functor
```

C# не имеет HKT. Discussion ongoing, complex добавить с reified generics.

### 10.3. Extensions — shipped в C# 14

Долгий proposal «roles/extension types» приземлился в виде **extension members** — реальная фича C# 14 (.NET 10, ноябрь 2025):

```csharp
public static class UserExtensions
{
    extension(User user)
    {
        public string DisplayName => $"{user.FirstName} {user.LastName}";   // extension property
    }
}
```

Extension properties / static members / operators — компилируются в static методы, старый `this`-синтаксис остаётся валиден. Полноценные «roles» (именованные view-типы поверх существующих) в исходном виде не вышли — LDM сузил scope до members. Детали — [[modern-features|Modern C# Features]] раздел 12.

### 10.4. Sound null safety

NRT остаётся opt-in + best-effort. Sound (Kotlin-style) — proposed long-term.

### 10.5. Macros / metaprogramming

C# имеет:
- **Source generators** (compile-time codegen) — actively used.
- **Attributes + reflection** (runtime).

NOT planned: macros (C++/Rust style) — Microsoft considers source generators sufficient.

### 10.6. Native AOT — production ready

.NET 8+ Native AOT — significant push. Trim, JIT-less, faster startup. Future:
- All BCL fully AOT-compatible.
- Source generators replace reflection-heavy patterns.
- Smaller binary footprint.

### 10.7. Roadmap public

[csharplang Issues](https://github.com/dotnet/csharplang/issues) labeled "Proposed Champion" — likely future versions. No promises.

---

## 11. Best practices как architect

### 11.1. Adoption strategy

```
✅ Use в new projects:
  - <LangVersion>latest</LangVersion>
  - <Nullable>enable</Nullable>
  - records для DTO / value objects
  - Pattern matching
  - Primary constructors (C# 12+)
  - Collection expressions (C# 12+)
  - Raw strings (C# 11+)
  - File-scoped namespaces

⚠️ Cautious adoption (mixed teams):
  - Generic Math (.NET 7+) — niche
  - field keyword (C# 14) — стабилен, но проверь что команда понимает семантику
  - Discriminated unions (C# 15 preview) — до GA только эксперименты

❌ Avoid в legacy projects:
  - NRT migration disruptive — gradual, per-file
  - Records replacing existing classes — careful (equality semantics changed!)
```

### 11.2. Migration strategy

```
1. <LangVersion>latest</LangVersion> — opt into new syntax
2. Per-file <#nullable enable> — gradual NRT
3. Refactor DTOs → records (one-by-one)
4. Adopt pattern matching incrementally
5. Adopt primary constructors per-file
6. NEVER mass-rewrite — feature-by-feature
```

### 11.3. Team education

```
Adopt feature только когда:
- 80% team понимает semantics
- Team agrees на code style
- Tooling supports (analyzers, IDE)
- Documentation written

Avoid:
- "Modern style" pressure
- One person uses, rest confused
```

### 11.4. Tracking C# evolution

Resources:
- **Microsoft Docs — What's new** — official.
- **csharplang GitHub** — proposals + LDM notes.
- **Mads Torgersen Twitter** — language design lead.
- **Andrew Lock blog** — andrewlock.net.
- **Stephen Cleary blog** — async patterns.
- **DevBlogs Microsoft** — feature deep dives.

### 11.5. Не делай

```
❌ Mix old and new в одном file (jarring)
❌ Adopt feature без understanding
❌ Use cutting-edge в production critical paths
❌ Refactor working code только потому что новая версия
```

---

## 12. Decision tree — adoption

```
Новый feature вышел?
│
├── Решает реальную проблему в моём codebase? → продолжаем
│   ├── Tooling supported (VS, Roslyn analyzers)?
│   ├── Team education ready?
│   ├── Production-tested (не preview)?
│   └── ⇒ Adopt
│
├── Marginal / theoretical use case?
│   └── Skip
│
├── Хочется попробовать?
│   ├── Side project — yes
│   ├── Prototype — yes
│   └── Production critical — wait
│
└── Migration существующего кода?
    ├── Cost > benefit? → skip
    ├── Per-file gradual → yes
    └── Mass rewrite → no
```

---

## 13. Cheat sheet — milestones

```
C# 1.0 (2002) — Java-like + classes/structs
C# 2.0 (2005) — Generics, nullable, iterators
C# 3.0 (2008) — LINQ, lambda, var, extension methods
C# 4.0 (2010) — dynamic, named/optional params
C# 5.0 (2012) — async/await
C# 6.0 (2015) — String interpolation, nameof, exception filters
C# 7.0 (2017) — Tuples, ref returns, pattern matching basic
C# 7.1-7.3 (2017-18) — async Main, default literals
C# 8.0 (2019) — NRT, async streams, ranges/indices, switch expr, default interface methods
C# 9.0 (2020) — Records, init, top-level, target-typed new, source generators
C# 10.0 (2021) — File-scoped namespaces, global usings, record struct
C# 11.0 (2022) — Raw strings, required, list patterns, generic attrs, static abstract
C# 12.0 (2023) — Primary constructors (all types), collection expressions
C# 13.0 (2024) — partial properties, params Span<T>, Lock type, field keyword (preview)
C# 14.0 (2025) — extension members, field (stable), ?.= assignment, partial ctors/events
C# 15.0 (~2026) — union types (preview в .NET 11)
```

---

## 14. Common pitfalls в design philosophy understanding

### 14.1. "C# slow" misconception

```
Заблуждение: C# slow because GC.

Реальность: 
- C# JIT + reified generics + value types ≈ Java performance
- Native AOT (.NET 8+) почти C++ performance
- GC pauses < 1ms with Server GC
- Allocation patterns (Span<T>, stackalloc) — zero alloc paths
```

### 14.2. "Records replace classes"

Records — для **immutable value semantics**. Classes остаются для:
- Mutable entities (User с changing state).
- Identity-based equality (User с Id).
- Inheritance hierarchies с behavior.

Не каждый class должен стать record.

### 14.3. "Modern всегда лучше"

Pattern matching — когда decisions tree подходит. Не replace всё через switch expression.

Records — когда value semantics important. Не replace existing classes.

Primary constructors — когда DI dependencies. Не replace explicit ctor с complex logic.

### 14.4. "C# competes with Rust для systems"

Не цель. C# — managed runtime + GC. Native AOT closes gap, но Rust остаётся для:
- Embedded systems (no GC).
- Operating systems.
- Browsers (Rust used в Firefox).

C# — application development, server-side, enterprise.

### 14.5. "Nullable Reference Types сломаются"

NRT compile-time only. Если игнорируешь warnings — runtime поведение **identical**. Никогда не **break** existing code, только helps catch bugs.

### 14.6. "Async — не для этого"

Async **не magic**. Не делает code parallel. Не speed up CPU-bound.

Async — для **I/O-bound**: file/network/DB. Освобождает thread waiting.

Для CPU-bound — `Task.Run` + parallelism (`Parallel.ForEach`, `IAsyncEnumerable<T>`, dataflow).

### 14.7. "Generics overhead"

C# generics specialization для value types — **no overhead**. JIT generates separate code.

Reference types — **shared code** для всех T (one specialization). Cost minimal.

Java generics erasure имеет boxing cost. C# не has это.

### 14.8. "Pattern matching expensive"

Pattern matching — **compile-time**. Compiler generates `if`/`switch` instructions. Runtime cost = same как manual `if (obj is T t)`.

### 14.9. "Records value equality slow"

`Equals` для records iterates all fields. Для most types — fast (compiler-generated optimal code).

Слабость только при **большое число fields** (~20+) — cache misses возможны. Override custom Equals если problem.

### 14.10. "NRT sound by default"

NRT — **best effort**, не sound. Holes: generics без constraint, reflection, `null!` escape, late init via `= null!`. Kotlin sound; C# pragmatic.

> [!question]- Интервью: почему C# evolves быстрее Java?
> 1) **Microsoft single owner** — language design lead имеет authority. Java через JCP committee + JEP process — slower. 2) **csharplang public process** — community feedback быстрее. 3) **Compiler open source** (Roslyn — github.com/dotnet/roslyn) — community contributions. 4) **Smaller ecosystem disruption** — Microsoft contrôle .NET fully. Java — Oracle + community (Spring, Apache, Eclipse) coordination. 5) **Annual cadence** — C# major release every year (Java now too но recent). C# 8-13 added: NRT, records, primary ctors, collection expr, raw strings. Java equivalent slower delivery cycle.

---

## 15. Cheat sheet philosophy

```
C# Design Tenets:
1. Pragmatic, не purist
2. Performance respected
3. Productivity для всех
4. Backward compat sacred
5. Type safety incremental

Process:
- C# Language Design Meeting (LDM) — weekly
- csharplang GitHub — public proposals
- Mads Torgersen — design lead
- Jared Parsons — compiler lead

Innovation impact:
- async/await — C# 5 (2012), inspired JS/Python/Rust
- LINQ — C# 3 (2008), inspired Java Streams
- Records — С# 9 (2020), borrowed Scala/Kotlin
- NRT — C# 8 (2019), inspired by Kotlin/TypeScript
- Source generators — C# 9, alternative reflection

Adoption strategy:
- New projects — adopt latest
- Legacy — gradual, per-file
- Mass rewrite — never
- Team education first

C# vs Java: faster evolution, better generics
C# vs Kotlin: more pragmatic, less sound
C# vs Rust: managed runtime, GC
C# vs F#: imperative-first, less FP
```

---

## 16. Practice — anti-tasks (architectural decisions)

### 16.1. Adopt records или нет?

Scenario: existing codebase 100k lines, DTO classes manually written с Equals/GetHashCode.

```
Decisions:
1. Existing DTO с Equals по всем полям — refactor → record
   ✅ Cleaner, no boilerplate
   
2. DTO с custom Equals (только Id) — DON'T refactor
   ❌ Records — value equality всех полей, semantic change
   
3. New DTO — always record
   ✅ Default in 2024+
```

### 16.2. NRT migration plan

Scenario: 10-year codebase, 500k lines, NRT not enabled.

```
Plan:
1. <Nullable>warnings</Nullable> — show warnings, не errors initially
2. Per-file <#nullable enable> — start с new files
3. Gradually enable warnings as errors per project
4. Refactor `null` returns → `T?` annotations
5. Add ArgumentNullException.ThrowIfNull в public APIs

Timeline: 6-12 months part-time для 500k.
```

### 16.3. Primary constructors strategy

Scenario: ASP.NET Core service classes — 50+ services с DI ctor.

```
Decision: refactor → primary constructors
Benefits: -6 lines boilerplate per class, ~300 lines saved
Cost: team learns syntax (1 hour talk)
Risk: minimal (compile checks all)

Order: 
1. Demo + team training (1 week)
2. Refactor 1-2 services (review)
3. Bulk apply remaining (1 day)
```

---

## 17. Что читать дальше

1. **csharplang GitHub** — github.com/dotnet/csharplang.
2. **Mads Torgersen blog/twitter** — language lead.
3. **Andrew Lock C# series** — andrewlock.net.
4. **Stephen Toub blogs** — performance.
5. **Книга — "C# in Depth" by Jon Skeet** — language deep dive.
6. **Книга — "Pro C#" by Andrew Troelsen** — comprehensive.

---

## 18. См. также

- [[modern-features|Modern Features]] — C# 8-13 syntax
- [[csharp-vs-other-langs|C# vs Other Languages]]
- [[generics-deep|Generics deep]]
- [[nullable-types|Nullable Types]]
- [[source-generators|Source Generators]]
- csharplang repository
- Microsoft DevBlogs

---

## 19. Reading list

- **csharplang** — github.com/dotnet/csharplang
- **csharplang/meetings** — LDM notes (weekly!)
- **Microsoft Docs — What's new in C#** — learn.microsoft.com/dotnet/csharp/whats-new/
- **Mads Torgersen Twitter** — @MadsTorgersen
- **Microsoft DevBlogs C#** — devblogs.microsoft.com/dotnet/category/csharp/
- **Roslyn repo** — github.com/dotnet/roslyn
- **Anders Hejlsberg interviews** — InfoQ, Channel 9
- **Bjarne Stroustrup** "Design and Evolution of C++" — parallel philosophy
- **"C# in Depth" by Jon Skeet** — 4th edition covers C# 7
- **"Pro C# 10 with .NET 6" by Troelsen** — comprehensive
- **C# Spec** — github.com/dotnet/csharpstandard (formal spec)
