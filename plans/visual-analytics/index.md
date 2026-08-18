# Plan — `epic:visual-analytics`

The numbers exist now. `epic:analytics` gave the hub an honest read API over what the fleet actually does — every file a
worker opened, every agent it spawned, every step's duration, outcome, and price — and a set of CLI verbs an analyst can
drive. What it deliberately did not give anyone was a way to *look*. Asking "what did last week cost, and where did it
go" means an operator with a terminal, ten calls, and a scratch script they throw away; the answer arrives once and
survives nowhere. The operator learns something real and then pays the same price to learn it again seven days later.

That is the gap this epic closes. A person who runs a fleet should be able to open a page and see where their money and
their hours went, then follow the surprise wherever it leads — without writing anything, and without asking anyone to
ship a new endpoint first.

## Who this is for, and what their week looks like

The operator wants a Monday ritual: open one page, see the week against the week before, and know within a minute
whether the factory is getting cheaper or dearer per thing shipped. They are not carrying a question when they arrive —
they are looking for the number that moved.

The harness engineer arrives with the opposite posture. They have a specific suspicion — that a review gate rejects more
than it should, that some context file nobody reads is being paid for on every spawn — and they need to slice the same
data along a dimension nobody anticipated. Their session ends when the suspicion is confirmed or killed, and the tool's
job is to get out of the way in between.

Both are underserved today, and they are underserved differently: the first needs a page that answers without being
asked, the second needs a page that answers anything. A surface that serves only one of them is half the epic.

## What the fleet already knows, and what it cannot yet say

The hub holds the facts. Every completed step carries its duration and the choice it resolved to; every attempt carries
its tokens and its price; every finalized transcript segment carries the reads, the skills, and the spawns that happened
inside it. `epic:analytics` exposed all of it through one filter vocabulary.

Three things stand between those facts and a picture, and each is small on its own.

The first is **identity**. Every rollup groups by node id, and a node id is minted fresh each time a graph is minted —
so the single node an operator calls "build" is sixty-seven different keys, and the client is left to join them back
together against every graph version the hub has ever held. Nothing can plot a stable series through that. Names, not
ids, are what a chart's axis is made of.

The second is **time**. The operational rollups accept a time range as a filter but carry no timestamp on a row, so
"cost per day" is not a slow query — it is one request per day, and the shape of a week has to be reassembled by the
caller. A time dimension on the fact is what turns a filter into an axis.

The third, and the one that decides the whole surface, is **grain**. Every endpoint returns a sum. A sum cannot be
re-sliced: once the hub has added up cost by node, no client can ask that same response for cost by node *and week*, and
every new question becomes a new endpoint. The way out is to serve the facts one level below the aggregate — one row per
step, one row per event, with their dimensions spelled out — and let the aggregation happen where the question is being
asked. This is not a query language, and the hub keeps its enumerated routes and its filter vocabulary; it simply stops
pre-adding what the reader might have wanted separately.

The size of the data is what makes this reasonable rather than reckless. The whole fleet's history is on the order of a
thousand steps and a few hundred chunks — small enough to hand a browser in one response, and small enough that
re-grouping it on every click is instant. The fleet would have to grow by two orders of magnitude before that stopped
being true, and the same fact endpoints serve a heavier back end unchanged when it does.

## What to build

**A fact surface.** Two read routes in the analytics namespace, sharing its filters and its operator gate: one row per
completed step, carrying its chunk, its graph and node *by name*, its source, its start and end, its duration, the
choice it resolved to, and its tokens and cost; one row per derived event, carrying its kind, subject, tool, agent type,
depth, and the node context it belongs to. Both stream as NDJSON like their neighbors.

**A page in the hub board**, reached by an operator who is already signed in — no second system to host, no second
identity to manage, no copy of the fleet's data leaving the hub. The board loads the facts once and does its grouping in
the browser.

**Aggregation the reader controls.** Cost per day, per node, per graph, per source, per model — and per any pair of
those — without a code change between the question and the answer. The measures are the ones the facts carry: money,
time, steps, tokens, events. The dimensions are the columns. What the operator picks is which of them lands on which
axis.

**Filters that compose**, in the vocabulary the API already speaks: a time range with the week as its natural unit,
graph, source, node, kind. A filter narrows every panel on the page at once, because the page holds one dataset rather
than one dataset per panel.

**A week that compares to the week before.** The Monday ritual only works if the previous period is on screen without
being asked for. Every headline number carries its delta, and the delta is the thing the eye lands on.

## What this epic is not

It is not a second product. No metrics stack, no separate dashboard service, no copy of the fleet's data in a system
with its own login and its own backup story — the board already authenticates, and the hub already owns the facts.

It is not a query language, and the hub does not grow one. The routes stay enumerated and filtered; what changes is that
they return facts as well as sums.

It is not interpretation. Deciding that a gate which never fails is a gate not worth running, or that an unread context
file should be deleted, remains a person's judgment — and, for the harder sweeps, the analyst LLM's. The page's job ends
at showing the number clearly enough that the judgment is easy.

And it is not a replacement for the CLI. The analyst's tool surface stays exactly as it is; the fact routes make it
better by giving it the same un-summed grain.

## Artifacts

Three explorations of what a fleet's own numbers should feel like to read, from the concept round before the build. They
are seeded with a real snapshot of the fleet — the totals, the node breakdown, and the gate outcomes are the live
figures, not invented ones — so each sheet is arguing about form rather than about plausibility. Where a sheet shows a
week-by-week series, the shape is illustrative: the time dimension the axis needs is exactly what this epic adds.

| Mockup                                                   | What it explores                                                                                                                                                                                                                   |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [pivot-workbench.html](./artifacts/pivot-workbench.html) | The question-builder: dimensions on a shelf, dragged onto rows and columns, with a heat-shaded grid underneath. Answers anything; asks the reader to know what they want. The harness engineer's surface.                          |
| [flight-deck.html](./artifacts/flight-deck.html)         | The instrument panel: a fixed set of readouts for the week, each carrying its delta against the week before, with filters that narrow all of them at once. Answers without being asked; answers only what it was built to. Monday. |
| [flow-atlas.html](./artifacts/flow-atlas.html)           | The workflow itself as the chart: the graph an operator already reads, with its nodes weighted by cost and its edges by traffic, so rework loops and dead gates are visible as shape rather than found in a table.                 |

The three are not exclusive. The workbench and the deck are plausibly one page in two modes — a curated set of views
that the reader can take apart — and the atlas is a view either could hold.
