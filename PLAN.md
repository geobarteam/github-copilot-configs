# Review Plan — Template Library Audit

Findings from a full review of all files. Grouped by priority.

---

## HIGH — Hardcoded Values & Broken References

### H1. `debug.agent.md` — Entirely project-specific

The file is hardcoded to "MySalon25" with real Azure subscription IDs, resource group names, URLs, domains, cookie names, and auth flows. None of it is tokenized.

**Hardcoded items** (non-exhaustive):
- Line 8: `MySalon25` → should be `{{SolutionName}}`
- Lines 42-47: All URLs (`mysalon25.com`, `salon25-dev-bo-api.azurewebsites.net`, etc.)
- Lines 56-64: Azure resource names (`salon25-dev-rg`, `appi-salon25-dev`, etc.)
- Lines 56-64: **Azure subscription IDs** — these are secrets, must be removed
- Lines 60-64: SQL Server names (`salon25-dev-sql`)
- Lines 100-180: All KQL queries hardcode `appi-salon25-{env}` and `salon25-{env}-rg`
- Lines 250-316: Auth playbook hardcodes `ciamlogin.com`, cookie name `Salon25.Customer`, localStorage key `salon25_customer_auth`
- Lines 340-370: Known issues table references project-specific patterns

**Fix**: Rewrite the entire file as a generic template:
- Replace all resource names with `<your-component-name>` tokens or placeholder patterns
- Replace URLs with `<your-url>` placeholders or tokenize
- Remove subscription IDs entirely — add a comment "fill in from Azure portal"
- Make the Environment Reference section a fillable template with instructions
- KQL queries should use `<your application insight name>` and `<your-resource-group>`
- Auth playbook should be generic (cookie name, localStorage key as configurable)
- Add `/init` integration: the init skill should be able to populate the Environment Reference

### H2. `copilot-instructions.md` (root) — `dotnet test` contradicts all other files

The root `copilot-instructions.md` uses `dotnet test` in two places:
- **Line 21** (Critical Rules §1): `dotnet test --project src/Test/Unit/`
- **Lines 225-227** (Commands section): `dotnet test --project src/Test/Unit/`, `dotnet test --project src/Test/Integration/`, `dotnet test --project src/Test/Unit/ --filter "<Class>"`

Every other file says `dotnet test` is NOT supported and uses `{{TestExePath}}` instead:
- `AGENTS.md` line 72: "`dotnet test` is not supported"
- `tests.instructions.md` line 128: "`dotnet test` is not supported"
- `build-feature/SKILL.md` line 48: "`dotnet test` is not supported"
- `sonar-review.agent.md` line 75: "DO NOT run `dotnet test`"

**Fix**: The root `copilot-instructions.md` appears to be a **different variant** (possibly for a project NOT using Microsoft.Testing.Platform). Two options:
- **Option A**: This is the Claude Code variant and should also use `{{TestExePath}}`. Replace `dotnet test` with `{{TestExePath}}` and add the MTP note.
- **Option B**: This variant supports `dotnet test` for projects that don't use MTP. If so, introduce a `{{TestCommand}}` token that the `/init` skill sets to either `dotnet test --project src/Test/Unit/` or the `.exe` path based on detection.

Recommend **Option A** for consistency.

### H3. `e2e-test/SKILL.md` — Hardcoded localhost ports

