---
tags: [infrastructure, docker, junior, basics, containers]
level: Junior
date: 2026-08-02
---

# Docker для разработчика — практические basics

> **Что такое контейнеры, daily команды, Dockerfile основы, docker-compose для local dev.** Введение перед `Senior/docker.md` (production deep). Минимум который нужен для daily .NET dev.

---

## 0. Как читать

Если вообще не работал с Docker — раздел 1 (что это) → 2 (install) → 3 (daily commands). После — 4 (Dockerfile) → 5 (compose). Production multi-stage builds, optimization, security — `Senior/docker.md`.

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Что такое контейнер

**Контейнер** — изолированная среда с приложением и всеми зависимостями. Запускается на любой машине одинаково.

```
Без Docker:
- "У меня работает!" (но не у тебя — другая версия .NET / SQL)
- Развёртывание: установи .NET 10, MS SQL, Redis, настрой connection strings...
- Onboarding: 1-3 дня

С Docker:
- docker compose up
- Всё запускается одинаково везде
- Onboarding: 5 минут
```

### 1.2. Контейнер vs Виртуальная машина

```
VM:                              Container:
┌──────────────────┐             ┌──────────────────┐
│ App              │             │ App              │
│ Libs             │             │ Libs             │
│ ──────────────── │             │ ──────────────── │
│ Guest OS (Linux) │             │  (shared host OS)│
│ ──────────────── │             ├──────────────────┤
│ Hypervisor       │             │ Docker Engine    │
├──────────────────┤             ├──────────────────┤
│ Host OS          │             │ Host OS          │
└──────────────────┘             └──────────────────┘

VM:                  Container:
- ~GB RAM           - ~MB RAM
- ~minutes start    - ~seconds start
- Full OS           - Process isolation
- Heavy             - Lightweight
```

### 1.3. Image vs Container

```
Image — read-only template (как класс)
Container — running instance из image (как объект)

Из одного image можно запустить много containers.
```

```bash
# Pull image (download)
docker pull mcr.microsoft.com/dotnet/aspnet:10.0

# Run container из image
docker run mcr.microsoft.com/dotnet/aspnet:10.0
```

### 1.4. Зачем тебе как .NET dev

```
✅ Local dev environment:
- SQL Server / Postgres / Redis в Docker — нет install
- Switch между versions (PostgreSQL 14 vs 16)
- Reset database — docker rm + docker run = clean state

✅ Production parity:
- Same environment local → CI → prod
- Меньше "works on my machine" багов

✅ Microservices:
- Каждый сервис в свой container
- Compose orchestrates локально

✅ CI/CD:
- GitHub Actions / GitLab CI run в containers
- Test isolation
- Reproducible builds

❌ НЕ для всего:
- Маленький pet project — overhead
- Native desktop app (WPF/WinUI) — не нужно
```

> [!info]- Если ты приходил из Java / Python / Node.js
> Концепция та же. Java — `Dockerfile FROM openjdk:17`, Python — `FROM python:3.11`, .NET — `FROM mcr.microsoft.com/dotnet/aspnet:10.0`. Команды `docker run / build / compose` идентичны. Microsoft images — official aspnet/sdk/runtime variants.

> [!question]- Интервью: что такое Docker?
> Платформа для **containerization** — упаковки приложения в isolated lightweight environment. **Container** = process + filesystem + network namespace. Запускается одинаково на любой OS с Docker. **Use cases**: 1) Dev environment (Postgres/Redis local). 2) Production deployment (k8s). 3) CI/CD (build + test isolation). 4) Microservices. **Vs VM**: containers share host kernel, лёгкие (MB не GB), быстрый start (seconds не minutes). **.NET 2024+**: official Microsoft images (`mcr.microsoft.com/dotnet/*`), multi-arch (x64/ARM64), Linux + Windows.

---

## 2. Install

### 2.1. Windows / Mac

**Docker Desktop** — все-в-одном, simplest:
- https://www.docker.com/products/docker-desktop/
- Включает: Docker Engine + CLI + compose + GUI

```bash
# Verify install
docker --version
docker compose version
```

⚠️ Docker Desktop платный для крупных компаний (>250 employees / >$10M revenue). Для personal use / маленьких компаний — бесплатно.

### 2.2. Альтернативы Docker Desktop

