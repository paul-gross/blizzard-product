# Carry-over

An installation that predates this epic keeps landing exactly as it did, with every forge it already talks to declared
for it, in one migration and with nothing reconfigured by hand.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, frozen once released
(`bzh:frozen-revisions`) and inside the portable surface (`bzh:sql-portable`), proven on Postgres as on SQLite
(`tests/test_store_migrations.py`) with every constraint and index name inside Postgres's 63 characters
(`tests/test_migration_identifier_length.py`), and tested beside its revision (`bzh:colocated-migration-tests`). Its
`down_revision` is the chain's head when it merges — after the actions epic's revision, since that epic lands first —
and it is re-chained if another revision lands first, as the identity and host revisions were. It follows the identity
revisions' shape: every step inspects first and skips what a stopped run already did, so a migration that dies midway is
finished by running it again. Within every tenant it:

1. **Refuses what it cannot carry.** Before it writes anything, it canonicalizes every origin the tenant's records name
   — each repository's `forge_api_url`, and each GitHub work source's `api_base`, or `https://api.github.com` where that
   is null, empty, or only whitespace, as the work-source factory reads it today — and each explicit `web_base`, under
   `canonical_origin`'s rule ([record.md](./record.md#the-table)), re-implemented in the revision as the slug rule is
   below, so a port such as a feature environment's `http://127.0.0.1:<port>` is kept. Retired records take part like
   enabled ones, since a retired repository keeps its coordinate claim and a retired work source its locator. The
   migration refuses the upgrade, before any step runs and naming every record involved by slug, as the identity
   revisions refuse a name they cannot carry, in four cases:
   - **An address that is not one.** A work source's addresses were never validated, so a stored `api_base` or
     `web_base` may lack a scheme (`ghe.acme`), name a scheme other than `http` or `https`, or have no host. The
     migration will not guess which server was meant. The operator edits the record's address to the server's origin.
   - **An address it would have to clean.** An origin carrying a user or password, a query, or a fragment, which
     `canonical_origin` refuses rather than strips. The operator edits the record's address to the bare origin.
   - **Two repositories that would become one coordinate.** Two repositories that differ today only in how their origin
     was spelled, such as a trailing slash, would collide under `uq_repositories_tenant_coordinate` once step 3 stores
     each address canonical, and would name one coordinate on one forge besides. Retiring one of the pair does not
     settle it, since a retired record keeps its claim and is never deleted. Editing a retired record is legal, so the
     operator edits one of the pair — usually the retired one — so the two no longer name one coordinate: its `repo` to
     a name that is no longer in use, or its address to the server it truly meant. Work sources cannot collide this way,
     since their locator constraint is unchanged through the transition ([record.md](./record.md#references)).
   - **Two web addresses for one server.** Two GitHub work sources that declare different explicit `web_base`s for one
     canonical origin, compared canonical, would leave its forge with no single web address. A work source whose
     `web_base` is null takes no part, since its effective web origin is derived and the forge derives the same one; but
     where one source on an origin declares an explicit `web_base` and another leaves it null, the forge takes the
     explicit one, and the migration's log names the null source whose web links move with it. The operator makes the
     explicit ones agree.

   On a hub that migrates as it starts, as the hosted hub does, a refused upgrade is a failed deploy, and the old hub is
   the only one left to make the edit on. So none of these four need first be met by the migration: the pre-flight check
   ([Landing order](./index.md#landing-order)) reads the store the current hub serves, from the build about to replace
   it, and names the same records, so the operator settles them on the current hub before moving — even when the build
   they move from predates the check. The migration's refusal is the backstop for an installation that skipped it, and
   since it refuses before any step runs, going back from it is the previous build alone, the store untouched.
2. **Creates the forges.** It creates one forge per distinct canonical origin, visiting origins in sorted order so a
   re-run slugs them identically: kind `github`, that `api_base`, and the explicit `web_base` a work source declared for
   that origin, else null. Its `frg_<ulid>` id is minted deterministically, as the identity revisions mint theirs — the
   timestamp from the forge's `created_at`, which is the earliest `created_at` among the records naming its origin, and
   the random bits hashed from the tenant and the origin — so a re-run mints the same id. A forge on `api.github.com` is
   slugged `github` and named `GitHub`; any other takes its web host — the host of its effective web origin, with its
   port when one is kept — as its slug under `derive_slug`'s rule, so `https://ghe.acme.internal/api/v3` is
   `ghe-acme-internal` and a feature environment's `http://127.0.0.1:8123` is `127-0-0-1-8123`, suffixed `-2` and
   onwards on collision, and that web host itself as its name. The revision re-implements the slug rule locally rather
   than importing it, as the identity revisions' `_derived` and `_suffixed` do, so a later change to the rule never
   changes what a frozen revision did. The operator changes either afterwards like any other forge's. Each forge gets
   revision 1 and a `config_changes` row attributed to `migration`; a forge created retired, below, also gets the
   `forge_lifecycle_facts` row that retires it, since creating a record writes no lifecycle fact.
3. **Points every record at its forge.** It adds `repositories.forge_id` and `work_sources.forge_id`, nullable and with
   no foreign key, sets each to the forge created for the record's origin, and rewrites the record's old address columns
   to its forge's, as the store keeps them from then on ([record.md](./record.md#references)): `forge_api_url` and
   `api_base` to the forge's canonical `api_base`, a null `api_base` included, and `web_base` to the forge's own. It
   marks the three old columns deprecated in `schema.py` (`info={"deprecated": …}`, naming the line of the release that
   carries the revision). A retired record is pointed at its forge like any other, and a forge named only by retired
   records is created retired.

   It changes no constraint the store already has. The repository coordinate keeps keying on `forge_api_url`, which now
   holds its forge's canonical origin, one per forge, so it keys on the forge as surely as `forge_id` would; the
   work-source locator is left as it is.

The downgrade walks the store back whole, as every revision's must (`docs/versioning.md`; `test_migrate_up_and_down`):
it drops the `forge_id` columns, the forge tables, and the forges' change rows, which an older hub cannot read, and
clears the deprecation marks. The old columns already hold every address a hub before the epic reads, so it restores
nothing and never refuses; an address reads canonical rather than as it was first typed, and a work source's null
`api_base` as the address it always resolved to. A repository's or work source's change row written with a `forge` field
stays, read back as any field is.

A repository whose branch links derived their web origin from its own `forge_api_url` may find them moved, where a work
source on the same server declared an explicit `web_base` the forge now carries ([record.md](./record.md#the-table)).
The migration's log names each such repository. No repository or work source changes revision, and no `config_changes`
row is written for them: the forge they name is the address they already held.

The legacy import, `blizzard hub config import-legacy` (`hub/domain/config/carry_over.py`), builds repositories from
`BZ_FORGE_URL` and work sources from their `api_base` and `web_base`. It resolves or creates forges exactly as the
migration does — canonicalizing each origin first, so a `BZ_FORGE_URL` that differs from a stored forge's origin only in
spelling resolves to that forge, where its `held` map keys on the raw string today — and records each forge it creates
under the import's own `MIGRATION_CONTEXT`, as it records every other row it writes, so an installation imported after
this epic lands in the same shape as one migrated.

## Expand now, contract at retirement

The transition changes nothing a supported release ships except by adding beside it (`bzh:released-surface-additive`),
so every step from 1 to 5 is additive to `0.2` and none is breaking. A hub walked back past the migration meets a store
it already knows how to read, and a client written before the epic meets the responses it always met.

- **The store only expands** (`bzh:store-expand-contract`). Step 3 adds the forge tables and two nullable `forge_id`
  columns, rewrites values the old columns already held, and marks those columns deprecated; it drops nothing, tightens
  nothing, and adds no constraint or foreign key to a table that already existed. What the constraints would hold, the
  store's writes and two invariants hold through the transition ([record.md](./record.md#invariants)).
- **The configuration document keeps its version.** `ConfigDocument`, `RepositoryDocument`, and `WorkSourceDocument`
  forbid a property they do not declare, and `GET /api/config/export` answers them, so a field any of them gained would
  reach a client that refuses it. They stay as they are, as the version-1 document and its entries; the forge's section
  and the `forge` reference arrive in a version-2 document beside them, and the create routes take request models of
  their own ([surfaces.md](./surfaces.md#the-configuration-document)).

The retirement, a cleanup that follows the epic, contracts what the transition left beside its replacement
([below](#the-transition-then-retirement)).

## The transition, then retirement

The epic is the transition — steps 1 through 5 on `master` — and it is delivered when they land; the `flows` and
`policies` epics build on it. The retirement that removes what the transition deprecated is a cleanup of its own that
follows the epic, belonging to none, and nothing waits for it.

**The transition** carries the migration, the forge record and its surfaces, the seam, and the `land` action ported onto
it, in the order [index.md](./index.md#landing-order) sets; a release cut partway through carries the steps before its
cut, each whole. Every old field is marked deprecated in `openapi/hub.openapi.json`, on the requests that accept it as
on the reads that return it, and the three old columns in `schema.py`, so the retirement removes what the releases it is
judged against already deprecate. Each deprecation is also announced as `actions` announces its own: a `deprecated` note
under `changes/unreleased/` (`changes/README.md`) whose `removes-in` is the minor after the release that carries it —
step 3's for the old request and response fields, the old CLI flags, and the three columns, and step 4's for the
version-1 configuration document. Should the field not be ready by that line, a `changed` note that `postpones` the
deprecation moves its `removes-in` on, before that line is cut, since the cut refuses to open a line with an unretired
promise. The old fields keep working at every door, resolved to forges ([surfaces.md](./surfaces.md#old-fields)), and
only ever to a GitHub forge, the one kind they could name: should the transition outlast the arrival of a second kind,
an old field whose origin carries that kind's API path, GitLab's `/api/v4`, is refused naming the `forge` field to use
instead, never turned silently into a GitHub forge. Nothing an operator wrote before the epic stops working, save one
write with nowhere left to go: a configuration document kept in their repository applies unchanged as version 1, and a
CLI invocation in a runbook still runs, except one that gives a work source a `web_base` disagreeing with its forge's,
which is refused naming the forge to set it on ([surfaces.md](./surfaces.md#old-fields)). The `land` action ships with
the hub, so a chunk mid-delivery when the hub restarts resumes on the ported action.

The workspace's feature-environment seeding (`.winter/config/winter-service-tmux/hub-host.sh`) writes a configuration
document naming `api_base` and `forge_api_url` today, and keeps doing so until the retirement. One script seeds every
feature environment, and the environments run blizzard at different revisions — some before the epic, which refuse an
entry they do not know — so the old fields are the only document all of them accept. A feature environment's
`config-old-field-used` events are expected and say nothing about the instances blizzard runs. The script moves to a
version-2 document, with a `forges` section and `forge` references, in the retirement's own change.

**The retirement** removes the old fields from every door, refusing them `422 field_moved_to_forge`, refuses a version-1
configuration document naming version 2, answers version 2 from the export and schema routes when no version is asked
for, and removes the old CLI flags. Its revision contracts the store: it tightens both `forge_id` columns to `NOT NULL`
foreign keys onto `forges`, rebuilds the repository coordinate as `(tenant_id, forge_id, owner, repo)`, widens the
work-source locator to its forge ([record.md](./record.md#references)), and drops `repositories.forge_api_url`,
`work_sources.api_base`, and `work_sources.web_base`. Its downgrade restores the three columns from each record's forge
and the constraints on them, and refuses a store holding one locator on two forges, naming the pairs. It is not part of
the epic. It is a standalone cleanup chunk, filed beside the epic but taken up only when a person decides the time has
come rather than decomposed from this spec, no sooner than one release after a release carrying all five transition
steps ships, and only once the hosted hub has recorded no `config-old-field-used` event through a whole release. Because
it removes request and response fields, a document version, and the old CLI flags, and contracts the store, its commit
is marked breaking, `feat!` (`docs/versioning.md`), its `removed` notes each `retire` their deprecation, and its release
notes name each removed field with what replaces it. The constraints it adds to `repositories` and `work_sources` sit on
tables no release marks deprecated, so they pass as a break the window admits: the cleanup raises `floor`
(`wire/window.toml`) to the line that first shipped the deprecations, which refuses a runner below that line from then
on. It is taken up together with the other advanced-delivery cleanups, the `actions` and `flows` epics', so `floor`
rises once for all of them. The event joins the closed `EventLogKind` and its severity map (`foundation/event_log.py`)
as a warning, and is removed with the fields it reports.

Runners are untouched by both: nothing in this epic crosses the runner's wire.

## The mock forge

`blizzard-mock`'s forge (`blizzard-mock/src/blizzard_mock/forge/`) already answers most of what the seam asks: the
compare, ref, commit, check-run, re-run, and pull-request routes, `GET /repos/{owner}/{repo}` for the adapter's 404
probe, a 404 for an unknown repository, and the `token_rejected` (401) and `rate_limited` (403 with `Retry-After`)
levers. It grows what the end-to-end proofs below need and does not yet have:

- **The mergeable states it cannot yet show.** Its `MergeableState` carries `clean`, `dirty`, `unknown`, `behind`, and
  `blocked`; a lever forcing `unstable`, `has_hooks`, or `draft` on a pull request lets the proof show that `land`
  merges through the first and waits on the other two, as it does today.
- **A conflicting `update-branch` answered as GitHub answers it.** The mock answers 405 for an update it cannot make,
  where GitHub answers 422 with a message naming the conflict, so `Update(conflict)` is exercised against the mock and
  not only on a later poll.
- **A check run that failed to start.** A per-pull-request lever for a `startup_failure` conclusion, so the proof shows
  that it waits rather than routing `failure`.

The adapter's own vocabulary and failure tables are pinned by its unit tests against a recorded `httpx` transport, not
the mock, so every row — a secondary rate limit, a 405 from `merge`, an Enterprise Server `/api/v3` origin — is proven
without the mock having to fake it.

## Proof

The invariant checker's `hub:forge-reference-standing` passes on a carried-over store. A fixture store captured before
the migration, carried over, serves the same repository and work-source documents with `forge_id`, `forge_slug`, and
`forge_name` beside the addresses they held, resolves every chunk's commits to the same repositories, and lands a chunk
against the mock forge through the seam with the same outcomes and markers the `land` action produced before it. A
configuration document written before the epic applies to it unchanged, with a warning per old field, and its version-2
export applies cleanly after retirement. A store holding a feature environment's `http://127.0.0.1:<port>` origin
carries over with its port; one holding two spellings of one coordinate, retired or not, is refused naming both, and
carries over once one of them is edited; one holding an address with a user in it, or a work-source address with no
scheme, is refused naming the record; and one whose work source stores an empty `api_base` carries over onto the
`github` forge. The downgrade walks a carried-over store back to the shape it came from, its addresses canonical, and a
hub from before the epic starts on it. Every case in `tests/fixtures/canonical_origin_cases.json` answers alike from
`canonical_origin` and from the revision's frozen copy. The pre-flight check names, on a store the migration would
refuse, the same records the migration names. Beyond the carry-over:

- A registry handed a capabilities map that offers less than `land` requires refuses the visit before the pass, the
  chunk escalated once with cause `forge-capability-missing` and a second `hub-advance` against the same visit finding
  it escalated, since GitHub, the one kind this epic builds, offers them all.
- A check re-run the forge refuses with a 403 writes no re-run marker and routes the failing check `failure`, never
  `forge-auth`; a 403 on any other call escalates `forge-auth`.
- A compare the forge rate-limits in the head gate answers `pending`, not the foreign-head `failure`; a compare
  returning fewer commits than `ahead` refuses the head as truncated; a missing base branch answers `pending` while a
  missing repository met at `branch_tip` escalates `forge-config`.
- The GitHub adapter's unit tests pin every row of the vocabulary and failure tables in [seam.md](./seam.md), including
  `has_hooks`, `draft`, `startup_failure`, an unknown state and conclusion, a secondary rate limit, a merge refused 405,
  a 404 told apart as a missing repository or a missing change, and a GitHub Enterprise Server `/api/v3` origin; a
  further test holds the adapter's offered constant equal to `FORGE_KIND_CAPABILITIES["github"]`. `land`'s tests pin
  every row of [landing.md](./landing.md#what-the-forge-answers).
- Against the mock: a green `unstable` pull request merges, a green `draft` or `has_hooks` one waits, a
  `startup_failure` check waits, a conflicting update routes `conflict`, a rejected token escalates `forge-auth`, and a
  repository the forge does not know escalates `forge-config`.
- An adapter superseded by a forge edit is closed on the action loop, and a pass still holding it finishes its call; a
  hub shut down with adapters cached closes them before the action loop stops.
- The garden's commit check resolves a delivered commit through the seam, and degrades to `None` for an unrevealable
  secret, an unavailable forge, a repository the credential cannot see, a check that outlasts its ten seconds, and a
  repository no link resolves.
- Retiring a forge in use answers `409 forge_in_use` listing its referrers; enabling a record whose forge is retired
  answers `422 forge_retired`; `forge_conflict`, `forge_kind_mismatch`, `forge_unavailable` (an unknown forge, and an
  old field whose origin a retired forge holds), `forge_required`, and `ForgeOriginTaken` each have a refusal test; two
  concurrent old-field writes for one new origin make one forge; a retire and an enable racing on one forge never leave
  an enabled record naming a retired forge; a configuration entry that omits a work source's old fields leaves its forge
  as stored, while `--clear api-base` re-points it at the `github` forge with a warning; and
  `hub:forge-reference-standing` and `hub:forge-references-in-tenant` each catch a planted breach.
