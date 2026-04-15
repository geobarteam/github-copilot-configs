# Agent Workflow — {{SolutionName}}

<!-- Shared agent behavior: plan-first discipline, red/green/refactor/proof loop,
     human gates, task execution rules. Referenced by CLAUDE.md for Claude Code.
     Repo policy (coding standards, architecture, naming) lives in .github/copilot-instructions.md. -->

---

## Planning Gate

> **Before any tool call**: does this change need a plan? If yes → create `_plans/<FeatureName>.md` (use `@planner` or follow the template in `planner.agent.md`) → **STOP and WAIT for approval**.

| Situation | Plan required? |
|-----------|---------------|
| New feature / vertical slice | **Yes** |
| Change touching ≥ 3 files | **Yes** |
| Risk area change (auth, PII, DB schema, shared contracts) | **Yes** |
| ≤ 2 file bugfix | No — but still RED-first |
| Config correction, simple refactor | No |

---

## Mandatory Workflow Rules

1. **Show plan before coding.**
2. **Steps must be verifiable and testable** — every step must produce observable behavior (a test passes, an API responds, a UI renders) and at least one automated test. Merge code-review-only steps into an adjacent step.
3. **ONE step per reply** — never batch multiple steps.
4. **RED before GREEN** — write a failing test, confirm it fails, then write production code.
5. **Every bugfix gets a regression test first.**
6. **Code analysis before gate** — after REFACTOR, fix all violations, then run the full test suite.
7. **🛑 STOP at HUMAN GATE** — do not proceed until user confirms.
8. **Mark done** — after approval, change `[ ]` → `[x]` in `_plans/<FeatureName>.md`. First unchecked `[ ]` = next step.

---

## Red-Green-Refactor-Proof Loop

Each implementation step follows this cycle:

```
1. READ    — read the plan step, understand scope and files
2. RED     — write the failing test FIRST
3. RUN     — {{TestExePath}} --filter "<TestMethod>" → confirm FAIL
4. GREEN   — write minimal production code to pass
5. RUN     — {{TestExePath}} --filter "<TestMethod>" → confirm PASS
6. REFACTOR — cleanup if needed
7. CODE ANALYSIS — fix all violations before proving:
   a. dotnet build src/{{SolutionName}}.sln 2>&1 | Select-String -Pattern ": (warning|error) (SA|SX|CA|CS|MSTEST)\d+" | Sort-Object | Get-Unique
   b. dotnet format src/{{SolutionName}}.sln  (auto-fix formatting)
   c. Fix any remaining violations manually (skip disabled rules)
   d. Repeat a–c until the Select-String output is empty
8. PROVE:
   - dotnet build src/{{SolutionName}}.sln → succeeded (zero warnings/errors)
   - {{TestExePath}} → all pass
   - dotnet format src/{{SolutionName}}.sln --verify-no-changes → exit 0
9. 🛑 STOP — present results, wait for user approval
10. MARK DONE — update _plans/<FeatureName>.md: change [ ] → [x] on this step's checkboxes
```

**Never batch steps. Never skip RED. Never skip CODE ANALYSIS. Never proceed past 🛑 without user confirmation.**

---

## Verification Commands

After every code change, run all three (same as defined in `copilot-instructions.md`):
```
dotnet build src/{{SolutionName}}.sln
{{TestExePath}}
dotnet format src/{{SolutionName}}.sln --verify-no-changes
```
`dotnet test` is not supported — use the `.exe` directly (`Microsoft.Testing.Platform` + .NET 10 SDK).

---

## Human Review Gates

The agent **stops and waits for user confirmation** at every gate. This is non-negotiable:

- After plan creation → user approves before any code is written.
- After each step → user reviews the changes before the next step starts.
- Before any Git push → user confirms the branch/commit/PR.
- Risk area changes (Auth/OIDC, PII, shared contracts, domain/application layers, DB schema, controller routes/DTOs, blob/file storage) → user explicitly reviews.

---

## Plan File Format

Plans live at `_plans/<FeatureName>.md` (repo root). Each step is one Red-Green-Refactor cycle delivering a **vertical behavior slice** (not a horizontal layer).

Step names must describe **what the user or system can do** after the step — never name a step after a layer or technical artifact.

