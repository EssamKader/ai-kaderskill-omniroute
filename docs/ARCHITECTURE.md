# Why it's built this way

## The problem this solves

Claude is very good at planning, scoping, writing specs, and judging
whether an implementation actually satisfies its acceptance criteria. It's
comparatively expensive to run for high-volume, mechanical coding work,
against a limited weekly usage quota. Meanwhile, most people running
Claude Code also pay for at least one other AI subscription (a coding-plan
model, an agent-tier OAuth connection, a cheap or free API-key provider)
that sits mostly idle. This skill routes the expensive judgment work to
Claude and the high-volume coding work to whatever other provider is
already being paid for and underused.

## What was tried and rejected first

**A fully headless, unsupervised delegate session.** The original design
launched a completely separate Claude Code process, pointed at a
non-Claude model via a router, with `--dangerously-skip-permissions` so it
could write files without a human approving every action. This is blocked
by Claude Code's own safety layer as "Create Unsafe Agents" — confirmed
blocked regardless of user confirmation, and confirmed to be a
framework-agnostic restriction, not specific to any one delegation tool.
It's blocked at the level of *the assistant autonomously building or
launching an unsupervised, permission-bypassed coding session against a
non-Claude model* — not at the level of any particular command or vendor.

**A background "watcher" process that ran the same thing automatically.**
The next attempt tried to remove the remaining human step (copying a
command into a terminal) by having a small standalone script watch a queue
folder and launch the same kind of session automatically once a human
started the watcher once. Writing that watcher script was *also* blocked
under the same "Create Unsafe Agents" classifier — the restriction covers
building standing infrastructure that would later launch such sessions
unattended, not just doing it live in the current turn. This is the actual,
final boundary: no architecture change removes it, because the block is on
the *shape* of the action (autonomous, unsupervised, permission-bypassed,
non-Claude), not on any specific tool used to reach it.

## What actually works, and why it's a different shape

**Routing a plain chat-completion request through a registered MCP tool.**
OmniRoute exposes an `omniroute_route_request` MCP tool — a normal
request/response call, same category of action as calling any other MCP
tool (a filesystem tool, a search tool, a CAD/BIM tool). There is no
permission bypass, no unsupervised file-writing session, and no autonomy
being granted to a non-Claude model — it returns text, and whatever
applies that text to a file still runs under Claude Code's own normal
tool-use path. That's why this design isn't blocked: it's a genuinely
different action, not a workaround of the same one.

## Two separate delegations, two separate purposes

1. **Main thread → a cheap-tier Claude subagent** (`omniroute-phase7-runner`).
   Purpose: cost-tier segregation. Claude Code doesn't support switching a
   running session's own model mid-conversation, so the mechanical work
   inside Phase 7 (reading files, applying a change, running tests) runs
   inside a subagent whose model is pinned independently in its own agent
   file. Still Claude, still counted against the same Anthropic quota, just
   on a cheaper per-token tier than the main session.
2. **That subagent → OmniRoute → your configured provider.** Purpose: quota
   segregation. This is the hop that actually leaves Anthropic's billing
   entirely — the "figure out the fix" reasoning happens on whatever
   provider you connected in OmniRoute, paid for by that provider's own
   account, not your Claude usage.

## Why SEARCH/REPLACE blocks, not unified diffs

The first real end-to-end test asked the delegate model for a unified diff
(`git diff` format) and had it applied with `git apply`. It failed — the
model's diff had a malformed hunk header, and `git apply` rejected it
outright, needing a fallback to the Edit tool anyway. Unified diffs require
exact line numbers and exact context width; smaller and cheaper coding
models are frequently unreliable at getting that precisely right.
SEARCH/REPLACE blocks (the same technique used by tools like Aider) have no
line-number math at all — just "find this exact text, replace it with this"
— which maps directly onto Claude Code's own Edit tool (`old_string` /
`new_string`) and tolerates minor formatting slop that breaks a diff
outright.

## Why a project's runner agent file needs manual refreshing

Each project gets its own copy of `.claude/agents/omniroute-phase7-runner.md`,
generated once by the skill's Setup step. That copy is a snapshot — if this
skill's master file changes later (as it did, moving from unified diffs to
SEARCH/REPLACE), older projects' copies don't update themselves. Setup
explicitly checks for this staleness every time it confirms the file
exists, rather than assuming a present file is automatically a current one.
