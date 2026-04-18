---
name: add-blazor-page
description: "Use when adding a new Blazor page, dialog, or ViewModel to {{SolutionName}} Presentation layer. Covers ViewModel interface + implementation, ServiceClient, Refit client method, Model record, .razor page with code-behind, MudBlazor components, localization, and error handling. Use for: new pages, new dialogs, new list/detail views, new forms."
argument-hint: "Describe the page, e.g. 'list page for prescriptions with search and add dialog'"
---

# Add Blazor Page — Presentation Layer

> **Scope: Blazor WASM + Client (BFF).** This skill is for the Blazor WASM project where the Presentation RCL calls the Client host API via Refit (`BffServiceClients.AddBffServiceClients()` / `CookieHandler`). For standalone WASM apps with direct `HttpClient` services (no Client host), use the **`/add-blazor-module`** skill instead.

Add a new Blazor page with ViewModel, ServiceClient, and Refit integration. UI uses **MudBlazor** components and **IStringLocalizer** for localization.

## File Inventory (per page)

For a new page in feature `<Feature>`:

| File | Location |
|------|----------|
| `I<Feature>ViewModel.cs` | `src/Presentation/<Feature>/ViewModels/` |
| `<Feature>ViewModel.cs` | `src/Presentation/<Feature>/ViewModels/` |
| `I<Feature>ServiceClient.cs` | `src/Presentation/<Feature>/ServiceClients/` |
| `<Feature>ServiceClient.cs` | `src/Presentation/<Feature>/ServiceClients/` |
| `<Entity>Model.cs` | `src/Presentation/<Feature>/Models/` |
| `<Page>.razor` | `src/Presentation/<Feature>/Pages/` |
| `<Page>.razor.cs` | `src/Presentation/<Feature>/Pages/` |
| Refit method on `I<Feature>Client` | `src/Presentation/Shared/ServiceClients/Bff/Clients/` |

For dialog pages, add:

| File | Location |
|------|----------|
| `IDialog<Action>ViewModel.cs` | `src/Presentation/<Feature>/ViewModels/` |
| `Dialog<Action>ViewModel.cs` | `src/Presentation/<Feature>/ViewModels/` |
| `Dialog<Action>.razor` | `src/Presentation/<Feature>/Pages/` |
| `Dialog<Action>.razor.cs` | `src/Presentation/<Feature>/Pages/` |
| `<Action>Model.cs` (with `[Required]`) | `src/Presentation/<Feature>/Models/` |

DI: ViewModels auto-registered by `PresentationModule` (suffix `ViewModel` → Transient). ServiceClients auto-registered (suffix `ServiceClient` → Transient).

---

## Model

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.Models;

// Read model (immutable)
public record <Entity>Model(string Name, string Email);

// Write/form model (mutable, with validation attributes)
using System.ComponentModel.DataAnnotations;

public class Add<Entity>Model
{
    [Required]
    public string Name { get; set; }

    [Required]
    public string Email { get; set; }
}
```

---

## ViewModel Interface

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.ViewModels;

using {{NamespaceRoot}}.Presentation.<Feature>.Models;
using {{NamespaceRoot}}.Presentation.Shared;

public interface I<Feature>ViewModel : IViewModel
{
    bool IsBusy { get; set; }
    IList<<Entity>Model> Items { get; set; }
}
```

---

## ViewModel Implementation

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.ViewModels;

using {{NamespaceRoot}}.Presentation.<Feature>.Models;
using {{NamespaceRoot}}.Presentation.<Feature>.ServiceClients;
using {{NamespaceRoot}}.Presentation.Shared;

public class <Feature>ViewModel(I<Feature>ServiceClient serviceClient) : I<Feature>ViewModel
{
    private readonly I<Feature>ServiceClient _serviceClient = serviceClient;

    public bool IsBusy { get; set; }
    public IList<<Entity>Model> Items { get; set; }

