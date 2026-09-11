# Plan — `epic:signals`

Blizzard only ever finds work by being told to look. Everything it does begins with an operator naming an item, which
means the systems that notice trouble first — the error tracker that just saw a new exception, the alerting service
watching a service degrade — have no way to reach the fleet that could act on it. The operator is the wire again, and
this time they are asleep. This epic lets something happen and become work: an external system pushes into the fleet,
and the item it raises is queued before morning.

## What to build

- **An inbound endpoint, per project.** Configured against the project's own sources, which is why this follows
  `epic:projects` rather than preceding it.
- **Signed requests.** A push surface on a hub that is reachable from the internet is only safe if the hub can tell who
  is pushing.
- **Rate bounds per source.** A flapping alert must flood nothing. A source that exceeds its bound is throttled rather
  than believed.
- **The forge stays the only place work is defined.** A signal raises an item in the project's own work source, and
  blizzard ingests it from there like any other item — so the non-goal holds by construction rather than by discipline.
  This is the first push binding of a seam that has only ever pulled.

## Open questions

- The signing scheme, and how a secret is rotated without a window where either the old or the new one is refused.
- What a signal is allowed to say about the work it raises — which graph, which tags, what priority — and how much of
  that a stranger should be trusted with.
- How a repeating alert is recognized as the same alert, so a service that is down for an hour raises one item rather
  than sixty.
