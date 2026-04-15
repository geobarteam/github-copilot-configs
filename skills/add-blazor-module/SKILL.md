---
name: add-blazor-module
description: "Use when creating a new self-contained UI module (feature folder) in a Blazor WebAssembly Presentation app. Scaffolds the module folder structure with Model, Pages, Services, and ViewModels following MVVM. Use for: new feature modules, new CRUD modules, new admin/backoffice sections."
argument-hint: "Describe the module, e.g. 'Customer module with list and detail pages'"
---

# Add Blazor Module — WASM Presentation

Scaffold a new self-contained module in a Blazor WebAssembly Presentation app. Each module is a feature folder with its own Models, Pages, Services, and ViewModels following MVVM.

> **Scope**: This skill targets standalone Blazor WASM Presentation apps (e.g., BackOffice, Admin) under `src/Host/<App>/Presentation/`. For the Presentation RCL with BFF/Refit pattern, use `/add-blazor-page` instead.

## Prerequisites

- Identify the **Presentation app** — which `src/Host/<App>/Presentation/` project the module belongs to.
- Identify the **reference module** — find an existing module in the same Presentation app and read its structure. Mirror its patterns.
- The **API endpoints** the module will call should already exist (or will be created separately via `/add-endpoint`).

## Module Folder Structure

```
src/Host/<App>/Presentation/<Module>/
├── Model/
│   └── <Entity>Model.cs
├── Pages/
│   ├── <Page>.razor
│   └── <Page>.razor.cs
├── Services/
│   ├── I<Module>Service.cs
│   └── <Module>Service.cs
└── ViewModels/
    ├── I<Page>ViewModel.cs
    └── <Page>ViewModel.cs
```

Cross-module shared components go in `Shared/` at the Presentation root. Module-specific components go in `<Module>/Components/`.

## Workflow

```
1. READ    — read the reference module across all folders (Model, Pages, Services, ViewModels)
2. SCAFFOLD — create the module folder structure with all files listed below
3. DI      — register services and ViewModels in BuilderExtensions.cs or Program.cs
4. VERIFY  — dotnet build → confirm compiles
5. 🛑 STOP — wait for user approval before adding tests or wiring API calls
```

---

## 1. Model

```csharp
namespace <PresentationNamespace>.<Module>.Model;

// Read model (immutable)
public record <Entity>Model(string Name, string Description);

// Write/form model (mutable, with validation)
using System.ComponentModel.DataAnnotations;

public class <Action><Entity>Model
{
    [Required]
    public string Name { get; set; } = string.Empty;
}
```

---

## 2. Service Interface + Implementation

The service wraps API calls and returns models. ViewModels depend on the interface only.

```csharp
namespace <PresentationNamespace>.<Module>.Services;

public interface I<Module>Service
{
    Task<List<<Entity>Model>> GetAllAsync();
    Task SaveAsync(<Action><Entity>Model model);
}
```

```csharp
namespace <PresentationNamespace>.<Module>.Services;

using System.Net.Http.Json;

public class <Module>Service(HttpClient httpClient) : I<Module>Service
{
    private readonly HttpClient _httpClient = httpClient
        ?? throw new ArgumentNullException(nameof(httpClient));

    public async Task<List<<Entity>Model>> GetAllAsync()
    {
        return await _httpClient.GetFromJsonAsync<List<<Entity>Model>>("api/<module-kebab>")
            ?? [];
    }

    public async Task SaveAsync(<Action><Entity>Model model)
    {
        var response = await _httpClient.PostAsJsonAsync("api/<module-kebab>", model);
        response.EnsureSuccessStatusCode();
    }
}
```

---

## 3. ViewModel Interface

Every ViewModel exposes state properties, a `StateChanged` event, and async operations.

```csharp
namespace <PresentationNamespace>.<Module>.ViewModels;

public interface I<Page>ViewModel
{
    bool IsLoading { get; }
    string ErrorMessage { get; }
    List<<Entity>Model> Items { get; }

    event Action? StateChanged;

    Task InitializeAsync();
}
```

---

## 4. ViewModel Implementation

All business logic, data loading, error handling, and navigation live in the ViewModel.

