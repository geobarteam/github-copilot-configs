---
name: add-endpoint
description: "Use when adding a new API endpoint, controller action, query, or command to an existing feature in {{SolutionName}}. Covers the vertical slice from Application layer through BFF controller to Refit client, without creating a new Domain entity or DB table. Use for: adding GET/POST/PUT/DELETE endpoints, adding query-only or command-only slices, extending existing controllers with new actions."
argument-hint: "Describe the endpoint, e.g. 'GET endpoint to retrieve product by ID'"
---

# Add Endpoint — Vertical Slice (No New Entity)

Add a new API endpoint to an existing feature. This skill covers Application → Contracts → Controller → Refit client → ServiceClient, assuming the Domain entity and DB table already exist.

## Prerequisites

- The target **entity/feature already exists** in `Core/Domain/` and `Core/Persistence/`.
- If you need a new entity + DB table, use the **`/build-feature`** skill instead.

## Workflow

```
1. Identify if this is a QUERY (read) or COMMAND (write) endpoint
2. RED — write the unit test (handler) or integration test (controller) FIRST
3. RUN — confirm FAIL
4. GREEN — implement the slice
5. RUN — confirm PASS
6. CODE ANALYSIS — dotnet build 2>&1 | Select-String violations → fix all → dotnet format → repeat until clean
7. PROVE — build succeeded · all tests pass · dotnet format --verify-no-changes exit 0
8. 🛑 STOP — wait for user approval
```

---

## Query Endpoint (GET)

### 1. Query Interface + Implementation

Location: `src/Core/Application/Functionalities/<Feature>/Queries/<Action>/`

- Interface: `IGet<Entity>By<Criteria>Query` with `Task<<Entity>> Execute(<paramType> <param>)`.
- Implementation: delegates to `I<Entity>Repository`. Use `AsNoTracking` in repository.
- DI: auto-registered by `ApplicationModule` (suffix `Query` → Scoped).

> Full pattern: see `application-layer.instructions.md`.

### 2. Repository Method (if needed)

Add method to `I<Entity>Repository` interface and `<Entity>Repository` implementation. Always `AsNoTracking()` for reads.

> Full pattern: see `persistence-layer.instructions.md`.

### 3. DTO (if needed)

Location: `src/Contracts/<Feature>/Api/`

```csharp
namespace {{NamespaceRoot}}.Contracts.<Feature>.Api;
public record <Entity>DetailDto(int Id, string Name, ...);
```

### 4. Controller Action

Add to existing controller in `src/Host/Client/Controllers/`. Inject the new query in the constructor. Pattern: execute query → null check (`NotFound()`) → audit log → map to DTO → `Ok(dto)`. Always include `CancellationToken` and `try/catch` with structured logging.

> Full pattern (GET action, audit logging): see `bff-controller.instructions.md`.

### 5. Refit Client Method

Add to `src/Presentation/Shared/ServiceClients/Bff/Clients/I<Feature>Client.cs`:

```csharp
[Get("/api/<feature-kebab>/{id}")]
Task<<Entity>DetailDto> Get<Entity>ByIdAsync(
    [AliasAs("id")] int id,
    [AliasAs("api-version")][Query] string apiVersion = null,
    CancellationToken cancellationToken = default);
```

### 6. Register the Refit Client (only for new client interfaces)

If you created a **new** `I<Feature>ServiceClient.cs` interface (not just added a method to an existing one), register it in `BffServiceClients`.

Add one line in `src/Presentation/Shared/ServiceClients/Bff/BffServiceClients.cs` → `AddBffServiceClients()`:

```csharp
services.AddRefitClientWithCookies<I<Feature>ServiceClient>(baseAddress);
```

The `AddRefitClientWithCookies<T>` method handles:
- `CookieHandler` for session cookie (`credentials: include`) + XSRF
- `PathBaseDelegatingHandler` for sub-path deployment
- Base address resolution
- `SystemTextJsonContentSerializer` with camelCase policy

> See `refit-client.instructions.md` for full registration details and CookieHandler pipeline.

### 7. ServiceClient Method

Add to `src/Presentation/<Feature>/Services/<Feature>Service.cs`:

```csharp
public async Task<<Entity>Model> GetByIdAsync(int id)
{
    var dto = await _client.Get<Entity>ByIdAsync(id, ApiConstants.ApiVersion);
    return new <Entity>Model(dto.Name, dto.Email, ...);
}
```

---

## Command Endpoint (POST/PUT/DELETE)

### 1. Command + Handler

Location: `src/Core/Application/Functionalities/<Feature>/Commands/<Action>/`

- **Command**: `record <Action><Entity>Command(...)` with `Validate()` returning `(bool IsValid, List<string> Errors)`.
- **Handler**: `<Action><Entity>CommandHandler` implementing `ICommandHandler<<Action><Entity>Command, Result<Unit>>`. Validates first, then persists.
- DI: auto-registered by `ApplicationModule` (suffix `Handler` → Scoped).

> Full pattern: see `application-layer.instructions.md`.

### 2. Controller Action

Add to existing controller in `src/Host/Client/Controllers/`. Use `_auditLogger.LogActionAsync<Unit>()` wrapping the command execution. Map `Result<T>` to HTTP: `!IsSuccess` → `BadRequest(error)`, success → `Ok()`.

> Full pattern (POST/PUT/DELETE, audit logging): see `bff-controller.instructions.md`.


---

## Checklist

- [ ] `AsNoTracking()` on all read queries
- [ ] `CancellationToken` propagated in controller actions
- [ ] Audit logging with `IAuditLogger` + `GdprAuditLog`
- [ ] `Result<T>` for error handling — never throw for business errors
- [ ] Controller injects query/command via interface
- [ ] Refit client has `apiVersion` + `CancellationToken` parameters
- [ ] ServiceClient catches `ApiException → ConvertApiExceptionToResult<T>()`
- [ ] Unit test for handler, integration test for controller
