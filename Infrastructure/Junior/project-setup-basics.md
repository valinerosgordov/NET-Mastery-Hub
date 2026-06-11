---
tags: [infrastructure, project-setup, junior, solutions, git, dotnet-cli]
level: Junior
date: 2026-05-10
---

# Project Setup Basics — solution structure, git, daily workflow

> **dotnet CLI, solution + projects organization, .gitignore, basic git workflow для C# проектов.** Введение перед `Senior/project-setup.md` (production deep).

---

## 0. Как читать

Если впервые делаешь .NET solution от нуля — раздел 1 (CLI) → 2 (structure) → 3 (git). Если уже умеешь dotnet CLI — пропусти к 4 (multi-project) и 5 (configuration).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. dotnet CLI — basics

### 1.1. Зачем CLI

```
dotnet CLI — cross-platform, IDE-independent, scriptable.
Используешь:
- В терминале (быстрее IDE для simple ops)
- В CI/CD (GitHub Actions, GitLab CI)
- В Docker
- В скриптах
```

### 1.2. Verify install

```bash
dotnet --version           # версия SDK (8.0.xxx, 9.0.xxx)
dotnet --list-sdks         # все установленные SDKs
dotnet --list-runtimes     # все runtimes
dotnet --info              # detailed info про OS, SDKs
```

⚠️ Версия .NET SDK ≠ runtime version. SDK = build tools, runtime = execute apps.

### 1.3. Templates — что можно создать

```bash
# Список встроенных templates
dotnet new list

# Популярные:
dotnet new console      # console application
dotnet new web          # ASP.NET Core empty
dotnet new webapi       # ASP.NET Core Web API
dotnet new mvc          # ASP.NET Core MVC
dotnet new blazor       # Blazor Web App
dotnet new classlib     # class library
dotnet new xunit        # xUnit test project
dotnet new nunit        # NUnit test project
dotnet new mstest       # MSTest project
dotnet new sln          # solution file
dotnet new gitignore    # .gitignore для .NET
dotnet new editorconfig # .editorconfig
```

### 1.4. Создание простого проекта

```bash
# Создать папку
mkdir MyApp && cd MyApp

# Создать console app
dotnet new console

# Build
dotnet build

# Run
dotnet run

# Output:
# Hello, World!
```

### 1.5. Часто используемые commands

```bash
# Build (compile)
dotnet build
dotnet build -c Release          # Release configuration

# Run (build + execute)
dotnet run
dotnet run -- arg1 arg2          # передать args

# Test (запустить unit-тесты)
dotnet test

# Publish (production deployment)
dotnet publish -c Release -o ./publish

# Restore packages (обычно auto при build)
dotnet restore

# Clean build outputs
dotnet clean

# Format code
dotnet format
```

### 1.6. Working with packages (NuGet)

```bash
# Добавить package
dotnet add package Newtonsoft.Json
dotnet add package Microsoft.EntityFrameworkCore --version 8.0.0

# Удалить package
dotnet remove package Newtonsoft.Json

# Список packages в проекте
dotnet list package

# Update outdated packages
dotnet list package --outdated
dotnet add package Newtonsoft.Json   # та же команда — обновит до latest
```

> [!info]- Если ты использовал npm / pip / cargo
> Команды эквивалентны: `dotnet new` ≈ `npm init` / `cargo new`. `dotnet add package` ≈ `npm install` / `pip install` / `cargo add`. `dotnet build` ≈ `cargo build` / `mvn package`. `dotnet run` ≈ `npm start` / `cargo run`. NuGet — package registry для .NET (как npm для JS).

> [!question]- Интервью: какие dotnet CLI команды used daily?
> 1) **`dotnet new <template>`** — create project (console / webapi / classlib). 2) **`dotnet build`** — compile. 3) **`dotnet run`** — build + execute. 4) **`dotnet test`** — run tests. 5) **`dotnet add package <name>`** — install NuGet. 6) **`dotnet publish -c Release`** — production build. 7) **`dotnet restore`** — download packages (rare manual, обычно auto). 8) **`dotnet format`** — format code per `.editorconfig`. **Bonus**: `dotnet ef migrations add` для EF Core, `dotnet user-secrets` для secrets management.

