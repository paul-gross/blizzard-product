# Fold contract

Both store trees are folded together, on the same occasion, at the same floor: the hub's
(`src/blizzard/hub/store/migrations/versions/`) and the runner's (`src/blizzard/runner/store/migrations/versions/`).

## When, and at which floor

| Before 1.0                                                      | Past 1.0                                                |
| --------------------------------------------------------------- | ------------------------------------------------------- |
| The first train of each calendar month ([train.md](./train.md)) | Opening `master` for a major ([majors.md](./majors.md)) |

The fold lands on `master` as a pull request merged before that train cuts its candidate, so the candidate ships the
baseline and the fleet runs it for the week like any other change. The **floor** is the oldest supported release on the
day the fold is written: the entry `blizzard-compat releases --json` marks `oldest`, computed by
`epic:release-engineering/compat-floor` ([baseline](../../compat-floor/spec/baseline.md) §The tool) and never derived
here. For each tree, the **floor revision** is the last revision that release carries, read from the head revision id in
`contracts/store/hub.json` or `contracts/store/runner.json` at the floor's tag
([surfaces](../../compat-floor/spec/surfaces.md) §Store schema). Every revision up to and including the floor revision
is folded; every revision after it is kept as it is. A fold whose floor revision is already the tree's baseline folds
nothing and is skipped for that tree.

## The baseline revision

The fold replaces the absorbed revisions with one file that keeps the floor revision's identifier and its file name:

- `revision` is the floor revision's id, and `down_revision` is `None`. The first kept revision already names that id as
  its `down_revision`, so the chain above the baseline is untouched.
- `upgrade()` recreates, in one step, every table, column, index, and constraint the floor's tree builds, and inserts
  every row its history inserts into an empty store. It is written as explicit Alembic operations, generated once and
  then frozen — never `metadata.create_all()` against the live `schema.py`, which keeps moving above the baseline.
- `downgrade()` drops everything, back to an empty store, so `tests/test_store_migrations.py::test_migrate_up_and_down`
  holds unchanged: every revision in the tree, the baseline included, still walks back.
- `BASELINE_FLOOR` names the floor release's tag, and `ABSORBED` is the frozen set of every revision id the baseline
  stands for — the ones this fold absorbed and every id the previous baseline's `ABSORBED` held.

`scripts/fold_store.py <floor-tag>` writes it. It installs the floor's released wheel into a throwaway environment,
renders the baseline's operations from that release's own `schema.py` metadata through Alembic's autogenerate against an
empty store, so the column types stay the portable ones the schema declares rather than what one dialect reflects, reads
the rows the history leaves in a freshly migrated store, and deletes the absorbed revision files from the tree.

## Turning an older store away

`MigrationRunner` (`src/blizzard/foundation/store/migrations.py`) learns one refusal, reading `BASELINE_FLOOR` and
`ABSORBED` from the tree's base revision through Alembic's `ScriptDirectory`. A store whose current revision is not in
the tree but is in `ABSORBED` is older than every supported release; `hub migrate`, `runner migrate`, both daemons'
start-up revision checks, and `runner ingest` (`src/blizzard/runner/ingest.py`, for its source store) refuse it with a
`StoreBelowFloorError` naming the store, its revision, and the release to install and migrate through first —
`BASELINE_FLOOR` — and change nothing. A tree whose base revision declares neither attribute behaves as it does today,
so the refusal lands ahead of the first fold and stays inert until one exists. A revision in neither the tree nor
`ABSORBED` keeps today's meaning, a store a newer build migrated. A store at or past the floor revision sees nothing
change: its `alembic_version` row already names a revision the tree holds.

`docs/upgrade.md` names each fold's floor beside the release that ships it, so an operator reading the guide sees which
release to step through before they meet the refusal.

## The two proofs

**The baseline matches the history it replaced.** `tests/test_store_baseline.py` builds two stores for each tree on each
database blizzard supports, sqlite and postgres: one by migrating an empty store with the floor's released wheel, one by
upgrading an empty store to the baseline alone. It compares them table for table — columns with their types,
nullability, and defaults; primary keys, foreign keys, unique constraints, and indexes; and every row — and fails on any
difference. It skips while the base revision declares no `BASELINE_FLOOR`. It cannot run in the gate, which is
token-free and reaches no network beyond package installs, while this proof downloads the floor's wheel from its GitHub
Release and needs a postgres service no other tier runs. So it is a job of its own, `store-baseline`, declared as
`blizzard:store-baseline` in `blizzard-context`'s `verification/blizzard.md`: `pr.yml` runs it on pull requests that
touch either tree's `versions/` directory, which every fold's pull request does, and `release.yml` on every candidate
and patch tag. A stable tag relies on its candidate's run ([train.md](./train.md) §Promotion), as it does for the
rehearsal.

**The rehearsal still walks from the floor.** The upgrade rehearsal is `epic:release-engineering/compat-floor`'s
([rehearsal](../../compat-floor/spec/rehearsal.md)), and it already starts from every supported release. After a fold it
walks the oldest one's store forward to the candidate and back down only as far as the baseline, never past it, which
proves the revisions kept above the floor still downgrade onto a store the floor itself wrote.

## The procedure

`blizzard-context`'s `workflows/store-fold.md` carries a fold from the floor to a merged pull request: read the floor
from `blizzard-compat releases --json`, run `scripts/fold_store.py <floor-tag>`, open the pull request with
`store-baseline` green and the rehearsal walking down only as far as the baseline, merge it before the train cuts its
candidate, and name the floor beside that release in `docs/upgrade.md`. It lands with the tool and the proof. Running it
is an operation, carried out on its day by an operator or an agent they ask, until `epic:cadence` schedules it.
