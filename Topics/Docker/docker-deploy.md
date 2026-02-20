# Docker и деплой

## Dockerfile для .NET

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 8080

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["MyApi/MyApi.csproj", "MyApi/"]
RUN dotnet restore "MyApi/MyApi.csproj"
COPY . .
RUN dotnet build "MyApi/MyApi.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApi/MyApi.csproj" -c Release -o /app/publish

FROM base AS final
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

Multi-stage — итоговый образ без SDK. Порядок: restore → build → publish → copy в runtime образ.

---

## docker-compose

```yaml
services:
  api:
    build: .
    ports: ["8080:8080"]
    environment:
      - ConnectionStrings__DefaultConnection=Host=db;Database=app
    depends_on: [db]

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes: [pgdata:/var/lib/postgresql/data]

volumes:
  pgdata:
```

---

## Git basics

Ветки (main, feature/*), коммиты (conventional commits), PR, merge/rebase. .gitignore для .NET. С первого дня — осознанные коммиты.

---

## CI/CD

**GitHub Actions** — workflow на push/PR. Steps: checkout → setup .NET → restore → build → test → publish → push to registry / deploy.

**Azure DevOps** — pipelines, stages (build, test, deploy). Переменные, секреты.

**Базовый flow**: коммит → build → unit tests → integration tests (Testcontainers) → Docker build → push → deploy to staging/prod.

---

## См. также

- [[Topics/Testing/testing-xunit-testcontainers|Testing]]
- [[Topics/Observability/opentelemetry-jaeger-seq|OpenTelemetry]]
- [[dotnet-knowledge-base|.NET Knowledge Base]]
