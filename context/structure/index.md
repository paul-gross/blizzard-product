# Repository structure

What lives where in `blizzard-product`, and which folders are allowed to exist. A file that fits none of these rows belongs somewhere else — most likely in a GitHub issue (work that is ready), or nowhere (`winter-canon:/organization.md`, `canon:admission-test`).

The registry files each have a structural guide beside this hub — the required shape a review asserts their contents against. Read the one guide for the file you are authoring or reviewing:

| File | Structural guide |
|------|------------------|
| `epics.md` | [structure/epics.md](./epics.md) |
| `milestones.md` | [structure/milestones.md](./milestones.md) |
| `delivered.md` | [structure/delivered.md](./delivered.md) |

## The layout

| Path | What belongs there |
|------|--------------------|
| `charter/` | The standing statement everything else answers to: [mission.md](../../charter/mission.md), [vision.md](../../charter/vision.md), and [personas.md](../../charter/personas.md) with its per-persona cards in `charter/personas/<persona-slug>.md`. The single home of every `persona:<slug>` id. Barely moves; amend it before making a change that contradicts it. |
| `epics.md` | The epic registry and the priority order of the plans: a single priority-ordered table of capability areas, carrying no execution status. An epic's id lives on its row here until the work completes. |
| `milestones.md` | The milestone registry: what users will be able to do. Each milestone demands the epics that reach it (an epic may serve several), and its id lives on its row here until the destination is reached. |
| `delivered.md` | The ledger of landed work, organized by destination: a section per delivered milestone with the grid of epic slices that carried it, and a closing section for work landed outside any milestone. A completed id resolves here. |
| `plans/` | What to build, one plan per promoted piece of work — an epic (large, multi-story) or a single-story plan. A lean plan is a solo `plans/<slug>.md`; a plan with mocks or supporting pieces is a folder `plans/<slug>/` whose `index.md` is the progressive-disclosure base routing to everything inside (`canon:hub-and-spoke`). A plan is written at promotion time, cited by the issues that decompose it, and frozen once shipped — what *is* built lives elsewhere, never here. The folder materializes with the first promotion. |
| `plans/<slug>/artifacts/` | Everything a plan carries that is not markdown — HTML mocks, code proofs-of-concept, images, whatever the plan needs to show rather than tell. One folder for all of it, routed from the plan's `index.md`, so the plan folder itself stays a readable markdown surface. |
| `context/` | These authoring conventions — the structure and the voice. Agent-facing, read on demand. |

**Every plan is accounted for.** Each plan under `plans/` is linked from a row in `epics.md` (live work — the epic registry aligns the priority of the plans) or in `delivered.md` (completed work). A plan referenced from neither is orphaned, and that is a defect to fix in the same change that surfaces it.

**Every epic is accounted for.** Each epic cited anywhere — a milestone's epic chart, a plan, an issue — resolves to a row in `epics.md` or `delivered.md`. A milestone may demand work that does not exist yet, but demanding it and creating its epic row happen in the same change.

## Deliberately absent

These folders are missing on purpose. Their absence is a design decision, not an oversight — do not introduce them.

- **Work-tracking folders of any kind** — blizzard's ready queue is GitHub and its execution record is closed issues plus `delivered.md`; mirroring that state in directories would create a second source of truth to drift.
- **Design or decision logs** — this repo states current intent only; when intent changes, its files change, and git history is the log.

## Admitting something new

A new folder earns its place only when a class of files exists that no current row can honestly hold, and admitting it means updating the table above and the root `index.md` routing in the same change (`canon:index-scrutiny`). One file is not a class — park it at the root until it has siblings.
