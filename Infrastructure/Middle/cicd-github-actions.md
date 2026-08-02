---
tags: [infrastructure, cicd, github-actions, ci-cd, pipeline, dotnet, middle]
level: Middle
date: 2026-08-02
---

# CI/CD с GitHub Actions

> **Continuous Integration / Continuous Delivery для .NET**: build, test, lint, security scan, deploy. GitHub Actions — самая популярная CI платформа в open-source 2026. Closes пробел "знаю что есть, не понимаю как написать pipeline".

---

## Что это, зачем и когда

### CI/CD

- **CI (Continuous Integration)** — автоматический build + tests на каждый PR / push
- **CD (Continuous Delivery)** — автоматический deploy в staging
- **CD (Continuous Deployment)** — автоматический deploy в production

### Зачем

| Без CI/CD | С CI/CD |
|-----------|---------|
| "Works on my machine" | Build on clean env |
| Manual testing | Tests automated |
| Bug в production | Caught before merge |
| Deploy = страшно | Deploy = ежедневно |
| 1 release / month | 1+ release / day |

### Где работает

- **GitHub Actions** — встроено в GitHub, бесплатно для public + 2000 min/month private
- **Azure DevOps Pipelines** — Microsoft, для Azure workloads
- **GitLab CI** — встроено в GitLab
- **Jenkins** — self-hosted, legacy но мощный
- **CircleCI / TeamCity / Travis** — другие SaaS

В 2026 — **GitHub Actions доминирует** для open-source и среднего бизнеса.

---

## 1. Структура GitHub Actions

### Workflow file

```
.github/
└── workflows/
    ├── ci.yml             # на push / PR
    ├── cd-staging.yml      # на merge to main
    └── cd-production.yml   # на git tag
```

### Базовая структура

```yaml
name: CI                          # имя workflow

on:                                # триггеры
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:                              # параллельные jobs
  build:
    runs-on: ubuntu-latest         # или windows-latest, macos-latest
    steps:                         # последовательные steps
    - uses: actions/checkout@v6    # action из marketplace
    
    - name: Setup .NET             # custom step
      uses: actions/setup-dotnet@v6
      with:
        dotnet-version: '10.0.x'
    
    - run: dotnet restore          # shell command
    - run: dotnet build --no-restore
    - run: dotnet test --no-build
```

### Концепты

| Term | Что |
|------|-----|
| **Workflow** | YAML file в `.github/workflows/` |
| **Event** | Триггер (push, pull_request, schedule, workflow_dispatch) |
| **Job** | Группа steps на одном runner |
| **Step** | Individual command или action |
| **Action** | Переиспользуемый block (`uses: actions/checkout@v6`) |
| **Runner** | VM где job выполняется (Ubuntu / Windows / macOS / self-hosted) |

---

## 2. Полный CI для .NET app

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  DOTNET_VERSION: '10.0.x'
  DOTNET_NOLOGO: true
  DOTNET_CLI_TELEMETRY_OPTOUT: true

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout
      uses: actions/checkout@v6
      with:
        fetch-depth: 0  # для git history (если нужно)
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v6
      with:
        dotnet-version: ${{ env.DOTNET_VERSION }}
    
    - name: Cache NuGet packages
      uses: actions/cache@v5
      with:
        path: ~/.nuget/packages
        key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
        restore-keys: |
          ${{ runner.os }}-nuget-
    
    - name: Restore
      run: dotnet restore
    
    - name: Build
      run: dotnet build --configuration Release --no-restore
    
    - name: Test
      run: dotnet test --configuration Release --no-build --verbosity normal --collect:"XPlat Code Coverage" --results-directory ./TestResults
    
    - name: Upload coverage
      uses: codecov/codecov-action@v5
      with:
        directory: ./TestResults
        token: ${{ secrets.CODECOV_TOKEN }}
    
    - name: Upload test results
      uses: actions/upload-artifact@v5
      if: always()
      with:
        name: test-results
        path: ./TestResults
```

---

## 3. Multi-job pipeline

```yaml
name: Full CI

