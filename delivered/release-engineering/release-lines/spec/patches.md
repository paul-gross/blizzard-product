# Patch contract

## Which lines are patched

A **line** is a minor's release branch once that minor has a stable release. The lines that receive patches:

| When                     | Lines patched                                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------- |
| Before 1.0               | The newest stable minor alone                                                                                       |
| Within a major, past 1.0 | The newest stable minor of the newest major                                                                         |
| After a new major ships  | Also the previous major's final minor, for three months from the new major's `.0` release date, critical fixes only |

A candidate that has not yet been promoted is not a line: a fix to it is a further candidate ([train.md](./train.md) §A
further candidate). A set-aside minor is never a line. Once the three months end, the old major's line receives nothing
more; how long its runners keep working against the new major's hub is the skew window's, owned by
`epic:release-engineering/first-release`, and holds for the whole of that major. The table is published in
`docs/versioning.md` §Support, with each closing date written out the day the major ships.

A patch is for what cannot wait the week: a hole in security, data lost or corrupted, or a fleet that has stopped
working. Anything else rides the next train. The same three classes are what *critical* means on an old major's line, so
the old line and the newest one apply one test.

## What a patch may carry

A patch carries the fix, the tests that prove it, and the change note `epic:release-engineering/notes` requires — and
nothing else. It adds no route, field, configuration key, span, event, or egress column; turns no flag on; and carries
no store revision. A revision on a branch would sit at a different place in the chain than the same revision on
`master`, and a store migrated by the patch would then skip whatever `master` added between them on its next upgrade; a
fix to stored data in a patch is a repair the daemon applies idempotently at start, or a CLI verb the operator runs,
never a revision. The rule lives in `blizzard-context` `standards/` as `bzh:patch-scope`, one rule a review asserts, and
the compatibility check that holds a release branch to every release already cut on its line
(`epic:release-engineering/compat-floor`) enforces the part of it a diff can see.

## Carrying a fix back

Every fix lands on `master` first, by the ordinary path. Then, for each line that carries the bug:

1. Branch `patch/X.Y/<slug>` from `release/X.Y`, and `git cherry-pick -x` each commit of the fix in order, adapting by
   hand where the line has drifted. When the fix needed `blizzard-mock` to change, carry the mock's commits the same way
   onto a same-named branch of the mock ([mock.md](./mock.md) §Resolving the mock's ref).
2. Open a pull request into `release/X.Y`, which runs §Proving a release branch.
3. Once every fix for the patch is merged, merge the patch's cut commit into `release/X.Y` ([train.md](./train.md) §The
   cut commit): it sets `pyproject.toml` to `X.Y.P` and carries `Release-Cut: X.Y.P`, and in it
   `blizzard-change-notes cut X.Y.P` moves the patch's notes (`epic:release-engineering/notes`,
   [assembly](../../notes/spec/assembly.md) §The cut) and `blizzard:flag-lifecycle` runs in its cut mode
   (`epic:release-engineering/dark-launch`, [lifecycle](../../dark-launch/spec/lifecycle.md)).
4. Tag `vX.Y.P` at that branch tip in both repositories, blizzard's last. Its `release` run is the full suite; a patch
   has no candidate to borrow a proof from.

A fix for code `master` has already removed is written on the line alone, as a pull request straight into `release/X.Y`.
Its commit carries a `Line-only:` trailer saying why it has no `master` counterpart.

Every commit on a release branch after its cut is therefore one of three: a cherry-pick whose
`(cherry picked from
commit <sha>)` line names a commit on `master`, a commit with a `Line-only:` trailer, or a cut
commit. `scripts/check-line-provenance.sh` asserts it over the commits a pull request adds, and fails the pull request
otherwise, so *fix `master` first* is a rule the branch keeps rather than one its author remembers.

## Proving a release branch

`pr.yml` runs on pull requests into `release/**` as well as `master`: the gate (`gate.yml`), the upper tiers
(`upper-tiers.yml`, bounded crash sweep), the provenance check above, and the compatibility check that holds the branch
to every release already cut on its line, beginning with its first, as `epic:release-engineering/compat-floor` owns
([baseline](../../compat-floor/spec/baseline.md) §The tags). The upper tiers run against the mock's ref for that branch
([mock.md](./mock.md) §Resolving the mock's ref), never the mock's `master`. A push to a release branch publishes
nothing: `push.yml` stays `master`'s, and `edge` and the dev wheel never come from a line.

## Publishing

`scripts/image-tags.sh` stops being a function of one tag alone: it also takes the stable versions already published,
read from the repository's tags, and moves `latest` only when the version being published is the highest stable version
there is. Every stable release and patch publishes its exact version and moves its own `X.Y` line; a patch on an old
major's line never moves `latest`. The GitHub Release follows the same rule, created with `--latest=false` unless it is
the highest stable version. `tests/test_image_tags.py` covers the old-line case beside the cases it holds today.

A line publishes its patches the way it published its first release. A release branch cut before
`epic:release-engineering/supply-chain`'s signing lands keeps publishing unsigned patches for as long as it is
supported, and signing is not backported to it.
