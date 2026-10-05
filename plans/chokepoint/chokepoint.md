# Plan — `epic:chokepoint`, chokepoint slice

Every so often a project has one large thing to do, and everything else has to stop before it can be done well. A
rewrite of one component from Python into Rust, the board moved from one framework to another, a round of dependency
upgrades, a tech-debt sweep that touches every corner of the codebase — the work behind `epic:architectural-sweep` ran
this way, dozens of agents in parallel across one enormous change, and it was only safe because nothing else was in
flight. Work that moves that much ground cannot share the road. A chunk running beside it lands on code that shifted
underneath it, and the merge that reconciles the two is exactly where functionality quietly goes missing.

Today the operator who knows this has to enforce it by hand: watch the queue, hold back new work, wait for the fleet to
drain, start the big change, and remember to let everything else go once it lands. A chokepoint lets them say it once,
on the chunk itself, and lets the fleet keep the discipline for them. The name is borrowed from the narrow pass every
army must cross single file: the fleet narrows toward the chunk, passes through it alone, and spreads back out once it
clears.

## What to build

- **A chunk the operator can mark as a chokepoint.** Marking is a property of the chunk, set and cleared from the board
  and from the CLI like priority.
- **A chokepoint you can see coming.** The board marks a chokepoint chunk with an indicator of its own — an icon, a
  color, a flag — that no ordinary chunk carries, on its card wherever it appears and from the moment it is marked, not
  only once it starts. An operator scanning the queue should spot the chokepoint ahead the way a driver spots a lane
  closure sign: early enough to do something about it.
- **A chokepoint you can move.** The operator can move a chokepoint up or down the queue from the board, and the board
  shows what each move means — which chunks it would now wait on and which it would hold back — so that deciding to do
  the big thing sooner or later is a drag, not a negotiation with the fleet.
- **Nothing behind it is claimed.** From the moment a chokepoint exists, no runner claims a chunk ranked behind it in
  the queue's order, until the chokepoint is done.
- **Everything ahead of it finishes first.** The chokepoint itself is not claimable while any chunk ranked ahead of it
  is still open; it starts only once the fleet has drained the work in front of it.
- **The narrowing is visible.** The board shows that the fleet is narrowing toward a chokepoint, which chunks it is
  still waiting on, and why idle runners are taking nothing — an operator should never mistake a deliberate drain for a
  wedged fleet.

## Open questions

- Whether a chokepoint seals the queue ahead of it. A chunk minted at higher priority after the chokepoint exists could
  jump in front of it and push it back, or wait behind it; letting it in risks a chokepoint that never starts.
- What a chokepoint does about work ahead of it that is not moving — paused, parked at a gate, waiting on a human. It
  should not wait in silence forever, and it should name exactly who it is waiting on.
- Its scope once `epic:projects` lands. A rewrite of the hub need not stall a project that shares nothing with it, which
  suggests a chokepoint narrows its own project rather than the whole fleet.
- How it composes with dependencies: whether a chunk that names the chokepoint as a prerequisite is simply behind it,
  and whether a chokepoint may itself stand on a chunk ranked below it.
