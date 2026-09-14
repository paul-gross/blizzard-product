# Maintaining

The mechanics of keeping this repo current: how intent is promoted, how ids are held stable, and the checks and delivery
path every change goes through. Raising an idea from outside is [CONTRIBUTING.md](./CONTRIBUTING.md).

## The promotion workflow

An epic moves from a registry row and epic plan, to slice plans, to filed issues, and each promotion is a deliberate act
rather than a drift. The epic plan's frontmatter is the per-epic view throughout — its refinement and each slice's
status — and its shape is owned by [context/structure/plans.md](./context/structure/plans.md):

1. **A registry row and its epic plan.** Think top-down: where a destination exists, declare it first in
   [milestones.md](./milestones.md) as what users will be able to do, and let it demand its epics. Each piece of
   demanded work, plus any epic standing on its own operational necessity, enters [epics.md](./epics.md) as a row with a
   stable `epic:<slug>` id, a paragraph of capability, and a priority position — and, in the same change, gets its epic
   plan `plans/<slug>.md`, linked from the row: frontmatter with `refinement: scaffolded` (or the grade the product
   owner gives) and its slices as far as they are known, all at `horizon`, above a capability-level statement of intent.
   No issues. Refinement is the owner's call: re-grade it only when they say so.
2. **A slice plan.** When a slice comes within striking distance, meaning work you would start in the next few weeks,
   write its plan: detailed requirements, decided scope, resolved open questions. It is its own file beside the epic
   plan (the epic plan becomes `plans/<slug>/index.md` if it is not a folder yet), linked from the slice's `plan` field.
   This is what a planning agent reads before decomposing work. A slice plan states what to *build*, never what is
   built: it freezes when the slice ships, and the durable behavioral record lands elsewhere as part of delivery.
3. **Filed issues.** File the GitHub epic issue and its child issues (the `/wg-issue` flow), each citing `epic:<slug>`,
   and set the slice `in-progress` in the epic plan and in every milestone epic chart naming it. The epic's row stays
   put in [epics.md](./epics.md) and is never annotated. The issue tracker is the factory's intake queue, not its
   memory: file only work that is startable, and keep the open count small.

Exceptions and the far end of the lifecycle:

- **Single-story plans.** Smaller work that still deserves a written plan before decomposing gets one under `plans/` the
  same way, with or without an epic row above it.
- **Bugs and small chores skip the registry and the plan** and go straight to GitHub. They are not product intent.
- **Closing issues is the execution record.** When an epic's slice completes, mark it `delivered` in its epic plan and
  in the epic chart of each [milestone](./milestones.md) it serves (a slice dropped unbuilt is marked `retired` the same
  way), and drop it from [epics.md](./epics.md) — a partially-landed epic keeps its row, re-scoped to the slice that
  remains. The slice enters [delivered.md](./delivered.md) when that milestone is reached, carried into the milestone's
  section, or immediately under the outside-the-milestones section if it served no milestone. When a milestone is
  reached, its delivered.md section is the record and its row leaves milestones.md. Delivered entries keep their links
  to plans.
- Nothing files to GitHub as a feature without a plan behind it.

## Epic and milestone ids

`epic:<slug>` and `milestone:<slug>` ids are stable: citations in code and issues depend on them, so renaming or
removing one is a breaking change. An id's single home is its row, in epics.md or milestones.md while the work is live
and in delivered.md once it lands. There is no separate registry.

Cite an epic by id, not by path: the epic's row carries the only deep link to its plan, so a plan can change shape, a
solo file growing into a folder, with a one-line row edit and no broken citations.

## Authoring conventions

The repo's structure rules and the voice its product documents are written in live under [context/](./context/index.md).
Read the relevant file before adding or editing anything here.

## Commit messages

Conventional Commits with a scope, matching the sibling repos:

    <type>(<scope>): <description>

- Types: `feat`, `fix`, `docs`, `chore`. `docs` is the common case; use `feat` for a new epic row or plan, and `chore`
  for row moves and status updates.
- Scope: the epic slug where one applies (`docs(security): …`), or `epics` / `milestones` for registry-wide changes.
- The `/wf-commit` skill generates commits in this format. Prefer it over hand-writing messages.

## Markdown checks

Every `.md` file is formatted with [dprint](https://dprint.dev/) and linted with
[rumdl](https://github.com/rvben/rumdl), configured in `dprint.json` and `.rumdl.toml`.

- Check: `dprint check` and `rumdl check .`
- Fix: `dprint fmt` and `rumdl check . --fix`

Run both before pushing.

## Delivery

- Default branch: `master`.
- Push directly to `master`, with no PR and no review. Rebase onto the latest `origin/master` first so history stays
  linear.
