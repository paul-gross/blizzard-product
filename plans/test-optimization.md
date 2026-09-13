# Plan — `epic:test-optimization`

Every chunk the fleet lands waits on the gate. Today a PR takes about 24 min and a push to `master` about 28. Since July
the unit and component job has gone from about 1 min to 10 min, while the test count grew 3.5× and the time grew 8.6×.
The goal is to cut that time without weakening any assertion, and to close the coverage gaps in the PR gate.

## Baseline

Medians over 60 successful runs:

| Job                            | Median   | Shape                                                             |
| ------------------------------ | -------- | ----------------------------------------------------------------- |
| service tier                   | 13.7 min | 104 tests run one at a time, ~8 s each, each with its own stack   |
| pytest unit + component        | 10.3 min | 5,443 tests, `-n auto` on 4 vCPU (2 min 32 s on a 20-core laptop) |
| crash sweep, CI profile        | 8.9 min  | 29 tests run one at a time, ~18 s each                            |
| dev hub image                  | 3.7 min  | multi-arch QEMU build on every push to `master`                   |
| eslint + vitest + client drift | 1.7 min  | four `ng test` projects run one after another                     |

The PR critical path is the unit job followed by the service tier, because `upper-tiers` has `needs: [gate]`. Queue time
is negligible, about 2 s.

## Work areas

Area 1's items are independent of each other. Areas 3 onward each contain a decision.

### 1. Pipeline quick wins

Workflow-only changes.

- **Drop `needs: [gate]` from `upper-tiers`** in `pr.yml` and `push.yml`. `dev-image` still waits on every check. The
  cost is upper-tier runner minutes spent on PRs that fail lint.
- **Cache all installs.** Share the uv cache for `blizzard-mock`'s `uv sync`, keyed on both lockfiles. Use `cache: npm`
  in `dev-build` and `release`. Persist the Angular build cache.
- **Build arm64 only for releases.** `dev-image` becomes `linux/amd64` only. This requires the hosted hub, which follows
  `edge`, to be on amd64 first.
- **Split the release job.** Break `release.yml` `full-suite-tiers` into parallel service, full crash sweep, and e2e
  jobs, with `release` needing all three.
- **Mark the 9 unmarked files** (`test_runner_*_cli.py`, `test_runner_*_api.py`, `test_worker_settings.py`). They run by
  default but `-m unit` and `-m component` both skip them. Add a guard that fails on any default-suite test that carries
  neither marker.

### 2. PR gate coverage

- **Angular AOT build on every PR.** Run `npm run build` in the frontend job, which today first runs on push or tag.
  Hand the output to the e2e job.
- **E2E on every PR.** Add a job in `upper-tiers.yml` with the three-repo checkout and Chromium, running against the
  frontend job's build. Measure its runtime before enabling it. Area 4 keeps it affordable.
- **Real-winter tests in CI** against a pinned winter (area 2a).
- **Journey in CI.** It is currently red: a local run on `master` times out at its first wait, with the first chunk
  still `ready` after 300 s, 5 min into the run. First determine whether that is rot or the local environment, then make
  it green. Its CI needs match the crash sweep's: three-repo checkout, mise, both `uv sync`s, and a git identity, with
  no token or network. Add it as an `upper-tiers.yml` job using the pinned winter. Measure its wall time, then decide
  between every PR and push/tag only. It is a single test that can't be parallelized, so it sets a floor on the pipeline
  it joins.

### 2a. Pinned winter source

The fixture scaffold `git clone --local`s a *winter source*: any repo with `tools/winter-cli/`. It runs that CLI via
`mise exec -- uv run`, so nothing is installed. Today the source is resolved inconsistently:

- **Upper tiers** check out `paul-gross/blizzard-workspace` at its current `master`, unpinned, through
  `BLIZZARD_MOCK_WINTER_SOURCE`. A winter change can break blizzard CI, and a red run can't be reproduced.
- **`test_runner_winter_provider.py`** walks up the directory tree to find an enclosing workspace. In the gate job,
  which has a single-repo checkout, it finds none and silently skips.

Source preference:

1. **`paul-gross/winter`** is upstream. Its CLI has every command the runner calls (`capabilities --json`,
   `ws worktrees --json`, `ws pull --standalone`, `ws init`, `ws checkout`, `ws fetch`, `ws disconnect`, `provision`,
   `service down`). Still unproven: that the fixture mints a working workspace from it and the upper tiers pass. The
   first issue settles this with a real run.
2. **`paul-gross/blizzard-workspace`** is the fallback. It works today, but it is blizzard's fork. If `winter` falls
   short, record the gap and fix it upstream.

Not usable: `winter-workspace` (unstable) and `winter-blizzard` (README only). Both choices above are public, so no
token is needed.

