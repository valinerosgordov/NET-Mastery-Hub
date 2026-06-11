---
tags: [csharp, comparison, senior, java, kotlin, typescript, go, rust, python, fsharp]
level: Senior
date: 2026-05-09
---

# C# vs Other Languages — детальная сравнения

> **Конкретные сравнения C# с Java, Kotlin, TypeScript, Go, Rust, Python, F#.** Где C# выигрывает, где проигрывает, где язык похож. Закрывает пробел: «знаю C#, рассматриваю проект на Kotlin/Go/Rust — где переход даст реальный benefit и где будут сюрпризы».

---

## 0. Как читать

Если выбираешь язык для проекта — раздел 1 (overall positioning) + конкретный соперник (раздел 2-7). Если migrating — раздел 9 (key idiom mappings). Decision matrix — раздел 10. Cross-cutting — раздел 11 (perf), 12 (productivity).

Cross-language якоря НЕ свёрнуты — это и есть тема файла. Interview-вопросы встроены.

---

## 1. C# positioning — что за язык

### 1.1. Языковые семьи

```
C# принадлежит к C-family + ECMAScript-family hybrid:
- C-syntax (curly braces, semicolons)
- Object-oriented (Java/C++)
- Functional features (F#/Haskell influences)
- Async-first (own innovation, JS adopted later)

Strict typed, GC-managed, JIT/AOT compiled, multi-paradigm.
```

### 1.2. Comparable languages

```
Most similar:
- Java (closest sibling, .NET vs JVM)
- Kotlin (cousin, JVM-based)
- TypeScript (similar in null safety, generics)

Different paradigms but compared:
- F# (functional .NET)
- Swift (similar features, Apple ecosystem)

Distant:
- Go (different philosophy — minimalism)
- Rust (no GC, borrow checker)
- Python (dynamic, scripting heritage)
```

### 1.3. Where C# fits

```
Sweet spot:
✅ Enterprise applications
✅ Web APIs / microservices (ASP.NET Core)
✅ Desktop (WPF, WinUI, MAUI)
✅ Cloud (Azure first-class)
✅ Game development (Unity)
✅ Big data / analytics

Less ideal:
⚠️ Systems programming (Rust лучше)
⚠️ Mobile (Kotlin/Swift native)
⚠️ Browser scripting (JS/TS dominant)
⚠️ Data science (Python ecosystem)
⚠️ Embedded (C/C++ дорогая)
```

### 1.4. Главное правило выбора

```
Choose C# когда:
  - .NET ecosystem fits (Azure, Microsoft stack)
  - Cross-platform service / API
  - Team has C# experience
  - Backward compat important
  - Performance + productivity balance

Choose competitor когда:
  - Rust: zero-cost abstractions, no GC, systems
  - Go: simplicity > features, Kubernetes ecosystem
  - Kotlin: Android, JVM ecosystem
  - TypeScript: browser, frontend ecosystem
  - Python: ML/data science, scripting
  - F#: pure functional, type-driven design
```

> [!question]- Интервью: какие основные сильные стороны C#?
> 1) **Performance** — reified generics + value types + JIT + Native AOT (.NET 8+) ≈ Java performance + closer to C++. 2) **Type safety** — strong static types, NRT (opt-in), pattern matching exhaustiveness. 3) **Productivity** — LINQ + async/await + records + primary constructors reduce boilerplate. 4) **Ecosystem** — NuGet 400k+ packages, Microsoft backing, Azure integration. 5) **Backward compat** — code C# 1.0 (2002) compiles в C# 13. 6) **Tooling** — Visual Studio + Rider top-tier IDE. 7) **Cross-platform** — Windows/Linux/Mac/iOS/Android (.NET 5+). Sweet spot: enterprise apps, web APIs, microservices.

---

## 2. C# vs Java

### 2.1. Overview

| Aspect | C# | Java |
|--------|-----|------|
| Year | 2002 | 1995 |
| Runtime | .NET (CLR) | JVM |
| Generics | **Reified** | Erased |
| Value types | **struct (real)** | All boxed primitives |
| Async | **async/await** native | CompletableFuture / virtual threads (21+) |
| Records | C# 9 (2020) | Java 14 (2020) |
| Pattern matching | C# 8+ comprehensive | Java 21 (2023) basic |
| Null safety | NRT opt-in (8+) | `Optional<T>` (8+) |
| Mobile | MAUI (cross-platform) | **Android first-class** |
| Cross-platform | .NET 5+ | JVM everywhere |

### 2.2. Generics — major difference

```csharp
// C# — reified, no boxing
List<int> list = new();
list.Add(42);   // direct stack-to-array
typeof(T)       // works в runtime
```

```java
// Java — erased, boxing
List<Integer> list = new ArrayList<>();
list.add(42);   // boxes int → Integer (heap allocation!)
// T.class — NOT available (erasure)
```

C# wins: 2-3x better performance для primitives, runtime type info, JIT specialization.

### 2.3. Value types

```csharp
// C# — true value semantics
public struct Point { public int X, Y; }
var p = new Point { X = 1, Y = 2 };   // stack allocated
var arr = new Point[1000];              // 1000 inline points в array
```

```java
// Java — only objects
public class Point { int x, y; }
Point p = new Point();   // heap allocated (Project Valhalla работает над value types — long-term)
Point[] arr = new Point[1000];   // array of references!
```

C# wins: predictable memory layout, no boxing для small data, cache-friendly arrays.

### 2.4. async/await

```csharp
// C# — async/await native
public async Task<string> FetchAsync(string url)
{
    var response = await client.GetAsync(url);
    return await response.Content.ReadAsStringAsync();
}
```

```java
// Java — CompletableFuture (verbose) or virtual threads (Java 21+)
public CompletableFuture<String> fetchAsync(String url) {
    return client.getAsync(url)
        .thenCompose(response -> response.bodyAsync());
}

// Java 21+ — virtual threads (Project Loom)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    var future = executor.submit(() -> fetchSync(url));
    return future.get();
}
```

C# async/await — much cleaner. Java 21 virtual threads — different model (preemptive lightweight).

### 2.5. Modern features evolution

