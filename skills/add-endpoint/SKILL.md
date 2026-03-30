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

```csharp
// Interface
namespace {{NamespaceRoot}}.Core.Application.Functionalities.<Feature>.Queries.<Action>;

public interface IGet<Entity>By<Criteria>Query
{
    Task<<Entity>> Execute(<paramType> <param>);
}

// Implementation
public class Get<Entity>By<Criteria>Query(I<Entity>Repository repository) : IGet<Entity>By<Criteria>Query
{
    private readonly I<Entity>Repository _repository = repository
        ?? throw new ArgumentNullException(nameof(repository));

    public async Task<<Entity>> Execute(<paramType> <param>)
        => await _repository.GetBy<Criteria>(<param>);  // AsNoTracking in repository
}
```

DI: auto-registered by `ApplicationModule` (suffix `Query` → Scoped).

### 2. Repository Method (if needed)

Add method to `I<Entity>Repository` interface and `<Entity>Repository` implementation:

```csharp
// In I<Entity>Repository
Task<<Entity>?> GetBy<Criteria>(<paramType> <param>);

// In <Entity>Repository
public async Task<<Entity>?> GetBy<Criteria>(<paramType> <param>)
    => await dbContext.<Entities>.AsNoTracking().FirstOrDefaultAsync(e => e.<Prop> == <param>);
```

### 3. DTO (if needed)

Location: `src/Contracts/<Feature>/Api/`

```csharp
namespace {{NamespaceRoot}}.Contracts.<Feature>.Api;
public record <Entity>DetailDto(int Id, string Name, ...);
```

### 4. Controller Action

Add to existing controller in `src/Host/BFF/Controllers/`:

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<<Entity>DetailDto>> GetById([FromRoute] int id, CancellationToken cancellationToken)
{
    try
    {
        var entity = await _getByIdQuery.Execute(id);
        if (entity is null) return NotFound();

        // Audit logging
        var correlationId = _correlationContextAccessor?.Current?.CorrelationId ?? Guid.NewGuid().ToString();
        await _auditLogger.LogMessageAsync(
            new GdprAuditLog(
                entity.INS ?? entity.Name,
                _userContextAccessor?.Current?.UserId,
                _userContextAccessor?.Current?.UserName,
                "Get <entity> by id",
                AuditAction.Read,
                AuditActionStatus.Succeeded,
                correlationId),
            cancellationToken);

        return Ok(new <Entity>DetailDto(entity.Id, entity.Name, ...));
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "{Controller} has throw an exception.", nameof(<Feature>Controller));
        throw;
    }
}
```

Inject the new query in the controller's constructor.

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

If you created a **new** `I<Feature>Client.cs` interface (not just added a method to an existing one), register it in `BffClientConfigurator`.

**How Refit/HttpClient DI works in this project:**

```
appsettings.json
  └─ Nihdi.HttpClientServiceRegistry.{{SolutionName}}-bff  (BaseAddress, AddClientAccessToken, TimeoutInSeconds)
       └─ HttpClientServicesNames.{{SolutionName}}BFF = "{{SolutionName}}-bff"
            └─ ConfigurePresentationServices.Configure{{SolutionName}}PresentationServices()
                 └─ BffClientConfigurator.ConfigureBffServiceClient()
                      └─ services.AddNihdiRefitClient<I<Feature>Client>(httpServiceConfiguration, ConfigureDefaultHeaders)
                           └─ AddNihdiHttpServiceClient + RestService.For<T>(httpClient, DefaultRefitSettings)
                           └─ UserContextHeaderHandler (adds user context headers for audit/GDPR)
```

Add one line in `src/Presentation/Shared/ServiceClients/Bff/BffClientConfigurator.cs` → `ConfigureBffServiceClient()`:

```csharp
services.AddNihdiRefitClient<I<Feature>Client>(httpServiceConfiguration, ConfigureDefaultHeaders);
```

**Do NOT** create your own `HttpClient` or call `AddRefitClient` directly — the `AddNihdiRefitClient<T>` helper handles:
- Token management (`AddClientAccessToken` from config)
- Base address resolution from `HttpClientServiceRegistry`
- `SystemTextJsonContentSerializer` with camelCase policy
- `UserContextHeaderHandler` for audit/GDPR headers

### 7. ServiceClient Method

Add to `src/Presentation/<Feature>/ServiceClients/<Feature>ServiceClient.cs`:

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

```csharp
// Command
public record <Action><Entity>Command(string Name, string Email)
{
    public (bool IsValid, List<string> Errors) Validate()
    {
        var errors = new List<string>();
        if (string.IsNullOrEmpty(Name)) errors.Add("Name cannot be empty");
        return (!errors.Any(), errors);
    }
}

// Handler
public class <Action><Entity>CommandHandler(I<Entity>Repository repository)
    : ICommandHandler<<Action><Entity>Command, Result<Unit>>
{
    private readonly I<Entity>Repository _repository = repository
        ?? throw new ArgumentNullException(nameof(repository));

    public async Task<Result<Unit>> Execute(<Action><Entity>Command command)
    {
        Result<Unit> validateResult = ValidateCommand(command);
        if (!validateResult.IsSuccess)
            return validateResult;

        // Business logic here
        await _repository.SaveChangesAsync();
        return new Result<Unit>(Unit.Default());
    }

    private static Result<Unit> ValidateCommand(<Action><Entity>Command command)
    {
        (bool isValid, List<string> errors) = command.Validate();
        if (!isValid)
        {
            var sb = new StringBuilder();
            errors.ForEach(err => sb.Append($"{err} {Environment.NewLine}"));
            return new Result<Unit>(sb.ToString());
        }
        return new Result<Unit>(Unit.Default());
    }
}
```

### 2. Controller Action

```csharp
[HttpPost]  // or [HttpPut("{id}")] or [HttpDelete("{id}")]
public async Task<ActionResult> <Action>([FromBody] <Action><Entity>Dto dto, CancellationToken cancellationToken)
{
    try
    {
        var result = await _auditLogger.LogActionAsync<Unit>(
            new GdprAuditLog(
                dto.Name,
                _userContextAccessor?.Current?.UserId,
                _userContextAccessor?.Current?.UserName,
                "<Action> <entity>",
                AuditAction.Create,  // or Update, Delete
                AuditActionStatus.Initiated,
                _correlationContextAccessor?.Current?.CorrelationId ?? Guid.NewGuid().ToString()),
            async () => await _commandHandler.Execute(new <Action><Entity>Command(dto.Name, dto.Email)),
            cancellationToken);

        if (!result.IsSuccess)
            return BadRequest(result.Error);
        return Ok();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "{Controller} has throw an exception.", nameof(<Feature>Controller));
        throw;
    }
}
```

### 3. Transactional Session

If the command publishes NServiceBus events, add `[RequiresTransactionalSession]` to the action:

```csharp
[HttpPost]
[RequiresTransactionalSession]
public async Task<ActionResult> <Action>(...) { ... }
```

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
