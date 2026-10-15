# Maintaining

The mechanics of keeping this repo current: how intent is promoted, how ids are held stable, and the checks and delivery
path every change goes through. Raising an idea from outside is [CONTRIBUTING.md](./CONTRIBUTING.md).

## The promotion workflow

Intent moves from a registry row and its plan, to a plan deepened into requirements, to filed issues, and each promotion
is a deliberate act rather than a drift. Every epic and initiative plan carries its refinement in frontmatter, and every
epic its status; their shape is owned by [context/structure/plans.md](./context/structure/plans.md):

1. **A registry row and its plan.** Think top-down: where a destination exists, declare it first in
   [milestones.md](./milestones.md) as what users will be able to do, and let it demand its epics. Each piece of
   demanded work, plus any standing on its own operational necessity, enters [epics.md](./epics.md) as a row with a
   stable id, a paragraph of capability, and a priority position — and, in the same change, gets its plan, linked from
   the row: an epic as `epic:<slug>` with `plans/<slug>.md` at `status: horizon`, or work already known to land as
   several epics as `initiative:<slug>` with a flat `plans/<slug>.md` naming them. Either carries
   `refinement: scaffolded` (or the grade the product owner gives) above a capability-level statement of intent. No
   issues. Refinement is the owner's call: re-grade it only when they say so.
2. **Requirements.** When an epic comes within striking distance, meaning work you would start in the next few weeks,
   deepen its plan into detailed requirements, decided scope, resolved open questions. An epic a flat initiative names
   first gets its own file, which turns the initiative into a folder (`plans/<slug>/initiative.md` beside an `index.md`
   and the epic files). This is what a planning agent reads before decomposing work. A plan states what to *build*,
   never what is built: it freezes when the epic ships, and the durable behavioral record lands elsewhere as part of
   delivery.
3. **Filed issues.** File the GitHub epic issue and its child issues (the `/wg-issue` flow), each citing the epic's id,
   and set the epic `in-progress` in its plan and in every milestone epic chart naming it. The registry row stays put in
   [epics.md](./epics.md) and is never annotated. The issue tracker is the factory's intake queue, not its memory: file
   only work that is startable, and keep the open count small.

Exceptions and the far end of the lifecycle:

- **Single-story plans.** Smaller work that still deserves a written plan before decomposing gets one under `plans/` the
  same way, with or without a registry row above it.
- **Bugs and small chores skip the registry and the plan** and go straight to GitHub. They are not product intent.
- **An epic that outgrows itself becomes an initiative** — deliberately, rewriting every live citation of its `epic:` id
  to the new `initiative:` id in the same change.
- **Closing issues is the execution record.** When an epic completes, mark it `delivered` and `pristine` in its plan and
  `delivered` in the epic chart of each [milestone](./milestones.md) it serves (an epic dropped unbuilt is marked
  `retired` the same way). A standalone epic leaves [epics.md](./epics.md) and its plan moves to
  [delivered/](./delivered/) in the same change; an initiative keeps its row, its capability re-scoped to the work that
  remains, until its last epic settles and its folder moves ([the plan guide](./context/structure/plans.md) owns the
  moves). The epic enters [delivered.md](./delivered.md) when that milestone is reached, carried into the milestone's
  section, or immediately under the outside-the-milestones section if it served no milestone. When a milestone is
  reached, its delivered.md section is the record and its row leaves milestones.md. Delivered entries keep their links
  to plans.
- Nothing files to GitHub as a feature without a plan behind it.

## Ids

`initiative:<slug>`, `epic:<slug>`, `epic:<group>/<slug>`, and `milestone:<slug>` ids are stable: citations in code and
issues depend on them, so renaming or removing one is a breaking change. An id's single home is its row, in epics.md or
milestones.md while the work is live — an initiative's row homing the epics it names — and in delivered.md once it
lands. There is no separate registry.

Cite by id, not by path: the row carries the only deep link to the plan, so a plan can change shape, a solo file growing
into a folder, with a one-line row edit and no broken citations.

## Authoring conventions

The repo's structure rules and the voice its product documents are written in live under [context/](./context/index.md).
Read the relevant file before adding or editing anything here.

## Commit messages

Conventional Commits with a scope, matching the sibling repos:

    <type>(<scope>): <description>

- Types: `feat`, `fix`, `docs`, `chore`. `docs` is the common case; use `feat` for a new registry row or plan, and
  `chore` for row moves and status updates.
- Scope: the initiative or epic slug where one applies (`docs(security): …`), or `epics` / `milestones` for
  registry-wide changes.
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