```
C# 8 (2019) — NRT, async streams, pattern matching
C# 9 (2020) — Records, init, top-level
C# 10 (2021) — File-scoped, global usings, record struct
C# 11 (2022) — Raw strings, required, list patterns
C# 12 (2023) — Primary constructors all types, collection expr
C# 13 (2024) — field keyword, partial properties

Java 14 (2020) — Records (preview)
Java 16 (2021) — Records final
Java 17 (2021) — Sealed classes (LTS)
Java 21 (2023) — Pattern matching for switch, virtual threads (LTS)
```

C# adds features faster, more comprehensive. Java more conservative — focused on JVM compatibility.

### 2.6. Ecosystem

```
Java strengths:
✅ Older, larger ecosystem (Spring, Hibernate, Apache)
✅ Android (Java/Kotlin first-class)
✅ Big data (Hadoop, Spark, Kafka — Java)
✅ Multiple JVM languages (Scala, Kotlin, Clojure, Groovy)

C# strengths:
✅ Microsoft tooling (Visual Studio, Rider)
✅ Azure first-class (Microsoft owns both)
✅ Game development (Unity)
✅ Desktop on Windows (WPF, WinUI)
```

### 2.7. Migration tips

```
Java → C# easier than reverse:
- Most concepts map 1:1
- Generics actually behave better in C# (no erasure)
- async/await replaces Future composition
- LINQ → Streams (similar)
- Records similar

Pitfalls:
- Class loader → Assembly differences
- finalize → Dispose pattern
- @Override → override (mandatory not annotation)
- Annotations → Attributes (similar idea)
- Maven/Gradle → NuGet/MSBuild
```

> [!question]- Интервью: основные различия C# и Java в 2024?
> 1) **Generics**: C# reified vs Java erasure → C# 2-3x faster для primitives, runtime type info. 2) **Value types**: C# struct truly value (stack/inline) vs Java all boxed (Project Valhalla long-term). 3) **async/await**: C# native (since C# 5, 2012) vs Java CompletableFuture composition или virtual threads (21+ different model). 4) **Evolution speed**: C# annual major (8-13 added LOTS) vs Java more conservative. 5) **Cross-platform**: C# .NET 5+ everywhere; Java JVM everywhere. 6) **Mobile**: C# MAUI cross-platform; Java Android first-class. 7) **Ecosystem**: Java older + bigger (Spring); C# Microsoft tooling + Azure. **Verdict**: similar capabilities, C# more modern features, Java larger ecosystem.

---

## 3. C# vs Kotlin

### 3.1. Overview

```
Kotlin — JetBrains language. Targeted JVM (interop с Java), теперь multiplatform (JVM/JS/Native/Wasm).
Designed как "lingua franca" для JVM ecosystem — fix Java's pain points.
```

| Aspect | C# | Kotlin |
|--------|-----|--------|
| Year | 2002 | 2011 |
| Owner | Microsoft | JetBrains |
| Runtime | .NET / Native AOT | JVM / JS / Native (Multiplatform) |
| Null safety | **Opt-in NRT** | **Sound (default)** |
| Variance | `in`/`out` declaration-site | `in`/`out` declaration-site |
| Coroutines/async | async/await | Coroutines (different model) |
| Data classes | `record` (C# 9+) | `data class` (since 1.0) |
| Extension funs | Extension methods | Extension functions (more flexible) |
| Operator overload | Full | Restricted set |
| Companion | `static` class | `companion object` |

### 3.2. Null safety — Kotlin wins

```kotlin
// Kotlin — sound nullability
val a: String = "hello"      // non-null
val b: String? = null         // nullable
b.length    // ❌ Compile error
b?.length   // ✅ safe call (returns null)
b!!.length  // ✅ force unwrap (throws if null)
```

```csharp
// C# — opt-in NRT, holes
#nullable enable
string a = "hello";
string? b = null;
b.Length;   // ⚠️ warning, but compiles
b!.Length;  // suppress warning
// Generics, reflection — escapes NRT
```

Kotlin **truly sound**, C# best-effort. Trade-off: Kotlin designed from scratch (no legacy), C# preserved 22 years backward compat.

### 3.3. Coroutines vs async/await

```kotlin
// Kotlin coroutines — structured concurrency
suspend fun fetchData(): String {
    return withContext(Dispatchers.IO) {
        client.get("url").body()
    }
}

// Launch
runBlocking {
    val data = fetchData()
}

// Structured — coroutineScope { ... } cancels children on failure
```

```csharp
// C# async/await — Tasks
public async Task<string> FetchDataAsync()
{
    return await client.GetStringAsync("url");
}

// Caller
var data = await FetchDataAsync();
```

Both achieve similar goals. Kotlin coroutines — **structured concurrency** built-in (cancellation propagates). C# requires manual `CancellationToken` propagation. Kotlin slightly cleaner для complex async.

### 3.4. Extension functions

```kotlin
// Kotlin — extension functions can be methods, properties, operators
fun String.lastChar(): Char = this[this.length - 1]
val String.lastChar: Char get() = this[this.length - 1]   // property

operator fun String.times(n: Int): String = this.repeat(n)
"abc" * 3   // "abcabcabc" — operator overload через extension!
```

```csharp
// C# — extension methods (no extension properties or operators)
public static class StringExtensions
{
    public static char LastChar(this string s) => s[s.Length - 1];
    // Cannot make extension property
    // Cannot make extension operator
}
```

Kotlin more flexible — extension properties + extension operators. C# планирует "extension members" в C# 14+.

### 3.5. Data classes vs records

```kotlin
// Kotlin
data class User(val id: Int, val name: String)
// Auto: equals, hashCode, toString, copy, componentN

val u2 = u1.copy(name = "Bob")
```

```csharp
// C#
public record User(int Id, string Name);
// Auto: Equals, GetHashCode, ToString, with-expression, Deconstruct, ==

var u2 = u1 with { Name = "Bob" };
```

Almost identical concepts. Kotlin: `copy(name = ...)`. C#: `with { Name = ... }`. **Functionally equivalent**.

### 3.6. Operator overloading

```kotlin
// Kotlin — restricted set, contractual
operator fun Vector.plus(other: Vector): Vector = ...
operator fun Vector.times(scalar: Double): Vector = ...

// Allowed: plus/minus/times/div/rem/get/set/contains/iterator/...
// NOT allowed: any operator name (must be в predefined set)
```

```csharp
// C# — full operator set
public static Vector operator +(Vector a, Vector b) => ...;
public static Vector operator *(Vector v, double s) => ...;
public static implicit operator double(Money m) => m.Amount;
```

C# more flexible (any operator). Kotlin safer (restricted to obvious cases).

### 3.7. When choose Kotlin

```
✅ Android development — first-class
✅ Java interop — seamless (same JVM)
✅ JVM ecosystem (Spring, Kafka)
✅ Multiplatform (JVM/JS/Native/Wasm)
✅ Sound null safety important

❌ Microsoft ecosystem
❌ Cross-platform desktop (less mature)
❌ Game development (no Unity)
❌ Native AOT smaller binaries
```

> [!question]- Интервью: когда Kotlin лучше C#?
> 1) **Android** — first-class language, Google official. C# через MAUI — second-class. 2) **JVM ecosystem** — Spring, Hibernate, Kafka (Java/Kotlin both). 3) **Sound null safety** — Kotlin guarantees compile-time, C# NRT opt-in с holes. 4) **Multiplatform** — Kotlin Multiplatform (JVM/JS/iOS/Wasm). 5) **Coroutines** — structured concurrency cleaner для complex async. **C# wins**: 1) Performance (reified generics, value types, AOT). 2) Visual Studio tooling. 3) Azure ecosystem. 4) Backward compat. 5) Cross-platform desktop (WinUI, WPF). 6) Game development (Unity). Bottom line: Kotlin для Android/JVM/multiplatform, C# для everything else в Microsoft world.

