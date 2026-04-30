---
tags: [csharp, design-patterns, gof, command, visitor, composite, proxy, memento, bridge, flyweight, prototype, senior]
level: Senior
date: 2026-04-30
---

# GoF Patterns Extended — 8 дополнительных паттернов

> **Дополнение к `design-patterns.md`**: Command, Visitor, Composite, Proxy, Memento, Bridge, Flyweight, Prototype. С качественными case studies — где, когда и почему именно этот паттерн.

[[design-patterns|Основные 13 паттернов]] (Strategy, Factory, Decorator, Observer, Builder, Singleton, Chain of Responsibility, Template Method, Adapter, Facade, State, Specification, Null Object) уже покрыты отдельно.

---

## Что это, зачем и когда

### Зачем эти 8 ещё знать

Все 23 GoF паттерна — это **словарь** для коммуникации между разработчиками. Senior должен **узнавать** паттерны при чтении чужого кода и **применять** правильный для задачи.

| Паттерн | Главная идея | Когда применять |
|---------|--------------|------------------|
| **Command** | Действие как объект (с undo) | Undo/redo, queueing, scheduling |
| **Visitor** | Операция над иерархией объектов | AST processing, ORM mapping |
| **Composite** | Tree из единого интерфейса | Filesystem, UI, menus |
| **Proxy** | Заместитель с дополнительной логикой | Lazy loading, caching, security |
| **Memento** | Snapshot состояния для restore | Undo, save/load, time travel |
| **Bridge** | Разделить abstraction и implementation | Cross-platform, multi-DB drivers |
| **Flyweight** | Sharing для memory savings | Глифы шрифта, particle systems |
| **Prototype** | Cloning без знания типа | Game spawning, dynamic templates |

См. [[design-patterns|Design Patterns]] для основных 13.

---

## 1. Command — действие как объект

### Идея

Инкапсулировать **действие** в объект — вместе с параметрами и контекстом. Позволяет:
- Queueing команд
- Logging / undo / redo
- Передачу как параметра
- Scheduling

### Базовая реализация

```csharp
public interface ICommand
{
    void Execute();
    void Undo();
}

public class MoveCommand : ICommand
{
    private readonly GameObject _obj;
    private readonly Vector3 _delta;
    private Vector3 _previousPosition;
    
    public MoveCommand(GameObject obj, Vector3 delta)
    {
        _obj = obj;
        _delta = delta;
    }
    
    public void Execute()
    {
        _previousPosition = _obj.Position;
        _obj.Position += _delta;
    }
    
    public void Undo()
    {
        _obj.Position = _previousPosition;
    }
}

public class CommandHistory
{
    private readonly Stack<ICommand> _undoStack = new();
    private readonly Stack<ICommand> _redoStack = new();
    
    public void Execute(ICommand cmd)
    {
        cmd.Execute();
        _undoStack.Push(cmd);
        _redoStack.Clear();  // новый action invalidates redo
    }
    
    public void Undo()
    {
        if (_undoStack.TryPop(out var cmd))
        {
            cmd.Undo();
            _redoStack.Push(cmd);
        }
    }
    
    public void Redo()
    {
        if (_redoStack.TryPop(out var cmd))
        {
            cmd.Execute();
            _undoStack.Push(cmd);
        }
    }
}
```

### Case Study: Графический редактор (Photoshop-like)

```csharp
// User actions:
// 1. Brush stroke
// 2. Apply filter
// 3. Resize
// → Все нужны с undo/redo!

public abstract class EditCommand : ICommand
{
    public abstract void Execute();
    public abstract void Undo();
}

public class BrushStrokeCommand : EditCommand
{
    private readonly Image _image;
    private readonly Stroke _stroke;
    private byte[]? _pixelsBackup;
    
    public BrushStrokeCommand(Image image, Stroke stroke)
    {
        _image = image;
        _stroke = stroke;
    }
    
    public override void Execute()
    {
        _pixelsBackup = _image.GetPixelsInArea(_stroke.BoundingBox);
        _image.ApplyStroke(_stroke);
    }
    
    public override void Undo()
    {
        _image.RestorePixels(_stroke.BoundingBox, _pixelsBackup);
    }
}

public class ApplyFilterCommand : EditCommand { /* ... */ }
public class ResizeCommand : EditCommand { /* ... */ }

// Editor
public class ImageEditor
{
    private readonly CommandHistory _history = new();
    private readonly Image _image;
    
    public void Brush(Stroke s) => _history.Execute(new BrushStrokeCommand(_image, s));
    public void Filter(IFilter f) => _history.Execute(new ApplyFilterCommand(_image, f));
    public void Resize(Size s) => _history.Execute(new ResizeCommand(_image, s));
    
    // Ctrl+Z / Ctrl+Y
    public void Undo() => _history.Undo();
    public void Redo() => _history.Redo();
}
```

### Case Study: Job queueing (background processing)

```csharp
// Команды как queueable jobs
public abstract class JobCommand : ICommand
{
    public string Id { get; } = Guid.NewGuid().ToString();
    public DateTime QueuedAt { get; } = DateTime.UtcNow;
    public abstract Task ExecuteAsync(CancellationToken ct);
}

public class SendEmailCommand : JobCommand
{
    public string To { get; init; }
    public string Subject { get; init; }
    public string Body { get; init; }
    
    public override async Task ExecuteAsync(CancellationToken ct)
    {
        await EmailService.SendAsync(To, Subject, Body, ct);
    }
}

public class ProcessImageCommand : JobCommand
{
    public string ImageId { get; init; }
    public override async Task ExecuteAsync(CancellationToken ct) { /* ... */ }
}

// Queue + worker
public class JobQueue
{
    private readonly Channel<JobCommand> _channel = Channel.CreateUnbounded<JobCommand>();
    
    public ValueTask EnqueueAsync(JobCommand cmd) => _channel.Writer.WriteAsync(cmd);
    
    public async Task ProcessAsync(CancellationToken ct)
    {
        await foreach (var cmd in _channel.Reader.ReadAllAsync(ct))
        {
            try { await cmd.ExecuteAsync(ct); }
            catch (Exception ex) { Logger.LogError(ex, "Job {Id} failed", cmd.Id); }
        }
    }
}
```

