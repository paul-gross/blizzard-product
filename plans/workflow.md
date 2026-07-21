# Plan — `epic:workflow`

> **Shipped.** This plan is frozen: it records what the workflow engine was conceived to be, and its entry in [delivered.md](../delivered.md) is the record of what landed.

Every team has its own answer to "what happens to work before it counts as done" — and a platform that hard-codes one answer has taken a side in someone else's argument. The workflow engine refuses to: a **chunk** of work travels a user-defined graph of **nodes**, a **judgement** at each exit picks the outgoing edge, and **artifacts** accumulate as it goes. The old fixed pipeline, the verify step, the human-gate matrix — all of them collapse into graph shapes, and editing the graph *is* the dial between reviewing every step and supervising outcomes.

## What to build

- **The chunk.** The unit of orchestrated work: a hub-minted identity wrapping one or more PM pointers, holding its current node, history, judgements, questions, and artifacts. What a runner acquires, what travels the graph, what the record is about.
- **Graphs as hub configuration, in YAML.** Runners never define workflow; they execute node-steps. A node names a lifecycle station — build, review, deliver — carrying a prompt, checks, a judgement spec, and retry bounds. Edges are keyed by judgement outcome, and every outcome must have an edge. **Cycles are the point**: review-fail flowing back to build is the canonical shape, with retry caps as the escape into escalation.
- **Immutable graphs, pinned chunks.** A graph never changes — an edit mints a new one — and every chunk records the graph it was minted under and travels it to completion, so no chunk is ever stranded at a node its graph no longer has. Moving in-flight work between graphs is an explicit operation (`epic:migration`'s territory), never a side effect of editing.
- **The runner drives, the hub applies the rules.** A finished node-step is submitted — judgement, check results, artifacts, one atomic epoch-fenced write — and the hub's reply records the transition and returns the next node's envelope, which lets the runner continue in place: same environment, artifacts at hand, no re-queue. The envelope is pure prompt material — the agent sets up its own workspace from what is presented; agents manage environments, never the reverse.
- **Two-phase judgement.** A worker's exit is its done declaration; the judgement prompt is then delivered into the same session, and the reply's structured choice — elicited by an engine-generated tail, never hand-authored — selects the edge. A missing or unparseable choice is a failure: absence of success is never success.
- **Artifacts as the chunk's memory.** Pointers (branches and commits, pushed to the forge before submission) and assets (a findings document, a spike write-up), stored at the hub, fed into subsequent envelopes. A review's findings ride the fail edge back into build's prompt — and a chunk whose whole purpose is non-code work simply ends with assets. The hub stores pointers and assets, never code, never transcripts.
- **The default graph.** Build → review → deliver, review-fail cycling back to build, bounded retries escaping to escalation. It ships as the default YAML and reproduces exactly what a hard-coded pipeline would have done — as data.

## What this engine is not

It governs a chunk's *lifecycle* — which station, who judges, when it lands — never how the agent inside a node decomposes its own work. That belongs to the tooling below the fleet layer, and the mission's non-goal stands one layer down.
