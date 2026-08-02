---
tags: [project-setup, ci-cd, github-actions, dotnet, secrets, analyzers, cpm]
level: Senior
date: 2026-04-30
---

# Setup нового .NET проекта в 2026

> Полный гайд по starting up .NET проекта с production-grade defaults. Закрывает: Directory.Build.props, .editorconfig, Central Package Management, analyzers (SonarAnalyzer/Meziantou/Roslynator), .NET Aspire, secrets management (user-secrets, Vault, Infisical), GitHub Actions CI/CD, .gitignore, repository structure, environments.

---

## Что это, зачем и когда

### Что это?
Чеклист настроек, которые нужно сделать **в самом начале** проекта. Если не настроить сразу — потом будет больно: предупреждения копятся, стиль кода разный у всех, зависимости конфликтуют, секреты в коде.

**Аналогия:** Фундамент дома. Закладываешь ОДИН раз в начале. Если криво — потом весь дом перекосит.

### Зачем?

| Без правильного setup | С правильным setup |
|----------------------|-----|
| Каждый разработчик пишет в своём стиле → unreadable PR | EditorConfig + analyzers — единый стиль |
| Конфликты версий пакетов между проектами | Central Package Management — одна версия |
| 500 warning'ов в build output, не виден важный | TreatWarningsAsErrors — нет warning'ов в repo |
| Секреты в коде → утечка в git | user-secrets, Vault, Infisical |
| Кто-то закоммитил bin/obj | .gitignore настроен правильно |
| "У меня работает" → у других нет | CI прогоняет каждый коммит |
| Production crash потому что забыли .ConfigureAwait(false) в библиотеке | Roslynator + Meziantou ловят |
| Lint-only PRs занимают часы DBA | Pre-commit hooks + auto-format |

### Когда применять?
- **Каждый новый проект** — с ПЕРВОГО дня
- **Существующий проект** — при первой возможности (чем раньше, тем дешевле)

---

## 1. Repository Structure

```
my-app/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml          # Build + test
│   │   ├── deploy.yml      # Deploy to environments
│   │   └── codeql.yml      # Security scanning
│   ├── dependabot.yml      # Auto-update deps
│   └── CODEOWNERS          # PR review routing
├── docs/
│   ├── adr/                # Architecture Decision Records
│   └── README.md
├── src/
│   ├── MyApp.AppHost/      # Aspire orchestrator
│   ├── MyApp.ServiceDefaults/
│   ├── MyApp.Api/
│   ├── MyApp.Application/
│   ├── MyApp.Domain/
│   ├── MyApp.Infrastructure/
│   └── MyApp.Worker/
├── tests/
│   ├── MyApp.UnitTests/
│   ├── MyApp.IntegrationTests/
│   └── MyApp.ArchitectureTests/
├── deploy/
│   ├── helm/               # K8s charts
│   └── docker-compose.yml
├── .editorconfig
├── .gitignore
├── .gitattributes
├── Directory.Build.props
├── Directory.Build.targets
├── Directory.Packages.props
├── global.json             # SDK pinning
├── nuget.config
├── MyApp.sln
└── README.md
```

---

## 2. global.json — pinning SDK

```json
{
    "sdk": {
        "version": "10.0.100",
        "rollForward": "latestFeature",
        "allowPrerelease": false
    }
}
```

| `rollForward` | Поведение |
|---------------|-----------|
| `disable` | Только указанная версия |
| `patch` | Любой patch (10.0.100 → 10.0.999) |
| `feature` | Любой feature (10.0.x) |
| `latestFeature` | Латест feature в major.minor |
| `latestMinor` | Любой minor в major |
| `latestMajor` | Любой выше |

> [!info] Зачем pinning SDK
> Гарантирует что все разработчики и CI используют одну версию. Без global.json — `dotnet build` использует latest installed → reproducibility проблемы.

---

## 3. Directory.Build.props — единые настройки

Файл рядом с `.sln`. Применяется ко **всем** `.csproj` в дереве (если не override).

