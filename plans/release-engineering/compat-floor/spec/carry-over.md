# Carry-over

The epic lands on a repository that has cut one candidate, `v0.1.0-rc.1`, and checks every change against the commit
before it. It leaves one that checks every change against what its releases shipped, without a window in which nothing
is checked.

## The order of landings

The epic follows `epic:release-engineering/first-release`, because a floor needs a release to stand on, but its first
landing does not wait for it.

1. **The documents and the window file.** `contracts/config/*.json`, `contracts/store/*.json`, the config key registries
   with `Table`'s strict mode, `blizzard-export-contracts` and its drift step, and `src/blizzard/wire/window.toml` with
   `floor` set to the line the first release will cut. Nothing reads them yet, and the documents are additive to every
   surface. Landed before the first release is cut, they make it a baseline for every surface; landed after, it is a
   baseline for the OpenAPI and dictionary surfaces only until the next release.
2. **The handshake.** The optional `version` fields, `version_window.py` with `handshake_since` set to the line it ships
   in, the refusal, the registration row's recorded version, the claim gate, the runner's halt, the board's column, and
   the mock hub's mirror. It is additive on the wire and admits every runner the fleet runs today: their floor is below
   `handshake_since`, so an unversioned runner is admitted.
3. **The check, and the old modes retired, in one change.** `blizzard-compat`, the gate job on every trigger, the
   release workflow's backstop, `compat-breaks.json`, and the removal of `blizzard-wire-compat`, `wire-compat-deployed`,
   and `dev-image`'s wait on it. Retiring the old modes in the same change as the new one starts means the repository is
   never without a check; landing it only once a release at or above `floor` exists means the new check is never empty
   on its first day. `blizzard-context`'s rules, method ids, and the procedure for judging a break land beside it.
4. **The rehearsal.** `rehearsal.yml`, the `tests/rehearsal/` journey and store walk, and their calls from `push.yml`
   and `release.yml`, once a supported release with a published wheel exists.

Between the first release and step 3, the per-commit checks keep running as they are. They are stricter than the new
check about surface no release carried, and they still catch every unacknowledged break, so the interval is safe if
occasionally inconvenient.

## What predates the floor

`v0.1.0-rc.1` sits below every floor the first release sets, so no check compares against it and no runner of it is
admitted once the floor passes `handshake_since`. The boundaries `docs/upgrade.md` records — the subscription-usage and
runner-identity exceptions among them — all precede the first release, and the version
`epic:release-engineering/first-release` gives each of them places it below the floor; neither the checks nor the
handshake reopen them. A store older than the first release is the first release's to carry, by its own upgrade
procedure, not the rehearsal's.

## A fleet whose runners predate the handshake

The dogfood instance's runner host moves onto the first release under `epic:release-engineering/first-release`. If that
release predates the handshake, its runners register without a version and are admitted, shown as `unversioned` on the
board. They start sending one the first time the host is moved onto a release carrying step 2. The first change that
raises `floor` to `handshake_since` or beyond is when an unversioned runner is refused, and by then every runner inside
the window sends a version, so the refusal only ever reaches a runner the window has already left behind.

## Blizzard-mock before mirrored tags

The rehearsal checks out `blizzard-mock` at the baseline's tag. Until `epic:release-engineering/release-lines` cuts the
mock's tags beside blizzard's, no baseline has one, and the rehearsal runs the mock's `master` against the old runner.
That holds while the mock only adds to its facades, which `bzh:wire-change-extends-mock` already asks of it; a mock
change that breaks an older runner's adapter shows as a rehearsal failure on `master`, which is where it should show.
