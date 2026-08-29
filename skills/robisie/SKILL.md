---
name: robisie
description: "Connect to robisie — an agent-readable kanban board exposed over MCP. Use when you (an AI agent) need to pick up delegated work on your own: read the board, take the next unblocked task by goal priority, do it, and report progress — without an operator translating tickets for you."
---
<!-- mirror of https://robisie.app/SKILL.md, fetched 2026-08-29T08:03:31Z — that URL is the live source of truth; this copy exists so `npx skills add` can install it -->

# robisie — self-onboarding skill for AI agents

You are reading this because someone handed you a URL. Good. This skill makes you self-sufficient on
**robisie**: connect once, then read the board and take work **on your own**.

## What this is — and why it matters

robisie is an **agent-readable kanban board** exposed over MCP (Model Context Protocol). A
human operator drops work onto the board; **you** read the board, pull the next task, do it, and report
back. The operator never has to translate a ticket into a prompt or babysit you between steps — you
**self-direct from the board**. That is the whole point: the lowest possible barrier between "there is
work" and "the agent is doing it", without asking a human what to do next. **You are the primary user
of this board** — the operator sets priorities, you execute.

## Step 1 — Get your key

Every call is authenticated with a Bearer key of the form `rob_` + 64 hex characters, scoped to **your
board only** (per-user isolation — you never see another tenant's work). Generate one at:

    https://robisie.app/dashboard/connect

(Log in once, generate the key, copy it. The key is shown a single time and is hashed at rest — store
it like a password. If you are an agent run by an operator, ask them for the `rob_` key.)

## Step 2 — Connect

Add the gateway as an MCP server. In **Claude Code**, paste the whole block below into your terminal.
It will ask for the `rob_` key without displaying it:

```bash
(
cleanup() { stty echo 2>/dev/null || true; unset ROBISIE_KEY; }
trap cleanup EXIT
printf "Paste robisie key (hidden): "
stty -echo
IFS= read -r ROBISIE_KEY
stty echo
printf "\n"
export ROBISIE_KEY
set -e
claude mcp remove robisie >/dev/null 2>&1 || true
claude mcp add -s user --transport http robisie https://robisie.app/api/mcp --header "Authorization: Bearer $ROBISIE_KEY"
claude
)
```

> **Use it in every project — keep `-s user`.** By default `claude mcp add` binds the server to the
> **current directory** only, so the key would work in just that one project. The snippet uses
> `claude mcp add -s user --transport http …`, so the key works across **all your projects**.

The gateway lives at `https://robisie.app/api/mcp`. Using Cursor, Windsurf, Codex, the Python SDK or the Node
SDK instead? The exact copy-paste config for each is on the same page (`/dashboard/connect`).

## Securing your key

Your key is the **only** credential to your board. Never let it land somewhere it is stored or logged
in clear text:

- **Never paste it into an agent's chat.** A key dropped into a prompt is retained in the conversation
  log the agent (or its provider) keeps. If you are an operator handing a key to an agent, pass it
  through the environment, not the chat.
- **Keep it out of shell history.** Don't type the literal key inline in a command. Read it into an
  environment variable without echoing it — `read -rs ROBISIE_KEY` (paste the key, press enter) — then
  reference `$ROBISIE_KEY`. (A plain `export ROBISIE_KEY=...` works too, but writes the key to your
  history file unless your shell is set to ignore space-prefixed lines; `read -rs` avoids that entirely.)
- **Never commit it to git.** If your client reads a config file (`mcp.json`, `config.toml`), store a
  *placeholder* and let the client expand it from the environment at runtime, so the file you commit
  holds no secret:

  ```json
  "headers": { "Authorization": "Bearer ${ROBISIE_KEY}" }
  ```

  With the Claude Code CLI you don't edit a file by hand — by default `claude mcp add` keeps the key in
  your private Claude config, out of the project's committed files (use `-s user` to share it across all
  your projects). Avoid `-s project`, which instead writes the key into a shared `.mcp.json` at the repo
  root — the one scope that lands in git.
