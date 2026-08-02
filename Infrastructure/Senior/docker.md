---
tags: [docker, containers, buildkit, multi-stage, distroless, docker-compose, healthchecks, cgroups, namespaces, pid1, signals, security, aspire, devcontainers]
level: Senior
date: 2026-08-02
---

# Docker и контейнеры для .NET

> Самая глубокая заметка по контейнеризации в этом vault. Цель — закрыть все вопросы по Docker, которые задают на Senior/Tech Lead интервью + всё, что реально нужно для production: от Linux internals до supply chain security.

---

## Что это, зачем и когда

### Что такое Docker?

**Контейнеризация — упаковка приложения с зависимостями в isolated unit, которое запускается одинаково на любой машине** где есть Docker / containerd. Не VM (нет своего OS), а sandbox процессов через **Linux namespaces** + **cgroups**.

**Аналогия:** VM — это отдельная квартира с собственной кухней / ванной. Container — это комната в общежитии: своя кровать (filesystem), но разделяет инфраструктуру здания (kernel) с другими комнатами. Легче, быстрее, дешевле.

### Зачем

| Без Docker | С Docker |
|------------|----------|
| "Работает на моей машине" | Работает везде где есть Docker |
| Установка .NET, конфиг, deps на каждой машине | `docker run` — всё включено |
| Production отличается от dev | Same image на всех environments |
| Долгий setup новой dev машины | `docker compose up` — готово |
| Тяжёлый rollback | `docker run prev-tag` — instant |

### Когда применять
- Любой backend — контейнер default 2026
- Microservices — контейнеры обязательны
- Local dev — Postgres / Redis / RabbitMQ как Docker
- Не для desktop app (WPF, WinForms — обычные exe)

---

## Container internals — namespaces и cgroups

Docker — не магия. Это **обёртка над Linux kernel features**, которые существуют с 2008 года: namespaces (изоляция) + cgroups (лимиты ресурсов) + chroot/pivot_root (filesystem).

### Linux Namespaces — изоляция

| Namespace | Что изолирует |
|-----------|---------------|
| **PID** | Список процессов — внутри контейнера PID 1 = твой процесс |
| **NET** | Сетевые интерфейсы, routing, iptables |
| **MNT** | Точки монтирования файловой системы |
| **UTS** | Hostname, domain name |
| **IPC** | System V IPC, POSIX message queues |
| **USER** | UID/GID mapping (`uid 0` в контейнере = `uid 1000` на хосте) |
| **CGROUP** | Изоляция cgroup view |
| **TIME** (Linux 5.6+) | Часы (CLOCK_MONOTONIC, CLOCK_BOOTTIME) |

```bash
# Посмотреть namespaces процесса
ls -la /proc/<PID>/ns/

# Lsof: namespaces всех контейнеров
docker inspect --format '{{.State.Pid}}' <container>
```

### CGroups — лимиты ресурсов

CGroups (Control Groups) — механизм ядра для **ограничения** (CPU, memory, I/O) и **учёта** (accounting) ресурсов для группы процессов.

#### CGroups v1 vs v2

| | CGroups v1 | CGroups v2 |
|--|-----------|-----------|
| Controllers | Каждый ресурс — отдельная иерархия | Единая иерархия |
| Memory | `/sys/fs/cgroup/memory/...` | `/sys/fs/cgroup/...` |
| CPU | Раздельно `cpu` и `cpuacct` | Объединено |
| Поддержка | Все Linux kernel | Linux 4.5+, default 5.8+ |
| systemd | По умолчанию v1 (старые дистрибутивы) | Default в Ubuntu 22+, RHEL 9+, Fedora 31+ |

#### Влияние на .NET

**.NET 6+** автоматически читает cgroups limits и подстраивает:
- `Environment.ProcessorCount` — отражает CPU quota (не количество физических ядер)
- ServerGC heap count — по `ProcessorCount`
- ThreadPool — adapts
- GC heap hard limit — соответствует memory limit (с .NET 8)

**.NET 7 и старше** имели проблемы с cgroups v2 — в некоторых случаях видели весь host. **.NET 8+** полностью поддерживает обе версии.

```csharp
// Внутри контейнера с CPU limit 0.5
Console.WriteLine(Environment.ProcessorCount);  // 1 (округление вверх)

// Memory limit 512 MB
GCMemoryInfo info = GC.GetGCMemoryInfo();
Console.WriteLine(info.TotalAvailableMemoryBytes);  // ~512 MB, не RAM хоста
```

> [!warning] Старые контейнерные образы
> Если ты запускаешь .NET 5 или ниже — он может игнорировать cgroups limits и видеть весь host. Это означает: ServerGC создаёт heap-per-host-core (например 32 heap в 0.5 CPU контейнере) → memory blowup, throttling. Решение: апгрейд минимум до .NET 8.

### Что Docker — НЕ изолирует

- **Kernel** — все контейнеры разделяют kernel хоста (не как VM)
- **Time/clock** (до Linux 5.6 / time namespace)
- **Hardware** (USB, PCI — нужен `--device`)
- **CPU MSRs / perf counters** — нужны privileged

---

## BuildKit — современный Docker build

С Docker 23+ BuildKit включён по умолчанию. Старый builder deprecated. BuildKit:
- **Параллельный build** независимых stage'ей
- **Cache mount** для package downloads
- **Layer caching** intelligent (содержимое, не timestamp)
- **Secrets** в build без leak в image
- **SBOM / provenance** для security

```bash
# Force включить (если на старом Docker)
DOCKER_BUILDKIT=1 docker build .

# Или через .env
echo "DOCKER_BUILDKIT=1" >> ~/.bashrc

# BuildKit CLI (buildx) — features больше
docker buildx build --platform=linux/amd64,linux/arm64 -t myapp:latest .
```

### Cache mount types

```dockerfile
# syntax=docker/dockerfile:1.7

# 1. Inline cache — в image, для CI cross-build
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore

# 2. Locally-mounted cache (default)
docker buildx build --cache-from=type=local,src=/tmp/buildcache .
```

### Cache backends в CI

| Backend | Где | Когда |
|---------|-----|-------|
| `inline` | В image (последний layer) | Простой setup, registry достаточно |
| `registry` | Отдельный registry tag | Distributed CI с registry |
| `gha` | GitHub Actions cache | GitHub Actions |
| `local` | Local FS | Развитие на dev машине |
| `s3` | S3 bucket | AWS-based CI/CD |
| `azblob` | Azure Blob | Azure DevOps |

Пример GHA:
```yaml
- uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

`mode=max` — сохраняет cache всех intermediate stages (не только final). Дороже по storage, но каждый stage можно переиспользовать.

---

## Multi-stage build — production pattern

Цель — final image маленький, без SDK / build artifacts / source code.

```dockerfile
# syntax=docker/dockerfile:1.7
ARG DOTNET_VERSION=10.0

# ------- Stage 1: build -------
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION} AS build
WORKDIR /src

# Сначала копируем только csproj — для cache restore deps
COPY ["src/MyApp.Api/MyApp.Api.csproj", "src/MyApp.Api/"]
COPY ["src/MyApp.Application/MyApp.Application.csproj", "src/MyApp.Application/"]
COPY ["src/MyApp.Infrastructure/MyApp.Infrastructure.csproj", "src/MyApp.Infrastructure/"]
COPY ["src/MyApp.Domain/MyApp.Domain.csproj", "src/MyApp.Domain/"]

# Cache mount для NuGet packages
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet restore "src/MyApp.Api/MyApp.Api.csproj"

# Теперь весь source
COPY . .

# Build с cache mount
RUN --mount=type=cache,target=/root/.nuget/packages \
    dotnet publish "src/MyApp.Api/MyApp.Api.csproj" \
        -c Release \
        -o /app/publish \
        --no-restore \
        /p:UseAppHost=false

# ------- Stage 2: runtime -------
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-chiseled AS runtime
WORKDIR /app

# Non-root user: во всех .NET 8+ images есть user `app` (UID 1654, env $APP_UID);
# chiseled-образы запускаются под ним по умолчанию
USER app

COPY --from=build --chown=app:app /app/publish .

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
    CMD ["/app/MyApp.Api", "--health-check"]

ENV ASPNETCORE_URLS=http://+:8080
ENV DOTNET_RUNNING_IN_CONTAINER=true
ENV DOTNET_USE_POLLING_FILE_WATCHER=false

