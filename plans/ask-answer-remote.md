# Plan — `epic:ask-answer`, remote slice

Answering a question away from the machine only feels safe when the answer visibly arrives. The person who taps out a reply between errands is left holding a question of their own — did it get there? is the agent moving again? — and until the product answers *that*, remote answering stays something an operator double-checks from a desk later. This slice closes the exchange: the answer's return trip becomes something any surface can show, end to end.

## What already holds

More of the reach story is delivered than this slice once assumed. The protocol underneath — the [hub slice](./ask-answer-hub.md) — ships and works: durable question rows, the parked `waiting_on_human` chunk, resume-with-answer, first-write-wins arbitration at the hub. And the board already carries questions both ways: an open question appears the moment it is asked (the live event stream re-reads on ask and answer), and the chunk's dock takes an answer through the same hub route as the CLI. With the board reachable from a phone, checking in and answering from anywhere is real today.

## What to build

- **The closed loop on the question row.** The row tells the whole story, not just its opening: who answered, that the answer was delivered, and that the agent resumed. Each milestone of the return trip lands on the row and flows out over the event stream, so the board — and any future surface — renders *the fleet went back to work*, not merely *your form submitted*.
- **A loss told as a fact, not a failure.** Two people answering at once has always resolved to exactly one winner at the hub; the board should say so. The beaten answerer sees who got there first and what they said — an outcome, never an error screen.

## What this slice is not

Fan-out is no longer here. Pushing a question to wherever its humans are — notification channels, subscriptions, chat — belongs to `epic:chat`, which builds the channel seam together with its first binding rather than ahead of it. This slice makes the exchange trustworthy on the surfaces that exist; that one makes the fleet able to summon you.

## Open questions

- Which roles hold the answer grant: the permission seam exists (answering is already a distinct capability), so what remains is policy — whether any authenticated viewer may answer or only an operator, decided as role configuration lands.
