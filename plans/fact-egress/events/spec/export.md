# Export contract

The `events` dataset joins the exporter the steps slice builds. Its [export contract](../../steps/spec/export.md) owns
the shared discipline: the sweep, settle window, directory layout, immutable files, manifests, formats, delivery,
configuration and operator surface. This file states only what the events dataset adds.

## What a pass reads

A pass reads two sources after the dataset's cursor, settled by the same window:

- **Derivation markers.** Each `transcript_event_derivations` row whose `derived_at` is past the cursor, with its
  events, written as one `derivation` row and its `event` rows. A re-derivation replaces the marker with a later
  `derived_at`, so the pass reaches it again on its own.
- **Drops.** Each `transcript_event_drops` row past the cursor, written as one `dropped` row.

The cursor is ordered by:

1. the source's time: `derived_at` or `dropped_at`
2. then `segment_id`
3. then `extractor_version`, empty for a drop

A derivation is written whole. A batch limit that falls inside one derivation's events extends to the end of that
derivation, so no file holds part of one.

A marker that is replaced again before a pass reaches it is simply missed in its earlier form. Only the newest
derivation of a segment is guaranteed to leave, which is the one current truth counts.

## The drop fact this slice adds

The derivation sweep deletes a segment's events and markers, for every extractor version, when the segment stops being
visible (`drop_segments`). That leaves nothing behind for the export to read. This slice adds one append-only fact,
`transcript_event_drops`, holding the segment's id, chunk, epoch and spawn generation, and `dropped_at`, so a `dropped`
row carries its step without reading anything the drop removed. It is written in the same transaction as the deletion,
whether or not the export is enabled, because a drop that happened while the export was off still has to reach the
directory once it is turned on and backfilled.

Re-deriving a segment through `replace_segment_events` writes no drop. The new marker supersedes the old derivation by
itself.

## Extractor versions

`[egress] extractor_versions` is `current` (the default) or `all`:

- **`current`.** Only derivations under the hub's current `EXTRACTOR_VERSION` are written. When a deploy bumps the
  version, the derivation sweep re-derives every segment under the new one, and the export writes them as they come,
  bounded by the batch limit like any other backlog. Drops are written whatever the setting.
- **`all`.** Every version's derivations are written.

## Backfill

`blizzard hub egress backfill --dataset events` selects by the step's start, the same time the rows are partitioned by:
every derivation in the store, and every drop, whose segment's step started in the window. It does not select by
`derived_at`. An extractor upgrade re-derives every segment at one moment, so a window over derivation time would find
nothing before the upgrade and all of history after it, and a segment last derived between the window and the moment the
export was turned on would reach the directory by neither path.

A derivation already replaced is not in the store and is not backfilled: history before the export began is available
only in its current form. A backfill that overlaps the live export writes derivations the export already wrote, under
the same `derivation_id`, and the duplicate rule absorbs them.

## Manifests

Each manifest a pass writes records the hub's current `EXTRACTOR_VERSION`, beside the shared fields. A reader watching
an upgrade compares it with the versions in the current-truth view to see how far the re-derivation has reached.

## File paths

`[egress] file_paths` is `relative`, `hashed`, `absolute` or `omit` (rows.md §File paths), and `[egress] path_key_env`
names the environment variable holding the hash key. A `hashed` setting, or a `relative` one that falls back to hashing,
with no key set does not stop the hub. The export stays off and records one `egress-config-rejected` event naming the
missing key.

The runner's half is one new nullable column on transcript segments, the worker's working directory, set when the
segment opens. The hub stores it, never shows it, and the export uses it only to make paths relative.

## The published shape

The `events` dataset joins `contracts/egress/` with its own golden output and `_schema/events.v1.json`. The
current-truth view is part of the contract, tested against the golden output.

`docs/deployment/egress.md` gains:

- the events section of the data dictionary
- the three record types, and why a derivation is written whole
- the current-truth view, ready to paste
- the extractor-version setting, the burst a version bump causes, and reading the manifest's version during it
- the file-path setting, and the care the directory deserves even with relative paths

## Verification

- **Unit.** Row assembly over fixtures for a derivation with events, an empty derivation, a sidechain event at depth
  two, and a drop. `derivation_id` is pinned by known vectors.
- **Service.** Against a real temporary directory:
  - a re-derived segment writes a newer derivation, and the view shows only its events
  - a segment re-derived to nothing writes an empty derivation, and the view shows none of its old events
  - a dropped segment writes a `dropped` row, and the view shows none of its events; derived again, it counts again
  - `drop_segments` writes `transcript_event_drops` in its own transaction, export enabled or not
  - a batch limit inside a derivation still writes the derivation whole
  - with `current`, a version bump writes the new version's derivations and none of the old, and the view's totals do
    not dip while the sweep works through the backlog
  - a turn with events of two kinds at one place exports both, and the view counts both
  - a re-derivation and a drop land in the partition of their step's start, not of their own time
  - a backfill by step start after an upgrade writes every segment's current derivation, including one last derived
    between the window and enabling the export
  - an event on a migrated chunk names its step row's graph, not the stored stamp
  - each `file_paths` setting, a path outside the working directory, a segment with no working directory, and a missing
    hash key
  - no row carries a planted prompt, tool input or payload field
- **End to end.** A `blizzard-mock` run with file reads, skills and agent spawns, then an extended transcript, a
  `re-derive`, and a superseded segment.
- **Proof.** DuckDB and a warehouse-backed BI tool, using only the dictionary and the shipped view, report the same
  current counts as `blizzard hub analytics summary` for files, skills, agent types and nodes.
