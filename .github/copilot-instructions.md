# Copilot Instructions — github-copilot-configs

You are the coding assistant for a **template library** of GitHub Copilot and Claude Code configuration files. This repo contains reusable instructions, agents, prompts, skills, and templates that are copied into new .NET projects.

<context>
This is NOT a .NET solution — it is a library of markdown-based configuration files for AI-assisted development workflows.
Target projects: Blazor Server + BFF + Worker solutions using Onion/Screaming Architecture, CQRS-lite, .NET 10, MSTest v4, StyleCop.
Code examples use generic names like `MyApp` — adapt them to your project when copying.
</context>

---

## Repo Purpose

This repo is a centralized library of AI coding assistant configurations. Copy these files into new projects to get a well-configured GitHub Copilot and Claude Code setup out of the box.

**What lives here:**
- `copilot-instructions.md` — always-on instructions (Claude Code variant at root, VS Code variant in `.github/`)
- `instructions/` — scoped instructions auto-loaded by file pattern (e.g. `tests.instructions.md` for test files)
- `agents/` — on-demand agents (`@planner`, `@code-analysis`, `@git`, `@sonar-review`)
- `skills/` — on-demand skills (`/build-feature`, `/add-endpoint`, `/add-blazor-page`, etc.)
- `prompts/` — reusable prompt templates
- `templates/` — spec templates and other reusable markdown
- `AGENTS.md` — shared agent workflow rules (plan-first, red-green-refactor, human gates)
- `CLAUDE.md` — Claude Code entrypoint
- `COPILOT-GUIDE.md` — developer guide explaining how to use the customizations

---

## Repo Structure

```
/
├── .github/
│   └── copilot-instructions.md   # This file (VS Code variant)
├── copilot-instructions.md       # Claude Code variant (root)
├── AGENTS.md                     # Agent workflow rules
├── CLAUDE.md                     # Claude Code entrypoint
├── COPILOT-GUIDE.md              # Developer best practice guide
├── agents/                       # Agent definitions (.agent.md)
├── instructions/                 # Scoped instructions (.instructions.md)
├── skills/                       # Skill definitions (SKILL.md per folder)
├── prompts/                      # Prompt templates (.prompt.md)
└── templates/                    # Reusable templates (spec-template.md, etc.)
```

---

## Rules for Editing Files in This Repo

1. **Use generic example names.** Code examples use `MyApp`, `AppDbContext`, etc. — never hardcode specific values.
2. **Keep files self-contained.** Each instruction/agent/skill file should work independently when copied into a target project.
3. **Match the existing style.** Read similar files before creating or editing — mirror their structure, tone, and XML tag usage.
4. **Optimize for Claude Sonnet 4.6.** Use XML tags for critical sections, markdown headers for structure, concise language, one example per pattern. Avoid `CRITICAL` / `MUST` / emoji emphasis — Sonnet follows normal instructions well.
5. **Two variants of `copilot-instructions.md` exist.** The root version targets Claude Code; the `.github/` version targets VS Code. Keep them aligned in content but adapted to their respective tools.
6. **No code files.** This repo contains only markdown. Do not add `.cs`, `.json`, `.csproj`, or other source files.
7. **Agents must follow the workflow in `AGENTS.md`.** Plan-first, red-green-refactor, human gates.

---

## File Type Conventions

| Type | Location | Naming | Purpose |
|------|----------|--------|---------|
| Instructions | `instructions/` | `<topic>.instructions.md` | Auto-loaded by `applyTo` glob pattern |
| Agents | `agents/` | `<name>.agent.md` | Invoked via `@name` in chat |
| Skills | `skills/<name>/` | `SKILL.md` | Invoked via `/name` in chat |
| Prompts | `prompts/` | `<name>.prompt.md` | Selected from prompt picker |
| Templates | `templates/` | `<name>.md` | Referenced by agents/skills |
