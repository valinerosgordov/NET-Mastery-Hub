---
tags: [architecture, case-study, migration, modular-monolith]
level: Senior
date: 2026-08-02
---

# WebAI C# Architecture — Production Case Study (anonymized)

> Landing page generator: user fills form -> AI generates texts + images -> site published instantly.
> Migration from Next.js 16 / TypeScript / Supabase to **.NET 10 / C# 14 modular monolith** — migration completed 2026-04.
>
> **Анонимизированный case study реального production-проекта.** Имя продукта, домены и инфраструктурные детали из публичной версии убраны; архитектурные решения, код и найденные уязвимости — подлинные. Раздел 12 — **исторический** security-аудит legacy Next.js-версии: все CRITICAL/HIGH закрыты hardening-проходом в апреле 2026, остальные пункты сняты самой миграцией на .NET (разделы 7–8 показывают, чем именно).

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Bounded Contexts (DDD)](#2-bounded-contexts)
3. [Project Structure](#3-project-structure)
4. [Domain Layer](#4-domain-layer)
5. [Application Layer](#5-application-layer)
6. [Infrastructure Layer](#6-infrastructure-layer)
7. [API Layer](#7-api-layer)
8. [Security Hardening](#8-security-hardening)
9. [Performance Optimization](#9-performance-optimization)
10. [Docker & CI/CD](#10-docker--cicd)
11. [Migration Plan](#11-migration-plan)
12. [Legacy Next.js Security Audit (historical)](#12-legacy-nextjs-security-audit-historical)

---

## 1. System Overview

```
User -> Funnel Form -> AI Content (Gemini) -> AI Images (Gemini Flash Image)
     -> Store in PostgreSQL + S3 -> Serve at /site/{id}
```

Key components:
- **15 visual styles**, 12 industries, 12 color presets
- **14 AI images per site** (hero, about, 6 portfolio, 6 gallery)
- **S3-compatible storage** (MinIO / AWS / Yandex Object Storage)
- **Rate limiting** per IP, per route
- **Admin panel** with JWT auth
- **Telegram notifications** on site creation
- **HTML export** for standalone deployment

---

## 2. Bounded Contexts

| Context | Type | Responsibility |
|---------|------|---------------|
| **SiteGeneration** | Core Domain | Site CRUD, lifecycle, authorization, export |
| **ContentGeneration** | Supporting | AI text/content generation, voice parsing, style recommendation |
| **ImageGeneration** | Supporting | AI image generation, S3 storage, Unsplash fallback |
| **Identity** | Generic | Admin JWT auth, edit token verification |
| **Notification** | Generic | Telegram fire-and-forget |
| **RateLimiting** | Infrastructure | Per-IP throttling with configurable limits |

---

## 3. Project Structure

```
src/
  WebAI.Domain/                      # Zero dependencies, pure C#
    Common/
      Result.cs                      # Result<T> + Error pattern
      Entity.cs                      # Entity<TId>, AggregateRoot<TId>
      IDomainEvent.cs
      IUnitOfWork.cs
    SiteGeneration/
      Entities/Site.cs               # Aggregate root (core)
      ValueObjects/
        SiteId.cs                    # Guid wrapper (NewV7)
        EditToken.cs
        ProjectInfo.cs               # XSS sanitization on create
        OwnerContact.cs              # Email validation
        DesignPreferences.cs         # URL scheme validation
        ColorPreset.cs               # FrozenSet validation
        AiGeneratedTexts.cs
        AiGeneratedContent.cs
        ImageHints.cs
        SocialLinks.cs
        PricelistCategory.cs
      Enums/
        VisualStyle.cs               # 15 styles
        Industry.cs                  # 18 industries
        ThemeMode.cs, ScrollMode.cs, NavStyle.cs
        ConversionGoal.cs, MessengerType.cs, SiteStatus.cs
      Events/
        SiteCreatedEvent.cs          # -> TelegramNotification, AiGeneration
        SiteUpdatedEvent.cs
        SiteDeactivatedEvent.cs
        SiteDeletedEvent.cs          # -> ImageCleanup
        SiteDuplicatedEvent.cs
      Repositories/ISiteRepository.cs
      Services/
        SiteEditPolicy.cs            # Field-level authorization
        ISiteExportService.cs
    ContentGeneration/
      Services/
        ITextGenerationService.cs
        IContentGenerationService.cs
        IVoiceParsingService.cs
        IStyleRecommendationService.cs
    ImageGeneration/
      Services/
        IImageGenerationService.cs   # Partial failure contract
        IImageStorageService.cs
        ImageFallbackService.cs      # Unsplash industry fallbacks
    Identity/
      Services/IAdminAuthService.cs
      ValueObjects/AdminCredentials.cs
    Notification/
      Services/INotificationService.cs
    RateLimiting/
      ValueObjects/RateLimitPolicy.cs, ClientIp.cs
      Services/IRateLimitService.cs

  WebAI.Application/                 # Use cases, DTOs, validators
    Abstractions/
      IAiClient.cs
      IStorageClient.cs
      ITelegramNotifier.cs
    Services/
      SiteGenerationService.cs       # Orchestrates: create site -> AI -> images -> save
      ImageGenerationOrchestrator.cs  # Background queue consumer
      HtmlExportService.cs
    Contracts/                       # Request/Response DTOs
    Validators/                      # FluentValidation

  WebAI.Infrastructure/              # External concerns
    Persistence/
      AppDbContext.cs                # EF Core + PostgreSQL
      Configurations/SiteConfiguration.cs  # JSONB mapping
      Repositories/SiteRepository.cs
    Storage/
      S3StorageClient.cs            # AWSSDK.S3 with MinIO compat
      LocalFileStorageClient.cs     # Filesystem fallback
    AI/
      GeminiClient.cs               # Polly v8 retry + circuit breaker
      GeminiImageClient.cs
    Auth/
      JwtService.cs                 # 15-min access + refresh rotation
      AdminAuthHandler.cs
    Notifications/TelegramNotifier.cs
    Health/
      GeminiHealthCheck.cs
      S3HealthCheck.cs

  WebAI.Api/                         # Minimal API host
    Program.cs                       # Composition root
    Endpoints/
      GenerateTextsEndpoint.cs
      GenerateContentEndpoint.cs
      SearchImagesEndpoint.cs
      RecommendStyleEndpoint.cs
      ParseVoiceEndpoint.cs
      ImageProxyEndpoint.cs
      ExportHtmlEndpoint.cs
      AdminEndpoints.cs
      SiteEndpoints.cs
    Middleware/
      RateLimitingMiddleware.cs
      ExceptionHandlingMiddleware.cs
      RequestTimingMiddleware.cs
    Filters/ValidationFilter.cs

  WebAI.Tests/
    Unit/
    Integration/
```

---

## 4. Domain Layer

### 4.1 Result Pattern

```csharp
namespace WebAI.Domain.Common;

public sealed record Error(string Code, string Message)
{
    public static readonly Error None = new(string.Empty, string.Empty);
    public static Error Validation(string message) => new("Validation", message);
    public static Error NotFound(string entity, string id) => new("NotFound", $"{entity} '{id}' not found");
    public static Error Unauthorized(string message = "Unauthorized") => new("Unauthorized", message);
    public static Error Conflict(string message) => new("Conflict", message);
    public static Error RateLimited(int retryAfterSeconds) => new("RateLimited", $"Retry after {retryAfterSeconds}s");
    public static Error ExternalService(string service, string message) => new("ExternalService", $"{service}: {message}");
}

public readonly struct Result<T>
{
    private readonly T? _value;
    private readonly Error? _error;

    private Result(T value) { _value = value; _error = null; IsSuccess = true; }
    private Result(Error error) { _value = default; _error = error; IsSuccess = false; }

    public bool IsSuccess { get; }
    public bool IsFailure => !IsSuccess;
    public T Value => IsSuccess ? _value! : throw new InvalidOperationException($"Failed result: {_error}");
    public Error Error => !IsSuccess ? _error! : throw new InvalidOperationException("Successful result");

    public static implicit operator Result<T>(T value) => new(value);
    public static implicit operator Result<T>(Error error) => new(error);

    public Result<TOut> Map<TOut>(Func<T, TOut> map) =>
        IsSuccess ? new Result<TOut>(map(_value!)) : _error!;

    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<Error, TOut> onFailure) =>
        IsSuccess ? onSuccess(_value!) : onFailure(_error!);
}
```

### 4.2 Site Aggregate Root

```csharp
namespace WebAI.Domain.SiteGeneration.Entities;

public sealed class Site : AggregateRoot<SiteId>
{
    public ProjectInfo ProjectInfo { get; private set; }
    public OwnerContact OwnerContact { get; private set; }
    public DesignPreferences Design { get; private set; }
    public IReadOnlyList<SiteSection> Sections { get; private set; }
    public ConversionGoal ConversionGoal { get; private set; }

    // AI-generated (JSONB in DB)
    public AiGeneratedTexts? AiTexts { get; private set; }
    public AiGeneratedContent? AiContent { get; private set; }
    public ImageHints? ImageHints { get; private set; }

    // Technical
    public EditToken? EditToken { get; private set; }
    public SiteStatus Status { get; private set; }
    public DateTime CreatedAt { get; private set; }
    public DateTime? UpdatedAt { get; private set; }
    public uint RowVersion { get; private set; } // Optimistic concurrency

    /// Factory: the ONLY way to create a Site
    public static Result<Site> Create(
        ProjectInfo projectInfo, OwnerContact ownerContact,
        DesignPreferences design, IEnumerable<SiteSection> sections,
        ConversionGoal conversionGoal) { /* ... */ }

    /// Idempotent: attach AI texts (partial success OK)
    public void AttachAiTexts(AiGeneratedTexts texts) { /* ... */ }
    public void AttachAiContent(AiGeneratedContent content) { /* ... */ }
    public void AttachImageHints(ImageHints hints) { /* ... */ }

    /// Validates edit authorization via SiteEditPolicy
    public Result Update(SiteChanges changes) { /* ... */ }
    public Result Deactivate() { /* ... */ }
    public Result Activate() { /* ... */ }
    public Result MarkDeleted() { /* ... */ }
    public Result<Site> Duplicate() { /* ... */ }
}
```

### 4.3 Key Value Objects

```csharp
// ProjectInfo — XSS sanitization at the domain boundary
public sealed record ProjectInfo
{
    public string ProjectName { get; }
    public string ValueProposition { get; }
    public string TargetAudience { get; }

    public static Result<ProjectInfo> Create(string projectName, string valueProposition, string targetAudience)
    {
        if (string.IsNullOrWhiteSpace(projectName)) return Error.Validation("Name required");
        if (projectName.Length > 200) return Error.Validation("Name max 200 chars");

        var sanitizedName = SanitizeInput(projectName.Trim());
        var sanitizedValue = SanitizeInput(valueProposition.Trim());
        return new ProjectInfo(sanitizedName, sanitizedValue, SanitizeInput(targetAudience.Trim()));
    }

    private static string SanitizeInput(string input) =>
        input.Replace("<", "&lt;").Replace(">", "&gt;").Replace("\"", "&quot;");
}

// SiteId — Guid v7 wrapper
public readonly record struct SiteId(Guid Value)
{
    public static SiteId New() => new(Guid.CreateVersion7());
    public static Result<SiteId> From(string raw) =>
        Guid.TryParse(raw, out var guid) ? new SiteId(guid) : Error.Validation("Invalid site ID");
}

// ColorPreset — FrozenSet validation
public sealed record ColorPreset(string Id, string Label, string Primary, string Accent)
{
    private static readonly FrozenSet<string> ValidIds = new[]
    {
        "purple", "blue", "green", "orange", "red", "gold",
        "cyan", "pink", "rose", "sage", "teal", "lime", "charcoal"
    }.ToFrozenSet();

    public static bool IsValid(string id) => ValidIds.Contains(id);
}

// DesignPreferences — URL scheme validation
public sealed record DesignPreferences
{
    public static Result<DesignPreferences> Create(VisualStyle style, string colorPresetId, ...)
    {
        if (!ColorPreset.IsValid(colorPresetId))
            return Error.Validation($"Unknown color preset: {colorPresetId}");

        // Block javascript:, data:, vbscript: URLs
        if (heroVideoUrl is not null)
        {
            var lower = heroVideoUrl.Trim().ToLowerInvariant();
            if (lower.StartsWith("javascript:") || lower.StartsWith("data:") || lower.StartsWith("vbscript:"))
                return Error.Validation("Invalid hero video URL");
        }
        // ...
    }
}
```

### 4.4 Enums

```csharp
public enum VisualStyle
{
    Minimal, Tech, Creative, Classic, Brutalist, Glass, Neon,
    Organic, Corporate, Editorial, Retro, Luxury, Cinematic, Aurora, Neobrutalist
}

public enum Industry
{
    Animal, Auto, Restaurant, Beauty, Fitness, Medical, Lawyer,
    Construction, It, Saas, Ecommerce, Education, RealEstate,
    Photography, Cleaning, Logistics, Travel, Default
}

public enum SiteStatus { Active, Inactive, Deleted }
public enum ThemeMode { Dark, Light }
public enum ConversionGoal { Form, Messenger, Call, Register }
```

### 4.5 Domain Events

```csharp
public sealed record SiteCreatedEvent(SiteId SiteId, string ProjectName, string OwnerEmail, DateTime OccurredOn) : IDomainEvent;
public sealed record SiteUpdatedEvent(SiteId SiteId, DateTime OccurredOn) : IDomainEvent;
public sealed record SiteDeletedEvent(SiteId SiteId, DateTime OccurredOn) : IDomainEvent;
public sealed record SiteDuplicatedEvent(SiteId NewSiteId, SiteId OriginalSiteId, DateTime OccurredOn) : IDomainEvent;
```

Event handlers (Application layer):
- `SiteCreatedEvent` -> `TelegramNotificationHandler` (fire-and-forget)
- `SiteCreatedEvent` -> `AiContentGenerationHandler` (parallel text+content+image)
- `SiteDeletedEvent` -> `ImageCleanupHandler` (S3 delete)

---

## 5. Application Layer

### 5.1 Image Generation Orchestrator (Background)

```csharp
public sealed class ImageGenerationBackgroundService(
    Channel<ImageGenerationJob> channel,
    IServiceScopeFactory scopeFactory,
    ILogger<ImageGenerationBackgroundService> logger) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        await foreach (var job in channel.Reader.ReadAllAsync(ct))
        {
            using var scope = scopeFactory.CreateScope();
            var orchestrator = scope.ServiceProvider.GetRequiredService<ImageGenerationOrchestrator>();
            try { await orchestrator.GenerateAndStoreAsync(job, ct); }
            catch (Exception ex) { logger.LogError(ex, "Image gen failed: {SiteId}", job.SiteId); }
        }
    }
}
```

---

## 6. Infrastructure Layer

### 6.1 AI Client with Polly v8

```csharp
// DI registration
builder.Services.AddHttpClient<IAiClient, GeminiClient>(client =>
{
    client.BaseAddress = new Uri("https://generativelanguage.googleapis.com/");
    client.DefaultRequestHeaders.Add("x-goog-api-key", config.ApiKey);
    client.Timeout = TimeSpan.FromSeconds(60);
})
.AddResilienceHandler("gemini", pipeline =>
{
    pipeline.AddRetry(new()
    {
        MaxRetryAttempts = 2,
        Delay = TimeSpan.FromSeconds(1),
        BackoffType = DelayBackoffType.Linear,
        ShouldHandle = new PredicateBuilder<HttpResponseMessage>()
            .HandleResult(r => r.StatusCode is
                HttpStatusCode.TooManyRequests or
                HttpStatusCode.ServiceUnavailable or
                HttpStatusCode.GatewayTimeout)
    });
    pipeline.AddCircuitBreaker(new()
    {
        FailureRatio = 0.5,
        SamplingDuration = TimeSpan.FromSeconds(30),
        MinimumThroughput = 5,
        BreakDuration = TimeSpan.FromSeconds(15),
    });
    pipeline.AddTimeout(TimeSpan.FromSeconds(45));
});
```

### 6.2 EF Core + PostgreSQL (JSONB)

```csharp
public sealed class SiteConfiguration : IEntityTypeConfiguration<Site>
{
    public void Configure(EntityTypeBuilder<Site> builder)
    {
        builder.ToTable("sites");
        builder.HasKey(s => s.Id);
        builder.Property(s => s.Id).HasColumnName("id");
        builder.Property(s => s.EditToken).HasColumnName("edit_token").HasDefaultValueSql("gen_random_uuid()");

        // JSONB columns
        builder.Property(s => s.AiTexts).HasColumnName("ai_texts").HasColumnType("jsonb");
        builder.Property(s => s.AiContent).HasColumnName("ai_content").HasColumnType("jsonb");
        builder.Property(s => s.ImageHints).HasColumnName("image_hints").HasColumnType("jsonb");
        builder.Property(s => s.Pricelist).HasColumnName("pricelist").HasColumnType("jsonb");

        // Optimistic concurrency
        builder.Property(s => s.RowVersion).IsRowVersion();

        // Indexes
        builder.HasIndex(s => s.OwnerEmail).HasDatabaseName("ix_sites_owner_email");
        builder.HasIndex(s => s.CreatedAt).HasDatabaseName("ix_sites_created_at");
    }
}
```

### 6.3 S3 Storage Client

```csharp
public sealed class S3StorageClient(IAmazonS3 s3Client, IOptions<S3Options> options, ILogger<S3StorageClient> logger) : IStorageClient
{
    public async Task<Result<string>> UploadImageAsync(string siteId, string filename, ReadOnlyMemory<byte> data, string contentType = "image/png", CancellationToken ct = default)
    {
        var key = $"{siteId}/{filename}";

        // Path traversal prevention
        if (key.Contains("..") || key.Contains('\\'))
            return Error.Validation("Invalid file path");

        // Size limit (10MB)
        if (data.Length > 10 * 1024 * 1024)
            return Error.Validation("Image exceeds 10MB limit");

        using var stream = new MemoryStream(data.ToArray());
        await s3Client.PutObjectAsync(new PutObjectRequest
        {
            BucketName = options.Value.Bucket, Key = key,
            InputStream = stream, ContentType = contentType,
        }, ct);

        return $"/api/img/{key}";
    }
}
```

### 6.4 JWT Auth (15-min + refresh rotation)

```csharp
public sealed class JwtService(IOptions<JwtOptions> options)
{
    public (string AccessToken, string RefreshToken) GenerateTokenPair(AdminUser user)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(options.Value.Secret));
        var claims = new Claim[]
        {
            new(ClaimTypes.Email, user.Email),
            new(ClaimTypes.Role, "Admin"),
            new("jti", Guid.NewGuid().ToString()),
        };
        var token = new JwtSecurityToken(
            issuer: options.Value.Issuer, audience: options.Value.Audience,
            claims: claims, expires: DateTime.UtcNow.AddMinutes(15),
            signingCredentials: new(key, SecurityAlgorithms.HmacSha256));
        var refreshToken = Convert.ToBase64String(RandomNumberGenerator.GetBytes(64));
        return (new JwtSecurityTokenHandler().WriteToken(token), refreshToken);
    }
}
```

---

## 7. API Layer (Minimal API)

```csharp
// Program.cs
var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();
app.UseMiddleware<RequestTimingMiddleware>();
app.UseSerilogRequestLogging();
app.UseRateLimiter();
app.UseAuthentication();
app.UseAuthorization();

app.MapGroup("/api/ai").WithTags("AI").RequireRateLimiting("ai-generate")
    .AddEndpointFilter<ValidationFilter>()
    .MapGenerateTextsEndpoint()
    .MapGenerateContentEndpoint()
    .MapRecommendStyleEndpoint()
    .MapParseVoiceEndpoint();

app.MapGroup("/api/images").WithTags("Images").RequireRateLimiting("images")
    .MapSearchImagesEndpoint()
    .MapImageProxyEndpoint();

app.MapGroup("/api/sites").WithTags("Sites").MapSiteEndpoints();
app.MapGroup("/api/admin").WithTags("Admin").RequireAuthorization("Admin").MapAdminEndpoints();
app.MapGroup("/api/export").WithTags("Export").MapExportEndpoints();

app.MapHealthChecks("/health");
```

### Rate Limiting (mirrors existing JS thresholds)

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = 429;

    options.AddPolicy("ai-generate", ctx =>
    {
        var ip = ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        return RateLimitPartition.GetFixedWindowLimiter($"ai:{ip}", _ => new()
        {
            PermitLimit = 10,  // API_LIMITS.generate
            Window = TimeSpan.FromMinutes(1),
        });
    });

    options.AddPolicy("images", ctx =>
    {
        var ip = ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        return RateLimitPartition.GetFixedWindowLimiter($"img:{ip}", _ => new()
        {
            PermitLimit = 5,   // API_LIMITS.images
            Window = TimeSpan.FromMinutes(1),
        });
    });
});
```

---

## 8. Security Hardening

### 8.1 Threat Model

| Threat | Mitigation |
|--------|-----------|
| **Race conditions in site generation** | `RowVersion` optimistic concurrency on Site aggregate |
| **Image generation partial failure** | `ImageGenerationResult` with counts; null entries allowed |
| **Rate limiting bypass (IP spoofing)** | `ForwardedHeaders` with `KnownProxies` only, NOT raw headers |
| **JSONB data corruption** | Value objects validate on construction; no raw JSON storage |
| **Concurrent edits** | Optimistic concurrency via EF Core `IsRowVersion()` |
| **File upload bombs** | 10MB per image limit, MIME validation, magic byte checking |
| **XSS in user input** | `ProjectInfo.Create()` sanitizes `<`, `>`, `"` |
| **Path traversal** | Regex whitelist + URL decode before check |
| **JWT theft/replay** | 15-min access, refresh rotation, `jti` blacklist |
| **SSRF via AI callbacks** | Pin base URLs in config, `HttpClient.BaseAddress` |
| **Timing attacks on auth** | `CryptographicOperations.FixedTimeEquals` |
| **Proxy credential leak** | HTTPS_PROXY at container level, CONNECT method for TLS |

### 8.2 Input Validation Boundaries

```
User Input -> API Endpoint -> FluentValidation -> Value Object.Create() -> Domain Logic
              ^                 ^                    ^
              Body size limit   Schema check         Sanitization + business rules
```

### 8.3 Image Proxy Security

```csharp
group.MapGet("/img/{**path}", async (string path, IStorageClient storage, CancellationToken ct) =>
{
    var decoded = Uri.UnescapeDataString(path);
    if (decoded.Contains("..") || decoded.Contains('\\') ||
        !Regex.IsMatch(decoded, @"^[a-zA-Z0-9\-_/]+\.(png|jpg|jpeg|webp)$"))
        return Results.BadRequest("Invalid path");

    var result = await storage.GetImageAsync(decoded, ct);
    return result.Match(
        file => Results.File(file.Data.ToArray(), file.ContentType, enableRangeProcessing: true),
        _ => Results.NotFound());
}).CacheOutput(p => p.Expire(TimeSpan.FromDays(365)));
```

---

## 9. Performance Optimization

### 9.1 Memory Management

```csharp
// Use ArrayPool for image buffers (avoid LOH fragmentation)
var buffer = ArrayPool<byte>.Shared.Rent(imageSize);
try { /* process image */ }
finally { ArrayPool<byte>.Shared.Return(buffer); }
```

### 9.2 Image Pipeline Concurrency

```csharp
// Limit parallel AI image requests to 4 (not 14 at once)
var semaphore = new SemaphoreSlim(4);
var tasks = imageJobs.Select(async job =>
{
    await semaphore.WaitAsync(ct);
    try { return await GenerateImageAsync(job, ct); }
    finally { semaphore.Release(); }
});
await Task.WhenAll(tasks);
```

### 9.3 Caching Strategy

| Layer | Store | TTL | Data |
|-------|-------|-----|------|
| **Output cache** | In-memory | 365 days | Image proxy (immutable) |
| **Response cache** | Redis | 1 hour | Generated site pages |
| **AI response cache** | Redis | 24 hours | Identical prompt results |
| **Static data** | In-memory | Infinite | Style options, color presets, industry aliases |

### 9.4 Database Optimization

- **Npgsql connection pooling** (100 connections default)
- **Select only needed columns** (never `select *` with large JSONB)
- **Read replicas** for site rendering (eventual consistency OK)
- **Indexes**: `owner_email`, `created_at`, `is_active`

### 9.5 Async Patterns

```csharp
// ConfigureAwait(false) in all library code
public async Task<Result<T>> GenerateAsync(CancellationToken ct)
{
    var response = await httpClient.PostAsync(url, content, ct).ConfigureAwait(false);
    // ...
}

// ValueTask for hot paths
public ValueTask<RateLimitResult> CheckAsync(ClientIp ip, RateLimitPolicy policy) { ... }
```

### 9.6 Proxy for Russia (Gemini API)

```csharp
// HttpClient with proxy support
builder.Services.AddHttpClient<IAiClient, GeminiClient>()
    .ConfigurePrimaryHttpMessageHandler(() =>
    {
        var handler = new HttpClientHandler();
        var proxyUrl = Environment.GetEnvironmentVariable("HTTPS_PROXY");
        if (!string.IsNullOrEmpty(proxyUrl))
        {
            handler.Proxy = new WebProxy(proxyUrl);
            handler.UseProxy = true;
        }
        return handler;
    });
```

---

## 10. Docker & CI/CD

### docker-compose.yml

```yaml
services:
  app:
    build: .
    ports: ["5000:8080"]
    environment:
      - ConnectionStrings__Default=Host=postgres;Database=webai;Username=webai;Password=${DB_PASSWORD}
      - S3__Endpoint=http://minio:9000
      - S3__Bucket=webai-images
      - Ai__ApiKey=${GEMINI_API_KEY}
      - HTTPS_PROXY=${HTTPS_PROXY}
    depends_on:
      postgres: { condition: service_healthy }

  postgres:
    image: postgres:17
    environment: { POSTGRES_DB: webai, POSTGRES_USER: webai, POSTGRES_PASSWORD: ${DB_PASSWORD} }
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U webai"], interval: 5s }

  minio:
    image: minio/minio
    command: server /data --console-address ":9001"
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

volumes: { pgdata:, miniodata: }
```

### Dockerfile (multi-stage)

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.sln .
COPY src/WebAI.Domain/*.csproj src/WebAI.Domain/
COPY src/WebAI.Application/*.csproj src/WebAI.Application/
COPY src/WebAI.Infrastructure/*.csproj src/WebAI.Infrastructure/
COPY src/WebAI.Api/*.csproj src/WebAI.Api/
RUN dotnet restore
COPY . .
RUN dotnet publish src/WebAI.Api -c Release -o /app --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /app .
USER $APP_UID
EXPOSE 8080
ENTRYPOINT ["dotnet", "WebAI.Api.dll"]
```

### GitHub Actions CI/CD

```yaml
name: CI
on: [push, pull_request]
jobs:
  build-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env: { POSTGRES_DB: webai_test, POSTGRES_USER: test, POSTGRES_PASSWORD: test }
        ports: ["5432:5432"]
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-dotnet@v6
        with: { dotnet-version: '10.0.x' }
      - run: dotnet build --verbosity minimal
      - run: dotnet test --verbosity minimal
      - run: dotnet publish src/WebAI.Api -c Release -o publish

  docker:
    needs: build-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: docker/login-action@v3
        with: { registry: ghcr.io, username: ${{ github.actor }}, password: ${{ secrets.GITHUB_TOKEN }} }
      - uses: docker/build-push-action@v6
        with: { push: true, tags: "ghcr.io/${{ github.repository }}:latest" }
```

### Health Checks + Telemetry

```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(connectionString, name: "postgresql")
    .AddRedis(redisConnection, name: "redis")
    .AddCheck<S3HealthCheck>("s3-storage")
    .AddCheck<GeminiHealthCheck>("gemini-api");

builder.Services.AddOpenTelemetry()
    .WithTracing(t => t.AddAspNetCoreInstrumentation().AddHttpClientInstrumentation().AddNpgsql())
    .WithMetrics(m => m.AddAspNetCoreInstrumentation().AddMeter("WebAI.ImageGeneration"));
```

---

## 11. Migration Plan

| Phase | Duration | Scope |
|-------|----------|-------|
| **1. Foundation** | Day 1 | Domain: Result, Error, Entity, AggregateRoot, Enums, Value Objects |
| **2. Core Aggregate** | Day 2 | Site entity, SiteChanges, SiteEditPolicy, ISiteRepository, Events |
| **3. Supporting Domains** | Day 3 | ContentGen, ImageGen, Notification, RateLimiting interfaces |
| **4. Infrastructure** | Day 4-5 | EF Core + PostgreSQL, S3 client, Gemini client with Polly |
| **5. API** | Day 6-7 | Minimal API endpoints, middleware pipeline, rate limiting |
| **6. Auth** | Day 8 | JWT service, admin endpoints, refresh token rotation |
| **7. Background** | Day 9 | Image generation queue (Channel<T> + IHostedService) |
| **8. Docker + CI/CD** | Day 10 | Dockerfile, docker-compose, GitHub Actions |
| **9. Tests** | Day 11-12 | Unit tests for domain, integration tests for API |
| **10. Frontend** | Day 13+ | Port Next.js frontend or serve as SPA from .NET |

---

## 12. Legacy Next.js Security Audit (historical)

> [!warning] Исторический раздел — все пункты закрыты
> Это реальный результат аудита legacy-версии (Next.js + Supabase) **до** миграции. CRITICAL/HIGH закрыты hardening-проходом (апрель 2026), MEDIUM — снят миграцией на .NET (см. разделы 7–8: rate limiting, path traversal, constant-time сравнение, body limits). Оставлено как учебный пример типовых дыр вайб-кодовой базы — и как чек-лист для аудита чужого Next.js-проекта.

### CRITICAL

1. **`edit_token` leaked to every visitor** — `page.tsx` does `select("*")`, exposes edit token in React hydration payload
2. **Real credentials in `.env.local`** — Gemini API key, admin password visible on disk

### HIGH

3. **SSRF via self-fetch** — `submit-lead.ts` derives base URL from `x-forwarded-host` header
4. **No request body size limits** — all API routes accept unlimited JSON
5. **`edit_token` comparison is not constant-time** — `edit/[id]/actions.ts` uses `!==`
6. **No rate limiting on admin login**

### MEDIUM

7. **Rate limiter is in-memory only** — bypassed in multi-instance deployment
8. **Path traversal incomplete** — no null byte, backslash, or URL-encoded checks
9. **Social links not URL-validated** — potential `javascript:` URLs
10. **CSP allows `'unsafe-inline'`** — weakens script injection protection

### Порядок, в котором чинили

```
Day 1: Rotate credentials, fix select("*"), fix SSRF
Day 2: Add body size limits, constant-time token comparison, admin rate limiting
Day 3: Redis rate limiting, path traversal hardening, proxy support for Gemini
```