```
Rancher Desktop:
- Free, open-source
- Замена Docker Desktop
- macOS / Windows / Linux

Podman Desktop:
- Free, daemonless
- Compatible с docker CLI

WSL2 + Docker Engine на Linux:
- Только для Windows
- Бесплатно
- Чуть больше setup
```

### 2.3. Linux

```bash
# Ubuntu / Debian
sudo apt install docker.io docker-compose-plugin

# Add user to docker group (avoid sudo)
sudo usermod -aG docker $USER
# Logout / login
```

### 2.4. Verification

```bash
# Запусти test container
docker run hello-world

# Output:
# "Hello from Docker!"
# This message shows that your installation appears to be working correctly.
```

---

## 3. Daily commands

### 3.1. Images

```bash
# Список images на машине
docker images

# Pull (download) image
docker pull postgres:16
docker pull mcr.microsoft.com/dotnet/aspnet:10.0

# Удалить image
docker rmi postgres:16

# Удалить unused images (cleanup)
docker image prune
docker image prune -a   # ВСЕ unused
```

### 3.2. Containers — run

```bash
# Run в foreground (видишь output)
docker run nginx

# Run в background (-d = detached)
docker run -d nginx

# С именем (легче управлять)
docker run -d --name my-postgres postgres:16

# Map port (host:container)
docker run -d --name web -p 8080:80 nginx
# Открой http://localhost:8080

# С переменными окружения
docker run -d --name pg -p 5432:5432 \
    -e POSTGRES_PASSWORD=secret \
    -e POSTGRES_DB=mydb \
    postgres:16

# С volume (persistent storage)
docker run -d --name pg \
    -v pgdata:/var/lib/postgresql/data \
    -p 5432:5432 \
    -e POSTGRES_PASSWORD=secret \
    postgres:16

# Auto-remove после stop
docker run --rm nginx
```

### 3.3. Containers — manage

```bash
# Список running containers
docker ps

# Список ВСЕХ (включая stopped)
docker ps -a

# Stop container
docker stop my-postgres

# Start (уже созданный)
docker start my-postgres

# Restart
docker restart my-postgres

# Удалить container (только stopped)
docker rm my-postgres

# Force-remove (даже running)
docker rm -f my-postgres

# Удалить ВСЕ stopped containers
docker container prune
```

### 3.4. Logs и debugging

```bash
# Logs container
docker logs my-postgres

# Follow logs (live)
docker logs -f my-postgres

# Last 100 lines
docker logs --tail 100 my-postgres

# С timestamps
docker logs -t my-postgres
```

### 3.5. Exec — войти в container

```bash
# Запустить bash внутри running container
docker exec -it my-postgres bash

# Запустить psql внутри Postgres container
docker exec -it my-postgres psql -U postgres

# Run single command
docker exec my-postgres ls /var/lib/postgresql

# -i = interactive, -t = TTY
```

### 3.6. Inspection

```bash
# Detailed info про container
docker inspect my-postgres

# Stats (CPU / memory)
docker stats

# Port mapping
docker port my-postgres

# Container processes
docker top my-postgres
```

### 3.7. Cleanup

```bash
# Stop ВСЕ running containers
docker stop $(docker ps -q)

# Remove ВСЕ stopped containers
docker rm $(docker ps -aq)

# System-wide cleanup (containers + images + volumes + networks)
docker system prune
docker system prune -a --volumes   # nuclear option

# Disk usage
docker system df
```

> [!question]- Интервью: ключевые Docker команды для daily dev?
> 1) **`docker run -d --name X -p PORT image`** — запустить container в background с port mapping. 2) **`docker ps`** — список running containers. 3) **`docker logs -f X`** — follow logs (debugging). 4) **`docker exec -it X bash`** — войти внутрь running container. 5) **`docker stop X / docker rm X`** — manage lifecycle. 6) **`docker compose up -d`** — multi-container apps (см. ниже). 7) **`docker system prune`** — cleanup disk. **Best practice**: используй `--name` для всех containers (легче управлять), `--rm` для one-off tasks.

---

## 4. Dockerfile — packaging своего приложения

### 4.1. Простой .NET API Dockerfile

