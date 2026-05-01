---
tags: [architecture, twelve-factor, cloud-native, devops, configuration, deployment, senior]
level: Senior
date: 2026-05-01
---

# 12-Factor App — стандарты cloud-native приложений

> **Manifesto от Heroku (2011), стал industry standard.** 12 принципов для apps which можно reliably deploy в cloud. Closes пробел "знаю Docker и Kubernetes, но какие принципы делают app cloud-native".

---

## Что это, зачем и когда

### История

В 2011 Heroku опубликовали **12factor.net** — observations что делает SaaS app deployable, scalable, maintainable. Не язык-зависимо.

В 2026 — base для:
- **Kubernetes** ready apps
- **Cloud Native** certification (CNCF)
- **DevOps maturity** assessment
- **Microservices** good practices

### Зачем знать .NET-разработчику

Если deploy в:
- Docker / Kubernetes
- Azure App Service / Container Apps
- AWS Elastic Beanstalk / ECS
- Heroku / Railway / Render

— application должна быть 12-factor compliant. Иначе scale broken, deploy painful, dev/prod inconsistent.

### 12 принципов кратко

```
I.    Codebase          — Один codebase в Git, много deploys
II.   Dependencies       — Explicit declaration, no system-wide
III.  Config             — Store config в environment
IV.   Backing services   — Treat as attached resources (DB, cache, queue)
V.    Build, release, run — Separate stages
VI.   Processes          — Stateless, share-nothing
VII.  Port binding       — Self-contained service (own port)
VIII. Concurrency        — Scale через process model (горизонтально)
IX.   Disposability      — Fast startup, graceful shutdown
X.    Dev/prod parity    — Make дev≈prod
XI.   Logs               — Treat logs как event streams
XII.  Admin processes    — Run admin tasks как one-off processes
```

---

## I. Codebase — один codebase, много deploys

### Принцип

```
ОДИН codebase (Git repo)  →  МНОГО deploys (dev / staging / prod)
```

### ❌ Anti-patterns

- Несколько repo для разных environments
- "Магические" branches dev/prod различающиеся business logic
- Shared code между apps без проjektного boundary

### ✅ Правильно

```
my-app/                     ← один repo
├── src/
├── tests/
├── deployments/
│   ├── dev/values.yaml     ← env-specific config (НЕ code)
│   ├── staging/values.yaml
│   └── prod/values.yaml
└── .github/workflows/
    └── ci-cd.yml            ← deploy всех env'ов с одной кодовой базы
```

### .NET implementation

```bash
# Один repo, build один artifact, deploy в разные env'ы
dotnet publish -c Release -o ./publish

# Deploy same artifact в каждый env с разным config
docker build -t myapp:1.2.3 .
docker push registry/myapp:1.2.3

# Dev
kubectl apply -f deployments/dev/

# Prod (тот же image!)
kubectl apply -f deployments/prod/
```

См. [[../Infrastructure/cicd-github-actions|CI/CD]].

---

## II. Dependencies — explicit declaration

### Принцип

App **никогда** не должна полагаться на system-wide packages. Все dependencies **явно объявлены и locked**.

### ❌ Anti-patterns

```bash
# Dockerfile
RUN apt-get install some-tool  # system-wide
# App ожидает что tool в PATH — fragile!
```

### ✅ Правильно в .NET

```xml
<!-- MyApp.csproj — все dependencies explicit -->
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="10.0.0" />
  <PackageReference Include="Serilog.AspNetCore" Version="9.0.0" />
  <PackageReference Include="MediatR" Version="13.0.0" />
</ItemGroup>
```

```bash
# Lock через packages.lock.json
dotnet restore --use-lock-file

# Reproducible builds
dotnet restore --locked-mode
```

### CLI tools — local manifest

```bash
# Не global tools (system-wide)!
dotnet new tool-manifest
dotnet tool install dotnet-ef --version 10.0.0

# .config/dotnet-tools.json — committed
# Все devs / CI получают same versions
```

См. [[../CSharp/dotnet-cli-getting-started|.NET CLI]] и [[../Infrastructure/project-setup|Project Setup]].

---

## III. Config — в environment variables

### Принцип

**Code и config — strictly separated.** Config меняется между deploys (dev / staging / prod), code — нет.

### Litmus test

