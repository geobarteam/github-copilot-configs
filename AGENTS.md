# Agent Workflow

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
3. RUN     — dotnet test --filter "<TestMethod>" → confirm FAIL
4. GREEN   — write minimal production code to pass
5. RUN     — dotnet test --filter "<TestMethod>" → confirm PASS
6. REFACTOR — cleanup if needed
7. CODE ANALYSIS — fix all violations before proving:
   a. dotnet build src/MyApp.sln 2>&1 | Select-String -Pattern ": (warning|error) (SA|SX|CA|CS|MSTEST)\d+" | Sort-Object | Get-Unique
   b. dotnet format src/MyApp.sln  (auto-fix formatting)
   c. Fix any remaining violations manually (skip disabled rules)
   d. Repeat a–c until the Select-String output is empty
8. PROVE:
   - dotnet build src/MyApp.sln → succeeded (zero warnings/errors)
   - dotnet test → all pass
   - dotnet format src/MyApp.sln --verify-no-changes → exit 0
9. 🛑 STOP — present results, wait for user approval
10. MARK DONE — update _plans/<FeatureName>.md: change [ ] → [x] on this step's checkboxes
```

**Never batch steps. Never skip RED. Never skip CODE ANALYSIS. Never proceed past 🛑 without user confirmation.**

---

## Verification Commands

After every code change, run all three (same as defined in `copilot-instructions.md`):
```
dotnet build src/MyApp.sln
dotnet test
dotnet format src/MyApp.sln --verify-no-changes
```

---

## Human Review Gates

The agent **stops and waits for user confirmation** at every gate. This is non-negotiable:

- After plan creation → user approves before any code is written.
- After each step → user reviews the changes before the next step starts.
- Before any Git push → user confirms the branch/commit/PR.
- Risk area changes (Auth/OIDC, PII, shared contracts, domain/application layers, DB schema, controller routes/DTOs, blob/file storage) → user explicitly reviews.

---

## Plan File Format

Plans live at `_plans/<FeatureName>.md` (repo root). Each step is one Red-Green-Refactor cycle:

```markdown
# Plan: <Feature Name> — <User Story>

## Overview
<1–3 sentences: what this feature does, which existing feature to use as pattern reference.>

---

## Step N — <Name>

**Scope**:
- `<path/file.cs>` *(create | modify)*
- `<path/testfile.cs>` *(create | modify)*

**RED** *(write this test first, run it, confirm it fails before writing production code)*:
- Test file: `<path>`
- Test method: `<MethodName>`
- Failing-run command: `dotnet test --filter "<MethodName>"`

**GREEN** *(minimal production code to make RED pass)*:
- <What to implement — be specific about class names, interfaces, properties, method signatures>

**REFACTOR**:
- <Cleanup notes or "none">

**DB changes**: <"none" or describe the SQL file to create/modify>

**AGENT PROOF**: build + all tests + code analysis + format — all green

**🛑 HUMAN GATE**:
- [ ] Behavioral verification: <how to confirm new behavior>
- [ ] Code review: <what the reviewer should check>
```

---

## Planning Rules

1. **Verifiable steps.** Each step must be independently verifiable through observable system behavior. The HUMAN GATE must include at least one concrete verification that asserts the system now does something it didn't before. "Code review" alone is never the sole verification.
2. **Testable steps.** Each step should produce at least one new or updated automated test.
3. **One step = one RGR cycle.** Never combine multiple layers or multiple test classes.
4. **Be specific.** List exact file paths, class names, method signatures, property names.
5. **Follow existing patterns.** Read the reference feature across all layers first. Mirror its structure.
6. **Flag Risk Areas** with ⚠️ and explain what the reviewer should verify.
7. **Respect the dependency matrix** from `copilot-instructions.md`. Order steps so no step depends on a later step.
8. **DB changes are SQL only** — `Database/Tables/`, user deploys via Schema Compare.
9. **Keep the plan updatable** — mark HUMAN GATE checkboxes `[x]` as steps are approved. First unchecked `[ ]` = next step to implement.
