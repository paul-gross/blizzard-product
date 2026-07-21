# Contributing

`blizzard-product` is the live product-intent repo — the roadmap and per-feature requirements the rest of the blizzard ecosystem plans against. It is the continuation of the frozen `blizzard-discovery` corpus: intent changes land here, never there.

## The promotion workflow

Product intent moves through three tiers, and promotion between them is a deliberate act, not a drift:

1. **Horizon** — a capability enters [roadmap.md](./roadmap.md) as an epic row with a stable `epic:<slug>` id, a paragraph of capability, and a priority position. No requirements, no issues.
2. **Requirements** — when an epic (or a slice of one) comes within striking distance — work you would start in the next few weeks — write its requirements doc as `epics/<epic-slug>.md`: detailed requirements, decided scope, resolved open questions. This is the doc the filed issues will cite and the doc a planning agent reads before decomposing work.
3. **Ready** — file the GitHub epic issue and its child issues (the `/wg-issue` flow), each citing `epic:<slug>` and linking the requirements doc. Move the roadmap row to **In flight**. The issue tracker is the factory's intake queue, not its memory: file only work that is startable, keep the open count small.

Exceptions and the far end of the lifecycle:

- **Bugs and small chores skip tiers 1–2** and go straight to GitHub. They are not product intent.
- **Closing issues is the execution record.** When an epic's slice completes, move its row to **Shipped** in roadmap.md — GitHub and the roadmap's Shipped table are the record.
- Nothing files to GitHub as a feature without passing through tier 2.

## Epic ids

`epic:<slug>` ids continue the scheme begun in `blizzard-discovery:/product/epics.md` and are stable: citations in the discovery corpus, code, and issues depend on them, so renaming or removing one is a breaking change. A row in roadmap.md is the id's single home — there is no separate registry.

## Commit messages

Conventional Commits with a scope, matching the sibling repos:

    <type>(<scope>): <description>

- Types: `feat`, `fix`, `docs`, `chore`. `docs` is the common case; use `feat` for a new epic or requirements doc, `chore` for row moves and status updates.
- Scope: the epic slug where one applies (`docs(security): …`), or `roadmap` for registry-wide changes.
- The `/wf-commit` skill generates commits in this format — prefer it over hand-writing messages.

## Authoring conventions

The repo's structure rules and the voice its product documents are written in live under [context/](./context/index.md) — read the relevant file before adding or editing anything here.

## Delivery

- Default branch: `master`.
- Push directly to `master` — no PR, no review. Rebase onto the latest `origin/master` first so history stays linear.