### Когда применять Command

✅ **Используй когда:**
- Undo/redo функциональность
- Queueing / scheduling операций
- Macro recording (последовательность действий)
- Transactional behavior (всё или ничего)
- Logging / audit изменений

❌ **НЕ используй когда:**
- Простые операции без необходимости encapsulation
- Каждый раз создавать класс — overkill для одноразовых

### .NET аналоги

- `MediatR.IRequest` — Command pattern для CQRS
- `IHostedService` — long-running commands
- `Channel<T>` — для queueing команд
- `Action`, `Func` — упрощённые commands (без undo)

См. [[../Architecture/cqrs-mediatr|CQRS & MediatR]].

---

## 2. Visitor — операция над иерархией

### Идея

Отделить **алгоритм** от **структуры данных**. Когда у тебя иерархия классов и нужно добавить операцию — без изменения классов.

```
Без Visitor:                 С Visitor:
─────────────────           ─────────────────
Каждый класс имеет           Класс имеет Accept(visitor)
много методов:                Visitor содержит логику:

class Circle {                class Circle {
  void Render() {}              void Accept(IVisitor v) => v.Visit(this);
  void Save() {}              }
  void Validate() {}          
  // 10+ методов              class RenderVisitor : IVisitor {
}                                void Visit(Circle c) {}
                                void Visit(Square s) {}
                              }
```

### Базовая реализация

```csharp
// Иерархия
public abstract class Shape
{
    public abstract void Accept(IShapeVisitor visitor);
}

public class Circle : Shape
{
    public double Radius { get; set; }
    public override void Accept(IShapeVisitor v) => v.Visit(this);
}

public class Rectangle : Shape
{
    public double Width { get; set; }
    public double Height { get; set; }
    public override void Accept(IShapeVisitor v) => v.Visit(this);
}

public class Triangle : Shape
{
    public double Base { get; set; }
    public double Height { get; set; }
    public override void Accept(IShapeVisitor v) => v.Visit(this);
}

// Visitor
public interface IShapeVisitor
{
    void Visit(Circle c);
    void Visit(Rectangle r);
    void Visit(Triangle t);
}

// Конкретные visitors
public class AreaCalculator : IShapeVisitor
{
    public double Total { get; private set; }
    
    public void Visit(Circle c) => Total += Math.PI * c.Radius * c.Radius;
    public void Visit(Rectangle r) => Total += r.Width * r.Height;
    public void Visit(Triangle t) => Total += 0.5 * t.Base * t.Height;
}

public class PerimeterCalculator : IShapeVisitor
{
    public double Total { get; private set; }
    
    public void Visit(Circle c) => Total += 2 * Math.PI * c.Radius;
    public void Visit(Rectangle r) => Total += 2 * (r.Width + r.Height);
    public void Visit(Triangle t) => Total += t.Base + 2 * Math.Sqrt(Math.Pow(t.Height, 2) + Math.Pow(t.Base/2, 2));
}

// Use
var shapes = new List<Shape> { new Circle { Radius = 5 }, new Rectangle { Width = 3, Height = 4 } };
var areaCalc = new AreaCalculator();
foreach (var s in shapes) s.Accept(areaCalc);
Console.WriteLine($"Total area: {areaCalc.Total}");
```

### Case Study: Compiler / AST processing

```csharp
// AST для simple expression language: 1 + 2 * 3
public abstract class AstNode
{
    public abstract T Accept<T>(IAstVisitor<T> visitor);
}

public class NumberNode : AstNode
{
    public double Value { get; set; }
    public override T Accept<T>(IAstVisitor<T> v) => v.Visit(this);
}

public class BinaryOpNode : AstNode
{
    public AstNode Left { get; set; }
    public AstNode Right { get; set; }
    public BinaryOperator Op { get; set; }
    public override T Accept<T>(IAstVisitor<T> v) => v.Visit(this);
}

public class VariableNode : AstNode
{
    public string Name { get; set; }
    public override T Accept<T>(IAstVisitor<T> v) => v.Visit(this);
}

public interface IAstVisitor<T>
{
    T Visit(NumberNode node);
    T Visit(BinaryOpNode node);
    T Visit(VariableNode node);
}

// Visitor 1: Evaluator
public class Evaluator : IAstVisitor<double>
{
    private readonly Dictionary<string, double> _variables;
    public Evaluator(Dictionary<string, double> vars) => _variables = vars;
    
    public double Visit(NumberNode n) => n.Value;
    public double Visit(VariableNode v) => _variables[v.Name];
    public double Visit(BinaryOpNode b)
    {
        double l = b.Left.Accept(this);
        double r = b.Right.Accept(this);
        return b.Op switch
        {
            BinaryOperator.Add => l + r,
            BinaryOperator.Sub => l - r,
            BinaryOperator.Mul => l * r,
            BinaryOperator.Div => l / r,
            _ => throw new NotSupportedException()
        };
    }
}

// Visitor 2: ToString (formatter)
public class Printer : IAstVisitor<string>
{
    public string Visit(NumberNode n) => n.Value.ToString();
    public string Visit(VariableNode v) => v.Name;
    public string Visit(BinaryOpNode b) => 
        $"({b.Left.Accept(this)} {OpToStr(b.Op)} {b.Right.Accept(this)})";
}

// Visitor 3: Optimizer (constant folding)
public class Optimizer : IAstVisitor<AstNode>
{
    public AstNode Visit(NumberNode n) => n;
    public AstNode Visit(VariableNode v) => v;
    public AstNode Visit(BinaryOpNode b)
    {
        var left = b.Left.Accept(this);
        var right = b.Right.Accept(this);
        
        // Если оба числа — вычислить compile-time
        if (left is NumberNode ln && right is NumberNode rn)
        {
            return new NumberNode { Value = Eval(ln.Value, b.Op, rn.Value) };
        }
        
        return new BinaryOpNode { Left = left, Right = right, Op = b.Op };
    }
}

// Use
AstNode tree = Parser.Parse("1 + 2 * 3");
var eval = tree.Accept(new Evaluator(new Dictionary<string, double>()));  // 7
var pretty = tree.Accept(new Printer());                                   // "(1 + (2 * 3))"
var optimized = tree.Accept(new Optimizer());                              // NumberNode(7)
```

