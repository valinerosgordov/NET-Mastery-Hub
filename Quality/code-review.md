---
tags: [quality, code-review, pull-request, collaboration]
level: All
date: 2026-04-30
---

# Code Review — практика и культура

> Полный гайд по code review: что искать, как давать feedback, как принимать критику, anti-patterns, growth-oriented review. Для author и reviewer.

---

## Что это, зачем и когда

### Что такое code review?

**Process в котором другой разработчик читает твой код перед merge** — ищет баги, design issues, дает обратную связь.

**Аналогия:** Корректор в журналистике. Журналист пишет статью — корректор проверяет. Не "редактирует за тебя", а помогает improve до публикации.

### Зачем code review

| Без code review | С code review |
|------------------|---------------|
| Баги обнаруживаются юзерами | Ловятся до prod |
| Каждый знает только свой код | Knowledge sharing |
| Junior учится медленно (только сам) | Учится у Senior через review |
| Разный стиль кода | Постепенно сходится к команде |
| Architecture drift | Senior видит когда new code не fit |
| Критические баги в security / perf | Reviewer ловит когда автор не подумал |

### Цели code review

| Цель | Приоритет |
|------|-----------|
| **Найти баги** | High |
| **Обмен знаниями** | High |
| **Поддержать стандарты** | High |
| **Mentoring** | High |
| **Architecture review** | Medium |
| **Style nitpicking** | LOW (это для analyzers!) |

> [!info] Самое важное правило
> **Reviewer и author работают над одной целью** — улучшить код. Это **collaboration**, не **критика друг друга**.

---

## 1. Author — как готовить PR

### 1. Маленькие PRs

| Размер | Quality of review |
|--------|-------------------|
| <100 lines | Excellent — reviewer reads carefully |
| 100-300 lines | Good |
| 300-500 lines | Mediocre — reviewer skims |
| 500-1000 lines | Poor — "LGTM" stamp |
| >1000 lines | Rubber stamp — wastes everyone's time |

**Цель: <300 lines per PR**.

```
✅ Малый PR
  feature/notifications-add-email-template     +120 -40

❌ Большой PR
  feature/notifications-system                 +2400 -800
```

Большую feature разбивай на куски:
- PR 1: Add database schema for notifications
- PR 2: Add NotificationService class (skeleton)
- PR 3: Add email template engine
- PR 4: Wire up notifications to user actions
- PR 5: Add UI for notification preferences

### 2. Self-review перед открытием

```
Before "Create Pull Request":
1. Просмотри diff целиком в GitHub UI (не в IDE)
2. Каждое изменение — оправдано?
3. Не оставил debug-prints?
4. Все TODO имеют ticket?
5. Тесты passing?
6. Linter clean?
```

Self-review ловит 50% issues. Reviewer тратит время на остальные 50%.

### 3. Хороший PR description

```markdown
## What

Adds email notifications when an order is placed.

## Why

JIRA-1234. Customer support requested feature.

## How

- New `EmailNotificationService` implementing `INotificationService`
- Wired up to `OrderPlacedEvent` via MediatR notification handler
- Email template stored in `wwwroot/templates/order-placed.html`
- Uses existing `ISmtpClient` infrastructure

## Testing

- Added unit tests for `EmailNotificationService`
- Added integration test that places order and verifies email sent (via WireMock)
- Manually tested with real SMTP — email received

## Screenshots

[email screenshot]

## Migrations

None.

## Deployment notes

Add SMTP credentials to environment:
- `SMTP_HOST`
- `SMTP_USER`
- `SMTP_PASS`
```

Чек-лист:
- ✅ **What** — что делает
- ✅ **Why** — зачем (с ticket link)
- ✅ **How** — high-level approach
- ✅ **Testing** — как проверял
- ✅ Screenshots если UI
- ✅ Migrations / deployment notes

### 4. Атомарные commits

