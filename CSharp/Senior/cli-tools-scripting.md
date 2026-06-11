---
tags: [csharp, cli, scripting, senior, system-commandline, dotnet-script, top-level-statements, native-aot]
level: Senior
date: 2026-05-09
---

# CLI Tools и Scripting на C#

> **Top-level statements, `dotnet script`, `System.CommandLine`, Spectre.Console, Native AOT для CLI.** Замена Bash/Python для DevOps/automation на типизированном языке. Закрывает пробел: «знаю console apps, не понимаю как написать production CLI с subcommands и почему AOT для CLI критичен».

---

## 0. Как читать

Если впервые — раздел 1→3 (top-level + script setup). System.CommandLine — раздел 4. Spectre.Console — раздел 5. Native AOT — раздел 7. Production guidance — раздел 9 (best practices), 11 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. C# для CLI — недооценённая ниша

Исторически CLI tools писали на Bash, Python, Go, Ruby. C# считался "тяжёлым" — JIT warmup, runtime dependency. Современный .NET (8+) изменил уравнение:

```
Современный C# для CLI:
- Top-level statements (C# 9+) — no boilerplate
- dotnet script (.csx files) — Bash-like scripting
- System.CommandLine — production CLI parser
- Native AOT (.NET 8+) — single binary, fast startup
- ~10-30ms cold start на AOT (~ Go)
```

### 1.2. Когда C# CLI оправдан

```
✅ Используй когда:
  - Team знает C# (no learning curve)
  - Integration с .NET ecosystem (NuGet, APIs)
  - Need strong types (refactoring, IDE help)
  - Cross-platform binary required
  - Performance matters (Native AOT)

❌ Не используй когда:
  - One-off shell snippet (Bash проще)
  - Glue code между Unix tools (sed/awk lighter)
  - Team Python-shop
  - System scripting (Bash native)
```

### 1.3. Главное правило

```
Простой скрипт (< 50 lines) → dotnet script (.csx)
Multi-command CLI → System.CommandLine + console project
Production tool (distribute) → Native AOT publish
Beautiful output → Spectre.Console
Configuration / settings → Microsoft.Extensions.Configuration

Distribute как single binary через `dotnet publish -c Release --aot`.
```

### 1.4. Эволюция

| Версия | Что |
|--------|-----|
| **.NET Core 1.0** | dotnet CLI, console project template |
| **C# 9.0** | Top-level statements (no `Main`) |
| **.NET 5+** | `dotnet script` mature (.csx) |
| **.NET 6+** | Implicit usings, file-scoped namespaces |
| **.NET 7+** | `[GeneratedRegex]` улучшает CLI parsers |
| **.NET 8+** | Native AOT production-ready, System.CommandLine stable |
| **.NET 9** | Better AOT support, smaller binaries |

### 1.5. Ecosystem options

```
Argument parsers:
- System.CommandLine (Microsoft) — official, modern
- CommandLineParser (community) — older, mature
- Cocona (community) — convention-based
- McMaster.Extensions.CommandLineUtils — enterprise

Output formatting:
- Spectre.Console — best для rich UI (tables, progress, prompts)
- ConsoleTables — simple tables
- Pastel — ANSI colors

Config / DI:
- Microsoft.Extensions.* (Configuration, DependencyInjection, Logging, Hosting)

Scripting:
- dotnet script (Roslyn-based)
- C# Interactive (csi.exe)
- LINQPad (commercial, GUI)
```

> [!info]- Если ты знаешь Bash / Python / Go / Rust
> **Bash:** simple glue code, no types. C# — when need strong types + NuGet ecosystem.
>
> **Python:** dominant scripting (`argparse`, `click`, `typer`, `rich`). C# competes via dotnet script + Spectre.Console.
>
> **Go:** `cobra`/`urfave/cli` для CLI, single binary native. C# Native AOT даёт same UX.
>
> **Rust:** `clap` parser, single binary. Slower compile, harder learning curve. C# easier productivity, similar binary size.

> [!question]- Интервью: когда выбрать C# для CLI tool?
> 1) **Team имеет C# expertise** — no Python/Bash learning curve. 2) **Integration с .NET ecosystem** — NuGet packages, internal APIs, EF Core. 3) **Strong typing critical** — refactoring/IDE help для complex CLIs. 4) **Cross-platform binary** — Native AOT (.NET 8+) даёт single executable Windows/Linux/macOS. 5) **Performance matters** — AOT startup ~10-30ms (≈ Go). **Не выбирай**: simple shell scripts (Bash проще), Unix tool gluing, Python-heavy team. Best practice: top-level statements C# 9 для prototypes, System.CommandLine + AOT для production.

---

## 2. Top-level statements

### 2.1. Минимальный CLI (C# 9+)

```csharp
// Program.cs — без класса, без Main
Console.WriteLine("Hello, CLI!");
var name = args.Length > 0 ? args[0] : "World";
Console.WriteLine($"Hello, {name}!");
```

```bash
dotnet run -- Vitaly
# Output: Hello, Vitaly!
```

`args` — implicit parameter. Compiler generates `Main` под капотом.

### 2.2. Async top-level

```csharp
// Program.cs
using System.Net.Http;

var client = new HttpClient();
var content = await client.GetStringAsync("https://api.github.com");
Console.WriteLine(content[..200]);
```

`await` работает прямо в top-level. Compiler оборачивает в `async Task Main`.

### 2.3. Local functions для structure

