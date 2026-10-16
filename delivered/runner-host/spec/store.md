# Host store contract

## One database per host

A host keeps exactly one store, `sqlite:///<host dir>/data/host.db`, holding every runner it runs. The runner store's
schema (`runner/store/schema.py`) becomes the host store's: one migration chain, continued from today's head
(`runner/store/migrations/versions/`), so a runner's existing `runner.db` migrates forward into a host store holding one
runner ([carry-over.md](./carry-over.md) §Graduation).

A host serves one tenant, so the store still carries no tenant column, as `epic:multi-tenancy/runner` settled
([plan](../../multi-tenancy/runner.md)). What it gains is a runner column.

## Two revisions

The runner column arrives in two steps, because the runners that build the host are also the first to run it, and every
redeploy migrates a store while the previous release's daemon is still running on it ([carry-over.md](./carry-over.md)
§How the host reaches a running fleet).

The **additive revision** lands first, ahead of the host. It adds a nullable `runner_id` to every runner-scoped table
and fills it on existing rows from the store's own `runner_identity` row; it adds the new host-scoped tables, and
`env_bindings`' nullable `workspace_id` and `project_id` ([workspaces.md](./workspaces.md) §Routing a claim). It changes
no constraint, index, or column the previous release writes through, so that release keeps writing safely between the
migration and its restart, its new rows simply carrying no runner. The runner it migrates is still a lone runner reading
`blizzard-runner.toml`; from this revision on it binds its view of the store to its own id (§Scoped access) and, before
it serves a request or ticks, stamps that id on every runner-scoped row that has none — every such row is its own, since
the store has only ever held one runner. A lone runner whose store holds no identity yet binds at its first successful
contact with the hub, and writes no runner-scoped row before it.

The **tightening revision** ships with the host itself and is applied at graduation, while the runner is stopped. It
stamps any row still lacking a runner from `runner_identity`, refusing a store that holds runner-scoped rows but no
identity rather than guessing; then makes `runner_id` `NOT NULL`, leading every index and every uniqueness constraint
that identifies a row within a runner, and moves the outboxes onto per-runner counters (§Runner-scoped and host-scoped
tables) — the table rebuilds SQLite needs for that, and a counter seeded while nothing else can write, are why it waits
for a stopped runner.

Both revisions downgrade: the additive one by dropping what it added, the tightening one by restoring today's
constraints, refusing while the store holds more than one runner and naming them, as the hub's tenancy downgrade refuses
over several tenants.

## Runner-scoped and host-scoped tables

Every table is one of two kinds, declared on the table (`info={"scope": "runner" | "host"}`) and enforced by a schema
self-test, so a new table cannot be added without choosing. A table is runner-scoped unless it is named host-scoped
below.

- **Host-scoped** tables carry no runner, because what they hold belongs to the machine or is the same for every runner
  on it: `daemon_liveness` (`schema.py:390`) — one process, one liveness stamp — `jwt_jti_seen` (`:633`),
  `workspace_prompt` (`:379`), keyed by workspace id, `external_usage_samples` (`:680`), `credential_renewal_claims` and
  `credential_renewal_outcomes` (`:698`, `:712`), `graph_artifacts` (`:833`) — a minted graph's immutable artifacts,
  stored once however many runners pin it — and four new ones: `host_identity`, a singleton holding the host id and its
  tenant once the host is registered ([process.md](./process.md) §One tenant); `host_outbound_buffer`, the host's own
  fact lane ([hub.md](./hub.md) §The host's fact lane); `host_high_water_seen`, the last mark the hub answered on that
  lane; and `host_ingests`, the record of each runner ingested into the host, which lets an interrupted ingest resume
  ([carry-over.md](./carry-over.md) §Ingest).
- **Runner-scoped** tables gain `runner_id`, the hub-minted id. That is every other table: every lease, binding, ask,
  park, pause, takeover, requeue, escalation, usage, check, nudge, attachment, transcript, elicitation, self-test,
  overload, context-sample, trace, and outbox table, and the runner's identity and hub answers. What is one row today
  becomes one row per runner: `runner_identity` (`:331`) and `hub_control` (`:319`); `hub_served_projects` (`:350`), one
  row per declared entry, becomes unique on `(runner_id, position)`. A table keyed by a natural key takes the runner
  into it, since one chunk can be held by two runners of one host over its life: `route_tokens` (`:480`) is keyed by
  `(runner_id, chunk_id)`.

The two outboxes, `outbound_buffer` (`:61`) and `transcript_outbound_buffer` (`:808`), count per runner, since the hub
keeps a high-water mark per runner on each lane. Their `seq` becomes unique on `(runner_id, seq)`, and a runner's next
value comes from a durable counter of its own per lane, `outbox_counters`, advanced in the transaction that writes the
row; SQLite's autoincrement, which can only count per table, gives way to it in the tightening revision, which seeds
each counter from the sequence SQLite kept (`sqlite_sequence`). A counter never moves backwards — not when acknowledged
rows are pruned, and not when a runner's rows are reset, which leaves its counters in place — so no runner's new fact
can land at or below the mark the hub already holds for it. Each runner's retained `seq`s stay gapless from its own
pending floor upward, which the invariant checker's outbox checks (`GaplessOutboundSeq`, `GaplessTranscriptOutboundSeq`
in `tools/invariants.py`) assert per runner rather than per table. Ingest carries a runner's rows with their `seq`s
unchanged and its counters with them ([carry-over.md](./carry-over.md) §Ingest).

