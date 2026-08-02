---
tags: [csharp, blazor, webassembly, wasm, spa, frontend, jsinterop]
level: Senior
date: 2026-08-02
---

# Blazor WebAssembly

> C# в браузере через WebAssembly. Альтернатива Blazor Server и React/Angular/Vue. Закрывает: WASM модель, hosting models, AOT compilation, JS interop, performance, lazy loading, PWA, deployment, когда WASM vs Server, comparison с другими SPA frameworks.

---

## Что это, зачем и когда

### Что такое Blazor WebAssembly?

**.NET runtime + твой C# код** компилируются в WebAssembly и **запускаются прямо в браузере**. Не Server-side rendering — настоящий клиент.

**Аналогия:** React/Vue использует JavaScript engine браузера. Blazor WASM — приносит .NET runtime в браузер. Твой C# код запускается **на машине пользователя**, не на сервере.

### Архитектурные модели Blazor

| | Blazor Server | Blazor WASM | InteractiveAuto (.NET 8+) |
|--|---------------|-------------|--------------------------|
| Где работает | Сервер (SignalR) | Браузер (WASM) | Гибрид |
| Initial download | ~5 KB JS | **~2-5 MB** WASM + DLLs | Server first, потом WASM |
| Latency | Network на каждое UI событие | **Локально** (instant) | Адаптивный |
| Server resources | Высокий (per user) | Минимум (static files) | Средний |
| Offline | ❌ | **✅ PWA** | После initial — да |
| Code execution | На сервере (security) | **На клиенте** (виден) | Зависит |
| Scale-out | Sticky sessions | Static files (CDN) | Mixed |
| Когда | Internal apps, low scale | Public apps, mobile | Optimal default (.NET 8+) |

### Зачем выбрать Blazor WASM

✅ **Хорошо:**
- C# вместо JavaScript для full-stack команды
- Reuse domain models между frontend / backend
- Strongly-typed throughout — нет runtime type errors
- LINQ / Records / Pattern matching в браузере
- Offline support (PWA)
- High scale (static files на CDN)
- Same skills для server + client

❌ **Минусы:**
- Большой initial download (~2-5 MB)
- Slower startup чем JS frameworks (~1-3 секунды)
- Dependent на WebAssembly support (все modern browsers OK)
- Не все .NET libraries работают (некоторые depend on file system, threading)
- Code visible client-side — нет secrets

### Когда WASM vs другие frameworks

| | Blazor WASM | React | Vue | Angular |
|--|-------------|-------|-----|---------|
| Language | C# | TypeScript / JS | TypeScript / JS | TypeScript |
| Bundle size | 2-5 MB | 100-500 KB | 100-300 KB | 200-800 KB |
| Startup | 1-3 sec | <1 sec | <1 sec | 1-2 sec |
| Type safety | ✅ Compile + runtime | TypeScript compile-time | TypeScript compile-time | Strong |
| Ecosystem | Smaller | Massive | Large | Large |
| Native feel | Like SPA | SPA | SPA | SPA |
| Mobile | Responsive | Responsive | Responsive | Responsive |
| Learning curve | C# devs — easy | Medium | Easy | Steep |

---

## 1. Setup

### Стандартный проект

```bash
# Standalone WASM
dotnet new blazorwasm -o MyApp

# Hosted (WASM + ASP.NET API)
dotnet new blazorwasm -o MyApp --hosted

# Empty WASM
dotnet new blazorwasm-empty -o MyApp
```

```bash
cd MyApp
dotnet run
# https://localhost:5001

```

### Структура

```
MyApp/
├── Pages/
│   ├── Home.razor
│   ├── Counter.razor
│   └── FetchData.razor
├── Layout/
│   └── MainLayout.razor
├── Shared/
│   └── NavMenu.razor
├── wwwroot/
│   ├── index.html
│   └── css/
├── Program.cs
└── App.razor
```

---

## 2. Razor компоненты

### Базовый компонент

```razor
@* Pages/Counter.razor *@
@page "/counter"

<PageTitle>Counter</PageTitle>

<h1>Counter</h1>

<p role="status">Current count: @currentCount</p>

<button class="btn btn-primary" @onclick="IncrementCount">Click me</button>

@code {
    private int currentCount = 0;

    private void IncrementCount()
    {
        currentCount++;
    }
}
```

### Параметры компонентов

