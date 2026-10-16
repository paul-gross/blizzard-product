# Carry-over contract

## Where it starts

`blizzard` has one tag, `v0.1.0-rc.1`, cut on `master`, and no release branch; `blizzard-mock` has neither tags nor
release branches, and its `pr.yml` and `push.yml` run on `master` alone. `epic:release-engineering/first-release` lands
before this epic's first train: it cuts the first real release, moves the dogfood instance onto pinned releases, and
opens a pin bump whenever a release publishes, which is what carries blizzard's own fleet from one candidate to the
next. This epic adds the rhythm on top of that cut; it changes nothing about how a single tag is cut and confirmed
(`bzh:release`), which stays the step every procedure here ends in.

## The order of landings

1. **What the first cut needs**, landed before `epic:release-engineering/first-release` cuts it: `check-version-tag.sh`
   comparing the tag with its `-rc.K` suffix removed, `pyproject.toml` naming the last cut and moving only in a cut
   commit ([train.md](./train.md) §Versions on `master` and on a branch, §The cut commit), and `push.yml`'s dev version
   naming the next minor. Each is covered by the tests beside today's (`tests/test_check_version_tag.py`,
   `tests/test_release_workflow.py`, `tests/test_push_workflow.py`). The first cut is then a cut commit on `master` and
   a candidate tag there, with no branch yet.
2. **The written promise and the procedures.** `docs/versioning.md` gains §The train, §Support, and, past 1.0,
   §Schedule, and drops its reading of the minor as a milestone; `docs/ci.md` describes the release-branch runs. In
   `blizzard-context`, `workflows/` gains `release-train.md` (cutting the weekly candidate and promoting it, or setting
   it aside), `release-patch.md` (patching a released line), `release-major.md` (opening `master` for a major and
   cutting it on its date), and `store-fold.md` (folding the store's history), each ending in `bzh:release` for the tag
   itself; `standards/` gains the rule for what a patch may carry; `verification/blizzard.md` declares the baseline
   proof.
3. **The lines, after the first cut.** The reachability and `opening` rules of `check-version-tag.sh`
   ([train.md](./train.md) §Versions on `master` and on a branch, [majors.md](./majors.md) §Candidates of the major);
   `pr.yml` on `release/**` in both repositories; the provenance check ([patches.md](./patches.md) §Carrying a fix
   back); the mock-ref composite action and the `proven-against` artifact ([mock.md](./mock.md)); `image-tags.sh`'s
   `latest` rule ([patches.md](./patches.md) §Publishing); and the stable tag's borrowed proof ([train.md](./train.md)
   §Promotion), each covered beside today's tests (`tests/test_image_tags.py` among them). The same landing gives the
   first cut its line after the fact: `release/X.Y` at the first candidate's commit in `blizzard`, and in
   `blizzard-mock` at the commit that candidate's tag run checked out, read from that run's log since nothing recorded
   it then, with the mock's tag of the same name there. Branch and tag protection for `release/*` and `v*` is applied on
   both by hand with `gh api` in the same change and asserted by the procedure that first relies on it. From the next
   train day on, every cut runs the procedure.
4. **The fold.** `MigrationRunner`'s refusal, `scripts/fold_store.py`, and `tests/test_store_baseline.py` with its
   postgres service land before the first train of the next calendar month, whose fold is the first. That first fold is
   the largest there will be — every revision both trees hold today, up to the floor — and a store at or past the floor
   sees nothing change, so the hosted hub and the runner host upgrade across it by the ordinary pass.

The compatibility check that holds a release branch to every release already cut on its line is
`epic:release-engineering/compat-floor`'s and joins the release-branch pull request when that epic lands, as its window
file does the open-for-a-major check; until then a patch is proven by the gate and the upper tiers alone, and the
provenance check and the standards rule are the only things holding a patch to what it may carry.

## What `epic:cadence` takes over

Until `epic:cadence` lands, an operator or an agent they ask runs the train on its day and the fold on its month,
following the procedures. Each procedure is written as a routine could carry it — every input read from the repositories
and their runs, never from memory, and every refusal stated as a condition to check — so `epic:cadence` schedules the
train and the fold as routines of the fleet's own without either procedure changing. Patches and majors stay operations
a person asks for.
