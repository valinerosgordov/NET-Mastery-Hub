# Infrastructure — DevOps & deploy

> 14 файлов / ~470 KB. Docker, Kubernetes, CI/CD, observability, messaging, NoSQL, IPC, AI/LLM integration.

[[README|← Главный README]] · [[INDEX|Полный INDEX]]

---

## 🎯 Где начать

| Кто ты | С чего начать |
|--------|---------------|
| Никогда не делал Docker | [[docker-for-dev|`Junior/docker-for-dev.md`]] → потом [[docker|`Senior/docker.md`]] |
| Первый .NET-проект с нуля | [[project-setup-basics|`Junior/project-setup-basics.md`]] |
| Setup CI/CD | [[cicd-github-actions|`Middle/cicd-github-actions.md`]] |
| Deploy в Kubernetes | [[kubernetes|`Middle/kubernetes.md`]] |
| Production observability | [[observability|`Senior/observability.md`]] |
| Message queue / Kafka | [[messaging|`Senior/messaging.md`]] |
| Когда NoSQL вместо Postgres | [[nosql-databases|`Senior/nosql-databases.md`]] |
| LLM / RAG в .NET | [[llm-rag-patterns|`Senior/llm-rag-patterns.md`]], [[semantic-kernel|`Senior/semantic-kernel.md`]] |

---

## 📚 Все 14 файлов

### 🌱 Junior

| Файл | Описание |
|------|----------|
| [[docker-for-dev|`docker-for-dev.md`]] | Контейнеры с нуля: daily команды, Dockerfile основы, compose для local dev |
| [[project-setup-basics|`project-setup-basics.md`]] | dotnet CLI, solution + projects, .gitignore, базовый git workflow |

### 🌿 Middle

| Файл | Описание |
|------|----------|
| [[cicd-github-actions|`cicd-github-actions.md`]] | CI/CD для .NET: build, test, lint, security scan, deploy |
| [[kubernetes|`kubernetes.md`]] | Pod/Deployment/Service, deploy .NET app, health checks, Helm |

### 🏆 Senior

| Файл | Описание |
|------|----------|
| [[docker|`docker.md`]] | Docker deep: multistage, optimization, security ⭐ |
| [[observability|`observability.md`]] | OpenTelemetry, Prometheus, Jaeger ⭐ |
| [[messaging|`messaging.md`]] | RabbitMQ, Kafka, MassTransit |
| [[api-gateway|`api-gateway.md`]] | YARP/Ocelot: routing, auth, rate limiting в одной точке |
| [[nosql-databases|`nosql-databases.md`]] | MongoDB / Redis / Cosmos / DynamoDB / Cassandra — когда какой |
| [[ipc-named-pipes-grpc|`ipc-named-pipes-grpc.md`]] | Named pipes, gRPC, межпроцессное взаимодействие |
| [[project-setup|`project-setup.md`]] | Directory.Build.props, CPM, .editorconfig, packaging |
| [[llm-rag-patterns|`llm-rag-patterns.md`]] | LLM integration, RAG, vector DBs |
| [[semantic-kernel|`semantic-kernel.md`]] | Microsoft Semantic Kernel |
| [[wpf-production|`wpf-production.md`]] | WPF production deployment |

---

## 🔗 Связанные папки

- [[distributed-systems|`Architecture/Senior/distributed-systems.md`]] — для messaging
- [[logging-observability|`AspNetCore/Senior/logging-observability.md`]] — code-level observability
- [[diagnostics-tools|`Runtime/Senior/diagnostics-tools.md`]] — runtime diagnostics
- [[desktop-frameworks|`CSharp/Senior/desktop-frameworks.md`]] — для wpf-production
