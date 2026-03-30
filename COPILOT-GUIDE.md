# Copilot Customization — Developer Best Practice Guide

This guide teaches you how to use the Copilot customizations shipped under `.github/` in this project. The goal is a **short-iteration, plan-driven development process** where Copilot acts as your pair programmer — but you stay in control at every gate.

> **Recommended editor: Visual Studio Code.** As of March 2026, Visual Studio 2026 does not support agents, skills, or prompts — only instructions and scoped instructions are loaded. To get the full power of this guide (plan-driven workflow, `@planner`, `/build-feature`, `@code-analysis`, `@git`), use **VS Code with GitHub Copilot**.

---

## 1. Repo Structure

```
/                              # repo root
├── _plans/                    # Feature implementation plans (@planner output, versioned in git)
├── _specs/                    # Feature specifications (user stories, acceptance criteria, data model)
├── .github/                   # Copilot customizations (instructions, agents, skills, prompts)
│   └── templates/             # Reusable templates (spec-template.md, ...)
├── README.md
└── COPILOT-GUIDE.md           # ← this file
```

---

## 2. What's in `.github/`?

```
.github/
├── copilot-instructions.md              # Always-on rules — loaded in EVERY Copilot interaction
├── instructions/
│   ├── tests.instructions.md            # Auto-loaded when editing files in src/Test/**
│   ├── refit-client.instructions.md     # Auto-loaded when editing ServiceClients/Services
│   ├── domain-entity.instructions.md    # Auto-loaded when editing src/Core/Domain/**
│   ├── application-layer.instructions.md # Auto-loaded when editing src/Core/Application/**
│   ├── persistence-layer.instructions.md # Auto-loaded when editing src/Core/Persistence/**
│   ├── bff-controller.instructions.md   # Auto-loaded when editing src/Host/BFF/**
│   └── blazor-presentation.instructions.md # Auto-loaded when editing Presentation pages/ViewModels
├── agents/
│   ├── planner.agent.md                 # @planner — creates _plans/<Name>.md, never writes code
│   ├── code-analysis.agent.md           # @code-analysis — finds and fixes SA/CA/CS violations
│   ├── sonar-review.agent.md            # @sonar-review — runs SonarQube analysis (read-only)
│   └── git.agent.md                     # @git — branching, tagging, PRs, GitFlow
├── skills/
│   ├── build-feature/SKILL.md           # /build-feature — full vertical slice (7-layer recipe)
│   ├── add-endpoint/SKILL.md            # /add-endpoint — add API endpoint to existing feature
│   ├── add-blazor-page/SKILL.md        # /add-blazor-page — page + ViewModel + ServiceClient
│   ├── fix-violations/SKILL.md          # /fix-violations — code analysis rule lookup & fixes
│   └── e2e-test/SKILL.md               # /e2e-test — full-stack browser smoke test (manual only)
└── prompts/
    └── new-feature.prompt.md            # Reusable prompt template for new features
```


### How each type works

| Type | When loaded | Invoked by | VS Code | Visual Studio |
|------|-------------|------------|---------|---------------|
| **Instructions** (`copilot-instructions.md`) | Every chat/inline completion | Automatic | ✅ | ✅ |
| **Scoped instructions** (`instructions/*.md`) | When editing matching files (`applyTo` pattern) | Automatic | ✅ | ✅ |
| **Agents** (`agents/*.agent.md`) | On demand | `@agent-name` in chat | ✅ | ❌ |
| **Skills** (`skills/*/SKILL.md`) | On demand | `/skill-name` in chat | ✅ | ❌ |
| **Prompts** (`prompts/*.prompt.md`) | On demand | From prompt picker | ✅ | ❌ |

