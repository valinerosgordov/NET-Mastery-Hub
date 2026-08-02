---
tags: [csharp, desktop, senior, wpf, winui, winforms, maui, avalonia, xaml]
level: Senior
date: 2026-08-02
---

# Desktop Frameworks — WPF, WinUI, MAUI, Avalonia

> **WPF (mature Windows), WinUI 3 (modern Windows), WinForms (legacy), MAUI (cross-platform Microsoft), Avalonia (cross-platform community).** Какой фреймворк выбрать в 2026, MVVM patterns, XAML basics, ecosystem comparison. Закрывает пробел: «знаю про WPF, не понимаю когда WinUI 3, чем MAUI отличается от Avalonia, и стоит ли мигрировать с WinForms».

---

## 0. Как читать

Если выбираешь framework — раздел 1 (decision matrix) + конкретный target (раздел 2-6). MVVM — раздел 7. Migration — раздел 9. Production guidance — раздел 11 (best practices), 12 (pitfalls).

Cross-language якоря свёрнуты. Interview-вопросы встроены.

---

## 1. Что это, зачем и когда

### 1.1. Landscape 2026

```
Microsoft official:
- WinForms (.NET Framework + .NET 5+) — legacy, простые apps
- WPF (.NET Framework + .NET 5+) — mature Windows desktop
- WinUI 3 (.NET 5+) — modern Windows 10/11 (formerly UWP)
- MAUI (.NET 6+) — cross-platform (Windows/Mac/iOS/Android)
- Blazor Hybrid (.NET 6+) — web tech в desktop

Community:
- Avalonia (.NET 5+) — cross-platform, WPF-like API
- Uno Platform — cross-platform XAML (UWP-style)
```

### 1.2. Decision matrix

```
| Target           | Best framework | Alternatives |
|------------------|----------------|--------------|
| Windows-only legacy | WPF / WinForms | WinUI 3 |
| Windows 10/11 modern | WinUI 3 | WPF |
| Cross-platform desktop | Avalonia | MAUI |
| Cross-platform mobile + desktop | MAUI | Avalonia (mobile experimental) |
| Web tech + desktop | Blazor Hybrid (MAUI) | Electron alternative |
| Industrial / enterprise Windows | WPF | WinForms (simple) |
| Game UI | Custom (Unity/MonoGame) | Avalonia (overlay) |
```

### 1.3. Главное правило выбора

```
Windows-only + 2026 → WinUI 3 (modern) или WPF (mature, теперь с Fluent-темой)
Cross-platform desktop → Avalonia (best)
Cross-platform desktop + mobile → MAUI
Legacy Windows app → WPF (если LOB) или WinForms (simple)
Brand new project в 2026 → Avalonia или WinUI 3
```

### 1.4. Эволюция

| Год | Framework | Status |
|-----|-----------|--------|
| **2002** | WinForms | Legacy maintenance only |
| **2006** | WPF | Mature, still recommended Windows |
| **2012** | UWP | Renamed → WinUI 3 (2021+) |
| **2016** | Xamarin.Forms | Replaced by MAUI (2022) |
| **2017** | Avalonia 0.x | Beta, growing |
| **2021** | WinUI 3 | Production-ready |
| **2022** | MAUI | First stable, .NET 6 |
| **2024** | Avalonia 11 | Production-ready, многие adopt |
| **2024** | MAUI 8 | Stable, production scale |
| **2024** | WinUI 3 Native AOT | Windows App SDK 1.6 (сентябрь 2024) |
| **2024** | WPF Fluent-тема | .NET 9: `ThemeMode` Light/Dark/System; .NET 10 расширил |
| **2024** | MAUI NativeAOT | .NET 9: iOS / Mac Catalyst (opt-in) |

> [!info]- Если ты знаешь Swift/SwiftUI / Kotlin/Compose / React Native / Flutter / Electron
> **SwiftUI:** declarative, Apple-only. C# Avalonia/MAUI — cross-platform alternative.
>
> **Jetpack Compose:** declarative Kotlin, Android first. C# через MAUI.
>
> **React Native:** JS native bridges. C# MAUI similar concept (native UI, business logic shared).
>
> **Flutter:** Dart, custom rendering. Avalonia closer (custom rendering, cross-platform).
>
> **Electron:** Web tech + Chromium. Heavyweight. C# Blazor Hybrid alternative — lighter.

> [!question]- Интервью: какой desktop framework выбрать для нового проекта в 2026?
> Зависит от target: 1) **Windows only enterprise** → WPF (mature, ecosystem) или WinUI 3 (modern Win 10/11). 2) **Cross-platform desktop** → Avalonia (best community, WPF-like). 3) **Cross-platform desktop + mobile** → MAUI (Microsoft official). 4) **Legacy migration** → WinForms only если simple LOB. 5) **Web tech preferred** → Blazor Hybrid в MAUI/WinUI. **Avoid for new**: WinForms (legacy), UWP (replaced by WinUI 3), Xamarin.Forms (replaced by MAUI). Best practice: Avalonia для cross-platform open-source, MAUI если deep Microsoft integration, WinUI 3 для modern Windows-only.

---

## 2. WinForms — legacy

### 2.1. Когда WinForms

```
✅ Use case:
  - Simple internal LOB apps (CRUD)
  - Existing .NET Framework apps (maintenance)
  - Quick prototypes
  - Teams без XAML skills

❌ Avoid:
  - Modern UI/UX requirements
  - Touch / responsive
  - Animations / rich graphics
  - Cross-platform
  - New projects
```

### 2.2. Базовый WinForms

```csharp
// Program.cs
[STAThread]
static void Main()
{
    ApplicationConfiguration.Initialize();
    Application.Run(new MainForm());
}

// MainForm.cs
public partial class MainForm : Form
{
    public MainForm()
    {
        InitializeComponent();
        // Designer-generated controls
    }
    
    private void button1_Click(object sender, EventArgs e)
    {
        MessageBox.Show("Hello!");
    }
}
```

Designer-driven — drag-drop в Visual Studio, code-behind events.

### 2.3. .NET Framework vs .NET 5+

