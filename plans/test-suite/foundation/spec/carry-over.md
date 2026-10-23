# Carry-over

The epic changes how the service tier runs, never what blizzard does, save the CLI's new variable and, if the
measurement asks for it, when a tenant's packaged graphs are minted. No store, configuration, or token an installation
holds moves. What carries over is the tier: its pytest half keeps running unchanged, and its first three tests cross
into the new package.

## The order of landings

Each change leaves `master` green and the required check intact. The plan's six children map onto these landings:

1. **Repositories on demand**, in `blizzard-mock` ([surfaces.md](./surfaces.md#repositories-on-demand)). Independent of
   everything; 5 waits on it.
2. **The CLI's tenant variable**, in `blizzard` ([surfaces.md](./surfaces.md#the-clis-tenant-variable)). Independent of
   everything.
3. **Unhandled request errors in the hub's log**, in `blizzard`
   ([surfaces.md](./surfaces.md#unhandled-request-errors-in-the-hubs-log)). Independent of everything; 5 waits on it.
4. **The daemons' environment contract**: `tests/daemon-env.json`, `tests/suite_plugin.py` reading it, and
   `tests/test_daemon_env_contract.py` ([daemons.md](./daemons.md#the-environment-contract)). The pytest suite runs
   exactly what it ran before, which every tier's CI run proves. Independent of 1 and 2; if 2 lands first, the file
   unsets `BZ_HUB_TENANT` from the start, and otherwise 2 adds it.
5. **The package and the harness**, waiting on 1, 3, and 4: everything in [package.md](./package.md),
   [daemons.md](./daemons.md), [hubs.md](./hubs.md), and [isolation.md](./isolation.md) but the cross-check runs; the
   `playwright` and `playwright-check` tasks, the `service-playwright` jobs under `service-tier-result` in both
   workflows, the static job, and the winter-free guard's three changes ([ci.md](./ci.md)). The rulebook companion lands
   with it, in `blizzard-context`, all of [rulebook.md](./rulebook.md) but the exemplar row. The package holds no spec
   of its own yet, so its service projects select nothing and pass, and the change proves the harness by one spec under
   `tests/harness/checks/`, a project of its own and not a `service-` one, that brings up a shared hub and a fresh one,
   resolves `tenant` and `project` at each level, and has a request answered `5xx` fail its test. That spec stays as the
   harness's own proof.
6. **The ports and the measurements**, waiting on 5 ([below](#the-ports), [below](#the-measurements)), with the
   cross-check's flag and workflow ([isolation.md](./isolation.md#the-cross-check),
   [ci.md](./ci.md#the-cross-check-runs)) and the exemplar row.

The cheaper tenant creation, if the measurement asks for it, lands as its own change after 6's measurement and before
the ratio it enables is recorded.

## The ports

Each Python original becomes one spec in `tests/service/hub/`, its tests keeping their function names as titles, and is
deleted in the same change:

| Original                                              | Spec                            | Tests |
| ----------------------------------------------------- | ------------------------------- | ----- |
| `tests/service/test_analytics_service.py`             | `analytics.spec.ts`             | 3     |
| `tests/service/test_analytics_events_service.py`      | `analytics-events.spec.ts`      | 2     |
| `tests/service/test_analytics_operational_service.py` | `analytics-operational.spec.ts` | 1     |

Each original builds its own stack, a forge and a hub per test, files issues in `blizzard/toy-api`, ingests them with
`POST /api/chunks`, and speaks the fleet plane with a runner token from `POST /api/runners`. A port does the same
through the fixtures: `repository()` for the forge and its configuration, `issue()` to file the issue and ingest it, and
`fleet(name)` for the runner's calls. Minting a runner token is a tenant-wide write, and `fleet` refuses below
`fresh-tenant`, so all three files land there. None can prove the cheaper levels, and the harness's own spec from step 5
is their proof until the hub tier ports a test that lands at one.

The originals' `_stack` and `_ingest` helpers are local copies and leave with them. The one helper they import that
another file uses, `transcript_segment_record` in `tests/service/support.py`, stays where it is for its other callers,
and the port restates the record it builds as a typed literal from the wire types. No blizzard-context file cites the
three originals today, so no citation moves; their entries in `scripts/prose-density-baseline.json` are dropped.

## The measurements

[The plan](../index.md#measurements-owed) owes two numbers, taken on the CI runner's shape, four vCPUs. Both are taken
with the three ports and the harness's own spec run under `--repeat-each`, enough repeats to fill a run of the tier's
present length, so the tenants a hub accumulates over a whole run are counted, as the plan asks.

- **Hubs per worker.** `WORKERS_PER_HUB` starts at `8`, one shared hub for every worker a shard runs. At the CI worker
  count, two to start, the suite runs once on that one hub and once with a hub per worker for comparison, with the
  statement guards on as always and once more with them off, and records each configuration's wall time and its hubs'
  CPU. The one hub stands unless its wall time is more than ten percent behind a hub per worker; then `WORKERS_PER_HUB`
  drops to the largest ratio that is not. A later change to the worker count takes the measurement again.
- **Store contention.** The clients record every call's latency in the run's report. At the chosen ratio, the slowest
  write's latency and the count of calls slower than one second are recorded. If the slowest comes within half of the
  store's five-second busy timeout, the shared hubs move to Postgres: the default profile names a Postgres store, the
  service jobs gain a `postgres` service container, and the companion amends the SQLite rule.

If the one hub falls behind because tenant creation saturates its core, cheaper tenant creation
([surfaces.md](./surfaces.md#cheaper-tenant-creation)) is built before the ratio is lowered, and the hub measured again
on it. The ratio, the store decision, the tier's CI time before the epic and after it, and whether cheaper tenant
creation was built are recorded in the plan's *Measurements owed* section, in the change that records the measurement.
