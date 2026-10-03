---
epic: projects
refinement: scaffolded
slices:
  - name: hub
    status: horizon
    plan: ./hub/index.md
  - name: runner
    status: horizon
---

# Plan — `epic:projects`

An operator with three repositories runs blizzard three times. Three hubs, three runners, three boards to keep an eye
on, three sets of credentials to rotate — because blizzard's only notion of "everything I am working on" is the
installation itself. The platform has no concept of a project, so the operator supplies one by duplicating the stack.
This epic makes project a first-class grouping of both what to do and who does it: three projects on a laptop should
mean three workspaces and one runner host, not three of everything.

A project is a stored record inside a tenant, and it owns the things that are about one body of work: its scopes and
routines, and the findings and garden proposals that gardening raises about it. It does not own the places work is
defined, or the places it lands. A work source is a connection to a tracker — a Jira site holding a hundred projects'
worth of tickets, a GitHub repository's issues, the hub's own source — and it belongs to the tenant, so three projects
that all draw from one Jira share one source rather than configuring it three times. Repositories are the same: a
repository belongs to the tenant, and a library two projects both build on is one repository both link. A project links
to the sources it draws from and the repositories it may land in, and an item enters a project when it is ingested into
one. Nor is a project a tag. `epic:tagging` lets a chunk say what kind of work it is, and it can filter by project, but
a tag cannot hold links to sources and repositories or own a garden, and a project must.

The work lands in two slices, hub then runner. It stands on `epic:live-config`, which first moves work sources, the
delivery target, and their credentials into the hub's store, where a tenant holds them and a project links them, and it
is built alongside `epic:multi-tenancy`, whose hub slice lands first so that every project record is born inside a
tenant. The runner slice builds on the host `epic:runner-host` introduces.

## A lens, not a wall

The tenant is the wall around a world, and a project is a lens over one. Reading across projects is ordinary — the board
shows the whole fleet, graphs are a shared library, and a person narrows to one project only when they choose to — so
the hub's store scopes every read to its tenant and to a project only when a caller asks. The one place a project
behaves like a wall is on the machine, where a worker in one project's workspace must have no path into another's.

A project is also never assumed, and almost nothing needs to be told which one it belongs to. An item enters a project
at ingest; a chunk belongs to the project its items were ingested into, and every fact, event, transcript, and question
to its chunk's. Ingest is the one moment a project is chosen, and it names the project it ingests into — though when the
source is linked to a single project, or the person ingesting is already looking through one project's lens, that choice
is made for them. A read that names no project means the whole tenant, which is what every caller gets today. Creating
something a project owns — a scope, a routine — or linking it to a source or a repository names the project outright.

## What to build — the hub slice

The hub slice gives a tenant its projects: the stored record, the links to shared work sources, ingest that chooses a
project, delivery only into the repositories a project links, runners that serve the projects they declare, the project
lens across every surface, and the single-project fleet carried over. Its requirements and its technical contract are
the [hub slice plan](./hub/index.md).

## What to build — the runner slice

- **A workspace per project, one host.** The runner host `epic:runner-host` introduces already holds a list of
  workspaces; here each one belongs to a project, and the host declares the projects it serves as the projects it holds
  workspaces for. Its runners claim across all of them, so the machine's capacity is shared rather than partitioned by
  installation.
- **Per-project isolation on the machine.** Checkouts, environments, and credentials are separated by project; a worker
  in one project's workspace has no path into another's.

## Where a project goes next

A project here is what to do, who does it, and the repositories its work may land in, landed the one way blizzard lands
work today. How a repository is landed into and what follows describes the repository, not the project:
`epic:advanced-delivery` gives each repository its own landing policy, and `epic:advanced-deployment` its own deployment
declaration, both on the tenant's repository record. A project-level default for either, if one is ever wanted, would
sit on the project's link to the repository. Both build on the repository links this epic introduces, and neither can
start until a project can link a repository.

## Open questions

The hub slice's own open questions live in its [slice plan](./hub/index.md#open-questions).

- Whether every runner under a host serves every project the host holds a workspace for, or a runner may narrow to some
  of them.