---

## 2. Solution + Project structure

### 2.1. Что такое solution

```
Solution (.sln file) — контейнер для НЕСКОЛЬКИХ projects.
Project (.csproj file) — отдельная единица сборки (DLL/EXE).

Простой пример: Solution с одним Console project.
Сложный: Solution с API + Library + Tests + Domain (4-5 проектов).
```

### 2.2. Создание solution с одним project

```bash
mkdir MyApp && cd MyApp

# Создать solution
dotnet new sln -n MyApp

# Создать project
dotnet new webapi -n MyApp.Api

# Добавить project в solution
dotnet sln add MyApp.Api/MyApp.Api.csproj

# Структура:
# MyApp/
# ├── MyApp.sln
# └── MyApp.Api/
#     ├── MyApp.Api.csproj
#     ├── Program.cs
#     └── ...
```

### 2.3. Multi-project solution

```bash
mkdir BlogApp && cd BlogApp
dotnet new sln -n BlogApp

# Создать проекты
dotnet new webapi -n BlogApp.Api
dotnet new classlib -n BlogApp.Domain
dotnet new classlib -n BlogApp.Infrastructure
dotnet new xunit -n BlogApp.Tests

# Добавить все в solution
dotnet sln add BlogApp.Api/BlogApp.Api.csproj
dotnet sln add BlogApp.Domain/BlogApp.Domain.csproj
dotnet sln add BlogApp.Infrastructure/BlogApp.Infrastructure.csproj
dotnet sln add BlogApp.Tests/BlogApp.Tests.csproj

# Добавить project references (Api ссылается на Domain + Infrastructure)
dotnet add BlogApp.Api/BlogApp.Api.csproj reference BlogApp.Domain/BlogApp.Domain.csproj
dotnet add BlogApp.Api/BlogApp.Api.csproj reference BlogApp.Infrastructure/BlogApp.Infrastructure.csproj
dotnet add BlogApp.Tests/BlogApp.Tests.csproj reference BlogApp.Api/BlogApp.Api.csproj
```

### 2.4. Структура папок

```
BlogApp/
├── BlogApp.sln                          ← solution file
├── .gitignore
├── README.md
├── docker-compose.yml
│
├── BlogApp.Domain/                      ← business entities
│   ├── BlogApp.Domain.csproj
│   ├── Entities/
│   │   ├── User.cs
│   │   └── Post.cs
│   └── Interfaces/
│       └── IUserRepository.cs
│
├── BlogApp.Infrastructure/              ← data access, external
│   ├── BlogApp.Infrastructure.csproj
│   ├── Data/
│   │   └── AppDbContext.cs
│   └── Repositories/
│       └── UserRepository.cs
│
├── BlogApp.Api/                         ← presentation (HTTP)
│   ├── BlogApp.Api.csproj
│   ├── Program.cs
│   ├── Controllers/
│   ├── Dtos/
│   ├── appsettings.json
│   └── appsettings.Development.json
│
└── BlogApp.Tests/                       ← unit tests
    ├── BlogApp.Tests.csproj
    └── UserServiceTests.cs
```

### 2.5. Conventions имён projects

```
Project naming: <Solution>.<Layer/Module>
- BlogApp.Api          (web/API)
- BlogApp.Web          (MVC / Blazor)
- BlogApp.Domain       (business logic, entities)
- BlogApp.Application  (use cases, services)
- BlogApp.Infrastructure (data, external services)
- BlogApp.Tests        (общий tests)
- BlogApp.Tests.Unit
- BlogApp.Tests.Integration

Не делай:
- BlogAppApi (без точки)
- blogapp.api (lowercase)
- BlogApp.MyApiServer (verbose)
```

