# Export contract

## Where it runs

The export is a hub sweep, yielded by `Sweep.all` beside the other reconcilers, and only when a directory is configured.
It follows the shape the fleet-spans sweep follows:

- **Assembly is pure.** One domain function turns one closed step's summary from the shared step module into its `steps`
  row, and another turns one usage fact into its `invocations` row, with no I/O (`bzh:domain-core`). The row functions
  only rename and reformat. Every value with a meaning comes from the summary.
- **Reads go through a repository.** Closed steps and usage facts after each cursor are read through a read repository.
- **Writes go through a seam.** An `IEgressWriter` protocol takes a dataset's rows and commits them as a file. It has an
  NDJSON binding, a Parquet binding, and an in-memory binding for tests (`bzh:pluggable-seams`).

Nothing in the fleet's path calls the writer.

## What a pass does

Each dataset has its own cursor. For each dataset, a pass:

1. **Selects** what closed after the cursor and at least the settle window before now, in cursor order, up to a batch
   limit.
2. **Assembles** each row.
3. **Writes** the rows as one file per date partition they fall in.
4. **Records** a manifest of the files it committed.
5. **Advances** the cursor to the last row written, only after every file and the manifest are in place.

The settle window (default five minutes) covers the same lags it covers for fleet spans: usage landing after its step
closed, and facts whose recorded time precedes their commit.

| Dataset       | Cursor order                                                          |
| ------------- | --------------------------------------------------------------------- |
| `steps`       | the closing fact's recorded time, then `chunk_id`, epoch, decision id |
| `invocations` | `recorded_at`, then `usage_facts.id`                                  |

Each pass that advances a cursor appends one fact row holding the dataset, the position, the row count and the files
written.

## Late usage

Usage can land after its step was written: a runner that stores and forwards can deliver it hours late. The `steps`
dataset therefore keeps a second position beside its cursor, over usage facts in the `invocations` cursor order. Each
pass reads the usage recorded past that position, settled by the same window. Every one whose step closed at or before
the `steps` cursor marks that step to be written again. The pass reassembles each marked step from the record as it
stands and writes it beside its new rows, then advances both positions together. The newer copy wins by the duplicate
rule (rows.md §Shared conventions), so a reader's step totals and invocation totals agree once both datasets have caught
up.

Only usage triggers a rewrite. A step row is otherwise final once written, as a step's span is final once told.

## The directory

```text
<directory>/
  steps/v1/date=2026-10-01/steps-20261001T061500Z-k7q2-000042.parquet
  invocations/v1/date=2026-10-01/invocations-20261001T061500Z-k7q2-000042.parquet
  _manifests/20261001T061500Z-k7q2-000042.json
  _schema/steps.v1.json
  _schema/invocations.v1.json
  .staging/
```

- **Layout.** Each dataset has its own tree, under its contract's major version, partitioned by the UTC date of the
  row's own time (`ended_at`, `recorded_at`). A backfilled row lands in its own date's partition, in a new file.
- **Names.** A file is named by dataset, the pass's start time, a token minted when the hub process starts, and a pass
  sequence number, so names sort in the order they were written and stay unique even when two hub processes share the
  directory across a redeploy. A backfill's files carry `backfill` in the name.
- **Files are immutable.** A file is written under `.staging/`, flushed to disk, then placed under its final name by an
  operation that fails rather than replace an existing file. A reader never sees a half-written file, and the exporter
  never opens a placed file again. A name that already exists fails the pass rather than overwriting it. Nothing in the
  directory is ever deleted or overwritten by blizzard; pruning it is the operator's call.
- **Manifests.** A pass writes its manifest last: every file it placed, with dataset, row count, cursor range, contract
  version and SHA-256. A loader that reads only files named in a manifest never sees a partial pass.
- **Schemas.** `_schema/` holds each dataset's column list, types, nullability and meanings, generated from the
  contract. Parquet carries its schema in the file as well; NDJSON relies on this.
- **Size.** A file holds at most `max_rows_per_file` rows. A pass with more rows for one partition writes several.

## Formats

`format` is `ndjson` or `parquet`, and one export writes one format.

- **`ndjson`** is the default and needs nothing new. Files are gzip-compressed, `.ndjson.gz`.
- **`parquet`** needs `pyarrow`, which comes only with the `blizzard[egress]` install extra. Files are zstd-compressed.

A hub configured for Parquet without the extra installed does not fail to start. The export stays off, the hub records
one `egress-config-rejected` event naming the missing extra, and `egress status` reports it.

## Delivery semantics

Delivery is at least once. A hub killed after placing a file but before writing its cursor writes the same rows again on
restart, in a new file, identical but for `exported_at`. Readers keep the newest copy of each identity (rows.md §Shared
conventions).

A failed write — a missing directory, a permission error, a full disk — leaves the cursor where it is and the staged
file removed. The pass retries on later ticks with exponential backoff up to a cap.