```
✅
  - "Add EmailNotificationService"
  - "Add unit tests for EmailNotificationService"
  - "Wire up to OrderPlacedEvent"
  - "Update README with SMTP setup"

❌
  - "stuff"
  - "fix"
  - "more changes"
  - "wip"
```

### 5. Не привлекай personal

```
❌ "Why are you doing this so weirdly?"
❌ "This is bad code"
✅ "What do you think about <alternative>? It might handle <edge case> better."
```

---

## 2. Reviewer — как делать review

### 1. SLA — review в течение 24h

```
PR opened → reviewer assigned → review started
   1h        4h max              24h max
```

Если задерживаешь — **снизишь team velocity**. Каждый PR висит 3 дня → 30 PRs в очереди → катастрофа.

Reserve 1-2 hours/day **only for reviews**.

### 2. Approach: top-down

#### Шаг 1: Понять "что и зачем"

Прочитай PR description. Если непонятно — задай вопрос **до** review кода.

#### Шаг 2: Architecture / design

- Это правильное место для этого кода?
- Соответствует ли архитектурным решениям проекта?
- Reusable ли подход или narrow?
- Trade-offs обсуждены?

#### Шаг 3: Logic / correctness

- Edge cases покрыты?
- Error handling?
- Race conditions?
- Memory leaks?
- Security issues?

#### Шаг 4: Tests

- Тесты test реальные кейсы?
- Mocks правильные?
- Не over-mocking (test theatre)?

#### Шаг 5: Code style (LAST!)

- Naming clear?
- Comments только где нужно?
- Magic numbers?

> [!info] Style — последнее
> Если architecture проблема — стиль не имеет значения, всё равно перепишется. Сначала big issues, потом nits.

### 3. Levels of feedback

Помечай severity:

```
🔴 BLOCKING — must fix before merge
   "Bug: this throws NullReferenceException when X is null"
   "Security: SQL injection in line 45"
   "Architecture: this leaks domain into controller"

🟡 SUGGEST — improvement, not blocking
   "Consider using LINQ for readability:
    items.Where(x => x.IsActive).ToList()"

🟢 NIT — minor preference
   "Nit: extra blank line"
   "Nit: this could be more readable as switch expression"

❓ QUESTION — нужно понять
   "Why we need this check? Isn't it covered by validator?"

📚 LEARN — share knowledge
   "FYI: .NET 8+ has IExceptionHandler — cleaner approach.
    Not blocking, but consider migrating later."
```

Approve PR если только 🟢 / ❓ / 📚 — не блокируй на nits.

### 4. Конкретность

```
❌ "This is wrong"
✅ "This will fail when input is empty because Length-1 underflows. Add a guard: if (input.Length == 0) return ..."

❌ "Refactor this"
✅ "Consider extracting these 3 validation steps into separate methods: ValidateName, ValidateEmail, ValidateAge"

❌ "Add tests"
✅ "Please add a test for the case when X is null — currently uncovered"
```

### 5. Provide alternatives

````
❌ "Don't use this approach"
✅ "I'd suggest <X> approach because <reason>. Example:

```csharp
// instead of
foreach (var item in items)
    if (item.IsActive) result.Add(item);

// could be
var result = items.Where(i => i.IsActive).ToList();
```
"
````

### 6. Acknowledge good work

```
✅ "Nice — this caching strategy is way cleaner than the old one"
✅ "Good catch on the null check — I missed this in the original code"
✅ "Love how you organized this into VSA — much easier to navigate"
```

Code review **не только про критику**. Положительная feedback — мотивирует.

---

## 3. Что искать (checklist)

### Functional correctness

- [ ] Делает что описано в PR description?
- [ ] Edge cases покрыты (null, empty, max int, negative, large data)?
- [ ] Error handling правильный?
- [ ] Возможны race conditions?
- [ ] Не throws unexpected exceptions?

### Bugs

