---
tags: [csharp, reflection, expression-trees, dynamic, il-emit, metaprogramming]
level: Senior
date: 2026-04-30
---

# Reflection и Expression Trees

> Метапрограммирование в .NET — фундамент для EF Core, AutoMapper, MediatR, JSON serializers, DI containers, validation libraries. Закрывает: Reflection API, Expression Trees, Dynamic, IL emit, Source Generators как альтернатива, performance trade-offs.

---

## Что это, зачем и когда

### Что такое Reflection?
**Способность кода узнавать о самом себе во время выполнения** — какие типы есть, какие у них свойства/методы/атрибуты, и динамически их вызывать.

**Аналогия:** Программа может «прочитать» свой собственный исходный код и принимать решения на его основе. Как актёр, который читает сценарий прямо во время спектакля.

### Что такое Expression Trees?
**Код как данные** — выражение `x => x.Name == "John"` представляется в виде древовидной структуры объектов, которую можно анализировать, модифицировать и компилировать в делегат во время выполнения.

**Аналогия:** Кулинарный рецепт можно либо просто приготовить (delegate), либо «прочитать» и разобрать на ингредиенты + шаги (Expression Tree). EF Core «читает» твой LINQ и переводит в SQL.

### Зачем?

| Без reflection / expressions | С ними |
|------------------------------|--------|
| Каждый раз пишешь mapper руками: `dto.Name = entity.Name; dto.Age = entity.Age...` | AutoMapper читает свойства, делает всё сам |
| Каждый repository пишет SQL руками | EF Core берёт `Where(x => x.Total > 100)`, переводит в SQL |
| DI container требует регистрировать всё | `AddTransient<IService, Service>()` — DI находит конструктор, его параметры |
| Сериализация — пишем код для каждого типа | `JsonSerializer.Serialize(obj)` — читает свойства автоматически |
| Plugin система? Невозможно | `Assembly.LoadFrom`, `CreateInstance` — динамическая загрузка |

### Когда что использовать

| Инструмент | Когда | Performance |
|-----------|-------|-------------|
| **Reflection** | Простой динамический доступ, плагины, runtime-инспекция | Медленно (~100x slower invoke) |
| **Expression Trees** | Translate code → SQL/другой язык, lazy evaluation | Быстрее reflection после compile |
| **Compiled Expression** (`.Compile()`) | Reusable invoke с почти native speed | После compile — близко к статической |
| **Dynamic** | Interop с COM, dynamic languages, JSON parsing | Среднее (DLR cache) |
| **IL Emit** (DynamicMethod) | Максимальная perf, jit-time codegen | Native speed после emit |
| **Source Generators** | Compile-time alternative к reflection | Нативная скорость, больше build time |

> [!info] Современный тренд (.NET 6+)
> **Source Generators вытесняют reflection** для известных в compile-time сценариев (JSON serialization через `JsonSerializerContext`, regex через `[GeneratedRegex]`, logging через `LoggerMessage`). Reflection остаётся для случаев когда типы неизвестны до runtime.

---

## 1. Reflection API — основы

### Получение типа

```csharp
// Из объекта
var obj = new Person { Name = "John" };
Type t1 = obj.GetType();

// Из generic параметра
Type t2 = typeof(Person);

// Из строки (e.g., из конфига)
Type? t3 = Type.GetType("MyApp.Domain.Person, MyApp.Domain");
Type? t4 = assembly.GetType("MyApp.Domain.Person");

// Все типы из сборки
Assembly asm = typeof(Person).Assembly;
Type[] types = asm.GetTypes();
Type[] exported = asm.GetExportedTypes();  // только public
```

### Чтение метаданных