### 2.6. Полезные dotnet sln команды

```bash
# Список projects в solution
dotnet sln list

# Удалить project
dotnet sln remove BlogApp.Tests/BlogApp.Tests.csproj

# Добавить с расположением в "solution folder"
dotnet sln add BlogApp.Domain/BlogApp.Domain.csproj --solution-folder src

# Build всю solution
dotnet build BlogApp.sln

# Test всё
dotnet test BlogApp.sln
```

> [!question]- Интервью: зачем разбивать на несколько проектов?
> 1) **Separation of concerns** — Domain не зависит от Infrastructure (Clean Architecture). 2) **Reusability** — Domain library можно использовать в нескольких apps (Api + Worker + Console). 3) **Testability** — Tests project отдельный, не deployed в production. 4) **Build times** — изменение в Tests не пересобирает Api. 5) **Boundary enforcement** — нельзя случайно добавить SQL в Domain (если нет project reference). **Compromise**: для маленьких проектов 1 project хватает. 3-5 projects — sweet spot для серьёзных apps. 10+ projects — обычно over-engineering.

---

## 3. Git basics — для daily work

### 3.1. Setup

```bash
# Глобальная конфигурация (один раз)
git config --global user.name "Vitaly Carli"
git config --global user.email "vitaly@example.com"
git config --global init.defaultBranch main

# Verify
git config --list
```

### 3.2. Init repository

```bash
# В существующем проекте
cd MyApp
git init
git add .
git commit -m "Initial commit"

# Connect к remote (GitHub)
git remote add origin https://github.com/user/myapp.git
git push -u origin main
```

### 3.3. Daily workflow

```bash
# Posmotret status (changed files)
git status

# Добавить files в staging
git add file.cs                    # один file
git add .                           # все changed
git add Controllers/                # папка

# Commit
git commit -m "Add user authentication"

# Push в remote
git push

# Pull latest
git pull

# Если tracking не set
git push -u origin main             # set upstream
```

### 3.4. .gitignore для .NET

`dotnet new gitignore` создаст:

```
# Build outputs
bin/
obj/
*.dll
*.exe
*.pdb

# IDE
.vs/
.vscode/
.idea/

# User-specific
*.user
*.suo
*.userprefs

# Test results
TestResults/
coverage/

# NuGet
*.nupkg
.nuget/

# Logs
*.log
logs/

# OS
.DS_Store
Thumbs.db

# Local config
appsettings.Development.json   # ⚠️ если содержит secrets
*.env
.env.*
```

⚠️ **Вопрос**: коммитить `appsettings.Development.json`?

```
✅ Если содержит local dev config (connection to localhost) — yes
❌ Если содержит passwords / API keys — no
✅ Best practice: использовать User Secrets (см. 5.4)
```

### 3.5. Branching

```bash
# Создать branch для feature
git checkout -b feature/user-login

# Switch между branches
git checkout main
git checkout feature/user-login

# Список branches
git branch                          # local
git branch -a                       # local + remote

# Push branch в remote
git push -u origin feature/user-login

# Удалить branch (после merge)
git branch -d feature/user-login    # local
git push origin --delete feature/user-login   # remote
```

### 3.6. Merging

```bash
# Из main pull latest
git checkout main
git pull

# Switch на feature
git checkout feature/user-login

# Merge main в feature (rebase / merge)
git merge main
# Или
git rebase main                     # cleaner history (но опасно если pushed)

# Когда feature готова — merge в main
git checkout main
git merge feature/user-login
git push
```

### 3.7. Common workflow — Feature branches

```
1. git checkout main && git pull       # latest main
2. git checkout -b feature/x           # branch для work
3. ... код, тесты ...
4. git add . && git commit -m "..."    # save progress
5. git push -u origin feature/x        # push branch
6. PR на GitHub / GitLab               # code review
7. После approve — merge в main        # обычно через UI
8. git checkout main && git pull       # обновить локально
9. git branch -d feature/x             # cleanup
```