```xml
<Project>
  <PropertyGroup>
    <!-- Target framework -->
    <TargetFramework>net10.0</TargetFramework>
    <LangVersion>latest</LangVersion>
    
    <!-- Modern C# features -->
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <WarningsNotAsErrors>NU1900;NU1901;NU1902;NU1903;NU1904</WarningsNotAsErrors>
    <NoWarn>CS1591</NoWarn> <!-- Missing XML doc — для internal -->
    
    <!-- Code analysis -->
    <AnalysisLevel>latest</AnalysisLevel>
    <AnalysisMode>All</AnalysisMode>
    <CodeAnalysisTreatWarningsAsErrors>true</CodeAnalysisTreatWarningsAsErrors>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    
    <!-- NuGet -->
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <CentralPackageTransitivePinningEnabled>true</CentralPackageTransitivePinningEnabled>
    
    <!-- Repro builds -->
    <Deterministic>true</Deterministic>
    <ContinuousIntegrationBuild Condition="'$(GITHUB_ACTIONS)' == 'true'">true</ContinuousIntegrationBuild>
    
    <!-- Source link для debugging deployed builds -->
    <PublishRepositoryUrl>true</PublishRepositoryUrl>
    <EmbedUntrackedSources>true</EmbedUntrackedSources>
    <IncludeSymbols>true</IncludeSymbols>
    <SymbolPackageFormat>snupkg</SymbolPackageFormat>
    
    <!-- Common assembly info -->
    <Authors>Your Name</Authors>
    <Company>Your Company</Company>
    <Copyright>Copyright © $([System.DateTime]::Now.Year)</Copyright>
  </PropertyGroup>
  
  <!-- Analyzers — applied to ALL projects -->
  <ItemGroup>
    <PackageReference Include="SonarAnalyzer.CSharp" PrivateAssets="all" />
    <PackageReference Include="Meziantou.Analyzer" PrivateAssets="all" />
    <PackageReference Include="Roslynator.Analyzers" PrivateAssets="all" />
    <PackageReference Include="Roslynator.Formatting.Analyzers" PrivateAssets="all" />
  </ItemGroup>
  
  <!-- Source Link -->
  <ItemGroup>
    <PackageReference Include="Microsoft.SourceLink.GitHub" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

### Test projects override

`tests/Directory.Build.props`:

```xml
<Project>
  <Import Project="$([MSBuild]::GetPathOfFileAbove('Directory.Build.props', '$(MSBuildThisFileDirectory)../'))" />
  
  <PropertyGroup>
    <!-- В тестах — нет XML doc -->
    <NoWarn>$(NoWarn);CS1591;CS8625</NoWarn>
    <IsPackable>false</IsPackable>
  </PropertyGroup>
  
  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" />
    <PackageReference Include="xunit" />
    <PackageReference Include="xunit.runner.visualstudio" PrivateAssets="all" />
    <PackageReference Include="coverlet.collector" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

---

## 4. Directory.Packages.props — Central Package Management (CPM)

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  
  <ItemGroup Label="Microsoft">
    <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="10.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore" Version="10.0.0" />
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Hosting" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Http.Resilience" Version="10.0.0" />
  </ItemGroup>
  
  <ItemGroup Label="Database">
    <PackageVersion Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="10.0.0" />
    <PackageVersion Include="EFCore.BulkExtensions" Version="10.0.0" />
  </ItemGroup>
  
  <ItemGroup Label="Validation, Mediator, Mapping">
    <PackageVersion Include="FluentValidation" Version="11.10.0" />
    <PackageVersion Include="MediatR" Version="13.0.0" />
    <PackageVersion Include="Mapperly" Version="4.0.0" />
  </ItemGroup>
  
  <ItemGroup Label="Observability">
    <PackageVersion Include="OpenTelemetry.Extensions.Hosting" Version="1.10.0" />
    <PackageVersion Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.10.0" />
    <PackageVersion Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.10.0" />
    <PackageVersion Include="Serilog.AspNetCore" Version="9.0.0" />
  </ItemGroup>
  
  <ItemGroup Label="Testing">
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.13.0" />
    <PackageVersion Include="xunit" Version="2.10.0" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="3.1.0" />
    <PackageVersion Include="Shouldly" Version="4.4.0" />
    <PackageVersion Include="Testcontainers.PostgreSql" Version="4.5.0" />
    <PackageVersion Include="NSubstitute" Version="5.5.0" />
    <PackageVersion Include="Bogus" Version="36.0.0" />
  </ItemGroup>
  
  <ItemGroup Label="Analyzers">
    <PackageVersion Include="SonarAnalyzer.CSharp" Version="11.0.0" />
    <PackageVersion Include="Meziantou.Analyzer" Version="2.0.200" />
    <PackageVersion Include="Roslynator.Analyzers" Version="4.13.0" />
    <PackageVersion Include="Roslynator.Formatting.Analyzers" Version="4.13.0" />
    <PackageVersion Include="Microsoft.SourceLink.GitHub" Version="9.0.0" />
  </ItemGroup>
