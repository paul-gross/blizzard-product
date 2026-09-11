# Plan — `epic:worker-profiles`

A graph binds session settings per node, which is right — a reviewer and a builder should not run the same way. What is
wrong is that every node spells its settings out, so changing how every reviewer in the fleet runs is one edit per node,
and a graph author comparing two configurations is comparing two piles of restated numbers. A profile is the indirection
that fixes it: settings named once and inherited whole.

## What to build

- **A profile as a named bundle.** The model tier, effort, context ceiling, rotation thresholds, and permission posture
  a session runs under, carried under one name.
- **A node that either spells it out or names one.** Nodes keep the freedom to state their own settings; naming a
  profile inherits them whole, and the two forms are equally legitimate.
- **The name on the fact.** A run records the profile it ran under, so a runner and a fact have a name to hold and an
  operator can ask what every reviewer-profile session cost last week.
- **Nothing more than that.** This epic refuses the identity layer that tends to grow around it — roles, reporting
  lines, an org chart of agents — which answers no question blizzard's fact log cannot already answer.

## Open questions

- Whether a node may name a profile and override a single field, which is obviously convenient and quietly reintroduces
  the restatement this epic exists to remove.
- Where profiles live: beside the graph that uses them, or in hub configuration where an operator can change them
  without touching a graph.
- What a changed profile does to comparison across nights, since two runs under the same profile name may no longer have
  run the same way.
