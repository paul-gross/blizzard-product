# Hand-back contract

## Hold or release

`[runner.throttle] gate_type` ([configuration.md](./configuration.md)) decides what a chunk does when the gate parks it
between steps: `hold`, the default, keeps it where it is — its route, environments, and warm session — until this
runner's gate reopens, as [gate.md](./gate.md) §The line holds mid-graph describes; `release` hands it back to the hub,
so a runner whose subscription still has room claims it and enters its node with a cold session there. The setting is
per runner, so one runner can be set to release while the rest of the fleet holds.

Release reaches only a chunk the gate holds between steps: one whose next start is a `STEP` the floor or a lever
declines, with no lease open. It never reaches a chunk a breaker holds, one an operator paused, one parked at a human
gate or held by a takeover, or one at a hub node, all of which hold as they do today.

A lease a usage limit stops mid-step answers to a second setting, `[runner.throttle] usage_limit_gate_type`, with the
same two values and the same default, `hold`. Holding, the lease stays parked in place until its floor opens
([exhaustion.md](./exhaustion.md) §A lease limited mid-step). Releasing hands its chunk back through the same hand-back,
and costs more than a release between steps: the step the session had half done is abandoned, its partial work goes with
the environments the runner lets go of, and the runner that claims the chunk enters the node from its start with a cold
session (§Releasing a usage-limited lease). Neither release consumes a retry.

## The hand-back

The release is a new write at the hub, the **hand-back**: the runner holding a chunk gives its tenure back, between
steps or from a step a usage limit stopped. It is the fourth write that gives tenure back, beside detach, the hub's
requeue of an escalated chunk, and a migration (`domain/execution/acquisition.md` §Giving tenure back), and it behaves
as detach does in everything but who asks: it releases the route, writes no requeue fact, bumps no epoch, and supersedes
nothing, so the chunk re-derives `ready` at its current node, keeps its place in the queue, and the next claim's
reserved epoch fences the old holder out. The next holder counts its attempts at the node from its own store, as after
any detach.

**The route.** `POST /api/fleet/chunks/{chunk_id}/hand-back`, in `hub/api/fleet.py` beside the route-token and
completion routes (`:740`, `:762`), under the runner's own token and carrying the chunk's route token as a completion
does. Its body is `HandBackRequest` — the `epoch` the runner last saw the chunk at, and a `reason` — `throttled` for a
chunk the gate parks between steps, `usage-limited` for a lease a usage limit parked mid-step — carried as a string so
another reason can join them without a version of its own. The hub answers:

- `200` with the release's id when the route is released;
- `409` with `HandBackRefusal` naming `stale`, the chunk having moved past `epoch`, or `no_runner_admitting`;
- `403` when the caller does not hold the chunk's route, or its route token does not match;
- `404` for a chunk the tenant does not hold.