Lines 23-26 hardcode specific ports (`5001`, `7094`, `7259`, `10000-10002`). These are project-specific (`launchSettings.json` values).
| **Client** | `https://localhost:{{ClientPort}}` | Blazor Client Frontend |
| **API** | `https://localhost:{{ApiPort}}` | Your backend API |
```
Or add a note: "Ports are read from `Properties/launchSettings.json` — verify before running."

### H4. `BLOGPOST-PROCESS.md` — Uses `dotnet test` throughout

Lines 263, 298-299, 329, 612 all use `dotnet test` instead of `{{TestExePath}}`.

**Fix**: Replace with `{{TestExePath}} --filter "..."` to match the template convention, or add a note that blog posts show both variants for audience accessibility.

### H5. Missing instruction files referenced in `COPILOT-GUIDE.md`

These files are listed in the guide (lines 43-47, 331-335, 450-456) but don't exist on disk:
- `instructions/application-layer.instructions.md` — referenced for `src/Core/Application/**`
- `instructions/bff-controller.instructions.md` — referenced for `src/Host/BFF/**`
- `instructions/nservicebus.instructions.md` — referenced for NServiceBus handlers/contracts
- `instructions/blazor-presentation.instructions.md` — referenced for Presentation pages/ViewModels

**Fix**: Create these four files. They are part of the template library and should contain the patterns already documented in skills/instructions. Alternatively, remove them from COPILOT-GUIDE.md if they are intentionally deferred.

---

## MEDIUM — Inconsistencies Between Files

### M1. DI registration pattern — `*Module` vs `ServiceCollectionExtensions`

- `copilot-instructions.md` line ~141: references `ServiceCollectionExtensions.Add{{SolutionName}}()`
- `copilot-instructions.md` Critical Rule 3: "DI via `*Module` + Scrutor suffix scanning only"
- `add-endpoint/SKILL.md`: references `*Module` classes

These two patterns (`*Module` vs `ServiceCollectionExtensions`) are in conflict.

**Fix**: Pick one and standardize across all files. Since Critical Rule 3 says `*Module`, the `ServiceCollectionExtensions` reference should be updated or removed.

### M2. Refit client registration — two competing patterns

- `refit-client.instructions.md` line 44: `BffServiceClients.AddBffServiceClients()` with `AddRefitClientWithCookies<T>()`
- `add-endpoint/SKILL.md` line 120: `BffClientConfigurator.ConfigureBffServiceClient()`

**Fix**: Consolidate to one canonical pattern. If both exist in different projects, document when to use which.

### M3. `copilot-instructions.md` (root) — Missing rules present in other files

The root variant has a different Critical Rules section than what other files expect:
- Rule 2 in root: "All public types require XML docs" — not mentioned in other files
- Rule 3 in root: "Versioning via GitVersion" — not in other files
- Rule 4 in root: "Minimize external dependencies" — not in other files
- Missing from root: "DI via `*Module` + Scrutor" (present in other files)
- Missing from root: "`AsNoTracking()` on all reads" (present in other files)
- Missing from root: "Match existing patterns" (present in other files)

**Fix**: Harmonize the critical rules between the two `copilot-instructions.md` variants. The root (Claude Code) variant should share the same core rules as the `.github/` (VS Code) variant, or differences should be intentional and documented.

### M4. `{{CompanyName}}` token underused

Only appears in `build-feature/SKILL.md` line 418 (copyright header). Should also appear in:
- Any copyright header templates shown in other skills
- `copilot-instructions.md` if it has a copyright convention section

**Fix**: Audit all copyright header references and ensure they all use `{{CompanyName}}`.

### M5. `csharp-coding-standards` skill — `invocable: false` but still a full skill

`skills/csharp-coding-standards/SKILL.md` has `invocable: false` but contains 4 sub-files:
- `anti-patterns-and-reflection.md`
- `composition-and-error-handling.md`
- `performance-and-api-design.md`
- `value-objects-and-patterns.md`

Not listed in the repo structure in `.github/copilot-instructions.md` or `COPILOT-GUIDE.md`.

**Fix**: Either (a) make it invocable and document it, (b) move to `instructions/` as a scoped instruction, or (c) add it to the repo structure docs and explain it's a reference-only skill.

### M6. `add-blazor-module/SKILL.md` vs `add-blazor-page/SKILL.md` — overlapping scope

Both skills create Blazor UI components but for different architectures:
- `add-blazor-page` → Presentation RCL with BFF/Refit (Blazor Server pattern)
- `add-blazor-module` → WASM client with direct `HttpClient` services

The distinction is documented in `add-blazor-module` but not in `add-blazor-page`.

**Fix**: Add a scope note to `add-blazor-page/SKILL.md` clarifying it's for Blazor Server + BFF, and reference `add-blazor-module` as the WASM alternative.

### M7. `e2e-test/SKILL.md` — References MCP tool names directly

Lines throughout reference `mcp_microsoft_pla_browser_*` tool names, which are VS Code extension-specific identifiers that may change.

**Fix**: Keep as-is (these are the actual tool names needed), but add a note: "These tool names require the Playwright MCP extension. If tool names change after an extension update, update this skill."

### M8. Workflow duplication across files

The Red-Green-Refactor-Proof loop is fully defined in:
1. `AGENTS.md` (lines 36-57) — canonical definition
2. `copilot-instructions.md` (root, Mandatory Workflow section)
3. `build-feature/SKILL.md` (lines 36-50)
4. `COPILOT-GUIDE.md` (section 4)

**Fix**: `AGENTS.md` is the source of truth. Other files should reference it rather than duplicating:
- `copilot-instructions.md`: keep a summary, add "see AGENTS.md for full loop"
- `build-feature/SKILL.md`: reference `@AGENTS.md` for the full loop
- `COPILOT-GUIDE.md`: keep for reader education (it's a guide), but note the canonical source

---

## MEDIUM — Token & Init Skill Gaps

### T1. E2E port numbers not tokenizable

The e2e-test skill hardcodes localhost ports. The `/init` skill should detect these from `launchSettings.json` and replace them, or the skill should read ports at runtime.

**Fix**: Add port discovery to `/init` skill (read `Properties/launchSettings.json` for each host project). Introduce tokens `{{StsPort}}`, `{{BffPort}}`, `{{WfePort}}` or alternatively instruct the skill to read ports from launchSettings at runtime.

### T2. `/init` skill doesn't cover `debug.agent.md` tokens

The init skill's Phase 4 lists target files but doesn't mention environment-specific tokens in `debug.agent.md` (Azure resource names, subscription IDs, URLs).

**Fix**: Either (a) add a Phase in `/init` for environment setup (Azure resources, URLs), or (b) mark `debug.agent.md` as a project-specific file that needs manual customization after init, with a checklist.

### T3. No token for test project path structure

`{{TestExePath}}` assumes a specific path (`src/Test/Unit/bin/Debug/net10.0/...`). If a project has tests at a different path, the token works but the surrounding text doesn't.

**Fix**: The `/init` skill correctly discovers this. No change needed, but add a note in `tests.instructions.md` that the path is project-specific and set by `/init`.

---

## LOW — Blog Post Improvements

### B1. `BLOGPOST.md`

1. **Missing conclusion**: Part 2 ends abruptly. Add a "What's Next" section pointing to Part 3 (process) or a call-to-action.
2. **copilot-sync link**: Line 541 links to `https://github.com/nickaerts/copilot-sync` — verify this is the intended public URL or replace with a generic placeholder if the repo isn't public.
3. **Token table example**: Lines 193-198 are good but should note that `{{SonarServerUrl}}` example `https://sonarqube.example.com` is intentionally a placeholder (not a real URL). This is fine.
4. **Navigation**: No "Part 1 → Part 2" links between the documents.
5. **Length**: At ~540 lines, could benefit from a table of contents at the top.

### B2. `BLOGPOST-PROCESS.md`

1. **`dotnet test` usage**: Lines 263, 298-299, 329, 612 use `dotnet test`. Since this is a public-facing blog post (not a template consumed by AI), this may be intentional for readability. If so, add a note: "Your project may use a different test runner — adapt the commands accordingly." If it should match the template convention, replace with `{{TestExePath}}`.
2. **"Vibe Coding" framing** (Part 1): The dismissive framing ("developer glances at output, hits Enter, and calls it done") could alienate readers. Suggest reframing as: "Even strong engineers lose architectural coherence without structured AI workflows."
3. **Missing limitations section**: No discussion of when AI-driven dev fails or needs human override.
4. **Spec example length**: The JSON/table spec example (~150 lines) is verbose for a blog. Consider a "minimal spec" first, then the full example as an appendix.
5. **Vertical slicing justification**: The "why vertical not horizontal" section states the rule but doesn't give a concrete failure scenario. Add: "If you build 5 layers of backend before testing the UI, you discover at step 6 that the DTO shape is wrong."
6. **copilot-sync link**: Line 749 same as BLOGPOST.md — verify URL.
7. **Navigation**: Same as BLOGPOST.md — add inter-document links.

---

## LOW — Structural / Quality Improvements

### S1. No central cross-reference index

No single file lists all agents, skills, instructions, and their relationships.

**Fix**: Add a "Reference Index" section to `COPILOT-GUIDE.md`:
```
| Agent/Skill | Trigger | Related Instructions | Related Skills |
|-------------|---------|---------------------|----------------|
| @planner | @planner in chat | — | /build-feature |
| @debug | @debug in chat | — | /e2e-test |
| /build-feature | /build-feature in chat | all instructions | @planner |
```

### S2. Instruction files missing `applyTo` frontmatter

The existing instruction files (`domain-entity`, `persistence-layer`, `refit-client`, `tests`) should have YAML frontmatter with `applyTo` glob patterns, but only some do.

**Fix**: Verify all instruction files have correct `applyTo` patterns matching what `COPILOT-GUIDE.md` documents.

### S3. `.github/copilot-instructions.md` repo structure is outdated

The structure listed in the file doesn't include:
- `skills/init/`
- `skills/add-blazor-module/`
- `skills/csharp-coding-standards/`
- `agents/debug.agent.md`
- `agents/bugfix.agent.md`

**Fix**: Update the repo structure tree to reflect all current files.

### S4. `CLAUDE.md` is minimal

Only 6 lines. Could reference more shared context or clarify how it differs from the VS Code variant.

**Fix**: Consider expanding to reference key skills and agents available in Claude Code mode.

---

## Execution Order

If implementing these fixes:

1. **H1** — Rewrite `debug.agent.md` (biggest hardcoded-value offender)
2. **H2** — Fix `dotnet test` in root `copilot-instructions.md`
3. **H5** — Create 4 missing instruction files
4. **H3** — Tokenize e2e-test ports or add launchSettings note
5. **M1-M3** — Resolve DI/Refit pattern conflicts
6. **H4** — Fix `dotnet test` in BLOGPOST-PROCESS
7. **M5-M8** — Structural consistency fixes
8. **S3** — Update repo structure
9. **T1-T2** — Update `/init` skill for new tokens
10. **B1-B2** — Blog post improvements
