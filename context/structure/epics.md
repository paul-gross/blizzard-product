# Structure of `epics.md`

The required shape of the epic registry. A review of `epics.md` asserts its contents against this guide.

## Purpose

The registry of live initiatives and epics and the place plan priority is decided: the order of the rows is the order of
the plans behind them. It is a reference list of capability areas, not a status board.

## Required shape

1. A short intro paragraph in the product voice — what the registry is, no governance.
2. A single priority-ordered table with one row per live initiative and one per live epic that belongs to no initiative
   — standalone, or in a plain grouping folder. Columns: `Id | Capability | Notes`. The top rows are the highest
   priority — next in line to be promoted to a plan. The order of an initiative's own epics is its plan's `epics`
   frontmatter, never extra rows here.

There is **no status** in this file. Whether an epic's work is filed, started, or ongoing is not tracked here — each
epic's status lives in its own plan's frontmatter ([structure/plans.md](./plans.md)), repeated in the milestone epic
charts, with issue-level progress in GitHub, and a fully-landed epic resolves in `delivered.md`. The registry never
carries issue numbers, and it has no `Horizon`/`In flight` split — a single flat, priority-ordered table is the whole
shape.

## Row rules

- The Id cell carries the stable `initiative:<slug>` or `epic:<id>` id and nothing else — which epics of an initiative
  remain is its plan's frontmatter, never a qualifier on the row.
- A live row is its id's single home; renaming or removing an id is a breaking change (`canon:one-owner` applies to
  ids). An initiative's row is also the home of the ids of the epics it names.
- The row carries the repo's only deep link to its plan — for a folder, its `index.md` — never to one of an initiative's
  epics, which the folder routes to in turn. Everything else cites the id, so a plan can change shape with a one-line
  row edit.
- An initiative's row describes the capability as a whole: its epics land separately, and the row outlives each one.
- The Notes cell holds durable reference only — plan links, cross-epic dependencies, and scope or priority intent (e.g.
  parked). It never states execution status ("shipped", "filed", "in flight", "landed") and never cites issue numbers.
- A row's epic, or at least one of an initiative's epics, appears in a milestone's epic chart, or the row stands on
  stated operational necessity — a row with neither is scope to challenge.
- Row ordering is meaningful: it is the priority of the plans. Reorder deliberately, never incidentally.

## Lifecycle

A row enters when a milestone demands the work (or operational necessity does), together with its plan, and leaves for
`delivered.md` when its work fully lands — a partially-delivered initiative keeps its row, its capability re-scoped to
the work that remains. Filing an epic's issues does not move or annotate the row: the row stays put and GitHub tracks
the work. The cross-file integrity rules — every plan accounted for, every cited id resolving to a row — live in
[the structure hub](./index.md).
