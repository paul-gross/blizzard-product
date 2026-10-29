# Fleet-tier specifications

The technical contracts behind `epic:test-suite/fleet-tier`. Product intent begins at the [epic plan](../index.md); the
package, the harness, and the clients are `epic:test-suite/foundation`'s ([its spec](../../foundation/spec/index.md)),
people, the CLI, the barrier, and a fresh hub's lifecycle are `epic:test-suite/hub-tier`'s
([its spec](../../hub-tier/spec/index.md)), and the stepping host, the host contract, and the runner's reads are
`epic:test-suite/runner-tier`'s ([its spec](../../runner-tier/spec/index.md)). Nothing here redefines them.
Implementation enters at the concern below.

It builds on the e2e tier as it runs today. `mise run e2e` runs `BLIZZARD_E2E=1 uv run pytest tests/e2e/`, serially,
after `web-build` fills `src/blizzard/static/`, with a pinned `otelcol-contrib` on `PATH` for the fleet-trace subtests.
Its 38 files hold 84 cases. Most start a forge and a hub through `tests/fleet_launchers.py`, tick the runner in-process
with `drive_until_done` or a file's own `_tick_until` or `_drive_until`, and serve the runner's API with `runner_api`;
three start a real `blizzard-runner host`. Thirteen drive a browser through Python Playwright. The upper tiers' `e2e`
job in `.github/workflows/upper-tiers.yml` runs it under `full_suite`, as the required check
`upper-tiers / e2e acceptance loop (blizzard:e2e)`.

Paths are relative to the `blizzard` repo root unless a leaf says otherwise. The epic leaves `blizzard-mock` unchanged,
and reaches `blizzard-context`, whose citations move with each test and whose e2e method retires
([retirement.md](./retirement.md)).

| Where                            | Read when                                                                                                          |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| [roster.md](./roster.md)         | Placing a file — where each moves, at which level, what moves down, and what is dropped                            |
| [surfaces.md](./surfaces.md)     | Replacing a store read or an oracle import — each read the hub, the runner, and the mock fleet gain                |
| [harness.md](./harness.md)       | Building the fleet and UI projects — hub profiles, trace companions, the sink and the collector, the runner host   |
| [retirement.md](./retirement.md) | Retiring the e2e tier — the support modules first, then every task, job, marker, check, and citation that names it |
| [carry-over.md](./carry-over.md) | Landing the epic — the order of changes, the citations, and the speed guard                                        |