EXPOSE 8080

ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

### Размер итогового image

| Variant | Size | Когда |
|---------|------|-------|
| `dotnet/sdk:10.0` (build только!) | ~1.0 GB | Не для prod |
| `dotnet/aspnet:10.0` (Debian-based) | 220 MB | Default |
| `dotnet/aspnet:10.0-jammy` (Ubuntu 22.04) | 230 MB | Для Ubuntu compatibility |
| `dotnet/aspnet:10.0-noble` (Ubuntu 24.04 LTS) | 230 MB | Latest LTS Ubuntu |
| `dotnet/aspnet:10.0-alpine` | 110 MB | musl libc, smaller |
| `dotnet/aspnet:10.0-azurelinux` (Azure Linux/Mariner) | 180 MB | Microsoft Linux distro |
| `dotnet/aspnet:10.0-chiseled` | 110 MB | Distroless-style, glibc |
| `dotnet/aspnet:10.0-noble-chiseled` | 105 MB | Ubuntu chiseled, latest |
| `dotnet/aspnet:10.0-chiseled-extra` | 130 MB | + ICU, tzdata, ca-certs |
| `dotnet/runtime-deps:10.0-chiseled` + AOT | 25 MB | Только AOT-binary |
| Distroless `cc-debian12` + AOT | 18 MB | См. [[native-aot|Native AOT]] |
| `scratch` + statically-linked AOT | 8 MB | Maximum minification |

> [!question]- **Интервью: чем chiseled отличается от alpine?**
> **Alpine** — minimal Linux distro с **musl libc** вместо glibc. Меньший размер. Иногда compatibility issues — некоторые .NET libraries требуют glibc (PerfView, sometimes EF.SqlServer).
> **Chiseled** — Microsoft's distroless: только runtime + minimum libraries, **glibc-based**, no shell, no package manager. Маленький размер + compatibility + security (нет attack surface from system tools).
> Default 2026 — **chiseled** (Noble-chiseled последняя), не alpine.

> [!question]- **Интервью: что такое Azure Linux / Mariner?**
> Microsoft's distribution Linux, оптимизирован для Azure. Меньше пакетов, быстрее security patches от Microsoft, идеально для AKS. Альтернатива Ubuntu/Debian для тех, кто полностью в Azure.

---

## Анатомия Docker image — OCI spec

### Что такое image

**OCI Image** = **manifest** + **config** + **layers** (tarballs).

```
my-image:tag
├── manifest.json    — список layers, их digests, platform
├── config.json      — env, cmd, entrypoint, history
└── layers/
    ├── layer1.tar.gz  (sha256:abc...)
    ├── layer2.tar.gz  (sha256:def...)
    └── layer3.tar.gz  (sha256:ghi...)
```

### Layers — copy-on-write

Каждая инструкция Dockerfile (`RUN`, `COPY`, `ADD`) создаёт **layer** — diff к предыдущему. Финальный filesystem — **union** всех layers через **OverlayFS**.

```
Stack of layers:
[layer 4: COPY app] ← top, writable thin layer at runtime
[layer 3: RUN install]
[layer 2: COPY csproj]
[layer 1: FROM aspnet:10.0]
```

```bash
# Посмотреть layers
docker history myapp:latest
docker inspect myapp:latest --format '{{json .RootFS.Layers}}'
```

### Storage drivers

| Driver | Описание | Default |
|--------|----------|---------|
| **overlay2** | OverlayFS, fast copy-on-write | Default Linux |
| **btrfs** | btrfs snapshots, лучше для большого I/O | Опция |
| **zfs** | ZFS snapshots, deduplication | Опция |
| **aufs** | Старый, deprecated | Не использовать |
| **devicemapper** | Старый | Deprecated |
| **fuse-overlayfs** | OverlayFS в userspace | Rootless Docker |

```bash
docker info | grep "Storage Driver"
```

---

## Layer caching — оптимизация build time

Docker кэширует layers в порядке Dockerfile. Если меняется upstream layer — все downstream rebuild.

### Главный паттерн: dependencies first

```dockerfile
# ❌ BAD — копируем всё, любое изменение source = full restore
FROM sdk AS build
COPY . .
RUN dotnet restore && dotnet publish -o /app

# ✅ GOOD — restore кэшируется пока не меняются csproj
FROM sdk AS build
COPY ["**/*.csproj", "./"]
RUN dotnet restore
COPY . .
RUN dotnet publish --no-restore -o /app
```

При изменении `.cs` файла restore не выполняется (csproj's same → cached). Build быстрее в 5-10 раз.

### .dockerignore — не копировать лишнее

```
# .dockerignore
**/bin
**/obj
**/.vs
**/.idea
**/.git
**/.gitignore
**/Dockerfile*
**/docker-compose*.yml
**/*.md
**/node_modules
**/coverage
**/.env
**/secrets/
**/.user.props
**/TestResults
```

Без этого `COPY . .` копирует огромный `bin/obj/`. Docker context blow up → slow builds.

---

## Distroless / chiseled — безопасность

### Microsoft Chiseled images

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0-chiseled AS runtime
```

Содержит:
- ✅ .NET runtime
- ✅ glibc + minimum libraries
- ❌ Никаких package managers (apt, dpkg)
- ❌ Никакого shell (sh, bash)
- ❌ No coreutils

**Преимущества:**
- Меньше CVEs (нет vulnerable packages)
- Меньше attack surface (нет shell для exploitation)
- Smaller image (~110 MB vs ~220 MB)
- Pre-configured non-root user `app` (UID **1654**, задан env-переменной `$APP_UID`; ранние .NET 8 preview использовали 64198 — в GA число сменили). UID практически значим: его же прописывают в k8s `securityContext.runAsUser: 1654`

**Недостатки:**
- Нельзя `docker exec -it container sh` для debugging
- Нет debugging tools
- Hard для troubleshooting

```bash
# Debugging chiseled — копируй diagnostics image для проверки
docker run --rm -it --pid=container:my_app mcr.microsoft.com/dotnet/runtime:10.0-extra sh

# Или: ephemeral container в k8s 1.23+
kubectl debug -it my-pod --image=mcr.microsoft.com/dotnet/runtime:10.0-extra --target=app
```

### Chiseled `extra` variants

Microsoft предлагает `-extra` варианты с дополнительными библиотеками:

```dockerfile
# С ICU (для globalization), tzdata, ca-certificates
FROM mcr.microsoft.com/dotnet/aspnet:10.0-chiseled-extra
```

Когда нужен:
- Globalization (`CultureInfo.GetCultureInfo("ru-RU")`)
- Time zones (`TimeZoneInfo`)
- Outgoing HTTPS к недефолтным CA

### Google Distroless

Альтернатива от Google. Несколько вариантов:
```dockerfile
# Только AOT-binaries (нет .NET runtime)
FROM gcr.io/distroless/cc-debian12 AS runtime
COPY --from=build /app/publish/MyApp /
ENTRYPOINT ["/MyApp"]
```

См. [[native-aot|Native AOT]] — distroless + AOT = 18 MB image.

---

## PID 1 problem — почему это важно для .NET

В Linux PID 1 — особый процесс (init). Он:
- Получает SIGCHLD от orphaned процессов и должен их reap'ить
- Является default destination для signals от orchestrator
- Обрабатывает специально: SIGTERM, SIGINT передаются ему

Когда `ENTRYPOINT ["dotnet", "MyApp.dll"]` — **dotnet становится PID 1**.

### Проблема 1: SIGTERM не доходит

Если ENTRYPOINT — shell-form `ENTRYPOINT dotnet MyApp.dll` (без брекетов), то PID 1 = `/bin/sh -c "dotnet MyApp.dll"`. SIGTERM пойдёт в shell, который **может не передать его** дочернему dotnet → graceful shutdown сломан.

```dockerfile
# ❌ BAD — shell form
ENTRYPOINT dotnet MyApp.dll

# ✅ GOOD — exec form, PID 1 = dotnet
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Проблема 2: Zombie processes

Если контейнер запускает дочерние процессы (например, через `Process.Start`), они становятся orphans при exit'е парента. PID 1 должен reap'ить их (`waitpid()`). **dotnet не делает этого** → зомби накапливаются → eventually fork() fails.

### Решение: init system

```dockerfile
# Использовать tini как init
FROM mcr.microsoft.com/dotnet/aspnet:10.0
RUN apt-get update && apt-get install -y tini
ENTRYPOINT ["tini", "--", "dotnet", "MyApp.dll"]
```

Или через Docker:
```bash
docker run --init my-image
```

```yaml
# docker-compose
services:
  api:
    init: true
```

`tini` (или `docker --init`'s default `dumb-init`) — крошечный init, который:
- Получает SIGTERM, передаёт child
- Reap'ит зомби

> [!info] Когда нужен `--init`
> Если приложение запускает дочерние процессы (Puppeteer, ffmpeg, headless browsers, scripts). Для чистого .NET API — обычно не нужно, но `--init` не вредит.

---

## Signal handling и Graceful Shutdown

### Что происходит при остановке контейнера

1. Orchestrator (k8s/compose) → `docker stop` → SIGTERM в PID 1
2. .NET получает SIGTERM → `IHostApplicationLifetime.ApplicationStopping` event
3. Generic Host вызывает `IHostedService.StopAsync()` для всех services
4. Если за `ShutdownTimeout` (default 30s) не завершилось → SIGKILL

### Graceful shutdown в ASP.NET Core

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.Configure<HostOptions>(opts =>
{
    opts.ShutdownTimeout = TimeSpan.FromSeconds(30);  // Default
});

var app = builder.Build();

// Drain in-flight requests
app.Lifetime.ApplicationStopping.Register(() =>
{
    Console.WriteLine("Shutdown initiated, draining...");
});

app.Lifetime.ApplicationStopped.Register(() =>
{
    Console.WriteLine("Shutdown complete");
});

await app.RunAsync();
```

### BackgroundService и cancellation

```csharp
public class WorkerService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await ProcessAsync(stoppingToken);
                await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                // Expected on shutdown
                break;
            }
        }
    }
}
```

### STOPSIGNAL — переопределить SIGTERM

```dockerfile
# Если приложение слушает SIGUSR1 для graceful (например nginx)
STOPSIGNAL SIGUSR1
```

.NET по умолчанию — SIGTERM, не меняй без причины.

### Kubernetes preStop hook

```yaml
spec:
  containers:
    - name: api
      image: myapp
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "5"]  # дать LB убрать pod из rotation
      terminationGracePeriodSeconds: 60
