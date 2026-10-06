# Delivery contract

## Repositories are the tenant's; projects link them

A repository is a tenant-wide configured record `epic:live-config` introduces
([records.md](../../../../delivered/live-config/spec/records.md)): an immutable `name`, the forge's API address, owner,
repository name, base branch, and the secret its token is read from. This slice adds no column to it. A project is given
a repository by a `project_repository_links` row ([model.md](./model.md) §Links to work sources and repositories), and
any number of projects may link one repository.

## A chunk lands only where its project links

Live-config owns resolution: each of a chunk's latest commit pointers resolves to one repository row by its exact
`forge` + `owner/repo` match, else by a bare name, else not at all
([records.md](../../../../delivered/live-config/spec/records.md) §Resolving a chunk's commits). This slice narrows the
rows resolution considers to the repositories the chunk's project links, as of the moment the hub step starts.

A pointer that resolves to no repository in that set refuses the hub step before its command runs. When the pointer
would have resolved to a tenant repository the project does not link, the outcome is `repository-not-linked`, its detail
naming the repository and the project; when it resolves to no tenant repository at all, the outcome is live-config's
`repository-unresolved`. Both are recorded in `event_log` and routed as a failed hub step, as live-config's outcomes
are. The sharper outcome earns its own name because it has a different remedy: an operator links the repository, or
decides the chunk's work strayed, where an unresolved pointer means the repository was never configured at all.

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
the repository, so chunks from every project that links it queue on one resource naturally. Until then, the hub's single
delivery slot serializes every landing as it does today.

## The `run:` environment

`bzh:hub-node-env-contract` gains one variable:

| Variable         | Carries                  |
| ---------------- | ------------------------ |
| `BZ_HUB_PROJECT` | The chunk's project name |

A platform script that delivers into the hub — `garden_deliver.py` through `BZ_HUB_GARDEN_DELIVERY_URL`,
`review_deliver.py` through `BZ_HUB_REVIEW_FINDINGS_URL` — needs nothing new: the callback is keyed by the chunk, and
the hub records what it delivers in the chunk's project.

## Landing

Fast-forward onto each repository's base branch remains the only landing, performed by the packaged scripts
(`hub/graphs/scripts/land_common.py` and its callers) as today. `delivery_repo_landed.repo` records the resolved
repository's `owner/repo` and gains no project column — its chunk carries the project. The chunk view's artifact branch
links resolve through the same repository row, not through the chunk's first work ref's source
(`api/chunk_views.py::_branch_url_source`).

## Where later epics attach

A landing policy and a deployment declaration describe the repository, not any project that uses it, so
`epic:advanced-delivery` and `epic:advanced-deployment` attach them to the tenant's repository record, as further
configured fields under the same verbs and revisions. A project-level default over either, if one is ever wanted, would
sit on the project's repository link. Neither changes the boundary: a chunk still lands only in repositories its project
links.
