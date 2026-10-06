# API contract

Every route resolves a tenant before it touches tenant-owned state, and it resolves it in exactly one place: a router
dependency, never a handler. No API route changes shape and nothing is mounted twice; the tenant travels beside the
path, not in it. The runner-reached wire only gains fields.

## Route families

| Family    | Paths                                                                                                   | Tenant comes from                                                    |
| --------- | ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Hub-level | `/api/health`, `/api/ready`, `/api/me`, `/api/auth/*`, `/api/admin/*`, `/api/egress/*`, `/api/traces/*` | none — these routes hold no `TenantStores`                           |
| People    | `/api/…` — every operator router, at its current path                                                   | the request's resolution order below                                 |
| Fleet     | `/api/fleet/*` — unchanged paths and requests; responses only gain fields                               | the machine credential the call already carries, never anything else |

`/api/fleet/system-artifacts*` serves blizzard's packaged documents — the finding and proposal formats agents write to —
which are read from the installed package and are the same in every tenant. It stays in the fleet family, so its
caller's credential still resolves a tenant, but it reads no tenant-owned state.

## Resolution order

Each request resolves its tenant by the first rule that applies:

1. **A machine credential decides.** A runner bearer token and a route token are each bound to exactly one tenant,
   resolved through `HubScopedReads`; a marker token carries the tenant it was minted in (§Marker tokens). The
   credential's tenant is the request's tenant. A request that also carries an `X-Blizzard-Tenant` header naming a
   different tenant is refused `400`; one naming the same tenant is accepted.
2. **A person names it.** An `X-Blizzard-Tenant: <tenant_id>` header is checked against the caller's memberships. A
   value that is no live tenant's id, or the id of a tenant the caller cannot enter, answers `404` — the two are
   indistinguishable.
3. **One membership settles it.** With no header, a caller holding exactly one membership acts in that tenant.
4. **Otherwise** the request answers `409 tenant_required`.

Under `auth.mode = "none"` the implicit operator is a member of every tenant ([identity.md](./identity.md)), so rule 3
applies only while the hub holds exactly one tenant.

### Naming a tenant

The header value, the CLI's `--tenant`, `runner init --tenant`, and every `/api/admin/tenants/{tenant_id}` path take the
tenant's id, `ten_<ulid>`, and nothing else. A tenant's name is never accepted in its place: names are labels, unique
nowhere ([store.md](./store.md) §The tenant record), so there is no name to resolve, no former name to honor, and no way
for a stale name to land on a different tenant. An id stays valid for as long as its tenant exists.

### Why the session holds no current tenant

A session belongs to a person, not to a tenant ([identity.md](./identity.md)), and it is shared by every tab and window
that person has open. A "current tenant" stored on it would be one value for all of them: switching tenant in one tab
would silently move every other open tab into the new tenant, and the next write from an older tab would land somewhere
its page never showed. The tenant is therefore stated per request, by whatever the request came from — the tab's own
state, the CLI's own context — and two tabs on two tenants never interfere.

### Where people's clients get the header

- **The board** keeps its page URLs exactly as they are today: no URL carries a tenant. Each tab holds the tenant it is
  working in, in the tab's own `sessionStorage`, and sends that id as `X-Blizzard-Tenant` on every API call the page
  makes, from one HTTP interceptor. It shows the tenant's name, from `/api/me`, wherever a person reads it.
- **Which tenant a tab opens in.** A caller with one membership is always in it, and never sees a choice. A caller with
  several opens in the tenant this browser last chose — its id kept in `localStorage` — while they still hold a
  membership there, and otherwise sees a tenant picker; a caller with none sees the page that tells them to reach out to
  their administrator for access ([identity.md](./identity.md) §Memberships).
- **Switching** is a tenant switcher in the profile menu, shown only to a caller with several memberships. It sets the
  tab's tenant and the browser's last choice, then reloads the tab's current view in the new tenant. It needs no new
  sign-in: the session is the person's, and the next request is checked against their memberships like any other.
