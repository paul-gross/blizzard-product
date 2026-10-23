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

| Sweep                 | Reconciler                        | File                                                            |
| --------------------- | --------------------------------- | --------------------------------------------------------------- |
| Forge status          | `AnnotationReconciler`            | `src/blizzard/hub/domain/observability/forge_status.py`         |
| Transcript derivation | `EventDerivationReconciler`       | `src/blizzard/hub/domain/observability/analytics/derivation.py` |
| Close-intent drain    | `CloseDrainSweep`                 | `src/blizzard/hub/domain/work_items/closure.py`                 |
| Tenant teardown       | `TenantTeardownSweep`             | `src/blizzard/hub/domain/tenancy/teardown.py`                   |
| Trace export, egress  | `TraceExportSweep`, `EgressSweep` | where each iterates tenants, if it does                         |

A pass's own summary line, `"sweep pass completed"` from `Sweep`, is hub-wide, written outside any tenant's loop, and
stays untagged. A unit test per reconciler drives one pass over two tenants and asserts every line written inside each
tenant's work carries that tenant's id. An operator reading one tenant's activity out of a shared hub's log gets the
passes now, not only the requests.

## Joining the stream from now

`GET /api/events/stream` resumes from `Last-Event-ID`, or `?last_event_id=`, and with neither, or with a cursor past the
broker's latest id, replays the tenant broker's whole ring, up to 256 events (`foundation/events/stream.py`). A client
that wants only what happens from now on has no way to say so, so the service tests drain the replay for a fixed
`settle` and an operator's client resuming after a long absence takes history it does not want.

The stream gains `?from=latest`. The stream subscribes, as it does today, then skips the replay entirely, and sends only
events published after the subscription; the first frame, the `: blizzard hub event stream` comment, is unchanged, so a
client knows when it is joined. Subscribing before skipping means no event published between the request and the
subscription is lost and none from before it arrives. `from` takes only `latest`; any other value answers `422`.
`from=latest` with a `Last-Event-ID` answers `422`, since the two ask for different things. The runner's stream, which
shares `Stream`, gains the same parameter.

The frame shapes in `contracts/sse/` are unchanged. The stream is excluded from the OpenAPI document, so the parameter
is documented where the stream is, in `docs/`'s event-stream reference, and `tests/test_sse_contract.py` gains a case
for each answer.

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
