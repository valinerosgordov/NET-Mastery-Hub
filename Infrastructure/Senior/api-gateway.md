---
tags: [infrastructure, api-gateway, yarp, ocelot, nginx, microservices, bff, senior]
level: Senior
date: 2026-05-01
---

# API Gateway — YARP, Ocelot, NGINX

> **Single entry point для микросервисов.** Routing, auth, rate limiting, transformation — в одном месте. Closes пробел "знаю про microservices, но не понимаю как frontend общается с 10+ сервисами".

---

## Что это, зачем и когда

### Проблема без gateway

```
Frontend
  ├─ /users   → users-service.com:5001
  ├─ /orders  → orders-service.com:5002
  ├─ /payments → payments-service.com:5003
  └─ /catalog → catalog-service.com:5004
```

Frontend знает 10+ endpoints, ports, auth schemes. Каждый сервис открыт публично. CORS на каждом. Auth dublicated.

### С gateway

```
Frontend → api.myapp.com (Gateway)
              ├─ /api/users    → internal users-service
              ├─ /api/orders   → internal orders-service
              ├─ /api/payments → internal payments-service
              └─ /api/catalog  → internal catalog-service
```

**Gateway responsibilities:**
- **Routing** — какой URL → какой сервис
- **Auth** — single auth check (JWT validation один раз)
- **Rate limiting** — global + per-user
- **Caching** — общий cache layer
- **Transformation** — modify request/response
- **Aggregation** — собрать ответ из нескольких сервисов
- **Logging / metrics** — все requests в одном месте

### Когда нужен Gateway

✅ **Используй когда:**
- Microservices (3+ сервисов)
- Multiple frontends (web, mobile, partners) с разными needs
- Need cross-cutting concerns (auth, rate limit, logging)
- Public API + internal API

❌ **НЕ нужен когда:**
- Monolith
- 1-2 сервисов (overkill)
- Прямой service-to-service communication (используй mesh)

См. [[microservices-vs-monolith|Microservices vs Monolith]].

---

## 1. YARP — Microsoft's reverse proxy

### Что это

**Yet Another Reverse Proxy** — .NET-native, от Microsoft. **Самый perfomance** vs Ocelot. Активно развивается.

### Установка

```bash
dotnet add package Yarp.ReverseProxy
```

### Базовый setup

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

app.MapReverseProxy();
app.Run();
```

```json
// appsettings.json
{
  "ReverseProxy": {
    "Routes": {
      "users-route": {
        "ClusterId": "users-cluster",
        "Match": { "Path": "/api/users/{**catch-all}" }
      },
      "orders-route": {
        "ClusterId": "orders-cluster",
        "Match": { "Path": "/api/orders/{**catch-all}" }
      }
    },
    "Clusters": {
      "users-cluster": {
        "Destinations": {
          "primary": { "Address": "http://users-service:5001/" }
        }
      },
      "orders-cluster": {
        "Destinations": {
          "primary": { "Address": "http://orders-service:5002/" }
        }
      }
    }
  }
}
```

### Load balancing

```json
"orders-cluster": {
  "LoadBalancingPolicy": "RoundRobin",  // или Random, LeastRequests, PowerOfTwoChoices
  "Destinations": {
    "instance1": { "Address": "http://orders-1:5002/" },
    "instance2": { "Address": "http://orders-2:5002/" },
    "instance3": { "Address": "http://orders-3:5002/" }
  }
}
```

### Health checks

```json
"orders-cluster": {
  "HealthCheck": {
    "Active": {
      "Enabled": true,
      "Interval": "00:00:10",
      "Timeout": "00:00:05",
      "Policy": "ConsecutiveFailures",
      "Path": "/health"
    }
  }
}
```

Unhealthy instance автоматически removed из rotation.

### Transforms — modify request/response

```json
"users-route": {
  "ClusterId": "users-cluster",
  "Match": { "Path": "/api/users/{**catch-all}" },
  "Transforms": [
    { "PathPattern": "/internal/v2/users/{**catch-all}" },
    { "RequestHeader": "X-Forwarded-For", "Set": "{RemoteIpAddress}" },
    { "ResponseHeader": "X-Service", "Set": "users-v2" }
  ]
}
```

### Authentication перед routing

```csharp
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer(options => { /* JWT config */ });

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapReverseProxy();
```

```json
"users-route": {
  "AuthorizationPolicy": "default",  // требует authenticated user
  ...
}
```

---

## 2. Ocelot — popular, простой

### Что это

Старее YARP, проще для start, но slower. До 2022 был default choice. Сейчас **YARP recommended**.

### Setup

```bash
dotnet add package Ocelot
```

```csharp
builder.Configuration.AddJsonFile("ocelot.json");
builder.Services.AddOcelot();

