# `persona:harness-engineer` — the harness engineer

## Who they are

The fleet's mechanic. They care less about any single task's output than about the *ecosystem of outputs*: the distribution of outcomes across many runs, and how that distribution shifts when a parameter moves. When the automated system goes wrong, their first instinct is not to fix the one incident but to reconstruct exactly what happened and ask what setting would have prevented the class.

## What their work looks like

Tuning, mostly. Lease TTLs, heartbeat intervals, retry caps, tick rates, model and harness routing — they change one thing, run the fleet, and read the results. Their enemy is anything that makes two runs incomparable or an incident unreconstructable.

## What they need from blizzard

- **A configuration-based system.** Every operational constant lives in config, changeable without touching code. A constant buried in code is a dial they cannot turn.
- **Cheap forensics.** The system records what actually happened — facts, not self-reported statuses that can lie — so an incident's lifecycle is reconstructable by querying it. (The agent's *reasoning* lives in transcripts on the runner machine; the facts cover the lifecycle.)
- **Comparable runs.** A parameter change is evaluated against real outcome data, never against vibes.

## What serves them

The facts-only schema, `blizzard runner selftest` and pinned harness versions, and the planner's shadow-mode comparison discipline. This persona is the reason operational constants must land in config, not in code.
