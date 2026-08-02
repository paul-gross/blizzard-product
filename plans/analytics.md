# Plan — `epic:analytics`

Counting what the fleet actually does. The first hand-run count — how often are skills used? — overturned an assumption the corpus had been maintained under for months: agents constantly, skills nearly never. This epic makes that kind of question cheap to ask and impossible to guess wrong: which files the workers read, which skills fire, which agents spawn — by node, with what that node was doing — plus the operational numbers the hub already holds, all reachable through one clean read API. There are no dashboards here by design: the consumer is an analyst LLM with tools, and what it needs is composable, predictable data it can transform however the question demands — not charts.

## What already holds

The transcript store (`epic:transcripts`) did the hard part. Normalized turns arrive at the hub with tool calls kept structured — tool name plus input as data, a contract that plan states is held for this epic — and every segment is keyed by chunk, transition, and spawn generation, so "which node did this, and what was it doing" is a join against tables the hub already has, not a mining job. Segments are version-stamped and re-derivable, backfilled history included.

The operational numbers already exist as hub facts: transitions carry step timing and outcomes, usage facts carry tokens and spend, judgements carry verdicts. What they lack is not existence but shape — they serve the board, not an exporting analyst.

The delivery pattern is established too: hub API wrapped in CLI verbs, and the operator role gate that transcripts-derived data inherits.

## What to build

- **Event derivation on ingest.** When a segment finalizes, the hub derives a compact event stream from its turns: one row per occurrence — a file read (tool and path), a skill invocation (skill name), an agent spawn (agent type) — each stamped with chunk, node, graph, transition, session, and timestamp. Kinds are extensible: adding one later is a new derivation, never a schema redesign. The derivation is versioned and re-runnable — a `re-derive` admin verb sweeps stored segments through a newer extractor, so history (backfill included) is never stranded on an old one's blind spots.
- **Sidechain attribution.** An event inside a subagent's conversation carries its parent node's full context plus an agent-type tag and nesting depth — "the review node read this file" stays true when the read happened three layers down inside a spawned explorer, and per-agent-type rollups fall out of the same tag. This is where the counting honesty lives: the sidechains are where most reads happen, and an extractor that skipped them would recreate the very blindness this epic exists to end.
- **The analytics API.** A read-only namespace on the hub: an events endpoint with honest filters — kind, tool, path prefix, node, graph, source, time range — returning plain JSON or NDJSON, plus a small set of canned aggregates (counts by file, by skill, by agent type, by node) so the common questions never require pulling raw events. Operator-gated, like the conversations the events derive from.
- **The operational datasets, exported cleanly.** The same namespace and the same filter vocabulary over what the hub already knows: step durations, tokens and spend per chunk, node, and graph, failure and retry rates by node. One agent session with hub CLI access answers "which context files does the review node actually read, what does that node cost, and how often does it fail" in a single sitting.
- **CLI verbs.** `blizzard hub analytics events …` and `… summary …`, thin over the API — which is precisely the tool surface the analyst LLM drives.

## What this epic is not

Not dashboards: the board does not change, and no chart ships. Not in-platform intelligence: deciding which paths are agent-facing, spotting trends, judging what a never-read file means — that is the analyst's work, done with the corpus in front of it; the platform's job ends at honest data. Not a query language: enumerated endpoints with filters, and SQL stays internal.

## Open questions

- Whether thinking turns yield indexable events or remain read-only prose in the transcript store — deferred until the first real mining session shows what thinking is worth asking about.
- Which aggregates beyond plain counts earn canning — decided after the first analysis sessions reveal the questions that repeat.
