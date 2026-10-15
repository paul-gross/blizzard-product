---
initiative: config
refinement: scaffolded
epics:
  - model-resolution
  - sweep
---

# Plan — `initiative:config`

A young platform is full of numbers somebody chose once. How long a lease lives, how many times a node retries before
escalating, how long the reconciler sleeps between sweeps — each was settled during a build by someone who was not
running this fleet, on this machine, against this repository. They are usually close enough to right, and when one is
wrong it is wrong in code, costing a change, a review, and a release.
[`persona:harness-engineer`](../../charter/personas/harness-engineer.md) feels this most sharply, because their whole
craft is comparison: run the same chunk twice with one variable moved and see what it cost. A constant they cannot reach
is a question they cannot ask. This is a survey-then-sweep effort rather than a large build.

One corner of that surface is answered early, because there the answer decides what a night costs: which model a session
actually runs on. The initiative lands in two epics:

| Epic                                            | What it builds                                                                                                                    |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| [model-resolution](./model-resolution/index.md) | A read-only runner command that shows how each tier, effort, and session pool resolves to a model, and which layer supplied each. |
| [sweep](./sweep.md)                             | The inventory and promotion of every other operational constant, under one resolution order.                                      |

The initiative's worked samples live in [artifacts/](./artifacts/model-resolution.txt): the model-resolution command's
terminal and JSON output, on one illustrative runner.
