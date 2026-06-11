---
tags: [wpf, mvvm, desktop, velopack, fluent-design, community-toolkit]
level: Senior
---

# WPF Production Guide — современный desktop на .NET

## Что это, зачем и когда

### Что такое WPF в 2026?
**Windows Presentation Foundation** — UI-фреймворк Microsoft с 2006 года, единственный desktop-фреймворк .NET, который реально стабилен, проверен и поддерживается на нынешний момент. Не путать с WinForms (legacy), WinUI 3 (буксует), MAUI (нацелен на mobile/cross-platform), Avalonia (open-source альтернатива WPF).

**Аналогия:** WPF — это надёжный пикап F-150. Не модный, не быстрый, но возит всё что нужно последние 20 лет, запчастей навалом. WinUI / MAUI — концепт-кары, фоторгафии красивые, в продакшен довезти сложнее.

### Зачем WPF в 2026

| Без современного стека (старый WPF) | Современный WPF |
|--------------------------------------|------------------|
| INotifyPropertyChanged через `OnPropertyChanged("Name")` — typo runtime | `[ObservableProperty]` source generator — compile-time |
| Окна выглядят из 2008 (light blue chrome, Aero) | Fluent Design (Mica, Win11-стиль) через WPF-UI |
| Установщик через WiX/MSI — мука для авто-апдейтов | Velopack — `dotnet pack`, релиз за 30 секунд, delta-апдейты |
| ViewModel-base classes написаны вручную (1000+ строк) | CommunityToolkit.Mvvm — 0 boilerplate |
| Проблемы с DI / Generic Host | `IServiceProvider` нативно с .NET Generic Host |

### Decision matrix: что использовать в 2026

| Задача | Pick |
|--------|------|
| Внутренний enterprise tool, Windows-only | **WPF** + WPF-UI + MVVM Toolkit + Velopack |
| Кроссплатформенный desktop (Win/Mac/Linux) | **Avalonia** (XAML-аналог WPF) |
| Mobile + desktop из одного кода | **MAUI** (но сырой ещё) |
| Modern look + Win11 native APIs | **WinUI 3** (вкатиться сложнее, ecosystem мельче) |
| Embedded webview как UI | **WebView2 + Blazor Hybrid** |

WPF — лучший выбор когда **только Windows и нужна стабильность** (DailyPlanner, TradingBotForex).

---

## Modern WPF stack 2026

### Базовый набор

```xml
<!-- .csproj — таргет с full WPF + dependency injection -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net10.0-windows</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UseWPF>true</UseWPF>
    <ApplicationManifest>app.manifest</ApplicationManifest>
    <ApplicationIcon>icon.ico</ApplicationIcon>
  </PropertyGroup>

  <ItemGroup>
    <!-- MVVM с source generators -->
    <PackageReference Include="CommunityToolkit.Mvvm" Version="8.4.0" />

    <!-- Fluent Design controls -->
    <PackageReference Include="WPF-UI" Version="3.0.5" />

    <!-- DI / Hosting -->
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="10.0.0" />
    <PackageReference Include="Microsoft.Extensions.Logging.Console" Version="10.0.0" />

    <!-- Auto-update -->
    <PackageReference Include="Velopack" Version="0.0.999" />

    <!-- Опционально -->
    <PackageReference Include="MaterialDesignThemes" Version="5.0.0" />  <!-- если Material а не Fluent -->
  </ItemGroup>
</Project>
```

### Проектная структура

```
DailyPlanner.sln
├── DailyPlanner.App                   # WPF host (App.xaml, MainWindow)
│   ├── App.xaml(.cs)
│   ├── Views/                         # XAML pages/usercontrols
│   ├── ViewModels/                    # ObservableObject inheritors
│   ├── Services/                      # IDialogService, INavigationService
│   ├── Resources/                     # ResourceDictionaries, themes
│   └── Localization/                  # *.resx
│
├── DailyPlanner.Core                  # ViewModel-agnostic business logic
│   ├── Domain/
│   ├── Repositories/
│   └── Services/
│
└── DailyPlanner.Infrastructure        # EF Core, file IO, etc.
    └── Persistence/
```

---

## CommunityToolkit.Mvvm deep

### `ObservableObject` + `[ObservableProperty]`

