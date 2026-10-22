# Forges specifications

The technical contracts behind `epic:advanced-delivery/forges`. Product intent begins at the [epic plan](../index.md);
implementation enters at the concern below. Configured records, their revisions, retirement, the change log, and secrets
are owned by `epic:live-config`'s [records](../../../../delivered/live-config/spec/records.md) and
[secrets](../../../../delivered/live-config/spec/secrets.md) contracts, and landing only where a chunk's project links
by `epic:projects/hub`' [delivery contract](../../../../delivered/projects/hub/spec/delivery.md). This spec adds the
forge to them; it redefines none.

The initiative's epics land one after another — actions, forges, flows, policies — and this spec assumes the
[actions epic](../../actions/spec/index.md) has landed and nothing after it. The epic is its five transition steps,
delivered when they land, and they are what the `flows` and `policies` epics build on; the retirement of the old fields
is a cleanup that follows the epic, waiting on the field for a release or more
([carry-over.md](./carry-over.md#the-transition-then-retirement)).

From the actions epic it takes this much: landing is the hub's `land` action, handed its forge through `VisitForges`
beside its `Visit`, no hub step is a process, and `HubNodeExecutor` still builds each visit. Where this spec puts work
in the executor — revealing credentials, bundling each `RepositoryForge`, checking capabilities before a pass — the
[flows epic](../../flows/spec/index.md) moves it, unchanged, into its driver.

It builds on two things already true of the code. Repositories (`repo_<ulid>`) and work sources (`wsrc_<ulid>`) carry an
id, a slug, and a name as a project does: every stored reference holds the id, every boundary accepts an id or a slug
and resolves it through the shared kernel in `hub/domain/kernel/id_or_slug.py`, and a slug a record leaves behind is
held in its kind's former-slug table through `hub/store/internal/former_slugs.py`. A forge is born with the same shape
([record.md](./record.md)). Wherever this spec references a repository or a work source, it holds its id. And each of a
chunk's commit pointers already resolves to its own repository record and lands against that record's own forge address,
owner, base branch, and credential, so one chunk may already land across servers and credentials; what this epic changes
is how a target names its server, not which targets a chunk may have.

Paths are relative to `blizzard/src/blizzard/`.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`, the section of
`architecture/system-shape/seam-positions.md` titled *The forge has no single seam* collapses its `GitHubCommitResolver`
and land rows into the landing seam, and `architecture/system-shape/configuration.md` and `domain/artifacts/delivery.md`
learn the forge record; `domain/projects.md` names forges beside work sources and repositories as the tenant's; and
`domain/operations.md` gains `forge-capability-missing` and `config-old-field-used`. In `blizzard`'s own docs,
`docs/deployment/repositories.md`, `work-sources.md`, and `config-documents.md` name the forge, and a new
`docs/deployment/forges.md` describes it, and `docs/deployment/observability.md` names `forge-capability-missing` and
`config-old-field-used`. The census `tests/store_read_census.py` learns the new reads, `contracts/store/hub.json` the
new tables and columns, `contracts/cli/hub.json` the `forge` commands, and `tests/test_layering.py`'s configured-write
gate the forge's writer ([record.md](./record.md#the-table)); the legacy import's store
(`hub/store/internal/config_import_store.py`) creates the forges the import resolves. The workspace's
feature-environment seeding (`hub-host.sh`) keeps the old fields until the retirement, since it seeds feature
environments at every revision ([carry-over.md](./carry-over.md#the-transition-then-retirement)). `blizzard-mock`'s
forge grows the few states and answers the end-to-end proofs need ([carry-over.md](./carry-over.md#the-mock-forge)).

## Landing order

An installation runs a pinned build and moves onto a newer one when its operator chooses, and a release may be cut
between any two landings, so each step below lands on its own and leaves `master` releasable: whichever step a release
ends at, the installations that run it must work, and must carry over cleanly onto the next. The fleet lands this very
epic through the `land` action it rewrites, so each step also leaves the fleet landing once its instance runs it. A step
does not start until the one before it has landed.

1. **The seam and the port.** `hub/forge/landing.py`'s roles and value types, the GitHub adapter and its unit tests, the
   registry, `ACTION_REQUIREMENTS` and the capability check, and `land` and the garden's commit check moved onto the
   seam, with `GitHubHandle` deleted. `hub/domain/config/capabilities.py` lands here, and so does
   `hub/domain/config/forges.py` holding only `KNOWN_FORGE_KINDS` and `FORGE_KIND_CAPABILITIES`, which step 3 grows into
   the record. No schema changes: the registry builds each adapter from the repository's own `forge_api_url`, `offered`
   answers the `github` kind's set for every repository and takes its `forge` argument from step 3, when there is a
   record to pass, and an escalation names the repository's forge by that address until step 3 gives it a record. A
   regression in the port shows on the fleet's own landings, on a build where nothing else has moved, and walking it
   back is a revert.
2. **The pre-flight check.** A check that reports, without changing anything, every record the migration would refuse
   and why, reusing the migration's canonicalization rule: `blizzard hub config check-forges`, and a warning in the
   hub's startup log naming each one. The command reads a hub's store directly, as `blizzard hub migrate` does, rather
   than asking a running hub, so an operator runs it from the build they are about to install against the store their
   current hub still serves, and settles what it names on that hub before the move — whether or not any build between
   the two ever ran. It reads the address columns as it finds them in the tables, never through the store's schema
   module, so it answers alike before the migration lands and after, and on a store already carried over says there is
   nothing left to carry. It lands ahead of the migration so that no build carries the migration without it
   ([carry-over.md](./carry-over.md#the-migration)).
3. **The record and the migration.** The forge table and references, the migration and its downgrade, the record store,
   `ConfigAuthoring`'s forge writes, `lock_forge_row`, the invariants, the change-log enumerations, and old-field
   handling at every door — API, CLI, and the configuration document — together in one change, since from the migration
   on every record names its forge and a door left behind would write one that names none. The registry switches from
   the repository's address to its forge in the same change. Every door still accepts the old fields and answers them
   with the warning, and the create routes take request models of their own so that `forge` reaches no document a client
   already reads; nothing outside the hub has to change for it. The store only expands: the old columns stay beside
   `forge_id`, kept equal to the forge and marked deprecated, and no constraint already there changes
   ([carry-over.md](./carry-over.md#expand-now-contract-at-retirement)).
4. **The forge's own surfaces.** `/api/forges`, `blizzard hub forge`, version 2 of the configuration document with its
   `forges` section, its apply, export, and schema, and the board's Forges tab, forge picker, `'select'` field kind, and
   labelled change-log rows.
5. **Documentation and harness.** `docs/deployment/`, and the `blizzard-context` rules named above.

Steps 1 through 5 are the transition, which may reach an installation across more than one release, and are the whole
epic. The retirement is not part of it: it is a standalone cleanup, marked breaking, taken up once the field says it is
time ([carry-over.md](./carry-over.md#the-transition-then-retirement)).

| Where                            | Read when                                                                                                                     |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| [record.md](./record.md)         | Shaping the forge record — its table, the references from repositories and work sources, retirement, invariants               |
| [seam.md](./seam.md)             | Building the landing seam — its roles, vocabulary, failures and retries, capabilities, the GitHub adapter, the registry       |
| [landing.md](./landing.md)       | Porting the `land` action onto the seam, what it answers to each forge refusal, and the capabilities an action requires       |
| [surfaces.md](./surfaces.md)     | Building the API, CLI, configuration document, and admin screens for forges and the records that name them                    |
| [carry-over.md](./carry-over.md) | Writing the migration — one forge per server already named, every reference filled, the old fields' transition and retirement |
