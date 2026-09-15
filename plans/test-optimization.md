---
epic: test-optimization
refinement: refined
slices:
  - name: full
    status: horizon
---

# Plan — `epic:test-optimization`

Every chunk the fleet lands waits on the gate. Since July the unit and component job has grown from about 1 min to 7,
while the test count grew 3.5×. A PR now waits about 8 min and a push to `master` about 12. The goal is to cut that time
without weakening any assertion, and to close the coverage gaps in the PR gate.

## Baseline

Recent successful runs on `master`:

| Job                            | Duration    | Shape                                                          |
| ------------------------------ | ----------- | -------------------------------------------------------------- |
| pytest unit + component        | 6.9–7.6 min | ~5,750 tests, `-n auto` on 4 vCPU (~2 min on a 20-core laptop) |
| service tier                   | 5.2–5.5 min | 103 tests, `-n auto` on 4 vCPU, each test with its own stack   |
| crash sweep, CI profile        | 3.2–4.9 min | 29 tests, `-n auto` on 4 vCPU                                  |
| dev hub image                  | 3.2–4.0 min | multi-arch QEMU build on every push to `master`                |
| eslint + vitest + client drift | 1.7 min     | four `ng test` projects run one after another                  |

Every check starts at once, so the PR critical path is the slowest single job, pytest unit + component. Queue time is
negligible, about 2 s.

## Work areas

Areas 4 onward each contain a decision.

### 1. Pipeline

- **Build arm64 only for releases.** `dev-image` becomes `linux/amd64` only, saving most of its QEMU time on every push.
  This requires the hosted hub, which follows `edge`, to be on amd64 first; `blizzard-infra` owns that answer.

### 2. PR gate coverage

- **Angular AOT build on every PR.** Run `npm run build` in the frontend job, which today first runs on push or tag.
  Hand the output to the e2e job.
- **E2E on every PR.** Add a job in `upper-tiers.yml` with the three-repo checkout and Chromium, running against the
  frontend job's build. Measure its runtime before enabling it. Area 4 keeps it affordable.
- **Real-winter tests in CI** against a pinned winter (area 3).
- **Journey in CI.** It is currently red: a local run on `master` times out at its first wait, with the first chunk
  still `ready` after 300 s, 5 min into the run. First determine whether that is rot or the local environment, then make
  it green. Its CI needs match the crash sweep's: three-repo checkout, mise, both `uv sync`s, and a git identity, with
  no token or network. Add it as an `upper-tiers.yml` job using the pinned winter. Measure its wall time, then decide
  between every PR and push/tag only. It is a single test that can't be parallelized, so it sets a floor on the pipeline
  it joins.

### 3. Pinned winter source

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

### 4. E2E under xdist

E2E is the one upper tier still run serially. Its ports come from `tests.support.free_port()`, which hands each xdist
worker a disjoint band, and its directories come from `tmp_path`, so most of its state is already per worker.

- Measure the serial e2e job first; its only CI run today is inside the tag `release` workflow.
- Run it under `pytest-xdist`. Its session fixtures (`tests/e2e/conftest.py`) become one copy per worker; decide per
  fixture whether that cost is acceptable or the fixture should be shared.
- The Playwright browser scenario serves one built frontend. Confirm concurrent browser tests don't contend for it, or
  group them onto one worker with `--dist loadgroup`.
- Choose the worker count by measurement, since 4 may not beat 2 with Chromium on a 4-vCPU runner.

Target: e2e job ≤ 33% of its serial time.

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
- Start the hub from a pre-migrated database (the unit tier's prototype database applied to a subprocess). This
  conflicts with `epic:test-architecture`'s ban on touching store files, so state the trade-off explicitly.
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
- **`test_runner_winter_provider.py`'s real-winter tests** take 11–15 s each. Placement follows area 3.

## Out of scope

Making tests implementation-independent belongs to `epic:test-architecture`, which owns any conflict such as area 5's
pre-migrated hub.
