# Release contract

## The fleet's mid-graph release

A **release** is the fleet's one way for the runner holding a chunk to let go of it mid-graph, so that another runner —
or the same one, later — picks it up from the queue. It is built once and for general use: the route and the service
know nothing about why a caller lets go, each caller brings a reason string and a call site of its own, and no caller's
logic lives in either. Throttling is its first caller, with two reasons — `throttled`, for a chunk the gate parks
between steps, and `usage-limited`, for a lease a usage limit stops mid-step. `epic:resilience` is its second, letting
go of held work when a provider is down (§Further callers). The fleet means to rely on it more broadly, as the way an
agent's work changes hands mid-graph reliably; nothing below is shaped to throttling alone.

A release puts the chunk back in the queue `ready` at its current node, whether or not any other runner could take it
now. Whichever runner's gate next admits a claim takes it — possibly this same runner once its own gate reopens — and
enters the node with a cold session in fresh environments. A runner whose own gate declines does not claim, so a
neighbour waiting on the same exhausted subscription simply leaves the chunk where it is; nothing at the hub has to
choose who may take it. Losing the warm session and the environment is the accepted price of a release.

## Hold or release

`[runner.throttle] on_decline` ([configuration.md](./configuration.md)) decides what a chunk does when the gate parks it
between steps: `hold`, the default, keeps it where it is — its route, environments, and warm session — until this
runner's gate reopens, as [gate.md](./gate.md) §The line holds mid-graph describes; `release` puts it back in the queue.
The setting is per runner, so one runner can be set to release while the rest of the fleet holds.

`on_decline` reaches only a chunk the gate holds between steps: one whose next start is a `STEP` the floor or a lever
declines, with no lease open. It never reaches a chunk a breaker holds, one an operator paused, one parked at a human
gate or held by a takeover, or one at a hub node, all of which hold as they do today.

A lease a usage limit stops mid-step answers to a second setting, `[runner.throttle] on_usage_limit`, with the same two
values and the same default, `hold`. Holding, the lease stays parked in place until its floor opens
([exhaustion.md](./exhaustion.md) §A lease limited mid-step). Releasing it costs more than a release between steps: the
step the session had half done is abandoned, its partial work goes with the environments the runner gives up, and the
runner that claims the chunk enters the node from its start with a cold session (§Releasing a usage-limited lease). That
loss too is accepted. Neither kind of release consumes a retry.

## At the hub

A release is the fourth write that gives tenure back, beside detach, the hub's requeue of an escalated chunk, and a
migration (`domain/execution/acquisition.md` §Giving tenure back). It behaves as detach does in everything but who asks:
the route is freed, no requeue fact is written, no epoch is bumped, and nothing is superseded, so the chunk re-derives
`ready` at its current node, keeps its place in the queue, and the next claim's reserved epoch fences the old holder
out. The next holder counts its attempts at the node from its own store, as after any detach.

**The route.** `POST /api/fleet/chunks/{chunk_id}/release`, in `hub/api/fleet.py` beside the route-token and completion
routes (`:740`, `:762`), under the runner's own token and carrying the chunk's route token as a completion does. Its
body is `ReleaseRequest`: the `epoch` the runner last saw the chunk at, and a `reason`, carried as an open string the
hub records and never interprets, so a new caller adds a reason without a change to the route. The hub answers:

- `200` with the new `route_released` row's id;
- `409` with `ReleaseRefusal` naming `stale` when the chunk has moved past `epoch`;
- `403` when the caller does not hold the chunk's route, or its route token does not match;
- `404` for a chunk the tenant does not hold.

Those are the only refusals: each guards the integrity of who holds the chunk at which epoch, and none judges whether
letting go is wise. That judgment is the caller's, made on its own runner before it asks.

**The rule.** `ReleaseService` in `hub/domain/execution/release.py` orchestrates, beside `DetachService`
(`hub/domain/execution/detach.py:54`), and the decision is a value on the model (`bzh:domain-orchestration-split`). A
new verb, `ChunkVerb.RELEASE`, joins `CHUNK_VERB_LEGALITY` (`hub/domain/chunk/model.py:900`), legal from `running`
alone. `refuse_release` takes the route, the chunk's facts, the caller, and the epoch, and refuses unless the caller
still holds the route (`releasable_by`, `detach.py:48`), the verb is legal from the derived status, and the epoch is the
chunk's latest. The service takes the chunk's exclusive lock, re-reads the route and facts under it, decides, and
writes, as `DetachService.detach` does (`detach.py:71`–`75`, `bzh:store-exclusive-write`). Neither reads the reason
beyond recording it.

**The record.** `route_released` (`hub/store/schema.py:1191`) gains two nullable columns: `cause` — `detach`,
`retirement`, or `release`, null on rows written before it — and `reason`, the caller's string on a release. Detach and
retirement's pass over a retiring runner's chunks (`hub/domain/execution/fleet.py:252`) write their own cause. The
activity feed's route-released entry (`hub/store/internal/chunk_events_store.py:217`) reads a release as
`released by r-claude (throttled)`, so a chunk's timeline says who let it go and why.

## The runner's side

