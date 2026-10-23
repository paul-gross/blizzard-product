---
epic: test-suite/optimization
refinement: pristine
status: horizon
---

# Plan — `epic:test-suite/optimization`

Every chunk the fleet lands waits on the gate. The unit and component tier, already split into four parallel shards,
takes 11–17 min per shard, and a PR waits on the slowest of them, about 17 min. The goal is to cut that time without
weakening any assertion, and to close the coverage gaps in the PR gate.

| Where                    | Read when                                                                                                       |
| ------------------------ | --------------------------------------------------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Provisional until the three tier epics land — reading where the epic's contract is headed, not building from it |

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
  Hand the output to the UI job.
- **UI tests on every PR.** Once `tests/ui/` exists, add a job in `upper-tiers.yml` with the two-repo checkout and
  Chromium, running the UI project against the frontend job's build. Measure its runtime before making it required.
  Today's e2e never joins the PR gate: it is moving into the service and UI tiers under `epic:test-architecture`, and
  the gate picks up its tests where they land.
- **Journey stays local.** It mints a real winter workspace, and the CI workflows carry no winter source on purpose. It
  is also currently red: a local run on `master` times out at its first wait, with the first chunk still `ready` after
  300 s, 5 min into the run. Determine whether that is rot or the local environment, then make it green where it runs.

### 3. Event-driven waits

The fixed waits outside the service tier get the treatment the hub-tier epic gives `sse_tap` as it ports those tests:
waits that return when the expected thing has happened, and a barrier event where an assertion is negative. That covers
the unit-tier sleeps in `test_cli.py` and `test_runner_harness_opencode_diagnostic.py`. E2E's `sleep(0.5)` tick spacing,
`poll_until` intervals, and 10 s termination allowance are not this epic's: they are rewritten as their scenarios move,
as the service tier's are.

### 4. Slow unit outliers

`test_runner_harness_opencode_diagnostic.py` takes ~164 s of CPU: one 35 s test and ~30 tests at 4–5 s, all spawning
subprocesses. Decide per test whether the subprocess is what's under test. Share one fake binary per module where
possible.

## Out of scope

The service tier's cost is not this epic's: the foundation and the two tier epics replace a stack per test with a tenant
per test on a shared hub, under the initiative's guard that the tier never becomes the gate's slowest job, so nothing
here shortens a stack the initiative is retiring. Making tests implementation-independent belongs to the initiative's
other epics and to `epic:test-architecture`, and where speed conflicts with it, the black-box line wins.