on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v6
    - uses: actions/setup-dotnet@v6
      with:
        dotnet-version: '10.0.x'
    - run: dotnet format --verify-no-changes

  build:
    runs-on: ubuntu-latest
    needs: lint                    # сначала lint, потом build
    steps:
    - uses: actions/checkout@v6
    - uses: actions/setup-dotnet@v6
      with:
        dotnet-version: '10.0.x'
    - run: dotnet build --configuration Release

  test:
    runs-on: ubuntu-latest
    needs: build
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
    runs-on: ${{ matrix.os }}
    steps:
    - uses: actions/checkout@v6
    - uses: actions/setup-dotnet@v6
      with:
        dotnet-version: '10.0.x'
    - run: dotnet test

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v6
    - name: Run CodeQL
      uses: github/codeql-action/init@v4
      with:
        languages: csharp
    - run: dotnet build
    - uses: github/codeql-action/analyze@v4

  docker-build:
    runs-on: ubuntu-latest
    needs: [test, security-scan]
    steps:
    - uses: actions/checkout@v6
    - uses: docker/setup-buildx-action@v3
    - uses: docker/login-action@v3
      with:
        registry: ghcr.io
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    - uses: docker/build-push-action@v6
      with:
        push: true
        tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

---

## 4. Triggers — when run

```yaml
on:
  # Push to specific branches
  push:
    branches: [ main, develop ]
    paths-ignore: [ '**.md', 'docs/**' ]
  
  # PR
  pull_request:
    branches: [ main ]
    types: [ opened, synchronize, reopened ]
  
  # Schedule (cron)
  schedule:
    - cron: '0 0 * * *'           # ежедневно в полночь UTC
  
  # Manual
  workflow_dispatch:
    inputs:
      environment:
        description: 'Where to deploy'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]
  
  # Tag push (для releases)
  push:
    tags: [ 'v*.*.*' ]
  
  # Other workflow completed
  workflow_run:
    workflows: ["CI"]
    types: [completed]
```

---

## 5. Secrets

### Создание

GitHub repo → Settings → Secrets and variables → Actions → New repository secret

Создай secrets:
- `AZURE_CREDENTIALS` (JSON для Azure CLI)
- `DOCKER_PASSWORD`
- `NUGET_API_KEY`
- `CODECOV_TOKEN`

### Использование

```yaml
steps:
- name: Login to Azure
  uses: azure/login@v2
  with:
    creds: ${{ secrets.AZURE_CREDENTIALS }}

- name: Push to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}
```

> [!warning] Secrets правила
> - **Никогда не logging secrets** — `echo $SECRET` не покажет, GitHub маскирует, но не безопасно
> - **Don't pass secrets как inputs** на 3rd party actions — они могут leak
> - **Rotate** регулярно
> - **Environment secrets** — для production (требует approval)

См. [[security-practices|Security Practices]].

---

## 6. Environments — staging, production

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment: staging          # ⚠️ Environment в GitHub
    steps:
    - run: deploy-script.sh
      env:
        DATABASE_URL: ${{ secrets.STAGING_DB_URL }}

  deploy-production:
    runs-on: ubuntu-latest
    environment: production       # требует approval
    needs: deploy-staging
    steps:
    - run: deploy-script.sh
      env:
        DATABASE_URL: ${{ secrets.PROD_DB_URL }}
```

В GitHub Settings → Environments → New environment:
- **production** — требует review от 1+ approver
- **Wait timer** — задержка перед deploy
- **Branch restrictions** — only `main`

---

## 7. Build артефакты + reuse

### Upload artifact

```yaml
- name: Build
  run: dotnet publish -c Release -o ./publish

- name: Upload publish
  uses: actions/upload-artifact@v5
  with:
    name: publish
    path: ./publish
    retention-days: 7
```

### Download в другом job

```yaml
deploy:
  needs: build
  runs-on: ubuntu-latest
  steps:
  - uses: actions/download-artifact@v6
    with:
      name: publish
      path: ./publish
  
  - run: # deploy ./publish
