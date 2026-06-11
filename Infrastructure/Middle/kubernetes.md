---
tags: [infrastructure, kubernetes, k8s, deployment, helm, dotnet, middle, senior]
level: Middle to Senior
date: 2026-04-30
---

# Kubernetes — k8s для .NET разработчика

> **Что Senior .NET должен знать про Kubernetes**: концепты, Pod/Deployment/Service, deploy .NET app, health checks, ConfigMap/Secret, Helm, debugging. Не deep DevOps — практическое для разработчика.

---

## Что это, зачем и когда

### Что такое Kubernetes

**Container orchestration** платформа. Управляет десятками-тысячами containers на кластере серверов:
- Где запускать (scheduling)
- Перезапуск при crash
- Auto-scaling
- Rolling updates
- Service discovery
- Load balancing

**Аналогия:** Docker — это "запустить container на одном host". Kubernetes — "запустить тысячи containers на сотне hosts с auto-recovery".

### Зачем .NET разработчику

В 2026 — k8s **стандарт** для production:
- 80% job vacancies упоминают k8s
- Senior — знать как deploy app, troubleshoot
- Не нужно быть DevOps экспертом, но **базовая навигация обязательна**

### Когда k8s НЕ нужен

- Маленький стартап, 1-2 service, <100 RPS — overkill
- App Service / Heroku — проще для малых команд
- Standalone desktop / mobile app — не applicable

См. [[microservices-vs-monolith|Microservices vs Monolith]].

---

## 1. Архитектура Kubernetes

### Cluster components

```
┌─────────────────────────────────────────┐
│           Control Plane (master)        │
│  ┌──────────────┐  ┌──────────────────┐│
│  │ API Server   │  │ Scheduler         ││
│  │ etcd         │  │ Controller Mgr    ││
│  └──────────────┘  └──────────────────┘│
└─────────────────────────────────────────┘
              │
   ┌──────────┼──────────┐
   ▼          ▼          ▼
┌─────┐   ┌─────┐    ┌─────┐
│Node1│   │Node2│    │Node3│
│ Pod │   │ Pod │    │ Pod │
│ Pod │   │ Pod │    │ Pod │
└─────┘   └─────┘    └─────┘
```

- **Control Plane** — мозги кластера
- **Nodes** — рабочие серверы где запускаются containers
- **Pods** — наименьшая deployable unit (1+ containers)

### Главные ресурсы (что разработчик использует)

| Resource | Назначение |
|----------|------------|
| **Pod** | Группа containers (обычно 1) |
| **Deployment** | Управляет N replicas Pod, rolling updates |
| **Service** | Stable IP/DNS для group of Pods |
| **ConfigMap** | Configuration (non-secret) |
| **Secret** | Secrets (passwords, tokens) |
| **Ingress** | HTTP routing внешних запросов |
| **Namespace** | Logical isolation в cluster |
| **PersistentVolume** | Storage |

---

## 2. Pod — базовая unit

### Что такое Pod

**Один или несколько containers** запущенные вместе:
- Shared network (один IP, localhost между containers)
- Shared storage volumes
- Lifecycle вместе

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-api
  labels:
    app: api
spec:
  containers:
  - name: api
    image: myregistry/myapi:1.0.0
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"        # 0.25 CPU
      limits:
        memory: "512Mi"
        cpu: "500m"
```

```bash
# Apply
kubectl apply -f pod.yaml

# View
kubectl get pods
kubectl describe pod my-api
kubectl logs my-api
```

### Когда multi-container Pod

- **Sidecar pattern** — log forwarder, service mesh proxy (Istio Envoy)
- **Init containers** — DB migrations перед app start
- **Tightly coupled processes** которые должны быть на одном host

В 95% случаев — один container per Pod.

---

## 3. Deployment — production way

Pod **сам по себе не используется** в production. Used **Deployment** который управляет Pods.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
  labels:
    app: api
spec:
  replicas: 3              # 3 Pods
  selector:
    matchLabels:
      app: api
  template:                # Pod template
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: myregistry/myapi:1.0.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          periodSeconds: 5
```

```bash
kubectl apply -f deployment.yaml

# Status
kubectl get deployment api-deployment
kubectl get pods -l app=api

# Scale
kubectl scale deployment api-deployment --replicas=5

# Rolling update
kubectl set image deployment/api-deployment api=myregistry/myapi:1.0.1

# Rollback
kubectl rollout undo deployment/api-deployment
kubectl rollout history deployment/api-deployment
```

### Rolling update strategy

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # max extra Pods во время update
      maxUnavailable: 0     # zero downtime
