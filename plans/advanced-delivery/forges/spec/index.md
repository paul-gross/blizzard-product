# Forges specifications

The technical contracts behind the forges slice of `epic:advanced-delivery`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. Configured records, their revisions, retirement,
the change log, and secrets are owned by `epic:live-config`'s
[records](../../../../delivered/live-config/spec/records.md) and
[secrets](../../../../delivered/live-config/spec/secrets.md) contracts, and landing only where a chunk's project links
by `epic:projects`' [delivery contract](../../../../delivered/projects/hub/spec/delivery.md). This spec adds the forge
to them; it redefines none.

This spec assumes [blizzard#1105](https://github.com/paul-gross/blizzard/issues/1105) (repositories) and
[blizzard#1106](https://github.com/paul-gross/blizzard/issues/1106) (work sources) have landed first: repositories and
work sources carry an id, a slug, and a name as a project does, every stored reference holds the id, and every boundary
accepts an id or a slug and resolves it to the id. A forge is born with the same shape. Wherever this spec references a
repository or a work source, it holds its id.

Paths are relative to `blizzard/src/blizzard/`.

The slice reaches beyond `blizzard`'s source. `blizzard-context` restates where the forge is reached and what a `run:`
step is handed: its forge row in `architecture/system-shape/seam-positions.md` gains the landing seam in place of the
land scripts' direct path, `standards/hub-nodes/env-contract.md` loses `BZ_FORGE_*` and gains the forge callback, and
`architecture/system-shape/configuration.md` and `domain/artifacts/delivery.md` learn the forge record and that a
delivery script never holds a credential. `blizzard-mock`'s forge is unchanged, since the hub now calls it where the
scripts did.

| Where                            | Read when                                                                                                                     |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| [record.md](./record.md)         | Shaping the forge record — its table, the references from repositories and work sources, retirement, invariants               |
| [seam.md](./seam.md)             | Building the landing seam — its operations, its vocabulary, capabilities, the GitHub adapter, the readers it absorbs          |
| [landing.md](./landing.md)       | Routing a delivery script through the hub — the callback, its authority, the `run:` environment, the ported scripts           |
| [surfaces.md](./surfaces.md)     | Building the API, CLI, configuration document, and admin screens for forges and the records that name them                    |
| [carry-over.md](./carry-over.md) | Writing the migration — one forge per server already named, every reference filled, the old fields' transition and retirement |
