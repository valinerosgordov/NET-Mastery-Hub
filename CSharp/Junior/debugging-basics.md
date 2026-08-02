---
tags: [csharp, debugging, junior, visual-studio, breakpoints, debugger, tools, diagnostics]
level: Junior
date: 2026-08-02
---

# Отладка — debugging для Junior

> **Как находить баги в .NET-коде быстро.** Breakpoints (все виды), stepping, inspect state, call stack, exception debugging, symbols, async debugging, production-инструменты dotnet-counters / trace / dump / gcdump. Закрывает пробел: «знаю, что есть отладчик, но реально использую только `Console.WriteLine`».

---

## 0. Как читать этот файл

Если ты впервые открыл debugger — читай разделы 1→6 подряд: получишь рабочий навык breakpoint → step → inspect → fix. Если уже отлаживаешь, но непонятно «как это работает изнутри» — раздел 8 (symbols), 15 (async debugging). Если интересует production — раздел 9-13 (dotnet-counters, trace, dump, gcdump), 16 (production debugging).

Все примеры рабочие — копируешь в проект, ставишь breakpoint, запускаешь. Где «`F5`» — это горячая клавиша Visual Studio (для VS Code / Rider — таблицы соответствия в разделе 21). Cross-language якоря (`> [!info]-`) свёрнуты — раскрывай, если переходишь из C++/Python/JS/Go/Java. Interview-вопросы (`> [!question]-`) встроены рядом с теорией.

---

## 1. Что это, зачем и когда

### 1.1. Что такое debugger

**Debugger** — это инструмент, который умеет:

1. **Останавливать выполнение** программы в произвольной точке (breakpoint).
2. **Читать состояние** — переменные, поля, локальные, this, call stack, threads.
3. **Менять состояние** — присвоить переменной новое значение, выполнить произвольное выражение в контексте остановленного кода.
4. **Управлять потоком** — пошагово выполнять код, перепрыгнуть в другую строку, выйти из функции.
5. **Реагировать на события** — exception throw, начало execution на другом потоке, изменение значения поля.

Под капотом debugger общается с **debugger API** (на Windows — Win32 Debug API, на Linux — `ptrace`, на .NET — ICorDebug / Managed Debugging API). Это позволяет внешнему процессу полностью контролировать другой процесс.

В .NET отладчик понимает managed code: знает структуру объектов, GC, threading model, async state machines. Это даёт несравненно больше, чем просто «прервать процесс и прочитать память».

### 1.2. Зачем уметь, когда есть `Console.WriteLine`

Самый частый паттерн junior'а на bug: расставить `Console.WriteLine` везде, запустить, посмотреть. Это работает, но **в разы медленнее**, чем debugger:

```
Print debugging                        Debugger
───────────────                        ────────
1. Добавил Console.WriteLine           1. F9 — breakpoint
2. Запустил, не туда — изменил         2. F5 — запуск
3. Запустил снова                      3. Hover на переменной — видишь значение
4. Не помогло — добавил ещё            4. F10 — следующая строка
5. Долбился час                        5. Bug ясен через минуту
6. Forgot remove перед commit
```

Проблемы print debugging:

1. **Засоряет код.** Каждое исследование оставляет следы, которые надо чистить.
2. **Каждое изменение — рестарт.** `Console.WriteLine` в новое место → перекомпилировать → перезапустить.
3. **Не видишь complex objects.** `Console.WriteLine(user)` напечатает имя класса, надо писать `JsonSerializer.Serialize(user)`.
4. **Легко забыть.** В production-коде встречаются забытые «debug log», которые засоряют логи или раскрывают данные.
5. **Не покрывает hot path.** В горячем цикле `Console.WriteLine` тормозит код в 100 раз — и реальное поведение не воспроизводится.

Debugger обходит всё это: state видно без модификации кода, можно исследовать, не запуская заново, complex objects expand-ятся в дереве.

### 1.3. Mental model — что debugger делает с программой

Когда ты ставишь breakpoint:

1. Debugger пишет в memory процесса специальную инструкцию (на x86 — `int 3`, byte `0xCC`) на месте оригинальной инструкции.
2. CPU при выполнении этого byte бросает software interrupt.
3. ОС перехватывает interrupt и передаёт управление debugger'у.
4. Debugger восстанавливает оригинальную инструкцию, читает state процесса, показывает тебе.
5. Когда ты жмёшь Continue — debugger выполняет одну инструкцию (single-step), потом возвращает breakpoint обратно на свое место и продолжает.

В managed коде это сложнее: JIT-компилятор должен сначала скомпилировать метод, прежде чем туда поставить breakpoint. Поэтому в .NET debugger ставит **breakpoint на метаданные**, и при JIT-компиляции метода JIT добавит точки прерывания согласно этим метаданным.

**Зачем знать:** если breakpoint «не срабатывает» — это не магия, а конкретный механизм. Чаще всего: метод не вызывается (проверь call stack), символы не загружены (раздел 8), оптимизация compiler-а выкинула строку (Release build), или код на другом потоке (раздел 15).

### 1.4. Эволюция отладочных инструментов в .NET

| Инструмент | Появился | Что |
|------------|----------|-----|
| **Visual Studio Debugger** | .NET Framework 1.0 (2002) | Полный managed debugger, IDE-интегрированный |
| **WinDbg + SOS** | .NET Framework 1.0 | Low-level отладка через crash dumps |
| **Edit and Continue** | .NET Framework 2.0 | Изменить код во время отладки без рестарта |
| **DebuggerDisplay attribute** | .NET Framework 2.0 | Кастомный display для своих типов |
| **VS Code Debugger** | 2015 | Cross-platform IDE с debugger |
| **dotnet-counters** | .NET Core 3.0 (2019) | Live мониторинг metrics без attach |
| **dotnet-trace** | .NET Core 3.0 | Sampling profiler через CLI |
| **dotnet-dump** | .NET Core 3.1 | Сбор memory dump на любой ОС |
| **dotnet-gcdump** | .NET Core 3.1 | Snapshot GC heap для анализа leak'ов |
| **dotnet-monitor** | .NET 6 (2021) | Sidecar для Kubernetes — REST API над diagnostics |
| **Async stack traces улучшения** | .NET 6 | Continuation chain в call stack как у sync |
| **Hot Reload** | .NET 6 | `dotnet watch run` применяет изменения без рестарта |

С .NET 6+ диагностические инструменты вышли из IDE наружу — стали обычными CLI-tools, которые работают одинаково на Windows / Linux / macOS, в Docker, в Kubernetes. Это ключ к production-debugging.

### 1.5. Когда что использовать

| Сценарий | Инструмент |
|----------|-----------|
| Бизнес-логика баг локально | IDE debugger (breakpoint + step) |
| Сложно воспроизвести bug | Conditional breakpoint, logpoint |
| Хочу пройти через 1000 итераций | Hit count breakpoint |
| Async deadlock | Tasks window, Parallel Stacks |
| Performance — slow method | Stopwatch logging, потом `dotnet-trace` |
| Memory leak | Diagnostic Tools snapshots или `dotnet-gcdump` |
| Production crash | Structured logging + correlation ID + APM |
| Production hang | `dotnet-dump` collect, `dotnet-dump analyze` |
| Production живой мониторинг | `dotnet-counters monitor`, OpenTelemetry metrics |
| Production memory rost | `dotnet-monitor` или sidecar с auto-collect |

> [!info]- Если ты знаешь C/C++ / Python / JavaScript / Go / Java
> **C/C++:** Visual Studio debugger ↔ MSVC debugger (один и тот же), gdb / lldb для остальных. .NET debugger умеет много того, чего gdb не умеет (managed structures, async traces).
>
> **Python:** `pdb`, `breakpoint()` в коде, IDE debuggers (PyCharm, VS Code). .NET-аналог — IDE breakpoint + Immediate Window вместо `pdb` REPL.
>
> **JavaScript:** `node --inspect`, Chrome DevTools, VS Code. По возможностям VS / VS Code .NET debugger сравним с Chrome DevTools для frontend (breakpoints, step, watch, source maps ↔ symbols + Source Link).
>
> **Go:** `delve` (`dlv`). .NET имеет схожие capabilities, но GUI-инструменты VS / Rider значительно мощнее. Из CLI — `dotnet-trace` ↔ `pprof`, `dotnet-dump` ↔ `dlv core`.
>
> **Java:** `jdb`, IDE (IntelliJ IDEA, Eclipse). Visual Studio / Rider — primary IDEs для .NET, схожи по функциональности с IntelliJ. JFR (Java Flight Recorder) ↔ EventPipe / `dotnet-trace`. JMC (Mission Control) ↔ PerfView / dotMemory.

> [!question]- Интервью: чем debugger лучше print-debugging?
> Скоростью и информативностью. Debugger останавливает программу без изменения кода, показывает состояние всех переменных, дерево объектов, call stack, threads. Можно вычислять выражения в текущем контексте (Immediate Window). Один debug-сеанс часто заменяет 10 итераций «добавил Console.WriteLine — перекомпилил — запустил». Print-debugging остаётся полезным в production (через ILogger), там debugger недоступен. Также для воспроизводимых тестов — assertion в коде надёжнее, чем breakpoint, который надо ставить руками.

---

## 2. Где отлаживают — окружения

### 2.1. Visual Studio (Windows)

Самый мощный .NET debugger. Актуальная версия — **Visual Studio 2026** (GA ноябрь 2025, вышла вместе с .NET 10); Visual Studio 2022 — предыдущая, всё описанное ниже работает в обеих. Включает:

- Все типы breakpoints (line, conditional, hit count, function, logpoint, dependent).
- Visual debugger UI — Locals, Watch, Autos, Call Stack, Threads, Tasks, Modules.
- IntelliTrace (Enterprise) — historical debugging «как далеко вперёд можно перемотать».
- Edit and Continue.
- Diagnostic Tools — встроенный CPU и memory profiler.
- Memory snapshots с comparison.
- Time Travel Debugging (через WinDbg в TTD-режиме).

