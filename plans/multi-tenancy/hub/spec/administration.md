# Administration contract

Tenants are created and deleted by the hub administrator, and people are brought into them by the administrator's
invitations ([identity.md](./identity.md) §Invitations). A tenant's admins manage the roles of the members it already
has. There is no self-service path: signing in never admits anyone.

## Verbs

| Verb                    | API                                                                  | CLI                                                                                                | Permission                                                                   |
| ----------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Create a tenant         | `POST /api/admin/tenants` `{name}`                                   | `blizzard hub tenant create <name> [--admin <user>]`                                               | `TENANT_ADMIN`                                                               |
| List tenants            | `GET /api/admin/tenants`                                             | `blizzard hub tenant list`                                                                         | `TENANT_ADMIN`                                                               |
| Rename a tenant         | `PATCH /api/admin/tenants/{tenant_id}` `{name}`                      | `blizzard hub tenant rename <tenant_id> <name>`                                                    | `TENANT_ADMIN`                                                               |
| Delete a tenant         | `DELETE /api/admin/tenants/{tenant_id}`                              | `blizzard hub tenant delete <tenant_id>`                                                           | `TENANT_ADMIN`                                                               |
| List members            | `GET /api/members`                                                   | `blizzard hub member list --tenant <tenant_id>`                                                    | `USER_MANAGE`                                                                |
| Change a member's role  | `PUT /api/members/{user_id}` `{role}`                                | `blizzard hub member grant <user> <role> --tenant <tenant_id>`                                     | `USER_MANAGE` for an existing member; `MEMBERSHIP_GRANT_ANY` to grant anyone |
| Revoke                  | `DELETE /api/members/{user_id}`                                      | `blizzard hub member revoke <user> --tenant <tenant_id>`                                           | `USER_MANAGE` or `MEMBERSHIP_GRANT_ANY`                                      |
| Invite                  | `POST /api/admin/invitations` `{tenant_id, email, role, expires_in}` | `blizzard hub invite create --tenant <tenant_id> --email <email> --role <role> [--expires <days>]` | `MEMBERSHIP_GRANT_ANY`                                                       |
| List invitations        | `GET /api/admin/invitations`                                         | `blizzard hub invite list [--tenant <tenant_id>]`                                                  | `MEMBERSHIP_GRANT_ANY`                                                       |
| Revoke an invitation    | `POST /api/admin/invitations/{invitation_id}/revocations`            | `blizzard hub invite revoke <invitation_id>`                                                       | `MEMBERSHIP_GRANT_ANY`                                                       |
| Add a runner            | `POST /api/runners` `{name}`                                         | `blizzard hub runner add <name> --tenant <tenant_id>`                                              | the runner-add permission                                                    |
| Rotate a runner's token | `POST /api/runners/{runner_id}/enrollments`                          | `blizzard hub runner enroll <id> --tenant <tenant_id>`                                             | `RUNNER_PAUSE`, as today                                                     |

The invitation verbs are CLI-only by design: the board offers no surface for them, and the routes are hub-level, naming
their tenant in the body. `invite create` prints the link once; `invite list` shows each invitation's tenant, email,
role, and state — `live`, `accepted` (by whom), `expired`, or `revoked` — and never the link.

Member and runner routes act in the request's tenant ([api.md](./api.md) §Resolution order), and a runner belongs to the
tenant it was added in. Grants are `membership_facts` rows ([identity.md](./identity.md)); a grant whose role equals the
one in force writes nothing. A tenant must keep at least one `admin` member once it has one: revoking or demoting the
last admin is refused. Today's users page (`/api/users`, `/api/users/{user_id}/role`) becomes the members page of the
tenant it is opened in.

## Creating a tenant

One write transaction:

1. Mint a `ten_<ulid>` and insert the `tenants` row. The name is a label and is never refused for being taken.
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
   resolve: its people get `404`, its runners' tokens get `401`, its broker is dropped, its cached `TenantStores` is
   evicted, every write through its stores is refused, and every sweep passes over it ([store.md](./store.md) §Sweeps).
   Deleting the hub's only tenant is refused.
2. **Drain.** Delete the tenant's rows table by table, children before parents, in the reverse of
   `schema.metadata.sorted_tables` restricted to tenant-owned tables — derived from the metadata, never a hand-kept
   list, so a table added later is drained without anyone remembering it. Each table drains in batches of at most 5,000
   rows per transaction, so another tenant's writer waits at most one batch. SQLite runs with `foreign_keys` off
   (`foundation/store/engine.py`); the order, not a cascade, is what keeps the store consistent.
3. **Complete.** Delete the `tenants` row and its `membership_facts`, and record `deleted_at` and `rows_removed` on the
   deletion fact. Its id is never reused.

The drain is resumable (`bzh:steppable-loop`): a `tenant_teardown` sweep finds every deletion fact with `deleted_at`
unset and continues it from wherever its tables stand, so a hub killed mid-drain finishes the job after restart. The
close and complete phases are crash points in the hub's registry (`bzh:crash-point-registry`), and the invariant checker
(`bzh:invariant-checker`) gains one invariant: no tenant-owned row names a tenant absent from `tenants` once its
deletion completed.

### How fast

`DELETE /api/admin/tenants/{tenant_id}` closes the tenant and then drains it inline, under a budget of 50,000 rows. It
never counts the tenant first: the drain itself discovers how much there is. When the drain finishes inside the budget,
the request runs the complete phase and answers `200` with `rows_removed`. When the budget runs out first, it answers
`202` and the `tenant_teardown` sweep carries on from wherever the request stopped. A tenant of the size a service test
creates — a few graphs, a handful of chunks and their facts — is expected to tear down in well under a second on SQLite;
`epic:test-shared-service` measures it and owns the number it relies on.

### Retrying, and watching a deletion finish

A caller whose inline delete was cut off — a crash, a timeout — has to be able to ask again and learn where things
stand, so a tenant mid-deletion stays reachable to the administration routes by its id:

- **`DELETE` on a tenant whose deletion is underway** answers `202` and changes nothing; it does not restart the drain.
- **`DELETE` on a tenant whose deletion has completed** answers `410`, from its `tenant_deletions` fact. An id is never
  reused, so a retry can never reach a different tenant.
- **`tenant list`** shows a tenant mid-deletion with the state `deleting` and the time its deletion began.
