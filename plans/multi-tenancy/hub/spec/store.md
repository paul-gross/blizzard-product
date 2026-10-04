# Store contract

The hub store keeps one database, and every tenant-owned row carries its tenant. This contract owns the key, the seam
that applies it, the list of what stays global, the uniqueness rework, the build check, and the migration that puts the
key in place.

## The tenant record

A tenant is identified by its id and known to people by its name. The two never stand in for each other inside the
store: every `tenant_id` column, foreign key, credential binding, cache key, and internal reference holds the id, and
the name appears only where a person reads or types it.

| Table                 | Columns                                                               | Notes                                                                               |
| --------------------- | --------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `tenants`             | `tenant_id` (`ten_<ulid>`, PK), `name`, `created_at`, `created_by`    | `name` unique among live tenants, compared case-insensitively                       |
| `tenant_former_names` | `name` (PK), `tenant_id`, `retired_at`                                | a name a tenant was renamed away from; at most one tenant holds a given former name |
| `tenant_deletions`    | `id`, `tenant_id`, `name`, `deleted_at`, `deleted_by`, `rows_removed` | append-only audit of a teardown; outlives the tenant it names, so carries no FK     |

- **The id** is a ULID with the prefix `ten_`, minted once and never reused.
- **The name** is the tenant's own choice and freely editable: 1–64 characters, no `/`, no leading or trailing
  whitespace, and not beginning with `ten_`, so a path segment is never ambiguous between a name and an id. Slug style
  (`acme-sandbox`) is the convention the board suggests, not a rule.
- **Renaming** updates `tenants.name` and records the old name in `tenant_former_names`, replacing any earlier holder of
  that former name. Nothing that references the tenant changes. A name another live tenant holds is refused `409`;
  taking a name that is some tenant's *former* name is allowed and removes that former-name row.
- **Deleting** a tenant removes its `tenant_former_names` rows with it; its name is free once the deletion completes
  ([administration.md](./administration.md)).

All three tables are global.

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
- **Ids stay hub-unique.** ULID-minted ids (`ch_`, `gr_`, `fin_`, `gprop_`, `wi_`, `qn_`, `dec_`, transcript segment
  ids, …) remain globally unique primary keys. A tenant scope is still applied to every read by id, so a guessed id from
  another tenant resolves to nothing, never to a row (`not found`, not `forbidden`).

## The scoped store seam

The hub builds its stores once today (`hub/composition.py::build_hub_core` over one `HubStoreConnections`). Tenancy adds
a scope between the connections and every adapter.

```python
TenantId = NewType("TenantId", str)
ProjectId = NewType("ProjectId", str)

@dataclass(frozen=True)
class StoreScope:
    tenant: TenantId
    project: ProjectId | None = None   # a lens: narrows reads of project-owned tables only
```

- **`HubStoreConnections.scoped(scope) -> ScopedConnections`.** The only connections object a `hub/store/internal/`
  adapter for a tenant-owned table may hold (`bzh:dependency-injection`). It keeps `read(operation)` and
  `write(operation, expect=…)` and adds the statement builders `select(table, *cols)`, `insert(table)`, `update(table)`,
  `delete(table)`, and `scope_join(left, right, onclause)`. Each builder applies `table.c.tenant_id == scope.tenant` to
  every tenant-owned table it names; `insert` stamps `tenant_id` into every row's values. When `scope.project` is set,
  the same builders add `table.c.project_id == scope.project` to tables `epic:projects` marks project-owned, and leave
  every other table at tenant scope.
- **Stores are opened per scope.** `HubCore.open(scope) -> TenantStores` constructs the tenant's repository set over
  `connections.scoped(scope)`. Adapters are stateless, so the hub caches one `TenantStores` per tenant (the project lens
  is applied per call, not cached) and evicts it when the tenant is deleted. Services built on stores (`build_services`)
  are built from a `TenantStores`, never from the unscoped core.
- **Repository Protocols do not change shape.** `IRead…`/`IWrite…` seams keep their signatures (`bzh:repository-split`,
  `bzh:controller-read-only`): the tenant is a property of the instance a caller was handed, not an argument it passes.
  A controller holding a read repository therefore cannot widen its own scope.