await app.UseOcelot();
```

```json
// ocelot.json
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/users/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        { "Host": "users-service", "Port": 5001 }
      ],
      "UpstreamPathTemplate": "/api/users/{everything}",
      "UpstreamHttpMethod": [ "Get", "Post", "Put", "Delete" ],
      "RateLimitOptions": {
        "ClientWhitelist": [],
        "EnableRateLimiting": true,
        "Period": "1m",
        "PeriodTimespan": 60,
        "Limit": 100
      }
    }
  ]
}
```

### YARP vs Ocelot

| | YARP | Ocelot |
|--|------|--------|
| **Vendor** | Microsoft | Community |
| **Performance** | 2-3x faster | Slower |
| **Active** | Very active | Slower releases |
| **Features** | Comprehensive | Comprehensive |
| **Production-ready** | ✅ | ✅ |
| **Recommended 2026** | ✅ | △ legacy |

См. бенчмарки на github.com/microsoft/reverse-proxy.

---

## 3. NGINX как gateway

### Когда NGINX вместо YARP/Ocelot

NGINX — universal reverse proxy. Хорошо для:
- **Существующая инфраструктура** уже на NGINX
- **Static content** + API на одном хосте
- **TLS termination** efficient
- **Mixed stacks** (.NET + Node + Python)

### Базовый config

```nginx
# nginx.conf
events {}