```csharp
// Program.cs
if (args.Length == 0)
{
    Console.WriteLine("Usage: tool <command>");
    return 1;
}

return args[0] switch
{
    "hello" => Hello(),
    "bye" => Bye(),
    _ => Unknown(args[0])
};

int Hello() { Console.WriteLine("Hello!"); return 0; }
int Bye()   { Console.WriteLine("Bye!");   return 0; }
int Unknown(string cmd) { Console.WriteLine($"Unknown: {cmd}"); return 1; }
```

Local functions — group helpers без class.

### 2.4. Exit codes

```csharp
return 0;      // success
return 1;      // generic error
Environment.Exit(2);   // immediate exit с code
```

POSIX convention: 0 = success, non-zero = error. CI/CD checks exit code.

### 2.5. Reading input

```csharp
// Stdin streaming
string? line;
while ((line = Console.ReadLine()) != null)
{
    Console.WriteLine($"Got: {line}");
}

// Single line
var name = Console.ReadLine();

// Read all stdin (piped input)
using var reader = new StreamReader(Console.OpenStandardInput(), Encoding.UTF8);
var content = await reader.ReadToEndAsync();
```

### 2.6. Stdout/Stderr

```csharp
Console.WriteLine("normal output");      // stdout
Console.Error.WriteLine("error output"); // stderr — для errors

// In tools — separate streams для logs vs data:
// echo "data" | tool | grep something
// errors не mix с data
```

> [!question]- Интервью: что такое top-level statements C# 9?
> Allow code **outside class/Main** в top-level Program.cs. Compiler generates hidden `class Program { static async Task<int> Main(string[] args) { ... } }`. **Implicit parameters**: `args` (string[]). **Implicit support**: `await`, return int (exit code). Use cases: 1) Scripts, prototypes (no boilerplate). 2) Simple CLI tools. 3) Microservices entry points. **Limitations**: only ONE file per project с top-level (compiler picks first). Mix с regular classes OK (other files normal). Local functions для grouping helpers.

---

## 3. dotnet script — Bash-like

### 3.1. Установка

```bash
dotnet tool install -g dotnet-script
```

Global tool. Добавляет команду `dotnet script`.

### 3.2. Простой script

```csharp
// hello.csx
#!/usr/bin/env dotnet-script
Console.WriteLine("Hello from script!");
```

```bash
dotnet script hello.csx
# Or на Linux/Mac с executable bit:
chmod +x hello.csx
./hello.csx
```

`.csx` — C# script extension. Roslyn compiles + runs immediately.

### 3.3. NuGet packages в scripts

```csharp
// http.csx
#r "nuget: Newtonsoft.Json, 13.0.3"

using Newtonsoft.Json;

var data = new { name = "Alice", age = 30 };
var json = JsonConvert.SerializeObject(data, Formatting.Indented);
Console.WriteLine(json);
```

`#r "nuget: ..."` — reference NuGet package directly. Cached locally после первого run.

### 3.4. Args в scripts

```csharp
// args.csx
foreach (var arg in Args)
{
    Console.WriteLine($"Arg: {arg}");
}
```

```bash
dotnet script args.csx -- hello world
# Args[0] = "hello", Args[1] = "world"
```

`Args` — `IList<string>` в scripts (vs `args` array в top-level).

### 3.5. Загрузка других files

```csharp
// utils.csx
public static int Square(int x) => x * x;
```

```csharp
// main.csx
#load "utils.csx"

Console.WriteLine(Square(5));   // 25
```

`#load` — include другой script. Compose modules.

### 3.6. Real example — process directories

```csharp
#!/usr/bin/env dotnet-script
#r "nuget: Spectre.Console, 0.49.1"
using Spectre.Console;
using System.IO;

if (Args.Count == 0)
{
    AnsiConsole.MarkupLine("[red]Usage: stats.csx <dir>[/]");
    return;
}

var dir = Args[0];
if (!Directory.Exists(dir))
{
    AnsiConsole.MarkupLine($"[red]Directory not found: {dir}[/]");
    return;
}

var files = Directory.GetFiles(dir, "*", SearchOption.AllDirectories);
var totalSize = files.Sum(f => new FileInfo(f).Length);

var table = new Table();
table.AddColumn("Metric");
table.AddColumn("Value");
table.AddRow("File count", files.Length.ToString("N0"));
table.AddRow("Total size", $"{totalSize / 1024.0 / 1024.0:F2} MB");

AnsiConsole.Write(table);
```

```bash
./stats.csx /home/user/projects
```

### 3.7. Когда csx vs project

```
.csx (dotnet script):
✅ One-off automation
✅ Bash-replacement scripts
✅ Quick prototypes
✅ Devops runbooks
❌ Production binaries
❌ Multi-file complex projects
❌ AOT compilation

Console project (.csproj):
✅ Production CLI
✅ Multi-file projects
✅ AOT compilation
✅ Library packaging
❌ Single-file simplicity
```

> [!question]- Интервью: что такое dotnet script?
> Roslyn-based **REPL и scripting tool** для запуска `.csx` files. Install: `dotnet tool install -g dotnet-script`. **Syntax**: top-level C# code (no class/Main needed). **`#r "nuget: ..."`** — reference NuGet packages directly. **`#load "other.csx"`** — include другие scripts. **`Args`** — script parameters. Use cases: DevOps automation, quick scripts (Bash replacement), prototyping. Trade-offs: cold start ~1-3s (Roslyn warmup), no AOT support, requires .NET runtime installed. Best practice: csx для glue/automation, console project для production tools.

---

## 4. System.CommandLine — production parser

### 4.1. Установка

```xml
<PackageReference Include="System.CommandLine" Version="2.0.0-beta4" />
```

Microsoft official, в long beta (2024). Готов production usage.

### 4.2. Simple command

