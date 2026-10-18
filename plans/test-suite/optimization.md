---
epic: test-suite/optimization
refinement: refined
status: horizon
---

# Plan — `epic:test-suite/optimization`

Every chunk the fleet lands waits on the gate. The unit and component tier, already split into four parallel shards,
takes 11–17 min per shard, and a PR waits on the slowest of them, about 17 min. The goal is to cut that time without
weakening any assertion, and to close the coverage gaps in the PR gate.

## Baseline

Successful pushes to `master`, 18 October 2026:

| Job                            | Duration                | Shape                                                                    |
| ------------------------------ | ----------------------- | ------------------------------------------------------------------------ |
| pytest unit + component        | 11.0–16.9 min per shard | 4 shards split by a hash of each test's id, `-n auto` on 4 vCPU each     |
| crash sweep, CI profile        | 7.0–7.2 min             | 43 tests, `-n auto` on 4 vCPU                                            |
| service tier                   | 3.2–5.8 min per shard   | 237 tests in 3 shards, `-n auto` on 4 vCPU, each test with its own stack |
| dev hub image                  | 3.0–4.5 min             | multi-arch QEMU build on every push to `master`                          |
| eslint + vitest + client drift | 1.9–2.2 min             | four `ng test` projects run one after another                            |

Every check starts at once, so the PR critical path is the slowest single job, a pytest unit + component shard, and the
crash sweep is next behind it. Queue time is negligible, about 2 s.

## Work areas

### 1. Pipeline

- **Balance the shards by duration.** A hash of each test's id spreads tests evenly by count but not by time, so one
  shard runs six minutes longer than another. Splitting by recorded durations brings every shard toward the mean, which
  shortens the critical path with no test touched.
- **Build arm64 only for releases.** `dev-image` becomes `linux/amd64` only, saving most of its QEMU time on every push.
  This requires the hosted hub, which follows `edge`, to be on amd64 first; `blizzard-infra` owns that answer.

### 2. PR gate coverage

- **Angular AOT build on every PR.** Run `npm run build` in the frontend job, which today first runs on push or tag.
  Hand the output to the e2e job.
- **E2E on every PR.** Add a job in `upper-tiers.yml` with the two-repo checkout and Chromium, running against the
  frontend job's build. Measure its runtime before enabling it. Area 3 keeps it affordable.
- **Journey stays local.** It mints a real winter workspace, and the CI workflows carry no winter source on purpose. It
  is also currently red: a local run on `master` times out at its first wait, with the first chunk still `ready` after
  300 s, 5 min into the run. Determine whether that is rot or the local environment, then make it green where it runs.

### 3. E2E under xdist

E2E is the one upper tier still run serially. Its ports come from `tests.support.free_port()`, which hands each xdist
worker a disjoint band, and its directories come from `tmp_path`, so most of its state is already per worker.

- Measure the serial e2e job first; its only CI run today is inside the tag `release` workflow.
- Run it under `pytest-xdist`. Its session fixtures (`tests/e2e/conftest.py`) become one copy per worker; decide per
  fixture whether that cost is acceptable or the fixture should be shared.
- The Playwright browser scenario serves one built frontend. Confirm concurrent browser tests don't contend for it, or
  group them onto one worker with `--dist loadgroup`.
- Choose the worker count by measurement, since 4 may not beat 2 with Chromium on a 4-vCPU runner.

Target: e2e job ≤ 33% of its serial time.

### 4. Event-driven waits

The fixed waits outside the service tier get the treatment the hub-tier epic gives `sse_tap` as it ports those tests:
waits that return when the expected thing has happened, and a barrier event where an assertion is negative. That covers
e2e's `sleep(0.5)` tick spacing, `poll_until` intervals, the 10 s termination allowance, and the unit-tier sleeps in
`test_cli.py` and `test_runner_harness_opencode_diagnostic.py`.

### 5. Slow unit outliers

`test_runner_harness_opencode_diagnostic.py` takes ~164 s of CPU: one 35 s test and ~30 tests at 4–5 s, all spawning
subprocesses. Decide per test whether the subprocess is what's under test. Share one fake binary per module where
possible.

## Out of scope

The service tier's cost is not this epic's: the foundation and the two tier epics replace a stack per test with a tenant
per test on a shared hub, under the initiative's guard that the tier never becomes the gate's slowest job, so nothing
here shortens a stack the initiative is retiring. Making tests implementation-independent belongs to the initiative's
other epics and to `epic:test-architecture`, and where speed conflicts with it, the black-box line wins.
