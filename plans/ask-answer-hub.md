# Plan — `epic:ask-answer`, hub slice

> **Shipped.** This plan is frozen: it records what the ask/answer protocol was conceived to be, and its entry in [delivered.md](../delivered.md) is the record of what landed. The reach half — fan-out to phones and chat — is the [remote slice](./ask-answer-remote.md), still live.

This is the one genuinely new primitive blizzard introduces: an agent facing an undecidable choice asks a human a question **without dying for it**. Every fleet eventually meets the moment where the spec runs out — should the export include archived records? — and the naive answers are all bad: block and burn a live process for hours, guess and be wrong, or fail and throw the work away. Ask-and-exit is the fourth way: the worker asks, ends its turn, and the chunk parks — dormant, not dead — until the answer arrives and the session is resumed *around* it.

## What to build

- **The worker convention.** On an undecidable choice, run `blizzard runner ask "question" --options "a|b"` and end the turn. No blocking, no polling. The ask is a durable local fact before the process ends — which is exactly how the supervisor tells "parked on a question" from "died without a verdict": same exit, and the recorded ask is the difference between patience and a retry.
- **The question as a durable row.** The runner forwards the ask to the hub, where it lives as a row — never an in-flight message — so every hop survives a crash and the answer is re-deliverable. Open versus answered is derived from the presence of the answer, never stored.
- **Parking that costs nothing.** A parked chunk keeps its lease dormant and its environments warm, its reap clock stopped — it is waiting on a person, and waiting overnight is free.
- **Answering at the hub.** `blizzard hub status` shows what waits; `blizzard hub answer` writes the answer with first-write-wins arbitration — two simultaneous answerers resolve to exactly one, and the loser is told who won.
- **Resume-with-answer.** The supervisor delivers the answer by resuming the dormant session — the agent is reconstituted around the answer under the same lease, at the same node, and the chunk's derived status returns to running. The answering surface confirms *delivered, agent resumed*, closing the loop for the human.

## What was deliberately rejected

A blocking `--wait` mode — the CLI long-polling until the answer arrives — was considered and deferred: it holds a live process open for hours against a human's schedule, which is exactly the fragility everything else in the design avoids. Ask-and-exit is crash-equivalent to the rest of the system, and a bounded-timeout compromise remains open for the fast-answer case.