    public async Task InitializeAsync(IErrorComponent errorComponent)
    {
        IsBusy = true;
        try
        {
            var items = await _serviceClient.GetAllAsync();
            Items = [.. items];
        }
        catch (Exception ex)
        {
            errorComponent.ProcessError(ex);
        }
        finally
        {
            IsBusy = false;
        }
    }
}
```

**Lifecycle rules**:
- Set `IsBusy = true` before async work, `false` in `finally`
- Catch exceptions → `errorComponent.ProcessError(ex)`
- Never throw from `InitializeAsync`

---

## Dialog ViewModel (for add/edit forms)

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.ViewModels;

using System.ComponentModel.DataAnnotations;
using {{NamespaceRoot}}.Presentation.<Feature>.ServiceClients;
using {{NamespaceRoot}}.Presentation.Resources.<Feature>;
using {{NamespaceRoot}}.Presentation.Shared;

public class Dialog<Action><Entity>ViewModel(
    I<Feature>ServiceClient serviceClient,
    IDistributedTraceService distributedTraceService) : IDialog<Action><Entity>ViewModel
{
    private readonly I<Feature>ServiceClient _serviceClient = serviceClient
        ?? throw new ArgumentNullException(nameof(serviceClient));
    private readonly IDistributedTraceService _distributedTraceService = distributedTraceService
        ?? throw new ArgumentNullException(nameof(distributedTraceService));
    private IErrorComponent _errorComponent;

    [Required(ErrorMessageResourceName = nameof(<Feature>.<FieldName>CannotBeEmpty),
              ErrorMessageResourceType = typeof(<Feature>))]
    public string Name { get; set; }

    // ... more validated properties ...

    public bool IsBusy { get; set; }

    public async Task HandleValidSubmitAsync()
    {
        try
        {
            IsBusy = true;
            await _distributedTraceService.UseDistributedTrace(async (dt) =>
            {
                await _serviceClient.AddAsync(Name, ...);
            });
        }
        catch (Exception ex)
        {
            _errorComponent.ProcessError(ex);
        }
        finally
        {
            IsBusy = false;
        }
    }

    public async Task InitializeAsync(IErrorComponent errorComponent)
    {
        _errorComponent = errorComponent ?? throw new ArgumentNullException(nameof(errorComponent));
        await Task.CompletedTask;
    }
}
```

---

## ServiceClient

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.ServiceClients;

using {{NamespaceRoot}}.Presentation.<Feature>.Models;
using {{NamespaceRoot}}.Presentation.Shared.ServiceClients.Bff;
using {{NamespaceRoot}}.Presentation.Shared.ServiceClients.Bff.Clients;

public class <Feature>ServiceClient(I<Feature>Client client) : I<Feature>ServiceClient
{
    private readonly I<Feature>Client _client = client
        ?? throw new ArgumentNullException(nameof(client));

    public async Task<IEnumerable<<Entity>Model>> GetAllAsync()
    {
        var items = await _client.Get<Feature>Async(null, ApiConstants.ApiVersion);
        return items.Select(x => new <Entity>Model(x.Name, x.Email));
    }

    public async Task<Result<Unit>> AddAsync(string name, string email)
    {
        try
        {
            await _client.Add<Entity>Async(
                new Add<Entity>Dto(name, email), ApiConstants.ApiVersion);
        }
        catch (ApiException ex)
        {
            return ex.ConvertApiExceptionToResult<Unit>();
        }
        return new Result<Unit>(Unit.Default());
    }
}
```

**Key patterns**:
- Always pass `ApiConstants.ApiVersion` to Refit calls
- Catch `ApiException` → `ConvertApiExceptionToResult<T>()` for write operations
- Map API DTOs to Presentation Models in the ServiceClient

---

## Refit Client

Add methods to `src/Presentation/Shared/ServiceClients/Bff/Clients/I<Feature>Client.cs`:

```csharp
namespace {{NamespaceRoot}}.Presentation.Shared.ServiceClients.Bff.Clients;

using Refit;

public interface I<Feature>Client
{
    [Get("/api/<feature-kebab>")]
    Task<ICollection<<Entity>Dto>> Get<Feature>Async(
        [Query] string name = null,
        [AliasAs("api-version")][Query] string apiVersion = null,
        CancellationToken cancellationToken = default);

    [Post("/api/<feature-kebab>")]
    Task Add<Entity>Async(
        [Body] Add<Entity>Dto dto,
        [AliasAs("api-version")][Query] string apiVersion = null,
        CancellationToken cancellationToken = default);
}
```

**Rules**: Always include `[AliasAs("api-version")][Query] string apiVersion = null` and `CancellationToken`.

---

## Razor Page (list view)

### `<Feature>.razor`

```html
@page "/<feature-kebab>"
@using {{NamespaceRoot}}.Presentation.<Feature>.Models

<PageTitle>@Localizer["PageTitle"]</PageTitle>