```csharp
namespace <PresentationNamespace>.<Module>.ViewModels;

using <PresentationNamespace>.<Module>.Model;
using <PresentationNamespace>.<Module>.Services;

public class <Page>ViewModel : I<Page>ViewModel
{
    private readonly I<Module>Service _service;
    private readonly IGlobalExceptionHandler _exceptionHandler;
    private readonly ISnackbar _snackbar;

    public bool IsLoading { get; private set; }
    public string ErrorMessage { get; private set; } = string.Empty;
    public List<<Entity>Model> Items { get; private set; } = [];

    public event Action? StateChanged;

    public <Page>ViewModel(
        I<Module>Service service,
        IGlobalExceptionHandler exceptionHandler,
        ISnackbar snackbar)
    {
        _service = service;
        _exceptionHandler = exceptionHandler;
        _snackbar = snackbar;
    }

    public async Task InitializeAsync()
    {
        await _exceptionHandler.SafeExecuteAsync(async () =>
        {
            IsLoading = true;
            NotifyStateChanged();

            Items = await _service.GetAllAsync();

            IsLoading = false;
            NotifyStateChanged();
        }, "Failed to load data");
    }

    private void NotifyStateChanged() => StateChanged?.Invoke();
}
```

**ViewModel rules**:
- Set `IsLoading = true` before async work, `false` after (in the lambda, not `finally`)
- Wrap async work in `_exceptionHandler.SafeExecuteAsync()` with a user-friendly message
- Call `NotifyStateChanged()` whenever state properties change
- Never throw from `InitializeAsync`
- Use `ISnackbar` for user feedback on actions:
  ```csharp
  _snackbar.Add("Saved successfully", Severity.Success);
  ```

---

## 5. Page — Razor View

The `.razor` file contains only UI markup and data binding. No business logic.

```razor
@page "/<module-kebab>"
@inject I<Page>ViewModel ViewModel

<PageTitle>@Localizer["PageTitle"]</PageTitle>

@if (ViewModel.IsLoading)
{
    <MudProgressCircular Indeterminate="true" />
}
else if (!string.IsNullOrEmpty(ViewModel.ErrorMessage))
{
    <MudAlert Severity="Severity.Error">@ViewModel.ErrorMessage</MudAlert>
}
else
{
    <MudDataGrid T="<Entity>Model"
                 Items="@ViewModel.Items"
                 SortMode="SortMode.Multiple"
                 Filterable
                 Hideable>
        <Columns>
            <PropertyColumn Property="x => x.Name" Title='@Localizer["Name"]' />
        </Columns>
        <PagerContent>
            <MudDataGridPager T="<Entity>Model" />
        </PagerContent>
    </MudDataGrid>
}
```

---

## 6. Page — Code-Behind

Code-behind is minimal: inject ViewModel, subscribe to `StateChanged`, delegate to ViewModel.

```csharp
namespace <PresentationNamespace>.<Module>.Pages;

using Microsoft.AspNetCore.Components;

public partial class <Page> : ComponentBase, IDisposable
{
    [Inject]
    public I<Page>ViewModel ViewModel { get; set; } = default!;

    protected override async Task OnInitializedAsync()
    {
        ViewModel.StateChanged += StateHasChanged;
        await ViewModel.InitializeAsync();
    }

    public void Dispose()
    {
        ViewModel.StateChanged -= StateHasChanged;
    }
}
```

**Code-behind rules**:
- Subscribe to `StateChanged` in `OnInitializedAsync`, unsubscribe in `Dispose`
- Implement `IDisposable`
- No business logic, no API calls, no state management
- Event handlers delegate to ViewModel: `private async Task HandleSave() => await ViewModel.SaveAsync();`

---

## 7. DI Registration

Register services and ViewModels in the Presentation app's DI setup (`BuilderExtensions.cs` or `Program.cs`):

```csharp
// Services — Scoped
builder.Services.AddScoped<I<Module>Service, <Module>Service>();

// ViewModels — Scoped
builder.Services.AddScoped<I<Page>ViewModel, <Page>ViewModel>();
```

If the project uses Scrutor suffix scanning, verify the suffixes `Service` and `ViewModel` are included in the scan — new registrations may then be automatic.

---

## Form Page Pattern (Add/Edit)

For pages with `EditForm` submission, use `DataAnnotationsValidator` and bind to a form model with `[Required]` attributes:

```razor
<EditForm Model="@ViewModel.FormModel" OnValidSubmit="@HandleValidSubmit">
    <DataAnnotationsValidator />
    <MudTextField @bind-Value="ViewModel.FormModel.Name"
                  Label="Name"
                  For="@(() => ViewModel.FormModel.Name)" />
    <MudButton ButtonType="ButtonType.Submit" Disabled="@ViewModel.IsLoading">Save</MudButton>
</EditForm>
```