```

Default — постепенно заменяет старые Pod на новые. Без downtime.

### Replica sets

`Deployment` создаёт `ReplicaSet` который реально управляет Pods. Обычно работаешь с Deployment.

---

## 4. Service — discovery + load balancing

Pod IPs **меняются** (Pod re-created → новый IP). Service — **stable endpoint**.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api               # выбирает Pods с этим label
  ports:
  - port: 80               # service port
    targetPort: 8080       # Pod port
  type: ClusterIP          # internal cluster only
```

### Типы Service

| Type | Когда |
|------|-------|
| **ClusterIP** (default) | Internal only — service-to-service внутри cluster |
| **NodePort** | Exposes на каждом node на high port (30000-32767) — dev/test |
| **LoadBalancer** | Cloud provider создаёт external LB (AWS ELB, Azure LB) |
| **Headless** (clusterIP: None) | Direct Pod IPs — для StatefulSets |

### DNS

Внутри cluster — service доступен по name:

```
http://api-service.default.svc.cluster.local
http://api-service       # короткая форма (same namespace)
```

В .NET app:
```csharp
// Просто использовать имя service как hostname
var url = "http://api-service/users";
var response = await httpClient.GetAsync(url);
```

См. [[api-design|API Design]].

---

## 5. Deploy .NET app в k8s — полный пример

### Шаг 1: Dockerfile

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -c Release -o /app /p:UseAppHost=false

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /app .
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV DOTNET_RUNNING_IN_CONTAINER=true
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

См. [[docker|Docker]] для полного гайда.

### Шаг 2: ASP.NET Core с health checks

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())
    .AddDbContextCheck<AppDbContext>(name: "database");

var app = builder.Build();

// Liveness — приложение живо?
app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // только self check
});

// Readiness — готов принимать трафик?
app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapControllers();
app.Run();
```

### Шаг 3: Build + push image

```bash
docker build -t myregistry.azurecr.io/myapi:1.0.0 .
docker push myregistry.azurecr.io/myapi:1.0.0
```

### Шаг 4: Kubernetes manifests

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapi
  template:
    metadata:
      labels:
        app: myapi
    spec:
      containers:
      - name: myapi
        image: myregistry.azurecr.io/myapi:1.0.0
        ports:
        - containerPort: 8080
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: ConnectionStrings__Default
          valueFrom:
            secretKeyRef:
              name: db-secrets
              key: connection-string
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "1000m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: myapi
  namespace: production
spec:
  selector:
    app: myapi
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapi
  namespace: production
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - api.mycompany.com
    secretName: myapi-tls
  rules:
  - host: api.mycompany.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapi
            port:
              number: 80
```

### Шаг 5: Apply

```bash
kubectl apply -f k8s/
kubectl get pods -n production
kubectl logs -f deployment/myapi -n production
```

---

## 6. ConfigMap и Secret

### ConfigMap — конфигурация

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapi-config
data:
  Logging__LogLevel__Default: "Information"
  Features__NewUI: "true"
  appsettings.Production.json: |
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information"
        }
      }
    }
```

```yaml
# В Deployment
env:
- name: Logging__LogLevel__Default
  valueFrom:
    configMapKeyRef:
      name: myapi-config
      key: Logging__LogLevel__Default

# Или mount как файл
volumeMounts:
- name: config
  mountPath: /app/appsettings.Production.json
  subPath: appsettings.Production.json
volumes:
- name: config
  configMap:
    name: myapi-config
```

### Secret — для паролей

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secrets
type: Opaque
stringData:
  connection-string: "Server=db.example.com;Database=mydb;User Id=app;Password=secret"
  jwt-key: "supersecretkey"
```

> [!warning] Secret в YAML — не secret!
> Base64 encoded, не encrypted. Для production:
> - Используй **Sealed Secrets** / **Azure Key Vault** / **AWS Secrets Manager** / **HashiCorp Vault**
> - Не commit Secret YAML в git

```bash
# Создать from CLI
kubectl create secret generic db-secrets \
  --from-literal=connection-string='Server=...' \
  --from-literal=jwt-key='...'

# View (base64)
kubectl get secret db-secrets -o yaml

# Decode
kubectl get secret db-secrets -o jsonpath='{.data.jwt-key}' | base64 -d
```

### .NET Configuration: env vars автоматом mapped

```yaml
env:
- name: ConnectionStrings__Default
  value: "..."
- name: Logging__LogLevel__Default
  value: "Information"
```

```csharp
// .NET Configuration parses через __
builder.Configuration["ConnectionStrings:Default"]
builder.Configuration["Logging:LogLevel:Default"]
```

См. [[di-configuration|DI & Configuration]].

---

## 7. Health checks — критично!

K8s использует health checks для:
- **Liveness** — restart Pod если unhealthy
- **Readiness** — не направлять трафик пока не ready
- **Startup** — give time for slow start

