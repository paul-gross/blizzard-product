# Journey specifications

The technical contracts behind `epic:test-suite/journey`. Product intent begins at the [epic plan](../index.md); the
package and the harness are `epic:test-suite/foundation`'s ([its spec](../../foundation/spec/index.md)), a fresh hub's
lifecycle is `epic:test-suite/hub-tier`'s ([its spec](../../hub-tier/spec/index.md)), the runner host is
`epic:test-suite/runner-tier`'s ([its spec](../../runner-tier/spec/index.md)), and the fleet folder and its profiles are
`epic:test-suite/fleet-tier`'s ([its spec](../../fleet-tier/spec/index.md)). Nothing here redefines them. Implementation
enters at the concern below.

It builds on three things as they are today. `blizzard.tools.invariants` holds the invariant checker
(`bzh:invariant-checker`): `RunnerInvariants`, fourteen checks over a host's store and a fifteenth after recovery, and
`HubInvariants`, thirty-four query checks and the per-tenant fact checks over the hub's, with `Invariants` running
either or both on engines of their own. No check reads both stores. `blizzard dev check-invariants`, in a hidden CLI
group, loads each daemon's configuration to find its database and runs the checker against the files.
`tests/journey/test_journey_rehearsal.py` runs `mise run journey` under the upper tiers' `journey` job, behind
`full_suite`, in about a minute and a half. In `blizzard-mock`, `tests/test_wire_parity.py` and
`tests/test_real_runner_against_mock_hub.py` carry the `needs_blizzard` marker, which its CI deselects.

Paths are relative to the `blizzard` repo root unless a leaf says otherwise. The epic reaches `blizzard-mock`, whose CI
gains the guard, and `blizzard-context`, whose statement of the checker and whose command pages move with it.

| Where                            | Read when                                                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| [check.md](./check.md)           | Building the consistency check — the checker's split, the two routes, the read, and the CLI's two modes        |
| [port.md](./port.md)             | Porting the rehearsal — its topology, each morning-after assertion and its surface, and what it no longer does |
| [parity.md](./parity.md)         | Putting the mock fleet's wire parity in CI — the repair, the ref it checks against, and the jobs               |
| [carry-over.md](./carry-over.md) | Landing the epic — the order of changes and the citations                                                      |
