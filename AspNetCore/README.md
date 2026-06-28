# AspNetCore — web framework

> 23 файла / ~678 KB. ASP.NET Core: API, auth, middleware, HttpClient, real-time, Blazor, AOT, raw Kestrel.

[← Главный README](../readme.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Не понимаю HTTP | [`Junior/http-fundamentals.md`](Junior/http-fundamentals.md) |
| Делаю первое API | [`Middle/aspnet-controllers-routing.md`](Middle/aspnet-controllers-routing.md) → [`Senior/api-design.md`](Senior/api-design.md) |
| DI не понимаю | [`Middle/aspnet-dependency-injection-deep.md`](Middle/aspnet-dependency-injection-deep.md) |
| Хожу во внешние API | [`Middle/http-client-resilience.md`](Middle/http-client-resilience.md) ⭐ NEW |
| Auth / JWT | [`Senior/auth-security.md`](Senior/auth-security.md) |
| Нужно кеширование | [`Senior/caching.md`](Senior/caching.md) |
| Background jobs | [`Senior/hosting-background.md`](Senior/hosting-background.md) |
| Real-time (chat, notifications) | [`Senior/signalr.md`](Senior/signalr.md) |

---

## 📚 Все 23 файла

### 🌱 Junior

| Файл | Описание |
|------|----------|
| [`http-fundamentals.md`](Junior/http-fundamentals.md) | HTTP протокол: методы, статусы, заголовки, REST-основы |

### 🌿 Middle

| Файл | Описание |
|------|----------|
| [`aspnet-controllers-routing.md`](Middle/aspnet-controllers-routing.md) | Controllers, Minimal API, routing, model binding |
| [`aspnet-dependency-injection-deep.md`](Middle/aspnet-dependency-injection-deep.md) | DI lifetimes, captive dependencies, keyed services |
| [`aspnet-error-handling.md`](Middle/aspnet-error-handling.md) | IExceptionHandler, ProblemDetails, error contracts |
| [`aspnet-rate-limiting.md`](Middle/aspnet-rate-limiting.md) | Rate limiting middleware, алгоритмы, partition keys |
| [`fluent-validation.md`](Middle/fluent-validation.md) | FluentValidation: правила, DI, тестирование |
| [`http-client-resilience.md`](Middle/http-client-resilience.md) ⭐ NEW | HttpClient, IHttpClientFactory, socket exhaustion, stale DNS, retry/breaker |
| [`object-mapping.md`](Middle/object-mapping.md) | AutoMapper vs Mapster vs ручной mapping |

### 🏆 Senior

| Файл | Описание |
|------|----------|
| [`api-design.md`](Senior/api-design.md) | REST design, versioning, OpenAPI, идемпотентность |
| [`pipeline-middleware.md`](Senior/pipeline-middleware.md) | Request pipeline, custom middleware |
| [`di-configuration.md`](Senior/di-configuration.md) | DI + Options pattern + IConfiguration |
| [`auth-security.md`](Senior/auth-security.md) | JWT, OAuth, OIDC, RBAC ⭐ |
| [`security-practices.md`](Senior/security-practices.md) | OWASP, CSP, secure headers |
| [`caching.md`](Senior/caching.md) | IDistributedCache, OutputCaching, Redis |
| [`logging-observability.md`](Senior/logging-observability.md) | Serilog, structured logging |
| [`resilience.md`](Senior/resilience.md) | Polly v8: стратегии retry/breaker/hedging |
| [`hosting-background.md`](Senior/hosting-background.md) | BackgroundService, IHostedService |
| [`signalr.md`](Senior/signalr.md) | SignalR, WebSockets, Redis backplane |
| [`graphql.md`](Senior/graphql.md) | HotChocolate, schemas, federation |
| [`blazor-server.md`](Senior/blazor-server.md) | Blazor Server architecture |
| [`blazor-wasm.md`](Senior/blazor-wasm.md) | Blazor WebAssembly |
| [`native-aot.md`](Senior/native-aot.md) | Native AOT compilation, trimming |
| [`kestrel-as-raw-host.md`](Senior/kestrel-as-raw-host.md) ⭐ NEW | Kestrel как raw HTTP host: CreateSlimBuilder, catch-all routing, transport-agnostic |

---

## 🔗 Связанные папки

- [`Architecture/`](../Architecture/) — CQRS/MediatR поверх ASP.NET
- [`EFCore/`](../EFCore/) — data access в ASP.NET
- [`Infrastructure/`](../Infrastructure/) — observability, messaging, deploy
- [`Performance/`](../Performance/) — caching strategies, async performance
