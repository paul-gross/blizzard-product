# OpenCode 2 transcript specification

The line-2 transcript source implements the harness transcript seam and feeds the transcript lane owned by
`epic:transcripts`. It reads the server API's documented surfaces, never OpenCode's SQLite schema.

## Reading outside a lease

Transcripts are read while a lease runs and long after it ends, by the transcript pump, usage recovery, and context
sampling. During a lease the source reads through the lease's own server. Outside a lease it launches a short-lived read
server on the line-2 store — the same `serve` invocation with no worker configuration and every tool denied — reads, and
stops it. Read servers are pooled per runner with an idle timeout, so a burst of reads does not start a server per read.
A read never starts, joins, or depends on the operator's managed service.

## Forward reads

`turns_since` reads `GET /api/session/{id}/message?order=asc` with cursor paging. Its opaque cursor carries the last
admitted message id and the session's last admitted event `seq`, not an array offset. A cold read returns the bounded
retained conversation; a forward read returns only messages after the cursor. A message still in progress — an assistant
step whose tool part reads `running` — is a valid turn, and a later read of the completed message produces the output
patch that finishes an already-shipped call.

The `prt_` part ids that `run --format json` prints are client-synthesized and absent from the API, so nothing keys on
them. Message ids are stable and time-sortable; within a message, parts are ordered by position.

`size_bytes` measures the serialized root conversation the source reads, excluding child sessions. `context_tokens`
reads the newest root assistant step's input, cache read, and cache write tokens, excluding `compaction` and `idle`
marker messages; where the proof shows `GET /api/session/{id}/context` reports the model-visible context more exactly,
that becomes the source.

Every read is bounded, timeout-limited, and safe against the server writing the session concurrently. A missing session
returns `not_found`, which the API distinguishes from failure, unlike 1.x export; any other failure returns `unreadable`
and logs once at the source boundary.

## Normalization

The line-2 normalizer stamps `opencode-v2/<normalizer-version>`, distinct from line 1's `opencode-export/1`, and maps
into Blizzard's existing vocabulary:

| OpenCode 2 source                                  | Normalized turn                                                               |
| -------------------------------------------------- | ----------------------------------------------------------------------------- |
| User text, synthetic and queued messages           | `env`                                                                         |
| Root assistant text                                | `asst`                                                                        |
| Reasoning                                          | `thinking`, redacted when only presence is available                          |
| Tool parts (`shell`, `read`, `write`, `subagent`…) | `tool`, retaining structured input, call id, output, error, and truncation    |
| `compaction` messages                              | A compaction boundary carrying its summary, never re-admitting pruned history |
| `idle` marker messages                             | Not a turn; consumed as a turn boundary                                       |
| Child sessions                                     | Sidechain conversation attached to the `subagent` call that spawned it        |

A child is attached to its spawning call through the session id the tool part records. The normalizer never manufactures
a parent link: a child with a `parentID` but no linked call becomes an unlinked sidechain of its root.

## Usage and cost

Turn usage is the root session's per-step usage for the turn's messages, read from the API, plus every descendant
session's usage over the same turn, because root totals exclude children. Each child's spend is attributed to the turn
whose `subagent` call spawned it, and recorded with the child's own model and variant. Cost is recorded as reported; a
cost of zero under subscription authentication is unknown, not free, unless the proof shows the 2.x line reports real
subscription cost, in which case the corpus records it and zero means zero.

Interrupted-invocation recovery reads the same per-step records, so a lease killed mid-turn still accounts for every
completed step. Price estimates for unknown cost come from OpenCode's models cache in the line-2 store's cache
directory.

Every transcript segment and usage record carries harness id, line, harness version, model, variant, and normalizer
version as provenance.
