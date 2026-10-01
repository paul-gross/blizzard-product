# Runner emission contract

The runner tells its leases the way the hub tells its steps. The fleet-spans
[emission contract](../../fleet-spans/spec/emission.md) owns the shared discipline: settle window, at-least-once
delivery, lag cap, replay, the `http/protobuf` exporter built from fact data, and the published shape. This file states
only where the runner differs.

## Where it runs

The sweep runs on its own thread in `runner host`, beside the loop's periodic driver, never as a step of the tick. The
tick is crash-swept coordination code, and an exporter is not coordination.

- **Assembly is pure.** One function turns one closed lease's facts into its span records (`bzh:domain-core`).
- **Reads go through a repository.** The runner's facts are read through a read repository. The invocation-boundary and
  context-sample tables are runner-plane data, and reading them here stays in the runner plane
  (`bzh:runner-plane-transcript-reads`).
- **Export goes through the seam.** Export uses the same `ITraceExporter` seam the hub uses.

The one-shot `runner tick` path does not emit.

## When a lease is told

A lease is ready to tell once its `lease_closures` row exists and the settle window has passed. Every lease gets one: a
chunk the hub stops or reassigns is found by the runner's pull, which abandons the attempt and closes its lease as
`released`. No separate record of the hub's view is needed.

`blizzard.lease.close_reason` is the closure's reason, verbatim:

- `transitioned`
- `reaped`
- `failed`
- `escalated`
- `parked`, for a runner-imposed gate
- `released`
- `preempted`, for an operator restart

The two store-only escalation-mint reasons, `owner-unresolvable-mint` and `no-acceptable-harness-mint`, read
`escalated`.

A runner that never reconnects to the hub never pulls, so its leases on stopped chunks stay open. The maximum-lag cap
bounds that, as it bounds every other backlog.

A lease the runner never minted to completion produces nothing: a claim that never spawned, or a lease whose spawn
failed before `lease_spawns` recorded identity.

The settle window covers usage and context samples written after the lease closes: a late invocation envelope, or the
shutdown drain's final usage.

## The cursor

The cursor is ordered by:

1. the closing fact's recorded time
2. then `lease_id`

It is stored as an append-only fact row in the runner's store, one per advancing sweep, the same shape as the hub's.

On first enable it starts at the current time, and re-enabling after a cursor older than the maximum lag jumps it the
same way the hub's does. Skipped windows go to the runner's own event log, under the same kinds the hub uses:
`trace-export-failed`, `trace-export-recovered`, `trace-window-skipped` and `trace-config-rejected`. They are registered
in the runner's event-kind vocabulary alongside the hub's, through `domain/operations.md` §Event kinds.

Runner retention prunes heartbeats, external usage samples, the outbound buffer and worker stdout. It prunes none of the
tables this slice reads, so retention and the cursor need no coupling. A future retention lane over a table read here
must hold rows newer than the cursor while tracing is on.

## Configuration

Runner spans are enabled by the same rule as hub fleet spans: a standard OTLP endpoint is set, `OTEL_TRACES_EXPORTER` is
not `none`, and `OTEL_SDK_DISABLED` is not `true`. Setting an endpoint on a runner therefore turns on its sweep and
nothing else. Platform spans keep their own `platform` switch.

`service.name` defaults to `blizzard-runner`.

The knobs go under `[tracing]` in `blizzard-runner.toml`, with the same names and defaults as the hub's:

- `sweep_seconds`
- `settle_seconds`
- `batch_limit`
- `max_lag_seconds`
- `replay_max_window`

## Operator surface

Both commands are runner API routes, reached by local operator auth, with CLI verbs that are pure clients:

- **`blizzard runner traces status`** reports the same fields as the hub's.
- **`blizzard runner traces replay --since <t> --until <t> [--dry-run]`** tells every lease that closed in the window,
  with the same ids, and never moves the cursor.

## The published shape

Runner span names, attributes and roles join the fleet-spans contract under `contracts/traces/` and share its
`blizzard.trace.schema_version`. So does the runner's instrumentation scope, `blizzard.runner.runner_spans`. A seeded
runner scenario gets its own golden output.

`docs/deployment/tracing.md` gains:

- the runner's section
- how runner and hub spans meet by id
- what the clock-skew gap between them means

## Verification

- **Unit.** Assembly over runner fact fixtures for:
  - a spawn that transitions
  - a spawn, a park on an ask, and a resume after the answer
  - a nudge after a quiet worker
  - a judgement, whose usage matches its `judge` boundary and not the worker's in the same generation
  - an overload backoff followed by a resume
  - a pause park
  - a takeover
  - leases closed with each `lease_closures` reason, and the two mint reasons read as `escalated`
  - a chunk the hub stopped, whose lease the runner closes as `released`
  - each of the three end sources
  - usage matched by generation and side, including the shutdown drain's late usage
  - spawn rows joined to generations by order, across a resume and a nudge
  - checks that all pass, and checks with one failing, indexed by row id
  - a boundary whose `opened_at` advanced between a first telling and a replay

  The derived ids are pinned by vectors shared with fleet-spans, so a runner span's parent equals the hub's root id for
  the same step.
- **Service.** These are tested against the in-memory exporter:
  - the cursor, settle window, lag cap and replay behave as the hub's do
  - the sweep thread never blocks a tick, shown with a hung exporter under a running loop
  - a kill between export and cursor write re-sends identical ids
- **End to end.** These scenarios run against `blizzard-mock` with a collector, for both the mock Claude Code and the
  mock OpenCode harness:
  - acceptance
  - ask and answer
  - review cycle
  - escalation
  - mixed harness

  Every runner span's trace id and parent match a step root the hub emitted for the same run.
- **Proof.** In a backend's model view, invocations from both harnesses appear side by side for the same node, with
  model, tokens and cost.
