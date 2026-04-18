---
name: build-feature
description: "Use when implementing a new feature, vertical slice, or multi-layer change in {{SolutionName}} after a _plans/<FeatureName>.md (repo root) is approved. Provides step-by-step Red-Green-Refactor implementation procedure with real code templates for each layer: Domain entity, Application command/query/handler, Persistence repository, Contracts DTOs, Controller, DbUp migration, Presentation ViewModel/ServiceClient/Refit. Use for: building new features, adding CRUD operations, implementing vertical slices, following the 7-step layer workflow."
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

> This workflow mirrors the canonical Red-Green-Refactor-Proof Loop defined in `AGENTS.md`. See `AGENTS.md` for the full rules and planning gate requirements.

> **Note**: `dotnet test` is not supported — `Microsoft.Testing.Platform` + .NET 10 SDK requires running the test `.exe` directly. If packages are missing, run `dotnet restore src/{{SolutionName}}.sln --interactive` first.

**Never batch steps. Never skip RED. Never skip CODE ANALYSIS. Never proceed past 🛑 without user confirmation.**

---

## Layer Templates (from the reference feature)

### Step 1 — Domain Entity

Location: `src/Core/Domain/Functionalities/<Feature>/<Entity>.cs`

Plain C# class implementing `IEntity` (`int Id { get; set; }`). No EF attributes, no dependencies.

> Full pattern: see `domain-entity.instructions.md`.

Test: construction + property assignment.

---

### Step 2 — Repository Interface + Implementation

**Interface**: `src/Core/Application/Functionalities/<Feature>/I<Entity>Repository.cs` — extends `IRepository<<Entity>>` with feature-specific methods.

**Implementation**: `src/Core/Persistence/Repositories/<Entity>Repository.cs` — extends `BaseRepository<<Entity>>`, uses `AsNoTracking()` for reads.

**EF Configuration**: `src/Core/Persistence/EntityTypeConfigurations/<Entity>Configuration.cs` — `IEntityTypeConfiguration<<Entity>>` with fluent API (no EF attributes on entity).

DI: auto-registered by `PersistenceModule` (suffix `Repository` → Scoped).

> Full patterns: see `persistence-layer.instructions.md`.

---

### Step 3 — Command + Handler

**Command**: `src/Core/Application/Functionalities/<Feature>/Commands/<Action>/Add<Entity>Command.cs` — `record` with `Validate()` method returning `(bool IsValid, List<string> Errors)`.

**Handler**: same folder, `Add<Entity>CommandHandler.cs` — implements `ICommandHandler<Add<Entity>Command, Result<Unit>>`. Validates first, then persists. Returns `Result<Unit>`.

**Query**: `src/Core/Application/Functionalities/<Feature>/Queries/<Action>/` — interface `IGet<Entity>ListQuery` + implementation delegating to repository.

DI: auto-registered by `ApplicationModule` (suffix `Handler` → Scoped, suffix `Query` → Scoped).

> Full patterns: see `application-layer.instructions.md`.

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

### Step 5 — Controller

Location: `src/Host/Client/Controllers/<Feature>Controller.cs`

`[ApiController]` with `[Route("api/<feature-kebab>")]`. Constructor-inject queries/commands, `IAuditLogger`, `ICorrelationContextAccessor`, `IUserContextAccessor`, `ILogger`.

- GET: execute query → map to DTO → audit log → `Ok(dtos)`. Return `NotFound()` for null.
- POST: execute command → `Result<T>` → `BadRequest(error)` or `Ok()`. Audit via `LogActionAsync`.
- Every action: `CancellationToken` as last param, `try/catch` with structured logging.

> Full patterns (GET/POST/PUT/DELETE, audit logging): see `api-controller.instructions.md`.

Test: `Test/Integration/Endpoints/` — HTTP tests with `CustomWebApplicationFactory`.

---

### Step 6 — Database Migration (DbUp)

Create a new DbUp migration script using the `/add-dbup` skill. The script goes in `src/Database/Scripts/` with sequential numbering:

```
src/Database/Scripts/<NNNN>_Create<Entity>Table.sql
```

> Full patterns (naming, templates, idempotency, script rules): see `/add-dbup` skill.

Ensure the EF Core `IEntityTypeConfiguration<T>` from Step 2 matches the SQL schema exactly (table name, column types, nullability, indexes).

---

### Step 7 — Presentation Layer

**Refit Client**: `src/Presentation/Shared/ServiceClients/Bff/Clients/I<Feature>Client.cs` — Refit interface with `[Get]`/`[Post]` attributes, `apiVersion` + `CancellationToken` on every method. Register new interfaces in `BffServiceClients.AddBffServiceClients()`.

**Feature ServiceClient**: `src/Presentation/<Feature>/ServiceClients/<Feature>ServiceClient.cs` — wraps `I<Feature>Client`, maps DTO → Model, catches `ApiException` → `ConvertApiExceptionToResult<T>()`.

DI: auto-registered by `PresentationModule` (suffix `ServiceClient` → Transient).

**ViewModel**: `src/Presentation/<Feature>/ViewModels/<Feature>ViewModel.cs` — implements `I<Feature>ViewModel : IViewModel`. `IsBusy` guard, `InitializeAsync(IErrorComponent)`, `try/catch → errorComponent.ProcessError(ex)`.

DI: auto-registered by `PresentationModule` (suffix `ViewModel` → Transient).

> Full patterns (ViewModel lifecycle, ServiceClient mapping, Refit conventions, Razor pages, dialogs): see `blazor-presentation.instructions.md` and `/add-blazor-page` skill.

---

## Key Reminders

- **ICommandHandler<TCommand, TResult>** is the Application-layer command interface. Handlers are auto-scanned.
- **Result<T>** — constructor `new Result<T>(value)` for success, `new Result<T>("error msg")` for failure. Check `result.IsSuccess`.
- **Unit** — use `Unit.Default()` for void-equivalent returns.
- **ApiException** — catch in ServiceClient, convert via `ex.ConvertApiExceptionToResult<T>()`.
- **BaseRepository<T>** — provides `AddAsync`, `GetByIdAsync`, `ListAllAsync`, `Update`, `Delete`, `SaveChangesAsync`.
- **DbSet registration** — add `DbSet<<Entity>>` to `{{DbContextName}}`.
- **Copyright header** — all files need: `// <copyright file="<File>.cs" company="{{CompanyName}}"> ... </copyright>`
- **CancellationToken** — propagate on every async call in controllers. Handlers may omit if not needed by repository.
