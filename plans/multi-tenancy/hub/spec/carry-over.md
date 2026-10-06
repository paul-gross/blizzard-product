# Carry-over contract

An installation that exists before this slice becomes a hub holding exactly one tenant, with nothing re-ingested,
nothing reconfigured, and nothing an operator has to do beyond the migrate command every release already asks for.

## The first tenant

The migration ([store.md](./store.md) §Migration shape) inserts one tenant:

| Field        | Value                                                    |
| ------------ | -------------------------------------------------------- |
| `tenant_id`  | a fresh `ten_<ulid>`                                     |
| `name`       | `default`, renamable afterwards by the hub administrator |
| `created_by` | `migration`                                              |

Nothing distinguishes it from any tenant created later: it can be renamed, and it can be deleted once another tenant
exists. It is the only tenant, which is why every unambiguous-resolution rule in [identity.md](./identity.md) and
[api.md](./api.md) resolves to it.

A fresh hub runs the same revision, so it too starts with an empty `default` tenant, and the person who claims the
superuser becomes its admin ([identity.md](./identity.md) §Memberships).

## What is stamped

- **Every row of every tenant-owned table** receives the first tenant's id, in the backfill step, before the column is
  tightened to `NOT NULL`. A table the build check classifies as tenant-owned and the backfill misses fails the
  tightening step, so the migration cannot complete with an unstamped row.
- **Every runner registration** is inside the first tenant. Its bearer token resolves exactly as before and now also
  settles its tenant, so a runner deployed before the upgrade keeps claiming and reporting without re-enrollment.
- **Every scope** receives its surrogate `scope_id`; rows that pointed at its slug are repointed in the same revision.
- **Each `work_item_sequence` row** keeps its `next_ref`, now under the first tenant, so the next hub-source item
  numbers on from where the installation left off.

## Who becomes a member

Each user's current `users.role` becomes one `membership_facts` row in the first tenant, `set_by = "migration"`:

| `users.role`  | Becomes                                                                                   |
| ------------- | ----------------------------------------------------------------------------------------- |
| `guest`       | a `guest` membership                                                                      |
| `contributor` | a `contributor` membership                                                                |
| `admin`       | an `admin` membership                                                                     |
| `superuser`   | an `admin` membership; the user stays the hub administrator through `superuser_bootstrap` |
| `pending`     | no membership — a pending user stays exactly as unable to act as before, until invited    |

`users.role` is dropped after the copy. Sessions survive the migration: a person signed in before the upgrade is still
signed in after it, now resolving into the first tenant.

## What does not change

- **Configuration.** No key is added to `blizzard-hub.toml` and none is required; `auth.mode`, the rollout modes, and
  `auth.superuser` keep their meaning.
- **URLs.** No API route changes path, and a client that names no tenant keeps resolving, because a caller on a
  one-tenant hub is never ambiguous. Links shared before the upgrade still open.
- **The fleet wire.** Runners keep working unchanged; the wire only gains fields ([api.md](./api.md) §Compatibility).

An operator who never creates a second tenant never sees the concept, beyond a tenant name in the board's header: the
board's URLs stay exactly as they are, and no link carries a tenant ([api.md](./api.md) §Where people's clients get the
header).
