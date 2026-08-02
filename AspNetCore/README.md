# AspNetCore — web framework

> 23 файла / ~678 KB. ASP.NET Core: API, auth, middleware, HttpClient, real-time, Blazor, AOT, raw Kestrel.

[[README|← Главный README]] · [[INDEX|Полный INDEX]]

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Не понимаю HTTP | [[http-fundamentals|`Junior/http-fundamentals.md`]] |
| Делаю первое API | [[aspnet-controllers-routing|`Middle/aspnet-controllers-routing.md`]] → [[api-design|`Senior/api-design.md`]] |
| DI не понимаю | [[aspnet-dependency-injection-deep|`Middle/aspnet-dependency-injection-deep.md`]] |
| Хожу во внешние API | [[http-client-resilience|`Middle/http-client-resilience.md`]] ⭐ NEW |
| Auth / JWT | [[auth-security|`Senior/auth-security.md`]] |
| Нужно кеширование | [[caching|`Senior/caching.md`]] |
| Background jobs | [[hosting-background|`Senior/hosting-background.md`]] |
| Real-time (chat, notifications) | [[signalr|`Senior/signalr.md`]] |

---

## 📚 Все 23 файла

### 🌱 Junior

| Файл | Описание |
|------|----------|
| [[http-fundamentals|`http-fundamentals.md`]] | HTTP протокол: методы, статусы, заголовки, REST-основы |

### 🌿 Middle

| Файл | Описание |
|------|----------|
| [[aspnet-controllers-routing|`aspnet-controllers-routing.md`]] | Controllers, Minimal API, routing, model binding |
| [[aspnet-dependency-injection-deep|`aspnet-dependency-injection-deep.md`]] | DI lifetimes, captive dependencies, keyed services |
| [[aspnet-error-handling|`aspnet-error-handling.md`]] | IExceptionHandler, ProblemDetails, error contracts |
| [[aspnet-rate-limiting|`aspnet-rate-limiting.md`]] | Rate limiting middleware, алгоритмы, partition keys |
| [[fluent-validation|`fluent-validation.md`]] | FluentValidation: правила, DI, тестирование |
| [[http-client-resilience|`http-client-resilience.md`]] ⭐ NEW | HttpClient, IHttpClientFactory, socket exhaustion, stale DNS, retry/breaker |
| [[object-mapping|`object-mapping.md`]] | AutoMapper vs Mapperly vs ручной mapping |

### 🏆 Senior

| Файл | Описание |
|------|----------|
| [[api-design|`api-design.md`]] | REST design, versioning, OpenAPI, идемпотентность |
| [[pipeline-middleware|`pipeline-middleware.md`]] | Request pipeline, custom middleware |
| [[di-configuration|`di-configuration.md`]] | DI + Options pattern + IConfiguration |
| [[auth-security|`auth-security.md`]] | JWT, OAuth, OIDC, RBAC ⭐ |
| [[security-practices|`security-practices.md`]] | OWASP, CSP, secure headers |
| [[caching|`caching.md`]] | IDistributedCache, OutputCaching, Redis |
| [[logging-observability|`logging-observability.md`]] | Serilog, structured logging |
| [[resilience|`resilience.md`]] | Polly v8: стратегии retry/breaker/hedging |
| [[hosting-background|`hosting-background.md`]] | BackgroundService, IHostedService |
| [[signalr|`signalr.md`]] | SignalR, WebSockets, Redis backplane |
| [[graphql|`graphql.md`]] | HotChocolate, schemas, federation |
| [[blazor-server|`blazor-server.md`]] | Blazor Server architecture |
| [[blazor-wasm|`blazor-wasm.md`]] | Blazor WebAssembly |
| [[native-aot|`native-aot.md`]] | Native AOT compilation, trimming |
| [[kestrel-as-raw-host|`kestrel-as-raw-host.md`]] ⭐ NEW | Kestrel как raw HTTP host: CreateSlimBuilder, catch-all routing, transport-agnostic |

---

## 🔗 Связанные папки

- [`Architecture/`](../Architecture/) — CQRS/MediatR поверх ASP.NET
- [`EFCore/`](../EFCore/) — data access в ASP.NET
- [`Infrastructure/`](../Infrastructure/) — observability, messaging, deploy
- [`Performance/`](../Performance/) — caching strategies, async performance
