# 07 — Native Windows App

> ByPotomac SDK · WinUI 3 / C# Desktop Application

**Source:** `C:\Users\SohaibAli\Documents\PotomacAnalyst`

---

## Technology Stack

| Technology | Version | Purpose |
|-----------|---------|---------|
| **WinUI 3** | Windows App SDK 1.8 | UI framework |
| **.NET** | 8.0 | Runtime |
| **Target** | `net8.0-windows10.0.19041.0` | Min Windows version |
| **Packaging** | MSIX | Distribution |
| **DI** | `Microsoft.Extensions.DependencyInjection` | Service container |
| **MVVM** | `CommunityToolkit.Mvvm 8.4.0` | MVVM framework |
| **HTTP** | `Microsoft.Extensions.Http` | HttpClient factory |
| **Platform** | x86, x64, ARM64 | Target architectures |

---

## Architecture Overview

```
┌───────────────────────────────────────────────────────────────┐
│                    PotomacAnalyst App                          │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  App.xaml.cs (Entry Point)                               │ │
│  │  • DI container setup                                   │ │
│  │  • Service registration (singletons)                    │ │
│  │  • Window creation                                      │ │
│  └──────────────────────────┬──────────────────────────────┘ │
│                              │                                │
│  ┌──────────────────────────▼──────────────────────────────┐ │
│  │  MainWindow.xaml                                         │ │
│  │  • NavigationView shell                                 │ │
│  │  • Mica backdrop (dark theme)                           │ │
│  │  • Potomac branding (Rajdhani + Quicksand fonts)        │ │
│  │  • Yellow accent (#FEC00F)                              │ │
│  └──────────────────────────┬──────────────────────────────┘ │
│                              │                                │
│  ┌──────────────────────────▼──────────────────────────────┐ │
│  │  Pages (16 total)                                        │ │
│  │                                                         │ │
│  │  Auth: Login · Register · ForgotPassword                │ │
│  │  App:  Dashboard · Chat · AflGenerator · Autopilot      │ │
│  │        Backtest · Content · DeckGenerator · Developer    │ │
│  │        KnowledgeBase · Researcher · ReverseEngineer     │ │
│  │        Settings · Skills                                │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Services Layer                                          │ │
│  │                                                         │ │
│  │  ApiService      → HTTP client (all API calls)          │ │
│  │  AuthService     → Login/register/logout                │ │
│  │  SessionService  → Token + user state management        │ │
│  └─────────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

---

## Dependency Injection Setup

```csharp
// App.xaml.cs
public partial class App : Application
{
    public static IServiceProvider Services { get; private set; }
    
    protected override void OnLaunched(LaunchActivatedEventArgs args)
    {
        var services = new ServiceCollection();
        
        // Core services (all singletons)
        services.AddSingleton<HttpClient>();
        services.AddSingleton<ApiService>();
        services.AddSingleton<SessionService>();
        services.AddSingleton<AuthService>();
        
        Services = services.BuildServiceProvider();
        
        m_window = new MainWindow();
        m_window.Activate();
    }
}
```

Services are resolved in pages:

```csharp
public sealed partial class ChatPage : Page
{
    private readonly ApiService _api;
    private readonly SessionService _session;
    
    public ChatPage()
    {
        InitializeComponent();
        _api = App.Services.GetRequiredService<ApiService>();
        _session = App.Services.GetRequiredService<SessionService>();
    }
}
```

---

## Services

### ApiService (`Services/ApiService.cs`)

The central HTTP gateway to the backend.

```csharp
public class ApiService
{
    private readonly HttpClient _httpClient;
    private readonly SessionService _sessionService;
    private readonly JsonSerializerOptions _jsonOptions;
    
    private const string BASE_URL = 
        "https://potomac-analyst-workbench-production.up.railway.app";
    
    public ApiService(HttpClient httpClient, SessionService sessionService)
    {
        _httpClient = httpClient;
        _sessionService = sessionService;
        _jsonOptions = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.SnakeCaseLower,
            PropertyNameCaseInsensitive = true
        };
    }