```csharp
using System.CommandLine;

var nameOption = new Option<string>(
    name: "--name",
    description: "Your name")
{ IsRequired = true };

var rootCommand = new RootCommand("Greeter CLI")
{
    nameOption
};

rootCommand.SetHandler(name =>
{
    Console.WriteLine($"Hello, {name}!");
}, nameOption);

return await rootCommand.InvokeAsync(args);
```

```bash
mytool --name Vitaly
# Output: Hello, Vitaly!

mytool --help
# Auto-generated help
```

### 4.3. Subcommands

```csharp
var rootCommand = new RootCommand("File tool");

// init subcommand
var initCmd = new Command("init", "Initialize directory");
var pathArg = new Argument<DirectoryInfo>("path", "Path to init");
initCmd.AddArgument(pathArg);
initCmd.SetHandler(path => InitDirectory(path), pathArg);
rootCommand.AddCommand(initCmd);

// stats subcommand
var statsCmd = new Command("stats", "Show statistics");
var verboseOpt = new Option<bool>("--verbose", "Show details");
statsCmd.AddOption(verboseOpt);
statsCmd.SetHandler(verbose => ShowStats(verbose), verboseOpt);
rootCommand.AddCommand(statsCmd);

return await rootCommand.InvokeAsync(args);

void InitDirectory(DirectoryInfo path) { /* ... */ }
void ShowStats(bool verbose) { /* ... */ }
```

```bash
mytool init /tmp/myproject
mytool stats --verbose
mytool init --help    # subcommand help
```

### 4.4. Arguments vs Options

```
Argument — positional, required by default:
  mytool COPY <source> <dest>
  
Option — named, optional by default:
  mytool --verbose --name Alice
```

```csharp
var sourceArg = new Argument<FileInfo>("source");
var destArg = new Argument<FileInfo>("dest");
var verboseOpt = new Option<bool>("--verbose");
var forceOpt = new Option<bool>(["--force", "-f"]);   // multiple aliases

cmd.AddArgument(sourceArg);
cmd.AddArgument(destArg);
cmd.AddOption(verboseOpt);
cmd.AddOption(forceOpt);
```

### 4.5. Built-in conversions

```csharp
new Argument<int>("count")              // parses int
new Argument<DirectoryInfo>("path")     // checks existence
new Argument<FileInfo>("file")          // checks existence
new Argument<Uri>("url")                // parses URL
new Option<TimeSpan>("--timeout")       // "00:00:30"
new Option<DateTime>("--date")          // ISO format
new Option<string[]>("--tag")           // multiple values

// Custom enum
new Option<LogLevel>("--level")         // LogLevel.Info etc
```

### 4.6. Validation

```csharp
var portOpt = new Option<int>("--port");
portOpt.AddValidator(result =>
{
    var value = result.GetValueForOption(portOpt);
    if (value < 1 || value > 65535)
        result.ErrorMessage = "Port must be 1-65535";
});
```

### 4.7. Auto help / completion

```bash
mytool --help        # full help
mytool --version     # version info
mytool init --help   # subcommand-specific
```

System.CommandLine generates help automatically. Также **bash/zsh/PowerShell completions**:

```bash
mytool [tab][tab]   # autocomplete subcommands and options
```

### 4.8. Real CLI example

```csharp
using System.CommandLine;

var rootCommand = new RootCommand("Database backup tool");

// connect subcommand
var connectCmd = new Command("connect", "Test DB connection");
var connStrOpt = new Option<string>("--connection-string") { IsRequired = true };
connectCmd.AddOption(connStrOpt);
connectCmd.SetHandler(async cs => await TestConnection(cs), connStrOpt);

// backup subcommand
var backupCmd = new Command("backup", "Backup database");
backupCmd.AddOption(connStrOpt);
var outputOpt = new Option<DirectoryInfo>("--output") { IsRequired = true };
backupCmd.AddOption(outputOpt);
var compressOpt = new Option<bool>(["--compress", "-c"]);
backupCmd.AddOption(compressOpt);
backupCmd.SetHandler(async (cs, output, compress) =>
{
    await BackupDatabase(cs, output, compress);
}, connStrOpt, outputOpt, compressOpt);

rootCommand.AddCommand(connectCmd);
rootCommand.AddCommand(backupCmd);

return await rootCommand.InvokeAsync(args);

async Task TestConnection(string connStr) { /* ... */ }
async Task BackupDatabase(string connStr, DirectoryInfo output, bool compress) { /* ... */ }
```

> [!question]- Интервью: что такое System.CommandLine?
> Microsoft **official CLI parser** — replacement для command-line argument parsing manual code. Provides: 1) **`RootCommand`** + nested `Command` (subcommands). 2) **`Argument<T>`** (positional, required default) и **`Option<T>`** (named, optional default). 3) **Built-in conversions** — int, FileInfo, DirectoryInfo, Uri, enum, arrays. 4) **Auto-generated help** (`--help`). 5) **Bash/zsh/PowerShell completions** generation. 6) **Validation** через `AddValidator`. 7) **Async handlers**. **Status**: long beta (2024) but production-ready. Replaces older CommandLineParser. Best practice 2024+: **always System.CommandLine** для new CLIs.

---

## 5. Spectre.Console — beautiful output

### 5.1. Установка

```xml
<PackageReference Include="Spectre.Console" Version="0.49.1" />
```

Best in class .NET console UI library.

### 5.2. Markup и colors

```csharp
using Spectre.Console;

AnsiConsole.MarkupLine("[red]Error:[/] something went wrong");
AnsiConsole.MarkupLine("[green]✓[/] Task completed");
AnsiConsole.MarkupLine("[yellow on blue]Highlighted[/] text");
AnsiConsole.MarkupLine("[bold underline]Important![/]");
```

