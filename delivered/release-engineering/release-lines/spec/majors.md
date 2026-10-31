# Majors contract

## The schedule

Past 1.0, two majors a year ship on dates published in `docs/versioning.md` §Schedule. A major's date is published no
later than the day the major before it ships, so every date is known a full cycle ahead, and it falls on that week's
train day ([train.md](./train.md) §The day). A date moves only by being republished, with the reason, and never earlier.

1.0 is not on the six-month cadence; the cadence counts from it. It is cut when `milestone:hardening` is reached,
through the same procedure with no open window (§Opening `master`): before 1.0 a breaking change already waits for the
minor after one that deprecated it (`epic:release-engineering/first-release` owns that policy), so there are no gathered
removals for a window to hold.

## Opening `master`

Four trains before a major's date, `master` opens for it. That week's train runs as usual first, cutting the outgoing
major's last minor; that minor becomes the outgoing major's **final line**, patched as [patches.md](./patches.md) §Which
lines are patched says. Then one change to `master` opens it: it sets `opening` to the new major and raises `floor` to
the final line in `src/blizzard/wire/window.toml`, the window file `epic:release-engineering/compat-floor` declares and
every check that asks whether `master` is open reads ([baseline](../../compat-floor/spec/baseline.md) §The window file).
`opening` is the one signal; nothing here infers it from `pyproject.toml`. The store's history is folded in the change
after it ([fold.md](./fold.md)), so the major ships the new baseline.

## Candidates of the major

The trains inside the open window cut no minors. Each cuts `v(X+1).0.0-rc.K` instead, at a cut commit on `master` itself
([train.md](./train.md) §The cut commit) whose `pyproject.toml` names `(X+1).0.0`, and the mock's tag sits on the commit
that proved it ([mock.md](./mock.md) §Which mock proved it). No release branch exists for the major yet, because the
window is still gathering removals on `master`. `scripts/check-version-tag.sh` accepts a `(X+1).0.0-rc.K` tag on a
commit reachable from `master` only when that commit's `window.toml` names `opening = "X+1"`. Blizzard's own fleet runs
each of them, through the pin `epic:release-engineering/first-release` opens, as it runs every candidate. The outgoing
final line meanwhile takes patches as any line does.

## Cutting the major

The major is cut at the last train before its date, so the release it ships has run on the fleet for a week like any
other. Its cut commit lands on `master` and removes `opening` from `window.toml` in the same commit, closing `master`;
`release/(X+1).0` is cut at it in both repositories and its last candidate tagged there, so `opening` is never present
on a release branch. A fix that week is a further candidate on the branch ([train.md](./train.md) §A further candidate).

On the date, the train promotes that candidate as it promotes a minor ([train.md](./train.md) §Promotion): `v(X+1).0.0`
at its commit. A candidate with an open `release-blocker` on the date is not promoted: the date slips by a week and is
republished, and the major ships from its branch once the blocker is fixed there.

The day the major ships, `docs/versioning.md` §Support writes out the date the outgoing final line's three months end,
and §Schedule publishes the date of the next major.
