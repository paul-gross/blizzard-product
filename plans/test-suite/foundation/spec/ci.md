# In CI

## The tasks

Two `mise` tasks run the tier, and `blizzard:service-test` is the two together:

```toml
[tasks.playwright]
description = "Run the Playwright package (tests/) — the service projects, and later the UI project; arguments after -- pass to playwright test"
dir = "tests"
run = "npx playwright test"

[tasks.service-test]
description = "Run the service tier's pytest half — the cases whose subject is Python itself; the Playwright half is `mise run playwright`"
run = "BLIZZARD_SERVICE=1 uv run pytest -n auto tests/service/"
```

`mise run playwright -- --project='service-*'` is the Playwright half, and arguments after `--` reach `playwright test`
unchanged, so the one task serves a profile (`--project=service-hub`), a test (`--grep`), and a shard (`--shard=2/2`).
The `service-test` task's command is unchanged, and its description narrows to the half it runs. A third task,
`playwright-check`, runs [the static job's](#the-static-job) checks locally.

## The service jobs

`upper-tiers.yml` gains a job beside `service-tier`:

| Job                   | Matrix                              | Runs                                                                                                                    |
| --------------------- | ----------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `service-tier`        | `shard: [1, 2, 3]`, unchanged       | the pytest half, unchanged                                                                                              |
| `service-playwright`  | `shard: [1, 2]`, `fail-fast: false` | `mise run playwright -- --project='service-*' --project=harness --shard=${{ matrix.shard }}/2`                          |
| `service-tier-result` | —                                   | named `service tier (blizzard:service-test)`; needs both jobs above, and passes only when every shard of each succeeded |

`service-playwright` checks out `blizzard` and `blizzard-mock` as `service-tier` does, through
`./blizzard/.github/actions/resolve-mock-ref` with the `mock-ref` job's sha and `persist-credentials: false`, and
`uv sync`s both, so the daemons it starts are the installed binaries. It sets up Node with `jdx/mise-action` at the
workflow's pinned mise version with `install: true`, so Node comes from `mise.toml`'s exact pin, then restores
`tests/node_modules` from `actions/cache`, keyed on the OS, the Node version, and
`hashFiles('blizzard/tests/package-lock.json')`, and runs `npm ci` in `tests/` only on a miss. It sets `CI=1`. The
required check's name is unchanged, so no repository setting moves.

Each shard is its own machine, with its own shared hubs and common tenants; nothing crosses between them. The `harness`
project runs in the shards beside the service projects, so the harness's own specs are part of the required check.

`release.yml` takes the same shape. Its single `service-tier` job is renamed `service-tier-pytest`, unsharded as now,
`service-playwright` is added with the same two shards and steps, its mock checkout at the tag's own ref as
`service-tier`'s is today, and a `service-tier-result` job named `service tier (blizzard:service-test)` needs both.
`build`'s `needs` names `service-tier-result` in place of `service-tier`.

## Failure evidence

A failing `service-playwright` shard uploads, with `actions/upload-artifact` and `if: failure()`, one artifact named
`service-playwright-<shard>`, holding `tests/playwright-report/`, `tests/test-results/`, which holds the traces and the
JSON report, and `tests/.run/`, which holds every daemon's log. It is kept for fourteen days. A red run can be read from
the artifact without reproducing it.

## The static job

Playwright runs TypeScript without checking its types, so the gate checks them. `gate.yml` gains a `playwright-static`
job: Node from `mise.toml` as above, `npm ci` in `tests/`, then `npm run typecheck` (`tsc --noEmit`), `npm run lint`,
and `npm run generate:wire` followed by `git diff --exit-code -- tests/harness/client/wire/`. It starts no daemon and
checks out no `blizzard-mock`. It is a required check of its own, `gate / playwright static`, recorded where
`docs/ci.md` lists the others.

## The winter-free guard

`tests/test_winter_free_ci.py` keeps every job that runs tests free of winter. It learns the Playwright jobs in three
places:

- `_RUNS_TESTS` matches `mise run playwright` and `playwright test` besides `pytest` and `mise run e2e`, so
  `service-playwright` counts as a job that runs tests and its checkouts are held to `persist-credentials: false`.
- The set of jobs it must find gains `("upper-tiers.yml", "service-playwright")`,
  `("release.yml", "service-playwright")`, and the cross-check's jobs.
- `_platform_modules` scans `*.ts` under `tests/harness/` and `tests/service/` as well as `*.py`, for the same
  winter-bound pattern, skipping `node_modules`.

## The cross-check runs

`.github/workflows/isolation-cross-check.yml` runs [the cross-check](./isolation.md#the-cross-check) on `schedule`
(weekly, Monday 06:00 UTC), on `workflow_dispatch`, and as a `workflow_call` that `release.yml` makes beside its tiers.
It has three jobs: `declared` and `raised` each run
`mise run playwright -- --project='service-*' --workers=<twice the CI default>` unsharded, the second with
`BLIZZARD_ISOLATION_RAISE=1`, each uploading its JSON report and, on failure, its evidence as above; `compare` downloads
both reports and runs `harness/scripts/cross-check.ts`. `compare` runs `if: ${{ !cancelled() }}`, so it reads both
reports even when a run went red, and is the workflow's verdict: it fails when either report is missing or the two
disagree, and names each disagreeing test, while a test both runs failed fails the workflow through the runs' own jobs.

In `release.yml`, `build` needs the call's `compare`, so a release is cut only over a suite whose placements hold. It
never runs on a pull request or a `master` push.
