# Surfaces contract

## The rule for a project parameter

- **Creating a project-owned record names its project.** Scopes, routines, operator-origin garden proposals, and links
  to sources and repositories are created under a project, and the call is refused without one. Work sources and
  repositories themselves are tenant records, created under live-config's verbs without a project.
- **Reading takes an optional project filter.** Omitted means the whole tenant — what every caller reads today. The
  filter is an ordinary argument, `project: ProjectId | None`, on the read repository methods that list project-owned
  rows or rows reached through a chunk; the multi-tenancy store seam scopes by tenant only and knows nothing of projects
  ([store.md](../../../multi-tenancy/hub/spec/store.md) §The scoped store seam).
- **Ingest** is the one write that may infer its project, under [ingest.md](./ingest.md)'s rules.
- **Managing projects takes the tenant `admin` role.** Creating and editing a project, and creating or removing its
  source and repository links, require `CONFIG_EDIT` — the permission `epic:live-config`'s record verbs already take,
  held by the `admin` bundle alone (`auth_core.ROLE_PERMISSIONS`). Reading projects and links takes `FLEET_VIEW`.
  Records created under a project keep the permission they take today: a scope or routine `GRAPH_EDIT`, a garden
  proposal `CHUNK_CONTROL`.

The tenant every call acts within is resolved before any of this, under the multi-tenancy contract
([api.md](../../../multi-tenancy/hub/spec/api.md)): an API call takes it from its machine credential, else from the
`X-Blizzard-Tenant` header, else from the caller's single membership, else answers `409 tenant_required`. No API path
carries a tenant. A project is always resolved within the tenant so resolved.

## Resolving a project

Every place that accepts a project — board URL segment, API path (`/api/projects/{project}/…`), `?project=`,
`--project`, `ChunkIngestRequest.project`, a runner declaration — accepts either its id or its slug, and resolves to the
id at the boundary; nothing past the boundary handles a slug. A display name is never accepted.

1. A value of the `proj_<ulid>` form is an id and resolves to that project, whatever its slug.
2. Otherwise it is matched exactly against the slugs of the tenant's projects. A value outside the slug alphabet matches
   nothing.
3. Otherwise it is matched against `project_former_slugs` ([model.md](./model.md)), and resolves to the project that
   left it until some project of the tenant takes that slug. The board answers a former slug with a redirect to the same
   route under the current slug; the API serves the request against the resolved project without redirecting, so a
   script or a runner configuration written against an old slug keeps working.
4. Otherwise it resolves to nothing, answered as the surface answers an unknown project — `404` on a path, `422` in a
   body or query.

A tenant is named by its id alone ([api.md](../../../multi-tenancy/hub/spec/api.md)); a project adds a slug because
people type and read it in every lensed URL and every `--project`. The board always writes links with the current slug;
the CLI saves a default project by id, so a slug change never breaks a saved context.

## API

Projects follow `epic:live-config`'s configured-record verbs and patch semantics
([api.md](../../../../delivered/live-config/spec/api.md)):

| Route                                                                           | Verbs                                                     |
| ------------------------------------------------------------------------------- | --------------------------------------------------------- |
| `/api/projects`                                                                 | `GET` list, `POST` create (`slug`, `name`, `description`) |
| `/api/projects/{project}`                                                       | `GET`, `PATCH` (slug, name, description)                  |
| `/api/projects/{project}/source-links`                                          | `GET`; `PUT /{source}` links; retire unlinks              |
| `/api/projects/{project}/repository-links`                                      | `GET`; `PUT /{repository}` links; retire unlinks          |
| `/api/projects/{project}/scopes`, `/routines`, `/findings`, `/garden-proposals` | the existing gardening verbs, moved under the project     |
| `/api/projects/{project}/runs`, `/runs/{chunk_id}`                              | the garden run list and a run's delta, within the project |

The tenant-wide reads gain `?project={project}`: `GET /api/chunks`, `/api/chunk-counts`, `/api/queue`, `/api/backlog`,
`/api/runs`, `/api/routines/trend`, `/api/routines/proposal-counts`, `/api/spend`, every `/api/analytics/*` read
(counts, durations, spend, outcomes, events), the event feed, and the SSE stream's subscription, which filters
chunk-scoped events by the chunk's project and passes runner-scoped events for runners that serve it. `GET /api/runners`
gains each runner's served declaration, and `POST /api/runners` an optional `projects` to add a runner serving
([eligibility.md](./eligibility.md) §The declaration a runner is added with). A project's own view carries `project_id`,
`slug`, and `name` as separate fields, and response models that describe a chunk gain `project_id`, `project_slug`, and
`project_name`; every addition is optional on the wire and additive.

The existing un-nested gardening routes (`/api/scopes`, `/api/routines`, `/api/findings`, `/api/garden-proposals`,
`/api/runs`) remain as tenant-wide reads with the same `?project=` filter, so the all-projects doorway reads every
garden in one call. Every route that names a scope by slug or a routine by name — scope and routine names are unique
only within a project ([model.md](./model.md) §Scope and routine identity) — gains its project:

