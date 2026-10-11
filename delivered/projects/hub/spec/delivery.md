# Delivery contract

## Repositories are the tenant's; projects link them

A repository is a tenant-wide configured record `epic:live-config` introduces
([records.md](../../../../delivered/live-config/spec/records.md)): an immutable `name`, the forge's API address, owner,
repository name, base branch, and the secret its token is read from. This slice adds no column to it. A project is given
a repository by a `project_repository_links` row ([model.md](./model.md) §Links to work sources and repositories), and
any number of projects may link one repository.

## A chunk lands only where its project links

Live-config owns resolution (`hub/domain/config/repositories.py::resolve_chunk_repositories`). Each of a chunk's latest
commit pointers is matched by its repository name against each record's `repo`, narrowed by the pointer's owner and
forge host when it carries them; a record never matches by its own `name`. Only records standing for the chunk are
candidates — enabled, or retired after the chunk was minted — and a pointer that matches none, or more than one, is
unresolved. This slice keeps every one of those rules and adds one: only the repositories the chunk's project links, as
of the moment the hub step starts, are candidates. Narrowing to the project's links can also turn a pointer that is
ambiguous across the tenant into one that resolves, when only one of the matching records is linked.

A pointer that resolves to no repository in that set refuses the hub step before its command runs. When the same pointer
would have resolved against every standing record of the tenant, to one the project does not link, the outcome is
`repository-not-linked`, its detail naming the repository and the project; when it resolves to no tenant repository at
all, the outcome is live-config's `repository-unresolved`. Both are recorded in `event_log` and routed as a failed hub
step, as live-config's outcomes are; `repository-not-linked` joins the closed `EventLogKind` and its severity map
(`foundation/event_log.py`) beside them. The sharper outcome earns its own name because it has a different remedy: an
operator links the repository, or decides the chunk's work strayed, where an unresolved pointer means the repository was
never configured at all.

This is the delivery boundary a project draws. A chunk's agents may commit wherever their workspace lets them, but the
hub lands nothing into a repository the chunk's project was not given, and no hub-wide owner or forge remains to fall
back to. Unlinking a repository takes effect at the next hub step that resolves against it; a step already running
finishes.

The single-valued `run:` variables live-config fills from the resolved rows — `BZ_FORGE_URL`, `BZ_FORGE_TOKEN`,
`BZ_FORGE_OWNER`, `BZ_HUB_BASE_BRANCH` — and its `repositories-disagree` refusal are unchanged. Landing across forges or
credentials stays `epic:advanced-delivery`'s.

## Two projects, one repository

Projects that link the same repository land on the same branch of the same repository row. Nothing per-project separates
their landings: `epic:advanced-delivery` models landing as a step holding the resource `repo:<name>/<branch>`, keyed by
the repository, so chunks from every project that links it queue on one resource naturally. Until then, `hub_exec_slot`
serializes them: it holds one hub-executed node at a time — every landing among them, not only landings — and under
`epic:multi-tenancy` there is one such slot per tenant, so every project of a tenant shares it.

## The `run:` environment

`bzh:hub-node-env-contract` gains two variables:

| Variable            | Carries                                                                    |
| ------------------- | -------------------------------------------------------------------------- |
| `BZ_HUB_PROJECT_ID` | The chunk's project id, which never changes                                |
| `BZ_HUB_PROJECT`    | The project's slug as the step starts, which a rename may change mid-chunk |

A script that keys anything it keeps — a file, a record, a cache — on the project uses `BZ_HUB_PROJECT_ID`, so a slug
change between two of a chunk's steps never splits it; the slug is for what a person reads.

A platform script that delivers into the hub — `garden_deliver.py` through `BZ_HUB_GARDEN_DELIVERY_URL`,
`review_deliver.py` through `BZ_HUB_REVIEW_FINDINGS_URL` — needs nothing new: the callback is keyed by the chunk, and
the hub records what it delivers in the chunk's project.

## Landing

Fast-forward onto each repository's base branch remains the only landing, performed by the packaged scripts
(`hub/graphs/scripts/land_common.py` and its callers) as today. `delivery_repo_landed.repo` keeps recording the resolved
repository's `owner/repo`, and the row gains `repository_name`, the name of the repository record the pointer resolved
to — within the row's tenant, exactly one record — written at landing. `repo` alone names no forge, so it cannot say
which record was landed into; `repository_name` can, and it is what the landing invariant checks against the project's
link facts ([model.md](./model.md) §Invariants). Rows written before this slice keep it null and are exempt. The row
gains no project column — its chunk carries the project. The chunk view's artifact branch links resolve through the same
repository row, not through the chunk's first work ref's source (`api/chunk_views.py::_branch_url_source`).

## Where later epics attach

A landing policy and a deployment declaration describe the repository, not any project that uses it, so
`epic:advanced-delivery` and `epic:advanced-deployment` attach them to the tenant's repository record, as further
configured fields under the same verbs and revisions. A project-level default over either, if one is ever wanted, would
sit on the project's repository link. Neither changes the boundary: a chunk still lands only in repositories its project
links.