**The rule.** `HandBackService` in `hub/domain/execution/hand_back.py` orchestrates, beside `DetachService`
(`hub/domain/execution/detach.py:54`), and the decisions are values on the model (`bzh:domain-orchestration-split`). A
new verb, `ChunkVerb.HAND_BACK`, joins `CHUNK_VERB_LEGALITY` (`hub/domain/chunk/model.py:900`), legal from `running`
alone. `refuse_hand_back` takes the route, the chunk's facts, the caller, and the epoch, and refuses unless the caller
still holds the route (`releasable_by`, `detach.py:48`), the verb is legal from the derived status, and the epoch is the
chunk's latest. `admitting_elsewhere` takes the chunk, its current node, and the tenant's registrations, and says
whether some other runner could take it now: online, not retired, neither brake engaged, declining nothing by its latest
registration ([surfaces.md](./surfaces.md) §The hub, the board, and the hub's CLI) — a runner that predates the report
declines nothing, since it has no throttle — serving the chunk's project, and fitting its node by the same capability
fit a claim is judged on (`hub/domain/execution/eligibility.py`). When none can, the hand-back is refused
`no_runner_admitting` and the chunk stays where it is, because a cold session is worth paying only when someone is there
to start it. The service takes the chunk's exclusive lock, re-reads the route and facts under it, decides, and writes,
as `DetachService.detach` does (`detach.py:71`–`75`, `bzh:store-exclusive-write`).

The registrations are as fresh as each runner's last tick, so a runner judged admitting may be declining by the time it
peeks. The chunk then waits `ready` until a runner that admits claims it — this runner among them, once its own gate
reopens. Nothing is lost: the chunk was going to wait either way.

**The record.** `route_released` (`hub/store/schema.py:1191`) gains two nullable columns: `cause` — `detach`,
`retirement`, or `hand-back`, null on rows written before it — and `reason`, the hand-back's. Detach and retirement's
release pass (`hub/domain/execution/fleet.py:252`) write their own cause. The activity feed's route-released entry
(`hub/store/internal/chunk_events_store.py:217`) names a hand-back as one, with its runner and reason, so a chunk's
timeline says why it changed hands.

## The runner's side

The hand-back is offered from the two places a held chunk's node is entered each tick — `HeldChunk._spawn_advanced_node`
(`runner/lifecycle/held_chunk.py:135`) and FILL's adopt (`runner/lifecycle/claim.py:404`) — before either fetches the
envelope. Each asks `Spawner.suppression` for a `STEP` ([gate.md](./gate.md) §Every start asks it); when the cause is
the gate's decline and the runner releases, it calls `HandBack.offer` in `runner/throttle/hand_back.py` — a step module
with its own context Protocol and sentinel (`bzh:narrow-seams`) — instead of entering. `offer` calls the new
`IHubClient.hand_back`, and on `200` releases the chunk's environments (`EnvironmentRelease.release_chunk`) and
invalidates its cached view; on a refusal, an unreachable hub, or any `404` it holds, and the next tick offers again. A
`404` here never releases the environments: the chunk being unknown is learned from the runner's own reads of it, which
already release a chunk the hub no longer holds (`held_chunk.py:70`–`73`). Neither outcome writes a runner fact.

The hub's release and the runner's release of its environments are two transactions in two processes, and a `kill -9`
between them is a real window, registered as `handback.after-release.before-env-release` and swept by
`tests/crash/test_kill9_sweep.py` (`bzh:crash-point-registry`). It closes without help: the restarted runner holds a
binding for a chunk with no route, which `InterruptedClaims` already answers — releasing it once another runner holds
the route, claiming it again once this runner's gate admits a claim, and holding it while neither
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
breaker — is offered back through `HandBack.offer` with the reason `usage-limited` when the runner's
`usage_limit_gate_type` is `release`. The registry test admits `suppression` as the gate call at this site. A park whose
floor has opened is woken, not offered: there is nothing left to wait out.

**How the attempt closes.** On the hub's `200`, the runner first books whatever usage the parked invocation has not yet
booked, as a wake does before it mints the next generation (`ctx.usage.record_worker`, `dormant.py:300`), so the
half-done step's spend is never lost; the write replays as a no-op under its `(lease, generation, kind)` key. It then
abandons the attempt through `Attempt.abandon(killed=True, via="hand-back")` (`attempt.py:369`): the worker has already
exited, any standing judge elicitation record is cleared, the environments are released with whatever the step left in
them, and the lease closes `released` (`foundation/leases.py:23`) — no epoch bump and no requeue, so no retry is spent.
A closed lease closes its park (`IReadPauseRepository.open_pause_parks`, `pause.py:254`), so nothing on this runner
wakes it again.

**Refused.** `no_runner_admitting`, `stale`, an unreachable hub, or any `404` leaves the park exactly as it was: it
keeps holding, is offered again on the next tick the floor still declines, and wakes in place once the floor opens. A
runner on the same subscription is never the one judged to have room, since its own registration declines on the same
floor; a runner on another subscription, or on none, can be.

**Across a crash.** The hub's release and the runner's abandon are two transactions in two processes, a real window
registered as `handback.usage-limited.after-release.before-abandon`. It closes by a path that exists today: the
restarted runner still holds an open, pause-parked lease on a chunk whose route is no longer its own, and PULL's
reconcile abandons exactly that lease (`lease_reconcile_move`, `runner/loop/steps.py:358`–`366`). Inside the abandon,
its own points (`_CP_ABANDON_AFTER_KILL`, `_CP_ABANDON_AFTER_RELEASE`) already cover a crash between its writes. The
usage record ahead of it replays as a no-op.

## What `epic:resilience` reuses

Resilience hands held work to another runner when a provider is down and another route exists. It takes the hand-back
whole — the route, the service, the verb, the record, and the runner's `HandBack.offer` — adding its own `reason` and
its own call site. What it may not do is give tenure back any other way, so the fleet keeps one write for a runner
letting go of a chunk. `admitting_elsewhere` is reason-agnostic — another runner able to start the chunk now — and
resilience narrows it, if it needs to, through what each runner already reports, never through a second route.

`blizzard-context` records the write as `domain/execution/recovery.md` §Hand-back, and `acquisition.md` §Giving tenure
back counts it among the releasing writes. `blizzard-mock`'s mock hub serves the route and records the release, and
`tests/service/test_parity_guard.py` holds `IHubClient.hand_back` to it (`bzh:wire-change-extends-mock`).