### 3.8. Полезные команды

```bash
# История commits
git log
git log --oneline                   # компактно
git log --graph --oneline --all     # граф branches

# Diff
git diff                            # working dir vs staged
git diff --staged                   # staged vs last commit
git diff main                       # vs main branch

# Undo
git restore file.cs                 # undo unstaged changes
git restore --staged file.cs        # unstage
git reset --soft HEAD~1             # undo last commit, keep changes
git reset --hard HEAD~1             # ⚠️ DESTRUCTIVE — undo + discard

# Stash (временно отложить)
git stash                           # save uncommitted
git stash pop                       # restore
git stash list                      # список stashes

# Tags (для releases)
git tag v1.0.0
git push origin v1.0.0
```

> [!question]- Интервью: чем отличается git rebase от git merge?
> **Merge** — создаёт merge commit, объединяющий два branches. History показывает branching. Сохраняет полную историю. **Rebase** — re-applies commits feature branch на top of main. Linear history без merge commits. **Когда merge**: shared branches, public history. **Когда rebase**: local feature branches до push. **Никогда rebase pushed branches** — переписывает history, сломает работу команды. **Best practice**: rebase локально (clean local history), merge в main через PR (preserves merge commits для feature tracking).

---

## 4. .editorconfig + dotnet format

### 4.1. Зачем

Автоматическое форматирование кода — все в команде используют одинаковые правила.

```bash
# Создать .editorconfig
dotnet new editorconfig

# Применить formatting
dotnet format
```

### 4.2. Базовый .editorconfig

```ini
root = true

[*]
indent_style = space
indent_size = 4
end_of_line = crlf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

[*.{json,yml,yaml}]
indent_size = 2

[*.cs]
# C# specific rules
csharp_prefer_braces = true:warning
csharp_style_var_when_type_is_apparent = true:suggestion
dotnet_sort_system_directives_first = true

# Naming conventions
dotnet_naming_rule.private_fields.symbols = private_fields
dotnet_naming_rule.private_fields.style = camel_case_underscore_prefix
dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private
dotnet_naming_style.camel_case_underscore_prefix.required_prefix = _
dotnet_naming_style.camel_case_underscore_prefix.capitalization = camel_case
```

### 4.3. dotnet format в CI

```yaml
# GitHub Actions
- name: Verify formatting
  run: dotnet format --verify-no-changes
```

`--verify-no-changes` exit code != 0 если нужны fixes — fails CI build.

---

## 5. Configuration — appsettings.json

### 5.1. Default ASP.NET Core конфигурация

```json
// appsettings.json — base config
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionStrings": {
    "Default": "Server=localhost;Database=myapp;..."
  },
  "ApiSettings": {
    "BaseUrl": "https://api.example.com",
    "TimeoutSeconds": 30
  }
}

// appsettings.Development.json — overrides для Development
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug"
    }
  }
}

// appsettings.Production.json — overrides для Production
{
  "AllowedHosts": "myapp.com"
}
```

### 5.2. Чтение в коде

```csharp
// Strongly-typed (preferred)
public class ApiSettings
{
    public string BaseUrl { get; set; } = "";
    public int TimeoutSeconds { get; set; }
}

// Program.cs
builder.Services.Configure<ApiSettings>(
    builder.Configuration.GetSection("ApiSettings"));

// Use в Service
public class MyService
{
    private readonly ApiSettings _settings;
    public MyService(IOptions<ApiSettings> options) => _settings = options.Value;
}
```

```csharp
// Direct read (less preferred — string keys)
var url = builder.Configuration["ApiSettings:BaseUrl"];
var timeout = builder.Configuration.GetValue<int>("ApiSettings:TimeoutSeconds");
var connStr = builder.Configuration.GetConnectionString("Default");
```

