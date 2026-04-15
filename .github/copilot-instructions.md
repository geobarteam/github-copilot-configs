# Copilot Instructions — github-copilot-configs

You are the coding assistant for a **template library** of GitHub Copilot and Claude Code configuration files. This repo contains reusable instructions, agents, prompts, skills, and templates that are copied into new .NET projects.

<context>
This is NOT a .NET solution — it is a library of markdown-based configuration files for AI-assisted development workflows.
Target projects: ASP.NET + Blazor + Worker solutions using Onion/Screaming Architecture, CQRS-lite, .NET 10.
</context>


## Repo Structure

```
/
├── .github/
│   └── copilot-instructions.md   # This file (VS Code variant)
├── copilot-instructions.md       # Claude Code variant (root) — NuGet library archetype
├── AGENTS.md                     # Agent workflow rules (plan-first, RGR, human gates)
├── CLAUDE.md                     # Claude Code entrypoint
├── COPILOT-GUIDE.md              # Developer best practice guide
├── BLOGPOST.md                   # Blog post: spec-driven development
├── BLOGPOST-PROCESS.md           # Blog post: the process in action
├── agents/
│   ├── bugfix.agent.md           # Bugfix with regression test
│   ├── code-analysis.agent.md    # StyleCop/Roslyn violation fixer
│   ├── debug.agent.md            # Debug engineer (App Insights + Playwright)
│   ├── git.agent.md              # GitFlow branching + .gitignore
│   ├── planner.agent.md          # Feature plan creator
├── instructions/
│   ├── application-layer.instructions.md   # src/Core/Application/**
│   ├── bff-controller.instructions.md      # src/Host/BFF/**
│   ├── blazor-presentation.instructions.md # src/Presentation/**
│   ├── domain-entity.instructions.md       # src/Core/Domain/**
│   ├── persistence-layer.instructions.md   # src/Core/Persistence/**
│   ├── refit-client.instructions.md        # Refit service clients (WASM)
│   └── tests.instructions.md              # src/Test/**
├── skills/
│   ├── add-blazor-module/        # Add Blazor WASM module (MVVM + HttpClient)
│   ├── add-blazor-page/          # Add Blazor Server page (ViewModel + Refit)
│   ├── add-dbup/                 # Add DbUp migration script
│   ├── add-endpoint/             # Add API endpoint (vertical slice)
│   ├── build-feature/            # Full feature build (RGR per step)
│   ├── csharp-coding-standards/  # Reference-only (invocable: false)
│   ├── e2e-test/                 # Full-stack browser testing (Playwright MCP)
│   ├── fix-violations/           # Fix StyleCop/Roslyn violations
│   └── init/                     # Bootstrap workspace tokens + instructions
├── prompts/
│   └── new-feature.prompt.md     # New feature spec + plan kickoff
└── templates/
    └── spec-template.md          # Feature specification template
```

---

## Rules for Editing Files in This Repo

1. **Keep files self-contained.** Each instruction/agent/skill file should work independently when copied into a target project.
2. **Match the existing style.** Read similar files before creating or editing — mirror their structure, tone, and XML tag usage.
3. **Optimize for Claude Sonnet 4.6. and Claude Opus 4.6.** Use XML tags for critical sections, markdown headers for structure, concise language, one example per pattern. Avoid `CRITICAL` / `MUST` / emoji emphasis — Sonnet follows normal instructions well.
4. **Agents must follow the workflow in `AGENTS.md`.** Plan-first, red-green-refactor, human gates.

---



## File Type Conventions

| Type | Location | Naming | Purpose |
|------|----------|--------|---------|
| Instructions | `instructions/` | `<topic>.instructions.md` | Auto-loaded by `applyTo` glob pattern |
| Agents | `agents/` | `<name>.agent.md` | Invoked via `@name` in chat |
| Skills | `skills/<name>/` | `SKILL.md` | Invoked via `/name` in chat |
| Prompts | `prompts/` | `<name>.prompt.md` | Selected from prompt picker |
| Templates | `templates/` | `<name>.md` | Referenced by agents/skills |
