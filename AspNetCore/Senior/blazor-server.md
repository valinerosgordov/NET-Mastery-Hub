---
tags: [blazor, blazor-server, signalr, render-modes, dotnet-10, mudblazor]
level: Senior
---

# Blazor Server — production guide

## Что это, зачем и когда

### Что такое Blazor?
**Web UI framework на C#/.NET вместо JavaScript.** Компоненты пишутся на C# + Razor (`.razor` файлы), компилируются в .NET, рендерятся как HTML. Три режима исполнения:

- **Blazor Server** — компоненты живут на сервере, UI обновляется через SignalR (WebSocket). Один процесс — много пользователей.
- **Blazor WebAssembly** — .NET-runtime скачивается в браузер, всё работает на клиенте.
- **Blazor United / InteractiveAuto (.NET 8+)** — гибрид: первый рендер SSR, потом WebAssembly или Server по выбору.

**Аналогия:** ASP.NET MVC + Razor — это статичный сайт с раз в N секунд POST-back'ом. SPA на React — приложение в браузере с клиентским state. Blazor Server — это «сервер симулирует SPA через WebSocket-канал»: каждое нажатие отправляется на сервер, тот пересчитывает компоненты и шлёт обратно diff.

### Когда какой выбрать

| | Blazor Server | Blazor WebAssembly | InteractiveAuto |
|--|---------------|---------------------|-----------------|
| Загрузка | Мгновенная (HTML) | Долгая (3-10 МБ runtime + DLL) | Мгновенная, потом upgrade |
| Offline-режим | Нет (нужно соединение) | Да | Зависит от страницы |
| Latency UI | Round-trip к серверу | Локальный | Локальный после upgrade |
| Нагрузка на сервер | Высокая (circuit per user) | Минимальная (статика) | Постепенно снижается |
| SEO | Из коробки | Сложнее (prerendering) | Из коробки (SSR baseline) |
| Native API в браузере | Через JS interop | Через JS interop | По обстоятельствам |
| Доступ к серверу | Нативный (DI инжектит сервисы) | Только через HTTP API | Зависит от режима страницы |

### Когда применять Blazor Server

| Идеально | Не подходит |
|----------|-------------|
| Внутренние инструменты, B2B-дашборды | Публичный сайт миллион MAU |
| Real-time сценарии (chat, dashboard, monitoring) | Mobile apps offline |
| Команда пишет на C#, не хочет JS-стек | Жесткие требования по latency UI |
| Тяжёлая бизнес-логика рядом с DB | Интернационально с большим RTT |

NexusAI выбрал Blazor Server для интерактивного редактирования презентаций — UI рядом с генератором, low-latency editing, SignalR держит сессию.

---

## Архитектура Blazor Server — circuit и SignalR

```
Browser                                     Server (Kestrel)
┌─────────────────┐  WebSocket (SignalR)   ┌─────────────────┐
│  HTML/CSS/JS    │ ◀────────────────────▶ │  Circuit (per   │
│  blazor.server  │   События UI →          │  connection)    │
│   .js           │   ← DOM diff            │                 │
└─────────────────┘                         │  ┌──────────┐   │
                                            │  │Components│   │
                                            │  │  state   │   │
                                            │  └──────────┘   │
                                            │  ┌──────────┐   │
                                            │  │Scoped DI │   │
                                            │  └──────────┘   │
                                            └─────────────────┘
```

**Circuit** — это "сессия" пользователя на сервере. Хранит state всех компонентов, scoped DI services, AuthenticationState. Живёт пока WebSocket открыт + retention period (по дефолту 3 минуты после disconnect — попытка восстановить).

**SignalR** — транспорт. Каждое событие (click, input change) — это сообщение. Сервер пересчитывает render tree и шлёт **diff DOM** (не весь HTML).

### Render Modes (.NET 8+)

```razor
@* Глобально (Program.cs) — серверный рендер по дефолту с интерактивностью на странице *@

@page "/dashboard"
@rendermode InteractiveServer

<h1>Dashboard</h1>
<button @onclick="Refresh">Refresh</button>
@code {
    private void Refresh() { /* интерактивно — выполняется на сервере, UI обновляется через circuit */ }
}
```