```

`preStop` выполняется **перед** SIGTERM. `sleep 5` даёт kube-proxy время удалить pod из endpoints. Иначе race: SIGTERM пришёл, мы перестали accept'ить, но balancer ещё шлёт. См. [[kubernetes|Kubernetes]].

---

## Secrets в build — без leak

Старый плохой паттерн:
```dockerfile
# ❌ BAD — token остаётся в layer history forever
ARG NUGET_TOKEN
RUN dotnet nuget add source --username u --password ${NUGET_TOKEN} ...
```

`docker history` покажет эти ARG, secrets exposed.

BuildKit secrets:
```dockerfile
# syntax=docker/dockerfile:1.7
RUN --mount=type=secret,id=nuget_token,target=/tmp/token \
    NUGET_TOKEN=$(cat /tmp/token) && \
    dotnet nuget add source --username u --password ${NUGET_TOKEN} ...
```

Build:
```bash
docker build --secret id=nuget_token,src=./.nuget-token .

# Или через env
docker build --secret id=nuget_token,env=NUGET_TOKEN .
```

Token mount'ится только при выполнении этого RUN, **не сохраняется** в layer.

### SSH agent forwarding (для private git deps)

```dockerfile
RUN --mount=type=ssh \
    git clone git@github.com:myorg/private-repo.git /deps
```

```bash
docker build --ssh default=$SSH_AUTH_SOCK .
```

---

## ENTRYPOINT vs CMD — комбинации

```dockerfile
# Pattern 1: ENTRYPOINT only (most common для .NET)
ENTRYPOINT ["dotnet", "MyApp.dll"]
# docker run myimage              → dotnet MyApp.dll
# docker run myimage --port 8080  → dotnet MyApp.dll --port 8080

# Pattern 2: ENTRYPOINT + CMD (default args)
ENTRYPOINT ["dotnet", "MyApp.dll"]
CMD ["--environment", "Production"]
# docker run myimage              → dotnet MyApp.dll --environment Production
# docker run myimage --debug      → dotnet MyApp.dll --debug (CMD overridden)

# Pattern 3: только CMD (полностью overridable)
CMD ["dotnet", "MyApp.dll"]
# docker run myimage              → dotnet MyApp.dll
# docker run myimage bash         → bash (всё CMD заменилось)

# Pattern 4: shell form (НЕ рекомендуется для .NET)
ENTRYPOINT dotnet MyApp.dll  # exec через /bin/sh -c
# Проблемы: SIGTERM не доходит, нет PID 1 → используй exec form

```

### exec form vs shell form

| | exec form | shell form |
|--|-----------|-----------|
| Синтаксис | `ENTRYPOINT ["bin", "arg"]` | `ENTRYPOINT bin arg` |
| Запуск | Прямой exec | Через `/bin/sh -c` |
| PID 1 | Твой бинарник | sh |
| SIGTERM | Доходит | Может не дойти |
| Variable expansion | Нет ($VAR не работает) | Да |
| Default | Рекомендуется | Не использовать |

### Variable expansion в exec form

```dockerfile
# ❌ Не работает в exec form
ENTRYPOINT ["dotnet", "$APP_DLL"]  # $APP_DLL не expand'ится

# ✅ Workaround через shell wrapper
ENTRYPOINT ["sh", "-c", "exec dotnet $APP_DLL \"$@\"", "--"]
# `exec` важен — заменяет sh на dotnet, dotnet становится PID 1

```

---

## ARG vs ENV — различия

| | ARG | ENV |
|--|-----|-----|
| Видим в | Build time | Build + run time |
| После build | Не сохраняется | Доступно в running container |
| Default value | Опционален | Должен быть задан |
| `docker history` | Видно | Видно |
| Secrets | ❌ Никогда (leak) | ❌ Не для secrets |
| Override at run | ❌ Нет | ✅ `-e VAR=value` |
| Override at build | ✅ `--build-arg` | Только через ARG |

```dockerfile
# Версия известна на build
ARG DOTNET_VERSION=10.0
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}

# Конфиг для runtime
ENV ASPNETCORE_URLS=http://+:8080
ENV DOTNET_RUNNING_IN_CONTAINER=true

# ARG → ENV (передать build arg в runtime)
ARG BUILD_ID
ENV BUILD_ID=${BUILD_ID}
```

---

## .NET environment variables — полный список

| Переменная | Что | Default |
|------------|-----|---------|
| `DOTNET_RUNNING_IN_CONTAINER` | Hint для .NET что в container | `true` (auto в MS images) |
| `DOTNET_USE_POLLING_FILE_WATCHER` | dotnet watch — polling вместо inotify | `false` |
| `DOTNET_PROCESSOR_COUNT` | Override CPU detection | auto |
| `DOTNET_GCHeapHardLimit` | Hard limit на heap (.NET 5+) | auto |
| `DOTNET_GCHeapHardLimitPercent` | % от cgroup memory limit | 75 |
| `DOTNET_GCServer` | Server GC on/off | 0 (default workstation) |
| `DOTNET_GCConcurrent` | Concurrent/Background GC | 1 |
| `DOTNET_GCRetainVM` | Hold virtual memory | 0 |
| `DOTNET_GCDynamicAdaptationMode` | DATAS (.NET 8+) | 1 (.NET 9 default) |
| `DOTNET_GCHeapCount` | Принудительное число heaps | auto |
| `DOTNET_TieredCompilation` | Tiered JIT | 1 |
| `DOTNET_TieredPGO` | Dynamic PGO | 1 (.NET 8+) |
| `DOTNET_ReadyToRun` | R2R precompiled | 1 |
| `DOTNET_ThreadPool_MinThreads` | Минимум threads в pool | auto (по CPU) |
| `DOTNET_ThreadPool_MaxThreads` | Максимум threads в pool | auto (very high) |
| `DOTNET_SYSTEM_GLOBALIZATION_INVARIANT` | Без ICU | 0 |
| `DOTNET_ENVIRONMENT` | Env name (Generic Host) | Production |
| `DOTNET_NOLOGO` | Скрыть .NET CLI welcome | unset |
| `DOTNET_CLI_TELEMETRY_OPTOUT` | Отключить telemetry | 0 |

### ASP.NET Core environment variables

| Переменная | Что | Default |
|------------|-----|---------|
| `ASPNETCORE_ENVIRONMENT` | Env name | Production |
| `ASPNETCORE_URLS` | Listening URLs | http://localhost:5000 |
| `ASPNETCORE_HTTP_PORTS` | Только HTTP ports (.NET 8+) | unset |
| `ASPNETCORE_HTTPS_PORTS` | Только HTTPS ports | unset |
| `ASPNETCORE_FORWARDEDHEADERS_ENABLED` | Trust X-Forwarded-* | false |
| `ASPNETCORE_DETAILEDERRORS` | Show detailed errors | false |
| `ASPNETCORE_LOGGING__CONSOLE__DISABLECOLORS` | No colors | false |

```dockerfile
ENV ASPNETCORE_ENVIRONMENT=Production
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_FORWARDEDHEADERS_ENABLED=true
ENV DOTNET_NOLOGO=true
ENV DOTNET_RUNNING_IN_CONTAINER=true
```

---

## docker-compose для local dev

### Базовый stack

```yaml
# docker-compose.yml
services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      ASPNETCORE_ENVIRONMENT: Development
      ConnectionStrings__Default: "Host=postgres;Database=app;Username=app;Password=secret"
      Redis__Connection: "redis:6379"
      RabbitMq__Host: "rabbitmq"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
      rabbitmq:
        condition: service_healthy
    restart: unless-stopped
    init: true  # PID 1 protection — tini

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped

  rabbitmq:
    image: rabbitmq:3-management-alpine
    environment:
      RABBITMQ_DEFAULT_USER: app
      RABBITMQ_DEFAULT_PASS: secret
    ports:
      - "5672:5672"
      - "15672:15672"
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
    restart: unless-stopped

