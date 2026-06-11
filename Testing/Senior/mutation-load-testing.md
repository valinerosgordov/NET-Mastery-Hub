---
tags: [testing, mutation, stryker, load-testing, nbomber, k6]
level: Senior
date: 2026-04-30
---

# Mutation & Load Testing

> Mutation testing проверяет качество **тестов**. Load testing — performance под нагрузкой. Закрывает: Stryker.NET для mutation, NBomber и k6 для load tests, performance testing patterns, CI integration.

---

## Что это, зачем и когда

### Mutation testing — что

**Тестирует твои тесты**: специально вносит изменения (mutations) в код и проверяет, поймали ли тесты эти изменения.

```csharp
public bool IsAdult(int age) => age >= 18;
```

Stryker создаст mutations:
- `age >= 18` → `age > 18` (boundary)
- `age >= 18` → `age <= 18`
- `age >= 18` → `false`

Если **твои тесты не отлавливают** эти изменения — mutation **выживает** = твои тесты слабые.

```
Mutation score: 87.50% (35 killed, 5 survived)
```

### Зачем

| Без mutation testing | С mutation testing |
|----------------------|---------------------|
| 90% coverage — но тесты могут быть theatre | Проверка что тесты ловят bugs |
| `Assert.True(true)` proходит — false confidence | Mutation вскрывает useless assertions |
| Edge cases остаются непротестированными | Видно конкретные пропущенные случаи |
| Reviewer пишет "добавь больше тестов" — расплывчато | Stryker конкретно говорит — какие |

### Load testing — что

Симуляция нагрузки на систему — десятки/тысячи concurrent users, замер latency, throughput, errors.

### Зачем load testing

- **Capacity planning** — сколько RPS выдержим до falling over?
- **Bottleneck detection** — где узкое место (CPU, DB, network)?
- **Regression** — после релиза проверить что perf не упал
- **SLA validation** — p99 latency < 500ms на 1000 RPS подтверждено?

---

## 1. Stryker.NET — mutation testing

