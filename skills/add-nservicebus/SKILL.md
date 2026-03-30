---
name: add-nservicebus
description: "Use when adding NServiceBus events, messages, or handlers to {{SolutionName}}. Covers event records, message classes, BFF/Worker handlers, IMessagingService usage, transactional sessions, and error handling patterns. Use for: publishing events from commands, handling events in Worker, sending messages between hosts, adding saga steps."
argument-hint: "Describe the event/message, e.g. 'publish PrescriptionCreatedEvent when a prescription is registered'"
---

# Add NServiceBus — Events, Messages & Handlers

Add NServiceBus messaging to an existing feature. Follow the patterns from the Appointment feature.

## Infrastructure — Nihdi.Core.Configuration.NServiceBus

NServiceBus is configured through the `Nihdi.Core.Configuration.NServiceBus` library. **You should never configure NServiceBus manually** — use the Nihdi wrappers.

### How it's wired (already done for BFF + Worker)

**BFF** (`src/Host/BFF/WebApplicationBuilderExtensions.cs`):

```csharp
builder.Host.UseNServiceBusForNihdi(nihdiConfiguration, loggerFactory)
    .AddNServiceBusSynchronizedDbContext<{{DbContextName}}>(connectionString, true);
```

**Worker** (`src/Host/Worker/IHostApplicationBuilderExtensions.cs`):

```csharp
builder.UseNServiceBusForNihdi(nihdiConfiguration, loggerFactory)
    .AddNServiceBusSynchronizedDbContext<{{DbContextName}}>(connectionString, true);
```

`AddNServiceBusSynchronizedDbContext` synchronizes the EF Core `DbContext` with NServiceBus transactional sessions, enabling atomic DB + message operations.

### appsettings.json — `Nihdi:NServiceBus` section

Each host that uses NServiceBus needs this config block:

```json
"NServiceBus": {
  "PersistenceType": "SqlServer",
  "SqlPersistence": {
    "ConnectionStringName": "{{SolutionName}}",
    "SchemaName": "NServiceBus"
  },
  "Transport": {
    "Type": "AzureServiceBus",
    "AzureServiceBusTransport": {
      "ConnectionString": "asb-riziv-it-dev-001.servicebus.windows.net",
      "DataBusStorageAccountName": "sarizividevasb001"
    },
    "EnableOutbox": true,
    "TransactionMode": "ReceiveOnly",
    "SendOnly": false
  },
  "EnableMessageEncryption": false,
  "CertificateSubjectName": "ASB-NIHDI-TST"
}
```

| Key | Values / Default | Purpose |
|-----|------------------|---------|
| `PersistenceType` | `"SqlServer"` / `"None"` | Saga, subscription, outbox storage |
| `SqlPersistence.ConnectionStringName` | references `ConnectionStrings` key | DB for NServiceBus tables |
| `SqlPersistence.SchemaName` | `"NServiceBus"` | SQL schema for NServiceBus tables |
| `Transport.Type` | `"AzureServiceBus"` / `"SqlServer"` / `"Learning"` | Transport mechanism |
| `Transport.EnableOutbox` | `true` / `false` | Idempotent message processing (requires SqlPersistence) |
| `Transport.TransactionMode` | `"ReceiveOnly"` / `"TransactionScope"` / `"None"` | Transaction boundary |
| `Transport.SendOnly` | `false` (BFF+Worker) | `true` = only sends, no receive queue |
| `EnableMessageEncryption` | `false` / `true` | Encrypt messages via certificate |
| `CertificateSubjectName` | `"ASB-NIHDI-TST"` | Certificate for encryption/decryption |

### UseNServiceBusForNihdi options (advanced)

```csharp
builder.Host.UseNServiceBusForNihdi(nihdiConfiguration, loggerFactory, options =>
{
    options.EndpointName = "MyCustomEndpoint";  // default: nihdiConfiguration.Application.EndpointQueueName()
    options.RouteConfigurator = (routingSettings, endpointName, destinations, log) => routingSettings;
    options.PipelineConfigurator = pipelineSettings => pipelineSettings;
    // options.ExcludeAssemblies = new[] { "SomeAssembly.dll" };
    // options.ExcludeTypes = new[] { typeof(MyExcludedType) };
});
```

### IMessagingService — the Application-layer abstraction

`IMessagingService` (in `Core/Application/Shared/Interfaces/Infrastructure/Messaging/`) wraps `ITransactionalSession` + `IMessageSession`. It is implemented by `MessagingService` in `Core/Infrastructure/Messaging/` and auto-registered by `InfrastructureModule` (suffix `Service`).

