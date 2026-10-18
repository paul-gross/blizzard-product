# Carry-over

An installation that predates this epic keeps landing exactly as it did, with every forge it already talks to declared
for it, in one migration and with nothing reconfigured by hand.

## The migration

One manual migration (`bzh:manual-migrations`) under `hub/store/migrations/versions/`, frozen once released
(`bzh:frozen-revisions`) and inside the portable surface (`bzh:sql-portable`), proven on Postgres as on SQLite
(`tests/test_store_migrations.py`) with every constraint and index name inside Postgres's 63 characters
(`tests/test_postgres_identifier_length.py`). Its `down_revision` is the chain's head when it merges — after the actions
epic's revision, since that epic lands first — and it is re-chained if another revision lands first, as the identity and
host revisions were. It follows the identity revisions' shape: every step inspects first and skips what a stopped run
already did, so a migration that dies midway is finished by running it again, and it defines the downgrade that restores
the address columns, deletes the forges' change rows, which an older hub cannot read, and accepts that a work source's
null `api_base` comes back as the address it resolved to. Within every tenant it:

1. **Refuses what it cannot carry.** Before it writes anything, it canonicalizes every origin the tenant's records name
   — each repository's `forge_api_url`, and each GitHub work source's `api_base`, or `https://api.github.com` where that
   is null — under `canonical_origin`'s rule ([record.md](./record.md#the-table)), re-implemented in the revision as the
   slug rule is below, so a port such as a feature environment's `http://127.0.0.1:<port>` is kept. Retired records take
   part like enabled ones, since a retired repository keeps its coordinate claim and a retired work source its locator.
   The migration refuses the upgrade, before any step runs and naming every record involved by slug, as the identity
   revisions refuse a name they cannot carry, in three cases:
   - **An address it would have to clean.** An origin carrying a user or password, a query, or a fragment, which
     `canonical_origin` refuses rather than strips. The operator edits the record's address to the bare origin and runs
     the migration again.
   - **Two records that would become one coordinate.** Two repositories that differ today only in how their origin was
     spelled, such as a trailing slash, would collide under the rebuilt `(tenant_id, forge_id, owner, repo)` constraint,
     and two work sources the same way under `(tenant_id, provider, forge_id, locator)`. Retiring one of the pair does
     not settle it, since a retired record keeps its claim and is never deleted. Editing a retired record is legal, so
     the operator edits one of each pair — usually the retired one — so the two no longer name one coordinate: its
     `repo` or `locator` to a name that is no longer in use, or its address to the server it truly meant.
   - **Two web addresses for one server.** Two GitHub work sources that declare different explicit `web_base`s for one
     canonical origin would leave its forge with no single web address. A work source whose `web_base` is null takes no
     part, since its effective web origin is derived and the forge derives the same one; but where one source on an
     origin declares an explicit `web_base` and another leaves it null, the forge takes the explicit one, and the
     migration's log names the null source whose web links move with it. The operator makes the explicit ones agree.
2. **Creates the forges.** It creates one forge per distinct canonical origin, visiting origins in sorted order so a
   re-run slugs them identically: kind `github`, that `api_base`, and the explicit `web_base` a work source declared for
   that origin, else null. Its `frg_<ulid>` id is minted deterministically, as the identity revisions mint theirs — the
   timestamp from the forge's `created_at`, which is the earliest `created_at` among the records naming its origin, and
   the random bits hashed from the tenant and the origin — so a re-run mints the same id. A forge on `api.github.com` is
   slugged `github` and named `GitHub`; any other takes its web host as its slug under `derive_slug`'s rule
   (`github-acme-internal`), suffixed `-2` and onwards on collision, and the web host itself as its name. The revision
   re-implements the slug rule locally rather than importing it, as the identity revisions' `_derived` and `_suffixed`
   do, so a later change to the rule never changes what a frozen revision did. The operator changes either afterwards
   like any other forge's. Each forge gets revision 1 and a `config_changes` row attributed to `migration`; a forge
   created retired, below, also gets the `forge_lifecycle_facts` row that retires it, since creating a record writes no
   lifecycle fact.