```

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `GetAsync<T>` | `Task<T> GetAsync<T>(string endpoint)` | GET request with auth |
| `PostAsync<TReq,TRes>` | `Task<TRes> PostAsync<TReq,TRes>(string endpoint, TReq body)` | POST with JSON body |
| `PutAsync<TReq,TRes>` | `Task<TRes> PutAsync<TReq,TRes>(string endpoint, TReq body)` | PUT with JSON body |
| `DeleteAsync` | `Task DeleteAsync(string endpoint)` | DELETE request |
| `PostFileAsync<T>` | `Task<T> PostFileAsync<T>(string endpoint, StorageFile file)` | Multipart file upload |
| `PostBytesAsync<T>` | `Task<T> PostBytesAsync<T>(string endpoint, byte[] bytes, string filename)` | Upload raw bytes |
| `StreamAsync<T>` | `IAsyncEnumerable<T> StreamAsync<T>(string endpoint, object body)` | SSE streaming |

**Auth header injection:**

```csharp
private HttpRequestMessage CreateRequest(HttpMethod method, string endpoint)
{
    var request = new HttpRequestMessage(method, $"{BASE_URL}{endpoint}");
    
    if (_sessionService.IsLoggedIn)
    {
        request.Headers.Authorization = 
            new AuthenticationHeaderValue("Bearer", _sessionService.Token);
    }
    
    return request;
}
```

### Stream Parsing

The `StreamAsync` method implements the **Vercel AI SDK Data Stream Protocol** parser:

```csharp
public async IAsyncEnumerable<StreamEvent> StreamChatAsync(
    string endpoint, ChatRequest body)
{
    var content = new StringContent(
        JsonSerializer.Serialize(body, _jsonOptions),
        Encoding.UTF8, "application/json");
    
    var response = await _httpClient.PostAsync($"{BASE_URL}{endpoint}", content);
    response.EnsureSuccessStatusCode();
    
    var stream = await response.Content.ReadAsStreamAsync();
    using var reader = new StreamReader(stream);
    
    while (!reader.EndOfStream)
    {
        var line = await reader.ReadLineAsync();
        if (string.IsNullOrWhiteSpace(line)) continue;
        
        var colonIdx = line.IndexOf(':');
        if (colonIdx < 1) continue;
        
        var typeCode = line[..colonIdx];
        var payload = line[(colonIdx + 1)..];
        
        yield return typeCode switch
        {
            "0" => new StreamEvent("text", 
                       JsonSerializer.Deserialize<string>(payload)),
            "3" => new StreamEvent("error", 
                       JsonSerializer.Deserialize<string>(payload)),
            "9" => new StreamEvent("tool_call", payload),
            "a" => new StreamEvent("tool_result", payload),
            "d" => new StreamEvent("finish", payload),
            _ => new StreamEvent("unknown", payload)
        };
    }
}
```

### SessionService (`Services/SessionService.cs`)

In-memory session state:

```csharp
public class SessionService
{
    public string? Token { get; private set; }
    public UserProfile? CurrentUser { get; private set; }
    public bool IsLoggedIn => !string.IsNullOrEmpty(Token);
    
    public event Action? SessionChanged;
    
    public void SetSession(string token, UserProfile user)
    {
        Token = token;
        CurrentUser = user;
        SessionChanged?.Invoke();
    }
    
    public void ClearSession()
    {
        Token = null;
        CurrentUser = null;
        SessionChanged?.Invoke();
    }
}
```

### AuthService (`Services/AuthService.cs`)

```csharp
public class AuthService
{
    private readonly ApiService _api;
    private readonly SessionService _session;
    
    public async Task<LoginResponse> LoginAsync(string email, string password)
    {
        var response = await _api.PostAsync<LoginRequest, LoginResponse>(
            "/auth/login",
            new LoginRequest { Email = email, Password = password });
        
        _session.SetSession(response.AccessToken, new UserProfile
        {
            Id = response.UserId,
            Email = response.Email,
            // ...
        });
        
        return response;
    }
    
    public async Task<LoginResponse> RegisterAsync(
        string email, string password, string? name)
    {
        var response = await _api.PostAsync<RegisterRequest, LoginResponse>(
            "/auth/register",
            new RegisterRequest { Email = email, Password = password, Name = name });
        
        _session.SetSession(response.AccessToken, /* ... */);
        return response;
    }
    
