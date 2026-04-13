# Spec-Driven Development with AI: A Practical Guide to Staying in Control

*How to move from vibe coding to a disciplined, verifiable development process when working with AI coding assistants.*

---

## The Problem: Vibe Coding

Most developers using AI coding assistants today fall into a pattern we call **vibe coding**. The conversation goes something like this:

> "Add a subscriptions feature to the app."

The AI starts generating code. Entity classes appear. A repository materialises. A controller takes shape. Pages get scaffolded. Twenty minutes later, there are fifteen new files across six projects. The developer glances at the output, sees that it looks roughly right, and commits.

What could go wrong?

Everything. The entity doesn't match the database schema. The DTO has properties the UI never uses and is missing ones it needs. The controller returns a shape the page doesn't expect. The repository uses a query pattern that differs from every other repository in the codebase. There are no tests. The naming conventions are inconsistent. And the developer — the person who should be accountable for the design — can no longer explain why half of these decisions were made.

**Vibe coding is not a productivity gain. It is a control loss.** The AI produces plausible-looking code at high speed, and the developer's role degrades from engineer to approver of things they didn't design and don't fully understand.

The alternative is **spec-driven development**: a structured process where the developer defines intent, the AI plans before coding, implementation happens in small verified steps, and the human stays in control at every stage.

---

## The Three Pillars

Spec-driven development with AI rests on three pillars:

1. **Specifications** — Define what to build before building it
2. **Plans** — Decompose work into vertical slices before writing code
3. **Red-Green-Refactor with human gates** — Implement one verified step at a time

Each pillar serves a specific purpose. Specifications prevent scope creep. Plans prevent architectural chaos. Red-Green-Refactor with gates prevents compounding errors.

Let's look at each in detail.

---

## Pillar 1: Specifications — The Contract Between Developer and AI

### Why Specs Matter

Without a specification, the AI infers intent from a vague prompt. "Add subscriptions" could mean a hundred different things. Which entity properties? What validation rules? Which API endpoints? What error messages? Who can access it? The AI will answer all these questions for you — and most of the answers will be wrong, because it's guessing.

A specification makes intent explicit. It is a structured document that captures:

- **Who** benefits and **what** they can do
- **Acceptance criteria** — concrete, testable conditions for "done"
- **Data model** — entities, properties, types, constraints
- **API surface** — endpoints, request/response shapes, error codes
- **Business rules** — validation, constraints, and their error messages
- **Edge cases** — what happens when things go wrong
- **Non-goals** — what is explicitly out of scope

### A Concrete Spec Example

Here's what a real spec looks like for a "Patient Subscriptions" feature:

````markdown
# Spec: Patient Subscriptions

## User Story

**As a** patient,
**I want to** view my active medication subscriptions,
**So that** I can track which medications are being dispensed on my behalf.

---

## Acceptance Criteria

| #    | Given                              | When                              | Then                                              |
|------|------------------------------------|-----------------------------------|---------------------------------------------------|
| AC-1 | Patient is authenticated           | They navigate to /subscriptions   | A list of their active subscriptions is displayed  |
| AC-2 | Patient has no subscriptions       | They navigate to /subscriptions   | An empty-state message is shown                    |
| AC-3 | Subscription has expired           | They view the list                | Expired subscriptions are not shown                |
| AC-4 | Doctor creates a subscription      | They submit the form              | The subscription appears in the patient's list     |
| AC-5 | Doctor submits with missing fields | They click Save                   | Validation errors are shown, nothing is persisted  |

---

## Data Model

### Entity: `Subscription`

| Property    | Type     | Required | Constraints                    |
|-------------|----------|----------|--------------------------------|
| Id          | int      | PK       | Auto-generated                 |
| PatientId   | int      | Yes      | FK → Patient                   |
| MedicationName | string | Yes     | Max 200 chars                  |
| StartDate   | DateOnly | Yes      | Must be today or future        |
| EndDate     | DateOnly | No       | Must be after StartDate        |
| CreatedBy   | string   | Yes      | Doctor's national register ID  |

---