```
.NET Framework 4.8 — legacy
- Windows only
- Maintenance mode

.NET 5+ (5/6/7/8/9)
- WinForms ported to modern .NET
- New APIs (high-DPI, accessibility)
- Designer support в Visual Studio
- Same APIs as .NET Framework (mostly)
```

Mostly compatible — most WinForms code runs на .NET 8 unchanged.

### 2.4. Когда лучше остаться

```
Migrate WinForms → WPF/WinUI когда:
✅ Need modern UI/UX
✅ Touch / responsive design
✅ Complex data binding
✅ Cross-platform planned

Stay в WinForms когда:
✅ Working LOB app, no UX complaints
✅ Team comfortable, no MVVM знаний
✅ Migration cost > benefit
✅ Simple data entry forms
```

### 2.5. Strengths/weaknesses

```
✅ Pros:
  - Easy designer (drag-drop)
  - Mature, stable
  - Smaller learning curve
  - Native Windows look
  - Less verbose чем WPF XAML

❌ Cons:
  - No data binding (manual)
  - No animation framework
  - Pixel-based layout
  - High-DPI issues (improving)
  - No real cross-platform
```

### 2.6. Modernization patterns

```csharp
// async/await в event handlers
private async void LoadButton_Click(object sender, EventArgs e)
{
    LoadButton.Enabled = false;
    try
    {
        var data = await dbService.LoadAsync();
        dataGrid.DataSource = data;
    }
    finally
    {
        LoadButton.Enabled = true;
    }
}

// DI через Microsoft.Extensions.Hosting
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddTransient<MainForm>();
builder.Services.AddSingleton<IDataService, DataService>();
var host = builder.Build();
Application.Run(host.Services.GetRequiredService<MainForm>());
```

WinForms может использовать modern .NET features.

---

## 3. WPF — Windows mature

### 3.1. Когда WPF

```
✅ Use case:
  - Windows desktop LOB
  - Rich data binding required
  - MVVM architecture
  - Animations / custom UI
  - Mature ecosystem (DevExpress, Syncfusion)
  - Long-term Windows-only product

❌ Avoid:
  - Cross-platform required
  - Modern Win 10/11 specific UI (use WinUI 3)
  - Mobile target
```

### 3.2. XAML basics

```xml
<Window x:Class="MyApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="My App" Height="450" Width="800">
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
        </Grid.RowDefinitions>
        
        <TextBlock Grid.Row="0" Text="{Binding Title}" FontSize="24"/>
        
        <ListBox Grid.Row="1" 
                 ItemsSource="{Binding Items}"
                 SelectedItem="{Binding SelectedItem, Mode=TwoWay}">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <TextBlock Text="{Binding Name}"/>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
    </Grid>
</Window>
```

XAML — declarative UI markup. Bindings, styles, templates first-class.

### 3.3. Data binding

```csharp
public class MainViewModel : INotifyPropertyChanged
{
    private string _title = "Hello";
    public string Title
    {
        get => _title;
        set { _title = value; OnPropertyChanged(); }
    }
    
    public ObservableCollection<Item> Items { get; } = new();
    
    public event PropertyChangedEventHandler? PropertyChanged;
    protected void OnPropertyChanged([CallerMemberName] string? name = null) =>
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
}

// MainWindow.xaml.cs
public MainWindow()
{
    InitializeComponent();
    DataContext = new MainViewModel();
}
```

WPF binding — most powerful в .NET ecosystem. ObservableCollection, INotifyPropertyChanged.

### 3.4. Styles + Templates

```xml
<Window.Resources>
    <Style x:Key="PrimaryButton" TargetType="Button">
        <Setter Property="Background" Value="#0078D4"/>
        <Setter Property="Foreground" Value="White"/>
        <Setter Property="Padding" Value="10,5"/>
        <Setter Property="FontSize" Value="14"/>
    </Style>
</Window.Resources>

<Button Style="{StaticResource PrimaryButton}" Content="Click me"/>

<!-- ControlTemplate -->
<Style TargetType="Button">
    <Setter Property="Template">
        <Setter.Value>
            <ControlTemplate TargetType="Button">
                <Border Background="{TemplateBinding Background}"
                        CornerRadius="4">
                    <ContentPresenter HorizontalAlignment="Center"
                                      VerticalAlignment="Center"/>
                </Border>
            </ControlTemplate>
        </Setter.Value>
    </Setter>
</Style>
```

Powerful theming via styles. ControlTemplate — replace control rendering.

### 3.5. Commands

```xml
<Button Content="Save" Command="{Binding SaveCommand}"/>
```

```csharp
public class MainViewModel
{
    public ICommand SaveCommand { get; }
    
    public MainViewModel()
    {
        SaveCommand = new RelayCommand(Save, CanSave);
    }
    
    private void Save() { /* ... */ }
    private bool CanSave() => !string.IsNullOrEmpty(_title);
}

// RelayCommand — typical pattern (или CommunityToolkit.Mvvm RelayCommand)
public class RelayCommand : ICommand
{
    private readonly Action _execute;
    private readonly Func<bool>? _canExecute;
    
    public RelayCommand(Action execute, Func<bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }
    
    public bool CanExecute(object? parameter) => _canExecute?.Invoke() ?? true;
    public void Execute(object? parameter) => _execute();
    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
}
```

Command pattern decouples UI events от business logic.

### 3.6. Strengths/weaknesses

```
✅ Pros:
  - Most mature .NET desktop
  - Powerful XAML + bindings
  - Massive ecosystem (DevExpress, Telerik, Syncfusion)
  - Deep customization (ControlTemplate)
  - Animations / 3D / DirectX
  - 18+ years of patterns / community

❌ Cons:
  - Windows only
  - Verbose XAML
  - Steep learning curve
  - .NET Framework legacy version still common
  - Touch / mobile second-class
```

### 3.7. CommunityToolkit.Mvvm

```xml
<PackageReference Include="CommunityToolkit.Mvvm" Version="8.2.0" />
```

```csharp
public partial class MainViewModel : ObservableObject
{
    [ObservableProperty]
    private string _title = "Hello";
    
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(FullTitle))]
    private string _suffix = "";
    
    public string FullTitle => $"{Title} - {Suffix}";
    
    [RelayCommand]
    private async Task SaveAsync()
    {
        await Task.Delay(1000);
    }
}
```

