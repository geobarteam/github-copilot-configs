---
description: "Layer guidance for API controllers. Covers controller action patterns, Result<T> to HTTP mapping, audit logging (IAuditLogger/GdprAuditLog), correlation IDs, user context. Activates when editing API controller files."
applyTo: "src/Host/Client/**"
---
# API Controller Conventions

## Controller Structure

Location: `src/Host/Api/Controllers/<Feature>Controller.cs`

Controllers are thin — they delegate to Application queries/commands and map results to HTTP.

### Standard Dependencies

```csharp
[ApiController]
[Route("api/[controller]")]
public class <Feature>Controller(
    IGet<Entity>Query getQuery,
    I<Action><Entity>CommandHandler commandHandler,
    IAuditLogger auditLogger,
    ICorrelationContextAccessor correlationContextAccessor,
    IUserContextAccessor userContextAccessor,
    ILogger<<Feature>Controller> logger) : ControllerBase
{
    // ...
}
```

## GET Action (Query)

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<<Entity>DetailDto>> GetById(
    [FromRoute] int id, CancellationToken cancellationToken)
{
    try
    {
        var entity = await _getByIdQuery.Execute(id);
        if (entity is null) return NotFound();

        var correlationId = _correlationContextAccessor?.Current?.CorrelationId
            ?? Guid.NewGuid().ToString();
        await _auditLogger.LogMessageAsync(
            new GdprAuditLog(
                entity.Name,
                _userContextAccessor?.Current?.UserId,
                _userContextAccessor?.Current?.UserName,
                "Get <entity> by id",
                AuditAction.Read,
                AuditActionStatus.Succeeded,
                correlationId),
            cancellationToken);

        return Ok(new <Entity>DetailDto(entity.Id, entity.Name));
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "{Controller} has thrown an exception.", nameof(<Feature>Controller));
        throw;
    }
}
```

## POST/PUT/DELETE Action (Command)

```csharp
[HttpPost]
public async Task<ActionResult> Create(
    [FromBody] <Action><Entity>Dto dto, CancellationToken cancellationToken)
{
    try
    {
        var result = await _auditLogger.LogActionAsync<Unit>(
            new GdprAuditLog(
                dto.Name,
                _userContextAccessor?.Current?.UserId,
                _userContextAccessor?.Current?.UserName,
                "<Action> <entity>",
                AuditAction.Create,
                AuditActionStatus.Initiated,
                _correlationContextAccessor?.Current?.CorrelationId
                    ?? Guid.NewGuid().ToString()),
            async () => await _commandHandler.Execute(
                new <Action><Entity>Command(dto.Name, dto.Email)),
            cancellationToken);

        if (!result.IsSuccess)
            return BadRequest(result.Error);
        return Ok();
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "{Controller} has thrown an exception.", nameof(<Feature>Controller));
        throw;
    }
}
```

## Result\<T\> → HTTP Mapping

| `Result<T>` state | HTTP response |
|--------------------|---------------|
| `IsSuccess` with value | `Ok(value)` |
| `IsSuccess` without value | `Ok()` |
| `!IsSuccess` | `BadRequest(result.Error)` |
| Entity not found | `NotFound()` |

## Rules

- Controllers are **thin** — no business logic, no direct DB access.
- Every action includes **audit logging** via `IAuditLogger` + `GdprAuditLog`.
- Correlation ID from `ICorrelationContextAccessor` (fallback to `Guid.NewGuid()`).
- User context from `IUserContextAccessor` for GDPR audit trail.
- **CancellationToken** on every action as the last parameter.
- Log exceptions with structured logging (`{Controller}` placeholder).
- DTOs live in `Contracts/<Feature>/Api/` — not in the Host project.

## Forbidden

- Business logic in controllers (move to Application handlers).
- Direct `DbContext` usage (use Application queries/commands).
- Creating `HttpClient` instances (use Refit clients in Presentation).
- Returning domain entities directly (map to DTOs).