Установка через [Visual Studio Installer](https://visualstudio.microsoft.com/), workload «.NET desktop development» или «ASP.NET and web development».

### 2.2. JetBrains Rider (Windows / Linux / macOS)

Альтернатива VS, особенно популярна на macOS / Linux. По возможностям близка к VS:

- Все типы breakpoints.
- Debugger UI с фильтрами и поиском.
- dotMemory + dotTrace интеграция (отдельные продукты, но интегрируются).
- Hot Reload.
- Поддержка Unity, Unreal, Avalonia.

Платный (есть бесплатная версия для open-source).

### 2.3. VS Code (Windows / Linux / macOS)

Лёгкий, бесплатный, кросс-платформенный. Для .NET — расширения:

- **C# Dev Kit** (Microsoft) — основное расширение, включает C# language server.
- **C#** (Microsoft) — language support.
- **.NET Install Tool** — для управления SDK.

Возможности:

- Все основные breakpoints (line, conditional, hit count, logpoint).
- Watch, Locals, Call Stack.
- Step Over / Into / Out.
- Debug Console (аналог Immediate Window).
- Multi-target debugging.

Что меньше, чем в VS / Rider: visual UI для дампов памяти, Tasks window для async, Edit and Continue (есть Hot Reload через `dotnet watch`).

`launch.json` для конфигурации:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": ".NET Core Launch",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "build",
            "program": "${workspaceFolder}/bin/Debug/net10.0/MyApp.dll",
            "args": [],
            "cwd": "${workspaceFolder}",
            "console": "internalConsole",
            "stopAtEntry": false
        }
    ]
}
```

### 2.4. CLI-инструменты — отладка без IDE

Для production / Docker / serverless / SSH-сессий, где IDE неприменима:

- **`dotnet-counters`** — live метрики (CPU, GC, requests/sec, custom Meter).
- **`dotnet-trace`** — sampling profiler (CPU, allocations, contention).
- **`dotnet-dump`** — сбор и анализ полного дампа памяти.
- **`dotnet-gcdump`** — лёгкий dump только GC heap (для leak detection).
- **`dotnet-stack`** — current call stacks всех потоков (на лету).
- **`dotnet-monitor`** — REST API над всем выше, как sidecar в Kubernetes.

Установка как .NET tool (раздел 9-13).

### 2.5. Attach vs Launch

Есть два режима:

- **Launch** — debugger сам запускает процесс. Debug build, breakpoints до старта, full control.
- **Attach** — debugger цепляется к уже работающему процессу. Production-like, можно подключиться к Docker-контейнеру / удалённой машине.

Visual Studio: **Debug → Attach to Process** (Ctrl+Alt+P). VS Code: configuration `"request": "attach"` в `launch.json`. Из CLI: `dotnet-trace collect --process-id 12345`.

В production предпочтительно attach к работающему процессу — не надо рестартовать. Для прикладных багов — обычно launch.

### 2.6. Remote debugging

Можно отладить процесс на другой машине / в Docker:

**Docker:**
```bash
# В контейнере должен быть установлен vsdbg
docker exec myapi-container apt-get install -y curl unzip
docker exec myapi-container curl -sSL https://aka.ms/getvsdbgsh | bash /dev/stdin -v latest -l /vsdbg
```

В VS Code (`launch.json`):
```json
{
    "name": "Attach to Docker",
    "type": "coreclr",
    "request": "attach",
    "processId": "${command:pickRemoteProcess}",
    "pipeTransport": {
        "pipeProgram": "docker",
        "pipeArgs": ["exec", "-i", "myapi-container"],
        "debuggerPath": "/vsdbg/vsdbg",
        "pipeCwd": "${workspaceFolder}",
        "quoteArgs": false
    }
}
```

**Remote machine (SSH):**
В VS Code — `Remote-SSH` extension, потом обычная отладка.

> [!question]- Интервью: что значит «attach to process» при отладке?
> Debugger подключается к уже запущенному процессу вместо того, чтобы запускать новый. Используется в production-like сценариях (без рестарта приложения), при отладке Docker-контейнеров, при анализе долгих процессов (cron, background services). Под капотом: ОС позволяет процессу A читать/менять память процесса B, если у A соответствующие привилегии. Visual Studio через Win32 Debug API (или ICorDebug для managed), Linux — через `ptrace`. Detach потом не убивает процесс — только отключает контроль.

---

## 3. Breakpoints — все виды

### 3.1. Line breakpoint — простой

Самый частый. Кликаешь слева от номера строки (или жмёшь `F9`), появляется красный кружок:

```csharp
public int Calculate(int a, int b)
{
    int sum = a + b;        // ← красный кружок здесь
    int product = a * b;
    return sum + product;
}
```

Когда выполнение доходит до этой строки — debugger останавливается **до выполнения**. Ты видишь значения `a` и `b`, но `sum` ещё не вычислено (покажется `0` или старое значение).

### 3.2. Conditional breakpoint — срабатывает по условию

В hot loop из миллиона итераций обычный breakpoint бесполезен — нажмёшь `F5` миллион раз. Conditional breakpoint срабатывает только если условие true:

Right-click на breakpoint → **Conditions...** → Conditional Expression → `i == 999_999`.

```csharp
for (int i = 0; i < 1_000_000; i++)
{
    Process(i);   // ← conditional: i == 999_999
}
```

Условие — любое C#-выражение, которое можно вычислить в текущем контексте. Можно использовать локальные переменные, поля, методы (но осторожно — методы вызываются на каждой итерации, тормозит выполнение).

Полезные паттерны:

```csharp
// Bug в конкретном пользователе
user.Id == 42

// Bug на определённом item
order.Items.Count > 100

// String matching
customer.Email.Contains("@company.com")

// Combined
order.Status == OrderStatus.Pending && order.Total > 1000m
```

### 3.3. Hit count breakpoint — срабатывает на N-ом проходе

Right-click breakpoint → **Conditions** → Hit Count.

Варианты:
- **Equal to** — сработает только на N-ом hit. Полезно: «bug в 100-ой итерации».
- **Multiple of** — каждый N-ый. Полезно: «хочу видеть прогресс каждые 1000 итераций».
- **Greater than or equal to** — после N-ого. Полезно: «первые 50 итераций OK, дальше странно».

### 3.4. Function breakpoint — на сигнатуру

Не на конкретной строке, а на любом вызове метода.

VS: **Debug → New Breakpoint → Function Breakpoint** (Ctrl+K, B). Вводишь имя:

```
MyApp.UserService.GetById
```

Срабатывает на входе в любой `UserService.GetById`, независимо от перегрузки. Полезно когда:
- Не знаешь, в каком файле метод (большой код-база).
- Хочешь поймать ВСЕ вызовы метода.
- Hot reload / реструктуризация — конкретная строка часто меняется.

### 3.5. Exception breakpoint — на throw

VS: **Debug → Windows → Exception Settings** (Ctrl+Alt+E). Чекбокс на типе exception → debugger остановится в момент `throw`, до catch.

```
[ ] Common Language Runtime Exceptions
    [✓] System.NullReferenceException
    [ ] System.InvalidOperationException
    ☐ System.IO.FileNotFoundException
[ ] JavaScript Runtime Exceptions
[ ] etc.
```

Зачем: если в коде `try { ... } catch { swallow }` глотает ошибку — без exception breakpoint ты её никогда не увидишь. С чекбоксом debugger остановится **в момент throw**, до того, как catch её поймает.

Сценарий типичный: метод иногда возвращает `null` или default, ты не знаешь почему. Включаешь break on `Exception` → debugger тебе показывает, что внутри происходит throw + swallow.

Также можно добавить **Conditions для exception**: например, останавливаться на `NullReferenceException` только если `e.StackTrace.Contains("MyService")`.

### 3.6. Logpoint (tracepoint) — без кода печатает в Output

Right-click breakpoint → **Actions** → ставишь Log Message и не отмечаешь «Continue execution» (по умолчанию ставится — это значит «не останавливаться, только напечатать»).

```
{user.Id}: {user.Name} processed in {duration}ms
```

Это **аналог `Console.WriteLine`, но без модификации кода**. Запустил, посмотрел Output Window, удалил breakpoint — никаких следов в коде.

Незаменимо для:
- Отладки production-like сценариев, где не хочешь модифицировать deployable код.
- Hot path — добавить тысячу `Console.WriteLine` тяжело, logpoint debugger сам управляет.
- Quick check «когда вообще этот метод вызывается».

В VS Code logpoint: правый клик breakpoint → **Add Logpoint**.

### 3.7. Dependent breakpoint (Visual Studio 2022+)

Срабатывает, **только если другой breakpoint уже сработал**. Полезно когда bug — это последовательность событий:

```csharp
public void ProcessOrder(Order order)
{
    Validate(order);                  // breakpoint A
    var enriched = Enrich(order);     // breakpoint B — dependent on A
    Save(enriched);                   // breakpoint C — dependent on B
}
```

`B` сработает только после прохождения `A`. Если `A` не пройден (например, bug убил выполнение раньше) — `B` пропускается.

### 3.8. Temporary breakpoint — одноразовый

Устанавливается на одно срабатывание. После hit удаляется автоматически.

VS: правый клик breakpoint → **Make Temporary**, или Ctrl+F8 на строке без breakpoint.

Полезно для «дойти до этой строки и забыть про breakpoint».

### 3.9. Run to Cursor — взамен временного breakpoint

`Ctrl+F10` (VS, VS Code, Rider) — запускает выполнение до строки, на которой курсор. Без явного breakpoint.

Это удобнее, чем поставить, дойти, удалить. Используй когда нужно «прыгнуть на эту строку» из любого места.

### 3.10. Breakpoint groups и labels (VS 2019+)

Если у тебя 50 breakpoints, можно их группировать по labels:

- Right-click breakpoint → **Edit labels** → "feature-X", "perf-investigation"
- В Breakpoints window (Ctrl+D, B) — фильтр по label

Энейблить/дисейблить группой удобно при переключении между задачами.

> [!question]- Интервью: чем отличается conditional breakpoint от exception breakpoint?
> Conditional breakpoint — срабатывает на конкретной строке кода с условием в логике (значение переменной, состояние объекта). Exception breakpoint — срабатывает на throw exception определённого типа, **независимо от того, где** в коде throw случился. Conditional ловит «в этом методе при этих данных», exception — «всякий раз, когда летит NRE» по всему процессу. Conditional требует знать точку, exception — знать тип ошибки. Часто используются вместе: вкл exception breakpoint на NRE, потом дальше conditional на конкретный case.

---

## 4. Stepping — пошаговое выполнение

### 4.1. F5 — Continue

Запускает / продолжает выполнение до следующего breakpoint или конца программы. Самая частая команда.

### 4.2. F10 — Step Over

Выполняет текущую строку и останавливается на следующей. **Если строка — вызов метода**, метод выполнится целиком, debugger не зайдёт внутрь.

```csharp
var data = LoadData();   // ← F10 здесь: LoadData() выполнится, остановимся на след. строке
var result = Process(data);
```

Используй F10, когда уверен, что метод работает правильно. 90% действий debugger — F10.

### 4.3. F11 — Step Into

Заходит **внутрь** метода. Debugger остановится на первой строке `LoadData()`.

```csharp
var data = LoadData();   // ← F11: останавливаемся внутри LoadData
```

Используй F11, когда подозреваешь bug внутри вызываемого метода. Если метод сложный — потом Shift+F11 чтобы выйти.

### 4.4. Shift+F11 — Step Out

Выполняет остаток текущего метода и останавливается на следующей строке вызывающего. Полезно когда:
- Случайно зашёл F11 в библиотечный метод (System.IO.File.ReadAllText) — Shift+F11 выходит обратно.
- Уже всё проверил внутри метода, хочешь дальше — не надо F10 до конца.

### 4.5. Step Into Specific (Ctrl+Shift+F11)

Когда строка содержит несколько вызовов:

```csharp
var result = Calculate(LoadX(), LoadY());
```

F11 на этой строке зайдёт в `LoadX()` (первый аргумент). А если хочешь сразу в `Calculate()`? Right-click на строке → **Step Into Specific** → выбираешь метод.

### 4.6. Set Next Statement — прыжок без выполнения

Иногда нужно «перепрыгнуть» через код, не выполняя его. Перетащи жёлтую стрелку слева от строки (или правый клик → Set Next Statement).

Применение:
- Пропустить `throw new Exception()` без рестарта debugger'а.
- Перевыполнить блок кода (drag вверх, потом F5).
- Тестировать «а что если бы выполнение пошло по другой ветке».

⚠️ **Осторожно:** Set Next Statement может оставить программу в неконсистентном состоянии (поля не инициализированы, locks не взяты). Для пробных запусков OK, для production-fix — нет.

### 4.7. Run to Cursor (Ctrl+F10)

Курсор на строке → Ctrl+F10 → debugger запускается до неё (как одноразовый breakpoint).

Удобно для «прыгнуть мимо безынтересного кода до подозрительной точки».

### 4.8. Continue Execution to specific scope

VS 2022+: **Force Run To Cursor** — пропускает все breakpoints на пути. Полезно когда между текущей точкой и целью много отвлекающих breakpoint'ов.

### 4.9. Step Over Properties / Operators

Visual Studio по умолчанию **не заходит** F11 в:
- Auto-property getters (`int X => _x;`).
- Trivial operators.

Это настраивается: **Tools → Options → Debugging → General → Step over properties and operators**.

В Rider — то же самое в Settings → Build, Execution, Deployment → Debugger → Stepping.

### 4.10. Just My Code

Опция, которая фильтрует non-user code (BCL, NuGet packages) из stepping и call stack.

VS: **Tools → Options → Debugging → General → Enable Just My Code**.

С Just My Code включённым:
- F11 в `int.Parse(s)` пропускает внутренности .NET и сразу возвращается.
- Call stack показывает только твой код, библиотеки помечены как `[External Code]`.

Удобно для прикладной разработки. Выключай, если отлаживаешь сам себе библиотеку или хочешь шагать в `System.Linq.Enumerable`.

> [!question]- Интервью: чем F10 отличается от F11?
> F10 (Step Over) — выполняет текущую строку целиком, не заходя внутрь вызовов методов. F11 (Step Into) — заходит внутрь вызова метода и останавливается на первой строке. F10 для шага через строки, когда уверен в внутреннем коде. F11 для исследования, что метод делает. Если случайно зашёл F11 в неинтересный метод — Shift+F11 (Step Out) выходит обратно к caller. На строке без вызовов F10 и F11 эквивалентны.

---

## 5. Inspecting state — окна и DataTip

### 5.1. DataTip — hover на переменной

Самый быстрый способ увидеть значение. Наводишь курсор на переменную:

```csharp
var user = LoadUser(42);   // ← после выполнения hover на user
//          ↑ показывает: { Id = 42, Name = "Alice", Email = "a@x.com" }
```

В DataTip можно:
- **Раскрыть объект** — стрелка слева, видишь поля.
- **Зафиксировать (pin)** — DataTip остаётся на экране, даже если уберёшь курсор.
- **Скопировать значение** — для вставки в другое место.
- **Edit Value** — изменить значение прямо в DataTip.

Для сложных объектов — это часто всё, что нужно.

### 5.2. Locals — все локальные переменные

VS: **Debug → Windows → Locals** (Alt+4).

Показывает все переменные текущего метода + `this`. Автоматически обновляется при step. Можно раскрывать объекты, edit-ить значения.

### 5.3. Watch — список интересующих выражений

VS: **Debug → Windows → Watch** (Ctrl+Alt+W, 1-4 — четыре раздельных окна).

Добавляешь произвольные expressions. Они вычисляются в текущем контексте на каждом step:

```
arr.Length            → 5
arr[i]                → 4
arr.Sum()             → 15
arr.Where(x => x > 2).ToArray()    → {3, 4, 5}
DateTime.Now.Hour     → 14
nameof(myVar)          → "myVar"
```

Watch окно полезно для:
- Сложные LINQ-выражения, которые проверять руками сложно.
- Computed properties, которые ты хочешь видеть постоянно.
- Зеркала state нескольких объектов.

### 5.4. Autos — авто-выбранные переменные

Visual Studio автоматически показывает переменные, использованные в текущей или предыдущей строке. Удобно — не надо добавлять руками.

### 5.5. Immediate Window — REPL для текущего scope

VS: **Debug → Windows → Immediate** (Ctrl+Alt+I).

Это полноценный REPL: выполняешь любой C#-код в контексте остановленного потока:

```
> arr.Sum()
15

