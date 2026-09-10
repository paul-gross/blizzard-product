# Fusion

Fusion — [github.com/Runfusion/Fusion](https://github.com/Runfusion/Fusion), MIT, published as `@runfusion/fusion` —
bills itself as "a software factory, run by a multi-agent orchestrator." It is the closest thing on the open market to
what blizzard is building: work arrives as a description or an imported issue, agents plan it, build it, review it, and
land it, and the operator watches from a board. Every product in this space is worth a glance; this one is worth a file.

The two products aim at the same day and reach it from opposite ends. Fusion begins at the developer's surface — the
board, the phone, the chat window, the model picker — and works inward toward the loop. Blizzard begins at the loop —
the queue, the lease, the fact log, the fencing — and works outward toward the surface. That difference explains almost
every row in the comparison, and it is why neither product's feature list reads as a subset of the other's.

| File                                                   | When to read                                                                                                                                    |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| [capability-comparison.md](./capability-comparison.md) | Weighing a capability against what Fusion ships — the two-way inventory, where the products converge, and what the differences ask us to decide |

## Reading these findings honestly

Fusion is roughly a year older in the market and ships weekly against a version number in the `0.78` range, with a
public task numbering past FN-9280; blizzard is at its first release candidate. Breadth comparisons flatter the older
product and say nothing about direction, so a Fusion feature blizzard lacks means one of three quite different things,
and the comparison is only useful when it says which:

- **A schedule difference.** Blizzard has already priced the work and placed it in [`epics.md`](../../../epics.md);
  Fusion got there first.
- **A position difference.** Blizzard's [mission](../../../charter/mission.md) names the capability a non-goal. The gap
  is the design, and the comparison confirms it rather than challenging it.
- **A genuine gap.** Neither scheduled nor refused — an idea worth a decision it has not yet had.

One method note, learned the hard way. A first pass built from both products' prose got several rows wrong in the same
direction — crediting Fusion with capabilities blizzard already ships, because blizzard's documentation is written for
operators and release engineers rather than as a feature inventory, and absence from the docs read as absence from the
product. Every claim of the form "blizzard does not have X" in the comparison has since been checked against shipped
source: the packaged graphs, the runner's harness adapters, the hub's delivery scripts. A future pass should assume the
same asymmetry and start from the code.

Findings are recorded against Fusion at commit `1a0b95b` (2026-09-09), read from the public repository and its `docs/`
tree, and against blizzard at `470989b`.
