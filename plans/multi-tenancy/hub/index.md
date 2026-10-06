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
  that tenant. Domain code never writes the filter by hand, so it cannot forget it. The seam keeps the tenant and
  nothing narrower: a project is a lens over a tenant's state where a tenant is a wall around it, so narrowing to a
  project is an ordinary filter a read takes, not a boundary the seam keeps.
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
  two tenants without disturbing each other. A person with one membership, which is nearly everyone, is never asked to
  choose and never sees a tenant in a link. Machines never choose at all: a runner's credentials belong to exactly one
  tenant, and the credential settles it.
- **Invite-only.** Signing in proves who someone is; it never lets them in. A person arrives only through an invitation
  to one tenant: one of that tenant's admins, or the hub administrator, issues it from the CLI for one email address and
  one role, and hands it over as a link. Whatever account the person signs in with must carry that address, so a
  forwarded link admits nobody else. Anyone who signs in uninvited, or who holds no membership, is told to reach out to
  their administrator — and the hub keeps no record of a stranger who merely tried.
- **Secrets inside the boundary.** The secret store `epic:live-config` builds becomes tenant-scoped, so a work source or
  repository can name only its own tenant's secrets.
- **One tenant per view.** The board shows one tenant at a time: every page, list, and live stream belongs to the tenant
  the person is working in. A person with several memberships switches between their tenants from the board, and can
  hold two of them open in two windows, but nothing on the board gathers several tenants into one view, and no board
  link carries a tenant.
- **A tenant has an id and a name.** Everything that refers to a tenant — every request and command — holds its id,
  which never changes and is never reused. The name is only for people to read: the hub administrator may change it at
  will, two tenants may share one, and since nothing ever looks a tenant up by name, a rename breaks nothing. The
  carried-over tenant is named `default`.
- **Two kinds of administrator.** The hub administrator — today's superuser — stands above every tenant: they alone
  create, rename, and delete tenants, and they may grant a membership in any of them. A tenant's own admins manage who
  belongs to their tenant and nothing beyond it, so a tenant can be handed to someone to run without handing them the
  hub. Standing above every tenant is not the same as seeing inside one: the hub administrator reads a tenant's state
  only through a membership of their own, like anyone else. The one membership the role brings with it is the first:
  whoever claims it on a hub whose only tenant has no admin yet becomes that tenant's admin, so the person who sets up a
  fresh hub is not locked out of the only world it holds.
- **Tenant administration.** Create, rename, list, and delete tenants, and grant memberships, from the CLI and API.
  Deleting a tenant removes everything it owns. Test suites depend on that teardown, so it must be complete and fast
  enough to run after every test.
- **A test needs no person only while auth is off.** On a hub whose auth is off, a test claims a tenant and acts inside
  it by naming the tenant on each request, with no user and no session. A hub that requires sign-in has no such
  shortcut: every caller there is a person with a membership or a machine with a credential, and a test mints a user and
  a session like anyone else. Whether a hub may keep running with auth off is not this epic's to settle; if that option
  goes, the shortcut goes with it.
- **The single-tenant hub, carried over.** An existing installation becomes one tenant holding all of its state, every
  existing user a member of it in the role they hold today, every runner registration inside it, with no re-ingest and
  no configuration change. An operator who never creates a second tenant never notices the concept.
- **Exports stay the hub's.** Trace export and fact egress keep the one destination whoever runs the hub chose, and stay
  theirs to operate. Every span and every exported row names its tenant, so each tenant's share can be told apart today
  and routed to the tenant itself later, by `epic:tenant-telemetry`.
- **Startup configuration stays hub-wide.** Auth mode is hub-wide because a person signs in before choosing a tenant.
  Route-token mode and produces mode are rollout brakes on the code's own security posture rather than anyone's
  preference, so they are hub-wide too. No startup setting varies by tenant.
- **The runners already deployed keep working.** A hub redeploys ahead of the runners that talk to it, so this slice
  must serve a runner built before it: a runner's credential resolves its tenant, its routes keep their paths, and the
  hub's responses only ever gain fields.

## Left for later

A hub on Postgres could back the store seam with the database's own row-level security, so that a mistake the seam lets
through is still caught. That second guard is worth having, but it follows this slice rather than belonging to it, and
it can only ever be a second guard. SQLite stays a supported store, so the seam remains the boundary on both, and the
isolation a tenant gets never depends on which database its hub happens to run on.
