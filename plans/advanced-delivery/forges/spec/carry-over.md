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
     was spelled, such as a trailing slash, would collide under the rebuilt `(tenant_id, forge_id, owner, repo)`
     constraint. Retiring one of the pair does not settle it, since a retired record keeps its claim and is never
     deleted. Editing a retired record is legal, so the operator edits one of the pair — usually the retired one — so
     the two no longer name one coordinate: its `repo` to a name that is no longer in use, or its address to the server
     it truly meant. Work sources cannot collide this way, since their locator constraint is unchanged through the
     transition ([record.md](./record.md#references)).
   - **Two web addresses for one server.** Two GitHub work sources that declare different explicit `web_base`s for one
     canonical origin, compared canonical, would leave its forge with no single web address. A work source whose
     `web_base` is null takes no part, since its effective web origin is derived and the forge derives the same one; but
     where one source on an origin declares an explicit `web_base` and another leaves it null, the forge takes the
     explicit one, and the migration's log names the null source whose web links move with it. The operator makes the
     explicit ones agree.

   On the hosted hub, which migrates as it starts, a refused upgrade is a failed deploy, and the old hub is the only one
   left to make the edit on. So none of these four is first met by the migration: a pre-flight check lands one change
   ahead of it ([Landing order](./index.md#landing-order)), and the migration's refusal is the backstop for an
   installation that skipped it.
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
3. **Points every record at its forge.** Each repository's `forge_id` and each work source's `forge_id` is set to the
   forge created for its origin, before both columns are tightened to `NOT NULL` and the old columns dropped. A retired
   record is pointed at its forge like any other, and a forge named only by retired records is created retired.
4. **Rebuilds the repository coordinate** as `(tenant_id, forge_id, owner, repo)`. The work-source locator constraint is
   left as it is.

The downgrade walks the store back whole, as every revision's must (`docs/versioning.md`; `test_migrate_up_and_down`):
it restores `repositories.forge_api_url` from each repository's forge's `api_base` and the coordinate constraint on it,
restores each work source's `api_base` from its forge's `api_base` and its `web_base` from the forge's own `web_base`,
drops the `forge_id` columns, the forge tables, and the forges' change rows, which an older hub cannot read, and accepts
that an address comes back canonical rather than as it was first typed, and a work source's null `api_base` as the
address it resolved to. It never refuses: two forges never share an API origin, so no two repositories land on one
restored coordinate, and the work-source locator never stopped being unique without its forge. That is why the locator
does not widen until the retirement release, which is breaking already and says that its own downgrade refuses a store
holding one locator on two forges, naming the pairs.

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

## The transition, then retirement

The epic lands in two releases. The transition release is what the `flows` and `policies` epics build on; the epic is
delivered when the retirement release has shipped, but nothing after it waits for that.

**The transition release** carries the migration, the forge record and its surfaces, the seam, and the `land` action
ported onto it, in the order [index.md](./index.md#landing-order) sets. The old fields keep working at every door,
resolved to forges ([surfaces.md](./surfaces.md#old-fields)), and only ever to a GitHub forge, the one kind they could
name: should the transition outlast the arrival of a second kind, an old field whose origin carries that kind's API
path, GitLab's `/api/v4`, is refused naming the `forge` field to use instead, never turned silently into a GitHub forge.
Nothing an operator wrote before the epic stops working, save one write with nowhere left to go: a configuration
document kept in their repository applies unchanged, and a CLI invocation in a runbook still runs, except one that gives
a work source a `web_base` disagreeing with its forge's, which is refused naming the forge to set it on
([surfaces.md](./surfaces.md#old-fields)). The `land` action ships with the hub, so a chunk mid-delivery when the hub
restarts resumes on the ported action.

The workspace's feature-environment seeding (`.winter/config/winter-service-tmux/hub-host.sh`) writes a configuration
document naming `api_base` and `forge_api_url` today, and keeps doing so until the retirement release. One script seeds
every feature environment, and the environments run blizzard at different revisions — some before the epic, which refuse
an entry they do not know — so the old fields are the only document all of them accept. A feature environment's
`config-old-field-used` events are expected and say nothing about the instances blizzard runs. The script moves to a
`forges` section and `forge` references in the retirement release's own change.

**The retirement release** removes the old fields from every door, refusing them `422 field_moved_to_forge`, removes the
old CLI flags, and widens the work-source locator to its forge ([record.md](./record.md#references)). It is not built
with the transition. It is a chunk of its own, minted by a person rather than decomposed from this spec, no sooner than
one release after the transition release ships and only once the hosted hub has recorded no `config-old-field-used`
event through a whole release. Because it removes request fields and the old CLI flags, its commit is marked breaking,
`feat!` (`docs/versioning.md`), and its release notes name each removed field with what replaces it. The event joins the
closed `EventLogKind` and its severity map (`foundation/event_log.py`) as a warning, and is removed with the fields it
reports.

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
configuration document written before the epic applies to it unchanged, with a warning per old field, and its export
applies cleanly after retirement. A store holding a feature environment's `http://127.0.0.1:<port>` origin carries over
with its port; one holding two spellings of one coordinate, retired or not, is refused naming both, and carries over
once one of them is edited; one holding an address with a user in it, or a work-source address with no scheme, is
refused naming the record; and one whose work source stores an empty `api_base` carries over onto the `github` forge.
The downgrade walks a carried-over store back to the shape it came from, its addresses canonical. Every case in
`tests/fixtures/canonical_origin_cases.json` answers alike from `canonical_origin` and from the revision's frozen copy.
The pre-flight check names, on a store the migration would refuse, the same records the migration names. Beyond the
carry-over:

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
