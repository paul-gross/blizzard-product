---
epic: hub
refinement: pristine
slices:
  - name: separation
    status: delivered
    plan: ./separation.md
  - name: remote
    status: delivered
    plan: ./remote.md
  - name: distribution
    status: delivered
    plan: ./distribution.md
---

# Plan — `epic:hub`

A fleet needs a shared memory and a front door, and neither can live inside any single runner. The hub is both: the
daemon that knows what work exists, where every chunk stands, and how a human reaches the fleet. Runners speak to it
only through an outbound-only protocol, whether it shares their machine or sits across the internet — so where the hub
runs is a deployment choice, and how it is spoken to never changes.

The [separation slice](./separation.md) built that daemon colocated with its runner. The [remote slice](./remote.md)
moved it off-machine, where every runner in the fleet signs in from wherever it runs. The
[distribution slice](./distribution.md) made it something a stranger can run: one public container image and the release
discipline that promise demands.
