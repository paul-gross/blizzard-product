# Runner host specifications

The technical contracts behind `epic:runner-host`. Product intent begins at the [epic plan](../index.md); implementation
enters at the concern below. The runner's tenant rule — one runner, one store, one tenant, for life — is owned by
[`epic:multi-tenancy/runner`](../../multi-tenancy/runner.md); the served-projects declaration and its wire are owned by
`epic:projects/hub`' [eligibility contract](../../projects/hub/spec/eligibility.md); the harness bundle and autonomy are
owned by `epic:security/worker-security`'s
[configuration contract](../../../plans/security/worker-security/spec/configuration.md). This spec places each of them
on the host; it redefines none.

Paths are relative to `blizzard/src/blizzard/`.

The host does not stop at `blizzard`'s source. The mock fleet in `blizzard-mock` mirrors every route and field the hub
gains (`bzh:wire-change-extends-mock`), and its data tooling and real-runner tests read a host's directory, store, and
token names where they read a runner's today. `blizzard-context` learns the host credential among the hub's scoped
reads, the runner store's scoping rule, and host, workspace, and binding as domain concepts. Blizzard's operator
documents carry the procedure the release ships (`docs/upgrade.md`, beside `docs/remote-runner.md` and the runner,
sign-in, and spend guides under `docs/deployment/`), and its packaged service unit
(`packaging/systemd/blizzard-runner.service`) starts a host.

| Where                                  | Read when                                                                                                                                                               |
| -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [store.md](./store.md)                 | Shaping the host's one database — its two revisions, runner-scoped and host-scoped tables, scoped access, writers, integer keys, invariants, per-runner reset           |
| [hub.md](./hub.md)                     | Building the hub's side — the host record and its credential, adding, adopting, and releasing runners, registration, the host's fact lane, sign-in, and the board       |
| [configuration.md](./configuration.md) | Shaping the host's configuration file — which key lives on the host, a workspace, a runner, or a binding, a runner's id, and validation                                 |
| [process.md](./process.md)             | Composing the daemon — what is per host and per runner, isolation, sign-in, the routes kept at the root, the panel, and the CLI                                         |
| [workspaces.md](./workspaces.md)       | Routing a claim — project bindings, parallel agents, the shared environment pool and its lock, its crash recovery, workspace prompts                                    |
| [subscriptions.md](./subscriptions.md) | Sampling and spending a plan — declarations, the host store, reporting to the hub, and the brake's fan-out                                                              |
| [frontend.md](./frontend.md)           | Building the host panel and the hub's host card — routes, runner selection, the generated client, query keys, and what is shared                                        |
| [carry-over.md](./carry-over.md)       | Landing the host on a fleet that runs it — the order of landings, graduating today's runners by hand, registering a host, ingesting a runner, backfilling held bindings |
