# Store contract

The hub store keeps one database, and every tenant-owned row carries its tenant. This contract owns the key, the seam
that applies it, the list of what stays global, the uniqueness rework, the build check, and the migration that puts the
key in place.

## The tenant record

A tenant is identified by its id and known to people by its name. The two never stand in for each other: every
`tenant_id` column, foreign key, credential binding, cache key, route, header, and command holds the id, and the name
appears only where a person reads it.

| Table              | Columns                                                               | Notes                                                                           |
| ------------------ | --------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| `tenants`          | `tenant_id` (`ten_<ulid>`, PK), `name`, `created_at`, `created_by`    | `name` is a display label, unique nowhere                                       |
| `tenant_deletions` | `id`, `tenant_id`, `name`, `deleted_at`, `deleted_by`, `rows_removed` | append-only audit of a teardown; outlives the tenant it names, so carries no FK |

- **The id** is a ULID with the prefix `ten_`, minted once and never reused. It is the only way anything names a tenant.
- **The name** is a label for people to read: 1–64 characters, no leading or trailing whitespace. It is unique nowhere,
  so two tenants may share one, and nothing resolves it — no route, header, query parameter, or command accepts a name
  in place of an id.
- **Renaming** updates `tenants.name` and nothing else. Every link, script, and saved context holds the id, so a rename
  breaks nothing and leaves nothing to redirect.

Both tables are global.

## The tenant key

Every tenant-owned table gains `tenant_id String NOT NULL REFERENCES tenants(tenant_id)`.

- **Every row carries it directly**, child rows included. A fact row under a chunk does not find its tenant through the
  chunk: the filter has to be expressible on the table a statement actually reads, and a join-derived tenant is exactly
  the predicate a hand-written query forgets.
- **It is stamped by the seam, never by the caller** (below). No domain type, request model, or wire schema carries a
  `tenant_id` field into a write.
- **Indexes lead with it where a read lists.** An index that serves a listing or a range read — every
  `ix_*_recorded_at_*`, `ix_*_at_id`, `ix_chunks_minted_at_chunk_id`, `ix_event_log_recorded_at`, the
  `(routine_name, scope_slug)` and `(source, ref)` families — is rebuilt with `tenant_id` as its leading column, so a
  tenant's feed never scans another tenant's rows. An index whose leading column is a ULID primary or foreign key
  (`*_id` lookups by chunk, finding, proposal, segment) keeps its shape: the id already selects one tenant's rows.
- **Every tenant-owned table can find a tenant's rows by index.** Each has at least one index whose leading column is
  `tenant_id` — a listing index rebuilt as above, or a plain `ix_<table>_tenant_id` where none already leads with it —
  so draining a tenant ([administration.md](./administration.md) §Deleting a tenant) touches that tenant's rows and
  never scans another's.
- **Ids the hub mints stay hub-unique.** ULID ids the hub mints (`ch_`, `gr_`, `fin_`, `gprop_`, `wi_`, `dec_`, …)
  remain globally unique primary keys. A tenant scope is still applied to every read by id, so a guessed id from another
  tenant resolves to nothing, never to a row (`not found`, not `forbidden`).
- **Ids a runner mints are unique per tenant.** A question's `qn_` id and a transcript segment's id are minted on the
  runner and travel the wire both ways — a runner asks, then polls for the answer by that id — so the hub keys its rows
  on `(tenant_id, <the runner's id>)` rather than minting an id of its own. An id reused in another tenant is simply a
  different row, and a collision reveals nothing. These columns are not marked hub-unique, so the build check holds
  every key built on them to carrying `tenant_id` (§Uniqueness, rescoped).

## The scoped store seam

The hub builds its stores once today (`hub/composition.py::build_hub_core` over one `HubStoreConnections`). Tenancy adds
a scope between the connections and every adapter.

```python
TenantId = NewType("TenantId", str)

@dataclass(frozen=True)
class StoreScope:
    tenant: TenantId
```

- **`HubStoreConnections.scoped(scope) -> ScopedConnections`.** The only connections object a `hub/store/internal/`
  adapter for a tenant-owned table may hold (`bzh:dependency-injection`). It keeps `read(operation)` and
  `write(operation, expect=…)` and adds the statement builders `select(table, *cols)`, `insert(table)`, `update(table)`,
  `delete(table)`, and `scope_join(left, right, onclause)`. Each builder applies `table.c.tenant_id == scope.tenant` to
  every tenant-owned table it names; `insert` stamps `tenant_id` into every row's values. The scope is the tenant and
  nothing narrower: a filter within a tenant, such as `epic:projects`' project filter, is an ordinary argument of the
  read that applies it, never a property of the scope.
