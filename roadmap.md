# Roadmap

The living epic registry — the continuation of `blizzard-discovery:/product/epics.md`, which is frozen history. Epic ids (`epic:<slug>`) are stable and carry forward unchanged from that registry, so citations in the discovery corpus, in code, and in issues keep resolving; this file is now the id's single home and the place a citation resolves for **current** status.

An epic is a capability area, not a version: several land partially (a *slice*) before they land fully, and a row's status names the slice it means. Rows move between the sections below as slices promote and ship — see [CONTRIBUTING.md](./CONTRIBUTING.md) for the promotion workflow.

## Horizon

Priority-ordered: the top rows are next in line to be promoted to a requirements doc. Nothing here has issues filed yet.

| Epic | Capability | Notes |
|------|------------|-------|
| `epic:security` (worker-lockdown slice) | Worker lockdown (D-087): permission profiles beyond the runner-configured skip-permissions default, network/credential blast radius, force-push protection, per-node permission config. | The runner-auth slice shipped (#85); this slice is unfiled. Live risk while dogfooding with real workers against real GitHub. |
| `epic:ci-feedback` | Routing CI and review results back into the owning session, capped feedback rounds. | Staged sketch in `blizzard-discovery:/research/hard-problems.md`. The merge-queue machinery it feeds into exists. |
| `epic:config` | The configuration surface: every operational constant as config, comparable-run tuning for the harness-engineer persona. | Likely a survey-then-sweep effort rather than a large build. |
| `epic:hub` (remote slice) | Off-machine hub, multiple runners — the `milestone:centralized-hub` spoke. | The separation slice (colocated hub, runner↔hub protocol) shipped in the MVP. |
| `epic:ask-answer` (remote slice) | Question fan-out and answers from board/chat clients. | The hub slice (question rows, `blizzard runner ask` / `blizzard hub answer`) shipped in the MVP. |
| `epic:chat` | Telegram bot (D-031): question notifications with one-tap answers, delivery confirmations. | Depends on `epic:ask-answer`'s remote slice. |
| `epic:team` | Team mode: shared visibility and multi-operator policy over one hub-arbitrated fleet — cross-machine arbitration is the hub's queue, never forge-native acquisition. | Waits on the human-auth epic (#89) landing. |
| `epic:batching` | Execution units, the heuristic packer, the LLM planner, conflict packing, batch failure semantics. | **Parked** by design (`blizzard-discovery:/design/planner.md`); keep captured, don't schedule. |

## In flight

Issues filed and work ongoing.

| Epic | Slice / scope | Issues |
|------|---------------|--------|
| `epic:board` (remote slice) | Human auth, roles, capability-aware session UI, runner SSO — the auth prerequisite of the board's remote slice. | [#89](https://github.com/paul-gross/blizzard/issues/89) (epic), #91–#96 |
| `epic:migration` | Correctness tail: transition-keyed derivations and fences made migration-aware. | [#109](https://github.com/paul-gross/blizzard/issues/109) (epic), #110 |

## Shipped

The capability landed; the closed issues and the discovery corpus are the record. Ids remain citable.

| Epic | Slice shipped | Notes |
|------|---------------|-------|
| `epic:hub` | separation slice (MVP) | Colocated hub + runner↔hub protocol, queue, HTTP API + SSE, merge queue. Remote slice is on the horizon above. |
| `epic:store` | full | The runner's embedded database: leases, epochs, facts-only schema. |
| `epic:supervisor` | full | The runner reconciliation loop, crash recovery, environment leasing. |
| `epic:adapters` | full | Harness adapters (spawn / resume / verdict), hooks, selftest, human takeover. |
| `epic:workflow` | full | Hub-defined YAML graphs, nodes and judgements, sticky advancement, artifacts. |
| `epic:review` | full | The default graph's review node. |
| `epic:delivery` | full | Hub-executed deliver node, merge queue, epoch-fenced submission (#62 series). |
| `epic:ask-answer` | hub slice (MVP) | Question rows at the hub, ask/answer CLI verbs. Remote slice is on the horizon above. |
| `epic:board` | local slice (MVP) | Hub-served web app: fleet observability, queue prioritization, chunk grouping. Remote/auth slice is in flight above. |
| `epic:gates` | full | Gate nodes plus runner-config gates by node name; decision parking; CLI + board surfacing. |
| `epic:cost` | core | Usage facts, hub flow-through, board surfacing, budget caps + spend kill-switch (#57 series). Remaining unfiled sub-bullet: model routing by cost. |
| `epic:migration` | core | Explicit migration, `migration_target` auto-drift, retire/re-enable, graph metadata edits (#101, #120, #124 …). Correctness tail is in flight above. |
| `epic:security` | runner-auth slice | Hub-minted per-runner tokens, partitioned fleet router, route capability tokens, spawn-env allowlist (#85 series). Worker-lockdown slice is on the horizon above. |
