---
tags: [dotnet, cli, junior, getting-started, sdk, project-templates, basics]
level: Junior
date: 2026-04-30
---

# .NET CLI — getting started для Junior

> **Старт с .NET без Visual Studio.** dotnet CLI — основной инструмент: создание проектов, сборка, run, test, package management. Closes пробел "видел dotnet команды в туториалах, но не понимаю что они делают".

---

## Что это, зачем и когда

### Что такое .NET CLI

`dotnet` — command-line tool который умеет всё:
- Создать проект (`dotnet new`)
- Восстановить пакеты (`dotnet restore`)
- Скомпилировать (`dotnet build`)
- Запустить (`dotnet run`)
- Прогнать тесты (`dotnet test`)
- Создать пакет (`dotnet pack`)
- Опубликовать (`dotnet publish`)

### Зачем уметь

- **Visual Studio не везде** — на Linux/macOS используют CLI + VS Code / Rider
- **CI/CD** — все pipelines используют CLI
- **Quick scripts** — dotnet-script для одноразовых задач
- **Базовое понимание** — что Visual Studio делает за тебя

### Установка

```bash
# Проверить что .NET установлен
dotnet --version
# 10.0.100  (или другая версия)

# Список installed SDKs
dotnet --list-sdks

# Список runtimes
dotnet --list-runtimes
```