## API Endpoints

### `GET /api/subscriptions?patientId={id}`
**Purpose**: Return active subscriptions for a patient.
**Auth**: Bearer (patient or doctor role)
**Response** (200):
```json
[
  {
    "id": 1,
    "medicationName": "Metformin 500mg",
    "startDate": "2026-04-01",
    "endDate": "2026-10-01"
  }
]
```
**Error responses**: 401 → Unauthorized, 403 → Wrong patient

### `POST /api/subscriptions`
**Purpose**: Create a new subscription.
**Auth**: Bearer (doctor role only)
**Request body**:
```json
{
  "patientId": 42,
  "medicationName": "Metformin 500mg",
  "startDate": "2026-04-01",
  "endDate": "2026-10-01"
}
```
**Response** (201): Created subscription
**Error responses**: 400 → Validation errors, 401, 403

---

## Business Rules

| #    | Rule                                        | Error message                               |
|------|---------------------------------------------|---------------------------------------------|
| BR-1 | MedicationName is required                  | "Medication name is required"               |
| BR-2 | StartDate must be today or in the future    | "Start date cannot be in the past"          |
| BR-3 | EndDate, if provided, must be after StartDate | "End date must be after start date"       |
| BR-4 | Only doctors can create subscriptions       | HTTP 403                                    |

---

## Out of Scope

- Editing or deleting existing subscriptions
- Medication lookup / auto-complete
- Notification when a subscription is created
- Prescription history
````

### What the Spec Gives You

Every decision is explicit. The AI doesn't need to guess whether `EndDate` is required (it's not). It doesn't need to invent validation rules (they're listed). It doesn't need to decide who can create subscriptions (doctors only). And when the developer reviews the AI's output, they can point to the spec and say: "BR-3 says EndDate must be after StartDate, but your validation checks `>=` instead of `>`."

The spec also explicitly says what's **out of scope**. Without this, the AI will happily add edit and delete endpoints, a medication search dropdown, and email notifications — none of which were requested.

### When to Write Specs

Not every change needs a spec. Here's a simple rule:

- **Spec required**: New feature, new user-facing behaviour, new API surface
- **Spec optional**: Multi-file refactor, infrastructure change
- **No spec needed**: Bugfix, config change, 1-2 file change

The spec lives at `_specs/<FeatureName>.md` in the repository root. It is committed to Git alongside the code, so any developer (or AI agent) can read it later to understand _why_ something was built the way it was.

---

## Pillar 2: Plans — Decomposing Work into Vertical Slices

### From Spec to Plan

The spec defines **what** to build. The plan defines **how** — in what order, touching which files, verified by which tests. The plan is derived from the spec after the AI has explored the codebase to understand existing patterns.

A plan is a markdown file at `_plans/<FeatureName>.md` with concrete implementation steps. Each step is a single Red-Green-Refactor cycle that delivers one vertical behaviour slice.

### The Planning Gate

Before any code is written, the plan must be **approved by the developer**. This is the planning gate — a hard stop between thinking and doing.

The AI agent asks itself: _Does this change need a plan?_

| Situation | Plan required? |
|-----------|---------------|
| New feature or vertical slice | **Yes** |
| Change touching 3+ files | **Yes** |
| Risk area (auth, PII, DB schema, shared contracts) | **Yes** |
| 1–2 file bugfix | No — but still test-first |
| Config correction, simple refactor | No |

If a plan is required, the AI produces it, presents it, and **stops**. No code is written until the developer says "approved."

### Vertical Slices, Not Horizontal Layers

This is the most important planning principle and the one most teams get wrong.

**Horizontal slicing** (the anti-pattern) plans by layer:

```
Step 1 — Create the Subscription entity
Step 2 — Add SubscriptionRepository
Step 3 — Create SubscriptionCreateHandler
Step 4 — Add GET /api/subscriptions endpoint
Step 5 — Add POST /api/subscriptions endpoint
Step 6 — Build the subscriptions list page
Step 7 — Build the create subscription form
```

This feels orderly. But the problems are severe:

- **Late integration.** The entity is built in Step 1, but the page that displays it isn't built until Step 6. If the entity shape doesn't match what the page needs, you don't find out until six steps in.
- **No early feedback.** The user can't see or validate anything until Step 6. Five steps of backend code are built before anyone checks whether the UI makes sense.
- **All-or-nothing.**  If Step 4 reveals that the query shape needs to change, Steps 1–3 might need rework. Changes cascade backward through completed steps.

**Vertical slicing** plans by user-visible behaviour:

```
Step 1 — Display subscription list (stubbed)
Step 2 — Wire subscription list to real API
Step 3 — Create subscription form (stubbed)
Step 4 — Wire subscription create to real API
```

Each vertical slice follows a two-phase pattern:

1. **Stub** — Build the UI with a mocked/stubbed backend. The user validates the interface, the layout, the data shape, the interaction flow. Tests verify the ViewModel works with the mock.
2. **Wire** — Replace the stub with real code from top to bottom: controller → application handler → domain → persistence → database. Integration tests verify the full stack. Remove the stub.

### Why This Order?

**Fail fast.** By integrating all layers in Step 2, you discover immediately whether the entity shape matches the DTO, whether the query returns what the page expects, whether the controller route matches the Refit client. Problems surface on the second step — not the seventh.

**UI feedback early.** The user sees and validates the interface in Step 1, before any backend work begins. If the columns are wrong, if the form layout is confusing, if a field is missing — you find out when the fix is cheap (change a ViewModel), not after building six layers of backend code.

**Smaller blast radius.** When something needs to change, only the current stub-wire pair is affected. You don't have to trace the change through five independent horizontal steps.

### A Concrete Plan Example

Here's what the plan looks like for the Patient Subscriptions feature:

```markdown
# Plan: Patient Subscriptions — Patient views active medication subscriptions

## Overview
Patients can view their active medication subscriptions. Doctors can create new 
subscriptions. Reference feature: **Doctors** (existing CRUD feature to follow 
as the pattern across all layers).

---

## Step 1 — Display subscription list (stubbed)

**Scope** *(Presentation only — stub phase)*:
- `src/Presentation/Subscriptions/ViewModels/SubscriptionsViewModel.cs` *(create)*
- `src/Presentation/Subscriptions/Services/ISubscriptionService.cs` *(create)*
- `src/Presentation/Subscriptions/Services/SubscriptionServiceStub.cs` *(create)*
- `src/Presentation/Subscriptions/Pages/Subscriptions.razor` *(create)*
- `src/Contracts/Subscriptions/Api/SubscriptionDto.cs` *(create)*
- `src/Test/Unit/Presentation/Subscriptions/SubscriptionsViewModelTests.cs` *(create)*

**RED** *(write this test first)*:
- Test: `InitializeAsync_WithSubscriptions_LoadsList`
- Command: `dotnet test --project src/Test/Unit/ --filter "SubscriptionsViewModelTests"`

**GREEN**:
- SubscriptionDto record with Id, MedicationName, StartDate, EndDate
- ISubscriptionService with GetByPatientAsync(int patientId)
- SubscriptionServiceStub returning 3 fake subscriptions
- SubscriptionsViewModel: IsBusy guard, InitializeAsync loads via service
- Subscriptions.razor: MudTable bound to ViewModel.Items

**DB changes**: none

**🛑 HUMAN GATE**:
- [ ] Behavioral: ViewModel test passes — list loads with 3 stubbed items
- [ ] Review: Page layout matches Doctor list page pattern

---

## Step 2 — Wire subscription list to real API

**Scope** *(all backend layers — wire phase)*:
- `src/Core/Domain/Shared/Entities/Subscription.cs` *(create)*
- `src/Core/Persistence/EntityTypeConfigurations/SubscriptionConfiguration.cs` *(create)*
- `src/Core/Persistence/Repositories/SubscriptionRepository.cs` *(create)*
- `src/Core/Application/Functionalities/Subscriptions/Queries/GetSubscriptions/` *(create)*
- `src/Host/BFF/Controllers/SubscriptionController.cs` *(create — GET only)*
- `src/Database/Tables/Subscription.sql` *(create)*
- `src/Presentation/Subscriptions/Services/SubscriptionService.cs` *(create — real Refit)*
- `src/Presentation/Shared/ServiceClients/Bff/Clients/ISubscriptionClient.cs` *(create)*
- `src/Test/Unit/Application/Subscriptions/GetSubscriptionsQueryTests.cs` *(create)*
- `src/Test/Integration/Endpoints/Subscriptions/GetSubscriptionsTest.cs` *(create)*

**RED**:
- Unit: `Execute_WithActiveSubscriptions_ReturnsList` (mock repo)
- Integration: `GetSubscriptions_Authenticated_ReturnsOk` (WebApplicationFactory + SQLite)
- Integration: `GetSubscriptions_WrongPatient_ReturnsForbidden`
- Command: `dotnet test --project src/Test/Unit/ --filter "GetSubscriptionsQueryTests"`
- Command: `dotnet test --project src/Test/Integration/ --filter "GetSubscriptionsTest"`

**GREEN**:
- Subscription entity (Id, PatientId, MedicationName, StartDate, EndDate, CreatedBy)
- SubscriptionRepository with GetActiveByPatientAsync (filter EndDate > today)
- IGetSubscriptionsQuery + GetSubscriptionsQuery
- SubscriptionController GET /api/subscriptions?patientId={id}
- ISubscriptionClient (Refit), SubscriptionService (replaces stub)
- SubscriptionConfiguration (fluent API)
- Subscription.sql (CREATE TABLE)

**DB changes**: `src/Database/Tables/Subscription.sql`

**🛑 HUMAN GATE**:
- [ ] Behavioral: Integration tests pass; GET /api/subscriptions returns list
- [ ] Review: Entity, repo, query, controller follow Doctors pattern

---

## Step 3 — Create subscription form (stubbed)

**Scope** *(Presentation only — stub phase)*:
- `src/Presentation/Subscriptions/ViewModels/AddSubscriptionViewModel.cs` *(create)*
- `src/Presentation/Subscriptions/Pages/AddSubscription.razor` *(create)*
- `src/Contracts/Subscriptions/Api/AddSubscriptionDto.cs` *(create)*
- `src/Test/Unit/Presentation/Subscriptions/AddSubscriptionViewModelTests.cs` *(create)*

**RED**:
- Test: `Submit_EmptyMedicationName_ShowsValidationError`
- Test: `Submit_ValidData_CallsService`
- Command: `dotnet test --project src/Test/Unit/ --filter "AddSubscriptionViewModelTests"`

**GREEN**:
- AddSubscriptionDto record (PatientId, MedicationName, StartDate, EndDate)
- AddSubscriptionViewModel with field validation (BR-1 through BR-3)
- SubmitAsync method calls stubbed service
- AddSubscription.razor: MudForm with validated fields

**DB changes**: none

**🛑 HUMAN GATE**:
- [ ] Behavioral: ViewModel tests pass — validation errors display correctly
- [ ] Review: Form matches spec business rules BR-1, BR-2, BR-3

---

## Step 4 — Wire subscription create to real API

**Scope** *(all backend layers — wire phase)*:
- `src/Core/Application/Functionalities/Subscriptions/Commands/AddSubscription/` *(create)*
- `src/Host/BFF/Controllers/SubscriptionController.cs` *(modify — add POST)*
- `src/Presentation/Subscriptions/Services/SubscriptionService.cs` *(modify — add AddAsync)*
- `src/Test/Unit/Application/Subscriptions/AddSubscriptionHandlerTests.cs` *(create)*
- `src/Test/Integration/Endpoints/Subscriptions/PostSubscriptionTest.cs` *(create)*

**RED**:
- Unit: `Execute_ValidCommand_ReturnsSuccess`
- Unit: `Execute_PastStartDate_ReturnsFailure` (BR-2)
- Integration: `PostSubscription_ValidData_ReturnsCreated`
- Integration: `PostSubscription_MissingFields_ReturnsBadRequest`

**GREEN**:
- AddSubscriptionCommand with Validate() (BR-1 through BR-3)
- AddSubscriptionCommandHandler using Result<T>
- SubscriptionController POST /api/subscriptions (doctor role ⚠️)
- SubscriptionService.AddAsync (real Refit call)

**DB changes**: none (table created in Step 2)

**🛑 HUMAN GATE**:
- [ ] Behavioral: Integration tests pass; POST creates record; validation errors return 400
- [ ] Review: ⚠️ Authorization check (doctor role only — BR-4)
```