```csharp
Type t = typeof(Person);

// Properties
PropertyInfo[] props = t.GetProperties(BindingFlags.Public | BindingFlags.Instance);

foreach (var prop in props)
{
    Console.WriteLine($"{prop.Name} : {prop.PropertyType}");
    Console.WriteLine($"  Read: {prop.CanRead}, Write: {prop.CanWrite}");
    
    // Атрибуты
    var required = prop.GetCustomAttribute<RequiredAttribute>();
    if (required != null) Console.WriteLine("  Required!");
}

// Methods
MethodInfo[] methods = t.GetMethods();
MethodInfo? specific = t.GetMethod("DoWork", new[] { typeof(string), typeof(int) });

// Constructors
ConstructorInfo[] ctors = t.GetConstructors();
ConstructorInfo? defaultCtor = t.GetConstructor(Type.EmptyTypes);

// Fields (включая private)
FieldInfo[] fields = t.GetFields(BindingFlags.NonPublic | BindingFlags.Instance);

// Атрибуты класса
var attrs = t.GetCustomAttributes<TableAttribute>();
```

### BindingFlags — какие члены искать

```csharp
// Public + Instance — default
t.GetProperties();

// Включить private
t.GetProperties(BindingFlags.NonPublic | BindingFlags.Instance);

// Только static
t.GetMethods(BindingFlags.Public | BindingFlags.Static);

// Все вместе
t.GetMembers(
    BindingFlags.Public | 
    BindingFlags.NonPublic | 
    BindingFlags.Instance | 
    BindingFlags.Static);
```

### Создание объектов

```csharp
// Через default constructor
var obj1 = Activator.CreateInstance(typeof(Person));
var obj2 = Activator.CreateInstance<Person>();

// С параметрами
var obj3 = Activator.CreateInstance(typeof(Person), "John", 30);

// Generic
var listType = typeof(List<>).MakeGenericType(typeof(string));
var list = Activator.CreateInstance(listType);

// Через ConstructorInfo (быстрее повторно)
var ctor = typeof(Person).GetConstructor(new[] { typeof(string), typeof(int) });
var person = ctor!.Invoke(new object[] { "John", 30 });
```

### Чтение/запись свойств

```csharp
var person = new Person();
PropertyInfo nameProp = typeof(Person).GetProperty("Name")!;

// Запись
nameProp.SetValue(person, "John");

// Чтение
string name = (string)nameProp.GetValue(person)!;

// Поля
FieldInfo field = typeof(Person).GetField("_internal", BindingFlags.NonPublic | BindingFlags.Instance)!;
field.SetValue(person, "value");
```

### Вызов методов

```csharp
var calc = new Calculator();
MethodInfo addMethod = typeof(Calculator).GetMethod("Add")!;

var result = (int)addMethod.Invoke(calc, new object[] { 2, 3 })!;  // 5

// Generic методы
MethodInfo genericMethod = typeof(Helper).GetMethod("Process")!;
MethodInfo concrete = genericMethod.MakeGenericMethod(typeof(string));
concrete.Invoke(null, new[] { "hello" });

// Static
MethodInfo staticMethod = typeof(Math).GetMethod("Sqrt", new[] { typeof(double) })!;
var root = staticMethod.Invoke(null, new object[] { 16.0 });  // null target для static
```

---

## 2. Reflection — performance

### Базовый замер

```csharp
[MemoryDiagnoser]
public class ReflectionBenchmark
{
    private readonly Person _person = new();
    private readonly PropertyInfo _nameProp = typeof(Person).GetProperty("Name")!;

    [Benchmark(Baseline = true)]
    public string Direct() => _person.Name;
    
    [Benchmark]
    public object? ReflectionGetValue() => _nameProp.GetValue(_person);
    
    [Benchmark]
    public object? ReflectionGetValueCached() => _nameProp.GetValue(_person);
}

// Результаты типичные:
// Direct                : 0.5 ns       1.00x
// ReflectionGetValue    : 50-100 ns   100-200x
// CachedDelegate        : 1-2 ns      ~3x
```

> [!warning] `GetProperty` в hot path — катастрофа
> Каждый вызов `t.GetProperty("Name")` — поиск в metadata (~1µs). В hot loop тысячи раз — производительность падает в 1000x. **Кешируй PropertyInfo.**

### Cache strategies

