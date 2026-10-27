---
epic: test-suite/runner-tier
refinement: pristine
status: horizon
---

# Plan — `epic:test-suite/runner-tier`

The runner's service tests are the ones most bound to the Python. They build a `RunnerConfig` in memory, serve the
runner's local API from `build_hosted_app` on a thread inside the test, step its loop by calling `LoopWiring.tick_once`,
and read leases, escalations, and usage back through `SqlAlchemyRunnerStore`. None of that survives a move to another
language, and none of it should: a runner that can only be tested from inside itself has never had its outside proven.
This epic gives the runner the outside its tests need, then moves them.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## The tests it moves

| Group                           | Size     | Coupling today                                                                                                                                                                                                                                                                                                                                                             |
| ------------------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Runner run in-process           | about 60 | `tick_once`, `build_hosted_app` on a thread, configuration through `dataclasses.replace`, store reads through the ORM.                                                                                                                                                                                                                                                     |
| Runner host started as a daemon | about 30 | The daemon is real, but its host layout and token files are written through internal classes, `platform_tracing` seeds a live runner's store, and graduation and ingest seed and read the runner's SQLite file because migrating it is their subject. `runner_init` and `idp_federation` arrive from the hub side for the same reason, and land in `tests/service/fleet/`. |

## First, the runner side of the spike

The foundation's spike never reached the runner. Before the rest is planned, one runner-side test and one lever-driven
failure test run from Playwright against a real runner host, which sizes what a host costs to start and shows where the
surfaces below fall short.

### What it measured

Taken on 27 October 2026 against blizzard at `c5f56834a` and blizzard-mock at `321b7dace`, on an Intel Core Ultra 7 258V
workstation pinned to four CPUs with `taskset -c 0-3`. This matches the foundation's four-core measurement method, not a
run on a hosted GitHub machine. Both daemons kept the suite's environment contract and statement guards. Each case owned
a freshly minted basic-provider workspace, a mock hub, and a real `blizzard-runner host`, initialized through its CLI
and driven by its timer at `BZ_RUNNER_TICK_SECONDS=0.5`. No test imported blizzard or accessed its store.

Each case passed three times at one worker and three times at two workers: twelve passes, no retries. The two-worker run
used the pinned Node `22.23.3` and completed its six tests in 42.0 seconds. These are its per-case medians:

| Measurement                                                   | Lease round trip | Unreachable hub and recovery |
| ------------------------------------------------------------- | ---------------- | ---------------------------- |
| Fixture mint                                                  | 0.30 s           | 0.29 s                       |
| Mock hub to healthy                                           | 0.97 s           | 0.97 s                       |
| Runner `init`                                                 | 3.84 s           | 3.87 s                       |
| Runner host launch to healthy                                 | 2.78 s           | 2.78 s                       |
| From starting `init` to healthy, including test configuration | 6.64 s           | 6.66 s                       |
| Whole setup                                                   | 7.90 s           | 8.02 s                       |
| Test body                                                     | 3.63 s           | 4.14 s                       |
| Tick scheduling idle within the body                          | 2.00 s           | 2.50 s                       |
| Setup, body, and shutdown                                     | 12.52 s          | 13.14 s                      |

Scheduling idle is the overlap of each logged `tick end` → next `tick start` interval with the scenario's lifetime. Some
of it overlaps a worker doing useful work, so it bounds what stepping could save rather than promising that saving. The
one-worker medians were similar: 6.57–6.70 seconds from `init` to healthy and 12.61–13.26 seconds for the whole test.

### What it settles before slicing

- **The outside is sufficient for both cases.** The round trip observes the claim at `POST /api/fleet/routes`, the event
  submission in the mock hub's capture, the completed chunk, its lease through the runner's local API, and the worker's
  actual commit in the bare origin. The recovery case identifies a pending `completion.submitted` fact for its own
  chunk, stages the outage before that completion is delivered, observes an event submission attempt with that same fact
  still unacknowledged, then restores the hub and observes the chunk complete, that fact acknowledged, and the buffer
  empty. Neither case needs a new API read or a change to blizzard-mock.
- **Buffer depth alone is not proof of a buffered completion.** The first version could pass with an earlier fact in the
  queue and finish the chunk only after recovery. The harness must expose the existing facts feed's sequence, kind,
  chunk, and acknowledgement timestamp, and the real port must track the particular completion throughout the outage.
  The stepped host removes the remaining race between observing that fact and the next timed flush.
