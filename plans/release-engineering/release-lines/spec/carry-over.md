# Carry-over contract

## Where it starts

`blizzard` has one tag, `v0.1.0-rc.1`, cut on `master`, and no release branch; `blizzard-mock` has neither tags nor
release branches, and its `pr.yml` and `push.yml` run on `master` alone. `epic:release-engineering/first-release` lands
before this epic's first train: it cuts the first real release, moves the dogfood instance onto pinned releases, and
opens a pin bump whenever a release publishes, which is what carries blizzard's own fleet from one candidate to the
next. This epic adds the rhythm on top of that cut; it changes nothing about how a single tag is cut and confirmed
(`workspace:/context/project/workflows/release.md`), which stays the step every procedure here ends in.

## The order of landings

Landings are fleet work; pushing a branch or a tag, changing a repository's rulesets or labels, and running a train or a
fold are attended steps, which no fleet worker performs. Each attended step sits in the order where the landings around
it need it.

1. **What the first cut needs**, landed before `epic:release-engineering/first-release` cuts it: `check-version-tag.sh`
   comparing the tag with its `-rc.K` suffix removed, `pyproject.toml` naming the last cut and moving only in a cut
   commit ([train.md](./train.md) §Versions on `master` and on a branch, §The cut commit), and `push.yml`'s dev version
   naming the next minor. Each is covered beside today's tests (`tests/test_check_version_tag.py`,
   `tests/test_release_workflow.py`, `tests/test_push_workflow.py`). The first cut is then a cut commit on `master` and
   a candidate tag there, with no branch yet.
2. **The refusal of an older store** ([fold.md](./fold.md) §Turning an older store away), at any point: no tree declares
   `ABSORBED` until the first fold, so it is inert when it lands.
3. **Release branches proven**, once the first candidate exists, since the line it proves is that candidate's: `pr.yml`
   on `release/**` in both repositories, the mock-ref composite action and the `proven-against` artifact
   ([mock.md](./mock.md)), the provenance check ([patches.md](./patches.md) §Carrying a fix back), and `docs/ci.md`'s
   account of them. None of it changes what a `master` pull request or push proves.
4. **Attended: the first line.** Straight after step 3, before any further tag is pushed, since every tag run from then
   on checks out the mock's tag of the same name: `release/X.Y` at the first candidate's commit in `blizzard`, and in
   `blizzard-mock` at the commit that candidate's tag run checked out, read from that run's log since nothing recorded
   it then, with a mock tag of the same name for every blizzard tag already on the line. Beside it, the rulesets
   protecting `release/*` and `v*` on both repositories and the `release-blocker` label on `paul-gross/blizzard`.
5. **The tag rules**, once the first line exists, since the reachability rule would otherwise refuse its next tag: the
   reachability and `opening` rules of `check-version-tag.sh` and the dev version while `master` is open
   ([train.md](./train.md) §Versions on `master` and on a branch, [majors.md](./majors.md) §Candidates of the major),
   which read `epic:release-engineering/compat-floor`'s window file and need it landed; the stable tag's borrowed proof
   ([train.md](./train.md) §Promotion); and `image-tags.sh`'s `latest` rule ([patches.md](./patches.md) §Publishing).
6. **The written promise and the procedures**, once steps 3 and 5 exist to be written against, and after
   `epic:release-engineering/first-release` has rewritten `docs/versioning.md` and
   `workspace:/context/project/workflows/release.md`: `docs/versioning.md` gains §The train, §Support, and §Schedule,
   and `blizzard-context`'s `standards/` gains `bzh:patch-scope`. With it, by hand, the operator writes
   `workspace:/context/project/workflows/release-train.md`, `release-patch.md`, and `release-major.md`, each ending in
   `release.md`, when the instance moves onto a build carrying the landing: no fleet chunk's environment holds the
   workspace. From the next train day on, an operator or an agent they ask runs the train by `release-train.md` — an
   attended step every week until `epic:cadence` schedules it.
7. **The fold's tool and proof**, once `epic:release-engineering/compat-floor` publishes its store documents and the
   supported releases (`blizzard-compat releases --json`) and a release with a published wheel exists:
   `scripts/fold_store.py`, `tests/test_store_baseline.py`, and its `store-baseline` job, with
   `workspace:/context/project/workflows/store-fold.md` written beside them by hand ([fold.md](./fold.md) §The
   procedure). It folds nothing.
8. **Attended: the first fold**, by `workspace:/context/project/workflows/store-fold.md`, once compat-floor's rehearsal
   has landed to walk it, merged before the first train of the next calendar month cuts its candidate. It is the largest
   fold there will be — every revision both trees hold today, up to the floor — and a store at or past the floor sees
   nothing change, so the hosted hub and the runner host take it with the release that carries it.

The compatibility check that holds a release branch to every release already cut on its line is
`epic:release-engineering/compat-floor`'s and joins the release-branch pull request when that epic lands; until then a
patch is proven by the gate and the upper tiers alone, and the provenance check and `bzh:patch-scope` are the only
things holding a patch to what it may carry.

## What `epic:cadence` takes over

Until `epic:cadence` lands, an operator or an agent they ask runs the train on its day and the fold on its month,
following the procedures. Each procedure is written as a routine could carry it — every input read from the repositories
and their runs, never from memory, and every refusal stated as a condition to check — so `epic:cadence` schedules the
train and the fold as routines of the fleet's own without either procedure changing. Patches and majors stay operations
a person asks for.
