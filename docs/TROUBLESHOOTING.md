# Troubleshooting

Every entry here is a real failure hit while building and testing this
skill, not a hypothetical — including the exact error text, so you can
match it directly.

## `/mcp` shows `omniroute` as failed / not connecting

**`Server rejected the configured Authorization header (HTTP 403)`**
The API key doesn't have the `manage` scope (or `mcp:connect`, or both).
Create a new key with `scopes: ["manage", "mcp:connect"]` — see
`docs/SETUP.md` step 4 — and update `~/.claude.json` with the new key.

**`MCP server is disabled. Enable it from the Endpoints page.`**
OmniRoute's `mcpEnabled` application setting defaults to `false` and isn't
exposed as a working toggle in the dashboard at the time of writing. Set it
via the settings API (`docs/SETUP.md` step 5) and restart the server.

**Configured but shows `ECONNREFUSED: Unable to connect` (not "failed" or "disabled" — just refused)**
This means Claude Code recognizes the `omniroute` entry but the OmniRoute
server process itself isn't running or isn't reachable at the URL you
registered — a config problem would show a different error (403, 400, a
disabled/stdio message); a refused connection means nothing is listening on
that port at all. Run `omniroute health`; if it says "Server not running,"
start it (`omniroute serve`), or better, enable `omniroute autostart` so
this doesn't happen again after a terminal window closes or a reboot (see
`docs/SETUP.md` step 3).

**Desktop app: `omniroute` doesn't show up at all, in any new conversation (unresolved)**
Real testing found: one already-running desktop conversation picked up the
`omniroute` entry mid-session and could call its tools successfully
(`omniroute_get_health` returned a real response). Every conversation
opened *after* that — including brand-new ones after a full restart of the
desktop app itself — showed no `omniroute` entry at all, not even as
"failed." This was never resolved. It's not a config problem (the same
`~/.claude.json` entry that worked in the first conversation didn't work in
any other one), and the cause is unknown — possibly the desktop app treats
`http`-transport ("Web") MCP servers differently from `stdio` ("Desktop")
ones for how they get discovered by new conversations, but this wasn't
confirmed. **Current recommendation: use the terminal for this skill.** If
you find what actually fixes desktop-app discovery, it's worth reopening
this.

**`MCP transport is set to "stdio", not "streamable-http". Change it from Settings.`**
OmniRoute's `mcpTransport` setting defaults to `stdio`, which doesn't match
Claude Code's HTTP-based MCP client. Set `mcpTransport` to
`"streamable-http"` via the settings API (`docs/SETUP.md` step 4) and
restart.

**`HTTP 400` with no further detail, right after fixing the above**
Usually means the settings change hasn't been picked up yet — restart the
OmniRoute server (`omniroute restart`), confirm `omniroute health` reports
healthy, *then* reconnect from `/mcp`.

**The server appears to hang or die during `omniroute restart`**
It's a real Next.js app with a lot of startup schedulers; a 60-second
"did not respond" warning during restart doesn't always mean it crashed.
Run `omniroute serve` in the foreground in one terminal, and
`omniroute health` from a second terminal to check whether it's actually up
before assuming it failed.

## Phase 7 fails immediately with a 401 for a provider you never picked

**`OmniRoute API error [401]: No active credentials for provider: openai`**
(or any provider you didn't intend) — this means the `omniroute-phase7-runner`
subagent wasn't given a literal model string and defaulted to a generic
placeholder. The invocation prompt to that subagent must spell out the
exact `provider/model` string (e.g. "use the model
`antigravity/gemini-2.5-pro`"), never a vague reference like "the
configured model." See the skill's own Phase 7 section.

## `Agent type 'omniroute-phase7-runner' not found`

Custom `.claude/agents/*.md` files are only loaded as invokable subagent
types when a Claude Code session **starts**, not hot-reloaded while it's
running. If this agent file was just created in the current session, exit
and restart the session (`cd` back into the project, `claude` again) before
invoking it. Agent files that already existed before the session started
work immediately.

## The delegate model's diff doesn't apply / `git apply` says "corrupt patch"

This is exactly why this skill uses SEARCH/REPLACE blocks instead of
unified diffs (see `docs/ARCHITECTURE.md`). If you're still seeing raw
unified-diff output from the runner, its agent file is stale — re-read
Setup step 4 in `SKILL.md`: a project's `omniroute-phase7-runner.md` is a
point-in-time copy and won't update itself when the master skill changes.
Compare it against `docs/omniroute-phase7-runner.md` and refresh it.

## A specific OMNIROUTE_MODEL keeps timing out

Not every model behind a router like OmniRoute is equally reliable —
some free/cheap tiers get rate-limited or degraded under load. Treat two
timeouts in a row on a trivial connectivity-test prompt as a real signal,
not transient flakiness, and try a different model from the ones OmniRoute
reports as `"available"` (check `omniroute_list_models_catalog`) rather than
retrying the same one indefinitely.

## The management/dashboard login is asked for and you don't have it handy

Some diagnostic pages (the OmniRoute dashboard itself) require its own
separate login, distinct from any API key. That's expected — the dashboard
is a convenience UI, not required for the skill to function once the CLI
setup steps above are done.