[stryker-mutator.io](https://stryker-mutator.io)

### Установка

```bash
dotnet tool install -g dotnet-stryker
```

### Запуск

```bash
cd MyProject.Tests
dotnet stryker
```

Output:
```
Mutation score:               87.50%
Mutants killed:               35
Mutants survived:              5
Mutants timeout:               0
Mutants no coverage:           2
```

### Configuration — `stryker-config.json`

```json
{
    "stryker-config": {
        "project": "MyProject.csproj",
        "test-projects": ["MyProject.Tests/MyProject.Tests.csproj"],
        "thresholds": {
            "high": 80,
            "low": 60,
            "break": 50
        },
        "reporters": ["html", "console", "json"],
        "mutate": [
            "src/MyProject.Domain/**.cs",
            "!src/MyProject.Domain/Migrations/**.cs"
        ],
        "ignore-methods": ["ToString", "GetHashCode", "Equals"],
        "concurrency": 4
    }
}
```

| | Что |
|--|------|
| `thresholds.break` | Если score ниже — exit code != 0 |
| `mutate` | Только эти файлы мутируем |
| `ignore-methods` | Не мутируем эти методы |
| `concurrency` | Параллельно сколько mutations |

### Reports

```bash
dotnet stryker --reporter html
# Output: StrykerOutput/<timestamp>/reports/mutation-report.html
```

HTML отчёт: видишь каждую mutation, killed/survived, в каком файле/строке.

### Mutation types

Stryker meняет:

```csharp
// Arithmetic
a + b → a - b
a * b → a / b

// Equality
== → !=
> → <
>= → <

// Boolean
&& → ||
true → false

// String
"hello" → ""

// Method calls
arr.Length → 0

// Conditional
if (x) → if (true) → if (false)
```

### Real example — поймали bug

```csharp
// Production code
public decimal CalculateDiscount(decimal price, int quantity)
{
    if (quantity > 10) return price * 0.1m;
    return 0;
}

// Тест — слабый
[Fact]
public void CalculateDiscount_returns_value()
{
    var result = _calc.CalculateDiscount(100, 15);
    result.ShouldBeGreaterThan(0);  // ⚠️ Только проверяет > 0
}
```

Stryker мутирует `quantity > 10` → `quantity >= 10`. Тест **проходит** на mutation (т.к. `15 >= 10` тоже true). Mutation выживает.

```csharp
// ✅ Сильный тест
[Theory]
[InlineData(10, 0)]      // boundary — 10 это edge, скидки нет
[InlineData(11, 11.0)]   // едва-едва
[InlineData(15, 15.0)]
public void CalculateDiscount_at_boundaries(int qty, decimal expected)
{
    var result = _calc.CalculateDiscount(100, qty);
    result.ShouldBe(expected);
}
```

Теперь mutation `> 10` → `>= 10` поломает тест на `qty=10`.

### Когда (не) использовать mutation testing

✅ **Хорошо:**
- Critical business logic (payment, auth, pricing)
- После уже >80% coverage — проверить качество тестов
- Domain core (где баги дороги)

❌ **Не нужно:**
- DTOs, entities без логики (мутировать нечего)
- Generated code
- UI / framework integration (тяжело mock'ать всё)
- Каждый PR (slow!) — schedule weekly

### CI integration

```yaml
# .github/workflows/mutation.yml
name: Mutation Testing
on:
  schedule: [{ cron: "0 6 * * 1" }]  # weekly Monday 06:00
  workflow_dispatch:  # manual

jobs:
  mutation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with: { dotnet-version: '10.0.x' }

      - name: Install Stryker
        run: dotnet tool install -g dotnet-stryker

      - name: Run Stryker
        run: |
          cd src/MyProject.Tests
          dotnet stryker --break-at 70 --reporter json --reporter html

      - name: Upload report
        uses: actions/upload-artifact@v4
        with:
          name: mutation-report
          path: '**/StrykerOutput/**/reports/*.html'
```

---

## 2. NBomber — load testing на C#

[nbomber.com](https://nbomber.com)

```bash
dotnet add package NBomber
dotnet add package NBomber.Http
```

### Базовый scenario

```csharp
using NBomber.CSharp;
using NBomber.Http.CSharp;

var httpClient = new HttpClient();

var scenario = Scenario.Create("get_orders", async ctx =>
{
    var response = await httpClient.GetAsync("https://api.example.com/orders");

    return response.IsSuccessStatusCode
        ? Response.Ok(statusCode: response.StatusCode.ToString())
        : Response.Fail(statusCode: response.StatusCode.ToString());
})
.WithLoadSimulations(
    // Постепенно поднимаем нагрузку
    Simulation.RampingInject(
        rate: 100,
        interval: TimeSpan.FromSeconds(1),
        during: TimeSpan.FromSeconds(30)),
    // Держим 100 одновременных пользователей
    Simulation.KeepConstant(
        copies: 100,
        during: TimeSpan.FromMinutes(2))
);

NBomberRunner
    .RegisterScenarios(scenario)
    .Run();
```

### Output

```
=================================================================
Scenario: get_orders
=================================================================
| step | total | ok    | fail | RPS  | min  | mean  | p95   | p99  |
|------|-------|-------|------|------|------|-------|-------|------|
| GET  | 18000 | 17995 | 5    | 150  | 12ms | 45ms  | 120ms | 280ms|
=================================================================
```

### Load simulations

| Тип | Что |
|-----|-----|
| `Inject(rate, interval, during)` | Constant rate (e.g., 100 RPS) |
| `RampingInject(rate, interval, during)` | Linear ramp до rate |
| `KeepConstant(copies, during)` | Постоянное число concurrent users |
| `RampingConstant(copies, during)` | Linear ramp users |
| `IterationsInject(count)` | Total iterations (не time-based) |

### Multi-step scenarios

```csharp
var scenario = Scenario.Create("checkout_flow", async ctx =>
{
    // Step 1: Login
    var loginResp = await httpClient.PostAsJsonAsync("/login", new { ... });
    if (!loginResp.IsSuccessStatusCode) return Response.Fail();
    var token = await loginResp.Content.ReadAsStringAsync();

    httpClient.DefaultRequestHeaders.Authorization = new("Bearer", token);

    // Step 2: Add to cart
    var addResp = await httpClient.PostAsJsonAsync("/cart", new { productId = 1 });
    if (!addResp.IsSuccessStatusCode) return Response.Fail();

    // Step 3: Checkout
    var checkoutResp = await httpClient.PostAsync("/checkout", null);

    return checkoutResp.IsSuccessStatusCode
        ? Response.Ok()
        : Response.Fail();
});
```

### Data feeds — параметризованные тесты

```csharp
var users = Bogus.Faker.Generate(1000);
var feed = DataFeed.CreateRandom("users", users);

var scenario = Scenario.Create("login", async ctx =>
{
    var user = ctx.Data.Get<User>(feed);
    var resp = await httpClient.PostAsJsonAsync("/login", user);
    return resp.IsSuccessStatusCode ? Response.Ok() : Response.Fail();
});
```

### CI integration с thresholds

```csharp
NBomberRunner
    .RegisterScenarios(scenario)
    .WithReportFormats(ReportFormat.Html, ReportFormat.Md, ReportFormat.Csv)
    .WithReportFolder("reports")
    .WithTestSuite("api_load")
    .WithTestName("checkout_flow")
    .Run();

// Thresholds для CI
var stats = NBomberRunner.RegisterScenarios(scenario).Run();

if (stats.AllOkCount < stats.AllRequestCount * 0.99)
{
    Environment.Exit(1);  // > 1% errors — fail
}

if (stats.ScenarioStats[0].Ok.Latency.Percent99 > 500)
{
    Environment.Exit(1);  // p99 > 500ms — fail
}
```

---

## 3. k6 — JS-based, mature

[k6.io](https://k6.io)

Не C#, но **стандарт индустрии** для load testing. Лучше чем NBomber для большинства задач.

### Установка

```bash
# macOS
brew install k6

# Linux
sudo apt install k6

# Windows
choco install k6
```

### Базовый тест

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
    stages: [
        { duration: '30s', target: 100 },   // ramp up
        { duration: '2m', target: 100 },    // stay at 100
        { duration: '30s', target: 0 },     // ramp down
    ],
    thresholds: {
        http_req_duration: ['p(95)<500'],   // 95% < 500ms
        http_req_failed: ['rate<0.01'],     // <1% errors
    },
};

export default function () {
    const res = http.get('https://api.example.com/orders');

    check(res, {
        'status is 200': (r) => r.status === 200,
        'response time < 500ms': (r) => r.timings.duration < 500,
    });

    sleep(1);
}
```

```bash
k6 run load-test.js
```

### Distributed k6 — для миллионов RPS

```bash
# k6 Cloud (платный)
k6 cloud load-test.js

# Self-hosted: k6-operator на Kubernetes
kubectl apply -f k6-operator.yaml
```

### Spike test

```javascript
export const options = {
    stages: [
        { duration: '10s', target: 100 },
        { duration: '1m', target: 100 },
        { duration: '10s', target: 1400 },  // spike!
        { duration: '3m', target: 1400 },
        { duration: '10s', target: 100 },
        { duration: '3m', target: 100 },
        { duration: '10s', target: 0 },
    ],
};
```

Проверяет: переживёт ли система внезапный bump.

---

## 4. Сравнение NBomber vs k6

| | NBomber | k6 |
|--|---------|-----|
| Язык | C# | JavaScript |
| Reuse существующих C# DTOs | ✅ | ❌ Нужно дублировать |
| Maturity | Growing | ✅ Industry standard |
| Distributed | Limited | ✅ k6 Cloud / k8s operator |
| Reports | ✅ Built-in | Через Grafana / Datadog |
| Integration с .NET CI | Easier | Через Docker container |
| Performance | Good | Excellent |
| Community | Smaller | Massive |

**Когда NBomber:** простые сценарии, .NET-only команда, не нужен massive scale.

**Когда k6:** standard для serious load testing, distributed, Grafana integration.

---

## 5. Что измерять

### Latency percentiles

- **p50 (median)** — типичный пользователь
- **p95** — 95% запросов быстрее этого
- **p99** — последний 1% — обычно медленный (GC pauses, cold cache)
- **p99.9** — для critical systems

```
p50:   45ms     ← typical
p95:   180ms    ← acceptable
p99:   450ms    ← long tail
p99.9: 2000ms   ← outliers (GC, I/O burst)
```

### Throughput (RPS)

Запросов в секунду которое система обрабатывает stable.

### Error rate

% запросов которые failed (5xx, timeouts, etc).

### Resource usage

CPU, RAM, network, disk I/O серверов под нагрузкой.

### Apdex (Application Performance Index)

```
Apdex = (Satisfied + Tolerating/2) / Total
```

- Satisfied: response < target_T
- Tolerating: response < 4 × target_T
- Frustrated: response > 4 × target_T

Apdex > 0.94 = excellent, < 0.5 = unacceptable.

---

## 6. Patterns

### Baseline test

```javascript
// Каждый release — same test
export const options = {
    stages: [
        { duration: '5m', target: 50 },
    ],
    thresholds: {
        http_req_duration: ['p(95)<300'],
    },
};
```

Сравниваешь между builds → catch performance regressions.

### Stress test

Нагружай **до failure** — где система ломается?

```javascript
export const options = {
    stages: [
        { duration: '2m', target: 100 },
        { duration: '5m', target: 500 },
        { duration: '5m', target: 1000 },
        { duration: '5m', target: 2000 },
        { duration: '5m', target: 5000 },  // система падает где-то?
    ],
};
```

### Soak test

Длительная нагрузка — найти memory leaks, slow degradation.

```javascript
export const options = {
    stages: [
        { duration: '5m', target: 100 },
        { duration: '4h', target: 100 },  // ⏱️ 4 часа!
        { duration: '5m', target: 0 },
    ],
};
```

### Spike test

См. выше — резкий bump.

---

## 7. Performance budgets

Для каждого endpoint — **budget**:

```csharp
public class PerformanceBudgets
{
    [Fact]
    public async Task GetOrders_p95_under_200ms()
    {
        // Setup, run NBomber...

        var p95 = stats.ScenarioStats[0].Ok.Latency.Percent95;
        p95.ShouldBeLessThan(200);  // p95 < 200ms
    }

    [Fact]
    public async Task GetOrders_throughput_at_least_500rps()
    {
        var rps = stats.ScenarioStats[0].Ok.Request.RPS;
        rps.ShouldBeGreaterThan(500);
    }
}
```

В CI — fail если budget exceeded.

---

## 8. Production monitoring vs load testing

| | Load Test | Production Monitoring |
|--|-----------|------------------------|
| Когда | Pre-release / scheduled | Always |
| Конкретный traffic | Synthetic | Real users |
| Goal | Capacity planning | SLO tracking |
| Tools | NBomber, k6 | OpenTelemetry, Prometheus, Datadog |

См.[[observability|Observability]] — production monitoring.

---

## 9. Best Practices

### Mutation testing

- **Schedule weekly** — не каждый PR (slow)
- **Threshold 75-85%** — не 100% (diminishing returns)
- **Focus on domain core** — не на DTOs, не на UI
- **Anti-pattern: weak assertions** — `result.ShouldNotBeNull()`. Используй конкретные expected values
- **Boundary tests** — это первое что Stryker ловит

### Load testing

- **Тесты в production-like окружении** — staging / dedicated environment
- **Тесты не должны влиять на production users** — отдельный hostname / DB
- **Realistic data** — не пустые DBs, реальный distribution
- **Realistic user behavior** — sleep между requests, разные scenarios
- **Warmup phase** — JIT, cache priming
- **Multiple runs** — variance важна, не один прогон
- **Report comparison** — текущий vs baseline
- **Performance budgets в CI** — break build при regressions
- **Monitor everything** — APM during load test (CPU, memory, GC, DB)

---

## 10. Когда что — резюме

| Scenario | Tool |
|----------|------|
| Critical business logic — quality of tests | **Stryker.NET** mutation |
| API capacity planning | **NBomber** или **k6** |
| Distributed massive scale (millions RPS) | **k6 Cloud** или k6 на k8s |
| Pre-release perf regression check | **NBomber** в CI |
| 24x7 production monitoring | **OpenTelemetry + Grafana/Datadog** |
| Specific endpoint perf budget | **NBomber** in xUnit test |
| Comparing optimization | **BenchmarkDotNet** (micro-benchmarks) |

---

## Case Studies

### Case Study #1 — Refactoring legacy "god class"

**Сценарий:** `OrderManager` класс — 3000 строк, 50 методов, 20 dependencies. Невозможно testить.

**Strategy:**
1. Identify responsibilities — найти SRP violations
2. Extract interfaces — `IOrderRepository`, `IPaymentService`, `INotificationService`
3. Move methods → small focused classes
4. Tests перед refactoring (characterization tests)
5. Refactor пошагово, run tests после каждого step

**Result:**
- 3000 строк → 8 классов по 200-400 строк each
- Test coverage: 5% → 80%
- Onboarding new developer: 2 weeks → 3 days

---

### Case Study #2 — Code review где AI помогает

**Сценарий:** Senior просматривает PR от Junior'а. Стандартные issues: naming, missing nulls, dead code.

**Workflow:**
1. **AI первый pass** — Copilot Chat / Claude review кода
2. **Senior валидирует AI feedback** — отбрасывает false positives
3. **Senior фокусируется на architecture** — что AI не видит:
   - Domain logic correctness
   - Business rule violations
   - Performance implications
   - Security in context

**Time saved:** 30 min review → 10 min (AI на mechanical, senior на important).

См.[[ai-coding-tools|AI Coding Tools]].

---

### Case Study #3 — Static analysis для legacy

**Сценарий:** 200K LOC unmaintained code. Hidden bugs, технический долг.

**Tools setup:**
```xml
<!-- Directory.Build.props -->
<Project>
  <PropertyGroup>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="*" />
    <PackageReference Include="StyleCop.Analyzers" Version="*" />
    <PackageReference Include="SonarAnalyzer.CSharp" Version="*" />
  </ItemGroup>
</Project>
```

**Phased approach:**
1. Week 1: Fix critical security issues (secrets, SQL injection)
2. Week 2-4: Fix top-10 most violated rules
3. Month 2: Enable analyzer warnings as errors на новый код
4. Quarter: Full compliance + maintain через CI gate

См. [[static-analysis|Static Analysis]] и[[cicd-github-actions|CI/CD]].


---

## Cheat sheet

| Quality concern | Tool / Practice |
|-----------------|-----------------|
| Code style enforcement | EditorConfig + dotnet format |
| Static analysis | Microsoft.CodeAnalysis.NetAnalyzers |
| Style rules | StyleCop.Analyzers |
| Security scanning | SonarAnalyzer.CSharp, GitHub CodeQL |
| Vulnerability scanning | `dotnet list package --vulnerable` |
| Outdated packages | `dotnet list package --outdated` |
| Dead code | ReSharper / dotnet-ide-cli `unused-code` |
| Cyclomatic complexity | NDepend, SonarQube |
| Test coverage | coverlet + ReportGenerator |
| Mutation testing | Stryker.NET |
| Architecture tests | NetArchTest, ArchUnitNET |
| Contract tests | PactNet, Pact.NET |
| Code review | PR + GitHub Copilot review |
| Pre-commit hooks | Husky.NET + lint-staged |
| CI quality gate | SonarCloud / Codacy |

| Refactoring smell | Action |
|-------------------|--------|
| Long method (50+ lines) | Extract method |
| Long parameter list (4+) | Parameter object |
| Duplicate code | Extract to function/class |
| Switch statement | Polymorphism (Strategy) |
| Feature envy | Move method к нужному classу |
| Data clumps | Wrap в class/record |
| Primitive obsession | Value objects (Money, Email) |
| God class | Split по SRP |
| Shotgun surgery | Cohesion problem — restructure |


---

## Decision tree

```
Quality issue?
│
├── Code style inconsistencies?
│   → EditorConfig + dotnet format в pre-commit
│
├── Hidden bugs / vulnerabilities?
│   ├── Logic bugs → Roslyn analyzers
│   ├── Security → SonarAnalyzer + CodeQL
│   └── Vulnerabilities → npm audit equivalent для NuGet
│
├── Test quality concerns?
│   ├── Coverage low → coverlet + minimum threshold в CI
│   ├── Tests pass but bugs ship → mutation testing (Stryker)
│   └── Flaky tests → identify + isolate (TestCategory)
│
├── Architectural drift?
│   ├── Boundaries violated → NetArchTest assertions в tests
│   ├── Dependencies wrong direction → dependency cruiser
│   └── Anti-patterns spreading → SonarQube + custom rules
│
├── Big tech debt?
│   ├── Identify → "boy scout rule" — лучше чем нашёл
│   ├── Plan → backlog с estimate
│   ├── Critical → 20% sprint capacity
│   └── Refactor → tests first, small steps
│
└── Code review bottleneck?
    ├── Junior уровень → AI first pass
    ├── Standards inconsistent → automated checks в CI
    └── Slow → smaller PRs, clear conventions
```


---

## См. также

- [[testing|Testing — общие principles]]
- [[integration-testing|Integration Testing]]
-[[performance|Performance — BenchmarkDotNet]]
-[[hft-low-latency|HFT / Low Latency]]
-[[observability|Observability]] — production metrics
-[[static-analysis|Static Analysis]]

## Reading list

- **Stryker.NET docs** — stryker-mutator.io/docs/stryker-net
- **NBomber docs** — nbomber.com/docs
- **k6 docs** — k6.io/docs
- **Brendan Gregg — Systems Performance** (книга, classic)
- **Designing Data-Intensive Applications** — Kleppmann (chapter про perf)
- **Pact Foundation** — pact.io (contract testing)
- **The Art of Capacity Planning** — John Allspaw (когда сколько RPS)
