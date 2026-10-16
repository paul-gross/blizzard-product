# Carry-over

The epic lands on a repository that has cut one candidate, `v0.1.0-rc.1`, and checks every change against the commit
before it. It leaves one that checks every change against what its releases shipped, without a window in which nothing
is checked.

## The order of landings

The epic straddles `epic:release-engineering/first-release`'s cut of `0.2.0`. What must be in the first release lands
before the cut; what needs a release to exist, or a `master` whose own version has reached `floor`, lands after it.

Before the cut:

1. **The store documents and the window file.** `contracts/store/*.json`, `blizzard-export-contracts` and its drift
   step, the SQLite reflection test, and `src/blizzard/wire/window.toml` with `floor = "0.2"` and its parser. Nothing
   reads them at runtime.
2. **The configuration documents.** The two key registries, both daemons' strict accessor, the hub's reads moved onto
   it, and `contracts/config/*.json`, through the same exporter and drift step.
3. **The tool.** `blizzard-compat` with every classifier, `releases --json`, and the pre-1.0 allowance, unit-tested on
   throwaway repositories, runnable locally, and called by nothing in CI yet; the per-commit check keeps gating.

Steps 1 and 2 must both land before the cut, which waits for them: landed after, `0.2.0` would be a baseline for the
OpenAPI and dictionary surfaces only, until the next release.

After the cut:

4. **The handshake.** The optional `version` fields, `version_window.py` with `handshake_since = "0.3"`, the line it
   ships in, the refusal, the registration row's recorded version, the claim gate, the hub's start guard, the runner's
   halt, the board's column, and the mock hub's mirror. It waits for the cut commit, not merely for the tag: before it,
   `master`'s `pyproject.toml` names `0.1.0`, and every hub built from it would sit below its own `floor` and refuse
   every runner — which is also why a hub whose own line is below `floor` refuses to start rather than run
   ([handshake.md](./handshake.md) §The rule). After it, every hub and runner on the instance reports line `0.2` or
   reports nothing, and both are admitted.
5. **The gate, and the old modes retired, in one change.** The gate job on every trigger, the release workflow's
   backstop, `compat-breaks.json`, the declared-break allowance on an open `master`, and the removal of
   `blizzard-wire-compat`, `wire-compat-deployed`, and `dev-image`'s wait on it. Retiring the old modes in the change
   that starts the new one means the repository is never without a check; landing it once `v0.2.0-rc.1` exists means the
   new check is never empty on its first day. `blizzard-context`'s rules, method ids, and the procedure for judging a
   break land beside it.
6. **The rehearsal.** `rehearsal.yml` with its Postgres service, the `tests/rehearsal/` journey and store walk, their
   calls from `push.yml` and `release.yml`, and `dev-image`'s wait on it, once `v0.2.0-rc.1` is published with its
   wheel.

Between the cut and step 5, the per-commit checks keep running as they are. They are stricter than the new check about
surface no release carried, and they still catch every unacknowledged break, so the interval is safe if occasionally
inconvenient.

The handshake reaches the instance with the first release that carries it, by `bzh:release-deploy`, hub first and the
runner host after. That procedure is the whole of its deployment; the epic has no attended step of its own.

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
board. They start sending one the first time the host is moved onto a release carrying step 4. The first change that
raises `floor` to `handshake_since` or beyond is when an unversioned runner is refused, and by then every runner inside
the window sends a version, so the refusal only ever reaches a runner the window has already left behind.

## Blizzard-mock before mirrored tags

The rehearsal checks out `blizzard-mock` at the baseline's tag. Until `epic:release-engineering/release-lines` cuts the
mock's tags beside blizzard's, no baseline has one, and the rehearsal runs the mock's `master` against the old runner.
That holds while the mock only adds to its facades, which `bzh:wire-change-extends-mock` already asks of it; a mock
change that breaks an older runner's adapter shows as a rehearsal failure on `master`, which is where it should show.
