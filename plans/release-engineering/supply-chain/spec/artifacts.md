# What a release publishes

## The inventory

A release `v<version>` publishes two things: the hub image on GHCR, and a GitHub release whose attached files are the
set below. The release page is the only place blizzard's Python artifacts are published. `<version>` is the tag without
its `v` (`0.7.0`, `0.7.0-rc.1`); the wheel alone keeps the PEP 440 filename the build gives it (`0.7.0rc1`).

| Attached file                                  | What it is                                                                                                                                    |
| ---------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `blizzard-<pep440>-py3-none-any.whl`           | The one wheel                                                                                                                                 |
| `blizzard-<version>-requirements.txt`          | The runtime set the gate proved: every dependency pinned and hashed, and the wheel itself pinned by its address on this release and its hash  |
| `blizzard-<version>-requirements-pg.txt`       | The same set with the `postgres` extra — what the image installs                                                                              |
| `blizzard-<version>.spdx.json`                 | The wheel's bill of materials ([attestation.md](./attestation.md#bills-of-materials))                                                         |
| `blizzard-hub-<version>-<os>-<arch>.spdx.json` | The image's bill of materials, one per platform the image is built for (`linux-amd64`, `linux-arm64`)                                         |
| `blizzard-hub-<version>-image.txt`             | One line: `ghcr.io/paul-gross/blizzard-hub@sha256:<digest>`, the multi-arch image the release pushed, by the digest of its manifest list      |
| `compat-breaks.json`                           | Every break the release's compatibility backstop let through, owned by `epic:release-engineering/compat-floor` (`compat-floor/spec/gates.md`) |
| `SHA256SUMS`                                   | `sha256sum` output over every other attached file, sorted by name                                                                             |

Nothing else is attached, and every one of these is attached on every release, candidates and patches alike, save
`compat-breaks.json`: it is attached whenever the build produced it, which is every release from the first one whose
workflow runs `epic:release-engineering/compat-floor`'s backstop, and from then on it is attached even when it lists no
break, so its absence is never mistaken for a release without any. Before that release, the file does not exist and is
not attached; no stand-in is written in its place, since its shape is compat-floor's. The release notes are the
release's body, not a file (`epic:release-engineering/notes`). The image's tag fan-out is unchanged
(`scripts/image-tags.sh`); the image file names the digest so a person who trusts the checksums file can pin the image
by content rather than by a tag that a registry lets move.

## The hashed requirements

`uv.lock` is the set the gate proves blizzard against — every job that runs tests `uv sync`s from it. A new
`scripts/export-requirements.sh <version> <wheel> [--extra <name>] [--no-project]` turns it into a release's
requirements file: `uv export --locked --no-dev --no-emit-project --format requirements-txt`, hashes included, with
`--extra postgres` for the `-pg` file, followed by one line naming the project by its address on the release it belongs
to:

```text
blizzard @ https://github.com/paul-gross/blizzard/releases/download/v<version>/blizzard-<pep440>-py3-none-any.whl --hash=sha256:<wheel digest>
```

`--locked` fails the export when `uv.lock` disagrees with `pyproject.toml`, so a release never ships a set the gate did
not run. `--no-project` leaves the last line off, for a build that installs the wheel from its own context. The script
is unit-tested beside `tests/test_image_tags.py`, and the release job runs it right after the wheel is built, before
anything is pushed.

The requirements file is the chain of trust an install hangs from: its own attestation vouches for it
([attestation.md](./attestation.md)), and it carries the hash of every package an install will fetch, the wheel
included. Verifying one file vouches for the whole install.

The image installs the same set. `packaging/docker/Dockerfile` copies the wheel and the `-pg` file's `--no-project` form
into the build context, runs `pip install --no-cache-dir --require-hashes -r /tmp/requirements.txt`, then
`pip install --no-cache-dir --no-deps` the wheel — the release page does not exist yet when the image is built, and the
wheel is the build's own, the file `SHA256SUMS` later names. `scripts/build-wheel.sh`'s node-free install proof installs
the same way from the base file, so the local build proves the install an operator will run. The dev channel
(`push.yml`) exports and installs the same `--no-project` files for the dev version, so `edge` and a release are built
by one recipe.

## The release workflow's jobs

The single `release` job becomes three, after the test jobs it already waits for:

| Job       | Needs     | Scopes                                                                        | Does                                                                                                                                                                                                                                                                                                                                                |
| --------- | --------- | ----------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `build`   | the tiers | `contents: read`, `packages: write`, `id-token: write`, `attestations: write` | Checks the tag, builds the wheel, exports both requirements files, runs the compatibility backstop that writes `compat-breaks.json`, generates the notes from it, builds and pushes the image, signs and attests it, writes every bill of materials, the image file, and `SHA256SUMS`, attests the files, and uploads them as one workflow artifact |
| `publish` | `build`   | `contents: write`                                                             | `gh release create` with every attached file in one call                                                                                                                                                                                                                                                                                            |
| `verify`  | `publish` | `contents: read`                                                              | Proves the published release from outside ([verification.md](./verification.md))                                                                                                                                                                                                                                                                    |

The order is the rule the workflow already keeps — nothing is advertised before it exists — extended to everything the
release now publishes. The release page comes last because it is what a person reads and what an install fetches from. A
run that fails after the image push leaves a version tag on GHCR with no release behind it, as a failed
`gh release create` does today, and is repaired forward with the next tag (`bzh:release` §A red run).

`compat-breaks.json` is written in `build`, not `publish`, so that it is covered by `SHA256SUMS` and the files'
provenance attestation like every other attached file, and travels in the same workflow artifact; `publish` attaches it
in the one `gh release create` call, before immutable releases lock the page. Its shape, and what it records, are
`epic:release-engineering/compat-floor`'s; this epic only carries it.

Each job holds only its own scopes, and the workflow level stays `contents: read`. `tests/test_release_workflow.py`
learns the new shape: each job's scopes exactly as tabled, the image signed and attested after its push and before
`publish`, `verify` after `publish`, and no job but `publish` holding `contents: write`. Every third-party action the
new jobs use is pinned by commit SHA.
