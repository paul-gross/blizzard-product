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

The tenant every call acts within is resolved before any of this, under the multi-tenancy contract
([api.md](../../../multi-tenancy/hub/spec/api.md)): an API call takes it from its machine credential, else from the
`X-Blizzard-Tenant` header, else from the caller's single membership, else answers `409 tenant_required`; the live
stream also accepts `?tenant=`. No API path carries a tenant. A project is always resolved within the tenant so
resolved.

## Resolving a project

Every place that accepts a project — board URL segment, API path (`/api/projects/{project}/…`), `?project=`,
`--project`, `ChunkIngestRequest.project`, a runner declaration — accepts either its id or its slug, and resolves to the
id at the boundary; nothing past the boundary handles a slug. A display name is never accepted.

1. A value of the `proj_<ulid>` form is an id and resolves to that project, live or retired, whatever its slug.
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

| Route                                                                           | Verbs                                                      |
| ------------------------------------------------------------------------------- | ---------------------------------------------------------- |
| `/api/projects`                                                                 | `GET` list, `POST` create (`slug`, `name`, `description`)  |
| `/api/projects/{project}`                                                       | `GET`, `PATCH` (slug, name, description), retire, enable   |
| `/api/projects/{project}/source-links`                                          | `GET`; `PUT /{source}` links or re-narrows; retire unlinks |
| `/api/projects/{project}/repository-links`                                      | `GET`; `PUT /{repository}` links; retire unlinks           |
| `/api/projects/{project}/scopes`, `/routines`, `/findings`, `/garden-proposals` | the existing gardening verbs, moved under the project      |

The tenant-wide reads gain `?project={project}`: `GET /api/chunks`, `/api/chunk-counts`, `/api/queue`, `/api/backlog`,
the event feed, and the SSE stream's subscription, which filters chunk-scoped events by the chunk's project and passes
runner-scoped events for runners that serve it. `GET /api/runners` gains each runner's served declaration
([eligibility.md](./eligibility.md)). A project's own view carries `project_id`, `slug`, and `name` as separate fields,
and response models that describe a chunk gain `project_id`, `project_slug`, and `project_name`; every addition is
optional on the wire and additive.

The existing un-nested gardening routes (`/api/scopes`, `/api/routines`, …) remain as tenant-wide reads with the same
`?project=` filter, so the all-projects doorway reads every garden in one call; their write verbs move under the
project.

## CLI

`blizzard hub project create|list|show|edit|retire|enable` manage projects — `create <slug> [--name <display name>]`,
the name defaulting to the slug, and `edit <project> [--slug …] [--name …] [--description …]` — and
`blizzard hub project link|unlink <project> --source <source> | --repository <repository>` manage their links. Every
verb that creates a project-owned record takes a required `--project`, an id or a slug; every listing verb takes an
optional one; `item ingest` takes `--project` under ingest's inference rules. The CLI context that holds a hub login and
default tenant also holds an optional default project, the CLI's own lens, which `--project` overrides on any call.
Every call sends the context's tenant as `X-Blizzard-Tenant` alongside whatever project it names. A verb's help states
its effect in operator vocabulary (`bzh:help-states-effect`, `bzh:operator-vocabulary`).

## The shell lens

The lens is shell state, owned by `fleet/shell` and imported eagerly (`bzh:frontend-eager-shell-entry`). It is derived
from the URL, never held separately, so a refresh or a shared link reproduces it.

```text
/t/{tenant_id}/{view}/…                    lens on All
/t/{tenant_id}/p/{project_slug}/{view}/…   lens on one project
```

The board's `/t/{tenant_id}` page prefix is the multi-tenancy contract's; the `p/{project_slug}` segment is this
slice's, and holds the project's current slug, resolved by §Resolving a project — a former slug redirects to the current
one, and an id is accepted too. The lens control and the breadcrumb show the project's display name. The existing route
table (`web/projects/hub/src/app/app.routes.ts`) mounts once under each prefix, so every view and every deep link is
reachable with and without a lens, and a chunk's detail opened from a lens keeps it. Changing the lens re-navigates to
the same view under the other prefix. The lens control sits in the shell's app strip beside the tenant; the header's
counts are read with the lens's `?project=` filter.

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