Source generators убирают MVVM boilerplate. Standard 2024+.

> [!question]- Интервью: что такое MVVM в WPF?
> **Model-View-ViewModel** — separation of concerns: 1) **Model** — domain entities, business logic. 2) **View** — XAML + minimal code-behind (binding setup). 3) **ViewModel** — exposes Model state via properties + commands, no UI references. **Bindings** через `INotifyPropertyChanged` + `ObservableCollection`. **Commands** через `ICommand` для actions. **Benefits**: testability (ViewModel — pure C#), separation, designer / developer collaboration, reusability. **Tools**: CommunityToolkit.Mvvm (source generators убирают boilerplate). **Anti-patterns**: code-behind с business logic, direct UI manipulation, ViewModel ссылается на View. WPF — first framework где MVVM became standard.

---

## 4. WinUI 3 — modern Windows

### 4.1. Что это

```
WinUI 3 — modern Windows UI framework (формально WinAppSDK).
- Windows 10 1809+ / Windows 11
- Native controls (Fluent Design)
- Replaces UWP (deprecated)
- .NET 5+ или C++/WinRT
- Single-package project (Windows App SDK)
```

### 4.2. Project setup

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0-windows10.0.19041.0</TargetFramework>
    <RootNamespace>MyApp</RootNamespace>
    <ApplicationManifest>app.manifest</ApplicationManifest>
    <Platforms>x86;x64;ARM64</Platforms>
    <UseWinUI>true</UseWinUI>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.WindowsAppSDK" Version="1.5.231129002" />
  </ItemGroup>
</Project>
```

### 4.3. XAML differences vs WPF

```xml
<!-- WinUI 3 XAML — modern, similar но different namespace -->
<Window xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml">
    <Grid>
        <TextBlock Text="Hello WinUI 3" 
                   FontSize="24"
                   Foreground="{ThemeResource TextFillColorPrimaryBrush}"/>
        <Button Content="Click me" Style="{StaticResource AccentButtonStyle}"/>
    </Grid>
</Window>
```

Key differences vs WPF:
- `ThemeResource` (vs StaticResource) — light/dark theme aware.
- `AccentButtonStyle` — built-in Fluent styles.
- No `app.xaml` resources tree (different model).

### 4.4. Fluent Design

```
Built-in modern UI:
- Acrylic backgrounds
- Mica (Windows 11 material)
- Reveal highlight
- Connected animations
- Adaptive layouts

Без third-party libraries (DevExpress).
```

### 4.5. WinUI 3 vs WPF

| | WPF | WinUI 3 |
|---|-----|---------|
| Windows version | All (XP+) | Windows 10 1809+ / Windows 11 |
| Look | Customizable; с .NET 9 — built-in Fluent-тема (`ThemeMode`) | Fluent Design (modern) |
| .NET Framework | Yes | No (.NET 5+) |
| Touch / pen | Limited | First-class |
| Modern controls | Manual / 3rd party | Built-in |
| Maturity | ~20 years | ~5 years |
| Ecosystem | Massive | Growing |
| MVVM | Standard | Same (CommunityToolkit) |

Важное обновление: «WPF выглядит как Vista» больше не аргумент. **.NET 9 добавил WPF Fluent-тему** — `ThemeMode` (`Light` / `Dark` / `System`) на уровне Application/Window, Windows 11 look без third-party библиотек; **.NET 10 расширил** поддержку (стабилизация API, доработка контролов). Для modern look на WPF теперь достаточно:

```xml
<Application x:Class="MyApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             ThemeMode="System">
</Application>
```

### 4.6. Strengths/weaknesses

```
✅ Pros:
  - Modern Windows look out of box
  - Touch / pen first-class
  - Native Win 11 integration (Mica, etc.)
  - Performance optimized
  - Future-direction Microsoft

❌ Cons:
  - Windows 10 1809+ only (no Win 7/8)
  - Younger ecosystem (less third-party)
  - Migration from WPF non-trivial
  - Some WPF features missing
  - .NET Framework not supported
```

### 4.7. Когда WinUI 3 vs WPF

```
WinUI 3 если:
- Modern Win 10/11 only (no Win 7/8/Server 2012)
- Need Fluent Design out of box
- Touch / pen interaction
- New project, brand-new

WPF если:
- Windows 7/8/Server compatibility
- Existing WPF codebase
- Heavy reliance on DevExpress/Telerik
- Mature ecosystem
- Custom rendering / DirectX integration
```

> [!question]- Интервью: чем WinUI 3 отличается от WPF?
> 1) **Target**: WinUI 3 — Windows 10 1809+ / Win 11 only; WPF — все Windows since XP. 2) **Look**: WinUI 3 — Fluent Design (Mica, Acrylic, Reveal) built-in; WPF — с .NET 9 тоже имеет built-in Fluent-тему (`ThemeMode` Light/Dark/System, расширено в .NET 10), до этого default был ~Vista era. 3) **Touch / pen**: WinUI 3 first-class, WPF retrofitted. 4) **.NET Framework**: WPF supports, WinUI 3 — only .NET 5+. 5) **Maturity**: WPF ~20 years, WinUI 3 ~5 years. 6) **Ecosystem**: WPF massive (DevExpress, Telerik), WinUI 3 growing. 7) **Migration**: XAML similar но different namespace + APIs. **When WinUI 3**: modern Windows-only apps, Fluent Design required. **When WPF**: legacy support, mature ecosystem dependence. Microsoft direction — WinUI 3 для new Windows-only apps.

---

## 5. MAUI — cross-platform Microsoft

### 5.1. Что это

```
.NET MAUI — Multi-platform App UI.
- Successor to Xamarin.Forms (deprecated 2024)
- One codebase: Windows + macOS + iOS + Android
- Native controls (NOT custom rendering как Flutter)
- .NET 6+
- XAML или C# UI
```

### 5.2. Project structure

```
MyApp/
├── MyApp.csproj          # multi-target framework
├── App.xaml(.cs)         # shared
├── MainPage.xaml(.cs)    # shared UI
├── Platforms/
│   ├── Android/
│   ├── iOS/
│   ├── MacCatalyst/
│   └── Windows/
├── Resources/
│   ├── AppIcon/
│   ├── Fonts/
│   ├── Images/
│   └── Styles/
└── ViewModels/           # shared business logic
```

```xml
<TargetFrameworks>net8.0-android;net8.0-ios;net8.0-maccatalyst;net8.0-windows10.0.19041.0</TargetFrameworks>
```

### 5.3. XAML

```xml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml">
    <VerticalStackLayout Padding="20" Spacing="10">
        <Label Text="{Binding Greeting}" FontSize="24"/>
        
        <Entry Text="{Binding Name, Mode=TwoWay}" Placeholder="Your name"/>
        
        <Button Text="Click me" Command="{Binding SubmitCommand}"/>
        
        <CollectionView ItemsSource="{Binding Items}">
            <CollectionView.ItemTemplate>
                <DataTemplate>
                    <Label Text="{Binding Name}"/>
                </DataTemplate>
            </CollectionView.ItemTemplate>
        </CollectionView>
    </VerticalStackLayout>
</ContentPage>
```

XAML similar to WPF/WinUI but namespace different. Some controls renamed (`StackLayout` → `VerticalStackLayout`).

### 5.4. Platform-specific

```csharp
#if ANDROID
    // Android-specific
    var context = Android.App.Application.Context;
#elif IOS
    // iOS-specific
    var window = UIKit.UIApplication.SharedApplication.KeyWindow;
#elif WINDOWS
    // Windows-specific
    var hwnd = Microsoft.UI.Xaml.Window.GetCurrentWindow();
#endif
```

```csharp
// Or через DependencyService / interfaces
public interface IFileService
{
    Task<string> ReadAsync(string path);
}

#if ANDROID
public class FileService : IFileService { ... }
#elif IOS
public class FileService : IFileService { ... }
#endif
```

### 5.5. Native controls

MAUI **wraps native** controls:
- Android: Material Design components.
- iOS: UIKit.
- Windows: WinUI 3.
- macOS: AppKit (через MacCatalyst).

vs Flutter — full custom rendering, all platforms identical look. MAUI — looks native.

### 5.6. Strengths/weaknesses

```
✅ Pros:
  - Single codebase across platforms
  - Native look (each platform feels native)
  - Microsoft official + ecosystem
  - Visual Studio + Hot Reload
  - .NET libraries (NuGet)
  - C# language productivity

❌ Cons:
  - Younger framework (3 years stable)
  - Bugs / quirks per platform
  - Performance per-platform varies
  - iOS / macOS — Mac required для build
  - Smaller community vs Flutter
  - Desktop second-class (mobile-first)
```

### 5.7. MAUI vs Flutter / React Native

```
MAUI:
✅ C# (productivity, types)
✅ Native controls (looks native)
✅ Microsoft ecosystem
❌ Smaller community
❌ Performance varies

Flutter:
✅ Custom rendering (consistent UI)
✅ Best perf
✅ Hot reload mature
❌ Dart (smaller ecosystem)
❌ Doesn't look native

React Native:
✅ JS/TS (huge community)
✅ Native bridges
❌ JS bridge perf
❌ Native look inconsistent
```

> [!question]- Интервью: когда MAUI vs Avalonia для cross-platform desktop?
> 1) **MAUI** — Microsoft official, includes mobile (iOS/Android). Native controls (each platform looks native). Hot reload, VS integration. **But**: desktop second-class, mobile-first design. Stable с 2022. 2) **Avalonia** — community-driven, desktop-first (mobile experimental). Custom rendering (consistent across platforms — Linux/Mac/Windows look identical). WPF-like API (easier migration). Mature (v11+). **Choose MAUI**: need iOS + Android + desktop. **Choose Avalonia**: desktop-only cross-platform, WPF migration, prefer consistent look. Bottom line: MAUI для multi-platform inc. mobile, Avalonia для desktop-only с fewer surprises.

---

## 6. Avalonia — community cross-platform

### 6.1. Что это

```
Avalonia UI — open-source XAML framework.
- WPF-like API
- Cross-platform: Windows, macOS, Linux, iOS (preview), Android (preview)
- Custom rendering (Skia)
- .NET 6+
- Mature (v11+, 2023)
- WebAssembly support (Avalonia Browser)
```

### 6.2. Project setup

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Avalonia" Version="11.0.10" />
    <PackageReference Include="Avalonia.Desktop" Version="11.0.10" />
    <PackageReference Include="Avalonia.Themes.Fluent" Version="11.0.10" />
  </ItemGroup>
</Project>
```

### 6.3. XAML — почти WPF

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        x:Class="MyApp.MainWindow"
        Title="My App" Width="800" Height="600">
    <Grid RowDefinitions="Auto,*">
        <TextBlock Grid.Row="0" Text="{Binding Title}" FontSize="24"/>
        
        <ListBox Grid.Row="1"
                 ItemsSource="{Binding Items}"
                 SelectedItem="{Binding SelectedItem}">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <TextBlock Text="{Binding Name}"/>
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
    </Grid>
</Window>
```

XAML almost identical to WPF — namespace different (`avaloniaui`), some controls renamed.

### 6.4. Custom rendering

Avalonia uses **Skia** для rendering — same engine as Chrome, Flutter:
- Identical look across Windows/Mac/Linux.
- No native controls (vs MAUI).
- High performance.
- Modern visual effects.

### 6.5. WPF migration

```
Easy migration:
✅ XAML 80% compatible
✅ Bindings same syntax
✅ Styles / templates similar
✅ Commands / DataContext same
✅ MVVM pattern same

Need adjustment:
⚠️ Some namespaces (xmlns)
⚠️ Some controls (Grid.RowDefinitions sugar)
⚠️ Triggers (less native, use behaviors)
⚠️ Visual states
```

WPF developers могут pick up Avalonia за 1-2 days.

### 6.6. Cross-platform consistency

```csharp
// Same code runs on:
// - Windows
// - macOS
// - Linux (X11, Wayland)
// - iOS (preview)
// - Android (preview)
// - WebAssembly (Avalonia Browser, preview)

dotnet publish -c Release -r win-x64
dotnet publish -c Release -r linux-x64
dotnet publish -c Release -r osx-arm64
```

Single binary per RID. Native AOT support.

### 6.7. Strengths/weaknesses

```
✅ Pros:
  - WPF-like API (easy migration)
  - Cross-platform desktop первый class
  - Custom rendering (consistent look)
  - Native AOT support
  - Open-source, community-driven
  - Good docs
  - Active development

❌ Cons:
  - Mobile experimental
  - Smaller ecosystem than WPF
  - Less third-party (но growing)
  - Custom rendering — not native look
  - Windows-only resources / themes harder
```

### 6.8. Real-world usage

```
Notable Avalonia apps:
- JetBrains Rider (refactored UI to Avalonia 2023)
- AvaloniaUI Visual Designer
- Kotor Tool, ButterUp
- Multiple internal Microsoft tools

Growing adoption — к 2026 это уже не outsider, а дефолтный выбор для cross-platform desktop на .NET.
```

> [!question]- Интервью: что такое Avalonia и почему её используют?
> Open-source XAML framework cross-platform desktop (+ mobile preview). **WPF-like API** — easy migration для existing teams. **Custom rendering** через Skia (same as Chrome/Flutter) — consistent look на Windows/Mac/Linux/iOS/Android/WebAssembly. **Mature** (v11+). **Strengths**: cross-platform desktop первый class, single codebase, Native AOT, modern features (animations, themes, hot reload). **Trade-offs**: smaller ecosystem чем WPF, mobile experimental, не native look (vs MAUI). **Adoption**: JetBrains Rider rewrote UI to Avalonia 2023. **Choose Avalonia**: cross-platform desktop, prefer consistent look over native, WPF migration. **Choose alternative**: MAUI (mobile required), WinUI 3 (Windows-only modern).

---

## 7. MVVM — universal pattern

### 7.1. Universal architecture

```
View (XAML) ↔ binding ↔ ViewModel ↔ Model

View — declarative UI, minimal code-behind
ViewModel — exposes state + commands, testable
Model — domain entities, business logic
```

Works в WPF, WinUI 3, MAUI, Avalonia — universal.

### 7.2. Полный MVVM пример

```csharp
// Model
public class Item
{
    public string Name { get; set; } = "";
    public DateTime CreatedAt { get; set; }
}

// ViewModel
public partial class ItemsViewModel : ObservableObject
{
    private readonly IDataService _dataService;
    
    [ObservableProperty]
    private ObservableCollection<Item> _items = new();
    
    [ObservableProperty]
    [NotifyCanExecuteChangedFor(nameof(DeleteCommand))]
    private Item? _selectedItem;
    
    [ObservableProperty]
    private bool _isLoading;
    
    public ItemsViewModel(IDataService dataService)
    {
        _dataService = dataService;
    }
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        IsLoading = true;
        try
        {
            var items = await _dataService.GetItemsAsync();
            Items = new ObservableCollection<Item>(items);
        }
        finally
        {
            IsLoading = false;
        }
    }
    
    [RelayCommand(CanExecute = nameof(CanDelete))]
    private async Task DeleteAsync()
    {
        if (SelectedItem == null) return;
        await _dataService.DeleteAsync(SelectedItem);
        Items.Remove(SelectedItem);
    }
    
    private bool CanDelete() => SelectedItem != null;
}
```

### 7.3. View — XAML

```xml
<Window>
    <Window.DataContext>
        <vm:ItemsViewModel/>   <!-- Or через DI -->
    </Window.DataContext>
    
    <Grid RowDefinitions="Auto,*,Auto">
        <Button Grid.Row="0" Content="Load" Command="{Binding LoadCommand}"/>
        
        <ListView Grid.Row="1"
                  ItemsSource="{Binding Items}"
                  SelectedItem="{Binding SelectedItem, Mode=TwoWay}">
            <ListView.ItemTemplate>
                <DataTemplate>
                    <StackPanel Orientation="Horizontal">
                        <TextBlock Text="{Binding Name}" Width="200"/>
                        <TextBlock Text="{Binding CreatedAt, StringFormat=d}"/>
                    </StackPanel>
                </DataTemplate>
            </ListView.ItemTemplate>
        </ListView>
        
        <Grid Grid.Row="2" Visibility="{Binding IsLoading, Converter={StaticResource BoolToVis}}">
            <ProgressBar IsIndeterminate="True"/>
        </Grid>
        
        <Button Grid.Row="3" Content="Delete" Command="{Binding DeleteCommand}"/>
    </Grid>