Notice several things about this plan:

1. **Step names describe behaviour**, not layers. "Display subscription list (stubbed)" tells you what the system can do after the step. "Create Entity" would not.
2. **Each slice is completed before the next starts.** Steps 1–2 are the "view" slice. Steps 3–4 are the "create" slice. They don't interleave.
3. **The stub phase is Presentation only.** No backend code, no database. Just the UI with fake data.
4. **The wire phase touches many layers in one step.** That's correct — a vertical slice is supposed to cross layers. What matters is that it delivers one coherent behaviour.
5. **Every step has concrete tests in the RED section.** The test method names, file paths, and run commands are specified before any production code is written.
6. **Risk areas are flagged.** Step 4 marks the authorization check with ⚠️ because it involves role-based access control.

### The Interview Before the Plan

Before writing a plan, the AI agent should confirm it has enough information. We encode an interview checklist:

1. **Reference feature identified?** Which existing feature in the codebase should be used as the pattern? (Mandatory — do not proceed without one.)
2. **Entity properties and types listed?** (From the spec.)
3. **Database changes needed?** New table, modify existing, or none?
4. **Relationships to existing entities?** Foreign keys, navigation properties?
5. **New API endpoints?** Verbs, routes, request/response shapes?
6. **New Blazor pages?** Routes, layouts, Components?
7. **NServiceBus events or messages?** Publishing, handling, side effects?
8. **Authentication claims needed?** Does the controller read user claims that need to be configured in the token server?

If any answer is missing, the AI asks focused questions at this stage and not later — not halfway through implementation.

---

## Pillar 3: Red-Green-Refactor with Human Gates

### The Loop

Each plan step is executed as a single Red-Green-Refactor cycle. This is the exact loop:

```
1. READ       Read the plan step. Understand scope, files, and the test to write.
2. RED        Write the failing test FIRST. Run it. Confirm it FAILS.
3. GREEN      Write minimal production code to make the test pass. Run. Confirm PASS.
4. REFACTOR   Clean up if needed. Do not change behaviour.
5. ANALYSE    Run static analysis. Fix all violations.
6. PROVE      Build (zero warnings) + all tests pass + format check passes.
7. 🛑 STOP    Present results. Wait for human approval.
8. MARK DONE  After approval, update plan checkboxes [ ] → [x].
```

### Why RED Must Come First

The most common AI mistake is writing the test and production code simultaneously. This seems efficient — but it bypasses the entire point of test-first development.

When you write the test first and confirm it fails, you prove two things:
1. **The test actually tests something.** A test that passes on the first run might be testing nothing — wrong assertion, wrong mock setup, wrong method under test. A confirmed failure proves the test would catch a real bug.
2. **The production code is causally linked to the test.** You know the code you wrote is what made the test pass. Without the confirmed failure, you can't be sure.

For AI coding, RED-first is even more critical. The AI generates plausible-looking tests that may have subtle bugs — wrong assertion values, mock setups that match any input, assertions that never execute because of early returns. A confirmed failure catches all of these before any production code is written.

### Example: RED-first in Practice

**Step 2 of the Subscriptions plan** asks us to implement the query that returns active subscriptions.

**RED** — The agent writes this test:

```csharp
[TestClass]
public class GetSubscriptionsQueryTests
{
    [TestMethod]
    public async Task Execute_WithActiveSubscriptions_ReturnsList()
    {
        // Arrange
        var mockRepo = new Mock<ISubscriptionRepository>();
        mockRepo.Setup(r => r.GetActiveByPatientAsync(42))
            .ReturnsAsync(new List<Subscription>
            {
                new() { Id = 1, PatientId = 42, MedicationName = "Metformin 500mg",
                         StartDate = new DateOnly(2026, 4, 1), EndDate = new DateOnly(2026, 10, 1) }
            });

        var query = new GetSubscriptionsQuery(mockRepo.Object);

        // Act
        var result = await query.Execute(42);

        // Assert
        Assert.AreEqual(1, result.Count);
        Assert.AreEqual("Metformin 500mg", result[0].MedicationName);
    }
}
```

The agent runs the test. **It fails** — `GetSubscriptionsQuery` doesn't exist yet. The failure is confirmed. This proves the test is wired correctly and will actually validate the query when it's built.

**GREEN** — Now the agent writes the minimal code: the entity, the repository interface, and the query implementation. It runs the test again. **It passes.** The agent knows its code is correct because the test that provably failed now provably passes.

**Without RED-first**, the agent would write the test and the query simultaneously. The test might pass — but did it pass because the code is correct, or because the mock setup accidentally matches everything? You can't tell. RED-first removes this ambiguity.

### The Human Gate

After PROVE, the agent presents its results and **stops**:

```
✅ Step 2 — Wire subscription list to real API

Build:       0 warnings, 0 errors
Tests:       24 passed, 0 failed
Format:      No changes needed

New files:
  - src/Core/Domain/Shared/Entities/Subscription.cs
  - src/Core/Persistence/Repositories/SubscriptionRepository.cs
  - src/Core/Application/Functionalities/Subscriptions/Queries/GetSubscriptions/...
  - src/Host/BFF/Controllers/SubscriptionController.cs
  - src/Test/Integration/Endpoints/Subscriptions/GetSubscriptionsTest.cs

🛑 HUMAN GATE:
  - [ ] Integration test GetSubscriptions_Authenticated_ReturnsOk passes
  - [ ] Entity, repo, query, controller follow Doctors pattern
```

The developer reviews the changes. They check that:
- The entity properties match the spec
- The repository query filters correctly (EndDate > today)
- The controller route matches the spec
- The integration test covers the documented error cases
- The naming follows the reference feature pattern

Only after explicit approval does the agent proceed to Step 3.

**Why this matters:** Without the gate, the AI compounds errors. A wrong DTO shape in Step 2 leads to a wrong Refit client in Step 3, which leads to a broken page in Step 4. By the time you notice, three steps need to be unwound. With gates, you catch the wrong DTO shape in Step 2 — before any dependent code exists.

### Bugfixes: Regression Test First

The same RED-first principle applies to bugfixes, but with a simpler workflow. No plan is needed for a 1-2 file fix, but a regression test is always required:

1. **Investigate** — Read the reported code, identify the root cause.
2. **RED** — Write a test that reproduces the bug. Confirm it fails.
3. **GREEN** — Make the minimal fix. Confirm the test passes.
4. **Verify** — Full build + test suite + format check.

The regression test ensures the bug can never return silently. It also forces the developer to understand the bug before fixing it — the AI can't just randomly change code until the symptoms disappear.

---

## Progress Tracking Across Sessions

AI coding sessions are stateless — context is lost when a session ends. This is a fundamental constraint. But our plan files solve it with a simple convention.

Each human gate has checkboxes:

```markdown
## Step 1 — Display subscription list (stubbed)
...
**🛑 HUMAN GATE**:
- [x] Behavioral: ViewModel test passes — list loads with 3 stubbed items
- [x] Review: Page layout matches Doctor list page pattern

## Step 2 — Wire subscription list to real API
...
**🛑 HUMAN GATE**:
- [ ] Behavioral: Integration tests pass; GET /api/subscriptions returns list
- [ ] Review: Entity, repo, query, controller follow Doctors pattern
```

When a new session starts — whether with the same AI model or a different one — the agent reads the plan file. It finds the first unchecked `[ ]`. That's Step 2. It knows:
- Step 1 is done and approved
- Step 2 is next
- What files to create, what tests to write, what behaviour to deliver

