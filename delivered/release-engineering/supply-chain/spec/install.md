# Installing at a named version

## The runner

An operator pins a runner by naming a release, and installs it from that release's page: its requirements file
([artifacts.md](./artifacts.md#the-hashed-requirements)), fetched by its address at that exact version, installed into
the virtualenv the packaged unit already expects. This is the one way a runner is installed.

```bash
v=0.7.2
curl -fsSLO "https://github.com/paul-gross/blizzard/releases/download/v$v/blizzard-$v-requirements.txt"
uv venv /opt/blizzard/venv --python 3.12
uv pip sync --python /opt/blizzard/venv --require-hashes "blizzard-$v-requirements.txt"
```

`uv pip sync` makes the virtualenv hold exactly the file's set, nothing more, so an upgrade never leaves a dependency
behind from the version before it; `--require-hashes` refuses any package whose bytes differ from what the gate proved.
The file's last line names the wheel by its address on the same release page, so the install fetches the release's own
wheel, at the hash the file records; every dependency comes at the version and hash the gate proved. The same install
with `pip` (`python3 -m venv` then `pip install --require-hashes -r`) is documented beside it for a machine without
`uv`.

Before installing, `gh attestation verify "blizzard-$v-requirements.txt" --repo paul-gross/blizzard` proves the file
came from the release workflow at that tag ([attestation.md](./attestation.md#provenance)). Because the file carries the
hash of every package the install fetches, the wheel included, that one check vouches for the whole install. The docs
present it as a step every operator should take, and the instance takes it every time
([verification.md](./verification.md#the-instance)).

`blizzard --version` reports the release's version exactly, with no local segment, so the version an operator named is
the version the host reports at every surface that shows it.

The packaged unit (`packaging/systemd/blizzard-runner.service`) is unchanged: its `ExecStartPre` still migrates and its
`ExecStart` still runs `/opt/blizzard/venv/bin/blizzard-runner host`. An upgrade is the same four lines with the next
version, then `systemctl restart blizzard-runner`; a rollback is the same with the previous one, subject to the store's
own rules for walking a revision back (`docs/rollback.md`).

## The hub

The hub image is pinned as it already can be — `BLIZZARD_HUB_IMAGE` in `packaging/docker/.env` — and the docs now name
the digest from the release's image file as the pin that cannot move, with an exact version tag as the readable
alternative and `latest` as what neither is. Verifying the image before running it is one command, shown in
`docs/install.md`:

```bash
cosign verify ghcr.io/paul-gross/blizzard-hub@sha256:<digest> \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp '^https://github\.com/paul-gross/blizzard/\.github/workflows/release\.yml@refs/tags/v'
```

A hub installed from the wheel under systemd installs exactly as a runner does, from the `-pg` file when its store is
Postgres.

## The operator documents

Blizzard's own documents carry the install, and say it once:

- `docs/deployment/install.md` — "First install" step 2 installs from a release's requirements file as above, and
  "Upgrades" replaces `pip install` of a wheel with syncing the next version's file. It owns the commands; every other
  document points at it.
- `docs/remote-runner.md` — its Install section names the version the runner should match, pointing at
  `docs/versioning.md`'s skew window for which versions may run against which hub.
- `docs/install.md` — the hub image pinned by digest and verified with `cosign`, as above.
- A new `docs/verifying.md` — what a release publishes, the release identity, and how a stranger checks each artifact
  without trusting blizzard: the image's signature, any file's provenance, the bills of materials, and the checksums
  file. It is the page the README links for the question *is this what they built*.
