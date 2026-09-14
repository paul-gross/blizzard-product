# Repository structure

What lives where in `blizzard-product`, and which folders are allowed to exist. A file that fits none of these rows
belongs somewhere else — most likely in a GitHub issue (work that is ready), or nowhere
(`winter-canon:/organization.md`, `canon:admission-test`).

The registry files and the epic plans each have a structural guide beside this hub — the required shape a review asserts
their contents against. Read the one guide for the file you are authoring or reviewing:

| File            | Structural guide                           |
| --------------- | ------------------------------------------ |
| `epics.md`      | [structure/epics.md](./epics.md)           |
| `milestones.md` | [structure/milestones.md](./milestones.md) |
| `delivered.md`  | [structure/delivered.md](./delivered.md)   |
| Epic plans      | [structure/plans.md](./plans.md)           |

## The layout

| Path                      | What belongs there                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `charter/`                | The standing statement everything else answers to: [mission.md](../../charter/mission.md), [vision.md](../../charter/vision.md), and [personas.md](../../charter/personas.md) with its per-persona cards in `charter/personas/<persona-slug>.md`. The single home of every `persona:<slug>` id. Barely moves; amend it before making a change that contradicts it.                                                                                                                                                                                                                                                                                                                                 |
| `epics.md`                | The epic registry and the priority order of the plans: a single priority-ordered table of capability areas, carrying no execution status — that lives in each epic plan's frontmatter. An epic's id lives on its row here until the work completes.                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| `milestones.md`           | The milestone registry: what users will be able to do. Each milestone demands the epics that reach it (an epic may serve several), and its id lives on its row here until the destination is reached.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `delivered.md`            | The ledger of landed work, organized by destination: a section per delivered milestone with the grid of epic slices that carried it, and a closing section for work landed outside any milestone. A completed id resolves here.                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `plans/`                  | What to build: one epic plan per epic with work still live, the slice plans beneath it, and single-story plans. The epic plan's frontmatter is the per-epic view — its refinement and the status of each named slice — and its shape is [structure/plans.md](./plans.md). A lean plan is a solo `plans/<slug>.md`; a plan with slice plans, mocks, or supporting pieces is a folder `plans/<slug>/` whose `index.md` is the progressive-disclosure base routing to everything inside (`canon:hub-and-spoke`). A slice plan is written when its slice comes within striking distance, cited by the issues that decompose it, and frozen once shipped — what *is* built lives elsewhere, never here. |
| `delivered/`              | The frozen plans of fully-delivered epics, each keeping the shape it had under `plans/`. An epic's plan — its file, or its folder with every slice plan, spec, and artifact — moves here in the change that settles its last slice, and never moves back. [delivered.md](../../delivered.md) is the ledger over it.                                                                                                                                                                                                                                                                                                                                                                                |
| `plans/<slug>/spec/`      | The highest-level technical contracts behind a plan — an epic plan or a slice plan folder — whose product intent and implementation mechanics serve different readers. The plan's `index.md` routes implementers into the spec hub; the hub routes one coherent contract per leaf. Specifications state the architecture to build and freeze with the plan, never implementation progress or a decision log.                                                                                                                                                                                                                                                                                       |
| `plans/<slug>/artifacts/` | Everything a plan carries that is not markdown — HTML mocks, code proofs-of-concept, images, whatever the plan needs to show rather than tell. One folder for all of it, routed from the plan's `index.md`, so the plan folder itself stays a readable markdown surface. An epic keeps one at the top of its folder, shared by its slices — never one per slice.                                                                                                                                                                                                                                                                                                                                   |
| `strategy/`               | Standing analysis that informs intent without stating it — one subfolder per class of analysis, each a hub over its own findings. `strategy/market-research/<product-slug>/` holds what a competing or adjacent product does and what its differences ask of us; a subject earns a folder when it will be revisited, never for a single glance. Findings inform the charter and the registry; they never substitute for either.                                                                                                                                                                                                                                                                    |
| `context/`                | These authoring conventions — the structure and the voice. Agent-facing, read on demand.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |

**Every plan is accounted for.** Each epic plan under `plans/` is linked from its row in `epics.md` (live work — the
epic registry aligns the priority of the plans); each epic plan under `delivered/` is a fully-delivered epic whose
frontmatter marks every slice `delivered` or `retired`; and each slice plan, in either place, is linked from its epic
plan's `slices` frontmatter. A plan referenced from none of these is orphaned, and that is a defect to fix in the same
change that surfaces it. One state is exempt: a slice that landed ahead of every milestone it serves has left `epics.md`
and cannot yet enter `delivered.md`, so its plan carries no link until that destination is reached — the epic chart
marking it `delivered` is the accounting in the meantime.

**Every epic is accounted for.** Each epic cited anywhere — a milestone's epic chart, a plan, an issue — resolves to a
row in `epics.md`, to a `delivered`-marked row in a live milestone's epic chart, or to `delivered.md`. A milestone may
demand work that does not exist yet, but demanding it and creating its epic row happen in the same change.

## Deliberately absent

These folders are missing on purpose. Their absence is a design decision, not an oversight — do not introduce them.

- **Work-tracking folders of any kind** — blizzard's ready queue is GitHub and its execution record is closed issues
  plus `delivered.md`. The execution state kept here is coarse and deliberate: per-slice status in each epic plan's
  frontmatter, and one one-way move of a finished epic from `plans/` to `delivered/`. Progress is never mirrored in
  folders — no `in-progress/`, no per-status directories, nothing that moves back and forth.
- **Design or decision logs** — this repo states current intent only; when intent changes, its files change, and git
  history is the log.

## Admitting something new

A new folder earns its place only when a class of files exists that no current row can honestly hold, and admitting it
means updating the table above and the root `index.md` routing in the same change (`canon:index-scrutiny`). One file is
not a class — park it at the root until it has siblings.
