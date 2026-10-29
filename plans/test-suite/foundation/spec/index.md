# Foundation specifications

The technical contracts behind `epic:test-suite/foundation`. Product intent begins at the [epic plan](../index.md), and
the rules every ported test keeps are the [initiative's](../../initiative.md); implementation enters at the concern
below. The hub and runner tiers fill the package this spec builds, and `epic:test-suite/fleet-tier` its UI folder. None
of them is specified here.

It builds on the service tier as it runs today. `mise run service-test` runs
`BLIZZARD_SERVICE=1 uv run pytest -n auto tests/service/`, about fifty `test_*_service.py` files with no conftest of
their own. Each test starts its own daemons through `tests/fleet_launchers.py` (`run_forge`, `run_hub`) and
`tests/service/support.py` (`mock_hub`, `mock_runner`, `stub_idp`), on ports from `tests/support.py`'s `free_port`, and
most mint a fixture world first through `tests/fleet_world.py`'s `mint_world`. The environment those daemons inherit is
assembled piecemeal in `tests/suite_plugin.py`, partly in `pytest_configure` and partly in autouse fixtures. CI runs the
tier as `upper-tiers.yml`'s three-shard `service-tier` job, reported under one check by `service-tier-result`, and as
`release.yml`'s single `service-tier` job. No `package.json` exists outside `web/`, and nothing in the repo uses
`@playwright/test`.

Paths are relative to the `blizzard` repo root unless a leaf says otherwise.

The epic reaches beyond `blizzard`. In `blizzard-mock`, the mock forge learns to create a repository at runtime
([surfaces.md](./surfaces.md#repositories-on-demand)). In `blizzard-context`, a companion redefines the tier and writes
down its architecture ([rulebook.md](./rulebook.md)). The workspace is unchanged.

| Where                            | Read when                                                                                                                                                                       |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [package.md](./package.md)       | Laying out the Playwright package — its files, its projects, its toolchain, the generated wire types, and what the Python tooling is told to skip                               |
| [daemons.md](./daemons.md)       | Starting and stopping the daemons a run spawns, or shaping the environment contract both suites read and how refusals in their logs fail a test or the run                      |
| [hubs.md](./hubs.md)             | Writing a profile, sharing a hub among workers, the setup and teardown projects, the common tenant, a fresh hub, or the mock fleet beside each hub                              |
| [isolation.md](./isolation.md)   | Declaring an isolation level, the fixtures a test names, the clients and the ledger, narrowing reads and the event stream, or the cross-check                                   |
| [surfaces.md](./surfaces.md)     | Building what the epic asks of `blizzard-mock` and of the product — repositories on demand, the CLI's tenant variable, request errors in the hub's log, cheaper tenant creation |
| [ci.md](./ci.md)                 | Wiring the tier into CI — the `mise` tasks, the shards, the static job, the toolchain pin, failure evidence, the winter-free guard, the cross-check runs                        |
| [rulebook.md](./rulebook.md)     | Writing the `blizzard-context` companion — the redefined tier, the scoped rules, the tier's invariants, the registry check, the exemplar                                        |
| [carry-over.md](./carry-over.md) | Landing the epic — the order of changes, the three ports, the measurements and what they decide                                                                                 |
