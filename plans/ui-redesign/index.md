---
epic: ui-redesign
refinement: scaffolded
status: horizon
---

# UI redesign

The board's second design: a hub board and fleet view built for the operator as the power user they are, on desktop and
on the phone. The first design proved what the operator needs on one screen: what is running, what is stuck, what is
waiting on them, and what it all cost. It is dense and honest. Its hierarchy is flat, though: the two chunks that need a
human weigh the same as fifty-three that are ready. Its visual language is a generic dark dashboard rather than an
instrument shaped to the work. This epic replaces that language and hierarchy and keeps the density and the truth.

It serves `persona:application-architect` first and `persona:harness-engineer` beside them. At the desk it is a
keyboard-first instrument the operator reads at a glance and acts from without hunting. On the phone it answers one
question, whether the fleet needs them, and lets them act on the answer.

## Proposals

Eight proposals explore the space before any requirement is decided. Three take the modern frontier, two deliberately
work outside current convention, and three are classic SaaS: a control to measure the others against, and two variations
on it drawn from other families of SaaS product. Each answers the shared
[brief](./artifacts/brief.md) with a concept, working desktop and mobile mocks of the board and fleet, and rendered
screenshots. The [baseline](./artifacts/baseline/) is the first design as captured from the live hub, and the
[comparison](./artifacts/comparison.md) sets the eight side by side.

| Proposal                                     | Direction   | In one line                                                             |
| -------------------------------------------- | ----------- | ----------------------------------------------------------------------- |
| [Editorial](./artifacts/editorial/README.md) | Modern      | The board as a front page; type weight and width carry urgency          |
| [Altitude](./artifacts/altitude/README.md)   | Modern      | One space at five fidelities, driven by a composable query line         |
| [Current](./artifacts/current/README.md)     | Modern      | The fleet as a timetable chart over time; bounces drawn as loops        |
| [Synop](./artifacts/synop/README.md)         | Avant-garde | The fleet as a weather chart; blockers are fronts, throttles fog        |
| [Tacit](./artifacts/tacit/README.md)         | Avant-garde | The fleet proceeds on tacit consent; the operator holds the veto        |
| [Console](./artifacts/console/README.md)     | Classic     | Classic SaaS: collapsible left nav, grouped table, drawer, compact mode |
| [Desk](./artifacts/desk/README.md)           | Classic     | A shared support inbox: what needs the operator is triaged to zero      |
| [Dispatch](./artifacts/dispatch/README.md)   | Classic     | A dispatch console: exceptions pinned, slots over time, honest spend    |
