# The notes in the harness

A note is only written if the worker making the change knows it owes one, so the epic is finished when an agent told to
make any change in `blizzard` meets the rule, the procedure, and the check through the routes it already reads. All
paths in this file are in `blizzard-context`.

## Rules

`standards/change-notes.md` is a hub row in `standards/index.md` — *writing or reviewing any change an operator would
notice, a breaking change, or a deprecation* — routing to three rules at file-per-rule granularity (`canon:rule-shape`):

| Rule                   | Says                                                                                                                                                                                                                           |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `bzh:change-note`      | When a note is owed and what its body says: the judgement of [owed.md](./owed.md) §The judgement, and the body rules of [fragments.md](./fragments.md) §The body, pointing to `blizzard`'s `changes/README.md` for the format. |
| `bzh:upgrade-note`     | What an upgrade section holds, and when the crossing earns a boundary section in `docs/upgrade.md` instead ([upgrade.md](./upgrade.md)).                                                                                       |
| `bzh:deprecation-note` | A deprecation names its removal line, a postponement is a note, and a removal names what it retires ([upgrade.md](./upgrade.md)).                                                                                              |

`standards/operator-vocabulary.md`'s scope grows to name change notes beside `docs/` operator prose.

## The procedure

`workflows/change-notes.md` (`bzh:change-notes`) is the procedure a worker follows inside its node, routed from
`workflows/index.md`: decide whether the change owes a note; choose an id; set the kind and fields; write the body; for
a break, write the upgrade section, add the `!` marker, and add a boundary to `docs/upgrade.md` when the crossing needs
one; for a deprecation, name the removal line; for a removal, name what it retires; for a contract change no operator
would notice, write the waiver trailer; and prove it with `blizzard:change-notes`. It is a choreography of the facts the
rules and `changes/README.md` own, restating none of them.

## The matrix

`verification/blizzard.md` gains `blizzard:change-notes` — `mise run change-notes` — with the rules. It runs `check`
from the start, and gains the merge-base run of `blizzard-change-notes owed` when the gate's trigger lands
([carry-over.md](./carry-over.md) §The order of landings); from then on the row for every surface the trigger watches
([owed.md](./owed.md) §The surface trigger) names it among the methods that prove a change to that surface.

## Cutting a release

`workflows/release.md` (`bzh:release`) loses the step that replaces the placeholder **Upgrade notes** by hand after the
tag. Its preparation step runs `blizzard-change-notes cut <version>` in release-lines' cut commit
([train.md](../../release-lines/spec/train.md) §The cut commit), and its confirmation step reads the published notes
against `blizzard-change-notes render <tag>`. Where the procedures for the candidate, the patch, and the major live is
`epic:release-engineering/release-lines`'; each calls the cut, and none restates it.