```csharp
// In code-behind
private async Task HandleValidSubmit() => await ViewModel.HandleValidSubmitAsync();
```

---

## Authorization

For pages that require authentication:

```razor
@page "/secure-page"
@attribute [Authorize]
```

For conditional UI based on auth state, use `<AuthorizeView>`.

---

## Naming Conventions

| Artifact | Convention | Example |
|----------|-----------|---------|
| Module folder | PascalCase feature name | `Customer/` |
| Page | `<Page>.razor` + `<Page>.razor.cs` | `CustomerPage.razor` |
| ViewModel | `<Page>ViewModel` / `I<Page>ViewModel` | `CustomerPageViewModel` |
| Model | `<Entity>Model` | `CustomerModel` |
| Service | `<Module>Service` / `I<Module>Service` | `CustomerService` |
| Route | kebab-case | `/customer`, `/customer-history` |
| Async methods | Suffix `Async` | `LoadDataAsync()` |
| Private fields | `_camelCase` | `_service` |

---

## Review Checklist

Before completing the module:

- [ ] Module folder is self-contained (Model, Pages, Services, ViewModels)
- [ ] Code-behind is minimal — delegates to ViewModel
- [ ] ViewModel has interface + implements `StateChanged` event
- [ ] Code-behind subscribes/unsubscribes `StateChanged` and implements `IDisposable`
- [ ] Error handling via `IGlobalExceptionHandler.SafeExecuteAsync`
- [ ] User feedback via `ISnackbar`
- [ ] Loading state properly toggled (`IsLoading`)
- [ ] No direct API calls in code-behind or `.razor`
- [ ] DI registration added
- [ ] MudBlazor components used for UI consistency
# Blazor WebAssembly - Coding Patterns & Standards


## Architecture Overview

This Blazor WebAssembly project follows **MVVM (Model-View-ViewModel)** pattern with **Clean Architecture** principles.

---

## 📁 File Organization

### Module-Centric Structure (Preferred)

All Blazor applications should follow a **module-centric** structure where each feature/domain is organized as a self-contained module with its own Pages, Services, ViewModels, and Models.

#### BackOffice Presentation (Blazor WebAssembly) - Reference Pattern
```
src/Host/{Application}/Presentation/
├── {ModuleName}/                    # Each module is a self-contained folder
│   ├── Model/                       # Module-specific models
│   │   └── {Entity}Model.cs
│   ├── Pages/                       # Razor pages for this module
│   │   ├── {Page}.razor
│   │   └── {Page}.razor.cs
│   ├── Services/                    # Module-specific services & API clients
│   │   ├── I{Module}Service.cs
│   │   └── {Module}Service.cs
│   └── ViewModels/                  # ViewModels for module pages
│       ├── I{Page}ViewModel.cs
│       └── {Page}ViewModel.cs
├── Shared/                          # Cross-module shared components
│   ├── Components/
│   ├── Extensions/
│   ├── Localization/
│   ├── Models/
│   └── Services/
├── Layout/                          # App layout components
├── Resources/                       # Localization resources
└── wwwroot/                         # Static assets
```

**Example - Customer Module:**
```
Customer/
├── Model/
│   └── CustomerModel.cs
├── Pages/
│   ├── CustomerPage.razor
│   ├── CustomerPage.razor.cs
│   ├── CustomerHistoryPage.razor
│   └── CustomerHistoryPage.razor.cs
├── Services/
│   └── CustomerService.cs
└── ViewModels/
    ├── CustomerPageViewModel.cs
    └── CustomerHistoryViewModel.cs
```


### Key Principles

1. **Self-Contained Modules**: Each module should be independently understandable
2. **Colocation**: Keep related files together (Page + ViewModel + Service + Model)
3. **Minimal Cross-Module Dependencies**: Modules should rarely reference each other
4. **Shared Components**: Global components go in `Shared/`, module-specific in `{Module}/Components/`
5. **Global Services**: Cross-cutting services (Culture, SEO, Auth) stay in root `Services/` folder

### Legacy Structure (Avoid for New Development)
```
# ❌ AVOID - Flat Pages structure
Components/
├── Pages/
│   ├── Home.razor
│   ├── About.razor
│   ├── Contact.razor
│   ├── Pricing.razor
│   ├── Register/         # Sub-folder for complex feature
│   │   └── ...
│   └── ViewModels/       # ViewModels mixed at Pages level
│       └── ...
```

