# The pull request gate

## The board's build on every pull request

A pull request today type-checks the board's templates only as far as `bundle-check`'s hub build reaches, and never
builds the runner's board at all; the full ahead-of-time build first runs on `master`, in `push.yml`'s `dev-build`.
`gate.yml`'s `frontend` job gains a step after `npm ci`: `npm run build`, which runs `build:hub` and `build:runner` into
`src/blizzard/static/hub` and `src/blizzard/static/runner`, as `angular.json`'s `outputPath`s place them. A template
error the runner's board alone would catch now fails the pull request.

The job then uploads `src/blizzard/static/` as the artifact `web-dist`, kept for one day, which is the hand-off the UI
job reads. `bundle-check` keeps its own build, since it builds with `--stats-json` into a scratch directory; whether it
can read the stats from the new build instead is a saving this epic takes only if `bundle-check.js` can be pointed at it
without changing what it checks.

## The UI job

`tests/ui/` is `epic:test-architecture`'s, and the job waits until it exists. When it does, `gate.yml` gains `ui`,
needing `frontend`: the two-repo checkout of `blizzard` and `blizzard-mock` with `persist-credentials: false`, `uv sync`
in both, Node from `mise.toml`, `npm ci` in `tests/`, `npx playwright install --with-deps chromium`, then `web-dist`
downloaded into `src/blizzard/static/` and `mise run playwright -- --project='ui-*'`. A failure uploads the same
evidence the service shards do.

The job lives in `gate.yml` rather than in `upper-tiers.yml`, as the plan first placed it, because the hand-off decides
it: `pr.yml` calls the two workflows side by side, and a job in one cannot wait on a job in the other. Placing it beside
`frontend` keeps the build made once and the UI tests started the moment it exists.

It is not a required check when it lands. Its runtime is recorded over its first twenty runs on `master`; it becomes
required, under the name `gate / ui`, once those runs show it finishing inside the slowest `python-tests` shard, so it
never becomes the gate's critical path. `test_winter_free_ci.py`'s set of expected test jobs gains `("gate.yml", "ui")`.

## The journey, green where it runs

The journey stays local, since it mints a real winter workspace and no workflow carries a winter source. It is red: its
first wait, `wait_status(hub, grp_api, {"done"}, timeout=300.0)` in `tests/journey/test_journey_rehearsal.py`, times out
with the grouped chunk never claimed. The chunk leaves `ready` only when the runner host started by
`tests/crash/support.py`'s `start_runner` claims it, on the winter provider over envs `e1` to `e4` that
`_widen_runner_pool` configures.

Two changes make the next failure readable, whatever its cause, and land first:

- `start_runner` waits for the runner's `GET /api/health` before returning, as every other daemon launch does, so a
  runner that never started fails there, with its log, instead of five minutes later as a chunk that never moved.
- `wait_status` records each distinct status it sees with its time, and on timeout fails with that history and the tail
  of the runner's `daemon.log`, so a chunk stuck at `ready` is told apart from one that was claimed and stalled.

Then the cause is found by running `mise run journey` in a provisioned feature environment and reading those. If the
runner cannot reach a winter env, the defect is the test's environment, and the fix is to the journey's setup, so it
mints the envs it names; if the runner refuses or skips the chunk, it is rot, and the fix is to the product or the test,
whichever drifted. Either way the change that turns it green says which it was. The journey's assertions are not
loosened to make it pass.
