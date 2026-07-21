# Structure of `epics.md`

The required shape of the epic registry. A review of `epics.md` asserts its contents against this guide.

## Purpose

The registry of live epics and the place plan priority is decided: the order of the rows is the order of the plans behind them.

## Required shape

1. A short intro paragraph in the product voice — what the registry is, no governance.
2. `## Horizon` — a priority-ordered table of epics with no issues filed. The top rows are next in line to be promoted to a plan. Columns: `Epic | Capability | Notes`.
3. `## In flight` — a table of epics whose issues are filed and work is ongoing. Columns: `Epic | Slice / scope | Issues`, with issue numbers as plain text — never links out of the repo.

## Row rules

- The Epic cell carries the stable `epic:<slug>` id, with a slice qualifier in parentheses when the row means one slice (`` `epic:security` (worker-lockdown slice) ``).
- A live epic's row is its id's single home; renaming or removing an id is a breaking change (`canon:one-owner` applies to ids).
- Once its plan exists, the row carries the repo's only deep link to that plan — everything else cites the id, so a plan can change shape with a one-line row edit.
- An epic is a capability area, not a version: slices land separately, and a row's status names the slice it means.
- An epic either appears in at least one milestone's epic chart or stands on stated operational necessity — a row with neither is scope to challenge.
- Row ordering within Horizon is meaningful: it is the priority of the plans. Reorder deliberately, never incidentally.

## Lifecycle

A row enters when a milestone demands the work (or operational necessity does), moves Horizon → In flight when its issues are filed, and leaves for `delivered.md` when its slice completes. The cross-file integrity rules — every plan accounted for, every cited epic resolving to a row — live in [the structure hub](./index.md).
