# AspNetCore — web framework

> 14 файлов / 392 KB. ASP.NET Core: API, auth, middleware, real-time, modern features.

[← Главный README]() · [Полный INDEX]()

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Делаю первое API | [`api-design.md`]() → [`pipeline-middleware.md`]() |
| Auth / JWT | [`auth-security.md`]() |
| DI не понимаю | [`di-configuration.md`]() |
| Нужно кеширование | [`caching.md`]() |
| Background jobs | [`hosting-background.md`]() |
| Real-time (chat, notifications) | [`signalr.md`]() |

---

## 📚 Все 14 файлов

### Core

| Файл | Описание |
|------|----------|
| [`api-design.md`]() | REST design, versioning, OpenAPI |
| [`pipeline-middleware.md`]() | Request pipeline, custom middleware |
| [`di-configuration.md`]() | DI lifetimes, configuration system |

### Security

| Файл | Описание |
|------|----------|
| [`auth-security.md`]() | JWT, OAuth, OIDC, RBAC (47 KB) ⭐ |
| [`security-practices.md`]() | OWASP, CSP, secure headers |

### Cross-cutting

| Файл | Описание |
|------|----------|
| [`caching.md`]() | IDistributedCache, OutputCaching, Redis |
| [`logging-observability.md`]() | Serilog, structured logging |
| [`resilience.md`]() | Polly, retry, circuit breaker |
| [`hosting-background.md`]() | BackgroundService, IHostedService |

### Real-time / advanced API

| Файл | Описание |
|------|----------|
| [`signalr.md`]() | SignalR, WebSockets, real-time |
| [`graphql.md`]() | HotChocolate, schemas, federation |

### Frontend integration

| Файл | Описание |
|------|----------|
| [`blazor-server.md`]() | Blazor Server architecture |
| [`blazor-wasm.md`]() | Blazor WebAssembly |

### Performance

| Файл | Описание |
|------|----------|
| [`native-aot.md`]() | Native AOT compilation |

---

## 🔗 Связанные папки

- [`Architecture/cqrs-mediatr`]() — CQRS поверх ASP.NET
- [`EFCore/`](../EFCore/) — data access в ASP.NET
- [`Infrastructure/observability`]() — production monitoring
- [`Performance/caching-strategies`]() — cache patterns