```csharp
public interface IMessagingService : IDisposable
{
    Task PublishTransactional<T>(T message) where T : notnull;
    Task SendTransactional<T>(string destination, T message) where T : notnull;
    Task SendLocalTransactional<T>(T message) where T : notnull;
    Task Publish<T>(T message) where T : notnull;
    Task Send<T>(string destination, T message) where T : notnull;
    Task SendLocal<T>(T message) where T : notnull;
}
```

**Transactional methods** use `ITransactionalSession` (committed with DB transaction via outbox).
**Non-transactional methods** use `IMessageSession` (fire-and-forget, no DB coordination).

### Host topology

| Host | `SubSystemType` | `SendOnly` | Role |
|------|-----------------|------------|------|
| **BFF** | `bff` | `false` | Sends + receives messages. Controllers publish events via `IMessagingService`. Handlers in `NServiceBusHandlers/`. |
| **Worker** | `wrk` | `false` | Background processing. Handles events from BFF or external systems. Handlers in `Handlers/`. |
| **Web** | `wfe` | N/A | No NServiceBus — Blazor Server UI only. Calls BFF via Refit. |

### DI modules per host

**BFF** — `BfboModule.RegisterDependencies()` registers `PersistenceModule` + `ApplicationModule` + `InfrastructureModule`.
**Worker** — `WorkerModule.RegisterServices()` registers `PersistenceModule` + `ApplicationModule` + `InfrastructureModule`.

Both hosts share the same Application/Infrastructure/Persistence layers. NServiceBus handler classes are in the Host project and discovered automatically by NServiceBus assembly scanning.

---

## Decision: Event vs Message

| Type | When | Shape | Location |
|------|------|-------|----------|
| **Event** | Something happened (notify subscribers) | `record` | `Contracts/<Feature>/Events/` |
| **Message** | Direct command to another endpoint | `class` with `get; set;` properties | `Contracts/<Feature>/Messages/` |

Events use **publish/subscribe** — multiple handlers can react.
Messages use **send** — exactly one handler receives it.

---

## Step 1 — Define the Contract

### Event (record)

```csharp
namespace {{NamespaceRoot}}.Contracts.<Feature>.Events;

public record <Entity><PastTenseVerb>Event(int Id, <DtoType> <Prop>, ...);
```

Events use `record` types. Include only the data subscribers need. Reference shared DTOs from `Contracts/Shared/` or define feature-specific DTOs in the same `Events/` folder:

```csharp
// Supporting DTO in same Events/ folder
namespace {{NamespaceRoot}}.Contracts.<Feature>.Events;
public record <Entity>Dto(int Id, string Name);
```

### Message (class)

```csharp
namespace {{NamespaceRoot}}.Contracts.<Feature>.Messages;

public class <Entity>Message
{
    public int Id { get; set; }
    public string Body { get; set; }
    // Use get; set; — NServiceBus needs mutable properties for serialization
    // For large payloads: public string LargeBodyDataBus { get; set; }
}
```

---

## Step 2 — Publish from Application Layer

**Never call `IMessageSession` directly from Application.** Use the `IMessagingService` abstraction.

### In a Command Handler

```csharp
public class Register<Entity>CommandHandler(
    I<Entity>Repository repository,
    IMessagingService messagingService)
    : ICommandHandler<Register<Entity>Command, Result<Unit>>
{
    public async Task<Result<Unit>> Execute(Register<Entity>Command command)
    {
        // ... business logic ...
        await _repository.AddAsync(entity);
        await _repository.SaveChangesAsync();

        // Publish event (transactional — committed with DB transaction)
        await _messagingService.PublishTransactional(
            new <Entity>CreatedEvent(entity.Id, new <Entity>Dto(entity.Id, entity.Name)));

        return new Result<Unit>(Unit.Default());
    }
}
```

### IMessagingService Methods