### 5.3. Environment variables

```bash
# Override любой config через env var
# Используй __ (double underscore) для nested
export ApiSettings__BaseUrl=https://staging.api.com
export ConnectionStrings__Default="Server=prod-db;..."

dotnet run
# .NET читает env vars и они override appsettings.json
```

### 5.4. User Secrets — для local secrets

⚠️ **Никогда не коммить passwords / API keys в appsettings.json!**

```bash
# Init user secrets для project
dotnet user-secrets init

# Set secret
dotnet user-secrets set "ConnectionStrings:Default" "Server=...;Password=secret"
dotnet user-secrets set "ApiKeys:OpenAI" "sk-..."

# List
dotnet user-secrets list

# Remove
dotnet user-secrets remove "ApiKeys:OpenAI"
```

Secrets хранятся **вне** проекта (`%APPDATA%\Microsoft\UserSecrets\<id>\secrets.json` на Windows). НЕ попадают в git.

```csharp
// Использование — то же что для appsettings
var apiKey = builder.Configuration["ApiKeys:OpenAI"];
```

### 5.5. Production secrets

```
Local dev:    User Secrets
Production:   - Azure Key Vault
              - AWS Secrets Manager
              - HashiCorp Vault
              - Kubernetes Secrets
              - Environment variables (если runtime managed)

Никогда:
- Plain text в appsettings.json
- Hardcoded в коде
- В git history (даже после удаления — git remembers!)
```

### 5.6. ASPNETCORE_ENVIRONMENT

Определяет какой `appsettings.{Env}.json` загружается.

```bash
# Local dev
export ASPNETCORE_ENVIRONMENT=Development

# Staging
export ASPNETCORE_ENVIRONMENT=Staging

# Production (default если не set)
export ASPNETCORE_ENVIRONMENT=Production
```

```csharp
// Чтение в коде
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

> [!question]- Интервью: где хранить secrets в .NET проекте?
> **Local dev**: User Secrets (`dotnet user-secrets set`) — хранится вне проекта, не в git. **Staging/Production**: cloud-specific (Azure Key Vault / AWS Secrets Manager / HashiCorp Vault) или environment variables через runtime (Docker / Kubernetes Secrets). **Никогда**: plain text в appsettings.json (commit'нется). **Workflow**: 1) Local — User Secrets. 2) CI/CD — env vars из secrets manager. 3) Production — Key Vault references в config. **Bonus**: `IConfiguration` API одинаков — кода не отличается между env'ами, только source меняется.

---

## 6. Common pitfalls

### 6.1. Bin/obj в git

```bash
# ❌ Закоммитил bin/ и obj/ — 100s MB мусора
git add bin/
git commit
```

**Фикс**: `dotnet new gitignore` в репо ROOT, потом:

```bash
git rm -r --cached bin obj
git commit -m "Remove build outputs from git"
```

### 6.2. Secrets в git

```json
// appsettings.json
{
  "ConnectionStrings": {
    "Default": "Server=...;Password=MyRealPassword123"
  }
}
```

**Фикс**: User Secrets для local, env vars + Key Vault для prod.

⚠️ Если secret уже в git history — **сменить secret** (history не очистить просто).

### 6.3. Project references в circular

```bash
# ❌ Domain ссылается на Infrastructure → Infrastructure на Domain
dotnet add Domain/Domain.csproj reference Infrastructure/Infrastructure.csproj
dotnet add Infrastructure/Infrastructure.csproj reference Domain/Domain.csproj
# Build error: circular dependency
```

**Фикс**: Domain не должен ссылаться на Infrastructure (Clean Architecture). Define interfaces в Domain, implementations в Infrastructure.

### 6.4. Версии packages не sync

```xml
<!-- Project A -->
<PackageReference Include="Newtonsoft.Json" Version="13.0.0" />