http {
    upstream users_service {
        server users-service:5001;
        server users-service-2:5001;
    }
    
    upstream orders_service {
        server orders-service:5002;
    }
    
    server {
        listen 443 ssl;
        server_name api.myapp.com;
        
        ssl_certificate /etc/ssl/cert.pem;
        ssl_certificate_key /etc/ssl/key.pem;
        
        location /api/users/ {
            proxy_pass http://users_service/;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
        
        location /api/orders/ {
            proxy_pass http://orders_service/;
        }
        
        # Rate limit
        limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
        location /api/ {
            limit_req zone=api burst=20;
        }
    }
}
```

### Минусы NGINX

- ❌ Config на nginx-specific syntax (не C#)
- ❌ Deploy / version separately
- ❌ Сложно integrate с .NET auth/identity
- ❌ Reload config = network blip

**Для .NET-only stack — YARP лучше.**

---

## 4. BFF Pattern — Backend for Frontend

### Идея

**Один gateway не подходит для всех frontends.** Web и mobile имеют разные нужды:

- **Web** — большие responses, много данных, hot caching
- **Mobile** — minimal data (battery/network), aggregated views
- **Partner API** — versioned, throttled, audit

### Решение — отдельные gateways

```
Web frontend     → Web BFF Gateway      → microservices
Mobile app       → Mobile BFF Gateway   → microservices
Partners         → Partner Gateway       → microservices (с extra throttling)
```

Каждый BFF — **специфичный** под свой client:
- Web BFF: aggregates 5 сервисов в один response
- Mobile BFF: returns minimal fields (40% data)
- Partner: версионирование, audit logging

### YARP пример — Mobile BFF aggregation

```csharp
// Mobile BFF endpoint — aggregates user + orders + recommendations
app.MapGet("/api/mobile/dashboard", async (
    int userId,
    HttpClient client) =>
{
    var userTask = client.GetFromJsonAsync<User>($"http://users-service/api/users/{userId}");
    var ordersTask = client.GetFromJsonAsync<List<Order>>($"http://orders-service/api/orders?userId={userId}&take=5");
    var recsTask = client.GetFromJsonAsync<List<Product>>($"http://recommendations-service/api/recs?userId={userId}");
    
    await Task.WhenAll(userTask, ordersTask, recsTask);
    
    // Mobile-specific minimal DTO
    return Results.Ok(new
    {
        user = new { userTask.Result.Name, userTask.Result.AvatarUrl },
        recentOrders = ordersTask.Result.Select(o => new { o.Id, o.Total, o.Status }),
        recommendations = recsTask.Result.Take(3).Select(p => new { p.Id, p.Name, p.Price })
    });
});
```

**Web BFF тот же data — но с full details.** Mobile меньше payload, web комплексные responses.

См. [[microservices-vs-monolith|Microservices]].

---

## 5. Service Mesh vs API Gateway

### Различие

| | API Gateway | Service Mesh |
|--|-------------|--------------|
| **Где** | Edge (north-south traffic) | Cluster internal (east-west) |
| **Кто использует** | External clients (frontend, mobile, partners) | Internal services |
| **Concerns** | Auth, rate limit, transformation | Service discovery, mTLS, retries, observability |
| **Examples** | YARP, Ocelot, Nginx, Kong | Istio, Linkerd, Consul Connect |
| **Configuration** | Centralized | Sidecar pattern (proxy per service) |

```
External Traffic
    ↓
┌───────────────┐
│ API Gateway   │  ← auth, rate limit, routing
│ (YARP/Nginx)  │
└───────┬───────┘
        ↓
┌───────────────┐
│ Service Mesh  │  ← service discovery, mTLS, retries
│ (Istio/Linkerd) │
│  ┌───┐ ┌───┐  │
│  │ S1 │ │ S2 │ │
│  └───┘ └───┘  │
└───────────────┘
```

В small microservices — gateway достаточно. В larger (50+ services) — оба.

См. [[kubernetes|Kubernetes]].

---

## 6. Case Study #1 — Migration монолита → microservices

**Сценарий:** ASP.NET Core monolith, 200K LOC. Решено разбить на microservices без big-bang rewrite.

### Strangler Fig pattern + Gateway

```
Phase 1: Gateway добавлен перед монолитом
─────────────────────────────────────────
Frontend → Gateway → Monolith (всё)


Phase 2: Один module вынесен (Users)
─────────────────────────────────────
Frontend → Gateway ┬─ /api/users   → Users service (new)
                   └─ /api/*        → Monolith (rest)


Phase 3-N: Постепенно остальные modules
─────────────────────────────────────
Frontend → Gateway ┬─ /api/users    → Users service
                   ├─ /api/orders   → Orders service
                   ├─ /api/payments → Payments service
                   └─ /api/legacy   → Monolith (что осталось)
```

### YARP config

```json
{
  "ReverseProxy": {
    "Routes": {
      "users-route": {
        "ClusterId": "users-cluster",
        "Match": { "Path": "/api/users/{**catch-all}" },
        "Order": 1
      },
      "orders-route": {
        "ClusterId": "orders-cluster",
        "Match": { "Path": "/api/orders/{**catch-all}" },
        "Order": 2
      },
      "monolith-fallback": {
        "ClusterId": "monolith-cluster",
        "Match": { "Path": "{**catch-all}" },
        "Order": 99   // последний — fallback
      }
    },
    "Clusters": {
      "users-cluster": { "Destinations": { "d1": { "Address": "http://users-svc/" } } },
      "orders-cluster": { "Destinations": { "d1": { "Address": "http://orders-svc/" } } },
      "monolith-cluster": { "Destinations": { "d1": { "Address": "http://monolith/" } } }
    }
  }
}
```

**Преимущество:** frontend не меняется. Backend постепенно мигрирует.

См. [[microservices-vs-monolith|Migration strategy]].

---

## 7. Case Study #2 — Multi-version API

**Сценарий:** API v1 в production, v2 — новый. Mobile старые версии используют v1, новые — v2. Postepenно migrate.

### Через gateway

```json
{
  "Routes": {
    "users-v1": {
      "ClusterId": "users-v1-cluster",
      "Match": { "Path": "/api/v1/users/{**catch-all}" }
    },
    "users-v2": {
      "ClusterId": "users-v2-cluster",
      "Match": { "Path": "/api/v2/users/{**catch-all}" }
    },
    "users-default": {
      "ClusterId": "users-v2-cluster",
      "Match": { "Path": "/api/users/{**catch-all}" },
      "Transforms": [
        { "PathPattern": "/api/v2/users/{**catch-all}" }  // default → v2
      ]
    }
  }
}
```

**Result:** Mobile v1.0 → /api/v1/users, mobile v2.0 → /api/v2/users, web → /api/users (= v2).

См. [[api-design|API Design]].

---

## 8. Case Study #3 — Rate limiting per tier

**Сценарий:** Public API с тарифами Free / Pro / Enterprise. Разные rate limits.

### YARP с custom middleware

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddPolicy("tier-based", context =>
    {
        var apiKey = context.Request.Headers["X-API-Key"].FirstOrDefault();
        var tier = ResolveTier(apiKey);  // DB lookup или JWT claim
        
        return tier switch
        {
            "Enterprise" => RateLimitPartition.GetNoLimiter(apiKey),
            "Pro" => RateLimitPartition.GetFixedWindowLimiter(apiKey, _ => 
                new FixedWindowRateLimiterOptions
                {
                    PermitLimit = 1000,
                    Window = TimeSpan.FromMinutes(1)
                }),
            _ => RateLimitPartition.GetFixedWindowLimiter(apiKey, _ => 
                new FixedWindowRateLimiterOptions
                {
                    PermitLimit = 60,
                    Window = TimeSpan.FromMinutes(1)
                })
        };
    });

    options.OnRejected = async (ctx, ct) =>
    {
        ctx.HttpContext.Response.StatusCode = 429;
        ctx.HttpContext.Response.Headers["Retry-After"] = "60";
        await ctx.HttpContext.Response.WriteAsync("Rate limit exceeded. Upgrade tier для higher limits.", ct);
    };
});

app.UseRateLimiter();
app.MapReverseProxy().RequireRateLimiting("tier-based");
```

См. [[http-fundamentals|HTTP Fundamentals]].

---

## 9. Case Study #4 — Authentication aggregation

**Сценарий:** 5 microservices, каждый раньше валидировал JWT отдельно. Дубликат кода + если auth логика меняется — 5 deploys.

### ❌ Без gateway

```csharp
// Каждый сервис:
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer(options => { /* JWT validation */ });

