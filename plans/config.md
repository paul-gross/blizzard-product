# Plan — `epic:config`

A young platform is full of numbers somebody chose once. How long a lease lives, how many times a node retries before
escalating, how long the reconciler sleeps between sweeps — each was settled during a build by someone who was not
running this fleet, on this machine, against this repository. They are usually close enough to right, and when one is
wrong it is wrong in code, costing a change, a review, and a release.
[`persona:harness-engineer`](../charter/personas/harness-engineer.md) feels this most sharply, because their whole craft
is comparison: run the same chunk twice with one variable moved and see what it cost. A constant they cannot reach is a
question they cannot ask. This is a survey-then-sweep effort rather than a large build.

## What to build

- **The inventory.** A deliberate sweep for operational constants across hub, runner, and graph execution, each one
  named and located. The sweep is the work; the plumbing after it is mechanical.
- **Every constant promoted.** Each becomes configuration with the current value as its declared default and a written
  account of what moving it does, so a reader can tell a tuning knob from a load-bearing invariant.
- **One resolution order.** Where a value may be stated in more than one place, the precedence is declared once and
  holds everywhere, so the answer to "what is this fleet actually running with" is a lookup rather than an
  investigation.
- **The comparable run.** The harness engineer changes one constant, runs the same chunk again, and can attribute the
  difference — which means the values a run used are recoverable after the fact, not only before it.

## Open questions

- Which constants belong to the hub, which to the runner, and which to a graph node — and the few that must stay fixed
  because correctness rests on them.
- Whether the resolved configuration of a run is recorded as a fact, which is what makes comparison across nights
  trustworthy and is also a new row on every run.
- How far the sweep reaches into the harness adapters, whose own knobs are not blizzard's to redefine.
