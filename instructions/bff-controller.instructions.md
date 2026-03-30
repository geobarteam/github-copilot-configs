---
description: "Layer guidance for BFF controllers, NServiceBus handlers, and API conventions. Covers Result<T> to HTTP mapping, audit logging, RequiresTransactionalSession, and route patterns. Activates when editing BFF host files."
applyTo: "src/Host/BFF/**"
---
# BFF Controller & Handler Conventions

## Controller Pattern

```csharp
namespace {{NamespaceRoot}}.Host.Bff.Controllers;

[ApiController]
[Route("api/[controller]")]
public class <Feature>Controller(
    IGet<Entity>Query get<Entity>Query,
    I<Entity><Action>Handler <entity><Action>Handler,
    IAuditLogger auditLogger) : ControllerBase
{
    /// <summary>Get all entities.</summary>
    [HttpGet]
    [ProducesResponseType(typeof(List<<Entity>Dto>), StatusCodes.Status200OK)]
    public async Task<IActionResult> GetAll(CancellationToken ct)
    {
        await auditLogger.LogMessageAsync("<Entity> list requested");
        var result = await get<Entity>Query.Execute(ct);
        return Ok(result);
    }

    /// <summary>Create a new entity.</summary>
    [HttpPost]
    [RequiresTransactionalSession]  // Required when publishing NServiceBus events
    [ProducesResponseType(typeof(<Entity>Dto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<IActionResult> Create([FromBody] <Entity><Action>Command command, CancellationToken ct)
    {
        await auditLogger.LogActionAsync("<Entity> creation requested");
        var result = await <entity><Action>Handler.Execute(command, ct);
        if (!result.IsSuccess)
            return BadRequest(result.Error);

        return Ok(result.Value);
    }
}
```

## Rules

- **Result<T> → HTTP**: `result.IsSuccess` → `Ok()` / `!result.IsSuccess` → `BadRequest(result.Error)`.
- **Routes**: `kebab-case` — `[Route("api/my-feature")]`.
- **Audit logging**: `IAuditLogger.LogMessageAsync()` for reads, `IAuditLogger.LogActionAsync()` for writes.
- **`[RequiresTransactionalSession]`** on any action that publishes NServiceBus events.
- **`CancellationToken`** on every action method.
- `[ProducesResponseType]` on every action.

## NServiceBus Handlers (BFF)

Location: `src/Host/BFF/NServiceBusHandlers/`

```csharp
namespace {{NamespaceRoot}}.Host.Bff.NServiceBusHandlers;

public class <Entity>MessageHandler(
    I<Entity><Action>Handler handler,
    ILogger<<Entity>MessageHandler> logger) : IHandleMessages<<Entity>Message>
{
    public async Task Handle(<Entity>Message message, IMessageHandlerContext context)
    {
        // Known error → log warning + return (no retry)
        // Unknown error → log + throw (triggers NServiceBus retry)
        try
        {
            var command = new <Entity><Action>Command(message.Id);
            var result = await handler.Execute(command, context.CancellationToken);
            if (!result.IsSuccess)
            {
                logger.LogWarning("Known error: {Error}", result.Error);
                return;
            }
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Unexpected error handling {Message}.", nameof(<Entity>Message));
            throw;
        }
    }
}
```
