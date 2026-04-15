# From Vibe Coding to Spec-Driven Development: A Reusable AI Agent Configuration for .NET Projects

*An open-source library of GitHub Copilot and Claude Code configuration files that encode a disciplined, repeatable engineering process into AI agents, skills, and instructions.*

> **Repository:** [github.com/geobarteam/github-copilot-configs](https://github.com/geobarteam/github-copilot-configs)
>
> **Part 1 of 2.** This article covers the motivation, best practices, and the configuration library. For a hands-on walkthrough of the spec-driven process, see [Part 2: The Process in Action](BLOGPOST-PROCESS.md).

---

## Why I Built This

Over the past year I've spent hundreds of hours working with AI coding agents — GitHub Copilot, Claude Code, and various MCP tool integrations. I learned a lot, made plenty of mistakes, and eventually landed on a workflow that works reliably.

The problem was: every time I started a new project, I was rebuilding the same configuration from scratch. The same planning discipline. The same Red-Green-Refactor loop. The same human gates. The same layer-specific coding conventions. Each project had slight variations, and none of them captured everything I'd learned.

So I built **[github-copilot-configs](https://github.com/geobarteam/github-copilot-configs)** — a reusable template library of AI agent configuration files that I copy into every new .NET project. It encodes the engineering process I've developed over my career — now translated into agents, skills, and instruction files that AI assistants follow automatically.

The goal is twofold:

1. **Quick project setup** — Copy the files, run the `/init` skill, and every AI assistant in the project immediately knows the architecture, conventions, test commands, and development workflow. No more explaining the same things in every chat session.
2. **Consistent engineering process** — The agents enforce plan-first development, test-first implementation, vertical slicing, and mandatory human review gates. This isn't just guidance — it's a process that prevents the most common AI coding failures I've encountered.

This article shares what I've learned and how the configuration library works. Whether you adopt it wholesale or just borrow ideas, I hope it saves you some of the trial and error I went through.

---

## The Problem: Vibe Coding

AI coding assistants like GitHub Copilot are changing how developers write software. But without clear practices, teams drift into what I call **vibe coding** — letting the AI generate code with no plan, no verification, and no accountability. The result? Code nobody fully understands.

---

## Part 1: Best Practices for AI-Assisted Coding

### Understand the Core Concepts

Before diving into workflows, every developer should understand three foundational concepts:

**The Model** — The LLM behind Copilot. Different models have different strengths, costs, and context windows. Choosing the right model for the task is a deliberate decision, not an afterthought.

**The Context** — Everything the model sees when generating a response: your open files, instructions, conversation history, and workspace structure. Context quality directly determines output quality.

**The Agent** — The orchestration layer that can inspect files, reason about tasks, propose plans, make changes, run checks, launch commands, and iterate through workflows. Agents interact with tools including MCP servers, making them far more capable than simple chat completions.

### Treat Tokens as a Limited Resource

Tokens — the unit of model consumption — are finite on any plan. Practical consequences:

- Use **short, focused sessions**
- Keep prompts **specific and bounded**
- Choose the **right model for the task**
- Avoid unnecessary retries in polluted sessions
- Use **lighter models** for simple work
- Reserve **premium models** for harder problems

### Choose the Right Model for the Job

Not every task needs the most powerful model. Here's how I think about model selection:

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

The antidote is **spec-driven development**: start with a specification, produce a plan, implement in small verified steps, and stay in control throughout. That's what this configuration library encodes.

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

In my workflow, every change follows: **RED → GREEN → REFACTOR → ANALYSIS → PROOF → HUMAN GATE**.

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

## Part 2: The Configuration Library — What's Inside

Knowing best practices is one thing. Making every AI session follow them automatically is another. That's what the [github-copilot-configs](https://github.com/geobarteam/github-copilot-configs) library does — it encodes your architecture, conventions, and development process into configuration files that GitHub Copilot and Claude Code follow without you repeating yourself every session.

**Target stack**: ASP.NET Core + Blazor WebAssembly · Onion/Screaming Architecture · CQRS-lite · .NET 10. But the patterns and workflow are adaptable to any .NET project.

### Agents — Specialized AI Roles

Agents are invoked via `@name` in VS Code chat. Each is a specialized role with its own constraints, workflow, and tools.

| Agent | What it does |
|-------|-------------|
| **`@planner`** | Reads `_specs/<Feature>.md` (if it exists) and creates `_plans/<Feature>.md` with vertical-slice steps before any code is written. Interviews you about the feature, reads the reference pattern, decomposes into testable behavior slices. Never writes production code — planning only. |
| **`@bugfix`** | Diagnoses bugs using regression-test-first discipline. Writes a test that reproduces the bug, confirms it fails, then writes the minimal fix. Escalates to `@planner` if the fix spans 3+ files. |
| **`@debug`** | A debug engineer that combines Application Insights log analysis with Playwright browser automation. Queries Azure telemetry for exceptions and traces, reproduces UI issues in a real browser, correlates frontend errors with backend traces, and produces structured debug reports. |
| **`@devops`** | Maintains GitHub Actions CI/CD pipelines and Azure Bicep infrastructure. Knows the exact workflow structure (build → deploy → summary), Bicep module conventions, resource naming patterns, and the Architecture.md/Deployment-Info pattern for tracking infrastructure state. |
| **`@smoke-test`** | Post-deployment smoke testing using browser automation. Discovers environment URLs from infrastructure docs, navigates to health endpoints and pages, checks for console errors and network failures, and produces a structured pass/fail report. |
| **`@git`** | GitFlow branching specialist. Creates feature/bugfix/hotfix branches, manages tags, guides PR workflows. Only runs mutating Git commands after explicit confirmation. |

Each agent follows the shared workflow rules in `AGENTS.md`: plan-first discipline, Red-Green-Refactor loops, and mandatory human gates.

#### The DevOps Agent in Detail

The `@devops` agent deserves special attention because it encodes a pattern I found particularly valuable: the **Architecture.md → Bicep → Deployment-Info pipeline**.

The workflow separates *what should exist* (Architecture.md) from *what does exist* (Deployment-Info-{env}.md):

1. **Architecture.md** is the specification — the desired infrastructure state, including resource topology, SKUs, cost estimates, DNS records, and URL maps
2. **Bicep templates** implement that specification
3. **Deployment-Info-stg.md / Deployment-Info-prd.md** record the actual deployed state — real resource names, FQDNs, and live URLs

This separation means the devops agent always knows the current state of each environment and can reason about what needs to change. It also means the `@smoke-test` and `@debug` agents can discover live URLs automatically — no hardcoded values.

#### The Smoke Test Agent in Detail

The `@smoke-test` agent is designed for post-deployment verification. Instead of relying on hardcoded URLs, it dynamically builds its environment URL table by reading the infrastructure documentation files. It then executes a structured test plan:

- **Health endpoints**: Checks `/health`, `/health/ready`, `/health/live` on every API
- **Page loads**: Navigates to each web application, verifies content renders
- **Console errors**: Captures JavaScript errors and failed resource loads
- **Network errors**: Monitors for 4xx/5xx HTTP responses
- **Authentication flows**: Verifies login redirects appear on protected routes (without entering real credentials)

The agent produces a structured report with pass/fail counts, collected errors, and details on every failure. It's read-only by design — it never submits forms or modifies data, especially in production.

### Skills — Step-by-Step Recipes

Skills are invoked via `/name` in chat and guide the AI through multi-step code generation tasks.

| Skill | Purpose |
|-------|---------|
| **`/init`** | **Start here** — discovers project tokens (`{{SolutionName}}`, `{{TestExePath}}`, etc.) and replaces placeholders across all config files |
| **`/build-feature`** | Implements an approved plan step by step using Red-Green-Refactor |
| **`/add-endpoint`** | Adds a vertical-slice API endpoint (domain → application → controller → tests) |
| **`/add-blazor-page`** | Adds a Blazor page with ViewModel, Refit client, and MudBlazor components |
| **`/add-blazor-module`** | Scaffolds a standalone WASM module with MVVM + HttpClient |
| **`/add-dbup`** | Creates a DbUp migration script (CREATE TABLE, ALTER, seed data) |
| **`/e2e-test`** | Full-stack browser testing with Playwright MCP |
| **`/csharp-coding-standards`** | Reference-only C# coding standards and patterns |
Specs and plans are created as project-level documentation at the repo root:
- **`_specs/<Feature>.md`** — Feature specifications (the contract between developer and AI). Written before planning using `templates/spec-template.md`.
- **`_plans/<Feature>.md`** — Implementation plans with vertical-slice steps, created by `@planner` from the spec.
### Instructions — Auto-Activated Layer Conventions

Instruction files activate automatically when you edit files matching their `applyTo` glob pattern. When you open a domain entity file, the AI silently loads domain conventions. When you edit a controller, it loads BFF patterns.

| Instruction | Activates for |
|-------------|---------------|
| `domain-entity` | `src/Core/Domain/**` |
| `application-layer` | `src/Core/Application/**` |
| `persistence-layer` | `src/Core/Persistence/**` |
| `bff-controller` | `src/Host/Client/**` |
| `blazor-presentation` | `src/Presentation/**` |
| `refit-client` | `src/Presentation/**/ServiceClients/**` |
| `tests` | `src/Test/**` |

This means the AI always knows the conventions for the layer you're working in — without you needing to mention them.

### Root Files — The Process Backbone

| File | Purpose |
|------|---------|
| **`AGENTS.md`** | Shared workflow rules — planning gates, Red-Green-Refactor-Proof loop, human gates, vertical slice decomposition rules |
| **`copilot-instructions.md`** | Project-level Copilot instructions — critical rules, project structure, dependency matrix, naming conventions, verification commands |
| **`CLAUDE.md`** | Claude Code entrypoint — references AGENTS.md, adds Claude-specific rules |

### Token System — Project-Specific Customization

Templates use `{{MustacheStyle}}` placeholders that the `/init` skill replaces with project-specific values:

| Token | Example |
|-------|---------|
| `{{SolutionName}}` | `FindMyDoctor` |
| `{{NamespaceRoot}}` | `Contoso.FindMyDoctor` |
| `{{DbContextName}}` | `FindMyDoctorDbContext` |
| `{{TestExePath}}` | `.\src\Test\Unit\bin\Debug\net10.0\Contoso.FindMyDoctor.Unit.Tests.exe` |

Copy the files, run `/init`, and the entire AI configuration is project-specific in seconds.

---

## Part 3: Getting Started — Setting Up Your Own Project

You don't need to use the exact same stack. The engineering process — plan first, test first, one step at a time — works for any project. Here's how to get started:

### Step 1: Copy the Configuration Files

Clone or download [github-copilot-configs](https://github.com/geobarteam/github-copilot-configs) and copy the relevant files into your project:

```
your-project/
├── .github/
│   └── copilot-instructions.md
├── AGENTS.md
├── CLAUDE.md                      ← optional, for Claude Code users
├── agents/
├── instructions/
├── skills/
├── prompts/
└── templates/
```

### Step 2: Run the Init Skill

Open VS Code chat and type `/init`. The skill will:

1. **Discover** your project's tokens automatically (solution name, namespace root, test executable path, etc.)
2. **Ask** for values that can't be auto-discovered
3. **Replace** all `{{token}}` placeholders across your config files
4. **Verify** no unreplaced tokens remain

### Step 3: Start Building Features

```
# 1. Plan the feature
@planner Add subscription management with list and create

# 2. Review and approve the plan

# 3. Implement step by step
/build-feature Step 1 — Display subscription list
```

### Step 4: Adapt to Your Architecture

These files are templates. After running `/init`, adapt them to your project:

- **Different architecture?** Edit the project structure in `copilot-instructions.md` and update `applyTo` globs in instruction files
- **Different test runner?** Update `{{TestExePath}}` references
- **Additional layers?** Create new `<topic>.instructions.md` files with an `applyTo` glob
- **Additional agents?** Create `<name>.agent.md` in `agents/` following the existing pattern
- **Different CI/CD?** Update the `@devops` agent with your pipeline conventions

### Step 5: Build Your Own Agent Library

The most valuable thing in this repo isn't the boilerplate — it's the **workflow encoded in agents**. Consider building agents for your own recurring tasks:

- A **code review agent** that checks PRs against your team's conventions
- A **migration agent** that handles framework or library upgrades
- A **documentation agent** that keeps API docs in sync with implementation
- A **performance agent** that profiles and suggests optimizations

The pattern is always the same: define the agent's role, constraints, workflow steps, and tools — then let it execute within guardrails you control.

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

Not every change needs a plan. The configuration library encodes a simple decision matrix:

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
| `devops.agent.md` | CI/CD pipeline structure, Bicep conventions, Architecture.md → Deployment-Info pipeline |
| `smoke-test.agent.md` | Post-deployment verification, URL discovery, browser-based health checks |
| `debug.agent.md` | App Insights telemetry analysis, Playwright reproduction, cross-layer correlation |
| `bugfix.agent.md` | Regression-test-first bug fixing, 2-file scope limit, escalation rules |
| `build-feature/SKILL.md` | How to execute plan steps with layer-specific code templates |
| `copilot-instructions.md` | Critical rules, project structure, dependency matrix, naming conventions |

When a developer says "build the Subscriptions feature", the AI reads these files, asks clarifying questions, produces a vertically-sliced plan, waits for approval, and executes one step at a time with test-first discipline and mandatory human gates. The developer's job shifts from writing boilerplate to **reviewing vertical slices of working behaviour**.

---

## Conclusion

AI coding assistants are powerful, but they amplify both good and bad practices. The developers who benefit most will be those who:

1. **Adopt deliberate workflows** — Explore, Specify, Plan, Implement, Commit
2. **Require verification** — never trust AI output without automated proof
3. **Keep sessions focused** — one task, one goal, clear validation
4. **Choose models wisely** — match cost to complexity
5. **Encode their process into AI configuration** — turn personal engineering discipline into agents that enforce it automatically

I built [github-copilot-configs](https://github.com/geobarteam/github-copilot-configs) because I got tired of re-explaining my process to every AI session in every project. Now I copy the files, run `/init`, and every agent — planner, bugfix, debug, devops, smoke-test, git — knows the architecture, the conventions, and the workflow. The AI becomes a disciplined teammate instead of an unpredictable code generator.

The configuration library is open source and designed to be forked and adapted. Take what works for you, modify what doesn't, and build your own agent library on top of it. The investment in encoding your engineering process pays for itself on the first feature you build with it.

---

*The practices described in this article are encoded in the [github-copilot-configs](https://github.com/geobarteam/github-copilot-configs) open-source repository — a reusable template library of GitHub Copilot and Claude Code configuration files for .NET projects.*

> **Next:** [Part 2 — Spec-Driven Development: The Process in Action](BLOGPOST-PROCESS.md)
