---
epic: projects
refinement: scaffolded
slices:
  - name: hub
    status: horizon
  - name: runner
    status: horizon
---

# Plan — `epic:projects`

An operator with three repositories runs blizzard three times. Three hubs, three runners, three boards to keep an eye
on, three sets of credentials to rotate — because blizzard's only notion of "everything I am working on" is the
installation itself. The platform has no concept of a project, so the operator supplies one by duplicating the stack.
This epic makes project a first-class grouping of both what to do and who does it: three projects on a laptop should
mean three workspaces and one runner host, not three of everything.

A project is a stored record inside a tenant, and it owns the things that are about one body of work: its repositories,
its scopes and routines, and the findings and garden proposals that gardening raises about it. It does not own the
places work is defined. A work source is a connection to a tracker — a Jira site holding a hundred projects' worth of
tickets, a GitHub repository's issues, the hub's own source — and it belongs to the tenant, so three projects that all
draw from one Jira share one source rather than configuring it three times. A project links to the sources it draws
from, and an item enters a project when it is ingested into one. Nor is a project a tag. `epic:tagging` lets a chunk say
what kind of work it is, and it can filter by project, but a tag cannot own a repository's forge or a garden, and a
project must.

The work lands in two slices, hub then runner. It stands on `epic:live-config`, which first moves work sources, the
delivery target, and their credentials into the hub's store, where a tenant can hold the sources and a project its
repositories, and it is built alongside `epic:multi-tenancy`, whose hub slice lands first so that every project record
is born inside a tenant. The runner slice builds on the host `epic:runner-host` introduces.

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
something a project owns — a repository, a scope, a routine — names the project outright.

## What to build — the hub slice

- **Project as a stored concept.** A tenant holds many projects, and each project owns its repositories, scopes,
  routines, findings, and garden proposals, and links to the work sources it draws from. Scope slugs and routine names
  are unique within a project, so two projects never collide on the name of a sweep.
- **Work sources shared, linked by project.** A work source belongs to the tenant, and any number of projects link to
  it. A link may narrow what that project usually draws from the source — a Jira project key, a GitHub repository — so
  browsing and bare references default sensibly, but it never makes the source the project's own. Closing and annotating
  an item still go through its source, whichever project carried the work.
- **A chunk belongs to exactly one project.** Its project is the one its items were ingested into, and grouping items
  ingested into two projects into one chunk is refused rather than resolved by a guess.
- **Ingest into a project.** An ingest names its source and the project it ingests into. The project may be left out
  only where it is unambiguous — the source is linked to a single project, or the caller's lens is on one — and a bare
  reference that more than one source would accept is refused with a request to name the source, where today it goes
  quietly to the first that matches. An item is held by one live chunk at a time, so it sits in one project at a time.
- **Repositories through their project.** A chunk's commits resolve against its project's repositories, each carrying
  the forge, owner, and base branch `epic:live-config` moved into the store, and landing behaves as it does today. This
  slice builds the place a repository's landing policy and deployment declaration will live; `epic:advanced-delivery`
  and `epic:advanced-deployment` fill it.
- **Project as a dimension of every fleet-wide surface.** The board and CLI gain project as a filter where they
  currently assume a single one, rather than gaining a second copy of themselves per project. The
  [projects-era admin mockup](./artifacts/projects-admin.html) shows where each level lands: the tenant in the header, a
  project lens across every tab, and inside each project the sources it links, the repositories it owns, and the runners
  that serve it. Gardening becomes per project entirely, as the
  [per-project gardening mockup](./artifacts/projects-gardening.html) shows: with the lens on a project the tab is that
  project's scopes, routines, runs and findings, and proposals, and with the lens on all of them it is a doorway into
  each project's garden.
- **The project lens belongs to the shell.** Which project a person is looking through is the app's state, not any one
  screen's: it sits in the shell beside the tenant, travels in the URL so a shared link opens through the same lens, and
  every view reads it. Each view decides what the lens means for it — the board and events narrow to the project's
  chunks, the header's counts follow, gardening opens that project's garden, admin narrows sources to the ones the
  project links — and a view the lens does not touch, such as the tenant-wide graph library or its members and secrets,
  says so rather than quietly ignoring it.
- **Runners serve what they declare.** A runner's registration records the projects it serves, and the hub offers it
  only chunks from those projects. A runner that declares nothing serves nothing, and the board says so rather than
  leaving its idleness to be puzzled out.
- **The single-project fleet, carried over.** An existing installation's state becomes one project — its repositories,
  chunks, scopes, routines, findings, and proposals, linked to every existing work source — and every existing runner
  registration is recorded as serving it. Nothing is re-ingested, and nothing that runs today is reconfigured.
- **The runners already deployed keep working.** A hub redeploys ahead of the runners that talk to it. A runner built
  before this slice declares no projects when it registers; the hub keeps the declaration it already holds for that
  runner rather than clearing it, so the fleet carries on until its runners are redeployed.

## What to build — the runner slice

- **A workspace per project, one host.** The runner host `epic:runner-host` introduces already holds a list of
  workspaces; here each one belongs to a project, and the host declares the projects it serves as the projects it holds
  workspaces for. Its runners claim across all of them, so the machine's capacity is shared rather than partitioned by
  installation.
- **Per-project isolation on the machine.** Checkouts, environments, and credentials are separated by project; a worker
  in one project's workspace has no path into another's.

## Where a project goes next

A project here is what to do, who does it, and the repositories its work lands in, landed the one way blizzard lands
work today. It is headed toward owning how that landing happens and what follows it: `epic:advanced-delivery` gives each
repository its own landing policy, and `epic:advanced-deployment` its own deployment declaration. Both build on the
repositories this epic gives a project, and neither can start until a project holds them.

## Open questions

- Whether a secret, inside its tenant, belongs to one project or may be shared by several — one forge token reaching
  every repository an operator owns, against the isolation of one project's credentials from another's.
- Whether the built-in `hub` source is one per tenant, linked to every project like any other source, with an accepted
  garden proposal ingested into the project it was about — the shape shared sources suggest — or one per project.
- What a link's narrowing may express beyond a single key or repository — a Jira query, a label — and whether it filters
  what may be ingested or only what is offered by default.
- Whether every runner under a host serves every project the host holds a workspace for, or a runner may narrow to some
  of them.
