---
description: "Layer guidance for Application commands, queries, and handlers. Covers CQRS-lite patterns, Result<T> error handling, repository interfaces, and handler conventions. Activates when editing Application layer files."
applyTo: "src/Core/Application/**"
---
# Application Layer Conventions

## Commands

```csharp
namespace MyApp.Core.Application.Functionalities.<Feature>.Commands.<Action>;

public record <Entity><Action>Command(string Name, int DoctorId);
```

## Command Handlers

```csharp
public class <Entity><Action>Handler(
    I<Entity>Repository <entity>Repository,
    ILogger<<Entity><Action>Handler> logger) : ICommandHandler<<Entity><Action>Command, <Entity>Dto>
{
    public async Task<Result<<Entity>Dto>> Execute(<Entity><Action>Command command, CancellationToken ct)
    {
        try
        {
            if (string.IsNullOrWhiteSpace(command.Name))
                return Result<<Entity>Dto>.Failure("Name is required");

            var entity = new <Entity> { Name = command.Name };
            await <entity>Repository.AddAsync(entity, ct);
            return Result<<Entity>Dto>.Success(new <Entity>Dto(entity.Id, entity.Name));
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "{Handler} exception.", nameof(<Entity><Action>Handler));
            throw;
        }
    }
}
```

## Queries

```csharp
// Interface
namespace MyApp.Core.Application.Functionalities.<Feature>.Queries.<Action>;

public interface IGet<Entity>Query
{
    Task<List<<Entity>>> Execute(CancellationToken ct);
}

// Implementation
public class Get<Entity>Query(I<Entity>Repository <entity>Repository) : IGet<Entity>Query
{
    public async Task<List<<Entity>>> Execute(CancellationToken ct)
    {
        return [.. await <entity>Repository.ListAllAsync(ct)];
    }
}
```

## Repository Interfaces

```csharp
namespace MyApp.Core.Application.Shared.Interfaces.Persistence.Repositories;

public interface I<Entity>Repository : IRepository<<Entity>>
{
    Task<IReadOnlyList<<Entity>>> GetBy<Criteria>Async(string value);
}
```

## Rules

- **Never throw for business errors** — return `Result<T>.Failure("message")`.
- Unexpected exceptions: `logger.LogError(ex, ...)` then re-throw.
- Validate inputs with `Preconditions.NotEmpty()` or guard clauses.
- `CancellationToken` on every async method.
- Application may reference Domain and Contracts only. **Never reference Persistence, Infrastructure, or Host.**
- DI registration: `*Query` and `*Handler` are scoped (registered via `ApplicationModule` + Scrutor).

## File Locations

| Type | Path |
|------|------|
| Command | `Functionalities/<Feature>/Commands/<Action>/<Entity><Action>Command.cs` |
| Handler | `Functionalities/<Feature>/Commands/<Action>/<Entity><Action>Handler.cs` |
| Query interface | `Functionalities/<Feature>/Queries/<Action>/IGet<Entity>Query.cs` |
| Query impl | `Functionalities/<Feature>/Queries/<Action>/Get<Entity>Query.cs` |
| Repo interface | `Shared/Interfaces/Persistence/Repositories/I<Entity>Repository.cs` |