// + middleware для verification
// + separate updates для каждого сервиса
```

### ✅ Auth в gateway

```csharp
// Gateway — единственное место для JWT validation
builder.Services.AddAuthentication("Bearer")
    .AddJwtBearer(options => 
    {
        options.TokenValidationParameters = new() { /* ... */ };
    });

builder.Services.AddAuthorization();

// Gateway extracts claims → forwards as headers
app.UseAuthentication();
app.UseAuthorization();

app.Use(async (context, next) =>
{
    if (context.User.Identity?.IsAuthenticated == true)
    {
        var userId = context.User.FindFirst("sub")?.Value;
        var roles = string.Join(",", context.User.FindAll("role").Select(c => c.Value));
        
        context.Request.Headers["X-User-Id"] = userId;
        context.Request.Headers["X-User-Roles"] = roles;
    }
    await next();
});

app.MapReverseProxy();
```

```csharp
// Microservices — просто читают X-User-Id header
[HttpGet]
public IActionResult GetMyOrders()
{
    var userId = Request.Headers["X-User-Id"].FirstOrDefault();
    return Ok(_service.GetForUser(userId));
}
```

**Service не знает JWT, OAuth, validation. Только headers.**

⚠️ **Важно:** services должны быть **только** в private network (за gateway). Если кто-то bypass'нул gateway — auth не проверится. **Network policies obligatorny.**

См. [[auth-security|Auth & Security]].

---

## 10. Case Study #5 — Caching common responses

**Сценарий:** `/api/products` запрашивается 10K RPS. Каждый раз hits catalog-service + DB. Очень expensive.

### Gateway-level caching

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("products-cache", builder =>
    {
        builder.Cache()
               .Expire(TimeSpan.FromMinutes(5))
               .Tag("products");
    });
});

app.UseOutputCache();

// Apply policy to specific routes
app.MapReverseProxy(proxyPipeline =>
{
    proxyPipeline.Use(async (context, next) =>
    {
        if (context.Request.Path.StartsWithSegments("/api/products"))
        {
            // Apply cache
        }
        await next();
    });
});
```

