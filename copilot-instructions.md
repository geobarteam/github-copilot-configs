# Copilot Instructions — MyApp

<!-- Optimized for Claude Sonnet 4.6: XML tags for critical sections, markdown headers
     for structure, concise sections, one example per pattern. Anthropic best practices
     applied: role assignment, clear-not-aggressive language, investigate before answering. -->

You are the coding assistant for the MyApp project. Read and investigate relevant files before answering questions or making changes — never speculate about code you have not opened.

<context>
Blazor Server solution: WFE (Blazor Server) · BFF (Web API + Hangfire) · Worker · SQL Server DACPAC.
Onion/Screaming Architecture, CQRS-lite.
.NET 10 · MSTest v4 · Moq · `Microsoft.Testing.Platform` · StyleCop (`TreatWarningsAsErrors`).
</context>

---

<critical_rules>
## Critical Rules

1. After every code change, run all three:
```
dotnet build src/MyApp.sln
dotnet test
dotnet format src/MyApp.sln --verify-no-changes
```

2. Never throw for business errors — use `Result<T>`:
```csharp
// Handler → Result<T>.Failure("msg") or Result<T>.Success(value)
// Controller → if (!result.IsSuccess) return BadRequest(result.Error); return Ok();
// Unexpected → _logger.LogError(ex, "{Handler} exception.", nameof(MyHandler)); throw;
```

3. DI via `*Module` + Scrutor suffix scanning only. Never `services.AddScoped<T>()` in `Program.cs`.

4. `AsNoTracking()` on all read queries. No N+1 patterns.

5. `EnableAuthentication: false` only in `appsettings.UnitTest.json`.

6. Match existing patterns in the same layer/feature before writing new code.

7. Follow the Mandatory Workflow Rules below — one step per reply, RED before GREEN, stop at every HUMAN GATE.
</critical_rules>

---

## Scope & Boundaries

<scope>

- **Changes under `src/` only.** One issue per change.
- **Features require `_plans/<FeatureName>.md` + user approval first.**
- **Non-goals**: no new architectures/libraries · no deployment/CI changes · no auth flow changes · no ORM migrations (SQL in `Database/`) · no build-system changes.
- **Risk areas** (require human review): Auth/OIDC · PII · shared contracts · domain/application layers · DB schema · controller routes/DTOs · blob/file storage.
- **Analyzer rules** in `Directory.Build.props` are **fixed** — do not modify.
</scope>

---

## Project Structure

