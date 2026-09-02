# Plan — `epic:queue` (dependencies slice)

Some work arrives in pairs: a second chunk that stands on the first — the API before the screen that calls it, the
migration before the code that reads the new column. Today the queue has no way to hear that coupling. It hands out
chunks in a flat order, and when the first of a pair stalls at a human gate, the second goes right ahead — to an agent
that, finding the ground it expected missing, builds it again. The operator returns to two implementations of one idea,
one of them parked at review, and a reconciliation neither agent could have done alone.

Grouping is the existing answer where the coupling is known before anything starts: fold the pair into one chunk, and
one agent does both in order. What grouping cannot reach is work already split or already moving — a claimed chunk can
no longer be folded, and a dependency discovered mid-flight has nowhere to go. This slice gives the queue its first
relation: a declared edge between chunks that claiming respects.

## What already holds

The hub already refuses a claim it must not grant, and it refuses it in the only place a refusal is safe: under the
claim lock, where a terminal chunk's denial is decided rather than merely predicted. That lock is shared with the edit
path, so a concurrent edit and a claim resolve to exactly one winner. A new refusal belongs there and inherits the
guarantee rather than reinventing it.

The operator can already declare a chunk done by hand — from the board, the CLI, or the API — and can do it from any
non-terminal status, a stopped chunk included. It records the same completion fact the fleet records when it finishes
the work honestly. The lever exists; what this slice adds is that the fact now satisfies something.

A chunk lives in one of two ranked lists, the ready queue or the backlog, and which one it occupies is derived from its
status. Grouping folds only chunks no runner has claimed, and deletion and the pre-claim property edits admit that same
unclaimed window. Each of those windows is a door a new idea about chunks can close by accident.

Requeue is runner-local: a requeued chunk's route is never released and it never re-enters the hub's queue, so nothing a
requeue does is visible to a dependent.

## What to build

- **The edge.** A chunk names the chunks it depends on — declared while the dependent is still unclaimed, the window its
  build properties are already editable in, and refused when it would close a cycle. The check and the commit happen
  together under the claim lock, so two declarations of opposite edges in the same moment resolve to one winner and a
  cycle is never assembled out of two innocent halves. An edge named against a chunk that is already done is accepted
  and is simply already satisfied.
- **Whoever knows, declares.** The edge is an API capability before it is a board control, and both declaring one and
  releasing one are reached the same way. The operator at the board is one caller; a model reading the API and
  orchestrating a feature across five chunks is another, and it is the better-placed one — it holds the reasoning about
  what stands on what. Blizzard's part is narrower and stays narrow: hold the relation, refuse the cycle, deny the
  claim. It infers no dependency, reads none out of a work source at ingest, and offers no place to configure how one
  might be guessed.
- **What the store carries.** A row between two chunk ids, released by a set-once marking rather than deleted, so that
  an operator's release is an act the fleet can account for rather than a row that quietly stops existing. Satisfaction
  is not stored: an edge is met when its prerequisite reads done, which is why an edge onto finished work needs no
  special case.
- **Blocked is a marking, not a station.** A chunk with an unmet dependency does not become something else. It keeps the
  status it derives and the rank it holds, stays in the list it already lived in, and remains exactly as groupable,
  deletable, and editable as it was a moment earlier. What it gains is a marking — blocked, carried beside the status
  and naming the chunk immediately above it, so a chunk that cannot be claimed always says why. It names that one
  prerequisite and stops there: where the chunk it waits on is itself blocked, walking the chain to whatever sits at its
  root costs more than the answer is worth, and an operator who wants the root follows the naming one hop at a time. No
  status ladder gains a rung, no admit window silently narrows, and the listings pay nothing new: the edges load in the
  same bulk pass the statuses already load in.
- **Enforcement where claims are decided.** The hub re-checks the edge under the claim lock, beside the refusal it
  already makes for a terminal chunk, and answers with a denial of its own that names the prerequisite still standing. A
  race between a peek and a claim cannot slip a blocked chunk through — the guarantee is structural, in the spirit of
  the lease protocol.
- **Done means done.** An edge is satisfied by exactly one outcome: the prerequisite reaching its terminal done. A
  prerequisite that is stopped instead leaves its dependents blocked and puts the question to the operator — is the
  dependent now moot, in need of reshaping, or free to run? The edge is released deliberately, never assumed away.