**Результат:**
- 10K RPS → catalog-service: 10K → ~100 RPS (1% miss rate)
- catalog-service load: -99%
- p50 latency: 50 ms → 2 ms (cached)

**Invalidation:**
```csharp
// Когда product updated — invalidate cache
await outputCacheStore.EvictByTagAsync("products", default);
```

См. [[caching|Caching]].

---

## 11. Case Study #6 — YARP vs Nginx benchmark

**Сценарий:** Greenfield project, выбираем gateway. Benchmark на real workload.

### Setup

- 4 microservices behind gateway
- 1000 RPS load test
- 100 концurrent users
- Hardware: 4 vCPU, 8 GB RAM

### Результаты

```
Metric          YARP              Nginx              Ocelot
────────────────────────────────────────────────────────────
RPS             95,000            120,000            45,000
p50 latency     2.1 ms            1.5 ms             4.8 ms
p99 latency     8 ms              5 ms               25 ms
CPU @ 1K RPS    8%                4%                 18%
Memory          180 MB            50 MB              250 MB
```

### Verdict

- **NGINX** — fastest на raw bytes, низкий memory footprint
- **YARP** — close second, но deeply integrated с .NET (auth, DI, logging)
- **Ocelot** — slowest, deprecated direction

**Decision:**
- **Pure performance** → Nginx
- **.NET ecosystem** (auth, observability, custom logic) → YARP
- **Greenfield 2026** → YARP

---

## 12. Common Pitfalls

### 1. Gateway как single point of failure

```
Frontend → Gateway (1 instance) → services
              ↓ Gateway crashes
              ↓
          Всё недоступно
```

**Защита:**
- Gateway в minimum 2 replicas (k8s deployment)
- Health checks + auto-restart
- Не deploy критичные изменения без canary

### 2. Hardcoded URLs в config

```json
// ❌ Hardcoded prod URL
"users-service": "http://users-prod-eastus.cloudapp.azure.com:5001"

// ✅ Service discovery (k8s) или config per env
"users-service": "http://users-service:5001"  // k8s service name
```

### 3. Auth перед routing — bypass

```
Если service напрямую достижим (минуя gateway):
  Attacker → service:5001/api/admin → НЕТ auth check!
```

**Защита:** services только в private network. Public IP только у gateway.

### 4. Circular dependencies через gateway

```
Service A → Gateway → Service B
Service B → Gateway → Service A
```

**Симптом:** retries увеличиваются, request loop.

**Защита:** internal service-to-service direct (минуя gateway), gateway только для external traffic.

### 5. Slow gateway = slow всё

Gateway добавляет ~1-5 ms на каждый request. Если gateway slow (плохо tuned) — всё затем slow.

**Profiling:**
- gateway отдельно (метрики)
- baseline без gateway

### 6. Gateway as god-object

```csharp
// ❌ Gateway accumulates business logic
app.MapPost("/api/checkout", async (Order o) =>
{
    // Validate
    // Calculate tax  
    // Charge payment
    // Send email
    // ...
});
```

Gateway должен быть **thin** — routing, auth, rate limit. Business logic — в services.

### 7. Не использовать health checks

Без health checks — gateway routes к dead instances → 502 errors для users.

```json
"HealthCheck": {
  "Active": { "Enabled": true, "Path": "/health", "Interval": "00:00:05" }
}
```

### 8. CORS на gateway + на services

```csharp
// Gateway:
app.UseCors("AllowFrontend");

// Service (за gateway):
app.UseCors("AllowFrontend");  // dublicate!
```

CORS только на gateway. Services internal — CORS не нужен.

### 9. Не logging request ID

```csharp
// Gateway добавляет correlation ID
app.Use(async (context, next) =>
{
    var correlationId = context.Request.Headers["X-Correlation-ID"].FirstOrDefault() ?? Guid.NewGuid().ToString();
    context.Request.Headers["X-Correlation-ID"] = correlationId;
    context.Response.Headers["X-Correlation-ID"] = correlationId;
    await next();
});
```