```csharp
// Уровень 1 — кешируем PropertyInfo
public static class PropertyCache<T>
{
    public static readonly PropertyInfo[] Properties = 
        typeof(T).GetProperties(BindingFlags.Public | BindingFlags.Instance);
}

// Уровень 2 — кешируем компилированный delegate (~30x быстрее reflection)
public static class FastAccessor<T, TProp>
{
    private static readonly Dictionary<string, Func<T, TProp>> _getters = new();
    
    public static Func<T, TProp> GetGetter(string propertyName)
    {
        if (_getters.TryGetValue(propertyName, out var cached))
            return cached;
        
        var param = Expression.Parameter(typeof(T), "x");
        var prop = Expression.Property(param, propertyName);
        var lambda = Expression.Lambda<Func<T, TProp>>(prop, param);
        var compiled = lambda.Compile();
        
        _getters[propertyName] = compiled;
        return compiled;
    }
}

// Использование
var getName = FastAccessor<Person, string>.GetGetter("Name");
string name = getName(person);  // ~3 ns vs 50 ns reflection
```

---

## 3. Expression Trees — фундамент

### Что это

Expression Tree — **AST (Abstract Syntax Tree)** в виде объектов. Любое C# выражение можно представить как дерево.

```csharp
// Это lambda — компилируется в IL и работает как делегат
Func<Person, bool> filter1 = p => p.Age > 18;
filter1(person);  // вызов

// Это expression tree — компилятор строит AST из C# кода
Expression<Func<Person, bool>> filter2 = p => p.Age > 18;
// filter2 — это объект BinaryExpression { Left=PropertyAccess, Right=Constant, NodeType=GreaterThan }
```

### Как выглядит дерево

```csharp
Expression<Func<Person, bool>> expr = p => p.Age > 18;

// Структура:
// LambdaExpression
//   Parameters: [ParameterExpression(p)]
//   Body: BinaryExpression(GreaterThan)
//     Left: MemberExpression(p.Age)
//       Expression: ParameterExpression(p)
//       Member: PropertyInfo(Age)
//     Right: ConstantExpression(18)

// Можно прочитать
var binary = (BinaryExpression)expr.Body;
Console.WriteLine(binary.NodeType);            // GreaterThan
Console.WriteLine(binary.Left);                // p.Age
Console.WriteLine(binary.Right);               // 18
Console.WriteLine(((MemberExpression)binary.Left).Member.Name);  // "Age"
```

### Зачем

Expression Tree позволяет **анализировать** код, не выполняя его. Это фундамент:

- **EF Core / LINQ to SQL** — твой `Where(x => x.Age > 18)` парсится → переводится в SQL `WHERE Age > 18`
- **AutoMapper** — анализирует выражения для mapping configurations
- **MongoDB driver** — переводит в MongoDB query
- **MassTransit / NServiceBus** — message routing
- **Specification Pattern** — composable predicates

---

## 4. Создание Expression Trees вручную

```csharp
// Цель: x => x > 5

// 1. Параметр
ParameterExpression param = Expression.Parameter(typeof(int), "x");

// 2. Константа
ConstantExpression five = Expression.Constant(5);

// 3. Бинарное выражение
BinaryExpression body = Expression.GreaterThan(param, five);

// 4. Lambda
Expression<Func<int, bool>> lambda = Expression.Lambda<Func<int, bool>>(body, param);

// 5. Compile (jit-time codegen → almost native speed)
Func<int, bool> compiled = lambda.Compile();

// Использование
bool result = compiled(7);  // true
```

### Build property accessor dynamically

```csharp
// Строим: x => x.PropertyName
public static Func<T, TProp> BuildGetter<T, TProp>(string propertyName)
{
    var param = Expression.Parameter(typeof(T), "x");
    var prop = Expression.Property(param, propertyName);
    var lambda = Expression.Lambda<Func<T, TProp>>(prop, param);
    return lambda.Compile();
}

// Build setter
public static Action<T, TProp> BuildSetter<T, TProp>(string propertyName)
{
    var param = Expression.Parameter(typeof(T), "x");
    var value = Expression.Parameter(typeof(TProp), "value");
    var prop = Expression.Property(param, propertyName);
    var assign = Expression.Assign(prop, value);
    var lambda = Expression.Lambda<Action<T, TProp>>(assign, param, value);
    return lambda.Compile();
}

// Использование
var getName = BuildGetter<Person, string>("Name");
var setName = BuildSetter<Person, string>("Name");

setName(person, "John");
string name = getName(person);
```

