# AspNetCore — web framework

> 14 файлов / 392 KB. ASP.NET Core: API, auth, middleware, real-time, modern features.

[← Главный README](../README.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Делаю первое API | [`api-design.md`](api-design.md) → [`pipeline-middleware.md`](pipeline-middleware.md) |
| Auth / JWT | [`auth-security.md`](auth-security.md) |
| DI не понимаю | [`di-configuration.md`](di-configuration.md) |
| Нужно кеширование | [`caching.md`](caching.md) |
| Background jobs | [`hosting-background.md`](hosting-background.md) |
| Real-time (chat, notifications) | [`signalr.md`](signalr.md) |

---

## 📚 Все 14 файлов

### Core

| Файл | Описание |
|------|----------|
| [`api-design.md`](api-design.md) | REST design, versioning, OpenAPI |
| [`pipeline-middleware.md`](pipeline-middleware.md) | Request pipeline, custom middleware |
| [`di-configuration.md`](di-configuration.md) | DI lifetimes, configuration system |

### Security

| Файл | Описание |
|------|----------|
| [`auth-security.md`](auth-security.md) | JWT, OAuth, OIDC, RBAC (47 KB) ⭐ |
| [`security-practices.md`](security-practices.md) | OWASP, CSP, secure headers |

### Cross-cutting

| Файл | Описание |
|------|----------|
| [`caching.md`](caching.md) | IDistributedCache, OutputCaching, Redis |
| [`logging-observability.md`](logging-observability.md) | Serilog, structured logging |
| [`resilience.md`](resilience.md) | Polly, retry, circuit breaker |
| [`hosting-background.md`](hosting-background.md) | BackgroundService, IHostedService |

### Real-time / advanced API

| Файл | Описание |
|------|----------|
| [`signalr.md`](signalr.md) | SignalR, WebSockets, real-time |
| [`graphql.md`](graphql.md) | HotChocolate, schemas, federation |

### Frontend integration

| Файл | Описание |
|------|----------|
| [`blazor-server.md`](blazor-server.md) | Blazor Server architecture |
| [`blazor-wasm.md`](blazor-wasm.md) | Blazor WebAssembly |

### Performance

| Файл | Описание |
|------|----------|
| [`native-aot.md`](native-aot.md) | Native AOT compilation |

---

## 🔗 Связанные папки

- [`Architecture/cqrs-mediatr`](../Architecture/cqrs-mediatr.md) — CQRS поверх ASP.NET
- [`EFCore/`](../EFCore/) — data access в ASP.NET
- [`Infrastructure/observability`](../Infrastructure/observability.md) — production monitoring
- [`Performance/caching-strategies`](../Performance/caching-strategies.md) — cache patterns
