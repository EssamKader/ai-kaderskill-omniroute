# Setup guide

This walks through everything needed once, per machine, before the skill can
run — installing OmniRoute in isolation from your normal Node/Claude Code
setup, connecting a provider, and registering its MCP server with Claude
Code. Three of these steps exist because of real bugs/gaps found in
OmniRoute's own onboarding — they're not optional polish, skip them and the
skill will fail with a specific, confusing error each time.

## 1. Install OmniRoute in an isolated Node runtime

Don't install OmniRoute against your system Node/npm — it pulls native
dependencies (`better-sqlite3`) that need to match the exact Node version,
and you don't want that colliding with whatever Node your other tools (or
Claude Code itself) rely on.

1. Download a portable Node.js build (zip, not the installer) — Node 22 LTS
   or newer. Extract it somewhere outside your normal PATH, e.g.
   `C:\tools\node-v24.x.x-win-x64`.
2. From that folder, install OmniRoute locally (not globally):
   ```powershell
   cd C:\tools\node-v24.x.x-win-x64
   .\npm.cmd install omniroute
   ```
3. Verify isolation — none of these should be affected:
   ```powershell
   node -v          # your normal system Node, unchanged
   npm list -g       # your normal global packages, unchanged
   ```
4. From here on, always invoke OmniRoute via its full path from that
   isolated folder: `C:\tools\node-v24.x.x-win-x64\omniroute.cmd`. Never add
   it to your system PATH.

## 2. Start the server and connect at least one provider

```powershell
& "C:\tools\node-v24.x.x-win-x64\omniroute.cmd" serve
```

Open the dashboard it prints (default `http://localhost:20128/dashboard`)
and connect at least one non-Anthropic provider under **Providers** — an
API key for a provider you already pay for (Groq, DeepSeek, OpenRouter,
Moonshot, etc.), or an OAuth-connected agent subscription (e.g. Google
Antigravity) under **Providers → OAuth connections**. This is the account
that will actually do Phase 7's code generation, off your Anthropic quota.

## 3. Keep the server running without a terminal

Don't rely on leaving `omniroute serve` running in a terminal window you
might close later — a real test lost the connection this way (closing the
window that was running it killed the server, and every MCP call started
failing with `ECONNREFUSED` until it was noticed and restarted). Enable
autostart instead, so it runs at login as a background process:

```powershell
& "C:\tools\node-v24.x.x-win-x64\omniroute.cmd" autostart enable
& "C:\tools\node-v24.x.x-win-x64\omniroute.cmd" autostart status
```

This matters more if you're using the Claude Code **desktop app** rather
than a terminal — there's no terminal window to notice has closed.

## 4. Create an MCP-capable API key

Go to **API Keys** (or **Endpoints**) in the dashboard and create a new key
with **both** of these scopes — a key with only `mcp:connect` can establish
the MCP connection but gets a `403 AUTH_001` on every actual tool call
(get_health, list_models_catalog, route_request, …), since those route
through OmniRoute's management API, which needs the broader scope:

- `mcp:connect`
- `manage`

Or via the CLI:

```powershell
& "C:\tools\node-v24.x.x-win-x64\omniroute.cmd" api api-keys post-api-keys --body '{\"name\":\"claude-code-mcp\",\"scopes\":[\"manage\",\"mcp:connect\"]}'
```

Copy the returned `sk-...` key — you'll need it in step 5. **Never commit
this key anywhere, including this repo.**

## 5. Enable MCP itself and set its transport to Streamable HTTP

These are two separate application settings, both of which default to a
state that doesn't work with Claude Code, and neither is exposed as an
obvious toggle in the dashboard's Endpoints/MCP page at the time of writing:

```powershell
& "C:\tools\node-v24.x.x-win-x64\omniroute.cmd" api settings patch-api-settings --body '{\"mcpEnabled\":true}'
& "C:\tools\node-v24.x.x-win-x64\omniroute.cmd" api settings patch-api-settings --body '{\"mcpTransport\":\"streamable-http\"}'
```

Then restart the server so it picks both up cleanly:

```powershell
& "C:\tools\node-v24.x.x-win-x64\omniroute.cmd" restart
```

Confirm it's actually up before continuing:

```powershell
& "C:\tools\node-v24.x.x-win-x64\omniroute.cmd" health
```

## 6. Register the MCP server with Claude Code

Add this to `~/.claude.json`'s top-level `mcpServers` block (global scope —
available in every terminal Claude Code session on this machine):

```json
"omniroute": {
  "type": "http",
  "url": "http://localhost:20128/api/mcp/stream",
  "headers": {
    "Authorization": "Bearer <the sk-... key from step 3>"
  }
}
```

## 7. Verify the connection

Start (or restart) a Claude Code terminal session, or open a fresh Code tab
in the desktop app, and run `/mcp`. You should see `omniroute` listed with a
green check and a tool count (dozens of tools — `omniroute_get_health`,
`omniroute_route_request`, `omniroute_list_models_catalog`, and more). If it
shows "failed" or "disabled" instead, see `docs/TROUBLESHOOTING.md`.

**Desktop app note:** confirmed working, but a newly registered MCP server
was not picked up by an already-running desktop conversation, or even a new
conversation in an already-running app instance — only a **full restart of
the desktop app** actually reloaded it. Do that before concluding it's
broken.

## 8. Install the skill

Copy `SKILL.md` from this repo into `~/.claude/skills/ai-kaderskill-omniroute/SKILL.md`
(create the folder if it doesn't exist). Claude Code picks up skills from
that location automatically — no restart needed for a brand-new skill file
(only *agent* files, created per-project inside Phase 7's Setup step, need a
session restart — see the skill's own Setup section for why).

From here on, run `/ai-kaderskill-omniroute` at the start of any project
where you want this workflow.
