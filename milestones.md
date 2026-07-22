# Milestones

What users will be able to do. A milestone is a destination stated in the user's terms, and it comes first: you declare where the product must reach, then ask what work the journey requires — the milestone demands its epics, never the other way around.

| Milestone | What users will be able to do |
|-----------|-------------------------------|
| `milestone:centralized-hub` | Run their fleet from anywhere: one hub off the operator's machine, serving multiple runners — the fleet stops being something you have to be sitting at. |

## `milestone:centralized-hub` — run the fleet from anywhere

Today the hub lives where the operator sits: it shares a machine with its runner, and being away from that machine means being away from the fleet. This milestone breaks that tie. The hub becomes something the operator reaches, not something they sit at — one hub, off the machine, and every runner in their fleet signs in to it from wherever it runs.

What that unlocks is told best through the people in the [charter](./charter/personas.md). The application architect queues work Friday evening and checks the board from wherever the weekend takes them. The product owner's questions stop being desk-bound: an agent's ask reaches their phone, and a one-tap answer sends the fleet back to work. And a second machine — a home server, a spare laptop — becomes more fleet with a runner install, not a second island with its own hub.

The journey demands four pieces of work, and the first has already landed: runners sign in to the hub with hub-minted tokens — the runner-auth slice of `epic:security` — the precondition for a hub that answers to the open network. The hub's remote slice does the heavy lifting: an off-machine deployment serving multiple runners over the existing outbound-only protocol. The ask-answer remote slice carries questions the last mile — fan-out to wherever the humans are, answers flowing back from the board or a chat client. And the board's remote slice makes the web app safe to expose: login, viewer and operator roles, session-aware UI — the auth work now delivered (#89), marked in the epic chart below; its phone-friendly reach remains in the [epic registry](./epics.md).

| Epic | Slice | Status |
|------|-------|--------|
| `epic:security` | runner-auth | delivered |
| `epic:board` | remote (auth) | delivered |
| `epic:hub` | remote | horizon |
| `epic:ask-answer` | remote | horizon |
