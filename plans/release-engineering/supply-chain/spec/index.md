# Supply-chain specifications

The technical contracts behind `epic:release-engineering/supply-chain`. Product intent begins at the
[epic plan](../index.md); implementation enters at the concern below. The version scheme and what counts as breaking are
owned by `blizzard`'s `docs/versioning.md`; cutting a release, and which exact release each deployment of the instance
is pinned to, are owned by `epic:release-engineering/first-release`; the release notes a release carries are owned by
`epic:release-engineering/notes`; release branches and patch tags by `epic:release-engineering/release-lines`. This spec
adds signatures, attestations, bills of materials, checksums, and the pinned runner install to the release those epics
cut; it redefines none of them.

It builds on what the release workflow already does. A `v*` tag runs `.github/workflows/release.yml`: the merge gate and
the three upper tiers, then one `release` job that checks the tag against `pyproject.toml`
(`scripts/check-version-tag.sh`), builds the one wheel with the embedded board (`scripts/build-wheel.sh`), generates the
notes (`scripts/release-notes.sh`), pushes the multi-arch hub image to `ghcr.io/paul-gross/blizzard-hub` under the tag
fan-out `scripts/image-tags.sh` derives, and only then publishes the GitHub release with the wheel attached. Every
`master` push publishes the dogfood channel — the dev wheel as a workflow artifact and the `edge` and
`sha-<full-git-sha>` images (`.github/workflows/push.yml`). Nothing is signed or attested, and the image's `pip install`
(`packaging/docker/Dockerfile`) resolves its dependencies afresh rather than from `uv.lock`.

Paths are relative to the `blizzard` repo root unless a leaf says otherwise.

The epic reaches beyond `blizzard`. In `blizzard-infra`, the updater (`deploy/bin/blizzard-update`) and the installer
(`deploy/bin/blizzard-install`) learn to verify an image before running it, and `docs/deployment.md` and
`docs/rollback.md` say so. In `blizzard-context`, `workflows/release.md` (`bzh:release`) gains the confirmation this
epic adds. In the workspace, the runner half's redeploy runbook (`context/project/local-instance.md`) installs the host
from a verified wheel. `blizzard-mock` publishes nothing and is unchanged.

| Where                                | Read when                                                                                                                                |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| [artifacts.md](./artifacts.md)       | Shaping what a release publishes — every attached file and its name, the checksums, the hashed requirements, the job's order             |
| [attestation.md](./attestation.md)   | Signing the image and attesting provenance and bills of materials — the identity, the tools, the formats, the scopes, the `edge`         |
| [install.md](./install.md)           | Installing a runner at a named version — the command, the unit, upgrading, and the operator docs that describe it                        |
| [verification.md](./verification.md) | Proving a release from outside — the workflow's own verify job, the `bzh:release` confirmation, a bad release, and the instance's checks |
| [carry-over.md](./carry-over.md)     | Landing the epic — the order of changes, the one-time settings no workflow can assert, and the releases cut before it                    |
