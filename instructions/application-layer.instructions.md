---
description: "Layer guidance for Application commands, queries, handlers, and Result<T> pattern. Covers CQRS-lite conventions, IMessagingService usage, validation, and DI registration. Activates when editing Application layer files."
applyTo: "src/Core/Application/**"
---
# Application Layer Conventions

## Query Pattern (Read)

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
        => await _repository.GetBy<Criteria>(<param>);
}
```

## Command + Handler Pattern (Write)

Location: `src/Core/Application/Functionalities/<Feature>/Commands/<Action>/`

```csharp
// Command — record with Validate()
public record <Action><Entity>Command(string Name, string Email)
{
    public (bool IsValid, List<string> Errors) Validate()
    {
        var errors = new List<string>();
        if (string.IsNullOrEmpty(Name)) errors.Add("Name cannot be empty");
        return (!errors.Any(), errors);
    }
}

// Handler — returns Result<Unit>
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

        // Business logic
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

## Result\<T\> — Error Handling

- Use `Result<T>` for all business errors — never throw for expected failures.
- `Result<Unit>` for commands that return no data.
- Check `result.IsSuccess` before accessing the value.
- Map `Result<T>` to HTTP in the controller layer, not here.

## IMessagingService — Async Messaging

When a command needs to publish events or send messages:

```csharp
public class Register<Entity>CommandHandler(
    I<Entity>Repository repository,
    IMessagingService messagingService)
    : ICommandHandler<Register<Entity>Command, Result<Unit>>
{
    public async Task<Result<Unit>> Execute(Register<Entity>Command command)
    {
        // ... business logic ...
        await _repository.SaveChangesAsync();

        await _messagingService.PublishTransactional(
            new <Entity>CreatedEvent(entity.Id, ...));

        return new Result<Unit>(Unit.Default());
    }
}
```

## Rules

- **DI**: auto-registered by `ApplicationModule` — suffix `Query` → Scoped, suffix `CommandHandler` → Scoped.
- **No infrastructure concerns** — no EF, no HTTP. Only `IMessagingService` abstraction.
- **Repository interfaces** live here (`I<Entity>Repository`), implementations live in Persistence.
- **CancellationToken** on every async method signature.
- **One query/command per file.** File matches class name.
- Commands use `record` types with a `Validate()` method.
- Queries return domain entities or DTOs; commands return `Result<Unit>`.

## Forbidden

- Throwing exceptions for business rule violations (use `Result<T>`).
- Direct references to EF Core, `DbContext`, or any persistence types.
- Direct references to `IMessageSession` or `ITransactionalSession` (use `IMessagingService`).
- Controller or HTTP concerns (status codes, `ActionResult`).
