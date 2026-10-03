# Span contract

## Steps

A trace is one step. A step is either one attempt at a node or one human decision. Neither store has a node-step id, and
none is introduced: the hub already names an attempt by `(chunk_id, epoch)` and fences it (`bzh:epoch-fencing`). Three
kinds of step produce a trace:

- **Runner step.** An epoch with `lease_facts` rows whose `epoch_owners` row names a runner. An epoch can hold several
  admitted lease rows, and the step starts at the earliest row's `minted_at`.
- **Hub step.** An epoch with `lease_facts` rows whose `epoch_owners.runner_id` is null. Ownership is read from
  `epoch_owners`, never from the lease row's `runner_id` text, so a runner an operator happens to name `hub` is still a
  runner. The hub mints that lease in the same write as the step's exit, so `minted_at` is its end, not its start. Its
  start is the latest fact that placed the chunk on the hub node before then:
  - a `transitions` row whose `to_node_id` is the node
  - a `chunk_migrations` landing on it
  - a `chunk_restarts` row onto it
  - a later `requeues` row
- **Gate step.** One `decisions` row. It runs from `submitted_at` to the fact that closes the decision (Closing a step).
  A decision has two origins:
  - **Graph gate.** A transition into a human-judged node opens it at the submitting step's epoch.
  - **Runner gate.** A runner-imposed gate, matched by node name, opens it at the worker's exit.

  Either way, the fact that closes it reuses that epoch. Two decisions can share one epoch (a runner gate followed by a
  graph gate), and each is its own gate step.

An epoch that only fences produces no trace. That covers an operator restart's hub-owned epoch with nothing behind it,
and a claim's reserved epoch that never minted a lease. Its effect shows on the next step as
`blizzard.step.preceded_by`.

A runner step's start is when the lease fact landed on the hub, not when the worker spawned. That gap is usually one
runner drain, but a runner that cannot reach the hub stores and forwards, and then the lag can run to hours. The hub's
root keeps this start. The runner-spans slice adds a `worker` span inside the step, carrying the runner's own measured
times, so the gap becomes visible rather than corrected.

## Closing a step

A step is told once it has closed. A runner or hub step closes at the earliest of these:

| Closing fact                                                            | `blizzard.step.outcome` |
| ----------------------------------------------------------------------- | ----------------------- |
| a `transitions` row at its epoch with no `decision_id`                  | `transitioned`          |
| a `decisions` row at its epoch, submitted by a runner gate              | `gated`                 |
| a `chunk_migrations` row at its epoch whose `source` is not `restart`   | `migrated`              |
| an `escalations` row at its epoch                                       | `escalated`             |
| a `route_released` row after its start, with no later fact at its epoch | `released`              |
| `chunk_stopped` after its start                                         | `stopped`               |
| `chunk_completed` after its start                                       | `completed`             |
| an `epoch_owners` row for a higher epoch                                | `superseded`            |

A legacy `chunk_migrations` row with a null `source` counts as not `restart`.

A gate step closes at the first fact carrying its `decision_id`, from the same four tables the hub's own decision store
treats as closing a decision:

| Closing fact             | `blizzard.step.outcome` |
| ------------------------ | ----------------------- |
| a `transitions` row      | `decided`               |
| a `chunk_migrations` row | `migrated`              |
| an `escalations` row     | `escalated`             |
| a `chunk_restarts` row   | `restarted`             |

A chunk stopped, completed or superseded while the decision is open closes it with that outcome instead.

A gate's closing fact is written when the holding runner picks the decision up, which can be long after the person
decided, through store-and-forward. The `gate` span therefore ends at `decision_resolutions.resolved_at`, the moment
they decided. The stretch from there to the closing fact is the `pickup` span.

A step ends when its closing fact was recorded. A transition into the reserved terminal still reads `transitioned`, with
`blizzard.step.to_node.name = "done"`.

## Where a step stood

The graph and node of a step are read from a pure point-in-time fold of the chunk's movement facts — transitions,
migrations and restarts — evaluated at the step's start. The chunk's current pin is a mutable value, so it is never
used.

- A gate step's node is its decision's `node_id`.
- A hub step's node is the node its start placed the chunk on.

`blizzard.step.visit` is 1 plus the number of the chunk's earlier arrivals at a node of the same name, through any
movement fact.

## Identity

Trace and span ids are derived, never random. A replay, a later slice and an inline span made elsewhere then land in the
same trace without anything crossing the wire.

- **Step key.**
  - `chunk_id + "/" + epoch` for runner and hub steps.
  - `chunk_id + "/" + epoch + "/gate/" + decision_id` for gate steps.
- **Trace id.** The first 16 bytes of `SHA-256("blizzard-trace/v1/" + step key)`.
- **Span id.** The first 8 bytes of `SHA-256("blizzard-span/v1/" + step key + "/" + role + "/" + discriminator)`.
  - `role` is a span's role from the table below.
  - `discriminator` is the source row's own id (`question_id`, `slot_id`, a pause fact's id), or empty for a role that
    occurs once per step.