</Project>
```

В `.csproj` — без версий:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore" />
  <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" />
</ItemGroup>
```

> [!info] CPM — почему важно
> Без CPM — каждый `.csproj` указывает свою версию. Через год — 5 проектов используют `EntityFrameworkCore 8.0.0`, `8.0.5`, `9.0.0` → конфликты транзитивных зависимостей, runtime errors.

---

## 5. .editorconfig — стиль кода

```ini
# .editorconfig in repo root
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 4
trim_trailing_whitespace = true

[*.{cs,csx}]
indent_size = 4

# Naming conventions
dotnet_diagnostic.IDE1006.severity = error  # Naming rule violation

# Code style
csharp_using_directive_placement = outside_namespace:warning
csharp_style_namespace_declarations = file_scoped:warning
csharp_style_prefer_method_group_conversion = true:suggestion
csharp_style_prefer_top_level_statements = true:suggestion
csharp_style_prefer_primary_constructors = true:suggestion

# Nullable
dotnet_diagnostic.CS8600.severity = error  # Converting null literal
dotnet_diagnostic.CS8601.severity = error  # Possible null reference assignment
dotnet_diagnostic.CS8602.severity = error  # Dereference of a possibly null reference
dotnet_diagnostic.CS8603.severity = error  # Possible null reference return

# Async
dotnet_diagnostic.CA2007.severity = none   # ConfigureAwait — для apps не нужно
dotnet_diagnostic.MA0040.severity = warning  # Pass CancellationToken
dotnet_diagnostic.MA0042.severity = error    # Don't use blocking calls in async
dotnet_diagnostic.MA0045.severity = error    # Sync over async

# Code quality
dotnet_diagnostic.CA1062.severity = error  # Validate arguments of public methods (для library)
dotnet_diagnostic.CA1707.severity = none   # Identifiers should not contain underscores

# Analyzer specific
dotnet_diagnostic.S1144.severity = warning  # Sonar: Unused private types
dotnet_diagnostic.RCS1090.severity = none   # Roslynator: Add ConfigureAwait(false) — for apps

[*.{xml,csproj,props,targets,config}]
indent_size = 2

[*.{json,yml,yaml,md}]
indent_size = 2

[*.sln]
indent_style = tab
```

### Naming Conventions

```ini
# Private fields — _camelCase
dotnet_naming_rule.private_fields_should_be_camel_case_with_underscore.symbols = private_fields
dotnet_naming_rule.private_fields_should_be_camel_case_with_underscore.style = camel_case_with_underscore_style
dotnet_naming_rule.private_fields_should_be_camel_case_with_underscore.severity = error

dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private

dotnet_naming_style.camel_case_with_underscore_style.required_prefix = _
dotnet_naming_style.camel_case_with_underscore_style.capitalization = camel_case

# Async methods — should end with Async
dotnet_naming_rule.async_methods_should_end_with_async.symbols = async_methods
dotnet_naming_rule.async_methods_should_end_with_async.style = async_method_style
dotnet_naming_rule.async_methods_should_end_with_async.severity = warning

dotnet_naming_symbols.async_methods.applicable_kinds = method
dotnet_naming_symbols.async_methods.required_modifiers = async

dotnet_naming_style.async_method_style.required_suffix = Async
dotnet_naming_style.async_method_style.capitalization = pascal_case
```

