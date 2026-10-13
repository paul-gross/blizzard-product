# Carry-over

An installation that predates this slice keeps landing exactly as it did, with every forge it already talks to declared
for it, in one migration and with nothing reconfigured by hand.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, frozen once released
(`bzh:frozen-revisions`) and inside the portable surface (`bzh:sql-portable`), proven on Postgres as on SQLite
(`tests/test_store_migrations.py`) with every constraint and index name inside Postgres's 63 characters
(`tests/test_postgres_identifier_length.py`). Its `down_revision` is the chain's head when it merges — today
`20261012_1500_work_source_identity`, the last of the identity revisions — and it is re-chained if another revision
lands first, as the identity and host revisions were. It follows the identity revisions' shape: every step inspects
first and skips what a stopped run already did, so a migration that dies midway is finished by running it again, and it
defines the downgrade that restores the address columns, deletes the forges' change rows, which an older hub cannot
read, and accepts that a work source's null `api_base` comes back as the address it resolved to. Within every tenant it:

1. **Refuses what it cannot carry.** Before it writes anything, it canonicalizes every origin the tenant's records name
   — each repository's `forge_api_url`, and each GitHub work source's `api_base`, or `https://api.github.com` where that
   is null — under `canonical_origin`'s rule ([record.md](./record.md#the-table)), re-implemented in the revision as the
   slug rule is below, so a port such as a feature environment's `http://127.0.0.1:<port>` is kept. Two repositories
   that differ today only in how their origin was spelled, such as a trailing slash, would become one coordinate under
   the rebuilt constraint, and two GitHub work sources that declare different `web_base`s for one origin would leave the
   forge with no single web address. The migration refuses the upgrade naming every such pair, by slug, before any step
   runs, as the identity revisions refuse a name they cannot carry; the operator retires one of each pair, or makes them
   agree, and runs it again.
2. **Creates the forges.** It creates one forge per distinct canonical origin, visiting origins in sorted order so a
   re-run slugs them identically: kind `github`, that `api_base`, and the `web_base` a work source declared for that
   origin, else null. Its `frg_<ulid>` id is minted deterministically, as the identity revisions mint theirs — the
   timestamp from the forge's `created_at`, which is the earliest `created_at` among the records naming its origin, and
   the random bits hashed from the tenant and the origin — so a re-run mints the same id. A forge on `api.github.com` is
   slugged `github` and named `GitHub`; any other takes its web host as its slug under `derive_slug`'s rule
   (`github-acme-internal`), suffixed `-2` and onwards on collision, and the web host itself as its name. The revision
   re-implements the slug rule locally rather than importing it, as the identity revisions' `_derived` and `_suffixed`
   do, so a later change to the rule never changes what a frozen revision did. The operator changes either afterwards
   like any other forge's. Each forge gets revision 1 and a `config_changes` row attributed to `migration`; a forge
   created retired, below, also gets the `forge_lifecycle_facts` row that retires it, since creating a record writes no
   lifecycle fact.
3. **Points every record at its forge.** Each repository's `forge_id` and each GitHub work source's `forge_id` is set to
   the forge created for its origin, before `repositories.forge_id` is tightened to `NOT NULL` and the old columns
   dropped. A retired record is pointed at its forge like any other, and a forge named only by retired records is
   created retired.
4. **Rebuilds the coordinate constraint** as `(tenant_id, forge_id, owner, repo)`.

No repository or work source changes revision, and no `config_changes` row is written for them: the forge they name is
the address they already held.

The legacy import, `blizzard hub config import-legacy` (`hub/domain/config/carry_over.py`), builds repositories from
`BZ_FORGE_URL` and work sources from their `api_base` and `web_base`. It resolves or creates forges exactly as the
migration does, so an installation imported after this slice lands in the same shape as one migrated.

## The transition, then retirement

The slice lands in two releases, and it is delivered when the second has.

**The transition release** carries the migration, the forge record and its surfaces, the seam, and the `land` action
ported onto it. The old fields keep working at every door, resolved to forges ([surfaces.md](./surfaces.md#old-fields)),
and only ever to a GitHub forge, the one kind they could name: should the transition outlast the arrival of a second
kind, an old field whose origin carries that kind's API path, GitLab's `/api/v4`, is refused naming the `forge` field to
use instead, never turned silently into a GitHub forge. Nothing an operator wrote before the slice stops working: a
configuration document kept in their repository applies unchanged, and a CLI invocation in a runbook still runs. The
`land` action ships with the hub, so a chunk mid-delivery when the hub restarts resumes on the ported action.

**The retirement release** removes the old fields from every door, refusing them `422 field_moved_to_forge`, and removes
the old CLI flags. It ships once the `config-old-field-used` event has gone quiet on the instances blizzard runs, and
its release notes name each removed field with what replaces it. The event joins the closed `EventLogKind` and its
severity map (`foundation/event_log.py`) as a warning, and is removed with the fields it reports.

Runners are untouched by both: nothing in this slice crosses the runner's wire.

## Proof

The invariant checker's `hub:forge-reference-standing` passes on a carried-over store. A fixture store captured before
the migration, carried over, serves the same repository and work-source documents with `forge` beside the addresses they
held, resolves every chunk's commits to the same repositories, and lands a chunk against the mock forge through the seam
with the same outcomes and markers the `land` action produced before it. A configuration document written before the
slice applies to it unchanged, with a warning per old field, and its export applies cleanly after retirement. A store
holding a feature environment's `http://127.0.0.1:<port>` origin carries over with its port, and one holding two
spellings of one coordinate is refused naming both. Beyond the carry-over:

- A test adapter offering less than every capability is refused before the pass, the chunk escalated with cause
  `forge-capability-missing`, since GitHub, the one kind this slice builds, offers them all.
- The GitHub adapter's unit tests pin every row of the vocabulary and failure tables in [seam.md](./seam.md), including
  `has_hooks`, a secondary rate limit, a merge refused 405, and a GitHub Enterprise Server `/api/v3` origin; `land`'s
  tests pin every row of [landing.md](./landing.md#what-the-forge-answers).
- Retiring a forge in use answers `409 forge_in_use` listing its referrers; `forge_conflict`, `forge_kind_mismatch`,
  `forge_unavailable`, and `ForgeOriginTaken` each have a refusal test; and `hub:forge-reference-standing` and
  `hub:forge-references-in-tenant` each catch a planted breach.
