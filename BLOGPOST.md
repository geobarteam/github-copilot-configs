# From Vibe Coding to Spec-Driven Development: Best Practices for AI-Assisted Coding with GitHub Copilot

*How to standardise AI coding guidance across teams — and how you can do the same.*

> **Part 1 of 2.** This article covers best practices and the copilot-sync template system. For a hands-on walkthrough of the spec-driven process, see [Part 2: The Process in Action](BLOGPOST-PROCESS.md).

---

## Introduction

AI coding assistants like GitHub Copilot are changing how developers write software. But without clear practices, teams drift into what we call **vibe coding** — letting the AI generate code with no plan, no verification, and no accountability. The result? Code nobody fully understands.

This article distils best practices for using GitHub Copilot effectively, and explains how to build a centralised template system — **copilot-sync** — to propagate these practices across every project in an organisation.

---

## Part 1: Best Practices for AI-Assisted Coding

### Understand the Core Concepts

Before diving into workflows, every developer should understand three foundational concepts:

**The Model** — The LLM behind Copilot. Different models have different strengths, costs, and context windows. Choosing the right model for the task is a deliberate decision, not an afterthought.

**The Context** — Everything the model sees when generating a response: your open files, instructions, conversation history, and workspace structure. Context quality directly determines output quality.

**The Agent** — The orchestration layer that can inspect files, reason about tasks, propose plans, make changes, run checks, launch commands, and iterate through workflows. Agents interact with tools including MCP servers, making them far more capable than simple chat completions.

### Treat Tokens as a Limited Resource

All developers in our organisation receive an institution-managed GitHub Copilot Pro subscription. But tokens — the unit of model consumption — are finite. Practical consequences:

- Use **short, focused sessions**
- Keep prompts **specific and bounded**
- Choose the **right model for the task**
- Avoid unnecessary retries in polluted sessions
- Use **lighter models** for simple work
- Reserve **premium models** for harder problems

### Choose the Right Model for the Job

Not every task needs the most powerful model. Here's how we think about model selection:

| Model | Best for | Relative cost |
|-------|----------|---------------|
| Claude Haiku 4.5 / Gemini Flash | Cheap, simple tasks | ~0.33× |
| Claude Sonnet 4.6 / GPT-5.3-Codex | Daily default for coding | 1× |
| Gemini 2.5 Pro | Large-context reasoning | 1× |
| Claude Opus 4.6 | Hardest coding and review tasks | 3× |

**The rule**: use Haiku or Flash for quick simple work, Sonnet as the daily default, and Opus only when the problem genuinely demands it. Don't burn premium tokens on formatting a JSON file.

### The Biggest Risk: Losing Control

The main danger of AI-assisted coding isn't wrong code — it's **no longer knowing what the AI changed, why it changed it, or whether it still matches the design**.

| Vibe Coding | Spec-Driven Development |
|-------------|------------------------|
| Vague goal, no constraints | Clear goal, explicit constraints, reviewed plan |
| Long unstructured sessions | Short iterations with build/test/review gates |
| AI decides design and validation | You stay accountable for design, code, and validation |

The antidote is **spec-driven development**: start with a specification, produce a plan, implement in small verified steps, and stay in control throughout.

### Keep Sessions Short

A good AI coding session has one task, one goal, a small scope, only relevant files, and clear validation criteria. A bad session involves multiple unrelated tasks, too much history, broad exploration, repeated retries, and unclear ownership.

**Rule: if the topic changes, start a new session.** Session pollution — accumulated irrelevant context — is one of the most common causes of degraded AI output.

### Anthropic Principle #1: Give the Model a Way to Verify Its Work

Verification is the **single biggest improvement** you can make in AI-assisted coding.

What counts as verification:
- Failing tests that turn green
- Successful builds
- Static analyzers passing
- Expected output matching
- Deterministic scripts

Without verification, the AI produces plausible but potentially wrong code, and you become the only feedback loop — a feedback loop that gets tired and loses attention.

In our workflow, every change follows: **RED → GREEN → REFACTOR → ANALYSIS → PROOF → HUMAN GATE**.

### Anthropic Principle #2: Explore → Specify → Plan → Implement → Commit

