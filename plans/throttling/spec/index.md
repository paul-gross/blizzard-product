# Throttling specifications

The technical contracts behind `epic:throttling`. Product intent begins at the [epic plan](../index.md): that every
lever is a throttle and the existing stops stay breakers, that the levers are the runner's own configuration, that a
subscription's exhaustion is the subscription's and clears itself — on a sample, on the grace when no sample comes, or
by an operator's hand — that the gate is asked before every claim and every step, that a held chunk may wait or be let
go, that every dollar line counts the same figure, that a declining runner says why while leaving no fact behind, and
that the runner shows its own settings. This spec states how each of those is built and redefines none of it.

The subscription a runner names is trusted as declared, as it is today
([`delivered/runner-host`'s subscriptions contract](../../../delivered/runner-host/spec/subscriptions.md)
§Declarations): nothing here binds a harness's credentials to the subscription its runner names, and nothing checks that
they agree.

Paths are relative to `blizzard/src/blizzard/` unless a path names another root: one beginning `tests/`, `docs/`,
`contracts/`, `openapi/`, or `web/` is relative to the `blizzard` repository's root. Line citations are to `master` as
this spec was written; they anchor a decision to the code it changes, and a moved line does not change the decision.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`, `domain/execution/pause.md` loses the usage-limit
engagement of the runner's own brake and `bzh:subscription-brake-reach` with it, and a new spoke,
`domain/execution/throttling.md`, takes the gate, its levers, the subscription floor, and the rule that replaces the
retired one (`bzh:subscription-floor-reach`, [exhaustion.md](./exhaustion.md)); `hosts.md` §Subscriptions and
`responsibilities.md` point to it, `recovery.md` and `acquisition.md` §Giving tenure back gain the release
([release.md](./release.md)), and `architecture/crash-correctness/runner.md` §The usage-limit brake fan-out becomes the
usage-limit mark. Blizzard's operator documents carry what an operator configures and reads: `docs/deployment/spend.md`
rewrites its caps, its usage-limit pause, its partial totals, and its estimate's reach ([dollars.md](./dollars.md),
[exhaustion.md](./exhaustion.md)) and gains the throttles; `docs/deployment/control-verbs.md` names what `runner status`
prints; `docs/upgrade.md` carries the operator's one-time steps ([carry-over.md](./carry-over.md)). The generated
documents follow the code: `contracts/config/host.json` from the runner's key registry, and `openapi/hub.openapi.json`
and `openapi/runner.openapi.json` from the wire. `blizzard-mock`'s mock hub and mock runner mirror every field and route
the wire gains (`bzh:wire-change-extends-mock`).

| Where                                  | Read when                                                                                                                                                                     |
| -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [configuration.md](./configuration.md) | Shaping or validating a runner's `[runner.throttle]` table — every key, its default, and what the key registry and its contract document gain                                 |
| [gate.md](./gate.md)                   | Building the one question — where it lives, what it reads, how breakers outrank it, and every claim and start that asks it                                                    |
| [levers.md](./levers.md)               | Computing one lever's answer — pacing, the utilization line, the dollar budget, and the schedule                                                                              |
| [exhaustion.md](./exhaustion.md)       | Building the subscription floor — the usage-limit mark, how exhaustion derives and clears, the grace, the resample, the clear by hand, a lease parked mid-step, and the crash |
| [dollars.md](./dollars.md)             | Counting a dollar line — the one figure every throttle and breaker reads, and what PARTIAL means once it does                                                                 |
| [surfaces.md](./surfaces.md)           | Showing the gate — the configured settings on the runner's own status and panel, and the declining reason on the wire, the board, and both CLIs                               |
| [release.md](./release.md)             | Holding or releasing a chunk the gate parks — the fleet's one mid-graph release, its two throttling callers, and how `epic:resilience` and later callers reuse it             |
| [carry-over.md](./carry-over.md)       | Landing the epic — the order of its pieces, the fleet across them, and what an operator does once                                                                             |