### Components Structure (Within Shared or Module)
```
Components/
├── MyComponent/
│   ├── MyComponent.razor
│   ├── MyComponent.razor.cs
│   └── ViewModels/
│       ├── IMyComponentViewModel.cs
│       └── MyComponentViewModel.cs
```

---

## ?? MVVM Pattern Rules

### 1. **Code-Behind (.razor.cs) Responsibilities**

Code-behind files should be **minimal** and only handle:
- ? Component lifecycle hooks (`OnInitializedAsync`, `OnParametersSetAsync`)
- ? Dependency injection of ViewModels
- ? Event subscription/unsubscription (`IDisposable`)
- ? Delegating calls to ViewModels
- ? NO business logic
- ? NO API calls
- ? NO state management

**Example Pattern:**
```csharp
public partial class MyPage : ComponentBase, IDisposable
{
    [Inject] 
    public IMyPageViewModel ViewModel { get; set; } = default!;

    protected override async Task OnInitializedAsync()
    {
        ViewModel.StateChanged += StateHasChanged;
        await ViewModel.InitializeAsync();
    }

    private async Task HandleSomeAction(SomeModel model)
    {
        await ViewModel.HandleActionAsync(model);
    }

    public void Dispose()
    {
        ViewModel.StateChanged -= StateHasChanged;
    }
}
```

### 2. **ViewModel Responsibilities**

ViewModels contain **all business logic**:
- ? State management (properties)
- ? Data loading and API calls
- ? Business logic and validation
- ? Navigation logic
- ? Error handling
- ? Dependency injection of services
- ? `StateChanged` event for UI updates

**Required Pattern:**
```csharp
public interface IMyPageViewModel
{
    // State properties
    bool IsLoading { get; }
    string ErrorMessage { get; }
    List<MyModel> Items { get; }
    
    // State change notification
    event Action? StateChanged;
    
    // Async operations
    Task InitializeAsync();
    Task LoadDataAsync();
    Task SaveAsync(MyModel model);
}

public class MyPageViewModel : IMyPageViewModel
{
    private readonly IMyService _service;
    private readonly IGlobalExceptionHandler _exceptionHandler;
    private readonly ISnackbar _snackbar;
    private readonly NavigationManager _navigationManager;

    public bool IsLoading { get; private set; }
    public string ErrorMessage { get; private set; } = string.Empty;
    public List<MyModel> Items { get; private set; } = new();
    
    public event Action? StateChanged;

    public MyPageViewModel(
        IMyService service,
        IGlobalExceptionHandler exceptionHandler,
        ISnackbar snackbar,
        NavigationManager navigationManager)
    {
        _service = service;
        _exceptionHandler = exceptionHandler;
        _snackbar = snackbar;
        _navigationManager = navigationManager;
    }

    public async Task InitializeAsync()
    {
        await _exceptionHandler.SafeExecuteAsync(async () =>
        {
            IsLoading = true;
            NotifyStateChanged();

            await LoadDataAsync();
            
            IsLoading = false;
            NotifyStateChanged();
        }, "Failed to initialize page");
    }

    public async Task LoadDataAsync()
    {
        Items = (await _service.GetItemsAsync()).ToList();
        NotifyStateChanged();
    }

    private void NotifyStateChanged()
    {
        StateChanged?.Invoke();
    }
}
```

### 3. **Razor View (.razor) Responsibilities**

Views should only contain:
- ? UI markup (HTML/Razor components)
- ? Data binding to ViewModel properties
- ? Event handlers that call ViewModel methods
- ? NO business logic
- ? NO direct service calls

**Example:**
```razor
@page "/mypage"
@inject IMyPageViewModel ViewModel

<MudContainer>
    @if (ViewModel.IsLoading)
    {
        <MudProgressCircular Indeterminate="true" />
    }
    else
    {
        @foreach (var item in ViewModel.Items)
        {
            <MudCard>
                <MudCardContent>@item.Name</MudCardContent>
                <MudCardActions>
                    <MudButton OnClick="@(() => HandleEdit(item))">Edit</MudButton>
                </MudCardActions>
            </MudCard>
        }
    }
</MudContainer>

@code {
    private async Task HandleEdit(MyModel item)
    {
        await ViewModel.EditItemAsync(item);
    }
}
```

---

