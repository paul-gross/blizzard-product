---
epic: strict-types
refinement: scaffolded
status: horizon
---

# Plan — `epic:strict-types`

A type annotation is a promise a reader can lean on without checking. Most of blizzard is now written and reviewed by
agents meeting a module for the first time (`persona:fleet-agent`), and a signature is the first thing they read: when
it says a function takes a connection and returns a chunk, they build on that, and when it says nothing, they guess. The
type checker is what keeps the promise honest, so a codebase whose checker is quietly told to look away has promises
nobody is keeping.

That is where blizzard stands. Pyright runs on every change, but in its `standard` mode, which never asks whether a
function's parameters are typed at all and never complains when it cannot tell what a value is. Around it, the code
carries some 2,500 `# type: ignore[...]` comments, written in the vocabulary of mypy, a checker the repository has never
run. Pyright reads every one of them as *silence anything on this line*, whatever the brackets say. Roughly 1,900 of
them silence nothing: they were copied from one module to the next because the module before had them. The 600 or so
that do work are hiding some 1,650 genuine type errors, 69 of them in shipping code. No review ever raised it, because
each comment looks deliberate on its own and the gate stays green. It is the kind of drift only a measurement finds.

This epic makes the checker strict and the code honest under it: every function's signature typed, every value's type
known, and every remaining suppression scoped to the one rule it waives, with the reason beside it. Measured on `master`
at `d2e3e916e`, strict mode with the blanket comments ignored reports about 21,000 errors — roughly 16,400 in the tests,
4,500 in `src`, and a handful in the exemplars. Most are not independent: a single untyped parameter makes every value
derived from it unknown, so a slice that types the root clears the errors downstream of it as well.

## How it lands

The first landing sets the ground every later slice stands on. It deletes the comments that silence nothing, rewrites
each one that does work as a pyright-native `# pyright: ignore[<rule>]` naming the rule it actually waives, and turns
off pyright's reading of bare `# type: ignore` so a blanket suppression can never return. In the same delivery it
switches the checker to strict, with every error the codebase carries today recorded as known debt: new code is held to
strict from that commit on, and the debt can only shrink. And it writes the rule down in `blizzard-context`, beside
`bzh:python-toolchain`, so a reviewer has a rule to cite, which is what was missing all along.

After that the debt is paid down in slices, each changing many lines for one reason in one place. The order is the
largest group of errors in the smallest area first, then each smaller group in turn, and where two groups are close in
size the smaller area goes first, so the early slices remove the most debt while touching the fewest files. Each slice
re-measures its group before it starts, because the slices before it will have cleared some of its knock-on errors. The
concentrated groups, in the order they measure today:

| Slice                                     |  Errors | Files | The one reason                                                             |
| ----------------------------------------- | ------: | ----: | -------------------------------------------------------------------------- |
| Hub store adapters                        |  ~2,100 |    46 | The SQL helpers take untyped connections, tables, columns, and rows        |
| Test factories with `object` overrides    |  ~1,000 |    81 | A few shared factories type their overrides as `object` and pass them on   |
| Compat OpenAPI classifier                 |    ~300 |     4 | Parsed OpenAPI documents are walked as untyped dictionaries                |
| Runner store adapters                     |    ~300 |    18 | Row-mapping helpers take untyped rows                                      |
| Runner harness adapters                   |    ~290 |    32 | Harness output and settings are read as untyped dictionaries               |
| Hub domain, auth, delivery, and both CLIs |    ~460 |    60 | Each package's own untyped edges, one slice per package                    |
| Test families, largest first              | ~10,000 |   550 | Each family's untyped helpers and fakes, one slice per family or subsystem |

The test families are the bulk of the debt but not one group: the runner's tests alone carry about 3,100 errors across
123 files, and split by the subsystem they exercise, each piece is smaller than the slices above it.

When what is left no longer clusters, the scraps are swept up across the whole codebase, one cause per sweep: test
fixture parameters (`tmp_path`, `monkeypatch`, the `hub` and `store` fixtures), about 2,600 of them across 180 files;
bare `dict` and other generics with no type arguments; context managers declared as returning an `Iterator`, which
typeshed now marks deprecated; the `_conforms_*` protocol checks strict mode reports as unused; and the optional,
attribute, and index errors left over. The last landing finds the recorded debt empty and removes the mechanism that
held it.

Every slice is behavior-preserving. A type error that turns out to be a real bug is fixed in its slice when the fix is
small and obvious, and filed on its own when it is not, so a typing slice never has to be reviewed as a behavior change.

## Decisions before filing

Three questions need answers before the first issue is filed. The first is how the debt is recorded. Pyright has no
baseline of its own: `basedpyright`, a fork of it, records one in a file each slice shrinks, while stock pyright can
only hold the line through a list of strict paths and per-rule switches, which is far coarser than the slices above. The
second is whether tests may reach into a module's private names: strict mode reports about 870 places where they do, 690
of them in tests, and the answer is either a rule waived for `tests/` or a sweep that removes the access. The third is
whether the `_conforms_*` checks keep their shape with the unused-function rule waived for them, or take a shape strict
mode reads as used.

## Out of scope

The web board, already held to Angular's strict template checking, and `blizzard-mock`, whose typing is its own repo's
concern.