- **The operator's release.** Because stopped never unblocks anything, the operator holds the lever that does: release
  an edge outright. Its counterpart already exists — a chunk declared done by hand, for work that landed outside the
  fleet or that the operator judges complete however it ended — and that hand-recorded completion satisfies every edge
  standing on the chunk exactly as a fleet-earned one does. Blocked is a held state, never a dead end.
- **A prerequisite that vanishes.** A deleted chunk disappears from every listing, which would leave its dependents
  waiting on an id that no longer resolves. Deletion therefore refuses while an unreleased edge stands on the chunk and
  names the dependents holding it — one more reason on a refusal the delete path already makes, answered by the release
  lever the operator already holds. Folding is the opposite case: a prerequisite folded into a survivor carries its
  incoming edges along, edges internal to a fold are dropped rather than curled into self-edges, and a fold that would
  close a cycle is refused the way a declaration is.
- **Reach-ahead by default, waiting by choice.** The queue a runner peeks at carries its blocked entries marked rather
  than hidden, and the runner decides what to do with them. By default it reaches past a blocked chunk to the next one
  truly ready, so the fleet flows around a held chunk with no skip logic anywhere. A runner whose operator would rather
  it hold the line can be configured strict: work in queue order or not at all, idling while the head is blocked.
  Strictness is that runner's configuration, never the fleet's default — and hiding blocked chunks from the peek would
  take the choice away, because a strict runner cannot hold a line it cannot see.
- **Verification, not selection.** The claim protocol already puts the choice in the runner's hands — the runner names
  the chunk it wants, and the hub's role is to refuse, never to pick. The dependency check joins the refusals: whichever
  chunk a runner reaches for, head of queue or not, an unmet edge is a denied claim.
- **What the wire owes.** The denial and the marked queue entries both change the hub-to-runner wire, so the same
  landing extends the mock fleet: a mock hub that refuses a claim on an unmet edge and serves a queue with its blocked
  entries marked, and a mock runner that reaches past one under the default and holds at one under strict. The board's
  client is generated from the API, so the edge and the marking reach the web side by regeneration rather than by hand.

## An optional addon — the neighborhood view

Edges are the first data the board has ever held about how chunks relate, and a small picture falls out of them: a chunk
drawn with one hop each way — what it stands on, greyed out where done, and what stands on it, waiting its turn. One hop
is deliberately little, and it is the same little the marking already offers, naming the prerequisite immediately above
and going no further. An operator walks a coupled feature the way the fleet does, a chunk at a time, and the board never
has to decide where one story ends and the next begins — nothing here names a story or gathers chunks into one.

It is read-only by design — the same philosophy the board already lives by, a rich visual reading of the fleet while the
changes themselves stay agentic — and it is deliberately optional: the edge, the marking, and the enforcement stand
complete without it, and the view ships only if the slice has room.

## Artifacts

Both mockups explore a shape the slice has **not** adopted. They draw a first-class *story* — a named, selectable
collection of chunks joined by edges — and a whole story laid out at once, where the slice commits to a chunk's
neighbors one hop each way and to no story entity at all. They are how the larger shape was looked at, kept for the day
the edges are asked to carry more.

| Mockup                                                       | What it explores                                                                                                                                                                                                                                                 |
| ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [story-map-desktop.html](./artifacts/story-map-desktop.html) | The map at a desk: stories in a sidebar, the selected story's chunks drawn left to right with their edges — satisfied, open, and open-from-a-chunk-that-needs-you — and a detail rail carrying the operator's two levers, release-edge and declare-done-by-hand. |
| [story-map-mobile.html](./artifacts/story-map-mobile.html)   | The same map at thumb size: a story-picker chip row, the flow drawn top to bottom, and a bottom sheet holding the detail and the same two levers.                                                                                                                |

## Open questions

- How far outside its own work item a chunk may reach for a prerequisite — another graph, another source, and eventually
  another project. The answer leans toward allowing it: projects that are distinct still have real dependencies between
  them, and [`milestone:projects`](../../milestones.md) puts many of them under one hub. What that costs the store and
  the queue is what remains to settle.
