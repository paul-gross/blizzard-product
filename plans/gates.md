# Plan — `epic:gates`

> **Shipped.** This plan is frozen: it records what human gates were conceived to be, and its entry in [delivered.md](../delivered.md) is the record of what landed.

Trust in an autonomous fleet is not granted, it is dialed — and the gates are the dial. [`persona:senior-se`](../charter/personas/senior-se.md) starts wanting a say at every station and earns their way toward supervising outcomes; [`persona:application-architect`](../charter/personas/application-architect.md) runs the gateless default from day one. Both must be configuration, never code, and never a fork of anyone else's workflow. A gate is simply a station whose judge is a person.

## What to build

- **Gate nodes in the graph.** A node judged by a human instead of a worker: the fleet-wide lever, expressed in the same graph YAML as every other station. The default graph carries none — the autonomous baseline survives as *the default graph containing no gate nodes*, a shape, not a switch.
- **The runner-config dial.** An individual operator imposes a gate on any node by name — `gates = ["deliver", …]` in their own runner's configuration, matched across all graphs — so one cautious operator dials their own oversight up without forking the fleet's graph or slowing anyone else down.
- **Decisions: parking with a form attached.** A chunk reaching a gate parks as an open **Decision** — a durable multiple-choice row carrying the step's artifacts, reusing the same machinery as an agent's question: the chunk derives waiting-on-human, its reap clock stops, its environments stay warm. The person's choice *is* the judgement — it resolves into the transition, rendered as buttons from the same choice set a worker would have emitted.
- **Surfaced where humans already look.** Open decisions appear through the CLI and the board, both resolving against the same hub route.

## The shape underneath

Mechanically the two levers are one mechanism: a graph gate is the hub refusing to transition at a human-judged node; a runner-config gate is the runner submitting a Decision instead of a transition for a node it was told to gate. One mechanism, two owners — the team's process in the graph, the individual's caution in their own config.
