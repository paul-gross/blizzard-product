# Proving a release from outside

## The verify script

A new `scripts/verify-release.sh <tag>` checks a published release the way a stranger would, from any machine with `gh`,
`cosign`, and `uv`, and with no checkout of `blizzard`. It exits non-zero on the first check that fails, naming it:

1. **The image.** It reads the digest from `blizzard-hub-<version>-image.txt`, runs `cosign verify` on it with the
   release identity ([attestation.md](./attestation.md#the-identity)), and checks that the image's version tag on GHCR
   resolves to that same digest.
2. **The image's attestations.**
   `gh attestation verify oci://ghcr.io/paul-gross/blizzard-hub@<digest> --repo paul-gross/blizzard` for provenance, and
   again with `--predicate-type https://spdx.dev/Document/v2.3` for each platform's bill of materials.
3. **The attached files.** It downloads every file on the release page, runs `sha256sum -c SHA256SUMS` over them, checks
   the page holds exactly the files `artifacts.md` lists for that version — `compat-breaks.json` among them once it is
   owed (below) — and nothing else, and verifies each file's provenance and the wheel's bill of materials with
   `gh attestation verify`. It checks the requirements files' project line names the wheel on this same release, at the
   hash `SHA256SUMS` gives it.
4. **The install.** It syncs a fresh virtualenv from the base requirements file, by its address on the release page,
   with `--require-hashes`, so the wheel is fetched from the release as an operator's install fetches it, and checks
   `blizzard --version` prints the version.

The expected files are the fixed inventory plus `compat-breaks.json` for every release at or after one version, held as
a constant in the script, `COMPAT_BREAKS_SINCE`. It starts unset, so the file is not yet required, and is set to the
first release whose workflow both runs `epic:release-engineering/compat-floor`'s backstop and carries this epic's
inventory, by whichever of the two changes lands second. A tag cut before this epic's release workflow is refused with a
message naming the first verifiable release, rather than failed on its first missing file.

The script's argument handling and its expected-file list are unit-tested; the checks themselves are proven only against
a real release, which is what the `verify` job is.

## The verify job

The release workflow's last job runs the script against the tag it just published, on a fresh runner holding only
`contents: read`, so it sees the release as an anonymous outsider would, save for the API token `gh` needs to read a
public repository. A green `verify` is the release's proof that every artifact it advertises is present, signed,
attested, and installable. A red one arrives after the release exists, since it can only check what was published, and
is a red run (§The procedure).

## The procedure

`workspace:/context/project/workflows/release.md` keeps its tag-and-watch shape; its **Confirm the cut** section gains
what this epic adds rather than restating each check, written by the operator by hand ([carry-over.md](./carry-over.md),
step 5):

- Watch the run through `verify`, not only through `publish`; the verify job is the confirmation of the signatures,
  attestations, bills of materials, checksums, and the install from the release page.
- Run `scripts/verify-release.sh <tag>` once from a machine outside GitHub's runners — the workspace is enough — so the
  release is proven by a verifier that shares nothing with the run that built it.
- The anonymous `docker pull` stays: package visibility is still a setting no workflow asserts.

Its **A red run** section gains the case of a release that published but does not verify. The release stays — a tag and
an immutable release are never re-pointed or replaced — and the fix is the next tag. A release is never deleted either:
an install that pinned it fetches its files by their address on its page, and removing the page would break exactly the
operators who did what the docs asked of them.

If what fails could hand an operator a wrong or unverifiable artifact, the release is marked bad where every reader of
it looks. A maintainer prepends a warning to the release's body — immutable releases lock the tag and the attached
files, not the body — naming what is wrong and the release that replaces it once one exists, and the instance is moved
off it, or never onto it, by `epic:release-engineering/first-release`'s pin procedure. What counts as *could hand an
operator a wrong artifact* is the procedure's to state: a missing bill of materials does not, a signature that fails to
verify always does.

## The instance

Blizzard's own fleet verifies every release before it runs it.

**The hosted hub.** The updater (`deploy/bin/blizzard-update`) is the one `epic:release-engineering/first-release`
rewrites to converge onto a pinned release by digest, and verification is one step of that algorithm: after the pin is
pulled and before its direction is read — so before any downgrade, recreate, or readiness wait — the updater runs
`cosign verify` of the pulled digest against the release identity. A host already converged on its pin verifies nothing,
since nothing is about to start.

An image that fails verification is fenced exactly as one that fails readiness — `state/blocked-digest` and
`state/last-failure` written with the reason `signature`, the hub left on the image it was running, cleared only by
`--clear-block` — so an unverified image is never started. A verification that cannot run at all — Sigstore or the
registry unreachable — is not a failed verification: the run exits without fencing and without recreating, and the next
timer tick tries again, so an outage on the verifier's side delays an upgrade but never blocks a good image for good.
The updater accepts the release identity only; `edge` images are never verified as releases, and a pin to a release cut
before this epic's release workflow is fenced like any image that fails. A deliberate move back that far goes through
the hold — `BLIZZARD_HUB_IMAGE` in `.env` — which bypasses the updater as it does for any emergency, and
`docs/rollback.md` says so beside its note that every other rollback target is verified like any pin move.

`deploy/bin/blizzard-install` installs a pinned `cosign` release, checked against its published checksum.
`tests/test_deploy_contract.py` asserts the verify step sits after the pull and before the direction check, and
`tests/local/updater-smoke.sh` runs it against a stubbed `cosign` that passes, fails, or cannot reach its verifier on
demand, as the harness already stubs the commands it cannot run for real. `docs/deployment.md` describes the step.

The change lands in `blizzard-infra` like any other and deploys nothing until `scripts/deploy.sh` runs it, which is an
attended operation. It is deployed only once the instance's pin already names a release cut after this epic's verify job
exists ([carry-over.md](./carry-over.md#the-order-of-landings)), so no pin move after it can name an unsigned release by
accident.

**The runner host.** The workspace's redeploy runbook (`context/project/local-instance.md`), once
`epic:release-engineering/first-release` moves it onto releases, installs the host from the pinned release's
requirements file after verifying its attestation, as [install.md](./install.md#the-runner) describes; the requirements
file pins the release's own wheel by its hash, so it is that wheel the host runs. The workspace's venv also serves as
the operator's CLI, and syncing it removes whatever the file does not list, so the runbook confirms the venv holds
nothing else before its first sync. The runbook lives in the workspace repository, which the fleet does not land to, so
this change is made by hand.