---

## 4. C# vs TypeScript

### 4.1. Overview

```
TypeScript — JavaScript + types. Compile to JS (no runtime). Browser-native ecosystem.
C# influence in design — Anders Hejlsberg works on both!
```

| Aspect | C# | TypeScript |
|--------|-----|------------|
| Runtime | CLR/.NET | V8/Node/Browser |
| Target | Native code | JavaScript |
| Types | **Runtime preserved** | **Erased** (compile-only) |
| Generics | Reified | Structural |
| Records | C# 9 | (none) — interfaces close |
| Pattern matching | C# 8+ comprehensive | Limited |
| Async/await | Tasks | Promises |
| Nullable | NRT opt-in | `strictNullChecks` |
| Ecosystem | NuGet (.NET) | npm (JS world) |

### 4.2. Type system — fundamentally different

```typescript
// TypeScript — structural typing
interface Point { x: number; y: number; }
function distance(p: Point) { return Math.sqrt(p.x * p.x + p.y * p.y); }

// Anything с x and y as number works:
distance({ x: 3, y: 4 });   // OK
distance({ x: 3, y: 4, name: "extra" });   // OK — extra fields allowed
```

```csharp
// C# — nominal typing
public class Point { public double X, Y; }
public class Vector { public double X, Y; }   // structurally identical

double Distance(Point p) => Math.Sqrt(p.X * p.X + p.Y * p.Y);

Distance(new Vector { X = 3, Y = 4 });   // ❌ Compile error
```

TypeScript — **structural** (duck typing). C# — **nominal** (named types). Different worldviews:
- TS easier для quick prototyping, JSON shapes.
- C# safer для domain modeling, refactoring.

### 4.3. Erased vs reified types

```typescript
// TS — types erased в compile
function isString<T>(x: T): boolean {
    return typeof x === "string";   // runtime check на JS type
}

// typeof T at compile-time — not available!
```

```csharp
// C# — types preserved
bool IsString<T>(T x)
{
    return typeof(T) == typeof(string);   // runtime check!
    // или: return x is string;
}
```

C# generics support runtime introspection. TS generics — compile-time only.

### 4.4. Discriminated unions

```typescript
// TS — sum types FIRST-CLASS
type Shape = 
    | { kind: "circle"; radius: number }
    | { kind: "square"; side: number }
    | { kind: "rectangle"; width: number; height: number };

function area(s: Shape): number {
    switch (s.kind) {
        case "circle": return Math.PI * s.radius ** 2;
        case "square": return s.side ** 2;
        case "rectangle": return s.width * s.height;
        // exhaustiveness checked!
    }
}
```

```csharp
// C# — workaround через records + sealed
public abstract record Shape;
public record Circle(double Radius) : Shape;
public record Square(double Side) : Shape;
public record Rectangle(double Width, double Height) : Shape;

double Area(Shape s) => s switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Square sq => sq.Side * sq.Side,
    Rectangle r => r.Width * r.Height,
    _ => throw new InvalidOperationException()   // not exhaustive!
};
```

TS — discriminated unions native. C# — workaround через abstract record + pattern matching. Discriminated unions roadmapped C# 14+.

### 4.5. Async — Promises vs Tasks

```typescript
// TS — Promise
async function fetchData(): Promise<string> {
    const response = await fetch("url");
    return response.text();
}
```

```csharp
// C# — Task
public async Task<string> FetchDataAsync()
{
    var response = await client.GetAsync("url");
    return await response.Content.ReadAsStringAsync();
}
```

Almost identical syntax (TS borrowed from C#). Differences:
- Task — has `CancellationToken` integration.
- Promise — chained `.then()` / `.catch()` style supported.
- Both support `async`/`await`.

### 4.6. Strengths

```
TypeScript wins:
✅ Browser ecosystem (frontend)
✅ npm (largest package registry)
✅ Discriminated unions native
✅ Lighter weight (no runtime in browser)
✅ JSON-shape friendly

C# wins:
✅ Performance (compiled, GC tuned, AOT)
✅ Server-side maturity
✅ Reified generics
✅ Tooling (VS / Rider)
✅ Records, pattern matching mature
✅ Backward compat
```

### 4.7. When choose TypeScript

```
✅ Frontend (React, Angular, Vue)
✅ Node.js services
✅ JS interop required
✅ JSON-heavy APIs

❌ CPU-bound performance critical
❌ Desktop / mobile native UIs
❌ Game development
❌ Legacy enterprise migration
```

> [!question]- Интервью: чем TypeScript и C# различаются?
> Anders Hejlsberg designed obа languages. Similar в syntax, разница в фундамент: 1) **Runtime**: TS compiles to JS (no native runtime), C# native CLR/AOT. 2) **Types**: TS structural (duck typing), C# nominal (named types). 3) **Generics**: TS erased, C# reified — С# может typeof(T), TS не может. 4) **Discriminated unions**: TS native (`type X = A | B`), C# workaround через abstract record. 5) **Performance**: C# compiled native, TS runs as JS (JIT'd by V8). 6) **Async**: similar syntax, TS Promise/C# Task. 7) **Ecosystem**: TS frontend/Node.js (npm), C# server/desktop/games (NuGet). Verdict: TS — frontend King, C# — backend/desktop King.

