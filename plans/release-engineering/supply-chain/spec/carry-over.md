# Carry-over

The epic changes how a release is built and published, never what blizzard does once installed, so nothing an
installation holds — its store, its configuration, its tokens — moves. What carries over is the release workflow, the
dev channel, the instance's deployment, and the releases cut before it.

## The order of landings

Each change leaves `master` releasable. Steps 1 to 6 and step 9 are ordinary landings; the rest are attended operations
no fleet worker can perform, since they change repository settings, cut a tag, deploy `blizzard-infra`, or edit the
workspace repository.

1. **The hashed requirements.** `scripts/export-requirements.sh`, the Dockerfile's hashed install, and
   `scripts/build-wheel.sh`'s proof installing the same way, on both the release and the dev channel. It changes no
   published artifact's shape, only where the image's dependencies come from, and `blizzard:image-smoke` proves it
   before anything is published. The hosted hub takes it with the next `edge` image.
2. **The settings, by hand.** The two repository settings below, made before the release workflow changes, so the first
   release it publishes is already protected by them.
3. **The release workflow's signature, provenance, and inventory.** The `build` and `publish` jobs, the image's
   signature, the provenance on the image and every attached file, the requirements files, the image file, and the
   checksums ([artifacts.md](./artifacts.md#the-release-workflows-jobs)), with `tests/test_release_workflow.py` updated
   in the same change and the settings recorded in `docs/ci.md`.
4. **The bills of materials** ([attestation.md](./attestation.md#bills-of-materials)), added to the `build` job and the
   inventory.
5. **The verify script and job** ([verification.md](./verification.md)). With them, by hand, the operator writes the
   confirmation into `workspace:/context/project/workflows/release.md` ([verification.md](./verification.md) §The
   procedure), when the instance moves onto a build carrying them and in any case before step 7 confirms through it: no
   fleet chunk's environment holds the workspace.
6. **The `edge` channel's signatures**, in `push.yml`, and **the operator documents**, including `docs/verifying.md`
   ([install.md](./install.md#the-operator-documents)). Neither depends on the other; the signatures depend on nothing
   above, and the documents on steps 3 to 5.
7. **The first verifiable release, by hand.** The next release cut by `workspace:/context/project/workflows/release.md`
   after step 5, by whichever of `epic:release-engineering/first-release`'s and
   `epic:release-engineering/release-lines`' procedures is in force, and confirmed through the verify job and one run of
   `scripts/verify-release.sh` from the workspace.
8. **The instance onto it, by hand.** The hub's pin and the runner host moved onto that release by
   `workspace:/context/project/workflows/release-deploy.md`, after `epic:release-engineering/first-release`'s cutovers,
   so that no pin the hub is moved to afterwards predates signing.
9. **The updater's verification**, in `blizzard-infra`, on the updater `epic:release-engineering/first-release` rewrites
   ([verification.md](./verification.md#the-instance)). It lands after step 8.
10. **Its deploy, by hand.** `blizzard-infra`'s `scripts/deploy.sh`, confirmed by the next pin move's log showing the
    digest verified before the hub was recreated.
11. **The runner host's runbook, by hand.** The workspace's `context/project/local-instance.md` installs the host
    through a verified requirements file ([verification.md](./verification.md#the-instance)), after step 7 and after
    `epic:release-engineering/first-release`'s runner-host cutover.

The epic does not wait for `epic:release-engineering/first-release` and does not hold it back. Whichever cut follows
step 5 is the first verifiable release; if that is the first real release, the instance starts on a signed one.

`compat-breaks.json` joins the inventory with whichever lands later, step 3 or `epic:release-engineering/compat-floor`'s
release-workflow backstop, and the verify script requires it from the first release that carries both
([verification.md](./verification.md#the-verify-script)).

## Settings made once, by hand

No workflow can create these, and the first cut after step 3 is what proves them. Each is recorded in `docs/ci.md` where
package visibility already is ([attestation.md](./attestation.md#settings-no-workflow-can-assert)):

- A tag ruleset on `v*` that lets only repository admins create a matching tag and blocks every update and deletion.
- Immutable releases, turned on for the repository.

## Releases cut before

`v0.1.0-rc.1`, and any release cut before step 5, stay as they were published; those cut before step 3 are unsigned,
unattested, and carry no hashed requirements file to install from. They are never signed after the fact, since a
signature minted today would attest a build this workflow did not make. `docs/verifying.md` names the first verifiable
release, and the verify script refuses an earlier tag ([verification.md](./verification.md#the-verify-script)).

A release branch cut before step 3 keeps the `release.yml` it was cut with, and a patch tagged from it publishes the way
that workflow does — unsigned, without the inventory. The signing and attestation steps are not carried back onto such a
branch; a line that must be verifiable is one branched after step 3. Before 1.0 only the newest minor is patched
(`epic:release-engineering/release-lines`), so the first minor cut after step 3 ends the case.

The dev wheel, the `edge` image's tags, and the image's tag fan-out are unchanged. Nothing outside the workspace follows
the dev wheel, and the instance stops following `edge` under `epic:release-engineering/first-release`.
