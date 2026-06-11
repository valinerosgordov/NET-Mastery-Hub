---
tags: [dotnet, cli, junior, getting-started, sdk, project-templates, msbuild]
level: Junior
date: 2026-05-04
---

# .NET CLI — getting started

> **Кросс-платформенный командный интерфейс к .NET.** `dotnet` — единственный обязательный инструмент: создание проектов, restore, build, run, test, publish, pack, tooling. Работает одинаково на Windows / Linux / macOS, лежит в основе всех CI-пайплайнов и заменяет Visual Studio в 90% повседневных задач. Закрывает пробел: «видел `dotnet` команды в туториалах, не понимаю, что они реально делают».

---

## 0. Как читать этот файл

Если ты впервые видишь `dotnet` в терминале — читай разделы 1→4 подряд: установишь SDK, создашь первый проект, поймёшь, что внутри. Если уже создаёшь проекты, но непонятно «как это всё связано» — раздел 4 (структура проекта, MSBuild), 5 (restore изнутри), 6 (build изнутри). Если интересует deploy — раздел 9 (publish с вариантами runtime), 17 (CI/CD recipes).

Все команды копируются в терминал и выполняются. Где видишь `# →` — это ожидаемый вывод. Cross-language якоря (`> [!info]-`) свёрнуты — раскрывай, если переходишь из Node.js / Python / Rust / Go / Java. Interview-вопросы (`> [!question]-`) встроены рядом с теорией.

---

## 1. Что это, зачем и когда

### 1.1. Что такое .NET CLI

**.NET CLI** — это `dotnet`, command-line интерфейс к платформе .NET. Ставится вместе с .NET SDK и предоставляет универсальный набор команд для работы с проектами:

- Создавать проекты из шаблонов — `dotnet new`
- Восстанавливать NuGet-зависимости — `dotnet restore`
- Компилировать — `dotnet build`
- Запускать — `dotnet run`
- Прогонять тесты — `dotnet test`
- Создавать NuGet-пакеты — `dotnet pack`
- Публиковать для деплоя — `dotnet publish`
- Управлять инструментами — `dotnet tool`
- Работать с EF Core, secrets, форматированием — через специализированные tools

CLI — это **обёртка над MSBuild + NuGet client + tooling host**. То есть ты запускаешь `dotnet build`, а внутри:

1. CLI парсит команду и опции.
2. Передаёт работу MSBuild (`Microsoft.Build.dll`).
3. MSBuild читает `.csproj` (это XML-сценарий сборки).
4. Запускает таски: компиляция Roslyn, копирование файлов, NuGet resolution.

Понимать это полезно, потому что любая ошибка `dotnet build` — это, по сути, ошибка MSBuild с конкретной задачей.

### 1.2. Зачем уметь CLI, если есть IDE

Visual Studio (Windows) или Rider — мощные IDE с GUI для всех CLI-операций. Но CLI обязателен, потому что:

1. **Кроссплатформенность.** Linux/macOS — нет VS, разработчики работают с CLI + VS Code / Rider.
2. **CI/CD.** Любой автоматический pipeline (GitHub Actions, GitLab CI, Azure DevOps, Jenkins) — это последовательность CLI-команд. Не зная CLI, ты не настроишь pipeline.
3. **Docker.** Все официальные .NET-образы — `mcr.microsoft.com/dotnet/sdk` — внутри только CLI, без IDE.
4. **Воспроизводимость.** Команда `dotnet test` даст одинаковый результат на машине разработчика и на CI. Кнопка «Run All Tests» в VS — нет.
5. **Quick scripts.** `dotnet-script`, `dotnet run` для одиночных файлов в .NET 11+ — заменяют bash-скрипты для одноразовых задач.
6. **Понимание происходящего.** IDE прячет процессы. Когда кнопка «Build» падает — без понимания CLI не разберёшься.

### 1.3. SDK vs Runtime — что разное

Есть два разных дистрибутива .NET:

| | **SDK** | **Runtime** |
|---|---------|-------------|
| Назначение | Разработка | Выполнение |
| Что включает | Compiler, MSBuild, CLI, Runtime, шаблоны | Только runtime + базовые библиотеки |
| Размер | ~700 MB | ~80–200 MB |
| Команды | `dotnet new`, `build`, `restore`, `publish`, `test`, `pack` | `dotnet MyApp.dll` (запуск собранного приложения) |
| Когда нужен | Машина разработчика, CI-сервер | Production-сервер |

Если ты разработчик — ставь **SDK** (он содержит Runtime внутри). Если деплоишь приложение на сервер — там нужен только **Runtime** (или вообще ничего, если используешь self-contained publish — см. раздел 9).

```bash
dotnet --info
# Покажет всё: SDK версию, runtime версии, OS, архитектуру

dotnet --list-sdks
# 8.0.404 [/usr/share/dotnet/sdk]
# 10.0.100 [/usr/share/dotnet/sdk]

dotnet --list-runtimes
# Microsoft.NETCore.App 8.0.11 [...]
# Microsoft.NETCore.App 10.0.0 [...]
# Microsoft.AspNetCore.App 10.0.0 [...]
```

### 1.4. Эволюция CLI

| Версия | Год | Что появилось |
|--------|-----|---------------|
| .NET Core 1.0 | 2016 | Первый `dotnet` CLI. Замена `xbuild` / `msbuild.exe` для cross-platform |
| .NET Core 2.0 | 2017 | `dotnet pack`, `dotnet test` стандартизованы |
| .NET Core 3.0 | 2019 | `dotnet tool install` (global tools) |
| .NET 5 | 2020 | Унификация: один `dotnet` на все платформы |
| .NET 6 | 2021 | `dotnet watch run` (hot reload), workloads (`dotnet workload install`) |
| .NET 7 | 2022 | `dotnet publish -p:PublishAot=true` (Native AOT GA) |
| .NET 8 | 2023 LTS | `dotnet publish` с улучшенным AOT |
| .NET 10 | 2025 | Расширенные shell completions (zsh/bash/fish/pwsh) |
| .NET 11 | 2026 preview | **File-based apps** — `dotnet run app.cs` без `.csproj` |

### 1.5. Когда CLI достаточно, когда нужна IDE

✅ **CLI справляется:**
- Создание / редактирование любого проекта (csproj — обычный XML).
- Add / remove packages, references, projects.
- Run, watch, test, publish.
- Debug через `dotnet-trace`, `dotnet-dump`, VS Code debugger.
- Refactoring через `dotnet format` + analyzers.

❌ **IDE сильнее:**
- Visual debugging (breakpoints, watch variables, call stack) — VS Code умеет, но Rider/VS удобнее.
- Heavy refactoring (Rename across 100 файлов, Extract Method).
- Performance profiling visual UI (PerfView, dotTrace).
- Code navigation (Go to Implementation, Find Usages) — Rider/VS быстрее на больших солюшенах.

В реальной работе — оба. CLI для всего автоматизированного, IDE для интерактивного debugging и heavy refactoring.

> [!info]- Если ты знаешь Node.js / Python / Rust / Go / Java
> **Node.js:** `dotnet` ≈ комбинация `node` + `npm` + `npx`. `dotnet build` ≈ `npm build`. `dotnet add package X` ≈ `npm install X`. `dotnet tool install -g` ≈ `npm install -g`. Существенная разница: `dotnet new` создаёт проект из шаблона, в Node — обычно `npx create-...`.
>
> **Python:** `dotnet` ≈ `pip` + `python` + `pyproject.toml`-tools. `dotnet add package X` ≈ `pip install X` (но с записью в csproj, как `poetry add`). Виртуальное окружение в .NET не нужно — каждый проект изолирован через csproj и obj/bin.
>
> **Rust:** `dotnet` ≈ `cargo`. Очень близкие концепции: `cargo new` ↔ `dotnet new`, `cargo build` ↔ `dotnet build`, `cargo run` ↔ `dotnet run`, `cargo test` ↔ `dotnet test`, `cargo publish` (для crates.io) ↔ `dotnet pack` (для NuGet). Cargo проще тем, что один формат `Cargo.toml` без MSBuild-сложностей.
>
> **Go:** `dotnet` ≈ `go` + `go mod`. `dotnet new` ≈ `go mod init`, `dotnet build` ≈ `go build`, `dotnet test` ≈ `go test`. Go проще: один module = одна папка с `go.mod`. В .NET projects могут быть вложенными в solutions.
>
> **Java:** `dotnet` ≈ `mvn` или `gradle`. csproj ≈ pom.xml / build.gradle. `dotnet restore` ≈ `mvn dependency:resolve`. По мощности MSBuild сравним с Maven.

> [!question]- Интервью: чем отличается .NET SDK от .NET Runtime?
> SDK включает всё для разработки: компилятор Roslyn, MSBuild, CLI, шаблоны проектов, плюс runtime внутри. Runtime — только то, что нужно для запуска уже собранного приложения: CLR (или CoreCLR), GC, JIT, базовые библиотеки. На production-сервере достаточно Runtime (~80–200 MB), на машине разработчика и CI — SDK (~700 MB). Self-contained publish может включить runtime в само приложение, чтобы на сервере не было даже Runtime.

---

## 2. Установка и управление SDK

### 2.1. Установка

```bash
# Windows (winget — встроен в Windows 10+ и 11)
winget install Microsoft.DotNet.SDK.10

# Windows (Chocolatey)
choco install dotnet-10.0-sdk

# macOS (Homebrew)
brew install --cask dotnet-sdk

# Linux (Ubuntu / Debian)
wget https://packages.microsoft.com/config/ubuntu/24.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt update
sudo apt install -y dotnet-sdk-10.0

# Linux (универсально, через скрипт)
curl -sSL https://dot.net/v1/dotnet-install.sh | bash -s -- --channel 10.0
```

После установки:

```bash
dotnet --version
# 10.0.100

dotnet --info
# Подробная информация: SDK, runtime, OS, паты
```

### 2.2. Side-by-side — несколько SDK на машине

.NET позволяет ставить **много версий SDK одновременно**. Это нужно когда:

- Один проект на .NET 8 (LTS), другой на .NET 10.
- Команда работает с .NET 6, ты обновился до .NET 10 — старые проекты должны собираться.

Каждый SDK живёт в своей папке:

```
/usr/share/dotnet/sdk/
├── 6.0.428/
├── 8.0.404/
└── 10.0.100/

/usr/share/dotnet/shared/
├── Microsoft.NETCore.App/
│   ├── 6.0.36/
│   ├── 8.0.11/
│   └── 10.0.0/
└── Microsoft.AspNetCore.App/
    ├── 6.0.36/
    ├── 8.0.11/
    └── 10.0.0/
```

`dotnet` (binary) выбирает нужную версию SDK для команды по правилам (см. 2.3).

### 2.3. global.json — пиннинг версии

Когда ты в папке с проектом запускаешь `dotnet build`, какой SDK берётся? По умолчанию — самый новый из установленных. Это плохо, если CI использует .NET 10, а локально стоит .NET 11 preview — поведение может различаться.

`global.json` решает проблему:

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestFeature"
  }
}
```

Положи этот файл в корень репозитория. Теперь `dotnet` будет искать SDK 10.0.100 (или новее в той же feature-band, в зависимости от `rollForward`).

`rollForward` варианты:

| Значение | Что делает |
|----------|-----------|
| `disable` | Только точная версия |
| `patch` | Та же major.minor, любой patch (10.0.100 → 10.0.X) |
| `feature` | Та же major.minor, любой feature (10.0.X) |
| `minor` | Та же major, любой minor (10.X.X) |
| `major` | Любой (10+) |
| `latestPatch` | Самая свежая в той же major.minor.feature |
| `latestFeature` | Самая свежая в той же major.minor |
| `latestMinor` | Самая свежая в той же major |
| `latestMajor` | Самая свежая |

Создать через CLI:

```bash
dotnet new globaljson --sdk-version 10.0.100 --roll-forward latestFeature
```

### 2.4. Удаление старых SDK

```bash
# Windows
winget uninstall Microsoft.DotNet.SDK.6

# macOS / Linux — удалить папку
sudo rm -rf /usr/share/dotnet/sdk/6.0.428