- **Hub-scoped reads are named and few.** A read that must span tenants goes through `HubScopedReads`, an explicitly
  separate seam with a closed list of members:
  - credential and token resolution — runner bearer hash → registration, route token, marker token, session hash — each
    returning the tenant it resolves to beside its principal;
  - the hub sweeps' corpus reads (below);
  - tenant administration (listing tenants, teardown);
  - the identity store.

  Nothing else may hold it. An ast-grep rule (`blizzard:structural-gate`) refuses an import of `HubScopedReads` outside
  `hub/auth/`, `hub/api/auth*.py`, the sweep reconcilers, and tenant administration.

### Sweeps

The hub's sweeps (`hub/app.py::Sweep.all` — annotation, event derivation, work-item materialization, close drain, trace
export) stay one pass per hub, not one per tenant (`bzh:steppable-loop`, `bzh:probe-gated-pass`). A pass reads its
corpus through `HubScopedReads`, and every row it acts on carries its `tenant_id`; each write the pass makes is issued
through `TenantStores` opened for that row's tenant. A probe that gates a pass (`bzh:probe-gated-pass`) stays hub-wide.
A pass never holds one tenant's scope while acting on another tenant's row.

### The fleet-wide hub-execution slot

`hub_exec_slot` serializes hub-executed nodes "one at a time across the fleet". It becomes one live slot per tenant: a
tenant's slow delivery never holds another tenant's chunks. `bzh:store-exclusive-write` holds within the tenant, with
the lock row read through the tenant scope.

## The global list

`schema.py` declares `GLOBAL_TABLES: frozenset[str]`, the complete list of tables that carry no tenant key. Each entry
names why it is global.