> "Можешь ли ты open-source свой codebase сейчас, не leak credentials?"

Если в коде есть `appsettings.Production.json` с paролем — **fail**.

### ❌ Anti-patterns

```csharp
// ❌ Hardcoded
private const string ConnectionString = "Server=prod-db;User=admin;Password=12345";

// ❌ В код committed appsettings.Production.json
{
  "ConnectionStrings": {
    "Default": "Server=prod;Password=secret"  // ⚠️ in git!
  }
}

// ❌ Per-env appsettings — анти-pattern даже без secrets
appsettings.Development.json  // 50 lines
appsettings.Production.json   // 50 lines (различия неявные)
```

### ✅ Правильно — env vars

```csharp
// Program.cs
var connectionString = builder.Configuration.GetConnectionString("Default")
    ?? throw new InvalidOperationException("Connection string not configured");

builder.Services.AddDbContext<AppDbContext>(opts =>
    opts.UseSqlServer(connectionString));

// JWT secret из env
var jwtSecret = builder.Configuration["Jwt:Secret"]
    ?? throw new InvalidOperationException("Jwt:Secret not configured");
```

```bash
# Dev — User Secrets
dotnet user-secrets set "ConnectionStrings:Default" "Server=localhost;..."
dotnet user-secrets set "Jwt:Secret" "dev-secret"

# Production — env vars
export ConnectionStrings__Default="Server=prod;..."
export Jwt__Secret="actual-prod-secret"
```

> [!info] ASP.NET Core конвенция
> `__` (double underscore) в env var = `:` в config path.
> `ConnectionStrings__Default` ⇒ `Configuration["ConnectionStrings:Default"]`.

### Production secrets — secret manager

```csharp
// Azure Key Vault
builder.Configuration.AddAzureKeyVault(
    new Uri("https://my-vault.vault.azure.net/"),
    new DefaultAzureCredential());

// AWS Secrets Manager
builder.Configuration.AddSecretsManager(
    region: RegionEndpoint.USEast1,
    configurator: opts => opts.SecretFilter = e => e.Name.StartsWith("myapp_"));

// HashiCorp Vault
builder.Configuration.AddVaultClient(/* ... */);
```

См. [[../AspNetCore/auth-security|Auth & Security]].

---

## IV. Backing services — attached resources

### Принцип

Все external services (DB, cache, queue, SMTP) — **attached resources**. Switchable через config без code changes.

### Что значит

```
Local Postgres (dev)        ↔  Azure SQL (prod)
Redis локальный (dev)        ↔  AWS ElastiCache (prod)
Папка ./uploads (dev)         ↔  S3 / Azure Blob (prod)
SMTP localhost (dev)          ↔  SendGrid (prod)
```

App **не должна знать** различие. Только URL/connection string меняются.

### ❌ Anti-pattern

```csharp
// ❌ Code зависит от типа storage
public class FileService
{
    public void Save(string path, byte[] data)
    {
        if (Environment.IsProduction())
            _s3.Upload(path, data);
        else
            File.WriteAllBytes(path, data);  // local
    }
}
```

### ✅ Правильно

```csharp
// Interface — abstraction
public interface IBlobStorage
{
    Task SaveAsync(string key, Stream data);
    Task<Stream> GetAsync(string key);
}

// Implementations
public class LocalBlobStorage : IBlobStorage { /* file system */ }
public class S3BlobStorage : IBlobStorage { /* AWS SDK */ }
public class AzureBlobStorage : IBlobStorage { /* Azure SDK */ }

// DI based на config
builder.Services.AddScoped<IBlobStorage>(sp =>
{
    var provider = sp.GetRequiredService<IConfiguration>()["Storage:Provider"];
    return provider switch
    {
        "S3" => new S3BlobStorage(/* ... */),
        "Azure" => new AzureBlobStorage(/* ... */),
        _ => new LocalBlobStorage(/* ... */)
    };
});
```

App's code не меняется при переходе local → S3 → Azure.

См. [[../Infrastructure/cloud-azure-basics|Cloud Azure Basics]] (TBD).

---

## V. Build, release, run — separate stages

### Принцип

```
BUILD                    RELEASE                   RUN
─────                    ───────                   ───
Code → Artifact          Artifact + Config        Execute
(immutable)              = Release                releases
                         (versioned)
```

