# Copilot Instructions — {{SolutionName}}

<!-- Repo-policy layer: architecture, coding standards, conventions, constraints.
     Agent workflow rules live in AGENTS.md (repo root). -->

You are the coding assistant for the {{SolutionName}} project. Read and investigate relevant files before answering questions or making changes — never speculate about code you have not opened.

<context>
Blazor Server solution: WFE (Blazor Server) · BFF (Web API + NServiceBus + Hangfire) · Worker · SQL Server DACPAC.
Onion/Screaming Architecture, CQRS-lite. All hosts wired via `Nihdi.Core.Configuration`.
.NET 10 · MSTest v4 · Moq · `Microsoft.Testing.Platform` · StyleCop (`TreatWarningsAsErrors`).
</context>

---

<critical_rules>
## Coding Standards

1. **Never throw for business errors** — use `Result<T>` from `Nihdi.Core.Functional`:
```csharp
// Handler → Result<T>.Failure("msg") or Result<T>.Success(value)
// Controller → if (!result.IsSuccess) return BadRequest(result.Error); return Ok();
// Unexpected → _logger.LogError(ex, "{Handler} exception.", nameof(MyHandler)); throw;
```

2. **DI via `*Module` + Scrutor suffix scanning only.** Never `services.AddScoped<T>()` in `Program.cs`.

3. **`AsNoTracking()` on all read queries.** No N+1 patterns.

4. **`EnableAuthentication: false` only in `appsettings.UnitTest.json`.**

5. **Match existing patterns** in the same layer/feature before writing new code.

6. **Analyzer rules** in `Directory.Build.props` are **fixed** — do not modify.
</critical_rules>

---

## Verification Commands

After every code change, run all three:
```
dotnet build src/{{SolutionName}}.sln
{{TestExePath}}
dotnet format src/{{SolutionName}}.sln --verify-no-changes
```
`dotnet test` is not supported — use the `.exe` directly (`Microsoft.Testing.Platform` + .NET 10 SDK).

---

## Scope & Boundaries

<scope>

- **Changes under `src/` only.** One issue per change.
- **Features require `_plans/<FeatureName>.md` + user approval first.**
- **Non-goals**: no new architectures/libraries · no deployment/CI changes · no auth flow changes · no ORM migrations (SQL in `Database/`) · no build-system changes.
- **Risk areas** (require human review): Auth/OIDC · PII · shared contracts · domain/application layers · DB schema · controller routes/DTOs · blob/file storage.
</scope>

---

## Project Structure

```
src/
├── Host/Web/            # Blazor Server (WFE)
├── Host/BFF/            # Web API + NServiceBus + Hangfire
├── Host/Api/            # Secondary Web API
├── Host/Worker/         # Background worker
├── Presentation/        # Razor Class Library (pages, ViewModels, Services)
├── Core/Application/    # Commands, Queries, Handlers (CQRS-lite)
├── Core/Domain/         # Entities, value objects (zero deps)
├── Core/Infrastructure/ # Messaging, external services
├── Core/Persistence/    # EF Core DbContext, repositories
├── Contracts/           # Shared DTOs + NServiceBus contracts
├── Database/            # SQL Server DACPAC
└── Test/{Unit,Bff,Common,UI}/
```

---

## Naming Conventions

| Type | Pattern | Example |
|------|---------|---------|
| Handler | `<Entity><Action>Handler` | `ProductCreateHandler` |
| Command | `<Entity><Action>Command` | `ProductCreateCommand` |
| Query | `I<Entity><Action>Query` / impl | `IProductGetAllQuery` |
| Entity | `<Entity> : IEntity` | `Product` |
| Repository | `I<Entity>Repository` / `<Entity>Repository : BaseRepository<T>` | `IProductRepository` |
| DTO | `<Entity>Dto` (record) | `ProductDto` |
| Controller | `<Entity>Controller` | `ProductController` |
| ViewModel | `<Feature>ViewModel` | `ProductsViewModel` |
| Test class | `<ClassUnderTest>Tests` | `ProductCreateHandlerTests` |
| Test method | `<Method>_<Scenario>_<Expected>` | `Execute_EmptyName_ReturnsFailure` |
| NServiceBus Event | `<Entity><PastTenseVerb>Event` (record) | `ProductCreatedEvent` |
| NServiceBus Message | `<Entity>Message` (class) | `ProductMessage` |