## ?? State Management Pattern

### StateChanged Event Pattern
All ViewModels must implement state change notifications:

```csharp
public event Action? StateChanged;

private void NotifyStateChanged()
{
    StateChanged?.Invoke();
}
```

Code-behind subscribes in `OnInitializedAsync`:
```csharp
protected override async Task OnInitializedAsync()
{
    ViewModel.StateChanged += StateHasChanged;
    await ViewModel.InitializeAsync();
}
```

And unsubscribes in `Dispose`:
```csharp
public void Dispose()
{
    ViewModel.StateChanged -= StateHasChanged;
}
```

---

## ?? Naming Conventions

### Files
- **Pages**: `{PageName}.razor`, `{PageName}.razor.cs`
- **ViewModels**: `{PageName}ViewModel.cs`, `I{PageName}ViewModel.cs`
- **Models**: `{EntityName}Model.cs`
- **Services**: `{Purpose}Service.cs`, `I{Purpose}Service.cs`

### Classes
- **ViewModels**: `{PageName}ViewModel` (e.g., `CustomerPageViewModel`)
- **Interfaces**: `I{ClassName}` (e.g., `ICustomerPageViewModel`)
- **Models**: `{Entity}Model` (e.g., `CustomerModel`)

### Properties & Methods
- **Properties**: PascalCase (e.g., `IsLoading`, `ErrorMessage`)
- **Private fields**: `_camelCase` (e.g., `_service`, `_navigationManager`)
- **Async methods**: End with `Async` (e.g., `LoadDataAsync`, `SaveAsync`)
- **Event handlers (code-behind)**: Start with `On` or `Handle` (e.g., `OnCustomerAdded`, `HandleSaveClick`)

---

## ?? Dependency Injection

### Registration
ViewModels and services are registered in `BuilderExtensions.cs` or `Program.cs`:

```csharp
// ViewModels - Scoped
builder.Services.AddScoped<IMyPageViewModel, MyPageViewModel>();

// Services - Scoped
builder.Services.AddScoped<IMyService, MyService>();

// Singletons for global state
builder.Services.AddSingleton<ISessionStateService, SessionStateService>();
```

### Injection in Code-Behind
```csharp
[Inject] 
public IMyPageViewModel ViewModel { get; set; } = default!;

[Inject] 
public NavigationManager Navigation { get; set; } = default!;
```

### Injection in ViewModel
```csharp
public MyViewModel(
    IService1 service1,
    IService2 service2,
    ILogger<MyViewModel> logger)
{
    _service1 = service1;
    _service2 = service2;
    _logger = logger;
}
```

---

## ?? UI Component Patterns

### MudBlazor Components
Always use MudBlazor components for consistency:
- `MudButton`, `MudTextField`, `MudSelect`, etc.
- `MudDialog` for confirmations
- `MudSnackbar` for notifications (via `ISnackbar`)
- `MudProgressCircular` for loading indicators

### Loading States
```razor
@if (ViewModel.IsLoading)
{
    <MudProgressCircular Indeterminate="true" />
}
else
{
    <!-- Content here -->
}
```

### Error Handling
```razor
@if (!string.IsNullOrEmpty(ViewModel.ErrorMessage))
{
    <MudAlert Severity="Severity.Error">@ViewModel.ErrorMessage</MudAlert>
}
```

---

## ?? Authentication & Authorization

### Page-Level Authorization
```razor
@page "/secure-page"
@attribute [Authorize]
```

### Conditional UI Based on Auth
```razor
<AuthorizeView>
    <Authorized>
        <!-- Content for authenticated users -->
    </Authorized>
    <NotAuthorized>
        <!-- Content for anonymous users -->
    </NotAuthorized>
</AuthorizeView>
```

### Authentication State in ViewModel
```csharp
private readonly AuthenticationStateProvider _authStateProvider;

public async Task InitializeAsync()
{
    var authState = await _authStateProvider.GetAuthenticationStateAsync();
    var isAuthenticated = authState.User.Identity?.IsAuthenticated ?? false;
    
    if (!isAuthenticated)
    {
        _navigationManager.NavigateTo("/login");
        return;
    }
    
    // Continue with initialization
}
```

---

## ?? Error Handling Pattern

### Global Exception Handler
Always use `IGlobalExceptionHandler.SafeExecuteAsync`:

```csharp
await _exceptionHandler.SafeExecuteAsync(async () =>
{
    // Code that might throw exceptions
    await SomeRiskyOperation();
}, "User-friendly error message if operation fails");
```

