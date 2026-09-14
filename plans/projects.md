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
mean three workspaces and one runner, not three of everything.

The work lands in two slices, hub then runner. It is built alongside `epic:multi-tenancy`, which touches the same store,
API, and runner seams and scopes more of them, so the two designs are settled together rather than one after the other.

## What to build — the hub slice

- **Project as a stored concept.** One hub hosts many projects, each with its own work sources and queue, and work
  carries its project from ingest through to landing — so a fact, a chunk, and a delivery all know which project they
  belong to without anyone inferring it.
- **Project as a dimension of every fleet-wide surface.** The board and CLI gain project where they currently assume a
  single one, rather than gaining a second copy of themselves per project.
- **The single-project fleet, carried over.** Existing work acquires a project without an operator re-ingesting anything
  or reconfiguring what already runs.

## What to build — the runner slice

- **A workspace per project, one daemon.** The runner hosts a local workspace for each project it serves and claims
  across all of them, so the machine's capacity is shared rather than partitioned by installation.
- **Per-project isolation on the machine.** Checkouts, environments, and credentials are separated by project; a worker
  in one project's workspace has no path into another's.

## Where a project goes next

A project here is what to do and who does it. It is headed toward also owning where finished work goes: its
repositories, named beside its work sources, each bound to the forge it lives on and carrying how work lands in it and
how it deploys once landed. `epic:advanced-delivery` and `epic:advanced-deployment` build that on top of this epic, and
neither can start until a project can hold repositories. Nothing in this plan builds them, but its design should leave
room for them: a project that can only ever be a label on work would have nowhere to keep a repository's forge, landing
policy, or deployment declaration.

## Open questions

- Whether project is a first-class column or expressed as a scoped tag once `epic:tagging` lands — the two designs make
  very different things cheap, and this decides it.
- Whether a runner may serve a subset of a hub's projects, and if so whether that declaration belongs here or with the
  appetite levers in `epic:throttling`.
- How per-project credentials are held and scoped, given that the hub now holds secrets for several projects that should
  not be able to reach each other.