Если не установлен — [dotnet.microsoft.com/download](https://dotnet.microsoft.com/download) или:
```bash
# Windows
winget install Microsoft.DotNet.SDK.10

# macOS
brew install dotnet-sdk

# Linux (Ubuntu)
sudo apt install dotnet-sdk-10.0
```

---

## 1. Первый проект — `dotnet new`

### Создать console app

```bash
mkdir MyFirstApp
cd MyFirstApp

dotnet new console
```

### Что создалось

```
MyFirstApp/
├── MyFirstApp.csproj    # настройки проекта
├── Program.cs           # main файл
└── obj/                 # внутренние файлы (gitignored)
```

**Program.cs** — top-level statements style (.NET 6+):

```csharp
// See https://aka.ms/new-console-template
Console.WriteLine("Hello, World!");
```

**MyFirstApp.csproj:**

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```

### Запустить

```bash
dotnet run
# Output: Hello, World!
```

`dotnet run` делает: restore → build → execute.

---

## 2. Templates — что можно создать

### Самые частые

```bash
dotnet new console              # console app
dotnet new web                  # ASP.NET Core minimal API
dotnet new webapi               # ASP.NET Core Web API (controllers)
dotnet new mvc                  # ASP.NET Core MVC
dotnet new blazor               # Blazor app
dotnet new classlib             # class library
dotnet new xunit                # xUnit test project
dotnet new nunit                # NUnit test project
dotnet new wpf                  # WPF app (Windows only)
dotnet new winforms             # WinForms (Windows only)
dotnet new maui                 # MAUI cross-platform
dotnet new console -lang F#     # F# instead of C#
dotnet new sln                  # пустой solution file
dotnet new gitignore            # .gitignore для .NET
dotnet new editorconfig         # .editorconfig
```

### Полный список

```bash
dotnet new list
```

### С параметрами

```bash
# Project name
dotnet new console -n MyApp

# Output directory  
dotnet new console -o ./apps/MyApp

# Override .NET version
dotnet new console --framework net10.0
dotnet new console -f net8.0
```

---

## 3. Case Study #1 — Создать API проект с нуля

### Задача

Создать ASP.NET Core Web API project с structure:
- Solution `OrderApp`
- API project `OrderApp.Api`
- Domain library `OrderApp.Domain`
- Tests `OrderApp.Tests`

### Шаги

```bash
# 1. Создать папку и solution
mkdir OrderApp
cd OrderApp

dotnet new sln -n OrderApp

# 2. Создать API project
dotnet new webapi -n OrderApp.Api -o src/OrderApp.Api

# 3. Создать Domain library
dotnet new classlib -n OrderApp.Domain -o src/OrderApp.Domain

# 4. Создать Tests
dotnet new xunit -n OrderApp.Tests -o tests/OrderApp.Tests

# 5. Добавить проекты в solution
dotnet sln add src/OrderApp.Api/OrderApp.Api.csproj
dotnet sln add src/OrderApp.Domain/OrderApp.Domain.csproj
dotnet sln add tests/OrderApp.Tests/OrderApp.Tests.csproj

# 6. API ссылается на Domain
dotnet add src/OrderApp.Api/OrderApp.Api.csproj reference src/OrderApp.Domain/OrderApp.Domain.csproj

# 7. Tests ссылается на оба
dotnet add tests/OrderApp.Tests/OrderApp.Tests.csproj reference src/OrderApp.Domain/OrderApp.Domain.csproj
dotnet add tests/OrderApp.Tests/OrderApp.Tests.csproj reference src/OrderApp.Api/OrderApp.Api.csproj

# 8. Добавить .gitignore
dotnet new gitignore

# 9. Запустить API
cd src/OrderApp.Api
dotnet run
# https://localhost:5001 — Swagger UI
```

### Финальная структура

```
OrderApp/
├── OrderApp.sln
├── .gitignore
├── src/
│   ├── OrderApp.Api/
│   │   ├── Program.cs
│   │   ├── Controllers/
│   │   └── OrderApp.Api.csproj
│   └── OrderApp.Domain/
│       ├── Class1.cs
│       └── OrderApp.Domain.csproj
└── tests/
    └── OrderApp.Tests/
        ├── UnitTest1.cs
        └── OrderApp.Tests.csproj
```

---

## 4. Управление пакетами

### Добавить NuGet package

```bash
# По имени
dotnet add package Newtonsoft.Json

# Конкретная версия
dotnet add package Newtonsoft.Json --version 13.0.3

# В конкретный проект
dotnet add src/OrderApp.Api package EntityFrameworkCore --version 10.0.0
```

Что делает: добавляет `<PackageReference>` в csproj.

```xml
<ItemGroup>
  <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
</ItemGroup>
```

### Удалить package

```bash
dotnet remove package Newtonsoft.Json
```

### Список packages

```bash
# Текущие packages
dotnet list package

# С устаревшими
dotnet list package --outdated

# С vulnerabilities
dotnet list package --vulnerable
```

### Restore — скачать packages

```bash
dotnet restore
```

Обычно происходит автоматически при `build`/`run`. Manual нужен:
- После клонирования repo
- В Docker images (отдельный layer)
- В CI/CD

### Часто встречающиеся packages

```bash
# JSON
dotnet add package Newtonsoft.Json
# Или встроенный System.Text.Json

# Database
dotnet add package Microsoft.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Npgsql.EntityFrameworkCore.PostgreSQL

# HTTP
dotnet add package Microsoft.Extensions.Http

# Logging
dotnet add package Serilog.AspNetCore

# Testing
dotnet add package xunit
dotnet add package Moq
dotnet add package FluentAssertions

# CQRS
dotnet add package MediatR

# Validation
dotnet add package FluentValidation

# Docs
dotnet add package Swashbuckle.AspNetCore   # Swagger
```

---

## 5. Build / run / publish

### `dotnet build`

```bash
dotnet build
# Build всё solution или current project
```

Что делает:
1. Restore packages если нужно
2. Compile в IL (DLL)
3. Output в `bin/Debug/net10.0/`

```bash
# Release build (optimized)
dotnet build -c Release
# Output: bin/Release/net10.0/
```

### `dotnet run`

```bash
dotnet run
# Build + execute
```

Параметры:

```bash
# Передать args
dotnet run -- arg1 arg2

# Конкретный проект
dotnet run --project src/OrderApp.Api/OrderApp.Api.csproj

# Конкретная конфигурация
dotnet run -c Release

# Hot reload (.NET 6+)
dotnet watch run
# При изменении файлов — автоматический rebuild + restart
```

### `dotnet publish` — для deploy

```bash
# Создать deployable bundle
dotnet publish -c Release -o ./publish
```

Что в `./publish`:
- Все DLL'ки + dependencies
- appsettings.json
- runtime config

Можно деплоить — копировать в server или Docker.

### Self-contained publish

```bash
# Включает .NET runtime — не нужен на target machine
dotnet publish -c Release -r linux-x64 --self-contained -o ./publish

# Single executable file
dotnet publish -c Release -r win-x64 --self-contained -p:PublishSingleFile=true

# Trimmed (меньше размер, .NET 8+)
dotnet publish -c Release -r linux-x64 --self-contained -p:PublishTrimmed=true

# Native AOT (.NET 7+)
dotnet publish -c Release -r linux-x64 -p:PublishAot=true
```

См. [[../AspNetCore/native-aot|Native AOT]].

---

## 6. Testing

### Запустить tests

```bash
# Все tests в solution
dotnet test

# Конкретный test project
dotnet test tests/OrderApp.Tests/

# По filter
dotnet test --filter "FullyQualifiedName~OrderTests"
dotnet test --filter "Category=Integration"

# С coverage
dotnet test --collect:"XPlat Code Coverage"
```

### Detailed output

```bash
dotnet test --logger "console;verbosity=detailed"
```

См. [[../Testing/testing-fundamentals|Testing Fundamentals]].

---

## 7. Case Study #2 — Daily workflow Junior разработчика

### Утро — pull updates

```bash
git pull
dotnet restore         # restore новые packages если есть
dotnet build           # check что compiles
```

### Развитие feature

```bash
# Создать branch
git checkout -b feature/add-search

# Hot reload run
dotnet watch run --project src/OrderApp.Api

# В другом терминале — tests
dotnet watch test --project tests/OrderApp.Tests
```

`dotnet watch` — restart на изменения. Очень удобно.

### Перед commit

```bash
# Format check (если используется dotnet-format)
dotnet format --verify-no-changes

# All tests
dotnet test

# Build clean
dotnet build -c Release
```

### Push

```bash
git add .
git commit -m "Add search endpoint"
git push origin feature/add-search
```

---

## 8. Global tools

`.NET CLI` поддерживает global utility tools.

### Useful tools

```bash
# Format code
dotnet tool install -g dotnet-format
dotnet format

# EF Core migrations
dotnet tool install -g dotnet-ef
dotnet ef migrations add InitialCreate
dotnet ef database update

# Outdated packages checker
dotnet tool install -g dotnet-outdated-tool
dotnet outdated

# Diagnostics
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-dump

# Reportgenerator (coverage reports)
dotnet tool install -g dotnet-reportgenerator-globaltool
```

### Local tools (recommended) — per-project

```bash
# Manifest file (один раз)
dotnet new tool-manifest

# Install local
dotnet tool install dotnet-format
dotnet tool install dotnet-ef

# В .config/dotnet-tools.json — committed в repo
# Все devs получают same versions

# Run
dotnet tool run dotnet-format
# или просто
dotnet format
```

---

## 9. Configuration / appsettings

### `appsettings.json`

ASP.NET Core читает по дефолту:

```json
{
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=mydb;..."
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "Jwt": {
    "Issuer": "myapp",
    "Audience": "users"
  }
}
```

### Read в коде

```csharp
public class MyService
{
    public MyService(IConfiguration config)
    {
        string connStr = config.GetConnectionString("Default");
        string issuer = config["Jwt:Issuer"];
    }
}
```

### Environment-specific

```
appsettings.json              # base
appsettings.Development.json  # development override
appsettings.Production.json   # production override
```

ASP.NET автоматически merge based on `ASPNETCORE_ENVIRONMENT`:

```bash
# Development (по дефолту в IDE)
dotnet run

# Production
ASPNETCORE_ENVIRONMENT=Production dotnet run
```

См. [[../AspNetCore/di-configuration|DI & Configuration]].

---

## 10. Case Study #3 — Решение common problems

### "The type or namespace 'X' could not be found"

```bash
# Возможно missing reference
dotnet add package XPackage

# Или missing using
# В файле: using XNamespace;
```

### "Could not load file or assembly"

```bash
# Restore + rebuild
dotnet clean
dotnet restore
dotnet build
```

### "Unable to find package"

```bash
# Сheck nuget sources
dotnet nuget list source

# Add nuget.org если missing
dotnet nuget add source https://api.nuget.org/v3/index.json --name nuget.org
```

### Старая версия package

```bash
# Update specific
dotnet add package PackageName --version 2.0.0

# Update все packages в проекте (через dotnet-outdated tool)
dotnet outdated
dotnet outdated -u   # update
```

### Multiple SDK versions installed

```bash
# Какой SDK использовать
echo "{ \"sdk\": { \"version\": \"10.0.100\" } }" > global.json

# Force specific version
dotnet --version
```

---

## 11. csproj basics

### Базовая структура

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <!-- NuGet packages -->
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  </ItemGroup>

  <ItemGroup>
    <!-- References на другие проекты -->
    <ProjectReference Include="..\OrderApp.Domain\OrderApp.Domain.csproj" />
  </ItemGroup>

</Project>
```

### Common SDK types

```xml
<Project Sdk="Microsoft.NET.Sdk">                    <!-- console / library -->
<Project Sdk="Microsoft.NET.Sdk.Web">                <!-- ASP.NET Core -->
<Project Sdk="Microsoft.NET.Sdk.BlazorWebAssembly"> <!-- Blazor WASM -->
<Project Sdk="Microsoft.NET.Sdk.Razor">              <!-- Razor class library -->
```

### TargetFramework — какая версия

```xml
<TargetFramework>net10.0</TargetFramework>           <!-- .NET 10 -->
<TargetFramework>net8.0</TargetFramework>            <!-- .NET 8 LTS -->
<TargetFramework>netstandard2.0</TargetFramework>    <!-- многоплатформ -->

<!-- Multi-targeting (создаёт несколько builds) -->
<TargetFrameworks>net8.0;net10.0</TargetFrameworks>
```

### Useful properties

```xml
<PropertyGroup>
  <Nullable>enable</Nullable>                <!-- NRT analyzer -->
  <ImplicitUsings>enable</ImplicitUsings>    <!-- auto using common namespaces -->
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>  <!-- strict -->
  <LangVersion>preview</LangVersion>          <!-- experimental C# features -->
  <RootNamespace>MyApp</RootNamespace>        <!-- override default namespace -->
  <AssemblyName>MyAppExe</AssemblyName>       <!-- override DLL name -->
  <Version>1.2.3</Version>                    <!-- assembly version -->
</PropertyGroup>
```

См. [[../Infrastructure/project-setup|Project Setup]].

---

## 12. Common Pitfalls

### 1. Не запускать `dotnet restore` после клонирования

```bash
git clone repo
dotnet build   # ошибка — нет packages
# ✅
dotnet restore
dotnet build
```

Хотя `build` обычно сам зовёт restore — иногда нужно явно.

### 2. Старая SDK не понимает новый `csproj`

```
error NETSDK1045: The current .NET SDK does not support targeting .NET 10.0
```

✅ Update SDK или используй старый TargetFramework.

### 3. `dotnet run` с другого места

```bash
# Запускаешь из root
cd MySolution
dotnet run            # ❌ "no project found"

# ✅ Указать project
dotnet run --project src/MyApp/MyApp.csproj
# или cd внутрь
cd src/MyApp
dotnet run
```

### 4. Forgot publish для production

```bash
# ❌ Не используй bin/Debug в production
# ❌ Не копируй просто .csproj

# ✅
dotnet publish -c Release -o ./publish
# Всё нужное в ./publish
```

### 5. Missing reference после добавления project

```bash
dotnet sln add new-project   # добавил в solution
# ⚠️ Но другие проекты не видят его!

# ✅ Add reference тоже
dotnet add other-project reference new-project
```

### 6. Hot reload не работает для async / generic changes

`dotnet watch run` rebuild на изменения. Но некоторые changes требуют restart:
- Изменение method signature
- Добавление async
- Изменение base class

→ `dotnet watch run` сам предложит restart, или Ctrl+R.

### 7. Globally installed tools — version conflicts

```bash
# Global — все проекты используют same version
dotnet tool install -g dotnet-ef

# ⚠️ Если разные проекты требуют разные версии EF tools
```

✅ Local tools per-project лучше:
```bash
dotnet new tool-manifest
dotnet tool install dotnet-ef --version 10.0.0
```

### 8. Не gitignore `bin/` и `obj/`

```bash
# .gitignore должен иметь:
bin/
obj/
*.user
```

`dotnet new gitignore` создаёт правильный.

---

## 13. Best Practices

### Project structure

- **Solution + multiple projects** — разделение слоёв
- **`src/`** для production code, **`tests/`** для tests
- **gitignore + editorconfig** обязательно
- **`Directory.Build.props`** для shared settings (см. [[../Infrastructure/project-setup|Project Setup]])

### Workflow

- **`dotnet watch run`** для development
- **`dotnet test`** перед commit
- **Local tools** через manifest file
- **`global.json`** если разные проекты требуют разные SDK

### Packages

- **`dotnet list package --outdated`** периодически
- **`dotnet list package --vulnerable`** для security
- **Pin versions** (не floating) для production
- **NuGet.Config** для private feeds

### Production

- **`dotnet publish -c Release`** не bin/Debug
- **Self-contained** для simplicity deploy
- **Native AOT** для performance + small size (.NET 7+)
- **Multi-stage Dockerfile** — small images

См. [[../Infrastructure/docker|Docker]] и [[../Infrastructure/cicd-github-actions|CI/CD]].

---

## 14. Cheat sheet

| Что нужно | Команда |
|-----------|---------|
| Версия .NET | `dotnet --version` |
| Список SDKs | `dotnet --list-sdks` |
| Создать console app | `dotnet new console` |
| Создать Web API | `dotnet new webapi` |
| Создать solution | `dotnet new sln` |
| Создать xUnit tests | `dotnet new xunit` |
| Добавить проект в solution | `dotnet sln add path/to/project.csproj` |
| Reference другого проекта | `dotnet add reference path/to/other.csproj` |
| Установить package | `dotnet add package PackageName` |
| Установить version | `dotnet add package PackageName --version 1.0.0` |
| Удалить package | `dotnet remove package PackageName` |
| Restore | `dotnet restore` |
| Build | `dotnet build` |
| Build Release | `dotnet build -c Release` |
| Run | `dotnet run` |
| Run конкретный | `dotnet run --project path/proj.csproj` |
| Hot reload | `dotnet watch run` |
| Test | `dotnet test` |
| Test filter | `dotnet test --filter "Category=Integration"` |
| Publish | `dotnet publish -c Release -o ./out` |
| Format | `dotnet format` |
| Self-contained publish | `dotnet publish -c Release -r linux-x64 --self-contained` |
| Native AOT | `dotnet publish -p:PublishAot=true` |
| Install global tool | `dotnet tool install -g toolname` |
| Local tools manifest | `dotnet new tool-manifest` |
| List installed packages | `dotnet list package` |
| Outdated packages | `dotnet list package --outdated` |
| Vulnerable packages | `dotnet list package --vulnerable` |
| Clean | `dotnet clean` |
| `.gitignore` | `dotnet new gitignore` |
| `.editorconfig` | `dotnet new editorconfig` |

---

## 15. См. также

- [[csharp-basics|C# Basics]] — после получения CLI
- [[debugging-basics|Debugging Basics]] — отладка
- [[../Infrastructure/project-setup|Project Setup]] — advanced csproj
- [[../Infrastructure/docker|Docker]] — containerize .NET app
- [[../Infrastructure/cicd-github-actions|CI/CD]] — autoation
- [[../Testing/testing-fundamentals|Testing]] — `dotnet test`
- [[../AspNetCore/di-configuration|DI & Configuration]] — appsettings

## Reading list

- **Microsoft Docs — .NET CLI** — learn.microsoft.com/dotnet/core/tools
- **Microsoft Docs — dotnet new** — learn.microsoft.com/dotnet/core/tools/dotnet-new
- **NuGet documentation** — learn.microsoft.com/nuget
- **dotnet/sdk GitHub** — github.com/dotnet/sdk
- **Andrew Lock blog** — andrewlock.net (.NET CLI tips)