```csharp
public sealed partial class TaskViewModel : ObservableObject
{
    [ObservableProperty]
    private string _title = string.Empty;

    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(IsOverdue))]
    [NotifyCanExecuteChangedFor(nameof(CompleteCommand))]
    private DateTime _dueDate;

    [ObservableProperty]
    private bool _isCompleted;

    public bool IsOverdue => !IsCompleted && DueDate < DateTime.Now;
}
```

Source generator разворачивает это в:
```csharp
public sealed partial class TaskViewModel : ObservableObject
{
    public string Title
    {
        get => _title;
        set
        {
            if (!EqualityComparer<string>.Default.Equals(_title, value))
            {
                OnTitleChanging(value);
                OnPropertyChanging();
                _title = value;
                OnTitleChanged(value);
                OnPropertyChanged();
            }
        }
    }
    // ... DueDate с NotifyPropertyChangedFor(IsOverdue)
}
```

Получаешь:
- Compile-time типизация (нет magic-string `"Title"`)
- `OnXxxChanging` / `OnXxxChanged` partial-методы (можно переопределить для side-effects)
- Атрибуты `NotifyPropertyChangedFor`, `NotifyCanExecuteChangedFor` — каскадные обновления

### `[RelayCommand]`

```csharp
public sealed partial class TaskViewModel : ObservableObject
{
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(CompleteCommand))]
    private bool _isCompleted;

    [RelayCommand(CanExecute = nameof(CanComplete))]
    private async Task CompleteAsync(CancellationToken ct)
    {
        await _repository.MarkCompletedAsync(Id, ct);
        IsCompleted = true;
    }

    private bool CanComplete() => !IsCompleted;

    [RelayCommand]
    private void Delete()
    {
        _messenger.Send(new TaskDeletedMessage(Id));
    }
}
```

Source generator создаёт `CompleteCommand` (типа `IAsyncRelayCommand`) и `DeleteCommand` (`IRelayCommand`). В XAML биндишь:
```xml
<Button Content="Complete" Command="{Binding CompleteCommand}" />
<Button Content="Delete" Command="{Binding DeleteCommand}" />
```

### `WeakReferenceMessenger` — pub-sub без утечек

```csharp
// Сообщения как records
public sealed record TaskDeletedMessage(Guid TaskId);

// Подписка
public sealed class TaskListViewModel : ObservableRecipient
{
    public TaskListViewModel(IMessenger messenger) : base(messenger)
    {
        IsActive = true;
    }

    protected override void OnActivated()
    {
        Messenger.Register<TaskListViewModel, TaskDeletedMessage>(this, (vm, msg) =>
        {
            var task = vm.Tasks.FirstOrDefault(t => t.Id == msg.TaskId);
            if (task is not null) vm.Tasks.Remove(task);
        });
    }
}

// Отправка
_messenger.Send(new TaskDeletedMessage(task.Id));
```

`WeakReferenceMessenger.Default` — singleton. Подписки по weak reference → ViewModel может быть собран GC если нет других ссылок (защита от утечек).

> [!question]- **Интервью: какая разница между `ObservableObject` и `ObservableRecipient`?**
> `ObservableObject` — базовый класс с `INotifyPropertyChanged`, `INotifyPropertyChanging`. Больше ничего.
>
> `ObservableRecipient` — наследует `ObservableObject` + интегрирует с `IMessenger`. Имеет `IsActive` свойство, при `IsActive = true` вызываются `OnActivated`/`OnDeactivated`. Удобно для подписок, привязанных к жизненному циклу ViewModel.
>
> Для большинства простых ViewModels — `ObservableObject`. Для тех, что подписываются на сообщения — `ObservableRecipient`.

### `ObservableValidator` — валидация по аннотациям

```csharp
public sealed partial class CreateTaskViewModel : ObservableValidator
{
    [ObservableProperty]
    [Required]
    [MinLength(3)]
    [MaxLength(100)]
    private string _title = string.Empty;

    [RelayCommand(CanExecute = nameof(CanSave))]
    private async Task SaveAsync()
    {
        ValidateAllProperties();
        if (HasErrors) return;

        await _service.CreateAsync(new TaskDto(Title));
    }

    private bool CanSave() => !HasErrors && !string.IsNullOrWhiteSpace(Title);
}
```

