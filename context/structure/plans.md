# Structure of epic plans

The required shape of an epic plan and its slice plans under `plans/`. A review of a plan asserts its frontmatter and
placement against this guide; the prose itself answers to the [writing guide](../writing-guide.md). Single-story plans
with no epic above them carry no frontmatter and are not governed here.

## Purpose

Every epic has exactly one epic plan, live or delivered, and it is the per-epic view: how refined the epic's intent is,
which named slices it lands in, and where each slice stands. Its frontmatter is the source of slice status; the
milestone epic charts repeat it.

## Placement

- **Epic plan.** `plans/<epic-slug>.md` while no slice has a plan file of its own; `plans/<epic-slug>/index.md` once one
  does, with each slice plan beside it as `plans/<epic-slug>/<slice-slug>.md`.
- **Slice plan.** A slice that earns written requirements gets its own file — never a section appended to the epic plan.
  A slice plan with supporting pieces — part plans, a `spec/` — is a folder of its own,
  `plans/<epic-slug>/<slice-slug>/index.md`, and everything specific to that slice lives inside it
  (`plans/adapters/opencode/`).
- **Artifacts.** An epic's mocks and proofs-of-concept live in one `plans/<epic-slug>/artifacts/` at the top of the epic
  folder, shared by every slice, never in a slice folder (`plans/queue/artifacts/`).
- **Inline slices stand.** Epic plans written before this guide may carry their slices' requirements inline
  (`plans/multi-tenancy.md`); those slices carry no `plan` field. New slice plans follow the placement above.

## Required frontmatter

Every epic plan opens with a YAML frontmatter block — here, `plans/adapters/index.md`:

```yaml
---
epic: adapters
refinement: refined
slices:
  - name: claude-code
    status: delivered
    plan: ./claude-code/index.md
  - name: opencode
    status: horizon
    plan: ./opencode/index.md
  - name: codex
    status: horizon
    plan: ./codex/index.md
---
```

| Field             | Rule                                                                                                                                |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| `epic`            | The epic's slug, matching its `epic:<slug>` id.                                                                                     |
| `refinement`      | One of `scaffolded`, `refined`, `pristine`.                                                                                         |
| `slices`          | Every named slice of the epic, delivered and retired ones included, in the order they are meant to land. At least one.              |
| `slices[].name`   | A kebab-case slug, unique within the epic. An epic not yet cut into slices has a single slice named `full`.                         |
| `slices[].status` | One of `horizon`, `in-progress`, `delivered`, `retired`.                                                                            |
| `slices[].plan`   | Relative link to the slice's own plan file. Omitted while the slice has none, or when its requirements sit inline in the epic plan. |

### Refinement

Refinement records how far the product owner has honed the epic's intent. It is the owner's judgment, not something a
document can be checked for:

- **`scaffolded`** — captured and committed, typically from a single conversation, without further honing.
- **`refined`** — the owner has gone back and forth on it to home in on what they want.
- **`pristine`** — the owner has declared the intent finished.

An epic whose every slice is `delivered` or `retired` is `pristine` — finishing the epic settles its grade. Short of
that, an agent never chooses the grade. It writes `scaffolded` when it creates an epic plan, and otherwise writes only
the grade the owner gives it — but it makes sure every epic plan carries one.

### Slice status

- **`horizon`** — named and intended, not yet started. A written slice plan does not change this.
- **`in-progress`** — the slice's GitHub epic issue is filed.
- **`delivered`** — the slice has fully landed.
- **`retired`** — the slice was dropped unbuilt.

## Lifecycle

- **An epic enters the registry with its epic plan.** Creating the `epics.md` row and writing the epic plan —
  frontmatter, a capability-level statement of intent, and the slices as far as they are known — happen in the same
  change.
- **Slice status moves with the work.** Filing a slice's GitHub epic issue sets it `in-progress`; completing or dropping
  it sets `delivered` or `retired`. Each change updates the epic plan's frontmatter and the `Status` cell of every
  milestone epic chart naming that slice, together.
- **A slice plan freezes when its slice ships**; the epic plan's frontmatter stays live until the epic leaves
  `epics.md`, and then the whole plan freezes with it — frontmatter included, so a delivered epic still names its
  slices.
- **A finished epic moves to `delivered/`.** In the change that marks its last slice `delivered` or `retired`, the
  epic's plan — `plans/<epic-slug>.md`, or the whole `plans/<epic-slug>/` folder — moves to `delivered/` with its shape
  intact, its grade becomes `pristine`, and every relative link to or from it is repointed. A partly-delivered epic
  stays under `plans/`, its frozen slice plans included.