# Или скрипт от Microsoft (Linux/macOS)
curl -sSL https://dot.net/v1/dotnet-uninstall.sh | bash -s -- --sdk 6.0.428
```

### 2.5. Где хранятся артефакты

- **NuGet кэш пакетов:** `~/.nuget/packages` (Linux/macOS) или `%USERPROFILE%\.nuget\packages` (Windows). Один кэш на все проекты.
- **Установленные SDK:** `/usr/share/dotnet/` (Linux), `/usr/local/share/dotnet/` (macOS), `C:\Program Files\dotnet\` (Windows).
- **Tools:** `~/.dotnet/tools/` (global tools), `<project>/.config/dotnet-tools.json` + локально в `<project>/.dotnet/` (local tools).
- **Templates:** `~/.templateengine/` (для `dotnet new`).
- **User secrets:** `~/.microsoft/usersecrets/<UserSecretsId>/` (Linux/macOS) или `%APPDATA%\Microsoft\UserSecrets\<id>\` (Windows).

Знание этих путей помогает при отладке («почему не находит пакет», «куда пропал tool», «откуда взялась эта версия»).

> [!question]- Интервью: зачем нужен global.json в репозитории?
> Чтобы зафиксировать версию SDK, которой собирается проект. Без него `dotnet` выберет последнюю установленную версию SDK на машине, что может отличаться между разработчиками и CI. Это приводит к «у меня собирается, у тебя нет». `global.json` гарантирует, что все собирают одной версией — у CI, у разработчиков, у Docker-builds.

---

## 3. dotnet new — шаблоны проектов

### 3.1. Базовое использование

```bash
mkdir MyApp
cd MyApp
dotnet new console
```

Создаст:

```
MyApp/
├── MyApp.csproj
└── Program.cs
```

Имя проекта = имя папки по умолчанию. Можно переопределить:

```bash
dotnet new console -n CustomerApp -o ./apps/customer
# Создаст ./apps/customer/CustomerApp.csproj
```

### 3.2. Самые частые шаблоны

```bash
# Console-приложение
dotnet new console

# Class library (DLL для подключения к другим проектам)
dotnet new classlib

# ASP.NET Core
dotnet new web              # Minimal API (один Program.cs)
dotnet new webapi           # Web API с контроллерами
dotnet new mvc              # MVC с views
dotnet new razor            # Razor Pages
dotnet new blazor           # Blazor (с .NET 8 — единый шаблон, render modes)
dotnet new grpc             # gRPC service

# Тесты
dotnet new xunit            # xUnit
dotnet new nunit            # NUnit
dotnet new mstest           # MSTest

# Solution
dotnet new sln              # Пустой .sln

# UI
dotnet new wpf              # WPF (только Windows)
dotnet new winforms         # Windows Forms
dotnet new maui             # MAUI (cross-platform mobile + desktop)

# Прочее
dotnet new gitignore        # .gitignore для .NET
dotnet new editorconfig     # .editorconfig с .NET-конвенциями
dotnet new globaljson       # global.json
dotnet new nugetconfig      # NuGet.config
dotnet new tool-manifest    # .config/dotnet-tools.json для local tools
```

### 3.3. Полный список и поиск

```bash
# Все установленные шаблоны
dotnet new list

# Поиск по подстроке
dotnet new list api
# Покажет: webapi, web, blazor, и т.д. содержащие "api"

# Поиск онлайн (NuGet templates)
dotnet new search clean
# Покажет community-шаблоны типа CleanArchitecture
```

### 3.4. Установка community-шаблонов

```bash
# Установить шаблон из NuGet
dotnet new install Clean.Architecture.Solution.Template

# Использовать
dotnet new ca-sln -n MyShop

# Удалить
dotnet new uninstall Clean.Architecture.Solution.Template
```

Популярные community-шаблоны:

- **Clean.Architecture.Solution.Template** (Jason Taylor) — Clean Architecture
- **Ardalis.CleanArchitecture.Template** — альтернатива от Steve Smith
- **Boxed.Templates** — куча готовых шаблонов с best practices

### 3.5. Параметры шаблонов

У каждого шаблона есть свои опции. Узнать их:

```bash
dotnet new webapi --help
# Опции: --use-controllers, --no-https, --auth, --framework и т.д.
```

Примеры:

```bash
# Web API без HTTPS
dotnet new webapi --no-https

# Web API с JWT-авторизацией
dotnet new webapi --auth IndividualB2C

# Указать .NET версию
dotnet new console -f net8.0
```

### 3.6. File-based apps (.NET 11 preview)

С .NET 11 можно запускать одиночные `.cs` файлы без проекта:

```bash
# Создать file-based app
dotnet new singlecs -n hello -o .

# Это создаёт hello.cs:
# Console.WriteLine("Hello!");

# Запустить напрямую
dotnet run hello.cs
# Hello!
```

Это для скриптов и одноразовых задач — не для production. Близко к `python script.py` или `node script.js`. Поддерживается `#:package` директива для NuGet-зависимостей внутри файла.

### 3.7. Создание собственного шаблона

Можно сделать свой шаблон и поделиться через NuGet. Краткий рецепт:

1. Создаёшь рабочий проект-образец.
2. В корне добавляешь `.template.config/template.json` с метаданными.
3. Параметризуешь имена и значения через placeholders.
4. Упаковываешь как NuGet-пакет с `<PackageType>Template</PackageType>` в csproj.
5. Публикуешь в NuGet.org (или приватный feed).

Деталь — в Reading list. Для большинства задач хватит готовых шаблонов.

> [!question]- Интервью: что делает `dotnet new console`?
> Создаёт минимальный console-проект из встроенного шаблона. Генерирует два файла: `<name>.csproj` (XML с метаданными MSBuild — TargetFramework, OutputType=Exe, ImplicitUsings, Nullable) и `Program.cs` (top-level statements с `Console.WriteLine`). После этого `dotnet run` соберёт и запустит. Имя проекта — имя текущей папки, можно переопределить через `-n`.

---

## 4. Структура проекта и MSBuild

### 4.1. Project file (csproj) — это XML-сценарий MSBuild

Простейший csproj:

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

Что это значит:

- **`Sdk="Microsoft.NET.Sdk"`** — указывает MSBuild использовать SDK-стиль проекта (модернизированный, начиная с .NET Core 1.0). Этот SDK включает в себя все стандартные таски: компиляция Roslyn, NuGet restore, копирование файлов.
- **`<OutputType>Exe</OutputType>`** — что генерировать (Exe = исполняемое, Library = DLL).
- **`<TargetFramework>net10.0</TargetFramework>`** — TFM (Target Framework Moniker), под какую версию .NET собирать.
- **`<ImplicitUsings>enable</ImplicitUsings>`** — автоматически добавить часто используемые using-ы (System, System.Linq, System.Collections.Generic и т.д.).
- **`<Nullable>enable</Nullable>`** — включить Nullable Reference Types (NRT), компилятор будет следить за null.

### 4.2. Распространённые SDK-типы

```xml
<Project Sdk="Microsoft.NET.Sdk">                   <!-- console / classlib -->
<Project Sdk="Microsoft.NET.Sdk.Web">                <!-- ASP.NET Core, Blazor Server -->
<Project Sdk="Microsoft.NET.Sdk.BlazorWebAssembly"> <!-- Blazor WASM -->
<Project Sdk="Microsoft.NET.Sdk.Razor">              <!-- Razor Class Library -->
<Project Sdk="Microsoft.NET.Sdk.Worker">             <!-- Worker Service -->
<Project Sdk="MSBuild.Sdk.Extras">                  <!-- Multi-targeting helper (community) -->
```

Разные SDK включают разные дефолтные таски, ItemGroups, шаблоны.

### 4.3. ItemGroups — packages и references

```xml
<ItemGroup>
  <!-- NuGet-пакеты -->
  <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  <PackageReference Include="Serilog" Version="3.1.1" />
</ItemGroup>

<ItemGroup>
  <!-- Ссылки на другие .csproj в той же solution -->
  <ProjectReference Include="..\Domain\Domain.csproj" />
</ItemGroup>

<ItemGroup>
  <!-- Ссылки на DLL вне NuGet (legacy) -->
  <Reference Include="LegacyLib">
    <HintPath>libs\LegacyLib.dll</HintPath>
  </Reference>
</ItemGroup>

<ItemGroup>
  <!-- Файлы, которые включаются в build (например, копируются в output) -->
  <None Include="appsettings.json" CopyToOutputDirectory="Always" />
  <Content Include="data\*.json" />
</ItemGroup>
```

### 4.4. Solution (.sln) — что это

`.sln` — это «папка солюшена», файл со списком проектов:

```
Microsoft Visual Studio Solution File, Format Version 12.00
# Visual Studio Version 17

Project("{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}") = "MyApi", "src\MyApi\MyApi.csproj", "{...}"
EndProject
Project("{FAE04EC0-301F-11D3-BF4B-00C04F79EFBC}") = "MyDomain", "src\MyDomain\MyDomain.csproj", "{...}"
EndProject

Global
    GlobalSection(SolutionConfigurationPlatforms) = preSolution
        Debug|Any CPU = Debug|Any CPU
        Release|Any CPU = Release|Any CPU
    EndGlobalSection
EndGlobal
```

`.sln` нужен, чтобы:

- Обработать сразу несколько проектов (`dotnet build SolutionName.sln`).
- В IDE (VS, Rider) видеть все проекты в одном дереве.
- Для CI — `dotnet build / test` запускается на solution, не на отдельных csproj.

В Visual Studio в .NET 9+ появился **slnx** — XML-формат вместо текстового `.sln`:

```xml
<Solution>
  <Project Path="src/MyApi/MyApi.csproj" />
  <Project Path="src/MyDomain/MyDomain.csproj" />
</Solution>
```

Совместим с CLI: `dotnet build MyApp.slnx`.

### 4.5. Команды управления solution

```bash
# Создать пустой sln
dotnet new sln -n MyApp

# Добавить проект в sln
dotnet sln add src/MyApi/MyApi.csproj
dotnet sln add **/*.csproj   # добавить все csproj в дереве

# Удалить
dotnet sln remove src/MyApi/MyApi.csproj

# Список
dotnet sln list

# Reference между проектами
dotnet add src/MyApi/MyApi.csproj reference src/MyDomain/MyDomain.csproj
```

### 4.6. Directory.Build.props — общие настройки

Если в репозитории много проектов с одинаковыми настройками, повторять в каждом csproj — мука. Положи в корень `Directory.Build.props`:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <LangVersion>latest</LangVersion>
    <Authors>My Company</Authors>
    <Copyright>© 2026 My Company</Copyright>
  </PropertyGroup>
</Project>
```

MSBuild автоматически подхватит `.props` из родительских папок и применит до всех csproj. Аналогично есть `Directory.Build.targets` — применяется **после** csproj (для override).

Это самый чистый способ синхронизировать настройки в большом solution.

### 4.7. Centralized Package Management (.NET 7+)

Если у тебя 20 проектов и в каждом `<PackageReference Include="Serilog" Version="3.1.1" />` — обновлять версию во всех руками больно. Решение — **CPM (Central Package Management)**.

Положи в корень `Directory.Packages.props`:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Serilog" Version="3.1.1" />
    <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
  </ItemGroup>
</Project>
```

В csproj-ах теперь без версий:

```xml
<ItemGroup>
  <PackageReference Include="Serilog" />
</ItemGroup>
```

Обновление версии — в одном месте.

> [!question]- Интервью: что такое Directory.Build.props и зачем нужен?
> Это файл MSBuild, который автоматически импортируется во все csproj в дереве папок. Используется для единых настроек в монорепе: TargetFramework, Nullable, версии пакетов, code style. Положил его в корень — все 20 проектов получили одинаковые настройки. Без него пришлось бы повторять `<TargetFramework>net10.0</TargetFramework>` в каждом csproj. Аналог — `Directory.Build.targets` (применяется после), `Directory.Packages.props` для централизованного управления версиями пакетов (с .NET 7).

---

## 5. Restore — как работает

### 5.1. Что делает dotnet restore

```bash
dotnet restore
```

Команда:

1. Парсит csproj — собирает список `<PackageReference>`.
2. Для каждого пакета — резолвит граф зависимостей рекурсивно (transitive dependencies).
3. Скачивает пакеты с NuGet-серверов (по умолчанию nuget.org) в локальный кэш `~/.nuget/packages/`.
4. Записывает `obj/project.assets.json` — точный snapshot всех зависимостей с версиями. Это «lock-файл» проекта.
5. Создаёт `obj/<project>.csproj.nuget.g.props` и `obj/<project>.csproj.nuget.g.targets` — MSBuild-сценарии, которые подскажут компилятору, где искать DLL.

