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

| Column      | Meaning                                                                                  |
| ----------- | ---------------------------------------------------------------------------------------- |
| `id`        | autoincrement; the newest fact per `(tenant_id, user_id)` is in force                    |
| `tenant_id` | the tenant granted into                                                                  |
| `user_id`   | the user granted                                                                         |
| `role`      | `guest`, `contributor`, or `admin`; `NULL` revokes                                       |
| `set_at`    | from the injected clock (`bzh:injected-clock`)                                           |
| `set_by`    | the granting user id, `migration`, `bootstrap`, or `operator` under `auth.mode = "none"` |

- **Roles move onto the membership.** `Role.GUEST`, `CONTRIBUTOR`, and `ADMIN` and their permission bundles
  (`auth_core.ROLE_PERMISSIONS`, `bzh:domain-core`) are unchanged; a role now means "in this tenant". `users.role` is
  dropped, and `Role.SUPERUSER` and `Role.PENDING` leave the enum a membership role is drawn from: a membership is
  `guest`, `contributor`, or `admin`, and nothing else.
- **`pending` becomes the absence of a membership.** A user with no membership — one whose last membership was revoked,
  or a `pending` user carried over — reaches `/api/me` and a page that says *You don't have access to anything here yet.
  Reach out to your administrator for access.* and nothing else. Signing in never mints such a user: the hub is
  invite-only (§Invitations).
- **`superuser` becomes the hub administrator.** The hub administrator is the role above every tenant, holding the
  hub-level permissions below, and any number of users may hold it (§Hub administrators). It is not a membership role
  and grants no permission inside any tenant by itself.
- **Role changes stay on the security log.** Every grant, change, and revocation of a membership keeps writing
  `user_role_changed` to the global `auth_facts`, as a role change does today, now naming the tenant it happened in.
- **The first administrator is let into the first tenant.** When the superuser is claimed while the hub holds exactly
  one tenant and that tenant has no `admin` member, the same transaction grants the claiming user an `admin` membership
  in it, `set_by = "bootstrap"`. On a fresh hub this is how the person who set it up reaches `default` at all; on a
  carried-over hub the migration has already made them its admin ([carry-over.md](./carry-over.md)), so the claim grants
  nothing. Once a tenant has an admin, or a second tenant exists, the claim never grants a membership.

### Hub administrators

The hub administrator is a role a set of users hold, not one user. Who holds it is recorded as facts in the global
`hub_admin_facts` table (`bzh:facts-not-status`):

| Column    | Meaning                                                               |
| --------- | --------------------------------------------------------------------- |
| `id`      | autoincrement; the newest fact per `user_id` is in force              |
| `user_id` | the user granted or revoked                                           |
| `granted` | `true` grants the role, `false` revokes it                            |
| `set_at`  | from the injected clock (`bzh:injected-clock`)                        |
| `set_by`  | the granting hub administrator's user id, `bootstrap`, or `migration` |

- **The first comes from configuration.** `superuser_bootstrap` keeps one job: the first claim. The user who first signs
  in as `auth.superuser` is recorded there as today, and the same transaction writes the first `hub_admin_facts` row,
  `set_by = "bootstrap"`. A carried-over hub's claimed superuser becomes a hub administrator through a row
  `set_by = "migration"` ([carry-over.md](./carry-over.md)).
- **Only a hub administrator makes another.** Granting and revoking the role takes `HUB_ADMIN_GRANT`, which only hub
  administrators hold: `blizzard hub admin list|grant|revoke <user_id>` over `GET /api/admin/hub-admins` and
  `PUT`/`DELETE /api/admin/hub-admins/{user_id}` ([administration.md](./administration.md)). No membership role reaches
  it, and no tenant admin can make anyone a hub administrator.
- **There is always one.** Revoking the last hub administrator is refused, so a hub can never be left with nobody able
  to create a tenant or grant the role again.

### Permissions

`auth_core` gains four hub-level permissions, held by hub administrators only and never expanded from a membership role:

- `TENANT_ADMIN` — create, list, rename, and delete tenants;
- `MEMBERSHIP_GRANT_ANY` — grant or revoke a membership in any tenant, and issue, list, and revoke any tenant's
  invitations;
- `HUB_ADMIN_GRANT` — grant and revoke the hub administrator role (§Hub administrators);
- `EXPORT_ADMIN` — operate the hub-wide exports whose cursors every tenant shares: read trace export's and fact egress's
  status, replay a trace window, and move or backfill an egress cursor ([store.md](./store.md) §Fact egress).

There are two levels of administrator, and neither reaches the other's. `USER_MANAGE` keeps its place in the `admin`
bundle and narrows to the tenant: a tenant's admin changes and revokes the roles of their tenant's existing members —
`admin` included — and brings people in by inviting them (§Invitations), with any role up to `admin`. This deliberately
replaces today's rule that only a superuser may grant or revoke `admin` (`hub/api/users.py::assign_role`): `admin` is
now a tenant's own role, so the tenant's admins hand it out, while the role above every tenant is granted only by hub
administrators. Two guards stay: nobody changes their own role, and a tenant that has an admin keeps at least one
([administration.md](./administration.md)). A tenant's admin names a member by user id; a user id that is no member of
their tenant answers `404`, exactly as a user id that does not exist, so a role change can never be used to probe who
else is on the hub. Bringing someone into a tenant — new to the hub or already on it — is always an invitation, or the
hub administrator's own direct grant under `MEMBERSHIP_GRANT_ANY`. A tenant's admin invites by email address and never
searches the hub's users, so no tenant learns who else is on the hub, or whether the address they invited already has an
account. Creating a tenant grants its creator nothing unless the creator names themselves as its first admin
([administration.md](./administration.md)).