> **Visual Studio 2026 (March 2026)**: Only `copilot-instructions.md` and scoped instructions are loaded — agents, skills, and prompts are **not supported**. See [Section 6](#6-walkthrough--building-a-feature-visual-studio) for the Visual Studio fallback workflow.

---

## 3. Getting Started — `copilot-sync`

The `src/.github/` files and this `COPILOT-GUIDE.md` are managed by **`copilot-sync`** — a dotnet tool that replaces project-specific tokens (solution name, namespace root, etc.) and keeps your files in sync with the latest central templates.

### First-time setup

```bash
# 1. Install the tool globally
dotnet tool install --global Nihdi.Copilot.Sync

# 2. Initialize your project
cd /path/to/your/repo
copilot-sync init
```

The `init` command prompts for six values:

| Token | Prompt | Example for this project |
|-------|--------|--------------------------|
| `SolutionName` | Solution name | `{{SolutionName}}` |
| `NamespaceRoot` | Namespace root | `{{NamespaceRoot}}` |
| `DbContextName` | DbContext class name | `{{DbContextName}}` |
| `TestExePath` | Test exe path | `{{TestExePath}}` |
| `SonarProjectKey` | SonarQube project key | (project-specific) |
| `SonarServerUrl` | SonarQube server URL | (project-specific) |

It creates:
- `src/.github/` — all Copilot customization files with your project names baked in
- `COPILOT-GUIDE.md` — this guide (with your project's values in all examples)
- `.copilotrc.json` — configuration file storing your tokens (**commit this**)
- `.copilot-sync/hashes.json` — file checksums for conflict detection (**commit this**)
- `_plans/` and `_specs/` folders

### Updating templates

---

## 3. The Development Process — Plan-Driven, Short Iterations

This is the core workflow. Every feature follows the same cycle:

```
 ┌─────────────────────────────────────────────────┐
 │  0. SPEC — write _specs/<Name>.md (optional)    │
 │  1. PLAN — @planner creates /_plans/<Name>.md   │
 │  2. HUMAN GATE — you review and approve         │
 │  3. STEP N — one Red-Green-Refactor cycle       │
 │     RED:     write failing test                  │
 │     GREEN:   minimal code to pass                │
 │     REFACTOR: cleanup                            │
 │     ANALYSIS: fix all violations                 │
 │     PROOF:   build + all tests + format          │
 │  4. HUMAN GATE — you review the step            │
 │  5. Repeat 3–4 for each step                     │
 │  6. @git — branch, commit, PR                    │
 └─────────────────────────────────────────────────┘
```


### Step 0 — Feature Spec (`_specs/`)

Before planning, you can capture **what** a feature should do in `_specs/<Name>.md`:

- **Template**: `src/.github/templates/spec-template.md` — copy it, fill in the blanks.
- **Content**: user story, acceptance criteria, data model, API endpoints, business rules, events/messaging, UI pages, edge cases.
- **Who writes it?** You (the developer). Copilot can help reverse-engineer one from existing code.
- **When?** Optional but recommended for any feature touching ≥ 3 layers. The `@planner` agent reads the spec automatically when creating a plan.

### Spec vs Plan — what goes where

| | **Spec** (`_specs/`) | **Plan** (`_plans/`) |
|---|---|---|
| **Question answered** | _What_ does the feature do? | _How_ do we build it? |
| **Audience** | Product owner, developer, reviewer | Developer, Copilot agent |
| **User story** | ✅ | ❌ (references the spec) |
| **Acceptance criteria** | ✅ (Given/When/Then) | ❌ |
| **Data model** | ✅ Conceptual (entity, properties, relationships) | ✅ EF config, SQL DDL, column types, FK constraint names |
| **API endpoints** | ✅ Routes, params, request/response shapes | ✅ Controller class, attributes, DI injections |
| **Business rules** | ✅ Rule + error message only | ✅ Which class/method enforces it |
**Rule of thumb**: If you mention a C# class name, a MudBlazor component, a file path, or a Scrutor registration — it belongs in the **plan**, not the spec.

### When is a plan required?

| Situation | Plan needed? |
|-----------|-------------|
| New feature / vertical slice | **Yes** |
| Change touching ≥ 3 files | **Yes** |
| Risk area change (auth, PII, DB schema, shared contracts) | **Yes** |
| ≤ 2 file bugfix | No — but still RED-first |
| Config correction, simple refactor | No |

### VS Code Plan Mode vs `@planner` — when to use which

| | VS Code Plan Mode | `@planner` agent |
|---|---|---|
| **Output** | Ephemeral — shown in chat only | Persisted `_plans/<Name>.md` — versioned in git |
| **Scope** | Generic: any task | Project-specific: 7-layer recipe, RGR cycles, Risk Area flags, HUMAN GATE checkboxes |
| **Tool access** | Can run terminal, edit files | Read-only — **cannot write code**, enforces hard stop |
| **Human gate** | Soft — click Accept once | Hard — named artifact, must be approved before any code |
| **Multi-session** | Session-only | Plan file survives sessions; `/build-feature` reads it in a later session |

**Rule of thumb**:
- **≥ 3 files / new feature / Risk Area** → use `@planner` to produce a named plan file, then `/build-feature` to execute it.
- **≤ 2 file bugfix / simple refactor** → Plan Mode (or no plan) is fine.

> Plan Mode and `@planner` can be used together: use Plan Mode to sketch your thoughts, then hand the description to `@planner` to produce the formal `_plans/<Name>.md`.

### The HUMAN GATE principle

Copilot **stops and waits for your confirmation** at every gate. This is non-negotiable:

- After plan creation → you approve before any code is written
- After each step → you review the changes before the next step starts
- Before any Git push → you confirm the branch/commit/PR

**Why?** You own the code. Copilot proposes, you decide. This prevents runaway changes and keeps you aware of what's happening.

---

## 5. Walkthrough — Building a Feature (VS Code)

### Step 1a: Write the spec (`_specs/`)

Before invoking the planner, capture **what** the feature should do:

1. Copy the template: `src/.github/templates/spec-template.md` → `_specs/MyFeature.md`
2. Fill in: user story, acceptance criteria, data model, API endpoints, business rules, events, UI pages, edge cases.

> **Tip**: Keep the spec focused on _what_ — no C# class names, no file paths, no MudBlazor components. Those belong in the plan.

### Step 1b: Plan with @planner

Now invoke the planner **referencing the spec**:

```
You:     @planner Create a plan for My Feature — spec is in _specs/MyFeature.md
```

The planner agent:
- **Reads `_specs/MyFeature.md`** — uses the acceptance criteria, data model, API endpoints, and business rules as its primary input
- **Asks which existing feature to use as the reference pattern** (e.g., you might say *"Use MyAppointments — it's a similar read-only pattern"*). If you include this in the spec or the prompt, it skips the question.
- Reads the reference feature's implementation across all layers
- Asks clarifying questions only for details **not covered by the spec**
- Creates `_plans/MyFeature.md` with step-by-step Red-Green-Refactor cycles
- **Stops and waits for your approval**

> The planner **automatically looks for a matching spec** in `_specs/` when you mention a feature name. You can also point to it explicitly. The spec saves a round-trip of clarifying questions.

> The planner has **no execute or edit tools** (except `_plans/<Name>.md`). It cannot accidentally modify your code.

### Step 2: Review and approve the plan

Read `_plans/<Name>.md`. Check:
- Are all layers covered?
- Are the test methods named correctly?
- Are the scoped files correct?
- Any missing Risk Area flags?

Tell Copilot: *"Plan approved, start Step 1"*

### Step 3: Implement with /build-feature

```
You:     /build-feature Implement Step 1 from _plans/MyFeature.md
```

The skill guides Copilot through the exact code templates for each layer. The mandatory workflow per step:

1. **RED** — Write the test, run it, confirm it **fails**
2. **GREEN** — Write minimal production code, run the test, confirm it **passes**
3. **REFACTOR** — Clean up if needed
4. **CODE ANALYSIS** — Collect analyzer violations (`dotnet build 2>&1 | Select-String`), fix all, reformat
5. **PROOF** — `dotnet build` (zero warnings/errors) + all tests pass + `dotnet format --verify-no-changes`
6. **🛑 HUMAN GATE** — Copilot stops. You review.

### Step 4: Git workflow

```
You:     @git Create a feature branch and commit
```

The git agent knows your GitFlow strategy: `feature/*` from `dev`, PRs, tagging on `main` only.

---

## 6. Walkthrough — Building a Feature (Visual Studio)

> **Note**: As of March 2026, Visual Studio 2026 does not support agents, skills, or prompts. The workflow below uses `copilot-instructions.md` (loaded automatically) **plus scoped instructions** that activate when you edit files in specific layers. For the full plan-driven experience with `@planner`, `/build-feature`, and `@code-analysis`, use **VS Code**.

Visual Studio loads `copilot-instructions.md` automatically, **and** loads scoped instructions based on the file you're editing (see [Section 9](#9-scoped-instructions--automatic-context)). Together these provide layer-specific code templates and conventions.

### What activates automatically

| Layer you're editing | Scoped instruction loaded | Key guidance provided |
|---------------------|--------------------------|----------------------|
| `src/Core/Domain/**` | `domain-entity.instructions.md` | `IEntity`, plain C#, no EF attributes |
| `src/Core/Application/**` | `application-layer.instructions.md` | Commands, handlers, queries, `Result<T>` |
| `src/Core/Persistence/**` | `persistence-layer.instructions.md` | `BaseRepository<T>`, `IEntityTypeConfiguration<T>` |
| `src/Host/BFF/**` | `bff-controller.instructions.md` | Controller pattern, audit logging |
| Presentation pages/ViewModels | `blazor-presentation.instructions.md` | ViewModel lifecycle, MudBlazor, `IsBusy` guard |
| `src/Test/**` | `tests.instructions.md` | MSTest, AAA, Moq, integration test factory |
| ServiceClients/Services | `refit-client.instructions.md` | Refit patterns, DTO→Model mapping |

### Using `_specs/` to drive the conversation

1. **Write or review the spec** in `_specs/<FeatureName>.md`.

2. **Attach it when prompting** — in VS 2026 Copilot Chat, use the **#file** reference:
   ```
   #file:_specs/MyFeature.md Create a plan for this feature. Use ExistingFeature
   as reference feature for existing patterns.
   ```

3. **Reference it in each step**:
   ```
   #file:_specs/MyFeature.md #file:_plans/MyFeature.md
   Implement Step 2 (Application layer).
   ```

4. **Validate against acceptance criteria**:
   ```
   #file:_specs/MyFeature.md Does the current implementation satisfy AC-1 through AC-3?
   ```

### What you lose vs VS Code

| VS Code feature | VS 2026 equivalent |
|----------------|---------------------|
| `@planner` agent | Ask Copilot Chat to create a plan (rules are in `copilot-instructions.md`) |
| `/build-feature` skill | Follow the 7-layer table in `copilot-instructions.md` manually |
| `@code-analysis` agent | Prompt: *"Fix all analyzer warnings in this file"* |
| `@git` agent | Use the Git Changes pane or command line |
| `/e2e-test` skill | Not available — run the app manually |

---

## 7. Using Individual Skills

Skills are on-demand — invoke them by name when you need a specific recipe.

### /build-feature
**When**: Full new feature, 7-layer vertical slice.
```
/build-feature Implement the feature following the plan
```

### /add-endpoint
**When**: Adding an API endpoint to an **existing** feature (no new entity/table).
```
/add-endpoint GET endpoint to retrieve items by category
```

### /add-blazor-page
**When**: Adding a new page, dialog, or ViewModel.
```
/add-blazor-page List page with search and add dialog
```

### /fix-violations
**When**: Looking up or fixing specific analyzer rules.
```
/fix-violations SA1412
/fix-violations fix all warnings in the solution
```

### /e2e-test ⚠️ Manual only
**When**: Smoke-testing the full stack in a browser after completing all implementation steps.
```
/e2e-test verify application launch without exceptions
```

> **Do not include `/e2e-test` in the normal `/build-feature` flow.** It launches multiple services, opens a Playwright browser, and inspects logs — this consumes significant tokens and is slow. Use it **manually** when you want to validate the running application after all steps are complete.

---

## 8. Using Agents

Agents have **restricted tool access** — this is a safety feature.

| Agent | Can do | Cannot do |
|-------|--------|-----------|
| **@planner** | Read files, search, create todos, edit `_plans/<Name>.md` | Execute commands, edit code files |
| **@code-analysis** | Read, edit, execute, search, invoke sub-agents | — |
| **@sonar-review** | Read, execute, search | Edit files (read-only analysis) |
| **@git** | Read, edit, run Git commands | — |

### Combining agents

A typical feature development session:

```
@planner       → creates _plans/<Name>.md → ⏸️ you approve
Copilot        → per step: RED → GREEN → REFACTOR → CODE ANALYSIS → PROOF → ⏸️ you review
@sonar-review  → checks for deeper issues (optional, after all steps)
@git           → creates branch, commits, opens PR
```

---

## 9. Scoped Instructions — Automatic Context

These load automatically when you edit files matching their `applyTo` pattern:

| File | Triggers when editing | What it adds |
|------|----------------------|--------------|
| `tests.instructions.md` | Any file in `src/Test/**` | MSTest conventions, AAA structure, Moq patterns, integration test factory |
| `refit-client.instructions.md` | Files in `ServiceClients/**` or `Services/**` | Refit patterns, CookieHandler, DTO→Model mapping, registration |
| `domain-entity.instructions.md` | Any file in `src/Core/Domain/**` | Entity conventions, `IEntity`, plain C#, no EF attributes |
| `application-layer.instructions.md` | Any file in `src/Core/Application/**` | Commands, handlers, queries, `Result<T>`, `IMessagingService` |
| `persistence-layer.instructions.md` | Any file in `src/Core/Persistence/**` | `BaseRepository<T>`, `IEntityTypeConfiguration<T>`, `AsNoTracking()` |
| `bff-controller.instructions.md` | Any file in `src/Host/BFF/**` | Controller pattern, `Result<T>` → HTTP, audit logging |
| `blazor-presentation.instructions.md` | Presentation pages, ViewModels, dialogs | ViewModel lifecycle, `IsBusy` guard, MudBlazor, `IStringLocalizer` |

You don't need to invoke these — they activate automatically and add context to Copilot's responses.

---

## 10. Key Principles to Remember

### The 7 Critical Rules

These are enforced in every Copilot interaction (from `copilot-instructions.md`):

1. **After EVERY code change** → build + test + format
2. **Never throw for business errors** → use `Result<T>`
3. **DI via `*Module` + Scrutor** → never `services.AddScoped<T>()` in Program.cs
4. **`AsNoTracking()` on ALL reads** → no N+1
5. **`EnableAuthentication: false`** → only in `appsettings.UnitTest.json`
6. **Match existing patterns** → same layer/feature first
7. **One step per reply** → RED first, stop at every HUMAN GATE

### The Anti-Pattern Table

| ❌ Don't | ✅ Do |
|----------|-------|
| Skip the plan | Create `_plans/<Name>.md` first for anything ≥ 3 files |
| Write production code before tests | RED first — failing test, then GREEN |
| Batch multiple steps | One step per reply, HUMAN GATE between each |
| Register services in Program.cs | Use `*Module` classes with Scrutor |
| Skip `CancellationToken` | Propagate on every async call |
| Throw for validation errors | Return `Result<T>` |
| Push without build+test+format | Always verify before push |

### Customizing for Your Project

You can customize these files for your own project by editing them directly. If you want to add project-specific conventions, markers, or rules, consider adding them to the appropriate instruction or agent file.

### Adding project-specific customizations

| To add | Where | How |
|--------|-------|-----|
| New skill | `.github/skills/<name>/SKILL.md` | Create with YAML frontmatter |
| New agent | `.github/agents/<name>.agent.md` | Create with YAML frontmatter |
| New scoped instruction | `.github/instructions/<name>.instructions.md` | Create with `applyTo` in frontmatter |
| Project convention | `.github/copilot-instructions.md` | Edit directly |

---

## 12. Tips for Effective Use

1. **Start every feature with `@planner`** — even if you think you know what to do. The plan surfaces edge cases and gives you a checklist.

2. **Be specific in your prompts** — *"Add a GET endpoint to retrieve items by category"* is better than *"add an endpoint"*.

3. **Don't skip HUMAN GATES** — Copilot works best in short cycles. Review each step, catch issues early.

4. **Use `/fix-violations` as a lookup tool** — not just for fixing. Need to know what SA1412 means? Ask.

5. **Use `/e2e-test` sparingly** — it's a manual validation step, not part of the build loop.

6. **Combine agents** — `@planner` → code → `@code-analysis` → `@sonar-review` → `@git` is the full pipeline.

7. **Trust the instructions** — If Copilot suggests something that violates the Critical Rules, it means the instructions aren't loaded. Check that `.github/copilot-instructions.md` exists.

8. **Keep plans in `_plans/<Name>.md`** — Named plan files persist across sessions and can be reviewed in PRs.

9. **Keep your templates updated** — improvement to agents, skills, and instructions are made periodically.

10. **Visual Studio users** — As of March 2026, VS 2026 lacks agent/skill/prompt support. Copy the Critical Rules and Anti-Pattern table to a sticky note — these are your guardrails. Consider switching to **VS Code** for the full workflow.
