# Carry-over

The epic changes how a release is built and published, never what blizzard does once installed, so nothing an
installation holds — its store, its configuration, its tokens — moves. What carries over is the release workflow, the
dev channel, the instance's deployment, and the releases cut before it.

## The order of landings

Each step is a change of its own and leaves `master` releasable:

1. **The hashed requirements.** `scripts/export-requirements.sh`, the Dockerfile's hashed install, and
   `scripts/build-wheel.sh`'s proof installing the same way, on both the release and the dev channel. It changes no
   published artifact's shape, only where the image's dependencies come from, and `blizzard:image-smoke` proves it
   before anything is published.
2. **The release workflow.** The three jobs, the signature, the attestations, the bills of materials, the image file,
   the requirements files, and the checksums ([artifacts.md](./artifacts.md#the-release-workflows-jobs)), with
   `tests/test_release_workflow.py` updated in the same change. It lands after the settings below exist, so the first
   release it publishes is already protected by them.
3. **The verify script and job**, and `bzh:release`'s confirmation in `blizzard-context`. The first release cut after
   this step is the first release a stranger can verify end to end.
4. **The `edge` channel's signatures**, in `push.yml`.
5. **The operator documents**, including `docs/verifying.md` ([install.md](./install.md#the-operator-documents)).
6. **The instance.** `blizzard-infra`'s updater verifying before it recreates, and the workspace's runbook installing
   the host through a verified requirements file. This waits until the instance is pinned to a release cut after step 3,
   by `epic:release-engineering/first-release`'s procedure; an updater that verifies would refuse every release before
   it.

The epic does not wait for `epic:release-engineering/first-release` and does not hold it back. Whichever cut follows
step 3 is the first verifiable release; if that is the first real release, the instance starts on a signed one.

## Settings made once, by hand

No workflow can create these, and the first cut after step 2 is what proves them. They are made before step 2 lands,
each recorded in `docs/ci.md` where package visibility already is
([attestation.md](./attestation.md#settings-no-workflow-can-assert)):

- The tag ruleset on `v*`.
- Immutable releases, turned on for the repository.

## Releases cut before

`v0.1.0-rc.1`, and any release cut before step 3, stay as they were published: unsigned, unattested, and with no hashed
requirements file to install from. They are never signed after the fact, since a signature minted today would attest a
build this workflow did not make. `docs/verifying.md` names the first verifiable release, and
`scripts/verify-release.sh` refuses an earlier tag with a message saying so rather than failing on its first missing
file.

The dev wheel, the `edge` image's tags, and the image's tag fan-out are unchanged. Nothing outside the workspace follows
the dev wheel, and the instance stops following `edge` under `epic:release-engineering/first-release`.
