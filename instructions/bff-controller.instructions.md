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
 [HttpGet("{appointmentId}")]
    public async Task<ActionResult<AppointmentDto>> GetAppointment([FromRoute] Guid salonId, [FromRoute] Guid appointmentId)
    {
        try
        {
            var result = await _getAppointmentQuery.Execute(appointmentId);
            return result.Match<ActionResult<AppointmentDto>>(
                appointment => appointment != null ? Ok(appointment.ToAppointmentDto()) : NotFound(),
                error => StatusCode(500, $"Error retrieving appointment: {error.Message}")
            );
        }
        catch (Exception ex)
        {
            return StatusCode(500, $"Error retrieving appointment: {ex.Message}");
        }
    }
```

## POST/PUT/DELETE Action (Command)

```csharp
[HttpPut]
public async Task<ActionResult<ResultDto<Unit>>> CreateAppointment(
    [FromRoute] Guid salonId,
    [FromBody] CreateDiaryAppointmentCommandDto commandDto)
{
    var userId = this.GetUserId();
    if (userId == null)
    {
        return BadRequest("User not found");
    }

    var command = new CreateDiaryAppointmentCommand(
        userId,
        commandDto.Moment.ToDateTime(),
        commandDto.ServiceId,
        commandDto.TeamMemberId,
        commandDto.Language,
        commandDto.CustomerId,
        commandDto.CustomerName);
        
    return (await _handler.Execute(command))
        .Match(
        result => Ok(new ResultDto<Unit>(Unit.Default)),
        err => Ok(new ResultDto<Unit>(err.Select(n => n.Message).ToList())));
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
