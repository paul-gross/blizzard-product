# Milestones

What users will be able to do. A milestone is a destination stated in the user's terms, and it comes first: you declare where the product must reach, then ask what work the journey requires — the milestone demands its epics, never the other way around.

| Milestone | What users will be able to do |
|-----------|-------------------------------|
| `milestone:centralized-hub` | Run their fleet from anywhere: one hub off the operator's machine, serving multiple runners — the fleet stops being something you have to be sitting at. |

## `milestone:centralized-hub` — run the fleet from anywhere

Today the hub lives where the operator sits: it shares a machine with its runner, and being away from that machine means being away from the fleet. This milestone breaks that tie. The hub becomes something the operator reaches, not something they sit at — one hub, off the machine, and every runner in their fleet signs in to it from wherever it runs.

What that unlocks is told best through the people in the [charter](./charter/personas.md). The application architect queues work Friday evening and checks the board from wherever the weekend takes them. The product owner checks in from a phone: the board in their pocket shows the question a parked agent is waiting on, and answering it right there sends the fleet back to work. And a second machine — a home server, a spare laptop — becomes more fleet with a runner install, not a second island with its own hub.

The boundary is deliberate: this milestone makes the fleet reachable, not insistent. Checking in is the promise; being summoned — a question pushed to a pocket the moment it is asked — waits for the notification fan-out that `epic:chat` builds with its first channel.

The journey demands four pieces of work, and two have already landed: runners sign in to the hub with hub-minted tokens — the runner-auth slice of `epic:security` — and the board is safe to expose, with login, viewer and operator roles, and a session-aware UI (#89). The hub's remote slice does the heavy lifting: an off-machine deployment serving multiple runners over the existing outbound-only protocol. What remains of the ask-answer remote slice is the closed loop: the person who answers — from the board, on any device — sees that the answer was delivered and the agent resumed, and a beaten answerer is told who won rather than shown an error.

| Epic | Slice | Status |
|------|-------|--------|
| `epic:security` | runner-auth | delivered |
| `epic:board` | remote (auth) | delivered |
| `epic:hub` | remote | horizon |
| `epic:ask-answer` | remote (closed loop) | horizon |
