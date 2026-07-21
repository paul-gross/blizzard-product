# Structure of `delivered.md`

The required shape of the ledger of landed work. A review of `delivered.md` asserts its contents against this guide.

## Purpose

The record of what has made it into the product, organized the way the work was conceived: by destination.

## Required shape

1. A short intro paragraph in the product voice — what the ledger is, no governance.
2. One `## \`milestone:<slug>\` — <short title>` section per **reached** milestone: a destination sentence, then a grid of the epic slices that carried it. Columns: `Epic | Slice delivered | Notes`. A milestone still being worked toward never has a section here — it lives in `milestones.md`, where a slice already landed is marked `delivered` in its epic chart.
3. When work has landed in service of no milestone, a closing `## Outside the milestones` section in the same grid shape holds it. Absent when everything delivered has a destination.

## Entry rules

- Entries are records, not registry rows: an epic that served several milestones appears under each of them, and an id that has fully landed *resolves* here (the live registries stay the single home for anything still in flight).
- A Notes cell states what the slice comprises and, where a further slice remains live, links the remainder to the epic registry.
- An entry keeps its link to the plan it was built from.
- The ledger is append-mostly: entries arrive when work lands and are never reshaped retroactively — the record is the point (`canon:no-retro`).

## Lifecycle

An epic slice completes and leaves `epics.md`. If every milestone it served is still live, its delivery is marked in those milestones' epic charts and it waits; it enters this ledger under a milestone's section when that destination is reached (or under `## Outside the milestones` immediately, if it served none). When a destination is reached, its row leaves `milestones.md` and its section here — carrying every slice that got it there — stands as the complete record.