---

## 5. C# vs Go

### 5.1. Overview

```
Go (Golang) — Google language. Designed для simplicity, fast compilation, concurrency.
Philosophy "less is more" — fewer features, fewer bugs, easier to read.
```

| Aspect | C# | Go |
|--------|-----|-----|
| Year | 2002 | 2009 |
| Owner | Microsoft | Google |
| Philosophy | Pragmatic features | **Minimalist** |
| Generics | Mature | **Late** (1.18, 2022) |
| OOP | Full (class/inheritance/interfaces) | Structural interfaces only |
| Async | async/await | **Goroutines + channels** |
| GC | Generational | Concurrent low-latency |
| Compile speed | ~Slow | **Very fast** |
| Binary size | Small (with AOT) | **Larger** (single binary) |
| Error handling | Exceptions | **Returns (error pattern)** |

### 5.2. Concurrency — fundamentally different

```go
// Go — goroutines + channels
func fetchData(url string, ch chan<- string) {
    response, _ := http.Get(url)
    body, _ := io.ReadAll(response.Body)
    ch <- string(body)
}

func main() {
    ch := make(chan string, 3)
    go fetchData("url1", ch)
    go fetchData("url2", ch)
    go fetchData("url3", ch)
    
    for i := 0; i < 3; i++ {
        fmt.Println(<-ch)
    }
}
```

```csharp
// C# — async/await + Task.WhenAll
public async Task<string[]> FetchAllAsync(string[] urls)
{
    var tasks = urls.Select(url => client.GetStringAsync(url));
    return await Task.WhenAll(tasks);
}
```

Go — **CSP** (Communicating Sequential Processes). Channels first-class. Goroutines preemptive lightweight.
C# — **continuation-based**. Tasks compose. Channels via `System.Threading.Channels`.

Both work, different mental models.

### 5.3. Generics — Go late game

```go
// Go 1.18+ (2022) — generics introduced
func Sum[T int | float64](values []T) T {
    var sum T
    for _, v := range values {
        sum += v
    }
    return sum
}
```

```csharp
// C# — generics from 2.0 (2005), generic math (.NET 7+)
public T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T sum = T.Zero;
    foreach (var v in values) sum += v;
    return sum;
}
```

Go generics — **17 years late** vs C#. More restrictive (constraints как unions of types). Less mature ecosystem.

### 5.4. Error handling

```go
// Go — explicit errors (no exceptions)
func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

result, err := divide(10, 2)
if err != nil {
    log.Fatal(err)
}
```

```csharp
// C# — exceptions
public int Divide(int a, int b)
{
    if (b == 0) throw new DivideByZeroException();
    return a / b;
}

try { var r = Divide(10, 0); }
catch (DivideByZeroException ex) { Console.WriteLine(ex); }
```

Go — explicit returns. Verbose, но **explicit control flow**. Cannot ignore.
C# — exceptions. Hidden control flow, но cleaner. Can ignore (silent swallow).

Both have proponents. Go style verbose; C# style hides flow.

### 5.5. Speed of development

```
Go advantages:
✅ Fast compilation (seconds for huge projects)
✅ Single binary deployment
✅ Cross-compilation built-in (one command)
✅ Less features → less to learn
✅ goroutines easier to grasp

C# advantages:
✅ Better IDE (intellisense, refactoring)
✅ Rich type system (records, pattern matching)
✅ LINQ для data manipulation
✅ More expressive (less verbose code)
✅ More ecosystem
```

### 5.6. Performance

```
Throughput similar (both compiled, both GC).
Startup: Go faster (no JIT warmup).
Memory: Go often smaller working set.
Native AOT (.NET 8+) closes gap dramatically.
```

### 5.7. When choose Go

```
✅ Cloud-native services (Kubernetes, Docker — Go)
✅ Network services (high concurrency simple)
✅ CLI tools (single binary, fast startup)
✅ Microservices (small footprint)
✅ Team values simplicity > features

❌ Complex business logic (Go verbose)
❌ Enterprise apps (less mature ecosystem)
❌ Desktop UIs (not designed для)
❌ Data processing (LINQ no equivalent)
```

> [!question]- Интервью: когда Go лучше C#?
> 1) **Cloud-native ecosystem** — Kubernetes, Docker, etcd, Terraform — все Go. Native fit. 2) **Simplicity preferred** — Go has fewer features, less to learn, easier code review. 3) **Fast compilation** — Go compiles в seconds, even huge projects. C# slower (especially MSBuild). 4) **Single binary deployment** — `go build` produces single executable. .NET requires runtime (until Native AOT). 5) **Team philosophy** — Google-style "less is more". **C# wins**: 1) Performance similar but Native AOT closes gap. 2) Richer type system (generics earlier, more mature). 3) Better IDE tooling. 4) LINQ for data. 5) Async/await cleaner для complex flows. 6) More expressive code. Bottom line: Go для cloud-native infrastructure / simple services, C# для complex business logic / enterprise.

---

## 6. C# vs Rust

### 6.1. Overview

```
Rust — Mozilla origins, now Rust Foundation. Memory safety без GC через ownership + borrow checker.
"Fast, reliable, productive — pick three" — но steep learning curve.
```

| Aspect | C# | Rust |
|--------|-----|------|
| Year | 2002 | 2010 |
| Memory | **GC managed** | **Ownership + borrow checker** |
| Performance | JIT/AOT | **Zero-cost abstractions** |
| Safety | Type-safe + GC | **Memory + thread safe by compiler** |
| Generics | Reified (JIT) | **Monomorphized** (compile-time) |
| Async | Tasks | Future + executor (no runtime built-in) |
| Error handling | Exceptions | **`Result<T, E>`** |
| Null | NRT opt-in | **`Option<T>`** |
| Compile time | Slow | **Slower** |
| Learning curve | Moderate | **Steep** |

### 6.2. Memory safety — Rust unique

```rust
// Rust — ownership + borrow checker
fn main() {
    let s = String::from("hello");
    let s2 = s;  // s moved, no longer valid
    // println!("{}", s);   // ❌ compile error!
    println!("{}", s2);
}

fn process(s: &str) {  // borrow (immutable reference)
    println!("{}", s);
}

let s = String::from("hello");
process(&s);  // OK
println!("{}", s);  // OK — immutable borrow released
```