</Window>
```

### 7.4. Dependency Injection

```csharp
// App.xaml.cs
public partial class App : Application
{
    private IServiceProvider _services;
    
    public App()
    {
        var services = new ServiceCollection();
        services.AddSingleton<IDataService, DataService>();
        services.AddTransient<ItemsViewModel>();
        services.AddTransient<MainWindow>();
        _services = services.BuildServiceProvider();
    }
    
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);
        var window = _services.GetRequiredService<MainWindow>();
        window.Show();
    }
}
```

### 7.5. CommunityToolkit.Mvvm benefits

```csharp
// Без CT — manual INotifyPropertyChanged
public class OldVm : INotifyPropertyChanged
{
    private string _name = "";
    public string Name
    {
        get => _name;
        set
        {
            if (_name != value)
            {
                _name = value;
                OnPropertyChanged();
            }
        }
    }
    
    public event PropertyChangedEventHandler? PropertyChanged;
    protected void OnPropertyChanged([CallerMemberName] string? name = null) =>
        PropertyChanged?.Invoke(this, new(name));
}

// С CT — source generator
public partial class NewVm : ObservableObject
{
    [ObservableProperty]
    private string _name = "";
}
// 50+ lines становятся 1-2.
```

### 7.6. Messaging (loose coupling)

```csharp
// Send message
WeakReferenceMessenger.Default.Send(new ItemUpdatedMessage(item));

