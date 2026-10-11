# Runner host specifications

The technical contracts behind `epic:runner-host`. Product intent begins at the [epic plan](../index.md); implementation
enters at the concern below. The runner's tenant rule — one runner, one store, one tenant, for life — is owned by
`epic:multi-tenancy`'s [runner slice](../../../delivered/multi-tenancy/runner.md); the served-projects declaration and
its wire are owned by `epic:projects`' [eligibility contract](../../../delivered/projects/hub/spec/eligibility.md); the
harness bundle and autonomy are owned by `epic:security`'s
[configuration contract](../../security/worker-security/spec/configuration.md). This spec places each of them on the
host; it redefines none.

Paths are relative to `blizzard/src/blizzard/`.

| Where                                  | Read when                                                                                                                |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| [hub.md](./hub.md)                     | Building the hub's side — the host record and its credential, adding and adopting runners, registration, and the board   |
| [configuration.md](./configuration.md) | Shaping the host's configuration file — which key lives on the host, a workspace, a runner, or a binding, and validation |
| [process.md](./process.md)             | Composing the daemon — what is per host and per runner, isolation, sign-in, listeners, the panel, and the CLI            |
| [workspaces.md](./workspaces.md)       | Routing a claim — project bindings, parallel agents, the shared environment pool, its crash recovery, workspace prompts  |
| [subscriptions.md](./subscriptions.md) | Sampling and spending a plan — declarations, the host store, reporting to the hub, and the brake's fan-out               |
| [carry-over.md](./carry-over.md)       | Moving today's runners under a host by hand, and what the hub accepts from a runner not yet moved                        |
