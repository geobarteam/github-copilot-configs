---
name: build-feature
description: "Use when implementing a new feature, vertical slice, or multi-layer change in {{SolutionName}} after a _plans/<FeatureName>.md (repo root) is approved. Provides step-by-step Red-Green-Refactor implementation procedure with real code templates for each layer: Domain entity, Application command/query/handler, Persistence repository, Contracts DTOs, BFF controller, Database SQL, Presentation ViewModel/ServiceClient/Refit. Use for: building new features, adding CRUD operations, implementing vertical slices, following the 7-step layer workflow."
argument-hint: "Which plan step to implement, e.g. 'Step 1 — Domain entity'"
---

# Build Feature — Implementation Skill

Execute approved `_plans/<FeatureName>.md` steps using Red-Green-Refactor, one step per reply, following existing patterns from the **reference feature** specified in the plan.

## Prerequisites

- A **`_plans/<FeatureName>.md`** (repo root) must exist and be **approved by the user** before using this skill.
- The plan's **Overview** section names the reference feature. If it doesn't, **ask the user** which existing feature to use as the pattern before proceeding.
- Read the plan step you're implementing. Read the reference feature files for that layer.
- **Do not skip ahead.** Complete one step, prove it, stop at the gate.

## Progress Tracking

The plan file is a **living document**. Track progress using the HUMAN GATE checkboxes:

- **Before starting**: read `_plans/<FeatureName>.md` and find the **first step with unchecked `[ ]`** checkboxes — that is the next step to implement. All steps with `[x]` are already completed and approved.
- **After user approves a step**: update `_plans/<FeatureName>.md` — change `[ ]` to `[x]` on every checkbox in that step's **🛑 HUMAN GATE** section.
- This ensures continuity across sessions — if the conversation restarts, any agent can read the plan and know exactly where to resume.

## Mandatory Workflow Per Step

```
1. READ the plan step from `_plans/<FeatureName>.md` — understand scope, files, test method
2. RED — write the failing test FIRST
3. RUN — {{TestExePath}} --filter "<TestMethod>" → confirm FAIL
4. GREEN — write minimal production code to pass
5. RUN — {{TestExePath}} --filter "<TestMethod>" → confirm PASS
6. REFACTOR — cleanup if needed
7. CODE ANALYSIS — fix all violations before proving:
   a. dotnet build src/{{SolutionName}}.sln 2>&1 | Select-String -Pattern ": (warning|error) (SA|SX|CA|CS|MSTEST)\d+" | Sort-Object | Get-Unique
   b. dotnet format src/{{SolutionName}}.sln  (auto-fix formatting)
   c. Fix any remaining violations manually (see @code-analysis agent ruleset — skip disabled rules)
   d. Repeat a–c until the Select-String output is empty
8. PROVE:
   - dotnet build src/{{SolutionName}}.sln → succeeded (zero warnings/errors)
   - {{TestExePath}} → all pass
   - dotnet format src/{{SolutionName}}.sln --verify-no-changes → exit 0
9. 🛑 STOP — present results, wait for user approval
10. MARK DONE — once the user approves, update `_plans/<FeatureName>.md`: change `[ ]` → `[x]` on this step's HUMAN GATE checkboxes
```

> **Note**: `dotnet test` is not supported — `Microsoft.Testing.Platform` + .NET 10 SDK requires running the test `.exe` directly. If packages are missing, run `dotnet restore src/{{SolutionName}}.sln --interactive` first.

**Never batch steps. Never skip RED. Never skip CODE ANALYSIS. Never proceed past 🛑 without user confirmation.**

---

## Layer Templates (from the reference feature)

### Step 1 — Domain Entity

Location: `src/Core/Domain/Shared/Entities/<Entity>.cs`

```csharp
namespace {{NamespaceRoot}}.Core.Domain.Shared.Entities;

public class <Entity> : IEntity
{
    public int Id { get; set; }
    // Properties — plain C#, no EF attributes, no dependencies
}
```

Interface: `src/Core/Domain/Shared/IEntity.cs` (already exists — `int Id { get; set; }`)

Test: construction + property assignment.

---

### Step 2 — Repository Interface + Implementation

**Interface** in `src/Core/Application/Shared/Interfaces/Persistence/Repositories/I<Entity>Repository.cs`:

```csharp
namespace {{NamespaceRoot}}.Core.Application.Shared.Interfaces.Persistence.Repositories;

public interface I<Entity>Repository : IRepository<<Entity>>
{
    // Feature-specific query methods
    Task<IReadOnlyList<<Entity>>> Search<Entity>(string param1, string param2);
}
```

**Implementation** in `src/Core/Persistence/Repositories/<Entity>Repository.cs`:

```csharp
namespace {{NamespaceRoot}}.Core.Persistence.Repositories;

using Microsoft.EntityFrameworkCore;

public class <Entity>Repository({{DbContextName}} dbContext)
    : BaseRepository<<Entity>>(dbContext), I<Entity>Repository
{
    public async Task<IReadOnlyList<<Entity>>> Search<Entity>(string param1, string param2)
    {
        var query = dbContext.<Entities>.AsQueryable();

        if (!string.IsNullOrWhiteSpace(param1))
            query = query.Where(e => e.Prop.Contains(param1));

        return await query.AsNoTracking().ToListAsync();  // Always AsNoTracking for reads
    }
}
```