---

## 6. .gitignore

Используй официальный шаблон от GitHub: github.com/github/gitignore/blob/main/VisualStudio.gitignore

Минимум:

```gitignore
# .NET
bin/
obj/
*.user
*.suo

# Build artifacts
publish/
out/
[Tt]est[Rr]esult*/
[Bb]uild[Ll]og.*

# Rider
.idea/

# VS Code
.vscode/
!.vscode/launch.json
!.vscode/tasks.json
!.vscode/settings.json

# JetBrains
*.user
*.userosscache
*.sln.docstates

# Logs
*.log
logs/

# Environment files
.env
.env.local
appsettings.Development.json
appsettings.Local.json

# User secrets
*.user
secrets.json

# Coverage
coverage/
*.opencover.xml
TestResults/

# OS
.DS_Store
Thumbs.db

# NuGet
*.nupkg
*.snupkg
.nuget/
packages/

# Aspire
.aspire/
```

### .gitattributes — line endings

```gitattributes
* text=auto eol=lf

*.cs text eol=lf
*.csproj text eol=lf
*.json text eol=lf
*.md text eol=lf

# Binary files
*.png binary
*.jpg binary
*.gif binary
*.dll binary
*.exe binary

# Diff settings
*.cs diff=csharp
```

---

## 7. Static Code Analysis — Analyzers

### SonarAnalyzer.CSharp

Quality + security + bugs. От SonarSource (создатели SonarQube).

Хорошо ловит:
- `S1144` — Unused private classes / methods
- `S2259` — Null pointers
- `S125` — Commented-out code
- `S2629` — Don't use string interpolation in logger calls (perf)

### Meziantou.Analyzer

Async/await + LINQ + perf. От Gérald Barré (Microsoft MVP).

Хорошо ловит:
- `MA0040` — Forward CancellationToken to methods accepting one
- `MA0042` — Do not use blocking calls in async method (`Task.Wait()`, `.Result`)
- `MA0045` — Don't use sync over async
- `MA0048` — File name should match type name
- `MA0004` — Use `Task.ConfigureAwait(false)` (для library, для apps можно отключить)

### Roslynator.Analyzers + Formatting

Идиоматичный C#, refactoring suggestions, formatting.

Хорошо для:
- Code style consistency
- Modern C# patterns (nullable, expression-bodied members, switch expressions)
- Auto-formatting

### CodeQL (security)

GitHub-native security scanning. Через GitHub Actions:

```yaml
# .github/workflows/codeql.yml
name: "CodeQL"
on:
  push: {branches: [main]}
  pull_request: {branches: [main]}
  schedule: [{cron: "0 6 * * 1"}]  # Раз в неделю

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with: {languages: csharp}
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```

### Severity tuning

В `.editorconfig` можно понизить шум:

```ini
# Слишком строгие — отключить
dotnet_diagnostic.CA1062.severity = none  # Validate arguments — для apps не нужно
dotnet_diagnostic.CA1303.severity = none  # Don't use literal strings — мешает локализации

# Критичные — error
dotnet_diagnostic.CS8602.severity = error  # Null dereference
dotnet_diagnostic.CA2007.severity = error  # Missing ConfigureAwait (для library)

# Стилистика — warning
dotnet_diagnostic.IDE0090.severity = warning  # 'new()' simplification
```

> [!info] Постепенное внедрение в legacy
> Сразу включить TreatWarningsAsErrors = true в legacy → 5000 warnings → невозможно сбилдить. Решения:
> 1. По проекту — `<TreatWarningsAsErrors>false</TreatWarningsAsErrors>` локально, true в CI
> 2. По правилу — `dotnet_diagnostic.X.severity = warning` (snake by snake включить error)
> 3. `<Nullable>warnings</Nullable>` сначала, потом `enable`

---

## 8. Secrets Management

### Никогда не в git

```csharp
// ❌ В appsettings.json:
{
  "ConnectionStrings": {
    "Default": "Host=...;Password=Secret123!"  // GIT history forever!
  }
}
```

### Local Development — User Secrets

