---
description: "Layer guidance for Blazor Presentation pages, ViewModels, ServiceClients, dialogs, and MudBlazor components. Covers ViewModel lifecycle (IsBusy, InitializeAsync), error handling, localization, and DI registration. Activates when editing Presentation layer files."
applyTo: "src/Presentation/**"
---
# Blazor Presentation Conventions

## File Inventory (per feature page)

| File | Location |
|------|----------|
| `I<Feature>ViewModel.cs` | `Presentation/<Feature>/ViewModels/` |
| `<Feature>ViewModel.cs` | `Presentation/<Feature>/ViewModels/` |
| `I<Feature>ServiceClient.cs` | `Presentation/<Feature>/ServiceClients/` |
| `<Feature>ServiceClient.cs` | `Presentation/<Feature>/ServiceClients/` |
| `<Entity>Model.cs` | `Presentation/<Feature>/Models/` |
| `<Page>.razor` | `Presentation/<Feature>/Pages/` |
| `<Page>.razor.cs` | `Presentation/<Feature>/Pages/` |

## ViewModel Pattern

```csharp
public class <Feature>ViewModel(I<Feature>ServiceClient serviceClient) : I<Feature>ViewModel
{
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
- Set `IsBusy = true` before async work, `false` in `finally`.
- Catch exceptions → `errorComponent.ProcessError(ex)`.
- Never throw from `InitializeAsync`.

## Dialog ViewModel (add/edit forms)

```csharp
public class Dialog<Action><Entity>ViewModel(
    I<Feature>ServiceClient serviceClient,
    IDistributedTraceService distributedTraceService) : IDialog<Action><Entity>ViewModel
{
    [Required(ErrorMessageResourceName = nameof(<Feature>.<FieldName>CannotBeEmpty),
              ErrorMessageResourceType = typeof(<Feature>))]
    public string Name { get; set; }

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
}
```

## Model

```csharp
// Read model (immutable)
public record <Entity>Model(string Name, string Email);

// Write/form model (mutable, with DataAnnotations)
public class Add<Entity>Model
{
    [Required] public string Name { get; set; }
    [Required] public string Email { get; set; }
}
```

## ServiceClient (DTO → Model mapping)

```csharp
public class <Feature>ServiceClient(I<Feature>Client client) : I<Feature>ServiceClient
{
    public async Task<IEnumerable<<Entity>Model>> GetAllAsync()
    {
        var items = await _client.Get<Feature>Async(null, ApiConstants.ApiVersion);
        return items.Select(x => new <Entity>Model(x.Name, x.Email));
    }
}
```

## Razor Page (code-behind)

```csharp
public partial class <Feature>Page : ComponentBase
{
    [Inject] public I<Feature>ViewModel ViewModel { get; set; }

    protected override async Task OnInitializedAsync()
    {
        await ViewModel.InitializeAsync(ErrorComponent);
    }
}
```

## Rules

- **DI**: ViewModels auto-registered by `PresentationModule` (suffix `ViewModel` → Transient). ServiceClients auto-registered (suffix `ServiceClient` → Transient).
- **MudBlazor** for all UI components — no raw HTML inputs.
- **IStringLocalizer** for all user-facing text — no hardcoded strings.
- **IsBusy guard** — every button/action must check `IsBusy` to prevent double-submit.
- ViewModel interfaces implement `IViewModel`.
- ServiceClients map DTOs → Models — ViewModels never see DTOs.
- One ViewModel per page/dialog.

## Forbidden

- Injecting Refit clients (`I<Feature>Client`) directly into ViewModels — use `I<Feature>ServiceClient` wrapper.
- Returning DTOs from ServiceClients to ViewModels (map to Models).
- Hardcoded UI strings (use `IStringLocalizer`).
- Business logic in ViewModels (delegate to ServiceClients → BFF API → Application layer).
- Direct HTTP calls (`HttpClient`) — use Refit clients via ServiceClients.