Опции `@rendermode`:
- `InteractiveServer` — Blazor Server, circuit
- `InteractiveWebAssembly` — компонент работает в браузере
- `InteractiveAuto` — первый рендер Server (быстрый старт), потом auto-upgrade на WebAssembly когда .NET runtime загружен
- `(не указан)` — Static SSR, без интерактивности (server-side render and forget)

```razor
@* Per-component render mode *@
<Counter @rendermode="InteractiveServer" />
<UserProfile @rendermode="InteractiveAuto" />
```

### Enhanced navigation

```razor
@* Standard <a href> в Blazor (.NET 8+) — без полной перезагрузки страницы *@
<a href="/about">About</a>

@* Можно отключить для конкретной ссылки *@
<a href="/legacy" data-enhance-nav="false">Legacy</a>
```

Enhanced navigation работает как SPA: смена URL без full page reload, plain HTML diff'ится с DOM. Без JS-фреймворка — встроено в .NET 8+.

> [!question]- **Интервью: чем `InteractiveServer` отличается от `InteractiveAuto`?**
> `InteractiveServer` — компонент **всегда** работает на сервере через circuit. Низкая latency (нужен round-trip), нагрузка на сервер пропорциональна пользователям.
>
> `InteractiveAuto` — first render через сервер (быстрый старт без скачивания WASM), параллельно скачивается WebAssembly runtime, при следующем визите страницы — компонент работает локально в браузере. Idea: пользователь не ждёт ни первый render (быстрый Server), ни subsequent visits (локальный WASM).
>
> Trade-off: WASM-режим не имеет доступа к scoped serverside services (DI), нужен HTTP API для данных.

---

## Component model

### Параметры

```razor
@* TaskCard.razor *@
<div class="task-card">
    <h3>@Task.Title</h3>
    <button @onclick="OnComplete">Complete</button>
</div>

@code {
    [Parameter, EditorRequired]
    public TaskDto Task { get; set; } = default!;

    [Parameter]
    public EventCallback<Guid> OnTaskCompleted { get; set; }

    private async Task OnComplete()
    {
        await OnTaskCompleted.InvokeAsync(Task.Id);
    }
}
```

```razor
@* Использование *@
<TaskCard Task="myTask" OnTaskCompleted="HandleCompleted" />
```

`[EditorRequired]` — IDE подскажет если не передал параметр.
`EventCallback<T>` (не `Action<T>`) — автоматически вызывает `StateHasChanged()` родителя.

### Lifecycle

```csharp
@code {
    [Parameter] public Guid TaskId { get; set; }
    [Inject] private ITaskService TaskService { get; set; } = default!;

    private TaskDto? _task;

    protected override async Task OnInitializedAsync()
    {
        // Вызывается раз при создании компонента
        _task = await TaskService.GetAsync(TaskId);
    }

    protected override async Task OnParametersSetAsync()
    {
        // Вызывается каждый раз когда параметры меняются (включая первый раз)
        // Используется когда нужно перезагрузить данные при смене параметров
        if (TaskId != _task?.Id)
            _task = await TaskService.GetAsync(TaskId);
    }

    protected override void OnAfterRender(bool firstRender)
    {
        // После каждого рендера — для JS-interop
        if (firstRender)
        {
            // ...
        }
    }

    public void Dispose()
    {
        // Если компонент IDisposable — освобождаем подписки
        _subscription?.Dispose();
    }
}
```

### `@key` для коллекций

```razor
@* ❌ BAD — Blazor может перепутать какой компонент к какой строке относится *@
@foreach (var task in tasks)
{
    <TaskCard Task="task" />
}

@* ✅ GOOD — @key привязывает компонент к конкретному элементу *@
@foreach (var task in tasks)
{
    <TaskCard @key="task.Id" Task="task" />
}
```