В XAML:
```xml
<TextBox Text="{Binding Title, UpdateSourceTrigger=PropertyChanged, ValidatesOnNotifyDataErrors=True}" />
```

---

## Generic Host + Dependency Injection

### App.xaml.cs с .NET Generic Host

```csharp
public partial class App : Application
{
    private readonly IHost _host;

    public App()
    {
        _host = Host.CreateDefaultBuilder()
            .ConfigureAppConfiguration((ctx, config) =>
            {
                config.AddJsonFile("appsettings.json", optional: false);
                config.AddJsonFile($"appsettings.{ctx.HostingEnvironment.EnvironmentName}.json",
                                   optional: true);
                config.AddUserSecrets<App>();  // dev-only
            })
            .ConfigureServices((ctx, services) =>
            {
                // Database
                var connStr = ctx.Configuration.GetConnectionString("Default")!;
                services.AddDbContext<AppDbContext>(opt => opt.UseSqlite(connStr));

                // Infrastructure
                services.AddSingleton<IDialogService, DialogService>();
                services.AddSingleton<INavigationService, NavigationService>();
                services.AddTransient<ITaskRepository, TaskRepository>();

                // Messaging
                services.AddSingleton<IMessenger>(WeakReferenceMessenger.Default);

                // ViewModels
                services.AddSingleton<MainViewModel>();
                services.AddTransient<TaskListViewModel>();
                services.AddTransient<CreateTaskViewModel>();

                // Views
                services.AddSingleton<MainWindow>();
            })
            .ConfigureLogging(logging =>
            {
                logging.AddDebug();
                logging.AddFile("logs/app-{Date}.log");  // через Serilog или другой провайдер
            })
            .Build();
    }

    protected override async void OnStartup(StartupEventArgs e)
    {
        await _host.StartAsync();

        // Apply DB migrations
        using (var scope = _host.Services.CreateScope())
        {
            var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
            await db.Database.MigrateAsync();
        }

        // Resolve and show main window
        var window = _host.Services.GetRequiredService<MainWindow>();
        window.Show();

        base.OnStartup(e);
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        await _host.StopAsync(TimeSpan.FromSeconds(5));
        _host.Dispose();
        base.OnExit(e);
    }
}
```

### ViewModelLocator pattern

В XAML биндишь View к ViewModel через locator:
```xml
<Window xmlns:vm="clr-namespace:DailyPlanner.ViewModels">
    <Window.DataContext>
        <Binding Path="MainVm" Source="{x:Static vm:ViewModelLocator.Instance}" />
    </Window.DataContext>
</Window>
```

```csharp
public sealed class ViewModelLocator
{
    public static ViewModelLocator Instance { get; } =
        new ViewModelLocator(((App)Application.Current).Services);

    private readonly IServiceProvider _sp;
    private ViewModelLocator(IServiceProvider sp) => _sp = sp;

    public MainViewModel MainVm => _sp.GetRequiredService<MainViewModel>();
    public TaskListViewModel TaskListVm => _sp.GetRequiredService<TaskListViewModel>();
}
```

Альтернатива — создать ViewModel в code-behind View:
```csharp
public partial class TaskListView : UserControl
{
    public TaskListView()
    {
        InitializeComponent();
        DataContext = ((App)Application.Current).Services
            .GetRequiredService<TaskListViewModel>();
    }
}
```

---

## WPF Performance — production patterns

### UI virtualization

Без virtualization при отрисовке 10000 элементов в `ListBox` создаются 10000 `ListBoxItem` контейнеров → лаги, фризы, OOM.

```xml
<ListBox ItemsSource="{Binding Tasks}"
         VirtualizingStackPanel.IsVirtualizing="True"
         VirtualizingStackPanel.VirtualizationMode="Recycling"
         VirtualizingStackPanel.ScrollUnit="Pixel"
         ScrollViewer.IsDeferredScrollingEnabled="True">
    <ListBox.ItemsPanel>
        <ItemsPanelTemplate>
            <VirtualizingStackPanel />
        </ItemsPanelTemplate>
    </ListBox.ItemsPanel>
</ListBox>
```

`VirtualizationMode=Recycling` — переиспользует контейнеры (10x быстрее, чем `Standard`).
`ScrollUnit=Pixel` — плавная прокрутка по пикселям, без скачков по элементам.
`IsDeferredScrollingEnabled` — UI обновляется только при отпускании скроллбара (хорошо для очень длинных списков).

