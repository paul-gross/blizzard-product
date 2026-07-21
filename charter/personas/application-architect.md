# `persona:application-architect` — the application architect

**The primary persona.** The agentic-native engineer this product is built by and for.

## Who they are

They have the technical range to describe a system in writing and work with models to produce larger technical plans and feature specs. They are far enough into agentic development that they more or less trust the output — not blindly, but because they trust themselves to recover it: when a run goes astray, they can drop into the exact session, see what the agent saw, and fix or redirect it. Trust, for this person, is not faith in the model; it is confidence in their own reach.

## What their work looks like

Evenings and weekends are when their fleet earns its keep. They file issues during the day as ideas arrive, queue work before stepping away, and expect the morning to greet them with landed changes — merged to main, or waiting at exactly the gates they chose — and, where something went wrong, an escalation precise enough to act on immediately. What they never expect to find is a wedged fleet or a mystery.

## What they need from blizzard

- **Unattended throughput** — file issues, walk away, return to landed work.
- **Truthful status at a glance** — the board never says something the system cannot back.
- **Cheap takeover** — one pasted resume command lands them in the stuck agent's full session context, not in a cold reconstruction of it.
- **Zero duplicated or clobbered work** — a structural guarantee, not a probabilistic one.
- **Polyrepo end to end** — their unit of work spans repositories, and the platform treats that as normal.

In short: the [mission](../mission.md)'s success criteria are this persona's needs, written down.

## What serves them

Everything in the core platform — the runner's store, the supervisor loop and its crash recovery, the harness adapters and their takeover flows, the board.
