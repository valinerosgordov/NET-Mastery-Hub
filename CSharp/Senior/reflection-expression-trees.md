---
tags: [csharp, reflection, expression-trees, senior, runtime-types, codegen, linq-providers]
level: Senior
date: 2026-05-09
---

# Reflection и Expression Trees — runtime типы и компиляция

> **`System.Reflection` — runtime introspection, dynamic invocation. Expression trees — lambda как data, foundation для LINQ providers (EF Core, Dapper).** Когда нужно (frameworks, mapping), когда избегать (hot path, AOT). Закрывает пробел: «знаю про typeof, не понимаю как написать generic mapper и почему `Expression<Func<T, bool>>` compiles в SQL».

---

## 0. Как читать

Если впервые — раздел 1→3 (Reflection basics + performance). Expression trees — раздел 5→7. AOT considerations — раздел 9. Production guidance — раздел 11 (best practices), 13 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Reflection vs Expression Trees

```
Reflection — runtime introspection:
- Read metadata (types, methods, attributes)
- Invoke methods dynamically
- Create instances dynamically
- Slow (~100-1000x slower than direct call)

Expression Trees — code as data:
- Lambda lambda represented as object tree
- Can be analyzed (LINQ to SQL, EF Core)
- Can be compiled to delegate
- Foundation для query providers
```

### 1.2. Когда reflection

```
✅ Frameworks (DI containers, ORMs, serializers)
✅ Plugin systems (load assembly, find types)
✅ Tooling (debuggers, profilers, IDE)
✅ Configuration mapping (object → config)
✅ Test runners (find [Test] methods)

❌ Hot path business logic (slow)
❌ AOT applications (limited)
❌ Frequent calls (cache results!)
❌ Type-safe alternatives existing (use them)
```

### 1.3. Когда expression trees

```
✅ LINQ providers (translate to SQL/etc.)
✅ Dynamic queries (build at runtime)
✅ Optimized property access (faster than reflection)
✅ Validation rules (type-safe)
✅ Mapping libraries (AutoMapper, Mapster)

❌ Simple compile-time code (just write it)
❌ AOT (limited support)
❌ One-off scripts
```

### 1.4. Главное правило

```
Reflection дешевле когда cached:
  - GetType, GetMethod, GetProperty — slow
  - Invoke — slow
  - Cache MethodInfo / PropertyInfo / Func<T> в static field
  - Compiled expression tree → ~direct call performance

Expression tree compile cost ~100-1000x slower than running compiled.
Cache compiled delegates. Pay once, reuse forever.

Source generators (C# 9+) replace reflection в most cases:
  - JsonSerializerContext (vs reflection-based JsonSerializer)
  - LibraryImport (vs DllImport reflection)
  - LoggerMessage (vs reflection logging)
```

### 1.5. Эволюция

| Версия | Что |
|--------|-----|
| **.NET 1.0** | `System.Reflection` базовый |
| **.NET 1.0** | `Reflection.Emit` — runtime IL generation |
| **.NET 3.5** | Expression trees (LINQ foundation) |
| **.NET 4.0** | `dynamic` keyword, DLR |
| **.NET 4.5** | `Expression<Func<T>>` improvements |
| **.NET 6+** | Source generators replace many reflection cases |
| **.NET 8+** | Native AOT — limited reflection (analyzer warnings) |
| **.NET 9** | More attributes для AOT-friendly reflection |

> [!info]- Если ты знаешь Java / Kotlin / Python / Rust
> **Java:** `java.lang.reflect` similar API. Method handles (Java 7+) faster alternative. Bytecode manipulation (ASM, javassist) — analog Reflection.Emit.
>
> **Kotlin:** `kotlin.reflect` поверх Java reflection. Inline reified types — compile-time где возможно.
>
> **Python:** dynamic by nature — getattr/setattr/__class__ everywhere. No "reflection" concept — это default behavior.
>
> **Rust:** **нет** runtime reflection. `std::any::TypeId` для basic type checks. Macros (`derive`) для compile-time generation. Same goals via source generators в C#.

> [!question]- Интервью: чем reflection отличается от expression trees?
> **Reflection** (`System.Reflection`) — runtime introspection: read metadata (types, methods, attributes), invoke methods dynamically. **Slow** ~100-1000x чем direct call. **Expression Trees** (`System.Linq.Expressions`) — lambdas as **data structures** (objects describing the code). Can be: 1) **Analyzed** (LINQ to SQL translates to SQL). 2) **Compiled** to delegate (~direct call performance after compile). 3) **Built dynamically** (composable queries). **Use cases**: reflection — frameworks (DI, ORM, serializers); expression trees — LINQ providers, dynamic queries, fast property access. **Both expensive cold**: cache results. Source generators (C# 9+) replace many reflection cases compile-time.

---

## 2. Reflection basics

### 2.1. Type and TypeOf

```csharp
// Get Type object
Type t1 = typeof(string);   // compile-time
Type t2 = "hello".GetType(); // runtime
Type t3 = Type.GetType("System.String"); // by name

// Inspect
Console.WriteLine(t1.FullName);    // System.String
Console.WriteLine(t1.IsClass);      // true
Console.WriteLine(t1.IsValueType);  // false
Console.WriteLine(t1.IsAbstract);   // false
Console.WriteLine(t1.BaseType);     // System.Object
Console.WriteLine(t1.Assembly.FullName);
```

### 2.2. Members — methods, properties, fields

```csharp
Type t = typeof(User);

// Methods
MethodInfo[] methods = t.GetMethods();
foreach (var m in methods)
    Console.WriteLine($"{m.Name}({string.Join(", ", m.GetParameters().Select(p => p.ParameterType.Name))})");

// Specific method
MethodInfo? saveMethod = t.GetMethod("Save", new[] { typeof(int) });

// Properties
PropertyInfo[] props = t.GetProperties();
foreach (var p in props)
    Console.WriteLine($"{p.PropertyType.Name} {p.Name}");

// Fields
FieldInfo[] fields = t.GetFields(BindingFlags.NonPublic | BindingFlags.Instance);
```

### 2.3. BindingFlags

```csharp
// Default: public + static OR public + instance
typeof(User).GetMethods();   // public only

// Include private
typeof(User).GetMethods(BindingFlags.Public | BindingFlags.NonPublic | BindingFlags.Instance);

// Static only
typeof(User).GetMethods(BindingFlags.Static | BindingFlags.Public);
```

`BindingFlags` — what to include в search. Public/NonPublic, Static/Instance, DeclaredOnly (no inherited).

### 2.4. Invoke method

```csharp
var instance = new User { Name = "Alice" };
Type t = instance.GetType();

MethodInfo? greetMethod = t.GetMethod("Greet");
object? result = greetMethod!.Invoke(instance, null);   // null = no args
Console.WriteLine(result);   // "Hello, Alice!"

// С arguments
MethodInfo? addMethod = t.GetMethod("AddAge");
addMethod!.Invoke(instance, new object[] { 5 });
```

`Invoke(target, args)` — call method via reflection. **Slow** (~1000x slower than direct).

### 2.5. Property access

```csharp
PropertyInfo? nameProp = t.GetProperty("Name");
string? value = (string?)nameProp!.GetValue(instance);
nameProp.SetValue(instance, "Bob");
```

### 2.6. Create instance

```csharp
// Default constructor
var user1 = (User)Activator.CreateInstance(typeof(User))!;

// With args
var user2 = (User)Activator.CreateInstance(typeof(User), "Alice", 30)!;

// Generic
T CreateInstance<T>() where T : new() => Activator.CreateInstance<T>();
```

### 2.7. Generic types

```csharp
// Open generic
Type listType = typeof(List<>);

// Closed generic (specific T)
Type listOfInt = listType.MakeGenericType(typeof(int));   // List<int>

// Create
var list = (List<int>)Activator.CreateInstance(listOfInt)!;
list.Add(42);

// Inspect generic
Type t = typeof(Dictionary<string, int>);
Type[] genericArgs = t.GetGenericArguments();   // [string, int]
Type genericDef = t.GetGenericTypeDefinition(); // Dictionary<,>
```

### 2.8. Attributes

```csharp
[Description("User entity")]
public class User
{
    [Key]
    public int Id { get; set; }
    
    [Required, MaxLength(100)]
    public string Name { get; set; } = "";
}

// Read attributes
var attr = typeof(User).GetCustomAttribute<DescriptionAttribute>();
Console.WriteLine(attr?.Description);   // "User entity"

// All attributes
var all = typeof(User).GetCustomAttributes(inherit: true);

// Per-property
var nameAttr = typeof(User).GetProperty("Name")!.GetCustomAttribute<MaxLengthAttribute>();
Console.WriteLine(nameAttr?.Length);   // 100
```

См. [[attributes-metadata]].

> [!question]- Интервью: что такое BindingFlags в reflection?
> Enum controlling **what to include** в reflection search. Без него — defaults: `Public | Static | Instance` (но не private). **Common combinations**: 1) `Public | Instance` — public instance methods. 2) `Public | NonPublic | Instance` — all instance methods (including private). 3) `Static | Public` — public static. 4) `DeclaredOnly` — only declared в this type, не inherited. 5) `IgnoreCase` — case-insensitive name match. **Use case**: реflection often needs `NonPublic` для frameworks (DI containers access internal constructors). Best practice: explicit flags даже когда default works — clarity.

