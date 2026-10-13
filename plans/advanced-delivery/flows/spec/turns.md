# Turns

## What a step holds

A hub step holds one resource per repository its chunk's commits resolve to, named `repo:<repository_id>/<branch>`,
where `<repository_id>` is the record's `repo_<ulid>` id and `<branch>` is that record's `base_branch`. Each commit
pointer resolves to its own record, with its own base branch, owner, forge, and secret, so one chunk's resources may sit
on different branches as well as different repositories. The resources are fixed when the visit submits
([verdict.md](./verdict.md#submitting)): `resolve_chunk_repositories` (`hub/domain/config/repositories.py`) runs then,
and the `targets` of its `RepositoryResolution` give the set. Each resource is one flow ([record.md](./record.md)).

A step holds nothing when its chunk has no commits (`NOTHING_TO_RESOLVE`), or when its action does not land: whether it
does is the action's own `lands` (`IHubAction`, the actions slice), never the graph's. `land` lands; `record-findings`
and `garden-deliver` do not, so a chunk recording its review findings or a garden run delivering waits behind no
landing, whichever graph version it is pinned to.

A step refused before it can name its resources holds nothing and is refused at once, before it submits, as `_refuse`
refuses it today: resolution refuses with `repository-unresolved` or `repository-not-linked`, and `GitCommits.of`
(`hub/delivery/hub_node.py`) with `unconverged-delivery`. `repository-secret-unavailable` is different: it is found only
when a repository's secret is revealed, which happens inside the pass, so a visit that meets it has already submitted;
its turn ends `refused` and routes `failure` as `_refuse` does, which is its verdict ([verdict.md](./verdict.md)).

## A turn

A turn is one pass of a visit's action, the unit the slot admitted. `HubNodeExecutor.run` (`hub/delivery/hub_node.py`)
no longer acquires a slot: it takes a turn, runs `_run_locked` with its one awaited pass, and ends the turn in a
`finally`. A visit has at most one open turn, and a resource has at most one open turn holding it. Steps that hold
nothing still take turns, so two requests can never run one visit at once, which is what the slot's non-reentrancy
guaranteed for them before.

A turn is taken in one write transaction under the tenant row lock, with `lock_keys(conn, "flow", resources)`
(`hub/store/internal/chunk_rows.py`) serializing every taker of the same resources in sorted order. Inside it, the turn
starts only if the visit is next on every flow it holds ([who is next](#who-is-next)), none of those flows has an open
turn, and the visit has none; it writes the turn and a `turn-started` fact per flow and commits. Ending a turn writes
its `ended_at`, its outcome, and a `turn-ended` fact per flow. Several resources are therefore taken together or not at
all, and two multi-repository chunks never hold one branch apiece waiting on each other.

## Who is next

Each submission carries its tenant's arrival sequence ([record.md](./record.md)). A submission is **eligible** when its
chunk's status is `delivering`, its visit has no open turn, and its next poll is due: no `hub_node_poll` row, or the
newest one older than `PollPolicy.of(node).interval`. A submission is **next** on a flow when no eligible submission
there arrived before it. A visit takes a turn when it is next on every flow it holds and each of them is free; a visit
holding nothing is next whenever it is eligible.

An ineligible submission keeps its place and blocks nothing: a chunk waiting on its CI, paused before it reached the
node, or escalated to `needs_human` is passed over while it is, and is next again, ahead of everything that arrived
after it, once it is eligible. An eligible submission that is next on one flow but not on another waits, and the flow it
is next on waits with it; a line on one branch never jumps a chunk that also needs another, so a chunk landing into
several repositories is never starved.

## The flow driver

A flow lives at the hub and needs no runner, so the hub starts turns itself. `FlowDriver`
(`hub/delivery/flow_driver.py`) is a loop on the hub's event loop, started from `_lifespan` (`hub/app.py`) beside the
sweeps. Each tick reads the state of every open tenant's flows and does what moves them on: for every visit that may
take a turn, oldest arrival first, it takes the turn and starts the action's pass as a task. It ticks every two seconds
and is woken at once whenever something could change who is next: a submission, a turn ending, a withdrawal, a pause or
resume, and a hub-advance.

A pass is short and bounded (`BZ_HUB_ACTION_PASS_SECONDS`, the actions slice), spent almost wholly waiting on a forge,
so passes need no pool and no process: each is a task on the event loop, and a tick never waits for one to finish. Turns
on different flows run side by side, which is the slice's point: a landing into `blizzard` and one into `blizzard-mock`
no longer wait on each other, and a garden delivery waits on neither.

The two places that ran a hub step synchronously stop doing so. `ApplyService._respond` and `_land_migration`, which
`_migrate_across` reaches (`hub/domain/execution/apply.py`), submit the visit and wake the driver, and answer
`HUB_NODE_TAKEN` as before; the runner holds the chunk's environments exactly as it does now. The actions slice's bridge
from these synchronous callers onto the event loop goes with them, since nothing outside the driver runs a pass.
`POST /api/fleet/chunks/{chunk_id}/hub-advance` (`hub/api/fleet.py`) wakes the driver and answers with where the visit
stands — "landing on blizzard/master", "3rd on blizzard/master", "pending — next poll at …" — in the detail it already
carries. A runner that disappears no longer stalls its chunk's landing, since the driver lands it regardless.

The driver serves one hub process, as the slot did: the open-turn rule is enforced in the store, but recovering turns at
boot ([carry-over.md](./carry-over.md)) assumes no other process is running them.

## Pending

A pass that answers `pending` ends its turn with outcome `pending` and records its `hub_node_poll` row as
`_record_pending` does today. Its submission keeps its place and is ineligible until its poll is due, so a branch is
never held through a CI wait: the flow gives the next eligible submission a turn, and the waiting chunk is next again
when its poll is due. Turns on a branch are taken in arrival order; landings complete in whatever order their checks
finish, exactly as they do under the slot.

The poll timeout, its bounce, and the bounce cap are unchanged: `_run_locked` still checks the timeout first and routes
`_route_pending_timeout` before the action is called.

## The turn's time bound

The slot was reclaimed after `DEFAULT_SLOT_STALE_AFTER` while the step that held it could still be running. A turn lasts
one pass, and a pass is cancelled at its bound and answers `pending`, so a turn always ends and is never taken over
while its pass runs. `DEFAULT_SLOT_STALE_AFTER` and its injection are removed.