- [ ] Off-by-one errors (`<` vs `<=`)
- [ ] Null reference exceptions
- [ ] Boundary conditions (empty list, single item, max)
- [ ] Async issues (deadlock, sync over async, missing await)
- [ ] Concurrent modification of collections
- [ ] Resource leaks (missing using / dispose)

### Security

- [ ] SQL injection (raw SQL, dynamic LINQ)
- [ ] XSS (HTML output без escaping)
- [ ] CSRF на mutating endpoints
- [ ] Secrets в коде / git
- [ ] Authorization на каждом endpoint
- [ ] Input validation
- [ ] Sensitive data в logs

### Performance

- [ ] N+1 query (EF Core)
- [ ] Loops с сложностью O(n²) на больших данных
- [ ] Allocations в hot path (LINQ, string concat)
- [ ] Async всё что I/O bound
- [ ] CancellationToken пробрасывается
- [ ] Caching где имеет смысл

### Code quality

- [ ] Имена переменных / методов / классов клевые
- [ ] Методы маленькие, делают одну вещь
- [ ] Нет дублирования
- [ ] DRY не over-applied (concepts могут быть похожи но независимы)
- [ ] Magic numbers — const с именем
- [ ] Нет закомментированного кода

### Tests

- [ ] Тесты test поведение, не implementation
- [ ] Test names descriptive (что тестируют)
- [ ] Arrange-Act-Assert pattern
- [ ] Edge cases в тестах
- [ ] Не over-mocking
- [ ] Тесты быстрые (<500ms each)

### Architecture

- [ ] Соответствует архитектуре проекта
- [ ] Domain не leak в Controller / View
- [ ] Не add dependencies в Domain
- [ ] DI lifetimes правильные (Singleton не depend на Scoped)
- [ ] Соответствует SOLID

### Documentation

- [ ] Public API имеет XML doc comments
- [ ] Сложные части — комментарии "почему" (не "что")
- [ ] README обновлён если новая feature
- [ ] CHANGELOG / migration notes

---

## 4. Anti-patterns

### Author anti-patterns

#### 1. PR drop-and-go

```
Author: opens PR
Author: never responds to feedback
Reviewer: 😡
```

Right approach: reply to comments в течение 24h, даже "good point, will fix today".

#### 2. Defensive

```
Reviewer: "Consider X here"
Author: "I think it's fine the way it is"
Author: pushes minor change but not the suggestion
```

Не значит ты обязан соглашаться. Но **обоснуй**:

```
Author: "I considered X but Y because <reason>. WDYT?"
```

#### 3. Force-push

После review reviewer'а — `git push --force` сtаирает history. Reviewer не видит что изменилось с прошлого review.

```
✅ Push дополнительных commits после feedback
✅ Squash при final merge
❌ Force-push в середине review
```

#### 4. Massive PR

См. выше — > 500 lines = мало кто реально reviewer.

### Reviewer anti-patterns

#### 1. Bikeshedding

```
Reviewer: 50 nits about whitespace and naming
PR architecture has critical bug — not mentioned
```

Сначала big issues, потом nits. Не блокируй на стиле — это **analyzers' job**.

#### 2. "I would have done it differently"

```
Reviewer: "I would have used pattern X"
Code uses pattern Y which works fine
```

Если не **wrong**, а **different style** — не блокируй. PR author имеет свой стиль.

#### 3. Ghost reviewer

```
Reviewer: assigned to PR
Reviewer: doesn't respond for 5 days
```

Если ты не reviewer — say so и unassign.

#### 4. Personal attacks

```
❌ "Why would you do that?"
❌ "This is amateur"
❌ "Did you even test this?"
```

```
✅ "I think this might fail in case <X>. Could you check?"
✅ "There may be an issue — let's discuss"
```

#### 5. Massive feedback на PR

```
Reviewer: 50 comments на 200 line PR
```

Если **architecture** wrong — stop, обсуди offline / pair, не пиши 50 comments. Один call > 50 comments.

#### 6. "Approve with comments" = "Approve"

```
Reviewer: "Approve! But please fix X, Y, Z"
Author: merges без fixing
```