- **A shared link** names a page and the records on it, never a tenant, so it opens in whichever tenant the recipient's
  tab is in. Record ids are unique across the hub, so a link to another tenant's record answers not found rather than
  ever reaching the wrong record. For the people with one membership, which is nearly everyone, every link simply works.
- **The CLI** sends the header from `--tenant <tenant_id>`, else from the context's saved default tenant. It lists the
  caller's tenants, ids beside names, from `/api/me`, so a person finds an id without leaving the terminal. Its sign-in
  (`GET /api/auth/authorize?client=cli`, `hub/api/idp.py`) stays a hub-level route and mints a session that belongs to
  the person and no tenant; the CLI names its tenant per request, as the board does.

## The dependency

`get_services(request) -> HubServices` (`hub/api/deps.py`) is replaced, for every tenant-owned route, by a router-level
dependency that produces one value per request:

```python
@dataclass(frozen=True)
class TenantRequest:
    identity: ResolvedIdentity        # identity.tenant == scope.tenant
    scope: StoreScope                 # the request's tenant
    services: TenantServices          # built over HubCore.open(scope)
```

- **People.** `require(permission)` (`hub/api/auth_session.py`) resolves the identity as today, applies the resolution
  order, checks the membership, expands the membership role, and only then checks `permission`, which answers `403` as
  today.
- **Fleet.** `require_runner_principal` (`hub/api/auth.py`) resolves the bearer token through `HubScopedReads` to
  `(runner_id, workspace_id, tenant_id)`, where `runner_id` is the id the hub minted when the runner was added;
  `RunnerPrincipal` gains `tenant_id`, and `FleetRequest` builds its services from that tenant's `TenantStores`.
  Route-token resolution does the same through the chunk its token is bound to.
- **Marker tokens.** A marker token is minted by the hub for one hub-executed node visit, `(chunk_id, node_id, epoch)`,
  and lives in the hub's memory (`hub/delivery/marker_auth.py`), not its store. The hub mints it inside that chunk's
  tenant-scoped step, so `MarkerAuthority` records the tenant beside the triple. Verifying one
  (`hub/api/marker_auth.py::require_marker_authority`) returns a `MarkerPrincipal` — `chunk_id`, `node_id`, `epoch`,
  `tenant_id` — in place of today's `IMPLICIT_OPERATOR`, which under tenancy is the hub administrator and a member of
  every tenant, far more than a marker write needs. The principal's tenant is the request's tenant, a header naming
  another is refused `400` as for any machine credential, and the principal grants the marker write for its own triple
  and nothing else.
- **Handlers never see an unscoped store.** A handler reads `TenantRequest.services`; nothing on `app.state` exposes a
  store that is not opened for a scope. `bzh:controller-read-only` holds as today — the services a controller is handed
  are read repositories opened for its tenant.
- **The resolved tenant is recorded on every request.** Because no URL carries it, the dependency binds `tenant_id` into
  the request's structured-log context and sets it as the `blizzard.tenant` attribute on the request's span, beside the
  caller attributes `annotate_caller` already sets. Every log line and span a request produces names its tenant's id.

## The event stream

The in-process `EventBroker` (`hub/events/broker.py` over `foundation/events/broker.py`) becomes one broker per tenant:

- `TenantBrokers.for_tenant(tenant_id) -> EventBroker`, created on first use and dropped when the tenant is deleted.
  Each broker keeps its own bounded ring and its own monotonic ids, so one tenant's churn never evicts another tenant's
  replay tail, and a `Last-Event-ID` is meaningful only within its tenant.
- `GET /api/events/stream` subscribes to the resolved tenant's broker after `require(FLEET_VIEW)`, its tenant named by
  `X-Blizzard-Tenant` like every other people route. The board's stream transport is fetch-based
  (`web/projects/fleet/src/lib/sse/sse.service.ts`, `fetchEventSourceFactory`), so it sends the header from the same
  interceptor as every API call, and the stream needs no query-parameter exception.
