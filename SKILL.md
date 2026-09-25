---
name: ai-kaderskill-omniroute
description: Standalone ticket-cycle skill — Scope → Advisor Strategy → Wayfinder → To-Spec → To-Tickets → Triage → Implement → Review — where Claude runs every planning and judgment phase, and the actual code-generation work (Phase 7) is delegated through OmniRoute's MCP tool to a non-Anthropic model, running inside a dedicated cheap-tier subagent. This keeps the expensive Anthropic model on planning and review, and puts the heavy-lifting coding work on a different AI subscription — saving your Anthropic weekly quota and getting real use out of other AI subscriptions you already pay for. Fully automatic once set up: no headless subprocess, no command to copy or paste. Requires OmniRoute installed, at least one non-Anthropic provider connected in it, and its MCP server registered and reachable from Claude Code. Confirmed working in the Claude Code terminal end-to-end. Desktop app support is unconfirmed — a live tool call succeeded in one already-running desktop conversation, but a fresh conversation never saw the server at all, even after a full app restart; use the terminal until that's resolved (see docs/TROUBLESHOOTING.md). Not tied to any specific project or language — works on any codebase.
---

# AI-KaderSkill — OmniRoute variant

A structured, ticket-based build cycle, phases run **in order, one at a
time**, never merged, never skipped.

## The core idea

Claude is excellent at planning, spec-writing, judging whether an
implementation actually meets its acceptance criteria — and comparatively
expensive to run for high-volume mechanical coding work against a limited
weekly usage quota. Most non-Anthropic AI subscriptions (a coding-plan
model, an OAuth-connected agent tier, a cheap/free API-key provider) are
underused by comparison. This skill splits the work accordingly:

- **Claude (this session), full price:** Scope, Advisor Strategy, Wayfinder,
  Spec, Tickets, Triage, and Review. All judgment calls stay here.
- **A cheap-tier Claude subagent:** the mechanical orchestration inside
  Phase 7 — reading the ticket and relevant files, building the request,
  applying what comes back, running tests. Still Claude, still your
  Anthropic quota, but on a cheaper model tier than the main session.
- **Your OmniRoute-connected provider, off Anthropic's quota entirely:** the
  actual "read this ticket and write the fix" reasoning, reached through a
  plain MCP tool call (`omniroute_route_request`), not a separate coding
  agent process.

Two separate delegations, two separate purposes — see `docs/ARCHITECTURE.md`
for the full breakdown of why it's structured this way and what was tried
and rejected along the way.

**Hard rule:** at the end of every phase, before doing anything else, output
a message in exactly this shape:

```
✅ Phase [N] done: [one-line summary of what just happened]
👉 Next: [Phase N+1 name] — [one sentence on what it will do]
[If input is needed from the user, ask for it here. Otherwise ask: "Go ahead?"]
```

Then **stop and wait** for the user's reply before starting the next phase.
Never proceed automatically.

---

## Setup (once per project, before the first Phase 7 of a session)

1. **Confirm the `omniroute` MCP server is connected.** Run `/mcp` and check
   it shows a green check with a tool count — not "failed" or "disabled."
   If it's missing or failing, stop and tell the user; don't fall back to
   anything else silently. See `docs/SETUP.md` if it's not connected yet —
   OmniRoute's MCP feature needs three things enabled that aren't obvious
   from its own UI.
2. **OMNIROUTE_MODEL** — the `provider/model` (or combo name) string to
   route Phase 7 requests to. Never pick this silently — ask the user, and
   offer whatever they've previously confirmed working as the default
   suggestion. If nothing's confirmed yet, call `omniroute_get_health` and
   `omniroute_list_models_catalog` to see which providers actually have
   active credentials before suggesting anything — don't guess a model id.
3. **First-ever use of a new OMNIROUTE_MODEL in this project:** send one
   small test request through `omniroute_route_request` yourself (a trivial
   prompt, no file changes) and show the user the response, so they can
   spot-check it actually came from the intended provider. This is a normal
   MCP tool call — fine to run directly. Do this once per model per
   project, not once per ticket. If it errors or times out, try the next
   candidate model rather than assuming the whole setup is broken.