```csharp
// C# — GC handles automatically
string s = "hello";
string s2 = s;   // both reference same (string immutable so safe)
Console.WriteLine(s);   // OK
```

Rust **prevents** memory bugs at compile time (use after free, double free, data races). C# uses GC (runtime overhead but easier).

### 6.3. Performance

```
Rust: zero-cost abstractions, monomorphization, no GC pauses.
C#: JIT good, Native AOT closes gap, GC has low-latency mode.

Benchmarks:
- Simple math/loops: Rust ≈ C# AOT (1-1.1x)
- Memory-intensive: Rust 1.5-2x faster (no GC)
- Async I/O: similar (tokio vs Tasks)
- Web servers: Rust (axum/actix) ≈ C# (Kestrel)
```

Rust noticeably faster в memory-heavy workloads (no GC pauses). C# Native AOT closes general gap.

### 6.4. Learning curve

```rust
// Rust — explicit lifetimes (advanced)
struct Parser<'a> {
    input: &'a str,
    position: usize,
}

impl<'a> Parser<'a> {
    fn new(input: &'a str) -> Self {
        Parser { input, position: 0 }
    }
    
    fn peek(&self) -> Option<char> {
        self.input[self.position..].chars().next()
    }
}
```

```csharp
// C# — no lifetimes
public class Parser
{
    private readonly string _input;
    private int _position;
    
    public Parser(string input) => _input = input;
    
    public char? Peek() => _position < _input.Length ? _input[_position] : null;
}
```

Rust requires understanding lifetimes — concept не существует в C#. **Significant learning curve**.

### 6.5. Error handling

```rust
// Rust — Result enum, no exceptions
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        Err(String::from("division by zero"))
    } else {
        Ok(a / b)
    }
}

let result = divide(10, 2)?;   // ? operator propagates Err
```

```csharp
// C# — exceptions or Result pattern
public Result<int> Divide(int a, int b) =>
    b == 0 ? new Result<int>.Failure("div by zero") : new Result<int>.Success(a / b);
```

Rust forces error handling at compile time (exhaustiveness). C# Result optional (recommended).

### 6.6. Async — Rust complex

```rust
// Rust — Future trait, requires executor
use tokio;

#[tokio::main]
async fn main() {
    let response = reqwest::get("url").await.unwrap();
    let body = response.text().await.unwrap();
    println!("{}", body);
}
```

```csharp
// C# — built-in
public async Task Main()
{
    var response = await client.GetStringAsync("url");
    Console.WriteLine(response);
}
```

Rust requires **external executor** (tokio, async-std). C# has built-in. Rust async — more complex (`Pin<T>`, lifetimes interact).

### 6.7. When choose Rust

```
✅ Systems programming (OS, drivers)
✅ Embedded (no GC, small footprint)
✅ Performance-critical (databases, compilers, browsers)
✅ Cryptography (memory safety crucial)
✅ WebAssembly (no GC alternative)
✅ Low-latency required (game engines, real-time)

❌ Rapid prototyping
❌ Business logic (verbose)
❌ Team without Rust experience (3-6 months ramp)
❌ Web APIs (C#/Go simpler)
❌ Enterprise apps (slower development)
```

> [!question]- Интервью: когда Rust лучше C#?
> 1) **Systems programming** — OS, drivers, embedded — Rust no GC, fine memory control. 2) **Performance critical** — databases (TiKV), compilers (rustc), browsers (Servo) — no GC pauses, predictable perf. 3) **Memory safety crucial** — cryptography, security tools. 4) **WebAssembly** — Rust top-tier compiler, C# Blazor heavier. 5) **Resource-constrained** — small binaries, low memory (embedded). **C# wins**: 1) Productivity — Rust learning curve 3-6 months. 2) Easier business logic. 3) Mature ecosystem (Azure, ASP.NET). 4) Tooling (VS top-tier). 5) Async simpler (built-in). 6) GC handles complex object graphs. Bottom line: Rust для systems / performance-critical, C# для applications / business logic.

---

## 7. C# vs Python

### 7.1. Overview

```
Python — dynamically typed, scripting heritage, "batteries included".
ML/data science dominant, scripting popular, web app via Django/Flask.
```

| Aspect | C# | Python |
|--------|-----|--------|
| Typing | **Static** | **Dynamic** (typing hints opt-in) |
| Performance | **Compiled, fast** | **Interpreted, slow** |
| GIL | No (true parallelism) | **Has GIL** (no true threading) |
| Async | async/await | asyncio (similar) |
| Ecosystem | NuGet (server) | PyPI (everything) |
| ML/Data | ML.NET (limited) | **Pandas, NumPy, PyTorch, sklearn** |
| Scripting | dotnet-script | **Native** |
| Beginner | Moderate | **Easy** |

### 7.2. Speed difference — huge

```
Naive benchmarks:
- Loops: Python 10-100x slower than C#
- Math: Python 50-100x slower (without NumPy)
- I/O bound: similar (both async-capable)

С NumPy/Cython — Python catches up для specific workloads (vectorized operations).
```

### 7.3. Type system

```python
# Python — duck typing, no compile-time check
def process(item):
    return item.value * 2   # works if item has .value

# Type hints (3.5+) — IDE help, no runtime
def process(item: Item) -> int:
    return item.value * 2
```

```csharp
// C# — static, compile-time check
int Process(Item item)
{
    return item.Value * 2;   // checked compile-time
}
```

Python flexibility = bugs hidden until runtime. C# strictness = caught early.

### 7.4. ML / data ecosystem

```python
# Python — first-class ML
import torch
import numpy as np
import pandas as pd

df = pd.read_csv("data.csv")
features = df[['x', 'y']].values
model = torch.nn.Linear(2, 1)
```

```csharp
// C# — limited ML
using Microsoft.ML;
var mlContext = new MLContext();
var data = mlContext.Data.LoadFromTextFile<HousingData>("housing.csv");
// ML.NET — much smaller ecosystem чем PyTorch/sklearn
```

Python wins ML hands down. C# improves (ML.NET, TorchSharp), but Python ecosystem dominant.

### 7.5. Scripting

```python
# Python — write file, run
# script.py
import requests
response = requests.get("https://api.example.com")
print(response.json())

# python script.py
```

```csharp
// C# — top-level statements (C# 9+) close gap
// script.csx
using System.Net.Http;

var client = new HttpClient();
var response = await client.GetStringAsync("https://api.example.com");
Console.WriteLine(response);

// dotnet script script.csx
// Or Program.cs as top-level
```

