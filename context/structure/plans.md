# Structure of plans

The required shape of the plans under `plans/`: epic plans, initiative plans, and the grouping folders that hold them. A
review of a plan asserts its frontmatter and placement against this guide; the prose itself answers to the
[writing guide](../writing-guide.md). Single-story plans with no registry entry above them carry no frontmatter and are
not governed here.

## Purpose

An **epic** is one unit of intent that lands as a whole: it is filed to GitHub as a single epic issue with its children,
and it is the unit every status and refinement grade attaches to. An **initiative** groups related epics under one
intent the owner is committed to, without committing to land every epic in the same milestone. Each epic and each
initiative has exactly one plan, live or delivered, and its frontmatter is the source of its refinement and, for an
epic, its status; the milestone epic charts repeat the status.

## Ids

- **`epic:<slug>`** — a standalone epic, `plans/<slug>.md` or `plans/<slug>/index.md`.
- **`epic:<group>/<slug>`** — an epic inside the grouping folder `plans/<group>/`, or named by the flat initiative
  `plans/<group>.md`. A group is one level deep; an id never carries a second `/`.
- **`initiative:<slug>`** — an initiative, `plans/<slug>.md` or `plans/<slug>/initiative.md`. The slug is the same on
  either side of the move into a folder, so the id survives it.

An id resolves through the frontmatter that declares it, never through the path alone.

## Placement

- **Standalone epic.** `plans/<slug>.md`; `plans/<slug>/index.md` once it carries supporting pieces — part plans, a
  `spec/`, `artifacts/` — all of which live inside its folder (`delivered/runner-host/`).
- **Flat initiative.** `plans/<slug>.md`, carrying the initiative's intent and naming its epics. Its epics have no files
  of their own: each is `horizon` and unwritten. The moment one of them earns a plan, a refinement grade, or a status
  other than `horizon`, the initiative becomes a grouping folder.
- **Grouping folder.** `plans/<group>/`, whose `index.md` carries no frontmatter and is a progressive-disclosure table
  routing to everything inside (`canon:hub-and-spoke`). It holds:
  - **Epics** — `<group>/<slug>.md`, or `<group>/<slug>/index.md` once that epic carries supporting pieces of its own
    (`plans/adapters/opencode/`).
  - **`initiative.md`**, optional — present when the group is an initiative. Every epic in the folder then belongs to
    it.
  - **`artifacts/`** — the group's mocks and proofs-of-concept, shared by its epics, never one per epic
    (`plans/chokepoint/artifacts/`).
  - **Shared documents** — a contract every epic in the group answers to, carrying no frontmatter
    (`plans/adapters/harness-selection.md`).

  A folder with no `initiative.md` is a plain grouping of related epics: each epic stands in the registry on its own.

## Required frontmatter

An epic plan opens with:

```yaml
---
epic: adapters/codex
refinement: refined
status: horizon
---
```

| Field        | Rule                                                                     |
| ------------ | ------------------------------------------------------------------------ |
| `epic`       | The epic's id without its `epic:` prefix — `<slug>` or `<group>/<slug>`. |
| `refinement` | One of `scaffolded`, `refined`, `pristine`.                              |
| `status`     | One of `horizon`, `in-progress`, `delivered`, `retired`.                 |

An initiative plan, flat or `initiative.md`, opens with:

```yaml
---
initiative: adapters
refinement: refined
epics:
  - claude-code
  - opencode
  - codex
---
```

| Field        | Rule                                                                                                                                                                      |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `initiative` | The initiative's slug, matching its `initiative:<slug>` id and, in a folder, the folder's name.                                                                           |
| `refinement` | One of `scaffolded`, `refined`, `pristine`.                                                                                                                               |
| `epics`      | The name of every epic in the initiative, delivered and retired ones included, in the order they are meant to land. Names only — status lives in each epic. At least two. |

An initiative carries no status of its own: it is finished when every epic in it is `delivered` or `retired`. One epic
is not an initiative — it is an epic.

### Refinement

Refinement records how far the product owner has honed the intent, and an epic and an initiative are graded
independently: an initiative's grade judges the intent and the cut into epics, an epic's grade judges that epic's own
requirements. It is the owner's judgment, not something a document can be checked for:

- **`scaffolded`** — captured and committed, typically from a single conversation, without further honing.
- **`refined`** — the owner has gone back and forth on it to home in on what they want.
- **`pristine`** — the owner has declared the intent finished.

An epic that is `delivered` or `retired` is `pristine`, and so is an initiative whose every epic is — finishing the work
settles its grade. Short of that, an agent never chooses the grade. It writes `scaffolded` when it creates a plan, and
otherwise writes only the grade the owner gives it — but it makes sure every plan carries one.

### Status

An epic's status, in its own frontmatter:

- **`horizon`** — named and intended, not yet started. A written plan does not change this.
- **`in-progress`** — the epic's GitHub epic issue is filed.
- **`delivered`** — the epic has fully landed.
- **`retired`** — the epic was dropped unbuilt.

An epic that landed or was dropped without a plan still has a file: its frontmatter and a sentence saying so
(`plans/security/runner-auth.md`).

## Lifecycle

- **A registry entry enters with its plan.** Creating the `epics.md` row and writing the plan — frontmatter and a
  statement of intent, plus an initiative's epics as far as they are known — happen in the same change.
- **An epic plan deepens in place.** It opens at capability level and, when the epic comes within striking distance,
  grows the detailed requirements a planning agent decomposes from — in the same file, or a folder once it needs
  supporting pieces.
- **Status moves with the work.** Filing an epic's GitHub epic issue sets it `in-progress`; completing or dropping it
  sets `delivered` or `retired`. Each change updates the epic's frontmatter and the `Status` cell of every milestone
  epic chart naming it, together.
- **Promoting an epic to an initiative is deliberate.** Cutting a standalone epic into several turns `epic:<slug>` into
  `initiative:<slug>`, and the change that does it rewrites every live citation of the old id.
- **A plan freezes when its work ships**; a delivered epic's plan states what was meant to be built, never what is
  built, and the durable behavioral record lands elsewhere as part of delivery.
- **Finished work moves to `delivered/`, shape intact**, and every relative link to or from it is repointed:
  - A standalone epic's file or folder moves in the change that marks it `delivered` or `retired`.
  - An initiative's folder moves whole in the change that settles its last epic, its grade becoming `pristine`; until
    then its settled epics stay beside the live ones.
  - An epic in a plain grouping moves alone, to `delivered/<group>/`, keeping its id; each side's `index.md` routes to
    what it holds, and the group's shared pieces follow its last epic out.
