# Delivered

The ledger of landed work, told the way the work was conceived: by destination. Each delivered milestone keeps a section with the grid of epic slices that carried it.

## `milestone:mvp` — one operator, one machine

A working fleet for a single operator on a single machine: colocated hub and runner, the local board, and the full autonomous baseline from ingest to merge.

| Epic | Slice delivered | Notes |
|------|-----------------|-------|
| `epic:hub` | separation slice | Colocated hub + runner↔hub protocol, queue, HTTP API + SSE, merge queue. Remote slice delivered with `milestone:centralized-hub` below. Built from its [plan](./plans/hub-separation.md). |
| `epic:store` | full | The runner's embedded database: leases, epochs, facts-only schema. Built from its [plan](./plans/store.md). |
| `epic:supervisor` | full | The runner reconciliation loop, crash recovery, environment leasing. Built from its [plan](./plans/supervisor.md). |
| `epic:adapters` | full | Harness adapters (spawn / resume / verdict), hooks, selftest, human takeover. Built from its [plan](./plans/adapters.md). |
| `epic:workflow` | full | Hub-defined YAML graphs, nodes and judgements, sticky advancement, artifacts. Built from its [plan](./plans/workflow.md). |
| `epic:review` | full | The default graph's review node. Built from its [plan](./plans/review.md). |
| `epic:delivery` | full | Hub-executed deliver node, merge queue, epoch-fenced submission (#62 series). Built from its [plan](./plans/delivery.md). |
| `epic:ask-answer` | hub slice | Question rows at the hub, ask/answer CLI verbs. Remote slice delivered with `milestone:centralized-hub` below. Built from its [plan](./plans/ask-answer-hub.md). |
| `epic:board` | local slice | Hub-served web app: fleet observability, queue prioritization, chunk grouping. Remote/auth slice delivered with `milestone:centralized-hub` below; reach slice is in the [epic registry](./epics.md). Built from its [plan](./plans/board-local/index.md). |
| `epic:gates` | full | Gate nodes plus runner-config gates by node name; decision parking; CLI + board surfacing. Built from its [plan](./plans/gates.md). |

## `milestone:mvp-feedback` — the lessons of running it for real

The MVP put a real fleet in an operator's hands, and running it taught what the platform still owed them: an operator can now see and bound what the fleet spends, and evolve workflow graphs underneath running work.

| Epic | Slice delivered | Notes |
|------|-----------------|-------|
| `epic:cost` | core | Usage facts, hub flow-through, board surfacing, budget caps + spend kill-switch (#57 series). Model-routing tail delivered separately (outside the milestones, below). Built from its [plan](./plans/cost.md). |
| `epic:migration` | core | Explicit migration, `migration_target` auto-drift, retire/re-enable, graph metadata edits (#101, #120, #124 …). Correctness tail delivered separately (outside the milestones, below); follow-latest slice is in the [epic registry](./epics.md). Built from its [plan](./plans/migration.md). |

## `milestone:centralized-hub` — run the fleet from anywhere

The hub stopped living where the operator sits: one hosted hub off the machine, runners signing in to it from wherever they run, and a board and answer loop safe to reach from any device. Away from the desk no longer means away from the fleet.

| Epic | Slice delivered | Notes |
|------|-----------------|-------|
| `epic:security` | runner-auth slice | Runners sign in with hub-minted per-runner tokens; fleet writes answer to an authenticated router (#85 series). Worker-lockdown slice is in the [epic registry](./epics.md). |
| `epic:board` | remote (auth) slice | Provider login, viewer and operator roles, a capability-aware UI, runner single sign-on (#89). Reach slice is in the [epic registry](./epics.md). Built from the auth half of its [plan](./plans/board-remote/index.md). |
| `epic:ask-answer` | remote slice | The closed answer loop: an answer from any device shows its delivery and the agent back at work, and a beaten answerer is told who won (#165). Built from its [plan](./plans/ask-answer-remote.md). |
| `epic:hub` | remote slice | The hub stands off-machine as a hosted deployment every runner reaches over the network. The plan's multi-runner proof and staged outage drill were retired by operator call: arbitration never assumed a runner count, and the reconnect-and-flush contract is exercised in production by every hub redeploy. Built from its [plan](./plans/hub-remote.md). |

## Outside the milestones

| Epic | Slice delivered | Notes |
|------|-----------------|-------|
| `epic:runner-panel` | core | The runner-served machine panel and transcript viewer: local truth — leases, capacities, escalations with copyable resume commands, transcripts — beside shared fleet views, one codebase with two thin apps. Built from its [plan](./plans/runner-panel/index.md). |
| `epic:migration` | correctness tail | Transition-keyed derivations and fences made migration-aware (#109 series: #108, #110, #111, #112). |
| `epic:cost` | model-routing tail | Named session pools: each graph node resolves to a model and effort tier, sending work to cheaper models where fit allows, and the session stamps what it ran (#193). Named as remaining scope in the core slice's [plan](./plans/cost.md). |
| `epic:migration` | follow-latest slice | Standing drift as policy: in-flight chunks re-pin to the newest enabled mint of their graph at each transition — a hub-level default with a per-graph override (#164). Rode the machinery of the core slice's [plan](./plans/migration.md). |
| `epic:hub` | distribution slice | Public multi-arch hub image with a migrate-then-serve entrypoint, tag-driven releases with generated notes, a written versioning policy, the reference compose deployment, and the operator's book (#196). Remote slice delivered with `milestone:centralized-hub` above. Built from its [plan](./plans/hub-distribution.md). |
| `epic:forge-status` | full | The fleet's hold projected onto the tracker: one-way labels from hub truth onto forge issues through the annotator seam, a per-source opt-in, reconciled by a periodic diff sweep (#197). Built from its [plan](./plans/forge-status.md). |
| `epic:board` | chunk-detail slice | Routed chunk detail page with General/Artifacts tabs; dock artifacts became links that land pre-selected in one shared renderer (#199). Reach slice is in the [epic registry](./epics.md). Built from its [plan](./plans/chunk-detail/index.md). |