- **Zero ids.** A derived id that is all zeros has its last byte set to `01`.
- **Context flags.** Every derived context carries `TraceFlags.SAMPLED`.

The `v1` prefix is the contract version, and changing the derivation is a breaking change.

Step identification, closing, the position fold and the id derivation live in one pure domain module with no hub storage
dependency, and runner spans import it rather than restate it.

The same module assembles each closed step's summary: every dimension and measure in §Attributes, computed once. That
covers outcome, choice, destination, `preceded_by`, bounce cause, asks, waits by kind, and the token and cost totals,
with billed and estimated cost kept apart until a span folds them. Spans are built from the summary, and the fact-egress
step rows are built from the same one, so a trace and an exported row never disagree about a step.

## Spans in a step's trace

Every span is `SpanKind.INTERNAL`. Children are parented on the root span: `step`, or `gate` for a gate step.

| Role       | Span name          | Present when                                                                            | Start → end                                                                    |
| ---------- | ------------------ | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `step`     | `step <node name>` | always — the root of a runner or hub step                                               | step start → closing fact                                                      |
| `gate`     | `gate <node name>` | always — the root of a gate step                                                        | `submitted_at` → `resolved_at`, or the closing fact when no one resolved it    |
| `queue`    | `queue wait`       | the first step of any kind after a `route_created`                                      | the instant the chunk last became claimable → `route_created.created_at`       |
| `claim`    | `claim`            | the first runner step after a `route_created`                                           | `route_created.created_at` → the step's `minted_at`                            |
| `ask`      | `ask`              | a `questions` row at the step's epoch                                                   | `asked_at` → `question_answers.answered_at`, or the step's end when unanswered |
| `pause`    | `pause`            | a pause set on the chunk while the step was open                                        | the pause fact's `set_at` → the fact that lifted it, or the step's end         |
| `pickup`   | `decision pickup`  | a gate step whose decision was resolved                                                 | `resolved_at` → the closing fact                                               |
| `hub-exec` | `hub exec`         | a hub step: one per `hub_exec_slot` row for its chunk and node within the step's window | `acquired_at` → `released_at`, clamped to the step's end                       |

**Claimable.** The instant the chunk last became claimable is the latest, before the claim, of:

- its promotion
- a route release
- a requeue
- the lifting of a pause
- the completion of its last unmet prerequisite

A paused or blocked chunk is not waiting in the queue.

**Unused claims.** A claim released before any lease attaches no spans. The next step records it as
`blizzard.step.preceded_by = "released-claim"`.

**Outside the parent.** `queue` and `claim` start before the step that owns them, and `pickup` starts as its gate ends.
The root span itself is not stretched to cover them. A chunk that waited a day for a runner reads as a day-long child
beside an hour-long step, rather than as one long trace.

**Clock skew.** `asked_at` comes from the runner's clock and `answered_at` from the hub's. An `ask` whose end would
precede its start is clamped to zero length, with `blizzard.clock_skew = true`.

**Gate content.** A gate step's only child is its `pickup`. Asks and pauses belong to the step the worker was in.

**Span events on a step root.** Events are attached by epoch for runner steps, and by chunk, node and time window for
hub steps. A hub node's polls are recorded at the incoming epoch, not the hub step's own. Any event timestamped after
the step's end is clamped to it.

