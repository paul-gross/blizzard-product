# OpenCode 2 transcript specification

The OpenCode 2 transcript source implements the harness transcript seam and feeds the transcript lane owned by
`epic:transcripts`. It reads the server API's documented surfaces, never OpenCode's SQLite schema.

## Reads

Transcripts are read while a turn runs and long after it ends, by the transcript pump, usage recovery, and context
sampling. Every read goes to the operator's service through the same API the driver uses, and only through its read
operations, so a read can never start or continue a turn. Reads apply the rule resume does: a session its runner did not
mint reads as `not_found` on every operation, so a 1.x session OpenCode 2 migrated into the store is never read through
the 2.x source, and segments from two normalizers never mix in one session. A cursor the OpenCode 2 source did not mint
is rejected as `unreadable`.

## Forward reads

`turns_since` reads `GET /api/session/{id}/message?order=asc`, paging with the API's own opaque `cursor.next`. The
source's cursor stores that API cursor together with the ids of every message it admitted while still in progress. A
cold read returns the bounded retained conversation; a forward read returns the messages after the stored API cursor,
then re-reads each still-unsettled message by id (`GET /api/session/{id}/message/{messageID}`). Messages are rewritten
in place as they complete and never move, so a forward read alone would never see a completed state; the re-read is what
produces the output patch that finishes an already-shipped call. A message is unsettled while any tool part in it reads
`streaming` or `running`.

The `prt_` part ids that `run --format json` prints are client-synthesized and absent from the API, so nothing keys on
them. Message ids are stable; within a message, parts are ordered by position.

`size_bytes` measures the serialized root conversation the source reads, excluding child sessions. `context_tokens`
reads the newest root assistant step's input, cache read, and cache write tokens, skipping `compaction` and `idle`
marker messages. `GET /api/session/{id}/context` returns the messages after the last compaction, not a token count, and
is not used for either.

A raw line is one source record: one API message, of the root or of a descendant, serialized with its session id, in the
order the source admitted it. `read_raw_lines(start, end)` returns the records between two positions, and
`sum_transcript_usage` and interrupted-turn recovery sum the step usage those records carry, descendants included, so
one session's range accounts for its children's spend. `tail_position` returns the position after the last record
admitted so far: the stored API cursor and unsettled-message ids for the root and for each descendant.

Every read is bounded, timeout-limited, and safe against the service writing the session concurrently. A missing session
returns `not_found`, which the API distinguishes from failure, unlike 1.x export; any other failure returns `unreadable`
and logs once at the source boundary.

## Normalization

The OpenCode 2 normalizer stamps `opencode-v2/<normalizer-version>`, distinct from OpenCode 1's `opencode-export/1`, and
maps into Blizzard's existing vocabulary:

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

Turn usage is the root session's per-step usage for the turn's messages plus every descendant's usage over the same
turn, read separately because root totals exclude children. The driver's result carries both for a completed turn; the
transcript source recomputes them for an interrupted one, so a turn killed mid-flight still accounts for every completed
step. Each child's spend is attributed to the turn whose `subagent` call spawned it, with the child's own model.

Cost is recorded as reported. A cost of zero under subscription authentication is unknown, not free, unless the corpus
shows OpenCode 2 reports real subscription cost, in which case zero means zero. Where a step reports no cost, an
estimate comes from the per-model prices `GET /api/model` serves, when the proof shows it carries them; otherwise cost
stays unknown.

Usage and transcript records carry the provenance fields that already exist: harness id, harness version, model, effort,
and normalizer version. The major version is derived from the harness version and the variant is recorded as effort, so
no field is added.
