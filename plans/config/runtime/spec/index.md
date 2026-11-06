# Runtime configuration specifications

The technical contracts behind `epic:config/runtime`. Product intent begins at the [epic plan](../index.md): that every
setting has a lifetime — `pass`, `session`, `boot`, or `immutable` — that says when a change bites; that each daemon
owns its own settings, changed only through its own API and seeded from its file where its store has none; that a pass
loads its settings once at its start and never mid-pass; that a worker keeps what it launched with; that a runner's
every setting but its id changes while it runs, and runners, workspaces, and subscriptions come and go on a running
host; that only the host's footing waits for a restart; and that the hub's operational settings follow the same rules.
This spec states how each of those is built and redefines none of it.

It builds on what the two daemons already do. The runner host reads `blizzard-host.toml` once, freezes a `LoopConfig`
per runner on its first tick, ticks every runner and itself on an interval fixed at boot, and serves its runners from
one process whose local API already carries human-lane writes — pause, takeover, the workspace prompt — over a unix
socket and a federated session. The hub gave its tenants' records revisions, a change log, a declarative apply, and a
read-on-use cache under `epic:live-config` (`bzh:configured-record`, `bzh:config-apply`, `bzh:config-read-on-use`); its
own operational settings are still bound into long-lived objects when it starts.

Neither half crosses the fleet wire: each daemon's settings are its own, a runner declares to the hub what it already
declares at each registration, and neither daemon reads or checks the other's settings, so `blizzard-mock` does not
change (`bzh:wire-change-extends-mock` is not engaged). The new routes regenerate the runner's and the hub's clients
(`bzh:generated-client`), and the store, CLI, and configuration contract documents follow the code.

The epic reaches beyond `blizzard`'s source. In `blizzard-context`, `bzh:config-key-declared` gains the lifetime,
`bzh:config-read-on-use` and `bzh:configured-record` gain entries and the hub-wide record, a new
`bzh:settings-snapshot-per-pass` states the evaluation every pass makes, the domain's bindings stop halting a runner for
a retired workspace, and the runner's crash-correctness record gains the add's window. Blizzard's operator pages on the
runner host and the hub's configuration gain lifetimes, entries, the seed, and the API.

| Where                                | Read when                                                                                                                                                                       |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [settings.md](./settings.md)         | Looking up a setting — every hub and runner setting with its lifetime and the reader that holds it today, the one new setting, environment variables, and seeding from the file |
| [lifetimes.md](./lifetimes.md)       | Declaring a lifetime — the registry's two new fields, the contract documents, entries, and the evaluation every pass makes at its start                                         |
| [runner-store.md](./runner-store.md) | Storing a runner's settings — the tables, a write and its validation, the seed's order, and the reader a pass resolves its configuration through                                |
| [runner-api.md](./runner-api.md)     | Changing a runner's settings — who may write, the routes, a view and a write, apply and export, the CLI, and the local page                                                     |
| [passes.md](./passes.md)             | Making a runner's pass evaluate its settings — the tick's rebuild, the trace sweep, the host tick's reconcile, the live interval, and what a session takes with it              |
| [host-records.md](./host-records.md) | Adding or retiring a runner on a running host, or shaping a workspace or subscription as a record — its verbs, what each change does, and the add's crash window                |
| [hub.md](./hub.md)                   | Changing the hub's settings — the hub-wide record, its permission, routes, apply, every holder read per pass, and the board                                                     |
| [carry-over.md](./carry-over.md)     | Landing the epic — the order of its pieces and their proof, version skew, what an operator does once, and what is left for later                                                |
