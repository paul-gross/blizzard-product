---
epic: queue
refinement: scaffolded
slices:
  - name: dependencies
    status: delivered
    plan: ./dependencies/index.md
  - name: priority
    status: horizon
    plan: ./priority/index.md
---

# Plan — `epic:queue`

A queue that hands out work in the order it arrived is honest about one thing and blind to two others. It cannot hear
that a second chunk stands on a first — the API before the screen that calls it, the migration before the code that
reads the new column — and offers the second to a runner anyway, so an agent lays ground another chunk already covers
while the first sits parked at a gate. And it cannot hear that chunks are not equally important: an operator who knows
one matters more than the four ahead of it has no way to say so, and a chunk nobody champions sinks a little further
every time newer work arrives.

The [dependencies slice](./dependencies/index.md) closed the first gap: a chunk can name what it stands on, the hub
refuses a claim while the prerequisite is still open, and a blocked chunk keeps its place in whichever list it already
lived in rather than becoming something else. The [priority slice](./priority/index.md) answers the second — priority
the operator can state, and aging that lets patience count for something — and is deliberately low in the build order,
since ordering only earns its keep once the queue runs deep enough for something in it to starve.
