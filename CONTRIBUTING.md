# Contributing

`blizzard-product` is the live product-intent repo — the charter, the epic and milestone registries, and the plans the rest of the blizzard ecosystem builds from.

## The promotion workflow

An epic moves from a registry row, to a plan, to filed issues — and each promotion is a deliberate act, not a drift:

1. **A registry row** — think top-down: where a destination exists, declare it first in [milestones.md](./milestones.md) as what users will be able to do, and let it demand its epics. Each piece of demanded work — plus any epic standing on its own operational necessity — enters [epics.md](./epics.md) as a row with a stable `epic:<slug>` id, a paragraph of capability, and a priority position. No requirements, no issues.
2. **A plan** — when an epic (or a slice of one) comes within striking distance — work you would start in the next few weeks — write its plan under `plans/`: detailed requirements, decided scope, resolved open questions. A lean plan is a solo `plans/<slug>.md`; one with mocks or supporting pieces is a folder whose `index.md` routes to them. Link the plan from the epic's row. This is what a planning agent reads before decomposing work. A plan states what to *build*, never what is built — it freezes when the work ships, and the durable behavioral record lands elsewhere as part of delivery.
3. **Filed issues** — file the GitHub epic issue and its child issues (the `/wg-issue` flow), each citing `epic:<slug>`. The epic's row stays put in [epics.md](./epics.md) — the registry carries no filed/ongoing state, so filing issues neither moves nor annotates the row; the milestone epic charts and GitHub track where the work stands. The issue tracker is the factory's intake queue, not its memory: file only work that is startable, keep the open count small.

Exceptions and the far end of the lifecycle:

- **Single-story plans** — smaller work that still deserves a written plan before decomposing gets one under `plans/` the same way, with or without an epic row above it.
- **Bugs and small chores skip the registry and the plan** and go straight to GitHub. They are not product intent.
- **Closing issues is the execution record.** When an epic's slice completes, drop it from [epics.md](./epics.md) (a partially-landed epic keeps its row, re-scoped to the slice that remains) and mark it `delivered` in the epic chart of each [milestone](./milestones.md) it serves. The slice enters [delivered.md](./delivered.md) when that milestone is reached — carried into the milestone's section — or immediately under the outside-the-milestones section if it served no milestone. When a milestone is reached, its delivered.md section is the record and its row leaves milestones.md. Delivered entries keep their links to plans.
- Nothing files to GitHub as a feature without a plan behind it.

## Epic and milestone ids

`epic:<slug>` and `milestone:<slug>` ids are stable: citations in code and issues depend on them, so renaming or removing one is a breaking change. An id's single home is its row — in epics.md or milestones.md while the work is live, in delivered.md once it lands — there is no separate registry.

Cite an epic by id, not by path: the epic's row carries the only deep link to its plan, so a plan can change shape — a solo file growing into a folder — with a one-line row edit and no broken citations.

## Commit messages

Conventional Commits with a scope, matching the sibling repos:

    <type>(<scope>): <description>

- Types: `feat`, `fix`, `docs`, `chore`. `docs` is the common case; use `feat` for a new epic row or plan, `chore` for row moves and status updates.
- Scope: the epic slug where one applies (`docs(security): …`), or `epics` / `milestones` for registry-wide changes.
- The `/wf-commit` skill generates commits in this format — prefer it over hand-writing messages.

## Authoring conventions

The repo's structure rules and the voice its product documents are written in live under [context/](./context/index.md) — read the relevant file before adding or editing anything here.

## Delivery

- Default branch: `master`.
- Push directly to `master` — no PR, no review. Rebase onto the latest `origin/master` first so history stays linear.
