---
description: "Use when creating, modifying, or reviewing Refit HTTP service clients, feature services, or the CookieHandler/XSRF pipeline in Blazor WASM projects. Covers the WASM → Cfe BFF client pattern, DTO mapping, and registration conventions. This is the Blazor WASM template — Blazor Server projects use a different pattern (typed Refit client registration / BffClientConfigurator / UserContextHeaderHandler) documented in the add-blazor-page and add-endpoint skills."
applyTo: "src/Presentation/**/ServiceClients/**,src/Presentation/**/Services/**,src/Infrastructure/Http/**"
---
# Refit + BFF Client Pattern — {{SolutionName}}Wasm

> **Template scope: Blazor WASM.** This instruction file covers the WASM client pattern (`CookieHandler` + `IAntiforgeryTokenStore` + `AddRefitClientWithCookies`). Blazor Server projects use a different registration pattern — see the `/add-blazor-page` and `/add-endpoint` skills for that variant.

## Architecture

```
Razor Page → ViewModel → I<Feature>ServiceClient → I<Feature>Client (Refit) → CookieHandler → Client API
```

- **Refit interfaces** (`I<Feature>Client`) call the Client API controllers.
- **CookieHandler** attaches the session cookie (`credentials: include`) and `X-XSRF-TOKEN` header on mutating requests.
- **Feature ServiceClients** (`I<Feature>ServiceClient`) wrap the Refit client, map `DTO → Model`, and return `Result<T>` for error handling.

## File Locations

| Kind | Location | Naming |
|------|----------|--------|
| Refit interface | `Presentation/Shared/ServiceClients/Bff/Clients/I<Feature>Client.cs` | `I<Feature>Client` |
| ServiceClient interface | `Presentation/<Feature>/ServiceClients/I<Feature>ServiceClient.cs` | `I<Feature>ServiceClient` |
| ServiceClient impl | `Presentation/<Feature>/ServiceClients/<Feature>ServiceClient.cs` | `<Feature>ServiceClient` |
| Model | `Presentation/<Feature>/Models/<Name>Model.cs` | `<Name>Model` |
| DTO (shared) | `Contracts/<Feature>/<Name>Dto.cs` | `<Name>Dto` (record) |

## 1. Refit Client Interface

Location: `Presentation/Shared/ServiceClients/Bff/Clients/I<Feature>Client.cs`  
Reference: `IDoctorClient.cs`

```csharp
namespace {{NamespaceRoot}}.Presentation.Shared.ServiceClients.Bff.Clients;

using {{NamespaceRoot}}.Contracts.<Feature>;
using Refit;

public interface I<Feature>Client
{
    [Get("/api/<feature>")]
    Task<List<<Feature>Dto>> GetAllAsync(CancellationToken cancellationToken = default);

    [Get("/api/<feature>/{id}")]
    Task<<Feature>Dto> GetByIdAsync(int id, CancellationToken cancellationToken = default);

    [Post("/api/<feature>")]
    Task<<Feature>Dto> CreateAsync([Body] Add<Feature>Dto dto, CancellationToken cancellationToken = default);

    [Delete("/api/<feature>/{id}")]
    Task DeleteAsync(int id, CancellationToken cancellationToken = default);
}
```

### Rules

- Every method **must** accept `CancellationToken cancellationToken = default` as the last parameter.
- Route must match the Cfe controller route (`/api/<controller>`).
- Use `[Body]` for request payloads, `[Query]` for query string params.
- Use `[AliasAs("api-version")][Query] string? apiVersion = null` only when the controller requires API versioning.
- Use `[Multipart]` for file uploads (see `IDossierServiceClient`).
- Return the DTO type directly — Refit deserializes automatically.

## 2. Registration — `BffServiceClients.cs`

After creating a new Refit interface, register it in `BffServiceClients.AddBffServiceClients()`:

```csharp
services.AddRefitClientWithCookies<I<Feature>Client>(baseAddress);
```

