# Structure of `epics.md`

The required shape of the epic registry. A review of `epics.md` asserts its contents against this guide.

## Purpose

The registry of live epics and the place plan priority is decided: the order of the rows is the order of the plans behind them. It is a reference list of capability areas, not a status board.

## Required shape

1. A short intro paragraph in the product voice — what the registry is, no governance.
2. A single priority-ordered table of every live epic. Columns: `Epic | Capability | Notes`. The top rows are the highest priority — next in line to be promoted to a plan.

There is **no status** in this file. Whether an epic's work is filed, started, or ongoing is not tracked here — that state lives in the milestone epic charts (`milestones.md`) and in GitHub, and a fully-landed slice resolves in `delivered.md`. The registry never carries issue numbers, and it has no `Horizon`/`In flight` split — a single flat, priority-ordered table is the whole shape.

## Row rules

- The Epic cell carries the stable `epic:<slug>` id, with a slice qualifier in parentheses when the row means one slice (`` `epic:security` (worker-lockdown slice) ``).
- A live epic's row is its id's single home; renaming or removing an id is a breaking change (`canon:one-owner` applies to ids).
- Once its plan exists, the row carries the repo's only deep link to that plan — everything else cites the id, so a plan can change shape with a one-line row edit.
- An epic is a capability area, not a version: slices land separately, and a slice-scoped row names its slice in the Epic cell's parenthetical.
- The Notes cell holds durable reference only — plan links, cross-epic dependencies, and scope or priority intent (e.g. parked). It never states execution status ("shipped", "filed", "in flight", "landed") and never cites issue numbers.
- An epic either appears in at least one milestone's epic chart or stands on stated operational necessity — a row with neither is scope to challenge.
- Row ordering is meaningful: it is the priority of the plans. Reorder deliberately, never incidentally.

## Lifecycle

A row enters when a milestone demands the work (or operational necessity does), and leaves for `delivered.md` when its work fully lands — a partially-delivered epic keeps its row for the slice that remains, its Epic cell re-scoped to name it. Filing the epic's issues does not move or annotate the row: the row stays put and GitHub tracks the work. The cross-file integrity rules — every plan accounted for, every cited epic resolving to a row — live in [the structure hub](./index.md).