### Freezable для общих ресурсов

`Freezable`-объекты (Brush, Geometry, Transform) можно «заморозить» — после этого WPF не делает change-notification, не клонирует, не блокирует thread. Огромный буст в производительности.

```xml
<!-- ❌ BAD — изменяемая кисть, тратит память на change-notification -->
<SolidColorBrush x:Key="PrimaryBrush" Color="#2E4053" />

<!-- ✅ GOOD — frozen, потребление памяти меньше, рендер быстрее -->
<SolidColorBrush x:Key="PrimaryBrush" Color="#2E4053" PresentationOptions:Freeze="True" />
```

Все «глобальные» ресурсы из `App.xaml` помечай `Freeze="True"` — это нагнетающая правило.

### Binding mode — выбирай минимально нужный

| Mode | Когда |
|------|-------|
| `OneTime` | Значение никогда не меняется (заголовок, иконка) — самый дешёвый |
| `OneWay` | Только UI читает (статусные индикаторы) |
| `OneWayToSource` | UI пишет, ViewModel не уведомляет (редкое) |
| `TwoWay` | Двусторонний (текстбоксы, чекбоксы) — дороже всего |
| `Default` | Зависит от свойства; обычно `OneWay` для большинства |

```xml
<!-- ❌ BAD — ScrollViewer.HorizontalOffset не нужен TwoWay -->
<ScrollViewer HorizontalOffset="{Binding Offset}" />

<!-- ✅ GOOD -->
<ScrollViewer HorizontalOffset="{Binding Offset, Mode=OneWay}" />
```

### IsAsync для медленных Bindings

```xml
<!-- Свойство тяжёлое, биндим асинхронно -->
<TextBlock Text="{Binding LoadStatistics, IsAsync=True, FallbackValue='Loading...'}" />
```

WPF вычислит свойство в фоне, пока показывает FallbackValue.

### Async loading паттерн в ViewModel

```csharp
public sealed partial class TaskListViewModel : ObservableObject
{
    [ObservableProperty]
    private bool _isLoading;

    [ObservableProperty]
    private ObservableCollection<TaskViewModel> _tasks = new();

    public TaskListViewModel(ITaskRepository repo, IDispatcher dispatcher)
    {
        _ = LoadAsync();  // fire-and-forget с логгированием в catch
    }

    private async Task LoadAsync()
    {
        try
        {
            IsLoading = true;
            var tasks = await _repo.GetAllAsync();

            // ОБЯЗАТЕЛЬНО на UI-thread!
            await Application.Current.Dispatcher.InvokeAsync(() =>
            {
                Tasks.Clear();
                foreach (var t in tasks)
                    Tasks.Add(new TaskViewModel(t));
            });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to load tasks");
        }
        finally
        {
            IsLoading = false;
        }
    }
}
```

---

## Threading в WPF

### Правило: всё что меняет UI — на Dispatcher

```csharp
// ❌ Background thread пишет в ObservableCollection — крашит
public async Task LoadAsync()
{
    var data = await _api.FetchAsync();
    Tasks.Add(new TaskViewModel(data));  // InvalidOperationException
}

// ✅ Через Dispatcher.InvokeAsync
public async Task LoadAsync()
{
    var data = await _api.FetchAsync();
    await Application.Current.Dispatcher.InvokeAsync(() =>
    {
        Tasks.Add(new TaskViewModel(data));
    });
}

// ✅ Альтернатива: ConfigureAwait(true) — continuation на исходном SyncContext
public async Task LoadAsync()
{
    var data = await _api.FetchAsync();  // ConfigureAwait(true) by default
    Tasks.Add(new TaskViewModel(data));   // ОК если LoadAsync вызвали с UI thread
}
```

### IDispatcher abstraction для тестируемости

