# Carry-over

The epic lands on a fleet whose runners take any work at any hour, whose only stops are breakers, and whose usage limits
brake every runner spending the plan until a person lifts each one. It leaves a fleet whose runners wait behind their
subscription's floor and their own levers and carry on by themselves, whose breakers count the same dollars the board
shows, and whose runners can put a parked chunk back in the queue for whichever runner claims it next. Nothing deployed
needs re-enrolling, loses a chunk, or changes id.

## The order of landings

Each piece lands on its own, leaves `master` green, and is proven by the methods it names. A piece that changes
`blizzard-context` lands that half there, proven by `blizzard-context:reference-lint`,
`blizzard-context:markdown-format`, `blizzard-context:markdown-lint`, and `blizzard-context:registry-drift`, and no half
cites a rule or method that has not landed. A piece that changes the wire extends `blizzard-mock` in the same change,
proven by `blizzard-mock:unit-test` (`bzh:wire-change-extends-mock`).

1. **The gate and the floor.** The usage limit stops being a breaker, and the one throttle every runner has arrives with
   the question every start asks. `runner/throttle/gate.py` and `floor.py` with a verdict of the floor alone;
   `StartKind` on `Spawner.suppressed` at every start site, and the registry test's new rule; FILL's one call and
   `InterruptedClaims`' widened `braked` ([gate.md](./gate.md)). The runner store's additive revision adding
   `subscription_usage_limits` and `runner_usage_limits`; the `usage_limits` host member and its runner twin; `_mark` in
   place of `_engage`; the retired fan-out and everything only it used; the renamed crash points and the rewritten
   crash-correctness section; the sampling pass's two early-sample triggers; and the retention prune
   ([exhaustion.md](./exhaustion.md)). `docs/deployment/spend.md` §Usage limits and §External subscription usage, and
   `docs/upgrade.md`'s first one-time step (§What an operator does once). Its context half:
   `domain/execution/throttling.md` with `bzh:subscription-floor-reach` and the routing row in `execution.md` that names
   it, `pause.md` without the usage-limit engagement or `bzh:subscription-brake-reach`, and `hosts.md`'s repointed
   sentence. Proven by `blizzard:unit-test` over the floor's derivation at every edge of a hold — reset ahead, passed
   and unsampled, passed and sampled open, a mark naming no reset, an unsampled subscription, a runner naming none;
   `blizzard:service-test` with two runners of one host spending one plan and a third on another, a limit met by the
   first holding back the first two and not the third, and all of them claiming again, and the limited lease waking,
   after a fresh open sample with nobody acting; `blizzard:crash-sweep` over the renamed points;
   `blizzard:store-contract`; and `blizzard:manual-runner` against a harness made to report a limit.
2. **The settings in view.** Waits for piece 1, and lands right after it, so the floor an operator cannot configure is
   visible the moment it starts holding runners back. `ThrottleStatusView` on `RunnerStatusView` and `HostRunnerView`,
   built by `RunnerStatusService`; the `throttle` block of `blizzard runner status`; the machine page's `throttle` panel
   with its container and view; and the host page's lines, through `throttleLines` ([surfaces.md](./surfaces.md) §The
   runner's own status, §`blizzard runner status`, §The runner's panel). With no lever yet it shows the floor and both
   hold-or-release choices at `hold`; each later piece adds its lever's line in the change that adds the lever.
   `docs/deployment/control-verbs.md` names the block. Proven by `blizzard:unit-test` over the view in every floor
   state, `web:client-drift`, `web:unit-test`, `web:lint`, and `web:manual-panel` on a runner whose subscription is
   exhausted and on one naming none.
