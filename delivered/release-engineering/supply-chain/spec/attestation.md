# Signatures and attestations

## The identity

Everything a release signs or attests is signed keylessly through Sigstore's public-good instance, with the workflow's
GitHub OIDC token as the identity. There is no signing key, no secret to rotate, and nothing an attacker could steal to
sign as blizzard; what a signature proves is that a given workflow, at a given ref, in `paul-gross/blizzard`, produced
the artifact, and the signing event is recorded in Sigstore's transparency log where anyone can see it.

A release's identity is fixed:

| Field                | Value                                                                                       |
| -------------------- | ------------------------------------------------------------------------------------------- |
| OIDC issuer          | `https://token.actions.githubusercontent.com`                                               |
| Certificate identity | `https://github.com/paul-gross/blizzard/.github/workflows/release.yml@refs/tags/v<version>` |

A verifier matches the identity with the regular expression
`^https://github\.com/paul-gross/blizzard/\.github/workflows/release\.yml@refs/tags/v`, and every verification blizzard
documents or runs uses exactly that pair. A release's signature therefore cannot come from a branch, a pull request, or
any other workflow — the `edge` channel included (§The `edge` channel).

## The image signature

After the push, the `build` job signs the image by the digest of its manifest list with
`cosign sign --yes --recursive ghcr.io/paul-gross/blizzard-hub@<digest>` (`sigstore/cosign-installer`, pinned).
`--recursive` signs the manifest list and each platform manifest under it, so an image pulled by a platform digest
verifies as well as one pulled by the multi-arch digest. The signature is stored beside the image in GHCR, where
`cosign verify` finds it. A tag is never signed, only a digest, so moving `latest` or `0.7` changes nothing a verifier
checks.

## Provenance

Every artifact carries SLSA v1 build provenance, generated and signed by `actions/attest-build-provenance` (pinned):

- **The image**, by the digest of its manifest list, with `push-to-registry: true`, so the attestation travels with the
  image in GHCR as well as in GitHub's attestation store.
- **Every attached file**, `SHA256SUMS` included, as `subject-path` subjects of one attestation.

The provenance names the repository, the workflow, the tag, the commit, and the run, which is what lets a person trace a
file on their disk to the commit it was built from. `gh attestation verify <file-or-oci-ref> --repo paul-gross/blizzard`
checks it, and `--cert-identity-regex` with the identity above narrows it to a release.

## Bills of materials

Each release carries SPDX 2.3 JSON bills of materials, generated with Syft (`anchore/sbom-action`, pinned), attached to
the release ([artifacts.md](./artifacts.md#the-inventory)) and attested with `actions/attest-sbom` (pinned):

- **The image**, one per platform, by scanning `registry:ghcr.io/paul-gross/blizzard-hub@<platform digest>`. It lists
  the Debian packages of the `python:3.12-slim` base as well as every Python package installed, which is what a
  vulnerability scanner needs. Each is attested against its platform digest with `push-to-registry: true`.
- **The wheel**, by scanning a clean virtualenv installed from the base requirements file — exactly what an operator's
  install holds — merged with the production packages of `web/package-lock.json`, since the wheel embeds the compiled
  board and its JavaScript dependencies ride inside it unseen by any Python scan. It is attested against the wheel's
  digest.

A bill of materials describes the artifact it is attested against and nothing else; a vulnerability verdict over it is
not part of a release, and a scanner's opinion never fails a cut.

## The `edge` channel

`push.yml`'s `dev-image` job signs and attests provenance for every `edge` and `sha-<full-git-sha>` image it pushes, in
the same way and with `id-token: write` and `attestations: write` added to that job only. Its identity is
`push.yml@refs/heads/master`, which the release regular expression never matches, so a verifier pinned to releases can
never be handed an `edge` image as one. The channel carries no bill of materials, and attaches nothing. The dev wheel
stays a workflow artifact, unattested: nothing installs it but the workflow that built it.

## Settings no workflow can assert

Two repository settings back the identity, and each is a one-time setting documented in `docs/ci.md` beside GHCR's
package visibility, proven only by a real cut:

- **A tag ruleset on `v*`** that lets only the repository's maintainers create a release tag and lets nobody update or
  delete one. A signature proves which workflow ran at which tag; the ruleset is what makes *who may cut a tag* a
  question with a short answer.
- **Immutable releases**, so a published release's tag and attached files cannot be replaced after the fact — the files
  a checksum vouches for stay the files on the page. Every file is therefore attached in the one `gh release create`
  call ([artifacts.md](./artifacts.md#the-release-workflows-jobs)).