```razor
@* Shared/UserCard.razor *@
<div class="card">
    <div class="card-body">
        <h5>@User.Name</h5>
        <p>@User.Email</p>
        @if (ShowActions)
        {
            <button @onclick="() => OnEdit.InvokeAsync(User)">Edit</button>
            <button @onclick="() => OnDelete.InvokeAsync(User.Id)">Delete</button>
        }
    </div>
</div>

@code {
    [Parameter, EditorRequired]
    public User User { get; set; } = null!;
    
    [Parameter]
    public bool ShowActions { get; set; } = true;
    
    [Parameter]
    public EventCallback<User> OnEdit { get; set; }
    
    [Parameter]
    public EventCallback<int> OnDelete { get; set; }
}
```

### Использование

```razor
<UserCard User="@currentUser" 
          OnEdit="HandleEdit" 
          OnDelete="HandleDelete" 
          ShowActions="@isAdmin" />

@code {
    private User currentUser = new();
    private bool isAdmin = true;
    
    private async Task HandleEdit(User u) { /* ... */ }
    private async Task HandleDelete(int id) { /* ... */ }
}
```

### Lifecycle methods

```razor
@page "/data"
@inject HttpClient Http

@code {
    private List<Order>? orders;
    
    // Вызывается один раз при инициализации
    protected override async Task OnInitializedAsync()
    {
        orders = await Http.GetFromJsonAsync<List<Order>>("api/orders");
    }
    
    // После каждого render
    protected override async Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            // Init JS interop, third-party libs
        }
    }
    
    // Когда параметры изменились
    protected override async Task OnParametersSetAsync()
    {
        // Re-fetch data based on new params
    }
    
    // Контролируемый rerender
    protected override bool ShouldRender() => true;
}
```

---

## 3. Routing

### Page directive

```razor
@page "/users"
@page "/users/{id:int}"

@code {
    [Parameter]
    public int? Id { get; set; }
}
```

### Параметры route

```razor
@page "/products/{category}/{id:int}"

@code {
    [Parameter] public string Category { get; set; } = "";
    [Parameter] public int Id { get; set; }
}
```

### Constraint типы

```
{id:int}       — int только
{id:guid}      — Guid
{date:datetime} — DateTime
{slug:regex(^[a-z0-9-]+$)} — regex pattern
```

### Programmatic navigation

```razor
@inject NavigationManager Navigation

@code {
    private void GoToUser(int id)
    {
        Navigation.NavigateTo($"/users/{id}");
    }
    
    private void GoExternal()
    {
        Navigation.NavigateTo("https://example.com", forceLoad: true);
    }
    
    // Текущий URL
    private string current => Navigation.Uri;
    private string baseUri => Navigation.BaseUri;
    
    // Subscribe на изменения
    protected override void OnInitialized()
    {
        Navigation.LocationChanged += HandleLocationChanged;
    }
    
    private void HandleLocationChanged(object? sender, LocationChangedEventArgs e)
    {
        Console.WriteLine($"Navigated to {e.Location}");
    }
}
```

---

## 4. State management

### Component state — обычные fields

```razor
@code {
    private int count = 0;
    private string name = "";
    
    // StateHasChanged() редко нужно — авто-rerender при event handlers
}
```

### Cascading parameters

```razor
@* MainLayout.razor — share state с child components *@
<CascadingValue Value="@theme">
    <CascadingValue Value="@user">
        @Body
    </CascadingValue>
</CascadingValue>

@code {
    private ThemeInfo theme = new();
    private UserInfo? user;
}

@* Любой child component *@
@code {
    [CascadingParameter] public ThemeInfo Theme { get; set; } = null!;
    [CascadingParameter] public UserInfo? User { get; set; }
}
```

### Singleton service для глобального state

```csharp
// Program.cs
builder.Services.AddSingleton<AppState>();

// AppState.cs
public class AppState
{
    public int CartItemCount { get; private set; }
    
    public event Action? OnChange;
    
    public void AddToCart()
    {
        CartItemCount++;
        OnChange?.Invoke();
    }
}

// Component
@inject AppState State
@implements IDisposable

<p>Items in cart: @State.CartItemCount</p>

@code {
    protected override void OnInitialized()
    {
        State.OnChange += StateHasChanged;
    }
    
    public void Dispose()
    {
        State.OnChange -= StateHasChanged;
    }
}
```

### Fluxor — Redux-style state management

```bash
dotnet add package Fluxor.Blazor.Web
```