| Table                                                | Why global                                                                                               |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tenants`, `tenant_former_names`, `tenant_deletions` | the boundary itself                                                                                      |
| `users`, `identities`, `sessions`, `auth_state`      | identity is resolved before a tenant is ([identity.md](./identity.md))                                   |
| `auth_facts`                                         | the sign-in and security log of hub-level identities                                                     |
| `superuser_bootstrap`                                | names the hub administrator, the role above every tenant                                                 |
| `membership_facts`                                   | carries `tenant_id` as data but is read hub-wide to list a person's tenants                              |
| `hub_event_log`                                      | hub-level occurrences that concern no tenant (startup, migration check, teardown)                        |
| `trace_cursor`                                       | trace export is hub configuration with one destination; spans carry the tenant's id as `blizzard.tenant` |

Outside the store and not tables: Alembic's `alembic_version`, the packaged system artifacts served from the wheel
(`/api/system-artifacts`), and the health and readiness routes. Packaged graphs are *not* global — they are minted into
each tenant ([administration.md](./administration.md)).

Every other table — 74 at this writing, including every chunk, fact, graph, garden, transcript, work-item, runner, and
event-log table — is tenant-owned. `event_log` keeps only tenant occurrences; an occurrence with no tenant goes to
`hub_event_log`.

## Uniqueness, rescoped

Every uniqueness rule a tenant can observe becomes unique per tenant. Rules keyed on a hub-unique ULID are unchanged.

| Table                      | Today                                                                                                          | Becomes                                                                                                        |
| -------------------------- | -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| `graphs`                   | `ix_graphs_name (name)`, name resolution hub-wide                                                              | `ix_graphs_tenant_name (tenant_id, name)`; resolution within scope                                             |
| `scopes`                   | PK `slug`; FKs from `scope_lifecycle_facts`, `routines.default_scope_slug`, `routine_scopes`, `work_item_runs` | surrogate PK `scope_id` (`scp_<ulid>`); `uq_scopes_tenant_slug (tenant_id, slug)`; FKs repointed to `scope_id` |
| `routines`                 | `uq_routines_name (name)`                                                                                      | `uq_routines_tenant_name (tenant_id, name)`                                                                    |
| `work_items`               | `uq_work_items_source_ref (source, ref)`                                                                       | `(tenant_id, source, ref)`                                                                                     |
| `work_item_sequence`       | PK `source`                                                                                                    | PK `(tenant_id, source)` — each tenant's built-in `hub` source numbers from 1                                  |
| `garden_proposal_closures` | `ix_garden_proposal_closures_source_ref (source, ref)` unique                                                  | `(tenant_id, source, ref)` unique                                                                              |
| `chunk_work_refs`          | `ix_chunk_work_refs_source_ref (source, ref)`                                                                  | `(tenant_id, source, ref)`                                                                                     |
| `runner_registrations`     | PK `runner_id`                                                                                                 | unchanged — runner ids are hub-minted (below)                                                                  |

`scopes` takes a surrogate key because its natural key is a primary key four foreign keys point at; rescoping a natural
primary key would force composite foreign keys onto every child, and `epic:projects` rescopes the slug again. With a
surrogate, each later rescoping is a change to one unique constraint. Columns that carry a scope slug or routine name as
a denormalized label (`findings.scope_slug`, `finding_sets`, `work_items.routine_name`, …) keep their label and resolve
it within the row's tenant.

**Runner ids are hub-minted.** `runner_id` is the primary key of `runner_registrations` and the key of the runner
high-water, pause, usage, and transcript tables. It is an `rn_<ulid>` the hub mints when a runner is added, so it is
unique across every tenant by construction and there is no collision to refuse. A runner's tenant is the one it was
added in. Its name is display only and unique nowhere, so two runners may share one, in one tenant or across several.
Tenants created for tests add their runners like any other.

Identity uniqueness — `users.username`, `uq_users_email`, `uq_identities_provider_subject` — stays hub-wide.

## The build check

A test over `schema.metadata` fails the build when any table either:

- carries no `tenant_id` column and is not in `GLOBAL_TABLES`;
- carries `tenant_id` nullable, without a foreign key to `tenants`, or while also listed in `GLOBAL_TABLES`
  (`membership_facts` excepted, by name);
- has a unique constraint or unique index that names a non-ULID column without naming `tenant_id`.

A second guard runs in the test suite: a SQLAlchemy `before_execute` listener on the test engine walks every compiled
`Select`, `Update`, and `Delete` and fails the test when a tenant-owned table appears in its `FROM` without a
`tenant_id` equality in its `WHERE` or join `ON`, and every `Insert` into a tenant-owned table whose values lack
`tenant_id`. `HubScopedReads` statements are marked and exempt. The listener ships disabled in production.

## Migration shape

Per `bzh:manual-migrations` and `bzh:frozen-revisions`, as a sequence of revisions in the hub tree, each with a real
`downgrade()`:

1. **Create** `tenants`, `tenant_former_names`, `tenant_deletions`, `membership_facts`, `hub_event_log`, and insert the
   carried-over tenant ([carry-over.md](./carry-over.md)).
2. **Add** `tenant_id` as nullable to every tenant-owned table, then **backfill** it with the carried-over tenant in
   batches of bounded size per table.
3. **Tighten**: `NOT NULL`, the foreign key to `tenants`, and the rebuilt indexes and unique constraints above. On
   SQLite each table is recreated through `batch_alter_table`'s copy — load-bearing for `NOT NULL` and the foreign key,
   not a style choice. Each table is its own batch so a failure leaves the store at a known revision.
4. **Rescope** `scopes` to its surrogate key and repoint its four foreign keys.
5. **Move** identity: copy each `users.role` into a membership, then drop `users.role` ([identity.md](./identity.md)).

`downgrade()` of revisions 2–5 refuses while more than one tenant exists, naming the tenants to delete first; with one
tenant it reverses exactly.

The table copy dominates on SQLite: the transcript tables are the largest, and the step is a full rewrite of each. The
migration runs offline under `blizzard hub migrate`, as every hub migration does; its expected duration on the dogfood
store is measured and stated in the release note that ships it.

## SQLite and Postgres

- **One code path.** Every statement the scoped builders produce stays inside `bzh:sql-portable`'s surface; nothing in
  this contract is dialect-specific.
- **Postgres row-level security** is not part of this slice. The seam is the boundary on both backends; whether Postgres
  additionally enforces `tenant_id = current_setting('blizzard.tenant')` is the slice plan's open question, and nothing
  here precludes it.
- **Write contention.** All tenants on one SQLite hub share one writer: WAL mode with `busy_timeout=5000`
  (`foundation/store/engine.py`) queues concurrent writers rather than failing them. A tenant's write latency therefore
  includes every other tenant's writes. This slice changes nothing about that; `epic:test-shared-service` measures how
  many concurrent tenants a SQLite hub carries before it promises a number, and the hub's own fleet is unaffected while
  it holds one tenant.
