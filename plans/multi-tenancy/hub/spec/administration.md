# Administration contract

Tenants are created and deleted by the hub administrator; memberships are granted by the administrator in any tenant and
by a tenant's admins in their own. There is no self-service path.

## Verbs

| Verb                    | API                                          | CLI                                                         | Permission                              |
| ----------------------- | -------------------------------------------- | ----------------------------------------------------------- | --------------------------------------- |
| Create a tenant         | `POST /api/admin/tenants` `{name}`           | `blizzard hub tenant create <name> [--admin <user>]`        | `TENANT_ADMIN`                          |
| List tenants            | `GET /api/admin/tenants`                     | `blizzard hub tenant list`                                  | `TENANT_ADMIN`                          |
| Rename a tenant         | `PATCH /api/admin/tenants/{tenant}` `{name}` | `blizzard hub tenant rename <tenant> <name>`                | `TENANT_ADMIN`                          |
| Delete a tenant         | `DELETE /api/admin/tenants/{tenant}`         | `blizzard hub tenant delete <tenant>`                       | `TENANT_ADMIN`                          |
| List members            | `GET /api/members`                           | `blizzard hub member list --tenant <tenant>`                | `USER_MANAGE`                           |
| Grant or change a role  | `PUT /api/members/{user_id}` `{role}`        | `blizzard hub member grant <user> <role> --tenant <tenant>` | `USER_MANAGE` or `MEMBERSHIP_GRANT_ANY` |
| Revoke                  | `DELETE /api/members/{user_id}`              | `blizzard hub member revoke <user> --tenant <tenant>`       | `USER_MANAGE` or `MEMBERSHIP_GRANT_ANY` |
| Add a runner            | `POST /api/runners` `{name}`                 | `blizzard hub runner add <name> --tenant <tenant>`          | the runner-add permission               |
| Rotate a runner's token | `POST /api/runners/{runner_id}/enrollments`  | `blizzard hub runner enroll <id> --tenant <tenant>`         | `RUNNER_PAUSE`, as today                |

Member and runner routes act in the request's tenant ([api.md](./api.md) §Resolution order), and a runner belongs to the
tenant it was added in. Grants are `membership_facts` rows ([identity.md](./identity.md)); a grant whose role equals the
one in force writes nothing. A tenant must keep at least one `admin` member once it has one: revoking or demoting the
last admin is refused. Today's users page (`/api/users`, `/api/users/{user_id}/role`) becomes the members page of the
tenant it is opened in.

## Creating a tenant

One write transaction:

1. Mint a `ten_<ulid>` and insert the `tenants` row; a name a live tenant holds is refused `409`.
2. Mint the packaged graphs into the tenant — the same reconciliation `blizzard hub graph sync` performs, scoped to the
   new tenant — so every tenant starts with the library a fresh hub starts with.
3. Seed the tenant's built-in `hub` work source: its `work_item_sequence` row at 1.
4. Grant `admin` to the user named by `--admin`, when one is named.

A tenant is usable the moment the transaction commits. `graph sync` after a deploy reconciles packaged graphs into every
tenant.

## Deleting a tenant

Deletion is complete — no row of the tenant survives — and bounded in how long it holds the writer, because other
tenants share it ([store.md](./store.md) §SQLite and Postgres). It runs in three phases, each committed on its own.

1. **Close.** Insert the `tenant_deletions` fact with `deleted_at` unset. From that commit on the tenant does not
   resolve: its people get `404`, its runners' tokens get `401`, its broker is dropped, and its cached `TenantStores` is
   evicted. Deleting the hub's only tenant is refused.
2. **Drain.** Delete the tenant's rows table by table, children before parents, in the reverse of
   `schema.metadata.sorted_tables` restricted to tenant-owned tables — derived from the metadata, never a hand-kept
   list, so a table added later is drained without anyone remembering it. Each table drains in batches of at most 5,000
   rows per transaction, so another tenant's writer waits at most one batch. SQLite runs with `foreign_keys` off
   (`foundation/store/engine.py`); the order, not a cascade, is what keeps the store consistent.
3. **Complete.** Delete the `tenants` row and its `membership_facts`, and record `deleted_at` and `rows_removed` on the
   deletion fact. The tenant's name and former names are free from this commit; its id is never reused.

The drain is resumable (`bzh:steppable-loop`): a `tenant_teardown` sweep finds every deletion fact with `deleted_at`
unset and continues it from wherever its tables stand, so a hub killed mid-drain finishes the job after restart. The
close and complete phases are crash points in the hub's registry (`bzh:crash-point-registry`), and the invariant checker
(`bzh:invariant-checker`) gains one invariant: no tenant-owned row names a tenant absent from `tenants` once its
deletion completed.

### How fast

`DELETE /api/admin/tenants/{tenant}` runs all three phases inline when the tenant holds fewer than 50,000 rows, and
answers `200` with `rows_removed`; above that it answers `202` after the close phase and the sweep finishes the drain. A
tenant of the size a service test creates — a few graphs, a handful of chunks and their facts — is expected to tear down
in well under a second on SQLite; `epic:test-shared-service` measures it and owns the number it relies on.
