---
tags: [wpf, mvvm, community-toolkit, viewmodel, xaml, dependency-injection]
level: Middle
date: 2026-08-02
---

# Snippets — WPF ViewModels (MVVM Toolkit)

> WPF на CommunityToolkit.Mvvm: `ObservableObject` с `[ObservableProperty]`/`[RelayCommand]`, async-safe загрузка из View.Loaded, DI через `IHostBuilder`, навигация и glassmorphism-стили в XAML.

> [!info] `IMediator` / `AddMediatR` ниже — MediatR 12.x
> MediatR 13+ коммерческий (community edition — только при выручке < $5M) — детали в [[choosing-dependencies|Choosing Dependencies]]. Сам MVVM-паттерн от библиотеки не зависит: замените `IMediator` на свой `IDispatcher` с тем же `Send(command, ct)` — [[vertical-slice-handler|Vertical Slice без MediatR]] — и ViewModel не изменится.

## Базовый ViewModel

```csharp
namespace MyApp.Presentation.ViewModels;

public sealed partial class OrdersViewModel : ObservableObject
{
    private readonly IMediator _mediator;
    private readonly ILogger<OrdersViewModel> _logger;

    [ObservableProperty]
    private ObservableCollection<OrderSummaryDto> _orders = [];

    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(HasOrders))]
    private bool _isLoading;

    [ObservableProperty]
    private string? _errorMessage;

    public bool HasOrders => Orders.Count > 0;

    public OrdersViewModel(IMediator mediator, ILogger<OrdersViewModel> logger)
    {
        _mediator = mediator;
        _logger = logger;
    }

    [RelayCommand]
    private async Task LoadOrdersAsync(CancellationToken ct)
    {
        IsLoading = true;
        ErrorMessage = null;

        try
        {
            var result = await _mediator.Send(new GetOrdersQuery(), ct);

            result.Match(
                onSuccess: items =>
                {
                    Orders = new ObservableCollection<OrderSummaryDto>(items);
                },
                onFailure: error =>
                {
                    ErrorMessage = error.Message;
                    _logger.LogWarning("Failed to load orders: {Error}", error.Message);
                }
            );
        }
        finally
        {
            IsLoading = false;
        }
    }

    [RelayCommand]
    private async Task CreateOrderAsync(CancellationToken ct)
    {
        // navigate or open dialog
    }
}
```

## DI + IHostBuilder setup (App.xaml.cs)

```csharp
namespace MyApp.Presentation;

public partial class App : System.Windows.Application
{
    private readonly IHost _host;

    public App()
    {
        _host = Host.CreateDefaultBuilder()
            .ConfigureServices((ctx, services) =>
            {
                // MediatR + Validators
                services.AddMediatR(cfg =>
                    cfg.RegisterServicesFromAssembly(typeof(CreateOrderCommand).Assembly));
                services.AddValidatorsFromAssembly(typeof(CreateOrderCommand).Assembly);

                // Infrastructure
                services.AddDbContext<AppDbContext>(opts =>
                    opts.UseSqlite(ctx.Configuration.GetConnectionString("Default")));

                // ViewModels — Transient (новый экземпляр на каждый View)
                services.AddTransient<OrdersViewModel>();
                services.AddTransient<DashboardViewModel>();

                // Views
                services.AddTransient<OrdersView>();
                services.AddSingleton<MainWindow>();
            })
            .Build();
    }

    protected override async void OnStartup(StartupEventArgs e)
    {
        await _host.StartAsync();

        var mainWindow = _host.Services.GetRequiredService<MainWindow>();
        mainWindow.Show();

        base.OnStartup(e);
    }

    protected override async void OnExit(ExitEventArgs e)
    {
        await _host.StopAsync();
        _host.Dispose();
        base.OnExit(e);
    }
}
```

## Glassmorphism стиль (XAML)

```xml
<!-- App.xaml — базовый стиль стекла -->
<Style x:Key="GlassPanel" TargetType="Border">
    <Setter Property="Background" Value="#1A1A2E"/>
    <Setter Property="BorderBrush" Value="#40FFFFFF"/>
    <Setter Property="BorderThickness" Value="1"/>
    <Setter Property="CornerRadius" Value="12"/>
    <Setter Property="Effect">
        <Setter.Value>
            <DropShadowEffect Color="#7C3AED" BlurRadius="20" Opacity="0.3" ShadowDepth="0"/>
        </Setter.Value>
    </Setter>
</Style>

<!-- Accent кнопка -->
<Style x:Key="AccentButton" TargetType="Button">
    <Setter Property="Background" Value="#7C3AED"/>
    <Setter Property="Foreground" Value="White"/>
    <Setter Property="BorderThickness" Value="0"/>
    <Setter Property="Padding" Value="16,8"/>
    <Setter Property="Cursor" Value="Hand"/>
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Border Background="{TemplateBinding Background}"
                        CornerRadius="8"
                        Padding="{TemplateBinding Padding}">
                    <ContentPresenter HorizontalAlignment="Center" VerticalAlignment="Center"/>
                </Border>
                <ControlTemplate.Triggers>
                    <Trigger Property="IsMouseOver" Value="True">
                        <Setter Property="Background" Value="#6D28D9"/>
                    </Trigger>
                    <Trigger Property="IsPressed" Value="True">
                        <Setter Property="Background" Value="#5B21B6"/>
                    </Trigger>
                </ControlTemplate.Triggers>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

## Навигация между Views (через DI)

```csharp
public interface INavigationService
{
    void NavigateTo<TViewModel>() where TViewModel : ObservableObject;
}

public sealed partial class MainViewModel(INavigationService navigation) : ObservableObject
{
    [ObservableProperty]
    private ObservableObject? _currentView;

    [RelayCommand]
    private void GoToOrders() => navigation.NavigateTo<OrdersViewModel>();

    [RelayCommand]
    private void GoToDashboard() => navigation.NavigateTo<DashboardViewModel>();
}
```

## Async-safe ViewModel инициализация

```csharp
// Паттерн: команда LoadAsync вызывается из View.Loaded
// НЕ делаем async в конструкторе!
public sealed partial class DashboardViewModel : ObservableObject
{
    [RelayCommand(CanExecute = nameof(CanLoad))]
    private async Task LoadAsync(CancellationToken ct)
    {
        // ...
    }

    private bool CanLoad() => !IsLoading;
}
```

```xml
<!-- View.xaml.cs code-behind — минимальный -->
public partial class DashboardView : UserControl
{
    public DashboardView(DashboardViewModel vm)
    {
        InitializeComponent();
        DataContext = vm;
        Loaded += async (_, _) => await vm.LoadCommand.ExecuteAsync(null);
    }
}
```

---

## См. также

- .NET Knowledge Base