Без `@key` при удалении/перемешивании элементов Blazor может переиспользовать компоненты неправильно (state одного перетекает в другой).

### `@ref` — ссылка на компонент или DOM-элемент

```razor
<input @ref="_inputRef" />
<button @onclick="FocusInput">Focus</button>

@code {
    private ElementReference _inputRef;

    private async Task FocusInput()
    {
        await _inputRef.FocusAsync();
    }
}
```

```razor
<MyChild @ref="_child" />
<button @onclick="CallChild">Call child method</button>

@code {
    private MyChild? _child;

    private void CallChild() => _child?.DoSomething();
}
```

---

## State management

### Per-component state (тривиальный)
Просто поля в `@code` блоке. Очищаются когда компонент уничтожен.

### Per-circuit state (singleton scope)

```csharp
// Сервис (Scoped в Blazor Server = per-circuit = per-user)
public sealed class UserPreferences
{
    public string Theme { get; set; } = "Dark";
    public int FontSize { get; set; } = 14;
    public event Action? OnChange;

    public void NotifyChange() => OnChange?.Invoke();
}

// Program.cs
builder.Services.AddScoped<UserPreferences>();
```

```razor
@* Theme.razor *@
@inject UserPreferences Prefs
@implements IDisposable

<select @bind="Prefs.Theme" @bind:after="OnThemeChanged">
    <option>Dark</option>
    <option>Light</option>
</select>

@code {
    protected override void OnInitialized()
    {
        Prefs.OnChange += StateHasChanged;
    }

    private void OnThemeChanged()
    {
        Prefs.NotifyChange();  // другие компоненты получают сигнал
    }

    public void Dispose() => Prefs.OnChange -= StateHasChanged;
}
```

### Persistent storage

```csharp
@inject ProtectedLocalStorage Storage

@code {
    private string? _theme;

    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            var result = await Storage.GetAsync<string>("theme");
            _theme = result.Success ? result.Value : "Dark";
            StateHasChanged();
        }
    }

    private async Task SaveThemeAsync(string theme)
    {
        await Storage.SetAsync("theme", theme);
    }
}
```