- **Stores are opened per scope.** `HubCore.open(scope) -> TenantStores` constructs the tenant's repository set over
  `connections.scoped(scope)`. Adapters are stateless, so the hub caches one `TenantStores` per tenant and evicts it
  when the tenant is deleted. Services built on stores (`build_services`) are built from a `TenantStores`, never from
  the unscoped core.
- **Repository Protocols do not change shape.** `IRead…`/`IWrite…` seams keep their signatures (`bzh:repository-split`,
  `bzh:controller-read-only`): the tenant is a property of the instance a caller was handed, not an argument it passes.
  A controller holding a read repository therefore cannot widen its own scope.
- **Hub-scoped reads are named and few.** A read that must span tenants goes through `HubScopedReads`, an explicitly
  separate seam with a closed list of members:
  - credential and token resolution — runner bearer hash → registration, route token, session hash, invitation token →
    invitation — each returning the tenant it resolves to beside its principal;
  - the hub sweeps' corpus reads (below);
  - tenant administration (listing tenants, teardown);
  - the identity store.

  Nothing else may hold it. An ast-grep rule (`blizzard:structural-gate`) refuses an import of `HubScopedReads` outside
  `hub/auth/`, `hub/api/auth*.py`, the sweep reconcilers, and tenant administration.

### Sweeps

Every sweep the hub runs — each one `hub/app.py::Sweep.all` yields (forge-status annotation, event derivation, work-item
materialization, close drain, trace export, fact egress) and the `tenant_teardown` sweep this slice adds — stays one
pass per hub, not one per tenant (`bzh:steppable-loop`, `bzh:probe-gated-pass`). A pass reads its corpus through
`HubScopedReads`, and every row it acts on carries its `tenant_id`; each write the pass makes is issued through
`TenantStores` opened for that row's tenant. A probe that gates a pass (`bzh:probe-gated-pass`) stays hub-wide. A pass
never holds one tenant's scope while acting on another tenant's row.

A pass never acts on a closed tenant — one with a `tenant_deletions` fact, whether its drain is still running or done.
The corpus reads leave closed tenants out, and the seam backs that up for every writer, sweep or not:
`HubCore.open(scope)` refuses a closed tenant, and `ScopedConnections.write` confirms inside its own transaction that
the scope's tenant is still open, raising `TenantClosed` when it is not. A hub-executed node or a request that opened
its stores before the close therefore cannot write a row into a table the drain has already passed. Reads are not
checked; a read of a draining tenant sees a shrinking set and writes nothing.

### Fact egress

The `epic:fact-egress` export stays hub-wide, as trace export does. Its `[egress]` directory is hub configuration with
one destination, chosen by whoever runs the hub, so the egress sweep keeps one cursor per dataset across every tenant
(`egress_cursor` is global) and writes every tenant's rows into the same tree, partitions, and manifests it writes
today.

- **Every row names its tenant.** Each row of every dataset — `steps`, `invocations`, and `events`, a `dropped` row
  included — gains `tenant_id`, and beside it `tenant_name` as the tenant was named when the row was written, the way
  the rows already pair `graph_id` with `graph_name`. The column is additive; the contract version moves with its
  goldens and the data dictionary. A loader that wants one tenant filters on `tenant_id`, and `epic:tenant-telemetry`
  routes each tenant's rows by it later.
- **One pass across tenants.** The sweep reads closed steps, usage facts, derivation markers, and drops through
  `HubScopedReads`, in its existing cursor order across every open tenant, and writes nothing into any tenant's store.
- **Files written before the upgrade** lack the column. Every row in them belongs to the carried-over tenant, which the
  data dictionary says; `egress backfill` rewrites any window with the column present.
- **Operating it is the hub administrator's.** `egress status`, `reset`, and `backfill` read or move a cursor that every
  tenant's rows share, so their routes join the hub-level family ([api.md](./api.md) §Route families) under
  `EXPORT_ADMIN` ([identity.md](./identity.md) §Permissions), in place of today's `FLEET_VIEW` and `ANALYTICS_ADMIN`.
