# Hub pin contract

Paths are relative to the `blizzard-infra` repository.

## The pin

A new committed file, `deploy/release.env`, holds the hosted hub's release, and nothing else:

```text
BLIZZARD_HUB_IMAGE=ghcr.io/paul-gross/blizzard-hub:0.2.0-rc.1@sha256:<64 hex>
```

The tag is the release's exact version as `scripts/image-tags.sh` publishes it; the digest is the multi-arch index
digest that tag resolved to when the pin was written. Docker pulls by the digest and ignores the tag, so the tag is
there for the reader and the digest is the identity: a tag moved in the registry can never move the host.
`tests/test_deploy_contract.py` holds the file to that shape — exactly one key, the `ghcr.io/paul-gross/blizzard-hub`
repository, a version matching `^[0-9]+\.[0-9]+\.[0-9]+(-rc\.[0-9]+)?$`, and a `sha256` digest — and refuses `edge`,
`latest`, an `X.Y` line, a `sha-` tag, or a missing digest. The file is shipped with the rest of `deploy/`
(`scripts/deploy.sh`) and synced by `deploy/bin/blizzard-install` like every other artifact, so it changes on the host
only when a reviewed change to it is deployed.

## Where the image is read

Every compose invocation on the host reads two env files, the pin first and the operator's `.env` second:
`--env-file release.env --env-file .env`. A later file overrides an earlier one, so `.env` keeps exactly the meaning its
`BLIZZARD_HUB_IMAGE` line has today — the operator's word — but its default changes. `deploy/env.example` drops the
`BLIZZARD_HUB_IMAGE=…:edge` default and documents the key as **the hold**: absent, the host runs the pin; present, the
host runs whatever it names and the updater applies nothing. `blizzard-install` still never writes `.env`
(`blizzard-infra:install-idempotence` keeps asserting it byte for byte), and a fresh install materializes a `.env`
without the key. The `compose()` helper in `deploy/bin/blizzard-update`, and every compose command in
`docs/deployment.md` and `docs/rollback.md`, gain the pin's `--env-file`.

## The updater applies the pin

`deploy/bin/blizzard-update` stops tracking a channel and converges the hub onto the pin. `EDGE_REF` and its test-only
override `BLIZZARD_UPDATE_EDGE_REF` go; `BLIZZARD_UPDATE_COMPOSE_OVERRIDE_FILE` and `BLIZZARD_UPDATE_READY_URL` stay for
the smoke harness. Its algorithm becomes:

0. Take the lock, as today.
1. **Hold.** If `.env` sets `BLIZZARD_HUB_IMAGE`, log `held at <ref>; release pin <ref> not applied` and exit 0. The log
   names both refs on every run, so a hold left behind after an emergency is never silent.
2. **Converged.** Read the running hub's digest (`running_digest`, as today). Equal to the pin's digest: up to date,
   exit 0. The prune runs before this check, so a converged host still clears superseded pulls.
3. **Fenced.** The pin's digest is in `state/blocked-digest`: no-op, as today.
4. **Pull** the pin by its digest, recording it in the `state/pulled-images` ledger; the prune keeps the running image
   and the pinned one. Once `epic:release-engineering/supply-chain` lands, its signature verification of the pulled
   digest sits here, after the pull and before the direction step, and fences an image that fails it
   ([verification](../../supply-chain/spec/verification.md#the-instance)).
5. **Direction.** Read the head revision the pinned image expects
   (`docker run --rm --entrypoint blizzard-hub <pin> migrate --head`, [versions.md](./versions.md) §Revision queries
   every release answers) and the store's revision from the running hub's `/api/ready` (`store_revision`). The pinned
   image `--knows` the store's revision: **forward**. Otherwise the running image `--knows` the pinned head, since only
   the newer image holds both: **back**. Neither — the two share no line, or an image cannot answer: fence the digest
   with that reason and exit 1, for a person.
6. **Record rollback state**, as today: `previous-digest`, a stopped, application-consistent copy of `hub.db` and its
   WAL sidecar, the last three kept.
7. **Back only: downgrade on the outgoing image.** With the hub stopped, run
   `blizzard-hub migrate --dir /var/lib/blizzard/hub --down <pinned head>` through `compose run --rm` on the running
   image, the step `blizzard`'s `docs/rollback.md` §3 prescribes by hand. A failed downgrade restores the copy taken in
   step 6, restarts the outgoing image, fences the pinned digest with the downgrade's output in `state/last-failure`,
   and exits 1.
8. **Recreate** the hub only (`--no-deps`), on the pinned image. A forward move migrates on boot as today.
9. **Readiness**, as today: the budget, the container checks on every poll, through Caddy.
10. **Success** records `current-digest` and a new `state/current-release` holding the pin's version; **failure** fences
    the digest, records the reason and the log tail, and exits 1. After a failed forward move the store may be ahead of
    the image it stopped; that case stays the operator's (`docs/rollback.md`), exactly as a failed `edge` update is
    today.

The timer stays at five minutes. With a pin that only moves when it is deployed, nearly every run is the step 2 no-op;
the timer exists to converge after a reboot, a failed pull, or a cleared fence. `blizzard-install`'s last step runs the
updater once, so `scripts/deploy.sh` applies a moved pin in the same command and prints the updater's log with the rest
of the install's output.

## The hold

The hold replaces the pin as the emergency brake: an operator who must take the hub off a release before a reviewed
change can land writes `BLIZZARD_HUB_IMAGE=<ref>` into `.env` over SSM, as the rollback document's step 2 does today,
and recreates the hub by hand. The documented way back is a reviewed change to `deploy/release.env` naming what the host
should run, followed by removing the hold. `docs/rollback.md` is rewritten around this: the reviewed pin change is the
rollback, the updater performs it, and the hand procedure remains for when the registry or the updater cannot be relied
on — with its schema-delta check unchanged. §The swap is the pin and §Returning to the channel are replaced by the hold
and its removal; nothing in the deployment tracks a channel any more.

## Tests

- **`blizzard-infra:static-contract`** (`tests/test_deploy_contract.py`): the pin's shape above; `env.example` sets no
  default `BLIZZARD_HUB_IMAGE`; every compose command the updater issues reads both env files, pin first;
  `test_default_hub_image_is_the_edge_channel_ref` is replaced by a test that no deployment artifact names `:edge`.
- **`blizzard-infra:updater-smoke`** (`tests/local/updater-smoke.sh`): the local `registry:2` is seeded only from
  immutable refs rather than the moving `edge`: the pinned `0.2.0-rc.1`, and a fixture image built `FROM` it under
  `tests/local/fixtures/` that adds one revision above its head, read with `migrate --head` at build time — the second
  image a schema delta needs while only one release exists, and the one whose revision can be made to fail its
  downgrade. It proves a forward move, a back move across that revision with the downgrade on the outgoing image, a
  failed downgrade restored and fenced, the hold logged and honoured, and the existing fence, pin, and prune cases.
  Seeded from fixed refs, it is a function of the change under test, and the reason it runs on push only
  (`context/verification.md` §CI) no longer holds: it joins the PR gate.
- **`blizzard-infra:install-idempotence`**: a second install with an unchanged pin recreates nothing, and an install
  carrying a moved pin runs the updater once.
