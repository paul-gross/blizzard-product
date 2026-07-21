# Plan — `epic:chat`

The fleet's best questions arrive at the worst times: overnight, over a weekend, while [`persona:product-owner`](../charter/personas/product-owner.md) is anywhere but at a browser. A chat bot is the channel that is already in their pocket. This epic gives the fleet one: question notifications that arrive as messages, answers that cost one tap, and a confirmation that the agent picked the answer up and went back to work.

It rides on `epic:ask-answer`'s remote slice — the fan-out and answer-back plumbing lives there ([plan](./ask-answer-remote.md)); this epic is the first chat binding of the human-channel seam.

## What to build

- **The Telegram bot.** Telegram is the first binding — chosen for having the best inline-answer-button experience of the candidates — and the chat layer stays a seam, so a second platform is an adapter, never a rewrite.
- **One-tap answers.** A question arrives carrying its options as inline buttons; tapping one is the whole act of answering. Free-text questions accept a typed reply. The bot submits through the same hub route as every other client, with the same first-write-wins arbitration — beaten answerers are told who got there first.
- **Delivery confirmations.** The message thread closes its own loop: *delivered, agent resumed* lands in the chat, so the person who answered from a parking lot knows the fleet is moving again.

## Open questions

- Identity: how a Telegram account maps to a hub user once the board's auth lands, and whether an unmapped account may answer at all.
- Where the bot runs — inside the hub daemon or beside it — and what its failure does to the fan-out contract (nothing may be lost: the question row is durable either way).
