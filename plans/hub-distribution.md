# Plan — `epic:hub`, distribution slice

Today, running a hub means being the person who built it: fetch a wheel from a release, arrange a Python environment,
copy a systemd unit out of the repo. This slice turns the hub into something a stranger can run — one public container
image, pulled from a registry, standing and serving its board within minutes of `docker compose up` — and gives the
project the release discipline that promise demands: versions that mean something, notes that say what changed, and a
clear word in advance when an upgrade will hurt. Its first customer is not a stranger at all: the off-machine deployment
of the [remote slice](./hub-remote.md) runs this image, which is how the slice earns its proof before anyone else
arrives.

## What already holds

The hub was built to be packaged. It is one process behind one port; the board ships inside the wheel, so there is no
second tier to serve; health and readiness are separate endpoints, so a supervisor can tell "starting" from "broken";
the store is sqlite or postgres by configured URL; and every schema revision carries a working downgrade, which is rarer
than it sounds — most self-hosted tools answer "roll back" with "restore from backup." Secrets already arrive by
environment-variable indirection, never by config value, which is exactly the posture a container wants.

The runner is deliberately absent from this plan. A runner lives on a developer's machine, among that machine's
worktrees, toolchains, and coding-harness credentials — the things a container exists to exclude. It installs as a
package where a development environment already is; putting the *workers* in containers is the worker-lockdown slice of
`epic:security`, a different promise (isolation) than the one made here (distribution).

## What to build

- **A public image.** `ghcr.io/<owner>/blizzard-hub`, built for amd64 and arm64, running as a non-root user, carrying
  `git` for delivery workdirs and the postgres driver so the store choice is truly configuration. Its entrypoint runs
  migrate-then-serve as ordered steps — the daemon itself still never migrates — and every piece of durable state (data
  directory, signing keys, hub workdirs, config) lives on a documented volume mount. GHCR is the registry on purpose:
  free and unlimited for public images, authenticated by the repository's own workflow token, and living beside the code
  and the releases rather than on a second platform with its own pull-rate weather.
- **Configuration through the environment.** The hub's config file keeps its role, but the values a deployment varies —
  the database URL first among them — become environment overrides, so a container is configured the way containers are
  configured: by its runtime, not by baking a file into an image.
- **The publish pipeline.** Every release tag builds and pushes the image alongside the wheel, fanning the version out
  the way operators expect to pin it: the exact version, its minor line, and `latest`. One tag produces one release
  carrying one wheel, one image, and one set of notes.
- **Versioning taken seriously.** Semantic versions, with a written policy on what *breaking* means for this product —
  not just the HTTP API, but config keys, volume layout, a migration that cannot walk back, and above all the hub↔runner
  protocol, because a hub that updates in the cloud leaves runners behind on home machines. The policy states the
  supported skew between a hub and its runners, so an operator knows how far behind a runner may safely fall.
- **Release notes.** The commit history already speaks Conventional Commits, so the notes' skeleton is generated; the
  judgment stays human — an upgrade-notes section, written by hand, leads any release that asks something of the
  operator, and breaking changes are named there before they are discovered in a terminal.
- **A reference deployment.** One compose file — hub, postgres, and a TLS-terminating reverse proxy — that is both the
  adopter's quickstart and the deployment this project itself runs. This settles the remote slice's open questions: the
  container image is the blessed shape (the systemd unit remains, for the machine that prefers it), and TLS terminates
  at a documented reverse-proxy front rather than in the hub.
- **The operator's book.** Install, upgrade, roll back, and back up — four short documents that treat the operator's
  worst evening as the design case. The upgrade contract is stated plainly: updates are restart-based, and the fleet's
  runners ride out the restart by design.

## Fast follows

Named here so the epic holds them, filed only when their turn comes: a smoke test that boots the published image in CI
and asks it for health and readiness on both architectures; supply-chain hygiene — a pinned base image, vulnerability
scanning, signed or attested artifacts — which becomes table stakes the day strangers are told to run this; the
update-available badge on the board, the first visible sprout of a self-update story whose acting half (the supervisor
pattern) remains its own future slice; the postgres proof — the test suite run against a live postgres before the
documentation claims the support the code was shaped for; and a one-sentence support statement, because "latest release,
upgrade forward" said honestly beats an enterprise policy nobody staffs.

## What this slice is not

It does not make the hub update itself — it makes the hub *updatable*, cleanly and legibly, by whatever hand or timer
the operator trusts. And it does not distribute the runner, for the reasons above: the hub is the thing strangers run on
boxes; the runner is the thing developers install at home.