### 5.2. Когда restore запускается

- Явно: `dotnet restore`.
- Автоматически перед `build`, `run`, `test`, `publish`, `pack` (если кэш не свежий).
- Можно отключить авто-restore: `dotnet build --no-restore` (полезно в CI, когда `restore` уже сделан отдельным шагом).

### 5.3. NuGet sources — откуда берутся пакеты

Источники конфигурируются в `NuGet.Config`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="company-feed" value="https://nuget.company.com/v3/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <company-feed>
      <add key="Username" value="user" />
      <add key="ClearTextPassword" value="%COMPANY_FEED_TOKEN%" />
    </company-feed>
  </packageSourceCredentials>
</configuration>
```

Команды управления:

```bash
# Список источников
dotnet nuget list source

# Добавить источник
dotnet nuget add source https://nuget.company.com/v3/index.json -n company

# Удалить
dotnet nuget remove source company

# С учётными данными
dotnet nuget add source https://nuget.company.com/v3/index.json \
  -n company -u myuser -p "$(secret-token)" --store-password-in-clear-text
```

### 5.4. Кэш пакетов

Все скачанные пакеты лежат в `~/.nuget/packages/`. Структура:

```
~/.nuget/packages/
├── newtonsoft.json/
│   ├── 13.0.3/
│   │   ├── newtonsoft.json.nuspec
│   │   ├── lib/
│   │   │   ├── net6.0/
│   │   │   │   └── Newtonsoft.Json.dll
│   │   │   └── netstandard2.0/
│   │   │       └── Newtonsoft.Json.dll
│   │   └── README.md
│   └── 12.0.3/
│       └── ...
└── serilog/
    └── 3.1.1/
        └── ...
```

Один пакет одной версии — одна папка. Все проекты на машине переиспользуют этот кэш. Очистить:

```bash
# Очистить весь кэш
dotnet nuget locals all --clear

# Только http-кэш (плохо скачалось)
dotnet nuget locals http-cache --clear
```

### 5.5. Transitive dependencies

Когда ты ставишь `Microsoft.AspNetCore.Authentication.JwtBearer`, NuGet тянет цепочку:

```
Microsoft.AspNetCore.Authentication.JwtBearer
├── Microsoft.IdentityModel.Protocols.OpenIdConnect
│   ├── Microsoft.IdentityModel.Protocols
│   │   └── Microsoft.IdentityModel.Tokens
│   │       └── Microsoft.IdentityModel.Logging
│   └── ...
└── ...
```

Все скачиваются и доступны. Видеть граф:

```bash
dotnet list package --include-transitive
```

### 5.6. Lock-файл — packages.lock.json

По умолчанию `restore` не записывает «жёсткий» lock — версии в csproj и transitive могут «плавать» в пределах правил semver. Для воспроизводимых билдов в CI нужен **lock-файл**:

В csproj:

```xml
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>
```

После `dotnet restore` появится `packages.lock.json` с точными хэшами. На CI:

```bash
dotnet restore --locked-mode
# Если зависимости изменились с момента lock — ошибка
```

Это аналог `package-lock.json` (npm), `Cargo.lock` (Rust), `poetry.lock` (Python).

### 5.7. Restore в Docker — отдельный layer

Стандартный паттерн multi-stage Dockerfile:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Скопировать только csproj — для restore
COPY *.csproj ./
RUN dotnet restore

# Скопировать остальной код и собрать
COPY . .
RUN dotnet publish -c Release -o /out

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /out ./
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

Зачем: Docker кэширует layer'ы. Если csproj не менялся — `RUN dotnet restore` берётся из кэша, не перекачивает 200 MB пакетов на каждом билде.

> [!question]- Интервью: что такое packages.lock.json и зачем нужен?
> Это файл с точными версиями и хэшами всех пакетов (включая transitive), которые были resolved при последнем `restore`. Гарантирует воспроизводимые билды: если включён lock-mode, `dotnet restore` упадёт, если зависимости изменились без обновления lock-файла. Аналог `package-lock.json` в Node.js, `Cargo.lock` в Rust, `poetry.lock` в Python. Включается через `<RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>` в csproj.

---

## 6. Build — что внутри

### 6.1. Что делает dotnet build

```bash
dotnet build
```

1. Если нужно — `restore` (с .NET 6+ авто).
2. MSBuild парсит csproj и зависимости.
3. Roslyn компилирует `.cs` файлы в IL и складывает в DLL.
4. Копируются ссылки (`PackageReference`, `ProjectReference`) в output.
5. Создаются артефакты в `bin/<Configuration>/<TargetFramework>/`.

### 6.2. Output структура

```
bin/Debug/net10.0/
├── MyApp.dll              ← скомпилированный код
├── MyApp.exe              ← обёртка-launcher (на Windows)
├── MyApp.pdb              ← debug-символы
├── MyApp.deps.json        ← манифест зависимостей (что грузить)
├── MyApp.runtimeconfig.json   ← runtime настройки
├── appsettings.json       ← скопированные content-файлы
├── Newtonsoft.Json.dll    ← DLL зависимостей
└── ...
```

`Debug` — конфигурация по умолчанию: символы, неоптимизировано, для разработки. `Release` — оптимизировано, для production:

```bash
dotnet build -c Release
# bin/Release/net10.0/...
```

### 6.3. Ключевые опции

```bash
# Пропустить restore (если только что был)
dotnet build --no-restore

# Verbose output для дебага build-проблем
dotnet build -v normal     # detailed | diagnostic — ещё подробнее

# Только определённый проект
dotnet build src/MyApi/MyApi.csproj

# Отдельная конфигурация platform
dotnet build -c Release -p:Platform=x64

# Передать любое MSBuild property
dotnet build -p:Version=1.2.3 -p:AssemblyVersion=1.2.3.0

# Очистка перед build
dotnet clean
dotnet build
```

### 6.4. Deterministic builds

Для воспроизводимости (одинаковые входы → одинаковые DLL по байтам):

```xml
<PropertyGroup>
  <Deterministic>true</Deterministic>
  <ContinuousIntegrationBuild>true</ContinuousIntegrationBuild>
</PropertyGroup>
```

Это убирает таймштампы из metadata, нормализует пути в PDB. Полезно для:

- Cache-driven CI (одинаковый artifact — pull from cache).
- Reproducible builds (compliance / supply chain security).

### 6.5. Параллельная сборка

По умолчанию MSBuild собирает проекты в solution параллельно. Можно настраивать:

```bash
# Максимум параллельных воркеров
dotnet build -m:8

# Отключить параллелизм
dotnet build -m:1
```

В CI имеет смысл выкручивать на максимум CPU. На слабом машине (4 GB RAM) большое число параллельных воркеров может ронять MSBuild по памяти — снижай.

### 6.6. Incremental build

MSBuild смотрит таймштампы файлов и пересобирает только то, что изменилось:

```bash
# Первый build — компилит всё
dotnet build       # ~10s

# Без изменений — почти моментально
dotnet build       # ~1s

# Изменил один файл — только зависимый код пересобирается
echo "new line" >> Program.cs
dotnet build       # ~3s
```

Если incremental не работает (всё пересобирается при каждом запуске) — обычно из-за неправильных `<None>` Items с `CopyToOutputDirectory="Always"`. Меняй на `PreserveNewest`.

### 6.7. Compiler warnings и errors

```bash
# Treat warnings as errors (production-style)
dotnet build -p:TreatWarningsAsErrors=true

# Показать все warnings
dotnet build -warnaserror   # alias
```

В csproj:

```xml
<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
<NoWarn>CS1591;CS8618</NoWarn>   <!-- исключения -->
<WarningsAsErrors>nullable</WarningsAsErrors>   <!-- только nullable warnings -->
```

> [!question]- Интервью: что такое incremental build и как он работает?
> MSBuild сравнивает таймштампы входных файлов (`.cs`, `.csproj`, ссылки) с output (`.dll`, `.pdb`). Если входы не новее output — пропускает компиляцию. Это даёт быстрые повторные сборки. Сломать можно неправильными `<None CopyToOutputDirectory="Always">` (всегда копирует, делает output «свежее») — лучше `PreserveNewest`. В CI инкрементальность зависит от кэша: если CI не сохраняет obj/bin между сборками, каждый build full. Решается caching steps (GitHub Actions actions/cache, GitLab cache).

---

## 7. Run и watch

### 7.1. dotnet run

```bash
dotnet run
```

1. Если нужно — `restore`, `build`.
2. Запускает приложение через `dotnet bin/Debug/net10.0/MyApp.dll`.

Параметры:

```bash
# Передать args в приложение (после --)
dotnet run -- --verbose --port 5000

# Конкретный проект (если в папке несколько csproj)
dotnet run --project src/MyApi/MyApi.csproj

# Конфигурация
dotnet run -c Release