---

## 3. Reflection performance

### 3.1. Cost overview

```
| Operation | Time |
|-----------|------|
| typeof(T)            | ~0ns (compile-time) |
| obj.GetType()        | ~5ns |
| t.GetMethod("Name")  | ~500ns |
| MethodInfo.Invoke()  | ~500-1000ns per call |
| Direct call           | ~1ns |
| PropertyInfo.GetValue | ~500ns |
| Direct property get   | ~1ns |
| Activator.CreateInstance() | ~1000ns |
| new T()                    | ~10ns |
```

Reflection 100-1000x slower than direct.

### 3.2. Caching pattern

```csharp
public static class UserMapper
{
    private static readonly PropertyInfo[] _properties = typeof(User).GetProperties();
    private static readonly MethodInfo _saveMethod = typeof(User).GetMethod("Save")!;
    
    public static void MapAll(User from, User to)
    {
        foreach (var prop in _properties)
        {
            var value = prop.GetValue(from);
            prop.SetValue(to, value);
        }
    }
}
```

Cache `PropertyInfo` / `MethodInfo` в static field. Lookup once, reuse forever.

### 3.3. Compiled delegates (faster)

```csharp
// Slow — Invoke per call
MethodInfo m = typeof(Calculator).GetMethod("Add")!;
int result = (int)m.Invoke(calc, new object[] { 2, 3 })!;

// Fast — compile to delegate (close to direct call)
var addFunc = (Func<Calculator, int, int, int>)Delegate.CreateDelegate(
    typeof(Func<Calculator, int, int, int>), m);
int result2 = addFunc(calc, 2, 3);
```

`Delegate.CreateDelegate` — converts MethodInfo to typed delegate. ~direct call speed.

### 3.4. Expression tree compile

```csharp
// Generic property setter (faster than PropertyInfo.SetValue)
public static Action<T, TValue> CreateSetter<T, TValue>(string propertyName)
{
    var instance = Expression.Parameter(typeof(T), "instance");
    var value = Expression.Parameter(typeof(TValue), "value");
    var property = Expression.Property(instance, propertyName);
    var assign = Expression.Assign(property, value);
    return Expression.Lambda<Action<T, TValue>>(assign, instance, value).Compile();
}

// Use
var setter = CreateSetter<User, string>("Name");
setter(user, "Bob");   // ~direct call performance
```

Compile expression tree once, cache. Reuse — fast.

### 3.5. Reflection.Emit (deepest)

```csharp
// Generate IL at runtime — fastest possible
var method = new DynamicMethod("Add", typeof(int), new[] { typeof(int), typeof(int) });
var il = method.GetILGenerator();
il.Emit(OpCodes.Ldarg_0);
il.Emit(OpCodes.Ldarg_1);
il.Emit(OpCodes.Add);
il.Emit(OpCodes.Ret);

var add = (Func<int, int, int>)method.CreateDelegate(typeof(Func<int, int, int>));
int r = add(2, 3);   // 5 — IL generated method
```

Used by frameworks (Dapper, AutoMapper, ASP.NET Core MVC). Direct IL — fastest dynamic dispatch.

### 3.6. Performance hierarchy

```
Direct call        ~1ns         baseline
DynamicInvoke      ~500ns       avoid!
MethodInfo.Invoke  ~500-1000ns
Compiled delegate  ~5-20ns
Compiled expr tree ~5-20ns
Reflection.Emit IL ~5-10ns
Source generator   ~1ns         compile-time
```