```csharp
// State
public record CounterState(int Count);

[FeatureState]
public class CounterFeature : Feature<CounterState>
{
    public override string GetName() => "Counter";
    protected override CounterState GetInitialState() => new(0);
}

// Action
public record IncrementAction;

// Reducer
public static class CounterReducers
{
    [ReducerMethod]
    public static CounterState Reduce(CounterState state, IncrementAction _) =>
        state with { Count = state.Count + 1 };
}

// Use в компоненте
@using Fluxor
@inject IState<CounterState> CounterState
@inject IDispatcher Dispatcher

<p>Count: @CounterState.Value.Count</p>
<button @onclick="Increment">+</button>

@code {
    private void Increment() => Dispatcher.Dispatch(new IncrementAction());
}
```

---

## 5. HTTP / API calls

### Базовый HttpClient

```razor
@page "/orders"
@inject HttpClient Http

@if (orders is null)
{
    <p>Loading...</p>
}
else
{
    <ul>
        @foreach (var order in orders)
        {
            <li>@order.Id - @order.Total</li>
        }
    </ul>
}

@code {
    private List<Order>? orders;
    
    protected override async Task OnInitializedAsync()
    {
        try
        {
            orders = await Http.GetFromJsonAsync<List<Order>>("api/orders");
        }
        catch (HttpRequestException ex)
        {
            Console.Error.WriteLine($"Error: {ex.Message}");
        }
    }
    
    private async Task CreateOrder()
    {
        var newOrder = new Order { Total = 100 };
        var response = await Http.PostAsJsonAsync("api/orders", newOrder);
        
        if (response.IsSuccessStatusCode)
        {
            var created = await response.Content.ReadFromJsonAsync<Order>();
            orders!.Add(created!);
        }
    }
}
```

### Typed HttpClient через DI

```csharp
// Program.cs
builder.Services.AddHttpClient<IOrderApi, OrderApi>(client =>
{
    client.BaseAddress = new Uri(builder.HostEnvironment.BaseAddress);
});

// OrderApi.cs
public interface IOrderApi
{
    Task<List<Order>> GetAllAsync();
    Task<Order?> GetByIdAsync(int id);
    Task<Order> CreateAsync(Order order);
}

public class OrderApi(HttpClient http) : IOrderApi
{
    public async Task<List<Order>> GetAllAsync() =>
        await http.GetFromJsonAsync<List<Order>>("api/orders") ?? [];
    
    public async Task<Order?> GetByIdAsync(int id) =>
        await http.GetFromJsonAsync<Order?>($"api/orders/{id}");
    
    public async Task<Order> CreateAsync(Order order)
    {
        var response = await http.PostAsJsonAsync("api/orders", order);
        return await response.Content.ReadFromJsonAsync<Order>() ?? order;
    }
}
```

### Authentication для API calls

```csharp
// Program.cs
builder.Services.AddHttpClient("WebAPI", client => 
    client.BaseAddress = new Uri("https://api.example.com"))
    .AddHttpMessageHandler<TokenHandler>();

builder.Services.AddTransient<TokenHandler>();

// TokenHandler.cs — добавляет JWT в каждый request
public class TokenHandler : DelegatingHandler
{
    private readonly IAccessTokenProvider _tokenProvider;
    
    public TokenHandler(IAccessTokenProvider tokenProvider) => _tokenProvider = tokenProvider;
    
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        var tokenResult = await _tokenProvider.RequestAccessToken();
        if (tokenResult.TryGetToken(out var token))
        {
            request.Headers.Authorization = new("Bearer", token.Value);
        }
        return await base.SendAsync(request, ct);
    }
}
```

---

## 6. JS Interop

### Вызов JS из C#

```razor
@inject IJSRuntime JS

<button @onclick="ShowAlert">Alert</button>
<input @ref="inputRef" />
<button @onclick="FocusInput">Focus</button>

@code {
    private ElementReference inputRef;
    
    private async Task ShowAlert()
    {
        await JS.InvokeVoidAsync("alert", "Hello from C#!");
    }
    
    private async Task FocusInput()
    {
        await JS.InvokeVoidAsync("HTMLElement.prototype.focus.call", inputRef);
    }
    
    private async Task<string> GetUserAgent()
    {
        return await JS.InvokeAsync<string>("eval", "navigator.userAgent");
    }
}
```

### Custom JS module