**The export never fills the hub's disk.** Before writing, a pass checks the free space on the directory's filesystem.
Below `min_free_bytes` (default 1 GiB) it writes nothing and records the shortfall. The directory may share a disk with
the hub's own store, and a fleet that stops because its export filled the disk would break the epic's first promise.

When an export is turned off and on again, its cursor resumes where it stopped, so the directory holds no gap. An
operator who would rather skip the gap moves the cursor with `egress reset`.

Three new `event_log` kinds record these moments, routed through `foundation/event_log.py` and `domain/operations.md`
§Event kinds:

| Kind                     | Severity | When                                                                     |
| ------------------------ | -------- | ------------------------------------------------------------------------ |
| `egress-write-failed`    | warning  | the first failed pass after a success, with the cause, low disk included |
| `egress-write-recovered` | info     | the first successful pass after a failure                                |
| `egress-config-rejected` | warning  | the export was configured in a way the hub cannot honor                  |

## Configuration

The export is enabled when `[egress] directory` is set in `blizzard-hub.toml`. With it unset, no sweep starts, no cursor
row is written, and the hub behaves as it does today. Every other key has a default that works unset:

- `format`: `ndjson` or `parquet`
- `datasets`: which to export, default all that exist
- `sweep_seconds`, `settle_seconds` and `batch_limit`
- `max_rows_per_file`
- `min_free_bytes`
- `backfill_max_window`

A directory that does not exist, or that the hub cannot write, fails the first pass rather than startup, and records
`egress-write-failed`.

## Operator surface

Each command is a hub API route behind operator auth, with a CLI verb that is a pure client of it.

`blizzard hub egress status` reports:

- whether the export is on, its directory and format, and any rejected setting
- each dataset's cursor position and lag
- the last pass, the last file written, and the last error
- the free space on the directory's filesystem

`blizzard hub egress backfill --since <t> --until <t> [--dataset <name>] [--dry-run]` writes every row in the window:

- **Same rows.** Backfilled rows are assembled exactly as live ones are, from the record as it stands now.
- **Cursor untouched.** It never moves a live cursor.
- **Bounded.** It refuses a window larger than `backfill_max_window`.
- **Dry run.** `--dry-run` reports row and file counts and writes nothing.

`blizzard hub egress reset --dataset <name> --to <t>` moves one dataset's cursor, recording the skipped or repeated
window in the event log.

## The published shape

The datasets' columns, types, nullability, meanings and enumerated values form a versioned contract under
`contracts/egress/`, beside the other wire contracts. It holds golden output for a seeded scenario, and `_schema/`, the
docs and the tests are all generated from or checked against it.

- **Additive change.** A new nullable column, or a new value of an enumerated column, is additive and keeps the major
  version.
- **Breaking change.** Removing, renaming or retyping a column, or changing a meaning, writes a new major version beside
  the old (`steps/v2/`), with both written for a deprecation period that `docs/versioning.md` governs.

`docs/deployment/egress.md` covers:

- turning the export on, and the formats
- the data dictionary, generated from the contract
- the directory layout, manifests and how to load only whole passes
- keeping the newest copy of each row, with the SQL view
- recipes for DuckDB over the directory, `rclone` to object storage, and a stock warehouse loader
- backfill and reset
- what never leaves

## Verification

- **Unit.** Row assembly over fact fixtures, reusing the fleet-spans fixtures so the two exports are checked against the
  same steps:
  - each step kind and each outcome, a gate resolved long before its pickup, a migration, a restart
  - names resolved from the pinned graph after the chunk has been repinned
  - billed, estimated, partial and missing cost, folded as `spend.md` defines
  - an invocation whose usage landed after its step was exported, and an invocation whose step is still open
- **Service.** These are tested against the in-memory writer and a real temporary directory:
  - each cursor advances only after its files and manifest are placed, in total order
  - a kill between placing a file and writing the cursor writes identical rows again
  - usage landing after its step was written writes the step again with corrected totals, and the newest copy's totals
    equal the sum of its `invocations` rows
  - two hub processes writing one directory never overwrite each other's files, and an existing name fails the pass
  - no partial file is ever visible outside `.staging/`
  - low free space writes nothing and records once; recovery records once
  - Parquet without the extra leaves the hub running with the export off and one `egress-config-rejected` event
  - no directory configured starts no sweep
  - backfill writes the same rows the live export did and leaves the cursors alone
  - the planted-content scan finds nothing in any file
- **Contract.** The golden scenario's files match `contracts/egress/`, and `_schema/` and the docs are generated from
  it.
- **End to end.** A `blizzard-mock` night of builds, a review bounce, a gate, an ask, a delivery conflict and an
  escalation is exported in both formats.
- **Proof.** DuckDB over the directory, and a stock loader into a warehouse read by a BI tool, each answer cost by node
  by day and the slowest station of the week, using only the data dictionary.
