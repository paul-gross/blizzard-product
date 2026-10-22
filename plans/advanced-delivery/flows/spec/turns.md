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

Each turn resolves the visit's repositories again when it builds the visit, as `_run_locked` does today, and a pass
lands only on the flows its submission holds. An operator may edit a repository's `base_branch`, or a project's links,
while a chunk is submitted; the turn then finds a set of resources that differs from the one the submission holds, and
landing on it would put the chunk on a branch whose line it never joined, beside whoever holds that branch's turn. So a
turn whose resolved resources differ from its submission's ends at once with outcome `superseded`, without a pass, and
closes the submission `superseded` in the same transaction, with a `withdrawn` fact on each flow it held. The next
reconcile submits the visit afresh, holding the resources it now resolves to, at the back of each of their lines: the
operator moved the ground under it, and the chunk lines up again on the new ground. A resubmission of the same visit
takes the next `generation` ([record.md](./record.md#the-tables)). A turn whose resolution refuses — the repository was
unlinked from the chunk's project, or its record was retired — ends and routes `failure` as `_refuse` does, which is its
verdict, exactly as the same refusal does at submit.

A step holds nothing when its chunk has no commits (`NOTHING_TO_RESOLVE`), when every repository it resolves to has
already landed, or when its action does not land: whether it does is its catalogue entry's `ActionSpec.lands`
(`hub/domain/graph/actions.py`, the actions epic), never the graph's. `land` lands; `record-findings`, `garden-deliver`,
and `pass` do not, so a chunk recording its review findings or a garden run delivering waits behind no landing,
whichever graph version it is pinned to.

A step refused before it can name its resources holds nothing and is refused at once, before it submits, as `_refuse`
refuses it today. That holds for every hub step, landing or not, exactly as `_run_locked` resolves and refuses for every
hub node now: resolution refuses with `repository-unresolved` or `repository-not-linked`, and `GitCommits.of` (moved by
the actions epic to `hub/domain/actions/visit.py`) with `unconverged-delivery`. One refusal applies only to a step that
lands, since only a landing writes markers: a visit whose resolved repositories share an `owner/repo` is refused with
the actions epic's `hub-log.repository-ambiguous`, at submit and again in each turn's build, as the one refusal it is
([record.md](./record.md#what-the-record-says)). `repository-secret-unavailable` is different: a repository's secret is
revealed when the driver builds the visit for a turn ([below](#building-the-visit)), so a visit that meets it has
already submitted. Its turn ends with outcome `failure` and routes `failure` as `_refuse` does, which is its verdict
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
(the forges epic), which reveals each repository's secret as it builds. A resolution that holds one `owner/repo` for two
records is refused here as the actions epic refuses it, `failure` with `hub-log.repository-ambiguous`, before anything
else is built. Before handing the pass over, it checks each resolved repository's capabilities,
`ForgeRegistry.offered(forge, repository)`, against what `ACTION_REQUIREMENTS` (`hub/domain/actions/requirements.py`)
says the action requires of it; a repository lacking one escalates the chunk with cause `forge-capability-missing`,
records the forges epic's event and `hub-log.forge-capability-missing` artifact, and ends the turn `escalated` without a
pass. These are the steps the actions and forges epics place in `HubNodeExecutor`, moved here unchanged in what they
check and answer.

Building can fail. A secret that cannot be revealed ends the turn `failure`, as above. Any other exception raised while
building — a resolution the code did not expect, a malformed artifact — ends the turn and routes `failure` with its
traceback in the action's `hub-log.<action>` artifact, as the actions epic answers an exception escaping a pass, so a
fault that would recur on every turn reaches the graph's failure edge and its bounce cap rather than retrying forever.
Building can also hang: it reads the work source for the feature title, and a read that never answers would hold the
turn open with no pass to cut off. The build is therefore bounded on its own, at sixty seconds on the turn's clock; a
build still running at its bound ends the turn `cut-off` as a pass past its deadline does, recording a
`turn-interrupted` event that names the build's bound, since no pass ran to overrun ([below](#a-turn-that-cannot-end)).
Only a turn whose end cannot be written at all is left to the driver's sweep.

## Who is next

Each submission carries its tenant's arrival sequence ([record.md](./record.md)). A submission is **eligible** when its
chunk's status is `delivering`, its visit is current — the chunk's newest movement enters the submission's node and its
`latest_epoch` is the submission's epoch — its visit has no open turn, and its next poll is due: no `hub_node_poll` row
for the visit, or `PollPolicy.due_at(row)` of the newest one has passed — the actions epic's one answer to when a
waiting visit is next due, the later of the node's `poll_interval` after the row and the `next_poll_at` its pass
answered with. `PollPolicy` moves from `hub/delivery/hub_node.py` to `hub/domain/delivery/flows.py`, beside `FlowLine`,
which reads it, and `due_at` moves with it unchanged; the `policies` epic gives a poll row its wait kind and `due_at` an
interval per kind. A submission is **next** on a flow when no eligible submission there arrived before it. A visit takes
a turn when it is next on every flow it holds and each of them is free; a visit holding nothing is next whenever it is
eligible.

A visit is keyed as hub-advance keys it today, by the node the chunk stands at and `latest_epoch` — the newest epoch
across its leases, restarts, and epoch owners (`ChunkFacts.latest_epoch`). Two writers can raise that epoch while the
chunk still stands at the node, and each starts a new visit by design: a hub-node escalation, whose hub lease opens the
next epoch ([verdict.md](./verdict.md#escalation)), and a restart onto the same node. A runner never mints a lease for a
chunk standing at a hub node, since a claim is legal only from `ready`. Each new visit is a fresh request to land, and
goes to the back of the line. Nothing else raises the epoch of a chunk standing at a hub node, and a writer added later
that does without meaning to is caught by `hub:flow-line-standing` ([record.md](./record.md#invariants)).

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
`shutdown` event. `_lifespan` awaits the driver's `run()` as it awaits the sweeps (`hub/app.py`), before the
`ActionLoop` stops and before the engine is disposed.

The work splits in two, along the line between what reads the store and what waits on a forge:

- **The tick** is a steppable `tick(store, clock) -> TickResult` (`bzh:steppable-loop`), synchronous and run through
  `asyncio.to_thread`, as every sweep's pass is. For every open tenant it reconciles the lines
  ([verdict.md](./verdict.md#the-reconcile)), ends any turn that cannot end ([below](#a-turn-that-cannot-end)), and
  then, for every visit that may take a turn, oldest arrival first, takes the turn. It answers the turns it started.
  Tests drive one tick at a time and stop at any boundary.
- **A turn's task** is an `asyncio` task the driver creates on the hub's event loop for each turn the tick started. Its
  first act is to add its turn's id to the set of running turns the driver owns ([below](#a-turn-that-cannot-end)), and
  every exit from it, normal or not, passes through one `finally` that removes the id. The task builds the visit through
  `asyncio.to_thread`, submits the pass to the `ActionLoop` and awaits it through `asyncio.wrap_future` on the loop's
  `run_coroutine_threadsafe` future, drains the pass's effects as they come, and, when the pass answers, routes the
  answer and ends the turn in one more `to_thread` write. No store call runs on the event loop, and no transaction is
  held open across a wait.

The pass runs on the `ActionLoop`'s own event loop and the turn's task on the hub's, so an effect crosses between them.
The actions epic has every pass hand its effects through an effect queue that the pass's caller drains, writing each
effect and releasing the action once it is durable, and it names this epic's driver as the caller that takes that role
over. So the turn's task drains its pass's queue, writes each effect through `asyncio.to_thread`, and releases the
action only once the write has committed; it always knows whether a write is in flight, and the pass never resumes ahead
of the record of what it did.

A tick never waits for a turn's task. The driver ticks every two seconds, and a `wake: asyncio.Event` it owns cuts the
wait short, set whenever something could change who is next: a submission, a turn ending, a withdrawal, a pause or
resume, a promote or demote, a requeue, and a hub-advance. A wake within a second of the previous tick waits out the
second, so a fleet of runners each calling hub-advance every thirty seconds never turns the driver into a busy loop. The
tick and every turn's task each catch and log their own failures, so a failing tick skips a cycle and a failing task
ends only its own turn, and neither stops the driver.

Request handlers run on worker threads and wake the driver through `loop.call_soon_threadsafe(wake.set)`, behind a port
the domain names. `ApplyService` (`hub/domain/execution/apply.py`) takes an `IHubStepWake` — `wake() -> None`, declared
in `apply.py` — in place of `IHubNodeExecutor`, which goes with this epic; composition hands it the driver's wake.

A pass is bounded (the hub configuration key `action_pass_seconds`, the actions epic) and spent almost wholly waiting on
a forge, so passes need no pool and no process: each runs on the `ActionLoop`'s one thread. Turns on different flows run
side by side, which is the epic's point: a landing into `blizzard` and one into `blizzard-mock` no longer wait on each
other, and a garden delivery waits on neither. The number of passes in flight is bounded by the number of open flows and
hold-nothing visits a tenant has at once, never by a pool size; each forge adapter's client bounds its own connections.

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

On shutdown the driver stops ticking and cancels its turns' tasks without writing their ends; a cancelled task cancels
its pass's future, and with it the pass on the `ActionLoop`. A turn left open is the next boot's to end `interrupted`
([carry-over.md](./carry-over.md#at-boot)), exactly as after a crash, so shutdown makes no write against an engine about
to be disposed and needs none of the seconds a container is given to stop. The driver stops first in `_lifespan`; the
forges epic then closes each tenant's `ConfigObjectCache`, whose adapters close on the `ActionLoop`, before the
`ActionLoop` stops and the engine is disposed. A merge a cancelled pass had already made is found by the next turn's
fresh pass, which reads the forge and records it.

The driver serves one hub process, as the slot did: the open-turn rule is enforced in the store, but recovering turns at
boot ([carry-over.md](./carry-over.md)) and ending turns that cannot end assume no other process is running them. A
tenant that closes mid-tick raises `TenantClosed` from its write; the driver skips that tenant for the rest of the tick,
and a turn's task whose tenant has closed ends without writing, its turn left to the tenant's teardown.

The tick is two passes under `bzh:probe-gated-pass`. Admission — taking a turn for every visit that may — acts on work
as it comes due, much of it made due by a poll interval elapsing, so it owes no probe. The reconcile converges the lines
from where chunks stand, which is a corpus a change probe can watch: each tenant's reconcile first compares a watermark
— the newest id across the tenant's transitions, escalations, requeues, pause and resume facts, promote and demote
facts, restarts, stops, and completions — with the value its last full reconcile saw, skips the rescan when they match,
and runs in full on every wake and at least once a minute whatever the probe says.

The rescan reads two bounded sets. Its first pass reads the live set (`bzh:live-set-read`): chunks that are not terminal
and whose newest movement enters a hub node, with their facts loaded in bulk (`bzh:bulk-reconstitution`) rather than one
chunk at a time. Its second pass reads the tenant's open submissions, which cannot come from the live set, since the
submissions it must close belong to chunks that have left it. They are read from an arrival floor the driver keeps in
memory per tenant: the lowest arrival among the open submissions its last full reconcile found. Arrivals only grow and a
closed submission never reopens, so every open submission sits at or above that floor, and the read walks
`(tenant_id, arrival)` from it — the line and whatever closed behind its oldest member, never the tenant's whole
history. Boot starts the floor at zero, so the boot reconcile alone reads every submission once, and each full reconcile
raises it. The submissions' chunks outside the live set are then resolved by id through the plural reconstitution.

## Pending

A pass that answers a reserved wait — `pending`, the one wait the actions epic reserves — ends its turn with that
outcome and records its `hub_node_poll` row through `HubNodeRouter.record_wait`, which writes the row as
`_record_pending` does today. Its submission keeps its place and is ineligible until its poll is due, so a branch is
never held through a CI wait: the flow gives the next eligible submission a turn, and the waiting chunk is next again
when its poll is due. Turns on a branch are taken in arrival order; landings complete in whatever order their checks
finish, exactly as they do under the slot. A reserved wait is never a verdict and never routed, so no graph authors an
edge for it and the unroutable rule never applies to it.

The poll timeout and its bounce are unchanged in shape: the driver checks the timeout before it builds the visit and
routes `HubNodeRouter.route_pending_timeout` without calling the action, against the timeout the actions epic sets,
three hours by default and in every shipped graph. The timeout runs from the visit's first `pending` row, as today, and
it is wall clock: the time a chunk spends behind turns on its branch counts, as the time it spent waiting for the slot
did. That wait is short beside the bound, since a chunk is behind only the turns of submissions that arrived before it,
each a single pass bounded at five minutes and most far shorter.

## A turn that cannot end

A turn ends in its task's `finally`. If the write that ends it fails — a dropped connection, a lock timeout — or the
task that would make it never starts, the turn stays open, and while it is open nobody else on its branch moves. The
slot's answer was to reclaim itself after `DEFAULT_SLOT_STALE_AFTER`, even while the step that held it could still be
running; a turn never needs that, because the driver knows which turns it is running.

`FlowDriver` keeps the set of turn ids whose task is running, guarded by a lock. A turn's task adds its id as its first
act and removes it in its `finally` — after the turn's end has committed, or after the attempt to write it has failed —
so the set holds exactly the turns some task is answering for. A turn is taken in its own transaction, and its task is
created as soon as the tick that took it returns; a tick that raises after committing one turn and before returning
leaves that turn with no task at all. So each tick, before admitting anyone, ends every open turn in the store that
started before the previous tick began and is not in the set: by then its task has either registered or will never
exist. Such a turn ends `interrupted`, with a `turn-ended` fact on each flow it held and a `turn-interrupted` event
(warning), and with any withdrawal a lever deferred to it ([levers.md](./levers.md)). Its answer, if it had one, is lost
with the failed write, and its submission is eligible again at once: its next turn takes a fresh pass, which skips
whatever its markers say is done, as a re-run does today, and answers again.

A pass that never answers is bounded by the driver as well as by the actions epic's cancellation. The `ActionLoop`
cancels an overrunning pass at its next `await`, and the pass answers as an overrun, as the actions epic has it; but a
pass stuck in synchronous code never reaches one, and while it is stuck it holds the `ActionLoop`'s one thread, so no
other pass runs either. The actions epic gives the caller a deadline of its own for that, the pass bound plus thirty
seconds, and the driver, which is the caller from this epic on, keeps it, adding a bound on the build: it bounds each
turn on its own clock, from the moment the turn starts, the build at sixty seconds ([above](#building-the-visit)) and
the pass at the pass bound plus thirty seconds from when it is handed over. A task still waiting at the pass's deadline
cuts the turn off, in this order. It closes the turn's forge handles first: the role handles the driver bundles into
each `RepositoryForge` for the turn check a flag the turn owns before they send a request, and refuse once it is set, so
nothing the pass does afterwards reaches a forge. The flag lives on the turn's handles, never on the adapter
`ForgeRegistry` caches for a repository, which every turn landing there shares. It closes the pass's sink, as the
actions epic's caller does, so a record the pass makes afterwards raises into it and writes nothing. It waits for the
effect write in flight, if there is one, so a write already running cannot commit under a turn that has ended. Then it
cancels the pass's future and records the actions epic's `hub-action-overran` event, naming the action, the chunk, and
the bound. The actions epic's caller answered a pass let go at its deadline as an overrun, routing `pending` or
`failure`; once the driver owns passes, its rule replaces that one. It routes nothing: it ends the turn `cut-off`
itself, the submission keeps its place, and the visit is eligible again, until three cut-offs in a row send it to a
person, as the next paragraph says. A request already on the wire when the flag is set cannot be recalled; whatever it
did at the forge is found by the next turn's fresh pass, which reads the forge before it acts, as after a crash.

A pass that hangs once may have met a forge on a bad minute; one that hangs every time will never answer, and each turn
it takes holds the `ActionLoop`'s thread for the whole hub until its deadline. So a visit whose last three turns all
ended `cut-off` takes no fourth: its next turn escalates the chunk to a person with the cause `hub-action-stuck`
(`foundation/escalation_causes.py`, with its sentence in the board's `CAUSE_SENTENCES`), naming the action and the
bound, and ends `escalated` ([verdict.md](./verdict.md#escalation)). A turn ended `interrupted` — by a hub that stopped,
or a turn end that failed to write — does not count, since neither is the pass's doing. A stuck loop also keeps every
later pass from starting, which the operator sees as a run of `hub-action-overran` events naming the bound.

`DEFAULT_SLOT_STALE_AFTER` and its injection are removed.

## Where routing lives

`HubNodeExecutor.run` and `IHubNodeExecutor` go with this epic, and the slot with them; what decided where an answer
goes does not. `hub/delivery/hub_node.py` keeps `HubNodeRouter`, holding what the executor's private methods did, now
public and each taking the transaction the turn's end opens: `route` (today's `_route`), `refuse` (`_refuse`),
`route_pending_timeout` (`_route_pending_timeout`), and `record_wait` (`_record_pending`, generalized to a reserved
wait). The turn's task calls them; nothing else does. `DEFAULT_POLL_TIMEOUT` moves with `PollPolicy` to
`hub/domain/delivery/flows.py`, and `PollPolicy`'s two importers outside it, `hub/api/fleet.py` and
`hub/api/chunk_views.py`, follow the move.

## Crash points

Each window between a durable write and the next is a crash point in the `flow.` family (`bzh:crash-point-registry`,
`hub/delivery/flow_driver.py`), armed by a scenario of the kill-9 sweep (`tests/crash/test_kill9_sweep.py`):

| Crash point                         | Durable                                     | Not yet                       | Recovered by                                                       |
| ----------------------------------- | ------------------------------------------- | ----------------------------- | ------------------------------------------------------------------ |
| `flow.after-submit.before-turn`     | The submission, its flow rows, `submitted`  | Any turn                      | The next tick takes the turn; the reconcile never submits it twice |
| `flow.after-turn-start.before-pass` | The turn and its `turn-started` facts       | The pass                      | Boot recovery ends the turn `interrupted`; the next turn passes    |
| `flow.after-poll.before-turn-end`   | A `pending` pass's `hub_node_poll` row      | The turn's end                | Boot recovery ends the turn; the poll row keeps the chunk waiting  |
| `flow.boot.after-interrupt`         | One tenant's open turns ended `interrupted` | The next tenant's, the driver | The next boot ends the rest; ending a turn twice writes nothing    |

`flow.after-poll.before-turn-end` takes the place of `hubnode.after-poll.before-slot-release`, which leaves the registry
and the kill-9 sweep in the chunk that stops acquiring the slot, since nothing can reach it after that
([index.md](./index.md#landing-order)). The windows inside a pass — `action.after-effect.before-write` and
`action.after-write.before-resume` — stay the actions epic's, armed the same way now that the turn's task is the writer.
Two windows the slot had are closed rather than armed. A verdict's transition and its turn's end are one transaction
([verdict.md](./verdict.md#the-verdict)). A landing's `merged/<repo>` marker, its `delivery_repo_landed` row, its close
intents, its flow's `landed` fact, and its `flow-landed` event row are one transaction
([record.md](./record.md#what-the-record-says)); the live publish of that event follows the commit and, like every live
frame, is not durable.