`[color]text[/]` — ANSI markup. Auto-fallback на terminals без color support.

### 5.3. Tables

```csharp
var table = new Table();
table.AddColumn("Name");
table.AddColumn("Age");
table.AddColumn(new TableColumn("Email").Centered());

table.AddRow("Alice", "30", "a@x.com");
table.AddRow("Bob", "25", "b@x.com");
table.AddRow("[red]Charlie[/]", "[yellow]?[/]", "[grey]N/A[/]");

AnsiConsole.Write(table);
```

Output — pretty Unicode tables с borders, alignment.

### 5.4. Progress

```csharp
await AnsiConsole.Progress()
    .StartAsync(async ctx =>
    {
        var task1 = ctx.AddTask("Downloading", maxValue: 100);
        var task2 = ctx.AddTask("Processing", maxValue: 50);
        
        while (!ctx.IsFinished)
        {
            task1.Increment(2);
            task2.Increment(1);
            await Task.Delay(100);
        }
    });
```

Multi-progress bars, spinners, percentages.

### 5.5. Prompts (interactive)

```csharp
var name = AnsiConsole.Prompt(
    new TextPrompt<string>("What's your name?")
        .ValidationErrorMessage("[red]Name required[/]")
        .Validate(name => name.Length > 0));

var age = AnsiConsole.Prompt(
    new TextPrompt<int>("Age?")
        .Validate(a => a >= 0 ? ValidationResult.Success() : ValidationResult.Error("Must be positive")));

var color = AnsiConsole.Prompt(
    new SelectionPrompt<string>()
        .Title("Favorite color?")
        .AddChoices("Red", "Green", "Blue"));

var hobbies = AnsiConsole.Prompt(
    new MultiSelectionPrompt<string>()
        .Title("Hobbies?")
        .AddChoices("Reading", "Gaming", "Hiking", "Cooking"));

var password = AnsiConsole.Prompt(
    new TextPrompt<string>("Password?")
        .Secret());
```

Rich interactive UI без external dependencies.

### 5.6. Tree / hierarchical

```csharp
var root = new Tree("Root");
var child1 = root.AddNode("Child 1");
child1.AddNode("Grandchild 1.1");
child1.AddNode("Grandchild 1.2");
root.AddNode("Child 2");

AnsiConsole.Write(root);
```

### 5.7. Bar chart / live displays

```csharp
AnsiConsole.Write(new BarChart()
    .Width(60)
    .Label("[green bold]Revenue[/]")
    .CenterLabel()
    .AddItem("Jan", 100, Color.Yellow)
    .AddItem("Feb", 150, Color.Green)
    .AddItem("Mar", 80, Color.Red));

// Live updating display
AnsiConsole.Live(new Panel("Initial"))
    .StartAsync(async ctx =>
    {
        for (int i = 0; i < 10; i++)
        {
            await Task.Delay(500);
            ctx.UpdateTarget(new Panel($"Update {i}"));
        }
    });
```

### 5.8. Status / spinner

```csharp
await AnsiConsole.Status()
    .Spinner(Spinner.Known.Star)
    .StartAsync("Working...", async ctx =>
    {
        await Task.Delay(1000);
        ctx.Status("Phase 2...");
        ctx.Spinner(Spinner.Known.Earth);
        await Task.Delay(1000);
    });
```

> [!question]- Интервью: что такое Spectre.Console?
> Best-in-class **.NET console UI library**. Provides: 1) **Markup**: `[red]text[/]` ANSI colors с fallback. 2) **Tables**: Unicode borders, alignment, column types. 3) **Progress**: multi-task bars, spinners. 4) **Prompts**: TextPrompt (validation, secret), SelectionPrompt, MultiSelectionPrompt. 5) **Tree**, **BarChart**, **Live displays**. 6) **Cross-platform**: auto-detects terminal capabilities. Open-source, MIT. Used widely в .NET community. Best practice 2024+: для any production CLI с rich output. Alternatives: Pastel (colors only), ConsoleTables (tables only) — Spectre superset.

---

## 6. Process / shell integration

### 6.1. Process.Start

```csharp
using System.Diagnostics;

var psi = new ProcessStartInfo("git", "status")
{
    RedirectStandardOutput = true,
    RedirectStandardError = true,
    UseShellExecute = false
};

using var process = Process.Start(psi)!;
string output = await process.StandardOutput.ReadToEndAsync();
string error = await process.StandardError.ReadToEndAsync();
await process.WaitForExitAsync();

Console.WriteLine($"Exit code: {process.ExitCode}");
Console.WriteLine(output);
```

### 6.2. CliWrap library

```xml
<PackageReference Include="CliWrap" Version="3.6.6" />
```

```csharp
using CliWrap;

var result = await Cli.Wrap("git")
    .WithArguments("status --porcelain")
    .ExecuteBufferedAsync();

Console.WriteLine($"Exit: {result.ExitCode}");
Console.WriteLine(result.StandardOutput);
```

CliWrap — fluent API над Process, much cleaner.

### 6.3. Pipelines между processes

```csharp
// Эквивалент: cat file | grep "TODO" | wc -l
var result = await Cli.Wrap("cat")
    .WithArguments("file.txt")
    .ExecuteBufferedAsync()
    | Cli.Wrap("grep")
        .WithArguments("TODO")
    | Cli.Wrap("wc")
        .WithArguments("-l");
```

CliWrap supports pipe operator.

### 6.4. Streaming output

```csharp
await foreach (var line in Cli.Wrap("ls")
    .WithArguments("-la")
    .ListenAsync())
{
    if (line is StandardOutputCommandEvent stdout)
        Console.WriteLine($"[OUT] {stdout.Text}");
    if (line is StandardErrorCommandEvent stderr)
        Console.WriteLine($"[ERR] {stderr.Text}");
}
```

