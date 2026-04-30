---
tags: [csharp, desktop, wpf, avalonia, uno, maui, cross-platform, ui, xaml]
level: Senior
date: 2026-04-30
---

# Desktop UI Frameworks для C#

> Сравнение и гайд по cross-platform desktop frameworks. Closes WPF (Windows-only) и расширяет на cross-platform: Avalonia, Uno Platform, .NET MAUI desktop. Когда что выбирать, миграция с WPF, XAML отличия, performance, deployment.

---

## Что это, зачем и когда

### Проблема: WPF только Windows

```
WPF — отличный фреймворк, но:
- Только Windows (нет Linux / macOS)
- Не получает major updates с .NET 5
- Не AOT-friendly
- Browser нет (только desktop)
```

### Современные альтернативы

| Framework | Windows | Linux | macOS | iOS | Android | Web (WASM) |
|-----------|---------|-------|-------|-----|---------|------------|
| **WPF** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| **WinForms** | ✅ | ⚠️ Mono | ⚠️ Mono | ❌ | ❌ | ❌ |
| **Avalonia** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Uno Platform** | ✅ | ✅ (Skia) | ✅ | ✅ | ✅ | ✅ |
| **.NET MAUI** | ✅ (WinUI) | ❌ | ✅ | ✅ | ✅ | ⚠️ Blazor Hybrid |
| **WinUI 3** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

### Что выбирать в 2026

| Сценарий | Выбор |
|----------|-------|
| Existing WPF app, всё хорошо, только Windows | Остаться на WPF |
| Хочу cross-platform desktop (Win + Linux + macOS) | **Avalonia** ✨ |
| Mobile + Desktop (iOS + Android + Win + Mac) | **MAUI** или Avalonia |
| Web + Desktop (одна codebase) | **Uno Platform** |
| Legacy WinForms migration | **WinForms** на .NET 8/9 (поддерживается) |
| Modern Windows-only (Fluent UI, WinUI 3) | **WinUI 3** + Windows App SDK |
| Mobile-first | MAUI / Uno |
| Native iOS/Android только | MAUI |

---

## 1. WPF — Windows-only классика

WPF (Windows Presentation Foundation) — флагман для Windows desktop с 2006 года.

### Где сейчас (2026)

- **Поддерживается** в .NET 8/9/10 — не deprecated
- **Не получает major features** — стабилен но не развивается
- **Лучшая Windows integration** — DirectX rendering, ICC, accessibility
- **Огромная экосистема** — DevExpress, Telerik, Syncfusion, MahApps.Metro

### Когда оставаться на WPF

✅ **Хорошо если:**
- Existing WPF app работает в production
- Только Windows нужен (corporate enterprise)
- Сложные controls с DirectX (charts, CAD, 3D)
- Нет ресурсов на migration

❌ **Не подходит:**
- Cross-platform (Linux, macOS) требуется
- Web / mobile в roadmap

### Когда мигрировать с WPF

- Нужен cross-platform → Avalonia (XAML similar)
- Хочется modern look + WinUI features → WinUI 3
- Mobile + Desktop → MAUI

> [!info] У Vitaly есть детальный wpf-production.md
> См. [WPF Production](../Infrastructure/wpf-production.md) — production patterns, MVVM, performance, deployment.

---

## 2. Avalonia — cross-platform XAML