    public void Logout()
    {
        _session.ClearSession();
    }
}
```

---

## Pages

### Auth Pages

| Page | File | Description |
|------|------|-------------|
| Login | `Pages/Auth/LoginPage.xaml` | Email + password login, links to register |
| Register | `Pages/Auth/RegisterPage.xaml` | Registration form |
| Forgot Password | `Pages/Auth/ForgotPasswordPage.xaml` | Password reset |

### App Pages

| Page | File | Description |
|------|------|-------------|
| Dashboard | `Pages/DashboardPage.xaml` | Overview with cards and stats |
| Chat | `Pages/ChatPage.xaml` | AI chat with streaming, tool displays |
| AFL Generator | `Pages/AflGeneratorPage.xaml` | American Funds Letter generator |
| Autopilot | `Pages/AutopilotPage.xaml` | Automated analysis workflows |
| Backtest | `Pages/BacktestPage.xaml` | Strategy backtesting |
| Content | `Pages/ContentPage.xaml` | Content generation |
| Deck Generator | `Pages/DeckGeneratorPage.xaml` | PPTX presentation builder |
| Developer | `Pages/DeveloperPage.xaml` | Developer tools/debug |
| Knowledge Base | `Pages/KnowledgeBasePage.xaml` | Document upload, brain management |
| Researcher | `Pages/ResearcherPage.xaml` | Multi-source research |
| Reverse Engineer | `Pages/ReverseEngineerPage.xaml` | Strategy reverse engineering |
| Settings | `Pages/SettingsPage.xaml` | App settings, API keys |
| Skills | `Pages/SkillsPage.xaml` | Manage AI skills |

---

## Navigation

The app uses a **NavigationView** shell pattern:

```xml
<!-- MainWindow.xaml -->
<NavigationView PaneDisplayMode="Left"
                IsBackButtonVisible="Auto"
                Background="Transparent">
    <NavigationView.MenuItems>
        <NavigationViewItem Content="Dashboard" Tag="Dashboard" Icon="Home" />
        <NavigationViewItem Content="Chat" Tag="Chat" Icon="Chat" />
        <NavigationViewItem Content="Research" Tag="Researcher" Icon="Find" />
        <NavigationViewItem Content="AFL" Tag="AflGenerator" Icon="Document" />
        <!-- ... -->
    </NavigationView.MenuItems>
    
    <Frame x:Name="ContentFrame" />
</NavigationView>
```

Navigation is handled in code-behind:

```csharp
private void NavView_SelectionChanged(NavigationView sender, 
    NavigationViewSelectionChangedEventArgs args)
{
    var tag = (args.SelectedItem as NavigationViewItem)?.Tag?.ToString();
    ContentFrame.Navigate(tag switch
    {
        "Dashboard" => typeof(DashboardPage),
        "Chat" => typeof(ChatPage),
        "Researcher" => typeof(ResearcherPage),
        // ...
    });
}
```

---

## Branding & Theming

| Element | Value |
|---------|-------|
| **Primary Color** | `#FEC00F` (Potomac Yellow) |
| **Background** | Mica (system backdrop) |
| **Theme** | Dark |
| **Heading Font** | Rajdhani |
| **Body Font** | Quicksand |
| **Icon Style** | Fluent System Icons |

---

## Models

DTOs use `snake_case` JSON serialization to match the Python backend:

```csharp
public class LoginRequest
{
    public string Email { get; set; }
    public string Password { get; set; }
}

public class LoginResponse
{
    public string AccessToken { get; set; }
    public string TokenType { get; set; }
    public string UserId { get; set; }
    public string Email { get; set; }
    public int ExpiresIn { get; set; }
}

public class UserProfile
{
    public string Id { get; set; }
    public string Email { get; set; }
    public string? Name { get; set; }
    public string? Nickname { get; set; }
    public bool IsAdmin { get; set; }
    public bool IsActive { get; set; }
}

public class ChatRequest
{
    public List<ChatMessage> Messages { get; set; }
    public string? ConversationId { get; set; }
    public string? Model { get; set; }
    public string? SystemPrompt { get; set; }
    public bool BrainEnabled { get; set; } = true;
    public bool ToolsEnabled { get; set; } = true;
    public List<string>? Skills { get; set; }
}

public class StreamEvent
{
    public string Type { get; set; }
    public string? Text { get; set; }
    public JsonElement? Data { get; set; }
}
```

---

## Build & Run

```powershell
# Restore packages
dotnet restore

# Build
dotnet build -c Release

# Run
dotnet run

# Package as MSIX
dotnet publish -c Release -r win-x64
```

**Prerequisites:**
- Windows 10 (build 19041+) or Windows 11
- .NET 8 SDK
- Windows App SDK 1.8
- Visual Studio 2022 (recommended)

---

*Next: [08 — Database Schema](08-DATABASE.md)*