volumes:
  postgres-data:
  redis-data:

networks:
  default:
    name: myapp-network
```

```bash
docker compose up -d
docker compose logs -f api
docker compose restart api
docker compose down
docker compose down -v  # + volumes
```

### Override для разных environments

```yaml
# docker-compose.override.yml — auto-applied для dev
services:
  api:
    volumes:
      - ./:/src
    environment:
      DOTNET_USE_POLLING_FILE_WATCHER: "true"
    command: dotnet watch --project src/MyApp.Api/MyApp.Api.csproj run --urls http://+:8080
```

```yaml
# docker-compose.prod.yml
services:
  api:
    image: myapp/api:${TAG:-latest}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
```

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### docker-compose watch (Compose v2.22+)

Замена volumes mount для hot-reload:

```yaml
services:
  api:
    build: .
    develop:
      watch:
        - action: sync
          path: ./src
          target: /src
          ignore:
            - "**/bin"
            - "**/obj"
        - action: rebuild
          path: ./src/MyApp.Api/MyApp.Api.csproj
```

```bash
docker compose watch
```

При изменении `.cs` — sync (быстро). При изменении `.csproj` — rebuild (медленно, но нужно).

### profiles — селективный запуск

```yaml
services:
  api:
    profiles: [default]
  test-runner:
    profiles: [testing]
  load-tester:
    profiles: [perf]
```

```bash
docker compose up                          # default
docker compose --profile testing up        # + test-runner
COMPOSE_PROFILES=testing,perf docker compose up
```

---

## Healthchecks

### В Dockerfile

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health/live || exit 1
```

| Param | Default | Что |
|-------|---------|-----|
| `--interval` | 30s | Как часто проверять |
| `--timeout` | 30s | Сколько ждать ответа |
| `--start-period` | 0s | Grace period после старта (для долгого initialization) |
| `--retries` | 3 | После N failed → unhealthy |

### В chiseled images — нет wget/curl

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0-chiseled

# ❌ Не работает — wget не существует
HEALTHCHECK CMD wget ...

# ✅ Вариант 1: использовать .NET бинарник для healthcheck
HEALTHCHECK CMD ["/app/MyApp.Api", "--health-check"]

# ✅ Вариант 2: HEALTHCHECK NONE + использовать k8s/compose probe
HEALTHCHECK NONE
```

```csharp
// Program.cs — добавить health-check режим
if (args.Contains("--health-check"))
{
    using var client = new HttpClient();
    try
    {
        var response = await client.GetAsync("http://localhost:8080/health/live");
        return response.IsSuccessStatusCode ? 0 : 1;
    }
    catch { return 1; }
}
```

### В docker-compose

```yaml
services:
  api:
    healthcheck:
      test: ["CMD", "wget", "-q", "--tries=1", "--spider", "http://localhost:8080/health/live"]
      interval: 30s
      timeout: 3s
      start_period: 10s
      retries: 3
```

`depends_on: condition: service_healthy` ждёт пока healthcheck зеленый перед стартом dependent сервиса.

### ASP.NET Core endpoints

```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(connStr, tags: ["db", "ready"])
    .AddRedis(redisConnection, tags: ["cache", "ready"])
    .AddRabbitMQ(rabbitConn, tags: ["broker", "ready"]);

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // только app живёт
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

`/health/live` — для Docker/K8s liveness (рестартить при fail).
`/health/ready` — для readiness (не слать traffic если БД недоступна).

См. [[observability|Observability]] — детальный setup health checks.

---

## Resource limits

Без лимитов один pod / контейнер может съесть всё, остальные страдают.

```yaml
# docker-compose
services:
  api:
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

**limits** — hard cap, контейнер прибьется при exceed
**reservations** — guaranteed minimum

```bash
# Kubernetes аналог
resources:
  limits:
    cpu: "1"
    memory: "512Mi"
  requests:
    cpu: "250m"
    memory: "256Mi"
```

### .NET internal awareness и tuning

.NET Core 3.0+ читает container limits через `/sys/fs/cgroup/...`. ServerGC, ThreadPool автоматически адаптируется. Можно явно:

```dockerfile
# Override CPU detection (редко надо)
ENV DOTNET_PROCESSOR_COUNT=2

# Жёсткий heap limit (если cgroup memory limit плох)
ENV DOTNET_GCHeapHardLimit=200000000      # 200 MB hard cap
ENV DOTNET_GCHeapHardLimitPercent=75      # или 75% от cgroup memory

# Heap count override (для DATAS — сколько максимум heaps на ServerGC)
ENV DOTNET_GCHeapCount=2

# Включить DATAS явно (default в .NET 9+)
ENV DOTNET_GCDynamicAdaptationMode=1

# ThreadPool minimum (минимизирует cold start latency)
ENV DOTNET_ThreadPool_MinThreads=4
```

### CPU shares vs CPU quota

| | CPU shares (relative) | CPU quota (absolute) |
|--|----------------------|----------------------|
| Compose | `cpu_shares: 1024` | `cpus: '0.5'` |
| Kubernetes | `requests.cpu` | `limits.cpu` |
| Cgroup | `cpu.shares` | `cpu.cfs_quota_us` |
| Поведение | Делит CPU при contention | Hard cap |

**CPU throttling** — когда контейнер достиг quota, он замораживается до конца period (default 100ms). Это **частая причина latency spikes** в .NET: GC во время throttling выглядит как очень долгая пауза.

```bash
# Проверить throttling
cat /sys/fs/cgroup/cpu.stat
# nr_throttled — сколько раз заthrottled
# throttled_time — наносекунд во throttling

```

**Совет:** установи `requests.cpu` достаточно высоким (или используй `Burstable`/`Guaranteed` QoS в k8s) чтобы избежать throttling.

---

## Volumes — три типа

### 1. Named volumes — managed Docker

```yaml
services:
  postgres:
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
    driver: local
```

- Хранится в `/var/lib/docker/volumes/postgres-data/_data`
- Сохраняется между restart'ами
- Управляется Docker'ом
- **Default для production**

### 2. Bind mounts — host directory

```yaml
services:
  api:
    volumes:
      - ./src:/src                    # rw
      - ./config.json:/app/config.json:ro
      - $HOME/.aws:/root/.aws:ro
```

- Прямой mount хостовой директории
- Для dev (hot reload), не для production
- Работает с правами host filesystem

### 3. tmpfs — RAM filesystem

```yaml
services:
  api:
    tmpfs:
      - /tmp:rw,size=64m,mode=1777