Python — natural scripting. C# possible but ceremony (project file, sometimes).

### 7.6. When choose Python

```
✅ Machine learning, data science
✅ Scientific computing
✅ Scripting / automation
✅ Web scraping
✅ Quick prototypes
✅ Beginner-friendly

❌ High-performance servers
❌ Native mobile / desktop
❌ Game engines
❌ Type-safe enterprise apps
```

> [!question]- Интервью: когда Python лучше C#?
> 1) **Machine learning / data science** — Python ecosystem (PyTorch, TensorFlow, NumPy, Pandas, scikit-learn) dominant. C# (ML.NET, TorchSharp) — much smaller. 2) **Scripting / automation** — Python natural, C# requires project setup (scripts via dotnet-script possible но less mature). 3) **Scientific computing** — NumPy/SciPy leverage decades of Fortran libs. 4) **Quick prototypes** — Python concise, no compile cycle. 5) **Educational** — easier для beginners. **C# wins**: 1) Performance 10-100x faster. 2) Static typing — early bug detection. 3) True parallelism (no GIL). 4) Enterprise apps (better refactoring, IDE). 5) Native UI (WPF, MAUI). 6) Game development (Unity). Bottom line: Python для ML/data/scripting, C# для performance/enterprise/native.

---

## 8. C# vs F#

### 8.1. Overview

```
F# — functional-first .NET language. Designed by Don Syme. Same runtime as C# (CLR), interop seamless.
"Pure functional когда возможно, OO когда practical".
```

| Aspect | C# | F# |
|--------|-----|-----|
| Paradigm | OO + functional | **Functional first**, OO available |
| Type inference | Limited (var, target-typed new) | **Hindley-Milner** (full) |
| Discriminated unions | No (planned) | **Yes** |
| Records | Yes (C# 9+) | **Yes (1.0)** |
| Immutability | Opt-in | **Default** |
| Pattern matching | Yes (8+) | **Yes**, more comprehensive |
| Computation expressions | LINQ + async | **Generic** (any monad-like) |
| Type providers | (none) | **Yes** (DB schemas, JSON, etc.) |
| Verbosity | Moderate | **Concise** |

### 8.2. Type inference difference

```csharp
// C# — limited inference
public int Add(int a, int b) => a + b;
var sum = Add(2, 3);
```

```fsharp
// F# — Hindley-Milner full inference
let add a b = a + b   // types inferred from usage
let sum = add 2 3      // int inferred
```

F# infers types throughout — almost no annotations. C# requires explicit types для parameters / fields.

### 8.3. Discriminated unions

```fsharp
// F# — first-class
type Shape =
    | Circle of radius: float
    | Square of side: float
    | Rectangle of width: float * height: float

let area shape =
    match shape with
    | Circle r -> System.Math.PI * r * r
    | Square s -> s * s
    | Rectangle (w, h) -> w * h
// Compiler enforces exhaustiveness
```

```csharp
// C# — workaround
public abstract record Shape;
public record Circle(double Radius) : Shape;
public record Square(double Side) : Shape;
public record Rectangle(double Width, double Height) : Shape;

double Area(Shape s) => s switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Square sq => sq.Side * sq.Side,
    Rectangle r => r.Width * r.Height,
    _ => throw new InvalidOperationException()
};
```

F# native union types — cleaner, exhaustive. C# workaround works но verbose, exhaustiveness не enforced.

### 8.4. Immutability

```fsharp
// F# — immutable by default
let x = 5
// x <- 10   // error — mutable required для assignment
let mutable y = 5
y <- 10   // OK с mutable

let user = { Id = 1; Name = "Alice" }
let user2 = { user with Name = "Bob" }   // copy
```

```csharp
// C# — mutable by default
int x = 5;
x = 10;   // OK

var user = new User { Id = 1, Name = "Alice" };
user.Name = "Bob";   // OK (если property is set)

// init-only / records — opt-in immutability
public record User(int Id, string Name);
var user2 = user with { Name = "Bob" };
```

F# enforces immutability discipline. C# allows but doesn't enforce.

### 8.5. Computation expressions

```fsharp
// F# — generic computation expressions
async {
    let! data = fetchAsync url
    let processed = process data
    return processed
}

// Custom builder для DB transactions, optionals, lists
result {
    let! a = parseInt "10"
    let! b = parseInt "20"
    return a + b
}
```

```csharp
// C# — async/await + LINQ for specific cases
async Task<string> Fetch(string url)
{
    var data = await FetchAsync(url);
    return Process(data);
}
```

F# computation expressions generalize async/LINQ — apply to any monad-like type. C# limited to specific keywords.

### 8.6. Interop с C#

```fsharp
// F# can use C# libraries seamlessly
open System.Collections.Generic

let dict = Dictionary<string, int>()
dict.Add("hello", 5)
```

Same CLR — F# uses NuGet packages, ASP.NET Core, EF Core. F# не isolated — integrates.

### 8.7. When choose F#

```
✅ Domain-driven design (rich types)
✅ Financial / actuarial calculations
✅ Data transformations / pipelines
✅ Machine learning prototyping (.NET)
✅ Compiler / parser projects
✅ Concurrent / immutable systems

❌ UI development (less popular)
❌ Game development (no Unity)
❌ Team unfamiliar with FP
❌ Mobile / cross-platform apps
```

> [!question]- Интервью: чем F# лучше C# для domain modeling?
> 1) **Discriminated unions native** — `type Shape = Circle | Square | Rectangle`. C# workaround через abstract record + sealed. F# compiler enforces exhaustiveness. 2) **Type inference Hindley-Milner** — almost no annotations needed. C# limited (var, target-typed). 3) **Immutability default** — F# values immutable, mutation requires `mutable` keyword. C# mutable default, immutability opt-in. 4) **Records mature** — F# 1.0 (2005), C# only 2020+. 5) **Computation expressions** — generic monadic syntax. C# only specific keywords (async, LINQ). 6) **Pattern matching deeper** — guards, active patterns. **C# wins**: bigger ecosystem, more popular, easier hiring, better tooling. F# specialized для FP-heavy domains.

---

## 9. Key idiom mappings (cheat sheet)

### 9.1. Iteration

```csharp
// C#
var doubled = numbers.Select(x => x * 2).Where(x => x > 10).ToList();
```