3. **Points every record at its forge.** Each repository's `forge_id` and each work source's `forge_id` is set to the
   forge created for its origin, before both columns are tightened to `NOT NULL` and the old columns dropped. A retired
   record is pointed at its forge like any other, and a forge named only by retired records is created retired.
4. **Rebuilds the two uniqueness constraints**: the repository coordinate as `(tenant_id, forge_id, owner, repo)` and
   the work-source locator as `(tenant_id, provider, forge_id, locator)`.

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

The epic lands in two releases, and it is delivered when the second has.

**The transition release** carries the migration, the forge record and its surfaces, the seam, and the `land` action
ported onto it. The old fields keep working at every door, resolved to forges ([surfaces.md](./surfaces.md#old-fields)),
and only ever to a GitHub forge, the one kind they could name: should the transition outlast the arrival of a second
kind, an old field whose origin carries that kind's API path, GitLab's `/api/v4`, is refused naming the `forge` field to
use instead, never turned silently into a GitHub forge. Nothing an operator wrote before the epic stops working: a
configuration document kept in their repository applies unchanged, and a CLI invocation in a runbook still runs. The
`land` action ships with the hub, so a chunk mid-delivery when the hub restarts resumes on the ported action.

The workspace's feature-environment seeding (`.winter/config/winter-service-tmux/hub-host.sh`) writes a configuration
document naming `api_base` and `forge_api_url` today. It moves to a `forges` section and `forge` references in the same
change that brings the transition release into the workspace's feature environments, and not before, since an older hub
refuses an entry it does not know; once moved, a feature environment never keeps `config-old-field-used` firing.

**The retirement release** removes the old fields from every door, refusing them `422 field_moved_to_forge`, and removes
the old CLI flags. It ships once the `config-old-field-used` event has gone quiet on the instances blizzard runs, and
its release notes name each removed field with what replaces it. The event joins the closed `EventLogKind` and its
severity map (`foundation/event_log.py`) as a warning, and is removed with the fields it reports.

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
once one of them is edited; one holding an address with a user in it is refused naming the record. Beyond the
carry-over:

- A test adapter offering less than every capability is refused before the pass, the chunk escalated with cause
  `forge-capability-missing`, since GitHub, the one kind this epic builds, offers them all.
- The GitHub adapter's unit tests pin every row of the vocabulary and failure tables in [seam.md](./seam.md), including
  `has_hooks`, `draft`, `startup_failure`, an unknown state and conclusion, a secondary rate limit, a merge refused 405,
  a 404 told apart as a missing repository or a missing change, and a GitHub Enterprise Server `/api/v3` origin; a
  further test holds the adapter's offered constant equal to `FORGE_KIND_CAPABILITIES["github"]`. `land`'s tests pin
  every row of [landing.md](./landing.md#what-the-forge-answers).
- Against the mock: a green `unstable` pull request merges, a green `draft` or `has_hooks` one waits, a
  `startup_failure` check waits, a conflicting update routes `conflict`, a rejected token escalates `forge-auth`, and a
  repository the forge does not know escalates `forge-config`.
- An adapter superseded by a forge edit is closed on the action loop, and a pass still holding it finishes its call.
- The garden's commit check resolves a delivered commit through the seam, and degrades to `None` for an unrevealable
  secret, an unavailable forge, and a repository no link resolves.
- Retiring a forge in use answers `409 forge_in_use` listing its referrers; enabling a record whose forge is retired
  answers `409 forge_retired`; `forge_conflict`, `forge_kind_mismatch`, `forge_unavailable` (an unknown forge, and an
  old field whose origin a retired forge holds), and `ForgeOriginTaken` each have a refusal test; two concurrent
  old-field writes for one new origin make one forge; and `hub:forge-reference-standing` and
  `hub:forge-references-in-tenant` each catch a planted breach.