```csharp
public interface IDispatcher
{
    Task InvokeAsync(Action action, CancellationToken ct = default);
    Task<T> InvokeAsync<T>(Func<T> func, CancellationToken ct = default);
    bool CheckAccess();
}

public sealed class WpfDispatcher : IDispatcher
{
    private readonly Dispatcher _dispatcher = Application.Current.Dispatcher;

    public Task InvokeAsync(Action action, CancellationToken ct = default) =>
        _dispatcher.InvokeAsync(action).Task;

    public Task<T> InvokeAsync<T>(Func<T> func, CancellationToken ct = default) =>
        _dispatcher.InvokeAsync(func).Task;

    public bool CheckAccess() => _dispatcher.CheckAccess();
}

// В тестах
public sealed class TestDispatcher : IDispatcher
{
    public Task InvokeAsync(Action action, CancellationToken ct = default)
    { action(); return Task.CompletedTask; }

    public Task<T> InvokeAsync<T>(Func<T> func, CancellationToken ct = default) =>
        Task.FromResult(func());

    public bool CheckAccess() => true;
}
```

ViewModel принимает `IDispatcher` через DI → можно тестировать без UI thread'а.

### IProgress\<T\> — нативный паттерн прогресса

```csharp
public async Task<Result> ExportAsync(IProgress<int> progress, CancellationToken ct)
{
    for (var i = 0; i < items.Count; i++)
    {
        await ProcessAsync(items[i], ct);
        progress.Report((i + 1) * 100 / items.Count);
    }
    return Result.Success();
}

// В ViewModel
[RelayCommand]
private async Task ExportAsync()
{
    var progress = new Progress<int>(value => ExportPercent = value);
    var result = await _service.ExportAsync(progress, _cts.Token);
    // ExportPercent обновляется на UI-thread автоматически — IProgress<T> capture'ит SyncContext
}
```

`Progress<T>` (конкретный класс, не интерфейс) автоматически захватывает SyncContext в момент создания → колбэки выполняются на UI-thread.

---

## Theming и Fluent Design (WPF-UI)

### Установка WPF-UI

```xml
<PackageReference Include="WPF-UI" Version="3.0.5" />
```

### App.xaml

```xml
<Application x:Class="DailyPlanner.App"
             xmlns:ui="http://schemas.lepo.co/wpfui/2022/xaml">
    <Application.Resources>
        <ResourceDictionary>
            <ResourceDictionary.MergedDictionaries>
                <ui:ThemesDictionary Theme="Dark" />
                <ui:ControlsDictionary />
            </ResourceDictionary.MergedDictionaries>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

### Fluent Window с Mica

```xml
<ui:FluentWindow x:Class="DailyPlanner.MainWindow"
                 xmlns:ui="http://schemas.lepo.co/wpfui/2022/xaml"
                 ExtendsContentIntoTitleBar="True"
                 WindowBackdropType="Mica"
                 WindowCornerPreference="Round"
                 Width="1200" Height="800">
    <Grid>
        <ui:NavigationView>
            <ui:NavigationView.MenuItems>
                <ui:NavigationViewItem Content="Tasks" Icon="{ui:SymbolIcon TaskList24}" />
                <ui:NavigationViewItem Content="Settings" Icon="{ui:SymbolIcon Settings24}" />
            </ui:NavigationView.MenuItems>
        </ui:NavigationView>
    </Grid>
</ui:FluentWindow>
```

### Динамическое переключение темы (Light/Dark/System)

```csharp
public sealed class ThemeService
{
    public void SetTheme(ApplicationTheme theme)
    {
        Wpf.Ui.Appearance.ApplicationThemeManager.Apply(theme);

        // Опционально — sync с системной темой
        if (theme == ApplicationTheme.Unknown)
        {
            var systemTheme = Wpf.Ui.Appearance.ApplicationThemeManager.GetSystemTheme();
            Wpf.Ui.Appearance.ApplicationThemeManager.Apply(systemTheme);
        }
    }
}
```

### `DynamicResource` vs `StaticResource`

| | `StaticResource` | `DynamicResource` |
|--|-----------------|-------------------|
| Когда резолвится | На стадии XAML-load | Во время runtime |
| Performance | Быстро (один раз) | Медленнее (каждое обращение) |
| Применение | Иммутабельные ресурсы (фиксированные цвета) | Темы, локализация (могут меняться) |

Используй `DynamicResource` только для тех ресурсов, которые **реально меняются** runtime'ом (тема, язык). Все остальное — `StaticResource`.

```xml
<!-- ✅ Тема может переключаться -->
<Border Background="{DynamicResource PrimaryBackground}" />