Каждая stage **immutable**. После build — артефакт не меняется. После release — config locked.

### ❌ Anti-patterns

```bash
# Build на production server
ssh prod-server
cd /var/www/myapp
git pull
dotnet build
# Если build fails — production down!
```

### ✅ Правильно

```yaml
# .github/workflows/ci-cd.yml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: dotnet publish -c Release -o ./out
      - run: docker build -t myapp:${{ github.sha }} .
      - run: docker push registry/myapp:${{ github.sha }}
    # Артефакт зафиксирован — image:abc123def

  release-staging:
    needs: build
    steps:
      - run: |
          # Deploy image:abc123def с staging config
          kubectl set image deployment/myapp \
            myapp=registry/myapp:${{ github.sha }} \
            -n staging

  release-prod:
    needs: release-staging
    environment: production  # требует approval
    steps:
      - run: |
          # Тот же image, prod config
          kubectl set image deployment/myapp \
            myapp=registry/myapp:${{ github.sha }} \
            -n production
```

**Ключ:** prod использует **тот же** Docker image что staging. Различие — только env vars.

См. [[../Infrastructure/cicd-github-actions|CI/CD]].

---

## VI. Processes — stateless

### Принцип

App processes — **stateless и share-nothing**. State хранится в **backing services** (DB, cache).

### ❌ Anti-patterns

```csharp
// ❌ In-memory session
builder.Services.AddSession();  // memory by default
// Restart → user logged out
// Multiple instances → sticky sessions нужны (anti-pattern)

// ❌ In-memory cache как single source of truth
private static Dictionary<int, User> _userCache = new();
// Restart → cache lost
// Multiple instances → каждая своя copy

// ❌ Файлы на local disk
public void SaveUpload(IFormFile file)
{
    file.CopyTo(File.Create($"./uploads/{file.FileName}"));
    // Какой instance потом серве? Не найдёт файл
}
```

### ✅ Правильно

```csharp
// Sessions — Redis
builder.Services.AddStackExchangeRedisCache(opts =>
    opts.Configuration = builder.Configuration["Redis:ConnectionString"]);
builder.Services.AddSession();

// Cache — IDistributedCache (Redis)
public class UserService(IDistributedCache cache)
{
    public async Task<User?> GetById(int id)
    {
        var bytes = await cache.GetAsync($"user:{id}");
        // ...
    }
}

// Files — Blob storage
public async Task SaveUpload(IFormFile file)
{
    using var stream = file.OpenReadStream();
    await _blobStorage.SaveAsync($"uploads/{Guid.NewGuid()}", stream);
}
```

**Любой instance может handle любой request.** Restart instance — нет потерь.

См. [[../Infrastructure/nosql-databases|NoSQL Databases]] (Redis sessions).

---

## VII. Port binding — self-contained service

### Принцип

App сама **bindится к port** через HTTP. Не requires apache/nginx предварительно установленный.

### ❌ Anti-pattern (старый ASP.NET)

```
IIS hosts ASP.NET (требует IIS установленный)
↓
Nginx → IIS → ASP.NET
```

### ✅ Правильно (ASP.NET Core)

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.Run("http://0.0.0.0:8080");  // self-hosted Kestrel
```

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0
COPY publish/ /app/
EXPOSE 8080
ENTRYPOINT ["dotnet", "/app/MyApp.dll"]
# Container — self-contained service
```

App может выступать как backing service для другой app:
```
App A → http://service-b:8080  ← Service B is just a service
```

---

## VIII. Concurrency — scale through process model

### Принцип

Scale **horizontally** добавлением process instances, не vertical (bigger server).

### ❌ Anti-pattern

```
Production: 1 huge VM (64 cores, 256 GB)
Load повышается → купи bigger VM
Single point of failure
```

### ✅ Правильно

```yaml
# Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3   # 3 instances initially
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
spec:
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 70 }
```

**3 → 50 replicas** автоматом при загрузке. Failed instance — replaced.

### Process types — разные роли

```
Web (HTTP requests)              → 10 replicas
Worker (background jobs)          → 5 replicas
Scheduler (cron)                  → 1 replica
```

Каждая шкалируется independent.

См. [[../Infrastructure/kubernetes|Kubernetes]] и [[../AspNetCore/hosting-background|Hosting Background]].

