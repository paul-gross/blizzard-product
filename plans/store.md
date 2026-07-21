# Plan — `epic:store`

> **Shipped.** This plan is frozen: it records what the runner store was conceived to be, and its entry in [delivered.md](../delivered.md) is the record of what landed.

The morning after a bad night, the operator asks the fleet what happened — and the answer is only worth having if the fleet cannot lie. A process that writes its own status and then dies leaves a database that says "running" forever. The runner store is built against exactly that failure: an embedded database that records only **facts** — things that definitely happened at a definite time — and derives every status by query. A status derived from "last heartbeat twenty minutes ago, process dead" tells the truth regardless of how the process ended.

## What to build

- **An embedded sqlite database inside the runner daemon**, its only reader and writer. Not a component anyone talks to: hooks and humans reach the runner's local API, and the schema stays private. No separate database server to operate, atomic crash-safe writes for free, and machine state inspectable with ordinary queries.
- **The machine-local fast path.** Leases, heartbeats, epochs, process identity (pid *and* start time, so a reused pid fools no one), environment bindings, verdicts, open asks, escalations. Everything fleet-visible lives at the hub; this store holds what only the machine can know.
- **Facts only, status derived.** Running, stalled, waiting-on-human, done — never a stored column, always a query over the facts. This is what makes crash recovery correct rather than aspirational.
- **Fencing epochs.** Every lease carries an incrementing epoch, minted here and reported upward. A worker reaped as dead but secretly still running carries a stale epoch, and its late submission is rejected — it can never clobber the successor that took over its chunk.
- **The outbound buffer.** Every hub-bound fact is written locally first, stamped with a per-runner sequence number, and drained in order — whether the hub is up or not. An outage is just a bigger backlog; replay is the ordinary code path, not a recovery mode.

## What was deliberately rejected

Lock files with hand-rolled atomicity, a separate Redis or Postgres to operate, git-as-database (too slow for second-granularity heartbeats), and forge-native assignment as the lease (rate limits, and arbitration belongs to the hub). The store's job is to make the boring choice and make it airtight.
