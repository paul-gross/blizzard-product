# Turns

## What a step holds

A hub step holds one resource per repository its chunk's commits resolve to, named `repo:<repository_id>/<branch>`,
where `<repository_id>` is the record's `repo_<ulid>` id and `<branch>` is that record's `base_branch`. Each commit
pointer resolves to its own record, with its own base branch, owner, forge address, and secret, so one chunk's resources
may sit on different branches as well as different repositories. The resources are fixed when the visit submits
([verdict.md](./verdict.md#submitting)): `resolve_chunk_repositories` (`hub/domain/config/repositories.py`) runs then,
and the `targets` of its `RepositoryResolution` give the set. Each resource is one flow ([record.md](./record.md)).

Each pass resolves the visit's repositories again, as `_run_locked` does today. An operator who edits a repository's
`base_branch` while a chunk is submitted therefore changes the branch that chunk's next pass lands on, while the chunk
still holds the old branch's flow; the new branch's flow sees that chunk only at its next submission. The edit is rare
and the operator's own, and the slice accepts it rather than guarding it.

A step holds nothing when its chunk has no commits (`NOTHING_TO_RESOLVE`), or when its action does not land: whether it
does is its catalogue entry's `ActionSpec.lands` (`hub/domain/actions/catalogue.py`, the actions slice), never the
graph's. `land` lands; `record-findings` and `garden-deliver` do not, so a chunk recording its review findings or a
garden run delivering waits behind no landing, whichever graph version it is pinned to.

A step refused before it can name its resources holds nothing and is refused at once, before it submits, as `_refuse`
refuses it today: resolution refuses with `repository-unresolved` or `repository-not-linked`, and `GitCommits.of`
(`hub/delivery/hub_node.py`) with `unconverged-delivery`. `repository-secret-unavailable` is different: a repository's
secret is revealed when the driver builds the visit for a turn, since an action receives its forge handles with their
credentials already bound, so a visit that meets it has already submitted. Its turn ends with outcome `failure` and
routes `failure` as `_refuse` does, which is its verdict ([verdict.md](./verdict.md)), with the cause in its
`hub-log.repository-secret-unavailable` artifact. Whatever else an action answers inside a turn — a choice it routes, or
a cause it escalates to a person — ends the turn as any answer does: a routed choice is the verdict, an escalation is
not ([verdict.md](./verdict.md#escalation)).

## A turn

A turn is one pass of a visit's action, together with the reads that build its visit and the routing of its answer — the
unit the slot admitted. Nothing calls `HubNodeExecutor.run` to run a pass any longer, and nothing acquires a slot: the
driver takes the turn, builds the visit, hands the pass to the `ActionLoop`, writes each effect the pass records through
its `IEffectSink`, and when the pass answers, routes the answer and ends the turn. A visit has at most one open turn,
and a resource has at most one open turn holding it. Steps that hold nothing still take turns, so one visit is never run
twice at once, which is what the slot's non-reentrancy guaranteed for them before.

A turn is taken in one write transaction under the tenant row lock, with `lock_keys(conn, "flow", resources)`
(`hub/store/internal/chunk_rows.py`) serializing every taker of the same resources in sorted order. Inside it, the turn
starts only if the visit is next on every flow it holds ([who is next](#who-is-next)), none of those flows has an open
turn, and the visit has none; it writes the turn and a `turn-started` fact per flow and commits. Ending a turn writes
its `ended_at`, its outcome, and a `turn-ended` fact per flow. Several resources are therefore taken together or not at
all, and two multi-repository chunks never hold one branch apiece waiting on each other.

## Who is next

Each submission carries its tenant's arrival sequence ([record.md](./record.md)). A submission is **eligible** when its
chunk's status is `delivering`, its visit is current — the chunk's newest movement enters the submission's node and its
`latest_epoch` is the submission's epoch — its visit has no open turn, and its next poll is due: no `hub_node_poll` row,
or the newest one older than `PollPolicy.of(node).interval` and past the `next_poll_at` its pass answered with
(`Pass.next_poll_at`, the actions slice), when it answered with one. A submission is **next** on a flow when no eligible
submission there arrived before it. A visit takes a turn when it is next on every flow it holds and each of them is
free; a visit holding nothing is next whenever it is eligible.

Being current is what keeps a line from waiting on a chunk that has left: a submission whose chunk has moved to another
node or into a later epoch is never eligible, and the driver's reconcile closes it in the same tick
([verdict.md](./verdict.md#the-reconcile)).

An ineligible submission keeps its place and blocks nothing: a chunk waiting on its CI, or escalated to `needs_human`,
is passed over while it is. An eligible submission that is next on one flow but not on another waits, and the flow it is
next on waits with it; a line on one branch never jumps a chunk that also needs another, so a chunk landing into several
repositories is never starved.

## The flow driver

A flow lives at the hub and needs no runner, so the hub starts turns itself. `FlowDriver`
(`hub/delivery/flow_driver.py`) is a hub lane hosted as a `Sweep` (`hub/app.py`), one of `Sweep.all`, under
`bzh:lane-contract` with the `flow.` crash-point family ([crash points](#crash-points)). Its pass is a steppable
`tick(store, clock, seams)` (`bzh:steppable-loop`), a pure function of the tenant's store, the clock, and the
`ActionLoop` it hands passes to, so tests drive one tick at a time and stop at any boundary. Like every sweep's, the
tick runs off the event loop through `asyncio.to_thread`, and so does every store write the driver makes; no store call
runs on the event loop, and no transaction is held open across a wait.

Each tick, for every open tenant, reconciles the tenant's lines ([verdict.md](./verdict.md#the-reconcile)), ends any
turn that cannot end ([below](#a-turn-that-cannot-end)), and then, for every visit that may take a turn, oldest arrival
first, takes the turn and hands the pass to the `ActionLoop`. A tick never waits for a pass to finish. The sweep runs
every two seconds and `Sweep` gains an optional `wake: asyncio.Event` that cuts its interval short, set whenever
something could change who is next: a submission, a turn ending, a withdrawal, a pause or resume, a requeue, and a
hub-advance. Those come from request handlers on worker threads, so they wake the driver through
`loop.call_soon_threadsafe(wake.set)`.

A pass is bounded (`BZ_HUB_ACTION_PASS_SECONDS`, the actions slice) and spent almost wholly waiting on a forge, so
passes need no pool and no process: each runs on the `ActionLoop`'s one thread. The actions slice has the executor write
the effects a pass records; from this slice the driver drains them instead, writing each through `asyncio.to_thread`,
and `effects.record()` returns to the pass only once its write is durable, so the pass never takes its next forge call
ahead of the record of its last. Turns on different flows run side by side, which is the slice's point: a landing into
`blizzard` and one into `blizzard-mock` no longer wait on each other, and a garden delivery waits on neither.

The two places that ran a hub step synchronously stop doing so. `ApplyService._respond` and `_land_migration`, which
`_migrate_across` reaches (`hub/domain/execution/apply.py`), wake the driver and answer `HUB_NODE_TAKEN` as before; the
driver's reconcile submits the visit on its next tick, and the runner holds the chunk's environments exactly as it does
now. The actions slice's bridge from these synchronous callers onto the `ActionLoop` goes with them, since nothing
outside the driver runs a pass. `POST /api/fleet/chunks/{chunk_id}/hub-advance` (`hub/api/fleet.py`) wakes the driver
and answers with where the visit stands — "landing on blizzard/master", "3rd on blizzard/master", "pending — next poll
at …" — in the detail it already carries. `HubAdvanceResponse` (`wire/chunk.py`) keeps `ran` and `outcome_choice` on the
wire, answering `false` and null, since a hub-advance never runs a pass; no runner reads either, and its docstring stops
describing a slot. A runner that disappears no longer stalls its chunk's landing, since the driver lands it regardless.

The driver serves one hub process, as the slot did: the open-turn rule is enforced in the store, but recovering turns at
boot ([carry-over.md](./carry-over.md)) and ending turns that cannot end assume no other process is running them.

## Pending

A pass that answers `pending` ends its turn with outcome `pending` and records its `hub_node_poll` row as
`_record_pending` does today. Its submission keeps its place and is ineligible until its poll is due, so a branch is
never held through a CI wait: the flow gives the next eligible submission a turn, and the waiting chunk is next again
when its poll is due. Turns on a branch are taken in arrival order; landings complete in whatever order their checks
finish, exactly as they do under the slot.

The poll timeout and its bounce are unchanged in shape: the driver checks the timeout before it builds the visit and
routes `_route_pending_timeout` without calling the action, against the timeout the actions slice sets, three hours by
default and in every shipped graph.

## A turn that cannot end

A turn ends in a `finally` that writes its end. If that write fails — a dropped connection, a lock timeout — or the code
that would make it dies before it runs, the turn stays open, and while it is open nobody else on its branch moves. The
slot's answer was to reclaim itself after `DEFAULT_SLOT_STALE_AFTER`, even while the step that held it could still be
running; a turn never needs that, because the driver knows which turns it is running.

`FlowDriver` keeps the set of turn ids whose pass it has handed to the `ActionLoop` and not yet ended, guarded by a
lock: a turn id joins the set in the tick that takes the turn, before that tick's transaction commits, and leaves only
after the turn's end has committed. Each tick, before admitting anyone, ends every open turn in the store that is not in
the set as `interrupted`, with a `turn-ended` fact on each flow it held and a `turn-interrupted` event (warning). Its
submission keeps its place and is eligible again at once, and its next turn takes a fresh pass, which skips whatever its
markers say is done, as a re-run does today. A turn whose pass is still running is always in the set, so it is never
ended under its pass. `DEFAULT_SLOT_STALE_AFTER` and its injection are removed.

## Crash points

Each window between a durable write and the next is a crash point in the `flow.` family (`bzh:crash-point-registry`,
`hub/delivery/flow_driver.py`), armed by a scenario of the kill-9 sweep (`tests/crash/test_kill9_sweep.py`):

| Crash point                         | Durable                                     | Not yet                       | Recovered by                                                       |
| ----------------------------------- | ------------------------------------------- | ----------------------------- | ------------------------------------------------------------------ |
| `flow.after-submit.before-turn`     | The submission, its flow rows, `submitted`  | Any turn                      | The next tick takes the turn; the reconcile never submits it twice |
| `flow.after-turn-start.before-pass` | The turn and its `turn-started` facts       | The pass                      | Boot recovery ends the turn `interrupted`; the next turn passes    |
| `flow.after-poll.before-turn-end`   | A `pending` pass's `hub_node_poll` row      | The turn's end                | Boot recovery ends the turn; the poll row keeps the chunk waiting  |
| `flow.boot.after-interrupt`         | One tenant's open turns ended `interrupted` | The next tenant's, the driver | The next boot ends the rest; ending a turn twice writes nothing    |

`flow.after-poll.before-turn-end` takes the place of `hubnode.after-poll.before-slot-release`, retired with the slot.
The windows inside a pass — `action.after-effect.before-write` and `action.after-write.before-resume` — stay the actions
slice's, armed the same way now that the driver is the writer. Two windows the slot had are closed rather than armed. A
verdict's transition and its turn's end are one transaction ([verdict.md](./verdict.md#the-verdict)). A landing's
`merged/<repo>` marker, its `delivery_repo_landed` row, its flow's `landed` fact, and its `flow-landed` event row are
one transaction ([record.md](./record.md#what-the-record-says)); the live publish of that event follows the commit and,
like every live frame, is not durable.