## Invitations

The hub is invite-only. Signing in proves who a person is; it never admits them. A provider identity the hub has not
seen before becomes a user only by accepting an invitation, or by being the configured superuser (§Memberships). Any
other first sign-in creates no user, no identity link, and no session, and lands on a page that says *This hub is
invite-only. Reach out to your administrator for an invitation.* An identity already linked to a user signs in as today,
and a new identity whose verified email matches an existing user still links to that user, under today's email-merge
rule — invitations change who can arrive, not how a returning person is recognized.

### What an invitation is

An invitation belongs to one tenant: it admits one email address into that tenant with one role. The tenant's admins
issue it — anyone holding `USER_MANAGE` there — and so does the hub administrator, for any tenant, which is how a new
tenant gets its first admin. It is issued from the CLI ([administration.md](./administration.md)); there is no board
surface for it yet. The CLI prints a link once — `https://<hub>/invite/<token>` — and whoever issued it hands it to the
person by whatever channel they already use; the hub sends no mail. An invitation is a tenant-owned record, so it lives
and dies with its tenant.

| Column          | Meaning                                                                     |
| --------------- | --------------------------------------------------------------------------- |
| `invitation_id` | `inv_<ulid>`                                                                |
| `tenant_id`     | the tenant the invitation admits into                                       |
| `email`         | the one address that may accept it, stored lowercased                       |
| `role`          | the membership role accepting it grants: `guest`, `contributor`, or `admin` |
| `token_hash`    | sha256 of the link's token; the token itself is shown once and never stored |
| `created_by`    | the inviting user's id                                                      |
| `created_at`    | from the injected clock                                                     |
| `expires_at`    | `created_at` plus the lifetime asked for: 7 days by default, at most 30     |

Invitations live in the tenant-owned `invitations` table. What became of one is recorded as one fact beside it in
`invitation_facts` (`bzh:facts-not-status`): `accepted` with the accepting user and time, or `revoked` with the revoking
admin and time. An invitation is live while it has neither fact and has not expired; it is used at most once.

### Accepting one

1. **The link starts a sign-in.** Opening `/invite/<token>` on the board takes the person to the hub's sign-in with the
   token carried through the provider round trip in `auth_state`, beside the state and PKCE values it already holds.
2. **The email must match.** On the provider's callback the hub resolves the token through `HubScopedReads`, and
   requires that the provider report a verified email equal to the invitation's, compared case-insensitively. A provider
   reports every verified address the account holds — the GitHub provider reads all of `/user/emails`, not only the
   primary — so a person whose invited address is a secondary one on their account is not turned away. Whatever account
   the person signs in with, it must carry that address.
3. **One transaction admits them.** It creates the user when the identity is new (or uses the user the identity or the
   email already resolves to), links the identity, writes the `membership_facts` row with `set_by` naming the
   invitation, records the `accepted` fact, and mints the session. The person lands in the invited tenant.

When the invitation is expired, revoked, or already used, nothing is written and the page says *This invitation is no
longer valid. Reach out to your administrator for a new one.* When no verified email on the account matches, nothing is
written, the invitation stays live, and the page says *This invitation is for a different email address. Sign in with
the account that uses it, or reach out to your administrator.* A person already signed in who opens an invitation goes
through the same check: the invitation admits their account into one more tenant only if that account carries the
invited address.

### Why email

Email is the key because GitHub is the one provider the hub trusts to start, and GitHub only reports an address as
verified once its owner has proven it. That trust is the hub administrator's to extend: a provider whose verified-email
claim is not authoritative for every address it reports — a client's own directory, say — must never admit through an
invitation by email, and would bind invitations to its own subject instead.

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

| Caller                          | How the tenant is named                                                                  |
| ------------------------------- | ---------------------------------------------------------------------------------------- |
| A runner                        | its bearer token: the registration it resolves to carries exactly one `tenant_id`        |
| A route token                   | the chunk it is bound to                                                                 |
| A marker token                  | the tenant the hub recorded when it minted the token ([api.md](./api.md) §Marker tokens) |
| A person on the board           | `X-Blizzard-Tenant`, from the tab's chosen tenant ([api.md](./api.md))                   |
| A person on the CLI             | `X-Blizzard-Tenant`, from `--tenant <tenant_id>` or the context's saved default tenant   |
| A person signing in to a runner | the runner's registration ([api.md](./api.md) §Signing in to a runner)                   |
| A person naming none            | their only membership, when they hold exactly one; otherwise `409 tenant_required`       |

The full resolution order, the rule that only an id names a tenant, and why the session never holds a current tenant are
owned by [api.md](./api.md) §Resolution order. A person's request is admitted when the named tenant exists and the
caller holds a membership in it; otherwise it is answered `404` — a tenant the caller cannot enter is indistinguishable
from one that does not exist. The hub administrator is admitted to hub-level routes regardless of membership, and to a
tenant's routes only through a membership of their own — with one exception: the invitation routes, which they may use
in any tenant so that a tenant with no admin yet can be given one. Those routes read and write invitations and nothing
else of the tenant's.

## Under `auth.mode = "none"`

The implicit operator (`hub/api/auth_session.py::IMPLICIT_OPERATOR`) is the hub administrator and acts as an `admin`
member of every tenant, acting in whichever tenant the request names. A request naming none resolves only while the hub
holds exactly one tenant.

Such a hub may hold any number of tenants, and this is the shortcut a test takes: it creates a tenant and acts inside it
by naming it in `X-Blizzard-Tenant`, with no user or session. Nothing equivalent exists while auth is on — every caller
there resolves to a person with a membership or to a machine credential. The shortcut lives exactly as long as
`auth.mode = "none"` does.