@if (ViewModel.IsBusy)
{
    <MudPaper Class="d-flex pa-4 justify-center align-center">
        <MudProgressCircular Indeterminate />
    </MudPaper>
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
            <PropertyColumn Property="x => x.Email" Title='@Localizer["Email"]' />
        </Columns>
        <PagerContent>
            <MudDataGridPager T="<Entity>Model" />
        </PagerContent>
    </MudDataGrid>

    <MudButton @onclick="OpenAddDialogAsync" Variant="Variant.Filled" Color="Color.Primary">
        @Localizer["Add"]
    </MudButton>
}
```

### `<Feature>.razor.cs` (code-behind)

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.Pages;

using Microsoft.AspNetCore.Components;
using Microsoft.Extensions.Localization;
using MudBlazor;
using {{NamespaceRoot}}.Presentation.<Feature>.ViewModels;
using {{NamespaceRoot}}.Presentation.Shared;

public partial class <Feature>
{
    [CascadingParameter]
    public Error Error { get; set; }

    [Inject]
    public I<Feature>ViewModel ViewModel { get; set; }

    [Inject]
    private IDialogService DialogService { get; set; }

    [Inject]
    private IStringLocalizer<<Feature>> Localizer { get; set; }

    protected async override Task OnInitializedAsync() => await ViewModel.InitializeAsync(Error);

    private async Task OpenAddDialogAsync()
    {
        var options = new DialogOptions { CloseOnEscapeKey = true, BackdropClick = false };
        var dialog = await DialogService.ShowAsync<Dialog<Action>>("Add item", options);
        var result = await dialog.Result;

        if (!result.Canceled)
        {
            await ViewModel.InitializeAsync(Error);
            StateHasChanged();
        }
    }
}
```

**Page patterns**:
- `[CascadingParameter] public Error Error { get; set; }` — cascading error component
- `[Inject]` for DI — ViewModel, DialogService, Localizer
- `OnInitializedAsync` → `ViewModel.InitializeAsync(Error)`
- After dialog closes → re-initialize + `StateHasChanged()`

---

## Dialog Razor Page

### `Dialog<Action>.razor`

```html
<EditForm Model="@ViewModel" OnValidSubmit="ValidSubmit">
    <MudDialog>
        <TitleContent>@Localizer["DialogTitle"]</TitleContent>
        <DialogContent>
            @if (ViewModel.IsBusy)
            {
                <MudSkeleton />
            }
            else
            {
                <DataAnnotationsValidator />
                <ValidationSummary />
                <div class="form-group">
                    <MudTextField Label="@Localizer["Name"]" T="string"
                                  @bind-Value="ViewModel.Name"
                                  For="@(() => ViewModel.Name)" />
                </div>
            }
        </DialogContent>
        <DialogActions>
            <MudButton Variant="Variant.Filled" Disabled="ViewModel.IsBusy"
                       Color="Color.Primary" ButtonType="ButtonType.Submit">
                @Localizer["Save"]
            </MudButton>
            <MudButton Variant="Variant.Outlined" Disabled="ViewModel.IsBusy"
                       OnClick="Cancel">
                @Localizer["Cancel"]
            </MudButton>
        </DialogActions>
    </MudDialog>
</EditForm>
```

### `Dialog<Action>.razor.cs`

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.Pages;

using Microsoft.AspNetCore.Components;
using Microsoft.AspNetCore.Components.Forms;
using Microsoft.Extensions.Localization;
using MudBlazor;
using {{NamespaceRoot}}.Presentation.<Feature>.ViewModels;

public partial class Dialog<Action>
{
    [Inject]
    private IStringLocalizer<<Feature>> Localizer { get; set; }

    [Inject]
    private IDialog<Action><Entity>ViewModel ViewModel { get; set; }

    [CascadingParameter]
    private IMudDialogInstance MudDialog { get; set; }

    private async Task ValidSubmit(EditContext editContext)
    {
        await ViewModel.HandleValidSubmitAsync();
        MudDialog.Close(DialogResult.Ok(ViewModel));
    }

    private void Cancel() => MudDialog.Close();
}
```

---

## Routing

Add the page route to `src/Presentation/Shared/Navigation/` if using the navigation service OR add a `NavLink` in the appropriate layout.

---

## Localization

Create resource files in `src/Presentation/Resources/<Feature>/`:
- `<Feature>.resx` (default/English)
- `<Feature>.nl.resx` (Dutch)
- `<Feature>.fr.resx` (French)

Reference in code: `@Localizer[nameof(Resources.<Feature>.<Feature>.KeyName)]`

---

## Checklist

- [ ] ViewModel implements `IViewModel` with `InitializeAsync(IErrorComponent)`
- [ ] `IsBusy` guard on all async operations
- [ ] `try/catch → errorComponent.ProcessError(ex)` in ViewModel
- [ ] Service Client catches `ApiException → ConvertApiExceptionToResult<T>()`
- [ ] Refit Client includes `apiVersion` + `CancellationToken`
- [ ] `[CascadingParameter] Error` on all pages
- [ ] `[Inject]` for ViewModel, DialogService, Localizer
- [ ] MudBlazor components (not raw HTML)
- [ ] Localization via `IStringLocalizer`
- [ ] Dialog uses `IMudDialogInstance` + `MudDialog.Close(DialogResult.Ok(...))`
- [ ] Copyright header on all new `.cs` files


