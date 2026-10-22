# Release lines specifications

The technical contracts behind `epic:release-engineering/release-lines`. Product intent begins at the
[epic plan](../index.md); implementation enters at the concern below. The supported releases, and every compatibility
check that compares against them — on a pull request to `master`, on one to a release branch, held to every release
already cut on that line, and at a tag — are owned by `epic:release-engineering/compat-floor`, as is the upgrade
rehearsal. What a release's notes carry, and the notes step of a cut, are owned by `epic:release-engineering/notes`; the
flag-lifecycle step of a cut by `epic:release-engineering/dark-launch`; the first cut, the skew-window policy, and the
dogfood instance's pins are owned by `epic:release-engineering/first-release`. This spec decides when a release is cut,
from which commit, under which name, and from which branch; it redefines none of theirs.

It builds on what the release machinery already does. A pushed `v*` tag is the release
(`workspace:/context/project/workflows/release.md`): the `release` workflow (`.github/workflows/release.yml`) runs the
gate and the three upper tiers, asserts `pyproject.toml`'s version against the tag (`scripts/check-version-tag.sh`),
builds the wheel and the multi-arch hub image, fans the image's tags out through `scripts/image-tags.sh`, and publishes
a GitHub Release, marking `-rc.` tags as prereleases. A green push to `master` publishes the `edge` and `sha-<sha>`
images and a dev wheel versioned `0.<minor>.0.dev<run>+<sha>` (`.github/workflows/push.yml`). Pull requests run the gate
and the upper tiers only against `master` (`pr.yml`), and the upper tiers resolve `blizzard-mock` to a same-named branch
or else its `master` (`upper-tiers.yml`); the tag run checks out the mock's default branch. Both stores are Alembic
trees driven by `src/blizzard/foundation/store/migrations.py`'s `MigrationRunner`, the hub's under
`src/blizzard/hub/store/migrations/versions/` and the runner's under `src/blizzard/runner/store/migrations/versions/`,
every revision keeping a working `downgrade()`.

Paths are relative to the `blizzard` repository's root.

The epic reaches beyond `blizzard`'s source. `blizzard-mock` gains release branches and tags of its own
([mock.md](./mock.md)). In the workspace, `workspace:/context/project/workflows/` gains a procedure per operation beside
`release.md`, written by hand ([carry-over.md](./carry-over.md)). In `blizzard-context`, `standards/` gains
`bzh:patch-scope`, the rule for what a patch may carry ([patches.md](./patches.md) §What a patch may carry), and
`verification/blizzard.md` declares `blizzard:store-baseline` ([fold.md](./fold.md) §The two proofs). In `blizzard`'s
own docs, `docs/versioning.md` gains the train, the branches, the support promise, and the published major dates, and
`docs/ci.md` describes the workflows on release branches; `docs/upgrade.md` names the baseline floor each fold sets
([fold.md](./fold.md)).

| Where                            | Read when                                                                                                                         |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| [train.md](./train.md)           | Building the weekly train — version numbers on `master` and a branch, tag and branch names, cutting a candidate, promoting it     |
| [patches.md](./patches.md)       | Patching a released line — what a patch may carry, the support promise, carrying a fix back, proving a release branch, publishing |
| [mock.md](./mock.md)             | Releasing `blizzard-mock` beside blizzard — which mock commit a release was proven against, its branches and tags, resolving refs |
| [majors.md](./majors.md)         | Scheduling a major — its published date, opening `master` for it, the outgoing final line, cutting it                             |
| [fold.md](./fold.md)             | Folding a store's migration history into a baseline — the floor, the baseline revision, refusing an older store, the two proofs   |
| [carry-over.md](./carry-over.md) | Landing the epic — its order behind the first release, the first train, the procedures, and what `epic:cadence` later takes over  |