```javascript
// wwwroot/js/myModule.js
export function showToast(message, type = 'info') {
    // Custom toast logic
    const toast = document.createElement('div');
    toast.className = `toast toast-${type}`;
    toast.textContent = message;
    document.body.appendChild(toast);
    setTimeout(() => toast.remove(), 3000);
}

export function getLocalStorage(key) {
    return localStorage.getItem(key);
}

export function setLocalStorage(key, value) {
    localStorage.setItem(key, value);
}
```

```csharp
// JsInterop wrapper
public class ToastService(IJSRuntime js) : IAsyncDisposable
{
    private IJSObjectReference? _module;
    
    private async Task<IJSObjectReference> GetModuleAsync() =>
        _module ??= await js.InvokeAsync<IJSObjectReference>(
            "import", "./js/myModule.js");
    
    public async Task ShowToastAsync(string message, string type = "info")
    {
        var module = await GetModuleAsync();
        await module.InvokeVoidAsync("showToast", message, type);
    }
    
    public async Task<string?> GetLocalStorageAsync(string key)
    {
        var module = await GetModuleAsync();
        return await module.InvokeAsync<string?>("getLocalStorage", key);
    }
    
    public async ValueTask DisposeAsync()
    {
        if (_module is not null)
            await _module.DisposeAsync();
    }
}

// Регистрация
builder.Services.AddScoped<ToastService>();
```

### Вызов C# из JS

```csharp
public class JsCallbackService
{
    [JSInvokable]
    public static int Add(int a, int b) => a + b;
    
    [JSInvokable]
    public static async Task<string> GetDataAsync()
    {
        await Task.Delay(100);
        return "data from C#";
    }
}
```

```javascript
// JS
const result = await DotNet.invokeMethodAsync('MyApp', 'Add', 2, 3);
console.log(result);  // 5

// Instance method
const dotNetRef = DotNetObjectReference.create(componentInstance);
await dotNetRef.invokeMethodAsync('OnEvent', data);
```

---

## 7. Forms и validation

```razor
@page "/register"
@using System.ComponentModel.DataAnnotations

<EditForm Model="model" OnValidSubmit="HandleSubmit">
    <DataAnnotationsValidator />
    <ValidationSummary />
    
    <div>
        <label>Name:</label>
        <InputText @bind-Value="model.Name" />
        <ValidationMessage For="@(() => model.Name)" />
    </div>
    
    <div>
        <label>Age:</label>
        <InputNumber @bind-Value="model.Age" />
        <ValidationMessage For="@(() => model.Age)" />
    </div>
    
    <div>
        <label>Email:</label>
        <InputText @bind-Value="model.Email" />
        <ValidationMessage For="@(() => model.Email)" />
    </div>
    
    <button type="submit">Register</button>
</EditForm>

@code {
    private RegisterModel model = new();
    
    private async Task HandleSubmit()
    {
        await Http.PostAsJsonAsync("api/register", model);
    }
    
    public class RegisterModel
    {
        [Required, StringLength(50)]
        public string Name { get; set; } = "";
        
        [Range(18, 120)]
        public int Age { get; set; }
        
        [Required, EmailAddress]
        public string Email { get; set; } = "";
    }
}
```

### FluentValidation

```bash
dotnet add package FluentValidation
```

```csharp
public class RegisterValidator : AbstractValidator<RegisterModel>
{
    public RegisterValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(50);
        RuleFor(x => x.Age).InclusiveBetween(18, 120);
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
    }
}
```

---

## 8. Authentication

### Authentication state

```csharp
// Program.cs
builder.Services.AddOidcAuthentication(options =>
{
    builder.Configuration.Bind("Auth0", options.ProviderOptions);
    options.ProviderOptions.ResponseType = "code";
});
```

```json
// appsettings.json
{
  "Auth0": {
    "Authority": "https://your-tenant.auth0.com",
    "ClientId": "your-client-id",
    "RedirectUri": "https://localhost:5001/authentication/login-callback",
    "PostLogoutRedirectUri": "https://localhost:5001",
    "ResponseType": "code"
  }
}
```

### AuthorizeView

```razor
@page "/secure"

<AuthorizeView>
    <Authorized>
        <p>Hello, @context.User.Identity?.Name!</p>
        <p>Roles: @string.Join(", ", context.User.FindAll(ClaimTypes.Role).Select(c => c.Value))</p>
    </Authorized>
    <NotAuthorized>
        <p>Please <a href="authentication/login">log in</a></p>
    </NotAuthorized>
</AuthorizeView>

<AuthorizeView Roles="Admin,Manager">
    <Authorized>
        <p>Admin panel</p>
    </Authorized>
</AuthorizeView>
```

