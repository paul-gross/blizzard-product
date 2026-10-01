# Row contract

## The unit: one derivation

The hub derives a transcript segment's events as a whole. One derivation is the set of `transcript_events` rows for one
`(segment_id, extractor_version)`, together with its `transcript_event_derivations` marker. Re-deriving a segment
deletes that set and writes a new one with a new `derived_at`. The export writes derivations whole, so a reader can
always tell which copy of a segment's events is the newest.

A derivation is identified by `derivation_id`: the first 16 bytes of
`SHA-256("blizzard-derivation/v1/" + segment_id + "/" + extractor_version + "/" + derived_at)`, as hex. The same
derivation exported twice carries the same id.

## Dataset: `events`

One dataset holds three record types, told apart by `record_type`, so that a derivation and its events land in the same
file and the same pass.

| `record_type` | Written                                           | Says                                                                    |
| ------------- | ------------------------------------------------- | ----------------------------------------------------------------------- |
| `derivation`  | once per exported derivation, empty ones included | this segment's events, under this extractor version, are now these      |
| `event`       | once per event in the derivation                  | one file read, skill invocation or agent spawn                          |
| `dropped`     | once when a segment stops counting                | every derivation of this segment, under every version, no longer counts |

A derivation with no events still writes its `derivation` row. Without it, a segment re-derived to nothing would leave
its older events standing.

| Column                          | Type   | On                | Null | Meaning                                                                                    |
| ------------------------------- | ------ | ----------------- | ---- | ------------------------------------------------------------------------------------------ |
| `record_type`                   | string | all               | no   | `derivation`, `event` or `dropped`                                                         |
| `segment_id`                    | string | all               | no   | The transcript segment                                                                     |
| `extractor_version`             | string | derivation, event | no   | e.g. `blizzard-analytics/4`                                                                |
| `derivation_id`                 | string | derivation, event | no   | Above                                                                                      |
| `derived_at`                    | time   | derivation, event | no   | When the hub derived it                                                                    |
| `complete`                      | bool   | derivation        | no   | The marker's own flag: false when the derivation stopped short                             |
| `event_count`                   | int    | derivation        | no   | How many `event` rows the derivation holds                                                 |
| `dropped_at`                    | time   | dropped           | no   | When the hub dropped the segment's events                                                  |
| `kind`                          | string | event             | no   | `file_read`, `skill_invocation` or `agent_spawn`; extensible                               |
| `subject`                       | string | event             | yes  | The path, the skill name, or the spawned agent type; null when the extractor names none    |
| `tool`                          | string | event             | yes  | The tool the turn called                                                                   |
| `turn_path`, `occurrence`       | —      | event             | no   | The event's place in the segment                                                           |
| `occurred_at`                   | time   | event             | yes  | The turn's own time, when the transcript carries one                                       |
| `depth`                         | int    | event             | no   | 0 for the main conversation, plus one per subagent nesting                                 |
| `agent_type`                    | string | event             | yes  | The nearest enclosing subagent's type; null at depth 0                                     |
| `step_key`, `trace_id`          | string | all               | no   | The runner step the segment came from, by chunk and epoch, as the steps slice derives them |
| `step_started_at`               | time   | all               | no   | When that step started, by the span contract; the row's partition and backfill time        |
| `chunk_id`, `epoch`             | —      | all               | no   | The segment's chunk and epoch                                                              |
| `spawn_generation`              | int    | all               | no   | Which worker spawn on the lease produced the segment                                       |
| `graph_id`, `graph_name`        | string | all               | no   | Where the step stood                                                                       |
| `node_id`, `node_name`          | string | all               | no   | The node                                                                                   |
| `harness_id`, `harness_version` | string | event             | yes  | As derived                                                                                 |
| `model`, `effort`               | string | event             | yes  | As derived                                                                                 |
| `exported_at`                   | time   | all               | no   | When this copy of the row was written                                                      |

An event's identity is its `derivation_id`, `kind`, `turn_path` and `occurrence`, the store's own natural key under one
derivation. One turn can yield events of different kinds at the same place, so a reader that drops `kind` from the key
loses rows.

Every step column comes from the step summary the steps slice builds rows from, looked up by `step_key`. The graph the
store stamps on a derived event follows a different rule, the newest transition into the node or else the chunk's mint
pin, and on a migrated chunk the two can disagree. The stored stamp is never exported, so an event names the same graph
and node as its step row.

The event's `payload` column is not exported. Every field a reader needs from it is a named column above, and a payload
that grows a field in a later extractor must not carry it out unreviewed.

## Current truth

An `event` row counts when both of these hold:

- its `derivation_id` is that of the newest `derivation` row for its `segment_id`, by `derived_at`, whatever its
  extractor version
- no `dropped` row for its `segment_id` has a `dropped_at` later than that derivation's `derived_at`

A segment dropped and later derived again counts again, because the new derivation is newer than the drop. The data
dictionary ships this as a SQL view over the dataset, in a dialect DuckDB and the common warehouses accept, and the
proof uses it unchanged.

Taking the newest derivation regardless of version is what lets the view ride through an extractor upgrade untouched.
The hub only ever derives under its current version, so a re-derived segment's newest derivation is the new version's,
and a segment the sweep has not reached yet keeps counting under the old one. Totals never dip during the burst, and the
view never has to be edited to name a version. Each manifest records the hub's current extractor version, so a reader
can tell when every segment has caught up.

A reader comparing extractors uses the dictionary's second view instead: the newest derivation per
`(segment_id, extractor_version)`, filtered to one version.

## Shared conventions

Times, lists, partitions and duplicate handling follow the steps slice's [row contract](../../steps/spec/rows.md)
§Shared conventions. An `events` row's partition date is the UTC date of `step_started_at`, the time of the work it
describes, not of when the hub last derived it. An extractor upgrade re-derives every segment at once, and partitioning
by derivation time would land the fleet's whole history in that one day. A segment belongs to one step, so one
derivation never splits across partitions, and a re-derivation lands beside the copy it supersedes, in a new file.

## What never leaves

No row carries:

- transcript content: a prompt, a tool's input or output, a model's reply
- the event's raw `payload`
- a segment's bytes, or its rejection reason

## File paths

The extractor stores a file read's path as the tool was given it, which is usually absolute: it names the machine's
user, the worktree's location, and a different string for the same file on every runner. None of that is the event's
point, and the last of it defeats the question the slice exists for. `[egress] file_paths` decides what leaves as a
`file_read` event's `subject`:

- **`relative`** (the default). The path relative to the worker's working directory, so the same file reads the same
  from every runner and worktree. The runner records that directory on each segment when it opens it, in a new nullable
  column frozen like the segment's model, with no backfill. A path outside it, or on a segment recorded without one,
  leaves hashed.
- **`hashed`.** A keyed hash, HMAC-SHA-256 under a key named by an environment variable the operator sets. Equal paths
  still group, and nobody holding the export can guess a path back from a list of likely names.
- **`absolute`.** The path exactly as stored, for an operator who wants it and says so.
- **`omit`.** `subject` is null on every `file_read` event.

The working directory itself never leaves. Even relative, a path names the repository's own layout, so the export
directory deserves the same care as the `transcript:read` permission, and the operator documentation says so.