```
src/
├── Host/Web/            # Blazor Server (WFE)
├── Host/BFF/            # Web API + Hangfire
├── Host/Api/            # Secondary Web API
├── Host/Worker/         # Background worker
├── Presentation/        # Razor Class Library (pages, ViewModels, Services)
├── Core/Application/    # Commands, Queries, Handlers (CQRS-lite)
├── Core/Domain/         # Entities, value objects (zero deps)
├── Core/Infrastructure/ # Messaging, external services
├── Core/Persistence/    # EF Core DbContext, repositories
├── Contracts/           # Shared DTOs
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

> **Note**: The examples above use `Product` as a placeholder. Replace with your project's **reference feature** entity — the feature that best represents your naming and structural patterns across all layers.

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

## Development Process

### Planning Gate

> **Before any tool call**: does this change need a plan? If yes → create `_plans/<FeatureName>.md` (use `@planner` or follow the template in `planner.agent.md`) → **STOP and WAIT for approval**.

- **Plan required**: ≥ 3 files · new feature/vertical slice · Risk Area changes.
- **Skip plan**: ≤ 2 file bugfixes, config corrections, simple refactors.

### Implementation
> After plan approval → use the **`/build-feature` skill** for step-by-step implementation with real code templates.

### Mandatory Workflow Rules

1. Show plan **before** coding.
2. **Steps must be verifiable and testable** — every step must produce observable behavior (a test passes, an API responds, a UI renders) and at least one automated test. Merge code-review-only steps into an adjacent step.
3. **ONE step per reply** — never batch multiple steps.
4. **RED before GREEN** — write a failing test, confirm it fails, then write production code.
5. Every bugfix gets a **regression test first**.
6. **Code analysis before gate** — after REFACTOR, fix all violations, then run the full test suite.
7. **🛑 STOP at HUMAN GATE** — do not proceed until user confirms.
8. **Mark done** — after approval, change `[ ]` → `[x]` in `_plans/<FeatureName>.md`. First unchecked `[ ]` = next step.

### Test Conventions

- AAA (Arrange/Act/Assert), MSTest v4, `[TestClass]` / `[TestMethod]`.
- Mock at boundary (interfaces only, Moq). No real DB/HTTP in unit tests.
- BFF integration (`Test.Bff`): Reqnroll (SpecFlow) `.feature` files + step definitions + `CustomWebApplicationFactory<Program, AppDbContext>` + SQLite in-memory + `TestAuthenticationHandler`.
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

- WFE → OIDC; BFF/Api → JWT bearer.
- WFE → BFF with client-credentials.
- `UserContextHeaderHandler` sends user identity headers; BFF reads via `HttpUserContextAccessor`.

### Logging

`ILogger<T>` + structured placeholders only. Shared configuration library owns Serilog — **no logging config in app code**.

### NServiceBus

- Events: `Contracts/<Feature>/Events/` (record). Messages: `Contracts/<Feature>/Messages/` (class).
- Handlers: BFF → `Host/BFF/NServiceBusHandlers/`, Worker → `Host/Worker/Handlers/`. All `IHandleMessages<T>`.
- **Never** call `IMessageSession` from Application — use `IMessagingService`.
- `[RequiresTransactionalSession]` for transactional consistency.
- Known error → `LogWarning` + return. Unknown → throw (triggers retry).

### Refit & HTTP Clients

- WFE → BFF: Refit in `Presentation/Shared/ServiceClients/Bff/Clients/`. Requires `apiVersion` param + `CancellationToken`.
- BFF → Api: `IApiClient` via HTTP service client registration.
- Feature Services: `Presentation/<Feature>/Services/` — wrap Refit, `catch ApiException → ConvertApiExceptionToResult<T>()`.

### Blazor ViewModel Lifecycle

`IsBusy` guard + `InitializeAsync(IErrorComponent)`. Page injects `IViewModel`, calls in `OnInitializedAsync`.
Error → `try/catch → errorComponent.ProcessError(ex)`. Set `IsBusy = true` before async, `false` in `finally`.

### Hangfire

`IBackgroundRecurringTask` in `Host/Worker/BackgroundRecurringTask/`. BFF hosts dashboard; Worker runs jobs. Shared SQL storage.

---

## New Feature Workflow

Before starting, identify a **reference feature** in the codebase — the existing feature whose patterns you will follow. If no reference feature has been specified by the user or in the spec, **ask the user which existing feature to use**. Study it across all layers first. Each step = one Red-Green-Refactor cycle with **🛑 HUMAN GATE**.

| Step | Layer | Test | Production |
|------|-------|------|------------|
| 1 | `Core/Domain/` | `Test/Unit/Domain/` — construction | Entity : `IEntity` |
| 2 | `Core/Application` + `Persistence` | `Test/Unit/Application/` — handler (mock repo) | `IMyRepo` + `MyRepo : BaseRepository<T>` |
| 3 | `Core/Application/Functionalities/` | `Test/Unit/Application/` — command/query | Handler + Query |
| 4 | `Contracts/` | DTO shape/mapping test | `record` DTOs |
| 5 | `Host/Bff/Controllers/` | `Test/Bff/Features/` — Reqnroll `.feature` + step def | Controller (`Result<T>` → HTTP status) |
| 6 | `Database/Tables/` | _(covered by step 5)_ | SQL `CREATE TABLE` — **user deploys via Schema Compare** |
| 7 | `Presentation/` | Manual UI test | Pages, ViewModels, Services, Refit clients |

---

## AppSettings Layering

`appsettings.json` → `*.Development.json` → `*.UnitTest.json` → `*.UnitTest.Development.json` (gitignored) → env vars.
Required block: `Application` with `BusinessSystemName`, `SubSystemName`, `SubSystemType`, `Environment`.
**No secrets in committed files. No `appsettings.TST/PRD.json`.**

---

<anti_patterns>
## Anti-Patterns

| Don’t | Do instead |
|-------|------------|
| `services.AddScoped<T>` in Program.cs | `*Module` + suffix auto-registration |
| Throw for validation errors | Return `Result<T>` |
| EF attributes on domain entities | `IEntityTypeConfiguration<T>` in Persistence |
| `IMessageSession` in Application | `IMessagingService` abstraction |
| Manual logging/security setup | Shared configuration library handles it |
| Service without interface | Scrutor needs `I<Name>` to register |
| Skip `CancellationToken` | Propagate on every async call |
| Multiple plan steps per reply | One step → stop at HUMAN GATE |
| Production code before test | RED first → confirm fail → then GREEN |
</anti_patterns>

---

<investigate_before_answering>
Before answering questions about the codebase, read the relevant files first. Never make claims about code without investigating. Give grounded, hallucination-free answers.
</investigate_before_answering>

---

## Commands

```powershell
dotnet build src/MyApp.sln                                                          # Build
dotnet restore src/MyApp.sln                                                       # Restore (NuGet sources configured at user level)
dotnet test                       # Unit tests
dotnet test --filter "<Class>"   # Filtered
dotnet format src/MyApp.sln --verify-no-changes                                    # Format check
```

## Conventions

- No `.sqlproj` changes — DB changes via SQL files only.
- `protected Program() {}` in every host.
- Commit format: `<type>(<scope>): <desc>`.
- Build + test + format before push.