## Scoped access

`RunnerStores` (`runner/composition.py:389`) is built once per runner, bound to its runner id — the `id` its
`[[runner]]` entry carries ([configuration.md](./configuration.md) §Hub credentials). Every runner-scoped adapter reads
and writes through scoped statement builders that add `runner_id = :bound` for every runner-scoped table a statement
names — in its joins, aliases, and correlated subqueries too, such as the held-binding and live-escalation predicates
(`runner/store/internal/base.py`) — and stamp it on every insert: the runner's counterpart of the hub's tenant scoping
(`blizzard-context:/architecture/system-shape/tenancy.md`), so no query can forget its runner. As with the tenant
boundary, a statement guard enabled in the test suite and off in production (the hub's is `hub/store/tenant_guard.py`)
fails any statement naming a runner-scoped table without that equality, and any insert into one without `runner_id`.

Host-scoped tables are reached only through a separate `HostStores`, built once per host. The few reads that span
runners — the environment pool's held set, the brake fan-out, and the invariant checker — are named members of
`HostStores`, marked exempt from the guard, never a runner adapter with its filter removed.

## Writers

Every runner's loop, each runner's API requests, and the host's own pass write the one file. The engine already runs
SQLite in WAL with a five-second busy timeout (`foundation/store/engine.py:36`), and each keeps its transactions as
short as a runner's are today — no transaction is ever held open across work that leaves the process, since every other
writer on the host waits behind it.

Two writes must not interleave, and each is serialised in the way its shape allows. Acquiring an environment holds a
lock per workspace, in the host's one process, across reading the held set, the provider's choice and preparation, and
recording the binding — never a database transaction, since preparation can take minutes
([workspaces.md](./workspaces.md) §The shared pool). Engaging the brake fan-out is one write transaction whose first
statement takes the store's writer lock, in the keyed-lock-row form the hub's exclusive writes use
(`bzh:store-exclusive-write`), before it reads which runners are already braked ([subscriptions.md](./subscriptions.md)
§The brake).

## Integer keys and references

Forty-two tables key their rows by an autoincrement integer, and a few rows name rows of another table by those integers
without a declared foreign key — a pause park names the elicitation it interrupted
(`pause_parks.interrupted_elicitation_id`), a renewal outcome the claim it settles
(`credential_renewal_outcomes.claim_id`). Within one store that is fine; it is what ingest must re-number when a
runner's rows move into another host's store ([carry-over.md](./carry-over.md) §Ingest). Every column holding another
table's surrogate key declares it on the column (`info={"refers_to": "in_flight_elicitations.id"}`), and the schema
self-test fails any integer column named like a key that declares neither a reference nor that it is not one. That
declaration is the ingest map; nothing else lists it.

## Invariants

The host's store makes two invariants single-database facts. The invariant checker (`tools/invariants.py`) opens the
host store and asserts them alongside its existing checks:

- an environment of a workspace is held by at most one binding across every runner of the host;
- every runner-scoped row names a runner whose `runner_identity` row exists in the store.

## Resetting one runner

`blizzard runner store reset --runner <name|id>` still never discards a runner's data. It extracts the runner's rows
into a timestamped archive store beside the host's — the same copy ingest performs, in the other direction — and deletes
them from the host store, refusing while the host's lock is held. Its outbox counters are copied into the archive but
stay in the host store, so a runner reset and run again never counts below a mark the hub already holds. A retired
runner's rows wait in the store for exactly this. Resetting the whole host moves `host.db` aside, as
`runner store reset` moves a runner's store aside today (`runner/runtime.py:161`).