<!-- ✅ Иконка не меняется -->
<Image Source="{StaticResource AppIcon}" />
```

---

## Velopack — production release pipeline

### Зачем Velopack

| Без Velopack | С Velopack |
|--------------|------------|
| MSI/WiX — день настройки на каждый релиз | `vpk pack` — 30 секунд |
| Переустановка целиком при апдейте (десятки MB) | Delta-апдейт (сотни KB) |
| Нет встроенного rollback | `vpk rollback` или auto-rollback при крэше первого запуска |
| Code signing руками через `signtool` | Встроенная поддержка SignTool / sigstore |
| Сами строишь канал доставки (own server / GitHub Releases / S3) | Автоматический upload в GitHub Releases / S3 / Azure |

### Установка и базовая настройка

```bash
dotnet tool install -g vpk
```

В `Program.cs` (или App.xaml.cs):
```csharp
public partial class App : Application
{
    [STAThread]
    public static void Main(string[] args)
    {
        // Velopack должен быть первой строчкой Main!
        VelopackApp.Build()
            .WithFirstRun(v => OnFirstRun())
            .WithAfterUpdateFastCallback(v => OnAfterUpdate(v))
            .Run();

        // Дальше обычный bootstrap
        var app = new App();
        app.InitializeComponent();
        app.Run();
    }
}
```

### Update flow в приложении

```csharp
public sealed class UpdateService
{
    private readonly UpdateManager _mgr;

    public UpdateService()
    {
        _mgr = new UpdateManager("https://github.com/valinerosgordov/DailyPlanner/releases/latest");
    }

    public async Task<UpdateInfo?> CheckAsync(CancellationToken ct)
    {
        try
        {
            return await _mgr.CheckForUpdatesAsync();
        }
        catch (Exception ex) { /* лог, не падать */ return null; }
    }

    public async Task DownloadAsync(UpdateInfo info, IProgress<int> progress, CancellationToken ct)
    {
        await _mgr.DownloadUpdatesAsync(info, p => progress.Report(p));
    }

    public void ApplyAndRestart(UpdateInfo info)
    {
        _mgr.ApplyUpdatesAndRestart(info);  // закроет приложение, применит, рестарт
    }
}
```

### Релиз в GitHub Actions

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  release:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Build
        run: dotnet publish DailyPlanner.App -c Release -r win-x64 -o publish

      - name: Install Velopack
        run: dotnet tool install -g vpk

      - name: Pack release
        env:
          VPK_VERSION: ${{ github.ref_name }}
        run: |
          vpk download github --repoUrl https://github.com/${{ github.repository }} --token ${{ secrets.GITHUB_TOKEN }}
          vpk pack `
            --packId DailyPlanner `
            --packVersion ${VPK_VERSION#v} `
            --packDir publish `
            --mainExe DailyPlanner.App.exe `
            --icon assets/icon.ico

      - name: Upload to GitHub Releases
        run: vpk upload github --repoUrl https://github.com/${{ github.repository }} --token ${{ secrets.GITHUB_TOKEN }} --tag ${{ github.ref_name }}
```

### Channels — beta / alpha / stable

```bash
vpk pack --channel beta ...
```

В клиенте можно подписаться на конкретный канал:
```csharp
var mgr = new UpdateManager(source, new UpdateOptions { ExplicitChannel = "beta" });
```

### Code signing

```bash
vpk pack \
  --signTool="signtool sign /td sha256 /fd sha256 /tr http://timestamp.digicert.com /sha1 <thumbprint>" \
  ...
```

Без подписи Windows SmartScreen будет ругаться при установке — для production это критично.

> [!question]- **Интервью: как Velopack делает delta-обновления?**
> Хранит каждый релиз как nupkg + binary diff (`*.delta` файлы) от предыдущей версии. При апдейте клиент скачивает только delta (обычно 1-5% от full size), применяет к локальной копии. Если delta невозможна (например, апдейт через несколько версий) — fallback на full nupkg.
> Бенефит: пользователю не приходится скачивать 100 МБ ради 50 КБ изменений. Особенно важно для пользователей с медленным интернетом и для частых релизов.

---

## Localization

### `.resx` файлы

```
Resources/
├── Strings.resx           # default (en)
├── Strings.ru.resx        # Russian
└── Strings.de.resx        # German
```

В XAML:
```xml
<Window xmlns:r="clr-namespace:DailyPlanner.Resources">
    <TextBlock Text="{x:Static r:Strings.AppTitle}" />
