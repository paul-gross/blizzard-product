# Multi-tenancy

The tenant — the one boundary on a hub that holds for everything — landed on the hub first, then on the runner. Each
epic is frozen as it shipped.

| Where                            | Read when                                                                                                           |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| [initiative.md](./initiative.md) | Reasoning about tenancy as a whole — a tenant against a project, and why it was built beside projects               |
| [hub/](./hub/index.md)           | Looking up the tenant key, the store seam, identity and membership, and tenant administration, with their contracts |
| [runner.md](./runner.md)         | Looking up a runner serving one tenant for life — its store, its set-up, and its sign-in                            |