---

## IX. Disposability — fast startup, graceful shutdown

### Принцип

Process должен:
- **Start fast** (1-10 сек) — для quick scaling
- **Stop gracefully** на SIGTERM — finish current requests перед exit

### Fast startup

```csharp
// ✅ Lazy initialization
builder.Services.AddSingleton<HeavyService>();  // создаётся при first use

// ✅ Native AOT — startup < 100 ms (vs 1-3 сек обычно)
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

### Graceful shutdown

```csharp
var app = builder.Build();

// ASP.NET Core делает graceful shutdown by default:
// 1. SIGTERM → app stops accepting new requests
// 2. Waits для finish current requests (up to ShutdownTimeout)
// 3. IHostedService.StopAsync called
// 4. Process exits

builder.Services.Configure<HostOptions>(opts =>
{
    opts.ShutdownTimeout = TimeSpan.FromSeconds(30);
});

// Custom IHostedService — graceful
public class WorkerService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            try
            {
                await ProcessJobAsync(ct);
            }
            catch (OperationCanceledException) when (ct.IsCancellationRequested)
            {
                // Graceful — finish current then exit
                break;
            }
        }
    }
}
```

### Idempotency для retries

```
Worker started job → Kubernetes kills pod → Job не finished
Replacement pod должен быть able to retry

Solution: idempotent operations с unique IDs
```

См. [[../Infrastructure/kubernetes|Kubernetes lifecycle]] и [[http-fundamentals|Idempotency]].

---

## X. Dev/prod parity — minimize gap

### Принцип

**Dev environment ≈ Prod environment.** Different по minor вещам, но same:
- Backing services (Postgres в обоих, не SQLite в dev / Postgres в prod)
- OS (Linux containers в обоих)
- Tools (same SDK version)

### ❌ Anti-pattern

```
Dev:  SQLite + IIS Express + Windows
Prod: Postgres + Linux + Kestrel + Kubernetes
```

→ "На моей машине работает!"-bugs.

### ✅ Правильно

```yaml
# docker-compose.yml для dev
services:
  app:
    build: .
    environment:
      - ConnectionStrings__Default=Host=postgres;...
  postgres:
    image: postgres:16     # SAME as prod
  redis:
    image: redis:7          # SAME as prod
```

Dev — same Postgres / Redis / Linux containers как prod.

### .NET Aspire — dev/prod parity tool

```csharp
// Aspire AppHost
var builder = DistributedApplication.CreateBuilder(args);

var postgres = builder.AddPostgres("postgres");
var redis = builder.AddRedis("redis");

builder.AddProject<Projects.MyApp_Api>("api")
    .WithReference(postgres)
    .WithReference(redis);

builder.Build().Run();
```

Запускаешь `dotnet run` → стартует Postgres + Redis + API. Same environment как prod.

См. [[../Infrastructure/dotnet-aspire|.NET Aspire]] (TBD).

---

## XI. Logs — event streams

### Принцип

App **не управляет** log files. Просто пишет в **stdout**. Внешняя система собирает.

### ❌ Anti-pattern

```csharp
// ❌ Управление файлами вручную
File.AppendAllText("./logs/app-2026-05-01.log", entry);

// Проблемы:
// - Container restart — logs lost
// - Multiple instances — fragmented logs
// - Rotation, cleanup — manual
// - Search — grep по файлам = боль
```

### ✅ Правильно — stdout

```csharp
// Serilog → stdout (default в ASP.NET Core)
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console(new JsonFormatter())
    .CreateLogger();

builder.Host.UseSerilog();
```

```csharp
// Просто log
_logger.LogInformation("Order {OrderId} created for user {UserId}", orderId, userId);
```

**Внешняя система** (k8s + Loki / ELK / Datadog) собирает stdout → searchable, retention, dashboards.

```yaml
# k8s — Loki sidecar или DaemonSet
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: app
      # logs to stdout
      image: myapp
    # Loki/Promtail собирает stdout всех containers
