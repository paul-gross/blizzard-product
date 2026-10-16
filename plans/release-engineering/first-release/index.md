---
epic: release-engineering/first-release
refinement: refined
status: horizon
---

# Plan — `epic:release-engineering/first-release`

The person who feels this first is the one who lands a change. The hosted hub swaps itself onto every commit `master`
proves, within five minutes, and the runner host on the desk beside it is rebuilt from `master` after every landing. So
every landing is a deployment, and a change that takes a week to finish in pieces has to leave a running fleet whole at
every one of those pieces. When one cannot, someone writes an exception into the skew window, orders the upgrade by
hand, and hopes the next landing does not need another. The version the hub reports, `0.1.0.dev<run>`, names a workflow
run rather than anything an operator could plan around, and the only tag blizzard has ever cut is a release candidate
from the summer that nothing runs.

This epic draws the line in the sand. It cuts blizzard's first real release, `0.2.0`, through a candidate that the
dogfood instance runs first, and it moves that instance off `master` for good. The hosted hub and the runner host are
each pinned to an exact release, and a pin moves only by a change someone reviewed. When a release publishes, the change
that would move the hub onto it opens by itself, carrying what the release asks of whoever takes it. Moving onto a
release, or back off a bad one, becomes one procedure an agent can carry out by name. Landing on `master` stops owing
the instance anything.

The epic changes how blizzard is deployed, not what blizzard promises. Measuring compatibility against what a release
shipped is `epic:release-engineering/compat-floor`'s work, and the weekly train is
`epic:release-engineering/release-lines`'. This epic gives both of them a release to measure against and an instance
that runs releases.

| Where                    | Read when                                                             |
| ------------------------ | --------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the epic or resolving its technical contract |

## What to build

- **The first release, `0.2.0`.** A candidate, `v0.2.0-rc.1`, is cut from a green `master` and runs the dogfood instance
  before anything else does. Once it has carried the fleet's ordinary work without a fault, the stable `v0.2.0` is cut
  at the very commit the candidate proved, so what the instance ran is what the release is. `0.2.0` rather than `0.1.0`,
  because the summer's `0.1.0` candidate described a different platform, and a stable release should never pretend to be
  what an old candidate promised.
- **Versions an operator can read.** `0.2.0` is cut the way every release after it will be: a cut commit on `master`
  sets the version and the candidate is tagged there, so a candidate and its stable release share one version, and every
  build `master` publishes afterwards names a version that sorts after the release it follows. The model is
  `epic:release-engineering/release-lines`'; this epic is its first use.
- **The skew window, restated around majors.** `docs/versioning.md` states the window the initiative draws: from 1.0, a
  runner works against any hub of its own major at the same minor or later, and a hub of a new major still accepts the
  previous major's final minor. Before 1.0, a hub accepts runners of every line from a declared floor up to its own, and
  the floor rises only in the change that removes surface a released minor deprecated, so a weekly train never shrinks
  an operator's window to a week; declaring the floor and refusing a runner below it are
  `epic:release-engineering/compat-floor`'s. A hub never requires a runner newer than itself. The window covers releases
  and nothing else: a build no release carried — `edge`, a dev wheel, a source checkout — sits outside every promise,
  and `0.2.0` is the first release inside one.
- **Every upgrade boundary given its version.** The boundaries `docs/upgrade.md` describes today were all crossed on the
  way to `0.2.0`, so they gather under that release, ordered for an operator arriving from the summer's candidate or
  from any `edge` build. The two exceptions the skew window granted move there too, since neither can bind a release
  that never carried the surface. The documents learn the image's real tag shape, which has no `v`.
- **The hosted hub pinned, and moved by review.** `blizzard-infra` holds the hub's pin as a committed file naming one
  exact release and its image digest, never a moving tag, and the on-host updater stops following `edge` and applies the
  pin instead, with the fence and the rollback state it keeps today. Moving the pin to an older release is a rollback
  the updater carries out whole, downgrading the store on the outgoing image before it swaps. An operator can still hold
  the host on another image for an emergency, and the updater says so on every run.
- **The pin change, opened for you.** When `blizzard` publishes a release, candidates included, a workflow in
  `blizzard-infra` opens the change that moves the hub's pin onto it, with the release's upgrade notes in its
  description, so reviewing the change and reading what the release asks are the same act. Merging it and shipping it to
  the host stays a deliberate step.
- **The runner host on releases.** The runner host is installed from the pinned release's own wheel, attached to its
  GitHub Release, rather than built from `master`. Its pin is recorded beside the rest of the instance's deployed
  surface, and the version the host reports — the exact release, with no commit stamp — is the proof of the redeploy.
  The redeploy is owed after the hub's pin moves, never after a landing, and the runner never moves ahead of the hub.
- **One procedure for moving onto a release, and back.** `blizzard-context` learns how to cut a candidate and promote
  it, and how to move a deployment onto a release or back off one: hub first and runners after on the way forward,
  runners first on the way back, each confirmed by the version it reports. The workspace's runbook for the instance
  keeps only what is particular to this instance and defers to that procedure for the rest.
- **`edge` keeps publishing.** Every green push to `master` still publishes `edge` and its immutable `sha-` tag for
  anyone who wants the newest proven build. Nothing of ours follows it.

## Left for later

Shipping the merged pin to the host stays a step someone takes with their own cloud credentials: carrying it there from
CI would put the first AWS credential into `blizzard-infra`'s workflows, and the instance gains little from saving one
command. The runner's pin is recorded by hand in the same procedure rather than opened as a change of its own, since the
workspace that holds it has no workflows; the hub's pin change reminds whoever merges it that the runner is owed. A
stable release is rebuilt from its candidate's commit rather than promoted from the candidate's own image; promotion's
mechanics, and whether they should reuse the very bytes, are `epic:release-engineering/release-lines`'. Compatibility
checked against releases, notes assembled from the changes themselves, and signed artifacts are their own epics.