`Release.offer` in `runner/throttle/release.py` is the runner half every caller shares — a step module with its own
context Protocol and sentinel (`bzh:narrow-seams`), taking the chunk, the epoch, the reason, and what to do once the hub
has accepted. It calls the new `IHubClient.release`. On `200` it runs the caller's completion; on a refusal, an
unreachable hub, or any `404` it leaves everything as it was, and the caller offers again on its next tick. A `404` here
never lets go of the environments: the chunk being unknown is learned from the runner's own reads of it, which already
let go of a chunk the hub no longer holds (`held_chunk.py:70`–`73`). No outcome writes a runner fact.

**Between steps.** The gate's caller offers from the two places a held chunk's node is entered each tick —
`HeldChunk._spawn_advanced_node` (`runner/lifecycle/held_chunk.py:135`) and FILL's adopt
(`runner/lifecycle/claim.py:404`) — before either fetches the envelope. Each asks `Spawner.suppression` for a `STEP`
([gate.md](./gate.md) §Every start asks it); when the cause is the gate's decline and the runner's `on_decline` is
`release`, it calls `Release.offer` with the reason `throttled` instead of entering. Its completion gives up the chunk's
environments (`EnvironmentRelease.release_chunk`) and invalidates the chunk's cached view; no lease is open to close.

The hub's write and the runner's giving up of its environments are two transactions in two processes, and a `kill -9`
between them is a real window, registered as `release.after-hub.before-environments` and swept by
`tests/crash/test_kill9_sweep.py` (`bzh:crash-point-registry`). It closes without help: the restarted runner holds a
binding for a chunk with no route, which `InterruptedClaims` already answers — letting the binding go once another
runner holds the route, claiming the chunk again once this runner's gate admits a claim, and holding it while neither
(`interrupted_claim_move`, `runner/lifecycle/model.py:306`).

## Releasing a usage-limited lease

A usage-limited park is told apart from an operator's per-chunk pause park by a cause of its own: `pause_parks`
(`runner/store/schema.py:331`) gains a nullable `cause` column, which `Attempt.park_usage_limited`
(`runner/lifecycle/attempt.py:449`) writes as `usage-limit` and `PausePark` (`runner/throttle/pause.py:209`) carries. A
park written before the column reads as an operator's and holds.

**Where it is offered.** From `DormantSession.on_unpause` (`runner/lifecycle/dormant.py:231`), which already visits
every open pause park each tick (`runner/loop/steps.py:517`–`526`). After the park's teardown has settled and the
chunk's view is read, a park whose cause is `usage-limit`, whose chunk no operator has paused, with no open ask and no
takeover holding it, and whose wake `Spawner.suppression` declines for a `CONTINUATION` on the floor alone — not on a
breaker — is offered through `Release.offer` with the reason `usage-limited` when the runner's `on_usage_limit` is
`release`. The registry test admits `suppression` as the gate call at this site. A park whose floor has opened is woken,
not offered: there is nothing left to wait out.

**How the attempt closes.** The completion first books whatever usage the parked invocation has not yet booked, as a
wake does before it mints the next generation (`ctx.usage.record_worker`, `dormant.py:300`), so the half-done step's
spend is never lost; the write replays as a no-op under its `(lease, generation, kind)` key. It then abandons the
attempt through `Attempt.abandon(killed=True, via="release")` (`attempt.py:369`): the worker has already exited, any
standing judge elicitation record is cleared, the environments are given up with whatever the step left in them, and the
lease closes `released` (`foundation/leases.py:23`) — no epoch bump and no requeue, so no retry is spent. A closed lease
closes its park (`IReadPauseRepository.open_pause_parks`, `pause.py:254`), so nothing on this runner wakes it again.

**Refused.** A `stale` refusal, an unreachable hub, or any `404` leaves the park exactly as it was: it keeps holding, is
offered again on the next tick the floor still declines, and wakes in place once the floor opens.

**Across a crash.** The hub's write and the runner's abandon are two transactions in two processes, a real window
registered as `release.usage-limited.after-hub.before-abandon`. It closes by a path that exists today: the restarted
runner still holds an open, pause-parked lease on a chunk whose route is no longer its own, and PULL's reconcile
abandons exactly that lease (`lease_reconcile_move`, `runner/loop/steps.py:358`–`366`). Inside the abandon, its own
points (`_CP_ABANDON_AFTER_KILL`, `_CP_ABANDON_AFTER_RELEASE`) already cover a crash between its writes, and the usage
record ahead of it replays as a no-op.

## Further callers

`epic:resilience` lets go of held work when a provider is down and another route exists. It takes the release whole —
the route, the service, the verb, the record, and `Release.offer` — and adds only its own reason and its own call site,
deciding on its own runner when letting go is worth the cold session, as throttling decides by its two settings. Every
later caller does the same. None gives tenure back any other way, and none adds a refusal, a reason the hub interprets,
or a branch of its own to the route or the service, so the fleet keeps one release whose behaviour every caller can rely
on.

`blizzard-context` records the write as `domain/execution/recovery.md` §Release, and `acquisition.md` §Giving tenure
back counts it among the writes that give tenure back. `blizzard-mock`'s mock hub serves the route and records the
write, and `tests/service/test_parity_guard.py` holds `IHubClient.release` to it (`bzh:wire-change-extends-mock`).