Если **must** fix — request changes, не approve. Approve = "ready to merge".

---

## 5. Conflicts — как разруливать

### Author и Reviewer не согласны

```
Reviewer: "I think X is better"
Author: "I think Y is better"
```

#### Workflow

1. **Try to understand** — спроси reviewer почему X лучше
2. **Provide context** — author объясняет why Y
3. **Compromise** — может быть Z (третий вариант)
4. **Escalate** — если не сходимся, пригласи 3rd person (другой Senior, Tech Lead)
5. **Disagree and commit** — author has ownership, reviewer **suggested** улучшение

> [!info] "Disagree and Commit"
> Если reviewer не согласен но не critical — author принимает решение. Reviewer commits to support implementation.

### Когда blocking

Blocking PR оправдано когда:

✅ **Yes block:**
- Bug, корректность нарушается
- Security issue
- Breaks architecture (нарушает principle проекта)
- Серьёзный perf issue
- Tests отсутствуют для critical logic

❌ **Don't block:**
- Style preference (use analyzers)
- Personal taste
- "I would do differently"

---

## 6. Code Review tools

### GitHub PR review

Standard для open-source. Comments, suggestions, batched feedback.

### GitLab Merge Request

Similar to GitHub.

### Azure DevOps Pull Request

Для Microsoft / Azure shops.

### Gerrit

Used by Google. Stricter (must approve every change).

### Reviewable.io

Layer on top of GitHub. Better для большие PRs.

### Tools-based

#### CodeRabbit / Cursor

AI-assisted review. Бот сам читает PR, оставляет comments. Хорошо как first pass.

#### SonarQube on PR

См. [[static-analysis|Static Analysis]]. Quality gate в PR — block если quality regression.

---

## 7. Review culture в команде

### Established team

Опытные ребята, established practices:

- **2+ reviewer approvals** required для merge
- **Tests + lint pass** required
- **Owner of file** automatic reviewer (CODEOWNERS)
- **Trust** — quick approval если автор senior

### New team / juniors mostly

- **Tech lead reviews everything** (training mode)
- **Pair programming** для complex tasks
- **Review meeting** weekly — go through PRs together (learning)
- **More verbose feedback** — explanations

### Mixed seniority

- **Junior reviews Junior** — learning opportunity
- **Senior reviews Senior** — different perspective
- **Senior must approve any major refactor**
- **Cross-team review** для architecture changes

---

## 8. Async vs sync review

### Async (default)

Author opens PR → reviewers comment async → author responds.

✅ Pro:
- Не блокирует author
- Reviewer reads when have time

❌ Con:
- Long PR cycles (2-3 дня)
- Cold context switches

### Sync (pair review)

Author + reviewer in same call (Zoom / in-person).

✅ Pro:
- Quick resolution
- Knowledge sharing
- Junior learns deeply

❌ Con:
- Time-consuming
- Both должны быть available

**Mix:** Async по default, sync для complex / architecture / pair learning.

---

## 9. Self-review patterns

Перед открытием PR:

### 1. Read diff in browser

GitHub UI показывает diff иначе чем IDE — иначе видишь.

### 2. Read commits one by one

Каждый commit — atomic unit. Понятно что делал?

### 3. Read out loud

```
"Adding NotificationService class with..."
"Wait, why am I making this internal? It should be public for tests..."
```

### 4. "Stranger" test

Если бы я **впервые** видел этот код — понял бы что он делает?

### 5. Diff против main

```bash
git diff main..HEAD
```

Видишь whole change в context of main.

---

## 10. Code Review для Junior (specifically)

### Author (Junior)

- **Don't be defensive** — feedback не на тебя личное
- **Ask "why"** — understand reasoning, learn
- **Take notes** — повторяющийся feedback = blind spot
- **Don't take shortcuts** — proper fix > workaround чтобы PR прошёл
- **Ask for help early** — лучше pair-program в начале чем 5 review iterations