> [!question]- Интервью: как ускорить reflection в hot path?
> Three strategies, ordered by effort: 1) **Cache `MethodInfo` / `PropertyInfo`** в static readonly — lookup once. Saves 500ns+ per call. 2) **Compiled delegate** через `Delegate.CreateDelegate` для known signatures — ~direct call speed (5-20ns vs 500ns Invoke). 3) **Compiled expression trees** — flexible (build dynamically), ~direct call speed после compile. 4) **Reflection.Emit** — fastest dynamic dispatch (5-10ns), но complex API. 5) **Source generators** (C# 9+) — replace reflection compile-time. Best (zero runtime cost). **Pattern**: heavily used reflection → compiled delegate / source generator. Rarely used → just cache MethodInfo.

---

## 4. Real-world reflection patterns

### 4.1. Plugin loader

```csharp
public static List<IPlugin> LoadPlugins(string directory)
{
    var plugins = new List<IPlugin>();
    
    foreach (var dll in Directory.GetFiles(directory, "*.dll"))
    {
        var assembly = Assembly.LoadFrom(dll);
        var pluginTypes = assembly.GetTypes()
            .Where(t => typeof(IPlugin).IsAssignableFrom(t) && !t.IsAbstract);
        
        foreach (var type in pluginTypes)
        {
            var plugin = (IPlugin)Activator.CreateInstance(type)!;
            plugins.Add(plugin);
        }
    }
    
    return plugins;
}
```

### 4.2. Generic deep clone

```csharp
public static T DeepClone<T>(T source) where T : new()
{
    var result = new T();
    var props = typeof(T).GetProperties().Where(p => p.CanWrite);
    foreach (var p in props)
    {
        p.SetValue(result, p.GetValue(source));
    }
    return result;
}
```

(Не handles nested objects properly — для real production use library like AutoMapper.)

### 4.3. Configuration mapping

```csharp
public static T BindConfig<T>(IConfigurationSection section) where T : new()
{
    var instance = new T();
    foreach (var prop in typeof(T).GetProperties())
    {
        var value = section[prop.Name];
        if (value == null) continue;
        
        var converted = Convert.ChangeType(value, prop.PropertyType);
        prop.SetValue(instance, converted);
    }
    return instance;
}
```

(Реальная реализация в Microsoft.Extensions.Configuration более robust.)

### 4.4. Test discovery

```csharp
public static List<MethodInfo> FindTestMethods(Assembly assembly)
{
    return assembly.GetTypes()
        .SelectMany(t => t.GetMethods(BindingFlags.Public | BindingFlags.Instance))
        .Where(m => m.GetCustomAttribute<TestAttribute>() != null)
        .ToList();
}
```

### 4.5. DI container basics

```csharp
public class SimpleContainer
{
    private readonly Dictionary<Type, Type> _registrations = new();
    
    public void Register<TInterface, TImpl>() where TImpl : TInterface =>
        _registrations[typeof(TInterface)] = typeof(TImpl);
    
    public T Resolve<T>()
    {
        var type = _registrations[typeof(T)];
        var ctor = type.GetConstructors().First();
        var args = ctor.GetParameters()
            .Select(p => GetType().GetMethod(nameof(Resolve))!
                .MakeGenericMethod(p.ParameterType)
                .Invoke(this, null))
            .ToArray();
        return (T)Activator.CreateInstance(type, args)!;
    }
}
```

DI containers (Microsoft.Extensions.DependencyInjection, Autofac) heavily use reflection.

### 4.6. Object initializer mapper

```csharp
public static T Map<T>(IDictionary<string, object> source) where T : new()
{
    var instance = new T();
    var props = typeof(T).GetProperties().ToDictionary(p => p.Name);
    
    foreach (var (key, value) in source)
    {
        if (props.TryGetValue(key, out var prop))
        {
            var converted = Convert.ChangeType(value, prop.PropertyType);
            prop.SetValue(instance, converted);
        }
    }
    return instance;
}

// Use
var dict = new Dictionary<string, object>
{
    ["Id"] = 1,
    ["Name"] = "Alice"
};
var user = Map<User>(dict);
```

> [!question]- Интервью: как работает DI container под капотом?
> 1) **Registration**: dictionary `Type → Type` (interface to implementation). 2) **Resolution**: lookup type, find constructor (longest или [Inject] attribute). 3) **Recursive parameter resolution**: для каждого ctor parameter — resolve через container. 4) **Activation**: `Activator.CreateInstance(type, args)`. **Optimization**: production containers compile expression trees (быстрее `Activator.CreateInstance`). **Lifecycle**: Singleton/Scoped/Transient — different storage. **Reflection cost**: heavy в startup, cached after. Modern containers (Microsoft.Extensions.DependencyInjection) use compiled factories — ~direct call speed для resolutions.

---

## 5. Expression Trees fundamentals

### 5.1. Lambda — два режима

```csharp
// Compiled lambda — runs immediately
Func<int, bool> compiled = x => x > 5;
bool r = compiled(10);   // calls function

// Expression tree — данные о коде
Expression<Func<int, bool>> tree = x => x > 5;
// tree.Body — BinaryExpression {Left=x, Operator=GreaterThan, Right=5}
```

`Expression<Func<...>>` — compile-time C# generates **expression tree object** instead of compiled function.

### 5.2. Anatomy

```csharp
Expression<Func<int, bool>> expr = x => x > 5;

ParameterExpression p = (ParameterExpression)expr.Parameters[0];   // x
BinaryExpression body = (BinaryExpression)expr.Body;
// body.NodeType = ExpressionType.GreaterThan
// body.Left = ParameterExpression (x)
// body.Right = ConstantExpression (5)
```

Lambda тело — tree of `Expression` nodes. Each operator/call/access — node type.

### 5.3. Major node types

```
ConstantExpression    — literal (5, "hello", null)
ParameterExpression   — lambda parameter (x)
BinaryExpression      — operators (+, -, *, ==, &&, etc.)
UnaryExpression       — unary (!, -, cast)
MemberExpression      — property/field access (user.Name)
MethodCallExpression  — method call (user.GetName())
LambdaExpression      — lambda definition
NewExpression         — new T(args)
NewArrayExpression    — new T[] { ... }
ConditionalExpression — ternary (a ? b : c)
```

### 5.4. Build expression tree manually

```csharp
// Build x => x > 5 from scratch
var x = Expression.Parameter(typeof(int), "x");
var five = Expression.Constant(5);
var greater = Expression.GreaterThan(x, five);
var lambda = Expression.Lambda<Func<int, bool>>(greater, x);

// Compile and use
var fn = lambda.Compile();
bool r = fn(10);   // true
```

Expression API — composable builder.

### 5.5. ExpressionVisitor для analysis

```csharp
public class PrintVisitor : ExpressionVisitor
{
    protected override Expression VisitBinary(BinaryExpression node)
    {
        Console.WriteLine($"Binary: {node.NodeType}");
        return base.VisitBinary(node);   // continue traversal
    }
    
    protected override Expression VisitMember(MemberExpression node)
    {
        Console.WriteLine($"Member: {node.Member.Name}");
        return base.VisitMember(node);
    }
}

Expression<Func<User, bool>> expr = u => u.Age > 18;
new PrintVisitor().Visit(expr);
// Binary: GreaterThan
// Member: Age
```

ExpressionVisitor — pattern для analyzing tree. EF Core использует для translation в SQL.

### 5.6. Compile vs Interpret

```csharp
Expression<Func<int, int>> expr = x => x * 2;

// Compile — generates IL (faster runtime, slow compile)
Func<int, int> fn = expr.Compile();

// Interpret — runs without compile (slower runtime, no compile cost)
Func<int, int> interpreted = expr.Compile(preferInterpretation: true);
```

