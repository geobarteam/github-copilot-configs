---
description: "Layer guidance for API controllers. Covers controller action patterns, Result<T> to HTTP mapping, audit logging (IAuditLogger/GdprAuditLog), correlation IDs, user context. Activates when editing API controller files."
applyTo: "src/Host/Client/**"
---
# API Controller Conventions

## Controller Structure

Location: `src/Host/Client/Controllers/<Feature>Controller.cs`

Controllers are thin — they delegate to Application queries/commands and map results to HTTP.

### Standard Dependencies

```csharp
[ApiController]
[Route("api/[controller]")]
public class <Feature>Controller(
    IGet<Entity>Query get<Entity>Query,
    I<Entity><Action>Handler <entity><Action>Handler,
    IAuditLogger auditLogger,
    ILogger<<Feature>Controller> logger) : ControllerBase
{
    // ...
}
```

## GET Action (Query)

```csharp
[HttpGet]
[ProducesResponseType(typeof(List<<Entity>Dto>), StatusCodes.Status200OK)]
public async Task<IActionResult> GetAll(CancellationToken ct)
{
    await auditLogger.LogMessageAsync("<Entity> list requested");
    var result = await get<Entity>Query.Execute(ct);
    return Ok(result);
}

[HttpGet("{id:guid}")]
[ProducesResponseType(typeof(<Entity>Dto), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public async Task<IActionResult> GetById([FromRoute] Guid id, CancellationToken ct)
{
    var result = await get<Entity>Query.GetByIdAsync(id, ct);
    if (result is null)
        return NotFound();
    return Ok(result);
}
```

## POST/PUT/DELETE Action (Command)

```csharp
[HttpPost]
[ProducesResponseType(typeof(<Entity>Dto), StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public async Task<IActionResult> Create([FromBody] <Entity>CreateDto dto, CancellationToken ct)
{
    await auditLogger.LogActionAsync("<Entity> creation requested");
    var result = await <entity>CreateHandler.Execute(new <Entity>CreateCommand(dto.Name), ct);
    if (!result.IsSuccess)
        return BadRequest(result.Error);
    return Ok(result.Value);
}

[HttpDelete("{id:guid}")]
[ProducesResponseType(StatusCodes.Status200OK)]
[ProducesResponseType(StatusCodes.Status400BadRequest)]
public async Task<IActionResult> Delete([FromRoute] Guid id, CancellationToken ct)
{
    await auditLogger.LogActionAsync("<Entity> deletion requested");
    var result = await <entity>DeleteHandler.Execute(new <Entity>DeleteCommand(id), ct);
    if (!result.IsSuccess)
        return BadRequest(result.Error);
    return Ok();
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
- **CancellationToken** on every action as the last parameter.

## Forbidden

- Business logic in controllers (move to Application handlers).
- Direct `DbContext` usage (use Application queries/commands).
- Creating `HttpClient` instances (use Refit clients in Presentation).
- Returning domain entities directly (map to DTOs).
