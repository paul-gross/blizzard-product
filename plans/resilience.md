# Plan — `epic:resilience`

At one in the morning a model provider stops answering. Nobody is awake, and the fleet — which has no concept for "the
weather turned bad" — reads the silence as work that failed. Every chunk in flight escalates, the queue fills with
failures that describe nothing an operator can act on, and by breakfast the night is gone and the morning starts with
cleanup. What the operator wanted was the obvious thing: for the fleet to wait, and then carry on. This epic teaches it
to.

## What to build

- **An outage is its own class of failure.** A provider that is unreachable, or a rate limit that has closed the window,
  is recognized as an environmental condition rather than as a chunk that went wrong — and nothing escalates on its
  account.
- **Waiting that keeps its place.** The affected work holds its lease, keeps its position in the queue, and resumes
  where it stopped once the provider answers again. A night interrupted becomes a night delayed.
- **Waiting that says so.** A fleet that is waiting looks different from a fleet that is idle. The operator who checks
  at six sees what the fleet is waiting on and since when, without reading a log.
- **Falling back before waiting.** Where another route exists the fleet takes it: down the model tiers first, and across
  coding harnesses once `epic:adapters` has given it a second one to fall back to.
- **A floor under the patience.** Waiting is bounded. An outage that outlasts the bound becomes one ordinary escalation
  about the outage, not one per chunk.

## Open questions

- How an outage is recognized through each harness, whose failures arrive as exit codes and stderr rather than as a
  status an API would hand over.
- What a held lease means to heartbeating and fencing while the worker is deliberately doing nothing — a waiting worker
  must not read as a dead one.
- Where the bound on patience is set, and whether it belongs to the runner (whose machine is idle) or to the graph node
  (whose work may be time-sensitive).