// Receive
WeakReferenceMessenger.Default.Register<ItemUpdatedMessage>(this, (recipient, message) =>
{
    Items.Refresh();
});

public record ItemUpdatedMessage(Item Item);
```

Decouples ViewModels — no direct references between them.

> [!question]- Интервью: зачем MVVM и тестируется ли он?
> 1) **Separation of concerns** — UI (View) от business logic (ViewModel) от domain (Model). 2) **Testability** — ViewModel не имеет UI references, can unit test directly. 3) **Designer / developer collaboration** — designer работает с XAML, developer с ViewModel. 4) **Reusability** — same ViewModel работает на разных Views. **Testing**: создаешь ViewModel в test, mock dependencies (IDataService через DI), exercise commands, assert state changes (PropertyChanged events). Critical: ViewModel **не должен иметь** UI references (Window/Page). Если нужно показать dialog — через service interface (IDialogService). Best practice 2026: CommunityToolkit.Mvvm источников генератор убирает 80% boilerplate.

---

## 8. Performance / startup

### 8.1. Startup times (cold)

```
| Framework | Cold start |
|-----------|------------|
| WinForms | ~50-100ms |
| WPF | ~200-500ms |
| WinUI 3 | ~300-600ms |
| MAUI Windows | ~500-1000ms |
| Avalonia | ~200-400ms |
| Avalonia Native AOT | ~50-100ms |
```

Native AOT dramatically improves Avalonia startup. **WinUI 3 — Native AOT c Windows App SDK 1.6** (сентябрь 2024): по замерам Microsoft (Contoso Camera sample) — до **~50% быстрее startup** и **~8x меньше размер пакета** (framework-dependent; в self-contained режиме ~2x меньше). **MAUI — NativeAOT для iOS / Mac Catalyst с .NET 9** (opt-in); MAUI Windows/Android остаются на JIT/R2R.

### 8.2. Memory footprint

```
| Framework | Working set |
|-----------|-------------|
| WinForms | 30-50 MB |
| WPF | 80-150 MB |
| WinUI 3 | 100-200 MB |
| MAUI | 150-300 MB (varies platform) |
| Avalonia | 60-100 MB |
| Avalonia AOT | 40-80 MB |
```

### 8.3. Rendering perf

```
WPF: software / GPU mixed, animation jank possible на complex UIs
WinUI 3: GPU-accelerated, optimized
MAUI: per-platform native renders
Avalonia: Skia GPU, consistent performance
```

### 8.4. Optimization tips

```
✅ Virtualize lists (VirtualizingStackPanel, ItemsRepeater)
✅ Compiled bindings (x:Bind в WinUI/MAUI)
✅ Avoid deep visual trees
✅ Profile with Visual Studio Diagnostic Tools
✅ Lazy-load tabs / pages
✅ DispatcherTimer.Tick — не каждые 16ms
✅ Image caching / async loading
✅ Reduce visual states transitions
```

---

## 9. Migration paths

### 9.1. WinForms → WPF

```
Pain points:
- Designer-driven → XAML markup
- Event-driven → MVVM mindset
- Manual binding → INPC + ObservableCollection
- Pixel layout → Grid/StackPanel/etc.