```

См. [[../AspNetCore/logging-observability|Logging & Observability]] и [[../Infrastructure/observability|Observability]].

---

## XII. Admin processes — one-off processes

### Принцип

Admin tasks (миграции DB, cleanup, scripts) — **one-off processes** запускаемые **с тем же кодом и configuration**, **в той же environment**.

### ❌ Anti-patterns

```bash
# ❌ SSH в prod, manual SQL
ssh prod-server
psql -U admin -d myapp
> DELETE FROM old_logs WHERE created_at < '2025-01-01';
# Ошибка человеческая = production catastrophe
```

### ✅ Правильно

```csharp
// Admin command — code в same repo
public class CleanupOldLogsCommand
{
    public async Task ExecuteAsync(IServiceProvider sp, CancellationToken ct)
    {
        var db = sp.GetRequiredService<AppDbContext>();
        var cutoff = DateTime.UtcNow.AddYears(-1);
        
        await db.AuditLogs
            .Where(l => l.CreatedAt < cutoff)
            .ExecuteDeleteAsync(ct);
    }
}

// Program.cs — режимы запуска
var builder = WebApplication.CreateBuilder(args);

if (args.Length > 0 && args[0] == "cleanup-logs")
{
    var app = builder.Build();
    using var scope = app.Services.CreateScope();
    await new CleanupOldLogsCommand().ExecuteAsync(scope.ServiceProvider, default);
    return;
}

// Normal web run
var app = builder.Build();
app.MapControllers();
app.Run();
```

```bash
# В production — same image, разная command
kubectl run cleanup --image=myapp:1.2.3 -- dotnet MyApp.dll cleanup-logs
# Same code, same config, one-off process
```

### EF Migrations

```bash
# Database migration — admin task
dotnet ef database update

# В CI/CD pipeline — отдельный step
- name: Migrate DB
  run: dotnet ef database update --connection "${{ secrets.PROD_DB }}"
```

Не auto-migrate на app startup в production!

См. [[../EFCore/migrations|EF Migrations]].

---

## Case Study #1 — Migration legacy → 12-factor

**Сценарий:** Legacy ASP.NET Framework app на Windows Server. Хотим cloud-native (Kubernetes, scale).

### Audit current state

```
I.    Codebase           ✓ один Git repo
II.   Dependencies       ✗ NuGet но system-wide tools (PowerShell, custom DLL)
III.  Config             ✗ web.config с hardcoded paths
IV.   Backing services   ✗ SQL Server connection в коде
V.    Build/release/run  ✗ build на dev machines, deploy через FTP
VI.   Processes          ✗ in-memory cache, files на disk
VII.  Port binding       ✗ IIS hosts (не self-contained)
VIII. Concurrency        ✗ vertical scaling (бigger server)
IX.   Disposability      ✗ slow startup (30+ сек), no graceful shutdown
X.    Dev/prod parity    ✗ dev = SQLite, prod = SQL Server
XI.   Logs               ✗ files в C:\logs\
XII.  Admin processes    ✗ remote desktop + manual SQL
```

### Migration plan

**Phase 1:** Migrate на ASP.NET Core
- Self-hosted Kestrel (VII)
- Linux containers (X)
- Graceful shutdown built-in (IX)

**Phase 2:** Stateless
- Move sessions → Redis (VI)
- Move file uploads → Blob storage (VI)
- Remove in-memory cache or distributed (VI)

**Phase 3:** Config из env
- Connection strings из env vars (III)
- Secrets в Key Vault (III)
- appsettings.{env}.json — only non-secret defaults

**Phase 4:** CI/CD
- GitHub Actions build (V)
- Docker images, semantic versioning (V)
- Same image dev → prod, different config (V, X)

**Phase 5:** Deploy
- Kubernetes + Helm (VIII, IX)
- HPA для autoscale (VIII)
- Loki для logs (XI)
- Migration command вместо manual SQL (XII)

### Результат

- 1 VM → 5-50 replicas auto-scaling
- Deploy time: 2 часа → 5 минут
- Recovery time: 30 минут (manual) → 30 секунд (automated)
- "Works on my machine"-bugs: 90% reduction

См. [[../Infrastructure/kubernetes|Kubernetes]] и [[../Infrastructure/cicd-github-actions|CI/CD]].

---

## Case Study #2 — Stateless processes — sessions migration

**Сценарий:** App на 1 instance (sticky sessions). Нужно scale до 10 instances.

### Problem

```csharp
builder.Services.AddSession();  // in-memory
// User logs in → instance 1 (session создан в memory)
// Next request → load balancer routes к instance 3
// Instance 3 не знает session → user logged out
```

### Solution — distributed session

```csharp
// Add Redis-backed cache
builder.Services.AddStackExchangeRedisCache(opts =>
    opts.Configuration = builder.Configuration["Redis:ConnectionString"]);

