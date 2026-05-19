# Codex Local Setup Handoff

This note is for another Codex or LLM agent that needs to clone and run Open Design locally, based on a successful Windows setup.

## Goal

Clone `https://github.com/nexu-io/open-design` directly into:

```powershell
C:\Users\Alexandre\Developer\open-design
```

Then run the web app locally inside Codex at:

```text
http://127.0.0.1:17573
```

## What Worked Here

- OS shell: Windows PowerShell.
- Node required by the repo: `24.x`.
- Node version used successfully: `24.13.1`.
- Package manager required by the repo: `pnpm@10.33.2`.
- Local lifecycle command: `pnpm tools-dev`.
- Working ports:
  - daemon: `17456`
  - web: `17573`

Do not use `pnpm dev`, `pnpm start`, or custom root scripts. This repository expects local runtime commands to go through `pnpm tools-dev`.

## Fresh Setup Commands

From PowerShell:

```powershell
cd C:\Users\Alexandre\Developer
git clone https://github.com/nexu-io/open-design.git open-design
cd C:\Users\Alexandre\Developer\open-design
```

Select Node 24. If `nvm` for Windows is available:

```powershell
nvm install 24.13.1
nvm use 24.13.1
node --version
```

Expected:

```text
v24.13.1
```

Install or expose the pinned PNPM version:

```powershell
npm install -g pnpm@10.33.2
pnpm --version
```

Expected:

```text
10.33.2
```

Install dependencies:

```powershell
pnpm install
```

Build the daemon once before starting the local runtime:

```powershell
pnpm --filter @open-design/daemon build
```

Start daemon and web:

```powershell
pnpm tools-dev start web --daemon-port 17456 --web-port 17573
```

Expected result:

```text
daemon: started - running - http://127.0.0.1:17456
web: started - running - http://127.0.0.1:17573
```

Open:

```text
http://127.0.0.1:17573
```

The page title should be `Open Design`.

## Verify Runtime

```powershell
pnpm tools-dev status --json
```

Expected shape:

```json
{
  "apps": {
    "daemon": {
      "state": "running",
      "url": "http://127.0.0.1:17456"
    },
    "web": {
      "state": "running",
      "url": "http://127.0.0.1:17573"
    },
    "desktop": {
      "state": "idle"
    }
  }
}
```

`status` can report `partial` when only daemon + web are running. That is fine for browser/Codex usage. The desktop shell is not required for this local browser flow.

## Stop Runtime

```powershell
pnpm tools-dev stop
```

## Problems Encountered Here

### Codex CLI detected, but runs fail with `"node" nao e reconhecido`

Open Design first detected Codex through the npm global shim:

```text
C:\Users\Alexandre\AppData\Roaming\npm\codex.cmd
```

That made `GET /api/agents` report Codex as installed, but actual design runs failed with:

```text
agent exited with code 1
"node" nao e reconhecido como um comando interno ou externo, um programa operavel ou um arquivo em lotes.
```

Cause: `codex.cmd` calls `node` by name. The Open Design-spawned agent process can have a narrower `PATH` than the interactive PowerShell session, so it may not find `node.exe` even when `node --version` works in the terminal.

Fix used: create a wrapper that calls Node by absolute path:

```powershell
New-Item -ItemType Directory -Force -Path 'C:\Users\Alexandre\bin' | Out-Null
```

Create `C:\Users\Alexandre\bin\codex-open-design.cmd` with:

```bat
@echo off
"C:\nvm4w\nodejs\node.exe" "C:\Users\Alexandre\AppData\Roaming\npm\node_modules\@openai\codex\bin\codex.js" %*
```

Test it:

```powershell
& 'C:\Users\Alexandre\bin\codex-open-design.cmd' --version
```

Expected:

```text
codex-cli 0.131.0
```

Then configure Open Design to use the wrapper as `CODEX_BIN`:

```powershell
$current = (Invoke-RestMethod -Uri 'http://127.0.0.1:17456/api/app-config').config
if ($null -eq $current.agentCliEnv) { $current | Add-Member -NotePropertyName agentCliEnv -NotePropertyValue ([pscustomobject]@{}) }
$current.agentCliEnv | Add-Member -Force -NotePropertyName codex -NotePropertyValue ([pscustomobject]@{ CODEX_BIN = 'C:\Users\Alexandre\bin\codex-open-design.cmd' })
$body = $current | ConvertTo-Json -Depth 12
Invoke-RestMethod -Method Put -Uri 'http://127.0.0.1:17456/api/app-config' -ContentType 'application/json' -Body $body
```

Verify:

```powershell
Invoke-RestMethod -Uri 'http://127.0.0.1:17456/api/agents' | ConvertTo-Json -Depth 8
```

Codex should show:

```text
available: true
path: C:\Users\Alexandre\bin\codex-open-design.cmd
version: codex-cli 0.131.0
modelsSource: live
```

Reload the Open Design page if the UI still shows the previous agent state.

### `pnpm` not recognized after `nvm use`

After switching from Node 22 to Node 24, `corepack pnpm` worked but plain `pnpm` was not available. That broke `pnpm tools-dev`, because the repo scripts call `pnpm` directly.

Fix used:

```powershell
npm install -g pnpm@10.33.2
```

### `tools-dev` cannot find its generated bin after moving folders

This happened because dependencies were installed while the repo was inside an extra nested folder, then the repo was moved up one level. PNPM generated links still pointed to the old nested path.

Fix used from the final repo root:

```powershell
$env:CI='true'
pnpm install
```

This forced PNPM to recreate `node_modules` non-interactively and refresh workspace links.

### First `tools-dev start web` timed out waiting for daemon status

The daemon began booting but did not expose status in time. Building the daemon package fixed the next start:

```powershell
pnpm --filter @open-design/daemon build
pnpm tools-dev start web --daemon-port 17456 --web-port 17573
```

## Folder Placement Warning

Clone directly into:

```powershell
C:\Users\Alexandre\Developer\open-design
```

Avoid this nested shape:

```powershell
C:\Users\Alexandre\Developer\open-design\open-design
```

If the nested shape already happened, move the repository contents up carefully, then rerun:

```powershell
$env:CI='true'
pnpm install
pnpm --filter @open-design/daemon build
```

Do not delete unrelated files in the parent folder unless the user explicitly confirms.

## Git Hygiene

Before committing or syncing:

```powershell
git status --short --branch
```

Expected local runtime directories that should not be committed include:

```text
.od/
.tmp/
.playwright-mcp/
node_modules/
```

If unrelated files are present, leave them alone unless the user asks to move, delete, or commit them.
