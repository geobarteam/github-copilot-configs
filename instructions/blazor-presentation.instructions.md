---
description: "Blazor presentation layer conventions for ViewModels, pages, components, and localization. Covers IsBusy guard pattern, IErrorComponent, IStringLocalizer, MudBlazor dialogs, and ViewModel lifecycle. Activates when editing Presentation project files (excluding ServiceClients which have their own instructions)."
applyTo: "src/Presentation/**/Pages/**,src/Presentation/**/ViewModels/**,src/Presentation/**/Models/**,src/Presentation/**/Components/**,src/Presentation/**/Dialogs/**"
---
# Blazor Presentation Conventions

## ViewModel Pattern

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.ViewModels;

public interface I<Feature>ViewModel
{
    bool IsBusy { get; }
    List<<Entity>Model> Items { get; }
    Task InitializeAsync(IErrorComponent errorComponent);
}

public class <Feature>ViewModel(
    I<Feature>ServiceClient serviceClient,
    IStringLocalizer<Translations> localizer) : I<Feature>ViewModel
{
    private IErrorComponent _errorComponent = null!;

    public bool IsBusy { get; private set; }
    public List<<Entity>Model> Items { get; private set; } = [];

    public async Task InitializeAsync(IErrorComponent errorComponent)
    {
        _errorComponent = errorComponent;
        await LoadDataAsync();
    }

    private async Task LoadDataAsync()
    {
        if (IsBusy) return;
        IsBusy = true;
        try
        {
            var dtos = await serviceClient.GetAllAsync(CancellationToken.None);
            Items = dtos.Select(d => new <Entity>Model(d)).ToList();
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

## Razor Page

```razor
@page "/<feature>"
@inject I<Feature>ViewModel ViewModel

<ErrorComponent @ref="_errorComponent" />

@if (ViewModel.IsBusy)
{
    <MudProgressCircular Indeterminate="true" />
}
else
{
    @* Render ViewModel.Items *@
}

@code {
    private ErrorComponent _errorComponent = null!;

    protected override async Task OnInitializedAsync()
    {
        await ViewModel.InitializeAsync(_errorComponent);
    }
}
```

## MudBlazor Dialog Pattern

```csharp
public class <Feature>ViewModel
{
    [Inject] private IDialogService DialogService { get; set; } = null!;

    private async Task OpenDialogAsync()
    {
        var parameters = new DialogParameters<MyDialog>
        {
            { x => x.EntityId, selectedId }
        };
        var options = new DialogOptions { MaxWidth = MaxWidth.Medium, FullWidth = true };
        var dialog = await DialogService.ShowAsync<MyDialog>(
            localizer["DialogTitle"], parameters, options);
        var result = await dialog.Result;
        if (!result.Canceled)
            await LoadDataAsync();
    }
}
```

## Localization

- Use `IStringLocalizer<Translations>` — never hardcode UI strings.
- Resource file: `Resources/Translations.resx` (default) + `Translations.<lang>.resx`.

## DI Registration

ViewModels and ServiceClients are registered via `PresentationModule` with Scrutor suffix scanning:
- `*ViewModel` → Transient
- `*ServiceClient` → Transient

Every ViewModel needs an `I<Name>ViewModel` interface.

## File Locations

| Type | Path |
|------|------|
| Page | `src/Presentation/<Feature>/Pages/<Feature>Page.razor` |
| ViewModel | `src/Presentation/<Feature>/ViewModels/<Feature>ViewModel.cs` |
| Model | `src/Presentation/<Feature>/Models/<Entity>Model.cs` |
| Dialog | `src/Presentation/<Feature>/Dialogs/<Name>Dialog.razor` |
| Service | `src/Presentation/<Feature>/Services/<Feature>Service.cs` |