// Session uses cache
builder.Services.AddSession(opts =>
{
    opts.IdleTimeout = TimeSpan.FromMinutes(30);
    opts.Cookie.HttpOnly = true;
    opts.Cookie.IsEssential = true;
});
```

Все instances share sessions через Redis. **Любой** может handle **любой** request.

См. [[../Infrastructure/nosql-databases|Redis для sessions]].

---

## Case Study #3 — Build/release/run separation

**Сценарий:** Старая команда builds локально, FTPs в prod. Bugs от inconsistencies build environments.

### Solution — pipeline

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
    tags: ['v*']

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '10.0.x' }
      
      - name: Restore dependencies
        run: dotnet restore --locked-mode
      
      - name: Build
        run: dotnet build -c Release --no-restore
      
      - name: Test
        run: dotnet test --no-build -c Release
      
      - name: Publish
        run: dotnet publish -c Release -o ./out --no-build
      
      - name: Build & Push Docker
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

  deploy-staging:
    needs: build
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            -n staging
      
      - name: Smoke tests
        run: ./scripts/smoke-test-staging.sh

  deploy-prod:
    needs: deploy-staging
    environment: production    # GitHub gate — manual approval
    runs-on: ubuntu-latest
    steps:
      - run: |
          kubectl set image deployment/myapp \
            myapp=ghcr.io/${{ github.repository }}:${{ github.sha }} \
            -n production
```

**Result:**
- One build → multiple environments same artifact
- Roll-back = redeploy previous tag
- Audit trail в GitHub Actions

См. [[../Infrastructure/cicd-github-actions|CI/CD]].

---

## Case Study #4 — Disposability — graceful shutdown

**Сценарий:** Worker service обрабатывает orders. Kubernetes kills pod (deploy/scale-down). Without graceful shutdown — orders lost / partially processed.

### ❌ Без graceful shutdown

```csharp
public class OrderProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var order = await GetNextOrderAsync();
            await ProcessOrderAsync(order);  // если cancel here — order half-done
            await MarkAsCompleteAsync(order);
        }
    }
}
```

### ✅ Graceful

```csharp
public class OrderProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            Order? order = null;
            try
            {
                order = await GetNextOrderAsync(ct);
                if (order == null)
                {
                    await Task.Delay(TimeSpan.FromSeconds(5), ct);
                    continue;
                }

                // Idempotent processing — может быть retried
                await ProcessOrderAsync(order, ct);
                await MarkAsCompleteAsync(order, ct);
            }
            catch (OperationCanceledException) when (ct.IsCancellationRequested)
            {
                // Graceful — finish current order
                if (order is { IsProcessing: true })
                {
                    await MarkForRetryAsync(order);
                }
                break;
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Failed to process order");
            }
        }
    }
}

// Configure shutdown timeout
builder.Services.Configure<HostOptions>(opts =>
{
    opts.ShutdownTimeout = TimeSpan.FromSeconds(30);  // время для finish
});
```

```yaml
# k8s
spec:
  containers:
    - name: worker
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]  # дать time start shutdown
      terminationGracePeriodSeconds: 60  # max wait для graceful
```

См. [[../AspNetCore/hosting-background|Hosting Background]] и [[../Infrastructure/kubernetes|Kubernetes lifecycle]].

---

## Case Study #5 — Logs as streams

**Сценарий:** App пишет в файлы. После rotation — старые logs lost. Multiple instances — fragmented logs.

### Migration план

```csharp
// 1. Remove file logging
// program.cs БЫЛО:
Log.Logger = new LoggerConfiguration()
    .WriteTo.File("logs/app.txt", rollingInterval: RollingInterval.Day)
    .CreateLogger();

// СТАЛО — только stdout (JSON)
Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .Enrich.WithMachineName()
    .Enrich.WithProperty("Application", "MyApp")
    .WriteTo.Console(new CompactJsonFormatter())
    .CreateLogger();
```