> arr.Where(x => x > 2).ToList()
{System.Collections.Generic.List<int>}
[0]: 3
[1]: 4
[2]: 5

> n = 100   // изменить переменную!
100

> ResetCounter()   // вызвать метод
```

Можно даже создавать новые переменные:

```
> var newList = new List<int>(arr); newList.Add(99); newList
{0: 1, 1: 2, 2: 3, 3: 4, 4: 5, 5: 99}
```

Незаменимо для:
- Quick check гипотезы без редактирования кода.
- Тестирования edge cases — поменять значение, посмотреть, что выйдет.
- Вызова метода с конкретными параметрами.

### 5.6. QuickWatch — расширенный hover

VS: правый клик на переменной → Quick Watch (Shift+F9).

Показывает значение в большом modal window. Удобно для глубоких объектов — больше места для дерева.

### 5.7. DebuggerDisplay — кастомный display

Свои типы по умолчанию показываются как `{Namespace.Type}`. Чтобы видеть осмысленную строку — атрибут `[DebuggerDisplay]`:

```csharp
[DebuggerDisplay("User #{Id}: {Name} ({Email})")]
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
}

// В DataTip / Locals: User #42: Alice (a@x.com)
```

Внутри строки `{...}` — любое выражение C#:

```csharp
[DebuggerDisplay("Order {Id} ({Items.Count} items, total = {Total:C})")]
public class Order
{
    public int Id { get; set; }
    public List<OrderItem> Items { get; set; } = [];
    public decimal Total => Items.Sum(i => i.Price * i.Quantity);
}
```

Огромная экономия времени, когда часто отлаживаешь свои типы.

### 5.8. DebuggerBrowsable — скрыть детали

Иногда у класса много internal-полей, которые засоряют debugger view:

```csharp
public class Cache
{
    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    private readonly object _syncRoot = new();

    [DebuggerBrowsable(DebuggerBrowsableState.RootHidden)]
    private readonly Dictionary<string, object> _items = new();

    public IReadOnlyDictionary<string, object> Items => _items;
}
```

`Never` — поле не показывается. `RootHidden` — поле раскрывается inline (не как `_items → {...}`, а его содержимое сразу).

### 5.9. DebuggerTypeProxy — кастомное представление

Для совсем сложных типов (например, дерева с custom navigation) — можно реализовать proxy:

```csharp
public class TreeNode<T>
{
    public T Value { get; set; }
    public List<TreeNode<T>> Children { get; set; } = [];
}

[DebuggerTypeProxy(typeof(TreeNodeDebugView<>))]
public class TreeNode<T>
{
    // ...
}

internal class TreeNodeDebugView<T>
{
    private readonly TreeNode<T> _node;
    public TreeNodeDebugView(TreeNode<T> node) => _node = node;

    [DebuggerBrowsable(DebuggerBrowsableState.RootHidden)]
    public T[] DescendantValues
    {
        get
        {
            var result = new List<T>();
            void Visit(TreeNode<T> n)
            {
                result.Add(n.Value);
                foreach (var c in n.Children) Visit(c);
            }
            Visit(_node);
            return result.ToArray();
        }
    }
}
```

В debugger вместо рекурсивного дерева увидишь плоский массив всех values. Используется во многих BCL-типах (List, Dictionary).

> [!question]- Интервью: что делает `[DebuggerDisplay]` атрибут?
> Кастомизирует, как объект отображается в DataTip / Locals / Watch debugger'а. По умолчанию debugger показывает `{Namespace.TypeName}`. С `[DebuggerDisplay("text {Property}")]` ты задаёшь шаблон с подстановками выражений в фигурных скобках. Атрибут не влияет на runtime-поведение, только на debugger view. Полезно для своих доменных типов (User, Order, Money), чтобы быстро видеть осмысленные данные. Аналог `__repr__` в Python, `toString()` в Java для отладочных целей.

---

## 6. Call Stack — откуда я попал сюда

### 6.1. Что такое call stack

**Call stack** — список фреймов, которые сейчас выполняются. Каждый фрейм = вызов метода. Самый верхний — текущий метод, ниже — кто его позвал, кто позвал того и так далее.

```
Calculate(int x)              ← текущий, breakpoint здесь
ProcessOrder(Order order)
HandleRequest(HttpContext ctx)
ApiController.Post(Request r)
Microsoft.AspNetCore.Routing
[External Code]                ← скрытые библиотечные фреймы (Just My Code)
```

VS: **Debug → Windows → Call Stack** (Ctrl+Alt+C).

### 6.2. Навигация по фреймам

Двойной клик на любой фрейм — debugger переключается в его контекст:
- Locals и Watch покажут переменные **этого** фрейма.
- Файл откроется на нужной строке.
- Можно осмотреть значения, переданные на каждом уровне.

Это критично для понимания «откуда взялись эти данные». Bug часто в том, что верхний caller передал неправильное значение, а текущий метод сам по себе работает корректно.

### 6.3. External code — что показывать

По умолчанию (с Just My Code) внешние библиотеки скрыты:

```
[External Code]
```

Это маскирует фреймы из `System.*`, `Microsoft.*`, NuGet-пакетов. Нужно увидеть — Right-click → **Show External Code**.

### 6.4. Async stack traces (.NET 6+)

Раньше отладка async-кода была болью: stack trace обрывался на `await`, не показывал continuation chain.

```csharp
// До .NET 6
async Task DoWorkAsync()
{
    await Task.Delay(100);
    Throw();   // ← exception здесь
}

// Stack trace:
//   at Throw()
//   at DoWorkAsync() в строке continuation
//   [нет инфы кто его вызвал!]
```

С .NET 6+ JIT записывает в metadata, кто запустил Task. Stack trace выглядит почти как sync:

```
   at Throw()
   at DoWorkAsync()
   at Program.Main()
   at Program.<Main>$()
```

Также в Visual Studio есть **Tasks window** (Debug → Windows → Tasks), который показывает все активные Task с их состоянием и stack-ами.

### 6.5. Parallel Stacks — для multi-threaded

VS: **Debug → Windows → Parallel Stacks** (Ctrl+Shift+D, S).

Визуализация call stacks **всех потоков** в одном дереве. Видно, где потоки пересекаются (например, ждут одного lock'а — потенциальный deadlock).

Полезно для:
- Race conditions.
- Deadlock investigation.
- Понимания «куда забрёл каждый из 30 thread pool потоков».

### 6.6. Stack trace в exception messages

Когда exception доходит до вершины и приложение крашится, ты видишь:

```
System.NullReferenceException: Object reference not set to an instance of an object.
   at MyApp.UserService.FormatName(User user) in /src/UserService.cs:line 25
   at MyApp.UserController.Get(Int32 id) in /src/UserController.cs:line 14
   at lambda_method...
   at Microsoft.AspNetCore.Mvc.Infrastructure.ActionMethodExecutor...
```

Это тот же call stack, но в текстовом виде. Читается **снизу вверх**: «контроллер вызвал сервис, сервис упал в FormatName». Для production отладки stack trace — единственный источник информации (debugger недоступен).

### 6.7. StackTrace API

В коде можно получить call stack программно:

```csharp
using System.Diagnostics;

void LogContext()
{
    var stack = new StackTrace(skipFrames: 1, fNeedFileInfo: true);
    foreach (var frame in stack.GetFrames() ?? [])
    {
        var m = frame.GetMethod();
        Console.WriteLine($"  at {m?.DeclaringType?.FullName}.{m?.Name} in {frame.GetFileName()}:line {frame.GetFileLineNumber()}");
    }
}
```

Используется в логгерах и инструментах диагностики.

### 6.8. CallerMemberName / CallerFilePath / CallerLineNumber

Compiler-attribute, которые подставляют caller info без рантайм-overhead:

```csharp
public void Log(
    string message,
    [CallerMemberName] string caller = "",
    [CallerFilePath] string file = "",
    [CallerLineNumber] int line = 0)
{
    Console.WriteLine($"[{file}:{line} in {caller}] {message}");
}

// Использование
public void DoWork()
{
    Log("Started");
    // [/path/Program.cs:25 in DoWork] Started
}
```

Compiler в момент компиляции подставляет константы. Ноль рантайм-затрат, в отличие от `StackTrace`.

> [!question]- Интервью: чем async stack trace отличается от sync в .NET 6+?
> До .NET 6 async stack trace обрывался на `await` — нельзя было увидеть, кто запустил `Task`. С .NET 6 JIT записывает «логического caller» в metadata Task, и debugger / `Exception.StackTrace` показывает полную цепочку как для sync-кода. Это решило одну из главных болей async-debugging. Visual Studio (2022 и новее, актуальная — VS 2026) + .NET 6+ отображает async stack как обычный, без специальных Tasks window (хотя Tasks window остаётся полезным для visualization активных задач).

---

## 7. Exception debugging

### 7.1. First-chance vs unhandled

Exception в .NET имеет два момента:

- **First-chance** — exception только что throw, ещё не обработан. Если есть catch — будет пойман, программа продолжит. Если нет — пойдёт дальше вверх по стеку.
- **Unhandled** — exception дошёл до вершины стека без обработки. Программа крашится (или handler `AppDomain.UnhandledException` срабатывает).

По умолчанию debugger останавливается **только на unhandled** — пропускает first-chance, если есть catch. Это удобно: каждое try/catch не сразу разбивает поток отладки.

Проблема: если catch блок «глотает» exception (`catch { }` без логирования) — ты не узнаешь, что внутри был throw. Решение — exception breakpoint (раздел 3.5).

### 7.2. Включение break on first-chance

VS: **Debug → Windows → Exception Settings** (Ctrl+Alt+E):

```
[ ] Common Language Runtime Exceptions
    [✓] System.NullReferenceException     ← теперь break и на first-chance
    [✓] System.InvalidOperationException
    ...
```

Чекбокс — break on **throw**, до того как catch его поймает.

### 7.3. Conditional exception breakpoints

Для шумных exception'ов (например, `OperationCanceledException` — может летать сотни раз в секунду в правильно работающем приложении) можно поставить condition:

Right-click на exception в Settings → Conditions → Module name not equal `Microsoft.AspNetCore.dll`.

Сработает только если exception летит **не из ASP.NET Core middleware**, а из твоего кода.

### 7.4. AppDomain.UnhandledException — last resort

В коде:

```csharp
AppDomain.CurrentDomain.UnhandledException += (sender, args) =>
{
    var ex = args.ExceptionObject as Exception;
    Console.Error.WriteLine($"FATAL: {ex}");
    // Запиши в файл / лог / отправь в Sentry
};