Effort: 30-70% rewrite typically.
Strategy: per-window migration, parallel apps возможно.
```

### 9.2. WPF → WinUI 3

```
Pain points:
- Namespace changes (xmlns)
- Some controls missing / renamed
- Resource system different (ThemeResource)
- Some APIs removed (no Drawing, fewer System.Windows.Controls)
- Dispatcher → DispatcherQueue

Effort: 20-40% rewrite (XAML mostly compatible).
Risk: target Windows 10 1809+ — кастомеры со старой Win lose support.
```

### 9.3. WPF → Avalonia

```
Pain points:
- Namespace (xmlns:avaloniaui)
- Triggers → behaviors
- Some property syntax differences
- Some controls renamed

Effort: 10-30% rewrite.
Strategy: easier чем WinUI 3 due to similar API.
Often: copy-paste XAML работает с minor edits.
```

### 9.4. WPF → MAUI

```
Pain points:
- Mobile-first paradigm (different layouts)
- Many controls renamed / missing
- Platform-specific code (Android/iOS specifics)
- Resource system different

Effort: 50-80% rewrite — substantial.
Use case: если нужен mobile, otherwise WPF/Avalonia stay better.
```

### 9.5. Xamarin.Forms → MAUI

```
Microsoft official migration tool.
Effort: 20-40% rewrite (mostly automated).
Benefits: better perf, modern .NET, single project, Hot Reload.
```

> [!question]- Интервью: миграция WPF → cross-platform?
> Variants: 1) **WPF → Avalonia** — самый простой путь, ~10-30% rewrite (XAML и API similar). Best для desktop-only cross-platform. 2) **WPF → MAUI** — 50-80% rewrite. Different layouts (mobile-first), platform-specific code. Use если нужен mobile. 3) **WPF → WinUI 3** — Windows-only modernization, 20-40% rewrite. Loses Win 7/8 support. **Strategy**: 1) Identify XAML/Code-behind — migrate XAML easier. 2) ViewModels mostly portable. 3) Native APIs need replacement (Windows specifics). 4) Test на target platforms early. 5) Per-window migration возможна. **Best path 2026**: Avalonia если cross-platform desktop важен; иначе stay в WPF.

---

## 10. Best practices

### 10.1. Architecture

- ✅ **MVVM** universal pattern.
- ✅ **CommunityToolkit.Mvvm** — source generators.
- ✅ **DI container** — Microsoft.Extensions.DependencyInjection.
- ✅ **Async/await** в commands (RelayCommand supports).
- ✅ **Service interfaces** для testability.
- ❌ Code-behind с business logic.
- ❌ Direct UI manipulation in ViewModel.

### 10.2. Performance

- ✅ **Virtualize** large lists.
- ✅ **Compiled bindings** (x:Bind в WinUI/MAUI).
- ✅ **Profile** с Visual Studio Diagnostic Tools.
- ✅ **Image caching** + async loading.
- ✅ **Lazy initialization** для expensive controls.
- ❌ Synchronous I/O в UI thread.
- ❌ Deep visual trees.

### 10.3. Cross-platform

- ✅ **Avalonia / MAUI** для cross-platform.
- ✅ **Platform-specific code** через `#if` или DI.
- ✅ **Test на каждой platform** early.
- ✅ **Native AOT** где supported (Avalonia; WinUI 3 c Windows App SDK 1.6; MAUI iOS/Mac Catalyst с .NET 9).
- ❌ Don't assume same behavior на всех platforms.

