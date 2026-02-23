---
tags: [docker, deploy, multi-stage, containers]
level: Senior
---

# Docker и деплой

## Dockerfile для .NET

### Multi-stage build

```dockerfile
# Stage 1: Build
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Копируем только csproj для кэширования restore
COPY ["src/MyApi/MyApi.csproj", "src/MyApi/"]
COPY ["src/Domain/Domain.csproj", "src/Domain/"]
COPY ["src/Infrastructure/Infrastructure.csproj", "src/Infrastructure/"]
RUN dotnet restore "src/MyApi/MyApi.csproj"

# Копируем остальное и билдим
COPY . .
RUN dotnet publish "src/MyApi/MyApi.csproj" -c Release -o /app/publish \
    --no-restore \
    /p:UseAppHost=false

# Stage 2: Runtime (без SDK — маленький образ)
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
EXPOSE 8080

# Non-root user (security)
USER $APP_UID

COPY --from=build /app/publish .
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

**Нюансы:**
- Порядок `COPY` → `restore` → `COPY . .` → `build` — кэширование слоёв. NuGet restore переделывается только при изменении `.csproj`
- `aspnet:8.0` (~220MB) vs `aspnet:8.0-alpine` (~100MB) — Alpine меньше, но могут быть проблемы с glibc
- `USER $APP_UID` — не запускать от root в production
- `UseAppHost=false` — не создавать native executable (не нужен в контейнере)

### .dockerignore

```
**/bin/
**/obj/
**/.git
**/node_modules
**/*.user
**/*.suo
**/Dockerfile*
**/docker-compose*
```

---

## docker-compose

### Полный пример с API + PostgreSQL + Redis

```yaml
services:
  api:
    build:
      context: .
      dockerfile: src/MyApi/Dockerfile
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__Default=Host=db;Port=5432;Database=app;Username=app;Password=secret
      - Redis__ConnectionString=redis:6379
    depends_on:
      db:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

**Нюансы:**
- `depends_on` с `condition: service_healthy` — API не стартует пока БД не готова
- `restart: unless-stopped` — автоматический рестарт при crash
- `volumes` — данные сохраняются между рестартами контейнера
- Alpine-образы — меньше размер, быстрее pull

### Полезные команды

```bash
docker compose up -d           # запуск в фоне
docker compose logs -f api     # логи API
docker compose down -v         # остановка + удаление volumes
docker compose build --no-cache # полная пересборка
docker compose exec db psql -U app -d app  # подключение к БД
```

---

## Git basics

### Branching strategy

```
main ────────────────────────────────────►
  │                    ▲
  └── feature/add-orders ──► PR ──► merge
  │                    ▲
  └── fix/order-validation ► PR ──► merge
```

- **main** — всегда deployable
- **feature/*** — новая функциональность
- **fix/*** — баг-фиксы
- **release/*** — подготовка релиза (если нужно)

### Conventional Commits

```
feat: add order creation endpoint
fix: correct total calculation for discounted items
refactor: extract validation into FluentValidation
docs: update API documentation
chore: upgrade EF Core to 8.0.2
test: add integration tests for orders API
```

### .gitignore для .NET

```gitignore
## Build results
[Bb]in/
[Oo]bj/

## User-specific
*.user
*.suo
*.DotSettings.user

## NuGet
**/packages/

## IDE
.vs/
.idea/
*.swp

## Environment
.env
appsettings.*.json
!appsettings.json
!appsettings.Development.json
```

---

## CI/CD

### GitHub Actions — базовый workflow

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

    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: testdb
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore -warnaserror

      - name: Unit Tests
        run: dotnet test --no-build --filter "Category=Unit" --logger trx

      - name: Integration Tests
        run: dotnet test --no-build --filter "Category=Integration" --logger trx
        env:
          ConnectionStrings__Default: "Host=localhost;Database=testdb;Username=test;Password=test"

      - name: Publish Test Results
        uses: dorny/test-reporter@v1
        if: always()
        with:
          name: Test Results
          path: '**/*.trx'
          reporter: dotnet-trx
```

### Docker build + push в CI

```yaml
  docker:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

### Базовый flow

```
commit → build → analyzers → unit tests → integration tests
  → Docker build → push to registry → deploy to staging → deploy to prod
```

**Нюанс:** миграции БД — отдельный step ПЕРЕД деплоем приложения. Не в коде приложения (`MigrateAsync`), а через CI/CD pipeline.

---

## Health Checks в Docker

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddNpgSql(connectionString, name: "postgres")
    .AddRedis(redisConnection, name: "redis");

app.MapHealthChecks("/health/live", new() { Predicate = _ => false }); // liveness
app.MapHealthChecks("/health/ready"); // readiness
```

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
    CMD curl -f http://localhost:8080/health/live || exit 1
```

---

## Best Practices

- **Multi-stage** — итоговый образ без SDK (~100-220MB вместо ~800MB)
- **Non-root** — `USER $APP_UID` в production
- **.dockerignore** — не копировать bin/, obj/, .git/
- **Healthcheck** — и в Dockerfile, и в docker-compose
- **Secrets** — не в environment variables, использовать Docker secrets или Vault
- **Logs** — писать в stdout/stderr, не в файлы (Docker logging driver собирает)

---

## См. также

- [Testing](../Testing/testing-xunit-testcontainers.md)
- [OpenTelemetry](../Observability/opentelemetry-jaeger-seq.md)
