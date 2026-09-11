# Plan — `epic:queue`, priority slice

The ready queue is ordered by the accident of arrival. An operator who knows one chunk matters more than the four ahead
of it has no way to say so, and a chunk nobody champions sinks a little further every time newer work arrives. Arrival
order is honest only while the queue drains as fast as it fills, and a fleet working through the night will outrun the
operator feeding it. Priority makes the first sayable. Aging makes the second impossible, so that having waited longest
eventually counts for something.

## What to build

- **Priority the operator can state.** A queued chunk carries a priority the operator sets and changes, from the board
  and from the CLI, without touching anything else about the chunk.
- **Aging that accrues.** Time waiting counts toward claim order, so a chunk nobody champions eventually reaches the
  front on its own rather than waiting for an advocate.
- **Order as a derived answer.** Claim order is computed from priority, age, and arrival at the moment of claiming —
  never a stored rank that drifts out of agreement with the facts behind it.
- **The reasoning visible.** An operator looking at the queue can tell why the chunk at the top is at the top, which is
  the difference between a queue they trust and one they fight.

## Open questions

- The shape of the aging curve, and the only question that really matters about it: how long a low-priority chunk can be
  made to wait before the fleet is simply wrong.
- Whether priority is scoped per project once `epic:projects` lands, since a global ordering across projects rewards
  whoever shouts loudest.
- How ordering composes with a runner's tag filters from `epic:tagging` — the top of the queue is not the top of every
  runner's queue.
