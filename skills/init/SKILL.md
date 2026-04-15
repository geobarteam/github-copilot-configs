---
name: init
description: "Initialize or re-initialize Copilot configuration files for the current .NET solution. Analyzes the workspace to discover solution name, namespace root, DbContext, test exe path, architecture type (Blazor WASM or Server), and other project-specific values. Replaces all {{token}} placeholders in .github/ files with discovered values. Run once after importing template files, or re-run to update after structural changes."
---

# Init — Solution Discovery & Token Replacement

Analyze the current .NET solution and replace `{{token}}` placeholders in all `.github/` configuration files with project-specific values.

## When to Use

- **First time**: after copying template files into `.github/` (or after `copilot-sync init`)
- **Re-init**: after renaming projects, changing architecture, or adding a test project
- **Audit**: to verify all tokens are replaced (search for remaining `{{` in `.github/`)

---

## Token Inventory

| Token | Description | Auto-discoverable? |
|-------|-------------|-------------------|
| `{{SolutionName}}` | Name of the `.sln` file (without extension) | Yes |
| `{{NamespaceRoot}}` | Root namespace prefix (e.g., `Contoso.MyApp`) | Yes |
| `{{DbContextName}}` | EF Core `DbContext` subclass name | Yes |
| `{{TestExePath}}` | Relative path to test `.exe` (MTP) | Yes |
| `{{CompanyName}}` | Company name for copyright headers | Partial — detect or ask |
| `{{CopilotSyncToolName}}` | dotnet tool name for copilot-sync | No — ask user |
| `{{ClientPort}}` | Client localhost port (e.g., `5001`) | Yes — from launchSettings.json |


---

## Workflow

### Phase 1 — Discovery

Run these searches to auto-detect token values. Do them in parallel where possible.

#### 1. Solution Name (`{{SolutionName}}`)

```
Search for *.sln files under src/ and repo root.
```

- If exactly **one** `.sln` → use its filename without extension.
- If multiple → ask the user which one is the primary solution.
- If none found → ask the user for the solution name.

#### 2. Namespace Root (`{{NamespaceRoot}}`)

```
Grep for <RootNamespace> in *.csproj files under src/.
If not set, derive from project filenames — take the longest common prefix.
```

- Look at `.csproj` filenames: e.g., `Contoso.MyApp.Core.Domain.csproj` → root is `Contoso.MyApp`.
- Cross-check with `namespace` declarations in a few `.cs` files in `Core/Domain/`.
- The namespace root is the common prefix before `.Core.`, `.Host.`, `.Presentation.`, `.Contracts.`, `.Test.`.

#### 3. Architecture Type (Blazor WASM)

```
Search *.csproj files for package references:
  - "Microsoft.AspNetCore.Components.WebAssembly" → Client
```
- **Client**: Blazor Client (typically `Host/Client/`).
- **Presentation**: Contains all presentation files like razor pages, viewmodels, and service clients (typically `Presentation/`).
- This determines which `copilot-instructions.md` context block to use (the `<context>` section).
- If both patterns are found (hybrid), ask the user which is primary.

#### 4. DbContext Name (`{{DbContextName}}`)

```
Grep for ": DbContext" or ": IdentityDbContext" in *.cs files under src/.
```

- Take the class name that inherits from `DbContext`.
- If multiple DbContexts exist, pick the one in `Core/Persistence/` or ask.

#### 5. Test Exe Path (`{{TestExePath}}`)

```
Find test projects: search for *.csproj containing <IsTestProject>true</IsTestProject>
or referencing MSTest/xUnit/NUnit packages.
```

- Derive the exe path: `.\src\Test\Unit\bin\Debug\net10.0\<ProjectName>.exe`
- Check `<TargetFramework>` in the test `.csproj` for the correct TFM (e.g., `net10.0`).
- Prefer the **unit test** project if multiple test projects exist.
- If the test project path differs from `src/Test/Unit/`, adjust accordingly.

#### 6. Company Name (`{{CompanyName}}`)

```
Grep for <Company> in *.csproj or Directory.Build.props.
Also check for "copyright" or "company" in existing .cs file headers.
```

- If found → use it.
- If not → ask the user.

#### 7. Localhost Ports (`{{ClientPort}}`)

```
Read Properties/launchSettings.json from each host project:
  - src/STS/**/launchSettings.json → StsPort
  - src/Host/Client/**/launchSettings.json → ClientPort
```