- **Pin to a SHA** (`winter` has no tags) in one committed place that every job reads. Bumps are deliberate PRs;
  consider a Renovate regex manager.
- **One resolution order.** `test_runner_winter_provider.py` checks `BLIZZARD_MOCK_WINTER_SOURCE` before walking the
  tree. Its real-winter tests move to the upper-tier jobs.
- **Detached HEAD.** A SHA checkout has no named branch. Verify that the fixture clone and the minted workspace still
  work.
- **Local runs stay unpinned.** Add a `mise` task that fetches the pinned source, so CI failures can be reproduced.

### 3. Prototype database

`build_hub()` (`tests/support.py`) runs all 89 hub migrations on a fresh SQLite file. That costs ~0.85 s per call across
1,000+ call sites, before parametrization. The runner's `init_environment()` does the same for 39 revisions. This is
likely more than half of the unit job's CPU.

- Build a session-scoped prototype per xdist worker, one for the hub and one for the runner.
- `build_hub()` and the runner equivalent copy the prototype into `tmp_path`.
- Migration tests (`test_store_migrations.py` and its kin) still migrate from empty.
- Add a guard that the prototype schema equals a freshly migrated schema.

Target: unit job ≤ 50% of current.

### 4. Upper tiers under xdist

Service, crash sweep, and e2e run serially. Ports already come from `_free_port()`, directories from `tmp_path`, and
crash-sweep shared state lives in a session fixture, so each xdist worker can hold its own copy.

- Run all three under `pytest-xdist`, with per-worker session fixtures (one crash-sweep fixture and forge per worker).
- **Port race:** `_free_port()` releases the port before the daemon binds it. Either bind port `0` and report the
  result, or retry on bind failure.
- **`os.environ`:** `_drive_until_done` mutates the process environment. Pass the environment to the spawned harness
  instead.
- Choose worker counts by measurement, since 2 may beat 4 on a 4-vCPU runner. Shard across a job matrix if one runner
  saturates.

Target: each upper-tier job ≤ 33% of current.

### 5. Per-test service cost (spike)

Each test runs, in order:

- `blizzard-mock-fixture reset` (~2.3 s on a laptop);
- mock forge start;
- `blizzard-hub init` (all migrations);
- `blizzard-hub host`;
- optional mock runner and IdP;
- teardown with up to 10 s SIGTERM per process.

A shared hub isn't possible today because the hub is single-tenant. Tests count hub-wide SSE events, read the whole
queue and feed, and register graphs by name; crash tests kill the hub. `epic:test-shared-service`, via
`epic:multi-tenancy`, is the path to sharing. Until then:

- Measure each phase first: mint, forge, init, ready, act, teardown.
- Reuse a minted fixture per worker, by reset or copy. Absolute `file://` origins complicate copying.
- Start the hub from a pre-migrated database (area 3 applied to a subprocess). This conflicts with
  `epic:test-architecture`'s ban on touching store files, so state the trade-off explicitly.
- Shorten teardown: expect a clean exit instead of spending the SIGTERM allowance.

### 6. Event-driven waits

`sse_tap` has two fixed waits:

- **Drain, 1.5–2 s:** discards the connect-time replay, because the end of the replay can't be detected.
- **`collect(window=6.0)`:** the assertions are negative ("exactly one `queue-changed`", "zero `chunk-changed`"), and
  the fixed window is the only current way to show nothing more arrived.

Replacements:

- **Replay:** events carry integer ids. Read the current cursor and subscribe with `Last-Event-ID`, or drain until that
  id.
- **Positive assertions:** return as soon as the expected events have arrived.
- **Negative assertions:** after the act, trigger a known harmless barrier event and wait for it. The stream is ordered,
  so anything the act emitted arrives before it. **This is valid only where fan-out is emitted before the request
  returns.** Verify each event path. Any path that fails keeps a bounded window and a comment explaining why.

SSE windows cover ~7 tests, about 1 min. The same treatment applies to e2e's `sleep(0.5)` tick spacing, `poll_until`
intervals, the 10 s termination allowance, and the unit-tier sleeps in `test_cli.py` and
`test_runner_harness_opencode_diagnostic.py`.

### 7. Slow unit outliers

- **`test_runner_harness_opencode_diagnostic.py`** takes ~164 s of CPU: one 35 s test and ~30 tests at 4–5 s, all
  spawning subprocesses. Decide per test whether the subprocess is what's under test. Share one fake binary per module
  where possible.
- **`test_runner_winter_provider.py`'s real-winter tests** take 11–15 s each. Placement follows area 2a.

## Out of scope

Making tests implementation-independent belongs to `epic:test-architecture`, which owns any conflict such as area 5's
pre-migrated hub.