> **Note**: The examples above use `Product` as a placeholder. Replace with your project's **reference feature** entity.

`_camelCase` fields (SA1309 disabled) · `kebab-case` routes · features in `Functionalities/<Feature>/`.

---

## DI Registration (Scrutor Suffix Scanning)

| Suffix | Module | Lifetime |
|--------|--------|----------|
| `Query`, `Handler`, `ICommandHandler<,>` | `ApplicationModule` | Scoped |
| `Repository` | `PersistenceModule` | Scoped |
| `Service`, `Handler` | `InfrastructureModule` | Scoped |
| `ViewModel`, `ServiceClient` | `PresentationModule` | Transient |

Every service needs an `I<Name>` interface for Scrutor to register it.

---

## Dependency Matrix

| Project | May reference |
|---------|--------------|
| `Core.Domain` | _(nothing)_ |
| `Core.Application` | `Core.Domain`, `Contracts` |
| `Core.Infrastructure` | `Core.Application`, `Core.Domain` |
| `Core.Persistence` | `Core.Application`, `Core.Domain` |
| `Contracts` | _(nothing)_ |
| `Presentation` | `Contracts`, `Core.Application` (interfaces only) |
| `Host.Bff` | `Core.*`, `Contracts`, `Presentation` |
| `Host.Wfe` | `Presentation`, `Contracts` |
| `Test.Unit` | `Core.*`, `Contracts` |
| `Test.Bff` | `Host.Bff` (Reqnroll + WebApplicationFactory) |
| `Test.UI` | `Presentation` (bUnit component tests) |
| `Test.Common` | _(shared test utilities, no prod refs)_ |

**Forbidden**: Domain → anything · Application → Infra/Persistence/Host · Contracts → Core/Host · circular refs.
EF Core only in `Core.Persistence`. Domain entities: plain C#.

---

## Test Conventions

- AAA (Arrange/Act/Assert), MSTest v4, `[TestClass]` / `[TestMethod]`.
- Mock at boundary (interfaces only, Moq). No real DB/HTTP in unit tests.
- BFF integration (`Test.Bff`): Reqnroll (SpecFlow) `.feature` files + step definitions + `CustomWebApplicationFactory<Program, {{DbContextName}}>` + SQLite in-memory + `TestAuthenticationHandler`.
- UI component (`Test.UI`): bUnit `BunitTest` base class, MSTest, renders Blazor components in isolation.

---

## Core Patterns

### Error Handling — `Result<T>`

```csharp
// Handler (replace <Entity> with your reference feature entity)
public async Task<Result<<Entity>Dto>> Execute(<Entity>CreateCommand cmd, CancellationToken ct)
{
    try
    {
        if (string.IsNullOrWhiteSpace(cmd.Name))
            return Result<<Entity>Dto>.Failure("Name is required");
        var entity = new <Entity> { Name = cmd.Name };
        await _repository.AddAsync(entity, ct);
        return Result<<Entity>Dto>.Success(new <Entity>Dto(entity.Id, entity.Name));
    }
    catch (Exception ex) { _logger.LogError(ex, "{Handler} exception.", nameof(<Entity>CreateHandler)); throw; }
}

// Controller
var result = await _handler.Execute(cmd, ct);
if (!result.IsSuccess) return BadRequest(result.Error);
return Ok(result.Value);
```

### Auth & User Context

- WFE → OIDC (`AddOpenIdConnectForNihdi`); BFF/Api → JWT (`AddOAuthForNihdi`).
- WFE → BFF with client-credentials.
- `UserContextHeaderHandler` sends `Nihdi-User-Id`/`Nihdi-User-Name`; BFF reads via `HttpUserContextAccessor`.
