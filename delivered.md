# Delivered

The ledger of landed work, told the way the work was conceived: by destination. Each delivered milestone keeps a section with the grid of epic slices that carried it.

## `milestone:mvp` — one operator, one machine

A working fleet for a single operator on a single machine: colocated hub and runner, the local board, and the full autonomous baseline from ingest to merge.

| Epic | Slice delivered | Notes |
|------|-----------------|-------|
| `epic:hub` | separation slice | Colocated hub + runner↔hub protocol, queue, HTTP API + SSE, merge queue. Remote slice is in the [epic registry](./epics.md). Built from its [plan](./plans/hub-separation.md). |
| `epic:store` | full | The runner's embedded database: leases, epochs, facts-only schema. Built from its [plan](./plans/store.md). |
| `epic:supervisor` | full | The runner reconciliation loop, crash recovery, environment leasing. Built from its [plan](./plans/supervisor.md). |
| `epic:adapters` | full | Harness adapters (spawn / resume / verdict), hooks, selftest, human takeover. Built from its [plan](./plans/adapters.md). |
| `epic:workflow` | full | Hub-defined YAML graphs, nodes and judgements, sticky advancement, artifacts. Built from its [plan](./plans/workflow.md). |
| `epic:review` | full | The default graph's review node. Built from its [plan](./plans/review.md). |
| `epic:delivery` | full | Hub-executed deliver node, merge queue, epoch-fenced submission (#62 series). Built from its [plan](./plans/delivery.md). |
| `epic:ask-answer` | hub slice | Question rows at the hub, ask/answer CLI verbs. Remote slice is in the [epic registry](./epics.md). Built from its [plan](./plans/ask-answer-hub.md). |
| `epic:board` | local slice | Hub-served web app: fleet observability, queue prioritization, chunk grouping. Remote/auth slice is in flight in the [epic registry](./epics.md). Built from its [plan](./plans/board-local/index.md). |
| `epic:gates` | full | Gate nodes plus runner-config gates by node name; decision parking; CLI + board surfacing. Built from its [plan](./plans/gates.md). |

## `milestone:mvp-feedback` — the lessons of running it for real

The MVP put a real fleet in an operator's hands, and running it taught what the platform still owed them: an operator can now see and bound what the fleet spends, and evolve workflow graphs underneath running work.

| Epic | Slice delivered | Notes |
|------|-----------------|-------|
| `epic:cost` | core | Usage facts, hub flow-through, board surfacing, budget caps + spend kill-switch (#57 series). Remaining unfiled sub-bullet: model routing by cost. Built from its [plan](./plans/cost.md). |
| `epic:migration` | core | Explicit migration, `migration_target` auto-drift, retire/re-enable, graph metadata edits (#101, #120, #124 …). Correctness tail is in flight in the [epic registry](./epics.md). Built from its [plan](./plans/migration.md). |

## Outside the milestones

| Epic | Slice delivered | Notes |
|------|-----------------|-------|
| `epic:runner-panel` | core | The runner-served machine panel and transcript viewer: local truth — leases, capacities, escalations with copyable resume commands, transcripts — beside shared fleet views, one codebase with two thin apps. Built from its [plan](./plans/runner-panel/index.md). |
