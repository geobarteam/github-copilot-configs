---
description: "Scaffold a complete vertical slice feature: domain entity → repo → query/command → controller → DTO → Refit client → service → ViewModel → Razor page + tests. Uses the canonical 7-step recipe from copilot-instructions.md."
agent: "agent"
tools: ["read", "edit", "search", "execute", "todo", "agent"]
---
# New Vertical Slice Feature

You are building a new feature end-to-end in the {{SolutionName}}Wasm project. Follow the **Canonical Recipe (§9)** from copilot-instructions.md — one Red → Green → Refactor cycle per step.

## Input

The user provides: **Feature name** (e.g. `Specialty`, `Insurance`, `Location`).
Ask for any additional details if not provided: entity properties, relationships, business rules.

## Before You Start

1. **Identify the reference feature** — ask the user which existing feature in the codebase to use as the pattern. If the user doesn't specify one, list the features visible under `src/Core/Domain/Functionalities/`, `src/Client/`, or `src/Host/BFF/Controllers/` and let them pick. Then read its implementation across all layers:
   - `Core/Domain/Shared/Entities/<ReferenceEntity>.cs`
   - `Core/Application/Functionalities/<ReferenceFeature>/` (all files)
   - `Core/Persistence/Repositories/<ReferenceEntity>Repository.cs`
   - `Contracts/<ReferenceFeature>/<ReferenceEntity>Dto.cs`
   - `Host/BFF/Controllers/<ReferenceEntity>Controller.cs`
   - `Client/Shared/ServiceClients/Bff/I<ReferenceEntity>ServiceClient.cs`
   - `Client/<ReferenceFeature>/` (all files: Services, ViewModels, Models, Pages)
   - `Test/Unit/Client/<ReferenceFeature>/` (all tests)

2. **Create `_plans/<FeatureName>.md`** at the repo root (e.g. `_plans/MyPrescriptions.md`) using this template:

```markdown
# Plan: Add <Feature> Feature (Vertical Slice)

## Step 1 — Domain Entity
**Scope**: `Core/Domain/Shared/Entities/<Feature>.cs`
**RED**: `Test/Unit/Domain/<Feature>Tests.cs` — construction + property tests
**GREEN**: Plain C# class `: IEntity`. Properties: Id + <user-specified properties>.
**REFACTOR**: none expected.

## Step 2 — Repository + Query + Command
**Scope**: `Core/Application/Shared/Interfaces/Persistence/Repositories/I<Feature>Repository.cs`,
  `Core/Persistence/Repositories/<Feature>Repository.cs`,
  `Core/Application/Functionalities/<Feature>/Queries/Get<Feature>s/IGet<Feature>sQuery.cs`,
  `Core/Application/Functionalities/<Feature>/Queries/Get<Feature>s/Get<Feature>sQuery.cs`,
  `Core/Application/Functionalities/<Feature>/Handlers/Add<Feature>/Add<Feature>Command.cs`,
  `Core/Application/Functionalities/<Feature>/Handlers/Add<Feature>/Add<Feature>CommandHandler.cs`
**RED**: `Test/Unit/Application/<Feature>/Get<Feature>sQueryTests.cs` — mock repo, assert list returned
  `Test/Unit/Application/<Feature>/Add<Feature>CommandHandlerTests.cs` — success + duplicate-name error
**GREEN**: Implement repo (`: BaseRepository<T>`), query, and command handler.
**REFACTOR**: none expected.

## Step 3 — Contracts (DTOs)
**Scope**: `Contracts/<Feature>s/<Feature>Dto.cs`, `Contracts/<Feature>s/Add<Feature>Dto.cs`
**RED**: construction test in existing handler test file (assert DTO records can be created).
**GREEN**: `record <Feature>Dto(int Id, string Name)`, `record Add<Feature>Dto { [Required] string Name }`.
**REFACTOR**: none expected.

## Step 4 — Bff Controller
**Scope**: `Host/BFF/Controllers/<Feature>Controller.cs`
**RED**: (tested indirectly at service/UI level; write a DTO mapping assertion in handler test if needed)
**GREEN**: `[ApiController][Authorize][Route("api/[controller]")]`. GET → query, POST → command handler.
  Use `[ApiAntiforgery]` on POST/PUT/DELETE/PATCH.
**REFACTOR**: none expected.

## Step 5 — Refit Client + Feature Service + Model
**Scope**: `Client/Shared/ServiceClients/Bff/I<Feature>ServiceClient.cs`,
  `Client/<Feature>s/Services/I<Feature>sService.cs`,
  `Client/<Feature>s/Services/<Feature>sService.cs`,
  `Client/<Feature>s/Models/<Feature>Model.cs`
**RED**: `Test/Unit/Client/<Feature>s/Services/<Feature>sServiceTest.cs` — mock Refit client, assert DTO→Model mapping
**GREEN**: Refit interface (GET/POST), service maps DTO→Model, register client in `BffServiceClients.AddBffServiceClients()`.
**REFACTOR**: none expected.

## Step 6 — ViewModel
**Scope**: `Client/<Feature>s/ViewModels/I<Feature>sViewModel.cs`,
  `Client/<Feature>s/ViewModels/<Feature>sViewModel.cs`
**RED**: `Test/Unit/Client/<Feature>s/ViewModels/<Feature>sViewModelTest.cs` — InitializeAsync, Add<Feature>
**GREEN**: Inject service + ISnackbar + AuthenticationStateProvider. IsBusy + InitializeAsync + Add<Feature>.
**REFACTOR**: none expected.

## Step 7 — Razor Page + bUnit Test
**Scope**: `Client/<Feature>s/Pages/<Feature>sPage.razor`,
  `Client/<Feature>s/Pages/<Feature>sPage.razor.cs`
**RED**: `Test/UI/<Feature>s/Pages/<Feature>sPageTests.cs` — render with mock ViewModel, assert content appears
**GREEN**: MudDataGrid bound to ViewModel. MudForm for add. @attribute [Authorize].
**REFACTOR**: Add localization resource file if needed.
```

3. **Show `_plans/<FeatureName>.md` to the user and wait for approval** before starting Step 1.

## Execution Rules

- **One step per reply.** After each step: run code analysis (fix all violations), run full test suite, present HUMAN GATE, STOP.
- **RED first.** Write the test, run it, confirm it fails. Only then write production code.
- Every file follows the exact conventions from the reference feature (namespace, folder, naming).
- Follow the `refit-client.instructions.md` for steps 5–7.
- Run **Code Analysis** agent after each step.
- **Mark done.** After the user approves a step, update `_plans/<FeatureName>.md` — change `[ ]` → `[x]` on that step's HUMAN GATE checkboxes. When resuming work, find the first unchecked `[ ]` to know where to continue.