```yaml
# 2. k8s + Loki (или Datadog/ELK)
# Promtail / Fluent Bit на каждой node собирает stdout → Loki
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
data:
  fluent-bit.conf: |
    [INPUT]
        Name tail
        Path /var/log/containers/*.log
    [OUTPUT]
        Name loki
        Match *
        Url http://loki:3100/loki/api/v1/push
```

**Результат:**
- Logs centralized в Loki / ELK
- Searchable: `{app="myapp",level="error",userId="42"}`
- Retention настраивается централизованно
- Dashboards в Grafana

См. [[../Infrastructure/observability|Observability]].

---

## Case Study #6 — Admin tasks без SSH

**Сценарий:** Periodic data cleanup. Раньше — DBA SSH в prod, manual SQL.

### ✅ Solution — admin command

```csharp
public class Program
{
    public static async Task<int> Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);
        ConfigureServices(builder.Services, builder.Configuration);

        // Admin commands
        if (args.Length > 0 && args[0] == "admin")
        {
            var app = builder.Build();
            return await RunAdminAsync(app, args.Skip(1).ToArray());
        }

        // Normal web app
        var webApp = builder.Build();
        ConfigureMiddleware(webApp);
        await webApp.RunAsync();
        return 0;
    }

    private static async Task<int> RunAdminAsync(WebApplication app, string[] args)
    {
        var command = args.FirstOrDefault();
        using var scope = app.Services.CreateScope();
        var sp = scope.ServiceProvider;

        return command switch
        {
            "cleanup-old-logs" => await new CleanupOldLogsCommand(sp).RunAsync(),
            "rebuild-search-index" => await new RebuildSearchIndexCommand(sp).RunAsync(),
            "migrate" => await new MigrateCommand(sp).RunAsync(),
            _ => Usage()
        };
    }

    private static int Usage()
    {
        Console.WriteLine("Usage: dotnet MyApp.dll admin <command>");
        Console.WriteLine("Commands: cleanup-old-logs, rebuild-search-index, migrate");
        return 1;
    }
}
```

```bash
# Run admin task в production — same image, разный command
kubectl run cleanup-job \
    --image=ghcr.io/myorg/myapp:v1.2.3 \
    --restart=Never \
    --rm -it \
    -- dotnet MyApp.dll admin cleanup-old-logs

# Or as scheduled K8s CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-logs
spec:
  schedule: "0 2 * * *"  # daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: cleanup
              image: ghcr.io/myorg/myapp:v1.2.3
              args: ["admin", "cleanup-old-logs"]
          restartPolicy: OnFailure
```

**Преимущества:**
- Admin code в same repo (review, tests)
- Same env как production
- Auditable (k8s logs)
- Не SSH в production

---

## Common Pitfalls

### 1. Multiple branches вместо config

```
❌ branch dev → SQLite
   branch prod → Postgres

✅ Один main branch, env vars decide
```

### 2. Hardcoded URLs / paths

```csharp
// ❌
private const string ApiUrl = "https://api.production.com";

// ✅
private readonly string _apiUrl = config["ExternalApi:Url"];
```

### 3. State в process memory

```csharp
// ❌ Static cache, in-memory sessions
private static Dictionary<int, User> _cache = new();

// ✅ Redis / IDistributedCache
```

### 4. File system как persistent storage

```csharp
// ❌ Сохранение в ./uploads
File.WriteAllBytes($"./uploads/{id}.bin", data);

// ✅ Blob storage
await _blobStorage.SaveAsync(id, stream);
```

### 5. Auto-migrate на startup

```csharp
// ❌ В Program.cs
db.Database.Migrate();  // each instance startup → race conditions
```

Migration — admin task. Запускай один раз через CI/CD.

### 6. No graceful shutdown

```csharp
// ❌ kill -9 проце ss
// In-flight requests dropped, jobs incomplete
```

### 7. Build на target server

```bash
# ❌ ssh prod && git pull && dotnet build
```

Build разный на каждой машине → inconsistencies.

### 8. Vertical scaling по умолчанию

```yaml
# ❌
resources:
  limits: { cpu: 8000m, memory: 16Gi }  # один pod, big

# ✅ Multi-replica + HPA
replicas: 5
resources:
  limits: { cpu: 1000m, memory: 1Gi }
```

### 9. Logs в файлы

