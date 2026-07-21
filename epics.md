# Epics

The epic registry, and the place plan priority is decided: the order of the rows below is the order of the plans behind them. An epic is a capability area, not a version — several land partially (a *slice*) before they land fully. Epics serve the destinations in [milestones.md](./milestones.md); a slice that completes is recorded in [delivered.md](./delivered.md).

## Horizon

Priority-ordered: the top rows are next in line to be promoted to a plan. Nothing here has issues filed yet.

| Epic | Capability | Notes |
|------|------------|-------|
| `epic:security` (worker-lockdown slice) | Worker lockdown (D-087): permission profiles beyond the runner-configured skip-permissions default, network/credential blast radius, force-push protection, per-node permission config. | The runner-auth slice shipped (#85); this slice is unfiled. Live risk while dogfooding with real workers against real GitHub. Plan: [worker-lockdown](./plans/worker-lockdown.md). |
| `epic:ci-feedback` | Routing CI and review results back into the owning session, capped feedback rounds. | The merge-queue machinery it feeds into exists. Plan: [ci-feedback](./plans/ci-feedback.md). |
| `epic:config` | The configuration surface: every operational constant as config, comparable-run tuning for the harness-engineer persona. | Likely a survey-then-sweep effort rather than a large build. |
| `epic:hub` (remote slice) | Off-machine hub, multiple runners — a [`milestone:centralized-hub`](./milestones.md) spoke. | The separation slice (colocated hub, runner↔hub protocol) shipped in the MVP. Plan: [hub-remote](./plans/hub-remote.md). |
| `epic:ask-answer` (remote slice) | Question fan-out and answers from board/chat clients. | The hub slice (question rows, `blizzard runner ask` / `blizzard hub answer`) shipped in the MVP. Plan: [ask-answer-remote](./plans/ask-answer-remote.md). |
| `epic:chat` | Telegram bot (D-031): question notifications with one-tap answers, delivery confirmations. | Depends on `epic:ask-answer`'s remote slice. Plan: [chat](./plans/chat.md). |
| `epic:team` | Team mode: shared visibility and multi-operator policy over one hub-arbitrated fleet — cross-machine arbitration is the hub's queue, never forge-native acquisition. | Waits on the human-auth epic (#89) landing. |
| `epic:batching` | Execution units, the heuristic packer, the LLM planner, conflict packing, batch failure semantics. | **Parked** by design; keep captured, don't schedule. Plan: [batching](./plans/batching.md). |

## In flight

Issues filed and work ongoing.

| Epic | Slice / scope | Issues |
|------|---------------|--------|
| `epic:board` (remote slice) | Human auth, roles, capability-aware session UI, runner SSO — the auth prerequisite of the board's remote slice. Plan: [board-remote](./plans/board-remote/index.md). | #89 (epic), #91–#96 |
| `epic:migration` | Correctness tail: transition-keyed derivations and fences made migration-aware. | #109 (epic), #110 |
