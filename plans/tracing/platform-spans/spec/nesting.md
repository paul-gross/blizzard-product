# Nesting contract

How a span made inline lands inside the fleet-spans trace of the step it served. Everything here rests on one property
of the fleet-spans [identity](../../fleet-spans/spec/spans.md#identity): a step's trace id and root span id are derived
from its step key. Any process that knows the chunk and the epoch can therefore parent a span on the step without asking
anyone, and the root arrives later, from the hub's sweep. Backends assemble a trace by id whatever order its spans
arrive in, within limits [Arrival order](#arrival-order) states.

## Into the worker

When platform tracing is enabled on the runner, every invocation the runner starts under a lease carries the step's
trace context in the worker's environment. That covers spawn, resume, nudge and judgement. The value is the W3C
`traceparent` of the step's root, derived from `(chunk_id, lease epoch)`, with the sampled flag set. It is set under two
names:

- **`BLIZZARD_TRACEPARENT`** is the one the `blizzard` CLI reads, and the only one. Any OpenTelemetry-aware process
  between the harness and a command may rewrite `TRACEPARENT` for its own children. The CLI would then send spans under
  a trace the receiver refuses, and they would vanish without a word. A blizzard-owned name nothing else writes keeps
  the CLI in its step.
- **`TRACEPARENT`** is the name OpenTelemetry's environment-carrier convention uses, for other programs (below).

Both are part of the identity environment the runner composes for the spawn, beside the other `BLIZZARD_*` variables,
not a passthrough. They carry ids and a flag only. Every invocation under one lease shares the lease's epoch, so a
resumed or judged session lands in the same step.

Most OpenTelemetry SDKs, Python's included, do not read `TRACEPARENT` from the environment on their own. A program joins
the step's trace only if it extracts the variable itself. Blizzard configures no such program and promises nothing about
one. The operator documentation says so.

## Through a command

`WorkerCall` is the one client every worker-facing `blizzard` command goes through. Today it calls `httpx`'s module
functions, which build a fresh client and connection for every request. This slice changes it to own one `httpx` client
per process, built on first use and reused for every request the command makes, the span post included.

When `BLIZZARD_TRACEPARENT` is present, `WorkerCall`:

1. Opens the command's span as a child of that context.
2. Injects the span's context into its request to the runner's local API.
3. Sends its spans to the runner (below) before the process exits, over that same client and connection, within the
   budgets `instrumentation.md` sets. The worker CLI does this without the OpenTelemetry SDK.

Hooks such as `session-end` go through the same client and are traced the same way.

**`heartbeat` is never traced.** It runs after every tool call the agent makes, and spans inside a step are never
sampled away. Tracing it would put thousands of spans into every step that say nothing more than that a tool ran. The
CLI opens no span for it and injects no context, and both daemons leave its routes untraced (`instrumentation.md`).

## Out of the worker

A worker's spans are sent to the runner it already reports to: OTLP over HTTP at `<BLIZZARD_RUNNER_URL>/v1/traces`, in
either JSON or protobuf encoding, authenticated by the lease token the worker already holds. The runner forwards them
through its own exporter. So:

- **No backend credential reaches the worker.** It never receives the trace backend's endpoint or credential. By default
  no `OTEL_EXPORTER_*` variable is put in its environment either, so no other program the worker runs is redirected at
  the runner. The opt-in below is the one exception, and it points only at the runner.
- **Its spans stay in its own step.** The runner accepts a span only when its trace id is the trace id of a step under
  the presenting lease's chunk and epoch. It drops the rest, counting them in the runner's tracing status.
- **The runner states who sent it.** The lease token sits in every worker's environment, so the agent itself can post
  anything to this receiver. Nothing a sender claims about itself is kept. The receiver replaces the resource with its
  own, setting `service.name` to `blizzard-cli`, and stamps `blizzard.caller = "worker"`, `blizzard.chunk.id` and
  `blizzard.lease.id` from the presenting lease, overwriting any value the span carried.
- **It keeps only what the CLI sends.** With `worker_programs` off, the receiver accepts only spans under the CLI's
  instrumentation scope, and keeps only the attributes the CLI defines (`instrumentation.md` §Attributes). Anything else
  is dropped, so an agent cannot use the receiver to carry content out.
- **The receiver is bounded.** It caps request size, per-lease rate, attribute count per span and attribute value
  length, and refuses or truncates past each.

## Other programs in the worker

A worker runs more than `blizzard` commands. Workspace tooling such as `winter`, the project's test suite, and anything
else that speaks OpenTelemetry can join the step's trace through `TRACEPARENT`. What it lacks by default is somewhere to
send its spans.

`[tracing] worker_programs = true` in `blizzard-runner.toml` supplies one. It is off by default, and it takes effect
only alongside `platform`. With it on, the runner adds three standard variables to the worker's environment:

- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`, set to the receiver above
- `OTEL_EXPORTER_OTLP_TRACES_PROTOCOL=http/protobuf`
- `OTEL_EXPORTER_OTLP_TRACES_HEADERS`, carrying the lease token

The lease token is already in the worker's environment, so no new credential crosses the boundary.

The harness is one of these programs. Claude Code reads the same variables, and with this switch on, whatever telemetry
the harness emits is sent to the runner and kept if it falls inside the step. The operator documentation names this.

Any OpenTelemetry-aware program the worker runs then sends to its runner, and its spans land inside the step under the
same acceptance rule: a span outside the presenting lease's step is dropped.

It is opt-in for one reason. Blizzard controls what its own spans carry, and it cannot control what a third-party
program puts in its spans. A project's test instrumentation may record request bodies or query parameters, and with this
switch on those reach the operator's backend. The operator documentation says so plainly. With it on, the receiver also
accepts other instrumentation scopes and their attributes. It still rewrites the resource, stamps the caller, chunk and
lease, and applies every cap.

**No backend credential through passthrough.** `[worker] env_passthrough` could name an `OTEL_*` variable and carry the
runner's own backend endpoint or headers into every worker, around everything above. While any tracing is on, the runner
drops `OTEL_*` names from the passthrough and logs a warning at startup naming them.

## Across the runner

The runner's API continues the incoming context. When it forwards to the hub through its hub proxy, the outbound request
carries that context too. A worker's read that the runner answers from hub state therefore shows as command → runner
request → hub request → queries, all in the step.

A worker write that lands in the runner's store and reaches the hub later by drain shows only as far as the runner. The
drain's request to the hub is the runner's own and serves many chunks at once, so it is not nested in any one step.

## The runner's own calls for a step

When the runner calls the hub on one step's behalf, it parents the request on that step's derived root:

- **Submitting a completion.** The step is the submission's chunk and epoch.
- **Submitting a decision.** The step that raised it: the decision's chunk and epoch.
- **Applying a resolved decision.** The gate step, keyed by the decision's chunk, epoch and `decision_id`. The runner
  step that raised the gate has already closed, and the pickup belongs to the gate.
- **Re-reading the envelope of a chunk it holds.** The step is the chunk's newest epoch the runner knows.

When it polls a hub node, it parents the poll on the hub step that node will become. A hub step is minted at its exit,
one above the epoch it ran from (`fleet-spans` §Steps). The epoch comes from the hub, not from the runner's own lease
history, which lags whenever one hub node leads straight into another: the poll uses `ChunkStatusView.latest_epoch` plus
one, from the status view the runner already holds when it drives the node. A restart that intervenes before the exit
leaves that guess pointing at a step that never closes. Its spans then sit under a root that never arrives, which
backends show as a missing parent. The documentation names this case.

## Inside the hub

The hub continues an incoming context only from an authenticated caller. An anonymous request starts a fresh root.

When the hub executes a hub node, whichever request drove it, it opens no execution span of its own. Fleet-spans already
tells each execution as the hub step's `hub-exec` span, and its id is derived from the step key and the `hub_exec_slot`
id, both of which the hub knows the moment it acquires the slot. The step is the chunk and the incoming epoch plus one.
So:

- each of the hub node's `run:` steps is an inline span parented on that derived `hub-exec` span
- the request that drove the node links to the derived `hub-exec` span rather than containing it, because a completion
  that triggers delivery belongs to the step that finished, and the delivery belongs to the next

The execution appears once, from the sweep, and its `run:` steps appear inside it.

## Arrival order

The spans in one step's trace arrive at different times. Inline spans from the worker, runner and hub arrive as they
happen. The step's root arrives from the hub's sweep at least one settle window after the step closes, and the runner's
spans from its own sweep after that. Backends that assemble by id at query time are unaffected. A collector doing tail
sampling, or any stage that waits a bounded time for a trace to complete, decides before the root arrives and splits or
drops the trace. The operator documentation states this, and gives a tail-sampling `decision_wait` longer than the
settle window, or none, as the configuration that works.

## When fleet spans are off

Nesting needs the hub's fleet spans to supply the roots. On a hub without them, platform spans still group by trace id
under a root that never arrives. They stay correct and complete, but backends render them beneath a missing parent. The
operator documentation recommends enabling both together.
