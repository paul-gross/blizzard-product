# blizzard-product

Blizzard's **live product intent** — what we are building and why, kept current. This repo is the carry-over and continuation of the frozen [blizzard-discovery](https://github.com/paul-gross/blizzard-discovery) corpus: the epic registry and its `epic:<slug>` ids continue from `blizzard-discovery:/product/epics.md` and live here now. Read this repo when **planning** what to build.

## Path notation

Files here are addressed with the `blizzard-product:` prefix — for example, `blizzard-product:/roadmap.md`. Resolve to the on-disk path via the `# Winter Extensions` block in workspace `CLAUDE.md`.

## The three tiers

Product intent flows through three tiers, each with its own home. Promotion between them is deliberate — the workflow is in [CONTRIBUTING.md](./CONTRIBUTING.md).

| Tier | What it holds | Where |
|------|---------------|-------|
| **Horizon** | The epic registry: capability areas in priority order, a paragraph each | [roadmap.md](./roadmap.md) |
| **Requirements** | One detailed doc per approaching feature, written at promotion time | `blizzard-product:/epics/<epic-slug>.md` |
| **Ready** | Sized, well-specified work the factory can ingest | [GitHub issues on `paul-gross/blizzard`](https://github.com/paul-gross/blizzard/issues) |

## Authoring

Writing anything here — a roadmap entry, a requirements doc, a new file — starts from the authoring conventions in [context/index.md](./context/index.md): the repo's structure rules and the voice its product documents are written in.
