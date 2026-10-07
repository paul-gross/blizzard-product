# Eligibility contract

## The served declaration

A runner declares the projects it serves at registration. `RunnerRegistrationRequest` (`wire/runner.py`) gains:

```python
projects: list[str] | None = None   # project ids or slugs in the runner's tenant
```

`None` and `[]` mean different things:

| Sent       | Effect on the stored declaration                                                 |
| ---------- | -------------------------------------------------------------------------------- |
| `None`     | Unchanged. A runner built before this slice sends nothing and keeps what it had. |
| `[]`       | Replaced with none: the runner serves nothing.                                   |
| `["a", …]` | Replaced with exactly these projects.                                            |

Keeping the stored value when `None` is sent is new. A runner's row is created when it is added, before it ever
registers (`RunnerRegistryStore.add`, behind `POST /api/runners`), and each registration after that updates the row
(`FleetService.register` in `hub/domain/execution/fleet.py`, over `record_registration` in
`hub/store/internal/runner_registry_store.py`), overwriting every field a runner reports, `subscriptions` included: a
`None` roster is stored as null. `record_registration` therefore leaves `projects` untouched when the request carries
none, while every other reported field is overwritten as before.

Each entry is a project id or a slug; ids are the durable form and slugs the friendly one a runner's own configuration
may use. `FleetService.register` (`hub/domain/execution/fleet.py`) resolves every entry by [surfaces.md](./surfaces.md)
§Resolving a project at registration and stores the result — `runner_registrations` gains `projects Text NULL`, a JSON
list of `{entry, project_id}` pairs, where `entry` is what the runner sent and `project_id` is what it resolved to. Its
first value is set when the runner is added (§The declaration a runner is added with).

Because the stored value is the id, changing a project's slug never strands a runner: the declaration keeps serving the
project until the runner next re-registers, and a runner whose configuration still carries the old slug keeps resolving
it for as long as the old slug resolves to that project. Registration never rejects over the declaration, because it
doubles as the heartbeat (`api/fleet.py::register_runner`): an entry that resolves to no project of the tenant is stored
with a null `project_id`, serves nothing, and is surfaced on the board as an unresolved declaration naming the entry. It
is re-resolved at every registration, so it starts serving once a project answers to it. Operators who want a
declaration that no slug change or reclaimed slug can affect configure ids.

Registration answers with what it stored. `RunnerRegistrationResponse` gains `projects`: one entry per declared entry,
`{entry, project_id, slug, name}`, with the last three null for an entry that resolved to nothing. The runner keeps the
latest answer in its store, as it keeps the hub's pause, so its own status can show what it serves and mark an
unresolved entry without asking the hub again.

Both fields are additive on the runner-reached wire (`bzh:fleet-wire-additive`): an older runner neither sends the
request field nor reads the response field.

## The declaration a runner is added with

A runner that has never heard of projects will never declare one, so its declaration cannot wait for its first
registration. The carry-over records one for every runner that exists when it runs ([carry-over.md](./carry-over.md)),
but runners keep being added after it: a feature environment's runner adds itself again after every reset of its hub's
data, and the mock fleet and the test harnesses add runners as they go. Each of those would register with nothing
declared and serve nothing until its runner was rebuilt. The declaration therefore starts when the runner is added.

- **Adding may name the projects.** `RunnerAddRequest` (`wire/runner.py`) gains `projects: list[str] | None = None`, ids
  or slugs resolved as at registration, and `blizzard hub runner add` gains a repeatable `--project`. An entry that
  resolves to nothing is refused `422`, since adding, unlike registration, is no heartbeat and can afford to say no.
- **Naming none, the tenant's only project is served.** When the tenant holds exactly one project, the runner is added
  serving it, its entry stored as the project's id so that no later slug change touches it — a project is assumed only
  where nothing else could be meant, as at ingest ([ingest.md](./ingest.md) §Resolution, in order). When the tenant
  holds several, the runner is added serving nothing, and the board marks it unassigned (§On the board).
- **Registration takes over from there.** A runner that declares projects replaces the added declaration on its next
  registration, under the table above; one that sends `None` keeps it.

`POST /api/runners` sits on the surface `blizzard:wire-compat` diffs, because `runner init` adds its runner through it;
the field is optional, so a `runner init` built before this slice adds through it unchanged.

## The matched peek

`POST /api/fleet/queue/peek` narrows the ready order to the calling runner's served projects *before*
`select_matched_entry` (`hub/domain/operations/queue.py`) scans it. The runner's stored declaration is read by its
authenticated principal, never from the request body. `HOLD` and `PASS_OVER` then apply to the narrowed order exactly as
today: a chunk of a project the runner does not serve is never its head, so `HOLD` cannot stall a runner on another
project's work. Capability eligibility (`EligibilityCheck`) and dependency blocking are unchanged and apply within the
narrowed order.

The unmatched reads — `GET /api/queue`, `GET /api/fleet/queue/peek`, `GET /api/backlog` — take an optional project
filter ([surfaces.md](./surfaces.md)) and otherwise show the whole tenant's order. Ranking stays one order per list per
tenant (`bzh:ranking-is-per-list`); the project filter selects from it and never reranks.

## The claim recheck

`ClaimService.claim` (`hub/domain/execution/claim.py`) rechecks the served declaration in `ClaimAdmission`, beside the
paused, terminal, dependency, and capability guards: claiming a chunk of a project the registration does not serve
raises `ClaimDeniedIncompatible`, answered with the existing `RouteClaimIncompatibleDenial` `409` body. The error and
the body gain a `reason` — `capability`, the default and today's meaning, or `project` — and a project denial's `detail`
names the project. A new `409` shape would break every deployed runner, which tells the four `409` bodies apart by their
fields (`runner/hub/internal/http_hub.py`) and would read an unknown one as a lost race; the incompatible shape already
means "do not take this chunk, move on", which is right for a project denial too. A project denial only happens in the
window between a peek and a claim, since the peek is already narrowed to the served projects, so a runner built before
this slice handling it as a capability denial loses nothing (`bzh:fleet-wire-additive`).

## On the board

A runner's row shows the projects it serves. A runner that serves nothing, or holds an unresolved declaration, is marked
as such, so an idle runner reads as unassigned rather than as starved. A project that no live runner serves is marked on
its own surfaces ([surfaces.md](./surfaces.md)), so its waiting work reads as unserved rather than as slow.