- **Its occurrences are the hub's.** `egress-config-rejected`, `egress-write-failed`, `egress-write-recovered`, and
  `egress-cursor-reset` are written to `hub_event_log`, never to a tenant's `event_log`.
- **Deleting a tenant stops at the directory.** Teardown removes a tenant's rows from the store, never from a file
  already written, because blizzard never touches a written file. The `tenant_deletions` fact keeps the tenant's id,
  which is what an operator prunes the exported rows by.

### The fleet-wide hub-execution slot

`hub_exec_slot` serializes hub-executed nodes "one at a time across the fleet". It becomes one live slot per tenant: a
tenant's slow delivery never holds another tenant's chunks. `bzh:store-exclusive-write` holds within the tenant, with
the lock row read through the tenant scope.

## The global list

`schema.py` declares `GLOBAL_TABLES: frozenset[str]`, the complete list of tables that carry no tenant key. Each entry
names why it is global.

| Table                                           | Why global                                                                                       |
| ----------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `tenants`, `tenant_deletions`                   | the boundary itself                                                                              |
| `users`, `identities`, `sessions`, `auth_state` | identity is resolved before a tenant is ([identity.md](./identity.md))                           |
| `auth_facts`                                    | the sign-in and security log of hub-level identities                                             |
| `superuser_bootstrap`                           | the first claim of the hub administrator role, from `auth.superuser`                             |
| `hub_admin_facts`                               | who holds the hub administrator role, the role above every tenant ([identity.md](./identity.md)) |
| `config_import_facts`                           | the hub-level carry-over of a file-configured hub, read by the startup legacy check              |
| `membership_facts`                              | carries `tenant_id` as data but is read hub-wide to list a person's tenants                      |
| `hub_event_log`                                 | hub-level occurrences that concern no tenant (§The hub's own event log)                          |
| `trace_cursor`                                  | trace export is hub configuration with one destination; request spans carry `blizzard.tenant`    |
| `egress_cursor`                                 | fact egress is hub configuration with one destination; rows carry the tenant's id (§Fact egress) |

Outside the store and not tables: Alembic's `alembic_version`, the packaged system artifacts served from the wheel
(`/api/fleet/system-artifacts`), and the health and readiness routes. Packaged graphs are *not* global — they are minted
into each tenant ([administration.md](./administration.md)).

Every other table is tenant-owned: 84 of the 93 tables in today's schema, including every chunk, fact, graph, garden,
transcript, work-item, runner, and event-log table, plus the `invitations` and `invitation_facts` tables this slice
adds.

### The hub's own event log

`event_log` keeps only tenant occurrences; an occurrence that concerns no tenant goes to `hub_event_log`. That is every
occurrence of the hub's own machinery:

- the exports': `trace-export-failed`, `trace-export-recovered`, `trace-window-skipped`, `trace-config-rejected`,
  `egress-write-failed`, `egress-write-recovered`, `egress-config-rejected`, and `egress-cursor-reset`
  (`foundation/event_log.py`);
- this slice's: startup, the migration check, and each tenant teardown's close and completion.

The carry-over moves every existing `event_log` row of those kinds into `hub_event_log`, so no tenant inherits the hub's
history. Only hub administrators read it, through the hub-level `GET /api/admin/events`; no tenant member, a tenant's
admins included, ever sees it.

## Uniqueness, rescoped

Every uniqueness rule a tenant can observe becomes unique per tenant. Rules keyed on a hub-unique ULID are unchanged.

| Table                           | Today                                                                                                                                      | Becomes                                                                                                                |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `scopes`                        | PK `slug`; FKs from `scope_lifecycle_facts`, `routines.default_scope_slug`, `routine_scopes`, `work_item_runs`, `findings`, `finding_sets` | surrogate PK `scope_id` (`scp_<ulid>`); `uq_scopes_tenant_slug (tenant_id, slug)`; all six FKs repointed to `scope_id` |
| `secrets`                       | PK `name`; FKs from `secret_lifecycle_facts.name`, `work_sources.secret_name`, `repositories.secret_name`                                  | PK `(tenant_id, name)`; the three FKs become `(tenant_id, …)`                                                          |
| `work_sources`                  | PK `name`; `uq_work_sources_provider_locator (provider, locator)`; FK from `work_source_lifecycle_facts.name`                              | PK `(tenant_id, name)`; `(tenant_id, provider, locator)`; the FK becomes `(tenant_id, name)`                           |
| `repositories`                  | PK `name`; `uq_repositories_coordinate (forge_api_url, owner, repo)`; FK from `repository_lifecycle_facts.name`                            | PK `(tenant_id, name)`; `(tenant_id, forge_api_url, owner, repo)`; the FK becomes `(tenant_id, name)`                  |
| `keyed_locks`                   | PK `(namespace, key)` — keys are graph names and work refs                                                                                 | PK `(tenant_id, namespace, key)`                                                                                       |
| `routines`                      | `uq_routines_name (name)`                                                                                                                  | `uq_routines_tenant_name (tenant_id, name)`                                                                            |
| `work_items`                    | `uq_work_items_source_ref (source, ref)`                                                                                                   | `(tenant_id, source, ref)`                                                                                             |
| `work_item_sequence`            | PK `source`                                                                                                                                | PK `(tenant_id, source)` — each tenant's built-in `hub` source numbers from 1                                          |
| `garden_proposal_closures`      | `ix_garden_proposal_closures_source_ref (source, ref)` unique                                                                              | `(tenant_id, source, ref)` unique                                                                                      |
| `runner_registrations`          | PK `runner_id`                                                                                                                             | unchanged — runner ids are hub-minted (below)                                                                          |
| `questions`, `question_answers` | PK `question_id` — minted by the runner                                                                                                    | PK `(tenant_id, question_id)`                                                                                          |
| `transcript_segments`           | `uq_transcript_segments_segment_turn_start (segment_id, turn_range_start)`                                                                 | `(tenant_id, segment_id, turn_range_start)`                                                                            |
| `transcript_events`             | `uq_transcript_events_natural_key (segment_id, extractor_version, kind, turn_path, occurrence)`                                            | the same, led by `tenant_id`                                                                                           |
| `transcript_event_derivations`  | PK `(segment_id, extractor_version)`                                                                                                       | PK `(tenant_id, segment_id, extractor_version)`                                                                        |

`scopes` takes a surrogate key because its natural key is a primary key six foreign keys point at; rescoping a natural
primary key would force composite foreign keys onto every child, and `epic:projects` rescopes the slug again. With a
surrogate, each later rescoping is a change to one unique constraint. Each of the six children gains `scope_id` as its
foreign key in place of the slug; `findings` and `finding_sets` keep `scope_slug` beside it as the label their reads key
on. Columns that carry a routine name as a denormalized label (`findings.routine_name`, `work_items.routine_name`, …)
keep it and resolve it within the row's tenant.

The live-config records — `secrets`, `work_sources`, `repositories` — take a composite key instead. Their names are the
handle every reference already holds: a work ref is `{source, ref}`, a record names its secret by `secret_name`, and
`epic:live-config` made each name immutable. Nothing rescopes them past the tenant, because `epic:projects` links these
records rather than owning them, so `(tenant_id, name)` is their final key, and every child already carries the
`tenant_id` a composite foreign key needs. Two tenants may each hold a source, repository, or secret of the same name,
and each may configure the same forge repository.

**Secrets keep their sealing.** Every tenant's secrets stay sealed under the one hub key, and each ciphertext's
associated data stays `(name, revision)`; the tenant id is not added to it. Adding it would bind a ciphertext to its
tenant, but would also force every existing secret to be re-sealed at carry-over, and the only case it guards — a sealed
row copied into another tenant's row of the same name — is a write that bypassed the scoped seam, which the build check
and the test-suite listener exist to catch. So nothing is re-sealed, the migration never needs the hub key, and a
secret's isolation rests on the seam like every other row's. This supersedes the associated-data step `epic:live-config`
anticipated ([secrets.md](../../../../delivered/live-config/spec/secrets.md) §Toward tenancy). Key rotation stays
hub-wide and re-seals every tenant's secrets in one pass.

`keyed_locks` is keyed by the tenant because its keys are names a tenant chooses. Under the old key, two tenants locking
the same graph name or work ref would share one row; the second tenant's insert would lose to the first's and its
tenant-scoped `UPDATE` would then lock nothing, so its write would run unserialized.

**Runner ids are hub-minted.** `runner_id` is the primary key of `runner_registrations` and the key of the runner
high-water, pause, usage, and transcript tables. It is an `rn_<ulid>` the hub mints when a runner is added, so it is
unique across every tenant by construction and there is no collision to refuse. A runner's tenant is the one it was
added in. Its name is display only and unique nowhere, so two runners may share one, in one tenant or across several.
Tenants created for tests add their runners like any other.

Two name-keyed indexes are lookups rather than uniqueness rules, and stay non-unique: `ix_graphs_name (name)` is rebuilt
as `ix_graphs_tenant_name (tenant_id, name)`, so graph name resolution runs within the tenant, and
`ix_chunk_work_refs_source_ref (source, ref)` as `(tenant_id, source, ref)`.

Identity uniqueness — `users.username`, `uq_users_email`, `uq_identities_provider_subject` — stays hub-wide.

## The build check

A test over `schema.metadata` fails the build when any table either:

- carries no `tenant_id` column and is not in `GLOBAL_TABLES`;
- carries `tenant_id` nullable, without a foreign key to `tenants`, or while also listed in `GLOBAL_TABLES`
  (`membership_facts` and `tenant_deletions` excepted, by name);
- is tenant-owned and has a primary key, unique constraint, or unique index holding neither `tenant_id` nor a column the
  schema marks hub-unique (`info={"hub_unique": True}`) — a hub-minted ULID or an autoincrement surrogate, the table's
  own or a foreign key to one. A key that holds such a column is already unique within one tenant, because the row that
  column names belongs to exactly one; a key that holds neither is a natural key two tenants could both claim;
- is tenant-owned and has no index whose leading column is `tenant_id`.

A second guard runs in the test suite: a SQLAlchemy `before_execute` listener on the test engine walks every compiled
`Select`, `Update`, and `Delete` and fails the test when a tenant-owned table appears in its `FROM` without a
`tenant_id` equality in its `WHERE` or join `ON`, and every `Insert` into a tenant-owned table whose values lack
`tenant_id`. `HubScopedReads` statements are marked and exempt. The listener ships disabled in production.

## Migration shape

Per `bzh:manual-migrations` and `bzh:frozen-revisions`, as a sequence of revisions in the hub tree, each with a real
`downgrade()`:

1. **Create** `tenants`, `tenant_deletions`, `membership_facts`, `hub_admin_facts`, `hub_event_log`, `invitations`, and
   `invitation_facts`, and insert the carried-over tenant ([carry-over.md](./carry-over.md)).
2. **Add** `tenant_id` as nullable to every tenant-owned table, then **backfill** it with the carried-over tenant in
   batches of bounded size per table.
3. **Tighten**: `NOT NULL`, the foreign key to `tenants`, and the rebuilt indexes and unique constraints above. On
   SQLite each table is recreated through `batch_alter_table`'s copy — load-bearing for `NOT NULL` and the foreign key,
   not a style choice. Each table is its own batch so a failure leaves the store at a known revision.
4. **Rescope** `scopes` to its surrogate key and repoint its six foreign keys.
5. **Move** identity: copy each `users.role` into a membership, record the claimed superuser as the first hub
   administrator, then drop `users.role` ([identity.md](./identity.md)); move every hub-level `event_log` row into
   `hub_event_log` (§The hub's own event log).

`downgrade()` of revisions 2–5 refuses while more than one tenant exists, naming the tenants to delete first; with one
tenant it reverses exactly.

The table copy dominates on SQLite: the transcript tables are the largest, and the step is a full rewrite of each. The
migration runs offline under `blizzard hub migrate`, as every hub migration does; its expected duration on the dogfood
store is measured and stated in the release note that ships it.

## SQLite and Postgres

- **One code path.** Every statement the scoped builders produce stays inside `bzh:sql-portable`'s surface; nothing in
  this contract is dialect-specific.
- **Postgres row-level security** is not part of this slice. The seam is the boundary on both backends, and SQLite stays
  supported, so isolation never depends on the dialect. Postgres additionally enforcing
  `tenant_id = current_setting('blizzard.tenant')` is a later second guard beside the seam, never a replacement for it
  ([slice plan](../index.md) §Left for later); nothing here precludes it.
- **Write contention.** All tenants on one SQLite hub share one writer: WAL mode with `busy_timeout=5000`
  (`foundation/store/engine.py`) queues concurrent writers rather than failing them. A tenant's write latency therefore
  includes every other tenant's writes. This slice changes nothing about that; `epic:test-shared-service` measures how
  many concurrent tenants a SQLite hub carries before it promises a number, and the hub's own fleet is unaffected while
  it holds one tenant.