TaskScheduler.UnobservedTaskException += (sender, args) =>
{
    args.SetObserved();
    Console.Error.WriteLine($"UnobservedTaskException: {args.Exception}");
};
```

Это последний шанс залогировать, что приложение упало. В production обычно дополняется через ASP.NET Core middleware (UseExceptionHandler).

### 7.5. Debug.Assert и Trace.Assert

```csharp
Debug.Assert(user != null, "user must not be null");
Debug.Assert(items.Count > 0, "items must not be empty");
```

Если condition false → debugger останавливается с сообщением. В **Release** build весь `Debug.*` код вырезается — ноль overhead.

`Trace.Assert` — то же самое, но не вырезается в Release. Используется реже, для production-сценариев с явным контрактом.

Полезно для invariants, которые точно должны выполняться. Если сработал — значит ты столкнулся с состоянием, которое не должно было возникнуть.

### 7.6. Throw vs throw

```csharp
catch (Exception ex)
{
    LogIt(ex);
    throw;        // ✅ сохраняет stack trace
}

catch (Exception ex)
{
    LogIt(ex);
    throw ex;     // ❌ перезаписывает stack trace! Original origin теряется
}
```

`throw;` без аргумента — rethrow с сохранённым stack. `throw ex;` обнуляет stack до текущей точки. Для отладки это огромная разница: с `throw ex;` ты не увидишь, **где originally throw был**.

### 7.7. Inner exception

```csharp
try
{
    DoWork();
}
catch (Exception inner)
{
    throw new ServiceException("Failed to do work", inner);
}
```

Wrapping — стандартная практика для абстрагирования. Original exception доступен через `ex.InnerException`. Stack trace показывает обе цепочки:

```
ServiceException: Failed to do work
 ---> InvalidOperationException: Original error
   at DoWork()
   at MyService.Run()
   --- End of inner exception ---
   at MyService.Run()
   at Caller.Process()
```

В debugger Locals покажут `ex.InnerException` — раскрываешь, видишь оригинал.

> [!question]- Интервью: чем `throw;` отличается от `throw ex;`?
> `throw;` — rethrow exception, сохраняя оригинальный stack trace. Используется в catch без модификации. `throw ex;` — re-raise exception с переписанным stack trace до текущей точки, оригинальное место throw теряется. Для отладки `throw;` всегда предпочтительнее. `throw new SomeException("...", ex)` — оборачивание с сохранением original в InnerException, тоже OK.

---

## 8. Symbols и Source Link

### 8.1. Что такое symbols (PDB)

PDB (Program Database) — файл с debug-метаданными для DLL/EXE:
- Имена методов, параметров, локальных переменных.
- Соответствие IL → исходные файлы и строки.
- Информация для async stack traces.
- Source Link URLs (если включён).

Без PDB:
- Call stack показывает только имена методов в IL (`<DoWorkAsync>b__0_1`).
- Локальные переменные видны как `local0`, `local1`.
- Step Into в библиотеку не работает (или работает в decompiled view).

В Debug-сборке PDB генерируется автоматически. В Release — тоже, но опционально.

### 8.2. Embedded vs portable PDB

Старый формат — **full PDB** (Windows-only). Новый — **portable PDB**, кросс-платформенный.

В csproj можно настроить:

```xml
<PropertyGroup>
  <DebugType>portable</DebugType>     <!-- portable PDB файл рядом с DLL -->
  <DebugType>embedded</DebugType>     <!-- встроен в DLL — один файл -->
  <DebugType>none</DebugType>         <!-- без PDB -->
  <DebugSymbols>true</DebugSymbols>
</PropertyGroup>
```

Embedded PDB удобен для дистрибуции — один файл вместо двух. Размер DLL чуть больше.

### 8.3. Source Link — отладка чужих библиотек

Source Link встраивает в PDB ссылку на git-репозиторий с исходниками. Когда debugger пытается step into в код библиотеки — VS / Rider скачивает соответствующий commit и показывает.

В csproj библиотеки (которая публикуется как NuGet):

```xml
<PropertyGroup>
  <PublishRepositoryUrl>true</PublishRepositoryUrl>
  <EmbedUntrackedSources>true</EmbedUntrackedSources>
  <DebugType>embedded</DebugType>
  <IncludeSymbols>true</IncludeSymbols>
  <SymbolPackageFormat>snupkg</SymbolPackageFormat>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="Microsoft.SourceLink.GitHub" Version="8.0.0" PrivateAssets="all" />
