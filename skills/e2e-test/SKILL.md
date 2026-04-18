---
name: e2e-test
description: "Use when running end-to-end browser tests against the full {{SolutionName}} stack (STS + BFF + Client + Azurite). Provides the exact launch sequence, port map, health-check procedure, Playwright MCP navigation and assertion patterns, and teardown steps. Use for: smoke-testing a feature in the browser, verifying a page renders after implementation, checking auth flows, validating empty/populated states."
argument-hint: "Describe the scenario, e.g. 'verify /new-appointment page loads and shows doctor dropdown'"
---

# E2E Test — Full-Stack Browser Testing Skill

Run end-to-end browser tests against the running {{SolutionName}} stack using the **Playwright MCP** tools.

## Prerequisites

- **Azurite** must be installed (`npm install -g azurite`)
- **Playwright MCP** tools must be available (the `mcp_microsoft_pla_browser_*` tool family). These tool names are specific to the Playwright MCP extension — if tool names change after an extension update, update this skill.
- Solution must build successfully before launching

---

## 1 — Port Map

> Ports are project-specific — read from `Properties/launchSettings.json` for each host project. The `/init` skill sets  `{{ClientPort}}`.

| Service | URL | Purpose |
|---------|-----|---------|
| **Client** | `https://localhost:{{ClientPort}}` | Client + Swagger |

---

## 2 — Launch Sequence

Start services in this exact order. Each step must succeed before the next.

### Step 2.2 — Build all hosts

Use the VS Code task `build-all-hosts` (builds STS, BFF, Client in parallel):

```
run_task: build-all-hosts
```

Or via terminal:

```powershell
dotnet build src/{{SolutionName}}.sln
```

### Step 2.5 — Client

```powershell
dotnet run --project src/Host/Web/{{NamespaceRoot}}.Host.Client.csproj
```

Run as **background**. Wait for the log line `Now listening on: https://localhost:{{ClientPort}}`.

**Health check**: Navigate to `https://localhost:{{ClientPort}}` — expect a redirect to STS login or the home page.


---

## 4 — Playwright MCP Patterns

### Navigate to a page

```
mcp_microsoft_pla_browser_navigate → url: "https://localhost:{{ClientPort}}/new-appointment"
```

### Wait for content to load

```
mcp_microsoft_pla_browser_wait_for → text: "Select a doctor"
```

### Take a snapshot (inspect DOM state)

```
mcp_microsoft_pla_browser_snapshot
```

Use snapshots to verify:
- Page title / headings rendered
- Table rows present or absent
- Empty-state messages displayed
- Error messages shown

### Click an element

```
mcp_microsoft_pla_browser_click → element: "Submit"
```

### Fill a form field

```
mcp_microsoft_pla_browser_fill_form → element: "INS", value: "01.02.03-456.78"
```

### Take a screenshot (visual proof)

```
mcp_microsoft_pla_browser_take_screenshot
```

### Check console for errors

```
mcp_microsoft_pla_browser_console_messages
```

Always check console messages after navigation — JS/Blazor errors will appear here.

### Inspect network requests

```
mcp_microsoft_pla_browser_network_requests
```

Shows all HTTP calls made by the browser (API calls to the BFF, static assets, OIDC token requests). Useful for spotting failed API calls (4xx/5xx).

### Check server-side logs

Each service launched as a background terminal returns a **terminal ID**. Use `get_terminal_output` with that ID to inspect server-side logs at any time during the test:

```
get_terminal_output → id: "<terminal-id-from-launch-step>"
```

**Tip:** After navigating to a page, check both browser console (`mcp_microsoft_pla_browser_console_messages`) **and** server logs (`get_terminal_output` for BFF/Client) to catch errors on both sides.

To search for specific patterns in large log output, pipe through `Select-String`:

```powershell
# Save terminal output to a variable and search for errors
get_terminal_output → id: "<bff-terminal-id>"
# Then in terminal:
Select-String -Pattern "\[ERR\]|\[WRN\]|Exception" -InputObject $logContent
```

---

## 5 — E2E Test Workflow

```
1. LAUNCH    — Start Azurite → Build → STS → BFF → Client (Steps 2.1–2.5)
2. NAVIGATE  — Open the target page in the browser
3. AUTH      — Handle OIDC redirect if needed (STS auto-issues token)
4. ASSERT    — Use snapshots and waits to verify expected content
5. CONSOLE   — Check browser console for errors
6. SCREENSHOT — Capture visual proof if needed
7. REPORT    — Summarize pass/fail to the user
```

### Example: Verify New Appointment page loads with doctor dropdown

```
1. mcp_microsoft_pla_browser_navigate → "https://localhost:{{ClientPort}}/new-appointment"
2. mcp_microsoft_pla_browser_wait_for → "Select a doctor" (doctor dropdown placeholder)
3. mcp_microsoft_pla_browser_snapshot → verify doctor dropdown, no date/time fields yet
4. mcp_microsoft_pla_browser_console_messages → no errors
5. mcp_microsoft_pla_browser_take_screenshot → visual proof
```

---

## 6 — Teardown

After testing, stop the background processes. The VS Code compound launcher has `stopAll: true` — stopping the debug session kills all three hosts.

If launched via terminal, stop each background process by its terminal ID.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Blank page on Client | BFF not running | Start BFF before Client |
| Certificate error | Dev cert not trusted | Run `dotnet dev-certs https --trust` |
| Port in use | Previous instance still running | Kill the process or stop all terminals |