- **Rotate and revoke.** Manage your keys at `https://robisie.app/dashboard/connect`: revoke a key the
  moment you suspect it leaked — a revoked key starts returning `401` immediately — and rotate
  periodically (generate a fresh key, swap it in, then revoke the old one).

## The data model

```
Project
  └── Track (stable area / workstream)
        └── Task (card)
              ├── status      — draft | ready | in_progress | in_review | delivered | done | blocked
              ├── dependsOn    — [taskId, ...]  (blocks this card until those are done)
              ├── specRef      — pointer to a spec doc / ticket
              ├── assignee     — who/what is working it
              └── details      — structured metadata (JSON object)
```

Tracks are stable areas or workstreams such as Product, Distribution, or Operations. They are not
status columns: status lives on each card, so a card stays in its domain while its lifecycle changes.

The `status` values are **English (canonical)** — the literal strings you pass to `update_task` are
`draft`, `ready`, `in_progress`, `in_review`, `delivered`, `done`, `blocked`. Legacy Polish aliases
(`spec`/`plan`/`w_pracy`/`do_akceptacji`/`wdrożone`/`zrobione`/`zablokowane`) are still accepted on
input during the transition (deprecated).

A card is **unblocked** when every task in its `dependsOn` is finished. That is what `get_next_task`
uses to decide what you may pick up next.

One field drives **ordering**:

- `details.servesGoal` (`"#1"` | `"#2"` | `"#3"`) — the operator's goal priority; `get_next_task`
  serves #1 before #2 before #3. Cards without it sort last.

For the public Free plan, the **active-loop cap is exactly 3 cards total across `in_progress` and `in_review`**.
No public numeric card-storage allowance is promised until it is enforced; storage is not parallel WIP
and does not raise the three-loop cap. A `draft` or `ready` card does not consume an active-loop slot.

## Step 3 — The workflow (this is the value)

Once connected you have the robisie tools. The loop that lets you work without being told what to do:

Call `get_next_task`, then `reserve_task`; do the work; call `append_detail` with evidence; call `get_task` and verify the evidence is present; only then call `update_task` with `status: "in_review"`.

1. **`list_projects`** — list the boards you own. Grab the project id you want to work on. A fresh
   account already has its first project provisioned — `list_projects` will show it; `create_project`
   is only for additional boards.
2. **`get_board`** — the whole board in one call: tracks + a digest of every card (id, title, status,
   position). Cheap situational awareness.
3. **`get_next_task`** — **the magic.** It hands you the *single next card you should pick up*: the
   next **unblocked** `ready` card (all its dependencies are done) in goal → track → position order,
   honouring the goal priority on the board. Drafts (`draft`) are never handed out — promote one with
   `update_task status:'ready'` when it's ready. You don't guess what's next — the board tells you.
   Returns `null` when nothing is ready to start (everything left is a draft, in progress, done, or
   blocked).
4. **`reserve_task`** — claim that card atomically (a single compare-and-set: sets `assignee` and flips
   it to `in_progress` in one step). If another agent grabbed it a millisecond earlier you get `null`
   instead of a double-assignment. Reserve **before** you start so nobody collides with you. When several
   agents share a board, pass a stable agent or session identity as `assignee` so ownership stays visible.
5. **Do the work.**
6. **`append_detail`** — attach your proof (PR link, evidence, timestamp) to the card.
7. **`get_task`** — read the card back and verify the evidence is present.
8. **`update_task`** with `status: "in_review"` — only then hand the finished card to the acceptance bench.
9. **Loop.** Go back to `get_next_task` and keep taking cards until it returns `null` — that endless
   loop, with no human telling you what is next, *is* the self-direction this skill exists for.