- **`invocation`.** One per `usage_facts` row at the epoch, timestamped `recorded_at`. Attributes:
  - `blizzard.invocation.kind`: `spawn`, `resume` or `judge`. The hub's usage records a nudge as `resume`, so this slice
    cannot tell the two apart; runner-spans adds `blizzard.invocation.nudge` where it can.
  - `blizzard.harness.id` and `blizzard.harness.version`
  - `gen_ai.response.model` and the GenAI token counts, mapped as [GenAI usage](#genai-usage) states
  - `blizzard.invocation.input_tokens`, `.output_tokens`, `.cache_read_tokens` and `.cache_create_tokens`
  - `blizzard.invocation.cost.usd` and `blizzard.invocation.cost.estimated`
- **`hub poll pending`.** One per `hub_node_poll` row in the step's window.
- **`bounce`.** The `chunk_bounces` row at the step's epoch, at most one. It carries `blizzard.bounce.cause`.

## Links

A step's root links to the root of the chunk's previous step: the nearest earlier step that produced a trace. Its
context is derived, so nothing is looked up. The link carries `blizzard.link.reason`, the first of these that applies:

| Reason      | When                                                                        |
| ----------- | --------------------------------------------------------------------------- |
| `restart`   | an operator restart sits between them                                       |
| `migration` | the steps ran under different graphs                                        |
| `bounce`    | the previous step bounced                                                   |
| `retry`     | the previous step closed without moving the chunk; the same node is retried |
| `next`      | otherwise                                                                   |

A chunk's first step has no link.

## Status

A step root is `ERROR` when its outcome is `escalated`, and `UNSET` otherwise. A choice that sends work backward,
including a hub failure that bounces, is routing rather than error. Children are always `UNSET`.

## Attributes

Every span carries the dimensions below, so any single span answers a query on its own. Names are resolved from ids when
the step is told.

Measures are different. A count or a cost copied onto every span would be counted once per span by any backend that sums
over spans, and the step's spend would read several times over. So the measures — tokens, cost and waits — sit only on
the step's root span, under `blizzard.step.*` names. Each invocation carries its own share under `blizzard.invocation.*`
names, on the invocation event here and on the invocation span runner-spans adds. Summing `blizzard.step.cost.usd` over
root spans, or `blizzard.invocation.cost.usd` over invocations, gives the true total.

**Dimensions**, on every span:

| Attribute                     | Type     | Meaning                                                                                 |
| ----------------------------- | -------- | --------------------------------------------------------------------------------------- |
| `blizzard.chunk.id`           | string   | the chunk                                                                               |
| `blizzard.chunk.work_refs`    | string[] | its work items as source-native tokens, e.g. `blizzard#745`                             |
| `blizzard.graph.name` / `.id` | string   | where the step stood                                                                    |
| `blizzard.node.name` / `.id`  | string   | the node                                                                                |
| `blizzard.node.executor`      | string   | `runner`, `hub`, or `human` for a gate step                                             |
| `blizzard.step.epoch`         | int      | the epoch                                                                               |
| `blizzard.step.visit`         | int      | which arrival at this node this is, counting from 1                                     |
| `blizzard.step.outcome`       | string   | see Closing a step                                                                      |
| `blizzard.step.choice`        | string   | the resolved choice, when there is one                                                  |
| `blizzard.step.to_node.name`  | string   | where it led — a node name, `done`, or `graph:<name>`                                   |
| `blizzard.step.preceded_by`   | string   | `restart`, `requeue`, or `released-claim`, when one sits between this step and the last |
| `blizzard.runner.id`          | string   | the runner holding the chunk                                                            |
| `blizzard.harness.id`         | string   | the harness of the step's last invocation                                               |
| `blizzard.step.models`        | string[] | distinct models across its invocations                                                  |
| `blizzard.bounce.cause`       | string   | `conflict`, `checks`, `master-moved`, `poll-timeout`, or a choice                       |
| `blizzard.ask.answered`       | bool     | on `ask`                                                                                |

**Measures**, on the step root only:

| Attribute                                                                                    | Type   | Meaning                                                                               |
| -------------------------------------------------------------------------------------------- | ------ | ------------------------------------------------------------------------------------- |
| `blizzard.step.input_tokens`, `.output_tokens`, `.cache_read_tokens`, `.cache_create_tokens` | int    | summed across its invocations                                                         |
| `blizzard.step.cost.usd`                                                                     | double | summed cost, folded as `docs/deployment/spend.md` defines                             |
| `blizzard.step.cost.estimated`                                                               | bool   | any summand was an estimate                                                           |
| `blizzard.step.cost.partial`                                                                 | bool   | any invocation carried neither a billed cost nor an estimate                          |
| `blizzard.step.wait.queue_ms`, `.claim_ms`, `.ask_ms`, `.pause_ms`, `.pickup_ms`             | int    | the step's waits, summed by kind, so a backend that cannot join spans still sees them |

Resource attributes:

- **`service.name`.** `blizzard-hub`, unless `OTEL_SERVICE_NAME` or `OTEL_RESOURCE_ATTRIBUTES` names one. The default is
  applied only when neither does.
- **`service.version`.** The hub's build version.
- **`blizzard.trace.schema_version`.** `"1"`.
- **Instrumentation scope.** `blizzard.hub.fleet_spans`, at the contract version.

## GenAI usage

Where the OpenTelemetry GenAI semantic conventions define an attribute — the model, and the token counts — invocation
events use it, pinned to a named semconv version. The GenAI conventions are still marked as in development, so the
implementation checks every name and meaning below against the pinned version before it ships.

The conventions and blizzard count input differently. Blizzard's input count excludes cached tokens, as Anthropic's
usage does. The conventions count cached input inside `gen_ai.usage.input_tokens` and break the cached part out in their
own cache-read and cache-creation attributes. Copying blizzard's count across would make a backend misprice every cached
run, so the mapping is explicit:

- `gen_ai.usage.input_tokens` is blizzard's input plus cache-read plus cache-creation tokens.
- The conventions' cache-read and cache-creation input attributes carry blizzard's two cache counts.
- `gen_ai.usage.output_tokens` is blizzard's output count.
- `blizzard.invocation.input_tokens` keeps blizzard's own uncached count, so `docs/deployment/spend.md` folds still
  hold.

Backends build their LLM views from spans carrying `gen_ai.operation.name`, not from events, so this slice does not
light those views up. Invocation spans arrive with runner-spans and use the same mapping.

## What never leaves

No span carries:

- prompt text, transcript content, or check output
- an ask's question or answer, or a gate's choice descriptions
- a bounce's envelope
- the name or login of anyone who resolved a gate or answered an ask
- a chunk's title or body

Work refs, names, ids, counts, durations and costs are the whole of what leaves.