</Window>
```

### Runtime смена языка через `WPFLocalizeExtension`

```xml
<PackageReference Include="WPFLocalizeExtension" Version="3.10.0" />
```

```xml
<Window xmlns:lex="http://wpflocalizeextension.codeplex.com"
        lex:LocalizeDictionary.DesignCulture="ru"
        lex:ResxLocalizationProvider.DefaultAssembly="DailyPlanner"
        lex:ResxLocalizationProvider.DefaultDictionary="Strings">
    <TextBlock Text="{lex:Loc AppTitle}" />
</Window>
```

```csharp
// Смена языка
LocalizeDictionary.Instance.Culture = new CultureInfo("ru-RU");
```

Все привязки `{lex:Loc}` обновляются автоматически без перезапуска.

---

## Common patterns

### IDialogService — асинхронные диалоги через DI

```csharp
public interface IDialogService
{
    Task<bool> ConfirmAsync(string title, string message);
    Task<string?> ShowInputAsync(string title, string prompt);
    Task ShowErrorAsync(string title, string message);
    Task<TViewModel?> ShowDialogAsync<TViewModel>(string title) where TViewModel : ObservableObject;
}

public sealed class DialogService(IServiceProvider sp) : IDialogService
{
    public async Task<bool> ConfirmAsync(string title, string message)
    {
        var dialog = new ConfirmDialog { Title = title, Message = message };
        await dialog.ShowAsync();  // WPF-UI ContentDialog
        return dialog.Result;
    }

    public async Task<TViewModel?> ShowDialogAsync<TViewModel>(string title)
        where TViewModel : ObservableObject
    {
        var vm = sp.GetRequiredService<TViewModel>();
        var view = sp.GetRequiredService<IViewFor<TViewModel>>();
        view.DataContext = vm;

        var dialog = new ContentDialog { Title = title, Content = view };
        var result = await dialog.ShowAsync();
        return result == ContentDialogResult.Primary ? vm : null;
    }
}
```

В ViewModel:
```csharp
[RelayCommand]
private async Task DeleteAsync()
{
    var confirmed = await _dialogs.ConfirmAsync("Удалить?", "Действие необратимо.");
    if (!confirmed) return;

    await _repo.DeleteAsync(SelectedTask.Id);
}
```

### INavigationService

```csharp
public interface INavigationService
{
    void NavigateTo<TViewModel>() where TViewModel : ObservableObject;
    void GoBack();
    bool CanGoBack { get; }
}
```

Реализация для Frame-based navigation в `MainWindow`:
```csharp
public sealed class FrameNavigationService(Frame frame, IServiceProvider sp) : INavigationService
{
    public void NavigateTo<TViewModel>() where TViewModel : ObservableObject
    {
        var vm = sp.GetRequiredService<TViewModel>();
        var page = sp.GetRequiredService<IViewFor<TViewModel>>();
        page.DataContext = vm;
        frame.Navigate(page);
    }
    // ...
}
```

### Crash reporter

```csharp
public partial class App : Application
{
    public App()
    {
        AppDomain.CurrentDomain.UnhandledException += OnUnhandled;
        DispatcherUnhandledException += OnDispatcherUnhandled;
        TaskScheduler.UnobservedTaskException += OnUnobservedTask;
    }

    private void OnDispatcherUnhandled(object sender, DispatcherUnhandledExceptionEventArgs e)
    {
        _logger.LogCritical(e.Exception, "Dispatcher exception");
        MessageBox.Show($"Произошла ошибка: {e.Exception.Message}", "Ошибка", MessageBoxButton.OK, MessageBoxImage.Error);
        e.Handled = true;  // не падать дальше, иначе процесс умрёт
    }

    private void OnUnhandled(object sender, UnhandledExceptionEventArgs e)
    {
        var ex = e.ExceptionObject as Exception;
        _logger.LogCritical(ex, "Unhandled exception (terminating: {T})", e.IsTerminating);
        // Опционально — sentry / crashlytics upload
    }
}
```

---

## Pitfalls

### 1. Memory leak через event handlers

```csharp
// ❌ ParentVm подписался на ChildVm.PropertyChanged → strong reference
// ChildVm не может быть собран GC даже когда удалён из коллекции
child.PropertyChanged += OnChildChanged;