```

- В RAM, не на диск
- Для temp файлов в read-only filesystem
- Очищается при restart

### Volume drivers

| Driver | Где |
|--------|-----|
| `local` | Local FS (default) |
| `nfs` | NFS server |
| `cifs` | SMB/CIFS share |
| `aws-efs` | AWS EFS plugin |
| `gcePersistentDisk` | GCP |

---

## Networking

### Network drivers

| Driver | Описание | Когда |
|--------|----------|-------|
| `bridge` | Default — virtual switch на хосте | Single-host |
| `host` | Без изоляции, container = host | Performance, нет network namespace |
| `overlay` | Multi-host через VXLAN | Swarm, K8s |
| `macvlan` | Container получает MAC адрес из physical network | Legacy apps требующие L2 |
| `ipvlan` | Аналогично, IP-level | Аналогично |
| `none` | Без сети | Полная изоляция |
| `container:<name>` | Использовать namespace другого container | Sidecar pattern |

### Default bridge — `docker0`

Без `network:` каждый контейнер подключён к default bridge `docker0`. Можно communicate by **IP**, но **не by name** (DNS не работает).

### User-defined bridge — DNS работает

```yaml
networks:
  myapp-network:
    driver: bridge
```

```yaml
services:
  api:
    networks: [myapp-network]
  postgres:
    networks: [myapp-network]
```

Теперь `api` может ходить в `postgres` (DNS resolves to container IP). Это **default в docker-compose**.

### Multiple networks (segmentation)

```yaml
services:
  api:
    networks:
      - frontend
      - backend
  postgres:
    networks: [backend]
  nginx:
    networks: [frontend]

networks:
  frontend:
  backend:
    internal: true  # без доступа в интернет
```

`internal: true` — контейнеры в этой сети не имеют outbound internet → защита БД от exfiltration.

### Port mapping

```yaml
ports:
  - "8080:8080"               # host:container
  - "127.0.0.1:8080:8080"     # bind только на localhost
  - "8080:8080/udp"           # UDP
  - "9000-9010:9000-9010"     # range
```

**Important:** `ports` — это `iptables DNAT` правила. Performance penalty небольшая, но есть. Для max throughput — `network: host`.

### Container DNS

Docker запускает встроенный DNS server в каждом user-defined network. Container hostname = service name.

```bash
# Внутри api container
nslookup postgres  # резолвится в IP postgres container
ping rabbitmq      # работает
```

---

## Multi-platform builds

```bash
# Создать builder с QEMU emulation
docker buildx create --name multi-platform --use
docker buildx inspect --bootstrap

# Build для нескольких архитектур
docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t myorg/myapp:latest \
    --push .
```

ARM build нужен для:
- Raspberry Pi deployment
- AWS Graviton (~1.5x дешевле x86)
- Apple Silicon (M1/M2/M3 Macs developing)
- Azure Ampere (Arm64 VMs)

### .NET specific — Arm64 поддержка

.NET 8+ имеет first-class Arm64 support. JIT, AOT, R2R — все работают. Кросс-компиляция:

```bash
# На amd64 build для arm64
dotnet publish -r linux-arm64 -c Release --self-contained
```

Внутри Dockerfile:
```dockerfile
ARG TARGETARCH
RUN dotnet publish -r linux-${TARGETARCH} ...
```

`TARGETARCH` — встроенная BuildKit переменная (amd64, arm64, arm).

---

## SDK container builds — dotnet publish без Dockerfile

.NET SDK умеет собирать OCI-image сам — таргет `PublishContainer` (встроен в SDK с .NET 8, для web-проектов — с .NET 7). Механизм: SDK уже знает всё, что руками пишут в Dockerfile — TFM, RID, entry point, порты; он берёт правильный базовый образ (`aspnet` / `runtime` / `runtime-deps` для AOT), кладёт publish-выход слоем поверх и пишет image прямо в локальный Docker daemon или registry. Docker при push в registry вообще не нужен.

```bash
# Локальный image в Docker daemon
dotnet publish /t:PublishContainer -p ContainerRepository=my-app -p ContainerImageTags=1.2.0

# Сразу push в registry (без Docker daemon)
dotnet publish /t:PublishContainer \
    -p ContainerRepository=myteam/my-app \
    -p ContainerImageTags='"1.2.0;latest"' \
    -p ContainerRegistry=ghcr.io
```

Настройки — обычные MSBuild-свойства в csproj:

```xml
<PropertyGroup>
  <ContainerRepository>my-app</ContainerRepository>
  <ContainerImageTags>1.2.0;latest</ContainerImageTags>
  <!-- базовый образ подменяется при необходимости -->
  <ContainerBaseImage>mcr.microsoft.com/dotnet/aspnet:10.0-chiseled</ContainerBaseImage>
</PropertyGroup>
```

Ключевое про security: **rootless по умолчанию** — сгенерированный образ запускается под `USER $APP_UID` (1654), не под root; порт — 8080. То, что в Dockerfile настраивают руками, здесь дефолт.

### Когда достаточно vs когда Dockerfile

| SDK container build достаточно | Нужен Dockerfile |
|-------------------------------|------------------|
| Стандартное ASP.NET Core / worker приложение | `apt-get install` нативных зависимостей (ffmpeg, ICU-кастом) |
| CI-конвейер «build → push → deploy» | Кастомные слои, файлы вне publish-выхода |
| Не хочется поддерживать Dockerfile-копипасту | Множественные custom stages (тесты внутри build, генерация клиентов) |
| Быстрый переход на chiseled (`ContainerBaseImage`) | Нестандартный ENTRYPOINT / init-скрипты |

Правило: начинай с `PublishContainer`; Dockerfile — когда понадобился шаг, которого нет среди `Container*`-свойств. Junior-версия — [[docker-for-dev|Docker для разработчика]] §6.1.

---

## Security best practices

### 1. Non-root user

```dockerfile
RUN addgroup -S app && adduser -S app -G app
USER app
```

Если container compromised → attacker не имеет root access. Defense-in-depth.

Во всех .NET 8+ images уже есть user `app` (UID 1654) — просто `USER app` или переносимый вариант `USER $APP_UID` (env-переменная задана в образе; то же число идёт в k8s `runAsUser: 1654`). Chiseled-образы под ним запускаются по умолчанию.

### 2. Read-only filesystem

```yaml
services:
  api:
    read_only: true
    tmpfs:
      - /tmp:rw,size=64m
```

Container не может писать на disk → защита от tampering.

В .NET для read-only:
```dockerfile
# Где .NET может писать
ENV TMPDIR=/tmp
ENV ASPNETCORE_TEMP=/tmp
```

### 3. Drop capabilities

Linux capabilities — гранулярные привилегии (раньше всё было либо root либо нет).

```yaml
services:
  api:
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # если нужен port < 1024
```

Полный список capabilities: `man capabilities(7)`.

| Capability | Что | Опасность |
|------------|-----|-----------|
| `CAP_NET_BIND_SERVICE` | Bind на port < 1024 | Низкая |
| `CAP_NET_ADMIN` | Network configuration | Высокая |
| `CAP_SYS_ADMIN` | "swiss army knife" — много опасного | Очень высокая |
| `CAP_DAC_OVERRIDE` | Игнорировать file permissions | Высокая |
| `CAP_SYS_PTRACE` | Debugging других процессов | Средняя |

### 4. seccomp profile

seccomp — фильтрует syscalls. Default Docker profile блокирует ~50 опасных syscalls (kexec_load, init_module).

```yaml
services:
  api:
    security_opt:
      - seccomp:./seccomp-profile.json
      - no-new-privileges:true  # запрет setuid escalation
```

### 5. AppArmor / SELinux

```yaml
services:
  api:
    security_opt:
      - apparmor:my-profile
```

Mandatory Access Control — дополнительный layer защиты.

### 6. User namespaces remapping

```json
// /etc/docker/daemon.json
{
  "userns-remap": "default"
}
```

`uid 0` (root) внутри контейнера → `uid 100000` (или другой) на host. Если container escape — attacker всё ещё не root на хосте.

### 7. Image scanning

```bash
# Trivy — open-source vulnerability scanner
trivy image myorg/myapp:latest

# Snyk
snyk container test myorg/myapp:latest

# Docker Scout
docker scout cves myorg/myapp:latest