```dockerfile
# Базовый image с runtime + ASP.NET Core
# (.NET 8+ images слушают порт 8080 под non-root user, не 80)
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
WORKDIR /app
EXPOSE 8080

# Build stage — image с SDK для compile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src

# Copy csproj и restore (cached если не менялся)
COPY ["MyApp.csproj", "."]
RUN dotnet restore

# Copy остальной код и build
COPY . .
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

# Publish — оптимизированная сборка
FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish

# Final stage — только runtime + опубликованное приложение
FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

Это **multi-stage build**:
1. SDK stage — компилирует
2. Final stage — только runtime (smaller image)

### 4.2. Dockerfile инструкции

```dockerfile
# Базовый image
FROM <image>:<tag>

# Working directory внутри container
WORKDIR /app

# Копирование файлов
COPY <src> <dest>
COPY ["file.txt", "/app/"]   # с пробелами в путях

# Запуск команды при build (создаёт layer)
RUN dotnet restore
RUN apt-get update && apt-get install -y curl

# Переменные окружения (build-time + runtime)
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production

# Открыть порт (документация — фактически не открывает)
EXPOSE 8080

# Команда при `docker run` (single command)
CMD ["dotnet", "MyApp.dll"]

# Точка входа (нельзя override при docker run)
ENTRYPOINT ["dotnet", "MyApp.dll"]

# Volume mount point
VOLUME /app/data

# Build arguments (только для build)
ARG VERSION=1.0.0
RUN echo "Building $VERSION"

# Метаданные
LABEL maintainer="vitaly@example.com"
```

### 4.3. .dockerignore — что не копировать

Важно: создай `.dockerignore` рядом с Dockerfile:

```
**/.dockerignore
**/.env
**/.git
**/.gitignore
**/.vs
**/.vscode
**/*.*proj.user
**/azds.yaml
**/bin
**/charts
**/docker-compose*
**/Dockerfile*
**/node_modules
**/npm-debug.log
**/obj
**/secrets.dev.yaml
**/values.dev.yaml
LICENSE
README.md
```

Ускоряет build, исключает мусор.

### 4.4. Build и run собственного image

```bash
# Build (из директории с Dockerfile)
docker build -t my-app:1.0 .

# Build с tag
docker build -t my-app:latest -t my-app:1.0 .

# Run собственного image (.NET app внутри слушает 8080)
docker run -d -p 8080:8080 --name my-app-1 my-app:1.0

# Проверь
curl http://localhost:8080/health
```

### 4.5. Health check

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1
```

Docker / Kubernetes используют это для restart unhealthy containers.

> [!question]- Интервью: что такое multi-stage build?
> Dockerfile с несколькими `FROM` — каждый = отдельный stage. **Зачем**: 1) **Stage 1 (SDK)** — содержит compiler, build tools, source code. Большой (~1 GB). 2) **Stage 2 (runtime)** — только runtime + скомпилированное приложение. Маленький (~220 MB). Source code НЕ в final image. **Преимущества**: 1) **Smaller final image** (3-5x меньше). 2) **Security** — нет SDK / source в production. 3) **Layer caching** — отдельный stage для restore (cached если csproj не менялся). **.NET pattern**: `mcr.microsoft.com/dotnet/sdk:10.0 AS build` → `mcr.microsoft.com/dotnet/aspnet:10.0 AS final`.

---

## 5. Docker Compose — несколько контейнеров вместе

### 5.1. Зачем compose

App обычно состоит из:
- API (.NET)
- БД (Postgres / SQL Server)
- Cache (Redis)
- Frontend (nginx)

Запустить вручную через `docker run` для каждого = боль. Compose описывает в YAML и одной командой.

### 5.2. docker-compose.yml — простой пример

```yaml
services:
  postgres:
    image: postgres:16
    container_name: myapp-pg
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U appuser"]
      interval: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: myapp-redis
    ports:
      - "6379:6379"

  api:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: myapp-api
    ports:
      - "8080:8080"
    environment:
      ConnectionStrings__Default: "Host=postgres;Database=myapp;Username=appuser;Password=secret"
      ConnectionStrings__Redis: "redis:6379"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started

volumes:
  pgdata:
```

### 5.3. Запуск compose

```bash
# Поднять всё
docker compose up

# В background
docker compose up -d

# Re-build перед запуском (если Dockerfile изменился)
docker compose up --build

# Остановить
docker compose stop

# Stop + remove containers
docker compose down

# Stop + remove + удалить volumes (DESTRUCTIVE — потеряешь данные)
docker compose down -v

# Logs всех
docker compose logs -f

# Logs конкретного service
docker compose logs -f api

# Restart service
docker compose restart api

# Только postgres
docker compose up postgres
```

