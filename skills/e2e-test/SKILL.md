---
name: e2e-test
description: "Use when running end-to-end browser tests against the full MyApp stack (STS + BFF + WFE + Azurite). Provides the exact launch sequence, port map, health-check procedure, Playwright MCP navigation and assertion patterns, and teardown steps. Use for: smoke-testing a feature in the browser, verifying a page renders after implementation, checking auth flows, validating empty/populated states."
argument-hint: "Describe the scenario, e.g. 'verify /new-appointment page loads and shows doctor dropdown'"
---

# E2E Test — Full-Stack Browser Testing Skill

Run end-to-end browser tests against the running MyApp stack using the **Playwright MCP** tools.

## Prerequisites

- **Azurite** must be installed (`npm install -g azurite`)
- **Playwright MCP** tools must be available (the `mcp_microsoft_pla_browser_*` tool family)
- Solution must build successfully before launching

---

## 1 — Port Map

| Service | URL | Purpose |
|---------|-----|---------|
| **STS** | `https://localhost:5001` | OIDC test identity provider |
| **BFF** | `https://localhost:7094` | Backend-for-Frontend API + Swagger |
| **WFE** | `https://localhost:7259` | Blazor Server frontend (user-facing) |
| **Azurite** | `localhost:10000-10002` | Azure Storage emulator (blob/queue/table) |

---

## 2 — Launch Sequence

Start services in this exact order. Each step must succeed before the next.

### Step 2.1 — Azurite

```powershell
azurite --silent --location c:\azurite --debug c:\azurite\debug.log
```

Run as a **background** terminal process. Azurite is ready immediately (no health check needed).

> Alternative (Docker): `docker run -p 10000:10000 -p 10001:10001 -p 10002:10002 mcr.microsoft.com/azure-storage/azurite`

### Step 2.2 — Build all hosts

Use the VS Code task `build-all-hosts` (builds STS, BFF, WFE in parallel):

```
run_task: build-all-hosts
```

Or via terminal:

```powershell
dotnet build src/{{SolutionName}}.sln
```

### Step 2.3 — STS

```powershell
dotnet run --project src/STS/{{NamespaceRoot}}.STS.TestServer/{{NamespaceRoot}}.STS.TestServer.csproj
```

Run as **background**. Wait for the log line `Now listening on: https://localhost:5001`.

**Health check**: Use `mcp_microsoft_pla_browser_navigate` to `https://localhost:5001/.well-known/openid-configuration` — expect a JSON response.

### Step 2.4 — BFF

```powershell
dotnet run --project src/Host/BFF/{{NamespaceRoot}}.Host.Bff.csproj
```

Run as **background**. Wait for the log line `Now listening on: https://localhost:7094`.

**Health check**: Navigate to `https://localhost:7094/swagger` — expect the Swagger UI page.

### Step 2.5 — WFE

```powershell
dotnet run --project src/Host/Web/{{NamespaceRoot}}.Host.Wfe.csproj
```

Run as **background**. Wait for the log line `Now listening on: https://localhost:7259`.

**Health check**: Navigate to `https://localhost:7259` — expect a redirect to STS login or the home page.

---

## 3 — Authentication

The STS TestServer auto-issues tokens for a hardcoded test user — **no interactive login is needed** in most configurations. When the browser navigates to the WFE, the OIDC flow redirects to the STS which immediately returns a token.

**Test user claims:**

| Claim | Value |
|-------|-------|
| `name` | `user@example.org` |
| `email` | `user@example.org` |
| `uid` | `123456789` |
| `account_name` | `user101` |

If the STS presents a login form, the credentials are pre-filled — just submit.

---

## 4 — Playwright MCP Patterns

### Navigate to a page

```
mcp_microsoft_pla_browser_navigate → url: "https://localhost:7259/new-appointment"
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

**Tip:** After navigating to a page, check both browser console (`mcp_microsoft_pla_browser_console_messages`) **and** server logs (`get_terminal_output` for BFF/WFE) to catch errors on both sides.

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
1. LAUNCH    — Start Azurite → Build → STS → BFF → WFE (Steps 2.1–2.5)
2. NAVIGATE  — Open the target page in the browser
3. AUTH      — Handle OIDC redirect if needed (STS auto-issues token)
4. ASSERT    — Use snapshots and waits to verify expected content
5. CONSOLE   — Check browser console for errors
6. SCREENSHOT — Capture visual proof if needed
7. REPORT    — Summarize pass/fail to the user
```

### Example: Verify New Appointment page loads with doctor dropdown

```
1. mcp_microsoft_pla_browser_navigate → "https://localhost:7259/new-appointment"
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
| OIDC redirect fails | STS not running | Start STS first, verify `https://localhost:5001/.well-known/openid-configuration` |
| Swagger 500 error | Azurite not running | Start Azurite before BFF |
| Blank page on WFE | BFF not running | Start BFF before WFE |
| Certificate error | Dev cert not trusted | Run `dotnet dev-certs https --trust` |
| Port in use | Previous instance still running | Kill the process or stop all terminals |
