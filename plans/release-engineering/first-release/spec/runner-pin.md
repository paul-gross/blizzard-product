# Runner pin contract

Paths are relative to the workspace root unless they name another repository.

## Where the pin lives

The runner host's release is a row of the deployed-surface table in `context/project/local-instance.md`, the file that
already owns the instance's deployed surface:

| Piece   | Location / value                                                                        |
| ------- | --------------------------------------------------------------------------------------- |
| Release | `0.2.0-rc.1` — the exact release the venv runs, installed from its GitHub Release wheel |

It replaces the **Wheel source** row, which names a build from `projects/blizzard` on `master`; the **Venv** row stays.
The row moves only in a workspace commit made by the procedure that installs the release
([procedures.md](./procedures.md)), so the committed row and the installed venv never disagree for longer than one
redeploy. The hub's pin is not restated here: the row next to it points to `blizzard-infra`'s `deploy/release.env` by
name, as the file already defers to that repository for the hub.

## Installing a release

The runbook (`context/project/local-instance.md` §Redeploy + restart runbook) changes in its first three steps; the rest
— migrating `data/host.db`, `blizzard runner host --check`, minting graphs, restarting the unit, the fleet-worker bus
exports — keeps its shape.

1. **Confirm the hub.** `curl -s https://blizzard.grosscode.net/api/health | jq -r .version` must equal the release
   being installed, or be newer within the window `docs/versioning.md` states. A runner host never moves to a release
   newer than the hub it works against; the runbook stops there.
2. **Fetch the wheel** from the release itself, never a build:
   `gh release download v<version> --repo paul-gross/blizzard --pattern 'blizzard-*.whl' --dir <scratch>`.
3. **Install it** into the venv: `uv pip install --python ../.venv --reinstall <scratch>/blizzard-*.whl`.

Once `epic:release-engineering/supply-chain` lands, steps 2 and 3 become its install: the release's attested, hashed
requirements file, verified and then synced into the venv ([install](../../supply-chain/spec/install.md#the-runner),
[verification](../../supply-chain/spec/verification.md#the-instance)). Because `../.venv` also holds the operator CLI,
the runbook first checks that the venv holds nothing the sync would remove, and stops if it does.

No step reads `projects/blizzard` any more: `winter ws fetch --all` and `mise run build` leave the runbook. The source
checkout keeps serving feature environments and nothing else.

## Graphs

Minting stays a step of the runbook and reads the installed venv, as today. What it diffs changes: the graphs to mint
are those changed between the release the host ran and the one it now runs,
`git -C projects/blizzard diff --name-only v<previous>..v<version> -- src/blizzard/hub/graphs/`, read from the tags so
the answer is the same however far `master` has moved. The rules the runbook already states about prompt-only changes
and minting from the venv are unchanged.

## The redeploy proof

A release wheel carries no local segment, so the host's `GET /api/health` reports the release exactly — `0.2.0rc1` for
`v0.2.0-rc.1`, `0.2.0` for `v0.2.0`. The runbook's proof becomes that string equal to the **Release** row's version in
PEP 440 normal form; the `+<short-sha>` check and the `.dirty` warning go, since a build from a source tree is no longer
how the instance is deployed. Any other version reported after a redeploy — a `+` segment above all — means the venv is
not what the row says, and the runbook stops.

## Going back

Going back is the same procedure towards an older release, runner host first and hub after
([procedures.md](./procedures.md)). The host's store may be ahead of what the older release expects, so before the older
wheel is installed:

1. Read the head revision the older release expects: install its wheel into a throwaway venv and run
   `blizzard runner migrate --head` from it ([versions.md](./versions.md) §Revision queries every release answers).
2. With the unit stopped, downgrade on the **current** venv, which holds the downgrade code:
   `../.venv/bin/blizzard runner migrate --dir ../runner --down <revision>`.
3. Install the older wheel, check the directory, and restart, as in a forward move.

The unit stays stopped through all three steps; a downgrade that fails leaves it stopped on the current venv, which is
restarted unchanged.

## `context/project/post-delivery.md`

The file's rule — every landing on `master` is followed by a rebuild and a redeploy of the runner host — is replaced. A
landing owes the instance nothing; the runner host is owed a redeploy when the hub's pin moves, and the procedure that
moves the hub moves the host after it. What the file keeps:

- **What a landing asks of its redeploy.** A landed issue's `## Landing and deployment` section stops being read at
  landing. Its extra steps become the release's: before installing a release, the procedure reads its **Upgrade notes**
  and, until `epic:release-engineering/notes` carries every such step into them, the `## Landing and deployment`
  sections of the issues the release's range closed. `Deploy: by hand` keeps its meaning, now of a release: no fleet
  worker installs it.
- **Never restart onto a broken build**, **the restart is fleet-wide**, and **restarting when you are a fleet worker**
  keep their content, now attached to installing a release rather than to delivering a chunk.
- **Confirm you are deploying what you landed** becomes confirming the venv runs the release the row names — the proof
  above — and leaves the fetch-before-build guidance behind with the build.

A fleet worker that lands a chunk therefore ends its delivery at landing. AGENTS.local.md's line that every landing owes
the host a hand redeploy changes with it.