### 5.4. Сетевое взаимодействие — service names

В compose сервисы общаются по имени:

```
api → postgres:5432   ✅ работает
api → localhost:5432  ❌ не работает (другой container)
```

```yaml
# .NET API connection string:
ConnectionStrings__Default: "Host=postgres;Database=myapp;..."
#                                ^^^^^^^^ имя service, не localhost
```

Compose создаёт internal network — все services видят друг друга.

### 5.5. depends_on — порядок запуска

```yaml
api:
  depends_on:
    postgres:
      condition: service_healthy   # ждать пока postgres healthy
    redis:
      condition: service_started   # просто wait для started
```

`service_healthy` требует `healthcheck` на target service.

### 5.6. Volumes — persistent data

```yaml
volumes:
  pgdata:                    # named volume — managed by Docker
  ./data:/app/data           # bind mount — мапит локальную папку
```

Без volume — data теряется при `docker compose down`.

```bash
# Список volumes
docker volume ls

# Inspect volume
docker volume inspect myapp_pgdata

# Remove volume
docker volume rm myapp_pgdata
```

### 5.7. Override files

```yaml
# docker-compose.yml — base
# docker-compose.override.yml — local dev (auto-loaded)
# docker-compose.prod.yml — production overrides

# Использование
docker compose -f docker-compose.yml -f docker-compose.prod.yml up
```

> [!question]- Интервью: зачем Docker Compose?
> Tool для **multi-container applications**. Вместо ручных `docker run` команд для каждого container — описание в YAML + одна команда `docker compose up`. **Что даёт**: 1) **Reproducible env** — один YAML, run anywhere. 2) **Networking** — services общаются по именам (api → postgres). 3) **Dependencies** — `depends_on` с health checks. 4) **Volume management** — persistent data. **Use cases**: local development environment (API + DB + Redis), CI/CD test setup, simple multi-service deployments. **Production**: для серьёзного prod — Kubernetes (compose не для scale).

---

## 6. .NET-specific patterns

### 6.1. dotnet publish с Docker SDK

`.NET 7+`: можно публиковать в container напрямую без Dockerfile:

```bash
dotnet publish /t:PublishContainer \
    -p ContainerRepository=my-app \
    -p ContainerImageTags=1.0
```

Microsoft built-in container generation — без Dockerfile для простых случаев. Деталь и когда это достаточно vs Dockerfile — [[docker|Senior/docker.md]], раздел «SDK container builds».

### 6.2. SQL Server в Docker

```yaml
sqlserver:
  image: mcr.microsoft.com/mssql/server:2022-latest
  environment:
    ACCEPT_EULA: "Y"
    MSSQL_SA_PASSWORD: "YourStrong@Passw0rd"
    MSSQL_PID: "Developer"
  ports:
    - "1433:1433"
  volumes:
    - sqlserverdata:/var/opt/mssql

volumes:
  sqlserverdata:
```

⚠️ Пароль должен соответствовать SQL Server complexity policy (8+ chars, upper/lower/digit/special).

### 6.3. PostgreSQL в Docker (preferred для dev)

```yaml
postgres:
  image: postgres:16
  environment:
    POSTGRES_DB: myapp
    POSTGRES_USER: appuser
    POSTGRES_PASSWORD: secret
  ports:
    - "5432:5432"
  volumes:
    - pgdata:/var/lib/postgresql/data
    - ./init.sql:/docker-entrypoint-initdb.d/init.sql   # auto-run при первом запуске

volumes:
  pgdata:
```

`./init.sql` запустится один раз при создании БД — useful для seed data.

### 6.4. Redis

```yaml
redis:
  image: redis:7-alpine
  ports:
    - "6379:6379"
  command: redis-server --requirepass secret   # с паролем
```

### 6.5. RabbitMQ

```yaml
rabbitmq:
  image: rabbitmq:3-management
  ports:
    - "5672:5672"      # AMQP
    - "15672:15672"    # Management UI (http://localhost:15672)
  environment:
    RABBITMQ_DEFAULT_USER: admin
    RABBITMQ_DEFAULT_PASS: admin
```

### 6.6. Connection strings в .NET API