```

---

## 8. Caching

NuGet cache **обязательно** — без него каждый build загружает packages с нуля.

```yaml
- name: Cache NuGet
  uses: actions/cache@v5
  with:
    path: ~/.nuget/packages
    key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
    restore-keys: |
      ${{ runner.os }}-nuget-
```

**Save time:**
- Без cache: 60-180 sec restore
- С cache: 5-15 sec

### Build cache

```yaml
- name: Cache build
  uses: actions/cache@v5
  with:
    path: |
      ~/.nuget/packages
      **/bin
      **/obj
    key: ${{ runner.os }}-build-${{ hashFiles('**/*.csproj', '**/*.cs') }}
```

⚠️ Build cache — risky, может cause stale builds. NuGet cache — безопасно.

---

## 9. Code coverage

### Сбор

```yaml
- name: Test with coverage
  run: dotnet test --collect:"XPlat Code Coverage" --results-directory ./TestResults
```

Создаст `coverage.cobertura.xml`.

### Upload в Codecov

```yaml
- name: Upload to Codecov
  uses: codecov/codecov-action@v5
  with:
    directory: ./TestResults
    token: ${{ secrets.CODECOV_TOKEN }}  # для public repo не нужен
```

### Quality gate

```yaml
- name: Verify coverage
  run: |
    pip install lxml
    python -c "
    import xml.etree.ElementTree as ET
    tree = ET.parse('./TestResults/coverage.cobertura.xml')
    line_rate = float(tree.getroot().attrib['line-rate'])
    print(f'Coverage: {line_rate:.1%}')
    if line_rate < 0.80:
        print('Coverage below 80%')
        exit(1)
    "
```

См. [[static-analysis|Static Analysis]].

---

## 10. Docker build & push

```yaml
docker-build-push:
  runs-on: ubuntu-latest
  permissions:
    packages: write              # для GHCR
  steps:
  - uses: actions/checkout@v6
  
  - name: Set up Docker Buildx
    uses: docker/setup-buildx-action@v3
  
  - name: Log in to GHCR
    uses: docker/login-action@v3
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}
  
  - name: Extract metadata
    id: meta
    uses: docker/metadata-action@v5
    with:
      images: ghcr.io/${{ github.repository }}
      tags: |
        type=ref,event=branch
        type=sha,prefix=,suffix=,format=short
        type=semver,pattern={{version}}
  
  - name: Build and push
    uses: docker/build-push-action@v6
    with:
      context: .
      file: ./Dockerfile
      push: true
      tags: ${{ steps.meta.outputs.tags }}
      labels: ${{ steps.meta.outputs.labels }}
      cache-from: type=gha
      cache-to: type=gha,mode=max
```

См. [[docker|Docker]].

---

## 11. Deploy в Azure / AWS / k8s

### Azure App Service

```yaml
deploy-azure:
  runs-on: ubuntu-latest
  environment: production
  steps:
  - uses: actions/checkout@v6
  
  - uses: actions/setup-dotnet@v6
    with:
      dotnet-version: '10.0.x'
  
  - run: dotnet publish -c Release -o ./publish
  
  - uses: azure/login@v2
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}
  
  - uses: azure/webapps-deploy@v3
    with:
      app-name: 'my-api-prod'
      package: ./publish
```

### Kubernetes

```yaml
deploy-k8s:
  runs-on: ubuntu-latest
  environment: production
  steps:
  - uses: actions/checkout@v6
  
  - uses: azure/setup-kubectl@v3
    with:
      version: 'latest'
  
  - uses: azure/aks-set-context@v3
    with:
      creds: ${{ secrets.AZURE_CREDENTIALS }}
      cluster-name: my-cluster
      resource-group: my-rg
  
  - run: |
      kubectl set image deployment/myapi \
        myapi=ghcr.io/${{ github.repository }}:${{ github.sha }} \
        -n production
      kubectl rollout status deployment/myapi -n production
```

См. [[kubernetes|Kubernetes]].

### Helm

```yaml
- uses: azure/setup-helm@v4