### 10.4. Не делай

- ❌ Mix old patterns (event handlers + MVVM).
- ❌ Tight coupling ViewModels (use messaging).
- ❌ Update UI from non-UI thread (use Dispatcher).
- ❌ Singleton state в desktop (test interference).
- ❌ Hardcoded strings без localization.

---

## 11. Decision tree

```
Что строишь?
│
├── Windows-only enterprise
│   ├── Modern Win 10/11 → WinUI 3
│   ├── Legacy support (Win 7+) → WPF
│   └── Simple LOB → WinForms (если existing) или WPF
│
├── Cross-platform desktop
│   ├── С mobile target → MAUI
│   ├── Desktop only, consistent look → Avalonia
│   ├── WPF migration → Avalonia
│   └── Web tech preferred → Blazor Hybrid (MAUI/WinUI)
│
├── Mobile + desktop
│   └── MAUI (single Microsoft option)
│
├── Web tech / Electron alternative
│   ├── Blazor Hybrid в MAUI
│   ├── Avalonia + WebView
│   └── Photino / WebView2 wrappers
│
└── Migration path
    ├── WinForms → WPF / Avalonia (modern)
    ├── WPF → Avalonia (cross-platform) или WinUI 3 (modern Windows)
    ├── UWP → WinUI 3 (official path)
    └── Xamarin.Forms → MAUI (Microsoft tool)
```

---

## 12. Common pitfalls

### 12.1. UI thread updates

```csharp
// ❌ Background thread updating UI
await Task.Run(() =>
{
    Items.Add(newItem);   // crash на WPF/Avalonia
});
```

**Фикс:** Dispatcher.Invoke (WPF) / DispatcherQueue (WinUI) / Dispatcher.UIThread (Avalonia):

```csharp
await Task.Run(async () =>
{
    var data = await LoadAsync();
    Application.Current.Dispatcher.Invoke(() => Items.Add(data));
});
```

Или используй `Task.ContinueWith(.., TaskScheduler.FromCurrentSynchronizationContext())`.

### 12.2. ObservableCollection vs List

```csharp
// ❌ List — UI не реагирует на changes
public List<Item> Items { get; } = new();
Items.Add(newItem);   // UI не updates

// ✅ ObservableCollection
public ObservableCollection<Item> Items { get; } = new();
Items.Add(newItem);   // UI auto-updates
```

### 12.3. PropertyChanged forgot

```csharp
// ❌ No PropertyChanged
private string _name;
public string Name
{
    get => _name;
    set => _name = value;   // UI не updates
}

// ✅ Implement INPC или use [ObservableProperty]
```

### 12.4. Memory leaks через events

```csharp
// ❌ ViewModel subscribes to long-living event
service.OnEvent += HandleEvent;
// ViewModel disposed но reference remains в service — leak
```

**Фикс:** unsubscribe in cleanup, или WeakEventManager.

### 12.5. Designer-time bindings

```xml
<!-- ❌ Crash в Designer (DataContext null) -->
<TextBlock Text="{Binding Name}"/>

<!-- ✅ Design-time data context -->
<TextBlock Text="{Binding Name}"
           d:DataContext="{d:DesignInstance Type=vm:MyVm}"/>
```

### 12.6. ViewModel ссылается на View

```csharp
// ❌ Anti-pattern
public class MainViewModel
{
    public MainWindow Window { get; set; }   // direct UI reference
    public void ShowDialog() => Window.ShowDialog();   // can't test
}

// ✅ Service abstraction
public class MainViewModel
{
    private IDialogService _dialog;
    public MainViewModel(IDialogService dialog) => _dialog = dialog;
    public void ShowDialog() => _dialog.Show();
}
```

### 12.7. Virtualization disabled

```xml
<!-- ❌ ScrollViewer disables virtualization для ListBox внутри -->
<ScrollViewer>
    <ListBox ItemsSource="{Binding HugeList}"/>
</ScrollViewer>
```

**Фикс:** ListBox sam scrolls, не оборачивай.

### 12.8. WinForms designer на .NET 5+

```
Visual Studio 2019 designer broken на .NET 5+ WinForms.
✅ Use VS 2022 17.0+
✅ Or: edit .Designer.cs manually
```

### 12.9. MAUI build slow

```
MAUI первый build very slow (10-30 min для iOS).
Cold incremental build all platforms — 1-3 min.
✅ Use single platform для daily dev (-t:Run -f net8.0-windows)
✅ CI builds full multi-platform
```

### 12.10. AOT не AOT-friendly libraries

```csharp
// ❌ Reflection-heavy library в AOT app
JsonConvert.SerializeObject(obj);   // Newtonsoft uses reflection
// AOT warning IL2026
```

**Фикс:** System.Text.Json + JsonSerializerContext.

> [!question]- Интервью: топ-3 ошибки в desktop app development?
> 1) **UI thread updates** — modify UI из background thread crashes app. Use Dispatcher.Invoke / DispatcherQueue.TryEnqueue. 2) **List vs ObservableCollection** — `List<T>` не triggers UI updates на Add/Remove. ObservableCollection raises CollectionChanged. 3) **Memory leaks через events** — ViewModel subscribes to long-living service event, ViewModel disposed but reference remains. Unsubscribe в cleanup или WeakEventManager. Бонус: ViewModel ссылается на View — кaнт unit test. Use IDialogService через DI.

---

## 13. Practice exercises

### 13.1. WPF + MVVM TODO app

