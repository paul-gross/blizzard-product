# Runner span contract

## The unit: one lease

A runner lease is one runner step: one epoch, minted by this runner (`leases.epoch`). The runner tells each of its
leases into the trace of that step, using the fleet-spans [identity](../../fleet-spans/spec/spans.md#identity):

- **Trace and step key.** The trace id is derived from `(chunk_id, leases.epoch)`.
- **Parent.** The `worker` span's parent is the hub's derived `step` root span id.
- **Runner span ids.** Each runner span id uses the same derivation, with the role prefixed `runner/`. For example, the
  `worker` span is role `runner/worker` with discriminator `lease_id`.

No message passes between the daemons for this. If the hub's root is missing, because the hub has fleet tracing off or a
crash lost it, the runner's spans still group by trace id under a parent that never arrives.

Hub steps and gate steps get nothing from the runner.

## Spans

Every span is `SpanKind.INTERNAL`.

| Role                | Span name                     | Parent          | Built from                                                                | Start → end                                                                                               |
| ------------------- | ----------------------------- | --------------- | ------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `runner/worker`     | `worker <node name>`          | hub `step` root | `leases`, `lease_context`, `lease_closures`                               | `leases.created_at` → the lease's close (emission.md §When a lease is told)                               |
| `runner/invocation` | `invoke_agent <session name>` | `worker`        | one `invocation_boundaries` row, discriminated by `generation` and `kind` | `opened_at` → its end (below)                                                                             |
| `runner/ask-park`   | `parked on ask`               | `worker`        | `park_facts` → `park_resumes` by `question_id`                            | `parked_at` → `resumed_at`, or the lease's close                                                          |
| `runner/pause-park` | `parked on pause`             | `worker`        | `pause_parks` → the next `pause_park_resumes` on the lease                | `parked_at` → `resumed_at`, or the lease's close                                                          |
| `runner/overload`   | `provider overload backoff`   | `worker`        | `overload_facts`                                                          | `observed_at` → the earlier of `resume_after` and the next invocation's `opened_at`, or the lease's close |
| `runner/takeover`   | `takeover`                    | `worker`        | `takeovers` → `takeover_ends`                                             | `opened_at` → `ended_at`, or the lease's close                                                            |

**Where an invocation ends.** An invocation boundary's own `closed_at` is set only when its lease closes, so it is not
the invocation's end. An invocation ends at the earliest of these:

1. the first `session_ends.ended_at` on the lease after its `opened_at`
2. the next boundary's `opened_at` on the same lease
3. its `closed_at`

`blizzard.invocation.end_source` records which one applied: `session_end`, `next_invocation` or `lease_close`.

**Moving starts.** A boundary's `opened_at` can be advanced after the fact. The sweep reads a lease only after the
settle window, and a replay reads the record as it stands when it runs, so a replayed invocation keeps its ids but may
start later than the first telling did.

**Joining the record.** Three tables describe one invocation, and only one of them names it:

- **Usage.** A worker and its judge share a generation, and a worker's usage `kind` does not reliably say whether a
  spawn, resume or nudge opened it. Usage therefore matches by generation and by side: a `usage_facts` row of kind
  `judge` belongs to the generation's `judge` boundary, and any other kind to the generation's worker boundary,
  whichever of `spawn`, `resume` or `nudge` opened it.
- **Spawns.** `lease_spawns` has no generation column. Its rows on a lease, ordered by id, are the worker spawns in
  order, so the n-th row is generation n. The session id, harness and identification time of a worker invocation come
  from that row.
- **Judges.** A judgement spawns no `lease_spawns` row. A judge invocation takes its harness from its own `usage_facts`
  rows, and carries no `session identified` event and no `gen_ai.conversation.id`.

**Clamping.** A child that would end after its parent is clamped to the parent's end.

**Clocks.** Runner spans are not clamped to the hub's `step` root. The two daemons stamp with different clocks, and the
runner's lease is minted before the hub hears of it, so a `worker` span may start before its parent. That gap is the
start bias the fleet-spans contract documents, now visible rather than hidden. The hub's root keeps its own times.

## Span events

| Event                | On         | Built from                                                             | Attributes                                                     |
| -------------------- | ---------- | ---------------------------------------------------------------------- | -------------------------------------------------------------- |
| `session identified` | invocation | `lease_spawns.identified_at`                                           | —                                                              |
| `session end`        | invocation | `session_ends.ended_at`                                                | —                                                              |
| `context sample`     | invocation | `context_samples` within its window                                    | `blizzard.context.tokens`                                      |
| `nudge`              | worker     | `nudge_facts.nudged_at`                                                | —                                                              |
| `check`              | worker     | each `check_results` row on the lease, timestamped `checks_ran.ran_at` | `blizzard.check.index`, `blizzard.check.passed`                |
| `checks ran`         | worker     | `checks_ran.ran_at`                                                    | `blizzard.checks.passed` (all passed), `blizzard.checks.count` |

**Checks.** Every declared check runs, none short-circuits, and the results are written together in one batch once the
last has finished. Each row therefore carries the same `ran_at`, and the record holds no time for any single check.
`blizzard.check.index` is a check's rank by row id among the lease's results at that epoch, counting from 1, which is
its position in the node's declared checks. Checks get events, not spans, because the record has no per-check duration.

## Attributes

Every runner span carries the fleet-spans dimension set for its step:

- `blizzard.chunk.id` and `blizzard.chunk.work_refs`
- `blizzard.graph.name`, `blizzard.graph.id`, `blizzard.node.id` and `blizzard.node.name`
- `blizzard.node.executor = "runner"`
- `blizzard.step.epoch`
- `blizzard.runner.id`

These come from `lease_context`. It holds the graph and node ids and the node name today. This slice adds the graph name
and the work refs to it, recorded from the node envelope when the lease is minted, so a runner span carries the same
names a hub span does. The envelope already carries `work_refs`, but not the graph's name. The hub's envelope builder
adds a `graph_name` field, which is an additive change to the envelope wire contract.

`blizzard.step.visit` is the one dimension left off. Only the hub's history counts arrivals at a node, and a backend
joins it from the step root.

Measures follow the fleet-spans rule: never on every span. Only invocation spans carry tokens and cost, under the
`blizzard.invocation.*` names.

| Attribute                                                                                                                          | On                 | Meaning                                                                                  |
| ---------------------------------------------------------------------------------------------------------------------------------- | ------------------ | ---------------------------------------------------------------------------------------- |
| `blizzard.lease.id`                                                                                                                | all                | the lease                                                                                |
| `blizzard.lease.close_reason`                                                                                                      | worker             | `lease_closures.reason` (emission.md)                                                    |
| `blizzard.session.name`                                                                                                            | worker, invocation | the declared session the node ran in (`lease_context.session_name`)                      |
| `blizzard.harness.id`, `.version`                                                                                                  | worker, invocation | from `lease_spawns`, or a judge's `usage_facts` (above)                                  |
| `blizzard.model.resolved`, `blizzard.effort.resolved`                                                                              | worker             | what the runner resolved the session's tier and effort to (`lease_context`)              |
| `blizzard.invocation.kind`                                                                                                         | invocation         | `spawn`, `resume` or `judge`, the same values fleet-spans uses; a nudge reads `resume`   |
| `blizzard.invocation.nudge`                                                                                                        | invocation         | `true` when a nudge opened it, which only the runner can tell                            |
| `blizzard.invocation.generation`                                                                                                   | invocation         | the boundary's generation                                                                |
| `blizzard.invocation.end_source`                                                                                                   | invocation         | see above                                                                                |
| `blizzard.overload.streak`                                                                                                         | overload           | `streak_ordinal`                                                                         |
| `blizzard.invocation.input_tokens`, `.output_tokens`, `.cache_read_tokens`, `.cache_create_tokens`, `.cost.usd`, `.cost.estimated` | invocation         | as fleet-spans defines for its invocation events, summed over the matching `usage_facts` |

## GenAI conventions

Invocation spans follow the OpenTelemetry GenAI conventions for an agent invocation, pinned to the same semconv version
as fleet-spans, so a backend's model views include them. The conventions are still marked as in development, so each
name below is checked against that version before it ships:

- **`gen_ai.operation.name`.** `invoke_agent`.
- **`gen_ai.agent.name`.** The session name, which also names the span: `invoke_agent <session name>`.
- **`gen_ai.conversation.id`.** The harness session id from `lease_spawns`, for worker invocations.
- **`gen_ai.request.model`.** The resolved model.
- **`gen_ai.response.model`.** The model the usage reports.
- **Token counts.** Summed from the matching `usage_facts` and mapped exactly as fleet-spans
  [GenAI usage](../../fleet-spans/spec/spans.md#genai-usage) states, with cached input counted inside
  `gen_ai.usage.input_tokens`.
- **`gen_ai.provider.name`.** Set only where the harness binding states a provider without inference. Otherwise it is
  omitted rather than guessed, since an OpenCode model can sit behind any provider.

An invocation may span several model calls, so these are invocation totals, not per-call figures. No `chat` spans are
emitted, because the runner does not see individual model calls.

## What never leaves

No runner span carries:

- an ask's question text or options
- a check's command line or output (`output_tail`)
- an attachment, or any transcript content
- a working directory, environment path, process id, or `pgid`
- a git branch or commit declaration
- a worker's stdout

Ids, names, counts, durations, outcomes and costs are the whole of what leaves.
