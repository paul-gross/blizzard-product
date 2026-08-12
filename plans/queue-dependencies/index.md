# Plan — `epic:queue` (dependencies slice)

Some work arrives in pairs: a second chunk that stands on the first — the API before the screen that calls it, the migration before the code that reads the new column. Today the queue has no way to hear that coupling. It hands out chunks in a flat order, and when the first of a pair stalls at a human gate, the second goes right ahead — to an agent that, finding the ground it expected missing, builds it again. The operator returns to two implementations of one idea, one of them parked at review, and a reconciliation neither agent could have done alone.

Grouping is the existing answer where the coupling is known before anything starts: fold the pair into one chunk, and one agent does both in order. What grouping cannot reach is work already split or already moving — a claimed chunk can no longer be folded, and a dependency discovered mid-flight has nowhere to go. This slice gives the queue its first relation: a declared edge between chunks that claiming respects.

## What to build

- **The edge.** A chunk names the chunks it depends on — declared by the operator from the board, any time before the dependent runs, and refused when it would close a cycle. The store carries it as a plain relation between chunk ids; nothing is inferred, and the work source is never asked to express it.
- **Blocked, visibly.** A chunk with an unmet dependency derives a blocked status of its own, shown on the board with the chunk it waits on — never a ready chunk that is mysteriously unclaimable. Blocked chunks stay out of the ready queue entirely, so a runner's fill pass never stalls behind one; the moment the last prerequisite lands, the chunk comes ready with no one's help.
- **Enforcement where claims are decided.** The hub re-checks the edge in the same place it already refuses a terminal chunk: under the claim lock. A race between a peek and a claim cannot slip a blocked chunk through — the guarantee is structural, in the spirit of the lease protocol.
- **Done means done.** An edge is satisfied by exactly one outcome: the prerequisite reaching its terminal done. A prerequisite that is stopped instead leaves its dependents blocked and puts the question to the operator — is the dependent now moot, in need of reshaping, or free to run? The edge is released deliberately, never assumed away.
- **The operator's override.** Because stopped never unblocks anything, the operator holds two levers and both are first-class: release an edge outright, and declare a chunk done by hand — for work that landed outside the fleet, or that the operator judges complete however it ended. A hand-declared done satisfies every edge standing on the chunk, exactly as a fleet-earned one would. Blocked is a held state, never a dead end.
- **Reach-ahead by default, waiting by choice.** A blocked chunk never enters the ready order, so the fleet flows around it without any skip logic — the runner simply takes the next chunk that is truly ready. A runner whose operator would rather it hold the line can be configured strict: work in queue order or not at all, idling while the head is blocked. Strictness is that runner's configuration, never the fleet's default.
- **Verification, not selection.** The claim protocol already puts the choice in the runner's hands — the runner names the chunk it wants, and the hub's role is to refuse, never to pick. The dependency check joins the refusals: whichever chunk a runner reaches for, head of queue or not, an unmet edge is a denied claim.

## An optional addon — the story map

Edges are the first data the board has ever held about how chunks relate, and a picture falls out of them: a chunk drawn in its lineage — what it stood on, greyed out where done; what stands on it, waiting its turn. For the operator planning a feature that arrives as five coupled chunks, that view answers at a glance what today takes reading five detail pages: how far the story has gotten, and what unlocks next. It is read-only by design — the same philosophy the board already lives by, a rich visual reading of the fleet while the changes themselves stay agentic — and it is deliberately optional: the edge, the blocked status, and the enforcement stand complete without it, and the map ships only if the slice has room.

## Artifacts

| Mockup                                                       | What it explores                                                                                                                                                                                                                                                 |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [story-map-desktop.html](./artifacts/story-map-desktop.html) | The map at a desk: stories in a sidebar, the selected story's chunks drawn left to right with their edges — satisfied, open, and open-from-a-chunk-that-needs-you — and a detail rail carrying the operator's two levers, release-edge and declare-done-by-hand. |
| [story-map-mobile.html](./artifacts/story-map-mobile.html)   | The same map at thumb size: a story-picker chip row, the flow drawn top to bottom, and a bottom sheet holding the detail and the same two levers.                                                                                                                |

## Open questions

- Whether the GitHub binding should read a "depends on #N" line in an issue body as a *suggested* edge for the operator to confirm at ingest. The work-source seam stays thin either way: a suggestion, never an authority.
- How the edge composes with grouping and requeue — folding a prerequisite into a survivor chunk should carry its incoming edges along, and a requeued prerequisite should leave its dependents' edges standing.
- Whether declaring an edge against an already-done chunk is accepted as trivially satisfied, or refused as noise.
- How far the story map reaches, if it ships — one chunk's neighborhood, or the whole queue drawn as a graph.
