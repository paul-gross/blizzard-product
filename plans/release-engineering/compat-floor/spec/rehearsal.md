# Upgrade rehearsal

The static checks prove a document did not shrink. The rehearsal proves the upgrade an operator performs works: a
supported release's real hub and runner host, holding real-shaped data, upgraded hub first to the candidate, with the
old runner still finishing work, and the store walked back to where it came from.

## Where it runs

`.github/workflows/rehearsal.yml` is reusable (`workflow_call`), called by `push.yml` after the gate on every push to
`master`, and by `release.yml` before the image push on every candidate tag, so neither `edge` nor a candidate publishes
from a commit whose rehearsal failed. A stable tag does not run it again: `epic:release-engineering/release-lines`
promotes a candidate by tagging the same commit, so the stable release relies on its candidate's run. It does not run on
pull requests: it takes minutes, and the compatibility check already holds every pull request to the same releases
statically. Its local equivalent is `mise run rehearsal`
(`BLIZZARD_REHEARSAL=1 uv run pytest -m rehearsal tests/rehearsal/`), and its method is `blizzard:rehearsal`. A tree
with no supported release skips it and says so.

It runs once per supported release ([baseline.md](./baseline.md)) that has a published GitHub Release, as a matrix
over them: a candidate whose `release` run went red published no wheel to upgrade from, and is skipped until the line's
next candidate replaces it as the newest tag. Each leg runs the hub
store on SQLite and on Postgres, the backend the hosted hub runs (`packaging/docker/`); the runner store is SQLite only.
The Postgres leg is a `services: postgres` container on the job, the first live Postgres in CI, and it also migrates an
empty hub store and compares its reflection with `contracts/store/hub.json` ([surfaces.md](./surfaces.md) §Store
schema).

`push.yml`'s `dev-image` needs the rehearsal beside `[gate, upper-tiers, dev-build]`, so `edge` advances only on a
commit whose upgrade from every supported release succeeded.

## Two installations side by side

Each leg holds two isolated virtualenvs and never imports either into the test process; every daemon and verb is a
subprocess, as the e2e tier drives them (`tests/e2e/test_acceptance_loop.py`).

- **The baseline**, the release's own published wheel, downloaded from its GitHub Release
  (`gh release download <tag> --pattern '*.whl'`), so the rehearsal runs exactly what shipped.
- **The candidate**, the wheel the calling workflow built from HEAD (`scripts/build-wheel.sh`).

`blizzard-mock` is checked out at the baseline's tag, so the mock harness and forge the old runner talks to are the ones
it was proven against; `epic:release-engineering/release-lines` cuts the mock's tags. A baseline with no mock tag runs
the mock's `master` ([carry-over.md](./carry-over.md)).

## The journey

1. **Seed the baseline.** The baseline's `blizzard hub init`, `blizzard runner init`, and host registration, as the
   journey tier does; then the mock's `blizzard-mock-data scenario board` against the baseline's hub and host, which
   fills every hub and runner table the release has with real-shaped rows, and one chunk driven to `delivering` and left
   there by the baseline's runner. The leg records each table's row count and a fingerprint of the seeded chunks' API
   views.
2. **Upgrade the hub.** Stop the baseline hub; run the candidate's `blizzard hub migrate` against its store; start the
   candidate hub. It must start, answer `/api/ready`, and serve the seeded chunks' views unchanged in every field the
   baseline's document carried.
3. **Keep the old runner.** The baseline runner host, untouched, registers with the candidate hub. The handshake must
   admit it ([handshake.md](./handshake.md)). It claims, works, and lands a fresh chunk through the mock forge and mock
   harness, and finishes the chunk it left delivering.
4. **Upgrade the runner.** Stop the baseline host; run the candidate's `blizzard runner migrate`; start the candidate
   host. It resumes its runners and lands one more chunk.

A step that fails names the step, the baseline, and the backend, and attaches both daemons' logs.

## The store walk

After the journey, each store is walked back to the baseline and forward again.

1. Stop both candidate daemons. Read the baseline's head revision for each store from the baseline tag's
   `contracts/store/*.json` ([surfaces.md](./surfaces.md) §Store schema).
2. Run the candidate's `migrate --down <revision>` on each store, then start the baseline hub and host on them. Every
   table the baseline had holds at least the rows it held after step 1, the seeded chunks' views match their
   fingerprints, and the chunks landed since read back.
3. Stop them, run the candidate's `migrate` again, and start the candidate daemons; each answers ready.

The walk back is what an operator does to roll back (`docs/rollback.md`), and it is lossless only because every revision
inside the window expands (`bzh:store-expand-contract`). When `epic:release-engineering/release-lines` folds the store's
history into a baseline at the oldest supported release, the walk proves the fold too: down to that release's revision,
and no further, is as far as any supported store ever walks.