`ProtectedLocalStorage` / `ProtectedSessionStorage` — обёртка с шифрованием через DataProtection. **Доступна только после первого рендера** — не вызывай в `OnInitializedAsync` (там ещё нет JS-interop'а в Blazor Server).

### State across navigation

```csharp
// Cascading values (через всё дерево вниз)
<CascadingValue Value="@_currentUser">
    <Router AppAssembly="@typeof(App).Assembly">
        ...
    </Router>
</CascadingValue>

@code {
    private UserContext _currentUser = new();
}

// В дочерних компонентах
@code {
    [CascadingParameter]
    public UserContext User { get; set; } = default!;
}
```

> [!question]- **Интервью: какие подводные камни state management в Blazor Server?**
> 1. **State leak между пользователями** — если зарегистрировал сервис как Singleton (а не Scoped), весь state делится. Помни: Scoped в Blazor Server = per-circuit = per-user, Singleton = глобальный.
> 2. **DbContext в Blazor Server** — ОБЯЗАТЕЛЬНО `AddDbContextFactory` + `using var ctx = factory.CreateDbContext()`, **не** scoped DbContext. Иначе одна и та же DbContext инстанция используется параллельными асинхронными вызовами в одном circuit → "DbContext is not thread-safe".
> 3. **Memory leak через event-handlers** — компонент подписался на `service.OnChange += StateHasChanged`, не отписался в `Dispose`. State хранится дальше, GC не может собрать компонент.
> 4. **`StateHasChanged` из background thread** — через `await InvokeAsync(StateHasChanged)`. Просто `StateHasChanged()` не из render-thread'а кидает `InvalidOperationException`.

---

## Forms и валидация

```razor
@page "/tasks/create"
@inject ITaskService TaskService
@inject NavigationManager Nav

<EditForm Model="@_model" OnValidSubmit="@HandleSubmit" FormName="CreateTask">
    <DataAnnotationsValidator />
    <ValidationSummary />

    <div>
        <label>Title</label>
        <InputText @bind-Value="_model.Title" />
        <ValidationMessage For="() => _model.Title" />
    </div>

    <div>
        <label>Due date</label>
        <InputDate @bind-Value="_model.DueDate" />
    </div>

    <div>
        <label>Priority</label>
        <InputSelect @bind-Value="_model.Priority">
            <option value="Low">Low</option>
            <option value="Medium">Medium</option>
            <option value="High">High</option>
        </InputSelect>
    </div>

    <button type="submit">Create</button>
</EditForm>

@code {
    [SupplyParameterFromForm]
    public TaskModel _model { get; set; } = new();

    private async Task HandleSubmit()
    {
        await TaskService.CreateAsync(_model);
        Nav.NavigateTo("/tasks");
    }

    public sealed class TaskModel
    {
        [Required, MinLength(3), MaxLength(100)]
        public string Title { get; set; } = "";

        [Required]
        public DateTime DueDate { get; set; } = DateTime.Today.AddDays(1);

        [Required]
        public string Priority { get; set; } = "Medium";
    }
}
```

`[SupplyParameterFromForm]` (.NET 8+) — позволяет SSR-форме (без circuit) сабмитить данные через стандартный POST. Прогрессивное улучшение: работает даже без JS.

### Кастомная валидация

```csharp
public sealed class TaskModelValidator : AbstractValidator<TaskModel>
{
    public TaskModelValidator(ITaskService service)
    {
        RuleFor(x => x.Title)
            .NotEmpty()
            .MaximumLength(100)
            .MustAsync(async (title, ct) => !await service.TitleExistsAsync(title, ct))
            .WithMessage("Title already exists");
    }
}

// EditForm с FluentValidation (через пакет Blazored.FluentValidation)
<EditForm Model="@_model" OnValidSubmit="@HandleSubmit">
    <FluentValidationValidator />
    ...
</EditForm>
```

---

## Authentication и Authorization

### Setup

```csharp
// Program.cs
builder.Services.AddAuthentication(IdentityConstants.ApplicationScheme)
    .AddCookie(IdentityConstants.ApplicationScheme);

builder.Services.AddAuthorization();
builder.Services.AddCascadingAuthenticationState();

// Middleware order
app.UseAuthentication();
app.UseAuthorization();
app.UseAntiforgery();
app.MapRazorComponents<App>()
    .AddInteractiveServerRenderMode();
```

### `AuthorizeView`

```razor
<AuthorizeView>
    <Authorized>
        Hello, @context.User.Identity?.Name!
        <LogoutButton />
    </Authorized>
    <NotAuthorized>
        <a href="/login">Sign in</a>
    </NotAuthorized>
</AuthorizeView>

<AuthorizeView Roles="Admin,Manager">
    <button>Admin panel</button>
</AuthorizeView>

<AuthorizeView Policy="MinimumAge18">
    <ContentForAdults />
</AuthorizeView>
```

### `[Authorize]` на странице

```razor
@page "/admin"
@attribute [Authorize(Roles = "Admin")]

<h1>Admin Panel</h1>
```

Если не авторизован — редирект на login (настраивается в `App.razor`):
```razor
<Router AppAssembly="@typeof(App).Assembly">
    <Found Context="routeData">
        <AuthorizeRouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)">
            <NotAuthorized>
                <RedirectToLogin />
            </NotAuthorized>
        </AuthorizeRouteView>
    </Found>
</Router>
```

### Доступ к ClaimsPrincipal в коде

```razor
@inject AuthenticationStateProvider AuthProvider

@code {
    private string? _userName;

    protected override async Task OnInitializedAsync()
    {
        var state = await AuthProvider.GetAuthenticationStateAsync();
        _userName = state.User.Identity?.Name;
    }
}
```

---

## Real-time updates

### `StateHasChanged` из background

```csharp
private async Task StartBackgroundUpdatesAsync(CancellationToken ct)
{
    await foreach (var update in _service.SubscribeAsync(ct))
    {
        // Из background thread — нужен InvokeAsync
        await InvokeAsync(() =>
        {
            _items.Add(update);
            StateHasChanged();
        });
    }
}
```

### `IAsyncEnumerable` стримы прямо в UI

```razor
@inject IChatService Chat

<div>@_streamedAnswer</div>
<button @onclick="StartStream">Ask</button>

@code {
    private string _streamedAnswer = "";

    private async Task StartStream()
    {
        _streamedAnswer = "";
        await foreach (var token in Chat.StreamAnswerAsync(_question, CancellationToken.None))
        {
            _streamedAnswer += token;
            await InvokeAsync(StateHasChanged);
        }
    }
}
```

См. [LLM / RAG patterns]() — как сделать streaming endpoint, который отдаёт `IAsyncEnumerable`.

### Прямой SignalR hub без circuit

Если нужен real-time без circuit (например, для public dashboard без логина), можешь использовать `HubConnection` напрямую:

```csharp
@inject NavigationManager Nav
@implements IAsyncDisposable

@code {
    private HubConnection? _hub;

    protected override async Task OnInitializedAsync()
    {
        _hub = new HubConnectionBuilder()
            .WithUrl(Nav.ToAbsoluteUri("/marketdatahub"))
            .WithAutomaticReconnect()
            .Build();

        _hub.On<MarketTick>("OnTick", async tick =>
        {
            _ticks.Add(tick);
            await InvokeAsync(StateHasChanged);
        });

        await _hub.StartAsync();
    }

    public async ValueTask DisposeAsync()
    {
        if (_hub is not null) await _hub.DisposeAsync();
    }
}
```

---

## Performance

### Virtualization

```razor
@* Без виртуализации — 10 000 элементов = тормоза *@
<div>
    @foreach (var item in _items)
    {
        <ItemCard Item="item" />
    }
</div>

@* С виртуализацией — рендерятся только видимые *@
<Virtualize Items="_items" Context="item" ItemSize="80">
    <ItemCard Item="item" />
</Virtualize>

@* С lazy loading *@
<Virtualize @ref="_virtualizeRef" ItemsProvider="LoadItemsAsync" Context="item" ItemSize="80">
    <ItemCard Item="item" />
    <Placeholder>
        <SkeletonRow />
    </Placeholder>
</Virtualize>

@code {
    private async ValueTask<ItemsProviderResult<TaskDto>> LoadItemsAsync(ItemsProviderRequest req)
    {
        var (items, total) = await _service.GetPagedAsync(
            skip: req.StartIndex, take: req.Count, ct: req.CancellationToken);
        return new ItemsProviderResult<TaskDto>(items, totalItemCount: total);
    }
}
```

`Virtualize` рендерит только видимые элементы + buffer сверху/снизу. Включай для любых списков > 100 элементов.

### Lazy load assemblies (WebAssembly only)

```xml
<ItemGroup>
    <BlazorWebAssemblyLazyLoad Include="HeavyFeature.dll" />
</ItemGroup>
```

```csharp
@inject LazyAssemblyLoader Loader

private async Task LoadFeatureAsync()
{
    var assemblies = await Loader.LoadAssembliesAsync(["HeavyFeature.dll"]);
    // теперь можно навигировать на страницу из этой сборки
}
```

### `ShouldRender` override

```csharp
@code {
    protected override bool ShouldRender()
    {
        // Не пересчитывать DOM если ничего важного не изменилось
        return _hasChanges;
    }
}
```

Тонкая оптимизация для часто-обновляющихся компонентов.

### Минимизация circuit memory

Каждый circuit = state всех компонентов + scoped services + render tree. На 10K пользователей это GB памяти. Стратегии:
- **Закрывать неактивные circuits** — `app.UseCircuitOptions(opt => opt.DetailedErrors = false; opt.DisconnectedCircuitMaxRetained = 100;)` — лимит retained circuits
- **Disposable everything** — компоненты `IDisposable` обязательно отписываются
- **Не хранить большие коллекции в state** — используй pagination, не загружай 100K записей в память

---

## Testing с bUnit

```bash
dotnet add package bunit
```

```csharp
public class TaskCardTests : TestContext
{
    [Fact]
    public async Task Click_Complete_Triggers_Callback()
    {
        // Arrange
        var task = new TaskDto(Guid.NewGuid(), "Test task", false);
        var clicked = false;
        Services.AddSingleton<ITaskService>(new FakeService());

        // Act
        var cut = RenderComponent<TaskCard>(p => p
            .Add(c => c.Task, task)
            .Add(c => c.OnTaskCompleted, EventCallback.Factory.Create<Guid>(this, _ => clicked = true)));

        await cut.Find("button").ClickAsync(new MouseEventArgs());

        // Assert
        clicked.ShouldBeTrue();
    }

    [Fact]
    public void Renders_Title()
    {
        var task = new TaskDto(Guid.NewGuid(), "My title", false);
        var cut = RenderComponent<TaskCard>(p => p.Add(c => c.Task, task));
        cut.Find("h3").TextContent.ShouldBe("My title");
    }
}
```

---

## Deployment

### Sticky sessions (LB конфиг)

Каждый пользователь привязан к конкретному инстансу через circuit. Load balancer должен **гарантированно** держать пользователя на одном сервере (sticky sessions / session affinity).

**Nginx:**
```nginx
upstream blazor_backend {
    ip_hash;  # или
    # hash $remote_addr consistent;
    server backend1:8080;
    server backend2:8080;
}

server {
    location / {
        proxy_pass http://blazor_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_read_timeout 86400;  # WebSocket — длинное соединение
    }
}
```

### Scale out — Azure SignalR / Redis backplane

Когда инстансов много — circuit на одном сервере, но pub-sub между серверами через Azure SignalR Service:
```csharp
builder.Services.AddSignalR().AddAzureSignalR(connectionString);
```

Или Redis backplane для open-source:
```csharp
builder.Services.AddSignalR().AddStackExchangeRedis("localhost:6379");
```

Любые `Clients.All`-сообщения приходят на все инстансы → доходят до правильных пользователей.

### Circuit retention

```csharp
builder.Services.Configure<CircuitOptions>(options =>
{
    options.DisconnectedCircuitRetentionPeriod = TimeSpan.FromMinutes(3);
    options.DisconnectedCircuitMaxRetained = 100;
    options.JSInteropDefaultCallTimeout = TimeSpan.FromMinutes(1);
    options.MaxBufferedUnacknowledgedRenderBatches = 10;
});
```

`DisconnectedCircuitRetentionPeriod` — сколько держать сессию после WebSocket разрыва (например, обновление страницы). Большее значение = лучше UX, больше памяти.

---

## MudBlazor — UI library

`MudBlazor` — Material Design компоненты для Blazor. В NexusAI зафиксирован 9.3.x.

```bash
dotnet add package MudBlazor
```

```razor
@* Program.cs *@
builder.Services.AddMudServices();

@* App.razor / MainLayout.razor *@
<MudThemeProvider Theme="@_theme" IsDarkMode="true" />
<MudPopoverProvider />
<MudDialogProvider />
<MudSnackbarProvider />

<MudLayout>
    <MudAppBar>
        <MudText Typo="Typo.h5">NexusAI</MudText>
    </MudAppBar>
    <MudDrawer Open="true">
        <MudNavMenu>
            <MudNavLink Href="/" Icon="@Icons.Material.Filled.Home">Home</MudNavLink>
            <MudNavLink Href="/decks" Icon="@Icons.Material.Filled.Slideshow">Decks</MudNavLink>
        </MudNavMenu>
    </MudDrawer>
    <MudMainContent>
        @Body
    </MudMainContent>
</MudLayout>
```

Альтернативы:
- **Radzen.Blazor** — большой набор enterprise-компонентов
- **AntDesign.Blazor** — стиль Ant Design
- **FluentUI Blazor** — Microsoft Fluent (Win11 style)

---

## Common pitfalls

### 1. DbContext scoped в Blazor Server

```csharp
// ❌ BAD — один DbContext per circuit, но параллельные async-запросы из компонентов конфликтуют
builder.Services.AddDbContext<AppDbContext>(...);

// ✅ GOOD — Factory, создаём DbContext per operation
builder.Services.AddDbContextFactory<AppDbContext>(...);

// В компоненте/сервисе
@inject IDbContextFactory<AppDbContext> Factory

private async Task LoadAsync()
{
    using var db = await Factory.CreateDbContextAsync();
    _items = await db.Tasks.ToListAsync();
}
```

### 2. JS-interop в `OnInitializedAsync`

```razor
@* ❌ BAD — JSRuntime недоступен в OnInitialized в Blazor Server (prerender phase) *@
protected override async Task OnInitializedAsync()
{
    await JS.InvokeVoidAsync("alert", "hi");  // throws InvalidOperationException
}

@* ✅ GOOD — в OnAfterRenderAsync *@
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender) await JS.InvokeVoidAsync("alert", "hi");
}
```

### 3. CascadingParameter без CascadingValue в дереве
Если `[CascadingParameter]` где-то в компоненте, но в дереве компонентов нет `<CascadingValue>` — параметр будет null, без warning.

### 4. `await Task.Yield()` или `await Task.Delay(1)` для force-render
Хак который использовали для немедленной отрисовки. В .NET 8+ — `StateHasChanged()` достаточно.

### 5. SSR-страница использует interactive component

```razor
@* Страница — Static SSR (без render mode) *@
@page "/mypage"
<Counter @rendermode="InteractiveServer" />  @* OK — компонент интерактивен *@
```
Но обратное (interactive страница содержит SSR-only компонент) даёт пустоту: компонент рендерится один раз и больше не обновляется.

### 6. EventCallback из родителя в child через `Action<T>`

```razor
@* ❌ Action<T> не вызывает StateHasChanged родителя *@
<MyChild OnSomething="@(x => DoIt(x))" />

@code {
    [Parameter] public Action<int> OnSomething { get; set; } = _ => { };
}

@* ✅ EventCallback — родитель re-renders автоматически *@
[Parameter] public EventCallback<int> OnSomething { get; set; }
```

### 7. Утечки через `event Action`

```csharp
// Service.cs
public event Action? OnChange;

// Component.razor (❌ no Dispose unsubscribe)
protected override void OnInitialized() { Service.OnChange += StateHasChanged; }
// circuit умирает, компонент должен быть GC'ed, но Service держит strong ref → leak

// ✅ IDisposable
public void Dispose() { Service.OnChange -= StateHasChanged; }
```

### 8. AntiForgery для form-submit

```csharp
app.UseAntiforgery();
```
Без этого SSR-формы получают `400 Bad Request`. На SPA-only режиме (только interactive) middleware всё равно нужен — `@formname` и `[SupplyParameterFromForm]` его требуют.

---

## См. также

- [LLM / RAG patterns]() — streaming `IAsyncEnumerable` в Blazor UI (NexusAI deck generation)
- [Pipeline и Middleware](pipeline-middleware.md) — порядок middleware для Blazor
- [Auth и Security](auth-security.md) — JWT, claims, policies (применимо к Blazor)
- [Resilience и HttpClient](resilience.md) — Polly для Blazor-клиентов API
- [Performance]() — общие принципы для server-side Blazor
- [Testing]() — bUnit как часть стека

## Reading list

- **Blazor docs** — learn.microsoft.com/aspnet/core/blazor/
- **Blazor University** — blazor-university.com (бесплатный курс)
- **Steve Sanderson's Blog** — blog.stevensanderson.com (создатель Blazor)
- **Awesome Blazor** — github.com/AdrienTorris/awesome-blazor (библиотеки + статьи)
- **MudBlazor docs** — mudblazor.com
- **bUnit docs** — bunit.dev
- **Carl Franklin — Blazor Train** — youtube series, разбор реальных приложений