No context is lost. No work is repeated. Any agent, in any session, picks up exactly where the last one left off.

---

## Encoding the Process as AI Instructions

The entire process described in this article isn't a convention developers must remember. It is **encoded as instruction files** that the AI reads and follows automatically.

At NIHDI, we distribute these files to every project via a centralised template system. But the pattern works with any distribution method — even a shared Git repository that teams copy from.

Here are the key files and what each one teaches the AI:

### `AGENTS.md` — The Workflow Backbone

This file defines the planning gate, the mandatory workflow rules, the Red-Green-Refactor-Proof loop, the human gate protocol, and the vertical slice decomposition strategy. Every AI agent in the project reads this file.

Key instructions it contains:

```markdown
## Mandatory Workflow Rules

1. Show plan before coding.
2. Steps must be verifiable and testable.
3. ONE step per reply — never batch multiple steps.
4. RED before GREEN — write a failing test, confirm it fails, then write production code.
5. Every bugfix gets a regression test first.
6. Code analysis before gate — fix all violations, then run the full test suite.
7. 🛑 STOP at HUMAN GATE — do not proceed until user confirms.
```

### `planner.agent.md` — The Planning Specialist

This is a dedicated AI agent that only produces plans. It is constrained to read-only access to the codebase and can only create or edit files under `_plans/`. It cannot write production code, test code, or run builds. Its sole output is a plan file that the developer approves.

Key instructions it contains:

```markdown
<constraints>
- Only create or edit _plans/<FeatureName>.md. No other files.
- No production code, test code, SQL, or configuration.
- No builds, tests, or terminal commands.
- Read-only on the codebase — explore freely to inform the plan.
</constraints>

<investigate_before_planning>
Never write a plan step that references a file, class, or pattern you have not read.
Read the reference feature's implementation across all affected layers BEFORE writing any steps.
</investigate_before_planning>
```

The planner also has a self-check that validates the plan before presenting it:

```markdown
<self_check>
1. Every step name describes user-visible behavior, not a layer or technical artifact.
2. No two consecutive steps belong to different vertical slices.
3. Every step has at least one concrete behavioral verification in its HUMAN GATE.
4. Every step that adds a controller includes integration tests.
5. All file paths are specific and exist in (or follow) the reference feature's pattern.
6. No step depends on a later step.
7. Risk Areas are flagged with ⚠️.
</self_check>
```

### `copilot-instructions.md` — Project-Specific Rules

This is the main instruction file that every AI interaction reads. It defines the project structure, naming conventions, dependency matrix, DI registration patterns, and critical rules. It is template-specific — a Blazor Server project gets different instructions than a Blazor WASM project or a class library.

Key instructions it contains:

```markdown
<critical_rules>
1. After every code change, run all three:
   dotnet build src/MySolution.sln
   dotnet test --project src/Test/Unit/
   dotnet format src/MySolution.sln --verify-no-changes

2. Never throw for business errors — use Result<T> from Nihdi.Core.Functional.

3. DI via *Module + Scrutor suffix scanning only. Never services.AddScoped<T>() in Program.cs.

4. AsNoTracking() on all read queries. No N+1 patterns.

5. Match existing patterns in the same layer/feature before writing new code.

6. Follow the Mandatory Workflow Rules — one step per reply, RED before GREEN,
   stop at every HUMAN GATE.
</critical_rules>
```

### `build-feature/SKILL.md` — The Execution Engine

This is the skill file that the AI invokes when executing plan steps. It contains layer-specific code templates — not as copy-paste targets, but as a reference catalog. The AI looks up which layers the current step touches and applies the matching template, adapted to the reference feature's patterns.

Key guidance it contains:

```markdown
## Vertical Slice Strategy

Plans decompose features into vertical slices, not horizontal layers.
Each slice follows a two-phase pattern:

1. **Stub phase** — Build the UI with a mocked/stubbed service returning fake data.
   This lets the user validate the UI immediately.
2. **Wire phase** — Replace the stub with real production code from top to bottom:
   controller → application handler → domain → persistence → DB.

When executing a plan step, check whether it is a stub step or a wire step,
and apply the appropriate layers from the reference catalog below.
```