**EF Configuration** in `src/Core/Persistence/EntityTypeConfigurations/<Entity>Configuration.cs`:

```csharp
namespace {{NamespaceRoot}}.Core.Persistence.EntityTypeConfigurations;

using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

public class <Entity>Configuration : IEntityTypeConfiguration<<Entity>>
{
    public void Configure(EntityTypeBuilder<<Entity>> builder)
    {
        builder.ToTable("<Entity>", schema: "<Schema>");
        builder.HasKey(p => p.Id);
        builder.Property(p => p.Name).IsRequired().HasMaxLength(50);
        // No EF attributes on entity — all config here via fluent API
    }
}
```

DI: auto-registered by `PersistenceModule` (suffix `Repository` → Scoped).

---

### Step 3 — Command + Handler

**Command** in `src/Core/Application/Functionalities/<Feature>/Commands/<Action>/Add<Entity>Command.cs`:

```csharp
namespace {{NamespaceRoot}}.Core.Application.Functionalities.<Feature>.Commands.<Action>;

public record Add<Entity>Command(string Name, string Email)
{
    public (bool IsValid, List<string> Errors) Validate()
    {
        var errors = new List<string>();
        if (string.IsNullOrEmpty(Name)) errors.Add("Name cannot be empty");
        return (!errors.Any(), errors);
    }
}
```

**Handler** in same folder `Add<Entity>CommandHandler.cs`:

```csharp
namespace {{NamespaceRoot}}.Core.Application.Functionalities.<Feature>.Commands.<Action>;

using Nihdi.Core.Functional;

public class Add<Entity>CommandHandler(I<Entity>Repository repository)
    : ICommandHandler<Add<Entity>Command, Result<Unit>>
{
    private readonly I<Entity>Repository _repository = repository
        ?? throw new ArgumentNullException(nameof(repository));

    public async Task<Result<Unit>> Execute(Add<Entity>Command command)
    {
        Result<Unit> validateResult = ValidateCommand(command);
        if (!validateResult.IsSuccess)
            return validateResult;

        var entity = new <Entity> { Name = command.Name };
        await _repository.AddAsync(entity);
        await _repository.SaveChangesAsync();

        return new Result<Unit>(Unit.Default());
    }

    private static Result<Unit> ValidateCommand(Add<Entity>Command command)
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

DI: auto-registered by `ApplicationModule` (suffix `Handler` → Scoped).

**Query** in `src/Core/Application/Functionalities/<Feature>/Queries/<Action>/`:

```csharp
// Interface
public interface IGet<Entity>ListQuery
{
    Task<List<<Entity>>> Execute(string param1, string param2);
}

// Implementation
public class Get<Entity>ListQuery(I<Entity>Repository repository) : IGet<Entity>ListQuery
{
    private readonly I<Entity>Repository _repository = repository
        ?? throw new ArgumentNullException(nameof(repository));

    public async Task<List<<Entity>>> Execute(string param1, string param2)
        => [.. await _repository.Search<Entity>(param1, param2)];
}
```

DI: auto-registered by `ApplicationModule` (suffix `Query` → Scoped).

Test: mock `I<Entity>Repository`, verify handler returns `Result<Unit>` success/failure.

---

### Step 4 — Contracts (DTOs)

Location: `src/Contracts/<Feature>/Api/`

```csharp
// Read DTO
namespace {{NamespaceRoot}}.Contracts.<Feature>.Api;
public record <Entity>Dto(int Id, string Name, string Email);

// Write DTO
public record Add<Entity>Dto(string Name, string Email);
```

DTOs are `record` types. No logic. No domain dependencies.

---

### Step 5 — BFF Controller

Location: `src/Host/BFF/Controllers/<Feature>Controller.cs`:

```csharp
namespace {{NamespaceRoot}}.Host.Bff.Controllers;