### Build constructor invocation

```csharp
// Строим: () => new Person()
public static Func<T> BuildConstructor<T>()
{
    var ctor = typeof(T).GetConstructor(Type.EmptyTypes)
        ?? throw new InvalidOperationException("No default constructor");
    
    var newExpr = Expression.New(ctor);
    var lambda = Expression.Lambda<Func<T>>(newExpr);
    return lambda.Compile();
}

// С параметрами: (string name, int age) => new Person(name, age)
public static Func<string, int, Person> BuildPersonCtor()
{
    var nameParam = Expression.Parameter(typeof(string), "name");
    var ageParam = Expression.Parameter(typeof(int), "age");
    
    var ctor = typeof(Person).GetConstructor(new[] { typeof(string), typeof(int) })!;
    var newExpr = Expression.New(ctor, nameParam, ageParam);
    
    return Expression.Lambda<Func<string, int, Person>>(newExpr, nameParam, ageParam).Compile();
}
```

---

## 5. Анализ Expression Trees — ExpressionVisitor

`ExpressionVisitor` — паттерн для обхода и трансформации деревьев.

```csharp
// Заменяет все обращения к старому свойству на новое
public class PropertyRenamer : ExpressionVisitor
{
    private readonly string _oldName;
    private readonly string _newName;
    
    public PropertyRenamer(string oldName, string newName)
    {
        _oldName = oldName;
        _newName = newName;
    }
    
    protected override Expression VisitMember(MemberExpression node)
    {
        if (node.Member.Name == _oldName)
        {
            var newMember = node.Expression!.Type.GetProperty(_newName);
            if (newMember != null)
                return Expression.Property(node.Expression, newMember);
        }
        return base.VisitMember(node);
    }
}

// Использование
Expression<Func<Person, bool>> original = p => p.OldName == "John";
var renamer = new PropertyRenamer("OldName", "NewName");
var modified = (Expression<Func<Person, bool>>)renamer.Visit(original);
// modified: p => p.NewName == "John"
```

### Real-world: Specification combination

```csharp
// Composable predicates через Expression composition
public static class ExpressionExtensions
{
    public static Expression<Func<T, bool>> AndAlso<T>(
        this Expression<Func<T, bool>> first,
        Expression<Func<T, bool>> second)
    {
        var param = Expression.Parameter(typeof(T), "x");
        
        var leftBody = ReplaceParameter(first.Body, first.Parameters[0], param);
        var rightBody = ReplaceParameter(second.Body, second.Parameters[0], param);
        
        return Expression.Lambda<Func<T, bool>>(
            Expression.AndAlso(leftBody, rightBody),
            param);
    }
    
    private static Expression ReplaceParameter(Expression expr, ParameterExpression oldParam, ParameterExpression newParam)
    {
        return new ParameterReplacer(oldParam, newParam).Visit(expr);
    }
    
    private class ParameterReplacer(ParameterExpression oldParam, ParameterExpression newParam) : ExpressionVisitor
    {
        protected override Expression VisitParameter(ParameterExpression node)
            => node == oldParam ? newParam : node;
    }
}

// Использование — composable specs
Expression<Func<Order, bool>> isActive = o => o.Status == "Active";
Expression<Func<Order, bool>> isHighValue = o => o.Total > 1000;

var combined = isActive.AndAlso(isHighValue);
// o => o.Status == "Active" && o.Total > 1000

// Передаём в EF — он переведёт в SQL
var orders = await context.Orders.Where(combined).ToListAsync();
```

---

## 6. Compiled Expressions vs Direct call vs Reflection

