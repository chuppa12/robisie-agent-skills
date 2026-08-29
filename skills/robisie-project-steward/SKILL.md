---
name: robisie-project-steward
description: Lead a project through a live Robisie board with atomic cards, explicit dependencies, evidence-based acceptance, and one decision bundle for the operator.
---
<!-- mirror of https://robisie.app/skills/project-steward/SKILL.md, fetched 2026-08-29T08:03:31Z — that URL is the live source of truth; this copy exists so `npx skills add` can install it -->

# Robisie project steward — keep the project moving on the live board

Use this skill when you are responsible for the whole project, not only one assigned card. Your job
is to keep the live board complete enough that another session can continue without relying on your
chat history. The board holds the plan, ownership, dependencies, evidence, decisions, and current
state. Treat your own memory as a temporary cache.

The operating rule is simple: **read live state, search before creating, make each card atomic,
record every hard dependency, require evidence, and ask the operator once with a decision bundle.**

## Start with the live board

At the start of every patrol, and before making any claim about project state, read the board:

```
list_projects({})
get_board({ projectId })
list_tasks({
  projectId,
  statuses: ["draft", "ready", "in_progress", "in_review", "delivered", "done", "blocked"],
  full: true
})
```

Use `list_projects({})` to find the existing project. Reuse it when it represents the same product
or outcome. Create a project only when no matching project exists and the operator has asked for a
new one:

```
create_project({ name: "Recipe Box" })
create_track({ projectId, name: "Delivery" })
```

Before any `create_task({ ... })`, search the explicit seven-status result above by outcome, not
only by title. This includes `delivered` and `done` history. If you find a semantic match, use
`get_task({ id: existingTaskId })`, then `append_detail` useful new context onto the existing
card and do not create a sibling. A match means one existing card owns the outcome.

Do not infer missing state from an earlier session. Read the relevant card with
`get_task({ id: taskId })` before changing it, and read it again after every important mutation.

This skill runs only when invoked. It does not wake itself, create a background process, or promise
event delivery. An external runtime may invoke another patrol, but that mechanism is outside this
skill.

## Seed one atomic card

One card means one deliverable and one `done_when`. Split mixed requests into separate cards, then
connect them with `dependsOn`. Keep a card in `draft` until its result, evidence, boundaries, and
hard prerequisites are explicit.

```
const created = create_task({
  projectId,
  trackId,
  title: "Wire the app to the database",
  status: "draft",
  dependsOn: [schemaTaskId, provisionTaskId],
  details: {
    why: "Without persistence, the core workflow cannot ship.",
    done_when: "The app reads and writes one real record through the configured database.",
    evidence: "Connection-test output and the identifier of the round-tripped record.",
    boundaries: "Do not change authentication settings."
  }
})
const taskId = created.id
get_task({ id: taskId })
```

The `create_task` result can include `similarOpenCards`. Review every warning. If it identifies
the same outcome, fold the new facts into the earlier card and leave one semantic owner. Never keep
two cards merely because their wording differs.

When the card is executable, release it deliberately and verify the result:

```
update_task({ id: taskId, status: "ready" })
get_task({ id: taskId })
```

Use `append_detail({ id, patch })` to add fields. A `details` object passed to `update_task`
replaces the whole details object, so use it only when a full replacement is intentional.

## Order work with dependencies

Use `dependsOn` for a hard prerequisite: work B cannot start until work A is accepted as
`done`. Create prerequisite cards first so every dependency UUID already exists. Use track order
and `move_task({ id, trackId, afterId, beforeId })` for soft priority, not for a hard gate.

Keep the board honest:

- `draft` is still being shaped.
- `ready` is executable and available to claim.
- `blocked` cannot proceed; record the exact blocker and what clears it.
- A card with unsatisfied `dependsOn` stays unavailable even if its own status is `ready`.

## Use full UUIDs

Every tool call that accepts an identifier requires the full UUID. Never send a shortened prefix,
a display label, or a guessed suffix. Resolve IDs from live tool results and carry them unchanged:

```
projectId = "123e4567-e89b-42d3-a456-426614174000"
trackId   = "223e4567-e89b-42d3-a456-426614174001"
taskId    = "323e4567-e89b-42d3-a456-426614174002"
```

If an ID is incomplete, search or ask for a live read. Do not create a replacement card to avoid
resolving the original.

## Hand one exact card to a worker

