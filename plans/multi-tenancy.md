---
epic: multi-tenancy
refinement: refined
slices:
  - name: hub
    status: horizon
  - name: runner
    status: horizon
---

# Plan — `epic:multi-tenancy`

A project and a tenant are easy to confuse, and they answer different questions. A project answers *what is this work
part of*. One operator builds winter, blizzard, and a couple of other products from a single hub and a single runner
host, and a project is how the platform tells those products apart: each draws on the tenant's work sources and owns its
repositories, its scopes and routines, and the findings and garden proposals gardening raises about it. But much of the
platform is project-independent by design: graphs are a shared library that any project's work can travel, and the board
shows the operator's whole fleet at once. A project organises one operator's world. It does not partition the hub.

A tenant answers *whose world is this*. It is the only boundary that holds for everything: graphs, projects, work
sources, chunks, the queue, findings, events, runners, secrets, and the board. Two tenants on one hub share a running
process, and nothing they own is shared. Neither can see or reach the other's state, and neither needs to know the other
exists.

The first person who needs that boundary is the test suite. A service test cannot share a running hub with its
neighbours today, because the hub has no unit below the installation that a test could claim as its own.
`epic:test-shared-service` runs many tests against one hub, each inside a tenant of its own, and it cannot start until
this epic lands. The same boundary also serves anyone who wants to run separate fleets without standing up separate
hubs. This is not a SaaS epic: there is no sign-up, billing, or self-service tenant creation. Tenants are created by
whoever administers the hub.

## Why it travels with `epic:projects`

Both epics add a grouping key to a store in which every table is global today. The hub has about eighty tables, and none
carries a grouping above the installation. Both then make the API, the event stream, and the runner's claiming respect
that key. Projects touches some of that surface, and tenancy touches all of it. Designed one after the other, the second
epic would reopen every table, route, and claim path the first had just finished, and it would find decisions already
made without it in mind. So the two are designed together and built back to back, their registry rows side by side and
their slices sharing a milestone.

The order is decided by which one would otherwise reopen the other. Projects creates tables — projects themselves, and
the records that hang beneath them — and tenancy keys every table there is, so tenancy's hub slice lands first and every
project record is born inside a tenant. Both stand on `epic:live-config`, which first moves work sources, repositories,
and their credentials out of the hub's file and environment and into its store, where a tenant can own them. The runner
slices of both epics follow their hub slices.

## How the boundary is kept

Every tenant-owned table carries the tenant as a column, in one database. A store per tenant — a database file or schema
each — was the alternative, and it would have left the domain code untouched and made deleting a tenant nearly free. It
moves the cost elsewhere: every background loop the hub runs would iterate tenants, every migration would run once per
tenant and could leave them at different versions, and any question asked across tenants would fan out over stores. A
row-level key keeps one database, one migration, and one place to ask a question of the whole hub, and it pays with a
key on every table and a filter on nearly every query.

That price carries a promise the database cannot keep on the hub's behalf. Postgres can enforce a row-level boundary
itself, but SQLite — the default store, and the one the tests run on — cannot, so the hub has to make an unscoped read
hard to write rather than merely wrong:

- **Scoping lives in the store seam.** A repository is opened for a tenant, and every read and write it issues carries
  that tenant. Domain code never writes the filter by hand, so it cannot forget it. The same seam narrows to a project
  when a caller asks, because a project is a lens over a tenant's state where a tenant is a wall around it.
- **Every table is accounted for.** Each table either carries the tenant or appears on the named list of what stays
  global, and a check fails the build when a table is neither.
- **Names are unique within a tenant.** Graph names and every other uniqueness rule a tenant can see becomes unique per
  tenant, so two tenants never collide on a name neither knows the other holds.

## What to build — the hub slice

- **Tenant as the root of all state.** Every hub-owned record (graphs, projects, work sources, repositories, secrets,
  chunks, facts, findings, garden proposals, scopes, routines, questions, escalations, usage, the event log,
  transcripts, runner registrations) belongs to exactly one tenant. Every read, write, queue operation, and SSE
  subscription is resolved within the caller's tenant, and the in-process event stream is partitioned by it. Nothing is
  reachable across the boundary by id-guessing.
- **One identity, many memberships.** A person signs in to the hub once, as one identity, and holds a membership in each
  tenant they may enter, with a role in each; the single role a user carries today moves onto that membership, so the
  same person can administer one tenant and only watch another. Each request names the tenant it acts within and is
  checked against the caller's memberships, so two windows can hold two tenants and a link one person sends another
  lands in the right world. A person with one membership is never asked to choose. Machines never choose at all: a
  runner's credentials and an API token each belong to exactly one tenant, and the credential settles it.
- **Secrets inside the boundary.** The secret store `epic:live-config` builds becomes tenant-scoped, so a work source or
  repository can name only its own tenant's secrets.
- **Tenant administration.** Create, list, and delete tenants, and grant memberships, from the CLI and API, for the
  hub's administrator. Deleting a tenant removes everything it owns. Test suites depend on that teardown, so it must be
  complete and fast enough to run after every test.
- **The single-tenant hub, carried over.** An existing installation becomes one tenant holding all of its state, every
  existing user a member of it in the role they hold today, every runner registration inside it, with no re-ingest and
  no configuration change. An operator who never creates a second tenant never notices the concept.
- **The runners already deployed keep working.** A hub redeploys ahead of the runners that talk to it, so this slice
  must serve a runner built before it: a runner's credential resolves its tenant, its routes keep their paths, and the
  hub's responses only ever gain fields.

## What to build — the runner slice

- **A runner belongs to one tenant.** It registers, claims, and reports within that tenant, and within it serves the
  projects its registration declares, as `epic:projects` describes.
- **The runner's local store and workspaces are tenant-aware** wherever a runner could plausibly be pointed at a
  different tenant over its lifetime. Whether a single runner process may ever serve two tenants is an open question,
  and the default answer is no.

## Open questions

- **What the hub administrator sees.** The administrator stands above every tenant: they create and delete tenants and
  grant memberships. Whether that role also reads inside a tenant, or must hold a membership there like anyone else, is
  undecided; the leaning is that administering a tenant does not mean reading it.
- **Whether a test needs a person.** A test could claim a tenant and act inside it with no sign-in at all, naming the
  tenant on each request, on a hub whose auth is off; otherwise every test first mints a user and a session. The leaning
  is to allow the shortcut only while auth is off.
- **Hub-wide startup configuration.** Auth mode stays hub-wide, because a person signs in before choosing a tenant.
  Route-token mode, runner-auth mode, and produces mode are rollout brakes on the code's own security posture rather
  than anyone's preference, and are expected to stay hub-wide too; whether any setting genuinely needs to vary by tenant
  is still to be shown.
- **What stays deliberately global.** The identity store — users, their linked sign-in identities, sessions, and
  memberships — the packaged system artifacts, migrations, and health and readiness endpoints are the known members. The
  rest needs naming, so that the boundary's exceptions are a list rather than a surprise.
- **Postgres as a second guard.** Whether a hub running on Postgres should back the store seam with the database's own
  row-level security, catching a mistake the seam lets through.
