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
  and from the CLI like urgency.
- **A chokepoint you can see coming.** The board marks a chokepoint chunk with an indicator of its own — an icon, a
  color, a flag — that no ordinary chunk carries, on its card wherever it appears and from the moment it is marked, not
  only once it starts. An operator scanning the queue should spot the chokepoint ahead the way a driver spots a lane
  closure sign: early enough to do something about it.
- **A chokepoint you can move.** The operator can move a chokepoint up or down the queue from the board, and the board
  shows what each move means — which chunks it would now wait on and which it would hold back — so that deciding to do
  the big thing sooner or later is a drag, not a negotiation with the fleet.
- **Rules applied at the moment of claiming.** A chokepoint is enforced when a runner goes to pick up work, against the
  queue as it stands at that moment, and nowhere else. Nothing ranked behind a chokepoint is claimed until the
  chokepoint is done, and the chokepoint itself is not claimed while any chunk ranked ahead of it is still open.
- **A chokepoint belongs to its project.** Once `epic:projects` gives the hub more than one project, a chokepoint
  narrows only the project its chunk belongs to. Ahead and behind are counted within that project's queue, and runners
  go on claiming every other project's work as if nothing had happened. A rewrite of one codebase has no reason to stall
  a project that shares none of it. Until projects land, the hub holds a single project, and a chokepoint narrows the
  whole fleet.
- **The narrowing is visible.** The board shows that the fleet is narrowing toward a chokepoint, which chunks it is
  still waiting on, and why runners are taking nothing from that project — an operator should never mistake a deliberate
  drain for a wedged fleet.

## How it behaves

A chokepoint never freezes the queue in front of it. Anyone can put work ahead of it — minting a chunk ranked ahead of
it, or moving one past it — and the chokepoint simply waits for that work too. Whether the big change happens now or
later is the operator's to decide by where they place it; the fleet only keeps the rules each time it reaches for work.

Anything open ahead of a chokepoint holds it back, whatever state it is in. A chunk that is running, paused, parked at a
gate, or waiting on an answer is still ahead, and the chokepoint waits for it. The board names each one, so the operator
can see exactly what stands in the way, but the fleet never routes around them.

A chokepoint that fails keeps the road closed. If it escalates, nothing ranked behind it is claimed until a human
intervenes, because releasing the fleet onto a half-finished sweep is exactly the merge a chokepoint exists to prevent.
The human's decision is what reopens the road: resolving and resuming it, or stopping it, which ends it and lets the
work behind it go.

A project may hold more than one chokepoint. They pass one at a time in the queue's order, and ordinary work between
them flows as usual.

A chokepoint does not police dependencies. A chunk that names the chokepoint as a prerequisite was behind it anyway, and
a chokepoint that names a chunk ranked behind it as a prerequisite will wait for it forever. Blizzard does not guard
against that; the board shows what the chokepoint is waiting on, and seeing the mistake is the operator's job.