```java
// Java
List<Integer> doubled = numbers.stream()
    .map(x -> x * 2)
    .filter(x -> x > 10)
    .collect(Collectors.toList());
```

```kotlin
// Kotlin
val doubled = numbers.map { it * 2 }.filter { it > 10 }
```

```typescript
// TypeScript
const doubled = numbers.map(x => x * 2).filter(x => x > 10);
```

```python
# Python
doubled = [x * 2 for x in numbers if x * 2 > 10]
# or: list(filter(lambda x: x > 10, map(lambda x: x*2, numbers)))
```

```rust
// Rust
let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).filter(|&x| x > 10).collect();
```

```go
// Go — manual loop, no built-in stream
var doubled []int
for _, x := range numbers {
    d := x * 2
    if d > 10 {
        doubled = append(doubled, d)
    }
}
```

```fsharp
// F#
let doubled = numbers |> List.map ((*) 2) |> List.filter (fun x -> x > 10)
```

### 9.2. Null safety

```csharp
// C# — NRT
string? name = GetName();
int len = name?.Length ?? 0;
```

```kotlin
// Kotlin — sound
val name: String? = getName()
val len = name?.length ?: 0
```

```typescript
// TypeScript
const name: string | null = getName();
const len = name?.length ?? 0;
```

```rust
// Rust — Option<T>
let name: Option<String> = get_name();
let len = name.as_ref().map(|s| s.len()).unwrap_or(0);
```

### 9.3. Async

```csharp
// C#
var data = await client.GetStringAsync(url);
```

```kotlin
// Kotlin — coroutines
val data = withContext(Dispatchers.IO) {
    client.get(url).bodyAsText()
}
```

```typescript
// TS — Promise
const data = await fetch(url).then(r => r.text());
```

```rust
// Rust — Future
let data = reqwest::get(url).await?.text().await?;
```

### 9.4. Records / data classes

```csharp
// C#
public record User(int Id, string Name);
```

```kotlin
// Kotlin
data class User(val id: Int, val name: String)
```

```fsharp
// F#
type User = { Id: int; Name: string }
```

```rust
// Rust — struct + derive
#[derive(Clone, Debug, PartialEq)]
struct User {
    id: i32,
    name: String,
}
```

```typescript
// TS — interface or class
interface User { id: number; name: string; }
```

---

## 10. Decision matrix

```
Project type                 | Best | Good | OK
-----------------------------+------+------+----
Web API / microservice       | C# / Go / Kotlin | TS / Rust | Java / Python
Enterprise app               | C# / Java / Kotlin | TS | Go / F#
Desktop UI                   | C# (WPF) / Swift / Kotlin | TS (Electron) | Rust
Mobile native                | Kotlin / Swift | C# (MAUI) | TS (RN)
Game development             | C# (Unity) / C++ | Rust | F#
Cloud-native infrastructure  | Go / Rust | C# | Java
ML / data science            | Python | F# / R / Julia | C#
Systems / OS                 | Rust / C | C++ | Zig
Scripting                    | Python / TS | C# (script) | Go
Frontend (browser)           | TS / JS | C# (Blazor) | -
Compiler / parser            | Rust / F# / OCaml | C# / Haskell | -
Financial / actuarial        | F# / C# / Scala | Java | Python
Real-time / low-latency      | Rust / C++ | C# (AOT) | Go
Embedded                     | Rust / C / C++ | - | -
WebAssembly                  | Rust | C# (Blazor) | Go
```

---

## 11. Performance comparison

### 11.1. Compute-heavy benchmarks (relative to C++ baseline)

```
| Language     | Relative Speed |
|--------------|----------------|
| C / C++      | 1.0x (baseline) |
| Rust         | 1.0-1.1x |
| C# Native AOT| 1.05-1.15x |
| C# JIT       | 1.1-1.3x |
| Java         | 1.2-1.4x |
| Go           | 1.2-1.5x |
| F# (.NET)    | 1.1-1.3x (= C#) |
| TypeScript/V8| 2-5x |
| Python (CPython) | 50-100x |
| Python + NumPy | 1.5-3x (vectorized) |
```

### 11.2. Memory usage

```
Native AOT (C#) ≈ Rust ≈ Go (KB-MB)
JVM (Java) > .NET JIT (C#) > Native AOT
Python > Node.js (TS) > .NET
```

### 11.3. Startup time

```
Best:  Native AOT (C#) ≈ Rust ≈ Go (ms)
Good:  Node.js (TS), CPython (Python)
Slow:  JVM (Java), .NET JIT (C#) — JIT warmup
```

### 11.4. Throughput (web servers)

```
ASP.NET Core (Kestrel) ≈ Tokio (Rust) > Go (net/http) > Spring (Java) > Express (TS) > Flask (Python)
(в TechEmpower benchmarks)
```

---

## 12. Productivity comparison

### 12.1. Lines of code (для same task)

```
Python:    1.0x (baseline — concise)
Kotlin:    1.0x
F#:        0.9x (sometimes shorter)
TS:        1.2x
C#:        1.3x
Java:      1.7x
Go:        1.5x (verbose error handling)
Rust:      1.4x (lifetimes)
```

### 12.2. Time to working code

```
Easy:    Python, TypeScript, C#, Kotlin
Medium:  Java, Go, F#
Hard:    Rust (3-6 months learning curve)
```

### 12.3. Hiring difficulty

```
Easy:    JavaScript/TypeScript, Python, Java
Medium:  C#, Kotlin, Go
Hard:    Rust, F#, Scala, Haskell
```

---

## 13. Summary table

| Language | Strengths | Weaknesses | Best for |
|----------|-----------|------------|----------|
| **C#** | Performance, ecosystem, tooling, productivity | Mobile second-class | Enterprise, web, games |
| **Java** | Mature ecosystem, Android | Slower evolution, no value types | Enterprise, Android |
| **Kotlin** | Sound nullability, Android, multiplatform | Smaller ecosystem | Android, JVM, multiplatform |
| **TypeScript** | Browser native, npm | Slower runtime | Frontend, Node |
| **Go** | Simplicity, fast compile, cloud-native | Less features, late generics | Infrastructure, microservices |
| **Rust** | Memory safety, performance, no GC | Steep learning curve | Systems, performance |
| **Python** | ML ecosystem, beginner-friendly | Slow, GIL | ML, data, scripting |
| **F#** | Discriminated unions, type inference, immutability | Less popular | DDD, financial, parsers |

