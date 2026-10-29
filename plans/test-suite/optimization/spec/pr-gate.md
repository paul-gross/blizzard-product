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

`epic:test-suite/fleet-tier` lands the UI tests with a job of their own in `upper-tiers.yml`, under `full_suite`, that
builds the board itself, as the required check `upper-tiers / ui (blizzard:ui)`. This epic moves it: `gate.yml` gains
`ui`, needing `frontend`, with the two-repo checkout of `blizzard` and `blizzard-mock` with
`persist-credentials: false`, `uv sync` in both, Node from `mise.toml`, `npm ci` in `tests/`,
`npx playwright install --with-deps chromium`, then `web-dist` downloaded into `src/blizzard/static/` and
`mise run playwright -- --project='ui-*'`. A failure uploads the same evidence the service shards do.

The job moves to `gate.yml` because the hand-off decides it: `pr.yml` calls the two workflows side by side, and a job in
one cannot wait on a job in the other. Placing it beside `frontend` keeps the build made once and the UI tests started
the moment it exists.

The move lands only once the job's runtime, recorded over its runs in `upper-tiers.yml`, shows it finishing inside the
slowest `python-tests` shard, so it never becomes the gate's critical path. The change renames the required check to
`gate / ui` in branch protection and blizzard-context's packaging page, and `test_winter_free_ci.py`'s set of expected
test jobs moves its entry from `upper-tiers.yml` to `("gate.yml", "ui")`.