```markdown
# Plan: <Feature Name> — <User Story>

## Overview
<1–3 sentences: what this feature does, which existing feature to use as pattern reference.>

---

## Step N — <Behavior slice: what the user/system can do after this step>

**Scope** *(all layers touched by this slice)*:
- `<path/file.cs>` *(create | modify)*
- `<path/testfile.cs>` *(create | modify)*

**RED** *(write these tests first, run them, confirm they fail before writing production code)*:
- Test file: `<path>`
- Test method: `<MethodName>`
- Failing-run command: `{{TestExePath}} --filter "<MethodName>"`

**GREEN** *(minimal production code across all necessary layers to make RED pass)*:
- <What to implement — be specific about class names, interfaces, properties, method signatures>

**REFACTOR**:
- <Cleanup notes or "none">

**DB changes**: <"none" or describe the SQL file to create/modify>

**AGENT PROOF**: build + all tests + code analysis + format — all green

**🛑 HUMAN GATE**:
- [ ] Behavioral verification: <how to confirm new behavior — e.g., integration test, API response, UI renders data>
- [ ] Code review: <what the reviewer should check>
```

---

## Vertical Slice Decomposition

Features must be decomposed into **vertical behavior slices**, not horizontal layers. Each slice delivers user-visible functionality across all necessary layers in a single step.

**Strategy: UI first with mocks, then replace mocks top-to-bottom.**

1. **Start at the UI** — build the page/form/component with a stubbed or mocked service that returns fake data. This lets the user validate the UI and interaction design immediately.
2. **Replace mocks with real code** — in subsequent steps, wire the real backend from top to bottom (controller → application handler → domain → persistence → DB). Remove the stub.
3. **Fail fast** — the goal is to integrate all layers as early as possible so mismatches between layers surface immediately, not at the end of a long horizontal build-out.

This means a single step may touch Domain, Application, Persistence, Contracts, Controller, and Database — that is expected and correct. What matters is that each step delivers one **observable behavior** the user or a test can verify.

> **Anti-pattern:** Step 1 "Create the Entity", Step 2 "Add the Repository", Step 3 "Build the Controller", Step 4 "Create the Page" — this is horizontal slicing. The entity, repo, controller, and page for reading a list should all be in the same step (or stubbed-UI step + wired-backend step).

### Example — "Subscriptions" feature (two slices: list + create)

**Slice A — View subscriptions:**
1. Build the subscription list page with a stubbed service returning fake data (Presentation only). User validates the UI.
2. Implement the real backend: controller GET endpoint, Refit client, Application query, Persistence repository, DB table. Remove the stub.

**Slice B — Create a subscription:**
3. Build the create-subscription form with validation and a stubbed submit (Presentation only). User validates form and error display.
4. Implement the real backend: controller POST endpoint, Refit client, Application command, Domain rules, Persistence, DB script. Wire the form to the real backend.

---

## Planning Rules

1. **Vertical slices, not horizontal layers.** Each step delivers user-visible behavior. Name steps by what the user or system can do after the step (e.g. "Display subscription list (stubbed)", "Wire subscription list to real API"), not by which layer is created (e.g. NOT "Create Entity", "Add Repository").
2. **One slice at a time.** Complete the full vertical flow (UI + backend) of one slice before starting the next. Never interleave steps from different slices.
3. **Verifiable steps.** Each step must be independently verifiable through observable system behavior. The HUMAN GATE must include at least one concrete verification that asserts the system now does something it didn't before. "Code review" alone is never the sole verification.
4. **Testable steps.** Each step should produce at least one new or updated automated test.
5. **One step = one RGR cycle.** A step may touch many layers (that is expected for vertical slices) but should represent one coherent behavior. If a slice is too large to verify in one cycle, split it into smaller behavior slices (not into layers).
6. **Be specific.** List exact file paths, class names, method signatures, property names.
7. **Follow existing patterns.** Read the reference feature across all layers first. Mirror its structure.
8. **Flag Risk Areas** with ⚠️ and explain what the reviewer should verify.
9. **Respect the dependency matrix** from `copilot-instructions.md`. Order steps so no step depends on a later step.
10. **DB changes are DbUp migration scripts** — `Database/Scripts/`, numbered sequentially. Use `/add-dbup` skill to create new scripts.
11. **Keep the plan updatable** — mark HUMAN GATE checkboxes `[x]` as steps are approved. First unchecked `[ ]` = next step to implement.