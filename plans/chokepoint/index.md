---
epic: chokepoint
refinement: scaffolded
slices:
  - name: chokepoint
    status: horizon
    plan: ./chokepoint.md
  - name: epic-dwf
    status: horizon
    plan: ./epic-dwf.md
---

# Plan — `epic:chokepoint`

A few times a year a project has one large thing to do, and everything else has to stop so that it can be done well: a
rewrite from Python into Rust, the board moved from one framework to another, a generation of dependency upgrades, a
tech-debt sweep across the whole codebase. The change behind `epic:architectural-sweep` was one of these. It was safe
only because nothing else was in flight, and it was possible only because dozens of agents worked it in parallel — and
both of those were arranged by hand, from outside the fleet.

This epic lets the fleet arrange both itself. The [chokepoint slice](./chokepoint.md) clears the road: the operator
marks a chunk as a chokepoint, sees it flagged on the board well before it arrives, moves it up or down the queue as
plans change, and the fleet finishes everything ranked ahead of it, works it alone, and claims nothing ranked behind it
until it clears. The [epic-dwf slice](./epic-dwf.md) gives the work a lane built for its size: a development workflow
for epic-level change of twenty thousand lines or more, or thousands of small adjustments, planned whole, built by a
swarm of agents, and landed as one. The chokepoint comes first, because a swarm without a clear road is the very merge
it exists to avoid.
