# blizzard-product

Blizzard's **live product intent** — what we are building and why, kept current. Read this repo when **planning** what to build.

## The charter

Who blizzard serves and why it exists — [mission](./charter/mission.md), [vision](./charter/vision.md), and [personas](./charter/personas.md) — lives in `charter/`: the standing statement every registry row and plan answers to.

## How intent becomes work

An epic starts as a row in the registry, grows a plan when its time approaches, and ends as filed issues the factory ingests. Each promotion is deliberate — the workflow is in [CONTRIBUTING.md](./CONTRIBUTING.md).

| Where | What it holds |
|-------|---------------|
| [epics.md](./epics.md) | The epic registry: capability areas in priority order, a paragraph each |
| [milestones.md](./milestones.md) | The milestone registry: what users will be able to do, each destination demanding its epics |
| `plans/` | What to build: one plan per promoted epic or single story — a solo file, or a folder whose `index.md` routes its pieces |
| Backlog source | Ready work: sized, well-specified items the fleet can take |
| [delivered.md](./delivered.md) | The ledger of landed work: a section per delivered milestone with the epic slices that carried it |

## Authoring

Writing anything here — a registry entry, a plan, a new file — starts from the authoring conventions in [context/index.md](./context/index.md): the repo's structure rules and the voice its product documents are written in.
