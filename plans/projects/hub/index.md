# Plan — `epic:projects`, hub slice

An operator who builds blizzard, winter, and celestial frontier from one hub today has one undifferentiated queue. Every
chunk lands through the same forge owner and base branch, every runner will claim anything, every scope and routine
shares one namespace, and the board cannot be asked to show only winter. The hub has no idea which body of work a chunk
belongs to, so the operator keeps that map in their head — or stands up a second hub to keep it for them.

This slice gives a tenant its projects.
[`persona:application-architect`](../../../charter/personas/application-architect.md) creates a project, links it to the
trackers it draws from and the repositories its work may land in, and from then on the hub carries the project for them:
from the moment an item is ingested, through the runner that claims it and the repositories it lands in, to the garden
that tends it.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

- **Project as a stored concept.** A tenant holds many projects. A project is known to the hub by an id that never
  changes; to the people who type it, by a short slug unique within its tenant (`winter`, `celestial-frontier`); and to
  the people who read it, by a display name that is unique nowhere. Slug and name may both change at will. A link or a
  runner's configuration written against an old slug keeps working until another project of the tenant claims that slug,
  which any project may do, the one that left it included. Each project owns its scopes, routines, findings, and garden
  proposals, and links to the work sources it draws from and the repositories it may land in. Scope slugs and routine
  names are unique within a project, so two projects never collide on the name of a sweep.
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
- **Repositories shared, linked by project.** A repository belongs to the tenant, carrying the forge, owner, and base
  branch `epic:live-config` moved into the store, and any number of projects link to it. A library two projects both
  build on is one repository both link, so their work lands on the same branch and waits its turn there like any other.
- **Work lands only where its project was given.** A chunk may land only in repositories its project links. Its agents
  may commit wherever their workspace lets them, but the hub will not land a commit into a repository the project was
  not given: the step is refused before anything is pushed, naming the repository and the project. Landing itself
  behaves as it does today. How a repository is landed into and deployed — `epic:advanced-delivery`'s landing policy,
  `epic:advanced-deployment`'s deployment declaration — belongs to the repository, because it describes the repository
  rather than any project that uses it.
- **Runners serve what they declare.** A runner's registration records the projects it serves, and the hub offers it
  only chunks from those projects. A runner that declares nothing serves nothing, and the board says so rather than
  leaving its idleness to be puzzled out.
- **The project lens belongs to the shell.** Which project a person is looking through is the app's state, not any one
  screen's: it sits in the shell beside the tenant, travels in the URL so a shared link opens through the same lens, and
  every view reads it. Each view decides what the lens means for it — the board and events narrow to the project's
  chunks, the header's counts follow, gardening opens that project's garden, admin narrows sources to the ones the
  project links — and a view the lens does not touch, such as the tenant-wide graph library or its members and secrets,
  says so rather than quietly ignoring it. The [projects-era admin mockup](../artifacts/projects-admin.html) and the
  [per-project gardening mockup](../artifacts/projects-gardening.html) show where each level lands.
- **Gardening per project, entirely.** With the lens on a project, the gardening tab is that project's scopes, routines,
  runs and findings, and proposals; with the lens on all of them, it is a doorway into each project's garden. A routine
  runs a graph from the tenant's shared library, and a run's work is ingested into the routine's own project.
- **The single-project fleet, carried over.** An existing installation's state becomes one project, with the slug
  `default` for the operator to change — its chunks, scopes, routines, findings, and proposals, linked to every existing
  work source and every existing repository — and every existing runner registration is recorded as serving it. Nothing
  is re-ingested, and nothing that runs today is reconfigured.
- **The runners already deployed keep working.** A hub redeploys ahead of the runners that talk to it. A runner built
  before this slice declares no projects when it registers; the hub keeps the declaration it already holds for that
  runner rather than clearing it, so the fleet carries on until its runners are redeployed.

## Open questions

- Whether a secret, inside its tenant, may be narrowed to one project, isolating one project's credentials from
  another's.
- Whether the built-in `hub` source stays one per tenant, linked to every project, or becomes one per project.
- What a link's narrowing may express beyond a single key or repository — a Jira query, a label — and whether it filters
  what may be ingested or only what is offered by default.