### Reviewer reviewing Junior

- **Be encouraging** — first PRs scary
- **Explain why** — не "fix this", а "fix this because X"
- **Suggest alternatives** — учи patterns
- **Less nitpick, more big picture** — junior пока не освоил basics, не наваливай 100 comments
- **Pair if confused** — иногда easier learn synchronously

---

## 11. Common Pitfalls

### 1. Big PR opened on Friday afternoon

Reviewer не успевает before weekend. PR висит до понедельника.

**Лечение:** open big PRs Monday-Wednesday.

### 2. "Drive-by" reviews

Reviewer skims за 5 минут, approve, не нашёл реальные issues.

**Лечение:** при assignment — block 30+ min. Если нет времени — decline.

### 3. Multi-reviewer dilution

5 reviewers assigned → каждый думает "другие проверят" → реально проверяет никто.

**Лечение:** 1-2 reviewer + automated CI gates.

### 4. Re-review fatigue

После 5 round reviews — reviewer устал, "approve чтобы избавиться".

**Лечение:** maximum 2-3 review rounds. Если больше — pair program.

### 5. Author "fixes" but doesn't show

```
Author: pushes new commit "fix review feedback"
Reviewer: doesn't see what changed (not re-requested review)
```

**Лечение:** author respond to comments + re-request review.

---

## 12. Best Practices summary

### Author

- **PRs <300 lines**
- **Self-review** перед opening
- **Atomic commits**
- **Хороший description** (what, why, how, testing)
- **Respond within 24h**
- **Re-request review** после fixes
- **Don't take feedback personal**
- **Approve responses** explicit ("done" / "addressed in commit X")

### Reviewer

- **Review within 24h**
- **Top-down** — architecture > logic > style
- **Severity tags** — blocking / suggest / nit
- **Provide alternatives** — не just "this is wrong"
- **Acknowledge good work**
- **Don't bikeshed**
- **Don't block on style**
- **Concrete + actionable** feedback

### Team

- **CODEOWNERS** для auto-assignment
- **Required review checks** в branch protection
- **CI gates** для quality (tests + lint + analyzers)
- **Reviews are part of work** — protect time для них
- **Culture of psychological safety** — feedback ≠ attack

---

## 13. Templates

### PR template

`.github/PULL_REQUEST_TEMPLATE.md`:

```markdown
## What

[Brief description of changes]

## Why

[Link to ticket / explanation]

## How

[High-level approach, key decisions]

## Testing

- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manually tested

## Migrations

[None / Description of migration]

## Deployment notes

[None / Special considerations]

## Screenshots

[If applicable]
```

### Review template

```markdown
🔴 BLOCKING:

🟡 SUGGEST:

🟢 NIT:

❓ QUESTION:

📚 LEARN:

Overall: [LGTM / Comments / Request changes]
```

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
1. **AI первый pass** — Copilot Chat review кода
2. **Senior валидирует AI feedback** — отбрасывает false positives
3. **Senior фокусируется на architecture** — что AI не видит:
   - Domain logic correctness
   - Business rule violations
   - Performance implications
   - Security in context

**Time saved:** 30 min review → 10 min (AI на mechanical, senior на important).

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

- [[clean-code|Clean Code]] — что делает код хорошим
- [[refactoring|Refactoring]] — как улучшать
- [[static-analysis|Static Analysis]] — automate style review
- [[code-quality|Code Quality]] — quality gates
- [[03_middle-to-senior|Middle → Senior]] — review skill для роста

## Reading list

- **Google's Code Review Guidelines** — google.github.io/eng-practices/review
- **Microsoft Code Review Best Practices** — learn.microsoft.com (search "code review")
- **The Art of Readable Code** — Boswell & Foucher
- **Clean Code** — Robert C. Martin (review-friendly code)
- **Atlassian Code Review Guide** — atlassian.com/agile/software-development/code-reviews
- **Best Kept Secrets of Peer Code Review** — Smartbear
