# Structure of `milestones.md`

The required shape of the milestone registry. A review of `milestones.md` asserts its contents against this guide.

## Purpose

The top-down origin of the registry: each milestone declares a destination in the user's terms and demands the epics that reach it. Thinking flows milestone → epics, never the reverse.

## Required shape

1. A short intro paragraph in the product voice — what a milestone is, no governance.
2. A summary table of every live milestone. Columns: `Milestone | What users will be able to do`.
3. One `## \`milestone:<slug>\` — <short title>` section per milestone, in the same order as the table: a prose breakdown — the destination, what it unlocks (told through the charter's personas where that sharpens it), and the work it demands — followed by the milestone's **epic chart**: the work it requires as a table with columns `Epic | Slice | Status`.

## Row and section rules

- The Milestone cell carries the stable `milestone:<slug>` id; a live milestone's row is its id's single home, and renaming or removing an id is a breaking change.
- The "What users will be able to do" cell is written in the user's terms — a capability a person gains, never an implementation description. A milestone is *reached*; an epic is *built*.
- The epic chart is the single home of the milestone's decomposition — the summary table never repeats it. Status is one of `horizon`, `in flight`, or `delivered`, matching where the epic's slice actually stands.
- Every epic in a chart resolves to a row in `epics.md` or `delivered.md` — demanding new work and creating its epic row happen in the same change, never a dangling reference.
- An epic may appear in more than one milestone's chart.
- Every summary-table row has its section and vice versa.

## Lifecycle

A milestone enters when a destination is declared — before its epics exist, whose rows it creates. When the destination is reached, the milestone leaves this file for its own section in `delivered.md`, carrying the grid of epic slices that got it there.