# Grype
grype myorg/myapp:latest
```

Run в CI — fail если CRITICAL CVEs.

### 8. Image signing — supply chain security

```bash
# Sigstore Cosign — keyless signing с OIDC
cosign sign myorg/myapp:latest

# Verify
cosign verify myorg/myapp:latest \
    --certificate-identity-regexp ".*" \
    --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

В k8s — admission controllers (Connaisseur, Kyverno) могут проверять signatures перед deployment.

### 9. SBOM (Software Bill of Materials)

```bash
docker buildx build --sbom=true --push -t myorg/app .

# Или syft
syft myorg/app:latest -o spdx-json > sbom.json
```

Список всех packages в image — для compliance / audit. SBOM в SPDX или CycloneDX format.

### 10. Provenance

```bash
docker buildx build --provenance=true --push -t myorg/app .
```

Метаданные о том, **как** image был собран: source repo, commit SHA, build environment. SLSA framework требует provenance levels 1-4.

### 11. Rootless Docker

Docker daemon обычно работает как root → если daemon compromised → root на host. Rootless Docker запускает daemon как обычный user.

```bash
# Установка rootless
dockerd-rootless-setuptool.sh install

# Использование
export DOCKER_HOST=unix:///run/user/$UID/docker.sock
docker info
```

**Альтернатива: Podman** — daemonless, rootless из коробки, drop-in replacement для docker CLI.

---

## Reproducible builds

Цель — два build'а одного source кода → побитово одинаковые image (одинаковый digest). Зачем: supply chain trust, audit, кеширование.

```dockerfile
# Зафиксировать timestamps
ARG SOURCE_DATE_EPOCH
ENV SOURCE_DATE_EPOCH=${SOURCE_DATE_EPOCH}

FROM mcr.microsoft.com/dotnet/sdk:10.0@sha256:abc...  # pinned by digest, не tag
```

```bash
# Build с фиксированной датой
SOURCE_DATE_EPOCH=$(git log -1 --format=%ct) \
docker buildx build \
    --output=type=image,rewrite-timestamp=true \
    -t myapp:repro .
```

`--output type=image,rewrite-timestamp=true` (BuildKit 0.13+) — убирает все timestamps в layer'ах.

---

## Production deployment patterns

### Build → push → deploy

```yaml
# .github/workflows/release.yml
- name: Build and push
  uses: docker/build-push-action@v5
  with:
    context: .
    file: ./Dockerfile
    push: true
    tags: |
      myorg/myapp:${{ github.sha }}
      myorg/myapp:latest
    cache-from: type=gha
    cache-to: type=gha,mode=max
    platforms: linux/amd64,linux/arm64
    sbom: true
    provenance: true

- name: Sign image
  run: cosign sign --yes myorg/myapp:${{ github.sha }}

- name: Deploy
  run: |
    ssh prod "docker pull myorg/myapp:${{ github.sha }} && \
              docker compose -f docker-compose.prod.yml up -d"
```

### Rolling update (zero-downtime)

```bash
# Compose
docker compose up -d --no-deps --scale api=2 api
sleep 30  # let new container become healthy
docker compose up -d --no-deps api  # remove old
```

В Kubernetes — `kubectl rollout` делает это автоматически. См. [[kubernetes|Kubernetes]].

### Blue-Green

```bash
# Старая версия — blue (running)
# Новая — green (deploy)
docker compose -p green up -d
curl https://green.staging.example.com/health
# Switch DNS / load balancer
docker compose -p blue down
```

---

## Container registries

| Registry | Hosting | Особенности |
|----------|---------|-------------|
| **Docker Hub** | Cloud | Pull rate limits для anonymous |
| **GitHub Container Registry (ghcr.io)** | GitHub | Free для public, integrated с GHA |
| **AWS ECR** | AWS | IAM-based, integrated с EKS/ECS |
| **Azure Container Registry (ACR)** | Azure | Integrated с AKS, Azure AD auth |
| **Google Artifact Registry** | GCP | Integrated с GKE |
| **Harbor** | Self-hosted | Open-source, vulnerability scanning, replication |
| **Quay** | Red Hat | Vulnerability scanning, mirror |
| **Distribution (CNCF)** | Self-hosted | Reference open-source registry |

### Pull-through cache / mirror

В корпоративной среде нельзя пускать nodes в Docker Hub напрямую (rate limits, security). Решение — registry mirror:

```json
// /etc/docker/daemon.json
{
  "registry-mirrors": ["https://my-internal-mirror.corp.local"]
}
```

Harbor / Distribution могут работать как proxy cache.

### Pull secrets в Kubernetes

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: registry-creds
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64>

---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      imagePullSecrets:
        - name: registry-creds
```

---

## .NET Aspire — orchestration для разработки

**.NET Aspire** — фреймворк от Microsoft для **local development** распределённых систем. Не replacement для Kubernetes — это **dev experience**.

```csharp
// AppHost/Program.cs
var builder = DistributedApplication.CreateBuilder(args);

var postgres = builder.AddPostgres("db")
    .WithDataVolume()
    .AddDatabase("appdb");

var redis = builder.AddRedis("cache");

var rabbit = builder.AddRabbitMQ("rabbit")
    .WithManagementPlugin();

var api = builder.AddProject<Projects.MyApp_Api>("api")
    .WithReference(postgres)
    .WithReference(redis)
    .WithReference(rabbit);

builder.AddProject<Projects.MyApp_Web>("web")
    .WithReference(api);

builder.Build().Run();
```

`dotnet run` в AppHost project — запускает:
- Postgres в Docker
- Redis в Docker
- RabbitMQ в Docker
- API проект (нативно или в Docker)
- Web проект
- **Aspire Dashboard** — UI с logs, traces, metrics всех компонентов

### Что Aspire даёт

- Автоматическая discovery (через env vars `services__db__connectionstring`)
- Автоматический OpenTelemetry — все services emit'ят traces в Aspire dashboard
- Health checks UI
- Service composition в C# (typed, refactor-safe)

### Aspire vs Compose

| | Aspire | docker-compose |
|--|--------|----------------|
| Конфиг | C# | YAML |
| Type safety | ✅ | ❌ |
| Production deployment | Не для prod | Можно для prod |
| Observability built-in | ✅ Dashboard | Нужен Grafana |
| Learning curve | Familiar для .NET dev | Шире применимо |

**Когда что:** Aspire — для dev времени .NET solution с 3+ services. Compose — для prod, multi-language, и dev если команда mixed.

---

## Devcontainers — VS Code dev environments

`.devcontainer/devcontainer.json` — стандарт от Microsoft для контейнеризованной dev среды.

```json
{
  "name": "MyApp",
  "image": "mcr.microsoft.com/dotnet/sdk:10.0",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {},
    "ghcr.io/devcontainers/features/azure-cli:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-dotnettools.csdevkit",
        "ms-azuretools.vscode-docker"
      ],
      "settings": {
        "dotnet.preferCSharpExtension": true
      }
    }
  },
  "postCreateCommand": "dotnet restore",
  "forwardPorts": [8080],
  "runArgs": ["--init"]
}
```

VS Code (или GitHub Codespaces) запускает контейнер, прокидывает source через volume, запускает remote VS Code server внутри. Получаешь identical dev env на любой машине, включая browser-based Codespaces.

### Compose-based devcontainer

```json
{
  "name": "MyApp",
  "dockerComposeFile": "../docker-compose.yml",
  "service": "api",
  "workspaceFolder": "/src",
  "features": { ... }
}
```

Использует docker-compose stack как dev среду. Postgres/Redis уже запущены.

---

## dotnet-monitor — production diagnostics в контейнере

**dotnet-monitor** — diagnostic sidecar от Microsoft для production. Endpoints для:
- GC dumps (heap snapshots)
- Process dumps (full memory)
- Logs streaming
- Live metrics
- Distributed traces

### Sidecar setup

```yaml
# docker-compose
services:
  api:
    image: myapp/api
    ports:
      - "8080:8080"
    volumes:
      - diagnostic-pipes:/tmp
    environment:
      DOTNET_DiagnosticPorts: /tmp/dotnet-monitor.sock,suspend

  monitor:
    image: mcr.microsoft.com/dotnet/monitor:8
    ports:
      - "52323:52323"  # API endpoint
      - "52325:52325"  # metrics (Prometheus)
    volumes:
      - diagnostic-pipes:/tmp
    environment:
      DOTNETMONITOR_DiagnosticPort__ConnectionMode: Listen
      DOTNETMONITOR_DiagnosticPort__EndpointName: /tmp/dotnet-monitor.sock

