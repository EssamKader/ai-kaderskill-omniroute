# ai-kaderskill-omniroute

A Claude Code skill that runs a structured, ticket-based build cycle where
**Claude plans and reviews**, and the **heavy-lifting coding work is
delegated to a different AI provider** — through [OmniRoute](https://www.npmjs.com/package/omniroute),
a local AI gateway — so it doesn't spend your Anthropic weekly quota.

```
Scope → Advisor Strategy → Wayfinder → Spec → Tickets → Triage → Implement → Review
        └──────────────── Claude, full price ─────────────────┘   └ delegated ┘  └ Claude ┘
```

## Why

Claude is excellent at planning, spec-writing, and judging whether an
implementation actually meets its acceptance criteria. It's comparatively
expensive to run for high-volume mechanical coding work against a limited
weekly quota — and most people running Claude Code also pay for at least
one other AI subscription that sits mostly idle. This skill puts the
judgment-heavy phases on Claude and the high-volume code generation on
whatever other provider you're already paying for.

## How it works, briefly

- **Phases 0–6, 8** (Scope, Spec, Tickets, Triage, Review) run natively, in
  your normal Claude Code session.
- **Phase 7** (Implement) runs inside a dedicated subagent pinned to a
  cheaper Claude model tier — not the main session — which calls out to
  your configured non-Anthropic model through OmniRoute's MCP tool, applies
  the result, and runs tests. No headless subprocess, no command to copy or
  paste — fully automatic once set up.

Full breakdown of why it's structured this way (including two earlier
designs that were tried and didn't work) is in
[`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

## Requirements

- [Claude Code](https://claude.com/claude-code) — **terminal confirmed
  working end-to-end. Desktop app support is unconfirmed, not recommended
  yet.** The mechanism itself (a normal MCP tool call plus a native
  subagent) has no terminal-specific requirement, and a live
  `omniroute_get_health` call did succeed from inside one already-running
  desktop conversation — but a fresh conversation, even after a full
  desktop-app restart, did not see the `omniroute` server at all. That one
  working case isn't evidence of general desktop support; it's evidence
  that conversation happened to pick up the config mid-session. Until a
  fresh desktop conversation reliably sees it, treat the desktop app as
  untested for this skill and use the terminal. See
  `docs/TROUBLESHOOTING.md` for what's been ruled out so far.
- [OmniRoute](https://www.npmjs.com/package/omniroute) installed and running,
  with at least one non-Anthropic provider connected
- OmniRoute's MCP server registered with Claude Code and connected (`/mcp`
  shows `omniroute` with a tool count)

## Setup

Full walkthrough, including three non-obvious OmniRoute settings that need
to be flipped before its MCP server actually works with Claude Code:
[`docs/SETUP.md`](docs/SETUP.md). Also covers `omniroute autostart enable`,
which keeps OmniRoute's server running at login instead of depending on a
terminal window staying open — this matters more once you're using the
desktop app, since there's no terminal to leave running in the background.

## Usage

**Terminal:**
1. `cd` into a project, start `claude`.
2. Run `/ai-kaderskill-omniroute`.
3. Answer its questions as they come — it drives the whole cycle itself,
   pausing between phases for your go-ahead.

**Desktop app:** not recommended yet — a fresh Code tab conversation did not
see the `omniroute` MCP server at all, even after a full app restart, in
real testing. Use the terminal until this is resolved; see
`docs/TROUBLESHOOTING.md` for what's been tried.

A step-by-step usage guide (Word document) covering the full workflow with
examples is in [`docs/AI-KaderSkill-OmniRoute-Usage-Guide.docx`](docs/AI-KaderSkill-OmniRoute-Usage-Guide.docx).

## Troubleshooting

Every error message this project has actually hit, with the exact fix:
[`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md).

## Not included

This skill deliberately does not attempt to fully eliminate the small
amount of manual setup per project (confirming a model, occasionally
refreshing a stale agent file). An earlier design tried to remove the last
human step entirely and was blocked by Claude Code's own safety layer,
on purpose — see `docs/ARCHITECTURE.md` for what was tried and why it
doesn't work, so it isn't re-attempted.

## License

MIT — see [`LICENSE`](LICENSE).