- Look for `applicationUrl` in the `https` or project profile.
- Extract the port from the URL (e.g., `https://localhost:7094` → `7094`).
- If no launchSettings found for a host, ask the user or leave the token.

### Phase 2 — User Confirmation & Missing Values

Present all discovered values in a summary table and ask the user to confirm or correct them. Then ask for values that could not be auto-detected:

**Present:**
```
Discovered values:
  SolutionName:  MyApp
  NamespaceRoot: Contoso.MyApp
  DbContextName: MyAppDbContext
  TestExePath:   .\src\Test\Unit\bin\Debug\net10.0\Contoso.MyApp.Unit.Tests.exe
  Architecture:  Blazor Server
  CompanyName:   Contoso
  ClientPort:       7259

Are these correct? (Adjust any values before proceeding.)
```

**Ask for missing values:**

- `CopilotSyncToolName` — "What is the copilot-sync dotnet tool package name? (Leave blank if not using copilot-sync)"

### Phase 3 — Architecture Adaptation

Based on the detected architecture, update the `<context>` block in `copilot-instructions.md`:



**Blazor WASM**:
```
Blazor WASM solution: WFE (Blazor WebAssembly) · BFF (Web API) · Worker · DbUp (SQL Server).
```

Also update the `refit-client.instructions.md` scope note if architecture affects it.

### Phase 4 — Token Replacement

Replace all `{{token}}` placeholders in every file under `.github/`:

```
Target files (recursive):
  .github/copilot-instructions.md
  .github/agents/*.agent.md
  .github/instructions/*.instructions.md
  .github/skills/*/SKILL.md
  .github/prompts/*.prompt.md
  .github/AGENTS.md
  .github/CLAUDE.md
```

Also replace in root-level files if present:
```
  COPILOT-GUIDE.md
  copilot-instructions.md  (root Claude Code variant)
  AGENTS.md                (root)
  CLAUDE.md                (root)
```

**Replacement rules:**
- Replace each `{{TokenName}}` with the confirmed value using exact literal replacement.
- Process files one at a time. Read → replace all tokens → write.
- Do **not** modify files outside `.github/` and the root config files listed above.
- Do **not** replace `{{TokenName}}` patterns that appear inside code fences as **examples of the token syntax itself** (e.g., in a table documenting the tokens). Use judgment — if the surrounding context describes the token system, leave it as-is.

### Phase 5 — Verification

After all replacements:

1. Search for remaining `{{` in all modified files:
   ```
   Grep for "\{\{" in .github/ and root config files.
   ```
2. Report any remaining unreplaced tokens (expected: only tokens the user chose to skip).
3. Confirm the architecture-specific sections are consistent.

### Phase 6 — Create `.copilotrc.json` (if not present)

If no `.copilotrc.json` exists at the repo root, create one to record the token values:

```json
{
  "version": "1.0.0",
  "tokens": {
    "SolutionName": "<value>",
    "NamespaceRoot": "<value>",
    "DbContextName": "<value>",
    "TestExePath": "<value>",
    "CompanyName": "<value>",
    "CopilotSyncToolName": "<value or empty>",
    "StsPort": "<value>",
    "BffPort": "<value>",
    "WfePort": "<value>"
  }
}
```

If `.copilotrc.json` already exists, update the `tokens` section with the new values.

> **Note on `debug.agent.md`:** The debug agent's Environment Reference section (Azure resource names, subscription IDs, URLs) is NOT populated by `/init`. These are environment-specific values that require manual setup. After running `/init`, review `agents/debug.agent.md` and fill in the `<your-...>` placeholders with your project’s Azure resource details.

---

## Re-init Behavior

When running `/init` on a project that has already been initialized (tokens already replaced):

1. Read existing `.copilotrc.json` for current values.
2. Re-run discovery to detect any changes (renamed solution, new test project, etc.).
3. Show a diff of old vs. new values.
4. Only replace values that changed — do not re-process files where nothing changed.
5. Update `.copilotrc.json` with new values.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| No `.sln` file found | Ask user for solution name; they may use a different build system |
| Multiple DbContexts | Ask which is the primary one used by the BFF/API |
| Test exe path wrong | User can correct it; verify by running the exe path |
| Token still in file after init | Check if it's inside a documentation example (intentional) or a missed replacement |
| Architecture unclear | Ask user directly — some projects use both Server and WASM |
