# Train contract

## Names

| Thing             | Spelling          | Cut from                                                                         |
| ----------------- | ----------------- | -------------------------------------------------------------------------------- |
| Release branch    | `release/X.Y`     | the minor's cut commit on `master` (§The cut commit)                             |
| Candidate         | `vX.Y.0-rc.K`     | `release/X.Y`, `K` counting from 1                                               |
| Stable minor      | `vX.Y.0`          | the commit its newest candidate was tagged at                                    |
| Patch             | `vX.Y.P`, `P ≥ 1` | `release/X.Y` ([patches.md](./patches.md))                                       |
| Major's candidate | `vX.0.0-rc.K`     | `master` while it is open for `X`, then `release/X.0` ([majors.md](./majors.md)) |

Before 1.0, `X` is `0`. A tag, once pushed, is never moved, deleted, or reused (`bzh:release`); a branch is never
deleted. Both repositories carry every name in this table ([mock.md](./mock.md)). `release/*` branches and `v*` tags are
protected on both: a branch changes only through a pull request, and only a maintainer's push creates a tag.

## Versions on `master` and on a branch

`pyproject.toml`'s `[project] version` names the release most recently cut on its branch. On `master` it moves only in a
cut commit, to the minor being cut; on `release/X.Y` it stays `X.Y.0` through every candidate and the stable release,
and moves only in a patch's cut commit. A dev build names what `master` is working toward instead: `push.yml` versions
its wheel `X.(Y+1).0.dev<run>+<sha>` from a `pyproject.toml` of `X.Y.Z`, or `X.0.0.dev<run>+<sha>` while `master` is
open for major `X` ([majors.md](./majors.md)). A minor counts trains, not milestones — the comments in `push.yml` and
`release.yml` that read it as a milestone are restated.

`scripts/check-version-tag.sh` compares the tag's version with any `-rc.K` suffix removed against `pyproject.toml`:
`v0.7.0-rc.2` and `v0.7.0` both pair with `version = "0.7.0"`. The build still stamps the tag's full version through
`BLIZZARD_VERSION`, so a candidate's wheel and image report `0.7.0rc2` and the stable build reports `0.7.0`. A tag on a
commit not reachable from its own `release/X.Y` fails the same check before anything is built; the one tag cut from
`master` itself is a major's candidate while `master` is open for it ([majors.md](./majors.md) §Candidates of the
major).

## The cut commit

Every candidate and every patch is tagged at a **cut commit**: the release-preparation commit that sets `pyproject.toml`
to the version being cut and carries the `Release-Cut: <X.Y.Z>` trailer. In it, `blizzard-change-notes cut <X.Y.Z>`
moves the release's notes, as `epic:release-engineering/notes` owns ([assembly](../../notes/spec/assembly.md) §The cut),
and `blizzard:flag-lifecycle` runs in its cut mode, as `epic:release-engineering/dark-launch` owns
([lifecycle](../../dark-launch/spec/lifecycle.md)). Both must pass before the commit merges, so nothing is tagged that
either refuses. A cut commit lands through a pull request like any other, and a candidate's or a patch's tag is pushed
only once its `push` or pull-request run is green.

| Cut                       | Where the cut commit lands                                                   | Its version |
| ------------------------- | ---------------------------------------------------------------------------- | ----------- |
| A minor's first candidate | `master`                                                                     | `X.Y.0`     |
| A further candidate       | `release/X.Y`                                                                | unchanged   |
| A patch                   | `release/X.Y`                                                                | `X.Y.P`     |
| A major's candidate       | `master`, while open, the last of them closing it ([majors.md](./majors.md)) | `X.0.0`     |

## The day

The train runs once a week, on a weekday declared once in `docs/versioning.md` §The train — Monday, UTC, until the
operator changes it. Its procedure may run at any hour of that day. An operator may hold a train, skipping that week,
but never runs two in one week; a held week is noted in the next candidate's notes. A train does two things, in order:

1. **Promote** the newest candidate of the minor cut a week before (§Promotion), if one is waiting.
2. **Cut** the next candidate (§Cutting a candidate), if `master` has moved since the last one was cut. A week with
   nothing landed runs the promotion alone.

## Cutting a candidate

1. Open the minor's cut commit (§The cut commit) against `master` and merge it once its checks pass.
2. Wait for the cut commit's `push` run to go green. A commit no gate proved is never cut; landings after the cut commit
   belong to the next minor and are not carried.
3. Read the `blizzard-mock` commit that run proved it against ([mock.md](./mock.md) §Which mock proved it).
4. Create `release/X.Y` at the cut commit and at the mock commit, in both repositories.
5. Tag `vX.Y.0-rc.1` on both branches' tips ([mock.md](./mock.md) §Branches and tags) and push blizzard's tag last; its
   `release` run is the cut (`bzh:release` §Tag and watch, §Confirm the cut). `epic:release-engineering/first-release`
   opens the pin bump that moves blizzard's own fleet onto it.

A candidate whose `release` run fails is never promoted; the line is repaired forward as `rc.K+1`.

## A further candidate

A fix the week finds is carried to `release/X.Y` exactly as a patch is carried ([patches.md](./patches.md) §Carrying a
fix back), followed by a further cut commit on the branch, and tagged `vX.Y.0-rc.(K+1)` there; the fleet's pin follows
it as it followed `rc.1`. A further candidate does not restart the week: promotion is on the next train day whichever
candidate is newest.

## Promotion

A candidate is promoted when its train day comes, its `release` run is green, and no open issue on `paul-gross/blizzard`
carrying the `release-blocker` label names its line (`release_line: "X.Y"` in the issue's metadata block). Promotion
pushes `vX.Y.0` at the newest candidate's commit, in both repositories.

The `release` run for a stable tag recognises a commit that already carries a candidate tag of the same version whose
`release` run succeeded, and takes that run as its proof: it skips the gate and the three upper tiers and runs only the
backstop compatibility check `epic:release-engineering/compat-floor` owns and `blizzard:flag-lifecycle`'s release-tag
mode (`epic:release-engineering/dark-launch`), then builds and publishes. A promotion writes no cut commit: the notes
were final at the candidate. A stable tag on a commit that carries no green candidate of its version is refused. The
stable build is the same commit as the candidate the fleet ran, rebuilt with its stable version stamped; nothing in it
is new code.

The stable release publishes the image's exact version, its `X.Y` line, and — when it is the highest stable version
published — `latest` ([patches.md](./patches.md) §Publishing). The candidate's GitHub Release stays a prerelease.

## Setting a candidate aside

A candidate with an open `release-blocker` on its train day is set aside rather than promoted: its minor never has a
stable release, its GitHub Release is edited to say so and names the minor that supersedes it, and its branch is kept
but never patched. The train then cuts the next candidate as usual, and the fix the blocker needed rides it from
`master`. Version numbers skip the set-aside minor; nothing is renumbered to hide it.