### Когда применять Visitor

✅ **Используй когда:**
- Иерархия классов **стабильна**, а **операций много**
- Compilers / interpreters (AST traversal)
- ORM — mapping entities → SQL / DTOs
- Document processing (Word, PDF AST)

❌ **НЕ используй когда:**
- Иерархия часто меняется (новые типы → updates всех visitors)
- Операций мало
- Pattern matching (C# 8+) проще

### Modern alternative — switch expression

```csharp
public double Area(Shape s) => s switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Rectangle r => r.Width * r.Height,
    Triangle t => 0.5 * t.Base * t.Height,
    _ => throw new NotSupportedException()
};

// Проще для маленьких иерархий, но requires modify при добавлении type
```

См. [[modern-features|Pattern Matching]] и [[functional-csharp|Discriminated Unions]].

---

## 3. Composite — tree через единый интерфейс

### Идея

Treat **individual objects** и **compositions of objects** одинаково — через единый интерфейс. Создание древовидной структуры.

```
Без Composite:                С Composite:
─────────────────             ─────────────────
File / Folder — разные       FileSystemItem — общий интерфейс
методы для каждого             File и Folder его реализуют
                                Folder содержит список FileSystemItem
                                Recursion работает естественно
```

### Базовая реализация

```csharp
public abstract class FileSystemItem
{
    public string Name { get; }
    public DateTime CreatedAt { get; }
    
    protected FileSystemItem(string name)
    {
        Name = name;
        CreatedAt = DateTime.UtcNow;
    }
    
    public abstract long GetSize();
    public abstract void Print(int indent = 0);
}

// Leaf
public class File : FileSystemItem
{
    public long SizeBytes { get; }
    
    public File(string name, long size) : base(name) => SizeBytes = size;
    
    public override long GetSize() => SizeBytes;
    
    public override void Print(int indent)
    {
        Console.WriteLine($"{new string(' ', indent)}📄 {Name} ({SizeBytes} B)");
    }
}

// Composite
public class Folder : FileSystemItem
{
    private readonly List<FileSystemItem> _children = new();
    
    public Folder(string name) : base(name) { }
    
    public void Add(FileSystemItem item) => _children.Add(item);
    public void Remove(FileSystemItem item) => _children.Remove(item);
    
    public override long GetSize() => _children.Sum(c => c.GetSize());
    
    public override void Print(int indent)
    {
        Console.WriteLine($"{new string(' ', indent)}📁 {Name}/");
        foreach (var child in _children)
            child.Print(indent + 2);
    }
}

// Use
var root = new Folder("project");
var src = new Folder("src");
src.Add(new File("Program.cs", 1024));
src.Add(new File("Startup.cs", 2048));
root.Add(src);
root.Add(new File("README.md", 512));

Console.WriteLine($"Total size: {root.GetSize()} bytes");
root.Print();

// Output:
// Total size: 3584 bytes
// 📁 project/
//   📁 src/
//     📄 Program.cs (1024 B)
//     📄 Startup.cs (2048 B)
//   📄 README.md (512 B)
```

### Case Study: UI элементы

```csharp
public abstract class UIElement
{
    public abstract void Render();
    public abstract Size Measure();
}

// Leaf
public class Button : UIElement
{
    public string Text { get; set; }
    public override void Render() => Console.WriteLine($"[{Text}]");
    public override Size Measure() => new(Text.Length * 10, 30);
}

public class TextBox : UIElement
{
    public string Value { get; set; }
    public override void Render() => Console.WriteLine($"|{Value}|");
    public override Size Measure() => new(200, 20);
}

// Composite
public class Panel : UIElement
{
    private readonly List<UIElement> _children = new();
    public string Layout { get; set; } = "vertical";
    
    public void Add(UIElement child) => _children.Add(child);
    
    public override void Render()
    {
        Console.WriteLine($"--- Panel ({Layout}) ---");
        foreach (var c in _children) c.Render();
        Console.WriteLine($"--- /Panel ---");
    }
    
    public override Size Measure()
    {
        // Compose sizes in some logical way
        var sizes = _children.Select(c => c.Measure()).ToList();
        return Layout == "vertical"
            ? new Size(sizes.Max(s => s.Width), sizes.Sum(s => s.Height))
            : new Size(sizes.Sum(s => s.Width), sizes.Max(s => s.Height));
    }
}

// Use
var loginPanel = new Panel { Layout = "vertical" };
loginPanel.Add(new TextBox { Value = "Username" });
loginPanel.Add(new TextBox { Value = "" });  // password
loginPanel.Add(new Button { Text = "Login" });

loginPanel.Render();  // recursively renders all
var totalSize = loginPanel.Measure();  // recursively measures
```

### Case Study: Меню навигации (как в [[../Architecture/real-world-scenarios|real-world-scenarios]])

```csharp
public abstract class MenuItem
{
    public string Label { get; }
    public abstract bool IsAccessible(User user);
}

public class MenuLink : MenuItem
{
    public string Url { get; }
    public IReadOnlyList<string> RequiredRoles { get; }
    
    public override bool IsAccessible(User u) =>
        RequiredRoles.Count == 0 || RequiredRoles.Any(u.IsInRole);
}

public class MenuFolder : MenuItem
{
    private readonly List<MenuItem> _children = new();
    
    public void Add(MenuItem item) => _children.Add(item);
    
    public override bool IsAccessible(User u) =>
        _children.Any(c => c.IsAccessible(u));  // папка видна если хоть один child accessible
    
    public IEnumerable<MenuItem> AccessibleChildren(User u) =>
        _children.Where(c => c.IsAccessible(u));
}

// Recursive filter естественно ложится
```

### Когда применять Composite

✅ **Используй когда:**
- Tree-like структура данных
- Клиенты должны treat individual и composite одинаково
- Filesystems, DOM, organization charts, AST

❌ **НЕ используй когда:**
- Flat коллекция объектов
- Разные типы должны иметь существенно разные API

### .NET примеры

- `XElement` (System.Xml.Linq) — Composite для XML
- WPF/WinUI visual tree
- ASP.NET Core middleware composition
- Roslyn syntax tree

---

## 4. Proxy — заместитель с дополнительной логикой

### Идея

Surrogate / placeholder для другого объекта. Контролирует доступ, добавляя:
- **Lazy loading** — load on first use
- **Caching** — cache results
- **Security** — check permissions
- **Logging** — log access
- **Remote** — call across network

### Типы Proxy

```
Virtual Proxy   → lazy initialization
Caching Proxy   → cache results
Protection Proxy → access control
Remote Proxy    → network call (RPC)
Smart Proxy    → counting refs, etc.
```

### Базовая реализация — Virtual (Lazy)

```csharp
public interface IImage
{
    void Display();
}

public class HighResImage : IImage
{
    private readonly string _filename;
    private byte[] _data;  // 50 MB!
    
    public HighResImage(string filename)
    {
        _filename = filename;
        _data = LoadFromDisk(filename);  // дорого!
        Console.WriteLine($"Loaded {filename}");
    }
    
    public void Display() => Console.WriteLine($"Displaying {_filename}");
    
    private byte[] LoadFromDisk(string fn) { /* ... 50 MB load ... */ return new byte[0]; }
}

// Proxy — lazy loading
public class HighResImageProxy : IImage
{
    private readonly string _filename;
    private HighResImage? _real;
    
    public HighResImageProxy(string filename) => _filename = filename;
    
    public void Display()
    {
        _real ??= new HighResImage(_filename);  // load только при первом display
        _real.Display();
    }
}

// Use
var images = new List<IImage>();
for (int i = 0; i < 1000; i++)
    images.Add(new HighResImageProxy($"img_{i}.jpg"));
// 1000 объектов созданы, но **0 загружено**

images[5].Display();  // только этот загружен
```

### Case Study: Caching Proxy для API

```csharp
public interface IUserService
{
    Task<User?> GetByIdAsync(int id);
    Task<List<User>> GetAllAsync();
}

public class UserService : IUserService
{
    private readonly HttpClient _http;
    
    public async Task<User?> GetByIdAsync(int id)
    {
        var resp = await _http.GetAsync($"/api/users/{id}");
        return await resp.Content.ReadFromJsonAsync<User>();
    }
    // ...
}

// Caching Proxy
public class CachingUserServiceProxy : IUserService
{
    private readonly IUserService _inner;
    private readonly IMemoryCache _cache;
    private readonly TimeSpan _ttl = TimeSpan.FromMinutes(5);
    
    public CachingUserServiceProxy(IUserService inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }
    
    public async Task<User?> GetByIdAsync(int id)
    {
        return await _cache.GetOrCreateAsync($"user:{id}", async entry =>
        {
            entry.SlidingExpiration = _ttl;
            return await _inner.GetByIdAsync(id);
        });
    }
    
    public async Task<List<User>> GetAllAsync()
    {
        return await _cache.GetOrCreateAsync("users:all", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = _ttl;
            return await _inner.GetAllAsync();
        }) ?? new();
    }
}

// DI registration
services.AddScoped<UserService>();
services.AddScoped<IUserService>(sp =>
    new CachingUserServiceProxy(
        sp.GetRequiredService<UserService>(),
        sp.GetRequiredService<IMemoryCache>()));

// Caller получает Proxy, прозрачно используя cache
public class HomeController(IUserService users) : Controller { /* ... */ }
```

### Case Study: Protection Proxy (security)

```csharp
public class SecuredDocumentProxy : IDocument
{
    private readonly IDocument _inner;
    private readonly ICurrentUser _user;
    
    public string Read()
    {
        if (!_user.HasPermission("doc.read"))
            throw new UnauthorizedAccessException();
        return _inner.Read();
    }
    
    public void Write(string content)
    {
        if (!_user.HasPermission("doc.write"))
            throw new UnauthorizedAccessException();
        _inner.Write(content);
    }
}
```

### Когда применять Proxy

✅ **Используй когда:**
- Lazy initialization expensive объектов
- Caching expensive operations
- Access control / authorization
- Logging / metrics без изменения original
- Remote object access (RPC)
- Reference counting, smart pointers

❌ **НЕ используй когда:**
- Logic простая (используй Decorator если просто wrapping)
- Контроль не нужен — просто inject dependency directly

### Proxy vs Decorator

| | Proxy | Decorator |
|--|-------|-----------|
| Goal | Контроль доступа к объекту | Добавить behavior |
| Typically | Manages lifecycle (lazy/cache/access) | Adds responsibilities |
| Pattern | Has-a (proxy hides target) | Wraps and extends |

В коде часто **выглядят одинаково** — оба implement common interface и forward calls. Различие в **намерении**.

### .NET примеры

- EF Core — lazy loading через proxy types (Castle.Proxy)
- WCF — Remote Proxy (auto-generated client)
- Castle DynamicProxy — для AOP scenarios
- Mock libraries (Moq) — создают proxies для tests

---

## 5. Memento — snapshot для undo

### Идея

Сохранить **состояние** объекта чтобы потом **восстановить**. Без exposing внутреннего состояния (encapsulation сохраняется).

### Базовая реализация

```csharp
// Originator — объект, чьё состояние сохраняем
public class TextEditor
{
    private string _text = "";
    private int _cursor = 0;
    
    public void Type(string s)
    {
        _text = _text.Insert(_cursor, s);
        _cursor += s.Length;
    }
    
    public void Backspace()
    {
        if (_cursor > 0)
        {
            _text = _text.Remove(_cursor - 1, 1);
            _cursor--;
        }
    }
    
    public string GetText() => _text;
    
    // Создание Memento
    public Memento CreateMemento() => new Memento(_text, _cursor);
    
    // Восстановление
    public void RestoreFromMemento(Memento m)
    {
        _text = m.Text;
        _cursor = m.Cursor;
    }
    
    // Memento — immutable snapshot, доступ только через Editor
    public class Memento
    {
        public string Text { get; }
        public int Cursor { get; }
        
        internal Memento(string text, int cursor)
        {
            Text = text;
            Cursor = cursor;
        }
    }
}

// Caretaker — хранит mementos, не зная их внутреннего устройства
public class History
{
    private readonly Stack<TextEditor.Memento> _undoStack = new();
    private readonly TextEditor _editor;
    
    public History(TextEditor editor) => _editor = editor;
    
    public void Save() => _undoStack.Push(_editor.CreateMemento());
    
    public void Undo()
    {
        if (_undoStack.TryPop(out var m))
            _editor.RestoreFromMemento(m);
    }
}

// Use
var editor = new TextEditor();
var history = new History(editor);

editor.Type("Hello ");
history.Save();
editor.Type("World!");
Console.WriteLine(editor.GetText());  // "Hello World!"

history.Undo();
Console.WriteLine(editor.GetText());  // "Hello "
```

### Case Study: Game save/load

```csharp
public class GameState
{
    public Vector3 PlayerPosition { get; set; }
    public int Health { get; set; }
    public List<Item> Inventory { get; set; } = new();
    public Dictionary<string, bool> Quests { get; set; } = new();
    
    // Memento — JSON serialization
    public string CreateMemento() => JsonSerializer.Serialize(this);
    
    public void RestoreFromMemento(string memento)
    {
        var restored = JsonSerializer.Deserialize<GameState>(memento)!;
        PlayerPosition = restored.PlayerPosition;
        Health = restored.Health;
        Inventory = restored.Inventory;
        Quests = restored.Quests;
    }
}

public class SaveSystem
{
    private readonly string _saveFolder;
    
    public async Task SaveAsync(GameState state, string slotName)
    {
        var memento = state.CreateMemento();
        await File.WriteAllTextAsync($"{_saveFolder}/{slotName}.sav", memento);
    }
    
    public async Task LoadAsync(GameState state, string slotName)
    {
        var memento = await File.ReadAllTextAsync($"{_saveFolder}/{slotName}.sav");
        state.RestoreFromMemento(memento);
    }
}
```

### Case Study: Form wizard с back navigation

```csharp
public class CheckoutForm
{
    public string ShippingAddress { get; set; }
    public PaymentMethod Payment { get; set; }
    public bool TermsAccepted { get; set; }
    
    public Memento Snapshot() => new(this);
    public void Restore(Memento m) { /* ... */ }
    
    public class Memento
    {
        public string ShippingAddress { get; }
        public PaymentMethod Payment { get; }
        public bool TermsAccepted { get; }
        
        internal Memento(CheckoutForm f)
        {
            ShippingAddress = f.ShippingAddress;
            Payment = f.Payment;
            TermsAccepted = f.TermsAccepted;
        }
    }
}

// User — кликает Back → восстанавливаем prev step
```

### Когда применять Memento

✅ **Используй когда:**
- Undo / redo функциональность
- Save/load состояния
- Snapshot / rollback в transactions
- Form wizards с navigation

❌ **НЕ используй когда:**
- Состояние огромное (сохранять — дорого)
- State часто меняется и snapshots устаревают мгновенно
- Простой immutable record достаточен

### Memento vs Command

- **Command** — описывает **что сделать**
- **Memento** — описывает **состояние до изменения**
- Часто **используются вместе**: Command имеет Memento для Undo

См. [[../CSharp/design-patterns#State|State Pattern]] и [[modern-features|Records для immutable snapshots]].

---

## 6. Bridge — разделение abstraction и implementation

### Идея

Когда у тебя **два orthogonal измерения** (e.g., тип объекта × платформа), без Bridge получается **N×M классов**. С Bridge — **N + M**.

```
Без Bridge:                         С Bridge:
─────────────────                   ─────────────────
WindowsButton                        Button (abstraction)
LinuxButton                            └─ uses IRenderer (implementation)
MacButton                            
WindowsCheckbox                      WindowsRenderer
LinuxCheckbox                        LinuxRenderer
MacCheckbox                          MacRenderer
... (N × M classes)                  Checkbox
                                       └─ uses IRenderer
                                     ... (N + M classes)
```

### Базовая реализация

```csharp
// Implementation hierarchy
public interface IDeviceRenderer
{
    void DrawCircle(int x, int y, int radius);
    void DrawText(string text, int x, int y);
}

public class WindowsRenderer : IDeviceRenderer
{
    public void DrawCircle(int x, int y, int r) => /* GDI calls */;
    public void DrawText(string t, int x, int y) => /* GDI calls */;
}

public class LinuxRenderer : IDeviceRenderer
{
    public void DrawCircle(int x, int y, int r) => /* Cairo calls */;
    public void DrawText(string t, int x, int y) => /* Cairo calls */;
}

// Abstraction hierarchy
public abstract class Shape
{
    protected IDeviceRenderer Renderer { get; }
    
    protected Shape(IDeviceRenderer renderer) => Renderer = renderer;
    
    public abstract void Draw();
}

public class Circle : Shape
{
    public int X { get; set; }
    public int Y { get; set; }
    public int Radius { get; set; }
    
    public Circle(IDeviceRenderer r) : base(r) { }
    
    public override void Draw() => Renderer.DrawCircle(X, Y, Radius);
}

public class Label : Shape
{
    public string Text { get; set; }
    public int X { get; set; }
    public int Y { get; set; }
    
    public Label(IDeviceRenderer r) : base(r) { }
    
    public override void Draw() => Renderer.DrawText(Text, X, Y);
}

// Use
IDeviceRenderer renderer = OperatingSystem.IsWindows() 
    ? new WindowsRenderer() 
    : new LinuxRenderer();

Shape circle = new Circle(renderer) { X = 10, Y = 10, Radius = 50 };
Shape label = new Label(renderer) { Text = "Hello", X = 20, Y = 70 };

circle.Draw();
label.Draw();
```

### Case Study: Multi-database driver

```csharp
// Implementation: разные DB drivers
public interface IDatabaseDriver
{
    Task<List<Dictionary<string, object>>> ExecuteAsync(string query, object? parameters);
    Task ExecuteNonQueryAsync(string query, object? parameters);
}

public class SqlServerDriver : IDatabaseDriver { /* ... */ }
public class PostgresDriver : IDatabaseDriver { /* ... */ }
public class MySqlDriver : IDatabaseDriver { /* ... */ }

// Abstraction: бизнес-операции
public abstract class Repository<T>
{
    protected readonly IDatabaseDriver Driver;
    
    protected Repository(IDatabaseDriver driver) => Driver = driver;
    
    public abstract Task<T?> GetByIdAsync(int id);
    public abstract Task SaveAsync(T entity);
}

public class UserRepository : Repository<User>
{
    public UserRepository(IDatabaseDriver d) : base(d) { }
    
    public override async Task<User?> GetByIdAsync(int id)
    {
        var rows = await Driver.ExecuteAsync(
            "SELECT * FROM Users WHERE Id = @id", new { id });
        return rows.FirstOrDefault()?.MapTo<User>();
    }
    
    public override async Task SaveAsync(User user) { /* ... */ }
}

// 3 drivers × 5 repositories = 5 классов (не 15!)
// Switch DB — поменять только Driver registration
```

### Case Study: Notification — channels × content types

```csharp
// Channels (implementation)
public interface INotificationChannel
{
    Task SendAsync(Recipient r, string subject, string body);
}

public class EmailChannel : INotificationChannel { /* SMTP */ }
public class SmsChannel : INotificationChannel { /* Twilio */ }
public class PushChannel : INotificationChannel { /* Firebase */ }

// Notification types (abstraction)
public abstract class Notification
{
    protected INotificationChannel Channel;
    
    protected Notification(INotificationChannel ch) => Channel = ch;
    
    public abstract Task SendAsync(Recipient r);
}

public class WelcomeNotification : Notification
{
    public WelcomeNotification(INotificationChannel ch) : base(ch) { }
    
    public override Task SendAsync(Recipient r) =>
        Channel.SendAsync(r, "Welcome!", "Thanks for joining...");
}

public class PasswordResetNotification : Notification
{
    public string ResetCode { get; set; }
    
    public PasswordResetNotification(INotificationChannel ch) : base(ch) { }
    
    public override Task SendAsync(Recipient r) =>
        Channel.SendAsync(r, "Password Reset", $"Code: {ResetCode}");
}

// 3 channels × 10 notification types = 13 классов (не 30!)
```

### Когда применять Bridge

✅ **Используй когда:**
- Два orthogonal измерения изменения
- Cross-platform UI / database / messaging
- Хочешь избежать **взрыва классов** (Cartesian product)
- Need to swap implementations runtime

❌ **НЕ используй когда:**
- Только одно измерение варьирования (use Strategy / Adapter)
- Простая иерархия

### Bridge vs Strategy

- **Strategy** — отделяет **алгоритм** (one method)
- **Bridge** — отделяет **целую implementation hierarchy** (multiple methods)

См. [[design-patterns|Strategy Pattern]].

---

## 7. Flyweight — sharing для memory savings

### Идея

Когда нужно **много идентичных или почти-идентичных объектов** — sharing intrinsic (immutable) state.

```
Без Flyweight:                       С Flyweight:
─────────────────                    ─────────────────
1M particles × 200 bytes              1M particles × 50 bytes  
= 200 MB                              = 50 MB

Каждый particle хранит                Particle хранит position
position, color, texture,             Type — shared (color, texture, behavior)
behavior...                           1M particles share 5 types
```

### Базовая реализация — Particle system

```csharp
// Flyweight — shared intrinsic state
public class ParticleType
{
    public string TextureName { get; }
    public Color Color { get; }
    public Vector3 Acceleration { get; }
    public float MaxLifetime { get; }
    
    public ParticleType(string texture, Color color, Vector3 accel, float lifetime)
    {
        TextureName = texture;
        Color = color;
        Acceleration = accel;
        MaxLifetime = lifetime;
    }
    
    public void Render(Vector3 position, float age)
    {
        // Render с использованием intrinsic + extrinsic state
        Renderer.Draw(TextureName, position, Color, age / MaxLifetime);
    }
}

// Context — extrinsic state (unique per instance)
public class Particle
{
    public Vector3 Position;     // unique
    public Vector3 Velocity;     // unique
    public float Age;             // unique
    public ParticleType Type;    // SHARED reference!
    
    public void Update(float dt)
    {
        Velocity += Type.Acceleration * dt;
        Position += Velocity * dt;
        Age += dt;
    }
    
    public void Render() => Type.Render(Position, Age);
}

// Factory — гарантирует sharing
public class ParticleTypeFactory
{
    private readonly Dictionary<string, ParticleType> _types = new();
    
    public ParticleType GetOrCreate(string name, Func<ParticleType> factory)
    {
        if (!_types.TryGetValue(name, out var type))
        {
            type = factory();
            _types[name] = type;
        }
        return type;
    }
}

// Use
var factory = new ParticleTypeFactory();
var sparkType = factory.GetOrCreate("spark", () => 
    new ParticleType("spark.png", Color.Yellow, new Vector3(0, -9.8f, 0), 1.0f));
var smokeType = factory.GetOrCreate("smoke", () => 
    new ParticleType("smoke.png", Color.Gray, new Vector3(0, 1f, 0), 3.0f));

var particles = new List<Particle>();
for (int i = 0; i < 1_000_000; i++)
{
    particles.Add(new Particle 
    { 
        Position = RandomPosition(), 
        Velocity = RandomVelocity(),
        Type = (i % 2 == 0) ? sparkType : smokeType  // shared!
    });
}

// Memory: 1M particles × ~60 bytes (Vector3 × 2 + float + reference)
//   vs 1M × 200 bytes без sharing
//   = 140 MB savings
```

### Case Study: Text rendering (glyphs)

```csharp
// Each character "A", "B"... renders как glyph (shape outline)
// "Aaa..." 1000 раз — не дублируем 1000 описаний 'a'

// Flyweight — glyph
public class Glyph
{
    public char Character { get; }
    public byte[] OutlinePoints { get; }  // ~500 bytes per glyph
    public Size Size { get; }
    
    public Glyph(char c, byte[] outline, Size size)
    {
        Character = c;
        OutlinePoints = outline;
        Size = size;
    }
    
    public void Render(int x, int y, float scale, Color color)
    {
        // Render outline at extrinsic position/scale/color
    }
}

// Context — character instance в тексте
public class CharInstance
{
    public Glyph Glyph;       // SHARED!
    public int X;              // unique
    public int Y;              // unique
    public Color Color;        // unique
}

// Factory
public class GlyphCache
{
    private readonly Dictionary<char, Glyph> _glyphs = new();
    
    public Glyph Get(char c)
    {
        if (!_glyphs.TryGetValue(c, out var g))
        {
            g = LoadGlyphFromFont(c);  // expensive
            _glyphs[c] = g;
        }
        return g;
    }
}

// Document с 1M chars — sharing glyphs
// 1M × 30 bytes (instance) + 128 × 500 bytes (glyphs) = 30 MB + 64 KB
// vs 1M × 530 bytes без sharing = 500 MB
```

### Case Study: Map tiles

```csharp
// Карта города — много дублирующихся "trees", "buildings"
// Каждый tile — 1 KB геометрии
// Без Flyweight: 100K trees × 1 KB = 100 MB
// С Flyweight: 100K positions × 24 bytes + 5 unique trees × 1 KB = 2.5 MB

public class TreeType  // Flyweight
{
    public byte[] Geometry { get; }
    public string TextureName { get; }
}

public class TreePosition  // Context
{
    public Vector3 Position;
    public TreeType Type;  // shared
}
```

### Когда применять Flyweight

✅ **Используй когда:**
- Много объектов с дублирующимся state
- Память — bottleneck
- Можно разделить state на intrinsic (immutable, shared) и extrinsic (per-instance)
- Game development (particles, terrain, NPCs)
- Document rendering (chars, glyphs)

❌ **НЕ используй когда:**
- Объектов мало (< 1000)
- State у всех уникален
- Memory не проблема
- Sharing усложняет код > benefit

### .NET примеры

- **String interning** — все literal strings shared
- **`ImmutableArray<T>` / `ImmutableList<T>`** — структурное sharing
- **Boxing cache** для small int (-128 to 127)
- **Brushes / Pens** в WPF (создаются один раз, переиспользуются)

См. [[../Performance/optimization-patterns|Optimization Patterns]] и [[memory-pooling|Memory Pooling]].

---

## 8. Prototype — cloning без знания типа

### Идея

Создание нового объекта **через копирование** существующего, а не через `new` с указанием типа. Полезно когда:
- Тип объекта известен только runtime
- Создание дорогое (template + clone дешевле)

### Базовая реализация

```csharp
public interface IPrototype<T>
{
    T DeepClone();
}

public class Document : IPrototype<Document>
{
    public string Title { get; set; } = "";
    public string Content { get; set; } = "";
    public List<string> Tags { get; set; } = new();
    public Author Author { get; set; } = new();
    
    public Document DeepClone()
    {
        return new Document
        {
            Title = Title,
            Content = Content,
            Tags = new List<string>(Tags),  // deep copy of list
            Author = new Author              // deep copy of nested
            {
                Name = Author.Name,
                Email = Author.Email
            }
        };
    }
}

public class Author
{
    public string Name { get; set; }
    public string Email { get; set; }
}

// Use
var template = new Document
{
    Title = "Default Template",
    Content = "Hello, World!",
    Tags = new List<string> { "draft", "template" },
    Author = new Author { Name = "Admin", Email = "admin@example.com" }
};

var doc1 = template.DeepClone();
doc1.Title = "My Document";
doc1.Tags.Add("personal");
// template НЕ изменён
```

### Case Study: Game enemy spawning

```csharp
public abstract class Enemy : IPrototype<Enemy>
{
    public string Name;
    public int Health;
    public int Damage;
    public float Speed;
    public List<Ability> Abilities = new();
    
    public abstract Enemy DeepClone();
    
    public void TakeDamage(int dmg) => Health -= dmg;
}

public class Goblin : Enemy
{
    public override Enemy DeepClone() => new Goblin
    {
        Name = Name,
        Health = Health,
        Damage = Damage,
        Speed = Speed,
        Abilities = Abilities.Select(a => a.Clone()).ToList()
    };
}

public class Dragon : Enemy { /* ... */ }

// Prototype registry
public class EnemyRegistry
{
    private readonly Dictionary<string, Enemy> _prototypes = new();
    
    public void Register(string id, Enemy prototype)
    {
        _prototypes[id] = prototype;
    }
    
    public Enemy Spawn(string id)
    {
        if (_prototypes.TryGetValue(id, out var proto))
            return proto.DeepClone();
        throw new ArgumentException($"Unknown enemy: {id}");
    }
}

// Setup
var registry = new EnemyRegistry();
registry.Register("goblin_warrior", new Goblin
{
    Name = "Goblin Warrior",
    Health = 50,
    Damage = 10,
    Speed = 5,
    Abilities = new() { new Slash(), new Block() }
});
registry.Register("ancient_dragon", new Dragon { Health = 1000, /* ... */ });

// Spawn 100 goblins — каждый independent copy
for (int i = 0; i < 100; i++)
{
    var goblin = registry.Spawn("goblin_warrior");
    goblin.TakeDamage(20);  // не affects template
    SpawnInWorld(goblin);
}
```

### Case Study: Configuration template

```csharp
// Template configuration → клонируется и адаптируется per environment
public class AppConfig : IPrototype<AppConfig>
{
    public string ConnectionString { get; set; }
    public LoggingConfig Logging { get; set; } = new();
    public SecurityConfig Security { get; set; } = new();
    public CacheConfig Cache { get; set; } = new();
    
    public AppConfig DeepClone()
    {
        return new AppConfig
        {
            ConnectionString = ConnectionString,
            Logging = new LoggingConfig
            {
                Level = Logging.Level,
                Targets = new List<string>(Logging.Targets)
            },
            Security = new SecurityConfig { /* ... */ },
            Cache = new CacheConfig { /* ... */ }
        };
    }
}

// Use
var template = new AppConfig
{
    Logging = new LoggingConfig { Level = "Info", Targets = new() { "Console" } },
    /* ... */
};

var devConfig = template.DeepClone();
devConfig.Logging.Level = "Debug";
devConfig.ConnectionString = "Server=localhost;...";

var prodConfig = template.DeepClone();
prodConfig.Logging.Level = "Warning";
prodConfig.Logging.Targets.Add("Seq");
prodConfig.ConnectionString = "Server=prod-db;...";
```

### Modern C# — `with` expression для records

```csharp
public record Document(string Title, string Content, ImmutableList<string> Tags);

var template = new Document(
    Title: "Template",
    Content: "Hello",
    Tags: ImmutableList.Create("draft"));

// `with` — built-in shallow clone with modifications
var doc1 = template with { Title = "My Doc" };
var doc2 = template with { Tags = template.Tags.Add("urgent") };

// Records + with = Prototype pattern built-in!
```

### Когда применять Prototype

✅ **Используй когда:**
- Создание объекта дорогое (parsing, DB, computation)
- Нужно много вариантов одного template
- Тип неизвестен compile time
- Game spawning, document templates

❌ **НЕ используй когда:**
- `new T(...)` достаточен (простой constructor)
- Мало instances
- Records + `with` достаточны

### Deep vs Shallow clone

```csharp
// Shallow — копирует только references
public Document ShallowClone()
{
    return (Document)MemberwiseClone();  // built-in
}
// Tags list — same reference! Изменение в одном — affects другого

// Deep — копирует и nested
public Document DeepClone()
{
    return new Document
    {
        // ... copy всех полей рекурсивно
    };
}
```

> [!warning] Pitfall — Shallow clone surprises
> ```csharp
> var doc = template.ShallowClone();
> doc.Tags.Add("new");  // ⚠️ template.Tags ТОЖЕ имеет "new"!
> ```

### .NET примеры

- `ICloneable` — старый interface (избегать, не type-safe)
- `record` + `with` — modern way
- `MemberwiseClone()` — shallow в любом классе
- JSON-based deep clone: `JsonSerializer.Deserialize<T>(JsonSerializer.Serialize(obj))`

См. [[modern-features|Modern C# Features — records]] и [[functional-csharp|Functional C#]].

---

## 9. Cheat sheet — какой паттерн под какую задачу

| Задача | Паттерн |
|--------|---------|
| Undo/redo в редакторе | **Command** + **Memento** |
| Job queue с retry | **Command** |
| Macro recording | **Command** |
| AST traversal (compiler) | **Visitor** |
| Многократные операции над иерархией | **Visitor** |
| Filesystem / DOM tree | **Composite** |
| UI element tree | **Composite** |
| Меню с правами доступа | **Composite** |
| Lazy loading expensive object | **Proxy** (Virtual) |
| Caching API responses | **Proxy** (Caching) |
| Authorization wrapper | **Proxy** (Protection) |
| Save/load game state | **Memento** |
| Form wizard with back | **Memento** |
| Cross-platform UI | **Bridge** |
| Multi-DB driver | **Bridge** |
| Notifications: channels × types | **Bridge** |
| Particle system | **Flyweight** |
| Text rendering (glyphs) | **Flyweight** |
| Map tiles / objects | **Flyweight** |
| Game enemy spawning | **Prototype** |
| Document templates | **Prototype** |
| Config templates per env | **Prototype** (или `with` expression) |

---

## 10. Decision tree

```
Какую задачу решаешь?
│
├── Нужно encapsulate действие (queueing/undo)
│   → Command
│
├── Алгоритм над иерархией классов
│   ├── Иерархия стабильна, операций много → Visitor
│   └── Иначе → switch expression / pattern matching
│
├── Tree-структура с полиморфизмом
│   → Composite
│
├── Контролировать доступ к объекту (lazy/cache/auth)
│   ├── Lazy load → Virtual Proxy
│   ├── Caching → Caching Proxy
│   └── Permissions → Protection Proxy
│
├── Сохранение/восстановление состояния
│   → Memento
│
├── Два orthogonal измерения изменения (взрыв классов)
│   → Bridge
│
├── Память — bottleneck из-за дублирующихся объектов
│   → Flyweight
│
└── Создание объекта дорогое или тип неизвестен
    ├── Records → with expression
    └── Иначе → Prototype + DeepClone
```

---

## 11. Anti-patterns

### Pattern overuse

```csharp
// ❌ Visitor для 2 типов и 1 операции
public interface IShapeVisitor { void Visit(Circle c); void Visit(Square s); }

// ✅ Просто метод
public double GetArea(Shape s) => s switch
{
    Circle c => Math.PI * c.Radius * c.Radius,
    Square s => s.Side * s.Side,
    _ => throw new NotSupportedException()
};
```

### Over-abstraction

```csharp
// ❌ Bridge с одной implementation
public interface IRenderer { /* ... */ }
public class WindowsRenderer : IRenderer { /* ... */ }
// Только Windows — interface не нужен!

// ✅ Просто class
public class Renderer { /* ... */ }
```

### Premature flyweight

```csharp
// ❌ Sharing объектов которых всего 100 штук
// Memory savings — 100 × 50 bytes = 5 KB
// Code complexity — high
// Не стоит того
```

---

## См. также

- [[design-patterns|Design Patterns — основные 13]]
- [[../Architecture/patterns-decision-guide|Patterns Decision Guide]]
- [[../Architecture/real-world-scenarios|Real-World Scenarios]]
- [[modern-features|Modern C# — records, pattern matching]]
- [[functional-csharp|Functional C#]]
- [[../EFCore/patterns|EF Patterns — Repository, Specification]]
- [[oop|OOP — inheritance vs composition]]
- [[memory-pooling|Memory Pooling]]

## Reading list

- **Design Patterns: Elements of Reusable OO Software** — Gang of Four (классика)
- **Head First Design Patterns** — Eric Freeman (visual, beginner-friendly)
- **Refactoring.Guru — Patterns** — refactoring.guru/design-patterns
- **Microsoft Cloud Design Patterns** — learn.microsoft.com/azure/architecture/patterns
- **Mark Seemann — Dependency Injection in .NET** (где DI заменяет паттерны)
- **Robert Martin — Clean Architecture** (как паттерны складываются в архитектуру)
