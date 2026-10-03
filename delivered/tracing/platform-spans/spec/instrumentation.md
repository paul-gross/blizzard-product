# Instrumentation contract

## What is traced

Platform spans are made inline. The daemons use the OpenTelemetry SDK, one tracer provider per process, exporting
through a batch span processor that queues and drops under pressure and never makes a caller wait. The CLI loads no
OpenTelemetry library at all, in either of its roles: see [The CLI never pays for it](#the-cli-never-pays-for-it).

| Process      | Spans                                                                                                                                                                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| hub          | each API request (FastAPI instrumentation), each store query (SQLAlchemy instrumentation), each outbound HTTP call (httpx instrumentation), each hub node's `run:` steps under the fleet-spans `hub-exec` span (nesting.md), each sweep pass as its own root |
| runner       | each local API request on TCP and on the socket, each store query, each call to the hub through its hub client and hub proxy, each tick as its own root, with its steps (reap, resume, pull, fill, advance, and the rest) as children                        |
| worker CLI   | each command through `WorkerCall`, hooks included, `heartbeat` excepted                                                                                                                                                                                      |
| operator CLI | each `blizzard hub …` and `blizzard runner …` command an operator runs, and its requests                                                                                                                                                                     |

The harness, the model it calls and the tools it uses are not instrumented here.

**Never traced.** Two runner routes are left out of its server instrumentation:

- the worker's `POST /heartbeat`, for the reason nesting.md gives. It is answered by the runner alone and never reaches
  the hub. The runner's own liveness heartbeat to the hub is a different request, and is traced as a sampled root like
  any other.
- the `/v1/traces` receiver, so that every export does not itself make a kept span

## Sampling

Both daemons use a parent-based sampler:

- A span with a sampled parent is always kept. That covers every call made inside a step, since derived step contexts
  are sampled.
- A root (a tick, a sweep pass, a board request, a drain) is kept at `platform_sample_ratio`, which defaults to `0.01`.

An explicitly set `OTEL_TRACES_SAMPLER` replaces this default.

Operator CLI commands are always sampled, because an operator runs them by hand and rarely.

## Configuration

| Where                      | Enabled when                                                                                              | Exports to              |
| -------------------------- | --------------------------------------------------------------------------------------------------------- | ----------------------- |
| hub                        | `[tracing] platform = true` in `blizzard-hub.toml`, and an OTLP endpoint is configured                    | the OTLP endpoint       |
| runner                     | `[tracing] platform = true` in `blizzard-runner.toml`, and an OTLP endpoint is configured                 | the OTLP endpoint       |
| worker CLI                 | `BLIZZARD_TRACEPARENT` and `BLIZZARD_RUNNER_URL` are both present                                         | the runner (nesting.md) |
| operator CLI               | no `BLIZZARD_TRACEPARENT` from a runner, and an OTLP endpoint is configured in the operator's environment | the OTLP endpoint       |
| other programs in a worker | `[tracing] worker_programs = true` in `blizzard-runner.toml`, alongside `platform`                        | the runner (nesting.md) |

"An OTLP endpoint is configured" means the same standard variables fleet-spans reads (`fleet-spans/spec/emission.md`
§Configuration). `OTEL_SDK_DISABLED=true` turns every row off.

The `platform` switch is separate from the endpoint, so an operator who enables fleet spans does not also start tracing
every request and query.

The daemons export `http/protobuf` only, as fleet spans do. The CLI sends OTLP/JSON. The runner's receiver accepts both
encodings (nesting.md). Not every backend accepts JSON, which is one more reason the operator CLI's documented path runs
through a local collector (below). `platform_sample_ratio` sits beside `platform` in each daemon's `[tracing]` table.

`service.name` defaults are applied only when the environment names none:

- `blizzard-hub`
- `blizzard-runner`
- `blizzard-cli`

## The CLI never pays for it

A worker runs `blizzard` commands throughout its step, so tracing must cost a command next to nothing.

**The CLI does not import the OpenTelemetry SDK.** A command's spans are a handful at most: one for the command, plus
one per request it makes. The CLI builds them itself and encodes them as OTLP/JSON:

- **Ids.** The trace id and parent from `BLIZZARD_TRACEPARENT`, and a fresh random span id per span. OTLP/JSON carries
  trace and span ids as hex strings, not base64.
- **Times.** OTLP needs Unix-epoch nanoseconds. The CLI reads the wall clock once at start, then measures durations on
  the monotonic clock and adds them to that anchor, so a clock step mid-command cannot give a span a negative length.
- **Attributes.** The ones below.

It posts them to the runner's receiver over `WorkerCall`'s one per-process `httpx` client (nesting.md), and therefore
the same open loopback or socket connection the command already used. There is no SDK import, no batch processor, no
`atexit` flush, no second client and no new connection.

**Budgets.** These are verified, not hoped for:

- **Startup.** Importing the worker CLI costs the same with tracing on as with it off. Nothing tracing-related is
  imported until a command has spans to send.
- **Added latency.** A traced command adds at most 5 ms at p95 over the same command untraced, against a healthy runner
  on loopback. It is measured on the shared-client path, since a second client would spend its budget building a TLS
  context before any I/O.
- **Hard cap.** The send is capped at 100 ms. A runner that is slow, unreachable or refusing costs the command at most
  that much.

Whatever happens to the send, it never changes the command's output or exit code, and it surfaces no error unless
`BLIZZARD_TRACE_DEBUG` is set.

**The worker CLI sends to its runner.** It posts over the client the command already holds, so the added cost is one
small loopback request.

**The operator CLI sends to the configured endpoint.** It uses the same hand-built OTLP/JSON spans, posted with the
standard `OTEL_EXPORTER_OTLP_*` endpoint and headers, and capped at 500 ms. The operator CLI has no runner to send to,
so pointing it at a remote backend costs that backend's round trip. The documentation recommends a local collector,
which brings the cost back to a loopback request. A backend that accepts only protobuf also needs a collector in front
of it.

## Attributes

HTTP and database spans follow OpenTelemetry's stable HTTP and database semantic conventions, pinned to a named semconv
version. Blizzard adds the attributes below where it knows them:

| Attribute                         | On                                                           | Meaning                                                                                                                                                                                |
| --------------------------------- | ------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `blizzard.cli.command`            | CLI spans                                                    | the command path, e.g. `runner ask`, `artifact get`, `hub chunk promote`                                                                                                               |
| `process.exit.code`               | CLI spans                                                    | the command's exit code                                                                                                                                                                |
| `blizzard.caller`                 | CLI and server spans                                         | `worker`, `runner`, `operator` or `board`, from the authenticated principal, never from a claim the caller makes about itself; on worker CLI spans the receiver stamps it (nesting.md) |
| `blizzard.chunk.id`               | worker CLI spans, and server spans whose route names a chunk | the chunk                                                                                                                                                                              |
| `blizzard.lease.id`               | worker CLI spans                                             | the lease the command acted under                                                                                                                                                      |
| `blizzard.runner.id`              | runner spans, and hub spans a runner made                    | the runner                                                                                                                                                                             |
| `blizzard.tick.step`              | runner tick children                                         | the tick step's name                                                                                                                                                                   |
| `blizzard.hub.run_step.name`      | `run:` step spans                                            | the step's authored name, never its command line                                                                                                                                       |
| `blizzard.hub.run_step.exit_code` | `run:` step spans                                            | the step's exit code                                                                                                                                                                   |

A server span is named by its route template (`GET /api/chunks/{chunk_id}`), never by its raw path.

These attributes, the CLI's instrumentation scope `blizzard.cli` and the hub and runner scopes join the fleet-spans
contract under `contracts/traces/`, under the same `blizzard.trace.schema_version`.

## What never leaves

No platform span carries any of these:

- **Request and response bodies.**
- **Credentials.** No header at all, so no authorization, cookie, lease token, route token or hub token.
- **URLs beyond the path.** No query string, no URL fragment.
- **Bound query parameters.** Statements are recorded in their parameterized form, and SQL commenting is off.
- **Command content.** No command-line argument or flag value; `runner ask "…"` records `runner ask`. No subprocess
  command line, output or environment.
- **Content and identities.** No artifact, question, answer or work-item text, and no person's name or login.

These are rules over the instrumentation's configuration and the export pipeline, not over a review:

- **Capture off.** Every instrumentation is set up with its body, header and parameter capture disabled.
- **A redacting processor.** Configuration alone cannot keep URLs clean. HTTP client and server instrumentation record
  `url.full` and `url.query` by default, with no switch to stop them, and an outbound call such as a forge API request
  can carry a token in its query string. Each daemon therefore runs a span processor ahead of export that removes
  `url.query` and strips the query and fragment from `url.full`.
- **Received spans.** Spans the runner receives from a worker pass the receiver's rules (nesting.md) and then the same
  processor.

The verification below scans spans for known secret and content values.

## Verification

- **Unit.**
  - `TRACEPARENT` composition for every invocation kind
  - `WorkerCall` span and context injection
  - command-path extraction with arguments stripped
  - the worker CLI's 100 ms cap against a hung runner receiver, and the operator CLI's 500 ms cap against a hung
    endpoint
  - the CLI's import graph with tracing on, in either role, contains no `opentelemetry` module
  - OTLP/JSON encoding: hex ids, and epoch-nanosecond times anchored once to the wall clock
  - `heartbeat` makes no span and injects no context
  - the CLI reads `BLIZZARD_TRACEPARENT` and ignores a `TRACEPARENT` some other process rewrote
  - the redacting processor strips a planted query-string token from `url.full` and `url.query`
  - the exit code preserved when export fails
- **Service.** In-memory exporters on both daemons:
  - the 5 ms p95 latency budget, measured over a run of `artifact get` calls against a live runner on the shared-client
    path
  - a worker command's spans chain command → runner request → hub request → queries under the derived step root
  - the worker's `heartbeat` and `/v1/traces` requests make no server span on the runner
  - the receiver replaces a planted resource and `blizzard.caller`, stamps chunk and lease from the presenting lease,
    and drops an attribute outside the CLI's set and one past the size cap
  - an `OTEL_*` name in `[worker] env_passthrough` is dropped while tracing is on
  - the receiver drops a span from another chunk's trace, and one past the rate cap
  - completion, decision and hub-advance calls parent as nesting.md says
  - `run:` step spans parent on the derived `hub-exec` span, no second execution span exists, and the driving request
    links to it
  - roots sample at the configured ratio, and step-parented spans at 1
  - the `platform` switch and the endpoint are independent
  - with `worker_programs` off, a worker's environment carries no `OTEL_EXPORTER_*` variable; with it on, an SDK-based
    child program's spans arrive inside the step, and its spans for another chunk are dropped
  - a scan of every emitted attribute finds none of the lease token, route token, hub token, ask text or artifact body
    the test planted
- **End to end.** The acceptance, ask-and-answer and delivery-conflict scenarios run against `blizzard-mock` with a
  collector whose file exporter is read back. The worker's commands sit inside the step traces that fleet-spans tells
  for the same run, and the hub-node execution sits inside the delivery's step. A second run puts a tail-sampling
  processor in the collector: with a `decision_wait` shorter than the settle window a step's trace arrives split, and
  with the documented setting it arrives whole.
- **Proof.** On the same collector setup as fleet-spans, a person opens a slow step and names the commands the agent ran
  inside it, and the slowest of them.
