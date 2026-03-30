---
description: "NServiceBus conventions for events, messages, and handler implementations. Events are records in Contracts; messages are classes. Handlers use IHandleMessages<T>. Application layer must use IMessagingService, never IMessageSession. Activates when editing NServiceBus-related files."
applyTo: "src/**/NServiceBusHandlers/**,src/**/Handlers/**,src/Contracts/**/Events/**,src/Contracts/**/Messages/**"
---
# NServiceBus Conventions

## Events vs Messages

| Type | Location | Shape | Purpose |
|------|----------|-------|---------|
| Event | `Contracts/<Feature>/Events/` | `record` | Pub/sub notifications |
| Message | `Contracts/<Feature>/Messages/` | `class` | Point-to-point commands |

### Event (record)

```csharp
namespace {{NamespaceRoot}}.Contracts.<Feature>.Events;

public record <Entity><PastTenseVerb>Event(Guid Id, string Name);
```

### Message (class)

```csharp
namespace {{NamespaceRoot}}.Contracts.<Feature>.Messages;

public class <Entity>Message
{
    public Guid Id { get; set; }
}
```

## Publishing from Application Layer

**Never** use `IMessageSession` in Application. Always use the `IMessagingService` abstraction:

```csharp
// In a command handler (Application layer)
await _messagingService.PublishAsync(new <Entity>CreatedEvent(entity.Id, entity.Name), ct);
```

## Handler Pattern

BFF handlers: `src/Host/BFF/NServiceBusHandlers/`
Worker handlers: `src/Host/Worker/Handlers/`

All handlers implement `IHandleMessages<T>`:

```csharp
public class <Entity><PastTenseVerb>EventHandler(
    I<Entity><Action>Handler handler,
    ILogger<<Entity><PastTenseVerb>EventHandler> logger) : IHandleMessages<<Entity><PastTenseVerb>Event>
{
    public async Task Handle(<Entity><PastTenseVerb>Event message, IMessageHandlerContext context)
    {
        try
        {
            var command = new <Entity><Action>Command(message.Id, message.Name);
            var result = await handler.Execute(command, context.CancellationToken);

            if (!result.IsSuccess)
            {
                // Known business error → log warning, do NOT throw (prevents retry)
                logger.LogWarning("Known error handling {Event}: {Error}",
                    nameof(<Entity><PastTenseVerb>Event), result.Error);
                return;
            }
        }
        catch (Exception ex)
        {
            // Unknown error → log + throw (triggers NServiceBus retry policy)
            logger.LogError(ex, "Unexpected error handling {Event}.",
                nameof(<Entity><PastTenseVerb>Event));
            throw;
        }
    }
}
```

## Rules

- `[RequiresTransactionalSession]` on BFF controller actions that publish events.
- Known error → `LogWarning` + `return`. Unknown → `LogError` + `throw`.
- Events are immutable records. Messages are mutable classes.
- Naming: Events use past tense (`Created`, `Updated`), Messages use noun (`<Entity>Message`).
