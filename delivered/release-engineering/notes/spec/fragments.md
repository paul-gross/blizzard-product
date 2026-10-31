# Change note contract

## Where a note lives

Every note is one Markdown file under `changes/` at the repository root.

| Path                         | Holds                                                                                                    |
| ---------------------------- | -------------------------------------------------------------------------------------------------------- |
| `changes/unreleased/<id>.md` | A note no release has carried yet. Written by the change it describes; editable by any change.           |
| `changes/<X.Y.Z>/<id>.md`    | A note the release `X.Y.Z` carried, moved there by the cut ([assembly.md](./assembly.md)).               |
| `changes/README.md`          | The contributor-facing reference for this format; the harness points at it ([harness.md](./harness.md)). |

A candidate files its notes under the version it is a candidate for — `v0.8.0-rc.1` and `v0.8.0` share `changes/0.8.0/`
— and a patch under its own patch version.

## The id

The file's stem is the note's id: lowercase kebab-case, at most 60 characters, `^[a-z0-9]+(-[a-z0-9]+)*$`, chosen by the
author to say what changed (`runner-add-takes-a-subscription`, `hub-drops-singular-usage`). An id is unique across the
whole of `changes/`, released directories included, because a removal cites a deprecation by it (§Fields) and because a
fix carried back to a released line keeps its id there ([assembly.md](./assembly.md) §Lines). Two chunks that choose the
same id meet as an add/add conflict when the second lands, which the chunk's conflict resolution settles by renaming its
own.

## Fields

A note opens with YAML frontmatter.

```markdown
---
kind: deprecated
surfaces: [http-api]
removes-in: "0.12"
---

`GET /api/runners/{id}/usage` is deprecated; read the runner's `subscriptions` instead. It is removed in 0.12.
```

| Field           | Required                                        | Rule                                                                                                                                                                                                                                        |
| --------------- | ----------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `kind`          | always                                          | One of `added`, `changed`, `deprecated`, `removed`, `fixed`, `security`.                                                                                                                                                                    |
| `surfaces`      | no                                              | Any of `http-api`, `wire`, `config`, `store`, `trace`, `egress` — the surfaces `docs/versioning.md` names — plus `cli` and `board`. Tells the reader where to look; the assembler groups nothing by it.                                     |
| `breaking`      | no, default `false`                             | `true` only on `changed` or `removed`. Requires an `## Upgrade` section and pairs with a `!` commit ([owed.md](./owed.md) §The breaking pair).                                                                                              |
| `breaks`        | on a `breaking` note the static diff sees       | The `where` of each break `epic:release-engineering/compat-floor` records for this change, so the cut can match the release's `compat-breaks.json` to its notes ([assembly.md](./assembly.md) §Breaks cross-checked).                       |
| `removes-in`    | on `deprecated`, and on a note that `postpones` | The release line that removes it, quoted: a major (`"2"`) from 1.0 on, a minor (`"0.12"`) before it. Must name a line later than the one the note is released in ([upgrade.md](./upgrade.md) §Deprecations).                                |
| `postpones`     | on a `changed` that moves a deprecation         | The id of a released deprecation whose removal moves to this note's `removes-in`, which must be later than the line it replaces.                                                                                                            |
| `retires`       | on a `removed` that carries out a deprecation   | The id of the deprecation note it carries out, which must exist in a released directory.                                                                                                                                                    |
| `upgrade-guide` | no                                              | An anchor in `docs/upgrade.md` (`host-graduation-boundary`) the note's upgrade section points to, which must resolve ([upgrade.md](./upgrade.md) §Boundaries).                                                                              |
| `flag`          | no                                              | A flag in `epic:release-engineering/dark-launch`'s registry, `src/blizzard/foundation/flags/registry.toml`, that the capability ships behind; the assembler holds the note back while it is off ([assembly.md](./assembly.md) §Held notes). |

Unknown fields fail validation, so a misspelt `remove-in` never silently drops a deprecation's date.

## The body

The body is what the operator reads. It is one to three sentences, present tense, in the operator's vocabulary
(`bzh:operator-vocabulary`), saying what is now true for them and naming the verb, key, route, or surface they would
recognise — never the module that changed or the chunk, issue, or pull request it came from, which the Vale
process-reference lint (`blizzard:process-ref-lint`) already refuses across every `.md` file. A `fixed` note names the
symptom the operator saw, not the cause.

An optional `## Upgrade` section follows, and is required when `breaking` is true or the kind is `removed`. It says what
the operator must do, in what order, before or after pulling the release; [upgrade.md](./upgrade.md) owns what it holds
and when it defers to `docs/upgrade.md`. No other heading is allowed.

## Validation

`blizzard-change-notes check` (`src/blizzard/tools/change_notes.py`, registered under `[project.scripts]` beside
`blizzard-compat`) validates the whole of `changes/` on every gate run — PR and push alike — and fails on:

- a file outside the two directory shapes above, or an id that breaks the pattern or repeats;
- frontmatter that is missing `kind`, carries an unknown field, or breaks a field's rule in the table;
- a body that is empty, or a heading other than `## Upgrade`;
- a `retires` or `postpones` that names no released deprecation, or an `upgrade-guide` anchor `docs/upgrade.md` does not
  have.

## Once released

A released note is the record of what its release said. A change may edit its body — a wording fix, a broken link — but
never its id, its directory, or its frontmatter, and never delete it; `blizzard-change-notes owed` refuses any of those
in a pull request's range ([owed.md](./owed.md)). Only the cut moves a note from `unreleased/` into a version's
directory. A note that was wrong in substance is answered by a new note in the next release, not by rewriting the old
one.