`Compile()` ~100-1000x slower than direct lambda creation, но runtime ~direct speed. `preferInterpretation` — для one-off (no compile cost).

> [!question]- Интервью: чем `Func<T>` отличается от `Expression<Func<T>>`?
> **`Func<T>`** — compiled delegate, runs immediately. Compiler generates IL для lambda body. **`Expression<Func<T>>`** — **expression tree object** describing code structure. Compiler analyzes lambda и generates **data structure** (tree of nodes — BinaryExpression, MemberExpression, etc.) instead of executable code. **Use cases**: Func — direct execution. Expression — analyze (LINQ to SQL translates Expression to SQL), build dynamically (composable queries), compile to delegate later. **Restriction**: Expression supports limited subset of C# — assignments, blocks, multiple statements, async/await не allowed (compile error). For complex code use Func.

---

## 6. LINQ providers — Expression Trees в action

### 6.1. `IQueryable<T>` vs `IEnumerable<T>`

```csharp
// IEnumerable<T> — Func<T, bool>
List<User> users = ...;
var adults = users.Where(u => u.Age > 18);   // executes in memory

// IQueryable<T> — Expression<Func<T, bool>>
IQueryable<User> dbUsers = context.Users;
var dbAdults = dbUsers.Where(u => u.Age > 18);   // translated to SQL!
```

Same syntax, different mechanism:
- `IEnumerable.Where(Func<T, bool>)` — runs lambda directly.
- `IQueryable.Where(Expression<Func<T, bool>>)` — captures tree, provider translates.

### 6.2. EF Core translation

```csharp
// User code
var query = context.Users
    .Where(u => u.Age > 18 && u.IsActive)
    .OrderBy(u => u.Name)
    .Select(u => new { u.Id, u.Name });

// EF Core analyzes Expression<Func<...>> and translates:
// SELECT u.Id, u.Name 
// FROM Users u 
// WHERE u.Age > 18 AND u.IsActive = 1 
// ORDER BY u.Name
```

EF Core ExpressionVisitor walks tree, generates SQL.

### 6.3. Translation limits

```csharp
// ✅ Works — translatable
var x = ctx.Users.Where(u => u.Age > 18).ToList();
var y = ctx.Users.Where(u => u.Name.StartsWith("A")).ToList();
var z = ctx.Users.Where(u => Names.Contains(u.Name)).ToList();   // IN clause

// ❌ Fails — non-translatable
var bad = ctx.Users.Where(u => MyCustomMethod(u.Age)).ToList();   // C# method, no SQL
var bad2 = ctx.Users.Where(u => u.Name == ProcessName()).ToList(); // unless evaluated client-side first
```

Provider должен **understand** every expression node. Custom methods — no SQL equivalent.

### 6.4. Client vs server evaluation

```csharp
// EF Core 3.0+ — client evaluation only at top level
var users = ctx.Users
    .Where(u => u.Age > 18)              // SQL
    .ToList()                             // ToList — switch to in-memory
    .Where(u => MyMethod(u.Name))         // C# in-memory — OK
    .ToList();
```

Best practice: filter / sort на server (SQL), then materialize, then client logic.

### 6.5. Dynamic queries

```csharp
public IQueryable<User> SearchUsers(string? name = null, int? minAge = null)
{
    IQueryable<User> q = ctx.Users;
    
    if (!string.IsNullOrEmpty(name))
        q = q.Where(u => u.Name.Contains(name));
    
    if (minAge.HasValue)
        q = q.Where(u => u.Age >= minAge.Value);
    
    return q;
}
```

Composable queries — each Where добавляет node к expression tree.

### 6.6. Custom predicate building

```csharp
// PredicateBuilder pattern — combine expressions с OR/AND
public static Expression<Func<T, bool>> Or<T>(
    Expression<Func<T, bool>> first,
    Expression<Func<T, bool>> second)
{
    var param = Expression.Parameter(typeof(T));
    var firstBody = new ParamReplacer(first.Parameters[0], param).Visit(first.Body);
    var secondBody = new ParamReplacer(second.Parameters[0], param).Visit(second.Body);
    var or = Expression.OrElse(firstBody, secondBody);
    return Expression.Lambda<Func<T, bool>>(or, param);
}

// Use
var p1 = (User u) => u.Age > 18;
var p2 = (User u) => u.IsAdmin;
var combined = Or(p1, p2);   // u => u.Age > 18 || u.IsAdmin
var users = ctx.Users.Where(combined).ToList();
```

PredicateBuilder libraries — dynamic SQL queries без string concat.

### 6.7. Specification pattern

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();
    
    public Specification<T> And(Specification<T> other) => 
        new AndSpec<T>(this, other);
    
    public Specification<T> Or(Specification<T> other) =>
        new OrSpec<T>(this, other);
}

public class IsAdult : Specification<User>
{
    public override Expression<Func<User, bool>> ToExpression() =>
        u => u.Age >= 18;
}

// Use
var spec = new IsAdult().And(new IsActive());
var users = ctx.Users.Where(spec.ToExpression()).ToList();
```

Specification — composable query patterns. Used в DDD.

> [!question]- Интервью: как EF Core превращает LINQ в SQL?
> 1) **Capture lambda как Expression**: `IQueryable<T>.Where(Expression<Func<T, bool>>)` принимает дерево, не делегат. 2) **ExpressionVisitor walks tree**: traverses each node (BinaryExpression, MemberExpression, MethodCallExpression). 3) **Map to SQL fragments**: `>` → SQL `>`, `string.StartsWith("A")` → `LIKE 'A%'`, `Contains` → `IN`. 4) **Build SQL string** с parameters (preventing injection). 5) **Execute against DB**, hydrate results to entities. **Limits**: provider must understand each node. Custom methods (`MyHelper(x)`) fail — no SQL equivalent. EF Core 3+ disallows client evaluation в WHERE (used to silently fall back, hiding perf issues). Best practice: keep WHERE/ORDER simple, use custom queries (FromSqlRaw) if complex.

---

## 7. Compiled expressions для performance

### 7.1. Property setter generator

```csharp
public static class FastReflection
{
    public static Action<object, object> CreateSetter(PropertyInfo prop)
    {
        var instance = Expression.Parameter(typeof(object), "instance");
        var value = Expression.Parameter(typeof(object), "value");
        
        var instanceCast = Expression.Convert(instance, prop.DeclaringType!);
        var valueCast = Expression.Convert(value, prop.PropertyType);
        var assignment = Expression.Assign(
            Expression.Property(instanceCast, prop),
            valueCast);
        
        var lambda = Expression.Lambda<Action<object, object>>(assignment, instance, value);
        return lambda.Compile();
    }
}

// Cache compiled setters
private static readonly Dictionary<PropertyInfo, Action<object, object>> _setters = new();