This is the workflow backbone:

1. **Explore** — Understand the codebase, existing patterns, and the real problem
2. **Specify** — Write a detailed, AI-friendly specification (acceptance criteria, constraints, edge cases)
3. **Plan** — Produce a concrete implementation plan *before* code changes
4. **Implement** — Execute one step at a time and verify after each step
5. **Commit** — Checkpoint small, validated changes in Git

Planning is most valuable for new features, multi-file changes, or risk areas. Tiny fixes can be done directly — but still verified.

### Anthropic Principle #3: Provide Specific Context

Good prompts are not long prompts — they are **scoped, specific, and verifiable**.

| Weak Prompt | Better Prompt |
|-------------|---------------|
| "Fix the login bug." | "Reproduce the session-timeout bug, inspect auth flow, write a failing test, fix root cause, verify it passes." |
| "Add an endpoint." | "Add GET endpoint X using feature Y as reference pattern; no new table; add tests." |
| "Improve this page." | "Use existing page pattern, keep same components, add validation and proof steps." |

The pattern is always the same: scope the task, reference existing patterns, and define what "done" looks like.

---

## Part 2: Scaling Best Practices with Centralised AI Templates

Knowing best practices is one thing. Making sure every project in the organisation follows them is another. We solved this with **copilot-sync** — a tool that centrally authors and distributes GitHub Copilot customisation files across all team projects.

### The Problem

GitHub Copilot's behaviour is shaped by files in the `.github/` directory: `copilot-instructions.md`, agent definitions, skills, instruction files, and prompts. These files encode coding standards, architectural patterns, verification commands, and domain knowledge.

Without centralisation, each team maintains its own copy. Problems quickly emerge:
- **Drift** — Teams diverge from shared standards
- **Duplication** — The same patterns are written and maintained independently in every repo
- **Stale guidance** — Improvements in one project never reach the others
- **Onboarding friction** — New projects start with no AI guidance at all

### The Solution: A Template Package + CLI Tool

Copilot-sync is a dual-package .NET solution:

| Package | Role |
|---------|------|
| **Copilot.Customizations** | NuGet content package — embeds tokenised `.github/` template files as assembly resources |
| **copilot-sync** | `dotnet tool` CLI — reads project config, replaces tokens, syncs files to the target repo |

The core idea: **author templates once, distribute everywhere, customise per project via tokens**.

### How Templates Are Organised

Templates follow a **hierarchical composition model** with shared and template-specific layers:

```
templates/
├── manifest.json                  # Template registry
├── shared/                        # Files common to ALL templates
│   ├── agents/                    # Agent definitions
│   ├── instructions/              # Domain-specific coding instructions
│   ├── skills/                    # Reusable skill definitions
│   ├── prompts/                   # Prompt templates
│   └── AGENTS.md                  # Agent workflow rules
├── BlazorServer-3Tier/            # Blazor Server + BFF specifics
│   ├── copilot-instructions.md
│   ├── COPILOT-GUIDE.md
│   ├── CLAUDE.md
│   └── manifest.json
├── BlazorWasm-3Tier/              # Blazor WASM + API specifics
└── Library/                       # .NET library template
```

**Shared files** provide common guidance — NServiceBus patterns, persistence layer conventions, test structure, domain entity rules, and reusable skills like PDF/Excel generation or feature scaffolding.

**Template-specific files** override shared files when architecture differs. A Blazor Server project needs different instructions than a class library.

The manifest in each folder maps source files to output paths, supporting **dual-write** — a single template file can be written to both `.github/` and `src/.github/` for tooling compatibility:

```json
{
  "copilot-instructions.md": [
    ".github/copilot-instructions.md",
    "src/.github/copilot-instructions.md"
  ]
}
```

### What Gets Distributed

The template system distributes a complete AI coding environment:

| Artefact | Purpose |
|----------|---------|
| `copilot-instructions.md` | Core coding standards, critical rules, verification commands |
| `COPILOT-GUIDE.md` | Architecture-specific guidance (layer structure, patterns, conventions) |
| `CLAUDE.md` | Claude Code / Opus-specific instructions |
| `AGENTS.md` | Workflow rules: planning gates, Red-Green-Refactor loops, human gates |
| Agent definitions (`.agent.md`) | Multi-turn workflow orchestrators (e.g., planner agent) |
| Instructions (`.instructions.md`) | Domain-specific rules (BFF controllers, persistence, NServiceBus) |
| Skills (`SKILL.md`) | Reusable code generation tasks (add endpoint, build feature, generate reports) |
| Prompts (`.prompt.md`) | Reusable query templates |

### Token Replacement: Project-Specific Customisation

Templates use `{{MustacheStyle}}` placeholders that get replaced with project-specific values at sync time:

| Token | Example |
|-------|---------|
| `{{SolutionName}}` | `FindMyDoctor` |
| `{{NamespaceRoot}}` | `Contoso.FindMyDoctor` |
| `{{DbContextName}}` | `FindMyDoctorDbContext` |
| `{{TestExePath}}` | `.\src\Test\Unit\bin\Debug\net10.0\Contoso.FindMyDoctor.Unit.Tests.exe` |
| `{{SonarProjectKey}}` | `contoso-find-my-doctor` |
| `{{SonarServerUrl}}` | `https://sonarqube.example.com` |

**Derived tokens** are computed automatically — solution file names, project file paths, and other values that follow naming conventions.

A template file containing:

```markdown
# Copilot Instructions — {{SolutionName}}
Run tests: {{TestExePath}}
Namespace: {{NamespaceRoot}}
```

Becomes, after sync:

```markdown
# Copilot Instructions — FindMyDoctor
Run tests: .\src\Test\Unit\bin\Debug\net10.0\Contoso.FindMyDoctor.Unit.Tests.exe
Namespace: Contoso.FindMyDoctor
```

### The CLI Workflow

#### Initialise a New Project

```bash
copilot-sync init --template BlazorServer-3Tier
```

The tool interactively prompts for token values, creates `.copilotrc.json`, and syncs all template files with token replacement. A new project gets a complete, organisation-standard AI coding environment in seconds.

#### Update to Latest Templates

```bash
copilot-sync update
```

When the central templates improve — new skills, better instructions, updated patterns — every project pulls the latest version. The update command detects local modifications and handles them safely.

#### CI Version Check

```bash
copilot-sync check
```

Returns exit code 1 if templates are outdated, making it easy to enforce currency in CI pipelines.

### Conflict Detection: Respecting Local Changes

The sync tool uses **SHA-256 hash-based conflict detection**. After each sync, file hashes are stored in `.copilot-sync/hashes.json`. On the next update:

1. The current file's hash is compared against the stored hash
2. If they differ, the file was locally modified → **conflict**
3. The incoming template version is saved as `.copilot-sync/conflicts/<file>.incoming`
4. The local file is preserved untouched
5. The developer merges manually

This means teams can safely customise their AI guidance without fear of losing changes on the next update. The `--force` flag overrides conflicts when needed, and the `overrides` array in config permanently exempts files from sync.

### The Config File

Each consumer project commits a single `.copilotrc.json`:

```json
{
  "version": "1.0.0",
  "packageVersion": "1.0.5",
  "template": "BlazorServer-3Tier",
  "tokens": {
    "SolutionName": "FindMyDoctor",
    "NamespaceRoot": "Contoso.FindMyDoctor",
    "DbContextName": "FindMyDoctorDbContext"
  },
  "overrides": [
    ".github/skills/proprietary-skill/SKILL.md"
  ],
  "lastSyncedAt": "2026-03-15T14:23:42Z"
}
```

This is the only file teams need to maintain. Everything else is generated.

### The Override Mechanism

Sometimes a project needs AI guidance that diverges from the template — a domain-specific skill, a custom instruction file, or a modified workflow. The `overrides` array permanently skips listed files during updates:

```json
{
  "overrides": [
    ".github/instructions/custom-domain.md",
    ".github/skills/proprietary-skill/SKILL.md"
  ]
}
```

This gives teams the flexibility to extend while still receiving updates for everything else.

---

## Part 3: Setting Up Your Own Centralised AI Template System

You don't need to use our exact tooling. The pattern is what matters. Here's how to build your own:

### Step 1: Audit Your AI Artefacts

Start by collecting what your best teams are already doing:
- What's in their `copilot-instructions.md`?
- What coding standards do they encode?
- What verification commands do they require?
- What architectural patterns are documented?
- What skills or prompts have they created?

### Step 2: Create a Shared Template Repository

Organise your findings into a template structure:
- **Shared layer**: coding standards, test conventions, common patterns — things every project needs
- **Template-specific layers**: architecture-specific guidance (microservices vs monolith, React vs Blazor, etc.)
- **Token placeholders**: anything project-specific (names, paths, URLs) becomes a `{{Token}}`

### Step 3: Build a Distribution Mechanism

Options range from simple to sophisticated:

| Approach | Complexity | Trade-offs |
|----------|-----------|------------|
| Git submodule | Low | Manual updates, no token replacement |
| Script that copies + sed | Low | Fragile, no conflict detection |
| NuGet/npm package + CLI tool | Medium | Versioned, automated, conflict-safe |
| Template repository + GitHub Actions | Medium | GitHub-native, but limited customisation |

We chose the NuGet package approach because it gives us versioning, conflict detection, and CI enforcement — all critical for a large organisation.

### Step 4: Encode Your Workflow

The most valuable thing to centralise isn't boilerplate — it's **workflow**. Encode your development process into agent definitions and instruction files:

- Red-Green-Refactor loops with mandatory verification
- Planning gates before implementation
- Human review checkpoints
- Build and analysis commands that must pass

When the AI follows these workflows, it produces code that meets your standards by default.

### Step 5: Create a Feedback Loop

The real power of centralisation is the **feedback loop**:

1. A developer discovers a better pattern or a new skill
2. They contribute it back to the central template
3. Every project in the organisation benefits on the next sync

This turns individual learning into **organisational learning**. AI guidance improves continuously, driven by real experience across all teams.

### Step 6: Enforce in CI

Add a check to your CI pipeline that fails when templates are outdated:

```yaml
- script: copilot-sync check
  displayName: 'Verify AI templates are current'
```

This ensures teams don't silently fall behind as practices evolve.

---

## Part 4: Best Practices for AI Development Process

Parts 1–3 covered principles and tooling. This part describes the **concrete development process** we encode into our templates — the workflow an AI agent follows when building features. This is where spec-driven development, Red-Green-Refactor, vertical slicing, and human gates come together into a single disciplined loop.

### The Five-Phase Workflow

Every feature follows five phases. The AI agent is responsible for executing them, but the developer stays in control at every gate.

```
Explore → Specify → Plan → Implement → Commit
```

**Explore** — The agent reads the codebase to understand existing patterns, the reference feature, and the problem space. No code changes happen here.

**Specify** — A specification file (`_specs/<FeatureName>.md`) captures user stories, acceptance criteria, data model, and business rules. This is the contract between the developer and the AI.

**Plan** — A plan file (`_plans/<FeatureName>.md`) decomposes the feature into concrete implementation steps. Each step is a vertical behaviour slice with explicit file paths, test methods, and verification criteria. The plan must be approved before any code is written.

**Implement** — The agent executes the plan one step at a time using Red-Green-Refactor. Each step ends at a human gate — the developer reviews and approves before the next step starts.

**Commit** — Small, validated changes are checkpointed in Git after each approved step.

### The Planning Gate: When Plans Are Required

Not every change needs a plan. Our templates encode a simple decision matrix:

| Situation | Plan required? |
|-----------|---------------|
| New feature or vertical slice | **Yes** |
| Change touching 3+ files | **Yes** |
| Risk area change (auth, PII, DB schema, shared contracts) | **Yes** |
| 1–2 file bugfix | No — but still test-first |
| Config correction, simple refactor | No |

The planning gate prevents the most common AI failure mode: the agent starts writing code before understanding the scope, creates files in the wrong locations, misses existing patterns, and produces work that has to be thrown away.

### Vertical Slice Decomposition

This is the most important planning principle and the one most teams get wrong when working with AI.