Без этого — невозможно trace request через services.

См. [[observability|Observability]].

### 10. Игнорировать timeout configuration

```json
"Clusters": {
  "slow-service": {
    "HttpRequest": {
      "ActivityTimeout": "00:00:30"  // если service вернул через 31 сек — timeout
    }
  }
}
```

Default — 100 секунд. Если service hangs — gateway thread blocked. Configure per-service.

---

## 13. Best Practices

### Architecture

- **YARP для .NET stack** (recommended 2026)
- **NGINX если уже есть инфраструктура**
- **Gateway thin** — routing/auth/rate limit, не business logic
- **Multi-replica** — minimum 2 instances
- **Health checks** для каждого backend cluster
- **Internal services в private network**

### Security

- **Auth в gateway** — single point validation
- **Forward claims** через headers (X-User-Id)
- **TLS termination** на gateway
- **Rate limiting** обязательно (per-IP минимум)
- **OWASP headers** (CSP, HSTS, X-Content-Type-Options)
- **API Keys для public API** + tier-based limits

### Performance

- **Caching на gateway** для hot endpoints
- **Connection pooling** (default в YARP)
- **HTTP/2 везде**
- **Gzip / Brotli compression**
- **Profiling регулярно**

### Observability

- **Correlation ID** обязательно (X-Correlation-ID)
- **Structured logging** всех requests
- **Metrics** per route (latency, errors, throughput)
- **Distributed tracing** (OpenTelemetry)

См. [[observability|Observability]] и [[security-practices|Security]].

---

## 14. Cheat sheet

| Need | Solution |
|------|----------|
| Single entry point для microservices | Gateway (YARP) |
| Different APIs для web/mobile | BFF pattern (multiple gateways) |
| Auth в одном месте | Gateway with JWT validation |
| Rate limiting per user | Gateway + AddRateLimiter |
| Caching common responses | Gateway + OutputCache |
| Migrate monolith → microservices | Strangler Fig + Gateway |
| Multi-version API | Gateway routes per version |
| Service-to-service в cluster | Service mesh (Istio/Linkerd) — не gateway |
| Public API с tiers | Gateway + tier-based rate limit |
| Internal load balancing | Gateway cluster или k8s service |

---

## 15. Decision tree

```
Нужен ли API Gateway?
│
├── Monolith?
│   → НЕТ gateway, ASP.NET достаточно
│
├── 1-2 микросервиса?
│   → Может НЕ нужен. Прямой routing достаточен
│
├── 3+ microservices?
│   ├── .NET stack → YARP
│   ├── Mixed languages → Nginx или Kong
│   └── Cloud-native → Cloud Gateway (Azure APIM, AWS API Gateway)
│
├── Multiple frontends с разными needs?
│   → BFF pattern — отдельные gateways
│
└── Service-to-service внутри cluster?
    → Service mesh (Istio/Linkerd), не gateway
```

---

## 16. См. также

- [[microservices-vs-monolith|Microservices vs Monolith]]
- [[distributed-systems|Distributed Systems]]
- [[auth-security|Auth & Security]] — JWT в gateway
- [[api-design|API Design]] — REST principles
- [[caching|Caching]] — gateway cache
- [[http-fundamentals|HTTP Fundamentals]] — rate limit, status codes
- [[kubernetes|Kubernetes]] — service mesh integration
- [[observability|Observability]] — correlation IDs, tracing
- [[twelve-factor-app|Twelve Factor App]] (TBD)

## 17. Reading list

- **YARP documentation** — microsoft.github.io/reverse-proxy
- **Ocelot documentation** — ocelot.readthedocs.io
- **NGINX as reverse proxy** — docs.nginx.com/nginx
- **Building Microservices** — Sam Newman (chapter про gateway)
- **Microservices Patterns** — Chris Richardson (API Gateway pattern)
- **Microsoft Architecture Guide — API Gateway** — learn.microsoft.com/azure/architecture/microservices/design/gateway
- **Kong vs AWS API Gateway vs Azure APIM** — architecture comparison articles