4. **Confirm (or create/refresh) the `omniroute-phase7-runner` subagent.**
   Check whether `.claude/agents/omniroute-phase7-runner.md` exists in this
   project.
   - If it doesn't exist, create it (a normal file write) with the content
     described below.
   - If it already exists, **read it and check it actually matches this
     skill's current Phase 7 instructions** — SEARCH/REPLACE blocks (not
     unified diffs), and the "use the exact literal model string, never a
     default/placeholder" rule — before trusting it. A project's copy is a
     point-in-time snapshot; it does not update itself when this skill file
     is updated elsewhere. If it's out of date, rewrite it to match.
   - `tools:` at least Read, Write, Edit, Glob, Grep, Bash, and
     `mcp__omniroute__omniroute_route_request`.
   - `model:` a tier the user picks — ask, don't assume. Offer `sonnet` as
     the default recommendation (cheaper than Opus, still reliable enough to
     apply changes and judge a test failure correctly); mention `haiku` as a
     cheaper-still option if the user wants to push savings further and
     accepts more retries/less judgment quality on messy output.
   - Persona: it receives a ticket id, an implementer-agent name, and a
     literal OMNIROUTE_MODEL string; reads the implementer file itself,
     gathers the relevant existing files, calls `omniroute_route_request`
     with that exact model string (never substituting a default or example
     model id of its own — if it wasn't clearly given a concrete
     `provider/model` string, it stops and reports that rather than
     guessing), asks for SEARCH/REPLACE blocks, applies them via Edit, runs
     tests, retries once on failure, and returns a structured Report back.
     See the full agent-file template in `docs/omniroute-phase7-runner.md`
     — copy it as-is into the project and adjust only `tools`/`model` if
     needed.
5. **If step 4 just created the file for the first time in this session,
   stop here and tell the user to restart this Claude Code session** (exit,
   `cd` back in, `claude` again) before continuing to Phase 7. Custom
   `.claude/agents/*.md` files are only picked up as invokable subagent
   types at session start, not hot-reloaded mid-session — confirmed by a
   real test where a freshly-written agent file was invoked in the same
   session that created it and failed with "Agent type not found," while
   agent files that already existed before a session started worked fine.
   Don't skip this restart and don't try to invoke it anyway "just to see"
   — it will fail every time until restarted, and taking the fallback
   option silently spends full-price quota instead, defeating the entire
   point of the cheap-tier subagent. If the file already existed before
   this session started, skip this step.

## Phase 0 — Setup check (run once per repo, skip if already confirmed this session)

- Confirm the project folder is a Git repo linked to a remote (GitHub/GitLab/etc). **Default to creating a real GitHub repo for the project before doing anything else, if one doesn't already exist** — check `gh auth status` first; if authenticated, create the repo (`gh repo create`, ask the user for name/visibility/license rather than guessing) and push the initial commit, instead of leaving the project as a local-only sandbox. Only skip this and stay local if the user explicitly says so (e.g. a throwaway experiment) or no GitHub auth is available.
- Confirm a context file exists (e.g. `CONTEXT.md`) and read it if present. If it doesn't exist and the project has meaningful standing rules or conventions, offer to create one — ask the user what belongs in it rather than guessing.
- **Default tracker: GitHub Issues on that repo**, not local ticket files — tickets become issues (`gh issue create`), triage labels become real GitHub labels, and "close the ticket" in every later phase means closing the issue (`gh issue close`), not editing a `label:` line in a markdown file. Confirm this default with the user rather than assuming silently; fall back to local files only if the user prefers that or no GitHub repo is in play for this project. Either way, confirm that these five triage labels exist: `needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`. Create any missing ones (`gh label create` for GitHub Issues).
- If any MCP servers or external tools are required for this project, confirm they're connected/toggled on before proceeding — including the `omniroute` MCP server itself (see Setup above).
- **If the repo is public on GitHub, set up branch protection on the default branch** (`gh api .../branches/<default>/protection`): require pull requests with at least 1 approving review, disable force-pushes and branch deletion. Leave `enforce_admins` off so the project owner can keep pushing directly after an in-conversation review. Skip this for a private/solo-only repo unless the user asks for it anyway.

## Phase 1 — Scope the task

Ask the user (if not already stated) what they want to work on. Based on the answer, decide:
- **Big / ambiguous** (spans many components, unresolved design questions, unclear approach) → go to Phase 2 (Advisor Strategy), then Phase 3 (Wayfinder).
- **Small / already clear** (one specific, well-understood change) → skip straight to Phase 4 (To-Spec).

State which path you're taking and why in the "Next" message. Don't let a
small task's simplicity be used as an excuse to bypass the whole pipeline —
"small" still goes through Spec → Tickets → Triage → the delegated Phase 7;
it just skips Advisor Strategy and Wayfinder.

