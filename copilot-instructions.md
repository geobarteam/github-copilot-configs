# Copilot Instructions — <MySolutionName>

<!-- Optimized for Claude Sonnet 4.6: XML tags for critical sections, markdown headers
     for structure, concise sections, one example per pattern. Anthropic best practices
     applied: role assignment, clear-not-aggressive language, investigate before answering. -->

You are the coding assistant for the <MySolutionName> project. Read and investigate relevant files before answering questions or making changes — never speculate about code you have not opened.

<context>
Blazor WebAssembly + Client (BFF) architecture. The **Client** (ASP.NET Core) serves WASM static files, exposes API endpoints, and hosts NServiceBus. The **WASM client** runs in the browser and calls Client APIs via session cookie (never holds tokens).
Onion/Screaming Architecture, CQRS-lite. 
.NET 10.
</context>

---

<critical_rules>
## Critical Rules

1. After every code change, run all three:
```
dotnet build src/<MySolutionName>.sln
dotnet test --project src/Test/Unit/
dotnet format src/<MySolutionName>.sln --verify-no-changes
```

2. Never throw for business errors — use `Result<T>` from LanguageExt.Core. Only throw for truly unexpected errors that should trigger retries or human investigation.

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
- **Non-goals**: no new architectures/libraries · no deployment/CI changes · no auth flow changes · no build-system changes.
- **Risk areas** (require human review): Auth/OIDC · PII · shared contracts · domain/application layers · DB schema · controller routes/DTOs · blob/file storage.
- **Analyzer rules** must be fixed in the same step as production code changes — do not defer to a later step. If a change causes new violations, fix them before proceeding.
</scope>

---

## Project Structure