public static void SetFast(object instance, PropertyInfo prop, object value)
{
    if (!_setters.TryGetValue(prop, out var setter))
        _setters[prop] = setter = FastReflection.CreateSetter(prop);
    setter(instance, value);
}
```

Compile once, reuse — ~direct speed vs PropertyInfo.SetValue.

### 7.2. Object factory

```csharp
public static Func<T> CreateFactory<T>() where T : new()
{
    var ctor = typeof(T).GetConstructor(Type.EmptyTypes)!;
    return Expression.Lambda<Func<T>>(Expression.New(ctor)).Compile();
}

// Cached factory ~5ns vs Activator.CreateInstance ~1000ns
private static readonly Func<User> _userFactory = CreateFactory<User>();
var user = _userFactory();
```

### 7.3. Object mapper (AutoMapper-like)

```csharp
public static class FastMapper<TSource, TDest> where TDest : new()
{
    private static readonly Func<TSource, TDest> _map = CreateMapper();
    
    public static TDest Map(TSource source) => _map(source);
    
    private static Func<TSource, TDest> CreateMapper()
    {
        var sourceParam = Expression.Parameter(typeof(TSource), "source");
        var bindings = new List<MemberBinding>();
        
        var destProps = typeof(TDest).GetProperties().Where(p => p.CanWrite);
        var sourceProps = typeof(TSource).GetProperties()
            .Where(p => p.CanRead).ToDictionary(p => p.Name);
        
        foreach (var dest in destProps)
        {
            if (sourceProps.TryGetValue(dest.Name, out var src) && src.PropertyType == dest.PropertyType)
            {
                var sourceProperty = Expression.Property(sourceParam, src);
                bindings.Add(Expression.Bind(dest, sourceProperty));
            }
        }
        
        var newDest = Expression.MemberInit(Expression.New(typeof(TDest)), bindings);
        return Expression.Lambda<Func<TSource, TDest>>(newDest, sourceParam).Compile();
    }
}

// Use
var dto = FastMapper<User, UserDto>.Map(user);
```

Compiled once — ~direct property copy speed.

### 7.4. Equality comparer

```csharp
public static Func<T, T, bool> CreateEqualityComparer<T>()
{
    var x = Expression.Parameter(typeof(T), "x");
    var y = Expression.Parameter(typeof(T), "y");
    
    var props = typeof(T).GetProperties();
    Expression body = Expression.Constant(true);
    
    foreach (var p in props)
    {
        var xProp = Expression.Property(x, p);
        var yProp = Expression.Property(y, p);
        var equals = Expression.Call(
            typeof(EqualityComparer<>).MakeGenericType(p.PropertyType)
                .GetProperty("Default")!.GetGetMethod()!.Invoke(null, null) as object,
            "Equals",
            null,
            xProp, yProp) as Expression;
        body = Expression.AndAlso(body, equals!);
    }
    
    return Expression.Lambda<Func<T, T, bool>>(body, x, y).Compile();
}
```

(Сложный — настоящие libraries handle edge cases. Concept demonstrated.)

### 7.5. Performance comparison

```
Direct property access:           ~1ns
Compiled expression tree access:  ~5-10ns
Reflection PropertyInfo.GetValue: ~500ns
Reflection.Emit IL generation:    ~5-10ns
```

Compiled expressions — close to direct, much faster than reflection invoke.

> [!question]- Интервью: как ускорить object mapping в C#?
> 1) **Source generator** (compile-time) — fastest, no runtime cost. Tools: Mapperly, MapsterCompiler. 2) **Compiled expression trees** — Mapster, AutoMapper после optimization. ~5-10ns per property copy. 3) **Reflection.Emit IL** — fastest dynamic option, complex API. 4) **Cached PropertyInfo + delegate** — middle ground. 5) **Avoid**: PropertyInfo.GetValue/SetValue в hot path (~500ns each). **Best practice 2024+**: source generator (compile-time), e.g. Mapperly. Failing that, compiled expressions cached by type pair. **Anti-pattern**: AutoMapper для simple mappings — overhead > benefit для < 10 properties. Hand-written code beats AutoMapper в most cases.

---

## 8. dynamic keyword

### 8.1. Что это

```csharp
dynamic d = "hello";
Console.WriteLine(d.Length);   // resolved at runtime — works для string

d = 42;
Console.WriteLine(d.GetType());   // System.Int32

d.NonExistentMethod();   // RuntimeBinderException at runtime!
```

`dynamic` — bypass compile-time type check. Runtime resolution через DLR (Dynamic Language Runtime).

### 8.2. Use cases

```
✅ COM interop (Office automation, Excel)
✅ Dynamic JSON (without strong typing)
✅ Python/Ruby interop (IronPython)
✅ Expandable objects (ExpandoObject)

❌ Replacement для proper types
❌ Hot path (slow)
❌ AOT (mostly broken)
```

### 8.3. ExpandoObject

```csharp
dynamic obj = new ExpandoObject();
obj.Name = "Alice";
obj.Age = 30;
obj.Greet = (Action)(() => Console.WriteLine($"Hello {obj.Name}"));
obj.Greet();

// Inspect as IDictionary
var dict = (IDictionary<string, object>)obj;
foreach (var (key, value) in dict)
    Console.WriteLine($"{key}: {value}");
```

`ExpandoObject` — runtime properties addable.

### 8.4. Performance

```
| Operation | Time |
|-----------|------|
| Direct call | ~1ns |
| Virtual call | ~2ns |
| Interface call | ~5ns |
| dynamic call | ~50-200ns |
| Reflection Invoke | ~500-1000ns |
```

`dynamic` faster than reflection (DLR caches), but slower than typed.

### 8.5. Не для AOT

```csharp
// AOT — minimal/no support для dynamic
// Code IL2026 warnings everywhere
```

`dynamic` requires DLR + runtime IL generation. AOT doesn't allow.

> [!question]- Интервью: чем `dynamic` отличается от `object`?
> **`object`** — base type, всё **statically typed** as object. Methods accessed через casting/reflection. Compile-time checks. **`dynamic`** — type "any", **runtime resolution** через DLR. `d.Method()` — compiler generates DLR call site, resolved at runtime. **Use cases dynamic**: COM interop (Excel/Word automation), dynamic JSON parsing, Python interop. **Costs**: 50-200ns per call (vs 1ns direct), AOT incompatible, no IDE intellisense. **Best practice**: avoid в new code unless interop with Office/COM. JSON парсинг — `JsonNode` или strongly-typed deserialization. ExpandoObject — для truly dynamic shapes.

---

## 9. AOT considerations

### 9.1. Что ломается в AOT

```
Reflection cases AOT cannot handle:
- Type.GetType("Namespace.TypeName") — no metadata
- MakeGenericType(...) при unknown T — code не generated
- Activator.CreateInstance(t) — depends on metadata
- DynamicInvoke — requires runtime code gen
- Reflection.Emit — disabled

