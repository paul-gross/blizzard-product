---
epic: security
refinement: scaffolded
slices:
  - name: runner-auth
    status: delivered
  - name: worker-security
    status: in-progress
    plan: ./worker-security/index.md
---

# Plan — `epic:security`

A fleet that works unattended needs two decisions about trust. The first faces outward: only runners the hub has vouched
for may join the fleet and write into its record, so a stranger on the network cannot pose as a worker. The second
belongs to the operator of each runner: what the coding harness may do when no one is watching, and which of its own
settings and extensions it brings to the work. The runner still needs its hooks and headless rules, but the operator
should be able to set the rest of the terms. A runner-supplied configuration joins the harness's existing user and
project settings rather than erasing them; where ordinary choices overlap, the runner's choice takes precedence, while
permission rules and managed policy keep their harness-native precedence.

The epic lands in two slices, one per decision:

| Slice                                         | What it builds                                                                                                                 |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| runner-auth                                   | The outward wall, delivered: runners sign in with tokens the hub mints for each of them. It landed without a plan.             |
| [worker-security](./worker-security/index.md) | Runner-loaded native configuration alongside existing local settings, with one autonomy choice and required headless behavior. |
