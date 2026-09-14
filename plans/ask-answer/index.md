---
epic: ask-answer
refinement: pristine
slices:
  - name: hub
    status: delivered
    plan: ./hub.md
  - name: remote
    status: delivered
    plan: ./remote.md
---

# Plan — `epic:ask-answer`

An agent facing an undecidable choice asks a human a question **without dying for it**. Every fleet eventually meets the
moment where the spec runs out, and the naive answers are all bad: block and burn a live process for hours, guess and be
wrong, or fail and throw the work away. Ask-and-exit is the fourth way: the worker asks, ends its turn, and the chunk
parks — dormant, not dead — until the answer arrives and the session is resumed around it.

The [hub slice](./hub.md) built the protocol: durable questions at the hub, the parked chunk, resume-with-answer, and
first-write-wins arbitration. The [remote slice](./remote.md) closed the exchange for a person answering away from the
machine, so the answer's return trip — delivered, and the agent back at work — is something any surface can show.
