# Multi-tenancy hub specifications

The technical contracts behind the hub slice of `epic:multi-tenancy`. Product intent begins at the
[slice plan](../index.md); implementation enters at the concern below. The secret store a tenant scopes is owned by
`epic:live-config`'s [secrets contract](../../../../delivered/live-config/spec/secrets.md).

| Where                                    | Read when                                                                                                                  |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| [store.md](./store.md)                   | Touching any table, repository, or query — the tenant key, the scoped store seam, the global list, uniqueness, migration   |
| [identity.md](./identity.md)             | Resolving who a request is and which tenant it acts within — users, memberships, roles, the hub administrator              |
| [api.md](./api.md)                       | Shaping a route, a dependency, or the event stream — tenant resolution, people and machine routes, SSE, wire compatibility |
| [administration.md](./administration.md) | Creating or deleting a tenant or granting a membership — and the teardown the shared-service tests depend on               |
| [carry-over.md](./carry-over.md)         | Migrating an existing installation into its first tenant                                                                   |