</ItemGroup>
```

Теперь любой debugger клиента может шагнуть в твою библиотеку.

### 8.4. .NET Source Link — отладка BCL

Microsoft публикует исходники .NET BCL с Source Link. В Visual Studio:

**Tools → Options → Debugging → General**:
- Disable **Just My Code**.
- Enable **Enable .NET source stepping**.
- Enable **Enable Source Link support**.

Теперь F11 в `int.Parse(s)` показывает реальные исходники из dotnet/runtime репозитория.

### 8.5. Symbol Servers

Если PDB не лежит рядом с DLL — debugger может скачать его с symbol server'а:

**Tools → Options → Debugging → Symbols**:
- Microsoft Symbol Servers (по умолчанию)
- NuGet.org Symbol Server (`https://symbols.nuget.org/download/symbols`)
- Custom (для корпоративных artifact'ов)

Кэш symbols — `~/.dotnet-symbol-cache/` (можно изменить путь).

### 8.6. dotnet-symbol — скачать symbols вручную

Tool для скачивания symbols к существующему dump или DLL:

```bash
dotnet tool install -g dotnet-symbol

# Скачать symbols к dump файлу
dotnet-symbol --symbols core.12345

# Скачать symbols ко всем DLL в папке
dotnet-symbol --symbols ./bin/Release/net10.0/*.dll
```

Полезно при анализе production-дампов.

### 8.7. Когда символы важны

- **Voice exception** в production: stack trace без symbols бесполезен.
- **Step into** библиотеки или своего модуля.
- **Memory dump analysis** — без symbols не разобрать дамп.
- **Async stack traces** требуют PDB для метаданных.

В CI всегда генерируй и публикуй symbols. Для приватных библиотек — на корпоративный symbol server.

> [!question]- Интервью: что такое portable PDB и зачем он нужен?
> Portable PDB — кросс-платформенный формат debug-символов для .NET (вместо старого full PDB, который только на Windows). Содержит соответствие IL → исходные файлы и строки, имена методов, локальных переменных, async-метаданные. Может встраиваться в DLL (embedded PDB) или лежать отдельным `.pdb`-файлом. Без PDB call stack показывает декомпилированные имена, локальные переменные не имеют имён, step into не работает корректно. Production-приложения должны включать symbols для возможности анализа production-дампов и stack traces.

---

## 9. dotnet-counters — live мониторинг

### 9.1. Что и зачем

`dotnet-counters` — CLI-tool для live мониторинга performance counter'ов работающего .NET-процесса. Без attach, без перезапуска, без модификации кода.

Установка:

```bash
dotnet tool install -g dotnet-counters
```

### 9.2. Базовое использование

```bash
# Список процессов .NET на машине
dotnet-counters ps

# Live мониторинг встроенных counter'ов
dotnet-counters monitor --process-id 12345
```

Что показывает (по умолчанию):

- **System.Runtime**: CPU usage, GC heap size, GC counts (Gen 0/1/2), exceptions/sec, working set, ThreadPool threads, lock contention/sec.
- **Microsoft.AspNetCore.Hosting**: requests/sec, current requests, failed requests.

Обновляется в реальном времени, как top.

### 9.3. Конкретные провайдеры

```bash
# Только AspNetCore
dotnet-counters monitor --process-id 12345 --counters Microsoft.AspNetCore.Hosting

# Несколько провайдеров
dotnet-counters monitor --process-id 12345 --counters System.Runtime,Microsoft.AspNetCore.Hosting

# Конкретные counter'ы
dotnet-counters monitor --process-id 12345 --counters System.Runtime[gc-heap-size,cpu-usage]
```

Список встроенных провайдеров: `dotnet-counters list`.

### 9.4. Custom Meter (из приложения)

В .NET 6+ можно создавать свои counter'ы через `System.Diagnostics.Metrics`:

```csharp
using System.Diagnostics.Metrics;

public class OrderService
{
    private static readonly Meter Meter = new("MyApp.Orders", "1.0.0");
    private static readonly Counter<long> OrdersProcessed = Meter.CreateCounter<long>("orders_processed");
    private static readonly Histogram<double> OrderDuration = Meter.CreateHistogram<double>("order_duration_ms");

    public async Task ProcessAsync(Order order)
    {
        var sw = Stopwatch.StartNew();
        try
        {
            await DoWork(order);
            OrdersProcessed.Add(1, new KeyValuePair<string, object?>("status", "success"));
        }
        catch
        {
            OrdersProcessed.Add(1, new KeyValuePair<string, object?>("status", "failed"));
            throw;
        }
        finally
        {
            OrderDuration.Record(sw.Elapsed.TotalMilliseconds);
        }
    }
}
```

Просмотр:

```bash
dotnet-counters monitor --process-id 12345 --counters MyApp.Orders
```

Эти же counter'ы можно отдавать в Prometheus / OpenTelemetry — единый код.

### 9.5. collect — сбор в файл для последующего анализа

```bash
# Записать в CSV для последующего анализа
dotnet-counters collect --process-id 12345 --output counters.csv --duration 00:05:00
```

Запись в формате CSV (с timestamps), потом открываешь в Excel / Grafana и строишь графики.

### 9.6. Use-case: production diagnostics

Сценарий: на проде CPU 90% непонятно почему. Steps:

1. SSH на машину / `kubectl exec` в под.
2. `dotnet-counters ps` → найти PID.
3. `dotnet-counters monitor -p <PID>` → 30 секунд смотришь counter'ы.
4. Видишь: GC time 80% — не CPU, а GC. Дальше — `dotnet-gcdump` (раздел 12).
5. Или: lock contention 10000/sec → bottleneck в lock. Дальше — `dotnet-trace` (раздел 10).

> [!question]- Интервью: чем `dotnet-counters` отличается от Prometheus / Grafana?
> `dotnet-counters` — лёгкий CLI-tool для on-demand мониторинга одного процесса, идёт в коробке с .NET. Не требует инфраструктуры. Подходит для quick diagnostics на конкретном поде / машине. Prometheus / Grafana — production-grade observability stack, собирает метрики со всех инстансов, хранит исторически, строит дашборды. В реальной системе используют OpenTelemetry / `System.Diagnostics.Metrics` для emit метрик, Prometheus scrape-ит их, Grafana визуализирует. `dotnet-counters` — для in-the-moment debugging, когда дашборд не покрывает.

---

## 10. dotnet-trace — sampling profiler

### 10.1. Что это

`dotnet-trace` — собирает trace-данные о работе процесса: какие методы выполнялись, сколько времени, какие allocations происходили, какие contention events.

```bash
dotnet tool install -g dotnet-trace
```

### 10.2. Базовый workflow

```bash
# Начать запись trace на 60 секунд
dotnet-trace collect --process-id 12345 --duration 00:01:00 --output trace.nettrace

# Или по нажатию Enter
dotnet-trace collect --process-id 12345
# ... подожди, потом Enter — запись остановится
```

Получишь файл `.nettrace`. Открыть можно:

- **Visual Studio**: File → Open → выбрать .nettrace.
- **PerfView** (Windows): drag-drop файл.
- **Speedscope** (web flame graph viewer):
  ```bash
  dotnet-trace convert trace.nettrace --format speedscope
  # → trace.speedscope.json — открыть на https://speedscope.app
  ```

### 10.3. Profiles — что собирать

```bash
dotnet-trace collect --profile cpu-sampling -p 12345     # CPU sampling (default)
dotnet-trace collect --profile gc-collect -p 12345       # GC events
dotnet-trace collect --profile gc-verbose -p 12345       # detailed GC
```

`cpu-sampling` — стандарт для «найти, где CPU горячий». Делает snapshot всех потоков ~1000 раз/сек, потом аггрегирует.

`gc-collect` — события сборщика мусора (gen, какие объекты).

### 10.4. EventPipe Providers

Можно собирать произвольные events:

```bash
dotnet-trace collect -p 12345 --providers Microsoft-Windows-DotNETRuntime:0x1F000080018:5
# 5 = Verbose, шестнадцатеричное число — keywords (что писать)
```

Provider — источник events (runtime, AspNetCore, custom EventSource). Keywords — bitmask с фильтром по типам событий.

### 10.5. Анализ в Speedscope (flame graph)

После `dotnet-trace convert ... --format speedscope` получаешь JSON, открываешь на speedscope.app:

- **Time Order** — flame graph по времени.
- **Left Heavy** — сгруппировано по самым долгим путям.
- **Sandwich** — показывает функцию + все её caller / callee.

Самое полезное для quick diagnosis: видишь широкий блок вверху flame graph — это CPU-bottleneck. Идёшь вниз — какие методы он зовёт.

### 10.6. Allocations tracing

```bash
dotnet-trace collect -p 12345 --providers Microsoft-DotNETCore-SampleProfiler --providers Microsoft-Windows-DotNETRuntime:0x000001:5
```

Показывает, какие методы аллоцировали много объектов. Для anti-allocation работы (hot path в HFT, parsers) — обязательный инструмент.

### 10.7. dotnet-trace в Docker

```bash
# В контейнере
kubectl exec -it my-pod -- /bin/bash
dotnet tool install -g dotnet-trace
export PATH="$PATH:$HOME/.dotnet/tools"
dotnet-trace ps
dotnet-trace collect -p 1 --duration 00:01:00 --output /tmp/trace.nettrace

# Скопировать наружу
kubectl cp my-pod:/tmp/trace.nettrace ./trace.nettrace
```

Дальше анализируешь локально.

> [!question]- Интервью: чем sampling profiler отличается от instrumentation profiler?
> Sampling: периодически (1000 раз/сек) делает snapshot стеков всех потоков. Накладные расходы маленькие (~5%), не нужна модификация кода. Точность статистическая — короткие методы могут не попасть в выборку. Instrumentation: модифицирует IL вокруг каждого метода для записи входа/выхода. Точность 100%, но overhead 50-200%. `dotnet-trace`, perf, async-profiler — sampling. dotTrace в режиме Tracing — instrumentation. Для production — sampling (низкий overhead). Для микро-оптимизаций — instrumentation.

---

## 11. dotnet-dump — full memory dumps

### 11.1. Что и зачем

`dotnet-dump` — собирает полный snapshot памяти процесса (full dump) и позволяет анализировать его offline. Ключевое для анализа production-крашей и зависаний.

```bash
dotnet tool install -g dotnet-dump
```

### 11.2. Сбор дампа

```bash
# Full dump живого процесса
dotnet-dump collect --process-id 12345 --output dump.dmp

# По типу dump'а
dotnet-dump collect -p 12345 --type Full        # все managed + native heap (огромный, GB)
dotnet-dump collect -p 12345 --type Heap        # только managed heap (для memory leaks)
dotnet-dump collect -p 12345 --type Mini        # минимум (только critical info)
```

Размер:
- **Mini**: 1-10 MB — для quick хеш-чекинга.
- **Heap**: 100 MB - 2 GB — для анализа managed objects.
- **Full**: 1-10 GB — для полного forensic анализа.

В production обычно **Heap** — компромисс размер/информативность.

### 11.3. Crash dump на падении

В коде:

```csharp
AppDomain.CurrentDomain.UnhandledException += (sender, args) =>
{
    var dumpPath = $"/tmp/crash-{DateTime.UtcNow:yyyyMMdd-HHmmss}.dmp";
    Process.Start("dotnet-dump", $"collect -p {Environment.ProcessId} -o {dumpPath} --type Heap").WaitForExit();
    Environment.Exit(1);
};
```

Или через переменные окружения для CLR Crash dumps:

```bash
export DOTNET_DbgEnableMiniDump=1
export DOTNET_DbgMiniDumpType=2          # 2 = Heap, 4 = Full
export DOTNET_DbgMiniDumpName=/tmp/coredump.%p
```

При краше runtime сам создаст дамп.

### 11.4. Анализ — dotnet-dump analyze

```bash
dotnet-dump analyze dump.dmp
```

Открывается интерактивный shell с командами SOS:

```
> clrstack          # call stack текущего потока
> clrstack -all     # все потоки
> threads           # список потоков
> dumpheap -stat    # статистика heap по типам
> dumpheap -mt 0x... # все объекты конкретного типа
> gcroot 0x...      # кто держит ссылку на объект
> syncblk           # все locked объекты, кто блокирует
> !analyze          # auto-analysis
> exit
```

Ключевые workflow:

**Memory leak:**
```
> dumpheap -stat
# Видишь top объектов по размеру
# Например: byte[] — 500 MB, MyCachedItem — 300 MB

> dumpheap -mt <MyCachedItem MT> -short
# Список адресов всех instance'ов

> gcroot <адрес>
# Видишь цепочку референсов — кто держит
```

**Hang / deadlock:**
```
> threads
# Список потоков

> clrstack -all
# Видишь, кто на чём ждёт

> syncblk
# Видишь, какие объекты locked, кто их забрал
```

### 11.5. dotnet-dump в Kubernetes

```bash
# В под (если есть SDK)
kubectl exec -it my-pod -- bash
dotnet tool install -g dotnet-dump
$HOME/.dotnet/tools/dotnet-dump ps
$HOME/.dotnet/tools/dotnet-dump collect -p 1 -o /tmp/dump.dmp --type Heap

# Скопировать наружу
kubectl cp my-pod:/tmp/dump.dmp ./dump.dmp

# Анализировать локально
dotnet-dump analyze ./dump.dmp
```

Лучше использовать `dotnet-monitor` (раздел 13) — он автоматизирует.

### 11.6. WinDbg + SOS

Для тяжёлых случаев — WinDbg (Windows только) с SOS extension. Это ультра-инструмент с возможностями глубже, чем `dotnet-dump`. Используется в Microsoft support, при анализе нативных багов CLR. Junior не нужен, на интервью — упомянуть, что знаешь.

> [!question]- Интервью: что делает `dumpheap -stat` в SOS?
> Показывает статистику managed heap — сколько объектов каждого типа и сколько они занимают суммарно. Колонки: Method Table address, Count, Total Size, Type Name. Используется как первый шаг при анализе memory leak — видишь, какой тип «пухнет» больше всего. Потом `dumpheap -mt <addr>` для списка instance'ов и `gcroot <obj-addr>` для нахождения, кто держит ссылку. Это базовый workflow forensic анализа дампов.

---

## 12. dotnet-gcdump — heap snapshots

### 12.1. Что это

`dotnet-gcdump` — снимок только managed heap, без native памяти и stack-frames. Намного легче full dump (50-200 MB vs 5 GB), быстрее собирается.

```bash
dotnet tool install -g dotnet-gcdump
```

### 12.2. Сбор и анализ

```bash
# Снять snapshot
dotnet-gcdump collect -p 12345 -o snap1.gcdump

# Открыть в Visual Studio (только Windows)
# File → Open → snap1.gcdump

# Или в PerfView, или в JetBrains dotMemory
```

### 12.3. Workflow для memory leak

```bash
# Snapshot 1 — после старта
dotnet-gcdump collect -p 12345 -o snap1.gcdump
sleep 600   # подожди 10 минут

# Snapshot 2 — позже
dotnet-gcdump collect -p 12345 -o snap2.gcdump

# В Visual Studio: загрузить оба, View → Compare
# Видишь, сколько объектов каждого типа добавилось/исчезло
```

```
Type                    Count diff    Size diff
─────────────────────  ───────────  ─────────────
MyCachedItem               +50000     +200 MB     ← leak здесь
byte[]                     +30000     +150 MB     ← leak (вторичный)
String                       +500       +80 KB
int[]                          +0         0 B
```

Видишь, какой тип растёт неконтролируемо.

### 12.4. Преимущества над dotnet-dump

- **Размер**: 50-200 MB vs 1-10 GB.
- **Скорость сбора**: секунды vs минуты.
- **Скорость анализа**: VS / PerfView открывает быстро.
- **Comparison**: легко увидеть delta между snapshot'ами.

Минусы:
- Нет native stack frames.
- Нет thread states / deadlock info.
- Только managed heap.

Для memory leak — `gcdump` идеально. Для hang / native crash — `dump`.

### 12.5. dotnet-gcdump в Docker

```bash
kubectl exec -it my-pod -- bash
dotnet tool install -g dotnet-gcdump
~/.dotnet/tools/dotnet-gcdump collect -p 1 -o /tmp/snap.gcdump
exit
kubectl cp my-pod:/tmp/snap.gcdump ./snap.gcdump
```

> [!question]- Интервью: чем `gcdump` отличается от `dump`?
> `dotnet-gcdump` снимает только managed heap (объекты + ссылки между ними). `dotnet-dump` — полный snapshot процесса (managed heap + native heap + stack frames + threads + lock state). `gcdump` лёгкий (50-200 MB), быстро собирается, удобен для memory leak detection через snapshot comparison. `dump` тяжёлый (1-10 GB), но даёт всё для forensic анализа крашей и зависаний. На production memory leak обычно начинают с `gcdump`, для hang / crash — с `dump`.

---

## 13. dotnet-monitor — sidecar для Kubernetes

### 13.1. Что это

`dotnet-monitor` — daemon, который предоставляет REST API над всеми diagnostic-инструментами (counters, trace, dump, gcdump, logs). Деплоится как sidecar в Kubernetes-под.

Основной use-case: автоматизированный сбор diagnostics в production:

- Auto-trigger при CPU > 80%.
- Health check endpoint.
- Hot-collected dumps на 503 errors.
- Egress в Azure Storage / S3 / локальный volume.

### 13.2. Установка

```bash
# Как .NET tool (для локальной отладки)
dotnet tool install -g dotnet-monitor

# Как Docker image
docker pull mcr.microsoft.com/dotnet/monitor:9
```

### 13.3. Базовый запуск

```bash
dotnet-monitor collect --no-auth --urls http://localhost:52323 --metrics-urls http://localhost:52325
```

REST API:

```bash
# Список процессов
curl http://localhost:52323/processes

# Counters в реальном времени (SSE stream)
curl http://localhost:52323/livemetrics?pid=12345

# Сбор trace на 30 секунд
curl -X POST "http://localhost:52323/trace?pid=12345&durationSeconds=30" --output trace.nettrace

# Сбор gcdump
curl -X POST "http://localhost:52323/gcdump?pid=12345" --output snap.gcdump

# Сбор full dump
curl -X POST "http://localhost:52323/dump?pid=12345" --output dump.dmp
```

### 13.4. Auto-collect rules

В `appsettings.json`:

```json
{
  "CollectionRules": {
    "HighCpuTrace": {
      "Trigger": {
        "Type": "EventCounter",
        "Settings": {
          "ProviderName": "System.Runtime",
          "CounterName": "cpu-usage",
          "GreaterThan": 80,
          "SlidingWindowDuration": "00:00:30"
        }
      },
      "Actions": [
        {
          "Type": "CollectTrace",
          "Settings": {
            "Profile": "Cpu",
            "Duration": "00:00:30",
            "Egress": "AzureBlob"
          }
        }
      ],
      "Limits": {
        "ActionCount": 5,
        "ActionCountSlidingWindowDuration": "01:00:00"
      }
    }
  }
}
```

Что это значит: «если CPU > 80% более 30 секунд — собери 30-секундный trace и положи в Azure Blob». Срабатывает максимум 5 раз в час.

### 13.5. Sidecar в Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-api
spec:
  template:
    spec:
      shareProcessNamespace: true
      containers:
      - name: app
        image: my-api:latest
        env:
        - name: DOTNET_DiagnosticPorts
          value: /diag/dotnet-monitor.sock
        volumeMounts:
        - name: diagsocket
          mountPath: /diag
      - name: monitor
        image: mcr.microsoft.com/dotnet/monitor:9
        ports:
        - containerPort: 52323
        env:
        - name: DOTNETMONITOR_DiagnosticPort__ConnectionMode
          value: Listen
        - name: DOTNETMONITOR_DiagnosticPort__EndpointName
          value: /diag/dotnet-monitor.sock
        volumeMounts:
        - name: diagsocket
          mountPath: /diag
      volumes:
      - name: diagsocket
        emptyDir: {}
```

Sidecar связывается с main container через UNIX socket. Дальше REST API работает на порту sidecar'а.

### 13.6. Когда использовать

- **Производственные сервисы в Kubernetes / cloud** — auto-collect при инцидентах.
- **High-traffic** — нужна low-overhead диагностика по запросу.
- **Compliance / SRE** — централизованный egress дампов в storage.

Для маленьких приложений — overkill, хватит ручных `dotnet-trace` / `dotnet-gcdump`.

> [!question]- Интервью: зачем нужен `dotnet-monitor` в Kubernetes?
> Это sidecar, который предоставляет REST API над всеми .NET diagnostic-инструментами. Решает проблему «как собрать trace / dump из работающего пода без kubectl exec». Основные функции: автоматический collect при превышении thresholds (CPU, GC, exceptions), health endpoint, egress в Azure / S3 / volume. SRE настраивают rules, при инциденте артефакты автоматически появляются в storage без ручного вмешательства.

---

## 14. PerfView и JetBrains tools

### 14.1. PerfView (Windows)

Microsoft-овский tool для анализа ETW-трасс и `.nettrace` файлов. Бесплатный, мощный, но UI «specialist-friendly» — не самый юзабельный.

Основное:
- Анализ CPU sampling traces.
- GC heap анализ.
- Thread time analysis (где потоки спят).
- Allocation tracking.

Когда использовать:
- Глубокий анализ traces, которые `dotnet-trace` собрал.
- Когда speedscope не показывает нужный аспект.
- ETW-trace анализ с Windows-системными events.

Скачать: github.com/Microsoft/perfview.

### 14.2. dotMemory (JetBrains)

Memory profiler для .NET. Платный (есть free trial).

Возможности:
- Live мониторинг heap при работающем приложении.
- Snapshots с comparison (как `gcdump`, но удобнее).
- Анализ retention paths (кто держит ссылки).
- Memory traffic — какие методы аллоцируют.
- Plugin к Rider.

Когда использовать:
- Глубокое investigation memory leaks.
- Production memory profiling (через dotMemory Standalone).

### 14.3. dotTrace (JetBrains)

CPU profiler для .NET. Платный.

Возможности:
- Sampling, Tracing, Line-by-line modes.
- Performance graphs с возможностью drill-down.
- Subsystems — группировка по логическим частям.
- Plugin к Rider.

### 14.4. Visual Studio Profiler

Встроен в VS Enterprise:
- CPU Usage tool.
- Memory Usage tool.
- .NET Object Allocation Tracking.
- .NET Async tool.

Удобно для quick checks без переключения на отдельный tool. По мощности уступает PerfView / dotTrace, но 80% задач закрывает.

### 14.5. Когда что выбирать

| Задача | Free | Paid |
|--------|------|------|
| Quick CPU profiling | dotnet-trace + speedscope | dotTrace |
| Memory leak detection | dotnet-gcdump + VS | dotMemory |
| Production live monitoring | dotnet-counters | Datadog / New Relic |
| Distributed tracing | OpenTelemetry + Jaeger | Datadog APM |
| ETW deep analysis | PerfView | — |
| Production sidecar | dotnet-monitor | — |

> [!question]- Интервью: какие инструменты используешь для профилирования .NET?
> Для quick CPU profiling — `dotnet-trace` + `speedscope.app` (бесплатно, кросс-платформенно). Для memory leaks — `dotnet-gcdump` + Visual Studio (или dotMemory, если коммерческий проект). Production live monitoring — `dotnet-counters` или OpenTelemetry → Prometheus / Grafana с alerts. Для глубокого ETW-анализа на Windows — PerfView. Из платных — dotTrace и dotMemory от JetBrains, удобный UI и интеграция с Rider. Visual Studio Enterprise имеет встроенный profiler — 80% задач покрывает, для углублённого анализа уступает специализированным.

---

## 15. Async debugging

### 15.1. Почему async сложнее sync

Синхронный код:
```
Main → A → B → C → throw
```
Stack trace показывает всю цепочку, отладка intuitive.

Async код:
```csharp
async Task Main()
{
    await DoWork();
}
async Task DoWork()
{
    await Task.Delay(100);
    Throw();   // ← exception
}
```

Под капотом каждый `await` — это разрезание метода на части (state machine). После `await Task.Delay(100)` поток освобождается, продолжение запускается на другом потоке (часто). Stack trace раньше обрывался — невозможно увидеть «кто запустил эту цепочку».

С .NET 6+ async stack traces улучшились: debugger восстанавливает logical caller chain.

### 15.2. Tasks window

VS: **Debug → Windows → Tasks** (Ctrl+Shift+D, K).

Показывает все активные `Task`-и в процессе:
- Что они делают.
- На каком потоке.
- Текущий status (Running, WaitingForActivation, RanToCompletion, Faulted).
- Stack каждой Task.

Полезно для:
- «Что делают все мои background tasks прямо сейчас?»
- Поиска зависших Task (не завершаются).
- Отладки `Task.WhenAll` — какая из задач упала.

### 15.3. Parallel Stacks для async

VS: **Debug → Windows → Parallel Stacks** (Ctrl+Shift+D, S).

Древовидная визуализация всех async chain'ов. Где две Task сливаются (`WhenAll`) или ветвятся (`Task.Run` внутри другой Task) — видно сразу.

### 15.4. Deadlock detection

Async deadlock — классика:

```csharp
public string GetData()
{
    return GetDataAsync().Result;   // ❌ может deadlock-нуть в UI / ASP.NET (старый pipeline)
}

public async Task<string> GetDataAsync()
{
    await Task.Delay(100);   // continuation на UI thread, который занят .Result
    return "data";
}
```

Симптомы: приложение зависло, не отвечает. Что делать в debugger:

1. Pause execution (Debug → Break All).
2. Tasks window — увидишь Task в WaitingForActivation, Continuation pending.
3. Threads window — main thread в Wait/Sleep/Join state.
4. Это deadlock — replace `.Result` на `await`.

### 15.5. ConfigureAwait(false)

```csharp
public async Task<string> GetDataAsync()
{
    await Task.Delay(100).ConfigureAwait(false);
    // Continuation НЕ требует синхронизационного контекста
    return "data";
}
```

В библиотечном коде — всегда `ConfigureAwait(false)`. Это позволяет continuation выполниться на любом thread pool потоке, а не специфичном (UI thread, ASP.NET classic context). Снижает риск deadlock в caller-коде.

В **ASP.NET Core** SynchronizationContext отсутствует — `ConfigureAwait(false)` бесполезен (но и не вредит). В **WPF / WinForms / Xamarin** — обязателен.

### 15.6. CancellationToken — debugging

```csharp
public async Task<string> LoadAsync(CancellationToken ct = default)
{
    var data = await _httpClient.GetStringAsync(url, ct);
    return data;
}
```

Добавь breakpoint, запусти, отмени через `cts.Cancel()`. В debugger увидишь `OperationCanceledException` (если включён в Exception Settings). Удобно проверить, что cancellation корректно проброшена.

### 15.7. Async-friendly logging

```csharp
public async Task ProcessAsync(int id)
{
    using (_logger.BeginScope(new { OrderId = id, CorrelationId = Activity.Current?.Id }))
    {
        _logger.LogInformation("Processing started");
        await DoWorkAsync(id);
        _logger.LogInformation("Processing finished");
    }
}
```

`BeginScope` сохраняет контекст для всех логов внутри (включая после `await`). `Activity.Current` (.NET 5+) — distributed tracing context, проходит через async-границы автоматически.

> [!question]- Интервью: почему async код сложнее отлаживать?
> `await` разрезает метод на state machine; continuation выполняется на потенциально другом потоке. До .NET 6 stack trace обрывался на `await` — нельзя было восстановить logical caller chain. С .NET 6+ JIT записывает caller-метаданные, stack traces показывают полную цепочку. Для visualization активных Task'ов используется Tasks window и Parallel Stacks. Главные ловушки: deadlock от `.Result` / `.Wait()` (continuation не получает thread); потерянная exception в orphaned task без `await`; race condition при параллельном выполнении.

---

## 16. Production debugging

### 16.1. Структурированное логирование с correlation

Production не имеет debugger. Заменяет — structured logging:

```csharp
public async Task<Order> ProcessAsync(OrderRequest req)
{
    var correlationId = Guid.NewGuid();
    using (_logger.BeginScope(new { CorrelationId = correlationId, OrderId = req.OrderId }))
    {
        _logger.LogInformation("Processing started: {ItemCount} items", req.Items.Count);
        try
        {
            var order = await DoWorkAsync(req);
            _logger.LogInformation("Processed: total={Total:C}, duration={Duration}ms", order.Total, sw.Elapsed.TotalMilliseconds);
            return order;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed: req={@Request}", req);
            throw;
        }
    }
}
```

В Seq / ELK / Datadog ищешь:
- `CorrelationId = "<id>"` → видишь все логи одного запроса.
- `Level = Error` за последний час → видишь все падения.
- `OrderId = 42` → история этого order через все микросервисы.

### 16.2. Distributed tracing (OpenTelemetry)

`Activity` API в .NET (System.Diagnostics):

```csharp
private static readonly ActivitySource Source = new("MyApp.Orders");

public async Task ProcessAsync(int orderId)
{
    using var activity = Source.StartActivity("ProcessOrder");
    activity?.SetTag("order.id", orderId);

    var customer = await GetCustomer(orderId);
    activity?.SetTag("customer.id", customer.Id);

    await Process(customer);

    activity?.SetStatus(ActivityStatusCode.Ok);
}
```

Через OpenTelemetry exporter — отправка в Jaeger / Zipkin / Datadog. Дальше видишь distributed trace со span'ами по всем сервисам, через которые прошёл request.

### 16.3. APM platforms

Для production — сразу платный APM:

- **Application Insights** (Azure) — встроенно с ASP.NET Core.
- **Datadog APM** — мощный, дорогой.
- **New Relic** — альтернатива Datadog.
- **Elastic APM** — OSS-friendly стек.
- **Sentry** — для error tracking + perf monitoring.

Они умеют:
- Distributed tracing.
- Auto-instrumentation HTTP, EF Core, gRPC.
- Anomaly detection (CPU spike, error rate spike).
- Alerting через Slack / PagerDuty.

### 16.4. Crash dumps в production

Для сложных production-багов:

```bash
# Включить автоматический dump при краше
export DOTNET_DbgEnableMiniDump=1
export DOTNET_DbgMiniDumpType=2          # Heap
export DOTNET_DbgMiniDumpName=/var/dumps/coredump.%p.%t

# Перезапустить процесс
```

При crash runtime сам создаст dump. Дальше анализируешь через `dotnet-dump analyze`.

В Kubernetes — связать с persistent volume, чтобы dump не пропал при рестарте пода:

```yaml
volumeMounts:
- name: dumps
  mountPath: /var/dumps

volumes:
- name: dumps
  persistentVolumeClaim:
    claimName: app-dumps-pvc
```

### 16.5. Health checks + readiness probes

```csharp
builder.Services.AddHealthChecks()
    .AddCheck("db", () => _db.Ping() ? HealthCheckResult.Healthy() : HealthCheckResult.Unhealthy())
    .AddCheck<RedisHealthCheck>("redis")
    .AddCheck<DiskSpaceCheck>("disk");

app.MapHealthChecks("/health");
app.MapHealthChecks("/ready", new HealthCheckOptions { Predicate = c => c.Tags.Contains("ready") });
```

Kubernetes использует endpoint для restart unhealthy pods. Если `/health` 503 → pod перезапускается. Это not debugging, но превентивная защита от плохого состояния.

### 16.6. Feature flags для продакшн A/B debugging

Когда ты не уверен, что фикс безопасный:

```csharp
if (await _featureFlags.IsEnabledAsync("new-pricing-algorithm", userId))
{
    return await _newPricing.CalculateAsync(request);
}
return await _oldPricing.CalculateAsync(request);
```

Флипаешь флаг для 1% юзеров → мониторишь metrics → если OK — для 50% → 100%. Если плохо — flag off, никто не пострадал.

> [!question]- Интервью: как отлаживают production-приложения, где debugger недоступен?
> Через несколько слоёв: 1) Structured logging с correlation ID и `BeginScope` для request tracing. 2) Distributed tracing через `Activity` / OpenTelemetry → Jaeger / Datadog APM. 3) Metrics через `System.Diagnostics.Metrics` → Prometheus / Grafana с alerts. 4) Crash dumps (DOTNET_DbgEnableMiniDump) с offline-анализом через `dotnet-dump analyze`. 5) `dotnet-monitor` sidecar для on-demand traces / dumps без kubectl exec. 6) Feature flags для безопасного rollout. На отладку production-бага обычно идёт несколько часов вместо минут локально — поэтому профилактика (тесты, логи, метрики) критичны.

---

## 17. Memory leaks — workflow

### 17.1. Признаки

- Memory растёт со временем без stabilization.
- OOMKilled в Kubernetes (sigkill 9 после превышения memory limit).
- GC time растёт пропорционально memory (Gen2 collections занимают всё больше).
- Working set / Private bytes counter не возвращается к baseline после нагрузки.

### 17.2. Workflow расследования

**Шаг 1.** Подтвердить, что это leak, а не просто растущий dataset:
```bash
dotnet-counters monitor -p <PID> --counters System.Runtime
# Смотреть gc-heap-size в течение 30 минут
```

Если GC heap размер растёт **после** GC (не просто на пиках) — leak.

**Шаг 2.** Снять первый snapshot:
```bash
dotnet-gcdump collect -p <PID> -o snap1.gcdump
```

**Шаг 3.** Подождать 10-30 минут под нагрузкой.

**Шаг 4.** Снять второй snapshot:
```bash
dotnet-gcdump collect -p <PID> -o snap2.gcdump
```

**Шаг 5.** Открыть оба в Visual Studio (или dotMemory):
- File → Open → snap1.gcdump
- File → Open → snap2.gcdump
- Compare

**Шаг 6.** Смотреть, какие типы выросли больше всего:
```
Type                    +Count    +Size
─────────────────────  ───────  ───────
EventHandler<...>       +5000    +20 MB
List<CachedItem>          +200    +50 MB
byte[]                  +30000   +200 MB
```

**Шаг 7.** Найти retention path — кто держит ссылки:
- Right-click на типе → "Show retention paths".
- Видишь дерево от root (статические поля, threadpool, GC roots) до leaked object.

**Шаг 8.** Fix — обычно один из:
- Незаписанный unsubscribe от события (`+=` без `-=`).
- Бесконечно растущий cache без TTL / eviction.
- Static-коллекция, в которую кладут, но не убирают.
- Captured variable в long-living closure.

### 17.3. Типичные причины leak'ов в .NET

**Event handlers без unsubscribe:**

```csharp
public class Subscriber : IDisposable
{
    private readonly Publisher _publisher;
    public Subscriber(Publisher p) { _publisher = p; _publisher.OnEvent += Handle; }
    private void Handle(object? s, EventArgs e) { /* ... */ }
    public void Dispose() => _publisher.OnEvent -= Handle;
}

// Если Subscriber не Dispose()-ится, Publisher держит ссылку на Handle method,
// который держит ссылку на this (Subscriber). Subscriber никогда не GC'ится.
```

**Растущий cache:**

```csharp
private static readonly Dictionary<int, ExpensiveObject> _cache = new();

public ExpensiveObject Get(int id)
{
    if (!_cache.TryGetValue(id, out var obj))
        _cache[id] = obj = LoadExpensive(id);
    return obj;
}
// Если уникальных id миллион — cache растёт неограниченно.
// Решение: MemoryCache с SizeLimit и SlidingExpiration.
```

**Long-running captured variable:**

```csharp
public Task StartBackground(byte[] hugeData)
{
    return Task.Run(() => Process(hugeData));
    // hugeData captured в closure, держится до завершения Task.
    // Если Task running 1 час — данные на куче 1 час.
}
```

**HttpClient без переиспользования:**

```csharp
public async Task<string> GetAsync(string url)
{
    using var client = new HttpClient();   // ❌ создаёт новый socket каждый раз
    return await client.GetStringAsync(url);
}
// Решение — IHttpClientFactory с DI
```

> [!question]- Интервью: как искать memory leak в .NET?
> 1) Подтвердить через `dotnet-counters` (gc-heap-size растёт). 2) Снять `dotnet-gcdump` snapshot. 3) Подождать. 4) Снять второй snapshot. 5) Compare в Visual Studio / dotMemory. 6) Найти top-grow типов. 7) Через retention paths (gcroot в SOS или UI tools) найти, кто держит ссылку. 8) Типичные причины: event handlers без unsubscribe, неограниченные кеши, captured variables в closures, static-коллекции, не-disposed singletons. Регулярная профилактика: code review на эти паттерны, integration tests с memory assertion (`GC.GetTotalMemory(true)` до/после).

---

## 18. Performance debugging — workflow

### 18.1. Этапы

```
1. Симптом → метрика, желательно числом
   («медленно» → «p99 latency 5s vs 200ms baseline»)

2. Гипотеза — где bottleneck?
   (database / network / CPU / memory / lock contention)

3. Подтвердить через dotnet-counters
   (CPU 90%? GC time 50%? thread pool starvation?)

4. Если CPU/methods — dotnet-trace + speedscope
5. Если allocations — dotnet-trace с allocation provider
6. Если memory — dotnet-gcdump
7. Если IO/network — distributed tracing (OpenTelemetry)

8. Hypothesis-driven fix → измерить → подтвердить
```

### 18.2. Stopwatch для quick check

```csharp
public async Task<Result> ComplexOperation(Request req)
{
    var sw = Stopwatch.StartNew();

    var data = await LoadData(req);
    _logger.LogInformation("LoadData: {Ms} ms", sw.ElapsedMilliseconds);
    sw.Restart();

    var processed = Transform(data);
    _logger.LogInformation("Transform: {Ms} ms", sw.ElapsedMilliseconds);
    sw.Restart();

    var result = await SaveResult(processed);
    _logger.LogInformation("SaveResult: {Ms} ms", sw.ElapsedMilliseconds);

    return result;
}

// Output:
// LoadData: 50 ms
// Transform: 5 ms
// SaveResult: 3500 ms  ← bottleneck!
```

Грубо, но даёт сразу понять где время. Дальше углубляешься в подозрительный компонент.

### 18.3. BenchmarkDotNet для микро-измерений

Когда нужно измерить точно, не «на глаз», для оптимизации hot path:

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

public class StringBuilderBench
{
    private readonly string[] _parts = Enumerable.Range(0, 1000).Select(i => $"part{i}").ToArray();

    [Benchmark(Baseline = true)]
    public string Concat()
    {
        var s = "";
        foreach (var p in _parts) s += p;
        return s;
    }

    [Benchmark]
    public string Builder()
    {
        var sb = new StringBuilder();
        foreach (var p in _parts) sb.Append(p);
        return sb.ToString();
    }

    [Benchmark]
    public string Join() => string.Join("", _parts);
}

BenchmarkRunner.Run<StringBuilderBench>();
```

Результат:
```
| Method  |        Mean |      Allocated |  Ratio |
|-------- |------------:|---------------:|-------:|
| Concat  | 1,250.34 us | 4,932,468 B    |   1.00 |
| Builder |     35.12 us |    65,792 B    |   0.03 |
| Join    |     12.45 us |    16,184 B    |   0.01 |
```

BenchmarkDotNet делает warmup, многократные runs, статистическую обработку — точные результаты, не «у меня показалось быстрее».

### 18.4. Profiling в production

В production BenchmarkDotNet нельзя — модифицирует поведение. Используется:

- `dotnet-counters` для baseline metrics.
- `dotnet-trace` для on-demand sampling (5-10% overhead — приемлемо).
- APM platforms (Datadog / Application Insights) для непрерывного мониторинга.
- Custom metrics через `System.Diagnostics.Metrics` для domain-specific KPI (orders/sec, average cart size).

> [!question]- Интервью: что делать, если приложение медленное в production?
> 1) Получить метрику в числах (p99 latency, throughput). 2) `dotnet-counters` для quick state — где CPU, GC time, thread pool. 3) Если CPU bottleneck — `dotnet-trace` 30 секунд + speedscope для flame graph. 4) Если memory — `dotnet-gcdump` snapshot. 5) Если I/O — distributed tracing (OpenTelemetry / Datadog APM) для span breakdown по downstream вызовам. 6) Локально воспроизвести через нагрузочный тест и BenchmarkDotNet. 7) Hypothesis → fix → re-measure. Никогда «оптимизация без замера»: 80% догадок ошибочны.

