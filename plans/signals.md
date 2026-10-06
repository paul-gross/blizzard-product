---
epic: signals
refinement: scaffolded
slices:
  - name: full
    status: horizon
---

# Plan — `epic:signals`

Blizzard only ever finds work by being told to look. Everything it does begins with an operator naming an item, which
means the systems that notice trouble first — the error tracker that just saw a new exception, the alerting service
watching a service degrade — have no way to reach the fleet that could act on it. The operator is the wire again, and
this time they are asleep. This epic lets something happen and become work: an external system pushes into the fleet,
and the item it raises is queued before morning.

## What to build

- **An inbound endpoint, per project.** Configured per project, against a source that project links, which is why this
  follows `epic:projects` rather than preceding it.
- **Signed requests.** A push surface on a hub that is reachable from the internet is only safe if the hub can tell who
  is pushing.
- **Rate bounds per source.** A flapping alert must flood nothing. A source that exceeds its bound is throttled rather
  than believed.
- **The forge stays the only place work is defined.** A signal raises an item in a work source the project links, and
  blizzard ingests it into that project like any other item — so the non-goal holds by construction rather than by
  discipline. This is the first push binding of a seam that has only ever pulled.

## Open questions

- The signing scheme, and how a secret is rotated without a window where either the old or the new one is refused.
- What a signal is allowed to say about the work it raises — which graph, which tags, what priority — and how much of
  that a stranger should be trusted with.
- How a repeating alert is recognized as the same alert, so a service that is down for an hour raises one item rather
  than sixty.
- What an endpoint belongs to. `epic:projects` gives a project no sources and no secrets of its own: a work source and
  the credentials it uses belong to the tenant, and a project only links them. An endpoint "per project" therefore has
  to keep its signing secret on some tenant record — the source it raises items in, the project's link to that source,
  or a signal record of its own — and the rate bound above is already stated per source.
- How a push names its tenant and project. A push carries no session and no runner credential, so the endpoint's address
  or its signing secret has to settle both, without a tenant id that is meant to stay private appearing in a public URL.
