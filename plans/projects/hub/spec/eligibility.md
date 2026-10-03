# Eligibility contract

## The served declaration

A runner declares the projects it serves at registration. `RunnerRegistrationRequest` (`wire/runner.py`) gains:

```python
projects: list[str] | None = None   # project ids or names in the runner's tenant
```

`None` and `[]` mean different things, the distinction `subscriptions` already draws:

| Sent       | Effect on the stored declaration                                                 |
| ---------- | -------------------------------------------------------------------------------- |
| `None`     | Unchanged. A runner built before this slice sends nothing and keeps what it had. |
| `[]`       | Replaced with none: the runner serves nothing.                                   |
| `["a", …]` | Replaced with exactly these projects.                                            |

Each entry is a project id or a name; ids are the durable form and names the friendly one a runner's own configuration
may use. `FleetService.register` (`hub/domain/registry.py`) resolves every entry by [surfaces.md](./surfaces.md)
§Resolving a project at registration and stores the result — `runner_registrations` gains `projects Text NULL`, a JSON
list of `{entry, project_id}` pairs, where `entry` is what the runner sent and `project_id` is what it resolved to. A
first registration that sends `None` stores none.

Because the stored value is the id, renaming a project never strands a runner: the declaration keeps serving the project
until the runner next re-registers, and a runner whose configuration still carries the old name keeps resolving it for
as long as the old name resolves. Registration never rejects over the declaration, because it doubles as the heartbeat
(`api/fleet.py::register_runner`): an entry that resolves to no live project of the tenant is stored with a null
`project_id`, serves nothing, and is surfaced on the board as an unresolved declaration naming the entry. It is
re-resolved at every registration, so it starts serving once a project answers to it. Operators who want a declaration
that no rename or reuse of a name can affect configure ids.

The field is additive on the runner-reached wire (`bzh:fleet-wire-additive`): an older runner neither sends nor reads
it.

## The matched peek

`POST /api/fleet/queue/peek` narrows the ready order to the calling runner's served projects *before*
`select_matched_entry` (`hub/domain/queue.py`) scans it. The runner's stored declaration is read by its authenticated
principal, never from the request body. `HOLD` and `PASS_OVER` then apply to the narrowed order exactly as today: a
chunk of a project the runner does not serve is never its head, so `HOLD` cannot stall a runner on another project's
work. Capability eligibility (`EligibilityCheck`) and dependency blocking are unchanged and apply within the narrowed
order.

The unmatched reads — `GET /api/queue`, `GET /api/fleet/queue/peek`, `GET /api/backlog` — take an optional project
filter ([surfaces.md](./surfaces.md)) and otherwise show the whole tenant's order. Ranking stays one order per list per
tenant (`bzh:ranking-is-per-list`); the project filter selects from it and never reranks.

## The claim recheck

`ClaimService.claim` (`hub/domain/claim.py`) rechecks the served declaration beside the paused, terminal, dependency,
and capability guards: claiming a chunk of a project the registration does not serve raises `ClaimDeniedIncompatible`,
answered with the existing `RouteClaimIncompatibleDenial` `409` body and a `detail` naming the project. Reusing the
existing denial keeps the claim response within what every deployed runner already parses (`bzh:fleet-wire-additive`).

## On the board

A runner's row shows the projects it serves. A runner that serves nothing, or holds an unresolved declaration, is marked
as such, so an idle runner reads as unassigned rather than as starved. A project that no live runner serves is marked on
its own surfaces ([surfaces.md](./surfaces.md)), so its waiting work reads as unserved rather than as slow.
