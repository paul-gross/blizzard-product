# Forges specifications

The technical contracts behind the forges slice of `epic:advanced-delivery`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. Configured records, their revisions, retirement,
the change log, and secrets are owned by `epic:live-config`'s
[records](../../../../delivered/live-config/spec/records.md) and
[secrets](../../../../delivered/live-config/spec/secrets.md) contracts, and landing only where a chunk's project links
by `epic:projects`' [delivery contract](../../../../delivered/projects/hub/spec/delivery.md). This spec adds the forge
to them; it redefines none.

This spec assumes the [actions slice](../../actions/spec/index.md) has landed, so landing is the hub's `land` action and
no hub step is a process.

It builds on two things already true of the code. Repositories (`repo_<ulid>`) and work sources (`wsrc_<ulid>`) carry an
id, a slug, and a name as a project does: every stored reference holds the id, every boundary accepts an id or a slug
and resolves it through the shared kernel in `hub/domain/kernel/id_or_slug.py`, and a slug a record leaves behind is
held in its kind's former-slug table through `hub/store/internal/former_slugs.py`. A forge is born with the same shape
([record.md](./record.md)). Wherever this spec references a repository or a work source, it holds its id. And each of a
chunk's commit pointers already resolves to its own repository record and lands against that record's own forge address,
owner, base branch, and credential, so one chunk may already land across servers and credentials; what this slice
changes is how a target names its server, not which targets a chunk may have.

Paths are relative to `blizzard/src/blizzard/`.

The slice reaches beyond `blizzard`'s source. In `blizzard-context`, the section of
`architecture/system-shape/seam-positions.md` titled *The forge has no single seam* collapses its `GitHubCommitResolver`
and land rows into the landing seam, and `architecture/system-shape/configuration.md` and `domain/artifacts/delivery.md`
learn the forge record; `domain/projects.md` names forges beside work sources and repositories as the tenant's. In
`blizzard`'s own docs, `docs/deployment/repositories.md`, `work-sources.md`, and `config-documents.md` name the forge,
and a new `docs/deployment/forges.md` describes it. The census `tests/store_read_census.py` learns the new reads. The
workspace's feature-environment seeding (`hub-host.sh`) names its forge by `forge` in the transition release, so a
feature environment never keeps `config-old-field-used` firing. `blizzard-mock`'s forge is unchanged.

| Where                            | Read when                                                                                                                     |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| [record.md](./record.md)         | Shaping the forge record — its table, the references from repositories and work sources, retirement, invariants               |
| [seam.md](./seam.md)             | Building the landing seam — its roles, vocabulary, failures and retries, capabilities, the GitHub adapter, the registry       |
| [landing.md](./landing.md)       | Porting the `land` action onto the seam, what it answers to each forge refusal, and the capabilities an action requires       |
| [surfaces.md](./surfaces.md)     | Building the API, CLI, configuration document, and admin screens for forges and the records that name them                    |
| [carry-over.md](./carry-over.md) | Writing the migration — one forge per server already named, every reference filled, the old fields' transition and retirement |