<!-- Project B -->
<PackageReference Include="Newtonsoft.Json" Version="12.0.3" />
```

Конфликты, runtime errors. **Фикс**: Central Package Management (`Directory.Packages.props` в root):

```xml
<!-- Directory.Packages.props -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Newtonsoft.Json" Version="13.0.0" />
  </ItemGroup>
</Project>

<!-- В каждом *.csproj — без version -->
<PackageReference Include="Newtonsoft.Json" />
```

### 6.5. .gitignore забыл

Поздно вспомнил, уже закоммитил bin/. После `.gitignore` добавил:

```bash
git rm -r --cached .         # remove all from git index
git add .                     # re-add (теперь .gitignore работает)
git commit -m "Apply .gitignore"
```

### 6.6. Push --force на main

```bash
git push --force origin main
# ❌ Перезаписал main — все changes коллег потеряны
```

**Фикс**: никогда `--force` на shared branches. Только локальные.

### 6.7. Hardcoded paths

```csharp
// ❌ Hardcoded абсолютный путь — работает только на твоей машине
var path = "C:\\Users\\Vitaly\\projects\\data\\file.csv";
```

**Фикс**: relative paths + Path.Combine:

```csharp
var path = Path.Combine(Directory.GetCurrentDirectory(), "data", "file.csv");
// Or use IConfiguration
```

### 6.8. Не использовать .editorconfig

Каждый разработчик форматирует по-своему — git diffs полны formatting changes.

**Фикс**: `dotnet new editorconfig` + `dotnet format` в pre-commit hook или CI.

### 6.9. Single project для всего

```
MyApp/
├── MyApp.csproj
├── Controllers/...
├── Services/...
├── Repositories/...
├── DbContext.cs
└── ... (1000+ files)
```

Когда вырастает — split на multi-project. Никаких boundaries → coupling растёт.

### 6.10. Wrong .NET version

```xml
<TargetFramework>netcoreapp3.1</TargetFramework>   <!-- old, EOL -->
<TargetFramework>net5.0</TargetFramework>          <!-- old, EOL -->
<TargetFramework>net6.0</TargetFramework>          <!-- LTS до Nov 2024 -->
<TargetFramework>net8.0</TargetFramework>          <!-- LTS, current -->
<TargetFramework>net9.0</TargetFramework>          <!-- STS -->
```

LTS (Long-term support) — 3 года поддержки. STS (Short-term) — 18 месяцев. Для production выбирай LTS (.NET 8 в 2024-2026).

> [!question]- Интервью: топ-3 ошибки в project setup?
> 1) **Secrets в git** — passwords / API keys в appsettings.json. Fix: User Secrets (local) + Key Vault (prod). Если уже committed — сменить secret. 2) **bin/obj в git** — 100MB мусора. Fix: `dotnet new gitignore` + `git rm -r --cached`. 3) **Circular project references** — Domain ↔ Infrastructure. Fix: Clean Architecture — Domain независим, Interfaces в Domain, implementations в Infrastructure. **Bonus**: `git push --force` на shared branches — никогда. **Bonus 2**: hardcoded absolute paths — `Path.Combine` + relative paths.

---

## 7. Cheat sheet

```bash
# === dotnet CLI ===
dotnet new sln -n MyApp                       # solution
dotnet new webapi -n MyApp.Api                # ASP.NET Core API
dotnet new classlib -n MyApp.Domain           # library
dotnet new xunit -n MyApp.Tests               # tests

dotnet sln add MyApp.Api/MyApp.Api.csproj
dotnet add MyApp.Api/MyApp.Api.csproj reference MyApp.Domain/MyApp.Domain.csproj

dotnet add package Newtonsoft.Json
dotnet remove package Newtonsoft.Json
dotnet list package --outdated

dotnet build
dotnet run
dotnet test
dotnet publish -c Release -o ./publish
dotnet format

# === Git daily ===
git status
git add .
git commit -m "message"
git push