See full interface in [Infrastructure section above](#imessagingservice--the-application-layer-abstraction). Quick reference:

| Method | Use case |
|--------|----------|
| `PublishTransactional<T>(message)` | Event within DB transaction (most common) |
| `SendTransactional<T>(destination, message)` | Message to specific endpoint within DB transaction |
| `Publish<T>(message)` | Event outside transaction (fire-and-forget) |
| `Send<T>(destination, message)` | Message outside transaction |
| `SendLocal<T>(message)` | Message to self (same endpoint) |
| `SendLocalTransactional<T>(message)` | Message to self within transaction |

### Controller: Enable Transactional Session

The controller action that triggers the command **must** have `[RequiresTransactionalSession]`:

```csharp
[HttpPost]
[RequiresTransactionalSession]
public async Task<ActionResult> Register([FromBody] RegisterDto dto, CancellationToken ct)
{
    // ... the handler inside will call PublishTransactional
}
```

This ensures the DB write + event publish are atomic.

---

## Step 3 — Handle in BFF or Worker

### Where to place handlers

| Host | Location | Use case |
|------|----------|----------|
| **BFF** | `src/Host/BFF/NServiceBusHandlers/` | Handle messages sent to BFF |
| **Worker** | `src/Host/Worker/Handlers/` | Handle events/messages for background processing |

### Worker Event Handler (full pattern)

```csharp
namespace {{NamespaceRoot}}.Host.Worker.Handlers;

using {{NamespaceRoot}}.Contracts.<Feature>.Events;
using NServiceBus;

public class <Entity><PastTenseVerb>EventHandler(
    TransactionalCommandExecutor commandExecutor,
    ILogger<<Entity><PastTenseVerb>EventHandler> logger) : IHandleMessages<<Entity><PastTenseVerb>Event>
{
    private readonly TransactionalCommandExecutor _commandExecutor = commandExecutor;
    private readonly ILogger<<Entity><PastTenseVerb>EventHandler> _logger = logger
        ?? throw new ArgumentNullException(nameof(logger));

    public async Task Handle(<Entity><PastTenseVerb>Event message, IMessageHandlerContext context)
    {
        try
        {
            _logger.LogInformation("Received {EventName}: {@Event}", nameof(<Entity><PastTenseVerb>Event), message);

            // Validate the event
            var validationResult = message.Validate();
            if (!validationResult.IsSuccess)
            {
                // Known error — log warning and return (don't retry)
                _logger.LogWarning("{EventName} validation failed: {Error}",
                    nameof(<Entity><PastTenseVerb>Event), validationResult.Error);
                return;
            }

            // Create and execute command
            var command = new <Action>Command(message.Id, message.Prop);
            var result = await _commandExecutor.Execute<<Action>Command, Unit>(command);

            if (!result.IsSuccess)
            {
                // Known error — log warning and return (don't retry)
                _logger.LogWarning("Known error: {Error}", result.Error);
                return;
            }
        }
        catch (Exception ex)
        {
            // Unknown error — log and throw (triggers NServiceBus retry)
            _logger.LogError(ex, "Error handling {EventName}", nameof(<Entity><PastTenseVerb>EventHandler));
            throw;
        }
    }
}
```

### BFF Message Handler

```csharp
namespace {{NamespaceRoot}}.Host.Bff.NServiceBusHandlers;

using {{NamespaceRoot}}.Contracts.<Feature>.Messages;
using NServiceBus;

public class <Entity>MessageHandler(
    ICommandHandler<<Action>Command, Result<Unit>> commandHandler,
    ILogger<<Entity>MessageHandler> logger) : IHandleMessages<<Entity>Message>
{
    private readonly ICommandHandler<<Action>Command, Result<Unit>> _commandHandler = commandHandler;
    private readonly ILogger<<Entity>MessageHandler> _logger = logger;

    public async Task Handle(<Entity>Message message, IMessageHandlerContext context)
    {
        try
        {
            await _commandHandler.Execute(new <Action>Command(message.Id));
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error handling {MessageType}", nameof(<Entity>Message));
            throw;
        }
    }
}
```

---

## Error Handling Rules

| Situation | Action | Retried? |
|-----------|--------|----------|
| Validation failure | `LogWarning` + `return` | No |
| Known business error (`!result.IsSuccess`) | `LogWarning` + `return` | No |
| Unknown/unexpected exception | `LogError` + `throw` | Yes (NServiceBus retry policy) |

**Never swallow unknown exceptions.** Rethrowing keeps the message in the queue for retry.

---

## Validation Extension Pattern

Create a validation extension for your event in `Core/Application/Shared/Validation/`:

```csharp
public static class <Entity><PastTenseVerb>EventExtensions
{
    public static Result<Unit> Validate(this <Entity><PastTenseVerb>Event evt)
    {
        if (evt.Id <= 0) return new Result<Unit>("Invalid Id");
        if (string.IsNullOrEmpty(evt.Prop)) return new Result<Unit>("Prop is required");
        return new Result<Unit>(Unit.Default());
    }
}
```

---

## Checklist

- [ ] Event = `record` in `Contracts/<Feature>/Events/`
- [ ] Message = `class` with `get; set;` in `Contracts/<Feature>/Messages/`
- [ ] Application layer uses `IMessagingService` — never `IMessageSession`
- [ ] Controller action has `[RequiresTransactionalSession]` when publishing transactionally
- [ ] Handler implements `IHandleMessages<T>`
- [ ] Known errors → `LogWarning` + return
- [ ] Unknown errors → `LogError` + throw (allows retry)
- [ ] Event validation extension in `Shared/Validation/`
- [ ] Unit test for command handler (mock `IMessagingService`)
- [ ] Copyright header on all new files