Real-time output streaming для long-running commands.

### 6.5. Environment variables

```csharp
var result = await Cli.Wrap("docker")
    .WithArguments("ps")
    .WithEnvironmentVariables(env => env
        .Set("DOCKER_HOST", "tcp://localhost:2375")
        .Set("DOCKER_TLS_VERIFY", "1"))
    .ExecuteBufferedAsync();
```

### 6.6. Working directory

```csharp
var result = await Cli.Wrap("git")
    .WithArguments("log --oneline")
    .WithWorkingDirectory("/home/user/project")
    .ExecuteBufferedAsync();
```

> [!question]- Интервью: чем CliWrap лучше Process.Start?
> 1) **Fluent API** — chained `.WithArguments()`, `.WithWorkingDirectory()`, `.WithEnvironmentVariables()`. 2) **Async-first** — `await Cli.Wrap(...).ExecuteAsync()`. 3) **Pipelines** — pipe operator (`|`) между processes. 4) **Streaming** — `await foreach` для real-time output. 5) **Buffered результат** — `ExecuteBufferedAsync()` returns stdout/stderr/exit code. 6) **Configuration** — timeouts, cancellation, validation. 7) **Cross-platform** — Windows/Linux/macOS. **Process.Start**: low-level, manual stream reading, no pipelines, easy to mishandle. Best practice 2024+: CliWrap для anything beyond trivial.

---

## 7. Native AOT для CLI

### 7.1. Project setup

```xml
<PropertyGroup>
  <OutputType>Exe</OutputType>
  <TargetFramework>net8.0</TargetFramework>
  <PublishAot>true</PublishAot>
  <InvariantGlobalization>true</InvariantGlobalization>
  <SelfContained>true</SelfContained>
</PropertyGroup>
```

### 7.2. Publish

```bash
dotnet publish -c Release -r win-x64    # Windows
dotnet publish -c Release -r linux-x64  # Linux
dotnet publish -c Release -r osx-x64    # macOS
```

Result: **single binary**, no runtime dependency, ~10-15 MB.

### 7.3. Performance

```
Cold start:
- Regular .NET: 100-300ms
- Native AOT: 10-30ms (≈ Go)

Memory:
- Regular .NET: 30-50 MB working set
- Native AOT: 5-15 MB

Binary size:
- Regular .NET (self-contained): 60-80 MB
- Native AOT: 10-15 MB
```

Critical для CLIs (called часто, expected to start fast).

### 7.4. Limitations

```
❌ No reflection (mostly)
❌ No dynamic code generation
❌ No JIT
❌ Some libraries не AOT-compatible (analyzer warns)
❌ Larger build time

✅ System.CommandLine — AOT-compatible
✅ Spectre.Console — AOT-compatible
✅ System.Text.Json (с source generators) — AOT-compatible
✅ HttpClient — AOT-compatible
```

### 7.5. Trim warnings

```bash
dotnet publish -c Release -r linux-x64 --self-contained -p:PublishAot=true
# Watch для:
# warning IL2026: Member 'X' which has 'RequiresUnreferencedCodeAttribute'
# warning IL3050: Generic methods used with reflection
```

Build пытается обнаружить unsafe-for-AOT patterns. Address via:
- Use source generators (JsonSerializerContext).
- Avoid reflection-heavy libraries.
- Mark public surface explicit.

### 7.6. Source generators essential

```csharp
// AOT-friendly JSON
[JsonSerializable(typeof(MyDto))]
public partial class MyJsonContext : JsonSerializerContext { }

var dto = JsonSerializer.Deserialize<MyDto>(json, MyJsonContext.Default.MyDto);
```

```csharp
// AOT-friendly P/Invoke
[LibraryImport("user32.dll", StringMarshalling = StringMarshalling.Utf16)]
static partial int MessageBox(IntPtr hWnd, string text, string caption, uint type);
```

См. [[source-generators]].

### 7.7. Distribution

```
Single binary deployment:
1. dotnet publish -c Release -r linux-x64 -p:PublishAot=true
2. ls bin/Release/net8.0/linux-x64/publish/
   → mytool (single file ~12 MB)
3. scp mytool server:/usr/local/bin/
4. ssh server 'mytool --help'

No runtime install needed!
```

> [!question]- Интервью: почему Native AOT критичен для CLI tools?
> 1) **Cold start** — regular .NET 100-300ms, AOT 10-30ms. CLI calls в loops, scripts, CI/CD — каждый ms matters. 2) **Distribution** — single binary, no .NET runtime install. Like Go/Rust. 3) **Memory footprint** — 5-15 MB vs 30-50 MB. Better для resource-constrained environments. 4) **Predictable performance** — no JIT warmup, no JIT compilation pauses. **Trade-offs**: 1) Reflection limited (use source generators). 2) Some libraries incompatible (analyzer warnings). 3) Larger build time. 4) Bigger binary (10-15 MB vs ~1 MB Bash script). Best practice 2024+: AOT для distributed CLI tools, regular .NET для dev/internal scripts.

---

## 8. Hosting / DI / Configuration

### 8.1. Generic Host

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Configuration;

var builder = Host.CreateApplicationBuilder(args);

// Configuration
builder.Configuration.AddJsonFile("appsettings.json", optional: true);
builder.Configuration.AddEnvironmentVariables(prefix: "MYAPP_");

// DI
builder.Services.AddSingleton<IMyService, MyService>();
builder.Services.AddTransient<IRepository, Repository>();

