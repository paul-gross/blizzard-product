# Row contract

## Steps come from tracing's definition

A step is exactly what the fleet-spans [span contract](../../../../delivered/tracing/fleet-spans/spec/spans.md) says it
is: a runner step, a hub step or a gate step. When it closes, how it closed, and where the chunk stood at its start all
follow that contract's §Steps, §Closing a step and §Where a step stood. Every column below that is not an identity or a
time is taken from the step summary that contract's §Identity describes: the same pure domain module, assembling the
same outcome, waits, asks, bounce cause and totals that a step's root span reports. This slice computes none of them a
second time. A change to what counts as a step, or to how a measure is summed, changes both exports together.

So this slice depends on that module landing first. It does not depend on tracing being enabled: the module is pure, and
an operator can export steps with no trace endpoint set.

## Datasets

### `steps`

One row per closed step.

| Column                                                                             | Type     | Null | Meaning                                                                                    |
| ---------------------------------------------------------------------------------- | -------- | ---- | ------------------------------------------------------------------------------------------ |
| `step_key`                                                                         | string   | no   | The row's identity: `chunk_id/epoch`, or `chunk_id/epoch/gate/decision_id` for a gate step |
| `trace_id`                                                                         | string   | no   | The step's derived trace id, as 32 hex characters, so a row joins to its trace             |
| `step_kind`                                                                        | string   | no   | `runner`, `hub` or `gate`                                                                  |
| `chunk_id`                                                                         | string   | no   | The chunk                                                                                  |
| `work_refs`                                                                        | string[] | no   | Its work items as source-native tokens, e.g. `blizzard#745`; empty when it has none        |
| `sources`                                                                          | string[] | no   | The distinct work sources of those items                                                   |
| `graph_id`, `graph_name`                                                           | string   | no   | Where the step stood                                                                       |
| `node_id`, `node_name`                                                             | string   | no   | The node                                                                                   |
| `epoch`                                                                            | int      | no   | The epoch                                                                                  |
| `decision_id`                                                                      | string   | yes  | A gate step's decision; null otherwise                                                     |
| `visit`                                                                            | int      | no   | Which arrival at this node this is, counting from 1                                        |
| `runner_id`                                                                        | string   | yes  | The runner that held a runner step; null for hub and gate steps                            |
| `harness_id`                                                                       | string   | yes  | The harness of the step's last invocation                                                  |
| `models`                                                                           | string[] | no   | Distinct models across its invocations                                                     |
| `started_at`, `ended_at`                                                           | time     | no   | As the span contract defines them; a gate ends at `resolved_at` when resolved              |
| `closed_at`                                                                        | time     | no   | When the closing fact was recorded; equal to `ended_at` except for a resolved gate         |
| `duration_ms`                                                                      | int      | no   | `ended_at − started_at`                                                                    |
| `outcome`                                                                          | string   | no   | The span contract's `blizzard.step.outcome`                                                |
| `choice`                                                                           | string   | yes  | The resolved choice, when there is one                                                     |
| `to_node_name`                                                                     | string   | yes  | Where it led — a node name, `done`, or `graph:<name>`                                      |
| `preceded_by`                                                                      | string   | yes  | `restart`, `requeue` or `released-claim`                                                   |
| `bounce_cause`                                                                     | string   | yes  | `conflict`, `checks`, `master-moved`, `poll-timeout`, or a choice                          |
| `asks`, `asks_unanswered`                                                          | int      | no   | Asks raised in the step, and how many were never answered                                  |
| `wait_queue_ms`, `wait_claim_ms`, `wait_ask_ms`, `wait_pause_ms`, `wait_pickup_ms` | int      | no   | The step's waits, summed by kind, as the span contract defines them                        |
| `invocations`                                                                      | int      | no   | How many `invocations` rows belong to the step                                             |
| `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_create_tokens`        | int      | no   | Summed across its invocations, with blizzard's uncached input count                        |
| `cost_billed_usd`                                                                  | decimal  | yes  | The harness-billed cost, summed; null when no invocation carried one                       |
| `cost_estimated_usd`                                                               | decimal  | yes  | The estimated cost, summed; null when no invocation carried one                            |
| `cost_partial`                                                                     | bool     | no   | Some invocation carried neither a billed nor an estimated cost                             |
| `billed_partial`                                                                   | bool     | no   | Some invocation carried no billed cost                                                     |
| `exported_at`                                                                      | time     | no   | When this copy of the row was written                                                      |

Billed and estimated cost stay in separate columns and are never merged, as `docs/deployment/spend.md` folds them. Their
sum is what the fleet-spans step root reports as `blizzard.step.cost.usd`.

### `invocations`

One row per hub `usage_facts` row.

| Column                                                                      | Type    | Null | Meaning                                              |
| --------------------------------------------------------------------------- | ------- | ---- | ---------------------------------------------------- |
| `usage_id`                                                                  | int     | no   | The row's identity: the `usage_facts` id             |
| `step_key`, `trace_id`                                                      | string  | no   | The runner step it belongs to, by chunk and epoch    |
| `chunk_id`, `epoch`                                                         | —       | no   | As on `steps`                                        |
| `graph_id`, `graph_name`, `node_id`, `node_name`                            | string  | no   | Where its step stood, from the same position fold    |
| `runner_id`                                                                 | string  | no   | The runner that reported it                          |
| `kind`                                                                      | string  | no   | `spawn`, `resume` or `judge`; a nudge reads `resume` |
| `model`, `harness_id`, `harness_version`                                    | string  | yes  | As reported                                          |
| `input_tokens`, `output_tokens`, `cache_read_tokens`, `cache_create_tokens` | int     | no   | As reported                                          |
| `cost_billed_usd`, `cost_estimated_usd`                                     | decimal | yes  | As reported, each null when absent                   |
| `recorded_at`                                                               | time    | no   | When the hub received it                             |
| `exported_at`                                                               | time    | no   | When this copy of the row was written                |

Usage is only ever reported by a runner holding the epoch, so every invocation belongs to a runner step, and an epoch
that only fences never carries one. The step may still be open when its invocation is exported. Its `steps` row follows
once it closes, under the `step_key` the invocation already carries.

An invocation row is exported on its own cursor, not with its step, so usage that lands after its step was exported
still leaves. That late usage also writes its step again with corrected totals (export.md §Late usage), so the two
datasets agree once both have caught up. The `invocations` dataset stays the finest grain of spend: it is the only one
that splits a step's cost by model.

## Shared conventions

- **Names beside ids.** Every graph and node appears by id and by name. Names are resolved when the row is written, from
  the graph the step was pinned to, never from the chunk's current pin.
- **Times.** UTC, microsecond precision. Parquet stores them as `timestamp[us, UTC]`; NDJSON as RFC 3339 strings with a
  `Z` suffix.
- **Money.** Parquet stores it as `decimal(18, 9)`; NDJSON as a string, so no reader rounds it through a float.
- **Lists.** Parquet stores them as `list<string>`; NDJSON as JSON arrays.
- **Duplicates.** A row may be written more than once: after a crash between a file and its cursor, or by a backfill.
  Copies of one identity are identical or newer, so a reader keeps the copy with the latest `exported_at`. The data
  dictionary ships that view.

## What never leaves

No row carries:

- prompt text, transcript content, or check output
- an ask's question or answer, or a gate's choice descriptions
- a bounce's envelope
- the name or login of anyone who resolved a gate or answered an ask
- a chunk's or work item's title or body
- an escalation's takeover command

Ids, names, counts, times and costs are the whole of what leaves. A test plants each of these values and scans every
exported file for it.
