# Plan — `epic:fact-egress`, events slice

The hub reads every transcript a worker leaves and turns it into a short list of what happened in it: each file read,
each skill invoked, each agent spawned, attributed to the station and the subagent that did it. That list is the most
interesting thing the fleet knows about itself. It answers which context files the review station actually opens, which
skills earn their keep, and whether a prompt change made the build station read half as much. Today it can be asked one
filtered question at a time from a terminal, and every answer is gone when the terminal closes.

[`persona:harness-engineer`](../../../charter/personas/harness-engineer.md) needs it most. Their craft is changing what
an agent is told and watching what it then does, and "what it then does" is exactly this list, compared week on week,
station by station, across two harnesses.

| Where                    | Read when                                                              |
| ------------------------ | ---------------------------------------------------------------------- |
| [spec/](./spec/index.md) | Implementing any part of the slice or resolving its technical contract |

## What to build

**The derived events, as a dataset.** Each event becomes a row carrying everything it can be grouped by: its kind, its
subject — the path, the skill, the agent type — the tool that produced it, how deep in a subagent it happened and what
kind of subagent that was, and the step, graph, node, harness and model it belongs to, by name. Every row carries its
step's identity and trace id, so events join to the step rows the [steps slice](../steps/index.md) exports and to the
traces [`epic:tracing`](../../../delivered/tracing/index.md) tells.

**A ledger that corrects itself.** Unlike the facts behind a step, these events are not written once. The hub derives
them again whenever a transcript changes, an operator asks for it, or a better extractor ships, and the old rows vanish
from the store. Files the export has already written cannot vanish with them. So the export keeps the honest record a
ledger keeps: when a transcript's events are derived again, the whole new derivation is written, marked as superseding
the one before it, and when a transcript stops counting at all, a row says so. A reader who wants today's truth reads
the newest derivation of each transcript through a view the documentation ships. A reader who wants to see how an
extractor change moved the numbers still can.

**Only the extractor that counts.** By default the export follows the hub's current extractor and leaves older versions'
events behind, the same default the analytics commands use. An operator comparing extractors turns on every version
deliberately.

## What it is not

It exports no transcript, prompt, tool input or tool output. A file's path is exported because it is the subject of the
event and the point of asking, but by default only relative to the worker's working directory. That way the same file
reads the same from every runner, and nobody's home directory or machine layout leaves with it. An operator can have
paths hashed, dropped, or exported whole, and has to choose that. Nothing else from the conversation leaves. It derives
nothing new: the kinds of event are the ones `epic:analytics` already extracts, and a new kind arrives through that
work, not this.

## How it is proven

Against `blizzard-mock`, a night of chunks whose transcripts read files, invoke skills and spawn agents is exported.
Then a transcript is extended, the events are derived again, and a segment is superseded. DuckDB and a warehouse-backed
BI tool, using only the data dictionary and its view, each report the same current counts the hub's own analytics
commands do, and each shows which files the review station read last week. The [specification](./spec/index.md) owns the
detail.
