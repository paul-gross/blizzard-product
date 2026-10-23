# Carry-over

The epic adds a stepped loop, two step routes, `blizzard-runner tick` against a running host, and the host contract's
new coverage, all additive. A timed host, the default, behaves as it does today. Nothing an installation holds moves.

## The order of landings

1. **The spike** ([spike.md](./spike.md)), on a branch, its findings recorded in the plan.
2. **The host contract's coverage** ([layout.md](./layout.md#what-the-contract-covers)), independent of everything
   after.
3. **The stepped loop and the step routes** ([stepping.md](./stepping.md)), with their component tests and the OpenAPI
   document.
4. **`blizzard-runner tick` against a running host**, after 3.
5. **The local API's gains**, each in the change that first needs it, or ahead if the spike named it
   ([reads.md](./reads.md#what-the-api-gains)).
6. **The harness's host** ([layout.md](./layout.md#how-the-harness-builds-a-host)), the `service-runner` and
   `service-fleet` projects, and the `runner/store/` lint exception, after 2 and 3, proven by the spike's two tests
   ported for real.
7. **The tests that move down**, independent of everything ([roster.md](./roster.md#what-moves-down)).
8. **The ports**, one change per file, after 6: `service/runner/` first, then `service/fleet/`, since the fleet files
   wait on the hub tier's `auth` profile and fresh-hub lifecycle as well.
9. **The support module** ([below](#the-support-module)), last.

## The support module

When the last port has landed, `tests/service/` holds only the pytest half's files and `support.py`. What other tiers
borrow from `support.py`, `stub_idp`, `require_stub_idp`, `mock_runner`, and `poll_until`, used by `tests/e2e/`, moves
to `tests/mock_fleet_support.py`, a support module of its own outside `tests/service/`, and its importers are re-pointed
in the same change. `sse_tap` and `SseTap` are deleted once nothing uses them. The modules e2e and the crash sweep
already borrow from outside `tests/service/`, `tests/fleet_launchers.py`, `tests/runner_join.py`,
`tests/runner_fakes.py`, `tests/host_files.py`, and `tests/crash/support.py`, are already support modules and stay as
they are.

## Citations and the rulebook

Each port moves its citations under `bzh:port-moves-citations`, as the hub tier's do. `blizzard-context`'s
`architecture/crash-correctness.md`, where `bzh:steppable-loop` lives, names the stepped host and its routes as the
loop's handle from outside, in the change that lands 3. The foundation's companion gains, under its rule that practices
join as ports teach them, how a runner test builds its host and steps it.

## The speed guard

The runner half is not expected to get faster, and the initiative's guard holds it: the tier never becomes the gate's
slowest job. Each port records the tier's CI time. If a port would cross the line, the next change, before any further
port, reuses one minted fixture world per worker, reset between tests with `blizzard-mock-fixture reset`, in place of a
mint per test.
