# Plan — `epic:tagging`

An operator with three machines has three identical appetites, because a runner has no way to say what kind of work it
is for. The one on the desk should take small, safe fixes; the one in the corner should be kept clear of anything risky;
and the queue, which the operator can only reorder, offers no way to express either. Work that says what kind of work it
is makes the queue something they can partition instead of merely shuffle.

## What to build

- **Scoped tags on a chunk.** `type:feature`, `risk:low` — a scope admits one value at a time, so a filter over a scope
  can be exhaustive and a chunk cannot quietly be two kinds of thing at once.
- **Where a tag comes from.** Tags arrive at ingest, are set by the graph, or are placed by the operator; all three
  write the same kind of fact, and the chunk carries them from ingest to landing.
- **A runner's accept-and-refuse declaration.** Each machine states the tags it will take and the tags it will not, and
  refusal wins where both could match.
- **The partitioned queue, visible.** The board and CLI show the queue as each runner sees it, so an operator can tell
  starvation from selectivity.

## Open questions

- The scope vocabulary: a small set blizzard ships and documents, or scopes an operator defines — and whether an unknown
  scope is an error or ordinary.
- Whether project is expressed as a tag once `epic:projects` gives projects a home, rather than as a parallel concept
  with its own filters.
- What a chunk with no tags means to a runner that accepts only some — taken by default, or left alone.