# Конкретный launchSettings.json profile
dotnet run --launch-profile https
```

### 7.2. launchSettings.json — запуск-конфиги

В ASP.NET Core проектах есть `Properties/launchSettings.json`:

```json
{
  "$schema": "http://json.schemastore.org/launchsettings.json",
  "profiles": {
    "http": {
      "commandName": "Project",
      "applicationUrl": "http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "applicationUrl": "https://localhost:5001;http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

`dotnet run --launch-profile https` использует профиль `https`. По умолчанию — первый профиль с `commandName: Project`. В IDE (VS, Rider) видны все профили в dropdown.

`launchSettings.json` — **только для разработки**, в production не используется. Не коммитим секреты сюда (используем `dotnet user-secrets`, см. раздел 13).

### 7.3. dotnet watch — hot reload

```bash
dotnet watch run
# Изменил Program.cs → автоматический rebuild + restart
```

Под капотом `watch` следит за файлами проекта (через FileSystemWatcher) и при изменении:

1. Пытается **hot reload** (применить изменение без перезапуска через MetadataUpdateHandler).
2. Если не получилось (rude edit — изменение сигнатуры, async) — спрашивает «restart?».
3. Перезапускает приложение.

### 7.4. Что hot-reload-вается

Не всё. Hot reload работает для:

- Изменения тела метода.
- Добавление новых методов.
- Изменение constants и literals.
- Изменения lambda внутри метода.

Не работает («rude edit», нужен restart):

- Изменение сигнатуры метода (параметры, return type).
- Добавление / удаление поля класса.
- Изменение базового класса.
- Добавление `async` к существующему методу.
- Изменение generic-параметров.

Сообщение в watch-консоли укажет конкретно. Можно нажать `r` — restart, `Ctrl+C` — stop.

### 7.5. Watch + tests

```bash
# Перезапускать тесты при изменении кода
dotnet watch test --project tests/MyTests
```

Удобно для TDD: пишешь тест → сохраняешь → автоматически зелёный/красный.

### 7.6. Watch с no-restore / no-build

Для скорости:

```bash
dotnet watch --no-hot-reload run    # без hot reload, только полный restart
dotnet watch --no-restore run        # пропустить restore (нужен один раз ручной)
```

> [!question]- Интервью: как работает hot reload в ASP.NET Core?
> `dotnet watch run` следит за файлами через FileSystemWatcher. При изменении пытается применить дельту через MetadataUpdateHandler API (.NET 6+). Если изменение «small» (тело метода, добавление метода) — runtime подменяет IL на лету, без перезапуска. Если «rude edit» (сигнатура, поля, async) — runtime не может подменить, нужен restart. Hot reload поддерживается Roslyn EditAndContinue API на стороне SDK, и MetadataUpdateHandler — на стороне runtime/ASP.NET Core.

---

## 8. Test

### 8.1. dotnet test

```bash
# Все тесты в текущей папке/sln
dotnet test

# Конкретный проект
dotnet test tests/MyTests/MyTests.csproj

# Конкретный фильтр
dotnet test --filter "FullyQualifiedName~OrderTests"
dotnet test --filter "Category=Integration"
dotnet test --filter "Priority=1&Category=Smoke"
```

Под капотом — VSTest или новый Microsoft.Testing.Platform. Запускает все обнаруженные test-сборки.

### 8.2. Test runners

`dotnet test` работает с любым runner-ом, который понимает VSTest API:

| Runner | NuGet | Атрибут |
|--------|-------|---------|
| xUnit | `xunit.runner.visualstudio` | `[Fact]`, `[Theory]` |
| NUnit | `NUnit3TestAdapter` | `[Test]`, `[TestCase]` |
| MSTest | `MSTest.TestAdapter` | `[TestMethod]` |

Шаблоны (`dotnet new xunit`, `dotnet new nunit`, `dotnet new mstest`) включают нужный adapter.

### 8.3. Coverage

```bash
# Сбор coverage в формате Cobertura
dotnet test --collect:"XPlat Code Coverage"

# Output: TestResults/<guid>/coverage.cobertura.xml
```

Для красивого отчёта:

```bash
dotnet tool install -g dotnet-reportgenerator-globaltool

reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:coverage-report -reporttypes:Html
```

Открой `coverage-report/index.html` — увидишь % покрытия по файлам и неприкрытые строки.

### 8.4. Параллельность тестов

```bash
# Максимум параллельных тестов
dotnet test -m:8

# В xunit.runner.json:
{
  "parallelizeAssembly": true,
  "parallelizeTestCollections": true,
  "maxParallelThreads": 4
}
```

Параллелить можно тесты, которые изолированы (не делят state). Integration-тесты с одной БД часто помечают `[Collection("Database")]` чтобы НЕ параллелить.

### 8.5. Verbose output

```bash
# Подробный вывод (показывает имена тестов)
dotnet test --logger "console;verbosity=detailed"

# Или в trx-формат для CI
dotnet test --logger trx --results-directory TestResults

# JUnit xml (для GitLab / Jenkins integration)
dotnet test --logger "junit;LogFilePath=test-results.xml"
```

### 8.6. Тесты в CI с разделением

Большие test-сюиты можно делить на shards:

```bash
# Run только integration tests
dotnet test --filter "Category=Integration"

# Run только unit tests (быстро, на каждом push)
dotnet test --filter "Category=Unit"
```

Полная картина — раздел 17 (CI/CD recipes).

> [!question]- Интервью: чем отличается VSTest от Microsoft.Testing.Platform?
> VSTest — старый test host (с .NET Framework). Microsoft.Testing.Platform (MTP) — новый, с .NET 8+, легче, быстрее, без зависимости от Visual Studio, можно запускать как обычное приложение (`./MyTests.exe --filter ...`). MTP пока опциональный — выбирается через `<UseMicrosoftTestingPlatformRunner>true</UseMicrosoftTestingPlatformRunner>` в csproj. xUnit/NUnit/MSTest имеют адаптеры под MTP.

---

## 9. Publish — для деплоя

### 9.1. Зачем publish (а не build)

`build` создаёт DLL для разработки. `publish` создаёт **готовый к деплою бандл**:

- Все нужные DLL (зависимости + сам код).
- `appsettings.json` и другие content-файлы.
- runtimeconfig.json с runtime настройками.
- (опционально) сам runtime — для self-contained.
- Опционально trimming, AOT, single file.

```bash
dotnet publish -c Release -o ./publish
```

Папка `./publish` — это всё, что нужно положить на сервер.

### 9.2. Framework-dependent (по умолчанию)

```bash
dotnet publish -c Release -o ./publish
```

Размер: ~5-50 MB (зависит от приложения).
На сервере должен быть установлен .NET Runtime той же major-версии.

Запуск:

```bash
dotnet ./publish/MyApi.dll
```

Это **дефолт** для production-серверов с управляемой инфраструктурой (есть DevOps, ставит runtime).

### 9.3. Self-contained — runtime внутри

```bash
dotnet publish -c Release -r linux-x64 --self-contained -o ./publish
```

Размер: 80-200 MB (включает .NET runtime).
На сервере **не нужен** установленный .NET — приложение само-достаточное.

Полезно для:
- Docker-образов на минимальной ОС (alpine, distroless).
- Раздачи как бинарника (одиночный exe).
- VMs / IoT, где обновлять runtime отдельно — морока.

### 9.4. Single-file publish

```bash
dotnet publish -c Release -r linux-x64 --self-contained \
    -p:PublishSingleFile=true -o ./publish
```

Получаешь один `.exe` (или ELF на Linux) который содержит всё. Удобно для distribution. Размер тот же ~80-200 MB.

### 9.5. ReadyToRun (R2R)

Pre-compile IL в native-код для быстрого старта:

```bash
dotnet publish -c Release -r linux-x64 -p:PublishReadyToRun=true -o ./publish
```

- IL сохраняется (на случай runtime-нужд).
- Native-код добавляется → больше размер (1.5-2x).
- Быстрее холодный старт (не нужно JIT-ить методы первого вызова).

Полезно для serverless / cold-start critical (Azure Functions, AWS Lambda на .NET).

### 9.6. Trimmed publish

```bash
dotnet publish -c Release -r linux-x64 --self-contained \
    -p:PublishTrimmed=true -o ./publish
```

Linker анализирует код и **выбрасывает неиспользуемый IL** из сборок (включая BCL). Размер: 20-60 MB вместо 200.

⚠️ **Внимание:** trimming может сломать reflection-зависимый код (EF Core, Newtonsoft.Json иногда, AutoMapper). Включай и тестируй.

В .NET 8+ trim-mode по умолчанию `partial` (только marked-as-trimmable assemblies). Можно поставить `full`:

```xml
<PropertyGroup>
  <PublishTrimmed>true</PublishTrimmed>
  <TrimMode>full</TrimMode>
</PropertyGroup>
```

### 9.7. Native AOT (.NET 7+, GA в .NET 8)

```bash
dotnet publish -c Release -r linux-x64 -p:PublishAot=true -o ./publish
```

AOT (Ahead-Of-Time) — компилирует **всё** в native машинный код перед запуском. Нет JIT, нет IL.

Преимущества:

- **Холодный старт ~10-50ms** (vs 500ms для JIT).
- **Размер ~10-30 MB** (с trimming).
- **Меньше RAM на старте.**
- **Можно деплоить в FROM scratch Docker** (нет даже runtime).

Ограничения:

- **Reflection строго ограничено** — нужны `DynamicallyAccessedMembers` атрибуты или статические альтернативы.
- **No dynamic code generation** — нет `Reflection.Emit`, `Expression.Compile` (хотя последнее в .NET 9 частично работает).
- **System.Text.Json** — нужен `JsonSerializerContext` source generator.
- **EF Core** — limitation (улучшается в каждом .NET версии).

Подробно — отдельная заметка по Native AOT.

### 9.8. Сравнительная таблица режимов publish

| Режим | Размер | Holod start | Runtime требуется на сервере |
|-------|--------|-------------|------------------------------|
| Framework-dependent | 5-50 MB | ~500ms | Да (.NET Runtime) |
| Self-contained | 80-200 MB | ~500ms | Нет |
| Self-contained + Single-file | 80-200 MB (один файл) | ~500ms | Нет |
| Self-contained + Trimmed | 20-60 MB | ~500ms | Нет |
| Self-contained + R2R | 150-300 MB | ~200ms | Нет |
| Native AOT | 10-30 MB | ~10-50ms | Нет |

Для большинства веб-приложений — framework-dependent (короткое время сборки, маленький артефакт). Для serverless / контейнеров с экстремальной плотностью — AOT.

> [!question]- Интервью: чем self-contained отличается от framework-dependent publish?
> Framework-dependent создаёт только сборки приложения (5-50 MB), требует установленного .NET Runtime на target-машине. Self-contained включает runtime + base libraries (80-200 MB), не требует ничего на target. Self-contained проще для дистрибуции (бинарник работает «из коробки»), framework-dependent — для управляемых production-серверов с центральным runtime. AOT идёт дальше: компилирует в native, без CLR/JIT, ~10-30 MB и ~10-50ms cold start.

---

## 10. Pack — создание NuGet-пакетов

### 10.1. Когда нужен pack

Если ты делаешь библиотеку, которой будут пользоваться другие (внутри компании или публично) — упаковывай в NuGet:

```bash
dotnet pack -c Release -o ./nupkgs
```

Получишь `MyLib.1.0.0.nupkg` в `./nupkgs/`. Это zip-файл с DLL + metadata.

### 10.2. Метаданные пакета — в csproj

```xml
<PropertyGroup>
  <PackageId>MyCompany.MyLib</PackageId>
  <Version>1.2.3</Version>
  <Authors>My Name</Authors>
  <Description>Useful library for X</Description>
  <PackageLicenseExpression>MIT</PackageLicenseExpression>
  <PackageProjectUrl>https://github.com/me/mylib</PackageProjectUrl>
  <RepositoryUrl>https://github.com/me/mylib</RepositoryUrl>
  <PackageTags>utility tool helper</PackageTags>
  <PackageReadmeFile>README.md</PackageReadmeFile>
  <IncludeSymbols>true</IncludeSymbols>
  <SymbolPackageFormat>snupkg</SymbolPackageFormat>
</PropertyGroup>

<ItemGroup>
  <None Include="README.md" Pack="true" PackagePath="\" />
</ItemGroup>
```

### 10.3. Symbols (snupkg)

`<IncludeSymbols>true</IncludeSymbols>` + `<SymbolPackageFormat>snupkg</SymbolPackageFormat>` создаст `MyLib.1.2.3.snupkg` со всеми PDB-файлами.

Symbol-пакет публикуется отдельно на symbols-сервер (NuGet.org поддерживает). Тогда консумер библиотеки может пошагово отлаживать твой код через Source Link.

### 10.4. Source Link

```xml
<PropertyGroup>
  <PublishRepositoryUrl>true</PublishRepositoryUrl>
  <EmbedUntrackedSources>true</EmbedUntrackedSources>
  <DebugType>embedded</DebugType>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="Microsoft.SourceLink.GitHub" Version="8.0.0" PrivateAssets="all" />
</ItemGroup>
```

Source Link встраивает в PDB ссылку на исходники (на GitHub). Когда консумер шагает дебаггером в твою библиотеку — VS / Rider скачивает исходники с GitHub и показывает.

### 10.5. Publish в NuGet.org

```bash
# Получить API key на nuget.org (раздел API Keys в профиле)

dotnet nuget push ./nupkgs/MyLib.1.2.3.nupkg \
    --api-key $NUGET_API_KEY \
    --source https://api.nuget.org/v3/index.json
```

После публикации пакет проиндексируется (5-30 минут) и станет доступен через `dotnet add package MyCompany.MyLib`.

### 10.6. Приватный feed (компания)

Корпоративные NuGet-серверы: Azure DevOps Artifacts, GitHub Packages, BaGet, Sonatype Nexus.

```bash
# Push на корпоративный feed
dotnet nuget push ./nupkgs/MyLib.1.2.3.nupkg \
    --api-key $COMPANY_NUGET_TOKEN \
    --source https://nuget.company.com/v3/index.json
```

Консумерам нужен `NuGet.Config` с этим source-ом (или их `dotnet nuget add source ...`).

### 10.7. Multi-targeting в pack

Если хочешь поддержать несколько TFM:

```xml
<TargetFrameworks>net6.0;net8.0;net10.0;netstandard2.0</TargetFrameworks>
```

`pack` создаст один nupkg со всеми вариантами:

```
MyLib.1.2.3.nupkg
└── lib/
    ├── net6.0/MyLib.dll
    ├── net8.0/MyLib.dll
    ├── net10.0/MyLib.dll
    └── netstandard2.0/MyLib.dll
```

Консумер автоматически берёт ближайший к своему TFM.

> [!question]- Интервью: что такое Source Link?
> Механизм встраивания в PDB-файлы ссылок на исходный код в публичном репозитории (GitHub, Azure Repos). Когда отлаживающий шагает в скомпилированную библиотеку, IDE использует эту ссылку, чтобы скачать исходники и показать. Включается в csproj через `Microsoft.SourceLink.<Provider>` пакет. Очень полезно для разработчиков библиотек: пользователь может дебажить в твой код без копирования исходников.

---

## 11. Tools — global vs local

### 11.1. Что такое .NET tool

Это NuGet-пакет, помеченный как «инструмент»: в нём есть исполняемый entry-point. Установив, ты получаешь команду в PATH:

```bash
dotnet tool install -g dotnet-ef
# Теперь:
dotnet ef --help
```

### 11.2. Global tools

```bash
# Установить
dotnet tool install -g dotnet-ef
dotnet tool install -g dotnet-format
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-dump

# Список
dotnet tool list -g

# Обновить
dotnet tool update -g dotnet-ef

# Удалить
dotnet tool uninstall -g dotnet-ef
```

Установка идёт в `~/.dotnet/tools/`. Этот путь должен быть в `$PATH`.

### 11.3. Local tools — рекомендованный подход

Global tools имеют один общий version на машине. Если проект A нуждается в `dotnet-ef 8.0`, а проект B — в `dotnet-ef 10.0`, конфликт.

**Local tools** живут в самом проекте, версии фиксируются в манифесте:

```bash
# В корне solution — создать манифест
dotnet new tool-manifest

# Это создаёт .config/dotnet-tools.json:
# {
#   "version": 1,
#   "isRoot": true,
#   "tools": {}
# }

# Установить tool локально
dotnet tool install dotnet-ef --version 10.0.0

# Манифест обновлён:
# {
#   "version": 1,
#   "isRoot": true,
#   "tools": {
#     "dotnet-ef": {
#       "version": "10.0.0",
#       "commands": ["dotnet-ef"]
#     }
#   }
# }

# Restore tools из манифеста (например, на CI)
dotnet tool restore

# Запуск
dotnet ef migrations add Initial
# или явно
dotnet tool run dotnet-ef migrations add Initial
```

Манифест **коммитится в git**. Все разработчики и CI получают одинаковые версии после `dotnet tool restore`.

### 11.4. Полезные tools

| Tool | Зачем |
|------|-------|
| `dotnet-ef` | Migration management для EF Core |
| `dotnet-format` | Форматирование кода по `.editorconfig` |
| `dotnet-outdated-tool` | Найти устаревшие пакеты |
| `dotnet-counters` | Live мониторинг metrics приложения |
| `dotnet-trace` | Профилирование (sampling profiler) |
| `dotnet-dump` | Crash dump анализ |
| `dotnet-gcdump` | GC heap snapshot |
| `dotnet-sos` | SOS extension для WinDbg/lldb |
| `dotnet-symbol` | Symbol downloader |
| `dotnet-monitor` | Sidecar для Kubernetes мониторинга |
| `dotnet-script` | Запускать `.csx`-скрипты |
| `dotnet-coverage` | Code coverage сбор |
| `nbgv` (Nerdbank.GitVersioning) | Авто-версионирование на git |
| `csharprepl` | C# REPL в терминале |

### 11.5. Manifest для team-based проектов

`dotnet new tool-manifest` создаёт `.config/dotnet-tools.json`. Вот production-ready пример:

```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "dotnet-ef": {
      "version": "10.0.0",
      "commands": ["dotnet-ef"]
    },
    "dotnet-format": {
      "version": "5.1.250801",
      "commands": ["dotnet-format"]
    },
    "dotnet-outdated-tool": {
      "version": "4.6.4",
      "commands": ["dotnet-outdated"]
    }
  }
}
```

В CI:

```yaml
- name: Restore tools
  run: dotnet tool restore

- name: Format check
  run: dotnet format --verify-no-changes

- name: Outdated check
  run: dotnet outdated --fail-on-updates
```

### 11.6. Создание собственного tool

Любой console-приложение можно превратить в tool:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <PackAsTool>true</PackAsTool>
    <ToolCommandName>my-cli</ToolCommandName>
    <PackageId>MyCompany.MyCli</PackageId>
    <Version>1.0.0</Version>
  </PropertyGroup>
</Project>
```

```bash
dotnet pack -c Release -o ./nupkgs
dotnet nuget push ./nupkgs/MyCompany.MyCli.1.0.0.nupkg --api-key $KEY --source https://api.nuget.org/v3/index.json

# Теперь можно установить:
dotnet tool install -g MyCompany.MyCli
my-cli --help
```

> [!question]- Интервью: чем local tools отличаются от global tools?
> Global tools — установлены однажды в `~/.dotnet/tools/`, доступны везде, одна версия для всех проектов на машине. Local tools — привязаны к проекту через `.config/dotnet-tools.json` манифест, версии зафиксированы и коммитятся в git, разные проекты могут иметь разные версии. Local — production-стандарт: все разработчики и CI после `dotnet tool restore` получают одинаковый набор tools с одинаковыми версиями. Global — для редко используемых утилит (dotnet-counters, csharprepl).

---

## 12. NuGet через CLI

### 12.1. Add / remove / update

```bash
# Установить последнюю версию
dotnet add package Newtonsoft.Json

# Конкретная версия
dotnet add package Newtonsoft.Json --version 13.0.3

# Конкретный source (если разные feeds)
dotnet add package InternalLib --source https://nuget.company.com/v3/index.json

# Препроцессированная (preview) версия
dotnet add package SomePackage --prerelease

# Удалить
dotnet remove package Newtonsoft.Json

# Обновить — нет прямой команды. Только через:
dotnet add package Newtonsoft.Json    # обновит до latest
# или вручную в csproj
```

### 12.2. List и проверки

```bash
# Список установленных пакетов
dotnet list package

# Включая transitive
dotnet list package --include-transitive

# Устаревшие (есть newer)
dotnet list package --outdated

# Уязвимые (CVE — Microsoft GitHub Advisory Database)
dotnet list package --vulnerable
dotnet list package --vulnerable --include-transitive   # включая transitive

# С deprecated (помеченные авторами как «больше не использовать»)
dotnet list package --deprecated
```

`--vulnerable` — критично проверять регулярно. Можно в CI:

```yaml
- name: Vulnerability check
  run: |
    dotnet list package --vulnerable --include-transitive 2>&1 | tee vuln.txt
    if grep -q "has the following vulnerable" vuln.txt; then
      echo "Found vulnerable packages!"
      exit 1
    fi
```

### 12.3. NuGet sources

```bash
# Список источников
dotnet nuget list source

# Добавить
dotnet nuget add source https://nuget.company.com/v3/index.json -n company

# С credentials
dotnet nuget add source https://nuget.company.com/v3/index.json \
  -n company -u user -p "$TOKEN" --store-password-in-clear-text

# Удалить
dotnet nuget remove source company

# Включить / отключить временно
dotnet nuget enable source company
dotnet nuget disable source company

# Установить как default
dotnet nuget set source company --priority 1
```

### 12.4. NuGet.Config — приоритеты

`NuGet.Config` решает несколько вопросов:

- Какие источники.
- В каком порядке.
- Где кэш.
- Где `packages.lock.json`.

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
    <add key="company" value="https://nuget.company.com/v3/index.json" protocolVersion="3" />
  </packageSources>

  <packageSourceMapping>
    <packageSource key="nuget.org">
      <package pattern="*" />
    </packageSource>
    <packageSource key="company">
      <package pattern="MyCompany.*" />
    </packageSource>
  </packageSourceMapping>

  <config>
    <add key="globalPackagesFolder" value="C:\nuget-cache" />
  </config>
</configuration>
```

`packageSourceMapping` (NuGet 6.0+) — supply chain security: конкретные пакеты искать только в конкретных feed-ах. `MyCompany.*` ищется только в `company`, иначе атакующий мог бы залить вредоносный `MyCompany.SuperSecret` в публичный nuget.org и подменить.

### 12.5. Cache management

```bash
# Очистить локальный кэш пакетов (полезно при corruption)
dotnet nuget locals all --clear

# Только http-cache
dotnet nuget locals http-cache --clear

# Только plugins-cache
dotnet nuget locals plugins-cache --clear

# Только global-packages
dotnet nuget locals global-packages --clear

# Показать пути кэшей (без очистки)
dotnet nuget locals all --list
```

### 12.6. Auditing — security в .NET 8+

`dotnet add` / `dotnet restore` теперь автоматически проверяет известные уязвимости и предупреждает:

```
warning NU1903: Package 'Newtonsoft.Json' 13.0.1 has a known high severity vulnerability,
https://github.com/advisories/GHSA-5crp-9r3c-p9vr
```

Включить как ошибку:

```xml
<PropertyGroup>
  <NuGetAudit>true</NuGetAudit>
  <NuGetAuditMode>all</NuGetAuditMode>            <!-- direct + transitive -->
  <NuGetAuditLevel>moderate</NuGetAuditLevel>     <!-- minimum severity -->
  <WarningsAsErrors>NU1903</WarningsAsErrors>
</PropertyGroup>
```

> [!question]- Интервью: что такое packageSourceMapping и зачем нужен?
> Это NuGet 6.0+ фича, которая ограничивает пакеты по источникам. В `NuGet.Config` указываешь pattern → source, и NuGet ищет пакеты только там. Например, `MyCompany.*` — только в корпоративном feed, всё остальное — в nuget.org. Защищает от dependency confusion attack: если злоумышленник зальёт `MyCompany.Internal` в nuget.org, без mapping NuGet может скачать оттуда (если версия выше); с mapping — нет, ищется только в корпоративном.

---

## 13. user-secrets и configuration

### 13.1. Зачем user-secrets

В Development часто нужны секреты: connection string к локальной БД, API-ключи. Положить в `appsettings.json` — **нельзя**, оно коммитится в git. Положить в `appsettings.Development.json` — можно, но если разные разработчики работают с разными БД, конфликт.

Решение — `dotnet user-secrets`. Секреты хранятся **вне репозитория**, в профиле пользователя:

- Linux/macOS: `~/.microsoft/usersecrets/<UserSecretsId>/secrets.json`
- Windows: `%APPDATA%\Microsoft\UserSecrets\<UserSecretsId>\secrets.json`

Никогда в git, никогда в Docker-image.

### 13.2. Workflow

```bash
# В корне проекта — инициализировать user-secrets
dotnet user-secrets init
# Это добавит в csproj:
# <UserSecretsId>aspnet-MyApi-abc123-...</UserSecretsId>

# Установить секрет
dotnet user-secrets set "ConnectionStrings:Default" "Server=localhost;Database=mydb;User=sa;Password=SecretPwd!"

# Установить вложенное
dotnet user-secrets set "Jwt:SigningKey" "super-secret-key-32-chars-min"

# Список
dotnet user-secrets list

# Удалить один
dotnet user-secrets remove "Jwt:SigningKey"

# Очистить все
dotnet user-secrets clear

# Применить из json
cat secrets.json | dotnet user-secrets set
```

### 13.3. Чтение в ASP.NET Core

ASP.NET Core автоматически загружает user-secrets в development:

```csharp
var builder = WebApplication.CreateBuilder(args);
// CreateBuilder уже добавил:
// - appsettings.json
// - appsettings.{Environment}.json
// - User secrets (в Development)
// - Environment variables
// - Command-line args

string connStr = builder.Configuration.GetConnectionString("Default")!;
```

Порядок загрузки имеет значение — последующие переопределяют предыдущие. User-secrets загружаются после appsettings.Development.json, но до env vars и args.

### 13.4. Production — переменные окружения

В production user-secrets **не используются** (только Development). Производственные секреты — переменные окружения, Azure Key Vault, AWS Secrets Manager, HashiCorp Vault и т.д.

```bash
# Через ASP.NET Core конвенцию
ConnectionStrings__Default="Server=prod-db;..."
Jwt__SigningKey="prod-key"
```

`__` (двойное подчёркивание) — разделитель уровней (вместо `:`, который не валиден в env vars Linux).

### 13.5. Configuration Pipeline

```csharp
var builder = WebApplication.CreateBuilder(args);

// Кастомизация:
builder.Configuration
    .AddJsonFile("custom-config.json", optional: true)
    .AddKeyPerFile("/run/secrets", optional: true)   // Docker secrets
    .AddAzureKeyVault(...)                           // Azure Key Vault
    .AddEnvironmentVariables(prefix: "MYAPP_");

string connStr = builder.Configuration.GetConnectionString("Default")!;
var jwt = builder.Configuration.GetSection("Jwt").Get<JwtOptions>()!;
```

> [!question]- Интервью: где хранятся .NET user-secrets?
> Не в репозитории — в профиле пользователя: `~/.microsoft/usersecrets/<id>/secrets.json` (Linux/macOS) или `%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` (Windows). `<id>` — это `<UserSecretsId>` из csproj. Используются только в Development. В production — переменные окружения, Key Vault, Secrets Manager.

---

## 14. EF Core CLI

### 14.1. Установка

```bash
# Local tool (recommended)
dotnet new tool-manifest                        # один раз
dotnet tool install dotnet-ef --version 10.0.0  # в манифест

# Или global
dotnet tool install -g dotnet-ef
```

В csproj проекта с EF Core нужен:

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.0" />
```

Этот пакет даёт design-time компоненты, которыми пользуется CLI.

### 14.2. Migrations

```bash
# Создать миграцию
dotnet ef migrations add AddCustomerTable

# Применить миграции к БД
dotnet ef database update

# Применить до конкретной миграции (rollback вперёд/назад)
dotnet ef database update AddCustomerTable

# Откатить ВСЕ миграции
dotnet ef database update 0

# Удалить последнюю миграцию (если ещё не применена в БД)
dotnet ef migrations remove

# Список миграций
dotnet ef migrations list

# Generate SQL без применения (для DBA-ревью)
dotnet ef migrations script
dotnet ef migrations script --idempotent --output migration.sql
```

### 14.3. Несколько проектов

В реальном solution часто DbContext в одном проекте, миграции — в другом, startup — в третьем:

```bash
dotnet ef migrations add Init \
    --project src/MyApp.Infrastructure \
    --startup-project src/MyApp.Api
```

`--project` — где код DbContext. `--startup-project` — что запускать как entry-point (Application/Web), оттуда EF берёт DI configuration и connection string.

### 14.4. Несколько DbContext

```bash
dotnet ef migrations add InitOrders --context OrdersDbContext
dotnet ef migrations add InitUsers --context UsersDbContext
```

### 14.5. Scaffold (Database-first)

Если есть готовая БД и нужно сгенерировать сущности:

```bash
dotnet ef dbcontext scaffold \
    "Server=localhost;Database=existing;User=sa;Password=..." \
    Microsoft.EntityFrameworkCore.SqlServer \
    --output-dir Models \
    --context ExistingDbContext
```

Создаст entity-классы и DbContext под существующие таблицы. Полезно для миграции legacy-баз.

### 14.6. Tips для CI

В CI миграции обычно не применяются командой — это разворачивает SQL заранее:

```bash
# Создать SQL-скрипт миграции
dotnet ef migrations script --idempotent --output ./artifacts/migration.sql

# Скрипт выполнить отдельным шагом, через psql / sqlcmd
```

`--idempotent` создаёт SQL, который можно запускать повторно — если миграция уже применена, он её пропустит.

> [!question]- Интервью: что делает `dotnet ef migrations add` под капотом?
> 1. Загружает приложение (без запуска main pipeline) для DI-конфигурации DbContext. 2. Сравнивает текущую модель (DbContext + entities) с предыдущим snapshot (`<DbContext>ModelSnapshot.cs`). 3. Генерирует C#-класс `<Timestamp>_<Name>.cs` с методами `Up()` (применить) и `Down()` (откатить) — конкретные ALTER TABLE / CREATE TABLE и т.д. 4. Обновляет `ModelSnapshot.cs`. Сама миграция в БД не применяется — только генерируется. Применить — `dotnet ef database update`.

---

## 15. Workloads — расширения SDK

### 15.1. Что такое workload

Workload — дополнительный набор шаблонов и tooling, который ставится поверх SDK. Введено в .NET 6 для модуляризации:

```bash
# Список доступных
dotnet workload list

# Поиск
dotnet workload search

# Установить (нужны права sudo на Linux)
sudo dotnet workload install maui
sudo dotnet workload install wasm-tools
sudo dotnet workload install android

# Обновить установленные
sudo dotnet workload update

# Удалить
sudo dotnet workload uninstall maui
```

### 15.2. Когда какой workload

| Workload | Зачем |
|----------|-------|
| `maui` | MAUI — кросс-платформенные мобильные/desktop приложения |
| `android` | Android-targeting (часть MAUI или standalone) |
| `ios` / `maccatalyst` | iOS / Mac Catalyst |
| `wasm-tools` | Blazor WebAssembly AOT compilation, оптимизация |
| `aspire` | .NET Aspire — orchestration cloud-native приложений |

### 15.3. Restore workloads из проекта

В `global.json` можно указать workloads:

```json
{
  "sdk": {
    "version": "10.0.100"
  },
  "msbuild-sdks": {
  }
}
```

Если csproj использует workload (например `<TargetFramework>net10.0-android</TargetFramework>`), команда:

```bash
dotnet workload restore
```

— установит нужные workloads автоматически. Полезно в CI: новый build agent поднимает workloads сам.

### 15.4. Где живут workloads

`/usr/share/dotnet/sdk-manifests/<sdk-version>/` (Linux). Каждый workload — пакет с manifest и набором SDK-расширений.

> [!question]- Интервью: зачем .NET workloads?
> Чтобы базовый SDK не вырастал до 5 GB, включая Android, iOS, MAUI tooling. С .NET 6 SDK ~700 MB, дополнительные target-platforms ставятся отдельно командой `dotnet workload install`. Это даёт быстрее install базового SDK для веб-разработчиков (которым MAUI не нужен) и независимое обновление SDK vs target-platforms.

---

## 16. Multi-targeting и TFM

### 16.1. TFM — Target Framework Moniker

TFM — короткая строка, идентифицирующая комбинацию runtime + версия + (опционально) платформа:

| TFM | Что значит |
|-----|-----------|
| `net10.0` | .NET 10 (cross-platform) |
| `net8.0` | .NET 8 (LTS) |
| `net6.0` | .NET 6 (LTS до ноября 2024) |
| `netstandard2.0` | .NET Standard 2.0 — переносимый между .NET Framework, Mono, .NET Core |
| `netstandard2.1` | .NET Standard 2.1 — последняя версия (не поддерживается .NET Framework) |
| `net48` | .NET Framework 4.8 (legacy Windows) |
| `net10.0-windows` | .NET 10 + Windows-specific (WPF, WinForms) |
| `net10.0-android` | .NET 10 + Android |
| `net10.0-ios` | .NET 10 + iOS |
| `net10.0-browser` | .NET 10 + browser (Blazor WebAssembly) |

### 16.2. Single TFM

Большинство проектов:

```xml
<TargetFramework>net10.0</TargetFramework>
```

Один TFM — собирается под него, использует все возможные API.

### 16.3. Multi-targeting

Если библиотека должна поддерживать несколько runtime:

```xml
<TargetFrameworks>net6.0;net8.0;net10.0;netstandard2.0</TargetFrameworks>
```

`build` соберёт **по одной DLL на TFM**. NuGet-пакет (через `pack`) включит все варианты.

Использование `#if` для conditional API:

```csharp
public class HttpClient
{
#if NET6_0_OR_GREATER
    public async ValueTask<string> GetStringAsync(Uri uri, CancellationToken ct = default) =>
        await _client.GetStringAsync(uri, ct);
#else
    public Task<string> GetStringAsync(Uri uri) =>
        _client.GetStringAsync(uri);
#endif
}
```

Compiler определяет константы автоматически: `NET6_0`, `NET6_0_OR_GREATER`, `NET10_0`, `NETSTANDARD2_0`.

### 16.4. .NET Standard — когда нужен

`netstandard2.0` — наименьший общий знаменатель между .NET Framework 4.7+, Mono, Xamarin, .NET Core 2+. Если библиотеку могут использовать legacy-проекты на .NET Framework — таргетируй `netstandard2.0`.

В новом коде, где нет .NET Framework — `netstandard2.0` устарел. Microsoft рекомендует:

- Для библиотек **общего применения** — `net8.0` (текущая LTS).
- Для библиотек, поддерживающих **многие версии .NET** — multi-target `net6.0;net8.0;net10.0`.
- Для библиотек, поддерживающих **.NET Framework + .NET Core** — `netstandard2.0`.

### 16.5. Как выбирается TFM при референсе

Когда `MyApp` (TFM `net10.0`) подключает `MyLib` (multi-target `net6.0;net8.0;net10.0`), NuGet выбирает **самый близкий совместимый TFM** из библиотеки. В этом случае — `net10.0`. Если бы у `MyLib` был только `net6.0` — взялся бы он, при условии что .NET 10 совместим с .NET 6 (это так — runtime forward-compatible).

> [!question]- Интервью: чем отличается `net10.0` от `netstandard2.0`?
> `net10.0` — полная .NET 10, включает все API runtime (Span, Memory, Generic Math и т.д.). `netstandard2.0` — спецификация API, common subset поддерживаемый .NET Framework 4.7.2+, Mono, Xamarin, .NET Core 2+. Сборки `netstandard2.0` запускаются на любой из этих платформ. Применяй netstandard2.0, если библиотеку используют legacy .NET Framework приложения. В новом коде — обычно прямой `net8.0` или multi-target.

---

## 17. CI/CD recipes

### 17.1. GitHub Actions — полный workflow

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 10.0.x

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/packages.lock.json') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore tools
        run: dotnet tool restore

      - name: Restore packages
        run: dotnet restore --locked-mode

      - name: Format check
        run: dotnet format --verify-no-changes

      - name: Build
        run: dotnet build -c Release --no-restore

      - name: Test with coverage
        run: |
          dotnet test -c Release --no-build \
            --logger "trx;LogFileName=test.trx" \
            --collect:"XPlat Code Coverage" \
            --results-directory ./TestResults

      - name: Vulnerability check
        run: dotnet list package --vulnerable --include-transitive 2>&1 | tee vuln.txt
        continue-on-error: false

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: TestResults/

      - name: Publish (if main branch)
        if: github.ref == 'refs/heads/main'
        run: dotnet publish src/MyApi/MyApi.csproj -c Release -o ./publish

      - name: Upload artifact
        if: github.ref == 'refs/heads/main'
        uses: actions/upload-artifact@v4
        with:
          name: app-bundle
          path: publish/
```

Что важно:

- `actions/cache` для NuGet — экономит 30-60 секунд на повторных сборках.
- `--locked-mode` гарантирует воспроизводимость.
- `tool restore` для local tools.
- `dotnet format --verify-no-changes` — fail если код не отформатирован.
- Coverage и trx — для отчётности.

### 17.2. Multi-stage Dockerfile

```dockerfile
# === Stage 1: Restore (для кэширования) ===
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS restore
WORKDIR /src

COPY *.sln ./
COPY src/MyApi/*.csproj src/MyApi/
COPY src/MyApi.Domain/*.csproj src/MyApi.Domain/
COPY tests/MyApi.Tests/*.csproj tests/MyApi.Tests/

RUN dotnet restore

# === Stage 2: Build & test ===
FROM restore AS build
COPY . .
RUN dotnet build -c Release --no-restore

FROM build AS test
RUN dotnet test -c Release --no-build --no-restore

# === Stage 3: Publish ===
FROM build AS publish
RUN dotnet publish src/MyApi/MyApi.csproj -c Release \
    --no-build --no-restore \
    -o /app/publish

# === Stage 4: Runtime (минимальный image) ===
FROM mcr.microsoft.com/dotnet/aspnet:10.0-alpine AS runtime
WORKDIR /app
COPY --from=publish /app/publish .

USER 1000
ENTRYPOINT ["dotnet", "MyApi.dll"]
EXPOSE 8080
```

Финальный image — тонкий (~120 MB на alpine), без SDK.

### 17.3. Native AOT image (.NET 8+)

Для AOT можно использовать **distroless** или **scratch** image:

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

COPY . .
RUN dotnet publish -c Release -r linux-x64 \
    -p:PublishAot=true \
    -o /app/publish

FROM mcr.microsoft.com/dotnet/runtime-deps:10.0-alpine AS runtime
WORKDIR /app
COPY --from=build /app/publish/MyApi ./

USER 1000
ENTRYPOINT ["./MyApi"]
EXPOSE 8080
```

Финальный image — ~30-50 MB, без CLR/JIT, instant startup.

### 17.4. GitLab CI

```yaml
image: mcr.microsoft.com/dotnet/sdk:10.0

stages:
  - build
  - test
  - deploy

cache:
  paths:
    - .nuget/

variables:
  DOTNET_CLI_TELEMETRY_OPTOUT: "1"
  NUGET_PACKAGES: "$CI_PROJECT_DIR/.nuget"

build:
  stage: build
  script:
    - dotnet restore
    - dotnet build -c Release --no-restore

test:
  stage: test
  script:
    - dotnet test -c Release --no-build --logger "junit;LogFilePath=test-results.xml"
  artifacts:
    when: always
    reports:
      junit: "**/test-results.xml"

deploy:
  stage: deploy
  only:
    - main
  script:
    - dotnet publish src/MyApi -c Release -o ./publish
    - docker build -t myapi:$CI_COMMIT_SHA .
    - docker push myapi:$CI_COMMIT_SHA
```

### 17.5. Azure DevOps Pipelines

```yaml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'

steps:
  - task: UseDotNet@2
    inputs:
      version: '10.0.x'

  - task: DotNetCoreCLI@2
    displayName: Restore
    inputs:
      command: restore
      projects: '**/*.csproj'

  - task: DotNetCoreCLI@2
    displayName: Build
    inputs:
      command: build
      projects: '**/*.csproj'
      arguments: '-c $(buildConfiguration) --no-restore'

  - task: DotNetCoreCLI@2
    displayName: Test
    inputs:
      command: test
      projects: '**/tests/*.csproj'
      arguments: '-c $(buildConfiguration) --no-build --collect "Code coverage"'

  - task: DotNetCoreCLI@2
    displayName: Publish
    inputs:
      command: publish
      publishWebProjects: false
      projects: 'src/MyApi/MyApi.csproj'
      arguments: '-c $(buildConfiguration) -o $(Build.ArtifactStagingDirectory)'
```

> [!question]- Интервью: как кэшировать NuGet-пакеты в CI?
> В GitHub Actions — `actions/cache@v4` с путём `~/.nuget/packages` и ключом по hash от csproj/`packages.lock.json`. В GitLab — `cache: paths: [.nuget/]`. В Docker — multi-stage Dockerfile, где `COPY *.csproj` отдельно от `COPY .` обеспечивает кэширование RUN dotnet restore. Это экономит 30-60 секунд на каждой сборке (для проектов с большим числом зависимостей — больше).

---

## 18. Common Pitfalls — с механизмами

### 18.1. dotnet build падает после клонирования

```bash
git clone repo
cd repo
dotnet build
# error: package not found
```

**Механизм:** restore не успел запуститься (или сломался).

**Фикс:**
```bash
dotnet restore
dotnet build
```

В .NET 6+ `build` автоматически зовёт `restore`, но иногда (старая версия SDK, broken csproj) нужно явно.

### 18.2. SDK version mismatch — NETSDK1045

```
error NETSDK1045: The current .NET SDK does not support targeting .NET 10.0.
```

**Механизм:** csproj указывает `<TargetFramework>net10.0</TargetFramework>`, но установлен только SDK 8.

**Фикс:** установить SDK 10 (`winget install Microsoft.DotNet.SDK.10`) или использовать TargetFramework `net8.0`.

### 18.3. dotnet run не находит проект

```bash
dotnet run
# error: could not find any project to run
```

**Механизм:** в текущей папке нет csproj (например, ты в корне solution с папками `src/`, `tests/`).

**Фикс:**
```bash
dotnet run --project src/MyApi/MyApi.csproj
# или
cd src/MyApi
dotnet run
```

### 18.4. Использовал bin/Debug в production

```bash
# ❌ Скопировал содержимое bin/Debug на сервер
```

**Механизм:** `bin/Debug` содержит non-optimized DLL и debug-символы. Это для разработки. `bin/Release` — оптимизировано, но всё ещё может включать pdb. **Только** `dotnet publish` создаёт правильный bundle для production.

**Фикс:**
```bash
dotnet publish -c Release -o ./publish
# Деплоить ./publish, не bin/...
```

### 18.5. Hot reload не подхватил изменение

```bash
dotnet watch run
# Изменил Program.cs — приложение не перезапустилось
```

**Механизм:** изменение могло быть «rude edit» (сигнатура метода, async, поля), которое hot reload не умеет применить.

**Фикс:** в watch-консоли нажать `r` (restart) или Ctrl+C / повторить `dotnet watch run`.

### 18.6. EF migration падает с «No service for type ... DbContext»

```bash
dotnet ef migrations add Init
# error: No service for type 'AppDbContext' has been registered
```

**Механизм:** EF загружает приложение через `IDesignTimeDbContextFactory` или Startup. Если в startup-проекте нет DI-регистрации DbContext или нет `IDesignTimeDbContextFactory<T>` — ломается.

**Фикс:** либо добавить DI в startup (обычно в `Program.cs`), либо реализовать factory:

```csharp
public class DesignTimeFactory : IDesignTimeDbContextFactory<AppDbContext>
{
    public AppDbContext CreateDbContext(string[] args)
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer("Server=localhost;Database=design;...")
            .Options;
        return new AppDbContext(options);
    }
}
```

### 18.7. Tool installed but not found in PATH

```bash
dotnet tool install -g dotnet-ef
dotnet ef
# command not found
```

**Механизм:** `~/.dotnet/tools` не в `$PATH`.

**Фикс:** добавить в `~/.bashrc` / `~/.zshrc`:
```bash
export PATH="$PATH:$HOME/.dotnet/tools"
```

Перезайти в shell.

### 18.8. Restore «зависает» в CI

**Механизм:** NuGet пытается достучаться до источника, который недоступен (приватный feed, забыли credentials).

**Фикс:** настроить timeouts и явные источники:

```bash
dotnet restore \
    --source https://api.nuget.org/v3/index.json \
    --source https://nuget.company.com/v3/index.json \
    --runtime linux-x64 \
    -p:RestoreSources="..."
```

Проверить `dotnet nuget list source` — нет ли отключённых.

### 18.9. PublishTrimmed сломал runtime

```bash
dotnet publish -c Release -p:PublishTrimmed=true -r linux-x64 --self-contained
# Запуск
./MyApi
# Internal exception: TypeLoadException...
```

**Механизм:** trimming выкинул классы, которые используются через reflection (EF entities, JSON serialization).

**Фикс:**
- Помечать assembly atribute: `[assembly: System.Diagnostics.CodeAnalysis.DynamicallyAccessedMembers]`.
- Добавить `<TrimmerRootAssembly>` для assembly с reflection-зависимым кодом:

```xml
<ItemGroup>
  <TrimmerRootAssembly Include="MyApi.Models" />
</ItemGroup>
```

- Использовать `JsonSerializerContext` source generator для AOT-friendly сериализации.

### 18.10. Workload или templates устарели

```bash
dotnet new install Clean.Architecture.Solution.Template
# Old version, нужна новая
```

**Механизм:** ты установил template год назад, а с тех пор шаблон обновился.

**Фикс:**
```bash
dotnet new uninstall Clean.Architecture.Solution.Template
dotnet new install Clean.Architecture.Solution.Template
# или
dotnet new update
```

> [!question]- Интервью: пять самых частых ошибок в `dotnet` CLI.
> 1. Забыл restore после клонирования. 2. SDK version mismatch — нет SDK 10, csproj требует. 3. `dotnet run` без `--project` в папке без csproj. 4. Деплой `bin/Debug` вместо `dotnet publish`. 5. Tool installed globally но не в PATH.

---

## 19. Best Practices

- **Используй `global.json`** в репозитории для пиннинга SDK-версии. Тестируется одинаково локально, на CI, в Docker.
- **Local tools через manifest** — `dotnet new tool-manifest` + `dotnet tool install dotnet-ef`. Версии фиксируются и коммитятся.
- **Directory.Build.props** в монорепе — общие настройки `TargetFramework`, `Nullable`, `LangVersion`, `TreatWarningsAsErrors`.
- **Centralized Package Management (CPM)** — `Directory.Packages.props` для версий пакетов в одном месте.
- **`packages.lock.json` + `--locked-mode`** в CI — гарантия воспроизводимости.
- **`dotnet user-secrets`** для development-секретов. Никогда — `appsettings.json`.
- **Multi-stage Dockerfile** — раздельные restore и build layer'ы, runtime image без SDK.
- **`dotnet format`** в pre-commit hook или CI — единый стиль кода.
- **Vulnerability scan** в CI — `dotnet list package --vulnerable --include-transitive`.
- **Solution-level commands** — `dotnet build`, `dotnet test` без аргументов в корне работают со всем солюшеном.
- **Native AOT для serverless / edge** — экстремальный cold start, минимальный размер.
- **Trimmed publish для веб-сервисов** — даёт меньший Docker image без AOT-ограничений.
- **`dotnet watch run`** для разработки — экономит секунды на перезапусках.
- **`dotnet test --filter "Category=Smoke"`** — отдельный быстрый smoke-pass на каждом push, full integration на nightly.
- **NuGet `packageSourceMapping`** — supply chain security, защита от dependency confusion.
- **Не коммить `bin/`, `obj/`, `*.user`, `.vs/`, `appsettings.*.local.json`** — `dotnet new gitignore` создаёт правильный.

---

## 20. Decision tree

```
Что нужно сделать?
│
├── Создать проект
│   ├── Console — `dotnet new console -n Name`
│   ├── Web API — `dotnet new webapi -n Name`
│   ├── Class library — `dotnet new classlib -n Name`
│   ├── Tests (xunit) — `dotnet new xunit -n Name.Tests`
│   ├── Solution — `dotnet new sln -n Solution`
│   └── File-based script (.NET 11) — `dotnet new singlecs -n script`
│
├── Добавить пакет / reference
│   ├── NuGet package — `dotnet add package Name [--version X]`
│   ├── Project reference — `dotnet add reference path/to/other.csproj`
│   └── В solution — `dotnet sln add path/to/Project.csproj`
│
├── Запустить
│   ├── Один раз — `dotnet run`
│   ├── Watch mode (hot reload) — `dotnet watch run`
│   ├── Конкретный проект — `dotnet run --project path/Proj.csproj`
│   ├── Production-like — `dotnet run -c Release`
│   └── С launch profile — `dotnet run --launch-profile https`
│
├── Тестировать
│   ├── Все — `dotnet test`
│   ├── Filter — `dotnet test --filter "Category=Unit"`
│   ├── Coverage — `dotnet test --collect:"XPlat Code Coverage"`
│   └── Watch — `dotnet watch test`
│
├── Готовить к deploy
│   ├── Framework-dependent — `dotnet publish -c Release -o ./out`
│   ├── Self-contained — `dotnet publish -c Release -r linux-x64 --self-contained -o ./out`
│   ├── Single file — `dotnet publish ... -p:PublishSingleFile=true`
│   ├── Trimmed — `dotnet publish ... -p:PublishTrimmed=true`
│   ├── ReadyToRun — `dotnet publish ... -p:PublishReadyToRun=true`
│   └── Native AOT — `dotnet publish ... -p:PublishAot=true`
│
├── Создать NuGet пакет
│   └── `dotnet pack -c Release -o ./nupkgs` + push через `dotnet nuget push`
│
├── Установить tool
│   ├── Глобально (своя машина) — `dotnet tool install -g name`
│   └── Локально (для проекта) — `dotnet new tool-manifest` + `dotnet tool install name`
│
├── EF Core миграции
│   ├── Создать — `dotnet ef migrations add Name`
│   ├── Применить — `dotnet ef database update`
│   ├── SQL для DBA — `dotnet ef migrations script --idempotent --output m.sql`
│   └── С разными проектами — `--project ... --startup-project ...`
│
├── Управление SDK
│   ├── Версия — `dotnet --version`
│   ├── Список — `dotnet --list-sdks`
│   ├── Pin — `dotnet new globaljson --sdk-version X`
│   └── Workloads — `sudo dotnet workload install maui`
│
└── Решение проблем
    ├── Кэш испорчен — `dotnet nuget locals all --clear`
    ├── Inкр. build не отрабатывает — `dotnet clean` + `dotnet build`
    ├── Vulnerable packages — `dotnet list package --vulnerable --include-transitive`
    └── Format issues — `dotnet format`
```

---

## 21. Cheat sheet

```bash
# === Информация ===
dotnet --version                                    # текущий SDK
dotnet --info                                       # подробная инфа
dotnet --list-sdks                                  # все установленные SDK
dotnet --list-runtimes                              # все runtime

# === Создание ===
dotnet new console -n MyApp                         # console
dotnet new webapi -n MyApi                          # Web API
dotnet new classlib -n MyLib                        # библиотека
dotnet new xunit -n MyTests                         # xUnit tests
dotnet new sln -n MySolution                        # solution
dotnet new gitignore                                # .gitignore
dotnet new editorconfig                             # .editorconfig
dotnet new globaljson --sdk-version 10.0.100        # global.json
dotnet new tool-manifest                            # для local tools
dotnet new list                                     # все шаблоны

# === Solution и references ===
dotnet sln add src/MyApi/MyApi.csproj               # добавить в sln
dotnet sln list                                     # список проектов
dotnet add MyApi reference MyDomain                 # project reference

# === Packages ===
dotnet add package Newtonsoft.Json                  # установить
dotnet add package Newtonsoft.Json --version 13.0.3 # конкретная версия
dotnet remove package Newtonsoft.Json               # удалить
dotnet list package                                 # список
dotnet list package --outdated                      # устаревшие
dotnet list package --vulnerable --include-transitive  # уязвимые

# === Build / Run / Test ===
dotnet restore                                      # restore packages
dotnet build                                        # build
dotnet build -c Release                             # release build
dotnet build --no-restore                           # без restore
dotnet run                                          # run
dotnet run --project src/MyApi                      # конкретный
dotnet run -c Release                               # release
dotnet watch run                                    # hot reload
dotnet test                                         # все тесты
dotnet test --filter "FullyQualifiedName~MyTest"    # filter
dotnet test --collect:"XPlat Code Coverage"         # с coverage
dotnet clean                                        # очистить

# === Publish (deploy) ===
dotnet publish -c Release -o ./publish              # framework-dep
dotnet publish -c Release -r linux-x64 --self-contained -o ./publish
dotnet publish -c Release -r linux-x64 -p:PublishSingleFile=true --self-contained
dotnet publish -c Release -r linux-x64 -p:PublishTrimmed=true --self-contained
dotnet publish -c Release -r linux-x64 -p:PublishAot=true

# === NuGet packages (создание) ===
dotnet pack -c Release -o ./nupkgs                  # создать .nupkg
dotnet nuget push ./pkg.nupkg --api-key X --source https://api.nuget.org/v3/index.json

# === Tools ===
dotnet tool install -g dotnet-ef                    # global
dotnet tool list -g                                 # глобальные
dotnet tool install dotnet-ef                       # local (нужен manifest)
dotnet tool restore                                 # из манифеста
dotnet tool update -g name                          # обновить

# === User secrets ===
dotnet user-secrets init                            # инит
dotnet user-secrets set "Key" "Value"               # установить
dotnet user-secrets list                            # список
dotnet user-secrets clear                           # очистить

# === EF Core ===
dotnet ef migrations add Name                       # новая миграция
dotnet ef database update                           # применить
dotnet ef migrations remove                         # удалить последнюю
dotnet ef migrations script --idempotent --output m.sql  # SQL
dotnet ef dbcontext scaffold "ConnStr" Provider     # database-first

# === Workloads ===
sudo dotnet workload install maui
sudo dotnet workload update
dotnet workload list

# === NuGet sources ===
dotnet nuget list source
dotnet nuget add source URL -n NAME
dotnet nuget remove source NAME
dotnet nuget locals all --clear                     # очистить кэш

# === Format ===
dotnet format                                       # отформатировать
dotnet format --verify-no-changes                   # check (для CI)
```

---

## 22. Practice — упражнения с разбором

### 22.1. Создать структуру solution с нуля

**Задача.** Создать solution `OrderApp` с тремя проектами: `OrderApp.Api` (Web API), `OrderApp.Domain` (class library), `OrderApp.Tests` (xUnit). Настроить references и запустить.

```bash
# 1. Папка и solution
mkdir OrderApp && cd OrderApp
dotnet new sln -n OrderApp

# 2. Проекты
dotnet new webapi -n OrderApp.Api -o src/OrderApp.Api
dotnet new classlib -n OrderApp.Domain -o src/OrderApp.Domain
dotnet new xunit -n OrderApp.Tests -o tests/OrderApp.Tests

# 3. В solution
dotnet sln add src/OrderApp.Api/OrderApp.Api.csproj
dotnet sln add src/OrderApp.Domain/OrderApp.Domain.csproj
dotnet sln add tests/OrderApp.Tests/OrderApp.Tests.csproj

# 4. References
dotnet add src/OrderApp.Api reference src/OrderApp.Domain
dotnet add tests/OrderApp.Tests reference src/OrderApp.Domain
dotnet add tests/OrderApp.Tests reference src/OrderApp.Api

# 5. .gitignore + global.json
dotnet new gitignore
dotnet new globaljson --sdk-version 10.0.100

# 6. Build all
dotnet build

# 7. Запустить API
cd src/OrderApp.Api && dotnet run
```

**Разбор:** структура `src/` + `tests/` — стандарт. Solution связывает всё вместе для команд `dotnet build` / `dotnet test` в корне. References направлены: Tests → API → Domain (Domain ничего не знает про остальных). global.json фиксирует SDK для CI и команды.

### 22.2. Multi-stage Dockerfile с restore-каэшированием

**Задача.** Написать Dockerfile, который кэширует NuGet-restore отдельным слоем, чтобы пересборка кода не требовала повторного скачивания пакетов.

```dockerfile
# Stage 1: restore (кэшируется отдельно)
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS restore
WORKDIR /src

# Скопировать ТОЛЬКО csproj/sln — это input для restore
COPY *.sln ./
COPY src/OrderApp.Api/*.csproj src/OrderApp.Api/
COPY src/OrderApp.Domain/*.csproj src/OrderApp.Domain/
COPY tests/OrderApp.Tests/*.csproj tests/OrderApp.Tests/

RUN dotnet restore

# Stage 2: build — restored layer переиспользуется если csproj не менялся
FROM restore AS build
COPY . .
RUN dotnet build -c Release --no-restore

# Stage 3: publish
FROM build AS publish
RUN dotnet publish src/OrderApp.Api -c Release --no-build --no-restore -o /app/publish

# Stage 4: runtime — без SDK
FROM mcr.microsoft.com/dotnet/aspnet:10.0-alpine AS runtime
WORKDIR /app
COPY --from=publish /app/publish .

USER 1000
ENTRYPOINT ["dotnet", "OrderApp.Api.dll"]
EXPOSE 8080
```

**Разбор:** ключевая идея — `COPY *.csproj` отдельно от `COPY .`. Docker кэширует layer'ы по контексту. Если csproj не изменился, RUN restore берётся из кэша. Изменение `Program.cs` не инвалидирует restore-layer. Это ускоряет re-build с 60s до 5s.

### 22.3. CI workflow — GitHub Actions

**Задача.** Написать CI, который при push в main: restore (с cache), build, test (с coverage), vuln-check, format-check.

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: 10.0.x

      - name: Cache NuGet
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: ${{ runner.os }}-nuget-

      - name: Restore tools
        run: dotnet tool restore

      - name: Restore packages
        run: dotnet restore

      - name: Format check
        run: dotnet format --verify-no-changes

      - name: Build
        run: dotnet build -c Release --no-restore

      - name: Test with coverage
        run: |
          dotnet test -c Release --no-build \
            --logger "trx" \
            --collect:"XPlat Code Coverage" \
            --results-directory TestResults

      - name: Vuln check
        run: |
          OUTPUT=$(dotnet list package --vulnerable --include-transitive 2>&1)
          echo "$OUTPUT"
          if echo "$OUTPUT" | grep -q "has the following vulnerable"; then
            exit 1
          fi

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: TestResults/
```

**Разбор:** кэш NuGet даёт +30-60 секунд экономии. `dotnet format --verify-no-changes` ловит несформатированный код. Vuln-check падает на CVE — заставляет регулярно обновлять зависимости. `if: always()` для upload — артефакты сохраняются даже при провале тестов (для разбора).

### 22.4. Local tools manifest + EF Core

**Задача.** Настроить локально проект с EF Core 10 через manifest, чтобы все разработчики получали одинаковый dotnet-ef.

```bash
# 1. Корень solution
dotnet new tool-manifest

# 2. Добавить EF tool
dotnet tool install dotnet-ef --version 10.0.0

# 3. Добавить EF design package в проект
dotnet add src/OrderApp.Infrastructure package Microsoft.EntityFrameworkCore.Design

# 4. .config/dotnet-tools.json должен быть в git
git add .config/dotnet-tools.json

# 5. Использование (после клонирования)
dotnet tool restore
dotnet ef migrations add Init \
    --project src/OrderApp.Infrastructure \
    --startup-project src/OrderApp.Api
dotnet ef database update \
    --project src/OrderApp.Infrastructure \
    --startup-project src/OrderApp.Api
```

**Разбор:** local tools версионируются в git. Новый разработчик клонирует и `dotnet tool restore` — всё готово. CI делает то же. Никакого «у Алисы EF 8, у Боба EF 10» из-за глобальной установки.

### 22.5. Native AOT публикация для serverless

**Задача.** Настроить проект так, чтобы `dotnet publish` создавал native-binary без CLR, размером < 30 MB, для AWS Lambda (linux-x64).

```xml
<!-- MyApi.csproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <PublishAot>true</PublishAot>
    <InvariantGlobalization>true</InvariantGlobalization>
    <StackTraceSupport>false</StackTraceSupport>
    <DebuggerSupport>false</DebuggerSupport>
    <UseSystemResourceKeys>true</UseSystemResourceKeys>
    <EventSourceSupport>false</EventSourceSupport>
    <HttpActivityPropagationSupport>false</HttpActivityPropagationSupport>
    <MetadataUpdaterSupport>false</MetadataUpdaterSupport>
  </PropertyGroup>
</Project>
```

```bash
dotnet publish -c Release -r linux-x64 -o ./publish
```

```dockerfile
# Lambda-friendly image
FROM public.ecr.aws/lambda/provided:al2023
COPY ./publish/MyApi /var/runtime/bootstrap
CMD ["MyApi"]
```

**Разбор:** PublishAot=true — native compilation. Дополнительные feature-flags (`InvariantGlobalization`, `StackTraceSupport=false` и т.д.) сокращают размер ещё на 30-50%. JSON serialization потребует `JsonSerializerContext`. Размер итог ~15-25 MB, cold start ~30ms на Lambda. Весь runtime — внутри binary.

---

## 23. Что читать дальше — порядок и почему

1. **[[csharp-basics|C# Basics]]** — первая программа после установки SDK. Теперь, когда CLI понятен, погружайся в язык.
2. **[[debugging-basics|Debugging Basics]]** — отладка через VS Code, Rider, dotnet-trace, dotnet-dump.
3. **[[testing-fundamentals|Testing Fundamentals]]** — `dotnet test` глубже: xUnit, Moq, FluentAssertions, integration tests.
4. **EF Core Basics** — миграции, `dotnet ef` подробно.
5. **ASP.NET Core Pipeline и Middleware** — `dotnet new webapi` создал — теперь разберись, что внутри.
6. **DI и Configuration** — IConfiguration, user-secrets, Options pattern.
7. **Native AOT (deep)** — после первого знакомства углубись.
8. **Docker для .NET** — multi-stage Dockerfile, optimization.
9. **CI/CD GitHub Actions** — pipeline-recipes для real-world проектов.

---

## 24. См. также

- [[csharp-basics|C# Basics]] — язык после установки SDK
- [[debugging-basics|Debugging Basics]] — отладка
- [[naming-conventions|Naming Conventions]] — стиль кода для проектов
- Native AOT — deep dive публикация
- Docker / Containerization
- CI/CD GitHub Actions
- Testing Fundamentals — `dotnet test` подробно
- EF Core CLI — миграции
- DI и Configuration — appsettings, user-secrets

---

## 25. Reading list

- **Microsoft Docs — .NET CLI overview** — learn.microsoft.com/dotnet/core/tools
- **Microsoft Docs — dotnet new templates** — learn.microsoft.com/dotnet/core/tools/dotnet-new
- **Microsoft Docs — global.json** — learn.microsoft.com/dotnet/core/tools/global-json
- **Microsoft Docs — Publishing** — learn.microsoft.com/dotnet/core/deploying
- **Microsoft Docs — Native AOT** — learn.microsoft.com/dotnet/core/deploying/native-aot
- **Microsoft Docs — NuGet** — learn.microsoft.com/nuget
- **Microsoft Docs — Trimming** — learn.microsoft.com/dotnet/core/deploying/trimming/trim-self-contained
- **Andrew Lock blog — .NET CLI tips** — andrewlock.net
- **Steve Gordon blog — Native AOT, ASP.NET deployment** — stevejgordon.co.uk
- **GitHub Actions for .NET** — github.com/actions/setup-dotnet
- **Microsoft Container images** — github.com/dotnet/dotnet-docker
- **dotnet/sdk source** — github.com/dotnet/sdk
- **dotnet/runtime source** — github.com/dotnet/runtime
- **NuGet/Home (issues, RFCs)** — github.com/NuGet/Home
- **.NET Aspire** — learn.microsoft.com/dotnet/aspire — orchestration cloud-native
- **SharpLab** — sharplab.io — посмотреть IL и оптимизации компилятора