3. **The reason at the hub.** Its hub half — the `declining` column, the views' field, the board's label, and the hub
   CLI's segment — waits for nothing and may land first, since a hub that stores a field no runner sends shows nothing.
   Its runner half, `declining` sent on every registration, waits for piece 1 and for the hub half
   ([surfaces.md](./surfaces.md) §The hub, the board, and the hub's CLI). Its context half: `responsibilities.md` names
   the declining report beside capabilities, as the runner's own assertion superseded whole on every registration.
   Proven by `blizzard:unit-test`, `blizzard:service-test` against both an older hub and an older runner,
   `blizzard:compat`, `web:client-drift`, `web:unit-test`, and `web:manual-board`.
4. **One way of counting dollars.** Waits for nothing, and lands after piece 1 by choice, so that the change in what the
   breakers see reaches the fleet with the levers that let an operator plan around it close behind. `UsageTotals` and
   `usage_since`, the chunk cap's reading, the shared rendering in `foundation/` and its callers' move onto it, the
   rewritten prose, `docs/deployment/spend.md` §The two caps, §Partial totals, and §Estimated cost
   ([dollars.md](./dollars.md)), and `docs/upgrade.md`'s second one-time step. Proven by `blizzard:unit-test` over a
   window of billed rows, estimated rows, and rows with neither, each reading its PARTIAL; `blizzard:service-test` with
   an OpenCode runner billed zero tripping its ceiling on its estimate; and `blizzard:manual-runner`.
5. **The reading levers and the schedule.** Waits for pieces 1 and 2. The `[runner.throttle]` table and its `pace`
   sub-table with its optional `taper` steps, and its `utilization` and `schedule` sub-tables, their keys in the
   registry and the regenerated `contracts/config/host.json`, `ThrottleSettings` and its parser, their validation, and
   the renderer's new tables ([configuration.md](./configuration.md)); the three lever functions and the gate asking
   them ([levers.md](./levers.md)); their status lines; and `docs/deployment/spend.md` §Throttles. Its context half: the
   levers and the gate's order in `throttling.md`. Proven by `blizzard:unit-test` over each lever's answer and `until` —
   pacing at the plan's five-days-in example, a taper's effective lead below, at, and between its steps with the step
   named in the answer, a taper out of order refused at load, a window missing or stale, a schedule span across midnight
   and across a daylight-saving change; `blizzard:config-contract`; `blizzard:service-test` with a held chunk declined
   mid-graph and entered once the gate opens, its retries untouched; and `blizzard:manual-runner`.
6. **The dollar budget.** Waits for pieces 4 and 5. The `budget` sub-table and its keys, its validation against the
   ceiling, the lever, and its status line. Proven by `blizzard:unit-test`, `blizzard:config-contract`, and
   `blizzard:service-test` with a runner throttled at its budget and carrying on as its spend ages out of the window,
   and a runner spending through it tripping its ceiling and staying tripped.
7. **Hold or release.** Waits for pieces 1 and 5: the release is offered where the gate's decline is read, and the two
   settings live in the throttle table. It reads nothing of piece 3's declining report, so it may land beside or before
   it. `gate_type` and `usage_limit_gate_type` and their validation; the hub's route, `ReleaseService`,
   `ChunkVerb.RELEASE`, `route_released`'s two columns, and the activity feed's wording; `IHubClient.release`,
   `Release.offer` at the two entry sites and from `DormantSession.on_unpause` for a usage-limited park,
   `pause_parks.cause` in the runner store's additive revision, the abandon that closes a released usage-limited
   attempt, and the two registered crash points ([release.md](./release.md)). Its context half: `recovery.md` §Release
   and `acquisition.md` §Giving tenure back. Proven by `blizzard:unit-test` over `refuse_release`;
   `blizzard:service-test` with one runner set to release putting a parked chunk back `ready` and a neighbour whose gate
   admits claiming it, the same chunk left `ready` while every neighbour declines and claimed cold by its first runner
   once its gate reopens, and a usage-limited lease on a runner set to release claimed elsewhere and entered from its
   start with no retry spent, the half-done step's usage booked, and nothing left to wake on the first runner;
   `blizzard:crash-sweep` over `release.after-hub.before-environments` and
   `release.usage-limited.after-hub.before-abandon`; `blizzard:store-contract`; `blizzard:compat`; and
   `blizzard:manual-runner` with one runner of the fleet set to release.

The epic is delivered when piece 7 lands.

## Hub and runners on different versions

Every wire change is additive in both directions (`bzh:fleet-wire-additive`), so a hub and its runners can run versions
on either side of any piece. A runner that predates piece 3 sends no `declining`, and the hub shows it declining
nothing. A hub that predates piece 3 ignores the field a newer runner sends. A hub that predates piece 7 answers the
release route `404`, which a runner reads as a refusal and holds ([release.md](./release.md) §The runner's side).

## What an operator does once

Two changes reach a running fleet that the fleet cannot make for itself, and `docs/upgrade.md` names each for the
version of blizzard that carries it.

**A brake a usage limit engaged before piece 1.** The runner's own brake is a fact only an operator clears, and
upgrading blizzard does not clear one by itself. A runner braked by a usage limit when piece 1 reaches it stays braked,
its reason still reading `usage limit: …`, until the operator runs `blizzard runner start --runner <id>` for it. From
then on the floor holds it for as long as the subscription is exhausted, and its parked leases wake when the floor
opens.

**Breakers that start seeing estimates.** From piece 4, `runner_ceiling_usd` and `chunk_cap_usd` count estimated spend
they were blind to, so a runner whose work is mostly estimated — an OpenCode runner on a ChatGPT subscription above all
— reaches them sooner than it did. Before moving onto that version, an operator reviews each runner's `[runner.cost]`
against its recent `~`-marked spend on the board, and raises a cap that was only ever set against billed spend.