// Logging
builder.Logging.AddConsole();

var host = builder.Build();
var service = host.Services.GetRequiredService<IMyService>();
await service.RunAsync(args);
```

Same DI/configuration patterns как ASP.NET Core. Suitable для complex CLIs.

### 8.2. appsettings.json

```json
{
  "Database": {
    "ConnectionString": "Server=localhost;..."
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  }
}
```

```csharp
var connStr = builder.Configuration["Database:ConnectionString"];
// Or strongly-typed
var dbConfig = builder.Configuration.GetSection("Database").Get<DatabaseConfig>();
```

### 8.3. CLI + Hosting integration

```csharp
var builder = Host.CreateApplicationBuilder(args);
// ... setup DI

var rootCommand = new RootCommand("Tool");
var cmd = new Command("process", "Process data");

cmd.SetHandler(async () =>
{
    using var host = builder.Build();
    var service = host.Services.GetRequiredService<IDataProcessor>();
    await service.ProcessAsync();
});

rootCommand.AddCommand(cmd);
await rootCommand.InvokeAsync(args);
```

Combine System.CommandLine + Hosting для production CLIs.

### 8.4. Config sources priority

```
Default order (later wins):
1. appsettings.json
2. appsettings.{Environment}.json
3. User secrets (development)
4. Environment variables
5. Command-line arguments
```

CLI flags override config file — standard 12-factor pattern.

> [!question]- Интервью: зачем DI / Hosting в CLI tool?
> 1) **Testability** — services injected, easy mock в tests. 2) **Configuration** — appsettings.json + env vars + CLI flags. 3) **Logging** — `ILogger<T>` structured, multiple sinks. 4) **Lifetime management** — singleton vs transient services. 5) **Async resources** — IHostedService для background tasks. 6) **Reuse patterns** — same setup как ASP.NET Core. **Overkill для**: simple scripts (< 100 lines), one-off tools. **Use для**: production CLIs, multi-command tools, complex business logic. Pattern: `Host.CreateApplicationBuilder(args)` + DI registration + handler resolves services.

---

## 9. Best practices

### 9.1. CLI design

- ✅ **Subcommands** для multiple operations (`tool init`, `tool stats`).
- ✅ **`--help` complete** для каждой subcommand.
- ✅ **Exit codes** — 0 success, non-zero error.
- ✅ **stderr для errors**, stdout для data (pipe-friendly).
- ✅ **Verbose / quiet flags** (`-v`, `-q`).
- ✅ **Version flag** (`--version`).
- ❌ Interactive prompts по default (hard для scripting).

### 9.2. Argument parsing

- ✅ **System.CommandLine** для production.
- ✅ **Validation** в parser, не в handler.
- ✅ **Type-safe options** (`Argument<int>`, `Option<DirectoryInfo>`).
- ✅ **Aliases** для frequently used (`--force` / `-f`).
- ❌ **Manual `args[0]`/`args[1]` parsing** для anything non-trivial.

### 9.3. Output

- ✅ **Spectre.Console** для rich output.
- ✅ **JSON output mode** (`--json`) для machine-readable.
- ✅ **Color detection** auto (TERM env var, NO_COLOR convention).
- ✅ **Progress bars** для long operations.
- ❌ Mixing stdout (data) и progress in same stream.

### 9.4. Native AOT

- ✅ **Production CLIs** — always AOT когда возможно.
- ✅ **Source generators** для JSON/Regex/Logging.
- ✅ **Test AOT build** в CI.
- ❌ Reflection-heavy code в AOT (analyzer warnings).
- ❌ Libraries с RequiresUnreferencedCode attributes.

### 9.5. Distribution

- ✅ **Single binary** (Native AOT).
- ✅ **GitHub Releases** + checksums.
- ✅ **Homebrew / Chocolatey / winget** для package managers.
- ✅ **Auto-update** через built-in mechanism (optional).
- ❌ Distribute как .csproj (требует .NET SDK).

### 9.6. Не делай

- ❌ Hardcoded paths (используй CLI arguments).
- ❌ Crash без exit code на errors.
- ❌ Sync I/O в async handlers (deadlock potential).
- ❌ Console.ReadLine() в non-interactive scripts (hangs).

---

## 10. Decision tree

```
Что строишь?
│
├── One-off script / automation
│   ├── Simple → Bash / .csx (если C# preferable)
│   ├── С NuGet packages → dotnet script (.csx)
│   └── Quick prototype → top-level statements
│
├── Production CLI tool
│   ├── Argument parsing → System.CommandLine
│   ├── Rich output → Spectre.Console
│   ├── DI / config → Microsoft.Extensions.Hosting
│   ├── Distribution → Native AOT publish
│   └── Multi-platform → publish per RID
│
├── Glue between processes
│   ├── Shell pipes → CliWrap
│   ├── Pipe-friendly → stdout data, stderr logs
│   └── Streaming output → CliWrap.ListenAsync
│
└── DevOps / CI tooling
    ├── Cross-platform → AOT single binary
    ├── Fast startup → AOT critical
    └── No runtime deps → AOT or `dotnet tool` install
```

---

## 11. Cheat sheet

```csharp
// === Top-level (Program.cs) ===
Console.WriteLine("Hello!");
return args.Length > 0 ? 0 : 1;

// === dotnet script (.csx) ===
#!/usr/bin/env dotnet-script
#r "nuget: Spectre.Console, 0.49.1"
using Spectre.Console;
AnsiConsole.MarkupLine("[green]Running[/]");

// === System.CommandLine ===
using System.CommandLine;

var nameOpt = new Option<string>("--name") { IsRequired = true };
var verboseOpt = new Option<bool>(["--verbose", "-v"]);
var rootCmd = new RootCommand("Tool") { nameOpt, verboseOpt };

rootCmd.SetHandler((name, verbose) =>
{
    if (verbose) Console.WriteLine($"[verbose] Greeting {name}");
    Console.WriteLine($"Hello, {name}!");
}, nameOpt, verboseOpt);

return await rootCmd.InvokeAsync(args);

// === Subcommands ===
var initCmd = new Command("init", "Initialize");
var pathArg = new Argument<DirectoryInfo>("path");
initCmd.AddArgument(pathArg);
initCmd.SetHandler(p => Console.WriteLine($"Init {p.FullName}"), pathArg);
rootCmd.AddCommand(initCmd);

// === Spectre.Console ===
using Spectre.Console;

AnsiConsole.MarkupLine("[red]Error[/]");
AnsiConsole.Write(new Table()
    .AddColumn("Name").AddColumn("Value")
    .AddRow("Alice", "30"));

await AnsiConsole.Progress().StartAsync(async ctx =>
{
    var task = ctx.AddTask("Working", maxValue: 100);
    while (!task.IsFinished) { task.Increment(1); await Task.Delay(10); }
});

var name = AnsiConsole.Prompt(new TextPrompt<string>("Name?"));
var color = AnsiConsole.Prompt(new SelectionPrompt<string>().AddChoices("Red", "Green", "Blue"));

// === CliWrap ===
using CliWrap;

var result = await Cli.Wrap("git")
    .WithArguments("status --porcelain")
    .WithWorkingDirectory("/path")
    .ExecuteBufferedAsync();

Console.WriteLine(result.StandardOutput);

// === Native AOT csproj ===
// <PropertyGroup>
//   <PublishAot>true</PublishAot>
//   <InvariantGlobalization>true</InvariantGlobalization>
// </PropertyGroup>
//
// dotnet publish -c Release -r linux-x64

// === Hosting + DI ===
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection;

var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddSingleton<IMyService, MyService>();
using var host = builder.Build();
var svc = host.Services.GetRequiredService<IMyService>();
await svc.RunAsync();
```

---

## 12. Common pitfalls

### 12.1. Top-level в multiple files

```csharp
// Program.cs
Console.WriteLine("First");

// Other.cs — top-level в same project
Console.WriteLine("Second");   // ❌ Compile error
```

**Фикс:** только один файл с top-level в проекте.

### 12.2. Console.ReadLine() hangs в CI

```csharp
var name = Console.ReadLine();   // ❌ В non-interactive CI hangs forever
```

**Фикс:** check `Console.IsInputRedirected` или CLI flag.

### 12.3. Sync I/O в async handler

```csharp
cmd.SetHandler(() =>
{
    var data = httpClient.GetStringAsync(url).Result;   // ❌ deadlock potential
});
```

**Фикс:** async handler:
```csharp
cmd.SetHandler(async () =>
{
    var data = await httpClient.GetStringAsync(url);
});
```

### 12.4. AOT с reflection

```csharp
// ❌ JsonSerializer reflection не AOT-friendly
JsonSerializer.Serialize(obj);
```

**Фикс:** `JsonSerializerContext` source generator.

### 12.5. Manual args[N] parsing

```csharp
var name = args[0];   // ❌ crash если no args
var age = int.Parse(args[1]);   // ❌ no validation
```

**Фикс:** System.CommandLine с required options.

### 12.6. Mixing stdout и stderr

```csharp
Console.WriteLine("Processing file...");   // ❌ goes to stdout
Console.WriteLine(JsonSerializer.Serialize(data));   // ❌ progress + data mixed
```

**Фикс:** progress to stderr, data to stdout:
```csharp
Console.Error.WriteLine("Processing...");
Console.WriteLine(JsonSerializer.Serialize(data));
```

### 12.7. Не handle Ctrl+C

```csharp
// Long operation без cancellation
await DoLongTask();   // ❌ Ctrl+C аborts immediately, no cleanup
```

**Фикс:** CancellationToken + handler:
```csharp
using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (s, e) => { e.Cancel = true; cts.Cancel(); };
await DoLongTask(cts.Token);
```

### 12.8. Hardcoded paths

```csharp
var data = File.ReadAllText("C:\\Users\\Vitaly\\data.txt");   // ❌
```

**Фикс:** CLI argument or working directory.

### 12.9. Exit code 0 on error

```csharp
try { await Process(); }
catch (Exception ex)
{
    Console.WriteLine(ex.Message);   // ❌ exit 0 — CI thinks success!
}
```

**Фикс:** `return 1` или `Environment.Exit(1)` after error.

### 12.10. dotnet script slow startup

```csharp
// .csx file with many #r directives
// Cold start 3-5s — acceptable для one-off, slow для CI
```

**Фикс:** для frequent execution — compile to console project + AOT.

> [!question]- Интервью: топ-3 ошибки в C# CLI tools?
> 1) **Manual `args[N]` parsing** — crashes без validation, no help, no completions. Always System.CommandLine. 2) **Console.ReadLine() в non-interactive** — hangs CI/scripts. Check `IsInputRedirected` или use CLI flag. 3) **Exit code 0 на error** — CI thinks success, missed failures. Always `return non-zero` после error. Бонус: Mixing stdout (data) и stderr (logs) — pipe ломается, can't grep output. stderr для progress/logs, stdout для data only.

---

## 13. Practice exercises

### 13.1. Простой file stats CLI

```csharp
// Program.cs
using System.CommandLine;
using Spectre.Console;

var rootCmd = new RootCommand("File stats tool");

var pathArg = new Argument<DirectoryInfo>(
    "path",
    () => new DirectoryInfo(Environment.CurrentDirectory),
    "Directory to analyze");

var patternOpt = new Option<string>(
    name: "--pattern",
    getDefaultValue: () => "*",
    description: "File pattern");

var recursiveOpt = new Option<bool>(["--recursive", "-r"], "Search recursively");

rootCmd.AddArgument(pathArg);
rootCmd.AddOption(patternOpt);
rootCmd.AddOption(recursiveOpt);

rootCmd.SetHandler((path, pattern, recursive) =>
{
    var searchOption = recursive ? SearchOption.AllDirectories : SearchOption.TopDirectoryOnly;
    var files = path.GetFiles(pattern, searchOption);
    
    var totalSize = files.Sum(f => f.Length);
    var avgSize = files.Length > 0 ? totalSize / files.Length : 0;
    
    var table = new Table();
    table.AddColumn("Metric");
    table.AddColumn("Value");
    
    table.AddRow("Path", path.FullName);
    table.AddRow("Pattern", pattern);
    table.AddRow("Recursive", recursive.ToString());
    table.AddRow("File count", files.Length.ToString("N0"));
    table.AddRow("Total size", $"{totalSize / 1024.0 / 1024.0:F2} MB");
    table.AddRow("Average size", $"{avgSize / 1024.0:F2} KB");
    
    AnsiConsole.Write(table);
}, pathArg, patternOpt, recursiveOpt);

return await rootCmd.InvokeAsync(args);
```

### 13.2. Multi-command Git wrapper

```csharp
var rootCmd = new RootCommand("Git helper");

// status
var statusCmd = new Command("status", "Show repo status");
statusCmd.SetHandler(async () =>
{
    var result = await Cli.Wrap("git").WithArguments("status").ExecuteBufferedAsync();
    AnsiConsole.WriteLine(result.StandardOutput);
});

// commit с message validation
var commitCmd = new Command("commit", "Commit with validation");
var msgOpt = new Option<string>(["--message", "-m"]) { IsRequired = true };
msgOpt.AddValidator(r =>
{
    var msg = r.GetValueForOption(msgOpt)!;
    if (msg.Length < 10) r.ErrorMessage = "Message too short (min 10 chars)";
});
commitCmd.AddOption(msgOpt);
commitCmd.SetHandler(async msg =>
{
    await Cli.Wrap("git").WithArguments(["commit", "-m", msg]).ExecuteAsync();
}, msgOpt);

rootCmd.AddCommand(statusCmd);
rootCmd.AddCommand(commitCmd);

return await rootCmd.InvokeAsync(args);
```

### 13.3. Interactive setup wizard

```csharp
using Spectre.Console;

AnsiConsole.MarkupLine("[bold green]Project Setup Wizard[/]");
AnsiConsole.WriteLine();

var name = AnsiConsole.Prompt(
    new TextPrompt<string>("Project [green]name[/]?")
        .ValidationErrorMessage("[red]Required[/]")
        .Validate(s => !string.IsNullOrWhiteSpace(s)));

var template = AnsiConsole.Prompt(
    new SelectionPrompt<string>()
        .Title("Project [green]template[/]?")
        .AddChoices("Console", "Web API", "Library", "Worker"));

var features = AnsiConsole.Prompt(
    new MultiSelectionPrompt<string>()
        .Title("Add [green]features[/]?")
        .AddChoices("Docker", "GitHub Actions", "Tests", "Documentation"));

AnsiConsole.WriteLine();
AnsiConsole.MarkupLine($"Creating [bold]{template}[/] project [green]{name}[/]...");

if (features.Any())
    AnsiConsole.MarkupLine($"With features: [yellow]{string.Join(", ", features)}[/]");

await AnsiConsole.Progress().StartAsync(async ctx =>
{
    var task = ctx.AddTask("Creating project", maxValue: 100);
    while (!task.IsFinished)
    {
        task.Increment(5);
        await Task.Delay(50);
    }
});

AnsiConsole.MarkupLine($"[green]✓[/] Project created!");
```

---

## 14. Что читать дальше

1. **[[source-generators|Source Generators]]** — JSON/Regex AOT.
2. **[[unsafe-pointers|Unsafe / AOT]]** — AOT-friendly patterns.
3. **System.CommandLine documentation**.
4. **Spectre.Console docs** — full UI capabilities.
5. **Native AOT deep** — Microsoft docs.

---

## 15. См. также

- [[source-generators|Source Generators]] — AOT compatibility
- [[unsafe-pointers|Unsafe / AOT]] — Native AOT
- [[modern-features|Modern Features]] — top-level statements
- System.CommandLine — github.com/dotnet/command-line-api
- Spectre.Console — spectreconsole.net
- CliWrap — github.com/Tyrrrz/CliWrap

---

## 16. Reading list

- **Microsoft Docs — Top-level statements** — learn.microsoft.com/dotnet/csharp/fundamentals/program-structure/top-level-statements
- **Microsoft Docs — System.CommandLine** — learn.microsoft.com/dotnet/standard/commandline/
- **Microsoft Docs — Native AOT** — learn.microsoft.com/dotnet/core/deploying/native-aot/
- **Microsoft Docs — Generic Host** — learn.microsoft.com/dotnet/core/extensions/generic-host
- **Spectre.Console docs** — spectreconsole.net
- **dotnet-script repo** — github.com/dotnet-script/dotnet-script
- **CliWrap docs** — github.com/Tyrrrz/CliWrap
- **Patrick Smacchia — CLI tooling** — blog.ndepend.com
- **Andrew Lock — System.CommandLine series** — andrewlock.net
- **Dave Glick — Spectre.Console deep** — daveaglick.com
