# Plan — `epic:tracing`, platform-spans slice

A worker does not spend its step in silence. It asks the runner for its chunk's history, reads the artifacts an earlier
station left, writes its own, asks a person a question, and reports in after every tool it uses — each of those a
`blizzard` command, each command a request to the runner, and some of them a request onward to the hub. The
[fleet-spans](../fleet-spans/index.md) slice shows the step as one long bar. What it cannot show is what the agent was
doing with the platform inside that bar: whether an hour went to thinking or to forty slow artifact reads, whether a
worker leans on `chunk history` in a loop, whether the hub was the thing that made it wait.

[`persona:harness-engineer`](../../../charter/personas/harness-engineer.md) is the person this is for. Their craft is
changing how agents work and watching what changes, and today the agent's use of blizzard is invisible to them unless
they read a transcript line by line. The operator of a hosted hub has the neighbouring question from the other side:
which requests are slow, which queries are hot, and whose calls they are.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

**The agent's calls, inside the step that made them.** Every `blizzard` command a worker runs is a span, and so is the
request it makes of the runner, the request the runner forwards to the hub, and the database queries each of those ran.
All of it nests inside the fleet-spans trace of the step the worker was working, so opening a slow step in a flame graph
shows the agent's calls laid along it. This needs no lookup and no new field on any wire: the step's trace identity is
derived from the chunk and the attempt, so the runner already knows it when it launches the worker and hands it over in
the standard place OpenTelemetry looks.

**The runner's own calls for a step, in the same place.** When the runner submits a step's outcome, puts a decision to
the hub, or drives a hub node on a chunk's behalf, those requests nest in the step they serve too, so a delivery's trace
shows the hub working through it.

**Everything else, traced and sampled.** Requests that serve no step — a runner checking the queue, an operator running
`blizzard hub status`, the board loading — are traced as their own traces and sampled, so a hosted hub's operator can
see where its time goes without paying to keep every one. Calls made inside a step are always kept: they are the ones a
person opening that step expects to find. The one call left out is the report a worker makes after every tool it uses.
Traced, it would bury each step in spans that say nothing more than that a tool ran.

**Traced as it happens, without slowing anything.** A request exists only while it runs, so these spans are made inline,
by OpenTelemetry's own instrumentation, and handed to an exporter that queues and drops rather than waits. A command
never fails, and never takes noticeably longer, because a trace backend is slow or gone.

**Credentials stay where they are.** A worker holds no hub credential, and it gains no trace-backend credential either:
its spans go to the runner it already reports to, which forwards them under its own configuration. What crosses into the
worker's environment is a trace identity and an address, nothing more. And because the agent can reach that address too,
the runner decides what a received span says about who sent it, and keeps only what blizzard's own commands send.

## What it is not

It is not a log of what the agent said or read. Spans name the command, the route, and the query, never their arguments,
bodies, or results, and never a header or a token. It is not the fleet layer either: whether a step happened, and how it
ended, is fleet-spans' account, and this slice only fills in what happened inside it. And it does not reach into the
harness: the agent's own model calls and tool use belong to whatever the harness reports of itself, and runner-spans
tells only each invocation as a whole.

## How it is proven

Against `blizzard-mock`, a worker that asks a question and reads and writes artifacts produces spans that land, through
a collector, inside the step trace fleet-spans tells for it — command, runner request, hub request, and queries in order
— with nothing secret on any of them. The [specification](./spec/index.md) owns the detail.