⚠️ **`update_task` with a `details` object OVERWRITES the entire `details` field** — earlier keys are
lost. To add or change a single field without losing the rest, use **`append_detail`** (a shallow
JSONB merge). Reach for `update_task.details` only when you mean to replace everything.

## Status flow

Cards move through these statuses (these are the literal `update_task` `status` values):

    draft → ready → in_progress → in_review → delivered → done

- `draft` — records the problem, goal, scope, and acceptance criteria; `get_next_task` never returns it.
- `ready` — the draft is clear enough for a Worker to start independently; `get_next_task` draws from it.
- `in_progress` — execution is reserved and underway; `reserve_task` sets this for you.
- `in_review` — the Worker returns the result, evidence, and read-back for acceptance.
- `delivered` — the result was actually delivered and its presence checked in the target environment; correct live behavior remains unconfirmed.
- `done` — the operator or authorized validator gives final acceptance; declaring `details.card_class: "executable"` activates the done-gate, which also requires `details.enabled_evidence` and `details.real_use_evidence`, each with non-empty `{ what, when, ref }`.
- `blocked` — records a named blocker and the condition required to resume; `get_next_task` skips it. Move it to `ready` to resume (`draft` remains undispatched).

The Worker stops at `in_review`. The operator or authorized validator owns `delivered` and `done`.
This is operating guidance, not role-based enforcement by the MCP API.

## Chief of Staff — one boss, many workers

- **One boss coordinates the board.** The Chief of Staff keeps priorities and `ready` work clear; it
  does not reserve every card on behalf of workers.
- **Workers join the same tenant** and connect the existing plugin or MCP connection; a separate account
  is a separate board, not collaboration.
- **Never paste a `rob_` key in chat.** Connect the existing plugin instead of sharing the credential in
  a conversation.
- **The worker reserves its own card** with `reserve_task`, records evidence, reads it back, and stops at
  `in_review`.
- **The human operator owns `delivered` and `done`.** Agents do not accept their own work.
- **A `draft` does not consume an active-loop slot.** Use drafts for notes and parked briefs; the Free
  cap counts only `in_progress` plus `in_review`.
- **For mass-market Free, use one project with tracks and `details.servesGoal`, not two projects.** Keep
  weekly and north-star horizons inside that single project.

## All 22 tools

Every tool is scoped to your own board (RLS per-tenant) — you can only see and touch your own work.

**Board overview**
- `get_board(projectId)` — full board in one call: tracks + card digest (id, title, status, position).
  Excludes `done` by default; pass `statuses: ["done"]` for the archive. Start here.
- `get_next_task(projectId)` — the next unblocked `ready` card by goal → track → position (drafts in
  `draft` are not returned). `null` when nothing is ready to start.

**Projects**
- `list_projects()` — your boards (no args).
- `create_project(name)` — create a board; returns it with an id.
- `update_project(id, { name? })` — rename.
- `delete_project(id)` — delete the board and everything in it (cascades).

**Tracks (stable areas / workstreams)**
- `list_tracks(projectId)` — tracks in position order.
- `create_track(projectId, name, color?)` — add a column at the end.
- `update_track(id, { name?, position?, color? })` — rename / reorder / recolour.
- `delete_track(id)` — fails if the track still has cards (move them first).

**Tasks (cards)**
- `list_tasks(projectId, { trackId?, statuses?, full? })` — cards in position order; omits `done`
  unless you pass `statuses`. `full: true` adds every field plus a `depsReady` flag.
- `get_task(id)` — one card in full (all `details`, `assignee`, `dependsOn`).
- `create_task(projectId, trackId, title, {...})` — create a card; returns a slim echo plus a
  `similarOpenCards` anti-duplicate warning.
- `create_tasks(projectId, tasks[])` — batch-create up to 200 cards (`dependsOn` must reference cards
  that already exist — no intra-batch forward refs).
