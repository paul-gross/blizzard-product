---
epic: tracing
refinement: refined
slices:
  - name: fleet-spans
    status: in-progress
    plan: ./fleet-spans/index.md
  - name: platform-spans
    status: in-progress
    plan: ./platform-spans/index.md
  - name: runner-spans
    status: in-progress
    plan: ./runner-spans/index.md
---

# Plan — `epic:tracing`

A chunk that took six hours and cost four times what anyone expected leaves almost nothing behind that explains itself.
The operator can see that it happened and can add up what it spent, but the shape of the night — where the hours
actually went, which station it bounced off and how many times, how long it sat in the queue before any of it started —
has to be reconstructed by hand from rows in a store, if it can be reconstructed at all. The harness engineer arrives at
the same wall from the other side: they have a suspicion, usually a specific one, and no way to follow it except to
write a query against a shape nobody designed for the question.

Every other industry that runs distributed work solved this with tracing, and the tools are extraordinary. A chunk
moving through a workflow graph is a trace in every sense that matters — a unit of work, crossing stations, each leg
with a duration and an outcome and a parent. Blizzard has the facts already. What it lacks is the one standard verb that
hands them to any of those tools.

## Why the harness cannot answer this

The obvious economy is to let the coding agent report on itself. Claude Code emits OpenTelemetry, and it emits genuinely
useful things: model, tokens, cost, tool decisions, all scoped to a session. It is tempting to stop there.

It does not work, for reasons that are structural rather than incidental. A harness knows it is in a session; it has no
idea it is a chunk, and no idea which station of which graph it is standing at. It exits before the fleet judges its
work, so whether the gate approved, rejected, or bounced it is a fact the session never learns. It is absent entirely
for the nodes the hub executes itself, delivery among them. It cannot account for the time a chunk waited in the queue,
because nothing is running then to account for it. And a chunk that bounced three times is three unrelated sessions with
nothing joining them.

Then there is the harness seam itself. OpenCode emits no telemetry at all, and the moment it becomes a live binding a
Claude-Code-shaped view stops describing the fleet — which the charter treats as a failure, not a gap: interoperability
is proven by seams with two occupants, and a measurement surface that works for only one occupant has quietly welded the
platform to it. Blizzard already normalizes both harnesses into one fact vocabulary. That makes blizzard the only place
where two harnesses are comparable at all, and the only honest place for the fleet's own account of itself.

## What to build

**Spans for the fleet layer**, which is precisely the layer no harness can see: the node step, the gate and what it
decided, the wait in the queue, the hub-executed deliver, the bounce that sent work backwards. These are coarse and few
— a fleet produces them by the thousand, not the million — and they are the skeleton every other number hangs from.

**The agent's own use of the platform, inside the step.** A worker spends its step talking to blizzard: asking, reading
artifacts, writing them, reporting in. Each of those commands, the requests it made of the runner and the hub, and the
queries they ran are traced too, nested inside the step that made them, so the flame graph of a step shows not only how
long the agent worked but how it used the platform while it did.

**Wide spans, carrying every dimension on the span itself**: chunk, graph name, node name, source, model, resolved
choice, epoch, how many times the work has been here before. Tokens and cost are the one exception: they sit once on the
step and once on each invocation, never copied onto every span, because a backend that sums over spans would otherwise
count each dollar several times. The temptation is to lean on the parent-child structure to supply context and keep each
span thin. Resist it. Some backends cannot join across spans at all, and a span that answers questions alone answers
them everywhere.

**One endpoint, configured by the operator, speaking OTLP.** This is the whole of the integration surface, and it is the
reason this epic ships no adapters. OpenTelemetry is the interface; blizzard implements it once and every backend that
exists or will exist works — the operator sets an endpoint through the protocol's own standard configuration and
blizzard learns nothing about what is on the other end. Where the protocol has settled conventions for describing model
invocations, blizzard uses them rather than inventing private attribute names, so that a backend's built-in views light
up without anyone configuring them.

**A span shape decided deliberately.** The intuitive choice — one trace per chunk — is likely the wrong one: chunks live
for days, sleep in queues, and come back after bouncing, while backends assemble traces inside bounded windows and
render hours-long spans poorly. A trace per node step, with the chunk carried as an attribute and a link between related
steps, is the shape to design against. It is expensive to change once anyone has built a view on it, so it is a decision
to make on purpose and early.

**Emission that cannot hurt the fleet.** The same rule the fact export lives under applies here with more force, because
a trace exporter talks to the network on the hot path if you let it. The fleet's steps are told from the record by a
sweep that runs beside the fleet rather than inside it, once each step has closed. The calls an agent makes are traced
as they happen, because a request exists only while it runs, and those spans are queued and dropped under pressure,
never awaited. Either way, a backend that goes down is a gap in a chart, never a stalled chunk.

## Retention is the operator's problem, and that is the point

Trace backends keep data for weeks, not years — the best of them for sixty days. That would be a real limitation if
blizzard were choosing the backend, and it is nothing at all now that it is not. OpenTelemetry's own collector fans one
stream out to as many destinations as an operator wants, each with its own retention: a hosted backend for the hunting,
object storage for the archive, a self-hosted store for whatever is left. All of that is configuration in a file the
operator owns, downstream of the single endpoint blizzard writes to.

So blizzard ships one emit and no opinion about how long a year is.

## Why this earns priority over things that look more urgent

The hub already keeps a dated fact for nearly every moment a span would describe — the claim, the attempt, the gate's
decision, the answer to an ask, the delivery and the bounce — so the fleet's history is not lost while this waits; it is
only locked away. Every week without an exit is a week of questions that go unasked because asking costs an afternoon.
That argues for landing a modest version early rather than a complete one late, and for building it so the same spans
can be told again from the record: a first shape that turns out imperfect is then a replay, not a loss.

## How it lands

The epic lands in three slices:

| Slice                                       | What it builds                                                                                                                                                                                                                                |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [fleet-spans](./fleet-spans/index.md)       | The hub tells every node step from its own facts — queue waits, gates, asks, hub-executed nodes, bounces, and what each step cost.                                                                                                            |
| [platform-spans](./platform-spans/index.md) | The calls a working agent makes — each `blizzard` command, the runner and hub requests behind it, and the queries those ran — nested inside the node step that made them, so a step's trace shows how the agent used the hub while it worked. |
| [runner-spans](./runner-spans/index.md)     | The runner adds what only it saw: each harness invocation inside a step — spawn, resume, nudge, judgement — and the checks it ran.                                                                                                            |

## What this epic is not

It is not a second copy of the harness's inner life. Where Claude Code reports on itself, blizzard does not compete;
what blizzard emits is the layer above, plus the normalized usage it alone can state across harnesses.

It is not a trace store. Blizzard keeps nothing, hosts nothing, and has no retention policy, because it never holds a
span longer than it takes to send one.

It is not a set of vendor integrations. There is no Honeycomb exporter and no Tempo exporter, and there never should be
— the protocol is the integration.

And it is not the fact export wearing different clothes. Spans are moments; the export is the record. An operator
wanting to know what happened during one bad night reaches for one, and an operator wanting to know whether the factory
grew cheaper this quarter reaches for the other.

## How it is proven

One emit, two backends, neither of them privileged: a hosted service and a self-hosted store, both receiving the same
spans through a collector the operator configured, with no blizzard change between them. Exercised against
`blizzard-mock`, so proving it costs no real transcripts. And one question answered end to end that could not be
answered before — where six hours actually went — because a capability nobody can use to learn something is not yet
finished.
