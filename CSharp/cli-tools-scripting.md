---
tags: [csharp, cli, scripting, system-commandline, spectre-console, dotnet-script, terminal-gui, automation]
level: Senior
date: 2026-04-30
---

# CLI Tools и Scripting в C#

> Полный гайд по написанию командных утилит и скриптов на C#. Для автоматизации рабочих процессов, DevOps, internal tools, freelance utilities. Закрывает: System.CommandLine, Spectre.Console (богатый UI), Terminal.Gui (TUI приложения), dotnet-script, single-file deployment, Native AOT для CLI, packaging как NuGet tool.

---

## Что это, зачем и когда

### Зачем CLI на C#?

Bash / Python — стандарт для скриптов. Но C# выигрывает когда:

| | Bash / Python | C# CLI |
|--|---------------|--------|
| Type safety | ❌ | ✅ Compile-time errors |
| IDE support | Minimal | ✅ Full IntelliSense, debugging |
| Reusable код из своих библиотек | Только Python | ✅ Любой .NET код |
| Performance | Python — медленно | ✅ AOT — native speed |
| Distribution | Python deps hell | ✅ Single .exe |
| Cross-platform | ✅ | ✅ (.NET 6+) |
| Concurrency | Async tricky | ✅ async/await native |
| Domain | Sysadmin scripts | Tools, build pipelines, internal DSLs |

**Когда C# CLI > Bash:**
- Утилита со сложной логикой / типизированными данными
- Reuse существующих C# библиотек (e.g. company DTO, EF Core models)
- Tool для разработчиков (NuGet tool, dotnet-X)
- Internal automation для команды
- Cross-platform с богатой UI (Spectre)
- Performance-critical (AOT)

### Что покроем

| Library | Use case |
|---------|----------|
| `System.CommandLine` | Парсинг arguments, subcommands, validation |
| `Spectre.Console` | Богатый UI: tables, prompts, progress, charts |
| `Terminal.Gui` | TUI приложения (Norton Commander style) |
| `dotnet-script` | Скрипты `.csx` без проекта |
| `CliWrap` | Запуск других CLI из C# |
| `Sharprompt` | Простые interactive prompts |

---

## 1. Простой console app — baseline

### Top-level statements (.NET 6+)

```csharp
// Program.cs — никакой Main, namespace, class
using System;

if (args.Length == 0)
{
    Console.WriteLine("Usage: hello <name>");
    return 1;
}

Console.WriteLine($"Hello, {args[0]}!");
return 0;
```

```bash
dotnet run -- World
# Hello, World!

```

### Для простых случаев — вот это всё

```csharp
// rename-files.csx (через dotnet-script)
foreach (var file in Directory.GetFiles("./", "*.txt"))
{
    var newName = Path.ChangeExtension(file, ".bak");
    File.Move(file, newName);
    Console.WriteLine($"{file} → {newName}");
}
```

```bash
dotnet script rename-files.csx
```

Но как только появляются опции, флаги, валидация — нужны библиотеки.

---

## 2. System.CommandLine — стандарт