### Защита роутов

```razor
@page "/admin"
@attribute [Authorize(Roles = "Admin")]

<h1>Admin only</h1>
```

### Программный доступ

```razor
@inject AuthenticationStateProvider AuthState

@code {
    private string? userName;
    
    protected override async Task OnInitializedAsync()
    {
        var state = await AuthState.GetAuthenticationStateAsync();
        var user = state.User;
        
        if (user.Identity?.IsAuthenticated == true)
        {
            userName = user.Identity.Name;
        }
    }
}
```

---

## 9. Performance optimization

### AOT compilation (.NET 6+)

Native compilation в WebAssembly — намного быстрее но **bigger bundle**.

```xml
<PropertyGroup>
    <RunAOTCompilation>true</RunAOTCompilation>
</PropertyGroup>
```

```bash
dotnet publish -c Release
# Build time: ~5-10x slower
# Runtime performance: 2-10x faster для CPU-bound
# Bundle size: ~2-3x larger (8-15 MB vs 3-5 MB)

```

> [!info] Когда AOT
> - Heavy computation в браузере (charts, simulations)
> - Image / video processing
> - Не нужно для simple CRUD UI

### Lazy loading assemblies

```xml
<!-- MyApp.csproj -->
<ItemGroup>
    <BlazorWebAssemblyLazyLoad Include="HeavyFeature.dll" />
    <BlazorWebAssemblyLazyLoad Include="Reports.dll" />
</ItemGroup>
```

```razor
@inject LazyAssemblyLoader Loader

@code {
    private async Task LoadFeature()
    {
        var assemblies = await Loader.LoadAssembliesAsync(
            new[] { "HeavyFeature.dll", "Reports.dll" });
        
        // Теперь типы из этих assemblies доступны
        var type = assemblies.First()
            .GetTypes()
            .First(t => t.Name == "HeavyService");
    }
}
```

```razor
@* App.razor — Router пропустит до load *@
<Router AppAssembly="@typeof(App).Assembly"
        AdditionalAssemblies="@_lazyLoadedAssemblies"
        OnNavigateAsync="OnNavigateAsync">
    <!-- ... -->
</Router>

@code {
    private readonly List<Assembly> _lazyLoadedAssemblies = new();
    
    private async Task OnNavigateAsync(NavigationContext args)
    {
        if (args.Path == "reports")
        {
            var assemblies = await Loader.LoadAssembliesAsync(["Reports.dll"]);
            _lazyLoadedAssemblies.AddRange(assemblies);
        }
    }
}
```

### Trimming

```xml
<PropertyGroup>
    <PublishTrimmed>true</PublishTrimmed>
    <TrimMode>partial</TrimMode>  <!-- или full для агрессивного -->
</PropertyGroup>
```

Может уменьшить bundle на 30-50%.

### Бандл оптимизация

- **Compression** — Brotli auto-applied (`.gz` / `.br` отдаются сервером)
- **CDN** — статические файлы → CloudFlare / CloudFront
- **PWA caching** — service worker кэширует runtime

### Virtualization для длинных списков

```razor
@* Render только видимые items, остальные lazy *@
<Virtualize Items="@items" Context="item" ItemSize="50">
    <ItemContent>
        <div>@item.Name</div>
    </ItemContent>
    <Placeholder>
        <div>Loading...</div>
    </Placeholder>
</Virtualize>

@code {
    private List<MyItem> items = Enumerable.Range(0, 100_000)
        .Select(i => new MyItem { Id = i }).ToList();
}
```

100k items — render только видимые ~20.

---

## 10. PWA (Progressive Web App)

```bash
dotnet new blazorwasm -o MyApp --pwa
```

### Service worker

```javascript
// wwwroot/service-worker.js
self.addEventListener('install', e => /* ... */);
self.addEventListener('fetch', e => /* ... */);
```

### Offline support

```javascript
// service-worker.js — кэшируем app shell
self.addEventListener('fetch', (event) => {
    event.respondWith(
        caches.match(event.request).then(response => {
            return response || fetch(event.request);
        })
    );
});
```

### Manifest