```csharp
[MemoryDiagnoser]
public class AccessorBenchmark
{
    private readonly Person _person = new() { Name = "John" };
    private readonly PropertyInfo _propInfo = typeof(Person).GetProperty("Name")!;
    private static readonly Func<Person, string> _compiledGetter;
    private static readonly Func<Person, object> _delegateGetter;
    
    static AccessorBenchmark()
    {
        // Build compiled expression
        var param = Expression.Parameter(typeof(Person), "p");
        var prop = Expression.Property(param, "Name");
        _compiledGetter = Expression.Lambda<Func<Person, string>>(prop, param).Compile();
        
        // Build delegate from MethodInfo
        var getMethod = typeof(Person).GetProperty("Name")!.GetGetMethod()!;
        _delegateGetter = (Func<Person, object>)Delegate.CreateDelegate(
            typeof(Func<Person, object>), getMethod);
    }
    
    [Benchmark(Baseline = true)] public string Direct() => _person.Name;
    [Benchmark] public object? Reflection() => _propInfo.GetValue(_person);
    [Benchmark] public string CompiledExpr() => _compiledGetter(_person);
    [Benchmark] public object Delegate() => _delegateGetter(_person);
}

// Типичные результаты:
// Direct           : 0.5 ns        1.00x  baseline
// CompiledExpr     : 1.5 ns        3x     отлично
// Delegate         : 2.0 ns        4x     отлично
// Reflection       : 60 ns         120x   плохо для hot path
```

> [!info] Когда что
> - **Direct** — статически известно — всегда лучше
> - **Compiled Expression** — типы известны runtime, но нужна высокая perf — кеш делегата
> - **Reflection** — простой одноразовый доступ, не критичный к perf
> - **Source Generator** — known compile-time → нативная скорость без runtime cost

---

## 7. AutoMapper — как это работает под капотом

Упрощённая копия принципа:

```csharp
public class SimpleMapper
{
    private readonly Dictionary<(Type, Type), Delegate> _maps = new();
    
    public TDestination Map<TSource, TDestination>(TSource source) where TDestination : new()
    {
        var key = (typeof(TSource), typeof(TDestination));
        
        if (!_maps.TryGetValue(key, out var mapper))
        {
            mapper = BuildMapper<TSource, TDestination>();
            _maps[key] = mapper;
        }
        
        return ((Func<TSource, TDestination>)mapper)(source);
    }
    
    private static Func<TSource, TDestination> BuildMapper<TSource, TDestination>() 
        where TDestination : new()
    {
        var sourceParam = Expression.Parameter(typeof(TSource), "src");
        var destVar = Expression.Variable(typeof(TDestination), "dest");
        
        var statements = new List<Expression>
        {
            // var dest = new TDestination();
            Expression.Assign(destVar, Expression.New(typeof(TDestination)))
        };
        
        var sourceProps = typeof(TSource).GetProperties();
        var destProps = typeof(TDestination).GetProperties()
            .Where(p => p.CanWrite)
            .ToDictionary(p => p.Name);
        
        foreach (var srcProp in sourceProps)
        {
            if (destProps.TryGetValue(srcProp.Name, out var destProp) &&
                destProp.PropertyType == srcProp.PropertyType)
            {
                // dest.PropertyX = src.PropertyX
                statements.Add(Expression.Assign(
                    Expression.Property(destVar, destProp),
                    Expression.Property(sourceParam, srcProp)));
            }
        }
        
        // return dest;
        statements.Add(destVar);
        
        var body = Expression.Block(
            new[] { destVar },
            statements);
        
        return Expression.Lambda<Func<TSource, TDestination>>(body, sourceParam).Compile();
    }
}

// Использование
var mapper = new SimpleMapper();
var dto = mapper.Map<Person, PersonDto>(person);
// Первый вызов: build expression + compile (~1ms)
// Последующие: ~5 ns (как ручной mapper!)
```