| Today                                                               | Becomes                                                    |
| ------------------------------------------------------------------- | ---------------------------------------------------------- |
| `GET`/`PATCH /api/scopes/{slug}`, `GET /api/scopes/{slug}/routines` | the same under `/api/projects/{project}/scopes/{slug}`     |
| `POST /api/scopes/{slug}/retire`, `/enable`                         | the same under `/api/projects/{project}/scopes/{slug}`     |
| `POST /api/scopes`, `POST /api/routines`                            | `POST /api/projects/{project}/scopes`, `/routines`         |
| `PUT`/`DELETE /api/routines/{routine_id}/scopes/{scope_slug}`       | unchanged path; the slug resolves in the routine's project |
| `GET /api/findings?routine=&scope=`                                 | gains `?project=`, required when `routine` or `scope` is   |

A route keyed by `routine_id` or another hub-unique id keeps its path, since the id already settles the project. Nothing
is removed: each un-nested by-name route stays as a deprecated alias, marked deprecated in the OpenAPI document, which
resolves the name across the tenant's projects and serves it when exactly one project holds it, and answers `409` naming
the candidate projects when more than one does. On a carried-over hub, which holds one project, every alias resolves.

## CLI

`blizzard hub project create|list|show|edit` manage projects — `create <slug> [--name <display name>]`, the name
defaulting to the slug, and `edit <project> [--slug …] [--name …] [--description …]` — and
`blizzard hub project link|unlink <project> --source <source> | --repository <repository>` manage their links. Every
verb that creates a project-owned record takes a required `--project`, an id or a slug; every listing verb takes an
optional one; `blizzard hub chunk ingest` takes `--project` under ingest's inference rules, and
`blizzard hub item create` takes a required one. The CLI context that holds a hub login and default tenant also holds an
optional default project, the CLI's own lens, which `--project` overrides on any call. Every call sends the context's
tenant as `X-Blizzard-Tenant` alongside whatever project it names. A verb's help states its effect in operator
vocabulary (`bzh:help-states-effect`, `bzh:operator-vocabulary`).

## The shell lens

The lens is shell state, owned by `fleet/shell` and imported eagerly (`bzh:frontend-eager-shell-entry`). It is derived
from the URL, never held separately, so a refresh or a shared link reproduces it.

```text
/{view}/…                    lens on All
/p/{project_slug}/{view}/…   lens on one project
```

Board URLs carry no tenant: the tab's tenant is the multi-tenancy contract's
([api.md](../../../multi-tenancy/hub/spec/api.md) §Where people's clients get the header), and the `p/{project_slug}`
segment is resolved within it. The segment is this slice's, and holds the project's current slug, resolved by §Resolving
a project — a former slug redirects to the current one, and an id is accepted too. The lens control and the breadcrumb
show the project's display name. The existing route table (`web/projects/hub/src/app/shell/app.routes.ts`) mounts once
at the root and once under `p/{project_slug}`, so every view and every deep link is reachable with and without a lens,
and a chunk's detail opened from a lens keeps it. Changing the lens re-navigates to the same view under the other
prefix.

The lens control sits in the app's view-tab row (`web/projects/hub/src/app/shell/nav/app-nav`, the shell's `shell-nav`
slot) beside the tabs, where the mockups place it, and it narrows what the board and every list the lens touches show;
the header's counts are read with the lens's `?project=` filter. The tenant is not beside it: a person changes tenant
only from the profile menu (`app-nav-menu`), which offers the switcher only to someone with several memberships
(multi-tenancy [api.md](../../../multi-tenancy/hub/spec/api.md) §Where people's clients get the header).

## What each view does with the lens

| View                                     | Lens on a project                                                 | Lens on All                              |
| ---------------------------------------- | ----------------------------------------------------------------- | ---------------------------------------- |
| Board                                    | Chunks of the project; ranking unchanged, filtered                | Every chunk in the tenant                |
| Events                                   | Events of the project's chunks, and of runners that serve it      | Every event                              |
| Gardening                                | That project's scopes, routines, runs and findings, and proposals | A doorway: one card per project, no tabs |
| Admin → Projects                         | Opens that project                                                | Every project                            |
| Admin → Work sources                     | The sources the project links                                     | Every source in the tenant               |
| Admin → Repositories                     | The repositories the project links                                | Every repository in the tenant           |
| Admin → Changes                          | The project's changes and tenant-wide ones                        | Every change                             |
| Graphs, Admin → Members, Admin → Secrets | Unchanged, with a visible note that the lens does not apply here  | —                                        |

Gardening's breadcrumb (`Gardening › {project} › {surface}`) is the lens shown in place: its project crumb switches the
lens and keeps the surface. The [projects-era admin mockup](../../artifacts/projects-admin.html) and the
[per-project gardening mockup](../../artifacts/projects-gardening.html) are the visual baseline; each view's empty and
loading states follow `bzh:frontend-empty-state-gated`, and a change to any of them is proven by a render
(`bzh:visual-change-needs-a-render`).

## Fact egress

Every `epic:fact-egress` row that belongs to a chunk — each `steps`, `invocations`, and `events` row, a `dropped` row
included — gains `project_id`, and beside it `project_slug` and `project_name` as the project was known when the row was
written, read from the chunk's project. The columns sit beside the tenant columns the multi-tenancy
[store contract](../../../multi-tenancy/hub/spec/store.md) §Fact egress adds. They are additive, so each dataset keeps
its major version and the writer widens its `_schema` document in place, with the goldens and the data dictionary moving
beside them (`docs/versioning.md`). A loader groups or filters by project from the rows alone, as it does by graph and
node; a script that must survive a slug change keys on `project_id`.