---

## 14. Common pitfalls in language migration

### 14.1. C# → Java

```
Pitfall: Generics work differently — primitives boxed.
Pitfall: No struct — все на heap.
Pitfall: No async/await — CompletableFuture verbose.
Pitfall: NullPointerException — Java has no NRT (use Optional<T>).
```

### 14.2. C# → Go

```
Pitfall: No exceptions — error returns везде.
Pitfall: No classes (only structs + interfaces).
Pitfall: No generics until 1.18.
Pitfall: No LINQ — manual loops.
Pitfall: GOPATH / modules confusion (resolved 1.16+).
```

### 14.3. C# → Rust

```
Pitfall: Ownership/borrow checker — fights compiler 3+ months.
Pitfall: No GC — explicit lifetimes everywhere.
Pitfall: No exceptions — Result<T, E> everywhere.
Pitfall: No null — Option<T> everywhere.
Pitfall: Async needs runtime (tokio).
Pitfall: Compile time slow.
```

### 14.4. C# → Kotlin

```
Pitfall: JVM ecosystem (Maven/Gradle, не NuGet).
Pitfall: Coroutines mental model different.
Pitfall: Sound nullability — strict.
Pitfall: Companion object вместо static.
Pitfall: Operator overloading restricted set.
```

### 14.5. C# → Python

```
Pitfall: Dynamic typing — runtime errors hidden.
Pitfall: GIL — true threading impossible.
Pitfall: Performance 10-100x slower (без NumPy).
Pitfall: Indentation matters (whitespace-sensitive).
Pitfall: Different async model (asyncio).
```

### 14.6. C# → TypeScript

```
Pitfall: Structural typing — types match by shape.
Pitfall: Erased types — no runtime introspection.
Pitfall: Promise vs Task — minor differences.
Pitfall: npm hell — dependency tree huge.
Pitfall: Browser ecosystem — no file I/O в browser.
```

> [!question]- Интервью: какие top mistakes мигрируют команды между языками?
> 1) **C# → Go**: expect classes/generics/LINQ — Go has different model (structs + interfaces, late generics, manual loops). 2) **C# → Rust**: ownership/borrow checker fight для 3+ months. Lifetimes — concept не существует в C#. 3) **C# → Java**: assume generics same — Java erasure boxing kills performance. 4) **C# → Kotlin**: ecosystem switch (NuGet → Maven), coroutines mental model. 5) **C# → Python**: dynamic typing surprise, GIL no parallelism, performance shock. **Universal**: don't migrate language without 2-3 month team training, prototype first, measure performance against actual goals.

---

## 15. Practice — language choice exercises

### 15.1. CLI tool для file processing

```
Options:
- Go: ✅ single binary, fast startup, simple
- Rust: ✅ fastest, but learning curve
- C# Native AOT: ✅ similar to Go, .NET ecosystem
- Python: ⚠️ requires runtime, slower

Recommendation: Go или C# AOT, depending on team.
```

### 15.2. ML platform для analytics

```
Options:
- Python: ✅ ecosystem dominant, prototyping fast
- C# (ML.NET, TorchSharp): ⚠️ smaller ecosystem
- F#: ⚠️ functional fits, но small community

Recommendation: Python для ML core, expose via API. C# для backend.
```

### 15.3. Microservices с heavy concurrency

```
Options:
- C#: ✅ async/await, ASP.NET Core mature
- Go: ✅ goroutines built-in, deployment simple
- Rust: ✅ fastest, но complex
- Java: ✅ virtual threads (21+), Spring

Recommendation: C# или Go обычно. Rust для extreme perf.
```

### 15.4. Desktop application

```
Options:
- C# (WPF/WinUI/MAUI): ✅ Microsoft stack, rich UI
- Kotlin (Jetpack Compose Desktop): ⚠️ multiplatform но young
- Swift (Mac only): ✅ native, restricted platform
- Electron (TS): ⚠️ resource-heavy

Recommendation: C# для Windows desktop, multiplatform — Avalonia (C#).
```

---

## 16. Cheat sheet — quick comparisons

```
For business logic        → C# > Kotlin > Java > Python
For performance           → Rust > C# AOT > Go > Java > C#
For ML/data               → Python > F# > R > C#
For mobile native         → Kotlin (Android) > Swift (iOS) > C# (MAUI)
For systems               → Rust > C/C++ > Zig
For scripting             → Python > TypeScript > C# (top-level)
For browser               → TypeScript > JavaScript > C# (Blazor)
For game development      → C# (Unity) > C++ (Unreal) > Rust (rising)
For infrastructure        → Go > Rust > C#

Performance hierarchy:
  C/C++/Rust  >  C# AOT  >  C# JIT  >  Go  >  Java  >  Node  >  Python

Productivity hierarchy:
  Python ≈ Kotlin > C# ≈ TS > Java > Go > Rust

Type safety hierarchy:
  Rust > F# > Kotlin > C# > TS > Java > Python
```

---

## 17. Что читать дальше

1. **[[csharp-language-design|C# Language Design]]** — почему C# такой.
2. **[[modern-features|Modern Features]]** — C# 8-13 features.
3. **The Rust Programming Language** (book) — официальный.
4. **Effective Java** by Joshua Bloch — Java patterns.
5. **Programming Kotlin** — JetBrains book.
6. **Programming Rust** (O'Reilly) — depth.

---

## 18. См. также

- [[csharp-language-design|C# Language Design]]
- [[modern-features|Modern Features]]
- [[generics-deep|Generics Deep]] — C# vs Java
- [[async-threading|Async Threading]] — async comparison
- [[functional-csharp|Functional C#]] — F# influences

---

## 19. Reading list

- **Microsoft Docs — C# vs Java** — learn.microsoft.com
- **Kotlin Language Guide** — kotlinlang.org
- **Rust Book** — doc.rust-lang.org/book
- **Go Programming** — go.dev/doc
- **TypeScript Handbook** — typescriptlang.org/docs/handbook
- **TechEmpower Web Framework Benchmarks** — techempower.com
- **Computer Language Benchmarks Game** — benchmarksgame-team.pages.debian.net
- **Anders Hejlsberg interviews** — works on C#, TypeScript
- **Don Syme interviews** — F# creator
- **"7 Languages in 7 Weeks" by Bruce Tate** — comparison book