### Liveness probe

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30  # ждём перед первым check
  periodSeconds: 10        # каждые 10 sec
  failureThreshold: 3      # 3 fails → restart
  timeoutSeconds: 1
```

Что проверять:
- App responds to HTTP
- Не deadlocked
- НЕ checks dependencies! (DB down → не убивай app)

### Readiness probe

```yaml
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```

Что проверять:
- DB reachable
- Caches warm
- Dependencies available

```csharp
// ASP.NET Core
builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy())                  // liveness
    .AddDbContextCheck<AppDbContext>(tags: new[] { "ready" })              // readiness
    .AddRedis(redisConnString, tags: new[] { "ready" });                  // readiness

app.MapHealthChecks("/health/live", new() { Predicate = _ => false });
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") });
```

### Startup probe (.NET startup может быть медленный)

```yaml
startupProbe:
  httpGet:
    path: /health/live
    port: 8080
  failureThreshold: 30  # 30 * 10s = 5 min для startup
  periodSeconds: 10
```

---

## 8. Resource limits и requests

```yaml
resources:
  requests:           # минимум что нужно (для scheduler)
    memory: "256Mi"
    cpu: "250m"       # 0.25 CPU
  limits:             # максимум (kill если exceeded)
    memory: "512Mi"
    cpu: "1000m"      # 1.0 CPU
```

### Memory limit — hard

Pod превышает memory limit → **OOMKilled** (Out Of Memory). Restart.

```bash
kubectl describe pod myapp
# Last State: Terminated
# Reason: OOMKilled
```

**Лечение:**
1. Find leak — profile с `dotnet-counters`, `dotnet-dump`
2. Increase limit
3. Optimize allocations

См. [[gc-memory|GC и память]] и [[diagnostics-tools|Diagnostics Tools]].

### CPU limit — soft (throttling)

Превышение → throttling, не kill. Но performance падает.

### .NET-specific: GC modes

```yaml
env:
- name: DOTNET_gcServer
  value: "1"           # Server GC — для multi-core
- name: DOTNET_GCHeapCount
  value: "4"            # сколько heaps
- name: DOTNET_GCHighMemPercent
  value: "75"           # aggressive GC при high memory
```

---

## 9. kubectl — daily commands

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes

# Namespaces
kubectl get namespaces
kubectl config set-context --current --namespace=production

# Pods
kubectl get pods                          # текущий namespace
kubectl get pods -A                       # all namespaces
kubectl get pods -l app=myapi             # по label
kubectl describe pod myapp-xyz             # детали
kubectl logs myapp-xyz                     # logs
kubectl logs -f myapp-xyz                  # follow
kubectl logs myapp-xyz --previous          # previous container (если crashed)
kubectl logs -l app=myapi --tail=100       # last 100 lines from all matching pods

# Exec в running container
kubectl exec -it myapp-xyz -- /bin/bash
kubectl exec myapp-xyz -- env

# Port forward (доступ к Pod локально)
kubectl port-forward myapp-xyz 8080:8080

# Apply / delete
kubectl apply -f deployment.yaml
kubectl delete -f deployment.yaml
kubectl delete pod myapp-xyz                # будет re-created (Deployment)

# Deployment
kubectl rollout status deployment/myapi
kubectl rollout history deployment/myapi
kubectl rollout undo deployment/myapi
kubectl scale deployment/myapi --replicas=5

# Resources
kubectl top pod                            # CPU/memory usage
kubectl top node

# Events
kubectl get events --sort-by='.lastTimestamp'

# Edit
kubectl edit deployment/myapi              # редактирует prod (осторожно!)

# Diff
kubectl diff -f deployment.yaml            # разница current vs file
```

---

## 10. Helm — package manager

K8s manifests становятся длинными. **Helm** — templating + package management.

### Структура Helm chart

```
myapi/
├── Chart.yaml         # метаданные
├── values.yaml         # default values
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── _helpers.tpl
└── values-prod.yaml    # override для prod
```

### values.yaml

```yaml
replicaCount: 3
image:
  repository: myregistry.azurecr.io/myapi
  tag: 1.0.0
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
ingress:
  enabled: true
  host: api.mycompany.com
resources:
  limits:
    memory: 512Mi
    cpu: 1000m
  requests:
    memory: 256Mi
    cpu: 250m
```

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: app
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        resources:
          {{- toYaml .Values.resources | nindent 12 }}
```

### Команды

```bash
helm install myapi ./chart                              # установить
helm install myapi ./chart -f values-prod.yaml          # с override
helm upgrade myapi ./chart                              # upgrade
helm rollback myapi 2                                    # rollback к ревизии 2
helm uninstall myapi                                     # удалить
helm list                                                # все releases
helm template myapi ./chart                              # debug — посмотреть готовый YAML
```

---

## 11. Debugging .NET app в k8s

### Logs

```bash
# Tail logs
kubectl logs -f deployment/myapi --tail=100

