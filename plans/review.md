# Plan — `epic:review`

> **Shipped.** This plan is frozen: it records what the review station was conceived to be, and its entry in [delivered.md](../delivered.md) is the record of what landed.

The autonomous baseline only works if somebody impartial looks at the work before it lands — and in the baseline that somebody is another agent. The deliberate scope decision here is that blizzard builds **no review machinery of its own**: the default graph's review station is one node with one prompt, and the prompt invokes the review tooling of the stack below the fleet layer. In the reference stack that means the reviewing agent runs the project's own review engine, checks, and end-to-end flows *inside the chunk's environment* — where the services are running and can actually be driven — and writes what it finds.

## What to build

- **The review node of the default graph.** A prompt that puts fresh eyes on the chunk: the node opts into a cold session rather than resuming the builder's, because a reviewer who shares the builder's context shares the builder's blind spots.
- **Findings as an artifact.** The review's output is a findings document submitted as an asset. A *fail* judgement carries it back along the edge into the build node's envelope — so the builder's next attempt starts from exactly what the reviewer objected to, not from a summary of a summary.
- **The verdict as the judgement.** The reviewing worker renders the structured pass/fail choice, informed by the checks it ran in-session. Blizzard supplies the station, the cycle, and the memory; the review's *content* is the stack's own tooling, expressed as data in the graph YAML.

## Why this is an epic at all

Because the promise it keeps is a product promise: work merges to main with no human in the loop *and someone looked at it first*. The engine makes stations cheap; this epic is the station that makes the gateless default trustworthy.