```bash
cd src/MyApp.Api
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Default" "Host=localhost;Database=app;Password=dev"
dotnet user-secrets set "Auth:JwtKey" "your-256-bit-secret"
```

Хранятся в `~/.microsoft/usersecrets/<UserSecretsId>/secrets.json` — вне репо.

```csharp
// Program.cs
builder.Configuration.AddUserSecrets<Program>(optional: true);
```

В `.csproj`:

```xml
<UserSecretsId>my-app-12345-67890</UserSecretsId>
```

### Environment Variables (Docker / k8s)

```bash
docker run -e "ConnectionStrings__Default=Host=db;Password=Secret" myapp
```

В .NET — `__` (double underscore) заменяет `:` для nested keys.

### Production — Cloud Secrets Manager

#### Azure Key Vault

```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{vaultName}.vault.azure.net/"),
    new DefaultAzureCredential());
```

#### AWS Secrets Manager

```csharp
builder.Configuration.AddSecretsManager(
    region: RegionEndpoint.EUWest1,
    configurator: opts =>
    {
        opts.SecretFilter = entry => entry.Name.StartsWith("myapp/");
        opts.KeyGenerator = (entry, key) => key.Replace("myapp/", "").Replace("/", ":");
    });
```

#### HashiCorp Vault

```csharp
// VaultSharp NuGet
var vaultClient = new VaultClient(new VaultClientSettings(
    "https://vault.example.com:8200",
    new TokenAuthMethodInfo(token)));

var secret = await vaultClient.V1.Secrets.KeyValue.V2
    .ReadSecretAsync("myapp/db", mountPoint: "secret");

builder.Configuration["ConnectionStrings:Default"] = secret.Data.Data["password"].ToString();
```

#### Infisical (open-source alternative)

```csharp
// Infisical.Net NuGet
var client = new InfisicalClient(new ClientSettings
{
    ClientId = "...",
    ClientSecret = "...",
    SiteUrl = "https://app.infisical.com"
});

var secrets = await client.ListSecretsAsync(new ListSecretsOptions
{
    Environment = "production",
    ProjectId = "..."
});

foreach (var secret in secrets)
    builder.Configuration[secret.SecretKey] = secret.SecretValue;
```

### Kubernetes — Secrets + External Secrets Operator

```yaml
# kubernetes/secret.yaml — НЕ в git!
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  ConnectionStrings__Default: "Host=...;Password=..."
```

Лучше — **External Secrets Operator** (синхронизирует Vault/Infisical/Azure → k8s Secret):

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-secrets
spec:
  refreshInterval: 5m
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: app-secrets
  data:
    - secretKey: ConnectionStrings__Default
      remoteRef:
        key: myapp/db
        property: connection_string
```

См. [[kubernetes|Kubernetes — Secrets]].

---

## 9. .NET Aspire (.NET 9+)

Cloud-native orchestration для local dev и production deploy.

### Структура

```
src/
├── MyApp.AppHost/          # Оркестратор (точка входа для dev)
├── MyApp.ServiceDefaults/  # Общая конфигурация для всех сервисов
├── MyApp.Api/              # Web API
└── MyApp.Worker/           # Background Worker
```

### AppHost

```csharp
// MyApp.AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

// Postgres
var postgres = builder.AddPostgres("postgres")
    .WithDataVolume()              // Persist data
    .WithPgAdmin();                // UI

var db = postgres.AddDatabase("appdb");

// Redis
var redis = builder.AddRedis("cache")
    .WithRedisCommander();

// RabbitMQ
var rabbitmq = builder.AddRabbitMQ("rabbitmq")
    .WithManagementPlugin();

// Services
var api = builder.AddProject<Projects.MyApp_Api>("api")
    .WithReference(db)             // Auto-inject connection string!
    .WithReference(redis)
    .WithReference(rabbitmq)
    .WithExternalHttpEndpoints();

builder.AddProject<Projects.MyApp_Worker>("worker")
    .WithReference(db)
    .WithReference(rabbitmq);