```json
// appsettings.Development.json
{
  "ConnectionStrings": {
    "Default": "Host=localhost;Database=myapp;Username=appuser;Password=secret"
  }
}
```

```yaml
# docker-compose.override.yml — для запуска API в Docker
services:
  api:
    environment:
      ConnectionStrings__Default: "Host=postgres;Database=myapp;Username=appuser;Password=secret"
      # double underscore = nested config
```

### 6.7. Visual Studio integration

VS 2022 поддерживает Docker:
- Right-click project → Add → Docker Support
- Auto-generates Dockerfile + docker-compose
- F5 запускает в Docker
- Debugger attaches к container

### 6.8. JetBrains Rider integration

Rider тоже поддерживает Docker:
- Build configurations с docker-compose
- Run/Debug в containers
- Database tool window работает с Postgres/SQL Server в Docker

---

## 7. Common pitfalls

### 7.1. localhost из container

```csharp
// ❌ В container localhost = container, не host machine
var connStr = "Host=localhost;Database=myapp";

// ✅ Через имя service в compose
var connStr = "Host=postgres;Database=myapp";

// ✅ Если postgres на host (не в Docker)
var connStr = "Host=host.docker.internal;Database=myapp";
// host.docker.internal — special DNS для доступа к host
```

### 7.2. Volume permissions

```bash
# ❌ App не может писать в volume
docker run -v /tmp/data:/app/data my-app

# Часто возникает на Linux — UID mismatch
# В container app запускается под non-root user (UID 1000)
# Host folder owned by root → permission denied

# Fix: указать UID
RUN useradd -u 1001 appuser
USER appuser
# Or: chown папки в Dockerfile
```

### 7.3. Image не пересобирается

```bash
# ❌ docker compose up — использует старый image
# Изменил Dockerfile, но не вижу changes

# ✅ Force rebuild
docker compose up --build
docker compose build --no-cache   # ignore cache полностью
```

### 7.4. .dockerignore забыл

Без `.dockerignore` копируется `bin/`, `obj/`, `.vs/`, `node_modules/`. Build медленнее и image bigger.

### 7.5. Multi-stage без cache optimization

```dockerfile
# ❌ Cache invalidated каждый раз
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .                              # копируем ВСЁ перед restore
RUN dotnet restore                    # cache invalid каждый раз
RUN dotnet publish -c Release -o /app

# ✅ Optimized cache
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["MyApp.csproj", "."]            # только csproj
RUN dotnet restore                    # cached если csproj не менялся!
COPY . .                              # потом остальной код
RUN dotnet publish -c Release -o /app
```

### 7.6. EXPOSE без -p

```bash
# ❌ EXPOSE в Dockerfile — только документация!
docker run my-app
# Port НЕ открыт извне

# ✅ Явный port mapping
docker run -p 8080:8080 my-app
```

### 7.7. Image size

```bash
# Image весит 1.5 GB?
docker images
# Проверь base image:
# FROM mcr.microsoft.com/dotnet/sdk:10.0  → SDK ~1 GB
# FROM mcr.microsoft.com/dotnet/aspnet:10.0  → runtime ~220 MB

# Используй runtime для final stage, SDK только для build
```

### 7.8. Latest tag

```dockerfile
# ❌ "Latest" — может неожиданно ломаться
FROM mcr.microsoft.com/dotnet/aspnet:latest

# ✅ Pin version
FROM mcr.microsoft.com/dotnet/aspnet:10.0
```

### 7.9. Secrets в Dockerfile

```dockerfile
# ❌ Hardcoded в image — visible через docker history
ENV DATABASE_PASSWORD=secret123
```

**Фикс**: pass at runtime:

```bash
docker run -e DATABASE_PASSWORD=$(cat secret.txt) my-app
# или через docker-compose secrets / Kubernetes Secrets
```

### 7.10. Не используешь --rm для one-off

```bash
# ❌ Каждый раз остаётся stopped container
docker run my-cli-tool arg1 arg2
# docker ps -a показывает 100 stopped containers

# ✅ Auto-remove
docker run --rm my-cli-tool arg1 arg2
```

> [!question]- Интервью: топ-3 ошибки с Docker?
> 1) **`localhost` в container vs host** — service-to-service в compose через service names (`postgres:5432`), доступ к host из container через `host.docker.internal`. 2) **Cache layer breaking** — `COPY . .` перед `dotnet restore` invalidates cache при каждом code change. Fix: copy csproj first, restore, then copy остальное. 3) **Не pin version в FROM** — `FROM aspnet:latest` приводит к unexpected breakage. Fix: explicit version (`aspnet:10.0`). **Bonus**: `EXPOSE` в Dockerfile это только документация — нужен `-p` при run.

