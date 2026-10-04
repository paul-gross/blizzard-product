# API contract

Every route resolves a tenant before it touches tenant-owned state, and it resolves it in exactly one place: a router
dependency, never a handler. No API route changes shape and nothing is mounted twice; the tenant travels beside the
path, not in it. The runner-reached wire does not change.

## Route families

| Family    | Paths                                                                                                  | Tenant comes from                                                    |
| --------- | ------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------- |
| Hub-level | `/api/health`, `/api/ready`, `/api/me`, `/api/auth/*`, `/api/system-artifacts*`, `/api/admin/tenants*` | none — these routes hold no `TenantStores`                           |
| People    | `/api/…` — every operator router, at its current path                                                  | the request's resolution order below                                 |
| Fleet     | `/api/fleet/*` — unchanged paths, requests, and responses                                              | the machine credential the call already carries, never anything else |

## Resolution order

Each request resolves its tenant by the first rule that applies:

1. **A machine credential decides.** A runner bearer token, a route token, a marker token, and an API token are each
   bound to exactly one tenant, resolved through `HubScopedReads`. The credential's tenant is the request's tenant. A
   request that also carries an `X-Blizzard-Tenant` header naming a different tenant is refused `400`; one naming the
   same tenant is accepted.
2. **A person names it.** An `X-Blizzard-Tenant: <id or name>` header is resolved by the naming rule below and checked
   against the caller's memberships. A value that resolves to nothing, or to a tenant the caller cannot enter, answers
   `404` — the two are indistinguishable.
3. **One membership settles it.** With no header, a caller holding exactly one membership acts in that tenant.
4. **Otherwise** the request answers `409 tenant_required`.

Under `auth.mode = "none"` the implicit operator is a member of every tenant ([identity.md](./identity.md)), so rule 3
applies only while the hub holds exactly one tenant.

### Naming a tenant

The header value, the stream's query parameter, the board's `/t/{tenant}/…` route, and the CLI's `--tenant` accept
either form:

1. **An id.** A value beginning `ten_` is looked up as a `tenant_id`. The id is always accepted, everywhere, for as long
   as the tenant exists.
2. **A current name.** Otherwise the value is matched, case-insensitively, against live tenants' names.
3. **A former name.** Failing that, it is matched against `tenant_former_names`. A former name keeps resolving to the
   tenant that last held it until another tenant takes that name, or the tenant is deleted. The board answers a former
   name in its page route with a `308` redirect to the current name; the API serves the request against the resolved
   tenant, so a script written against an old name keeps working.

Whatever form was presented, resolution yields the `tenant_id`, and nothing after this point sees the name.

### Why the session holds no current tenant

A session belongs to a person, not to a tenant ([identity.md](./identity.md)), and it is shared by every tab and window
that person has open. A "current tenant" stored on it would be one value for all of them: switching tenant in one tab
would silently move every other open tab into the new tenant, and the next write from an older tab would land somewhere
its page never showed. The tenant is therefore stated per request, by whatever the request came from — the tab's own
route, the CLI's own context — and two tabs on two tenants never interfere.

### Where people's clients get the header

- **The board** keeps shareable page URLs of the form `/t/{name}/…`. The SPA resolves its route's `{tenant}` once per
  navigation and sends the tenant's id as `X-Blizzard-Tenant` on every API call that page makes, from one HTTP
  interceptor. Links it writes use the tenant's current name. `/` redirects to the caller's only tenant, to a tenant
  picker when they hold several, or to a "no tenants yet" page when they hold none.
- **The CLI** sends the header from `--tenant`, else from the context's saved default tenant, which it stores by id so a
  rename never breaks it.

## The dependency

`get_services(request) -> HubServices` (`hub/api/deps.py`) is replaced, for every tenant-owned route, by a router-level
dependency that produces one value per request:

```python
@dataclass(frozen=True)
class TenantRequest:
    identity: ResolvedIdentity        # identity.tenant == scope.tenant
    scope: StoreScope                 # the tenant, plus the project lens a projects route adds
    services: TenantServices          # built over HubCore.open(scope)
```

- **People.** `require(permission)` (`hub/api/auth_session.py`) resolves the identity as today, applies the resolution
  order, checks the membership, expands the membership role, and only then checks `permission`, which answers `403` as
  today.
- **Fleet.** `require_runner_principal` (`hub/api/auth.py`) resolves the bearer token through `HubScopedReads` to
  `(runner_id, workspace_id, tenant_id)`, where `runner_id` is the id the hub minted when the runner was added;
  `RunnerPrincipal` gains `tenant_id`, and `FleetRequest` builds its services from that tenant's `TenantStores`.
  Route-token and marker-token resolution do the same through the chunk their token is bound to.
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
- `GET /api/events/stream` subscribes to the resolved tenant's broker after `require(FLEET_VIEW)`.
- Every `publish_*` call site receives its broker from `TenantServices`, never from `app.state.events`, so a publish can
  only reach the tenant whose write produced it.

**The one exception to header-only naming.** The board subscribes with the browser's native `EventSource`
(`web/projects/fleet/src/lib/sse/sse.service.ts`), which cannot send custom headers. The stream endpoint, and no other
route, therefore also accepts `?tenant=<id or name>`, resolved by the same naming rule and slotted into the resolution
order where the header would be. When both the header and the parameter are present and resolve to different tenants,
the request answers `400`.

## Compatibility

- **The fleet wire is untouched** (`bzh:fleet-wire-additive`). No `/api/fleet/*` path, request, response, or enum
  changes, and runners send nothing new; a runner built before this slice registers, claims, reports, and streams
  transcripts against a tenanted hub unchanged, its tenant settled by its token. `blizzard:wire-compat` passes without
  an acknowledged break.
- **Operator routes keep their paths, and responses only gain fields.** `/api/me` gains
  `tenants: [{tenant_id, name, role}]` and `hub_admin`; nothing is removed or renamed. A client that sends no header
  keeps working for any caller with one membership, which is every caller of a carried-over hub.
- **Status codes gain meanings.** `404` additionally covers "a tenant you cannot enter"; `409 tenant_required` is new
  and appears only when a person with several memberships names none; `400` covers a header that disagrees with a
  machine credential or with the stream's query parameter.