builder.Build().Run();
```

### ServiceDefaults

```csharp
// MyApp.ServiceDefaults/Extensions.cs
public static class Extensions
{
    public static IHostApplicationBuilder AddServiceDefaults(this IHostApplicationBuilder builder)
    {
        builder.ConfigureOpenTelemetry();
        builder.AddDefaultHealthChecks();
        
        builder.Services.ConfigureHttpClientDefaults(http =>
        {
            http.AddStandardResilienceHandler();  // Retry + CB + Timeout
        });
        
        builder.Services.AddServiceDiscovery();
        return builder;
    }
    
    public static IHostApplicationBuilder ConfigureOpenTelemetry(this IHostApplicationBuilder builder)
    {
        builder.Logging.AddOpenTelemetry(logging =>
        {
            logging.IncludeFormattedMessage = true;
            logging.IncludeScopes = true;
        });
        
        builder.Services.AddOpenTelemetry()
            .WithMetrics(metrics =>
            {
                metrics.AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation()
                       .AddRuntimeInstrumentation();
            })
            .WithTracing(tracing =>
            {
                tracing.AddSource(builder.Environment.ApplicationName)
                       .AddAspNetCoreInstrumentation()
                       .AddHttpClientInstrumentation();
            });
        
        var otlpEndpoint = builder.Configuration["OTEL_EXPORTER_OTLP_ENDPOINT"];
        if (!string.IsNullOrWhiteSpace(otlpEndpoint))
        {
            builder.Services.AddOpenTelemetry().UseOtlpExporter();
        }
        
        return builder;
    }
}
```

### В каждом сервисе

```csharp
// MyApp.Api/Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults();
builder.AddNpgsqlDbContext<AppDbContext>("appdb");  // Уже знает connection string!
builder.AddRedisDistributedCache("cache");

// ... rest of configuration
```

### Aspire Dashboard

При запуске AppHost автоматически:
- **Traces** — distributed tracing всех сервисов
- **Metrics** — CPU, memory, HTTP requests
- **Logs** — structured logs всех сервисов
- **Resources** — статус каждого container/project

http://localhost:17221

### Publish

```bash
# Aspire 9+ — генерация k8s/docker-compose
aspire publish --output-path ./deploy
```

Альтернативы Aspire:
- Docker Compose — простой, без авто-конфигурации
- Tilt — для dev в k8s
- Skaffold — для dev в k8s

---

## 10. GitHub Actions — CI/CD

### Базовый CI

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push: {branches: [main]}
  pull_request: {branches: [main]}

env:
  DOTNET_VERSION: '10.0.x'
  DOTNET_NOLOGO: true
  DOTNET_CLI_TELEMETRY_OPTOUT: true

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
        with: {fetch-depth: 0}  # для GitVersion
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with: {dotnet-version: ${{ env.DOTNET_VERSION }}}
      
      - name: Cache NuGet
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/Directory.Packages.props') }}
          restore-keys: |
            ${{ runner.os }}-nuget-
      
      - name: Restore
        run: dotnet restore
      
      - name: Build
        run: dotnet build --configuration Release --no-restore
      
      - name: Test
        run: |
          dotnet test --configuration Release --no-build \
            --logger "trx;LogFileName=test-results.trx" \
            --collect "XPlat Code Coverage" \
            --results-directory ./TestResults
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: ./TestResults
      
      - name: Code coverage report
        uses: codecov/codecov-action@v4
        with:
          files: ./TestResults/**/coverage.cobertura.xml
          token: ${{ secrets.CODECOV_TOKEN }}
```

### Docker build + push

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build-image:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      id-token: write  # для cosign
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=sha,format=short
      
      - name: Build and push
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./src/MyApp.Api/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64
      
      - name: Sign image with cosign
        uses: sigstore/cosign-installer@v3
      - run: |
          cosign sign --yes ghcr.io/${{ github.repository }}@${{ steps.meta.outputs.digest }}