- run: |
    helm upgrade --install myapi ./charts/myapi \
      --namespace production \
      --set image.tag=${{ github.sha }} \
      --wait --timeout 5m
```

---

## 12. NuGet publish

```yaml
nuget-publish:
  runs-on: ubuntu-latest
  if: startsWith(github.ref, 'refs/tags/v')   # только при tag
  steps:
  - uses: actions/checkout@v6
  
  - uses: actions/setup-dotnet@v6
    with:
      dotnet-version: '10.0.x'
  
  - run: dotnet pack --configuration Release -o ./packages
  
  - run: |
      dotnet nuget push ./packages/*.nupkg \
        --api-key ${{ secrets.NUGET_API_KEY }} \
        --source https://api.nuget.org/v3/index.json
```

---

## 13. Matrix builds

Для multi-OS / multi-version testing:

```yaml
test:
  strategy:
    fail-fast: false              # не останавливай other matrix entries
    matrix:
      os: [ubuntu-latest, windows-latest, macos-latest]
      dotnet: ['8.0.x', '10.0.x']
  runs-on: ${{ matrix.os }}
  steps:
  - uses: actions/checkout@v6
  - uses: actions/setup-dotnet@v6
    with:
      dotnet-version: ${{ matrix.dotnet }}
  - run: dotnet test
```

Создаст 6 jobs (3 OS × 2 .NET versions).

---

## 14. Reusable workflows

`.github/workflows/build.yml`:
```yaml
name: Reusable build
on:
  workflow_call:
    inputs:
      dotnet-version:
        required: true
        type: string
    secrets:
      NUGET_API_KEY:
        required: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v6
    - uses: actions/setup-dotnet@v6
      with:
        dotnet-version: ${{ inputs.dotnet-version }}
    - run: dotnet build
```

Использование в другом workflow:
```yaml
jobs:
  call-build:
    uses: ./.github/workflows/build.yml
    with:
      dotnet-version: '10.0.x'
    secrets:
      NUGET_API_KEY: ${{ secrets.NUGET_API_KEY }}
```

---

## 15. Common Pitfalls

### 1. No caching → slow builds

3-минутный build vs 30-секундный — добавь NuGet cache.

### 2. Secrets в logs

```yaml
# ❌
- run: echo "Password is ${{ secrets.PASSWORD }}"  # masked но don't try

# ✅ Используй env
- run: deploy.sh
  env:
    DB_PASSWORD: ${{ secrets.PASSWORD }}
```

### 3. Устаревшие мажорки actions (`checkout@v4`, `cache@v4`)

Официальные actions регулярно поднимают мажорку — в основном из-за смены Node-runtime на runner'е (август 2025: волна node24 → checkout v5, setup-dotnet v5; конец 2025 — начало 2026: checkout v6, setup-dotnet v6, cache v5, upload-artifact v5). Старые мажорки продолжают работать до отключения их Node-версии на runner'ах, потом workflow ломается разом. Механизм защиты: Dependabot с `package-ecosystem: github-actions` — обновляет пины автоматически. Актуальные на 2026: `checkout@v6`, `setup-dotnet@v6`, `cache@v5`, `upload-artifact@v5`, `download-artifact@v6`, `codeql-action@v4`.

### 4. Hardcoded versions

```yaml
# ❌
- uses: actions/setup-dotnet@v6
  with:
    dotnet-version: '10.0.0'        # specific patch — может не быть на runner

# ✅ wildcard
- uses: actions/setup-dotnet@v6
  with:
    dotnet-version: '10.0.x'
```

### 5. Missing `--no-restore` / `--no-build`

```yaml
# ❌ Inefficient — restore трижды
- run: dotnet restore
- run: dotnet build         # restores again
- run: dotnet test          # builds again

# ✅
- run: dotnet restore
- run: dotnet build --no-restore
- run: dotnet test --no-build
```

### 6. Permissions issues

```yaml
permissions:
  contents: read
  packages: write              # для GHCR push
  pull-requests: write          # для PR comments

jobs:
  deploy:
    permissions:
      id-token: write             # для Azure OIDC
      contents: read
```

### 7. Slow tests in CI

- Parallel test run: `dotnet test --parallel`
- Split в matrix по test class
- Cache test data
- Test categories — `--filter Category!=Slow` для CI

См. [[integration-testing|Integration Testing]].

### 8. Not pinning action version

```yaml
# ❌ Версия меняется без warning
- uses: some-org/some-action@main

# ✅ Pin к specific version OR SHA
- uses: some-org/some-action@v1.2.3
- uses: some-org/some-action@abc123def456  # SHA — даже более safe
```

### 9. workflow_dispatch без proper inputs

Если manual trigger — добавь нормальные defaults и validation.

### 10. No status badges

```markdown
# README.md
![CI](https://github.com/user/repo/actions/workflows/ci.yml/badge.svg)
```

Видимо что pipeline проходит.

---

## 16. Best Practices

- **Маленькие jobs** — easier to debug, parallel
- **Cache NuGet packages** — must
- **Pin action versions** — актуальная мажорка (`@v6`) минимум, `@sha` лучше (см. Supply-chain hardening ниже)
- **`needs:`** для dependencies — не запускай deploy если test failed
- **Environments** для staging/production
- **Approvals** для production
- **Secrets** — никогда не commit в YAML
- **Status checks** — required for merge в branch protection
- **`fail-fast: false`** в matrix если хочешь видеть все failures
- **CodeQL** для security scan
- **Dependabot** для dep updates
- **Concurrency** — cancel duplicate runs:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

### Supply-chain hardening CI

CI — это машина с твоими секретами, которая исполняет чужой код (actions). Три уровня защиты:

**1. Pin actions по SHA, не по тегу.** Тег (`@v45`) — mutable: владелец (или взломщик) может переставить его на другой коммит, и твой workflow молча начнёт исполнять новый код. Урок — атака на `tj-actions/changed-files` (март 2025, CVE-2025-30066): злоумышленник переписал существующие version-теги на вредоносный коммит, который дампил память runner'а и печатал секреты (`GITHUB_TOKEN`, cloud-ключи) в публичные build-логи. Пострадали тысячи репозиториев с тег-пинами; SHA-пины атаку не подхватили.

```yaml
# ❌ mutable — тег могут переставить
- uses: some-org/some-action@v45

# ✅ immutable — полный 40-символьный commit SHA (+ комментарий с версией для читаемости)
- uses: some-org/some-action@0123456789abcdef0123456789abcdef01234567 # v45.0.3
```

Dependabot умеет обновлять SHA-пины (`package-ecosystem: github-actions`) — безопасность без ручного труда. Official `actions/*` — минимум мажорный тег, для third-party — только SHA.

**2. Минимальный `GITHUB_TOKEN`.** По умолчанию токен может иметь широкие права на репо. Объявляй на уровне workflow read-only и расширяй точечно per-job:

```yaml
permissions:
  contents: read          # default для всего workflow

jobs:
  docker-push:
    permissions:
      contents: read
      packages: write     # только этому job нужен GHCR push
```

Скомпрометированная action получает ровно те права, что ты выдал, а не «всё».

**3. Artifact attestations (SLSA provenance).** GitHub подписывает метаданные сборки — какой workflow, из какого коммита, на каком runner'е собрал артефакт. Потребитель проверяет `gh attestation verify` — защита от подмены артефакта после сборки:

```yaml
jobs:
  build:
    permissions:
      id-token: write
      attestations: write
    steps:
      - run: dotnet publish -c Release -o ./publish
      - uses: actions/attest-build-provenance@v4   # с v4 — обёртка над actions/attest
        with:
          subject-path: ./publish/MyApp.dll
```

---

## 17. Cheat sheet

| Что | Solution |
|-----|----------|
| Запустить на push | `on: push` |
| Запустить на PR | `on: pull_request` |
| Manual trigger | `workflow_dispatch` |
| Schedule | `on: schedule: cron: '0 0 * * *'` |
| Setup .NET | `actions/setup-dotnet@v6` |
| Cache NuGet | `actions/cache@v5` + `~/.nuget/packages` |
| Secret | `${{ secrets.NAME }}` |
| Если файлы | `paths: ['**.cs']` |
| Многоплатформенно | `strategy.matrix.os` |
| Job dependency | `needs: [build]` |
| Conditional step | `if: github.event_name == 'push'` |
| Output между steps | `id: step1` + `${{ steps.step1.outputs.x }}` |
| Artifact | `actions/upload-artifact@v5` |
| Approval gate | `environment: production` |
| Cancel duplicate | `concurrency` |

---

## 18. Decision tree

```
Что нужно автоматизировать?
│
├── Build + test on PR
│   → CI workflow (push + pull_request triggers)
│
├── Deploy to staging
│   → CD workflow (push to main → staging)
│
├── Deploy to production
│   → Tag-based (push tag v*) + environment approval
│
├── Periodic task (DB cleanup, monitoring)
│   → schedule trigger (cron)
│
├── Manual trigger
│   → workflow_dispatch
│
├── NuGet publish
│   → tag-based + dotnet pack/push
│
└── Cross-repo workflow
    → workflow_call (reusable workflows)
```

---

## Case Studies

### Case Study #1 — Multi-stage build с testing

**Сценарий:** ASP.NET Core app, нужен CI: build → test → publish to staging → manual approval → production.

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-dotnet@v6
        with: { dotnet-version: '10.0.x' }
      
      - name: Restore
        run: dotnet restore
      
      - name: Build
        run: dotnet build -c Release --no-restore
      
      - name: Test
        run: dotnet test -c Release --no-build --collect:"XPlat Code Coverage"
      
      - name: Publish artifacts
        run: dotnet publish src/MyApp.Api -c Release -o ./publish --no-build
      
      - uses: actions/upload-artifact@v5
        with:
          name: app-package
          path: ./publish

  deploy-staging:
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/download-artifact@v6
        with: { name: app-package }
      - name: Deploy to Azure
        uses: azure/webapps-deploy@v3
        with:
          app-name: myapp-staging
          publish-profile: SECRET_REF_AZURE_STAGING

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production  # требует manual approval в GitHub
    steps:
      - uses: actions/download-artifact@v6
        with: { name: app-package }
      - name: Deploy to Azure
        uses: azure/webapps-deploy@v3
        with:
          app-name: myapp-prod
          publish-profile: SECRET_REF_AZURE_PROD
```

---

### Case Study #2 — Docker image build + push в registry

```yaml
- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: GITHUB_ACTOR_REF
    password: GITHUB_TOKEN_REF

- name: Build and push
  uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: |
      ghcr.io/REPO_REF:SHA_REF
      ghcr.io/REPO_REF:latest
```

---

### Case Study #3 — Secrets management

```yaml
# Read connection string из GitHub secrets
- name: Run integration tests
  env:
    ConnectionStrings__Default: SECRET_DB_CONN
    ApiKeys__Stripe: SECRET_STRIPE_KEY
  run: dotnet test
```

> [!warning] Никогда не коммить secrets в repo
> Используй GitHub Secrets, environment-specific secrets, или OIDC для cloud auth.

См. [[auth-security|Auth & Security]].


---

## См. также

- [[docker|Docker]] — build images в CI
- [[kubernetes|Kubernetes]] — deploy through CI
- [[static-analysis|Static Analysis]] — analyzers в CI
- [[code-review|Code Review]] — CI checks для PR
- [[integration-testing|Integration Testing]] — tests в CI
- [[observability|Observability]] — мониторинг после deploy
- [[project-setup|Project Setup]] — Directory.Build.props

## Reading list

- **GitHub Actions documentation** — docs.github.com/en/actions
- **Awesome Actions** — github.com/sdras/awesome-actions
- **GitHub Actions for .NET** — learn.microsoft.com/dotnet/devops/dotnet-cicd-with-github-actions
- **Actions marketplace** — github.com/marketplace?type=actions
- **Azure DevOps comparison** — learn.microsoft.com/azure/devops/pipelines
- **Andrew Lock — CI/CD series** — andrewlock.net