This is the **only** registration needed. The private `AddRefitClientWithCookies<T>` method:
1. Splits `baseAddress` into origin + path base.
2. Configures `HttpClient.BaseAddress` to the origin.
3. Adds `PathBaseDelegatingHandler` (if deployed under a sub-path).
4. Adds `CookieHandler` (always — handles cookies + XSRF).

## 3. CookieHandler — Security Pipeline

`Infrastructure/Http/CookieHandler.cs` — a `DelegatingHandler` that:
1. Sets `BrowserRequestCredentials.Include` on every request → browser sends `HttpOnly` session cookie.
2. On **mutating methods** (POST, PUT, DELETE, PATCH): reads `IAntiforgeryTokenStore.Token` and adds the `X-XSRF-TOKEN` header.

The token is populated by `BffAuthenticationStateProvider` from the Cfe's user endpoint response.

**Never bypass or replace this handler.** All Refit clients go through it automatically via `AddRefitClientWithCookies`.

## 4. Feature ServiceClient (DTO → Model Mapping)

Location: `Presentation/<Feature>/ServiceClients/<Feature>ServiceClient.cs`  
Reference: `DoctorsServiceClient.cs`

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.ServiceClients;

using {{NamespaceRoot}}.Core.Functional;
using {{NamespaceRoot}}.Contracts.<Feature>;
using {{NamespaceRoot}}.Presentation.<Feature>.Models;
using {{NamespaceRoot}}.Presentation.Shared.ServiceClients.Bff.Clients;

public class <Feature>ServiceClient(I<Feature>Client client) : I<Feature>ServiceClient
{
    public async Task<List<<Feature>Model>> GetAll()
    {
        List<<Feature>Dto> dtos = await client.GetAllAsync();
        return dtos.Select(x => new <Feature>Model { Id = x.Id, Name = x.Name }).ToList();
    }

    public async Task<Result<<Feature>Model>> CreateAsync(<Feature>Model model)
    {
        var dto = new Add<Feature>Dto { Name = model.Name };
        <Feature>Dto created = await client.CreateAsync(dto);
        return new Result<<Feature>Model>(new <Feature>Model { Id = created.Id, Name = created.Name });
    }
}
```

### Rules

- Constructor-inject the Refit interface (`I<Feature>Client`). Use primary constructor syntax.
- Map DTO → Model in the ServiceClient. ViewModels never see DTOs.
- Return `Result<T>` for operations that can fail with business errors.
- Return plain collections for read operations.
- **No registration needed** — Scrutor auto-registers classes ending in `ServiceClient` as transient via `PresentationModule`.

## 5. Model

Models are plain C# classes or records in `Presentation/<Feature>/Models/`. They are the UI-facing representation — no framework dependencies.

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.Models;

public class <Feature>Model
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
}
```

## 6. Error Handling

Use `ApiExceptionExtensions` from `Presentation/Shared/ServiceClients/Bff/`:

- `exception.GetUserFriendlyMessage()` — maps HTTP status codes to user messages.
- `exception.RequiresReauthentication()` — returns `true` for 401.
- `exception.TryGetErrorMessage(out string msg)` — handles both `ApiException` and `HttpRequestException`.

In ViewModels, errors are caught in `try/catch` and routed to `_errorComponent.ProcessError(ex)` or `_snackbar.Add(...)`.

## Anti-Patterns

| Don't | Do |
|-------|-----|
| Register Refit clients manually in `Program.cs` | Add to `BffServiceClients.AddBffServiceClients()` |
| Skip `CancellationToken` on Refit methods | Always include as last param with `= default` |
| Return DTOs from feature services | Map DTO → Model in the service |
| Inject Refit clients into ViewModels | Inject the `I<Feature>ServiceClient` wrapper |
| Create a new `HttpClient` or `DelegatingHandler` | Use the existing `CookieHandler` pipeline |
| Store or forward tokens manually | `CookieHandler` + `IAntiforgeryTokenStore` handles this |