```csharp
// Model
public record TodoItem(int Id, string Title, bool IsDone);

// Service
public interface ITodoService
{
    Task<List<TodoItem>> GetAllAsync();
    Task<TodoItem> AddAsync(string title);
    Task ToggleAsync(int id);
}

// ViewModel
public partial class TodoViewModel : ObservableObject
{
    private readonly ITodoService _service;
    
    [ObservableProperty]
    private ObservableCollection<TodoItem> _items = new();
    
    [ObservableProperty]
    private string _newTitle = "";
    
    public TodoViewModel(ITodoService service)
    {
        _service = service;
        LoadCommand.Execute(null);
    }
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        var items = await _service.GetAllAsync();
        Items = new ObservableCollection<TodoItem>(items);
    }
    
    [RelayCommand(CanExecute = nameof(CanAdd))]
    private async Task AddAsync()
    {
        var item = await _service.AddAsync(NewTitle);
        Items.Add(item);
        NewTitle = "";
    }
    
    private bool CanAdd() => !string.IsNullOrWhiteSpace(NewTitle);
    
    [RelayCommand]
    private async Task ToggleAsync(TodoItem item)
    {
        await _service.ToggleAsync(item.Id);
        await LoadAsync();
    }
}

// XAML
// <Grid RowDefinitions="Auto,*">
//     <Grid Grid.Row="0" ColumnDefinitions="*,Auto">
//         <TextBox Grid.Column="0" Text="{Binding NewTitle, UpdateSourceTrigger=PropertyChanged}"/>
//         <Button Grid.Column="1" Content="Add" Command="{Binding AddCommand}"/>
//     </Grid>
//     <ListView Grid.Row="1" ItemsSource="{Binding Items}">
//         <ListView.ItemTemplate>
//             <DataTemplate>
//                 <CheckBox IsChecked="{Binding IsDone}" Content="{Binding Title}"
//                           Command="{Binding DataContext.ToggleCommand, RelativeSource={RelativeSource AncestorType=ListView}}"
//                           CommandParameter="{Binding}"/>
//             </DataTemplate>
//         </ListView.ItemTemplate>
//     </ListView>
// </Grid>
```

### 13.2. Avalonia weather app

```csharp
public partial class WeatherViewModel : ObservableObject
{
    private readonly IWeatherService _weatherService;
    
    [ObservableProperty]
    private string _city = "Moscow";
    
    [ObservableProperty]
    private WeatherInfo? _weather;
    
    [ObservableProperty]
    private bool _isLoading;
    
    public WeatherViewModel(IWeatherService service) => _weatherService = service;
    
    [RelayCommand]
    private async Task LoadAsync()
    {
        IsLoading = true;
        try
        {
            Weather = await _weatherService.GetWeatherAsync(City);
        }
        catch (HttpRequestException ex)
        {
            // Show error через IDialogService
        }
        finally
        {
            IsLoading = false;
        }
    }
}

// XAML — Avalonia
// <StackPanel>
//     <TextBox Text="{Binding City}"/>
//     <Button Content="Get weather" Command="{Binding LoadCommand}"/>
//     <Grid IsVisible="{Binding IsLoading}">
//         <ProgressBar IsIndeterminate="True"/>
//     </Grid>
//     <StackPanel IsVisible="{Binding Weather, Converter={x:Static ObjectConverters.IsNotNull}}">
//         <TextBlock Text="{Binding Weather.Description}"/>
//         <TextBlock Text="{Binding Weather.Temperature, StringFormat='{}{0}°C'}"/>
//     </StackPanel>
// </StackPanel>
```

### 13.3. Cross-framework MVVM ViewModel

```csharp
// ViewModel works для WPF / WinUI / MAUI / Avalonia identically
public partial class CounterViewModel : ObservableObject
{
    [ObservableProperty]
    private int _count;
    
    [RelayCommand]
    private void Increment() => Count++;
    
    [RelayCommand]
    private void Decrement() => Count--;
    
    [RelayCommand(CanExecute = nameof(CanReset))]
    private void Reset() => Count = 0;
    
    private bool CanReset() => Count != 0;
    
    partial void OnCountChanged(int value)
    {
        ResetCommand.NotifyCanExecuteChanged();
    }
}

// Same ViewModel reused across:
// - WPF MainWindow.xaml
// - WinUI 3 Page.xaml
// - MAUI ContentPage.xaml
// - Avalonia Window.xaml
// Each XAML has different namespace но same bindings.
```

---

## 14. Что читать дальше

1. **CommunityToolkit.Mvvm docs** — github.com/CommunityToolkit/dotnet.
2. **WPF Microsoft Docs**.
3. **WinUI 3 / Windows App SDK docs**.
4. **MAUI Microsoft Docs** + sample apps.
5. **Avalonia documentation** — docs.avaloniaui.net.
6. **Books**: "Pro WPF in C# 2010" (still relevant), "Mobile App Development with .NET MAUI".

---

## 15. См. также

- [[oop|OOP]] — class hierarchies
- [[delegates-events|Delegates and Events]] — INotifyPropertyChanged
- [[csharp-vs-other-langs|C# vs Other]] — vs Flutter/RN
- [[source-generators|Source Generators]] — CommunityToolkit.Mvvm
- WPF — learn.microsoft.com/dotnet/desktop/wpf
- WinUI 3 — learn.microsoft.com/windows/apps/winui
- MAUI — learn.microsoft.com/dotnet/maui
- Avalonia — avaloniaui.net

---

## 16. Reading list

- **Microsoft Docs — WPF** — learn.microsoft.com/dotnet/desktop/wpf
- **Microsoft Docs — WinUI 3** — learn.microsoft.com/windows/apps/winui/winui3
- **Microsoft Docs — .NET MAUI** — learn.microsoft.com/dotnet/maui
- **Microsoft Docs — WinForms** — learn.microsoft.com/dotnet/desktop/winforms
- **Avalonia docs** — docs.avaloniaui.net
- **Avalonia from WPF guide** — docs.avaloniaui.net/docs/guides/upgrade-from-wpf-or-uwp
- **CommunityToolkit.Mvvm** — github.com/CommunityToolkit/dotnet
- **MAUI samples** — github.com/dotnet/maui-samples
- **Josh Smith — MVVM articles** — joshsmithonwpf.wordpress.com
- **Christian Moser — WPF Tutorial** — wpf-tutorial.com
- **Frank Krueger — Avalonia patterns** — praeclarum.org