---

## 19. Common Pitfalls — с механизмами

### 19.1. Console.WriteLine в production-коде

```csharp
public int Calculate(int x)
{
    Console.WriteLine($"x = {x}");   // ❌ забыл убрать
    return x * 2;
}
```

**Механизм:** при code review проскочило, в production засоряет stdout, при высокой нагрузке тормозит код.

**Фикс:** использовать `_logger.LogDebug` — отключается через config в production. Анализатор `RoslynanalyzeBannedSymbols` может банить `Console.Write*`.

### 19.2. Breakpoint в hot loop

```csharp
for (int i = 0; i < 1_000_000; i++)
{
    Process(i);   // ← breakpoint срабатывает миллион раз
}
```

**Механизм:** обычный breakpoint бесполезен — F5 миллион раз нажимать.

**Фикс:** conditional breakpoint `i == 999_999` или hit count, или временный breakpoint после loop.

### 19.3. F11 в библиотечный код

Жмёшь F11 на `int.Parse(s)` — оказался внутри .NET BCL, путаешься. **Механизм:** Just My Code не было выключено, но debugger всё равно зашёл (или Just My Code выключено, и зашёл сразу).

**Фикс:** Shift+F11 (Step Out) для выхода. Включи Just My Code в Tools → Options.

### 19.4. Edit and Continue для async

