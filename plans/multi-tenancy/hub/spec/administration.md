# Administration contract

Tenants are created and deleted by hub administrators, and only a hub administrator makes another
([identity.md](./identity.md) §Hub administrators). People are brought into a tenant by invitations to it
([identity.md](./identity.md) §Invitations), which the tenant's own admins issue, and the hub administrator for any
tenant. A tenant's admins also manage the roles of the members it already has. There is no self-service path: signing in
never admits anyone.

## Verbs

| Verb                     | API                                                 | CLI                                                                                                | Permission                                                                   |
| ------------------------ | --------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------- |
| Create a tenant          | `POST /api/admin/tenants` `{name, admin_user_id?}`  | `blizzard hub tenant create <name> [--admin <user_id>]`                                            | `TENANT_ADMIN`                                                               |
| List tenants             | `GET /api/admin/tenants`                            | `blizzard hub tenant list`                                                                         | `TENANT_ADMIN`                                                               |
| Rename a tenant          | `PATCH /api/admin/tenants/{tenant_id}` `{name}`     | `blizzard hub tenant rename <tenant_id> <name>`                                                    | `TENANT_ADMIN`                                                               |
| Delete a tenant          | `DELETE /api/admin/tenants/{tenant_id}`             | `blizzard hub tenant delete <tenant_id>`                                                           | `TENANT_ADMIN`                                                               |
| List members             | `GET /api/members`                                  | `blizzard hub member list --tenant <tenant_id>`                                                    | `USER_MANAGE`                                                                |
| Change a member's role   | `PUT /api/members/{user_id}` `{role}`               | `blizzard hub member grant <user_id> <role> --tenant <tenant_id>`                                  | `USER_MANAGE` for an existing member; `MEMBERSHIP_GRANT_ANY` to grant anyone |
| Revoke                   | `DELETE /api/members/{user_id}`                     | `blizzard hub member revoke <user_id> --tenant <tenant_id>`                                        | `USER_MANAGE` or `MEMBERSHIP_GRANT_ANY`                                      |
| Invite                   | `POST /api/invitations` `{email, role, expires_in}` | `blizzard hub invite create --tenant <tenant_id> --email <email> --role <role> [--expires <days>]` | `USER_MANAGE` in the tenant, or `MEMBERSHIP_GRANT_ANY`                       |
| List invitations         | `GET /api/invitations`                              | `blizzard hub invite list --tenant <tenant_id>`                                                    | `USER_MANAGE` in the tenant, or `MEMBERSHIP_GRANT_ANY`                       |
| Revoke an invitation     | `POST /api/invitations/{invitation_id}/revocations` | `blizzard hub invite revoke <invitation_id> --tenant <tenant_id>`                                  | `USER_MANAGE` in the tenant, or `MEMBERSHIP_GRANT_ANY`                       |
| Add a runner             | `POST /api/runners` `{name}`                        | `blizzard hub runner add <name> --tenant <tenant_id>`                                              | `runner:add`                                                                 |
| Rotate a runner's token  | `POST /api/runners/{runner_id}/enrollments`         | `blizzard hub runner enroll <id> --tenant <tenant_id>`                                             | `runner:add`                                                                 |
| List hub administrators  | `GET /api/admin/hub-admins`                         | `blizzard hub admin list`                                                                          | `HUB_ADMIN_GRANT`                                                            |
| Grant hub administrator  | `PUT /api/admin/hub-admins/{user_id}`               | `blizzard hub admin grant <user_id>`                                                               | `HUB_ADMIN_GRANT`                                                            |
| Revoke hub administrator | `DELETE /api/admin/hub-admins/{user_id}`            | `blizzard hub admin revoke <user_id>`                                                              | `HUB_ADMIN_GRANT`                                                            |
| Sync packaged graphs     | `POST /api/admin/graphs/sync`                       | `blizzard hub graph sync`                                                                          | `TENANT_ADMIN`                                                               |

The invitation verbs are CLI-only for now: the board offers no surface for them. Their routes act in the request's
tenant like any member route, and admit the hub administrator in any tenant without a membership
([identity.md](./identity.md) §Naming the tenant). `invite create` prints the link once; `invite list` shows each of the
tenant's invitations with its email, role, who issued it, and its state — `live`, `accepted` (by whom), `expired`, or
`revoked` — and never the link.

Member and runner routes act in the request's tenant ([api.md](./api.md) §Resolution order), and a runner belongs to the
tenant it was added in. Grants are `membership_facts` rows ([identity.md](./identity.md)); a grant whose role equals the
one in force writes nothing.

- **Members are named by user id.** `member grant` and `member revoke` take a user id, never a username or an email.
  Under `USER_MANAGE`, a user id that is not a member of the request's tenant answers `404`, exactly as one that does
  not exist; bringing in someone who is not yet a member is an invitation, or a hub administrator's direct grant under
  `MEMBERSHIP_GRANT_ANY`.
- **A tenant admin may hand out `admin`.** Granting, changing, and revoking the tenant `admin` role takes `USER_MANAGE`
  in the tenant, like any other membership role ([identity.md](./identity.md) §Permissions). Nobody changes their own
  role, and a tenant must keep at least one `admin` member once it has one: revoking or demoting the last admin is
  refused.
- **Hub administrators are granted only by hub administrators.** Revoking the last one is refused.
- **Today's users routes stay, deprecated.** `GET /api/users` and `POST /api/users/{user_id}/role` remain as aliases of
  `GET /api/members` and `PUT /api/members/{user_id}`, acting in the request's tenant, until a later change removes them
  ([api.md](./api.md) §Compatibility). The board's users page becomes the members page of the tenant it is opened in.

## Creating a tenant

One write transaction:

1. Mint a `ten_<ulid>` and insert the `tenants` row. The name is a label and is never refused for being taken.
2. Mint the packaged graphs into the tenant — the same reconciliation the hub-level graph sync performs for each tenant
   (§Syncing packaged graphs) — so every tenant starts with the library a fresh hub starts with.
3. Seed the tenant's built-in `hub` work source: its `work_item_sequence` row at 1.
4. Create the tenant's `default` project, so the tenant can ingest from its first moment (`epic:projects`
   [model.md](../../../projects/hub/spec/model.md) §Every tenant starts with a project).
5. Grant `admin` to the user named by `admin_user_id` (`--admin`), when one is named.

A tenant is usable the moment the transaction commits.

## Syncing packaged graphs

A deploy that ships new packaged graphs reconciles them into every tenant with one hub-level call.
`POST /api/admin/graphs/sync` (`TENANT_ADMIN`) runs today's reconciliation once per open tenant, each through that
tenant's own stores, and answers what it minted or retired per tenant; `blizzard hub graph sync` calls it, and the
hosted hub's deploy step runs that command after it migrates. The existing `POST /api/graphs/sync` stays as a
deprecated, tenant-scoped alias: it reconciles only the request's tenant, under `GRAPH_EDIT` as today, until a later
change removes it.

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
