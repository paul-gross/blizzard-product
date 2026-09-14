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
part of*. One operator builds winter, blizzard, and a couple of other products from a single hub and a single runner,
and a project is how the platform tells those products apart. Findings and garden proposals belong to a project. A
runner can serve every project on its hub or just one. But much of the platform is project-independent by design: graphs
are a shared library that any project's work can travel, and the board shows the operator's whole fleet at once. A
project organises one operator's world. It does not partition the hub.

A tenant answers *whose world is this*. It is the only boundary that holds for everything: graphs, projects, work
sources, chunks, the queue, findings, events, runners, and the board. Two tenants on one hub share a running process,
and nothing they own is shared. Neither can see or reach the other's state, and neither needs to know the other exists.

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
made without it in mind: how a project is keyed, how work carries its grouping from ingest to landing, how a runner
declares what it serves. So the two are designed together and built back to back. Their registry rows sit side by side,
and their slices share a milestone.

## What to build — the hub slice

- **Tenant as the root of all state.** Every hub-owned record (graphs, projects, work sources, chunks, facts, findings,
  garden proposals, questions, escalations, usage, the event log, transcripts, runner registrations) belongs to exactly
  one tenant. Every read, write, queue operation, and SSE subscription is resolved within the caller's tenant. Nothing
  is reachable across the boundary by id-guessing.
- **Identity resolves to a tenant.** A session, an API token, and a runner's credentials each carry the tenant they act
  within. How users relate to tenants is an open question below.
- **Tenant administration.** Create, list, and delete tenants from the CLI and API, for the hub's administrator.
  Deleting a tenant removes everything it owns. Test suites depend on that teardown, so it must be complete and
  reasonably fast.
- **The single-tenant hub, carried over.** An existing installation becomes one default tenant, with no re-ingest and no
  configuration change. An operator who never creates a second tenant never notices the concept.

## What to build — the runner slice

- **A runner belongs to one tenant.** It registers, claims, and reports within that tenant. Within its tenant it serves
  every project or a declared subset, as `epic:projects` describes.
- **The runner's local store and workspaces are tenant-aware** wherever a runner could plausibly be pointed at a
  different tenant over its lifetime. Whether a single runner process may ever serve two tenants is an open question,
  and the default answer is no.

## Open questions

- **Row-level key or store per tenant.** A `tenant_id` threaded through every table is invasive but keeps one database.
  Alternatively, the hub could route each tenant to its own database or schema: a SQLite file per tenant, or a Postgres
  schema per tenant. That leaves most of the domain code untouched, moves the boundary into the storage layer, and makes
  deleting a tenant nearly free. This is the design decision the epic hinges on, and it is settled together with
  `epic:projects`' own keying question.
- **Users and tenants.** Does a user belong to one tenant or several? Is there a hub administrator above all tenants,
  and what can that administrator see?
- **Hub-wide startup configuration.** Auth mode, route-token mode, runner-auth mode, and produces mode are set when the
  hub starts. Should any of them become per-tenant, or does a tenant that needs different settings need a different hub?
- **What stays deliberately global.** Examples are the packaged system artifacts, migrations, and health and readiness
  endpoints. Each needs to be named, so that the boundary's exceptions are a list rather than a surprise.