```csharp
async Task DoWork()
{
    await Task.Delay(1000);
    // изменить код тут — может не сработать!
}
```

**Механизм:** EnC не умеет менять state machine async-метода. Часть изменений принимается, часть — нет.

**Фикс:** restart debugger через Ctrl+Shift+F5 или Hot Reload (`dotnet watch run`).

### 19.5. Catch без logging

```csharp
catch (Exception)
{
    return null;   // ❌ swallow silently
}
```

**Механизм:** exception проглочена, в логах нет следа. Bug не воспроизводится — никто не знает.

**Фикс:**
```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "Failed in MethodX");
    return null;
}
```

### 19.6. throw ex; вместо throw;

```csharp
catch (Exception ex)
{
    LogIt(ex);
    throw ex;   // ❌ обнуляет stack trace
}
```

**Механизм:** `throw ex;` сбрасывает stack до текущей строки. Original origin теряется.

**Фикс:** `throw;` без аргумента — rethrow с сохранением stack.

### 19.7. Logging в hot path

```csharp
public int Add(int a, int b)
{
    _logger.LogDebug("Add({A}, {B})", a, b);   // 1M calls/sec → log overhead!
    return a + b;
}
```

**Механизм:** даже LogDebug-уровень с filtering выключен — message-template форматирование стоит.

**Фикс:** `IsEnabled` check для дорогих логов:
```csharp
if (_logger.IsEnabled(LogLevel.Debug))
    _logger.LogDebug("Complex {Data}", expensive.GetDetails());
```

### 19.8. Symbols missing в production

Production crash dump получили, открываем в `dotnet-dump analyze`, stack trace показывает `<unknown>` методы.

**Механизм:** PDB не задеплоились вместе с DLL.

**Фикс:** в csproj — `<DebugType>embedded</DebugType>` (PDB внутри DLL) или гарантировать копирование `.pdb` рядом с `.dll` при deploy.

### 19.9. .Result в async-цепочке

```csharp
public string GetData() => GetDataAsync().Result;   // ❌ потенциальный deadlock
```

**Механизм:** в UI/legacy ASP.NET continuation требует определённого SynchronizationContext, который занят `.Result` блокировкой.

**Фикс:** async всю цепочку. Если действительно нужно sync (entry point, нет async API) — `.GetAwaiter().GetResult()` чуть лучше из-за exception propagation, но всё равно опасно.

### 19.10. Не использовать `nameof` в логах

```csharp
_logger.LogError("Argument 'userId' was null");   // ❌
```

**Механизм:** при рефакторинге переменную переименовали в `customerId`, лог остался — устарел.

**Фикс:**
```csharp
_logger.LogError("Argument {Param} was null", nameof(userId));
```

При переименовании `nameof` обновится автоматически.

> [!question]- Интервью: топ-5 ошибок начинающих в отладке.
> 1) `Console.WriteLine`-debugging вместо breakpoint'ов — медленно, мусор в коде. 2) F11 повсюду вместо F10 — потеря времени в библиотеках. 3) `throw ex` вместо `throw` — стирание stack trace. 4) Catch без логирования — баг исчезает в production. 5) `.Result` / `.Wait()` в async — потенциальный deadlock. Все эти ошибки видны при code review, поэтому regular code review с фокусом на debugging hygiene — лучшая профилактика.

---

## 20. Best Practices

### 20.1. Mindset

- **Hypothesize first** — что должно быть? что вижу? в чём расхождение?
- **Binary search в коде** — bug где-то в большом коде → проверь invariant в середине, half it.
- **Read errors carefully** — stack trace говорит почти всё. Внимательно читай, не пропускай.
- **Reproduce locally first** — production-bug нельзя «отладить через VPN». Сначала минимальный repro.
- **Fix → regression test** — каждый bug должен оставить failing test, который теперь зелёный.
- **Don't trust the comment, trust the code** — код выполняется, comment — нет.

### 20.2. Daily workflow

```
1. Bug report пришёл
   ↓
2. Воспроизвести локально (минимальный repro)
   ↓
3. Поставить breakpoint в подозрительном месте
   ↓
4. Запустить debugger, проверить state
   ↓
5. Если непонятно — call stack, watch, conditional breakpoint
   ↓
6. Найти root cause (не симптом)
   ↓
7. Fix
   ↓
8. Написать regression test
   ↓
9. Verify fix + test pass
```

