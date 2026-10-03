# Plan — `epic:multi-tenancy`, hub slice

A test that wants a hub of its own today has to start one. It cannot borrow a running hub from its neighbour, because
nothing inside a hub is smaller than the whole installation: every graph, chunk, finding, and runner is everyone's. The
same wall stands in front of an operator who wants two fleets that never meet — a client's work and their own, a sandbox
and the real thing — and has to run two hubs to get them. This slice gives the hub a unit below the installation that
holds everything, and a promise that nothing crosses it.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

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

## What to build

- **Tenant as the root of all state.** Every hub-owned record (graphs, projects, work sources, repositories, secrets,
  chunks, facts, findings, garden proposals, scopes, routines, questions, escalations, usage, the event log,
  transcripts, runner registrations) belongs to exactly one tenant. Every read, write, queue operation, and SSE
  subscription is resolved within the caller's tenant, and the in-process event stream is partitioned by it. Nothing is
  reachable across the boundary by id-guessing.
- **One identity, many memberships.** A person signs in to the hub once, as one identity, and holds a membership in each
  tenant they may enter, with a role in each; the single role a user carries today moves onto that membership, so the
  same person can administer one tenant and only watch another. Each request names the tenant it acts within and is
  checked against the caller's memberships — the sign-in never remembers a "current" tenant — so two windows can hold
  two tenants without disturbing each other, and a link one person sends another lands in the right world. A person with
  one membership is never asked to choose. Machines never choose at all: a runner's credentials belong to exactly one
  tenant, and the credential settles it.
- **Secrets inside the boundary.** The secret store `epic:live-config` builds becomes tenant-scoped, so a work source or
  repository can name only its own tenant's secrets.
- **A tenant has an id and a name.** Everything that refers to a tenant holds its id, which never changes; people know
  it by a name the tenant chooses and may change at will — slug style by convention, not by rule. The carried-over
  tenant is named `default`. A link written with an old name keeps working until another tenant takes that name, and a
  link written with the id always works.
- **Tenant administration.** Create, rename, list, and delete tenants, and grant memberships, from the CLI and API, for
  the hub's administrator. Deleting a tenant removes everything it owns. Test suites depend on that teardown, so it must
  be complete and fast enough to run after every test.
- **The single-tenant hub, carried over.** An existing installation becomes one tenant holding all of its state, every
  existing user a member of it in the role they hold today, every runner registration inside it, with no re-ingest and
  no configuration change. An operator who never creates a second tenant never notices the concept.
- **The runners already deployed keep working.** A hub redeploys ahead of the runners that talk to it, so this slice
  must serve a runner built before it: a runner's credential resolves its tenant, its routes keep their paths, and the
  hub's responses only ever gain fields.

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
- **Postgres as a second guard.** Whether a hub running on Postgres should back the store seam with the database's own
  row-level security, catching a mistake the seam lets through.
