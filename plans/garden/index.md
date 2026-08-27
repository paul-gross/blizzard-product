# Plan — `epic:garden`

Standing tending of what a project accumulates. The name is the thesis: a garden is something growing, and growth is
exactly why maintenance is its steady state rather than an intervention — nothing tended holds still between visits. A
young project with little code and little context needs no gardening at all, and has no routines to its name; that is
the correct state, not a gap to fill. Gardening begins when there is growth worth pruning, and from then on it does not
end.

This is a hard problem that will take real exploration, and this epic is shaped for exploring it: it builds the
machinery for named, repeatable evaluation passes an operator kicks off by hand, watches, and refines — with fully
autonomous recurrence as the end state the machinery is built to grow into, not the thing built first.

What blizzard builds is that machinery and nothing more. It never learns what a weed is. The strategy — what to look
for, and what to judge it against — is the deployment's own, and blizzard is a deployment like any other. That split is
what this plan is organized around: one file for the platform, and the rest for what this particular project brings to
it.

| Where                                      | What it holds                                                                                                                                   |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| [machinery.md](./machinery.md)             | Everything blizzard builds — routines, runs, findings, proposals, delivery, and the management surface. Carries no fact about any target.       |
| [user-interface.md](./user-interface.md)   | The gardening tab in the hub webapp — the surface the human half of the loop happens on, stated against three mockups                           |
| [blizzard-garden.md](./blizzard-garden.md) | What this deployment brings: the beds it will tend, the routines it declares, its own proposal vocabulary, and what its corpus owes before then |
| [blizzard-graph.md](./blizzard-graph.md)   | The graph blizzard's own routines run, prebaked at [artifacts/garden-routine/](./artifacts/garden-routine/) so it can be argued with            |

## What already holds

`epic:self-sourced-work` built the lane a pass's opinions eventually travel — structured proposals on node completions,
materialized by the hub at delivery, filterable at a human gate before they exist — so nothing this fleet thinks becomes
work by fiat. What rides that lane here is narrower than the whole of a pass's output: a proposal a person has accepted,
and nothing else. A finding never becomes a work item, and a proposal becomes one only in the act of being accepted. The
backlog those items land in is rankable, the promote gate holds, and closure facts record how every one of them ends.

Work already runs as graphs with human gates where a graph wants them, and the hub has no work scheduler of any kind —
the only recurring machinery is the reconciler sweep loop. That absence is now deliberate: this epic leaves it mostly
intact.

What `epic:self-sourced-work` did not cover is everything durable this epic adds. Findings, their sets, and proposals
are new hub entities with their own identity, endpoints, and management surface. That epic built the lane for turning
opinions into work; this one builds the evidence they are formed from.

## What this epic is not

Not a scheduler yet: manual kickoff is the exploration mode, and the cadence ships when a pass has earned it. Not edit
authority: a pass never prunes anything directly — everything it touches changes only through promoted chunks riding
normal delivery. Not a definition of what a weed is: blizzard supplies the machinery and the deployment supplies the
strategy, so nothing built here decides what any project ought to prune. Not analytics or mutation testing: a strategy
consumes their evidence where it exists, and works from judgment and git where it does not.