- Every `publish_*` call site receives its broker from `TenantServices`, never from `app.state.events`, so a publish can
  only reach the tenant whose write produced it.

## Runner-facing additions

Two things a runner sees grow a tenant, both additively, so that a runner can name the tenant it serves and its set-up
can refuse to cross tenants. What the runner does with them is the [runner slice](../../runner.md)'s.

### What registration answers

`RunnerRegistrationResponse` (`wire/runner.py`), which already carries the runner's hub-minted `runner_id`, gains
`tenant_id` and `tenant_name`, read from the registration the runner's token resolved to. Registration doubles as the
heartbeat, so a runner learns of a tenant's rename on its next tick. Nothing a runner sends changes, and a runner built
before this slice ignores both fields.

### What the identity route answers

`GET /api/fleet/identity` — which answers a runner's bearer token with its id and name, and tells an unknown token, a
revoked one, and a retired runner apart — gains `tenant_id` and `tenant_name` beside them, read the same way.
`runner init` asks it before adding anything, so it can see that a token already in `.env` belongs to a different tenant
than the one it was asked to add the runner in, and stop ([runner slice](../../runner.md)).

### Signing in to a runner

`GET /api/auth/authorize?client=<runner_id>` (`hub/api/idp.py::authorize`) mints the short-lived token a person presents
to a runner's own web surface. It stays a hub-level route, and the tenant it signs the person into is the runner's,
never one the person names:

- **The client settles the tenant.** `client` resolves through `HubScopedReads` to the runner's registration and its
  `tenant_id`, as the runner's own bearer token does.
- **The person must be a member there.** A signed-in person with no membership in the runner's tenant is answered with
  the same undifferentiated `400` as an unknown client or an unregistered redirect URI, so probing `client` reveals
  nothing about runners in tenants the caller cannot enter. The hub administrator is no exception
  ([identity.md](./identity.md) §Naming the tenant). Under `auth.mode = "none"` the implicit operator is a member of
  every tenant and always passes.
- **The claims carry the membership.** `role` becomes the person's membership role in the runner's tenant, since
  `users.role` no longer exists to supply it. `sub`, `username`, `email`, `aud` (the runner's id), and `jti` are
  unchanged, and no claim is added. The runner already accepts a token only when `aud` is its own id, and a runner id
  belongs to exactly one tenant, so a token the hub minted for it is always a token for its tenant.

## Compatibility

- **The fleet wire only gains fields** (`bzh:fleet-wire-additive`). No `/api/fleet/*` path, request, or enum changes,
  runners send nothing new, and the only responses that grow are registration's and the identity route's (§Runner-facing
  additions). A runner built before this slice registers, claims, reports, and streams transcripts against a tenanted
  hub unchanged, its tenant settled by its token. `blizzard:wire-compat` passes without an acknowledged break.
- **Operator routes keep their paths, and responses only gain fields.** `/api/me` gains
  `tenants: [{tenant_id, name, role}]` and `hub_admin`. A client that sends no header keeps working for any caller with
  one membership, which is every caller of a carried-over hub.
- **Deprecate, never remove.** A route this slice supersedes stays as a deprecated alias of its replacement, acting in
  the request's tenant, until a later change removes it: `/api/users` and `POST /api/users/{user_id}/role` alias the
  members routes ([administration.md](./administration.md)), and `POST /api/graphs/sync` keeps syncing the request's
  tenant beside the hub-level sync. A breaking change is acceptable only where no alias can be kept, because one
  operator controls every hub and runner through this transition and redeploys them together.
- **Export operations move up a level.** `/api/egress/*` and `/api/traces/*` keep their paths but join the hub-level
  family under `EXPORT_ADMIN` ([identity.md](./identity.md) §Permissions), because the cursors they read and move are
  shared by every tenant ([store.md](./store.md) §Fact egress). A tenant member who could read their status before no
  longer can.
- **Status codes gain meanings.** `404` additionally covers "a tenant you cannot enter"; `409 tenant_required` is new
  and appears only when a person with several memberships names none; `400` covers a header that disagrees with a
  machine credential.