Reflection cases AOT supports (с annotations):
- typeof(KnownType) — works
- GetMethod / GetProperty на known types — works но trim warnings
- Custom attributes — works
```

### 9.2. Trim warnings

```bash
warning IL2026: Method 'X' which has 'RequiresUnreferencedCodeAttribute'
warning IL3050: Generic methods used with reflection (AOT specific)
```

Build тревожит, что reflection может break trim-removed code.

### 9.3. Annotations для AOT-friendly reflection

```csharp
public T CreateInstance<[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors)] T>()
{
    return Activator.CreateInstance<T>();
}

[RequiresUnreferencedCode("Calls Reflection")]
public void Process(Type type)
{
    var ctor = type.GetConstructors().First();
    // ...
}
```

`[DynamicallyAccessedMembers]` — tells trim tool keep these members. `[RequiresUnreferencedCode]` — propagate warnings.

### 9.4. Source generators replace

```csharp
// Reflection-based — breaks AOT
var json = JsonSerializer.Serialize(user);   // reflection inside

// Source generator — works AOT
[JsonSerializable(typeof(User))]
internal partial class MyContext : JsonSerializerContext { }
var json = JsonSerializer.Serialize(user, MyContext.Default.User);
```

```csharp
// Reflection P/Invoke — slow
[DllImport("user32.dll")]
static extern int MessageBox(...);

// Source generator P/Invoke — AOT-friendly
[LibraryImport("user32.dll")]
static partial int MessageBox(...);
```

### 9.5. Common reflection alternatives

| Reflection use | AOT alternative |
|----------------|-----------------|
| JsonSerializer (default) | JsonSerializerContext (source-gen) |
| DllImport | LibraryImport (source-gen) |
| ASP.NET MVC reflection | RDG (Request Delegate Generator, source-gen) |
| EF Core reflection | Compiled queries, AOT support .NET 8+ |
| Microsoft.Extensions.Configuration | Configuration source-gen (.NET 8+) |
| Logging | LoggerMessage (source-gen) |

### 9.6. Test AOT в CI

```bash
dotnet publish -c Release -r linux-x64 -p:PublishAot=true
# Watch for warnings IL2026/IL3050
# Build fails если critical issues
```

> [!question]- Интервью: почему reflection problematic для Native AOT?
> Native AOT (.NET 8+) — ahead-of-time compilation, **no JIT**. Reflection requires: 1) **Type metadata** preserved (trimming removes unused → reflection sees "missing type"). 2) **Runtime code generation** (AOT disallows). 3) **Generic instantiation** на unknown T (compiler must generate compile-time). **What breaks**: `Type.GetType(string)`, `MakeGenericType` с unknown, `Activator.CreateInstance(t)`, `Reflection.Emit`, `DynamicInvoke`. **What works**: typeof on known types, GetMethod/GetProperty с annotations. **AOT-friendly alternatives**: source generators (JsonSerializerContext, LibraryImport, LoggerMessage), compiled queries (EF Core), Reflection.Emit replaced by source generation. Best practice: design AOT-first if Native AOT critical.

---

## 10. Common reflection patterns в .NET

### 10.1. Microsoft.Extensions.DependencyInjection

```csharp
// Под капотом — reflection для constructor inspection
public T Resolve<T>()
{
    var type = typeof(T);
    var ctor = type.GetConstructors()
        .OrderByDescending(c => c.GetParameters().Length)
        .First();
    
    var args = ctor.GetParameters()
        .Select(p => Resolve(p.ParameterType))
        .ToArray();
    
    return (T)ctor.Invoke(args);
}
```

(Production version uses compiled expression trees / source generators.)

### 10.2. AutoMapper

```csharp
// Maps based on convention + configuration
var config = new MapperConfiguration(cfg => cfg.CreateMap<User, UserDto>());
var mapper = config.CreateMapper();
var dto = mapper.Map<UserDto>(user);

// Internally — compiled expression trees
```

### 10.3. Dapper

```csharp
// Dapper — micro-ORM, uses Reflection.Emit
var users = connection.Query<User>("SELECT * FROM Users").ToList();
// Generates IL для column → property mapping
```

### 10.4. Newtonsoft.Json / System.Text.Json

```csharp
// Newtonsoft — reflection-heavy
var obj = JsonConvert.DeserializeObject<User>(json);

// System.Text.Json — also reflection (but optimizable)
var obj2 = JsonSerializer.Deserialize<User>(json);

// AOT — source generator
var obj3 = JsonSerializer.Deserialize(json, MyContext.Default.User);
```

### 10.5. ASP.NET Core MVC

```csharp
// Controller method discovery — reflection
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult Get(int id) => Ok(new User { Id = id });
}
```

ASP.NET Core scans assemblies for `[ApiController]`, finds `[HttpGet]` methods, generates routes — reflection. Modern minimal APIs and RDG (.NET 7+) reduce reflection.

### 10.6. xUnit / NUnit

```csharp
[Fact]
public void Test_Something() { }

