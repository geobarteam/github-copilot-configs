# Copilot Configs — Spec-Driven AI Development for Blazor WASM + .NET

A reusable template library of **GitHub Copilot agents, skills, instructions, and prompts** for AI-assisted .NET development. Copy these files into your project to get structured, repeatable AI guidance that follows [Anthropic best practices](https://docs.anthropic.com/en/docs/build-with-claude/prompt-engineering/overview) — plan first, test first, one step at a time.

## Why This Exists

AI coding assistants generate code fast — but without guardrails, teams drift into **vibe coding**: no plan, no verification, no accountability. This library solves that by encoding your architecture, conventions, and development process into configuration files that Copilot and Claude Code follow automatically.

**Target stack**: ASP.NET Core + Blazor WebAssembly + Client (BFF) · Onion/Screaming Architecture · CQRS-lite · .NET 10

---

## What's Inside

### Agents (`agents/`)

Specialized modes invoked via `@name` in VS Code chat.

| Agent | Purpose |
|-------|---------|
| `@planner` | Creates `_plans/<Feature>.md` with vertical-slice steps before any code is written |
| `@bugfix` | Diagnoses bugs — writes a regression test first, then fixes |
| `@debug` | Debug engineer using App Insights + Playwright MCP |
| `@git` | GitFlow branching, .gitignore management |

### Skills (`skills/`)

Step-by-step recipes invoked via `/name` in VS Code chat.

| Skill | Purpose |
|-------|---------|
| `/init` | **Start here** — bootstraps workspace by discovering tokens and replacing placeholders |
| `/build-feature` | Implements an approved plan step-by-step with Red-Green-Refactor |
| `/add-endpoint` | Adds a vertical-slice API endpoint to an existing feature |
| `/add-blazor-page` | Adds a Blazor page with ViewModel, ServiceClient, Refit, and MudBlazor |
| `/add-blazor-module` | Scaffolds a standalone WASM module (MVVM + HttpClient) |
| `/add-dbup` | Adds a new DbUp migration script (CREATE TABLE, ALTER, seed data, etc.) |
| `/e2e-test` | Full-stack browser testing with Playwright MCP |
| `/csharp-coding-standards` | Reference-only — C# coding standards and patterns (not invocable) |

### Instructions (`instructions/`)

Layer-specific conventions that **auto-activate** when you edit files matching their `applyTo` glob pattern.

| Instruction | Activates for |
|-------------|---------------|
| `domain-entity` | `src/Core/Domain/**` |
| `application-layer` | `src/Core/Application/**` |
| `persistence-layer` | `src/Core/Persistence/**` |
| `bff-controller` | `src/Host/Client/**` |
| `blazor-presentation` | `src/Presentation/**` |
| `refit-client` | `src/Presentation/**/ServiceClients/**`, `src/Infrastructure/Http/**` |
| `tests` | `src/Test/**` |

### Prompts (`prompts/`)

| Prompt | Purpose |
|--------|---------|
| `new-feature` | Kicks off a new vertical-slice feature: writes spec (`_specs/`) → identifies reference feature → creates plan (`_plans/`) → awaits approval |

### Templates (`templates/`)

| Template | Purpose |
|----------|---------|
| `spec-template.md` | Feature specification format — user stories, acceptance criteria, data model, business rules. Used to create `_specs/<Feature>.md` |

### Root Files

| File | Purpose |
|------|---------|
| `copilot-instructions.md` | Project-level Copilot instructions (copied to target project root) |
| `AGENTS.md` | Shared agent workflow rules — plan-first, Red-Green-Refactor, human gates |
| `CLAUDE.md` | Claude Code entrypoint (references `AGENTS.md`) |
| `BLOGPOST.md` | Part 1: Spec-driven development best practices |
| `BLOGPOST-PROCESS.md` | Part 2: The process in action |

---

## Getting Started

### 1. Copy files into your .NET project

Copy the following into your project repository:

```
your-project/
├── .github/
│   └── copilot-instructions.md   ← from this repo's copilot-instructions.md
├── AGENTS.md
├── CLAUDE.md                      ← optional, for Claude Code users
├── agents/
├── instructions/
├── skills/
├── prompts/
└── templates/
```

### 2. Run the `/init` skill

Open VS Code chat and type:

```
/init
```

The init skill will:

1. **Discover** your project's tokens automatically:
   - `{{SolutionName}}` — from `*.sln` file
   - `{{NamespaceRoot}}` — from `<RootNamespace>` in `*.csproj`
   - `{{DbContextName}}` — from classes extending `DbContext`
   - `{{TestExePath}}` — from test projects (Microsoft.Testing.Platform)
   - `{{CompanyName}}` — from `Directory.Build.props` or `*.csproj`
   - `{{ClientPort}}` — from `launchSettings.json`

2. **Ask** for values that can't be auto-discovered (copilot-sync tool name, etc.)

3. **Replace** all `{{token}}` placeholders across your config files

4. **Verify** no unreplaced tokens remain

### 3. Start building features

```
# 1. Write the spec (creates _specs/Subscriptions.md)
#    Use templates/spec-template.md as the format

# 2. Plan the feature (creates _plans/Subscriptions.md)
@planner Add subscription management with list and create

# 3. Review and approve the plan

# 4. Implement step by step
/build-feature Step 1 — Display subscription list
```

---

## Development Process

This library enforces a **spec-driven development process** built on three principles:

### Specify First

Every feature starts with a specification (`_specs/<Feature>.md`). The spec captures user stories, acceptance criteria, data model, business rules, and non-goals. It is the contract between developer intent and AI execution — without it, the AI guesses what you want. The `templates/spec-template.md` provides the format.

### Plan Before Code

Once the spec is approved, the `@planner` agent produces a plan (`_plans/<Feature>.md`). The plan decomposes the feature into **vertical behavior slices** — not horizontal layers. Each slice delivers observable, testable functionality across all necessary layers. Every plan step traces back to an acceptance criterion in the spec.

### Red-Green-Refactor

Every implementation step follows the cycle:

```
RED     → write a failing test first
GREEN   → write minimal code to pass
REFACTOR → clean up
PROVE   → build + all tests + format check = all green
🛑 STOP → wait for human approval before proceeding
```

### Human Gates

The AI never proceeds past a checkpoint without explicit user approval. This applies after every plan, every implementation step, and before any git push.

---

## Project Structure (Target)

The instructions and skills in this library target projects with this structure:

```
src/
├── Host/Client/          # ASP.NET Core host: serves WASM static files + BFF API
├── Host/Wasm/            # Blazor WebAssembly browser entry point
├── Presentation/         # Razor Class Library (pages, ViewModels, Services)
├── Infrastructure/       # WASM-side: CookieHandler, AuthState, AntiforgeryTokenStore
├── Core/Application/     # Commands, Queries, Handlers (CQRS-lite)
├── Core/Domain/          # Entities, value objects (zero dependencies)
├── Core/Persistence/     # EF Core DbContext, repositories
├── Contracts/            # Shared DTOs 
└── Test/{Unit,Common,UI,Integration}/
```

---

## Customization

These files are **templates**. After running `/init`, adapt them to your project:

- **Different architecture?** Edit the project structure in `copilot-instructions.md` and update `applyTo` globs in instruction files.
- **Different test runner?** Update `{{TestExePath}}` references in `AGENTS.md` and `copilot-instructions.md`.
- **Additional layers?** Create new instruction files following the naming convention `<topic>.instructions.md` with an `applyTo` glob.
- **Additional agents?** Create `<name>.agent.md` in `agents/` following the existing pattern.

---

## File Type Conventions

| Type | Location | Naming | Invocation |
|------|----------|--------|------------|
| Instructions | `instructions/` | `<topic>.instructions.md` | Auto-loaded by `applyTo` glob |
| Agents | `agents/` | `<name>.agent.md` | `@name` in chat |
| Skills | `skills/<name>/` | `SKILL.md` | `/name` in chat |
| Prompts | `prompts/` | `<name>.prompt.md` | Prompt picker in VS Code |
| Templates | `templates/` | `<name>.md` | Referenced by agents/skills |

---

## Further Reading

- [Part 1: From Vibe Coding to Spec-Driven Development](BLOGPOST.md) — best practices for AI-assisted coding
- [Part 2: The Process in Action](BLOGPOST-PROCESS.md) — hands-on walkthrough

---

## License

This is a template library. Copy, adapt, and use freely in your projects.
