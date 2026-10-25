# Surfaces the ports need

## The tenant on every pass's lines

A per-tenant background pass writes its lines with no `tenant_id`. The hub's sweeps run from `Sweep` in
`src/blizzard/hub/app.py`, one reconciler per sweep, each looping over the hub's tenants itself, and only a request
binds the tenant to its lines, through `structlog.contextvars.bind_contextvars` in `tenant_bound`
(`src/blizzard/hub/api/auth_session.py`). A sweep's failure and skip lines pass `tenant=` as a keyword, and its summary
lines carry counts.

Each reconciler that loops over tenants binds the tenant for the length of that tenant's work, with
`structlog.contextvars.bound_contextvars(tenant_id=<tenant>)` around the loop body, so every line written inside it, the
reconciler's own and anything it calls, carries `tenant_id` exactly as a request's lines do. The `tenant=` keyword those
lines pass today is removed in the same change, since the bound value replaces it. The reconcilers it applies to are
each one that iterates tenants:

| Sweep                 | Reconciler                  | File                                                            |
| --------------------- | --------------------------- | --------------------------------------------------------------- |
| Forge status          | `AnnotationReconciler`      | `src/blizzard/hub/domain/observability/forge_status.py`         |
| Transcript derivation | `EventDerivationReconciler` | `src/blizzard/hub/domain/observability/analytics/derivation.py` |
| Close-intent drain    | `CloseDrainSweep`           | `src/blizzard/hub/domain/work_items/closure.py`                 |
| Tenant teardown       | `TenantTeardownSweep`       | `src/blizzard/hub/domain/tenancy/teardown.py`                   |

A pass's own summary line, `"sweep pass completed"` from `Sweep`, is hub-wide, written outside any tenant's loop, and
stays untagged. So do the lines of `TraceExportSweep` and `EgressSweep`, which export hub-wide and loop over no tenant.
A unit test per reconciler drives one pass over two tenants and asserts every line written inside each tenant's work
carries that tenant's id, reading the lines through `capture_logs(processors=[merge_contextvars])`, since a bare
`capture_logs()` never sees a bound value. The `tenant=` keyword's removal renames a field an operator may filter on,
and no contract document covers the hub's log, so the change carries a `changed` change note. An operator reading one
tenant's activity out of a shared hub's log gets the passes now, not only the requests.

## Sweeps on request

Every hub runs its sweeps on timers, from `Sweep` in `src/blizzard/hub/app.py`: forge-status annotation every
`annotation_interval_seconds`, 120 by default, transcript derivation and the close-intent drain every 60 seconds, tenant
teardown every 30, and trace export and egress when they are configured, each with its first pass at boot. A test cannot
know when a pass will run, so a test whose subject is a sweep waits on its log lines or on the clock, and a pass over a
test's data mostly runs after the test has ended.

Each sweep gains a mode in the hub file, declared in `blizzard.hub.config_keys` so it reaches
`contracts/config/hub.json`:

```toml
[sweeps.forge_status]
mode = "timed"   # the default; "stepped" starts no timer, and a pass runs only when a route asks
```

The sweeps are `forge_status`, `event_derivation`, `close_drain`, `tenant_teardown`, `trace_export`, and `egress`. Their
trace spans keep the names they are documented under, so transcript derivation's is still `sweep transcript_events` and
the close-intent drain's `sweep work_closure`. A stepped sweep starts no timer and runs no boot pass. Everything else
about it is as when timed.

Two routes run a pass. Each holds its request open for the whole pass and answers once it has completed, so a caller
that gets its answer can read every effect the pass had:

| Route                                  | Runs                                                                                                                                                                                                                              | Who may call      |
| -------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------- |
| `POST /api/sweeps/{name}/passes`       | One pass of a per-tenant sweep over the request's tenant alone. Refused with `422` for `trace_export` and `egress`, which are hub-wide, and for `tenant_teardown`, whose tenants are closed and so can never be the request's own | `analytics:admin` |
| `POST /api/admin/sweeps/{name}/passes` | One pass over every tenant, as a timed pass runs, and the only route for the hub-wide sweeps and for tenant teardown                                                                                                              | `tenant:admin`    |

The per-tenant route asks for `analytics:admin`, which already gates the one route that forces a pass today, the
transcript re-derivation. A new permission would add a value a released client does not know.

Each answers `200` with `{ "started_at", "finished_at", "outcome": "completed" | "failed", "error"? }`, its `outcome` an
open set. A timed pass's failure is logged and swallowed, and a requested one reports it to its caller: a pass over one
tenant answers `failed`, with the error's message, when that tenant's work fails, and a pass over every tenant, whose
sweep already catches each tenant's fault and moves on, answers `failed` only when the pass itself raises. Both routes
serve in either mode. A pass over one tenant holds a lock for its sweep and tenant, and a pass over every tenant, timed
or requested, holds its sweep's lock across all of them; a request meeting a held lock answers `409` with
`{ "code": "sweep_in_progress" }` at once, and a timed pass that meets one skips its interval and logs that it did. Two
tests stepping the same sweep in their own tenants never meet, and a test that meets the refusal raced a pass it should
not have shared. The locks are in memory and end with the process.

