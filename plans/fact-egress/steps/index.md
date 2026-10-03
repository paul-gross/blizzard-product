# Plan — `epic:fact-egress`, steps slice

On a Monday morning the operator wants three numbers: what the fleet cost last week, which station cost the most, and
whether either is moving. The hub can answer each of them, once. `blizzard hub analytics summary` adds up spend by node
for any window it is asked about, but the answer is a total, already summed, and keyed by a node id that changes every
time the graph is published again. To see a trend, someone asks the same question seven times, renames the ids by hand,
and pastes the results into a spreadsheet. Next Monday it costs the same afternoon again.

[`persona:application-architect`](../../../charter/personas/application-architect.md), who runs the fleet, feels it
every week. [`persona:harness-engineer`](../../../charter/personas/harness-engineer.md) feels it the moment they change
a prompt and want to know whether the station got cheaper, because the question is not "what did it cost" but "what did
it cost before and after", and no total answers that.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

**The exit itself.** A directory the operator names, and a hub that writes finished files into it on a cadence, reading
its record forward from where it last stopped. A file, once written, is never touched again, so anything that copies new
files elsewhere — a nightly `rclone`, a warehouse's own loader, a person with DuckDB — can trust that what it already
copied is still true. A hub with no directory named behaves exactly as it does today.

**A row for every node step.** One attempt at one node is one row, and the row says everything a chart might group by,
by name: the chunk and its work items, the graph and node it stood at, which visit to that node it was, the runner that
held it, how it ended, the choice it resolved to and where that led, when it began and when it ended, how long it waited
in the queue or on a person, and what it spent in tokens and money. A gate a person decided is a step of its own, and so
is a delivery the hub ran itself. Nothing is summed beyond the step, so "cost by node by week" and "cost by graph by
runner" are both one query in the operator's own tool.

**A row for every invocation.** A step that ran on two models has no single model, and its cost cannot be split once it
has been added up. So each launch, resume and judgement gets a row of its own, with its model, harness, tokens and cost,
carrying the step it belongs to. This is the finest grain the hub knows about spending, and every step total can be
rebuilt from it.

**The same steps tracing tells.** [`epic:tracing`](../../../delivered/tracing/index.md) tells these steps as traces, and
the two must never disagree about when a step ended or what it cost. Both read the step from one shared definition,
built in the first tracing slice, and every row carries the trace id of its step. A spike in a chart is one click from
the trace that explains it.

**A shape that holds still.** The columns are a published interface. They are versioned, documented column by column
with what a null means and which values are possible, and changed only on a deprecation path, so a dashboard built in
the spring still works in the autumn. The data dictionary is a deliverable of this slice, generated from the same
contract the tests check.

**From now, and history on request.** Turning the export on starts at that moment. An operator who wants last quarter
backfills it, deliberately, and gets the rows the live export would have written.

## What it is not

It is not the derived events — files read, skills fired, agents spawned — which the [events slice](../events/index.md)
exports once this exit exists. It does not move files anywhere beyond the directory, hold a credential to anything, or
load a warehouse. It exports no content: no prompt, question, answer, chunk title or body, check output, or the name of
anyone who decided a gate.

## How it is proven

Against `blizzard-mock`, a night of chunks that build, bounce, wait on a gate and escalate is exported, and DuckDB,
pointed at the directory with nothing but the data dictionary, answers cost by node by day and the slowest station of
the week. The same files, copied by a stock tool into a warehouse, answer the same questions in a BI tool with no
blizzard change between them. The [specification](./spec/index.md) owns the detail.