## Phase 2 — Advisor Strategy (only if routing to Wayfinder)

- If this project has a strategic-advisor command (e.g. `/consult`) configured, run it with a short description of the task and why Phase 1 classified it as big/ambiguous, and present the resulting report before spending effort on Wayfinder's own decomposition. If no such command exists, skip this step and go straight to Wayfinder — it's optional infrastructure, not a hard requirement of this skill.
- This is advisory, not a gate: proceed as-is unless the user explicitly accepts a recommendation. Never choose or switch a model on the user's behalf.
- Skipped entirely on the small/already-clear path.

## Phase 3 — Wayfinder (only if triggered above)

- Open one root ticket labeled `wayfinder:map`.
- Break the ambiguity into decision tickets, each tagged by type: Research, Grill, Prototype, Routine, or Manual.
- Resolve them one at a time. For **Grill** tickets, ask the user directly — don't guess. For **Research** tickets, investigate and write a short markdown summary before moving on.
- Do not proceed to Phase 4 until every decision ticket under `wayfinder:map` is resolved or explicitly deferred by the user.

## Phase 4 — To-Spec

- Convert the resolved decisions (from Wayfinder, or from the Phase 1 conversation if Wayfinder was skipped) into a spec document of user stories: *"As [role], I want [goal], so that [reason]."*
- Save it to the project (e.g. `specs/<short-name>.md`) and post it in full for the user to confirm before moving on.

## Phase 5 — To-Tickets

- Split the confirmed spec into tracer-bullet tickets. Each ticket must declare its own blocking dependencies on other tickets explicitly.
- Open them as real tickets on the configured tracker.
- Every new ticket starts labeled `needs-triage`.

## Phase 6 — Triage

For each open ticket, assign one label:
- `ready-for-agent` — fully clear, no open questions, no design decision pending.
- `ready-for-human` — clear, but the work itself isn't something an agent should do (a design call, physical/manual work, a business decision).
- `needs-info` — missing a specific detail; ask the user for exactly that detail.
- `wontfix` — explicitly out of scope; confirm with the user before applying.
- Leave as `needs-triage` if genuinely undetermined.

Only tickets labeled `ready-for-agent` move to Phase 7.

## Phase 7 — Implement (via OmniRoute MCP tool, run inside a cheap-tier subagent)

For the next `ready-for-agent` ticket:

- Identify which existing `*-implementer.md` agent file applies to this
  ticket. Most projects only need one, generically named `implementer`; a
  larger project may define several scoped by area (e.g.
  `backend-implementer`, `frontend-implementer`) — use whichever the
  project already defines by its own scoping rule. If none exists yet for
  this project, create one first: ask the user what model tier and tools it
  should have, don't assume, and include `effort: medium` alongside
  whatever `model:` tier they choose.
- Invoke the `omniroute-phase7-runner` subagent (confirmed/created in
  Setup) via the Agent tool. The invocation prompt **must spell out the
  literal OMNIROUTE_MODEL string** (e.g. "use the model
  `provider/model-name`"), not a vague reference like "the configured
  model" — a real test showed a subagent invoked without the literal string
  substituted a generic placeholder model the provider had no credentials
  for, and the ticket failed before it ever reached the delegate model.
  Also pass the ticket id and the applicable implementer agent name. Do the
  rest of this phase's work **inside that subagent's own turns, not the
  main thread** — that's the entire point of routing it there instead of
  doing it here directly.
- Inside the subagent: read the implementer agent file's own instructions
  (persona, standing rules, architecture constraints, mandatory tests);
  gather the minimum context needed — the ticket's full body (id, title,
  description, acceptance criteria, any prior review comments if this is a
  re-run after a Review failure) plus the contents of the specific existing
  files the ticket is likely to touch. Don't dump the whole repo — keep the
  request scoped, the way a careful human reviewer would scope a diff
  request.
