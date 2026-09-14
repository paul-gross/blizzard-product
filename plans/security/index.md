---
epic: security
refinement: scaffolded
slices:
  - name: runner-auth
    status: delivered
  - name: worker-lockdown
    status: horizon
    plan: ./worker-lockdown.md
---

# Plan — `epic:security`

A fleet that works unattended is only as trustworthy as the walls around it. Two walls matter, and they face different
directions. The first faces outward: only runners the hub has vouched for may join the fleet and write into its record,
so a stranger on the network cannot pose as a worker. The second faces inward: a worker nobody is watching runs with the
permission prompts off, and the operator — not the agent's judgment — decides in advance what that worker may touch,
with the platform enforcing the decision rather than hoping.

The epic lands in two slices, one per wall:

| Slice                                   | What it builds                                                                                                                                     |
| --------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| runner-auth                             | The outward wall, delivered: runners sign in with tokens the hub mints for each of them. It landed without a plan.                                 |
| [worker-lockdown](./worker-lockdown.md) | The inward wall: permission profiles, a bounded reach over network and credentials, protection against force-pushes, and permissions set per node. |