git checkout main
git pull
git checkout -b feature/x
git merge main
git push -u origin feature/x

git log --oneline
git diff
git restore file.cs
git stash / git stash pop

# === User Secrets ===
dotnet user-secrets init
dotnet user-secrets set "ApiKey" "secret123"
dotnet user-secrets list
dotnet user-secrets remove "ApiKey"

# === Configuration ===
# appsettings.json — base
# appsettings.Development.json — dev override
# appsettings.Production.json — prod override
# Environment variable: ApiSettings__BaseUrl=...
# User Secrets: для local secrets
# Production: Azure Key Vault / AWS Secrets Manager
```

```xml
<!-- Common .csproj structure -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UserSecretsId>my-secrets-id</UserSecretsId>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
  </ItemGroup>
  
  <ItemGroup>
    <ProjectReference Include="..\MyApp.Domain\MyApp.Domain.csproj" />
  </ItemGroup>
</Project>
```

---

## 8. Practice exercises

### 8.1. Создай blog API solution с нуля

```bash
mkdir BlogApp && cd BlogApp
dotnet new sln -n BlogApp
dotnet new webapi -n BlogApp.Api
dotnet new classlib -n BlogApp.Domain
dotnet new classlib -n BlogApp.Infrastructure
dotnet new xunit -n BlogApp.Tests

# Add to solution
# Add references
# Add packages (EF Core, xUnit)

# Initial git
dotnet new gitignore
dotnet new editorconfig
git init
git add .
git commit -m "Initial setup"
```

Реализуй:
- Domain имеет Post, Author entities + IPostRepository interface
- Infrastructure реализует IPostRepository через EF Core
- Api имеет PostsController с CRUD endpoints
- Tests — unit-тест для PostService

### 8.2. Setup secrets management

В существующем API проекте:
1. Init user secrets (`dotnet user-secrets init`)
2. Move connection string из appsettings.json в user secrets
3. Verify работает локально
4. Create README "How to setup secrets" для нового разработчика

### 8.3. Migration от single project к multi-project

Существующий MyApp.csproj с Controllers, Services, Data. Разнеси:
1. Создай MyApp.Domain (entities + interfaces)
2. Создай MyApp.Infrastructure (DbContext, repositories)
3. MyApp остаётся как Api (controllers только)
4. Все references правильно настроены
5. Build и tests passing

---

## 9. Что читать дальше

1. **`Infrastructure/Senior/project-setup.md`** — production-grade setups
2. **`Architecture/Junior/architecture-basics.md`** — layered architecture
3. **`Infrastructure/Junior/docker-for-dev.md`** — containerize project
4. **`Infrastructure/Middle/cicd-github-actions.md`** — CI/CD pipelines
5. **`Architecture/Senior/architecture-decisions.md`** — Clean Architecture deep

---

## 10. См. также

- [[project-setup|Infrastructure/Senior/project-setup]] — deep setup
- [[docker-for-dev|Infrastructure/Junior/docker-for-dev]] — Docker basics
- [[architecture-basics|Architecture/Junior/architecture-basics]] — layers
- [[ef-basics|EFCore/Junior/ef-basics]] — EF Core setup
- [[cicd-github-actions|Infrastructure/Middle/cicd-github-actions]] — CI/CD

---

## 11. Reading list

- **Microsoft Docs — dotnet CLI** — learn.microsoft.com/dotnet/core/tools/
- **Microsoft Docs — Configuration** — learn.microsoft.com/aspnet/core/fundamentals/configuration/
- **Microsoft Docs — User Secrets** — learn.microsoft.com/aspnet/core/security/app-secrets
- **Pro Git book (free)** — git-scm.com/book/en/v2
- **Atlassian Git tutorials** — atlassian.com/git/tutorials
- **NuGet docs** — learn.microsoft.com/nuget/
- **EditorConfig** — editorconfig.org
- **.NET versioning** — dotnet.microsoft.com/platform/support/policy
