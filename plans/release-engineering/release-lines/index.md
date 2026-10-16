---
epic: release-engineering/release-lines
refinement: refined
status: horizon
---

# Plan — `epic:release-engineering/release-lines`

Once the first real release is cut, blizzard has a version an operator can stand on, and nothing yet that tells them
when the next one comes or what happens when the one they stand on turns out to be wrong. A release made whenever
someone remembers to make one is a release nobody can plan around. And a fix for a release that has already shipped has
only one road to an operator today: wait for the next cut from `master`, and take everything else `master` has learned
since, wanted or not. The operator who pinned last month's release to keep their fleet quiet is told, in effect, that a
security fix costs them a month of unrelated change.

Blizzard's own fleet feels the other half of it. Running every candidate first is only worth something if a candidate
arrives on a rhythm, stays long enough to be lived with, and then becomes the release everyone else takes — or is set
aside because the week found something wrong with it.

This epic gives releases a rhythm and gives every released line a way to be repaired. A minor leaves on a weekly train:
cut as a candidate, run by blizzard's own fleet for the week, and promoted to stable as the next candidate is cut. Every
release gets a branch at its candidate, so a fix reaches a released line as a patch that carries that fix and nothing
else. The promise of which lines are patched is short and written down. A major, once blizzard is past 1.0, arrives on a
date published a cycle ahead, from a `master` opened for it in the weeks before. And because a release now says exactly
which schema histories nobody inside the window will ever replay again, the store's migration history folds into a
single baseline on the same rhythm, so a fresh install stops replaying every schema decision the project ever made.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What to build

- **A weekly train.** Once a week a candidate for the next minor is cut from a proven `master` and published, and
  blizzard's own fleet moves onto it through the pin that `epic:release-engineering/first-release` opens. When the next
  candidate is cut, the one before it is promoted to stable: the same commit, published under its stable version, with
  nothing rebuilt from new code. A week with nothing landed runs no train. A candidate the week found wrong is either
  repaired on its branch as a further candidate, or set aside, its minor never becoming stable, and the next candidate
  carries on. The calendar sets the release; no train waits for an epic.
- **A branch for every release.** Each minor gets its branch at its first candidate, and every later candidate, the
  stable release, and every patch of that minor are tagged from it. `master` moves on to the next minor the moment the
  branch is cut, so a released line is never asked to absorb what `master` learns afterwards.
- **A way to patch.** A fix lands on `master` first and is carried back to each supported line that carries the bug by
  cherry-pick, proven on that branch by the same gate `master` passes, held to every release already cut on that line,
  then versioned and tagged from the branch. A fix for code `master` has already removed is written on the old line
  alone. Most fixes ride the next train instead; a patch is for what cannot wait the week — a hole in security, lost
  data, a fleet that has stopped.
- **A short promise, written down.** Within a major only the newest minor is patched, since every minor only adds and
  moving to the newest is always a safe fix. Once a new major ships, the previous major's final line receives critical
  fixes, and only those, for three months, and its runners keep working against the new major's hub for the whole of
  that major. Before 1.0 only the newest minor is patched. The promise is published beside the version scheme, where an
  operator choosing what to pin will read it.
- **The mock released beside blizzard.** A release is proven by `blizzard` and `blizzard-mock` and nothing else, so the
  mock gets the same branch, cut from the commit that proved the candidate, and the same tag on every release, whether
  or not the mock changed. Checking out one tag in both repositories reproduces exactly what that release was proven
  against, and every proof of a release branch or a tag runs against the mock's matching ref.
- **Majors on their dates.** Past 1.0, two majors a year arrive on dates published a cycle ahead. In the weeks before
  one, `master` opens for its removals and its candidates carry the new major's number, while the outgoing major's final
  line continues from its branch. The major's last candidate is cut a train before its date, closing `master`, and
  promoted on the date after a week on the fleet.
- **The store's history folded at the floor.** At each major — and, before 1.0, on the first train of each month — every
  migration revision up to the last one the oldest supported release carries is folded into a single baseline that
  recreates the schema in one step and keeps that revision's identifier. A store already at or past it sees nothing
  change; only a store older than every supported release is turned away, with a message naming the release to step
  through first. A store built from the baseline is proven to match one built by replaying the history it replaced, on
  every database blizzard supports.
- **Every operation a written procedure.** Cutting the weekly candidate and promoting it, patching a released line,
  opening `master` for a major and cutting it, and folding the store's history are each a procedure in
  `blizzard-context`, so an operator says *patch 0.7 with this fix* and an agent carries it to a confirmed release
  without improvising a step.

## Left for later

Running the train and the fold on a schedule of the fleet's own belongs to `epic:cadence`: until it lands, an operator
or an agent they ask carries each procedure out by hand on its day. A longer promise for security fixes, an LTS line,
waits until operators outside blizzard ask for one. What a release's compatibility checks compare against is
`epic:release-engineering/compat-floor`'s, what its notes say is `epic:release-engineering/notes`'s, and how unfinished
work shares `master` with a weekly cut is `epic:release-engineering/dark-launch`'s; this epic decides when releases are
cut and from where, and relies on each of them for what a cut must prove and carry.
