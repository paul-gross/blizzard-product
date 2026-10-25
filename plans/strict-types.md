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
known, and no suppression left that does not name the one rule it waives. Measured on `master` at `d2e3e916e`, strict
mode with the blanket comments ignored reports about 21,000 errors — roughly 16,400 in the tests, 4,500 in `src`, and a
handful in the exemplars. Most are not independent: a single untyped parameter makes every value derived from it
unknown, so a slice that types the root clears the errors downstream of it as well.

## How it lands

The work is a burn-down first and a switch second. Nothing in the gate changes while the count comes down: the debt is
not recorded, no area is locked as it is cleaned, and pyright stays in `standard` mode until the number is low enough
that turning strict on is a small change rather than a project.

The first landing clears the ground and gives every later slice the same measuring stick. It deletes the comments in
shipping code that silence nothing, about 150 of the 1,900 (the rest sit in tests, and go with the test wave), and adds
a strict report that gates nothing: pyright in strict mode, reading no bare `# type: ignore`, run on demand, so every
slice measures its group before it starts and proves it gone when it finishes.

After that the debt is paid down in slices, each changing many lines for one reason in one place. The order is the
largest group of errors in the smallest area first, then each smaller group in turn, and where two groups are close in
size the smaller area goes first, so the early slices remove the most debt while touching the fewest files. Each slice
re-measures its group before it starts, because the slices before it will have cleared some of its knock-on errors. A
slice that types a line a blanket comment was covering deletes the comment with it. The concentrated groups, in the
order they measure today:

| Slice                                     |  Errors | Files | The one reason                                                             |
| ----------------------------------------- | ------: | ----: | -------------------------------------------------------------------------- |
| Hub store adapters                        |  ~2,100 |    46 | The SQL helpers take untyped connections, tables, columns, and rows        |
| Test factories with `object` overrides    |  ~1,000 |    81 | A few shared factories type their overrides as `object` and pass them on   |
| Compat OpenAPI classifier                 |    ~300 |     4 | Parsed OpenAPI documents are walked as untyped dictionaries                |
| Runner store adapters                     |    ~300 |    18 | Row-mapping helpers take untyped rows                                      |
| Runner harness adapters                   |    ~290 |    32 | Harness output and settings are read as untyped dictionaries               |
| Hub domain, auth, delivery, and both CLIs |    ~460 |    60 | Each package's own untyped edges, one slice per package                    |
| Test families, largest first              | ~11,000 |   550 | Each family's untyped helpers and fakes, one slice per family or subsystem |

The test families are the bulk of the debt but not one group: the runner's tests alone carry about 3,300 errors across
some 160 files, and split by the subsystem they exercise, each piece is smaller than the slices above it.

When what is left no longer clusters, the scraps are swept up across the whole codebase, one cause per sweep: test
fixture parameters (`tmp_path`, `monkeypatch`, the `hub` and `store` fixtures), about 2,600 of them across 180 files;
bare `dict` and other generics with no type arguments; context managers declared as returning an `Iterator`, which
typeshed now marks deprecated; and the optional, attribute, and index errors left over.

Two of the sweeps change a shape rather than add an annotation. The `_conforms_*` checks, some 270 functions that exist
only so pyright proves a class satisfies a protocol it never inherits, are never called, so strict mode reports every
one as unused. Each becomes a single annotated assignment of the class to the protocol's type,
`_permit_store_conforms: type[IWritePermitRepository] = PermitStore`, which proves the same fit, missing members and
mismatched signatures alike, without a function nobody calls; the architecture rule that prescribes the sentinel changes
in the same delivery. And tests stop reaching into private names: some 830 places, across 260 files, read a module's
underscored internals. A test reaches a module only through what it publishes, so each place either goes through the
public surface or reveals a name that was public all along and loses its underscore. These slices are grouped by the
name reached for, and they change how a test is written, not only how it is typed.

The exemplars move with the code they model. Agents write new code by imitating them, so an exemplar left loose while
its production counterpart is cleaned teaches the old shape to every module written in the meantime. Each slice brings
the exemplar of its layer along in the same change — the permit store with the hub store adapters, the clean land action
with hub delivery, the conformance sentinels and the private-name reach with their sweeps — and an exemplar is never the
last thing in its layer to be typed, as `bzh:exemplar-conformance` already demands.

Every slice is behavior-preserving. A type error that turns out to be a real bug is fixed in its slice when the fix is
small and obvious, and filed on its own when it is not, so a typing slice never has to be reviewed as a behavior change.

## Shipping code first, tests after the migration

The burn-down starts with `src/` and the exemplars and leaves the tests alone. `initiative:test-suite` is moving the
service tier out of Python and into Playwright, and typing a test that is about to be ported is work thrown away, while
touching the files it is being ported from only invites conflicts. So the first wave is every slice above that lives in
shipping code — the store, compat, harness, domain, auth, delivery, and CLI slices, with their exemplars — and the
sweeps for bare generics, context managers, conformance sentinels, and leftovers run over `src/` and the exemplars only.
That is about 4,500 of the 21,000, and the part an agent reads before it builds on anything.

Every test slice waits: the factories, the test families, the fixture-parameter sweep, the private-name slices, and the
store migration tests that live under `src/`. They are cut and filed once the service tier has moved, against whatever
tests remain in Python, and the ported service tests are never typed at all.

## Turning it on

How strict mode is switched on, and how it is held once it is, is decided when the count is low, not before: what is
left by then decides whether a final sweep clears it or a short list of waivers is written down. The epic is delivered
when the gate runs strict. The rule lands with it, in `blizzard-context` beside `bzh:python-toolchain`, in the same
change that makes the code obey it, so a reviewer has a rule to cite, which is what was missing all along.

## Out of scope

The web board, already held to Angular's strict template checking, and `blizzard-mock`, whose typing is its own repo's
concern.
