# Plan — `epic:board`, remote slice

The board already answers "where is everything?" — for anyone standing at the machine. This slice takes that answer wherever its people are: the same app, reachable from a phone on a Saturday, safe to expose because it finally knows who is looking at it. It is the human-facing spoke of [`milestone:centralized-hub`](../../milestones.md), and it is one app throughout — going remote adds reach and roles to the board that exists, never a second board.

## What to build

**Auth comes first, and is in flight.** A board on the open network needs a front door before it needs anything else:

- **Login and human identity.** Operators sign in; a hub that answers to the network stops being a hub that answers to anyone who finds it. Locally, the authless-behind-the-network-perimeter posture remains a supported configuration — auth is a capability, not a toll.
- **Viewer and operator roles.** The person who may look and the person who may act are different grants. A viewer sees everything and touches nothing; an operator answers questions, resolves gates, and shapes the queue. Exactly which controls the operator role spans is decided as the role lands.
- **A capability-aware UI.** The board renders what the session may do: a viewer is never shown buttons that would fail, and an operator's controls follow their grant.
- **Runner single sign-on.** The runner's own panel participates in the same identity, so a fleet with a signed-in board never has an unsigned back door.

**Then reach.** The phone-friendly progressive-web-app shell — the board installed on a home screen, laid out for a thumb — and question notifications surfacing through it, with answers flowing back over the same routes as everywhere else (`epic:ask-answer`'s remote slice carries the fan-out).

## Artifacts

The board's founding mockups — the concept round that settled the design language the remote slice keeps — live with the [local slice's plan](../board-local/index.md), the work they were drawn for.

## Open questions

- The identity provider posture for a cloud-hosted hub: which OAuth2-class arrangement blizzard blesses first, and what the self-hosted default is.
- Whether answering a question requires the operator role or any authenticated viewer may answer — shared with the ask/answer plan.