[Theory]
[InlineData(1)]
[InlineData(2)]
public void Test_With_Data(int x) { }
```

Test framework finds `[Fact]`/`[Theory]` через reflection scan.

> [!question]- Интервью: какие .NET frameworks heavily используют reflection?
> 1) **DI containers** (Microsoft.Extensions.DependencyInjection, Autofac) — constructor inspection, dependency resolution. 2) **ORM** (EF Core, Dapper) — entity property mapping, query generation. 3) **Serializers** (Newtonsoft.Json, System.Text.Json) — property/field discovery. 4) **AutoMapper / Mapster** — object mapping. 5) **ASP.NET Core MVC** — controller discovery, model binding, action invocation. 6) **Test frameworks** (xUnit, NUnit) — `[Fact]` / `[Test]` discovery. 7) **MEF / VS Extensions** — plugin systems. **Modern trend**: source generators replace runtime reflection (compile-time codegen). Examples: System.Text.Json source gen, ASP.NET Core RDG, EF Core compiled queries. Better perf + AOT compat.

---

## 11. Best practices

### 11.1. Reflection performance

- ✅ **Cache `MethodInfo` / `PropertyInfo`** в static field.
- ✅ **Compile delegates** через `Delegate.CreateDelegate` или expression trees.
- ✅ **Source generators** где возможно (compile-time).
- ✅ **Fluent builder pattern** для cached lookups.
- ❌ Reflection в hot path tight loops.
- ❌ `Activator.CreateInstance` без caching (use `Func<T>`).

### 11.2. Expression trees

- ✅ **Cache compiled** expression trees.
- ✅ **ExpressionVisitor** для analysis.
- ✅ **PredicateBuilder** для dynamic queries.
- ✅ **Specification pattern** для composable queries.
- ❌ Compile внутри loop (compile cost ~100-1000x slower than execution).
- ❌ Complex async/multi-statement expressions (limited support).

### 11.3. AOT compatibility

- ✅ **`[DynamicallyAccessedMembers]`** annotations.
- ✅ **Source generators** (JsonSerializer, LibraryImport).
- ✅ **Test AOT build** в CI.
- ❌ `Type.GetType(string)` — fragile.
- ❌ `MakeGenericType` с unknown T.
- ❌ Reflection.Emit (no support).

### 11.4. Не делай

- ❌ Reflection для simple cases (just write code).
- ❌ `dynamic` без COM interop reason.
- ❌ Skip caching — assume each call cheap.
- ❌ Trust users — sanitize Type.GetType inputs.
- ❌ Reflection через many layers of indirection (debugging hell).

---

## 12. Decision tree

```
Need runtime introspection?
│
├── Read metadata (types, attributes)
│   ├── Compile-time known → typeof(T), nameof()
│   └── Runtime discovery → Type.GetType / Assembly.GetTypes
│
├── Invoke method dynamically
│   ├── Frequent → compile to delegate (Delegate.CreateDelegate / expression tree)
│   ├── Rare → MethodInfo.Invoke + cache MethodInfo
│   └── Complex generic → Reflection.Emit (frameworks only)
│
├── Property access
│   ├── Frequent → compiled expression tree (Action<T, TValue>)
│   ├── Rare → PropertyInfo.GetValue + cache PropertyInfo
│   └── Source generator if structure known
│
├── Create instance
│   ├── Frequent → cached Func<T> (compiled expression)
│   ├── Rare → Activator.CreateInstance
│   └── Generic → Activator.CreateInstance<T>
│
├── Build dynamic query
│   ├── LINQ provider (EF Core) → IQueryable + Expression
│   ├── Custom predicate → PredicateBuilder pattern
│   └── Specification pattern для DDD
│
├── AOT compilation
│   ├── Replace JsonSerializer reflection → JsonSerializerContext
│   ├── Replace DllImport → LibraryImport
│   ├── Replace logging reflection → LoggerMessage
│   └── Annotate reflection с [DynamicallyAccessedMembers]
│
└── Plugin / extension system
    ├── Assembly.LoadFrom + scan types
    ├── Cache discovery results
    └── Consider AssemblyLoadContext (isolation)
```

---

## 13. Cheat sheet

```csharp
using System.Reflection;
using System.Linq.Expressions;

// === Reflection basics ===
Type t = typeof(User);
MethodInfo m = t.GetMethod("Save")!;
PropertyInfo p = t.GetProperty("Name")!;
FieldInfo f = t.GetField("_internal", BindingFlags.NonPublic | BindingFlags.Instance)!;

// Invoke
var instance = Activator.CreateInstance(t);
m.Invoke(instance, args: null);
var name = (string)p.GetValue(instance)!;
p.SetValue(instance, "Bob");

// === Generics ===
Type listOfInt = typeof(List<>).MakeGenericType(typeof(int));
var list = Activator.CreateInstance(listOfInt);

// Generic method
MethodInfo generic = m.MakeGenericMethod(typeof(int));

// === Attributes ===
var attr = t.GetCustomAttribute<DescriptionAttribute>();
var allAttrs = t.GetCustomAttributes<ValidationAttribute>(inherit: true);

// === Compiled delegate (faster) ===
var addFunc = (Func<Calc, int, int, int>)Delegate.CreateDelegate(
    typeof(Func<Calc, int, int, int>), m);

// === Expression Trees ===
Expression<Func<int, bool>> expr = x => x > 5;

// Manual building
var x = Expression.Parameter(typeof(int), "x");
var body = Expression.GreaterThan(x, Expression.Constant(5));
var lambda = Expression.Lambda<Func<int, bool>>(body, x);
var fn = lambda.Compile();

// === Property setter (compiled, fast) ===
public static Action<T, TValue> CreateSetter<T, TValue>(string name)
{
    var instance = Expression.Parameter(typeof(T));
    var value = Expression.Parameter(typeof(TValue));
    var prop = Expression.Property(instance, name);
    var assign = Expression.Assign(prop, value);
    return Expression.Lambda<Action<T, TValue>>(assign, instance, value).Compile();
}

// === ExpressionVisitor ===
public class MyVisitor : ExpressionVisitor
{
    protected override Expression VisitBinary(BinaryExpression node)
    {
        Console.WriteLine($"Op: {node.NodeType}");
        return base.VisitBinary(node);
    }
}

new MyVisitor().Visit(expr);

// === Object factory ===
public static Func<T> CreateFactory<T>() where T : new()
{
    return Expression.Lambda<Func<T>>(
        Expression.New(typeof(T))).Compile();
}

// === dynamic ===
dynamic obj = new ExpandoObject();
obj.Name = "Alice";
obj.Age = 30;

// === AOT-friendly ===
[JsonSerializable(typeof(User))]
internal partial class MyContext : JsonSerializerContext { }
JsonSerializer.Serialize(user, MyContext.Default.User);
```

---

## 14. Common pitfalls

### 14.1. Reflection без caching

```csharp
public string GetName(User u)
{
    var prop = typeof(User).GetProperty("Name")!;   // ❌ each call expensive
    return (string)prop.GetValue(u)!;
}
```

**Фикс:** static field cache.

### 14.2. Compile expression в loop

```csharp
foreach (var item in items)
{
    Expression<Func<int>> e = () => item.Calc();
    var fn = e.Compile();   // ❌ compile per item — slow
    fn();
}
```

**Фикс:** compile once outside loop, или just write delegate.

### 14.3. Type.GetType(string) хрупкий

```csharp
var t = Type.GetType("MyApp.MyClass");   // ❌ depends on assembly load
// May return null если type not found
```

**Фикс:** verify result, use full assembly-qualified name.

### 14.4. Assembly не loaded

```csharp
var types = AppDomain.CurrentDomain.GetAssemblies()
    .SelectMany(a => a.GetTypes());   // ❌ только loaded assemblies
```

**Фикс:** explicitly `Assembly.LoadFrom` для plugin DLLs.

### 14.5. ReflectionTypeLoadException

```csharp
var types = assembly.GetTypes();   // ❌ throws если any dependency missing
```

**Фикс:** try/catch + handle `ex.Types`:
```csharp
try { var types = assembly.GetTypes(); }
catch (ReflectionTypeLoadException ex) { var types = ex.Types.Where(t => t != null); }
```

### 14.6. Generic methods через reflection

```csharp
// Reflection-invoke generic method
var method = typeof(MyClass).GetMethod("Process")!;
var generic = method.MakeGenericMethod(typeof(int));
generic.Invoke(instance, args);   // ❌ slow — compile if frequent
```

**Фикс:** compile to delegate.

### 14.7. EF Core non-translatable

```csharp
// ❌ Won't translate to SQL
var users = ctx.Users.Where(u => MyMethod(u.Age) > 18).ToList();