Each per-tenant reconciler's loop body, which [the tags above](#the-tenant-on-every-passs-lines) already bind to its
tenant, becomes a pass for one tenant, and its timed pass loops that over every tenant, so a requested pass and a timed
one run the same code. Component tests cover each mode, both routes, the lock's refusal, a failed pass's answer, and a
per-tenant pass touching no other tenant.

In the harness, every profile, the default among them, sets every sweep `stepped`, so no pass runs on a shared hub
except one a test asks for. `api.sweep(name)` calls the per-tenant route in the test's tenant and fails the test unless
the pass completed; `api.sweep(name, { allTenants: true })` calls the admin route and is refused below `fresh-hub`. A
test whose subject is a sweep's timer takes `fresh-hub` and sets that sweep `timed` through its `hubConfig`
([harness.md](./harness.md#a-fresh-hubs-own-configuration)), so timers run only on hubs of their own.

## Joining the stream from now

`GET /api/events/stream` resumes from `Last-Event-ID`, or `?last_event_id=`, and with neither, or with a cursor past the
broker's latest id, replays the tenant broker's whole ring, up to 256 events (`foundation/events/stream.py`). A client
that wants only what happens from now on has no way to say so, so the service tests drain the replay for a fixed
`settle` and an operator's client resuming after a long absence takes history it does not want.

The stream gains `?from=latest`. The stream subscribes, as it does today, then skips the replay entirely, and sends only
events published after the subscription; the first frame, the `: blizzard hub event stream` comment, is unchanged, so a
client knows when it is joined. Subscribing before skipping means no event published between the request and the
subscription is lost and none from before it arrives, so the broker records its latest id under the same lock it
registers the subscriber in, and nothing can be published between the two. `from` takes only `latest`; any other value
answers `422`. A `Last-Event-ID` header wins over `from=latest`, because a reconnecting client, a browser's
`EventSource` or the harness's reader, sends that header on the same URL it first opened, and must resume rather than be
refused. Only `from=latest` and `?last_event_id=` together in the query answer `422`, since the two ask for different
things. The runner's stream, which shares `Stream`, gains the same parameter.

The frame shapes in `contracts/sse/` are unchanged. The stream is excluded from the OpenAPI document, and `docs/` has no
event-stream reference, so the parameter is documented in `contracts/sse/README.md` and, for the runner's stream, in
`docs/deployment/runner-doors.md`. `tests/test_sse_contract.py` checks frame shapes and never opens a stream, so each
answer's case goes in `tests/test_foundation_events.py` and `tests/test_events.py`.

## A configurable login throttle

The hub throttles sign-ins per client address, `IpThrottle` in `src/blizzard/hub/auth/throttle.py`: ten tokens, one
refilled every six seconds, spent at `authorize` and `callback`, answering `429` when empty. It is built in
`src/blizzard/hub/composition.py` with those defaults and nothing reaches it from configuration, so an operator cannot
tune it and a hub signing many people in from one address cannot lift it.

The hub file gains a table, declared in `blizzard.hub.config_keys` so it reaches `contracts/config/hub.json`:

```toml
[auth.login_throttle]
enabled = true        # false admits every sign-in unthrottled
capacity = 10         # tokens per client address
refill_seconds = 6    # seconds per token refilled
```

The defaults are today's values, so a hub whose file names none of them throttles exactly as now. `capacity` and
`refill_seconds` must be positive, and a hub refuses a non-positive one at boot. `enabled = false` builds no throttle at
all, and the hub logs once at boot that sign-ins are unthrottled, so a production hub left that way says so. The
deployment documentation names the table beside `trusted_proxies`, which still decides what a client address is.
Component tests cover each key's effect and the boot refusal.

## Keyed sign-in

The stub IdP holds one identity for the whole process, `IdpState.profile` in
`blizzard-mock/src/blizzard_mock/idp/domain/state.py`, set by `PUT /_levers/profile` and bound to a code when
`authorize` runs. On a shared hub, two workers signing in at once race on it, and one comes back as the other's person.

`blizzard-mock`'s IdP learns to hold many identities and to choose among them by `login_hint`, the standard OIDC
parameter, so it stays shaped like a real provider:

- **`POST /_levers/identities`**, body a `Profile`, registers that identity under its `email`, lower-cased. Registering
  an email again replaces it. `GET /_levers/identities` lists them, and `POST /_levers/reset` clears them with
  everything else.
- **`GET /oidc/authorize`** and **`GET /login/oauth/authorize`** read `login_hint`. With a hint that names a registered
  identity, the code they mint is bound to that identity. With a hint that names none, they redirect back with
  `error=access_denied`, as a provider does for a refused account. With no hint, they bind the code to
  `IdpState.profile`, exactly as today, so every existing test and lever is unchanged.

The hub is untouched: it sends no `login_hint`, and the harness adds it ([harness.md](./harness.md#people)).
`blizzard-mock`'s own tests cover a hinted sign-in, two identities signed in in turn and each coming back as itself, an
unknown hint, and an unhinted sign-in unchanged.

## Anything a port proves missing

A test that cannot be written against these and the foundation's surfaces finds its state through the real flow, or
earns a surface an operator would want too, specified here in the change that adds it. A test-only route behind a
configuration flag is the last resort, and none is planned.