```
src/
├── Host/Client/         # Client – ASP.NET Core host: serves WASM static files + BFF API.
├── Host/Wasm/           # Wasm – Blazor WebAssembly browser entry point
├── Presentation/        # Razor Class Library (pages, ViewModels, Services)
├── Infrastructure/      # WASM-side: CookieHandler, BffAuthenticationStateProvider, AntiforgeryTokenStore
├── Core/Application/    # Commands, Queries, Handlers (CQRS-lite)
├── Core/Domain/         # Entities, value objects (zero deps)
├── Core/Persistence/    # EF Core DbContext, repositories
├── Contracts/           # Shared DTOs + NServiceBus contracts
└── Test/{Unit,Common,UI,Integration}/
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

> **Note**: The examples above use `Product` as a placeholder. Replace with your project's **reference feature** entity — the feature that best represents your naming and structural patterns across all layers.

`_camelCase` fields (SA1309 disabled) · `kebab-case` routes · features in `Functionalities/<Feature>/`.

---

## DI Registration (Scrutor Suffix Scanning)

| Suffix | Module | Lifetime |
|--------|--------|----------|
| `Query`, `Handler`, `ICommandHandler<,>` | `ApplicationModule` | Scoped |
| `Repository` | `PersistenceModule` | Scoped |
| `ViewModel`, `Service` | `PresentationModule` | Transient |

Every service needs an `I<Name>` interface for Scrutor to register it.
`ClientModule` composes `ApplicationModule` + `PersistenceModule` for the Client host.

---

## Dependency Matrix

| Project | May reference |
|---------|--------------|
| `Core.Domain` | _(nothing)_ |
| `Core.Application` | `Core.Domain`, `Contracts` |
| `Core.Persistence` | `Core.Application`, `Core.Domain` |
| `Contracts` | _(nothing)_ |
| `Infrastructure` | `Contracts` (WASM-side: CookieHandler, AuthState, Antiforgery) |
| `Presentation` | `Contracts`, `Infrastructure` |
| `Host.Client` | `Core.*`, `Contracts`, `Presentation` |
| `Host.Wasm` | `Presentation`, `Contracts`, `Infrastructure` |
| `Test.Unit` | `Presentation`, `Contracts` |
| `Test.UI` | `Presentation` (bUnit component tests) |
| `Test.Common` | _(shared test utilities, no prod refs)_ |

**Forbidden**: Domain → anything · Application → Infra/Persistence/Host · Contracts → Core/Host · Infrastructure → Core/Host · Presentation → Host/Persistence · circular refs.
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
- Unit tests (`Test/Unit/`): ViewModels, Services, Application handlers. Mirror source structure.
- UI component (`Test.UI`): bUnit `BunitTest` base class, MSTest, renders Blazor components in isolation.
- Shared fixtures (`Test/Common/`): `TestData.cs`, `AuthenticationStateProviderMockExtensions`.
- Integration tests (`Test/Integration/`): `WebApplicationFactory` + SQLite in-memory + fake auth / real authz. See `tests.instructions.md` for full conventions, folder structure, and patterns.

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

- Client acts as OIDC confidential client. WASM client **never receives tokens**.
- Browser holds only an `HttpOnly` session cookie. All WASM → Client calls use `credentials: include` (via `CookieHandler`).
- State-changing calls (POST/PUT/DELETE/PATCH) include the `X-XSRF-TOKEN` anti-forgery header (via `AntiforgeryTokenStore`).
- `BffAuthenticationStateProvider` calls `GET /api/user` to retrieve claims; anonymous if not authenticated.

### Logging

`ILogger<T>` + structured placeholders only. 


### Refit & HTTP Clients

- WASM → Client: Refit interfaces in `Presentation/Shared/ServiceClients/Bff/`. Every method needs `CancellationToken`.
- Registered via `BffServiceClients.AddBffServiceClients()` using `AddRefitClientWithCookies<T>()`.
- `CookieHandler` ensures `credentials: include` + XSRF on every call.
- Feature Services: `Presentation/<Feature>/Services/` — wrap Refit client, map DTO→Model.

### Blazor ViewModel Lifecycle

`IsBusy` guard + `InitializeAsync(IErrorComponent)`. Page injects `IViewModel`, calls in `OnInitializedAsync`.
Error → `try/catch → errorComponent.ProcessError(ex)`. Set `IsBusy = true` before async, `false` in `finally`.

---

## New Feature Workflow — Vertical Slices

Before starting, identify a **reference feature** in the codebase — the existing feature whose patterns you will follow. If no reference feature has been specified by the user or in the spec, **ask the user which existing feature to use**. Study it across all layers first. Each step = one Red-Green-Refactor cycle with **🛑 HUMAN GATE**.

**Plan by behavior, not by layer.** Each step delivers a thin, end-to-end slice of user-visible functionality that may cross multiple layers (Domain, Application, Persistence, Contracts, Controller, Database, Presentation). Name steps by what the user or system can do after the step — not by which layer is created.

### Layer reference (what exists — not the step order)

| Layer | Purpose |
|-------|--------|
| `Core/Domain/` | Entities : `IEntity`, value objects, plain C# |
| `Core/Application/` + `Persistence/` | Handlers, Queries, Repos (`BaseRepository<T>`) |
| `Contracts/` | `record` DTOs |
| `Host/Client/Controllers/` | REST API (`Result<T>` → HTTP status) |
| `Presentation/` | Pages, ViewModels, Services, Refit clients |

### Example — vertical slice steps

1. **Display the list** — read-only flow across all layers from DB → API → UI.
2. **Retrieve detail and edit fields** — detail view, edit form, all layers.
3. **Validate on Save** — business rules in handler, error display in UI.
4. **Persist changes** — save to DB, confirmation feedback.

---

<anti_patterns>
## Anti-Patterns

| Don't | Do instead |
|-------|------------|
| `services.AddScoped<T>` in Program.cs | `*Module` + suffix auto-registration |
| Throw for validation errors | Return `Result<T>` |
| EF attributes on domain entities | `IEntityTypeConfiguration<T>` in Persistence |
| `IMessageSession` in Application | `IMessagingService` abstraction |
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
dotnet build src/MySolutionName.sln                # Build
dotnet restore src/MySolutionName.sln              # Restore (NuGet sources configured at user level)
dotnet test --project src/Test/Unit/                 # Unit tests
dotnet test --project src/Test/Integration/          # Integration tests
dotnet test --project src/Test/UI/                   # bUnit component tests
dotnet test --project src/Test/Unit/ --filter "<Class>" # Filtered
dotnet format src/MySolutionName.sln --verify-no-changes  # Format check
```

## Conventions

- `protected Program() {}` in every host.
- Commit format: `<type>(<scope>): <desc>`.
- Build + test + format before push.
