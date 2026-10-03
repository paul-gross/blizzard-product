# Identity contract

Who a request is and which tenant it acts within are resolved separately, in that order. Identity is hub-level and is
resolved before any tenant is known; the tenant is named by the request and checked against the identity's memberships.

## What stays hub-level

`users`, `identities`, `sessions`, `auth_state`, and `auth_facts` keep their shape and stay global
([store.md](./store.md) §The global list). A person has one user row and one set of linked sign-in identities however
many tenants they belong to; a session belongs to a user, not to a tenant, so one sign-in serves every tenant the person
can enter. Usernames, emails, and provider subjects stay unique across the hub.

`auth.mode` stays hub-wide: it decides how a person signs in, which happens before a tenant is chosen.

## Memberships

A membership grants one user one role in one tenant. It is recorded as facts (`bzh:facts-not-status`):

| Column      | Meaning                                                                     |
| ----------- | --------------------------------------------------------------------------- |
| `id`        | autoincrement; the newest fact per `(tenant_id, user_id)` is in force       |
| `tenant_id` | the tenant granted into                                                     |
| `user_id`   | the user granted                                                            |
| `role`      | `guest`, `contributor`, or `admin`; `NULL` revokes                          |
| `set_at`    | from the injected clock (`bzh:injected-clock`)                              |
| `set_by`    | the granting user id, `migration`, or `operator` under `auth.mode = "none"` |

- **Roles move onto the membership.** `Role.GUEST`, `CONTRIBUTOR`, and `ADMIN` and their permission bundles
  (`auth_core.ROLE_PERMISSIONS`, `bzh:domain-core`) are unchanged; a role now means "in this tenant". `users.role` is
  dropped.
- **`pending` becomes the absence of a membership.** A person who signs in for the first time has an identity and no
  memberships; they reach `/api/me` and a "no tenants yet" page and nothing else, which is what `pending` grants today.
- **`superuser` becomes the hub administrator.** The `superuser_bootstrap` singleton keeps naming one user, claimed from
  `auth.superuser` as it is today. That user is the hub administrator: the role above every tenant, holding the new
  hub-level permissions below. It is not a membership role and grants no permission inside any tenant by itself.

### Permissions

`auth_core` gains two hub-level permissions, held by the hub administrator only and never expanded from a membership
role:

- `TENANT_ADMIN` — create, list, and delete tenants;
- `MEMBERSHIP_GRANT_ANY` — grant or revoke a membership in any tenant.

`USER_MANAGE` keeps its place in the `admin` bundle and narrows to the tenant: a tenant's admin grants, changes, and
revokes memberships in that tenant, for users who already exist on the hub. Creating a tenant grants its creator nothing
unless the creator names themselves as its first admin ([administration.md](./administration.md)).

## The resolved principal

`ResolvedIdentity` (`hub/auth/models.py`) gains the tenant it was resolved for:

```python
@dataclass(frozen=True)
class ResolvedIdentity:
    user_id: str
    username: str
    display_name: str
    tenant: TenantId | None          # None only on hub-level routes
    role: Role | None                # the membership's role in `tenant`
    permissions: frozenset[Permission]
    hub_admin: bool
```

`permissions` is `expand(role)` for the request's tenant, plus the hub-level permissions when `hub_admin` is true. It is
computed once at resolution, as today.

## Naming the tenant

| Caller                        | How the tenant is named                                                                               |
| ----------------------------- | ----------------------------------------------------------------------------------------------------- |
| A runner                      | its bearer token: the registration it resolves to carries exactly one `tenant_id`                     |
| A route token or marker token | the chunk it is bound to                                                                              |
| An API token                  | the tenant it was issued in                                                                           |
| A person on the board         | `X-Blizzard-Tenant`, which the board derives from its page route `/t/{name}/…`                        |
| A person on the CLI           | `X-Blizzard-Tenant`, from `--tenant <name-or-id>` or the context's saved default tenant, stored by id |
| A person naming none          | their only membership, when they hold exactly one; otherwise `409 tenant_required`                    |

The full resolution order, the naming rule for ids, names, and former names, and why the session never holds a current
tenant are owned by [api.md](./api.md) §Resolution order. A person's request is admitted when the named tenant exists
and the caller holds a membership in it; otherwise it is answered `404` — a tenant the caller cannot enter is
indistinguishable from one that does not exist. The hub administrator is admitted to hub-level routes regardless of
membership, and to a tenant's routes only through a membership of their own.

## Under `auth.mode = "none"`

The implicit operator (`hub/api/auth_session.py::IMPLICIT_OPERATOR`) is the hub administrator and acts as an `admin`
member of every tenant, acting in whichever tenant the request names. A request naming none resolves only while the hub
holds exactly one tenant.

## Open

- **What the hub administrator sees.** This contract admits the administrator to a tenant only through a membership.
  Whether administering a tenant should also let them read inside it is the slice plan's open question; the contract is
  built so that the answer is a change to the admission rule above and nothing else.
- **The auth-off shortcut for tests.** Whether a hub with `auth.mode = "none"` may hold more than one tenant — letting a
  test claim a tenant and act inside it by naming it in the `X-Blizzard-Tenant` header, with no user or session — is
  open. Nothing in this contract forbids it; deciding against it would make tenant creation refuse a second tenant while
  auth is off.