volumes:
  diagnostic-pipes:
```

```bash
# Получить heap dump через monitor
curl http://localhost:52323/dump?type=Mini > heap.dmp

# GC dump
curl http://localhost:52323/gcdump > app.gcdump

# Live logs
curl http://localhost:52323/logs?level=Warning
```

В Kubernetes — это sidecar контейнер в том же pod.

### Альтернатива: dotnet diagnostic tools в контейнере

```bash
# Войти в running container
docker exec -it api bash

# Установить tools
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-dump

# Использовать
dotnet-trace collect -p $(pidof dotnet)
```

В chiseled нет shell → используй sidecar или ephemeral container.

---

## Trimming, AOT и оптимизация размера для контейнеров

### Trimming — удалить неиспользуемые сборки

```xml
<PropertyGroup>
  <PublishTrimmed>true</PublishTrimmed>
  <TrimMode>full</TrimMode>
</PropertyGroup>
```

```dockerfile
RUN dotnet publish -c Release -o /app /p:PublishTrimmed=true /p:TrimMode=full
```

Размер: 220 MB → 60-80 MB. **Ломает reflection**, нужны annotations. См. [[native-aot|Native AOT]].

### Native AOT — полная компиляция в native

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

Размер: 60 MB → 20-25 MB. + быстрый cold start (10x), низкая память (3x). Подробно: [[native-aot|Native AOT]].

### ReadyToRun (R2R) — pre-compiled IL → native

```xml
<PropertyGroup>
  <PublishReadyToRun>true</PublishReadyToRun>
  <PublishReadyToRunComposite>true</PublishReadyToRunComposite>
</PropertyGroup>
```

Не уменьшает размер (даже увеличивает чуть-чуть), но **дает faster startup** — JIT не компилирует с нуля. Default для Microsoft images.

### PGO в контейнерах

**Profile-Guided Optimization** — статистика runtime'а помогает JIT оптимизировать hot paths.

- **Static PGO** — сбор профиля во время testing → publish с профилем
- **Dynamic PGO** — runtime собирает и применяет (default .NET 8+)

```xml
<PropertyGroup>
  <TieredPGO>true</TieredPGO>  <!-- default .NET 8+ -->
</PropertyGroup>
```

В containers: автоматически работает, ничего не надо.

---

## Container runtime: что под капотом

Docker — это **client + daemon** (containerd) + **runtime** (runc).

```
docker CLI → dockerd → containerd → runc → namespace + cgroup setup → процесс
```

### Альтернативные runtimes

| Runtime | Описание | Когда |
|---------|----------|-------|
| **runc** | Default, OCI-compliant | Default |
| **crun** | Faster runc на C | Performance |
| **gVisor (runsc)** | Userspace kernel — sandboxing | Multi-tenant security |
| **Kata Containers** | Lightweight VM per container | Untrusted code |
| **Firecracker** | AWS Lambda's runtime, microVM | Serverless |

### CRI runtimes (Kubernetes)

Kubernetes использует **CRI** (Container Runtime Interface), а не Docker:

- **containerd** (default) — directly от Docker stack без daemon
- **CRI-O** — Red Hat's runtime для OpenShift
- ~~**Docker (deprecated)**~~ — был removed in K8s 1.24

Поэтому **Docker** — это inner-loop dev tool. **Production runtime** — containerd / CRI-O.

---

## Common pitfalls

### 1. `latest` tag в production

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:latest  # ❌
```
"Latest" меняется → reproducible build broken.
**Решение:** pinned version или digest:
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0.1-noble-chiseled
# или
FROM mcr.microsoft.com/dotnet/aspnet@sha256:abc...
```

### 2. SDK image в production

```dockerfile
FROM dotnet/sdk:10.0  # ❌ 1 GB
```
Production не нужен SDK / compiler. Multi-stage с `aspnet` final.

### 3. Run as root
По дефолту `root` user. Compromise → root → host (через volumes).
**Решение:** non-root user или chiseled (там уже).

### 4. Volumes без backup
Postgres data в named volume → исчезает при `docker compose down -v`.
**Решение:** `pg_dump` per day to S3.

### 5. .env с secrets в git
**Решение:** `.env` в `.gitignore`, `.env.example` с placeholder'ами. В prod — secrets manager (Infisical, Vault, AWS Secrets Manager).

### 6. No logging driver rotation
Default `json-file` накапливает всё в /var/lib/docker без ротации → disk full.
**Решение:**
```yaml
services:
  api:
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
```

### 7. `docker exec` для исправлений в production
Hot-fix через `docker exec` — теряется при restart. State degrades.
**Решение:** rebuild image, redeploy.

### 8. Ignore healthchecks
"Healthy" status = что-то работает. Не настроен — оrchestrator не знает падает ли container.

### 9. Same image для всех environments без env-config
Dev / staging / prod нуждаются в разных configs. Container должен быть **same**, config — через env vars (12-factor).

### 10. Не очищать old images
`docker images` накапливает GBs неиспользуемых tags.
```bash
docker image prune -a -f --filter "until=720h"  # старше 30 дней
docker system prune -a --volumes  # nuke everything (CAREFUL)
```

### 11. Запуск .NET 5/6 в .NET 8/9 manifest mode
.NET 8/9 имеют DATAS/regions, влияющие на memory profile. Если копируешь старый Dockerfile с .NET 5 пользовательскими `DOTNET_GC*` env — могут конфликтовать. **Тестируй после апгрейда.**

### 12. `dotnet watch` без polling в Docker volume
File watcher через inotify не работает на bind mount от Docker Desktop (Mac/Windows).
```yaml
environment:
  DOTNET_USE_POLLING_FILE_WATCHER: "true"
```

### 13. Незакрытые TCP connections при shutdown
Без `terminationGracePeriodSeconds` k8s убивает pod через 30s SIGKILL → недоставленные responses.
**Решение:** в-flight request draining + pre-stop sleep + IHostApplicationLifetime. См. [[kubernetes|Kubernetes]].

### 14. Heavy reflection в trim'нутом приложении
PublishTrimmed=true ломает reflection. Без `TrimAnalyzer` warnings → runtime crashes.
**Решение:** включить trim warnings as errors, использовать `JsonSerializerContext` для AOT.

### 15. Unicode без `chiseled-extra`
Базовый chiseled не имеет ICU. `CultureInfo.GetCultureInfo("ru-RU")` → InvariantCulture.
**Решение:** `chiseled-extra` или `DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1` (если ОК работать в invariant).

### 16. Build не reproducible из-за `apt-get update`
Каждый раз `apt-get` — разная версия packages → image другой.
**Решение:** pinned versions, или скипай `apt-get` (chiseled images).

### 17. CPU limits ниже Server GC heap count
Если у тебя `cpus: 0.25` и Server GC default — он создаст 1 heap (округление вверх), но GC pauses всё равно будут throttled. Используй DATAS (.NET 9+).

### 18. Layer ordering breaks cache
Любой `COPY .` перед `dotnet restore` → restore при каждом изменении кода.
**Решение:** csproj первым, `dotnet restore`, потом весь source.

---

## Production checklist

- [ ] Multi-stage build (sdk → aspnet/runtime-deps)
- [ ] `.dockerignore` исключает bin/obj/source-control
- [ ] Non-root user (или chiseled)
- [ ] Pinned base image versions (not `latest`)
- [ ] **Pinned base by digest** для immutable supply chain
- [ ] Healthchecks в Dockerfile + docker-compose/k8s
- [ ] Resource limits (CPU + memory) в orchestrator
- [ ] Logging driver с ротацией (json-file max-size + max-file)
- [ ] BuildKit включён (default 23+)
- [ ] Layer caching через restore separate from build
- [ ] Secrets через `--mount=type=secret`, не ARG/ENV
- [ ] Image vulnerability scanning в CI (Trivy / Snyk / Scout)
- [ ] Multi-platform builds (amd64 + arm64)
- [ ] SBOM generation для compliance
- [ ] Provenance metadata
- [ ] Image signing (cosign)
- [ ] Read-only filesystem где возможно
- [ ] Capability dropping (cap_drop ALL, cap_add только нужное)
- [ ] seccomp profile (default Docker — OK)
- [ ] no-new-privileges: true
- [ ] env через environment variables, не in image
- [ ] Graceful shutdown работает (SIGTERM → drain → exit)
- [ ] `init: true` или `tini` если запускаешь child processes
- [ ] preStop hook + terminationGracePeriodSeconds в k8s
- [ ] DATAS включён (.NET 9+ default, .NET 8 — opt-in)
- [ ] DOTNET_GCHeapHardLimitPercent если строгий memory limit
- [ ] dotnet-monitor sidecar для production diagnostics
- [ ] Сделан registry mirror для anonymous Docker Hub pull rate limits
- [ ] CI/CD pipeline pushes by SHA tag (myapp:abc123), не только latest

---

## Case Studies

### Case Study #1 — Postgres + .NET — production setup

**Сценарий:** ASP.NET Core API в Docker, Postgres в Docker, EF Core. 

```dockerfile
# Dockerfile (multi-stage)
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

