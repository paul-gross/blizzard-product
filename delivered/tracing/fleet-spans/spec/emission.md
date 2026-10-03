# Emission contract

## Where it runs

Emission is a hub sweep, started from the hub's lifespan alongside its existing reconcilers, and only when fleet tracing
is enabled. It follows the system shape the other sweeps already honor:

- **Assembly is pure.** A domain function turns one closed step's facts into its step summary (spans.md §Identity), and
  the summary into its span records — ids, times, attributes, events, links — with no I/O. Every shape in
  [spans.md](./spans.md) is therefore a unit test over fact fixtures (`bzh:domain-core`).
- **Reads go through a repository.** The sweep reads closed steps after the cursor through a read repository.
- **Export goes through a seam.** An `ITraceExporter` protocol has an OTLP binding for production and an in-memory
  binding for tests (`bzh:pluggable-seams`).

The runner is untouched in this slice.

## What a sweep does

1. **Select** the steps that closed after the cursor and at least the settle window before now, in cursor order, up to a
   batch limit.
2. **Assemble** each step's spans.
3. **Export** the batch.
4. **Advance** the cursor to the last exported step, only after the exporter accepts the batch.

The settle window (default five minutes) covers two lags:

- **Runner drain lag.** A step's `usage.recorded` can land after its transition.
- **Commit lag.** A fact's recorded time is stamped before its transaction commits, so it can become visible after a
  sweep has already passed that time.

A fact that lands after its step was told is not re-sent by the live sweep. A replay tells the step again from the
record as it stands then.

## The cursor

The cursor is the position of the last step told, ordered by:

1. the closing fact's recorded time
2. then `chunk_id`
3. then epoch
4. then decision id, empty for runner and hub steps

That gives a total order across every closing-fact table. Each sweep that advances the cursor appends one fact row
holding the position, the span count and the export time. The table grows by one row per advancing sweep.

When fleet tracing is enabled on a hub with no cursor row, the cursor starts at the current time. When it is re-enabled
after a cursor older than the maximum lag, it jumps to the current time and records the window it skipped. History is
told only by replay.

## Delivery semantics

Delivery is at least once. A hub killed between export and cursor write re-sends that batch on restart with identical
ids. Backends that deduplicate by span id absorb it; others show a duplicate. The documentation says which is which.

A failed export leaves the cursor where it is. The sweep retries on later ticks with exponential backoff up to a cap.

If the oldest unsent step falls further behind than the maximum lag (default 24 hours), the cursor jumps to the lag
boundary. An outage costs a gap in the chart, never an unbounded backlog on the hub.

Four new `event_log` kinds record these moments. The change routes through the event-kind vocabulary in
`foundation/event_log.py` and `domain/operations.md` §Event kinds:

| Kind                     | Severity | When                                                      |
| ------------------------ | -------- | --------------------------------------------------------- |
| `trace-export-failed`    | warning  | the first failed export after a success                   |
| `trace-export-recovered` | info     | the first success after a failure                         |
| `trace-window-skipped`   | warning  | the cursor jumped, carrying the skipped window for replay |
| `trace-config-rejected`  | warning  | tracing was configured in a way the hub cannot honor      |

Nothing outside the sweep waits on the exporter, and no fleet write path calls it.

## Building spans

The ids are derived (spans.md §Identity), and the SDK's tracer mints its own. So the hub builds finished span data
directly and hands batches to the OTLP span exporter.

It installs no global tracer provider and no instrumentation. Request, CLI and query tracing belong to the
platform-spans slice, through their own pipeline.

Dependencies are `opentelemetry-sdk` and `opentelemetry-exporter-otlp-proto-http`. Only `http/protobuf` ships.

A protocol the hub does not ship, such as `OTEL_EXPORTER_OTLP_PROTOCOL=grpc` or its traces-specific form, never stops
the hub from starting. A hub that redeploys itself must not be taken down by a tracing setting. Instead fleet tracing
stays off, the hub records one `trace-config-rejected` event naming the setting, and `traces status` reports it with a
pointer to a collector, which can bridge any protocol.

## Configuration

Fleet tracing is enabled when all of these hold:

- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` or `OTEL_EXPORTER_OTLP_ENDPOINT` is set
- `OTEL_TRACES_EXPORTER` is not `none`
- `OTEL_SDK_DISABLED` is not `true`

Headers, timeout, compression, certificates and resource variables are the protocol's own, honored as the SDK defines
them. Blizzard adds no endpoint variable of its own.

The endpoint enables fleet spans and nothing else. Platform spans keep their own switch, so setting an endpoint never
starts request or query tracing as a side effect.

With fleet tracing off, no sweep starts, no cursor row is written, and the hub behaves as it does today.

Blizzard's own knobs go in `blizzard-hub.toml` under `[tracing]`. Each has a default that works unset:

- `sweep_seconds`
- `settle_seconds`
- `batch_limit`
- `max_lag_seconds`
- `replay_max_window`

## Operator surface

Each command is a hub API route behind operator auth, with a CLI verb that is a pure client of it.

`blizzard hub traces status` reports:

- whether fleet tracing is on, and if a setting was rejected, which one
- the endpoint's scheme and host, never its path, query or headers
- the cursor position and its lag
- the last successful export
- the last error

`blizzard hub traces replay --since <t> --until <t> [--dry-run]` tells every step that closed in the window:

- **Same assembly, same ids.** A replayed step uses the same assembly and the same ids as the live sweep. What it says
  reflects the record when it runs, late-landing facts included.
- **Cursor untouched.** It never moves the live cursor.
- **Bounded window.** It refuses a window larger than `replay_max_window`.
- **Dry run.** `--dry-run` reports step and span counts without sending anything.

## The published shape

Span names, attribute names and types, the id derivation and the resource attributes form a versioned contract under
`contracts/traces/`. That directory holds the attribute dictionary and golden span output for a seeded scenario. A
change to the shape fails the golden comparison and follows `docs/versioning.md`'s deprecation path, and a breaking
change raises `blizzard.trace.schema_version`.

`docs/deployment/tracing.md` is a new page beside `docs/deployment/observability.md`, which links to it. It covers:

- turning tracing on
- the trace shape and every attribute
- what never leaves
- the start-time bias of runner steps
- duplicate handling
- replay
- arrival order: a step's root arrives at least a settle window after it closes, while spans other slices make inside it
  arrive live, so a collector's tail sampling or trace-assembly window must outlast the settle window, or it splits or
  drops those traces
- a collector configuration that fans one stream to two backends

## Verification

- **Unit.** Assembly is tested over fact fixtures for each of these cases:
  - a runner step
  - a first claim with its queue and claim spans, including time paused or blocked before the claim
  - a claim released unused
  - a graph gate and a runner gate, both resolved and unresolved, including two on one epoch
  - a gate closed by each of a transition, a migration, an escalation and a restart
  - a gate resolved hours before its runner picked it up, with the `gate` span ending at `resolved_at`
  - an ask, including runner-clock skew
  - a pause inside a step
  - a hub step that polls pending across several slots and then bounces
  - a hub step that escalates at its bounce cap
  - a migration, and a migration landing on a hub node
  - an escalation followed by a requeue
  - a detach
  - a stop, a completion, and an operator restart
  - usage landing inside and outside the settle window
  - the point-in-time position fold

  Id derivation is pinned by known vectors.
- **Service.** These are tested against the in-memory exporter:
  - the cursor advances only on success, in total order across closing-fact tables
  - a failure backs off and records once
  - the lag cap jumps and records the window
  - re-enabling after a long disable
  - a disabled hub starts no sweep
  - a `grpc` protocol setting leaves the hub running with tracing off, one `trace-config-rejected` event and the reason
    in `traces status`
  - replay emits identical ids and leaves the cursor alone
  - a kill between export and cursor write re-sends the same ids, which joins the hub's crash sweep per
    `architecture/crash-correctness`
- **End to end.** These e2e scenarios run against `blizzard-mock`, with an OpenTelemetry collector whose file exporter
  is read back, and must match the golden shape:
  - acceptance
  - review cycle
  - gate decision
  - ask and answer
  - delivery conflict
  - migration
  - escalation
  - a hub poll timeout
- **Proof.** One collector forwards the same emission to a self-hosted trace store and a hosted service, with no
  blizzard change between them. A chunk's slowest step is found from each.
