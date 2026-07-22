# Epics

The registry of blizzard's live epics: the capability areas the product is built from, in priority order — the order of the rows is the order of the plans behind them. An epic is a capability area, not a version, and it may land in slices. Each epic serves one or more destinations in [milestones.md](./milestones.md); this is a reference list of what is being built and why, not a status board — where an epic's slices actually stand is tracked in the milestone epic charts and in GitHub, and a slice that has fully landed resolves in [delivered.md](./delivered.md).

| Epic | Capability | Notes |
|------|------------|-------|
| `epic:migration` | Correctness tail: transition-keyed derivations and fences made migration-aware. | Plan: [migration](./plans/migration.md). |
| `epic:security` (worker-lockdown slice) | Worker lockdown (D-087): permission profiles beyond the runner-configured skip-permissions default, network/credential blast radius, force-push protection, per-node permission config. | Live risk while dogfooding with real workers against real GitHub. Plan: [worker-lockdown](./plans/worker-lockdown.md). |
| `epic:ci-feedback` | Routing CI and review results back into the owning session, capped feedback rounds. | Feeds the existing merge-queue machinery. Plan: [ci-feedback](./plans/ci-feedback.md). |
| `epic:config` | The configuration surface: every operational constant as config, comparable-run tuning for the harness-engineer persona. | A survey-then-sweep effort rather than a large build. |
| `epic:hub` (remote slice) | Off-machine hub, multiple runners — a [`milestone:centralized-hub`](./milestones.md) spoke. | Plan: [hub-remote](./plans/hub-remote.md). |
| `epic:ask-answer` (remote slice) | Question fan-out and answers from board/chat clients. | Plan: [ask-answer-remote](./plans/ask-answer-remote.md). |
| `epic:chat` | Telegram bot (D-031): question notifications with one-tap answers, delivery confirmations. | Depends on `epic:ask-answer`'s remote slice. Plan: [chat](./plans/chat.md). |
| `epic:board` (reach slice) | Phone-friendly progressive-web-app shell — the board installed on a home screen, laid out for a thumb — and question notifications surfacing through it. | Depends on `epic:ask-answer`'s remote slice for answer fan-out. Plan: [board-remote](./plans/board-remote/index.md). |
| `epic:team` | Team mode: shared visibility and multi-operator policy over one hub-arbitrated fleet — cross-machine arbitration is the hub's queue, never forge-native acquisition. | Builds on the human-auth work (`epic:board`'s remote slice). |
| `epic:batching` | Execution units, the heuristic packer, the LLM planner, conflict packing, batch failure semantics. | Parked by design — captured, not scheduled. Plan: [batching](./plans/batching.md). |