[avaloniaui.net](https://avaloniaui.net) — самый зрелый cross-platform XAML framework. Часто называют "WPF for Linux/Mac".

### Зачем

- **Same XAML mindset** — миграция с WPF простая
- **Truly cross-platform** — Win/Linux/macOS/iOS/Android/WebAssembly
- **Skia rendering** — быстрая отрисовка через GPU
- **Active development** — v11 (2024), v12 (2025), регулярные релизы
- **Companies используют:** JetBrains Rider, GitHub Copilot for CLI, Schneider Electric

### Установка

```bash
dotnet new install Avalonia.Templates
dotnet new avalonia.app -o MyApp
cd MyApp
dotnet run
```

### Базовый MainWindow.axaml

```xml
<Window xmlns="https://github.com/avaloniaui"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:vm="clr-namespace:MyApp.ViewModels"
        x:Class="MyApp.MainWindow"
        Title="My App" Width="800" Height="600">
    <Window.DataContext>
        <vm:MainViewModel />
    </Window.DataContext>
    
    <Grid RowDefinitions="Auto,*,Auto">
        <Menu Grid.Row="0">
            <MenuItem Header="_File">
                <MenuItem Header="_Open" Command="{Binding OpenCommand}" />
                <MenuItem Header="E_xit" Command="{Binding ExitCommand}" />
            </MenuItem>
        </Menu>
        
        <ListBox Grid.Row="1" 
                 ItemsSource="{Binding Items}"
                 SelectedItem="{Binding SelectedItem}">
            <ListBox.ItemTemplate>
                <DataTemplate>
                    <TextBlock Text="{Binding Name}" />
                </DataTemplate>
            </ListBox.ItemTemplate>
        </ListBox>
        
        <StackPanel Grid.Row="2" Orientation="Horizontal" Margin="10">
            <Button Content="Add" Command="{Binding AddCommand}" />
            <Button Content="Remove" Command="{Binding RemoveCommand}" />
        </StackPanel>
    </Grid>
</Window>
```

### MVVM ViewModel

```csharp
using CommunityToolkit.Mvvm.ComponentModel;
using CommunityToolkit.Mvvm.Input;

public partial class MainViewModel : ObservableObject
{
    [ObservableProperty]
    private ObservableCollection<Item> _items = [];
    
    [ObservableProperty]
    private Item? _selectedItem;
    
    [RelayCommand]
    private async Task Open()
    {
        var topLevel = TopLevel.GetTopLevel(GetCurrentWindow());
        var files = await topLevel!.StorageProvider.OpenFilePickerAsync(new()
        {
            Title = "Open file",
            AllowMultiple = false
        });
        
        if (files.Count > 0)
        {
            var file = files[0];
            // ...
        }
    }
    
    [RelayCommand]
    private void Add()
    {
        Items.Add(new Item { Name = $"Item {Items.Count + 1}" });
    }
    
    [RelayCommand(CanExecute = nameof(CanRemove))]
    private void Remove() => Items.Remove(SelectedItem!);
    
    private bool CanRemove() => SelectedItem is not null;
    
    partial void OnSelectedItemChanged(Item? value) => RemoveCommand.NotifyCanExecuteChanged();
}
```

### WPF → Avalonia migration

| WPF | Avalonia |
|-----|---------|
| `xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"` | `xmlns="https://github.com/avaloniaui"` |
| `*.xaml` | `*.axaml` (Avalonia XAML) |
| `Visibility="Collapsed"` | `IsVisible="False"` |
| `Background="#FF0000"` | `Background="Red"` |
| `RoutedCommand` | `ICommand` (CommunityToolkit.Mvvm) |
| `<Grid Grid.Row=...>` | Same syntax |
| `Storyboard` (animations) | Animations есть, но другой API |
| `INotifyPropertyChanged` | Same (но CommunityToolkit упрощает) |

### Диалоги через StorageProvider

```csharp
var topLevel = TopLevel.GetTopLevel(this);
var files = await topLevel.StorageProvider.OpenFilePickerAsync(new()
{
    Title = "Open file",
    FileTypeFilter = new[]
    {
        new FilePickerFileType("Images")
        {
            Patterns = new[] { "*.png", "*.jpg" }
        }
    }
});

// Save dialog
var file = await topLevel.StorageProvider.SaveFilePickerAsync(new()
{
    Title = "Save",
    DefaultExtension = "txt",
    SuggestedFileName = "untitled"
});

// Folder dialog
var folder = await topLevel.StorageProvider.OpenFolderPickerAsync(new()
{
    Title = "Select folder"
});
```

### Платформо-специфика

```csharp
// Avalonia.Platform — runtime detection
if (OperatingSystem.IsWindows())
{
    // Windows-specific
}
else if (OperatingSystem.IsLinux())
{
    // Linux
}
else if (OperatingSystem.IsMacOS())
{
    // macOS
}

// Native API через P/Invoke (см. interop-pinvoke.md)
[LibraryImport("libnotify.so.4")]
private static partial void notify_init([MarshalAs(UnmanagedType.LPUTF8Str)] string app_name);
```

### Hot Reload

Avalonia 11+ поддерживает hot reload:

```bash
dotnet watch
# Изменения в XAML — мгновенно в running app

```

### Theming

```xml
<!-- App.axaml -->
<Application.Styles>
    <FluentTheme />
    <!-- или -->
    <SimpleTheme />
</Application.Styles>
```

Кастомный theme:
```xml
<Style Selector="Button">
    <Setter Property="Background" Value="#3498db" />
    <Setter Property="CornerRadius" Value="4" />
</Style>
```

CSS-like selectors:
```xml
<Style Selector="Button.danger">
    <Setter Property="Background" Value="Red" />
</Style>

<Button Classes="danger" Content="Delete" />
```

---

## 3. Uno Platform — single codebase для всего

[platform.uno](https://platform.uno) — пишешь код **один раз**, deploy на **все платформы** включая Web (WASM).

### Зачем

- **WinUI 3 syntax everywhere** — Microsoft-style XAML
- **Web (WASM)** — single codebase desktop + browser
- **Mobile + desktop + web** — максимальное reuse
- **Skia рендеринг** опционально (как Avalonia) или native controls

### Targets

```
- WinUI on Windows
- iOS (native UIKit) или Skia
- Android (native Views) или Skia
- macOS (Catalyst или Skia)
- Linux (Skia или GTK)
- WebAssembly (Skia в browser canvas)
- Tizen, Apple TV, custom
```

### Создание проекта

```bash
dotnet new install Uno.Templates
dotnet new unoapp -o MyApp -preset blank
```

Структура:
```
MyApp/
├── MyApp/                    # Cross-platform code
│   ├── App.xaml.cs
│   └── MainPage.xaml
├── MyApp.Mobile/             # iOS + Android targets
├── MyApp.Skia.WPF/           # Windows desktop (Skia)
├── MyApp.Skia.Linux.FrameBuffer/  # Linux
├── MyApp.Wasm/               # WebAssembly
└── MyApp.Windows/            # Windows native (WinUI)
```

### Плюсы

- True one codebase для всего
- WinUI 3 syntax — будущее Microsoft UI
- Web target — никто другой не делает (кроме Blazor)
- Multi-platform host model

### Минусы vs Avalonia

- Сложнее setup (multiple host projects)
- Native targets зависят от platform-specific tooling
- Web (WASM) — большой bundle size (3-15 MB)

### Когда Uno

- **Web + Desktop одна codebase** (где Avalonia не предлагает хорошего web)
- **Команда уже знает WinUI** / UWP
- **Microsoft ecosystem-aligned**

---

## 4. .NET MAUI — Microsoft cross-platform

[dotnet.microsoft.com/maui](https://dotnet.microsoft.com/apps/maui) — преемник Xamarin.Forms. Mobile-first с desktop поддержкой.

### Targets

| Platform | UI |
|----------|-----|
| iOS | UIKit |
| Android | Views |
| **Windows** | WinUI 3 |
| **macOS** | Mac Catalyst (iOS на Mac) |

> [!warning] Linux НЕ поддерживается официально
> Microsoft не поддерживает MAUI на Linux. Если нужен Linux desktop — Avalonia или Uno.

### Когда MAUI

✅ **Хорошо для:**
- Mobile-first с desktop как bonus
- Microsoft / Azure ecosystem
- Команда знает Xamarin (упрощённая миграция)
- iOS + Android главное target

❌ **Не подходит:**
- Linux нужен
- Только desktop (Avalonia/Uno более подходят)
- Web target нужен

### Создание

```bash
dotnet new maui -n MyApp
cd MyApp
dotnet build -t:Run -f net10.0-android
dotnet build -t:Run -f net10.0-ios
dotnet build -t:Run -f net10.0-windows10.0.19041
```

### XAML

```xml
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui"
             xmlns:x="http://schemas.microsoft.com/winfx/2009/xaml"
             x:Class="MyApp.MainPage">
    <ScrollView>
        <VerticalStackLayout Spacing="25" Padding="30">
            <Label Text="Hello, MAUI!" 
                   FontSize="32"
                   HorizontalOptions="Center" />
            
            <Button Text="Click me"
                    Clicked="OnButtonClicked"
                    HorizontalOptions="Center" />
            
            <Entry Placeholder="Enter text" />
            
            <CollectionView ItemsSource="{Binding Items}">
                <CollectionView.ItemTemplate>
                    <DataTemplate>
                        <Label Text="{Binding Name}" />
                    </DataTemplate>
                </CollectionView.ItemTemplate>
            </CollectionView>
        </VerticalStackLayout>
    </ScrollView>
</ContentPage>
```

### Platform-specific

```csharp
// Через partial classes
public partial class MainPage : ContentPage
{
#if IOS
    private async Task DoIOSStuff() { /* ... */ }
#endif
#if ANDROID
    private async Task DoAndroidStuff() { /* ... */ }
#endif
}

// Через DependencyService
public interface IDeviceInfo
{
    string GetDeviceName();
}

#if ANDROID
public class DeviceInfo : IDeviceInfo
{
    public string GetDeviceName() => Android.OS.Build.Model;
}
#endif

#if IOS
public class DeviceInfo : IDeviceInfo
{
    public string GetDeviceName() => UIKit.UIDevice.CurrentDevice.Name;
}
#endif
```

### Native interop (.NET 6+ — multi-targeting)

```csharp
using Microsoft.Maui.Devices;

DeviceInfo.Current.Platform        // Android, iOS, WinUI, MacCatalyst
DeviceInfo.Current.Idiom           // Phone, Tablet, Desktop
DeviceInfo.Current.Manufacturer
DeviceInfo.Current.Model
DeviceInfo.Current.VersionString
```

### Platform-specific styling

```xml
<Button Text="Click me">
    <Button.Style>
        <Style TargetType="Button">
            <Setter Property="BackgroundColor">
                <Setter.Value>
                    <OnPlatform x:TypeArguments="Color">
                        <On Platform="iOS" Value="Blue" />
                        <On Platform="Android" Value="Green" />
                        <On Platform="WinUI" Value="Red" />
                    </OnPlatform>
                </Setter.Value>
            </Setter>
        </Style>
    </Button.Style>
</Button>
```

### Blazor Hybrid в MAUI

Используй Blazor компоненты внутри MAUI app:

```csharp
// MauiProgram.cs
builder.Services.AddMauiBlazorWebView();

// MainPage.xaml
<BlazorWebView x:Name="blazorWebView" HostPage="wwwroot/index.html">
    <BlazorWebView.RootComponents>
        <RootComponent Selector="#app" ComponentType="{x:Type local:Main}" />
    </BlazorWebView.RootComponents>
</BlazorWebView>
```

### Свежие фишки .NET 10+

- **Native AOT** для iOS / macOS
- **Hot Reload** in MAUI
- **CarouselView** improvements
- **Performance** в CollectionView

---

## 5. WinUI 3 — modern Windows-only

[learn.microsoft.com/windows/apps/winui](https://learn.microsoft.com/windows/apps/winui)

WinUI 3 — modern UI framework Microsoft. Преемник UWP / WPF для Windows.

### Когда

- ✅ Microsoft Store apps на Windows 10/11
- ✅ Modern Fluent design (Mica, acrylic effects)
- ✅ Лучшая Windows 11 integration
- ❌ Cross-platform НЕ поддерживается

### Базовый

```bash
dotnet new winui -o MyApp
```

```xml
<Window x:Class="MyApp.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation">
    <StackPanel Padding="20">
        <TextBlock Text="Hello WinUI 3!" FontSize="24" />
        
        <Button Click="Button_Click" Content="Click me" />
        
        <Expander Header="Details">
            <TextBlock Text="Hidden content" />
        </Expander>
        
        <NavigationView>
            <NavigationView.MenuItems>
                <NavigationViewItem Content="Home" Icon="Home" />
                <NavigationViewItem Content="Settings" Icon="Setting" />
            </NavigationView.MenuItems>
        </NavigationView>
    </StackPanel>
</Window>
```

### vs WPF

| | WPF | WinUI 3 |
|--|-----|---------|
| Look | Classic Windows | Fluent, modern |
| Cross-platform | ❌ | ❌ |
| New features | Слабо развивается | Active |
| Mica / acrylic | ❌ | ✅ |
| Migration from WPF | Native — переписать | Можно использовать XamlIslands |

---

## 6. Сравнение производительности

| Framework | Startup time | Memory | Bundle size | Performance UI |
|-----------|--------------|--------|-------------|----------------|
| **WPF** | ~500ms | ~80 MB | ~5 MB | Excellent (DirectX) |
| **Avalonia** | ~300ms | ~50 MB | ~30 MB | Excellent (Skia/GPU) |
| **Uno (Skia)** | ~400ms | ~70 MB | ~40 MB | Good |
| **Uno (WinUI)** | ~300ms | ~60 MB | ~10 MB | Excellent |
| **MAUI Windows** | ~700ms | ~100 MB | ~15 MB | Good |
| **WinUI 3** | ~300ms | ~50 MB | ~5 MB | Excellent |
| **Blazor Hybrid** | ~1500ms | ~150 MB | ~50 MB | OK (WebView) |

> [!info] Native AOT меняет картину
> С Native AOT — startup в разы быстрее, bundle меньше. Avalonia 11+ поддерживает AOT.

---

## 7. MVVM — общий паттерн

Все frameworks используют MVVM. Лучшая библиотека — **CommunityToolkit.Mvvm**.

```bash
dotnet add package CommunityToolkit.Mvvm
```

### Source generator-based ViewModel

```csharp
public partial class UserViewModel : ObservableObject
{
    [ObservableProperty]
    private string _firstName = "";
    
    [ObservableProperty]
    private string _lastName = "";
    
    [ObservableProperty]
    [NotifyPropertyChangedFor(nameof(FullName))]
    private bool _isActive;
    
    public string FullName => $"{FirstName} {LastName}";
    
    [RelayCommand(CanExecute = nameof(CanSave))]
    private async Task SaveAsync()
    {
        // ...
    }
    
    private bool CanSave() => !string.IsNullOrEmpty(FirstName);
    
    partial void OnFirstNameChanged(string value)
    {
        // Invoke after property changed
        SaveCommand.NotifyCanExecuteChanged();
    }
}
```

Generates:
```csharp
public partial class UserViewModel
{
    public string FirstName 
    { 
        get => _firstName; 
        set 
        {
            if (SetProperty(ref _firstName, value))
            {
                OnFirstNameChanged(value);
            }
        }
    }
    
    public IRelayCommand SaveCommand { get; }
    
    static UserViewModel()
    {
        // Generated init
    }
}
```

### Messaging — communication между VM

```csharp
public class UserSelectedMessage : ValueChangedMessage<User>
{
    public UserSelectedMessage(User user) : base(user) { }
}

// Sender
WeakReferenceMessenger.Default.Send(new UserSelectedMessage(user));

// Receiver
WeakReferenceMessenger.Default.Register<UserSelectedMessage>(this, (recipient, message) =>
{
    var user = message.Value;
    // ...
});
```

См. также [WPF Production](../Infrastructure/wpf-production.md) — детальный MVVM паттерн.

---

## 8. Real-world патерн: structure большого приложения

```
MyApp/
├── MyApp.Core/                # Domain / Logic — без UI зависимостей
│   ├── Models/
│   ├── Services/
│   └── ViewModels/             # ⚠️ Платформо-независимые VM!
├── MyApp.Avalonia/             # Avalonia desktop UI
│   ├── App.axaml
│   ├── Views/
│   └── Program.cs
├── MyApp.MAUI/                 # MAUI mobile UI (если нужно)
│   ├── App.xaml
│   ├── Pages/
│   └── MauiProgram.cs
└── MyApp.Tests/
```

ViewModels — в `Core` проекте. UI references VM из там, но VM не знает об UI. Можно подключать новые UI projects (Avalonia, MAUI) переиспользуя VM.

```csharp
// MyApp.Core
public class MainViewModel : ObservableObject
{
    [ObservableProperty]
    private string _greeting = "Hello!";
    
    // Никаких WPF / Avalonia / MAUI типов!
}

// MyApp.Avalonia
<Window xmlns="https://github.com/avaloniaui"
        xmlns:vm="using:MyApp.Core.ViewModels">
    <Window.DataContext>
        <vm:MainViewModel />
    </Window.DataContext>
    <TextBlock Text="{Binding Greeting}" />
</Window>

// MyApp.MAUI
<ContentPage xmlns="http://schemas.microsoft.com/dotnet/2021/maui">
    <ContentPage.BindingContext>
        <vm:MainViewModel xmlns:vm="clr-namespace:MyApp.Core.ViewModels;assembly=MyApp.Core" />
    </ContentPage.BindingContext>
    <Label Text="{Binding Greeting}" />
</ContentPage>
```

---

## 9. Deployment

### WPF / WinUI 3 / WinForms

```bash
# Self-contained single-file
dotnet publish -c Release -r win-x64 --self-contained true \
    /p:PublishSingleFile=true \
    /p:IncludeNativeLibrariesForSelfExtract=true
```

### Avalonia

```bash
# Все платформы из одного solution
dotnet publish MyApp -c Release -r win-x64 --self-contained -p:PublishSingleFile=true
dotnet publish MyApp -c Release -r linux-x64 --self-contained -p:PublishSingleFile=true
dotnet publish MyApp -c Release -r osx-arm64 --self-contained -p:PublishSingleFile=true
```

### MAUI

```bash
# iOS
dotnet publish -f net10.0-ios -c Release

# Android
dotnet publish -f net10.0-android -c Release

# Windows
dotnet publish -f net10.0-windows10.0.19041 -c Release \
    /p:RuntimeIdentifier=win-x64 /p:Configuration=Release
```

iOS требует Mac для signing. Android — APK / AAB.

### Native AOT

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

```bash
dotnet publish -c Release -r linux-x64
# Native binary, ~30-50 MB вместо 100+ MB

```

Avalonia 11+ — best AOT support среди cross-platform UI.

---

## 10. Common Pitfalls

### 1. Платформо-специфические thread / dispatcher

```csharp
// ❌ Прямой доступ к UI не из UI thread
private async void ButtonClicked()
{
    await Task.Run(() =>
    {
        Title = "New title";  // throws — wrong thread!
    });
}

// ✅ Через Dispatcher
await Task.Run(() =>
{
    // ... work ...
    
    Dispatcher.UIThread.Post(() =>  // Avalonia
    {
        Title = "New title";
    });
});

// MAUI
MainThread.BeginInvokeOnMainThread(() => Title = "New title");

// WPF
Application.Current.Dispatcher.Invoke(() => Title = "New title");
```

### 2. Memory leaks от events

```csharp
// ❌ Subscribe но никогда unsubscribe
public void Init()
{
    SomeService.Updated += OnUpdated;
}

// VM остаётся в памяти потому что service её держит

// ✅ WeakReferenceMessenger или явный unsubscribe в Dispose
```

### 3. Async void в command handlers

```csharp
// ❌ Если throw — приложение crash'нется
public async void OnClick(...) { ... }

// ✅ async Task + RelayCommand
[RelayCommand]
private async Task OnClick() { ... }
```

### 4. Большие images в memory

```csharp
// ❌ Загрузка 4K image целиком
var img = new Bitmap("photo.jpg");

// ✅ Resize на load
var img = Bitmap.DecodeToWidth(stream, targetWidth: 800);
```

### 5. UI freezes при синхронных операциях

```csharp
// ❌ Блокирует UI
private void OnClick()
{
    Thread.Sleep(5000);
    // или
    var data = httpClient.GetStringAsync(url).Result;  // .Result!
}

// ✅
[RelayCommand]
private async Task OnClickAsync(CancellationToken ct)
{
    var data = await httpClient.GetStringAsync(url, ct);
}
```

### 6. Avalonia XAML отличия от WPF

```xml
<!-- WPF -->
<Visibility="Collapsed" />
<TextBlock TextWrapping="Wrap" />
<Grid Background="#FF0000" />

<!-- Avalonia -->
<IsVisible="False" />
<TextBlock TextWrapping="Wrap" />  <!-- same -->
<Grid Background="Red" />  <!-- short colors -->
```

### 7. MAUI на Windows — нужен Visual Studio

`dotnet build -t:Run -f net10.0-windows...` требует VS Workloads. CLI alone — не достаточно для Windows target.

---

## 11. Best Practices

- **Avalonia** — default cross-platform desktop в 2026
- **MAUI** — mobile-first с desktop как bonus
- **Uno** — single codebase desktop + web
- **WPF** — оставайся если уже на нём, нужен только Windows
- **WinUI 3** — modern Windows-only с Fluent design
- **MVVM через CommunityToolkit.Mvvm** — source generators reduce boilerplate
- **ViewModels в shared Core project** — переиспользование между UI projects
- **Async/await везде**, никогда `.Result` / `.Wait()`
- **CancellationToken в commands** — для cleanup
- **Native AOT для production** — startup speed + bundle size
- **Hot Reload** в development — productivity
- **Platform abstractions** через `IPlatformService` (зависит от target)
- **Test ViewModels отдельно** — без UI зависимости

---

## 12. Когда что — резюме

| Задача | Best choice |
|--------|-------------|
| Internal corporate tool, only Windows | **WPF** или WinForms |
| Modern Windows app с Fluent UI | **WinUI 3** |
| Cross-platform desktop (Win+Linux+Mac) | **Avalonia** |
| Mobile + Desktop (iOS+Android+Win+Mac) | **MAUI** или Avalonia |
| Web + Desktop одна codebase | **Uno Platform** |
| Полноценная PWA | Blazor (Server / WASM) |
| Embedded UI (Raspberry Pi) | Avalonia с framebuffer |
| Game UI | Avalonia или Unity |

---

## См. также

- [WPF Production](../Infrastructure/wpf-production.md) — детальный WPF deep-dive
- [Functional C#](functional-csharp.md) — Records / patterns в ViewModels
- [Native AOT](../AspNetCore/native-aot.md) — для desktop deployment
- [Project Setup](../Infrastructure/project-setup.md) — solution structure
- [Reflection и Expression Trees](reflection-expression-trees.md) — MVVM bindings под капотом
- [Modern C# Features](modern-features.md) — records для immutable models

## Reading list

- **Avalonia docs** — docs.avaloniaui.net
- **Uno Platform docs** — platform.uno/docs
- **MAUI docs** — learn.microsoft.com/dotnet/maui
- **WinUI 3 docs** — learn.microsoft.com/windows/apps/winui
- **CommunityToolkit.Mvvm** — github.com/CommunityToolkit/dotnet
- **Avalonia from WPF migration** — docs.avaloniaui.net/docs/getting-started/wpf
- **JetBrains Rider** — лучший IDE для cross-platform .NET UI dev
- **Avalonia samples** — github.com/AvaloniaUI/Avalonia.Samples