# Все Pod в deployment
kubectl logs -f -l app=myapi --max-log-requests=10
```

### Exec в container

```bash
kubectl exec -it deployment/myapi -- /bin/bash

# Внутри container
ps aux                          # processes
ls -la /app                     # files
cat /app/appsettings.json
env | grep ConnectionStrings
```

### Diagnostics tools

В image добавить .NET diagnostic tools:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0
RUN dotnet tool install -g dotnet-counters dotnet-trace dotnet-dump
ENV PATH="${PATH}:/root/.dotnet/tools"
```

Использование:
```bash
kubectl exec -it pod-xyz -- dotnet-counters monitor --process-id 1
kubectl exec -it pod-xyz -- dotnet-trace collect --process-id 1
kubectl exec -it pod-xyz -- dotnet-dump collect --process-id 1
kubectl cp pod-xyz:/tmp/dump.dmp ./local-dump.dmp
```

См. [[diagnostics-tools|Diagnostics Tools]].

### Common issues

| Симптом | Проверка |
|---------|----------|
| Pod CrashLoopBackOff | `kubectl logs --previous` — что было до crash |
| Pod Pending | `kubectl describe pod` — нет ресурсов? |
| Pod ImagePullBackOff | `kubectl describe pod` — image registry auth? |
| OOMKilled | Memory leak / increase limit |
| 503 errors | Readiness probe failing? `kubectl describe pod` |
| Slow rollout | Health checks too strict? probe timeouts? |

---

## 12. Production checklist

```
Image:
□ Multi-stage Dockerfile (small final)
□ Pinned base image version (не latest)
□ Non-root user
□ Health endpoints

Deployment:
□ replicas: 2+ (HA)
□ resource requests AND limits
□ liveness + readiness probes
□ rolling update strategy
□ HorizontalPodAutoscaler если variable load

Configuration:
□ Secrets — не в YAML committed
□ External secrets manager (Azure KV, Vault)
□ ConfigMap для non-secret config

Observability:
□ Structured logging → Loki / ELK
□ OpenTelemetry → Prometheus / Jaeger
□ Health checks правильные (liveness vs readiness)
□ Application metrics

Security:
□ NetworkPolicy — restricts traffic
□ PodSecurityPolicy / PodSecurity Admission
□ Service mesh (Istio) для mTLS если needed
□ Image scanning (Trivy / Snyk)
```

См. [[observability|Observability]].

---

## 13. Cheat sheet

| Что нужно | Команда |
|-----------|---------|
| Применить YAML | `kubectl apply -f file.yaml` |
| Все Pod | `kubectl get pods` |
| Logs | `kubectl logs -f deployment/myapi` |
| Exec | `kubectl exec -it pod-xyz -- bash` |
| Port forward | `kubectl port-forward pod-xyz 8080:8080` |
| Scale | `kubectl scale deployment/myapi --replicas=5` |
| Rollback | `kubectl rollout undo deployment/myapi` |
| Edit live | `kubectl edit deployment/myapi` (осторожно!) |
| Top usage | `kubectl top pod` |
| Describe | `kubectl describe pod xyz` |
| Events | `kubectl get events --sort-by='.lastTimestamp'` |
| Switch namespace | `kubectl config set-context --current --namespace=prod` |
| Delete pod (will recreate) | `kubectl delete pod xyz` |

---

## 14. Чему учиться дальше

После basics:

- **Helm** — package management
- **Operators** — для stateful apps
- **Service Mesh** (Istio / Linkerd) — для микросервисов
- **GitOps** (ArgoCD / Flux) — declarative deploy
- **Monitoring** (Prometheus / Grafana / Loki)
- **Security** (NetworkPolicies, RBAC, OPA)

---

## См. также

- [[docker|Docker]] — fundament k8s
- [[observability|Observability]] — monitoring k8s apps
- [[native-aot|Native AOT]] — small images для k8s
- [[microservices-vs-monolith|Microservices vs Monolith]]
- [[distributed-systems|Distributed Systems]]
- [[diagnostics-tools|Diagnostics Tools]]

## Reading list

- **Kubernetes Documentation** — kubernetes.io/docs
- **Kubernetes The Hard Way** — github.com/kelseyhightower/kubernetes-the-hard-way
- **Kubernetes Up & Running** — Kelsey Hightower (книга)
- **Helm Documentation** — helm.sh/docs
- **Microsoft Docs — .NET on Kubernetes** — learn.microsoft.com/dotnet/architecture/cloud-native
- **kubectl cheat sheet** — kubernetes.io/docs/reference/kubectl/cheatsheet
- **Killercoda** — killercoda.com (interactive labs)
- **CKAD certification** — kubernetes.io/training (для углубления)