[github.com/dotnet/command-line-api](https://github.com/dotnet/command-line-api) — официальная библиотека от Microsoft.

### Установка

```xml
<PackageReference Include="System.CommandLine" Version="2.0.0" />
```

### Базовый пример

```csharp
using System.CommandLine;

var nameOption = new Option<string>(
    aliases: ["--name", "-n"],
    description: "Name to greet")
{
    IsRequired = true
};

var greetingOption = new Option<string>(
    aliases: ["--greeting", "-g"],
    description: "Greeting word",
    getDefaultValue: () => "Hello");

var rootCommand = new RootCommand("Hello CLI tool")
{
    nameOption,
    greetingOption
};

rootCommand.SetHandler((name, greeting) =>
{
    Console.WriteLine($"{greeting}, {name}!");
}, nameOption, greetingOption);

return await rootCommand.InvokeAsync(args);
```

```bash
dotnet run -- --name World --greeting "Bonjour"
# Bonjour, World!

dotnet run -- -n World -g "Привет"
# Привет, World!

dotnet run -- --help
# Hello CLI tool
# Usage: hello [options]
# Options:
#   -n, --name <name> (REQUIRED)  Name to greet
#   -g, --greeting <greeting>     Greeting word [default: Hello]

```

Auto-generated `--help`, validation, alias resolution — всё бесплатно.

### Subcommands

```csharp
var addCommand = new Command("add", "Add new item")
{
    new Argument<string>("name", "Item name"),
    new Option<int>("--quantity", () => 1)
};
addCommand.SetHandler((string name, int quantity) =>
{
    Console.WriteLine($"Added {quantity} × {name}");
}, addCommand.Arguments[0] as Argument<string>, addCommand.Options[0] as Option<int>);

var listCommand = new Command("list", "List items");
listCommand.SetHandler(() => 
{
    Console.WriteLine("Listing all items...");
});

var removeCommand = new Command("remove", "Remove item")
{
    new Argument<string>("name")
};

var rootCommand = new RootCommand("Inventory CLI")
{
    addCommand,
    listCommand,
    removeCommand
};

return await rootCommand.InvokeAsync(args);
```

```bash
inventory add laptop --quantity 5
inventory list
inventory remove laptop
```

### Argument validation

```csharp
var portOption = new Option<int>("--port", "Port number")
{
    IsRequired = true
};
portOption.AddValidator(result =>
{
    var port = result.GetValueForOption(portOption);
    if (port < 1 || port > 65535)
        result.ErrorMessage = "Port must be between 1 and 65535";
});

var fileArg = new Argument<FileInfo>("file", "Input file");
fileArg.AddValidator(result =>
{
    var file = result.GetValueForArgument(fileArg);
    if (!file.Exists)
        result.ErrorMessage = $"File not found: {file.FullName}";
});
```

### Globals и DI

```csharp
// Создать host для DI
var builder = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        services.AddSingleton<IInventoryService, InventoryService>();
        services.AddTransient<AddCommandHandler>();
    });

var host = builder.Build();

// Создать команды с handlers из DI
var rootCommand = new RootCommand("Inventory CLI");
var addCommand = new Command("add", "Add item");
addCommand.SetHandler(async (name, quantity) =>
{
    var handler = host.Services.GetRequiredService<AddCommandHandler>();
    await handler.ExecuteAsync(name, quantity);
}, nameArg, quantityOption);

rootCommand.AddCommand(addCommand);
return await rootCommand.InvokeAsync(args);
```

---

## 3. Spectre.Console — богатый UI

[spectreconsole.net](https://spectreconsole.net) — must-have для красивых CLI.

### Установка

```xml
<PackageReference Include="Spectre.Console" Version="0.50.0" />
<PackageReference Include="Spectre.Console.Cli" Version="0.50.0" />
```

### Цветной вывод и markup

```csharp
using Spectre.Console;

AnsiConsole.MarkupLine("[bold red]Error:[/] Something went wrong!");
AnsiConsole.MarkupLine("[green]✓[/] Build successful");
AnsiConsole.MarkupLine("[yellow on blue]Highlighted[/] text");

// Hyperlinks (terminals supporting OSC 8)
AnsiConsole.MarkupLine("[link=https://example.com]Click me[/]");

// Emoji
AnsiConsole.MarkupLine(":rocket: Launching!");
AnsiConsole.MarkupLine(":construction: Work in progress");
```

### Tables

```csharp
var table = new Table();
table.AddColumn("Name");
table.AddColumn(new TableColumn("Status").Centered());
table.AddColumn(new TableColumn("Progress").RightAligned());

table.AddRow("Service A", "[green]Running[/]", "100%");
table.AddRow("Service B", "[yellow]Starting[/]", "45%");
table.AddRow("Service C", "[red]Failed[/]", "0%");

table.Border = TableBorder.Rounded;
table.Title = new TableTitle("Service Status");

AnsiConsole.Write(table);
```

```
┌──────────┬──────────┬──────────┐
│ Name     │  Status  │ Progress │
├──────────┼──────────┼──────────┤
│ Service A│  Running │     100% │
│ Service B│  Starting│      45% │
│ Service C│  Failed  │       0% │
└──────────┴──────────┴──────────┘
```

### Prompts

```csharp
// Text input с validation
var name = AnsiConsole.Ask<string>("What's your [green]name[/]?");

var age = AnsiConsole.Prompt(
    new TextPrompt<int>("Age?")
        .Validate(age => age switch
        {
            < 0 => ValidationResult.Error("Negative age?"),
            > 130 => ValidationResult.Error("Really?"),
            _ => ValidationResult.Success()
        }));

// Password (без echo)
var password = AnsiConsole.Prompt(
    new TextPrompt<string>("Password:")
        .PromptStyle("red")
        .Secret());

// Confirmation
if (!AnsiConsole.Confirm("Continue?"))
    return;

// Selection
var fruit = AnsiConsole.Prompt(
    new SelectionPrompt<string>()
        .Title("Pick a fruit")
        .AddChoices("Apple", "Banana", "Cherry"));

// Multi-selection
var toppings = AnsiConsole.Prompt(
    new MultiSelectionPrompt<string>()
        .Title("Pick toppings")
        .Required()
        .PageSize(10)
        .AddChoices("Pepperoni", "Mushrooms", "Onions", "Olives", "Cheese"));
```

### Progress bars

```csharp
await AnsiConsole.Progress()
    .Columns(
        new TaskDescriptionColumn(),
        new ProgressBarColumn(),
        new PercentageColumn(),
        new SpinnerColumn())
    .StartAsync(async ctx =>
    {
        var download = ctx.AddTask("[green]Downloading[/]");
        var process = ctx.AddTask("[yellow]Processing[/]");
        
        while (!ctx.IsFinished)
        {
            download.Increment(1.5);
            process.Increment(0.5);
            await Task.Delay(50);
        }
    });
```

### Status (spinner)

```csharp
await AnsiConsole.Status()
    .Spinner(Spinner.Known.Dots)
    .StartAsync("Loading...", async ctx =>
    {
        ctx.Status("Connecting to server...");
        await Task.Delay(1000);
        
        ctx.Status("Fetching data...");
        await Task.Delay(1500);
        
        ctx.Status("Processing...");
        await Task.Delay(1000);
    });
```

### Tree

```csharp
var tree = new Tree("📁 Project")
    .AddNode("📂 src")
        .AddNode("📄 Program.cs")
        .AddNode("📄 Startup.cs");

tree.AddNode("📂 tests")
    .AddNode("📄 UnitTests.cs");

AnsiConsole.Write(tree);
```

### Bar charts

```csharp
AnsiConsole.Write(new BarChart()
    .Width(60)
    .Label("[green bold]Sales by region[/]")
    .CenterLabel()
    .AddItem("Europe", 312, Color.Yellow)
    .AddItem("North America", 410, Color.Green)
    .AddItem("Asia", 530, Color.Red)
    .AddItem("South America", 100, Color.Aqua));
```

### Live display — обновляемый UI

```csharp
await AnsiConsole.Live(new Table())
    .StartAsync(async ctx =>
    {
        for (int i = 0; i < 10; i++)
        {
            var table = new Table()
                .AddColumn("Counter")
                .AddColumn("Status")
                .AddRow(i.ToString(), i < 5 ? "[yellow]Pending[/]" : "[green]Done[/]");
            
            ctx.UpdateTarget(table);
            await Task.Delay(500);
        }
    });
```

---

## 4. Spectre.Console.Cli — комбинация с CommandLine

Spectre имеет свой command system — альтернатива `System.CommandLine`.

```csharp
using Spectre.Console;
using Spectre.Console.Cli;

public class AddSettings : CommandSettings
{
    [CommandArgument(0, "<name>")]
    [Description("Name of the item")]
    public string Name { get; set; } = "";
    
    [CommandOption("-q|--quantity <qty>")]
    [Description("Quantity")]
    [DefaultValue(1)]
    public int Quantity { get; set; }
    
    public override ValidationResult Validate()
    {
        return Quantity > 0 
            ? ValidationResult.Success()
            : ValidationResult.Error("Quantity must be positive");
    }
}

public class AddCommand : Command<AddSettings>
{
    public override int Execute(CommandContext context, AddSettings settings)
    {
        AnsiConsole.MarkupLine($"[green]✓[/] Added {settings.Quantity} × {settings.Name}");
        return 0;
    }
}

// Async version
public class FetchCommand : AsyncCommand<FetchSettings>
{
    public override async Task<int> ExecuteAsync(CommandContext context, FetchSettings settings)
    {
        await Task.Delay(1000);
        return 0;
    }
}

// Setup
var app = new CommandApp();
app.Configure(config =>
{
    config.SetApplicationName("inventory");
    
    config.AddCommand<AddCommand>("add")
        .WithDescription("Add a new item")
        .WithExample("add laptop --quantity 5");
    
    config.AddCommand<ListCommand>("list");
    
    config.AddBranch("settings", settings =>
    {
        settings.AddCommand<ShowSettingsCommand>("show");
        settings.AddCommand<EditSettingsCommand>("edit");
    });
});

return await app.RunAsync(args);
```

```bash
inventory add laptop -q 5
inventory list
inventory settings show
```

### DI integration с Spectre.Console.Cli

```csharp
public sealed class TypeRegistrar(IServiceCollection builder) : ITypeRegistrar
{
    public ITypeResolver Build() => new TypeResolver(builder.BuildServiceProvider());
    public void Register(Type service, Type implementation) => 
        builder.AddSingleton(service, implementation);
    public void RegisterInstance(Type service, object impl) => 
        builder.AddSingleton(service, impl);
    public void RegisterLazy(Type service, Func<object> factory) =>
        builder.AddSingleton(service, _ => factory());
}

public sealed class TypeResolver(IServiceProvider provider) : ITypeResolver, IDisposable
{
    public object? Resolve(Type? type) => type is null ? null : provider.GetService(type);
    public void Dispose() => (provider as IDisposable)?.Dispose();
}

// Setup
var services = new ServiceCollection();
services.AddSingleton<IInventoryService, InventoryService>();

var registrar = new TypeRegistrar(services);
var app = new CommandApp(registrar);
app.Configure(config => /* ... */);

return await app.RunAsync(args);

// Команды получают зависимости через ctor
public class AddCommand(IInventoryService inventory) : AsyncCommand<AddSettings>
{
    public override async Task<int> ExecuteAsync(CommandContext context, AddSettings settings)
    {
        await inventory.AddAsync(settings.Name, settings.Quantity);
        return 0;
    }
}
```

---

## 5. Terminal.Gui — TUI приложения

[gui-cs](https://github.com/gui-cs/Terminal.Gui) — Norton Commander / Midnight Commander style апп.

```xml
<PackageReference Include="Terminal.Gui" Version="2.0.0" />
```

```csharp
using Terminal.Gui;

Application.Init();

var top = Application.Top;

var win = new Window("My TUI App")
{
    X = 0, Y = 1,
    Width = Dim.Fill(),
    Height = Dim.Fill() - 1
};
top.Add(win);

// Menu
var menu = new MenuBar(new MenuBarItem[]
{
    new("_File", new MenuItem[]
    {
        new("_Open", "", () => MessageBox.Query("Open", "Open file?", "Yes", "No")),
        new("_Quit", "", () => Application.RequestStop())
    }),
    new("_Help", new MenuItem[]
    {
        new("_About", "", () => MessageBox.Query("About", "TUI app v1.0", "OK"))
    })
});
top.Add(menu);

// Login form
var loginLabel = new Label("Login:") { X = 3, Y = 2 };
var loginText = new TextField("") { X = 14, Y = 2, Width = 40 };

var passwordLabel = new Label("Password:") { X = 3, Y = 4 };
var passwordText = new TextField("") { X = 14, Y = 4, Width = 40, Secret = true };

var loginButton = new Button("Login") { X = 14, Y = 6 };
loginButton.Clicked += () =>
{
    MessageBox.Query("Login", $"Welcome, {loginText.Text}!", "OK");
};

win.Add(loginLabel, loginText, passwordLabel, passwordText, loginButton);

Application.Run();
Application.Shutdown();
```

Полноценные TUI apps — file managers, monitoring tools (htop-like), git UI.

---

## 6. dotnet-script — скрипты без проекта

```bash
dotnet tool install -g dotnet-script
```

### Простой скрипт `.csx`

```csharp
// hello.csx
#r "nuget: Newtonsoft.Json, 13.0.3"
using Newtonsoft.Json;

var data = new { Name = "World", Time = DateTime.UtcNow };
var json = JsonConvert.SerializeObject(data, Formatting.Indented);
Console.WriteLine(json);
```

```bash
dotnet script hello.csx
# {
#   "Name": "World",
#   "Time": "2026-04-30T..."
# }

```

### Argument access

```csharp
// args.csx
if (Args.Count == 0)
{
    Console.WriteLine("Usage: dotnet script args.csx <name>");
    Environment.Exit(1);
}

var name = Args[0];
Console.WriteLine($"Hello, {name}!");
```

```bash
dotnet script args.csx -- World
```

### Включение существующего проекта

```csharp
// uses-project.csx
#r "../MyProject/bin/Debug/net10.0/MyProject.dll"
using MyProject.Services;

var service = new MyService();
service.Run();
```

### Shebang для Linux

```csharp
#!/usr/bin/env dotnet-script
Console.WriteLine("Hello!");
```

```bash
chmod +x hello.csx
./hello.csx
```

> [!info] Когда dotnet-script vs полноценный проект
> - Скрипт <100 строк, разовый — `.csx`
> - Production tool с тестами, refactoring — полноценный project

---

## 7. CliWrap — запуск других CLI

[github.com/Tyrrrz/CliWrap](https://github.com/Tyrrrz/CliWrap) — лучшая библиотека для process invocation.

```xml
<PackageReference Include="CliWrap" Version="3.7.0" />
```

```csharp
using CliWrap;
using CliWrap.Buffered;

// Запуск + buffered output
var result = await Cli.Wrap("git")
    .WithArguments(["log", "--oneline", "-10"])
    .WithWorkingDirectory("/path/to/repo")
    .ExecuteBufferedAsync();

Console.WriteLine(result.StandardOutput);
Console.WriteLine($"Exit: {result.ExitCode}");
Console.WriteLine($"Time: {result.RunTime}");

// Streaming — для long-running processes
await Cli.Wrap("docker")
    .WithArguments(["build", "-t", "myimage", "."])
    .WithStandardOutputPipe(PipeTarget.ToDelegate(line => 
        Console.WriteLine($"[STDOUT] {line}")))
    .WithStandardErrorPipe(PipeTarget.ToDelegate(line =>
        Console.WriteLine($"[STDERR] {line}")))
    .ExecuteAsync();

// Pipe — chain commands
await (Cli.Wrap("git").WithArguments("log") 
       | Cli.Wrap("grep").WithArguments("fix")
       | Cli.Wrap("wc").WithArguments("-l"))
    .ExecuteBufferedAsync();

// Env vars
await Cli.Wrap("npm")
    .WithArguments("run", "build")
    .WithEnvironmentVariables(env => env
        .Set("NODE_ENV", "production")
        .Set("API_URL", "https://api.example.com"))
    .ExecuteAsync();

// Cancellation + timeout
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
try
{
    await Cli.Wrap("long-running-task").ExecuteAsync(cts.Token);
}
catch (OperationCanceledException)
{
    Console.WriteLine("Timeout");
}
```

### Validation

```csharp
// По умолчанию exit code != 0 → throw
var result = await Cli.Wrap("might-fail")
    .WithValidation(CommandResultValidation.None)  // не throw
    .ExecuteBufferedAsync();

if (result.ExitCode != 0)
{
    Console.WriteLine($"Failed: {result.StandardError}");
}
```

---

## 8. Sharprompt — простые prompts (alternative)

Если нужны только prompts без всего Spectre:

```xml
<PackageReference Include="Sharprompt" Version="3.0.0" />
```

```csharp
using Sharprompt;

var name = Prompt.Input<string>("Name", placeholder: "John");

var role = Prompt.Select("Role", new[] { "Admin", "User", "Guest" });

var modules = Prompt.MultiSelect(
    "Select modules", 
    new[] { "Logging", "Caching", "Auth" },
    minimum: 1);

var confirm = Prompt.Confirm("Continue?", defaultValue: true);

var password = Prompt.Password("Password", validators: new[]
{
    Validators.Required(),
    Validators.MinLength(8)
});
```

---

## 9. Distribution — single .exe

### Self-contained executable

```bash
# Linux
dotnet publish -c Release -r linux-x64 --self-contained true \
    /p:PublishSingleFile=true \
    /p:PublishTrimmed=true

# Windows
dotnet publish -c Release -r win-x64 --self-contained true \
    /p:PublishSingleFile=true

# macOS
dotnet publish -c Release -r osx-arm64 --self-contained true \
    /p:PublishSingleFile=true
```

Результат — **один файл** (~15-50 MB), включает .NET runtime. Запускается без установки .NET.

### Native AOT (.NET 8+) — ~5 MB, native speed

```xml
<PropertyGroup>
  <OutputType>Exe</OutputType>
  <PublishAot>true</PublishAot>
  <InvariantGlobalization>true</InvariantGlobalization>
  <StripSymbols>true</StripSymbols>
</PropertyGroup>
```

```bash
dotnet publish -c Release -r linux-x64
# Output: ~5 MB native binary, instant startup

```

> [!info] AOT ограничения
> - Reflection limited (`DynamicallyAccessedMembers`)
> - Source generators ОК, runtime codegen — нет
> - Не все NuGet packages AOT-compat

См. [Native AOT](../AspNetCore/native-aot.md).

### Cross-platform packaging

```yaml
# .github/workflows/release.yml
- name: Publish all platforms
  run: |
    for rid in linux-x64 linux-arm64 win-x64 osx-x64 osx-arm64; do
      dotnet publish -c Release -r $rid --self-contained \
        /p:PublishSingleFile=true \
        -o publish/$rid
    done

- name: Create archives
  run: |
    cd publish
    for dir in */; do
      tar czf ${dir%/}.tar.gz $dir
    done
```

### Как NuGet tool

```xml
<PropertyGroup>
  <PackAsTool>true</PackAsTool>
  <ToolCommandName>my-tool</ToolCommandName>
  <PackageOutputPath>./nupkg</PackageOutputPath>
</PropertyGroup>
```

```bash
dotnet pack
dotnet tool install --global --add-source ./nupkg MyApp.Tool

# Использование
my-tool --help

# Update
dotnet tool update --global MyApp.Tool
```

Публикуй в NuGet.org или приватный feed → команда устанавливает `dotnet tool install -g your-tool`.

---

## 10. Real-world примеры

### Pattern 1: Build automation

```csharp
// build.csx
#r "nuget: CliWrap, 3.7.0"
using CliWrap;
using CliWrap.Buffered;

await Cli.Wrap("dotnet").WithArguments(["restore"]).ExecuteAsync();
await Cli.Wrap("dotnet").WithArguments(["build", "-c", "Release", "--no-restore"]).ExecuteAsync();
await Cli.Wrap("dotnet").WithArguments(["test", "--no-build"]).ExecuteAsync();

Console.WriteLine("✓ Build complete");
```

### Pattern 2: Database backup tool

```csharp
public class BackupCommand : AsyncCommand<BackupSettings>
{
    public override async Task<int> ExecuteAsync(CommandContext context, BackupSettings settings)
    {
        return await AnsiConsole.Status()
            .Spinner(Spinner.Known.Dots)
            .StartAsync($"Backing up {settings.Database}...", async ctx =>
            {
                ctx.Status("Connecting to PostgreSQL...");
                
                var result = await Cli.Wrap("pg_dump")
                    .WithArguments(["-h", settings.Host, "-U", settings.User, "-d", settings.Database, "-f", settings.Output])
                    .WithEnvironmentVariables(env => env.Set("PGPASSWORD", settings.Password))
                    .ExecuteBufferedAsync();
                
                if (result.ExitCode == 0)
                {
                    AnsiConsole.MarkupLine($"[green]✓[/] Backup saved to {settings.Output}");
                    return 0;
                }
                
                AnsiConsole.MarkupLine($"[red]✗[/] {result.StandardError}");
                return 1;
            });
    }
}
```

### Pattern 3: Interactive CLI menu

```csharp
while (true)
{
    var action = AnsiConsole.Prompt(
        new SelectionPrompt<string>()
            .Title("What to do?")
            .AddChoices("Add", "List", "Remove", "Settings", "Exit"));
    
    switch (action)
    {
        case "Add":
            var name = AnsiConsole.Ask<string>("Name?");
            var qty = AnsiConsole.Ask<int>("Quantity?");
            // ...
            break;
        case "List":
            ShowList();
            break;
        case "Exit":
            return;
    }
}
```

### Pattern 4: dev tool для команды

```csharp
// Внутренний tool: gen-feature
// Использование: gen-feature Order --slice CreateOrder

public class GenerateFeatureCommand : Command<GenerateFeatureSettings>
{
    public override int Execute(CommandContext ctx, GenerateFeatureSettings s)
    {
        var dir = Path.Combine("src", "Features", s.Module, s.Name);
        Directory.CreateDirectory(dir);
        
        // Generate from templates
        File.WriteAllText(Path.Combine(dir, $"{s.Name}Command.cs"), 
            CommandTemplate.Render(s));
        File.WriteAllText(Path.Combine(dir, $"{s.Name}Handler.cs"), 
            HandlerTemplate.Render(s));
        File.WriteAllText(Path.Combine(dir, $"{s.Name}Validator.cs"), 
            ValidatorTemplate.Render(s));
        File.WriteAllText(Path.Combine(dir, $"{s.Name}Endpoint.cs"), 
            EndpointTemplate.Render(s));
        
        AnsiConsole.MarkupLine($"[green]✓[/] Generated feature [yellow]{s.Module}/{s.Name}[/]");
        return 0;
    }
}
```

---

## 11. Cross-platform pitfalls

### 1. Путей separator

```csharp
// ❌ Hard-coded /
var path = $"data/users/{id}.json";  // Linux works, Windows тоже понимает но...

// ✅ Path.Combine — cross-platform
var path = Path.Combine("data", "users", $"{id}.json");
```

### 2. Line endings

```csharp
// ❌ Hard-coded
text.Split("\n");  // ломается на Windows CRLF

// ✅
text.Split('\n', StringSplitOptions.None);
text.ReplaceLineEndings();  // .NET 6+ нормализация
```

### 3. Console encoding

```csharp
// Windows может быть cp1251, нужно UTF-8 для emoji
Console.OutputEncoding = System.Text.Encoding.UTF8;
```

### 4. Чтение скрытого input (passwords)

```csharp
// ❌ Console.ReadKey() ведёт себя по-разному в IDE vs terminal
// ✅ Используй Spectre.Console.Prompt.Secret() или Sharprompt.Password
```

### 5. ANSI colors в Windows < 10

Windows 10+ поддерживает ANSI escape codes. Для legacy — Spectre.Console сам fallback'ит.

---

## 12. Best Practices

- **System.CommandLine** для simple commands, **Spectre.Console.Cli** для рекач TUI / multi-command
- **Spectre.Console** для богатого UI — tables, prompts, progress
- **CliWrap** для запуска других CLI (вместо `Process.Start`)
- **dotnet-script** для разовых скриптов
- **Native AOT** для production CLI tools (small, fast startup)
- **Single-file publish** для distribution
- **Pack as NuGet tool** для команды/community
- **`.editorconfig` + `dotnet format`** в скриптах
- **Args validation** — `ValidationResult` или `AddValidator`
- **Exit codes** — 0 success, 1+ error (стандарт)
- **stderr для ошибок** — `Console.Error.WriteLine()` не stdout
- **CancellationToken everywhere** — `Ctrl+C` обработка
- **Logging через ILogger** даже в CLI (в файл, не stdout)
- **Configuration через appsettings.json** + env vars, как в ASP.NET

### Ctrl+C handling

```csharp
using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) =>
{
    e.Cancel = true;  // не убивай процесс сразу
    cts.Cancel();
    AnsiConsole.MarkupLine("[yellow]Cancelling...[/]");
};

try
{
    await DoWorkAsync(cts.Token);
}
catch (OperationCanceledException)
{
    return 130;  // standard "Cancelled" exit code
}
```

### Exit codes — стандарт

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Misuse of shell builtins (по conventions) |
| 64-78 | sysexits.h — usage / data err / no host etc |
| 130 | Killed by Ctrl+C (128 + SIGINT 2) |
| 137 | Killed by SIGKILL (128 + 9) |

---

## 13. Альтернативы и сравнение

| Library | Use case | Сложность |
|---------|----------|-----------|
| `System.CommandLine` | Standard parsing, validation | Средняя |
| `Spectre.Console.Cli` | Rich CLI с UI | Низкая (дружелюбный) |
| `CommandLineParser` | Legacy | Низкая |
| `McMaster.Extensions.CommandLineUtils` | Mid-weight | Средняя |
| `CliFx` | Минималистичный | Низкая |
| `ConsoleAppFramework` | High-perf, AOT-friendly | Низкая |

### Когда что

- **Свой tool на 5 commands, простой** — `Spectre.Console.Cli`
- **Сложная иерархия subcommands** — `System.CommandLine` или `Spectre.Console.Cli`
- **Maximum performance** — `ConsoleAppFramework` (Source Generator-based)
- **Скрипты разовые** — `dotnet-script`

---

## См. также

- [Modern C# Features](modern-features.md) — top-level statements, records
- [Native AOT](../AspNetCore/native-aot.md) — компиляция CLI в native
- [Source Generators](source-generators.md) — для AOT-friendly tools
- [Functional C#](functional-csharp.md) — pipelines в скриптах
- [Project Setup](../Infrastructure/project-setup.md) — Directory.Build.props для CLI
- [Diagnostics Tools](../Runtime/diagnostics-tools.md) — построены на System.CommandLine

## Reading list

- **Spectre.Console docs** — spectreconsole.net
- **System.CommandLine docs** — learn.microsoft.com/dotnet/standard/commandline
- **CliWrap GitHub** — github.com/Tyrrrz/CliWrap
- **dotnet-script GitHub** — github.com/dotnet-script/dotnet-script
- **Terminal.Gui** — github.com/gui-cs/Terminal.Gui
- **Andrew Lock — System.CommandLine series** — andrewlock.net
- **Tyrrrz — Async-friendly CliWrap** — tyrrrz.me