### 20.3. Logging discipline

- **`LogTrace`** — для очень детальной диагностики (отключено в production).
- **`LogDebug`** — для отладки разработки (отключено в production).
- **`LogInformation`** — для бизнес-events (старт/завершение операций, успешные действия).
- **`LogWarning`** — для подозрительного, но не error (retry, fallback).
- **`LogError`** — обязательно с **exception** объектом и context.
- **`LogCritical`** — для catastrophic failures, требует immediate action.
- **Structured arguments** всегда: `{UserId}` вместо string concat.
- **Correlation ID** в каждом request scope через `BeginScope`.
- **`{@Object}`** для serialize complex types (Serilog destructuring).

### 20.4. Debugging hygiene

- **`[DebuggerDisplay]` для своих типов** — экономит часы.
- **`Debug.Assert` для invariants** — bug ловится в момент, а не позже.
- **Symbols встроены в DLL** (`<DebugType>embedded</DebugType>`) — production-дампы анализируемы.
- **Source Link для библиотек** — клиенты могут отлаживать твой код.
- **Health checks + readiness probes** — Kubernetes сам отрабатывает unhealthy state.
- **Cancellation tokens до самого низа** — debugger видит cancel paths.

### 20.5. Production readiness

- `dotnet-counters` baseline собирать с первого дня в production.
- APM platform с distributed tracing (OpenTelemetry — vendor-neutral).
- Crash dumps включены через `DOTNET_DbgEnableMiniDump`.
- `dotnet-monitor` или альтернатива sidecar для on-demand artifacts.
- Feature flags для безопасного rollout.
- Runbooks: «high CPU → шаги расследования» документированы.

---

## 21. Decision tree

```
Какой подход к отладке?
│
├── Локальная разработка?
│   ├── Reproducible bug — debugger (F9, F5, F10)
│   ├── Сложно воспроизвести — conditional breakpoint
│   ├── Глубоко в стеке — Call Stack window
│   ├── Async — Tasks window, Parallel Stacks
│   ├── LINQ chain — разбить на строки или LINQ Visualizer
│   └── Логические ошибки — Watch + Immediate Window
│
├── Production?
│   ├── Crash / exception — structured logging + correlation ID
│   ├── Slow performance — dotnet-counters → dotnet-trace
│   ├── Memory leak — dotnet-gcdump comparisons
│   ├── Hang / deadlock — dotnet-dump + clrstack -all
│   ├── Distributed bug — OpenTelemetry / APM
│   └── Need diagnostics on demand — dotnet-monitor sidecar
│
├── Bug непонятен?
│   ├── Сначала reproduce — write failing test
│   ├── Затем binary search — half code suspect, проверить middle
│   └── Read stack trace carefully — обычно говорит почти всё
│
└── Async проблема?
    ├── Tasks window — какие задачи активны
    ├── Parallel Stacks — как они связаны
    ├── ConfigureAwait(false) в библиотечном коде
    └── Avoid .Result / .Wait() — async всю цепочку
```

---

## 22. Cheat sheet

| Hotkey (VS) | VS Code | Rider | Действие |
|-------------|---------|-------|----------|
| `F9` | `F9` | `Ctrl+F8` | Toggle breakpoint |
| `F5` | `F5` | `F5` | Start / Continue |
| `F10` | `F10` | `F10` | Step Over |
| `F11` | `F11` | `F11` | Step Into |
| `Shift+F11` | `Shift+F11` | `Shift+F11` | Step Out |
| `Ctrl+F10` | — | `Alt+F9` | Run to Cursor |
| `Ctrl+Shift+F5` | `Ctrl+Shift+F5` | `Ctrl+Shift+F5` | Restart |
| `Shift+F5` | `Shift+F5` | `Ctrl+F2` | Stop |
| `Ctrl+Alt+W, 1` | View → Watch | `Alt+5` | Watch window |
| `Ctrl+Alt+I` | Debug Console | `Alt+8` | Immediate Window |
| `Ctrl+Alt+C` | Call Stack panel | `Alt+7` | Call Stack |
| `Ctrl+Alt+E` | breakpoints panel | `Ctrl+Alt+B` | Exception Settings |

| CLI tool | Назначение |
|----------|-----------|
| `dotnet-counters monitor -p PID` | Live мониторинг counter'ов |
| `dotnet-trace collect -p PID` | Sampling profiler trace |
| `dotnet-trace convert --format speedscope` | Convert для speedscope.app |
| `dotnet-dump collect -p PID` | Full memory dump |
| `dotnet-dump analyze dump.dmp` | Интерактивный анализ |
| `dotnet-gcdump collect -p PID` | Snapshot managed heap |
| `dotnet-stack report -p PID` | Stack traces всех потоков |
| `dotnet-monitor collect` | Sidecar REST API |
| `dotnet-symbol --symbols X.dll` | Скачать PDB |

| Сценарий | Решение |
|----------|---------|
| Variable value | Hover или Watch |
| Quick eval | Immediate Window |
| Bug в определённом случае | Conditional breakpoint |
| Bug глубоко в stack | Call Stack |
| Custom display | `[DebuggerDisplay]` |
| Production debugging | `ILogger` + correlation ID |
| Production performance | `dotnet-trace` + speedscope |
| Production memory leak | `dotnet-gcdump` snapshots |
| Production hang | `dotnet-dump` + SOS clrstack |
| Async deadlock | Tasks window + replace .Result |
| Edit code на лету | `dotnet watch run` (Hot Reload) |

---

## 23. Practice — упражнения с разбором

### 23.1. Off-by-one bug в LINQ

**Задача.** Метод возвращает топ-N элементов, но один лишний.

```csharp
public List<int> TopN(int[] arr, int n)
{
    return arr.OrderByDescending(x => x).Take(n + 1).ToList();
    //                                       ^^^^^^^ bug
}
```

**Как находить через debugger:**
1. Breakpoint на `return`.
2. Watch `arr.Length`, `n`, `arr.OrderByDescending(x => x).Take(n).Count()`, `arr.OrderByDescending(x => x).Take(n + 1).Count()`.
3. Видишь, что `Take(n + 1)` даёт на 1 больше — bug в формуле.

**Разбор:** Watch с произвольными expressions ловит off-by-one за 30 секунд без модификации кода. Альтернатива через `Console.WriteLine` потребовала бы добавить три вывода и перекомпилировать.

### 23.2. NullReferenceException — какое именно поле null

**Задача.** Throws NRE, stack trace указывает на строку:

```csharp
public string Format(User user) =>
    $"{user.Name} <{user.Email.Trim()}> [{user.Address.City}]";
```

**Как находить:**
1. Breakpoint на этой строке.
2. F5, останавливается до выполнения.
3. Hover на `user` — видишь все поля.
4. `user.Email = null` или `user.Address = null` — сразу понятно.
5. Дальше — поднимаемся по Call Stack: где user был создан без Email/Address?

**Разбор:** debugger показывает точно, какое поле null. Stack trace из exception только указывает строку, но не подсвечивает виновника.

### 23.3. Deadlock — async код висит

**Задача.** Метод не возвращается, приложение зависло:

```csharp
public string LoadData() => LoadDataAsync().Result;

public async Task<string> LoadDataAsync()
{
    var http = new HttpClient();
    return await http.GetStringAsync("https://example.com");
}
```

**Как находить:**
1. Pause execution (Debug → Break All).
2. Tasks window: видишь Task в `WaitingForActivation`, continuation pending.
3. Threads window: main thread в `WaitSleepJoin`.
4. Это deadlock от `.Result`.

**Fix:** async всю цепочку, либо `.GetAwaiter().GetResult()` если действительно нужен sync entry-point (но всё равно опасно).

### 23.4. Memory leak — найти причину

**Задача.** Приложение OOMKilled через 6 часов работы.

**Workflow:**
1. `dotnet-counters monitor -p PID --counters System.Runtime` — gc-heap-size растёт неконтролируемо.
2. `dotnet-gcdump collect -p PID -o snap1.gcdump`.
3. Подождать 30 минут.
4. `dotnet-gcdump collect -p PID -o snap2.gcdump`.
5. Открыть оба в Visual Studio, View → Compare.
6. Top-grow: `EventHandler<OrderEvent>` +50,000 instances.
7. Trace retention path: каждый Subscriber подписывается, но не отписывается.

**Fix:** добавить `IDisposable` в Subscriber с unsubscribe, или использовать `WeakEventManager`.

### 23.5. Production — slow endpoint

**Задача.** API endpoint p99 latency 5 секунд (baseline 200ms).

**Workflow:**
1. APM (Datadog) — посмотреть distributed trace одного из медленных запросов.
2. Видишь, что 4.5 секунды — в одном downstream call к `/users/{id}/orders`.
3. SSH в pod этого сервиса.
4. `dotnet-counters monitor -p 1` — CPU 30%, GC 5%, ничего критичного.
5. `dotnet-trace collect -p 1 --duration 00:00:30 -o trace.nettrace`.
6. `dotnet-trace convert trace.nettrace --format speedscope`.
7. Открываешь в speedscope.app — flame graph показывает 80% времени в `EntityFrameworkCore.Query` методах.
8. Включаешь EF Core logging → видишь N+1 запросы.
9. Fix через `Include(...)` или projection в DTO.

**Разбор:** production-debugging — это многослойный workflow. APM → counters → trace → app-specific tools (EF logging). Каждый слой сужает search space.

---

## 24. Что читать дальше — порядок и почему

1. **[[csharp-basics|C# Basics]]** — без основ языка debugger мало помогает.
2. **[[error-handling|Error Handling]]** — try/catch/finally, exception types, when clauses.
3. **[[testing-fundamentals|Testing Fundamentals]]** — xUnit/NUnit/MSTest, regression tests из debug-сессии.
4. **Logging & Observability** — Serilog, structured logging, OpenTelemetry.
5. **EF Core Debugging** — query logging, change tracker inspection.
6. **Performance & Profiling** — BenchmarkDotNet, allocation analysis, hot path optimization.
7. **Memory Management** — GC generations, large object heap, weak references.
8. **Async Deep Dive** — async state machine, ConfigureAwait, Task vs ValueTask.
9. **Diagnostics in Kubernetes** — dotnet-monitor production setup, alerting rules.

---

## 25. См. также

- [[csharp-basics|C# Basics]] — основы языка
- [[error-handling|Error Handling]] — try/catch + log
- [[dotnet-cli-getting-started|.NET CLI]] — все CLI tools установлены через dotnet tool
- [[testing-fundamentals|Testing]] — regression tests
- Logging & Observability — production logging
- Performance Tools — BenchmarkDotNet, dotTrace
- Memory Profiling — dotMemory, dotnet-gcdump
- Async Deep Dive — state machines, deadlocks
- Native AOT — debugging особенности (limited reflection)

---

## 26. Reading list

- **Microsoft Docs — Debug in Visual Studio** — learn.microsoft.com/visualstudio/debugger
- **Microsoft Docs — .NET Diagnostics tools** — learn.microsoft.com/dotnet/core/diagnostics
- **Microsoft Docs — Logging in .NET** — learn.microsoft.com/dotnet/core/extensions/logging
- **Microsoft Docs — DiagnosticSource users guide** — learn.microsoft.com/dotnet/core/diagnostics/diagnosticsource-diagnosticlistener
- **Microsoft Docs — System.Diagnostics.Metrics** — learn.microsoft.com/dotnet/core/diagnostics/metrics-instrumentation
- **Tess Ferrandez blog** — debugging blog (классика по managed debugging)
- **Sasha Goldstein — Pro .NET Performance** (книга)
- **Konrad Kokosa — Pro .NET Memory Management** (deep memory debugging)
- **Adam Sitnik blog** — adamsitnik.com (BenchmarkDotNet, perf)
- **Christophe Nasarre — speakerdeck.com/chnasarre** (CLR diagnostics)
- **PerfView documentation** — github.com/Microsoft/perfview
- **Speedscope flame graph viewer** — speedscope.app
- **OpenTelemetry .NET** — opentelemetry.io/docs/instrumentation/net
- **Datadog APM .NET** — docs.datadoghq.com/tracing/setup/dotnet-core
- **JetBrains dotMemory tutorials** — jetbrains.com/help/dotmemory
- **SOS commands reference** — learn.microsoft.com/dotnet/core/diagnostics/sos-debugging-extension
