# OpenCode transcript specification

The OpenCode transcript source implements the established harness transcript seam and feeds the transcript lane owned by
`epic:transcripts`. It reads OpenCode's documented surfaces, never its private database schema.
`opencode export
<session-id>` returns `{info, messages[{info, parts}]}` and resolves a session by id from any working
directory, which makes it the first implementation; a managed server may later optimize reads, but it remains a read
client rather than the lease's worker process.

## Forward reads

`turns_since` reads the root session's messages and parts in stable order. Its opaque cursor carries the last admitted
message and part identities, not an array offset that changes when history is compacted. A cold read returns the bounded
retained conversation; a forward read returns only identities after the cursor. Partial live tool states remain valid
turns, and a later completed state produces the output patch needed to finish an already-shipped call.

`read_raw_lines` returns one stable serialized usage-bearing record per assistant message within the requested position
range, for interrupted-invocation recovery. `size_bytes` measures the serialized root-session representation the source
actually reads, excluding child sessions because they are not context a parent resume pays again. `context_tokens`
derives the newest root assistant turn's prompt-context size only after the compatibility proof establishes which
OpenCode token fields represent that quantity; otherwise it returns unknown.

Every read is bounded, timeout-limited, and safe against OpenCode writing the session concurrently. Missing sessions
return `not_found`; an export or API failure returns `unreadable` and logs once at the source boundary. Transcript
absence never blocks completion, but it leaves context and byte rotation honestly unmeasured.

## Normalization

The OpenCode normalizer stamps `opencode/<normalizer-version>` and maps source records into Blizzard's existing
vocabulary:

| OpenCode source                              | Normalized turn                                                                        |
| -------------------------------------------- | -------------------------------------------------------------------------------------- |
| User text and injected continuation messages | `env`                                                                                  |
| Completed root assistant text                | `asst`                                                                                 |
| Reasoning parts                              | `thinking`, redacted when only presence is available                                   |
| Tool parts                                   | `tool`, retaining structured input, call id, output, error, and truncation             |
| Child sessions                               | Sidechain conversation attached to its spawning tool when a stable relationship exists |

The normalizer never manufactures a parent link. A child that OpenCode identifies only by root parent becomes an
unlinked sidechain and remains visible. Tool output arriving after its call's read window uses the seam's late-output
shape rather than duplicating the call.

## Invocation boundaries

Recovering the usage of a worker that died mid-turn works today by reading a session's whole transcript and summing it.
That is correct for Claude Code only because a Blizzard session has so far belonged to one generation at a time. It does
not survive a harness whose sessions are read through a paged export, and it was already the reason a resumed session
could be charged twice.

This slice replaces it with a range read, and does so on the shared seam rather than inside the OpenCode binding.
`read_raw_lines` gains a start and an end position and returns only the usage-bearing records between them; the Claude
Code source implements the same signature over its byte cursors, where its existing whole-file read is the degenerate
case of a range that starts at zero. Both sources therefore change, and this slice owns both.

Every fleet-driven invocation records a durable transcript start position before sending its message: worker spawn
generation, resume generation, judgement, and nudge each own one boundary, stored beside the invocation's own facts in
the runner store. For a fresh session the beginning of the newly captured session is the start position; for every
existing session the source reads and records the current tail before process launch or synchronous prompting. That
write is ordered before the launch it describes, so a crash between them leaves a boundary describing a turn that never
happened — harmless, because the range it opens stays empty — rather than a turn with no boundary at all.

The closing position is the next invocation's durable start when one exists, otherwise the current readable tail after
the process exits. A recovery read with no trustworthy start returns no sample rather than charging the whole session to
one generation. Because both ends are durable, a crash can repeat the read and produce the same message-id set, which is
what makes repeating recovery a no-op instead of a second charge.

## Usage recovery and provenance

The source exposes per-message usage without summing cumulative session totals repeatedly. Interrupted worker recovery
counts only model steps inside the invocation's durable position range, deduplicated by OpenCode message id. It records
token counts when available and leaves cost unknown when no invocation result supplied a provider-reported figure. Usage
remains keyed by lease, generation, and invocation kind, so repeating recovery is an exact no-op rather than another
charge.

Each batch carries the OpenCode normalizer version and observed OpenCode version. Transcript segments gain the producing
harness id beside the harness version they already carry — on the shipped record, on the runner's segment table, and on
the hub's — because a version alone cannot say which harness produced it once two are in the fleet. The field is
optional in the same way `harness_version` already is, so a runner one minor behind ships records without it and a hub
reading them stores a segment of unknown origin rather than rejecting the batch. The hub receives normalized turns only;
raw OpenCode exports remain runner-local.

## Performance boundary

The export implementation is accepted only if transcript pumping, panel reads, and context sampling remain bounded at
the runner's supported concurrency. If process startup or whole-session export makes that false, the source moves to the
documented OpenCode server API behind the same seam. The server may be shared for reads because it owns no lease PID and
receives the recorded workspace on each request; worker execution remains per-turn.
