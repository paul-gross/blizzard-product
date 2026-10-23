# Runner-tier specifications

The technical contracts behind `epic:test-suite/runner-tier`. Product intent begins at the [epic plan](../index.md); the
package, the harness, and the clients are `epic:test-suite/foundation`'s ([its spec](../../foundation/spec/index.md)),
people, the CLI, the barrier, and a fresh hub's lifecycle are `epic:test-suite/hub-tier`'s
([its spec](../../hub-tier/spec/index.md)), and nothing here redefines them. Implementation enters at the concern below.

It builds on the runner as it runs today. `blizzard-runner host` (`src/blizzard/runner/cli/runtime.py`) holds
`data/host.lock` for its whole life and gives each runner a `PeriodicDriver` (`src/blizzard/runner/loop_wiring.py`) that
calls `tick` (`src/blizzard/runner/loop/tick.py`) every `BZ_RUNNER_TICK_SECONDS`, thirty by default, beside a host-wide
`HostTick` on its own timer, while serving the local API from `build_hosted_app` (`src/blizzard/runner/app.py`) over TCP
and `host.sock`. `blizzard-runner tick` runs one pass under the same lock and serves no API, so it is refused while a
host runs. The runner's service tests mostly bypass both: they build a `RunnerConfig` in memory with
`dataclasses.replace` (`tests/fleet_launchers.py`'s `runner_config`), step it with `LoopWiring.tick_once`, serve
`build_hosted_app` on a thread (`runner_api`), and read the store through `SqlAlchemyRunnerStore`, `daemon_store`, or
`reflect`. About thirty start a host as a process but lay out its directory through `tests/host_files.py` and
`tests/runner_join.py`.

Paths are relative to the `blizzard` repo root unless a leaf says otherwise. The epic reaches `blizzard-context`, whose
citations move with each test, and which gains the stepping host where the runner's architecture is stated
([carry-over.md](./carry-over.md#citations-and-the-rulebook)); `blizzard-mock` is unchanged.

| Where                            | Read when                                                                                                                                      |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| [spike.md](./spike.md)           | Running the runner side of the spike first — what it ports, what it measures, and what it decides                                              |
| [stepping.md](./stepping.md)     | Building the host that steps on request — its configuration, the step routes, concurrency, and `blizzard-runner tick` against a running host   |
| [layout.md](./layout.md)         | Laying out a host from files — the contract's new coverage, how the harness builds a host, and the worker fence                                |
| [reads.md](./reads.md)           | Replacing a store read — each value the tests read today, and the surface or schema read that replaces it                                      |
| [roster.md](./roster.md)         | Placing a file — what moves to `service/runner/` or `service/fleet/`, what moves down to the component tier, and what stays in the pytest half |
| [carry-over.md](./carry-over.md) | Landing the epic — the order of changes, the support module, the citations, and the speed guard                                                |