- Inside the subagent: call the `omniroute` MCP server's
  `omniroute_route_request` tool with a single request that includes the
  implementer agent's instructions, the ticket, the relevant file contents,
  and an explicit, firm ask for **SEARCH/REPLACE blocks**, one per change,
  in this exact format:

  ```
  path/to/file.ext
  <<<<<<< SEARCH
  (exact existing text, copied verbatim from the file content given above —
  not retyped from memory, not paraphrased)
  =======
  (the replacement text)
  >>>>>>> REPLACE
  ```

  plus a structured Report back (what was implemented, spec gaps,
  unverifiable items, unmet criteria), same shape the agent file itself
  already asks for. **Why SEARCH/REPLACE, not a unified diff:** a real test
  run showed a unified diff from a delegate model came back with a
  malformed hunk header and `git apply` rejected it outright — unified
  diffs need exact line numbers and context width, which smaller/cheaper
  coding models are often unreliable at. SEARCH/REPLACE has no line-number
  math at all: it's just "find this exact text, replace it with this text,"
  which maps directly onto the Edit tool's `old_string`/`new_string` and
  tolerates the kind of minor formatting slop a diff can't.
- Inside the subagent: for each SEARCH/REPLACE block, apply it with the Edit
  tool directly (`old_string` = the SEARCH text, `new_string` = the REPLACE
  text) against the named file. If a SEARCH block doesn't match anything in
  the file (model paraphrased instead of copying verbatim, or the file
  changed since the request was built), that one block failed — don't guess
  at a fix; treat it the same as a failed apply below. Run the project's
  tests (or whatever the ticket/agent file specifies) via Bash. If tests
  fail, a SEARCH block didn't match, or the response clearly ignored the
  ticket, send **one** follow-up `omniroute_route_request` with the failure
  appended, then re-apply and re-test. If still wrong after that, the
  subagent stops and reports the failure plainly rather than looping.
- Enforce the same standing rules the agent file already enforces
  (`CONTEXT.md`, "do not commit," any architecture constraints, mandatory
  tests) — read from the agent file and folded into the request, all inside
  the subagent.
- Back in the main thread: read the subagent's final Report back and present
  it as the ticket's Report back, unchanged. If the subagent reports it
  couldn't complete the ticket (a SEARCH block never matched, tests never
  passed after the retry, or OMNIROUTE_MODEL errored), report that to the
  user plainly and ask whether to retry, switch OMNIROUTE_MODEL, or fall
  back to a native (main-thread) Agent call for this one ticket — don't
  silently retry in a loop.

## Phase 8 — Review (native — no OmniRoute, fully automatic)

- **Run this yourself, in this session, no delegation.** Reading a diff and
  judging it against acceptance criteria is normal Claude Code operation.
  Run `/code-review` (or the project's configured review step) directly, or
  read the diff and acceptance criteria yourself and judge it inline.
- Read the actual diff (`git diff`, or the created/changed files directly if
  there's no git repo for this project) and check it against the ticket's
  acceptance criteria, citing spec sections the way the implementer agent
  would have. Report pass/fail per criterion plainly. Independently verify
  — re-run the tests yourself, check `git status --porcelain` for unexpected
  file changes — rather than trusting the subagent's self-report alone.
- If it fails: go back to Phase 7 with the specific review comments appended
  to the ticket body, re-run the delegated implementer, then re-review
  (still native). Don't fix it yourself even here — the fix still belongs to
  the delegated implementer, only the *judgment* is native.
- On pass: close the ticket, return to Phase 6 to triage/pick the next
  ticket. **Closing a ticket does not by itself cut a release** — see
  "Versioning & Release" below.
- If no `ready-for-agent` or `needs-triage` tickets remain, tell the user
  the cycle is complete and summarize what closed.

## Versioning & Release (applies whenever this project's code is deployed anywhere outside the repo itself)

A merge to the default branch means "the code exists," not "this is safe to deploy." Keep those two separate, deliberate steps:

- Maintain a `CHANGELOG.md` at the project root, one entry per release, listing which tickets/issues it closes.
- Only cut a version tag (semantic versioning, e.g. `v0.5.0`) and a GitHub Release once a batch of closed tickets has actually been verified and the user wants to deploy — never tag automatically just because a ticket closed; ask first, same as any other hard-to-reverse/public action.
- Treat the tagged release, not the default branch's HEAD, as the only thing that's "safe to install/deploy."
- For any code that can't be executed or unit-tested in this environment, require a standalone verification write-up — a mock-object simulation proving the logic — before a ticket touching that logic can close in Phase 8.

## Throughout

- If context usage is climbing, run `/compact` instead of ending the session. Preserve: everything in `CONTEXT.md`, and any decisions made in this run's Wayfinder phase.
- Never invent a decision on the user's behalf for a Grill-type or design question — always ask.
- If the `omniroute` MCP server drops (check `/mcp` if a Phase 7 call fails unexpectedly), stop and tell the user rather than silently falling back to a native subagent mid-cycle — that would spend quota without them choosing it.