- **The host needs a short scratch path.** The ordinary Playwright output path made `host.sock` 153–169 bytes long and
  the host correctly refused it; Linux permits 107. The host fixture must allocate a short, unique runtime directory and
  validate the final socket path, rather than putting the host under a descriptive test-output path. Logs and metrics
  can still be attached to that test's output.
- **Initialization needs no internal file writer.** The CLI writes and registers the identities, tokens, store, and
  harness files. TypeScript preserves those identities and tokens while editing the documented host configuration,
  supplies a fake Claude credential file, disables the unused OpenCode binding, points the external-usage sampler at
  absent fixture credentials, and writes the mock harness fence into the disposable workspace. Nothing reads a
  developer's provider credentials or contacts a real hub or provider.
- **Keep a host and a mint per test.** A mint costs about 0.3 seconds; reusing it would not remove the roughly
  6.6-second host initialization. At this sample's cost, ninety tests on two workers project to roughly ten minutes,
  below the plan's seventeen-minute gate baseline, even without distributing them over service shards. This is a sizing
  estimate, not a measurement of the full ported roster. It does not justify fixture-world reuse in the initial harness;
  each port still measures the tier against the speed guard.

## Surfaces it builds

- **A host that steps on request.** Today a runner's loop can be driven two ways, and neither serves a test from
  outside. `blizzard-runner host` runs the loop on its own timer while serving the local API that a spawned worker calls
  back into. `blizzard-runner tick` runs one pass and exits, but serves no API, and refuses to run while a host holds
  the directory. The runner gains a host configuration in which the loop does not run on its own, and a call on its
  local API that runs one REAP → PULL → FILL → ADVANCE pass for a named runner and returns when the pass completes.
  Workers it spawns keep running between steps, as they do between timed passes, and a second step requested while one
  is running is refused at once. An operator gets the same thing a test does: a misbehaving runner can be frozen and
  walked one pass at a time. `blizzard-runner tick` becomes the same call when a host is running, instead of a refusal.
- **Configuration as files.** `contracts/config/host.json` already documents the host file's keys and its durable paths,
  and `blizzard-runner init` and `add` already write the host file and its token. The epic extends that contract to what
  it does not yet cover, such as the tree under `runners/` and the token files, so a test can lay out a host without the
  classes that write one today.
- **Reading state.** The local API already serves leases, escalations, facts, and status. For each value a test still
  reads from the store, such as pending outbound facts, usage samples, and a local pause, the epic decides between an
  API or CLI read and letting tests query the schema `contracts/store/runner.json` already pins with plain SQL. The ORM
  is never the contract.

## Sharing and stepping

A test that steps a host owns that host, so two such tests never share one; each test gets a host of its own, as each
test has its own runner today, and the parallelism lives across hosts. A test in `tests/service/runner/` keeps a mock
hub of its own, since the mock is cheap to start and holds no migrations, and its levers and tenants' work are not
isolated from one another. A test in `tests/service/fleet/`, where the hub is real too, runs on a shared hub at
`fresh-tenant`, since its runner claims tenant-wide work, unless its subject is hub-wide: `runner_init`'s reset of the
hub's data and `idp_federation`'s provider changes and key rotation each take `fresh-hub`.

This half of the tier is not expected to get faster. Sharing buys a test against the mock hub nothing, and a host
started as a process costs more than a loop called in place. The initiative's guard holds it: the tier never becomes the
gate's slowest job, and if the runner's ports push it there, reusing a minted fixture world per worker comes before
anything else.

## What stays in Python

A test whose subject is the loop's internal machinery, such as one that injects frames into the runner's event broker to
prove how a restarted broker numbers them, moves down to the component tier rather than being ported. The graduation,
ingest, and directory-ownership tests are about the store's files and keep dedicated runners; they may prepare those
files before the daemon starts and read them after it stops, and may stay in the service tier's pytest half if porting
them buys nothing.

The runner's own event stream tests wait on fixed windows, as the hub's did, and are rewritten the way the hub tier
rewrites them.

## Done

Every runner-side test that exercises the runner from outside runs in `tests/service/runner/` or `tests/service/fleet/`
against a stepped or timed host. Tests about the loop's internals run in the component tier. The Python files left in
`tests/service/` are only the pytest half's, and the helpers e2e and the crash sweep still borrow from it live in a
support module of their own.
