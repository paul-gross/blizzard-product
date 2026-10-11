# Host store contract

## One database per host

A host keeps exactly one store, `sqlite:///<host dir>/data/host.db`, holding every runner it runs. The runner store's
schema (`runner/store/schema.py`) becomes the host store's: one migration chain, continued from today's head
(`runner/store/migrations/versions/`), so a runner's existing `runner.db` migrates forward into a host store holding one
runner ([carry-over.md](./carry-over.md) §Graduation).

A host serves one tenant, so the store still carries no tenant column, as the multi-tenancy runner slice settled
([runner slice](../../../delivered/multi-tenancy/runner.md)). What it gains is a runner column.

## Runner-scoped and host-scoped tables

Every table is one of two kinds, declared on the table (`info={"scope": "runner" | "host"}`) and enforced by a schema
self-test, so a new table cannot be added without choosing.

- **Runner-scoped** tables gain `runner_id` — the hub-minted id — `NOT NULL`, leading every index and every uniqueness
  constraint that identifies a row within a runner. This is every lease, binding, ask, park, pause, takeover, requeue,
  usage, check, attachment, transcript, trace-cursor, and outbox table. The tables that are singletons today become one
  row per runner: `runner_identity` (`schema.py:329`), `hub_control` (`:317`), `hub_served_projects` (`:348`), and the
  outbox sequences of `outbound_buffer` (`:61`) and `transcript_outbound_buffer` (`:806`), whose `seq` becomes unique on
  `(runner_id, seq)` and keeps counting per runner, since the hub's high-water mark is per runner.
- **Host-scoped** tables carry no runner: `daemon_liveness` (`:388`) — one process, one liveness stamp — `jwt_jti_seen`
  (`:631`), `workspace_prompt` (`:377`), `external_usage_samples` (`:678`), `credential_renewal_claims` and
  `credential_renewal_outcomes` (`:696`, `:710`), and three new ones: `host_identity`, a singleton holding the host id
  and its tenant once the host is registered ([process.md](./process.md) §One tenant); `host_outbound_buffer`, the
  host's own fact lane ([hub.md](./hub.md) §The host's fact lane); and `host_high_water_seen`, the last mark the hub
  answered on that lane.

## Scoped access

`RunnerStores` (`runner/composition.py:389`) is built once per runner, bound to its runner id. Every runner-scoped
adapter reads and writes through a connection that adds `runner_id = :bound` to every statement against a runner-scoped
table and stamps it on every insert — the runner's counterpart of the hub's tenant scoping
(`blizzard-context:/architecture/system-shape/tenancy.md`) — so no query can forget its runner. Host-scoped tables are
reached only through a separate `HostStores`, built once per host. The few reads that span runners — the environment
pool's held set, the brake fan-out, and the invariant checker — are named members of `HostStores`, never a runner
adapter with its filter removed.

## Writers

Every runner's loop and the host's own pass write the one file. The engine already runs SQLite in WAL with a five-second
busy timeout (`foundation/store/engine.py:34`); each loop keeps its transactions as short as a runner's are today, and
the two writes that must not interleave — acquiring an environment and engaging the brake fan-out — run as
`BEGIN IMMEDIATE` transactions ([workspaces.md](./workspaces.md) §The shared pool,
[subscriptions.md](./subscriptions.md) §The brake).

## Integer keys and references

Forty-two tables key their rows by an autoincrement integer, and rows of one table name rows of another by those
integers without a declared foreign key — a lease's id appears in its spawns, closures, heartbeats, and tokens. Within
one store that is fine; it is what ingest must re-number when a runner's rows move into another host's store
([carry-over.md](./carry-over.md) §Ingest). Every column holding another table's surrogate key declares it on the column
(`info={"refers_to": "leases.id"}`), and the schema self-test fails any integer column named like a key that declares
neither a reference nor that it is not one. That declaration is the ingest map; nothing else lists it.

## Invariants

The host's store makes two invariants single-database facts. The invariant checker (`tools/invariants.py`) opens the
host store and asserts them alongside its existing checks:

- an environment of a workspace is held by at most one binding across every runner of the host;
- every runner-scoped row names a runner whose `runner_identity` row exists in the store.

## Resetting one runner

`blizzard runner store reset --runner <name|id>` still never discards a runner's data. It extracts the runner's rows
into a timestamped archive store beside the host's — the same copy ingest performs, in the other direction — and deletes
them from the host store, refusing while the host's lock is held. Resetting the whole host moves `host.db` aside, as
`runner store reset` moves a runner's store aside today (`runner/runtime.py:161`).