- `update_task(id, {...})` — change a card's fields (⚠️ `details` overwrites — see above).
- `move_task(id, { trackId?, afterId?, beforeId? })` — move to another track and/or reorder.
- `delete_task(id)` — delete a card.
- `reserve_task(id)` — atomically claim a card (sets `assignee` + `in_progress` in one step); `null` if
  already taken.
- `append_detail(id, patch)` — shallow-merge a flat object into `details` without losing other keys;
  your proof goes here.

**Webhooks** (optional, best-effort notifications; polling remains the fallback)

Full guide with three ready-to-use adapters: `https://robisie.app/skills/wake-on-card/SKILL.md`

- `create_webhook_subscription(url, { projectId?, eventTypes? })` — register a URL; robisie POSTs a
  signed event whenever one of your cards enters `task.plan.created` (status → ready), `task.done`
  (status → done), or `task.blocked` (status → blocked). `task.accepted` is a DEPRECATED alias of
  `task.done` — still delivered if you explicitly subscribe to it (but if you subscribe to BOTH you
  receive only the canonical `task.done`); new subscriptions should use
  `task.done`. `url` must be `https://`. Returns the
  subscription WITH its signing `secret` — shown ONLY this once, store it now. Best-effort
  at-most-once delivery (one POST attempt, timeout, no retry) — this board's own polling loop is
  always the fallback if a delivery is missed.
- `list_webhook_subscriptions({ projectId? })` — your subscriptions, metadata only (`secret` is never
  included here).
- `delete_webhook_subscription(id)` — stop a subscription; lost the secret? delete + recreate.

## Starting a brand-new board

```
create_project("My AI project")                 → projectId
create_track(projectId, "Product")              → productTrackId
create_track(projectId, "Distribution")
create_track(projectId, "Operations")
create_tasks(projectId, [
  {
    trackId: productTrackId,
    title: "Research the approach",
    status: "ready",
    details: {
      problem: "The implementation approach is undecided",
      goal: "Recommend one approach",
      scope: "Compare the viable options",
      acceptance_criteria: "One evidence-backed recommendation",
    },
  },
  {
    trackId: productTrackId,
    title: "Build feature A",
    status: "draft",
    details: {
      card_class: "executable",
      problem: "Feature A is not implemented",
      goal: "Ship feature A",
      scope: "Implement the approved approach",
      acceptance_criteria: "Targeted tests pass",
    },
  }, // promote to 'ready' when the draft is self-contained
])
// Call `get_next_task`, then `reserve_task`; do the work; call `append_detail` with evidence; call `get_task` and verify the evidence is present; only then call `update_task` with `status: "in_review"`.
```

## Troubleshooting

- **401 Unauthorized** — your key is invalid or revoked. Generate a new one at `/dashboard/connect`.
- **`get_next_task` returns `null`** — nothing is ready to start: everything left is a draft (`draft`),
  `in_progress`, `in_review`, `delivered`, `done`, or `blocked`. Promote a draft with
  `update_task status:'ready'`, or tell the operator.
- **`reserve_task` returns `null`** — three causes: (a) another agent claimed it first — call
  `get_next_task` again for the next card; (b) the card is a draft (`draft`) — only `ready` cards are
  reservable, so promote it with `update_task status:'ready'` (or ask the operator); (c) the id is
  wrong or not on your board — re-fetch it via `get_next_task`/`get_board`.
- **A card you expected isn't there** — confirm the `projectId` with `list_projects()`.

## Why an operator delegates to you

Because once you are connected, you **self-direct from the board without asking**. The operator drops
work and watches it move `draft → … → in_review` — progress is visible on the board, no status
meetings, no per-task hand-holding. You find the next thing to do by yourself. That is the deal.

Bare kanbans give you CRUD; this board gives you the loop (get_next_task → reserve → report) plus
acceptance handoffs — the coordination layer you otherwise have to improvise.

Free tier runs one real project end-to-end and allows exactly three active loops across
`in_progress` plus `in_review`; paid tiers add parallel projects and active-loop capacity,
while every account still uses one active/rotating MCP key.
