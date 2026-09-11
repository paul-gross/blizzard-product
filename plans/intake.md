# Plan — `epic:intake`

Queuing work today means having the id in hand. The operator reads the backlog in the forge, copies an identifier, hands
it to blizzard, and repeats — which is fine for one chunk and absurd for an afternoon of triage. Working a backlog down
is a different act from queuing a chunk, and it deserves a surface built for it: somewhere the operator browses what is
open, chooses several, and shapes what lands. The forge stays the backlog's home. This screen moves its items into the
fleet with less friction; it never defines work.

## What to build

- **A screen of its own.** The board's per-card grouping controls grow into a workbench: queued and candidate work side
  by side, inspected, grouped, ungrouped, and sent through ingest in one sitting.
- **Browsing, as a loosening of the work-source seam.** The seam gains an optional capability to enumerate and search,
  which is a deliberate departure from id-addressed ingest and is decided here. A source that cannot enumerate still
  ingests by id, and the surface degrades to that honestly.
- **Many at once.** Selecting several items and pulling them through ingest together, then grouping what belongs
  together before any of it is claimed.
- **Ungrouping.** Splitting a folded chunk back apart, which grouping cannot do today and which an operator who grouped
  hastily needs immediately.

## Open questions

- What search means across a hub of several projects and sources with different query languages — one federated box, or
  one source at a time.
- Whether ungrouping is a real split of a chunk or a re-ingest of its parts, and what happens to facts already recorded
  against the folded chunk.
- How much of the forge's own item detail this surface shows before it starts re-implementing the forge.