[ApiController]
[Route("api/<feature-kebab>")]
public class <Feature>Controller(
    IGet<Entity>ListQuery getQuery,
    ICommandHandler<Add<Entity>Command, Result<Unit>> addCommand,
    ILogger<<Feature>Controller> logger,
    IAuditLogger auditLogger,
    ICorrelationContextAccessor correlationContextAccessor,
    IUserContextAccessor userContextAccessor) : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<IEnumerable<<Entity>Dto>>> Get(
        CancellationToken cancellationToken,
        [FromQuery] string name = null)
    {
        try
        {
            var entities = await getQuery.Execute(name, ...);
            // Audit logging
            return Ok(entities.Select(e => new <Entity>Dto(e.Id, e.Name, e.Email)));
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "{Controller} has throw an exception.", nameof(<Feature>Controller));
            throw;
        }
    }

    [HttpPost]
    public async Task<ActionResult> Add([FromBody] Add<Entity>Dto dto, CancellationToken cancellationToken)
    {
        try
        {
            var result = await addCommand.Execute(new Add<Entity>Command(dto.Name, dto.Email));
            if (!result.IsSuccess)
                return BadRequest(result.Error);
            return Ok();
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "{Controller} has throw an exception.", nameof(<Feature>Controller));
            throw;
        }
    }
}
```

Test: `Test/Integration/Endpoints/` — HTTP tests with `CustomWebApplicationFactory`.

---

### Step 6 — Database SQL

Location: `src/Database/Tables/<Entity>.sql`

```sql
CREATE TABLE [<Schema>].[<Entity>] (
    [Id]     INT           IDENTITY (1, 1) NOT NULL,
    [Name]   VARCHAR (50)  NOT NULL,
    -- Encrypted columns use: COLLATE Latin1_General_BIN2 ENCRYPTED WITH (...)
    CONSTRAINT [PK_<Entity>] PRIMARY KEY CLUSTERED ([Id] ASC)
);
```

**User deploys via Schema Compare** — never modify `.sqlproj` directly.

---

### Step 7 — Presentation Layer

**Refit Client** in `src/Presentation/Shared/ServiceClients/Bff/Clients/I<Feature>Client.cs`:

```csharp
namespace {{NamespaceRoot}}.Presentation.Shared.ServiceClients.Bff.Clients;

using Refit;

public interface I<Feature>Client
{
    [Get("/api/<feature-kebab>")]
    Task<ICollection<<Entity>Dto>> Get<Feature>Async(
        [Query] string name = null,
        [AliasAs("api-version")][Query] string apiVersion = null,
        CancellationToken cancellationToken = default);

    [Post("/api/<feature-kebab>")]
    Task Add<Entity>Async(
        [Body] Add<Entity>Dto dto,
        [AliasAs("api-version")][Query] string apiVersion = null,
        CancellationToken cancellationToken = default);
}
```

**Feature ServiceClient** in `src/Presentation/<Feature>/ServiceClients/<Feature>ServiceClient.cs`:

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.ServiceClients;

using {{NamespaceRoot}}.Presentation.Shared.ServiceClients.Bff;

public class <Feature>ServiceClient(I<Feature>Client client) : I<Feature>ServiceClient
{
    private readonly I<Feature>Client _client = client
        ?? throw new ArgumentNullException(nameof(client));

    public async Task<IEnumerable<<Entity>Model>> GetAllAsync()
    {
        var items = await _client.Get<Feature>Async(null, ApiConstants.ApiVersion);
        return items.Select(x => new <Entity>Model(x.Name, x.Email));
    }

    public async Task<Result<Unit>> AddAsync(string name, string email)
    {
        try
        {
            await _client.Add<Entity>Async(new Add<Entity>Dto(name, email), ApiConstants.ApiVersion);
        }
        catch (ApiException ex)
        {
            return ex.ConvertApiExceptionToResult<Unit>();
        }
        return new Result<Unit>(Unit.Default());
    }
}
```

DI: auto-registered by `PresentationModule` (suffix `ServiceClient` → Transient).

**ViewModel** in `src/Presentation/<Feature>/ViewModels/<Feature>ViewModel.cs`:

```csharp
namespace {{NamespaceRoot}}.Presentation.<Feature>.ViewModels;

public class <Feature>ViewModel(I<Feature>ServiceClient serviceClient) : I<Feature>ViewModel
{
    private readonly I<Feature>ServiceClient _serviceClient = serviceClient;

    public bool IsBusy { get; set; }
    public IList<<Entity>Model> Items { get; set; }

    public async Task InitializeAsync(IErrorComponent errorComponent)
    {
        IsBusy = true;
        try
        {
            var items = await _serviceClient.GetAllAsync();
            Items = [.. items];
        }
        catch (Exception ex)
        {
            errorComponent.ProcessError(ex);
        }
        finally
        {
            IsBusy = false;
        }
    }
}
```

DI: auto-registered by `PresentationModule` (suffix `ViewModel` → Transient).

---

## Key Reminders

- **ICommandHandler<TCommand, TResult>** is the Application-layer command interface. Handlers are auto-scanned.
- **Result<T>** from `Nihdi.Core.Functional` — constructor `new Result<T>(value)` for success, `new Result<T>("error msg")` for failure. Check `result.IsSuccess`.
- **Unit** from `Nihdi.Core.Functional` — use `Unit.Default()` for void-equivalent returns.
- **ApiException** — catch in ServiceClient, convert via `ex.ConvertApiExceptionToResult<T>()`.
- **BaseRepository<T>** — provides `AddAsync`, `GetByIdAsync`, `ListAllAsync`, `Update`, `Delete`, `SaveChangesAsync`.
- **DbSet registration** — add `DbSet<<Entity>>` to `{{DbContextName}}`.
- **Copyright header** — all files need: `// <copyright file="<File>.cs" company="Riziv-Inami"> ... </copyright>`
- **CancellationToken** — propagate on every async call in controllers. Handlers may omit if not needed by repository.
