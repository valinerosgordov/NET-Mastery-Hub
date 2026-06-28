# Infrastructure — DevOps & deploy

> 14 файлов / ~470 KB. Docker, Kubernetes, CI/CD, observability, messaging, NoSQL, IPC, AI/LLM integration.

[← Главный README](../readme.md) · [Полный INDEX](../INDEX.md)

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Никогда не делал Docker | [`Junior/docker-for-dev.md`](Junior/docker-for-dev.md) → потом [`Senior/docker.md`](Senior/docker.md) |
| Первый .NET-проект с нуля | [`Junior/project-setup-basics.md`](Junior/project-setup-basics.md) |
| Setup CI/CD | [`Middle/cicd-github-actions.md`](Middle/cicd-github-actions.md) |
| Deploy в Kubernetes | [`Middle/kubernetes.md`](Middle/kubernetes.md) |
| Production observability | [`Senior/observability.md`](Senior/observability.md) |
| Message queue / Kafka | [`Senior/messaging.md`](Senior/messaging.md) |
| Когда NoSQL вместо Postgres | [`Senior/nosql-databases.md`](Senior/nosql-databases.md) |
| LLM / RAG в .NET | [`Senior/llm-rag-patterns.md`](Senior/llm-rag-patterns.md), [`Senior/semantic-kernel.md`](Senior/semantic-kernel.md) |

---

## 📚 Все 14 файлов

### 🌱 Junior

| Файл | Описание |
|------|----------|
| [`docker-for-dev.md`](Junior/docker-for-dev.md) | Контейнеры с нуля: daily команды, Dockerfile основы, compose для local dev |
| [`project-setup-basics.md`](Junior/project-setup-basics.md) | dotnet CLI, solution + projects, .gitignore, базовый git workflow |

### 🌿 Middle

| Файл | Описание |
|------|----------|
| [`cicd-github-actions.md`](Middle/cicd-github-actions.md) | CI/CD для .NET: build, test, lint, security scan, deploy |
| [`kubernetes.md`](Middle/kubernetes.md) | Pod/Deployment/Service, deploy .NET app, health checks, Helm |

### 🏆 Senior

| Файл | Описание |
|------|----------|
| [`docker.md`](Senior/docker.md) | Docker deep: multistage, optimization, security ⭐ |
| [`observability.md`](Senior/observability.md) | OpenTelemetry, Prometheus, Jaeger ⭐ |
| [`messaging.md`](Senior/messaging.md) | RabbitMQ, Kafka, MassTransit |
| [`api-gateway.md`](Senior/api-gateway.md) | YARP/Ocelot: routing, auth, rate limiting в одной точке |
| [`nosql-databases.md`](Senior/nosql-databases.md) | MongoDB / Redis / Cosmos / DynamoDB / Cassandra — когда какой |
| [`ipc-named-pipes-grpc.md`](Senior/ipc-named-pipes-grpc.md) | Named pipes, gRPC, межпроцессное взаимодействие |
| [`project-setup.md`](Senior/project-setup.md) | Directory.Build.props, CPM, .editorconfig, packaging |
| [`llm-rag-patterns.md`](Senior/llm-rag-patterns.md) | LLM integration, RAG, vector DBs |
| [`semantic-kernel.md`](Senior/semantic-kernel.md) | Microsoft Semantic Kernel |
| [`wpf-production.md`](Senior/wpf-production.md) | WPF production deployment |

---

## 🔗 Связанные папки

- [`Architecture/Senior/distributed-systems.md`](../Architecture/Senior/distributed-systems.md) — для messaging
- [`AspNetCore/Senior/logging-observability.md`](../AspNetCore/Senior/logging-observability.md) — code-level observability
- [`Runtime/Senior/diagnostics-tools.md`](../Runtime/Senior/diagnostics-tools.md) — runtime diagnostics
- [`CSharp/Senior/desktop-frameworks.md`](../CSharp/Senior/desktop-frameworks.md) — для wpf-production