```yaml
# docker-compose.yml
services:
  api:
    build: .
    ports: ["5000:8080"]
    environment:
      - ConnectionStrings__Default=Host=db;Database=myapp;Username=postgres;Password=secret
    depends_on:
      db:
        condition: service_healthy
  
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
volumes:
  pgdata:
```

---

### Case Study #2 — Multi-stage build (smaller image)

**❌ Single-stage — 2.3 GB image:**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0
WORKDIR /app
COPY . .
RUN dotnet publish -c Release -o /publish
ENTRYPOINT ["dotnet", "/publish/MyApp.dll"]
```

**✅ Multi-stage — 220 MB:**
```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj .
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app --no-restore

# Runtime stage (smaller!)
FROM mcr.microsoft.com/dotnet/aspnet:10.0
WORKDIR /app
COPY --from=build /app .
USER $APP_UID  # non-root!
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

**Result:** 2.3 GB → 220 MB (10x smaller).

**Native AOT — 50 MB:**
```dockerfile
FROM mcr.microsoft.com/dotnet/runtime-deps:10.0
COPY publish/ /app/
ENTRYPOINT ["/app/MyApp"]
# AOT — нет JIT, native binary, ~50 MB total
```

---

### Case Study #3 — Layer caching для fast rebuilds

**❌ Naive — каждое изменение → restore packages заново (3 min):**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet restore  # changes если ANY file changed!
RUN dotnet publish -c Release
```

**✅ Smart layering — restore cached если csproj не изменился:**
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.sln .
COPY src/MyApp.Api/*.csproj ./src/MyApp.Api/
COPY src/MyApp.Domain/*.csproj ./src/MyApp.Domain/
RUN dotnet restore  # cached! только если csproj changed

COPY . .
RUN dotnet publish -c Release -o /app --no-restore
```

**Result:** code changes → 30 sec rebuild (не 3 min).

См. [[cicd-github-actions|CI/CD]] и[[native-aot|Native AOT]].


---

## Cheat sheet

| Need | Command / Pattern |
|------|-------------------|
| Build image | `docker build -t myapp:1.0 .` |
| Run container | `docker run -p 5000:8080 myapp:1.0` |
| Run detached | `docker run -d --name api myapp:1.0` |
| List containers | `docker ps` (running), `docker ps -a` (all) |
| Container logs | `docker logs -f api` |
| Exec into container | `docker exec -it api bash` |
| Stop container | `docker stop api` |
| Remove container | `docker rm api` |
| Remove image | `docker rmi myapp:1.0` |
| Compose up | `docker compose up -d` |
| Compose logs | `docker compose logs -f api` |
| Compose down | `docker compose down -v` (с volumes) |
| Build no cache | `docker build --no-cache -t myapp .` |
| Tag image | `docker tag myapp:latest registry/myapp:1.0` |
| Push to registry | `docker push registry/myapp:1.0` |
| Pull image | `docker pull registry/myapp:1.0` |
| Disk usage | `docker system df` |
| Cleanup | `docker system prune -a` |

**Dockerfile best practices:**
- Multi-stage builds (build → runtime)
- `.dockerignore` обязательно (`bin/`, `obj/`, `node_modules/`)
- `COPY *.csproj` first для layer caching
- `USER $APP_UID` (non-root)
- HEALTHCHECK для liveness
- Pin versions (`postgres:16` not `postgres:latest`)

**Image sizes (.NET 10):**
- `mcr.microsoft.com/dotnet/sdk:10.0` — 800 MB (build only!)
- `mcr.microsoft.com/dotnet/aspnet:10.0` — 220 MB (runtime, web)
- `mcr.microsoft.com/dotnet/runtime:10.0` — 190 MB (console)
- `mcr.microsoft.com/dotnet/runtime-deps:10.0` — 116 MB (для AOT)
- Native AOT compiled — 50-80 MB


---

## Decision tree

```
Какой Docker подход?
│
├── Какой base image?
│   ├── ASP.NET Web → mcr.microsoft.com/dotnet/aspnet:10.0
│   ├── Console / worker → mcr.microsoft.com/dotnet/runtime:10.0
│   ├── Native AOT → mcr.microsoft.com/dotnet/runtime-deps:10.0
│   ├── Самый маленький → Alpine variants (suffix `-alpine`)
│   └── Distroless → Google distroless для max security
│
├── Build approach?
│   ├── Multi-stage → ALWAYS (не один stage)
│   ├── Layer caching → COPY csproj first, restore, then code
│   └── BuildKit → DOCKER_BUILDKIT=1 (cache mounts, secrets)
│
├── Local development?
│   ├── docker-compose с .NET app → standard
│   ├── .NET Aspire → лучше Compose для .NET stack
│   └── DevContainers → VS Code + Docker
│
├── Production?
│   ├── Single host → docker-compose or systemd
│   ├── Cluster → Kubernetes (см. kubernetes.md)
│   └── Managed → Azure Container Apps / AWS ECS
│
└── Image size optimization?
    ├── < 100 MB → Native AOT + runtime-deps
    ├── < 250 MB → multi-stage + aspnet:10.0
    ├── < 50 MB → distroless + AOT (продвинутые)
    └── Bigger OK → standard runtime image
```


---

## См. также

- [[native-aot|Native AOT]] — distroless + AOT для extra-small images
- [[kubernetes|Kubernetes]] — orchestration, probes, graceful shutdown под .NET
- [[observability|Observability]] — health checks, metrics, OpenTelemetry в containerized
- [[resilience|Resilience]] — что делать когда container перезапустился
- [[project-setup|Project Setup]] — Directory.Build.props + Dockerfile setup
- [[testing|Testing]] — Testcontainers
- [[gc-memory|GC и память в контейнерах]] — DATAS, cgroup awareness, heap limits
- [[system-design|System Design]] — Docker как deployment layer
- [[hft-low-latency|HFT/Low-Latency]] — почему контейнеры могут давать latency spikes (CPU throttling)

## Reading list

- **Docker docs** — docs.docker.com
- **Microsoft Learn — .NET in containers** — learn.microsoft.com/dotnet/architecture/microservices/
- **Best practices for Dockerfiles** — docs.docker.com/develop/develop-images/dockerfile_best-practices/
- **BuildKit features** — docs.docker.com/build/buildkit/
- **Docker Security** — docs.docker.com/engine/security/
- **Trivy** — trivy.dev (vulnerability scanner)
- **Aaron Holt — chiseled containers** — devblogs.microsoft.com/dotnet/announcing-dotnet-chiseled-containers/
- **Aaron Holt — chiseled Ubuntu noble** — devblogs.microsoft.com/dotnet/dotnet-9-on-noble-chiseled/
- **12-Factor App** — 12factor.net (canonical для container-first apps)
- **Sigstore Cosign** — docs.sigstore.dev/cosign/overview/
- **OCI Image Spec** — github.com/opencontainers/image-spec
- **CGroups v2 design** — kernel.org/doc/Documentation/cgroup-v2.txt
- **Maoni Stephens — DATAS deep dive** — devblogs.microsoft.com/dotnet/dynamically-adapting-to-application-sizes/
- **.NET Aspire** — learn.microsoft.com/dotnet/aspire/
- **dotnet-monitor** — github.com/dotnet/dotnet-monitor
- **Devcontainers spec** — containers.dev
- **CNCF Landscape — Container Runtime** — landscape.cncf.io