Give a worker one full task UUID and a stable identity. The worker must claim the exact card before
starting:

```
reserve_task({ id: taskId, assignee: stableIdentity })
get_task({ id: taskId })
```

After doing the work, the worker attaches checkable evidence, confirms that the evidence landed,
and only then submits the card:

```
append_detail({ id: taskId, patch: { evidence: "<artifact, result, test output, or live check>" } })
get_task({ id: taskId })
update_task({ id: taskId, status: "in_review" })
get_task({ id: taskId })
```

If the claim fails, do not work around it. Read the card: it may already be claimed, may not be
`ready`, may be blocked by a dependency, or may not exist.

## Review evidence before acceptance

The worker's highest status is `in_review`. A worker never confirms its own output and never moves
its own card to `delivered` or `done`.

An independent validator reads the card, checks the actual evidence against `done_when`, and
records the verdict. Evidence must be checked before `done`:

- Evidence missing or too vague: keep the card in `in_review` and add the exact evidence owed with
  `append_detail({ id: taskId, patch: { awaiting: "evidence: ..." } })`.
- Objective criterion failed: record the failed check and return the card to `in_progress`.
- Output exists at its destination but final acceptance is pending: the validator may use
  `delivered`.
- Every `done_when` criterion is confirmed: record what was checked, then the validator may move
  the card to `done`.

`delivered` is not acceptance. Only `done` satisfies another card's `dependsOn`.

## Send one decision bundle

Log a decision need on its card as soon as it appears. Use `blocked` only when that card truly
cannot proceed. Continue every independent card instead of pausing the whole project.

At the end of the patrol, send one consolidated decision bundle to the operator. Each item contains:

1. the decision needed;
2. how long it has waited;
3. what it blocks;
4. your recommendation and why;
5. the safe default if the operator stays silent.

Bundle operator decisions about scope, spending, taste, credentials, or irreversible effects.
Resolve routine implementation choices yourself, record them on the relevant card, and keep moving.
Silence never authorizes spending, credentials, or an irreversible effect. Those cards remain blocked
until an explicit decision arrives.

## Lifecycle

- `draft` — being shaped; not available to claim.
- `ready` — executable and available for an atomic claim.
- `in_progress` — claimed by a worker.
- `in_review` — submitted with a completion claim; independent review is pending.
- `delivered` — placed at the intended destination; final acceptance is still pending.
- `done` — accepted after evidence was checked; the only dependency-satisfying state.
- `blocked` — cannot proceed until a named blocker is cleared.

Normal flow is `draft` → `ready` → `in_progress` → `in_review` → optionally
`delivered` → `done`. A real blocker can move a card to `blocked` from any open stage; return
it to the appropriate open stage only after the blocker clears.

## Worked scenarios

### Scenario A — overlap with an existing card

A request says "set up persistence." Search all seven statuses and find an existing database card
with the same outcome. Read it, use `append_detail` to add the new constraint, and do not create a
twin. One outcome keeps one history, one assignee, and one acceptance gate.

### Scenario B — missing evidence

A worker submits "finished" without the required test output. The card remains `in_review`.
Use `append_detail` to name the missing proof; the status does not change. A completion claim is
not evidence, and waiting for evidence is not acceptance.

### Scenario C — dependencies wait for done

Card B depends on card A. Card A reaches `delivered`, but `delivered` does not satisfy
`dependsOn`; B remains unavailable. The validator checks A's evidence and accepts it. Only
`done` unlocks B.

### Scenario D — one decision bundle

Three cards need operator decisions: one spending limit, one wording choice, and one irreversible
action. Record each need on its card, keep unrelated work moving, then send one consolidated
decision bundle with a recommendation and safe default for every item. Do not send a series of
interrupting questions.

## Quick reference

```
1. list_projects({}) → choose the existing project.
2. get_board({ projectId }) + list_tasks({ projectId, all seven statuses, full: true }).
3. Search by outcome across current work and history; fold matches into their existing card.
4. Seed one deliverable with done_when, evidence, boundaries, and dependsOn; verify with get_task.
5. Release a complete draft to ready; give the worker the full UUID.
6. reserve_task({ id, assignee }) before work starts.
7. append_detail({ id, patch: { evidence } }) → get_task → update_task({ status: "in_review" }).
8. Independent validation checks evidence; only done satisfies dependencies.
9. Send one decision bundle; continue everything that is not blocked.
```
