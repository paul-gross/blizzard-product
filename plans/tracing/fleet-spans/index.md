# Plan — `epic:tracing`, fleet-spans slice

The morning after a chunk took six hours, the operator wants to walk the night the way they would walk a slow request:
open the trace, see the long bar, and click into it. Today they cannot. The hub knows the answer — it recorded when the
chunk was claimed, when each attempt began and ended, how long a gate sat waiting for a person, which delivery bounced
and why — but it keeps those moments as separate rows in a store, and assembling them into a timeline is a script
somebody writes for one chunk and throws away.

This slice gives the hub its exit. Every node step a chunk takes becomes a trace, built from the facts the hub already
holds and sent to whatever observability backend the operator has pointed it at. It covers the whole fleet layer the
[epic plan](../index.md) names — the step, the wait before it, the gate, the ask, the hub's own delivery, the bounce —
using only what reaches the hub. What the runner alone saw, each harness invocation inside a step, comes in a later
slice.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

**A trace per node step, told from the record.** One attempt at one node is one trace: the claim that started it, the
work, and the outcome it ended in. Inside it, the time the chunk sat in the queue before a runner took it, the time it
spent paused, and the time an ask waited for its answer each appear as their own span, so the long bar names itself. A
gate is a step of its own, because the person deciding it is doing the work at that moment: its trace runs from the
instant the decision was put to them until they made it. A step the hub executed — a delivery, a findings record — is a
trace of the same shape, and a delivery that bounced the work backward says so, with the cause. Each step links to the
attempt before it, so a chunk that went round three times reads as three traces joined in order rather than three
strangers.

**Wide spans, carrying names.** A person asks about the "build" station, not about a minted id that changes every time
the graph is published again. Every span carries the chunk, its work item, its graph and node by name, the runner that
held it, the harness and model that ran it, the choice it resolved to, and which visit to this node it was. A backend
that cannot join across spans answers the same questions as one that can. The tokens and cost a step spent sit on the
step itself, once, so adding up a night never counts a dollar twice.

**Emission that waits on nothing.** The hub does not trace as it works; a separate sweep reads the record forward and
tells each step once it has closed. Nothing in the fleet's path calls the exporter, so a backend that is down or slow
leaves a gap to fill later, never a stalled chunk, and a hub killed mid-sweep picks up where its record says it stopped.
The price is that a step appears when it ends, not while it runs — the right trade for a capability whose questions are
asked the morning after.

**Off until asked for, and history on request.** An operator turns tracing on by naming an endpoint in OpenTelemetry's
own standard configuration, and blizzard learns nothing about what listens there. A hub with no endpoint behaves exactly
as it does today. Turning it on starts the story at that moment rather than flooding someone's backend with a year of
history; an operator who wants the history replays a window of their choosing, deliberately, and gets the same traces
the live sweep would have sent.

**A published shape.** The span names and attributes are an interface the operator's views will be built on, so they are
documented and versioned like blizzard's other wire contracts, and change on a deprecation path rather than in a
refactor. The documentation is a deliverable: what each attribute means, how a trace is shaped, and an example collector
configuration that fans one stream to two backends.

## What it is not

It is not a live view of work in progress; a step still running has not been told yet. It does not reach into the
runner, the harness, or the worker's environment, and it changes no fact the fleet records. Content stays home: spans
carry dimensions and numbers, never prompts, transcripts, file contents, or the text of an ask or its answer.

## How it is proven

Against `blizzard-mock`, a chunk driven through a build, a review that sends it back, a gate, an ask, and a delivery
that bounces once produces the traces the [specification](./spec/index.md) describes — shape, links, and attributes —
checked against what a collector actually received. Then the same emission is pointed, through one collector the
operator configures, at a self-hosted trace store and a hosted service with no blizzard change between them, and someone
who has never seen the store answers where a slow chunk's hours went.
