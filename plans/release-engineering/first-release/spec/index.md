# First release specifications

The technical contracts behind `epic:release-engineering/first-release`. Product intent begins at the
[epic plan](../index.md), and the rhythm and the skew window it restates are the [initiative's](../../initiative.md).
Comparing a change against the supported releases, and refusing a runner outside the window at registration, are
`epic:release-engineering/compat-floor`'s; the weekly train, release branches, and patches are
`epic:release-engineering/release-lines`'; the content of a release's notes is `epic:release-engineering/notes`'. This
spec cuts one release and moves one instance onto releases; it defines none of theirs.

The epic spans four repositories and the workspace. In `blizzard`: the revision queries on both `migrate` verbs, the
release tooling under `scripts/` and `.github/workflows/`, and the operator documents `docs/versioning.md`,
`docs/upgrade.md`, `docs/rollback.md`, and `docs/ci.md`. In `blizzard-infra`: the deployment payload under `deploy/`,
its contract and smoke tests under `tests/`, a new workflow under `.github/workflows/`, and `docs/deployment.md` and
`docs/rollback.md`. In `blizzard-context`: `workflows/release.md` and a new workflow beside it. In the workspace:
`context/project/local-instance.md` and `context/project/post-delivery.md`. `blizzard-mock` is unchanged by this epic:
the release workflow checks it out at its default branch for the first cut, and its matching `release/0.2` branch and
tag are created after the fact, as an attended step of `epic:release-engineering/release-lines`'
[carry-over](../../release-lines/spec/carry-over.md).

| Where                            | Read when                                                                                                                                       |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| [versions.md](./versions.md)     | Cutting `0.2.0` — the candidate and its promotion, the tag check, the version `master` carries, and the rewrites of versioning, upgrade, and CI |
| [hub-pin.md](./hub-pin.md)       | Changing the hosted hub's deployment — the pin file, the updater applying it forward and back, the hold, and their tests                        |
| [pin-change.md](./pin-change.md) | Building the workflow that opens the hub's pin change when a release publishes — its trigger, its choice of release, its token, its description |
| [runner-pin.md](./runner-pin.md) | Moving the runner host onto releases — where its pin lives, installing a release's wheel, graphs, the redeploy proof, and going back            |
| [procedures.md](./procedures.md) | Writing the procedures — cutting and promoting, moving a deployment onto a release or back off one — and what the workspace runbook keeps       |
| [carry-over.md](./carry-over.md) | Landing the epic on an instance that follows `master` today — the order of landings and the one-time cutover of each half                       |
