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
5. **A configurable subscription usage base**, carrying the samplers' existing override through the host file, and a
   fixture-owned local vendor counterpart ([reads.md](./reads.md#producing-samples-and-spend)). Neither a test nor the
   counterpart writes the host's store.
6. **The local API's gains**, each in the change that first needs it, or ahead if the spike named it
   ([reads.md](./reads.md#what-the-api-gains)).
7. **The harness**, in three landings: test-owned mock hubs and runner clients; CLI-initialized runner hosts and
   validated profiles, including the vendor counterpart, after 2, 3, and 5; then the `service-runner` and
   `service-fleet` projects, CI selection, and `runner/store/` lint exception
   ([layout.md](./layout.md#how-the-harness-builds-a-host)). The integration is proven by the spike's two scenarios on
   the production fixtures, never by merging the throwaway code.
8. **Shared Python helpers**, before deleting any source file: extract the helpers other files import from
   `test_runner_service.py`, `test_host_halted_runner_service.py`, `test_runner_garden_findings_service.py`,
   `test_subscription_floor_service.py`, and `test_utilization_lever_service.py` into a non-collected support module.
   Update every importer without changing the assertions; no port is held behind another file's incidental helper
   imports.
9. **The tests that move down** ([roster.md](./roster.md#what-moves-down)), after their shared helper extraction, and
   **the retained graduation/ingest proofs' schema reads**, after the contract coverage. Neither waits on the external
   harness.
10. **The ports**, one change per file, after 7 and 8: `service/runner/` first, then `service/fleet/`, since the fleet
    files wait on the hub tier's `auth` profile and fresh-hub lifecycle as well.
11. **The support module** ([below](#the-support-module)), after every port and retained-tier move.

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
slowest job. No port records the tier's CI time. If the tier ever crosses the line, the change that brings it back
reuses one minted fixture world per worker, reset between tests with `blizzard-mock-fixture reset`, in place of a mint
per test.