```csharp
.WriteTo.File("logs.txt")  // ❌
```

### 10. `appsettings.Production.json` в git

```gitignore
# .gitignore — обязательно
appsettings.Production.json
appsettings.Local.json
```

Production config — env vars или secret manager.

---

## Best Practices

### Architecture

- **Stateless processes** — все state в backing services
- **Self-contained services** — own port, no IIS dependency
- **Horizontal scaling first** — replicas, не bigger VM
- **Same image dev/prod** — config меняется, code нет

### Configuration

- **Env vars для config** — не hardcoded
- **Secret manager** для credentials (Key Vault, AWS Secrets, Vault)
- **Один codebase, много deploys** — branches не для env'ов
- **`.env.example`** в repo (без values)

### Deployment

- **Build один раз** — same artifact дев/staging/prod
- **Blue/green или canary** для production
- **Migration через admin command** не auto-startup
- **Graceful shutdown** обязательно

### Observability

- **Logs в stdout** — внешняя система соберёт
- **Structured logging** (JSON)
- **Correlation IDs** во всех services
- **Health endpoints** для k8s probes
- **Metrics через OpenTelemetry**

См. [[../Infrastructure/kubernetes|Kubernetes]], [[../Infrastructure/observability|Observability]], [[../Infrastructure/cicd-github-actions|CI/CD]].

---

## Cheat sheet

| Factor | Что значит для .NET |
|--------|---------------------|
| I. Codebase | Один Git repo + multiple deploys |
| II. Dependencies | csproj + lock files, no system tools |
| III. Config | env vars, User Secrets, Key Vault |
| IV. Backing services | Interface + DI based на config |
| V. Build/release/run | CI/CD pipeline, immutable artifacts |
| VI. Processes | Stateless, Redis sessions, blob storage |
| VII. Port binding | Self-hosted Kestrel |
| VIII. Concurrency | Replicas + HPA, не bigger VM |
| IX. Disposability | Native AOT для startup, graceful shutdown |
| X. Dev/prod parity | Docker Compose / .NET Aspire с same backing services |
| XI. Logs | Serilog → stdout, Loki/ELK collects |
| XII. Admin processes | dotnet MyApp.dll admin <command>, k8s Jobs |

---

## Decision tree

```
Готова ли app для cloud?
│
├── Сохраняет state в memory / files?
│   → НЕТ. Move в Redis / Blob storage (VI)
│
├── Hardcoded config / secrets?
│   → НЕТ. Move в env vars / secret manager (III)
│
├── Build на dev машинах?
│   → НЕТ. CI/CD pipeline (V)
│
├── Logs в файлы?
│   → НЕТ. stdout + collector (XI)
│
├── Manual SSH для admin tasks?
│   → НЕТ. Admin commands (XII)
│
├── Single VM, vertical scale?
│   → НЕТ. Replicas + HPA (VIII)
│
├── Slow startup, abrupt shutdown?
│   → Optimize (IX)
│
├── Dev environment ≠ Prod?
│   → Docker Compose / Aspire (X)
│
└── ✅ All green → cloud-native ready
```

---

## См. также

- [[../Infrastructure/kubernetes|Kubernetes]] — runtime для 12-factor apps
- [[../Infrastructure/docker|Docker]] — packaging
- [[../Infrastructure/cicd-github-actions|CI/CD]] — build/release/run
- [[../Infrastructure/observability|Observability]] — logs as streams
- [[../Infrastructure/nosql-databases|NoSQL]] — Redis for stateless
- [[../AspNetCore/auth-security|Auth & Security]] — secrets management
- [[../AspNetCore/hosting-background|Hosting Background]] — workers, graceful shutdown
- [[../EFCore/migrations|EF Migrations]] — admin tasks
- [[../Infrastructure/dotnet-aspire|.NET Aspire]] (TBD) — dev/prod parity tool

## Reading list

- **12factor.net** — оригинальный manifesto (must-read, free)
- **The Twelve-Factor App** — Adam Wiggins (PDF, 50 pages)
- **Cloud Native Patterns** — Cornelia Davis (book)
- **Designing Distributed Systems** — Brendan Burns (Microsoft)
- **Microsoft Architecture Guide** — learn.microsoft.com/azure/architecture/guide
- **Kubernetes Up & Running** — Kelsey Hightower