**The problem with horizontal slicing:** A natural instinct — for both humans and AI — is to plan layer by layer: "Step 1: Create the entity. Step 2: Add the repository. Step 3: Build the controller. Step 4: Create the page." This feels orderly, but it has a critical flaw: **you don't discover integration problems until the very end**. The entity might not match the DTO. The query might return the wrong shape. The page might need data the controller doesn't provide. By the time you find out, you've built four layers of code that need rework.

**The vertical slice alternative:** Decompose features into the smallest possible **user-visible behaviours**, not into layers. Each slice delivers something a user or a test can verify end-to-end.

The strategy is **UI first with mocks, then replace mocks top-to-bottom**:

1. **Stub phase** — Build the page, form, or component at the Presentation layer with a stubbed service returning fake data. The user validates the UI and interaction design immediately. Tests verify the ViewModel behaviour against the mock.

2. **Wire phase** — Replace the stub with real production code, working from top to bottom: controller → application handler → domain → persistence → database. Integration tests verify the full stack. Remove the stub.

This two-phase approach has three benefits:

- **Fail fast** — Integration mismatches between layers surface on the second step, not after building the entire feature in isolation.
- **UI feedback early** — The user sees and validates the interface before any backend work begins, catching UX issues when they're cheap to fix.
- **Smaller blast radius** — If the backend step reveals that the contract needs to change, only one stub step and one wire step are affected — not four independent layers.

### Example: Two Slices for a "Subscriptions" Feature

**Slice A — View subscriptions:**

| Step | What it delivers | Layers touched |
|------|-----------------|----------------|
| 1. Display subscription list (stubbed) | Page renders a list with fake data; user validates layout and columns | Presentation only |
| 2. Wire subscription list to real API | GET endpoint returns real data; integration test passes | Controller, Application, Persistence, DB, Refit client |

**Slice B — Create a subscription:**

| Step | What it delivers | Layers touched |
|------|-----------------|----------------|
| 3. Create subscription form (stubbed) | Form renders with validation; user validates error handling | Presentation only |
| 4. Wire subscription create to real API | POST endpoint creates record; validation errors returned; integration test passes | Controller, Application, Domain, Persistence, DB, Refit client |

Notice: each slice is completed (UI + backend) before the next slice starts. Step 1 and Step 2 are adjacent — they belong to the same user behaviour. The agent never jumps to Slice B before Slice A is fully wired.

**Compare with the anti-pattern:** Step 1 "Create Entity + Repository", Step 2 "Build Application handlers", Step 3 "GET endpoint", Step 4 "POST endpoint", Step 5 "List page", Step 6 "Create page". This builds layers in isolation. The UI is validated last, when it should be validated first.

### Red-Green-Refactor-Proof: The Implementation Loop

Each plan step is executed as a single Red-Green-Refactor cycle. This is the loop the AI agent follows for every step:

```
READ       — Read the plan step. Understand scope, files, and the test to write.
RED        — Write the failing test FIRST. Run it. Confirm it fails.
GREEN      — Write the minimal production code to make the test pass. Run it. Confirm it passes.
REFACTOR   — Clean up if needed. Do not change behaviour.
ANALYSE    — Run static analysis. Fix all violations. Run the full test suite.
PROVE      — Build (zero warnings) + all tests pass + format check passes.
🛑 STOP    — Present results. Wait for human approval.
MARK DONE  — After approval, update the plan file checkboxes.
```

Three rules are non-negotiable:

1. **Never skip RED.** The test must exist and fail before any production code is written. This proves the test actually validates something. An AI that writes the test and production code simultaneously has no guarantee the test would have caught a real failure.

2. **Never batch steps.** One step per interaction. The agent completes a step, proves it, presents results, and stops. The developer reviews before the next step begins. This prevents the common AI failure of compounding errors across multiple steps.

3. **Never proceed past the human gate.** The 🛑 STOP is absolute. The agent does not continue until the developer explicitly approves. This is the single most important control mechanism — it keeps the developer accountable for every change.

### Human Gates: The Developer Stays in Control

The human gate is not a rubber stamp. Each gate requires **behavioural verification** — a concrete assertion that the system now does something it didn't before:

- "Integration test `GetSubscriptionsTest` passes — GET `/api/subscriptions` returns a list."
- "Form renders with validation — entering an empty name shows 'Name is required'."
- "POST `/api/subscriptions` with valid data returns 200; with invalid data returns 400 with error message."

"Code review" alone is never sufficient as the sole verification. If a step cannot be verified through observable behaviour, it should be merged into a step that can.

Gates also trigger at higher-risk moments:
- After plan creation (before any code is written)
- Before any Git push
- On any change to authentication, PII handling, shared contracts, or database schema

### Spec-Driven Development: The Spec as Contract

The specification file (`_specs/<FeatureName>.md`) is written before the plan and serves as the contract between developer intent and AI execution. A good spec contains:

- **User stories** — Who benefits and what they can do
- **Acceptance criteria** — Concrete, testable conditions for "done"
- **Data model** — Entities, properties, relationships
- **Business rules** — Validation, constraints, edge cases
- **Non-goals** — What is explicitly out of scope

The spec prevents a common AI failure: scope creep. Without a spec, the AI infers what "Add subscriptions" means and may add features nobody asked for, use patterns that don't match the codebase, or miss critical business rules. The spec makes intent unambiguous.

The plan is then derived from the spec. Every plan step traces back to an acceptance criterion. If a step doesn't serve an acceptance criterion, it shouldn't exist.

### Progress Tracking Across Sessions

AI coding sessions are ephemeral — context is lost when a session ends. Our plan files solve this with a simple convention: **checkbox tracking**.

```markdown
## Step 1 — Display subscription list (stubbed)
...
**🛑 HUMAN GATE**:
- [x] Behavioral verification: ViewModel test passes, page renders list with fake data
- [x] Code review: Page layout matches reference feature

## Step 2 — Wire subscription list to real API
...
**🛑 HUMAN GATE**:
- [ ] Behavioral verification: Integration test passes, GET /api/subscriptions returns list
- [ ] Code review: Repository, query, controller follow reference patterns
```

When a new session starts, the agent reads the plan file and finds the first unchecked `[ ]` — that's where to resume. No context is lost. Any agent, in any session, can pick up exactly where the last one left off.

### Encoding All of This as AI Instructions

The key insight is that this entire process — planning gates, vertical slicing, Red-Green-Refactor, human gates, progress tracking — is **encoded as AI instruction files** that the agent reads and follows automatically. It's not a convention that developers need to remember; it's a constraint the AI enforces on itself.

| File | What it encodes |
|------|----------------|
| `AGENTS.md` | Planning gate, mandatory workflow rules, RGR-Proof loop, human gates, vertical slice rules |
| `planner.agent.md` | How to decompose features into vertical slices, interview checklist, self-check validation |
| `build-feature/SKILL.md` | How to execute plan steps with layer-specific code templates |
| `copilot-instructions.md` | Critical rules, project structure, dependency matrix, naming conventions |
| `plan-template.md` | Exact format for plan files with stub/wire guidance |

When a developer says "build the Subscriptions feature", the AI reads these files, asks clarifying questions, produces a vertically-sliced plan, waits for approval, and executes one step at a time with test-first discipline and mandatory human gates. The developer's job shifts from writing boilerplate to **reviewing vertical slices of working behaviour**.

---

## Conclusion

AI coding assistants are powerful, but they amplify both good and bad practices. The organisations that benefit most will be those that:

1. **Adopt deliberate workflows** — Explore, Specify, Plan, Implement, Commit
2. **Require verification** — never trust AI output without automated proof
3. **Keep sessions focused** — one task, one goal, clear validation
4. **Choose models wisely** — match cost to complexity
5. **Centralise and share AI guidance** — turn individual insights into organisational standards

The tooling doesn't have to be complex. What matters is the commitment to treating AI coding artefacts — instructions, skills, agents, prompts — as **first-class engineering assets** that deserve the same rigour as your application code: versioned, tested, reviewed, and continuously improved.

---

*This article describes practices encoded in the [copilot-sync](https://github.com/nickaerts/copilot-sync) open-source project, which distributes AI coding instructions as versioned templates across .NET projects.*

> **Next:** [Part 2 — Spec-Driven Development: The Process in Action](BLOGPOST-PROCESS.md)