> [!info] Mapperly — современная альтернатива
> [Mapperly](https://mapperly.riok.app/) использует **Source Generators** вместо expression trees → код генерируется в compile-time, нет runtime cost вообще, AOT-friendly. Рекомендуется для новых проектов.

---

## 8. EF Core — translation в SQL

EF Core берёт твой LINQ Expression Tree и переводит в SQL через `IQueryable<T>` provider.

```csharp
// Твой код
var query = context.Orders
    .Where(o => o.Total > 100 && o.Status == "Active")
    .OrderByDescending(o => o.CreatedAt)
    .Select(o => new { o.Id, o.Total });

// Что происходит:
// 1. context.Orders — это IQueryable<Order>, имеет Expression и Provider
// 2. .Where(...) — НЕ выполняет lambda, а ОБОРАЧИВАЕТ её в expression tree
//    query.Expression теперь содержит: Orders.Where(o => o.Total > 100 && ...)
// 3. .ToListAsync() — Provider обходит дерево и переводит в SQL:
//    SELECT Id, Total FROM Orders WHERE Total > 100 AND Status = 'Active'
//    ORDER BY CreatedAt DESC

await query.ToListAsync();
```

### Почему `Func<T, bool>` не работает в EF

```csharp
// ❌ Func — это уже скомпилированный делегат, не дерево
Func<Order, bool> filter = o => o.Total > 100;
context.Orders.Where(filter);  // Где Where принимает Expression<Func<Order, bool>>!
// EF не может прочитать делегат → in-memory eval (загрузит ВСЕ orders, потом фильтр в памяти!)

// ✅ Expression — это дерево, EF может прочитать
Expression<Func<Order, bool>> filter = o => o.Total > 100;
context.Orders.Where(filter);  // Корректно, переводится в SQL
```

### Client vs Server evaluation

```csharp
// EF Core 3+ — kicks ⚠️ exception если не может перевести
var orders = await context.Orders
    .Where(o => MyCustomMethod(o.Status))  // throws "could not translate"
    .ToListAsync();

// ✅ Решение: загрузить и фильтровать в памяти явно
var orders = (await context.Orders.ToListAsync())
    .Where(o => MyCustomMethod(o.Status))
    .ToList();
```

См. [EF Core Basics & Tracking](../EFCore/basics-tracking.md).

---

## 9. Dynamic — DLR (Dynamic Language Runtime)

```csharp
dynamic obj = new ExpandoObject();
obj.Name = "John";       // компилируется в DLR call site
obj.Age = 30;
Console.WriteLine(obj.Name);

// Под капотом — call site cache
// Первый вызов: DLR определяет тип и метод (медленно ~1µs)
// Последующие: cached call (~50 ns)
```

### Использование

```csharp
// JSON parsing без типизации
dynamic data = JsonConvert.DeserializeObject(jsonString);
string name = data.user.name;  // not type-safe!

// COM Interop
dynamic excel = Activator.CreateInstance(Type.GetTypeFromProgID("Excel.Application")!);
excel.Workbooks.Add();
excel.Quit();

// IronPython integration
dynamic pythonModule = scriptEngine.Runtime.ImportModule("my_python_module");
pythonModule.do_something(42);
```

> [!warning] Dynamic — пожиратель type safety
> Все ошибки → runtime. Используй только когда **действительно** нужна динамика (COM, scripting). Для JSON — `JsonDocument` / `JsonNode` лучше.

---

## 10. IL Emit — DynamicMethod (advanced)

Для **максимальной** производительности — генерация IL вручную.

```csharp
// Создаём метод динамически
var method = new DynamicMethod(
    name: "AddInts",
    returnType: typeof(int),
    parameterTypes: new[] { typeof(int), typeof(int) });

ILGenerator il = method.GetILGenerator();
il.Emit(OpCodes.Ldarg_0);   // Load arg 0 onto stack
il.Emit(OpCodes.Ldarg_1);   // Load arg 1 onto stack
il.Emit(OpCodes.Add);        // Pop two, push sum
il.Emit(OpCodes.Ret);        // Return top of stack

// Compile to delegate
var del = (Func<int, int, int>)method.CreateDelegate(typeof(Func<int, int, int>));
int result = del(2, 3);  // 5

// Производительность: native speed (как обычный method call)
```

### Когда нужно

- Сериализаторы maximum speed (например, `MessagePack-CSharp`)
- ORM internals (legacy NHibernate)
- Mock libraries (Castle.DynamicProxy)

> [!info] В .NET 6+ — Source Generators предпочтительнее
> Source Generators дают тот же результат (нативный код), но проще поддерживать, видны в debugger, AOT-compatible. IL emit оставляем для extreme cases.

---

## 11. Source Generators — modern alternative

Compile-time альтернатива reflection:

```csharp
// Вместо runtime reflection
var props = typeof(Person).GetProperties();  // runtime, slow

// Используй generator
[GenerateProperties]
public partial class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}

// Generator создаёт в compile-time:
public partial class Person
{
    public static readonly string[] PropertyNames = { "Name", "Age" };
    public static Person Create(IDataReader reader) { /* generated code */ }
}
```

### Реальные примеры (в .NET 8+)

| Source Generator | Что заменяет |
|------------------|--------------|
| `System.Text.Json` source generator | runtime JSON reflection |
| `[GeneratedRegex]` | `new Regex(pattern)` (lazy compile) |
| `[LoggerMessage]` | structured logging без allocation |
| `[GeneratedComInterface]` (.NET 8) | runtime COM Interop reflection |
| Mapperly | AutoMapper expression-tree compilation |
| MediatR.SourceGenerator | runtime handler discovery |

См. подробно [Source Generators](source-generators.md).

---

## 12. Reflection и Native AOT — ловушки

Native AOT не поддерживает большую часть reflection:

```csharp
// ❌ В Native AOT — IL2026 / IL3050 warning
var props = typeof(Person).GetProperties();
var obj = Activator.CreateInstance(typeof(MyClass));

// ✅ Помечать что reflection нужен
[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicProperties)]
public class Person { ... }

// ✅ Suppress если не используется в AOT
[RequiresUnreferencedCode("Uses reflection")]
public void DoSomething() { ... }

// ✅ Better — Source Generators вместо reflection
```

См. [Native AOT](../AspNetCore/native-aot.md).

---

## 13. Attributes — read & process

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class ColumnAttribute(string name) : Attribute
{
    public string Name { get; } = name;
}

public class User
{
    [Column("user_id")]
    public Guid Id { get; set; }
    
    [Column("user_name")]
    public string Name { get; set; } = "";
}

// Reading
foreach (var prop in typeof(User).GetProperties())
{
    var col = prop.GetCustomAttribute<ColumnAttribute>();
    Console.WriteLine($"{prop.Name} → {col?.Name ?? prop.Name}");
}

// EF Core, Dapper, JSON serializers — все используют этот паттерн
```

### Caller info attributes (compile-time, не reflection!)

```csharp
public void Log(
    string message,
    [CallerMemberName] string member = "",
    [CallerFilePath] string file = "",
    [CallerLineNumber] int line = 0)
{
    Console.WriteLine($"{file}:{line} [{member}] {message}");
}

// Вызов
Log("Hello");  // компилятор подставляет member, file, line на этапе compile
```

---

## 14. Common Pitfalls

### 1. Кеш PropertyInfo сразу при загрузке

```csharp
// ❌ Каждый вызов — поиск в metadata
public string GetName(object obj) => 
    obj.GetType().GetProperty("Name")!.GetValue(obj)!.ToString()!;

// ✅ Кешировать
private static readonly Dictionary<Type, PropertyInfo> _cache = new();

public string GetName(object obj)
{
    var type = obj.GetType();
    if (!_cache.TryGetValue(type, out var prop))
    {
        prop = type.GetProperty("Name")!;
        _cache[type] = prop;
    }
    return prop.GetValue(obj)!.ToString()!;
}
```

### 2. Использование `Func<T, bool>` где нужен `Expression<Func<T, bool>>`

```csharp
// ❌ EF загружает все Orders, фильтрует в памяти!
public async Task<List<Order>> GetByCondition(Func<Order, bool> condition)
    => (await context.Orders.ToListAsync()).Where(condition).ToList();

// ✅
public async Task<List<Order>> GetByCondition(Expression<Func<Order, bool>> condition)
    => await context.Orders.Where(condition).ToListAsync();
```

### 3. Reflection в hot path без кеша

Вызов `GetProperty` или `Activator.CreateInstance` миллион раз в секунду — производительность падает в 1000x.

### 4. `Type.GetType("...")` без assembly qualified name

```csharp
// ❌ Type.GetType ищет только в текущей assembly + mscorlib
Type? t = Type.GetType("MyApp.Person");  // null если не в текущей assembly

// ✅ Assembly qualified name
Type? t = Type.GetType("MyApp.Person, MyApp.Domain");

// ✅ Или из конкретной assembly
var asm = Assembly.LoadFrom("MyApp.Domain.dll");
Type? t = asm.GetType("MyApp.Person");
```

### 5. Compiled expression создаётся в каждом вызове

```csharp
// ❌ Compile при каждом вызове
public int Get(int x)
{
    var param = Expression.Parameter(typeof(int));
    var body = Expression.Multiply(param, Expression.Constant(2));
    var lambda = Expression.Lambda<Func<int, int>>(body, param);
    return lambda.Compile()(x);  // ~10ms!
}

// ✅ Один compile, потом use
private static readonly Func<int, int> _compiled;
static Class()
{
    var param = Expression.Parameter(typeof(int));
    var body = Expression.Multiply(param, Expression.Constant(2));
    _compiled = Expression.Lambda<Func<int, int>>(body, param).Compile();
}
public int Get(int x) => _compiled(x);  // ~1 ns
```

### 6. Loading assemblies через `Assembly.LoadFrom` без isolation

```csharp
// ❌ Plugin assembly загружен в default context — нельзя выгрузить
Assembly plugin = Assembly.LoadFrom("Plugin.dll");

// ✅ Изолированный AssemblyLoadContext — можно Unload
public class PluginLoadContext : AssemblyLoadContext
{
    public PluginLoadContext() : base(isCollectible: true) { }
}

var ctx = new PluginLoadContext();
Assembly plugin = ctx.LoadFromAssemblyPath("Plugin.dll");
// ... использование ...
ctx.Unload();  // освобождает плагин
```

### 7. Reflection приводит к security holes

`InvokeMember` с user-controlled input → arbitrary code execution. Никогда не позволяй пользователю выбирать тип/метод.

### 8. Generic constraint обходится через reflection

```csharp
// Generic constraint
public T DoSomething<T>(T x) where T : class { ... }

// Через reflection — constraint не проверяется в runtime!
var method = typeof(MyClass).GetMethod("DoSomething")!.MakeGenericMethod(typeof(int));
method.Invoke(obj, new object[] { 5 });
// Бывает throw, бывает странное поведение
```

---

## 15. Best Practices

- **Кеш Type / PropertyInfo / MethodInfo** — никогда не вызывай GetX в hot path
- **Compiled Expression > Reflection** — для повторяемых вызовов
- **Source Generators > Reflection** — для compile-time known scenarios
- **Expression\<Func\<T, bool\>\> в публичных API** — для composability с EF/IQueryable
- **DynamicallyAccessedMembers attribute** — для AOT-compat кода
- **AssemblyLoadContext** для plugins — позволяет Unload
- **Не используй dynamic** если можно типизировано
- **Атрибуты для metadata** — стандартный pattern
- **`nameof(X)` вместо `"X"`** — refactor-safe
- **PerfView / dotMemory** — найти reflection hotspots в production

---

## См. также

- [Source Generators](source-generators.md) — modern alternative
- [Modern C# Features](modern-features.md) — pattern matching, records
- [Native AOT](../AspNetCore/native-aot.md) — что ломается без reflection
- [EF Core Basics](../EFCore/basics-tracking.md) — IQueryable + Expression Trees
- [Compilation/JIT](../Runtime/compilation-jit.md) — как expression compile работает

## Reading list

- **Microsoft Docs — Reflection** — learn.microsoft.com/dotnet/csharp/programming-guide/concepts/reflection
- **Microsoft Docs — Expression Trees** — learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees
- **Joel Pobar — Reflection performance** (классика) — learn.microsoft.com/archive/msdn-magazine
- **Matt Warren — How does Expression Tree get compiled** — mattwarren.org/2017/03/30/Performance-Implications-of-Boxing-in-Expression-Trees
- **Stephen Toub — DynamicMethod blog** — devblogs.microsoft.com/dotnet
- **Mapperly** — mapperly.riok.app
- **Roslyn API book** — Roslyn syntax tree (родственник Expression tree но для C# code)
- **CLR via C# (Jeffrey Richter)** — главы про reflection и AppDomain (legacy AppDomain → AssemblyLoadContext в .NET Core)