```json
// wwwroot/manifest.json
{
  "name": "My Blazor App",
  "short_name": "MyApp",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#fff",
  "theme_color": "#03173d",
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

После — пользователь может **install** приложение как native (Chrome shows "Install" button).

---

## 11. Deployment

### Static files на CDN

Blazor WASM публикуется как **обычные static files**:

```bash
dotnet publish -c Release -o publish
# publish/wwwroot/ — содержит index.html, .dll, .wasm files

```

### Azure Static Web Apps

```yaml
# .github/workflows/azure-static-web-apps.yml
- name: Build and Deploy
  uses: Azure/static-web-apps-deploy@v1
  with:
    azure_static_web_apps_api_token: ${{ secrets.AZURE_TOKEN }}
    repo_token: ${{ secrets.GITHUB_TOKEN }}
    action: "upload"
    app_location: "/MyApp"
    output_location: "wwwroot"
```

### GitHub Pages

```yaml
- name: Publish
  run: dotnet publish -c Release -o publish

- name: Deploy
  uses: peaceiris/actions-gh-pages@v3
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: ./publish/wwwroot
```

### Nginx hosting

```nginx
server {
    listen 80;
    root /var/www/myapp;
    
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    # Proper MIME types
    types {
        application/wasm wasm;
        application/octet-stream dll;
    }
    
    # Compression
    gzip on;
    gzip_types application/wasm application/octet-stream application/javascript;
    
    # Caching
    location ~* \.(dll|wasm)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### Docker

```dockerfile
# Build
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /publish

# Serve
FROM nginx:alpine
COPY --from=build /publish/wwwroot /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

---

## 12. Common Pitfalls

### 1. Большой initial bundle

5 MB при первом visit → плохой UX на mobile.

**Решения:**
- AOT с trimming
- Lazy loading нерелевантных features
- Splitting reading-heavy / write-heavy
- Loading screen с прогрессом

### 2. Code visible client-side

```csharp
// ❌ Secret в WASM — DECOMPILE'ится тривиально!
public class ApiClient
{
    private const string ApiKey = "sk-secret-key";
}

// ✅ API key только на server, WASM делает запросы через your API
```

### 3. CORS issues

```
Access to fetch at 'https://api.example.com/data' from 'https://myapp.com' has been blocked by CORS
```

**Решение:** API сервер настраивает CORS для домена WASM app.

### 4. Threading limitations

WebAssembly до .NET 9 — **single-threaded**. `Thread.Sleep`, `Parallel.ForEach` блокируют UI.

```csharp
// ❌ Блокирует main UI thread в WASM
public void HeavyWork()
{
    Thread.Sleep(5000);  // freeze browser!
}

// ✅ Use Task / Web Workers (.NET 8+)
public async Task HeavyWorkAsync()
{
    await Task.Delay(5000);
}
```

.NET 8+ multi-threading в WASM (experimental):
```xml
<WasmEnableThreads>true</WasmEnableThreads>
```

### 5. File system limits

WebAssembly в браузере не имеет настоящего file system. `File.ReadAllText` использует in-memory virtual FS.

**Для real files:** drag-and-drop API + `IBrowserFile`:

```razor
<InputFile OnChange="LoadFiles" />

@code {
    private async Task LoadFiles(InputFileChangeEventArgs e)
    {
        var file = e.File;
        await using var stream = file.OpenReadStream(maxAllowedSize: 10_000_000);
        // Process stream
    }
}
```

### 6. GC pauses в браузере

WASM использует Boehm GC (отличается от server-side Gen0/Gen1/Gen2). Могут быть pause spikes.

**Лечение:**
- `.NET 8+` — newer GC implementations
- ArrayPool / Span — minimize allocations
- AOT может помочь

### 7. Browser back button и SPA routing

Native back button может ломать SPA navigation если не настроен правильно.

**Лечение:** `NavigationManager` обрабатывает корректно по умолчанию для standard navigation.

### 8. Initial render flicker

Bundle грузится → blank screen → render. Плохой UX.

**Лечение:**
```html
<!-- index.html -->
<div id="app">
    <div class="initial-loader">
        <div class="spinner"></div>
        <p>Loading application...</p>
    </div>
</div>
```

CSS spinner показан до hydration.

### 9. Не работает с Native AOT

Blazor WASM сам уже WebAssembly, но это **interpreted IL** (не native code). AOT доступен через `<RunAOTCompilation>true</RunAOTCompilation>`, но это другое чем Native AOT для server.

---

## 13. Best Practices

- **Hosted model** для full-stack — same solution, shared models, тесная integration
- **Standalone** для frontend-only — public hosting, max scale
- **AOT** для compute-heavy частей
- **Lazy loading** для больших features
- **Trimming** в production
- **Brotli compression** обязательно
- **PWA** для offline-capable apps
- **CDN delivery** для static files
- **HttpClient через IHttpClientFactory** — proper config
- **Token-based auth** через `IAccessTokenProvider`
- **Виртуализация** для длинных списков
- **Не храни secrets в WASM**
- **Auto-render** — избегай ручного `StateHasChanged()` если можно
- **Async lifecycle methods** — `OnInitializedAsync`, не `OnInitialized` для I/O
- **CancellationToken** в HTTP calls

---

## 14. Когда WASM vs Server vs render modes

```
Внутренний tool, low scale, real-time updates критично:
→ Blazor Server (low latency, server resources OK)

Public app, must scale, mobile-friendly:
→ Blazor WASM (CDN, no server load)

Best of both — modern default:
→ Render modes (.NET 8+): static SSR + InteractiveAuto
   - Server-side rendering для initial load (SEO!)
   - WebAssembly для interactivity после
   - Mix per-page render mode
```

> Термин «Blazor United» был рабочим названием этой модели и отброшен Microsoft до релиза .NET 8 — официально это **render modes**.

### Per-page render modes (.NET 8+)

```razor
@* Server-side render — SEO friendly *@
@page "/products"
@rendermode InteractiveServer

@* WebAssembly *@
@page "/dashboard"
@rendermode InteractiveWebAssembly

@* Auto — server first, потом WASM *@
@page "/profile"
@rendermode InteractiveAuto

@* Static — никакой interactivity *@
@page "/about"
```

**.NET 10 — WASM preloading:** framework-ассеты (dotnet.wasm, DLL) подгружаются заранее: в Blazor Web App — автоматически через `Link`-headers / компонент `<LinkPreload />`, в standalone WASM — через `<link rel="preload" id="webassembly" />` в `index.html` + MSBuild-свойство `OverrideHtmlAssetPlaceholders`. Браузер качает runtime параллельно с первым рендером → меньше time-to-interactive.

См. [[blazor-server|Blazor Server]].

---

## 15. Comparison: Blazor WASM vs React

| | Blazor WASM | React |
|--|-------------|-------|
| Language | C# | TypeScript |
| Bundle size | 2-5 MB | 100-500 KB |
| Startup | 1-3 sec | <1 sec |
| Type safety | Compile + runtime | TypeScript compile-time |
| Reuse models with backend | ✅ Same C# | Need DTOs |
| Ecosystem (UI libs) | Smaller (Radzen, MudBlazor) | Massive (MUI, Ant, Chakra) |
| Performance (typical) | Good | Excellent |
| Mobile native (RN) | MAUI | React Native |

**Когда Blazor WASM:** C# team, full-stack reuse, internal apps.

**Когда React:** public apps, mobile (RN), мессивный component ecosystem.

---

## Case Studies

### Case Study #1 — SPA в Blazor WASM vs Server

**Сценарий:** Internal admin panel, 50 одновременных users, sensitive data.

**Blazor Server:**
- ✅ Малый initial download (~250 KB)
- ✅ Full .NET runtime, debugging easy
- ✅ Code не leakают (server-side execution)
- ❌ SignalR connection всегда нужен
- ❌ Latency на каждое UI событие

**Blazor WASM:**
- ✅ Offline capable
- ✅ Static hosting (CDN)
- ✅ Latency только при API call
- ❌ Initial download 5-10 MB
- ❌ .NET runtime в браузер (slower startup)

**Решение:** Internal admin → Blazor Server (low concurrent, server-friendly).

---

### Case Study #2 — Hybrid с Blazor Server для real-time

**Сценарий:** Live trading dashboard. Updates каждую секунду.

**Blazor Server идеален:**
- SignalR built-in (real-time без extra setup)
- State server-side (не synchronization issues)
- Small client bundle

**Code:**
```csharp
@inject TradingService Trading
@implements IAsyncDisposable

<div>Price: @price</div>

@code {
    decimal price;
    Timer? timer;

    protected override void OnInitialized()
    {
        timer = new Timer(async _ => 
        {
            price = await Trading.GetCurrentPriceAsync("BTC");
            await InvokeAsync(StateHasChanged);
        }, null, 0, 1000);
    }

    public ValueTask DisposeAsync()
    {
        timer?.Dispose();
        return ValueTask.CompletedTask;
    }
}
```

---

### Case Study #3 — Migration старого AngularJS → Blazor

**Сценарий:** Internal app на AngularJS (deprecated). Migration к modern stack.

**Approach:**
1. **Blazor Hybrid в существующее ASP.NET MVC** — постепенно
2. **Component-by-component** — replace screens
3. **Shared API** — back-end не меняется
4. **Same auth** — cookie или JWT

**Result:** 6 month migration без big bang.

См. [[desktop-frameworks|Desktop Frameworks]] для full picture.


---

## Cheat sheet

| Need | Blazor Mode |
|------|-------------|
| Static site | Blazor WASM (или SSG) |
| Internal admin | Blazor Server |
| Public SPA с offline | Blazor WASM |
| Real-time critical | Blazor Server |
| Low latency UX | Blazor WASM (после load) |
| Mobile + Desktop одна codebase | Blazor MAUI Hybrid |
| SEO important | Blazor SSR (.NET 8+) |

| Component lifecycle | Hook |
|---------------------|------|
| Initialize | `OnInitialized` / `OnInitializedAsync` |
| Parameters changed | `OnParametersSet` |
| After render | `OnAfterRender(firstRender)` |
| Dispose | `IDisposable` / `IAsyncDisposable` |

| Common patterns | API |
|-----------------|-----|
| State management | `@code { }` для local, services для shared |
| Form validation | `EditForm` + DataAnnotations |
| HTTP calls | `HttpClient` (DI) |
| Authentication | `AuthorizeView`, `AuthenticationStateProvider` |
| Routing | `@page "/path"` directive |
| Parameters | `[Parameter]` attribute |
| Two-way binding | `@bind-Value` |
| Event handling | `@onclick`, `@onchange` |
| Conditional rendering | `@if`, `@foreach` |
| CSS isolation | `Component.razor.css` |


---

## Decision tree

```
Какой Blazor mode?
│
├── Internal app, low users (< 100)?
│   → Blazor Server (best DX, real-time built-in)
│
├── Public SPA, many users?
│   ├── SEO needed → Blazor SSR (.NET 8+) с interactivity
│   ├── Offline capability → Blazor WASM
│   └── Quick interactions → Blazor WASM
│
├── Mobile / desktop?
│   ├── Native apps → MAUI Blazor Hybrid (reuse web components)
│   ├── PWA → Blazor WASM
│   └── Cross-platform desktop → Avalonia (не Blazor)
│
├── Real-time?
│   ├── Live data dashboard → Blazor Server (SignalR built-in)
│   ├── Chat → Blazor + SignalR
│   └── Streaming → Blazor Server + IAsyncEnumerable
│
├── Performance critical?
│   ├── First page load → Blazor Server (smaller bundle)
│   ├── Subsequent interactions → Blazor WASM (no SignalR roundtrip)
│   └── Native AOT → WASM AOT (.NET 8+)
│
└── React/Vue alternative?
    ├── Team знает .NET → Blazor (tooling, debugging better)
    ├── Frontend team существует → React/Vue + .NET API
    └── Mix → Blazor компоненты в React via custom elements
```


---

## См. также

- [[blazor-server|Blazor Server]] — server-side rendering model
- [[desktop-frameworks|Desktop Frameworks]] — Avalonia / MAUI / Uno comparison
- [[native-aot|Native AOT]] — AOT compilation для server (отличается от WASM AOT)
- [[auth-security|Auth & Security]] — JWT для WASM
- [[resilience|Resilience]] — retry policies для WASM HTTP

## Reading list

- **Microsoft Docs — Blazor** — learn.microsoft.com/aspnet/core/blazor
- **Microsoft Docs — Blazor WebAssembly** — learn.microsoft.com/aspnet/core/blazor/host-and-deploy/webassembly
- **MudBlazor** — mudblazor.com (Material Design UI for Blazor)
- **Radzen Blazor** — blazor.radzen.com (commercial UI components)
- **Steve Sanderson talks** — Blazor co-creator
- **Blazor University** — blazor-university.com (community resource)
- **State of Blazor** — yearly community surveys
- **Awesome Blazor** — github.com/AdrienTorris/awesome-blazor