// Error: "could not be translated"
```

**Фикс:** evaluate client-side after SQL filter:
```csharp
var users = ctx.Users.Where(u => u.Age > 0).ToList()   // SQL filter
    .Where(u => MyMethod(u.Age) > 18).ToList();         // C# filter
```

### 14.8. Expression compile is heavy

```csharp
// ❌ 1000+ ns to compile
var fn = expr.Compile();
```

Cache compiled, do once per type/method combination.

### 14.9. dynamic + struct

```csharp
struct Point { public int X, Y; }

dynamic d = new Point();
d.X = 5;   // ❌ — dynamic boxes struct, modifies copy
```

**Фикс:** не используй dynamic с structs.

### 14.10. AOT trim warnings ignored

```bash
warning IL2026: ...
warning IL3050: ...
# Build succeeds but runtime crashes
```

**Фикс:** address все warnings перед production AOT build.

> [!question]- Интервью: топ-3 ошибки с reflection?
> 1) **No caching** — `typeof(T).GetProperty("Name")` в hot path 500ns+ per call. Cache `PropertyInfo` в static. 2) **Compile expression в loop** — `expr.Compile()` 100-1000x slower than execution. Compile once outside loop, reuse. 3) **`Type.GetType(string)` returns null silently** — type not found из-за assembly не loaded или wrong name. Always verify result, use AssemblyQualifiedName for cross-assembly. Бонус: `ReflectionTypeLoadException` при missing dependencies — try/catch и use ex.Types для partial results.

---

## 15. Practice exercises

### 15.1. Generic property mapper

```csharp
public static class Mapper<TSrc, TDest> where TDest : new()
{
    private static readonly Func<TSrc, TDest> _map = BuildMapper();
    
    public static TDest Map(TSrc source) => _map(source);
    
    private static Func<TSrc, TDest> BuildMapper()
    {
        var sourceParam = Expression.Parameter(typeof(TSrc), "src");
        var bindings = new List<MemberBinding>();
        
        var sourceProps = typeof(TSrc).GetProperties()
            .Where(p => p.CanRead).ToDictionary(p => p.Name);
        
        foreach (var dest in typeof(TDest).GetProperties().Where(p => p.CanWrite))
        {
            if (sourceProps.TryGetValue(dest.Name, out var src))
            {
                if (src.PropertyType == dest.PropertyType)
                {
                    var srcAccess = Expression.Property(sourceParam, src);
                    bindings.Add(Expression.Bind(dest, srcAccess));
                }
            }
        }
        
        var newDest = Expression.MemberInit(Expression.New(typeof(TDest)), bindings);
        return Expression.Lambda<Func<TSrc, TDest>>(newDest, sourceParam).Compile();
    }
}

// Use
var dto = Mapper<User, UserDto>.Map(user);
```

### 15.2. Plugin discovery

```csharp
public class PluginLoader
{
    public List<TPlugin> Load<TPlugin>(string directory) where TPlugin : class
    {
        var plugins = new List<TPlugin>();
        
        foreach (var dll in Directory.GetFiles(directory, "*.dll"))
        {
            try
            {
                var assembly = Assembly.LoadFrom(dll);
                var types = assembly.GetTypes()
                    .Where(t => typeof(TPlugin).IsAssignableFrom(t) 
                                && !t.IsAbstract 
                                && t.GetConstructor(Type.EmptyTypes) != null);
                
                foreach (var type in types)
                {
                    var plugin = (TPlugin)Activator.CreateInstance(type)!;
                    plugins.Add(plugin);
                }
            }
            catch (ReflectionTypeLoadException ex)
            {
                Console.Error.WriteLine($"Could not load {dll}: {ex.Message}");
            }
        }
        
        return plugins;
    }
}

// Use
public interface IPlugin { void Execute(); }
var loader = new PluginLoader();
var plugins = loader.Load<IPlugin>("./plugins");
foreach (var p in plugins) p.Execute();
```

### 15.3. Specification pattern для EF Core

```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T, bool>> ToExpression();
    
    public Specification<T> And(Specification<T> other) => new AndSpec<T>(this, other);
    public Specification<T> Or(Specification<T> other) => new OrSpec<T>(this, other);
    public Specification<T> Not() => new NotSpec<T>(this);
}

public class AndSpec<T> : Specification<T>
{
    private readonly Specification<T> _left, _right;
    public AndSpec(Specification<T> left, Specification<T> right)
    {
        _left = left; _right = right;
    }
    public override Expression<Func<T, bool>> ToExpression()
    {
        var l = _left.ToExpression();
        var r = _right.ToExpression();
        var param = Expression.Parameter(typeof(T));
        var visitor = new ParamReplacer(param);
        var body = Expression.AndAlso(
            visitor.Visit(l.Body)!,
            visitor.Visit(r.Body)!);
        return Expression.Lambda<Func<T, bool>>(body, param);
    }
}

public class IsAdult : Specification<User>
{
    public override Expression<Func<User, bool>> ToExpression() => u => u.Age >= 18;
}

// Use
var spec = new IsAdult().And(new IsActive());
var users = ctx.Users.Where(spec.ToExpression()).ToList();
```

---

## 16. Что читать дальше

1. **[[source-generators|Source Generators]]** — replace reflection.
2. **[[attributes-metadata|Attributes]]** — base для reflection.
3. **EF Core internals — query translation**.
4. **AutoMapper / Mapster source code**.
5. **Native AOT documentation**.

---

## 17. См. также

- [[source-generators|Source Generators]] — compile-time alternative
- [[attributes-metadata|Attributes and Metadata]]
- [[generics-deep|Generics Deep]] — open/closed types
- [[unsafe-pointers|Unsafe / AOT]] — Native AOT considerations
- System.Reflection namespace
- System.Linq.Expressions namespace
- Roslyn — github.com/dotnet/roslyn

---

## 18. Reading list

- **Microsoft Docs — Reflection** — learn.microsoft.com/dotnet/csharp/programming-guide/concepts/reflection
- **Microsoft Docs — Expression Trees** — learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/
- **Microsoft Docs — DynamicallyAccessedMembers** — learn.microsoft.com/dotnet/api/system.diagnostics.codeanalysis.dynamicallyaccessedmembersattribute
- **Microsoft Docs — Native AOT trim** — learn.microsoft.com/dotnet/core/deploying/native-aot/
- **Stephen Toub — Reflection performance** — devblogs.microsoft.com
- **Marc Gravell — Dapper, FastMember** — github.com/mgravell
- **Jon Skeet — Reflection chapter** "C# in Depth"
- **Jeffrey Richter — CLR via C#** (book) — reflection internals
- **Bart de Smet — Expression trees** blog series
- **AutoMapper documentation** — automapper.org
- **Mapster** — github.com/MapsterMapper/Mapster
