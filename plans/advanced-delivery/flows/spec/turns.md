# Turns

## What a step holds

A hub step holds one resource per repository it still has to land, named `repo:<repository_id>/<branch>`, where
`<repository_id>` is the record's `repo_<ulid>` id and `<branch>` is that record's `base_branch`. Each commit pointer
resolves to its own record, with its own base branch, owner, forge, and secret, so one chunk's resources may sit on
different branches as well as different repositories. The resources are fixed when the visit submits
([verdict.md](./verdict.md#submitting)): `resolve_chunk_repositories` (`hub/domain/config/repositories.py`) runs then,
and the set is the `targets` of its `RepositoryResolution` less every repository the chunk already carries a
`merged/<repo>` marker for. Markers are the chunk's, not the visit's — `LandedRepos.of` (`hub/domain/chunk/model.py`)
reads them across every node and epoch — and a marker names `owner/repo`, so a target is left out when its `owner/repo`
is among `LandedRepos.names`, the same match `_route` makes when it lists what is still unlanded. A chunk that landed
one repository in an earlier visit, or before the upgrade, therefore holds only the flows of the repositories it has
left. Each resource is one flow ([record.md](./record.md)).

Each pass resolves the visit's repositories again, as `_run_locked` does today. An operator who edits a repository's
`base_branch` while a chunk is submitted therefore changes the branch that chunk's next pass lands on, while the chunk
still holds the old branch's flow; the new branch's flow sees that chunk only at its next submission. The edit is rare
and the operator's own, and the epic accepts it rather than guarding it.

A step holds nothing when its chunk has no commits (`NOTHING_TO_RESOLVE`), when every repository it resolves to has
already landed, or when its action does not land: whether it does is its catalogue entry's `ActionSpec.lands`
(`hub/domain/actions/catalogue.py`, the actions epic), never the graph's. `land` lands; `record-findings`,
`garden-deliver`, and `pass` do not, so a chunk recording its review findings or a garden run delivering waits behind no
landing, whichever graph version it is pinned to.

A step refused before it can name its resources holds nothing and is refused at once, before it submits, as `_refuse`
refuses it today. That holds for every hub step, landing or not, exactly as `_run_locked` resolves and refuses for every
hub node now: resolution refuses with `repository-unresolved` or `repository-not-linked`, and `GitCommits.of` (moved by
the actions epic to `hub/domain/actions/visit.py`) with `unconverged-delivery`. One refusal applies only to a step that
lands, since only a landing writes markers: a visit whose resolved repositories share an `owner/repo` is refused at
submit ([record.md](./record.md#what-the-record-says)). `repository-secret-unavailable` is different: a repository's
secret is revealed when the driver builds the visit for a turn ([below](#building-the-visit)), so a visit that meets it
has already submitted. Its turn ends with outcome `failure` and routes `failure` as `_refuse` does, which is its verdict
([verdict.md](./verdict.md)), with the cause in its `hub-log.repository-secret-unavailable` artifact. Whatever else an
action answers inside a turn — a choice it routes, or a cause it escalates to a person — ends the turn as any answer
does: a routed choice is the verdict, an escalation is not ([verdict.md](./verdict.md#escalation)).

## A turn

A turn is one pass of a visit's action, together with the reads that build its visit and the routing of its answer — the
unit the slot admitted. Nothing calls `HubNodeExecutor.run` to run a pass any longer, and nothing acquires a slot: the
driver takes the turn, builds the visit, hands the pass to the `ActionLoop`, writes each effect the pass records through
its `IEffectSink`, and when the pass answers, routes the answer and ends the turn. A visit has at most one open turn,
and a resource has at most one open turn holding it. Steps that hold nothing still take turns, so one visit is never run
twice at once, which is what the slot's non-reentrancy guaranteed for them before.

A turn is taken in one write transaction. Every scoped write takes the tenant row lock first (`hub/store/scoped.py`),
which already serializes every taker in the tenant; the transaction also takes `lock_keys(conn, "flow", resources)`
(`hub/store/internal/chunk_rows.py`) in sorted order, so the rule stays correct should the tenant lock ever narrow.
Inside it, the turn starts only if the visit is next on every flow it holds ([who is next](#who-is-next)), none of those
flows has an open turn, and the visit has none; it writes the turn and a `turn-started` fact per flow and commits.
Ending a turn writes its `ended_at`, its outcome, and a `turn-ended` fact per flow. Several resources are therefore
taken together or not at all, and two multi-repository chunks never hold one branch apiece waiting on each other.

## Building the visit

Once a turn has started, the driver builds what the pass is handed, off the event loop: the `Visit` as the actions epic
defines it, and, for an action whose spec `lands`, one `RepositoryForge` per resolved repository from `ForgeRegistry`
(the forges epic), which reveals each repository's secret as it builds. Before handing the pass over, it checks each
resolved repository's `offered` capabilities against the action's `ActionSpec.requires(repository)`; a repository
lacking one escalates the chunk with cause `forge-capability-missing`, records the forges epic's event and
`hub-log.forge-capability-missing` artifact, and ends the turn `escalated` without a pass. Both steps are the ones the
forges epic places in `HubNodeExecutor`, moved here unchanged in what they check and answer.

Building can fail. A secret that cannot be revealed ends the turn `failure`, as above. Any other exception raised while
building — a resolution the code did not expect, a malformed artifact — ends the turn and routes `failure` with its
traceback in the action's `hub-log.<action>` artifact, as the actions epic answers an exception escaping a pass, so a
fault that would recur on every turn reaches the graph's failure edge and its bounce cap rather than retrying forever.
Only a turn whose end cannot be written at all is left to the driver's sweep ([below](#a-turn-that-cannot-end)).

## Who is next

Each submission carries its tenant's arrival sequence ([record.md](./record.md)). A submission is **eligible** when its
chunk's status is `delivering`, its visit is current — the chunk's newest movement enters the submission's node and its
`latest_epoch` is the submission's epoch — its visit has no open turn, and its next poll is due: no `hub_node_poll` row
for the visit, or the newest one older than the interval its wait asks for and past the `next_poll_at` its pass answered
with (`Pass.next_poll_at`, the actions epic), when it answered with one. The interval is
`PollPolicy.interval_for(wait)`, read for the newest row's wait: in this epoch the only wait is `pending`, and it
answers the node's `poll_interval` as `PollPolicy.of(node).interval` does today. `PollPolicy` moves from
`hub/delivery/hub_node.py` to `hub/domain/delivery/flows.py`, beside `FlowLine`, which reads it; the `policies` epic
gives a poll row its wait kind and `interval_for` its second answer. A submission is **next** on a flow when no eligible
submission there arrived before it. A visit takes a turn when it is next on every flow it holds and each of them is
free; a visit holding nothing is next whenever it is eligible.

A visit is keyed as hub-advance keys it today, by the node the chunk stands at and `latest_epoch` — the newest epoch
across its leases, restarts, and epoch owners (`Chunk.latest_epoch`). Every writer that can raise that epoch while the
chunk still stands at the node starts a new visit by design: a hub-node escalation, whose hub lease opens the next epoch
(`record_bounce_escalation`, `hub/store/internal/chunk_escalations_store.py`); a restart onto the same node; and a
runner's lease minted for the chunk after a requeue. Each of those is a fresh request to land, and goes to the back of
the line. Nothing else raises the epoch of a chunk standing at a hub node, and a writer added later that does without
meaning to is caught by `hub:flow-line-standing` ([record.md](./record.md#invariants)).

Being current is what keeps a line from waiting on a chunk that has left: a submission whose chunk has moved to another
node or into a later epoch is never eligible, and the driver's reconcile closes it in the same tick
([verdict.md](./verdict.md#the-reconcile)).

An ineligible submission keeps its place and blocks nothing: a chunk waiting on its CI, escalated to `needs_human`, or
paused after an escalation is passed over while it is. An eligible submission that is next on one flow but not on
another waits, and the flow it is next on waits with it; a line on one branch never jumps a chunk that also needs
another, so a chunk landing into several repositories is never starved.

## The flow driver

A flow lives at the hub and needs no runner, so the hub starts turns itself. `FlowDriver`
(`hub/delivery/flow_driver.py`) is a hub lane under `bzh:lane-contract` with the `flow.` crash-point family
([crash points](#crash-points)), and the one hub lane that is not a `Sweep`. A `Sweep` steps a synchronous `sweep()`
through `asyncio.to_thread` and waits for it (`hub/app.py`), which suits a pass that finishes within its interval; a
turn's pass outlives many ticks, so the driver needs a host that can keep work running between them. It has its own
coroutine, `FlowDriver.run()`, started in the hub's `_lifespan` beside `Sweep.all`'s tasks, raced against the same
`shutdown` event, and cancelled with them.

The work splits in two, along the line between what reads the store and what waits on a forge:

- **The tick** is a steppable `tick(store, clock) -> TickResult` (`bzh:steppable-loop`), synchronous and run through
  `asyncio.to_thread`, as every sweep's pass is. For every open tenant it reconciles the lines
  ([verdict.md](./verdict.md#the-reconcile)), ends any turn that cannot end ([below](#a-turn-that-cannot-end)), and
  then, for every visit that may take a turn, oldest arrival first, takes the turn. It answers the turns it started.
  Tests drive one tick at a time and stop at any boundary.
- **A turn's task** is an `asyncio` task the driver creates on the hub's event loop for each turn the tick started, and
  keeps in a set it owns. The task builds the visit through `asyncio.to_thread`, hands the pass to the `ActionLoop` and
  awaits it through `asyncio.wrap_future` on the loop's `run_coroutine_threadsafe` future, writes each effect the pass
  hands its sink through `asyncio.to_thread` before releasing the pass to its next forge call, and, when the pass
  answers, routes the answer and ends the turn in one more `to_thread` write. Every exit from the task, normal or not,
  passes through one `finally` that removes its turn from the driver's set. No store call runs on the event loop, and no
  transaction is held open across a wait.

A tick never waits for a turn's task. The driver ticks every two seconds, and a `wake: asyncio.Event` it owns cuts the
wait short, set whenever something could change who is next: a submission, a turn ending, a withdrawal, a pause or
resume, a requeue, and a hub-advance. A wake within a second of the previous tick waits out the second, so a fleet of
runners each calling hub-advance every thirty seconds never turns the driver into a busy loop. The tick and every turn's
task each catch and log their own failures, so a failing tick skips a cycle and a failing task ends only its own turn,
and neither stops the driver.

Request handlers run on worker threads and wake the driver through `loop.call_soon_threadsafe(wake.set)`, behind a port
the domain names. `ApplyService` (`hub/domain/execution/apply.py`) takes an `IHubStepWake` — `wake() -> None`, declared
in `apply.py` — in place of `IHubNodeExecutor`, which goes with this epic; composition hands it the driver's wake.

A pass is bounded (the hub configuration key `action_pass_seconds`, the actions epic) and spent almost wholly waiting on
a forge, so passes need no pool and no process: each runs on the `ActionLoop`'s one thread. The actions epic has the
executor write the effects a pass records; from this epic the turn's task does, and `effects.record()` returns to the
pass only once its write is durable, so the pass never takes its next forge call ahead of the record of its last. Turns
on different flows run side by side, which is the epic's point: a landing into `blizzard` and one into `blizzard-mock`
no longer wait on each other, and a garden delivery waits on neither. The number of passes in flight is bounded by the
number of open flows and hold-nothing visits a tenant has at once, never by a pool size; each forge handle's client
bounds its own connections.

The two places that ran a hub step synchronously stop doing so. `ApplyService._respond` and `_land_migration`, which
`_migrate_across` reaches (`hub/domain/execution/apply.py`), wake the driver and answer `HUB_NODE_TAKEN` as before; the
driver's reconcile submits the visit on its next tick, and the runner holds the chunk's environments exactly as it does
now. The actions epic's bridge from these synchronous callers onto the `ActionLoop` goes with them, since nothing
outside the driver runs a pass. `POST /api/fleet/chunks/{chunk_id}/hub-advance` (`hub/api/fleet.py`) wakes the driver
and answers with where the visit stands — "landing on blizzard/master", "3rd on blizzard/master", "pending — next poll
at …" — in the detail it already carries. `HubAdvanceResponse` (`wire/chunk.py`) keeps `ran` and `outcome_choice` on the
wire, answering `false` and null, since a hub-advance never runs a pass; no runner reads either, and its docstring stops
describing a slot. A runner that disappears no longer stalls its chunk's landing, since the driver lands it regardless,
and no runner request waits on a pass any longer.

The driver serves one hub process, as the slot did: the open-turn rule is enforced in the store, but recovering turns at
boot ([carry-over.md](./carry-over.md)) and ending turns that cannot end assume no other process is running them. A
tenant that closes mid-tick raises `TenantClosed` from its write; the driver skips that tenant for the rest of the tick,
and a turn's task whose tenant has closed ends without writing, its turn left to the tenant's teardown.

The tick is two passes under `bzh:probe-gated-pass`. Admission — taking a turn for every visit that may — acts on work
as it comes due, much of it made due by a poll interval elapsing, so it owes no probe. The reconcile converges the lines
from where chunks stand, which is a corpus a change probe can watch: each tenant's reconcile first compares a watermark
— the newest id across the tenant's transitions, escalations, requeues, pause and resume facts, restarts, stops, and
completions — with the value its last full reconcile saw, skips the rescan when they match, and runs in full on every
wake and at least once a minute whatever the probe says. The rescan reads only the live set (`bzh:live-set-read`):
chunks that are not terminal and whose newest movement enters a hub node, with their facts loaded in bulk
(`bzh:bulk-reconstitution`) rather than one chunk at a time.

## Pending

A pass that answers a reserved wait — `pending`, the one wait the actions epic reserves — ends its turn with that
outcome and records its `hub_node_poll` row through the driver's `_record_wait`, which writes the row as
`_record_pending` does today. Its submission keeps its place and is ineligible until its poll is due, so a branch is
never held through a CI wait: the flow gives the next eligible submission a turn, and the waiting chunk is next again
when its poll is due. Turns on a branch are taken in arrival order; landings complete in whatever order their checks
finish, exactly as they do under the slot. A reserved wait is never a verdict and never routed, so no graph authors an
edge for it and the unroutable rule never applies to it.

The poll timeout and its bounce are unchanged in shape: the driver checks the timeout before it builds the visit and
routes `_route_pending_timeout` without calling the action, against the timeout the actions epic sets, three hours by
default and in every shipped graph. The timeout runs from the visit's first `pending` row, as today, and it is wall
clock: the time a chunk spends behind turns on its branch counts, as the time it spent waiting for the slot did. That
wait is short beside the bound, since a chunk is behind only the turns of submissions that arrived before it, each a
single pass bounded at five minutes and most far shorter.

## A turn that cannot end

A turn ends in its task's `finally`. If the write that ends it fails — a dropped connection, a lock timeout — or the
code that would make it dies before it runs, the turn stays open, and while it is open nobody else on its branch moves.
The slot's answer was to reclaim itself after `DEFAULT_SLOT_STALE_AFTER`, even while the step that held it could still
be running; a turn never needs that, because the driver knows which turns it is running.

`FlowDriver` keeps the set of turn ids whose task it is running, guarded by a lock: a turn id joins the set in the tick
that takes the turn, before that tick's transaction commits, and leaves it when the task's `finally` runs — after the
turn's end has committed, or after the attempt to write it has failed. Each tick, before admitting anyone, ends every
open turn in the store that is not in the set as `interrupted`, with a `turn-ended` fact on each flow it held and a
`turn-interrupted` event (warning). A turn whose ending write failed is therefore ended by the next tick. Its answer is
lost with the failed write, and its submission is eligible again at once: its next turn takes a fresh pass, which skips
whatever its markers say is done, as a re-run does today, and answers again.

A pass that never answers is bounded by the driver as well as by the actions epic's cancellation. The `ActionLoop`
cancels an overrunning pass at its next `await`, but a pass stuck in synchronous code never reaches one, and while it is
stuck it holds the `ActionLoop`'s one thread, so no other pass runs either. The driver therefore gives each turn a
deadline of its own, the pass bound plus thirty seconds, measured on its own clock. A turn's task still waiting at the
deadline cancels its future, marks the pass's sink closed so any effect the pass hands it afterwards is refused, records
a `turn-interrupted` event naming the deadline and the action, and ends the turn `interrupted` itself. Because a
cancelled pass raises at its next `await`, and every forge call is one, the pass makes no forge write once its turn has
ended; and because its sink is closed, nothing it learned is recorded under a turn that is over. A stuck loop also keeps
every later pass from starting, which the operator sees as a run of `turn-interrupted` events naming the deadline.

`DEFAULT_SLOT_STALE_AFTER` and its injection are removed.

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
epic's, armed the same way now that the turn's task is the writer. Two windows the slot had are closed rather than
armed. A verdict's transition and its turn's end are one transaction ([verdict.md](./verdict.md#the-verdict)). A
landing's `merged/<repo>` marker, its `delivery_repo_landed` row, its close intents, its flow's `landed` fact, and its
`flow-landed` event row are one transaction ([record.md](./record.md#what-the-record-says)); the live publish of that
event follows the commit and, like every live frame, is not durable.
