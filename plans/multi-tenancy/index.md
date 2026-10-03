---
epic: multi-tenancy
refinement: refined
slices:
  - name: hub
    status: horizon
    plan: ./hub/index.md
  - name: runner
    status: horizon
---

# Plan — `epic:multi-tenancy`

A project and a tenant are easy to confuse, and they answer different questions. A project answers *what is this work
part of*. One operator builds winter, blizzard, and a couple of other products from a single hub and a single runner
host, and a project is how the platform tells those products apart: each links the tenant's work sources and
repositories and owns its scopes and routines, and the findings and garden proposals gardening raises about it. But much
of the platform is project-independent by design: graphs are a shared library that any project's work can travel, and
the board shows the operator's whole fleet at once. A project organises one operator's world. It does not partition the
hub.

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

## The hub slice

The hub slice — the tenant key on every table, the store seam that keeps it, one identity with a membership per tenant,
tenant administration, and the installation carried over as a single tenant — is planned in
[its slice plan](./hub/index.md), with its technical contracts beneath it.

## What to build — the runner slice

- **A runner belongs to one tenant.** It registers, claims, and reports within that tenant, and within it serves the
  projects its registration declares, as `epic:projects` describes.
- **The runner's local store and workspaces are tenant-aware** wherever a runner could plausibly be pointed at a
  different tenant over its lifetime. Whether a single runner process may ever serve two tenants is an open question,
  and the default answer is no.