### `bugfix.agent.md` — The Regression-First Specialist

A dedicated agent for small fixes that enforces the regression test requirement:

```markdown
## Workflow

1. Investigate — read the code, identify the root cause.
2. RED — write a test that reproduces the bug. Confirm it FAILS.
   Do not write any production code yet.
3. GREEN — make the minimal fix. Confirm the test PASSES.
4. Verify — full build + test suite + format check.

Rules:
- Never skip the regression test. RED before GREEN.
- Never change more than 2 production files. Escalate to @planner if larger.
- Never refactor surrounding code — fix the bug only.
```

---

## Putting It All Together

The complete workflow looks like this:

```
Developer: "I need a subscriptions feature for patients."

     ┌─────────────────────────────────────┐
     │  1. SPECIFY                          │
     │  Write _specs/Subscriptions.md       │
     │  Define: stories, criteria, model,   │
     │  endpoints, rules, edge cases,       │
     │  out-of-scope                        │
     └──────────────┬──────────────────────┘
                    │
                    ▼
     ┌─────────────────────────────────────┐
     │  2. PLAN                             │
     │  AI reads spec + codebase            │
     │  AI asks clarifying questions        │
     │  AI writes _plans/Subscriptions.md   │
     │  4 steps: stub→wire × 2 slices      │
     │  🛑 Developer approves plan          │
     └──────────────┬──────────────────────┘
                    │
          ┌─────────┴─────────┐
          │                    │
          ▼                    ▼
     ┌──────────┐        ┌──────────┐
     │ Step 1   │        │ Step 3   │
     │ List UI  │        │ Form UI  │
     │ (stub)   │        │ (stub)   │
     └────┬─────┘        └────┬─────┘
          │                    │
          ▼                    ▼
     ┌──────────┐        ┌──────────┐
     │ Step 2   │        │ Step 4   │
     │ List API │        │ Create   │
     │ (wire)   │        │ API      │
     └────┬─────┘        │ (wire)   │
          │              └────┬─────┘
          │                    │
          └────────┬───────────┘
                   │
                   ▼
          Each step follows:
          RED → GREEN → REFACTOR
          → ANALYSE → PROVE → 🛑
```

At every 🛑, the developer reviews:
- Do the tests cover the spec's acceptance criteria?
- Does the code follow the reference feature's patterns?
- Are the naming conventions consistent?
- Are risk areas properly handled?

Only after approval does the next step begin. The developer stays in control of every design decision while the AI handles the mechanical work of writing code that conforms to established patterns.

---

## Summary

| Aspect | Vibe Coding | Spec-Driven Development |
|--------|-------------|------------------------|
| **Starting point** | Vague prompt | Structured spec with acceptance criteria |
| **Planning** | None — AI decides scope and approach | Explicit plan with vertical slices, approved before coding |
| **Decomposition** | Random or by layer | By user-visible behaviour (stub → wire) |
| **Testing** | Maybe, after the fact | RED-first — test must fail before code is written |
| **Verification** | Eyeball check | Build + tests + analysis + format — automated and mandatory |
| **Human involvement** | Rubber-stamp at the end | Gate at every step — developer reviews and approves |
| **Session continuity** | Lost on restart | Checkbox tracking in plan files |
| **Accountability** | Unclear — who designed this? | Developer owns spec and plan; AI executes |

The shift from vibe coding to spec-driven development is not about writing more documents. It's about **staying in control** of software you're responsible for. The spec makes intent explicit. The plan makes decomposition deliberate. Red-Green-Refactor makes every change verifiable. And human gates make the developer accountable for every step.

AI coding assistants are powerful. But power without control is just risk. Spec-driven development is how you keep the power and add the control.

---

*This article describes practices developed at NIHDI and encoded in the [Nihdi.Copilot.Sync](https://github.com/nickaerts/nihdi-copilot-sync) project, which distributes AI coding instructions as versioned templates across .NET projects.*