### Snackbar Notifications
```csharp
_snackbar.Add("Operation successful!", Severity.Success);
_snackbar.Add("Something went wrong", Severity.Error);
_snackbar.Add("Please note...", Severity.Warning);
_snackbar.Add("Information message", Severity.Info);
```

---

## ?? Data Validation

### ViewModel Validation
```csharp
public async Task SaveAsync(MyModel model)
{
    // Validate
    if (string.IsNullOrWhiteSpace(model.Name))
    {
        _snackbar.Add("Name is required", Severity.Error);
        return;
    }
    
    // Save
    await _service.SaveAsync(model);
    _snackbar.Add("Saved successfully", Severity.Success);
}
```

### Form Validation (with EditForm)
```razor
<EditForm Model="@ViewModel.CurrentModel" OnValidSubmit="@HandleValidSubmit">
    <DataAnnotationsValidator />
    <ValidationSummary />
    
    <MudTextField @bind-Value="ViewModel.CurrentModel.Name" 
                  Label="Name" 
                  For="@(() => ViewModel.CurrentModel.Name)" />
    
    <MudButton ButtonType="ButtonType.Submit">Save</MudButton>
</EditForm>
```

---

## ?? Testing Considerations

### ViewModel Testing
ViewModels should be **easily testable** without Blazor components:

```csharp
[Fact]
public async Task LoadDataAsync_ShouldLoadItems()
{
    // Arrange
    var mockService = new Mock<IMyService>();
    mockService.Setup(s => s.GetItemsAsync()).ReturnsAsync(new List<MyModel>());
    var viewModel = new MyViewModel(mockService.Object, ...);
    
    // Act
    await viewModel.LoadDataAsync();
    
    // Assert
    Assert.NotNull(viewModel.Items);
}
```

---

## ? Code Review Checklist

Before committing, verify:

- [ ] Code-behind is minimal (< 50 lines typically)
- [ ] All business logic is in ViewModel
- [ ] ViewModel has an interface (`I{ViewModel}`)
- [ ] ViewModel implements `StateChanged` event
- [ ] Code-behind subscribes/unsubscribes to `StateChanged`
- [ ] Async methods end with `Async`
- [ ] Dependencies injected via constructor (ViewModel) or `[Inject]` (code-behind)
- [ ] Error handling uses `IGlobalExceptionHandler.SafeExecuteAsync`
- [ ] User feedback via `ISnackbar`
- [ ] No direct API calls in code-behind or .razor files
- [ ] Loading states properly handled
- [ ] Proper logging with `ILogger`

---

## ?? Quick Reference: Creating a New Page

1. **Create files:**
   ```
   Pages/MyPage/
   ??? MyPage.razor
   ??? MyPage.razor.cs
   ??? ViewModels/
       ??? IMyPageViewModel.cs
       ??? MyPageViewModel.cs
   ```

2. **Define interface:**
   ```csharp
   public interface IMyPageViewModel
   {
       bool IsLoading { get; }
       event Action? StateChanged;
       Task InitializeAsync();
   }
   ```

3. **Implement ViewModel:**
   ```csharp
   public class MyPageViewModel : IMyPageViewModel
   {
       // Implementation...
   }
   ```

4. **Code-behind:**
   ```csharp
   public partial class MyPage : ComponentBase, IDisposable
   {
       [Inject] public IMyPageViewModel ViewModel { get; set; }
       
       protected override async Task OnInitializedAsync()
       {
           ViewModel.StateChanged += StateHasChanged;
           await ViewModel.InitializeAsync();
       }
       
       public void Dispose()
       {
           ViewModel.StateChanged -= StateHasChanged;
       }
   }
   ```

5. **Register in DI:**
   ```csharp
   builder.Services.AddScoped<IMyPageViewModel, MyPageViewModel>();
   ```

6. **Create view:**
   ```razor
   @page "/mypage"
   @inject IMyPageViewModel ViewModel
   
   <!-- UI markup here -->
   ```

---

## ?? Summary

**Remember:**
- **Separation of Concerns**: UI ? Code-behind ? ViewModel ? Services
- **Testability**: ViewModels should be unit-testable
- **Consistency**: Follow established patterns for maintainability
- **MVVM**: View knows about ViewModel, ViewModel doesn't know about View

This pattern ensures **scalable, maintainable, and testable** Blazor applications.