---

## 8. Cheat sheet

```bash
# === Daily commands ===
docker run -d --name X -p 8080:80 image     # run в background
docker ps                                    # список running
docker ps -a                                 # все (включая stopped)
docker logs -f X                            # follow logs
docker exec -it X bash                       # shell в container
docker stop X                                # stop
docker rm X                                  # remove
docker rm -f X                               # force remove (running)

# === Images ===
docker images                                # список
docker pull image:tag                        # download
docker build -t my-app:1.0 .                 # build
docker rmi image                             # delete
docker image prune                           # cleanup unused

# === Compose ===
docker compose up -d                         # start всех services
docker compose down                          # stop + remove
docker compose down -v                       # + удалить volumes (DESTRUCTIVE)
docker compose logs -f api                   # logs одного service
docker compose restart api                   # restart
docker compose up --build                    # rebuild перед start

# === Cleanup ===
docker system prune                          # cleanup containers/networks/images
docker system prune -a --volumes             # nuclear option
docker volume ls                             # list volumes
docker volume rm volume_name                 # remove
```

```dockerfile
# === Optimized .NET multi-stage Dockerfile ===
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY ["MyApp.csproj", "."]
RUN dotnet restore
COPY . .
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

```yaml
# === Common compose template ===
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready"]
      interval: 5s
  
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  
  api:
    build: .
    ports: ["8080:8080"]
    depends_on:
      postgres: { condition: service_healthy }

volumes:
  pgdata:
```

---

## 9. Practice exercises

### 9.1. Dockerize простой .NET API

Возьми существующий ASP.NET Core Web API и:
1. Создай Dockerfile (multi-stage, Microsoft images)
2. Создай `.dockerignore`
3. `docker build -t my-api:1.0 .`
4. `docker run -d -p 8080:8080 my-api:1.0`
5. `curl http://localhost:8080/health` или `/swagger`
6. Размер image после build — `docker images`

### 9.2. Dev environment с compose

Создай `docker-compose.yml` с:
- PostgreSQL 16
- Redis 7
- pgAdmin (для UI к Postgres)
- Твой API (build из Dockerfile)

Все services должны:
- Иметь named volumes
- Communicate по service names
- API должен ждать пока Postgres healthy
- Открыть API на localhost:8080, pgAdmin на :5050

### 9.3. Optimize image size

Возьми Dockerfile из 9.1. Цель: уменьшить final image:
1. Использовать alpine variant (`mcr.microsoft.com/dotnet/aspnet:10.0-alpine`)
2. Скомбинировать `RUN` команды
3. Использовать `.dockerignore`
4. Сравни размеры: до/после

Цель: < 100 MB final image.

---

## 10. Что читать дальше

1. **`Infrastructure/Senior/docker.md`** — production-grade Dockerfiles, security, optimization
2. **`Infrastructure/Middle/kubernetes.md`** — orchestration для production
3. **`Infrastructure/Middle/cicd-github-actions.md`** — Docker в CI/CD pipeline
4. **`Infrastructure/Senior/observability.md`** — logging / monitoring containers

---

## 11. См. также

- [[docker|Infrastructure/Senior/docker]] — production deep
- [[kubernetes|Infrastructure/Middle/kubernetes]] — K8s
- [[cicd-github-actions|Infrastructure/Middle/cicd-github-actions]] — CI/CD
- [[project-setup-basics|Infrastructure/Junior/project-setup-basics]] — solution structure
- [[ef-basics|EFCore/Junior/ef-basics]] — EF Core с Postgres в Docker

---

## 12. Reading list

- **Docker Docs** — docs.docker.com/get-started/
- **Microsoft Docs — Docker для .NET** — learn.microsoft.com/dotnet/core/docker/build-container
- **"Docker Deep Dive" — Nigel Poulton** (book)
- **Microsoft images** — hub.docker.com/_/microsoft-dotnet
- **Best practices Dockerfile** — docs.docker.com/develop/develop-images/dockerfile_best-practices/
- **Awesome Compose** — github.com/docker/awesome-compose