// ✅ WeakEventManager (или WeakReferenceMessenger вместо events)
WeakEventManager<ChildVm, PropertyChangedEventArgs>.AddHandler(
    child, "PropertyChanged", OnChildChanged);
```

### 2. Bindings на disposed object — спам в Output

```
System.Windows.Data Error: 40 : BindingExpression path error: 'X' property not found on 'object'
```
Возникает когда биндинг указывает на свойство, которое отсутствует или DataContext null. Лечится:
- Установкой `FallbackValue` в Binding
- Корректным сбросом DataContext при unload View
- В Production — `PresentationTraceSources.TraceLevel="None"` для подавления (но лучше починить)

### 3. Deadlock в `ConfigureAwait(false)` после `await`

```csharp
// ❌ После ConfigureAwait(false) обновление UI = деадлок/исключение
public async Task LoadAsync()
{
    var data = await _api.FetchAsync().ConfigureAwait(false);
    Tasks.Add(new TaskViewModel(data));  // мы НЕ на UI-thread!
}

// ✅ Возвращаемся на UI-thread через Dispatcher
public async Task LoadAsync()
{
    var data = await _api.FetchAsync().ConfigureAwait(false);
    await _dispatcher.InvokeAsync(() => Tasks.Add(new TaskViewModel(data)));
}
```

### 4. Изменение `ObservableCollection<T>` из background thread
До .NET 4.5 — InvalidOperationException. Сейчас можно через `BindingOperations.EnableCollectionSynchronization`:
```csharp
var lockObj = new object();
var collection = new ObservableCollection<TaskViewModel>();
BindingOperations.EnableCollectionSynchronization(collection, lockObj);

// Из background thread
lock (lockObj) { collection.Add(new TaskViewModel(data)); }
```

Но лучше — просто всегда делать changes на UI-thread.

### 5. Stale `CommandManager.RequerySuggested`
При изменении свойства, влияющего на `CanExecute`, нужно вызвать `(MyCommand as IRelayCommand)?.NotifyCanExecuteChanged()` — иначе UI не обновит enabled-состояние кнопки. С `[NotifyCanExecuteChangedFor(...)]` в MVVM Toolkit это автоматизируется.

### 6. Window.Show() в OnStartup без ShutdownMode
По умолчанию WPF приложение завершается когда последнее окно закрыто. Если у тебя есть tray-icon без окна — нужно `Application.Current.ShutdownMode = ShutdownMode.OnExplicitShutdown`.

### 7. Race conditions при start/stop EF migrations
Миграция БД в `OnStartup` асинхронно — а `MainWindow` уже хочет данные.
```csharp
// ❌ MainWindow.Loaded может стрельнуть до завершения миграции
await _host.StartAsync();
window.Show();

// ✅ Дождаться миграции до показа окна
await _host.StartAsync();
using (var scope = _host.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}
var window = _host.Services.GetRequiredService<MainWindow>();
window.Show();
```

---

## См. также

- [Async и Threading]() — `IProgress<T>`, `ConfigureAwait`, Dispatcher pitfalls
- [DI и Configuration]() — Generic Host, Options pattern (применимо и к WPF)
- [HFT / Low-Latency]() — когда WPF UI-thread vs hot-path threads (для TradingBotForex)
- [WPF ViewModel snippet]() — короткие готовые сниппеты Toolkit'а
- [Testing]() — как тестировать ViewModel без WPF-thread (через IDispatcher abstraction)

## Reading list

- **WPF-UI** — github.com/lepoco/wpfui (документация и примеры Fluent Design)
- **CommunityToolkit.Mvvm** — learn.microsoft.com/dotnet/communitytoolkit/mvvm/
- **Velopack** — github.com/velopack/velopack — docs, examples, vs Squirrel
- **ModernWPF** — github.com/Kinnara/ModernWpf (альтернатива WPF-UI с другим стилем)
- **Adam Nathan — WPF Unleashed** — старая книга, но фундамент понимания layouts/bindings актуален
- **Sara Ford & Tom Archer — Windows Internals via WPF** — для глубокого понимания Dispatcher и render-pipeline
- **Microsoft Learn — WPF docs** — learn.microsoft.com/dotnet/desktop/wpf/
