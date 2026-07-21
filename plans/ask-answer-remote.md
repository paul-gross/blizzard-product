# Plan — `epic:ask-answer`, remote slice

An agent halfway through a feature hits a question no spec ever answered — should the export include archived records? — asks it, and goes dormant. Today the answer requires being at the machine: the question surfaces in `blizzard hub status`, and `blizzard hub answer` sends the agent back to work. The person who actually holds that answer — [`persona:product-owner`](../charter/personas/product-owner.md) — is away from a desk more often than at one. This slice carries the question the last mile: out to wherever the humans are, and their answer back.

## What already holds

The protocol underneath — the [hub slice](./ask-answer-hub.md) — is delivered and does not change. Ask-and-exit, the durable question row at the hub, the parked `waiting_on_human` chunk with its reap clock stopped, resume-with-answer under the same lease — all of it ships and works. What is missing is purely *reach*: the question exists as a row a phone could read, and nothing yet puts it on the phone.

## What to build

- **Fan-out.** An open question reaches its humans instead of waiting to be queried: it appears on the board the moment it is asked, and is pushed to whatever notification channels are configured — the chat bot (`epic:chat`) is the first, and the channel is a seam, not a Telegram assumption.
- **Answers from remote clients.** The board and any chat client answer through one hub route. Two people answering at once resolve the way the protocol always has: first write wins, and the loser is told who beat them — an answer is never silently discarded and never applied twice.
- **The closed loop.** The answering surface confirms *delivered, agent resumed* — the person who answered sees the fleet go back to work, not just their half of the exchange. That confirmation is what makes answering from a phone feel safe enough to do from a phone.

## Open questions

- The notification transport's shape: whether the hub pushes to channels directly or exposes a subscription the channels poll — decided together with `epic:chat`'s bot.
- Whether answering from the board requires the operator role once the board's auth lands, or any authenticated viewer may answer.