```

### Migrations job

```yaml
  migrate:
    needs: build-image
    runs-on: ubuntu-latest
    environment: production  # требует approval!
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: {dotnet-version: '10.0.x'}
      
      - name: Generate migration script
        run: |
          dotnet tool install --global dotnet-ef
          dotnet ef migrations script --idempotent \
            --project src/MyApp.Infrastructure \
            --startup-project src/MyApp.Api \
            -o migration.sql
      
      - name: Apply migration
        env:
          PGPASSWORD: ${{ secrets.DB_PASSWORD }}
        run: |
          psql -h ${{ secrets.DB_HOST }} \
               -U ${{ secrets.DB_USER }} \
               -d ${{ secrets.DB_NAME }} \
               -f migration.sql
```

### Environments + protected secrets

В GitHub: Settings → Environments:
- `development` — auto deploy
- `staging` — auto после tests
- `production` — manual approval (1+ reviewers)

Secrets per environment:
```
production environment:
  DB_HOST=prod-db.example.com
  DB_PASSWORD=<from Vault>

staging environment:
  DB_HOST=staging-db.example.com
  DB_PASSWORD=<from Vault>
```

---

## 11. Pre-commit hooks

### Husky.NET

```bash
dotnet new tool-manifest
dotnet tool install Husky
dotnet husky install
```

`.husky/pre-commit`:
```bash
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

dotnet husky run --name pre-commit
```

`task-runner.json`:
```json
{
    "tasks": [
        {
            "name": "pre-commit",
            "command": "dotnet",
            "args": ["format", "--include", "${staged}"],
            "include": ["**/*.cs"]
        },
        {
            "name": "pre-commit",
            "command": "dotnet",
            "args": ["build", "--no-restore"]
        }
    ]
}
```

Автоматически форматирует код перед commit.

---

## 12. Dependabot — auto-update

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: nuget
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 10
    groups:
      microsoft-extensions:
        patterns: ["Microsoft.Extensions.*"]
      microsoft-aspnetcore:
        patterns: ["Microsoft.AspNetCore.*"]
      entity-framework:
        patterns: ["Microsoft.EntityFrameworkCore*", "Npgsql*"]
      testing:
        patterns: ["xunit*", "*Test*", "Shouldly", "NSubstitute"]
      analyzers:
        patterns: ["*Analyzer*", "Roslynator*", "SonarAnalyzer*"]
  
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
  
  - package-ecosystem: docker
    directory: /
    schedule:
      interval: weekly
```

---

## Best Practices

- **TreatWarningsAsErrors** — с первого дня. В legacy — постепенно через override на проектах
- **Central Package Management** — одна версия пакетов для всего solution
- **Directory.Build.props** — настройки централизованно, не дублировать
- **Analyzers**: SonarAnalyzer + Meziantou + Roslynator — защита от классических багов
- **User Secrets для local** — никогда не в git
- **Environment Variables / Vault для production** — никогда не в git
- **CI cache** — `~/.nuget/packages` с key по `Directory.Packages.props`
- **Fail fast** — restore → build → test → docker (в этом порядке)
- **Migrations отдельным job** — с approval для production
- **Image signing (cosign)** — supply chain security
- **CodeQL** — security scanning, weekly
- **Dependabot** — auto-update, group updates чтобы PRs не множились
- **Pre-commit hooks** — auto-format
- **global.json** — pinning SDK для reproducibility
- **Aspire** — для local dev orchestration с .NET 9+ проектов

---

## См. также

- [[code-quality|Code Quality]] — analyzers deep
- [[docker|Docker]] — Dockerfile best practices
- [[kubernetes|Kubernetes]] — k8s deployment
- [[architecture-decisions|Architecture Decisions]] — ADR process
- [[observability|Observability]] — OpenTelemetry deep

## Reading list

- **Anton Martyniuk — How to start a new .NET project in 2026** — antondevtips.com
- **Andrew Lock — .NET 8/9/10 series** — andrewlock.net
- **Tim Heuer — Microsoft .NET blog** — devblogs.microsoft.com/dotnet
- **Microsoft Docs — Central Package Management** — learn.microsoft.com/nuget/consume-packages/Central-Package-Management
- **Microsoft Docs — .NET Aspire** — learn.microsoft.com/dotnet/aspire
- **GitHub Actions for .NET** — docs.github.com/en/actions
- **External Secrets Operator** — external-secrets.io
- **Infisical docs** — infisical.com/docs
